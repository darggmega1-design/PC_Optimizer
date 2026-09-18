
"""
Расширенные твики Windows, управление питанием, очистка ОЗУ.
"""
import os
import subprocess
import ctypes
import winreg
import psutil


def run_cmd(cmd: str) -> tuple:
    try:
        result = subprocess.run(
            cmd, shell=True, capture_output=True, text=True, encoding="cp866", errors="ignore"
        )
        return result.returncode == 0, result.stdout.strip() or result.stderr.strip()
    except Exception as e:
        return False, str(e)


# ============================================================
# 1. ОЧИСТКА ОПЕРАТИВНОЙ ПАМЯТИ
# ============================================================

def clean_ram() -> tuple:
    """Освобождает оперативную память через EmptyWorkingSet."""
    try:
        before = psutil.virtual_memory().percent

        # EmptyWorkingSet для каждого процесса
        psapi = ctypes.WinDLL("psapi.dll")
        kernel32 = ctypes.WinDLL("kernel32.dll")
        PROCESS_QUERY_INFORMATION = 0x0400
        PROCESS_SET_QUOTA = 0x0100

        freed_count = 0
        for proc in psutil.process_iter(["pid"]):
            try:
                handle = kernel32.OpenProcess(
                    PROCESS_QUERY_INFORMATION | PROCESS_SET_QUOTA, False, proc.info["pid"]
                )
                if handle:
                    psapi.EmptyWorkingSet(handle)
                    kernel32.CloseHandle(handle)
                    freed_count += 1
            except Exception:
                continue

        after = psutil.virtual_memory().percent
        diff = before - after
        return True, f"ОЗУ очищено: {freed_count} процессов, -{diff:.1f}% ({before:.0f}%→{after:.0f}%)"
    except Exception as e:
        return False, f"Ошибка: {e}"


# ============================================================
# 2. ТОЧКА ВОССТАНОВЛЕНИЯ
# ============================================================

def create_restore_point(description: str = "PC Optimizer") -> tuple:
    """Создаёт точку восстановления системы."""
    # Проверяем, включена ли защита системы
    ok, out = run_cmd(
        'powershell -Command "Get-ComputerRestorePoint"'
    )
    # Создаём через WMI
    ps_script = (
        'Checkpoint-Computer -Description "' + description + '" '
        '-RestorePointType "MODIFY_SETTINGS"'
    )
    ok, msg = run_cmd(f'powershell -Command "{ps_script}"')
    if ok:
        return True, f"Точка восстановления «{description}» создана"
    return False, f"Не удалось создать точку (включите защиту системы): {msg[:100]}"


def enable_system_protection() -> tuple:
    """Включает защиту системы на диске C."""
    ok, msg = run_cmd(
        'powershell -Command "Enable-ComputerRestore -Drive \'C:\\\'"'
    )
    return ok, "Защита системы включена" if ok else msg[:120]


# ============================================================
# 3. WINDOWS UPDATE
# ============================================================

def disable_windows_update() -> tuple:
    """Отключает автоматическое обновление Windows."""
    cmds = [
        'sc config wuauserv start= disabled & sc stop wuauserv',
        'sc config UsoSvc start= disabled & sc stop UsoSvc',
        'sc config BITS start= disabled & sc stop BITS',
        'reg add "HKLM\\SOFTWARE\\Policies\\Microsoft\\Windows\\WindowsUpdate\\AU" '
        '/v NoAutoUpdate /t REG_DWORD /d 1 /f',
        'reg add "HKLM\\SOFTWARE\\Policies\\Microsoft\\Windows\\WindowsUpdate\\AU" '
        '/v AUOptions /t REG_DWORD /d 1 /f',
    ]
    for c in cmds:
        run_cmd(c)
    return True, "Автообновление Windows отключено (перезагрузка)"


def enable_windows_update() -> tuple:
    """Включает автоматическое обновление обратно."""
    cmds = [
        'sc config wuauserv start= auto & sc start wuauserv',
        'sc config UsoSvc start= auto & sc start UsoSvc',
        'sc config BITS start= delayed-auto & sc start BITS',
        'reg delete "HKLM\\SOFTWARE\\Policies\\Microsoft\\Windows\\WindowsUpdate\\AU" /v NoAutoUpdate /f',
        'reg delete "HKLM\\SOFTWARE\\Policies\\Microsoft\\Windows\\WindowsUpdate\\AU" /v AUOptions /f',
    ]
    for c in cmds:
        run_cmd(c)
    return True, "Автообновление Windows включено"


# ============================================================
# 4. ТЕЛЕМЕТРИЯ И КОНФИДЕНЦИАЛЬНОСТЬ
# ============================================================

def disable_telemetry_full() -> tuple:
    """Полное отключение телеметрии."""
    cmds = [
        'sc config DiagTrack start= disabled & sc stop DiagTrack',
        'sc config dmwappushservice start= disabled & sc stop dmwappushservice',
        # Телеметрия в реестре
        'reg add "HKLM\\SOFTWARE\\Policies\\Microsoft\\Windows\\DataCollection" '
        '/v AllowTelemetry /t REG_DWORD /d 0 /f',
        # Отключение ID рекламы
        'reg add "HKCU\\Software\\Microsoft\\Windows\\CurrentVersion\\AdvertisingInfo" '
        '/v Enabled /t REG_DWORD /d 0 /f',
        # Отключение истории активности
        'reg add "HKLM\\SOFTWARE\\Policies\\Microsoft\\Windows\\System" '
        '/v EnableActivityFeed /t REG_DWORD /d 0 /f',
        'reg add "HKLM\\SOFTWARE\\Policies\\Microsoft\\Windows\\System" '
        '/v PublishUserActivities /t REG_DWORD /d 0 /f',
        'reg add "HKLM\\SOFTWARE\\Policies\\Microsoft\\Windows\\System" '
        '/v UploadUserActivities /t REG_DWORD /d 0 /f',
    ]
    for c in cmds:
        run_cmd(c)
    return True, "Телеметрия полностью отключена"


# ============================================================
# 5. ПИТАНИЕ И ЭНЕРГОСБЕРЕЖЕНИЕ
# ============================================================

def apply_power_tweaks() -> tuple:
    """Отключает энергосбережение для максимальной производительности."""
    cmds = [
        # Отключить USB selective suspend
        'powercfg /setacvalueindex SCHEME_CURRENT 2a737441-1930-4402-8d77-b2bebba308a3 48e6b7a6-50f5-4782-a5d4-53bb8f07e226 0',
        # Отключить PCIe ASPM
        'powercfg /setacvalueindex SCHEME_CURRENT SUB_PCIEXPRESS ASPM 0',
        # Отключить throttling процессора
        'powercfg /setacvalueindex SCHEME_CURRENT SUB_PROCESSOR PROCTHROTTLEMAX 100',
        'powercfg /setacvalueindex SCHEME_CURRENT SUB_PROCESSOR PROCTHROTTLEMIN 100',
        # Отключить sleep HDD
        'powercfg /setacvalueindex SCHEME_CURRENT SUB_DISK DISKIDLE 0',
        # Отключить гибернацию
        'powercfg -h off',
        # Применить
        'powercfg /setactive SCHEME_CURRENT',
    ]
    for c in cmds:
        run_cmd(c)
    return True, "Настройки питания применены (макс. производительность)"


# ============================================================
# 6. ПРОВОДНИК И ПУСК
# ============================================================

def optimize_explorer() -> tuple:
    """Оптимизация Проводника и меню Пуск."""
    cmds = [
        # Отключить недавние файлы
        'reg add "HKCU\\Software\\Microsoft\\Windows\\CurrentVersion\\Explorer\\Advanced" '
        '/v Start_TrackDocs /t REG_DWORD /d 0 /f',
        # Отключить недавние программы
        'reg add "HKCU\\Software\\Microsoft\\Windows\\CurrentVersion\\Explorer\\Advanced" '
        '/v Start_TrackProgs /t REG_DWORD /d 0 /f',
        # Открывать Проводник в "Этот компьютер"
        'reg add "HKCU\\Software\\Microsoft\\Windows\\CurrentVersion\\Explorer\\Advanced" '
        '/v LaunchTo /t REG_DWORD /d 1 /f',
        # Показывать расширения файлов
        'reg add "HKCU\\Software\\Microsoft\\Windows\\CurrentVersion\\Explorer\\Advanced" '
        '/v HideFileExt /t REG_DWORD /d 0 /f',
        # Показывать скрытые файлы
        'reg add "HKCU\\Software\\Microsoft\\Windows\\CurrentVersion\\Explorer\\Advanced" '
        '/v Hidden /t REG_DWORD /d 1 /f',
        # Отключить "Cortana"
        'reg add "HKLM\\SOFTWARE\\Policies\\Microsoft\\Windows\\Windows Search" '
        '/v AllowCortana /t REG_DWORD /d 0 /f',
        # Отключить Bing в поиске
        'reg add "HKCU\\Software\\Microsoft\\Windows\\CurrentVersion\\Search" '
        '/v BingSearchEnabled /t REG_DWORD /d 0 /f',
    ]
    for c in cmds:
        run_cmd(c)
    return True, "Проводник и Пуск оптимизированы"


# ============================================================
# 7. АВТОЗАПУСК ПРИЛОЖЕНИЯ С WINDOWS
# ============================================================

APP_REG_PATH = r"Software\Microsoft\Windows\CurrentVersion\Run"
APP_REG_NAME = "PCOptimizer"


def enable_autostart_app() -> tuple:
    """Добавляет приложение в автозапуск Windows."""
    try:
        import sys
        exe_path = sys.executable if getattr(sys, "frozen", False) else os.path.abspath(sys.argv[0])
        with winreg.OpenKey(winreg.HKEY_CURRENT_USER, APP_REG_PATH, 0, winreg.KEY_SET_VALUE) as key:
            winreg.SetValueEx(key, APP_REG_NAME, 0, winreg.REG_SZ, f'"{exe_path}"')
        return True, "Автозапуск включён"
    except Exception as e:
        return False, f"Ошибка: {e}"


def disable_autostart_app() -> tuple:
    """Убирает приложение из автозапуска."""
    try:
        with winreg.OpenKey(winreg.HKEY_CURRENT_USER, APP_REG_PATH, 0, winreg.KEY_SET_VALUE) as key:
            winreg.DeleteValue(key, APP_REG_NAME)
        return True, "Автозапуск отключён"
    except FileNotFoundError:
        return True, "Автозапуск уже отключён"
    except Exception as e:
        return False, f"Ошибка: {e}"


def is_app_in_autostart() -> bool:
    """Проверяет, есть ли приложение в автозапуске."""
    try:
        with winreg.OpenKey(winreg.HKEY_CURRENT_USER, APP_REG_PATH, 0, winreg.KEY_READ) as key:
            winreg.QueryValueEx(key, APP_REG_NAME)
            return True
    except Exception:
        return False


# ============================================================
# 8. ОЧИСТКА КОРЗИНЫ И КЭША ПРИЛОЖЕНИЙ
# ============================================================

def empty_recycle_bin() -> tuple:
    """Очищает корзину."""
    try:
        ctypes.windll.shell32.SHEmptyRecycleBinW(None, None, 0x00000001 | 0x00000002 | 0x00000004)
        return True, "Корзина очищена"
    except Exception as e:
        return False, f"Ошибка: {e}"


def clean_browser_cache() -> tuple:
    """Очищает кэш популярных браузеров."""
    paths = [
        os.path.expandvars(r"%LOCALAPPDATA%\Google\Chrome\User Data\Default\Cache"),
        os.path.expandvars(r"%LOCALAPPDATA%\Microsoft\Edge\User Data\Default\Cache"),
        os.path.expandvars(r"%LOCALAPPDATA%\Mozilla\Firefox\Profiles"),
        os.path.expandvars(r"%LOCALAPPDATA%\Yandex\YandexBrowser\User Data\Default\Cache"),
    ]
    freed = 0
    import shutil
    for p in paths:
        if not os.path.exists(p):
            continue
        try:
            for root, dirs, files in os.walk(p):
                for f in files:
                    fp = os.path.join(root, f)
                    try:
                        freed += os.path.getsize(fp)
                        os.remove(fp)
                    except Exception:
                        pass
        except Exception:
            continue
    mb = freed / (1024 * 1024)
    return True, f"Кэш браузеров очищен: ~{mb:.1f} МБ"
"""
Дополнительные модули: профили игр, процессы, автозагрузка, экспорт отчёта.
Все сообщения и комментарии — на русском языке.
"""
import os
import json
import psutil
import winreg
from datetime import datetime


# ============================================================
# 1. ПРОФИЛИ ИГР
# ============================================================

PROFILES_FILE = os.path.join(os.path.dirname(os.path.abspath(__file__)), "profiles.json")


def load_profiles() -> dict:
    """Загрузить все сохранённые профили игр."""
    if not os.path.exists(PROFILES_FILE):
        return {}
    try:
        with open(PROFILES_FILE, "r", encoding="utf-8") as f:
            return json.load(f)
    except Exception:
        return {}


def save_profiles(profiles: dict) -> bool:
    """Сохранить профили игр в файл."""
    try:
        with open(PROFILES_FILE, "w", encoding="utf-8") as f:
            json.dump(profiles, f, indent=2, ensure_ascii=False)
        return True
    except Exception:
        return False


def add_profile(name: str, exe_path: str, tweaks: list) -> tuple[bool, str]:
    """Добавить новый профиль игры."""
    if not name.strip():
        return False, "Имя профиля не может быть пустым"
    profiles = load_profiles()
    profiles[name] = {
        "exe_path": exe_path,
        "tweaks": tweaks,
        "created": datetime.now().strftime("%Y-%m-%d %H:%M"),
    }
    save_profiles(profiles)
    return True, f"Профиль «{name}» сохранён"


def remove_profile(name: str) -> tuple[bool, str]:
    """Удалить профиль игры."""
    profiles = load_profiles()
    if name in profiles:
        del profiles[name]
        save_profiles(profiles)
        return True, f"Профиль «{name}» удалён"
    return False, "Профиль не найден"


def get_profile_names() -> list:
    """Получить список имён всех профилей."""
    return list(load_profiles().keys())


# ============================================================
# 2. УПРАВЛЕНИЕ ПРОЦЕССАМИ
# ============================================================

def get_top_processes(limit: int = 20) -> list:
    """
    Вернуть список процессов, отсортированных по потреблению памяти.
    Каждый элемент: (имя, pid, память_МБ, CPU_%, статус)
    """
    processes = []
    for proc in psutil.process_iter(["pid", "name", "memory_info", "cpu_percent", "status"]):
        try:
            info = proc.info
            mem_mb = info["memory_info"].rss / (1024 * 1024) if info["memory_info"] else 0
            processes.append({
                "name": info["name"] or "—",
                "pid": info["pid"],
                "memory_mb": mem_mb,
                "cpu": info["cpu_percent"] or 0,
                "status": info["status"] or "—",
            })
        except (psutil.NoSuchProcess, psutil.AccessDenied):
            continue
    processes.sort(key=lambda p: p["memory_mb"], reverse=True)
    return processes[:limit]


def kill_process(pid: int) -> tuple[bool, str]:
    """Завершить процесс по PID."""
    try:
        proc = psutil.Process(pid)
        name = proc.name()
        proc.terminate()
        try:
            proc.wait(timeout=3)
        except psutil.TimeoutExpired:
            proc.kill()
        return True, f"Процесс «{name}» завершён"
    except psutil.NoSuchProcess:
        return False, "Процесс уже завершён"
    except psutil.AccessDenied:
        return False, "Нет прав для завершения процесса"
    except Exception as e:
        return False, f"Ошибка: {e}"


def set_process_priority(pid: int, priority: str) -> tuple[bool, str]:
    """
    Установить приоритет процесса.
    priority: 'high' | 'above_normal' | 'normal' | 'below_normal' | 'low'
    """
    mapping = {
        "high": psutil.HIGH_PRIORITY_CLASS,
        "above_normal": psutil.ABOVE_NORMAL_PRIORITY_CLASS,
        "normal": psutil.NORMAL_PRIORITY_CLASS,
        "below_normal": psutil.BELOW_NORMAL_PRIORITY_CLASS,
        "low": psutil.IDLE_PRIORITY_CLASS,
    }
    try:
        proc = psutil.Process(pid)
        proc.nice(mapping.get(priority, psutil.NORMAL_PRIORITY_CLASS))
        return True, f"Приоритет процесса «{proc.name()}» изменён на «{priority}»"
    except psutil.AccessDenied:
        return False, "Нет прав для изменения приоритета"
    except Exception as e:
        return False, f"Ошибка: {e}"


# ============================================================
# 3. УПРАВЛЕНИЕ АВТОЗАГРУЗКОЙ
# ============================================================

# Ключи реестра, где Windows хранит автозагрузку
AUTOSTART_KEYS = [
    (winreg.HKEY_CURRENT_USER,
     r"Software\Microsoft\Windows\CurrentVersion\Run",
     "Пользователь (HKCU)"),
    (winreg.HKEY_LOCAL_MACHINE,
     r"Software\Microsoft\Windows\CurrentVersion\Run",
     "Система (HKLM, 64-бит)"),
    (winreg.HKEY_LOCAL_MACHINE,
     r"Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Run",
     "Система (HKLM, 32-бит)"),
]


def get_autostart_items() -> list:
    """
    Получить список программ в автозагрузке.
    Возвращает: [(имя, путь, ключ_реестра, описание_ключа), ...]
    """
    items = []
    for hive, path, desc in AUTOSTART_KEYS:
        try:
            with winreg.OpenKey(hive, path, 0, winreg.KEY_READ) as key:
                i = 0
                while True:
                    try:
                        name, value, _ = winreg.EnumValue(key, i)
                        items.append((name, value, path, desc, hive))
                        i += 1
                    except OSError:
                        break
        except FileNotFoundError:
            continue
        except PermissionError:
            continue
    return items


def remove_autostart_item(name: str, path: str, hive) -> tuple[bool, str]:
    """Удалить программу из автозагрузки."""
    try:
        with winreg.OpenKey(hive, path, 0, winreg.KEY_SET_VALUE) as key:
            winreg.DeleteValue(key, name)
        return True, f"«{name}» удалён из автозагрузки"
    except FileNotFoundError:
        return False, "Программа не найдена в автозагрузке"
    except PermissionError:
        return False, "Нет прав для изменения (запустите от администратора)"
    except Exception as e:
        return False, f"Ошибка: {e}"


# ============================================================
# 4. ЭКСПОРТ ОТЧЁТА
# ============================================================

REPORT_FILE = os.path.join(os.path.dirname(os.path.abspath(__file__)), "report.txt")


def export_report(results: list, filename: str = None) -> tuple[bool, str]:
    """
    Сохранить отчёт об оптимизации в текстовый файл.
    results: список строк с результатами
    """
    if filename is None:
        filename = REPORT_FILE

    try:
        with open(filename, "w", encoding="utf-8") as f:
            f.write("=" * 60 + "\n")
            f.write("  ОТЧЁТ ОБ ОПТИМИЗАЦИИ ПК\n")
            f.write("=" * 60 + "\n")
            f.write(f"Дата: {datetime.now().strftime('%d.%m.%Y %H:%M:%S')}\n")
            f.write(f"Компьютер: {os.environ.get('COMPUTERNAME', '—')}\n")
            f.write(f"Пользователь: {os.environ.get('USERNAME', '—')}\n")
            f.write("=" * 60 + "\n\n")

            f.write("РЕЗУЛЬТАТЫ ВЫПОЛНЕННЫХ ОПЕРАЦИЙ:\n\n")
            for i, line in enumerate(results, 1):
                f.write(f"{i}. {line}\n")

            f.write("\n" + "=" * 60 + "\n")
            f.write("Отчёт создан автоматически программой PC Optimizer\n")
            f.write("=" * 60 + "\n")

        return True, f"Отчёт сохранён: {filename}"
    except Exception as e:
        return False, f"Ошибка сохранения: {e}"


# ============================================================
# 5. ИНФОРМАЦИЯ О СИСТЕМЕ
# ============================================================

def get_system_info() -> dict:
    """Собрать информацию о системе."""
    try:
        import platform
        cpu_freq = psutil.cpu_freq()
        return {
            "os": f"{platform.system()} {platform.release()}",
            "версия": platform.version(),
            "процессор": platform.processor() or "—",
            "ядра": f"{psutil.cpu_count(logical=False)} физ. / {psutil.cpu_count()} лог.",
            "частота": f"{cpu_freq.current:.0f} МГц" if cpu_freq else "—",
            "ОЗУ": f"{psutil.virtual_memory().total / (1024**3):.1f} ГБ",
            "имя ПК": platform.node(),
        }
    except Exception as e:
        return {"ошибка": str(e)}
#define MyAppName "PC Optimizer"
#define MyAppVersion "2.1"
#define MyAppPublisher "PC Optimizer Team"
#define MyAppExeName "PC_Optimizer.exe"

[Setup]
AppId={{8F3A7B2C-4E1D-4C8A-9B5E-1F2A3D4E5F60}
AppName={#MyAppName}
AppVersion={#MyAppVersion}
AppPublisher={#MyAppPublisher}
DefaultDirName={autopf}\{#MyAppName}
DefaultGroupName={#MyAppName}
DisableProgramGroupPage=yes
OutputDir=installer_output
OutputBaseFilename=PC_Optimizer_Setup
SetupIconFile=icon.ico
Compression=lzma2/max
SolidCompression=yes
PrivilegesRequired=admin
MinVersion=10.0
WizardStyle=modern
ArchitecturesInstallIn64BitMode=x64
UninstallDisplayIcon={app}\{#MyAppExeName}
UninstallDisplayName={#MyAppName}

[Languages]
Name: "russian"; MessagesFile: "compiler:Languages\Russian.isl"

[Tasks]
Name: "desktopicon"; Description: "{cm:CreateDesktopIcon}"; GroupDescription: "{cm:AdditionalIcons}"; Flags: checkedonce

[Files]
Source: "dist\PC_Optimizer\*"; DestDir: "{app}"; Flags: ignoreversion recursesubdirs createallsubdirs
Source: "icon.ico"; DestDir: "{app}"; Flags: ignoreversion
Source: "README.txt"; DestDir: "{app}"; Flags: ignoreversion isreadme

[Icons]
Name: "{group}\{#MyAppName}"; Filename: "{app}\{#MyAppExeName}"; IconFilename: "{app}\icon.ico"
Name: "{group}\Uninstall {#MyAppName}"; Filename: "{uninstallexe}"
Name: "{autodesktop}\{#MyAppName}"; Filename: "{app}\{#MyAppExeName}"; IconFilename: "{app}\icon.ico"; Tasks: desktopicon

[Run]
Filename: "{app}\{#MyAppExeName}"; Description: "{cm:LaunchProgram,{#StringChange(MyAppName, '&', '&&')}}"; Flags: nowait postinstall skipifsilent

[UninstallDelete]
Type: filesandordirs; Name: "{app}"
"""
Главный файл PC Optimizer.
Современный интерфейс с боковым меню и расширенными настройками.
"""
import os
import threading
from tkinter import messagebox, filedialog
import customtkinter as ctk
import optimizer as opt
import settings as st
import extras as ex
import nvidia as nv
import advanced as adv
import notify as nt
import ui_helpers as ui

# Загружаем настройки ДО создания окна
НАСТРОЙКИ = st.load_settings()

ctk.set_appearance_mode(НАСТРОЙКИ["appearance"])
ctk.set_default_color_theme(НАСТРОЙКИ["accent_color"])


class Приложение(ctk.CTk):
    def __init__(self):
        super().__init__()

        self.settings = НАСТРОЙКИ.copy()
        self.last_results = []
        self.активные_кнопки = {}
        self.текущий_раздел = None
        self.разделы = {}

        self.title("PC Optimizer — Ускорение для игр")
        self.geometry("1100x720")
        self.minsize(1000, 650)
        self.configure(fg_color=ui.COLORS["bg_dark"])

        # Горячие клавиши
        self.bind("<Control-Shift-KeyPress-O>", lambda e: self.применить_всё())
        self.bind("<F5>", lambda e: self.показать_раздел("main"))
        self.bind("<Escape>", lambda e: self.iconify())

        # ============================================================
        # ЛЕВОЕ МЕНЮ (SIDEBAR)
        # ============================================================
        self.sidebar = ctk.CTkFrame(
            self, width=240, corner_radius=0,
            fg_color=ui.COLORS["bg_sidebar"]
        )
        self.sidebar.pack(side="left", fill="y")
        self.sidebar.pack_propagate(False)

        # Логотип
        logo_frame = ctk.CTkFrame(self.sidebar, fg_color="transparent")
        logo_frame.pack(fill="x", padx=20, pady=(24, 20))

        ctk.CTkLabel(
            logo_frame, text="⚡",
            font=ctk.CTkFont(size=26)
        ).pack(side="left")

        ctk.CTkLabel(
            logo_frame, text="PC Optimizer",
            font=ctk.CTkFont(size=16, weight="bold"),
            text_color=ui.COLORS["text"]
        ).pack(side="left", padx=(8, 0))

        # Разделитель
        ctk.CTkFrame(
            self.sidebar, height=1, fg_color=ui.COLORS["border"]
        ).pack(fill="x", padx=20, pady=(0, 16))

        # Статус админа
        admin_frame = ctk.CTkFrame(
            self.sidebar, fg_color=ui.COLORS["bg_card"],
            corner_radius=8
        )
        admin_frame.pack(fill="x", padx=14, pady=(0, 20))

        if opt.is_admin():
            admin_icon, admin_text, admin_color = "✅", "Администратор", ui.COLORS["success"]
        else:
            admin_icon, admin_text, admin_color = "⚠️", "Запустите от админа", ui.COLORS["warning"]

        ctk.CTkLabel(
            admin_frame, text=f"{admin_icon}  {admin_text}",
            font=ctk.CTkFont(size=11, weight="bold"),
            text_color=admin_color
        ).pack(pady=10, padx=12)

        # Меню
        self.меню = [
            ("main", "Главная", "🏠"),
            ("clean", "Очистка", "🧹"),
            ("perf", "Производительность", "⚡"),
            ("advanced", "Расширенные", "🔥"),
            ("power", "Питание", "🔌"),
            ("nvidia", "NVIDIA", "🎮"),
            ("processes", "Процессы", "📊"),
            ("autostart", "Автозагрузка", "🚀"),
            ("profiles", "Профили игр", "💾"),
            ("info", "О системе", "ℹ️"),
            ("settings", "Настройки", "⚙️"),
        ]

        for key, name, icon in self.меню:
            btn = ui.SidebarButton(
                self.sidebar, name, icon=icon,
                command=lambda k=key: self.показать_раздел(k)
            )
            btn.pack(fill="x", padx=14, pady=2)
            self.активные_кнопки[key] = btn

        # Версия внизу
        ctk.CTkLabel(
            self.sidebar, text="v2.1 • Premium",
            font=ctk.CTkFont(size=10),
            text_color=ui.COLORS["text_dim"]
        ).pack(side="bottom", pady=15)

        # ============================================================
        # ПРАВАЯ ЧАСТЬ (КОНТЕНТ)
        # ============================================================
        self.content = ctk.CTkFrame(
            self, fg_color=ui.COLORS["bg_dark"], corner_radius=0
        )
        self.content.pack(side="right", fill="both", expand=True)

        # Верхняя панель
        self.topbar = ctk.CTkFrame(
            self.content, height=70, fg_color="transparent"
        )
        self.topbar.pack(fill="x", padx=25, pady=(20, 5))
        self.topbar.pack_propagate(False)

        self.section_title = ctk.CTkLabel(
            self.topbar, text="Главная",
            font=ctk.CTkFont(size=24, weight="bold"),
            text_color=ui.COLORS["text"], anchor="w"
        )
        self.section_title.pack(side="left", pady=15)

        ctk.CTkButton(
            self.topbar, text="💾 Сохранить отчёт",
            width=150, height=32,
            fg_color=ui.COLORS["bg_card"],
            hover_color=ui.COLORS["bg_hover"],
            border_width=1, border_color=ui.COLORS["border"],
            text_color=ui.COLORS["text"],
            font=ctk.CTkFont(size=12),
            command=self.сохранить_отчёт
        ).pack(side="right", pady=15)

        # Контейнер для раздела
        self.section_container = ctk.CTkFrame(
            self.content, fg_color="transparent"
        )
        self.section_container.pack(
            fill="both", expand=True, padx=25, pady=(0, 20)
        )

        # Статус-бар
        self.status = ctk.CTkLabel(
            self.content, text="Готов к работе",
            font=ctk.CTkFont(size=11),
            text_color=ui.COLORS["text_dim"],
            anchor="w"
        )
        self.status.pack(fill="x", padx=25, pady=(0, 15))

        # Создаём все разделы
        self.создать_разделы()

        # Показываем главную
        self.показать_раздел("main")

    # ============================================================
    # СОЗДАНИЕ РАЗДЕЛОВ
    # ============================================================
    def создать_разделы(self):
        """Создаёт все разделы один раз, чтобы сохранять состояние."""
        self.разделы["main"] = self.создать_главную()
        self.разделы["clean"] = self.создать_очистку()
        self.разделы["perf"] = self.создать_производительность()
        self.разделы["nvidia"] = self.создать_nvidia()
        self.разделы["processes"] = self.создать_процессы()
        self.разделы["autostart"] = self.создать_автозагрузку()
        self.разделы["profiles"] = self.создать_профили()
        self.разделы["info"] = self.создать_о_системе()
        self.разделы["advanced"] = self.создать_расширенные()
        self.разделы["power"] = self.создать_питание()
        self.разделы["settings"] = self.создать_настройки()

    def показать_раздел(self, ключ):
        for frame in self.разделы.values():
            frame.pack_forget()

        self.разделы[ключ].pack(fill="both", expand=True)

        for k, btn in self.активные_кнопки.items():
            btn.set_active(k == ключ)

        названия = {
            "main": "🏠  Главная",
            "clean": "🧹  Очистка системы",
            "perf": "⚡  Производительность",
            "advanced": "🔥  Расширенные твики",
            "power": "🔌  Электропитание",
            "nvidia": "🎮  Настройки NVIDIA",
            "processes": "📊  Процессы",
            "autostart": "🚀  Автозагрузка",
            "profiles": "💾  Профили игр",
            "info": "ℹ️  О системе",
            "settings": "⚙️  Настройки",
        }
        self.section_title.configure(text=названия.get(ключ, ""))

        self.текущий_раздел = ключ
        self.анимация_раздела()

        if ключ == "processes":
            self.обновить_процессы()
        elif ключ == "autostart":
            self.обновить_автозагрузку()
        elif ключ == "nvidia":
            self.обновить_nvidia_инфо()
        elif ключ == "profiles":
            self.обновить_профили()
        elif ключ == "power":
            self.обновить_статус_автозапуска()

    def анимация_раздела(self):
        if not self.settings.get("animations", True):
            return
        frame = self.разделы.get(self.текущий_раздел)
        if not frame:
            return

        карточки = [w for w in frame.winfo_children()]
        for i, card in enumerate(карточки[:8]):
            try:
                card.configure(fg_color=ui.COLORS["bg_dark"])
                self.after(40 * i, lambda c=card: c.configure(fg_color=ui.COLORS["bg_card"]))
            except Exception:
                pass

    # ============================================================
    # РАЗДЕЛ: ГЛАВНАЯ
    # ============================================================
    def создать_главную(self):
        frame = ctk.CTkFrame(self.section_container, fg_color="transparent")

        stats_row = ctk.CTkFrame(frame, fg_color="transparent")
        stats_row.pack(fill="x", pady=(0, 15))

        self.card_cpu = ui.StatCard(stats_row, "ЦП", "🖥️", ui.COLORS["accent"])
        self.card_cpu.pack(side="left", fill="both", expand=True, padx=(0, 8))

        self.card_ram = ui.StatCard(stats_row, "ОЗУ", "📊", ui.COLORS["success"])
        self.card_ram.pack(side="left", fill="both", expand=True, padx=8)

        self.card_disk = ui.StatCard(stats_row, "Диск C", "💾", ui.COLORS["warning"])
        self.card_disk.pack(side="left", fill="both", expand=True, padx=(8, 0))

        self.btn_all = ctk.CTkButton(
            frame,
            text="🔥  ПРИМЕНИТЬ ВСЁ ДЛЯ FPS",
            font=ctk.CTkFont(size=16, weight="bold"),
            fg_color=ui.COLORS["danger"],
            hover_color="#dc2626",
            height=55, corner_radius=12,
            command=self.применить_всё
        )
        self.btn_all.pack(fill="x", pady=8)

        quick = ui.Card(frame, title="⚡ Быстрые действия")
        quick.pack(fill="both", expand=True, pady=(10, 0))

        btn_frame = ctk.CTkFrame(quick, fg_color="transparent")
        btn_frame.pack(fill="both", expand=True, padx=16, pady=(0, 16))

        quick_actions = [
            ("🧠 Очистить оперативную память", adv.clean_ram, "success"),
            ("🎮 Отключить Game DVR", opt.disable_game_dvr, "success"),
            ("⚡ Максимальная производительность", opt.enable_ultimate_performance, "default"),
            ("🗑️ Очистить корзину", adv.empty_recycle_bin, "default"),
        ]
        for text, func, style in quick_actions:
            ui.ActionButton(
                btn_frame, text, style=style,
                command=lambda f=func: self.выполнить(f)
            ).pack(fill="x", pady=4)

        return frame

    # ============================================================
    # РАЗДЕЛ: ОЧИСТКА
    # ============================================================
    def создать_очистку(self):
        frame = ctk.CTkFrame(self.section_container, fg_color="transparent")

        card = ui.Card(frame, title="🧹 Инструменты очистки")
        card.pack(fill="both", expand=True)

        actions = [
            ("🧹 Очистить временные файлы", opt.clean_temp_files),
            ("🗑️ Очистить Prefetch", opt.clean_prefetch),
            ("🌐 Очистить кэш DNS", opt.flush_dns),
            ("💾 Очистить кэш обновлений Windows", opt.clean_windows_update_cache),
            ("📋 Очистить журналы событий", opt.clear_event_logs),
            ("💿 Выполнить TRIM для SSD", opt.trim_ssd),
            ("🗑️ Очистить корзину", adv.empty_recycle_bin),
            ("🌐 Очистить кэш браузеров", adv.clean_browser_cache),
        ]
        inner = ctk.CTkFrame(card, fg_color="transparent")
        inner.pack(fill="both", expand=True, padx=16, pady=(0, 16))
        for text, func in actions:
            ui.ActionButton(
                inner, text,
                command=lambda f=func: self.выполнить(f)
            ).pack(fill="x", pady=4)

        return frame

    # ============================================================
    # РАЗДЕЛ: ПРОИЗВОДИТЕЛЬНОСТЬ
    # ============================================================
    def создать_производительность(self):
        frame = ctk.CTkFrame(self.section_container, fg_color="transparent")

        card = ui.Card(frame, title="⚡ Оптимизация системы")
        card.pack(fill="both", expand=True)

        actions = [
            ("🎮 Отключить Game DVR / Game Bar", opt.disable_game_dvr),
            ("⚡ Включить макс. производительность", opt.enable_ultimate_performance),
            ("🚫 Отключить лишние службы", opt.disable_unused_services),
            ("🖼️ Отключить визуальные эффекты", opt.disable_visual_effects),
            ("🖱️ Отключить акселерацию мыши", opt.disable_mouse_acceleration),
            ("🔊 Снизить задержку аудио", opt.reduce_audio_latency),
            ("🚫 Отключить фоновые приложения", opt.disable_background_apps),
            ("🖥️ Отключить оптимизации полного экрана", opt.disable_fullscreen_optimizations),
            ("💤 Отключить гибернацию", opt.disable_hibernation),
            ("🔧 Применить системные твики Windows", opt.apply_system_gaming_tweaks),
            ("🌐 Отключить Nagle (снижение пинга)", opt.disable_nagle),
        ]
        inner = ctk.CTkFrame(card, fg_color="transparent")
        inner.pack(fill="both", expand=True, padx=16, pady=(0, 16))
        for text, func in actions:
            ui.ActionButton(
                inner, text,
                command=lambda f=func: self.выполнить(f)
            ).pack(fill="x", pady=4)

        return frame

    # ============================================================
    # РАЗДЕЛ: РАСШИРЕННЫЕ ТВИКИ
    # ============================================================
    def создать_расширенные(self):
        frame = ctk.CTkFrame(self.section_container, fg_color="transparent")

        card1 = ui.Card(frame, title="🧠 Очистка и восстановление")
        card1.pack(fill="x", pady=(0, 10))
        inner1 = ctk.CTkFrame(card1, fg_color="transparent")
        inner1.pack(fill="x", padx=16, pady=(0, 16))

        ui.ActionButton(
            inner1, "🧠 Очистить оперативную память (RAM)",
            style="success",
            command=lambda: self.выполнить(adv.clean_ram)
        ).pack(fill="x", pady=4)

        ui.ActionButton(
            inner1, "🗑️ Очистить корзину",
            command=lambda: self.выполнить(adv.empty_recycle_bin)
        ).pack(fill="x", pady=4)

        ui.ActionButton(
            inner1, "🌐 Очистить кэш браузеров",
            command=lambda: self.выполнить(adv.clean_browser_cache)
        ).pack(fill="x", pady=4)

        ui.ActionButton(
            inner1, "🛡️ Создать точку восстановления",
            style="warning",
            command=lambda: self.выполнить(adv.create_restore_point)
        ).pack(fill="x", pady=4)

        ui.ActionButton(
            inner1, "✅ Включить защиту системы",
            command=lambda: self.выполнить(adv.enable_system_protection)
        ).pack(fill="x", pady=4)

        card2 = ui.Card(frame, title="🔒 Приватность и обновления")
        card2.pack(fill="x", pady=(0, 10))
        inner2 = ctk.CTkFrame(card2, fg_color="transparent")
        inner2.pack(fill="x", padx=16, pady=(0, 16))

        ui.ActionButton(
            inner2, "🛑 Отключить Windows Update",
            style="danger",
            command=lambda: self.выполнить(adv.disable_windows_update)
        ).pack(fill="x", pady=4)

        ui.ActionButton(
            inner2, "✅ Включить Windows Update",
            style="success",
            command=lambda: self.выполнить(adv.enable_windows_update)
        ).pack(fill="x", pady=4)

        ui.ActionButton(
            inner2, "🚫 Полностью отключить телеметрию",
            command=lambda: self.выполнить(adv.disable_telemetry_full)
        ).pack(fill="x", pady=4)

        ui.ActionButton(
            inner2, "🖥️ Оптимизировать Проводник и Пуск",
            command=lambda: self.выполнить(adv.optimize_explorer)
        ).pack(fill="x", pady=4)

        return frame

    # ============================================================
    # РАЗДЕЛ: ПИТАНИЕ
    # ============================================================
    def создать_питание(self):
        frame = ctk.CTkFrame(self.section_container, fg_color="transparent")

        card = ui.Card(frame, title="🔌 Настройки электропитания")
        card.pack(fill="x", pady=(0, 10))

        inner = ctk.CTkFrame(card, fg_color="transparent")
        inner.pack(fill="x", padx=16, pady=(0, 16))

        ctk.CTkLabel(
            inner,
            text="• Отключает USB selective suspend\n"
                 "• Отключает PCIe ASPM (энергосбережение шины)\n"
                 "• Фиксирует частоту ЦП на 100%\n"
                 "• Отключает спящий режим диска\n"
                 "• Отключает гибернацию",
            font=ctk.CTkFont(size=11),
            text_color=ui.COLORS["text_dim"],
            anchor="w", justify="left"
        ).pack(fill="x", pady=(0, 10))

        ui.ActionButton(
            inner, "🔌  Применить макс. производительность",
            style="danger",
            command=lambda: self.выполнить(adv.apply_power_tweaks)
        ).pack(fill="x", pady=4)

        card2 = ui.Card(frame, title="⏰ Автозапуск приложения")
        card2.pack(fill="x", pady=(10, 10))

        inner2 = ctk.CTkFrame(card2, fg_color="transparent")
        inner2.pack(fill="x", padx=16, pady=(0, 16))

        self.autostart_status = ctk.CTkLabel(
            inner2, text="", font=ctk.CTkFont(size=12),
            text_color=ui.COLORS["text_dim"], anchor="w"
        )
        self.autostart_status.pack(fill="x", pady=(0, 8))

        btn_row = ctk.CTkFrame(inner2, fg_color="transparent")
        btn_row.pack(fill="x", pady=4)

        ui.ActionButton(
            btn_row, "✅ Включить",
            style="success",
            command=self.включить_автозапуск
        ).pack(side="left", fill="x", expand=True, padx=(0, 4))

        ui.ActionButton(
            btn_row, "❌ Отключить",
            style="danger",
            command=self.отключить_автозапуск
        ).pack(side="left", fill="x", expand=True, padx=(4, 0))

        card3 = ui.Card(frame, title="🔔 Уведомления и звуки")
        card3.pack(fill="x")

        inner3 = ctk.CTkFrame(card3, fg_color="transparent")
        inner3.pack(fill="x", padx=16, pady=(0, 16))

        self.sounds_switch = ctk.CTkSwitch(
            inner3, text="Проигрывать звук после оптимизации",
            font=ctk.CTkFont(size=12),
            text_color=ui.COLORS["text"],
            progress_color=ui.COLORS["accent"],
            command=self.переключить_звуки
        )
        self.sounds_switch.pack(fill="x", pady=5)
        if self.settings.get("sounds", True):
            self.sounds_switch.select()

        self.toast_switch = ctk.CTkSwitch(
            inner3, text="Показывать уведомления Windows",
            font=ctk.CTkFont(size=12),
            text_color=ui.COLORS["text"],
            progress_color=ui.COLORS["accent"],
            command=self.переключить_toast
        )
        self.toast_switch.pack(fill="x", pady=5)
        if self.settings.get("toasts", True):
            self.toast_switch.select()

        return frame

    def обновить_статус_автозапуска(self):
        try:
            if adv.is_app_in_autostart():
                self.autostart_status.configure(
                    text="🟢 Автозапуск включён", text_color=ui.COLORS["success"]
                )
            else:
                self.autostart_status.configure(
                    text="🔴 Автозапуск выключен", text_color=ui.COLORS["danger"]
                )
        except Exception:
            pass

    def включить_автозапуск(self):
        ok, msg = adv.enable_autostart_app()
        self.установить_статус(msg, ui.COLORS["success"] if ok else ui.COLORS["danger"])
        self.обновить_статус_автозапуска()

    def отключить_автозапуск(self):
        ok, msg = adv.disable_autostart_app()
        self.установить_статус(msg, ui.COLORS["success"] if ok else ui.COLORS["danger"])
        self.обновить_статус_автозапуска()

    def переключить_звуки(self):
        val = bool(self.sounds_switch.get())
        self.settings["sounds"] = val
        st.save_settings(self.settings)
        self.установить_статус(f"Звуки: {'вкл' if val else 'выкл'}", ui.COLORS["success"])

    def переключить_toast(self):
        val = bool(self.toast_switch.get())
        self.settings["toasts"] = val
        st.save_settings(self.settings)
        self.установить_статус(f"Уведомления: {'вкл' if val else 'выкл'}", ui.COLORS["success"])

    # ============================================================
    # РАЗДЕЛ: NVIDIA
    # ============================================================
    def создать_nvidia(self):
        frame = ctk.CTkFrame(self.section_container, fg_color="transparent")

        self.gpu_card = ui.Card(frame, title="🎮 Ваша видеокарта")
        self.gpu_card.pack(fill="x", pady=(0, 15))

        self.gpu_info_frame = ctk.CTkFrame(self.gpu_card, fg_color="transparent")
        self.gpu_info_frame.pack(fill="x", padx=16, pady=(0, 16))

        profiles_card = ui.Card(frame, title="⚙️ Профили производительности")
        profiles_card.pack(fill="x", pady=(0, 15))

        inner = ctk.CTkFrame(profiles_card, fg_color="transparent")
        inner.pack(fill="x", padx=16, pady=(0, 16))

        ui.ActionButton(
            inner, "🚀  Режим максимальной производительности",
            style="danger",
            command=lambda: self.выполнить(nv.apply_max_performance_profile)
        ).pack(fill="x", pady=4)

        ui.ActionButton(
            inner, "⚡  Профиль минимальной задержки (для шутеров)",
            style="warning",
            command=lambda: self.выполнить(nv.apply_low_latency_profile)
        ).pack(fill="x", pady=4)

        ui.ActionButton(
            inner, "🎨  Профиль максимального качества",
            command=lambda: self.выполнить(nv.apply_quality_profile)
        ).pack(fill="x", pady=4)

        panel_card = ui.Card(frame, title="🛠️ Открыть панели NVIDIA")
        panel_card.pack(fill="x")

        panel_inner = ctk.CTkFrame(panel_card, fg_color="transparent")
        panel_inner.pack(fill="x", padx=16, pady=(0, 16))

        ui.ActionButton(
            panel_inner, "🎮  Открыть NVIDIA App",
            command=lambda: self.выполнить(nv.open_nvidia_app)
        ).pack(fill="x", pady=4)

        ui.ActionButton(
            panel_inner, "⚙️  Открыть Панель управления NVIDIA",
            command=lambda: self.выполнить(nv.open_control_panel)
        ).pack(fill="x", pady=4)

        ui.ActionButton(
            panel_inner, "🚀  Включить HAGS (аппаратное планирование GPU)",
            command=lambda: self.выполнить(opt.enable_hags)
        ).pack(fill="x", pady=4)

        return frame

    def обновить_nvidia_инфо(self):
        for w in self.gpu_info_frame.winfo_children():
            w.destroy()

        try:
            info = nv.get_gpu_info()
        except Exception:
            info = {"name": "NVIDIA GPU не найден"}

        rows = [
            ("Название", info.get("name", "—")),
            ("Драйвер", info.get("driver", "—")),
            ("Видеопамять", info.get("vram", "—")),
            ("Температура", info.get("temp", "—")),
            ("Загрузка GPU", info.get("utilization", "—")),
            ("Потребление", info.get("power", "—")),
        ]

        for label, value in rows:
            row = ctk.CTkFrame(self.gpu_info_frame, fg_color="transparent")
            row.pack(fill="x", pady=3)

            ctk.CTkLabel(
                row, text=label + ":",
                font=ctk.CTkFont(size=12, weight="bold"),
                text_color=ui.COLORS["text_dim"],
                width=140, anchor="w"
            ).pack(side="left")

            ctk.CTkLabel(
                row, text=str(value),
                font=ctk.CTkFont(size=12),
                text_color=ui.COLORS["text"],
                anchor="w"
            ).pack(side="left", fill="x", expand=True)

    # ============================================================
    # РАЗДЕЛ: ПРОЦЕССЫ
    # ============================================================
    def создать_процессы(self):
        frame = ctk.CTkFrame(self.section_container, fg_color="transparent")

        card = ui.Card(frame, title="📊 Активные процессы")
        card.pack(fill="both", expand=True)

        top = ctk.CTkFrame(card, fg_color="transparent")
        top.pack(fill="x", padx=16, pady=(0, 8))

        ctk.CTkButton(
            top, text="🔄 Обновить", width=120, height=30,
            fg_color=ui.COLORS["accent"],
            hover_color=ui.COLORS["accent_hover"],
            command=self.обновить_процессы
        ).pack(side="left")

        self.proc_count = ctk.CTkLabel(
            top, text="", text_color=ui.COLORS["text_dim"],
            font=ctk.CTkFont(size=11)
        )
        self.proc_count.pack(side="right")

        self.proc_scroll = ctk.CTkScrollableFrame(
            card, fg_color=ui.COLORS["bg_dark"], corner_radius=8
        )
        self.proc_scroll.pack(fill="both", expand=True, padx=16, pady=(0, 16))

        return frame

    def обновить_процессы(self):
        for w in self.proc_scroll.winfo_children():
            w.destroy()

        try:
            procs = ex.get_top_processes(limit=30)
        except Exception:
            procs = []

        self.proc_count.configure(text=f"Показано: {len(procs)}")

        for p in procs:
            row = ctk.CTkFrame(
                self.proc_scroll, fg_color=ui.COLORS["bg_card"], corner_radius=6
            )
            row.pack(fill="x", pady=2, padx=4)

            ctk.CTkLabel(
                row, text=p["name"][:40],
                font=ctk.CTkFont(size=12),
                text_color=ui.COLORS["text"],
                width=300, anchor="w"
            ).pack(side="left", padx=10, pady=8)

            ctk.CTkLabel(
                row, text=f"{p['memory_mb']:.0f} МБ",
                font=ctk.CTkFont(size=11),
                text_color=ui.COLORS["text_dim"],
                width=100, anchor="w"
            ).pack(side="left")

            ctk.CTkLabel(
                row, text=f"PID {p['pid']}",
                font=ctk.CTkFont(size=11),
                text_color=ui.COLORS["text_dim"],
                width=80, anchor="w"
            ).pack(side="left")

            ctk.CTkButton(
                row, text="Завершить", width=100, height=26,
                fg_color=ui.COLORS["danger"],
                hover_color="#dc2626",
                font=ctk.CTkFont(size=11),
                command=lambda pid=p["pid"], n=p["name"]: self.завершить(pid, n)
            ).pack(side="right", padx=10, pady=6)

    def завершить(self, pid, name):
        if not messagebox.askyesno("Подтверждение", f"Завершить «{name}»?"):
            return
        ok, msg = ex.kill_process(pid)
        self.установить_статус(msg, ui.COLORS["success"] if ok else ui.COLORS["danger"])
        self.обновить_процессы()

    # ============================================================
    # РАЗДЕЛ: АВТОЗАГРУЗКА
    # ============================================================
    def создать_автозагрузку(self):
        frame = ctk.CTkFrame(self.section_container, fg_color="transparent")

        card = ui.Card(frame, title="🚀 Программы в автозагрузке")
        card.pack(fill="both", expand=True)

        top = ctk.CTkFrame(card, fg_color="transparent")
        top.pack(fill="x", padx=16, pady=(0, 8))

        ctk.CTkButton(
            top, text="🔄 Обновить", width=120, height=30,
            fg_color=ui.COLORS["accent"],
            hover_color=ui.COLORS["accent_hover"],
            command=self.обновить_автозагрузку
        ).pack(side="left")

        self.auto_count = ctk.CTkLabel(
            top, text="", text_color=ui.COLORS["text_dim"],
            font=ctk.CTkFont(size=11)
        )
        self.auto_count.pack(side="right")

        self.auto_scroll = ctk.CTkScrollableFrame(
            card, fg_color=ui.COLORS["bg_dark"], corner_radius=8
        )
        self.auto_scroll.pack(fill="both", expand=True, padx=16, pady=(0, 16))

        return frame

    def обновить_автозагрузку(self):
        for w in self.auto_scroll.winfo_children():
            w.destroy()

        try:
            items = ex.get_autostart_items()
        except Exception:
            items = []

        self.auto_count.configure(text=f"Найдено: {len(items)}")

        if not items:
            ctk.CTkLabel(
                self.auto_scroll, text="Автозагрузка пуста",
                text_color=ui.COLORS["text_dim"]
            ).pack(pady=30)
            return

        for name, path, reg_path, desc, hive in items:
            row = ctk.CTkFrame(
                self.auto_scroll, fg_color=ui.COLORS["bg_card"], corner_radius=6
            )
            row.pack(fill="x", pady=2, padx=4)

            info = ctk.CTkFrame(row, fg_color="transparent")
            info.pack(side="left", fill="x", expand=True, padx=10, pady=8)

            ctk.CTkLabel(
                info, text=name,
                font=ctk.CTkFont(size=12, weight="bold"),
                text_color=ui.COLORS["text"], anchor="w"
            ).pack(anchor="w")

            ctk.CTkLabel(
                info, text=path[:70] + ("..." if len(path) > 70 else ""),
                font=ctk.CTkFont(size=10),
                text_color=ui.COLORS["text_dim"], anchor="w"
            ).pack(anchor="w")

            ctk.CTkButton(
                row, text="Удалить", width=90, height=26,
                fg_color=ui.COLORS["danger"],
                hover_color="#dc2626",
                font=ctk.CTkFont(size=11),
                command=lambda n=name, p=reg_path, h=hive: self.удалить_авто(n, p, h)
            ).pack(side="right", padx=10)

    def удалить_авто(self, name, path, hive):
        if not messagebox.askyesno("Подтверждение", f"Удалить «{name}» из автозагрузки?"):
            return
        ok, msg = ex.remove_autostart_item(name, path, hive)
        self.установить_статус(msg, ui.COLORS["success"] if ok else ui.COLORS["danger"])
        self.обновить_автозагрузку()

    # ============================================================
    # РАЗДЕЛ: ПРОФИЛИ ИГР
    # ============================================================
    def создать_профили(self):
        frame = ctk.CTkFrame(self.section_container, fg_color="transparent")

        list_card = ui.Card(frame, title="💾 Сохранённые профили")
        list_card.pack(fill="both", expand=True, pady=(0, 15))

        self.profiles_scroll = ctk.CTkScrollableFrame(
            list_card, fg_color=ui.COLORS["bg_dark"], corner_radius=8
        )
        self.profiles_scroll.pack(fill="both", expand=True, padx=16, pady=(0, 16))

        add_card = ui.Card(frame, title="➕ Добавить новый профиль")
        add_card.pack(fill="x")

        inner = ctk.CTkFrame(add_card, fg_color="transparent")
        inner.pack(fill="x", padx=16, pady=(0, 16))

        self.prof_name = ctk.CTkEntry(
            inner, placeholder_text="Имя профиля (например: CS2)",
            height=36, corner_radius=8,
            fg_color=ui.COLORS["bg_dark"],
            border_color=ui.COLORS["border"]
        )
        self.prof_name.pack(fill="x", pady=4)

        self.prof_path = ctk.CTkEntry(
            inner, placeholder_text="Путь к .exe игры",
            height=36, corner_radius=8,
            fg_color=ui.COLORS["bg_dark"],
            border_color=ui.COLORS["border"]
        )
        self.prof_path.pack(fill="x", pady=4)

        btn_row = ctk.CTkFrame(inner, fg_color="transparent")
        btn_row.pack(fill="x", pady=4)

        ctk.CTkButton(
            btn_row, text="📁 Выбрать .exe", height=34,
            fg_color=ui.COLORS["bg_hover"],
            hover_color=ui.COLORS["bg_card"],
            command=self.выбрать_exe
        ).pack(side="left", fill="x", expand=True, padx=(0, 4))

        ctk.CTkButton(
            btn_row, text="💾 Сохранить", height=34,
            fg_color=ui.COLORS["success"],
            hover_color="#059669",
            command=self.сохранить_профиль
        ).pack(side="left", fill="x", expand=True, padx=(4, 0))

        return frame

    def выбрать_exe(self):
        path = filedialog.askopenfilename(
            title="Выберите .exe игры",
            filetypes=[("Исполняемые", "*.exe"), ("Все файлы", "*.*")]
        )
        if path:
            self.prof_path.delete(0, "end")
            self.prof_path.insert(0, path)

    def сохранить_профиль(self):
        name = self.prof_name.get().strip()
        path = self.prof_path.get().strip()
        if not name:
            self.установить_статус("Введите имя профиля", ui.COLORS["danger"])
            return
        ok, msg = ex.add_profile(name, path, ["Max Performance", "Low Latency"])
        self.установить_статус(msg, ui.COLORS["success"] if ok else ui.COLORS["danger"])
        self.prof_name.delete(0, "end")
        self.prof_path.delete(0, "end")
        self.обновить_профили()

    def обновить_профили(self):
        for w in self.profiles_scroll.winfo_children():
            w.destroy()

        try:
            profiles = ex.load_profiles()
        except Exception:
            profiles = {}

        if not profiles:
            ctk.CTkLabel(
                self.profiles_scroll,
                text="Профилей пока нет. Добавьте первый ниже.",
                text_color=ui.COLORS["text_dim"]
            ).pack(pady=30)
            return

        for name, data in profiles.items():
            row = ctk.CTkFrame(
                self.profiles_scroll, fg_color=ui.COLORS["bg_card"], corner_radius=6
            )
            row.pack(fill="x", pady=2, padx=4)

            info = ctk.CTkFrame(row, fg_color="transparent")
            info.pack(side="left", fill="x", expand=True, padx=10, pady=8)

            ctk.CTkLabel(
                info, text=f"🎮 {name}",
                font=ctk.CTkFont(size=12, weight="bold"),
                text_color=ui.COLORS["text"], anchor="w"
            ).pack(anchor="w")

            ctk.CTkLabel(
                info, text=data.get("exe_path", "—")[:70],
                font=ctk.CTkFont(size=10),
                text_color=ui.COLORS["text_dim"], anchor="w"
            ).pack(anchor="w")

            ctk.CTkButton(
                row, text="🗑️", width=40, height=26,
                fg_color=ui.COLORS["danger"],
                hover_color="#dc2626",
                command=lambda n=name: self.удалить_профиль(n)
            ).pack(side="right", padx=10)

    def удалить_профиль(self, name):
        if not messagebox.askyesno("Подтверждение", f"Удалить профиль «{name}»?"):
            return
        ok, msg = ex.remove_profile(name)
        self.установить_статус(msg, ui.COLORS["success"] if ok else ui.COLORS["danger"])
        self.обновить_профили()

    # ============================================================
    # РАЗДЕЛ: О СИСТЕМЕ
    # ============================================================
    def создать_о_системе(self):
        frame = ctk.CTkFrame(self.section_container, fg_color="transparent")

        card = ui.Card(frame, title="ℹ️ Информация о системе")
        card.pack(fill="both", expand=True)

        info = ex.get_system_info()
        перевод = {
            "os": "Операционная система",
            "версия": "Версия сборки",
            "процессор": "Процессор",
            "ядра": "Ядра",
            "частота": "Частота ЦП",
            "ОЗУ": "Оперативная память",
            "имя ПК": "Имя компьютера",
        }

        inner = ctk.CTkFrame(card, fg_color="transparent")
        inner.pack(fill="both", expand=True, padx=16, pady=(0, 16))

        for key, value in info.items():
            row = ctk.CTkFrame(
                inner, fg_color=ui.COLORS["bg_dark"], corner_radius=6
            )
            row.pack(fill="x", pady=3)

            ctk.CTkLabel(
                row, text=перевод.get(key, key) + ":",
                font=ctk.CTkFont(size=12, weight="bold"),
                text_color=ui.COLORS["text_dim"],
                width=200, anchor="w"
            ).pack(side="left", padx=12, pady=10)

            ctk.CTkLabel(
                row, text=str(value),
                font=ctk.CTkFont(size=12),
                text_color=ui.COLORS["text"], anchor="w"
            ).pack(side="left", fill="x", expand=True)

        return frame

    # ============================================================
    # РАЗДЕЛ: НАСТРОЙКИ
    # ============================================================
    def создать_настройки(self):
        frame = ctk.CTkFrame(self.section_container, fg_color="transparent")

        card = ui.Card(frame, title="⚙️ Настройки приложения")
        card.pack(fill="both", expand=True)

        inner = ctk.CTkFrame(card, fg_color="transparent")
        inner.pack(fill="both", expand=True, padx=16, pady=(0, 16))

        ctk.CTkLabel(
            inner, text="Тема оформления:",
            font=ctk.CTkFont(size=12, weight="bold"),
            text_color=ui.COLORS["text"], anchor="w"
        ).pack(fill="x", pady=(5, 3))

        self.theme_var = ctk.StringVar(value=self.settings["appearance"])
        ctk.CTkOptionMenu(
            inner, values=["dark", "light", "system"],
            variable=self.theme_var, command=self.сменить_тему,
            height=34, fg_color=ui.COLORS["bg_dark"],
            button_color=ui.COLORS["accent"]
        ).pack(fill="x", pady=3)

        ctk.CTkLabel(
            inner, text="Акцентный цвет (перезапуск):",
            font=ctk.CTkFont(size=12, weight="bold"),
            text_color=ui.COLORS["text"], anchor="w"
        ).pack(fill="x", pady=(15, 3))

        self.color_var = ctk.StringVar(value=self.settings["accent_color"])
        ctk.CTkOptionMenu(
            inner, values=["blue", "green", "dark-blue"],
            variable=self.color_var, command=self.сменить_цвет,
            height=34, fg_color=ui.COLORS["bg_dark"],
            button_color=ui.COLORS["accent"]
        ).pack(fill="x", pady=3)

        self.anim_switch = ctk.CTkSwitch(
            inner, text="Плавные анимации",
            font=ctk.CTkFont(size=12),
            text_color=ui.COLORS["text"],
            progress_color=ui.COLORS["accent"],
            command=self.переключить_анимации
        )
        self.anim_switch.pack(fill="x", pady=(20, 5))
        if self.settings["animations"]:
            self.anim_switch.select()

        self.stats_switch = ctk.CTkSwitch(
            inner, text="Показывать мониторинг",
            font=ctk.CTkFont(size=12),
            text_color=ui.COLORS["text"],
            progress_color=ui.COLORS["accent"],
            command=self.переключить_мониторинг
        )
        self.stats_switch.pack(fill="x", pady=5)
        if self.settings["show_stats"]:
            self.stats_switch.select()

        self.confirm_switch = ctk.CTkSwitch(
            inner, text="Подтверждение перед «ПРИМЕНИТЬ ВСЁ»",
            font=ctk.CTkFont(size=12),
            text_color=ui.COLORS["text"],
            progress_color=ui.COLORS["accent"],
            command=self.переключить_подтверждение
        )
        self.confirm_switch.pack(fill="x", pady=5)
        if self.settings["confirm_all"]:
            self.confirm_switch.select()

        ctk.CTkButton(
            inner, text="🔄 Сбросить настройки",
            height=36, fg_color=ui.COLORS["bg_hover"],
            hover_color=ui.COLORS["bg_card"],
            command=self.сбросить_настройки
        ).pack(fill="x", pady=(20, 5))

        return frame

    def сменить_тему(self, value):
        self.settings["appearance"] = value
        st.save_settings(self.settings)
        ctk.set_appearance_mode(value)
        self.установить_статус(f"Тема: {value}", ui.COLORS["success"])

    def сменить_цвет(self, value):
        self.settings["accent_color"] = value
        st.save_settings(self.settings)
        self.установить_статус("Цвет сохранён (перезапуск)", ui.COLORS["warning"])

    def переключить_анимации(self):
        val = bool(self.anim_switch.get())
        self.settings["animations"] = val
        st.save_settings(self.settings)
        self.установить_статус(f"Анимации: {'вкл' if val else 'выкл'}", ui.COLORS["success"])

    def переключить_мониторинг(self):
        val = bool(self.stats_switch.get())
        self.settings["show_stats"] = val
        st.save_settings(self.settings)
        self.установить_статус(f"Мониторинг: {'вкл' if val else 'выкл'}", ui.COLORS["success"])

    def переключить_подтверждение(self):
        val = bool(self.confirm_switch.get())
        self.settings["confirm_all"] = val
        st.save_settings(self.settings)
        self.установить_статус(f"Подтверждение: {'вкл' if val else 'выкл'}", ui.COLORS["success"])

    def сбросить_настройки(self):
        self.settings = st.reset_settings()
        ctk.set_appearance_mode(self.settings["appearance"])
        self.установить_статус("Настройки сброшены", ui.COLORS["warning"])

    # ============================================================
    # ЛОГИКА
    # ============================================================
    def установить_статус(self, текст, цвет=None):
        self.status.configure(
            text=текст,
            text_color=цвет or ui.COLORS["text_dim"]
        )

    def выполнить(self, функция):
        self.установить_статус("Выполняется...", ui.COLORS["warning"])

        def работник():
            try:
                ok, msg = функция()
                цвет = ui.COLORS["success"] if ok else ui.COLORS["danger"]
                self.after(0, lambda: self.установить_статус(msg, цвет))
                self.last_results.append(f"{'✅' if ok else '❌'} {msg}")

                if self.settings.get("sounds", True):
                    nt.play_sound("success" if ok else "error")
            except Exception as e:
                self.after(0, lambda: self.установить_статус(f"Ошибка: {e}", ui.COLORS["danger"]))

        threading.Thread(target=работник, daemon=True).start()

    def применить_всё(self):
        if self.settings.get("confirm_all", True):
            if not messagebox.askyesno(
                "Подтверждение",
                "Применить все оптимизации?\n\n"
                "Будет отключена телеметрия, Game DVR, некоторые службы.\n"
                "Рекомендуется создать точку восстановления."
            ):
                return

        задачи = [
            opt.clean_temp_files, opt.clean_prefetch, opt.flush_dns,
            opt.clean_windows_update_cache, opt.disable_game_dvr,
            opt.enable_ultimate_performance, opt.disable_nagle,
            opt.disable_unused_services, opt.disable_visual_effects,
            opt.disable_mouse_acceleration, opt.reduce_audio_latency,
            opt.disable_background_apps, opt.disable_fullscreen_optimizations,
            opt.apply_system_gaming_tweaks, opt.enable_hags,
            adv.clean_ram, adv.empty_recycle_bin,
        ]

        self.установить_статус("Применяю все твики...", ui.COLORS["warning"])
        self.last_results = []

        def работник():
            результаты = []
            for t in задачи:
                try:
                    ok, msg = t()
                    результаты.append(f"{'✅' if ok else '❌'} {msg}")
                except Exception as e:
                    результаты.append(f"❌ {e}")
            self.last_results = результаты
            финал = " | ".join(результаты)
            self.after(0, lambda: self.установить_статус(финал[:180], ui.COLORS["success"]))

            if self.settings.get("sounds", True):
                nt.play_sound("success")
            if self.settings.get("toasts", True):
                nt.show_toast("PC Optimizer", f"Оптимизация завершена ({len(результаты)} твиков)")

        threading.Thread(target=работник, daemon=True).start()

    def сохранить_отчёт(self):
        if not self.last_results:
            self.установить_статус("Сначала примените оптимизации", ui.COLORS["warning"])
            return
        путь = filedialog.asksaveasfilename(
            title="Сохранить отчёт",
            defaultextension=".txt",
            filetypes=[("Текстовые файлы", "*.txt")],
            initialfile=f"отчёт_{os.environ.get('COMPUTERNAME', 'ПК')}.txt"
        )
        if not путь:
            return
        ok, msg = ex.export_report(self.last_results, путь)
        self.установить_статус(msg, ui.COLORS["success"] if ok else ui.COLORS["danger"])

    def update_stats(self):
        try:
            stats = opt.get_stats()
            self.card_cpu.update_value(
                f"{stats['cpu']:.0f}%", stats["cpu"] / 100,
                f"{stats['cpu']:.1f}% загрузки"
            )
            self.card_ram.update_value(
                f"{stats['ram_percent']:.0f}%", stats["ram_percent"] / 100,
                f"{stats['ram_used_gb']:.1f} / {stats['ram_total_gb']:.1f} ГБ"
            )
            self.card_disk.update_value(
                f"{stats['disk_percent']:.0f}%", stats["disk_percent"] / 100,
                f"свободно {stats['disk_free_gb']:.1f} ГБ"
            )
        except Exception:
            pass
        self.after(2000, self.update_stats)


if __name__ == "__main__":
    app = Приложение()
    app.mainloop()
from PIL import Image, ImageDraw
import os


def create_icon(output_path="icon.ico"):
    size = 256
    img = Image.new("RGBA", (size, size), (0, 0, 0, 0))
    draw = ImageDraw.Draw(img)

    # Градиент: тёмно-синий → голубой
    for y in range(size):
        ratio = y / size
        r = int(15 + (59 - 15) * ratio)
        g = int(23 + (130 - 23) * ratio)
        b = int(42 + (246 - 42) * ratio)
        draw.line([(0, y), (size, y)], fill=(r, g, b, 255))

    # Скруглённые углы
    mask = Image.new("L", (size, size), 0)
    mask_draw = ImageDraw.Draw(mask)
    mask_draw.rounded_rectangle([(0, 0), (size, size)], radius=48, fill=255)

    # Молния
    lightning = [
        (148, 30), (85, 140), (125, 140),
        (108, 226), (170, 115), (130, 115),
    ]
    draw.polygon(lightning, fill=(255, 255, 255, 255))

    # Маска
    result = Image.new("RGBA", (size, size), (0, 0, 0, 0))
    result.paste(img, (0, 0), mask)

    # Все размеры для Windows
    sizes = [(16, 16), (32, 32), (48, 48), (64, 64), (128, 128), (256, 256)]
    result.save(output_path, format="ICO", sizes=sizes)
    print(f"Готово: {os.path.abspath(output_path)}")


if __name__ == "__main__":
    try:
        create_icon()
    except ImportError:
        print("Нужна библиотека Pillow: pip install Pillow")
"""Система уведомлений и звуков."""
import os
import sys
import winsound
import threading

# Звуки Windows
SOUND_SUCCESS = "SystemAsterisk"
SOUND_WARNING = "SystemExclamation"
SOUND_ERROR = "SystemHand"


def play_sound(kind: str = "success") -> None:
    """Проигрывает системный звук."""
    sound_map = {
        "success": SOUND_SUCCESS,
        "warning": SOUND_WARNING,
        "error": SOUND_ERROR,
    }
    sound = sound_map.get(kind, SOUND_SUCCESS)
    try:
        import winsound
        # Используем алиас системного звука
        winsound.PlaySound(sound, winsound.SND_ALIAS | winsound.SND_ASYNC)
    except Exception:
        pass


def show_toast(title: str, message: str) -> None:
    """Показывает уведомление в системном трее (Windows 10/11)."""
    def _show():
        try:
            # PowerShell + BurntToast или нативный toast
            ps = f'''
            [Windows.UI.Notifications.ToastNotificationManager, Windows.UI.Notifications, ContentType = WindowsRuntime] | Out-Null
            [Windows.Data.Xml.Dom.XmlDocument, Windows.Data.Xml.Dom.XmlDocument, ContentType = WindowsRuntime] | Out-Null
            $template = @"
            <toast>
                <visual>
                    <binding template="ToastGeneric">
                        <text>{title}</text>
                        <text>{message}</text>
                    </binding>
                </visual>
            </toast>
"@
            $xml = New-Object Windows.Data.Xml.Dom.XmlDocument
            $xml.LoadXml($template)
            $toast = [Windows.UI.Notifications.ToastNotification]::new($xml)
            [Windows.UI.Notifications.ToastNotificationManager]::CreateToastNotifier("PC Optimizer").Show($toast)
            '''
            import subprocess
            subprocess.Popen(
                ["powershell", "-WindowStyle", "Hidden", "-Command", ps],
                creationflags=0x08000000  # CREATE_NO_WINDOW
            )
        except Exception:
            pass

    threading.Thread(target=_show, daemon=True).start()
"""
Модуль работы с NVIDIA GPU.
Определяет видеокарту, читает состояние, применяет профили производительности.
"""
import os
import subprocess
import winreg


def is_nvidia_installed() -> bool:
    """Проверяет, есть ли NVIDIA GPU в системе."""
    ok, out = run_cmd("nvidia-smi --query-gpu=name --format=csv,noheader")
    return ok and out.strip() != ""


def get_gpu_info() -> dict:
    """Собирает информацию о NVIDIA GPU."""
    info = {
        "name": "—",
        "driver": "—",
        "vram": "—",
        "temp": "—",
        "utilization": "—",
        "power": "—",
    }
    ok, out = run_cmd("nvidia-smi --query-gpu=name,driver_version,memory.total,temperature.gpu,utilization.gpu,power.draw --format=csv,noheader,nounits")
    if ok and out.strip():
        parts = [p.strip() for p in out.strip().split(",")]
        if len(parts) >= 6:
            info["name"] = parts[0]
            info["driver"] = parts[1]
            info["vram"] = f"{parts[2]} МБ"
            info["temp"] = f"{parts[3]}°C"
            info["utilization"] = f"{parts[4]}%"
            info["power"] = f"{parts[5]} Вт"
    return info


def run_cmd(cmd: str) -> tuple:
    """Запускает команду и возвращает результат."""
    try:
        result = subprocess.run(
            cmd, shell=True, capture_output=True, text=True,
            encoding="cp866", errors="ignore"
        )
        if result.returncode == 0:
            return True, result.stdout.strip()
        return False, result.stderr.strip() or result.stdout.strip()
    except Exception as e:
        return False, str(e)


# ============================================================
# ПРИМЕНЕНИЕ ПРОФИЛЕЙ NVIDIA ЧЕРЕЗ РЕЕСТР
# ============================================================

def apply_max_performance_profile() -> tuple:
    """
    Устанавливает режим 'Максимальная производительность' для NVIDIA.
    Отключает энергосбережение GPU.
    """
    results = []

    # 1. Режим управления электропитанием — максимальная производительность
    try:
        key_path = r"SYSTEM\CurrentControlSet\Control\Class\{4d36e968-e325-11ce-bfc1-08002be10318}\0000"
        with winreg.OpenKey(winreg.HKEY_LOCAL_MACHINE, key_path, 0, winreg.KEY_SET_VALUE) as key:
            # Отключает динамическое переключение частот
            winreg.SetValueEx(key, "PowerMizerEnable", 0, winreg.REG_DWORD, 1)
            winreg.SetValueEx(key, "PowerMizerLevel", 0, winreg.REG_DWORD, 1)
            winreg.SetValueEx(key, "PowerMizerLevelAC", 0, winreg.REG_DWORD, 1)
            # Максимальная производительность
            winreg.SetValueEx(key, "PerfLevelSrc", 0, winreg.REG_DWORD, 0x2222)
        results.append("PowerMizer: макс. производительность")
    except Exception as e:
        results.append(f"PowerMizer: ошибка ({e})")

    # 2. Отключение энергосбережения PCI Express
    try:
        run_cmd("powercfg /setacvalueindex SCHEME_CURRENT SUB_PCIEXPRESS ASPM 0")
        run_cmd("powercfg /setactive SCHEME_CURRENT")
        results.append("PCIe ASPM: выкл")
    except Exception as e:
        results.append(f"PCIe ASPM: ошибка ({e})")

    return True, " | ".join(results)


def apply_low_latency_profile() -> tuple:
    """
    Настраивает минимальную задержку для игр.
    """
    results = []

    # 1. Отключение потоковой оптимизации (Threaded Optimization)
    try:
        key_path = r"SYSTEM\CurrentControlSet\Control\Class\{4d36e968-e325-11ce-bfc1-08002be10318}\0000"
        with winreg.OpenKey(winreg.HKEY_LOCAL_MACHINE, key_path, 0, winreg.KEY_SET_VALUE) as key:
            # Отключает VSync
            winreg.SetValueEx(key, "VSync", 0, winreg.REG_DWORD, 0)
            # Тройная буферизация — выкл
            winreg.SetValueEx(key, "TripleBuffer", 0, winreg.REG_DWORD, 0)
            # Потоковая оптимизация — авто
            winreg.SetValueEx(key, "ThreadedOptimization", 0, winreg.REG_DWORD, 0)
        results.append("VSync/TripleBuffer: выкл")
    except Exception as e:
        results.append(f"Ошибка: {e}")

    return True, " | ".join(results)


def apply_quality_profile() -> tuple:
    """
    Настраивает максимальное качество графики.
    """
    try:
        key_path = r"SYSTEM\CurrentControlSet\Control\Class\{4d36e968-e325-11ce-bfc1-08002be10318}\0000"
        with winreg.OpenKey(winreg.HKEY_LOCAL_MACHINE, key_path, 0, winreg.KEY_SET_VALUE) as key:
            # Качество текстур — высокое
            winreg.SetValueEx(key, "TextureQuality", 0, winreg.REG_DWORD, 0)
            # Сглаживание — улучшенное
            winreg.SetValueEx(key, "AntialiasingMode", 0, winreg.REG_DWORD, 1)
        return True, "Профиль качества применён"
    except Exception as e:
        return False, f"Ошибка: {e}"


# ============================================================
# ОТКРЫТИЕ ПАНЕЛИ УПРАВЛЕНИЯ NVIDIA
# ============================================================

def open_control_panel() -> tuple:
    """Открывает классическую панель управления NVIDIA."""
    try:
        subprocess.Popen(["control.exe", "/name", "Microsoft.NVIDIAControlPanel"])
        return True, "Панель управления NVIDIA открыта"
    except Exception:
        paths = [
            r"C:\Program Files\NVIDIA Corporation\Control Panel Client\nvcplui.exe",
            r"C:\Windows\System32\nvcplui.exe",
        ]
        for p in paths:
            if os.path.exists(p):
                subprocess.Popen([p])
                return True, "Панель управления NVIDIA открыта"
        return False, "Панель управления NVIDIA не найдена"


def open_nvidia_app() -> tuple:
    """Открывает новый NVIDIA App."""
    try:
        os.startfile("nvidiaapp:")
        return True, "NVIDIA App открыт"
    except Exception:
        paths = [
            r"C:\Program Files\NVIDIA Corporation\NVIDIA App\CEF\NVIDIA App.exe",
            r"C:\Program Files\NVIDIA Corporation\NVIDIA App\NVIDIA App.exe",
        ]
        for p in paths:
            if os.path.exists(p):
                subprocess.Popen([p])
                return True, "NVIDIA App открыт"
        return False, "NVIDIA App не установлен"


# ============================================================
# ИНФОРМАЦИЯ О ДРАЙВЕРЕ
# ============================================================

def get_driver_version() -> str:
    """Возвращает версию драйвера NVIDIA."""
    ok, out = run_cmd("nvidia-smi --query-gpu=driver_version --format=csv,noheader")
    return out.strip() if ok else "неизвестно"
import os
import subprocess
import tempfile
import psutil


def is_admin() -> bool:
    """Проверка прав администратора."""
    try:
        import ctypes
        return ctypes.windll.shell32.IsUserAnAdmin() != 0
    except Exception:
        return False


def run_cmd(cmd: str) -> tuple[bool, str]:
    """Запуск команды с проверкой результата."""
    try:
        result = subprocess.run(
            cmd, shell=True, capture_output=True, text=True, encoding="cp866"
        )
        if result.returncode == 0:
            return True, result.stdout.strip()
        return False, result.stderr.strip() or result.stdout.strip()
    except Exception as e:
        return False, str(e)


# ---------- ОЧИСТКА ----------

def clean_temp_files() -> tuple[bool, str]:
    paths = [
        tempfile.gettempdir(),
        os.path.join(os.environ.get("SystemRoot", "C:\\Windows"), "Temp"),
        os.path.join(os.environ.get("LOCALAPPDATA", ""), "Temp"),
    ]
    freed = 0
    errors = 0
    for path in paths:
        if not os.path.exists(path):
            continue
        for root, dirs, files in os.walk(path):
            for f in files:
                fp = os.path.join(root, f)
                try:
                    freed += os.path.getsize(fp)
                    os.remove(fp)
                except Exception:
                    errors += 1
    mb = freed / (1024 * 1024)
    return True, f"Освобождено ~{mb:.1f} МБ (ошибок: {errors})"


def clean_prefetch() -> tuple[bool, str]:
    path = os.path.join(os.environ.get("SystemRoot", "C:\\Windows"), "Prefetch")
    if not os.path.exists(path):
        return False, "Папка Prefetch не найдена"
    errors = 0
    for f in os.listdir(path):
        try:
            os.remove(os.path.join(path, f))
        except Exception:
            errors += 1
    return True, f"Prefetch очищен (ошибок: {errors})"


def flush_dns() -> tuple[bool, str]:
    """Очистка кэша DNS для ускорения сетевых подключений."""
    ok, msg = run_cmd("ipconfig /flushdns")
    return ok, "Кэш DNS очищен" if ok else msg


def clean_windows_update_cache() -> tuple[bool, str]:
    """Очистка кэша обновлений Windows (освобождает ГБ)."""
    ok, msg = run_cmd("Dism.exe /online /Cleanup-Image /StartComponentCleanup")
    return ok, "Кэш обновлений Windows очищен" if ok else msg


# ---------- ИГРОВЫЕ ТВИКИ ----------

def disable_game_dvr() -> tuple[bool, str]:
    cmds = [
        'reg add "HKCU\\System\\GameConfigStore" /v GameDVR_Enabled /t REG_DWORD /d 0 /f',
        'reg add "HKCU\\Software\\Microsoft\\Windows\\CurrentVersion\\GameDVR" '
        '/v AppCaptureEnabled /t REG_DWORD /d 0 /f',
        'reg add "HKLM\\SOFTWARE\\Policies\\Microsoft\\Windows\\GameDVR" '
        '/v AllowGameDVR /t REG_DWORD /d 0 /f',
    ]
    for c in cmds:
        ok, msg = run_cmd(c)
        if not ok:
            return False, msg
    return True, "Game DVR и Game Bar отключены"


def enable_ultimate_performance() -> tuple[bool, str]:
    guid = "e9a42b02-d5df-448d-aa00-03f14749eb61"
    run_cmd(f"powercfg -duplicatescheme {guid}")
    ok, msg = run_cmd(f"powercfg /setactive {guid}")
    if ok:
        return True, "Схема 'Максимальная производительность' активна"
    return False, msg


def disable_nagle() -> tuple[bool, str]:
    ok, out = run_cmd(
        'reg query "HKLM\\SYSTEM\\CurrentControlSet\\Services\\Tcpip\\Parameters\\Interfaces"'
    )
    if not ok:
        return False, out
    keys = [line.split()[-1] for line in out.splitlines() if "Interfaces\\" in line]
    for k in keys:
        run_cmd(f'reg add "{k}" /v TcpAckFrequency /t REG_DWORD /d 1 /f')
        run_cmd(f'reg add "{k}" /v TCPNoDelay /t REG_DWORD /d 1 /f')
        run_cmd(f'reg add "{k}" /v TcpDelAckTicks /t REG_DWORD /d 0 /f')
    return True, "Nagle отключён (нужна перезагрузка)"


def disable_unused_services() -> tuple[bool, str]:
    services = ["DiagTrack", "dmwappushservice", "SysMain", "WSearch"]
    results = []
    for s in services:
        ok, msg = run_cmd(f"sc stop {s} & sc config {s} start=disabled")
        results.append(f"{s}: {'OK' if ok else 'ошибка'}")
    return True, " | ".join(results)


# ---------- НОВЫЕ ФУНКЦИИ: NVIDIA И СИСТЕМА ----------

def open_nvidia_app() -> tuple[bool, str]:
    """Открывает NVIDIA App для ручной настройки графики."""
    try:
        os.startfile("nvidiaapp:")
        return True, "NVIDIA App открыт. Настройте 'Программные настройки' вручную."
    except Exception:
        try:
            paths = [
                r"C:\Program Files\NVIDIA Corporation\NVIDIA App\CEF\NVIDIA App.exe",
                r"C:\Program Files\NVIDIA Corporation\NVIDIA App\NVIDIA App.exe",
            ]
            for p in paths:
                if os.path.exists(p):
                    subprocess.Popen([p])
                    return True, "NVIDIA App открыт."
            return False, "NVIDIA App не найден. Установите его из Microsoft Store."
        except Exception as e:
            return False, f"Ошибка: {e}"


def open_nvidia_control_panel() -> tuple[bool, str]:
    """Открывает классическую Панель управления NVIDIA."""
    try:
        subprocess.Popen(["control.exe", "/name", "Microsoft.NVIDIAControlPanel"])
        return True, "Панель управления NVIDIA открыта."
    except Exception:
        try:
            paths = [
                r"C:\Program Files\NVIDIA Corporation\Control Panel Client\nvcplui.exe",
                r"C:\Windows\System32\nvcplui.exe",
            ]
            for p in paths:
                if os.path.exists(p):
                    subprocess.Popen([p])
                    return True, "Панель управления NVIDIA открыта."
            return False, "Панель управления NVIDIA не найдена. Проверьте драйверы."
        except Exception as e:
            return False, f"Ошибка: {e}"


def apply_system_gaming_tweaks() -> tuple[bool, str]:
    """Применяет системные твики Windows для игр."""
    results = []

    # 1. Отключить визуальные эффекты
    try:
        run_cmd(
            'reg add "HKCU\\Software\\Microsoft\\Windows\\CurrentVersion\\Explorer\\VisualEffects" '
            '/v VisualFXSetting /t REG_DWORD /d 2 /f'
        )
        results.append("Визуальные эффекты: выкл")
    except Exception:
        results.append("Визуальные эффекты: ошибка")

    # 2. Отключить Power Throttling
    try:
        run_cmd(
            'reg add "HKLM\\SYSTEM\\CurrentControlSet\\Control\\Power\\PowerThrottling" '
            '/v PowerThrottlingOff /t REG_DWORD /d 1 /f'
        )
        results.append("Power Throttling: выкл")
    except Exception:
        results.append("Power Throttling: ошибка")

    # 3. Оптимизация Multimedia SystemProfile для игр
    try:
        base_key = "HKLM\\SOFTWARE\\Microsoft\\Windows NT\\CurrentVersion\\Multimedia\\SystemProfile"
        run_cmd(f'reg add "{base_key}" /v SystemResponsiveness /t REG_DWORD /d 0 /f')
        run_cmd(f'reg add "{base_key}" /v NetworkThrottlingIndex /t REG_DWORD /d 0xffffffff /f')

        games_key = f"{base_key}\\Tasks\\Games"
        run_cmd(f'reg add "{games_key}" /v "GPU Priority" /t REG_DWORD /d 8 /f')
        run_cmd(f'reg add "{games_key}" /v "Priority" /t REG_DWORD /d 6 /f')
        run_cmd(f'reg add "{games_key}" /v "Scheduling Category" /t REG_SZ /d "High" /f')
        results.append("Multimedia: оптимизировано")
    except Exception:
        results.append("Multimedia: ошибка")

    # 4. Включить Game Mode
    try:
        run_cmd(
            'reg add "HKCU\\Software\\Microsoft\\GameBar" '
            '/v AutoGameModeEnabled /t REG_DWORD /d 1 /f'
        )
        results.append("Game Mode: вкл")
    except Exception:
        results.append("Game Mode: ошибка")

    return True, " | ".join(results)


# ---------- НОВЫЕ ФУНКЦИИ ОТ ДРУГИХ ОПТИМИЗАТОРОВ ----------

def disable_visual_effects() -> tuple[bool, str]:
    """Отключение визуальных эффектов для повышения FPS."""
    cmds = [
        'reg add "HKCU\\Software\\Microsoft\\Windows\\CurrentVersion\\Explorer\\VisualEffects" /v VisualFXSetting /t REG_DWORD /d 2 /f',
        'reg add "HKCU\\Control Panel\\Desktop" /v UserPreferencesMask /t REG_BINARY /d 9012038010000000 /f',
        'reg add "HKCU\\Control Panel\\Desktop\\WindowMetrics" /v MinAnimate /t REG_SZ /d 0 /f',
    ]
    for c in cmds:
        run_cmd(c)
    return True, "Визуальные эффекты отключены"


def disable_mouse_acceleration() -> tuple[bool, str]:
    """Отключение акселерации мыши (улучшение точности)."""
    run_cmd('reg add "HKCU\\Control Panel\\Mouse" /v MouseSpeed /t REG_SZ /d 0 /f')
    run_cmd('reg add "HKCU\\Control Panel\\Mouse" /v MouseThreshold1 /t REG_SZ /d 0 /f')
    run_cmd('reg add "HKCU\\Control Panel\\Mouse" /v MouseThreshold2 /t REG_SZ /d 0 /f')
    return True, "Акселерация мыши отключена"


def reduce_audio_latency() -> tuple[bool, str]:
    """Снижение задержки аудио для игр и стриминга."""
    base = "HKLM\\SOFTWARE\\Microsoft\\Windows NT\\CurrentVersion\\Multimedia\\SystemProfile"
    run_cmd(f'reg add "{base}\\Tasks\\Audio" /v "GPU Priority" /t REG_DWORD /d 8 /f')
    run_cmd(f'reg add "{base}\\Tasks\\Audio" /v "Priority" /t REG_DWORD /d 6 /f')
    run_cmd(f'reg add "{base}\\Tasks\\Audio" /v "Scheduling Category" /t REG_SZ /d "High" /f')
    return True, "Задержка аудио снижена"


def disable_background_apps() -> tuple[bool, str]:
    """Отключение фоновых приложений для освобождения ресурсов."""
    run_cmd(
        'reg add "HKCU\\Software\\Microsoft\\Windows\\CurrentVersion\\BackgroundAccessApplications" '
        '/v GlobalUserDisabled /t REG_DWORD /d 1 /f'
    )
    run_cmd(
        'reg add "HKLM\\SOFTWARE\\Policies\\Microsoft\\Windows\\AppPrivacy" '
        '/v LetAppsRunInBackground /t REG_DWORD /d 2 /f'
    )
    return True, "Фоновые приложения отключены"


def disable_fullscreen_optimizations() -> tuple[bool, str]:
    """Отключение оптимизаций полноэкранного режима (снижает задержку ввода)."""
    cmds = [
        'reg add "HKCU\\System\\GameConfigStore" /v GameDVR_FSEBehavior /t REG_DWORD /d 2 /f',
        'reg add "HKCU\\System\\GameConfigStore" /v GameDVR_FSEBehaviorMode /t REG_DWORD /d 2 /f',
        'reg add "HKCU\\System\\GameConfigStore" /v GameDVR_HonorUserFSEBehaviorMode /t REG_DWORD /d 1 /f',
        'reg add "HKCU\\System\\GameConfigStore" /v GameDVR_DXGIHonorFSEWindowsCompatible /t REG_DWORD /d 1 /f',
        'reg add "HKCU\\System\\GameConfigStore" /v GameDVR_EFSEFeatureFlags /t REG_DWORD /d 0 /f',
    ]
    for c in cmds:
        run_cmd(c)
    return True, "Оптимизации полного экрана отключены (нужна перезагрузка)"


def disable_hibernation() -> tuple[bool, str]:
    """Отключение гибернации (освобождает место на диске)."""
    ok, msg = run_cmd("powercfg -h off")
    return ok, "Гибернация отключена" if ok else msg


def enable_hags() -> tuple[bool, str]:
    """Включение планирования GPU с аппаратным ускорением (HAGS)."""
    try:
        run_cmd(
            'reg add "HKLM\\SYSTEM\\CurrentControlSet\\Control\\GraphicsDrivers" '
            '/v HwSchMode /t REG_DWORD /d 2 /f'
        )
        return True, "HAGS включён (нужна перезагрузка)"
    except Exception as e:
        return False, f"Ошибка: {e}"


def clear_event_logs() -> tuple[bool, str]:
    """Очистка журналов событий Windows (освобождает место)."""
    logs = ["Application", "System", "Security", "Setup"]
    for log in logs:
        run_cmd(f'wevtutil cl {log}')
    return True, "Журналы событий очищены"


def trim_ssd() -> tuple[bool, str]:
    """Оптимизация SSD через TRIM."""
    ok, msg = run_cmd("defrag C: /L")
    return ok, "TRIM выполнен" if ok else msg


# ---------- МОНИТОРИНГ ----------

def get_stats() -> dict:
    """Текущая загрузка системы."""
    cpu = psutil.cpu_percent(interval=0.5)
    ram = psutil.virtual_memory()
    disk = psutil.disk_usage("C:\\")
    return {
        "cpu": cpu,
        "ram_percent": ram.percent,
        "ram_used_gb": ram.used / (1024 ** 3),
        "ram_total_gb": ram.total / (1024 ** 3),
        "disk_percent": disk.percent,
        "disk_free_gb": disk.free / (1024 ** 3),
    }
# -*- mode: python ; coding: utf-8 -*-


a = Analysis(
    ['main.py'],
    pathex=[],
    binaries=[],
    datas=[],
    hiddenimports=[],
    hookspath=[],
    hooksconfig={},
    runtime_hooks=[],
    excludes=[],
    noarchive=False,
    optimize=0,
)
pyz = PYZ(a.pure)

exe = EXE(
    pyz,
    a.scripts,
    [],
    exclude_binaries=True,
    name='PC_Optimizer',
    debug=False,
    bootloader_ignore_signals=False,
    strip=False,
    upx=True,
    console=False,
    disable_windowed_traceback=False,
    argv_emulation=False,
    target_arch=None,
    codesign_identity=None,
    entitlements_file=None,
    uac_admin=True,
    icon=['icon.ico'],
)
coll = COLLECT(
    exe,
    a.binaries,
    a.datas,
    strip=False,
    upx=True,
    upx_exclude=[],
    name='PC_Optimizer',
)

========================================
  PC OPTIMIZER — УСКОРЕНИЕ ДЛЯ ИГР
========================================

Версия: 2.0
Разработчик: PC Optimizer Team

----------------------------------------
ЧТО ДЕЛАЕТ ПРОГРАММА
----------------------------------------
PC Optimizer оптимизирует Windows для игр и работы:
• Отключает Game DVR / Game Bar (даёт +FPS)
• Включает максимальную производительность
• Очищает временные файлы и кэш
• Настраивает NVIDIA GPU
• Управляет автозагрузкой и процессами
• Сохраняет профили игр

----------------------------------------
КАК ПОЛЬЗОВАТЬСЯ
----------------------------------------
1. Запустите приложение от имени администратора
   (правой кнопкой → «Запуск от имени администратора»)

2. Откройте раздел «Главная» и нажмите
   «ПРИМЕНИТЬ ВСЁ ДЛЯ FPS» для базовой оптимизации.

3. Для тонкой настройки используйте разделы:
   🧹 Очистка
   ⚡ Производительность
   🎮 NVIDIA
   📊 Процессы
   🚀 Автозагрузка

4. Перезагрузите компьютер после применения твиков.

----------------------------------------
ВАЖНО
----------------------------------------
• ПЕРЕД первым использованием создайте точку
  восстановления системы:
  Win+R → sysdm.cpl → Защита системы → Создать

• Не завершайте системные процессы
  (explorer.exe, svchost.exe, lsass.exe и др.)

• Некоторые твики можно откатить через
  точку восстановления Windows.

----------------------------------------
УДАЛЕНИЕ
----------------------------------------
Панель управления → Программы → PC Optimizer → Удалить

Или: Параметры → Приложения → PC Optimizer → Удалить

----------------------------------------
ЛИЦЕНЗИЯ
----------------------------------------
Бесплатное использование.
Автор не несёт ответственности за возможный ущерб.
Используйте на свой риск.

========================================
import json
import os

SETTINGS_FILE = os.path.join(os.path.dirname(os.path.abspath(__file__)), "settings.json")

DEFAULT_SETTINGS = {
    "appearance": "dark",         # dark / light / system
    "accent_color": "blue",       # blue / green / dark-blue
    "window_size": "large",       # small / medium / large
    "animations": True,           # вкл/выкл анимации
    "animation_speed": 250,       # мс
    "show_stats": True,           # показывать мониторинг
    "confirm_all": True,          # подтверждение при "ПРИМЕНИТЬ ВСЁ"
}


def load_settings() -> dict:
    """Загрузить настройки из файла."""
    if not os.path.exists(SETTINGS_FILE):
        return DEFAULT_SETTINGS.copy()
    try:
        with open(SETTINGS_FILE, "r", encoding="utf-8") as f:
            data = json.load(f)
        # Добавляем отсутствующие ключи
        for k, v in DEFAULT_SETTINGS.items():
            data.setdefault(k, v)
        return data
    except Exception:
        return DEFAULT_SETTINGS.copy()


def save_settings(settings: dict) -> bool:
    """Сохранить настройки в файл."""
    try:
        with open(SETTINGS_FILE, "w", encoding="utf-8") as f:
            json.dump(settings, f, indent=2, ensure_ascii=False)
        return True
    except Exception:
        return False


def reset_settings() -> dict:
    """Сброс к настройкам по умолчанию."""
    save_settings(DEFAULT_SETTINGS)
    return DEFAULT_SETTINGS.copy()
"""Вспомогательные UI-компоненты для современного интерфейса."""
import customtkinter as ctk


# Цветовая схема (тёмная, реалистичная)
COLORS = {
    "bg_dark": "#0f1117",
    "bg_sidebar": "#151823",
    "bg_card": "#1c2030",
    "bg_hover": "#232839",
    "accent": "#3b82f6",
    "accent_hover": "#2563eb",
    "success": "#10b981",
    "warning": "#f59e0b",
    "danger": "#ef4444",
    "text": "#e5e7eb",
    "text_dim": "#9ca3af",
    "border": "#2d3347",
}


class SidebarButton(ctk.CTkButton):
    """Кнопка бокового меню с подсветкой активного состояния."""

    def __init__(self, parent, text, command=None, icon=""):
        super().__init__(
            parent,
            text=f"  {icon}  {text}",
            anchor="w",
            height=44,
            corner_radius=8,
            fg_color="transparent",
            hover_color=COLORS["bg_hover"],
            text_color=COLORS["text_dim"],
            font=ctk.CTkFont(size=13),
            command=command,
        )
        self._active = False

    def set_active(self, active: bool):
        self._active = active
        if active:
            self.configure(
                fg_color=COLORS["accent"],
                text_color="#ffffff",
                font=ctk.CTkFont(size=13, weight="bold"),
            )
        else:
            self.configure(
                fg_color="transparent",
                text_color=COLORS["text_dim"],
                font=ctk.CTkFont(size=13),
            )


class Card(ctk.CTkFrame):
    """Карточка с закруглёнными углами и заголовком."""

    def __init__(self, parent, title="", **kwargs):
        super().__init__(
            parent,
            fg_color=COLORS["bg_card"],
            corner_radius=12,
            border_width=1,
            border_color=COLORS["border"],
            **kwargs,
        )
        if title:
            self.title_label = ctk.CTkLabel(
                self,
                text=title,
                font=ctk.CTkFont(size=14, weight="bold"),
                text_color=COLORS["text"],
                anchor="w",
            )
            self.title_label.pack(fill="x", padx=16, pady=(12, 8))


class ActionButton(ctk.CTkButton):
    """Кнопка действия в карточке."""

    def __init__(self, parent, text, command=None, style="default"):
        colors = {
            "default": (COLORS["accent"], COLORS["accent_hover"]),
            "success": (COLORS["success"], "#059669"),
            "warning": (COLORS["warning"], "#d97706"),
            "danger": (COLORS["danger"], "#dc2626"),
        }
        fg, hover = colors.get(style, colors["default"])
        super().__init__(
            parent,
            text=text,
            anchor="w",
            height=38,
            corner_radius=8,
            fg_color=fg,
            hover_color=hover,
            text_color="#ffffff",
            font=ctk.CTkFont(size=12),
            command=command,
        )


class StatCard(ctk.CTkFrame):
    """Карточка статистики (ЦП / ОЗУ / Диск)."""

    def __init__(self, parent, title, icon="", accent_color=None):
        super().__init__(
            parent,
            fg_color=COLORS["bg_card"],
            corner_radius=10,
            border_width=1,
            border_color=COLORS["border"],
        )
        self.accent = accent_color or COLORS["accent"]

        # Заголовок
        header = ctk.CTkFrame(self, fg_color="transparent")
        header.pack(fill="x", padx=14, pady=(10, 5))

        ctk.CTkLabel(
            header, text=icon, font=ctk.CTkFont(size=16), text_color=self.accent
        ).pack(side="left")

        ctk.CTkLabel(
            header, text=title, font=ctk.CTkFont(size=12, weight="bold"),
            text_color=COLORS["text"], anchor="w"
        ).pack(side="left", padx=(6, 0))

        # Значение
        self.value_label = ctk.CTkLabel(
            self, text="—", font=ctk.CTkFont(size=22, weight="bold"),
            text_color=COLORS["text"], anchor="w"
        )
        self.value_label.pack(fill="x", padx=14)

        # Подпись
        self.sub_label = ctk.CTkLabel(
            self, text="", font=ctk.CTkFont(size=10),
            text_color=COLORS["text_dim"], anchor="w"
        )
        self.sub_label.pack(fill="x", padx=14, pady=(0, 5))

        # Прогресс-бар
        self.progress = ctk.CTkProgressBar(
            self, height=5, corner_radius=3,
            progress_color=self.accent,
            fg_color=COLORS["bg_hover"],
        )
        self.progress.pack(fill="x", padx=14, pady=(0, 12))
        self.progress.set(0)

    def update_value(self, value_text, progress_ratio, sub_text=""):
        self.value_label.configure(text=value_text)
        self.progress.set(max(0.0, min(1.0, progress_ratio)))
        if sub_text:
            self.sub_label.configure(text=sub_text)


def make_section_header(parent, text, icon=""):
    """Заголовок раздела."""
    return ctk.CTkLabel(
        parent,
        text=f"{icon}  {text}" if icon else text,
        font=ctk.CTkFont(size=15, weight="bold"),
        text_color=COLORS["text"],
        anchor="w",
    )
