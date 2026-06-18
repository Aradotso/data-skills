---
name: coinmarketcap-diamonds-premium-analytics
description: CoinMarketCap Diamonds premium analytics and trading tools for cryptocurrency market data and blockchain insights
triggers:
  - how do I use CoinMarketCap Diamonds premium features
  - analyze crypto market data with CoinMarketCap Diamonds
  - set up CoinMarketCap Diamonds analytics tools
  - access premium trading analytics for cryptocurrency
  - configure CoinMarketCap Diamonds for market tracking
  - use blockchain analytics with CoinMarketCap premium
  - extract cryptocurrency market insights with Diamonds
  - integrate CoinMarketCap premium API features
---

# CoinMarketCap Diamonds Premium Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

CoinMarketCap Diamonds is a premium analytics and trading toolkit for cryptocurrency market data analysis. It provides enhanced access to CoinMarketCap's professional features including advanced market analytics, real-time trading signals, blockchain metrics, and portfolio tracking capabilities for Windows environments.

**Key Capabilities:**
- Premium market data access and analytics
- Real-time cryptocurrency price tracking
- Advanced trading indicators and signals
- Portfolio management and tracking
- Blockchain metrics and on-chain analysis
- Historical data extraction
- Custom alerts and notifications

## Installation

### Windows Setup

1. Download the premium build from the official repository
2. Extract to your preferred installation directory (e.g., `C:\Program Files\CoinMarketCap-Diamonds`)
3. Run the installer as administrator
4. Configure your API credentials

### Environment Configuration

Set up required environment variables:

```bash
# CoinMarketCap API credentials
set CMC_API_KEY=your_api_key_here
set CMC_PRO_TIER=premium

# Data storage location
set CMC_DATA_DIR=C:\Users\YourUser\Documents\CoinMarketCap\data

# Analytics settings
set CMC_CACHE_ENABLED=true
set CMC_UPDATE_INTERVAL=60
```

For PowerShell:

```powershell
$env:CMC_API_KEY = $env:CMC_API_KEY
$env:CMC_PRO_TIER = "premium"
$env:CMC_DATA_DIR = "C:\Users\YourUser\Documents\CoinMarketCap\data"
```

## Core Features

### Market Data Access

Access real-time and historical cryptocurrency market data:

```python
import coinmarketcap_diamonds as cmc

# Initialize with premium credentials
client = cmc.Client(api_key=os.environ['CMC_API_KEY'])

# Get latest market data for top cryptocurrencies
market_data = client.get_latest_listings(
    limit=100,
    sort='market_cap',
    convert='USD'
)

for coin in market_data['data']:
    print(f"{coin['name']}: ${coin['quote']['USD']['price']:.2f}")
    print(f"  24h Change: {coin['quote']['USD']['percent_change_24h']:.2f}%")
    print(f"  Market Cap: ${coin['quote']['USD']['market_cap']:,.0f}")
```

### Historical Data Extraction

Extract historical price and volume data:

```python
from datetime import datetime, timedelta

# Get historical data for Bitcoin
end_date = datetime.now()
start_date = end_date - timedelta(days=30)

btc_history = client.get_historical_data(
    symbol='BTC',
    start_date=start_date.strftime('%Y-%m-%d'),
    end_date=end_date.strftime('%Y-%m-%d'),
    interval='daily'
)

# Process historical data
for datapoint in btc_history['data']:
    timestamp = datapoint['timestamp']
    price = datapoint['quote']['USD']['price']
    volume = datapoint['quote']['USD']['volume_24h']
    print(f"{timestamp}: ${price:.2f} (Vol: ${volume:,.0f})")
```

### Advanced Analytics

Perform technical analysis and generate trading signals:

```python
# Calculate technical indicators
indicators = client.analytics.calculate_indicators(
    symbol='ETH',
    period=14,
    indicators=['RSI', 'MACD', 'BB']  # Relative Strength Index, MACD, Bollinger Bands
)

rsi = indicators['RSI'][-1]
macd = indicators['MACD'][-1]

print(f"Ethereum Technical Analysis:")
print(f"  RSI(14): {rsi:.2f}")
print(f"  MACD: {macd['value']:.2f}")
print(f"  Signal: {macd['signal']:.2f}")

# Generate trading signal
if rsi < 30:
    print("  Signal: OVERSOLD - Potential BUY")
elif rsi > 70:
    print("  Signal: OVERBOUGHT - Potential SELL")
```

### Portfolio Tracking

Track and analyze cryptocurrency portfolios:

```python
# Create a portfolio
portfolio = client.portfolio.create(name="My Crypto Portfolio")

# Add holdings
portfolio.add_holding(
    symbol='BTC',
    amount=0.5,
    purchase_price=45000.0,
    purchase_date='2024-01-15'
)

portfolio.add_holding(
    symbol='ETH',
    amount=10.0,
    purchase_price=2800.0,
    purchase_date='2024-02-01'
)

# Get portfolio summary
summary = portfolio.get_summary()
print(f"Total Value: ${summary['total_value']:,.2f}")
print(f"Total Cost: ${summary['total_cost']:,.2f}")
print(f"P&L: ${summary['profit_loss']:,.2f} ({summary['roi']:.2f}%)")

# Detailed breakdown
for holding in summary['holdings']:
    print(f"\n{holding['symbol']}:")
    print(f"  Amount: {holding['amount']}")
    print(f"  Current Value: ${holding['current_value']:,.2f}")
    print(f"  P&L: ${holding['profit_loss']:,.2f}")
```

### Real-Time Alerts

Set up custom price alerts and notifications:

```python
# Create price alert
alert = client.alerts.create_price_alert(
    symbol='BTC',
    condition='above',
    threshold=50000.0,
    notification_method='email'
)

# Volume spike alert
volume_alert = client.alerts.create_volume_alert(
    symbol='ETH',
    spike_percentage=200.0,  # Alert if volume spikes 200% above average
    lookback_hours=24
)

# Check active alerts
active_alerts = client.alerts.get_active()
for alert in active_alerts:
    print(f"Alert: {alert['symbol']} {alert['condition']} ${alert['threshold']}")
```

### Blockchain Metrics

Access on-chain analytics and blockchain data:

```python
# Get blockchain metrics
btc_metrics = client.blockchain.get_metrics(symbol='BTC')

print("Bitcoin On-Chain Metrics:")
print(f"  Active Addresses: {btc_metrics['active_addresses']:,}")
print(f"  Transaction Count: {btc_metrics['transaction_count']:,}")
print(f"  Network Hash Rate: {btc_metrics['hash_rate']:.2f} TH/s")
print(f"  Mining Difficulty: {btc_metrics['difficulty']:,}")

# Wallet distribution analysis
distribution = client.blockchain.get_wallet_distribution(symbol='BTC')
print("\nBTC Holder Distribution:")
for tier in distribution['tiers']:
    print(f"  {tier['range']}: {tier['percentage']:.2f}% ({tier['count']:,} wallets)")
```

## Configuration

### Application Settings

Create a configuration file at `%APPDATA%\CoinMarketCap\config.json`:

```json
{
  "api": {
    "key": "${CMC_API_KEY}",
    "tier": "premium",
    "rate_limit": 333,
    "timeout": 30
  },
  "data": {
    "cache_enabled": true,
    "cache_ttl": 300,
    "storage_path": "${CMC_DATA_DIR}",
    "auto_backup": true
  },
  "analytics": {
    "default_indicators": ["RSI", "MACD", "SMA", "EMA"],
    "default_period": 14,
    "default_interval": "daily"
  },
  "alerts": {
    "email": "your_email@example.com",
    "push_enabled": false,
    "check_interval": 60
  },
  "ui": {
    "theme": "dark",
    "default_currency": "USD",
    "decimal_places": 2
  }
}
```

### Advanced Configuration

```python
# Configure client with custom settings
config = {
    'api_key': os.environ['CMC_API_KEY'],
    'sandbox_mode': False,
    'rate_limit': 333,
    'retry_attempts': 3,
    'retry_delay': 1.0,
    'cache_enabled': True,
    'cache_ttl': 300
}

client = cmc.Client(**config)

# Set custom request headers
client.set_headers({
    'Accept-Encoding': 'gzip, deflate',
    'User-Agent': 'CoinMarketCap-Diamonds/1.0'
})
```

## Common Patterns

### Batch Data Processing

```python
# Process multiple cryptocurrencies efficiently
symbols = ['BTC', 'ETH', 'BNB', 'ADA', 'SOL', 'XRP', 'DOT', 'DOGE']

# Batch request for efficiency
batch_data = client.get_quotes_latest(symbols=symbols)

results = []
for symbol in symbols:
    data = batch_data['data'][symbol]
    results.append({
        'symbol': symbol,
        'price': data['quote']['USD']['price'],
        'volume_24h': data['quote']['USD']['volume_24h'],
        'market_cap': data['quote']['USD']['market_cap']
    })

# Sort by market cap
results.sort(key=lambda x: x['market_cap'], reverse=True)

for coin in results:
    print(f"{coin['symbol']}: ${coin['price']:.2f} | MCap: ${coin['market_cap']:,.0f}")
```

### Data Export

```python
import csv
import json

# Export market data to CSV
def export_to_csv(data, filename):
    with open(filename, 'w', newline='') as f:
        writer = csv.DictWriter(f, fieldnames=['symbol', 'price', 'change_24h', 'volume', 'market_cap'])
        writer.writeheader()
        
        for coin in data:
            writer.writerow({
                'symbol': coin['symbol'],
                'price': coin['quote']['USD']['price'],
                'change_24h': coin['quote']['USD']['percent_change_24h'],
                'volume': coin['quote']['USD']['volume_24h'],
                'market_cap': coin['quote']['USD']['market_cap']
            })

market_data = client.get_latest_listings(limit=100)
export_to_csv(market_data['data'], 'market_snapshot.csv')
```

### Automated Reporting

```python
from datetime import datetime

def generate_daily_report():
    report_date = datetime.now().strftime('%Y-%m-%d')
    
    # Get top movers
    gainers = client.get_top_gainers(limit=10, time_period='24h')
    losers = client.get_top_losers(limit=10, time_period='24h')
    
    # Get market overview
    global_metrics = client.get_global_metrics()
    
    report = f"""
    Cryptocurrency Market Report - {report_date}
    ===============================================
    
    Global Market Metrics:
    - Total Market Cap: ${global_metrics['quote']['USD']['total_market_cap']:,.0f}
    - 24h Volume: ${global_metrics['quote']['USD']['total_volume_24h']:,.0f}
    - BTC Dominance: {global_metrics['btc_dominance']:.2f}%
    
    Top Gainers (24h):
    """
    
    for i, coin in enumerate(gainers['data'], 1):
        report += f"\n  {i}. {coin['symbol']}: +{coin['quote']['USD']['percent_change_24h']:.2f}%"
    
    report += "\n\n  Top Losers (24h):"
    for i, coin in enumerate(losers['data'], 1):
        report += f"\n  {i}. {coin['symbol']}: {coin['quote']['USD']['percent_change_24h']:.2f}%"
    
    return report

# Generate and save report
report = generate_daily_report()
print(report)

with open(f"report_{datetime.now().strftime('%Y%m%d')}.txt", 'w') as f:
    f.write(report)
```

## Troubleshooting

### API Rate Limits

```python
from time import sleep

def rate_limited_request(func, *args, **kwargs):
    """Handle rate limiting with exponential backoff"""
    max_retries = 5
    base_delay = 1.0
    
    for attempt in range(max_retries):
        try:
            return func(*args, **kwargs)
        except cmc.RateLimitError as e:
            if attempt == max_retries - 1:
                raise
            delay = base_delay * (2 ** attempt)
            print(f"Rate limit hit. Retrying in {delay}s...")
            sleep(delay)
```

### Authentication Issues

```python
# Verify API key
try:
    client = cmc.Client(api_key=os.environ.get('CMC_API_KEY'))
    status = client.verify_api_key()
    print(f"API Key Valid: {status['valid']}")
    print(f"Tier: {status['tier']}")
    print(f"Credits Remaining: {status['credits_remaining']}")
except cmc.AuthenticationError as e:
    print(f"Authentication failed: {e}")
    print("Please check your CMC_API_KEY environment variable")
```

### Data Caching

```python
# Clear cache if data seems stale
client.cache.clear()

# Disable cache for specific requests
fresh_data = client.get_latest_listings(use_cache=False)

# Check cache status
cache_info = client.cache.info()
print(f"Cache size: {cache_info['size']} entries")
print(f"Cache hits: {cache_info['hits']}")
print(f"Cache misses: {cache_info['misses']}")
```

### Connection Timeouts

```python
# Increase timeout for slow connections
client = cmc.Client(
    api_key=os.environ['CMC_API_KEY'],
    timeout=60  # 60 seconds
)

# Retry logic for network issues
def robust_request(client, method, *args, **kwargs):
    max_attempts = 3
    for attempt in range(max_attempts):
        try:
            return getattr(client, method)(*args, **kwargs)
        except cmc.NetworkError as e:
            if attempt == max_attempts - 1:
                raise
            print(f"Network error: {e}. Retrying...")
            sleep(2)
```

## Best Practices

1. **Always use environment variables** for API keys and sensitive data
2. **Enable caching** for frequently accessed data to reduce API calls
3. **Implement rate limiting** to stay within API quota
4. **Handle errors gracefully** with try-except blocks and retries
5. **Batch requests** when possible to minimize API calls
6. **Export data regularly** for backup and offline analysis
7. **Monitor API usage** to avoid exceeding credit limits
8. **Use appropriate intervals** for real-time vs. historical data needs
