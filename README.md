# TRADING-API
BINANCE API

import ccxt
import time
import numpy as np

# Binance API keys
api_key = 'YOUR_API_KEY'
api_secret = 'YOUR_API_SECRET'

# Initialize Binance client
exchange = ccxt.binance({
    'apiKey': api_key,
    'secret': api_secret,
})

# Trading pairs
trading_pairs = ['BTC/USDT', 'ETH/USDT']

# ICT Strategy Parameters
premium_discount_threshold = 0.5
order_block_threshold = 0.02
trade_duration = 3 * 60 * 60  # 3 hours in seconds
leverage = 10
risk_per_trade = 0.02  # 2% risk per trade

def get_fibonacci_levels(high, low):
    """Calculate Fibonacci levels for premium and discount zones."""
    diff = high - low
    return {
        'premium': high - diff * premium_discount_threshold,
        'discount': low + diff * premium_discount_threshold
    }

def identify_order_blocks(candles):
    """Identify order blocks in the given candles."""
    order_blocks = []
    for i in range(1, len(candles) - 1):
        if candles[i]['high'] > candles[i-1]['high'] and candles[i]['high'] > candles[i+1]['high']:
            order_blocks.append({'type': 'bearish', 'price': candles[i]['high']})
        if candles[i]['low'] < candles[i-1]['low'] and candles[i]['low'] < candles[i+1]['low']:
            order_blocks.append({'type': 'bullish', 'price': candles[i]['low']})
    return order_blocks

def check_trade_conditions(candles, order_blocks, fib_levels):
    """Check if trade conditions are met."""
    last_candle = candles[-1]
    for block in order_blocks:
        if block['type'] == 'bullish' and last_candle['close'] < fib_levels['discount'] and last_candle['close'] > block['price']:
            return 'buy'
        if block['type'] == 'bearish' and last_candle['close'] > fib_levels['premium'] and last_candle['close'] < block['price']:
            return 'sell'
    return None

def calculate_position_size(balance, risk_per_trade, stop_loss_distance, leverage):
    """Calculate the position size based on risk management."""
    risk_amount = balance * risk_per_trade
    position_size = (risk_amount / stop_loss_distance) * leverage
    return position_size

def execute_trade(pair, side, amount):
    """Execute a trade on Binance."""
    order = exchange.create_order(pair, 'market', side, amount)
    return order

def main():
    while True:
        for pair in trading_pairs:
            candles = exchange.fetch_ohlcv(pair, timeframe='15m', limit=50)
            high = max([candle[2] for candle in candles])
            low = min([candle[3] for candle in candles])
            fib_levels = get_fibonacci_levels(high, low)
            order_blocks = identify_order_blocks(candles)
            trade_signal = check_trade_conditions(candles, order_blocks, fib_levels)
            
            if trade_signal:
                balance = exchange.fetch_balance()['total']['USDT']
                stop_loss_distance = abs(fib_levels['premium'] - fib_levels['discount']) * order_block_threshold
                position_size = calculate_position_size(balance, risk_per_trade, stop_loss_distance, leverage)
                execute_trade(pair, trade_signal, position_size)
                time.sleep(trade_duration)
        
        time.sleep(60)  # Check every minute

if __name__ == "__main__":
    main()
