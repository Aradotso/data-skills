---
name: llm-public-opinion-analytics-assistant
description: Deploy and use this Chinese public opinion analytics system with multi-platform data crawling, LLM-powered analysis, and multi-channel alerting
triggers:
  - how do I set up the public opinion analytics assistant
  - configure multi-platform sentiment analysis crawler
  - analyze hot topics and trending news with LLM
  - set up automated opinion monitoring and alerts
  - deploy the Chinese social media sentiment analyzer
  - integrate Pangu LLM for opinion analysis
  - configure hot search crawlers for Weibo Bilibili
  - send sentiment reports via WeChat or Telegram
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is a comprehensive Chinese public opinion analytics system that combines real-time data from **15 major platforms** (26 trending lists total) with LLM analysis capabilities. It provides:

- **Multi-platform crawling**: Weibo, Bilibili, Douyin, Zhihu, Baidu, and 10+ other Chinese platforms
- **LLM-powered analysis**: Topic clustering, sentiment analysis, trend detection using Huawei Pangu or OpenAI-compatible models
- **Conversational interface**: Natural language queries for hot search data, topic analysis, and sentiment trends
- **Multi-channel alerts**: Push reports via Enterprise WeChat, Telegram, SMTP email
- **Video content extraction**: Analyzes video-based news content, not just text
- **Keyboard shortcuts**: Quick crawler control from the web interface

The project consists of two main components:
- **Analysis system** (`hotsearch_analysis_agent`): Flask web app with LLM integration
- **Crawler cluster** (`hotsearchcrawler`): Scrapy-based multi-platform data collection

## Installation

### Prerequisites

1. **Browser Driver Setup** (required for Selenium-based crawlers):

```bash
# Check your browser version first
# Chrome: Settings → About
# Edge: Settings → About

# Download matching driver:
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Linux/macOS - move to PATH
sudo mv chromedriver /usr/local/bin/
chmod +x /usr/local/bin/chromedriver

# Windows - add to System PATH
# Or place in browser installation directory
```

2. **Python Environment**:

```bash
# Clone the repository
git clone https://github.com/hmmnxkl/LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant.git
cd LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant

# Create virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

3. **MySQL Database**:

```bash
# Install MySQL 8.0+
# Create database and tables using init.py as reference

# Example schema setup
mysql -u root -p
CREATE DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE hotsearch;

# Run init.py to create tables
python init.py
```

### Configuration

#### 1. Crawler Configuration (`hotsearchcrawler/settings.py`)

```python
# MySQL connection
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_user'
MYSQL_PASSWORD = 'your_password'
MYSQL_DATABASE = 'hotsearch'

# Optional: Platform-specific cookies
# Some platforms require authentication
COOKIES = {
    'weibo': 'your_weibo_cookie',
    # Add others as needed
}
```

#### 2. Analysis System Configuration (`.env` file)

```bash
# MySQL connection
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_BASE_URL=https://api.openai.com/v1  # Or custom endpoint

# Or use Huawei Pangu model
PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model
PANGU_API_URL=http://localhost:8000  # If using model server

# Push notification channels
# Enterprise WeChat Robot
WECHAT_WEBHOOK_URL=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Enterprise WeChat App
WECHAT_CORP_ID=your_corp_id
WECHAT_AGENT_ID=your_agent_id
WECHAT_SECRET=your_secret

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email (SMTP)
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
EMAIL_RECIPIENTS=recipient1@example.com,recipient2@example.com
```

## Running the System

### Start the Analysis Web Interface

```bash
# Main application
python app.py

# Access at http://localhost:5000
# Default port can be changed in app.py
```

### Start Crawlers

```bash
# From web interface (recommended):
# Use keyboard shortcut or button to start/stop crawlers

# Manual start (all platforms):
python run_spiders.py

# Test individual spider:
cd hotsearchcrawler
scrapy crawl weibo_hot  # Weibo hot search
scrapy crawl bilibili_hot  # Bilibili trending
scrapy crawl douyin_hot  # Douyin hot videos
```

### Test Push Notifications

```bash
# Test configured push channels
python test_push_task.py

# This will send a test report to all configured channels
```

## Key Features and Usage

### 1. Conversational Queries

The system supports natural language queries through the web interface:

```python
# Example queries users can ask:
"今天微博热搜前10是什么？"  # Top 10 Weibo hot searches
"搜索关于人工智能的新闻"  # Search AI-related news
"分析最近科技类话题的情感倾向"  # Sentiment analysis for tech topics
"聚类分析今天的热点话题"  # Cluster today's hot topics
```

### 2. Data Collection and Storage

The crawler system stores data in MySQL with this schema:

```python
# Example: Accessing collected data
import pymysql

connection = pymysql.connect(
    host='localhost',
    user='your_user',
    password='your_password',
    database='hotsearch',
    charset='utf8mb4'
)

with connection.cursor() as cursor:
    # Get latest hot searches from Weibo
    sql = """
        SELECT title, hot_value, url, created_at 
        FROM weibo_hot 
        ORDER BY created_at DESC 
        LIMIT 10
    """
    cursor.execute(sql)
    results = cursor.fetchall()
    for row in results:
        print(f"{row[0]} - 热度: {row[1]}")
```

### 3. LLM-Powered Analysis

The system uses LLM for intelligent analysis:

```python
# hotsearch_analysis_agent/llm_service.py

from openai import OpenAI

class OpinionAnalyzer:
    def __init__(self):
        self.client = OpenAI(
            api_key=os.getenv('OPENAI_API_KEY'),
            base_url=os.getenv('OPENAI_BASE_URL')
        )
    
    def analyze_sentiment(self, news_items):
        """Analyze sentiment of news items"""
        prompt = f"""
        分析以下新闻的情感倾向(正面/负面/中性):
        {json.dumps(news_items, ensure_ascii=False)}
        
        返回JSON格式,包含每条新闻的sentiment和confidence。
        """
        
        response = self.client.chat.completions.create(
            model="gpt-4",  # Or pangu model endpoint
            messages=[
                {"role": "system", "content": "你是一个专业的舆情分析助手"},
                {"role": "user", "content": prompt}
            ],
            temperature=0.3
        )
        
        return response.choices[0].message.content
    
    def cluster_topics(self, news_items):
        """Cluster related topics"""
        prompt = f"""
        对以下新闻进行话题聚类分析:
        {json.dumps(news_items, ensure_ascii=False)}
        
        返回主要话题分类和每个话题下的新闻。
        """
        
        response = self.client.chat.completions.create(
            model="gpt-4",
            messages=[
                {"role": "system", "content": "你是一个专业的舆情分析助手"},
                {"role": "user", "content": prompt}
            ],
            temperature=0.3
        )
        
        return response.choices[0].message.content
```

### 4. Automated Push Notifications

Configure scheduled reports:

```python
# Example: Setting up a push task

import requests
import json
from datetime import datetime

class PushTaskManager:
    def __init__(self):
        self.wechat_webhook = os.getenv('WECHAT_WEBHOOK_URL')
        self.telegram_token = os.getenv('TELEGRAM_BOT_TOKEN')
        self.telegram_chat_id = os.getenv('TELEGRAM_CHAT_ID')
    
    def send_wechat_report(self, report_content):
        """Send report to Enterprise WeChat"""
        data = {
            "msgtype": "markdown",
            "markdown": {
                "content": report_content
            }
        }
        
        response = requests.post(
            self.wechat_webhook,
            json=data,
            headers={'Content-Type': 'application/json'}
        )
        
        return response.json()
    
    def send_telegram_report(self, report_content):
        """Send report to Telegram"""
        url = f"https://api.telegram.org/bot{self.telegram_token}/sendMessage"
        
        data = {
            "chat_id": self.telegram_chat_id,
            "text": report_content,
            "parse_mode": "Markdown"
        }
        
        response = requests.post(url, json=data)
        return response.json()
    
    def generate_and_send_report(self, query="人工智能与前沿科技"):
        """Generate LLM analysis and send via all channels"""
        # 1. Fetch relevant news from database
        news_items = self.fetch_news(query)
        
        # 2. Analyze with LLM
        analyzer = OpinionAnalyzer()
        analysis = analyzer.cluster_topics(news_items)
        sentiment = analyzer.analyze_sentiment(news_items)
        
        # 3. Format report
        report = self.format_report(analysis, sentiment)
        
        # 4. Send via all configured channels
        self.send_wechat_report(report)
        self.send_telegram_report(report)
        
        return report
```

### 5. Video Content Extraction

The system can extract information even from video-based news:

```python
# Example spider extracting video metadata

class BilibiliHotSpider(scrapy.Spider):
    name = 'bilibili_hot'
    
    def parse(self, response):
        # Extract video information
        videos = response.css('div.video-item')
        
        for video in videos:
            item = {
                'title': video.css('a.title::text').get(),
                'author': video.css('span.up-name::text').get(),
                'play_count': video.css('span.play::text').get(),
                'url': video.css('a::attr(href)').get(),
                'cover_url': video.css('img::attr(src)').get()
            }
            
            # Follow to detail page for more content
            yield scrapy.Request(
                item['url'],
                callback=self.parse_video_detail,
                meta={'item': item}
            )
    
    def parse_video_detail(self, response):
        """Extract video description, tags, comments"""
        item = response.meta['item']
        
        item['description'] = response.css('div.desc::text').get()
        item['tags'] = response.css('span.tag::text').getall()
        
        # Use Selenium for dynamic content if needed
        # to get video transcript or AI-generated summary
        
        yield item
```

## Common Patterns

### Pattern 1: Daily Automated Monitoring

```python
# Schedule daily reports for specific topics
from apscheduler.schedulers.background import BackgroundScheduler

scheduler = BackgroundScheduler()

def daily_tech_report():
    """Generate daily tech opinion report"""
    manager = PushTaskManager()
    report = manager.generate_and_send_report(query="科技创新")
    print(f"Daily report sent at {datetime.now()}")

# Run every day at 9 AM
scheduler.add_job(daily_tech_report, 'cron', hour=9, minute=0)
scheduler.start()
```

### Pattern 2: Real-time Alert for Keywords

```python
# Monitor specific keywords and alert immediately
import time

def monitor_keywords(keywords=['突发', '紧急', '重大']):
    """Monitor for critical keywords"""
    last_check = datetime.now()
    
    while True:
        # Check new entries since last check
        new_items = fetch_news_since(last_check)
        
        for item in new_items:
            if any(kw in item['title'] for kw in keywords):
                # Immediate alert
                alert_content = f"⚠️ 重要舆情提醒\n\n{item['title']}\n{item['url']}"
                send_immediate_alert(alert_content)
        
        last_check = datetime.now()
        time.sleep(300)  # Check every 5 minutes
```

### Pattern 3: Multi-Platform Aggregation

```python
# Aggregate same topic from multiple platforms
def aggregate_topic(topic_keyword):
    """Get topic coverage across all platforms"""
    platforms = ['weibo', 'bilibili', 'douyin', 'zhihu', 'baidu']
    results = {}
    
    for platform in platforms:
        query = f"SELECT * FROM {platform}_hot WHERE title LIKE %s ORDER BY created_at DESC LIMIT 5"
        results[platform] = execute_query(query, (f'%{topic_keyword}%',))
    
    # Analyze differences in coverage/sentiment across platforms
    return analyze_cross_platform(results)
```

## Troubleshooting

### Issue: Crawler fails with "Driver not found"

```bash
# Verify driver is in PATH
which chromedriver  # Linux/macOS
where chromedriver  # Windows

# Test driver manually
chromedriver --version

# If not found, add to PATH or specify in Scrapy settings
SELENIUM_DRIVER_EXECUTABLE_PATH = '/usr/local/bin/chromedriver'
```

### Issue: MySQL connection errors

```python
# Test connection independently
import pymysql

try:
    connection = pymysql.connect(
        host='localhost',
        user='your_user',
        password='your_password',
        database='hotsearch'
    )
    print("Connection successful")
    connection.close()
except Exception as e:
    print(f"Connection failed: {e}")
    # Check: MySQL running? Correct credentials? Database exists?
```

### Issue: LLM API rate limits

```python
# Add retry logic with exponential backoff
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=4, max=10))
def call_llm_api(prompt):
    response = client.chat.completions.create(
        model="gpt-4",
        messages=[{"role": "user", "content": prompt}]
    )
    return response.choices[0].message.content
```

### Issue: Some platforms return empty data

```bash
# Check if cookies are needed
# Update settings.py with valid cookies for authenticated platforms

# Test spider with verbose output
scrapy crawl weibo_hot -L DEBUG

# Check robots.txt compliance
# Add custom headers/delays in settings.py:
DOWNLOAD_DELAY = 2
USER_AGENT = 'Mozilla/5.0 (compatible; OpinionBot/1.0)'
```

### Issue: Push notifications not sending

```python
# Test each channel independently
python test_push_task.py

# Verify webhook URLs and tokens
# Check enterprise WeChat: IP whitelist configured?
# Check Telegram: Bot has permission to send to chat?
# Check SMTP: Less secure apps enabled? App password created?

# Enable debug logging
import logging
logging.basicConfig(level=logging.DEBUG)
```

## Advanced Configuration

### Using Huawei Pangu Model

```python
# Download model from:
# https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

# Configure in .env
PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model

# Update LLM service to use local model
from transformers import AutoModelForCausalLM, AutoTokenizer

class PanguAnalyzer:
    def __init__(self):
        self.tokenizer = AutoTokenizer.from_pretrained(
            os.getenv('PANGU_MODEL_PATH')
        )
        self.model = AutoModelForCausalLM.from_pretrained(
            os.getenv('PANGU_MODEL_PATH')
        )
    
    def generate(self, prompt):
        inputs = self.tokenizer(prompt, return_tensors="pt")
        outputs = self.model.generate(**inputs, max_length=2048)
        return self.tokenizer.decode(outputs[0])
```

### Custom Crawler Development

```python
# Add new platform spider in hotsearchcrawler/spiders/

import scrapy

class NewPlatformSpider(scrapy.Spider):
    name = 'newplatform_hot'
    start_urls = ['https://newplatform.com/trending']
    
    def parse(self, response):
        for item in response.css('div.trending-item'):
            yield {
                'platform': 'newplatform',
                'title': item.css('h3::text').get(),
                'url': item.css('a::attr(href)').get(),
                'hot_value': item.css('span.score::text').get(),
                'created_at': datetime.now()
            }
```
