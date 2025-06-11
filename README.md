# Test-Bot-1
app.py
import streamlit as st
import datetime
import requests
from streamlit_autorefresh import st_autorefresh
from price_loaders.tradingview import load_asset_price

# =================== Konfiguration ===================

st.set_page_config(page_title="Krypto Leverage Bot", layout="centered")
st.title("🚀 Krypto 5x Leverage Signal Bot")
st.caption("Ziel: ±0,40 % Bewegung = ±2 % Gewinn bei 5x Hebel")

# Auswahl: deine Coins (RESOLVUSDT inkludiert, falls bei TradingView verfügbar)
coins = ["BTCUSDT", "ETHUSDT", "SOLUSDT", "ADAUSDT", "RESOLVUSDT"]
coin = st.selectbox("📈 Welche Kryptowährung willst du analysieren?", coins)

# Auto-Refresh alle 5 Sekunden
st_autorefresh(interval=5000, key="refresh")

# ============== Daten von Bitget laden ===================
def load_bitget(symbol, limit=6):
    try:
        url = f"https://api.bitget.com/api/v2/market/candles?symbol={symbol}&granularity=1min&limit={limit}"
        response = requests.get(url, timeout=5)
        data = response.json()
        if 'data' not in data or not data['data']:
            return None
        candles = data['data']
        import pandas as pd
        df = pd.DataFrame(candles, columns=["timestamp", "open", "high", "low", "close", "volume"])
        df = df.astype({"open": float, "high": float, "low": float, "close": float, "volume": float})
        df = df[::-1].reset_index(drop=True)  # Reihenfolge umdrehen: älteste zuerst
        return df
    except Exception as e:
        return None

# ============== Datenquelle: Bitget oder TradingView ===================
def load_data(symbol):
    # Bitget verwendet Symbol evtl. mit "-USDT" oder "_USDT" - hier anpassen falls nötig
    # Beispiel: "btcusdt" -> "btcusdt" (klein), Bitget API ist case sensitive
    bitget_symbol = symbol.lower()
    df = load_bitget(bitget_symbol)
    if df is not None:
        return df
    # Fallback auf TradingView
    try:
        df = load_asset_price(f"BINANCE:{symbol}", 6, "1m", None)
        return df
    except:
        return None

# ============== Strategielogik: Leverage Bewegung =================

def strategy_leverage_target(symbol):
    df = load_data(symbol)
    if df is None or df.empty:
        st.warning("Keine Daten empfangen.")
        return None, None

    close_prices = df['close'].tolist()
    current_price = close_prices[-1]
    threshold = 0.004 * current_price  # 0.40 %

    # Bewegung der letzten Minuten analysieren
    big_moves = 0
    direction_votes = 0
    for i in range(1, len(close_prices)):
        diff = close_prices[i] - close_prices[i - 1]
        if abs(diff) >= threshold:
            big_moves += 1
            direction_votes += 1 if diff > 0 else -1

    confidence = min(95, 50 + big_moves * 10)

    if direction_votes > 0:
        return 1, confidence  # LONG
    else:
        return 2, confidence  # SHORT

# ============== Zeitsteuerung für Prognose ==============

now = datetime.datetime.utcnow()
sec = now.second

if 55 <= sec <= 59:
    signal, confidence = strategy_leverage_target(coin)
    if signal is None:
        st.warning("⚠️ Keine Prognose verfügbar.")
    elif signal == 1:
        st.success(f"📈 LONG → ~{confidence}% Chance auf +0,40 % Bewegung!")
    else:
        st.error(f"📉 SHORT → ~{confidence}% Chance auf -0,40 % Bewegung!")
else:
    wait = 55 - sec if sec < 55 else 60 - sec + 55
    st.info(f"⏳ Warte auf Signal in {wait}s...")
