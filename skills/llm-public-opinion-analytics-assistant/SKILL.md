---
name: llm-public-opinion-analytics-assistant
description: Multi-platform hot search crawler and LLM-powered sentiment analysis system with push notifications
triggers:
  - how do I set up the public opinion analytics assistant
  - crawl hot search data from multiple platforms
  - analyze sentiment and cluster topics with LLM
  - configure push notifications for trending topics
  - set up the hotsearch crawler system
  - integrate Pangu model for sentiment analysis
  - how to query hot topics with natural language
  - deploy the opinion monitoring system
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is a comprehensive public opinion monitoring system that combines real-time data crawling from 15 mainstream platforms (26 ranking lists) with large language model analysis capabilities. It provides conversational hot search queries, topic clustering, sentiment analysis, and multi-channel push notifications (email, WeChat, Enterprise WeChat, Telegram).

## Architecture Overview

The project consists of two main components:

1. **Crawler Cluster** (`hotsearchcrawler/`) - Scrapy-based distributed crawlers for data collection
2. **Analysis System** (`hotsearch_analysis_agent/`) - LLM-powered analysis engine with web interface

## Installation

### Prerequisites

**Browser Driver Setup (Critical):**

```bash
# For Chrome - download ChromeDriver matching your Chrome version
# https://chromedriver.chromium.org/
# Place in system PATH or project directory

# For Edge - download EdgeDriver
# https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Verify installation
chromedriver --version
# or
msedgedriver --version
```

**Python Environment:**

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

**Database Setup:**

```bash
# Install MySQL and create database
mysql -u root -p

CREATE DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

## Configuration

### Environment Variables (`.env`)

```bash
# MySQL Configuration
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=your_password
DB_NAME=hotsearch

# OpenAI-Compatible API (for LLM analysis)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Or use Huawei Pangu Model (local deployment)
PANGU_MODEL_PATH=/path/to/pangu-model

# Push Notification Channels
# Enterprise WeChat Bot
WECHAT_WEBHOOK_URL=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=your_key

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
SMTP_TO=recipient@example.com
```

### Crawler Configuration (`hotsearchcrawler/settings.py`)

```python
# MySQL Connection for Crawler
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'root'
MYSQL_PASSWORD = 'your_password'
MYSQL_DB = 'hotsearch'

# Optional: Platform-specific cookies
COOKIES = {
    'weibo': 'your_weibo_cookie',
    'bilibili': 'your_bilibili_cookie',
    # Add as needed
}

# Crawler Settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
ROBOTSTXT_OBEY = False
```

## Running the System

### Initialize Database

```python
# Reference: init.py
from hotsearch_analysis_agent.database import init_db

# Create tables
init_db()
```

### Start Crawler Cluster

```bash
# Test individual crawler
cd hotsearchcrawler
scrapy crawl weibo_hot  # Test Weibo crawler

# Run all crawlers (via web interface or command)
python run_spiders.py
```

### Start Analysis Web Interface

```bash
# Launch Flask application
python app.py

# Access at http://localhost:5000
```

## Core Features & Usage

### 1. Conversational Hot Search Query

```python
# Example API call from frontend
import requests

response = requests.post('http://localhost:5000/api/query', json={
    'query': '查询今天科技类热搜',
    'platform': 'all'  # or specific: 'weibo', 'zhihu', etc.
})

results = response.json()
# Returns: list of hot topics with metadata
```

### 2. Topic Clustering Analysis

```python
from hotsearch_analysis_agent.analysis import TopicCluster

cluster = TopicCluster()

# Analyze related topics
topics = cluster.analyze(
    query="人工智能",
    start_date="2026-05-01",
    end_date="2026-05-19"
)

# Returns clustered topics with sentiment scores
for topic_group in topics:
    print(f"Cluster: {topic_group['theme']}")
    print(f"Articles: {topic_group['articles']}")
    print(f"Sentiment: {topic_group['sentiment']}")
```

### 3. Sentiment Analysis with LLM

```python
from hotsearch_analysis_agent.llm import SentimentAnalyzer

analyzer = SentimentAnalyzer(
    model_name=os.getenv('MODEL_NAME'),
    api_key=os.getenv('OPENAI_API_KEY')
)

# Analyze news article
result = analyzer.analyze_sentiment(
    title="GPT-6遭提前曝光, 2M超长上下文来了",
    content="详细新闻内容...",
    url="https://example.com/news/123"
)

print(f"Sentiment: {result['sentiment']}")  # positive/negative/neutral
print(f"Score: {result['score']}")  # -1.0 to 1.0
print(f"Keywords: {result['keywords']}")
```

### 4. Push Notification Task

```python
from hotsearch_analysis_agent.push import PushManager

push_manager = PushManager()

# Create scheduled push task
task = push_manager.create_task(
    name="AI Tech Daily Report",
    query="人工智能与前沿科技",
    channels=['wechat', 'telegram', 'email'],
    schedule="0 12 * * *",  # Daily at 12:00 PM
    filters={
        'min_heat': 1000,
        'sentiment': 'all',
        'platforms': ['weibo', 'zhihu', 'bilibili']
    }
)

# Manual push
push_manager.push_now(task_id=task['id'])
```

### 5. Crawler Control via Shortcuts

```python
# In web interface, use keyboard shortcuts:
# Ctrl+S: Start all crawlers
# Ctrl+E: Stop all crawlers
# Ctrl+R: Restart crawlers

# Programmatic control
from hotsearchcrawler.control import CrawlerManager

manager = CrawlerManager()
manager.start_all()  # Start all configured crawlers
manager.stop_spider('weibo_hot')  # Stop specific crawler
manager.get_status()  # Check crawler status
```

## Database Schema

```python
# Key tables structure
from sqlalchemy import Column, Integer, String, Text, DateTime
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class HotTopic(Base):
    __tablename__ = 'hot_topics'
    
    id = Column(Integer, primary_key=True)
    platform = Column(String(50))  # weibo, zhihu, bilibili, etc.
    title = Column(String(500))
    url = Column(String(1000))
    heat = Column(Integer)  # Heat index/rank
    content = Column(Text)  # Extracted content
    crawl_time = Column(DateTime)
    sentiment = Column(String(20))  # positive/negative/neutral
    keywords = Column(Text)  # JSON array

class PushTask(Base):
    __tablename__ = 'push_tasks'
    
    id = Column(Integer, primary_key=True)
    name = Column(String(200))
    query = Column(String(500))
    channels = Column(Text)  # JSON array
    schedule = Column(String(100))  # Cron expression
    filters = Column(Text)  # JSON object
    status = Column(String(20))  # active/paused
```

## Common Patterns

### Custom Crawler for New Platform

```python
# hotsearchcrawler/spiders/custom_spider.py
import scrapy
from hotsearchcrawler.items import HotTopicItem

class CustomSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://example.com/trending']
    
    def parse(self, response):
        for item in response.css('.trending-item'):
            topic = HotTopicItem()
            topic['platform'] = 'custom_platform'
            topic['title'] = item.css('.title::text').get()
            topic['url'] = item.css('a::attr(href)').get()
            topic['heat'] = int(item.css('.heat::text').get())
            
            # Follow link to get full content
            yield response.follow(
                topic['url'],
                callback=self.parse_detail,
                meta={'item': topic}
            )
    
    def parse_detail(self, response):
        item = response.meta['item']
        item['content'] = response.css('.article-content').get()
        yield item
```

### Custom Analysis Prompt

```python
# hotsearch_analysis_agent/prompts.py
CUSTOM_ANALYSIS_PROMPT = """
分析以下舆情数据,重点关注:
1. 主要观点和态度
2. 潜在风险或机会
3. 传播趋势预测

数据:
{data}

请以JSON格式返回分析结果。
"""

from hotsearch_analysis_agent.llm import analyze_with_prompt

results = analyze_with_prompt(
    data=topic_data,
    prompt=CUSTOM_ANALYSIS_PROMPT
)
```

### Video Content Extraction

```python
# For platforms with video content (Bilibili, etc.)
from hotsearch_analysis_agent.extractors import VideoExtractor

extractor = VideoExtractor()

video_info = extractor.extract(
    url="https://www.bilibili.com/video/BV13pSoBBEvX",
    use_subtitle=True,  # Extract subtitle text
    use_comments=True   # Include top comments
)

print(f"Title: {video_info['title']}")
print(f"Text: {video_info['transcript']}")
print(f"Comments: {video_info['comments']}")
```

## Troubleshooting

**Crawler fails with "ChromeDriver not found":**
```bash
# Ensure ChromeDriver is in PATH
export PATH=$PATH:/path/to/chromedriver

# Or specify in settings.py
SELENIUM_DRIVER_PATH = '/path/to/chromedriver'
```

**Database connection error:**
```python
# Test connection
import pymysql
conn = pymysql.connect(
    host='localhost',
    user='root',
    password='your_password',
    database='hotsearch'
)
print("Connected successfully")
```

**LLM API timeout:**
```python
# Increase timeout in settings
OPENAI_TIMEOUT = 60  # seconds

# Or use local Pangu model
USE_LOCAL_MODEL = True
PANGU_MODEL_PATH = '/path/to/pangu-embedded-7b'
```

**Push notification not working:**
```bash
# Test individual channel
python test_push_task.py --channel wechat --test

# Check webhook URL validity
curl -X POST $WECHAT_WEBHOOK_URL \
  -H 'Content-Type: application/json' \
  -d '{"msgtype": "text", "text": {"content": "Test"}}'
```

**Memory issues with large crawls:**
```python
# Adjust Scrapy settings
CONCURRENT_REQUESTS = 8  # Reduce from default 16
DOWNLOAD_DELAY = 2  # Increase delay
AUTOTHROTTLE_ENABLED = True
```

## Advanced: Pangu Model Integration

```python
# hotsearch_analysis_agent/models/pangu.py
from transformers import AutoModelForCausalLM, AutoTokenizer

class PanguAnalyzer:
    def __init__(self, model_path):
        self.tokenizer = AutoTokenizer.from_pretrained(model_path)
        self.model = AutoModelForCausalLM.from_pretrained(
            model_path,
            device_map="auto",
            torch_dtype="auto"
        )
    
    def analyze(self, text, task="sentiment"):
        prompt = f"分析以下文本的{task}:\n{text}\n\n分析:"
        inputs = self.tokenizer(prompt, return_tensors="pt")
        outputs = self.model.generate(**inputs, max_length=512)
        return self.tokenizer.decode(outputs[0])

# Usage
analyzer = PanguAnalyzer(model_path=os.getenv('PANGU_MODEL_PATH'))
result = analyzer.analyze("新闻内容...", task="情感倾向")
```

This skill enables AI agents to help developers deploy, configure, and extend the public opinion analytics system for real-time monitoring and LLM-powered analysis of trending topics across Chinese social media platforms.
