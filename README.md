# requirements.txt
python-telegram-bot==20.0 openai==0.27.0 python-pptx==0.6.21
import openai
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Updater, CommandHandler, MessageHandler, Filters, CallbackQueryHandler, CallbackContext
from pptx import Presentation
from io import BytesIO

# OpenAI API kaliti
openai.api_key = "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"

# Telegram bot tokeni
TOKEN = '7577376285:AAFKoWUtbl0GZqkTo5CE8yz_b3B-uqZZeOw'

# Kanal ID-si (Foydalanuvchilar shundan kanalga obuna bo'lishlari kerak)
CHANNEL_ID = "@YourChannelUsername"  # Kanalni username (masalan: @my_channel)

# Admin ID: adminlarning Telegram ID sini yozing
ADMIN_IDS = [123456789]  # Bu yerga adminlar Telegram ID larini qo'shing

# Tilni saqlash uchun o'zgaruvchi
user_language = {}

# Foydalanuvchilarni ro'yxatga olish
users = set()

# OpenAI bilan savol-javob funksiyasi
def get_openai_response(query: str, lang: str) -> str:
    response = openai.Completion.create(
        engine="text-davinci-003",  # Yoki yangilangan modelni tanlang
        prompt=query,
        max_tokens=150
    )
    return response.choices[0].text.strip()

# Slayd yaratish funksiyasi
def create_slide(text: str) -> BytesIO:
    presentation = Presentation()
    slide_layout = presentation.slide_layouts[1]  # Title and Content layout
    slide = presentation.slides.add_slide(slide_layout)
    
    title = slide.shapes.title
    content = slide.shapes.placeholders[1]
    
    title.text = "Siz yuborgan matn"
    content.text = text
    
    pptx_stream = BytesIO()
    presentation.save(pptx_stream)
    pptx_stream.seek(0)
    return pptx_stream

# Tilni tanlash tugmalari
def language_buttons(update: Update) -> None:
    keyboard = [
        [InlineKeyboardButton("O'zbek", callback_data='uz')],
        [InlineKeyboardButton("Русский", callback_data='ru')],
        [InlineKeyboardButton("English", callback_data='en')]
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)
    update.message.reply_text('Tilni tanlang:', reply_markup=reply_markup)

# Tilni sozlash
def set_language(update: Update, context: CallbackContext) -> None:
    query = update.callback_query
    language = query.data
    user_language[query.from_user.id] = language  # Foydalanuvchi tilini saqlash
    query.answer()
    query.edit_message_text(f"Sizning tilingiz {language} bo'ldi.")

# Kanalga obuna bo'lishni tekshirish
def check_subscription(update: Update, context: CallbackContext) -> bool:
    user_id = update.message.from_user.id
    try:
        # Kanalga obuna bo'lganligini tekshirish
        member = context.bot.get_chat_member(CHANNEL_ID, user_id)
        if member.status in ['member', 'administrator', 'creator']:
            return True
        else:
            update.message.reply_text("Botdan foydalanish uchun kanalga obuna bo'lishingiz kerak.")
            return False
    except:
        update.message.reply_text("Botdan foydalanish uchun kanalga obuna bo'lishingiz kerak.")
        return False

# Kanal havolasini o'zgartirish
def change_channel_link(update: Update, context: CallbackContext) -> None:
    if update.message.from_user.id in ADMIN_IDS:
        if context.args:
            global CHANNEL_ID
            CHANNEL_ID = context.args[0]
            update.message.reply_text(f"Kanal havolasi muvaffaqiyatli o'zgartirildi: {CHANNEL_ID}")
        else:
            update.message.reply_text("Iltimos, yangi kanal havolasini kiriting.")
    else:
        update.message.reply_text("Siz admin emassiz!")

# Kanal qo'shish
def add_new_channel(update: Update, context: CallbackContext) -> None:
    if update.message.from_user.id in ADMIN_IDS:
        if context.args:
            global CHANNEL_ID
            CHANNEL_ID = context.args[0]
            update.message.reply_text(f"Yangi kanal qo'shildi: {CHANNEL_ID}")
        else:
            update.message.reply_text("Iltimos, yangi kanal havolasini kiriting.")
    else:
        update.message.reply_text("Siz admin emassiz!")

# Start komanda
def start(update: Update, context: CallbackContext) -> None:
    if not check_subscription(update, context):
        return  # Kanalga obuna bo'lmagan foydalanuvchi uchun bot ishlamaydi
    users.add(update.message.from_user.id)  # Foydalanuvchini ro'yxatga olish
    update.message.reply_text('Salom! Iltimos, tilni tanlang.', reply_markup=InlineKeyboardMarkup([[
        InlineKeyboardButton("Tilni tanlang", callback_data="choose_language")
    ]]))

# Help komanda
def help_command(update: Update, context: CallbackContext) -> None:
    help_text = (
        "/start - Botni ishga tushirish\n"
        "/help - Yordam\n"
        "/slayd - Slayd yaratish"
    )
    update.message.reply_text(help_text)

# OpenAI javobi
def answer(update: Update, context: CallbackContext) -> None:
    if not check_subscription(update, context):
        return  # Kanalga obuna bo'lmagan foydalanuvchi uchun bot ishlamaydi
    
    user_input = update.message.text
    lang = user_language.get(update.message.from_user.id, 'uz')  # Foydalanuvchining tilini olish

    response = get_openai_response(user_input, lang)
    if lang == 'uz':
        update.message.reply_text(response)
    elif lang == 'ru':
        update.message.reply_text(response)  # Rus tilida javob
    else:
        update.message.reply_text(response)  # Ingliz tilida javob

# Slayd yuborish
def send_slide(update: Update, context: CallbackContext) -> None:
    if not check_subscription(update, context):
        return  # Kanalga obuna bo'lmagan foydalanuvchi uchun bot ishlamaydi
    
    user_text = update.message.text
    slide_stream = create_slide(user_text)
    update.message.reply_document(document=slide_stream, filename="slide.pptx")

# Admin paneli - Foydalanuvchilarni ko'rish
def admin_users(update: Update, context: CallbackContext) -> None:
    if update.message.from_user.id in ADMIN_IDS:
        users_count = len(users)
        update.message.reply_text(f"Jami foydalanuvchilar soni: {users_count}")
    else:
        update.message.reply_text("Siz admin emassiz!")

# Admin paneli - Umumiy xabar yuborish
def send_broadcast(update: Update, context: CallbackContext) -> None:
    if update.message.from_user.id in ADMIN_IDS:
        message = " ".join(context.args)
        for user_id in users:
            try:
                context.bot.send_message(user_id, message)
            except:
                pass  # Foydalanuvchi bloklagan bo'lsa yoki boshqa xatolik yuz bersa

        update.message.reply_text("Umumiy xabar yuborildi!")
    else:
        update.message.reply_text("Siz admin emassiz!")

# Rasmni saqlash
def save_image(update: Update, context: CallbackContext) -> None:
    photo = update.message.photo[-1]
    file = photo.get_file()
    file.download('image.jpg')
    update.message.reply_text("Rasm saqlandi!")

# Botni ishga tushirish
def main():
    updater = Updater(TOKEN)
    dispatcher = updater.dispatcher

    # Start komandasini ishga tushirish
    dispatcher.add_handler(CommandHandler("start", start))
    dispatcher.add_handler(CommandHandler("help", help_command))
    
    # Slayd yaratish
    dispatcher.add_handler(CommandHandler("slayd", send_slide))
    
    # Admin paneli
    dispatcher.add_handler(CommandHandler("admin_users", admin_users))
    dispatcher.add_handler(CommandHandler("send_broadcast", send_broadcast, pass_args=True))

    # Kanalni qo'shish yoki o'zgartirish
    dispatcher.add_handler(CommandHandler("change_channel", change_channel_link, pass_args=True))
    dispatcher.add_handler(CommandHandler("add_channel", add_new_channel, pass_args=True))

    # Tilni tanlash
    dispatcher.add_handler(CallbackQueryHandler(set_language, pattern='^(uz|ru|en)$'))

    # Foydalanuvchi yuborgan xabarlarni qayta ishlash
    dispatcher.add_handler(MessageHandler(Filters.text & ~Filters.command, answer))

    # Rasmni saqlash
    dispatcher.add_handler(MessageHandler(Filters.photo, save_image))

    # Botni ishga tushirish
    updater.start_polling()
    updater.idle()

if __name__ == '__main__':
    main()
    
