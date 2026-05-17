---
name: llm-public-opinion-analytics-assistant
description: Build and deploy LLM-powered public opinion analytics systems that aggregate hot topics from 15+ platforms, perform sentiment analysis, topic clustering, and multi-channel alert notifications
triggers:
  - how do I set up a public opinion monitoring system
  - analyze social media hot topics with LLM
  - build a sentiment analysis dashboard for trending news
  - crawl and aggregate hot search rankings from multiple platforms
  - implement topic clustering for social media analytics
  - set up automated hot topic alerts to WeChat or Telegram
  - deploy a real-time public opinion analysis assistant
  - create an LLM-based news sentiment tracker
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is a comprehensive public opinion analytics system that combines real-time data from **26 hot search lists across 15 mainstream platforms** with LLM analysis capabilities. It provides conversational hot topic queries, theme searches, topic clustering, and sentiment analysis through a web interface. The system supports:

- Multi-platform data crawling (Weibo, Bilibili, Douyin, Baidu, etc.)
- LLM-powered content analysis (even for video-based news)
- Topic clustering and sentiment classification
- Multi-channel notifications (Email, WeChat Work, Telegram)
- Keyboard shortcuts for crawler control
- Quick platform data lookup with direct link navigation

## Installation

### Prerequisites

1. **Python 3.8+** with virtual environment
2. **MySQL 5.7+** database
3. **Browser Driver** (Edge or Chrome/Chromium)
4. **LLM Model** (Huawei Pangu or OpenAI-compatible API)

### Browser Driver Setup

```bash
# 1. Check your Chrome/Edge version (e.g., 115.0.5790.102)

# 2. Download matching driver:
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# 3. Place driver in system PATH or project directory
# Linux/macOS example:
sudo mv chromedriver /usr/local/bin/
chmod +x /usr/local/bin/chromedriver

# Windows example:
# Move chromedriver.exe to C:\Windows\System32\

# 4. Verify installation
chromedriver --version
```

### Project Setup

```bash
# Clone repository
git clone https://github.com/hmmnxkl/LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant.git
cd LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant

# Create virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Database Configuration

```python
# Reference init.py for database schema
# Create database and tables in MySQL

import mysql.connector

db_config = {
    'host': 'localhost',
    'user': 'your_user',
    'password': 'your_password',
    'database': 'opinion_analytics'
}

conn = mysql.connector.connect(**db_config)
cursor = conn.cursor()

# Execute schema from init.py
# Tables: hot_search_items, analysis_results, push_tasks, etc.
```

## Configuration

### Environment Variables (.env)

```bash
# Database Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=opinion_analytics

# LLM Configuration (OpenAI-compatible)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Or use Huawei Pangu Model
# PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model

# Notification Channels
# WeChat Work
WECHAT_WEBHOOK_URL=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY
WECHAT_CORP_ID=your_corp_id
WECHAT_AGENT_ID=your_agent_id
WECHAT_SECRET=your_secret

# Telegram
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
EMAIL_RECIPIENTS=recipient1@example.com,recipient2@example.com
```

### Crawler Settings (hotsearchcrawler/settings.py)

```python
# MySQL connection for crawler
MYSQL_SETTINGS = {
    'host': 'localhost',
    'port': 3306,
    'user': 'your_user',
    'password': 'your_password',
    'database': 'opinion_analytics',
    'charset': 'utf8mb4'
}

# Optional: Platform-specific cookies
COOKIES = {
    'weibo': 'your_weibo_cookie',
    'bilibili': 'your_bilibili_cookie'
    # Add others as needed
}

# Crawler settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
ROBOTSTXT_OBEY = False
```

## Key Commands

### Start the Web Application

```bash
# Run the main application
python app.py

# Access web interface at http://localhost:5000
```

### Run Crawlers

```bash
# Test individual crawler
python runspider-test.py

# Start all crawlers (or use web interface shortcut)
python run_spiders.py

# The web interface provides keyboard shortcuts for crawler control
# See hotsearch_analysis_agent for crawler management UI
```

### Test Push Notifications

```bash
# Test notification channels
python test_push_task.py
```

## Core API Usage

### Analysis Agent (hotsearch_analysis_agent)

```python
from hotsearch_analysis_agent.analyzer import HotTopicAnalyzer
from hotsearch_analysis_agent.database import DatabaseManager

# Initialize analyzer
analyzer = HotTopicAnalyzer(
    model_name=os.getenv('MODEL_NAME', 'gpt-4'),
    api_key=os.getenv('OPENAI_API_KEY')
)

# Query hot topics by keyword
results = analyzer.query_topics(
    keyword="人工智能",
    platforms=["weibo", "zhihu", "bilibili"],
    days=7
)

# Perform sentiment analysis
sentiment = analyzer.analyze_sentiment(
    text="这次技术突破真的太厉害了！",
    context="AI technology news"
)
# Returns: {'sentiment': 'positive', 'confidence': 0.92}

# Topic clustering
db = DatabaseManager()
topics = db.fetch_hot_topics(days=3)
clusters = analyzer.cluster_topics(topics)
# Returns grouped topics by similarity

for cluster_id, items in clusters.items():
    print(f"Cluster {cluster_id}:")
    for item in items:
        print(f"  - {item['title']} ({item['platform']})")
```

### Crawler Integration

```python
from hotsearchcrawler.manager import CrawlerManager

# Initialize crawler manager
manager = CrawlerManager()

# Start specific platform crawlers
manager.start_crawlers(['weibo', 'bilibili', 'douyin'])

# Stop all crawlers
manager.stop_all()

# Get crawler status
status = manager.get_status()
# Returns: {'weibo': 'running', 'bilibili': 'stopped', ...}

# Fetch latest data
from hotsearch_analysis_agent.database import DatabaseManager

db = DatabaseManager()
latest_items = db.fetch_latest_items(platform='weibo', limit=50)

for item in latest_items:
    print(f"{item['rank']}. {item['title']} - {item['hot_value']}")
```

### Push Notification System

```python
from hotsearch_analysis_agent.notifier import MultiChannelNotifier

# Initialize notifier
notifier = MultiChannelNotifier()

# Send to WeChat Work
notifier.send_wechat_work(
    webhook_url=os.getenv('WECHAT_WEBHOOK_URL'),
    content="【热点警报】AI技术突破相关话题热度上升200%",
    mentioned_list=["@all"]
)

# Send to Telegram
notifier.send_telegram(
    bot_token=os.getenv('TELEGRAM_BOT_TOKEN'),
    chat_id=os.getenv('TELEGRAM_CHAT_ID'),
    message="🔥 Hot Topic Alert: AI breakthrough trending"
)

# Send email report
notifier.send_email(
    smtp_config={
        'host': os.getenv('SMTP_HOST'),
        'port': int(os.getenv('SMTP_PORT')),
        'user': os.getenv('SMTP_USER'),
        'password': os.getenv('SMTP_PASSWORD')
    },
    recipients=os.getenv('EMAIL_RECIPIENTS').split(','),
    subject="Public Opinion Report - 2026-05-15",
    html_content=report_html
)
```

### Creating Automated Tasks

```python
from hotsearch_analysis_agent.task_scheduler import TaskScheduler
from datetime import datetime, timedelta

scheduler = TaskScheduler()

# Schedule daily report
scheduler.add_task(
    task_id="daily_ai_report",
    query_keywords=["人工智能", "AI", "大模型"],
    platforms=["weibo", "zhihu", "bilibili"],
    notification_channels=["wechat_work", "email"],
    schedule_time="08:00",  # Daily at 8 AM
    enabled=True
)

# Schedule real-time alert (check every 30 min)
scheduler.add_task(
    task_id="urgent_tech_alert",
    query_keywords=["技术突破", "科技创新"],
    threshold={'hot_value': 100000},  # Alert if hot value > 100k
    notification_channels=["telegram", "wechat_work"],
    interval_minutes=30,
    enabled=True
)

# List all tasks
tasks = scheduler.list_tasks()
for task in tasks:
    print(f"{task['task_id']}: {task['status']}")
```

## Real-World Usage Patterns

### Pattern 1: Daily Public Opinion Report

```python
from hotsearch_analysis_agent.reporter import OpinionReporter
from datetime import datetime, timedelta

reporter = OpinionReporter()

# Generate comprehensive report
report = reporter.generate_report(
    title="关于人工智能与前沿科技的热点分析",
    keywords=["人工智能", "大模型", "算力", "芯片"],
    date_range=(datetime.now() - timedelta(days=7), datetime.now()),
    platforms=["weibo", "zhihu", "bilibili", "toutiao"],
    include_sentiment=True,
    include_clustering=True
)

# Report structure:
# - Core findings with data highlights
# - Detailed news items with URLs
# - Analysis and summary
# - Information spread characteristics

print(report['markdown'])  # Markdown formatted report
report['send_to'](['email', 'wechat_work'])
```

### Pattern 2: Real-Time Topic Monitoring

```python
from hotsearch_analysis_agent.monitor import RealTimeMonitor

monitor = RealTimeMonitor()

# Monitor specific topics
monitor.watch(
    keywords=["产品发布", "新政策"],
    platforms=["weibo", "toutiao"],
    alert_threshold={'hot_value': 50000, 'rise_rate': 2.0},
    callback=lambda topic: notifier.send_telegram(
        bot_token=os.getenv('TELEGRAM_BOT_TOKEN'),
        chat_id=os.getenv('TELEGRAM_CHAT_ID'),
        message=f"🚨 Hot topic detected: {topic['title']}"
    )
)

# Start monitoring loop
monitor.start(interval_seconds=300)  # Check every 5 minutes
```

### Pattern 3: Multi-Platform Topic Clustering

```python
from hotsearch_analysis_agent.analyzer import HotTopicAnalyzer
from hotsearch_analysis_agent.database import DatabaseManager

db = DatabaseManager()
analyzer = HotTopicAnalyzer()

# Fetch topics from last 24 hours
topics = db.fetch_hot_topics(
    platforms=["weibo", "zhihu", "bilibili", "douyin", "toutiao"],
    days=1,
    min_hot_value=10000
)

# Cluster similar topics
clusters = analyzer.cluster_topics(
    topics,
    method='semantic',  # Use LLM embeddings
    similarity_threshold=0.75
)

# Generate cluster summary
for cluster_id, items in clusters.items():
    summary = analyzer.summarize_cluster(items)
    print(f"\n=== Cluster {cluster_id}: {summary['theme']} ===")
    print(f"Platforms: {summary['platforms']}")
    print(f"Total engagement: {summary['total_hot_value']}")
    print(f"Sentiment: {summary['overall_sentiment']}")
    print("\nTop items:")
    for item in summary['top_items'][:5]:
        print(f"  - {item['title']} ({item['platform']})")
```

### Pattern 4: Video Content Analysis

```python
from hotsearch_analysis_agent.content_extractor import VideoContentExtractor

extractor = VideoContentExtractor()

# Extract content from video news (e.g., Bilibili)
video_url = "https://www.bilibili.com/video/BV13pSoBBEvX/"
content = extractor.extract(video_url)

# Content includes:
# - Video title and description
# - Auto-generated subtitles or transcripts
# - Comment sentiment analysis
# - Related metadata

analysis = analyzer.analyze_content(
    content=content['full_text'],
    content_type='video',
    platform='bilibili'
)

print(f"Topic: {analysis['main_topics']}")
print(f"Sentiment: {analysis['sentiment']}")
print(f"Key entities: {analysis['entities']}")
```

## Troubleshooting

### Browser Driver Issues

```bash
# Error: "chromedriver not found"
# Solution: Verify driver is in PATH
which chromedriver  # Linux/macOS
where chromedriver  # Windows

# Error: "session not created: This version of ChromeDriver only supports Chrome version XX"
# Solution: Download matching driver version for your browser
```

### Database Connection Errors

```python
# Error: "Access denied for user"
# Check .env file and MySQL user permissions

# Verify connection
import mysql.connector
try:
    conn = mysql.connector.connect(
        host=os.getenv('MYSQL_HOST'),
        user=os.getenv('MYSQL_USER'),
        password=os.getenv('MYSQL_PASSWORD'),
        database=os.getenv('MYSQL_DATABASE')
    )
    print("Database connected successfully")
    conn.close()
except Exception as e:
    print(f"Connection failed: {e}")
```

### LLM API Issues

```python
# Error: "Invalid API key"
# Verify OPENAI_API_KEY in .env

# Error: "Rate limit exceeded"
# Implement retry logic with backoff

from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=4, max=10))
def call_llm_with_retry(prompt):
    return analyzer.generate_response(prompt)
```

### Crawler Blocking

```python
# If crawler gets blocked, adjust settings.py:

# Increase delay between requests
DOWNLOAD_DELAY = 3

# Rotate user agents
USER_AGENT_LIST = [
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64)...',
    'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)...'
]

# Use cookies for authenticated access
COOKIES = {
    'platform_name': 'your_valid_cookie_string'
}
```

### Memory Issues with Large Datasets

```python
# Process data in batches
from hotsearch_analysis_agent.database import DatabaseManager

db = DatabaseManager()
batch_size = 1000

total_count = db.count_items(days=30)
for offset in range(0, total_count, batch_size):
    batch = db.fetch_items(limit=batch_size, offset=offset)
    process_batch(batch)
    # Clear memory
    del batch
```

## Advanced Configuration

### Custom Platform Crawlers

```python
# Add new platform in hotsearchcrawler/spiders/

from scrapy import Spider

class CustomPlatformSpider(Spider):
    name = 'custom_platform'
    
    def start_requests(self):
        yield scrapy.Request('https://platform.com/hot', self.parse)
    
    def parse(self, response):
        for item in response.css('.hot-item'):
            yield {
                'platform': 'custom_platform',
                'title': item.css('.title::text').get(),
                'hot_value': item.css('.hot::text').get(),
                'url': item.css('a::attr(href)').get(),
                'rank': item.css('.rank::text').get()
            }
```

### Custom LLM Models

```python
# Use Huawei Pangu or other local models

from hotsearch_analysis_agent.analyzer import HotTopicAnalyzer

# Initialize with local model
analyzer = HotTopicAnalyzer(
    model_type='pangu',
    model_path='/path/to/openpangu-embedded-7b-model',
    device='cuda'  # or 'cpu'
)

# Use same API for analysis
result = analyzer.analyze_sentiment(text="新技术发布会取得圆满成功")
```

This skill provides comprehensive coverage for deploying and operating an LLM-powered public opinion analytics system with multi-platform data aggregation, intelligent analysis, and automated alerting capabilities.
