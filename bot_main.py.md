import os
import time
import pandas as pd
import pandas_ta as ta
from dotenv import load_dotenv
from binance.client import Client
import requests

# Cargar las variables de entorno desde el archivo .env
load_dotenv()

# Credenciales de la API de Binance
API_KEY = os.getenv("API_KEY")
API_SECRET = os.getenv("API_SECRET")

# Configuración de Telegram
TELEGRAM_BOT_TOKEN = os.getenv("TELEGRAM_BOT_TOKEN")
TELEGRAM_CHAT_ID = os.getenv("TELEGRAM_CHAT_ID")

# Conexión al cliente de Binance
client = Client(API_KEY, API_SECRET)

# Lista por defecto de pares de monedas
DEFAULT_PAIRS = ["BTCUSDT", "ETHUSDT", "BNBUSDT"]

def enviar_mensaje_telegram(mensaje):
    if TELEGRAM_BOT_TOKEN and TELEGRAM_CHAT_ID:
        url = f"https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/sendMessage"
        payload = {"chat_id": TELEGRAM_CHAT_ID, "text": mensaje}
        try:
            response = requests.post(url, json=payload)
            response.raise_for_status()
        except requests.exceptions.RequestException as e:
            print(f"Error al enviar mensaje a Telegram: {e}")

def get_historical_data(symbol, interval, lookback):
    try:
        klines = client.get_klines(symbol=symbol, interval=interval, limit=lookback)
        df = pd.DataFrame(klines, columns=[
            "timestamp", "open", "high", "low", "close", "volume",
            "close_time", "quote_asset_volume", "number_of_trades",
            "taker_buy_base", "taker_buy_quote", "ignore"
        ])
        df["open"] = df["open"].astype(float)
        df["high"] = df["high"].astype(float)
        df["low"] = df["low"].astype(float)
        df["close"] = df["close"].astype(float)
        df["volume"] = df["volume"].astype(float)
        return df
    except Exception as e:
        print(f"Error al obtener datos históricos para {symbol}: {str(e)}")
        return None

def calculate_indicators(df):
    try:
        df["RSI"] = ta.rsi(df["close"], length=14)
        macd = ta.macd(df["close"], fast=12, slow=26, signal=9)
        df["MACD"] = macd["MACD_12_26_9"]
        df["Signal"] = macd["MACDs_12_26_9"]
        bb = ta.bbands(df["close"], length=20)
        df["BB_upper"] = bb["BBU_20_2.0"]
        df["BB_middle"] = bb["BBM_20_2.0"]
        df["BB_lower"] = bb["BBL_20_2.0"]
        return df
    except Exception as e:
        print(f"Error al calcular indicadores: {str(e)}")
        return None

def calculate_fibonacci_levels(df):
    try:
        high = df["high"].iloc[-1]
        low = df["low"].iloc[-1]
        diff = high - low
        levels = {
            "0%": high,
            "23.6%": high - 0.236 * diff,
            "38.2%": high - 0.382 * diff,
            "50%": high - 0.5 * diff,
            "61.8%": high - 0.618 * diff,
            "100%": low
        }
        return levels
    except Exception as e:
        print(f"Error al calcular niveles de Fibonacci: {str(e)}")
        return None

def make_trading_decision(df, fib_levels):
    try:
        latest_close = df["close"].iloc[-1]
        rsi = df["RSI"].iloc[-1]
        macd = df["MACD"].iloc[-1]
        signal = df["Signal"].iloc[-1]
        if (
            rsi < 30 and
            macd > signal and
            latest_close > fib_levels["61.8%"]
        ):
            return True
        return False
    except Exception as e:
        print(f"Error al tomar decisión de trading: {str(e)}")
        return False

def find_volatile_pairs(excluded_pairs):
    try:
        tickers = client.get_ticker()
        usdt_pairs = [ticker for ticker in tickers if ticker["symbol"].endswith("USDT") and ticker["symbol"] not in excluded_pairs]
        volatility = []
        for pair in usdt_pairs:
            high = float(pair["highPrice"])
            low = float(pair["lowPrice"])
            # Validación para evitar división por cero
            if low == 0:
                continue
            change = (high - low) / low
            volume = float(pair["quoteVolume"])
            volatility.append({"pair": pair["symbol"], "volatility": change, "volume": volume})
        # Ordenar por volatilidad y volumen
        sorted_pairs = sorted(volatility, key=lambda x: (x["volatility"], x["volume"]), reverse=True)
        return [pair["pair"] for pair in sorted_pairs[:3]]
    except Exception as e:
        print(f"Error al buscar pares volátiles: {str(e)}")
        return []

def analyze_pairs(pairs):
    for pair in pairs:
        print(f"Analizando {pair}...")
        df = get_historical_data(pair, "1h", 100)
        if df is not None and not df.empty:
            df = calculate_indicators(df)
            fib_levels = calculate_fibonacci_levels(df)
            if make_trading_decision(df, fib_levels):
                print(f"📈 Señal de entrada detectada en {pair}.")
                user_input = input(f"¿Deseas operar en {pair}? (s/n): ")
                if user_input.lower() == "s":
                    amount = float(input("Introduce el monto para operar (mínimo $5): "))
                    if amount >= 5:
                        realizar_compra(pair, amount)
                    else:
                        print("El monto debe ser mayor o igual a $5.")
                return True
    return False

def realizar_compra(symbol, amount):
    try:
        order = client.order_market_buy(symbol=symbol, quoteOrderQty=amount)
        print(f"🛒 Orden de COMPRA ejecutada: {order}")
        enviar_mensaje_telegram(f"🛒 Orden de COMPRA ejecutada: {order}")
    except Exception as e:
        print(f"Error al realizar la compra: {str(e)}")

if __name__ == "__main__":
    print("¡El bot está listo para operar! 🚀")
    interval = 15  # Intervalo de verificación en segundos
    while True:
        print("\nAnalizando lista por defecto...")
        if not analyze_pairs(DEFAULT_PAIRS):
            print("\nNo se detectaron señales en la lista por defecto. Buscando pares dinámicos...")
            dynamic_pairs = find_volatile_pairs(DEFAULT_PAIRS)
            print(f"Pares dinámicos seleccionados: {dynamic_pairs}")
            if not analyze_pairs(dynamic_pairs):
                print("No se detectaron señales en pares dinámicos. Reiniciando ciclo...")
        print(f"Esperando {interval} segundos antes de la próxima iteración...")
        time.sleep(interval)
