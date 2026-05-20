---
name: llm-public-opinion-analytics-assistant
description: Multi-platform hot search data crawler and LLM-powered public opinion analysis system with sentiment analysis, topic clustering, and multi-channel alert notifications
triggers:
  - analyze public opinion trends across social platforms
  - set up hot topic monitoring and alerts
  - crawl real-time trending data from Chinese platforms
  - perform sentiment analysis on social media topics
  - cluster and analyze trending topics with LLM
  - configure multi-channel notifications for trending alerts
  - scrape and analyze video platform hot searches
  - build a public opinion monitoring system
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is a comprehensive public opinion analytics platform that combines web scraping from 15 major Chinese platforms (26 different trending lists) with LLM-powered analysis capabilities. It features:

- Real-time hot search data collection from platforms like Weibo, Bilibili, Douyin, Baidu, etc.
- LLM-driven conversational analysis (topic clustering, sentiment analysis, trend detection)
- Multi-channel push notifications (Email, WeChat Work, Telegram, Enterprise WeChat)
- Web UI for querying, analyzing, and managing crawlers
- Separate crawler cluster architecture for scalability

## Architecture

The project consists of two main components:

1. **Crawler Cluster** (`hotsearchcrawler/`) - Scrapy-based distributed crawler system
2. **Analysis System** (`hotsearch_analysis_agent/`) - LLM-powered analytics engine with web interface

## Installation

### Prerequisites

```bash
# Required software
- Python 3.8+
- MySQL 5.7+
- Chrome/Edge browser
- ChromeDriver or EdgeDriver (matching browser version)
```

### Browser Driver Setup

```bash
# Download driver matching your browser version
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/microsoft-edge/tools/webdriver/

# Place driver in system PATH or project directory
# Verify installation
chromedriver --version
```

### Python Environment

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

### Database Setup

```python
# Reference init.py for database schema
# Create database and tables in MySQL

import pymysql

connection = pymysql.connect(
    host=os.getenv('MYSQL_HOST', 'localhost'),
    user=os.getenv('MYSQL_USER', 'root'),
    password=os.getenv('MYSQL_PASSWORD'),
    database=os.getenv('MYSQL_DATABASE', 'hotsearch'),
    charset='utf8mb4'
)

# Run init.py to create necessary tables
# Tables include: hot_search_data, analysis_results, push_tasks, etc.
```

## Configuration

### Environment Variables (.env)

```bash
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch

# OpenAI-Compatible LLM API (supports Huawei Pangu, etc.)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://your-llm-endpoint/v1
LLM_MODEL=pangu-7b

# Push Notification Credentials
# Email
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password

# WeChat Work Bot
WECHAT_WORK_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxx

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Enterprise WeChat Application
ENTERPRISE_WECHAT_CORPID=your_corp_id
ENTERPRISE_WECHAT_SECRET=your_secret
ENTERPRISE_WECHAT_AGENTID=your_agent_id
```

### Crawler Configuration (hotsearchcrawler/settings.py)

```python
# MySQL pipeline settings
MYSQL_HOST = os.getenv('MYSQL_HOST', 'localhost')
MYSQL_PORT = int(os.getenv('MYSQL_PORT', 3306))
MYSQL_USER = os.getenv('MYSQL_USER', 'root')
MYSQL_PASSWORD = os.getenv('MYSQL_PASSWORD')
MYSQL_DATABASE = os.getenv('MYSQL_DATABASE', 'hotsearch')

# Optional: Platform-specific cookies (for platforms requiring auth)
COOKIES = {
    'weibo': 'your_weibo_cookies',
    'bilibili': 'your_bilibili_cookies'
}

# Crawler settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
ROBOTSTXT_OBEY = False
```

## Running the System

### Start the Analysis System (Web UI)

```bash
# From project root
python app.py

# Access web interface at http://localhost:5000
```

### Start Crawler Cluster

```bash
# Via Web UI: Use keyboard shortcuts or buttons to start/stop crawlers

# Or manually run all spiders
python run_spiders.py

# Or test individual spider
python runspider-test.py spider_name
```

## Core Features & Usage

### 1. Data Collection

```python
# Supported platforms (26 trending lists from 15 platforms):
# Weibo, Bilibili, Douyin, Baidu, Zhihu, 36Kr, Toutiao, 
# Sina News, iThome, Tencent News, NetEase News, etc.

# Crawlers automatically:
# - Fetch trending topic titles and rankings
# - Extract detailed content from news pages
# - Handle video content extraction
# - Store structured data in MySQL
```

### 2. LLM-Powered Analysis

```python
# Example: Query trending topics via conversational interface
from hotsearch_analysis_agent import AnalysisAgent

agent = AnalysisAgent(
    api_key=os.getenv('OPENAI_API_KEY'),
    api_base=os.getenv('OPENAI_API_BASE'),
    model=os.getenv('LLM_MODEL')
)

# Natural language query
response = agent.query("分析最近关于人工智能的热点趋势")
# Returns: Clustered topics, sentiment analysis, trend insights

# Topic clustering
clusters = agent.cluster_topics(
    query="科技新闻",
    time_range="24h",
    platforms=["weibo", "zhihu", "36kr"]
)

# Sentiment analysis
sentiment = agent.analyze_sentiment(
    topic="某个热门话题",
    depth="detailed"  # or "summary"
)
```

### 3. Multi-Channel Push Notifications

```python
# Configure push task in Web UI or programmatically
from hotsearch_analysis_agent.push import PushManager

push_manager = PushManager()

# Create monitoring task
task = push_manager.create_task(
    name="AI Technology Monitor",
    keywords=["人工智能", "大模型", "算力"],
    platforms=["weibo", "zhihu", "36kr"],
    channels=["email", "wechat_work", "telegram"],
    schedule="0 */6 * * *",  # Every 6 hours
    analysis_enabled=True  # Include LLM analysis in report
)

# Manual push test
push_manager.test_push(
    task_id=task['id'],
    channel="email"
)
```

### 4. Data Query & Export

```python
# Query hot search data
from hotsearch_analysis_agent.database import HotSearchDB

db = HotSearchDB()

# Get trending topics from specific platform
topics = db.get_trending(
    platform="weibo",
    limit=50,
    time_range="24h"
)

# Search by keyword
results = db.search_topics(
    keyword="人工智能",
    platforms=["weibo", "zhihu"],
    start_date="2026-05-01",
    end_date="2026-05-19"
)

# Export to CSV
db.export_to_csv(
    query_results=results,
    output_path="export_ai_topics.csv"
)
```

## Common Patterns

### Pattern 1: Scheduled Topic Monitoring

```python
# Set up automated monitoring for specific topics
import schedule
import time

def monitor_keywords():
    agent = AnalysisAgent()
    keywords = ["新能源汽车", "电池技术", "充电桩"]
    
    # Analyze trends
    analysis = agent.analyze_keywords(
        keywords=keywords,
        time_window="24h",
        include_sentiment=True,
        cluster_topics=True
    )
    
    # Push if significant trend detected
    if analysis['significance_score'] > 0.7:
        push_manager.send_alert(
            title=f"High-priority trend: {analysis['main_topic']}",
            content=analysis['report'],
            channels=["wechat_work", "email"]
        )

# Schedule every 6 hours
schedule.every(6).hours.do(monitor_keywords)

while True:
    schedule.run_pending()
    time.sleep(60)
```

### Pattern 2: Cross-Platform Trend Correlation

```python
# Identify topics trending across multiple platforms
def find_cross_platform_trends():
    db = HotSearchDB()
    
    # Get top topics from each platform
    platforms = ["weibo", "zhihu", "douyin", "baidu"]
    platform_topics = {}
    
    for platform in platforms:
        topics = db.get_trending(
            platform=platform,
            limit=30,
            time_range="24h"
        )
        platform_topics[platform] = [t['title'] for t in topics]
    
    # Use LLM to find semantic similarities
    agent = AnalysisAgent()
    correlations = agent.find_topic_correlations(platform_topics)
    
    return correlations
```

### Pattern 3: Sentiment Timeline Analysis

```python
# Track sentiment evolution for a specific topic
def track_sentiment_timeline(topic, days=7):
    db = HotSearchDB()
    agent = AnalysisAgent()
    
    timeline = []
    for day in range(days):
        date = datetime.now() - timedelta(days=day)
        
        # Get related posts from that day
        posts = db.search_topics(
            keyword=topic,
            start_date=date.strftime("%Y-%m-%d"),
            end_date=(date + timedelta(days=1)).strftime("%Y-%m-%d")
        )
        
        # Analyze sentiment
        if posts:
            sentiment = agent.analyze_sentiment(
                texts=[p['content'] for p in posts]
            )
            timeline.append({
                'date': date.strftime("%Y-%m-%d"),
                'sentiment_score': sentiment['score'],
                'volume': len(posts)
            })
    
    return timeline
```

## Troubleshooting

### Issue: Crawler fails to start

```bash
# Check browser driver
chromedriver --version

# Verify driver is in PATH
echo $PATH  # Linux/Mac
echo %PATH%  # Windows

# Check MySQL connection
mysql -h localhost -u root -p
USE hotsearch;
SHOW TABLES;

# Test individual spider
cd hotsearchcrawler
scrapy crawl weibo_spider -s LOG_LEVEL=DEBUG
```

### Issue: LLM API errors

```python
# Test API connectivity
import requests
import os

response = requests.post(
    f"{os.getenv('OPENAI_API_BASE')}/chat/completions",
    headers={"Authorization": f"Bearer {os.getenv('OPENAI_API_KEY')}"},
    json={
        "model": os.getenv('LLM_MODEL'),
        "messages": [{"role": "user", "content": "test"}],
        "max_tokens": 10
    }
)
print(response.status_code, response.json())

# Check for rate limits, API key validity, model availability
```

### Issue: Push notifications not working

```python
# Test individual push channels
from hotsearch_analysis_agent.push import test_push_channels

# Test email
test_push_channels.test_email(
    smtp_host=os.getenv('SMTP_HOST'),
    smtp_port=int(os.getenv('SMTP_PORT')),
    smtp_user=os.getenv('SMTP_USER'),
    smtp_password=os.getenv('SMTP_PASSWORD'),
    recipient="test@example.com"
)

# Test WeChat Work
test_push_channels.test_wechat_work(
    webhook_url=os.getenv('WECHAT_WORK_WEBHOOK')
)

# Test Telegram
test_push_channels.test_telegram(
    bot_token=os.getenv('TELEGRAM_BOT_TOKEN'),
    chat_id=os.getenv('TELEGRAM_CHAT_ID')
)

# Check firewall, proxy settings, credential validity
```

### Issue: Video content extraction fails

```bash
# Ensure browser driver supports headless mode
# Update driver to latest version
# Check platform-specific selectors in spider code

# Debug video spider
cd hotsearchcrawler/spiders
# Edit bilibili_spider.py or douyin_spider.py
# Add breakpoints or logging at video extraction points
```

## Advanced Usage

### Custom Spider Development

```python
# Create custom spider in hotsearchcrawler/spiders/

import scrapy
from hotsearchcrawler.items import HotSearchItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://example.com/trending']
    
    def parse(self, response):
        for topic in response.css('.trending-item'):
            item = HotSearchItem()
            item['platform'] = 'custom_platform'
            item['title'] = topic.css('.title::text').get()
            item['rank'] = topic.css('.rank::text').get()
            item['hot_value'] = topic.css('.hot-score::text').get()
            item['url'] = topic.css('a::attr(href)').get()
            item['timestamp'] = datetime.now()
            
            # Follow link for detail content
            yield response.follow(
                item['url'],
                callback=self.parse_detail,
                meta={'item': item}
            )
    
    def parse_detail(self, response):
        item = response.meta['item']
        item['content'] = response.css('.article-content::text').getall()
        yield item
```

### Custom LLM Integration

```python
# Replace with custom LLM endpoint
from openai import OpenAI

client = OpenAI(
    api_key=os.getenv('CUSTOM_LLM_API_KEY'),
    base_url=os.getenv('CUSTOM_LLM_BASE_URL')
)

def custom_analysis(topic, context):
    response = client.chat.completions.create(
        model=os.getenv('CUSTOM_LLM_MODEL'),
        messages=[
            {"role": "system", "content": "You are a public opinion analyst."},
            {"role": "user", "content": f"Analyze this topic: {topic}\nContext: {context}"}
        ],
        temperature=0.7,
        max_tokens=2000
    )
    return response.choices[0].message.content
```

## API Reference

### Web API Endpoints

```python
# If running as API service (app.py with Flask)

# GET /api/trending?platform=weibo&limit=50
# Returns: Latest trending topics

# POST /api/analyze
# Body: {"query": "分析AI趋势", "platforms": ["weibo", "zhihu"]}
# Returns: LLM analysis results

# POST /api/push/create
# Body: Push task configuration
# Returns: Task ID

# GET /api/push/status?task_id=123
# Returns: Task status and history
```

## Performance Tips

- Run crawlers during off-peak hours to avoid rate limiting
- Use Redis for caching frequently accessed trending data
- Deploy crawler cluster on multiple machines for better distribution
- Optimize MySQL indexes on timestamp and platform columns
- Use asynchronous processing for LLM analysis of large datasets

## License

MIT License - See project repository for full license text.
