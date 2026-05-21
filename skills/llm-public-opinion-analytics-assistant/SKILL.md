---
name: llm-public-opinion-analytics-assistant
description: Master the LLM-Based Intelligent Public Opinion Analytics Assistant - a Python system for real-time hot search data crawling, sentiment analysis, topic clustering, and multi-channel alert push from 15+ platforms
triggers:
  - how do I set up the public opinion analytics assistant
  - crawl hot search data from weibo bilibili and douyin
  - analyze sentiment and cluster topics with llm
  - configure multi-channel push notifications for trending topics
  - deploy the hotsearch crawler and analysis system
  - integrate pangu llm model for chinese sentiment analysis
  - create scheduled tasks for hot topic monitoring
  - query and filter trending news across platforms
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The LLM-Based Intelligent Public Opinion Analytics Assistant is a comprehensive Python-based system that monitors real-time trending topics from 15 major Chinese platforms (26 different ranking lists), performs LLM-powered sentiment analysis and topic clustering, and pushes alerts through multiple channels (email, WeChat, Enterprise WeChat, Telegram). The system separates crawling (Scrapy-based) from analysis, supports video content extraction, and provides a conversational web interface for querying trends.

**Key capabilities:**
- Real-time data collection from Weibo, Bilibili, Douyin, Zhihu, etc.
- Natural language querying of hot topics
- LLM-powered sentiment analysis and topic clustering
- Multi-channel alert push (email, WeChat Work, Telegram)
- Keyboard shortcuts for crawler control
- Video content information extraction

## Installation

### Prerequisites

**1. Browser Driver Setup**

Download and configure ChromeDriver or EdgeDriver:

```bash
# Check your browser version first (e.g., Chrome 115.x)
# Download matching driver from:
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Add driver to system PATH
# Linux/macOS:
sudo mv chromedriver /usr/local/bin/
sudo chmod +x /usr/local/bin/chromedriver

# Verify installation
chromedriver --version
```

**2. MySQL Database**

```bash
# Install MySQL 5.7+ or 8.0+
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
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Database Initialization

Reference `init.py` to create required tables:

```python
import pymysql

# Connection configuration
connection = pymysql.connect(
    host='localhost',
    user='root',
    password='your_password',
    database='hotsearch_db',
    charset='utf8mb4'
)

# Create tables (simplified example)
CREATE_TABLES = """
CREATE TABLE IF NOT EXISTS hot_search_items (
    id INT AUTO_INCREMENT PRIMARY KEY,
    platform VARCHAR(50),
    title VARCHAR(500),
    url VARCHAR(1000),
    hot_value VARCHAR(100),
    rank_position INT,
    collected_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_platform (platform),
    INDEX idx_collected (collected_at)
);

CREATE TABLE IF NOT EXISTS analysis_results (
    id INT AUTO_INCREMENT PRIMARY KEY,
    query TEXT,
    result LONGTEXT,
    sentiment VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS push_tasks (
    id INT AUTO_INCREMENT PRIMARY KEY,
    task_name VARCHAR(200),
    query_keywords TEXT,
    schedule_cron VARCHAR(100),
    push_channels JSON,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
"""

with connection.cursor() as cursor:
    for statement in CREATE_TABLES.split(';'):
        if statement.strip():
            cursor.execute(statement)
connection.commit()
```

## Configuration

### Environment Variables

Create `.env` file in project root:

```bash
# Database Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
LLM_API_BASE=http://localhost:8000/v1  # For local Pangu model
LLM_API_KEY=your_api_key
LLM_MODEL=pangu-embedded-7b

# Push Notification Channels
# Email (SMTP)
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
SMTP_RECEIVER=recipient@example.com

# Enterprise WeChat Robot
WEWORK_ROBOT_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Enterprise WeChat App (push to personal WeChat)
WEWORK_CORP_ID=your_corp_id
WEWORK_AGENT_ID=your_agent_id
WEWORK_SECRET=your_secret

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Browser Driver Path (if not in PATH)
CHROME_DRIVER_PATH=/usr/local/bin/chromedriver
```

### Crawler Settings

Edit `hotsearchcrawler/settings.py`:

```python
# MySQL connection for crawler
MYSQL_CONFIG = {
    'host': 'localhost',
    'port': 3306,
    'user': 'root',
    'password': 'your_password',
    'database': 'hotsearch_db'
}

# Scrapy settings
BOT_NAME = 'hotsearchcrawler'
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1  # Respect rate limits
COOKIES_ENABLED = True

# Optional: Platform-specific cookies for authenticated access
PLATFORM_COOKIES = {
    'weibo': {
        'SUB': 'your_cookie_value',
        # Add other cookies if needed
    }
}
```

### LLM Model Configuration

For **Huawei Pangu model** (recommended):

```bash
# Download model from:
# https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

# Start local inference server (example with vLLM)
python -m vllm.entrypoints.openai.api_server \
    --model /path/to/openpangu-embedded-7b-model \
    --host 0.0.0.0 \
    --port 8000
```

## Usage

### Starting the System

**1. Launch Analysis System (Web UI)**

```bash
# From project root
python app.py

# Server starts at http://localhost:5000
# Access web interface to query trends and configure tasks
```

**2. Run Crawlers**

```bash
# Test individual crawler
cd hotsearchcrawler
scrapy crawl weibo_spider

# Run all crawlers (can be triggered from web UI with keyboard shortcut)
python run_spiders.py
```

### Querying Hot Topics

**Via Web Interface (Natural Language)**

```
User: "Show me trending topics about AI in the last 24 hours"
User: "What's hot on Bilibili today?"
User: "Analyze sentiment for news about new energy vehicles"
User: "Cluster topics related to technology policy"
```

**Programmatic Query Example**

```python
from hotsearch_analysis_agent.query_engine import QueryEngine
from hotsearch_analysis_agent.llm_client import LLMClient

# Initialize
llm_client = LLMClient(
    api_base=os.getenv('LLM_API_BASE'),
    api_key=os.getenv('LLM_API_KEY'),
    model=os.getenv('LLM_MODEL')
)
query_engine = QueryEngine(llm_client=llm_client)

# Execute query
result = query_engine.query(
    user_input="人工智能与前沿科技的热点",
    platforms=["weibo", "bilibili", "zhihu"],
    time_range_hours=24
)

print(result['summary'])
print(result['sentiment_analysis'])
print(result['clustered_topics'])
```

### Setting Up Push Tasks

**Create Scheduled Push Task**

```python
from hotsearch_analysis_agent.push_manager import PushTaskManager
from datetime import datetime

push_manager = PushTaskManager()

# Define task
task_config = {
    'task_name': '每日AI热点推送',
    'query_keywords': ['人工智能', '大模型', 'AI技术'],
    'platforms': ['weibo', 'zhihu', 'sina'],
    'schedule_cron': '0 12 * * *',  # Daily at 12:00 PM
    'push_channels': {
        'email': True,
        'wework_robot': True,
        'telegram': False
    },
    'min_hot_value': 500000,  # Only push if hot value > 500k
    'sentiment_filter': None  # None = all, 'positive', 'negative', 'neutral'
}

# Create task
task_id = push_manager.create_task(task_config)

# Activate task
push_manager.activate_task(task_id)
```

**Test Push Manually**

```bash
# From project root
python test_push_task.py --task_id 1
```

**Push Task Implementation**

```python
import requests
import json
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
import smtplib

class PushService:
    def __init__(self):
        self.smtp_config = {
            'server': os.getenv('SMTP_SERVER'),
            'port': int(os.getenv('SMTP_PORT')),
            'user': os.getenv('SMTP_USER'),
            'password': os.getenv('SMTP_PASSWORD')
        }
    
    def push_email(self, subject: str, content: str, receiver: str):
        """Send email notification"""
        msg = MIMEMultipart('alternative')
        msg['Subject'] = subject
        msg['From'] = self.smtp_config['user']
        msg['To'] = receiver
        
        html_content = MIMEText(content, 'html', 'utf-8')
        msg.attach(html_content)
        
        with smtplib.SMTP(self.smtp_config['server'], self.smtp_config['port']) as server:
            server.starttls()
            server.login(self.smtp_config['user'], self.smtp_config['password'])
            server.send_message(msg)
    
    def push_wework_robot(self, content: str):
        """Push to Enterprise WeChat group robot"""
        webhook = os.getenv('WEWORK_ROBOT_WEBHOOK')
        
        payload = {
            'msgtype': 'markdown',
            'markdown': {
                'content': content
            }
        }
        
        response = requests.post(webhook, json=payload)
        return response.json()
    
    def push_telegram(self, message: str, parse_mode='Markdown'):
        """Send Telegram notification"""
        bot_token = os.getenv('TELEGRAM_BOT_TOKEN')
        chat_id = os.getenv('TELEGRAM_CHAT_ID')
        
        url = f'https://api.telegram.org/bot{bot_token}/sendMessage'
        payload = {
            'chat_id': chat_id,
            'text': message,
            'parse_mode': parse_mode
        }
        
        response = requests.post(url, json=payload)
        return response.json()
```

### Sentiment Analysis & Clustering

**Analyze Sentiment for Topics**

```python
from hotsearch_analysis_agent.sentiment_analyzer import SentimentAnalyzer

analyzer = SentimentAnalyzer(llm_client)

# Analyze single item
sentiment = analyzer.analyze(
    text="DeepSeek V4采用华为算力,国产芯片生态走到哪一步了?",
    context="tech_news"
)
# Returns: {'sentiment': 'positive', 'confidence': 0.85, 'keywords': ['国产', '芯片', '生态']}

# Batch analysis
items = [
    {'title': 'GPT-6遭提前曝光, 2M超长上下文来了', 'platform': 'weibo'},
    {'title': '内存太贵?厂商将复产DDR3主板', 'platform': 'bilibili'}
]

results = analyzer.batch_analyze(items)
```

**Topic Clustering**

```python
from hotsearch_analysis_agent.topic_clusterer import TopicClusterer

clusterer = TopicClusterer(llm_client)

# Get hot search items from multiple platforms
hot_items = query_engine.fetch_recent_items(
    platforms=['weibo', 'zhihu', 'bilibili'],
    hours=24,
    min_hot_value=100000
)

# Cluster related topics
clusters = clusterer.cluster_topics(hot_items, num_clusters=5)

for cluster in clusters:
    print(f"Cluster: {cluster['label']}")
    print(f"Items: {len(cluster['items'])}")
    print(f"Summary: {cluster['summary']}")
    print("---")
```

### Video Content Extraction

The system can extract information from video-based news:

```python
from hotsearchcrawler.video_extractor import VideoExtractor

extractor = VideoExtractor()

# Extract from Bilibili video
video_info = extractor.extract_bilibili(
    url="https://www.bilibili.com/video/BV13pSoBBEvX/"
)

print(video_info['title'])
print(video_info['description'])
print(video_info['tags'])
print(video_info['transcript'])  # If available
```

## Common Patterns

### Daily Monitoring Workflow

```python
# 1. Schedule crawler runs every hour
# 2. Aggregate and deduplicate data
# 3. Run clustering analysis every 6 hours
# 4. Push daily summary report at 9 AM

from apscheduler.schedulers.background import BackgroundScheduler

scheduler = BackgroundScheduler()

def crawl_job():
    os.system('python run_spiders.py')

def analysis_job():
    results = query_engine.query("今日热点话题", time_range_hours=6)
    # Store results

def daily_report_job():
    report = query_engine.generate_report(
        query="今日综合热点分析",
        time_range_hours=24
    )
    push_service.push_email(
        subject="每日舆情报告",
        content=report,
        receiver=os.getenv('SMTP_RECEIVER')
    )

scheduler.add_job(crawl_job, 'interval', hours=1)
scheduler.add_job(analysis_job, 'interval', hours=6)
scheduler.add_job(daily_report_job, 'cron', hour=9, minute=0)

scheduler.start()
```

### Custom Platform Integration

```python
# Add new crawler in hotsearchcrawler/spiders/

import scrapy
from hotsearchcrawler.items import HotSearchItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    
    def start_requests(self):
        url = 'https://customplatform.com/hot'
        yield scrapy.Request(url, callback=self.parse)
    
    def parse(self, response):
        for item_selector in response.css('.hot-item'):
            item = HotSearchItem()
            item['platform'] = 'custom_platform'
            item['title'] = item_selector.css('.title::text').get()
            item['url'] = item_selector.css('a::attr(href)').get()
            item['hot_value'] = item_selector.css('.hot-value::text').get()
            item['rank_position'] = item_selector.css('.rank::text').get()
            
            yield item
```

### Alert Threshold Configuration

```python
# Configure multi-level alerts based on hot value and sentiment

alert_rules = {
    'critical': {
        'hot_value_threshold': 10000000,
        'sentiment': 'negative',
        'channels': ['email', 'wework_robot', 'telegram']
    },
    'warning': {
        'hot_value_threshold': 5000000,
        'sentiment': 'negative',
        'channels': ['wework_robot']
    },
    'info': {
        'hot_value_threshold': 1000000,
        'sentiment': None,
        'channels': ['email']
    }
}

def check_and_alert(item):
    for level, rule in alert_rules.items():
        if (item['hot_value'] >= rule['hot_value_threshold'] and
            (rule['sentiment'] is None or item['sentiment'] == rule['sentiment'])):
            
            message = f"[{level.upper()}] {item['title']} - {item['platform']}"
            
            for channel in rule['channels']:
                if channel == 'email':
                    push_service.push_email("Hot Topic Alert", message, receiver)
                elif channel == 'wework_robot':
                    push_service.push_wework_robot(message)
                elif channel == 'telegram':
                    push_service.push_telegram(message)
            
            break  # Only trigger highest priority level
```

## Troubleshooting

### ChromeDriver Issues

```bash
# Error: Message: 'chromedriver' executable needs to be in PATH

# Solution 1: Add to PATH
export PATH=$PATH:/path/to/driver/directory

# Solution 2: Set explicit path in settings
# In .env:
CHROME_DRIVER_PATH=/usr/local/bin/chromedriver
```

### MySQL Connection Errors

```python
# Error: Can't connect to MySQL server

# Check connection parameters
import pymysql
try:
    connection = pymysql.connect(
        host=os.getenv('MYSQL_HOST'),
        port=int(os.getenv('MYSQL_PORT')),
        user=os.getenv('MYSQL_USER'),
        password=os.getenv('MYSQL_PASSWORD'),
        database=os.getenv('MYSQL_DATABASE')
    )
    print("Connection successful")
except Exception as e:
    print(f"Connection failed: {e}")
```

### LLM API Connection Issues

```python
# Test LLM connectivity
import requests

response = requests.post(
    f"{os.getenv('LLM_API_BASE')}/chat/completions",
    headers={'Authorization': f"Bearer {os.getenv('LLM_API_KEY')}"},
    json={
        'model': os.getenv('LLM_MODEL'),
        'messages': [{'role': 'user', 'content': 'test'}]
    }
)

print(response.status_code)
print(response.json())
```

### Crawler Rate Limiting

```python
# If encountering 429 or IP blocks:

# 1. Increase delay in settings.py
DOWNLOAD_DELAY = 3  # Increase from 1 to 3 seconds

# 2. Use rotating proxies (in settings.py)
ROTATING_PROXY_LIST = [
    'http://proxy1.com:8080',
    'http://proxy2.com:8080'
]
DOWNLOADER_MIDDLEWARES = {
    'rotating_proxies.middlewares.RotatingProxyMiddleware': 610,
    'rotating_proxies.middlewares.BanDetectionMiddleware': 620,
}

# 3. Add User-Agent rotation
USER_AGENT_LIST = [
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64)...',
    'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)...'
]
```

### Push Notification Failures

```python
# Verify channel configurations

# Email test
python -c "
from hotsearch_analysis_agent.push_manager import PushService
ps = PushService()
ps.push_email('Test', 'This is a test', 'recipient@example.com')
print('Email sent successfully')
"

# WeChat Work Robot test
curl -X POST "YOUR_WEWORK_WEBHOOK_URL" \
  -H "Content-Type: application/json" \
  -d '{"msgtype":"text","text":{"content":"Test message"}}'

# Telegram test
curl -X POST "https://api.telegram.org/botYOUR_BOT_TOKEN/sendMessage" \
  -d "chat_id=YOUR_CHAT_ID&text=Test"
```

### Memory Issues with Large Datasets

```python
# Use batch processing for large queries

def query_large_dataset(platforms, days=7):
    results = []
    
    # Process day by day
    for day in range(days):
        daily_results = query_engine.query(
            platforms=platforms,
            time_range_hours=24,
            offset_hours=day * 24
        )
        results.extend(daily_results)
        
        # Clear cache periodically
        if day % 3 == 0:
            query_engine.clear_cache()
    
    return results
```

This skill provides comprehensive guidance for deploying and using the LLM-Based Intelligent Public Opinion Analytics Assistant for monitoring, analyzing, and responding to trending topics across Chinese social media platforms.
