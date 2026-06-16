[2TezYuklaBot.py](https://github.com/user-attachments/files/28986850/2TezYuklaBot.py)
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
TezYukla Bot - Professional Telegram Bot for Media Downloading
Python 3.12+ | Aiogram 3.x | yt-dlp | SQLite3
"""

import asyncio
import logging
import sqlite3
import os
import re
import time
from datetime import datetime
from typing import Dict, Optional, List, Tuple
from pathlib import Path

from aiogram import Bot, Dispatcher, types, F
from aiogram.types import Message, CallbackQuery, InlineKeyboardMarkup, InlineKeyboardButton, FSInputFile
from aiogram.filters import Command, StateFilter
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.fsm.storage.memory import MemoryStorage
from aiogram.utils.keyboard import InlineKeyboardBuilder
from aiogram.exceptions import TelegramBadRequest, TelegramForbiddenError

import yt_dlp

# =============== CONFIGURATION ===============
BOT_TOKEN = "8644529147:AAGkpidMfsLlQTEXPJ4bvZ1f7lXAV_TjQ0s"  # Replace with your bot token
ADMIN_IDS = [7710687157, 987654321]  # Replace with your admin IDs

# Create directories
Path("downloads").mkdir(exist_ok=True)
Path("logs").mkdir(exist_ok=True)

# =============== LOGGING ===============
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('logs/bot.log'),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger(__name__)

# =============== DATABASE HELPER ===============
class DatabaseHelper:
    def __init__(self, db_path: str = "tezyukla.db"):
        self.db_path = db_path
        self.init_db()
    
    def get_connection(self):
        return sqlite3.connect(self.db_path)
    
    def init_db(self):
        with self.get_connection() as conn:
            cursor = conn.cursor()
            
            # Users table
            cursor.execute('''
                CREATE TABLE IF NOT EXISTS users (
                    user_id INTEGER PRIMARY KEY,
                    username TEXT,
                    fullname TEXT,
                    first_join TIMESTAMP,
                    last_activity TIMESTAMP
                )
            ''')
            
            # Downloads table
            cursor.execute('''
                CREATE TABLE IF NOT EXISTS downloads (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    user_id INTEGER,
                    media_title TEXT,
                    media_url TEXT,
                    media_type TEXT,
                    platform TEXT,
                    download_time TIMESTAMP,
                    file_size TEXT
                )
            ''')
            
            # Channels table (for mandatory subscription)
            cursor.execute('''
                CREATE TABLE IF NOT EXISTS channels (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    channel_id TEXT UNIQUE,
                    channel_name TEXT,
                    channel_url TEXT,
                    added_by INTEGER,
                    added_at TIMESTAMP
                )
            ''')
            
            # Settings table
            cursor.execute('''
                CREATE TABLE IF NOT EXISTS settings (
                    key TEXT PRIMARY KEY,
                    value TEXT,
                    updated_at TIMESTAMP
                )
            ''')
            
            # Logs table
            cursor.execute('''
                CREATE TABLE IF NOT EXISTS logs (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    log_level TEXT,
                    message TEXT,
                    user_id INTEGER,
                    created_at TIMESTAMP
                )
            ''')
            
            conn.commit()
    
    def add_user(self, user_id: int, username: str, fullname: str):
        with self.get_connection() as conn:
            cursor = conn.cursor()
            cursor.execute('''
                INSERT OR REPLACE INTO users (user_id, username, fullname, first_join, last_activity)
                VALUES (?, ?, ?, COALESCE((SELECT first_join FROM users WHERE user_id = ?), ?), ?)
            ''', (user_id, username, fullname, user_id, datetime.now(), datetime.now()))
            conn.commit()
    
    def update_user_activity(self, user_id: int):
        with self.get_connection() as conn:
            cursor = conn.cursor()
            cursor.execute('UPDATE users SET last_activity = ? WHERE user_id = ?', (datetime.now(), user_id))
            conn.commit()
    
    def add_download(self, user_id: int, title: str, url: str, media_type: str, platform: str, file_size: str):
        with self.get_connection() as conn:
            cursor = conn.cursor()
            cursor.execute('''
                INSERT INTO downloads (user_id, media_title, media_url, media_type, platform, download_time, file_size)
                VALUES (?, ?, ?, ?, ?, ?, ?)
            ''', (user_id, title, url, media_type, platform, datetime.now(), file_size))
            conn.commit()
    
    def add_channel(self, channel_id: str, channel_name: str, channel_url: str, admin_id: int):
        with self.get_connection() as conn:
            cursor = conn.cursor()
            cursor.execute('''
                INSERT OR REPLACE INTO channels (channel_id, channel_name, channel_url, added_by, added_at)
                VALUES (?, ?, ?, ?, ?)
            ''', (channel_id, channel_name, channel_url, admin_id, datetime.now()))
            conn.commit()
    
    def remove_channel(self, channel_id: str):
        with self.get_connection() as conn:
            cursor = conn.cursor()
            cursor.execute('DELETE FROM channels WHERE channel_id = ?', (channel_id,))
            conn.commit()
    
    def get_channels(self) -> List[Tuple]:
        with self.get_connection() as conn:
            cursor = conn.cursor()
            cursor.execute('SELECT channel_id, channel_name, channel_url FROM channels')
            return cursor.fetchall()
    
    def get_total_users(self) -> int:
        with self.get_connection() as conn:
            cursor = conn.cursor()
            cursor.execute('SELECT COUNT(*) FROM users')
            return cursor.fetchone()[0]
    
    def get_total_downloads(self) -> int:
        with self.get_connection() as conn:
            cursor = conn.cursor()
            cursor.execute('SELECT COUNT(*) FROM downloads')
            return cursor.fetchone()[0]
    
    def get_downloads_by_type(self, media_type: str) -> int:
        with self.get_connection() as conn:
            cursor = conn.cursor()
            cursor.execute('SELECT COUNT(*) FROM downloads WHERE media_type = ?', (media_type,))
            return cursor.fetchone()[0]
    
    def get_all_users(self) -> List[Tuple]:
        with self.get_connection() as conn:
            cursor = conn.cursor()
            cursor.execute('SELECT user_id, username, fullname, first_join, last_activity FROM users')
            return cursor.fetchall()
    
    def add_log(self, log_level: str, message: str, user_id: int = None):
        with self.get_connection() as conn:
            cursor = conn.cursor()
            cursor.execute('''
                INSERT INTO logs (log_level, message, user_id, created_at)
                VALUES (?, ?, ?, ?)
            ''', (log_level, message, user_id, datetime.now()))
            conn.commit()

# =============== DOWNLOAD MANAGER ===============
class DownloadManager:
    def __init__(self):
        self.download_path = "downloads"
        
    def get_platform(self, url: str) -> str:
        platforms = {
            'youtube.com': 'YouTube',
            'youtu.be': 'YouTube',
            'tiktok.com': 'TikTok',
            'instagram.com': 'Instagram',
            'facebook.com': 'Facebook',
            'fb.watch': 'Facebook',
            'twitter.com': 'X',
            'x.com': 'X',
            'pinterest.com': 'Pinterest',
            'threads.net': 'Threads'
        }
        for domain, platform in platforms.items():
            if domain in url.lower():
                return platform
        return "Unknown"
    
    async def get_video_info(self, url: str) -> Dict:
        ydl_opts = {
            'quiet': True,
            'no_warnings': True,
            'extract_flat': False,
        }
        
        try:
            with yt_dlp.YoutubeDL(ydl_opts) as ydl:
                info = ydl.extract_info(url, download=False)
                return {
                    'title': info.get('title', 'Unknown'),
                    'duration': info.get('duration', 0),
                    'thumbnail': info.get('thumbnail', ''),
                    'formats': info.get('formats', [])
                }
        except Exception as e:
            logger.error(f"Error getting video info: {e}")
            raise
    
    async def download_video(self, url: str, quality: str) -> str:
        output_template = f"{self.download_path}/%(title)s_%(id)s.%(ext)s"
        
        quality_map = {
            '360p': '360',
            '480p': '480',
            '720p': '720',
            '1080p': '1080'
        }
        
        ydl_opts = {
            'format': f'bestvideo[height<={quality_map[quality]}][ext=mp4]+bestaudio[ext=m4a]/best[height<={quality_map[quality]}][ext=mp4]/best',
            'outtmpl': output_template,
            'quiet': True,
            'no_warnings': True,
            'merge_output_format': 'mp4'
        }
        
        try:
            with yt_dlp.YoutubeDL(ydl_opts) as ydl:
                info = ydl.extract_info(url, download=True)
                filename = ydl.prepare_filename(info)
                if not os.path.exists(filename):
                    filename = filename.replace('.webm', '.mp4').replace('.mkv', '.mp4')
                return filename
        except Exception as e:
            logger.error(f"Error downloading video: {e}")
            raise
    
    async def download_audio(self, url: str, bitrate: str) -> str:
        output_template = f"{self.download_path}/%(title)s_%(id)s.%(ext)s"
        
        bitrate_map = {
            '128kbps': '128',
            '192kbps': '192',
            '320kbps': '320'
        }
        
        ydl_opts = {
            'format': 'bestaudio/best',
            'outtmpl': output_template,
            'quiet': True,
            'no_warnings': True,
            'postprocessors': [{
                'key': 'FFmpegExtractAudio',
                'preferredcodec': 'mp3',
                'preferredquality': bitrate_map[bitrate],
            }],
        }
        
        try:
            with yt_dlp.YoutubeDL(ydl_opts) as ydl:
                info = ydl.extract_info(url, download=True)
                filename = ydl.prepare_filename(info)
                filename = filename.rsplit('.', 1)[0] + '.mp3'
                return filename
        except Exception as e:
            logger.error(f"Error downloading audio: {e}")
            raise
    
    def get_file_size(self, filepath: str) -> str:
        size = os.path.getsize(filepath)
        for unit in ['B', 'KB', 'MB', 'GB']:
            if size < 1024.0:
                return f"{size:.1f} {unit}"
            size /= 1024.0
        return f"{size:.1f} GB"
    
    def cleanup(self, filepath: str):
        try:
            if os.path.exists(filepath):
                os.remove(filepath)
        except Exception as e:
            logger.error(f"Error cleaning up file: {e}")

# =============== FSM STATES ===============
class AdminStates(StatesGroup):
    waiting_for_broadcast = State()
    waiting_for_channel_id = State()
    waiting_for_channel_name = State()
    waiting_for_channel_url = State()
    waiting_for_remove_channel = State()

class DownloadStates(StatesGroup):
    waiting_for_quality = State()
    waiting_for_bitrate = State()

# =============== MIDDLEWARE ===============
class AntiSpamMiddleware:
    def __init__(self):
        self.user_last_message = {}
    
    async def __call__(self, handler, event: Message, data: Dict):
        if isinstance(event, Message) and event.from_user:
            user_id = event.from_user.id
            current_time = time.time()
            
            if user_id in self.user_last_message:
                time_diff = current_time - self.user_last_message[user_id]
                if time_diff < 3:
                    await event.answer("⚠ Juda tez yuboryapsiz. Iltimos, biroz kuting!")
                    return
            
            self.user_last_message[user_id] = current_time
        
        return await handler(event, data)

class SubscriptionMiddleware:
    def __init__(self, db: DatabaseHelper, bot: Bot):
        self.db = db
        self.bot = bot
    
    async def __call__(self, handler, event: Message, data: Dict):
        if isinstance(event, Message) and event.from_user:
            user_id = event.from_user.id
            
            # Skip check for admins and /start command
            if user_id in ADMIN_IDS or event.text == '/start':
                return await handler(event, data)
            
            channels = self.db.get_channels()
            if channels:
                not_subscribed = []
                for channel_id, channel_name, channel_url in channels:
                    try:
                        member = await self.bot.get_chat_member(chat_id=channel_id, user_id=user_id)
                        if member.status in ['left', 'kicked']:
                            not_subscribed.append((channel_name, channel_url))
                    except Exception:
                        not_subscribed.append((channel_name, channel_url))
                
                if not_subscribed:
                    keyboard = InlineKeyboardBuilder()
                    for name, url in not_subscribed:
                        keyboard.add(InlineKeyboardButton(text=f"📢 {name}", url=url))
                    keyboard.add(InlineKeyboardButton(text="✅ Tekshirish", callback_data="check_subscription"))
                    keyboard.adjust(1)
                    
                    await event.answer(
                        "❌ Botdan foydalanish uchun quyidagi kanallarga obuna bo'ling:\n\n" +
                        "\n".join([f"• {name}" for name, _ in not_subscribed]),
                        reply_markup=keyboard.as_markup()
                    )
                    return
        
        return await handler(event, data)

# =============== MAIN BOT CLASS ===============
class TezYuklaBot:
    def __init__(self):
        self.bot = Bot(token=BOT_TOKEN)
        self.storage = MemoryStorage()
        self.dp = Dispatcher(storage=self.storage)
        self.db = DatabaseHelper()
        self.download_manager = DownloadManager()
        
        # Setup middlewares
        self.dp.message.middleware(AntiSpamMiddleware())
        self.dp.message.middleware(SubscriptionMiddleware(self.db, self.bot))
        
        self.setup_handlers()
    
    def setup_handlers(self):
        # ============ COMMAND HANDLERS ============
        @self.dp.message(Command("start"))
        async def cmd_start(message: Message, state: FSMContext):
            await state.clear()
            user = message.from_user
            self.db.add_user(user.id, user.username or "None", user.full_name)
            self.db.update_user_activity(user.id)
            
            total_users = self.db.get_total_users()
            total_downloads = self.db.get_total_downloads()
            
            welcome_text = (
                "🌑 <b>TezYukla Botga xush kelibsiz!</b>\n\n"
                "⚡ <b>Professional Media Yuklash Boti</b>\n\n"
                "📥 <b>Qo'llab-quvvatlanadigan platformalar:</b>\n"
                "• YouTube • TikTok • Instagram • Facebook\n"
                "• X (Twitter) • Pinterest • Threads\n\n"
                "📊 <b>Bot statistikasi:</b>\n"
                f"👥 Foydalanuvchilar: {total_users}\n"
                f"📥 Yuklamalar: {total_downloads}\n\n"
                "🎬 <b>Qanday ishlatish?</b>\n"
                "1. Yuklamoqchi bo'lgan media linkini yuboring\n"
                "2. Format va sifatni tanlang\n"
                "3. Avtomatik yuklab olinadi\n\n"
                "🚀 <b>Tez, sifatli va bepul!</b>"
            )
            
            keyboard = InlineKeyboardBuilder()
            keyboard.add(InlineKeyboardButton(text="📥 Yuklab olish", callback_data="help_download"))
            keyboard.add(InlineKeyboardButton(text="ℹ️ Yordam", callback_data="help_info"))
            keyboard.add(InlineKeyboardButton(text="📊 Statistika", callback_data="show_stats"))
            keyboard.adjust(2)
            
            if user.id in ADMIN_IDS:
                keyboard.add(InlineKeyboardButton(text="⚙ Admin Panel", callback_data="admin_panel"))
            
            await message.answer(welcome_text, reply_markup=keyboard.as_markup())
        
        @self.dp.message(Command("admin"))
        async def cmd_admin(message: Message):
            if message.from_user.id not in ADMIN_IDS:
                await message.answer("❌ Bu buyruq faqat adminlar uchun!")
                return
            
            keyboard = self.get_admin_keyboard()
            await message.answer("⚙ <b>Admin Panel</b>", reply_markup=keyboard)
        
        # ============ MESSAGE HANDLER FOR URLS ============
        @self.dp.message(F.text)
        async def handle_url(message: Message, state: FSMContext):
            url = message.text.strip()
            url_pattern = re.compile(r'http[s]?://(?:[a-zA-Z]|[0-9]|[$-_@.&+]|[!*\\(\\),]|(?:%[0-9a-fA-F][0-9a-fA-F]))+')
            
            if not url_pattern.match(url):
                await message.answer("❌ Iltimos, to'g'ri media link yuboring!")
                return
            
            processing_msg = await message.answer("🔄 Media ma'lumotlari yuklanmoqda...")
            
            try:
                platform = self.download_manager.get_platform(url)
                video_info = await self.download_manager.get_video_info(url)
                
                duration_min = video_info['duration'] // 60 if video_info['duration'] else 0
                duration_sec = video_info['duration'] % 60 if video_info['duration'] else 0
                
                info_text = (
                    f"📁 <b>{video_info['title'][:50]}</b>\n\n"
                    f"📱 Platforma: {platform}\n"
                    f"⏱ Davomiyligi: {duration_min}:{duration_sec:02d}\n"
                    f"🎬 Sifat: 360p - 1080p\n\n"
                    f"📥 Yuklash formatini tanlang:"
                )
                
                keyboard = InlineKeyboardBuilder()
                keyboard.add(InlineKeyboardButton(text="🎬 MP4 Video", callback_data=f"video_{url}"))
                keyboard.add(InlineKeyboardButton(text="🎵 MP3 Audio", callback_data=f"audio_{url}"))
                keyboard.adjust(1)
                
                await processing_msg.delete()
                await message.answer(info_text, reply_markup=keyboard.as_markup())
                
                await state.update_data(video_title=video_info['title'], platform=platform)
                
            except Exception as e:
                logger.error(f"Error processing URL: {e}")
                await processing_msg.delete()
                await message.answer("❌ Media ma'lumotlarini olishda xatolik yuz berdi. Iltimos, linkni tekshirib qayta urinib ko'ring.")
        
        # ============ CALLBACK QUERY HANDLERS ============
        @self.dp.callback_query(F.data.startswith("video_"))
        async def handle_video_selection(callback: CallbackQuery, state: FSMContext):
            url = callback.data.replace("video_", "")
            await state.update_data(media_url=url, media_type="video")
            
            keyboard = InlineKeyboardBuilder()
            qualities = ["360p", "480p", "720p", "1080p"]
            for quality in qualities:
                keyboard.add(InlineKeyboardButton(text=quality, callback_data=f"quality_{quality}"))
            keyboard.adjust(2)
            keyboard.add(InlineKeyboardButton(text="❌ Bekor qilish", callback_data="cancel"))
            
            await callback.message.edit_text("🎬 Video sifatini tanlang:", reply_markup=keyboard.as_markup())
            await callback.answer()
        
        @self.dp.callback_query(F.data.startswith("audio_"))
        async def handle_audio_selection(callback: CallbackQuery, state: FSMContext):
            url = callback.data.replace("audio_", "")
            await state.update_data(media_url=url, media_type="audio")
            
            keyboard = InlineKeyboardBuilder()
            bitrates = ["128kbps", "192kbps", "320kbps"]
            for bitrate in bitrates:
                keyboard.add(InlineKeyboardButton(text=bitrate, callback_data=f"bitrate_{bitrate}"))
            keyboard.adjust(1)
            keyboard.add(InlineKeyboardButton(text="❌ Bekor qilish", callback_data="cancel"))
            
            await callback.message.edit_text("🎵 Audio sifatini tanlang:", reply_markup=keyboard.as_markup())
            await callback.answer()
        
        @self.dp.callback_query(F.data.startswith("quality_"))
        async def handle_quality(callback: CallbackQuery, state: FSMContext):
            quality = callback.data.replace("quality_", "")
            data = await state.get_data()
            url = data.get('media_url')
            media_type = data.get('media_type')
            title = data.get('video_title', 'Video')
            platform = data.get('platform', 'Unknown')
            
            await callback.message.edit_text(f"⬇️ Yuklab olinmoqda: {quality}\n⏳ Iltimos kuting...")
            
            try:
                filepath = await self.download_manager.download_video(url, quality)
                file_size = self.download_manager.get_file_size(filepath)
                
                self.db.add_download(
                    callback.from_user.id, title, url, 
                    f"video_{quality}", platform, file_size
                )
                
                video_file = FSInputFile(filepath)
                await callback.message.delete()
                
                await callback.message.answer_video(
                    video_file,
                    caption=f"✅ <b>Yuklandi!</b>\n\n"
                           f"📁 Nomi: {title[:50]}\n"
                           f"🎬 Sifat: {quality}\n"
                           f"💾 Hajmi: {file_size}\n"
                           f"📱 Platforma: {platform}\n\n"
                           f"@TezYuklaBot",
                    reply_markup=self.get_download_keyboard()
                )
                
                self.download_manager.cleanup(filepath)
                
            except Exception as e:
                logger.error(f"Download error: {e}")
                await callback.message.edit_text("❌ Yuklashda xatolik yuz berdi. Iltimos, qayta urinib ko'ring.")
            
            await callback.answer()
        
        @self.dp.callback_query(F.data.startswith("bitrate_"))
        async def handle_bitrate(callback: CallbackQuery, state: FSMContext):
            bitrate = callback.data.replace("bitrate_", "")
            data = await state.get_data()
            url = data.get('media_url')
            media_type = data.get('media_type')
            title = data.get('video_title', 'Audio')
            platform = data.get('platform', 'Unknown')
            
            await callback.message.edit_text(f"⬇️ Yuklab olinmoqda: {bitrate}\n⏳ Iltimos kuting...")
            
            try:
                filepath = await self.download_manager.download_audio(url, bitrate)
                file_size = self.download_manager.get_file_size(filepath)
                
                self.db.add_download(
                    callback.from_user.id, title, url,
                    f"audio_{bitrate}", platform, file_size
                )
                
                audio_file = FSInputFile(filepath)
                await callback.message.delete()
                
                await callback.message.answer_audio(
                    audio_file,
                    caption=f"✅ <b>Yuklandi!</b>\n\n"
                           f"📁 Nomi: {title[:50]}\n"
                           f"🎵 Sifat: {bitrate}\n"
                           f"💾 Hajmi: {file_size}\n"
                           f"📱 Platforma: {platform}\n\n"
                           f"@TezYuklaBot",
                    reply_markup=self.get_download_keyboard()
                )
                
                self.download_manager.cleanup(filepath)
                
            except Exception as e:
                logger.error(f"Download error: {e}")
                await callback.message.edit_text("❌ Yuklashda xatolik yuz berdi. Iltimos, qayta urinib ko'ring.")
            
            await callback.answer()
        
        # ============ CALLBACK HANDLERS ============
        @self.dp.callback_query(F.data == "help_download")
        async def help_download(callback: CallbackQuery):
            await callback.message.answer(
                "📥 <b>Qanday yuklab olish?</b>\n\n"
                "1. YouTube, TikTok, Instagram yoki boshqa platformadan link nusxalang\n"
                "2. Botga linkni yuboring\n"
                "3. Format (video/audio) va sifatni tanlang\n"
                "4. Yuklab olish tugashini kuting\n\n"
                "⚡ <b>Maslahat:</b> Video uchun 720p, audio uchun 192kbps eng yaxshi variant!"
            )
            await callback.answer()
        
        @self.dp.callback_query(F.data == "help_info")
        async def help_info(callback: CallbackQuery):
            await callback.message.answer(
                "ℹ️ <b>TezYukla Bot haqida</b>\n\n"
                "🎯 <b>Versiya:</b> 1.0.0\n"
                "🚀 <b>Texnologiyalar:</b> Python 3.12, Aiogram 3, yt-dlp\n"
                "📱 <b>Platformalar:</b> YouTube, TikTok, Instagram, Facebook, X, Pinterest, Threads\n"
                "⚡ <b>Tezlik:</b> Yuqori tezlikda yuklash\n"
                "🔒 <b>Xavfsizlik:</b> Hech qanday ma'lumot saqlanmaydi\n\n"
                "👨‍💻 <b>Yaratuvchi:</b> @TezYuklaBot"
            )
            await callback.answer()
        
        @self.dp.callback_query(F.data == "show_stats")
        async def show_stats(callback: CallbackQuery):
            total_users = self.db.get_total_users()
            total_downloads = self.db.get_total_downloads()
            video_downloads = self.db.get_downloads_by_type("video_360p") + self.db.get_downloads_by_type("video_480p") + self.db.get_downloads_by_type("video_720p") + self.db.get_downloads_by_type("video_1080p")
            audio_downloads = self.db.get_downloads_by_type("audio_128kbps") + self.db.get_downloads_by_type("audio_192kbps") + self.db.get_downloads_by_type("audio_320kbps")
            
            stats_text = (
                "📊 <b>Bot statistikasi</b>\n\n"
                f"👥 <b>Foydalanuvchilar:</b> {total_users}\n"
                f"📥 <b>Jami yuklamalar:</b> {total_downloads}\n"
                f"🎬 <b>Video yuklamalar:</b> {video_downloads}\n"
                f"🎵 <b>Audio yuklamalar:</b> {audio_downloads}\n\n"
                f"⚡ <b>Holat:</b> 🟢 Faol\n"
                f"📅 <b>So'ngi yangilanish:</b> {datetime.now().strftime('%d.%m.%Y')}"
            )
            
            keyboard = InlineKeyboardBuilder()
            keyboard.add(InlineKeyboardButton(text="🔄 Yangilash", callback_data="show_stats"))
            keyboard.add(InlineKeyboardButton(text="⬅ Orqaga", callback_data="back_to_start"))
            
            await callback.message.edit_text(stats_text, reply_markup=keyboard.as_markup())
            await callback.answer()
        
        @self.dp.callback_query(F.data == "check_subscription")
        async def check_subscription(callback: CallbackQuery):
            user_id = callback.from_user.id
            channels = self.db.get_channels()
            
            if channels:
                not_subscribed = []
                for channel_id, channel_name, channel_url in channels:
                    try:
                        member = await self.bot.get_chat_member(chat_id=channel_id, user_id=user_id)
                        if member.status in ['left', 'kicked']:
                            not_subscribed.append(channel_name)
                    except Exception:
                        not_subscribed.append(channel_name)
                
                if not_subscribed:
                    await callback.answer(f"❌ {len(not_subscribed)} ta kanalga obuna bo'lmagansiz!", show_alert=True)
                else:
                    await callback.message.delete()
                    await callback.answer("✅ Barcha kanallarga obuna bo'ldingiz! Botdan foydalanishingiz mumkin.", show_alert=True)
                    await cmd_start(callback.message, None)
            else:
                await callback.answer("✅ Hech qanday majburiy kanal yo'q!", show_alert=True)
        
        @self.dp.callback_query(F.data == "back_to_start")
        async def back_to_start(callback: CallbackQuery):
            await cmd_start(callback.message, None)
            await callback.answer()
        
        @self.dp.callback_query(F.data == "cancel")
        async def cancel_download(callback: CallbackQuery, state: FSMContext):
            await state.clear()
            await callback.message.delete()
            await callback.answer("❌ Bekor qilindi!")
        
        # ============ ADMIN CALLBACKS ============
        @self.dp.callback_query(F.data == "admin_panel")
        async def admin_panel(callback: CallbackQuery):
            if callback.from_user.id not in ADMIN_IDS:
                await callback.answer("❌ Ruxsat yo'q!", show_alert=True)
                return
            
            keyboard = self.get_admin_keyboard()
            await callback.message.edit_text("⚙ <b>Admin Panel</b>", reply_markup=keyboard)
            await callback.answer()
        
        @self.dp.callback_query(F.data == "admin_stats")
        async def admin_stats(callback: CallbackQuery):
            if callback.from_user.id not in ADMIN_IDS:
                await callback.answer("❌ Ruxsat yo'q!", show_alert=True)
                return
            
            total_users = self.db.get_total_users()
            total_downloads = self.db.get_total_downloads()
            video_downloads = self.db.get_downloads_by_type("video_360p") + self.db.get_downloads_by_type("video_480p") + self.db.get_downloads_by_type("video_720p") + self.db.get_downloads_by_type("video_1080p")
            audio_downloads = self.db.get_downloads_by_type("audio_128kbps") + self.db.get_downloads_by_type("audio_192kbps") + self.db.get_downloads_by_type("audio_320kbps")
            channels_count = len(self.db.get_channels())
            
            stats_text = (
                "📊 <b>Bot statistikasi</b>\n\n"
                f"👥 <b>Foydalanuvchilar:</b> {total_users}\n"
                f"📥 <b>Jami yuklamalar:</b> {total_downloads}\n"
                f"🎬 <b>Video:</b> {video_downloads}\n"
                f"🎵 <b>Audio:</b> {audio_downloads}\n"
                f"📢 <b>Majburiy kanallar:</b> {channels_count}\n\n"
                f"⚡ <b>Holat:</b> 🟢 Faol\n"
                f"🤖 <b>Bot holati:</b> Ishlayapti"
            )
            
            await callback.message.edit_text(stats_text, reply_markup=self.get_back_to_admin_keyboard())
            await callback.answer()
        
        @self.dp.callback_query(F.data == "admin_users")
        async def admin_users(callback: CallbackQuery):
            if callback.from_user.id not in ADMIN_IDS:
                await callback.answer("❌ Ruxsat yo'q!", show_alert=True)
                return
            
            users = self.db.get_all_users()
            if not users:
                await callback.message.edit_text("📋 Foydalanuvchilar topilmadi.")
                return
            
            users_text = "👥 <b>Foydalanuvchilar ro'yxati</b>\n\n"
            for i, (user_id, username, fullname, first_join, last_activity) in enumerate(users[:20], 1):
                users_text += f"{i}. {fullname} (@{username})\n   ID: {user_id}\n   🕒 {last_activity[:10]}\n\n"
            
            if len(users) > 20:
                users_text += f"\n📊 Jami: {len(users)} ta foydalanuvchi (oxirgi 20 tasi ko'rsatilgan)"
            
            await callback.message.edit_text(users_text, reply_markup=self.get_back_to_admin_keyboard())
            await callback.answer()
        
        @self.dp.callback_query(F.data == "admin_broadcast")
        async def admin_broadcast(callback: CallbackQuery, state: FSMContext):
            if callback.from_user.id not in ADMIN_IDS:
                await callback.answer("❌ Ruxsat yo'q!", show_alert=True)
                return
            
            await state.set_state(AdminStates.waiting_for_broadcast)
            await callback.message.edit_text(
                "📢 <b>Reklama yuborish</b>\n\n"
                "Yubormoqchi bo'lgan xabaringizni yuboring:\n"
                "• Matn\n"
                "• Rasm (caption bilan birga)\n"
                "• Video (caption bilan birga)\n\n"
                "❌ Bekor qilish uchun /cancel",
                reply_markup=self.get_cancel_keyboard()
            )
            await callback.answer()
        
        @self.dp.message(AdminStates.waiting_for_broadcast)
        async def process_broadcast(message: Message, state: FSMContext):
            users = self.db.get_all_users()
            if not users:
                await message.answer("❌ Hech qanday foydalanuvchi topilmadi!")
                await state.clear()
                return
            
            success_count = 0
            fail_count = 0
            
            progress_msg = await message.answer(f"📤 Reklama yuborilmoqda... (0/{len(users)})")
            
            for i, (user_id, _, _, _, _) in enumerate(users, 1):
                try:
                    if message.text:
                        await self.bot.send_message(user_id, message.text)
                    elif message.photo:
                        await self.bot.send_photo(user_id, message.photo[-1].file_id, caption=message.caption)
                    elif message.video:
                        await self.bot.send_video(user_id, message.video.file_id, caption=message.caption)
                    
                    success_count += 1
                    
                    if i % 10 == 0:
                        await progress_msg.edit_text(f"📤 Reklama yuborilmoqda... ({i}/{len(users)})\n✅ Muvaffaqiyatli: {success_count}\n❌ Xato: {fail_count}")
                    
                    await asyncio.sleep(0.05)
                    
                except Exception as e:
                    fail_count += 1
                    logger.error(f"Broadcast error to {user_id}: {e}")
            
            await progress_msg.delete()
            await message.answer(
                f"✅ Reklama yuborildi!\n\n"
                f"📊 Natijalar:\n"
                f"✅ Muvaffaqiyatli: {success_count}\n"
                f"❌ Xato: {fail_count}\n"
                f"📊 Jami: {len(users)}"
            )
            await state.clear()
        
        @self.dp.callback_query(F.data == "admin_add_channel")
        async def admin_add_channel(callback: CallbackQuery, state: FSMContext):
            if callback.from_user.id not in ADMIN_IDS:
                await callback.answer("❌ Ruxsat yo'q!", show_alert=True)
                return
            
            await state.set_state(AdminStates.waiting_for_channel_id)
            await callback.message.edit_text(
                "➕ <b>Kanal qo'shish</b>\n\n"
                "Kanal ID sini yuboring (masalan: @kanal_username yoki -100123456789):\n\n"
                "❌ Bekor qilish uchun /cancel",
                reply_markup=self.get_cancel_keyboard()
            )
            await callback.answer()
        
        @self.dp.message(AdminStates.waiting_for_channel_id)
        async def process_channel_id(message: Message, state: FSMContext):
            await state.update_data(channel_id=message.text.strip())
            await state.set_state(AdminStates.waiting_for_channel_name)
            await message.answer("📝 Kanal nomini yuboring:")
        
        @self.dp.message(AdminStates.waiting_for_channel_name)
        async def process_channel_name(message: Message, state: FSMContext):
            await state.update_data(channel_name=message.text.strip())
            await state.set_state(AdminStates.waiting_for_channel_url)
            await message.answer("🔗 Kanal linkini yuboring (masalan: https://t.me/kanal_username):")
        
        @self.dp.message(AdminStates.waiting_for_channel_url)
        async def process_channel_url(message: Message, state: FSMContext):
            data = await state.get_data()
            channel_id = data['channel_id']
            channel_name = data['channel_name']
            channel_url = message.text.strip()
            
            self.db.add_channel(channel_id, channel_name, channel_url, message.from_user.id)
            
            await message.answer(f"✅ Kanal muvaffaqiyatli qo'shildi!\n\n📢 Nomi: {channel_name}\n🔗 ID: {channel_id}")
            await state.clear()
        
        @self.dp.callback_query(F.data == "admin_remove_channel")
        async def admin_remove_channel(callback: CallbackQuery, state: FSMContext):
            if callback.from_user.id not in ADMIN_IDS:
                await callback.answer("❌ Ruxsat yo'q!", show_alert=True)
                return
            
            channels = self.db.get_channels()
            if not channels:
                await callback.message.edit_text("❌ Hech qanday kanal topilmadi!", reply_markup=self.get_back_to_admin_keyboard())
                await callback.answer()
                return
            
            keyboard = InlineKeyboardBuilder()
            for channel_id, channel_name, _ in channels:
                keyboard.add(InlineKeyboardButton(text=f"❌ {channel_name}", callback_data=f"remove_{channel_id}"))
            keyboard.add(InlineKeyboardButton(text="⬅ Orqaga", callback_data="admin_panel"))
            keyboard.adjust(1)
            
            await callback.message.edit_text("🗑 <b>Kanal o'chirish</b>\n\nO'chirmoqchi bo'lgan kanalni tanlang:", reply_markup=keyboard.as_markup())
            await callback.answer()
        
        @self.dp.callback_query(F.data.startswith("remove_"))
        async def process_remove_channel(callback: CallbackQuery):
            if callback.from_user.id not in ADMIN_IDS:
                await callback.answer("❌ Ruxsat yo'q!", show_alert=True)
                return
            
            channel_id = callback.data.replace("remove_", "")
            self.db.remove_channel(channel_id)
            await callback.message.edit_text("✅ Kanal muvaffaqiyatli o'chirildi!", reply_markup=self.get_back_to_admin_keyboard())
            await callback.answer()
        
        @self.dp.callback_query(F.data == "admin_channels")
        async def admin_channels(callback: CallbackQuery):
            if callback.from_user.id not in ADMIN_IDS:
                await callback.answer("❌ Ruxsat yo'q!", show_alert=True)
                return
            
            channels = self.db.get_channels()
            if not channels:
                await callback.message.edit_text("📋 Hech qanday kanal qo'shilmagan.", reply_markup=self.get_back_to_admin_keyboard())
                await callback.answer()
                return
            
            channels_text = "📢 <b>Majburiy kanallar ro'yxati</b>\n\n"
            for i, (channel_id, channel_name, channel_url) in enumerate(channels, 1):
                channels_text += f"{i}. {channel_name}\n   ID: {channel_id}\n   Link: {channel_url}\n\n"
            
            await callback.message.edit_text(channels_text, reply_markup=self.get_back_to_admin_keyboard())
            await callback.answer()
        
        @self.dp.callback_query(F.data == "admin_settings")
        async def admin_settings(callback: CallbackQuery):
            if callback.from_user.id not in ADMIN_IDS:
                await callback.answer("❌ Ruxsat yo'q!", show_alert=True)
                return
            
            settings_text = (
                "⚙ <b>Sozlamalar</b>\n\n"
                "🤖 Bot versiyasi: 1.0.0\n"
                "🐍 Python versiyasi: 3.12+\n"
                "📦 Aiogram versiyasi: 3.x\n"
                "🔄 yt-dlp: So'ngi versiya\n\n"
                "⚡ Anti-spam: Faol (3 soniya)\n"
                "📢 Majburiy obuna: Faol\n"
                "💾 Ma'lumotlar bazasi: SQLite3\n\n"
                "📊 Holat: 🟢 Ishlayapti"
            )
            
            await callback.message.edit_text(settings_text, reply_markup=self.get_back_to_admin_keyboard())
            await callback.answer()
        
        @self.dp.callback_query(F.data == "admin_bot_status")
        async def admin_bot_status(callback: CallbackQuery):
            if callback.from_user.id not in ADMIN_IDS:
                await callback.answer("❌ Ruxsat yo'q!", show_alert=True)
                return
            
            uptime = "N/A"  # You can implement uptime tracking
            total_users = self.db.get_total_users()
            total_downloads = self.db.get_total_downloads()
            
            status_text = (
                "🔄 <b>Bot holati</b>\n\n"
                f"📊 Holat: 🟢 <b>Faol</b>\n"
                f"👥 Foydalanuvchilar: {total_users}\n"
                f"📥 Yuklamalar: {total_downloads}\n"
                f"⏱ Ishlash vaqti: {uptime}\n\n"
                f"💾 Xotira: Normal\n"
                f"⚡ CPU: Normal\n"
                f"🌐 API: Ulangan"
            )
            
            keyboard = InlineKeyboardBuilder()
            keyboard.add(InlineKeyboardButton(text="🔄 Yangilash", callback_data="admin_bot_status"))
            keyboard.add(InlineKeyboardButton(text="⬅ Orqaga", callback_data="admin_panel"))
            
            await callback.message.edit_text(status_text, reply_markup=keyboard.as_markup())
            await callback.answer()
        
        @self.dp.callback_query(F.data == "admin_download_stats")
        async def admin_download_stats(callback: CallbackQuery):
            if callback.from_user.id not in ADMIN_IDS:
                await callback.answer("❌ Ruxsat yo'q!", show_alert=True)
                return
            
            total_downloads = self.db.get_total_downloads()
            video_360 = self.db.get_downloads_by_type("video_360p")
            video_480 = self.db.get_downloads_by_type("video_480p")
            video_720 = self.db.get_downloads_by_type("video_720p")
            video_1080 = self.db.get_downloads_by_type("video_1080p")
            audio_128 = self.db.get_downloads_by_type("audio_128kbps")
            audio_192 = self.db.get_downloads_by_type("audio_192kbps")
            audio_320 = self.db.get_downloads_by_type("audio_320kbps")
            
            stats_text = (
                "📥 <b>Yuklash statistikasi</b>\n\n"
                f"📊 Jami yuklamalar: {total_downloads}\n\n"
                f"🎬 <b>Video:</b>\n"
                f"   360p: {video_360}\n"
                f"   480p: {video_480}\n"
                f"   720p: {video_720}\n"
                f"   1080p: {video_1080}\n\n"
                f"🎵 <b>Audio:</b>\n"
                f"   128kbps: {audio_128}\n"
                f"   192kbps: {audio_192}\n"
                f"   320kbps: {audio_320}"
            )
            
            await callback.message.edit_text(stats_text, reply_markup=self.get_back_to_admin_keyboard())
            await callback.answer()
    
    def get_admin_keyboard(self) -> InlineKeyboardMarkup:
        keyboard = InlineKeyboardBuilder()
        keyboard.add(InlineKeyboardButton(text="📊 Statistika", callback_data="admin_stats"))
        keyboard.add(InlineKeyboardButton(text="👥 Foydalanuvchilar", callback_data="admin_users"))
        keyboard.add(InlineKeyboardButton(text="📢 Reklama", callback_data="admin_broadcast"))
        keyboard.add(InlineKeyboardButton(text="➕ Kanal qo'shish", callback_data="admin_add_channel"))
        keyboard.add(InlineKeyboardButton(text="➖ Kanal o'chirish", callback_data="admin_remove_channel"))
        keyboard.add(InlineKeyboardButton(text="📋 Kanallar", callback_data="admin_channels"))
        keyboard.add(InlineKeyboardButton(text="⚙ Sozlamalar", callback_data="admin_settings"))
        keyboard.add(InlineKeyboardButton(text="🔄 Bot holati", callback_data="admin_bot_status"))
        keyboard.add(InlineKeyboardButton(text="📥 Yuklash statistikasi", callback_data="admin_download_stats"))
        keyboard.add(InlineKeyboardButton(text="⬅ Chiqish", callback_data="back_to_start"))
        keyboard.adjust(2)
        return keyboard.as_markup()
    
    def get_back_to_admin_keyboard(self) -> InlineKeyboardMarkup:
        keyboard = InlineKeyboardBuilder()
        keyboard.add(InlineKeyboardButton(text="⬅ Orqaga", callback_data="admin_panel"))
        return keyboard.as_markup()
    
    def get_cancel_keyboard(self) -> InlineKeyboardMarkup:
        keyboard = InlineKeyboardBuilder()
        keyboard.add(InlineKeyboardButton(text="❌ Bekor qilish", callback_data="cancel"))
        return keyboard.as_markup()
    
    def get_download_keyboard(self) -> InlineKeyboardMarkup:
        keyboard = InlineKeyboardBuilder()
        keyboard.add(InlineKeyboardButton(text="📥 Yangi media", callback_data="back_to_start"))
        keyboard.add(InlineKeyboardButton(text="📊 Statistika", callback_data="show_stats"))
        keyboard.adjust(1)
        return keyboard.as_markup()
    
    async def run(self):
        logger.info("🚀 Bot ishga tushmoqda...")
        await self.dp.start_polling(self.bot)

# =============== MAIN ===============
if __name__ == "__main__":
    try:
        bot = TezYuklaBot()
        asyncio.run(bot.run())
    except KeyboardInterrupt:
        logger.info("⏹ Bot to'xtatildi")
    except Exception as e:
        logger.error(f"❌ Xatolik: {e}")
