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
import tkinter as tk
from tkinter import scrolledtext, ttk, messagebox, Menu, filedialog
import math
import sys
from datetime import datetime, timedelta
from groq import Groq

# ============================================================
# ДОПОЛНИТЕЛЬНЫЕ ИМПОРТЫ ДЛЯ ВОСПРОИЗВЕДЕНИЯ МУЗЫКИ
# ============================================================
try:
    import yt_dlp
    HAS_YDL = True
except ImportError:
    HAS_YDL = False

try:
    import vlc
    HAS_VLC = True
except ImportError:
    HAS_VLC = False

# ============================================================
# ЕСТЕСТВЕННЫЙ ЖЕНСКИЙ ГОЛОС MITA (Microsoft Edge TTS)
# ============================================================
try:
    import asyncio
    import edge_tts
    import pygame
    HAS_EDGE_TTS = True
except ImportError:
    HAS_EDGE_TTS = False

# ============================================================
# НАСТРОЙКИ ДЛЯ COOKIES YOUTUBE
# ============================================================

def get_base_path():
    if getattr(sys, 'frozen', False):
        return os.path.dirname(sys.executable)
    else:
        return os.path.dirname(os.path.abspath(__file__))

BASE_DIR = get_base_path()
COOKIES_DIR = os.path.join(BASE_DIR, "YouCookie")
COOKIES_FILE = os.path.join(COOKIES_DIR, "cookies.txt")

# Создаем папку для cookies если её нет
if not os.path.exists(COOKIES_DIR):
    try:
        os.makedirs(COOKIES_DIR)
        print(f"📁 Создана папка для cookies: {COOKIES_DIR}")
    except Exception as e:
        print(f"⚠️ Не удалось создать папку для cookies: {e}")

# Функция для проверки наличия cookies
def has_cookies():
    return os.path.exists(COOKIES_FILE) and os.path.getsize(COOKIES_FILE) > 0

# Функция для получения cookies из браузера
def get_cookies_from_browser():
    """Пытается получить cookies из браузера с помощью yt-dlp"""
    try:
        if not HAS_YDL:
            return False
        
        # Пробуем получить cookies из Chrome
        try:
            result = subprocess.run(
                ["yt-dlp", "--cookies-from-browser", "chrome", "--cookies", COOKIES_FILE, "--simulate", "https://www.youtube.com/watch?v=dQw4w9WgXcQ"],
                capture_output=True,
                text=True,
                timeout=30
            )
            if os.path.exists(COOKIES_FILE) and os.path.getsize(COOKIES_FILE) > 0:
                print("✅ Cookies успешно получены из Chrome")
                return True
        except:
            pass
        
        # Пробуем Firefox
        try:
            result = subprocess.run(
                ["yt-dlp", "--cookies-from-browser", "firefox", "--cookies", COOKIES_FILE, "--simulate", "https://www.youtube.com/watch?v=dQw4w9WgXcQ"],
                capture_output=True,
                text=True,
                timeout=30
            )
            if os.path.exists(COOKIES_FILE) and os.path.getsize(COOKIES_FILE) > 0:
                print("✅ Cookies успешно получены из Firefox")
                return True
        except:
            pass
        
        # Пробуем Edge
        try:
            result = subprocess.run(
                ["yt-dlp", "--cookies-from-browser", "edge", "--cookies", COOKIES_FILE, "--simulate", "https://www.youtube.com/watch?v=dQw4w9WgXcQ"],
                capture_output=True,
                text=True,
                timeout=30
            )
            if os.path.exists(COOKIES_FILE) and os.path.getsize(COOKIES_FILE) > 0:
                print("✅ Cookies успешно получены из Edge")
                return True
        except:
            pass
            
        return False
    except Exception as e:
        print(f"⚠️ Ошибка получения cookies: {e}")
        return False

# Функция для показа окна ввода cookies
def show_cookies_input_window(parent=None):
    """Показывает окно для ввода cookies вручную"""
    result = False
    
    def on_save():
        nonlocal result
        cookies_text = text_widget.get("1.0", tk.END).strip()
        if cookies_text:
            try:
                with open(COOKIES_FILE, "w", encoding="utf-8") as f:
                    f.write(cookies_text)
                result = True
                messagebox.showinfo("Успех", "🍪 Cookies сохранены!")
                window.destroy()
            except Exception as e:
                messagebox.showerror("Ошибка", f"Не удалось сохранить cookies: {e}")
        else:
            messagebox.showwarning("Внимание", "Введите cookies!")
    
    def on_cancel():
        nonlocal result
        result = False
        window.destroy()
    
    window = tk.Toplevel(parent) if parent else tk.Tk()
    window.title("🍪 Введите cookies для YouTube")
    window.geometry("650x550")
    window.configure(bg="#09070d")
    window.resizable(True, True)
    
    if parent:
        window.transient(parent)
        window.grab_set()
    
    # Заголовок
    tk.Label(
        window,
        text="🍪 Введите cookies для YouTube",
        font=("Arial", 16, "bold"),
        bg="#09070d",
        fg="#ff8bc4"
    ).pack(pady=(20, 5))
    
    tk.Label(
        window,
        text="Скопируйте cookies из браузера и вставьте сюда",
        font=("Arial", 10),
        bg="#09070d",
        fg="#a58ca9"
    ).pack(pady=(0, 10))
    
    # Текст с инструкцией
    info_frame = tk.Frame(window, bg="#150d1e", relief="flat", bd=1)
    info_frame.pack(fill="x", padx=20, pady=(0, 10))
    
    tk.Label(
        info_frame,
        text="📖 Как получить cookies:\n"
             "1. Установите расширение для браузера (например, 'Get cookies.txt')\n"
             "2. Зайдите на YouTube и войдите в аккаунт\n"
             "3. Экспортируйте cookies в файл .txt\n"
             "4. Скопируйте содержимое файла и вставьте сюда\n\n"
             "Или используйте команду:\n"
             "yt-dlp --cookies-from-browser chrome --cookies cookies.txt",
        font=("Arial", 9),
        bg="#150d1e",
        fg="#a58ca9",
        justify="left"
    ).pack(padx=10, pady=10)
    
    # Поле для ввода текста
    text_frame = tk.Frame(window, bg="#0d0812")
    text_frame.pack(fill="both", expand=True, padx=20, pady=(0, 10))
    
    text_widget = tk.Text(
        text_frame,
        bg="#0d0812",
        fg="#fff4fb",
        font=("Courier New", 10),
        wrap=tk.WORD,
        relief="flat",
        bd=1,
        highlightthickness=1,
        highlightcolor="#2b1835",
        highlightbackground="#2b1835"
    )
    text_widget.pack(fill="both", expand=True)
    
    # Кнопки
    btn_frame = tk.Frame(window, bg="#09070d")
    btn_frame.pack(pady=(0, 20))
    
    tk.Button(
        btn_frame,
        text="💾 Сохранить cookies",
        command=on_save,
        font=("Arial", 10, "bold"),
        bg="#ff5caa",
        fg="white",
        activebackground="#ff8bc4",
        relief="flat",
        bd=0,
        cursor="hand2",
        padx=20
    ).pack(side="left", padx=10)
    
    tk.Button(
        btn_frame,
        text="❌ Отмена",
        command=on_cancel,
        font=("Arial", 10, "bold"),
        bg="#2b1835",
        fg="#a58ca9",
        activebackground="#3b1b43",
        relief="flat",
        bd=0,
        cursor="hand2",
        padx=20
    ).pack(side="left", padx=10)
    
    window.bind("<Return>", lambda e: on_save())
    window.protocol("WM_DELETE_WINDOW", on_cancel)
    
    window.mainloop()
    return result

# Проверяем и получаем cookies при запуске
def ensure_cookies(parent=None):
    """Проверяет наличие cookies и запрашивает если их нет"""
    if has_cookies():
        print("✅ Cookies найдены")
        return True
    
    print("⚠️ Cookies не найдены! Пытаюсь получить автоматически...")
    
    # Пытаемся получить из браузера
    if get_cookies_from_browser():
        return True
    
    # Если не получилось - просим ввести вручную
    print("❌ Не удалось получить cookies автоматически")
    print("Пожалуйста, введите cookies вручную...")
    
    return show_cookies_input_window(parent)

# ============================================================
# ПЕРЕМЕННЫЕ ДЛЯ ВИЗУАЛИЗАЦИИ
# ============================================================
_is_music_mode = False
_music_particles = []
_music_stars = []
_music_phase = 0.0
_music_intensity = 0.0
_music_beat_time = 0

# ============================================================
# ПЕРЕМЕННЫЕ ДЛЯ ГРОМКОСТИ МУЗЫКИ
# ============================================================
_music_volume = 100

# ============================================================
# НАСТРОЙКИ ЯЗЫКА ИНТЕРФЕЙСА
# ============================================================
UI_LANGUAGE = "ru"
UI_LANG_FILE = os.path.join(BASE_DIR, "mita_ui_lang.json")

# Голоса для разных языков
TTS_VOICE_RU = "ru-RU-SvetlanaNeural"
TTS_VOICE_UA = "uk-UA-PolinaNeural"
TTS_VOICE_DEFAULT = TTS_VOICE_RU

TTS_RATE = "-5%"
TTS_PITCH = "+2Hz"
TTS_VOLUME = 1.0
TTS_FILE = os.path.join(BASE_DIR, "mita_tts.mp3")
_tts_lock = threading.Lock()
_tts_pygame_ready = False
_tts_stop_requested = False

def get_ui_language():
    global UI_LANGUAGE
    return UI_LANGUAGE

def set_ui_language(lang: str):
    global UI_LANGUAGE
    if lang in ["ru", "ua"]:
        UI_LANGUAGE = lang
        _save_ui_language()
        return True
    return False

def _save_ui_language():
    try:
        with open(UI_LANG_FILE, "w", encoding="utf-8") as f:
            json.dump({"language": UI_LANGUAGE}, f)
    except:
        pass

def _load_ui_language():
    global UI_LANGUAGE
    try:
        if os.path.exists(UI_LANG_FILE):
            with open(UI_LANG_FILE, "r", encoding="utf-8") as f:
                data = json.load(f)
                if data.get("language") in ["ru", "ua"]:
                    UI_LANGUAGE = data["language"]
                    return
    except:
        pass
    UI_LANGUAGE = "ru"

# ============================================================
# ТЕКСТЫ ИНТЕРФЕЙСА НА ДВУХ ЯЗЫКАХ
# ============================================================
def T(key: str) -> str:
    texts = {
        "app_title": {"ru": "MITA AI - Голосовой Ассистент", "ua": "MITA AI - Голосовий Асистент"},
        "online": {"ru": "ONLINE", "ua": "ОНЛАЙН"},
        "control_center": {"ru": "CONTROL CENTER", "ua": "ЦЕНТР КЕРУВАННЯ"},
        "nav_main": {"ru": "Главная", "ua": "Головна"},
        "nav_chat": {"ru": "Чат", "ua": "Чат"},
        "nav_commands": {"ru": "Команды", "ua": "Команди"},
        "nav_settings": {"ru": "Настройки", "ua": "Налаштування"},
        "voice_engine": {"ru": "VOICE ENGINE", "ua": "ГОЛОСОВИЙ РУШІЙ"},
        "hotword": {"ru": "HOTWORD", "ua": "КЛЮЧОВЕ СЛОВО"},
        "mode": {"ru": "РЕЖИМ РАБОТЫ", "ua": "РЕЖИМ РОБОТИ"},
        "change_mode": {"ru": "🔄 Сменить режим", "ua": "🔄 Змінити режим"},
        "manual_input": {"ru": "РУЧНОЙ ВВОД", "ua": "РУЧНЕ ВВЕДЕННЯ"},
        "manual_input_btn": {"ru": "🎤 НАЖМИТЕ И ГОВОРИТЕ", "ua": "🎤 НАТИСНІТЬ І ГОВОРІТЬ"},
        "stop_record": {"ru": "⏹ ОСТАНОВИТЬ ЗАПИСЬ", "ua": "⏹ ЗУПИНИТИ ЗАПИС"},
        "mita_voice": {"ru": "ГОЛОС МИТЫ", "ua": "ГОЛОС МІТИ"},
        "voice_on": {"ru": "🔊 ВКЛЮЧЕН", "ua": "🔊 УВІМКНЕНО"},
        "voice_off": {"ru": "🔇 ВЫКЛЮЧЕН", "ua": "🔇 ВИМКНЕНО"},
        "text_corrector": {"ru": "ИСПРАВИТЕЛЬ ТЕКСТА", "ua": "ВИПРАВЛЯЧ ТЕКСТУ"},
        "corrector_on": {"ru": "🟢 ВКЛЮЧЕН ⌨️", "ua": "🟢 УВІМКНЕНО ⌨️"},
        "corrector_off": {"ru": "⚪ ВЫКЛЮЧЕН", "ua": "⚪ ВИМКНЕНО"},
        "volume": {"ru": "ГРОМКОСТЬ", "ua": "ГУЧНІСТЬ"},
        "change_key": {"ru": "🔄 Сменить ключ", "ua": "🔄 Змінити ключ"},
        "no_key": {"ru": "🔑 Нет ключа", "ua": "🔑 Немає ключа"},
        "music_stop": {"ru": "⏹️ МУЗЫКА НЕ ИГРАЕТ", "ua": "⏹️ МУЗИКА НЕ ГРАЄ"},
        "music_stop_click": {"ru": "⏹️ ОСТАНОВИТЬ МУЗЫКУ", "ua": "⏹️ ЗУПИНИТИ МУЗИКУ"},
        "percent": {"ru": "%", "ua": "%"},
        "chat_with_mita": {"ru": "💬 ЧАТ С МИТОЙ", "ua": "💬 ЧАТ З МІТОЮ"},
        "chat_hint": {"ru": "Введите текст или скажите голосом (правый клик для копирования, Ctrl+V для вставки)",
                      "ua": "Введіть текст або скажіть голосом (правий клік для копіювання, Ctrl+V для вставки)"},
        "commands_title": {"ru": "Команды Миты", "ua": "Команди Міти"},
        "cmd_launch": {"ru": "Запусти", "ua": "Запусти"},
        "cmd_open": {"ru": "Открой", "ua": "Відкрий"},
        "cmd_close": {"ru": "Закрой", "ua": "Закрий"},
        "cmd_write": {"ru": "Напиши", "ua": "Напиши"},
        "cmd_minimize": {"ru": "Сверни", "ua": "Згорни"},
        "cmd_screenshot": {"ru": "Скриншот", "ua": "Скріншот"},
        "cmd_copy": {"ru": "Скопируй", "ua": "Скопіюй"},
        "cmd_paste": {"ru": "Вставь", "ua": "Встав"},
        "cmd_lang": {"ru": "Смени язык", "ua": "Зміни мову"},
        "cmd_click": {"ru": "Клик", "ua": "Клік"},
        "cmd_move": {"ru": "Переведи", "ua": "Переведи"},
        "cmd_song": {"ru": "Песня", "ua": "Пісня"},
        "cmd_stop_music": {"ru": "Стоп музыка", "ua": "Стоп музика"},
        "cmd_louder": {"ru": "Громче", "ua": "Гучніше"},
        "cmd_quieter": {"ru": "Тише", "ua": "Тихіше"},
        "cmd_volume": {"ru": "Громкость", "ua": "Гучність"},
        "ready": {"ru": "✧ ГОТОВА К РАБОТЕ ✧", "ua": "✧ ГОТОВА ДО РОБОТИ ✧"},
        "listening": {"ru": "♪ СЛУШАЮ ВАС... ♪", "ua": "♪ СЛУХАЮ ВАС... ♪"},
        "processing": {"ru": "✦ ОБРАБОТКА... ✦", "ua": "✦ ОБРОБКА... ✦"},
        "manual_recording": {"ru": "🎤 РУЧНАЯ ЗАПИСЬ...", "ua": "🎤 РУЧНЕ ЗАПИС..."},
        "voice_off_status": {"ru": "🔇 Голос отключен", "ua": "🔇 Голос вимкнено"},
        "waiting": {"ru": "Ожидаю голосовую команду", "ua": "Очікую голосову команду"},
        "listening_sub": {"ru": "Говорите — Стелла распознаёт голос", "ua": "Говоріть — Стелла розпізнає голос"},
        "processing_sub": {"ru": "Выполняю вашу команду", "ua": "Виконую вашу команду"},
        "manual_sub": {"ru": "Говорите, я слушаю...", "ua": "Говоріть, я слухаю..."},
        "hello": {"ru": "Привет! Я Мита, твой голосовой ассистент ✨", "ua": "Привіт! Я Міта, твій голосовий асистент ✨"},
        "corrector_info": {"ru": "📝 Исправитель текста: ", "ua": "📝 Виправляч тексту: "},
        "corrector_on_info": {"ru": "ВКЛЮЧЕН ⌨️", "ua": "УВІМКНЕНО ⌨️"},
        "corrector_off_info": {"ru": "ВЫКЛЮЧЕН", "ua": "ВИМКНЕНО"},
        "lang_hint": {"ru": "Говори на русском или украинском - я пойму! 💕", "ua": "Кажи російською або українською - я зрозумію! 💕"},
        "mode_system": {"ru": "🔧 Только система", "ua": "🔧 Тільки система"},
        "mode_ai": {"ru": "🤖 Только ИИ", "ua": "🤖 Тільки ШІ"},
        "mode_all": {"ru": "🌟 Все вместе", "ua": "🌟 Все разом"},
        "mita_greeting": {"ru": "Привет! Я Мита, твой голосовой ассистент ✨", "ua": "Привіт! Я Міта, твій голосовий асистент ✨"},
        "mita_mode": {"ru": "Текущий режим: ", "ua": "Поточний режим: "},
        "mita_corrector": {"ru": "📝 Исправитель текста: ", "ua": "📝 Виправляч тексту: "},
        "mita_lang": {"ru": "Говори на русском или украинском - я пойму! 💕", "ua": "Кажи російською або українською - я зрозумію! 💕"},
        "mita_help": {"ru": "Скажи «Стелла» + команда или нажми кнопки в левой панели! 💕", "ua": "Скажи «Стелла» + команда або натисни кнопки в лівій панелі! 💕"},
        "settings_title": {"ru": "⚙ НАСТРОЙКИ", "ua": "⚙ НАЛАШТУВАННЯ"},
        "settings_language": {"ru": "🌍 ЯЗЫК ИНТЕРФЕЙСА", "ua": "🌍 МОВА ІНТЕРФЕЙСУ"},
        "lang_ru": {"ru": "🇷🇺 Русский", "ua": "🇷🇺 Російська"},
        "lang_ua": {"ru": "🇺🇦 Украинский", "ua": "🇺🇦 Українська"},
        "apply": {"ru": "ПРИМЕНИТЬ", "ua": "ЗАСТОСУВАТИ"},
        "error": {"ru": "❌ Ошибка: ", "ua": "❌ Помилка: "},
        "command_not_found": {"ru": "❌ Команда не распознана.", "ua": "❌ Команду не розпізнано."},
        "hello_response_ru": {"ru": "Привет! Что нужно запустить или открыть?", "ua": "Привіт! Що потрібно запустити або відкрити?"},
        "hello_response_ua": {"ru": "Привет! Что нужно запустить или открыть?", "ua": "Привіт! Що потрібно запустити або відкрити?"},
        "how_are_you": {"ru": "Все отлично, готова к работе!", "ua": "Все добре, готова до роботи!"},
        "im_here": {"ru": "Я здесь! Слушаю вас.", "ua": "Я тут! Слухаю вас."},
        "im_always_here": {"ru": "Я всегда на связи!", "ua": "Я завжди на зв'язку!"},
        "music_stopped": {"ru": "🎵 Музыка остановлена", "ua": "🎵 Музику зупинено"},
        "music_playing": {"ru": "🎵 Ищу: ", "ua": "🎵 Шукаю: "},
        "typing": {"ru": "✏️ Напечатано!", "ua": "✏️ Надруковано!"},
        "typing_corrected": {"ru": "✏️ Исправлено и напечатано", "ua": "✏️ Виправлено та надруковано"},
        "typed": {"ru": "✏️ Напечатал: ", "ua": "✏️ Надрукував: "},
        "minimized_all": {"ru": "✨ Все окна свернуты", "ua": "✨ Усі вікна згорнуто"},
        "screenshot_done": {"ru": "📸 Скриншот сделан", "ua": "📸 Скріншот зроблено"},
        "copied": {"ru": "📋 Скопировано", "ua": "📋 Скопійовано"},
        "pasted": {"ru": "📎 Вставлено", "ua": "📎 Вставлено"},
        "language_changed": {"ru": "💬 Язык изменен", "ua": "💬 Мову змінено"},
        "executed": {"ru": "✅ Выполнено", "ua": "✅ Виконано"},
        "shutting_up": {"ru": "Хорошо, я замолкаю 🤐", "ua": "Добре, я замовкаю 🤐"},
        "command_not_found_text": {"ru": "❌ Команда не распознана.", "ua": "❌ Команду не розпізнано."},
        "window_moved": {"ru": "✅ Окно перемещено на {} монитор", "ua": "✅ Вікно переміщено на {} монітор"},
        "window_move_failed": {"ru": "❌ Не удалось переместить окно", "ua": "❌ Не вдалося перемістити вікно"},
        "no_active_window": {"ru": "❌ Нет активного окна для перемещения", "ua": "❌ Немає активного вікна для переміщення"},
        "move_to_monitor_ask": {"ru": "На какой монитор перевести? Скажи 1 или 2", "ua": "На який монітор перевести? Скажи 1 або 2"},
        "app_launching": {"ru": "✅ Запускаю {}", "ua": "✅ Запускаю {}"},
        "app_not_found": {"ru": "❌ Не удалось найти {}", "ua": "❌ Не вдалося знайти {}"},
        "web_opening": {"ru": "🌐 Открываю {}", "ua": "🌐 Відкриваю {}"},
        "app_closing": {"ru": "✅ Закрываю {}", "ua": "✅ Закриваю {}"},
        "app_close_failed": {"ru": "❌ Не удалось закрыть {}", "ua": "❌ Не вдалося закрити {}"},
        "typing_corrected_msg": {"ru": "📝 Исправлено: '{}' → '{}'", "ua": "📝 Виправлено: '{}' → '{}'"},
        "typing_writing": {"ru": "✏️ Напиши: {}", "ua": "✏️ Напиши: {}"},
        "volume_text": {"ru": "Громкость {}%", "ua": "Гучність {}%"},
        "volume_up": {"ru": "🔊 Громкость: {}%", "ua": "🔊 Гучність: {}%"},
        "volume_down": {"ru": "🔉 Громкость: {}%", "ua": "🔉 Гучність: {}%"},
        "no_audio": {"ru": "❌ Ничего не записано. Попробуйте снова.", "ua": "❌ Нічого не записано. Спробуйте ще раз."},
        "no_data": {"ru": "❌ Нет данных для распознавания", "ua": "❌ Немає даних для розпізнавання"},
        "recognition_failed": {"ru": "❌ Не удалось распознать речь. Попробуйте еще раз.", "ua": "❌ Не вдалося розпізнати мову. Спробуйте ще раз."},
        "corrector_on_text": {"ru": "Исправитель текста включен", "ua": "Виправляч тексту увімкнено"},
        "corrector_off_text": {"ru": "Исправитель текста выключен", "ua": "Виправляч тексту вимкнено"},
        "voice_on_text": {"ru": "Голос включен", "ua": "Голос увімкнено"},
        "voice_off_text": {"ru": "Голос отключен", "ua": "Голос вимкнено"},
        "mode_changed": {"ru": "🔄 Режим изменен на: {}", "ua": "🔄 Режим змінено на: {}"},
        "mode_changed_text": {"ru": "Режим изменен на {}", "ua": "Режим змінено на {}"},
        "lang_changed": {"ru": "🌍 Язык интерфейса изменен на Русский", "ua": "🌍 Мову інтерфейсу змінено на Українську"},
        "chat_cleared": {"ru": "✨ Чат очищен. Чем могу помочь?", "ua": "✨ Чат очищено. Чим можу допомогти?"},
        "copy_success": {"ru": "✅ Скопировано!", "ua": "✅ Скопійовано!"},
        "copy_failed": {"ru": "❌ Нет сообщений для копирования", "ua": "❌ Немає повідомлень для копіювання"},
        "copy_all_success": {"ru": "✅ Весь чат скопирован!", "ua": "✅ Весь чат скопійовано!"},
        "chat_empty": {"ru": "❌ Чат пуст", "ua": "❌ Чат порожній"},
        "clear_confirm": {"ru": "Вы уверены, что хотите очистить историю чата?", "ua": "Ви впевнені, що хочете очистити історію чату?"},
        "key_change_confirm": {"ru": "Текущий ключ будет удален. Продолжить?", "ua": "Поточний ключ буде видалено. Продовжити?"},
        "key_success": {"ru": "Ключ успешно изменен!", "ua": "Ключ успішно змінено!"},
        "key_cancel": {"ru": "Смена ключа отменена", "ua": "Зміну ключа скасовано"},
        "music_stopped_click": {"ru": "🎵 Музыка остановлена по кнопке", "ua": "🎵 Музику зупинено кнопкою"},
        "music_stopped_voice": {"ru": "Музыка остановлена", "ua": "Музику зупинено"},
        "moving_window": {"ru": "Переместила на {} монитор", "ua": "Перемістила на {} монітор"},
        "move_failed": {"ru": "Не удалось переместить", "ua": "Не вдалося перемістити"},
        "window_on_monitor": {"ru": "Окно на {} мониторе", "ua": "Вікно на {} моніторі"},
        "error_text": {"ru": "❌ Ошибка: {}", "ua": "❌ Помилка: {}"},
        "no_cookies": {"ru": "❌ Нет cookies для YouTube. Используйте локальную музыку или настройте cookies.", "ua": "❌ Немає cookies для YouTube. Використовуйте локальну музику або налаштуйте cookies."},
    }
    result = texts.get(key, {})
    if isinstance(result, dict):
        return result.get(UI_LANGUAGE, result.get("ru", key))
    return result

# ============================================================
# РЕЖИМЫ РАБОТЫ МИТЫ
# ============================================================
MODE_SYSTEM = "system"
MODE_AI = "ai"
MODE_ALL = "all"
_mita_mode = MODE_ALL

def set_mita_mode(mode: str):
    global _mita_mode
    if mode in [MODE_SYSTEM, MODE_AI, MODE_ALL]:
        _mita_mode = mode
        return True
    return False

def get_mita_mode():
    return _mita_mode

def get_mode_name(mode: str):
    if mode == MODE_SYSTEM:
        return T("mode_system")
    elif mode == MODE_AI:
        return T("mode_ai")
    else:
        return T("mode_all")

def detect_user_language(text: str) -> str:
    if not text:
        return UI_LANGUAGE if UI_LANGUAGE in ["ru", "ua"] else "ru"
    text_lower = text.lower()
    ua_chars = set('абвгґдеєжзиіїйклмнопрстуфхцчшщьюя')
    ru_chars = set('абвгдеёжзийклмнопрстуфхцчшщъыьэюя')
    ua_count = 0
    ru_count = 0
    for c in text_lower:
        if c in ua_chars:
            ua_count += 1
        elif c in ru_chars and c not in ua_chars:
            ru_count += 1
    ua_markers = ['і', 'є', 'ї', 'ґ', 'привіт', 'спасибі', 'дякую', 'будь-ласка', 'так', 'ні']
    ru_markers = ['привет', 'спасибо', 'пожалуйста', 'да', 'нет', 'хорошо']
    for marker in ua_markers:
        if marker in text_lower:
            ua_count += 2
    for marker in ru_markers:
        if marker in text_lower:
            ru_count += 2
    if ua_count > ru_count:
        return "ua"
    elif ru_count > ua_count:
        return "ru"
    return UI_LANGUAGE if UI_LANGUAGE in ["ru", "ua"] else "ru"

def get_tts_voice(text: str) -> str:
    if UI_LANGUAGE == "ua":
        return TTS_VOICE_UA
    return TTS_VOICE_RU

# ============================================================
# НАСТРОЙКИ - ИСПРАВИТЕЛЬ ТЕКСТА
# ============================================================
_text_corrector_enabled = False

def set_text_corrector(enabled: bool):
    global _text_corrector_enabled
    _text_corrector_enabled = enabled
    return _text_corrector_enabled

def get_text_corrector():
    return _text_corrector_enabled

def correct_text(text: str) -> str:
    if not text or len(text.strip()) < 2:
        return text
    ui_lang = UI_LANGUAGE
    lang_name = "українській" if ui_lang == "ua" else "русском"
    lang_code = "ua" if ui_lang == "ua" else "ru"
    try:
        system_prompt = f"""Ты — ИСПРАВИТЕЛЬ ТЕКСТА.
1. Язык ИНТЕРФЕЙСА: {lang_name} (код: {lang_code})
2. Ты ДОЛЖЕН исправить текст ИМЕННО НА ЭТОМ ЯЗЫКЕ! {lang_name}!
3. Если текст на другом языке - ПЕРЕВЕДИ на {lang_name}!
4. Исправь все грамматические ошибки на этом языке
5. Расставь правильные знаки препинания
6. Исправь орфографию
7. Сделай текст грамотным и читаемым
8. НЕ добавляй лишние слова, НЕ перефразируй
9. Верни ТОЛЬКО исправленный текст, без пояснений
ОЧЕНЬ ВАЖНО: ВСЕГДА ОТВЕЧАЙ ТОЛЬКО НА {lang_name} ЯЗЫКЕ!"""
        completion = client.chat.completions.create(
            model="openai/gpt-oss-120b",
            messages=[
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": f"Исправь этот текст на {lang_name} языке: {text}"}
            ],
            temperature=0.2,
            max_completion_tokens=512,
            top_p=1,
            stream=False
        )
        corrected = completion.choices[0].message.content.strip()
        if not corrected or len(corrected) < 1:
            return text
        corrected = corrected.strip('"').strip("'")
        if corrected.startswith('-') or corrected.startswith('*'):
            corrected = corrected[1:].strip()
        for prefix in ["Вот исправленный текст:", "Исправленный текст:",
                       "Ось виправлений текст:", "Виправлений текст:"]:
            if prefix in corrected:
                corrected = corrected.split(prefix)[-1].strip()
        return corrected if corrected else text
    except Exception as e:
        return text

# ============================================================
# ФУНКЦИИ ДЛЯ ВОСПРОИЗВЕДЕНИЯ ПЕСЕН (ИСПРАВЛЕННЫЕ)
# ============================================================

_music_thread = None
_music_stop = False
_current_music_file = None
_vlc_instance = None
_vlc_player = None
_current_track_title = ""

def set_music_volume(percent: int):
    global _music_volume, _vlc_player
    _music_volume = max(0, min(200, percent))
    if _vlc_player is not None:
        try:
            _vlc_player.audio_set_volume(_music_volume)
            return True
        except:
            pass
    return False

def get_music_volume():
    global _music_volume
    return _music_volume

def volume_up(interface=None):
    result = set_music_volume(get_music_volume() + 10)
    if interface:
        interface.update_volume_display()
        msg = T("volume_up").format(_music_volume)
        interface.add_chat_message("Мита", msg, is_mita=True)
        speak(msg, force=True)
    return result

def volume_down(interface=None):
    result = set_music_volume(get_music_volume() - 10)
    if interface:
        interface.update_volume_display()
        msg = T("volume_down").format(_music_volume)
        interface.add_chat_message("Мита", msg, is_mita=True)
        speak(msg, force=True)
    return result

def find_local_music(query: str) -> str:
    music_dir = os.path.join(BASE_DIR, "music")
    if not os.path.exists(music_dir):
        try:
            os.makedirs(music_dir)
        except:
            pass
        return None
    query_lower = query.lower()
    for file in os.listdir(music_dir):
        if file.lower().endswith(('.mp3', '.wav', '.flac', '.ogg')):
            file_name = file.lower().replace('.mp3', '').replace('.wav', '').replace('.flac', '').replace('.ogg', '')
            if query_lower in file_name or file_name in query_lower:
                return os.path.join(music_dir, file)
    words = query_lower.split()
    for file in os.listdir(music_dir):
        if file.lower().endswith(('.mp3', '.wav', '.flac', '.ogg')):
            file_name = file.lower().replace('.mp3', '').replace('.wav', '').replace('.flac', '').replace('.ogg', '')
            for word in words:
                if len(word) > 2 and word in file_name:
                    return os.path.join(music_dir, file)
    return None

def play_youtube_audio(query: str, interface=None):
    global _music_thread, _music_stop, _current_music_file, _vlc_instance, _vlc_player, _current_track_title, _is_music_mode

    if not HAS_YDL or not HAS_VLC:
        return False

    # Проверяем наличие cookies
    if not has_cookies():
        msg = T("no_cookies")
        if interface:
            interface.add_chat_message("Мита", msg, is_mita=True)
            speak(msg, force=True)
        return False

    try:
        _music_stop = False
        _current_track_title = ""

        # Очищаем запрос от триггерных слов
        clean_query = query
        for tw in ["мита", "стелла", "кепочка", "стелам", "стеллу", "міта", "стела", "включи", "поставь", "песню"]:
            clean_query = clean_query.replace(tw, '').strip()
        if not clean_query or len(clean_query) < 2:
            return False

        # Проверяем локальную музыку
        music_file = find_local_music(clean_query)
        if music_file:
            _current_track_title = os.path.basename(music_file)
            if interface:
                interface.update_music_button(True)
            return _play_audio_vlc(music_file, _current_track_title, interface)

        # Ищем на YouTube с улучшенными настройками
        print(f"[Музыка] Ищу: {clean_query}")

        ydl_opts = {
            'format': 'bestaudio/best',
            'quiet': True,
            'no_warnings': True,
            'default_search': 'ytsearch10',
            'extract_flat': False,
            'noplaylist': True,
            'skip_download': True,
            'socket_timeout': 30,
            'retries': 10,
            'fragment_retries': 10,
            'ignoreerrors': True,
            'cachedir': False,
            'geo_bypass': True,
            'geo_bypass_country': 'US',
            'cookiefile': COOKIES_FILE,  # Используем cookies файл
            'extractor_args': {
                'youtube': {
                    'player_client': ['android', 'web'],
                    'player_skip': ['webpage'],
                }
            }
        }

        audio_url = None
        title = None

        with yt_dlp.YoutubeDL(ydl_opts) as ydl:
            try:
                info = ydl.extract_info(f"ytsearch10:{clean_query}", download=False)
                entries = info.get('entries', []) if 'entries' in info else []

                for entry in entries:
                    if not entry:
                        continue
                    try:
                        test_url = entry.get('url') or entry.get('webpage_url')
                        if test_url:
                            audio_url = test_url
                            title = entry.get('title', 'Песня')
                            print(f"[Музыка] Нашел: {title}")
                            break
                    except:
                        continue

                if not audio_url:
                    for entry in entries:
                        if not entry:
                            continue
                        try:
                            video_id = entry.get('id')
                            if video_id:
                                alt_url = f"https://www.youtube.com/watch?v={video_id}"
                                audio_url = alt_url
                                title = entry.get('title', 'Песня')
                                print(f"[Музыка] Нашел: {title}")
                                break
                        except:
                            continue

                if not audio_url:
                    words = clean_query.split()
                    if len(words) > 3:
                        short_query = ' '.join(words[:3])
                        info = ydl.extract_info(f"ytsearch5:{short_query}", download=False)
                        entries = info.get('entries', []) if 'entries' in info else []
                        for entry in entries:
                            if entry and entry.get('url'):
                                audio_url = entry.get('url')
                                title = entry.get('title', 'Песня')
                                break

            except Exception as e:
                print(f"[Музыка] Ошибка поиска: {e}")
                # Пробуем снова с обновленными cookies
                if "Sign in to confirm" in str(e) or "cookies" in str(e).lower():
                    if interface:
                        msg = "⚠️ Нужно обновить cookies для YouTube. Попробуйте получить их заново."
                        interface.add_chat_message("Мита", msg, is_mita=True)
                        speak(msg, force=True)
                        # Предлагаем обновить cookies
                        if show_cookies_input_window(interface.root):
                            # Повторяем попытку с новыми cookies
                            return play_youtube_audio(query, interface)
                return False

        if not audio_url:
            print("[Музыка] Не удалось найти песню")
            return False

        _current_track_title = title or "Песня"
        if interface:
            interface.update_music_button(True)
            interface.add_chat_message("Мита", f"🎵 Играет: {_current_track_title}", is_mita=True)

        return _play_audio_vlc(audio_url, _current_track_title, interface)

    except Exception as e:
        print(f"[Ошибка музыки]: {e}")
        return False

def _play_audio_vlc(audio_url: str, title: str, interface=None):
    global _music_thread, _music_stop, _vlc_instance, _vlc_player, _is_music_mode, _music_volume

    try:
        _music_stop = False
        _is_music_mode = True

        if _vlc_instance is None:
            _vlc_instance = vlc.Instance('--quiet', '--no-video', '--network-caching=2000')

        _vlc_player = _vlc_instance.media_player_new()
        media = _vlc_instance.media_new(audio_url)
        media.add_option(':no-video')
        media.add_option(':network-caching=2000')
        media.add_option(':http-caching=1000')
        media.add_option(':file-caching=500')
        _vlc_player.set_media(media)
        _vlc_player.audio_set_volume(_music_volume)

        if _vlc_player.play() == -1:
            raise Exception("VLC не смог открыть аудиопоток")

        def monitor():
            global _music_stop, _vlc_player, _is_music_mode, _music_intensity
            started = False
            deadline = time.time() + 60

            while not _music_stop:
                player = _vlc_player
                if player is None:
                    break
                try:
                    state = player.get_state()
                except Exception:
                    break

                if state in (vlc.State.Playing, vlc.State.Paused):
                    started = True
                    _music_intensity = 0.7 + random.random() * 0.3
                elif started and state in (vlc.State.Ended, vlc.State.Stopped, vlc.State.Error):
                    break
                elif not started and time.time() > deadline:
                    try:
                        player.stop()
                        player.play()
                    except:
                        pass
                    break
                time.sleep(0.1)

            try:
                if _vlc_player is not None:
                    _vlc_player.stop()
                    _vlc_player.release()
            except Exception:
                pass
            _vlc_player = None
            _is_music_mode = False
            _music_intensity = 0.0

            if interface:
                interface.update_music_button(False)
                interface.update_volume_display()
            _music_stop = False

        _music_thread = threading.Thread(target=monitor, daemon=True)
        _music_thread.start()
        return True

    except Exception as e:
        print(f"[Ошибка VLC]: {e}")
        return False

def stop_music():
    global _music_stop, _vlc_player, _is_music_mode, _music_intensity
    _music_stop = True
    _is_music_mode = False
    _music_intensity = 0.0
    try:
        if _vlc_player is not None:
            _vlc_player.stop()
            _vlc_player.release()
    except Exception:
        pass
    _vlc_player = None
    return True

def is_music_playing():
    try:
        return _vlc_player is not None and _vlc_player.get_state() == vlc.State.Playing
    except Exception:
        return False

def process_music_command(text: str, interface) -> bool:
    global _music_volume
    cleaned = text.lower().strip()

    # Стоп музыка
    stop_phrases_ru = ['стоп музыка', 'выключи музыку', 'останови музыку', 'хватит музыки', 'прекрати музыку']
    stop_phrases_ua = ['стоп музика', 'вимкни музику', 'зупини музику', 'досить музики', 'припини музику']
    stop_phrases = stop_phrases_ru + stop_phrases_ua

    if any(p in cleaned for p in stop_phrases):
        stop_music()
        msg = T("music_stopped")
        interface.add_chat_message("Мита", msg, is_mita=True)
        interface.update_music_button(False)
        speak(msg, force=True)
        return True

    # Регулировка громкости
    if 'громкост' in cleaned or 'гучн' in cleaned:
        vol_match = re.search(r'(\d{1,3})%?', cleaned)
        if vol_match:
            vol = int(vol_match.group(1))
            if 0 <= vol <= 200:
                set_music_volume(vol)
                interface.update_volume_display()
                msg = T("volume_up").format(_music_volume)
                interface.add_chat_message("Мита", msg, is_mita=True)
                speak(msg, force=True)
                return True

        if 'громче' in cleaned or 'гучніше' in cleaned:
            volume_up(interface)
            return True
        elif 'тише' in cleaned or 'тихіше' in cleaned:
            volume_down(interface)
            return True

    # Включение песни
    if any(kw in cleaned for kw in ['песн', 'музык', 'трек', 'пісн']):
        song_name = None
        keywords_ru = ['песню', 'песня', 'песни', 'музыку', 'музыка', 'трек', 'включи', 'поставь']
        keywords_ua = ['пісню', 'пісня', 'пісні', 'музику', 'музика', 'включи', 'постав']

        for keyword in keywords_ru + keywords_ua:
            if keyword in cleaned:
                parts = cleaned.split(keyword, 1)
                if len(parts) > 1:
                    song_name = parts[1].strip()
                    for word in ['мита', 'стелла', 'пожалуйста', 'сейчас', 'міта', 'будь-ласка', 'зараз']:
                        song_name = song_name.replace(word, '').strip()
                    break

        if song_name and len(song_name) > 2:
            interface.add_chat_message("Мита", f"🎵 Ищу: {song_name}", is_mita=True)
            threading.Thread(target=play_youtube_audio, args=(song_name, interface), daemon=True).start()
            return True

    return False

# ============================================================
# TTS ФУНКЦИИ
# ============================================================

def speak(text: str, force: bool = False):
    global _tts_pygame_ready, _tts_stop_requested, TTS_VOLUME

    try:
        if 'interface' in globals() and interface:
            if hasattr(interface, 'tts_muted') and interface.tts_muted:
                return False
    except:
        pass

    text = str(text or "").strip()
    if not text:
        return False

    if is_music_playing() and not force:
        return False

    cleaned_text = re.sub(r'[^\w\s\.\,\!\?\-\()]', '', text, flags=re.UNICODE)
    cleaned_text = re.sub(r'\s+', ' ', cleaned_text).strip()
    cleaned_text = cleaned_text.replace('*', '').replace('_', '').replace('`', '')

    if not cleaned_text:
        return False

    if not HAS_EDGE_TTS:
        return False

    voice = TTS_VOICE_UA if UI_LANGUAGE == "ua" else TTS_VOICE_RU

    def worker():
        global _tts_pygame_ready, _tts_stop_requested, TTS_VOLUME
        with _tts_lock:
            try:
                try:
                    if 'interface' in globals() and interface:
                        if hasattr(interface, 'tts_muted') and interface.tts_muted:
                            return
                except:
                    pass

                _tts_stop_requested = False

                if _tts_pygame_ready:
                    try:
                        pygame.mixer.music.stop()
                        pygame.mixer.music.unload()
                    except:
                        pass

                if os.path.exists(TTS_FILE):
                    try:
                        os.remove(TTS_FILE)
                    except:
                        pass

                async def generate():
                    communicate = edge_tts.Communicate(
                        cleaned_text,
                        voice=voice,
                        rate=TTS_RATE,
                        pitch=TTS_PITCH,
                        volume="+0%"
                    )
                    await communicate.save(TTS_FILE)

                asyncio.run(generate())

                if not os.path.exists(TTS_FILE):
                    return

                if not _tts_pygame_ready:
                    pygame.mixer.init()
                    _tts_pygame_ready = True

                try:
                    if 'interface' in globals() and interface:
                        if hasattr(interface, 'tts_muted') and interface.tts_muted:
                            return
                except:
                    pass

                time.sleep(0.1)
                pygame.mixer.music.load(TTS_FILE)
                pygame.mixer.music.set_volume(TTS_VOLUME)
                pygame.mixer.music.play()

                while pygame.mixer.music.get_busy():
                    if _tts_stop_requested:
                        pygame.mixer.music.stop()
                        break
                    try:
                        if 'interface' in globals() and interface:
                            if hasattr(interface, 'tts_muted') and interface.tts_muted:
                                pygame.mixer.music.stop()
                                break
                    except:
                        pass
                    time.sleep(0.03)

            except Exception as e:
                print(f"[TTS Error]: {e}")

    threading.Thread(target=worker, daemon=True).start()
    return True

def stop_tts():
    global _tts_stop_requested
    _tts_stop_requested = True
    try:
        if _tts_pygame_ready:
            pygame.mixer.music.stop()
            return True
    except:
        pass
    return False

# ============================================================
# GROQ API
# ============================================================

GROQ_API_KEY = "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"
client = Groq(api_key=GROQ_API_KEY)

def ask_groq(question):
    try:
        ui_lang = UI_LANGUAGE
        lang_text = "українською" if ui_lang == "ua" else "русском"
        system_prompt = f"""Ты — голосовой ассистент Мита. Отвечай кратко и по делу.
Ты ОБЯЗАН отвечать ТОЛЬКО на {lang_text} языке!
Даже если вопрос на другом языке - ВСЕГДА отвечай на {lang_text}!
Отвечай обычным текстом, без форматирования."""
        completion = client.chat.completions.create(
            model="openai/gpt-oss-120b",
            messages=[
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": question}
            ],
            temperature=1,
            max_completion_tokens=2048,
            top_p=1,
            reasoning_effort="medium",
            stream=False
        )
        return completion.choices[0].message.content
    except Exception as e:
        return T("error_text").format(e)

def ask_groq_chat(question, chat_history=None):
    try:
        ui_lang = UI_LANGUAGE
        lang_text = "українською" if ui_lang == "ua" else "русском"
        system_prompt = f"""Ты — голосовой ассистент Мита. Ты милая и дружелюбная девушка-помощник.
Ты ОБЯЗАН отвечать ТОЛЬКО на {lang_text} языке!
Даже если вопрос на другом языке - ВСЕГДА отвечай на {lang_text}!
Отвечай кратко и по делу. Без форматирования."""
        messages = [{"role": "system", "content": system_prompt}]
        if chat_history:
            for msg in chat_history[-10:]:
                messages.append(msg)
        messages.append({"role": "user", "content": question})
        completion = client.chat.completions.create(
            model="openai/gpt-oss-120b",
            messages=messages,
            temperature=0.9,
            max_completion_tokens=1024,
            top_p=1,
            reasoning_effort="medium",
            stream=False
        )
        return completion.choices[0].message.content
    except Exception as e:
        return T("error_text").format(e)

# ============================================================
# КЛЮЧИ И АВТОРИЗАЦИЯ
# ============================================================

SOUNDS_DIR = os.path.join(BASE_DIR, "sounds")
JSON_CACHE_FILE = os.path.join(BASE_DIR, "saveAppPty.json")

KEY_DB_FILE = os.path.join(BASE_DIR, "stella_keys.json")
ACTIVATED_KEY_FILE = os.path.join(BASE_DIR, "stella_activated_key.json")

KEY_TABLE = {
    "STELLA-1MIN": {"seconds": 60, "created_at": "2026-08-30 12:00:00"},
    "STELLA-1H": {"seconds": 3600, "created_at": "2026-08-30 12:00:00"},
    "STELLA-1DAY": {"seconds": 86400, "created_at": "2026-08-30 12:00:00"},
    "STELLA-1WEEK": {"seconds": 604800, "created_at": "2026-08-30 12:00:00"},
}

def _load_json_file(path):
    try:
        if os.path.exists(path):
            with open(path, "r", encoding="utf-8") as f:
                return json.load(f)
    except Exception:
        pass
    return {}

def _save_json_file(path, data):
    try:
        with open(path, "w", encoding="utf-8") as f:
            json.dump(data, f, ensure_ascii=False, indent=4)
        return True
    except Exception:
        return False

def _remaining_text(seconds):
    seconds = max(0, int(seconds))
    d, rem = divmod(seconds, 86400)
    h, rem = divmod(rem, 3600)
    m, s = divmod(rem, 60)
    if d: return f"{d}д {h}ч"
    if h: return f"{h}ч {m}м"
    if m: return f"{m}м {s}с"
    return f"{s}с"

def _build_key_db():
    db = _load_json_file(KEY_DB_FILE)
    changed = False
    for key, cfg in KEY_TABLE.items():
        if key not in db:
            created = datetime.strptime(cfg["created_at"], "%Y-%m-%d %H:%M:%S").timestamp()
            db[key] = {
                "created_at": created,
                "expires_at": created + cfg["seconds"],
                "duration_seconds": cfg["seconds"],
                "activated": False,
                "activated_at": None,
            }
            changed = True
    if changed:
        _save_json_file(KEY_DB_FILE, db)
    return db

def _get_saved_key():
    saved = _load_json_file(ACTIVATED_KEY_FILE)
    key = str(saved.get("key", "")).strip().upper()
    if not key: return None
    db = _build_key_db()
    info = db.get(key)
    if not info: return None
    if time.time() >= float(info.get("expires_at", 0)):
        return None
    return key

def _remember_key(key):
    _save_json_file(ACTIVATED_KEY_FILE, {"key": key, "saved_at": time.time()})

def _clear_saved_key():
    if os.path.exists(ACTIVATED_KEY_FILE):
        try:
            os.remove(ACTIVATED_KEY_FILE)
            return True
        except:
            pass
    return False

def validate_stella_key(key):
    key = key.strip().upper()
    db = _build_key_db()
    if key not in db:
        return False, "Неверный ключ"
    info = db[key]
    remaining = float(info["expires_at"]) - time.time()
    if remaining <= 0:
        return False, "Срок действия ключа закончился"
    info["activated"] = True
    info["activated_at"] = info.get("activated_at") or time.time()
    db[key] = info
    _save_json_file(KEY_DB_FILE, db)
    _remember_key(key)
    return True, f"Ключ принят • осталось {_remaining_text(remaining)}"

# ============================================================
# ОКНО ВЫБОРА РЕЖИМА ПРИ ЗАПУСКЕ
# ============================================================

class ModeSelectionWindow:
    def __init__(self, parent=None):
        self.result = MODE_ALL
        self.parent = parent
        self.root = tk.Tk() if parent is None else tk.Toplevel(parent)
        self.root.title("MITA AI — Выберите режим работы")
        self.root.geometry("520x520")
        self.root.resizable(False, False)
        self.root.configure(bg="#09070d")

        if parent is not None:
            self.root.transient(parent)
            self.root.grab_set()

        self.canvas = tk.Canvas(self.root, bg="#09070d", highlightthickness=0)
        self.canvas.pack(fill="both", expand=True)

        for y in range(520):
            t = y / 520
            r = int(9 + 8 * t)
            g = int(7 + 3 * t)
            b = int(13 + 14 * t)
            self.canvas.create_line(0, y, 520, y, fill=f"#{r:02x}{g:02x}{b:02x}")

        self.canvas.create_text(260, 40, text="✦ MITA AI ✦", font=("Arial", 22, "bold"), fill="#ff8bc4")
        self.canvas.create_text(260, 75, text="ВЫБЕРИТЕ РЕЖИМ РАБОТЫ", font=("Arial", 10, "bold"), fill="#a58ca9")

        modes = [
            (MODE_SYSTEM, "🔧 Только система", "Выполняет системные команды\n(запуск, открытие, управление окнами)"),
            (MODE_AI, "🤖 Только ИИ", "Отвечает только через ИИ Groq\n(как чат-бот)"),
            (MODE_ALL, "🌟 Все вместе", "И системные команды, и ИИ ответы\n(рекомендуемый режим)"),
        ]

        self.selected_mode = tk.StringVar(value=MODE_ALL)

        y_pos = 120
        for mode, label, desc in modes:
            bg_id = self.canvas.create_rectangle(50, y_pos - 8, 470, y_pos + 75,
                                                 fill="#150d1e", outline="#2b1835", width=1)

            rb = tk.Radiobutton(
                self.root,
                text=label,
                variable=self.selected_mode,
                value=mode,
                font=("Arial", 13, "bold"),
                bg="#150d1e",
                fg="#fff4fb",
                selectcolor="#2b1835",
                activebackground="#150d1e",
                activeforeground="#ff8bc4",
                relief="flat",
                bd=0,
                cursor="hand2"
            )
            rb.place(x=70, y=y_pos, width=400, height=30)

            desc_text = self.canvas.create_text(260, y_pos + 45, text=desc,
                                                font=("Arial", 9), fill="#a58ca9")

            def make_handler(mode_val):
                return lambda e: self.selected_mode.set(mode_val)

            self.canvas.tag_bind(bg_id, "<Button-1>", make_handler(mode))

            y_pos += 95

        self.button = tk.Button(
            self.root,
            text="ПРИМЕНИТЬ И ЗАПУСТИТЬ",
            command=self.confirm,
            font=("Arial", 11, "bold"),
            bg="#ff5caa",
            fg="white",
            activebackground="#ff8bc4",
            relief="flat",
            bd=0,
            cursor="hand2"
        )
        self.button.place(x=105, y=430, width=310, height=45)

        self.dont_show_var = tk.IntVar(value=0)
        self.dont_show_cb = tk.Checkbutton(
            self.root,
            text="Не показывать при запуске",
            variable=self.dont_show_var,
            font=("Arial", 9),
            bg="#09070d",
            fg="#a58ca9",
            selectcolor="#09070d",
            activebackground="#09070d",
            activeforeground="#ff8bc4",
            relief="flat",
            bd=0,
            cursor="hand2"
        )
        self.dont_show_cb.place(x=140, y=485, width=250, height=25)

        self.root.bind("<Return>", lambda e: self.confirm())
        self.root.protocol("WM_DELETE_WINDOW", self.cancel)

    def confirm(self):
        self.result = self.selected_mode.get()
        if self.dont_show_var.get() == 1:
            self._save_preference()
        self.root.destroy()

    def cancel(self):
        self.result = MODE_ALL
        self.root.destroy()

    def _save_preference(self):
        try:
            pref_file = os.path.join(BASE_DIR, "mita_mode_pref.json")
            with open(pref_file, "w", encoding="utf-8") as f:
                json.dump({"mode": self.result, "dont_show": True}, f)
        except:
            pass

    def show(self):
        self.root.mainloop()
        return self.result

def get_saved_mode():
    try:
        pref_file = os.path.join(BASE_DIR, "mita_mode_pref.json")
        if os.path.exists(pref_file):
            with open(pref_file, "r", encoding="utf-8") as f:
                data = json.load(f)
                if data.get("dont_show", False):
                    return data.get("mode", MODE_ALL)
    except:
        pass
    return None

# ============================================================
# КЛАСС КЛЮЧЕВОГО ОКНА
# ============================================================

class KeyLoginWindow:
    def __init__(self, parent=None):
        self.result = False
        self.parent = parent
        self.root = tk.Tk() if parent is None else tk.Toplevel(parent)
        self.root.title("MITA AI — Введите ключ")
        self.root.geometry("520x400")
        self.root.resizable(False, False)
        self.root.configure(bg="#09070d")

        if parent is not None:
            self.root.transient(parent)
            self.root.grab_set()

        self.canvas = tk.Canvas(self.root, bg="#09070d", highlightthickness=0)
        self.canvas.pack(fill="both", expand=True)

        for y in range(400):
            t = y / 400
            r = int(9 + 8 * t)
            g = int(7 + 3 * t)
            b = int(13 + 14 * t)
            self.canvas.create_line(0, y, 520, y, fill=f"#{r:02x}{g:02x}{b:02x}")

        self.canvas.create_text(260, 50, text="✦ MITA AI", font=("Arial", 24, "bold"), fill="#ff8bc4")
        self.canvas.create_text(260, 145, text="SECURE ACCESS", font=("Arial", 9, "bold"), fill="#a58ca9")
        self.canvas.create_text(260, 175, text="Введите ключ доступа", font=("Arial", 11, "bold"), fill="#fff4fb")

        self.entry = tk.Entry(self.root, font=("Arial", 13, "bold"), justify="center",
                              bg="#150d1e", fg="#fff4fb", insertbackground="#ff8bc4", relief="flat", bd=0)
        self.entry.place(x=70, y=200, width=380, height=42)

        self.status = self.canvas.create_text(260, 265, text="", font=("Arial", 9, "bold"), fill="#a58ca9")

        self.button = tk.Button(self.root, text="ВОЙТИ В MITA", command=self.try_login,
                                font=("Arial", 10, "bold"), bg="#ff5caa", fg="white",
                                activebackground="#ff8bc4", relief="flat", bd=0, cursor="hand2")
        self.button.place(x=145, y=290, width=230, height=42)

        self.entry.focus_set()
        self.root.bind("<Return>", lambda e: self.try_login())
        self.root.protocol("WM_DELETE_WINDOW", self.cancel)

    def try_login(self):
        ok, msg = validate_stella_key(self.entry.get())
        if ok:
            self.canvas.itemconfig(self.status, text=msg, fill="#ff8bc4")
            self.result = True
            self.root.after(450, self.root.destroy)
        else:
            self.canvas.itemconfig(self.status, text=msg, fill="#ff557f")
            self.entry.delete(0, tk.END)
            self.entry.focus_set()

    def cancel(self):
        self.result = False
        self.root.destroy()

    def show(self):
        self.root.mainloop()
        return self.result

def require_stella_key(parent=None):
    _build_key_db()
    if _get_saved_key():
        return True
    return KeyLoginWindow(parent).show()

# ============================================================
# СИСТЕМНЫЙ АУДИО МОНИТОР
# ============================================================

class SystemAudioMonitor:
    def __init__(self, interface):
        self.interface = interface
        self.running = False
        self.thread = None
        self.audio_data = []
        self.sample_rate = 44100
        self.chunk_size = 1024
        self.threshold = 0.02
        self.last_beat_time = 0
        self.beat_smoothing = 0.0

    def start(self):
        if self.running:
            return
        self.running = True
        self.thread = threading.Thread(target=self._monitor_loop, daemon=True)
        self.thread.start()

    def stop(self):
        self.running = False
        if self.thread:
            self.thread.join(timeout=1)

    def _monitor_loop(self):
        try:
            with sd.InputStream(
                    samplerate=self.sample_rate,
                    channels=1,
                    dtype='float32',
                    blocksize=self.chunk_size,
                    device=None,
                    callback=self._audio_callback
            ):
                while self.running:
                    if len(self.audio_data) > 0:
                        data = self.audio_data[-1]
                        amplitude = np.max(np.abs(data))
                        heart_beat = min(1.0, amplitude * 25)
                        self.beat_smoothing = self.beat_smoothing * 0.85 + heart_beat * 0.15
                        self.interface.update_heart_beat(self.beat_smoothing)
                        if len(self.audio_data) > 10:
                            self.audio_data.pop(0)
                    time.sleep(0.03)
        except Exception as e:
            print(f"[Audio Monitor Error]: {e}")
            self._fallback_monitor()

    def _audio_callback(self, indata, frames, time, status):
        if status:
            pass
        self.audio_data.append(indata.copy())

    def _fallback_monitor(self):
        try:
            with sd.InputStream(
                    samplerate=self.sample_rate,
                    channels=1,
                    dtype='float32',
                    blocksize=self.chunk_size,
                    callback=self._audio_callback
            ):
                while self.running:
                    if len(self.audio_data) > 0:
                        data = self.audio_data[-1]
                        amplitude = np.max(np.abs(data))
                        heart_beat = min(1.0, amplitude * 30)
                        self.beat_smoothing = self.beat_smoothing * 0.85 + heart_beat * 0.15
                        self.interface.update_heart_beat(self.beat_smoothing)
                        if len(self.audio_data) > 10:
                            self.audio_data.pop(0)
                    time.sleep(0.03)
        except Exception as e:
            print(f"[Audio Fallback Error]: {e}")

# ============================================================
# ОСНОВНОЙ ИНТЕРФЕЙС
# ============================================================

class MisideInterface:
    W, H = 1200, 750

    def __init__(self):
        self.root = tk.Tk()
        self.root.title(T("app_title"))
        self.root.geometry(f"{self.W}x{self.H}")
        self.root.minsize(1000, 650)
        self.root.configure(bg="#09070d")
        self.root.overrideredirect(False)
        self.root.resizable(True, True)
        self.root.attributes("-topmost", False)
        self.root.attributes("-alpha", 0.0)

        self.running = True
        self.is_listening = False
        self.is_processing = False
        self.angle = 0.0
        self.phase = 0.0
        self.drag_x = 0
        self.drag_y = 0
        self.particles = []
        self.stars = []
        self.bars = []
        self.chat_history = []
        self.message_tags = []
        self.is_dragging = False

        self.heart_beat_strength = 0.0
        self.heart_beat_target = 0.0
        self.heart_bpm = 72
        self.last_beat_time = 0
        self.is_beating = False
        self.beat_phase = 0.0

        self.music_mode = False
        self.music_particles = []
        self.floating_stars = []
        self.music_phase = 0.0
        self.music_intensity = 0.0

        self.volume_scale = None
        self.volume_label_id = None
        self.music_stop_button = None
        self.manual_record_button = None
        self.is_manual_recording = False
        self.manual_audio_buffer = []
        self.manual_recording_thread = None
        self.mode_label = None

        self.tts_mute_button = None
        self.tts_muted = False

        self.corrector_button = None
        self.corrector_enabled = False

        self.lang_button = None
        self.lang_label = None

        self.colors = {
            "bg": "#09070d",
            "panel": "#100b17",
            "panel2": "#150d1e",
            "line": "#2b1835",
            "primary": "#ff5caa",
            "primary2": "#ff8bc4",
            "purple": "#a77bff",
            "text": "#fff4fb",
            "muted": "#a58ca9",
            "dim": "#5d4965",
            "danger": "#ff557f",
            "chat_bg": "#0d0812",
            "chat_user": "#2b1835",
            "chat_mita": "#1a0d24",
        }

        self.root.bind("<Button-1>", self.start_move)
        self.root.bind("<B1-Motion>", self.on_move)
        self.root.bind("<ButtonRelease-1>", self.stop_move)

        self.setup_ui()
        self.root.after(20, self.fade_in)

        self.audio_monitor = SystemAudioMonitor(self)
        self.audio_monitor.start()

        self.animate()

    def start_move(self, event):
        widget = event.widget
        if widget in [self.chat_display, self.chat_input, self.volume_scale,
                      self.music_stop_button, self.manual_record_button,
                      self.tts_mute_button, self.corrector_button, self.lang_button]:
            self.is_dragging = False
            return

        try:
            x, y = event.x_root, event.y_root
            chat_x = self.chat_frame.winfo_rootx()
            chat_y = self.chat_frame.winfo_rooty()
            chat_w = self.chat_frame.winfo_width()
            chat_h = self.chat_frame.winfo_height()

            if (chat_x <= x <= chat_x + chat_w and
                    chat_y <= y <= chat_y + chat_h):
                self.is_dragging = False
                return
        except:
            pass

        self.is_dragging = True
        self.drag_x = event.x
        self.drag_y = event.y

    def on_move(self, event):
        if not self.is_dragging:
            return
        dx = event.x - self.drag_x
        dy = event.y - self.drag_y
        x = self.root.winfo_x() + dx
        y = self.root.winfo_y() + dy
        self.root.geometry(f"+{x}+{y}")

    def stop_move(self, event):
        self.is_dragging = False

    def update_heart_beat(self, strength):
        self.heart_beat_target = strength
        if strength > 0.1:
            if self.music_mode:
                self.heart_bpm = 120 + int(strength * 80)
            else:
                self.heart_bpm = 72 + int(strength * 48)

    def rounded_rect(self, x1, y1, x2, y2, r=18, fill=None, outline=None, width=1, tags=None):
        pts = [
            x1 + r, y1, x2 - r, y1, x2, y1, x2, y1 + r,
            x2, y2 - r, x2, y2, x2 - r, y2, x1 + r, y2,
            x1, y2, x2, y2 - r, x1, y2 - r, x1, y1 + r, x1, y1
        ]
        return self.canvas.create_polygon(
            pts, smooth=True, splinesteps=12,
            fill=fill or "", outline=outline or "", width=width,
            tags=tags
        )

    def setup_ui(self):
        self.canvas = tk.Canvas(
            self.root, bg=self.colors["bg"], highlightthickness=0,
            bd=0, relief="flat"
        )
        self.canvas.pack(fill=tk.BOTH, expand=True)
        self.create_background()

        self.rounded_rect(18, 15, self.W - 18, 76, 20,
                          self.colors["panel"], self.colors["line"])
        self.canvas.create_text(42, 35, text="✦", font=("Arial", 20, "bold"),
                                fill=self.colors["primary2"], anchor="w")
        self.canvas.create_text(72, 33, text="MITA", font=("Arial", 19, "bold"),
                                fill=self.colors["text"], anchor="w")
        self.canvas.create_text(72, 56, text=f"AI VOICE ASSISTANT  •  {T('online')}",
                                font=("Arial", 8, "bold"), fill=self.colors["muted"], anchor="w")

        self.rounded_rect(18, 92, 235, 690, 20,
                          self.colors["panel"], self.colors["line"])
        self.canvas.create_text(42, 120, text=T("control_center"),
                                font=("Arial", 9, "bold"), fill=self.colors["dim"], anchor="w")

        self.nav_items = []
        nav_labels = [
            (T("nav_main"), "⌂"),
            (T("nav_chat"), "💬"),
            (T("nav_commands"), "⌘"),
            (T("nav_settings"), "⚙")
        ]
        for i, (label, icon) in enumerate(nav_labels):
            y = 160 + i * 48
            bg = self.rounded_rect(
                32, y - 20, 221, y + 22, 12,
                self.colors["panel2"] if i == 0 else "",
                self.colors["line"] if i == 0 else ""
            )
            txt = self.canvas.create_text(
                54, y, text=f"{icon}   {label}",
                font=("Arial", 10, "bold"),
                fill=self.colors["text"] if i == 0 else self.colors["muted"],
                anchor="w"
            )
            self.nav_items.append((bg, txt))
            self.canvas.tag_bind(txt, "<Button-1>", lambda e, idx=i: self.switch_tab(idx))
            self.canvas.tag_bind(bg, "<Button-1>", lambda e, idx=i: self.switch_tab(idx))

        self.canvas.create_text(42, 380, text=T("voice_engine"),
                                font=("Arial", 9, "bold"), fill=self.colors["dim"], anchor="w")
        self.engine_dot = self.canvas.create_oval(43, 406, 51, 414, fill=self.colors["primary"], outline="")
        self.engine_text = self.canvas.create_text(61, 410, text="Mita AI ",
                                                   font=("Arial", 9, "bold"), fill=self.colors["muted"], anchor="w")

        self.canvas.create_text(42, 448, text=T("hotword"), font=("Arial", 9, "bold"), fill=self.colors["dim"],
                                anchor="w")
        self.canvas.create_text(42, 470, text="«Стелла»  •  «Мита» / «Стела» • «Міта»",
                                font=("Arial", 10, "bold"), fill=self.colors["text"], anchor="w")

        # ================ РЕЖИМ РАБОТЫ ================
        self.rounded_rect(32, 488, 221, 525, 12, self.colors["panel2"], self.colors["line"])
        self.canvas.create_text(42, 500, text=T("mode"),
                                font=("Arial", 8, "bold"), fill=self.colors["dim"], anchor="w")
        self.mode_label = self.canvas.create_text(42, 518, text=get_mode_name(_mita_mode),
                                                  font=("Arial", 9, "bold"), fill=self.colors["primary2"], anchor="w")

        self.mode_button = tk.Button(
            self.root,
            text=T("change_mode"),
            font=("Arial", 8, "bold"),
            bg="#2b1835",
            fg="#a58ca9",
            relief="flat",
            bd=0,
            cursor="hand2",
            command=self.change_mode
        )
        self.mode_button.place(x=42, y=530, width=180, height=22)

        # ================ КНОПКА РУЧНОГО ВВОДА ================
        self.rounded_rect(32, 556, 221, 596, 12, self.colors["panel2"], self.colors["line"])
        self.canvas.create_text(42, 568, text=T("manual_input"),
                                font=("Arial", 8, "bold"), fill=self.colors["dim"], anchor="w")

        self.manual_record_button = tk.Button(
            self.root,
            text=T("manual_input_btn"),
            font=("Arial", 9, "bold"),
            bg="#ff5caa",
            fg="white",
            relief="flat",
            bd=0,
            cursor="hand2",
            command=self.toggle_manual_record
        )
        self.manual_record_button.place(x=42, y=582, width=180, height=26)

        # ================ КНОПКА ОТКЛЮЧЕНИЯ ГОЛОСА ================
        self.rounded_rect(32, 600, 221, 635, 12, self.colors["panel2"], self.colors["line"])
        self.canvas.create_text(42, 612, text=T("mita_voice"),
                                font=("Arial", 8, "bold"), fill=self.colors["dim"], anchor="w")

        self.tts_mute_button = tk.Button(
            self.root,
            text=T("voice_on"),
            font=("Arial", 9, "bold"),
            bg="#2b1835",
            fg="#4aff8b",
            relief="flat",
            bd=0,
            cursor="hand2",
            command=self.toggle_tts_mute
        )
        self.tts_mute_button.place(x=42, y=625, width=180, height=26)

        # ================ ИСПРАВИТЕЛЬ ТЕКСТА ================
        self.rounded_rect(32, 640, 221, 675, 12, self.colors["panel2"], self.colors["line"])
        self.canvas.create_text(42, 652, text=T("text_corrector"),
                                font=("Arial", 8, "bold"), fill=self.colors["dim"], anchor="w")

        self.corrector_button = tk.Button(
            self.root,
            text=T("corrector_off"),
            font=("Arial", 9, "bold"),
            bg="#2b1835",
            fg="#a58ca9",
            relief="flat",
            bd=0,
            cursor="hand2",
            command=self.toggle_corrector
        )
        self.corrector_button.place(x=42, y=667, width=180, height=26)

        # ================ НАСТРОЙКИ ЯЗЫКА ================
        self.rounded_rect(32, 700, 221, 735, 12, self.colors["panel2"], self.colors["line"])
        self.lang_label = self.canvas.create_text(42, 712, text="🌍 " + T("settings_language"),
                                                  font=("Arial", 8, "bold"), fill=self.colors["dim"], anchor="w")

        self.lang_button = tk.Button(
            self.root,
            text="🇺🇦 Українська" if UI_LANGUAGE == "ru" else "🇷🇺 Русский",
            font=("Arial", 9, "bold"),
            bg="#2b1835",
            fg="#a58ca9",
            relief="flat",
            bd=0,
            cursor="hand2",
            command=self.toggle_language
        )
        self.lang_button.place(x=42, y=725, width=180, height=24)

        # Информация о ключе
        self.key_info_text = self.canvas.create_text(42, 768, text="",
                                                     font=("Arial", 9, "bold"),
                                                     fill=self.colors["text"], anchor="w")
        self.update_key_info()

        self.change_key_btn = tk.Button(
            self.root,
            text=T("change_key"),
            font=("Arial", 8, "bold"),
            bg="#2b1835",
            fg="#a58ca9",
            relief="flat",
            bd=0,
            cursor="hand2",
            command=self.change_key
        )
        self.change_key_btn.place(x=42, y=785, width=180, height=22)

        # ================ ПОЛЗУНОК ГРОМКОСТИ ================
        self.rounded_rect(32, 815, 221, 845, 12, self.colors["panel2"], self.colors["line"])
        self.volume_label_id = self.canvas.create_text(42, 827, text=f"🔊 {T('volume')}: {_music_volume}%",
                                                       font=("Arial", 8, "bold"), fill=self.colors["primary2"],
                                                       anchor="w")

        self.volume_scale = tk.Scale(
            self.root,
            from_=0,
            to=200,
            orient=tk.HORIZONTAL,
            length=160,
            width=8,
            sliderlength=14,
            bg="#150d1e",
            fg="#ff8bc4",
            activebackground="#ff5caa",
            highlightthickness=0,
            troughcolor="#2b1835",
            bd=0,
            relief="flat",
            font=("Arial", 6),
            command=self.on_volume_change
        )
        self.volume_scale.place(x=48, y=840, width=170, height=28)
        self.volume_scale.set(_music_volume)

        # ================ КНОПКА ОСТАНОВКИ МУЗЫКИ ================
        self.music_stop_button = tk.Button(
            self.root,
            text=T("music_stop"),
            font=("Arial", 8, "bold"),
            bg="#2b1835",
            fg="#a58ca9",
            relief="flat",
            bd=1,
            cursor="hand2",
            command=self.stop_music_click,
            state=tk.DISABLED
        )
        self.music_stop_button.place(x=42, y=875, width=180, height=22)

        # Центральная панель
        self.rounded_rect(253, 92, 790, 370, 24,
                          self.colors["panel"], self.colors["line"])
        self.canvas.create_text(522, 118, text="Mita CORE",
                                font=("Arial", 9, "bold"), fill=self.colors["dim"])

        self.cx, self.cy = 522, 260

        self.aura = []
        for radius, outline, width in [(168, "#24142d", 1), (145, "#3b1b43", 1),
                                       (122, "#54204e", 2), (101, "#70265c", 1)]:
            self.aura.append(self.canvas.create_oval(
                self.cx - radius, self.cy - radius,
                self.cx + radius, self.cy + radius,
                outline=outline, width=width
            ))

        self.core_outer = self.canvas.create_oval(
            self.cx - 76, self.cy - 76, self.cx + 76, self.cy + 76,
            fill="#170b1d", outline="#ff5caa", width=2
        )
        self.core_inner = self.canvas.create_oval(
            self.cx - 58, self.cy - 58, self.cx + 58, self.cy + 58,
            fill="#221024", outline="#a77bff", width=1
        )

        self.heart_size = 45
        self.heart = self.create_heart(self.cx, self.cy, self.heart_size, self.colors["primary"])
        self.heart_glow = self.canvas.create_oval(
            self.cx - 70, self.cy - 70, self.cx + 70, self.cy + 70,
            fill="", outline="#ff5caa", width=3, stipple="gray75"
        )

        self.orbit_elements = []
        for i in range(10):
            a = i * (math.pi * 2 / 10)
            obj = self.canvas.create_text(self.cx, self.cy, text="✦",
                                          font=("Arial", 9 + i % 3, "bold"),
                                          fill=self.colors["purple"])
            self.orbit_elements.append({"id": obj, "angle": a,
                                        "radius": 130 + (i % 2) * 24,
                                        "speed": 0.008 + i * 0.0012})

        self.status_text = self.canvas.create_text(
            self.cx, 440, text=T("ready"),
            font=("Arial", 14, "bold"), fill=self.colors["primary2"]
        )
        self.status_sub = self.canvas.create_text(
            self.cx, 468, text=T("waiting"),
            font=("Arial", 10), fill=self.colors["muted"]
        )
        self.create_bars()

        self.rounded_rect(253, 386, 790, 690, 24, self.colors["panel"], self.colors["line"])
        self.canvas.create_text(272, 410, text=T("chat_with_mita"),
                                font=("Arial", 10, "bold"), fill=self.colors["primary2"], anchor="w")
        self.canvas.create_text(272, 430, text=T("chat_hint"),
                                font=("Arial", 8), fill=self.colors["muted"], anchor="w")

        self.chat_frame = tk.Frame(self.root, bg=self.colors["panel"])
        self.chat_frame.place(x=263, y=448, width=517, height=160)

        self.chat_display = scrolledtext.ScrolledText(
            self.chat_frame, bg="#0d0812", fg="#fff4fb",
            font=("Arial", 10), wrap=tk.WORD, height=8,
            bd=0, relief="flat", highlightthickness=1,
            highlightcolor="#2b1835", highlightbackground="#2b1835"
        )
        self.chat_display.pack(fill=tk.BOTH, expand=True)
        self.chat_display.config(state=tk.DISABLED)

        self.chat_menu = Menu(self.root, tearoff=0, bg="#150d1e", fg="#fff4fb",
                              activebackground="#2b1835", activeforeground="#ff8bc4")
        self.chat_menu.add_command(label="📋 Копировать сообщение", command=self.copy_selected_message)
        self.chat_menu.add_command(label="📋 Копировать всё", command=self.copy_all_chat)
        self.chat_menu.add_separator()
        self.chat_menu.add_command(label="🗑️ Очистить чат", command=self.clear_chat)

        self.chat_display.bind("<Button-3>", self.show_chat_menu)
        self.chat_display.bind("<Control-c>", lambda e: self.copy_selected_message())
        self.chat_display.bind("<Control-a>", lambda e: self.select_all_chat())
        self.chat_display.bind("<Button-1>", self.on_chat_click)
        self.chat_display.bind("<ButtonRelease-1>", self.on_chat_release)

        # Приветствие
        hello_msg = f"{T('mita_greeting')}\n{T('mita_mode')}{get_mode_name(_mita_mode)}\n{T('mita_corrector')}{T('corrector_off_info') if not _text_corrector_enabled else T('corrector_on_info')}\n{T('mita_lang')}\n{T('mita_help')}"
        self.add_chat_message("Мита", hello_msg, is_mita=True)

        self.input_frame = tk.Frame(self.root, bg=self.colors["panel"])
        self.input_frame.place(x=263, y=608, width=517, height=45)

        self.chat_input = tk.Entry(self.input_frame, bg="#150d1e", fg="#fff4fb",
                                   font=("Arial", 11), bd=0, relief="flat",
                                   insertbackground="#ff8bc4")
        self.chat_input.pack(side=tk.LEFT, fill=tk.BOTH, expand=True, padx=(0, 5))

        self.input_menu = Menu(self.root, tearoff=0, bg="#150d1e", fg="#fff4fb",
                               activebackground="#2b1835", activeforeground="#ff8bc4")
        self.input_menu.add_command(label="✂️ Вырезать", command=self.cut_input_text)
        self.input_menu.add_command(label="📋 Копировать", command=self.copy_input_text)
        self.input_menu.add_command(label="📎 Вставить", command=self.paste_input_text)
        self.input_menu.add_separator()
        self.input_menu.add_command(label="🗑️ Очистить", command=self.clear_input_text)
        self.input_menu.add_separator()
        self.input_menu.add_command(label="🔄 Выделить всё", command=self.select_all_input)

        self.chat_input.bind("<Button-1>", self.on_chat_click)
        self.chat_input.bind("<ButtonRelease-1>", self.on_chat_release)
        self.chat_input.bind("<Return>", lambda e: self.send_chat_message())
        self.chat_input.bind("<Button-3>", self.show_input_menu)
        self.chat_input.bind("<Control-a>", lambda e: self.select_all_input())
        self.chat_input.bind("<Control-c>", lambda e: self.copy_input_text())
        self.chat_input.bind("<Control-v>", lambda e: self.paste_input_text())
        self.chat_input.bind("<Control-x>", lambda e: self.cut_input_text())

        self.send_button = tk.Button(self.input_frame, text="➤", font=("Arial", 14, "bold"),
                                     bg="#ff5caa", fg="white", bd=0, relief="flat",
                                     cursor="hand2", command=self.send_chat_message)
        self.send_button.pack(side=tk.RIGHT, padx=(0, 0))

        self.rounded_rect(808, 92, 1082, 690, 20, self.colors["panel"], self.colors["line"])
        self.canvas.create_text(833, 120, text=T("commands_title"),
                                font=("Arial", 9, "bold"), fill=self.colors["dim"], anchor="w")

        commands = [
            ("01", T("cmd_launch"), "[программа]"),
            ("02", T("cmd_open"), "[сайт]"),
            ("03", T("cmd_close"), "[программа]"),
            ("04", T("cmd_write"), "[текст]"),
            ("05", T("cmd_minimize"), "[окно]"),
            ("06", T("cmd_screenshot"), ""),
            ("07", T("cmd_copy"), ""),
            ("08", T("cmd_paste"), ""),
            ("09", T("cmd_lang"), ""),
            ("10", T("cmd_click"), ""),
            ("11", T("cmd_move"), "[на 1/2 экран]"),
            ("12", T("cmd_song"), "[название]"),
            ("13", T("cmd_stop_music"), ""),
            ("14", T("cmd_louder"), ""),
            ("15", T("cmd_quieter"), ""),
            ("16", T("cmd_volume"), "[0-200%]"),
        ]

        y = 158
        for num, title, tail in commands:
            self.canvas.create_text(833, y, text=num, font=("Arial", 8, "bold"),
                                    fill=self.colors["primary"], anchor="w")
            self.canvas.create_text(862, y, text=title, font=("Arial", 9, "bold"),
                                    fill=self.colors["text"], anchor="w")
            if tail:
                self.canvas.create_text(862, y + 17, text=tail, font=("Arial", 8),
                                        fill=self.colors["muted"], anchor="w")
            y += 46

        self.rounded_rect(290, 600, 754, 638, 12, self.colors["panel2"], self.colors["line"])
        self.command_text = self.canvas.create_text(307, 619, text="Последняя команда: —",
                                                    font=("Arial", 9), fill=self.colors["muted"], anchor="w")

        self.canvas.create_line(32, 665, 221, 665, fill=self.colors["line"])
        self.canvas.create_text(42, 685, text="RAM", font=("Arial", 8, "bold"),
                                fill=self.colors["dim"], anchor="w")
        self.ram_text = self.canvas.create_text(205, 685, text=f"{self.get_ram_usage()} MB",
                                                font=("Arial", 8, "bold"), fill=self.colors["muted"], anchor="e")

        self.canvas.create_text(820, 685, text="AUDIO", font=("Arial", 8, "bold"),
                                fill=self.colors["dim"], anchor="w")
        self.audio_indicator = self.canvas.create_text(930, 685, text="⚪", font=("Arial", 12, "bold"),
                                                       fill=self.colors["dim"], anchor="e")

        self.canvas.create_text(820, 740, text="Mita AI  •  2026",
                                font=("Arial", 8), fill=self.colors["dim"], anchor="w")

        self.create_particles()
        self.create_stars()
        self.create_floating_stars()
        self.current_tab = 0
        self.switch_tab(0)

    def toggle_language(self):
        global UI_LANGUAGE
        if UI_LANGUAGE == "ru":
            UI_LANGUAGE = "ua"
        else:
            UI_LANGUAGE = "ru"
        _save_ui_language()
        self.refresh_ui()

    def refresh_ui(self):
        self.root.title(T("app_title"))
        self.mode_button.config(text=T("change_mode"))
        self.manual_record_button.config(text=T("manual_input_btn"))
        self.tts_mute_button.config(text=T("voice_on") if not self.tts_muted else T("voice_off"))
        self.corrector_button.config(text=T("corrector_on") if _text_corrector_enabled else T("corrector_off"))
        self.lang_button.config(text="🇺🇦 Українська" if UI_LANGUAGE == "ru" else "🇷🇺 Русский")
        self.change_key_btn.config(text=T("change_key"))
        self.music_stop_button.config(text=T("music_stop"))
        self.canvas.itemconfig(self.engine_text, text=f"Mita AI {T('online')}")
        nav_labels = [
            (T("nav_main"), "⌂"),
            (T("nav_chat"), "💬"),
            (T("nav_commands"), "⌘"),
            (T("nav_settings"), "⚙")
        ]
        for i, (bg, txt) in enumerate(self.nav_items):
            label, icon = nav_labels[i]
            self.canvas.itemconfig(txt, text=f"{icon}   {label}")
        self.canvas.itemconfig(self.status_text, text=T("ready"))
        self.canvas.itemconfig(self.status_sub, text=T("waiting"))
        self.canvas.itemconfig(self.volume_label_id, text=f"🔊 {T('volume')}: {_music_volume}%")
        msg = T("lang_changed")
        self.add_chat_message("Мита", msg, is_mita=True)
        speak(msg, force=True)

    def toggle_corrector(self):
        self.corrector_enabled = not self.corrector_enabled
        set_text_corrector(self.corrector_enabled)
        if self.corrector_enabled:
            self.corrector_button.config(text=T("corrector_on"), bg="#1a3d2b", fg="#4aff8b")
            msg = T("corrector_on_text")
            self.add_chat_message("Мита", f"📝 {T('text_corrector')}: {msg}", is_mita=True)
            speak(msg, force=True)
        else:
            self.corrector_button.config(text=T("corrector_off"), bg="#2b1835", fg="#a58ca9")
            msg = T("corrector_off_text")
            self.add_chat_message("Мита", f"📝 {T('text_corrector')}: {msg}", is_mita=True)
            speak(msg, force=True)

    def change_mode(self):
        dialog = tk.Toplevel(self.root)
        dialog.title(T("mode"))
        dialog.geometry("400x350")
        dialog.configure(bg="#09070d")
        dialog.resizable(False, False)
        dialog.transient(self.root)
        dialog.grab_set()

        tk.Label(dialog, text="🎯 " + T("mode"),
                 font=("Arial", 14, "bold"), bg="#09070d", fg="#ff8bc4").pack(pady=(20, 10))
        tk.Label(dialog,
                 text="Как Мита должна отвечать на команды?" if UI_LANGUAGE == "ru" else "Як Міта повинна відповідати на команди?",
                 font=("Arial", 10), bg="#09070d", fg="#a58ca9").pack(pady=(0, 15))

        mode_var = tk.StringVar(value=_mita_mode)

        modes = [
            (MODE_SYSTEM, T("mode_system"),
             "Выполняет системные команды" if UI_LANGUAGE == "ru" else "Виконує системні команди"),
            (MODE_AI, T("mode_ai"),
             "Отвечает через искусственный интеллект" if UI_LANGUAGE == "ru" else "Відповідає через штучний інтелект"),
            (MODE_ALL, T("mode_all"),
             "И команды, и ИИ (рекомендуется)" if UI_LANGUAGE == "ru" else "І команди, і ШІ (рекомендується)"),
        ]

        for mode, label, desc in modes:
            frame = tk.Frame(dialog, bg="#150d1e", relief="flat", bd=0)
            frame.pack(fill="x", padx=20, pady=3)

            rb = tk.Radiobutton(
                frame,
                text=label,
                variable=mode_var,
                value=mode,
                font=("Arial", 11, "bold"),
                bg="#150d1e",
                fg="#fff4fb",
                selectcolor="#2b1835",
                activebackground="#150d1e",
                activeforeground="#ff8bc4",
                relief="flat",
                bd=0,
                cursor="hand2"
            )
            rb.pack(side="left", padx=10, pady=5)

            tk.Label(frame, text=desc, font=("Arial", 8),
                     bg="#150d1e", fg="#a58ca9").pack(side="right", padx=10)

        def apply_mode():
            mode = mode_var.get()
            set_mita_mode(mode)
            self.update_mode_display()
            dialog.destroy()
            msg = T("mode_changed").format(get_mode_name(mode))
            self.add_chat_message("Мита", msg, is_mita=True)
            speak(T("mode_changed_text").format(get_mode_name(mode)), force=True)

        btn = tk.Button(
            dialog,
            text=T("apply"),
            command=apply_mode,
            font=("Arial", 10, "bold"),
            bg="#ff5caa",
            fg="white",
            activebackground="#ff8bc4",
            relief="flat",
            bd=0,
            cursor="hand2"
        )
        btn.pack(pady=20)

        dialog.bind("<Return>", lambda e: apply_mode())

    def update_mode_display(self):
        if self.mode_label:
            self.canvas.itemconfig(self.mode_label, text=get_mode_name(_mita_mode))

    def toggle_tts_mute(self):
        global TTS_VOLUME
        self.tts_muted = not self.tts_muted
        if self.tts_muted:
            self.tts_mute_button.config(text=T("voice_off"), bg="#ff557f", fg="white")
            stop_tts()
            TTS_VOLUME = 0.0
            self.add_chat_message("Мита", T("voice_off"), is_mita=True)
            self.canvas.itemconfig(self.status_sub, text=T("voice_off_status"))
        else:
            self.tts_mute_button.config(text=T("voice_on"), bg="#2b1835", fg="#4aff8b")
            TTS_VOLUME = 1.0
            self.add_chat_message("Мита", T("voice_on"), is_mita=True)
            self.canvas.itemconfig(self.status_sub, text=T("waiting"))
            speak(T("voice_on_text"), force=True)

    def toggle_manual_record(self):
        if self.is_manual_recording:
            self.stop_manual_recording()
        else:
            self.start_manual_recording()

    def start_manual_recording(self):
        if self.is_manual_recording:
            return
        self.is_manual_recording = True
        self.manual_audio_buffer = []
        self.manual_record_button.config(text=T("stop_record"), bg="#ff557f", fg="white")
        msg = "🎤 " + (T("manual_sub"))
        self.add_chat_message("Мита", msg, is_mita=True)
        self.canvas.itemconfig(self.status_text, text=T("manual_recording"), fill="#ff8bc4")
        self.canvas.itemconfig(self.status_sub, text=T("manual_sub"))
        self.manual_recording_thread = threading.Thread(target=self._manual_record_loop, daemon=True)
        self.manual_recording_thread.start()

    def stop_manual_recording(self):
        if not self.is_manual_recording:
            return
        self.is_manual_recording = False
        self.manual_record_button.config(text=T("manual_input_btn"), bg="#ff5caa", fg="white")
        self.canvas.itemconfig(self.status_text, text=T("processing"), fill=self.colors["purple"])
        self.canvas.itemconfig(self.status_sub, text=T("processing_sub"))
        if self.manual_audio_buffer:
            threading.Thread(target=self._process_manual_audio, daemon=True).start()
        else:
            msg = T("no_audio")
            self.add_chat_message("Мита", msg, is_mita=True)
            self._reset_manual_state()

    def _manual_record_loop(self):
        try:
            sample_rate = 16000
            chunk_size = 1024
            with sd.InputStream(
                    samplerate=sample_rate,
                    channels=1,
                    dtype='float32',
                    blocksize=chunk_size,
                    callback=self._manual_audio_callback
            ):
                while self.is_manual_recording:
                    time.sleep(0.05)
        except Exception as e:
            print(f"[Manual Record Error]: {e}")
            self.root.after(0, lambda: self.add_chat_message("Мита", T("error_text").format(e), is_mita=True))
            self.root.after(0, lambda: self.manual_record_button.config(
                text=T("manual_input_btn"), bg="#ff5caa", fg="white"))
            self.is_manual_recording = False

    def _manual_audio_callback(self, indata, frames, time, status):
        if self.is_manual_recording:
            self.manual_audio_buffer.append(indata.copy())

    def _process_manual_audio(self):
        try:
            if not self.manual_audio_buffer:
                msg = T("no_data")
                self.root.after(0, lambda: self.add_chat_message("Мита", msg, is_mita=True))
                self.root.after(0, self._reset_manual_state)
                return
            full_audio = np.concatenate(self.manual_audio_buffer)
            self.manual_audio_buffer = []
            sample_rate = 16000
            stream = recognizer.create_stream()
            stream.accept_waveform(sample_rate, full_audio)
            recognizer.decode_stream(stream)
            recognized_text = stream.result.text.strip()
            if recognized_text:
                print(f"[Ручной ввод]: {recognized_text}")
                self.root.after(0, lambda: self.add_chat_message("Вы", f"🎤 {recognized_text}"))
                self.root.after(0, lambda: self._process_manual_command(recognized_text))
            else:
                msg = T("recognition_failed")
                self.root.after(0, lambda: self.add_chat_message("Мита", msg, is_mita=True))
                self.root.after(0, self._reset_manual_state)
        except Exception as e:
            print(f"[Manual Process Error]: {e}")
            self.root.after(0, lambda: self.add_chat_message("Мита", T("error_text").format(e), is_mita=True))
            self.root.after(0, self._reset_manual_state)

    def _process_manual_command(self, text: str):
        self.is_processing = True
        self.canvas.itemconfig(self.status_text, text=T("processing"), fill=self.colors["purple"])
        self.canvas.itemconfig(self.status_sub, text=T("processing_sub"))
        try:
            write_verbs_ru = ['напиши', 'напечатай', 'пиши', 'печатай', 'набор']
            write_verbs_ua = ['напишіть', 'надрукуй', 'пиши', 'друкуй']
            write_verbs = write_verbs_ru + write_verbs_ua
            if any(verb in text.lower() for verb in write_verbs):
                words = text.lower().split()
                for verb in write_verbs:
                    if verb in words:
                        verb_index = words.index(verb)
                        text_to_type = " ".join(words[verb_index + 1:]).strip()
                        if text_to_type:
                            if not play_sound("write"):
                                play_sound("ok")
                            if _text_corrector_enabled:
                                corrected = correct_text(text_to_type)
                                if corrected != text_to_type:
                                    msg = T("typing_corrected_msg").format(text_to_type, corrected)
                                    self.root.after(0, lambda: self.add_chat_message("Мита", msg, is_mita=True))
                                    speak(T("corrector_on_text"), force=True)
                                    text_to_type = corrected
                                else:
                                    self.root.after(0, lambda: self.add_chat_message("Мита",
                                                                                     T("typing_writing").format(
                                                                                         text_to_type),
                                                                                     is_mita=True))
                            else:
                                self.root.after(0, lambda: self.add_chat_message("Мита",
                                                                                 T("typing_writing").format(
                                                                                     text_to_type),
                                                                                 is_mita=True))
                            time.sleep(0.3)
                            keyboard.write(text_to_type, delay=0.02)
                            msg = T("typing_corrected") if _text_corrector_enabled else T("typing")
                            self.root.after(0, lambda: self.add_chat_message("Мита", msg, is_mita=True))
                            self.root.after(0, self._reset_manual_state)
                            return

            result = None
            if _mita_mode in [MODE_SYSTEM, MODE_ALL]:
                result = self.process_chat_system_command(text)
                if result:
                    self.root.after(0, lambda: self.add_chat_message("Мита", result, is_mita=True))
                    speak(result)
                    self.root.after(0, self._reset_manual_state)
                    return

            if _mita_mode in [MODE_AI, MODE_ALL]:
                response = ask_groq_chat(text, self.chat_history)
                self.root.after(0, lambda: self.add_chat_message("Мита", response, is_mita=True))
                self.chat_history.append({"role": "assistant", "content": response})
                speak(response)
            else:
                if not result:
                    response = T("command_not_found")
                    self.root.after(0, lambda: self.add_chat_message("Мита", response, is_mita=True))
                    speak(response)

        except Exception as e:
            self.root.after(0, lambda: self.add_chat_message("Мита", T("error_text").format(e), is_mita=True))
        self.root.after(0, self._reset_manual_state)

    def _reset_manual_state(self):
        self.is_processing = False
        self.is_manual_recording = False
        self.manual_audio_buffer = []
        self.manual_record_button.config(text=T("manual_input_btn"), bg="#ff5caa", fg="white")
        self.canvas.itemconfig(self.status_text, text=T("ready"), fill=self.colors["primary2"])
        self.canvas.itemconfig(self.status_sub, text=T("waiting"))

    def update_music_button(self, is_playing: bool):
        if self.music_stop_button:
            if is_playing:
                self.music_stop_button.config(text=T("music_stop_click"), bg="#ff557f", fg="white", state=tk.NORMAL)
            else:
                self.music_stop_button.config(text=T("music_stop"), bg="#2b1835", fg="#a58ca9", state=tk.DISABLED)

    def stop_music_click(self):
        stop_music()
        msg = T("music_stopped_click")
        self.add_chat_message("Мита", msg, is_mita=True)
        self.update_music_button(False)
        speak(T("music_stopped_voice"), force=True)

    def on_volume_change(self, value):
        global _music_volume
        vol = int(float(value))
        set_music_volume(vol)
        self.update_volume_display()

    def update_volume_display(self):
        global _music_volume
        if self.volume_scale:
            self.volume_scale.set(_music_volume)
        if self.volume_label_id:
            self.canvas.itemconfig(self.volume_label_id, text=f"🔊 {T('volume')}: {_music_volume}%")
        if _is_music_mode:
            self.canvas.itemconfig(self.status_text, text=f"🎵 МУЗЫКА ИГРАЕТ ({_music_volume}%) 🎵", fill="#ff8bc4")

    def show_input_menu(self, event):
        try:
            self.input_menu.post(event.x_root, event.y_root)
        except:
            pass

    def cut_input_text(self):
        try:
            self.chat_input.event_generate("<<Cut>>")
        except:
            pass

    def copy_input_text(self):
        try:
            self.chat_input.event_generate("<<Copy>>")
            self.root.after(100, lambda: self.show_temp_status(T("copy_success")))
        except:
            pass

    def paste_input_text(self):
        try:
            self.chat_input.event_generate("<<Paste>>")
            self.root.after(100, lambda: self.show_temp_status("✅ Вставлено!"))
        except:
            pass

    def clear_input_text(self):
        self.chat_input.delete(0, tk.END)
        self.root.after(100, lambda: self.show_temp_status("🗑️ Очищено!" if UI_LANGUAGE == "ru" else "🗑️ Очищено!"))

    def select_all_input(self):
        self.chat_input.select_range(0, tk.END)
        self.chat_input.focus_set()
        return "break"

    def show_temp_status(self, text):
        self.canvas.itemconfig(self.status_sub, text=text)
        self.root.after(1500, lambda: self.canvas.itemconfig(self.status_sub, text=T("waiting")))

    def select_all_chat(self):
        self.chat_display.config(state=tk.NORMAL)
        self.chat_display.tag_add(tk.SEL, "1.0", tk.END)
        self.chat_display.mark_set(tk.INSERT, "1.0")
        self.chat_display.see(tk.INSERT)
        self.chat_display.config(state=tk.DISABLED)
        return "break"

    def on_chat_click(self, event):
        self.is_dragging = False
        event.widget.focus_set()

    def on_chat_release(self, event):
        self.is_dragging = False

    def show_chat_menu(self, event):
        try:
            self.chat_menu.post(event.x_root, event.y_root)
        except:
            pass

    def copy_selected_message(self):
        try:
            selected = self.chat_display.get(tk.SEL_FIRST, tk.SEL_LAST)
            if selected:
                self.root.clipboard_clear()
                self.root.clipboard_append(selected.strip())
                self.root.update()
                self.show_temp_status(T("copy_success"))
                return
        except tk.TclError:
            pass
        try:
            content = self.chat_display.get("1.0", tk.END)
            lines = content.split('\n')
            mita_messages = []
            for i, line in enumerate(lines):
                if '🌸 Мита' in line:
                    msg = []
                    for j in range(i + 1, len(lines)):
                        if '🌸 Мита' in lines[j] or '👤 Вы' in lines[j]:
                            break
                        if lines[j].strip():
                            msg.append(lines[j].strip())
                    if msg:
                        mita_messages.append('\n'.join(msg))
            if mita_messages:
                last_msg = mita_messages[-1]
                self.root.clipboard_clear()
                self.root.clipboard_append(last_msg)
                self.root.update()
                self.show_temp_status(T("copy_success"))
            else:
                self.show_temp_status(T("copy_failed"))
        except Exception as e:
            self.show_temp_status(T("error_text").format(e))

    def copy_all_chat(self):
        try:
            content = self.chat_display.get("1.0", tk.END)
            if content.strip():
                self.root.clipboard_clear()
                self.root.clipboard_append(content.strip())
                self.root.update()
                self.show_temp_status(T("copy_all_success"))
            else:
                self.show_temp_status(T("chat_empty"))
        except Exception as e:
            self.show_temp_status(T("error_text").format(e))

    def clear_chat(self):
        if messagebox.askyesno("Очистка чата" if UI_LANGUAGE == "ru" else "Очищення чату", T("clear_confirm")):
            self.chat_display.config(state=tk.NORMAL)
            self.chat_display.delete("1.0", tk.END)
            self.chat_display.config(state=tk.DISABLED)
            self.chat_history = []
            msg = T("chat_cleared")
            self.add_chat_message("Мита", msg, is_mita=True)
            self.show_temp_status("🗑️ Чат очищен!" if UI_LANGUAGE == "ru" else "🗑️ Чат очищено!")

    def change_key(self):
        if messagebox.askyesno("Смена ключа" if UI_LANGUAGE == "ru" else "Зміна ключа", T("key_change_confirm")):
            _clear_saved_key()
            if KeyLoginWindow(self.root).show():
                self.update_key_info()
                messagebox.showinfo("Успешно" if UI_LANGUAGE == "ru" else "Успішно", T("key_success"))
            else:
                messagebox.showinfo("Отмена" if UI_LANGUAGE == "ru" else "Скасування", T("key_cancel"))
                self.update_key_info()

    def switch_tab(self, idx):
        self.current_tab = idx
        for i, (bg, txt) in enumerate(self.nav_items):
            if i == idx:
                self.canvas.itemconfig(bg, fill=self.colors["panel2"], outline=self.colors["line"])
                self.canvas.itemconfig(txt, fill=self.colors["text"])
            else:
                self.canvas.itemconfig(bg, fill="", outline="")
                self.canvas.itemconfig(txt, fill=self.colors["muted"])
        if idx == 0:
            self.chat_frame.place(x=263, y=448, width=517, height=160)
            self.input_frame.place(x=263, y=608, width=517, height=45)
            self.chat_display.config(height=8)
        elif idx == 1:
            self.chat_frame.place(x=263, y=420, width=517, height=210)
            self.input_frame.place(x=263, y=630, width=517, height=45)
            self.chat_display.config(height=12)
        else:
            self.chat_frame.place(x=263, y=448, width=517, height=160)
            self.input_frame.place(x=263, y=608, width=517, height=45)
            self.chat_display.config(height=8)

    def add_chat_message(self, sender, message, is_mita=False):
        if is_mita and message:
            message = re.sub(r'[\*_`]', '', message)
            message = re.sub(r'\s+', ' ', message).strip()
        self.chat_display.config(state=tk.NORMAL)
        timestamp = datetime.now().strftime("%H:%M")
        if is_mita:
            self.chat_display.insert(tk.END, f"🌸 Мита ({timestamp}):\n", "mita_name")
            tag_name = f"mita_msg_{len(self.message_tags)}"
            start_pos = self.chat_display.index(tk.END)
            self.chat_display.insert(tk.END, f"  {message}\n\n", "mita_text")
            end_pos = self.chat_display.index(tk.END)
            self.chat_display.tag_add(tag_name, start_pos, end_pos)
            self.chat_display.tag_config(tag_name, foreground="#e8d5e0", font=("Arial", 10))
            self.message_tags.append(tag_name)
            self.chat_display.tag_config("mita_name", foreground="#ff8bc4", font=("Arial", 10, "bold"))
            self.chat_display.tag_config("mita_text", foreground="#e8d5e0", font=("Arial", 10))
            self.chat_display.tag_bind(tag_name, "<Enter>", lambda e: self.chat_display.config(cursor="hand2"))
            self.chat_display.tag_bind(tag_name, "<Leave>", lambda e: self.chat_display.config(cursor=""))
        else:
            self.chat_display.insert(tk.END, f"👤 Вы ({timestamp}):\n", "user_name")
            self.chat_display.insert(tk.END, f"  {message}\n\n", "user_text")
            self.chat_display.tag_config("user_name", foreground="#a77bff", font=("Arial", 10, "bold"))
            self.chat_display.tag_config("user_text", foreground="#fff4fb", font=("Arial", 10))
        self.chat_display.see(tk.END)
        self.chat_display.config(state=tk.DISABLED)

    def send_chat_message(self):
        message = self.chat_input.get().strip()
        if not message:
            return
        self.chat_input.delete(0, tk.END)
        self.add_chat_message("Вы", message)
        self.chat_history.append({"role": "user", "content": message})
        self.is_processing = True
        self.canvas.itemconfig(self.status_text, text=T("processing"), fill=self.colors["purple"])
        self.canvas.itemconfig(self.status_sub, text=T("processing_sub"))
        threading.Thread(target=self.process_chat_command, args=(message,), daemon=True).start()

    def process_chat_command(self, message):
        try:
            if _mita_mode in [MODE_SYSTEM, MODE_ALL]:
                result = self.process_chat_system_command(message)
                if result:
                    response = result
                    if _mita_mode == MODE_SYSTEM:
                        self.root.after(0, lambda: self.add_chat_message("Мита", response, is_mita=True))
                        speak(response)
                        self.root.after(0, lambda: self.canvas.itemconfig(self.status_text,
                                                                          text=T("ready"),
                                                                          fill=self.colors["primary2"]))
                        self.root.after(0, lambda: self.canvas.itemconfig(self.status_sub,
                                                                          text=T("waiting")))
                        self.is_processing = False
                        return
            if _mita_mode in [MODE_AI, MODE_ALL]:
                if _mita_mode == MODE_AI or not result:
                    response = ask_groq_chat(message, self.chat_history)
                    self.root.after(0, lambda: self.add_chat_message("Мита", response, is_mita=True))
                    self.chat_history.append({"role": "assistant", "content": response})
                    speak(response)
            else:
                if not result:
                    response = T("command_not_found")
                    self.root.after(0, lambda: self.add_chat_message("Мита", response, is_mita=True))
                    speak(response)
        except Exception as e:
            self.root.after(0, lambda: self.add_chat_message("Мита", T("error_text").format(e), is_mita=True))
        self.root.after(0, lambda: self.canvas.itemconfig(self.status_text, text=T("ready"),
                                                          fill=self.colors["primary2"]))
        self.root.after(0, lambda: self.canvas.itemconfig(self.status_sub, text=T("waiting")))
        self.is_processing = False

    def process_chat_system_command(self, text):
        global _music_volume
        cleaned = text.lower().strip()
        words = cleaned.split()

        if process_music_command(cleaned, self):
            return "🎵 " + (T("processing_sub"))

        if 'громкость' in cleaned or 'гучн' in cleaned:
            return T("volume_text").format(_music_volume)

        stop_tts_phrases_ru = ["хватит", "стоп", "прекрати", "замолчи", "хватит читать", "перестань", "остановись",
                               "тихо", "заткнись"]
        stop_tts_phrases_ua = ["досить", "припини", "замовкни", "перестань", "зупинись", "тихо"]
        stop_tts_phrases = stop_tts_phrases_ru + stop_tts_phrases_ua

        if any(p in cleaned for p in stop_tts_phrases):
            stop_tts()
            if not self.tts_muted:
                self.toggle_tts_mute()
            return T("shutting_up")

        if "все окна" in cleaned or "сверни все" in cleaned or "усі вікна" in cleaned or "згорни всі" in cleaned:
            minimize_all_windows()
            return T("minimized_all")

        if "смени язык" in cleaned or "поменяй язык" in cleaned or "зміни мову" in cleaned or "поміняй мову" in cleaned:
            change_language()
            return T("language_changed")

        if "скриншот" in cleaned or "скрин" in cleaned:
            pyautogui.press("printscreen")
            return T("screenshot_done")

        if "скопируй" in cleaned or "скопіюй" in cleaned:
            keyboard.press_and_release('ctrl+c')
            return T("copied")

        if "вставь" in cleaned or "встав" in cleaned or "вставте" in cleaned:
            keyboard.press_and_release('ctrl+v')
            return T("pasted")

        write_verbs_ru = ['напиши', 'напечатай', 'пиши', 'печатай']
        write_verbs_ua = ['напишіть', 'надрукуй', 'пиши', 'друкуй']
        write_verbs = write_verbs_ru + write_verbs_ua

        for verb in write_verbs:
            if verb in cleaned:
                verb_index = -1
                for i, w in enumerate(words):
                    if w == verb:
                        verb_index = i
                        break
                if verb_index != -1:
                    text_to_type = " ".join(words[verb_index + 1:]).strip()
                    if text_to_type:
                        if not play_sound("write"):
                            play_sound("ok")
                        if _text_corrector_enabled:
                            corrected = correct_text(text_to_type)
                            if corrected != text_to_type:
                                msg = T("typing_corrected_msg").format(text_to_type, corrected)
                                self.add_chat_message("Мита", msg, is_mita=True)
                                speak(T("corrector_on_text"), force=True)
                                text_to_type = corrected
                            else:
                                self.add_chat_message("Мита", T("typing_writing").format(text_to_type), is_mita=True)
                        else:
                            self.add_chat_message("Мита", T("typing_writing").format(text_to_type), is_mita=True)
                        time.sleep(0.3)
                        keyboard.write(text_to_type, delay=0.02)
                        if _text_corrector_enabled:
                            return T("typing_corrected")
                        else:
                            return T("typed") + text_to_type

        move_verbs_ru = ["переведи", "перемести", "перекинь", "отправь"]
        move_verbs_ua = ["переведи", "перемісти", "перекинь", "відправ"]
        move_verbs = move_verbs_ru + move_verbs_ua

        for verb in move_verbs:
            if verb in cleaned:
                if "1" in cleaned or "первый" in cleaned or "один" in cleaned or "перший" in cleaned:
                    monitor_num = 1
                elif "2" in cleaned or "второй" in cleaned or "два" in cleaned or "другий" in cleaned:
                    monitor_num = 2
                else:
                    return T("move_to_monitor_ask")
                target_window = None
                apps = ["дискорд", "роблокс", "стим", "телеграм", "браузер", "хром", "яндекс",
                        "спотифай", "дота", "кс", "калькулятор", "блокнот", "роблокс"]
                for app in apps:
                    if app in cleaned:
                        target_window = app
                        break
                if not target_window:
                    skip_words = move_verbs + ["на", "в", "экран", "монитор", "1", "2", "первый", "второй", "один",
                                               "два", "на", "в", "екран", "монітор", "перший", "другий"]
                    for w in words:
                        if w not in skip_words and len(w) > 2:
                            target_window = w
                            break
                if target_window:
                    result = move_window_to_monitor(target_window, monitor_num)
                    if result:
                        return T("moving_window").format(monitor_num)
                    else:
                        return T("move_failed")
                else:
                    win = gw.getActiveWindow()
                    if win:
                        PRIMARY_MONITOR_WIDTH = 1920
                        new_x = 100 if monitor_num == 1 else PRIMARY_MONITOR_WIDTH + 100
                        try:
                            if win.isMinimized:
                                win.restore()
                            if win.isMaximized:
                                win.restore()
                            win.moveTo(new_x, 100)
                            win.maximize()
                            play_sound("move")
                            return T("window_moved").format(monitor_num)
                        except:
                            return T("window_move_failed")
                    return T("no_active_window")

        app_verbs_ru = ["запусти", "запустил", "включи", "запустить", "включить"]
        app_verbs_ua = ["запусти", "увімкни", "відкрий", "відкрити"]
        app_verbs = app_verbs_ru + app_verbs_ua

        for verb in app_verbs:
            if verb in cleaned:
                target = " ".join([w for w in words if w != verb]).strip()
                if target:
                    if launch_application(target):
                        return T("app_launching").format(target)
                    else:
                        return T("app_not_found").format(target)

        web_verbs_ru = ["открой", "открыть", "перейди", "перейти", "покажи"]
        web_verbs_ua = ["відкрий", "відкрити", "перейди", "перейти", "покажи"]
        web_verbs = web_verbs_ru + web_verbs_ua

        for verb in web_verbs:
            if verb in cleaned:
                target = " ".join([w for w in words if w != verb]).strip()
                if target:
                    open_website(target)
                    return T("web_opening").format(target)

        close_verbs_ru = ["закрой", "закрыть", "выключи", "выключить", "убей"]
        close_verbs_ua = ["закрий", "закрити", "вимкни", "вимкнути"]
        close_verbs = close_verbs_ru + close_verbs_ua

        for verb in close_verbs:
            if verb in cleaned:
                target = " ".join([w for w in words if w != verb]).strip()
                if target:
                    if kill_application(target):
                        return T("app_closing").format(target)
                    else:
                        return T("app_close_failed").format(target)

        if cleaned in CONTROL_COMMANDS:
            CONTROL_COMMANDS[cleaned]()
            return T("executed")

        greetings_map = {
            "ты тут": T("im_here"),
            "ты здесь": T("im_here"),
            "привет": T("hello_response_ru") if UI_LANGUAGE == "ru" else T("hello_response_ua"),
            "привіт": T("hello_response_ru") if UI_LANGUAGE == "ru" else T("hello_response_ua"),
            "как дела": T("how_are_you"),
            "як справи": T("how_are_you"),
            "не спишь": T("im_always_here"),
            "не спиш": T("im_always_here"),
        }

        for greeting in greetings_map:
            if greeting in cleaned:
                return greetings_map[greeting]

        return None

    def update_key_info(self):
        saved_key = _get_saved_key()
        if saved_key:
            db = _load_json_file(KEY_DB_FILE)
            info = db.get(saved_key)
            if info:
                remaining = float(info["expires_at"]) - time.time()
                if remaining > 0:
                    key_display = saved_key[:8] + "..." if len(saved_key) > 8 else saved_key
                    time_left = _remaining_text(remaining)
                    self.canvas.itemconfig(self.key_info_text, text=f"🔑 {key_display}  •  {time_left}",
                                           fill=self.colors["text"])
                    self.root.after(10000, self.update_key_info)
                    return
        self.canvas.itemconfig(self.key_info_text, text=T("no_key"), fill=self.colors["danger"])
        self.root.after(10000, self.update_key_info)

    def create_background(self):
        for y in range(self.H):
            t = y / max(1, self.H - 1)
            r = int(9 + 8 * t)
            g = int(7 + 3 * t)
            b = int(13 + 14 * t)
            self.canvas.create_line(0, y, self.W, y, fill=f"#{r:02x}{g:02x}{b:02x}")
        for x in range(-self.H, self.W, 90):
            self.canvas.create_line(x, 0, x + self.H, self.H, fill="#120b18", width=1)

    def create_heart(self, x, y, size, color):
        points = []
        for t in range(0, 360, 5):
            rad = math.radians(t)
            hx = 16 * math.sin(rad) ** 3
            hy = 13 * math.cos(rad) - 5 * math.cos(2 * rad) - 2 * math.cos(3 * rad) - math.cos(4 * rad)
            points.extend([x + hx * size / 16, y - hy * size / 16])
        heart_id = self.canvas.create_polygon(points, fill=color, outline=color, width=2)
        glow_id = self.canvas.create_polygon(points, fill="", outline="#ff8bc4", width=3)
        return {"id": heart_id, "glow": glow_id}

    def create_particles(self):
        symbols = ["✦", "·", "✧", "◇"]
        for _ in range(36):
            self.particles.append({"angle": random.uniform(0, math.pi * 2),
                                   "radius": random.uniform(90, 215),
                                   "speed": random.uniform(.15, .8),
                                   "size": random.randint(7, 12),
                                   "symbol": random.choice(symbols), "id": None})

    def create_stars(self):
        for _ in range(42):
            x = random.randint(15, self.W - 15)
            y = random.randint(85, self.H - 15)
            size = random.choice([2, 2, 3])
            obj = self.canvas.create_oval(x, y, x + size, y + size, fill="#4a2d55", outline="")
            self.stars.append({"id": obj, "x": x, "y": y, "speed": random.uniform(.4, 1.5)})

    def create_floating_stars(self):
        for _ in range(30):
            x = random.randint(50, self.W - 50)
            y = random.randint(100, self.H - 50)
            size = random.randint(8, 18)
            speed_x = random.uniform(-0.8, 0.8)
            speed_y = random.uniform(-0.8, 0.8)
            obj = self.canvas.create_text(x, y, text="✦",
                                          font=("Arial", size, "bold"),
                                          fill="#ff8bc4")
            self.floating_stars.append({
                "id": obj,
                "x": x,
                "y": y,
                "speed_x": speed_x,
                "speed_y": speed_y,
                "size": size,
                "phase": random.uniform(0, math.pi * 2),
                "opacity": random.uniform(0.3, 0.9)
            })

    def create_bars(self):
        self.bars = []
        start_x = 402
        for i in range(25):
            x = start_x + i * 10
            obj = self.canvas.create_rectangle(x, self.cy + 92, x + 5, self.cy + 94,
                                               fill=self.colors["purple"], outline="")
            self.bars.append({"id": obj, "x": x, "base": self.cy + 93,
                              "height": 2.0, "phase": random.uniform(0, math.pi * 2)})

    def get_ram_usage(self):
        try:
            return int(psutil.Process().memory_info().rss / 1024 / 1024)
        except:
            return 0

    def update_heart(self, size, color, glow_intensity=1.0):
        points = []
        beat_offset = 0
        if self.heart_beat_strength > 0.05:
            beat_offset = math.sin(self.beat_phase) * self.heart_beat_strength * 8
        for t in range(0, 360, 5):
            rad = math.radians(t)
            hx = 16 * math.sin(rad) ** 3
            hy = 13 * math.cos(rad) - 5 * math.cos(2 * rad) - 2 * math.cos(3 * rad) - math.cos(4 * rad)
            scale = 1 + beat_offset / 100
            points.extend([self.cx + hx * size / 16 * scale, self.cy - hy * size / 16 * scale])
        self.canvas.coords(self.heart["id"], *points)
        self.canvas.coords(self.heart["glow"], *points)
        self.canvas.itemconfig(self.heart["id"], fill=color, outline=color)
        self.canvas.itemconfig(self.heart_glow, outline=color)
        self.canvas.coords(self.heart_glow, self.cx - 60 - beat_offset, self.cy - 60 - beat_offset,
                           self.cx + 60 + beat_offset, self.cy + 60 + beat_offset)

    def animate_heart_beat(self):
        self.heart_beat_strength += (self.heart_beat_target - self.heart_beat_strength) * 0.15
        bpm_factor = self.heart_bpm / 60.0
        self.beat_phase += 0.025 * bpm_factor * 2

        if self.music_mode:
            self.music_intensity = min(1.0, self.music_intensity + 0.01)
            self.heart_beat_strength = max(self.heart_beat_strength, 0.3 + self.music_intensity * 0.5)
        else:
            self.music_intensity = max(0.0, self.music_intensity - 0.02)

        if self.heart_beat_strength > 0.1:
            glow = 0.5 + self.heart_beat_strength * 0.5
            color_pulse = min(1.0, self.heart_beat_strength * 1.5)
            r = int(255 - (255 - 167) * (1 - color_pulse))
            g = int(92 - 92 * (1 - color_pulse))
            b = int(170 - 170 * (1 - color_pulse))
            color = f"#{r:02x}{g:02x}{b:02x}"
        else:
            glow = 0.3
            color = self.colors["primary"]
        if self.heart_beat_strength > 0.15:
            self.canvas.itemconfig(self.audio_indicator, text="🔴", fill="#ff5caa")
        elif self.heart_beat_strength > 0.05:
            self.canvas.itemconfig(self.audio_indicator, text="🟡", fill="#a77bff")
        else:
            self.canvas.itemconfig(self.audio_indicator, text="⚪", fill=self.colors["dim"])
        return glow, color

    def animate_floating_stars(self):
        for star in self.floating_stars:
            star["x"] += star["speed_x"] + math.sin(self.phase + star["phase"]) * 0.3
            star["y"] += star["speed_y"] + math.cos(self.phase * 0.7 + star["phase"]) * 0.3

            if star["x"] < 50 or star["x"] > self.W - 50:
                star["speed_x"] *= -1
            if star["y"] < 100 or star["y"] > self.H - 50:
                star["speed_y"] *= -1

            if self.music_mode:
                pulse = 1 + self.heart_beat_strength * 0.5 * math.sin(self.beat_phase + star["phase"])
                size = star["size"] * pulse
                color_intensity = int(180 + 75 * self.heart_beat_strength)
                color = f"#{color_intensity:02x}{int(color_intensity * 0.55):02x}{int(color_intensity * 0.75):02x}"
                self.canvas.itemconfig(star["id"], font=("Arial", int(size), "bold"), fill=color)
            else:
                self.canvas.itemconfig(star["id"], font=("Arial", star["size"], "bold"), fill="#4a2d55")

            self.canvas.coords(star["id"], star["x"], star["y"])

    def animate(self):
        if not self.running:
            return

        self.music_mode = _is_music_mode
        self.music_intensity = _music_intensity

        self.angle += 0.035
        self.phase += 0.12
        glow_intensity, heart_color = self.animate_heart_beat()

        mode_text = get_mode_name(_mita_mode)
        corrector_text = "📝⌨️" if _text_corrector_enabled else ""

        if self.music_mode:
            status, sub, color = f"🎵 МУЗЫКА ИГРАЕТ ({_music_volume}%) 🎵", "Наслаждайся ритмом!", "#ff8bc4"
        elif self.is_listening:
            status, sub, color = T("listening"), T("listening_sub"), self.colors["primary"]
        elif self.is_processing:
            status, sub, color = T("processing"), T("processing_sub"), self.colors["purple"]
        elif self.is_manual_recording:
            status, sub, color = T("manual_recording"), T("manual_sub"), "#ff8bc4"
        elif self.tts_muted:
            status, sub, color = f"🔇 {mode_text}", T("voice_off_status"), self.colors["danger"]
        else:
            status, sub, color = f"✧ {mode_text} {corrector_text}".strip(), T("waiting"), self.colors["primary2"]

        pulse_size = 45 + math.sin(self.angle * 2.0) * 4
        if self.heart_beat_strength > 0.1:
            pulse_size = 45 * (1 + math.sin(self.beat_phase) * self.heart_beat_strength * 0.3)
        if self.music_mode:
            pulse_size *= 1.2

        self.update_heart(pulse_size, heart_color, glow_intensity)
        self.canvas.itemconfig(self.status_text, text=status, fill=color)
        self.canvas.itemconfig(self.status_sub, text=sub)
        self.canvas.itemconfig(self.core_outer, outline=color)

        for i, obj in enumerate(self.aura):
            delta = math.sin(self.angle * 1.4 + i) * (2 + i)
            audio_delta = self.heart_beat_strength * 10 * math.sin(self.beat_phase + i)
            if self.music_mode:
                audio_delta *= 2
            base = [168, 145, 122, 101][i]
            total_delta = delta + audio_delta
            self.canvas.coords(obj, self.cx - base - total_delta, self.cy - base - total_delta,
                               self.cx + base + total_delta, self.cy + base + total_delta)
            if self.heart_beat_strength > 0.1:
                alpha = int(100 + 155 * self.heart_beat_strength)
                self.canvas.itemconfig(obj, outline=f"#{alpha:02x}2d{alpha:02x}")

        for item in self.orbit_elements:
            item["angle"] += item["speed"]
            orbit_offset = self.heart_beat_strength * 15 * math.sin(self.beat_phase + item["angle"])
            if self.music_mode:
                orbit_offset *= 1.5
            radius = item["radius"] + orbit_offset
            x = self.cx + math.cos(item["angle"]) * radius
            y = self.cy + math.sin(item["angle"]) * radius
            self.canvas.coords(item["id"], x, y)
            if self.music_mode:
                self.canvas.itemconfig(item["id"], fill="#ff8bc4")
            else:
                self.canvas.itemconfig(item["id"], fill=color)

        for p in self.particles:
            p["angle"] += p["speed"] * 0.01
            particle_pulse = 1 + self.heart_beat_strength * 0.2 * math.sin(self.beat_phase + p["angle"])
            if self.music_mode:
                particle_pulse *= 1.3
            radius = p["radius"] * particle_pulse
            x = self.cx + math.cos(p["angle"]) * radius
            y = self.cy + math.sin(p["angle"]) * radius
            if p["id"] is None:
                p["id"] = self.canvas.create_text(x, y, text=p["symbol"],
                                                  font=("Arial", p["size"]), fill=color)
            else:
                self.canvas.coords(p["id"], x, y)
                if self.music_mode:
                    self.canvas.itemconfig(p["id"], fill="#ff8bc4")
                else:
                    self.canvas.itemconfig(p["id"], fill=color)

        self.animate_floating_stars()

        now = time.time()
        for i, star in enumerate(self.stars):
            v = 0.35 + 0.35 * (math.sin(now * star["speed"] + i) + 1) / 2
            if self.music_mode:
                v *= 1.5
            c = int(55 + 75 * v)
            self.canvas.itemconfig(star["id"], fill=f"#{c:02x}{int(c * 0.55):02x}{int(c * 0.75):02x}")

        for i, bar in enumerate(self.bars):
            if self.music_mode:
                target = 5 + abs(math.sin(self.phase * 1.5 + i * .55)) * random.randint(15, 40)
                bar["height"] += (target - bar["height"]) * .35
                bar_color = "#ff8bc4" if bar["height"] > 20 else self.colors["primary"]
            elif self.is_listening or self.is_processing or self.is_manual_recording:
                target = 5 + abs(math.sin(self.phase + i * .55)) * random.randint(8, 24)
                bar["height"] += (target - bar["height"]) * .28
                bar_color = color
            else:
                audio_boost = self.heart_beat_strength * 20
                target = 2.5 + abs(math.sin(self.phase * .4 + i)) * 2 + audio_boost * abs(
                    math.sin(self.beat_phase + i * 0.5))
                bar["height"] += (target - bar["height"]) * .12
                h = bar["height"]
                bar_color = self.colors["primary"] if h > 10 and self.heart_beat_strength > 0.1 else color
            h = max(2, bar["height"])
            self.canvas.coords(bar["id"], bar["x"], bar["base"] - h, bar["x"] + 5, bar["base"])
            self.canvas.itemconfig(bar["id"], fill=bar_color)

        self.canvas.itemconfig(self.ram_text, text=f"{self.get_ram_usage()} MB")
        self.root.after(20, self.animate)

    def fade_in(self, alpha=0.0):
        if not self.running:
            return
        alpha += 0.06
        if alpha >= 0.98:
            self.root.attributes("-alpha", 0.98)
            return
        self.root.attributes("-alpha", alpha)
        self.root.after(20, lambda: self.fade_in(alpha))

    def set_listening(self, state):
        self.is_listening = state
        if state:
            self.is_processing = False

    def set_processing(self, state):
        self.is_processing = state
        if state:
            self.is_listening = False

    def show_command(self, text):
        short = text.replace("\n", " ").strip()
        if len(short) > 48:
            short = short[:48] + "..."
        self.canvas.itemconfig(self.command_text, text=f"Последняя команда: {short or '—'}")

    def quit(self):
        self.running = False
        if hasattr(self, 'audio_monitor'):
            self.audio_monitor.stop()
        try:
            self.root.quit()
            self.root.destroy()
        except:
            pass
        try:
            stop_music()
        except:
            pass
        try:
            if HAS_EDGE_TTS and _tts_pygame_ready:
                pygame.mixer.music.stop()
                pygame.mixer.quit()
        except:
            pass
        try:
            global _vlc_instance
            if _vlc_instance is not None:
                _vlc_instance.release()
                _vlc_instance = None
        except:
            pass
        os._exit(0)

    def run(self):
        self.root.mainloop()


def type_corrected_text(original_text: str, interface=None):
    if not _text_corrector_enabled:
        if interface:
            interface.add_chat_message("Мита", T("typing_writing").format(original_text), is_mita=True)
        time.sleep(0.2)
        keyboard.write(original_text, delay=0.02)
        return original_text
    corrected = correct_text(original_text)
    if interface and corrected != original_text:
        msg = T("typing_corrected_msg").format(original_text, corrected)
        interface.add_chat_message("Мита", msg, is_mita=True)
        speak(T("corrector_on_text"), force=True)
    elif interface:
        interface.add_chat_message("Мита", T("typing_writing").format(corrected), is_mita=True)
    time.sleep(0.3)
    keyboard.write(corrected, delay=0.02)
    return corrected


# ============================================================
# ГОЛОСОВОЙ ПОТОК
# ============================================================

def voice_assistant_thread(interface, recognizer, audio_queue, sample_rate, ENERGY_THRESHOLD, SILENCE_LIMIT):
    word_buffer = []
    silence_counter = 0
    is_speaking = False

    with sd.InputStream(
            samplerate=sample_rate,
            channels=1,
            dtype="float32",
            blocksize=800,
            callback=lambda indata, frames, time, status: audio_queue.put(indata.copy()),
    ):
        while interface.running:
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
                if not interface.is_listening:
                    interface.set_listening(True)
            elif is_speaking:
                word_buffer.append(samples)
                silence_counter += 1

                if silence_counter >= SILENCE_LIMIT:
                    interface.set_listening(False)

                    if word_buffer:
                        try:
                            full_phrase_audio = np.concatenate(word_buffer)
                            stream = recognizer.create_stream()
                            stream.accept_waveform(sample_rate, full_phrase_audio)
                            recognizer.decode_stream(stream)
                            recognized_text = stream.result.text.strip()

                            if recognized_text:
                                process_command(recognized_text, interface)
                        except Exception as e:
                            print(f"[Ошибка распознавания]: {e}")

                    word_buffer = []
                    silence_counter = 0
                    is_speaking = False


# ============================================================
# СИСТЕМНЫЕ ФУНКЦИИ
# ============================================================

try:
    import soundfile as sf
    HAS_SOUNDFILE = True
except ImportError:
    HAS_SOUNDFILE = False

pyautogui.FAILSAFE = False
pyautogui.PAUSE = 0.05

BAD_WORDS = ["нахуй", "блядь", "блять", "сука", "хуй", "пизда", "пиздец",
             "ебан", "ебать", "заеб", "пидор", "гандон", "мудак", "уебок",
             "тупой", "дебил", "идиот", "кретин", "олень", "козел", "козёл",
             "лох", "лошара", "чмо", "шлюха", "курва", "бля", "нах", "хер"]


def get_available_drives() -> list[str]:
    drives = []
    for letter in string.ascii_uppercase:
        drive_path = f"{letter}:\\"
        if os.path.exists(drive_path):
            drives.append(drive_path)
    return drives


def find_mita_folder():
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
        os.path.join(BASE_DIR, "MitaAIShka"),
        os.path.join(BASE_DIR, "AI MITA", "MitaAIShka"),
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


MODEL_DIR = find_mita_folder()
model_path = os.path.join(MODEL_DIR, "model.int8.onnx")
tokens_path = os.path.join(MODEL_DIR, "tokens.txt")

if not os.path.exists(model_path) or not os.path.exists(tokens_path):
    print("❌ ОШИБКА: Файлы модели не найдены!")
    print(f"   Искал в: {MODEL_DIR}")
    model_path = os.path.join(BASE_DIR, "model.int8.onnx")
    tokens_path = os.path.join(BASE_DIR, "tokens.txt")
    if os.path.exists(model_path) and os.path.exists(tokens_path):
        MODEL_DIR = BASE_DIR
        print(f"✅ Модель найдена в: {MODEL_DIR}")
    else:
        print(f"❌ Модель не найдена! Проверьте папку с программой.")
        input("Нажмите Enter для выхода...")
        sys.exit(1)

print(f"✅ Модель загружена из: {MODEL_DIR}")

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

TRIGGER_WORDS = ["мита", "стелла", "кепочка", "стелам", "стеллу", "міта", "стела", "стелі"]

APP_VERBS = ["запусти", "запустил", "включи", "запустить", "включить", "запусти", "увімкни", "відкрий", "відкрити"]
WEB_VERBS = ["открой", "открыть", "перейди", "перейти", "покажи", "відкрий", "відкрити", "перейди", "перейти", "покажи"]
CLOSE_VERBS = ["закрой", "закрыть", "выключи", "выключить", "убей", "закрий", "закрити", "вимкни", "вимкнути"]
LANG_VERBS = ["измени", "смени", "поменяй", "переключи", "зміни", "поміняй", "переключи"]
MOVE_VERBS = ["переведи", "перемести", "перекинь", "отправь", "переведи", "перемісти", "перекинь", "відправ"]
WRITE_VERBS = ["напиши", "напечатай", "пиши", "печатай", "набор", "напишіть", "надрукуй", "пиши", "друкуй"]
MINIMIZE_VERBS = ["сверни", "свернуть", "скрой", "спрячь", "згорни", "згорнути", "скрий", "сховай"]

GREETINGS_MAP = {
    "ты тут": ("Да, я здесь! Чем помочь?\nТак, я тут! Чим допомогти?", "hello"),
    "ты здесь": ("Здесь. Слушаю вас.\nТут. Слухаю вас.", "hello"),
    "привет": ("Привет! Что нужно запустить или открыть?\nПривіт! Що потрібно запустити або відкрити?", "hello"),
    "привіт": ("Привіт! Що потрібно запустити або відкрити?\nПривет! Что нужно запустить или открыть?", "hello"),
    "как дела": ("Все отлично, готова к работе!\nВсе добре, готова до роботи!", "ok"),
    "як справи": ("Все добре, готова до роботи!\nВсе отлично, готова к работе!", "ok"),
    "не спишь": ("Я всегда на связи!\nЯ завжди на зв'язку!", "ok"),
    "не спиш": ("Я завжди на зв'язку!\nЯ всегда на связи!", "ok"),
}

APP_TRANSLIT_MAP = {
    "дискорд": "discord", "роблокс": "roblox", "стим": "steam",
    "тим": "steam", "телеграм": "telegram", "телеграмму": "telegram",
    "телеграмм": "telegram", "телега": "telegram", "браузер": "chrome",
    "хром": "chrome", "яндекс": "yandex", "спотифай": "spotify",
    "дота": "dota2", "кс": "cs2", "калькулятор": "calc", "блокнот": "notepad",
    "телеграм": "telegram", "спотіфай": "spotify", "калькулятор": "calc", "блокнот": "notepad",
}

APP_EXE_MAP = {
    "roblox": "RobloxPlayerBeta.exe", "discord": "Discord.exe",
    "telegram": "Telegram.exe", "steam": "steam.exe",
    "chrome": "chrome.exe", "yandex": "browser.exe",
    "spotify": "Spotify.exe", "dota2": "dota2.exe",
    "геншин": "launcher.exe", "obs": "obs64.exe",
    "cs2": "cs2.exe", "calc": "calc.exe", "notepad": "notepad.exe",
}

PROCESS_KILL_MAP = {
    "obs": "obs64.exe", "discord": "Discord.exe",
    "telegram": "Telegram.exe", "steam": "steam.exe",
    "chrome": "chrome.exe", "yandex": "browser.exe",
    "spotify": "Spotify.exe", "геншин": "HoYoPlay.exe",
    "dota2": "dota2.exe", "roblox": "RobloxPlayerBeta.exe",
    "cs2": "cs2.exe", "calc": "calc.exe", "notepad": "notepad.exe",
}

WEB_URLS_MAP = {
    "тик ток": "https://www.tiktok.com",
    "тикток": "https://www.tiktok.com",
    "ютуб": "https://www.youtube.com",
    "ютубчик": "https://www.youtube.com",
    "гугл": "https://www.google.com",
    "яндекс": "https://ya.ru",
    "вк": "https://vk.com",
    "вконтакте": "https://vk.com",
    "телеграм": "https://web.telegram.org",
    "гитхаб": "https://github.com",
    "покет": "https://m.pocketoption.com",
}


def minimize_all_windows():
    if not play_sound("minimize_all") and not play_sound("minimize"):
        play_sound("ok")
    pyautogui.hotkey('win', 'm')
    print("[Стелла]: Все окна свернуты")
    return True


def minimize_window(target_raw: str):
    target_clean = target_raw.lower().strip()
    app_key = APP_TRANSLIT_MAP.get(target_clean, target_clean)
    exe_name = APP_EXE_MAP.get(app_key, app_key)
    if not play_sound(f"minimize_{app_key}") and not play_sound("minimize"):
        play_sound("ok")
    win = get_window_by_exe_name(exe_name)
    if win:
        try:
            if win.isMinimized:
                return True
            win.minimize()
            return True
        except:
            return False
    return False


def change_language():
    if not play_sound("change_lang"):
        if not play_sound("language"):
            play_sound("ok")
    pyautogui.hotkey("alt", "shift")
    pyautogui.hotkey("ctrl", "shift")


def get_window_by_exe_name(exe_name: str):
    target_exe = exe_name.lower()
    if not target_exe.endswith(".exe"):
        target_exe += ".exe"
    matching_pids = set()
    for proc in psutil.process_iter(['pid', 'name']):
        try:
            if proc.info['name'] and proc.info['name'].lower() == target_exe:
                matching_pids.add(proc.info['pid'])
        except:
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
        except:
            pass
    return None


def move_window_to_monitor(target_raw: str, monitor_num: int):
    target_clean = target_raw.lower().strip()
    app_key = APP_TRANSLIT_MAP.get(target_clean, target_clean)
    exe_name = APP_EXE_MAP.get(app_key, app_key)
    win = None
    if exe_name:
        win = get_window_by_exe_name(exe_name)
    if not win:
        all_wins = gw.getAllWindows()
        for w in all_wins:
            if w.title and target_clean in w.title.lower():
                win = w
                break
    if not win:
        win = gw.getActiveWindow()
    if not win:
        return False
    PRIMARY_MONITOR_WIDTH = 1920
    new_x = 100 if monitor_num == 1 else PRIMARY_MONITOR_WIDTH + 100
    try:
        if win.isMinimized:
            win.restore()
        if win.isMaximized:
            win.restore()
        win.moveTo(new_x, 100)
        win.maximize()
        play_sound("move")
        return True
    except:
        return False


CONTROL_COMMANDS = {
    "вверх": lambda: pyautogui.scroll(400),
    "вниз": lambda: pyautogui.scroll(-400),
    "вгору": lambda: pyautogui.scroll(400),
    "вниз": lambda: pyautogui.scroll(-400),
    "клик": lambda: pyautogui.click(),
    "нажми": lambda: pyautogui.click(),
    "клік": lambda: pyautogui.click(),
    "натисни": lambda: pyautogui.click(),
    "дабл клик": lambda: pyautogui.doubleClick(),
    "двойной клик": lambda: pyautogui.doubleClick(),
    "подвійний клік": lambda: pyautogui.doubleClick(),
    "скопировать": lambda: keyboard.press_and_release('ctrl+c'),
    "скопируй": lambda: keyboard.press_and_release('ctrl+c'),
    "скопіювати": lambda: keyboard.press_and_release('ctrl+c'),
    "скопіюй": lambda: keyboard.press_and_release('ctrl+c'),
    "вставить": lambda: keyboard.press_and_release('ctrl+v'),
    "вставь": lambda: keyboard.press_and_release('ctrl+v'),
    "вставити": lambda: keyboard.press_and_release('ctrl+v'),
    "вставте": lambda: keyboard.press_and_release('ctrl+v'),
    "скрин": lambda: pyautogui.press("printscreen"),
    "скриншот": lambda: pyautogui.press("printscreen"),
    "скрін": lambda: pyautogui.press("printscreen"),
    "скріншот": lambda: pyautogui.press("printscreen"),
    "смени язык": change_language,
    "поменяй язык": change_language,
    "переключи язык": change_language,
    "зміни мову": change_language,
    "поміняй мову": change_language,
    "переключи мову": change_language,
}


def play_sound_worker(file_path: str):
    try:
        if HAS_SOUNDFILE:
            data, fs = sf.read(file_path, dtype="float32")
            sd.play(data, fs)
            sd.wait()
    except:
        pass


def play_sound(sound_name: str) -> bool:
    if not os.path.exists(SOUNDS_DIR):
        return False
    target_clean = sound_name.lower().strip()
    for file in os.listdir(SOUNDS_DIR):
        file_base, ext = os.path.splitext(file)
        if ext.lower() not in [".wav", ".mp3", ".ogg", ".flac"]:
            continue
        if file_base.lower() == target_clean or file_base.lower() == target_clean.replace(" ", "_"):
            sound_path = os.path.join(SOUNDS_DIR, file)
            threading.Thread(target=play_sound_worker, args=(sound_path,), daemon=True).start()
            return True
    return False


def load_app_cache() -> dict:
    if not os.path.exists(JSON_CACHE_FILE):
        return {}
    try:
        with open(JSON_CACHE_FILE, "r", encoding="utf-8") as f:
            return json.load(f)
    except:
        return {}


def save_app_cache(cache_data: dict):
    with open(JSON_CACHE_FILE, "w", encoding="utf-8") as f:
        json.dump(cache_data, f, ensure_ascii=False, indent=4)


def find_exe_fast_registry(exe_name: str):
    if not exe_name.endswith(".exe"):
        exe_name = f"{exe_name}.exe"
    reg_key = rf"SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\{exe_name}"
    for hive in (winreg.HKEY_CURRENT_USER, winreg.HKEY_LOCAL_MACHINE):
        try:
            with winreg.OpenKey(hive, reg_key) as key:
                path, _ = winreg.QueryValueEx(key, "")
                if os.path.exists(path):
                    return path
        except:
            continue
    return None


def search_exe_on_disks(exe_name: str):
    target_exe = exe_name.lower() if exe_name.endswith(".exe") else f"{exe_name}.exe".lower()
    drives = get_available_drives()
    priority_dirs = ["Program Files", "Program Files (x86)", "Users", "Games"]
    for drive in drives:
        for p_dir in priority_dirs:
            full_p_path = os.path.join(drive, p_dir)
            if os.path.exists(full_p_path):
                for root, _, files in os.walk(full_p_path):
                    for file in files:
                        if file.lower() == target_exe:
                            return os.path.join(root, file)
        for root, _, files in os.walk(drive):
            if any(skip in root.lower() for skip in ["$recycle.bin", "windows\\winsxs", "windows\\servicing"]):
                continue
            for file in files:
                if file.lower() == target_exe:
                    return os.path.join(root, file)
    return None


def launch_application(target_raw: str):
    target_clean = target_raw.lower().strip()
    app_key = APP_TRANSLIT_MAP.get(target_clean, target_clean)
    exe_name = APP_EXE_MAP.get(app_key, app_key)
    if not play_sound(target_clean) and not play_sound(app_key):
        play_sound("ok")
    cache = load_app_cache()
    if app_key in cache and os.path.exists(cache[app_key]):
        os.startfile(cache[app_key])
        return True
    found_path = find_exe_fast_registry(exe_name)
    if not found_path:
        found_path = search_exe_on_disks(exe_name)
    if found_path and os.path.exists(found_path):
        cache[app_key] = found_path
        save_app_cache(cache)
        os.startfile(found_path)
        return True
    try:
        subprocess.Popen(exe_name, shell=True, stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
        return True
    except:
        pass
    play_sound("error")
    return False


def kill_application(target_raw: str):
    target_clean = target_raw.lower().strip()
    app_key = APP_TRANSLIT_MAP.get(target_clean, target_clean)
    exe_name = PROCESS_KILL_MAP.get(app_key, app_key)
    if not exe_name.endswith(".exe"):
        exe_name += ".exe"
    if not play_sound(f"close_{app_key}") and not play_sound("close"):
        play_sound("ok")
    cmd = f'taskkill /F /IM "{exe_name}"'
    result = subprocess.run(cmd, shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True)
    return result.returncode == 0


def open_website(target_raw: str):
    target_clean = target_raw.lower().strip()
    if not play_sound(target_clean):
        play_sound("ok")
    if target_clean in WEB_URLS_MAP:
        webbrowser.open(WEB_URLS_MAP[target_clean])
        return True
    if "." in target_clean and not target_clean.endswith(".exe"):
        url = f"https://{target_clean}" if not target_clean.startswith("http") else target_clean
        webbrowser.open(url)
        return True
    webbrowser.open(f"https://www.google.com/search?q={target_clean}")
    return True


def process_command(text: str, interface):
    cleaned = text.lower().strip()
    words = cleaned.split()

    if not any(tw in words for tw in TRIGGER_WORDS):
        return

    print(f"\n[Распознано]: {text}")

    interface.set_processing(True)
    interface.show_command(text[:40])

    if process_music_command(cleaned, interface):
        interface.set_processing(False)
        return

    filtered_words = [w for w in words if w not in TRIGGER_WORDS]
    filtered_words = [w for w in filtered_words if w not in BAD_WORDS]

    if not filtered_words:
        print("[Стелла]: Слушаю вас!")
        play_sound("da")
        interface.set_processing(False)
        return

    phrase = " ".join(filtered_words).strip()
    ui_lang = UI_LANGUAGE

    stop_tts_phrases_ru = ["хватит", "стоп", "прекрати", "замолчи", "перестань", "остановись", "тихо", "заткнись"]
    stop_tts_phrases_ua = ["досить", "припини", "замовкни", "перестань", "зупинись", "тихо"]
    stop_tts_phrases = stop_tts_phrases_ru + stop_tts_phrases_ua

    if any(p in phrase for p in stop_tts_phrases):
        stop_tts()
        if not interface.tts_muted:
            interface.toggle_tts_mute()
        msg = T("shutting_up")
        interface.add_chat_message("Мита", msg, is_mita=True)
        play_sound("ok")
        interface.set_processing(False)
        return

    if "понял" in phrase or "поняла" in phrase or "зрозумів" in phrase or "зрозуміла" in phrase:
        stop_tts()
        play_sound("ok")
        interface.set_processing(False)
        return

    write_verbs_ru = ['напиши', 'напечатай', 'пиши', 'печатай']
    write_verbs_ua = ['напишіть', 'надрукуй', 'пиши', 'друкуй']
    write_verbs = write_verbs_ru + write_verbs_ua

    for verb in write_verbs:
        if verb in phrase:
            clean_phrase = phrase.replace(verb, "").strip()
            for tw in TRIGGER_WORDS:
                clean_phrase = clean_phrase.replace(tw, "").strip()
            if clean_phrase:
                if not play_sound("write"):
                    play_sound("ok")
                if _text_corrector_enabled:
                    corrected = correct_text(clean_phrase)
                    if corrected != clean_phrase:
                        msg = T("typing_corrected_msg").format(clean_phrase, corrected)
                        interface.add_chat_message("Мита", msg, is_mita=True)
                        speak(T("corrector_on_text"), force=True)
                        clean_phrase = corrected
                    else:
                        interface.add_chat_message("Мита", T("typing_writing").format(clean_phrase), is_mita=True)
                else:
                    interface.add_chat_message("Мита", T("typing_writing").format(clean_phrase), is_mita=True)
                time.sleep(0.3)
                keyboard.write(clean_phrase, delay=0.02)
                msg = T("typing_corrected") if _text_corrector_enabled else T("typing")
                interface.add_chat_message("Мита", msg, is_mita=True)
                interface.set_processing(False)
                return

    system_response = False
    if _mita_mode in [MODE_SYSTEM, MODE_ALL]:
        system_response = process_system_command(phrase, interface)
        if system_response and _mita_mode == MODE_SYSTEM:
            interface.set_processing(False)
            return

    if _mita_mode in [MODE_AI, MODE_ALL]:
        if _mita_mode == MODE_AI or not system_response:
            print("[Стелла]: Обращаюсь к ИИ...")
            try:
                response = ask_groq(phrase)
                print(f"[Стелла]: {response}")
                interface.add_chat_message("Мита", response, is_mita=True)
                speak(response)
            except Exception as e:
                print(f"[Ошибка Groq]: {e}")
                interface.add_chat_message("Мита", T("error_text").format(e), is_mita=True)
    else:
        if not system_response:
            response = T("command_not_found")
            interface.add_chat_message("Мита", response, is_mita=True)
            speak(response)

    interface.set_processing(False)


def process_system_command(phrase: str, interface):
    cleaned = phrase.lower().strip()
    words = cleaned.split()
    ui_lang = UI_LANGUAGE

    if "все окна" in cleaned or "сверни все" in cleaned or "усі вікна" in cleaned or "згорни всі" in cleaned:
        minimize_all_windows()
        speak(T("minimized_all"), force=True)
        return True

    move_verbs_ru = ["переведи", "перемести", "перекинь", "отправь"]
    move_verbs_ua = ["переведи", "перемісти", "перекинь", "відправ"]
    move_verbs = move_verbs_ru + move_verbs_ua

    for verb in move_verbs:
        if verb in cleaned:
            if "1" in cleaned or "первый" in cleaned or "один" in cleaned or "перший" in cleaned:
                monitor_num = 1
            elif "2" in cleaned or "второй" in cleaned or "два" in cleaned or "другий" in cleaned:
                monitor_num = 2
            else:
                speak(T("move_to_monitor_ask"), force=True)
                return True

            target = cleaned.replace(verb, "").strip()
            skip_words = move_verbs + ["на", "в", "экран", "монитор", "1", "2", "первый", "второй", "один", "два",
                                       "на", "в", "екран", "монітор", "перший", "другий"]
            for w in skip_words:
                target = target.replace(w, "").strip()

            if target:
                if move_window_to_monitor(target, monitor_num):
                    speak(T("moving_window").format(monitor_num), force=True)
                else:
                    speak(T("move_failed"), force=True)
            else:
                win = gw.getActiveWindow()
                if win:
                    try:
                        PRIMARY_MONITOR_WIDTH = 1920
                        new_x = 100 if monitor_num == 1 else PRIMARY_MONITOR_WIDTH + 100
                        if win.isMinimized:
                            win.restore()
                        if win.isMaximized:
                            win.restore()
                        win.moveTo(new_x, 100)
                        win.maximize()
                        speak(T("window_on_monitor").format(monitor_num), force=True)
                    except:
                        speak(T("window_move_failed"), force=True)
                else:
                    speak(T("no_active_window"), force=True)
            return True

    app_verbs_ru = ["запусти", "запустил", "включи", "запустить", "включить"]
    app_verbs_ua = ["запусти", "увімкни", "відкрий", "відкрити"]
    app_verbs = app_verbs_ru + app_verbs_ua

    for verb in app_verbs:
        if verb in cleaned:
            target = cleaned.replace(verb, "").strip()
            if target:
                if launch_application(target):
                    speak(T("app_launching").format(target), force=True)
                else:
                    speak(T("app_not_found").format(target), force=True)
            return True

    web_verbs_ru = ["открой", "открыть", "перейди", "перейти", "покажи"]
    web_verbs_ua = ["відкрий", "відкрити", "перейди", "перейти", "покажи"]
    web_verbs = web_verbs_ru + web_verbs_ua

    for verb in web_verbs:
        if verb in cleaned:
            target = cleaned.replace(verb, "").strip()
            if target:
                open_website(target)
                speak(T("web_opening").format(target), force=True)
            return True

    close_verbs_ru = ["закрой", "закрыть", "выключи", "выключить", "убей"]
    close_verbs_ua = ["закрий", "закрити", "вимкни", "вимкнути"]
    close_verbs = close_verbs_ru + close_verbs_ua

    for verb in close_verbs:
        if verb in cleaned:
            target = cleaned.replace(verb, "").strip()
            if target:
                if kill_application(target):
                    speak(T("app_closing").format(target), force=True)
                else:
                    speak(T("app_close_failed").format(target), force=True)
            return True

    lang_verbs_ru = ["измени", "смени", "поменяй", "переключи"]
    lang_verbs_ua = ["зміни", "поміняй", "переключи"]
    lang_verbs = lang_verbs_ru + lang_verbs_ua

    for verb in lang_verbs:
        if verb in cleaned:
            if "язык" in cleaned or "раскладку" in cleaned or "мову" in cleaned or "розкладку" in cleaned:
                change_language()
                speak(T("language_changed"), force=True)
            return True

    if cleaned in CONTROL_COMMANDS:
        CONTROL_COMMANDS[cleaned]()
        return True

    greetings_map = {
        "ты тут": T("im_here"),
        "ты здесь": T("im_here"),
        "привет": T("hello_response_ru") if UI_LANGUAGE == "ru" else T("hello_response_ua"),
        "привіт": T("hello_response_ru") if UI_LANGUAGE == "ru" else T("hello_response_ua"),
        "как дела": T("how_are_you"),
        "як справи": T("how_are_you"),
        "не спишь": T("im_always_here"),
        "не спиш": T("im_always_here"),
    }

    for greeting in greetings_map:
        if greeting in cleaned:
            speak(greetings_map[greeting], force=True)
            return True

    return False


# ============================================================
# ГЛАВНАЯ ФУНКЦИЯ
# ============================================================

def main():
    global _mita_mode, interface

    _load_ui_language()
    print(f"🌍 Язык интерфейса: {UI_LANGUAGE}")

    music_dir = os.path.join(BASE_DIR, "music")
    if not os.path.exists(music_dir):
        try:
            os.makedirs(music_dir)
            print(f"📁 Создана папка для музыки: {music_dir}")
        except:
            pass

    if not os.path.exists(SOUNDS_DIR):
        os.makedirs(SOUNDS_DIR)
        print(f"📁 Создана папка для звуков: {SOUNDS_DIR}")

    # Проверяем cookies для YouTube
    if not ensure_cookies():
        print("⚠️ Cookies не установлены! YouTube музыка может не работать.")
        print("   Чтобы исправить, положите cookies.txt в папку YouCookie")
        print("   или воспользуйтесь локальной музыкой из папки music/")

    if not require_stella_key():
        sys.exit(0)

    saved_mode = get_saved_mode()
    if saved_mode:
        set_mita_mode(saved_mode)
        print(f"✅ Режим загружен из настроек: {get_mode_name(_mita_mode)}")
    else:
        print("🔧 Показываю выбор режима...")
        mode_selector = ModeSelectionWindow()
        selected = mode_selector.show()
        set_mita_mode(selected)
        print(f"✅ Выбран режим: {get_mode_name(_mita_mode)}")

    load_app_cache()

    interface = MisideInterface()
    globals()['interface'] = interface

    voice_thread = threading.Thread(
        target=voice_assistant_thread,
        args=(interface, recognizer, audio_queue, sample_rate, ENERGY_THRESHOLD, SILENCE_LIMIT),
        daemon=True
    )
    voice_thread.start()

    try:
        interface.run()
    except KeyboardInterrupt:
        interface.quit()


if __name__ == "__main__":
    main()
