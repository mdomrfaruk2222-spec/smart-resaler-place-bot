import os
import sqlite3
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    Application,
    CommandHandler,
    CallbackQueryHandler,
    MessageHandler,
    ContextTypes,
    filters,
)

BOT_USERNAME = "smartresalerplacebot"
DB_NAME = "resalers.db"


# ---------------- DATABASE ----------------

def get_db():
    return sqlite3.connect(DB_NAME)


def setup_database():
    db = get_db()
    cursor = db.cursor()

    cursor.execute("""
        CREATE TABLE IF NOT EXISTS users (
            telegram_id INTEGER PRIMARY KEY,
            name TEXT,
            phone TEXT,
            reseller_id TEXT UNIQUE,
            referral_code TEXT UNIQUE,
            parent_id INTEGER,
            balance REAL DEFAULT 0
        )
    """)

    db.commit()
    db.close()


def get_user(telegram_id):
    db = get_db()
    cursor = db.cursor()

    cursor.execute(
        "SELECT * FROM users WHERE telegram_id = ?",
        (telegram_id,)
    )

    user = cursor.fetchone()
    db.close()

    return user


def create_user(telegram_id, name, phone, parent_id=None):
    db = get_db()
    cursor = db.cursor()

    cursor.execute("SELECT COUNT(*) FROM users")
    count = cursor.fetchone()[0] + 1

    reseller_id = f"SRP-{count:06d}"
    referral_code = f"SRP{count:06d}"

    cursor.execute("""
        INSERT INTO users
        (telegram_id, name, phone, reseller_id, referral_code, parent_id)
        VALUES (?, ?, ?, ?, ?, ?)
    """, (
        telegram_id,
        name,
        phone,
        reseller_id,
        referral_code,
        parent_id
    ))

    db.commit()
    db.close()

    return reseller_id, referral_code


# ---------------- START ----------------

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):

    telegram_id = update.effective_user.id
    user = get_user(telegram_id)

    if user:
        await show_main_menu(update)
        return

    parent_id = None

    if context.args:
        referral_code = context.args[0]

        db = get_db()
        cursor = db.cursor()

        cursor.execute(
            "SELECT telegram_id FROM users WHERE referral_code = ?",
            (referral_code,)
        )

        result = cursor.fetchone()

        if result:
            parent_id = result[0]

        db.close()

    keyboard = [
        [
            InlineKeyboardButton(
                "👤 Register",
                callback_data=f"register:{parent_id or 0}"
            )
        ]
    ]

    await update.message.reply_text(
        "🛍️ Smart Resaler Place-এ স্বাগতম!\n\n"
        "এখানে আপনি বিভিন্ন পণ্য নিয়ে নিজের ব্যবসা করতে পারবেন।\n\n"
        "👇 শুরু করতে Register চাপুন।",
        reply_markup=InlineKeyboardMarkup(keyboard)
    )


# ---------------- MAIN MENU ----------------

async def show_main_menu(update: Update):

    keyboard = [
        [
            InlineKeyboardButton("🛍️ Products", callback_data="products"),
            InlineKeyboardButton("📦 My Orders", callback_data="orders"),
        ],
        [
            InlineKeyboardButton("💰 My Earnings", callback_data="earnings"),
            InlineKeyboardButton("👥 My Network", callback_data="network"),
        ],
        [
            InlineKeyboardButton("🔗 My Referral", callback_data="referral"),
            InlineKeyboardButton("💳 Withdraw", callback_data="withdraw"),
        ],
        [
            InlineKeyboardButton("👤 My Profile", callback_data="profile"),
            InlineKeyboardButton("🆘 Support", callback_data="support"),
        ],
    ]

    text = (
        "🏠 Smart Resaler Place\n\n"
        "আপনার Main Menu-তে স্বাগতম।\n\n"
        "নিচের মেনু থেকে একটি অপশন নির্বাচন করুন:"
    )

    if update.callback_query:
        await update.callback_query.edit_message_text(
            text,
            reply_markup=InlineKeyboardMarkup(keyboard)
        )
    else:
        await update.message.reply_text(
            text,
            reply_markup=InlineKeyboardMarkup(keyboard)
        )


# ---------------- REGISTER ----------------

async def register_start(update: Update, context: ContextTypes.DEFAULT_TYPE):

    query = update.callback_query
    await query.answer()

    parent_id = int(query.data.split(":")[1])

    context.user_data["parent_id"] = parent_id if parent_id != 0 else None
    context.user_data["register_step"] = "name"

    await query.edit_message_text(
        "👤 Registration শুরু হচ্ছে।\n\n"
        "আপনার নাম লিখুন:"
    )


async def registration_message(update: Update, context: ContextTypes.DEFAULT_TYPE):

    step = context.user_data.get("register_step")

    if not step:
        return

    if step == "name":

        context.user_data["name"] = update.message.text
        context.user_data["register_step"] = "phone"

        await update.message.reply_text(
            "📱 এখন আপনার মোবাইল নম্বর লিখুন:"
        )

        return

    if step == "phone":

        name = context.user_data["name"]
        phone = update.message.text
        parent_id = context.user_data.get("parent_id")

        telegram_id = update.effective_user.id

        try:

            reseller_id, referral_code = create_user(
                telegram_id,
                name,
                phone,
                parent_id
            )

            context.user_data.clear()

            await update.message.reply_text(
                "🎉 Registration সফল হয়েছে!\n\n"
                f"👤 নাম: {name}\n"
                f"🆔 Reseller ID: {reseller_id}\n"
                f"🔑 Reseller Code: {referral_code}\n\n"
                f"🔗 আপনার Referral Link:\n"
                f"https://t.me/{BOT_USERNAME}?start={referral_code}\n\n"
                "এখন আপনি Smart Resaler Place ব্যবহার করতে পারবেন।"
            )

            await show_main_menu(update)

        except sqlite3.IntegrityError:

            await update.message.reply_text(
                "⚠️ এই Telegram account দিয়ে ইতিমধ্যে registration করা হয়েছে।"
            )

            context.user_data.clear()


# ---------------- BUTTON HANDLER ----------------

async def button_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):

    query = update.callback_query
    await query.answer()

    data = query.data

    if data.startswith("register:"):
        await register_start(update, context)
        return

    telegram_id = query.from_user.id
    user = get_user(telegram_id)

    if not user:

        await query.edit_message_text(
            "⚠️ আগে Registration করুন।\n\n"
            "Telegram-এ /start লিখে আবার শুরু করুন."
        )

        return

    if data == "profile":

        await query.edit_message_text(
            "👤 My Profile\n\n"
            f"নাম: {user[1]}\n"
            f"মোবাইল: {user[2]}\n"
            f"Reseller ID: {user[3]}\n"
            f"Reseller Code: {user[4]}\n"
            f"Balance: ৳{user[6]:.2f}"
        )

    elif data == "referral":

        await query.edit_message_text(
            "🔗 My Referral\n\n"
            f"আপনার Referral Code:\n{user[4]}\n\n"
            "আপনার Referral Link:\n"
            f"https://t.me/{BOT_USERNAME}?start={user[4]}"
        )

    elif data == "network":

        db = get_db()
        cursor = db.cursor()

        cursor.execute(
            "SELECT COUNT(*) FROM users WHERE parent_id = ?",
            (telegram_id,)
        )

        direct = cursor.fetchone()[0]

        db.close()

        await query.edit_message_text(
            "👥 My Network\n\n"
            f"Generation 1 Members: {direct}\n"
            "Generation 2: শীঘ্রই যোগ হবে\n"
            "Generation 3: শীঘ্রই যোগ হবে\n"
            "Generation 4: শীঘ্রই যোগ হবে"
        )

    elif data == "earnings":

        await query.edit_message_text(
            "💰 My Earnings\n\n"
            f"বর্তমান Balance: ৳{user[6]:.2f}\n\n"
            "Delivered order অনুযায়ী commission এখানে যোগ হবে।"
        )

    elif data == "products":

        await query.edit_message_text(
            "🛍️ Products\n\n"
            "Product system পরের ধাপে যোগ করা হবে।"
        )

    elif data == "orders":

        await query.edit_message_text(
            "📦 My Orders\n\n"
            "Order system পরের ধাপে যোগ করা হবে।"
        )

    elif data == "withdraw":

        await query.edit_message_text(
            "💳 Withdraw\n\n"
            "Withdrawal system পরের ধাপে যোগ করা হবে।"
        )

    elif data == "support":

        await query.edit_message_text(
            "🆘 Support\n\n"
            "Support system পরের ধাপে যোগ করা হবে।"
        )


# ---------------- MAIN ----------------

def main():

    setup_database()

    token = os.getenv("BOT_TOKEN")

    if not token:
        print("ERROR: BOT_TOKEN environment variable is missing.")
        return

    application = Application.builder().token(token).build()

    application.add_handler(CommandHandler("start", start))
    application.add_handler(CallbackQueryHandler(button_handler))
    application.add_handler(
        MessageHandler(
            filters.TEXT & ~filters.COMMAND,
            registration_message
        )
    )

    print("Smart Resaler Place Bot is running...")

    application.run_polling()


if __name__ == "__main__":
    main()
