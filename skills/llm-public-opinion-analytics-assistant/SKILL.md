---
name: llm-public-opinion-analytics-assistant
description: Build multi-platform public opinion monitoring systems with hot search aggregation, LLM analysis, sentiment detection, and multi-channel alerting
triggers:
  - how do I set up a public opinion monitoring system
  - scrape hot search trends from multiple platforms
  - analyze sentiment and cluster topics with LLM
  - create a hot topic alert system with push notifications
  - build a web crawler for social media trends
  - implement automated opinion analysis with AI
  - aggregate and analyze trending topics across platforms
  - set up multi-channel notifications for trending news
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is an intelligent public opinion analytics system that aggregates real-time data from 15 mainstream platforms (26 different ranking lists), combines it with LLM analysis capabilities, and provides conversational hot search queries, topic clustering, sentiment analysis, and multi-channel push notifications.

## What It Does

- **Multi-Platform Data Collection**: Scrapes hot search rankings from 15 platforms including Weibo, Bilibili, Douyin, Zhihu, Baidu, etc.
- **LLM-Powered Analysis**: Uses large language models (supports Huawei Pangu, OpenAI-compatible APIs) for topic clustering and sentiment analysis
- **Conversational Interface**: Web-based chat interface for querying trends and analyzing topics
- **Content Extraction**: Extracts content from news detail pages including video transcripts
- **Multi-Channel Alerts**: Pushes analysis reports via WeChat Work, Telegram, email (SMTP)
- **Crawler Control**: Hotkey support for starting/stopping crawlers from the frontend

## Architecture

The project consists of two main components:

1. **Crawler Cluster** (`hotsearchcrawler/`) - Scrapy-based distributed crawlers for data collection
2. **Analysis System** (`hotsearch_analysis_agent/`) - Flask backend with LLM integration and frontend interface

## Installation

### Prerequisites

**Browser Driver Setup** (for dynamic content extraction):

```bash
# Download ChromeDriver or EdgeDriver matching your browser version
# ChromeDriver: https://chromedriver.chromium.org/
# EdgeDriver: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Place driver in system PATH or browser installation directory
# Verify installation:
chromedriver --version
```

**Python Environment**:

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

**Database Setup**:

```bash
# Install MySQL and create database
# Reference init.py for table schemas

mysql -u root -p
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

## Configuration

### Environment Variables (`.env`)

```bash
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch_db

# LLM API Configuration (OpenAI-compatible format)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
OPENAI_MODEL=gpt-4

# Or use Huawei Pangu model
PANGU_API_KEY=your_pangu_key
PANGU_MODEL_NAME=openpangu-embedded-7b

# Push Notification Settings
# WeChat Work Robot
WECHAT_WORK_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email (SMTP)
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
```

### Crawler Settings (`hotsearchcrawler/settings.py`)

```python
# MySQL connection for crawler
MYSQL_CONFIG = {
    'host': 'localhost',
    'port': 3306,
    'user': 'your_user',
    'password': 'your_password',
    'database': 'hotsearch_db',
    'charset': 'utf8mb4'
}

# Optional: Platform-specific cookies
COOKIES = {
    'weibo': 'your_weibo_cookies',
    'douyin': 'your_douyin_cookies'
}

# Concurrent requests
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
```

## Starting the System

### Run Analysis System (Backend + Frontend)

```bash
python app.py
# Access web interface at http://localhost:5000
```

### Run Crawlers

**From Frontend**: Use the web interface hotkey to start/stop crawlers

**Manual Execution**:

```bash
# Test single platform
python runspider-test.py

# Run all spiders
python run_spiders.py
```

**Scheduled Crawling** (production):

```bash
# Add to crontab for hourly updates
0 * * * * cd /path/to/project && /path/to/venv/bin/python run_spiders.py
```

## Core Usage Patterns

### 1. Querying Hot Search Data

**Natural Language Query**:

```python
# In the web chat interface, type:
"显示微博热搜前10"
"最近关于人工智能的新闻"
"分析科技类话题的情感倾向"
```

**Direct API Call**:

```python
from hotsearch_analysis_agent.database import get_hot_searches

# Get top 20 from Weibo
results = get_hot_searches(platform='weibo', limit=20)

for item in results:
    print(f"{item['rank']}. {item['title']} - {item['heat']}")
```

### 2. Topic Clustering Analysis

```python
from hotsearch_analysis_agent.llm_service import analyze_topics

# Analyze related topics
query = "人工智能"
analysis = analyze_topics(query, days=7)

print(analysis['summary'])
print(f"Sentiment: {analysis['sentiment']}")
print(f"Key Topics: {', '.join(analysis['clusters'])}")
```

### 3. Setting Up Push Notifications

**Create Push Task**:

```python
from hotsearch_analysis_agent.push_service import create_push_task

task = create_push_task(
    query="人工智能与前沿科技",
    channels=['wechat_work', 'telegram', 'email'],
    schedule='0 12 * * *',  # Daily at noon
    min_relevance=0.7
)

print(f"Task created: {task['id']}")
```

**Test Push Task**:

```bash
python test_push_task.py
```

### 4. Extracting Content from URLs

```python
from hotsearch_analysis_agent.content_extractor import extract_content

# Extract article/video content
url = "https://www.bilibili.com/video/BV13pSoBBEvX"
content = extract_content(url)

print(f"Title: {content['title']}")
print(f"Summary: {content['summary']}")
print(f"Transcript: {content['text'][:500]}...")
```

### 5. Custom Crawler Development

**Add New Platform Spider**:

```python
# hotsearchcrawler/spiders/newplatform.py
import scrapy
from hotsearchcrawler.items import HotSearchItem

class NewPlatformSpider(scrapy.Spider):
    name = 'newplatform'
    start_urls = ['https://newplatform.com/hot']
    
    def parse(self, response):
        for idx, item in enumerate(response.css('.hot-item')):
            yield HotSearchItem(
                platform='newplatform',
                rank=idx + 1,
                title=item.css('.title::text').get(),
                url=item.css('a::attr(href)').get(),
                heat=item.css('.heat::text').get(),
                category=item.css('.tag::text').get()
            )
```

**Register in Pipeline**:

```python
# hotsearchcrawler/pipelines.py
class MySQLPipeline:
    def process_item(self, item, spider):
        # Insert into database
        self.cursor.execute("""
            INSERT INTO hot_searches 
            (platform, rank, title, url, heat, category, created_at)
            VALUES (%s, %s, %s, %s, %s, %s, NOW())
        """, (
            item['platform'],
            item['rank'],
            item['title'],
            item['url'],
            item['heat'],
            item.get('category', '')
        ))
        return item
```

## Database Schema Reference

```sql
-- Hot search records
CREATE TABLE hot_searches (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    platform VARCHAR(50) NOT NULL,
    rank INT NOT NULL,
    title VARCHAR(500) NOT NULL,
    url TEXT,
    heat VARCHAR(100),
    category VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_platform_time (platform, created_at),
    INDEX idx_title (title(255))
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Analysis results cache
CREATE TABLE analysis_cache (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    query VARCHAR(500) NOT NULL,
    result_type VARCHAR(50),
    result_data JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_query (query(255))
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Push tasks
CREATE TABLE push_tasks (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    query VARCHAR(500) NOT NULL,
    channels JSON,
    schedule VARCHAR(100),
    last_run TIMESTAMP NULL,
    status VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## Supported Platforms

| Platform | Spider Name | Content Type |
|----------|-------------|--------------|
| Weibo | `weibo` | Hot search, trending topics |
| Bilibili | `bilibili` | Video hot list |
| Douyin | `douyin` | Hot videos, trending |
| Zhihu | `zhihu` | Hot questions |
| Baidu | `baidu` | Search trends |
| Toutiao | `toutiao` | News hot list |
| 36Kr | `36kr` | Tech news |
| Sina | `sina` | News trends |

## Common Troubleshooting

### Crawler Issues

**Problem**: Driver not found error

```bash
# Solution: Verify driver in PATH
which chromedriver  # Linux/Mac
where chromedriver  # Windows

# Or set explicit path in settings.py
CHROME_DRIVER_PATH = '/usr/local/bin/chromedriver'
```

**Problem**: Anti-scraping blocks (403, captcha)

```python
# Add delays and user agents in settings.py
DOWNLOAD_DELAY = 3
RANDOMIZE_DOWNLOAD_DELAY = True

USER_AGENT = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'

# Use cookies for authenticated platforms
COOKIES = {'platform_name': 'your_cookies_here'}
```

**Problem**: MySQL connection timeout

```python
# Increase pool size in settings.py
MYSQL_CONFIG = {
    'pool_size': 10,
    'max_overflow': 20,
    'pool_recycle': 3600
}
```

### LLM Analysis Issues

**Problem**: API rate limits

```python
# Add retry logic with exponential backoff
import time
from tenacity import retry, wait_exponential, stop_after_attempt

@retry(wait=wait_exponential(min=1, max=60), stop=stop_after_attempt(5))
def call_llm_api(prompt):
    # Your API call here
    pass
```

**Problem**: Out of memory with local Pangu model

```bash
# Use quantized model or reduce batch size
# Download INT4 quantized version:
# https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

# Or use API mode instead of local deployment
```

### Push Notification Issues

**Problem**: WeChat Work webhook not receiving

```python
# Verify webhook format
import requests

webhook_url = "https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY"
data = {
    "msgtype": "markdown",
    "markdown": {
        "content": "Test message"
    }
}
response = requests.post(webhook_url, json=data)
print(response.json())  # Should return {"errcode":0,"errmsg":"ok"}
```

**Problem**: Email SMTP authentication failed

```python
# For Gmail, use App Password instead of account password
# Enable 2FA, then generate app password at:
# https://myaccount.google.com/apppasswords

SMTP_PASSWORD = "your_16_char_app_password"
```

## Performance Optimization

**Parallel Crawler Execution**:

```python
# run_spiders.py with multiprocessing
from multiprocessing import Process
from scrapy.crawler import CrawlerProcess

def run_spider(spider_name):
    process = CrawlerProcess(get_project_settings())
    process.crawl(spider_name)
    process.start()

if __name__ == '__main__':
    spiders = ['weibo', 'bilibili', 'douyin', 'zhihu']
    processes = [Process(target=run_spider, args=(s,)) for s in spiders]
    
    for p in processes:
        p.start()
    
    for p in processes:
        p.join()
```

**Database Query Optimization**:

```python
# Add composite indexes for common queries
CREATE INDEX idx_platform_rank_time 
ON hot_searches(platform, rank, created_at DESC);

# Use query result caching
from functools import lru_cache

@lru_cache(maxsize=128)
def get_platform_trends(platform, hours=24):
    # Cache results for repeated queries
    pass
```

This skill enables AI coding agents to help developers build, configure, and extend multi-platform public opinion monitoring systems with LLM-powered analysis and automated alerting capabilities.
