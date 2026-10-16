from pathlib import Path

code = r'''import tkinter as tk
from tkinter import messagebox, scrolledtext
import ast
import operator
import asyncio
import threading
import queue
from datetime import datetime

from telegram import Update
from telegram.ext import (
    Application,
    CommandHandler,
    MessageHandler,
    ContextTypes,
    filters,
)

# Повідомлення з потоку бота передаються у графічне вікно через чергу.
logs = queue.Queue()
bot_thread = None
stop_event = None
running = False

# Нотатки зберігаються окремо для кожного користувача.
# Після перезапуску програми вони очищуються.
notes = {}

# Дозволені математичні операції — без використання eval().
OPERATIONS = {
    ast.Add: operator.add,
    ast.Sub: operator.sub,
    ast.Mult: operator.mul,
    ast.Div: operator.truediv,
    ast.FloorDiv: operator.floordiv,
    ast.Mod: operator.mod,
    ast.Pow: operator.pow,
    ast.UAdd: operator.pos,
    ast.USub: operator.neg,
}


def calculate(expression):
    """Безпечно обчислює простий математичний вираз."""
    if not expression or len(expression) > 100:
        raise ValueError("Некоректна довжина виразу.")

    tree = ast.parse(expression, mode="eval")

    def evaluate(node):
        if isinstance(node, ast.Expression):
            return evaluate(node.body)

        if isinstance(node, ast.Constant) and type(node.value) in (int, float):
            if abs(node.value) > 10**12:
                raise ValueError("Число занадто велике.")
            return node.value

        if isinstance(node, ast.BinOp):
            operation = OPERATIONS.get(type(node.op))
            if operation is None:
                raise ValueError("Ця операція заборонена.")

            left = evaluate(node.left)
            right = evaluate(node.right)

            if isinstance(node.op, ast.Pow) and abs(right) > 10:
                raise ValueError("Занадто великий степінь.")

            result = operation(left, right)

            if not isinstance(result, (int, float)) or abs(result) > 10**100:
                raise ValueError("Результат занадто великий.")
            return result

        if isinstance(node, ast.UnaryOp):
            operation = OPERATIONS.get(type(node.op))
            if operation is None:
                raise ValueError("Ця операція заборонена.")
            return operation(evaluate(node.operand))

        raise ValueError("Використовуй лише числа та арифметичні знаки.")

    return evaluate(tree)


def get_user_id(update: Update):
    return update.effective_user.id


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.effective_message.reply_text(
        "Привіт! Я бот-помічник.\n\n"
        "Можеш писати звичайні повідомлення:\n"
        "• привіт — привітання\n"
        "• час — поточна дата й час\n"
        "• порахуй 12*3 — калькулятор\n"
        "• нотатка: повторити Python — зберегти нотатку\n"
        "• моя нотатка — показати нотатку\n\n"
        "Команди: /time, /calc, /note, /mynote, /help"
    )


async def help_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await start(update, context)


async def send_time(update: Update, context: ContextTypes.DEFAULT_TYPE):
    now = datetime.now().strftime("%d.%m.%Y, %H:%M:%S")
    await update.effective_message.reply_text(f"Зараз: {now}")


async def calc_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    expression = " ".join(context.args)
    await calculate_and_reply(update, expression)


async def calculate_and_reply(update: Update, expression: str):
    if not expression:
        await update.effective_message.reply_text(
            "Напиши вираз, наприклад: /calc 25*4 або «порахуй 25*4»."
        )
        return

    try:
        result = calculate(expression)
        await update.effective_message.reply_text(f"Результат: {result}")
    except (ValueError, TypeError, ArithmeticError, SyntaxError, OverflowError):
        await update.effective_message.reply_text(
            "Не вдалося обчислити. Приклад: порахуй 10+5"
        )


async def save_note(update: Update, context: ContextTypes.DEFAULT_TYPE):
    note = " ".join(context.args).strip()
    await save_note_text(update, note)


async def save_note_text(update: Update, note: str):
    if not note:
        await update.effective_message.reply_text(
            "Напиши текст нотатки. Наприклад: нотатка: зробити домашнє завдання"
        )
        return

    notes[get_user_id(update)] = note
    await update.effective_message.reply_text("Нотатку збережено!")


async def show_note(update: Update, context: ContextTypes.DEFAULT_TYPE):
    note = notes.get(get_user_id(update))
    if note:
        await update.effective_message.reply_text(f"Твоя нотатка:\n{note}")
    else:
        await update.effective_message.reply_text(
            "Нотаток поки немає. Напиши: нотатка: твій текст"
        )


async def handle_text(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Обробник звичайних текстових повідомлень, не лише команд."""
    text = update.effective_message.text.strip()
    lowered = text.casefold()

    greetings = ("привіт", "привіт!", "доброго дня", "добрий вечір", "hello", "hi")
    if lowered in greetings:
        await update.effective_message.reply_text(
            "Привіт! 😊 Я твій помічник. Напиши «допомога», щоб побачити можливості."
        )
        return

    if lowered in ("час", "котра година", "дата", "який час"):
        await send_time(update, context)
        return

    if lowered.startswith("порахуй "):
        await calculate_and_reply(update, text[len("порахуй "):].strip())
        return

    if lowered.startswith("calc "):
        await calculate_and_reply(update, text[len("calc "):].strip())
        return

    if lowered.startswith("нотатка:"):
        note = text.split(":", 1)[1].strip()
        await save_note_text(update, note)
        return

    if lowered in ("моя нотатка", "покажи нотатку"):
        await show_note(update, context)
        return

    if lowered in ("допомога", "що ти вмієш", "help"):
        await help_command(update, context)
        return

    await update.effective_message.reply_text(
        "Не зрозумів повідомлення. Напиши «допомога», щоб побачити приклади."
    )


async def run_bot(token: str, stop_flag: threading.Event):
    app = Application.builder().token(token).build()

    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("help", help_command))
    app.add_handler(CommandHandler("time", send_time))
    app.add_handler(CommandHandler("calc", calc_command))
    app.add_handler(CommandHandler("note", save_note))
    app.add_handler(CommandHandler("mynote", show_note))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_text))

    await app.initialize()
    await app.start()

    try:
        await app.updater.start_polling()
        logs.put(("status", "Бот працює"))
        logs.put(("log", "Бота успішно запущено. Відкрий його в Telegram і натисни Start."))

        while not stop_flag.is_set():
            await asyncio.sleep(0.2)

    finally:
        if app.updater and app.updater.running:
            await app.updater.stop()
        if app.running:
            await app.stop()
        await app.shutdown()


def bot_worker(token: str, stop_flag: threading.Event):
    try:
        asyncio.run(run_bot(token, stop_flag))
    except Exception as error:
        # Токен навмисно не записується у журнал.
        logs.put(("log", f"Не вдалося запустити бота: {error}"))
    finally:
        logs.put(("stopped", "Бот зупинений"))


def start_bot():
    global bot_thread, stop_event, running

    if running:
        return

    token = token_entry.get().strip()
    if not token:
        messagebox.showwarning("Потрібен токен", "Встав токен, отриманий від BotFather.")
        return

    running = True
    stop_event = threading.Event()
    start_button.config(state="disabled")
    stop_button.config(state="normal")
    status_label.config(text="Підключення...")
    log_box.insert(tk.END, "Запускаємо бота...\n")

    bot_thread = threading.Thread(
        target=bot_worker,
        args=(token, stop_event),
        daemon=True,
    )
    bot_thread.start()


def stop_bot():
    if stop_event is not None:
        stop_event.set()
        stop_button.config(state="disabled")
        status_label.config(text="Зупинення...")


def check_logs():
    global running, stop_event

    try:
        while True:
            kind, message = logs.get_nowait()

            if kind == "status":
                status_label.config(text=message)
            elif kind == "stopped":
                running = False
                stop_event = None
                start_button.config(state="normal")
                stop_button.config(state="disabled")
                status_label.config(text=message)
                log_box.insert(tk.END, message + "\n")
            else:
                log_box.insert(tk.END, message + "\n")
                log_box.see(tk.END)
    except queue.Empty:
        pass

    root.after(200, check_logs)


def close_window():
    if stop_event is not None:
        stop_event.set()
    root.destroy()


# ---------------- Графічний інтерфейс ----------------
root = tk.Tk()
root.title("Telegram Bot Helper — ITSTEP")
root.geometry("590x470")
root.resizable(False, False)

title_label = tk.Label(
    root,
    text="TELEGRAM BOT HELPER",
    font=("Arial", 18, "bold"),
)
title_label.pack(pady=(16, 8))

tk.Label(root, text="Встав токен від BotFather:").pack()

token_entry = tk.Entry(root, width=60, show="*")
token_entry.pack(pady=7)

buttons_frame = tk.Frame(root)
buttons_frame.pack(pady=7)

start_button = tk.Button(
    buttons_frame,
    text="Задати та запустити",
    command=start_bot,
    width=20,
)
start_button.pack(side=tk.LEFT, padx=5)

stop_button = tk.Button(
    buttons_frame,
    text="Зупинити бота",
    command=stop_bot,
    width=16,
    state="disabled",
)
stop_button.pack(side=tk.LEFT, padx=5)

status_label = tk.Label(root, text="Бот не працює", font=("Arial", 11))
status_label.pack(pady=5)

tk.Label(root, text="Журнал роботи:").pack()

log_box = scrolledtext.ScrolledText(root, width=68, height=12)
log_box.pack(padx=10, pady=6)

root.protocol("WM_DELETE_WINDOW", close_window)
root.after(200, check_logs)
root.mainloop()
'''
path = Path("/mnt/data/telegram_helper_itstep.py")
path.write_text(code, encoding="utf-8")
print(f"Створено файл: {path}")
