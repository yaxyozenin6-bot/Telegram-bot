import telebot
from telebot import types
import json
import os
import random

TOKEN = '8071863367:AAG5oeoSfrpGUFt7T2nzCXlVnOIAkxZWkqo'
ADMIN_ID = 7569090252

bot = telebot.TeleBot(TOKEN)
DATA_FILE = 'videos.json'

search_mode = {}
user_recommendations = {}

def load_data():
    if not os.path.exists(DATA_FILE):
        return []
    with open(DATA_FILE, 'r', encoding='utf-8') as f:
        return json.load(f)

def save_data(data):
    with open(DATA_FILE, 'w', encoding='utf-8') as f:
        json.dump(data, f, ensure_ascii=False, indent=2)

@bot.message_handler(commands=['start'])
def send_welcome(message):
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True)
    markup.add(types.KeyboardButton("🔍 Anime izlash"))
    markup.add(types.KeyboardButton("🎁 Konkurs"), types.KeyboardButton("📢 E’lonlar"))
    markup.add(types.KeyboardButton("📗 Qo‘llanma"), types.KeyboardButton("💼 Reklama va Homiylik"))
    bot.send_message(message.chat.id,
                     "Assalomu alaykum! Quyidagi bo‘limlardan birini tanlang:",
                     reply_markup=markup)

@bot.message_handler(func=lambda message: message.text == "🔍 Anime izlash")
def handle_anime(message):
    bot.send_message(message.chat.id, "Qaysi anime kerak? Ismini yozing...")
    search_mode[message.chat.id] = True

@bot.message_handler(func=lambda message: message.text == "🎁 Konkurs")
def handle_konkurs(message):
    bot.send_message(message.chat.id, "Hozircha aktiv konkurslar yo‘q.")

@bot.message_handler(func=lambda message: message.text == "📢 E’lonlar")
def handle_elon(message):
    bot.send_message(message.chat.id, "So‘nggi e’lonlar: hech narsa yo'q.")

@bot.message_handler(func=lambda message: message.text == "📗 Qo‘llanma")
def handle_qollanma(message):
    bot.send_message(message.chat.id, "Botdan foydalanish bo‘yicha qo‘llanma...")

@bot.message_handler(func=lambda message: message.text == "💼 Reklama va Homiylik")
def handle_reklama(message):
    bot.send_message(message.chat.id, "Reklama uchun @psixo_666 bilan bog‘laning.")

@bot.message_handler(commands=['anime'])
def command_anime(message):
    text = message.text.split(maxsplit=1)
    if len(text) == 1:
        bot.reply_to(message, "Iltimos, anime nomini yozing. Masalan:\n/anime Naruto")
        return
    anime_name = text[1]
    bot.reply_to(message, f"🔍 '{anime_name}' bo‘yicha qidiryapman...")

@bot.message_handler(commands=['hello'])
def command_hello(message):
    bot.reply_to(message, "Salom! Yordam kerak bo‘lsa, /start buyrug‘ini bosing 😊")

@bot.message_handler(content_types=['video'])
def handle_video(message):
    if message.from_user.id != ADMIN_ID:
        bot.reply_to(message, "❌ Sizda video yuklash uchun ruxsat yo‘q.")
        return
    bot.send_message(message.chat.id, "📌 Videoga nom kiriting:")
    bot.register_next_step_handler(message, lambda msg: save_video_info(msg, message.video.file_id))

def save_video_info(message, file_id):
    title = message.text.strip().lower()
    data = load_data()

    # Takror nomni tekshirish
    if any(item['title'] == title for item in data):
        bot.send_message(message.chat.id, f"⚠️ '{title}' nomli video allaqachon saqlangan.")
        return

    data.append({
        "title": title,
        "file_id": file_id
    })
    save_data(data)
    bot.send_message(message.chat.id, f"✅ '{title}' nomli video saqlandi!")

@bot.message_handler(func=lambda message: True)
def handle_all_messages(message):
    if search_mode.get(message.chat.id):
        keyword = message.text.lower()
        data = load_data()
        results = [item for item in data if keyword in item['title']]
        if not results:
            if data:
                recommended = random.sample(data, min(3, len(data)))
                user_recommendations[message.chat.id] = recommended

                markup = types.InlineKeyboardMarkup()
                for idx, item in enumerate(recommended):
                    markup.add(types.InlineKeyboardButton(
                        text=item['title'].capitalize(),
                        callback_data=f"video_{idx}"
                    ))

                bot.send_message(
                    message.chat.id,
                    "❌ Video topilmadi. Quyidagilardan tanlang:",
                    reply_markup=markup
                )
            else:
                bot.send_message(message.chat.id, "❌ Hech qanday video topilmadi.")
        else:
            item = results[0]
            bot.send_video(message.chat.id, item['file_id'], caption=item['title'])

        search_mode[message.chat.id] = False
    else:
        markup = types.ReplyKeyboardMarkup(resize_keyboard=True)
        markup.add(types.KeyboardButton("🔍 Anime izlash"))
        markup.add(types.KeyboardButton("🎁 Konkurs"), types.KeyboardButton("📢 E’lonlar"))
        markup.add(types.KeyboardButton("📗 Qo‘llanma"), types.KeyboardButton("💼 Reklama va Homiylik"))
        bot.send_message(
            message.chat.id,
            "❗ Noto‘g‘ri buyruq yoki xabar. Iltimos, quyidagi menyudan tanlang:",
            reply_markup=markup
        )

@bot.callback_query_handler(func=lambda call: call.data.startswith("video_"))
def callback_video(call):
    try:
        idx = int(call.data.split('_')[1])
        video_item = user_recommendations.get(call.message.chat.id, [])[idx]
        bot.send_video(call.message.chat.id, video_item['file_id'], caption=video_item['title'])
        bot.edit_message_reply_markup(call.message.chat.id, call.message.message_id, reply_markup=None)
        user_recommendations.pop(call.message.chat.id, None)
        bot.answer_callback_query(call.id, text=f"'{video_item['title'].capitalize()}' video tanlandi.")
    except (IndexError, ValueError):
        bot.answer_callback_query(call.id, "Xato: video topilmadi yoki noto'g'ri tugma.", show_alert=True)

# Botni ishga tushiramiz
bot.polling(none_stop=True)
