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
from tkinter import scrolledtext, ttk, messagebox, Menu
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

# Голоса для разных языков
TTS_VOICE_RU = "ru-RU-SvetlanaNeural"
TTS_VOICE_UA = "uk-UA-PolinaNeural"
TTS_VOICE_DEFAULT = TTS_VOICE_RU

TTS_RATE = "-5%"
TTS_PITCH = "+2Hz"
TTS_VOLUME = 1.0
TTS_FILE = os.path.join(os.path.dirname(os.path.abspath(__file__)) if "__file__" in globals() else os.getcwd(),
                        "mita_tts.mp3")
_tts_lock = threading.Lock()
_tts_pygame_ready = False
_tts_stop_requested = False

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
UI_LANG_FILE = os.path.join(os.path.dirname(os.path.abspath(__file__)) if "__file__" in globals() else os.getcwd(),
                            "mita_ui_lang.json")


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
# БЫСТРОЕ ВОСПРОИЗВЕДЕНИЕ МУЗЫКИ
# ============================================================

_music_thread = None
_music_stop = False
_current_music_file = None
_vlc_instance = None
_vlc_player = None
_current_track_title = ""
_music_search_cache = {}
_music_cache_lock = threading.Lock()

def set_music_volume(percent: int):
    global _music_volume, _vlc_player
    _music_volume = max(0, min(200, int(percent)))
    player = _vlc_player
    if player is not None:
        try:
            player.audio_set_volume(_music_volume)
        except Exception:
            pass
    return True

def get_music_volume():
    return _music_volume

def volume_up(interface=None):
    set_music_volume(get_music_volume() + 10)
    if interface:
        interface.update_volume_display()
        msg = T("volume_up").format(_music_volume)
        interface.add_chat_message("Мита", msg, is_mita=True)
        speak(msg, force=True)
    return True

def volume_down(interface=None):
    set_music_volume(get_music_volume() - 10)
    if interface:
        interface.update_volume_display()
        msg = T("volume_down").format(_music_volume)
        interface.add_chat_message("Мита", msg, is_mita=True)
        speak(msg, force=True)
    return True

def find_local_music(query: str) -> str:
    music_dir = os.path.join(BASE_DIR, "music")
    try:
        os.makedirs(music_dir, exist_ok=True)
    except Exception:
        return None

    query_lower = query.lower().strip()
    files = []
    try:
        files = os.listdir(music_dir)
    except Exception:
        return None

    audio_ext = (".mp3", ".wav", ".flac", ".ogg", ".m4a")
    for file in files:
        if file.lower().endswith(audio_ext):
            stem = os.path.splitext(file)[0].lower()
            if query_lower in stem or stem in query_lower:
                return os.path.join(music_dir, file)

    words = [w for w in re.findall(r"[\w'-]+", query_lower) if len(w) > 2]
    best = None
    best_score = 0
    for file in files:
        if not file.lower().endswith(audio_ext):
            continue
        stem = os.path.splitext(file)[0].lower()
        score = sum(1 for w in words if w in stem)
        if score > best_score:
            best_score = score
            best = os.path.join(music_dir, file)
    return best

def _clean_song_query(query: str) -> str:
    q = re.sub(r"\s+", " ", str(query or "").lower()).strip()
    # Убираем только команды/обращения, не ломая название песни.
    q = re.sub(
        r"\b(?:мита|стелла|стела|міта|стелі|кепочка|пожалуйста|будь-ласка|сейчас|зараз|"
        r"включи|включить|поставь|постав|запусти|запустить|песню|песня|пісню|пісня|"
        r"музыку|музика|музику|трек)\b",
        " ", q
    )
    return re.sub(r"\s+", " ", q).strip()

def _youtube_find_fast(query: str):
    """Минимальный сетевой путь: один результат, без playlist/retry-петель."""
    if not HAS_YDL:
        return None, None

    clean_query = _clean_song_query(query)
    if len(clean_query) < 2:
        return None, None

    cache_key = clean_query.casefold()
    with _music_cache_lock:
        cached = _music_search_cache.get(cache_key)
    if cached:
        return cached

    opts = {
        "quiet": True,
        "no_warnings": True,
        "noplaylist": True,
        "skip_download": True,
        "extract_flat": True,
        "default_search": "ytsearch1",
        "socket_timeout": 7,
        "retries": 0,
        "fragment_retries": 0,
        "cachedir": False,
    }

    try:
        with yt_dlp.YoutubeDL(opts) as ydl:
            info = ydl.extract_info(f"ytsearch1:{clean_query}", download=False)
        entries = info.get("entries") or []
        entry = next((e for e in entries if e), None)
        if not entry:
            return None, None

        video_url = entry.get("webpage_url")
        video_id = entry.get("id")
        if not video_url and video_id:
            video_url = f"https://www.youtube.com/watch?v={video_id}"

        title = entry.get("title") or clean_query
        result = (video_url, title)
        with _music_cache_lock:
            _music_search_cache[cache_key] = result
            # Ограничиваем память кэша.
            if len(_music_search_cache) > 50:
                _music_search_cache.pop(next(iter(_music_search_cache)))
        return result
    except Exception as e:
        print(f"[Музыка] Быстрый поиск: {e}")
        return None, None

def _resolve_audio_url(video_url: str):
    """Получает прямой аудиопоток без десятка повторных запросов."""
    if not HAS_YDL or not video_url:
        return None, None

    opts = {
        "quiet": True,
        "no_warnings": True,
        "noplaylist": True,
        "skip_download": True,
        "format": "bestaudio[ext=m4a]/bestaudio/best",
        "socket_timeout": 8,
        "retries": 0,
        "fragment_retries": 0,
        "cachedir": False,
    }
    try:
        with yt_dlp.YoutubeDL(opts) as ydl:
            info = ydl.extract_info(video_url, download=False)
        if not info:
            return None, None
        return info.get("url"), info.get("title") or "Песня"
    except Exception as e:
        print(f"[Музыка] Получение потока: {e}")
        return None, None

def play_youtube_audio(query: str, interface=None):
    global _current_music_file, _current_track_title, _is_music_mode

    if not HAS_YDL or not HAS_VLC:
        print("[Музыка] Нужны yt-dlp и python-vlc.")
        return False

    clean_query = _clean_song_query(query)
    if len(clean_query) < 2:
        return False

    try:
        # 1) Локальная песня запускается практически мгновенно.
        local_file = find_local_music(clean_query)
        if local_file:
            title = os.path.basename(local_file)
            _current_music_file = local_file
            _current_track_title = title
            if interface:
                interface.root.after(0, lambda: interface.update_music_button(True))
                interface.root.after(0, lambda: interface.set_music_track(title))
            return _play_audio_vlc(local_file, title, interface)

        # 2) Один быстрый результат YouTube вместо ytsearch10 + перебора.
        print(f"[Музыка] Быстрый поиск: {clean_query}")
        video_url, title = _youtube_find_fast(clean_query)
        if not video_url:
            return False

        # Сразу показываем найденное название, пока резолвится поток.
        if interface:
            interface.root.after(0, lambda t=title: interface.set_music_track(t, loading=True))

        audio_url, resolved_title = _resolve_audio_url(video_url)
        if not audio_url:
            return False

        _current_track_title = resolved_title or title or "Песня"
        if interface:
            interface.root.after(
                0,
                lambda t=_current_track_title: (
                    interface.update_music_button(True),
                    interface.set_music_track(t)
                )
            )

        return _play_audio_vlc(audio_url, _current_track_title, interface)
    except Exception as e:
        print(f"[Ошибка музыки]: {e}")
        return False

def _play_audio_vlc(audio_url: str, title: str, interface=None):
    global _music_thread, _music_stop, _vlc_instance, _vlc_player
    global _is_music_mode, _music_intensity

    try:
        _music_stop = True
        old_player = _vlc_player
        if old_player is not None:
            try:
                old_player.stop()
                old_player.release()
            except Exception:
                pass
        _vlc_player = None
        _music_stop = False
        _is_music_mode = True

        if _vlc_instance is None:
            _vlc_instance = vlc.Instance(
                "--quiet",
                "--no-video",
                "--network-caching=350"
            )

        player = _vlc_instance.media_player_new()
        media = _vlc_instance.media_new(audio_url)
        media.add_option(":no-video")
        media.add_option(":network-caching=350")
        media.add_option(":http-caching=300")
        media.add_option(":file-caching=300")
        player.set_media(media)
        player.audio_set_volume(_music_volume)
        _vlc_player = player

        if player.play() == -1:
            raise RuntimeError("VLC не смог открыть аудиопоток")

        def monitor(local_player):
            global _music_stop, _vlc_player, _is_music_mode, _music_intensity
            deadline = time.time() + 15
            started = False

            while not _music_stop:
                try:
                    state = local_player.get_state()
                except Exception:
                    break

                if state in (vlc.State.Playing, vlc.State.Paused):
                    started = True
                    _music_intensity = 0.8
                elif started and state in (
                    vlc.State.Ended, vlc.State.Stopped, vlc.State.Error
                ):
                    break
                elif not started and time.time() > deadline:
                    break
                time.sleep(0.08)

            try:
                local_player.stop()
                local_player.release()
            except Exception:
                pass

            if _vlc_player is local_player:
                _vlc_player = None
                _is_music_mode = False
                _music_intensity = 0.0
                if interface:
                    try:
                        interface.root.after(0, lambda: interface.update_music_button(False))
                    except Exception:
                        pass

        _music_thread = threading.Thread(
            target=monitor, args=(player,), daemon=True
        )
        _music_thread.start()
        return True

    except Exception as e:
        _is_music_mode = False
        _music_intensity = 0.0
        print(f"[Ошибка VLC]: {e}")
        return False

def stop_music():
    global _music_stop, _vlc_player, _is_music_mode, _music_intensity
    _music_stop = True
    player = _vlc_player
    _vlc_player = None
    _is_music_mode = False
    _music_intensity = 0.0
    if player is not None:
        try:
            player.stop()
            player.release()
        except Exception:
            pass
    return True

def is_music_playing():
    player = _vlc_player
    if player is None or not HAS_VLC:
        return False
    try:
        return player.get_state() == vlc.State.Playing
    except Exception:
        return False

def process_music_command(text: str, interface) -> bool:
    cleaned = re.sub(r"\s+", " ", text.lower()).strip()

    stop_phrases = [
        "стоп музыка", "выключи музыку", "останови музыку",
        "хватит музыки", "прекрати музыку",
        "стоп музика", "вимкни музику", "зупини музику",
        "досить музики", "припини музику"
    ]
    if any(p in cleaned for p in stop_phrases):
        stop_music()
        msg = T("music_stopped")
        interface.add_chat_message("Мита", msg, is_mita=True)
        interface.update_music_button(False)
        speak(msg, force=True)
        return True

    if "громкост" in cleaned or "гучн" in cleaned:
        vol_match = re.search(r"(\d{1,3})\s*%?", cleaned)
        if vol_match:
            vol = max(0, min(200, int(vol_match.group(1))))
            set_music_volume(vol)
            interface.update_volume_display()
            msg = T("volume_text").format(_music_volume)
            interface.add_chat_message("Мита", msg, is_mita=True)
            speak(msg, force=True)
            return True
        if "громче" in cleaned or "гучніше" in cleaned:
            volume_up(interface)
            return True
        if "тише" in cleaned or "тихіше" in cleaned:
            volume_down(interface)
            return True

    music_words = ["песн", "музык", "трек", "пісн", "музик"]
    if any(w in cleaned for w in music_words):
        song_name = _clean_song_query(cleaned)
        if len(song_name) > 1:
            interface.add_chat_message(
                "Мита",
                f"🎵 {T('music_playing')}{song_name}",
                is_mita=True
            )
            # Поиск полностью в фоне — UI и голос не блокируются.
            threading.Thread(
                target=play_youtube_audio,
                args=(song_name, interface),
                daemon=True
            ).start()
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

def get_base_path():
    if getattr(sys, 'frozen', False):
        return os.path.dirname(sys.executable)
    else:
        return os.path.dirname(os.path.abspath(__file__))


BASE_DIR = get_base_path()
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
    W, H = 1280, 780

    def __init__(self):
        self.root = tk.Tk()
        self.root.title(T("app_title"))
        self.root.geometry(f"{self.W}x{self.H}")
        self.root.minsize(1080, 680)
        self.root.configure(bg="#070911")
        self.root.resizable(True, True)

        self.running = True
        self.is_listening = False
        self.is_processing = False
        self.is_manual_recording = False
        self.is_dragging = False
        self.drag_x = self.drag_y = 0

        self.chat_history = []
        self.message_tags = []
        self.manual_audio_buffer = []
        self.manual_recording_thread = None

        self.tts_mute_button = None
        self.tts_muted = False
        self.corrector_button = None
        self.corrector_enabled = False
        self.lang_button = None
        self.mode_label = None
        self.volume_scale = None
        self.music_stop_button = None
        self.volume_label_id = None

        self.heart_beat_strength = 0.0
        self.heart_beat_target = 0.0
        self.heart_bpm = 72
        self.beat_phase = 0.0
        self.angle = 0.0
        self.phase = 0.0

        self.music_mode = False
        self.music_intensity = 0.0
        self.music_track = "Ничего не играет"

        self.colors = {
            "bg": "#070911",
            "bg2": "#0b0f1b",
            "panel": "#0d1220",
            "panel2": "#11182a",
            "line": "#1d2a42",
            "primary": "#7c5cff",
            "primary2": "#b69cff",
            "cyan": "#35d9ff",
            "pink": "#ff65b5",
            "green": "#49e6a2",
            "red": "#ff5577",
            "text": "#f4f7ff",
            "muted": "#8792aa",
            "dim": "#4e5a73",
        }

        self.root.bind("<Button-1>", self.start_move)
        self.root.bind("<B1-Motion>", self.on_move)
        self.root.bind("<ButtonRelease-1>", self.stop_move)
        self.root.protocol("WM_DELETE_WINDOW", self.quit)

        self.setup_ui()
        self.root.after(30, self.fade_in)
        self.audio_monitor = SystemAudioMonitor(self)
        self.audio_monitor.start()
        self.animate()

    def start_move(self, event):
        if event.widget in {
            getattr(self, "chat_display", None),
            getattr(self, "chat_input", None),
            getattr(self, "volume_scale", None),
            getattr(self, "music_stop_button", None),
            getattr(self, "manual_record_button", None),
            getattr(self, "tts_mute_button", None),
            getattr(self, "corrector_button", None),
            getattr(self, "lang_button", None),
            getattr(self, "mode_button", None),
        }:
            self.is_dragging = False
            return
        self.is_dragging = True
        self.drag_x = event.x_root
        self.drag_y = event.y_root

    def on_move(self, event):
        if not self.is_dragging:
            return
        dx = event.x_root - self.drag_x
        dy = event.y_root - self.drag_y
        self.root.geometry(
            f"+{self.root.winfo_x() + dx}+{self.root.winfo_y() + dy}"
        )
        self.drag_x = event.x_root
        self.drag_y = event.y_root

    def stop_move(self, event):
        self.is_dragging = False

    def rounded_rect(self, x1, y1, x2, y2, r=18, fill=None, outline=None, width=1):
        pts = [
            x1+r,y1, x2-r,y1, x2,y1+r,
            x2,y2-r, x2-r,y2, x1+r,y2,
            x1,y2-r, x1,y1+r
        ]
        return self.canvas.create_polygon(
            pts, smooth=True, splinesteps=16,
            fill=fill or "", outline=outline or "", width=width
        )

    def setup_ui(self):
        self.canvas = tk.Canvas(
            self.root, bg=self.colors["bg"], highlightthickness=0, bd=0
        )
        self.canvas.pack(fill=tk.BOTH, expand=True)
        self.create_background()

        # Верхняя панель
        self.rounded_rect(
            18, 16, self.W-18, 76, 20,
            self.colors["panel"], self.colors["line"]
        )
        self.canvas.create_text(
            42, 46, text="✦", font=("Arial", 22, "bold"),
            fill=self.colors["cyan"], anchor="w"
        )
        self.canvas.create_text(
            75, 38, text="MITA", font=("Arial", 18, "bold"),
            fill=self.colors["text"], anchor="w"
        )
        self.canvas.create_text(
            75, 58, text="VOICE INTELLIGENCE  /  ONLINE",
            font=("Arial", 7, "bold"), fill=self.colors["muted"], anchor="w"
        )

        self.online_dot = self.canvas.create_oval(
            self.W-165, 37, self.W-153, 49,
            fill=self.colors["green"], outline=""
        )
        self.canvas.create_text(
            self.W-143, 43, text="SYSTEM READY",
            font=("Arial", 8, "bold"), fill=self.colors["green"], anchor="w"
        )

        # Левая панель
        self.rounded_rect(
            18, 92, 290, self.H-18, 22,
            self.colors["panel"], self.colors["line"]
        )
        self.canvas.create_text(
            42, 120, text="CONTROL DECK",
            font=("Arial", 8, "bold"), fill=self.colors["dim"], anchor="w"
        )

        self.nav_items = []
        for i, (icon, label) in enumerate([
            ("⌂", T("nav_main")),
            ("◈", T("nav_chat")),
            ("⌘", T("nav_commands")),
            ("⚙", T("nav_settings")),
        ]):
            y = 153 + i*43
            bg = self.rounded_rect(
                34, y-17, 274, y+18, 11,
                self.colors["panel2"] if i == 0 else "",
                self.colors["line"] if i == 0 else ""
            )
            txt = self.canvas.create_text(
                52, y, text=f"{icon}   {label}",
                font=("Arial", 9, "bold"),
                fill=self.colors["text"] if i == 0 else self.colors["muted"],
                anchor="w"
            )
            self.nav_items.append((bg, txt))
            self.canvas.tag_bind(bg, "<Button-1>", lambda e, n=i: self.switch_tab(n))
            self.canvas.tag_bind(txt, "<Button-1>", lambda e, n=i: self.switch_tab(n))

        # Статус движка
        self.rounded_rect(34, 340, 274, 410, 14, self.colors["panel2"], self.colors["line"])
        self.canvas.create_text(
            50, 358, text="VOICE ENGINE",
            font=("Arial", 7, "bold"), fill=self.colors["dim"], anchor="w"
        )
        self.engine_dot = self.canvas.create_oval(
            50, 377, 58, 385, fill=self.colors["cyan"], outline=""
        )
        self.canvas.create_text(
            50, 399, text="RU / UA • ",
            font=("Arial", 7), fill=self.colors["muted"], anchor="w"
        )

        # Режим
        self.rounded_rect(34, 422, 274, 482, 14, self.colors["panel2"], self.colors["line"])
        self.canvas.create_text(
            50, 440, text=T("mode"),
            font=("Arial", 7, "bold"), fill=self.colors["dim"], anchor="w"
        )
        self.mode_label = self.canvas.create_text(
            50, 460, text=get_mode_name(_mita_mode),
            font=("Arial", 8, "bold"), fill=self.colors["primary2"], anchor="w"
        )
        self.mode_button = tk.Button(
            self.root, text=T("change_mode"), command=self.change_mode,
            font=("Arial", 8, "bold"), bg=self.colors["panel2"],
            fg=self.colors["muted"], activebackground=self.colors["primary"],
            activeforeground="white", relief="flat", bd=0, cursor="hand2"
        )
        self.mode_button.place(x=50, y=475, width=210, height=25)

        # Голос
        self.tts_mute_button = tk.Button(
            self.root, text=T("voice_on"), command=self.toggle_tts_mute,
            font=("Arial", 8, "bold"), bg=self.colors["panel2"],
            fg=self.colors["green"], activebackground=self.colors["panel2"],
            relief="flat", bd=0, cursor="hand2"
        )
        self.tts_mute_button.place(x=50, y=518, width=210, height=28)

        # Исправитель
        self.corrector_button = tk.Button(
            self.root, text=T("corrector_off"), command=self.toggle_corrector,
            font=("Arial", 8, "bold"), bg=self.colors["panel2"],
            fg=self.colors["muted"], activebackground=self.colors["panel2"],
            relief="flat", bd=0, cursor="hand2"
        )
        self.corrector_button.place(x=50, y=551, width=210, height=28)

        # Язык
        self.lang_button = tk.Button(
            self.root,
            text="🇺🇦 Українська" if UI_LANGUAGE == "ru" else "🇷🇺 Русский",
            command=self.toggle_language,
            font=("Arial", 8, "bold"), bg=self.colors["panel2"],
            fg=self.colors["muted"], relief="flat", bd=0, cursor="hand2"
        )
        self.lang_button.place(x=50, y=584, width=210, height=28)

        # Ручная запись
        self.manual_record_button = tk.Button(
            self.root, text=T("manual_input_btn"),
            command=self.toggle_manual_record,
            font=("Arial", 8, "bold"), bg=self.colors["primary"],
            fg="white", activebackground=self.colors["primary2"],
            relief="flat", bd=0, cursor="hand2"
        )
        self.manual_record_button.place(x=50, y=620, width=210, height=31)

        # Громкость
        self.canvas.create_text(
            50, 674, text=f"VOLUME  {_music_volume}%",
            font=("Arial", 7, "bold"), fill=self.colors["dim"], anchor="w"
        )
        self.volume_label_id = self.canvas.create_text(
            260, 674, text=f"{_music_volume}%",
            font=("Arial", 7, "bold"), fill=self.colors["primary2"], anchor="e"
        )
        self.volume_scale = tk.Scale(
            self.root, from_=0, to=200, orient=tk.HORIZONTAL,
            bg=self.colors["panel"], fg=self.colors["primary2"],
            troughcolor=self.colors["line"], activebackground=self.colors["primary"],
            highlightthickness=0, bd=0, sliderlength=14,
            command=self.on_volume_change
        )
        self.volume_scale.place(x=45, y=684, width=220, height=30)
        self.volume_scale.set(_music_volume)

        self.music_stop_button = tk.Button(
            self.root, text=T("music_stop"),
            command=self.stop_music_click,
            font=("Arial", 7, "bold"), bg=self.colors["panel2"],
            fg=self.colors["muted"], activebackground=self.colors["red"],
            relief="flat", bd=0, state=tk.DISABLED, cursor="hand2"
        )
        self.music_stop_button.place(x=50, y=720, width=210, height=26)

        # Центральный блок
        self.rounded_rect(
            308, 92, 900, 500, 26,
            self.colors["panel"], self.colors["line"]
        )
        self.canvas.create_text(
            338, 120, text="MITA CORE",
            font=("Arial", 8, "bold"), fill=self.colors["dim"], anchor="w"
        )
        self.canvas.create_text(
            870, 120, text="LIVE",
            font=("Arial", 7, "bold"), fill=self.colors["green"], anchor="e"
        )

        self.cx, self.cy = 604, 292
        self.aura = []
        for radius in (190, 160, 132, 108):
            self.aura.append(
                self.canvas.create_oval(
                    self.cx-radius, self.cy-radius,
                    self.cx+radius, self.cy+radius,
                    outline=self.colors["line"], width=1
                )
            )

        self.core_outer = self.canvas.create_oval(
            self.cx-88, self.cy-88, self.cx+88, self.cy+88,
            fill="#10172a", outline=self.colors["primary"], width=2
        )
        self.core_inner = self.canvas.create_oval(
            self.cx-67, self.cy-67, self.cx+67, self.cy+67,
            fill="#151c31", outline=self.colors["cyan"], width=1
        )

        self.heart_size = 48
        self.heart = self.create_heart(
            self.cx, self.cy, self.heart_size, self.colors["primary"]
        )
        self.heart_glow = self.canvas.create_oval(
            self.cx-80, self.cy-80, self.cx+80, self.cy+80,
            outline=self.colors["primary"], width=2
        )

        self.orbit_elements = []
        for i in range(12):
            obj = self.canvas.create_text(
                self.cx, self.cy, text="•",
                font=("Arial", 7 + i % 4, "bold"),
                fill=self.colors["cyan"]
            )
            self.orbit_elements.append({
                "id": obj,
                "angle": i * math.pi*2/12,
                "radius": 120 + (i % 3)*24,
                "speed": 0.008 + i*0.001
            })

        self.status_text = self.canvas.create_text(
            self.cx, 478, text=T("ready"),
            font=("Arial", 13, "bold"), fill=self.colors["primary2"]
        )
        self.status_sub = self.canvas.create_text(
            self.cx, 502, text=T("waiting"),
            font=("Arial", 8), fill=self.colors["muted"]
        )

        # Аудио-эквалайзер
        self.bars = []
        for i in range(31):
            x = self.cx - 155 + i*10
            bar = self.canvas.create_rectangle(
                x, 548, x+5, 552,
                fill=self.colors["line"], outline=""
            )
            self.bars.append({
                "id": bar, "x": x, "base": 552,
                "height": 4, "phase": random.random()*math.pi*2
            })

        # Карточка музыки
        self.rounded_rect(
            326, 132, 882, 222, 16,
            self.colors["panel2"], self.colors["line"]
        )
        self.canvas.create_text(
            350, 153, text="NOW PLAYING",
            font=("Arial", 7, "bold"), fill=self.colors["dim"], anchor="w"
        )
        self.music_icon = self.canvas.create_text(
            360, 184, text="♫",
            font=("Arial", 24, "bold"), fill=self.colors["pink"]
        )
        self.music_title = self.canvas.create_text(
            395, 178, text=self.music_track,
            font=("Arial", 9, "bold"), fill=self.colors["text"], anchor="w"
        )
        self.music_state = self.canvas.create_text(
            395, 198, text="Ожидаю команду",
            font=("Arial", 7), fill=self.colors["muted"], anchor="w"
        )

        # Правая панель
        self.rounded_rect(
            918, 92, self.W-18, self.H-18, 22,
            self.colors["panel"], self.colors["line"]
        )
        self.canvas.create_text(
            944, 120, text="COMMAND MATRIX",
            font=("Arial", 8, "bold"), fill=self.colors["dim"], anchor="w"
        )

        commands = [
            ("01", "ЗАПУСК", "запусти [программа]"),
            ("02", "САЙТ", "открой [сайт]"),
            ("03", "ОКНО", "перемести [окно] на 2"),
            ("04", "ПЕЧАТЬ", "напиши [текст]"),
            ("05", "МУЗЫКА", "включи песню [название]"),
            ("06", "СТОП", "стоп музыка"),
            ("07", "ГРОМКОСТЬ", "громкость 80"),
            ("08", "СКРИН", "сделай скриншот"),
            ("09", "КОПИ", "скопируй"),
            ("10", "ВСТАВКА", "вставь"),
        ]
        y = 153
        for num, title, desc in commands:
            self.canvas.create_text(
                944, y, text=num, font=("Arial", 7, "bold"),
                fill=self.colors["cyan"], anchor="w"
            )
            self.canvas.create_text(
                978, y, text=title, font=("Arial", 8, "bold"),
                fill=self.colors["text"], anchor="w"
            )
            self.canvas.create_text(
                978, y+16, text=desc, font=("Arial", 7),
                fill=self.colors["muted"], anchor="w"
            )
            y += 51

        # Чат снизу по центру
        self.rounded_rect(
            308, 610, 900, self.H-18, 22,
            self.colors["panel"], self.colors["line"]
        )
        self.canvas.create_text(
            332, 635, text=T("chat_with_mita"),
            font=("Arial", 9, "bold"), fill=self.colors["primary2"], anchor="w"
        )
        self.canvas.create_text(
            332, 654, text="Текстовый канал • Ctrl+A / Ctrl+C / Ctrl+V",
            font=("Arial", 7), fill=self.colors["muted"], anchor="w"
        )

        self.chat_frame = tk.Frame(self.root, bg=self.colors["panel"])
        self.chat_frame.place(x=325, y=666, width=560, height=62)
        self.chat_display = scrolledtext.ScrolledText(
            self.chat_frame, bg="#080c15", fg=self.colors["text"],
            font=("Arial", 8), wrap=tk.WORD, bd=0,
            relief="flat", highlightthickness=1,
            highlightbackground=self.colors["line"],
            highlightcolor=self.colors["primary"]
        )
        self.chat_display.pack(fill=tk.BOTH, expand=True)
        self.chat_display.config(state=tk.DISABLED)

        self.chat_menu = Menu(
            self.root, tearoff=0, bg=self.colors["panel2"],
            fg=self.colors["text"], activebackground=self.colors["primary"]
        )
        self.chat_menu.add_command(
            label="📋 Копировать", command=self.copy_selected_message
        )
        self.chat_menu.add_command(
            label="📋 Копировать всё", command=self.copy_all_chat
        )
        self.chat_menu.add_separator()
        self.chat_menu.add_command(
            label="🗑 Очистить", command=self.clear_chat
        )
        self.chat_display.bind("<Button-3>", self.show_chat_menu)
        self.chat_display.bind("<Control-c>", lambda e: self.copy_selected_message())
        self.chat_display.bind("<Control-a>", lambda e: self.select_all_chat())

        self.input_frame = tk.Frame(self.root, bg=self.colors["panel"])
        self.input_frame.place(x=325, y=730, width=560, height=35)
        self.chat_input = tk.Entry(
            self.input_frame, bg=self.colors["panel2"], fg=self.colors["text"],
            insertbackground=self.colors["cyan"], font=("Arial", 9),
            bd=0, relief="flat"
        )
        self.chat_input.pack(side=tk.LEFT, fill=tk.BOTH, expand=True, padx=(0,5))
        self.chat_input.bind("<Return>", lambda e: self.send_chat_message())
        self.chat_input.bind("<Control-a>", lambda e: self.select_all_input())
        self.chat_input.bind("<Control-c>", lambda e: self.copy_input_text())
        self.chat_input.bind("<Control-v>", lambda e: self.paste_input_text())
        self.chat_input.bind("<Button-3>", self.show_input_menu)

        self.send_button = tk.Button(
            self.input_frame, text="➤", command=self.send_chat_message,
            font=("Arial", 11, "bold"), bg=self.colors["primary"],
            fg="white", activebackground=self.colors["primary2"],
            relief="flat", bd=0, cursor="hand2"
        )
        self.send_button.pack(side=tk.RIGHT, fill=tk.Y)

        self.command_text = self.canvas.create_text(
            944, self.H-55, text="LAST COMMAND  —",
            font=("Arial", 7, "bold"), fill=self.colors["dim"], anchor="w"
        )

        self.audio_indicator = self.canvas.create_text(
            self.W-45, self.H-40, text="●",
            font=("Arial", 10, "bold"), fill=self.colors["dim"]
        )

        self.ram_text = self.canvas.create_text(
            944, self.H-35, text=f"RAM {self.get_ram_usage()} MB",
            font=("Arial", 7), fill=self.colors["muted"], anchor="w"
        )

        self.create_particles()
        self.create_stars()

        hello_msg = (
            f"{T('mita_greeting')}\n"
            f"{T('mita_mode')}{get_mode_name(_mita_mode)}\n"
            f"{T('mita_corrector')}"
            f"{T('corrector_off_info') if not _text_corrector_enabled else T('corrector_on_info')}\n"
            f"{T('mita_lang')}\n{T('mita_help')}"
        )
        self.add_chat_message("Мита", hello_msg, is_mita=True)
        self.current_tab = 0
        self.update_key_info()

    # --------------------- UI helpers ---------------------

    def create_background(self):
        for y in range(self.H):
            t = y / max(1, self.H-1)
            r = int(7 + 5*t)
            g = int(9 + 6*t)
            b = int(17 + 10*t)
            self.canvas.create_line(0, y, self.W, y, fill=f"#{r:02x}{g:02x}{b:02x}")
        for x in range(-self.H, self.W, 100):
            self.canvas.create_line(
                x, 0, x+self.H, self.H,
                fill="#0d1423"
            )

    def create_heart(self, x, y, size, color):
        points = []
        for t in range(0, 360, 5):
            rad = math.radians(t)
            hx = 16*math.sin(rad)**3
            hy = (
                13*math.cos(rad)
                - 5*math.cos(2*rad)
                - 2*math.cos(3*rad)
                - math.cos(4*rad)
            )
            points += [x+hx*size/16, y-hy*size/16]
        return {
            "id": self.canvas.create_polygon(
                points, fill=color, outline=color, width=2
            ),
            "glow": self.canvas.create_polygon(
                points, fill="", outline=self.colors["pink"], width=2
            )
        }

    def create_particles(self):
        self.particles = []
        for _ in range(38):
            self.particles.append({
                "angle": random.random()*math.pi*2,
                "radius": random.uniform(95, 215),
                "speed": random.uniform(.12,.7),
                "symbol": random.choice(["·","✦","•","◇"]),
                "id": None
            })

    def create_stars(self):
        self.stars = []
        for _ in range(48):
            x = random.randint(15, self.W-15)
            y = random.randint(85, self.H-15)
            size = random.choice([1,2,2,3])
            obj = self.canvas.create_oval(
                x,y,x+size,y+size, fill="#28344e", outline=""
            )
            self.stars.append({
                "id": obj, "x": x, "y": y,
                "speed": random.uniform(.3,1.1)
            })

    def get_ram_usage(self):
        try:
            return int(psutil.Process().memory_info().rss/1024/1024)
        except Exception:
            return 0

    def update_heart(self, size, color):
        points = []
        scale = 1 + math.sin(self.beat_phase)*self.heart_beat_strength*.18
        for t in range(0,360,5):
            rad = math.radians(t)
            hx = 16*math.sin(rad)**3
            hy = (
                13*math.cos(rad)
                - 5*math.cos(2*rad)
                - 2*math.cos(3*rad)
                - math.cos(4*rad)
            )
            points += [
                self.cx+hx*size/16*scale,
                self.cy-hy*size/16*scale
            ]
        self.canvas.coords(self.heart["id"], *points)
        self.canvas.coords(self.heart["glow"], *points)
        self.canvas.itemconfig(self.heart["id"], fill=color, outline=color)
        self.canvas.itemconfig(self.heart["glow"], outline=self.colors["pink"])

    def update_heart_beat(self, strength):
        self.heart_beat_target = strength
        if strength > .08:
            self.heart_bpm = (
                125 + int(strength*75) if self.music_mode
                else 72 + int(strength*45)
            )

    def animate(self):
        if not self.running:
            return

        self.music_mode = _is_music_mode
        self.music_intensity = _music_intensity
        self.angle += .035
        self.phase += .10
        self.beat_phase += .035*(self.heart_bpm/60)

        self.heart_beat_strength += (
            self.heart_beat_target-self.heart_beat_strength
        )*.14

        if self.music_mode:
            self.heart_beat_strength = max(
                self.heart_beat_strength,
                .42 + self.music_intensity*.35
            )
        else:
            self.heart_beat_strength *= .97

        if self.music_mode:
            color = self.colors["pink"]
            status = f"♫  {self.music_track}"
            sub = "МУЗЫКА ИГРАЕТ • LIVE VISUALIZER"
        elif self.is_listening:
            color = self.colors["cyan"]
            status, sub = T("listening"), T("listening_sub")
        elif self.is_processing:
            color = self.colors["primary2"]
            status, sub = T("processing"), T("processing_sub")
        elif self.is_manual_recording:
            color = self.colors["pink"]
            status, sub = T("manual_recording"), T("manual_sub")
        elif self.tts_muted:
            color = self.colors["red"]
            status, sub = "🔇 VOICE MUTED", T("voice_off_status")
        else:
            color = self.colors["primary2"]
            status, sub = "✦ MITA ONLINE", T("waiting")

        pulse = 48 + math.sin(self.angle*2)*3
        if self.music_mode:
            pulse *= 1.10 + self.heart_beat_strength*.08
        self.update_heart(pulse, color)

        self.canvas.itemconfig(self.status_text, text=status, fill=color)
        self.canvas.itemconfig(self.status_sub, text=sub)
        self.canvas.itemconfig(self.core_outer, outline=color)

        for i, obj in enumerate(self.aura):
            delta = math.sin(self.angle*1.2+i)*2
            delta += self.heart_beat_strength*12*math.sin(self.beat_phase+i)
            if self.music_mode:
                delta *= 1.8
            base = (190,160,132,108)[i]
            self.canvas.coords(
                obj,
                self.cx-base-delta, self.cy-base-delta,
                self.cx+base+delta, self.cy+base+delta
            )
            self.canvas.itemconfig(
                obj,
                outline=color if self.music_mode and i == 2 else self.colors["line"]
            )

        for item in self.orbit_elements:
            item["angle"] += item["speed"]*(2.0 if self.music_mode else 1)
            r = item["radius"] + self.heart_beat_strength*16
            x = self.cx + math.cos(item["angle"])*r
            y = self.cy + math.sin(item["angle"])*r
            self.canvas.coords(item["id"], x, y)
            self.canvas.itemconfig(
                item["id"], fill=self.colors["pink"] if self.music_mode else color
            )

        for p in self.particles:
            p["angle"] += p["speed"]*.01
            r = p["radius"]*(1+self.heart_beat_strength*.08)
            x = self.cx+math.cos(p["angle"])*r
            y = self.cy+math.sin(p["angle"])*r
            if p["id"] is None:
                p["id"] = self.canvas.create_text(
                    x,y,text=p["symbol"],
                    font=("Arial",random.randint(7,11)),
                    fill=color
                )
            else:
                self.canvas.coords(p["id"],x,y)
                self.canvas.itemconfig(
                    p["id"], fill=self.colors["pink"] if self.music_mode else color
                )

        now = time.time()
        for i, star in enumerate(self.stars):
            v = .3 + .4*(math.sin(now*star["speed"]+i)+1)/2
            c = int(38+55*v)
            self.canvas.itemconfig(
                star["id"],
                fill=f"#{c:02x}{int(c*1.1):02x}{int(c*1.45):02x}"
            )

        for i, bar in enumerate(self.bars):
            if self.music_mode:
                target = 6 + abs(
                    math.sin(self.phase*1.8+i*.48)
                )*random.randint(18,48)
            elif self.is_listening or self.is_processing or self.is_manual_recording:
                target = 6 + abs(
                    math.sin(self.phase+i*.48)
                )*random.randint(8,25)
            else:
                target = 3 + abs(math.sin(self.phase*.4+i)) * 3
            bar["height"] += (target-bar["height"])*.28
            h = max(3, bar["height"])
            self.canvas.coords(
                bar["id"], bar["x"], bar["base"]-h,
                bar["x"]+5, bar["base"]
            )
            self.canvas.itemconfig(
                bar["id"],
                fill=self.colors["pink"] if self.music_mode else color
            )

        self.canvas.itemconfig(
            self.audio_indicator,
            fill=self.colors["pink"] if self.music_mode else (
                self.colors["cyan"] if self.heart_beat_strength > .1
                else self.colors["dim"]
            )
        )
        self.canvas.itemconfig(
            self.ram_text, text=f"RAM {self.get_ram_usage()} MB"
        )
        self.root.after(20, self.animate)

    def fade_in(self, alpha=0):
        if not self.running:
            return
        alpha += .08
        if alpha >= 1:
            self.root.attributes("-alpha", .98)
            return
        self.root.attributes("-alpha", alpha)
        self.root.after(15, lambda: self.fade_in(alpha))

    # --------------------- state ---------------------

    def set_listening(self, state):
        self.is_listening = state
        if state:
            self.is_processing = False

    def set_processing(self, state):
        self.is_processing = state
        if state:
            self.is_listening = False

    def show_command(self, text):
        short = re.sub(r"\s+", " ", str(text)).strip()
        if len(short) > 48:
            short = short[:48] + "..."
        self.canvas.itemconfig(
            self.command_text,
            text=f"LAST COMMAND  {short or '—'}"
        )

    def set_music_track(self, title, loading=False):
        self.music_track = title or "Ничего не играет"
        try:
            self.canvas.itemconfig(self.music_title, text=self.music_track)
            self.canvas.itemconfig(
                self.music_state,
                text="Поиск и подключение..." if loading
                else ("ИГРАЕТ • LIVE" if _is_music_mode else "Ожидаю команду")
            )
        except Exception:
            pass

    def update_music_button(self, is_playing):
        if is_playing:
            self.music_stop_button.config(
                text="■  STOP MUSIC", bg=self.colors["red"],
                fg="white", state=tk.NORMAL
            )
            self.set_music_track(_current_track_title or self.music_track)
        else:
            self.music_stop_button.config(
                text=T("music_stop"), bg=self.colors["panel2"],
                fg=self.colors["muted"], state=tk.DISABLED
            )
            if not _is_music_mode:
                self.set_music_track(self.music_track)

    def update_volume_display(self):
        if self.volume_scale:
            self.volume_scale.set(_music_volume)
        if self.volume_label_id:
            self.canvas.itemconfig(
                self.volume_label_id, text=f"{_music_volume}%"
            )

    # --------------------- language/mode/settings ---------------------

    def toggle_language(self):
        global UI_LANGUAGE
        UI_LANGUAGE = "ua" if UI_LANGUAGE == "ru" else "ru"
        _save_ui_language()
        self.refresh_ui()

    def refresh_ui(self):
        self.root.title(T("app_title"))
        self.mode_button.config(text=T("change_mode"))
        self.manual_record_button.config(text=T("manual_input_btn"))
        self.tts_mute_button.config(
            text=T("voice_on") if not self.tts_muted else T("voice_off")
        )
        self.corrector_button.config(
            text=T("corrector_on") if _text_corrector_enabled else T("corrector_off")
        )
        self.lang_button.config(
            text="🇺🇦 Українська" if UI_LANGUAGE == "ru" else "🇷🇺 Русский"
        )
        self.music_stop_button.config(
            text="■  STOP MUSIC" if _is_music_mode else T("music_stop")
        )
        self.canvas.itemconfig(
            self.status_text, text=T("ready") if not _is_music_mode else self.music_track
        )
        self.canvas.itemconfig(
            self.status_sub, text=T("waiting")
        )
        self.canvas.itemconfig(
            self.mode_label, text=get_mode_name(_mita_mode)
        )

    def change_mode(self):
        dialog = tk.Toplevel(self.root)
        dialog.title(T("mode"))
        dialog.geometry("440x330")
        dialog.configure(bg=self.colors["bg"])
        dialog.resizable(False, False)
        dialog.transient(self.root)
        dialog.grab_set()

        tk.Label(
            dialog, text="MITA MODE",
            font=("Arial",16,"bold"),
            bg=self.colors["bg"], fg=self.colors["cyan"]
        ).pack(pady=(24,5))
        tk.Label(
            dialog,
            text="Выберите источник ответов",
            font=("Arial",9),
            bg=self.colors["bg"], fg=self.colors["muted"]
        ).pack(pady=(0,18))

        var = tk.StringVar(value=_mita_mode)
        for mode, label, desc in [
            (MODE_SYSTEM,T("mode_system"),"Системные действия"),
            (MODE_AI,T("mode_ai"),"Только ИИ"),
            (MODE_ALL,T("mode_all"),"Система + ИИ"),
        ]:
            frame = tk.Frame(dialog,bg=self.colors["panel2"])
            frame.pack(fill="x",padx=24,pady=4)
            tk.Radiobutton(
                frame,text=label,variable=var,value=mode,
                font=("Arial",9,"bold"),bg=self.colors["panel2"],
                fg=self.colors["text"],selectcolor=self.colors["panel2"],
                activebackground=self.colors["panel2"],
                activeforeground=self.colors["cyan"],bd=0
            ).pack(side="left",padx=10,pady=9)
            tk.Label(
                frame,text=desc,font=("Arial",7),
                bg=self.colors["panel2"],fg=self.colors["muted"]
            ).pack(side="right",padx=10)

        def apply():
            mode = var.get()
            set_mita_mode(mode)
            self.update_mode_display()
            dialog.destroy()
            msg = T("mode_changed").format(get_mode_name(mode))
            self.add_chat_message("Мита",msg,is_mita=True)
            speak(T("mode_changed_text").format(get_mode_name(mode)),force=True)

        tk.Button(
            dialog,text=T("apply"),command=apply,
            font=("Arial",9,"bold"),bg=self.colors["primary"],
            fg="white",relief="flat",bd=0,cursor="hand2"
        ).pack(pady=18,ipadx=30,ipady=7)

    def update_mode_display(self):
        if self.mode_label:
            self.canvas.itemconfig(
                self.mode_label,text=get_mode_name(_mita_mode)
            )

    def toggle_tts_mute(self):
        global TTS_VOLUME
        self.tts_muted = not self.tts_muted
        if self.tts_muted:
            stop_tts()
            TTS_VOLUME = 0
            self.tts_mute_button.config(
                text=T("voice_off"),bg=self.colors["red"],fg="white"
            )
            self.add_chat_message("Мита",T("voice_off"),is_mita=True)
        else:
            TTS_VOLUME = 1
            self.tts_mute_button.config(
                text=T("voice_on"),bg=self.colors["panel2"],fg=self.colors["green"]
            )
            self.add_chat_message("Мита",T("voice_on"),is_mita=True)
            speak(T("voice_on_text"),force=True)

    def toggle_corrector(self):
        self.corrector_enabled = not self.corrector_enabled
        set_text_corrector(self.corrector_enabled)
        if self.corrector_enabled:
            self.corrector_button.config(
                text=T("corrector_on"),fg=self.colors["green"]
            )
            msg=T("corrector_on_text")
        else:
            self.corrector_button.config(
                text=T("corrector_off"),fg=self.colors["muted"]
            )
            msg=T("corrector_off_text")
        self.add_chat_message("Мита",msg,is_mita=True)
        speak(msg,force=True)

    def toggle_manual_record(self):
        if self.is_manual_recording:
            self.stop_manual_recording()
        else:
            self.start_manual_recording()

    # --------------------- manual voice ---------------------

    def start_manual_recording(self):
        if self.is_manual_recording:
            return
        self.is_manual_recording = True
        self.manual_audio_buffer = []
        self.manual_record_button.config(
            text=T("stop_record"),bg=self.colors["red"]
        )
        self.set_processing(False)
        self.canvas.itemconfig(
            self.status_text,text=T("manual_recording"),
            fill=self.colors["pink"]
        )
        self.manual_recording_thread = threading.Thread(
            target=self._manual_record_loop,daemon=True
        )
        self.manual_recording_thread.start()

    def stop_manual_recording(self):
        if not self.is_manual_recording:
            return
        self.is_manual_recording=False
        self.manual_record_button.config(
            text=T("manual_input_btn"),bg=self.colors["primary"]
        )
        self.set_processing(True)
        if self.manual_audio_buffer:
            threading.Thread(
                target=self._process_manual_audio,daemon=True
            ).start()
        else:
            self.add_chat_message("Мита",T("no_audio"),is_mita=True)
            self._reset_manual_state()

    def _manual_record_loop(self):
        try:
            with sd.InputStream(
                samplerate=16000,channels=1,dtype="float32",
                blocksize=1024,callback=self._manual_audio_callback
            ):
                while self.is_manual_recording:
                    time.sleep(.04)
        except Exception as e:
            self.root.after(
                0,lambda:self.add_chat_message(
                    "Мита",T("error_text").format(e),is_mita=True
                )
            )
            self.root.after(0,self._reset_manual_state)

    def _manual_audio_callback(self,indata,frames,time,status):
        if self.is_manual_recording:
            self.manual_audio_buffer.append(indata.copy())

    def _process_manual_audio(self):
        try:
            full=np.concatenate(self.manual_audio_buffer)
            self.manual_audio_buffer=[]
            stream=recognizer.create_stream()
            stream.accept_waveform(16000,full)
            recognizer.decode_stream(stream)
            text=stream.result.text.strip()
            if text:
                self.root.after(0,lambda:self.add_chat_message("Вы",f"🎤 {text}"))
                self.root.after(0,lambda:self._process_manual_command(text))
            else:
                self.root.after(
                    0,lambda:self.add_chat_message(
                        "Мита",T("recognition_failed"),is_mita=True
                    )
                )
                self.root.after(0,self._reset_manual_state)
        except Exception as e:
            self.root.after(
                0,lambda:self.add_chat_message(
                    "Мита",T("error_text").format(e),is_mita=True
                )
            )
            self.root.after(0,self._reset_manual_state)

    def _process_manual_command(self,text):
        self.is_processing=True
        if self.process_chat_system_command(text):
            self._reset_manual_state()
            return
        if _mita_mode in [MODE_AI,MODE_ALL]:
            response=ask_groq_chat(text,self.chat_history)
            self.add_chat_message("Мита",response,is_mita=True)
            self.chat_history.append({"role":"assistant","content":response})
            speak(response)
        else:
            response=T("command_not_found")
            self.add_chat_message("Мита",response,is_mita=True)
            speak(response)
        self._reset_manual_state()

    def _reset_manual_state(self):
        self.is_processing=False
        self.is_manual_recording=False
        self.manual_audio_buffer=[]
        self.manual_record_button.config(
            text=T("manual_input_btn"),bg=self.colors["primary"]
        )
        self.canvas.itemconfig(
            self.status_text,text=T("ready"),fill=self.colors["primary2"]
        )
        self.canvas.itemconfig(self.status_sub,text=T("waiting"))

    # --------------------- chat ---------------------

    def show_input_menu(self,event):
        menu=Menu(
            self.root,tearoff=0,bg=self.colors["panel2"],
            fg=self.colors["text"],activebackground=self.colors["primary"]
        )
        menu.add_command(label="✂ Вырезать",command=self.cut_input_text)
        menu.add_command(label="📋 Копировать",command=self.copy_input_text)
        menu.add_command(label="📎 Вставить",command=self.paste_input_text)
        menu.add_command(label="⌁ Выделить всё",command=self.select_all_input)
        menu.post(event.x_root,event.y_root)

    def cut_input_text(self):
        self.chat_input.event_generate("<<Cut>>")

    def copy_input_text(self):
        self.chat_input.event_generate("<<Copy>>")

    def paste_input_text(self):
        self.chat_input.event_generate("<<Paste>>")

    def select_all_input(self):
        self.chat_input.select_range(0,tk.END)
        self.chat_input.focus_set()
        return "break"

    def select_all_chat(self):
        self.chat_display.config(state=tk.NORMAL)
        self.chat_display.tag_add(tk.SEL,"1.0",tk.END)
        self.chat_display.config(state=tk.DISABLED)
        return "break"

    def show_chat_menu(self,event):
        self.chat_menu.post(event.x_root,event.y_root)

    def copy_selected_message(self):
        try:
            text=self.chat_display.get(tk.SEL_FIRST,tk.SEL_LAST).strip()
        except tk.TclError:
            text=""
        if not text:
            content=self.chat_display.get("1.0",tk.END).strip()
            if content:
                text=content
        if text:
            self.root.clipboard_clear()
            self.root.clipboard_append(text)
            self.root.update()
            self.show_temp_status(T("copy_success"))
        else:
            self.show_temp_status(T("copy_failed"))

    def copy_all_chat(self):
        content=self.chat_display.get("1.0",tk.END).strip()
        if not content:
            self.show_temp_status(T("chat_empty"))
            return
        self.root.clipboard_clear()
        self.root.clipboard_append(content)
        self.root.update()
        self.show_temp_status(T("copy_all_success"))

    def clear_chat(self):
        if messagebox.askyesno(
            "Очистка чата" if UI_LANGUAGE=="ru" else "Очищення чату",
            T("clear_confirm")
        ):
            self.chat_display.config(state=tk.NORMAL)
            self.chat_display.delete("1.0",tk.END)
            self.chat_display.config(state=tk.DISABLED)
            self.chat_history=[]
            self.add_chat_message(
                "Мита",T("chat_cleared"),is_mita=True
            )

    def add_chat_message(self,sender,message,is_mita=False):
        message=re.sub(r"[\*_`]","",str(message or ""))
        self.chat_display.config(state=tk.NORMAL)
        stamp=datetime.now().strftime("%H:%M")
        if is_mita:
            self.chat_display.insert(
                tk.END,f"◆ МИТА  {stamp}\n","mita_name"
            )
            self.chat_display.insert(
                tk.END,f"{message}\n\n","mita_text"
            )
        else:
            self.chat_display.insert(
                tk.END,f"◇ ВЫ  {stamp}\n","user_name"
            )
            self.chat_display.insert(
                tk.END,f"{message}\n\n","user_text"
            )
        self.chat_display.tag_config(
            "mita_name",foreground=self.colors["cyan"],
            font=("Arial",8,"bold")
        )
        self.chat_display.tag_config(
            "mita_text",foreground="#d7deee",font=("Arial",8)
        )
        self.chat_display.tag_config(
            "user_name",foreground=self.colors["primary2"],
            font=("Arial",8,"bold")
        )
        self.chat_display.tag_config(
            "user_text",foreground=self.colors["text"],font=("Arial",8)
        )
        self.chat_display.see(tk.END)
        self.chat_display.config(state=tk.DISABLED)

    def send_chat_message(self):
        msg=self.chat_input.get().strip()
        if not msg:
            return
        self.chat_input.delete(0,tk.END)
        self.add_chat_message("Вы",msg)
        self.chat_history.append({"role":"user","content":msg})
        self.set_processing(True)
        threading.Thread(
            target=self.process_chat_command,args=(msg,),daemon=True
        ).start()

    def process_chat_command(self,message):
        try:
            if _mita_mode in [MODE_SYSTEM,MODE_ALL]:
                result=self.process_chat_system_command(message)
                if result and _mita_mode==MODE_SYSTEM:
                    self.root.after(
                        0,lambda:self.add_chat_message(
                            "Мита",result,is_mita=True
                        )
                    )
                    self.root.after(0,lambda:self.set_processing(False))
                    return
            if _mita_mode in [MODE_AI,MODE_ALL]:
                if _mita_mode==MODE_AI or not result:
                    response=ask_groq_chat(message,self.chat_history)
                    self.root.after(
                        0,lambda r=response:self.add_chat_message(
                            "Мита",r,is_mita=True
                        )
                    )
                    self.chat_history.append(
                        {"role":"assistant","content":response}
                    )
                    speak(response)
            elif not result:
                response=T("command_not_found")
                self.root.after(
                    0,lambda:self.add_chat_message(
                        "Мита",response,is_mita=True
                    )
                )
                speak(response)
        except Exception as e:
            self.root.after(
                0,lambda:self.add_chat_message(
                    "Мита",T("error_text").format(e),is_mita=True
                )
            )
        self.root.after(0,lambda:self.set_processing(False))

    def process_chat_system_command(self,text):
        cleaned=text.lower().strip()
        if process_music_command(cleaned,self):
            return "🎵 Команда музыки принята"

        # Быстрые системные команды.
        if "все окна" in cleaned or "сверни все" in cleaned:
            minimize_all_windows()
            return T("minimized_all")

        if "смени язык" in cleaned or "поменяй язык" in cleaned or "зміни мову" in cleaned:
            change_language()
            return T("language_changed")

        if "скриншот" in cleaned or "скрин" in cleaned:
            pyautogui.press("printscreen")
            return T("screenshot_done")

        if "скопируй" in cleaned or "скопіюй" in cleaned:
            keyboard.press_and_release("ctrl+c")
            return T("copied")

        if "вставь" in cleaned or "встав" in cleaned:
            keyboard.press_and_release("ctrl+v")
            return T("pasted")

        # Напечатать
        words=cleaned.split()
        for verb in WRITE_VERBS:
            if verb in words:
                idx=words.index(verb)
                text_to_type=" ".join(words[idx+1:]).strip()
                if text_to_type:
                    if _text_corrector_enabled:
                        text_to_type=correct_text(text_to_type)
                    keyboard.write(text_to_type,delay=.01)
                    return T("typing")

        # Перемещение окна
        for verb in MOVE_VERBS:
            if verb in cleaned:
                monitor=2 if any(x in cleaned for x in ["2","второй","два","другий"]) else 1
                target=cleaned.replace(verb,"").strip()
                for w in ["на","в","экран","монитор","1","2","первый","второй","один","два","перший","другий"]:
                    target=target.replace(w," ").strip()
                if target:
                    return (
                        T("moving_window").format(monitor)
                        if move_window_to_monitor(target,monitor)
                        else T("move_failed")
                    )
                return T("move_to_monitor_ask")

        # Запуск приложения
        for verb in APP_VERBS:
            if verb in cleaned:
                target=cleaned.replace(verb,"").strip()
                if target:
                    return (
                        T("app_launching").format(target)
                        if launch_application(target)
                        else T("app_not_found").format(target)
                    )

        # Сайт
        for verb in WEB_VERBS:
            if verb in cleaned:
                target=cleaned.replace(verb,"").strip()
                if target:
                    open_website(target)
                    return T("web_opening").format(target)

        # Закрытие
        for verb in CLOSE_VERBS:
            if verb in cleaned:
                target=cleaned.replace(verb,"").strip()
                if target:
                    return (
                        T("app_closing").format(target)
                        if kill_application(target)
                        else T("app_close_failed").format(target)
                    )

        if cleaned in CONTROL_COMMANDS:
            CONTROL_COMMANDS[cleaned]()
            return T("executed")

        greetings={
            "привет":T("hello_response_ru") if UI_LANGUAGE=="ru" else T("hello_response_ua"),
            "привіт":T("hello_response_ru") if UI_LANGUAGE=="ru" else T("hello_response_ua"),
            "как дела":T("how_are_you"),
            "як справи":T("how_are_you"),
            "ты тут":T("im_here"),
            "ты здесь":T("im_here"),
        }
        for phrase,response in greetings.items():
            if phrase in cleaned:
                return response
        return None

    # --------------------- key ---------------------

    def update_key_info(self):
        # Ключ хранится так же, как в исходной версии.
        return

    def change_key(self):
        if messagebox.askyesno(
            "Смена ключа" if UI_LANGUAGE=="ru" else "Зміна ключа",
            T("key_change_confirm")
        ):
            _clear_saved_key()
            if KeyLoginWindow(self.root).show():
                messagebox.showinfo(
                    "Успешно" if UI_LANGUAGE=="ru" else "Успішно",
                    T("key_success")
                )

    def switch_tab(self,idx):
        self.current_tab=idx
        for i,(bg,txt) in enumerate(self.nav_items):
            if i==idx:
                self.canvas.itemconfig(
                    bg,fill=self.colors["panel2"],outline=self.colors["line"]
                )
                self.canvas.itemconfig(txt,fill=self.colors["text"])
            else:
                self.canvas.itemconfig(bg,fill="",outline="")
                self.canvas.itemconfig(txt,fill=self.colors["muted"])

    def show_temp_status(self,text):
        self.canvas.itemconfig(self.status_sub,text=text)
        self.root.after(
            1600,lambda:self.canvas.itemconfig(
                self.status_sub,text=T("waiting")
            )
        )

    def on_volume_change(self,value):
        set_music_volume(int(float(value)))
        self.update_volume_display()

    def stop_music_click(self):
        stop_music()
        self.update_music_button(False)
        self.add_chat_message(
            "Мита",T("music_stopped_click"),is_mita=True
        )
        speak(T("music_stopped_voice"),force=True)

    def quit(self):
        self.running=False
        try:
            self.audio_monitor.stop()
        except Exception:
            pass
        try:
            stop_music()
        except Exception:
            pass
        try:
            if HAS_EDGE_TTS and _tts_pygame_ready:
                pygame.mixer.music.stop()
                pygame.mixer.quit()
        except Exception:
            pass
        try:
            if _vlc_instance is not None:
                _vlc_instance.release()
        except Exception:
            pass
        try:
            self.root.destroy()
        except Exception:
            pass

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
