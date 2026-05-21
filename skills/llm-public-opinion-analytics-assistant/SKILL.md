---
name: llm-public-opinion-analytics-assistant
description: Multi-platform Chinese public opinion analysis system with LLM-powered sentiment analysis, topic clustering, and automated alert pushing
triggers:
  - analyze trending topics from Chinese social media platforms
  - set up public opinion monitoring with sentiment analysis
  - crawl hot search data from Weibo Douyin Bilibili
  - configure multi-channel alerts for trending news
  - cluster and analyze Chinese social media discussions
  - deploy opinion analytics dashboard with LLM integration
  - extract insights from Chinese video platform content
  - monitor cross-platform trending topics in real-time
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is a comprehensive Chinese public opinion monitoring system that aggregates real-time trending data from **15 mainstream platforms** across **26 ranking lists**. It combines web scraping, LLM-powered analysis, and multi-channel alerting to provide:

- **Conversational hot topic queries** via natural language interface
- **Topic clustering and sentiment analysis** using LLMs (supports Pangu, OpenAI-compatible models)
- **Multi-platform data crawling** (Weibo, Douyin, Bilibili, Baidu, Zhihu, etc.)
- **Multi-channel push notifications** (Email, WeChat Work, Telegram)
- **Video content extraction** from news detail pages
- **Real-time dashboard** with direct links to source content

## Architecture

The project consists of two main components:

1. **Analysis System** (`hotsearch_analysis_agent/`) - Frontend + LLM analysis engine
2. **Crawler Cluster** (`hotsearchcrawler/`) - Distributed Scrapy-based data collection

## Installation

### Prerequisites

**1. Browser Driver Setup** (required for detail page scraping):

```bash
# Check your Chrome/Edge version first
google-chrome --version  # or msedge --version

# Download matching ChromeDriver from:
# https://chromedriver.chromium.org/

# Place driver in system PATH or project directory
# Verify installation:
chromedriver --version
```

**2. Database Setup**:

```bash
# Install MySQL and create database
mysql -u root -p
CREATE DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

**3. Python Environment**:

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

### Configuration

**1. Initialize Database** (reference `init.py`):

```python
import pymysql

connection = pymysql.connect(
    host='localhost',
    user='your_user',
    password='your_password',
    database='hotsearch',
    charset='utf8mb4'
)

# Create tables for platforms, hot_search, news_detail, etc.
# See init.py for full schema
```

**2. Configure Crawler Settings** (`hotsearchcrawler/settings.py`):

```python
# MySQL Configuration
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_user'
MYSQL_PASSWORD = 'your_password'
MYSQL_DATABASE = 'hotsearch'

# Optional: Platform cookies for authenticated scraping
COOKIES = {
    'weibo': 'your_weibo_cookie_string',
    'douyin': 'your_douyin_cookie_string'
}

# Scrapy settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
```

**3. Configure Analysis System** (`.env` file):

```env
# Database
DB_HOST=localhost
DB_PORT=3306
DB_USER=your_user
DB_PASSWORD=your_password
DB_NAME=hotsearch

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Or use Huawei Pangu Model (local deployment)
# PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b

# Push Notification Channels
# WeChat Work Robot
WECHAT_WORK_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
ALERT_EMAIL_TO=recipient@example.com
```

## Running the System

### Start Crawler Cluster

```bash
# Test individual spider
cd hotsearchcrawler
scrapy crawl weibo_hot  # Test Weibo spider

# Run all spiders (via frontend trigger or manual)
python run_spiders.py
```

### Start Analysis System

```bash
# Launch web interface and API
python app.py

# Access dashboard at http://localhost:5000
```

### Test Push Notifications

```bash
# Test all configured channels
python test_push_task.py
```

## Core API Usage

### Querying Hot Topics

```python
from hotsearch_analysis_agent.query_engine import QueryEngine

engine = QueryEngine()

# Natural language query
result = engine.query("过去24小时关于人工智能的热搜")
# Returns: {
#   'topics': [...],
#   'sentiment': 'positive',
#   'cluster_analysis': {...}
# }

# Platform-specific query
weibo_trends = engine.query("微博热搜前10", platform="weibo")
```

### Sentiment Analysis

```python
from hotsearch_analysis_agent.sentiment_analyzer import SentimentAnalyzer

analyzer = SentimentAnalyzer(model="gpt-4")

text = "这个新政策真的太好了,支持!"
result = analyzer.analyze(text)
# Returns: {
#   'sentiment': 'positive',
#   'confidence': 0.92,
#   'keywords': ['政策', '支持']
# }
```

### Topic Clustering

```python
from hotsearch_analysis_agent.topic_cluster import TopicCluster

cluster = TopicCluster()

# Cluster today's hot topics
topics = cluster.cluster_topics(
    start_date="2026-05-01",
    end_date="2026-05-21",
    platform=["weibo", "douyin", "bilibili"]
)

# Returns grouped topics with similarity scores
for group in topics:
    print(f"Topic: {group['main_topic']}")
    print(f"Related: {group['related_topics']}")
    print(f"Sentiment: {group['avg_sentiment']}")
```

### Setting Up Push Tasks

```python
from hotsearch_analysis_agent.push_manager import PushManager

push_mgr = PushManager()

# Create scheduled analysis task
task = push_mgr.create_task(
    name="AI Tech Daily Report",
    keywords=["人工智能", "大模型", "芯片"],
    platforms=["weibo", "36kr", "iheima"],
    frequency="daily",  # or "hourly", "weekly"
    channels=["wechat", "telegram", "email"],
    sentiment_threshold=0.7  # Only push high-impact news
)

# Manual trigger
push_mgr.execute_task(task.id)
```

## Crawler Development

### Adding New Platform Spider

```python
# hotsearchcrawler/spiders/new_platform.py
import scrapy
from hotsearchcrawler.items import HotSearchItem

class NewPlatformSpider(scrapy.Spider):
    name = 'newplatform_hot'
    allowed_domains = ['newplatform.com']
    start_urls = ['https://newplatform.com/trending']
    
    def parse(self, response):
        for rank, item in enumerate(response.css('.hot-item'), 1):
            yield HotSearchItem(
                platform='newplatform',
                title=item.css('.title::text').get(),
                rank=rank,
                hot_value=item.css('.hot-count::text').get(),
                url=item.css('a::attr(href)').get(),
                crawl_time=datetime.now()
            )
```

### Custom Pipeline for Data Processing

```python
# hotsearchcrawler/pipelines.py
class SentimentPipeline:
    def process_item(self, item, spider):
        # Add real-time sentiment scoring
        from hotsearch_analysis_agent.sentiment_analyzer import SentimentAnalyzer
        
        analyzer = SentimentAnalyzer()
        item['sentiment'] = analyzer.analyze(item['title'])
        return item
```

## Common Patterns

### Generating Daily Report

```python
from hotsearch_analysis_agent.report_generator import ReportGenerator

generator = ReportGenerator(model="gpt-4")

report = generator.generate_report(
    topic="人工智能与前沿科技",
    date_range=("2026-05-01", "2026-05-21"),
    platforms=["weibo", "36kr", "bilibili"],
    include_video_content=True  # Extract video transcripts
)

# report contains:
# - Core findings
# - Detailed news breakdown
# - Sentiment trends
# - Cross-platform analysis
# - Formatted for markdown/HTML/email
```

### Real-time Monitoring Dashboard

```python
# Frontend integration example (Flask route)
from flask import Flask, jsonify
from hotsearch_analysis_agent.live_monitor import LiveMonitor

app = Flask(__name__)
monitor = LiveMonitor()

@app.route('/api/live-trends')
def get_live_trends():
    trends = monitor.get_current_trends(
        platforms=['weibo', 'douyin', 'zhihu'],
        limit=50
    )
    return jsonify(trends)

@app.route('/api/search/<query>')
def search_topics(query):
    results = monitor.search(
        query=query,
        time_range='24h',
        enable_clustering=True
    )
    return jsonify(results)
```

### Video Content Extraction

```python
from hotsearch_analysis_agent.content_extractor import VideoExtractor

extractor = VideoExtractor(driver_path='/usr/bin/chromedriver')

# Extract content from Bilibili video
video_data = extractor.extract(
    url='https://www.bilibili.com/video/BV13pSoBBEvX',
    include_comments=True,
    max_comments=100
)

# Returns:
# {
#   'title': '...',
#   'description': '...',
#   'transcript': '...',  # If available
#   'comments': [...],
#   'stats': {'views': 10000, 'likes': 500}
# }
```

## Troubleshooting

### Browser Driver Issues

```bash
# Error: "chromedriver not found"
which chromedriver  # Verify PATH

# Version mismatch error
google-chrome --version
chromedriver --version
# Ensure versions match (e.g., both 115.x)

# Headless mode failures
export DISPLAY=:99  # Use virtual display on Linux
```

### Database Connection Errors

```python
# Test connection
import pymysql
try:
    conn = pymysql.connect(
        host='localhost',
        user='your_user',
        password='your_password',
        database='hotsearch'
    )
    print("Connected successfully")
except Exception as e:
    print(f"Error: {e}")
    # Check: 1) MySQL running 2) Credentials correct 3) Database exists
```

### Crawler Rate Limiting

```python
# hotsearchcrawler/settings.py
# Adjust these settings if getting blocked

DOWNLOAD_DELAY = 3  # Increase delay between requests
CONCURRENT_REQUESTS_PER_DOMAIN = 1  # Reduce concurrency

# Use rotating user agents
DOWNLOADER_MIDDLEWARES = {
    'scrapy.downloadermiddlewares.useragent.UserAgentMiddleware': None,
    'scrapy_user_agents.middlewares.RandomUserAgentMiddleware': 400,
}

# Add retry logic
RETRY_TIMES = 5
RETRY_HTTP_CODES = [500, 502, 503, 504, 408, 429]
```

### LLM Analysis Timeout

```python
# Increase timeout for large text analysis
from hotsearch_analysis_agent.sentiment_analyzer import SentimentAnalyzer

analyzer = SentimentAnalyzer(
    model="gpt-4",
    timeout=120,  # seconds
    max_tokens=2000  # Limit response size
)

# For local Pangu model - adjust batch size
analyzer = SentimentAnalyzer(
    model_type="pangu",
    model_path="/path/to/model",
    batch_size=4,  # Reduce if OOM
    device="cuda"
)
```

### Push Notification Failures

```python
# Test individual channels
from hotsearch_analysis_agent.push_manager import PushManager

push_mgr = PushManager()

# Test WeChat Work webhook
push_mgr.test_channel('wechat', message="Test message")

# Test Telegram (check bot permissions)
push_mgr.test_channel('telegram', message="Test message")

# Test email (verify SMTP credentials)
push_mgr.test_channel('email', 
    subject="Test", 
    body="Test message"
)
```

## Environment Variables Reference

```env
# Database (required)
DB_HOST=localhost
DB_PORT=3306
DB_USER=hotsearch_user
DB_PASSWORD=secure_password
DB_NAME=hotsearch

# LLM API (choose one)
OPENAI_API_KEY=sk-...
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-4

# OR Pangu Model (local)
PANGU_MODEL_PATH=/models/openpangu-embedded-7b
PANGU_DEVICE=cuda

# Browser Driver
CHROME_DRIVER_PATH=/usr/bin/chromedriver

# Push Channels (optional)
WECHAT_WORK_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxx
TELEGRAM_BOT_TOKEN=123456:ABC-DEF...
TELEGRAM_CHAT_ID=123456789
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=alerts@yourdomain.com
SMTP_PASSWORD=app_specific_password
```

## Performance Tips

- **Use local Pangu model** for lower latency and cost-free analysis
- **Enable caching** for repeated queries: `QueryEngine(use_cache=True)`
- **Batch processing** for large datasets: `analyzer.analyze_batch(texts, batch_size=32)`
- **Async crawling** for multiple platforms: `asyncio.gather(*[crawl(platform) for platform in platforms])`
- **Database indexing** on `platform`, `crawl_time`, and `hot_value` columns for faster queries
