---
name: llm-public-opinion-analytics-assistant
description: Multi-platform hot search crawler and LLM-powered sentiment analysis system with push notifications
triggers:
  - "set up public opinion monitoring system"
  - "configure hot search crawler for multiple platforms"
  - "analyze sentiment trends across social media"
  - "create topic clustering analysis with LLM"
  - "send hot topic alerts to telegram or wechat"
  - "scrape trending news from weibo and bilibili"
  - "build real-time sentiment monitoring dashboard"
  - "configure push notifications for trending topics"
---

# LLM Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant is a comprehensive Chinese public opinion monitoring system that:
- Crawls 26 trending lists from 15 major Chinese platforms (Weibo, Bilibili, Douyin, Baike, etc.)
- Analyzes sentiment and clusters topics using LLM (Huawei Pangu model recommended)
- Provides conversational interface for querying hot topics
- Sends automated reports via email, WeChat, Enterprise WeChat, or Telegram
- Extracts content from news detail pages including video transcripts

## Installation

### Prerequisites

1. **Browser Driver Setup**:
```bash
# Download ChromeDriver matching your Chrome version
# From: https://chromedriver.chromium.org/
# Or EdgeDriver from: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Linux/Mac: place driver in /usr/local/bin/
sudo mv chromedriver /usr/local/bin/
sudo chmod +x /usr/local/bin/chromedriver

# Windows: add driver directory to PATH environment variable
# Verify installation:
chromedriver --version
```

2. **MySQL Database**:
```bash
# Install MySQL and create database
mysql -u root -p
CREATE DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

3. **Python Environment**:
```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # Linux/Mac
# venv\Scripts\activate  # Windows

# Install dependencies
pip install -r requirements.txt
```

### Database Initialization

```python
# Reference init.py for table structure
import pymysql

# Create connection
conn = pymysql.connect(
    host=os.getenv('MYSQL_HOST', 'localhost'),
    user=os.getenv('MYSQL_USER'),
    password=os.getenv('MYSQL_PASSWORD'),
    database='hotsearch',
    charset='utf8mb4'
)

# Main tables structure:
# - hot_search_data: stores crawled trending topics
# - analysis_results: stores LLM analysis outputs
# - push_tasks: manages scheduled push notifications
# - user_queries: logs conversational queries
```

## Configuration

### Environment Variables (.env)

```bash
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch

# LLM API (OpenAI-compatible format)
OPENAI_API_BASE=http://localhost:8000/v1  # For local Pangu model
OPENAI_API_KEY=your_api_key

# Push Notification Services
# Email
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password

# Enterprise WeChat
WECHAT_CORP_ID=your_corp_id
WECHAT_CORP_SECRET=your_corp_secret
WECHAT_AGENT_ID=your_agent_id

# Telegram
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Browser Driver Path (if not in PATH)
CHROME_DRIVER_PATH=/usr/local/bin/chromedriver
```

### Crawler Settings (hotsearchcrawler/settings.py)

```python
# MySQL Configuration
MYSQL_CONFIG = {
    'host': os.getenv('MYSQL_HOST'),
    'user': os.getenv('MYSQL_USER'),
    'password': os.getenv('MYSQL_PASSWORD'),
    'database': os.getenv('MYSQL_DATABASE'),
    'charset': 'utf8mb4'
}

# Optional: Platform-specific cookies
COOKIES = {
    'weibo': 'your_weibo_cookie',  # Optional for enhanced access
    'bilibili': 'your_bilibili_cookie'
}

# Crawler Settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
ROBOTSTXT_OBEY = False
```

## Usage

### Starting the Web Interface

```bash
# Launch main application
python app.py

# Access web interface at http://localhost:5000
```

### Running Crawlers

```python
# Manual crawler execution (all platforms)
python run_spiders.py

# Test specific platform crawler
cd hotsearchcrawler
scrapy crawl weibo_spider
scrapy crawl bilibili_spider
scrapy crawl douyin_spider

# From web interface: use keyboard shortcuts to start/stop
```

### Conversational Query Interface

```python
from hotsearch_analysis_agent.chat_interface import ChatInterface

# Initialize chat interface
chat = ChatInterface()

# Query hot topics
response = chat.query("Show me today's trending topics on Weibo")
print(response)

# Topic clustering
response = chat.query("Cluster topics related to AI and technology")
print(response)

# Sentiment analysis
response = chat.query("Analyze sentiment for topic: GPT-6 release")
print(response)
```

### Programmatic Data Access

```python
import pymysql
from datetime import datetime, timedelta

# Connect to database
conn = pymysql.connect(
    host=os.getenv('MYSQL_HOST'),
    user=os.getenv('MYSQL_USER'),
    password=os.getenv('MYSQL_PASSWORD'),
    database='hotsearch',
    charset='utf8mb4'
)

# Query recent hot topics
with conn.cursor() as cursor:
    sql = """
        SELECT platform, title, hot_value, url, crawl_time
        FROM hot_search_data
        WHERE crawl_time >= %s
        ORDER BY hot_value DESC
        LIMIT 50
    """
    cursor.execute(sql, (datetime.now() - timedelta(hours=24),))
    results = cursor.fetchall()
    
    for row in results:
        print(f"{row[0]}: {row[1]} (热度: {row[2]})")

conn.close()
```

### LLM Analysis Integration

```python
from hotsearch_analysis_agent.llm_analyzer import LLMAnalyzer

# Initialize analyzer with Pangu model
analyzer = LLMAnalyzer(
    api_base=os.getenv('OPENAI_API_BASE'),
    api_key=os.getenv('OPENAI_API_KEY')
)

# Analyze topic sentiment
topics = [
    "GPT-6遭提前曝光, 2M超长上下文来了",
    "DeepSeek V4采用华为算力",
    "Anthropic年化收入暴涨至300亿美元"
]

analysis = analyzer.analyze_sentiment(topics)
print(analysis)

# Topic clustering
clusters = analyzer.cluster_topics(topics, num_clusters=3)
print(clusters)

# Generate report
report = analyzer.generate_report(
    topics=topics,
    query="人工智能与前沿科技",
    time_range="2026-04-07"
)
print(report)
```

### Setting Up Push Notifications

```python
from hotsearch_analysis_agent.push_manager import PushManager

# Initialize push manager
push_mgr = PushManager()

# Create scheduled push task
task = {
    'name': 'AI Tech Daily Digest',
    'query': '人工智能与前沿科技',
    'schedule': 'daily',  # daily, hourly, weekly
    'time': '12:00',
    'channels': ['email', 'telegram'],
    'min_hot_value': 10000
}

push_mgr.create_task(task)

# Manual push test
push_mgr.send_report(
    title="AI热点分析",
    content=report_content,
    channels=['telegram']
)
```

### Testing Push Notifications

```bash
# Run push task test
python test_push_task.py

# Test specific channel
python test_push_task.py --channel telegram
python test_push_task.py --channel email
python test_push_task.py --channel wechat
```

## Key Components

### Crawler Spiders (hotsearchcrawler/spiders/)

```python
# Example: Custom spider for new platform
import scrapy

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    
    def start_requests(self):
        urls = ['https://example.com/trending']
        for url in urls:
            yield scrapy.Request(url, callback=self.parse)
    
    def parse(self, response):
        for item in response.css('.trending-item'):
            yield {
                'platform': 'custom_platform',
                'title': item.css('.title::text').get(),
                'hot_value': item.css('.hot-value::text').get(),
                'url': item.css('a::attr(href)').get(),
                'crawl_time': datetime.now()
            }
```

### Detail Page Content Extraction

```python
from hotsearch_analysis_agent.content_extractor import ContentExtractor

# Extract content from news URL (including video transcripts)
extractor = ContentExtractor(
    driver_path=os.getenv('CHROME_DRIVER_PATH')
)

content = extractor.extract(
    url='https://www.bilibili.com/video/BV13pSoBBEvX/',
    platform='bilibili'
)

print(content['title'])
print(content['text'])
print(content['transcript'])  # Video transcript if available
```

### Analysis Agent Workflow

```python
from hotsearch_analysis_agent.agent import AnalysisAgent

# Initialize agent
agent = AnalysisAgent()

# Full analysis pipeline
result = agent.analyze(
    query="分析最近关于人工智能的热点",
    time_range=24,  # hours
    platforms=['weibo', 'bilibili', 'toutiao'],
    min_hot_value=5000
)

# Result structure:
{
    'query': str,
    'topics': List[dict],
    'clusters': List[dict],
    'sentiment': dict,
    'summary': str,
    'recommendations': List[str]
}
```

## Common Patterns

### Daily Monitoring Script

```python
import schedule
import time
from hotsearch_analysis_agent.agent import AnalysisAgent
from hotsearch_analysis_agent.push_manager import PushManager

def daily_analysis():
    agent = AnalysisAgent()
    push_mgr = PushManager()
    
    # Analyze hot topics
    result = agent.analyze(
        query="综合热点分析",
        time_range=24,
        platforms=['weibo', 'bilibili', 'zhihu']
    )
    
    # Generate and send report
    report = agent.generate_report(result)
    push_mgr.send_report(
        title=f"每日热点分析 - {datetime.now().strftime('%Y-%m-%d')}",
        content=report,
        channels=['email', 'wechat']
    )

# Schedule daily at 9 AM
schedule.every().day.at("09:00").do(daily_analysis)

while True:
    schedule.run_pending()
    time.sleep(60)
```

### Real-time Alert System

```python
from hotsearch_analysis_agent.monitor import HotTopicMonitor

# Monitor for specific keywords
monitor = HotTopicMonitor(
    keywords=['GPT', '人工智能', 'DeepSeek', '华为'],
    hot_value_threshold=50000,
    check_interval=300  # 5 minutes
)

def on_alert(topic):
    print(f"🚨 Alert: {topic['title']} (热度: {topic['hot_value']})")
    # Send immediate notification
    push_mgr.send_alert(topic, channels=['telegram'])

monitor.start(callback=on_alert)
```

## Troubleshooting

### Browser Driver Issues

```bash
# Error: ChromeDriver not found
# Solution: Verify driver in PATH
which chromedriver  # Linux/Mac
where chromedriver  # Windows

# Error: Version mismatch
# Solution: Download matching version
google-chrome --version
# Download corresponding ChromeDriver version
```

### MySQL Connection Errors

```python
# Test connection
import pymysql
try:
    conn = pymysql.connect(
        host=os.getenv('MYSQL_HOST'),
        user=os.getenv('MYSQL_USER'),
        password=os.getenv('MYSQL_PASSWORD'),
        database='hotsearch'
    )
    print("✓ MySQL connection successful")
    conn.close()
except Exception as e:
    print(f"✗ Connection failed: {e}")
```

### Crawler Blocked or Rate Limited

```python
# In hotsearchcrawler/settings.py
# Adjust download delay
DOWNLOAD_DELAY = 3  # Increase delay between requests

# Rotate user agents
USER_AGENT_LIST = [
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64)...',
    'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)...'
]

# Enable proxy middleware
DOWNLOADER_MIDDLEWARES = {
    'scrapy.downloadermiddlewares.useragent.UserAgentMiddleware': None,
    'hotsearchcrawler.middlewares.RandomUserAgentMiddleware': 400,
    'scrapy.downloadermiddlewares.httpproxy.HttpProxyMiddleware': 110,
}
```

### LLM Analysis Errors

```python
# Check API connectivity
import requests

response = requests.post(
    f"{os.getenv('OPENAI_API_BASE')}/chat/completions",
    headers={'Authorization': f"Bearer {os.getenv('OPENAI_API_KEY')}"},
    json={
        'model': 'pangu',
        'messages': [{'role': 'user', 'content': 'test'}]
    }
)
print(response.status_code, response.json())

# Handle timeout/retry
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=4, max=10))
def analyze_with_retry(text):
    return analyzer.analyze_sentiment(text)
```

### Push Notification Failures

```python
# Test each channel individually
from hotsearch_analysis_agent.push_manager import PushManager

push_mgr = PushManager()

# Test email
try:
    push_mgr.send_email(
        to=os.getenv('SMTP_USER'),
        subject='Test',
        body='Test message'
    )
    print("✓ Email sent")
except Exception as e:
    print(f"✗ Email failed: {e}")

# Test Telegram
try:
    push_mgr.send_telegram('Test message')
    print("✓ Telegram sent")
except Exception as e:
    print(f"✗ Telegram failed: {e}")
```

## Performance Optimization

```python
# Batch database inserts
def batch_insert_topics(topics, batch_size=1000):
    with conn.cursor() as cursor:
        for i in range(0, len(topics), batch_size):
            batch = topics[i:i+batch_size]
            sql = """INSERT INTO hot_search_data 
                     (platform, title, hot_value, url, crawl_time)
                     VALUES (%s, %s, %s, %s, %s)"""
            cursor.executemany(sql, batch)
        conn.commit()

# Cache LLM results
from functools import lru_cache

@lru_cache(maxsize=1000)
def cached_analysis(topic_hash):
    return analyzer.analyze_sentiment(topic)
```
