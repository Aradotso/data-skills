---
name: llm-public-opinion-analytics-assistant
description: Deploy and use an LLM-powered multi-platform public opinion monitoring system with real-time crawling, sentiment analysis, and multi-channel alerting
triggers:
  - set up public opinion monitoring system
  - analyze trending topics across platforms
  - configure sentiment analysis for news
  - deploy multi-platform crawler for hot search
  - build topic clustering analysis tool
  - integrate LLM for opinion analytics
  - set up automated hotspot push notifications
  - monitor social media trends with AI
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is a comprehensive public opinion analytics system that monitors **26 trending lists** from **15 mainstream platforms** (Weibo, Bilibili, Baidu, Zhihu, etc.). It combines web crawling, LLM analysis (using Huawei Pangu model or OpenAI-compatible APIs), and multi-channel notifications to provide real-time sentiment analysis, topic clustering, and automated alerts via Email, WeChat Work, Telegram, and more.

**Key Capabilities:**
- Real-time multi-platform hot search data collection
- Conversational query interface for trending topics
- LLM-powered sentiment analysis and topic clustering
- Video content extraction and analysis
- Multi-channel push notifications (Email, WeChat, Telegram, Enterprise WeChat)
- Keyboard shortcut control for crawler management

## Installation

### Prerequisites

**1. Browser Driver Setup (Critical for Content Extraction)**

The system requires a browser driver to extract detailed content from news pages:

```bash
# Check your Chrome/Edge version first
# Chrome: chrome://settings/help
# Edge: edge://settings/help

# Download matching driver version:
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Linux/macOS - move to PATH
sudo mv chromedriver /usr/local/bin/
chmod +x /usr/local/bin/chromedriver

# Verify installation
chromedriver --version
```

**2. Database Setup**

```bash
# Install MySQL
# Create database and tables using init.py structure

# Example MySQL setup
mysql -u root -p
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE hotsearch_db;
# Run schema from init.py
```

**3. Python Environment**

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

**1. Environment Variables (`.env` file)**

```bash
# Database Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_username
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key_here
OPENAI_API_BASE=https://api.openai.com/v1  # Or Pangu model endpoint
LLM_MODEL=gpt-4  # Or pangu-embedded-7b

# Notification Channels
# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
NOTIFICATION_EMAIL=recipient@example.com

# WeChat Work Bot
WECHAT_WORK_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Enterprise WeChat App
ENTERPRISE_WECHAT_CORP_ID=your_corp_id
ENTERPRISE_WECHAT_AGENT_ID=your_agent_id
ENTERPRISE_WECHAT_SECRET=your_secret
```

**2. Crawler Configuration (`hotsearchcrawler/settings.py`)**

```python
# MySQL connection for crawler
MYSQL_CONFIG = {
    'host': os.getenv('MYSQL_HOST', 'localhost'),
    'port': int(os.getenv('MYSQL_PORT', 3306)),
    'user': os.getenv('MYSQL_USER'),
    'password': os.getenv('MYSQL_PASSWORD'),
    'database': os.getenv('MYSQL_DATABASE'),
    'charset': 'utf8mb4'
}

# Optional: Platform-specific cookies for authenticated crawling
COOKIES = {
    'weibo': 'your_weibo_cookie_if_needed',
    'bilibili': 'your_bilibili_cookie_if_needed'
}

# Crawler settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
ROBOTSTXT_OBEY = False
```

## Core Usage Patterns

### Starting the Web Application

```bash
# Launch the main web interface
python app.py

# Access at http://localhost:5000
# Default port can be configured in app.py
```

### Crawler Management

**Via Web Interface:**
- Press designated hotkey (configurable) to start/stop crawlers
- Monitor crawling status in real-time dashboard

**Via Command Line:**

```bash
# Test individual spider
python runspider-test.py

# Start all crawlers programmatically
python run_spiders.py

# The system will crawl these platforms:
# - Weibo (weibo hot search, weibo video)
# - Bilibili (hot, ranking)
# - Baidu (hot search)
# - Zhihu (hot list)
# - Douyin, Kuaishou, Toutiao, NetEase, Tencent, etc.
```

### Conversational Query Examples

```python
# The web interface supports natural language queries:

# Query specific platform
"Show me today's Weibo hot search"
"What's trending on Bilibili?"

# Topic-based search
"Find news about artificial intelligence"
"Search for topics related to climate change"

# Sentiment analysis
"Analyze sentiment for [topic name]"
"What's the public opinion on [event]?"

# Clustering analysis
"Cluster related topics about technology"
"Group similar trending topics"
```

### Programmatic Analysis API

```python
from hotsearch_analysis_agent.analyzer import OpinionAnalyzer
from hotsearch_analysis_agent.database import DatabaseManager

# Initialize analyzer with LLM config
analyzer = OpinionAnalyzer(
    api_key=os.getenv('OPENAI_API_KEY'),
    api_base=os.getenv('OPENAI_API_BASE'),
    model=os.getenv('LLM_MODEL', 'gpt-4')
)

# Get trending topics from database
db = DatabaseManager()
topics = db.query_topics(
    platform='weibo',
    start_date='2026-05-01',
    limit=50
)

# Perform sentiment analysis
for topic in topics:
    sentiment = analyzer.analyze_sentiment(topic['content'])
    print(f"Topic: {topic['title']}")
    print(f"Sentiment: {sentiment['label']} ({sentiment['score']:.2f})")
    print(f"Keywords: {', '.join(sentiment['keywords'])}\n")

# Topic clustering
clusters = analyzer.cluster_topics(topics, num_clusters=5)
for i, cluster in enumerate(clusters):
    print(f"Cluster {i+1}: {cluster['theme']}")
    print(f"Topics: {len(cluster['topics'])}")
    print(f"Summary: {cluster['summary']}\n")

# Generate comprehensive report
report = analyzer.generate_report(
    query="artificial intelligence and frontier technology",
    date_range=('2026-04-01', '2026-04-07')
)
print(report)
```

### Setting Up Push Notifications

```python
from hotsearch_analysis_agent.pusher import NotificationPusher

# Initialize pusher with multiple channels
pusher = NotificationPusher(config={
    'email': {
        'enabled': True,
        'smtp_host': os.getenv('SMTP_HOST'),
        'smtp_port': int(os.getenv('SMTP_PORT')),
        'username': os.getenv('SMTP_USER'),
        'password': os.getenv('SMTP_PASSWORD'),
        'recipients': [os.getenv('NOTIFICATION_EMAIL')]
    },
    'wechat_work': {
        'enabled': True,
        'webhook': os.getenv('WECHAT_WORK_WEBHOOK')
    },
    'telegram': {
        'enabled': True,
        'bot_token': os.getenv('TELEGRAM_BOT_TOKEN'),
        'chat_id': os.getenv('TELEGRAM_CHAT_ID')
    },
    'enterprise_wechat': {
        'enabled': True,
        'corp_id': os.getenv('ENTERPRISE_WECHAT_CORP_ID'),
        'agent_id': os.getenv('ENTERPRISE_WECHAT_AGENT_ID'),
        'secret': os.getenv('ENTERPRISE_WECHAT_SECRET')
    }
})

# Create scheduled push task
task = {
    'name': 'Daily AI Trends Report',
    'query': 'artificial intelligence',
    'schedule': 'daily',  # or 'hourly', 'weekly'
    'time': '09:00',
    'channels': ['email', 'telegram'],
    'threshold': {
        'min_topics': 3,
        'min_heat': 1000  # Minimum heat score to trigger
    }
}

pusher.create_task(task)

# Manual push
report_content = analyzer.generate_report(query="AI trends")
pusher.send_notification(
    content=report_content,
    channels=['email', 'wechat_work'],
    title="AI Trends Analysis Report - 2026-05-17"
)
```

### Testing Push Functionality

```bash
# Test all configured notification channels
python test_push_task.py

# This will send test messages to:
# - Email (if configured)
# - WeChat Work webhook
# - Telegram bot
# - Enterprise WeChat app
```

## Advanced Features

### Video Content Analysis

The system can extract and analyze content from video platforms like Bilibili:

```python
from hotsearch_analysis_agent.video_extractor import VideoContentExtractor

extractor = VideoContentExtractor()

# Extract video metadata and transcripts
video_data = extractor.extract_content(
    url='https://www.bilibili.com/video/BV13pSoBBEvX'
)

print(f"Title: {video_data['title']}")
print(f"Views: {video_data['views']}")
print(f"Transcript: {video_data['transcript'][:200]}...")

# Analyze video content with LLM
analysis = analyzer.analyze_video_content(video_data)
print(f"Summary: {analysis['summary']}")
print(f"Key Points: {', '.join(analysis['key_points'])}")
```

### Custom Platform Integration

```python
# Add custom spider in hotsearchcrawler/spiders/

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
                heat=int(item.css('.heat::text').get()),
                rank=int(item.css('.rank::text').get()),
                timestamp=datetime.now()
            )
```

### Database Query Patterns

```python
from hotsearch_analysis_agent.database import DatabaseManager

db = DatabaseManager()

# Query by platform and date range
topics = db.query("""
    SELECT * FROM hot_topics 
    WHERE platform = %s 
    AND created_at BETWEEN %s AND %s
    ORDER BY heat DESC
    LIMIT %s
""", ('weibo', '2026-05-01', '2026-05-17', 100))

# Aggregate heat trends
trend_data = db.query("""
    SELECT DATE(created_at) as date, 
           platform,
           AVG(heat) as avg_heat,
           COUNT(*) as topic_count
    FROM hot_topics
    WHERE created_at >= DATE_SUB(NOW(), INTERVAL 7 DAY)
    GROUP BY date, platform
    ORDER BY date DESC
""")

# Search by keyword
keyword_results = db.search_topics(
    keyword='人工智能',
    platforms=['weibo', 'zhihu', 'bilibili'],
    min_heat=5000
)
```

## Troubleshooting

### ChromeDriver Issues

```bash
# Error: chromedriver not found or version mismatch
# Solution: Verify driver version matches browser
chromedriver --version
google-chrome --version  # or microsoft-edge --version

# Error: Permission denied
chmod +x /usr/local/bin/chromedriver

# Error: Driver crashes or timeouts
# Add headless options in crawler settings:
CHROME_OPTIONS = [
    '--headless',
    '--no-sandbox',
    '--disable-dev-shm-usage',
    '--disable-gpu'
]
```

### Database Connection Issues

```python
# Test database connection
from hotsearch_analysis_agent.database import DatabaseManager

try:
    db = DatabaseManager()
    db.test_connection()
    print("Database connection successful")
except Exception as e:
    print(f"Connection failed: {e}")
    # Check .env file credentials
    # Verify MySQL service is running
    # Check firewall settings
```

### LLM API Rate Limits

```python
# Implement retry logic with exponential backoff
import time
from functools import wraps

def retry_with_backoff(max_retries=3, base_delay=1):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_retries - 1:
                        raise
                    delay = base_delay * (2 ** attempt)
                    print(f"Retry {attempt+1}/{max_retries} after {delay}s")
                    time.sleep(delay)
        return wrapper
    return decorator

@retry_with_backoff(max_retries=5, base_delay=2)
def call_llm_api(prompt):
    return analyzer.generate_response(prompt)
```

### Crawler Blocking Issues

```python
# Rotate user agents in settings.py
USER_AGENT_LIST = [
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
    'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36',
    # Add more user agents
]

# Add random delays
DOWNLOAD_DELAY = 2
RANDOMIZE_DOWNLOAD_DELAY = True

# Use proxy middleware if needed
DOWNLOADER_MIDDLEWARES = {
    'scrapy.downloadermiddlewares.httpproxy.HttpProxyMiddleware': 110,
}
```

### Notification Delivery Failures

```bash
# Test individual channels
python -c "
from hotsearch_analysis_agent.pusher import NotificationPusher
pusher = NotificationPusher()
result = pusher.test_channel('email')
print(f'Email test: {result}')
"

# Check logs for SMTP errors
# Verify firewall allows outbound SMTP (port 587)
# For Gmail: enable "Less secure app access" or use App Password
# For WeChat/Telegram: verify webhook URLs and tokens
```

## Production Deployment

```bash
# Use gunicorn for production WSGI server
pip install gunicorn

# Run with multiple workers
gunicorn -w 4 -b 0.0.0.0:8000 app:app

# Set up systemd service (Linux)
sudo nano /etc/systemd/system/hotsearch.service

# Add:
[Unit]
Description=Hot Search Analysis Service
After=network.target

[Service]
User=youruser
WorkingDirectory=/path/to/project
Environment="PATH=/path/to/venv/bin"
ExecStart=/path/to/venv/bin/gunicorn -w 4 -b 0.0.0.0:8000 app:app

[Install]
WantedBy=multi-user.target

# Enable and start
sudo systemctl enable hotsearch
sudo systemctl start hotsearch

# Schedule crawler with cron
crontab -e
# Add: Run crawler every hour
0 * * * * /path/to/venv/bin/python /path/to/project/run_spiders.py
```
