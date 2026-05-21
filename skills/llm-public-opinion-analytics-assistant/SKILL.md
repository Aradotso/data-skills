---
name: llm-public-opinion-analytics-assistant
description: Multi-platform hot search crawler and LLM-powered public opinion analysis assistant with clustering, sentiment analysis, and multi-channel alerts
triggers:
  - set up public opinion monitoring system
  - analyze hot topics from social platforms
  - crawl trending news from multiple platforms
  - configure sentiment analysis for news feeds
  - create hot topic alert notifications
  - build real-time social media monitoring
  - implement news clustering with LLM
  - set up multi-platform trend tracking
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is an intelligent public opinion analytics assistant that combines real-time data from **15 mainstream platforms** (26 ranking lists total) with large language model analysis capabilities. It provides conversational hot search queries, topic clustering, sentiment analysis, and multi-channel alert push notifications via email, WeChat, Enterprise WeChat, and Telegram.

**Key capabilities:**
- Real-time crawling of trending topics from 15+ platforms (Weibo, Bilibili, Zhihu, Douyin, etc.)
- LLM-powered analysis: clustering, sentiment detection, report generation
- Detailed content extraction (including video platform content)
- Multi-channel push notifications (Enterprise WeChat, Telegram, email)
- Web UI for conversational queries and control
- Keyboard shortcuts to start/stop crawlers

## Architecture

The project consists of two independent systems:

1. **Crawler Cluster** (`hotsearchcrawler/`) - Scrapy-based distributed crawler
2. **Analysis System** (`hotsearch_analysis_agent/`) - LLM-powered analytics engine with web UI

Both systems communicate through a MySQL database.

## Installation

### Prerequisites

**1. Browser Driver Setup**

For content extraction, you need either Chrome/Chromium or Edge browser driver:

```bash
# Check your browser version first
google-chrome --version  # or: microsoft-edge --version

# Download matching driver version:
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Move driver to system PATH
# Linux/macOS:
sudo mv chromedriver /usr/local/bin/
sudo chmod +x /usr/local/bin/chromedriver

# Windows: Add driver directory to PATH environment variable
# Verify installation:
chromedriver --version
```

**2. MySQL Database**

```bash
# Install MySQL (example for Ubuntu)
sudo apt-get install mysql-server

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
python3 -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Database Initialization

Use the reference file to create tables:

```python
# Reference init.py for table schemas
# Key tables include:
# - hot_search_items (crawled trending topics)
# - analysis_results (LLM analysis outputs)
# - push_tasks (scheduled alert configurations)
# - user_queries (conversation history)
```

## Configuration

### 1. Crawler Configuration

Edit `hotsearchcrawler/settings.py`:

```python
# MySQL connection
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_user'
MYSQL_PASSWORD = 'your_password'
MYSQL_DATABASE = 'hotsearch_db'

# Optional: Platform-specific cookies for authenticated endpoints
# Add to hotsearchcrawler/cookies/ directory if needed
```

### 2. Analysis System Configuration

Create `.env` file in project root:

```env
# Database
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Alternative: Huawei Pangu Model (recommended for Chinese content)
# PANGU_API_KEY=your_pangu_key
# PANGU_MODEL_PATH=/path/to/local/model

# Push Notification Channels (optional)
# Enterprise WeChat
WECHAT_WEBHOOK_URL=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=your_key

# Telegram
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
EMAIL_RECIPIENTS=recipient1@example.com,recipient2@example.com
```

## Usage

### Starting the System

```bash
# 1. Start the web application
python app.py
# Access UI at http://localhost:5000

# 2. Start crawlers (via UI or command line)
# Via UI: Use keyboard shortcuts or click "Start Crawlers" button
# Via command line:
python run_spiders.py
```

### Crawler Management

```python
# Test individual spider
cd hotsearchcrawler
scrapy crawl weibo_hot  # Test Weibo crawler

# Available spiders (partial list):
# weibo_hot, bilibili_hot, zhihu_hot, douyin_hot, baidu_hot, etc.

# Run all spiders
python runspider-test.py
```

### Conversational Queries

Through the web UI, you can ask natural language questions:

```python
# Example queries:
# "Show me today's top 10 trending topics on Weibo"
# "What are people talking about AI recently?"
# "Analyze sentiment around [topic name]"
# "Cluster news about technology trends"
# "Show me trending videos on Bilibili about science"
```

### Programmatic Analysis

```python
from hotsearch_analysis_agent.analyzer import OpinionAnalyzer
from hotsearch_analysis_agent.database import DatabaseManager

# Initialize
db = DatabaseManager()
analyzer = OpinionAnalyzer(model_name="gpt-4")

# Fetch recent hot search items
items = db.get_recent_items(platform="weibo", limit=50)

# Perform clustering analysis
clusters = analyzer.cluster_topics(items)
print(f"Found {len(clusters)} topic clusters")

# Sentiment analysis
for cluster in clusters:
    sentiment = analyzer.analyze_sentiment(cluster['items'])
    print(f"Cluster: {cluster['name']}")
    print(f"Sentiment: {sentiment['overall']} (positive: {sentiment['positive_ratio']})")

# Generate report
report = analyzer.generate_report(
    clusters=clusters,
    topic="人工智能与前沿科技",
    time_range="2026-04-01 to 2026-04-07"
)
print(report)
```

### Setting Up Push Notifications

```python
from hotsearch_analysis_agent.push_manager import PushTaskManager

# Initialize
push_mgr = PushTaskManager()

# Create scheduled push task
task = push_mgr.create_task(
    name="AI Tech Daily Digest",
    query_keywords=["人工智能", "大模型", "芯片"],
    schedule="0 9 * * *",  # Daily at 9:00 AM (cron format)
    channels=["wechat", "telegram", "email"],
    analysis_depth="detailed"  # or "summary"
)

# Test push notification
from test_push_task import test_push

test_push(
    topic="人工智能",
    channels=["wechat"],
    test_mode=True
)
```

### Database Queries

```python
from hotsearch_analysis_agent.database import DatabaseManager

db = DatabaseManager()

# Get trending items by platform
weibo_trends = db.query(
    """
    SELECT title, hot_value, url, create_time 
    FROM hot_search_items 
    WHERE platform = 'weibo' 
    AND create_time >= DATE_SUB(NOW(), INTERVAL 24 HOUR)
    ORDER BY hot_value DESC 
    LIMIT 20
    """
)

# Search by keyword
ai_news = db.search_items(
    keywords=["人工智能", "AI"],
    platforms=["weibo", "zhihu", "bilibili"],
    date_from="2026-04-01",
    date_to="2026-04-07"
)

# Get detailed content (including video transcripts)
for item in ai_news:
    detail = db.get_item_detail(item['id'])
    print(f"Title: {detail['title']}")
    print(f"Content: {detail['content'][:200]}...")  # Extracted from detail page
```

## Common Patterns

### Pattern 1: Daily Trend Monitoring

```python
# Schedule crawler to run every hour
import schedule
import time
from run_spiders import run_all_spiders

def monitor_job():
    print("Starting crawler run...")
    run_all_spiders()
    
schedule.every(1).hours.do(monitor_job)

while True:
    schedule.run_pending()
    time.sleep(60)
```

### Pattern 2: Custom Topic Analysis Pipeline

```python
from hotsearch_analysis_agent.analyzer import OpinionAnalyzer
from hotsearch_analysis_agent.database import DatabaseManager

def analyze_topic_pipeline(topic_keywords, days_back=7):
    """Complete analysis pipeline for a topic"""
    db = DatabaseManager()
    analyzer = OpinionAnalyzer()
    
    # 1. Fetch relevant items
    items = db.search_items(
        keywords=topic_keywords,
        date_from=f"-{days_back} days",
        limit=200
    )
    
    # 2. Cluster related topics
    clusters = analyzer.cluster_topics(items, num_clusters=5)
    
    # 3. Sentiment analysis per cluster
    results = []
    for cluster in clusters:
        sentiment = analyzer.analyze_sentiment(cluster['items'])
        summary = analyzer.summarize_cluster(cluster)
        
        results.append({
            'cluster_name': cluster['name'],
            'item_count': len(cluster['items']),
            'sentiment': sentiment,
            'summary': summary,
            'top_sources': cluster['top_platforms']
        })
    
    # 4. Generate report
    report = analyzer.generate_report(
        clusters=clusters,
        topic=" ".join(topic_keywords),
        analysis_results=results
    )
    
    return report

# Usage
report = analyze_topic_pipeline(["人工智能", "GPT", "大模型"], days_back=7)
print(report)
```

### Pattern 3: Multi-Channel Alert System

```python
from hotsearch_analysis_agent.push_manager import PushTaskManager
from hotsearch_analysis_agent.analyzer import OpinionAnalyzer

def setup_alert_system():
    """Configure alerts for breaking news"""
    push_mgr = PushTaskManager()
    
    # High-priority topics
    critical_topics = [
        {
            'name': 'Tech Breaking News',
            'keywords': ['突发', '重大', '发布会'],
            'threshold': 100000,  # Hot value threshold
            'channels': ['wechat', 'telegram']
        },
        {
            'name': 'Industry Policy Updates',
            'keywords': ['政策', '监管', '新规'],
            'threshold': 50000,
            'channels': ['email']
        }
    ]
    
    for topic in critical_topics:
        push_mgr.create_realtime_alert(
            name=topic['name'],
            keywords=topic['keywords'],
            hot_value_threshold=topic['threshold'],
            channels=topic['channels'],
            cooldown_minutes=30  # Avoid spam
        )

setup_alert_system()
```

## Advanced Configuration

### Using Huawei Pangu Model (Recommended for Chinese)

The project recommends Huawei's Pangu model for better Chinese language understanding:

```python
# Download model from:
# https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

# Configure in .env
USE_PANGU_MODEL=true
PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model

# In code
from hotsearch_analysis_agent.models import PanguAnalyzer

analyzer = PanguAnalyzer(model_path=os.getenv('PANGU_MODEL_PATH'))
result = analyzer.analyze_sentiment(text)
```

### Custom Spider Development

```python
# Create new spider in hotsearchcrawler/spiders/
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
                hot_value=item.css('.hot-score::text').get(),
                rank=item.css('.rank::text').get()
            )
```

## Troubleshooting

### Issue: Crawler fails with "WebDriver not found"

```bash
# Verify driver is in PATH
which chromedriver  # Linux/macOS
where chromedriver  # Windows

# If not found, add to PATH or specify in settings
# hotsearchcrawler/settings.py:
SELENIUM_DRIVER_PATH = '/path/to/chromedriver'
```

### Issue: Database connection errors

```python
# Test database connection
from hotsearch_analysis_agent.database import DatabaseManager

try:
    db = DatabaseManager()
    result = db.test_connection()
    print("Connection successful:", result)
except Exception as e:
    print(f"Connection failed: {e}")
    # Check .env file settings
    # Verify MySQL service is running
```

### Issue: LLM analysis returns empty results

```python
# Check API configuration
import os
print("API Key set:", bool(os.getenv('OPENAI_API_KEY')))
print("API Base:", os.getenv('OPENAI_API_BASE'))

# Test with simple query
from hotsearch_analysis_agent.analyzer import OpinionAnalyzer
analyzer = OpinionAnalyzer()
test_result = analyzer.test_connection()
print("LLM connection test:", test_result)
```

### Issue: Push notifications not sending

```python
# Test individual channels
from test_push_task import test_push

# Test WeChat
test_push(topic="test", channels=["wechat"], test_mode=True)

# Test Telegram
test_push(topic="test", channels=["telegram"], test_mode=True)

# Check webhook URLs and tokens in .env
# Verify network connectivity to push services
```

### Issue: Memory usage too high during clustering

```python
# Reduce batch size for large datasets
from hotsearch_analysis_agent.analyzer import OpinionAnalyzer

analyzer = OpinionAnalyzer()
items = db.get_recent_items(limit=1000)

# Process in batches
batch_size = 100
clusters = []
for i in range(0, len(items), batch_size):
    batch = items[i:i+batch_size]
    batch_clusters = analyzer.cluster_topics(batch, num_clusters=3)
    clusters.extend(batch_clusters)
```

## API Reference

### Core Classes

```python
# DatabaseManager
db = DatabaseManager()
db.get_recent_items(platform=None, limit=100, date_from=None)
db.search_items(keywords=[], platforms=[], date_from=None, date_to=None)
db.get_item_detail(item_id)
db.save_analysis_result(item_id, analysis_data)

# OpinionAnalyzer
analyzer = OpinionAnalyzer(model_name="gpt-4")
analyzer.cluster_topics(items, num_clusters=5, method='auto')
analyzer.analyze_sentiment(items, detail_level='full')
analyzer.generate_report(clusters, topic, time_range)
analyzer.summarize_cluster(cluster)

# PushTaskManager
push_mgr = PushTaskManager()
push_mgr.create_task(name, query_keywords, schedule, channels)
push_mgr.execute_task(task_id)
push_mgr.send_notification(content, channels, format='markdown')
```
