---
name: llm-public-opinion-analytics-assistant
description: Multi-platform real-time hot search data crawler with LLM-powered sentiment analysis, topic clustering, and multi-channel alert notifications
triggers:
  - "set up a public opinion monitoring system"
  - "analyze trending topics across social media platforms"
  - "crawl hot search data from Chinese platforms"
  - "implement sentiment analysis for news and social posts"
  - "configure multi-channel notifications for trending topics"
  - "build a hot topic aggregation dashboard"
  - "monitor public sentiment across Weibo, Bilibili, and other platforms"
  - "analyze and cluster trending news topics with LLM"
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is a comprehensive public opinion monitoring system that aggregates **26 trending lists from 15 major Chinese platforms** (Weibo, Bilibili, Douyin, Zhihu, etc.) and provides LLM-powered analysis capabilities. It combines distributed web crawling with large language model analysis to deliver conversational hot search queries, topic clustering, sentiment analysis, and multi-channel notifications (Email, WeChat Work, Telegram).

**Key Features:**
- Real-time crawling of trending topics from 15+ platforms
- LLM-powered content analysis (even video content extraction)
- Topic clustering and sentiment analysis
- Multi-channel push notifications (WeChat Work, Telegram, Email)
- Web-based conversational interface for data queries
- Keyboard shortcut-controlled crawler management

## Installation

### Prerequisites

```bash
# Python 3.8+
# MySQL 5.7+
# Chrome/Edge browser installed
```

### Step 1: Browser Driver Setup

Download the appropriate WebDriver for your browser:

**Chrome:** [ChromeDriver](https://chromedriver.chromium.org/)
**Edge:** [EdgeDriver](https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/)

Place the driver in your system PATH or the browser installation directory.

Verify installation:
```bash
chromedriver --version
# or
msedgedriver --version
```

### Step 2: Clone and Install Dependencies

```bash
git clone https://github.com/hmmnxkl/LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant.git
cd LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Step 3: Database Setup

```bash
# Install MySQL and create database
mysql -u root -p

# In MySQL console:
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

Reference `init.py` for table creation:
```python
# Database initialization example
import pymysql

connection = pymysql.connect(
    host='localhost',
    user='your_user',
    password='your_password',
    database='hotsearch_db',
    charset='utf8mb4'
)

# Create tables (refer to init.py for full schema)
```

### Step 4: Configuration

#### Crawler Configuration (`hotsearchcrawler/settings.py`)

```python
# MySQL Configuration
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_user'
MYSQL_PASSWORD = 'your_password'
MYSQL_DATABASE = 'hotsearch_db'

# Optional: Platform cookies for authenticated access
COOKIES = {
    'weibo': 'your_weibo_cookie',
    'bilibili': 'your_bilibili_cookie'
}
```

#### Analysis System Configuration (`.env`)

```bash
# MySQL Configuration
DB_HOST=localhost
DB_PORT=3306
DB_USER=your_user
DB_PASSWORD=your_password
DB_NAME=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
LLM_API_KEY=your_api_key
LLM_API_BASE=https://api.openai.com/v1
LLM_MODEL=gpt-4

# Or use Huawei Pangu model locally
PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model

# Notification Channels
# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
NOTIFICATION_EMAIL=recipient@example.com

# WeChat Work Bot
WEWORK_BOT_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# WeChat Work App
WEWORK_CORP_ID=your_corp_id
WEWORK_AGENT_ID=your_agent_id
WEWORK_SECRET=your_secret

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

## Running the System

### Start the Crawler

```bash
# Test single crawler
python runspider-test.py

# Run all crawlers (26 spiders across 15 platforms)
python run_spiders.py

# Or start from web interface (see below)
```

### Start the Analysis System

```bash
# Main application
python app.py

# Access web interface at http://localhost:5000
```

### Test Notification Push

```bash
# Test push task configuration
python test_push_task.py
```

## Key Components

### 1. Web Crawlers (`hotsearchcrawler/`)

The system includes 26 specialized crawlers for platforms like:
- **Social Media:** Weibo, Douyin, Kuaishou
- **Video Platforms:** Bilibili, iQiyi, Tencent Video
- **News Sites:** Toutiao, NetEase, Sina
- **Forums:** Zhihu, Tieba, Douban

Example crawler structure:
```python
import scrapy
from hotsearchcrawler.items import HotsearchItem

class WeiboSpider(scrapy.Spider):
    name = 'weibo'
    start_urls = ['https://s.weibo.com/top/summary']
    
    def parse(self, response):
        for item in response.css('.td-02'):
            hotsearch_item = HotsearchItem()
            hotsearch_item['title'] = item.css('a::text').get()
            hotsearch_item['url'] = item.css('a::attr(href)').get()
            hotsearch_item['hot_value'] = item.css('span::text').get()
            hotsearch_item['platform'] = 'weibo'
            hotsearch_item['rank'] = item.css('.td-01::text').get()
            
            # Fetch detail page for content analysis
            yield scrapy.Request(
                hotsearch_item['url'],
                callback=self.parse_detail,
                meta={'item': hotsearch_item}
            )
    
    def parse_detail(self, response):
        item = response.meta['item']
        item['content'] = response.css('.content::text').getall()
        yield item
```

### 2. LLM Analysis Engine (`hotsearch_analysis_agent/`)

```python
from hotsearch_analysis_agent.analyzer import TopicAnalyzer

# Initialize analyzer
analyzer = TopicAnalyzer(
    model_type='openai',  # or 'pangu' for local model
    api_key=os.getenv('LLM_API_KEY')
)

# Sentiment analysis
sentiment = analyzer.analyze_sentiment(
    text="这个新闻太棒了,我很喜欢!"
)
print(sentiment)  # {'sentiment': 'positive', 'score': 0.92}

# Topic clustering
topics = analyzer.cluster_topics(
    articles=[
        {'title': 'AI breakthrough', 'content': '...'},
        {'title': 'Machine learning advances', 'content': '...'}
    ],
    num_clusters=3
)

# Generate summary report
report = analyzer.generate_report(
    query='人工智能与前沿科技',
    time_range='2026-04-07',
    platforms=['weibo', 'bilibili', 'zhihu']
)
```

### 3. Conversational Query Interface

```python
# API endpoint for conversational queries
@app.route('/api/query', methods=['POST'])
def query_hotsearch():
    user_query = request.json.get('query')
    
    # Natural language processing
    from hotsearch_analysis_agent.nlp import QueryParser
    parser = QueryParser()
    
    intent = parser.parse_intent(user_query)
    # Supports queries like:
    # "微博今天有什么热搜?" (What's trending on Weibo today?)
    # "关于人工智能的新闻" (News about AI)
    # "分析一下科技类话题的情感" (Analyze sentiment of tech topics)
    
    if intent['type'] == 'platform_query':
        results = fetch_platform_data(intent['platform'])
    elif intent['type'] == 'topic_search':
        results = search_topic(intent['keywords'])
    elif intent['type'] == 'sentiment_analysis':
        results = analyze_sentiment_by_topic(intent['topic'])
    
    return jsonify(results)
```

### 4. Multi-Channel Notifications

```python
from hotsearch_analysis_agent.notifier import NotificationManager

notifier = NotificationManager()

# Email notification
notifier.send_email(
    subject='热点话题推送: AI技术突破',
    body=report_content,
    recipients=['user@example.com']
)

# WeChat Work Bot
notifier.send_wework_bot(
    webhook_url=os.getenv('WEWORK_BOT_WEBHOOK'),
    content=report_content
)

# Telegram
notifier.send_telegram(
    bot_token=os.getenv('TELEGRAM_BOT_TOKEN'),
    chat_id=os.getenv('TELEGRAM_CHAT_ID'),
    message=report_content
)

# Schedule periodic push tasks
from apscheduler.schedulers.background import BackgroundScheduler

scheduler = BackgroundScheduler()
scheduler.add_job(
    func=lambda: notifier.push_hot_topics(
        query='科技',
        channels=['email', 'wework', 'telegram']
    ),
    trigger='cron',
    hour=9,  # Daily at 9 AM
    minute=0
)
scheduler.start()
```

## Common Usage Patterns

### Pattern 1: Monitor Specific Topics

```python
from hotsearch_analysis_agent.monitor import TopicMonitor

monitor = TopicMonitor(db_config={
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
})

# Track keyword across all platforms
monitor.track_keywords(
    keywords=['人工智能', 'ChatGPT', '大模型'],
    platforms=['weibo', 'zhihu', 'bilibili'],
    notification_threshold=1000  # Alert when hot value > 1000
)

# Get aggregated results
results = monitor.get_tracked_topics(time_range='24h')
```

### Pattern 2: Generate Periodic Reports

```python
from hotsearch_analysis_agent.reporter import ReportGenerator

reporter = ReportGenerator(
    llm_config={'api_key': os.getenv('LLM_API_KEY')}
)

# Daily tech news digest
daily_report = reporter.generate_digest(
    categories=['科技', '互联网', 'AI'],
    date='2026-04-07',
    format='markdown'  # or 'html', 'pdf'
)

# Save or send report
with open('daily_digest.md', 'w', encoding='utf-8') as f:
    f.write(daily_report)
```

### Pattern 3: Real-time Crawler Control

```python
# Via keyboard shortcuts (configured in frontend)
# F5: Start all crawlers
# F6: Stop all crawlers

# Programmatic control
from hotsearchcrawler.controller import CrawlerController

controller = CrawlerController()

# Start specific crawlers
controller.start_crawlers(['weibo', 'bilibili', 'zhihu'])

# Stop all
controller.stop_all()

# Get crawler status
status = controller.get_status()
print(status)
# {'weibo': 'running', 'bilibili': 'running', 'zhihu': 'stopped'}
```

### Pattern 4: Topic Clustering Analysis

```python
from hotsearch_analysis_agent.clustering import TopicClusterer

clusterer = TopicClusterer()

# Fetch recent data
from hotsearch_analysis_agent.database import DatabaseManager
db = DatabaseManager()
recent_topics = db.fetch_topics(hours=24)

# Cluster similar topics
clusters = clusterer.cluster(
    topics=recent_topics,
    algorithm='kmeans',  # or 'dbscan', 'hierarchical'
    num_clusters=5
)

for cluster_id, topics in clusters.items():
    print(f"Cluster {cluster_id}:")
    for topic in topics:
        print(f"  - {topic['title']} ({topic['platform']})")
```

## Configuration Examples

### Push Task Configuration

```python
# Configure scheduled push tasks
push_config = {
    'tasks': [
        {
            'name': 'morning_tech_digest',
            'query': '科技 人工智能',
            'schedule': 'cron',
            'hour': 9,
            'minute': 0,
            'channels': ['email', 'wework'],
            'platforms': ['weibo', 'zhihu', 'bilibili']
        },
        {
            'name': 'hot_topic_alert',
            'query': '*',  # All topics
            'schedule': 'interval',
            'minutes': 30,
            'channels': ['telegram'],
            'threshold': 5000  # Hot value threshold
        }
    ]
}

from hotsearch_analysis_agent.scheduler import TaskScheduler
scheduler = TaskScheduler(push_config)
scheduler.start()
```

### Crawler Settings

```python
# hotsearchcrawler/settings.py

# Concurrent requests
CONCURRENT_REQUESTS = 16
CONCURRENT_REQUESTS_PER_DOMAIN = 8

# Delays
DOWNLOAD_DELAY = 1
RANDOMIZE_DOWNLOAD_DELAY = True

# User agents
USER_AGENT = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'

# Pipelines
ITEM_PIPELINES = {
    'hotsearchcrawler.pipelines.MysqlPipeline': 300,
    'hotsearchcrawler.pipelines.ContentEnrichmentPipeline': 400,
}

# Retry settings
RETRY_TIMES = 3
RETRY_HTTP_CODES = [500, 502, 503, 504, 408]
```

## Troubleshooting

### Issue: Crawler fails to start

**Solution:**
```bash
# Check WebDriver installation
which chromedriver  # or msedgedriver

# Verify browser version matches driver
chromedriver --version
google-chrome --version

# Update driver if needed
```

### Issue: Database connection error

**Solution:**
```python
# Test database connection
import pymysql

try:
    conn = pymysql.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    print("Database connected successfully")
    conn.close()
except Exception as e:
    print(f"Connection failed: {e}")
```

### Issue: LLM API timeout or rate limit

**Solution:**
```python
# Implement retry logic with exponential backoff
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=4, max=10)
)
def call_llm_api(prompt):
    # Your API call here
    pass
```

### Issue: Platform returns anti-crawler measures

**Solution:**
```python
# Add cookies and headers in settings.py
COOKIES = {
    'platform_name': 'your_authenticated_cookie'
}

# Increase download delay
DOWNLOAD_DELAY = 3

# Rotate user agents
from scrapy.downloadermiddlewares.useragent import UserAgentMiddleware
# Configure in middlewares
```

### Issue: Video content extraction fails

**Solution:**
```python
# Use selenium for dynamic content
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

driver = webdriver.Chrome()
driver.get(video_url)

# Wait for video description to load
description = WebDriverWait(driver, 10).until(
    EC.presence_of_element_located((By.CLASS_NAME, 'video-desc'))
)
content = description.text
driver.quit()
```

## Advanced Features

### Custom LLM Integration

```python
# Use Huawei Pangu model locally
from hotsearch_analysis_agent.models.pangu import PanguModel

model = PanguModel(model_path=os.getenv('PANGU_MODEL_PATH'))

# Inference
result = model.generate(
    prompt="分析以下新闻的情感倾向:\n{news_content}",
    max_length=512
)
```

### API Endpoints

```python
# Main API routes (app.py)

# Query hot searches
GET /api/hotsearch?platform=weibo&limit=20

# Search by keyword
POST /api/search
{
    "query": "人工智能",
    "platforms": ["weibo", "zhihu"],
    "time_range": "24h"
}

# Sentiment analysis
POST /api/analyze/sentiment
{
    "text": "这是一条测试文本"
}

# Topic clustering
POST /api/analyze/cluster
{
    "topic_ids": [1, 2, 3, 4, 5],
    "num_clusters": 3
}

# Start/Stop crawler
POST /api/crawler/control
{
    "action": "start",  # or "stop"
    "spiders": ["weibo", "bilibili"]
}
```

This skill provides comprehensive guidance for deploying and using the LLM-Based Intelligent Public Opinion Analytics Assistant in real-world scenarios.
