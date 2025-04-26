import os
import time
import pandas as pd
import pandas_ta as ta
from dotenv import load_dotenv
from binance.client import Client

# Cargar las variables de entorno desde el archivo .env
load_dotenv()

# Credenciales de la API de Binance
API_KEY = os.getenv("API_KEY")
API_SECRET = os.getenv("API_SECRET")

# Conexión al cliente de Binance
client = Client(API_KEY, API_SECRET)

# Lista por defecto de pares de monedas
DEFAULT_PAIRS = ["BTCUSDT", "ETHUSDT", "BNBUSDT"]

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

def make_trading_decision(df):
    try:
        latest_close = df["close"].iloc[-1]
        rsi = df["RSI"].iloc[-1]
        macd = df["MACD"].iloc[-1]
        signal = df["Signal"].iloc[-1]
        if rsi < 30 and macd > signal:
            return True
        return False
    except Exception as e:
        print(f"Error al tomar decisión de trading: {str(e)}")
        return False

def analyze_pairs(pairs):
    for pair in pairs:
        print(f"Analizando {pair}...")
        df = get_historical_data(pair, "1h", 100)
        if df is not None and not df.empty:
            df = calculate_indicators(df)
            if make_trading_decision(df):
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
    except Exception as e:
        print(f"Error al realizar la compra: {str(e)}")

if __name__ == "__main__":
    print("¡El bot está listo para operar! 🚀")

    # Pedir el intervalo inicial al usuario
    try:
        interval = int(input("Ingrese el intervalo de verificación (en segundos): "))
    except ValueError:
        print("Intervalo no válido. Usando el valor por defecto de 15 segundos.")
        interval = 15

    while True:
        print("\nAnalizando lista por defecto...")
        if not analyze_pairs(DEFAULT_PAIRS):
            print("No se detectaron señales. Reiniciando ciclo...")
        print(f"Esperando {interval} segundos antes de la próxima iteración...")
        time.sleep(interval)
