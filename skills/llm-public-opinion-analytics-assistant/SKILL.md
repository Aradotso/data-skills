---
name: llm-public-opinion-analytics-assistant
description: Multi-platform hot search crawler and LLM-powered public opinion analysis system with sentiment analysis, topic clustering, and multi-channel alerts
triggers:
  - set up a public opinion monitoring system
  - analyze hot topics from chinese social media platforms
  - build a sentiment analysis dashboard with LLM
  - create automated alerts for trending topics
  - scrape weibo douyin bilibili hot search rankings
  - configure multi-channel notification system for news monitoring
  - analyze public sentiment using pangu model
  - cluster and track trending topics across platforms
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is a comprehensive public opinion monitoring and analysis system that:
- Crawls hot search rankings from **15 major Chinese platforms** (26 total ranking lists)
- Stores data in MySQL with timestamp tracking
- Provides conversational query interface for hot topics
- Performs topic clustering and sentiment analysis using LLMs
- Sends alerts via Email, WeChat Work, Telegram, and Enterprise WeChat
- Supports keyboard shortcuts for crawler control
- Extracts content from news detail pages (including video content)

**Key Components:**
- `hotsearch_analysis_agent/` - Analysis system with LLM integration
- `hotsearchcrawler/` - Scrapy-based crawler cluster
- `app.py` - Main Flask application
- `run_spiders.py` - Crawler orchestration script

## Installation

### Prerequisites

1. **Browser Driver Setup** (required for content extraction):

```bash
# Check your Chrome/Edge version first
# Chrome: chrome://version
# Edge: edge://version

# Download matching ChromeDriver or EdgeDriver
# Place driver in system PATH or browser directory

# Verify installation
chromedriver --version
# or
msedgedriver --version
```

2. **MySQL Database**:

```bash
# Install MySQL 5.7+ or 8.0+
# Create database and tables using init.py as reference

mysql -u root -p
CREATE DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

3. **Python Environment**:

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Core Dependencies

```python
# requirements.txt includes:
# - scrapy (web crawling)
# - flask (web interface)
# - pymysql (database)
# - openai (LLM API)
# - selenium (browser automation)
# - requests (HTTP client)
# - beautifulsoup4 (HTML parsing)
# - apscheduler (task scheduling)
```

## Configuration

### 1. Environment Variables (.env)

```bash
# Database Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_username
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch

# LLM API Configuration (OpenAI-compatible format)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
OPENAI_MODEL=gpt-4

# Or use Huawei Pangu Model (recommended)
PANGU_API_KEY=your_pangu_key
PANGU_API_BASE=https://pangu-api-endpoint
PANGU_MODEL=openpangu-embedded-7b

# Notification Channels
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_USER=your_email@example.com
SMTP_PASSWORD=your_email_password

WECHAT_WORK_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxx
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

### 2. Crawler Settings (hotsearchcrawler/settings.py)

```python
# MySQL Configuration
MYSQL_CONFIG = {
    'host': os.getenv('MYSQL_HOST', 'localhost'),
    'port': int(os.getenv('MYSQL_PORT', 3306)),
    'user': os.getenv('MYSQL_USER'),
    'password': os.getenv('MYSQL_PASSWORD'),
    'database': os.getenv('MYSQL_DATABASE'),
    'charset': 'utf8mb4'
}

# Optional: Platform-specific cookies
COOKIES_ENABLED = True
COOKIES = {
    'weibo': 'your_weibo_cookie',  # Optional, for better access
    'douyin': 'your_douyin_cookie'
}

# Crawler Settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
RANDOMIZE_DOWNLOAD_DELAY = True
```

### 3. Database Schema

```python
# Reference from init.py
from hotsearch_analysis_agent.database import Database

db = Database()

# Create tables
db.execute("""
CREATE TABLE IF NOT EXISTS hot_search_data (
    id INT AUTO_INCREMENT PRIMARY KEY,
    platform VARCHAR(50) NOT NULL,
    rank_type VARCHAR(50) NOT NULL,
    title VARCHAR(500) NOT NULL,
    url VARCHAR(1000),
    heat_value VARCHAR(100),
    crawl_time DATETIME NOT NULL,
    detail_content TEXT,
    sentiment VARCHAR(20),
    INDEX idx_platform (platform),
    INDEX idx_crawl_time (crawl_time),
    INDEX idx_sentiment (sentiment)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
""")

db.execute("""
CREATE TABLE IF NOT EXISTS push_tasks (
    id INT AUTO_INCREMENT PRIMARY KEY,
    task_name VARCHAR(200) NOT NULL,
    query_keywords TEXT NOT NULL,
    schedule_type VARCHAR(20) NOT NULL,
    schedule_config JSON,
    channels JSON NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    last_run DATETIME,
    INDEX idx_active (is_active)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
""")
```

## Running the System

### Start the Web Application

```bash
# Main application (includes web UI and API)
python app.py

# Access at http://localhost:5000
```

### Run Crawlers

```python
# Method 1: Via Web UI
# - Navigate to http://localhost:5000
# - Use keyboard shortcuts or UI buttons to start/stop crawlers

# Method 2: Direct script execution
python run_spiders.py

# Method 3: Test individual spider
cd hotsearchcrawler
scrapy crawl weibo_hot  # Run Weibo spider
scrapy crawl douyin_hot  # Run Douyin spider
scrapy crawl bilibili_hot  # Run Bilibili spider
```

### Test Components

```python
# Test push notifications
python test_push_task.py

# Test crawler functionality
python runspider-test.py
```

## Key API Usage

### Query Hot Topics

```python
from hotsearch_analysis_agent.agent import HotSearchAgent

agent = HotSearchAgent()

# Natural language query
response = agent.query("最近关于人工智能的热点有哪些?")
print(response)

# Specific platform query
response = agent.query("微博热搜前10名")
print(response)

# Time-range query
response = agent.query("过去24小时内的科技类热点")
print(response)
```

### Sentiment Analysis

```python
from hotsearch_analysis_agent.sentiment_analyzer import SentimentAnalyzer

analyzer = SentimentAnalyzer()

# Analyze single text
text = "这款产品太好用了，强烈推荐！"
sentiment = analyzer.analyze(text)
print(sentiment)  # Output: {'label': 'positive', 'score': 0.95}

# Batch analysis from database
results = analyzer.analyze_hot_topics(
    platform='weibo',
    time_range='24h'
)
```

### Topic Clustering

```python
from hotsearch_analysis_agent.topic_cluster import TopicCluster

cluster = TopicCluster()

# Cluster recent topics
topics = cluster.cluster_topics(
    keywords=['人工智能', '科技'],
    time_range='7d',
    min_cluster_size=3
)

for cluster_id, cluster_topics in topics.items():
    print(f"Cluster {cluster_id}:")
    for topic in cluster_topics:
        print(f"  - {topic['title']} ({topic['platform']})")
```

### Configure Push Tasks

```python
from hotsearch_analysis_agent.push_manager import PushManager

push_mgr = PushManager()

# Create scheduled push task
task = push_mgr.create_task(
    task_name="AI Daily Digest",
    keywords=["人工智能", "机器学习", "深度学习"],
    schedule_type="daily",
    schedule_config={"hour": 9, "minute": 0},
    channels={
        "email": {
            "enabled": True,
            "recipients": ["user@example.com"]
        },
        "wechat_work": {
            "enabled": True,
            "webhook": os.getenv('WECHAT_WORK_WEBHOOK')
        },
        "telegram": {
            "enabled": True,
            "bot_token": os.getenv('TELEGRAM_BOT_TOKEN'),
            "chat_id": os.getenv('TELEGRAM_CHAT_ID')
        }
    }
)

# List all tasks
tasks = push_mgr.list_tasks()

# Toggle task status
push_mgr.toggle_task(task_id=1, is_active=False)
```

## Crawler Development

### Add New Platform Spider

```python
# hotsearchcrawler/spiders/new_platform.py
import scrapy
from hotsearchcrawler.items import HotSearchItem

class NewPlatformSpider(scrapy.Spider):
    name = 'new_platform_hot'
    allowed_domains = ['newplatform.com']
    start_urls = ['https://newplatform.com/hot']
    
    custom_settings = {
        'CONCURRENT_REQUESTS': 8,
        'DOWNLOAD_DELAY': 1
    }
    
    def parse(self, response):
        for rank, item in enumerate(response.css('.hot-item'), 1):
            hot_item = HotSearchItem()
            hot_item['platform'] = 'newplatform'
            hot_item['rank_type'] = 'hot_search'
            hot_item['title'] = item.css('.title::text').get()
            hot_item['url'] = item.css('a::attr(href)').get()
            hot_item['heat_value'] = item.css('.heat::text').get()
            hot_item['rank'] = rank
            
            # Follow link to get detail content
            if hot_item['url']:
                yield response.follow(
                    hot_item['url'],
                    callback=self.parse_detail,
                    meta={'item': hot_item}
                )
    
    def parse_detail(self, response):
        item = response.meta['item']
        item['detail_content'] = response.css('.content::text').getall()
        yield item
```

### Custom Pipeline for Data Processing

```python
# hotsearchcrawler/pipelines.py
from hotsearch_analysis_agent.database import Database
import datetime

class HotSearchPipeline:
    def __init__(self):
        self.db = Database()
    
    def process_item(self, item, spider):
        # Clean data
        title = item.get('title', '').strip()
        if not title:
            return item
        
        # Insert into database
        self.db.execute("""
            INSERT INTO hot_search_data 
            (platform, rank_type, title, url, heat_value, crawl_time, detail_content)
            VALUES (%s, %s, %s, %s, %s, %s, %s)
        """, (
            item['platform'],
            item['rank_type'],
            title,
            item.get('url'),
            item.get('heat_value'),
            datetime.datetime.now(),
            item.get('detail_content')
        ))
        
        return item
```

## LLM Integration

### Using Huawei Pangu Model (Recommended)

```python
from hotsearch_analysis_agent.llm_client import LLMClient

# Initialize with Pangu model
llm = LLMClient(
    api_key=os.getenv('PANGU_API_KEY'),
    api_base=os.getenv('PANGU_API_BASE'),
    model='openpangu-embedded-7b'
)

# Analyze hot topics
prompt = """
分析以下热搜话题的舆情特点:
1. GPT-6遭提前曝光, 2M超长上下文来了
2. DeepSeek V4采用华为算力
3. 中国主流大模型周调用量连续五周超越美国

请提供:
1. 核心主题归纳
2. 情感倾向分析
3. 舆情风险评估
"""

response = llm.chat(prompt)
print(response)
```

### Generate Analysis Report

```python
from hotsearch_analysis_agent.report_generator import ReportGenerator

generator = ReportGenerator()

# Generate daily report
report = generator.generate_report(
    keywords=['人工智能', '前沿科技'],
    time_range='24h',
    include_sentiment=True,
    include_clustering=True
)

# Send report
generator.send_report(
    report=report,
    channels=['email', 'wechat_work']
)
```

## Common Patterns

### Real-time Monitoring Workflow

```python
from apscheduler.schedulers.background import BackgroundScheduler
from hotsearch_analysis_agent.monitor import HotSearchMonitor

def setup_monitoring():
    monitor = HotSearchMonitor()
    scheduler = BackgroundScheduler()
    
    # Schedule crawler runs every 30 minutes
    scheduler.add_job(
        monitor.run_crawlers,
        'interval',
        minutes=30,
        id='crawler_job'
    )
    
    # Schedule analysis every hour
    scheduler.add_job(
        monitor.analyze_and_alert,
        'interval',
        hours=1,
        id='analysis_job',
        kwargs={
            'keywords': ['突发', '重要', '热点'],
            'threshold_heat': 100000
        }
    )
    
    scheduler.start()
    return scheduler

# Usage
scheduler = setup_monitoring()
```

### Custom Alert Rules

```python
from hotsearch_analysis_agent.alert_rules import AlertRule

# Define custom rule
class BreakingNewsRule(AlertRule):
    def __init__(self):
        self.keywords = ['突发', '紧急', '重大']
        self.min_heat = 50000
    
    def should_alert(self, item):
        # Check if title contains breaking news keywords
        has_keyword = any(kw in item['title'] for kw in self.keywords)
        
        # Check heat value
        heat = self.parse_heat(item.get('heat_value', '0'))
        
        # Alert if both conditions met
        return has_keyword and heat >= self.min_heat
    
    def parse_heat(self, heat_str):
        # Parse heat value (handles "10万", "100k", etc.)
        import re
        if '万' in heat_str:
            return float(re.findall(r'\d+\.?\d*', heat_str)[0]) * 10000
        return float(re.findall(r'\d+', heat_str)[0])

# Apply rule
rule = BreakingNewsRule()
monitor.add_rule(rule)
```

### Multi-Platform Aggregation

```python
from hotsearch_analysis_agent.aggregator import PlatformAggregator

aggregator = PlatformAggregator()

# Get cross-platform trending topics
trending = aggregator.get_cross_platform_topics(
    platforms=['weibo', 'douyin', 'bilibili'],
    time_range='6h',
    min_platforms=2  # Topic must appear on at least 2 platforms
)

for topic in trending:
    print(f"Topic: {topic['normalized_title']}")
    print(f"Platforms: {', '.join(topic['platforms'])}")
    print(f"Total Heat: {topic['total_heat']}")
    print(f"URLs: {topic['urls']}")
    print("---")
```

## Troubleshooting

### Browser Driver Issues

```bash
# Error: chromedriver not found
# Solution 1: Add to PATH
export PATH=$PATH:/path/to/driver/directory

# Solution 2: Specify driver location in code
from selenium import webdriver
options = webdriver.ChromeOptions()
driver = webdriver.Chrome(
    executable_path='/path/to/chromedriver',
    options=options
)

# Error: version mismatch
# Solution: Download matching driver version
# Check browser: chrome://version
# Download from: https://chromedriver.chromium.org/
```

### Database Connection Issues

```python
# Error: Can't connect to MySQL server
# Check connection
import pymysql

try:
    conn = pymysql.connect(
        host=os.getenv('MYSQL_HOST'),
        port=int(os.getenv('MYSQL_PORT')),
        user=os.getenv('MYSQL_USER'),
        password=os.getenv('MYSQL_PASSWORD'),
        charset='utf8mb4'
    )
    print("Connection successful")
except Exception as e:
    print(f"Connection failed: {e}")

# Error: Table doesn't exist
# Run database initialization
python init.py
```

### Crawler Being Blocked

```python
# Add delays and user agents
# In hotsearchcrawler/settings.py

DOWNLOAD_DELAY = 2
RANDOMIZE_DOWNLOAD_DELAY = True

USER_AGENT = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'

# Use rotating proxies
DOWNLOADER_MIDDLEWARES = {
    'scrapy.downloadermiddlewares.httpproxy.HttpProxyMiddleware': 110,
    'hotsearchcrawler.middlewares.ProxyMiddleware': 100,
}

# Add cookies for authenticated access
DEFAULT_REQUEST_HEADERS = {
    'Cookie': 'your_cookies_here'
}
```

### LLM API Errors

```python
# Error: Rate limit exceeded
# Implement retry logic with backoff
import time
from functools import wraps

def retry_with_backoff(retries=3, backoff_in_seconds=1):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for i in range(retries):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if i == retries - 1:
                        raise
                    sleep_time = backoff_in_seconds * (2 ** i)
                    print(f"Retry {i+1}/{retries} after {sleep_time}s")
                    time.sleep(sleep_time)
        return wrapper
    return decorator

@retry_with_backoff(retries=3)
def call_llm(prompt):
    return llm.chat(prompt)
```

### Memory Issues with Large Datasets

```python
# Use batch processing for large queries
from hotsearch_analysis_agent.database import Database

db = Database()

def process_in_batches(query, batch_size=1000):
    offset = 0
    while True:
        batch = db.query(
            f"{query} LIMIT %s OFFSET %s",
            (batch_size, offset)
        )
        if not batch:
            break
        
        # Process batch
        yield batch
        offset += batch_size

# Usage
for batch in process_in_batches("SELECT * FROM hot_search_data"):
    # Process each batch
    analyze_batch(batch)
```

## Advanced Features

### Video Content Extraction

```python
from hotsearch_analysis_agent.video_extractor import VideoExtractor

extractor = VideoExtractor()

# Extract content from video news
video_url = "https://www.bilibili.com/video/BV13pSoBBEvX"
content = extractor.extract(video_url)

print(content)
# Output includes:
# - title
# - description
# - comments (top comments)
# - auto-generated subtitles (if available)
```

### Emotion Trend Analysis

```python
from hotsearch_analysis_agent.emotion_trend import EmotionTrend

trend = EmotionTrend()

# Analyze emotion changes over time
emotions = trend.analyze_trend(
    keywords=['某产品发布'],
    time_range='7d',
    interval='1h'
)

# Plot results
import matplotlib.pyplot as plt
plt.plot(emotions['timestamps'], emotions['positive_ratio'])
plt.plot(emotions['timestamps'], emotions['negative_ratio'])
plt.legend(['Positive', 'Negative'])
plt.show()
```

This skill provides comprehensive coverage of the LLM-Based Public Opinion Analytics Assistant, enabling AI coding agents to help developers deploy, configure, and extend this multi-platform monitoring system effectively.
