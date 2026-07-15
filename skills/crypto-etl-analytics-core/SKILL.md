---
name: crypto-etl-analytics-core
description: End-to-end Python and MySQL data engineering pipeline with OOP analytics library for cryptocurrency price data
triggers:
  - "build a crypto ETL pipeline"
  - "analyze cryptocurrency price data"
  - "create OHLCV candlestick data pipeline"
  - "calculate crypto returns and volatility"
  - "set up MySQL crypto database schema"
  - "implement object-oriented financial analytics"
  - "extract and transform cryptocurrency candles"
  - "compute drawdowns and rolling statistics"
---

# crypto-etl-analytics-core

> Skill by [ara.so](https://ara.so) — Data Skills collection

A foundational data engineering pipeline and object-oriented analytics library for cryptocurrency market data. This project demonstrates pure Python financial calculations (without Pandas/NumPy) using MySQL as the data warehouse, OHLCV candlestick objects, and computed metrics like returns, drawdowns, and volatility.

## What It Does

- **ETL Pipeline**: Extracts crypto price data, transforms via staging tables, loads into strict MySQL schema
- **Data Validation**: Enforces integrity constraints (no negative volumes, valid wicks, chronological order)
- **OOP Design**: Encapsulates OHLCV data in `Candle` objects with built-in validation
- **Financial Analytics**: Calculates simple/log returns, drawdowns, rolling volatility without external math libraries
- **Visualization**: Generates matplotlib charts for price action and computed metrics

## Installation

```bash
git clone https://github.com/Jad-srifi/crypto-etl-analytics-core.git
cd crypto-etl-analytics-core
pip install mysql-connector-python matplotlib
```

## Database Setup

1. **Execute the schema creation script** in your MySQL instance:

```bash
mysql -u your_user -p < 00_table_creation.sql
```

2. **Create environment variables** for database credentials:

```bash
export DB_HOST=localhost
export DB_USER=your_user
export DB_PASSWORD=your_password
export DB_NAME=crypto_db
```

## Core Architecture

### 1. Database Schema

The `crypto_1d_candles` table enforces strict data integrity:

```sql
CREATE TABLE crypto_1d_candles (
    candle_id INT AUTO_INCREMENT PRIMARY KEY,
    asset_id INT NOT NULL,
    candle_date DATE NOT NULL,
    open_price DECIMAL(18,8) NOT NULL CHECK (open_price > 0),
    high_price DECIMAL(18,8) NOT NULL CHECK (high_price > 0),
    low_price DECIMAL(18,8) NOT NULL CHECK (low_price > 0),
    close_price DECIMAL(18,8) NOT NULL CHECK (close_price > 0),
    volume DECIMAL(20,8) NOT NULL CHECK (volume >= 0),
    CONSTRAINT valid_wick CHECK (high_price >= open_price 
                                 AND high_price >= close_price 
                                 AND low_price <= open_price 
                                 AND low_price <= close_price),
    UNIQUE KEY unique_asset_date (asset_id, candle_date)
);
```

### 2. Candle Object (OOP Model)

The core data structure encapsulating OHLCV data:

```python
class Candle:
    def __init__(self, date, open_price, high, low, close, volume):
        self.date = date
        self.open = float(open_price)
        self.high = float(high)
        self.low = float(low)
        self.close = float(close)
        self.volume = float(volume)
        
        # Validation
        if not (self.high >= self.open and self.high >= self.close):
            raise ValueError(f"Invalid wick on {date}: high must be >= open and close")
        if not (self.low <= self.open and self.low <= self.close):
            raise ValueError(f"Invalid wick on {date}: low must be <= open and close")
        if self.volume < 0:
            raise ValueError(f"Negative volume on {date}")
    
    def __repr__(self):
        return f"Candle({self.date}, O:{self.open}, H:{self.high}, L:{self.low}, C:{self.close}, V:{self.volume})"
```

### 3. ETL Pipeline

**Extract from MySQL using CTEs:**

```python
import mysql.connector
import os

def extract_candles(asset_name):
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST', 'localhost'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    cursor = conn.cursor()
    
    query = """
    WITH asset_filter AS (
        SELECT asset_id FROM assets WHERE asset_name = %s
    )
    SELECT candle_date, open_price, high_price, low_price, close_price, volume
    FROM crypto_1d_candles
    WHERE asset_id = (SELECT asset_id FROM asset_filter)
    ORDER BY candle_date ASC
    """
    
    cursor.execute(query, (asset_name,))
    rows = cursor.fetchall()
    
    candles = []
    for row in rows:
        candle = Candle(
            date=row[0],
            open_price=row[1],
            high=row[2],
            low=row[3],
            close=row[4],
            volume=row[5]
        )
        candles.append(candle)
    
    cursor.close()
    conn.close()
    
    return candles
```

## Analytics Library

### Returns Calculation

**Simple Returns:**

```python
def calculate_simple_returns(candles):
    """Calculate period-over-period simple returns."""
    if len(candles) < 2:
        return []
    
    returns = []
    for i in range(1, len(candles)):
        prev_close = candles[i-1].close
        curr_close = candles[i].close
        simple_return = (curr_close - prev_close) / prev_close
        returns.append(simple_return)
    
    return returns
```

**Log Returns:**

```python
import math

def calculate_log_returns(candles):
    """Calculate natural logarithm returns for statistical analysis."""
    if len(candles) < 2:
        return []
    
    log_returns = []
    for i in range(1, len(candles)):
        prev_close = candles[i-1].close
        curr_close = candles[i].close
        log_return = math.log(curr_close / prev_close)
        log_returns.append(log_return)
    
    return log_returns
```

**Cumulative Returns:**

```python
def calculate_cumulative_returns(returns):
    """Calculate cumulative returns from simple return series."""
    cumulative = []
    cum_value = 1.0
    
    for ret in returns:
        cum_value *= (1 + ret)
        cumulative.append(cum_value - 1)  # Convert back to return format
    
    return cumulative
```

### Drawdown Calculation

```python
def calculate_drawdowns(candles):
    """Calculate drawdown from rolling peak for each candle."""
    if not candles:
        return []
    
    drawdowns = []
    running_peak = candles[0].close
    
    for candle in candles:
        if candle.close > running_peak:
            running_peak = candle.close
        
        drawdown = (candle.close - running_peak) / running_peak
        drawdowns.append(drawdown)
    
    return drawdowns

def max_drawdown(drawdowns):
    """Find the maximum drawdown from a drawdown series."""
    if not drawdowns:
        return 0.0
    return min(drawdowns)  # Most negative value
```

### Rolling Volatility

```python
def calculate_rolling_volatility(log_returns, window=30):
    """Calculate rolling standard deviation of log returns."""
    if len(log_returns) < window:
        return []
    
    volatilities = []
    
    for i in range(window - 1, len(log_returns)):
        window_returns = log_returns[i - window + 1 : i + 1]
        
        # Calculate mean
        mean_return = sum(window_returns) / len(window_returns)
        
        # Calculate variance
        variance = sum((r - mean_return) ** 2 for r in window_returns) / (len(window_returns) - 1)
        
        # Standard deviation
        volatility = math.sqrt(variance)
        volatilities.append(volatility)
    
    return volatilities
```

## Visualization

```python
import matplotlib.pyplot as plt

def plot_price_and_returns(candles, returns):
    """Generate dual-axis plot for price and returns."""
    dates = [c.date for c in candles]
    closes = [c.close for c in candles]
    
    fig, (ax1, ax2) = plt.subplots(2, 1, figsize=(12, 8), sharex=True)
    
    # Price chart
    ax1.plot(dates, closes, label='Close Price', color='blue', linewidth=1.5)
    ax1.set_ylabel('Price (USD)')
    ax1.set_title('Cryptocurrency Price Analysis')
    ax1.legend()
    ax1.grid(True, alpha=0.3)
    
    # Returns chart
    return_dates = dates[1:]  # Offset by 1 since returns start at t=1
    ax2.plot(return_dates, returns, label='Daily Returns', color='green', linewidth=1)
    ax2.axhline(y=0, color='red', linestyle='--', alpha=0.5)
    ax2.set_ylabel('Return')
    ax2.set_xlabel('Date')
    ax2.legend()
    ax2.grid(True, alpha=0.3)
    
    plt.tight_layout()
    plt.savefig('crypto_analysis.png', dpi=300)
    plt.show()
```

## Complete Usage Example

```python
import os
from candle import Candle
from etl import extract_candles
from analytics import (
    calculate_log_returns, 
    calculate_cumulative_returns,
    calculate_drawdowns,
    calculate_rolling_volatility
)
from visualizations import plot_price_and_returns

def main():
    # Extract data
    asset = 'BTC'
    candles = extract_candles(asset)
    print(f"Extracted {len(candles)} candles for {asset}")
    
    # Calculate metrics
    log_returns = calculate_log_returns(candles)
    cumulative_returns = calculate_cumulative_returns(log_returns)
    drawdowns = calculate_drawdowns(candles)
    volatility = calculate_rolling_volatility(log_returns, window=30)
    
    # Print summary statistics
    print(f"Total Return: {cumulative_returns[-1]:.2%}")
    print(f"Max Drawdown: {max(drawdowns):.2%}")
    print(f"Avg 30-day Volatility: {sum(volatility)/len(volatility):.4f}")
    
    # Generate visualizations
    plot_price_and_returns(candles, log_returns)

if __name__ == '__main__':
    main()
```

## Data Validation Patterns

**Chronological Order Check:**

```python
def validate_chronological_order(candles):
    """Ensure candles are in ascending date order."""
    for i in range(1, len(candles)):
        if candles[i].date <= candles[i-1].date:
            raise ValueError(f"Candles not in chronological order at index {i}")
    return True
```

**Completeness Check:**

```python
from datetime import timedelta

def check_missing_dates(candles):
    """Identify gaps in daily data."""
    if len(candles) < 2:
        return []
    
    missing = []
    for i in range(1, len(candles)):
        date_diff = (candles[i].date - candles[i-1].date).days
        if date_diff > 1:
            missing.append((candles[i-1].date, candles[i].date, date_diff - 1))
    
    return missing
```

## Common Patterns

### Staging Table Pattern

Use a staging table to dynamically map asset names to IDs:

```sql
CREATE TEMPORARY TABLE staging_candles (
    asset_name VARCHAR(50),
    candle_date DATE,
    open_price DECIMAL(18,8),
    high_price DECIMAL(18,8),
    low_price DECIMAL(18,8),
    close_price DECIMAL(18,8),
    volume DECIMAL(20,8)
);

INSERT INTO crypto_1d_candles (asset_id, candle_date, open_price, high_price, low_price, close_price, volume)
SELECT a.asset_id, s.candle_date, s.open_price, s.high_price, s.low_price, s.close_price, s.volume
FROM staging_candles s
JOIN assets a ON s.asset_name = a.asset_name;
```

### Batch Processing Pattern

```python
def process_multiple_assets(asset_list):
    """Process analytics for multiple cryptocurrencies."""
    results = {}
    
    for asset in asset_list:
        try:
            candles = extract_candles(asset)
            returns = calculate_log_returns(candles)
            cum_returns = calculate_cumulative_returns(returns)
            
            results[asset] = {
                'total_return': cum_returns[-1] if cum_returns else 0,
                'num_candles': len(candles)
            }
        except Exception as e:
            print(f"Error processing {asset}: {e}")
            results[asset] = {'error': str(e)}
    
    return results
```

## Troubleshooting

**Issue: MySQL connection fails**
- Verify environment variables are set: `echo $DB_USER`
- Check MySQL service is running: `systemctl status mysql`
- Test connection: `mysql -u $DB_USER -p$DB_PASSWORD`

**Issue: Invalid wick constraint violation**
- Check raw data before insertion: `high >= max(open, close)` and `low <= min(open, close)`
- Use staging table to filter invalid rows before final insert

**Issue: Returns calculation produces NaN**
- Ensure candles list has at least 2 elements
- Check for zero or negative prices in data
- Validate chronological order with `validate_chronological_order()`

**Issue: Plotting fails with index mismatch**
- Remember returns series is length n-1 for n candles
- Use `dates[1:]` when plotting returns alongside prices

**Issue: Slow queries for large datasets**
- Add indexes: `CREATE INDEX idx_asset_date ON crypto_1d_candles(asset_id, candle_date)`
- Use date range filters in WHERE clauses
- Consider materialized views for frequently accessed aggregations

## Next Steps

This pure-Python implementation demonstrates core concepts. For production scale:

- Migrate to **Pandas** for vectorized operations
- Use **NumPy** for optimized mathematical computations
- Implement **async** database connections for concurrent asset processing
- Add **caching layer** (Redis) for frequently accessed candle series
- Integrate **Airflow/Prefect** for orchestrated ETL scheduling
