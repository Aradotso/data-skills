---
name: llm-public-opinion-analytics-assistant
description: Chinese public opinion analysis system integrating 26 hot search lists from 15 platforms with LLM-powered sentiment analysis, topic clustering, and multi-channel alerts
triggers:
  - set up public opinion monitoring system
  - analyze hot search trends across Chinese platforms
  - configure multi-platform sentiment analysis
  - build real-time hot topic crawler
  - implement intelligent opinion analytics with LLM
  - create automated hot topic push notifications
  - deploy Chinese social media monitoring tool
  - integrate weibo douyin bilibili hot search tracking
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is a comprehensive Chinese public opinion analysis system that crawls and analyzes hot search lists from 26 different rankings across 15 major Chinese platforms (Weibo, Douyin, Bilibili, Zhihu, Baidu, etc.). It combines web scraping with LLM-powered analysis to provide:

- Real-time hot topic tracking and ranking
- Conversational interface for querying trends
- Topic clustering and sentiment analysis
- Multi-channel alerts (Email, WeChat Work, Telegram)
- Video content extraction and analysis
- Keyboard shortcuts for crawler control

**Key Platforms Supported**: Weibo, Douyin, Bilibili, Zhihu, Baidu, Toutiao, 36Kr, IT Home, Tencent News, NetEase News, Sina Finance, and more.

## Installation

### Prerequisites

**1. Browser Driver Setup**

The system requires a browser driver (ChromeDriver or EdgeDriver) for content extraction:

```bash
# For Chrome - download matching version from:
# https://chromedriver.chromium.org/

# For Edge - download from:
# https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# macOS/Linux - place driver in PATH
sudo mv chromedriver /usr/local/bin/
chmod +x /usr/local/bin/chromedriver

# Windows - add driver directory to PATH environment variable
```

**2. MySQL Database**

```bash
# Install MySQL 5.7+ or 8.0+
# Create database
mysql -u root -p
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

**3. Python Environment**

```bash
# Clone repository
git clone https://github.com/hmmnxkl/LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant.git
cd LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Database Initialization

Reference the `init.py` file to create required tables:

```python
# Key tables structure
# - hot_search_items: stores crawled hot topics
# - analysis_results: stores LLM analysis results
# - push_tasks: manages scheduled push notifications
# - platform_configs: platform-specific settings
```

## Configuration

### Core Configuration Files

**1. Crawler Settings** (`hotsearchcrawler/settings.py`)

```python
# MySQL connection
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_user'
MYSQL_PASSWORD = 'your_password'
MYSQL_DB = 'hotsearch_db'

# Crawler settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
COOKIES_ENABLED = True

# Optional: Platform-specific cookies for authenticated access
PLATFORM_COOKIES = {
    'weibo': 'your_weibo_cookies',
    'zhihu': 'your_zhihu_cookies'
}
```

**2. Environment Variables** (`.env`)

```bash
# Database
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DB=hotsearch_db

# LLM API (OpenAI-compatible format)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
LLM_MODEL=gpt-4

# Huawei Pangu Model (recommended for Chinese text)
PANGU_API_KEY=your_pangu_key
PANGU_API_BASE=https://pangu-api.huawei.com

# Push Notification Services
# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password

# WeChat Work Robot
WECHAT_WORK_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

## Starting the System

### Launch Web Interface

```bash
# Start main application
python app.py

# Access at http://localhost:5000
```

### Control Crawlers

**Via Web Interface**: Use keyboard shortcuts or buttons to start/stop crawlers

**Direct Crawler Execution**:

```bash
# Test individual spider
python runspider-test.py --spider weibo

# Run all spiders
python run_spiders.py

# Run specific platform group
python run_spiders.py --platforms weibo,douyin,bilibili
```

## Core API Usage

### Querying Hot Search Data

```python
from hotsearch_analysis_agent.query_engine import QueryEngine

# Initialize query engine
engine = QueryEngine(db_config={
    'host': 'localhost',
    'user': 'your_user',
    'password': 'your_password',
    'database': 'hotsearch_db'
})

# Get latest hot topics from specific platform
weibo_trends = engine.query_platform('weibo', limit=10)
for item in weibo_trends:
    print(f"{item['rank']}. {item['title']} - {item['heat']}")

# Search topics by keyword
ai_topics = engine.search_topics('人工智能', platforms=['weibo', 'zhihu'])

# Get topics within time range
from datetime import datetime, timedelta
recent = engine.query_by_timerange(
    start_time=datetime.now() - timedelta(hours=24),
    platforms=['weibo', 'douyin']
)
```

### LLM-Powered Analysis

```python
from hotsearch_analysis_agent.analyzer import TopicAnalyzer

analyzer = TopicAnalyzer(
    api_key='your_api_key',
    model='gpt-4'  # or use Pangu model
)

# Sentiment analysis
sentiment = analyzer.analyze_sentiment(
    title="某明星离婚事件引热议",
    content="详细新闻内容..."
)
print(f"Sentiment: {sentiment['polarity']} ({sentiment['score']})")

# Topic clustering
topics = engine.query_platform('weibo', limit=50)
clusters = analyzer.cluster_topics(topics)
for cluster_id, items in clusters.items():
    print(f"Cluster {cluster_id}: {len(items)} topics")
    print(f"Keywords: {', '.join(items['keywords'])}")

# Generate analysis report
report = analyzer.generate_report(
    query="人工智能与前沿科技",
    time_range=timedelta(days=7)
)
print(report)
```

### Setting Up Push Tasks

```python
from hotsearch_analysis_agent.push_manager import PushTaskManager

push_mgr = PushTaskManager()

# Create email push task
email_task = push_mgr.create_task(
    name="AI热点日报",
    query_keywords=["人工智能", "ChatGPT", "大模型"],
    channels=['email'],
    schedule="0 8 * * *",  # Daily at 8 AM
    recipients=["team@example.com"],
    threshold={'heat': 5000}  # Only topics with heat > 5000
)

# Create WeChat Work push task
wechat_task = push_mgr.create_task(
    name="实时舆情预警",
    query_keywords=["公司名称", "产品名称"],
    channels=['wechat_work'],
    schedule="*/30 * * * *",  # Every 30 minutes
    sentiment_filter='negative',  # Only negative sentiment
    webhook_url='${WECHAT_WORK_WEBHOOK}'
)

# Test push task
push_mgr.test_push(task_id=email_task.id)

# List active tasks
tasks = push_mgr.list_tasks(status='active')
for task in tasks:
    print(f"{task['name']}: {task['schedule']} -> {task['channels']}")
```

## Working with Specific Platforms

### Weibo Hot Search

```python
from hotsearchcrawler.spiders.weibo_spider import WeiboSpider

# Custom crawl with specific settings
spider = WeiboSpider()
spider.custom_settings = {
    'DOWNLOAD_DELAY': 2,
    'COOKIES': 'your_cookies_here'
}

# Crawl returns items with structure:
# {
#   'platform': 'weibo',
#   'title': '话题标题',
#   'url': 'https://s.weibo.com/...',
#   'heat': 1234567,
#   'rank': 1,
#   'category': '娱乐',
#   'crawl_time': datetime
# }
```

### Bilibili Video Analysis

The system extracts content even from video-based hot topics:

```python
from hotsearch_analysis_agent.content_extractor import VideoContentExtractor

extractor = VideoContentExtractor()

# Extract video metadata and transcript
video_url = "https://www.bilibili.com/video/BV13pSoBBEvX"
content = extractor.extract(video_url)

print(f"Title: {content['title']}")
print(f"Author: {content['author']}")
print(f"Transcript: {content['transcript'][:200]}...")
print(f"Comments: {len(content['comments'])} loaded")

# Use in analysis
analysis = analyzer.analyze_content(
    title=content['title'],
    content=content['transcript'],
    comments=content['comments'][:100]
)
```

## Common Patterns

### Daily Hot Topic Report Generation

```python
from datetime import datetime, timedelta
from hotsearch_analysis_agent.reporter import ReportGenerator

reporter = ReportGenerator()

# Generate comprehensive daily report
report = reporter.generate_daily_report(
    date=datetime.now(),
    platforms=['weibo', 'douyin', 'zhihu', 'bilibili'],
    include_sentiment=True,
    include_clustering=True,
    top_n=20
)

# Save report
reporter.save_report(report, format='html', path='./reports/')

# Auto-send via configured channels
reporter.send_report(
    report=report,
    channels=['email', 'wechat_work'],
    recipients=['team@example.com']
)
```

### Real-time Keyword Monitoring

```python
from hotsearch_analysis_agent.monitor import KeywordMonitor

monitor = KeywordMonitor(
    keywords=["产品名", "品牌名", "竞品关键词"],
    platforms=['weibo', 'douyin', 'zhihu'],
    check_interval=300  # 5 minutes
)

# Define alert callback
def on_keyword_detected(item):
    print(f"⚠️ Keyword detected: {item['title']}")
    # Trigger immediate analysis
    analysis = analyzer.analyze_sentiment(item['title'], item['content'])
    
    if analysis['polarity'] == 'negative' and analysis['score'] < -0.5:
        # Send urgent alert
        push_mgr.send_immediate_alert(
            message=f"负面舆情预警: {item['title']}",
            channels=['telegram', 'wechat_work']
        )

monitor.set_callback(on_keyword_detected)
monitor.start()
```

### Multi-Platform Trend Comparison

```python
# Compare same topic across platforms
def compare_platforms(keyword, platforms):
    results = {}
    
    for platform in platforms:
        items = engine.search_topics(keyword, platforms=[platform])
        if items:
            results[platform] = {
                'count': len(items),
                'avg_heat': sum(i['heat'] for i in items) / len(items),
                'top_item': items[0],
                'sentiment': analyzer.aggregate_sentiment(items)
            }
    
    return results

comparison = compare_platforms(
    keyword="ChatGPT",
    platforms=['weibo', 'zhihu', 'douyin', 'bilibili']
)

for platform, stats in comparison.items():
    print(f"{platform}:")
    print(f"  话题数: {stats['count']}")
    print(f"  平均热度: {stats['avg_heat']:.0f}")
    print(f"  情感倾向: {stats['sentiment']}")
```

## Troubleshooting

### Crawler Issues

**Problem**: Crawler returns empty results

```python
# Check platform accessibility
from hotsearchcrawler.utils import check_platform_status

status = check_platform_status('weibo')
if not status['accessible']:
    print(f"Platform blocked: {status['reason']}")
    # May need to update cookies or add delays
```

**Problem**: Browser driver not found

```bash
# Verify driver installation
which chromedriver  # macOS/Linux
where chromedriver  # Windows

# Test driver manually
chromedriver --version

# If missing, download and add to PATH
export PATH=$PATH:/path/to/driver/directory
```

### Database Connection Issues

```python
# Test database connection
from hotsearch_analysis_agent.db import test_connection

try:
    test_connection()
    print("Database connected successfully")
except Exception as e:
    print(f"Connection failed: {e}")
    # Check credentials in .env file
```

### LLM Analysis Errors

```python
# Handle API rate limits
from hotsearch_analysis_agent.analyzer import RateLimitedAnalyzer

analyzer = RateLimitedAnalyzer(
    max_requests_per_minute=20,
    retry_attempts=3,
    backoff_factor=2
)

# Batch processing for large datasets
topics = engine.query_platform('weibo', limit=100)
for batch in analyzer.batch_analyze(topics, batch_size=10):
    # Process results
    pass
```

### Push Notification Failures

```python
# Test push channels individually
from hotsearch_analysis_agent.push_manager import test_channels

results = test_channels(['email', 'wechat_work', 'telegram'])
for channel, status in results.items():
    if not status['success']:
        print(f"{channel} failed: {status['error']}")
        # Check corresponding env vars and webhooks
```

## Performance Optimization

### Efficient Querying

```python
# Use indexed queries
engine.create_index('title')
engine.create_index('platform', 'crawl_time')

# Query with pagination
def get_paginated_results(platform, page=1, per_page=50):
    offset = (page - 1) * per_page
    return engine.query_platform(
        platform,
        limit=per_page,
        offset=offset,
        order_by='heat DESC'
    )
```

### Caching Analysis Results

```python
from functools import lru_cache
from hotsearch_analysis_agent.cache import RedisCache

# Use Redis for caching
cache = RedisCache(host='localhost', port=6379)

@cache.cached(expire=3600)  # Cache for 1 hour
def get_trending_analysis(platform):
    topics = engine.query_platform(platform, limit=50)
    return analyzer.cluster_topics(topics)
```

## Advanced Features

### Custom LLM Model Integration

```python
# Use Huawei Pangu model (recommended for Chinese)
from hotsearch_analysis_agent.llm import PanguClient

pangu = PanguClient(
    api_key='${PANGU_API_KEY}',
    model='openpangu-embedded-7b'
)

analyzer = TopicAnalyzer(llm_client=pangu)

# Better Chinese sentiment analysis and domain understanding
result = analyzer.analyze_sentiment("华为发布新款芯片")
```

### Web Interface Keyboard Shortcuts

- `Ctrl+S`: Start all crawlers
- `Ctrl+E`: Stop all crawlers
- `Ctrl+R`: Refresh current view
- `Ctrl+F`: Focus search box
- `Ctrl+P`: Open push task manager

## Project Structure Reference

```
├── app.py                          # Main web application entry
├── hotsearch_analysis_agent/       # Analysis system
│   ├── analyzer.py                 # LLM analysis engine
│   ├── query_engine.py             # Data query interface
│   ├── push_manager.py             # Push notification manager
│   ├── content_extractor.py        # Content extraction (including video)
│   └── reporter.py                 # Report generation
├── hotsearchcrawler/               # Scrapy-based crawler cluster
│   ├── spiders/                    # Platform-specific spiders
│   │   ├── weibo_spider.py
│   │   ├── douyin_spider.py
│   │   ├── bilibili_spider.py
│   │   └── ...
│   └── settings.py                 # Crawler configuration
├── run_spiders.py                  # Crawler orchestration
├── test_push_task.py               # Push task testing
└── requirements.txt                # Python dependencies
```

This skill enables AI agents to help developers deploy and operate a comprehensive Chinese public opinion monitoring system with LLM-powered insights and automated alerting.
