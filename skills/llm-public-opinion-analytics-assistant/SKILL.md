---
name: llm-public-opinion-analytics-assistant
description: Chinese public opinion analysis assistant with 26 hot search lists from 15 platforms, LLM-powered clustering, sentiment analysis, and multi-channel alerts
triggers:
  - set up public opinion monitoring in Chinese
  - analyze hot topics from Weibo and Bilibili
  - create sentiment analysis for Chinese social media
  - configure multi-platform hot search crawler
  - send alerts to WeChat or Telegram for trending topics
  - cluster and analyze Chinese news topics with LLM
  - monitor public sentiment across Chinese platforms
  - deploy real-time hot search analysis system
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is a comprehensive Chinese public opinion monitoring system that:
- Crawls 26 hot search lists from 15 major Chinese platforms (Weibo, Bilibili, Douyin, Zhihu, etc.)
- Uses LLM (Pangu/OpenAI-compatible models) for topic clustering and sentiment analysis
- Provides conversational query interface for hot topics
- Extracts content from news detail pages (including video transcripts)
- Sends multi-channel alerts (email, WeChat, Enterprise WeChat, Telegram)
- Supports hotkey-controlled crawler management

## Installation

### Prerequisites

**1. Browser Driver Setup**

Download driver matching your browser version:
- Chrome: https://chromedriver.chromium.org/
- Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

Place driver in system PATH or browser installation directory.

Verify installation:
```bash
chromedriver --version
```

**2. Database Setup**

Install MySQL and create database:
```sql
CREATE DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

Refer to `init.py` for table schemas.

**3. Python Environment**

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

## Configuration

### Environment Variables (`.env`)

```bash
# Database
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
OPENAI_MODEL=gpt-4

# Or use Huawei Pangu Model
# PANGU_API_KEY=your_pangu_key
# PANGU_API_BASE=https://pangu-api-endpoint

# Push Notifications
# Email
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
SMTP_FROM=your_email@gmail.com

# Enterprise WeChat Bot
WECOM_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=your_key

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

### Crawler Configuration (`hotsearchcrawler/settings.py`)

```python
# MySQL connection
MYSQL_CONFIG = {
    'host': os.getenv('MYSQL_HOST', 'localhost'),
    'port': int(os.getenv('MYSQL_PORT', 3306)),
    'user': os.getenv('MYSQL_USER', 'root'),
    'password': os.getenv('MYSQL_PASSWORD'),
    'database': os.getenv('MYSQL_DATABASE', 'hotsearch'),
    'charset': 'utf8mb4'
}

# Optional: Platform-specific cookies
COOKIES = {
    'weibo': 'your_weibo_cookie',  # Optional for better access
    'bilibili': 'your_bilibili_cookie'
}
```

## Running the System

### Start Analysis System

```bash
# Main application (web interface + API)
python app.py
```

Access at `http://localhost:5000`

### Start Crawlers

```bash
# Run all crawlers
python run_spiders.py

# Or test specific crawler
python runspider-test.py weibo
```

Available crawlers: `weibo`, `bilibili`, `douyin`, `zhihu`, `toutiao`, `baidu`, `36kr`, `ithome`, etc.

## Usage Examples

### Conversational Hot Topic Query

```python
from hotsearch_analysis_agent.agent import HotSearchAgent

agent = HotSearchAgent()

# Query hot topics
response = agent.query("今天微博上有什么热搜?")
# Returns: Current Weibo hot search list with rankings

# Topic search
response = agent.query("搜索关于人工智能的新闻")
# Returns: AI-related news from all platforms

# Sentiment analysis
response = agent.query("分析最近关于电动车的舆情")
# Returns: Sentiment trends and analysis
```

### Programmatic Data Access

```python
import mysql.connector
import os

# Connect to database
conn = mysql.connector.connect(
    host=os.getenv('MYSQL_HOST'),
    user=os.getenv('MYSQL_USER'),
    password=os.getenv('MYSQL_PASSWORD'),
    database=os.getenv('MYSQL_DATABASE')
)

cursor = conn.cursor(dictionary=True)

# Fetch latest hot topics from Weibo
cursor.execute("""
    SELECT title, hot_value, url, created_at 
    FROM hotsearch 
    WHERE platform = 'weibo' 
    ORDER BY created_at DESC 
    LIMIT 10
""")

topics = cursor.fetchall()
for topic in topics:
    print(f"{topic['title']} - {topic['hot_value']}")
```

### Topic Clustering Analysis

```python
from hotsearch_analysis_agent.cluster import TopicCluster

cluster = TopicCluster()

# Analyze topics from last 24 hours
keywords = ["人工智能", "科技"]
clusters = cluster.analyze(keywords=keywords, hours=24)

for cluster_group in clusters:
    print(f"Cluster: {cluster_group['theme']}")
    print(f"Sentiment: {cluster_group['sentiment']}")
    print(f"Articles: {len(cluster_group['articles'])}")
```

### Setting Up Push Notifications

```python
from hotsearch_analysis_agent.push import PushTaskManager

push_manager = PushTaskManager()

# Create email alert task
push_manager.create_task(
    name="AI News Daily",
    keywords=["人工智能", "ChatGPT", "大模型"],
    channels=["email"],
    recipients=["user@example.com"],
    schedule="0 9 * * *"  # Daily at 9 AM
)

# Create WeChat Enterprise alert
push_manager.create_task(
    name="Tech Hot Topics",
    keywords=["科技", "芯片"],
    channels=["wecom"],
    threshold=5000,  # Alert when hot_value > 5000
    schedule="*/30 * * * *"  # Every 30 minutes
)
```

### Web Scraping Detail Pages

```python
from hotsearchcrawler.spiders.detail_spider import DetailSpider

spider = DetailSpider()

# Extract content from news URL (including video transcripts)
content = spider.extract_content(
    url="https://www.bilibili.com/video/BV1234567890",
    platform="bilibili"
)

print(content['title'])
print(content['text'])  # Text or transcript
print(content['media_type'])  # 'video', 'article', etc.
```

## Testing Push Tasks

```bash
# Test all configured push channels
python test_push_task.py

# Test specific channel
python test_push_task.py --channel email
python test_push_task.py --channel wecom
python test_push_task.py --channel telegram
```

## Common Patterns

### Daily Public Opinion Report

```python
from datetime import datetime, timedelta
from hotsearch_analysis_agent.report import ReportGenerator

generator = ReportGenerator()

# Generate report for specific topics
report = generator.generate(
    title="AI & Tech Daily Report",
    keywords=["人工智能", "前沿科技", "芯片"],
    start_time=datetime.now() - timedelta(days=1),
    end_time=datetime.now(),
    platforms=["weibo", "bilibili", "36kr", "ithome"]
)

# Report includes:
# - Core findings
# - Detailed news items
# - Sentiment analysis
# - Information spread characteristics
```

### Real-time Monitoring Dashboard

```python
from flask import Flask, jsonify
from hotsearch_analysis_agent.monitor import RealTimeMonitor

app = Flask(__name__)
monitor = RealTimeMonitor()

@app.route('/api/trending')
def get_trending():
    # Get trending topics across all platforms
    trending = monitor.get_trending(limit=50)
    return jsonify(trending)

@app.route('/api/sentiment/<keyword>')
def get_sentiment(keyword):
    # Get sentiment for specific keyword
    sentiment = monitor.analyze_sentiment(keyword)
    return jsonify({
        'keyword': keyword,
        'positive': sentiment['positive'],
        'negative': sentiment['negative'],
        'neutral': sentiment['neutral']
    })
```

### Custom Crawler for New Platform

```python
# hotsearchcrawler/spiders/custom_platform.py
import scrapy
from ..items import HotSearchItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://custom-platform.com/hot']
    
    def parse(self, response):
        for item in response.css('.hot-item'):
            yield HotSearchItem(
                platform='custom_platform',
                title=item.css('.title::text').get(),
                hot_value=item.css('.hot-value::text').get(),
                url=item.css('a::attr(href)').get(),
                rank=item.css('.rank::text').get()
            )
```

## Troubleshooting

### Browser Driver Issues

**Error: "chromedriver executable needs to be in PATH"**

```bash
# Linux/Mac: Add to ~/.bashrc or ~/.zshrc
export PATH=$PATH:/path/to/driver/directory

# Windows: Add driver directory to system PATH via Environment Variables
```

**Error: "session not created: This version of ChromeDriver only supports Chrome version X"**

- Check browser version: `chrome://version` or `edge://version`
- Download matching driver version
- Update driver in PATH location

### Database Connection Issues

```python
# Test connection
import mysql.connector
try:
    conn = mysql.connector.connect(
        host=os.getenv('MYSQL_HOST'),
        user=os.getenv('MYSQL_USER'),
        password=os.getenv('MYSQL_PASSWORD')
    )
    print("Database connected successfully")
except Exception as e:
    print(f"Connection failed: {e}")
```

### LLM API Issues

**Check API connectivity:**

```python
import openai
import os

openai.api_key = os.getenv('OPENAI_API_KEY')
openai.api_base = os.getenv('OPENAI_API_BASE')

try:
    response = openai.ChatCompletion.create(
        model=os.getenv('OPENAI_MODEL'),
        messages=[{"role": "user", "content": "测试"}],
        max_tokens=10
    )
    print("LLM API working")
except Exception as e:
    print(f"API error: {e}")
```

### Crawler Failures

**Enable debug logging:**

```python
# hotsearchcrawler/settings.py
LOG_LEVEL = 'DEBUG'
LOG_FILE = 'crawler_debug.log'
```

**Common fixes:**
- Update platform-specific CSS selectors if page structure changed
- Add appropriate delays: `DOWNLOAD_DELAY = 2`
- Use cookies for platforms requiring authentication
- Check for IP rate limiting and add user-agent rotation

### Push Notification Failures

**Test individual channels:**

```python
from hotsearch_analysis_agent.push.email_sender import EmailSender

sender = EmailSender()
try:
    sender.send(
        to=["test@example.com"],
        subject="Test",
        content="Testing email notification"
    )
    print("Email sent successfully")
except Exception as e:
    print(f"Email failed: {e}")
```

## Advanced Configuration

### Using Huawei Pangu Model (Local Deployment)

```python
# Download model from:
# https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

# Configure in .env
PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model
PANGU_DEVICE=cuda  # or 'cpu'

# Use in code
from hotsearch_analysis_agent.llm import PanguModel

model = PanguModel(
    model_path=os.getenv('PANGU_MODEL_PATH'),
    device=os.getenv('PANGU_DEVICE')
)

result = model.analyze_sentiment("这个产品很不错")
# Returns: {'sentiment': 'positive', 'confidence': 0.92}
```

### Scheduler Configuration

```python
# Set up automated crawler runs
from apscheduler.schedulers.background import BackgroundScheduler

scheduler = BackgroundScheduler()

# Run crawlers every 30 minutes
scheduler.add_job(
    func=run_all_crawlers,
    trigger='interval',
    minutes=30
)

scheduler.start()
```
