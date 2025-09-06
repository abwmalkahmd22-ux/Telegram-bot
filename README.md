# Telegram-bot
import sqlite3
from telegram import InlineKeyboardButton, InlineKeyboardMarkup, Update
from telegram.ext import Updater, CommandHandler, CallbackQueryHandler, CallbackContext
import logging

# 🔹 Bot ayarları
BOT_TOKEN = "7626042752:AAE1b1G1nSnV5BOa1G_DUSVqFX2GEcCi3xo"
BOT_USERNAME = "@Canrefansbot"  # örn: mybot

# 🔹 Logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# 📌 Database kurulum
def db_connect():
    conn = sqlite3.connect("users.db")
    c = conn.cursor()
    c.execute("""CREATE TABLE IF NOT EXISTS users (
        user_id INTEGER PRIMARY KEY,
        username TEXT,
        jeton INTEGER DEFAULT 0,
        ref INTEGER DEFAULT 0
    )""")
    conn.commit()
    return conn

# 📌 Kullanıcı ekle
def add_user(user_id, username):
    conn = db_connect()
    c = conn.cursor()
    c.execute("INSERT OR IGNORE INTO users (user_id, username, jeton, ref) VALUES (?, ?, 0, 0)", (user_id, username))
    conn.commit()
    conn.close()

# 📌 Jeton/ref güncelle
def update_ref(ref_id, username):
    conn = db_connect()
    c = conn.cursor()
    c.execute("UPDATE users SET jeton = jeton + 1, ref = ref + 1 WHERE user_id = ?", (ref_id,))
    conn.commit()
    conn.close()

# 📌 Kullanıcı bilgisi
def get_user(user_id):
    conn = db_connect()
    c = conn.cursor()
    c.execute("SELECT jeton, ref FROM users WHERE user_id = ?", (user_id,))
    result = c.fetchone()
    conn.close()
    return result if result else (0, 0)

# 📌 Liderlik listesi
def get_leaderboard():
    conn = db_connect()
    c = conn.cursor()
    c.execute("SELECT username, jeton, ref FROM users ORDER BY jeton DESC LIMIT 10")
    result = c.fetchall()
    conn.close()
    return result

# 📌 Start
def start(update: Update, context: CallbackContext):
    user = update.effective_user
    user_id = user.id
    username = user.username or f"id{user_id}"

    add_user(user_id, username)

    # Referans kontrol
    if context.args:
        try:
            ref_id = int(context.args[0])
            if ref_id != user_id:
                update_ref(ref_id, username)
                context.bot.send_message(
                    chat_id=ref_id,
                    text=f"🎉 Yeni referans! @{username} katıldı. +1 jeton 💎"
                )
        except ValueError:
            pass

    jeton, ref = get_user(user_id)

    # Menü
    keyboard = [
        [InlineKeyboardButton("👥 Referans Linkim", callback_data="referans")],
        [InlineKeyboardButton("🏆 Liderlik", callback_data="liderlik")],
        [InlineKeyboardButton("📩 Destek", callback_data="destek")]
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)

    update.message.reply_text(
        f"🤖 Merhaba @{username}!\n"
        f"Şu an 💎 {jeton} jetonun var.\n"
        f"Toplam 👥 {ref} referansın var.\n"
        f"Arkadaşlarını davet ederek jeton kazanabilirsin!",
        reply_markup=reply_markup
    )

# 📌 Butonlar
def button_handler(update: Update, context: CallbackContext):
    query = update.callback_query
    query.answer()

    user_id = query.from_user.id
    username = query.from_user.username or f"id{user_id}"

    if query.data == "destek":
        query.message.reply_text("📩 Destek için: @fethetimm")

    elif query.data == "liderlik":
        board = get_leaderboard()
        if not board:
            query.message.reply_text("🏆 Henüz kimse yok liderlikte.")
        else:
            text = "🏆 Liderlik Tablosu 🏆\n\n"
            for idx, (uname, jeton, ref) in enumerate(board, start=1):
                text += f"{idx}. @{uname} — 💎 {jeton} Jeton | 👥 {ref} Ref\n"
            query.message.reply_text(text)

    elif query.data == "referans":
        ref_link = f"https://t.me/{BOT_USERNAME}?start={user_id}"
        jeton, ref = get_user(user_id)
        query.message.reply_text(
            f"🔗 Referans linkin:\n{ref_link}\n\n"
            f"Şu an 💎 {jeton} jetonun, 👥 {ref} referansın var."
        )

# 📌 Hata
def error(update, context):
    logger.warning(f"Hata: {context.error}")

# 📌 Main
def main():
    updater = Updater(BOT_TOKEN)
    dp = updater.dispatcher

    dp.add_handler(CommandHandler("start", start))
    dp.add_handler(CallbackQueryHandler(button_handler))
    dp.add_error_handler(error)

    print("🤖 Bot çalışıyor...")
    updater.start_polling()
    updater.idle()

if __name__ == "__main__":
    main()
