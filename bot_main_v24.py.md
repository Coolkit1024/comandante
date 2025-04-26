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

def find_volatile_pairs(excluded_pairs):
    try:
        tickers = client.get_ticker()
        usdt_pairs = [ticker for ticker in tickers if ticker["symbol"].endswith("USDT") and ticker["symbol"] not in excluded_pairs]

        volatile_pairs = []
        for pair in usdt_pairs:
            high = float(pair["highPrice"])
            low = float(pair["lowPrice"])
            open_price = float(pair["openPrice"])
            close_price = float(pair["lastPrice"])
            volume = float(pair["quoteVolume"])

            if low == 0 or open_price == 0 or volume == 0:
                continue

            rango_precio = (high - low) / low
            cambio_porcentual = abs((close_price - open_price) / open_price)
            es_volatil_rango = rango_precio > 0.05
            es_volatil_cambio = cambio_porcentual > 0.03
            es_volatil_volumen = volume > 1000000

            indicadores_volatiles = sum([es_volatil_rango, es_volatil_cambio, es_volatil_volumen])
            if indicadores_volatiles >= 2:
                volatilidad_promedio = (rango_precio + cambio_porcentual) / 2
                volatile_pairs.append({"pair": pair["symbol"], "volatility": volatilidad_promedio, "volume": volume})

        sorted_pairs = sorted(volatile_pairs, key=lambda x: (x["volatility"], x["volume"]), reverse=True)
        return [pair["pair"] for pair in sorted_pairs[:3]]

    except Exception as e:
        print(f"Error al buscar pares volátiles: {str(e)}")
        return []

def list_account_balances():
    try:
        account_info = client.get_account()
        balances = account_info["balances"]
        print("Activos disponibles en tu cuenta de Binance:")
        for balance in balances:
            asset = balance["asset"]
            free = float(balance["free"])
            if free > 0:
                print(f"{asset}: {free}")
    except Exception as e:
        print(f"Error al obtener los balances de la cuenta: {str(e)}")

def analyze_pairs(pairs):
    found_signal = False
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
                found_signal = True
    return found_signal

def realizar_compra(symbol, amount):
    try:
        order = client.order_market_buy(symbol=symbol, quoteOrderQty=amount)
        print(f"🛒 Orden de COMPRA ejecutada: {order}")
    except Exception as e:
        print(f"Error al realizar la compra: {str(e)}")

if __name__ == "__main__":
    print("¡El bot está listo para operar! 🚀")
    list_account_balances()

    try:
        interval = int(input("Ingrese el intervalo de verificación (en segundos): "))
    except ValueError:
        print("Intervalo no válido. Usando el valor por defecto de 15 segundos.")
        interval = 15

    while True:
        print("\nAnalizando lista por defecto...")
        if not analyze_pairs(DEFAULT_PAIRS):
            print("No se detectaron señales en la lista por defecto. Generando lista dinámica...")
            dynamic_pairs = find_volatile_pairs(DEFAULT_PAIRS)
            if dynamic_pairs:
                print(f"Pares dinámicos seleccionados: {', '.join(dynamic_pairs)}")
                if not analyze_pairs(dynamic_pairs):
                    print("No se detectaron señales en la lista dinámica.")
            else:
                print("No se encontraron pares dinámicos.")
        print(f"Esperando {interval} segundos antes de la próxima iteración...")
        time.sleep(interval)
