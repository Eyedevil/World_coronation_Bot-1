from aiogram.types import ContentType
import logging
import sqlite3
import asyncio
from quart import Quart, request, jsonify
from aiogram import Bot, Dispatcher, types
from aiogram.enums import ParseMode
from aiogram.filters import Command
from aiogram.fsm.state import State, StatesGroup
from aiogram.fsm.context import FSMContext
from aiogram.fsm.storage.memory import MemoryStorage

# Logging setup
logging.basicConfig(level=logging.INFO)

import os

# Quart app setup
app = Quart(__name__)

# Bot setup
BOT_USERNAME = "@WCS99_Bot"
API_TOKEN = os.getenv("8061678073:AAFdr2QpYLF7IiXJQBgHM1ExqwMCDCdFkiw")
ADMIN_ID = int(os.getenv("8059940397", 0))
TARGET_GROUP_ID = int(os.getenv("-1002352388643", 0))

# Automatically set the WEBHOOK_URL based on Railway's assigned domain

WEBHOOK_PATH = "/webhook"
RAILWAY_STATIC_URL = os.getenv("RAILWAY_STATIC_URL", "http://127.0.0.1:5000")
WEBHOOK_URL = f"{os.getenv('RAILWAY_STATIC_URL', 'https://your-default-domain.com')}/webhook"

# Validate WEBHOOK_URL
if not WEBHOOK_URL:
    raise RuntimeError(
        "Error: RAILWAY_STATIC_URL is not set. Ensure your project is hosted on Railway and the environment variable is available."
    )

# Set webhook on startup
async def on_startup():
    await bot.set_webhook(WEBHOOK_URL)

@app.before_serving
async def startup():
    await on_startup()

from aiogram.types import Update

@app.route("/webhook", methods=["POST"])
async def webhook():
    request_data = await request.get_json()
    update = Update(**request_data)
    await dp.process_update(update)
    return jsonify({"ok": True})

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.getenv("PORT", 5000)), use_reloader=False)

# Initialize the bot and dispatcher
bot = Bot(token=API_TOKEN, parse_mode=ParseMode.HTML)
storage = MemoryStorage()
dp = Dispatcher(bot=bot, storage=storage)

# SQLite setup
conn = sqlite3.connect("database.db", check_same_thread=False)
cursor = conn.cursor()

# Database setup
cursor.execute('''
CREATE TABLE IF NOT EXISTS users (
    user_id INTEGER PRIMARY KEY,
    name TEXT,
    username TEXT,
    points INTEGER DEFAULT 0,
    rank TEXT DEFAULT 'Beginner',
    wins INTEGER DEFAULT 0
)
''')

cursor.execute('''
CREATE TABLE IF NOT EXISTS events (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    event_name TEXT NOT NULL,
    event_details TEXT NOT NULL,
    is_active INTEGER DEFAULT 0
)
''')
conn.commit()

# Webhook route
@app.route("/webhook", methods=["POST"])
async def webhook():
    update = await request.get_json()  # Process incoming webhook data
    await dp.feed_update(update)
    return jsonify({"ok": True})

@app.route("/")
async def index():
    return jsonify({"message": "World Coronation Bot is running!"})

# Set webhook on startup
# Define BOT_USERNAME as a global variable
BOT_USERNAME = None

# Set webhook on startup
async def on_startup():
    global BOT_USERNAME  # Use the global variable
    await bot.set_webhook(WEBHOOK_URL)
    BOT_USERNAME = (await bot.get_me()).username  # Initialize BOT_USERNAME here

@app.before_serving
async def startup():
    await on_startup()

@app.after_serving
async def shutdown():
    await bot.delete_webhook()
    await dp.storage.close()
    await dp.storage.wait_closed()

@dp.message(Command("invite"))
async def invite_command(message: types.Message):
    if BOT_USERNAME:
        invite_link = f"https://t.me/{BOT_USERNAME}"
        await message.answer(f"Invite others to join using this link: {invite_link}")
    else:
        await message.answer("Bot is still initializing. Please try again later.")
async def on_startup():
    await bot.set_webhook(WEBHOOK_URL)

@app.before_serving
async def startup():
    await on_startup()

@app.after_serving
async def shutdown():
    await bot.delete_webhook()
    await dp.storage.close()
    await dp.storage.wait_closed()

# --- User Commands ---
@dp.message(Command("start"))
async def start_command(message: types.Message):
    await message.answer(
        "Welcome to World Coronation Bot! Use /help to view all available commands."
    )

@dp.message(Command("help"))
async def help_command(message: types.Message):
    commands = """
<b>Available Commands:</b>
- /start
- /help
- /register
- /profile
- /rank
- /leaderboard
- /leaderboard_full
- /event
- /stats
- /invite
- /top_winners
- /report
- /giveprize250
- /lose150
- /increase
- /decrease
"""
    await message.answer(commands)

@dp.message(Command("register"))
async def register(message: types.Message, state: FSMContext):
    await message.answer("Please send your name.")
    await state.set_state(RegisterState.name)

@dp.message(RegisterState.name)
async def process_name(message: types.Message, state: FSMContext):
    await state.update_data(name=message.text)
    await message.answer("Please provide your username (without @).")
    await state.set_state(RegisterState.username)

@dp.message(RegisterState.username)
async def process_username(message: types.Message, state: FSMContext):
    await state.update_data(username=message.text)
    await message.answer("Please upload your profile picture.")
    await state.set_state(RegisterState.photo)

@dp.message(RegisterState.photo, F.content_type == ContentType.PHOTO)
async def process_photo(message: types.Message, state: FSMContext):
    user_data = await state.get_data()
    cursor.execute(
        "INSERT OR REPLACE INTO users (user_id, name, username, rank) VALUES (?, ?, ?, ?)",
        (message.from_user.id, user_data["name"], user_data["username"], "Beginner"),
    )
    conn.commit()
    await message.answer("Registration complete!✅")
    await state.clear()

# Add other commands here (similar to the Flask implementation)...

# Start the Quart app with the asyncio event loop
if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, use_reloader=False)

# Admin ID
ADMIN_ID = 8059940397  # Replace with your Telegram admin ID
TARGET_GROUP_ID = -1002408269202  # Replace with your group's chat ID


# --- States for Registration ---
class RegisterState(StatesGroup):
    name = State()
    username = State()
    photo = State()

@dp.message(Command("profile"))
async def profile_command(message: types.Message):
    user_id = message.from_user.id
    cursor.execute(
        "SELECT name, username, points, rank FROM users WHERE user_id = ?", (user_id,)
    )
    user_data = cursor.fetchone()
    if user_data:
        name, username, points, rank = user_data
        await message.answer(
            f"<b>Your Profile:</b>\n"
            f"Name: {name}\n"
            f"Username: @{username}\n"
            f"Points: {points}\n"
            f"Rank: {rank}"
        )
    else:
        await message.answer("You are not registered yet. Use /register to sign up.")


@dp.message(Command("rank"))
async def rank_command(message: types.Message):
    user_id = message.from_user.id
    cursor.execute("SELECT rank FROM users WHERE user_id = ?", (user_id,))
    rank = cursor.fetchone()
    if rank:
        await message.answer(f"Your current rank is <b>{rank[0]}</b>.")
    else:
        await message.answer("You are not ranked yet.")


@dp.message(Command("leaderboard"))
async def leaderboard_command(message: types.Message):
    cursor.execute("SELECT username, points FROM users ORDER BY points DESC LIMIT 10")
    data = cursor.fetchall()
    leaderboard = "\n".join([f"{i+1}. {u[0]} - {u[1]} points" for i, u in enumerate(data)])
    await message.answer(f"<b>Leaderboard:</b>\n{leaderboard}")

@dp.message(Command("leaderboard_full"))
async def leaderboard_full_command(message: types.Message):
    # Fetch all users sorted by points in descending order
    cursor.execute("SELECT username, points FROM users ORDER BY points DESC")
    data = cursor.fetchall()

    if not data:
        # If no data exists in the database
        await message.answer("The leaderboard is empty. No users have points yet.")
        return

    # Initialize the message to send
    full_board = "<b>Full Leaderboard:</b>\n"

    # Prepare leaderboard data
    for i, user in enumerate(data, start=1):
        # Append each user's rank, username, and points to the leaderboard message
        full_board += f"{i}. @{user[0]} - {user[1]} points\n"
        # Split long leaderboards into multiple messages (Telegram's limit: ~4096 characters)
        if len(full_board) > 4000:
            await message.answer(full_board)  # Send the current part of the leaderboard
            full_board = ""  # Reset for the next part of the leaderboard

    # Send the remaining part of the leaderboard (if any)
    if full_board:
        await message.answer(full_board)


@dp.message(Command("stats"))
async def stats_command(message: types.Message):
    cursor.execute("SELECT COUNT(*), SUM(points) FROM users")
    stats_data = cursor.fetchone()
    total_users = stats_data[0] if stats_data[0] else 0
    total_points = stats_data[1] if stats_data[1] else 0
    await message.answer(
        f"<b>Stats:</b>\n"
        f"Total users: {total_users}\n"
        f"Total points: {total_points}"
    )


@dp.message(Command("top_winners"))
async def top_winners_command(message: types.Message):
    cursor.execute("SELECT username, wins FROM users ORDER BY wins DESC LIMIT 10")
    data = cursor.fetchall()
    top_winners = "\n".join(
        [f"{i+1}. {u[0]} - {u[1]} wins" for i, u in enumerate(data)]
    )
    await message.answer(f"<b>Top Winners:</b>\n{top_winners}")


@dp.message(Command("report"))
async def report_command(message: types.Message):
    issue = message.text.split(" ", 1)[-1].strip()  # Extract the report content
    if issue:
        await bot.send_message(
            ADMIN_ID,
            f"User @{message.from_user.username} reported:\n{issue}",
        )
        await message.answer("Your report has been sent to the admins.")
    else:
        await message.answer("Please specify the issue you want to report.")


# --- Admin Commands ---
@dp.message(Command("giveprize250"))
async def giveprize250_command(message: types.Message):
    if message.from_user.id == ADMIN_ID:
        try:
            user_id = int(message.text.split()[1])  # Get user ID from the command
            cursor.execute(
                "UPDATE users SET points = points + 250 WHERE user_id = ?", (user_id,)
            )
            conn.commit()
            await message.answer(f"✅ 250 points awarded to user ID: {user_id}.")
        except (IndexError, ValueError):
            await message.answer("Error: Please specify a valid user ID.")
    else:
        await message.answer("You are not authorized to use this command.")

@dp.message(Command("lose150"))
async def lose150_command(message: types.Message):
    if message.from_user.id == ADMIN_ID:
        try:
            user_id = int(message.text.split()[1])  # Get user ID from the command
            cursor.execute(
                "UPDATE users SET points = points - 150 WHERE user_id = ?", (user_id,)
            )
            conn.commit()
            await message.answer(f"❌ 150 points deducted from user ID: {user_id}.")
        except (IndexError, ValueError):
            await message.answer("Error: Please specify a valid user ID.")
    else:
        await message.answer("You are not authorized to use this command.")
    return

@dp.message(Command("increase"))
async def increase_points_command(message: types.Message):
    if message.from_user.id == ADMIN_ID:
        try:
            user_id, points = map(int, message.text.split()[1:])  # Expecting 2 args
            cursor.execute(
                "UPDATE users SET points = points + ? WHERE user_id = ?", (points, user_id)
            )
            conn.commit()
            await message.answer(f"✅ {points} points added to user ID: {user_id}.")
        except ValueError:
            await message.answer("Error: Please provide a valid user ID and points.")
    else:
        await message.answer("You are not authorized to use this command.")

@dp.message(Command("start_event"))
async def start_event_command(message: types.Message):
    if message.from_user.id == ADMIN_ID:
        args = message.text.split(" ", 2)
        if len(args) < 3:
            await message.answer("⚠️ Usage: /start_event <event_name> <event_details>")
            return

        event_name, event_details = args[1].strip(), args[2].strip()

        cursor.execute(
            "SELECT * FROM events WHERE event_name = ? AND is_active = 1",
            (event_name,)
        )
        if cursor.fetchone():
            await message.answer(f"⚠️ Event '{event_name}' is already active.")
        else:
            cursor.execute(
                "INSERT INTO events (event_name, event_details, is_active) VALUES (?, ?, 1)",
                (event_name, event_details)
            )
            conn.commit()
            await message.answer(f"✅ Event '{event_name}' has started with details:\n{event_details}")


@dp.message(Command("stop_event"))
async def stop_event_command(message: types.Message):
    if message.from_user.id == ADMIN_ID:
        cursor.execute("UPDATE events SET is_active = 0 WHERE is_active = 1")
        conn.commit()
        await message.answer("❌ All active events have been stopped.")        

@dp.message(Command("event"))
async def event_command(message: types.Message):
    cursor.execute("SELECT event_name, event_details FROM events WHERE is_active = 1")
    event_data = cursor.fetchone()

    if event_data:
        event_name, event_details = event_data
        await message.answer(
            f"<b>Ongoing Event:</b>\n"
            f"• {event_name}\n\n"
            f"<b>Details:</b>\n{event_details}"
        )
    else:
        await message.answer("There are no ongoing events currently.")


cursor.execute(
    "INSERT INTO events (event_name, event_details, is_active) VALUES (?, ?, ?)",
    ("Christmas Eve 🎄🎆", "Make a streak of 5 wins and earn 250 points, it means win 5 matches without losing any.", 1)
)
conn.commit()


async def on_shutdown():
    await bot.delete_webhook()
    await dp.storage.close()
    await dp.storage.wait_closed()
