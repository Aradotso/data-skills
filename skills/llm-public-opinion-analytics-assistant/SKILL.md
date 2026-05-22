---
name: llm-public-opinion-analytics-assistant
description: Multi-platform public opinion analysis assistant that crawls 26 hot search lists from 15 platforms, analyzes with LLM, performs sentiment analysis, topic clustering, and sends multi-channel alerts
triggers:
  - set up public opinion monitoring system
  - crawl hot search data from multiple platforms
  - analyze sentiment and cluster topics with LLM
  - configure hot topic push notifications
  - build real-time social media monitoring
  - create intelligent opinion analytics dashboard
  - integrate PanGu model for Chinese text analysis
  - set up automated trend reporting system
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is an intelligent public opinion analysis assistant that integrates real-time data from **26 hot search lists** across **15 mainstream platforms** (Weibo, Bilibili, Baidu, Zhihu, Douyin, etc.) with large language model analysis capabilities. It provides conversational queries, topic clustering, sentiment analysis, and multi-channel alert push (WeChat, Email, Telegram).

**Key Features:**
- Multi-platform web scraping with Scrapy
- LLM-powered sentiment and topic analysis (supports PanGu, OpenAI-compatible APIs)
- Natural language querying of trending topics
- Automated clustering and reporting
- Multi-channel push notifications (Enterprise WeChat, Telegram, Email)
- Keyboard shortcuts for crawler control
- Video content extraction support

## Installation

### Prerequisites

1. **Browser Driver Setup** (required for detail page scraping):
   ```bash
   # Download ChromeDriver or EdgeDriver matching your browser version
   # Chrome: https://chromedriver.chromium.org/
   # Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/
   
   # Add driver to PATH or place in browser directory
   chromedriver --version  # Verify installation
   ```

2. **MySQL Database**:
   ```bash
   # Install MySQL and create database
   mysql -u root -p
   CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
   ```

3. **Python Environment**:
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

1. **Database Initialization**:
   ```python
   # Reference init.py for table structure
   python init.py
   ```

2. **Crawler Configuration** (`hotsearchcrawler/settings.py`):
   ```python
   # MySQL settings
   MYSQL_HOST = 'localhost'
   MYSQL_PORT = 3306
   MYSQL_USER = 'your_user'
   MYSQL_PASSWORD = 'your_password'
   MYSQL_DATABASE = 'hotsearch_db'
   
   # Optional: Add platform cookies for authenticated access
   COOKIES = {
       'weibo': 'your_cookies_here',
       # ...
   }
   ```

3. **Analysis System Configuration** (`.env` file):
   ```bash
   # MySQL
   MYSQL_HOST=localhost
   MYSQL_PORT=3306
   MYSQL_USER=your_user
   MYSQL_PASSWORD=your_password
   MYSQL_DATABASE=hotsearch_db
   
   # LLM API (OpenAI-compatible)
   OPENAI_API_KEY=your_api_key
   OPENAI_API_BASE=https://api.openai.com/v1
   MODEL_NAME=gpt-4
   
   # Or use PanGu model (local deployment)
   USE_PANGU=true
   PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model
   
   # Push Notifications
   WECHAT_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=your_key
   TELEGRAM_BOT_TOKEN=your_bot_token
   TELEGRAM_CHAT_ID=your_chat_id
   SMTP_HOST=smtp.gmail.com
   SMTP_PORT=587
   SMTP_USER=your_email@gmail.com
   SMTP_PASSWORD=your_app_password
   ```

## Core Usage

### Starting the System

1. **Launch Web Application**:
   ```bash
   python app.py
   # Access at http://localhost:5000
   ```

2. **Start Crawlers** (via frontend or CLI):
   ```bash
   # Test single crawler
   python runspider-test.py
   
   # Start all crawlers (usually triggered from UI)
   python run_spiders.py
   ```

### Crawler Management

The system includes 15 platform crawlers located in `hotsearchcrawler/hotsearchcrawler/spiders/`:

```python
# Example: Weibo crawler (weibo_spider.py)
from scrapy import Spider
from hotsearchcrawler.items import HotSearchItem

class WeiboSpider(Spider):
    name = 'weibo'
    allowed_domains = ['weibo.com']
    start_urls = ['https://s.weibo.com/top/summary']
    
    def parse(self, response):
        for item in response.css('tr.item'):
            hot_item = HotSearchItem()
            hot_item['title'] = item.css('td.td-02 a::text').get()
            hot_item['url'] = response.urljoin(item.css('td.td-02 a::attr(href)').get())
            hot_item['heat'] = item.css('td.td-02 span::text').get()
            hot_item['platform'] = 'weibo'
            hot_item['rank'] = item.css('td.td-01::text').get()
            yield hot_item
```

### Analysis API Usage

```python
from hotsearch_analysis_agent.analyzer import OpinionAnalyzer
from hotsearch_analysis_agent.database import DatabaseManager

# Initialize analyzer
db = DatabaseManager()
analyzer = OpinionAnalyzer(model_name=os.getenv('MODEL_NAME'))

# Query trending topics
query = "人工智能"
results = db.query_topics(query, days=7)

# Perform sentiment analysis
for result in results:
    sentiment = analyzer.analyze_sentiment(result['content'])
    print(f"Title: {result['title']}")
    print(f"Sentiment: {sentiment['label']} (score: {sentiment['score']})")
    
# Topic clustering
clusters = analyzer.cluster_topics(results, num_clusters=5)
for idx, cluster in enumerate(clusters):
    print(f"\nCluster {idx + 1}:")
    for topic in cluster['topics']:
        print(f"  - {topic['title']}")
    print(f"Keywords: {', '.join(cluster['keywords'])}")
```

### Conversational Query Interface

```python
from hotsearch_analysis_agent.chat_interface import ChatInterface

# Initialize chat interface
chat = ChatInterface()

# Natural language queries
response = chat.query("最近关于人工智能的热点有哪些?")
print(response)

# Specific platform query
response = chat.query("微博今天的热搜榜前10")
print(response)

# Sentiment analysis request
response = chat.query("分析下DeepSeek相关话题的情感倾向")
print(response)
```

### Push Notification Setup

```python
from hotsearch_analysis_agent.push_service import PushService

# Initialize push service
pusher = PushService()

# Create monitoring task
task = {
    'keywords': ['人工智能', 'AI', '大模型'],
    'platforms': ['weibo', 'zhihu', 'bilibili'],
    'threshold': 50000,  # Heat threshold
    'channels': ['wechat', 'telegram', 'email'],
    'frequency': 'hourly'
}

# Register task
pusher.create_task(task)

# Test push
pusher.test_push(
    channel='wechat',
    content='测试推送消息'
)

# Manual report generation
report = analyzer.generate_report(
    query='人工智能',
    start_date='2026-05-01',
    end_date='2026-05-21'
)
pusher.send_report(report, channels=['email', 'telegram'])
```

### Detail Page Content Extraction

```python
from hotsearch_analysis_agent.content_extractor import ContentExtractor

# Initialize extractor (uses browser driver)
extractor = ContentExtractor(driver_type='chrome')

# Extract article content
url = 'https://www.toutiao.com/article/7625514106853818934/'
content = extractor.extract(url)

print(f"Title: {content['title']}")
print(f"Text: {content['text'][:500]}...")
print(f"Images: {len(content['images'])}")

# Extract video content (with transcription support)
video_url = 'https://www.bilibili.com/video/BV13pSoBBEvX/'
video_content = extractor.extract(video_url, extract_video=True)
print(f"Video Title: {video_content['title']}")
print(f"Description: {video_content['description']}")
print(f"Transcript: {video_content['transcript'][:500]}...")
```

## Common Patterns

### Custom Spider Integration

```python
# Add new platform spider
# File: hotsearchcrawler/hotsearchcrawler/spiders/custom_spider.py

from scrapy import Spider
from hotsearchcrawler.items import HotSearchItem

class CustomSpider(Spider):
    name = 'custom_platform'
    
    def start_requests(self):
        urls = ['https://example.com/trending']
        for url in urls:
            yield scrapy.Request(url, callback=self.parse)
    
    def parse(self, response):
        for item in response.css('.trending-item'):
            hot_item = HotSearchItem()
            hot_item['title'] = item.css('.title::text').get()
            hot_item['url'] = item.css('a::attr(href)').get()
            hot_item['heat'] = item.css('.heat::text').get()
            hot_item['platform'] = self.name
            hot_item['timestamp'] = datetime.now()
            yield hot_item
```

### Custom Analysis Pipeline

```python
from hotsearch_analysis_agent.analyzer import BaseAnalyzer

class CustomAnalyzer(BaseAnalyzer):
    def analyze(self, data):
        """Custom analysis logic"""
        results = []
        
        # Preprocess
        processed = self.preprocess(data)
        
        # LLM analysis
        for item in processed:
            prompt = f"""
            分析以下舆情数据:
            标题: {item['title']}
            内容: {item['content']}
            
            请提供:
            1. 情感倾向 (正面/负面/中性)
            2. 关键主题
            3. 潜在影响
            """
            
            analysis = self.llm_call(prompt)
            results.append({
                'item': item,
                'analysis': analysis
            })
        
        return self.generate_summary(results)
```

### Automated Reporting

```python
from apscheduler.schedulers.background import BackgroundScheduler
from hotsearch_analysis_agent.reporter import ReportGenerator

def scheduled_report():
    reporter = ReportGenerator()
    report = reporter.generate_daily_report(
        topics=['科技', '经济', '社会'],
        platforms='all'
    )
    
    pusher = PushService()
    pusher.send_report(
        report,
        channels=['email'],
        recipients=['team@company.com']
    )

# Schedule daily at 9:00 AM
scheduler = BackgroundScheduler()
scheduler.add_job(scheduled_report, 'cron', hour=9, minute=0)
scheduler.start()
```

## Troubleshooting

### Browser Driver Issues

```bash
# Error: WebDriver not found
# Solution: Verify driver in PATH
echo $PATH  # Check if driver directory is included

# Error: Version mismatch
# Solution: Download exact version matching browser
google-chrome --version
chromedriver --version  # Should match major version
```

### Database Connection Errors

```python
# Test database connection
from hotsearch_analysis_agent.database import DatabaseManager

try:
    db = DatabaseManager()
    db.test_connection()
    print("✓ Database connected")
except Exception as e:
    print(f"✗ Database error: {e}")
    # Check .env settings and MySQL service status
```

### Crawler Timeout/Blocking

```python
# Adjust crawler settings for rate limiting
# File: hotsearchcrawler/settings.py

DOWNLOAD_DELAY = 3  # Increase delay between requests
CONCURRENT_REQUESTS_PER_DOMAIN = 1  # Reduce concurrency
RETRY_TIMES = 5
RETRY_HTTP_CODES = [500, 502, 503, 504, 408, 429]

# Add user agent rotation
USER_AGENTS = [
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
    'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36',
    # Add more...
]
```

### LLM API Errors

```python
# Handle API failures gracefully
from hotsearch_analysis_agent.analyzer import OpinionAnalyzer

analyzer = OpinionAnalyzer(
    model_name=os.getenv('MODEL_NAME'),
    max_retries=3,
    timeout=30
)

try:
    result = analyzer.analyze_sentiment(text)
except Exception as e:
    # Fallback to rule-based analysis
    result = analyzer.fallback_sentiment(text)
```

### Memory Issues with Large Datasets

```python
# Process data in batches
def analyze_in_batches(data, batch_size=100):
    results = []
    for i in range(0, len(data), batch_size):
        batch = data[i:i+batch_size]
        batch_results = analyzer.analyze_batch(batch)
        results.extend(batch_results)
        
        # Clear memory
        import gc
        gc.collect()
    
    return results
```

## Project Structure Reference

```
├── hotsearch_analysis_agent/      # Analysis system
│   ├── analyzer.py               # LLM analysis engine
│   ├── chat_interface.py         # Conversational interface
│   ├── content_extractor.py      # Detail page extraction
│   ├── database.py               # Database manager
│   ├── push_service.py           # Push notifications
│   └── reporter.py               # Report generation
├── hotsearchcrawler/             # Crawler cluster
│   ├── hotsearchcrawler/
│   │   ├── spiders/              # Platform spiders
│   │   ├── items.py              # Data items
│   │   ├── pipelines.py          # Processing pipelines
│   │   └── settings.py           # Crawler settings
│   └── scrapy.cfg
├── app.py                        # Main application entry
├── run_spiders.py                # Crawler launcher
├── test_push_task.py             # Push testing
├── init.py                       # Database initialization
└── requirements.txt              # Dependencies
```

## Advanced Configuration

### PanGu Model Integration (Recommended for Chinese)

```bash
# Download PanGu model
# https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

# Configure in .env
USE_PANGU=true
PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model
PANGU_DEVICE=cuda  # or cpu
```

```python
# Use in code
from hotsearch_analysis_agent.models import PanGuModel

model = PanGuModel(
    model_path=os.getenv('PANGU_MODEL_PATH'),
    device=os.getenv('PANGU_DEVICE', 'cpu')
)

analysis = model.analyze(
    text="华为盘古大模型在舆情分析中表现出色",
    task="sentiment"
)
```

This skill provides comprehensive coverage of the LLM-Based Intelligent Public Opinion Analytics Assistant for AI coding agents to effectively assist developers in deploying and using this multi-platform monitoring and analysis system.
