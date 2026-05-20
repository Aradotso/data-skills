---
name: llm-public-opinion-analytics-assistant
description: Multi-platform hot topic crawler and LLM-powered sentiment analysis system with push notifications
triggers:
  - set up public opinion monitoring system
  - crawl hot search trends from multiple platforms
  - analyze sentiment and cluster trending topics
  - configure push notifications for trending news
  - deploy multi-platform hot search crawler
  - implement LLM-based opinion analytics
  - monitor social media trending topics
  - set up automated sentiment analysis pipeline
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

LLM-Based Intelligent Public Opinion Analytics Assistant is a comprehensive sentiment analysis system that combines real-time data from 15 mainstream platforms (26 trending lists total) with large language model analysis capabilities. It provides conversational hot search queries, topic clustering, sentiment analysis, and multi-channel push notifications (email, WeChat, Telegram).

**Key Features:**
- Multi-platform web scraping (Weibo, Bilibili, Toutiao, etc.)
- LLM-powered sentiment and clustering analysis
- Natural language query interface
- Multi-channel push notifications
- Browser-based content extraction (including video content)
- Hotkey-controlled crawler management

## Installation

### Prerequisites

1. **Browser Driver Setup** (required for content extraction):

```bash
# For Chrome
# Download ChromeDriver matching your Chrome version from:
# https://chromedriver.chromium.org/

# For Edge
# Download EdgeDriver from:
# https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Place driver in system PATH or project directory
# Verify installation:
chromedriver --version
```

2. **Database Setup**:

```bash
# Install MySQL
# Create database and tables using init.py as reference
mysql -u root -p < init.sql
```

3. **Python Environment**:

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```bash
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_db_user
MYSQL_PASSWORD=your_db_password
MYSQL_DATABASE=hotsearch_db

# LLM API Configuration (OpenAI-compatible format)
LLM_API_KEY=your_llm_api_key
LLM_API_BASE=https://your-llm-endpoint.com/v1
LLM_MODEL=your-model-name

# Push Notification Configurations
# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_email_password

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_telegram_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# WeChat Work (Enterprise WeChat)
WECHAT_WEBHOOK_URL=your_wechat_webhook_url
WECHAT_CORP_ID=your_corp_id
WECHAT_AGENT_ID=your_agent_id
WECHAT_SECRET=your_secret
```

### Crawler Settings

Edit `hotsearchcrawler/settings.py`:

```python
# MySQL connection for crawler
MYSQL_CONFIG = {
    'host': 'localhost',
    'port': 3306,
    'user': 'your_db_user',
    'password': 'your_db_password',
    'database': 'hotsearch_db',
    'charset': 'utf8mb4'
}

# Optional: Platform-specific cookies
COOKIES = {
    'weibo': 'your_weibo_cookies_if_needed',
    'bilibili': 'your_bilibili_cookies_if_needed'
}
```

## Running the System

### Start the Analysis System

```bash
# Run the main application
python app.py

# Application will start on http://localhost:5000
```

### Run Crawlers

```python
# Via web interface (preferred):
# Access frontend and use hotkey controls

# Or run directly:
python run_spiders.py

# Test individual spiders:
cd hotsearchcrawler
scrapy crawl weibo_spider
scrapy crawl bilibili_spider
```

### Test Push Notifications

```bash
python test_push_task.py
```

## Core API Usage

### Data Collection

```python
from hotsearchcrawler.spiders import WeiboSpider, BilibiliSpider
from scrapy.crawler import CrawlerProcess

# Run crawler programmatically
process = CrawlerProcess()
process.crawl(WeiboSpider)
process.crawl(BilibiliSpider)
process.start()
```

### LLM Analysis

```python
from hotsearch_analysis_agent.analyzer import TopicAnalyzer
import os

# Initialize analyzer
analyzer = TopicAnalyzer(
    api_key=os.getenv('LLM_API_KEY'),
    api_base=os.getenv('LLM_API_BASE'),
    model=os.getenv('LLM_MODEL')
)

# Perform sentiment analysis
topics = [
    {"title": "某科技公司发布新产品", "content": "..."},
    {"title": "政策变化引发讨论", "content": "..."}
]

sentiment_results = analyzer.analyze_sentiment(topics)
# Returns: [{"topic": "...", "sentiment": "positive/negative/neutral", "score": 0.85}, ...]

# Perform topic clustering
clusters = analyzer.cluster_topics(topics)
# Returns: {"cluster_1": [...], "cluster_2": [...], ...}

# Generate analysis report
report = analyzer.generate_report(topics, clusters, sentiment_results)
print(report)
```

### Database Operations

```python
import pymysql
import os

# Connect to database
connection = pymysql.connect(
    host=os.getenv('MYSQL_HOST'),
    port=int(os.getenv('MYSQL_PORT')),
    user=os.getenv('MYSQL_USER'),
    password=os.getenv('MYSQL_PASSWORD'),
    database=os.getenv('MYSQL_DATABASE'),
    charset='utf8mb4'
)

# Query hot search data
with connection.cursor() as cursor:
    sql = """
    SELECT platform, title, rank, heat_score, url, created_at 
    FROM hot_search 
    WHERE platform = %s AND created_at > NOW() - INTERVAL 1 HOUR
    ORDER BY rank ASC
    """
    cursor.execute(sql, ('weibo',))
    results = cursor.fetchall()
    
    for row in results:
        print(f"Rank {row[2]}: {row[1]} (heat: {row[3]})")

connection.close()
```

### Push Notification Setup

```python
from hotsearch_analysis_agent.pusher import NotificationPusher
import os

# Initialize pusher
pusher = NotificationPusher()

# Email push
pusher.send_email(
    to='recipient@example.com',
    subject='热点分析报告 - 人工智能',
    content=report_html,
    smtp_config={
        'host': os.getenv('SMTP_HOST'),
        'port': int(os.getenv('SMTP_PORT')),
        'user': os.getenv('SMTP_USER'),
        'password': os.getenv('SMTP_PASSWORD')
    }
)

# Telegram push
pusher.send_telegram(
    message=report_text,
    bot_token=os.getenv('TELEGRAM_BOT_TOKEN'),
    chat_id=os.getenv('TELEGRAM_CHAT_ID')
)

# WeChat Work push
pusher.send_wechat_work(
    content=report_text,
    webhook_url=os.getenv('WECHAT_WEBHOOK_URL')
)
```

## Common Patterns

### Scheduled Analysis Pipeline

```python
import schedule
import time
from hotsearch_analysis_agent.analyzer import TopicAnalyzer
from hotsearch_analysis_agent.pusher import NotificationPusher
import pymysql
import os

def run_analysis_pipeline():
    """Run complete analysis and push pipeline"""
    
    # 1. Fetch latest hot topics from database
    connection = pymysql.connect(
        host=os.getenv('MYSQL_HOST'),
        user=os.getenv('MYSQL_USER'),
        password=os.getenv('MYSQL_PASSWORD'),
        database=os.getenv('MYSQL_DATABASE')
    )
    
    with connection.cursor() as cursor:
        cursor.execute("""
            SELECT platform, title, content, url 
            FROM hot_search 
            WHERE created_at > NOW() - INTERVAL 2 HOUR
            AND keywords LIKE '%人工智能%'
        """)
        topics = cursor.fetchall()
    
    connection.close()
    
    # 2. Perform LLM analysis
    analyzer = TopicAnalyzer(
        api_key=os.getenv('LLM_API_KEY'),
        api_base=os.getenv('LLM_API_BASE')
    )
    
    sentiment = analyzer.analyze_sentiment(topics)
    clusters = analyzer.cluster_topics(topics)
    report = analyzer.generate_report(topics, clusters, sentiment)
    
    # 3. Push notifications
    pusher = NotificationPusher()
    pusher.send_email(
        to='team@company.com',
        subject=f'舆情分析报告 - {time.strftime("%Y-%m-%d")}',
        content=report
    )
    
    print(f"Pipeline completed at {time.strftime('%Y-%m-%d %H:%M:%S')}")

# Schedule every 4 hours
schedule.every(4).hours.do(run_analysis_pipeline)

while True:
    schedule.run_pending()
    time.sleep(60)
```

### Custom Spider Integration

```python
import scrapy
from hotsearchcrawler.items import HotSearchItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://custom-platform.com/trending']
    
    def parse(self, response):
        for item in response.css('.trending-item'):
            yield HotSearchItem(
                platform='custom_platform',
                title=item.css('.title::text').get(),
                rank=item.css('.rank::text').get(),
                heat_score=item.css('.heat::text').get(),
                url=item.css('a::attr(href)').get(),
                keywords=item.css('.tags::text').getall()
            )
```

### Natural Language Query Interface

```python
from hotsearch_analysis_agent.query_agent import QueryAgent

# Initialize query agent
agent = QueryAgent(
    db_config={
        'host': os.getenv('MYSQL_HOST'),
        'user': os.getenv('MYSQL_USER'),
        'password': os.getenv('MYSQL_PASSWORD'),
        'database': os.getenv('MYSQL_DATABASE')
    },
    llm_config={
        'api_key': os.getenv('LLM_API_KEY'),
        'api_base': os.getenv('LLM_API_BASE')
    }
)

# Natural language queries
response = agent.query("最近关于人工智能的热点有哪些?")
response = agent.query("帮我分析微博上科技类话题的情感倾向")
response = agent.query("对比今日头条和知乎的热搜榜单差异")

print(response)
```

## Troubleshooting

### ChromeDriver Version Mismatch

```bash
# Error: SessionNotCreatedException: Message: session not created: 
# This version of ChromeDriver only supports Chrome version X

# Solution: Download matching ChromeDriver version
# Check Chrome version: chrome://version/
# Download from: https://chromedriver.chromium.org/downloads
```

### Database Connection Issues

```python
# Error: pymysql.err.OperationalError: (2003, "Can't connect to MySQL server")

# Check MySQL service is running:
# Linux: sudo systemctl status mysql
# Windows: Check Services panel

# Verify credentials in .env file
# Test connection:
import pymysql
try:
    conn = pymysql.connect(
        host=os.getenv('MYSQL_HOST'),
        user=os.getenv('MYSQL_USER'),
        password=os.getenv('MYSQL_PASSWORD')
    )
    print("Connection successful")
    conn.close()
except Exception as e:
    print(f"Connection failed: {e}")
```

### LLM API Timeout

```python
# Error: Request timeout or rate limit exceeded

# Solution: Configure retry logic
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(min=1, max=10))
def analyze_with_retry(analyzer, topics):
    return analyzer.analyze_sentiment(topics)
```

### Crawler Blocked by Platform

```python
# Error: 403 Forbidden or 429 Too Many Requests

# Solution 1: Add delays in settings.py
DOWNLOAD_DELAY = 2  # seconds between requests
RANDOMIZE_DOWNLOAD_DELAY = True

# Solution 2: Rotate user agents
USER_AGENT_LIST = [
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64)...',
    'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)...'
]

# Solution 3: Use proxy middleware
DOWNLOADER_MIDDLEWARES = {
    'scrapy.downloadermiddlewares.httpproxy.HttpProxyMiddleware': 110,
}
```

### Memory Issues with Large Datasets

```python
# Process data in batches
def process_large_dataset(topics, batch_size=100):
    results = []
    for i in range(0, len(topics), batch_size):
        batch = topics[i:i+batch_size]
        batch_results = analyzer.analyze_sentiment(batch)
        results.extend(batch_results)
        # Clear cache if needed
        import gc
        gc.collect()
    return results
```

## Project Structure Reference

```
project_root/
├── app.py                          # Main application entry point
├── init.py                         # Database initialization reference
├── run_spiders.py                  # Crawler launcher (via frontend)
├── runspider-test.py              # Crawler testing script
├── test_push_task.py              # Push notification testing
├── requirements.txt                # Python dependencies
├── .env                           # Environment configuration
├── hotsearch_analysis_agent/      # Analysis system
│   ├── analyzer.py                # LLM analysis engine
│   ├── query_agent.py             # Natural language query
│   └── pusher.py                  # Push notification handler
└── hotsearchcrawler/              # Crawler cluster
    ├── settings.py                # Scrapy settings
    ├── spiders/                   # Platform-specific spiders
    │   ├── weibo_spider.py
    │   ├── bilibili_spider.py
    │   └── ...
    └── items.py                   # Data models
```
