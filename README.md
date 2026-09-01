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

# Пути для поиска cookies
COOKIES_DIR = os.path.join(BASE_DIR, "YouCookie")
COOKIE_FILE = os.path.join(COOKIES_DIR, "cookie.txt")
COOKIES_FILE = os.path.join(COOKIES_DIR, "cookies.txt")

# Функция поиска cookies
def find_cookie_file():
    """Ищет файл cookies в папке YouCookie"""
    if os.path.exists(COOKIE_FILE) and os.path.getsize(COOKIE_FILE) > 0:
        return COOKIE_FILE
    elif os.path.exists(COOKIES_FILE) and os.path.getsize(COOKIES_FILE) > 0:
        return COOKIES_FILE
    return None

# Функция проверки наличия cookies
def has_cookies():
    return find_cookie_file() is not None

def get_cookie_path():
    return find_cookie_file()

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
        "no_cookies": {"ru": "❌ Нет cookies для YouTube. Положите cookie.txt в папку YouCookie", "ua": "❌ Немає cookies для YouTube. Покладіть cookie.txt в папку YouCookie"},
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
        if interface:
            interface.add_chat_message("Мита", "❌ Нет yt-dlp или VLC", is_mita=True)
        return False

    # Проверяем наличие cookies
    cookie_path = get_cookie_path()
    if not cookie_path:
        msg = T("no_cookies")
        if interface:
            interface.add_chat_message("Мита", msg, is_mita=True)
            speak("Положите файл cookie.txt в папку YouCookie", force=True)
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
        print(f"[Музыка] Использую cookies: {cookie_path}")

        ydl_opts = {
            'format': 'bestaudio[ext=m4a]/bestaudio/best',  # Указываем конкретный формат
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
            'cookiefile': cookie_path,
            'extractor_args': {
                'youtube': {
                    'player_client': ['android', 'web'],
                    'player_skip': ['webpage'],
                    'skip_dash': False,
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
                if "Sign in to confirm" in str(e) or "cookies" in str(e).lower():
                    if interface:
                        msg = "⚠️ Нужно обновить cookies для YouTube. Положите свежий cookie.txt в папку YouCookie"
                        interface.add_chat_message("Мита", msg, is_mita=True)
                        speak(msg, force=True)
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

GROQ_API_KEY =  "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"
client = Groq(api_key=GROQ_API_KEY) if GROQ_API_KEY else None

def ask_groq(question):
    if client is None:
        return "⚠️ Groq API ключ не настроен. Добавьте переменную окружения GROQ_API_KEY."
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
    if client is None:
        return "⚠️ Groq API ключ не настроен. Добавьте переменную окружения GROQ_API_KEY."
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


class CyberButton(tk.Canvas):
    """Canvas-based modern beveled button with hover/glow and Tk-like config()."""
    def __init__(self, master, text="", command=None, accent=False, danger=False,
                 width=None, height=38, **kwargs):
        self.command = command
        self.text = str(text)
        self.state = tk.NORMAL
        self.accent = accent
        self.danger = danger
        self._hover = False
        self._custom_bg = None
        self._custom_fg = None
        self._base_bg = "#081923"
        self._panel = "#06151e"
        self._line = "#0d4e66"
        self._cyan = "#19cfff"
        self._cyan2 = "#7be9ff"
        self._green = "#00f59a"
        self._amber = "#ffb52e"
        reqw = 150 if width is None else max(90, int(width)*9)
        super().__init__(master, width=reqw, height=height, bg=master.cget("bg"),
                         highlightthickness=0, bd=0, relief="flat", cursor="hand2")
        self._h = height
        self.bind("<Configure>", lambda e: self._draw())
        self.bind("<Enter>", self._enter)
        self.bind("<Leave>", self._leave)
        self.bind("<Button-1>", self._click)
        self.bind("<ButtonRelease-1>", lambda e: self._draw())
        self._draw()

    def _palette(self):
        if self.state == tk.DISABLED:
            return "#061017", "#0a2a37", "#42616d", "#0b2632"
        if self._custom_bg:
            bg = self._custom_bg
            border = self._cyan if bg not in ("#ffb020", "#ffb52e") else self._amber
            fg = self._custom_fg or "#e9fbff"
            glow = "#103b4b"
            return bg, border, fg, glow
        if self.danger:
            return ("#2b1b08" if not self._hover else "#3b260b"), self._amber, "#ffd37d", "#3b2a10"
        if self.accent:
            return ("#053244" if not self._hover else "#07455d"), self._cyan, "#dff9ff", "#0a3d50"
        return (self._base_bg if not self._hover else "#0b2633"), (self._line if not self._hover else self._cyan), (self._custom_fg or "#bce9f6"), "#0a2936"

    def _draw(self):
        self.delete("all")
        w=max(20,self.winfo_width()); h=max(24,self.winfo_height())
        bg,border,fg,glow=self._palette()
        cut=8
        pts=(cut,1,w-cut,1,w-1,cut,w-1,h-cut,w-cut,h-1,cut,h-1,1,h-cut,1,cut)
        if self._hover and self.state != tk.DISABLED:
            self.create_polygon(pts, fill=glow, outline="", smooth=False)
            inset=2
            pts2=(cut+inset,inset,w-cut-inset,inset,w-inset,cut+inset,w-inset,h-cut-inset,
                  w-cut-inset,h-inset,cut+inset,h-inset,inset,h-cut-inset,inset,cut+inset)
            self.create_polygon(pts2, fill=bg, outline=border, width=1)
        else:
            self.create_polygon(pts, fill=bg, outline=border, width=1)
        self.create_line(cut+3,3,min(w-20,cut+55),3,fill=border,width=2)
        self.create_line(w-14,h-4,w-5,h-4,fill=border,width=1)
        self.create_text(w/2,h/2+1,text=self.text,fill=fg,font=("Consolas",9,"bold"))

    def _enter(self, e=None):
        if self.state != tk.DISABLED:
            self._hover=True; self._draw()
    def _leave(self, e=None):
        self._hover=False; self._draw()
    def _click(self, e=None):
        if self.state != tk.DISABLED and self.command:
            self.command()

    def config(self, cnf=None, **kwargs):
        if cnf and isinstance(cnf, dict): kwargs.update(cnf)
        if "text" in kwargs: self.text=str(kwargs.pop("text"))
        if "state" in kwargs: self.state=kwargs.pop("state")
        if "bg" in kwargs: self._custom_bg=kwargs.pop("bg")
        if "background" in kwargs: self._custom_bg=kwargs.pop("background")
        if "fg" in kwargs: self._custom_fg=kwargs.pop("fg")
        if "foreground" in kwargs: self._custom_fg=kwargs.pop("foreground")
        if "command" in kwargs: self.command=kwargs.pop("command")
        try:
            if kwargs: super().config(**kwargs)
        except tk.TclError:
            pass
        self._draw()
    configure = config


class ModeSelectionWindow:
    def __init__(self, parent=None):
        self.result = MODE_ALL
        self.parent = parent
        self.root = tk.Tk() if parent is None else tk.Toplevel(parent)
        self.root.title("MITA // SECURE MODE SELECTOR")
        self.root.geometry("620x560")
        self.root.resizable(False, False)
        self.root.configure(bg="#020a10")

        if parent is not None:
            self.root.transient(parent)
            self.root.grab_set()

        self.canvas = tk.Canvas(self.root, bg="#020a10", highlightthickness=0)
        self.canvas.pack(fill="both", expand=True)

        for y in range(520):
            t = y / 520
            r = int(9 + 8 * t)
            g = int(7 + 3 * t)
            b = int(13 + 14 * t)
            self.canvas.create_line(0, y, 520, y, fill=f"#{r:02x}{g:02x}{b:02x}")

        self.canvas.create_text(260, 40, text="MITA // SECURE INTELLIGENCE", font=("Arial", 22, "bold"), fill="#4bdcff")
        self.canvas.create_text(260, 75, text="SELECT OPERATION PROTOCOL", font=("Arial", 10, "bold"), fill="#6e9aad")

        modes = [
            (MODE_SYSTEM, "🔧 Только система", "Выполняет системные команды\n(запуск, открытие, управление окнами)"),
            (MODE_AI, "🤖 Только ИИ", "Отвечает только через ИИ Groq\n(как чат-бот)"),
            (MODE_ALL, "🌟 Все вместе", "И системные команды, и ИИ ответы\n(рекомендуемый режим)"),
        ]

        self.selected_mode = tk.StringVar(value=MODE_ALL)

        y_pos = 120
        for mode, label, desc in modes:
            bg_id = self.canvas.create_rectangle(50, y_pos - 8, 470, y_pos + 75,
                                                 fill="#06151e", outline="#0b4054", width=1)

            rb = tk.Radiobutton(
                self.root,
                text=label,
                variable=self.selected_mode,
                value=mode,
                font=("Arial", 13, "bold"),
                bg="#06151e",
                fg="#c9e9f5",
                selectcolor="#0b4054",
                activebackground="#06151e",
                activeforeground="#4bdcff",
                relief="flat",
                bd=0,
                cursor="hand2"
            )
            rb.place(x=70, y=y_pos, width=400, height=30)

            desc_text = self.canvas.create_text(260, y_pos + 45, text=desc,
                                                font=("Arial", 9), fill="#6e9aad")

            def make_handler(mode_val):
                return lambda e: self.selected_mode.set(mode_val)

            self.canvas.tag_bind(bg_id, "<Button-1>", make_handler(mode))

            y_pos += 95

        self.button = CyberButton(self.root, text="APPLY PROTOCOL / BOOT", command=self.confirm, accent=True, height=44)
        self.button.place(x=145, y=452, width=330, height=44)

        self.dont_show_var = tk.IntVar(value=0)
        self.dont_show_cb = tk.Checkbutton(
            self.root,
            text="Не показывать при запуске",
            variable=self.dont_show_var,
            font=("Arial", 9),
            bg="#020a10",
            fg="#6e9aad",
            selectcolor="#020a10",
            activebackground="#020a10",
            activeforeground="#4bdcff",
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
        self.root.title("MITA // SECURE ACCESS")
        self.root.geometry("620x430")
        self.root.resizable(False, False)
        self.root.configure(bg="#020a10")

        if parent is not None:
            self.root.transient(parent)
            self.root.grab_set()

        self.canvas = tk.Canvas(self.root, bg="#020a10", highlightthickness=0)
        self.canvas.pack(fill="both", expand=True)

        for y in range(400):
            t = y / 400
            r = int(9 + 8 * t)
            g = int(7 + 3 * t)
            b = int(13 + 14 * t)
            self.canvas.create_line(0, y, 520, y, fill=f"#{r:02x}{g:02x}{b:02x}")

        self.canvas.create_text(260, 50, text="MITA // ACCESS NODE", font=("Arial", 24, "bold"), fill="#4bdcff")
        self.canvas.create_text(260, 145, text="AUTHENTICATION GATEWAY", font=("Arial", 9, "bold"), fill="#6e9aad")
        self.canvas.create_text(260, 175, text="ENTER ACCESS KEY", font=("Arial", 11, "bold"), fill="#c9e9f5")

        self.entry = tk.Entry(self.root, font=("Arial", 13, "bold"), justify="center",
                              bg="#06151e", fg="#c9e9f5", insertbackground="#4bdcff", relief="flat", bd=0)
        self.entry.place(x=70, y=200, width=380, height=42)

        self.status = self.canvas.create_text(260, 265, text="", font=("Arial", 9, "bold"), fill="#6e9aad")

        self.button = CyberButton(self.root, text="ESTABLISH SECURE SESSION", command=self.try_login, accent=True, height=42)
        self.button.place(x=170, y=312, width=280, height=42)

        self.entry.focus_set()
        self.root.bind("<Return>", lambda e: self.try_login())
        self.root.protocol("WM_DELETE_WINDOW", self.cancel)

    def try_login(self):
        ok, msg = validate_stella_key(self.entry.get())
        if ok:
            self.canvas.itemconfig(self.status, text=msg, fill="#4bdcff")
            self.result = True
            self.root.after(450, self.root.destroy)
        else:
            self.canvas.itemconfig(self.status, text=msg, fill="#ffb020")
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
    """Modern MITA / MISIDE inspired interface.
    Logic and voice/music back-end remain compatible with the original script.
    """

    W, H = 1480, 900

    def __init__(self):
        self.root = tk.Tk()
        self.root.title("MITA // SECURE INTELLIGENCE CONSOLE")
        self.root.geometry(f"{self.W}x{self.H}")
        self.root.minsize(1180, 740)
        self.root.configure(bg="#020a10")
        self.root.resizable(True, True)

        self.running = True
        self.is_listening = False
        self.is_processing = False
        self.is_manual_recording = False
        self.manual_audio_buffer = []
        self.manual_recording_thread = None
        self.chat_history = []
        self.message_tags = []
        self.current_tab = 0

        self.heart_beat_strength = 0.0
        self.heart_beat_target = 0.0
        self.heart_bpm = 72
        self.beat_phase = 0.0
        self.angle = 0.0
        self.phase = 0.0

        self.music_mode = False
        self.music_intensity = 0.0
        self.music_particles = []

        self.tts_muted = False
        self.corrector_enabled = _text_corrector_enabled

        self.drag_x = 0
        self.drag_y = 0
        self.is_dragging = False
        self._toast_after = None

        self.colors = {
            "bg": "#020a10",
            "bg2": "#04121b",
            "panel": "#06151e",
            "panel2": "#081c27",
            "panel3": "#0a2532",
            "line": "#0b4054",
            "line2": "#12627d",
            "primary": "#00bfff",
            "primary2": "#4bdcff",
            "purple": "#00e5ff",
            "cyan": "#00bfff",
            "green": "#00ff9c",
            "danger": "#ffb020",
            "text": "#c9e9f5",
            "muted": "#6e9aad",
            "dim": "#315c6d",
            "chat_bg": "#020c12",
            "user": "#082532",
            "mita": "#061a23",
        }

        self.root.protocol("WM_DELETE_WINDOW", self.quit)
        self.setup_ui()
        self.root.after(50, self.fade_in)

        self.audio_monitor = SystemAudioMonitor(self)
        self.audio_monitor.start()
        self.animate()

    # -------------------- visual helpers --------------------

    def rounded_rect(self, canvas, x1, y1, x2, y2, r=18, fill=None,
                     outline=None, width=1, tags=None):
        r = min(r, abs(x2-x1)/2, abs(y2-y1)/2)
        pts = [
            x1+r,y1, x2-r,y1, x2,y1+r, x2,y2-r,
            x2-r,y2, x1+r,y2, x1,y2-r, x1,y1+r
        ]
        return canvas.create_polygon(
            pts, smooth=True, splinesteps=16,
            fill=fill or "", outline=outline or "", width=width, tags=tags
        )

    def _label(self, parent, text, size=10, weight="normal", color=None, **kw):
        return tk.Label(
            parent, text=text, font=("Consolas", size, weight),
            bg=parent.cget("bg"), fg=color or self.colors["text"],
            **kw
        )

    def _button(self, parent, text, command, accent=False, danger=False, width=None):
        return CyberButton(parent, text=text, command=command, accent=accent,
                           danger=danger, width=width, height=38)

    def _card(self, parent, title=None, subtitle=None):
        outer = tk.Frame(parent, bg=self.colors["line"], bd=0)
        frame = tk.Frame(outer, bg=self.colors["panel"], bd=0)
        frame.pack(fill="both", expand=True, padx=1, pady=1)
        # cyan micro-rail on top makes every panel feel like a live module
        rail = tk.Frame(frame, bg=self.colors["line2"], height=2)
        rail.pack(fill="x")
        if title:
            head = tk.Frame(frame, bg=self.colors["panel"], height=38)
            head.pack(fill="x", padx=14, pady=(8, 0))
            head.pack_propagate(False)
            tk.Label(head, text=f"// {title}", font=("Consolas", 9, "bold"),
                     bg=self.colors["panel"], fg=self.colors["text"]).pack(side="left", pady=8)
            if subtitle:
                tk.Label(head, text=subtitle, font=("Consolas", 7, "bold"),
                         bg=self.colors["panel"], fg=self.colors["green"]).pack(side="right", pady=8)
        return outer

    def _bind_hover(self, widget, normal, hover):
        widget.bind("<Enter>", lambda e: widget.config(bg=hover))
        widget.bind("<Leave>", lambda e: widget.config(bg=normal))

    def _make_nav(self, parent, key, icon, idx):
        row = tk.Frame(parent, bg=self.colors["panel"], height=46, cursor="hand2")
        row.pack(fill="x", padx=10, pady=2)
        row.pack_propagate(False)
        marker = tk.Frame(row, bg=self.colors["panel"], width=3)
        marker.pack(side="left", fill="y")
        icon_lbl = tk.Label(row, text=icon, font=("Consolas", 13, "bold"),
                            bg=self.colors["panel"], fg=self.colors["muted"], width=3)
        icon_lbl.pack(side="left", padx=(7, 2))
        text_lbl = tk.Label(row, text=key.upper(), font=("Consolas", 8, "bold"),
                            bg=self.colors["panel"], fg=self.colors["muted"], anchor="w")
        text_lbl.pack(side="left", fill="x", expand=True)
        code = tk.Label(row, text=f"0{idx+1}", font=("Consolas", 7, "bold"),
                        bg=self.colors["panel"], fg=self.colors["dim"])
        code.pack(side="right", padx=10)
        for w in (row, icon_lbl, text_lbl, marker, code):
            w.bind("<Button-1>", lambda e, i=idx: self.switch_tab(i))
            w.bind("<Enter>", lambda e, r=row: r.config(bg=self.colors["panel2"]))
        self.nav_items.append((row, icon_lbl, text_lbl))
        return row

    def setup_ui(self):
        self.canvas = tk.Canvas(self.root, bg=self.colors["bg"], highlightthickness=0, bd=0)
        self.canvas.pack(fill="both", expand=True)
        self.create_background()

        # ultra-thin intelligence ticker
        self.intel_wire = tk.Frame(self.root, bg="#03131d",
                                   highlightbackground=self.colors["line"], highlightthickness=1)
        self.intel_wire.place(x=14, y=8, relwidth=0.981, height=28)
        tk.Label(self.intel_wire, text="INTEL WIRE", font=("Consolas",8,"bold"),
                 bg="#03131d", fg=self.colors["primary2"]).pack(side="left", padx=(12,8))
        tk.Label(self.intel_wire, text="●", font=("Consolas",7,"bold"),
                 bg="#03131d", fg=self.colors["danger"]).pack(side="left")
        tk.Label(self.intel_wire, text="  SECURE VOICE CHANNEL / AI CORE / SYSTEM CONTROL / ENCRYPTED SESSION",
                 font=("Consolas",7,"bold"), bg="#03131d", fg=self.colors["muted"]).pack(side="left")
        self.wire_hash = tk.Label(self.intel_wire, text="HASH 1441D6FD4D7F", font=("Consolas",7,"bold"),
                                  bg="#03131d", fg=self.colors["green"])
        self.wire_hash.pack(side="right", padx=12)

        # header status matrix
        self.header = tk.Frame(self.root, bg=self.colors["panel"],
                               highlightbackground=self.colors["line"], highlightthickness=1)
        self.header.place(x=14, y=42, relwidth=0.981, height=70)

        brand = tk.Frame(self.header, bg=self.colors["panel"], width=330)
        brand.pack(side="left", fill="y", padx=(14,4)); brand.pack_propagate(False)
        tk.Label(brand, text="Ω", font=("Consolas",23,"bold"), bg=self.colors["panel"],
                 fg=self.colors["primary2"]).pack(side="left", padx=(0,10))
        bt=tk.Frame(brand,bg=self.colors["panel"]); bt.pack(side="left", pady=13)
        tk.Label(bt,text="MITA // SECURE CORE",font=("Consolas",14,"bold"),bg=self.colors["panel"],
                 fg=self.colors["text"]).pack(anchor="w")
        tk.Label(bt,text="OPERATOR INTELLIGENCE CONSOLE",font=("Consolas",7,"bold"),bg=self.colors["panel"],
                 fg=self.colors["muted"]).pack(anchor="w")

        def header_metric(title, value, color):
            f=tk.Frame(self.header,bg=self.colors["panel"],highlightbackground="#0a2d3b",highlightthickness=1)
            f.pack(side="left",fill="both",expand=True,padx=3,pady=9)
            tk.Label(f,text=title,font=("Consolas",6,"bold"),bg=self.colors["panel"],fg=self.colors["dim"]).pack(anchor="w",padx=10,pady=(7,0))
            tk.Label(f,text=value,font=("Consolas",8,"bold"),bg=self.colors["panel"],fg=color).pack(anchor="e",padx=10,pady=(0,5))
            return f
        header_metric("SYSTEM STATE","● OPERATIONAL",self.colors["green"])
        header_metric("CIPHER ROTATION","00:40",self.colors["primary2"])
        header_metric("RELAY HEALTH","99.995%",self.colors["text"])

        self.header_clock = tk.Label(self.header,text="",font=("Consolas",8,"bold"),
                                     bg=self.colors["panel"],fg=self.colors["primary2"],padx=14)
        self.header_clock.pack(side="right",fill="y")

        # three-column body: navigation / content / telemetry
        self.body = tk.Frame(self.root, bg=self.colors["bg"])
        self.body.place(x=14, y=120, relwidth=0.981, relheight=0.845)

        self.sidebar = tk.Frame(self.body,bg=self.colors["panel"],
                                highlightbackground=self.colors["line"],highlightthickness=1,width=238)
        self.sidebar.pack(side="left",fill="y",padx=(0,8)); self.sidebar.pack_propagate(False)

        self.content = tk.Frame(self.body,bg=self.colors["bg2"],
                                highlightbackground=self.colors["line"],highlightthickness=1)
        self.content.pack(side="left",fill="both",expand=True,padx=(0,8))

        self.rightbar = tk.Frame(self.body,bg=self.colors["panel"],
                                 highlightbackground=self.colors["line"],highlightthickness=1,width=238)
        self.rightbar.pack(side="right",fill="y"); self.rightbar.pack_propagate(False)

        self.build_sidebar()
        self.build_content()
        self.build_rightbar()
        self.create_particles(); self.create_stars(); self.create_floating_stars()
        self.update_key_info(); self.switch_tab(0)

    def build_sidebar(self):
        top=tk.Frame(self.sidebar,bg=self.colors["panel"]); top.pack(fill="x",padx=14,pady=(16,8))
        tk.Label(top,text="OPERATOR PANEL",font=("Consolas",7,"bold"),bg=self.colors["panel"],fg=self.colors["dim"]).pack(anchor="w")
        tk.Label(top,text="CONTROL NODE",font=("Consolas",13,"bold"),bg=self.colors["panel"],fg=self.colors["text"]).pack(anchor="w",pady=(3,0))
        tk.Label(top,text="AUTHENTICATED // OMEGA",font=("Consolas",6,"bold"),bg=self.colors["panel"],fg=self.colors["green"]).pack(anchor="w",pady=(2,0))

        self.nav_items=[]
        self._make_nav(self.sidebar,T("nav_main"),"⌂",0)
        self._make_nav(self.sidebar,T("nav_chat"),"◉",1)
        self._make_nav(self.sidebar,T("nav_commands"),"⌘",2)
        self._make_nav(self.sidebar,T("nav_settings"),"⚙",3)

        tk.Frame(self.sidebar,bg=self.colors["line"],height=1).pack(fill="x",padx=12,pady=12)

        # network status module
        net=self._card(self.sidebar,"NETWORK STATUS","LIVE")
        net.pack(fill="x",padx=10,pady=4)
        nf=tk.Frame(net.winfo_children()[0],bg=self.colors["panel"]); nf.pack(fill="x",padx=12,pady=(2,10))
        tk.Label(nf,text="9/9 NODES ONLINE",font=("Consolas",9,"bold"),bg=self.colors["panel"],fg=self.colors["green"]).pack(anchor="w")
        tk.Label(nf,text="VOICE RELAY  •  AI CORE",font=("Consolas",6,"bold"),bg=self.colors["panel"],fg=self.colors["muted"]).pack(anchor="w",pady=(2,0))

        mode=self._card(self.sidebar,"PROCESSING MODE","MANUAL")
        mode.pack(fill="x",padx=10,pady=5)
        inner=mode.winfo_children()[0]
        self.mode_label=tk.Label(inner,text=get_mode_name(_mita_mode),font=("Consolas",8,"bold"),
                                 bg=self.colors["panel"],fg=self.colors["primary2"])
        self.mode_label.pack(anchor="w",padx=12,pady=(3,5))
        self.mode_button=self._button(inner,T("change_mode"),self.change_mode)
        self.mode_button.pack(fill="x",padx=10,pady=(0,10))

        voice=self._card(self.sidebar,"VOICE ENGINE","READY")
        voice.pack(fill="x",padx=10,pady=5); vi=voice.winfo_children()[0]
        self.engine_status=tk.Label(vi,text="● MITA CORE ONLINE",font=("Consolas",7,"bold"),bg=self.colors["panel"],fg=self.colors["green"])
        self.engine_status.pack(anchor="w",padx=12,pady=(3,5))
        self.manual_record_button=self._button(vi,"[ MIC ]  "+T("manual_input_btn"),self.toggle_manual_record,accent=True)
        self.manual_record_button.pack(fill="x",padx=10,pady=3)
        self.tts_mute_button=self._button(vi,"[ VOICE ]  "+T("voice_on"),self.toggle_tts_mute)
        self.tts_mute_button.pack(fill="x",padx=10,pady=3)
        self.corrector_button=self._button(vi,"[ TEXT ]  "+T("corrector_off"),self.toggle_corrector)
        self.corrector_button.pack(fill="x",padx=10,pady=(3,10))

        access=self._card(self.sidebar,"ACCESS TOKEN","SECURE")
        access.pack(fill="x",padx=10,pady=5); ai=access.winfo_children()[0]
        self.key_info_label=tk.Label(ai,text="",font=("Consolas",7,"bold"),bg=self.colors["panel"],fg=self.colors["text"],justify="left")
        self.key_info_label.pack(anchor="w",padx=12,pady=(3,4))
        self.change_key_btn=self._button(ai,T("change_key"),self.change_key)
        self.change_key_btn.pack(fill="x",padx=10,pady=(0,10))

    def build_rightbar(self):
        top=tk.Frame(self.rightbar,bg=self.colors["panel"]); top.pack(fill="x",padx=13,pady=(15,8))
        tk.Label(top,text="TRACE DEFENSE",font=("Consolas",7,"bold"),bg=self.colors["panel"],fg=self.colors["dim"]).pack(anchor="w")
        tk.Label(top,text="SYNCHRONIZATION",font=("Consolas",11,"bold"),bg=self.colors["panel"],fg=self.colors["text"]).pack(anchor="w",pady=(3,0))

        shield=self._card(self.rightbar,"DEFENSE MATRIX","ACTIVE")
        shield.pack(fill="x",padx=10,pady=5); si=shield.winfo_children()[0]
        self.shield_canvas=tk.Canvas(si,bg=self.colors["panel"],highlightthickness=0,height=150)
        self.shield_canvas.pack(fill="x",padx=8,pady=4)
        c=self.shield_canvas; cx,cy=105,70
        for r,col in [(52,"#0b3242"),(41,"#0d4b60"),(29,"#0e667e")]: c.create_oval(cx-r,cy-r,cx+r,cy+r,outline=col,width=1)
        c.create_text(cx,cy,text="Ω",font=("Consolas",19,"bold"),fill=self.colors["green"])
        c.create_text(cx,132,text="IDENTITY MASK  100%",font=("Consolas",6,"bold"),fill=self.colors["primary2"])

        sync=self._card(self.rightbar,"LIVE SESSION","OMEGA ONLY")
        sync.pack(fill="x",padx=10,pady=5); sy=sync.winfo_children()[0]
        self.sync_canvas=tk.Canvas(sy,bg=self.colors["panel"],highlightthickness=0,height=140)
        self.sync_canvas.pack(fill="x",padx=8,pady=5)
        cc=self.sync_canvas; cc.create_oval(47,15,157,125,outline="#0b4054",width=10)
        cc.create_arc(47,15,157,125,start=90,extent=-250,style="arc",outline=self.colors["primary"],width=6)
        cc.create_text(102,62,text="99%",font=("Consolas",19,"bold"),fill=self.colors["primary"])
        cc.create_text(102,88,text="LINK STABLE",font=("Consolas",7,"bold"),fill=self.colors["muted"])

        log=self._card(self.rightbar,"ENCRYPTED EVENT LOG","LIVE")
        log.pack(fill="both",expand=True,padx=10,pady=5); li=log.winfo_children()[0]
        self.event_log=tk.Text(li,bg="#020c12",fg=self.colors["muted"],font=("Consolas",7),bd=0,relief="flat",height=10,padx=8,pady=8)
        self.event_log.pack(fill="both",expand=True,padx=8,pady=(4,8))
        self.event_log.insert("end","22:54:58  CORE BOOT OK\n22:54:59  VOICE RELAY READY\n22:55:00  AI CHANNEL SEALED\n22:55:01  TRACE SHIELD ACTIVE\n22:55:02  WAITING REQUEST...\n")
        self.event_log.config(state="disabled")

    def build_content(self):
        self.pages = {}
        self.build_home_page()
        self.build_chat_page()
        self.build_commands_page()
        self.build_settings_page()

    def _page(self):
        f = tk.Frame(self.content, bg=self.colors["bg2"])
        return f

    def build_home_page(self):
        page=self._page(); self.pages[0]=page
        # operation notice strip
        notice=tk.Frame(page,bg="#07141b",highlightbackground="#7b5616",highlightthickness=1)
        notice.pack(fill="x",padx=14,pady=(14,8))
        tk.Label(notice,text="OPERATOR NOTICE",font=("Consolas",7,"bold"),bg="#07141b",fg=self.colors["danger"],padx=10,pady=8).pack(side="left")
        tk.Label(notice,text="Secure voice dispatch active. Commands are relayed individually through MITA CORE.",font=("Consolas",7),bg="#07141b",fg=self.colors["muted"]).pack(side="left")
        tk.Label(notice,text="PROTOCOL 7.4",font=("Consolas",6,"bold"),bg="#07141b",fg="#a87820",padx=10).pack(side="right")

        grid=tk.Frame(page,bg=self.colors["bg2"]); grid.pack(fill="both",expand=True,padx=14,pady=(0,8))
        grid.columnconfigure(0,weight=4); grid.columnconfigure(1,weight=2)
        grid.rowconfigure(0,weight=3); grid.rowconfigure(1,weight=2)

        core=self._card(grid,"MITA NETWORK CORE","STREAMING")
        core.grid(row=0,column=0,rowspan=2,sticky="nsew",padx=(0,6),pady=4)
        ci=core.winfo_children()[0]
        self.core_canvas=tk.Canvas(ci,bg="#020d14",highlightthickness=0)
        self.core_canvas.pack(fill="both",expand=True,padx=8,pady=(3,8)); self.core_canvas.bind("<Configure>",self._resize_core)

        status=self._card(grid,"SYSTEM TELEMETRY","REAL TIME")
        status.grid(row=0,column=1,sticky="nsew",padx=(6,0),pady=4); si=status.winfo_children()[0]
        sf=tk.Frame(si,bg=self.colors["panel"]); sf.pack(fill="both",expand=True,padx=12,pady=(3,10))
        self.status_text=tk.Label(sf,text=T("ready"),font=("Consolas",11,"bold"),bg=self.colors["panel"],fg=self.colors["primary2"])
        self.status_text.pack(anchor="w")
        self.status_sub=tk.Label(sf,text=T("waiting"),font=("Consolas",7),bg=self.colors["panel"],fg=self.colors["muted"])
        self.status_sub.pack(anchor="w",pady=(2,12))
        # segmented telemetry bars
        for label,val,col in [("RELAY HEALTH",99,self.colors["green"]),("ROUTE ENTROPY",97,self.colors["primary"]),("VOICE LINK",92,self.colors["primary2"])]:
            tk.Label(sf,text=f"{label}  {val}%",font=("Consolas",6,"bold"),bg=self.colors["panel"],fg=self.colors["muted"]).pack(anchor="w",pady=(3,1))
            bar=tk.Frame(sf,bg="#0a2733",height=4); bar.pack(fill="x"); fill=tk.Frame(bar,bg=col,height=4); fill.place(x=0,y=0,relwidth=val/100,relheight=1)
        self.ram_value=tk.Label(sf,text="— MB",font=("Consolas",16,"bold"),bg=self.colors["panel"],fg=self.colors["text"])
        self.ram_value.pack(anchor="w",pady=(14,0))
        tk.Label(sf,text="RAM / PROCESS MEMORY",font=("Consolas",6,"bold"),bg=self.colors["panel"],fg=self.colors["dim"]).pack(anchor="w")
        self.audio_value=tk.Label(sf,text="● QUIET",font=("Consolas",8,"bold"),bg=self.colors["panel"],fg=self.colors["muted"])
        self.audio_value.pack(anchor="w",pady=(10,0))

        quick=self._card(grid,"MANUAL DISPATCH","CONTROL")
        quick.grid(row=1,column=1,sticky="nsew",padx=(6,0),pady=4); qi=quick.winfo_children()[0]
        q=tk.Frame(qi,bg=self.colors["panel"]); q.pack(fill="both",expand=True,padx=10,pady=(2,10))
        self._button(q,"[ MIC ]  REQUEST SIGNAL",self.toggle_manual_record,accent=True).pack(fill="x",pady=3)
        self._button(q,"[ CHAT ]  OPEN CHANNEL",lambda:self.switch_tab(1)).pack(fill="x",pady=3)
        self._button(q,"[ STOP ]  MUSIC RELAY",self.stop_music_click,danger=True).pack(fill="x",pady=3)
        self._button(q,"[ CFG ]  SYSTEM SETTINGS",lambda:self.switch_tab(3)).pack(fill="x",pady=3)

        music=self._card(page,"NOW PLAYING / AUDIO RELAY","MUSIC CORE")
        music.pack(fill="x",padx=14,pady=(0,14)); mi=music.winfo_children()[0]
        mf=tk.Frame(mi,bg=self.colors["panel"]); mf.pack(fill="x",padx=12,pady=(2,10))
        eq=tk.Canvas(mf,width=66,height=38,bg=self.colors["panel"],highlightthickness=0); eq.pack(side="left",padx=(0,10))
        for i,h in enumerate([9,18,28,15,24,11,20]): eq.create_rectangle(4+i*8,32-h,8+i*8,32,fill=self.colors["primary"],outline="")
        mt=tk.Frame(mf,bg=self.colors["panel"]); mt.pack(side="left",fill="x",expand=True)
        self.music_title=tk.Label(mt,text="NO ACTIVE AUDIO STREAM",font=("Consolas",9,"bold"),bg=self.colors["panel"],fg=self.colors["text"]); self.music_title.pack(anchor="w")
        self.music_meta=tk.Label(mt,text="VOICE QUERY: «Мита, включи песню…»",font=("Consolas",6,"bold"),bg=self.colors["panel"],fg=self.colors["muted"]); self.music_meta.pack(anchor="w",pady=(2,0))
        self.music_stop_button=self._button(mf,"TERMINATE STREAM",self.stop_music_click,danger=True); self.music_stop_button.pack(side="right"); self.music_stop_button.config(state=tk.DISABLED)

    def build_chat_page(self):
        page=self._page(); self.pages[1]=page
        head=tk.Frame(page,bg=self.colors["bg2"]); head.pack(fill="x",padx=16,pady=(15,8))
        tk.Label(head,text="ENCRYPTED AI CHANNEL",font=("Consolas",14,"bold"),bg=self.colors["bg2"],fg=self.colors["text"]).pack(side="left")
        self.chat_counter=tk.Label(head,text="0 SIGNALS",font=("Consolas",7,"bold"),bg=self.colors["bg2"],fg=self.colors["green"]); self.chat_counter.pack(side="right",pady=6)

        card=self._card(page,"CLASSIFIED SIGNAL ARCHIVE","MANUAL RELAY"); card.pack(fill="both",expand=True,padx=16,pady=(0,14)); ci=card.winfo_children()[0]
        self.chat_display=scrolledtext.ScrolledText(ci,bg="#020b11",fg=self.colors["text"],font=("Consolas",9),wrap=tk.WORD,bd=0,relief="flat",insertbackground=self.colors["primary2"],padx=18,pady=14,selectbackground="#0b4f65")
        self.chat_display.pack(fill="both",expand=True,padx=8,pady=(3,8)); self.chat_display.config(state=tk.DISABLED)
        self.chat_display.tag_config("mita_name",foreground=self.colors["green"],font=("Consolas",8,"bold"))
        self.chat_display.tag_config("mita_text",foreground="#b9dce7",font=("Consolas",9))
        self.chat_display.tag_config("user_name",foreground=self.colors["primary2"],font=("Consolas",8,"bold"))
        self.chat_display.tag_config("user_text",foreground=self.colors["text"],font=("Consolas",9))

        self.chat_menu=Menu(self.root,tearoff=0,bg=self.colors["panel2"],fg=self.colors["text"],activebackground="#0b4f65",activeforeground="white",bd=0)
        self.chat_menu.add_command(label="COPY SIGNAL",command=self.copy_selected_message); self.chat_menu.add_command(label="COPY ARCHIVE",command=self.copy_all_chat); self.chat_menu.add_separator(); self.chat_menu.add_command(label="PURGE ARCHIVE",command=self.clear_chat)
        self.chat_display.bind("<Button-3>",self.show_chat_menu); self.chat_display.bind("<Control-c>",lambda e:self.copy_selected_message()); self.chat_display.bind("<Control-a>",lambda e:self.select_all_chat())

        bottom=tk.Frame(ci,bg=self.colors["panel"]); bottom.pack(fill="x",padx=8,pady=(0,8))
        shell=tk.Frame(bottom,bg=self.colors["line"],bd=0); shell.pack(side="left",fill="x",expand=True,padx=(0,8))
        self.chat_input=tk.Entry(shell,bg="#03131d",fg=self.colors["text"],font=("Consolas",9),relief="flat",bd=0,insertbackground=self.colors["primary2"])
        self.chat_input.pack(fill="x",padx=1,pady=1,ipady=12); self.chat_input.bind("<Return>",lambda e:self.send_chat_message()); self.chat_input.bind("<Button-3>",self.show_input_menu); self.chat_input.bind("<Control-a>",lambda e:self.select_all_input()); self.chat_input.bind("<Control-c>",lambda e:self.copy_input_text()); self.chat_input.bind("<Control-v>",lambda e:self.paste_input_text()); self.chat_input.bind("<Control-x>",lambda e:self.cut_input_text())
        self.send_button=self._button(bottom,"TRANSMIT  ▶",self.send_chat_message,accent=True); self.send_button.pack(side="right")
        self.input_menu=Menu(self.root,tearoff=0,bg=self.colors["panel2"],fg=self.colors["text"],activebackground="#0b4f65",bd=0)
        self.input_menu.add_command(label="CUT",command=self.cut_input_text); self.input_menu.add_command(label="COPY",command=self.copy_input_text); self.input_menu.add_command(label="PASTE",command=self.paste_input_text); self.input_menu.add_separator(); self.input_menu.add_command(label="CLEAR",command=self.clear_input_text); self.input_menu.add_command(label="SELECT ALL",command=self.select_all_input)
        hello_msg=(f"{T('mita_greeting')}\n{T('mita_mode')}{get_mode_name(_mita_mode)}\n{T('mita_corrector')}{T('corrector_off_info') if not _text_corrector_enabled else T('corrector_on_info')}\n{T('mita_lang')}\n{T('mita_help')}")
        self.add_chat_message("Мита",hello_msg,is_mita=True)

    def build_commands_page(self):
        page=self._page(); self.pages[2]=page
        head=tk.Frame(page,bg=self.colors["bg2"]); head.pack(fill="x",padx=16,pady=(15,8))
        tk.Label(head,text="COMMAND PROTOCOLS",font=("Consolas",14,"bold"),bg=self.colors["bg2"],fg=self.colors["text"]).pack(anchor="w")
        tk.Label(head,text="AUTHORIZED VOICE / SYSTEM OPERATIONS",font=("Consolas",7,"bold"),bg=self.colors["bg2"],fg=self.colors["muted"]).pack(anchor="w",pady=(2,0))
        wrap=tk.Frame(page,bg=self.colors["bg2"]); wrap.pack(fill="both",expand=True,padx=16,pady=(0,14)); wrap.columnconfigure(0,weight=1); wrap.columnconfigure(1,weight=1); wrap.columnconfigure(2,weight=1)
        commands=[("01","LAUNCH","Запусти [программа]","Discord / Roblox / Steam"),("02","WEB RELAY","Открой [сайт]","YouTube / Google / GitHub"),("03","TERMINATE","Закрой [программа]","Process shutdown"),("04","TYPE","Напиши [текст]","Input to active window"),("05","MINIMIZE","Сверни все","Windows desktop control"),("06","CAPTURE","Скриншот","Screen snapshot"),("07","CLIPBOARD","Скопируй / Вставь","Ctrl+C / Ctrl+V"),("08","DISPLAY ROUTE","Переведи на 1/2 монитор","Move active window"),("09","AUDIO QUERY","Песня [название]","Local / YouTube search"),("10","AUDIO GAIN","Громкость [0-200%]","Music volume"),("11","AUDIO STOP","Стоп музыка","Terminate stream"),("12","LANGUAGE","Смени язык","Keyboard language")]
        for i,(code,title,cmd,desc) in enumerate(commands):
            c=self._card(wrap,title,"PROTO "+code); r,col=divmod(i,3); c.grid(row=r,column=col,sticky="nsew",padx=4,pady=4); inner=c.winfo_children()[0]
            box=tk.Frame(inner,bg=self.colors["panel"]); box.pack(fill="both",expand=True,padx=11,pady=(2,10))
            tk.Label(box,text=cmd,font=("Consolas",8,"bold"),bg=self.colors["panel"],fg=self.colors["primary2"]).pack(anchor="w")
            tk.Label(box,text=desc,font=("Consolas",6),bg=self.colors["panel"],fg=self.colors["muted"]).pack(anchor="w",pady=(3,0))

    def build_settings_page(self):
        page=self._page(); self.pages[3]=page
        head=tk.Frame(page,bg=self.colors["bg2"]); head.pack(fill="x",padx=16,pady=(15,8))
        tk.Label(head,text="SYSTEM CONFIGURATION",font=("Consolas",14,"bold"),bg=self.colors["bg2"],fg=self.colors["text"]).pack(anchor="w")
        tk.Label(head,text="OPERATOR-LEVEL PREFERENCES / SECURE LOCAL STATE",font=("Consolas",7,"bold"),bg=self.colors["bg2"],fg=self.colors["muted"]).pack(anchor="w",pady=(2,0))
        scroll=tk.Frame(page,bg=self.colors["bg2"]); scroll.pack(fill="both",expand=True,padx=16,pady=(0,14)); scroll.columnconfigure(0,weight=1); scroll.columnconfigure(1,weight=1)

        lang=self._card(scroll,"LANGUAGE MATRIX","INTERFACE"); lang.grid(row=0,column=0,sticky="nsew",padx=(0,5),pady=5); li=lang.winfo_children()[0]
        lf=tk.Frame(li,bg=self.colors["panel"]); lf.pack(fill="x",padx=12,pady=(3,10))
        tk.Label(lf,text="Active locale / response language",font=("Consolas",7),bg=self.colors["panel"],fg=self.colors["muted"]).pack(anchor="w",pady=(0,7))
        self.lang_button=self._button(lf,"SWITCH → "+("УКРАЇНСЬКА" if UI_LANGUAGE=="ru" else "РУССКИЙ"),self.toggle_language); self.lang_button.pack(fill="x")

        mode=self._card(scroll,"AI PROCESSING MODE","CORE"); mode.grid(row=0,column=1,sticky="nsew",padx=(5,0),pady=5); mi=mode.winfo_children()[0]
        self.mode_setting_label=tk.Label(mi,text=get_mode_name(_mita_mode),font=("Consolas",9,"bold"),bg=self.colors["panel"],fg=self.colors["primary2"]); self.mode_setting_label.pack(anchor="w",padx=12,pady=(3,7))
        self._button(mi,"CHANGE PROCESSING PROTOCOL",self.change_mode).pack(fill="x",padx=10,pady=(0,10))

        music=self._card(scroll,"AUDIO GAIN CONTROL","0-200%"); music.grid(row=1,column=0,columnspan=2,sticky="nsew",pady=5); mui=music.winfo_children()[0]
        mv=tk.Frame(mui,bg=self.colors["panel"]); mv.pack(fill="x",padx=12,pady=(3,10))
        self.volume_label=tk.Label(mv,text=f"AUDIO GAIN: {_music_volume}%",font=("Consolas",8,"bold"),bg=self.colors["panel"],fg=self.colors["text"]); self.volume_label.pack(anchor="w")
        self.volume_scale=tk.Scale(mv,from_=0,to=200,orient=tk.HORIZONTAL,bg=self.colors["panel"],fg=self.colors["primary2"],activebackground=self.colors["primary"],troughcolor="#092b39",highlightthickness=0,bd=0,showvalue=False,sliderrelief="flat",sliderlength=16,command=self.on_volume_change); self.volume_scale.pack(fill="x",pady=(6,0)); self.volume_scale.set(_music_volume)

        access=self._card(scroll,"ACCESS CREDENTIAL","SECURITY"); access.grid(row=2,column=0,columnspan=2,sticky="nsew",pady=5); ac=access.winfo_children()[0]
        ar=tk.Frame(ac,bg=self.colors["panel"]); ar.pack(fill="x",padx=12,pady=(3,10))
        self.settings_key_label=tk.Label(ar,text="",font=("Consolas",8,"bold"),bg=self.colors["panel"],fg=self.colors["text"]); self.settings_key_label.pack(side="left")
        self._button(ar,"ROTATE ACCESS KEY",self.change_key).pack(side="right")

    def switch_tab(self, idx):
        self.current_tab = idx
        for i, (row, icon, label) in enumerate(self.nav_items):
            active = i == idx
            bg = "#0a2b39" if active else self.colors["panel"]
            fg = self.colors["primary2"] if active else self.colors["muted"]
            row.config(bg=bg)
            icon.config(bg=bg, fg=self.colors["primary2"] if active else self.colors["muted"])
            label.config(bg=bg, fg=fg)
        for p in self.pages.values():
            p.pack_forget()
        self.pages[idx].pack(fill="both", expand=True)

    # -------------------- core animation --------------------

    def _resize_core(self, event=None):
        if not hasattr(self, "core_canvas"):
            return
        w = max(300, self.core_canvas.winfo_width())
        h = max(260, self.core_canvas.winfo_height())
        self.cx, self.cy = w/2, h/2 + 5
        if not hasattr(self, "core_ready"):
            self._build_core_art(w, h)
            self.core_ready = True

    def _build_core_art(self, w, h):
        c = self.core_canvas
        self.aura = []
        for r, col in [(150,"#062332"),(125,"#07364a"),(102,"#084b64"),(83,"#096782")]:
            self.aura.append(c.create_oval(self.cx-r,self.cy-r,self.cx+r,self.cy+r,
                                           outline=col, width=1))
        self.core_outer = c.create_oval(self.cx-66,self.cy-66,self.cx+66,self.cy+66,
                                        fill="#03131d", outline=self.colors["primary"], width=2)
        self.core_inner = c.create_oval(self.cx-51,self.cy-51,self.cx+51,self.cy+51,
                                        fill="#062534", outline=self.colors["purple"], width=1)
        self.heart = self.create_heart(self.cx, self.cy, 40, self.colors["primary"])
        self.heart_glow = c.create_oval(self.cx-60,self.cy-60,self.cx+60,self.cy+60,
                                        outline=self.colors["primary"], width=2)
        self.orbit_elements = []
        for i in range(12):
            obj = c.create_text(self.cx,self.cy,text="✦",
                                font=("Consolas", 8+(i%3),"bold"),
                                fill=self.colors["purple"])
            self.orbit_elements.append({
                "id":obj, "angle":i*math.pi*2/12,
                "radius":112+(i%2)*20, "speed":0.006+i*0.0009
            })
        self.bars = []
        start = self.cx - 115
        for i in range(24):
            x = start+i*10
            o = c.create_rectangle(x,self.cy+95,x+5,self.cy+97,
                                   fill=self.colors["purple"], outline="")
            self.bars.append({"id":o,"x":x,"height":2,"phase":random.random()*6.28})

    def create_heart(self, x, y, size, color):
        points=[]
        for t in range(0,360,5):
            r=math.radians(t)
            hx=16*math.sin(r)**3
            hy=13*math.cos(r)-5*math.cos(2*r)-2*math.cos(3*r)-math.cos(4*r)
            points += [x+hx*size/16, y-hy*size/16]
        hid=self.core_canvas.create_polygon(points,fill=color,outline=color,width=2)
        gid=self.core_canvas.create_polygon(points,fill="",outline=self.colors["primary2"],width=2)
        return {"id":hid,"glow":gid}

    def update_heart(self, size, color, glow_intensity=1.0):
        if not hasattr(self, "heart"):
            return
        points=[]
        scale=1+(math.sin(self.beat_phase)*self.heart_beat_strength*0.06)
        for t in range(0,360,5):
            r=math.radians(t)
            hx=16*math.sin(r)**3
            hy=13*math.cos(r)-5*math.cos(2*r)-2*math.cos(3*r)-math.cos(4*r)
            points += [self.cx+hx*size/16*scale, self.cy-hy*size/16*scale]
        self.core_canvas.coords(self.heart["id"],*points)
        self.core_canvas.coords(self.heart["glow"],*points)
        self.core_canvas.itemconfig(self.heart["id"],fill=color,outline=color)
        self.core_canvas.itemconfig(self.heart["glow"],outline=self.colors["primary2"])

    def update_heart_beat(self, strength):
        self.heart_beat_target = strength
        if strength > .1:
            self.heart_bpm = (120+int(strength*80)) if self.music_mode else (72+int(strength*48))

    def animate_heart_beat(self):
        self.heart_beat_strength += (self.heart_beat_target-self.heart_beat_strength)*.15
        self.beat_phase += .025*(self.heart_bpm/60)*2
        if self.music_mode:
            self.music_intensity=min(1,self.music_intensity+.01)
            self.heart_beat_strength=max(self.heart_beat_strength,.28+self.music_intensity*.5)
        else:
            self.music_intensity=max(0,self.music_intensity-.02)

        if self.heart_beat_strength>.1:
            p=min(1,self.heart_beat_strength*1.5)
            r=int(255-(255-167)*(1-p)); g=int(92-92*(1-p)); b=int(170-170*(1-p))
            color=f"#{r:02x}{g:02x}{b:02x}"
        else:
            color=self.colors["primary"]
        if hasattr(self,"audio_value"):
            if self.heart_beat_strength>.15:
                self.audio_value.config(text=f"● AUDIO • {self.heart_bpm} BPM",fg=self.colors["primary2"])
            elif self.heart_beat_strength>.05:
                self.audio_value.config(text="● AUDIO • DETECTED",fg=self.colors["purple"])
            else:
                self.audio_value.config(text="● QUIET",fg=self.colors["muted"])
        return .5+self.heart_beat_strength*.5,color

    def animate(self):
        if not self.running:
            return
        self.music_mode=_is_music_mode
        self.music_intensity=_music_intensity
        self.angle += .035
        self.phase += .12
        _, heart_color=self.animate_heart_beat()

        if self.music_mode:
            status, sub, color = f"♫  МУЗЫКА ИГРАЕТ  •  {_music_volume}%", "Наслаждайся ритмом", self.colors["primary2"]
            self.music_title.config(text=_current_track_title or "Музыка")
            self.music_meta.config(text=f"Громкость {_music_volume}%  •  MUSIC CORE ACTIVE")
            self.music_stop_button.config(state=tk.NORMAL)
        elif self.is_listening:
            status, sub, color=T("listening"),T("listening_sub"),self.colors["primary"]
        elif self.is_processing:
            status, sub, color=T("processing"),T("processing_sub"),self.colors["purple"]
        elif self.is_manual_recording:
            status, sub, color=T("manual_recording"),T("manual_sub"),self.colors["primary2"]
        elif self.tts_muted:
            status, sub, color="🔇  VOICE MUTED",T("voice_off_status"),self.colors["danger"]
        else:
            status, sub, color=f"✦  {get_mode_name(_mita_mode)}",T("waiting"),self.colors["primary2"]

        if hasattr(self,"status_text"):
            self.status_text.config(text=status,fg=color)
            self.status_sub.config(text=sub)

        if hasattr(self,"core_canvas") and hasattr(self,"heart"):
            pulse=40+math.sin(self.angle*2)*3
            if self.heart_beat_strength>.1:
                pulse=40*(1+math.sin(self.beat_phase)*self.heart_beat_strength*.25)
            if self.music_mode: pulse*=1.18
            self.update_heart(pulse,heart_color)

            for i,obj in enumerate(self.aura):
                d=math.sin(self.angle*1.4+i)*(2+i)+self.heart_beat_strength*8*math.sin(self.beat_phase+i)
                if self.music_mode: d*=1.7
                base=[150,125,102,83][i]
                self.core_canvas.coords(obj,self.cx-base-d,self.cy-base-d,self.cx+base+d,self.cy+base+d)

            for item in self.orbit_elements:
                item["angle"]+=item["speed"]
                rad=item["radius"]+self.heart_beat_strength*12*math.sin(self.beat_phase+item["angle"])
                if self.music_mode: rad += 5*math.sin(self.phase+item["angle"])
                self.core_canvas.coords(item["id"],
                    self.cx+math.cos(item["angle"])*rad,
                    self.cy+math.sin(item["angle"])*rad)
                self.core_canvas.itemconfig(item["id"],fill=color)

            for i,bar in enumerate(self.bars):
                if self.music_mode:
                    target=5+abs(math.sin(self.phase*1.5+i*.55))*random.randint(15,42)
                elif self.is_listening or self.is_processing or self.is_manual_recording:
                    target=5+abs(math.sin(self.phase+i*.55))*random.randint(8,24)
                else:
                    target=3+self.heart_beat_strength*18
                bar["height"]+=(target-bar["height"])*.3
                self.core_canvas.coords(bar["id"],bar["x"],self.cy+95-bar["height"],bar["x"]+5,self.cy+95)
                self.core_canvas.itemconfig(bar["id"],fill=color)

        if hasattr(self,"ram_value"):
            self.ram_value.config(text=f"{self.get_ram_usage()} MB")
        if hasattr(self,"header_clock"):
            self.header_clock.config(text=datetime.now().strftime("%H:%M:%S  •  %d.%m.%Y"))

        self.root.after(35,self.animate)

    # -------------------- background --------------------

    def create_background(self):
        c=self.canvas; c.delete("background")
        w=max(self.W,self.root.winfo_width()); h=max(self.H,self.root.winfo_height())
        # deep navy vertical gradient
        for y in range(0,h,4):
            t=y/max(1,h); r=int(1+2*t); g=int(8+9*t); b=int(13+15*t)
            c.create_rectangle(0,y,w,y+4,fill=f"#{r:02x}{g:02x}{b:02x}",outline="",tags="background")
        # technical grid and coordinates
        for x in range(0,w+1,44): c.create_line(x,0,x,h,fill="#031b25",width=1,tags="background")
        for y in range(0,h+1,44): c.create_line(0,y,w,y,fill="#031b25",width=1,tags="background")
        for x in range(0,w+1,220): c.create_line(x,0,x,h,fill="#063246",width=1,tags="background")
        for y in range(0,h+1,220): c.create_line(0,y,w,y,fill="#063246",width=1,tags="background")
        random.seed(41)
        for _ in range(60):
            x=random.randint(5,w-5); y=random.randint(120,h-5); c.create_oval(x-1,y-1,x+1,y+1,fill="#0b5973",outline="",tags="background")
        c.tag_lower("background")

    def create_particles(self):
        self.particles=[]
        for _ in range(30):
            self.particles.append({"x":random.randint(0,self.W),"y":random.randint(80,self.H),
                                   "speed":random.uniform(.2,.7),"phase":random.random()*6.28})

    def create_stars(self):
        self.stars=[]
        for _ in range(38):
            x=random.randint(0,self.W); y=random.randint(80,self.H)
            o=self.canvas.create_oval(x,y,x+2,y+2,fill="#303342",outline="",tags="stars")
            self.stars.append({"id":o,"x":x,"y":y,"speed":random.uniform(.5,1.4)})

    def create_floating_stars(self):
        self.floating_stars=[]
        for _ in range(18):
            self.floating_stars.append({"x":random.randint(0,self.W),"y":random.randint(80,self.H),
                                        "sx":random.uniform(-.3,.3),"sy":random.uniform(-.25,.25),
                                        "phase":random.random()*6.28})

    def animate_floating_stars(self):
        now=time.time()
        for s in self.floating_stars:
            s["x"]+=s["sx"]+math.sin(now+s["phase"])*.05
            s["y"]+=s["sy"]+math.cos(now*.7+s["phase"])*.05
            if s["x"]<0 or s["x"]>self.W: s["sx"]*=-1
            if s["y"]<80 or s["y"]>self.H: s["sy"]*=-1

    # -------------------- controls --------------------

    def toggle_language(self):
        global UI_LANGUAGE
        UI_LANGUAGE="ua" if UI_LANGUAGE=="ru" else "ru"
        _save_ui_language()
        self.refresh_ui()

    def refresh_ui(self):
        self.root.title("MITA // SECURE INTELLIGENCE CONSOLE")
        self.mode_button.config(text=T("change_mode"))
        self.manual_record_button.config(text="[ MIC ]  "+T("manual_input_btn"))
        self.tts_mute_button.config(text="[ VOICE ]  "+(T("voice_on") if not self.tts_muted else T("voice_off")))
        self.corrector_button.config(text="[ TEXT ]  "+(T("corrector_on") if _text_corrector_enabled else T("corrector_off")))
        self.lang_button.config(text="SWITCH → "+("УКРАЇНСЬКА" if UI_LANGUAGE=="ru" else "РУССКИЙ"))
        self.change_key_btn.config(text=T("change_key"))
        self.music_stop_button.config(text=T("music_stop"))
        self.volume_label.config(text=f"🔊 {T('volume')}: {_music_volume}%")
        for i,(row,icon,label) in enumerate(self.nav_items):
            key=[T("nav_main"),T("nav_chat"),T("nav_commands"),T("nav_settings")][i]
            label.config(text=key)
        self.update_mode_display()
        self.update_key_info()
        self.add_chat_message("Мита",T("lang_changed"),is_mita=True)
        speak(T("lang_changed"),force=True)

    def toggle_corrector(self):
        self.corrector_enabled=not self.corrector_enabled
        set_text_corrector(self.corrector_enabled)
        self.corrector_button.config(
            text=T("corrector_on") if self.corrector_enabled else T("corrector_off"),
            bg="#06382c" if self.corrector_enabled else self.colors["panel3"],
            fg=self.colors["green"] if self.corrector_enabled else self.colors["text"]
        )
        msg=T("corrector_on_text") if self.corrector_enabled else T("corrector_off_text")
        self.add_chat_message("Мита",f"📝 {msg}",is_mita=True)
        speak(msg,force=True)

    def change_mode(self):
        dialog=tk.Toplevel(self.root)
        dialog.title(T("mode"))
        dialog.geometry("500x420")
        dialog.configure(bg=self.colors["bg"])
        dialog.resizable(False,False)
        dialog.transient(self.root); dialog.grab_set()

        tk.Label(dialog,text="AI MODE",font=("Consolas",9,"bold"),
                 bg=self.colors["bg"],fg=self.colors["dim"]).pack(pady=(24,3))
        tk.Label(dialog,text="Как MITA должна обрабатывать команды?",
                 font=("Consolas",16,"bold"),bg=self.colors["bg"],fg=self.colors["text"]).pack(pady=(0,18))

        var=tk.StringVar(value=_mita_mode)
        modes=[
            (MODE_SYSTEM,"🔧 Только система","Запуск приложений, окна, ввод, скриншоты."),
            (MODE_AI,"🤖 Только ИИ","Все запросы отправляются в Groq."),
            (MODE_ALL,"✦ Все вместе","Сначала системные команды, затем ИИ.")
        ]
        for mode,label,desc in modes:
            card=tk.Frame(dialog,bg=self.colors["panel"],highlightbackground=self.colors["line"],highlightthickness=1)
            card.pack(fill="x",padx=30,pady=5)
            rb=tk.Radiobutton(card,text=label,variable=var,value=mode,
                              font=("Consolas",10,"bold"),bg=self.colors["panel"],
                              fg=self.colors["text"],selectcolor=self.colors["panel3"],
                              activebackground=self.colors["panel"],activeforeground=self.colors["primary2"],
                              relief="flat",bd=0)
            rb.pack(anchor="w",padx=12,pady=(9,2))
            tk.Label(card,text=desc,font=("Consolas",8),bg=self.colors["panel"],
                     fg=self.colors["muted"]).pack(anchor="w",padx=34,pady=(0,9))

        def apply():
            mode=var.get()
            set_mita_mode(mode)
            self.update_mode_display()
            dialog.destroy()
            msg=T("mode_changed").format(get_mode_name(mode))
            self.add_chat_message("Мита",msg,is_mita=True)
            speak(T("mode_changed_text").format(get_mode_name(mode)),force=True)

        self._button(dialog,T("apply"),apply,accent=True).pack(fill="x",padx=120,pady=18)

    def update_mode_display(self):
        if hasattr(self,"mode_label"):
            self.mode_label.config(text=get_mode_name(_mita_mode))
        if hasattr(self,"mode_setting_label"):
            self.mode_setting_label.config(text=get_mode_name(_mita_mode))

    def toggle_tts_mute(self):
        global TTS_VOLUME
        self.tts_muted=not self.tts_muted
        if self.tts_muted:
            stop_tts(); TTS_VOLUME=0.0
            self.tts_mute_button.config(text=T("voice_off"),bg=self.colors["danger"],fg="white")
            self.add_chat_message("Мита",T("voice_off"),is_mita=True)
        else:
            TTS_VOLUME=1.0
            self.tts_mute_button.config(text=T("voice_on"),bg=self.colors["panel3"],fg=self.colors["green"])
            self.add_chat_message("Мита",T("voice_on"),is_mita=True)
            speak(T("voice_on_text"),force=True)

    def toggle_manual_record(self):
        if self.is_manual_recording: self.stop_manual_recording()
        else: self.start_manual_recording()

    def start_manual_recording(self):
        if self.is_manual_recording: return
        self.is_manual_recording=True; self.manual_audio_buffer=[]
        self.manual_record_button.config(text=T("stop_record"),bg=self.colors["danger"])
        self.add_chat_message("Мита","🎤 "+T("manual_sub"),is_mita=True)
        self.manual_recording_thread=threading.Thread(target=self._manual_record_loop,daemon=True)
        self.manual_recording_thread.start()

    def stop_manual_recording(self):
        if not self.is_manual_recording: return
        self.is_manual_recording=False
        self.manual_record_button.config(text=T("manual_input_btn"),bg=self.colors["primary"])
        if self.manual_audio_buffer:
            threading.Thread(target=self._process_manual_audio,daemon=True).start()
        else:
            self.add_chat_message("Мита",T("no_audio"),is_mita=True)
            self._reset_manual_state()

    def _manual_record_loop(self):
        try:
            with sd.InputStream(samplerate=16000,channels=1,dtype="float32",
                                blocksize=1024,callback=self._manual_audio_callback):
                while self.is_manual_recording: time.sleep(.05)
        except Exception as e:
            self.root.after(0,lambda:self.add_chat_message("Мита",T("error_text").format(e),is_mita=True))
            self.root.after(0,self._reset_manual_state)

    def _manual_audio_callback(self,indata,frames,time,status):
        if self.is_manual_recording: self.manual_audio_buffer.append(indata.copy())

    def _process_manual_audio(self):
        try:
            if not self.manual_audio_buffer:
                self.root.after(0,lambda:self.add_chat_message("Мита",T("no_data"),is_mita=True))
                self.root.after(0,self._reset_manual_state); return
            audio=np.concatenate(self.manual_audio_buffer); self.manual_audio_buffer=[]
            stream=recognizer.create_stream(); stream.accept_waveform(16000,audio); recognizer.decode_stream(stream)
            recognized_text=stream.result.text.strip()
            if recognized_text:
                self.root.after(0,lambda:self.add_chat_message("Вы","🎤 "+recognized_text))
                self.root.after(0,lambda:self._process_manual_command(recognized_text))
            else:
                self.root.after(0,lambda:self.add_chat_message("Мита",T("recognition_failed"),is_mita=True))
                self.root.after(0,self._reset_manual_state)
        except Exception as e:
            self.root.after(0,lambda:self.add_chat_message("Мита",T("error_text").format(e),is_mita=True))
            self.root.after(0,self._reset_manual_state)

    def _process_manual_command(self,text):
        self.is_processing=True
        try:
            result=None
            if _mita_mode in [MODE_SYSTEM,MODE_ALL]:
                result=self.process_chat_system_command(text)
                if result:
                    self.add_chat_message("Мита",result,is_mita=True); speak(result); self._reset_manual_state(); return
            if _mita_mode in [MODE_AI,MODE_ALL]:
                response=ask_groq_chat(text,self.chat_history)
                self.add_chat_message("Мита",response,is_mita=True)
                self.chat_history.append({"role":"assistant","content":response}); speak(response)
            elif not result:
                self.add_chat_message("Мита",T("command_not_found"),is_mita=True); speak(T("command_not_found"))
        except Exception as e:
            self.add_chat_message("Мита",T("error_text").format(e),is_mita=True)
        self._reset_manual_state()

    def _reset_manual_state(self):
        self.is_processing=False; self.is_manual_recording=False; self.manual_audio_buffer=[]
        self.manual_record_button.config(text=T("manual_input_btn"),bg=self.colors["primary"],fg="white")

    # -------------------- music --------------------

    def update_music_button(self,is_playing):
        if hasattr(self,"music_stop_button"):
            self.music_stop_button.config(
                text=T("music_stop_click") if is_playing else T("music_stop"),
                bg=self.colors["danger"] if is_playing else self.colors["panel3"],
                fg="white" if is_playing else self.colors["text"],
                state=tk.NORMAL if is_playing else tk.DISABLED
            )

    def stop_music_click(self):
        stop_music()
        self.add_chat_message("Мита",T("music_stopped_click"),is_mita=True)
        self.update_music_button(False)
        speak(T("music_stopped_voice"),force=True)

    def on_volume_change(self,value):
        set_music_volume(int(float(value))); self.update_volume_display()

    def update_volume_display(self):
        if hasattr(self,"volume_scale"): self.volume_scale.set(_music_volume)
        if hasattr(self,"volume_label"): self.volume_label.config(text=f"🔊 {T('volume')}: {_music_volume}%")
        if self.music_mode: self.status_text.config(text=f"♫  МУЗЫКА ИГРАЕТ  •  {_music_volume}%")

    # -------------------- chat helpers --------------------

    def show_input_menu(self,event):
        try:self.input_menu.post(event.x_root,event.y_root)
        except:pass
    def cut_input_text(self):
        try:self.chat_input.event_generate("<<Cut>>")
        except:pass
    def copy_input_text(self):
        try:self.chat_input.event_generate("<<Copy>>"); self.show_temp_status(T("copy_success"))
        except:pass
    def paste_input_text(self):
        try:self.chat_input.event_generate("<<Paste>>"); self.show_temp_status(T("pasted"))
        except:pass
    def clear_input_text(self):
        self.chat_input.delete(0,tk.END)
    def select_all_input(self):
        self.chat_input.select_range(0,tk.END); self.chat_input.focus_set(); return "break"
    def show_temp_status(self,text):
        if hasattr(self,"status_sub"):
            self.status_sub.config(text=text)
            if self._toast_after:
                try:self.root.after_cancel(self._toast_after)
                except:pass
            self._toast_after=self.root.after(1500,lambda:self.status_sub.config(text=T("waiting")))

    def select_all_chat(self):
        self.chat_display.config(state=tk.NORMAL); self.chat_display.tag_add(tk.SEL,"1.0",tk.END)
        self.chat_display.config(state=tk.DISABLED); return "break"
    def show_chat_menu(self,event):
        try:self.chat_menu.post(event.x_root,event.y_root)
        except:pass

    def copy_selected_message(self):
        try:
            selected=self.chat_display.get(tk.SEL_FIRST,tk.SEL_LAST).strip()
            if not selected: raise tk.TclError()
            self.root.clipboard_clear(); self.root.clipboard_append(selected); self.root.update()
            self.show_temp_status(T("copy_success")); return
        except: pass
        self.copy_all_chat()

    def copy_all_chat(self):
        try:
            content=self.chat_display.get("1.0",tk.END).strip()
            if not content: self.show_temp_status(T("chat_empty")); return
            self.root.clipboard_clear(); self.root.clipboard_append(content); self.root.update()
            self.show_temp_status(T("copy_all_success"))
        except Exception as e:self.show_temp_status(T("error_text").format(e))

    def clear_chat(self):
        if messagebox.askyesno("Очистка чата",T("clear_confirm")):
            self.chat_display.config(state=tk.NORMAL); self.chat_display.delete("1.0",tk.END); self.chat_display.config(state=tk.DISABLED)
            self.chat_history=[]
            self.add_chat_message("Мита",T("chat_cleared"),is_mita=True)

    def add_chat_message(self,sender,message,is_mita=False):
        if not hasattr(self,"chat_display"): return
        if is_mita: message=re.sub(r"[\*_`]","",str(message)); message=re.sub(r"\s+"," ",message).strip()
        self.chat_display.config(state=tk.NORMAL)
        timestamp=datetime.now().strftime("%H:%M")
        if is_mita:
            self.chat_display.insert(tk.END,f"🌸 Мита  •  {timestamp}\n","mita_name")
            self.chat_display.insert(tk.END,f"{message}\n\n","mita_text")
        else:
            self.chat_display.insert(tk.END,f"👤 Вы  •  {timestamp}\n","user_name")
            self.chat_display.insert(tk.END,f"{message}\n\n","user_text")
        self.chat_display.see(tk.END); self.chat_display.config(state=tk.DISABLED)
        if hasattr(self,"chat_counter"):
            try:
                count=int(self.chat_display.get("1.0",tk.END).count("\n\n"))
                self.chat_counter.config(text=f"{count} сообщений")
            except:pass

    def send_chat_message(self):
        message=self.chat_input.get().strip()
        if not message:return
        self.chat_input.delete(0,tk.END); self.add_chat_message("Вы",message)
        self.chat_history.append({"role":"user","content":message})
        self.is_processing=True
        threading.Thread(target=self.process_chat_command,args=(message,),daemon=True).start()

    def process_chat_command(self,message):
        try:
            result=None
            if _mita_mode in [MODE_SYSTEM,MODE_ALL]:
                result=self.process_chat_system_command(message)
                if result and _mita_mode==MODE_SYSTEM:
                    self.root.after(0,lambda:self.add_chat_message("Мита",result,is_mita=True))
                    speak(result); self.is_processing=False; return
            if _mita_mode in [MODE_AI,MODE_ALL]:
                if _mita_mode==MODE_AI or not result:
                    response=ask_groq_chat(message,self.chat_history)
                    self.root.after(0,lambda:self.add_chat_message("Мита",response,is_mita=True))
                    self.chat_history.append({"role":"assistant","content":response}); speak(response)
            elif not result:
                self.root.after(0,lambda:self.add_chat_message("Мита",T("command_not_found"),is_mita=True))
                speak(T("command_not_found"))
        except Exception as e:
            self.root.after(0,lambda:self.add_chat_message("Мита",T("error_text").format(e),is_mita=True))
        self.is_processing=False

    def process_chat_system_command(self,text):
        cleaned=text.lower().strip()
        if process_music_command(cleaned,self): return "🎵 "+T("processing_sub")
        if "громкость" in cleaned or "гучн" in cleaned: return T("volume_text").format(_music_volume)
        stop_phrases=["хватит","стоп","прекрати","замолчи","перестань","остановись","тихо","досить","припини","замовкни","зупинись"]
        if any(p in cleaned for p in stop_phrases):
            stop_tts()
            if not self.tts_muted:self.toggle_tts_mute()
            return T("shutting_up")
        if "все окна" in cleaned or "сверни все" in cleaned or "усі вікна" in cleaned or "згорни всі" in cleaned:
            minimize_all_windows(); return T("minimized_all")
        if "скриншот" in cleaned or "скрин" in cleaned or "скрін" in cleaned:
            pyautogui.press("printscreen"); return T("screenshot_done")
        if "скопируй" in cleaned or "скопіюй" in cleaned:
            keyboard.press_and_release("ctrl+c"); return T("copied")
        if "вставь" in cleaned or "встав" in cleaned:
            keyboard.press_and_release("ctrl+v"); return T("pasted")
        write_verbs=['напиши','напечатай','пиши','печатай','напишіть','надрукуй','друкуй']
        for verb in write_verbs:
            if verb in cleaned:
                target=cleaned.split(verb,1)[1].strip()
                if target:
                    if _text_corrector_enabled: target=correct_text(target)
                    keyboard.write(target,delay=.02); return T("typing_corrected") if _text_corrector_enabled else T("typed")+target
        move_verbs=["переведи","перемести","перекинь","отправь","перемісти","відправ"]
        for verb in move_verbs:
            if verb in cleaned:
                monitor=1 if any(x in cleaned for x in ["1","первый","один","перший"]) else 2 if any(x in cleaned for x in ["2","второй","два","другий"]) else None
                if monitor is None:return T("move_to_monitor_ask")
                target=cleaned.replace(verb,"").strip()
                for w in ["на","в","экран","монитор","1","2","первый","второй","один","два","перший","другий","екран","монітор"]:
                    target=target.replace(w,"").strip()
                return T("moving_window").format(monitor) if move_window_to_monitor(target,monitor) else T("move_failed")
        app_verbs=["запусти","запустил","включи","запустить","включить","увімкни","відкрий","відкрити"]
        for verb in app_verbs:
            if verb in cleaned:
                target=cleaned.replace(verb,"").strip()
                return T("app_launching").format(target) if launch_application(target) else T("app_not_found").format(target)
        web_verbs=["открой","открыть","перейди","перейти","покажи","відкрий","відкрити"]
        for verb in web_verbs:
            if verb in cleaned:
                target=cleaned.replace(verb,"").strip(); open_website(target); return T("web_opening").format(target)
        close_verbs=["закрой","закрыть","выключи","выключить","убей","закрий","закрити","вимкни","вимкнути"]
        for verb in close_verbs:
            if verb in cleaned:
                target=cleaned.replace(verb,"").strip()
                return T("app_closing").format(target) if kill_application(target) else T("app_close_failed").format(target)
        if cleaned in CONTROL_COMMANDS:
            CONTROL_COMMANDS[cleaned](); return T("executed")
        greetings={"ты тут":T("im_here"),"ты здесь":T("im_here"),"привет":T("hello_response_ru") if UI_LANGUAGE=="ru" else T("hello_response_ua"),
                   "привіт":T("hello_response_ru") if UI_LANGUAGE=="ru" else T("hello_response_ua"),
                   "как дела":T("how_are_you"),"як справи":T("how_are_you"),
                   "не спишь":T("im_always_here"),"не спиш":T("im_always_here")}
        for g,v in greetings.items():
            if g in cleaned:return v
        return None

    # -------------------- key / voice state --------------------

    def update_key_info(self):
        saved_key=_get_saved_key()
        text=T("no_key"); color=self.colors["danger"]
        if saved_key:
            db=_load_json_file(KEY_DB_FILE); info=db.get(saved_key)
            if info:
                remaining=float(info["expires_at"])-time.time()
                if remaining>0:
                    key_display=saved_key[:8]+"..." if len(saved_key)>8 else saved_key
                    text=f"🔑 {key_display}  •  {_remaining_text(remaining)}"; color=self.colors["green"]
        if hasattr(self,"key_info_label"): self.key_info_label.config(text=text,fg=color)
        if hasattr(self,"settings_key_label"): self.settings_key_label.config(text=text,fg=color)
        self.root.after(10000,self.update_key_info)

    def change_key(self):
        if messagebox.askyesno("Смена ключа",T("key_change_confirm")):
            _clear_saved_key()
            if KeyLoginWindow(self.root).show():
                self.update_key_info()
                messagebox.showinfo("Успешно",T("key_success"))
            else:self.update_key_info()

    def set_listening(self,listening):
        self.is_listening=bool(listening)

    def set_processing(self,processing):
        self.is_processing=bool(processing)

    def show_command(self,text):
        if hasattr(self,"status_sub"):
            self.status_sub.config(text=f"⌘ {text[:60]}")

    # -------------------- lifecycle --------------------

    def get_ram_usage(self):
        try:return int(psutil.Process().memory_info().rss/1024/1024)
        except:return 0

    def fade_in(self):
        try:
            a=float(self.root.attributes("-alpha"))
            if a<1:
                self.root.attributes("-alpha",min(1,a+.08)); self.root.after(25,self.fade_in)
        except:
            try:self.root.attributes("-alpha",1.0)
            except:pass

    def quit(self):
        if not self.running:return
        self.running=False
        try:self.audio_monitor.stop()
        except:pass
        try:stop_tts()
        except:pass
        try:stop_music()
        except:pass
        try:
            if _vlc_instance is not None:_vlc_instance.release()
        except:pass
        try:self.root.destroy()
        except:pass

    def run(self):
        self.root.mainloop()


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

    # СОЗДАНИЕ ПАПКИ ДЛЯ COOKIES
    if not os.path.exists(COOKIES_DIR):
        try:
            os.makedirs(COOKIES_DIR)
            print(f"📁 Создана папка для cookies: {COOKIES_DIR}")
            print(f"📁 Положите сюда файл cookie.txt для YouTube")
        except Exception as e:
            print(f"⚠️ Не удалось создать папку для cookies: {e}")

    # Проверяем наличие cookie.txt
    if has_cookies():
        print(f"✅ Найден файл cookies: {get_cookie_path()}")
    else:
        print(f"⚠️ Файл cookie.txt не найден в папке YouCookie")
        print(f"   Чтобы музыка работала, положите cookie.txt в папку YouCookie")

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

