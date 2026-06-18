---
name: coinmarketcap-diamonds-premium-analytics
description: CoinMarketCap Diamonds premium analytics tool for cryptocurrency trading data and blockchain insights on Windows
triggers:
  - how do I use CoinMarketCap Diamonds for crypto analytics
  - set up CoinMarketCap Diamonds premium features
  - analyze cryptocurrency trading data with Diamonds
  - get blockchain analytics from CoinMarketCap
  - configure CoinMarketCap Diamonds on Windows
  - troubleshoot CoinMarketCap Diamonds installation
  - use premium analytics features in CoinMarketCap
  - extract crypto market data with Diamonds
---

# CoinMarketCap Diamonds Premium Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

CoinMarketCap Diamonds is a Windows-based premium analytics application that provides advanced cryptocurrency trading data, blockchain analytics, and market insights. This tool unlocks professional-grade features for comprehensive crypto market analysis.

## Installation

### Windows Setup

1. **Download the Application**
   - Clone or download the repository
   - Extract files to a dedicated directory (e.g., `C:\Program Files\CoinMarketCap-Diamonds`)

2. **System Requirements**
   - Windows 10 or later (64-bit)
   - Minimum 4GB RAM
   - Internet connection for real-time data

3. **Initial Configuration**
   - Run the executable as Administrator for first-time setup
   - Configure API credentials if required

## Configuration

### Environment Variables

Set up required environment variables for API access:

```bash
# API Configuration
set CMC_API_KEY=%YOUR_API_KEY%
set CMC_API_SECRET=%YOUR_API_SECRET%
set CMC_PREMIUM_TOKEN=%YOUR_PREMIUM_TOKEN%

# Data Storage Path
set CMC_DATA_PATH=C:\CoinMarketCapData

# Cache Settings
set CMC_CACHE_SIZE=1000
set CMC_CACHE_TTL=300
```

### Configuration File

Create a `config.json` in the application directory:

```json
{
  "api": {
    "endpoint": "https://pro-api.coinmarketcap.com/v1",
    "timeout": 30000,
    "retries": 3
  },
  "analytics": {
    "updateInterval": 60,
    "historicalDataDays": 365,
    "enableRealtime": true
  },
  "display": {
    "theme": "dark",
    "chartRefreshRate": 5,
    "defaultCurrency": "USD"
  },
  "export": {
    "format": "csv",
    "outputPath": "./exports",
    "includeMetadata": true
  }
}
```

## Key Features & Usage

### Market Data Retrieval

Access real-time and historical cryptocurrency data:

```python
# Example Python integration script
import requests
import json
import os

class CoinMarketCapDiamonds:
    def __init__(self):
        self.api_key = os.getenv('CMC_API_KEY')
        self.base_url = 'https://pro-api.coinmarketcap.com/v1'
        self.headers = {
            'X-CMC_PRO_API_KEY': self.api_key,
            'Accept': 'application/json'
        }
    
    def get_latest_listings(self, limit=100, convert='USD'):
        """Fetch latest cryptocurrency listings"""
        endpoint = f'{self.base_url}/cryptocurrency/listings/latest'
        params = {
            'limit': limit,
            'convert': convert
        }
        
        response = requests.get(endpoint, headers=self.headers, params=params)
        return response.json()
    
    def get_quotes(self, symbols, convert='USD'):
        """Get quotes for specific cryptocurrencies"""
        endpoint = f'{self.base_url}/cryptocurrency/quotes/latest'
        params = {
            'symbol': ','.join(symbols),
            'convert': convert
        }
        
        response = requests.get(endpoint, headers=self.headers, params=params)
        return response.json()
    
    def get_historical_data(self, symbol, start_date, end_date):
        """Retrieve historical OHLCV data"""
        endpoint = f'{self.base_url}/cryptocurrency/ohlcv/historical'
        params = {
            'symbol': symbol,
            'time_start': start_date,
            'time_end': end_date,
            'interval': 'daily'
        }
        
        response = requests.get(endpoint, headers=self.headers, params=params)
        return response.json()

# Usage
cmc = CoinMarketCapDiamonds()
listings = cmc.get_latest_listings(limit=50)
print(json.dumps(listings, indent=2))
```

### Advanced Analytics

Perform technical analysis on cryptocurrency data:

```python
import pandas as pd
import numpy as np

class CryptoAnalytics:
    def __init__(self, data):
        self.df = pd.DataFrame(data)
    
    def calculate_moving_average(self, window=20):
        """Calculate moving average"""
        self.df['MA'] = self.df['close'].rolling(window=window).mean()
        return self.df
    
    def calculate_rsi(self, period=14):
        """Calculate Relative Strength Index"""
        delta = self.df['close'].diff()
        gain = (delta.where(delta > 0, 0)).rolling(window=period).mean()
        loss = (-delta.where(delta < 0, 0)).rolling(window=period).mean()
        
        rs = gain / loss
        rsi = 100 - (100 / (1 + rs))
        self.df['RSI'] = rsi
        return self.df
    
    def calculate_volatility(self, window=30):
        """Calculate rolling volatility"""
        returns = self.df['close'].pct_change()
        self.df['Volatility'] = returns.rolling(window=window).std() * np.sqrt(365)
        return self.df
    
    def export_analysis(self, filename):
        """Export analysis results"""
        output_path = os.getenv('CMC_DATA_PATH', './output')
        filepath = os.path.join(output_path, filename)
        self.df.to_csv(filepath, index=False)
        print(f"Analysis exported to {filepath}")

# Usage
historical_data = cmc.get_historical_data('BTC', '2025-01-01', '2026-01-01')
analytics = CryptoAnalytics(historical_data['data']['quotes'])
analytics.calculate_moving_average(window=50)
analytics.calculate_rsi(period=14)
analytics.export_analysis('btc_analysis.csv')
```

### Portfolio Tracking

Monitor and analyze cryptocurrency portfolios:

```python
class PortfolioTracker:
    def __init__(self, cmc_client):
        self.cmc = cmc_client
        self.holdings = {}
    
    def add_holding(self, symbol, amount, purchase_price):
        """Add cryptocurrency holding"""
        self.holdings[symbol] = {
            'amount': amount,
            'purchase_price': purchase_price
        }
    
    def calculate_portfolio_value(self):
        """Calculate current portfolio value"""
        symbols = list(self.holdings.keys())
        quotes = self.cmc.get_quotes(symbols)
        
        total_value = 0
        portfolio_data = []
        
        for symbol, holding in self.holdings.items():
            current_price = quotes['data'][symbol]['quote']['USD']['price']
            holding_value = holding['amount'] * current_price
            cost_basis = holding['amount'] * holding['purchase_price']
            profit_loss = holding_value - cost_basis
            profit_loss_pct = (profit_loss / cost_basis) * 100
            
            portfolio_data.append({
                'symbol': symbol,
                'amount': holding['amount'],
                'current_price': current_price,
                'value': holding_value,
                'profit_loss': profit_loss,
                'profit_loss_pct': profit_loss_pct
            })
            
            total_value += holding_value
        
        return {
            'total_value': total_value,
            'holdings': portfolio_data
        }

# Usage
tracker = PortfolioTracker(cmc)
tracker.add_holding('BTC', 0.5, 45000)
tracker.add_holding('ETH', 5.0, 3000)
tracker.add_holding('ADA', 1000, 1.2)

portfolio = tracker.calculate_portfolio_value()
print(f"Total Portfolio Value: ${portfolio['total_value']:,.2f}")
for holding in portfolio['holdings']:
    print(f"{holding['symbol']}: ${holding['value']:,.2f} ({holding['profit_loss_pct']:.2f}%)")
```

## Common Patterns

### Real-time Price Monitoring

```python
import time
from datetime import datetime

def monitor_prices(symbols, threshold_pct=5):
    """Monitor prices for significant changes"""
    cmc = CoinMarketCapDiamonds()
    last_prices = {}
    
    while True:
        try:
            quotes = cmc.get_quotes(symbols)
            
            for symbol in symbols:
                current_price = quotes['data'][symbol]['quote']['USD']['price']
                
                if symbol in last_prices:
                    change_pct = ((current_price - last_prices[symbol]) / last_prices[symbol]) * 100
                    
                    if abs(change_pct) >= threshold_pct:
                        timestamp = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
                        print(f"[{timestamp}] ALERT: {symbol} changed {change_pct:.2f}% to ${current_price:,.2f}")
                
                last_prices[symbol] = current_price
            
            time.sleep(60)  # Check every minute
            
        except Exception as e:
            print(f"Error monitoring prices: {e}")
            time.sleep(60)

# Usage
monitor_prices(['BTC', 'ETH', 'BNB'], threshold_pct=3)
```

### Data Export and Reporting

```python
def generate_market_report(symbols, output_format='csv'):
    """Generate comprehensive market report"""
    cmc = CoinMarketCapDiamonds()
    quotes = cmc.get_quotes(symbols)
    
    report_data = []
    for symbol in symbols:
        data = quotes['data'][symbol]
        quote = data['quote']['USD']
        
        report_data.append({
            'Symbol': symbol,
            'Name': data['name'],
            'Price': quote['price'],
            'Market Cap': quote['market_cap'],
            '24h Volume': quote['volume_24h'],
            '24h Change': quote['percent_change_24h'],
            '7d Change': quote['percent_change_7d'],
            'Circulating Supply': data['circulating_supply']
        })
    
    df = pd.DataFrame(report_data)
    
    output_path = os.getenv('CMC_DATA_PATH', './reports')
    timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
    
    if output_format == 'csv':
        filename = f'market_report_{timestamp}.csv'
        df.to_csv(os.path.join(output_path, filename), index=False)
    elif output_format == 'json':
        filename = f'market_report_{timestamp}.json'
        df.to_json(os.path.join(output_path, filename), orient='records', indent=2)
    
    print(f"Report generated: {filename}")
    return df

# Usage
report = generate_market_report(['BTC', 'ETH', 'XRP', 'ADA', 'SOL'], output_format='csv')
print(report.head())
```

## Troubleshooting

### API Connection Issues

**Problem**: Unable to connect to CoinMarketCap API

**Solutions**:
- Verify API key is correctly set: `echo %CMC_API_KEY%`
- Check internet connectivity
- Confirm API endpoint is accessible
- Review rate limits (free tier: 333 requests/day)

### Data Export Failures

**Problem**: Export operations failing

**Solutions**:
```python
# Ensure output directory exists
import os

output_path = os.getenv('CMC_DATA_PATH', './output')
os.makedirs(output_path, exist_ok=True)

# Add error handling to exports
try:
    df.to_csv(filepath, index=False)
    print(f"Successfully exported to {filepath}")
except PermissionError:
    print(f"Permission denied: {filepath}")
except Exception as e:
    print(f"Export failed: {e}")
```

### Memory Issues with Large Datasets

**Problem**: Application crashes with large historical data requests

**Solutions**:
```python
# Process data in chunks
def fetch_historical_chunked(symbol, start_date, end_date, chunk_days=30):
    """Fetch historical data in chunks to avoid memory issues"""
    from datetime import datetime, timedelta
    
    start = datetime.strptime(start_date, '%Y-%m-%d')
    end = datetime.strptime(end_date, '%Y-%m-%d')
    
    all_data = []
    current = start
    
    while current < end:
        chunk_end = min(current + timedelta(days=chunk_days), end)
        
        chunk_data = cmc.get_historical_data(
            symbol,
            current.strftime('%Y-%m-%d'),
            chunk_end.strftime('%Y-%m-%d')
        )
        
        all_data.extend(chunk_data['data']['quotes'])
        current = chunk_end
    
    return all_data
```

### Rate Limiting

**Problem**: API rate limit exceeded

**Solutions**:
```python
import time
from functools import wraps

def rate_limit(calls_per_minute=30):
    """Decorator to enforce rate limiting"""
    min_interval = 60.0 / calls_per_minute
    last_called = [0.0]
    
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            left_to_wait = min_interval - elapsed
            
            if left_to_wait > 0:
                time.sleep(left_to_wait)
            
            ret = func(*args, **kwargs)
            last_called[0] = time.time()
            return ret
        return wrapper
    return decorator

# Apply to API calls
@rate_limit(calls_per_minute=20)
def safe_get_quotes(symbols):
    return cmc.get_quotes(symbols)
```

## Best Practices

1. **Always use environment variables** for API credentials
2. **Implement error handling** for all API calls
3. **Cache frequently accessed data** to minimize API calls
4. **Use batch requests** when fetching multiple assets
5. **Monitor rate limits** and implement backoff strategies
6. **Validate data** before performing analytics
7. **Export results regularly** to avoid data loss
8. **Keep historical data local** for offline analysis
