# TajikAccounts.bot4
git add bot.py
     git commit -m "import telebot
import json
import os
import time
from telebot.types import ReplyKeyboardMarkup, KeyboardButton, InlineKeyboardMarkup, InlineKeyboardButton

TOKEN = "7542895049:AAGQIquLCB9l2kbcZJsXQ9x9CcAKS3yRwxE"
bot = telebot.TeleBot(TOKEN)

# Файли хотира
DATA_FILE = "bot_data.json"

# Боркунии маълумот
def load_data():
    if os.path.exists(DATA_FILE):
        with open(DATA_FILE, "r") as file:
            data = json.load(file)
            for account in data.get("accounts", []):
                if "timestamp" not in account:
                    account["timestamp"] = time.time()
            return data
    return {"accounts": []}

# Захираи маълумот
def save_data(data):
    with open(DATA_FILE, "w") as file:
        json.dump(data, file, indent=4)

# Нест кардани аккаунтҳои кӯҳна (пас аз 72 соат)
def clear_old_accounts():
    global data
    current_time = time.time()
    data["accounts"] = [
        acc for acc in data["accounts"] if current_time - acc["timestamp"] < 3 * 24 * 60 * 60
    ]
    save_data(data)

# Ибтидоӣ боркунии маълумот
data = load_data()

# Менюи асосӣ
def main_menu():
    markup = ReplyKeyboardMarkup(resize_keyboard=True)
    markup.add(KeyboardButton("💵 ХАРИДОРИ АККАУНТ"), KeyboardButton("💰 ФРУШИ АККАУНТҲО"))
    return markup

# /start
@bot.message_handler(commands=['start'])
def send_welcome(message):
    clear_old_accounts()
    bot.send_message(message.chat.id, "Салом! Хуш омадед ба боти FAYZ. Як вариантро интихоб кунед:", reply_markup=main_menu())

# Фурӯши аккаунтҳо
@bot.message_handler(func=lambda message: message.text == "💰 ФРУШИ АККАУНТҲО")
def sell_account(message):
    chat_id = str(message.chat.id)
    data[chat_id] = {}

    markup = ReplyKeyboardMarkup(resize_keyboard=True)
    games = ["Free Fire", "PUBG", "Clash of Clans", "Instagram", "TikTok"]
    for game in games:
        markup.add(KeyboardButton(game))
    markup.add(KeyboardButton("🔙 БАРГАШТ"))

    bot.send_message(chat_id, "Лутфан бозии марбутро интихоб кунед:", reply_markup=markup)
    data[chat_id] = {"step": "choose_game"}
    save_data(data)

@bot.message_handler(func=lambda message: message.text in ["Free Fire", "PUBG", "Clash of Clans", "Instagram", "TikTok"])
def receive_game(message):
    chat_id = str(message.chat.id)
    data[chat_id] = {
        "game": message.text,
        "step": "waiting_for_images",
        "images": [],
        "timestamp": time.time()
    }
    save_data(data)
    bot.send_message(chat_id, "Лутфан якчанд акси аккаунти худро боргузорӣ кунед. Баъди тамом, '✅ Тамом' -ро пахш кунед.", reply_markup=finish_button())

# Кабули аксҳо
@bot.message_handler(content_types=['photo'])
def receive_images(message):
    chat_id = str(message.chat.id)
    if chat_id in data and data[chat_id]["step"] == "waiting_for_images":
        if "images" not in data[chat_id]:
            data[chat_id]["images"] = []
        data[chat_id]["images"].append(message.photo[-1].file_id)
        save_data(data)
        bot.send_message(chat_id, "Акс қабул шуд. Агар тамом кардед, '✅ Тамом' -ро пахш кунед.")

# Тугмаи '✅ Тамом'
def finish_button():
    markup = ReplyKeyboardMarkup(resize_keyboard=True)
    markup.add(KeyboardButton("✅ Тамом"))
    return markup

# Қабули '✅ Тамом'
@bot.message_handler(func=lambda message: message.text == "✅ Тамом")
def ask_bind_method(message):
    chat_id = str(message.chat.id)
    data[chat_id]["step"] = "waiting_for_bind"
    save_data(data)
    markup = ReplyKeyboardMarkup(resize_keyboard=True)
    markup.add(KeyboardButton("Google"), KeyboardButton("VKontakte"), KeyboardButton("Facebook"), KeyboardButton("Twitter"))
    bot.send_message(chat_id, "Лутфан аккаунти худро бо кадом пайвастшавӣ истифода мебаред?", reply_markup=markup)

# Қабули нарх
@bot.message_handler(func=lambda message: data.get(str(message.chat.id), {}).get("step") == "waiting_for_bind")
def ask_price(message):
    chat_id = str(message.chat.id)
    data[chat_id]["bind"] = message.text
    data[chat_id]["step"] = "waiting_for_price"
    save_data(data)
    bot.send_message(chat_id, "Лутфан нархи аккаунти худро ворид кунед (фақат рақам).")

# Қабули рақами ватсап
@bot.message_handler(func=lambda message: data.get(str(message.chat.id), {}).get("step") == "waiting_for_price")
def ask_whatsapp(message):
    chat_id = str(message.chat.id)
    if not message.text.isdigit():
        bot.send_message(chat_id, "❌ Лутфан фақат рақам ворид кунед.")
        return
    data[chat_id]["price"] = int(message.text)
    data[chat_id]["step"] = "waiting_for_whatsapp"
    save_data(data)
    bot.send_message(chat_id, "Лутфан рақами ватсапи худро ворид кунед (+992XXXXXXXXX).")

# Қабули гарант
@bot.message_handler(func=lambda message: data.get(str(message.chat.id), {}).get("step") == "waiting_for_whatsapp")
def ask_guarantor(message):
    chat_id = str(message.chat.id)
    if not message.text.startswith("+992") or not message.text[1:].isdigit():
        bot.send_message(chat_id, "❌ Лутфан рақами WhatsApp-ро дуруст ворид кунед (+992XXXXXXXXX).")
        return
    data[chat_id]["whatsapp"] = message.text
    data[chat_id]["step"] = "waiting_for_guarantor"
    save_data(data)

    text = ("✅ Ҳоло як гарантро интихоб кунед:\n\n"
            "📌 **Барои санҷиши айтиҳо:** Instagram - [ff.fayz_tjk](https://www.instagram.com/ff.fayz_tjk)\n\n"
            "1️⃣ *GARANT* - +992111126306\n"
            "2️⃣ *GARANT* - +992009445264")

    markup = ReplyKeyboardMarkup(resize_keyboard=True)
    markup.add(KeyboardButton("GARANT 1 (+992111126306)"))
    markup.add(KeyboardButton("GARANT 2 (+992009445264)"))

    bot.send_message(chat_id, text, reply_markup=markup, parse_mode="Markdown")

# Сабти аккаунт
@bot.message_handler(func=lambda message: data.get(str(message.chat.id), {}).get("step") == "waiting_for_guarantor")
def save_account(message):
    chat_id = str(message.chat.id)
    if message.text.startswith("GARANT"):
        data[chat_id]["guarantor"] = message.text
        data["accounts"].insert(0, data[chat_id])
        del data[chat_id]
        save_data(data)
        bot.send_message(chat_id, "✅ Аккаунти шумо бо гарант ба фурӯш гузошта шуд!", reply_markup=main_menu())
    else:
        bot.send_message(chat_id, "❌ Лутфан яке аз гарантҳои пешниҳодшударо интихоб кунед.")

# Бахши харид
@bot.message_handler(func=lambda message: message.text == "💵 ХАРИДОРИ АККАУНТ")
def show_accounts(message):
    chat_id = str(message.chat.id)
    if not data["accounts"]:
        bot.send_message(chat_id, "Ҳоло аккаунтҳо барои фурӯш вуҷуд надоранд.")
        return
    for acc in data["accounts"]:
        if "images" in acc and acc["images"]:
            caption = f"🎮 Бозӣ: {acc['game']}\n🔗 Пайвастшавӣ: {acc['bind']}\n💰 Нарх: {acc['price']}\n📞 WhatsApp: {acc['whatsapp']}\n🔰 ГАРАНТ: {acc['guarantor']}"
            markup = InlineKeyboardMarkup().add(InlineKeyboardButton("📩 Харидан", url=f"https://wa.me/{acc['whatsapp']}"))
            bot.send_photo(chat_id, acc["images"][0], caption=caption, reply_markup=markup)

# Ботро фаъол нигоҳ доштан
bot.polling(none_stop=True)"
     git push -u origin main
