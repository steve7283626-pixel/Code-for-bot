# Code-for-bot
Code for bot
# bot.py
# Требования:
# pip install python-telegram-bot==20.5 cryptography
#


import sqlite3
import secrets
import hashlib
import time
from datetime import datetime, timedelta
from typing import Optional

from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, ContextTypes

# ----------------- Настройки -----------------
DB = 'checks.db'
ADMIN_IDS = {6621272273}  # 
BOT_TOKEN = 8195685245:AAGaYsZpNdFLufZL7smq_AUSR37oNaZ0qm4 # 
# ---------------------------------------------

# ---------- База данных и инициализация ----------
def init_db():
    conn = sqlite3.connect(DB)
    cur = conn.cursor()
    cur.execute('''
    CREATE TABLE IF NOT EXISTS users (
        telegram_id INTEGER PRIMARY KEY,
        username TEXT,
        balance INTEGER DEFAULT 0
    )''')
    cur.execute('''
    CREATE TABLE IF NOT EXISTS checks (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        code_hash TEXT UNIQUE,
        code_display TEXT,
        amount INTEGER,
        created_by INTEGER,
        created_at INTEGER,
        expires_at INTEGER,
        used_by INTEGER,
        used_at INTEGER,
        used_ip TEXT,
        max_uses INTEGER DEFAULT 1,
        uses INTEGER DEFAULT 0
    )''')
    # Таблица истории отправленных чеков / попыток доставки
    cur.execute('''
    CREATE TABLE IF NOT EXISTS transfers (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        check_id INTEGER,
        code_display TEXT,
        amount INTEGER,
        created_by INTEGER,
        target TEXT, -- @username или user_id (строка)
        target_chat_id INTEGER, -- фактический chat_id если удалось определить
        created_at INTEGER,
        delivered INTEGER DEFAULT 0, -- 0/1
        delivered_at INTEGER,
        error TEXT
    )''')
    conn.commit()
    conn.close()

def hash_code(code: str) -> str:
    return hashlib.sha256(code.encode('utf-8')).hexdigest()

def generate_code(display_len=12) -> str:
    # генерируем читаемый код, разбиваем через '-'
    raw = secrets.token_urlsafe(16)  # ~22 chars
    cleaned = ''.join(ch for ch in raw.upper() if ch.isalnum())
    # гарантируем минимальную длину
    if len(cleaned) < display_len:
        cleaned = (cleaned * ((display_len // len(cleaned)) + 1))[:display_len]
    parts = [cleaned[i:i+4] for i in range(0, display_len, 4)]
    return '-'.join(parts)

# ---------- Утилиты БД ----------
def db_connect():
    return sqlite3.connect(DB)

def format_ts(ts: Optional[int]) -> str:
    if not ts:
        return "—"
    return datetime.utcfromtimestamp(ts).strftime('%Y-%m-%d %H:%M UTC')

# ---------- Хэндлеры ----------
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "Привет! Я чек-бот.\n"
        "Доступные команды:\n"
        "/balance — показать баланс\n"
        "/redeem <КОД> — погасить чек\n"
        "\nКоманды админов:\n"
        "/makecheck <amount> [days_valid] — создать чек\n"
        "/sendcheck <@username|user_id> <amount> [days_valid] — создать и отправить чек\n"
        "/history [user_id] — показать историю передач (только админам)"
    )

async def balance_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    uid = update.effective_user.id
    conn = db_connect()
    cur = conn.cursor()
    cur.execute("SELECT balance FROM users WHERE telegram_id = ?", (uid,))
    r = cur.fetchone()
    if not r:
        cur.execute("INSERT INTO users(telegram_id, username, balance) VALUES (?,?,?)",
                    (uid, update.effective_user.username or '', 0))
        conn.commit()
        balance = 0
    else:
        balance = r[0]
    conn.close()
    await update.message.reply_text(f"Ваш баланс: {balance} звёзд")

async def makecheck(update: Update, context: ContextTypes.DEFAULT_TYPE):
    uid = update.effective_user.id

if uid not in ADMIN_IDS:
        return await update.message.reply_text("Команда доступна только админам.")
    if len(context.args) < 1:
        return await update.message.reply_text("Использование: /makecheck <amount> [days_valid]")
    try:
        amount = int(context.args[0])
        days = int(context.args[1]) if len(context.args) >= 2 else 7
    except:
        return await update.message.reply_text("Неверный формат. /makecheck <amount> [days_valid]")
    code = generate_code()
    code_h = hash_code(code)
    now = int(time.time())
    expires = int((datetime.utcnow() + timedelta(days=days)).timestamp())
    conn = db_connect()
    cur = conn.cursor()
    cur.execute('INSERT INTO checks(code_hash, code_display, amount, created_by, created_at, expires_at) VALUES (?,?,?,?,?,?)',
                (code_h, code, amount, uid, now, expires))
    conn.commit()
    conn.close()
    await update.message.reply_text(
        f"Чек создан на {amount} звёзд, действует до {format_ts(expires)}\nКод: {code}"
    )

async def redeem(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if len(context.args) < 1:
        return await update.message.reply_text("Использование: /redeem <КОД>")
    raw_code = context.args[0].strip().upper()
    raw_code = ''.join(ch for ch in raw_code if ch.isalnum())
    # Попытка восстановить возможный формат с дефисами (4-4-4 ...)
    variants = []
    if len(raw_code) >= 12:
        variants.append(raw_code[:4] + '-' + raw_code[4:8] + '-' + raw_code[8:12])
    variants.append(raw_code)
    conn = db_connect()
    cur = conn.cursor()
    found = None
    for v in variants:
        h = hash_code(v)
        cur.execute("SELECT id, amount, expires_at, uses, max_uses, code_display FROM checks WHERE code_hash = ?", (h,))
        row = cur.fetchone()
        if row:
            found = (row, v, h)
            break
    if not found:
        conn.close()
        return await update.message.reply_text("Чек не найден или недействителен.")
    row, used_display, h = found
    cid, amount, expires_at, uses, max_uses, stored_display = row
    now = int(time.time())
    if expires_at and now > expires_at:
        conn.close()
        return await update.message.reply_text("Чек истёк.")
    if uses >= max_uses:
        conn.close()
        return await update.message.reply_text("Чек уже использован.")
    # Начисляем баланс
    uid = update.effective_user.id
    cur.execute("INSERT OR IGNORE INTO users(telegram_id, username, balance) VALUES (?,?,?)",
                (uid, update.effective_user.username or '', 0))
    cur.execute("UPDATE users SET balance = balance + ? WHERE telegram_id = ?", (amount, uid))
    cur.execute("UPDATE checks SET uses = uses + 1, used_by = ?, used_at = ?, used_ip = ? WHERE id = ?",
                (uid, now, update.effective_user.id, cid))
    conn.commit()
    conn.close()
    await update.message.reply_text(f"Успешно! Вам начислено {amount} звёзд. Используйте /balance чтобы проверить остаток.")

# Импорт ошибки здесь не нужен отдельно; telegram.ext выбрасывает свои исключения при отправке
async def sendcheck(update: Update, context: ContextTypes.DEFAULT_TYPE):
    uid = update.effective_user.id
    if uid not in ADMIN_IDS:
        return await update.message.reply_text("Команда доступна только админам.")
    if len(context.args) < 2:
        return await update.message.reply_text("Использование: /sendcheck <@username|user_id> <amount> [days_valid]")
    target = context.args[0]
    try:
        amount = int(context.args[1])
    except:
        return await update.message.reply_text("Неверная сумма. Использование: /sendcheck <@username|user_id> <amount> [days_valid]")
    days = int(context.args[2]) if len(context.args) >= 3 else 7

    # Генерируем чек
    code = generate_code()
    code_h = hash_code(code)
    now = int(time.time())
    expires = int((datetime.utcnow() + timedelta(days=days)).timestamp())

conn = db_connect()
    cur = conn.cursor()
    cur.execute('INSERT INTO checks(code_hash, code_display, amount, created_by, created_at, expires_at) VALUES (?,?,?,?,?,?)',
                (code_h, code, amount, uid, now, expires))
    check_id = cur.lastrowid
    conn.commit()

    # Попытка получить chat_id по username или использовать переданный id
    chat_id = None
    error_text = None
    try:
        if target.startswith('@'):
            chat = await context.bot.get_chat(target)
            chat_id = chat.id
        else:
            # попробуем преобразовать в int (user_id)
            chat_id = int(target)
    except Exception as e:
        chat_id = None
        error_text = str(e)

    # Записываем в историю transfer
    cur.execute('''
        INSERT INTO transfers(check_id, code_display, amount, created_by, target, target_chat_id, created_at, delivered, error)
        VALUES (?,?,?,?,?,?,?,?,?)
    ''', (check_id, code, amount, uid, target, chat_id, now, 0 if chat_id is None else 1, error_text))
    transfer_id = cur.lastrowid
    conn.commit()
    conn.close()

    message_text = (
        f"Вам отправлен чек на {amount} звёзд.\n\n"
        f"Код: {code}\n\n"
        "Чтобы получить звёзды, отправьте боту:\n"
        f"/redeem {code}\n\n"
        f"Чек действует до {format_ts(expires)}."
    )

    if chat_id is None:
        # не получилось определить — возвращаем код админу
        await update.message.reply_text(
            "Не удалось доставить сообщение автоматически (возможно, пользователь не запускал бота). "
            "Вот код, пересылайте вручную:\n\n" + code
        )
        return

    try:
        await context.bot.send_message(chat_id=chat_id, text=message_text)
        # Обновим запись transfers как доставленную
        conn = db_connect()
        cur = conn.cursor()
        cur.execute("UPDATE transfers SET delivered = 1, delivered_at = ? WHERE id = ?", (int(time.time()), transfer_id))
        conn.commit()
        conn.close()
        await update.message.reply_text(f"Чек отправлен пользователю {target}.")
    except Exception as e:
        # обновим запись с ошибкой
        conn = db_connect()
        cur = conn.cursor()
        cur.execute("UPDATE transfers SET delivered = 0, error = ? WHERE id = ?", (str(e), transfer_id))
        conn.commit()
        conn.close()
        await update.message.reply_text(
            f"Не удалось отправить чек пользователю {target}: {e}\n\nПерешлите ему этот код вручную:\n{code}"
        )

async def history(update: Update, context: ContextTypes.DEFAULT_TYPE):
    uid = update.effective_user.id
    if uid not in ADMIN_IDS:
        return await update.message.reply_text("Команда доступна только админам.")
    # опционально: /history [user_id]
    user_filter = None
    if len(context.args) >= 1:
        try:
            user_filter = int(context.args[0])
        except:
            user_filter = None
    conn = db_connect()
    cur = conn.cursor()
    if user_filter:
        cur.execute('''
            SELECT id, code_display, amount, created_by, target, target_chat_id, created_at, delivered, delivered_at, error
            FROM transfers WHERE created_by = ? OR target_chat_id = ? ORDER BY created_at DESC LIMIT 50
        ''', (user_filter, user_filter))
    else:
        cur.execute('''
            SELECT id, code_display, amount, created_by, target, target_chat_id, created_at, delivered, delivered_at, error
            FROM transfers ORDER BY created_at DESC LIMIT 50
        ''')
    rows = cur.fetchall()
    conn.close()
    if not rows:
        return await update.message.reply_text("Записей не найдено.")
    lines = []
    for r in rows:
        tid, code_display, amount, created_by, target, target_chat_id, created_at, delivered, delivered_at, error = r

line = (
            f"#{tid} {code_display} — {amount}⭐ от {created_by} -> {target or '—'} "
            f"(chat_id:{target_chat_id or '—'})\n"
            f"создан: {format_ts(created_at)}, доставлен: {'да' if delivered else 'нет'} {format_ts(delivered_at) if delivered_at else ''}"
        )
        if error:
            line += f"\nОшибка: {error}"
        lines.append(line)
    # Telegram message limit: отправляем в нескольких сообщениях, если нужно
    chunk = []
    msg = ""
    for l in lines:
        if len(msg) + len(l) + 2 > 4000:
            await update.message.reply_text(msg)
            msg = l + "\n\n"
        else:
            msg += l + "\n\n"
    if msg:
        await update.message.reply_text(msg)

# ---------- Запуск ----------
def main():
    init_db()
    app = ApplicationBuilder().token(BOT_TOKEN).build()

    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("balance", balance_cmd))
    app.add_handler(CommandHandler("makecheck", makecheck))
    app.add_handler(CommandHandler("redeem", redeem))
    app.add_handler(CommandHandler("sendcheck", sendcheck))
    app.add_handler(CommandHandler("history", history))

    print("Bot is starting...")
    app.run_polling()

if name == "main":
    main()
