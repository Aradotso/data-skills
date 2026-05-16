---
name: roblox-mm2-analytics-toolkit
description: Analytics and inventory tracking toolkit for Roblox Murder Mystery 2 gameplay optimization
triggers:
  - "help me analyze my Murder Mystery 2 inventory"
  - "track my MM2 knife skins and stats"
  - "set up Roblox MM2 analytics dashboard"
  - "optimize my Murder Mystery 2 collection"
  - "analyze MM2 gameplay patterns and strategy"
  - "configure MM2 inventory tracker"
  - "export my Murder Mystery 2 statistics"
  - "integrate MM2 analytics with AI predictions"
---

# Roblox MM2 Analytics Toolkit

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Roblox MM2 Analytics Toolkit is a comprehensive data analysis and inventory management system for Murder Mystery 2 players. It provides:

- **Inventory tracking** for knife skins, gamepasses, and collectibles
- **Analytics dashboard** with visualization of gameplay statistics
- **Strategy analysis** using pattern recognition and AI predictions
- **Data export** in multiple formats (CSV, JSON)
- **Multi-platform support** for desktop, web, and mobile interfaces

## Installation

### Quick Install (Automated)

```bash
chmod +x setup.sh
./setup.sh --install
```

### Manual Installation

```bash
# Clone the repository
git clone https://github.com/8015238355/mm2-analytics-dashboard-2026.git
cd mm2-analytics-dashboard-2026

# Install Node.js dependencies
npm install

# Install Python dependencies
python3 -m pip install -r requirements.txt
```

### Environment Setup

Create a `.env` file in the project root:

```bash
API_OPENAI_KEY=${OPENAI_API_KEY}
API_CLAUDE_KEY=${CLAUDE_API_KEY}
DATA_DIRECTORY=./data/collections
ANALYTICS_INTERVAL=300
ENABLE_LIVE_TRACKING=true
LOG_LEVEL=INFO
```

## Configuration

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

### Language Configuration

Set language in `config/settings.json`:

```json
{
  "language": "en",
  "supported_languages": ["en", "es", "fr", "de", "ja", "pt"],
  "ui_theme": "dark",
  "responsive_mode": true
}
```

## CLI Commands

### Basic Analytics

```bash
# Run analytics with default profile
python3 main.py --mode analytics

# Specify profile and export format
python3 main.py --mode analytics \
    --profile mystery_solver_01 \
    --export statistics_2026.json \
    --format json \
    --verbose
```

### Inventory Management

```bash
# Scan inventory
python3 main.py --mode inventory \
    --scan-all \
    --output inventory_snapshot.csv

# Filter by rarity
python3 main.py --mode inventory \
    --filter rarity=legendary \
    --export-format json
```

### Strategy Analysis

```bash
# Analyze gameplay patterns
python3 main.py --mode strategy \
    --analyze-sessions 50 \
    --ai-predictions \
    --export strategy_report.pdf

# Debug mode with verbose logging
python3 main.py --mode strategy \
    --log-level DEBUG \
    --verbose
```

## Python API Usage

### Inventory Tracker

```python
from mm2_toolkit import InventoryTracker

# Initialize tracker
tracker = InventoryTracker(
    profile="mystery_solver_01",
    data_dir="./data/collections"
)

# Scan inventory
inventory = tracker.scan_inventory()
print(f"Total items: {len(inventory)}")

# Filter by category and rarity
legendary_knives = tracker.filter(
    category="knife_skins",
    rarity=["legendary", "ancient"]
)

# Export to CSV
tracker.export(
    items=legendary_knives,
    format="csv",
    output="legendary_knives.csv"
)
```

### Analytics Engine

```python
from mm2_toolkit import AnalyticsEngine
import datetime

# Initialize analytics
analytics = AnalyticsEngine(
    profile="mystery_solver_01",
    refresh_rate=30
)

# Get gameplay statistics
stats = analytics.get_statistics(
    start_date=datetime.datetime(2026, 1, 1),
    end_date=datetime.datetime.now()
)

print(f"Win rate: {stats['win_rate']:.2%}")
print(f"Favorite role: {stats['favorite_role']}")
print(f"Total games: {stats['total_games']}")

# Analyze by role
sheriff_stats = analytics.get_role_statistics(role="sheriff")
print(f"Sheriff win rate: {sheriff_stats['win_rate']:.2%}")
```

### Strategy Module

```python
from mm2_toolkit import StrategyAnalyzer

# Initialize analyzer
analyzer = StrategyAnalyzer(
    profile="mystery_solver_01",
    ai_enabled=True
)

# Analyze patterns
patterns = analyzer.analyze_gameplay_patterns(
    sessions=50,
    roles=["sheriff", "murderer", "innocent"]
)

# Get AI recommendations
recommendations = analyzer.get_ai_recommendations(
    current_role="sheriff",
    map_name="factory"
)

for rec in recommendations:
    print(f"Strategy: {rec['name']}")
    print(f"Confidence: {rec['confidence']:.2%}")
    print(f"Description: {rec['description']}\n")
```

### Data Visualization

```python
from mm2_toolkit import DataVisualizer
import matplotlib.pyplot as plt

# Initialize visualizer
viz = DataVisualizer(theme="dark")

# Create win rate chart
win_data = analytics.get_win_rate_by_role()
viz.create_bar_chart(
    data=win_data,
    title="Win Rate by Role",
    output="win_rate_chart.png"
)

# Create inventory distribution
inventory_data = tracker.get_rarity_distribution()
viz.create_pie_chart(
    data=inventory_data,
    title="Inventory Rarity Distribution",
    output="inventory_pie.png"
)

# Interactive dashboard
viz.launch_dashboard(
    port=8080,
    auto_refresh=True
)
```

## Common Patterns

### Session Analysis Workflow

```python
from mm2_toolkit import AnalyticsEngine, StrategyAnalyzer, DataVisualizer

def analyze_recent_sessions(profile_name, session_count=20):
    """Analyze recent gameplay sessions and generate report"""
    
    # Initialize components
    analytics = AnalyticsEngine(profile=profile_name)
    strategy = StrategyAnalyzer(profile=profile_name, ai_enabled=True)
    viz = DataVisualizer(theme="dark")
    
    # Get session data
    sessions = analytics.get_recent_sessions(count=session_count)
    
    # Calculate metrics
    metrics = {
        'total_sessions': len(sessions),
        'win_rate': analytics.calculate_win_rate(sessions),
        'avg_duration': analytics.calculate_avg_duration(sessions),
        'role_distribution': analytics.get_role_distribution(sessions)
    }
    
    # Get AI insights
    insights = strategy.get_ai_recommendations(
        sessions=sessions,
        focus_areas=['positioning', 'timing', 'deduction']
    )
    
    # Generate visualizations
    viz.create_session_timeline(
        sessions=sessions,
        output="session_timeline.png"
    )
    
    return {
        'metrics': metrics,
        'insights': insights,
        'visualizations': ['session_timeline.png']
    }

# Usage
report = analyze_recent_sessions("mystery_solver_01", session_count=30)
print(f"Win Rate: {report['metrics']['win_rate']:.2%}")
```

### Inventory Optimization

```python
from mm2_toolkit import InventoryTracker, TradeRecommender

def optimize_inventory(profile_name):
    """Find duplicate items and suggest trades"""
    
    tracker = InventoryTracker(profile=profile_name)
    recommender = TradeRecommender(api_key="${TRADE_API_KEY}")
    
    # Scan inventory
    inventory = tracker.scan_inventory()
    
    # Find duplicates
    duplicates = tracker.find_duplicates()
    
    # Get trade recommendations
    recommendations = recommender.suggest_trades(
        inventory=inventory,
        duplicates=duplicates,
        target_rarities=["legendary", "ancient"]
    )
    
    # Export report
    tracker.export_trade_report(
        recommendations=recommendations,
        output="trade_suggestions.json"
    )
    
    return recommendations

# Usage
trades = optimize_inventory("mystery_solver_01")
for trade in trades:
    print(f"Trade: {trade['offer']} -> {trade['receive']}")
    print(f"Value gain: {trade['value_increase']}")
```

### AI-Powered Predictions

```python
from mm2_toolkit import PredictiveModel
import os

def predict_match_outcome(current_role, map_name, players):
    """Use AI to predict match outcome"""
    
    model = PredictiveModel(
        openai_key=os.getenv('API_OPENAI_KEY'),
        claude_key=os.getenv('API_CLAUDE_KEY')
    )
    
    # Prepare context
    context = {
        'role': current_role,
        'map': map_name,
        'player_count': len(players),
        'player_skill_levels': [p['skill_level'] for p in players]
    }
    
    # Get predictions
    prediction = model.predict_outcome(context)
    
    # Get strategy suggestions
    strategy = model.suggest_strategy(
        context=context,
        prediction=prediction
    )
    
    return {
        'win_probability': prediction['win_probability'],
        'key_factors': prediction['factors'],
        'recommended_strategy': strategy
    }

# Usage
result = predict_match_outcome(
    current_role="sheriff",
    map_name="factory",
    players=[{'skill_level': 0.75}, {'skill_level': 0.60}]
)
print(f"Win probability: {result['win_probability']:.2%}")
print(f"Strategy: {result['recommended_strategy']}")
```

## Troubleshooting

### Data Directory Issues

```python
# Ensure data directory exists
import os
from pathlib import Path

data_dir = Path(os.getenv('DATA_DIRECTORY', './data/collections'))
data_dir.mkdir(parents=True, exist_ok=True)

# Verify permissions
if not os.access(data_dir, os.W_OK):
    raise PermissionError(f"Cannot write to {data_dir}")
```

### API Connection Errors

```python
from mm2_toolkit import APIHealthCheck

# Check API connectivity
health = APIHealthCheck()

if not health.check_openai():
    print("Warning: OpenAI API unavailable")
    # Fallback to local analysis

if not health.check_claude():
    print("Warning: Claude API unavailable")
    # Use alternative prediction model
```

### Memory Optimization for Large Inventories

```python
from mm2_toolkit import InventoryTracker

# Use pagination for large inventories
tracker = InventoryTracker(
    profile="mystery_solver_01",
    pagination=True,
    page_size=100
)

# Process in batches
for batch in tracker.scan_inventory_batched():
    # Process each batch
    filtered = [item for item in batch if item['rarity'] == 'legendary']
    tracker.export(filtered, format="json", append=True)
```

### Export Format Issues

```python
from mm2_toolkit import DataExporter

# Handle export errors gracefully
exporter = DataExporter()

try:
    exporter.export(
        data=inventory_data,
        format="csv",
        output="inventory.csv"
    )
except ValueError as e:
    print(f"CSV export failed: {e}")
    # Fallback to JSON
    exporter.export(
        data=inventory_data,
        format="json",
        output="inventory.json"
    )
```

## Integration Examples

### With Web Dashboard

```python
from mm2_toolkit import WebDashboard
from flask import Flask

app = Flask(__name__)
dashboard = WebDashboard(app)

# Configure dashboard
dashboard.configure(
    port=8080,
    auto_refresh=30,
    theme="dark",
    profile="mystery_solver_01"
)

# Launch
if __name__ == "__main__":
    dashboard.launch(debug=False)
```

### Scheduled Analysis

```python
import schedule
import time
from mm2_toolkit import AnalyticsEngine

def scheduled_analysis():
    """Run analytics every hour"""
    analytics = AnalyticsEngine(profile="mystery_solver_01")
    stats = analytics.get_statistics()
    analytics.export(stats, format="json", output="hourly_stats.json")
    print(f"Analysis complete: {time.strftime('%Y-%m-%d %H:%M:%S')}")

# Schedule task
schedule.every().hour.do(scheduled_analysis)

while True:
    schedule.run_pending()
    time.sleep(60)
```
