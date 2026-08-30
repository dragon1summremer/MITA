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

# Загрузка soundfile при наличии
try:
    import soundfile as sf

    HAS_SOUNDFILE = True
except ImportError:
    HAS_SOUNDFILE = False

# Настройки PyAutoGUI
pyautogui.FAILSAFE = False
pyautogui.PAUSE = 0.05

# Добавьте этот словарь в начало скрипта, после импортов
STOP_WORDS = [
    "нахуй", "нахуй", "блядь", "блять", "сука", "сук", "хуй", "хую",
    "пизда", "пиздец", "ебан", "ебать", "заеб", "пидор", "гандон",
    "мудак", "уебок", "тупой", "дебил", "идиот", "кретин", "олень"
]

# Или более полный список
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
        # Стандартные пути установки
        r"C:\AI MITA\MitaAIShka",
        r"D:\AI MITA\MitaAIShka",
        r"E:\AI MITA\MitaAIShka",
        r"C:\Program Files\AI MITA\MitaAIShka",
        r"C:\Program Files (x86)\AI MITA\MitaAIShka",
        r"C:\Users\{}\Documents\AI MITA\MitaAIShka".format(os.getlogin()),
        r"C:\Users\{}\Desktop\AI MITA\MitaAIShka".format(os.getlogin()),
        # Текущая директория
        os.path.join(os.getcwd(), "MitaAIShka"),
        os.path.join(os.getcwd(), "AI MITA", "MitaAIShka"),
    ]

    # Поиск на всех доступных дисках
    for drive in get_available_drives():
        possible_paths.append(os.path.join(drive, "AI MITA", "MitaAIShka"))
        possible_paths.append(os.path.join(drive, "MitaAIShka"))
        possible_paths.append(os.path.join(drive, "Users", os.getlogin(), "Documents", "AI MITA", "MitaAIShka"))
        possible_paths.append(os.path.join(drive, "Users", os.getlogin(), "Desktop", "AI MITA", "MitaAIShka"))

    # Проверяем все пути (быстрый поиск)
    print("🔍 Ищу папку MitaAIShka...")
    for path in possible_paths:
        if os.path.exists(path):
            # Проверяем наличие model.int8.onnx и tokens.txt
            model_file = os.path.join(path, "model.int8.onnx")
            tokens_file = os.path.join(path, "tokens.txt")
            if os.path.exists(model_file) and os.path.exists(tokens_file):
                print(f"✓ Найдена папка MitaAIShka: {path}")
                return path

    # Если не нашли, ищем рекурсивно (только в Program Files и Users)
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
            # Ограничиваем глубину для скорости
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

# Проверка наличия файлов модели
if not os.path.exists(model_path) or not os.path.exists(tokens_path):
    print("❌ ОШИБКА: Файлы модели не найдены!")
    print(f"   Искал в: {MODEL_DIR}")
    print("   Убедитесь, что папка MitaAIShka содержит model.int8.onnx и tokens.txt")
    print("   Или укажите путь вручную в коде (строка MODEL_DIR = ...)")
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

# 1. Маппинг голосовых команд в системные ключи
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

# 2. Маппинг ключей в реальные имена файлов (ДЛЯ ЗАПУСКА)
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

# 3. Маппинг для ЗАКРЫТИЯ (реальные имена .exe процессов)
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

# Списки URL для случайно выбираемой музыки
UKRAINIAN_SONGS_URLS = [
    "https://www.youtube.com/results?search_query=KAZKA+-+%D0%9F%D0%BB%D0%B0%D0%BA%D0%B0%D0%BB%D0%B0",
    "https://www.youtube.com/results?search_query=KALUSH+%26+SKOFKA+-+%D0%94%D0%BE%D0%B4%D0%BE%D0%BC%D1%83",
    "https://www.youtube.com/results?search_query=%D0%92%D1%80%D0%B5%D0%BC%D1%8F+%D0%B8+%D0%A1%D1%82%D0%B5%D0%BA%D0%BB%D0%BE+-+%D0%94%D0%B8%D0%BC",
    "https://www.youtube.com/results?search_query=Kalush+Orchestra+-+Stefania",
    "https://www.youtube.com/results?search_query=Artem+Pivovarov+-+%D0%94%D0%B5%D0%B6%D0%B0%D0%B2%D1%8E",
    "https://www.youtube.com/results?search_query=%D0%9E%D0%BA%D0%B5%D0%B0%D0%BD+%D0%95%D0%BB%D1%8C%D0%B7%D0%B8+-+%D0%9E%D0%B1%D1%96%D0%B9%D0%BC%D0%B8",
    "https://www.youtube.com/results?search_query=SKOFKA+-+%D0%A7%D1%83%D1%82%D0%B8+%D0%B3%D1%96%D0%BC%D0%BD",
    "https://www.youtube.com/results?search_query=%D0%A5%D1%80%D0%B8%D1%81%D1%82%D0%B8%D0%BD%D0%B0+%D0%A1%D0%BE%D0%BB%D0%BE%D0%B2%D1%96%D0%B9+-+%D0%A2%D1%80%D0%B8%D0%BC%D0%B0%D0%B9",
    "https://www.youtube.com/results?search_query=Artem+Pivovarov+%26+DOROFEEVA+-+%D0%94%D1%83%D0%BC%D0%B8",
    "https://www.youtube.com/results?search_query=YAKTAK+%26+SOBOL+-+%D0%9F%D0%BE%D0%B3%D0%BB%D1%8F%D0%B4",
    "https://www.youtube.com/results?search_query=%D0%9C%D0%B0%D0%BA%D1%81%D0%B8%D0%BC+%D0%91%D0%BE%D1%80%D0%BE%D0%B4%D1%96%D0%BD+-+%D0%AF%D0%BA%D0%B1%D0%B8+%D0%BD%D0%B5+%D1%82%D0%B8",
    "https://www.youtube.com/results?search_query=DZIDZIO+-+%D0%92%D0%B8%D1%85%D1%96%D0%B4%D0%BD%D0%B8%D0%B9",
    "https://www.youtube.com/results?search_query=YAKTAK+%26+KOLA+-+%D0%9F%D0%BE%D1%80%D1%96%D1%87%D0%BA%D0%B0",
    "https://www.youtube.com/results?search_query=ZOZULYA+-+%D0%A7%D0%BE%D1%80%D0%BD%D0%B5+%D1%96+%D0%B1%D1%96%D0%BB%D0%B5",
    "https://www.youtube.com/results?search_query=Artem+Pivovarov+%26+NK+-+%D0%A2%D0%B0%D0%BC+%D1%83+%D1%82%D0%BE%D0%BF%D0%BE%D0%BB%D1%96",
    "https://www.youtube.com/results?search_query=O%D0%BB%D0%B5%D0%B3+%D0%92%D0%B8%D0%BD%D0%BD%D0%B8%D0%BA+-+%D0%92%D0%BE%D0%B2%D1%87%D0%B8%D1%86%D1%8F",
    "https://www.youtube.com/results?search_query=Artem+Pivovarov+%26+Klavdia+Petrivna+-+%D0%91%D0%B0%D1%80%D0%B0%D0%B1%D0%B0%D0%BD",
    "https://www.youtube.com/results?search_query=Wellboy+-+%D0%93%D1%83%D1%81%D0%B8",
    "https://www.youtube.com/results?search_query=%D0%9F%D0%BE%D1%82%D0%B0%D0%BF+%26+%D0%9E%D0%BB%D0%B5%D0%B3+%D0%92%D0%B8%D0%BD%D0%BD%D0%B8%D0%BA+-+%D0%9D%D0%B0%D0%B9%D0%BA%D1%80%D0%B0%D1%89%D0%B8%D0%B9+%D0%B4%D0%B5%D0%BD%D1%8C",
    "https://www.youtube.com/results?search_query=Klavdia+Petrivna+%26+%D0%9C%D0%B0%D1%88%D0%B0+%D0%9A%D0%BE%D0%BD%D0%B4%D1%80%D0%B0%D1%82%D0%B5%D0%BD%D0%BA%D0%BE+-+%D0%87%D0%B4%D0%B5+%D0%B4%D0%B0%D1%85",
    "https://www.youtube.com/results?search_query=%D0%9E%D0%BA%D0%B5%D0%B0%D0%BD+%D0%95%D0%BB%D1%8C%D0%B7%D0%B8+%26+YAKTAK+%26+KOLA+%26+SHUMEI+%26+Jerry+Heil+-+%D0%A2%D0%BE%D0%B9+%D0%B4%D0%B5%D0%BD%D1%8C",
    "https://www.youtube.com/results?search_query=YAKTAK+-+%D0%A3%D0%BD%D0%BE%D1%87%D1%96",
    "https://www.youtube.com/results?search_query=YAKTAK+-+LaLaLa",
    "https://www.youtube.com/results?search_query=YAKTAK+-+%D0%9D%D0%B5%D0%B1%D0%BE",
    "https://www.youtube.com/results?search_query=YAKTAK+-+%D0%9C%D0%BE%D1%80%D0%B5",
    "https://www.youtube.com/results?search_query=KOLA+-+%D0%91%D1%96%D0%BB%D1%8F+%D1%81%D0%B5%D1%80%D1%86%D1%8F",
    "https://www.youtube.com/results?search_query=KOLA+-+%D0%94%D0%BE%D1%87%D0%B5%D0%BA%D0%B0%D1%8E%D1%81%D1%8C",
    "https://www.youtube.com/results?search_query=KOLA+-+%D0%A7%D0%B8+%D1%80%D0%B0%D0%B7%D0%BE%D0%BC%3F",
    "https://www.youtube.com/results?search_query=KOLA+-+%D0%A6%D1%96%D0%BB%D1%83%D0%B9",
    "https://www.youtube.com/results?search_query=Jerry+Heil+-+CATHARTICUS",
    "https://www.youtube.com/results?search_query=Jerry+Heil+-+%D0%A2%D1%80%D0%B8+%D1%81%D0%BC%D1%83%D0%B6%D0%BA%D0%B8",
    "https://www.youtube.com/results?search_query=Jerry+Heil+-+%23%D0%90%D0%A0%D0%A2%D0%98%D0%9B%D0%95%D0%A0%D0%86%D0%AF",
    "https://www.youtube.com/results?search_query=alyona+alyona+%26+Jerry+Heil+-+Teresa+%26+Maria",
    "https://www.youtube.com/results?search_query=alyona+alyona+-+%D0%9F%D1%83%D1%88%D0%BA%D0%B0",
    "https://www.youtube.com/results?search_query=alyona+alyona+-+%D0%A0%D0%B8%D0%B1%D0%BA%D0%B8",
    "https://www.youtube.com/results?search_query=DOROFEEVA+-+747",
    "https://www.youtube.com/results?search_query=DOROFEEVA+-+%D0%AF%D0%BF%D0%BE%D0%BD%D1%96%D1%8F",
    "https://www.youtube.com/results?search_query=DOROFEEVA+-+%D0%9A%D0%BE%D1%85%D0%B0%D1%8E%2C+%D0%B0%D0%BB%D0%B5+%D0%BD%D0%B5+%D0%B7%D0%BE%D0%B2%D1%81%D1%96%D0%BC",
    "https://www.youtube.com/results?search_query=DOROFEEVA+-+%D0%A5%D0%B0%D0%B9+%D0%BF%D0%B8%D1%88%D1%83%D1%82%D1%8C",
    "https://www.youtube.com/results?search_query=DOROFEEVA+-+%D0%A9%D0%BE%D0%B1+%D0%BD%D0%B5+%D0%B1%D1%83%D0%BB%D0%BE",
    "https://www.youtube.com/results?search_query=Artem+Pivovarov+-+%D0%9C%D1%96%D1%80%D0%B0%D0%B6",
    "https://www.youtube.com/results?search_query=Artem+Pivovarov+-+%D0%9C%D0%B0%D0%BD%D1%96%D1%84%D0%B5%D1%81%D1%82",
    "https://www.youtube.com/results?search_query=Artem+Pivovarov+-+%D0%A0%D0%B0%D0%BD%D0%B4%D0%B5%D0%B2%D1%83",
    "https://www.youtube.com/results?search_query=Artem+Pivovarov+-+8+%D0%B4%D0%B8%D0%B2%D0%BE",
    "https://www.youtube.com/results?search_query=Artem+Pivovarov+-+%D0%9C%D1%96%D1%81%D1%8F%D1%86%D1%8C",
    "https://www.youtube.com/results?search_query=Artem+Pivovarov+-+%D0%94%D1%83%D0%BC%D0%B8",
    "https://www.youtube.com/results?search_query=Artem+Pivovarov+-+%D0%9C%D0%B0%D0%BC%D0%B0",
    "https://www.youtube.com/results?search_query=Tina+Karol+-+%D0%9D%D1%96%D0%B6%D0%BD%D0%BE",
    "https://www.youtube.com/results?search_query=Tina+Karol+-+%D0%A3%D0%BA%D1%80%D0%B0%D1%97%D0%BD%D0%B0+%E2%80%94+%D1%86%D0%B5+%D1%82%D0%B8",
    "https://www.youtube.com/results?search_query=Tina+Karol+-+%D0%97%D0%B4%D0%B0%D1%82%D0%B8%D1%81%D1%8C+%D1%82%D0%B8+%D0%B7%D0%B0%D0%B2%D0%B6%D0%B4%D0%B8+%D0%B2%D1%81%D1%82%D0%B8%D0%B3%D0%BD%D0%B5%D1%88",
    "https://www.youtube.com/results?search_query=Tina+Karol+-+%D0%A2%D1%80%D0%BE%D1%8F%D0%BD%D0%B4%D0%B8",
    "https://www.youtube.com/results?search_query=Tina+Karol+-+%D0%9D%D0%B0%D0%BC%D0%B0%D0%BB%D1%8E%D1%8E+%D1%82%D0%BE%D0%B1%D1%96+%D0%B7%D0%BE%D1%80%D1%96",
    "https://www.youtube.com/results?search_query=Tina+Karol+-+%D0%A1%D0%BA%D0%B0%D0%BD%D0%B4%D0%B0%D0%BB",
    "https://www.youtube.com/results?search_query=MONATIK+-+%D0%9A%D1%80%D1%83%D0%B6%D0%B8%D1%82",
    "https://www.youtube.com/results?search_query=MONATIK+-+Vitamin+D",
    "https://www.youtube.com/results?search_query=MONATIK+-+%D0%90+%D1%89%D0%BE%3F",
    "https://www.youtube.com/results?search_query=MONATIK+-+%D0%A1%D0%B8%D0%BB%D1%8C%D0%BD%D0%BE",
    "https://www.youtube.com/results?search_query=Max+Barskih+-+%D0%A1%D0%BB%D1%8C%D0%BE%D0%B7%D0%B8",
    "https://www.youtube.com/results?search_query=Max+Barskih+-+%D0%91%D1%83%D0%B4%D0%B5+%D0%B2%D0%B5%D1%81%D0%BD%D0%B0",
    "https://www.youtube.com/results?search_query=Max+Barskih+-+%D0%91%D0%B5%D1%80%D0%B5%D0%B3%D0%B0",
    "https://www.youtube.com/results?search_query=Max+Barskih+-+Bestseller",
    "https://www.youtube.com/results?search_query=Max+Barskih+-+%D0%A0%D0%B8%D1%82%D0%BC%D0%B8",
    "https://www.youtube.com/results?search_query=Max+Barskih+-+%D0%A2%D1%83%D0%BC%D0%B0%D0%BD%D0%B8",
    "https://www.youtube.com/results?search_query=%D0%90%D0%BD%D1%82%D0%B8%D1%82%D1%96%D0%BB%D0%B0+-+%D0%A4%D0%BE%D1%80%D1%82%D0%B5%D1%86%D1%8F+%D0%91%D0%B0%D1%85%D0%BC%D1%83%D1%82",
    "https://www.youtube.com/results?search_query=%D0%90%D0%BD%D1%82%D0%B8%D1%82%D1%96%D0%BB%D0%B0+-+%D0%9B%D0%BE%D0%B2%D0%B8+%D0%BC%D0%BE%D0%BC%D0%B5%D0%BD%D1%82",
    "https://www.youtube.com/results?search_query=%D0%90%D0%BD%D1%82%D0%B8%D1%82%D1%96%D0%BB%D0%B0+-+%D0%92%D0%B4%D0%BE%D0%BC%D0%B0",
    "https://www.youtube.com/results?search_query=%D0%90%D0%BD%D1%82%D0%B8%D1%82%D1%96%D0%BB%D0%B0+-+%D0%A2%D0%94%D0%9C%D0%95",
    "https://www.youtube.com/results?search_query=%D0%90%D0%BD%D1%82%D0%B8%D1%82%D1%96%D0%BB%D0%B0+-+%D0%9C%D0%B0%D1%8F%D0%BC%D1%96",
    "https://www.youtube.com/results?search_query=%D0%90%D0%BD%D1%82%D0%B8%D1%82%D1%96%D0%BB%D0%B0+-+%D0%A4%D0%B0%D1%80%D0%B8",
    "https://www.youtube.com/results?search_query=%D0%91%D1%83%D0%BC%D0%B1%D0%BE%D0%BA%D1%81+-+%D0%92%D0%B0%D1%85%D1%82%D0%B5%D1%80%D0%B0%D0%BC",
    "https://www.youtube.com/results?search_query=%D0%91%D1%83%D0%BC%D0%B1%D0%BE%D0%BA%D1%81+-+%D0%9D%D0%B0%D0%BE%D0%B4%D0%B8%D0%BD%D1%86%D1%96",
    "https://www.youtube.com/results?search_query=%D0%91%D1%83%D0%BC%D0%B1%D0%BE%D0%BA%D1%81+-+%D0%9B%D1%8E%D0%B4%D0%B8",
    "https://www.youtube.com/results?search_query=%D0%91%D1%83%D0%BC%D0%B1%D0%BE%D0%BA%D1%81+-+%D0%A2%D0%BE4%D1%82%D0%BE",
    "https://www.youtube.com/results?search_query=%D0%91%D1%83%D0%BC%D0%B1%D0%BE%D0%BA%D1%81+-+%D0%9F%D0%BE%D0%BB%D1%96%D0%BD%D0%B0",
    "https://www.youtube.com/results?search_query=%D0%91%D1%83%D0%BC%D0%B1%D0%BE%D0%BA%D1%81+-+%D0%9A%D0%B2%D1%96%D1%82%D0%B8+%D0%B2+%D0%B2%D0%BE%D0%BB%D0%BE%D1%81%D1%81%D1%96",
    "https://www.youtube.com/results?search_query=%D0%9E%D0%BA%D0%B5%D0%B0%D0%BD+%D0%95%D0%BB%D1%8C%D0%B7%D0%B8+-+%D0%91%D0%B5%D0%B7+%D0%B1%D0%BE%D1%8E",
    "https://www.youtube.com/results?search_query=%D0%9E%D0%BA%D0%B5%D0%B0%D0%BD+%D0%95%D0%BB%D1%8C%D0%B7%D0%B8+-+%D0%9D%D0%B5+%D1%82%D0%B2%D0%BE%D1%8F+%D0%B2%D1%96%D0%B9%D0%BD%D0%B0",
    "https://www.youtube.com/results?search_query=%D0%9E%D0%BA%D0%B5%D0%B0%D0%BD+%D0%95%D0%BB%D1%8C%D0%B7%D0%B8+-+%D0%9D%D0%B0+%D0%BD%D0%B5%D0%B1%D1%96",
    "https://www.youtube.com/results?search_query=%D0%9E%D0%BA%D0%B5%D0%B0%D0%BD+%D0%95%D0%BB%D1%8C%D0%B7%D0%B8+-+%D0%AF+%D1%82%D0%B0%D0%BA+%D1%85%D0%BE%D1%87%D1%83",
    "https://www.youtube.com/results?search_query=%D0%9E%D0%BA%D0%B5%D0%B0%D0%BD+%D0%95%D0%BB%D1%8C%D0%B7%D0%B8+-+%D0%9C%D0%B8%D1%82%D1%8C",
    "https://www.youtube.com/results?search_query=%D0%9E%D0%BA%D0%B5%D0%B0%D0%BD+%D0%95%D0%BB%D1%8C%D0%B7%D0%B8+-+%D0%92%D1%81%D0%B5+%D0%B1%D1%83%D0%B4%D0%B5+%D0%B4%D0%BE%D0%B1%D1%80%D0%B5",
    "https://www.youtube.com/results?search_query=%D0%9E%D0%BA%D0%B5%D0%B0%D0%BD+%D0%95%D0%BB%D1%8C%D0%B7%D0%B8+-+%D0%A2%D0%BE%D0%B9+%D0%B4%D0%B5%D0%BD%D1%8C",
    "https://www.youtube.com/results?search_query=%D0%9E%D0%BA%D0%B5%D0%B0%D0%BD+%D0%95%D0%BB%D1%8C%D0%B7%D0%B8+-+%D0%A2%D0%B0%D0%BC%2C+%D0%B4%D0%B5+%D0%BD%D0%B0%D1%81+%D0%BD%D0%B5%D0%BC%D0%B0",
    "https://www.youtube.com/results?search_query=%D0%9E%D0%BA%D0%B5%D0%B0%D0%BD+%D0%95%D0%BB%D1%8C%D0%B7%D0%B8+-+%D0%A1%D1%82%D1%80%D1%96%D0%BB%D1%8F%D0%B9",
    "https://www.youtube.com/results?search_query=%D0%9E%D0%BA%D0%B5%D0%B0%D0%BD+%D0%95%D0%BB%D1%8C%D0%B7%D0%B8+-+%D0%97%D0%B5%D0%BB%D0%B5%D0%BD%D1%96+%D0%BE%D1%87%D1%96",
    "https://www.youtube.com/results?search_query=%D0%9E%D0%BA%D0%B5%D0%B0%D0%BD+%D0%95%D0%BB%D1%8C%D0%B7%D0%B8+-+%D0%9C%D0%B0%D0%B9%D0%B6%D0%B5+%D0%B2%D0%B5%D1%81%D0%BD%D0%B0",
    "https://www.youtube.com/results?search_query=%D0%A1%D0%BA%D1%80%D1%8F%D0%B1%D1%96%D0%BD+-+%D0%A1%D1%82%D0%B0%D1%80%D1%96+%D1%84%D0%BE%D1%82%D0%BE%D0%B3%D1%80%D0%B0%D1%84%D1%96%D1%97",
    "https://www.youtube.com/results?search_query=%D0%A1%D0%BA%D1%80%D1%8F%D0%B1%D1%96%D0%BD+-+%D0%9B%D1%8E%D0%B4%D0%B8+%D1%8F%D0%BA+%D0%BA%D0%BE%D1%80%D0%B0%D0%B1%D0%BB%D1%96",
    "https://www.youtube.com/results?search_query=%D0%A1%D0%BA%D1%80%D1%8F%D0%B1%D1%96%D0%BD+-+%D0%9C%D1%96%D1%81%D1%86%D1%8F+%D1%89%D0%B0%D1%81%D0%BB%D0%B8%D0%B2%D0%B8%D1%85+%D0%BB%D1%8E%D0%B4%D0%B5%D0%B9",
    "https://www.youtube.com/results?search_query=%D0%A1%D0%BA%D1%80%D1%8F%D0%B1%D1%96%D0%BD+-+%D0%A1%D0%BF%D0%B8+%D1%81%D0%BE%D0%B1%D1%96+%D1%81%D0%B0%D0%BC%D0%B0",
    "https://www.youtube.com/results?search_query=%D0%A1%D0%BA%D1%80%D1%8F%D0%B1%D1%96%D0%BD+-+%D0%A8%D0%B0%D0%BC%D0%BF%D0%B0%D0%BD%D1%81%D1%8C%D0%BA%D1%96+%D0%BE%D1%87%D1%96",
    "https://www.youtube.com/results?search_query=%D0%A1%D0%BA%D1%80%D1%8F%D0%B1%D1%96%D0%BD+-+%D0%9C%D0%B0%D0%BC",
    "https://www.youtube.com/results?search_query=%D0%A1%D0%BA%D1%80%D1%8F%D0%B1%D1%96%D0%BD+-+%D0%A8%D1%83%D0%BA%D0%B0%D0%B2+%D1%81%D0%B2%D1%96%D0%B9+%D0%B4%D1%96%D0%BC",
    "https://www.youtube.com/results?search_query=%D0%A1%D0%BA%D1%80%D1%8F%D0%B1%D1%96%D0%BD+-+%D0%9C%D0%B0%D1%80%D1%88%D1%80%D1%83%D1%82%D0%BA%D0%B0",
    "https://www.youtube.com/results?search_query=%D0%A1%D0%BA%D1%80%D1%8F%D0%B1%D1%96%D0%BD+-+%D0%93%D0%BE%D0%B2%D0%BE%D1%80%D0%B8%D0%BB%D0%B8+%D1%96+%D0%BA%D1%83%D1%80%D0%B8%D0%BB%D0%B8",
    "https://www.youtube.com/results?search_query=The+Hardkiss+-+%D0%96%D1%83%D1%80%D0%B0%D0%B2%D0%BB%D1%96",
    "https://www.youtube.com/results?search_query=The+Hardkiss+-+%D0%9F%D1%80%D1%96%D1%80%D0%B2%D0%B0",
    "https://www.youtube.com/results?search_query=The+Hardkiss+-+%D0%96%D0%B8%D0%B2%D0%B0",
    "https://www.youtube.com/results?search_query=The+Hardkiss+-+%D0%90%D0%BD%D1%82%D0%B0%D1%80%D0%BA%D1%82%D0%B8%D0%B4%D0%B0",
    "https://www.youtube.com/results?search_query=The+Hardkiss+-+Make-Up",
    "https://www.youtube.com/results?search_query=The+Hardkiss+-+%D0%9A%D0%BE%D1%80%D0%B0%D0%B1%D0%BB%D1%96",
    "https://www.youtube.com/results?search_query=Go_A+-+SHUM",
    "https://www.youtube.com/watch?v=NBB0DGJSbJM",
    "https://www.youtube.com/results?search_query=Go_A+-+Solovey",
    "https://www.youtube.com/results?search_query=Go_A+-+Kalyna",
    "https://www.youtube.com/results?search_query=Go_A+-+%D0%96%D0%B0%D0%BB%D1%8C",
    "https://www.youtube.com/results?search_query=Tvorchi+-+Heart+of+Steel",
    "https://www.youtube.com/results?search_query=Tvorchi+-+%D0%9C%D0%BE%D0%B2%D0%B0+%D1%82%D1%96%D0%BB%D0%B0",
    "https://www.youtube.com/results?search_query=Tvorchi+-+%D0%91%D0%BE%D1%80%D0%B5%D0%BC%D0%BE%D1%81%D1%8F",
    "https://www.youtube.com/results?search_query=Tvorchi+-+%D0%92%D1%96%D1%87-%D0%BD%D0%B0-%D0%B2%D1%96%D1%87",
    "https://www.youtube.com/results?search_query=Tvorchi+-+%D0%9C%D0%B8%D0%BB%D0%B0+%D0%BC%D0%BE%D1%8F",
    "https://www.youtube.com/results?search_query=%D0%A5%D1%80%D0%B8%D1%81%D1%82%D0%B8%D0%BD%D0%B0+%D0%A1%D0%BE%D0%BB%D0%BE%D0%B2%D1%96%D0%B9+-+%D0%A5%D1%82%D0%BE%2C+%D1%8F%D0%BA+%D0%BD%D0%B5+%D1%82%D0%B8%3F",
    "https://www.youtube.com/results?search_query=%D0%A5%D1%80%D0%B8%D1%81%D1%82%D0%B8%D0%BD%D0%B0+%D0%A1%D0%BE%D0%BB%D0%BE%D0%B2%D1%96%D0%B9+-+Fortepiano",
    "https://www.youtube.com/results?search_query=%D0%A5%D1%80%D0%B8%D1%81%D1%82%D0%B8%D0%BD%D0%B0+%D0%A1%D0%BE%D0%BB%D0%BE%D0%B2%D1%96%D0%B9+-+%D0%9E%D1%81%D1%96%D0%BD%D1%8C",
    "https://www.youtube.com/results?search_query=%D0%A5%D1%80%D0%B8%D1%81%D1%82%D0%B8%D0%BD%D0%B0+%D0%A1%D0%BE%D0%BB%D0%BE%D0%B2%D1%96%D0%B9+-+%D0%9B%D1%8E%D0%B1%D0%B8%D0%B9+%D0%B4%D1%80%D1%83%D0%B3",
    "https://www.youtube.com/results?search_query=%D0%A5%D1%80%D0%B8%D1%81%D1%82%D0%B8%D0%BD%D0%B0+%D0%A1%D0%BE%D0%BB%D0%BE%D0%B2%D1%96%D0%B9+-+%D0%A3%D0%BA%D1%80%D0%B0%D1%97%D0%BD%D1%81%D1%8C%D0%BA%D0%B0+%D0%BB%D1%8E%D1%82%D1%8C",
    "https://www.youtube.com/results?search_query=Jamala+-+1944",
    "https://www.youtube.com/results?search_query=Jamala+-+%D0%9A%D1%80%D0%B8%D0%BB%D0%B0",
    "https://www.youtube.com/results?search_query=Jamala+-+%D0%A8%D0%BB%D1%8F%D1%85+%D0%B4%D0%BE%D0%B4%D0%BE%D0%BC%D1%83",
    "https://www.youtube.com/results?search_query=Jamala+-+Solo",
    "https://www.youtube.com/results?search_query=MELOVIN+-+%D0%A2%D0%B8",
    "https://www.youtube.com/results?search_query=MELOVIN+-+%D0%97%D0%B0%D0%BC%D0%B0%D0%BD%D0%B8%D0%BB%D0%B8",
    "https://www.youtube.com/results?search_query=MELOVIN+-+%D0%9E%D0%B1%D0%B8%D1%80%D0%B0%D1%82%D0%B8%D0%BC%D1%83+%D1%82%D0%B5%D0%B1%D0%B5",
    "https://www.youtube.com/results?search_query=Wellboy+-+%D0%92%D0%B8%D1%88%D0%BD%D1%96",
    "https://www.youtube.com/results?search_query=Wellboy+-+%D0%94%D0%BE%D0%B4%D0%BE%D0%BC%D1%83",
    "https://www.youtube.com/results?search_query=CHEEV+-+%D0%93%D0%B0%D1%80%D0%BD%D0%BE+%D1%82%D0%B0%D0%BA",
    "https://www.youtube.com/results?search_query=CHEEV+-+%D0%A0%D0%B0%D0%BD%D0%B0",
    "https://www.youtube.com/results?search_query=CHEEV+-+%D0%9C%D1%80%D1%96%D1%94%D1%88%D1%81%D1%8F",
    "https://www.youtube.com/results?search_query=CHEEV+-+Good+Luck",
    "https://www.youtube.com/results?search_query=ADAM+-+%D0%9F%D0%BE%D0%B2%D1%96%D0%BB%D1%8C%D0%BD%D0%BE",
    "https://www.youtube.com/results?search_query=ADAM+-+%D0%A1%D0%B8%D0%BB%D1%8C%D0%BD%D0%BE-%D1%81%D0%B8%D0%BB%D1%8C%D0%BD%D0%BE",
    "https://www.youtube.com/results?search_query=ADAM+-+%D0%A2%D0%B0%D0%BA%D1%83+%D1%8F%D0%BA+%D1%94",
    "https://www.youtube.com/results?search_query=ADAM+-+%D0%9B%D1%8E%D0%B1%D0%B0",
    "https://www.youtube.com/results?search_query=MamaRika+-+%D0%AF+%D0%B7%D0%B0%D0%BA%D0%BE%D1%85%D0%B0%D0%BB%D0%B0%D1%81%D1%8F",
    "https://www.youtube.com/results?search_query=MamaRika+-+%D0%A1%D0%B8%D0%BD%D1%83",
    "https://www.youtube.com/results?search_query=MamaRika+-+%D0%9D%D0%B0+%D0%BD%D0%B0%D1%88%D1%96%D0%B9+%D0%B2%D1%83%D0%BB%D0%B8%D1%86%D1%96",
    "https://www.youtube.com/results?search_query=MamaRika+-+%D0%9B%D1%8E%D0%B4%D0%B8+%D1%8F%D0%BA+%D0%BA%D0%BE%D1%80%D0%B0%D0%B1%D0%BB%D1%96",
    "https://www.youtube.com/results?search_query=MamaRika+-+%D0%9A%D0%BE%D0%BB%D0%B8+%D0%BC%D0%B8+%D0%B2%D0%B4%D0%BE%D0%BC%D0%B0",
    "https://www.youtube.com/results?search_query=%D0%90%D0%BD%D0%BD%D0%B0+%D0%A2%D1%80%D1%96%D0%BD%D1%87%D0%B5%D1%80+-+%D0%9C%D0%B5%D1%82%D0%B5%D0%BB%D0%B8%D0%BA%D0%B8",
    "https://www.youtube.com/results?search_query=%D0%90%D0%BD%D0%BD%D0%B0+%D0%A2%D1%80%D1%96%D0%BD%D1%87%D0%B5%D1%80+-+%D0%97%D0%B0%D0%B9",
    "https://www.youtube.com/results?search_query=%D0%90%D0%BD%D0%BD%D0%B0+%D0%A2%D1%80%D1%96%D0%BD%D1%87%D0%B5%D1%80+-+%D0%9C%D0%B0%D1%80%D0%B3%D0%B0%D1%80%D0%B8%D1%82%D0%B0",
    "https://www.youtube.com/results?search_query=%D0%90%D0%BD%D0%BD%D0%B0+%D0%A2%D1%80%D1%96%D0%BD%D1%87%D0%B5%D1%80+-+%D0%9B%D0%B8%D1%88%D0%B5+%D1%82%D0%B8+%D1%96+%D1%8F",
    "https://www.youtube.com/results?search_query=%D0%90%D0%BD%D0%BD%D0%B0+%D0%A2%D1%80%D1%96%D0%BD%D1%87%D0%B5%D1%80+-+%D0%9E%D1%87%D1%96",
    "https://www.youtube.com/results?search_query=Parfeniuk+-+%D0%92%D1%80%D1%83%D0%B1%D0%B0%D0%B9",
    "https://www.youtube.com/results?search_query=Parfeniuk+-+%D0%9F%D1%80%D0%BE%D0%B2%D0%B5%D0%BB%D0%B0+%D0%B5%D0%BA%D1%81%D0%BA%D1%83%D1%80%D1%81%D1%96%D1%8E",
    "https://www.youtube.com/results?search_query=Parfeniuk+-+%D0%92%D1%96%D0%B4%D1%80%D0%B8%D0%B2%D0%B0%D0%B9%D1%81%D1%8F",
    "https://www.youtube.com/results?search_query=Chico+%26+Qatoshi+-+%D0%9B%D0%B0%D1%81%D1%82%D1%96%D0%B2%D0%BA%D0%B8",
    "https://www.youtube.com/results?search_query=Chico+%26+Qatoshi+-+%D0%9F%D0%BE%D0%BA%D0%BE%D1%85%D0%B0%D0%B9+%D0%BC%D0%B5%D0%BD%D0%B5",
    "https://www.youtube.com/results?search_query=100%D0%BB%D0%B8%D1%86%D1%8F+-+%D0%A2%D1%80%D0%BE%D1%8F%D0%BD%D0%B4%D0%B8",
    "https://www.youtube.com/results?search_query=100%D0%BB%D0%B8%D1%86%D1%8F+-+%D0%97%D0%A1%D0%A3",
    "https://www.youtube.com/results?search_query=PROBASS+%26+HARDI+-+%D0%94%D0%BE%D0%B1%D1%80%D0%BE%D0%B3%D0%BE+%D0%B2%D0%B5%D1%87%D0%BE%D1%80%D0%B0",
    "https://www.youtube.com/results?search_query=PROBASS+%26+HARDI+-+%D0%9A%D0%BE%D0%B7%D0%B0%D1%86%D1%8C%D0%BA%D0%BE%D0%BC%D1%83+%D1%80%D0%BE%D0%B4%D1%83",
    "https://www.youtube.com/results?search_query=SHUMEI+-+%D0%A0%D0%BE%D0%B7%D1%81%D1%82%D1%80%D1%96%D0%BB%D1%8F%D0%BD%D0%B5",
    "https://www.youtube.com/results?search_query=SHUMEI+-+%D0%91%D1%83%D1%80%D0%B5%D0%B2%D1%96%D1%8F%D0%BC%D0%B8",
    "https://www.youtube.com/results?search_query=SHUMEI+-+%D0%A2%D1%80%D0%B8%D0%B2%D0%BE%D0%B3%D0%B0",
    "https://www.youtube.com/results?search_query=SHUMEI+-+%D0%92%D1%96%D1%82%D0%B5%D1%80",
    "https://www.youtube.com/results?search_query=DREVO+-+%D0%95%D0%BD%D0%BA%D0%B0%D1%80%D0%B0%D0%BF%D1%96%D1%81%D1%82%D0%B0",
    "https://www.youtube.com/results?search_query=DREVO+-+%D0%A1%D0%B0%D0%BC%D0%BE%D0%BB%D1%96%D1%82%D0%BE%D0%BC",
    "https://www.youtube.com/results?search_query=DREVO+-+%D0%92%D1%96%D1%87%D0%BD%D1%96%D1%81%D1%82%D1%8C",
    "https://www.youtube.com/results?search_query=%D0%9D%D1%96%D0%BA%D1%96%D1%82%D0%B0+%D0%9A%D1%96%D1%81%D0%B5%D0%BB%D1%8C%D0%BE%D0%B2+-+%D0%92%D1%96%D0%B2%D1%82%D0%B0%D1%80",
    "https://www.youtube.com/results?search_query=%D0%9D%D1%96%D0%BA%D1%96%D1%82%D0%B0+%D0%9A%D1%96%D1%81%D0%B5%D0%BB%D1%8C%D0%BE%D0%B2+-+%D0%9A%D1%80%D0%B8%D0%BB%D0%B0+%D0%BB%D0%B5%D0%BB%D0%B5%D0%BA%D0%B8",
    "https://www.youtube.com/results?search_query=%D0%9D%D1%96%D0%BA%D1%96%D1%82%D0%B0+%D0%9A%D1%96%D1%81%D0%B5%D0%BB%D1%8C%D0%BE%D0%B2+-+%D0%A0%D0%BE%D0%BC%D0%B0%D1%88%D0%BA%D0%B0",
    "https://www.youtube.com/results?search_query=Lida+Lee+-+%D0%A1%D1%85%D0%BE%D0%B6%D1%96",
    "https://www.youtube.com/results?search_query=Lida+Lee+-+%D0%9F%D0%BE%D1%8F%D1%81%D0%BD%D0%B8",
    "https://www.youtube.com/results?search_query=Lida+Lee+-+%D0%97%D0%BE%D1%80%D1%96",
    "https://www.youtube.com/results?search_query=Volodymyr+Dantes+-+%D0%A2%D0%B8+%D0%BD%D0%B5+%D0%B7%D0%B0%D0%B1%D1%83%D0%B2%D0%B0%D0%B9",
    "https://www.youtube.com/results?search_query=Volodymyr+Dantes+-+%D0%9E%D0%BB%D1%8F",
    "https://www.youtube.com/results?search_query=Volodymyr+Dantes+-+%D0%9A%D0%B8%D1%82%D0%B8",
    "https://www.youtube.com/results?search_query=POSITIFF+-+%D0%9D%D0%B5+%D0%B7%D1%83%D0%BF%D0%B8%D0%BD%D1%8F%D0%B9+%D0%B2%D0%B5%D1%87%D1%96%D1%80%D0%BA%D1%83",
    "https://www.youtube.com/results?search_query=POSITIFF+-+%D0%A1%D0%BF%D0%B0%D0%BB%D0%B0%D1%85%D0%B8",
    "https://www.youtube.com/results?search_query=POSITIFF+-+%D0%91%D1%83%D0%B4%D1%83+%D0%BA%D0%BE%D1%85%D0%B0%D1%82%D0%B8",
    "https://www.youtube.com/results?search_query=FI%D0%87NKA+-+%D0%92%D1%96%D0%B2%D1%82%D0%B0%D1%80",
    "https://www.youtube.com/results?search_query=FI%D0%87NKA+-+%D0%93%D1%83%D1%86%D1%83%D0%BB%D1%8F%D0%BD%D0%BA%D0%B0",
    "https://www.youtube.com/results?search_query=FI%D0%87NKA+-+%D0%93%D1%80%D1%83%D1%88%D0%B5%D1%87%D0%BA%D0%B0",
    "https://www.youtube.com/results?search_query=VOLIANSKA+-+%D0%9A%D1%83%D1%81%D1%8C",
    "https://www.youtube.com/results?search_query=%D0%A3%D0%BB%D1%8F%D0%BD%D0%B0+%D0%A8%D1%83%D0%B1%D0%B0+-+%D0%9E%D0%B9%2C+%D0%BC%D0%B0%D0%BC",
    "https://www.youtube.com/results?search_query=SKYLERR+-+%D0%93%D0%BE%D1%80%D0%BB%D0%B8%D1%86%D1%96",
    "https://www.youtube.com/results?search_query=SKYLERR+-+%D0%9B%D1%96%D0%B4+%D1%96+%D0%B2%D0%BE%D0%B3%D0%BE%D0%BD%D1%8C",
    "https://www.youtube.com/results?search_query=Alena+Omargalieva+-+%D0%9D%D0%B5+%D0%BF%27%D1%8F%D0%BD%D0%B0+-+%D0%B7%D0%B0%D0%BA%D0%BE%D1%85%D0%B0%D0%BD%D0%B0",
    "https://www.youtube.com/results?search_query=Alena+Omargalieva+-+%D0%A4%D0%B0%D0%BD%D0%B0%D1%82",
    "https://www.youtube.com/results?search_query=YAGODA+-+%D0%97%D0%B0%D0%BA%D0%BE%D1%85%D0%B0%D0%BB%D0%B0%D1%81%D1%8C+%D0%B4%D0%B8%D0%BA%D0%BE",
    "https://www.youtube.com/results?search_query=YAGODA+%26+MELISA+SADYK+-+%D0%A7%D0%BE%D0%B1%D0%BE%D1%82%D0%B8",
    "https://www.youtube.com/results?search_query=YAGODA+-+%D0%AE%D1%80%D0%B0",
    "https://www.youtube.com/results?search_query=%D0%A8%D1%83%D0%B3%D0%B0%D1%80+-+%D0%9E%D0%BB%D1%94%D0%B3",
    "https://www.youtube.com/results?search_query=Kolaba+%26+%D0%86%D0%B2%D0%BE+%D0%91%D0%BE%D0%B1%D1%83%D0%BB+-+%D0%90+%D0%BB%D0%B8%D0%BF%D0%B8+%D1%86%D0%B2%D1%96%D1%82%D1%83%D1%82%D1%8C",
    "https://www.youtube.com/results?search_query=TARABAROVA+-+%D0%9C%D0%B5%D0%B9%D0%B1%D1%96",
    "https://www.youtube.com/results?search_query=CHEEV+-+%D0%9C%D1%96%D1%81%D1%86%D1%8F",
    "https://www.youtube.com/results?search_query=Kalush+Orchestra+%26+SIMBOCHKA+-+%D0%9D%D1%96%D0%BA%D0%BE%D0%BC%D1%83+%D0%BD%D0%B5+%D1%81%D0%BA%D0%B0%D0%B6%D0%B5%D0%BC%D0%BE",
    "https://www.youtube.com/results?search_query=Artem+Pivovarov+%26+The+%D0%92%D1%83%D1%81%D0%B0+%26+%D0%A1%D1%82%D0%B5%D0%BF%D0%B0%D0%BD+%D0%93%D1%96%D0%B3%D0%B0+-+Mamma+Mia",
    "https://www.youtube.com/results?search_query=%D0%9E%D0%B4%D0%B8%D0%BD+%D0%B2+%D0%BA%D0%B0%D0%BD%D0%BE%D0%B5+-+%D0%A7%D0%BE%D0%B2%D0%B5%D0%BD",
    "https://www.youtube.com/results?search_query=%D0%9E%D0%B4%D0%B8%D0%BD+%D0%B2+%D0%BA%D0%B0%D0%BD%D0%BE%D0%B5+-+%D0%A3+%D0%BC%D0%B5%D0%BD%D0%B5+%D0%BD%D0%B5%D0%BC%D0%B0%D1%94+%D0%B4%D0%BE%D0%BC%D1%83",
    "https://www.youtube.com/results?search_query=%D0%9E%D0%B4%D0%B8%D0%BD+%D0%B2+%D0%BA%D0%B0%D0%BD%D0%BE%D0%B5+-+%D0%9D%D0%B5%D0%B1%D0%BE",
    "https://www.youtube.com/results?search_query=%D0%91%D0%95%D0%97+%D0%9E%D0%91%D0%9C%D0%95%D0%96%D0%95%D0%9D%D0%AC+-+%D0%97%D0%BE%D1%80%D1%96+%D0%B7%D0%B0%D0%BF%D0%B0%D0%BB%D0%B0%D0%BB%D0%B8",
    "https://www.youtube.com/results?search_query=%D0%91%D0%95%D0%97+%D0%9E%D0%91%D0%9C%D0%95%D0%96%D0%95%D0%9D%D0%AC+-+%D0%A2%D0%BE%D0%BD%D1%83",
    "https://www.youtube.com/results?search_query=%D0%91%D0%95%D0%97+%D0%9E%D0%91%D0%9C%D0%95%D0%96%D0%95%D0%9D%D0%AC+-+%D0%9C%D1%96%D0%BB%D1%8C%D1%8F%D1%80%D0%B4%D0%B8",
    "https://www.youtube.com/results?search_query=%D0%91%D0%95%D0%97+%D0%9E%D0%91%D0%9C%D0%95%D0%96%D0%95%D0%9D%D0%AC+-+%D0%97%D0%BD%D0%B0%D0%B9%D0%B4%D0%B8+%D0%BC%D0%B5%D0%BD%D0%B5",
    "https://www.youtube.com/results?search_query=%D0%91%D0%95%D0%97+%D0%9E%D0%91%D0%9C%D0%95%D0%96%D0%95%D0%9D%D0%AC+-+%D0%A5%D0%BE%D1%87%D0%B5%D1%88",
    "https://www.youtube.com/results?search_query=%D0%91%D0%95%D0%97+%D0%9E%D0%91%D0%9C%D0%95%D0%96%D0%95%D0%9D%D0%AC+-+%D0%94%D0%B8%D0%BC",
    "https://www.youtube.com/results?search_query=Pianoboy+-+%D0%9A%D0%BE%D1%85%D0%B0%D0%BD%D0%BD%D1%8F",
    "https://www.youtube.com/results?search_query=Pianoboy+-+%D0%A0%D0%BE%D0%B4%D0%B8%D0%BC%D0%BA%D0%B8",
    "https://www.youtube.com/results?search_query=Pianoboy+-+%D0%A8%D0%B0%D0%BC%D0%BF%D0%B0%D0%BD%D1%81%D1%8C%D0%BA%D1%96+%D0%BE%D1%87%D1%96",
    "https://www.youtube.com/results?search_query=Latexfauna+-+Surfer",
    "https://www.youtube.com/results?search_query=Latexfauna+-+Bounty",
    "https://www.youtube.com/results?search_query=Latexfauna+-+Aubergine",
    "https://www.youtube.com/results?search_query=%D0%9E.Torvald+-+%D0%92%D0%B8%D1%80%D0%B2%D0%B0%D0%BD%D0%B8%D0%B9",
    "https://www.youtube.com/results?search_query=%D0%9E.Torvald+-+%D0%9C%D0%BE%D0%B2%D1%87%D0%B8",
    "https://www.youtube.com/results?search_query=%D0%9E.Torvald+-+%D0%A4%D0%BE%D1%82%D0%BE%D0%B3%D1%80%D0%B0%D1%84%D1%83%D0%B9",
    "https://www.youtube.com/results?search_query=%D0%92%D0%BE%D0%BF%D0%BB%D1%96+%D0%92%D1%96%D0%B4%D0%BE%D0%BF%D0%BB%D1%8F%D1%81%D0%BE%D0%B2%D0%B0+-+%D0%92%D0%B5%D1%81%D0%BD%D0%B0",
    "https://www.youtube.com/results?search_query=%D0%92%D0%BE%D0%BF%D0%BB%D1%96+%D0%92%D1%96%D0%B4%D0%BE%D0%BF%D0%BB%D1%8F%D1%81%D0%BE%D0%B2%D0%B0+-+%D0%94%D0%B5%D0%BD%D1%8C+%D0%BD%D0%B0%D1%80%D0%BE%D0%B4%D0%B6%D0%B5%D0%BD%D0%BD%D1%8F",
    "https://www.youtube.com/results?search_query=%D0%92%D0%BE%D0%BF%D0%BB%D1%96+%D0%92%D1%96%D0%B4%D0%BE%D0%BF%D0%BB%D1%8F%D1%81%D0%BE%D0%B2%D0%B0+-+%D0%A2%D0%B0%D1%94%D0%BC%D0%BD%D0%B8%D1%86%D1%96",
    "https://www.youtube.com/results?search_query=%D0%9F%D0%BB%D0%B0%D1%87+%D0%84%D1%80%D0%B5%D0%BC%D1%96%D1%97+-+%D0%92%D0%BE%D0%BD%D0%B0",
    "https://www.youtube.com/results?search_query=%D0%9F%D0%BB%D0%B0%D1%87+%D0%84%D1%80%D0%B5%D0%BC%D1%96%D1%97+-+%D0%A1%D1%82%D0%B0%D1%80%D1%96+%D1%84%D0%BE%D1%82%D0%BE%D0%B3%D1%80%D0%B0%D1%84%D1%96%D1%97",
    "https://www.youtube.com/results?search_query=%D0%94%D1%80%D1%83%D0%B3%D0%B0+%D0%A0%D1%96%D0%BA%D0%B0+-+%D0%A2%D1%80%D0%B8+%D1%85%D0%B2%D0%B8%D0%BB%D0%B8%D0%BD%D0%B8",
    "https://www.youtube.com/results?search_query=%D0%94%D1%80%D1%83%D0%B3%D0%B0+%D0%A0%D1%96%D0%BA%D0%B0+-+%D0%A2%D0%B0%D0%BA+%D0%BC%D0%B0%D0%BB%D0%BE+%D1%82%D1%83%D1%82+%D1%82%D0%B5%D0%B1%D0%B5",
    "https://www.youtube.com/results?search_query=%D0%94%D1%80%D1%83%D0%B3%D0%B0+%D0%A0%D1%96%D0%BA%D0%B0+-+%D0%92%D0%B6%D0%B5+%D0%BD%D0%B5+%D1%81%D0%B0%D0%BC",
    "https://www.youtube.com/results?search_query=%D0%A1%D0%9A%D0%90%D0%99+-+%D0%A2%D0%B5%D0%B1%D0%B5+%D1%86%D0%B5+%D0%BC%D0%BE%D0%B6%D0%B5+%D0%B2%D0%B1%D0%B8%D1%82%D0%B8",
    "https://www.youtube.com/results?search_query=%D0%A1%D0%9A%D0%90%D0%99+-+%D0%9F%D0%BE%D0%B4%D0%B0%D1%80%D1%83%D0%B9+%D0%BC%D0%B5%D0%BD%D1%96+%D0%BB%D1%8E%D0%B1%D0%BE%D0%B2",
    "https://www.youtube.com/results?search_query=%D0%A1%D0%9A%D0%90%D0%99+-+Best+%D0%94%D1%80%D1%83%D0%B3",
    "https://www.youtube.com/results?search_query=%D0%91%D1%80%D0%B0%D1%82%D0%B8+%D0%93%D0%B0%D0%B4%D1%8E%D0%BA%D1%96%D0%BD%D0%B8+-+%D0%9D%D0%B0%D1%80%D0%BA%D0%BE%D0%BC%D0%B0%D0%BD%D0%B8+%D0%BD%D0%B0+%D0%B3%D0%BE%D1%80%D0%BE%D0%B4%D1%96",
    "https://www.youtube.com/results?search_query=%D0%91%D1%80%D0%B0%D1%82%D0%B8+%D0%93%D0%B0%D0%B4%D1%8E%D0%BA%D1%96%D0%BD%D0%B8+-+%D0%A4%D0%B0%D0%B9%D0%BD%D0%B5+%D0%BC%D1%96%D1%81%D1%82%D0%BE+%D0%A2%D0%B5%D1%80%D0%BD%D0%BE%D0%BF%D1%96%D0%BB%D1%8C",
    "https://www.youtube.com/results?search_query=%D0%86%D1%80%D0%B8%D0%BD%D0%B0+%D0%91%D1%96%D0%BB%D0%B8%D0%BA+-+%D0%A2%D0%B0%D0%BA+%D0%BF%D1%80%D0%BE%D1%81%D1%82%D0%BE",
    "https://www.youtube.com/results?search_query=%D0%86%D1%80%D0%B8%D0%BD%D0%B0+%D0%91%D1%96%D0%BB%D0%B8%D0%BA+-+%D0%90+%D1%8F+%D0%BF%D0%BB%D0%B8%D0%B2%D1%83",
    "https://www.youtube.com/results?search_query=%D0%86%D1%80%D0%B8%D0%BD%D0%B0+%D0%91%D1%96%D0%BB%D0%B8%D0%BA+-+%D0%A2%D0%B8+%D0%BC%D1%96%D0%B9",
    "https://www.youtube.com/results?search_query=%D0%9D%D0%B0%D1%82%D0%B0%D0%BB%D0%BA%D0%B0+%D0%9A%D0%B0%D1%80%D0%BF%D0%B0+-+%D0%9A%D0%B0%D0%BB%D0%B0%D0%BC%D0%B1%D1%83%D1%80",
    "https://www.youtube.com/results?search_query=%D0%9D%D0%B0%D1%82%D0%B0%D0%BB%D0%BA%D0%B0+%D0%9A%D0%B0%D1%80%D0%BF%D0%B0+-+%D0%9D%D0%B5+%D0%BF%D1%96%D0%B4%D0%B2%D0%B5%D0%B4%D0%B8",
    "https://www.youtube.com/results?search_query=%D0%A0%D1%83%D1%81%D0%BB%D0%B0%D0%BD%D0%B0+-+%D0%94%D0%B8%D0%BA%D1%96+%D1%82%D0%B0%D0%BD%D1%86%D1%96",
    "https://www.youtube.com/results?search_query=%D0%A0%D1%83%D1%81%D0%BB%D0%B0%D0%BD%D0%B0+-+%D0%97%D0%BD%D0%B0%D1%8E+%D1%8F",
    "https://www.youtube.com/results?search_query=KAZKA+-+%D0%A1%D0%B2%D1%8F%D1%82%D0%B0",
    "https://www.youtube.com/results?search_query=KAZKA+-+%D0%9C%27%D1%8F%D1%82%D0%B0",
    "https://www.youtube.com/results?search_query=KAZKA+-+%D0%9A%D0%BE%D0%BB%D1%8C%D0%BE%D1%80%D0%BE%D0%B2%D1%96",
    "https://www.youtube.com/results?search_query=NK+-+%D0%9E%D0%B1%D1%96%D1%86%D1%8F%D1%8E",
    "https://www.youtube.com/results?search_query=NK+-+%D0%A7%D0%B5%D1%80%D0%B2%D0%BE%D0%BD%D0%B5+%D0%B2%D0%B8%D0%BD%D0%BE",
    "https://www.youtube.com/results?search_query=NK+-+Elefante",
    "https://www.youtube.com/results?search_query=NK+-+%D0%9F%D0%BE%D0%BF%D0%B0+%D1%8F%D0%BA+%D1%83+%D0%9A%D1%96%D0%BC",
    "https://www.youtube.com/results?search_query=LAUD+-+%D0%A1%D0%B2%D1%96%D1%82%D0%BB%D0%BE%D0%BE%D1%85%D0%BE%D1%80%D0%BE%D0%BD%D0%B5%D1%86%D1%8C",
    "https://www.youtube.com/results?search_query=LAUD+-+2+%D0%B4%D0%BD%D1%96",
    "https://www.youtube.com/results?search_query=LAUD+-+%D0%A3+%D1%86%D1%8E+%D0%BD%D1%96%D1%87",
    "https://www.youtube.com/results?search_query=%D0%86%D0%B2%D0%B0%D0%BD+NAVI+-+%D0%A2%D0%B0%D0%BA%D1%96+%D0%BC%D0%BE%D0%BB%D0%BE%D0%B4%D1%96",
    "https://www.youtube.com/results?search_query=%D0%86%D0%B2%D0%B0%D0%BD+NAVI+-+%D0%A2%D0%B8%D0%BC%D1%87%D0%B0%D1%81%D0%BE%D0%B2%D0%B8%D0%B9+%D1%80%D0%B5%D0%BB%D0%B0%D0%BA%D1%81",
    "https://www.youtube.com/results?search_query=%D0%86%D0%B2%D0%B0%D0%BD+NAVI+-+%D0%A5%D1%96%D0%BC%D1%96%D1%8F",
    "https://www.youtube.com/results?search_query=%D0%94%D0%BC%D0%B8%D1%82%D1%80%D0%BE+%D0%A8%D1%83%D1%80%D0%BE%D0%B2+-+%D0%9A%D0%BE%D1%85%D0%B0%D0%BD%D0%BD%D1%8F",
    "https://www.youtube.com/results?search_query=%D0%9E%D0%BB%D1%8F+%D0%9F%D0%BE%D0%BB%D1%8F%D0%BA%D0%BE%D0%B2%D0%B0+-+%D0%9A%D0%BE%D1%80%D0%BE%D0%BB%D0%B5%D0%B2%D0%B0+%D0%BD%D0%BE%D1%87%D1%96",
    "https://www.youtube.com/results?search_query=%D0%9E%D0%BB%D1%8F+%D0%9F%D0%BE%D0%BB%D1%8F%D0%BA%D0%BE%D0%B2%D0%B0+-+%D0%A8%D0%BB%D1%8C%D0%BE%D0%BF%D0%BA%D0%B8",
    "https://www.youtube.com/results?search_query=%D0%9D%D0%B0%D1%81%D1%82%D1%8F+%D0%9A%D0%B0%D0%BC%D0%B5%D0%BD%D1%81%D1%8C%D0%BA%D0%B8%D1%85+-+%D0%9E%D0%B1%D1%96%D1%86%D1%8F%D1%8E",
    "https://www.youtube.com/results?search_query=%D0%9D%D0%B0%D1%81%D1%82%D1%8F+%D0%9A%D0%B0%D0%BC%D0%B5%D0%BD%D1%81%D1%8C%D0%BA%D0%B8%D1%85+-+%D0%9F%D0%BE%D0%BF%D0%B0+%D0%BA%D0%B0%D0%BA+%D1%83+%D0%9A%D0%B8%D0%BC",
    "https://www.youtube.com/results?search_query=%D0%9D%D0%B0%D1%81%D1%82%D1%8F+%D0%9A%D0%B0%D0%BC%D0%B5%D0%BD%D1%81%D1%8C%D0%BA%D0%B8%D1%85+-+%D0%9A%D1%80%D0%B0%D1%81%D0%BD%D0%BE%D0%B5+%D0%B2%D0%B8%D0%BD%D0%BE",
    "https://www.youtube.com/results?search_query=MONATIK+%26+DOROFEEVA+-+%D0%93%D0%BB%D1%83%D0%B1%D0%BE%D0%BA%D0%BE",
    "https://www.youtube.com/results?search_query=MONATIK+%26+%D0%92%D1%96%D1%80%D0%B0+%D0%91%D1%80%D0%B5%D0%B6%D0%BD%D1%94%D0%B2%D0%B0+-+%D0%92%D1%96%D1%80%D1%83%D1%8E",
    "https://www.youtube.com/results?search_query=%D0%90%D1%80%D1%81%D0%B5%D0%BD+%D0%9C%D1%96%D1%80%D0%B7%D0%BE%D1%8F%D0%BD+-+%D0%86%D0%B2%D0%B0%D0%BD%D0%B0+%D0%9A%D1%83%D0%BF%D0%B0%D0%BB%D0%B0",
    "https://www.youtube.com/results?search_query=%D0%90%D1%80%D1%81%D0%B5%D0%BD+%D0%9C%D1%96%D1%80%D0%B7%D0%BE%D1%8F%D0%BD+-+%D0%9F%D0%BE%D1%86%D1%96%D0%BB%D1%83%D0%B9+%D0%BC%D0%B5%D0%BD%D0%B5",
    "https://www.youtube.com/results?search_query=%D0%A2%D1%96%D0%BD%D0%B0+%D0%9A%D0%B0%D1%80%D0%BE%D0%BB%D1%8C+-+%D0%9D%D0%B0%D0%BC%D0%B0%D0%BB%D1%8E%D1%8E+%D1%82%D0%BE%D0%B1%D1%96+%D0%B7%D0%BE%D1%80%D1%96",
    "https://www.youtube.com/results?search_query=%D0%A1%D0%B5%D1%80%D0%B3%D1%96%D0%B9+%D0%91%D0%B0%D0%B1%D0%BA%D1%96%D0%BD+-+%D0%94%D0%B5+%D0%B1%D0%B8+%D1%8F",
    "https://www.youtube.com/results?search_query=%D0%A1%D0%B5%D1%80%D0%B3%D1%96%D0%B9+%D0%91%D0%B0%D0%B1%D0%BA%D1%96%D0%BD+-+%D0%9F%D1%80%D0%BE%D0%B1%D0%B0%D1%87",
    "https://www.youtube.com/results?search_query=%D0%A1%D0%B5%D1%80%D0%B3%D1%96%D0%B9+%D0%91%D0%B0%D0%B1%D0%BA%D1%96%D0%BD+-+%D0%9C%D0%BE%D1%8F+%D0%BB%D1%8E%D0%B1%D0%BE%D0%B2",
    "https://www.youtube.com/results?search_query=%D0%9E%D0%BB%D0%B5%D0%BA%D1%81%D0%B0%D0%BD%D0%B4%D1%80+%D0%9F%D0%BE%D0%BD%D0%BE%D0%BC%D0%B0%D1%80%D1%8C%D0%BE%D0%B2+-+%D0%AF+%D0%BB%D1%8E%D0%B1%D0%BB%D1%8E+%D1%82%D1%96%D0%BB%D1%8C%D0%BA%D0%B8+%D1%82%D0%B5%D0%B1%D0%B5",
    "https://www.youtube.com/results?search_query=%D0%9E%D0%BB%D0%B5%D0%BA%D1%81%D0%B0%D0%BD%D0%B4%D1%80+%D0%9F%D0%BE%D0%BD%D0%BE%D0%BC%D0%B0%D1%80%D1%8C%D0%BE%D0%B2+-+%D0%92%D0%B0%D1%80%D1%82%D0%BE",
    "https://www.youtube.com/results?search_query=%D0%92%D1%96%D1%82%D0%B0%D0%BB%D1%96%D0%B9+%D0%9A%D0%BE%D0%B7%D0%BB%D0%BE%D0%B2%D1%81%D1%8C%D0%BA%D0%B8%D0%B9+-+%D0%9F%D1%96%D0%BD%D0%B0+%D0%B4%D0%BD%D1%96%D0%B2",
    "https://www.youtube.com/results?search_query=%D0%92%D1%96%D1%82%D0%B0%D0%BB%D1%96%D0%B9+%D0%9A%D0%BE%D0%B7%D0%BB%D0%BE%D0%B2%D1%81%D1%8C%D0%BA%D0%B8%D0%B9+-+%D0%9C%D0%BE%D1%94+%D0%BC%D0%BE%D1%80%D0%B5",
    "https://www.youtube.com/results?search_query=%D0%92%D1%96%D1%82%D0%B0%D0%BB%D1%96%D0%B9+%D0%9A%D0%BE%D0%B7%D0%BB%D0%BE%D0%B2%D1%81%D1%8C%D0%BA%D0%B8%D0%B9+-+%D0%A2%D1%96%D0%BB%D1%8C%D0%BA%D0%B8+%D0%BA%D0%BE%D1%85%D0%B0%D0%BD%D0%BD%D1%8F",
]

RANDOM_SONGS_URLS = [
    "https://www.youtube.com/watch?v=pgN-vvVVxMA",
    "https://www.youtube.com/watch?v=P1t9T1TAOBI",
    "https://www.youtube.com/watch?v=GX8Hg6kWQYI",
    "https://www.youtube.com/watch?v=iAeYPfrXwk4",
    "https://www.youtube.com/watch?v=nyxRebRhaK0",
    "https://www.youtube.com/watch?v=maigqMT9KPw&t=1716s",
    "https://www.youtube.com/watch?v=zq2pagG8_ok",
    "https://www.youtube.com/watch?v=LqxcHcdGkvM",
    "https://www.youtube.com/watch?v=MIuJVYNvC-s",
    "https://www.youtube.com/watch?v=GFihEj6DAtM",
    "https://www.youtube.com/watch?v=W0DM5lcj6mw",
    "https://www.youtube.com/watch?v=SA7AIQw-7Ms",
    "https://www.youtube.com/watch?v=nidQCt_HEsY",
    "https://www.youtube.com/watch?v=cEQ6-8J-P3I",
    "https://www.youtube.com/watch?v=oPA0z4W-kcU ",
    "https://www.youtube.com/watch?v=Na_LRWNbyJ8",
    "https://www.youtube.com/watch?v=6PNPx0koe2E&list=PLQdn7YisXz3PS9dJjn3H35ohAkXleMoVM",
    "https://www.youtube.com/watch?v=mZBtA4iQZmQ",
    "https://www.youtube.com/watch?v=5Fv19KVVya8",
    "https://www.youtube.com/watch?v=04x3RshrpG8",
    "https://www.youtube.com/watch?v=feGgQxcNYZ0",
    "https://www.youtube.com/watch?v=0_wQc-6uAME",
    "https://www.youtube.com/watch?v=zQ7Zrowa-gY",
    "https://www.youtube.com/watch?v=MBG3Gdt5OGs",
    "https://www.youtube.com/watch?v=WJF5Z1WRcqw",
    "https://www.youtube.com/watch?v=yH5FTh3OfPE",
    "https://www.youtube.com/watch?v=NZffGy4TLno",
    "https://www.youtube.com/watch?v=pY-WQxgx2b8",
    "https://www.youtube.com/watch?v=i3U9Eesh-Ys",
    "https://www.youtube.com/watch?v=E2nDCLsGsqQ",
    "https://www.youtube.com/watch?v=9CDJpnIDS4c",
    "https://www.youtube.com/watch?v=l7v8DAbIOx0&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&start_radio=1&rv=9CDJpnIDS4c",
    "https://www.youtube.com/watch?v=hgNaOU0UOaQ&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=2",
    "https://www.youtube.com/watch?v=3rkJ3L5Ce80&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=3",
    "https://www.youtube.com/watch?v=p1uh40IvF4c&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=4",
    "https://www.youtube.com/watch?v=IuG3PhJOKRM&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=5",
    "https://www.youtube.com/watch?v=pzv8DhGXOa8&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=6",
    "https://www.youtube.com/watch?v=uHu28rVAg_I&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=7",
    "https://www.youtube.com/watch?v=IyuJP9S68n0&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=8",
    "https://www.youtube.com/watch?v=Qzm44kVp7QA&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=9",
    "https://www.youtube.com/watch?v=dZ9v9PYCWoM&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=10",
    "https://www.youtube.com/watch?v=XALLZHKnS_U&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=11",
    "https://www.youtube.com/watch?v=GRvRIS--JRo&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=12",
    "https://www.youtube.com/watch?v=unrs1wnz0r4&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=13",
    "https://www.youtube.com/watch?v=dqdbVlU1f0M&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=14",
    "https://www.youtube.com/watch?v=VbqVh6iUFS4&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=15",
    "https://www.youtube.com/watch?v=zuBKLzYirMc&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=16",
    "https://www.youtube.com/watch?v=napUAIh4UE0&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=17",
    "https://www.youtube.com/watch?v=jWDwdYSdM64&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=18",
    "https://www.youtube.com/watch?v=c-5jld1Uf0g&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=19",
    "https://www.youtube.com/watch?v=wDsU4H2w48k&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=20",
    "https://www.youtube.com/watch?v=eA4_E4Qvsw0&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=21",
    "https://www.youtube.com/watch?v=PhHI1bXG3Yo&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=22",
    "https://www.youtube.com/watch?v=aD1IyIBdqjQ&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=23",
    "https://www.youtube.com/watch?v=FZ-he-64ms4&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=24",
    "https://www.youtube.com/watch?v=PtqM4ThqZUc&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=25",
    "https://www.youtube.com/watch?v=lsH2M6OcS_g&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=26",
    "https://www.youtube.com/watch?v=gCDPTnpYvtc&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=29",
    "https://www.youtube.com/watch?v=-q68__Slg-8&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=27",
    "https://www.youtube.com/watch?v=szEgSYloU-0&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=27",
    "https://www.youtube.com/watch?v=6i8eq93gYHM&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=28",
    "https://www.youtube.com/watch?v=ZOI8ib7k4Ic&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=27",
    "https://www.youtube.com/watch?v=BFSpI9aJ4As&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=27",
    "https://www.youtube.com/watch?v=KpPcmkuYjfw&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=27",
    "https://www.youtube.com/watch?v=BRtQfVtAHxY&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=27",
    "https://www.youtube.com/watch?v=EsX-0VBb0j0&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=27",
    "https://www.youtube.com/watch?v=tMfVDZZno88&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=27",
    "https://www.youtube.com/watch?v=hQO0Adj-tfw&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=38",
    "https://www.youtube.com/watch?v=-Rdo49VtEJs&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=34",
    "https://www.youtube.com/watch?v=MyB3AlaCSDU&list=RDEMMyKIrFtyXwr_Bmk8U4B2zg&index=50",
    "https://www.youtube.com/watch?v=DDPecetdvMM",
]


# ============================================================
# ОСНОВНЫЕ ФУНКЦИИ
# ============================================================

def minimize_all_windows():
    """Сворачивает все открытые окна (Win + M)."""
    if not play_sound("minimize_all") and not play_sound("minimize"):
        play_sound("ok")

    # Используем Win + M для сворачивания всех окон
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

    # Находим окно по имени процесса
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
    url = random.choice(UKRAINIAN_SONGS_URLS)
    print(f"[Музыка]: Рандомно выбрана украинская песня -> {url}")
    webbrowser.open(url)


def play_random_song():
    """Воспроизводит случайную песню на YouTube."""
    if not play_sound("random_song") and not play_sound("random"):
        play_sound("ok")
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


# Добавьте эту функцию после импортов
def send_ctrl_a():
    """Отправляет Ctrl+A несколькими способами для гарантии"""
    try:
        # Способ 1: через pyautogui (обычный)
        pyautogui.hotkey('ctrl', 'a')
        return True
    except Exception as e1:
        print(f"[PyAutoGUI Ctrl+A] Ошибка: {e1}")
        try:
            # Способ 2: через keyboard (более низкий уровень)
            keyboard.press('ctrl')
            time.sleep(0.05)
            keyboard.press('a')
            time.sleep(0.05)
            keyboard.release('a')
            time.sleep(0.05)
            keyboard.release('ctrl')
            return True
        except Exception as e2:
            print(f"[Keyboard Ctrl+A] Ошибка: {e2}")
            try:
                # Способ 3: через keybd_event (Win API) - самый надёжный
                import ctypes
                from ctypes import wintypes

                # Константы клавиш
                VK_CONTROL = 0x11
                VK_A = 0x41
                KEYEVENTF_KEYDOWN = 0x0000
                KEYEVENTF_KEYUP = 0x0002

                user32 = ctypes.windll.user32

                # Нажимаем Ctrl
                user32.keybd_event(VK_CONTROL, 0, KEYEVENTF_KEYDOWN, 0)
                time.sleep(0.05)
                # Нажимаем A
                user32.keybd_event(VK_A, 0, KEYEVENTF_KEYDOWN, 0)
                time.sleep(0.05)
                # Отпускаем A
                user32.keybd_event(VK_A, 0, KEYEVENTF_KEYUP, 0)
                time.sleep(0.05)
                # Отпускаем Ctrl
                user32.keybd_event(VK_CONTROL, 0, KEYEVENTF_KEYUP, 0)
                return True
            except Exception as e3:
                print(f"[WinAPI Ctrl+A] Ошибка: {e3}")
                return False


CONTROL_COMMANDS = {
    "вверх": lambda: pyautogui.scroll(400),
    "скролл вверх": lambda: pyautogui.scroll(400),
    "вниз": lambda: pyautogui.scroll(-400),
    "скролл вниз": lambda: pyautogui.scroll(-400),
    "клик": lambda: pyautogui.click(),
    "нажми": lambda: pyautogui.click(),
    "дабл клик": lambda: pyautogui.doubleClick(),
    "двойной клик": lambda: pyautogui.doubleClick(),
    "пауза": lambda: pyautogui.press("space"),
    "выдели все": lambda:  keyboard.press_and_release('ctrl+a'),
    "выдели": lambda: keyboard.press_and_release('ctrl+a'),
    "скопировать": lambda: keyboard.press_and_release('ctrl+c'),
    "скопируй": lambda: keyboard.press_and_release('ctrl+c'),
    "копируй": lambda: keyboard.press_and_release('ctrl+c'),
    "вставить": lambda: keyboard.press_and_release('ctrl+v'),
    "вставь":lambda: keyboard.press_and_release('ctrl+v'),
    "отправь": lambda: keyboard.press_and_release('enter'),
    "скрин": lambda: pyautogui.press("printscreen"),
    "скриншот": lambda: pyautogui.press("printscreen"),
    "пробел": lambda: pyautogui.press("space"),
    "полный экран": lambda: pyautogui.doubleClick(),
    "на весь экран": lambda: pyautogui.doubleClick(),
    "фулл экран": lambda: pyautogui.doubleClick(),
    "смени язык": change_language,
    "измени язык": change_language,
    "поменяй язык": change_language,
    "переключи язык": change_language,
}


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


def get_available_drives() -> list[str]:
    drives = []
    for letter in string.ascii_uppercase:
        drive_path = f"{letter}:\\"
        if os.path.exists(drive_path):
            drives.append(drive_path)
    return drives


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

    # ИСПОЛЬЗУЕМ PROCESS_KILL_MAP для закрытия
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


def process_command(text: str):
    cleaned = text.lower().strip()
    words = cleaned.split()

    if not any(tw in words for tw in TRIGGER_WORDS):
        return

    print(f"\n[Распознано]: {text}")

    # Убираем триггер-слова
    filtered_words = [w for w in words if w not in TRIGGER_WORDS]

    # Убираем мат и стоп-слова (важно!)
    filtered_words = [w for w in filtered_words if w not in BAD_WORDS]

    if not filtered_words:
        print("[Стелла]: Слушаю вас!")
        play_sound("da")
        return

    phrase_after_trigger = " ".join(filtered_words).strip()

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

    # 1. ПРИОРИТЕТНЫЕ МУЗЫКАЛЬНЫЕ КОМАНДЫ
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

    # 3. Обработка команды "напиши" (печать текста) через keyboard
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

    # 4.5. Открытие веб-сайтов
    for verb in WEB_VERBS:
        if verb in filtered_words:
            target = " ".join([w for w in filtered_words if w != verb]).strip()
            if target:
                print(f"[Стелла]: Открываю сайт -> {target}")
                open_website(target)
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

    # 7. Команды управления интерфейсом
    if phrase_after_trigger in CONTROL_COMMANDS:
        print(f"[Стелла]: Выполнение команды -> '{phrase_after_trigger}'")
        CONTROL_COMMANDS[phrase_after_trigger]()
        return

    # 8. Разговорные ответы
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
    print("Команды:")
    print("  'Стелла НАПИШИ [текст]'      -> Мгновенная печать текста")
    print("  'Стелла СВЕРНИ ВСЕ ОКНА'     -> Свернуть все окна (Win + M)")
    print("  'Стелла СВЕРНИ [прогу]'      -> Свернуть конкретное окно")
    print("  'Стелла ВКЛЮЧИ украинскую песню' -> Укр треки на YouTube")
    print("  'Стелла ВКЛЮЧИ рандомную песню'  -> Рандомный трек")
    print("  'Стелла ОТКРОЙ [сайт]'       -> Сайты")
    print("  'Стелла ЗАПУСТИ [прогу]'     -> Программы (.exe)")
    print("  'Стелла ЗАКРОЙ [прогу]'      -> Завершить процесс")
    print("  'Стелла ПЕРЕВЕДИ окно на 2 монитор' -> Перемещение окна")
    print("  'Стелла ИЗМЕНИ ЯЗЫК'         -> Переключить язык")
    print("=" * 60)
    print("💡 Новые команды сворачивания:")
    print("   - 'Стелла СВЕРНИ ВСЕ ОКНА' - сворачивает все окна (Win + M)")
    print("   - 'Стелла СВЕРНИ ДИСКОРД'  - сворачивает конкретное окно")
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
