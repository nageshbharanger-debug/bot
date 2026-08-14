import os
import re
import logging
import asyncio
import aiosqlite
from urllib.parse import urlparse, parse_qs, urlencode, urlunparse
from dotenv import load_dotenv
import httpx
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    Application,
    CommandHandler,
    MessageHandler,
    CallbackQueryHandler,
    ContextTypes,
    filters,
)

# Load Environment Variables
load_dotenv()
BOT_TOKEN = os.getenv("BOT_TOKEN")
DEFAULT_AMAZON_TAG = os.getenv("AMAZON_TAG", "demo-21")
DEFAULT_FLIPKART_TAG = os.getenv("FLIPKART_TAG", "demo")
OWNER_ID = int(os.getenv("OWNER_ID", 0))
DB_FILE = "bot_database.db"

# Logging Setup
logging.basicConfig(
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
    level=logging.INFO
)
logger = logging.getLogger(__name__)

# URL Cache to speed up repeat links
URL_CACHE = {}

# --- DATABASE ENGINE (SQLite) ---

async def init_db():
    """Creates database tables on startup"""
    async with aiosqlite.connect(DB_FILE) as db:
        await db.execute("""
            CREATE TABLE IF NOT EXISTS users (
                user_id INTEGER PRIMARY KEY
            )
        """)
        await db.execute("""
            CREATE TABLE IF NOT EXISTS settings (
                key TEXT PRIMARY KEY,
                value TEXT
            )
        """)
        await db.execute("""
            CREATE TABLE IF NOT EXISTS stats (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                messages_processed INTEGER DEFAULT 0,
                links_converted INTEGER DEFAULT 0
            )
        """)
        
        # Init stats row if not exists
        async with db.execute("SELECT COUNT(*) FROM stats") as cursor:
            if (await cursor.fetchone())[0] == 0:
                await db.execute("INSERT INTO stats (messages_processed, links_converted) VALUES (0, 0)")
        
        # Set default tags if not exists
        await db.execute("INSERT OR IGNORE INTO settings (key, value) VALUES ('amazon_tag', ?)", (DEFAULT_AMAZON_TAG,))
        await db.execute("INSERT OR IGNORE INTO settings (key, value) VALUES ('flipkart_tag', ?)", (DEFAULT_FLIPKART_TAG,))
        await db.commit()

async def get_setting(key: str, default: str) -> str:
    async with aiosqlite.connect(DB_FILE) as db:
        async with db.execute("SELECT value FROM settings WHERE key = ?", (key,)) as cursor:
            row = await cursor.fetchone()
            return row[0] if row else default

async def set_setting(key: str, value: str):
    async with aiosqlite.connect(DB_FILE) as db:
        await db.execute("INSERT OR REPLACE INTO settings (key, value) VALUES (?, ?)", (key, value))
        await db.commit()

async def register_user(user_id: int):
    async with aiosqlite.connect(DB_FILE) as db:
        await db.execute("INSERT OR IGNORE INTO users (user_id) VALUES (?)", (user_id,))
        await db.commit()

async def increment_stats(messages: int = 1, links: int = 0):
    async with aiosqlite.connect(DB_FILE) as db:
        await db.execute(
            "UPDATE stats SET messages_processed = messages_processed + ?, links_converted = links_converted + ? WHERE id = 1",
            (messages, links)
        )
        await db.commit()

async def fetch_stats():
    async with aiosqlite.connect(DB_FILE) as db:
        async with db.execute("SELECT messages_processed, links_converted FROM stats WHERE id = 1") as cursor:
            stats = await cursor.fetchone()
        async with db.execute("SELECT COUNT(*) FROM users") as cursor:
            total_users = (await cursor.fetchone())[0]
        return stats[0] if stats else 0, stats[1] if stats else 0, total_users

# --- ADVANCED LINK EXPANDER & PARSER ---

async def expand_url_cached(url: str) -> str:
    """Async URL Unshortener with In-Memory Caching"""
    if url in URL_CACHE:
        return URL_CACHE[url]

    shorteners = ["amzn.to", "bit.ly", "fkrt.it", "tinyurl.com", "cutt.ly", "m.tb.cn"]
    if not any(domain in url for domain in shorteners):
        return url

    try:
        async with httpx.AsyncClient(follow_redirects=True, timeout=6.0) as client:
            res = await client.head(url)
            expanded = str(res.url)
            URL_CACHE[url] = expanded
            return expanded
    except Exception as e:
        logger.warning(f"Failed to expand URL {url}: {e}")
        return url

async def convert_affiliate_links(text: str) -> tuple[str, str | None, int]:
    """Multi-Store Link Detection & Tag Injection Engine"""
    if not text:
        return "", None, 0

    amazon_tag = await get_setting("amazon_tag", DEFAULT_AMAZON_TAG)
    flipkart_tag = await get_setting("flipkart_tag", DEFAULT_FLIPKART_TAG)

    url_pattern = r"(https?://[^\s]+)"
    urls = re.findall(url_pattern, text)
    converted_count = 0
    first_deal_url = None

    for original_url in urls:
        expanded_url = await expand_url_cached(original_url)
        parsed = urlparse(expanded_url)
        query = parse_qs(parsed.query)

        new_url = expanded_url

        # 1. Amazon
        if "amazon" in parsed.netloc or "amzn" in parsed.netloc:
            query["tag"] = [amazon_tag]
            converted_count += 1
            new_query = urlencode(query, doseq=True)
            new_url = urlunparse((parsed.scheme, parsed.netloc, parsed.path, parsed.params, new_query, parsed.fragment))

        # 2. Flipkart
        elif "flipkart" in parsed.netloc or "fkrt" in parsed.netloc:
            query["affid"] = [flipkart_tag]
            converted_count += 1
            new_query = urlencode(query, doseq=True)
            new_url = urlunparse((parsed.scheme, parsed.netloc, parsed.path, parsed.params, new_query, parsed.fragment))

        if not first_deal_url and converted_count > 0:
            first_deal_url = new_url

        text = text.replace(original_url, new_url)

    # Clean Spams, External Links, and Watermarks
    patterns_to_clean = [
        r"@[A-Za-z0-9_]+",                         # Remove @usernames
        r"https?://t\.me/[^\s]+",                  # Remove Telegram channel links
        r"https?://instagram\.com/[^\s]+",         # Remove IG links
        r"\+?\d{10,12}",                           # Remove Phone numbers
        r"(?i)join\s+our\s+channel.*",             # Remove promotional lines
        r"(?i)credit\s*:.*"                        # Remove credit lines
    ]
    for pat in patterns_to_clean:
        text = re.sub(pat, "", text)

    return text.strip(), first_deal_url, converted_count

# --- COMMAND HANDLERS ---

async def start_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    await register_user(user.id)

    welcome_text = (
        f"🚀 **Enterprise Affiliate Engine Active!**\n\n"
        f"Namaste **{user.first_name}**!\n"
        "Send me any deal text, image, or video with Amazon/Flipkart links.\n\n"
        "⚙️ **Features:** Auto Tag Replacing, Watermark Cleaner, Persistent Database & Dynamic Buttons."
    )

    keyboard = [
        [InlineKeyboardButton("📊 Analytics Dashboard", callback_data="view_stats")],
        [InlineKeyboardButton("⚡ System Status", callback_data="view_status")]
    ]
    await update.message.reply_text(welcome_text, parse_mode="Markdown", reply_markup=InlineKeyboardMarkup(keyboard))

async def settag_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Admin Command: Dynamic tag update (/settag amazon yourtag-21)"""
    user = update.effective_user
    if user.id != OWNER_ID and OWNER_ID != 0:
        await update.message.reply_text("❌ Aapko is command ka access nahi hai.")
        return

    if len(context.args) < 2:
        await update.message.reply_text("Usage: `/settag [amazon/flipkart] [your_tag]`", parse_mode="Markdown")
        return

    store = context.args[0].lower()
    tag = context.args[1]

    if store == "amazon":
        await set_setting("amazon_tag", tag)
        await update.message.reply_text(f"✅ Amazon Tag updated to: `{tag}`", parse_mode="Markdown")
    elif store == "flipkart":
        await set_setting("flipkart_tag", tag)
        await update.message.reply_text(f"✅ Flipkart Tag updated to: `{tag}`", parse_mode="Markdown")
    else:
        await update.message.reply_text("Invalid store! Use `amazon` or `flipkart`.")

async def broadcast_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Admin Command: Send message to all users (/broadcast Hello World)"""
    user = update.effective_user
    if user.id != OWNER_ID and OWNER_ID != 0:
        await update.message.reply_text("❌ Admin access required!")
        return

    if not context.args:
        await update.message.reply_text("Usage: `/broadcast Your message here`", parse_mode="Markdown")
        return

    msg = " ".join(context.args)
    count = 0

    async with aiosqlite.connect(DB_FILE) as db:
        async with db.execute("SELECT user_id FROM users") as cursor:
            users = await cursor.fetchall()

    for uid in users:
        try:
            await context.bot.send_message(chat_id=uid[0], text=msg)
            count += 1
            await asyncio.sleep(0.05)
        except Exception:
            continue

    await update.message.reply_text(f"📢 Broadcast sent to `{count}` users!")

async def button_callback_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()

    if query.data == "view_stats":
        m_count, l_count, users = await fetch_stats()
        text = (
            "📈 **Live Server Analytics**\n\n"
            f"• Total Users Registered: `{users}`\n"
            f"• Total Posts Processed: `{m_count}`\n"
            f"• Affiliate Links Converted: `{l_count}`"
        )
        await query.edit_message_text(text, parse_mode="Markdown")
    elif query.data == "view_status":
        amz = await get_setting("amazon_tag", DEFAULT_AMAZON_TAG)
        fk = await get_setting("flipkart_tag", DEFAULT_FLIPKART_TAG)
        text = (
            "⚙️ **Active Configuration**\n\n"
            f"• Amazon Tag: `{amz}`\n"
            f"• Flipkart Tag: `{fk}`\n"
            f"• Database Status: `Connected (SQLite)`"
        )
        await query.edit_message_text(text, parse_mode="Markdown")

# --- CORE MESSAGE PROCESSOR ---

async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    msg = update.effective_message
    raw_text = msg.text or msg.caption or ""

    if not raw_text:
        return

    cleaned_text, deal_url, converted_count = await convert_affiliate_links(raw_text)
    await increment_stats(messages=1, links=converted_count)

    # Dynamic Buttons
    keyboard = []
    if deal_url:
        keyboard.append([InlineKeyboardButton("🛒 Direct Deal Link", url=deal_url)])
    keyboard.append([InlineKeyboardButton("📢 Share Deal", url=f"https://t.me/share/url?url={deal_url or ''}&text=Amazing%20Offer!")])

    reply_markup = InlineKeyboardMarkup(keyboard)

    # Send Cleaned Output
    if msg.photo:
        await msg.reply_photo(photo=msg.photo[-1].file_id, caption=cleaned_text, reply_markup=reply_markup)
    elif msg.video:
        await msg.reply_video(video=msg.video.file_id, caption=cleaned_text, reply_markup=reply_markup)
    else:
        await msg.reply_text(cleaned_text, reply_markup=reply_markup, disable_web_page_preview=False)

# --- MAIN INITIALIZER ---

def main():
    if not BOT_TOKEN:
        print("❌ ERROR: BOT_TOKEN missing in .env file!")
        return

    # Init DB asynchronously
    asyncio.run(init_db())

    app = Application.builder().token(BOT_TOKEN).build()

    # Register Handlers
    app.add_handler(CommandHandler("start", start_command))
    app.add_handler(CommandHandler("settag", settag_command))
    app.add_handler(CommandHandler("broadcast", broadcast_command))
    app.add_handler(CallbackQueryHandler(button_callback_handler))
    app.add_handler(MessageHandler(filters.TEXT | filters.PHOTO | filters.VIDEO, handle_message))

    print("🚀 Enterprise Bot Engine Started Successfully!")
    app.run_polling()

if __name__ == "__main__":
    main()
