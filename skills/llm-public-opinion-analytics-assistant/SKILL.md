---
name: llm-public-opinion-analytics-assistant
description: Multi-platform real-time public opinion analysis system with LLM-powered sentiment analysis, topic clustering, and multi-channel alert pushing
triggers:
  - analyze public opinion trends across social platforms
  - set up hot topic monitoring and alerts
  - crawl trending news from multiple platforms
  - perform sentiment analysis on social media topics
  - cluster related news topics using AI
  - configure multi-channel notification for trending topics
  - analyze public sentiment with large language models
  - monitor real-time hot search rankings
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is an intelligent public opinion analysis assistant that integrates real-time data from **15 mainstream platforms** across **26 trending lists**. It combines web scraping, LLM analysis, and multi-channel alerting to provide:

- **Conversational hot topic queries** via natural language interface
- **Topic clustering and sentiment analysis** using large language models (Pangu, OpenAI-compatible APIs)
- **Multi-platform data aggregation** (Weibo, Bilibili, Douyin, Zhihu, Baidu, etc.)
- **Multi-channel push notifications** (Email, WeChat Work, Telegram)
- **Video content analysis** - extracts insights even from video-based news
- **Hotkey-controlled crawler management** from the frontend

The architecture separates the crawler cluster (`hotsearchcrawler`) from the analysis system (`hotsearch_analysis_agent`), allowing independent scaling and deployment.

## Installation

### Prerequisites

1. **Browser Driver Setup** (required for content extraction):

```bash
# Download ChromeDriver matching your Chrome version
# from https://chromedriver.chromium.org/
# or EdgeDriver from https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Verify installation
chromedriver --version

# Add to PATH (Linux/macOS example)
export PATH=$PATH:/path/to/driver/directory
```

2. **Python Environment**:

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

3. **MySQL Database**:

```bash
# Install MySQL and create database
mysql -u root -p
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### Database Initialization

Reference `init.py` for schema creation:

```python
import pymysql

connection = pymysql.connect(
    host='localhost',
    user='root',
    password='your_password',
    database='hotsearch_db',
    charset='utf8mb4'
)

cursor = connection.cursor()

# Create trending topics table
cursor.execute("""
CREATE TABLE IF NOT EXISTS trending_topics (
    id INT AUTO_INCREMENT PRIMARY KEY,
    platform VARCHAR(50),
    title VARCHAR(500),
    url VARCHAR(1000),
    rank INT,
    heat_value VARCHAR(100),
    crawl_time DATETIME,
    content TEXT,
    sentiment VARCHAR(20),
    INDEX idx_platform_time (platform, crawl_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
""")

connection.commit()
connection.close()
```

### Configuration

1. **Crawler Configuration** (`hotsearchcrawler/settings.py`):

```python
# MySQL settings
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'root'
MYSQL_PASSWORD = os.getenv('MYSQL_PASSWORD')
MYSQL_DB = 'hotsearch_db'

# Optional: Platform-specific cookies
COOKIES = {
    'weibo': 'your_weibo_cookie',  # Optional for better access
}
```

2. **Analysis System Configuration** (`.env` file):

```bash
# Database
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=your_password
MYSQL_DB=hotsearch_db

# LLM API (OpenAI-compatible)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Or Pangu model (local deployment)
USE_PANGU=true
PANGU_MODEL_PATH=/path/to/pangu/model

# Push notification channels
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password

WECHAT_WORK_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=your_key
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

## Core Usage Patterns

### Starting the Crawler System

```bash
# Test individual spider
python runspider-test.py

# Run all spiders (production)
python run_spiders.py
```

The crawler can also be controlled via frontend hotkeys once the web interface is running.

### Starting the Analysis System

```bash
# Launch web interface and API
python app.py
```

Access the frontend at `http://localhost:5000` for conversational queries.

### Programmatic Analysis

```python
from hotsearch_analysis_agent.analyzer import TopicAnalyzer
from hotsearch_analysis_agent.db import MySQLClient

# Initialize components
db = MySQLClient()
analyzer = TopicAnalyzer(model_name=os.getenv('MODEL_NAME'))

# Fetch recent trending topics
topics = db.query("""
    SELECT platform, title, url, content, crawl_time
    FROM trending_topics
    WHERE crawl_time > NOW() - INTERVAL 24 HOUR
    ORDER BY crawl_time DESC
""")

# Perform sentiment analysis
for topic in topics:
    sentiment = analyzer.analyze_sentiment(topic['content'])
    db.update_sentiment(topic['id'], sentiment)
    print(f"{topic['title']}: {sentiment}")
```

### Topic Clustering

```python
from hotsearch_analysis_agent.clustering import TopicClusterer

clusterer = TopicClusterer(min_similarity=0.7)

# Cluster related topics
clusters = clusterer.cluster_topics(topics, fields=['title', 'content'])

for cluster_id, cluster_topics in clusters.items():
    print(f"\nCluster {cluster_id}:")
    for topic in cluster_topics:
        print(f"  - {topic['title']} ({topic['platform']})")
```

### Multi-Channel Push Notifications

```python
from hotsearch_analysis_agent.push import PushManager

push_manager = PushManager()

# Generate analysis report
report = analyzer.generate_report(
    query="人工智能与前沿科技",
    topics=topics,
    clusters=clusters
)

# Push via multiple channels
push_manager.send_email(
    to=['analyst@company.com'],
    subject="热点分析报告 - AI领域",
    content=report
)

push_manager.send_wechat_work(
    webhook_url=os.getenv('WECHAT_WORK_WEBHOOK'),
    content=report
)

push_manager.send_telegram(
    bot_token=os.getenv('TELEGRAM_BOT_TOKEN'),
    chat_id=os.getenv('TELEGRAM_CHAT_ID'),
    message=report
)
```

### Testing Push Tasks

```bash
# Test all configured push channels
python test_push_task.py
```

## Advanced Patterns

### Custom Spider for New Platform

```python
# hotsearchcrawler/spiders/new_platform.py
import scrapy
from hotsearchcrawler.items import HotSearchItem

class NewPlatformSpider(scrapy.Spider):
    name = 'new_platform'
    start_urls = ['https://newplatform.com/trending']
    
    def parse(self, response):
        for item in response.css('.trending-item'):
            yield HotSearchItem(
                platform='NewPlatform',
                title=item.css('.title::text').get(),
                url=item.css('a::attr(href)').get(),
                rank=item.css('.rank::text').get(),
                heat_value=item.css('.heat::text').get()
            )
```

### Video Content Extraction

The system automatically extracts content from video platforms:

```python
from hotsearch_analysis_agent.video_extractor import VideoContentExtractor

extractor = VideoContentExtractor()

# Extract insights from video news
video_url = "https://www.bilibili.com/video/BV13pSoBBEvX"
content = extractor.extract(video_url)

# Returns structured data including:
# - title, description, tags
# - transcript (if available)
# - comments summary
# - visual content analysis
print(content['summary'])
```

### Scheduled Monitoring Tasks

```python
from hotsearch_analysis_agent.scheduler import MonitoringScheduler

scheduler = MonitoringScheduler()

# Monitor specific keywords with custom filters
scheduler.add_task(
    keywords=["芯片", "半导体", "AI"],
    platforms=["weibo", "zhihu", "36kr"],
    interval_hours=2,
    min_heat=5000,
    callback=lambda topics: push_manager.send_alert(topics)
)

scheduler.start()
```

### LLM-Powered Conversational Query

```python
from hotsearch_analysis_agent.chat import ChatInterface

chat = ChatInterface()

# Natural language queries
response = chat.query("最近三天关于人工智能的热门话题有哪些?")
print(response)

# Follow-up with context
response = chat.query("帮我分析一下这些话题的情感倾向")
print(response)

# Request clustering
response = chat.query("把相关的话题聚类并生成报告")
print(response)
```

## Configuration Examples

### Using Pangu Model (Local)

```python
# hotsearch_analysis_agent/config.py
LLM_CONFIG = {
    'type': 'pangu',
    'model_path': '/models/openpangu-embedded-7b',
    'device': 'cuda:0',  # or 'cpu'
    'max_length': 2048,
    'temperature': 0.7
}
```

### Using OpenAI-Compatible API

```python
# hotsearch_analysis_agent/config.py
LLM_CONFIG = {
    'type': 'openai',
    'api_base': os.getenv('OPENAI_API_BASE'),
    'api_key': os.getenv('OPENAI_API_KEY'),
    'model': 'gpt-4',
    'temperature': 0.7
}
```

## Common Tasks

### Export Analysis Report

```python
from hotsearch_analysis_agent.export import ReportExporter

exporter = ReportExporter()

# Export to markdown
exporter.to_markdown(report, 'report_2026-05-15.md')

# Export to PDF
exporter.to_pdf(report, 'report_2026-05-15.pdf')

# Export to structured JSON
exporter.to_json(report, 'report_2026-05-15.json')
```

### Query Historical Trends

```python
# Analyze trend evolution
trend_analyzer = TrendAnalyzer(db)

trend = trend_analyzer.analyze_keyword(
    keyword="AI",
    start_date="2026-05-01",
    end_date="2026-05-15",
    platforms=["weibo", "zhihu", "douyin"]
)

# Visualize trend
trend.plot(output='ai_trend.png')
```

## Troubleshooting

### Crawler Issues

**Problem**: ChromeDriver version mismatch
```bash
# Check Chrome version
google-chrome --version

# Download matching driver version
# Ensure driver is in PATH
which chromedriver
```

**Problem**: Rate limiting from platforms
```python
# hotsearchcrawler/settings.py
DOWNLOAD_DELAY = 3  # Increase delay between requests
CONCURRENT_REQUESTS = 4  # Reduce concurrency
COOKIES = {...}  # Add authentication cookies
```

### Database Connection Issues

```python
# Test connection
python -c "import pymysql; pymysql.connect(host='localhost', user='root', password='your_pwd', db='hotsearch_db')"

# Check encoding
# ALTER DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### LLM Analysis Errors

**Problem**: Out of memory with local Pangu model
```python
# Reduce batch size
LLM_CONFIG['batch_size'] = 1
# Use CPU if GPU memory insufficient
LLM_CONFIG['device'] = 'cpu'
```

**Problem**: API rate limits
```python
# Add retry logic
from tenacity import retry, wait_exponential

@retry(wait=wait_exponential(min=1, max=60))
def analyze_with_retry(content):
    return analyzer.analyze_sentiment(content)
```

### Push Notification Failures

```bash
# Test SMTP connection
python -c "import smtplib; smtplib.SMTP('smtp.gmail.com', 587).starttls()"

# Verify webhook URLs
curl -X POST $WECHAT_WORK_WEBHOOK -d '{"msgtype":"text","text":{"content":"test"}}'

# Check Telegram bot token
curl https://api.telegram.org/bot$TELEGRAM_BOT_TOKEN/getMe
```

## Performance Optimization

### Parallel Crawling

```python
# run_spiders.py
from scrapy.crawler import CrawlerProcess
from scrapy.utils.project import get_project_settings

process = CrawlerProcess(get_project_settings())

# Run spiders in parallel
spiders = ['weibo', 'zhihu', 'douyin', 'bilibili']
for spider in spiders:
    process.crawl(spider)

process.start()
```

### Caching Analysis Results

```python
from functools import lru_cache

@lru_cache(maxsize=1000)
def cached_sentiment_analysis(content_hash):
    return analyzer.analyze_sentiment(content)

# Use hash for cache key
import hashlib
content_hash = hashlib.md5(content.encode()).hexdigest()
sentiment = cached_sentiment_analysis(content_hash)
```

This skill enables AI agents to help developers deploy, configure, and extend this comprehensive public opinion monitoring system with LLM-powered analysis capabilities.
