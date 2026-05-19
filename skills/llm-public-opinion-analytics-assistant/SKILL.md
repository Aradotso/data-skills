---
name: llm-public-opinion-analytics-assistant
description: Master the LLM-based intelligent public opinion analytics assistant for real-time hotlist crawling, sentiment analysis, and multi-channel alert pushing
triggers:
  - how do I set up the public opinion analytics assistant
  - crawl hot search data from multiple platforms
  - analyze sentiment and cluster topics with LLM
  - configure push notifications for trending topics
  - set up WeChat or Telegram alerts for hot trends
  - troubleshoot the hotsearch crawler system
  - integrate Pangu model for Chinese sentiment analysis
  - query real-time trending data from social platforms
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is an intelligent public opinion analytics assistant that combines real-time data from **15 mainstream platforms** (26 trending lists) with large language model analysis capabilities. It provides conversational hotlist queries, topic-specific searches, cluster analysis, and sentiment analysis through a web interface.

**Core capabilities:**
- Real-time crawling from 15 platforms (Weibo, Bilibili, Baidu, Douyin, etc.)
- LLM-powered sentiment analysis and topic clustering
- Multi-channel push notifications (Email, WeChat, Enterprise WeChat, Telegram)
- Video content analysis extraction
- Keyboard shortcut-controlled crawler management
- Cross-platform data aggregation and jump-to-source

## Installation

### Prerequisites

**1. Browser Driver Setup (Required for Content Extraction)**

Download the appropriate WebDriver for your browser:

```bash
# For Chrome
# Visit https://chromedriver.chromium.org/
# Download version matching your Chrome browser

# For Edge
# Visit https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/
# Download version matching your Edge browser

# Place driver in system PATH or project directory
# Verify installation:
chromedriver --version
```

**2. Database Setup**

```bash
# Install MySQL 8.0+
# Create database and tables using init.py reference

mysql -u root -p
CREATE DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

**3. Python Environment**

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Configuration

**1. Crawler Configuration (`hotsearchcrawler/settings.py`)**

```python
# MySQL settings
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_user'
MYSQL_PASSWORD = 'your_password'
MYSQL_DB = 'hotsearch'

# Optional: Platform-specific cookies
COOKIES = {
    'weibo': 'your_weibo_cookie',
    'bilibili': 'your_bilibili_cookie'
}
```

**2. Analysis System Configuration (`.env`)**

```bash
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DB=hotsearch

# OpenAI-compatible API (for LLM analysis)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Or use Pangu model (recommended for Chinese)
# PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model

# Push Notifications
# Email
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password

# Enterprise WeChat
WECHAT_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxx

# Telegram
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

## Project Structure

```
├── app.py                          # Main application entry
├── hotsearch_analysis_agent/       # Analysis system
│   ├── agent.py                    # LLM agent logic
│   ├── sentiment_analyzer.py      # Sentiment analysis
│   └── topic_cluster.py            # Topic clustering
├── hotsearchcrawler/               # Crawler cluster
│   ├── spiders/                    # Platform spiders
│   │   ├── weibo_spider.py
│   │   ├── bilibili_spider.py
│   │   └── ...
│   └── settings.py                 # Crawler settings
├── run_spiders.py                  # Crawler launcher
└── test_push_task.py               # Push notification test
```

## Usage

### Starting the System

```bash
# Start the main application
python app.py

# The web interface will be available at http://localhost:5000
```

### Running Crawlers

**Option 1: Via Web Interface**
- Access the web UI and use keyboard shortcuts to start/stop crawlers
- Default shortcut: `Ctrl+Shift+C` to toggle crawler

**Option 2: Command Line**

```bash
# Run all spiders
python run_spiders.py

# Test single spider
python runspider-test.py --spider weibo

# Supported platforms:
# weibo, bilibili, douyin, baidu, zhihu, toutiao, 
# kuaishou, xiaohongshu, 36kr, ithome, etc.
```

### Querying Data

**Natural Language Query (via Web UI)**

```python
# Example queries users can make:
"今天微博热搜前10是什么？"
"最近关于人工智能的热点新闻"
"分析一下科技类话题的情感倾向"
"聚类最近三天的热门话题"
```

**Programmatic Query**

```python
from hotsearch_analysis_agent.query import HotSearchQuery

query = HotSearchQuery()

# Get latest hotlist from specific platform
weibo_hot = query.get_platform_hotlist('weibo', limit=10)

# Search by keyword
ai_news = query.search_by_keyword('人工智能', platforms=['weibo', 'bilibili'])

# Get trending topics in time range
trending = query.get_trending(
    start_date='2026-05-01',
    end_date='2026-05-19',
    platforms=['all']
)
```

### Sentiment Analysis

```python
from hotsearch_analysis_agent.sentiment_analyzer import SentimentAnalyzer

analyzer = SentimentAnalyzer()

# Analyze single text
result = analyzer.analyze_sentiment("这个产品真的很不错,值得推荐!")
# Returns: {'sentiment': 'positive', 'score': 0.87, 'keywords': ['不错', '推荐']}

# Batch analysis
texts = ["新闻标题1", "新闻标题2", ...]
results = analyzer.batch_analyze(texts)

# Aggregate sentiment for a topic
topic_sentiment = analyzer.analyze_topic_sentiment(
    topic="人工智能",
    time_range="7d"
)
```

### Topic Clustering

```python
from hotsearch_analysis_agent.topic_cluster import TopicCluster

cluster = TopicCluster()

# Cluster recent hot topics
clusters = cluster.cluster_topics(
    days=7,
    min_cluster_size=5,
    platforms=['weibo', 'zhihu', 'bilibili']
)

# Output format:
# [
#   {
#     'cluster_id': 1,
#     'main_topic': 'AI大模型',
#     'related_topics': ['GPT-6', 'DeepSeek', '华为昇腾'],
#     'sentiment': 'positive',
#     'trend': 'rising'
#   },
#   ...
# ]
```

### Push Notifications

**Configure Push Tasks**

```python
from hotsearch_analysis_agent.push_task import PushTask

task = PushTask()

# Create a scheduled push
task.create_task(
    name="AI科技日报",
    keywords=["人工智能", "大模型", "芯片"],
    platforms=["weibo", "36kr", "ithome"],
    channels=["email", "wechat"],  # email, wechat, telegram
    schedule="0 9 * * *",  # Daily at 9 AM (cron format)
    sentiment_filter="all",  # positive, negative, neutral, all
    min_heat=1000  # Minimum heat score
)

# List active tasks
tasks = task.list_tasks()

# Delete task
task.delete_task(task_id=1)
```

**Test Push Manually**

```bash
# Test all configured channels
python test_push_task.py

# Test specific channel
python test_push_task.py --channel email
python test_push_task.py --channel wechat
python test_push_task.py --channel telegram
```

**Push Channel Examples**

```python
# Email push
from hotsearch_analysis_agent.push.email_push import EmailPush

email = EmailPush()
email.send(
    subject="舆情分析报告 - 2026-05-19",
    content=report_html,
    recipients=["user@example.com"]
)

# Enterprise WeChat push
from hotsearch_analysis_agent.push.wechat_push import WeChatPush

wechat = WeChatPush()
wechat.send_markdown(
    content="## AI热点分析\n- GPT-6曝光\n- DeepSeek V4采用华为算力"
)

# Telegram push
from hotsearch_analysis_agent.push.telegram_push import TelegramPush

telegram = TelegramPush()
telegram.send_message(
    text="📊 今日热点分析已生成",
    parse_mode="Markdown"
)
```

## Advanced Features

### Using Pangu Model for Chinese Analysis

```python
from hotsearch_analysis_agent.models.pangu_model import PanguAnalyzer

# Initialize Pangu model (better for Chinese sentiment)
pangu = PanguAnalyzer(
    model_path="/path/to/openpangu-embedded-7b-model"
)

# Analyze long Chinese text
analysis = pangu.analyze(
    text=long_news_article,
    tasks=["sentiment", "summary", "keywords"]
)

# Returns:
# {
#   'sentiment': {'label': 'positive', 'confidence': 0.92},
#   'summary': '华为盘古大模型在舆情分析中表现出色...',
#   'keywords': ['盘古模型', '舆情分析', '长文本']
# }
```

### Video Content Extraction

```python
from hotsearch_analysis_agent.content_extractor import VideoExtractor

extractor = VideoExtractor()

# Extract content from video news (Bilibili, Douyin, etc.)
content = extractor.extract_video_content(
    url="https://www.bilibili.com/video/BV13pSoBBEvX"
)

# Returns:
# {
#   'title': 'GPT-6遭提前曝光',
#   'description': '...',
#   'comments_sentiment': 'positive',
#   'keywords': ['GPT-6', '2M上下文']
# }
```

### Custom Spider Development

```python
# hotsearchcrawler/spiders/custom_platform_spider.py
import scrapy
from hotsearchcrawler.items import HotSearchItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://example.com/trending']
    
    def parse(self, response):
        for item in response.css('.trending-item'):
            yield HotSearchItem(
                platform='custom_platform',
                title=item.css('.title::text').get(),
                url=item.css('a::attr(href)').get(),
                heat=item.css('.heat::text').get(),
                rank=item.css('.rank::text').get(),
                timestamp=datetime.now()
            )
```

## Common Patterns

### Daily Report Generation

```python
from hotsearch_analysis_agent.report_generator import ReportGenerator

generator = ReportGenerator()

# Generate daily report
report = generator.generate_daily_report(
    date="2026-05-19",
    topics=["人工智能", "科技"],
    platforms=["weibo", "zhihu", "36kr"],
    include_sentiment=True,
    include_clusters=True
)

# Save as HTML
report.save_html("report_2026-05-19.html")

# Auto-push via configured channels
report.push(channels=["email", "wechat"])
```

### Real-time Monitoring

```python
from hotsearch_analysis_agent.monitor import RealTimeMonitor

monitor = RealTimeMonitor()

# Set up keyword alert
monitor.add_alert(
    keywords=["突发", "重大"],
    platforms=["weibo", "toutiao"],
    callback=lambda item: send_urgent_notification(item),
    sentiment_filter="negative"
)

# Start monitoring
monitor.start(interval=300)  # Check every 5 minutes
```

## Troubleshooting

### Crawler Issues

**Problem: Spider returns empty results**

```bash
# Check browser driver
chromedriver --version

# Verify cookies (if required)
# Update cookies in hotsearchcrawler/settings.py

# Test individual spider with verbose output
scrapy crawl weibo -L DEBUG
```

**Problem: MySQL connection error**

```python
# Verify MySQL settings in .env
# Test connection:
import pymysql
conn = pymysql.connect(
    host='localhost',
    user='your_user',
    password='your_password',
    database='hotsearch'
)
print("Connection successful!")
conn.close()
```

### LLM Analysis Issues

**Problem: Slow analysis performance**

```python
# Switch to local Pangu model for better Chinese support
# Download from: https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

# Update .env:
# USE_LOCAL_MODEL=true
# PANGU_MODEL_PATH=/path/to/model

# Or use batch processing:
analyzer.set_batch_size(10)  # Process 10 items at once
```

**Problem: Poor sentiment accuracy**

```python
# Fine-tune sentiment threshold
analyzer = SentimentAnalyzer(
    positive_threshold=0.7,  # Increase for stricter positive
    negative_threshold=0.3   # Decrease for stricter negative
)

# Use Pangu for better Chinese understanding
from hotsearch_analysis_agent.models.pangu_model import PanguAnalyzer
analyzer = PanguAnalyzer()
```

### Push Notification Issues

**Problem: Email not sending**

```bash
# Test SMTP connection
python test_push_task.py --channel email --debug

# Common fixes:
# 1. Enable "Less secure app access" (Gmail)
# 2. Use app-specific password (Gmail, Outlook)
# 3. Check firewall/port 587 access
```

**Problem: WeChat webhook not working**

```python
# Verify webhook URL format
# Correct: https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Test webhook:
import requests
requests.post(WECHAT_WEBHOOK, json={
    "msgtype": "text",
    "text": {"content": "Test message"}
})
```

## Performance Optimization

```python
# Enable async crawling for better performance
# In hotsearchcrawler/settings.py:
CONCURRENT_REQUESTS = 16
CONCURRENT_REQUESTS_PER_DOMAIN = 8
DOWNLOAD_DELAY = 0.5

# Use Redis for caching
REDIS_HOST = 'localhost'
REDIS_PORT = 6379
CACHE_ENABLED = True
CACHE_EXPIRE_TIME = 3600

# Database connection pooling
MYSQL_POOL_SIZE = 10
MYSQL_MAX_OVERFLOW = 20
```

## API Reference

Full API documentation available in the codebase. Key modules:

- `hotsearch_analysis_agent.agent` - Main LLM agent
- `hotsearch_analysis_agent.sentiment_analyzer` - Sentiment analysis
- `hotsearch_analysis_agent.topic_cluster` - Topic clustering
- `hotsearch_analysis_agent.push_task` - Push notifications
- `hotsearchcrawler.spiders` - Platform crawlers
