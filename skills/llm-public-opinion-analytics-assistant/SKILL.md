---
name: llm-public-opinion-analytics-assistant
description: A real-time public opinion analysis assistant that aggregates 26 trending lists from 15 platforms with LLM-powered sentiment analysis, topic clustering, and multi-channel alerting
triggers:
  - set up public opinion monitoring system
  - analyze trending topics across multiple platforms
  - create sentiment analysis with LLM for social media
  - configure hot topic crawler and alert system
  - build multi-platform trending news aggregator
  - implement topic clustering and sentiment analysis
  - deploy opinion analytics with push notifications
  - scrape and analyze trending lists with AI
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is an intelligent public opinion analysis assistant that combines real-time data from **26 trending lists across 15 major platforms** (Weibo, Bilibili, Douyin, Baidu, etc.) with large language model analysis capabilities. It provides conversational hot search queries, topic clustering, sentiment analysis, and multi-channel push notifications (Email, WeChat, Telegram).

## What It Does

- **Multi-Platform Data Aggregation**: Crawls trending lists from 15 Chinese platforms including Weibo, Bilibili, Douyin, Zhihu, Baidu, Toutiao, and more
- **LLM-Powered Analysis**: Uses large language models (recommends Huawei Pangu model) for sentiment analysis, topic clustering, and trend summarization
- **Conversational Interface**: Natural language queries for trending topics, theme searches, and analysis
- **Content Extraction**: Extracts full article/video content from detail pages for deep analysis
- **Multi-Channel Alerts**: Push notifications via Enterprise WeChat, Telegram, Email (SMTP)
- **Hotkey Controls**: Start/stop crawlers with keyboard shortcuts
- **Scheduled Reports**: Automated periodic analysis reports on specific topics

## Architecture

The project consists of two main components:

1. **Analysis System** (`hotsearch_analysis_agent/`): Frontend interface, LLM integration, data analysis, push notifications
2. **Crawler Cluster** (`hotsearchcrawler/`): Scrapy-based distributed crawlers for 15 platforms

## Installation

### Prerequisites

**Browser Driver Setup** (required for content extraction):

```bash
# For Chrome/Chromium
# 1. Check your browser version: chrome://version/
# 2. Download matching ChromeDriver from https://chromedriver.chromium.org/
# 3. Place driver in system PATH or project directory

# For Edge
# Download from https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Verify installation
chromedriver --version
# or
msedgedriver --version
```

**Database Setup**:

```bash
# Install MySQL 5.7+
# Create database and tables using init.py as reference
```

### Install Dependencies

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install requirements
pip install -r requirements.txt
```

### Configuration

**1. Database Configuration** (`hotsearchcrawler/settings.py`):

```python
# MySQL settings for crawler
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_user'
MYSQL_PASSWORD = 'your_password'
MYSQL_DATABASE = 'hotsearch_db'
```

**2. Environment Variables** (`.env` file in project root):

```bash
# Database
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch_db

# LLM API (OpenAI-compatible format)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Or use Huawei Pangu model (recommended)
# Download from: https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model
PANGU_MODEL_PATH=/path/to/pangu/model

# Email notifications (optional)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
NOTIFICATION_EMAIL=recipient@example.com

# Telegram Bot (optional)
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Enterprise WeChat (optional)
WECHAT_WEBHOOK_URL=your_webhook_url
WECHAT_APP_ID=your_app_id
WECHAT_APP_SECRET=your_app_secret
```

**3. Initialize Database**:

```python
# Reference init.py for table schemas
from init import initialize_database

initialize_database()
```

## Running the System

### Start the Analysis System

```bash
# Start the web interface and API server
python app.py

# Access at http://localhost:5000
```

### Start Crawlers

**Via Web Interface** (recommended):
- Navigate to the crawler control panel
- Use hotkeys or buttons to start/stop specific platform crawlers

**Command Line**:

```bash
# Test single crawler
python runspider-test.py

# Start all crawlers
python run_spiders.py

# Start specific platform
cd hotsearchcrawler
scrapy crawl weibo_spider
scrapy crawl bilibili_spider
scrapy crawl douyin_spider
```

## Key Usage Patterns

### 1. Query Trending Topics via Natural Language

```python
# In the web interface, ask questions like:
"今天微博热搜榜前十是什么?"
"帮我找找关于人工智能的热点新闻"
"分析一下最近科技类话题的情感倾向"
"哪些平台在讨论ChatGPT?"
```

### 2. Programmatic Data Access

```python
from hotsearch_analysis_agent.database import get_trending_data
from datetime import datetime, timedelta

# Get trending data from last 24 hours
end_time = datetime.now()
start_time = end_time - timedelta(days=1)

data = get_trending_data(
    platforms=['weibo', 'bilibili', 'douyin'],
    start_time=start_time,
    end_time=end_time,
    limit=50
)

for item in data:
    print(f"{item['title']} - {item['platform']} - {item['hot_score']}")
```

### 3. Topic Clustering Analysis

```python
from hotsearch_analysis_agent.analysis import TopicAnalyzer

analyzer = TopicAnalyzer(model='pangu')  # or 'gpt-4'

# Analyze topics from last 24 hours
results = analyzer.cluster_topics(
    query="人工智能",
    time_range=24,  # hours
    min_cluster_size=3
)

for cluster in results['clusters']:
    print(f"\n话题簇: {cluster['theme']}")
    print(f"相关新闻数: {len(cluster['articles'])}")
    print(f"情感倾向: {cluster['sentiment']}")
    for article in cluster['articles'][:5]:
        print(f"  - {article['title']} ({article['platform']})")
```

### 4. Sentiment Analysis

```python
from hotsearch_analysis_agent.analysis import SentimentAnalyzer

sentiment_analyzer = SentimentAnalyzer(model='pangu')

# Analyze sentiment for specific topic
sentiment = sentiment_analyzer.analyze_topic(
    topic="电动汽车降价",
    platforms=['weibo', 'zhihu', 'toutiao']
)

print(f"正面: {sentiment['positive']}%")
print(f"中性: {sentiment['neutral']}%")
print(f"负面: {sentiment['negative']}%")
print(f"关键观点: {sentiment['key_points']}")
```

### 5. Create Push Notification Task

```python
from hotsearch_analysis_agent.push_tasks import create_push_task

# Create scheduled analysis report
task = create_push_task(
    name="AI科技日报",
    query="人工智能 OR 大模型 OR ChatGPT",
    schedule="0 9 * * *",  # Daily at 9 AM (cron format)
    channels=['email', 'telegram', 'wechat'],
    analysis_type='comprehensive',  # or 'sentiment', 'cluster'
    platforms=['weibo', 'bilibili', 'zhihu']
)

print(f"Task created: {task.id}")
```

### 6. Test Push Notifications

```python
# Run the test file to verify push channels
python test_push_task.py

# Or programmatically
from hotsearch_analysis_agent.notifications import send_notification

send_notification(
    title="测试通知",
    content="这是一条测试消息",
    channels=['email', 'telegram'],
    priority='high'
)
```

### 7. Custom Crawler for New Platform

```python
# hotsearchcrawler/spiders/new_platform_spider.py
import scrapy
from hotsearchcrawler.items import HotSearchItem

class NewPlatformSpider(scrapy.Spider):
    name = 'newplatform_spider'
    start_urls = ['https://newplatform.com/trending']
    
    def parse(self, response):
        for item in response.css('.trending-item'):
            hot_item = HotSearchItem()
            hot_item['platform'] = 'newplatform'
            hot_item['title'] = item.css('.title::text').get()
            hot_item['url'] = item.css('a::attr(href)').get()
            hot_item['hot_score'] = item.css('.score::text').get()
            hot_item['rank'] = item.css('.rank::text').get()
            hot_item['crawl_time'] = datetime.now()
            
            # Follow detail page for full content
            yield scrapy.Request(
                hot_item['url'],
                callback=self.parse_detail,
                meta={'item': hot_item}
            )
    
    def parse_detail(self, response):
        item = response.meta['item']
        item['content'] = response.css('.article-content::text').getall()
        item['author'] = response.css('.author::text').get()
        item['publish_time'] = response.css('.time::text').get()
        yield item
```

## Configuration Reference

### Crawler Settings (`hotsearchcrawler/settings.py`)

```python
# Scrapy settings
BOT_NAME = 'hotsearchcrawler'
SPIDER_MODULES = ['hotsearchcrawler.spiders']

# Concurrency
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1  # seconds between requests

# User-Agent rotation
USER_AGENT = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'

# Pipelines (order matters)
ITEM_PIPELINES = {
    'hotsearchcrawler.pipelines.ContentExtractorPipeline': 100,
    'hotsearchcrawler.pipelines.DeduplicationPipeline': 200,
    'hotsearchcrawler.pipelines.MySQLPipeline': 300,
}

# Middleware
DOWNLOADER_MIDDLEWARES = {
    'hotsearchcrawler.middlewares.RandomUserAgentMiddleware': 400,
    'hotsearchcrawler.middlewares.ProxyMiddleware': 410,
}

# Browser settings for content extraction
SELENIUM_DRIVER_NAME = 'chrome'  # or 'edge'
SELENIUM_DRIVER_EXECUTABLE_PATH = '/path/to/chromedriver'
SELENIUM_BROWSER_EXECUTABLE_PATH = '/path/to/chrome'
```

### Analysis System Configuration

```python
# hotsearch_analysis_agent/config.py
LLM_CONFIG = {
    'provider': 'pangu',  # or 'openai', 'local'
    'model_path': '/path/to/pangu/model',
    'max_tokens': 2048,
    'temperature': 0.7,
    'context_window': 200000  # 200K for long context
}

ANALYSIS_CONFIG = {
    'clustering': {
        'min_similarity': 0.7,
        'min_cluster_size': 3,
        'method': 'hierarchical'  # or 'kmeans', 'dbscan'
    },
    'sentiment': {
        'model': 'pangu',
        'granularity': 'sentence',  # or 'document'
        'return_scores': True
    }
}

NOTIFICATION_CONFIG = {
    'rate_limit': 10,  # max per hour
    'batch_size': 5,  # combine multiple alerts
    'quiet_hours': (22, 8)  # no notifications from 10PM to 8AM
}
```

## Common Patterns

### Pattern 1: Daily Monitoring Report

```python
from hotsearch_analysis_agent import ReportGenerator
from datetime import datetime, timedelta

generator = ReportGenerator()

report = generator.create_daily_report(
    topics=["人工智能", "新能源汽车", "经济政策"],
    date=datetime.now() - timedelta(days=1),
    include_sentiment=True,
    include_clustering=True,
    platforms=['weibo', 'bilibili', 'zhihu', 'toutiao']
)

# Push via multiple channels
report.push(channels=['email', 'wechat'])

# Save to file
report.save('reports/daily_report_20260517.md')
```

### Pattern 2: Real-time Alert on Keywords

```python
from hotsearch_analysis_agent.monitoring import KeywordMonitor

monitor = KeywordMonitor(
    keywords=["公司名称", "品牌危机", "产品召回"],
    platforms=['weibo', 'zhihu', 'douyin'],
    alert_threshold=1000,  # hot score threshold
    check_interval=300  # check every 5 minutes
)

@monitor.on_match
def handle_alert(item):
    print(f"ALERT: {item['title']} on {item['platform']}")
    print(f"Hot score: {item['hot_score']}")
    
    # Immediate notification
    send_notification(
        title=f"舆情预警: {item['title']}",
        content=f"平台: {item['platform']}\n热度: {item['hot_score']}\n链接: {item['url']}",
        channels=['telegram', 'wechat'],
        priority='urgent'
    )

monitor.start()
```

### Pattern 3: Topic Tracking Over Time

```python
from hotsearch_analysis_agent.tracking import TopicTracker

tracker = TopicTracker(topic="GPT-5发布")

# Track for 7 days
tracker.start(duration_days=7, interval_hours=6)

# Get timeline data
timeline = tracker.get_timeline()

for snapshot in timeline:
    print(f"\n{snapshot['timestamp']}")
    print(f"讨论量: {snapshot['volume']}")
    print(f"热度趋势: {snapshot['trend']}")  # rising, stable, declining
    print(f"主要平台: {snapshot['top_platforms']}")
    print(f"情感变化: {snapshot['sentiment_shift']}")

# Generate trend chart
tracker.plot_trend(save_path='trends/gpt5_tracking.png')
```

## Troubleshooting

### Crawler Issues

**Problem**: ChromeDriver version mismatch
```bash
# Error: session not created: This version of ChromeDriver only supports Chrome version XX
# Solution: Download matching driver version
wget https://chromedriver.storage.googleapis.com/LATEST_RELEASE_XX
```

**Problem**: High failure rate on specific platform
```python
# Check crawler logs
tail -f hotsearchcrawler/logs/weibo_spider.log

# Adjust settings
# In settings.py, increase delay and reduce concurrency
DOWNLOAD_DELAY = 3
CONCURRENT_REQUESTS = 4

# Add cookies for authenticated access (optional)
# In spiders/weibo_spider.py
custom_settings = {
    'COOKIES_ENABLED': True,
    'COOKIES_DEBUG': True
}
```

**Problem**: Content extraction fails for video platforms
```python
# Enable JavaScript rendering
# In pipelines.py
from selenium import webdriver
from selenium.webdriver.support.ui import WebDriverWait

class ContentExtractorPipeline:
    def process_item(self, item, spider):
        if item['platform'] in ['bilibili', 'douyin']:
            driver = webdriver.Chrome()
            driver.get(item['url'])
            WebDriverWait(driver, 10).until(
                lambda d: d.find_element_by_class_name('video-title')
            )
            item['content'] = driver.find_element_by_class_name('description').text
            driver.quit()
        return item
```

### LLM Analysis Issues

**Problem**: Out of memory with Pangu model
```python
# Reduce batch size and context length
LLM_CONFIG = {
    'max_tokens': 1024,  # reduced from 2048
    'batch_size': 1,
    'use_gpu': True,
    'gpu_memory_fraction': 0.8
}
```

**Problem**: Slow analysis performance
```python
# Use streaming for real-time results
from hotsearch_analysis_agent.analysis import StreamingAnalyzer

analyzer = StreamingAnalyzer()
for chunk in analyzer.analyze_stream(topic="AI发展"):
    print(chunk, end='', flush=True)
```

### Database Issues

**Problem**: Connection pool exhausted
```python
# In .env
MYSQL_POOL_SIZE=20
MYSQL_MAX_OVERFLOW=10
MYSQL_POOL_RECYCLE=3600
```

**Problem**: Duplicate entries
```python
# Enable deduplication pipeline
# In settings.py
ITEM_PIPELINES = {
    'hotsearchcrawler.pipelines.DeduplicationPipeline': 200,  # runs before MySQL
}

# Check duplicate detection logic
# In pipelines.py
class DeduplicationPipeline:
    def process_item(self, item, spider):
        item_hash = hashlib.md5(
            f"{item['platform']}{item['title']}{item['url']}".encode()
        ).hexdigest()
        
        if self.redis_client.exists(item_hash):
            raise DropItem(f"Duplicate: {item['title']}")
        
        self.redis_client.setex(item_hash, 86400, 1)  # 24h TTL
        return item
```

### Push Notification Issues

**Problem**: Telegram bot not responding
```bash
# Test bot token
curl https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/getMe

# Get chat ID
curl https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/getUpdates
```

**Problem**: Email stuck in spam
```python
# Add proper headers
# In notifications/email.py
msg['From'] = formataddr(('舆情分析助手', SMTP_USER))
msg['Subject'] = Header(subject, 'utf-8')
msg['X-Priority'] = '3'
msg['X-Mailer'] = 'HotSearch Analytics'
```

## Advanced Usage

### Custom LLM Integration

```python
# hotsearch_analysis_agent/llm/custom_provider.py
from hotsearch_analysis_agent.llm.base import BaseLLMProvider

class CustomLLMProvider(BaseLLMProvider):
    def __init__(self, config):
        self.api_url = config['api_url']
        self.api_key = config['api_key']
    
    def generate(self, prompt, **kwargs):
        response = requests.post(
            self.api_url,
            json={
                'prompt': prompt,
                'max_tokens': kwargs.get('max_tokens', 1024),
                'temperature': kwargs.get('temperature', 0.7)
            },
            headers={'Authorization': f'Bearer {self.api_key}'}
        )
        return response.json()['text']

# Register provider
# In config.py
LLM_PROVIDERS = {
    'custom': CustomLLMProvider
}
```

### Distributed Crawler Deployment

```bash
# Deploy scrapy with scrapyd
pip install scrapyd scrapyd-client

# Start scrapyd server
scrapyd

# Deploy project
scrapyd-deploy

# Schedule spider
curl http://localhost:6800/schedule.json -d project=hotsearchcrawler -d spider=weibo_spider
```

This skill provides comprehensive guidance for deploying and using the LLM-Based Public Opinion Analytics Assistant for real-time trending topic monitoring, sentiment analysis, and intelligent alerting across Chinese social media platforms.
