---
name: hyperliquid-leaderboard-analytics
description: Terminal analytics dashboard for tracking Hyperliquid DEX top traders by PnL, ROI, and performance metrics
triggers:
  - analyze hyperliquid leaderboard data
  - track top traders on hyperliquid
  - export hyperliquid trader statistics
  - filter hyperliquid leaderboard by roi
  - compare hyperliquid trader performance
  - visualize hyperliquid trading metrics
  - build terminal dashboard for crypto trading analytics
  - scrape hyperliquid public leaderboard
---

# Hyperliquid Leaderboard Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

Terminal-based analytics dashboard for Hyperliquid perpetual DEX leaderboard data. Track top traders, filter by performance metrics (PnL, ROI, win rate, Sharpe ratio), visualize equity curves, and export data for quantitative analysis. Built with Textual for rich terminal UI and uses the public Hyperliquid info API (read-only, no authentication required).

## Installation

```bash
git clone https://github.com/lassepaladin/hyperliquid-leaderboard-analytics.git
cd hyperliquid-leaderboard-analytics
pip install -r requirements.txt
```

**Dependencies** (from requirements.txt):
- `textual>=0.47.0` - Terminal UI framework
- `rich>=13.7.0` - Rich text formatting
- `httpx>=0.26.0` - Async HTTP client
- `toml>=0.10.2` - Configuration parsing

## CLI Usage

### Basic Commands

```bash
# Launch live dashboard with public API data
python main.py

# Run with demo/offline dataset (no API calls)
python main.py --demo

# As Python module
python -m hl_leaderboard_analytics

# Specify custom config
python main.py --config ~/.custom-hl-config.toml
```

### Keyboard Navigation

Once launched, the TUI accepts these keybindings:

| Key | Action |
|-----|--------|
| `1`-`6` | Switch tabs (Board/Detail/Filters/Compare/Export/Settings) |
| `/` | Filter current table |
| `s` | Cycle sort column |
| `w` | Cycle time window (7d/30d/90d/all) |
| `Enter` | Open trader detail view |
| `c` | Compare selected trader |
| `e` | Export current view |
| `t` | Cycle theme |
| `q` | Quit |

## Configuration

Config file location: `~/.hl-leaderboard/config.toml`

```toml
[network]
api_url = "https://api.hyperliquid.xyz"
timeout = 30
max_retries = 3

[board]
default_window = "90d"        # 7d | 30d | 90d | all
default_sort = "roi"          # roi | volume | win_rate | sharpe | profit_factor | drawdown
page_size = 100
refresh_interval = 300        # seconds

[export]
format = "csv"                # csv | json | markdown
out_dir = "./exports"
include_timestamp = true

[ui]
theme = "monokai"             # monokai | nord | dracula | gruvbox
show_help = true
vim_bindings = true
```

## Python API Usage

### Fetching Leaderboard Data

```python
from hl_leaderboard_analytics.core.api_client import HyperliquidAPIClient
import asyncio

async def fetch_leaderboard():
    client = HyperliquidAPIClient(base_url="https://api.hyperliquid.xyz")
    
    # Get full leaderboard
    leaderboard = await client.get_leaderboard()
    
    # Filter by time window
    traders_90d = await client.get_leaderboard(window="90d")
    
    # Get specific trader details
    trader = await client.get_trader_detail("0x7f3a...c4e1")
    
    return leaderboard

# Run async
leaderboard_data = asyncio.run(fetch_leaderboard())
```

### Data Models

```python
from hl_leaderboard_analytics.core.models import Trader, TraderMetrics

# Trader model structure
trader = Trader(
    address="0x7f3a...c4e1",
    alias="quant_kappa",
    account_value=125000.50,
    pnl_90d=51600.20,
    roi_90d=0.4128,  # 412.8%
    roi_30d=0.0582,
    volume_90d=4810000.0,
    win_rate=0.713,
    sharpe_ratio=1.82,
    profit_factor=2.31,
    max_drawdown=0.15,
    num_trades=847,
    avg_trade_size=5682.0
)

# Access metrics
print(f"ROI (90d): {trader.roi_90d * 100:.1f}%")
print(f"Win Rate: {trader.win_rate * 100:.1f}%")
print(f"Sharpe: {trader.sharpe_ratio:.2f}")
```

### Filtering and Ranking

```python
from hl_leaderboard_analytics.core.analytics import LeaderboardAnalytics

analytics = LeaderboardAnalytics(leaderboard_data)

# Filter by ROI threshold
high_roi_traders = analytics.filter_by_roi(min_roi=0.50, window="90d")

# Filter by asset
btc_traders = analytics.filter_by_asset("BTC")

# Filter by leverage range
low_lev_traders = analytics.filter_by_leverage(min_lev=1, max_lev=5)

# Rank by multiple criteria
top_traders = analytics.rank_by(
    sort_key="sharpe_ratio",
    ascending=False,
    min_trades=100,
    min_account_value=10000
)

# Get percentile statistics
stats = analytics.get_percentile_stats(metric="roi_90d")
print(f"P50 ROI: {stats['p50'] * 100:.1f}%")
print(f"P90 ROI: {stats['p90'] * 100:.1f}%")
```

### Exporting Data

```python
from hl_leaderboard_analytics.core.export import DataExporter

exporter = DataExporter(output_dir="./exports")

# Export to CSV
exporter.to_csv(
    traders=top_traders,
    filename="top_traders_90d.csv",
    columns=["alias", "address", "roi_90d", "sharpe_ratio", "win_rate"]
)

# Export to JSON
exporter.to_json(
    traders=high_roi_traders,
    filename="high_roi_traders.json",
    pretty=True
)

# Export to Markdown table
exporter.to_markdown(
    traders=btc_traders,
    filename="btc_specialists.md",
    title="BTC Perp Specialists"
)
```

### Period Comparison

```python
from hl_leaderboard_analytics.core.analytics import ComparisonAnalyzer

analyzer = ComparisonAnalyzer()

# Compare 30d vs 90d performance
comparison = analyzer.compare_periods(
    trader_address="0x7f3a...c4e1",
    period_1="30d",
    period_2="90d"
)

print(f"ROI delta: {comparison.roi_delta * 100:.1f}%")
print(f"Rank change: {comparison.rank_change}")
print(f"Volume change: {comparison.volume_delta:.0f}")

# Identify top movers
movers = analyzer.get_top_movers(period="24h", limit=10)
for trader, delta in movers:
    print(f"{trader.alias}: {delta * 100:+.1f}%")
```

### Building Custom TUI Components

```python
from textual.app import App, ComposeResult
from textual.widgets import DataTable, Header, Footer
from hl_leaderboard_analytics.core.api_client import HyperliquidAPIClient

class CustomLeaderboardApp(App):
    CSS = """
    DataTable {
        height: 100%;
    }
    """
    
    def compose(self) -> ComposeResult:
        yield Header()
        yield DataTable()
        yield Footer()
    
    async def on_mount(self) -> None:
        table = self.query_one(DataTable)
        table.add_columns("Rank", "Alias", "ROI (90d)", "Sharpe")
        
        # Fetch and populate data
        client = HyperliquidAPIClient()
        leaderboard = await client.get_leaderboard(window="90d")
        
        for i, trader in enumerate(leaderboard[:50], 1):
            table.add_row(
                str(i),
                trader.alias,
                f"{trader.roi_90d * 100:.1f}%",
                f"{trader.sharpe_ratio:.2f}"
            )

if __name__ == "__main__":
    app = CustomLeaderboardApp()
    app.run()
```

## Common Patterns

### 1. Track Consistent Performers

```python
from hl_leaderboard_analytics.core.analytics import ConsistencyAnalyzer

analyzer = ConsistencyAnalyzer(leaderboard_data)

# Find traders profitable across all periods
consistent = analyzer.find_consistent_performers(
    periods=["7d", "30d", "90d"],
    min_roi=0.05,  # 5% minimum per period
    min_sharpe=1.0
)

# Identify one-hit wonders (high short-term, low long-term)
one_hit = analyzer.find_one_hit_wonders(
    short_period="7d",
    long_period="90d",
    short_roi_min=0.50,
    long_roi_max=0.10
)
```

### 2. Asset Specialization Analysis

```python
from hl_leaderboard_analytics.core.analytics import AssetAnalyzer

asset_analyzer = AssetAnalyzer(leaderboard_data)

# Get per-asset PnL breakdown for a trader
breakdown = asset_analyzer.get_asset_breakdown("0x7f3a...c4e1")
for asset, metrics in breakdown.items():
    print(f"{asset}: PnL ${metrics['pnl']:,.0f}, ROI {metrics['roi']*100:.1f}%")

# Find specialists (>80% PnL from single asset)
btc_specialists = asset_analyzer.find_specialists(asset="BTC", min_concentration=0.80)
eth_specialists = asset_analyzer.find_specialists(asset="ETH", min_concentration=0.80)
```

### 3. Real-time Dashboard Updates

```python
from textual.reactive import reactive
from textual.widgets import Static
import asyncio

class LiveMetricsWidget(Static):
    total_traders = reactive(0)
    avg_roi = reactive(0.0)
    
    async def on_mount(self) -> None:
        self.update_metrics_loop()
    
    async def update_metrics_loop(self) -> None:
        client = HyperliquidAPIClient()
        while True:
            leaderboard = await client.get_leaderboard()
            self.total_traders = len(leaderboard)
            self.avg_roi = sum(t.roi_90d for t in leaderboard) / len(leaderboard)
            await asyncio.sleep(300)  # 5 min refresh
    
    def render(self) -> str:
        return f"Traders: {self.total_traders} | Avg ROI: {self.avg_roi*100:.1f}%"
```

### 4. Batch Export for Analysis

```python
import pandas as pd

def export_for_analysis(leaderboard_data, output_dir="./analysis"):
    # Convert to DataFrame
    df = pd.DataFrame([
        {
            "address": t.address,
            "alias": t.alias,
            "roi_90d": t.roi_90d,
            "roi_30d": t.roi_30d,
            "sharpe": t.sharpe_ratio,
            "win_rate": t.win_rate,
            "volume": t.volume_90d,
            "max_dd": t.max_drawdown
        }
        for t in leaderboard_data
    ])
    
    # Export multiple formats
    df.to_csv(f"{output_dir}/leaderboard_full.csv", index=False)
    df.to_parquet(f"{output_dir}/leaderboard_full.parquet")
    
    # Export top performers only
    top_50 = df.nlargest(50, "roi_90d")
    top_50.to_excel(f"{output_dir}/top_50_roi.xlsx", index=False)
    
    # Summary statistics
    stats = df.describe()
    stats.to_csv(f"{output_dir}/summary_stats.csv")
    
    return df
```

## Troubleshooting

### API Connection Issues

```python
from hl_leaderboard_analytics.core.api_client import HyperliquidAPIClient
import logging

# Enable debug logging
logging.basicConfig(level=logging.DEBUG)

client = HyperliquidAPIClient(
    base_url="https://api.hyperliquid.xyz",
    timeout=60,  # Increase timeout
    max_retries=5
)

try:
    leaderboard = await client.get_leaderboard()
except Exception as e:
    print(f"API Error: {e}")
    # Fallback to demo mode
    from hl_leaderboard_analytics.core.mock_data import load_demo_data
    leaderboard = load_demo_data()
```

### Memory Issues with Large Datasets

```python
# Use generators for large exports
def export_large_dataset_streaming(traders, output_file):
    import csv
    
    with open(output_file, 'w', newline='') as f:
        writer = csv.DictWriter(f, fieldnames=['address', 'roi_90d', 'sharpe'])
        writer.writeheader()
        
        # Stream write instead of loading all into memory
        for trader in traders:
            writer.writerow({
                'address': trader.address,
                'roi_90d': trader.roi_90d,
                'sharpe': trader.sharpe_ratio
            })
```

### TUI Rendering Issues

```bash
# Force 256 color mode
export TERM=xterm-256color
python main.py

# Disable Unicode (Windows compatibility)
python main.py --no-unicode

# Run in headless mode (export only, no TUI)
python -m hl_leaderboard_analytics.cli export --format csv --output ./data.csv
```

### Rate Limiting

```python
import asyncio
from hl_leaderboard_analytics.core.api_client import HyperliquidAPIClient

client = HyperliquidAPIClient()

# Add delay between requests
async def fetch_with_rate_limit(addresses):
    results = []
    for addr in addresses:
        trader = await client.get_trader_detail(addr)
        results.append(trader)
        await asyncio.sleep(0.5)  # 500ms delay
    return results
```

## Advanced Use Cases

### Correlation Analysis

```python
import numpy as np
from scipy.stats import pearsonr

def analyze_metric_correlation(leaderboard_data):
    roi_values = [t.roi_90d for t in leaderboard_data]
    sharpe_values = [t.sharpe_ratio for t in leaderboard_data]
    win_rate_values = [t.win_rate for t in leaderboard_data]
    
    # ROI vs Sharpe correlation
    corr_roi_sharpe, p_value = pearsonr(roi_values, sharpe_values)
    print(f"ROI-Sharpe correlation: {corr_roi_sharpe:.3f} (p={p_value:.4f})")
    
    # ROI vs Win Rate correlation
    corr_roi_wr, p_value = pearsonr(roi_values, win_rate_values)
    print(f"ROI-WinRate correlation: {corr_roi_wr:.3f} (p={p_value:.4f})")
```

### Time Series Analysis

```python
from hl_leaderboard_analytics.core.time_series import EquityCurveAnalyzer

analyzer = EquityCurveAnalyzer()

# Get historical snapshots
snapshots = await client.get_historical_snapshots(
    trader_address="0x7f3a...c4e1",
    period="90d",
    interval="1d"
)

# Calculate metrics
max_dd = analyzer.calculate_max_drawdown(snapshots)
cagr = analyzer.calculate_cagr(snapshots)
volatility = analyzer.calculate_volatility(snapshots)

print(f"Max Drawdown: {max_dd*100:.1f}%")
print(f"CAGR: {cagr*100:.1f}%")
print(f"Volatility: {volatility*100:.1f}%")
```
