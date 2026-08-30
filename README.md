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
import math
import sys
from datetime import datetime, timedelta


# ============================================================
# ОПРЕДЕЛЕНИЕ БАЗОВОГО ПУТИ ДЛЯ .EXE
# ============================================================

def get_base_path():
    """Возвращает правильный путь для .exe и .py"""
    if getattr(sys, 'frozen', False):
        # Запущено как .exe
        return os.path.dirname(sys.executable)
    else:
        # Запущено как .py
        return os.path.dirname(os.path.abspath(__file__))


# Установите BASE_DIR в начало файла
BASE_DIR = get_base_path()

# Пути к файлам (используем BASE_DIR)
SOUNDS_DIR = os.path.join(BASE_DIR, "sounds")
JSON_CACHE_FILE = os.path.join(BASE_DIR, "saveAppPty.json")

# ============================================================
# ИНТЕРФЕЙС В СТИЛЕ MИСИДЕ (MISIDE)
# ============================================================

# ============================================================
# СИСТЕМА КЛЮЧЕЙ STELLA AI
# ============================================================
# Срок каждого ключа начинается с created_at, даже если ключ ещё
# не вводили. Активированный ключ сохраняется локально.
# ============================================================
KEY_DB_FILE = os.path.join(BASE_DIR, "stella_keys.json")
ACTIVATED_KEY_FILE = os.path.join(BASE_DIR, "stella_activated_key.json")

# Таблица ключей. created_at задаётся один раз и НЕ сбрасывается.
# Здесь можно менять/добавлять ключи и сроки.
KEY_TABLE = {
    "STELLA-1MIN": {"seconds": 60, "created_at": "2026-08-30 12:00:00"},
    "STELLA-1H": {"seconds": 3600, "created_at": "2026-08-30 12:00:00"},
    "STELLA-2H": {"seconds": 7200, "created_at": "2026-08-30 12:00:00"},
    "STELLA-3H": {"seconds": 10800, "created_at": "2026-08-30 12:00:00"},
    "STELLA-1DAY": {"seconds": 86400, "created_at": "2026-08-30 12:00:00"},
    "STELLA-1WEEK": {"seconds": 604800, "created_at": "2026-08-30 12:00:00"},
    "STELLA-1YEAR": {"seconds": 31536000, "created_at": "2026-08-30 12:00:00"},
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


class KeyLoginWindow:
    def __init__(self):
        self.result = False
        self.root = tk.Tk()
        self.root.title("MITA AI — Введите ключ")
        self.root.geometry("520x400")  # Увеличил высоту для фото
        self.root.resizable(False, False)
        self.root.configure(bg="#09070d")

        self.canvas = tk.Canvas(self.root, bg="#09070d", highlightthickness=0)
        self.canvas.pack(fill="both", expand=True)

        # Градиентный фон
        for y in range(400):
            t = y / 400
            r = int(9 + 8 * t)
            g = int(7 + 3 * t)
            b = int(13 + 14 * t)
            self.canvas.create_line(0, y, 520, y, fill=f"#{r:02x}{g:02x}{b:02x}")

        # Загрузка и отображение фото Миты
        self.mita_image = None
        try:
            # Путь к фото Миты
            mita_path = r"C:\Users\dubaz\OneDrive\Desktop\AI MITA\MitaPhoto\Mita.png"
            if os.path.exists(mita_path):
                from PIL import Image, ImageTk
                image = Image.open(mita_path)
                # Изменяем размер фото
                image = image.resize((120, 120), Image.Resampling.LANCZOS)
                self.mita_image = ImageTk.PhotoImage(image)
                # Размещаем фото в центре верхней части
                self.canvas.create_image(260, 60, image=self.mita_image)
            else:
                print(f"[Предупреждение] Фото Миты не найдено: {mita_path}")
                # Если фото нет, показываем текст
                self.canvas.create_text(260, 50, text="✦ MITA AI", font=("Arial", 24, "bold"), fill="#ff8bc4")
        except Exception as e:
            print(f"[Ошибка загрузки фото]: {e}")
            # Если ошибка, показываем текст
            self.canvas.create_text(260, 50, text="✦ MITA AI", font=("Arial", 24, "bold"), fill="#ff8bc4")

        # Текст "SECURE ACCESS"
        self.canvas.create_text(260, 145, text="SECURE ACCESS", font=("Arial", 9, "bold"), fill="#a58ca9")

        # Текст "Введите ключ доступа"
        self.canvas.create_text(260, 175, text="Введите ключ доступа", font=("Arial", 11, "bold"), fill="#fff4fb")

        # Поле ввода
        self.entry = tk.Entry(self.root, font=("Arial", 13, "bold"), justify="center",
                              bg="#150d1e", fg="#fff4fb",
                              insertbackground="#ff8bc4", relief="flat", bd=0)
        self.entry.place(x=70, y=200, width=380, height=42)

        # Статус
        self.status = self.canvas.create_text(260, 265, text="", font=("Arial", 9, "bold"), fill="#a58ca9")

        # Кнопка входа
        self.button = tk.Button(self.root, text="ВОЙТИ В MITA", command=self.try_login,
                                font=("Arial", 10, "bold"),
                                bg="#ff5caa", fg="white", activebackground="#ff8bc4",
                                relief="flat", bd=0, cursor="hand2")
        self.button.place(x=145, y=290, width=230, height=42)

        # Подпись внизу
        self.canvas.create_text(260, 365, text="Хочешь миту? пиши -> Discord - va#5572",
                                font=("Arial", 8), fill="#5d4965")

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
            self.entry.delete(0, tk.END);
            self.entry.focus_set()

    def cancel(self):
        self.result = False;
        self.root.destroy()

    def show(self):
        self.root.mainloop()
        return self.result


def require_stella_key():
    _build_key_db()
    if _get_saved_key():
        return True
    return KeyLoginWindow().show()


class SystemAudioMonitor:
    """Мониторинг системного аудио для визуализации сердцебиения"""

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
        """Запуск мониторинга аудио"""
        if self.running:
            return

        self.running = True
        self.thread = threading.Thread(target=self._monitor_loop, daemon=True)
        self.thread.start()

    def stop(self):
        """Остановка мониторинга"""
        self.running = False
        if self.thread:
            self.thread.join(timeout=1)

    def _monitor_loop(self):
        """Основной цикл захвата аудио"""
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

                        # Нормализуем амплитуду для сердцебиения
                        heart_beat = min(1.0, amplitude * 25)

                        # Плавное сглаживание
                        self.beat_smoothing = self.beat_smoothing * 0.85 + heart_beat * 0.15

                        # Обновляем сердцебиение
                        self.interface.update_heart_beat(self.beat_smoothing)

                        if len(self.audio_data) > 10:
                            self.audio_data.pop(0)

                    time.sleep(0.03)

        except Exception as e:
            print(f"[Audio Monitor Error]: {e}")
            self._fallback_monitor()

    def _audio_callback(self, indata, frames, time, status):
        """Колбэк для захвата аудио"""
        if status:
            pass
        self.audio_data.append(indata.copy())

    def _fallback_monitor(self):
        """Запасной вариант - используем микрофон"""
        print("[Audio] Использую микрофон для визуализации")
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


class MisideInterface:
    """Современный неоновый интерфейс Стеллы AI.
    Сохраняет совместимость с существующим backend:
    set_listening(), set_processing(), show_command(), run(), quit().
    """

    W, H = 1100, 720

    def __init__(self):
        self.root = tk.Tk()
        self.root.title("MITA AI")
        self.root.geometry(f"{self.W}x{self.H}")
        self.root.minsize(900, 620)
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

        # Параметры сердцебиения
        self.heart_beat_strength = 0.0
        self.heart_beat_target = 0.0
        self.heart_bpm = 72
        self.last_beat_time = 0
        self.is_beating = False
        self.beat_phase = 0.0

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
        }

        self.root.bind("<Button-1>", self.start_move)
        self.root.bind("<B1-Motion>", self.on_move)

        self.setup_ui()
        self.root.after(20, self.fade_in)

        # Запускаем монитор аудио для сердцебиения
        self.audio_monitor = SystemAudioMonitor(self)
        self.audio_monitor.start()

        self.animate()

    def update_heart_beat(self, strength):
        """Обновляет силу сердцебиения на основе аудио"""
        self.heart_beat_target = strength
        if strength > 0.1:
            self.heart_bpm = 72 + int(strength * 48)  # От 72 до 120 BPM

    def rounded_rect(self, x1, y1, x2, y2, r=18, fill=None, outline=None, width=1, tags=None):
        """Рисует сглаженную панель на Canvas."""
        pts = [
            x1 + r, y1, x2 - r, y1, x2, y1, x2, y1 + r,
            x2, y2 - r, x2, y2, x2 - r, y2, x1 + r, y2,
            x1, y2, x1, y2 - r, x1, y1 + r, x1, y1
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

        # Фон
        self.create_background()

        # Верхняя панель
        self.rounded_rect(18, 15, self.W - 18, 76, 20,
                          self.colors["panel"], self.colors["line"])
        self.canvas.create_text(
            42, 35, text="✦", font=("Arial", 20, "bold"),
            fill=self.colors["primary2"], anchor="w"
        )
        self.canvas.create_text(
            72, 33, text="MITA", font=("Arial", 19, "bold"),
            fill=self.colors["text"], anchor="w"
        )
        self.canvas.create_text(
            72, 56, text="AI VOICE ASSISTANT  •  ONLINE",
            font=("Arial", 8, "bold"), fill=self.colors["muted"], anchor="w"
        )




        # Окна
        self.canvas.create_rectangle(
            1010, 25, 1044, 63, fill=self.colors["panel2"],
            outline=self.colors["line"], width=1, tags=("minimize_bg",)
        )
        self.canvas.create_text(
            1027, 44, text="—", font=("Arial", 15, "bold"),
            fill=self.colors["muted"], tags=("minimize",)
        )



        # Левая навигация
        self.rounded_rect(18, 92, 235, 650, 20,
                          self.colors["panel"], self.colors["line"])
        self.canvas.create_text(
            42, 120, text="CONTROL CENTER",
            font=("Arial", 9, "bold"), fill=self.colors["dim"], anchor="w"
        )

        self.nav_items = []
        for i, (label, icon) in enumerate([
            ("Главная", "⌂"), ("Команды", "⌘"), ("Настройки", "⚙"), ("О системе", "ⓘ")
        ]):
            y = 160 + i * 54
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

        self.canvas.create_text(
            42, 390, text="VOICE ENGINE",
            font=("Arial", 9, "bold"), fill=self.colors["dim"], anchor="w"
        )

        self.engine_dot = self.canvas.create_oval(
            43, 416, 51, 424, fill=self.colors["primary"], outline=""
        )
        self.engine_text = self.canvas.create_text(
            61, 420, text="Mita AI ",
            font=("Arial", 9, "bold"), fill=self.colors["muted"], anchor="w"
        )

        # HOTWORD секция с ключом
        self.canvas.create_text(
            42, 468, text="HOTWORD",
            font=("Arial", 9, "bold"), fill=self.colors["dim"], anchor="w"
        )
        self.canvas.create_text(
            42, 490, text="«Стелла»  •  «Мита»",
            font=("Arial", 10, "bold"), fill=self.colors["text"], anchor="w"
        )

        # Информация о ключе
        self.key_info_text = self.canvas.create_text(
            42, 516, text="", font=("Arial", 9, "bold"), fill=self.colors["text"], anchor="w"
        )

        # Кнопка смены/ввода ключа
        change_key_bg = self.canvas.create_rectangle(
            42, 536, 155, 554, fill=self.colors["panel2"],
            outline=self.colors["line"], width=1, tags=("change_key_bg",)
        )
        self.change_key_btn = self.canvas.create_text(
            98, 545, text="[ Сменить ключ ]", font=("Arial", 8, "bold"),
            fill=self.colors["purple"], tags=("change_key",)
        )
        self.canvas.tag_bind("change_key", "<Button-1>", lambda e: self.reenter_key())
        self.canvas.tag_bind("change_key_bg", "<Button-1>", lambda e: self.reenter_key())
        self.canvas.tag_bind("change_key", "<Enter>",
                             lambda e: self.canvas.itemconfig(self.change_key_btn, fill=self.colors["primary2"]))
        self.canvas.tag_bind("change_key", "<Leave>",
                             lambda e: self.canvas.itemconfig(self.change_key_btn, fill=self.colors["purple"]))

        self.update_key_info()

        # Центральный блок
        self.rounded_rect(253, 92, 790, 650, 24,
                          self.colors["panel"], self.colors["line"])

        self.canvas.create_text(
            522, 120, text="Mita CORE",
            font=("Arial", 9, "bold"), fill=self.colors["dim"]
        )

        self.cx, self.cy = 522, 340

        # Аура
        self.aura = []
        for radius, outline, width in [
            (168, "#24142d", 1), (145, "#3b1b43", 1),
            (122, "#54204e", 2), (101, "#70265c", 1)
        ]:
            self.aura.append(
                self.canvas.create_oval(
                    self.cx - radius, self.cy - radius,
                    self.cx + radius, self.cy + radius,
                    outline=outline, width=width
                )
            )

        # Центральное ядро
        self.core_outer = self.canvas.create_oval(
            self.cx - 76, self.cy - 76, self.cx + 76, self.cy + 76,
            fill="#170b1d", outline="#ff5caa", width=2
        )
        self.core_inner = self.canvas.create_oval(
            self.cx - 58, self.cy - 58, self.cx + 58, self.cy + 58,
            fill="#221024", outline="#a77bff", width=1
        )

        self.heart_size = 45
        self.heart = self.create_heart(
            self.cx, self.cy, self.heart_size, self.colors["primary"]
        )

        # Добавляем свечение для сердца
        self.heart_glow = self.canvas.create_oval(
            self.cx - 70, self.cy - 70, self.cx + 70, self.cy + 70,
            fill="", outline="#ff5caa", width=3, stipple="gray75"
        )

        # Орбиты
        self.orbit_elements = []
        for i in range(10):
            a = i * (math.pi * 2 / 10)
            obj = self.canvas.create_text(
                self.cx, self.cy, text="✦",
                font=("Arial", 9 + i % 3, "bold"),
                fill=self.colors["purple"]
            )
            self.orbit_elements.append({
                "id": obj, "angle": a,
                "radius": 130 + (i % 2) * 24,
                "speed": 0.008 + i * 0.0012
            })

        self.status_text = self.canvas.create_text(
            self.cx, 525, text="✧ ГОТОВА К РАБОТЕ ✧",
            font=("Arial", 16, "bold"), fill=self.colors["primary2"]
        )
        self.status_sub = self.canvas.create_text(
            self.cx, 552, text="Ожидаю голосовую команду",
            font=("Arial", 10), fill=self.colors["muted"]
        )

        # Визуализатор
        self.create_bars()

        # Последняя команда
        self.rounded_rect(290, 590, 754, 628, 12,
                          self.colors["panel2"], self.colors["line"])
        self.command_text = self.canvas.create_text(
            307, 609, text="Последняя команда: —",
            font=("Arial", 9), fill=self.colors["muted"], anchor="w"
        )

        # Правая панель
        self.rounded_rect(808, 92, 1082, 650, 20,
                          self.colors["panel"], self.colors["line"])

        self.canvas.create_text(
            833, 120, text="Команды Миты",
            font=("Arial", 9, "bold"), fill=self.colors["dim"], anchor="w"
        )

        commands = [
            ("01", "Запусти", "[программа]"),
            ("02", "Открой", "[сайт]"),
            ("03", "Закрой", "[программа]"),
            ("04", "Напиши", "[текст]"),
            ("05", "Сверни", "[окно]"),
            ("06", "Скриншот", ""),
            ("07", "Скопируй", ""),
            ("08", "Вставь", ""),
        ]

        y = 158
        for num, title, tail in commands:
            self.canvas.create_text(
                833, y, text=num, font=("Arial", 8, "bold"),
                fill=self.colors["primary"], anchor="w"
            )
            self.canvas.create_text(
                862, y, text=title, font=("Arial", 9, "bold"),
                fill=self.colors["text"], anchor="w"
            )
            if tail:
                self.canvas.create_text(
                    862, y + 17, text=tail, font=("Arial", 8),
                    fill=self.colors["muted"], anchor="w"
                )
            y += 48

        # Нижняя информация
        self.canvas.create_line(
            32, 625, 221, 625, fill=self.colors["line"]
        )
        self.canvas.create_text(
            42, 645, text="RAM", font=("Arial", 8, "bold"),
            fill=self.colors["dim"], anchor="w"
        )
        self.ram_text = self.canvas.create_text(
            205, 645, text=f"{self.get_ram_usage()} MB",
            font=("Arial", 8, "bold"), fill=self.colors["muted"], anchor="e"
        )

        # Добавляем индикатор аудио
        self.canvas.create_text(
            820, 645, text="AUDIO", font=("Arial", 8, "bold"),
            fill=self.colors["dim"], anchor="w"
        )
        self.audio_indicator = self.canvas.create_text(
            930, 645, text="⚪", font=("Arial", 12, "bold"),
            fill=self.colors["dim"], anchor="e"
        )

        self.canvas.create_text(
            820, 700, text="Mita AI  •  2026",
            font=("Arial", 8), fill=self.colors["dim"], anchor="w"
        )
        self.create_particles()
        self.create_stars()

    def update_key_info(self):
        """Обновляет информацию о ключе и оставшемся времени"""
        saved_key = _get_saved_key()
        if saved_key:
            db = _load_json_file(KEY_DB_FILE)
            info = db.get(saved_key)
            if info:
                remaining = float(info["expires_at"]) - time.time()
                if remaining > 0:
                    key_display = saved_key[:8] + "..." if len(saved_key) > 8 else saved_key
                    time_left = _remaining_text(remaining)
                    self.canvas.itemconfig(
                        self.key_info_text,
                        text=f"🔑 {key_display}  •  {time_left}",
                        fill=self.colors["text"]
                    )
                    self.root.after(10000, self.update_key_info)
                    return

        self.canvas.itemconfig(
            self.key_info_text,
            text="🔑 Нет ключа",
            fill=self.colors["danger"]
        )
        self.root.after(10000, self.update_key_info)

    def reenter_key(self):
        """Позволяет ввести новый ключ"""
        self.root.attributes("-topmost", False)
        if KeyLoginWindow().show():
            self.update_key_info()
        self.root.attributes("-topmost", False)

    def create_background(self):
        # Мягкий вертикальный градиент
        for y in range(self.H):
            t = y / max(1, self.H - 1)
            r = int(9 + 8 * t)
            g = int(7 + 3 * t)
            b = int(13 + 14 * t)
            self.canvas.create_line(
                0, y, self.W, y,
                fill=f"#{r:02x}{g:02x}{b:02x}"
            )

        # Декоративные диагонали
        for x in range(-self.H, self.W, 90):
            self.canvas.create_line(
                x, 0, x + self.H, self.H,
                fill="#120b18", width=1
            )

    def create_heart(self, x, y, size, color):
        points = []
        for t in range(0, 360, 5):
            rad = math.radians(t)
            hx = 16 * math.sin(rad) ** 3
            hy = (
                    13 * math.cos(rad)
                    - 5 * math.cos(2 * rad)
                    - 2 * math.cos(3 * rad)
                    - math.cos(4 * rad)
            )
            points.extend([
                x + hx * size / 16,
                y - hy * size / 16
            ])

        heart_id = self.canvas.create_polygon(
            points, fill=color, outline=color, width=2
        )
        glow_id = self.canvas.create_polygon(
            points, fill="", outline="#ff8bc4", width=3
        )
        return {"id": heart_id, "glow": glow_id}

    def create_particles(self):
        symbols = ["✦", "·", "✧", "◇"]
        for _ in range(36):
            self.particles.append({
                "angle": random.uniform(0, math.pi * 2),
                "radius": random.uniform(90, 215),
                "speed": random.uniform(.15, .8),
                "size": random.randint(7, 12),
                "symbol": random.choice(symbols),
                "id": None
            })

    def create_stars(self):
        for _ in range(42):
            x = random.randint(15, self.W - 15)
            y = random.randint(85, self.H - 15)
            size = random.choice([2, 2, 3])
            obj = self.canvas.create_oval(
                x, y, x + size, y + size,
                fill="#4a2d55", outline=""
            )
            self.stars.append({
                "id": obj, "x": x, "y": y,
                "speed": random.uniform(.4, 1.5)
            })

    def create_bars(self):
        self.bars = []
        start_x = 402
        for i in range(25):
            x = start_x + i * 10
            obj = self.canvas.create_rectangle(
                x, self.cy + 92, x + 5, self.cy + 94,
                fill=self.colors["purple"], outline=""
            )
            self.bars.append({
                "id": obj, "x": x,
                "base": self.cy + 93,
                "height": 2.0,
                "phase": random.uniform(0, math.pi * 2)
            })

    def get_ram_usage(self):
        try:
            return int(psutil.Process().memory_info().rss / 1024 / 1024)
        except Exception:
            return 0

    def update_heart(self, size, color, glow_intensity=1.0):
        """Обновляет сердце с пульсацией"""
        points = []
        # Добавляем небольшое искажение для эффекта биения
        beat_offset = 0
        if self.heart_beat_strength > 0.05:
            beat_offset = math.sin(self.beat_phase) * self.heart_beat_strength * 8

        for t in range(0, 360, 5):
            rad = math.radians(t)
            hx = 16 * math.sin(rad) ** 3
            hy = (
                    13 * math.cos(rad)
                    - 5 * math.cos(2 * rad)
                    - 2 * math.cos(3 * rad)
                    - math.cos(4 * rad)
            )
            # Добавляем эффект пульсации
            scale = 1 + beat_offset / 100
            points.extend([
                self.cx + hx * size / 16 * scale,
                self.cy - hy * size / 16 * scale
            ])

        self.canvas.coords(self.heart["id"], *points)
        self.canvas.coords(self.heart["glow"], *points)
        self.canvas.itemconfig(self.heart["id"], fill=color, outline=color)

        # Обновляем свечение
        glow_alpha = int(50 + 150 * glow_intensity)
        self.canvas.itemconfig(self.heart_glow, outline=color)
        self.canvas.coords(
            self.heart_glow,
            self.cx - 60 - beat_offset, self.cy - 60 - beat_offset,
            self.cx + 60 + beat_offset, self.cy + 60 + beat_offset
        )

    def animate_heart_beat(self):
        """Анимация сердцебиения на основе аудио"""
        # Плавное изменение силы биения
        self.heart_beat_strength += (self.heart_beat_target - self.heart_beat_strength) * 0.15

        # Обновляем фазу биения
        bpm_factor = self.heart_bpm / 60.0
        self.beat_phase += 0.025 * bpm_factor * 2

        # Определяем интенсивность свечения
        if self.heart_beat_strength > 0.1:
            glow = 0.5 + self.heart_beat_strength * 0.5
            # Добавляем пульсацию цвета
            color_pulse = min(1.0, self.heart_beat_strength * 1.5)
            r = int(255 - (255 - 167) * (1 - color_pulse))
            g = int(92 - 92 * (1 - color_pulse))
            b = int(170 - 170 * (1 - color_pulse))
            color = f"#{r:02x}{g:02x}{b:02x}"
        else:
            glow = 0.3
            color = self.colors["primary"]

        # Обновляем BPM на индикаторе



        # Обновляем индикатор аудио
        if self.heart_beat_strength > 0.15:
            self.canvas.itemconfig(self.audio_indicator, text="🔴", fill="#ff5caa")
        elif self.heart_beat_strength > 0.05:
            self.canvas.itemconfig(self.audio_indicator, text="🟡", fill="#a77bff")
        else:
            self.canvas.itemconfig(self.audio_indicator, text="⚪", fill=self.colors["dim"])

        return glow, color

    def animate(self):
        if not self.running:
            return

        self.angle += 0.035
        self.phase += 0.12

        # Анимируем сердцебиение
        glow_intensity, heart_color = self.animate_heart_beat()

        if self.is_listening:
            status = "♪ СЛУШАЮ ВАС... ♪"
            sub = "Говорите — Стелла распознаёт голос"
            color = self.colors["primary"]
        elif self.is_processing:
            status = "✦ ОБРАБОТКА... ✦"
            sub = "Выполняю вашу команду"
            color = self.colors["purple"]
        else:
            status = "✧ ГОТОВА К РАБОТЕ ✧"
            sub = "Ожидаю голосовую команду"
            color = self.colors["primary2"]

        # Пульс сердца зависит от аудио
        pulse_size = 45 + math.sin(self.angle * 2.0) * 4
        if self.heart_beat_strength > 0.1:
            # Добавляем дополнительную пульсацию от аудио
            audio_pulse = 1 + math.sin(self.beat_phase) * self.heart_beat_strength * 0.3
            pulse_size = 45 * audio_pulse

        self.update_heart(pulse_size, heart_color, glow_intensity)

        self.canvas.itemconfig(self.status_text, text=status, fill=color)
        self.canvas.itemconfig(self.status_sub, text=sub)
        self.canvas.itemconfig(self.core_outer, outline=color)

        # Пульс ауры
        for i, obj in enumerate(self.aura):
            delta = math.sin(self.angle * 1.4 + i) * (2 + i)
            # Добавляем влияние аудио на ауру
            audio_delta = self.heart_beat_strength * 10 * math.sin(self.beat_phase + i)
            base = [168, 145, 122, 101][i]
            total_delta = delta + audio_delta
            self.canvas.coords(
                obj,
                self.cx - base - total_delta, self.cy - base - total_delta,
                self.cx + base + total_delta, self.cy + base + total_delta
            )
            # Меняем цвет ауры в зависимости от аудио
            if self.heart_beat_strength > 0.1:
                alpha = int(100 + 155 * self.heart_beat_strength)
                self.canvas.itemconfig(obj, outline=f"#{alpha:02x}2d{alpha:02x}")

        # Орбиты
        for item in self.orbit_elements:
            item["angle"] += item["speed"]
            # Добавляем влияние аудио на орбиты
            orbit_offset = self.heart_beat_strength * 15 * math.sin(self.beat_phase + item["angle"])
            radius = item["radius"] + orbit_offset
            x = self.cx + math.cos(item["angle"]) * radius
            y = self.cy + math.sin(item["angle"]) * radius
            self.canvas.coords(item["id"], x, y)
            self.canvas.itemconfig(item["id"], fill=color)

        # Частицы
        for p in self.particles:
            p["angle"] += p["speed"] * 0.01
            # Добавляем влияние аудио на частицы
            particle_pulse = 1 + self.heart_beat_strength * 0.2 * math.sin(self.beat_phase + p["angle"])
            radius = p["radius"] * particle_pulse
            x = self.cx + math.cos(p["angle"]) * radius
            y = self.cy + math.sin(p["angle"]) * radius
            if p["id"] is None:
                p["id"] = self.canvas.create_text(
                    x, y, text=p["symbol"],
                    font=("Arial", p["size"]),
                    fill=color
                )
            else:
                self.canvas.coords(p["id"], x, y)
                self.canvas.itemconfig(p["id"], fill=color)

        # Мерцание
        now = time.time()
        for i, star in enumerate(self.stars):
            v = 0.35 + 0.35 * (math.sin(now * star["speed"] + i) + 1) / 2
            c = int(55 + 75 * v)
            self.canvas.itemconfig(
                star["id"], fill=f"#{c:02x}{int(c * 0.55):02x}{int(c * 0.75):02x}"
            )

        # Аудио-визуализатор - теперь реагирует на аудио
        for i, bar in enumerate(self.bars):
            if self.is_listening or self.is_processing:
                target = 5 + abs(math.sin(self.phase + i * .55)) * random.randint(8, 24)
                bar["height"] += (target - bar["height"]) * .28
            else:
                # Добавляем реакцию на аудио в фоновом режиме
                audio_boost = self.heart_beat_strength * 20
                target = 2.5 + abs(math.sin(self.phase * .4 + i)) * 2 + audio_boost * abs(
                    math.sin(self.beat_phase + i * 0.5))
                bar["height"] += (target - bar["height"]) * .12

            h = max(2, bar["height"])
            self.canvas.coords(
                bar["id"],
                bar["x"], bar["base"] - h,
                          bar["x"] + 5, bar["base"]
            )
            # Меняем цвет баров в зависимости от аудио
            if self.heart_beat_strength > 0.1:
                bar_color = self.colors["primary"] if h > 10 else color
            else:
                bar_color = color
            self.canvas.itemconfig(bar["id"], fill=bar_color)

        self.canvas.itemconfig(
            self.ram_text, text=f"{self.get_ram_usage()} MB"
        )

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

    def start_move(self, event):
        self.drag_x = event.x
        self.drag_y = event.y

    def on_move(self, event):
        dx = event.x - self.drag_x
        dy = event.y - self.drag_y
        x = self.root.winfo_x() + dx
        y = self.root.winfo_y() + dy
        self.root.geometry(f"+{x}+{y}")

    def minimize(self):
        self.root.withdraw()

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
        self.canvas.itemconfig(
            self.command_text,
            text=f"Последняя команда: {short or '—'}"
        )

    def quit(self):
        self.running = False
        # Останавливаем аудио монитор
        if hasattr(self, 'audio_monitor'):
            self.audio_monitor.stop()
        try:
            self.root.quit()
            self.root.destroy()
        except Exception:
            pass
        os._exit(0)

    def run(self):
        self.root.mainloop()


# ============================================================
# ФУНКЦИЯ ГОЛОСОВОГО АССИСТЕНТА
# ============================================================

def voice_assistant_thread(interface, recognizer, audio_queue, sample_rate, ENERGY_THRESHOLD, SILENCE_LIMIT):
    """Запускает голосовой ассистент в отдельном потоке"""
    word_buffer = []
    silence_counter = 0
    is_speaking = False

    print("=" * 60)
    print("✦ MISIDE - Стелла AI ✦")
    print("=" * 60)
    print("🎤 Скажите 'Стелла' + команда")
    print("=" * 60)

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
# ГЛОБАЛЬНЫЕ ПЕРЕМЕННЫЕ И ФУНКЦИИ
# ============================================================

try:
    import soundfile as sf

    HAS_SOUNDFILE = True
except ImportError:
    HAS_SOUNDFILE = False

pyautogui.FAILSAFE = False
pyautogui.PAUSE = 0.05

BAD_WORDS = [
    "нахуй", "блядь", "блять", "сука", "хуй", "пизда", "пиздец",
    "ебан", "ебать", "заеб", "пидор", "гандон", "мудак", "уебок",
    "тупой", "дебил", "идиот", "кретин", "олень", "козел", "козёл",
    "лох", "лошара", "чмо", "шлюха", "курва", "бля", "нах", "хер"
]


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

TRIGGER_WORDS = ["мита", "стелла", "стелам", "стеллу"]

APP_VERBS = ["запусти", "запустил", "включи", "запустить", "включить"]
WEB_VERBS = ["открой", "открыть", "перейди", "перейти", "покажи"]
CLOSE_VERBS = ["закрой", "закрыть", "выключи", "выключить", "убей"]
LANG_VERBS = ["измени", "смени", "поменяй", "переключи"]
MOVE_VERBS = ["переведи", "перемести", "перекинь", "отправь"]
WRITE_VERBS = ["напиши", "напечатай", "пиши", "печатай", "набор"]
MINIMIZE_VERBS = ["сверни", "свернуть", "скрой", "спрячь"]

GREETINGS_MAP = {
    "ты тут": ("Да, я здесь! Чем помочь?", ["you_here"]),
    "ты здесь": ("Здесь. Слушаю вас.", ["you_here"]),
    "привет": ("Привет! Что нужно запустить или открыть?", ["hello"]),
    "как дела": ("Все отлично, готова к работе!", ["how_are_you"]),
    "не спишь": ("Я всегда на связи!", ["how_are_you"]),
    "спишь": ("Я всегда на связи!", ["how_are_you"]),
}

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
    "яндекс": "yandex",
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
    "геншин": "launcher.exe",
    "genshin impact": "launcher.exe",
    "obs": "obs64.exe",
    "cs2": "cs2.exe",
    "xeno": "Xeno-v1.3.60.exe",
    "calc": "calc.exe",
    "notepad": "notepad.exe",
}

PROCESS_KILL_MAP = {
    "obs": "obs64.exe",
    "discord": "Discord.exe",
    "telegram": "Telegram.exe",
    "steam": "steam.exe",
    "chrome": "chrome.exe",
    "yandex": "browser.exe",
    "spotify": "Spotify.exe",
    "xeno": "Xeno-v1.3.60.exe",
    "геншин": "HoYoPlay.exe",
    "dota2": "dota2.exe",
    "roblox": "RobloxPlayerBeta.exe",
    "cs2": "cs2.exe",
    "calc": "calc.exe",
    "notepad": "notepad.exe",
}

WEB_URLS_MAP = {
    "покет": ["https://m.pocketoption.com/ru/cabinet/demo-quick-high-low/", "https://blacksignalsus.com/"],
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
    "почту": "https://mail.google.com",
    "почта": "https://mail.google.com",
}

UKRAINIAN_SONGS_URLS = [
    "https://www.youtube.com/results?search_query=KAZKA+-+%D0%9F%D0%BB%D0%B0%D0%BA%D0%B0%D0%BB%D0%B0",
    "https://www.youtube.com/results?search_query=KALUSH+%26+SKOFKA+-+%D0%94%D0%BE%D0%B4%D0%BE%D0%BC%D1%83",
    "https://www.youtube.com/results?search_query=Kalush+Orchestra+-+Stefania",
]

RANDOM_SONGS_URLS = [
    "https://www.youtube.com/watch?v=pgN-vvVVxMA",
    "https://www.youtube.com/watch?v=P1t9T1TAOBI",
    "https://www.youtube.com/watch?v=GX8Hg6kWQYI",
]


# ============================================================
# ФУНКЦИИ КОМАНД
# ============================================================

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


def play_ukrainian_song():
    if not play_sound("ukr_song") and not play_sound("ukr"):
        play_sound("ok")
    url = random.choice(UKRAINIAN_SONGS_URLS)
    webbrowser.open(url)


def play_random_song():
    if not play_sound("random_song") and not play_sound("random"):
        play_sound("ok")
    url = random.choice(RANDOM_SONGS_URLS)
    webbrowser.open(url)


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


def move_window_to_monitor(target_raw: str):
    target_clean = target_raw.lower().strip()

    target_monitor = None
    if "первый" in target_clean or "1" in target_clean or "один" in target_clean:
        target_monitor = 1
    elif "второй" in target_clean or "2" in target_clean or "два" in target_clean:
        target_monitor = 2
    else:
        play_sound("error")
        return False

    words_to_remove = ["на", "в", "до", "экран", "монитор", "первый", "второй", "1", "2", "один", "два",
                       "переведи", "перемести", "перекинь", "отправь"]
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
        play_sound("error")
        return False

    PRIMARY_MONITOR_WIDTH = 1920
    new_x = 100 if target_monitor == 1 else PRIMARY_MONITOR_WIDTH + 100

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
        play_sound("error")
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
    "выдели все": lambda: keyboard.press_and_release('ctrl+a'),
    "выдели": lambda: keyboard.press_and_release('ctrl+a'),
    "скопировать": lambda: keyboard.press_and_release('ctrl+c'),
    "скопируй": lambda: keyboard.press_and_release('ctrl+c'),
    "копируй": lambda: keyboard.press_and_release('ctrl+c'),
    "вставить": lambda: keyboard.press_and_release('ctrl+v'),
    "вставь": lambda: keyboard.press_and_release('ctrl+v'),
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

    if not play_sound(target_clean):
        if not play_sound(app_key):
            play_sound("ok")

    cache = load_app_cache()

    if app_key in cache:
        cached_path = cache[app_key]
        if os.path.exists(cached_path):
            os.startfile(cached_path)
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

    if not play_sound(f"close_{app_key}"):
        if not play_sound("close"):
            play_sound("ok")

    cmd = f'taskkill /F /IM "{exe_name}"'
    result = subprocess.run(cmd, shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True)

    if result.returncode == 0:
        return True
    else:
        play_sound("error")
        return False


def open_website(target_raw: str):
    target_clean = target_raw.lower().strip()

    if not play_sound(target_clean):
        play_sound("ok")

    if target_clean in WEB_URLS_MAP:
        url_or_list = WEB_URLS_MAP[target_clean]
        if isinstance(url_or_list, list):
            for url in url_or_list:
                webbrowser.open(url)
        else:
            webbrowser.open(url_or_list)
        return True

    if "." in target_clean and not target_clean.endswith(".exe"):
        url = f"https://{target_clean}" if not target_clean.startswith("http") else target_clean
        webbrowser.open(url)
        return True

    search_url = f"https://www.google.com/search?q={target_clean}"
    webbrowser.open(search_url)
    return True


def process_command(text: str, interface):
    """Обработка голосовой команды"""
    cleaned = text.lower().strip()
    words = cleaned.split()

    if not any(tw in words for tw in TRIGGER_WORDS):
        return

    print(f"\n[Распознано]: {text}")

    interface.set_processing(True)
    interface.show_command(text[:40])

    filtered_words = [w for w in words if w not in TRIGGER_WORDS]
    filtered_words = [w for w in filtered_words if w not in BAD_WORDS]

    if not filtered_words:
        print("[Стелла]: Слушаю вас!")
        play_sound("da")
        interface.set_processing(False)
        return

    phrase_after_trigger = " ".join(filtered_words).strip()

    if "все" in filtered_words and any(w in filtered_words for w in ["окна", "окон", "окно"]):
        minimize_all_windows()
        interface.set_processing(False)
        return

    for verb in MINIMIZE_VERBS:
        if verb in filtered_words:
            if "все" in filtered_words and any(w in filtered_words for w in ["окна", "окон", "окно"]):
                continue
            target = " ".join([w for w in filtered_words if w != verb]).strip()
            if target:
                minimize_window(target)
            else:
                win = gw.getActiveWindow()
                if win:
                    try:
                        win.minimize()
                        play_sound("minimize")
                    except:
                        pass
            interface.set_processing(False)
            return

    if any(w in phrase_after_trigger for w in ["украинскую", "укр"]) and "песн" in phrase_after_trigger:
        play_ukrainian_song()
        interface.set_processing(False)
        return

    if any(w in phrase_after_trigger for w in ["рандомную", "рандом", "случайную"]) and "песн" in phrase_after_trigger:
        play_random_song()
        interface.set_processing(False)
        return

    for verb in MOVE_VERBS:
        if verb in filtered_words:
            target = " ".join([w for w in filtered_words if w != verb]).strip()
            if target:
                move_window_to_monitor(target)
            interface.set_processing(False)
            return

    for verb in WRITE_VERBS:
        if verb in filtered_words:
            verb_index = filtered_words.index(verb)
            text_to_type = " ".join(filtered_words[verb_index + 1:]).strip()
            if text_to_type:
                if not play_sound("write"):
                    play_sound("ok")
                time.sleep(0.1)
                keyboard.write(text_to_type, delay=0.02)
                if verb in ["напечатай", "набор"]:
                    keyboard.press_and_release('enter')
            interface.set_processing(False)
            return

    for verb in APP_VERBS:
        if verb in filtered_words:
            target = " ".join([w for w in filtered_words if w != verb]).strip()
            if target:
                launch_application(target)
            interface.set_processing(False)
            return

    for verb in WEB_VERBS:
        if verb in filtered_words:
            target = " ".join([w for w in filtered_words if w != verb]).strip()
            if target:
                open_website(target)
            interface.set_processing(False)
            return

    for verb in CLOSE_VERBS:
        if verb in filtered_words:
            target = " ".join([w for w in filtered_words if w != verb]).strip()
            if target:
                kill_application(target)
            interface.set_processing(False)
            return

    for verb in LANG_VERBS:
        if verb in filtered_words:
            target = " ".join([w for w in filtered_words if w != verb]).strip()
            if target in ["язык", "раскладку"]:
                change_language()
            interface.set_processing(False)
            return

    if phrase_after_trigger in CONTROL_COMMANDS:
        CONTROL_COMMANDS[phrase_after_trigger]()
        interface.set_processing(False)
        return

    if phrase_after_trigger in GREETINGS_MAP:
        text_response, sound_variants = GREETINGS_MAP[phrase_after_trigger]
        print(f"[Стелла]: {text_response}")
        for s_name in sound_variants:
            if play_sound(s_name):
                break
        interface.set_processing(False)
        return

    print(f"[Стелла]: Команда '{phrase_after_trigger}' не распознана.")
    play_sound("error")
    interface.set_processing(False)


# ============================================================
# ГЛАВНАЯ ФУНКЦИЯ
# ============================================================

def main():
    # Создаём папку для звуков
    if not os.path.exists(SOUNDS_DIR):
        os.makedirs(SOUNDS_DIR)
        print(f"📁 Создана папка для звуков: {SOUNDS_DIR}")

    # Авторизация ДО запуска Stella AI.
    if not require_stella_key():
        sys.exit(0)

    load_app_cache()

    # Создаём интерфейс
    interface = MisideInterface()

    # Запускаем голосовой ассистент в отдельном потоке
    voice_thread = threading.Thread(
        target=voice_assistant_thread,
        args=(interface, recognizer, audio_queue, sample_rate, ENERGY_THRESHOLD, SILENCE_LIMIT),
        daemon=True
    )
    voice_thread.start()

    # Запускаем интерфейс
    try:
        interface.run()
    except KeyboardInterrupt:
        interface.quit()


if __name__ == "__main__":
    main()
