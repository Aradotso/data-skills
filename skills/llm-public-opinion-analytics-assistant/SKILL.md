---
name: llm-public-opinion-analytics-assistant
description: Use the LLM-based intelligent public opinion analytics assistant to crawl hot search data from 15 platforms, analyze sentiment, cluster topics, and push reports via multiple channels.
triggers:
  - set up public opinion monitoring system
  - crawl hot search trends from multiple platforms
  - analyze sentiment and cluster trending topics
  - configure hot topic push notifications to wechat
  - build a multilingual opinion analytics dashboard
  - extract content from news articles and videos
  - create automated trending topic reports
  - monitor social media sentiment across platforms
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is an intelligent public opinion analytics assistant that combines real-time data from **26 trending lists across 15 mainstream platforms** with large language model analysis capabilities. It provides conversational hot search queries, topic clustering, sentiment analysis, and multi-channel alert pushing (email, WeChat, Enterprise WeChat, Telegram).

## What It Does

- **Multi-platform data crawling**: Scrapes trending topics from 15 platforms including Weibo, Bilibili, Zhihu, etc.
- **LLM-powered analysis**: Performs topic clustering, sentiment analysis, and trend detection using configurable LLMs (Pangu, OpenAI-compatible)
- **Conversational interface**: Natural language queries for trending topics and analysis
- **Content extraction**: Deep crawling of news detail pages, including video content metadata
- **Multi-channel alerting**: Push reports via Enterprise WeChat, Telegram, email (SMTP)
- **Automated reporting**: Generate comprehensive trend analysis reports with clustering

## Installation

### Prerequisites

**1. Browser Driver Setup**

The project requires a browser driver (Chrome or Edge) for content extraction:

```bash
# Check your browser version first
# Chrome: Settings → About
# Edge: Settings → About

# Download matching driver version:
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Add driver to PATH or place in project directory
# Verify installation:
chromedriver --version
```

**2. Python Environment**

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

**3. Database Setup**

```bash
# Install MySQL and create database
mysql -u root -p

CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

Reference `init.py` for table schemas. Key tables:
- `hot_search_data`: Stores crawled trending topics
- `analysis_results`: Stores LLM analysis results
- `push_tasks`: Manages scheduled push notifications

### Configuration

**1. Environment Variables (`.env`)**

```bash
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=${MYSQL_PASSWORD}
MYSQL_DATABASE=hotsearch_db

# OpenAI-compatible LLM API (for analysis)
OPENAI_API_KEY=${OPENAI_API_KEY}
OPENAI_API_BASE=https://api.openai.com/v1  # or local endpoint

# Push Notification Channels
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=${EMAIL_USER}
SMTP_PASSWORD=${EMAIL_PASSWORD}

TELEGRAM_BOT_TOKEN=${TELEGRAM_BOT_TOKEN}
TELEGRAM_CHAT_ID=${TELEGRAM_CHAT_ID}

WECHAT_WEBHOOK=${WECHAT_WEBHOOK_URL}
WECHAT_APP_ID=${WECHAT_APP_ID}
WECHAT_APP_SECRET=${WECHAT_APP_SECRET}
```

**2. Crawler Settings (`hotsearchcrawler/settings.py`)**

```python
# MySQL connection for crawlers
MYSQL_CONFIG = {
    'host': 'localhost',
    'port': 3306,
    'user': 'root',
    'password': os.getenv('MYSQL_PASSWORD'),
    'database': 'hotsearch_db',
    'charset': 'utf8mb4'
}

# Optional: Platform-specific cookies
COOKIES = {
    'weibo': 'your_cookie_string',  # if needed
    'bilibili': 'your_cookie_string'
}

# Crawler settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
ROBOTSTXT_OBEY = False
```

## Key Commands

### Start Web Interface

```bash
# Launch the analysis dashboard
python app.py

# Access at http://localhost:5000
```

### Run Crawlers

```bash
# Test individual spider
python runspider-test.py

# Run all spiders (26 trending lists)
python run_spiders.py

# Or use keyboard shortcut in web interface (configurable)
```

### Test Push Notifications

```bash
# Test all configured push channels
python test_push_task.py

# This will send a test report via:
# - Email (SMTP)
# - Telegram
# - Enterprise WeChat
# - WeChat Work App
```

## Core API Usage

### Programmatic Data Access

```python
from hotsearch_analysis_agent.database import DatabaseManager
from hotsearch_analysis_agent.llm_analyzer import LLMAnalyzer

# Initialize database connection
db = DatabaseManager()

# Query trending topics from specific platform
results = db.query(
    """
    SELECT title, url, platform, hot_score, publish_time 
    FROM hot_search_data 
    WHERE platform = %s 
    ORDER BY hot_score DESC 
    LIMIT 20
    """,
    ('weibo',)
)

for row in results:
    print(f"{row['title']} - Score: {row['hot_score']}")
```

### LLM Analysis

```python
from hotsearch_analysis_agent.llm_analyzer import LLMAnalyzer
import os

# Initialize analyzer with OpenAI-compatible API
analyzer = LLMAnalyzer(
    api_key=os.getenv('OPENAI_API_KEY'),
    api_base=os.getenv('OPENAI_API_BASE', 'https://api.openai.com/v1'),
    model='gpt-4'  # or 'pangu-embedded-7b' for local deployment
)

# Sentiment analysis
text = "某科技公司发布新产品,用户反响热烈"
sentiment = analyzer.analyze_sentiment(text)
print(f"Sentiment: {sentiment['polarity']}, Score: {sentiment['score']}")

# Topic clustering
topics = [
    "AI大模型新突破",
    "人工智能商业化进展",
    "DeepSeek采用国产算力",
    "芯片供应链调整"
]
clusters = analyzer.cluster_topics(topics)
for cluster_id, cluster_topics in clusters.items():
    print(f"Cluster {cluster_id}: {cluster_topics}")
```

### Query Interface

```python
from hotsearch_analysis_agent.query_engine import QueryEngine

engine = QueryEngine(db_manager=db, llm_analyzer=analyzer)

# Natural language query
response = engine.query("最近人工智能领域有什么热点?")
print(response['answer'])
print(response['sources'])  # List of source articles

# Filtered query with time range
response = engine.query(
    "分析科技领域的情感倾向",
    filters={
        'platforms': ['weibo', 'zhihu'],
        'start_date': '2026-05-01',
        'end_date': '2026-05-19'
    }
)
```

### Push Notification Setup

```python
from hotsearch_analysis_agent.push_manager import PushManager
import os

push_manager = PushManager()

# Create scheduled push task
task = push_manager.create_task(
    name="AI Technology Daily Report",
    query="人工智能与前沿科技",
    channels=['email', 'telegram', 'wechat_work'],
    schedule='0 18 * * *',  # Daily at 6 PM (cron format)
    recipients={
        'email': ['team@company.com'],
        'telegram': [os.getenv('TELEGRAM_CHAT_ID')],
        'wechat_work': [os.getenv('WECHAT_WEBHOOK')]
    }
)

# Execute task manually
push_manager.execute_task(task['id'])

# List all tasks
tasks = push_manager.list_tasks()
for t in tasks:
    print(f"{t['name']}: {t['schedule']} -> {t['channels']}")
```

## Common Patterns

### Custom Spider for New Platform

```python
# hotsearchcrawler/spiders/my_platform_spider.py
import scrapy
from hotsearchcrawler.items import HotSearchItem

class MyPlatformSpider(scrapy.Spider):
    name = 'my_platform'
    start_urls = ['https://myplatform.com/trending']
    
    def parse(self, response):
        for trend in response.css('.trend-item'):
            item = HotSearchItem()
            item['title'] = trend.css('.title::text').get()
            item['url'] = trend.css('a::attr(href)').get()
            item['hot_score'] = trend.css('.score::text').get()
            item['platform'] = 'my_platform'
            yield item
```

### Custom Analysis Prompt

```python
from hotsearch_analysis_agent.llm_analyzer import LLMAnalyzer

analyzer = LLMAnalyzer(
    api_key=os.getenv('OPENAI_API_KEY'),
    model='gpt-4'
)

# Override default sentiment analysis prompt
custom_prompt = """
分析以下新闻标题的情感倾向,考虑中文语境中的隐喻和反讽:
标题: {text}

返回JSON格式: {{"polarity": "positive/negative/neutral", "score": 0.0-1.0, "reasoning": "..."}}
"""

result = analyzer.analyze_with_custom_prompt(
    text="内存太贵?厂商将复产DDR3主板",
    prompt_template=custom_prompt
)
```

### Batch Content Extraction

```python
from hotsearch_analysis_agent.content_extractor import ContentExtractor

extractor = ContentExtractor()

urls = [
    'https://www.bilibili.com/video/BV13pSoBBEvX',
    'https://www.iheima.com/article-395922.html',
    'https://news.sina.cn/2026-04-07/detail-inhtrumc5298053.d.html'
]

# Extract content (including video metadata)
for url in urls:
    content = extractor.extract(url)
    print(f"Title: {content['title']}")
    print(f"Summary: {content['summary'][:200]}...")
    if content.get('video_info'):
        print(f"Video duration: {content['video_info']['duration']}")
```

### Generate Comprehensive Report

```python
from hotsearch_analysis_agent.report_generator import ReportGenerator
from datetime import datetime, timedelta

generator = ReportGenerator(
    db_manager=db,
    llm_analyzer=analyzer
)

# Generate report for last 7 days
end_date = datetime.now()
start_date = end_date - timedelta(days=7)

report = generator.generate(
    query="人工智能与前沿科技",
    start_date=start_date,
    end_date=end_date,
    include_clustering=True,
    include_sentiment=True,
    max_articles=50
)

print(report['markdown'])  # Formatted markdown report
```

## Troubleshooting

### Browser Driver Issues

```python
# If chromedriver not found in PATH
from selenium import webdriver
from selenium.webdriver.chrome.service import Service

service = Service(executable_path='/path/to/chromedriver')
driver = webdriver.Chrome(service=service)
```

### Database Connection Errors

```python
# Test MySQL connection
import pymysql

try:
    conn = pymysql.connect(
        host='localhost',
        user='root',
        password=os.getenv('MYSQL_PASSWORD'),
        database='hotsearch_db',
        charset='utf8mb4'
    )
    print("Database connected successfully")
    conn.close()
except Exception as e:
    print(f"Connection failed: {e}")
```

### Crawler Rate Limiting

```python
# In hotsearchcrawler/settings.py
DOWNLOAD_DELAY = 2  # Increase delay between requests
AUTOTHROTTLE_ENABLED = True
AUTOTHROTTLE_START_DELAY = 1
AUTOTHROTTLE_MAX_DELAY = 10

# Add retry middleware
RETRY_TIMES = 3
RETRY_HTTP_CODES = [500, 502, 503, 504, 522, 524, 408, 429]
```

### LLM API Timeout

```python
from hotsearch_analysis_agent.llm_analyzer import LLMAnalyzer

analyzer = LLMAnalyzer(
    api_key=os.getenv('OPENAI_API_KEY'),
    timeout=60,  # Increase timeout to 60 seconds
    max_retries=3
)
```

### Content Extraction Failures

```python
from hotsearch_analysis_agent.content_extractor import ContentExtractor

extractor = ContentExtractor(
    headless=True,  # Run browser in background
    timeout=30,
    retry_on_failure=True
)

# Fallback to simpler extraction
try:
    content = extractor.extract(url, method='selenium')
except Exception:
    content = extractor.extract(url, method='requests')  # Fallback
```

## Local Pangu Model Deployment

For enhanced privacy and offline analysis, deploy the Pangu model locally:

```bash
# Download Pangu Embedded 7B model
wget https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

# Configure LLM analyzer to use local endpoint
# In .env:
OPENAI_API_BASE=http://localhost:8000/v1
OPENAI_API_KEY=dummy  # Not required for local deployment
```

```python
# Use local Pangu model
analyzer = LLMAnalyzer(
    api_base='http://localhost:8000/v1',
    model='pangu-embedded-7b',
    api_key='dummy'
)
```

## Advanced Features

### Custom Push Channel

```python
from hotsearch_analysis_agent.push_manager import PushChannel

class SlackPushChannel(PushChannel):
    def __init__(self, webhook_url):
        self.webhook_url = webhook_url
    
    def send(self, report):
        import requests
        payload = {
            'text': report['title'],
            'blocks': [
                {
                    'type': 'section',
                    'text': {'type': 'mrkdwn', 'text': report['summary']}
                }
            ]
        }
        response = requests.post(self.webhook_url, json=payload)
        return response.status_code == 200

# Register custom channel
push_manager.register_channel('slack', SlackPushChannel(
    webhook_url=os.getenv('SLACK_WEBHOOK')
))
```

This skill covers the complete workflow from installation to advanced usage, enabling AI agents to effectively help developers deploy and use this public opinion analytics system.
