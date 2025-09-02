# video-downloader-bot
import os
import yt_dlp
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import ApplicationBuilder, CommandHandler, MessageHandler, CallbackQueryHandler, ContextTypes, filters

API_TOKEN = "8323949995:AAF7dIxGpD01MAALToG0XSe36lfBUs_fFbE"

# Стартовое сообщение
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "👋 Привет! Я бот для скачивания видео с YouTube.\n"
        "Отправь мне ссылку на видео, и я предложу выбрать качество."
    )

# Обработка ссылки
async def handle_link(update: Update, context: ContextTypes.DEFAULT_TYPE):
    url = update.message.text.strip()
    if not url.startswith("http"):
        await update.message.reply_text("❌ Пожалуйста, отправь корректную ссылку.")
        return

    await update.message.reply_text("⏳ Получаю информацию о видео...")

    try:
        ydl_opts = {"quiet": True}
        with yt_dlp.YoutubeDL(ydl_opts) as ydl:
            info = ydl.extract_info(url, download=False)
            formats = info.get("formats", [])

        # Фильтруем форматы до 2 ГБ
        available_formats = [
            (f"{f['format_note']} ({round(f['filesize'] / (1024*1024), 2)} MB)" if f.get('filesize') else f['format_note'], f['format_id'])
            for f in formats if f.get("filesize") and f["filesize"] <= 2 * 1024 * 1024 * 1024
        ]

        if not available_formats:
            await update.message.reply_text("⚠ Нет доступных форматов меньше 2 ГБ.")
            return

        # Кнопки выбора качества
        keyboard = [[InlineKeyboardButton(text, callback_data=f"{url}|{format_id}")]
                    for text, format_id in available_formats[:10]]  # первые 10 форматов
        reply_markup = InlineKeyboardMarkup(keyboard)

        await update.message.reply_text("Выбери качество:", reply_markup=reply_markup)

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка: {e}")

# Обработка выбора качества
async def button(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()

    url, format_id = query.data.split("|")
    await query.edit_message_text("⏳ Скачиваю видео, подожди...")

    try:
        ydl_opts = {
            "format": format_id,
            "outtmpl": "%(title)s.%(ext)s",
        }
        with yt_dlp.YoutubeDL(ydl_opts) as ydl:
            info = ydl.extract_info(url, download=True)
            file_path = ydl.prepare_filename(info)

        # Отправка файла
        await query.message.reply_document(open(file_path, "rb"), caption="✅ Вот твое видео!")

    except Exception as e:
        await query.message.reply_text(f"❌ Ошибка при загрузке: {e}")

# Основной блок
app = ApplicationBuilder().token(API_TOKEN).build()
app.add_handler(CommandHandler("start", start))
app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_link))
app.add_handler(CallbackQueryHandler(button))

print("Бот запущен...")
app.run_polling()
