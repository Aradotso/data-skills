---
name: llm-public-opinion-analytics-assistant
description: Use this Chinese public opinion analytics assistant with 15 platforms, 26 rankings, LLM analysis, web crawler control, and multi-channel alerting
triggers:
  - analyze public opinion from Chinese social media platforms
  - set up hot topic monitoring and push notifications
  - crawl trending topics from Weibo Bilibili and other platforms
  - perform sentiment analysis on Chinese news and social content
  - cluster related topics across multiple platforms
  - configure Telegram WeChat or email alerts for trending topics
  - query real-time rankings from Chinese platforms
  - extract content from video-based news articles
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is a Chinese public opinion (舆情) analytics assistant that integrates real-time data from **15 mainstream platforms** across **26 ranking lists**, combined with large language model (LLM) analysis capabilities. It enables conversational hot search queries, topic clustering, sentiment analysis, and multi-channel alerting (WeChat, Telegram, email).

## What This Project Does

- **Multi-Platform Data Collection**: Crawls trending topics from 15 Chinese platforms (Weibo, Bilibili, Douyin, Zhihu, etc.)
- **LLM-Powered Analysis**: Uses LLMs (including Huawei Pangu model) for topic clustering, sentiment analysis, and report generation
- **Conversational Interface**: Natural language queries for hot searches and specific topics
- **Crawler Control**: Keyboard shortcuts to start/stop crawlers via the frontend
- **Content Extraction**: Extracts text content even from video-based news pages
- **Multi-Channel Alerts**: Push notifications via Enterprise WeChat, Telegram, or email

## Project Structure

```
├── hotsearchcrawler/          # Crawler cluster (decoupled from analysis)
│   ├── spiders/               # Platform-specific spiders
│   └── settings.py            # Crawler configuration
├── hotsearch_analysis_agent/  # LLM analysis system
│   ├── models/                # Database models
│   ├── agents/                # LLM agent logic
│   └── push/                  # Push notification modules
├── app.py                     # Main application entry
├── run_spiders.py             # Crawler launcher
├── init.py                    # Database initialization reference
└── requirements.txt           # Python dependencies
```

## Installation

### 1. Prerequisites

**Browser Driver Setup** (required for news detail extraction):

```bash
# For Chrome
# 1. Check Chrome version: chrome://version/
# 2. Download matching ChromeDriver from https://chromedriver.chromium.org/
# 3. Place in PATH or project directory

# For Edge
# Download from https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Verify installation
chromedriver --version
```

**Database Setup**:

```bash
# Install MySQL and create database
mysql -u root -p

# In MySQL shell:
CREATE DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### 2. Install Python Dependencies

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### 3. Configuration

**Crawler Configuration** (`hotsearchcrawler/settings.py`):

```python
# MySQL settings
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_user'
MYSQL_PASSWORD = 'your_password'
MYSQL_DB = 'hotsearch'

# Optional: Platform cookies for authenticated access
COOKIES = {
    'weibo': 'your_weibo_cookie',
    'bilibili': 'your_bilibili_cookie'
}
```

**Analysis System Configuration** (`.env` file):

```env
# MySQL
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DB=hotsearch

# LLM API (OpenAI-compatible format)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1

# Or use Huawei Pangu model (local deployment)
PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model

# Push notification settings
# Enterprise WeChat
WECHAT_WEBHOOK_URL=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY
WECHAT_CORP_ID=your_corp_id
WECHAT_AGENT_ID=your_agent_id
WECHAT_SECRET=your_secret

# Telegram
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
```

### 4. Initialize Database

```python
# Reference init.py for table schemas
# Run database initialization
python init.py
```

## Running the Application

### Start the Analysis System

```python
# app.py - Main application
from flask import Flask, render_template, request, jsonify
from hotsearch_analysis_agent.agents import AnalysisAgent
from hotsearch_analysis_agent.push import PushManager

app = Flask(__name__)
agent = AnalysisAgent()
push_manager = PushManager()

# Start the web interface
if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
```

```bash
# Launch the application
python app.py
```

### Control Crawlers

```python
# run_spiders.py - Programmatic crawler control
from scrapy.crawler import CrawlerProcess
from scrapy.utils.project import get_project_settings
from hotsearchcrawler.spiders import *

def start_crawlers(spider_names=None):
    """Start specified crawlers or all if none specified"""
    process = CrawlerProcess(get_project_settings())
    
    if spider_names is None:
        # Start all spiders
        spider_names = ['weibo', 'bilibili', 'zhihu', 'douyin', 'toutiao']
    
    for spider_name in spider_names:
        process.crawl(spider_name)
    
    process.start()

# Start specific crawlers
start_crawlers(['weibo', 'bilibili'])
```

Or use the web UI keyboard shortcuts:
- Press `Ctrl+S` to start crawlers
- Press `Ctrl+E` to stop crawlers

## Key API Usage

### Query Hot Topics

```python
from hotsearch_analysis_agent.query import QueryEngine

query_engine = QueryEngine()

# Natural language query
results = query_engine.search("最近关于人工智能的热搜")
# Returns: List of relevant hot topics with metadata

# Platform-specific query
results = query_engine.search("微博热搜", platform="weibo")

# Time range query
results = query_engine.search("昨天的热点新闻", time_range="1d")
```

### Topic Clustering Analysis

```python
from hotsearch_analysis_agent.agents import ClusteringAgent

clustering = ClusteringAgent()

# Cluster related topics
clusters = clustering.analyze_topics(
    query="人工智能 芯片 大模型",
    platforms=["weibo", "bilibili", "toutiao"],
    time_range="7d"
)

# Result structure:
# {
#     "clusters": [
#         {
#             "theme": "国产芯片与大模型协同",
#             "topics": [...],
#             "sentiment": "positive",
#             "trend": "rising"
#         }
#     ],
#     "summary": "近期AI领域主要关注..."
# }
```

### Sentiment Analysis

```python
from hotsearch_analysis_agent.agents import SentimentAgent

sentiment = SentimentAgent()

# Analyze sentiment for a topic
result = sentiment.analyze("华为发布新芯片")

# Result:
# {
#     "sentiment": "positive",
#     "confidence": 0.87,
#     "details": {
#         "positive_ratio": 0.75,
#         "negative_ratio": 0.15,
#         "neutral_ratio": 0.10
#     },
#     "key_opinions": [...]
# }
```

### Generate Analysis Report

```python
from hotsearch_analysis_agent.agents import ReportAgent

report_agent = ReportAgent()

# Generate comprehensive report
report = report_agent.generate_report(
    query="人工智能与前沿科技",
    time_range="7d",
    platforms=["weibo", "bilibili", "toutiao", "zhihu"]
)

# Report includes:
# - Core findings and data highlights
# - Detailed news content
# - Analysis and summary
# - Information dissemination characteristics
```

### Configure Push Tasks

```python
from hotsearch_analysis_agent.push import PushTask

# Create a push task
task = PushTask.create(
    name="AI热点监控",
    query="人工智能 大模型 芯片",
    schedule="0 8,20 * * *",  # Cron format: 8AM and 8PM daily
    channels=["wechat", "telegram", "email"],
    platforms=["weibo", "bilibili", "toutiao"],
    alert_threshold=100,  # Alert if topic heat exceeds 100
    sentiment_filter="all"  # or "positive", "negative"
)

# Test push task
from test_push_task import test_push

test_push(
    query="人工智能",
    channels=["telegram"],
    recipients=["@your_telegram_id"]
)
```

## Common Patterns

### Building a Custom Spider

```python
# hotsearchcrawler/spiders/custom_spider.py
import scrapy
from hotsearchcrawler.items import HotSearchItem

class CustomSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://example.com/trending']
    
    def parse(self, response):
        for topic in response.css('.trending-topic'):
            item = HotSearchItem()
            item['title'] = topic.css('.title::text').get()
            item['url'] = topic.css('a::attr(href)').get()
            item['heat'] = topic.css('.heat::text').get()
            item['rank'] = topic.css('.rank::text').get()
            item['platform'] = self.name
            item['category'] = topic.css('.category::text').get()
            
            # Follow detail page for content extraction
            yield scrapy.Request(
                item['url'],
                callback=self.parse_detail,
                meta={'item': item}
            )
    
    def parse_detail(self, response):
        item = response.meta['item']
        item['content'] = response.css('.article-content::text').getall()
        item['publish_time'] = response.css('.publish-time::text').get()
        yield item
```

### Custom LLM Integration

```python
# Using Huawei Pangu model locally
from hotsearch_analysis_agent.llm import PanguModel

model = PanguModel(model_path="/path/to/openpangu-embedded-7b-model")

# Perform analysis
result = model.analyze(
    prompt="""请分析以下热搜话题的情感倾向和主要观点:
    标题: 华为发布昇腾芯片
    内容: ...""",
    temperature=0.7,
    max_tokens=2000
)

# Or use OpenAI-compatible API
from hotsearch_analysis_agent.llm import OpenAIModel

model = OpenAIModel(
    api_key=os.getenv("OPENAI_API_KEY"),
    base_url=os.getenv("OPENAI_API_BASE")
)

result = model.analyze(prompt="...")
```

### Multi-Channel Push Implementation

```python
from hotsearch_analysis_agent.push import (
    WeChatPusher, TelegramPusher, EmailPusher
)

# WeChat Enterprise push
wechat = WeChatPusher(webhook_url=os.getenv("WECHAT_WEBHOOK_URL"))
wechat.push(
    title="AI热点日报",
    content=report_content,
    mentioned_list=["@all"]
)

# Telegram push
telegram = TelegramPusher(
    bot_token=os.getenv("TELEGRAM_BOT_TOKEN"),
    chat_id=os.getenv("TELEGRAM_CHAT_ID")
)
telegram.push(
    text=report_content,
    parse_mode="Markdown"
)

# Email push
email = EmailPusher(
    smtp_host=os.getenv("SMTP_HOST"),
    smtp_port=int(os.getenv("SMTP_PORT")),
    username=os.getenv("SMTP_USER"),
    password=os.getenv("SMTP_PASSWORD")
)
email.push(
    subject="舆情分析报告",
    body=report_content,
    recipients=["user@example.com"]
)
```

## Troubleshooting

**Crawler not collecting data**:
- Verify browser driver is in PATH: `chromedriver --version`
- Check MySQL connection in `hotsearchcrawler/settings.py`
- Some platforms may require valid cookies in settings

**LLM analysis errors**:
- Ensure `OPENAI_API_KEY` and `OPENAI_API_BASE` are set in `.env`
- For Pangu model, verify model path exists and has correct permissions
- Check model context length limits for long documents

**Push notifications failing**:
- WeChat: Verify webhook URL and corporate ID in `.env`
- Telegram: Check bot token with `curl https://api.telegram.org/bot<TOKEN>/getMe`
- Email: Enable "less secure app access" or use app-specific password

**Database connection issues**:
- Verify MySQL is running: `systemctl status mysql`
- Test connection: `mysql -u <user> -p -h <host> <database>`
- Check character set: `SHOW VARIABLES LIKE 'character_set%';`

**Video content extraction failing**:
- Ensure Selenium WebDriver is properly configured
- Check browser version matches driver version
- Increase wait times for slow-loading video pages

**Memory issues with large crawls**:
- Limit concurrent requests in `settings.py`: `CONCURRENT_REQUESTS = 16`
- Enable autothrottle: `AUTOTHROTTLE_ENABLED = True`
- Reduce batch size in database pipeline
