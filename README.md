
import requests
import psycopg2
import logging
from telegram import Update
from telegram.ext import Application, CommandHandler, CallbackContext
import asyncio

# Telegram Bot Token
TOKEN = "7772749532:AAGhqOlB6nb1BNrb04WPdqFQW3UKFVrN4_k"

# Database Configuration
DB_NAME = "your_db_name"
DB_USER = "your_db_user"
DB_PASSWORD = "your_db_password"
DB_HOST = "your_db_host"
DB_PORT = "your_db_port"

# Scraper Function to Fetch Latest Lottery Results
LOTTERY_URL = "https://example.com/latest-lottery"

def fetch_lottery_results():
    try:
        response = requests.get(LOTTERY_URL)
        data = response.json()
        latest_result = data.get("latest_winning_number")
        countdown = data.get("next_draw_time")
        return latest_result, countdown
    except Exception as e:
        logging.error(f"Error fetching lottery results: {e}")
        return None, None

# Database Connection
def connect_db():
    return psycopg2.connect(
        dbname=DB_NAME,
        user=DB_USER,
        password=DB_PASSWORD,
        host=DB_HOST,
        port=DB_PORT
    )

def save_result(number, time):
    conn = connect_db()
    cur = conn.cursor()
    cur.execute("INSERT INTO lottery_results (winning_number, draw_time) VALUES (%s, %s)", (number, time))
    conn.commit()
    cur.close()
    conn.close()

def get_game_history():
    conn = connect_db()
    cur = conn.cursor()
    cur.execute("SELECT * FROM lottery_results ORDER BY id DESC LIMIT 10")
    history = cur.fetchall()
    cur.close()
    conn.close()
    return history

# Telegram Bot Handlers
async def start(update: Update, context: CallbackContext):
    await update.message.reply_text("Welcome to 82Lottery Bot! Use /latest to get results.")

async def latest(update: Update, context: CallbackContext):
    number, countdown = fetch_lottery_results()
    if number:
        save_result(number, countdown)
        await update.message.reply_text(f"Latest Winning Number: {number}\nNext Draw: {countdown}")
    else:
        await update.message.reply_text("Failed to fetch results. Try again later.")

async def history(update: Update, context: CallbackContext):
    history = get_game_history()
    if history:
        response = "Game History:\n"
        for record in history:
            response += f"{record[1]} - {record[2]}\n"
        await update.message.reply_text(response)
    else:
        await update.message.reply_text("No history available.")

# Auto-update function
async def auto_update():
    while True:
        number, countdown = fetch_lottery_results()
        if number:
            save_result(number, countdown)
        await asyncio.sleep(60)  # Run every minute

# Main Function
def main():
    app = Application.builder().token(TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("latest", latest))
    app.add_handler(CommandHandler("history", history))
    
    loop = asyncio.get_event_loop()
    loop.create_task(auto_update())
    
    app.run_polling()

if __name__ == "__main__":
    main()
    
