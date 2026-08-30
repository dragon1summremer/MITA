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


# ============================================================
# ИНТЕРФЕЙС В СТИЛЕ MИСИДЕ (MISIDE)
# ============================================================

class MisideInterface:
    """Интерфейс в стиле игры Miside - аниме/игровой стиль"""

    def minimize(self):
        """Сворачивает окно в трей"""
        self.root.withdraw()  # Скрывает окно полностью

    def __init__(self):
        self.root = tk.Tk()
        self.root.title("MISIDE - Стелла AI")
        self.root.geometry("950x700")
        self.root.configure(bg='#1a0a1a')
        self.root.overrideredirect(True)
        self.root.attributes('-topmost', True)
        self.root.attributes('-alpha', 0.98)


        # Для перетаскивания
        self.root.bind('<Button-1>', self.start_move)

        self.root.bind('<B1-Motion>', self.on_move)

        # Переменные
        self.angle = 0
        self.is_listening = False
        self.is_processing = False
        self.particles = []
        self.hearts = []
        self.stars = []
        self.running = True

        # Цвета в стиле Miside (розовый/фиолетовый/неон)
        self.colors = {
            'primary': '#ff6b9d',  # Розовый
            'secondary': '#c084fc',  # Фиолетовый
            'accent': '#f472b6',  # Светло-розовый
            'neon': '#ff2d75',  # Неон-розовый
            'dark': '#1a0a1a',  # Тёмный фон
            'text': '#fce7f3',  # Светло-розовый текст
            'glow': '#ff6b9d44'
        }

        # Создаём интерфейс
        self.setup_ui()

        # Запускаем анимацию
        self.animate()

    def setup_ui(self):
        """Создание интерфейса в стиле Miside"""
        # Основной холст
        self.canvas = tk.Canvas(
            self.root,
            width=950,
            height=700,
            bg='#1a0a1a',
            highlightthickness=0
        )
        self.canvas.pack(fill=tk.BOTH, expand=True)

        # Градиентный фон
        self.create_gradient_bg()

        # ===== ВЕРХНЯЯ ПАНЕЛЬ =====
        # Заголовок в стиле аниме
        self.canvas.create_text(
            50, 35,
            text="✦ MISIDE",
            font=('Arial', 24, 'bold'),
            fill='#ff6b9d',
            anchor='w'
        )

        # Версия
        self.canvas.create_text(
            50, 65,
            text="v0.1.0 BETA ✧",
            font=('Arial', 10),
            fill='#a855a8',
            anchor='w'
        )

        # Кнопки управления в стиле игры
        # Свернуть
        self.canvas.create_rectangle(
            890, 15, 915, 35,
            fill='#2a0a2a',
            outline='#ff6b9d',
            width=1,
            tags=('minimize_bg',)
        )
        self.min_btn = self.canvas.create_text(
            902, 25,
            text="─",
            font=('Arial', 14, 'bold'),
            fill='#ff6b9d',
            tags=('minimize',)
        )
        self.canvas.tag_bind('minimize', '<Button-1>', lambda e: self.minimize())
        self.canvas.tag_bind('minimize_bg', '<Button-1>', lambda e: self.minimize())

        # Закрыть
        self.canvas.create_rectangle(
            915, 15, 935, 35,
            fill='#2a0a2a',
            outline='#ff2d75',
            width=1,
            tags=('close_bg',)
        )
        self.close_btn = self.canvas.create_text(
            925, 25,
            text="✕",
            font=('Arial', 12, 'bold'),
            fill='#ff2d75',
            tags=('close',)
        )
        self.canvas.tag_bind('close', '<Button-1>', lambda e: self.quit())
        self.canvas.tag_bind('close_bg', '<Button-1>', lambda e: self.quit())

        # ===== ЛЕВАЯ ПАНЕЛЬ (МЕНЮ) =====
        menu_items = [
            ("✦ КОМАНДЫ", 100),
            ("✦ НАСТРОЙКИ", 140),
            ("✦ ИНФО", 180)
        ]

        for text, y in menu_items:
            self.canvas.create_text(
                40, y,
                text=text,
                font=('Arial', 11, 'bold'),
                fill='#c084fc',
                anchor='w'
            )

        # Декоративная линия
        for i in range(3):
            y = 85 + i * 40
            self.canvas.create_line(
                20, y, 180, y,
                fill='#ff6b9d',  # ← Убрал формат
                width=1,
                dash=(3, 3)
            )

        # ===== ЦЕНТРАЛЬНАЯ ОБЛАСТЬ =====
        self.cx, self.cy = 530, 310

        # Анимированное сердечко вместо круга
        self.heart_size = 80
        self.heart = self.create_heart(self.cx, self.cy, self.heart_size, '#ff6b9d')

        # Вращающиеся элементы (звёздочки вместо колец)
        self.orbit_elements = []
        for i in range(8):
            angle = i * math.pi / 4
            element = self.canvas.create_text(
                self.cx, self.cy,
                text="✦",
                font=('Arial', 12),
                fill='#c084fc',
                tags=('orbit',)
            )
            self.orbit_elements.append({
                'id': element,
                'angle': angle,
                'radius': 120 + i * 5,
                'speed': 0.01 + i * 0.002
            })

        # Вращающиеся маленькие сердечки
        self.small_hearts = []
        for i in range(6):
            angle = i * math.pi / 3
            heart = self.canvas.create_text(
                self.cx, self.cy,
                text="♡",
                font=('Arial', 14),
                fill='#ff6b9d',
                tags=('small_heart',)
            )
            self.small_hearts.append({
                'id': heart,
                'angle': angle,
                'radius': 160 + i * 8,
                'speed': -0.015 - i * 0.001
            })

        # Статус в стиле аниме
        self.status_text = self.canvas.create_text(
            self.cx, self.cy + 140,
            text="✧ ГОТОВА ✧",
            font=('Arial', 16, 'bold'),
            fill='#ff6b9d'
        )

        # Статус подпись
        self.status_sub = self.canvas.create_text(
            self.cx, self.cy + 170,
            text="~ Ожидаю команду ~",
            font=('Arial', 11),
            fill='#a855a8'
        )

        # ===== АУДИО-ВИЗУАЛИЗАТОР (в стиле Miside) =====
        self.create_bars()

        # ===== НИЖНЯЯ ПАНЕЛЬ =====
        # Микрофон
        self.canvas.create_text(
            30, 640,
            text="🎤 Микрофон",
            font=('Arial', 10),
            fill='#a855a8',
            anchor='w'
        )

        self.mic_text = self.canvas.create_text(
            30, 660,
            text="Микрофон (WO Mic Device)",
            font=('Arial', 9),
            fill='#7c3a7c',
            anchor='w'
        )

        # Нейросети
        self.canvas.create_text(
            260, 640,
            text="🧠 Нейросети",
            font=('Arial', 10),
            fill='#a855a8',
            anchor='w'
        )

        self.ai_text = self.canvas.create_text(
            260, 660,
            text="Senior AI",
            font=('Arial', 9),
            fill='#7c3a7c',
            anchor='w'
        )

        # Ресурсы
        self.canvas.create_text(
            490, 640,
            text="📊 Ресурсы",
            font=('Arial', 10),
            fill='#a855a8',
            anchor='w'
        )

        self.ram_text = self.canvas.create_text(
            490, 660,
            text=f"RAM: {self.get_ram_usage()} MB",
            font=('Arial', 9),
            fill='#7c3a7c',
            anchor='w'
        )

        # Ссылки
        self.canvas.create_text(
            720, 640,
            text="💖 ",
            font=('Arial', 9),
            fill='#7c3a7c',
            anchor='w'
        )

        self.canvas.create_text(
            720, 660,
            text="💎 ",
            font=('Arial', 9),
            fill='#7c3a7c',
            anchor='w'
        )

        # Копирайт в стиле игры
        self.canvas.create_text(
            480, 690,
            text="© 2026 ✦ Miside Project ✦ Стелла AI",
            font=('Arial', 8),
            fill='#4a2a4a'
        )

        # ===== КОМАНДЫ (справа) =====
        self.canvas.create_text(
            750, 100,
            text="✦ ДОСТУПНЫЕ КОМАНДЫ ✦",
            font=('Arial', 10, 'bold'),
            fill='#ff6b9d',
            anchor='w'
        )

        commands = [
            "♪ Запусти [программа]",
            "♪ Открой [сайт]",
            "♪ Закрой [программа]",
            "♪ Напиши [текст]",
            "♪ Сверни [окно]",
            "♪ Смени язык",
            "♪ Включи песню",
            "♪ Клик / Скролл"
        ]

        cmd_y = 130
        for cmd in commands:
            self.canvas.create_text(
                750, cmd_y,
                text=cmd,
                font=('Arial', 9),
                fill='#a855a8',
                anchor='w'
            )
            cmd_y += 25

        # Создаём частицы (цветочки/звёздочки)
        self.create_particles()

        # Создаём звёзды на фоне
        self.create_stars()

    def create_gradient_bg(self):
        """Создаёт градиентный фон в стиле Miside"""
        for y in range(700):
            r = 26 + int((y / 700) * 20)
            g = 10 + int((y / 700) * 5)
            b = 26 + int((y / 700) * 30)
            color = f'#{r:02x}{g:02x}{b:02x}'
            self.canvas.create_line(0, y, 950, y, fill=color, tags=('bg',))

    def create_heart(self, x, y, size, color):
        """Создаёт сердечко на canvas"""
        points = []
        for t in range(0, 360, 5):
            rad = math.radians(t)
            heart_x = 16 * math.sin(rad) ** 3
            heart_y = 13 * math.cos(rad) - 5 * math.cos(2 * rad) - 2 * math.cos(3 * rad) - math.cos(4 * rad)
            points.append(x + heart_x * size / 16)  # X
            points.append(y - heart_y * size / 16)  # Y

        # Создаём полигон сердца
        heart_id = self.canvas.create_polygon(
            points,
            fill=color,
            outline=color,
            width=2,
            tags=('heart',)
        )

        # Добавляем свечение
        # Добавляем свечение
        glow_id = self.canvas.create_polygon(
            points,
            fill='',
            outline='#ff6b9d',
            width=2,
            tags=('heart_glow',)
        )
        self.canvas.itemconfig(glow_id, stipple='gray50')  # Эффект полупрозрачности
        return {'id': heart_id, 'glow': glow_id, 'points': points}

    def create_particles(self):
        """Создаёт частицы в стиле Miside (цветочки/звёздочки)"""
        emojis = ["✦", "✧", "♡", "◇", "✿"]
        for _ in range(40):
            angle = random.uniform(0, 2 * math.pi)
            radius = random.uniform(80, 250)
            speed = random.uniform(0.3, 1.5)
            self.particles.append({
                'angle': angle,
                'radius': radius,
                'speed': speed,
                'size': random.randint(8, 14),
                'emoji': random.choice(emojis),
                'id': None,
                'alpha': random.uniform(0.3, 0.7)
            })

    def create_stars(self):
        """Создаёт мерцающие звёзды на фоне"""
        for _ in range(30):
            x = random.randint(0, 950)
            y = random.randint(0, 700)
            size = random.randint(1, 3)
            star = self.canvas.create_text(
                x, y,
                text="✦",
                font=('Arial', size * 3),
                fill='#7c3a7c',
                tags=('star',)
            )
            self.stars.append({
                'id': star,
                'x': x,
                'y': y,
                'size': size,
                'alpha': random.uniform(0.2, 0.6),
                'speed': random.uniform(0.5, 2)
            })

    def create_bars(self):
        """Создаёт красивый визуализатор в стиле аниме"""
        self.bars = []
        base_x = 380

        # Используем разные формы для визуализации
        shapes = ["♡", "✦", "✧", "◇"]

        for i in range(30):
            x = base_x + i * 10

            # Выбираем случайную форму
            shape = random.choice(shapes)

            # Размер в зависимости от позиции
            size = random.randint(8, 14)

            # Цвет от розового к фиолетовому
            color_intensity = i / 30
            r = int(255 * (1 - color_intensity * 0.5))
            g = int(107 * (1 - color_intensity * 0.3))
            b = int(157 + color_intensity * 98)
            color = f'#{r:02x}{g:02x}{b:02x}'

            # Создаём текст вместо прямоугольников
            bar = self.canvas.create_text(
                x, self.cy + 180,
                text=shape,
                font=('Arial', size),
                fill=color,
                tags=('bar',)
            )

            self.bars.append({
                'id': bar,
                'x': x,
                'y': self.cy + 180,
                'size': size,
                'max_size': size,
                'target': size,
                'shape': shape,
                'color': color
            })

    def get_ram_usage(self):
        """Получает использование RAM"""
        try:
            return int(psutil.Process().memory_info().rss / 1024 / 1024)
        except:
            return 0

    def update_heart(self, size, color):
        points = []
        for t in range(0, 360, 5):
            rad = math.radians(t)
            heart_x = 16 * math.sin(rad) ** 3
            heart_y = 13 * math.cos(rad) - 5 * math.cos(2 * rad) - 2 * math.cos(3 * rad) - math.cos(4 * rad)
            points.append(self.cx + heart_x * size / 16)  # X координата
            points.append(self.cy - heart_y * size / 16)  # Y координата

        self.canvas.coords(self.heart['id'], *points)  # ← ЗВЁЗДОЧКА *
        self.canvas.coords(self.heart['glow'], *points)  # ← ЗВЁЗДОЧКА *
        self.canvas.itemconfig(self.heart['id'], fill=color, outline=color)
        self.canvas.itemconfig(self.heart['glow'], fill='#ff6b9d')  # ← Просто цвет

    def animate(self):
        """Главный цикл анимации"""
        if not self.running:
            return

        self.angle += 0.02

        # Определяем цвета для состояния
        if self.is_listening:
            status = "♪ СЛУШАЮ... ♪"
            color = '#ff2d75'
            glow = '#ff2d7544'
            bar_color_start = '#ff2d75'
            bar_color_end = '#ff6b9d'
        elif self.is_processing:
            status = "✦ ОБРАБОТКА... ✦"
            color = '#c084fc'
            glow = '#c084fc44'
            bar_color_start = '#c084fc'
            bar_color_end = '#ff6b9d'
        else:
            status = "✧ ГОТОВА ✧"
            color = '#ff6b9d'
            glow = '#ff6b9d44'
            bar_color_start = '#ff6b9d'
            bar_color_end = '#c084fc'

        # Обновляем сердце (пульсация)
        heart_pulse = 80 + math.sin(self.angle * 2) * 5
        self.update_heart(heart_pulse, color)

        # Обновляем статус
        self.canvas.itemconfig(self.status_text, text=status, fill=color)

        # Обновляем вращающиеся элементы (звёздочки)
        for elem in self.orbit_elements:
            elem['angle'] += elem['speed']
            x = self.cx + math.cos(elem['angle']) * elem['radius']
            y = self.cy + math.sin(elem['angle']) * elem['radius']
            self.canvas.coords(elem['id'], x, y)
            self.canvas.itemconfig(elem['id'], fill=color)

        # Обновляем маленькие сердечки
        for heart in self.small_hearts:
            heart['angle'] += heart['speed']
            x = self.cx + math.cos(heart['angle']) * heart['radius']
            y = self.cy + math.sin(heart['angle']) * heart['radius']
            self.canvas.coords(heart['id'], x, y)
            self.canvas.itemconfig(heart['id'], fill=color)

        # Обновляем частицы
        for particle in self.particles:
            particle['angle'] += particle['speed'] * 0.015
            x = self.cx + math.cos(particle['angle']) * particle['radius']
            y = self.cy + math.sin(particle['angle']) * particle['radius']

            if particle['id'] is None:
                particle['id'] = self.canvas.create_text(
                    x, y,
                    text=particle['emoji'],
                    font=('Arial', particle['size']),
                    fill=color,
                    tags=('particle',)
                )
            else:
                self.canvas.coords(particle['id'], x, y)
                self.canvas.itemconfig(particle['id'], fill=color)

        # Обновляем звёзды (мерцание)
        for star in self.stars:
            star['alpha'] = 0.3 + 0.3 * math.sin(time.time() * star['speed'])
            alpha = int(star['alpha'] * 255)
            self.canvas.itemconfig(star['id'], fill='#7c3a7c')  # ← Убрал формат

        # Обновляем аудио-визуализатор
        # Обновляем аудио-визуализатор (анимированные символы)
        for i, bar in enumerate(self.bars):
            if self.is_listening or self.is_processing:
                bar['target'] = random.randint(8, 28)
                bar['max_size'] += (bar['target'] - bar['max_size']) * 0.15
            else:
                bar['max_size'] += (10 - bar['max_size']) * 0.05

            new_size = max(6, bar['max_size'])

            # Меняем размер шрифта
            self.canvas.itemconfig(bar['id'], font=('Arial', int(new_size)))

            # Меняем цвет
            intensity = i / len(self.bars)
            if self.is_listening:
                r = int(255 * (1 - intensity * 0.3))
                g = int(45 * (1 - intensity * 0.5))
                b = int(117 * (1 - intensity * 0.3))
            elif self.is_processing:
                r = int(192 * (1 - intensity * 0.3))
                g = int(132 * (1 - intensity * 0.4))
                b = int(252 * (1 - intensity * 0.3))
            else:
                r = int(255 * (1 - intensity * 0.4))
                g = int(107 * (1 - intensity * 0.3))
                b = int(157 + intensity * 98)

            bar_color = f'#{r:02x}{g:02x}{b:02x}'
            self.canvas.itemconfig(bar['id'], fill=bar_color)

        # Обновляем RAM
        self.canvas.itemconfig(self.ram_text, text=f"RAM: {self.get_ram_usage()} MB")

        # Запускаем следующий кадр
        self.root.after(20, self.animate)

    def start_move(self, event):
        self.x = event.x
        self.y = event.y

    def on_move(self, event):
        deltax = event.x - self.x
        deltay = event.y - self.y
        x = self.root.winfo_x() + deltax
        y = self.root.winfo_y() + deltay
        self.root.geometry(f"+{x}+{y}")

    def set_listening(self, state):
        self.is_listening = state
        if state:
            self.is_processing = False
            self.canvas.itemconfig(self.status_sub, text="🎤 Ожидание голоса...")
        else:
            self.canvas.itemconfig(self.status_sub, text="~ Ожидаю команду ~")

    def set_processing(self, state):
        self.is_processing = state
        if state:
            self.is_listening = False
            self.canvas.itemconfig(self.status_sub, text="⚙️ Выполнение команды...")
        else:
            self.canvas.itemconfig(self.status_sub, text="~ Ожидаю команду ~")

    def show_command(self, text):
        """Показывает команду в статусе"""
        self.canvas.itemconfig(self.status_sub, text=f"✦ {text[:30]}... ✦")
        self.root.after(3000, lambda: self.canvas.itemconfig(self.status_sub, text="~ Ожидаю команду ~"))

    def quit(self):
        self.running = False
        self.root.quit()
        self.root.destroy()
        sys.exit(0)

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

# Загрузка soundfile
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
JSON_CACHE_FILE = os.path.join(os.getcwd(), "saveAppPty.json")
SOUNDS_DIR = os.path.join(os.getcwd(), "sounds")

model_path = os.path.join(MODEL_DIR, "model.int8.onnx")
tokens_path = os.path.join(MODEL_DIR, "tokens.txt")

if not os.path.exists(model_path) or not os.path.exists(tokens_path):
    print("❌ ОШИБКА: Файлы модели не найдены!")
    print(f"   Искал в: {MODEL_DIR}")
    exit(1)

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

    # СВОРАЧИВАНИЕ ВСЕХ ОКОН
    if "все" in filtered_words and any(w in filtered_words for w in ["окна", "окон", "окно"]):
        minimize_all_windows()
        interface.set_processing(False)
        return

    # Сворачивание конкретного окна
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

    # МУЗЫКА
    if any(w in phrase_after_trigger for w in ["украинскую", "укр"]) and "песн" in phrase_after_trigger:
        play_ukrainian_song()
        interface.set_processing(False)
        return

    if any(w in phrase_after_trigger for w in ["рандомную", "рандом", "случайную"]) and "песн" in phrase_after_trigger:
        play_random_song()
        interface.set_processing(False)
        return

    # ПЕРЕМЕЩЕНИЕ ОКОН
    for verb in MOVE_VERBS:
        if verb in filtered_words:
            target = " ".join([w for w in filtered_words if w != verb]).strip()
            if target:
                move_window_to_monitor(target)
            interface.set_processing(False)
            return

    # ПЕЧАТЬ ТЕКСТА
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

    # ЗАПУСК ПРИЛОЖЕНИЙ
    for verb in APP_VERBS:
        if verb in filtered_words:
            target = " ".join([w for w in filtered_words if w != verb]).strip()
            if target:
                launch_application(target)
            interface.set_processing(False)
            return

    # ОТКРЫТИЕ САЙТОВ
    for verb in WEB_VERBS:
        if verb in filtered_words:
            target = " ".join([w for w in filtered_words if w != verb]).strip()
            if target:
                open_website(target)
            interface.set_processing(False)
            return

    # ЗАКРЫТИЕ ПРИЛОЖЕНИЙ
    for verb in CLOSE_VERBS:
        if verb in filtered_words:
            target = " ".join([w for w in filtered_words if w != verb]).strip()
            if target:
                kill_application(target)
            interface.set_processing(False)
            return

    # СМЕНА ЯЗЫКА
    for verb in LANG_VERBS:
        if verb in filtered_words:
            target = " ".join([w for w in filtered_words if w != verb]).strip()
            if target in ["язык", "раскладку"]:
                change_language()
            interface.set_processing(False)
            return

    # КОМАНДЫ УПРАВЛЕНИЯ
    if phrase_after_trigger in CONTROL_COMMANDS:
        CONTROL_COMMANDS[phrase_after_trigger]()
        interface.set_processing(False)
        return

    # РАЗГОВОРНЫЕ ОТВЕТЫ
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
    # Создаём интерфейс (в главном потоке)
    interface = MisideInterface()

    # Создаём папку для звуков
    if not os.path.exists(SOUNDS_DIR):
        os.makedirs(SOUNDS_DIR)

    load_app_cache()

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
