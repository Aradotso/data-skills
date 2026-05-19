---
name: llm-public-opinion-analytics-assistant
description: Multi-platform hot search data crawler and LLM-powered public opinion analysis system with sentiment analysis, topic clustering, and multi-channel push notifications
triggers:
  - analyze public opinion from social media platforms
  - set up hot topic monitoring and alerts
  - crawl trending news from multiple platforms
  - perform sentiment analysis on news articles
  - configure push notifications for trending topics
  - cluster and analyze social media discussions
  - monitor real-time hot search rankings
  - extract insights from video content news
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

An intelligent public opinion analysis system that crawls real-time data from 15 mainstream platforms (26 ranking lists) and uses LLM capabilities for sentiment analysis, topic clustering, and multi-channel alert notifications. Supports conversational queries, hotkey-controlled crawlers, and detailed content extraction even from video-based news.

## What It Does

- **Multi-Platform Data Collection**: Crawls hot search rankings from 15 platforms including Weibo, Bilibili, Douyin, Baidu, etc.
- **LLM-Powered Analysis**: Sentiment analysis, topic clustering, and trend identification using large language models
- **Conversational Interface**: Natural language queries for hot topics and trend analysis
- **Content Extraction**: Deep crawling of news detail pages including video content transcription
- **Multi-Channel Push**: Supports Email, WeChat Work, Enterprise WeChat, and Telegram notifications
- **Real-Time Monitoring**: Hotkey-controlled crawler management with scheduled task support

## Installation

### Prerequisites

**1. Browser Driver Setup**

For Chrome:
```bash
# Check your Chrome version first (Settings → About)
# Download matching ChromeDriver from https://chromedriver.chromium.org/
# Place in system PATH or project directory
```

For Edge:
```bash
# Check your Edge version
# Download matching EdgeDriver from https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/
```

**2. Database Setup**

Install MySQL and create the database:
```sql
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

Reference `init.py` for table schema initialization.

**3. Python Environment**

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Configuration

**1. Crawler Configuration** (`hotsearchcrawler/settings.py`):

```python
# MySQL Configuration
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_user'
MYSQL_PASSWORD = 'your_password'
MYSQL_DATABASE = 'hotsearch_db'

# Optional: Platform-specific cookies
COOKIES = {
    'weibo': 'your_cookies_here',
    'bilibili': 'your_cookies_here'
}
```

**2. Analysis System Configuration** (`.env` file):

```env
# Database
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key_here
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Push Notification Channels
# Email (SMTP)
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password

# WeChat Work Bot
WECHAT_WORK_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

## Core Usage Patterns

### Starting the Application

```bash
# Start the main application server
python app.py
```

### Running Crawlers

**Manual Test Run**:
```bash
# Test specific crawler
python runspider-test.py
```

**Production Run** (controlled via frontend or programmatically):
```python
from hotsearchcrawler.run_spiders import CrawlerManager

manager = CrawlerManager()

# Start all crawlers
manager.start_all_crawlers()

# Start specific platform crawler
manager.start_crawler('weibo')
manager.start_crawler('bilibili')

# Stop crawlers
manager.stop_all_crawlers()
```

### Data Query and Analysis

```python
from hotsearch_analysis_agent.analyzer import OpinionAnalyzer
from hotsearch_analysis_agent.database import DatabaseConnector

# Initialize analyzer with LLM
analyzer = OpinionAnalyzer(
    api_key=os.getenv('OPENAI_API_KEY'),
    model_name=os.getenv('MODEL_NAME', 'gpt-4')
)

# Query hot topics
db = DatabaseConnector()
topics = db.query_hot_topics(
    platform='weibo',
    limit=50,
    start_date='2026-05-01'
)

# Perform sentiment analysis
sentiment_results = analyzer.analyze_sentiment(topics)
print(f"Positive: {sentiment_results['positive']}%")
print(f"Negative: {sentiment_results['negative']}%")
print(f"Neutral: {sentiment_results['neutral']}%")

# Topic clustering
clusters = analyzer.cluster_topics(topics, num_clusters=5)
for cluster_id, cluster_topics in clusters.items():
    print(f"Cluster {cluster_id}: {len(cluster_topics)} topics")
    print(f"Main theme: {cluster_topics['theme']}")
```

### Content Extraction from Detail Pages

```python
from hotsearchcrawler.detail_crawler import DetailCrawler

crawler = DetailCrawler()

# Extract content from news URL (including video content)
content = crawler.extract_content(
    url='https://www.bilibili.com/video/BV13pSoBBEvX/',
    content_type='video'
)

print(f"Title: {content['title']}")
print(f"Summary: {content['summary']}")
print(f"Transcript: {content['transcript']}")  # For video content
```

### Setting Up Push Notifications

```python
from hotsearch_analysis_agent.push_notifier import PushNotifier

notifier = PushNotifier()

# Create a monitoring task
task = {
    'name': '人工智能与前沿科技监测',
    'keywords': ['人工智能', 'AI', 'GPT', '大模型'],
    'platforms': ['weibo', 'zhihu', 'bilibili'],
    'channels': ['email', 'wechat_work', 'telegram'],
    'frequency': 'daily',  # daily, hourly, real-time
    'threshold': {
        'min_heat': 10000,  # Minimum heat score
        'sentiment_change': 0.3  # Alert if sentiment changes by 30%
    }
}

# Register task
notifier.create_task(task)

# Test push notification
notifier.send_test_notification(
    channel='email',
    recipient=os.getenv('SMTP_USER')
)
```

### Conversational Query Interface

```python
from hotsearch_analysis_agent.chat_interface import ChatInterface

chat = ChatInterface()

# Natural language queries
response = chat.query("最近关于人工智能的热点新闻有哪些?")
print(response)

# Topic-specific search
response = chat.query("分析一下DeepSeek V4的舆情趋势")
print(response)

# Multi-platform comparison
response = chat.query("对比微博和知乎上关于AI的讨论情感倾向")
print(response)
```

## Advanced Patterns

### Custom Analysis Pipeline

```python
from hotsearch_analysis_agent.pipeline import AnalysisPipeline

pipeline = AnalysisPipeline()

# Define custom analysis workflow
pipeline.add_step('fetch', fetch_params={'platforms': ['weibo', 'zhihu']})
pipeline.add_step('filter', filter_params={'min_heat': 50000})
pipeline.add_step('extract_details', parallel=True, max_workers=5)
pipeline.add_step('sentiment_analysis', model='gpt-4')
pipeline.add_step('cluster', num_clusters=10)
pipeline.add_step('generate_report', template='detailed')
pipeline.add_step('push', channels=['email', 'telegram'])

# Execute pipeline
results = pipeline.run(
    query="人工智能",
    start_date="2026-05-01",
    end_date="2026-05-15"
)

print(f"Report generated: {results['report_path']}")
```

### Scheduled Monitoring with Custom Rules

```python
from hotsearch_analysis_agent.scheduler import TaskScheduler

scheduler = TaskScheduler()

# Define monitoring rule
rule = {
    'name': 'AI突发事件监测',
    'condition': {
        'heat_spike': 5.0,  # 5x increase in heat
        'time_window': '1h',
        'keywords': ['AI', '人工智能', '大模型'],
        'exclude_keywords': ['广告', '营销']
    },
    'action': {
        'analyze': True,
        'generate_report': True,
        'push_immediately': True,
        'channels': ['telegram', 'wechat_work']
    }
}

scheduler.add_rule(rule)
scheduler.start()
```

### Batch Processing Historical Data

```python
from hotsearch_analysis_agent.batch_processor import BatchProcessor
from datetime import datetime, timedelta

processor = BatchProcessor()

# Analyze historical trends
end_date = datetime.now()
start_date = end_date - timedelta(days=30)

trend_analysis = processor.analyze_trends(
    keywords=['AI', 'ChatGPT', 'DeepSeek'],
    platforms=['all'],
    start_date=start_date,
    end_date=end_date,
    granularity='daily'
)

# Export results
processor.export_to_csv(trend_analysis, 'ai_trends_30days.csv')
processor.export_to_json(trend_analysis, 'ai_trends_30days.json')

# Generate visualization
processor.plot_trends(
    trend_analysis,
    output='ai_trends_chart.png',
    chart_type='line'
)
```

## Troubleshooting

### Browser Driver Issues

**Problem**: `selenium.common.exceptions.WebDriverException: Message: 'chromedriver' executable needs to be in PATH`

**Solution**:
```bash
# Verify driver is in PATH
which chromedriver  # Unix
where chromedriver  # Windows

# Or set explicitly in code
from selenium import webdriver
from selenium.webdriver.chrome.service import Service

service = Service(executable_path='/path/to/chromedriver')
driver = webdriver.Chrome(service=service)
```

### Database Connection Errors

**Problem**: `pymysql.err.OperationalError: (2003, "Can't connect to MySQL server")`

**Solution**:
```python
# Verify connection settings
import pymysql

try:
    connection = pymysql.connect(
        host=os.getenv('MYSQL_HOST'),
        port=int(os.getenv('MYSQL_PORT')),
        user=os.getenv('MYSQL_USER'),
        password=os.getenv('MYSQL_PASSWORD'),
        database=os.getenv('MYSQL_DATABASE'),
        charset='utf8mb4'
    )
    print("Database connection successful")
except Exception as e:
    print(f"Connection error: {e}")
```

### LLM API Rate Limits

**Problem**: Rate limit exceeded when processing large batches

**Solution**:
```python
from hotsearch_analysis_agent.analyzer import OpinionAnalyzer
import time

analyzer = OpinionAnalyzer(
    rate_limit=10,  # Max 10 requests per minute
    retry_on_limit=True,
    retry_delay=60
)

# Process with batching
for batch in analyzer.batch_process(topics, batch_size=5):
    results = analyzer.analyze_sentiment(batch)
    time.sleep(6)  # Respect rate limits
```

### Cookie Expiration

**Problem**: Platform returns 401/403 when crawling

**Solution**:
```python
# Update cookies in settings
# hotsearchcrawler/settings.py

def refresh_cookies():
    """Manually refresh cookies from browser"""
    # Use browser extension to export cookies
    # Or use selenium to login and extract cookies
    from selenium import webdriver
    
    driver = webdriver.Chrome()
    driver.get('https://weibo.com')
    # Login manually
    input("Press Enter after login...")
    
    cookies = driver.get_cookies()
    driver.quit()
    
    return cookies

# Auto-refresh in crawler middleware
DOWNLOADER_MIDDLEWARES = {
    'hotsearchcrawler.middlewares.CookieRefreshMiddleware': 543,
}
```

### Push Notification Failures

**Problem**: Notifications not being delivered

**Solution**:
```python
from hotsearch_analysis_agent.push_notifier import PushNotifier

notifier = PushNotifier(debug=True)

# Test each channel individually
test_results = notifier.test_all_channels()

for channel, status in test_results.items():
    if not status['success']:
        print(f"{channel} failed: {status['error']}")
        # Check specific config
        if channel == 'email':
            print(f"SMTP settings: {os.getenv('SMTP_SERVER')}")
        elif channel == 'telegram':
            print(f"Bot token: {os.getenv('TELEGRAM_BOT_TOKEN')[:10]}...")
```

## Key Commands Summary

```bash
# Start main application
python app.py

# Test crawlers
python runspider-test.py

# Initialize database
python init.py

# Test push notifications
python test_push_task.py

# Run specific crawler (programmatic)
python -c "from hotsearchcrawler.run_spiders import CrawlerManager; CrawlerManager().start_crawler('weibo')"
```

## Integration with Huawei Pangu Model (Optional)

For local deployment with Huawei's Pangu model:

```python
from hotsearch_analysis_agent.models import PanguModel

# Initialize Pangu model
model = PanguModel(
    model_path='/path/to/openpangu-embedded-7b-model',
    device='cuda'  # or 'cpu'
)

# Use in analyzer
analyzer = OpinionAnalyzer(model=model)

# Perform analysis with local model
sentiment = analyzer.analyze_sentiment(topics, use_local=True)
clusters = analyzer.cluster_topics(topics, use_local=True)
```

Download Pangu model from: https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model
