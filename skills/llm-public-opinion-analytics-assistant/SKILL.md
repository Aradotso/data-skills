---
name: llm-public-opinion-analytics-assistant
description: Use this comprehensive public opinion analytics system that crawls 26 trending lists from 15 platforms and provides LLM-powered sentiment analysis, topic clustering, and multi-channel alert pushing
triggers:
  - "set up public opinion monitoring system"
  - "analyze trending topics across multiple platforms"
  - "crawl hot search data from weibo douyin bilibili"
  - "perform sentiment analysis on news and social media"
  - "create automated trending topic alerts"
  - "cluster and summarize social media discussions"
  - "deploy multi-platform web scraping for trends"
  - "configure hot topic push notifications"
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is a comprehensive public opinion analytics system that integrates real-time data from **26 trending lists across 15 major platforms** (Weibo, Douyin, Bilibili, Zhihu, Baidu, etc.) with large language model analysis capabilities. It provides:

- Conversational hot search queries via natural language
- Topic-specific search and tracking
- Automated topic clustering analysis
- Sentiment tendency analysis
- Multi-channel push notifications (Email, WeChat, Enterprise WeChat, Telegram)
- Web scraping with keyboard shortcuts for crawler control
- Deep content extraction (including video-based news)

The system is split into two independent components:
- **Analysis System**: `hotsearch_analysis_agent/`
- **Crawler Cluster**: `hotsearchcrawler/`

## Installation

### Prerequisites

**1. Browser Driver Setup**

The project requires a browser driver (Chrome/Edge) for content extraction:

```bash
# Check your Chrome version
google-chrome --version  # Linux
# or open chrome://settings/help in browser

# Download matching ChromeDriver from:
# https://chromedriver.chromium.org/

# For Edge:
# https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Place driver in system PATH or project directory
# Linux/macOS example:
sudo mv chromedriver /usr/local/bin/
sudo chmod +x /usr/local/bin/chromedriver

# Verify installation
chromedriver --version
```

**2. MySQL Database**

```bash
# Install MySQL
sudo apt-get install mysql-server  # Ubuntu/Debian
# or brew install mysql  # macOS

# Create database
mysql -u root -p
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

**3. Python Environment**

```bash
# Clone repository
git clone https://github.com/hmmnxkl/LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant.git
cd LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant

# Create virtual environment
python -m venv venv
source venv/bin/activate  # Linux/macOS
# or venv\Scripts\activate  # Windows

# Install dependencies
pip install -r requirements.txt
```

### Database Initialization

```python
# Reference: init.py
import pymysql

def init_database():
    connection = pymysql.connect(
        host='localhost',
        user='your_user',
        password='your_password',
        database='hotsearch_db',
        charset='utf8mb4'
    )
    
    with connection.cursor() as cursor:
        # Create hot search table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS hot_searches (
                id INT AUTO_INCREMENT PRIMARY KEY,
                platform VARCHAR(50),
                title VARCHAR(500),
                url VARCHAR(1000),
                rank_position INT,
                heat_value VARCHAR(100),
                crawl_time DATETIME,
                category VARCHAR(50),
                INDEX idx_platform (platform),
                INDEX idx_crawl_time (crawl_time)
            )
        """)
        
        # Create analysis results table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS analysis_results (
                id INT AUTO_INCREMENT PRIMARY KEY,
                query_text TEXT,
                cluster_result JSON,
                sentiment_analysis JSON,
                related_news JSON,
                created_at DATETIME,
                INDEX idx_created_at (created_at)
            )
        """)
        
        # Create push tasks table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS push_tasks (
                id INT AUTO_INCREMENT PRIMARY KEY,
                task_name VARCHAR(200),
                query_keywords TEXT,
                push_channels JSON,
                schedule_config VARCHAR(100),
                is_active BOOLEAN,
                created_at DATETIME
            )
        """)
    
    connection.commit()
    connection.close()

init_database()
```

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```env
# Database Configuration
DB_HOST=localhost
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
# Using Huawei Pangu model or any OpenAI-compatible endpoint
LLM_API_BASE=http://localhost:8000/v1
LLM_API_KEY=your_api_key_here
LLM_MODEL=pangu-embedded-7b

# Push Notification Channels
# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
EMAIL_RECIPIENTS=recipient1@example.com,recipient2@example.com

# WeChat Enterprise Robot
WECHAT_WEBHOOK_URL=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=your_key

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Browser Driver
CHROME_DRIVER_PATH=/usr/local/bin/chromedriver
```

### Crawler Settings

Edit `hotsearchcrawler/settings.py`:

```python
# MySQL Pipeline Configuration
MYSQL_CONFIG = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME'),
    'charset': 'utf8mb4'
}

# Crawler Settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
ROBOTSTXT_OBEY = False

# Optional: Platform-specific cookies
COOKIES_ENABLED = True
# Add cookies in individual spider files if needed
```

## Running the System

### Start the Web Application

```bash
# Main application entry point
python app.py

# Access web interface at:
# http://localhost:5000
```

### Run Crawlers

**Via Web Interface** (recommended):
- Navigate to the crawler control panel
- Use keyboard shortcuts to start/stop specific platform crawlers

**Via Command Line**:

```bash
# Test individual crawler
python runspider-test.py --spider weibo

# Run all crawlers
python run_spiders.py

# Run specific platform crawlers
cd hotsearchcrawler
scrapy crawl weibo_hot
scrapy crawl douyin_hot
scrapy crawl bilibili_hot
scrapy crawl zhihu_hot
```

### Test Push Notifications

```python
# test_push_task.py
from hotsearch_analysis_agent.push_service import PushService
import os

def test_email_push():
    """Test email notification"""
    push_service = PushService()
    
    report_content = {
        'title': 'AI Technology Hot Topics Analysis',
        'time': '2026-05-15 10:00:00',
        'summary': 'Recent AI technology developments...',
        'topics': [
            {'title': 'GPT-6 Leak', 'url': 'https://...', 'heat': 1500000},
            {'title': 'DeepSeek V4 on Huawei Chips', 'url': 'https://...', 'heat': 980000}
        ],
        'sentiment': {'positive': 0.65, 'neutral': 0.25, 'negative': 0.10}
    }
    
    push_service.send_email(
        subject='[Hot Topic Alert] AI Technology Trends',
        content=report_content
    )

def test_wechat_push():
    """Test WeChat Enterprise notification"""
    push_service = PushService()
    
    push_service.send_wechat({
        'msgtype': 'markdown',
        'markdown': {
            'content': f"""## 🔥 Hot Topic Alert
            
**AI Technology Developments**
- GPT-6 leaked with 2M context window
- DeepSeek V4 using Huawei Ascend chips
            
[View Full Report](http://localhost:5000/reports/latest)
            """
        }
    })

def test_telegram_push():
    """Test Telegram notification"""
    push_service = PushService()
    
    push_service.send_telegram(
        '🚨 *Hot Topic Alert*\n\n'
        '*AI Technology Trends*\n'
        '• GPT-6 leak: 2M context\n'
        '• DeepSeek V4 on Huawei chips\n\n'
        '[View Report](http://localhost:5000/reports/latest)',
        parse_mode='Markdown'
    )

if __name__ == '__main__':
    test_email_push()
    test_wechat_push()
    test_telegram_push()
```

## Key API and Usage Patterns

### Query Hot Search Data

```python
from hotsearch_analysis_agent.query_engine import QueryEngine

# Initialize query engine
engine = QueryEngine()

# Natural language query
result = engine.query("What are the trending AI topics today?")
print(result['response'])
print(result['related_news'])

# Platform-specific query
weibo_trends = engine.query_platform('weibo', limit=20)
for item in weibo_trends:
    print(f"{item['rank']}. {item['title']} - Heat: {item['heat']}")

# Topic search
ai_news = engine.search_topic('artificial intelligence', days=7)
```

### Topic Clustering Analysis

```python
from hotsearch_analysis_agent.analyzer import TopicAnalyzer

analyzer = TopicAnalyzer()

# Cluster related topics
clusters = analyzer.cluster_topics(
    keywords=['AI', 'machine learning', 'LLM'],
    time_range='7d',
    min_cluster_size=3
)

for cluster_id, topics in clusters.items():
    print(f"\n=== Cluster {cluster_id} ===")
    print(f"Theme: {topics['theme']}")
    for topic in topics['items']:
        print(f"  • {topic['title']} ({topic['platform']})")
```

### Sentiment Analysis

```python
from hotsearch_analysis_agent.sentiment import SentimentAnalyzer

analyzer = SentimentAnalyzer()

# Analyze single topic
sentiment = analyzer.analyze_topic("DeepSeek V4 release")
print(f"Sentiment: {sentiment['label']}")  # positive/neutral/negative
print(f"Confidence: {sentiment['score']}")
print(f"Key phrases: {sentiment['key_phrases']}")

# Batch analysis
topics = [
    "New AI model breakthrough",
    "Data privacy concerns",
    "Tech company layoffs"
]
results = analyzer.batch_analyze(topics)
```

### Create Push Task

```python
from hotsearch_analysis_agent.task_manager import TaskManager

task_mgr = TaskManager()

# Create scheduled push task
task = task_mgr.create_task(
    name="AI Tech Daily Digest",
    keywords=["artificial intelligence", "machine learning", "LLM"],
    channels=['email', 'wechat'],
    schedule='0 9 * * *',  # Daily at 9 AM (cron format)
    filters={
        'min_heat': 100000,
        'sentiment': ['positive', 'neutral'],
        'platforms': ['weibo', 'zhihu', 'bilibili']
    }
)

# List active tasks
tasks = task_mgr.list_tasks(active_only=True)

# Update task
task_mgr.update_task(task_id=1, schedule='0 18 * * *')

# Deactivate task
task_mgr.deactivate_task(task_id=1)
```

### Advanced Content Extraction

```python
from hotsearch_analysis_agent.extractor import ContentExtractor

extractor = ContentExtractor()

# Extract from news page
content = extractor.extract_url(
    'https://www.example.com/news/ai-breakthrough',
    extract_video=True  # Also extract video transcripts
)

print(f"Title: {content['title']}")
print(f"Text: {content['text']}")
print(f"Images: {content['images']}")
if content.get('video_transcript'):
    print(f"Video content: {content['video_transcript']}")

# Batch extraction
urls = [
    'https://news1.com/article',
    'https://news2.com/video',
    'https://news3.com/post'
]
results = extractor.batch_extract(urls, max_workers=5)
```

### LLM-Powered Analysis

```python
from hotsearch_analysis_agent.llm_client import LLMClient

# Initialize LLM client (OpenAI-compatible)
llm = LLMClient(
    api_base=os.getenv('LLM_API_BASE'),
    api_key=os.getenv('LLM_API_KEY'),
    model=os.getenv('LLM_MODEL', 'pangu-embedded-7b')
)

# Generate analysis report
news_items = [
    {'title': 'GPT-6 leak...', 'content': '...', 'url': '...'},
    {'title': 'DeepSeek V4...', 'content': '...', 'url': '...'},
]

report = llm.generate_report(
    topic="AI Technology Trends",
    news_items=news_items,
    analysis_type='comprehensive'
)

print(report['summary'])
print(report['key_findings'])
print(report['sentiment_overview'])
```

## Common Workflows

### Workflow 1: Monitor Specific Topic

```python
from hotsearch_analysis_agent import monitor_topic

# Set up continuous monitoring
monitor = monitor_topic(
    keywords=['COVID-19', 'pandemic'],
    platforms=['weibo', 'zhihu', 'toutiao'],
    alert_threshold=500000,  # Heat value threshold
    notification_channels=['email', 'telegram'],
    check_interval=300  # Check every 5 minutes
)

# The monitor runs in background and sends alerts
# when matching topics exceed threshold
```

### Workflow 2: Daily Digest Generation

```python
from hotsearch_analysis_agent.digest import DailyDigest
from datetime import datetime, timedelta

digest = DailyDigest()

# Generate yesterday's digest
report = digest.generate(
    date=datetime.now() - timedelta(days=1),
    categories=['tech', 'finance', 'entertainment'],
    top_n=10,  # Top 10 per category
    include_clustering=True,
    include_sentiment=True
)

# Auto-send via configured channels
digest.send_report(report, channels=['email', 'wechat'])
```

### Workflow 3: Multi-Platform Comparison

```python
from hotsearch_analysis_agent.comparison import PlatformComparison

comp = PlatformComparison()

# Compare trending topics across platforms
analysis = comp.compare_platforms(
    platforms=['weibo', 'douyin', 'bilibili'],
    time_range='24h',
    metrics=['overlap', 'unique_topics', 'sentiment_divergence']
)

print(f"Topic overlap: {analysis['overlap_rate']}")
print(f"Platform-unique topics:")
for platform, topics in analysis['unique_topics'].items():
    print(f"  {platform}: {len(topics)} unique")
    
print(f"Sentiment divergence: {analysis['sentiment_divergence']}")
```

## Crawler Spider Examples

### Custom Spider for New Platform

```python
# hotsearchcrawler/spiders/custom_platform.py
import scrapy
from hotsearchcrawler.items import HotSearchItem
from datetime import datetime

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform_hot'
    allowed_domains = ['customplatform.com']
    start_urls = ['https://customplatform.com/trending']
    
    def parse(self, response):
        # Extract trending items
        for rank, item in enumerate(response.css('.trending-item'), 1):
            hot_item = HotSearchItem()
            hot_item['platform'] = 'CustomPlatform'
            hot_item['title'] = item.css('.title::text').get()
            hot_item['url'] = item.css('a::attr(href)').get()
            hot_item['rank_position'] = rank
            hot_item['heat_value'] = item.css('.heat::text').get()
            hot_item['crawl_time'] = datetime.now()
            hot_item['category'] = item.css('.category::text').get()
            
            yield hot_item
```

## Troubleshooting

### ChromeDriver Issues

```bash
# Error: ChromeDriver version mismatch
# Solution: Download matching version
chrome --version
# Download corresponding driver from chromedriver.chromium.org

# Error: ChromeDriver not in PATH
export PATH=$PATH:/path/to/driver
# Or specify in .env:
CHROME_DRIVER_PATH=/usr/local/bin/chromedriver
```

### Database Connection Errors

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
    print("✓ Database connected")
    conn.close()
except Exception as e:
    print(f"✗ Connection failed: {e}")
    # Check: MySQL service running, credentials correct, database exists
```

### Crawler Rate Limiting

```python
# Adjust in hotsearchcrawler/settings.py
CONCURRENT_REQUESTS = 8  # Reduce concurrent requests
DOWNLOAD_DELAY = 2  # Increase delay between requests
AUTOTHROTTLE_ENABLED = True
AUTOTHROTTLE_START_DELAY = 1
AUTOTHROTTLE_MAX_DELAY = 10
```

### LLM API Timeout

```python
# Increase timeout in LLM client
llm = LLMClient(
    api_base=os.getenv('LLM_API_BASE'),
    api_key=os.getenv('LLM_API_KEY'),
    timeout=120,  # Increase to 120 seconds
    max_retries=3
)
```

### Push Notification Failures

```python
# Debug push service
from hotsearch_analysis_agent.push_service import PushService

push = PushService(debug=True)  # Enable debug logging

# Test each channel individually
try:
    push.test_email()
    print("✓ Email working")
except Exception as e:
    print(f"✗ Email failed: {e}")

try:
    push.test_wechat()
    print("✓ WeChat working")
except Exception as e:
    print(f"✗ WeChat failed: {e}")
```

### Memory Issues with Large Datasets

```python
# Use pagination for large queries
from hotsearch_analysis_agent.query_engine import QueryEngine

engine = QueryEngine()

# Instead of loading all at once:
# results = engine.query_all(time_range='30d')  # ✗ Memory intensive

# Use pagination:
page_size = 1000
offset = 0
while True:
    batch = engine.query_paginated(
        time_range='30d',
        limit=page_size,
        offset=offset
    )
    if not batch:
        break
    
    # Process batch
    for item in batch:
        # ... process item ...
        pass
    
    offset += page_size
```

## Performance Tips

- **Enable caching** for frequently accessed trending data
- **Use database indexes** on `platform`, `crawl_time`, and `heat_value` columns
- **Batch LLM API calls** when analyzing multiple topics
- **Schedule crawlers** during off-peak hours to reduce server load
- **Compress historical data** older than 30 days
