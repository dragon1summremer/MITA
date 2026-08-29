import json
import os
import queue
import random
import re
import time
import string
import subprocess
import threading
import webbrowser
import winreg
import numpy as np
import pyautogui
import sherpa_onnx
import sounddevice as sd
import pygetwindow as gw
import psutil
import keyboard
import ctypes
import sys

# Загрузка soundfile при наличии
try:
    import soundfile as sf

    HAS_SOUNDFILE = True
except ImportError:
    HAS_SOUNDFILE = False

# Настройки PyAutoGUI
pyautogui.FAILSAFE = False
pyautogui.PAUSE = 0.05


# Проверка прав администратора
def is_admin():
    try:
        return ctypes.windll.shell32.IsUserAnAdmin()
    except:
        return False


if not is_admin():
    print("⚠️ ВНИМАНИЕ: Скрипт запущен без прав администратора!")
    print("Некоторые команды могут не работать в защищенных приложениях.")
    print("Рекомендуется перезапустить скрипт от имени администратора.\n")
    # Можно автоматически перезапустить с правами админа
    # ctypes.windll.shell32.ShellExecuteW(None, "runas", sys.executable, " ".join(sys.argv), None, 1)

# Словарь стоп-слов и мата
STOP_WORDS = [
    "нахуй", "нахуй", "блядь", "блять", "сука", "сук", "хуй", "хую",
    "пизда", "пиздец", "ебан", "ебать", "заеб", "пидор", "гандон",
    "мудак", "уебок", "тупой", "дебил", "идиот", "кретин", "олень"
]

BAD_WORDS = [
    "нахуй", "блядь", "блять", "сука", "хуй", "пизда", "пиздец",
    "ебан", "ебать", "заеб", "пидор", "гандон", "мудак", "уебок",
    "тупой", "дебил", "идиот", "кретин", "олень", "козел", "козёл",
    "лох", "лошара", "чмо", "шлюха", "курва", "бля", "нах", "хер"
]


# ============================================================
# АВТОМАТИЧЕСКИЙ ПОИСК ПАПКИ MitaAIShka
# ============================================================

def get_available_drives() -> list[str]:
    """Возвращает список доступных дисков"""
    drives = []
    for letter in string.ascii_uppercase:
        drive_path = f"{letter}:\\"
        if os.path.exists(drive_path):
            drives.append(drive_path)
    return drives


def find_mita_folder():
    """Автоматически ищет папку MitaAIShka на дисках"""
    possible_paths = [
        r"C:\AI MITA\MitaAIShka",
        r"D:\AI MITA\MitaAIShka",
        r"E:\AI MITA\MitaAIShka",
        r"C:\Program Files\AI MITA\MitaAIShka",
        r"C:\Program Files (x86)\AI MITA\MitaAIShka",
        r"C:\Users\{}\Documents\AI MITA\MitaAIShka".format(os.getlogin()),
        r"C:\Users\{}\Desktop\AI MITA\MitaAIShka".format(os.getlogin()),
        os.path.join(os.getcwd(), "MitaAIShka"),
        os.path.join(os.getcwd(), "AI MITA", "MitaAIShka"),
    ]

    for drive in get_available_drives():
        possible_paths.append(os.path.join(drive, "AI MITA", "MitaAIShka"))
        possible_paths.append(os.path.join(drive, "MitaAIShka"))
        possible_paths.append(os.path.join(drive, "Users", os.getlogin(), "Documents", "AI MITA", "MitaAIShka"))
        possible_paths.append(os.path.join(drive, "Users", os.getlogin(), "Desktop", "AI MITA", "MitaAIShka"))

    print("🔍 Ищу папку MitaAIShka...")
    for path in possible_paths:
        if os.path.exists(path):
            model_file = os.path.join(path, "model.int8.onnx")
            tokens_file = os.path.join(path, "tokens.txt")
            if os.path.exists(model_file) and os.path.exists(tokens_file):
                print(f"✓ Найдена папка MitaAIShka: {path}")
                return path

    print("🔍 Расширенный поиск MitaAIShka...")
    search_roots = []
    for drive in get_available_drives():
        search_roots.append(os.path.join(drive, "Program Files"))
        search_roots.append(os.path.join(drive, "Program Files (x86)"))
        search_roots.append(os.path.join(drive, "Users", os.getlogin()))

    for root in search_roots:
        if not os.path.exists(root):
            continue
        for dirpath, dirnames, _ in os.walk(root):
            if "MitaAIShka" in dirnames:
                test_path = os.path.join(dirpath, "MitaAIShka")
                model_file = os.path.join(test_path, "model.int8.onnx")
                tokens_file = os.path.join(test_path, "tokens.txt")
                if os.path.exists(model_file) and os.path.exists(tokens_file):
                    print(f"✓ Найдена папка MitaAIShka: {test_path}")
                    return test_path
            if dirpath.count(os.sep) - root.count(os.sep) > 4:
                continue

    print("⚠ Папка MitaAIShka не найдена! Использую текущую директорию.")
    return os.getcwd()


# Поиск папки с моделью
MODEL_DIR = find_mita_folder()
JSON_CACHE_FILE = os.path.join(os.getcwd(), "saveAppPty.json")
SOUNDS_DIR = os.path.join(os.getcwd(), "sounds")

model_path = os.path.join(MODEL_DIR, "model.int8.onnx")
tokens_path = os.path.join(MODEL_DIR, "tokens.txt")

if not os.path.exists(model_path) or not os.path.exists(tokens_path):
    print("❌ ОШИБКА: Файлы модели не найдены!")
    print(f"   Искал в: {MODEL_DIR}")
    print("   Убедитесь, что папка MitaAIShka содержит model.int8.onnx и tokens.txt")
    exit(1)

print(f"✅ Модель загружена из: {MODEL_DIR}")
print("=" * 60)

recognizer = sherpa_onnx.OfflineRecognizer.from_nemo_ctc(
    model=model_path,
    tokens=tokens_path,
    num_threads=6,
    debug=False,
)

sample_rate = 16000
audio_queue = queue.Queue()

ENERGY_THRESHOLD = 0.015
SILENCE_LIMIT = 4

TRIGGER_WORDS = ["мита", "стелла", "стелам", "стеллу"]

# Глаголы действий
APP_VERBS = ["запусти", "запустил", "включи", "запустить", "включить"]
WEB_VERBS = ["открой", "открыть", "перейди", "перейти", "покажи"]
CLOSE_VERBS = ["закрой", "закрыть", "выключи", "выключить", "убей"]
LANG_VERBS = ["измени", "смени", "поменяй", "переключи"]
MOVE_VERBS = ["переведи", "перемести", "перекинь", "отправь"]
WRITE_VERBS = ["напиши", "напечатай", "пиши", "печатай", "набор"]
MINIMIZE_VERBS = ["сверни", "свернуть", "скрой", "спрячь"]

# Разговорные фразы
GREETINGS_MAP = {
    "ты тут": ("Да, я здесь! Чем помочь?", ["you_here", "ty_tut", "tut"]),
    "ты здесь": ("Здесь. Слушаю вас.", ["you_here", "ty_tut", "tut"]),
    "привет": ("Привет! Что нужно запустить или открыть?", ["hello", "privet"]),
    "как дела": ("Все отличнo, готова к работе!", ["how_are_you", "kak_dela"]),
    "не спишь": ("", ["how_are_you", "ne_spish"]),
    "спишь": ("", ["how_are_you", "ne_spish"]),
}

# Маппинг голосовых команд в системные ключи
APP_TRANSLIT_MAP = {
    "дискорд": "discord",
    "роблокс": "roblox",
    "стим": "steam",
    "тим": "steam",
    "телеграм": "telegram",
    "телеграмму": "telegram",
    "телеграмм": "telegram",
    "телега": "telegram",
    "браузер": "chrome",
    "хром": "chrome",
    "янндекс": "yandex",
    "спотифай": "spotify",
    "дота": "dota2",
    "кс": "cs2",
    "калькулятор": "calc",
    "блокнот": "notepad",
}

APP_EXE_MAP = {
    "roblox": "RobloxPlayerBeta.exe",
    "discord": "Discord.exe",
    "telegram": "Telegram.exe",
    "steam": "steam.exe",
    "chrome": "chrome.exe",
    "yandex": "browser.exe",
    "spotify": "Spotify.exe",
    "dota2": "dota2.exe",
    "геншн": "launcher.exe",
    "геншин": "launcher.exe",
    "генше": "launcher.exe",
    "геншин инпакт": "launcher.exe",
    "genshin impact": "launcher.exe",
    "геншен": "launcher.exe",
    "obs": "obs.bat",
    "обс": "obs.bat",
    "об": "obs.bat",
    "бс": "obs.bat",
    "cs2": "cs2.exe",
    "xeno": "Xeno-v1.3.60.exe",
    "ксена": "Xeno-v1.3.60.exe",
    "ксено": "Xeno-v1.3.60.exe",
    "calc": "calc.exe",
    "notepad": "notepad.exe",
}

PROCESS_KILL_MAP = {
    "obs": "obs64.exe",
    "обс": "obs64.exe",
    "об": "obs64.exe",
    "discord": "Discord.exe",
    "telegram": "Telegram.exe",
    "steam": "steam.exe",
    "chrome": "chrome.exe",
    "yandex": "browser.exe",
    "spotify": "Spotify.exe",
    "xeno": "Xeno-v1.3.60.exe",
    "ксена": "Xeno-v1.3.60.exe",
    "ксено": "Xeno-v1.3.60.exe",
    "геншн": "HoYoPlay.exe",
    "геншин": "HoYoPlay.exe",
    "генше": "HoYoPlay.exe",
    "геншин инпакт": "HoYoPlay.exe",
    "genshin impact": "HoYoPlay.exe",
    "геншен": "HoYoPlay.exe",
    "dota2": "dota2.exe",
    "roblox": "RobloxPlayerBeta.exe",
    "cs2": "cs2.exe",
    "calc": "calc.exe",
    "notepad": "notepad.exe",
}

WEB_URLS_MAP = {
    "покет": ["https://m.pocketoption.com/ru/cabinet/demo-quick-high-low/", "https://blacksignalsus.com/"],
    "покет опшен": ["https://m.pocketoption.com/ru/cabinet/demo-quick-high-low/", "https://blacksignalsus.com/"],
    "pocket": ["https://m.pocketoption.com/ru/cabinet/demo-quick-high-low/", "https://blacksignalsus.com/"],
    "тик ток": "https://www.tiktok.com",
    "тикток": "https://www.tiktok.com",
    "ютуб": "https://www.youtube.com",
    "ютубчик": "https://www.youtube.com",
    "гугл": "https://www.google.com",
    "яндекс": "https://ya.ru",
    "вк": "https://vk.com",
    "вконтакте": "https://vk.com",
    "телеграм": "https://web.telegram.org",
    "телеграмм": "https://web.telegram.org",
    "гитхаб": "https://github.com",
    "почту": "https://mail.google.com",
    "почта": "https://mail.google.com",
}


# ============================================================
# КОМАНДЫ УПРАВЛЕНИЯ С ПРАВИЛЬНОЙ РАБОТОЙ КЛАВИШ
# ============================================================

def press_key_with_delay(key_combination, delay=0.1):
    """Нажимает комбинацию клавиш с задержкой"""
    time.sleep(delay)
    if isinstance(key_combination, tuple):
        pyautogui.hotkey(*key_combination)
    else:
        pyautogui.press(key_combination)
    time.sleep(delay)


def keyboard_action_select_all():
    """Выделить всё (Ctrl+A)"""
    print("[Стелла]: Выделяю всё (Ctrl+A)")
    time.sleep(0.05)
    pyautogui.hotkey('ctrl', 'a')
    time.sleep(0.05)
    play_sound("select") or play_sound("ok")


def keyboard_action_copy():
    """Копировать (Ctrl+C)"""
    print("[Стелла]: Копирую (Ctrl+C)")
    time.sleep(0.05)
    pyautogui.hotkey('ctrl', 'c')
    time.sleep(0.05)
    play_sound("copy") or play_sound("ok")


def keyboard_action_paste():
    """Вставить (Ctrl+V)"""
    print("[Стелла]: Вставляю (Ctrl+V)")
    time.sleep(0.05)
    pyautogui.hotkey('ctrl', 'v')
    time.sleep(0.05)
    play_sound("paste") or play_sound("ok")


def keyboard_action_cut():
    """Вырезать (Ctrl+X)"""
    print("[Стелла]: Вырезаю (Ctrl+X)")
    time.sleep(0.05)
    pyautogui.hotkey('ctrl', 'x')
    time.sleep(0.05)
    play_sound("cut") or play_sound("ok")


def keyboard_action_undo():
    """Отменить (Ctrl+Z)"""
    print("[Стелла]: Отменяю (Ctrl+Z)")
    time.sleep(0.05)
    pyautogui.hotkey('ctrl', 'z')
    time.sleep(0.05)
    play_sound("undo") or play_sound("ok")


def keyboard_action_redo():
    """Повторить (Ctrl+Y)"""
    print("[Стелла]: Повторяю (Ctrl+Y)")
    time.sleep(0.05)
    pyautogui.hotkey('ctrl', 'y')
    time.sleep(0.05)
    play_sound("redo") or play_sound("ok")


def keyboard_action_save():
    """Сохранить (Ctrl+S)"""
    print("[Стелла]: Сохраняю (Ctrl+S)")
    time.sleep(0.05)
    pyautogui.hotkey('ctrl', 's')
    time.sleep(0.05)
    play_sound("save") or play_sound("ok")


def keyboard_action_find():
    """Найти (Ctrl+F)"""
    print("[Стелла]: Открываю поиск (Ctrl+F)")
    time.sleep(0.05)
    pyautogui.hotkey('ctrl', 'f')
    time.sleep(0.05)
    play_sound("find") or play_sound("ok")


def keyboard_action_print():
    """Печать (Ctrl+P)"""
    print("[Стелла]: Открываю печать (Ctrl+P)")
    time.sleep(0.05)
    pyautogui.hotkey('ctrl', 'p')
    time.sleep(0.05)
    play_sound("print") or play_sound("ok")


def keyboard_action_select_word():
    """Выделить слово (двойной клик)"""
    print("[Стелла]: Выделяю слово")
    time.sleep(0.05)
    pyautogui.doubleClick()
    time.sleep(0.05)
    play_sound("select") or play_sound("ok")


def keyboard_action_enter():
    """Нажать Enter"""
    print("[Стелла]: Нажимаю Enter")
    time.sleep(0.05)
    pyautogui.press('enter')
    time.sleep(0.05)
    play_sound("enter") or play_sound("ok")


def keyboard_action_escape():
    """Нажать Escape"""
    print("[Стелла]: Нажимаю Escape")
    time.sleep(0.05)
    pyautogui.press('esc')
    time.sleep(0.05)
    play_sound("escape") or play_sound("ok")


def keyboard_action_tab():
    """Нажать Tab"""
    print("[Стелла]: Нажимаю Tab")
    time.sleep(0.05)
    pyautogui.press('tab')
    time.sleep(0.05)
    play_sound("tab") or play_sound("ok")


def keyboard_action_space():
    """Нажать Пробел"""
    print("[Стелла]: Нажимаю Пробел")
    time.sleep(0.05)
    pyautogui.press('space')
    time.sleep(0.05)
    play_sound("space") or play_sound("ok")


def keyboard_action_delete():
    """Нажать Delete"""
    print("[Стелла]: Нажимаю Delete")
    time.sleep(0.05)
    pyautogui.press('delete')
    time.sleep(0.05)
    play_sound("delete") or play_sound("ok")


def keyboard_action_backspace():
    """Нажать Backspace"""
    print("[Стелла]: Нажимаю Backspace")
    time.sleep(0.05)
    pyautogui.press('backspace')
    time.sleep(0.05)
    play_sound("backspace") or play_sound("ok")


def keyboard_action_shift_tab():
    """Shift+Tab (назад по элементам)"""
    print("[Стелла]: Нажимаю Shift+Tab")
    time.sleep(0.05)
    pyautogui.hotkey('shift', 'tab')
    time.sleep(0.05)
    play_sound("tab") or play_sound("ok")


def keyboard_action_alt_tab():
    """Alt+Tab (переключение окон)"""
    print("[Стелла]: Переключаю окна (Alt+Tab)")
    time.sleep(0.05)
    pyautogui.hotkey('alt', 'tab')
    time.sleep(0.05)
    play_sound("switch") or play_sound("ok")


def keyboard_action_win_d():
    """Win+D (показать рабочий стол)"""
    print("[Стелла]: Показываю рабочий стол (Win+D)")
    time.sleep(0.05)
    pyautogui.hotkey('win', 'd')
    time.sleep(0.05)
    play_sound("desktop") or play_sound("ok")


def keyboard_action_win_m():
    """Win+M (свернуть все окна)"""
    print("[Стелла]: Сворачиваю все окна (Win+M)")
    time.sleep(0.05)
    pyautogui.hotkey('win', 'm')
    time.sleep(0.05)
    play_sound("minimize_all") or play_sound("ok")


def keyboard_action_alt_f4():
    """Alt+F4 (закрыть окно)"""
    print("[Стелла]: Закрываю окно (Alt+F4)")
    time.sleep(0.05)
    pyautogui.hotkey('alt', 'f4')
    time.sleep(0.05)
    play_sound("close_window") or play_sound("ok")


# ============================================================
# СЛОВАРЬ КОМАНД С ПРАВИЛЬНОЙ РАБОТОЙ КЛАВИШ
# ============================================================

CONTROL_COMMANDS = {
    # КОМАНДЫ РАБОТЫ С ТЕКСТОМ
    "выдели все": keyboard_action_select_all,
    "выделить все": keyboard_action_select_all,
    "всё выделить": keyboard_action_select_all,
    "выдели всё": keyboard_action_select_all,

    "скопировать": keyboard_action_copy,
    "скопируй": keyboard_action_copy,
    "копируй": keyboard_action_copy,
    "копировать": keyboard_action_copy,

    "вставить": keyboard_action_paste,
    "вставь": keyboard_action_paste,
    "вклей": keyboard_action_paste,

    "вырезать": keyboard_action_cut,
    "вырежь": keyboard_action_cut,

    "отменить": keyboard_action_undo,
    "отмена": keyboard_action_undo,
    "отмени": keyboard_action_undo,

    "повторить": keyboard_action_redo,
    "повтор": keyboard_action_redo,
    "повтори": keyboard_action_redo,

    "сохранить": keyboard_action_save,
    "сохрани": keyboard_action_save,
    "сейв": keyboard_action_save,

    "найти": keyboard_action_find,
    "поиск": keyboard_action_find,
    "искать": keyboard_action_find,

    "печать": keyboard_action_print,
    "распечатать": keyboard_action_print,

    "выделить слово": keyboard_action_select_word,
    "выдели слово": keyboard_action_select_word,

    # КЛАВИШИ
    "энтер": keyboard_action_enter,
    "ввод": keyboard_action_enter,
    "enter": keyboard_action_enter,

    "эскейп": keyboard_action_escape,
    "esc": keyboard_action_escape,
    "выход": keyboard_action_escape,

    "таб": keyboard_action_tab,
    "tab": keyboard_action_tab,

    "пробел": keyboard_action_space,
    "space": keyboard_action_space,

    "делит": keyboard_action_delete,
    "delete": keyboard_action_delete,
    "удали": keyboard_action_delete,
    "удалить": keyboard_action_delete,

    "бэкспейс": keyboard_action_backspace,
    "backspace": keyboard_action_backspace,
    "бекспейс": keyboard_action_backspace,

    "шифт таб": keyboard_action_shift_tab,
    "shift tab": keyboard_action_shift_tab,

    "альт таб": keyboard_action_alt_tab,
    "alt tab": keyboard_action_alt_tab,
    "переключить окно": keyboard_action_alt_tab,
    "переключи окно": keyboard_action_alt_tab,

    "рабочий стол": keyboard_action_win_d,
    "показать стол": keyboard_action_win_d,
    "вин д": keyboard_action_win_d,

    "свернуть все": keyboard_action_win_m,
    "сверни все": keyboard_action_win_m,
    "вин м": keyboard_action_win_m,

    "закрыть окно": keyboard_action_alt_f4,
    "закрой окно": keyboard_action_alt_f4,
    "альт ф4": keyboard_action_alt_f4,

    # СКРОЛЛ
    "вверх": lambda: (pyautogui.scroll(400), play_sound("scroll_up") or play_sound("ok")),
    "скролл вверх": lambda: (pyautogui.scroll(400), play_sound("scroll_up") or play_sound("ok")),
    "вниз": lambda: (pyautogui.scroll(-400), play_sound("scroll_down") or play_sound("ok")),
    "скролл вниз": lambda: (pyautogui.scroll(-400), play_sound("scroll_down") or play_sound("ok")),

    # КЛИКИ
    "клик": lambda: (pyautogui.click(), play_sound("click") or play_sound("ok")),
    "нажми": lambda: (pyautogui.click(), play_sound("click") or play_sound("ok")),
    "дабл клик": lambda: (pyautogui.doubleClick(), play_sound("double_click") or play_sound("ok")),
    "двойной клик": lambda: (pyautogui.doubleClick(), play_sound("double_click") or play_sound("ok")),
}


# ============================================================
# ОСТАЛЬНЫЕ ФУНКЦИИ (БЕЗ ИЗМЕНЕНИЙ)
# ============================================================

def minimize_all_windows():
    """Сворачивает все открытые окна (Win + M)."""
    if not play_sound("minimize_all") and not play_sound("minimize"):
        play_sound("ok")
    pyautogui.hotkey('win', 'm')
    print("[Стелла]: Все окна свернуты (Win + M)")
    return True


def minimize_window(target_raw: str):
    """Сворачивает конкретное окно по названию приложения."""
    target_clean = target_raw.lower().strip()
    app_key = APP_TRANSLIT_MAP.get(target_clean, target_clean)
    exe_name = APP_EXE_MAP.get(app_key, app_key)

    if not play_sound(f"minimize_{app_key}") and not play_sound("minimize"):
        play_sound("ok")

    win = get_window_by_exe_name(exe_name)

    if win:
        try:
            if win.isMinimized:
                print(f"[Стелла]: Окно '{win.title or app_key}' уже свернуто.")
                return True
            win.minimize()
            print(f"[Стелла]: Окно '{win.title or app_key}' свернуто.")
            return True
        except Exception as e:
            print(f"[Ошибка сворачивания окна]: {e}")
            play_sound("error")
            return False
    else:
        print(f"[Ошибка]: Окно программы '{target_raw}' не найдено.")
        play_sound("error")
        return False


def toggle_fullscreen():
    """Фокусирует текущее активное окно и разворачивает его на весь экран."""
    pyautogui.hotkey("f")
    if not play_sound("fullscreen"):
        play_sound()


def change_language():
    """Смена языка ввода на ПК (Alt + Shift)."""
    if not play_sound("change_lang"):
        if not play_sound("language"):
            play_sound("ok")
    pyautogui.hotkey("alt", "shift")
    pyautogui.hotkey("ctrl", "shift")


def play_ukrainian_song():
    """Воспроизводит случайную украинскую песню на YouTube."""
    if not play_sound("ukr_song") and not play_sound("ukr"):
        play_sound("ok")
    # Сокращенный список для примера
    UKRAINIAN_SONGS_URLS = [
        "https://www.youtube.com/results?search_query=KAZKA+-+%D0%9F%D0%BB%D0%B0%D0%BA%D0%B0%D0%BB%D0%B0",
        "https://www.youtube.com/results?search_query=KALUSH+%26+SKOFKA+-+%D0%94%D0%BE%D0%B4%D0%BE%D0%BC%D1%83",
    ]
    url = random.choice(UKRAINIAN_SONGS_URLS)
    print(f"[Музыка]: Рандомно выбрана украинская песня -> {url}")
    webbrowser.open(url)


def play_random_song():
    """Воспроизводит случайную песню на YouTube."""
    if not play_sound("random_song") and not play_sound("random"):
        play_sound("ok")
    RANDOM_SONGS_URLS = [
        "https://www.youtube.com/watch?v=pgN-vvVVxMA",
        "https://www.youtube.com/watch?v=P1t9T1TAOBI",
    ]
    url = random.choice(RANDOM_SONGS_URLS)
    print(f"[Музыка]: Рандомно выбрана случайная песня -> {url}")
    webbrowser.open(url)


def get_window_by_exe_name(exe_name: str):
    """Находит окно приложения по названию его .exe процесса."""
    target_exe = exe_name.lower()
    if not target_exe.endswith(".exe"):
        target_exe += ".exe"

    matching_pids = set()
    for proc in psutil.process_iter(['pid', 'name']):
        try:
            if proc.info['name'] and proc.info['name'].lower() == target_exe:
                matching_pids.add(proc.info['pid'])
        except (psutil.NoSuchProcess, psutil.AccessDenied):
            continue

    if not matching_pids:
        return None

    all_wins = gw.getAllWindows()
    for w in all_wins:
        if not w.title and w.width == 0 and w.height == 0:
            continue
        try:
            if hasattr(w, '_hWnd'):
                import win32process
                _, win_pid = win32process.GetWindowThreadProcessId(w._hWnd)
                if win_pid in matching_pids and w.visible:
                    return w
        except Exception:
            pass

    clean_name = exe_name.replace(".exe", "").lower()
    for w in all_wins:
        if w.title and clean_name in w.title.lower():
            return w

    return None


def move_window_to_monitor(target_raw: str):
    """Перемещает активное окно или окно с указанным именем на 1-й или 2-й монитор."""
    target_clean = target_raw.lower().strip()

    target_monitor = None
    if "первый" in target_clean or "1" in target_clean or "один" in target_clean:
        target_monitor = 1
    elif "второй" in target_clean or "2" in target_clean or "два" in target_clean:
        target_monitor = 2
    else:
        print("[Ошибка]: Не указан номер монитора (первый/второй или 1/2).")
        play_sound("error")
        return False

    words_to_remove = [
        "на", "в", "до", "экран", "монитор", "первый", "второй",
        "1", "2", "один", "два", "переведи", "перемести", "перекинь", "отправь"
    ]
    cleaned_query_words = [w for w in target_clean.split() if w not in words_to_remove]
    app_query_str = " ".join(cleaned_query_words).strip()

    win = None

    if not app_query_str or any(w in app_query_str for w in ["окно", "это", "текущее", "активное"]):
        win = gw.getActiveWindow()
    else:
        app_key = APP_TRANSLIT_MAP.get(app_query_str, app_query_str)
        exe_name = APP_EXE_MAP.get(app_key, app_key)
        win = get_window_by_exe_name(exe_name)

    if not win:
        print(f"[Ошибка]: Окно программы '{app_query_str}' не найдено среди запущенных.")
        play_sound("error")
        return False

    PRIMARY_MONITOR_WIDTH = 1920

    if target_monitor == 1:
        new_x = 100
    else:
        new_x = PRIMARY_MONITOR_WIDTH + 100

    new_y = 100

    try:
        if win.isMinimized:
            win.restore()
        if win.isMaximized:
            win.restore()

        win.moveTo(new_x, new_y)
        win.maximize()

        print(f"[Успех]: Окно '{win.title or app_query_str}' перемещено на монитор {target_monitor}.")
        if not play_sound("move") and not play_sound("move_window"):
            play_sound("Okno")
        return True
    except Exception as e:
        print(f"[Ошибка перемещения]: {e}")
        play_sound("error")
        return False


def play_sound_worker(file_path: str):
    """Воспроизведение звука с проверкой soundfile."""
    try:
        if HAS_SOUNDFILE:
            data, fs = sf.read(file_path, dtype="float32")
            sd.play(data, fs)
            sd.wait()
    except Exception as e:
        print(f"[Звук Ошибка]: {e}")


def play_sound(sound_name: str) -> bool:
    """Ищет и воспроизводит аудиофайл из папки sounds."""
    if not os.path.exists(SOUNDS_DIR):
        return False

    target_clean = sound_name.lower().strip()
    mapped_name = APP_TRANSLIT_MAP.get(target_clean, target_clean)

    if "тик" in target_clean:
        mapped_name = "tiktok"
    elif "ютуб" in target_clean:
        mapped_name = "youtube"
    elif "вк" in target_clean or "контакт" in target_clean:
        mapped_name = "vk"

    variants = {
        target_clean,
        target_clean.replace(" ", ""),
        target_clean.replace(" ", "_"),
        mapped_name,
        mapped_name.replace(" ", ""),
        mapped_name.replace(" ", "_"),
    }

    for file in os.listdir(SOUNDS_DIR):
        file_base, ext = os.path.splitext(file)
        if ext.lower() not in [".wav", ".mp3", ".ogg", ".flac"]:
            continue

        if file_base.lower() in variants:
            sound_path = os.path.join(SOUNDS_DIR, file)
            print(f"[Звук]: Воспроизведение -> {file}")
            threading.Thread(
                target=play_sound_worker, args=(sound_path,), daemon=True
            ).start()
            return True

    return False


def load_app_cache() -> dict:
    if not os.path.exists(JSON_CACHE_FILE):
        save_app_cache({})
        return {}
    try:
        with open(JSON_CACHE_FILE, "r", encoding="utf-8") as f:
            return json.load(f)
    except (json.JSONDecodeError, OSError):
        return {}


def save_app_cache(cache_data: dict):
    with open(JSON_CACHE_FILE, "w", encoding="utf-8") as f:
        json.dump(cache_data, f, ensure_ascii=False, indent=4)


def find_exe_fast_registry(exe_name: str) -> str | None:
    if not exe_name.endswith(".exe"):
        exe_name = f"{exe_name}.exe"

    reg_key = rf"SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\{exe_name}"

    for hive in (winreg.HKEY_CURRENT_USER, winreg.HKEY_LOCAL_MACHINE):
        try:
            with winreg.OpenKey(hive, reg_key) as key:
                path, _ = winreg.QueryValueEx(key, "")
                if os.path.exists(path):
                    return path
        except OSError:
            continue
    return None


def search_exe_on_disks(exe_name: str) -> str | None:
    target_exe = (
        exe_name.lower()
        if exe_name.endswith(".exe")
        else f"{exe_name}.exe".lower()
    )
    drives = get_available_drives()
    priority_dirs = ["Program Files", "Program Files (x86)", "Users", "Games"]

    print(f"[Поиск ПК] Сканирую диски для '{target_exe}'...")

    for drive in drives:
        for p_dir in priority_dirs:
            full_p_path = os.path.join(drive, p_dir)
            if os.path.exists(full_p_path):
                for root, _, files in os.walk(full_p_path):
                    for file in files:
                        if file.lower() == target_exe:
                            return os.path.join(root, file)

        for root, _, files in os.walk(drive):
            if any(
                    skip in root.lower()
                    for skip in ["$recycle.bin", "windows\\winsxs", "windows\\servicing"]
            ):
                continue
            for file in files:
                if file.lower() == target_exe:
                    return os.path.join(root, file)

    return None


def launch_application(target_raw: str):
    target_clean = target_raw.lower().strip()
    app_key = APP_TRANSLIT_MAP.get(target_clean, target_clean)
    exe_name = APP_EXE_MAP.get(app_key, app_key)

    if not play_sound(target_clean):
        if not play_sound(app_key):
            play_sound("ok")

    cache = load_app_cache()

    if app_key in cache:
        cached_path = cache[app_key]
        if os.path.exists(cached_path):
            print(f"[Кэш JSON]: Запуск из saveAppPty.json -> {cached_path}")
            os.startfile(cached_path)
            return True

    found_path = find_exe_fast_registry(exe_name)

    if not found_path:
        found_path = search_exe_on_disks(exe_name)

    if found_path and os.path.exists(found_path):
        print(f"[Успех]: Приложение найдено -> {found_path}")
        cache[app_key] = found_path
        save_app_cache(cache)
        os.startfile(found_path)
        return True

    try:
        subprocess.Popen(
            exe_name,
            shell=True,
            stdout=subprocess.DEVNULL,
            stderr=subprocess.DEVNULL,
        )
        return True
    except Exception:
        pass

    print(
        f"[Ошибка]: Не удалось найти программу '{target_raw}' (искали: {exe_name})"
    )
    play_sound("error")
    return False


def kill_application(target_raw: str):
    """Принудительно закрывает процесс приложения через taskkill с озвучкой close_<app_key>."""
    target_clean = target_raw.lower().strip()
    app_key = APP_TRANSLIT_MAP.get(target_clean, target_clean)

    specific_close_sound = f"close_{app_key}"

    if not play_sound(specific_close_sound):
        if not play_sound("close"):
            play_sound("ok")

    exe_name = PROCESS_KILL_MAP.get(app_key, app_key)
    if not exe_name.endswith(".exe"):
        exe_name += ".exe"

    print(f"[Убить процесс]: Завершение {exe_name}...")

    cmd = f'taskkill /F /IM "{exe_name}"'

    result = subprocess.run(
        cmd, shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True
    )

    if result.returncode == 0:
        print(f"[Успех]: Процесс {exe_name} успешно завершен.")
        return True
    else:
        print(f"[Ошибка]: Процесс {exe_name} не найден или не может быть закрыт.")
        play_sound("error")
        return False


def open_website(target_raw: str):
    target_clean = target_raw.lower().strip()

    if not play_sound(target_clean):
        play_sound("ok")

    if target_clean in WEB_URLS_MAP:
        url_or_list = WEB_URLS_MAP[target_clean]

        if isinstance(url_or_list, list):
            print(f"[Веб]: Открываю {len(url_or_list)} ссылок")
            for url in url_or_list:
                print(f"[Веб]: Перехожу на {url}")
                webbrowser.open(url)
        else:
            url = url_or_list
            print(f"[Веб]: Перехожу на {url}")
            webbrowser.open(url)
        return True

    if "." in target_clean and not target_clean.endswith(".exe"):
        url = (
            f"https://{target_clean}"
            if not target_clean.startswith("http")
            else target_clean
        )
        print(f"[Веб]: Перехожу на {url}")
        webbrowser.open(url)
        return True

    search_url = f"https://www.google.com/search?q={target_clean}"
    print(f"[Веб]: Ищу в Google -> {target_clean}")
    webbrowser.open(search_url)
    return True


# ============================================================
# ОСНОВНАЯ ФУНКЦИЯ ОБРАБОТКИ КОМАНД
# ============================================================

def process_command(text: str):
    cleaned = text.lower().strip()
    words = cleaned.split()

    if not any(tw in words for tw in TRIGGER_WORDS):
        return

    print(f"\n[Распознано]: {text}")

    # Убираем триггер-слова
    filtered_words = [w for w in words if w not in TRIGGER_WORDS]

    # Убираем мат и стоп-слова
    filtered_words = [w for w in filtered_words if w not in BAD_WORDS]

    if not filtered_words:
        print("[Стелла]: Слушаю вас!")
        play_sound("da")
        return

    phrase_after_trigger = " ".join(filtered_words).strip()

    # ============================================================
    # ПРОВЕРКА КОМАНД ИЗ CONTROL_COMMANDS (С КЛАВИАТУРНЫМИ ДЕЙСТВИЯМИ)
    # ============================================================

    # Проверяем точное совпадение
    if phrase_after_trigger in CONTROL_COMMANDS:
        print(f"[Стелла]: Выполнение команды -> '{phrase_after_trigger}'")
        try:
            CONTROL_COMMANDS[phrase_after_trigger]()
            return
        except Exception as e:
            print(f"[Ошибка выполнения]: {e}")
            play_sound("error")
            return

    # Проверяем частичное совпадение (для фраз с дополнительными словами)
    for cmd in CONTROL_COMMANDS:
        if cmd in phrase_after_trigger:
            print(f"[Стелла]: Выполнение команды -> '{cmd}'")
            try:
                CONTROL_COMMANDS[cmd]()
                return
            except Exception as e:
                print(f"[Ошибка выполнения]: {e}")
                play_sound("error")
                return

    # 0. СВОРАЧИВАНИЕ ВСЕХ ОКОН (Win + M)
    if "все" in filtered_words and any(w in filtered_words for w in ["окна", "окон", "окно"]):
        play_sound("vseokna")
        minimize_all_windows()
        return

    # Сворачивание конкретного окна
    for verb in MINIMIZE_VERBS:
        if verb in filtered_words:
            if "все" in filtered_words and any(w in filtered_words for w in ["окна", "окон", "окно"]):
                continue

            target = " ".join([w for w in filtered_words if w != verb]).strip()
            if target:
                print(f"[Стелла]: Сворачиваю окно -> {target}")
                minimize_window(target)
                return
            else:
                win = gw.getActiveWindow()
                if win:
                    try:
                        win.minimize()
                        print("[Стелла]: Активное окно свернуто.")
                        play_sound("minimize") or play_sound("ok")
                    except:
                        print("[Ошибка]: Не удалось свернуть активное окно.")
                        play_sound("error")
                return

    # 1. МУЗЫКАЛЬНЫЕ КОМАНДЫ
    if any(w in phrase_after_trigger for w in ["украинскую", "укр"]) and "песн" in phrase_after_trigger:
        play_ukrainian_song()
        return

    if any(w in phrase_after_trigger for w in ["рандомную", "рандом", "случайную"]) and "песн" in phrase_after_trigger:
        play_random_song()
        return

    # 2. Перемещение окон на мониторы
    for verb in MOVE_VERBS:
        if verb in filtered_words:
            target = " ".join([w for w in filtered_words if w != verb]).strip()
            if target:
                print(f"[Стелла]: Перемещение окна -> {target}")
                move_window_to_monitor(target)
                return

    # 3. Обработка команды "напиши" (печать текста)
    for verb in WRITE_VERBS:
        if verb in filtered_words:
            verb_index = filtered_words.index(verb)
            text_to_type = " ".join(filtered_words[verb_index + 1:]).strip()

            if text_to_type:
                print(f"[Стелла]: Печатаю текст -> {text_to_type}")

                if not play_sound("write"):
                    play_sound("Pechat")

                time.sleep(0.1)
                keyboard.write(text_to_type, delay=0.02)

                if verb in ["напечатай", "набор"]:
                    keyboard.press_and_release('enter')

                return
            else:
                print("[Стелла]: Не указан текст для печати")
                play_sound("error")
                return

    # 4. Запуск локальных приложений
    for verb in APP_VERBS:
        if verb in filtered_words:
            target = " ".join([w for w in filtered_words if w != verb]).strip()
            if target:
                print(f"[Стелла]: Запуск приложения -> {target}")
                launch_application(target)
                return

    # 5. Закрытие приложений
    for verb in CLOSE_VERBS:
        if verb in filtered_words:
            target = " ".join([w for w in filtered_words if w != verb]).strip()
            if target:
                print(f"[Стелла]: Закрытие процесса -> {target}")
                kill_application(target)
                return

    # 6. Смена языка
    for verb in LANG_VERBS:
        if verb in filtered_words:
            target = " ".join([w for w in filtered_words if w != verb]).strip()
            if target in ["язык", "раскладку"]:
                print("[Стелла]: Смена языка ввода...")
                change_language()
                return

    # 7. Разговорные ответы
    if phrase_after_trigger in GREETINGS_MAP:
        text_response, sound_variants = GREETINGS_MAP[phrase_after_trigger]
        print(f"[Стелла]: {text_response}")

        sound_played = False
        for s_name in sound_variants:
            if play_sound(s_name):
                sound_played = True
                break

        if not sound_played:
            play_sound("ok")
        return

    print(f"[Стелла]: Команда '{phrase_after_trigger}' не распознана.")
    play_sound("error")


# ============================================================
# ФУНКЦИЯ АУДИО-ОБРАБОТКИ
# ============================================================

def audio_callback(indata, frames, time, status):
    audio_queue.put(indata.copy())


def main():
    word_buffer = []
    silence_counter = 0
    is_speaking = False

    if not os.path.exists(SOUNDS_DIR):
        os.makedirs(SOUNDS_DIR)
        print(f"[Инфо]: Создана папка для звуков -> {SOUNDS_DIR}")

    load_app_cache()

    print("=" * 60)
    print("Детектор 'Стелла' активен.")
    print("\n📋 КОМАНДЫ РАБОТЫ С ТЕКСТОМ:")
    print("  'Стелла ВЫДЕЛИ ВСЁ'      -> Ctrl+A")
    print("  'Стелла СКОПИРУЙ'        -> Ctrl+C")
    print("  'Стелла ВСТАВЬ'          -> Ctrl+V")
    print("  'Стелла ВЫРЕЖЬ'          -> Ctrl+X")
    print("  'Стелла ОТМЕНИ'          -> Ctrl+Z")
    print("  'Стелла ПОВТОРИ'         -> Ctrl+Y")
    print("  'Стелла СОХРАНИ'         -> Ctrl+S")
    print("  'Стелла НАЙДИ'           -> Ctrl+F")
    print("\n⌨️ КОМАНДЫ КЛАВИШ:")
    print("  'Стелла ЭНТЕР'           -> Enter")
    print("  'Стелла ПРОБЕЛ'          -> Space")
    print("  'Стелла ТАБ'             -> Tab")
    print("  'Стелла ЭСКЕЙП'          -> Escape")
    print("  'Стелла ДЕЛИТ'           -> Delete")
    print("  'Стелла БЭКСПЕЙС'        -> Backspace")
    print("  'Стелла АЛЬТ ТАБ'        -> Alt+Tab")
    print("  'Стелла ВИН Д'           -> Win+D (рабочий стол)")
    print("  'Стелла ВИН М'           -> Win+M (свернуть всё)")
    print("  'Стелла АЛЬТ Ф4'         -> Alt+F4 (закрыть окно)")
    print("\n📝 ОСТАЛЬНЫЕ КОМАНДЫ:")
    print("  'Стелла НАПИШИ [текст]'  -> Печать текста")
    print("  'Стелла СВЕРНИ ВСЕ'      -> Свернуть все окна")
    print("  'Стелла ЗАПУСТИ [прогу]' -> Запуск программы")
    print("  'Стелла ЗАКРОЙ [прогу]'  -> Закрыть программу")
    print("  'Стелла ОТКРОЙ [сайт]'   -> Открыть сайт")
    print("=" * 60)
    print("💡 ВСЕ КОМАНДЫ РАБОТАЮТ КАК РЕАЛЬНЫЕ НАЖАТИЯ НА КЛАВИАТУРЕ!")
    print("=" * 60)

    with sd.InputStream(
            samplerate=sample_rate,
            channels=1,
            dtype="float32",
            blocksize=800,
            callback=audio_callback,
    ):
        while True:
            try:
                data = audio_queue.get(timeout=0.02)
            except queue.Empty:
                continue

            samples = data[:, 0]
            amplitude = np.max(np.abs(samples))

            if amplitude > ENERGY_THRESHOLD:
                is_speaking = True
                silence_counter = 0
                word_buffer.append(samples)
            elif is_speaking:
                word_buffer.append(samples)
                silence_counter += 1

                if silence_counter >= SILENCE_LIMIT:
                    full_phrase_audio = np.concatenate(word_buffer)

                    stream = recognizer.create_stream()
                    stream.accept_waveform(sample_rate, full_phrase_audio)
                    recognizer.decode_stream(stream)

                    recognized_text = stream.result.text.strip()

                    if recognized_text:
                        process_command(recognized_text)

                    word_buffer = []
                    silence_counter = 0
                    is_speaking = False


if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt:
        print("\nОстановка.")
