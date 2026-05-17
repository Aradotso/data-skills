---
name: llm-public-opinion-analytics-assistant
description: Build real-time public opinion analytics systems with multi-platform web scraping, LLM-powered sentiment analysis, topic clustering, and multi-channel alert pushing
triggers:
  - scrape hot search rankings from weibo douyin bilibili
  - analyze public opinion sentiment with llm
  - set up real-time topic monitoring and alerts
  - cluster trending topics across social platforms
  - push hot topic reports to wechat telegram email
  - build a public opinion analytics dashboard
  - crawl news detail pages including video content
  - configure pangu llm for chinese sentiment analysis
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build and operate intelligent public opinion analytics systems that aggregate data from 15 mainstream Chinese platforms (26 ranking lists total), analyze sentiment and topics using LLMs, and push alerts via email, WeChat, Enterprise WeChat, and Telegram.

## What This Project Does

The LLM-Based Intelligent Public Opinion Analytics Assistant is a Python-based system that:

- **Scrapes real-time trending data** from 15 platforms including Weibo, Douyin, Bilibili, Baidu, Zhihu, etc.
- **Analyzes content** using large language models (Huawei Pangu or OpenAI-compatible APIs)
- **Performs sentiment analysis** and topic clustering on Chinese text
- **Extracts video content** from news detail pages for deeper analysis
- **Pushes reports** to multiple channels (Enterprise WeChat, Telegram, email)
- **Provides a conversational UI** for natural language queries about trending topics

## Installation

### Prerequisites

1. **Browser Driver Setup** (required for scraping):

```bash
# Download ChromeDriver or EdgeDriver matching your browser version
# ChromeDriver: https://chromedriver.chromium.org/
# EdgeDriver: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Place driver in system PATH or project directory
# Verify installation:
chromedriver --version
```

2. **MySQL Database**:

```bash
# Install MySQL 5.7+ and create database
mysql -u root -p
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

3. **Python Environment**:

```bash
git clone https://github.com/hmmnxkl/LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant.git
cd LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### Initialize Database

```python
# Reference init.py for schema creation
# Key tables: hot_searches, news_details, analysis_results, push_tasks

from hotsearch_analysis_agent.database import init_database
init_database()
```

## Configuration

### Environment Variables (`.env`)

```bash
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key_here
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Or use Huawei Pangu (recommended for Chinese text)
PANGU_API_KEY=your_pangu_key
PANGU_API_BASE=https://pangu-api.huaweicloud.com

# Push Notification Channels
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password

WECHAT_WEBHOOK_URL=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxx
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

### Crawler Settings (`hotsearchcrawler/settings.py`)

```python
# MySQL connection for crawler
MYSQL_SETTINGS = {
    'host': 'localhost',
    'port': 3306,
    'user': 'root',
    'password': 'your_password',
    'database': 'hotsearch_db',
}

# Optional: Platform-specific cookies
COOKIES = {
    'weibo': 'your_weibo_cookie',
    'douyin': 'your_douyin_cookie',
}

# Crawler concurrency
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
```

## Running the System

### Start the Web Interface

```bash
python app.py
# Access at http://localhost:5000
```

### Run Crawlers Manually

```bash
# Test single spider
python runspider-test.py weibo

# Run all spiders
python run_spiders.py
```

### Test Push Notifications

```bash
python test_push_task.py
```

## Key Components & API

### 1. Scraping Hot Search Data

```python
from hotsearchcrawler.spiders.weibo_spider import WeiboSpider
from scrapy.crawler import CrawlerProcess

# Run Weibo spider
process = CrawlerProcess()
process.crawl(WeiboSpider)
process.start()
```

Supported platforms: Weibo, Douyin, Bilibili, Baidu, Zhihu, Toutiao, 36Kr, Tencent News, NetEase News, Sina News, iHeima, Hupu, Tieba, Kuaishou, Xiaohongshu

### 2. LLM-Based Analysis

```python
from hotsearch_analysis_agent.llm_analyzer import TopicAnalyzer

analyzer = TopicAnalyzer(
    api_key=os.getenv('OPENAI_API_KEY'),
    model='gpt-4'
)

# Sentiment analysis
sentiment = analyzer.analyze_sentiment(
    "华为发布新款AI芯片,性能提升300%"
)
# Returns: {'sentiment': 'positive', 'score': 0.85, 'keywords': ['华为', 'AI芯片', '性能提升']}

# Topic clustering
topics = analyzer.cluster_topics(news_list)
# Returns: [{'topic': 'AI技术突破', 'articles': [...], 'sentiment': 'positive'}]
```

### 3. Query Trending Topics

```python
from hotsearch_analysis_agent.query_engine import QueryEngine

engine = QueryEngine()

# Natural language query
results = engine.query("最近关于人工智能的热点新闻")
# Returns structured data with titles, URLs, sentiment, platform

# Filter by platform
weibo_trends = engine.query("微博热搜前10", platform='weibo', limit=10)

# Filter by time
recent = engine.query("今天的科技新闻", hours=24, category='tech')
```

### 4. Push Notifications

```python
from hotsearch_analysis_agent.push_service import PushService

pusher = PushService()

# Enterprise WeChat
pusher.push_to_wechat(
    webhook_url=os.getenv('WECHAT_WEBHOOK_URL'),
    title="AI热点分析",
    content=report_content,
    mentioned_list=["@all"]
)

# Telegram
pusher.push_to_telegram(
    bot_token=os.getenv('TELEGRAM_BOT_TOKEN'),
    chat_id=os.getenv('TELEGRAM_CHAT_ID'),
    message=report_content,
    parse_mode='Markdown'
)

# Email
pusher.push_to_email(
    smtp_host=os.getenv('SMTP_HOST'),
    smtp_port=int(os.getenv('SMTP_PORT')),
    sender=os.getenv('SMTP_USER'),
    password=os.getenv('SMTP_PASSWORD'),
    recipients=['user@example.com'],
    subject="舆情分析报告",
    html_content=report_html
)
```

### 5. Create Automated Monitoring Tasks

```python
from hotsearch_analysis_agent.task_scheduler import TaskScheduler

scheduler = TaskScheduler()

# Monitor specific keywords
task = scheduler.create_task(
    name="AI技术监控",
    keywords=["人工智能", "大模型", "芯片"],
    platforms=["weibo", "toutiao", "36kr"],
    schedule="*/30 * * * *",  # Every 30 minutes
    sentiment_threshold=-0.5,  # Alert on negative sentiment
    push_channels=["wechat", "email"]
)

scheduler.start()
```

## Common Patterns

### Full Analysis Pipeline

```python
import os
from hotsearch_analysis_agent.pipeline import AnalysisPipeline

pipeline = AnalysisPipeline(
    db_config={
        'host': os.getenv('MYSQL_HOST'),
        'user': os.getenv('MYSQL_USER'),
        'password': os.getenv('MYSQL_PASSWORD'),
        'database': os.getenv('MYSQL_DATABASE')
    },
    llm_config={
        'api_key': os.getenv('OPENAI_API_KEY'),
        'model': 'gpt-4'
    }
)

# Full workflow
report = pipeline.run(
    query="近期人工智能领域的热点",
    platforms=["weibo", "toutiao", "bilibili"],
    hours=24,
    cluster=True,
    sentiment_analysis=True,
    generate_report=True
)

# Push report
pipeline.push(
    report=report,
    channels=["wechat", "telegram"],
    wechat_webhook=os.getenv('WECHAT_WEBHOOK_URL'),
    telegram_bot_token=os.getenv('TELEGRAM_BOT_TOKEN'),
    telegram_chat_id=os.getenv('TELEGRAM_CHAT_ID')
)
```

### Extract Video Content Details

```python
from hotsearch_analysis_agent.content_extractor import VideoExtractor

extractor = VideoExtractor()

# Extract from Bilibili video page
video_data = extractor.extract_bilibili(
    url="https://www.bilibili.com/video/BV13pSoBBEvX"
)
# Returns: {'title': '...', 'description': '...', 'comments': [...], 'transcription': '...'}

# Extract from Douyin
douyin_data = extractor.extract_douyin(
    url="https://www.douyin.com/video/..."
)
```

### Custom LLM Prompts

```python
from hotsearch_analysis_agent.llm_analyzer import TopicAnalyzer

analyzer = TopicAnalyzer(
    api_key=os.getenv('OPENAI_API_KEY'),
    model='gpt-4'
)

# Custom analysis prompt
custom_prompt = """
分析以下新闻的关键信息:
1. 核心事件
2. 涉及主体
3. 潜在影响
4. 舆论倾向

新闻内容: {content}
"""

result = analyzer.custom_analysis(
    content=news_content,
    prompt=custom_prompt
)
```

## Troubleshooting

### Browser Driver Issues

```bash
# Error: 'chromedriver' executable needs to be in PATH
# Solution: Add driver to PATH or specify location

from selenium import webdriver
from selenium.webdriver.chrome.service import Service

service = Service(executable_path='/path/to/chromedriver')
driver = webdriver.Chrome(service=service)
```

### Database Connection Errors

```python
# Error: (2003, "Can't connect to MySQL server")
# Check MySQL is running and credentials are correct

import pymysql

try:
    conn = pymysql.connect(
        host=os.getenv('MYSQL_HOST'),
        user=os.getenv('MYSQL_USER'),
        password=os.getenv('MYSQL_PASSWORD'),
        database=os.getenv('MYSQL_DATABASE'),
        charset='utf8mb4'
    )
    print("Database connection successful")
except Exception as e:
    print(f"Connection failed: {e}")
```

### LLM API Timeout

```python
# For slow responses, increase timeout
from hotsearch_analysis_agent.llm_analyzer import TopicAnalyzer

analyzer = TopicAnalyzer(
    api_key=os.getenv('OPENAI_API_KEY'),
    model='gpt-4',
    timeout=120  # Increase timeout to 120 seconds
)
```

### Scraper Blocked by Anti-Bot

```python
# Add random delays and user agents
import time
import random

from hotsearchcrawler.settings import USER_AGENTS

headers = {
    'User-Agent': random.choice(USER_AGENTS),
    'Referer': 'https://www.weibo.com/'
}

time.sleep(random.uniform(1, 3))  # Random delay between requests
```

### Pangu LLM Setup (for Chinese)

```python
# Download and configure Pangu model
# URL: https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

from hotsearch_analysis_agent.llm_analyzer import PanguAnalyzer

analyzer = PanguAnalyzer(
    model_path='/path/to/openpangu-embedded-7b-model',
    device='cuda'  # or 'cpu'
)

sentiment = analyzer.analyze_sentiment(
    "中国大模型应用热度持续领先,连续五周调用量全球第一"
)
```

## Advanced Usage

### Multi-Platform Aggregation

```python
from hotsearch_analysis_agent.aggregator import MultiPlatformAggregator

aggregator = MultiPlatformAggregator()

# Get unified hot topics across platforms
unified_topics = aggregator.aggregate(
    platforms=["weibo", "douyin", "bilibili", "toutiao"],
    hours=6,
    dedup=True,  # Remove duplicates
    top_n=50
)

# Cross-platform trend analysis
trends = aggregator.analyze_trends(
    unified_topics,
    group_by='topic',
    sentiment_weighted=True
)
```

### Scheduled Reporting

```python
from apscheduler.schedulers.background import BackgroundScheduler
from hotsearch_analysis_agent.pipeline import AnalysisPipeline

scheduler = BackgroundScheduler()
pipeline = AnalysisPipeline()

def daily_report():
    report = pipeline.run(
        query="今日热点",
        hours=24,
        cluster=True,
        generate_report=True
    )
    pipeline.push(report, channels=["email"])

# Run daily at 9 AM
scheduler.add_job(daily_report, 'cron', hour=9)
scheduler.start()
```

This skill provides comprehensive coverage of building, deploying, and operating a production-ready public opinion analytics system with LLM-powered insights and multi-channel alerting.
