# Test-Bot-1
app.py
import logging
import datetime
import requests
import pandas as pd
from telegram import Update
from telegram.ext import Updater, CommandHandler, CallbackContext, JobQueue

# Telegram Token (hier einfügen)
TOKEN = "DEIN_TELEGRAM_BOT_TOKEN"

# Standard-Coin
current_coin = "BTCUSDT"

# Log Setup
logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO)

# Funktion: Daten von Bitget laden
def load_bitget(symbol, limit=6):
    try:
        url = f"https://api.bitget.com/api/v2/market/candles?symbol={symbol.lower()}&granularity=1min&limit={limit}"
        r = requests.get(url, timeout=5)
        data = r.json()
        if 'data' not in data or not data['data']:
            return None
        df = pd.DataFrame(data['data'], columns=["timestamp", "open", "high", "low", "close", "volume"])
        df = df.astype({"open": float, "high": float, "low": float, "close": float, "volume": float})
        df = df[::-1].reset_index(drop=True)
        return df
    except Exception as e:
        return None

# Load fallback (hier könntest du TradingView-API ergänzen)
def load_data(symbol):
    df = load_bitget(symbol)
    if df is not None:
        return df
    return None  # Für Demo nur Bitget

# Strategie: Signal berechnen
def strategy_leverage_target(symbol):
    df = load_data(symbol)
    if df is None or df.empty:
        return None, None
    close_prices = df['close'].tolist()
    current_price = close_prices[-1]
    threshold = 0.004 * current_price  # 0.40 %

    big_moves = 0
    direction_votes = 0
    for i in range(1, len(close_prices)):
        diff = close_prices[i] - close_prices[i-1]
        if abs(diff) >= threshold:
            big_moves += 1
            direction_votes += 1 if diff > 0 else -1

    confidence = min(95, 50 + big_moves * 10)
    if direction_votes > 0:
        return "LONG", confidence
    else:
        return "SHORT", confidence

# Telegram Command: /start
def start(update: Update, context: CallbackContext):
    update.message.reply_text(
        "Hallo! Ich sende dir 5 Sekunden vor Ablauf der Minute ein Signal.\n"
        "Nutze /setcoin <Coin> um die Kryptowährung zu wechseln, z.B. /setcoin BTCUSDT"
    )

# Telegram Command: /setcoin
def set_coin(update: Update, context: CallbackContext):
    global current_coin
    if len(context.args) != 1:
        update.message.reply_text("Bitte gib einen Coin an, z.B. /setcoin SOLUSDT")
        return
    coin = context.args[0].upper()
    current_coin = coin
    update.message.reply_text(f"Coin geändert auf {coin}")

# Job, der jede Minute geprüft wird, 5 Sekunden vor Ablauf
def job_callback(context: CallbackContext):
    now = datetime.datetime.utcnow()
    if now.second == 55:
        signal, confidence = strategy_leverage_target(current_coin)
        if signal is None:
            msg = f"Keine Daten für {current_coin} verfügbar."
        else:
            msg = f"Signal für {current_coin}: {signal} mit ca. {confidence}% Wahrscheinlichkeit auf ±0,40% Bewegung bei 5x Hebel."
        context.bot.send_message(chat_id=context.job.context, text=msg)

def main():
    updater = Updater(TOKEN)
    dp = updater.dispatcher

    dp.add_handler(CommandHandler("start", start))
    dp.add_handler(CommandHandler("setcoin", set_coin))

    # Starte den Bot
    updater.start_polling()

    # Füge Job für Signal alle Sekunde hinzu (prüft nur bei Sekunde 55)
    jq = updater.job_queue
    # Hier deine Telegram Chat-ID eintragen (kann man per /start oder getUpdates ermitteln)
    chat_id = "DEINE_CHAT_ID"
    jq.run_repeating(job_callback, interval=1, first=0, context=chat_id)

    updater.idle()

if __name__ == '__main__':
    main()
