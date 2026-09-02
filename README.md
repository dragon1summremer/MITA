import json
import os
import queue
import random
import re
import time
import string
import difflib
import unicodedata
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
import urllib.request
import urllib.parse
import urllib.error
from datetime import datetime, timedelta
from pathlib import Path
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


# ============================================================
# MITA LOCAL APPS — запуск ТОЛЬКО из папки MitaApps, без ИИ
# ============================================================
# Положи сюда .exe или .lnk нужных программ:
#   <папка со скриптом>/MitaApps/
# Можно делать подпапки. Для установленных программ лучше класть ярлыки .lnk,
# потому что простой перенос EXE из папки установленной программы иногда ломает её.
MITA_APPS_DIR = os.path.join(BASE_DIR, "MitaApps")
_MITA_APPS_INDEX = []
_MITA_APPS_INDEX_TIME = 0.0
_MITA_APPS_INDEX_TTL = 2.0

_CYR_TO_LAT = {
    "а":"a","б":"b","в":"v","г":"g","ґ":"g","д":"d","е":"e","ё":"yo","є":"ye",
    "ж":"zh","з":"z","и":"i","і":"i","ї":"yi","й":"y","к":"k","л":"l","м":"m",
    "н":"n","о":"o","п":"p","р":"r","с":"s","т":"t","у":"u","ф":"f","х":"h",
    "ц":"ts","ч":"ch","ш":"sh","щ":"sch","ъ":"","ы":"y","ь":"","э":"e","ю":"yu","я":"ya"
}

# Частые варианты того, как английские звуки попадают в русское распознавание.
_PHONETIC_REPLACEMENTS = (
    ("wallpaper", "volpeyper"), ("wall paper", "volpeyper"),
    ("lively", "liveli"), ("liveley", "liveli"),
    ("discord", "diskord"), ("telegram", "telegram"),
    ("spotify", "spotifay"), ("steam", "stim"),
    ("chrome", "hrom"), ("firefox", "fayrfoks"),
    ("roblox", "robloks"), ("launcher", "loncher"),
)

def _mita_translit(value):
    s = str(value or "").lower()
    return "".join(_CYR_TO_LAT.get(ch, ch) for ch in s)

def _mita_app_norm(value):
    s = str(value or "").lower().strip()
    s = os.path.splitext(os.path.basename(s))[0]
    s = _mita_translit(s)
    for old, new in _PHONETIC_REPLACEMENTS:
        s = s.replace(old, new)
    s = re.sub(r"[^a-z0-9]+", "", s)
    # Небольшая фонетическая нормализация: разные записи похожих звуков
    for old, new in (
        ("ye", "e"), ("yo", "o"), ("yu", "u"), ("ya", "a"),
        ("sch", "sh"), ("zh", "j"), ("ch", "c"), ("ts", "c"),
        ("ph", "f"), ("ck", "k"), ("qu", "kv"), ("w", "v"),
        ("x", "ks"), ("oo", "u"), ("ee", "i"),
    ):
        s = s.replace(old, new)
    # Повторные буквы распознавания не должны мешать: livellly -> lively
    s = re.sub(r"(.)\\1+", r"\\1", s)
    return s

def _mita_bigrams(s):
    if len(s) < 2:
        return {s} if s else set()
    return {s[i:i+2] for i in range(len(s)-1)}

def _mita_similarity(a, b):
    a = _mita_app_norm(a)
    b = _mita_app_norm(b)
    if not a or not b:
        return 0.0
    if a == b:
        return 1.0
    if a in b or b in a:
        shorter = min(len(a), len(b))
        longer = max(len(a), len(b))
        return 0.86 + 0.12 * (shorter / max(1, longer))

    seq = difflib.SequenceMatcher(None, a, b).ratio()
    ag, bg = _mita_bigrams(a), _mita_bigrams(b)
    dice = (2.0 * len(ag & bg) / (len(ag) + len(bg))) if ag and bg else 0.0

    # Сильнее ценим начало слова: "ливили" -> "Lively Wallpaper"
    prefix = 0.0
    lim = min(len(a), len(b), 8)
    same = 0
    for i in range(lim):
        if a[i] == b[i]:
            same += 1
        else:
            break
    if lim:
        prefix = same / lim

    return max(seq, 0.58 * seq + 0.27 * dice + 0.15 * prefix)

def _ensure_mita_apps_dir():
    try:
        os.makedirs(MITA_APPS_DIR, exist_ok=True)
    except Exception as e:
        print(f"[Mita Folder Apps] Не удалось создать папку: {e}")

def _build_mita_apps_index(force=False):
    global _MITA_APPS_INDEX, _MITA_APPS_INDEX_TIME
    _ensure_mita_apps_dir()
    now = time.time()
    if not force and _MITA_APPS_INDEX and now - _MITA_APPS_INDEX_TIME < _MITA_APPS_INDEX_TTL:
        return _MITA_APPS_INDEX

    items = []
    try:
        for root, _, files in os.walk(MITA_APPS_DIR):
            for filename in files:
                ext = os.path.splitext(filename)[1].lower()
                if ext not in (".exe", ".lnk"):
                    continue
                path = os.path.join(root, filename)
                display = os.path.splitext(filename)[0]
                aliases = {display}

                # Имя подпапки тоже участвует в поиске.
                rel_root = os.path.relpath(root, MITA_APPS_DIR)
                if rel_root != ".":
                    aliases.add(os.path.basename(rel_root))
                    aliases.add(os.path.basename(rel_root) + " " + display)

                items.append({
                    "name": display,
                    "path": path,
                    "aliases": list(aliases),
                })
    except Exception as e:
        print(f"[Mita Folder Apps] Ошибка сканирования: {e}")

    _MITA_APPS_INDEX = items
    _MITA_APPS_INDEX_TIME = now
    print(f"[Mita Folder Apps] Найдено приложений: {len(items)} | {MITA_APPS_DIR}")
    return items

def _find_mita_folder_app(spoken_name):
    query = str(spoken_name or "").strip()
    if not query:
        return None, 0.0

    best = None
    best_score = 0.0
    for item in _build_mita_apps_index(force=True):
        score = 0.0
        for alias in item.get("aliases", []):
            score = max(score, _mita_similarity(query, alias))
        if score > best_score:
            best, best_score = item, score

    if best:
        print(f"[Mita Folder Match] {query!r} -> {best['name']!r} | score={best_score:.2f}")
    return best, best_score

def smart_launch_application(target_raw):
    """Запускает приложение ТОЛЬКО из MitaApps. Groq/Steam/реестр не используются."""
    target = str(target_raw or "").strip()
    if not target:
        return False, None

    item, score = _find_mita_folder_app(target)
    # Порог достаточно мягкий для голосовых ошибок, но не запускаем совсем случайные совпадения.
    if not item or score < 0.48:
        play_sound("error")
        print(f"[Mita Folder Apps] Не найдено: {target!r}; best={score:.2f}")
        return False, None

    try:
        os.startfile(item["path"])
        print(f"[Mita Folder Apps] ЗАПУСК: {item['path']}")
        return True, item["name"]
    except Exception as e:
        play_sound("error")
        print(f"[Mita Folder Apps] Ошибка запуска {item['path']}: {e}")
        return False, None

def try_folder_app_voice_command(phrase, interface):
    """Перехватывает команды запуска ДО ИИ-планировщика."""
    text = str(phrase or "").lower().strip()
    verbs = (
        "запусти", "запустить", "включи", "включить", "открой", "открыть",
        "увімкни", "відкрий", "відкрити", "запускай"
    )
    target = ""
    for verb in verbs:
        if text == verb:
            return False
        if text.startswith(verb + " "):
            target = text[len(verb):].strip()
            break
    if not target:
        return False

    item, score = _find_mita_folder_app(target)
    if not item or score < 0.48:
        # Если команда явно "запусти/включи", считаем её обработанной и не даём ИИ
        # угадывать другое приложение. Для "открой" оставляем шанс открыть сайт.
        hard_launch = text.startswith(("запусти ", "запустить ", "включи ", "включить ", "увімкни ", "запускай "))
        if hard_launch:
            msg = T("app_not_found").format(target)
            interface.add_chat_message("Мита", msg, is_mita=True)
            speak(msg, force=True)
            return True
        return False

    try:
        os.startfile(item["path"])
        msg = T("app_launching").format(item["name"])
        interface.add_chat_message("Мита", msg, is_mita=True)
        speak(msg, force=True)
        print(f"[Mita Folder Voice] {target!r} -> {item['name']!r} ({score:.2f})")
        return True
    except Exception as e:
        msg = T("app_not_found").format(target)
        interface.add_chat_message("Мита", msg, is_mita=True)
        speak(msg, force=True)
        print(f"[Mita Folder Voice] Ошибка: {e}")
        return True

# ============================================================
# MITA SMART WEATHER / IP GEOLOCATION ENGINE
# ============================================================
# Работает без API-ключа:
#   1) определяет приблизительный город по публичному IP;
#   2) берёт координаты;
#   3) получает погоду Open-Meteo;
#   4) кэширует результат, чтобы не делать лишние запросы.
# Важно: геолокация по IP приблизительная и может указывать город провайдера/VPN.

_WEATHER_CACHE = {
    "created": 0.0,
    "location": None,
    "weather": None,
    "report_ru": None,
    "report_ua": None,
}
_WEATHER_CACHE_TTL = 600.0

def _http_json(url: str, timeout: float = 6.0):
    req = urllib.request.Request(
        url,
        headers={
            "User-Agent": "MitaDesktopAssistant/4.0",
            "Accept": "application/json",
            "Cache-Control": "no-cache",
        },
        method="GET",
    )
    with urllib.request.urlopen(req, timeout=timeout) as response:
        raw = response.read()
    return json.loads(raw.decode("utf-8", errors="replace"))

def _detect_location_by_ip():
    """Определяет город/координаты по IP с несколькими резервными сервисами."""
    errors = []
    providers = [
        (
            "ipapi",
            "https://ipapi.co/json/",
            lambda d: {
                "city": d.get("city") or "",
                "region": d.get("region") or "",
                "country": d.get("country_name") or d.get("country") or "",
                "latitude": d.get("latitude"),
                "longitude": d.get("longitude"),
                "timezone": d.get("timezone") or "auto",
                "ip": d.get("ip") or "",
                "provider": "ipapi.co",
            },
        ),
        (
            "ipwho",
            "https://ipwho.is/",
            lambda d: {
                "city": d.get("city") or "",
                "region": d.get("region") or "",
                "country": d.get("country") or "",
                "latitude": d.get("latitude"),
                "longitude": d.get("longitude"),
                "timezone": (d.get("timezone") or {}).get("id", "auto") if isinstance(d.get("timezone"), dict) else "auto",
                "ip": d.get("ip") or "",
                "provider": "ipwho.is",
            },
        ),
    ]
    for name, url, parser in providers:
        try:
            data = _http_json(url, timeout=5.0)
            if name == "ipwho" and data.get("success") is False:
                raise RuntimeError(data.get("message") or "IP service returned failure")
            loc = parser(data)
            if loc.get("latitude") is None or loc.get("longitude") is None:
                raise RuntimeError("Нет координат")
            loc["latitude"] = float(loc["latitude"])
            loc["longitude"] = float(loc["longitude"])
            return loc
        except Exception as e:
            errors.append(f"{name}: {e}")
    raise RuntimeError("Не удалось определить город по IP: " + " | ".join(errors[-2:]))

def _weather_code_info(code: int, lang: str = "ru"):
    table_ru = {
        0:("ясно","☀"), 1:("преимущественно ясно","🌤"), 2:("переменная облачность","⛅"), 3:("пасмурно","☁"),
        45:("туман","🌫"), 48:("изморозь и туман","🌫"),
        51:("слабая морось","🌦"), 53:("морось","🌦"), 55:("сильная морось","🌧"),
        56:("слабая ледяная морось","🌧"), 57:("ледяная морось","🌧"),
        61:("небольшой дождь","🌦"), 63:("дождь","🌧"), 65:("сильный дождь","🌧"),
        66:("ледяной дождь","🌧"), 67:("сильный ледяной дождь","🌧"),
        71:("небольшой снег","🌨"), 73:("снег","❄"), 75:("сильный снег","❄"), 77:("снежные зёрна","🌨"),
        80:("небольшие ливни","🌦"), 81:("ливни","🌧"), 82:("сильные ливни","⛈"),
        85:("слабый снегопад","🌨"), 86:("сильный снегопад","❄"),
        95:("гроза","⛈"), 96:("гроза с градом","⛈"), 99:("сильная гроза с градом","⛈"),
    }
    table_ua = {
        0:("ясно","☀"), 1:("переважно ясно","🌤"), 2:("мінлива хмарність","⛅"), 3:("хмарно","☁"),
        45:("туман","🌫"), 48:("паморозь і туман","🌫"),
        51:("слабка мряка","🌦"), 53:("мряка","🌦"), 55:("сильна мряка","🌧"),
        56:("слабка крижана мряка","🌧"), 57:("крижана мряка","🌧"),
        61:("невеликий дощ","🌦"), 63:("дощ","🌧"), 65:("сильний дощ","🌧"),
        66:("крижаний дощ","🌧"), 67:("сильний крижаний дощ","🌧"),
        71:("невеликий сніг","🌨"), 73:("сніг","❄"), 75:("сильний сніг","❄"), 77:("снігові зерна","🌨"),
        80:("невеликі зливи","🌦"), 81:("зливи","🌧"), 82:("сильні зливи","⛈"),
        85:("слабкий снігопад","🌨"), 86:("сильний снігопад","❄"),
        95:("гроза","⛈"), 96:("гроза з градом","⛈"), 99:("сильна гроза з градом","⛈"),
    }
    table = table_ua if lang == "ua" else table_ru
    return table.get(int(code or -1), (("невідома погода" if lang == "ua" else "неизвестная погода"), "◌"))

def _wind_direction_name(deg, lang="ru"):
    try:
        deg = float(deg) % 360
    except Exception:
        return ""
    dirs_ru = ["северный","северо-восточный","восточный","юго-восточный","южный","юго-западный","западный","северо-западный"]
    dirs_ua = ["північний","північно-східний","східний","південно-східний","південний","південно-західний","західний","північно-західний"]
    arr = dirs_ua if lang == "ua" else dirs_ru
    return arr[int((deg + 22.5) // 45) % 8]

def _fetch_open_meteo(lat: float, lon: float):
    params = {
        "latitude": f"{lat:.6f}",
        "longitude": f"{lon:.6f}",
        "current": ",".join([
            "temperature_2m", "apparent_temperature", "relative_humidity_2m",
            "precipitation", "rain", "weather_code", "cloud_cover",
            "surface_pressure", "wind_speed_10m", "wind_direction_10m", "wind_gusts_10m"
        ]),
        "daily": ",".join([
            "weather_code", "temperature_2m_max", "temperature_2m_min",
            "precipitation_probability_max", "wind_speed_10m_max"
        ]),
        "forecast_days": "4",
        "timezone": "auto",
        "wind_speed_unit": "kmh",
    }
    url = "https://api.open-meteo.com/v1/forecast?" + urllib.parse.urlencode(params)
    return _http_json(url, timeout=7.0)

def _format_weather_report(location, data, lang="ru", compact=False):
    cur = data.get("current") or {}
    daily = data.get("daily") or {}
    city = location.get("city") or ("вашем городе" if lang == "ru" else "вашому місті")
    region = location.get("region") or ""
    temp = cur.get("temperature_2m")
    feels = cur.get("apparent_temperature")
    humidity = cur.get("relative_humidity_2m")
    wind = cur.get("wind_speed_10m")
    gust = cur.get("wind_gusts_10m")
    wind_dir = _wind_direction_name(cur.get("wind_direction_10m"), lang)
    clouds = cur.get("cloud_cover")
    pressure = cur.get("surface_pressure")
    precip = cur.get("precipitation")
    desc, icon = _weather_code_info(cur.get("weather_code"), lang)

    def n(v, digits=0, fallback="—"):
        try:
            return f"{float(v):.{digits}f}"
        except Exception:
            return fallback

    if lang == "ua":
        if compact:
            return f"{icon} {city}: {n(temp,0)}°C, {desc}. Відчувається як {n(feels,0)}°C. Вологість {n(humidity)}%. Вітер {n(wind)} км/год."
        report = (
            f"{icon} Зараз у місті {city}" + (f", {region}" if region else "") +
            f": {n(temp,0)}°C, {desc}. Відчувається як {n(feels,0)}°C. "
            f"Вологість {n(humidity)}%, хмарність {n(clouds)}%. "
            f"Вітер {wind_dir} {n(wind)} км/год" + (f", пориви до {n(gust)} км/год" if gust is not None else "") + ". "
            f"Тиск приблизно {n(pressure)} гПа, опади {n(precip,1)} мм."
        )
    else:
        if compact:
            return f"{icon} {city}: {n(temp,0)}°C, {desc}. Ощущается как {n(feels,0)}°C. Влажность {n(humidity)}%. Ветер {n(wind)} км/ч."
        report = (
            f"{icon} Сейчас в городе {city}" + (f", {region}" if region else "") +
            f": {n(temp,0)}°C, {desc}. Ощущается как {n(feels,0)}°C. "
            f"Влажность {n(humidity)}%, облачность {n(clouds)}%. "
            f"Ветер {wind_dir} {n(wind)} км/ч" + (f", порывы до {n(gust)} км/ч" if gust is not None else "") + ". "
            f"Давление примерно {n(pressure)} гПа, осадки {n(precip,1)} мм."
        )

    # Короткий прогноз на завтра — полезная дополнительная фишка.
    try:
        if len(daily.get("time", [])) > 1:
            code = daily.get("weather_code", [None, None])[1]
            ddesc, dicon = _weather_code_info(code, lang)
            tmin = daily.get("temperature_2m_min", [None, None])[1]
            tmax = daily.get("temperature_2m_max", [None, None])[1]
            rainp = daily.get("precipitation_probability_max", [None, None])[1]
            if lang == "ua":
                report += f" Завтра: {dicon} {ddesc}, від {n(tmin,0)}° до {n(tmax,0)}°, імовірність опадів до {n(rainp)}%."
            else:
                report += f" Завтра: {dicon} {ddesc}, от {n(tmin,0)}° до {n(tmax,0)}°, вероятность осадков до {n(rainp)}%."
    except Exception:
        pass
    return report

def get_local_weather(force=False, lang=None):
    lang = lang or ("ua" if UI_LANGUAGE == "ua" else "ru")
    now = time.time()
    if (
        not force and _WEATHER_CACHE.get("location") and _WEATHER_CACHE.get("weather") and
        now - float(_WEATHER_CACHE.get("created") or 0) < _WEATHER_CACHE_TTL
    ):
        report_key = "report_ua" if lang == "ua" else "report_ru"
        report = _WEATHER_CACHE.get(report_key)
        if not report:
            report = _format_weather_report(_WEATHER_CACHE["location"], _WEATHER_CACHE["weather"], lang)
            _WEATHER_CACHE[report_key] = report
        return {"ok": True, "location": _WEATHER_CACHE["location"], "weather": _WEATHER_CACHE["weather"], "report": report, "cached": True}

    try:
        location = _detect_location_by_ip()
        weather = _fetch_open_meteo(location["latitude"], location["longitude"])
        report_ru = _format_weather_report(location, weather, "ru")
        report_ua = _format_weather_report(location, weather, "ua")
        _WEATHER_CACHE.update({
            "created": now, "location": location, "weather": weather,
            "report_ru": report_ru, "report_ua": report_ua,
        })
        return {"ok": True, "location": location, "weather": weather, "report": report_ua if lang == "ua" else report_ru, "cached": False}
    except urllib.error.URLError as e:
        msg = "Нет доступа к интернету для получения погоды." if lang == "ru" else "Немає доступу до інтернету для отримання погоди."
        return {"ok": False, "error": msg, "details": str(e)}
    except Exception as e:
        msg = "Не удалось определить местоположение или получить погоду." if lang == "ru" else "Не вдалося визначити місцезнаходження або отримати погоду."
        return {"ok": False, "error": msg, "details": str(e)}

def is_weather_request(text: str):
    s = str(text or "").lower()
    keys = [
        "погода", "температура на улице", "температура на вулиці",
        "сколько градусов", "скільки градусів", "что на улице", "що надворі",
        "дождь сейчас", "дощ зараз", "какая температура", "яка температура"
    ]
    return any(k in s for k in keys)

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

def _local_text_cleanup(text: str) -> str:
    """Безопасная локальная обработка, если ИИ временно недоступен."""
    if not text:
        return text

    result = str(text).strip()
    result = re.sub(r'[ \t]+', ' ', result)
    result = re.sub(r'\s+([,.;:!?])', r'\1', result)
    result = re.sub(r'([,.;:!?])(?=[^\s\n])', r'\1 ', result)
    result = re.sub(r'\s*\n\s*', '\n', result)
    result = re.sub(r'\n{3,}', '\n\n', result)

    # Не ломаем ссылки, e-mail, код, ID и прочие специальные строки.
    if result and result[0].isalpha():
        result = result[0].upper() + result[1:]

    return result


def _clean_corrector_answer(answer: str) -> str:
    """Убирает служебный текст/markdown, если модель всё же его добавила."""
    if not answer:
        return ""

    result = str(answer).strip()

    # Удаляем markdown-кодовые блоки, но сохраняем их содержимое.
    if result.startswith("```") and result.endswith("```"):
        result = re.sub(r'^```[a-zA-Zа-яА-ЯёЁіїєґІЇЄҐ0-9_-]*\s*', '', result)
        result = re.sub(r'\s*```$', '', result)

    prefixes = [
        "Вот исправленный текст:", "Исправленный текст:", "Исправленный вариант:",
        "Вот исправленный вариант:", "Ось виправлений текст:", "Виправлений текст:",
        "Виправлений варіант:", "Ось виправлений варіант:", "Corrected text:"
    ]
    lowered = result.lower()
    for prefix in prefixes:
        if lowered.startswith(prefix.lower()):
            result = result[len(prefix):].strip()
            break

    # Убираем только внешние кавычки; кавычки внутри текста не трогаем.
    if len(result) >= 2 and result[0] == result[-1] and result[0] in ('"', "'"):
        result = result[1:-1].strip()

    return result.strip()


def _language_score(text: str):
    """Простая проверка RU/UA для контроля результата перевода."""
    s = " " + re.sub(r"[^а-яёіїєґ\\s'-]", " ", str(text or "").lower()) + " "

    ua_score = 0
    ru_score = 0

    # Буквы, однозначно указывающие на язык.
    ua_score += sum(s.count(ch) * 5 for ch in "іїєґ")
    ru_score += sum(s.count(ch) * 5 for ch in "ыэёъ")

    ua_words = [
        " що ", " як ", " це ", " ти ", " він ", " вона ", " вони ", " ми ",
        " сьогодні ", " вчора ", " завтра ", " робив ", " робила ", " робити ",
        " привіт ", " дякую ", " будь ласка ", " добре ", " дуже ", " мене ",
        " тебе ", " твій ", " твоя ", " твого ", " зараз ", " можна ", " треба "
    ]
    ru_words = [
        " что ", " как ", " это ", " ты ", " он ", " она ", " они ", " мы ",
        " сегодня ", " вчера ", " завтра ", " делал ", " делала ", " делать ",
        " привет ", " спасибо ", " пожалуйста ", " хорошо ", " очень ", " меня ",
        " тебя ", " твой ", " твоя ", " твоего ", " сейчас ", " можно ", " нужно "
    ]
    for w in ua_words:
        if w in s:
            ua_score += 3
    for w in ru_words:
        if w in s:
            ru_score += 3

    return ru_score, ua_score


def _is_wrong_target_language(text: str, target_lang: str, source_text: str = "") -> bool:
    """True, если результат явно остался на неправильном языке."""
    ru, ua = _language_score(text)
    src_ru, src_ua = _language_score(source_text)

    if target_lang == "ru":
        # Любые украинские уникальные буквы — недопустимы для русского результата.
        if any(ch in str(text).lower() for ch in "іїєґ"):
            return True
        if ua > ru and ua >= 3:
            return True
        # Если исходник явно украинский и ответ почти не изменился — перевод не выполнен.
        if src_ua > src_ru and str(text).strip().lower() == str(source_text).strip().lower():
            return True
    else:
        # Русские уникальные буквы недопустимы для украинского результата.
        if any(ch in str(text).lower() for ch in "ыэёъ"):
            return True
        if ru > ua and ru >= 3:
            return True
        if src_ru > src_ua and str(text).strip().lower() == str(source_text).strip().lower():
            return True

    return False


def _finalize_ai_text(text: str, target_lang: str) -> str:
    """Финальная страховка после ИИ: заглавная буква и конечный знак препинания."""
    result = str(text or "").strip()
    if not result:
        return result

    # Заглавная первая буквенная буква, не ломая emoji/кавычки/скобки в начале.
    chars = list(result)
    for i, ch in enumerate(chars):
        if ch.isalpha():
            chars[i] = ch.upper()
            break
    result = "".join(chars)

    # Если ИИ забыл конечный знак, определяем вопрос по типичным вопросительным словам.
    if result and result[-1] not in '.!?…':
        lowered = result.lower().lstrip('«"\'([{ ')
        question_words_ru = (
            'что ', 'кто ', 'как ', 'где ', 'куда ', 'откуда ', 'когда ',
            'почему ', 'зачем ', 'сколько ', 'какой ', 'какая ', 'какие ',
            'чей ', 'чья ', 'можно ли ', 'ты ', 'вы '
        )
        question_words_ua = (
            'що ', 'хто ', 'як ', 'де ', 'куди ', 'звідки ', 'коли ',
            'чому ', 'навіщо ', 'скільки ', 'який ', 'яка ', 'які ',
            'чий ', 'чия ', 'можна ', 'ти ', 'ви '
        )
        words = question_words_ua if target_lang == 'ua' else question_words_ru
        result += '?' if lowered.startswith(words) else '.'

    return result


def _process_text_for_language(text: str, correct_errors: bool = True) -> str:
    """
    ЖЁСТКАЯ языковая логика:
    - выбран RU -> печатаем ТОЛЬКО по-русски;
    - выбран UA -> печатаем ТОЛЬКО по-украински;
    - если вход на другом языке, обязательно переводим;
    - результат проверяется, и при неверном языке перевод повторяется.
    """
    original = str(text or "").strip()
    if not original:
        return original

    fallback = _local_text_cleanup(original)
    target_lang = "ua" if get_ui_language() == "ua" else "ru"
    target_name = "УКРАИНСКИЙ" if target_lang == "ua" else "РУССКИЙ"

    if client is None:
        print("[Язык текста] Groq недоступен — обязательный перевод невозможен")
        return fallback

    correction = (
        "ОБЯЗАТЕЛЬНО исправь орфографию, грамматику, пунктуацию, регистр букв и очевидные ошибки распознавания речи. "
        "Каждое предложение должно начинаться с заглавной буквы и заканчиваться подходящим знаком: точкой, вопросительным или восклицательным знаком. "
        "Если фраза по смыслу является вопросом, обязательно поставь знак вопроса."
        if correct_errors else
        "Сохрани смысл и стиль, но всё равно оформи текст грамотно: нормальный регистр и конечная пунктуация."
    )

    # Несколько попыток: если модель вдруг оставила исходный язык, мы это обнаружим.
    for attempt in range(3):
        strict_note = ""
        if attempt > 0:
            strict_note = f"\nПРЕДЫДУЩАЯ ПОПЫТКА БЫЛА НА НЕВЕРНОМ ЯЗЫКЕ. ПЕРЕВЕДИ ВЕСЬ ТЕКСТ НА {target_name}."

        system_prompt = f"""Ты выполняешь только перевод/коррекцию текста для голосового ассистента.

КРИТИЧЕСКОЕ ТРЕБОВАНИЕ:
ФИНАЛЬНЫЙ РЕЗУЛЬТАТ ДОЛЖЕН БЫТЬ ТОЛЬКО НА ЯЗЫКЕ: {target_name}.

Если исходный текст на другом языке — ОБЯЗАТЕЛЬНО ПЕРЕВЕДИ ВЕСЬ ТЕКСТ НА {target_name}.
Если исходный текст уже на {target_name} — сохрани этот язык.
{correction}
Сохрани смысл, тон, имена, числа, URL, e-mail, ID, никнеймы и emoji.
Не отвечай на содержание текста. Не добавляй объяснений.
ВАЖНО: итог должен выглядеть как полностью грамотный готовый текст: первая буква предложения — заглавная, в конце — правильный знак препинания.
Если это вопрос по смыслу — в конце ОБЯЗАТЕЛЬНО должен быть знак вопроса.
Верни ТОЛЬКО готовый текст без кавычек, markdown и префиксов.{strict_note}"""

        try:
            completion = client.chat.completions.create(
                model="openai/gpt-oss-120b",
                messages=[
                    {"role": "system", "content": system_prompt},
                    {"role": "user", "content": original}
                ],
                temperature=0.0,
                max_completion_tokens=1024,
                top_p=1,
                reasoning_effort="low",
                stream=False
            )
            result = _clean_corrector_answer(completion.choices[0].message.content)
            if not result:
                continue

            result = _finalize_ai_text(result, target_lang)

            if len(result) > max(len(original) * 4, len(original) + 300):
                print("[Язык текста] Ответ отклонён: слишком длинный")
                continue

            if _is_wrong_target_language(result, target_lang, original):
                print(f"[Язык текста] Попытка {attempt + 1}: модель оставила неверный язык, повторяю...")
                continue

            print(f"[Язык текста] target={target_lang} | {original!r} -> {result!r}")
            return result.strip()

        except Exception as e:
            print(f"[Язык текста] Попытка {attempt + 1}, ошибка Groq: {e}")
            time.sleep(0.15)

    # Последняя сверхстрогая попытка отдельным запросом-переводом.
    try:
        final_prompt = (
            f"Переведи следующий текст ПОЛНОСТЬЮ на {'украинский' if target_lang == 'ua' else 'русский'} язык. "
            "Одновременно исправь грамматику, орфографию, регистр и пунктуацию. "
            "Начни предложение с заглавной буквы. Если это вопрос — обязательно поставь знак вопроса. "
            "Верни только полностью готовый текст, без пояснений:\n\n" + original
        )
        completion = client.chat.completions.create(
            model="openai/gpt-oss-120b",
            messages=[{"role": "user", "content": final_prompt}],
            temperature=0.0,
            max_completion_tokens=1024,
            top_p=1,
            stream=False
        )
        result = _clean_corrector_answer(completion.choices[0].message.content)
        result = _finalize_ai_text(result, target_lang) if result else result
        if result and not _is_wrong_target_language(result, target_lang, original):
            print(f"[Язык текста] строгий перевод target={target_lang}: {result!r}")
            return result.strip()
    except Exception as e:
        print(f"[Язык текста] Финальная ошибка перевода: {e}")

    # Не выдаём заведомо неправильный язык как будто всё успешно.
    print(f"[Язык текста] Не удалось гарантировать язык {target_lang}. Оставлен исходный текст.")
    return fallback


def correct_text(text: str) -> str:
    """
    Исправляет текст и одновременно приводит его к выбранному языку интерфейса.
    RU -> всегда русский.
    UA -> всегда украинский.
    """
    return _process_text_for_language(text, correct_errors=True)


def prepare_text_for_typing(text: str) -> str:
    """
    Финальная подготовка текста перед печатью.

    Даже если исправитель выключен, выбранный язык всё равно соблюдается:
    - русский режим -> русский текст;
    - украинский режим -> украинский текст.

    Исправление орфографии/грамматики зависит от переключателя исправителя.
    """
    return _process_text_for_language(
        text,
        correct_errors=_text_corrector_enabled
    )


def _get_clipboard_unicode():
    """Читает текстовый буфер Windows. Возвращает None, если прочитать не удалось."""
    try:
        import ctypes
        from ctypes import wintypes

        CF_UNICODETEXT = 13
        user32 = ctypes.windll.user32
        kernel32 = ctypes.windll.kernel32

        # ВАЖНО: на 64-битной Windows все HANDLE/HGLOBAL должны быть
        # объявлены как указатели. Иначе ctypes по умолчанию использует c_int
        # и может выбросить OverflowError: int too long to convert.
        user32.OpenClipboard.argtypes = [wintypes.HWND]
        user32.OpenClipboard.restype = wintypes.BOOL
        user32.CloseClipboard.argtypes = []
        user32.CloseClipboard.restype = wintypes.BOOL
        user32.GetClipboardData.argtypes = [wintypes.UINT]
        user32.GetClipboardData.restype = wintypes.HANDLE

        kernel32.GlobalLock.argtypes = [wintypes.HGLOBAL]
        kernel32.GlobalLock.restype = wintypes.LPVOID
        kernel32.GlobalUnlock.argtypes = [wintypes.HGLOBAL]
        kernel32.GlobalUnlock.restype = wintypes.BOOL

        if not user32.OpenClipboard(None):
            return None
        try:
            handle = user32.GetClipboardData(CF_UNICODETEXT)
            if not handle:
                return None
            ptr = kernel32.GlobalLock(handle)
            if not ptr:
                return None
            try:
                return ctypes.wstring_at(ptr)
            finally:
                kernel32.GlobalUnlock(handle)
        finally:
            user32.CloseClipboard()
    except Exception:
        return None


def _set_clipboard_unicode(value: str) -> bool:
    """Надёжно помещает Unicode-текст в буфер Windows без сторонних библиотек."""
    try:
        import ctypes
        from ctypes import wintypes

        CF_UNICODETEXT = 13
        GMEM_MOVEABLE = 0x0002

        user32 = ctypes.windll.user32
        kernel32 = ctypes.windll.kernel32

        # Полные сигнатуры WinAPI обязательны для x64 Windows.
        user32.OpenClipboard.argtypes = [wintypes.HWND]
        user32.OpenClipboard.restype = wintypes.BOOL
        user32.CloseClipboard.argtypes = []
        user32.CloseClipboard.restype = wintypes.BOOL
        user32.EmptyClipboard.argtypes = []
        user32.EmptyClipboard.restype = wintypes.BOOL
        user32.SetClipboardData.argtypes = [wintypes.UINT, wintypes.HANDLE]
        user32.SetClipboardData.restype = wintypes.HANDLE

        kernel32.GlobalAlloc.argtypes = [wintypes.UINT, ctypes.c_size_t]
        kernel32.GlobalAlloc.restype = wintypes.HGLOBAL
        kernel32.GlobalLock.argtypes = [wintypes.HGLOBAL]
        kernel32.GlobalLock.restype = wintypes.LPVOID
        kernel32.GlobalUnlock.argtypes = [wintypes.HGLOBAL]
        kernel32.GlobalUnlock.restype = wintypes.BOOL
        kernel32.GlobalFree.argtypes = [wintypes.HGLOBAL]
        kernel32.GlobalFree.restype = wintypes.HGLOBAL

        data = (str(value) + "\0").encode("utf-16-le")
        handle = kernel32.GlobalAlloc(GMEM_MOVEABLE, len(data))
        if not handle:
            return False

        ptr = kernel32.GlobalLock(handle)
        if not ptr:
            kernel32.GlobalFree(handle)
            return False

        ctypes.memmove(ptr, data, len(data))
        kernel32.GlobalUnlock(handle)

        opened = False
        for _ in range(12):
            if user32.OpenClipboard(None):
                opened = True
                break
            time.sleep(0.02)

        if not opened:
            kernel32.GlobalFree(handle)
            return False

        try:
            if not user32.EmptyClipboard():
                kernel32.GlobalFree(handle)
                return False

            # После успешного SetClipboardData память принадлежит Windows.
            if not user32.SetClipboardData(CF_UNICODETEXT, handle):
                kernel32.GlobalFree(handle)
                return False
            handle = None
            return True
        finally:
            user32.CloseClipboard()
    except Exception as e:
        print(f"[Буфер] Ошибка: {e}")
        return False


def type_unicode_text(text: str, restore_clipboard: bool = True) -> bool:
    """
    Печатает любой Unicode-текст в активное окно через буфер + Ctrl+V.
    В отличие от keyboard.write(), корректно работает с русским/украинским.
    """
    value = str(text or "")
    if not value:
        return False

    old_clipboard = _get_clipboard_unicode() if restore_clipboard else None

    if not _set_clipboard_unicode(value):
        # Fallback для ASCII, если Windows clipboard неожиданно недоступен.
        try:
            if value.isascii():
                keyboard.write(value, delay=0.01)
                return True
        except Exception:
            pass
        return False

    try:
        time.sleep(0.05)
        keyboard.press_and_release("ctrl+v")
        time.sleep(0.12)
        return True
    finally:
        if restore_clipboard and old_clipboard is not None:
            _set_clipboard_unicode(old_clipboard)

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
    _ensure_mita_apps_dir()
    _build_mita_apps_index(force=True)
    print(f"📁 Папка приложений Mita: {MITA_APPS_DIR}")

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

        if interface:
            interface.set_now_playing(title)
            interface.update_music_button(True)

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
                    if interface:
                        try:
                            interface.update_music_progress(player.get_time(), player.get_length())
                        except Exception:
                            pass
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
# MITA ULTRA INTELLIGENCE CORE
# ============================================================

_MITA_MEMORY_FILE = os.path.join(BASE_DIR, "mita_ai_memory.json")
_MITA_DIALOG_MEMORY = []
_MITA_APP_INDEX = None
_MITA_APP_INDEX_TIME = 0.0

def _mita_load_memory():
    global _MITA_DIALOG_MEMORY
    try:
        if os.path.exists(_MITA_MEMORY_FILE):
            data = json.loads(Path(_MITA_MEMORY_FILE).read_text(encoding="utf-8"))
            if isinstance(data, list):
                _MITA_DIALOG_MEMORY = data[-30:]
    except Exception:
        _MITA_DIALOG_MEMORY = []

def _mita_save_memory():
    try:
        Path(_MITA_MEMORY_FILE).write_text(
            json.dumps(_MITA_DIALOG_MEMORY[-30:], ensure_ascii=False, indent=2),
            encoding="utf-8"
        )
    except Exception:
        pass

def _mita_remember(role, content):
    content = str(content or "").strip()
    if not content:
        return
    _MITA_DIALOG_MEMORY.append({"role": role, "content": content[:5000]})
    del _MITA_DIALOG_MEMORY[:-30]
    _mita_save_memory()

def _norm_app_name(value):
    s = unicodedata.normalize("NFKC", str(value or "")).lower().strip()
    s = re.sub(r"\.(exe|lnk|url)$", "", s)
    s = s.replace("_", " ").replace("-", " ")
    s = re.sub(r"[^0-9a-zа-яёіїєґ+.# ]+", " ", s, flags=re.I)
    s = re.sub(r"\s+", " ", s).strip()
    replacements = {
        "дискорд":"discord", "дискордд":"discord", "discord":"discord",
        "телега":"telegram", "телеграмм":"telegram", "телеграм":"telegram",
        "telegram":"telegram", "стим":"steam", "steam":"steam",
        "хром":"chrome", "гугл хром":"chrome", "google chrome":"chrome",
        "браузер":"chrome", "роблокс":"roblox", "roblox":"roblox",
        "спотифай":"spotify", "spotify":"spotify", "обс":"obs", "обс студио":"obs", "obs studio":"obs",
        "бс":"bluestacks", "блюстакс":"bluestacks", "блустакс":"bluestacks",
        "blue stacks":"bluestacks", "bluestacks":"bluestacks",
        "кс":"cs2", "кс 2":"cs2", "counter strike 2":"cs2",
        "дота":"dota 2", "dota2":"dota 2", "дота 2":"dota 2",
        "калькулятор":"calculator", "блокнот":"notepad", "проводник":"explorer",
        "диспетчер задач":"task manager", "параметры":"settings"
    }
    return replacements.get(s, s)

def _build_smart_app_index(force=False):
    global _MITA_APP_INDEX, _MITA_APP_INDEX_TIME
    now = time.time()
    if _MITA_APP_INDEX is not None and not force and now - _MITA_APP_INDEX_TIME < 600:
        return _MITA_APP_INDEX

    items = []
    seen = set()

    def add(name, path, kind="file"):
        if not name or not path:
            return
        key = (str(path).lower(), _norm_app_name(name))
        if key in seen:
            return
        seen.add(key)
        items.append({"name": str(name), "norm": _norm_app_name(name), "path": str(path), "kind": kind})

    # Системные приложения Windows.
    system_apps = {
        "calculator": "calc.exe", "калькулятор": "calc.exe",
        "notepad": "notepad.exe", "блокнот": "notepad.exe",
        "explorer": "explorer.exe", "проводник": "explorer.exe",
        "task manager": "taskmgr.exe", "диспетчер задач": "taskmgr.exe",
        "paint": "mspaint.exe", "ножницы": "snippingtool.exe",
        "cmd": "cmd.exe", "командная строка": "cmd.exe",
        "powershell": "powershell.exe"
    }
    for n, command in system_apps.items():
        add(n, command, "command")

    # Уже известные приложения из базы Миты.
    for alias, app_key in APP_TRANSLIT_MAP.items():
        exe = APP_EXE_MAP.get(app_key)
        if exe:
            add(alias, exe, "command")
            add(app_key, exe, "command")
    for app_key, exe in APP_EXE_MAP.items():
        add(app_key, exe, "command")

    # Ярлыки Start Menu — это самый надёжный способ узнать реальное имя программы.
    start_dirs = [
        os.path.join(os.environ.get("APPDATA", ""), "Microsoft", "Windows", "Start Menu", "Programs"),
        os.path.join(os.environ.get("PROGRAMDATA", ""), "Microsoft", "Windows", "Start Menu", "Programs"),
        os.path.join(os.environ.get("USERPROFILE", ""), "Desktop"),
        os.path.join(os.environ.get("PUBLIC", r"C:\Users\Public"), "Desktop"),
    ]
    for base in start_dirs:
        if not base or not os.path.isdir(base):
            continue
        try:
            for root, dirs, files in os.walk(base):
                dirs[:] = [d for d in dirs if not d.startswith(".")]
                for fn in files:
                    if fn.lower().endswith((".lnk", ".url", ".exe")):
                        add(os.path.splitext(fn)[0], os.path.join(root, fn), "file")
        except Exception:
            pass

    # App Paths из реестра.
    registry_roots = [
        (winreg.HKEY_CURRENT_USER, r"SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths"),
        (winreg.HKEY_LOCAL_MACHINE, r"SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths"),
    ]
    for hive, root_path in registry_roots:
        try:
            with winreg.OpenKey(hive, root_path) as root:
                count, _, _ = winreg.QueryInfoKey(root)
                for i in range(count):
                    try:
                        sub = winreg.EnumKey(root, i)
                        with winreg.OpenKey(root, sub) as k:
                            path, _ = winreg.QueryValueEx(k, None)
                            if path:
                                add(os.path.splitext(sub)[0], str(path).strip('"'), "file")
                    except Exception:
                        pass
        except Exception:
            pass

    _MITA_APP_INDEX = items
    _MITA_APP_INDEX_TIME = now
    return items

def _smart_app_match(query):
    q = _norm_app_name(query)
    if not q:
        return None, 0.0
    items = _build_smart_app_index()
    best = None
    best_score = 0.0
    q_tokens = set(q.split())
    for item in items:
        n = item["norm"]
        if not n:
            continue
        if q == n:
            score = 1.0
        elif q in n or n in q:
            score = 0.92 if min(len(q), len(n)) >= 3 else 0.75
        else:
            seq = difflib.SequenceMatcher(None, q, n).ratio()
            nt = set(n.split())
            token = len(q_tokens & nt) / max(1, len(q_tokens | nt))
            score = max(seq, token * 0.92)
        if score > best_score:
            best, best_score = item, score
    return best, best_score

def _launch_index_item(item):
    """Запускает только реально найденный элемент индекса."""
    if not item:
        return False
    try:
        path = str(item.get("path") or "").strip()
        kind = item.get("kind")
        if not path:
            return False

        if kind == "command":
            # Для встроенных Windows-команд допускаем запуск по имени.
            subprocess.Popen(path, shell=False, stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
        else:
            # Ярлык / exe / url существует на диске.
            if not os.path.exists(path):
                return False
            os.startfile(path)
        return True
    except Exception as e:
        print(f"[Mita App] Ошибка запуска найденного приложения: {e}")
        return False


def _ai_resolve_app_identity(target_raw):
    """
    ИИ превращает разговорное/ошибочно распознанное название программы
    в нормальное имя и вероятные имена exe.
    НИЧЕГО не запускает — только возвращает подсказки для локального поиска.
    """
    target = str(target_raw or "").strip()
    if not target or client is None:
        return None

    prompt = f"""
Ты — модуль распознавания названий Windows-приложений.
Пользователь голосом произнёс название приложения: {target!r}

Нужно понять, какую ПРОГРАММУ он имеет в виду, даже если:
- название сказано по-русски/украински/английски;
- это транслит или фонетическая запись;
- есть ошибка распознавания речи;
- используется сокращение или сленг.

Примеры:
"обс", "о б с", "obs" -> OBS Studio, exe обычно obs64.exe.
"бс", "блюстакс", "blue stacks" -> BlueStacks, exe может быть HD-Player.exe / BlueStacks.exe.
"дс", "дискордик" -> Discord, exe Discord.exe.
"тг", "телега" -> Telegram, exe Telegram.exe.
"спотик" -> Spotify, exe Spotify.exe.
"хром", "chrome" -> Google Chrome, exe chrome.exe.
"яндекс", "яндекс браузер" -> Yandex Browser, exe browser.exe.
"опера", "opera" -> Opera Browser, exe opera.exe / launcher.exe.
"фаерфокс", "firefox" -> Mozilla Firefox, exe firefox.exe.
"эдж", "edge" -> Microsoft Edge, exe msedge.exe.
"кс", "кс два" -> Counter-Strike 2, exe cs2.exe.
"гта пять" -> Grand Theft Auto V, возможны GTA5.exe / PlayGTAV.exe.
"майнкрафт" -> Minecraft / Minecraft Launcher, найди наиболее вероятный Windows launcher.

Верни ТОЛЬКО JSON без markdown:
{{
  "app_name": "официальное или наиболее вероятное название",
  "search_names": ["вариант 1", "вариант 2", "вариант 3"],
  "exe_candidates": ["program.exe", "another.exe"],
  "confidence": 0.0
}}

Правила:
- Не придумывай shell-команды, аргументы запуска, URL или пути.
- exe_candidates должны содержать только имена файлов *.exe.
- Браузер, игра, лаунчер, мессенджер, редактор или любая другая установленная программа — это приложение.
- Если пользователь назвал браузер (Chrome, Opera, Yandex, Firefox, Edge и т.п.), верни именно приложение браузера.
- Если сокращение неоднозначно, выбери наиболее вероятную известную программу, но снизь confidence.
- Максимум 6 search_names и 6 exe_candidates.
"""
    try:
        completion = client.chat.completions.create(
            model="openai/gpt-oss-120b",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.0,
            max_completion_tokens=300,
            top_p=1,
            reasoning_effort="medium",
            stream=False
        )
        obj = _extract_json_object(completion.choices[0].message.content)
        if not isinstance(obj, dict):
            return None

        app_name = str(obj.get("app_name") or "").strip()
        search_names = obj.get("search_names") or []
        exe_candidates = obj.get("exe_candidates") or []

        if not isinstance(search_names, list):
            search_names = []
        if not isinstance(exe_candidates, list):
            exe_candidates = []

        safe_names = []
        for x in search_names[:6]:
            x = str(x or "").strip()
            if x and len(x) <= 100:
                safe_names.append(x)

        safe_exes = []
        for x in exe_candidates[:6]:
            x = os.path.basename(str(x or "").strip())
            if re.fullmatch(r"(?i)[a-z0-9_. +()-]{1,100}\.exe", x):
                safe_exes.append(x)

        try:
            confidence = max(0.0, min(1.0, float(obj.get("confidence", 0.0))))
        except Exception:
            confidence = 0.0

        return {
            "app_name": app_name,
            "search_names": safe_names,
            "exe_candidates": safe_exes,
            "confidence": confidence,
        }
    except Exception as e:
        print(f"[Mita App AI] Ошибка определения приложения: {e}")
        return None


def _find_and_launch_exe_candidate(exe_name, cache_key=None):
    """Ищет конкретный exe и запускает только если путь действительно найден."""
    exe_name = os.path.basename(str(exe_name or "").strip())
    if not re.fullmatch(r"(?i)[a-z0-9_. +()-]{1,100}\.exe", exe_name):
        return False, None

    try:
        found_path = find_exe_fast_registry(exe_name)
    except Exception:
        found_path = None

    if not found_path:
        try:
            found_path = search_exe_on_disks(exe_name)
        except Exception as e:
            print(f"[Mita App] Ошибка поиска {exe_name}: {e}")
            found_path = None

    if not found_path or not os.path.isfile(found_path):
        return False, None

    try:
        os.startfile(found_path)
        if cache_key:
            try:
                cache = load_app_cache()
                cache[str(cache_key)] = found_path
                save_app_cache(cache)
            except Exception:
                pass
        return True, found_path
    except Exception as e:
        print(f"[Mita App] Не удалось запустить {found_path}: {e}")
        return False, None


def _legacy_smart_launch_application_ai(target_raw):
    """
    AI-FIRST запуск приложения.

    Любое название сначала отправляется в ИИ. ИИ определяет, что пользователь
    имел в виду, и возвращает только безопасные поисковые подсказки: название
    программы и возможные *.exe. Уже после этого Mita ищет РЕАЛЬНЫЙ ярлык/EXE
    на компьютере и только тогда запускает его.

    Если ИИ недоступен, остаётся локальный fallback по ярлыкам/реестру/алиасам.
    """
    target = str(target_raw or "").strip()
    if not target:
        return False, None

    print(f"[Mita App AI-FIRST] Запрос: {target!r}")

    # 1) ВСЕГДА сначала спрашиваем ИИ, что это за приложение.
    ai = _ai_resolve_app_identity(target)
    if ai:
        print(
            f"[Mita App AI] {target!r} -> {ai.get('app_name')!r}, "
            f"names={ai.get('search_names')}, exe={ai.get('exe_candidates')}, "
            f"confidence={ai.get('confidence')}"
        )

        search_names = []
        for name in [ai.get("app_name"), *(ai.get("search_names") or []), target]:
            name = str(name or "").strip()
            if name and name not in search_names:
                search_names.append(name)

        # Обновляем индекс, чтобы видеть недавно установленные программы.
        _build_smart_app_index(force=True)

        # Сначала реальные ярлыки / App Paths / Desktop / Start Menu.
        best_item = None
        best_score = 0.0
        for name in search_names:
            candidate, candidate_score = _smart_app_match(name)
            if candidate and candidate_score > best_score:
                best_item, best_score = candidate, candidate_score

        # Для уверенного AI-ответа допускаем чуть более мягкое совпадение.
        min_score = 0.56 if float(ai.get("confidence") or 0.0) >= 0.72 else 0.64
        if best_item and best_score >= min_score:
            if _launch_index_item(best_item):
                print(f"[Mita App] Запущено по AI имени: {best_item['name']} ({best_score:.2f})")
                return True, ai.get("app_name") or best_item["name"]

        # Затем проверяем EXE-кандидаты от ИИ. Никаких shell-команд от модели:
        # имя валидируется, а файл обязательно реально ищется на этом ПК.
        for exe in ai.get("exe_candidates") or []:
            ok, found = _find_and_launch_exe_candidate(
                exe,
                cache_key=_norm_app_name(ai.get("app_name") or target)
            )
            if ok:
                print(f"[Mita App] Запущен найденный EXE: {found}")
                return True, ai.get("app_name") or os.path.basename(found)

    # 2) Fallback, если ИИ временно недоступен/не угадал.
    # Локальный индекс всё равно умеет находить реальные установленные программы.
    item, score = _smart_app_match(target)
    if item and score >= 0.66 and _launch_index_item(item):
        print(f"[Mita App] Fallback: {item['name']} ({score:.2f})")
        return True, item["name"]

    # 3) Старые известные алиасы — только с проверкой существования EXE.
    target_clean = target.lower().strip()
    app_key = APP_TRANSLIT_MAP.get(target_clean, target_clean)
    exe_name = APP_EXE_MAP.get(app_key)
    if exe_name:
        try:
            cache = load_app_cache()
            cached_path = cache.get(app_key)
            if cached_path and os.path.isfile(cached_path):
                os.startfile(cached_path)
                return True, app_key
        except Exception:
            pass

        ok, found = _find_and_launch_exe_candidate(exe_name, cache_key=app_key)
        if ok:
            return True, app_key

    play_sound("error")
    print(f"[Mita App] Не найдено на ПК: {target!r}")
    return False, None

def _extract_json_object(text):
    s = str(text or "").strip()
    s = re.sub(r"^```(?:json)?\s*", "", s, flags=re.I)
    s = re.sub(r"\s*```$", "", s)
    start, end = s.find("{"), s.rfind("}")
    if start < 0 or end <= start:
        return None
    try:
        return json.loads(s[start:end+1])
    except Exception:
        return None

# ============================================================
# ПОДТВЕРЖДЕНИЕ ЗАПУСКА ПРИЛОЖЕНИЙ / ИГР
# ============================================================
_PENDING_APP_LAUNCH = None
_PENDING_APP_LAUNCH_TIMEOUT = 30.0


def _set_pending_app_launch(spoken_target, resolved_name=None):
    """Запоминает приложение, которое Мита предложила запустить."""
    global _PENDING_APP_LAUNCH
    resolved = str(resolved_name or spoken_target or "").strip()
    spoken = str(spoken_target or resolved).strip()
    _PENDING_APP_LAUNCH = {
        "spoken_target": spoken,
        "resolved_name": resolved,
        "created_at": time.time(),
    }
    return resolved


def _clear_pending_app_launch():
    global _PENDING_APP_LAUNCH
    _PENDING_APP_LAUNCH = None


def _get_pending_app_launch():
    global _PENDING_APP_LAUNCH
    if not _PENDING_APP_LAUNCH:
        return None
    if time.time() - float(_PENDING_APP_LAUNCH.get("created_at", 0)) > _PENDING_APP_LAUNCH_TIMEOUT:
        _PENDING_APP_LAUNCH = None
        return None
    return _PENDING_APP_LAUNCH


def _handle_pending_app_confirmation(phrase, interface):
    """
    Обрабатывает короткий ответ пользователя после вопроса
    «Хотите запустить ...?».
    Возвращает True, если фраза была ответом на подтверждение.
    """
    pending = _get_pending_app_launch()
    if not pending:
        return False

    text = str(phrase or "").lower().strip()
    words = set(re.findall(r"[а-яёіїєґa-z0-9]+", text, flags=re.I))

    yes_words = {
        "да", "ага", "угу", "конечно", "запускай", "запусти", "включай", "включи",
        "так", "авжеж", "звісно", "запускай", "запусти", "вмикай", "увімкни", "ок", "okay"
    }
    no_words = {
        "нет", "не", "отмена", "отмени", "ненадо", "стоп", "неа",
        "ні", "скасуй", "скасувати", "досить"
    }

    positive = bool(words & yes_words) or text in {"давай", "можно", "добро", "точно"}
    negative = bool(words & no_words) or "не надо" in text or "не запускай" in text or "не потрібно" in text

    if negative:
        name = pending.get("resolved_name") or pending.get("spoken_target") or "приложение"
        _clear_pending_app_launch()
        msg = f"Хорошо, не запускаю {name}." if UI_LANGUAGE == "ru" else f"Добре, не запускаю {name}."
        interface.add_chat_message("Мита", msg, is_mita=True)
        speak(msg, force=True)
        return True

    if positive:
        name = pending.get("resolved_name") or pending.get("spoken_target") or ""
        _clear_pending_app_launch()
        msg = f"Запускаю {name}." if UI_LANGUAGE == "ru" else f"Запускаю {name}."
        interface.add_chat_message("Мита", msg, is_mita=True)
        speak(msg, force=True)

        # Сам поиск может быть долгим, поэтому запускаем его отдельно от UI-потока.
        def _do_launch():
            ok, matched = smart_launch_application(name)
            if not ok:
                fail = T("app_not_found").format(name)
                try:
                    interface.root.after(0, lambda: interface.add_chat_message("Мита", fail, is_mita=True))
                except Exception:
                    pass
                speak(fail, force=True)

        threading.Thread(target=_do_launch, daemon=True).start()
        return True

    # Если пользователь вместо «да/нет» сказал новую полноценную команду,
    # старое подтверждение не должно мешать.
    command_hints = {
        "запусти", "открой", "включи", "закрой", "выключи", "напиши", "погода",
        "запусти", "відкрий", "увімкни", "закрий", "вимкни", "напиши", "погода"
    }
    if words & command_hints:
        _clear_pending_app_launch()
        return False

    # Пока ждём подтверждения, мягко напоминаем, что нужен ответ да/нет.
    name = pending.get("resolved_name") or pending.get("spoken_target") or "приложение"
    msg = (f"Запустить {name}? Скажите да или нет."
           if UI_LANGUAGE == "ru" else f"Запустити {name}? Скажіть так або ні.")
    interface.add_chat_message("Мита", msg, is_mita=True)
    speak(msg, force=True)
    return True


def mita_plan_intent(user_text):
    """ИИ понимает свободную фразу и превращает её только в разрешённое действие."""
    if client is None:
        return None
    lang = "uk" if UI_LANGUAGE == "ua" else "ru"
    prompt = f"""
Ты — модуль понимания команд Windows-ассистента Mita.
Пойми намерение пользователя, даже если есть ошибки распознавания, сленг, падежи,
русский/украинский/английский, транслит или неточная формулировка.

Верни ТОЛЬКО один JSON-объект без markdown.
Разрешённые intent:
launch_app, close_app, open_web, minimize_all, minimize_app, move_window,
write_text, weather, stop_speech, chat.

Схема:
{{"intent":"...", "target":"", "text":"", "monitor":0, "confidence":0.0}}

Правила:
- "зайди/открой/вруби/запусти дискордик" => launch_app.
- Любая просьба запустить/включить/открыть НАЗВАНИЕ ПРОГРАММЫ, БРАУЗЕРА, ИГРЫ или ЛАУНЧЕРА => launch_app.
- "запусти хром/яндекс/оперу/firefox/edge" => launch_app, а НЕ open_web.
- "запусти кс/доту/гта/майнкрафт/роблокс" => launch_app.
- Только если явно про сайт, веб-страницу, URL или поиск в интернете => open_web.
- "убери/закрой/выруби программу" => close_app.
- "напиши/введи/напечатай ..." => write_text, поле text содержит только текст для печати.
- Если не просит управлять ПК, intent=chat.
- Не придумывай опасные shell-команды и не возвращай код.
- Язык интерфейса: {lang}.
Фраза пользователя: {user_text!r}
"""
    try:
        completion = client.chat.completions.create(
            model="openai/gpt-oss-120b",
            messages=[{"role":"user","content":prompt}],
            temperature=0.0,
            max_completion_tokens=300,
            top_p=1,
            reasoning_effort="medium",
            stream=False
        )
        obj = _extract_json_object(completion.choices[0].message.content)
        if not isinstance(obj, dict):
            return None
        if obj.get("intent") not in {
            "launch_app","close_app","open_web","minimize_all","minimize_app",
            "move_window","write_text","weather","stop_speech","chat"
        }:
            return None
        try:
            obj["confidence"] = float(obj.get("confidence", 0.0))
        except Exception:
            obj["confidence"] = 0.0
        return obj
    except Exception as e:
        print(f"[Mita Brain] Ошибка планировщика: {e}")
        return None

def process_smart_ai_command(phrase, interface):
    plan = mita_plan_intent(phrase)
    if not plan or plan.get("confidence", 0.0) < 0.55:
        return False
    intent = plan.get("intent")
    target = str(plan.get("target") or "").strip()

    if intent == "chat":
        return False
    if intent == "launch_app":
        # Запуск приложений — только локальная папка MitaApps, без AI-resolver/Steam.
        ok, matched = smart_launch_application(target)
        msg = T("app_launching").format(matched or target) if ok else T("app_not_found").format(target)
        interface.add_chat_message("Мита", msg, is_mita=True)
        speak(msg, force=True)
        return True
    if intent == "close_app":
        ok = kill_application(target)
        msg = T("app_closing").format(target) if ok else T("app_close_failed").format(target)
        interface.add_chat_message("Мита", msg, is_mita=True); speak(msg, force=True)
        return True
    if intent == "open_web":
        open_website(target)
        msg = T("web_opening").format(target)
        interface.add_chat_message("Мита", msg, is_mita=True); speak(msg, force=True)
        return True
    if intent == "minimize_all":
        minimize_all_windows(); speak(T("minimized_all"), force=True); return True
    if intent == "minimize_app":
        ok = minimize_window(target)
        msg = ("Свернула " if UI_LANGUAGE == "ru" else "Згорнула ") + target if ok else T("app_not_found").format(target)
        interface.add_chat_message("Мита", msg, is_mita=True); speak(msg, force=True); return True
    if intent == "move_window":
        try: monitor = int(plan.get("monitor") or 0)
        except Exception: monitor = 0
        if monitor not in (1,2):
            speak(T("move_to_monitor_ask"), force=True); return True
        ok = move_window_to_monitor(target, monitor)
        speak(T("moving_window").format(monitor) if ok else T("move_failed"), force=True)
        return True
    if intent == "write_text":
        value = str(plan.get("text") or "")
        if value:
            value = prepare_text_for_typing(value)
            type_unicode_text(value)
            interface.add_chat_message("Мита", T("typing"), is_mita=True)
        return True
    if intent == "weather":
        result = get_local_weather(force=False, lang=("ua" if UI_LANGUAGE == "ua" else "ru"))
        msg = result.get("report") if result.get("ok") else result.get("error", "Ошибка погоды")
        interface.add_chat_message("Мита", msg, is_mita=True); speak(msg, force=True)
        return True
    if intent == "stop_speech":
        stop_tts()
        return True
    return False

_mita_load_memory()

# ============================================================
# GROQ API
# ============================================================

GROQ_API_KEY =  "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"
client = Groq(api_key=GROQ_API_KEY) if GROQ_API_KEY else None

def ask_groq(question):
    if client is None:
        return "⚠️ Groq API ключ не настроен."
    try:
        ui_lang = UI_LANGUAGE
        lang_text = "українською" if ui_lang == "ua" else "русском"
        system_prompt = f"""Ты — Mita, очень умный персональный голосовой ассистент для Windows.
Отвечай ТОЛЬКО на {lang_text} языке, естественно и по-человечески.
Ты должна понимать сленг, опечатки, неполные фразы, контекст и продолжения разговора.
Не притворяйся, что выполнила действие на ПК: системные действия выполняет отдельный модуль.
Если это обычный вопрос — дай полезный, точный ответ. Если вопрос простой — отвечай кратко.
Не добавляй markdown без необходимости. Не выдумывай факты."""
        messages = [{"role":"system","content":system_prompt}]
        for msg in _MITA_DIALOG_MEMORY[-12:]:
            if isinstance(msg, dict) and msg.get("role") in ("user","assistant"):
                messages.append({"role":msg["role"],"content":str(msg.get("content",""))})
        messages.append({"role":"user","content":str(question)})
        completion = client.chat.completions.create(
            model="openai/gpt-oss-120b",
            messages=messages,
            temperature=0.55,
            max_completion_tokens=2048,
            top_p=1,
            reasoning_effort="high",
            stream=False
        )
        answer = str(completion.choices[0].message.content or "").strip()
        _mita_remember("user", question)
        _mita_remember("assistant", answer)
        return answer
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
        self._base_bg = "#151123"
        self._panel = "#100d19"
        self._line = "#34264c"
        self._cyan = "#8f5cff"
        self._cyan2 = "#c3a5ff"
        self._green = "#52e6a5"
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
        self.root.configure(bg="#0b0912")

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
        self.root.configure(bg="#0b0912")

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

    W, H = 1280, 820

    def __init__(self):
        self.root = tk.Tk()
        self.root.title("Mita — Голосовой ассистент")
        self.root.geometry(f"{self.W}x{self.H}")
        self.root.minsize(1100, 700)
        self.root.configure(bg="#0b0912")
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

        # NEO UI live state
        self.live_voice_text = ""
        self.live_voice_partial = False
        self.live_voice_updated = 0.0
        self.now_playing_title = ""
        self.music_progress = 0.0
        self.music_elapsed = 0
        self.music_duration = 0
        self._neo_wave_phase = 0.0
        self._neo_player_phase = 0.0
        self._neo_status_flash = 0.0

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

        # Расширенная оболочка интерфейса MITA
        self.focus_mode = False
        self.fullscreen_mode = False
        self.always_on_top = False
        self.compact_mode = False
        self.command_palette = None
        self.palette_query = tk.StringVar(value="")
        self.clipboard_history = []
        self._last_clipboard_value = ""
        self.toast_stack = []
        self.recent_ui_actions = []
        self.telemetry_history = {"cpu": [], "ram": [], "disk": []}
        self._net_last_bytes = None
        self._net_last_time = time.time()
        self._net_speed_mbps = 0.0
        self._effects_enabled = True
        self._reduce_motion = False
        self._glow_phase = 0.0
        self._logo_phase = 0.0
        self._mouse_x = self.W // 2
        self._mouse_y = self.H // 2
        self._accent_index = 0
        self._accent_presets = [
            ("VIOLET", "#7c4dff", "#a978ff", "#c7a8ff"),
            ("NEON BLUE", "#3d7bff", "#6ca8ff", "#a8cbff"),
            ("MAGENTA", "#d343ff", "#ef7dff", "#f8b6ff"),
            ("EMERALD", "#27c98b", "#65e3b6", "#a2f1d3"),
            ("SUNSET", "#ff5f7a", "#ff8a9e", "#ffc0cb"),
        ]
        self.focus_timer_end = None
        self.focus_timer_running = False
        self.notes_window = None
        self.telemetry_window = None
        self.clipboard_window = None
        self.palette_commands = []

        # FX 3.0 / weather HUD state
        self._weather_loading = False
        self._weather_last_ui_update = 0.0
        self._weather_icon_phase = 0.0
        self._fx_frame = 0
        self._cursor_trail = []
        self._ripple_items = []
        self._shooting_stars = []
        self._aurora_phase = 0.0
        self._ambient_energy = 0.0
        self._transition_lock = False
        self._last_tab_change = 0.0

        self.colors = {
            "bg": "#0b0912", "bg2": "#0b0912", "topbar": "#0d0a15",
            "sidebar": "#0c0a13", "panel": "#141022", "panel2": "#100d1a",
            "panel3": "#19132a", "chip": "#1b1430", "active": "#24183d",
            "border": "#292039", "line": "#292039", "line2": "#4b3472",
            "primary": "#7c4dff", "primary2": "#a978ff", "purple": "#7c4dff",
            "purple2": "#a978ff", "purple3": "#c7a8ff", "cyan": "#a978ff",
            "green": "#52e6a5", "danger": "#ffb65c", "text": "#f0edf7",
            "muted": "#8f879d", "dim": "#5e566b", "chat_bg": "#0f0c17",
            "user": "#19132a", "mita": "#171121"
        }

        self.root.protocol("WM_DELETE_WINDOW", self.quit)
        self.setup_ui()
        self.setup_unique_features()
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
        outer = tk.Frame(parent, bg=self.colors["border"], bd=0)
        frame = tk.Frame(outer, bg=self.colors["panel"], bd=0)
        frame.pack(fill="both", expand=True, padx=1, pady=1)
        if title:
            head = tk.Frame(frame, bg=self.colors["panel"], height=38)
            head.pack(fill="x", padx=14, pady=(8, 0)); head.pack_propagate(False)
            tk.Label(head, text=title, font=("Segoe UI", 9, "bold"), bg=self.colors["panel"], fg=self.colors["text"]).pack(side="left", pady=7)
            if subtitle:
                tk.Label(head, text=subtitle, font=("Segoe UI", 7, "bold"), bg=self.colors["panel"], fg=self.colors["purple2"]).pack(side="right", pady=8)
        return outer

    def _bind_hover(self, widget, normal, hover):
        widget.bind("<Enter>", lambda e: widget.config(bg=hover))
        widget.bind("<Leave>", lambda e: widget.config(bg=normal))

    def _make_nav(self, parent, key, icon, idx):
        row = tk.Frame(parent, bg=self.colors["sidebar"], height=44, cursor="hand2")
        row.pack(fill="x", padx=10, pady=2); row.pack_propagate(False)
        marker = tk.Frame(row, bg=self.colors["sidebar"], width=3); marker.pack(side="left", fill="y")
        icon_lbl = tk.Label(row, text=icon, font=("Segoe UI Symbol", 12), bg=self.colors["sidebar"], fg=self.colors["muted"], width=3)
        icon_lbl.pack(side="left", padx=(5, 2))
        text_lbl = tk.Label(row, text=key, font=("Segoe UI", 9, "bold"), bg=self.colors["sidebar"], fg=self.colors["muted"], anchor="w")
        text_lbl.pack(side="left", fill="x", expand=True)
        for w in (row, icon_lbl, text_lbl, marker):
            w.bind("<Button-1>", lambda e, i=idx: self.switch_tab(i))
        self.nav_items.append((row, icon_lbl, text_lbl, marker))
        return row

    def setup_ui(self):
        self.canvas = tk.Canvas(self.root, bg=self.colors["bg"], highlightthickness=0, bd=0)
        self.canvas.pack(fill="both", expand=True)

        self.header = tk.Frame(self.root, bg=self.colors["topbar"], highlightbackground=self.colors["border"], highlightthickness=1)
        self.header.place(x=0, y=0, relwidth=1, height=58)
        brand = tk.Frame(self.header, bg=self.colors["topbar"], width=260); brand.pack(side="left", fill="y", padx=(16, 0)); brand.pack_propagate(False)
        self.logo_canvas = tk.Canvas(brand, width=34, height=34, bg=self.colors["topbar"], highlightthickness=0); self.logo_canvas.pack(side="left", pady=11)
        self.logo_glow_outer = self.logo_canvas.create_oval(3,3,31,31,outline=self.colors["line2"],width=1)
        self.logo_glow_mid = self.logo_canvas.create_oval(7,7,27,27,outline=self.colors["purple"],width=2)
        self.logo_core = self.logo_canvas.create_oval(11,11,23,23,fill=self.colors["purple2"],outline=self.colors["purple3"],width=1)
        self.logo_dot = self.logo_canvas.create_oval(15,15,19,19,fill="#ffffff",outline="")
        tk.Label(brand,text="M I T A",font=("Segoe UI",11,"bold"),bg=self.colors["topbar"],fg=self.colors["text"]).pack(side="left",padx=10)

        center = tk.Frame(self.header,bg=self.colors["topbar"]); center.pack(side="left",fill="both",expand=True)
        pill=tk.Frame(center,bg=self.colors["panel2"],highlightbackground=self.colors["border"],highlightthickness=1)
        pill.pack(pady=12)
        tk.Label(pill,text="●",font=("Segoe UI",8),bg=self.colors["panel2"],fg=self.colors["purple2"]).pack(side="left",padx=(12,5),pady=6)
        tk.Label(pill,text="MITA  NEO 4.0",font=("Segoe UI",8,"bold"),bg=self.colors["panel2"],fg=self.colors["text"]).pack(side="left",pady=6)
        tk.Label(pill,text="  •  ONLINE",font=("Segoe UI",7,"bold"),bg=self.colors["panel2"],fg=self.colors["muted"]).pack(side="left",padx=(8,12),pady=6)

        right=tk.Frame(self.header,bg=self.colors["topbar"]); right.pack(side="right",fill="y",padx=14)
        tk.Label(right,text="КОМАНДЫ",font=("Segoe UI",8,"bold"),bg=self.colors["topbar"],fg=self.colors["muted"]).pack(side="left",padx=14)
        tk.Label(right,text="439",font=("Segoe UI",7,"bold"),bg=self.colors["chip"],fg=self.colors["purple2"],padx=7,pady=2).pack(side="left")
        tk.Label(right,text="НАСТРОЙКИ",font=("Segoe UI",8,"bold"),bg=self.colors["topbar"],fg=self.colors["muted"]).pack(side="left",padx=18)
        self.header_clock=tk.Label(right,text="",font=("Segoe UI",8,"bold"),bg=self.colors["topbar"],fg=self.colors["text"]); self.header_clock.pack(side="left",padx=8)

        self.body=tk.Frame(self.root,bg=self.colors["bg"]); self.body.place(x=0,y=58,relwidth=1,relheight=.872)
        self.sidebar=tk.Frame(self.body,bg=self.colors["sidebar"],width=250,highlightbackground=self.colors["border"],highlightthickness=1)
        self.sidebar.pack(side="left",fill="y"); self.sidebar.pack_propagate(False)
        self.content=tk.Frame(self.body,bg=self.colors["bg"]); self.content.pack(side="left",fill="both",expand=True)
        self.rightbar=tk.Frame(self.body,bg=self.colors["bg"],width=325); self.rightbar.pack(side="right",fill="y",padx=(0,12),pady=(12,8)); self.rightbar.pack_propagate(False)

        self.bottom_bar=tk.Frame(self.root,bg=self.colors["topbar"],highlightbackground=self.colors["border"],highlightthickness=1)
        self.bottom_bar.place(x=0,rely=1,y=-56,relwidth=1,height=56)
        leftb=tk.Frame(self.bottom_bar,bg=self.colors["topbar"]); leftb.pack(side="left",fill="y",padx=14)
        tk.Label(leftb,text="☼",font=("Segoe UI Symbol",12),bg=self.colors["chip"],fg=self.colors["muted"],padx=10,pady=7).pack(side="left",pady=9)
        tk.Label(leftb,text="🔊",font=("Segoe UI Emoji",10),bg=self.colors["topbar"],fg=self.colors["muted"]).pack(side="left",padx=(14,4))
        self.bottom_volume=tk.Scale(leftb,from_=0,to=100,orient=tk.HORIZONTAL,showvalue=False,length=120,bg=self.colors["topbar"],troughcolor=self.colors["purple"],activebackground=self.colors["purple2"],highlightthickness=0,bd=0,sliderrelief="flat")
        self.bottom_volume.set(70); self.bottom_volume.pack(side="left")
        tk.Label(leftb,text="70%",font=("Segoe UI",8,"bold"),bg=self.colors["topbar"],fg=self.colors["muted"]).pack(side="left",padx=6)
        home=tk.Label(self.bottom_bar,text="⌂",font=("Segoe UI Symbol",16,"bold"),bg=self.colors["purple"],fg="white",padx=17,pady=7,cursor="hand2")
        home.place(relx=.5,rely=.5,anchor="center"); home.bind("<Button-1>",lambda e:self.switch_tab(0))

        self.build_sidebar(); self.build_content(); self.build_rightbar()
        self.create_particles(); self.create_stars(); self.create_floating_stars()
        self.update_key_info(); self.switch_tab(0)

    def build_sidebar(self):
        self.nav_items=[]
        self._make_nav(self.sidebar,"Главная","⌂",0)
        self._make_nav(self.sidebar,"Чат с Mita","◯",1)
        self._make_nav(self.sidebar,"Управление ПК","▣",2)
        self._make_nav(self.sidebar,"Команды","⌘",2)
        self._make_nav(self.sidebar,"Сценарии","▱",2)
        self._make_nav(self.sidebar,"Память","▤",1)
        self._make_nav(self.sidebar,"Плагины","✧",3)
        self._make_nav(self.sidebar,"Настройки","⚙",3)
        tk.Label(self.sidebar,text="С О С Т О Я Н И Е",font=("Segoe UI",7,"bold"),bg=self.colors["sidebar"],fg=self.colors["dim"]).pack(anchor="w",padx=18,pady=(18,8))
        for icon,name,state,col in [("⌁","Голос","Включено",self.colors["green"]),("♩","Микрофон","Включён",self.colors["green"]),("▣","Система","Оптимально",self.colors["green"])]:
            r=tk.Frame(self.sidebar,bg=self.colors["sidebar"]); r.pack(fill="x",padx=16,pady=4)
            tk.Label(r,text=icon,font=("Segoe UI Symbol",10),bg=self.colors["sidebar"],fg=self.colors["muted"],width=3).pack(side="left")
            t=tk.Frame(r,bg=self.colors["sidebar"]); t.pack(side="left")
            tk.Label(t,text=name,font=("Segoe UI",8,"bold"),bg=self.colors["sidebar"],fg=self.colors["text"]).pack(anchor="w")
            tk.Label(t,text=state,font=("Segoe UI",7),bg=self.colors["sidebar"],fg=col).pack(anchor="w")

        spacer=tk.Frame(self.sidebar,bg=self.colors["sidebar"]); spacer.pack(fill="both",expand=True)
        orb=tk.Canvas(self.sidebar,width=100,height=100,bg=self.colors["sidebar"],highlightthickness=0); orb.pack(pady=(0,6))
        for r,c in [(42,"#1e1235"),(34,"#2d1951"),(25,self.colors["purple"])] : orb.create_oval(50-r,50-r,50+r,50+r,fill=c,outline="")
        for i,h in enumerate([20,32,42,28,38,24,34]): orb.create_rectangle(30+i*6,50-h/2,33+i*6,50+h/2,fill="#e1c8ff",outline="")
        self.manual_record_button=self._button(self.sidebar,"Запустить",self.toggle_manual_record,accent=True); self.manual_record_button.pack(fill="x",padx=28,pady=(0,10))
        self.mode_label=tk.Label(self.sidebar,text=get_mode_name(_mita_mode),font=("Segoe UI",7),bg=self.colors["sidebar"],fg=self.colors["muted"]); self.mode_label.pack()
        self.mode_button=self._button(self.sidebar,"Сменить режим",self.change_mode); self.mode_button.pack(fill="x",padx=28,pady=4)
        self.tts_mute_button=self._button(self.sidebar,T("voice_on"),self.toggle_tts_mute); self.tts_mute_button.pack(fill="x",padx=28,pady=4)
        self.corrector_button=self._button(self.sidebar,T("corrector_off"),self.toggle_corrector); self.corrector_button.pack(fill="x",padx=28,pady=4)
        self.engine_status=tk.Label(self.sidebar,text="● MITA ONLINE",font=("Segoe UI",7,"bold"),bg=self.colors["sidebar"],fg=self.colors["green"]); self.engine_status.pack(pady=(4,0))
        self.key_info_label=tk.Label(self.sidebar,text="",font=("Segoe UI",6),bg=self.colors["sidebar"],fg=self.colors["muted"]); self.key_info_label.pack(pady=(2,0))
        self.change_key_btn=self._button(self.sidebar,T("change_key"),self.change_key); self.change_key_btn.pack(fill="x",padx=28,pady=(4,10))

    def build_rightbar(self):
        ai=self._card(self.rightbar,"AI Помощник",None); ai.pack(fill="x",pady=(0,12)); a=ai.winfo_children()[0]
        row=tk.Frame(a,bg=self.colors["panel"]); row.pack(fill="x",padx=14,pady=(5,14))
        txt=tk.Frame(row,bg=self.colors["panel"]); txt.pack(side="left",fill="both",expand=True)
        tk.Label(txt,text="Я готова помочь вам с любыми\nзадачами на компьютере",font=("Segoe UI",8),justify="left",bg=self.colors["panel"],fg=self.colors["muted"]).pack(anchor="w")
        tk.Label(txt,text="●  Онлайн",font=("Segoe UI",7,"bold"),bg=self.colors["panel"],fg=self.colors["green"]).pack(anchor="w",pady=(8,0))
        self.shield_canvas=tk.Canvas(row,width=96,height=96,bg=self.colors["panel"],highlightthickness=0); self.shield_canvas.pack(side="right")
        for r,c in [(40,"#2b1748"),(31,"#472174"),(22,"#6734a4"),(12,"#8c58dc")]: self.shield_canvas.create_oval(48-r,48-r,48+r,48+r,outline=c,width=2)
        self.shield_canvas.create_oval(42,42,54,54,fill=self.colors["purple2"],outline="")

        sysc=self._card(self.rightbar,"Система",None); sysc.pack(fill="x",pady=(0,12)); s=sysc.winfo_children()[0]
        meters=tk.Frame(s,bg=self.colors["panel"]); meters.pack(fill="x",padx=12,pady=(4,12))
        for title,val in [("CPU","13%"),("RAM","45%"),("ДИСК","83%")]:
            c=tk.Canvas(meters,width=78,height=78,bg=self.colors["panel"],highlightthickness=0); c.pack(side="left",expand=True)
            c.create_oval(12,8,66,62,outline="#2a2440",width=5); c.create_arc(12,8,66,62,start=90,extent=-235,style="arc",outline=self.colors["purple2"],width=5)
            c.create_text(39,31,text=val,font=("Segoe UI",9,"bold"),fill=self.colors["text"]); c.create_text(39,69,text=title,font=("Segoe UI",6,"bold"),fill=self.colors["muted"])
        self.ram_value=tk.Label(s,text="Температура: —",font=("Segoe UI",7),bg=self.colors["panel"],fg=self.colors["muted"]); self.ram_value.pack(side="left",padx=14,pady=(0,10))
        self.audio_value=tk.Label(s,text="Сеть: 0.3 Мбит/с",font=("Segoe UI",7),bg=self.colors["panel"],fg=self.colors["muted"]); self.audio_value.pack(side="right",padx=14,pady=(0,10))

        proc=self._card(self.rightbar,"Активные процессы","Посмотреть всё"); proc.pack(fill="x",pady=(0,12)); p=proc.winfo_children()[0]
        for n,v in [("obs64","22.2%"),("Msedge","11.3%"),("msedgewebview2","11.1%"),("Msedge","5.8%")]:
            r=tk.Frame(p,bg=self.colors["panel"]); r.pack(fill="x",padx=12,pady=5)
            tk.Label(r,text="●",font=("Segoe UI",8),bg=self.colors["panel"],fg=self.colors["purple2"]).pack(side="left")
            tk.Label(r,text=n,font=("Segoe UI",8),bg=self.colors["panel"],fg=self.colors["text"]).pack(side="left",padx=8)
            tk.Label(r,text=v,font=("Segoe UI",8,"bold"),bg=self.colors["panel"],fg=self.colors["muted"]).pack(side="right")
        log=self._card(self.rightbar,"Голосовые команды",None); log.pack(fill="both",expand=True); li=log.winfo_children()[0]
        self.event_log=tk.Text(li,bg=self.colors["panel"],fg=self.colors["muted"],font=("Segoe UI",7),bd=0,height=5,padx=12,pady=8)
        self.event_log.pack(fill="both",expand=True); self.event_log.insert("end","439 активных команд\nMita готова к работе\n"); self.event_log.config(state="disabled")
        self.sync_canvas=tk.Canvas(self.rightbar,width=1,height=1,bg=self.colors["bg"],highlightthickness=0)

    def build_content(self):
        self.pages={}; self.build_home_page(); self.build_chat_page(); self.build_commands_page(); self.build_settings_page()

    def _page(self):
        return tk.Frame(self.content,bg=self.colors["bg"])

    def build_home_page(self):
        page = self._page(); self.pages[0] = page

        # ===== NEO HERO =====
        hero = tk.Frame(page, bg=self.colors["bg"])
        hero.pack(fill="x", padx=28, pady=(22, 10))

        hero_left = tk.Frame(hero, bg=self.colors["bg"])
        hero_left.pack(side="left", fill="x", expand=True)
        tk.Label(hero_left, text="M I T A  //  NEO CORE", font=("Segoe UI", 8, "bold"),
                 bg=self.colors["bg"], fg=self.colors["purple2"]).pack(anchor="w")
        title_row = tk.Frame(hero_left, bg=self.colors["bg"]); title_row.pack(anchor="w", pady=(3, 0))
        tk.Label(title_row, text="Твой AI-помощник", font=("Segoe UI", 26, "bold"),
                 bg=self.colors["bg"], fg=self.colors["text"]).pack(side="left")
        self.neo_online_dot = tk.Label(title_row, text="  ● ONLINE", font=("Segoe UI", 8, "bold"),
                                       bg=self.colors["bg"], fg=self.colors["green"])
        self.neo_online_dot.pack(side="left", padx=(12, 0), pady=(9, 0))
        self.neo_subtitle = tk.Label(hero_left,
            text="Голос • AI • музыка • управление Windows • быстрые действия",
            font=("Segoe UI", 9), bg=self.colors["bg"], fg=self.colors["muted"])
        self.neo_subtitle.pack(anchor="w", pady=(3, 0))

        hero_right = tk.Frame(hero, bg=self.colors["panel2"], highlightbackground=self.colors["border"], highlightthickness=1)
        hero_right.pack(side="right", padx=(12,0))
        self.neo_clock = tk.Label(hero_right, text="--:--", font=("Segoe UI", 16, "bold"),
                                  bg=self.colors["panel2"], fg=self.colors["text"], padx=16, pady=6)
        self.neo_clock.pack()
        self.neo_mode = tk.Label(hero_right, text=get_mode_name(_mita_mode), font=("Segoe UI", 7, "bold"),
                                 bg=self.colors["panel2"], fg=self.colors["purple2"], padx=16, pady=0)
        self.neo_mode.pack()

        # ===== LIVE WEATHER CARD =====
        weather_outer = tk.Frame(page, bg=self.colors["line2"])
        weather_outer.pack(fill="x", padx=28, pady=(0, 10))
        weather = tk.Frame(weather_outer, bg=self.colors["panel2"], cursor="hand2")
        weather.pack(fill="x", expand=True, padx=1, pady=1)

        weather_left = tk.Frame(weather, bg=self.colors["panel2"])
        weather_left.pack(side="left", fill="x", expand=True, padx=(14, 6), pady=10)

        self.weather_orb = tk.Canvas(weather_left, width=54, height=54, bg=self.colors["panel2"], highlightthickness=0)
        self.weather_orb.pack(side="left", padx=(0, 12))
        self.weather_orb.create_oval(4, 4, 50, 50, fill=self.colors["chip"], outline=self.colors["purple"], width=1)
        self.weather_orb_text = self.weather_orb.create_text(27, 27, text="☁", font=("Segoe UI Emoji", 22), fill=self.colors["text"])

        weather_text = tk.Frame(weather_left, bg=self.colors["panel2"])
        weather_text.pack(side="left", fill="x", expand=True)
        tk.Label(weather_text, text="LIVE WEATHER", font=("Segoe UI", 7, "bold"),
                 bg=self.colors["panel2"], fg=self.colors["purple2"]).pack(anchor="w")
        self.weather_main_label = tk.Label(weather_text, text="Загружаю погоду…",
                 font=("Segoe UI", 12, "bold"), bg=self.colors["panel2"], fg=self.colors["text"], anchor="w")
        self.weather_main_label.pack(fill="x", pady=(2, 1))
        self.weather_sub_label = tk.Label(weather_text, text="Определяю местоположение автоматически",
                 font=("Segoe UI", 7), bg=self.colors["panel2"], fg=self.colors["muted"], anchor="w")
        self.weather_sub_label.pack(fill="x")

        weather_right = tk.Frame(weather, bg=self.colors["panel2"])
        weather_right.pack(side="right", padx=(6, 14), pady=10)
        self.weather_refresh_label = tk.Label(weather_right, text="↻  ОБНОВИТЬ", font=("Segoe UI", 7, "bold"),
                 bg=self.colors["chip"], fg=self.colors["purple2"], padx=10, pady=6, cursor="hand2")
        self.weather_refresh_label.pack()
        self.weather_updated_label = tk.Label(weather_right, text="AUTO • 10 MIN", font=("Segoe UI", 6),
                 bg=self.colors["panel2"], fg=self.colors["muted"])
        self.weather_updated_label.pack(pady=(4, 0))

        for w in (weather, weather_left, weather_text, self.weather_main_label, self.weather_sub_label,
                  self.weather_orb, weather_right, self.weather_refresh_label):
            w.bind("<Button-1>", lambda e: self.refresh_weather_widget(force=True))

        # ===== LIVE VOICE CARD =====
        voice_outer = tk.Frame(page, bg=self.colors["line2"])
        voice_outer.pack(fill="x", padx=28, pady=(4, 10))
        voice = tk.Frame(voice_outer, bg=self.colors["panel2"])
        voice.pack(fill="both", expand=True, padx=1, pady=1)

        top = tk.Frame(voice, bg=self.colors["panel2"]); top.pack(fill="x", padx=16, pady=(12, 5))
        self.live_voice_badge = tk.Label(top, text="●  МИКРОФОН ГОТОВ", font=("Segoe UI", 8, "bold"),
                                         bg=self.colors["panel2"], fg=self.colors["green"])
        self.live_voice_badge.pack(side="left")
        tk.Label(top, text="говори: «Стелла ...»", font=("Segoe UI", 7),
                 bg=self.colors["panel2"], fg=self.colors["muted"]).pack(side="right")

        self.live_voice_label = tk.Label(
            voice, text="Я покажу здесь то, что слышу...",
            font=("Segoe UI", 15, "bold"), anchor="w", justify="left",
            bg=self.colors["panel2"], fg=self.colors["text"], wraplength=690
        )
        self.live_voice_label.pack(fill="x", padx=16, pady=(1, 5))

        self.voice_wave_canvas = tk.Canvas(voice, height=40, bg=self.colors["panel2"], highlightthickness=0)
        self.voice_wave_canvas.pack(fill="x", padx=14, pady=(0, 10))
        self.voice_wave_bars = []
        for i in range(46):
            x = 5 + i * 15
            bar = self.voice_wave_canvas.create_rectangle(x, 19, x+7, 21, fill=self.colors["line2"], outline="")
            self.voice_wave_bars.append(bar)

        # ===== CENTER GRID: NOW PLAYING + QUICK CORE =====
        middle = tk.Frame(page, bg=self.colors["bg"])
        middle.pack(fill="x", padx=28, pady=(0, 10))
        middle.columnconfigure(0, weight=3); middle.columnconfigure(1, weight=2)

        player_outer = tk.Frame(middle, bg=self.colors["border"])
        player_outer.grid(row=0, column=0, sticky="nsew", padx=(0,6))
        player = tk.Frame(player_outer, bg=self.colors["panel"])
        player.pack(fill="both", expand=True, padx=1, pady=1)

        ph = tk.Frame(player, bg=self.colors["panel"]); ph.pack(fill="x", padx=15, pady=(11,4))
        tk.Label(ph, text="♫  NOW PLAYING", font=("Segoe UI",8,"bold"), bg=self.colors["panel"], fg=self.colors["purple2"]).pack(side="left")
        self.player_state = tk.Label(ph, text="IDLE", font=("Segoe UI",7,"bold"), bg=self.colors["chip"], fg=self.colors["muted"], padx=8, pady=2)
        self.player_state.pack(side="right")

        self.player_title = tk.Label(player, text="Музыка не запущена", font=("Segoe UI",13,"bold"),
                                     bg=self.colors["panel"], fg=self.colors["text"], anchor="w")
        self.player_title.pack(fill="x", padx=15, pady=(2,1))
        self.player_meta = tk.Label(player, text="Скажи: «Стелла, включи песню ...»", font=("Segoe UI",7),
                                    bg=self.colors["panel"], fg=self.colors["muted"], anchor="w")
        self.player_meta.pack(fill="x", padx=15)

        self.player_vis_canvas = tk.Canvas(player, height=52, bg=self.colors["panel"], highlightthickness=0)
        self.player_vis_canvas.pack(fill="x", padx=14, pady=(5,2))
        self.player_vis_bars=[]
        for i in range(34):
            x=4+i*14
            b=self.player_vis_canvas.create_rectangle(x,25,x+7,28,fill=self.colors["purple"],outline="")
            self.player_vis_bars.append(b)

        progress_row = tk.Frame(player,bg=self.colors["panel"]); progress_row.pack(fill="x",padx=15,pady=(0,5))
        self.player_time = tk.Label(progress_row,text="00:00",font=("Consolas",7),bg=self.colors["panel"],fg=self.colors["muted"]); self.player_time.pack(side="left")
        self.player_progress_canvas=tk.Canvas(progress_row,height=8,bg=self.colors["panel"],highlightthickness=0); self.player_progress_canvas.pack(side="left",fill="x",expand=True,padx=8)
        self.player_progress_bg=self.player_progress_canvas.create_rectangle(0,2,400,6,fill=self.colors["chip"],outline="")
        self.player_progress_fg=self.player_progress_canvas.create_rectangle(0,2,1,6,fill=self.colors["purple2"],outline="")
        self.player_duration_label=tk.Label(progress_row,text="--:--",font=("Consolas",7),bg=self.colors["panel"],fg=self.colors["muted"]); self.player_duration_label.pack(side="right")

        controls=tk.Frame(player,bg=self.colors["panel"]); controls.pack(fill="x",padx=15,pady=(2,12))
        self.neo_music_stop=self._button(controls,"■  СТОП",self.stop_music_click,danger=True); self.neo_music_stop.pack(side="left")
        self._button(controls,"−10",lambda:volume_down(self)).pack(side="left",padx=(7,4))
        self._button(controls,"+10",lambda:volume_up(self)).pack(side="left",padx=4)
        self.neo_volume_label=tk.Label(controls,text=f"{_music_volume}%",font=("Segoe UI",8,"bold"),bg=self.colors["panel"],fg=self.colors["purple2"]); self.neo_volume_label.pack(side="right")

        quick_outer = tk.Frame(middle,bg=self.colors["border"])
        quick_outer.grid(row=0,column=1,sticky="nsew",padx=(6,0))
        quick=tk.Frame(quick_outer,bg=self.colors["panel"]); quick.pack(fill="both",expand=True,padx=1,pady=1)
        tk.Label(quick,text="QUICK CORE",font=("Segoe UI",8,"bold"),bg=self.colors["panel"],fg=self.colors["purple2"]).pack(anchor="w",padx=14,pady=(11,7))
        qgrid=tk.Frame(quick,bg=self.colors["panel"]); qgrid.pack(fill="both",expand=True,padx=10,pady=(0,10))
        qitems=[
            ("🎙","Голос",self.toggle_manual_record),
            ("💬","Чат",lambda:self.switch_tab(1)),
            ("⌘","Команды",lambda:self.switch_tab(2)),
            ("☁","Погода",lambda:self.refresh_weather_widget(force=True)),
            ("✎","Заметки",self.open_notes),
            ("⌁","Система",self.open_telemetry_center),
        ]
        for i,(ic,tx,cmd) in enumerate(qitems):
            cell=tk.Frame(qgrid,bg=self.colors["panel2"],highlightbackground=self.colors["border"],highlightthickness=1,cursor="hand2")
            cell.grid(row=i//2,column=i%2,sticky="nsew",padx=4,pady=4,ipady=7)
            qgrid.columnconfigure(i%2,weight=1); qgrid.rowconfigure(i//2,weight=1)
            ico=tk.Label(cell,text=ic,font=("Segoe UI Emoji",12),bg=self.colors["panel2"],fg=self.colors["purple2"]); ico.pack(pady=(5,1))
            lab=tk.Label(cell,text=tx,font=("Segoe UI",7,"bold"),bg=self.colors["panel2"],fg=self.colors["text"]); lab.pack(pady=(0,5))
            for w in (cell,ico,lab): w.bind("<Button-1>",lambda e,c=cmd:c())

        # ===== BOTTOM SMART STRIP =====
        strip=tk.Frame(page,bg=self.colors["bg"]); strip.pack(fill="x",padx=28,pady=(0,12))
        cards=[
            ("AI MODE",lambda:get_mode_name(_mita_mode),"◇"),
            ("VOICE",lambda:"ON" if not self.tts_muted else "MUTED","♩"),
            ("CORRECTOR",lambda:"ON" if self.corrector_enabled else "OFF","Aa"),
            ("FX",lambda:"ON" if self._effects_enabled else "OFF","✦"),
        ]
        self.neo_info_values=[]
        for i,(name,getter,ic) in enumerate(cards):
            c=tk.Frame(strip,bg=self.colors["panel2"],highlightbackground=self.colors["border"],highlightthickness=1)
            c.pack(side="left",fill="x",expand=True,padx=(0 if i==0 else 4,0))
            tk.Label(c,text=ic,font=("Segoe UI Symbol",11,"bold"),bg=self.colors["panel2"],fg=self.colors["purple2"]).pack(side="left",padx=(10,7),pady=8)
            tx=tk.Frame(c,bg=self.colors["panel2"]); tx.pack(side="left",fill="x",expand=True,pady=6)
            tk.Label(tx,text=name,font=("Segoe UI",6,"bold"),bg=self.colors["panel2"],fg=self.colors["muted"]).pack(anchor="w")
            val=tk.Label(tx,text=getter(),font=("Segoe UI",7,"bold"),bg=self.colors["panel2"],fg=self.colors["text"]); val.pack(anchor="w")
            self.neo_info_values.append((val,getter))

        # Compatibility widgets used by old logic; hidden but kept alive.
        hidden=tk.Frame(page,bg=self.colors["bg"]); hidden.pack()
        self.core_canvas=tk.Canvas(hidden,width=1,height=1,bg=self.colors["bg"],highlightthickness=0); self.core_canvas.pack(); self.core_canvas.bind("<Configure>",self._resize_core)
        self.status_text=tk.Label(hidden,text=T("ready"),bg=self.colors["bg"]); self.status_sub=tk.Label(hidden,text=T("waiting"),bg=self.colors["bg"])
        self.music_title=tk.Label(hidden,text="NO ACTIVE AUDIO STREAM",bg=self.colors["bg"]); self.music_meta=tk.Label(hidden,text="",bg=self.colors["bg"])
        self.music_stop_button=self._button(hidden,"STOP",self.stop_music_click,danger=True); self.music_stop_button.config(state=tk.DISABLED)

    def build_chat_page(self):
        page=self._page(); self.pages[1]=page
        head=tk.Frame(page,bg=self.colors["bg"]); head.pack(fill="x",padx=34,pady=(28,10))
        tk.Label(head,text="Чат с Mita",font=("Segoe UI",20,"bold"),bg=self.colors["bg"],fg=self.colors["text"]).pack(side="left")
        self.chat_counter=tk.Label(head,text="0 сообщений",font=("Segoe UI",8),bg=self.colors["bg"],fg=self.colors["purple2"]); self.chat_counter.pack(side="right")
        card=self._card(page,"Диалог",None); card.pack(fill="both",expand=True,padx=34,pady=(0,16)); ci=card.winfo_children()[0]
        self.chat_display=scrolledtext.ScrolledText(ci,bg=self.colors["panel"],fg=self.colors["text"],font=("Segoe UI",9),wrap=tk.WORD,bd=0,relief="flat",insertbackground=self.colors["purple2"],padx=18,pady=14,selectbackground=self.colors["purple"])
        self.chat_display.pack(fill="both",expand=True,padx=8,pady=(3,8)); self.chat_display.config(state=tk.DISABLED)
        self.chat_display.tag_config("mita_name",foreground=self.colors["purple2"],font=("Segoe UI",8,"bold")); self.chat_display.tag_config("mita_text",foreground=self.colors["text"],font=("Segoe UI",9))
        self.chat_display.tag_config("user_name",foreground=self.colors["green"],font=("Segoe UI",8,"bold")); self.chat_display.tag_config("user_text",foreground=self.colors["text"],font=("Segoe UI",9))
        self.chat_menu=Menu(self.root,tearoff=0,bg=self.colors["panel2"],fg=self.colors["text"],activebackground=self.colors["purple"],activeforeground="white",bd=0)
        self.chat_menu.add_command(label="Копировать",command=self.copy_selected_message); self.chat_menu.add_command(label="Копировать всё",command=self.copy_all_chat); self.chat_menu.add_separator(); self.chat_menu.add_command(label="Очистить",command=self.clear_chat)
        self.chat_display.bind("<Button-3>",self.show_chat_menu); self.chat_display.bind("<Control-c>",lambda e:self.copy_selected_message()); self.chat_display.bind("<Control-a>",lambda e:self.select_all_chat())
        bottom=tk.Frame(ci,bg=self.colors["panel"]); bottom.pack(fill="x",padx=8,pady=(0,8))
        self.chat_input=tk.Entry(bottom,bg=self.colors["panel2"],fg=self.colors["text"],font=("Segoe UI",9),relief="flat",bd=0,insertbackground=self.colors["purple2"]); self.chat_input.pack(side="left",fill="x",expand=True,ipady=11,padx=(0,8))
        self.chat_input.bind("<Return>",lambda e:self.send_chat_message()); self.chat_input.bind("<Button-3>",self.show_input_menu); self.chat_input.bind("<Control-a>",lambda e:self.select_all_input()); self.chat_input.bind("<Control-c>",lambda e:self.copy_input_text()); self.chat_input.bind("<Control-v>",lambda e:self.paste_input_text()); self.chat_input.bind("<Control-x>",lambda e:self.cut_input_text())
        self.send_button=self._button(bottom,"Отправить  ▶",self.send_chat_message,accent=True); self.send_button.pack(side="right")
        self.input_menu=Menu(self.root,tearoff=0,bg=self.colors["panel2"],fg=self.colors["text"],activebackground=self.colors["purple"],bd=0)
        self.input_menu.add_command(label="Вырезать",command=self.cut_input_text); self.input_menu.add_command(label="Копировать",command=self.copy_input_text); self.input_menu.add_command(label="Вставить",command=self.paste_input_text); self.input_menu.add_separator(); self.input_menu.add_command(label="Очистить",command=self.clear_input_text); self.input_menu.add_command(label="Выделить всё",command=self.select_all_input)
        hello_msg=(f"{T('mita_greeting')}\n{T('mita_mode')}{get_mode_name(_mita_mode)}\n{T('mita_corrector')}{T('corrector_off_info') if not _text_corrector_enabled else T('corrector_on_info')}\n{T('mita_lang')}\n{T('mita_help')}")
        self.add_chat_message("Мита",hello_msg,is_mita=True)

    def build_commands_page(self):
        page=self._page(); self.pages[2]=page
        tk.Label(page,text="Команды",font=("Segoe UI",20,"bold"),bg=self.colors["bg"],fg=self.colors["text"]).pack(anchor="w",padx=34,pady=(28,12))
        wrap=tk.Frame(page,bg=self.colors["bg"]); wrap.pack(fill="both",expand=True,padx=30,pady=(0,16))
        commands=[("Открыть приложение","Запусти Discord / Roblox / Steam"),("Открыть сайт","Открой YouTube / Google"),("Закрыть приложение","Закрой Discord"),("Напечатать текст","Напиши [текст]"),("Свернуть окна","Сверни все"),("Скриншот","Сделай скриншот"),("Буфер обмена","Скопируй / Вставь"),("Монитор","Переведи на 1/2 монитор"),("Музыка","Песня [название]"),("Громкость","Громкость 50%"),("Стоп музыки","Стоп музыка"),("Язык","Смени язык"),("Погода рядом","Мита, какая сейчас погода?")]
        for i,(title,desc) in enumerate(commands):
            c=self._card(wrap,title,None); c.grid(row=i//3,column=i%3,sticky="nsew",padx=5,pady=5); wrap.columnconfigure(i%3,weight=1)
            inner=c.winfo_children()[0]; tk.Label(inner,text=desc,font=("Segoe UI",8),bg=self.colors["panel"],fg=self.colors["muted"]).pack(anchor="w",padx=14,pady=(4,14))

    def build_settings_page(self):
        page=self._page(); self.pages[3]=page
        tk.Label(page,text="Настройки",font=("Segoe UI",20,"bold"),bg=self.colors["bg"],fg=self.colors["text"]).pack(anchor="w",padx=34,pady=(28,12))
        wrap=tk.Frame(page,bg=self.colors["bg"]); wrap.pack(fill="both",expand=True,padx=34,pady=(0,16)); wrap.columnconfigure(0,weight=1); wrap.columnconfigure(1,weight=1)
        lang=self._card(wrap,"Язык интерфейса",None); lang.grid(row=0,column=0,sticky="nsew",padx=(0,6),pady=6); li=lang.winfo_children()[0]
        self.lang_button=self._button(li,"Украинский" if UI_LANGUAGE=="ru" else "Русский",self.toggle_language); self.lang_button.pack(fill="x",padx=14,pady=(5,14))
        mode=self._card(wrap,"Режим Mita",None); mode.grid(row=0,column=1,sticky="nsew",padx=(6,0),pady=6); mi=mode.winfo_children()[0]
        self.mode_setting_label=tk.Label(mi,text=get_mode_name(_mita_mode),font=("Segoe UI",9,"bold"),bg=self.colors["panel"],fg=self.colors["purple2"]); self.mode_setting_label.pack(anchor="w",padx=14,pady=(5,8)); self._button(mi,"Сменить режим",self.change_mode).pack(fill="x",padx=14,pady=(0,14))
        music=self._card(wrap,"Громкость музыки",None); music.grid(row=1,column=0,columnspan=2,sticky="nsew",pady=6); mui=music.winfo_children()[0]
        self.volume_label=tk.Label(mui,text=f"Громкость: {_music_volume}%",font=("Segoe UI",9,"bold"),bg=self.colors["panel"],fg=self.colors["text"]); self.volume_label.pack(anchor="w",padx=14,pady=(5,4))
        self.volume_scale=tk.Scale(mui,from_=0,to=200,orient=tk.HORIZONTAL,bg=self.colors["panel"],fg=self.colors["purple2"],activebackground=self.colors["purple2"],troughcolor=self.colors["chip"],highlightthickness=0,bd=0,showvalue=False,sliderrelief="flat",command=self.on_volume_change); self.volume_scale.pack(fill="x",padx=14,pady=(0,14)); self.volume_scale.set(_music_volume)
        access=self._card(wrap,"Ключ доступа",None); access.grid(row=2,column=0,columnspan=2,sticky="nsew",pady=6); ac=access.winfo_children()[0]
        ar=tk.Frame(ac,bg=self.colors["panel"]); ar.pack(fill="x",padx=14,pady=(5,14)); self.settings_key_label=tk.Label(ar,text="",font=("Segoe UI",8),bg=self.colors["panel"],fg=self.colors["text"]); self.settings_key_label.pack(side="left"); self._button(ar,"Сменить ключ",self.change_key).pack(side="right")

    def switch_tab(self, idx):
        self.current_tab=idx
        for i,(row,icon,label,marker) in enumerate(self.nav_items):
            target=[0,1,2,2,2,1,3,3][i]
            active=target==idx
            bg=self.colors["active"] if active else self.colors["sidebar"]
            row.config(bg=bg); icon.config(bg=bg,fg=self.colors["purple2"] if active else self.colors["muted"]); label.config(bg=bg,fg=self.colors["text"] if active else self.colors["muted"]); marker.config(bg=self.colors["purple2"] if active else bg)
        for p in self.pages.values(): p.pack_forget()
        self.pages[idx].pack(fill="both",expand=True)


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
        for r, col in [(150,"#1a102b"),(125,"#25163b"),(102,"#342052"),(83,"#4b2c78")]:
            self.aura.append(c.create_oval(self.cx-r,self.cy-r,self.cx+r,self.cy+r,
                                           outline=col, width=1))
        self.core_outer = c.create_oval(self.cx-66,self.cy-66,self.cx+66,self.cy+66,
                                        fill="#100d1a", outline=self.colors["primary"], width=2)
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
        if hasattr(self,"neo_clock"):
            self.neo_clock.config(text=datetime.now().strftime("%H:%M"))
        if hasattr(self,"neo_mode"):
            self.neo_mode.config(text=get_mode_name(_mita_mode))

        # NEO voice waveform
        self._neo_wave_phase += .22
        if hasattr(self,"voice_wave_canvas"):
            try:
                wh=max(32,self.voice_wave_canvas.winfo_height())
                for i,bar in enumerate(self.voice_wave_bars):
                    if self.is_listening:
                        amp=5+abs(math.sin(self._neo_wave_phase+i*.43))*16+random.random()*7
                        col=self.colors["purple2"]
                    elif self.is_processing:
                        amp=4+abs(math.sin(self._neo_wave_phase*.7+i*.31))*10
                        col=self.colors["purple"]
                    else:
                        amp=2+abs(math.sin(self._neo_wave_phase*.35+i*.28))*3
                        col=self.colors["line2"]
                    x=5+i*15; cy=wh/2
                    self.voice_wave_canvas.coords(bar,x,cy-amp/2,x+7,cy+amp/2)
                    self.voice_wave_canvas.itemconfig(bar,fill=col)
            except Exception: pass

        # Animated music equalizer + smooth progress line
        self._neo_player_phase += .18
        if hasattr(self,"player_vis_canvas"):
            try:
                vh=max(42,self.player_vis_canvas.winfo_height())
                active=self.music_mode or is_music_playing()
                for i,bar in enumerate(self.player_vis_bars):
                    amp=(7+abs(math.sin(self._neo_player_phase+i*.52))*30*max(.35,_music_intensity)) if active else (3+abs(math.sin(self._neo_player_phase*.25+i*.4))*3)
                    x=4+i*14
                    self.player_vis_canvas.coords(bar,x,vh/2-amp/2,x+7,vh/2+amp/2)
                    self.player_vis_canvas.itemconfig(bar,fill=self.colors["purple2"] if active else self.colors["line2"])
            except Exception: pass
        if hasattr(self,"player_progress_canvas"):
            try:
                pw=max(10,self.player_progress_canvas.winfo_width())
                self.player_progress_canvas.coords(self.player_progress_bg,0,2,pw,6)
                self.player_progress_canvas.coords(self.player_progress_fg,0,2,max(1,pw*self.music_progress),6)
            except Exception: pass
        if hasattr(self,"neo_info_values"):
            for lab,getter in self.neo_info_values:
                try: lab.config(text=getter())
                except Exception: pass

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


    # ============================================================
    # UNIQUE UI / UX FEATURES
    # ============================================================

    def setup_unique_features(self):
        """Подключает дополнительные визуальные эффекты и desktop-функции."""
        self.root.bind_all("<Control-k>", lambda e: self.open_command_palette())
        self.root.bind_all("<Control-K>", lambda e: self.open_command_palette())
        self.root.bind_all("<F11>", lambda e: self.toggle_fullscreen())
        self.root.bind_all("<F10>", lambda e: self.toggle_focus_mode())
        self.root.bind_all("<F6>", lambda e: self.cycle_accent_theme())
        self.root.bind_all("<Control-Shift-S>", lambda e: self.quick_screenshot())
        self.root.bind_all("<Control-Shift-V>", lambda e: self.show_clipboard_history())
        self.root.bind_all("<Control-n>", lambda e: self.open_notes())
        self.root.bind_all("<Control-N>", lambda e: self.open_notes())
        self.root.bind_all("<Control-Shift-P>", lambda e: self.open_telemetry_center())
        self.root.bind_all("<Alt-m>", lambda e: self.toggle_manual_record())
        self.root.bind_all("<Alt-M>", lambda e: self.toggle_manual_record())
        self.root.bind_all("<Escape>", self._escape_overlay)
        self.root.bind("<Motion>", self._track_mouse)
        self.root.bind_all("<Button-1>", self._spawn_click_ripple, add="+")
        self.root.bind_all("<Control-Shift-W>", lambda e: self.refresh_weather_widget(force=True))
        self.root.bind_all("<F7>", lambda e: self.toggle_cinematic_fx())

        self._build_command_registry()
        self._build_floating_dock()
        self._build_corner_hud()
        self._build_fx_layer()
        self._build_cinematic_fx_layer()
        self._load_clipboard_history()
        self._start_clipboard_watcher()
        self._start_telemetry_loop()
        self._animate_extra_fx()
        self.root.after(1400, lambda: self.refresh_weather_widget(force=False, silent=True))
        self.root.after(600000, self._schedule_weather_refresh)

        self.root.after(700, lambda: self.show_toast(
            "MITA NEO 4.0", "Интерфейс загружен • Ctrl+K — командная палитра", "success", 2800
        ))

    def _build_command_registry(self):
        self.palette_commands = [
            ("Главная", "Перейти на главную страницу", "⌂", lambda: self.switch_tab(0)),
            ("Чат с Mita", "Открыть AI-чат", "◯", lambda: self.switch_tab(1)),
            ("Команды", "Открыть список голосовых команд", "⌘", lambda: self.switch_tab(2)),
            ("Настройки", "Открыть параметры Mita", "⚙", lambda: self.switch_tab(3)),
            ("Начать запись", "Ручное голосовое распознавание", "🎙", self.toggle_manual_record),
            ("Сделать скриншот", "Сохранить снимок экрана", "▣", self.quick_screenshot),
            ("Погода рядом", "Автоматически определить", "☁", lambda: self.refresh_weather_widget(force=True, speak_result=True)),
            ("Заметки", "Быстрый блокнот Mita", "✎", self.open_notes),
            ("Буфер обмена", "История последних скопированных фрагментов", "▤", self.show_clipboard_history),
            ("Телеметрия", "CPU / RAM / диск / сеть в реальном времени", "⌁", self.open_telemetry_center),
            ("Фокус-режим", "Скрыть боковые панели и оставить главное", "◈", self.toggle_focus_mode),
            ("Полный экран", "Переключить fullscreen", "□", self.toggle_fullscreen),
            ("Поверх окон", "Закрепить Mita поверх остальных окон", "◆", self.toggle_always_on_top),
            ("Сменить тему", "Следующий неоновый accent preset", "✦", self.cycle_accent_theme),
            ("Эффекты", "Включить/выключить анимации", "✧", self.toggle_effects),
            ("Уменьшить движение", "Мягкий режим анимаций", "≈", self.toggle_reduce_motion),
            ("Таймер 25 минут", "Запустить фокус-таймер", "◷", lambda: self.start_focus_timer(25)),
            ("Таймер 5 минут", "Запустить короткий таймер", "◷", lambda: self.start_focus_timer(5)),
            ("Остановить таймер", "Сбросить активный focus timer", "×", self.stop_focus_timer),
            ("Остановить музыку", "Немедленно остановить воспроизведение", "■", self.stop_music_click),
            ("Сменить режим AI", "System / AI / All", "◇", self.change_mode),
            ("Переключить голос", "Включить или отключить TTS", "♩", self.toggle_tts_mute),
            ("Исправитель текста", "Включить или отключить корректор", "Aa", self.toggle_corrector),
            ("Очистить чат", "Удалить локальную историю текущего чата", "⌫", self.clear_chat),
        ]

    def _build_floating_dock(self):
        self.floating_dock = tk.Frame(
            self.root, bg=self.colors["panel2"],
            highlightbackground=self.colors["line2"], highlightthickness=1
        )
        self.floating_dock.place(relx=.5, rely=1, y=-74, anchor="s")
        dock_items = [
            ("⌂", "Главная", lambda: self.switch_tab(0)),
            ("⌘", "Палитра", self.open_command_palette),
            ("🎙", "Микрофон", self.toggle_manual_record),
            ("✎", "Заметки", self.open_notes),
            ("⌁", "Система", self.open_telemetry_center),
        ]
        self.dock_buttons = []
        for icon, tip, cmd in dock_items:
            b = tk.Label(
                self.floating_dock, text=icon, font=("Segoe UI Symbol", 12, "bold"),
                bg=self.colors["panel2"], fg=self.colors["muted"], cursor="hand2",
                width=3, pady=6
            )
            b.pack(side="left", padx=2, pady=2)
            b.bind("<Button-1>", lambda e, c=cmd: c())
            b.bind("<Enter>", lambda e, w=b, t=tip: self._dock_hover(w, True, t))
            b.bind("<Leave>", lambda e, w=b: self._dock_hover(w, False, ""))
            self.dock_buttons.append(b)

        self.dock_hint = tk.Label(
            self.root, text="", font=("Segoe UI", 7, "bold"),
            bg=self.colors["chip"], fg=self.colors["text"], padx=8, pady=3
        )

    def _dock_hover(self, widget, active, text):
        widget.config(
            bg=self.colors["active"] if active else self.colors["panel2"],
            fg=self.colors["purple2"] if active else self.colors["muted"]
        )
        if active:
            try:
                x = widget.winfo_rootx() - self.root.winfo_rootx() + widget.winfo_width() // 2
                y = self.root.winfo_height() - 107
                self.dock_hint.config(text=text)
                self.dock_hint.place(x=x, y=y, anchor="s")
                self.dock_hint.lift()
            except Exception:
                pass
        else:
            self.dock_hint.place_forget()

    def _build_corner_hud(self):
        self.corner_hud = tk.Frame(self.root, bg=self.colors["topbar"])
        self.corner_hud.place(relx=1, rely=1, x=-14, y=-15, anchor="se")
        self.timer_hud = tk.Label(
            self.corner_hud, text="", font=("Segoe UI", 7, "bold"),
            bg=self.colors["topbar"], fg=self.colors["purple2"]
        )
        self.timer_hud.pack(side="left", padx=(0, 10))
        self.net_hud = tk.Label(
            self.corner_hud, text="NET 0.0 Mbps", font=("Consolas", 7, "bold"),
            bg=self.colors["topbar"], fg=self.colors["muted"]
        )
        self.net_hud.pack(side="left")

    def _build_fx_layer(self):
        """Лёгкая декоративная сетка/частицы только в свободных областях окна."""
        self.fx_canvas = tk.Canvas(
            self.header, bg=self.colors["topbar"], highlightthickness=0,
            bd=0, width=170, height=56
        )
        self.fx_canvas.place(relx=.72, y=1, width=170, height=54)
        self.fx_dots = []
        for _ in range(18):
            x = random.randint(4, 166)
            y = random.randint(5, 49)
            r = random.choice((1, 1, 1, 2))
            obj = self.fx_canvas.create_oval(x-r, y-r, x+r, y+r, fill=self.colors["line2"], outline="")
            self.fx_dots.append({
                "id": obj, "x": float(x), "y": float(y),
                "speed": random.uniform(.15, .55), "phase": random.random() * math.tau
            })

    def _build_cinematic_fx_layer(self):
        """Дополнительный cinematic FX-слой: aurora, trail, meteor particles и ripples."""
        self.cinematic_canvas = tk.Canvas(
            self.bottom_bar, bg=self.colors["topbar"], highlightthickness=0, bd=0,
            width=380, height=52
        )
        self.cinematic_canvas.place(relx=.5, rely=.5, anchor="center", width=390, height=48)
        self.cinematic_canvas.tk.call('lower', self.cinematic_canvas._w)
        self._aurora_lines = []
        for i in range(7):
            obj = self.cinematic_canvas.create_line(0,24,390,24,fill=self.colors["line2"],width=1,smooth=True)
            self._aurora_lines.append(obj)

        # Маленькие «метеоры» в шапке.
        if hasattr(self, "fx_canvas"):
            for _ in range(4):
                x = random.uniform(30, 220)
                y = random.uniform(2, 35)
                length = random.uniform(10, 26)
                item = self.fx_canvas.create_line(x,y,x+length,y-4,fill=self.colors["line2"],width=1)
                self._shooting_stars.append({"id":item,"x":x,"y":y,"vx":random.uniform(-1.7,-.7),"vy":random.uniform(.08,.22),"life":random.uniform(0,1)})

    def _spawn_click_ripple(self, event):
        if not self._effects_enabled or self._reduce_motion:
            return
        try:
            # Рисуем ripple только если клик был в основном окне, а не в popup.
            if event.widget.winfo_toplevel() is not self.root:
                return
            x = event.x_root - self.root.winfo_rootx()
            y = event.y_root - self.root.winfo_rooty()
            item = self.canvas.create_oval(x-2,y-2,x+2,y+2,outline=self.colors["purple2"],width=2)
            self._ripple_items.append({"id":item,"x":x,"y":y,"r":2.0,"life":1.0})
            self.canvas.tag_raise(item)
        except Exception:
            pass

    def _animate_cinematic_fx(self):
        self._fx_frame += 1
        self._aurora_phase += .045 if not self._reduce_motion else .012
        target_energy = .85 if (self.is_listening or self.is_processing or self.is_manual_recording) else (.7 if self.music_mode else .18)
        self._ambient_energy += (target_energy - self._ambient_energy) * .06

        # Aurora waveform в нижней панели.
        if hasattr(self, "cinematic_canvas"):
            w = max(100, self.cinematic_canvas.winfo_width())
            h = max(20, self.cinematic_canvas.winfo_height())
            for layer, item in enumerate(self._aurora_lines):
                pts = []
                amp = (2.0 + layer * .8) * (1 + self._ambient_energy * 2.7)
                for x in range(0, int(w)+12, 12):
                    y = h/2 + math.sin(x*.028 + self._aurora_phase*(1.0+layer*.08) + layer*.8)*amp
                    y += math.sin(x*.071 - self._aurora_phase*.6 + layer)*amp*.24
                    pts.extend((x,y))
                try:
                    self.cinematic_canvas.coords(item,*pts)
                    self.cinematic_canvas.itemconfig(item,fill=self.colors["purple2"] if layer in (2,3,4) else self.colors["line2"],width=1 if layer not in (3,) else 2)
                except Exception:
                    pass

        # Ripples плавно расширяются и исчезают.
        kept=[]
        for rp in self._ripple_items:
            rp["r"] += 2.5 + self._ambient_energy*1.2
            rp["life"] -= .045
            if rp["life"] <= 0:
                try:self.canvas.delete(rp["id"])
                except Exception:pass
                continue
            r=rp["r"]
            try:
                self.canvas.coords(rp["id"],rp["x"]-r,rp["y"]-r,rp["x"]+r,rp["y"]+r)
                self.canvas.itemconfig(rp["id"],width=max(1,int(rp["life"]*2)))
            except Exception:
                pass
            kept.append(rp)
        self._ripple_items=kept[-12:]

        # Метеоры.
        if hasattr(self,"fx_canvas"):
            fw=max(50,self.fx_canvas.winfo_width()); fh=max(30,self.fx_canvas.winfo_height())
            for st in self._shooting_stars:
                st["x"] += st["vx"] * (1.0 + self._ambient_energy*.7)
                st["y"] += st["vy"]
                st["life"] -= .008
                if st["x"] < -30 or st["y"] > fh+10 or st["life"] <= 0:
                    st["x"] = fw + random.uniform(10,90); st["y"] = random.uniform(1,22)
                    st["life"] = random.uniform(.55,1.2); st["vx"] = random.uniform(-1.8,-.8)
                try:
                    self.fx_canvas.coords(st["id"],st["x"],st["y"],st["x"]+18,st["y"]-4)
                    self.fx_canvas.itemconfig(st["id"],fill=self.colors["purple2"] if self._ambient_energy>.45 else self.colors["line2"])
                except Exception:
                    pass

    def toggle_cinematic_fx(self):
        self._effects_enabled = not self._effects_enabled
        self.show_toast("Cinematic FX", "Эффекты включены" if self._effects_enabled else "Эффекты выключены", "success" if self._effects_enabled else "warning", 1800)

    def refresh_weather_widget(self, force=False, silent=False, speak_result=False):
        if self._weather_loading:
            return
        self._weather_loading = True
        if hasattr(self,"weather_main_label"):
            self.weather_main_label.config(text="Определяю город и получаю погоду…",fg=self.colors["purple2"])
        if hasattr(self,"weather_sub_label"):
            self.weather_sub_label.config(text="IP GEO → координаты → Open-Meteo")
        def worker():
            result = get_local_weather(force=force, lang=("ua" if UI_LANGUAGE=="ua" else "ru"))
            self.root.after(0, lambda r=result: self._apply_weather_result(r, silent=silent, speak_result=speak_result))
        threading.Thread(target=worker,daemon=True).start()

    def _apply_weather_result(self, result, silent=False, speak_result=False):
        self._weather_loading = False
        if not result.get("ok"):
            msg=result.get("error") or "Ошибка погоды"
            if hasattr(self,"weather_main_label"): self.weather_main_label.config(text=msg,fg="#ff8a9e")
            if hasattr(self,"weather_sub_label"): self.weather_sub_label.config(text=result.get("details","")[:120])
            if not silent:self.show_toast("Погода",msg,"danger")
            return
        loc=result["location"]; data=result["weather"]; cur=data.get("current") or {}
        desc, icon=_weather_code_info(cur.get("weather_code"), "ua" if UI_LANGUAGE=="ua" else "ru")
        city=loc.get("city") or "—"; temp=cur.get("temperature_2m"); hum=cur.get("relative_humidity_2m"); wind=cur.get("wind_speed_10m")
        try: t=f"{float(temp):.0f}°C"
        except: t="—°C"
        if hasattr(self,"weather_main_label"):
            self.weather_main_label.config(text=f"{icon} {city}  •  {t}  •  {desc}",fg=self.colors["text"])
        if hasattr(self,"weather_sub_label"):
            self.weather_sub_label.config(text=f"Влажность {hum}% • ветер {wind} км/ч •")
        if hasattr(self,"weather_orb_text"):
            try:self.weather_orb.itemconfig(self.weather_orb_text,text=icon)
            except Exception:pass
        if hasattr(self, "weather_updated_label"):
            try:
                self.weather_updated_label.config(text=f"ОБНОВЛЕНО • {datetime.now().strftime('%H:%M')}")
            except Exception:
                pass
        if not silent:self.show_toast("Погода обновлена",f"{city}: {t}, {desc}","success",2400)
        if speak_result:
            speak(result["report"],force=True)

    def _schedule_weather_refresh(self):
        if not self.running:
            return
        self.refresh_weather_widget(force=True, silent=True)
        self.root.after(600000, self._schedule_weather_refresh)

    def get_weather_for_chat(self, force=False):
        result=get_local_weather(force=force,lang=("ua" if UI_LANGUAGE=="ua" else "ru"))
        try:self.root.after(0,lambda r=result:self._apply_weather_result(r,silent=True,speak_result=False))
        except Exception:pass
        if result.get("ok"):return result["report"]
        return result.get("error") or T("error") + " weather"

    def _track_mouse(self, event):
        try:
            self._mouse_x = event.x_root - self.root.winfo_rootx()
            self._mouse_y = event.y_root - self.root.winfo_rooty()
        except Exception:
            pass

    def _animate_extra_fx(self):
        if not self.running:
            return
        self._glow_phase += .055 if not self._reduce_motion else .018
        self._logo_phase += .07 if not self._reduce_motion else .02

        if hasattr(self, "logo_canvas") and self._effects_enabled:
            pulse = (math.sin(self._logo_phase) + 1) / 2
            try:
                # имитация glow через изменение толщины и размеров колец
                pad = 2 + pulse * 2
                self.logo_canvas.coords(self.logo_glow_outer, pad, pad, 34-pad, 34-pad)
                self.logo_canvas.itemconfig(self.logo_glow_outer, width=1 + int(pulse * 2), outline=self.colors["line2"])
                self.logo_canvas.itemconfig(self.logo_glow_mid, outline=self.colors["purple2"])
                self.logo_canvas.itemconfig(self.logo_core, fill=self.colors["purple"])
            except Exception:
                pass

        if hasattr(self, "shield_canvas") and self._effects_enabled:
            try:
                cx = 48 + math.sin(self._glow_phase * .8) * 2
                cy = 48 + math.cos(self._glow_phase * .9) * 2
                self.shield_canvas.coords(5, cx-6, cy-6, cx+6, cy+6)
            except Exception:
                pass

        if hasattr(self, "fx_canvas"):
            w = max(20, self.fx_canvas.winfo_width())
            for i, dot in enumerate(self.fx_dots):
                if self._effects_enabled:
                    dot["x"] -= dot["speed"]
                    dot["y"] += math.sin(self._glow_phase + dot["phase"]) * .08
                    if dot["x"] < 0:
                        dot["x"] = w + random.randint(0, 20)
                        dot["y"] = random.randint(5, 49)
                    a = (math.sin(self._glow_phase * 1.3 + dot["phase"]) + 1) / 2
                    col = self.colors["purple2"] if a > .68 else self.colors["line2"]
                    self.fx_canvas.itemconfig(dot["id"], fill=col)
                self.fx_canvas.coords(dot["id"], dot["x"]-1, dot["y"]-1, dot["x"]+1, dot["y"]+1)

        self._animate_cinematic_fx()

        if hasattr(self,"weather_orb") and self._effects_enabled:
            try:
                self._weather_icon_phase += .05 if not self._reduce_motion else .015
                p=(math.sin(self._weather_icon_phase)+1)/2
                pad=4+p*3
                self.weather_orb.coords(self.weather_orb_ring,pad,pad,50-pad,50-pad)
                self.weather_orb.itemconfig(self.weather_orb_ring,width=1+int(p*2),outline=self.colors["purple2"] if p>.55 else self.colors["line2"])
            except Exception:pass

        self._update_focus_timer_hud()
        self.root.after(35 if not self._reduce_motion else 85, self._animate_extra_fx)

    # -------------------- Toast system --------------------

    def show_toast(self, title, message, kind="info", duration=3200):
        palette = {
            "info": self.colors["purple2"],
            "success": self.colors["green"],
            "warning": "#ffbe63",
            "danger": "#ff6584",
        }
        accent = palette.get(kind, self.colors["purple2"])
        toast = tk.Frame(
            self.root, bg=self.colors["panel"],
            highlightbackground=accent, highlightthickness=1
        )
        toast.place(relx=1, x=-24, y=76 + len(self.toast_stack) * 82, anchor="ne", width=330, height=70)
        side = tk.Frame(toast, bg=accent, width=4); side.pack(side="left", fill="y")
        body = tk.Frame(toast, bg=self.colors["panel"]); body.pack(side="left", fill="both", expand=True, padx=12, pady=8)
        tk.Label(body, text=title, font=("Segoe UI", 9, "bold"), bg=self.colors["panel"], fg=self.colors["text"]).pack(anchor="w")
        tk.Label(body, text=message, font=("Segoe UI", 7), bg=self.colors["panel"], fg=self.colors["muted"], wraplength=285, justify="left").pack(anchor="w", pady=(3,0))
        close = tk.Label(toast, text="×", font=("Segoe UI", 10, "bold"), bg=self.colors["panel"], fg=self.colors["muted"], cursor="hand2")
        close.place(relx=1, x=-8, y=5, anchor="ne")
        close.bind("<Button-1>", lambda e, t=toast: self._close_toast(t))
        self.toast_stack.append(toast)
        toast.lift()
        self.root.after(duration, lambda t=toast: self._close_toast(t))

    def _close_toast(self, toast):
        if toast not in self.toast_stack:
            return
        try:
            toast.destroy()
        except Exception:
            pass
        try:
            self.toast_stack.remove(toast)
        except ValueError:
            pass
        for i, item in enumerate(self.toast_stack):
            try:
                item.place_configure(y=76 + i * 82)
            except Exception:
                pass

    # -------------------- Command palette --------------------

    def open_command_palette(self):
        if self.command_palette is not None:
            try:
                self.command_palette.lift(); self.palette_entry.focus_set()
                return
            except Exception:
                self.command_palette = None

        win = tk.Toplevel(self.root)
        self.command_palette = win
        win.title("Mita Command Palette")
        win.geometry("680x560")
        win.configure(bg=self.colors["bg"])
        win.transient(self.root)
        win.attributes("-topmost", True)
        try: win.attributes("-alpha", .98)
        except Exception: pass
        win.protocol("WM_DELETE_WINDOW", self.close_command_palette)

        shell = tk.Frame(win, bg=self.colors["line2"])
        shell.pack(fill="both", expand=True, padx=1, pady=1)
        body = tk.Frame(shell, bg=self.colors["panel2"])
        body.pack(fill="both", expand=True, padx=1, pady=1)

        top = tk.Frame(body, bg=self.colors["panel2"])
        top.pack(fill="x", padx=18, pady=(18, 10))
        tk.Label(top, text="⌘", font=("Segoe UI Symbol", 16, "bold"), bg=self.colors["panel2"], fg=self.colors["purple2"]).pack(side="left", padx=(0,10))
        self.palette_entry = tk.Entry(
            top, textvariable=self.palette_query, font=("Segoe UI", 12),
            bg=self.colors["panel"], fg=self.colors["text"], insertbackground=self.colors["purple2"],
            relief="flat", bd=0
        )
        self.palette_entry.pack(side="left", fill="x", expand=True, ipady=10, padx=(0,10))
        tk.Label(top, text="ESC", font=("Consolas", 7, "bold"), bg=self.colors["chip"], fg=self.colors["muted"], padx=8, pady=5).pack(side="right")

        self.palette_status = tk.Label(body, text="", font=("Segoe UI", 7), bg=self.colors["panel2"], fg=self.colors["muted"])
        self.palette_status.pack(anchor="w", padx=20)

        self.palette_list = tk.Frame(body, bg=self.colors["panel2"])
        self.palette_list.pack(fill="both", expand=True, padx=14, pady=(8,14))
        self.palette_query.set("")
        self.palette_query.trace_add("write", lambda *_: self._refresh_palette())
        self.palette_entry.bind("<Return>", lambda e: self._run_palette_first())
        self.palette_entry.bind("<Down>", lambda e: self._palette_select_delta(1))
        self.palette_entry.bind("<Up>", lambda e: self._palette_select_delta(-1))
        self.palette_entry.focus_set()
        self._palette_selected = 0
        self._refresh_palette()

    def close_command_palette(self):
        if self.command_palette is not None:
            try: self.command_palette.destroy()
            except Exception: pass
        self.command_palette = None

    def _palette_score(self, query, title, desc):
        q = query.lower().strip()
        if not q:
            return 100
        t = title.lower(); d = desc.lower()
        if q == t: return 1000
        if t.startswith(q): return 800
        if q in t: return 650
        if q in d: return 420
        words = [w for w in q.split() if w]
        score = sum(90 for w in words if w in t) + sum(35 for w in words if w in d)
        return score

    def _refresh_palette(self):
        if self.command_palette is None:
            return
        for child in self.palette_list.winfo_children():
            child.destroy()
        q = self.palette_query.get()
        ranked = []
        for item in self.palette_commands:
            score = self._palette_score(q, item[0], item[1])
            if score > 0:
                ranked.append((score, item))
        ranked.sort(key=lambda x: x[0], reverse=True)
        self._palette_visible = [it for _, it in ranked[:9]]
        self._palette_selected = min(self._palette_selected, max(0, len(self._palette_visible)-1))
        self.palette_status.config(text=f"{len(self._palette_visible)} быстрых действий • Enter — выполнить")

        for i, item in enumerate(self._palette_visible):
            title, desc, icon, cmd = item
            active = i == self._palette_selected
            bg = self.colors["active"] if active else self.colors["panel"]
            row = tk.Frame(self.palette_list, bg=bg, cursor="hand2", highlightthickness=1, highlightbackground=self.colors["line2"] if active else self.colors["panel"])
            row.pack(fill="x", pady=3, ipady=6)
            ic = tk.Label(row, text=icon, width=4, font=("Segoe UI Symbol", 11, "bold"), bg=bg, fg=self.colors["purple2"])
            ic.pack(side="left", padx=(6,4))
            text = tk.Frame(row, bg=bg); text.pack(side="left", fill="x", expand=True)
            tk.Label(text, text=title, font=("Segoe UI", 9, "bold"), bg=bg, fg=self.colors["text"]).pack(anchor="w")
            tk.Label(text, text=desc, font=("Segoe UI", 7), bg=bg, fg=self.colors["muted"]).pack(anchor="w")
            hot = tk.Label(row, text="RUN", font=("Consolas", 6, "bold"), bg=self.colors["chip"], fg=self.colors["muted"], padx=6, pady=3)
            hot.pack(side="right", padx=10)
            for w in (row, ic, text, hot) + tuple(text.winfo_children()):
                w.bind("<Button-1>", lambda e, c=cmd: self._run_palette_command(c))
                w.bind("<Enter>", lambda e, r=row: r.config(bg=self.colors["active"]))

    def _palette_select_delta(self, delta):
        if not getattr(self, "_palette_visible", None):
            return "break"
        self._palette_selected = (self._palette_selected + delta) % len(self._palette_visible)
        self._refresh_palette()
        return "break"

    def _run_palette_first(self):
        if not getattr(self, "_palette_visible", None):
            return
        idx = max(0, min(self._palette_selected, len(self._palette_visible)-1))
        self._run_palette_command(self._palette_visible[idx][3])

    def _run_palette_command(self, command):
        self.close_command_palette()
        try:
            command()
        except Exception as e:
            self.show_toast("Ошибка", str(e), "danger")

    def _escape_overlay(self, event=None):
        if self.command_palette is not None:
            self.close_command_palette(); return "break"
        if self.fullscreen_mode:
            self.toggle_fullscreen(); return "break"
        return None

    # -------------------- Window modes --------------------

    def toggle_fullscreen(self):
        self.fullscreen_mode = not self.fullscreen_mode
        try: self.root.attributes("-fullscreen", self.fullscreen_mode)
        except Exception: pass
        self.show_toast("Полный экран", "Включён" if self.fullscreen_mode else "Выключен", "info", 1800)

    def toggle_always_on_top(self):
        self.always_on_top = not self.always_on_top
        try: self.root.attributes("-topmost", self.always_on_top)
        except Exception: pass
        self.show_toast("Поверх окон", "Mita закреплена" if self.always_on_top else "Обычный режим", "success" if self.always_on_top else "info", 1800)

    def toggle_focus_mode(self):
        self.focus_mode = not self.focus_mode
        if self.focus_mode:
            try:
                self.sidebar.pack_forget()
                self.rightbar.pack_forget()
                self.floating_dock.place_forget()
            except Exception:
                pass
            self.show_toast("Focus Mode", "Боковые панели скрыты • F10 для выхода", "success", 2200)
        else:
            try:
                self.sidebar.pack(side="left", fill="y", before=self.content)
                self.rightbar.pack(side="right", fill="y", padx=(0,12), pady=(12,8), after=self.content)
                self.floating_dock.place(relx=.5, rely=1, y=-74, anchor="s")
            except Exception:
                # надёжный fallback: пересобираем порядок pack без уничтожения виджетов
                try:
                    self.sidebar.pack(side="left", fill="y")
                    self.content.pack(side="left", fill="both", expand=True)
                    self.rightbar.pack(side="right", fill="y", padx=(0,12), pady=(12,8))
                except Exception:
                    pass
            self.show_toast("Focus Mode", "Обычный интерфейс восстановлен", "info", 1800)

    def toggle_effects(self):
        self._effects_enabled = not self._effects_enabled
        self.show_toast("Эффекты", "Анимации включены" if self._effects_enabled else "Анимации отключены", "info", 1800)

    def toggle_reduce_motion(self):
        self._reduce_motion = not self._reduce_motion
        self.show_toast("Motion", "Мягкая анимация" if self._reduce_motion else "Полная анимация", "info", 1800)

    # -------------------- Accent themes --------------------

    def cycle_accent_theme(self):
        self._accent_index = (self._accent_index + 1) % len(self._accent_presets)
        name, primary, primary2, primary3 = self._accent_presets[self._accent_index]
        self.colors["primary"] = primary
        self.colors["purple"] = primary
        self.colors["primary2"] = primary2
        self.colors["purple2"] = primary2
        self.colors["purple3"] = primary3
        self.colors["line2"] = primary
        self._apply_accent_recursive(self.root)
        try:
            self.logo_canvas.itemconfig(self.logo_glow_mid, outline=primary)
            self.logo_canvas.itemconfig(self.logo_core, fill=primary2, outline=primary3)
        except Exception:
            pass
        self.show_toast("NEON THEME", f"Акцент: {name}", "success", 2200)

    def _apply_accent_recursive(self, widget):
        """Аккуратно перекрашивает только явно фиолетовые элементы, не ломая фон."""
        try:
            cls = widget.winfo_class()
            if cls in ("Label", "Button"):
                fg = widget.cget("fg")
                old_accents = {"#7c4dff", "#a978ff", "#c7a8ff", self.colors.get("primary"), self.colors.get("primary2")}
                if fg in old_accents or "purple" in str(fg).lower():
                    widget.config(fg=self.colors["purple2"])
            if cls == "Scale":
                widget.config(activebackground=self.colors["purple2"])
        except Exception:
            pass
        try:
            for child in widget.winfo_children():
                self._apply_accent_recursive(child)
        except Exception:
            pass

    # -------------------- Screenshot --------------------

    def quick_screenshot(self):
        try:
            folder = os.path.join(BASE_DIR, "Screenshots")
            os.makedirs(folder, exist_ok=True)
            name = datetime.now().strftime("mita_%Y-%m-%d_%H-%M-%S.png")
            path = os.path.join(folder, name)
            pyautogui.screenshot(path)
            self._add_recent_ui_action("Скриншот", path)
            self.show_toast("Скриншот сохранён", name, "success")
        except Exception as e:
            self.show_toast("Скриншот", f"Не удалось сохранить: {e}", "danger")

    # -------------------- Clipboard history --------------------

    def _clipboard_file(self):
        return os.path.join(BASE_DIR, "mita_clipboard_history.json")

    def _load_clipboard_history(self):
        try:
            path = self._clipboard_file()
            if os.path.exists(path):
                data = json.loads(Path(path).read_text(encoding="utf-8"))
                if isinstance(data, list):
                    self.clipboard_history = [str(x) for x in data[:30]]
        except Exception:
            self.clipboard_history = []

    def _save_clipboard_history(self):
        try:
            Path(self._clipboard_file()).write_text(json.dumps(self.clipboard_history[:30], ensure_ascii=False, indent=2), encoding="utf-8")
        except Exception:
            pass

    def _start_clipboard_watcher(self):
        self._poll_clipboard()

    def _poll_clipboard(self):
        if not self.running:
            return
        try:
            value = self.root.clipboard_get()
            value = str(value).strip()
            if value and value != self._last_clipboard_value and len(value) <= 20000:
                self._last_clipboard_value = value
                if value in self.clipboard_history:
                    self.clipboard_history.remove(value)
                self.clipboard_history.insert(0, value)
                self.clipboard_history = self.clipboard_history[:30]
                self._save_clipboard_history()
        except Exception:
            pass
        self.root.after(1200, self._poll_clipboard)

    def show_clipboard_history(self):
        if self.clipboard_window is not None:
            try: self.clipboard_window.destroy()
            except Exception: pass
        win = tk.Toplevel(self.root); self.clipboard_window = win
        win.title("Mita Clipboard")
        win.geometry("620x520")
        win.configure(bg=self.colors["bg"])
        win.transient(self.root)
        win.attributes("-topmost", True)
        win.protocol("WM_DELETE_WINDOW", lambda: self._close_clipboard_window())

        tk.Label(win, text="БУФЕР ОБМЕНА", font=("Segoe UI", 15, "bold"), bg=self.colors["bg"], fg=self.colors["text"]).pack(anchor="w", padx=22, pady=(20,2))
        tk.Label(win, text="Нажмите на карточку, чтобы вернуть текст в буфер", font=("Segoe UI", 8), bg=self.colors["bg"], fg=self.colors["muted"]).pack(anchor="w", padx=22, pady=(0,12))
        outer = tk.Frame(win, bg=self.colors["bg"]); outer.pack(fill="both", expand=True, padx=18, pady=(0,18))
        canvas = tk.Canvas(outer, bg=self.colors["bg"], highlightthickness=0)
        scroll = ttk.Scrollbar(outer, orient="vertical", command=canvas.yview)
        inner = tk.Frame(canvas, bg=self.colors["bg"])
        inner.bind("<Configure>", lambda e: canvas.configure(scrollregion=canvas.bbox("all")))
        canvas.create_window((0,0), window=inner, anchor="nw", width=560)
        canvas.configure(yscrollcommand=scroll.set)
        canvas.pack(side="left", fill="both", expand=True); scroll.pack(side="right", fill="y")

        if not self.clipboard_history:
            tk.Label(inner, text="История пока пустая", font=("Segoe UI", 9), bg=self.colors["bg"], fg=self.colors["muted"]).pack(pady=30)
        for i, text in enumerate(self.clipboard_history[:20]):
            preview = text.replace("\n", " ↵ ")
            if len(preview) > 140: preview = preview[:137] + "..."
            row = tk.Frame(inner, bg=self.colors["panel"], highlightbackground=self.colors["border"], highlightthickness=1, cursor="hand2")
            row.pack(fill="x", pady=4, padx=2)
            tk.Label(row, text=f"{i+1:02d}", font=("Consolas", 7, "bold"), bg=self.colors["chip"], fg=self.colors["purple2"], width=4, pady=10).pack(side="left", padx=(7,10), pady=7)
            label = tk.Label(row, text=preview, font=("Segoe UI", 8), bg=self.colors["panel"], fg=self.colors["text"], justify="left", anchor="w", wraplength=430)
            label.pack(side="left", fill="x", expand=True, padx=(0,8), pady=8)
            for w in (row, label):
                w.bind("<Button-1>", lambda e, v=text: self._restore_clipboard(v))

    def _close_clipboard_window(self):
        if self.clipboard_window is not None:
            try: self.clipboard_window.destroy()
            except Exception: pass
        self.clipboard_window = None

    def _restore_clipboard(self, text):
        try:
            self.root.clipboard_clear(); self.root.clipboard_append(text); self.root.update()
            self._last_clipboard_value = text
            self.show_toast("Буфер обмена", "Фрагмент восстановлен", "success", 1600)
            self._close_clipboard_window()
        except Exception as e:
            self.show_toast("Буфер обмена", str(e), "danger")

    # -------------------- Notes --------------------

    def _notes_file(self):
        return os.path.join(BASE_DIR, "mita_notes.txt")

    def open_notes(self):
        if self.notes_window is not None:
            try: self.notes_window.lift(); return
            except Exception: self.notes_window = None
        win = tk.Toplevel(self.root); self.notes_window = win
        win.title("Mita Notes")
        win.geometry("720x560")
        win.configure(bg=self.colors["bg"])
        win.transient(self.root)
        win.protocol("WM_DELETE_WINDOW", self._close_notes)

        top = tk.Frame(win, bg=self.colors["bg"]); top.pack(fill="x", padx=20, pady=(18,10))
        tk.Label(top, text="MITA NOTES", font=("Segoe UI", 15, "bold"), bg=self.colors["bg"], fg=self.colors["text"]).pack(side="left")
        self.notes_status = tk.Label(top, text="AUTO SAVE", font=("Consolas", 7, "bold"), bg=self.colors["chip"], fg=self.colors["green"], padx=8, pady=4)
        self.notes_status.pack(side="right")

        self.notes_text = tk.Text(
            win, bg=self.colors["panel"], fg=self.colors["text"],
            insertbackground=self.colors["purple2"], selectbackground=self.colors["purple"],
            font=("Segoe UI", 10), wrap="word", bd=0, padx=16, pady=14
        )
        self.notes_text.pack(fill="both", expand=True, padx=20, pady=(0,10))
        try:
            if os.path.exists(self._notes_file()):
                self.notes_text.insert("1.0", Path(self._notes_file()).read_text(encoding="utf-8"))
        except Exception:
            pass
        self.notes_text.bind("<KeyRelease>", lambda e: self._schedule_notes_save())
        bottom = tk.Frame(win, bg=self.colors["bg"]); bottom.pack(fill="x", padx=20, pady=(0,18))
        self._button(bottom, "СОХРАНИТЬ", self._save_notes_now, accent=True).pack(side="left")
        self._button(bottom, "ОЧИСТИТЬ", self._clear_notes).pack(side="left", padx=8)
        self._button(bottom, "ЗАКРЫТЬ", self._close_notes).pack(side="right")

    def _schedule_notes_save(self):
        try:
            if hasattr(self, "_notes_after") and self._notes_after:
                self.root.after_cancel(self._notes_after)
        except Exception:
            pass
        self.notes_status.config(text="EDITING...", fg=self.colors["purple2"])
        self._notes_after = self.root.after(700, self._save_notes_now)

    def _save_notes_now(self):
        if self.notes_window is None:
            return
        try:
            text = self.notes_text.get("1.0", "end-1c")
            Path(self._notes_file()).write_text(text, encoding="utf-8")
            self.notes_status.config(text="SAVED", fg=self.colors["green"])
        except Exception as e:
            self.notes_status.config(text="SAVE ERROR", fg="#ff6584")
            self.show_toast("Заметки", str(e), "danger")

    def _clear_notes(self):
        if self.notes_window is None: return
        self.notes_text.delete("1.0", "end")
        self._save_notes_now()

    def _close_notes(self):
        try: self._save_notes_now()
        except Exception: pass
        if self.notes_window is not None:
            try: self.notes_window.destroy()
            except Exception: pass
        self.notes_window = None

    # -------------------- Telemetry center --------------------

    def _start_telemetry_loop(self):
        self._poll_telemetry()

    def _poll_telemetry(self):
        if not self.running:
            return
        try:
            cpu = float(psutil.cpu_percent(interval=None))
            ram = float(psutil.virtual_memory().percent)
            disk = float(psutil.disk_usage(os.path.abspath(os.sep)).percent)
            for key, val in (("cpu",cpu),("ram",ram),("disk",disk)):
                self.telemetry_history[key].append(val)
                self.telemetry_history[key] = self.telemetry_history[key][-70:]

            net = psutil.net_io_counters()
            total = net.bytes_sent + net.bytes_recv
            now = time.time()
            if self._net_last_bytes is not None:
                dt = max(.1, now - self._net_last_time)
                self._net_speed_mbps = max(0.0, (total - self._net_last_bytes) * 8 / dt / 1_000_000)
            self._net_last_bytes = total
            self._net_last_time = now

            if hasattr(self, "net_hud"):
                self.net_hud.config(text=f"NET {self._net_speed_mbps:.1f} Mbps")
            if self.telemetry_window is not None:
                self._draw_telemetry_graphs(cpu, ram, disk)
        except Exception:
            pass
        self.root.after(1000, self._poll_telemetry)

    def open_telemetry_center(self):
        if self.telemetry_window is not None:
            try: self.telemetry_window.lift(); return
            except Exception: self.telemetry_window = None
        win = tk.Toplevel(self.root); self.telemetry_window = win
        win.title("Mita Live Telemetry")
        win.geometry("760x560")
        win.configure(bg=self.colors["bg"])
        win.transient(self.root)
        win.protocol("WM_DELETE_WINDOW", self._close_telemetry)

        tk.Label(win, text="LIVE SYSTEM TELEMETRY", font=("Segoe UI", 15, "bold"), bg=self.colors["bg"], fg=self.colors["text"]).pack(anchor="w", padx=22, pady=(20,2))
        self.telemetry_sub = tk.Label(win, text="Обновление каждую секунду", font=("Segoe UI", 8), bg=self.colors["bg"], fg=self.colors["muted"])
        self.telemetry_sub.pack(anchor="w", padx=22, pady=(0,12))
        self.telemetry_cards = tk.Frame(win, bg=self.colors["bg"]); self.telemetry_cards.pack(fill="x", padx=18)
        self.telemetry_labels = {}
        for i, key in enumerate(("CPU", "RAM", "DISK")):
            card = tk.Frame(self.telemetry_cards, bg=self.colors["panel"], highlightbackground=self.colors["border"], highlightthickness=1)
            card.grid(row=0, column=i, sticky="nsew", padx=4, pady=4); self.telemetry_cards.columnconfigure(i, weight=1)
            tk.Label(card, text=key, font=("Consolas", 8, "bold"), bg=self.colors["panel"], fg=self.colors["muted"]).pack(anchor="w", padx=12, pady=(10,2))
            val = tk.Label(card, text="0%", font=("Segoe UI", 20, "bold"), bg=self.colors["panel"], fg=self.colors["purple2"])
            val.pack(anchor="w", padx=12, pady=(0,10)); self.telemetry_labels[key.lower()] = val

        self.telemetry_graph = tk.Canvas(win, bg=self.colors["panel2"], highlightbackground=self.colors["border"], highlightthickness=1)
        self.telemetry_graph.pack(fill="both", expand=True, padx=22, pady=14)
        self.telemetry_graph.bind("<Configure>", lambda e: self._draw_telemetry_graphs())

        bottom = tk.Frame(win, bg=self.colors["bg"]); bottom.pack(fill="x", padx=22, pady=(0,18))
        self.telemetry_net = tk.Label(bottom, text="Network: 0.0 Mbps", font=("Consolas", 8, "bold"), bg=self.colors["bg"], fg=self.colors["green"])
        self.telemetry_net.pack(side="left")
        self._button(bottom, "ЗАКРЫТЬ", self._close_telemetry).pack(side="right")
        self._draw_telemetry_graphs()

    def _draw_telemetry_graphs(self, cpu=None, ram=None, disk=None):
        if self.telemetry_window is None or not hasattr(self, "telemetry_graph"):
            return
        c = self.telemetry_graph
        c.delete("all")
        w = max(200, c.winfo_width()); h = max(180, c.winfo_height())
        for i in range(1,5):
            y = h * i / 5
            c.create_line(0,y,w,y,fill=self.colors["border"],width=1)
            c.create_text(8,y-8,text=f"{100-i*20}%",anchor="w",fill=self.colors["dim"],font=("Consolas",6))
        series = [
            ("cpu", self.colors["purple2"], "CPU"),
            ("ram", self.colors["green"], "RAM"),
            ("disk", "#ffb65c", "DISK"),
        ]
        for key, color, label in series:
            vals = self.telemetry_history.get(key, [])
            if len(vals) >= 2:
                pts=[]
                for i,v in enumerate(vals):
                    x = i * (w-20) / max(1, len(vals)-1) + 10
                    y = h - 10 - (max(0,min(100,v))/100)*(h-30)
                    pts.extend((x,y))
                c.create_line(*pts, fill=color, width=2, smooth=True)
        c.create_text(w-145, 12, text="● CPU", fill=self.colors["purple2"], font=("Consolas",7,"bold"))
        c.create_text(w-90, 12, text="● RAM", fill=self.colors["green"], font=("Consolas",7,"bold"))
        c.create_text(w-34, 12, text="● DISK", fill="#ffb65c", font=("Consolas",7,"bold"))

        try:
            current = {
                "cpu": self.telemetry_history["cpu"][-1],
                "ram": self.telemetry_history["ram"][-1],
                "disk": self.telemetry_history["disk"][-1],
            }
            for k,v in current.items(): self.telemetry_labels[k].config(text=f"{v:.0f}%")
            self.telemetry_net.config(text=f"Network: {self._net_speed_mbps:.2f} Mbps")
        except Exception:
            pass

    def _close_telemetry(self):
        if self.telemetry_window is not None:
            try: self.telemetry_window.destroy()
            except Exception: pass
        self.telemetry_window = None

    # -------------------- Focus timer --------------------

    def start_focus_timer(self, minutes=25):
        try: minutes = max(1, int(minutes))
        except Exception: minutes = 25
        self.focus_timer_end = time.time() + minutes * 60
        self.focus_timer_running = True
        self.show_toast("Focus Timer", f"Запущен на {minutes} мин.", "success", 2000)

    def stop_focus_timer(self):
        self.focus_timer_running = False
        self.focus_timer_end = None
        if hasattr(self, "timer_hud"): self.timer_hud.config(text="")
        self.show_toast("Focus Timer", "Таймер остановлен", "info", 1600)

    def _update_focus_timer_hud(self):
        if not self.focus_timer_running or not self.focus_timer_end:
            return
        left = int(self.focus_timer_end - time.time())
        if left <= 0:
            self.focus_timer_running = False; self.focus_timer_end = None
            if hasattr(self, "timer_hud"): self.timer_hud.config(text="")
            self.show_toast("Focus Timer", "Время закончилось ✦", "success", 5000)
            try: self.root.bell()
            except Exception: pass
            return
        mm, ss = divmod(left, 60)
        if hasattr(self, "timer_hud"):
            self.timer_hud.config(text=f"FOCUS {mm:02d}:{ss:02d}")

    # -------------------- Recent UI actions --------------------

    def _add_recent_ui_action(self, title, detail=""):
        item = {"time": datetime.now().strftime("%H:%M:%S"), "title": title, "detail": detail}
        self.recent_ui_actions.insert(0, item)
        self.recent_ui_actions = self.recent_ui_actions[:20]
        try:
            if hasattr(self, "event_log"):
                self.event_log.config(state="normal")
                self.event_log.insert("1.0", f"{item['time']}  {title}\n")
                self.event_log.config(state="disabled")
        except Exception:
            pass

    # -------------------- controls --------------------

    def toggle_language(self):
        global UI_LANGUAGE
        UI_LANGUAGE="ua" if UI_LANGUAGE=="ru" else "ru"
        _save_ui_language()
        self.refresh_ui()

    def refresh_ui(self):
        self.root.title("Mita — Голосовой ассистент")
        self.mode_button.config(text=T("change_mode"))
        self.manual_record_button.config(text="[ MIC ]  "+T("manual_input_btn"))
        self.tts_mute_button.config(text="[ VOICE ]  "+(T("voice_on") if not self.tts_muted else T("voice_off")))
        self.corrector_button.config(text="[ TEXT ]  "+(T("corrector_on") if _text_corrector_enabled else T("corrector_off")))
        self.lang_button.config(text="SWITCH → "+("УКРАЇНСЬКА" if UI_LANGUAGE=="ru" else "РУССКИЙ"))
        self.change_key_btn.config(text=T("change_key"))
        self.music_stop_button.config(text=T("music_stop"))
        self.volume_label.config(text=f"🔊 {T('volume')}: {_music_volume}%")
        nav_names = [
            T("nav_main"), "Чат с Mita", "Управление ПК", T("nav_commands"),
            "Сценарии", "Память", "Плагины", T("nav_settings")
        ]
        for i,(row,icon,label,marker) in enumerate(self.nav_items):
            if i < len(nav_names):
                label.config(text=nav_names[i])
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

            # Ручной голосовой ввод тоже AI-FIRST.
            if _mita_mode in [MODE_AI,MODE_ALL]:
                try:
                    if process_smart_ai_command(text,self):
                        self._reset_manual_state(); return
                except Exception as e:
                    print(f"[Mita Brain] Manual AI-FIRST error: {e}")

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
        if hasattr(self,"neo_music_stop"):
            self.neo_music_stop.config(state=tk.NORMAL if is_playing else tk.DISABLED)
        if not is_playing:
            self.clear_now_playing()

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
        if hasattr(self,"neo_volume_label"): self.neo_volume_label.config(text=f"{_music_volume}%")
        if hasattr(self,"player_meta") and self.now_playing_title:
            self.player_meta.config(text=f"VLC STREAM  •  {_music_volume}%  •  LIVE AUDIO")
        if self.music_mode and hasattr(self,"status_text"): self.status_text.config(text=f"♫  МУЗЫКА ИГРАЕТ  •  {_music_volume}%")

    # -------------------- NEO live UI --------------------

    def _ui(self, fn):
        """Safely execute UI work from voice/music worker threads."""
        try:
            self.root.after(0, fn)
        except Exception:
            pass

    def show_live_speech(self, text, partial=False):
        value = str(text or "").strip()
        if not value:
            return
        def apply():
            self.live_voice_text = value
            self.live_voice_partial = bool(partial)
            self.live_voice_updated = time.time()
            if hasattr(self, "live_voice_label"):
                self.live_voice_label.config(text=value, fg=self.colors["text"])
            if hasattr(self, "live_voice_badge"):
                self.live_voice_badge.config(
                    text="●  СЛЫШУ ВАС..." if partial else "●  РАСПОЗНАНО",
                    fg=self.colors["purple2"] if partial else self.colors["green"]
                )
            if hasattr(self, "event_log") and not partial:
                try:
                    self.event_log.config(state="normal")
                    self.event_log.insert("end", f"🎙 {value[:90]}\n")
                    self.event_log.see("end")
                    self.event_log.config(state="disabled")
                except Exception:
                    pass
        self._ui(apply)

    def set_now_playing(self, title):
        value = str(title or "Музыка").strip()
        def apply():
            self.now_playing_title = value
            self.music_progress = 0.0
            if hasattr(self, "player_title"):
                self.player_title.config(text=value)
            if hasattr(self, "player_state"):
                self.player_state.config(text="PLAYING", fg=self.colors["green"])
            if hasattr(self, "player_meta"):
                self.player_meta.config(text=f"VLC STREAM  •  {_music_volume}%  •  LIVE AUDIO")
            if hasattr(self, "neo_music_stop"):
                self.neo_music_stop.config(state=tk.NORMAL)
            self.show_toast("NOW PLAYING", value[:60], "success", 2200)
        self._ui(apply)

    def clear_now_playing(self):
        def apply():
            self.now_playing_title = ""
            self.music_progress = 0.0
            self.music_elapsed = 0
            self.music_duration = 0
            if hasattr(self, "player_title"):
                self.player_title.config(text="Музыка не запущена")
            if hasattr(self, "player_state"):
                self.player_state.config(text="IDLE", fg=self.colors["muted"])
            if hasattr(self, "player_meta"):
                self.player_meta.config(text="Скажи: «Стелла, включи песню ...»")
            if hasattr(self, "player_time"):
                self.player_time.config(text="00:00")
            if hasattr(self, "player_duration_label"):
                self.player_duration_label.config(text="--:--")
        self._ui(apply)

    def update_music_progress(self, elapsed_ms, duration_ms):
        def fmt(ms):
            sec=max(0,int(ms)//1000); return f"{sec//60:02d}:{sec%60:02d}"
        def apply():
            self.music_elapsed=max(0,int(elapsed_ms or 0))
            self.music_duration=max(0,int(duration_ms or 0))
            self.music_progress=(self.music_elapsed/self.music_duration) if self.music_duration>0 else 0.0
            if hasattr(self,"player_time"): self.player_time.config(text=fmt(self.music_elapsed))
            if hasattr(self,"player_duration_label"): self.player_duration_label.config(text=fmt(self.music_duration) if self.music_duration>0 else "--:--")
        self._ui(apply)

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

            # Текстовый чат ведёт себя так же, как голос: сначала ИИ понимает намерение.
            if _mita_mode in [MODE_AI,MODE_ALL]:
                try:
                    if process_smart_ai_command(message,self):
                        self.is_processing=False; return
                except Exception as e:
                    print(f"[Mita Brain] Chat AI-FIRST error: {e}")

            if _mita_mode in [MODE_SYSTEM,MODE_ALL]:
                result=self.process_chat_system_command(message)
                if result:
                    self.root.after(0,lambda:self.add_chat_message("Мита",result,is_mita=True))
                    speak(result); self.is_processing=False; return

            if _mita_mode in [MODE_AI,MODE_ALL]:
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
        if is_weather_request(cleaned):
            return self.get_weather_for_chat(force=False)
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
                    target = prepare_text_for_typing(target)
                    type_unicode_text(target)
                    return T("typing_corrected") if _text_corrector_enabled else T("typed") + target
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
                ok, matched = smart_launch_application(target)
                return T("app_launching").format(matched or target) if ok else T("app_not_found").format(target)
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
        def apply():
            if hasattr(self,"live_voice_badge"):
                self.live_voice_badge.config(
                    text="●  СЛУШАЮ..." if self.is_listening else "●  МИКРОФОН ГОТОВ",
                    fg=self.colors["purple2"] if self.is_listening else self.colors["green"]
                )
            if self.is_listening and hasattr(self,"live_voice_label"):
                if time.time()-self.live_voice_updated > 1.0:
                    self.live_voice_label.config(text="Слушаю тебя...", fg=self.colors["muted"])
        self._ui(apply)

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
        for child_name in ("command_palette", "notes_window", "telemetry_window", "clipboard_window"):
            try:
                child = getattr(self, child_name, None)
                if child is not None: child.destroy()
            except Exception:
                pass
        try:self.root.destroy()
        except:pass

    def run(self):
        self.root.mainloop()


# ============================================================
# УМНЫЙ VAD / ШУМОПОДАВЛЕНИЕ МИКРОФОНА
# Игнорирует постоянный фон (вентилятор/гул), быстрее ловит голос.
# Не требует дополнительных библиотек.
# ============================================================

VOICE_MIN_RMS = 0.0040
VOICE_NOISE_MULTIPLIER = 2.8
VOICE_START_BLOCKS = 2
VOICE_END_BLOCKS = 8
VOICE_PREROLL_BLOCKS = 8
VOICE_MAX_SECONDS = 18.0
VOICE_LOW_HZ = 90.0
VOICE_HIGH_HZ = 3900.0
VOICE_MIN_BAND_RATIO = 0.50
VOICE_MIN_CENTROID_HZ = 170.0
VOICE_NOISE_LEARN_RATE = 0.035
VOICE_NOISE_FLOOR_INIT = 0.0030
VOICE_DEBUG = False


def _voice_features(samples, sample_rate):
    """Быстрые признаки речи: RMS + энергия речевого диапазона + спектральный центр."""
    x = np.asarray(samples, dtype=np.float32)
    if x.size < 16:
        return 0.0, 0.0, 0.0

    # Убираем DC и немного подавляем низкочастотный гул вентилятора.
    x = x - float(np.mean(x))
    hp = np.empty_like(x)
    hp[0] = x[0]
    hp[1:] = x[1:] - 0.965 * x[:-1]

    rms = float(np.sqrt(np.mean(hp * hp) + 1e-12))

    n = min(1024, hp.size)
    frame = hp[-n:]
    if n < 64:
        return rms, 0.0, 0.0

    window = np.hanning(n).astype(np.float32)
    spec = np.abs(np.fft.rfft(frame * window)) ** 2
    freqs = np.fft.rfftfreq(n, 1.0 / sample_rate)
    total = float(np.sum(spec)) + 1e-12

    speech_mask = (freqs >= VOICE_LOW_HZ) & (freqs <= VOICE_HIGH_HZ)
    speech_energy = float(np.sum(spec[speech_mask]))
    band_ratio = speech_energy / total
    centroid = float(np.sum(freqs * spec) / total)
    return rms, band_ratio, centroid


def voice_assistant_thread(interface, recognizer, audio_queue, sample_rate, ENERGY_THRESHOLD, SILENCE_LIMIT):
    from collections import deque

    word_buffer = []
    pre_roll = deque(maxlen=VOICE_PREROLL_BLOCKS)
    silence_counter = 0
    speech_counter = 0
    is_speaking = False
    noise_floor = VOICE_NOISE_FLOOR_INIT
    preview_counter = 0
    last_preview_text = ""
    max_blocks = max(1, int(VOICE_MAX_SECONDS * sample_rate / 480))

    def _finish_phrase():
        nonlocal word_buffer, silence_counter, speech_counter, is_speaking, preview_counter, last_preview_text
        interface.set_listening(False)

        if word_buffer:
            try:
                full_phrase_audio = np.concatenate(word_buffer).astype(np.float32, copy=False)

                # Срезаем очень тихие хвосты, но не трогаем саму речь.
                if full_phrase_audio.size >= int(sample_rate * 0.18):
                    stream = recognizer.create_stream()
                    stream.accept_waveform(sample_rate, full_phrase_audio)
                    recognizer.decode_stream(stream)
                    recognized_text = stream.result.text.strip()

                    if recognized_text:
                        if VOICE_DEBUG:
                            print(f"[VOICE] Распознано: {recognized_text}")
                        interface.show_live_speech(recognized_text, partial=False)
                        process_command(recognized_text, interface)
            except Exception as e:
                print(f"[Ошибка распознавания]: {e}")

        word_buffer = []
        silence_counter = 0
        speech_counter = 0
        is_speaking = False
        preview_counter = 0
        last_preview_text = ""
        pre_roll.clear()

    with sd.InputStream(
            samplerate=sample_rate,
            channels=1,
            dtype="float32",
            blocksize=480,  # 30 мс вместо 50 мс — быстрее реакция
            latency="low",
            callback=lambda indata, frames, time, status: audio_queue.put(indata.copy()),
    ):
        while interface.running:
            try:
                data = audio_queue.get(timeout=0.02)
            except queue.Empty:
                continue

            samples = data[:, 0].astype(np.float32, copy=False)
            pre_roll.append(samples.copy())

            rms, band_ratio, centroid = _voice_features(samples, sample_rate)

            # Порог сам подстраивается под комнату. Постоянный вентилятор становится
            # частью noise_floor и через пару секунд почти перестаёт запускать запись.
            dynamic_threshold = max(VOICE_MIN_RMS, noise_floor * VOICE_NOISE_MULTIPLIER)

            voice_shape = (
                band_ratio >= VOICE_MIN_BAND_RATIO and
                centroid >= VOICE_MIN_CENTROID_HZ
            )
            loud_enough = rms >= dynamic_threshold
            looks_like_voice = loud_enough and voice_shape

            if not is_speaking:
                # Учим фон только когда уверены, что пользователь не говорит.
                if not looks_like_voice:
                    capped = min(rms, max(noise_floor * 2.0, VOICE_MIN_RMS * 2.0))
                    noise_floor = (
                        (1.0 - VOICE_NOISE_LEARN_RATE) * noise_floor +
                        VOICE_NOISE_LEARN_RATE * capped
                    )

                if looks_like_voice:
                    speech_counter += 1
                else:
                    speech_counter = max(0, speech_counter - 1)

                # Нужны два речевых блока подряд: щелчок/удар/шум не сработает.
                if speech_counter >= VOICE_START_BLOCKS:
                    is_speaking = True
                    silence_counter = 0
                    word_buffer = list(pre_roll)
                    if not interface.is_listening:
                        interface.set_listening(True)
                    if VOICE_DEBUG:
                        print(
                            f"[VOICE] START rms={rms:.4f} noise={noise_floor:.4f} "
                            f"thr={dynamic_threshold:.4f} band={band_ratio:.2f} cent={centroid:.0f}"
                        )

            else:
                word_buffer.append(samples.copy())
                preview_counter += 1

                # Live preview approximately every 450 ms. It is visual only and never executes commands.
                if preview_counter >= 15 and len(word_buffer) >= 12:
                    preview_counter = 0
                    try:
                        preview_audio = np.concatenate(word_buffer).astype(np.float32, copy=False)
                        if preview_audio.size <= int(sample_rate * 8.0):
                            pstream = recognizer.create_stream()
                            pstream.accept_waveform(sample_rate, preview_audio)
                            recognizer.decode_stream(pstream)
                            ptxt = pstream.result.text.strip()
                            if ptxt and ptxt != last_preview_text:
                                last_preview_text = ptxt
                                interface.show_live_speech(ptxt, partial=True)
                    except Exception:
                        pass

                # Во время фразы порог чуть мягче, чтобы не обрезать тихие слоги.
                continue_threshold = max(VOICE_MIN_RMS * 0.75, noise_floor * 1.65)
                still_voice = (
                    rms >= continue_threshold and
                    band_ratio >= VOICE_MIN_BAND_RATIO * 0.72
                )

                if still_voice:
                    silence_counter = 0
                else:
                    silence_counter += 1

                # Быстро отдаём фразу в распознавание после ~240 мс тишины.
                if silence_counter >= VOICE_END_BLOCKS or len(word_buffer) >= max_blocks:
                    _finish_phrase()


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
    num_threads=max(4, min(10, (os.cpu_count() or 6))),
    debug=False,
)

sample_rate = 16000
audio_queue = queue.Queue()
ENERGY_THRESHOLD = 0.015  # legacy: умный VAD использует динамический порог
SILENCE_LIMIT = 4       # legacy: завершение фразы теперь ~240 мс

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
    "обс": "obs", "о б с": "obs", "обс студио": "obs",
    "бс": "bluestacks", "блюстакс": "bluestacks", "блустакс": "bluestacks", "blue stacks": "bluestacks",
    "дс": "discord", "дискордик": "discord", "тг": "telegram", "спотик": "spotify",
}

APP_EXE_MAP = {
    "roblox": "RobloxPlayerBeta.exe", "discord": "Discord.exe",
    "telegram": "Telegram.exe", "steam": "steam.exe",
    "chrome": "chrome.exe", "yandex": "browser.exe",
    "spotify": "Spotify.exe", "dota2": "dota2.exe",
    "геншин": "launcher.exe", "obs": "obs64.exe",
    "bluestacks": "HD-Player.exe",
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

    # СНАЧАЛА локальная папка приложений. Здесь вообще нет Groq/Steam/Fuzzy по всему ПК.
    if try_folder_app_voice_command(phrase, interface):
        interface.set_processing(False)
        return

    # Если Мита только что спросила «Хотите запустить ...?»,
    # короткое «да/нет» обрабатывается ДО нового AI-запроса.
    if _handle_pending_app_confirmation(phrase, interface):
        interface.set_processing(False)
        return

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

                original_phrase = clean_phrase
                clean_phrase = prepare_text_for_typing(clean_phrase)

                if clean_phrase != original_phrase:
                    if _text_corrector_enabled:
                        msg = T("typing_corrected_msg").format(original_phrase, clean_phrase)
                    else:
                        target_lang_name = "Українська" if UI_LANGUAGE == "ua" else "Русский"
                        msg = f"🌍 {target_lang_name}: {clean_phrase}"
                    interface.add_chat_message("Мита", msg, is_mita=True)
                else:
                    interface.add_chat_message(
                        "Мита",
                        T("typing_writing").format(clean_phrase),
                        is_mita=True
                    )

                time.sleep(0.3)
                type_unicode_text(clean_phrase)
                msg = T("typing_corrected") if _text_corrector_enabled else T("typing")
                interface.add_chat_message("Мита", msg, is_mita=True)
                interface.set_processing(False)
                return

    system_response = False

    # AI-FIRST: в режимах AI/ALL КАЖДАЯ фраза сначала проходит через ИИ-планировщик.
    # Он решает: запуск приложения, закрытие, сайт, окно, текст, погода или обычный чат.
    if _mita_mode in [MODE_AI, MODE_ALL]:
        try:
            system_response = process_smart_ai_command(phrase, interface)
        except Exception as e:
            print(f"[Mita Brain] Ошибка AI-FIRST команды: {e}")

        if system_response:
            interface.set_processing(False)
            return

    # Локальные правила — резерв и отдельный режим MODE_SYSTEM.
    if _mita_mode in [MODE_SYSTEM, MODE_ALL] and not system_response:
        system_response = process_system_command(phrase, interface)
        if system_response:
            interface.set_processing(False)
            return

    # Если ИИ решил, что это обычный разговор, отправляем запрос в разговорную модель.
    if _mita_mode in [MODE_AI, MODE_ALL]:
        print("[Стелла]: Обращаюсь к ИИ...")
        try:
            response = ask_groq(phrase)
            print(f"[Стелла]: {response}")
            interface.add_chat_message("Мита", response, is_mita=True)
            speak(response)
        except Exception as e:
            print(f"[Ошибка Groq]: {e}")
            interface.add_chat_message("Мита", T("error_text").format(e), is_mita=True)
    elif not system_response:
        response = T("command_not_found")
        interface.add_chat_message("Мита", response, is_mita=True)
        speak(response)

    interface.set_processing(False)


def process_system_command(phrase: str, interface):
    cleaned = phrase.lower().strip()
    words = cleaned.split()
    ui_lang = UI_LANGUAGE

    if is_weather_request(cleaned):
        result = get_local_weather(force=False, lang=("ua" if UI_LANGUAGE == "ua" else "ru"))
        if result.get("ok"):
            msg = result["report"]
            interface.add_chat_message("Мита", msg, is_mita=True)
            try: interface.root.after(0, lambda r=result: interface._apply_weather_result(r, silent=True, speak_result=False))
            except Exception: pass
            speak(msg, force=True)
        else:
            msg = result.get("error") or "Не удалось получить погоду."
            interface.add_chat_message("Мита", msg, is_mita=True)
            speak(msg, force=True)
        return True

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
                ok, matched = smart_launch_application(target)
                if ok:
                    msg = T("app_launching").format(matched or target)
                    interface.add_chat_message("Мита", msg, is_mita=True)
                    speak(msg, force=True)
                else:
                    msg = T("app_not_found").format(target)
                    interface.add_chat_message("Мита", msg, is_mita=True)
                    speak(msg, force=True)
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

