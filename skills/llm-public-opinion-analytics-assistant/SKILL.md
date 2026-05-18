---
name: llm-public-opinion-analytics-assistant
description: Use this Chinese public opinion analytics system that aggregates 26 trending lists from 15 platforms with LLM analysis for sentiment analysis, topic clustering, and multi-channel alerts
triggers:
  - set up public opinion monitoring system
  - analyze social media trending topics in Chinese
  - deploy sentiment analysis crawler with LLM
  - configure hot topic alert system
  - implement Chinese social media analytics
  - create multi-platform trend monitoring
  - build opinion analysis dashboard
  - set up automated sentiment reporting
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is a Chinese public opinion analytics system that combines real-time data from 26 trending lists across 15 major platforms (Weibo, Bilibili, Douyin, etc.) with large language model analysis capabilities. It provides conversational hot search queries, topic clustering, sentiment analysis, and multi-channel push notifications (email, WeChat, Telegram).

## Installation

### Prerequisites

1. **Browser Driver Setup** (required for content scraping):

```bash
# Download ChromeDriver or EdgeDriver matching your browser version
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Linux/macOS - move to PATH
sudo mv chromedriver /usr/local/bin/
sudo chmod +x /usr/local/bin/chromedriver

# Verify installation
chromedriver --version
```

2. **Python Environment**:

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # Linux/macOS
# or
venv\Scripts\activate  # Windows

# Install dependencies
pip install -r requirements.txt
```

3. **MySQL Database**:

```bash
# Install MySQL and create database
mysql -u root -p

CREATE DATABASE opinion_analytics CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE opinion_analytics;

# Run initialization script
python init.py
```

### Configuration

1. **Environment Variables** (`.env`):

```env
# Database Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_username
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=opinion_analytics

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key_here
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Or use Huawei Pangu Model
PANGU_API_KEY=your_pangu_key
PANGU_MODEL_PATH=/path/to/pangu/model

# Push Notification Channels
# Email (SMTP)
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
NOTIFICATION_EMAIL=recipient@example.com

# WeChat Work Bot
WECHAT_WORK_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

2. **Crawler Settings** (`hotsearchcrawler/settings.py`):

```python
# MySQL connection for crawler
MYSQL_CONFIG = {
    'host': 'localhost',
    'user': 'your_username',
    'password': 'your_password',
    'database': 'opinion_analytics',
    'charset': 'utf8mb4'
}

# Optional: Platform-specific cookies for authenticated access
COOKIES = {
    'weibo': 'your_weibo_cookies',
    'bilibili': 'your_bilibili_cookies'
}

# Crawler intervals (seconds)
DOWNLOAD_DELAY = 3
CONCURRENT_REQUESTS = 8
```

## Starting the System

### Launch Web Application

```bash
# Start the main application server
python app.py

# Application runs on http://localhost:5000
```

### Launch Crawler Cluster

```bash
# Method 1: Start all spiders via web UI
# Navigate to http://localhost:5000 and use shortcut keys

# Method 2: Start manually
python run_spiders.py

# Method 3: Test individual spiders
python runspider-test.py --spider weibo
python runspider-test.py --spider bilibili
python runspider-test.py --spider douyin
```

## Core Usage Patterns

### 1. Query Trending Topics

```python
from hotsearch_analysis_agent.query_engine import QueryEngine

# Initialize query engine
engine = QueryEngine()

# Natural language query
result = engine.query("最近人工智能相关的热点有哪些?")
# Returns: List of AI-related trending topics with metadata

# Platform-specific query
result = engine.query("微博今日热搜前10", platform="weibo")

# Time-range query
result = engine.query("昨天到今天的科技新闻", 
                     start_date="2026-05-17",
                     end_date="2026-05-18")
```

### 2. Topic Clustering Analysis

```python
from hotsearch_analysis_agent.clustering import TopicClusterer

clusterer = TopicClusterer()

# Cluster topics by keyword
clusters = clusterer.cluster_by_keyword("人工智能", days=7)
# Returns: Grouped topics with similarity scores

# Automatic clustering of current hot topics
auto_clusters = clusterer.auto_cluster(min_topics=20, threshold=0.7)

# Result structure:
# {
#   'cluster_0': {
#     'keywords': ['GPT-6', '大模型', '上下文'],
#     'topics': [{'title': '...', 'platform': '...', 'heat': 12345}],
#     'summary': 'AI model context window breakthrough...'
#   }
# }
```

### 3. Sentiment Analysis

```python
from hotsearch_analysis_agent.sentiment import SentimentAnalyzer

analyzer = SentimentAnalyzer()

# Analyze single topic sentiment
sentiment = analyzer.analyze_topic("DeepSeek V4采用华为算力")
# Returns: {'score': 0.75, 'label': 'positive', 'confidence': 0.92}

# Batch sentiment analysis
topics = ["topic1", "topic2", "topic3"]
results = analyzer.batch_analyze(topics)

# Sentiment trend over time
trend = analyzer.sentiment_trend("人工智能", days=30)
# Returns time-series sentiment data
```

### 4. Content Detail Extraction

```python
from hotsearchcrawler.detail_scraper import DetailScraper

scraper = DetailScraper()

# Scrape article content (including video transcripts)
url = "https://www.bilibili.com/video/BV13pSoBBEvX"
content = scraper.scrape_detail(url)

# Result includes:
# {
#   'title': '...',
#   'content': '...',  # Text or video transcript
#   'author': '...',
#   'publish_time': '...',
#   'tags': [...],
#   'comments': [...]
# }
```

### 5. Automated Push Tasks

```python
from hotsearch_analysis_agent.push_service import PushService

push = PushService()

# Create monitoring task
task = push.create_task(
    name="AI热点监控",
    keywords=["人工智能", "大模型", "GPT"],
    platforms=["weibo", "zhihu", "36kr"],
    channels=["email", "wechat_work", "telegram"],
    frequency="daily",  # hourly, daily, weekly
    threshold_heat=10000,  # Minimum heat score
    sentiment_filter="positive"  # positive, negative, neutral, all
)

# Test push task
push.test_push(task_id=task['id'])

# Manual trigger
report = push.generate_report(
    query="人工智能与前沿科技",
    days=1
)
push.send_report(report, channels=["email", "telegram"])
```

### 6. Direct Database Queries

```python
import pymysql
import os

# Connect to database
conn = pymysql.connect(
    host=os.getenv('MYSQL_HOST'),
    user=os.getenv('MYSQL_USER'),
    password=os.getenv('MYSQL_PASSWORD'),
    database=os.getenv('MYSQL_DATABASE'),
    charset='utf8mb4'
)

cursor = conn.cursor(pymysql.cursors.DictCursor)

# Query hot topics
cursor.execute("""
    SELECT title, platform, heat_score, url, created_at
    FROM hot_topics
    WHERE created_at >= DATE_SUB(NOW(), INTERVAL 24 HOUR)
    ORDER BY heat_score DESC
    LIMIT 50
""")
topics = cursor.fetchall()

# Query by keyword
cursor.execute("""
    SELECT * FROM hot_topics
    WHERE title LIKE %s OR content LIKE %s
    ORDER BY created_at DESC
""", ('%人工智能%', '%人工智能%'))

cursor.close()
conn.close()
```

## Supported Platforms

The system monitors 26 trending lists from 15 platforms:

- **Social Media**: Weibo (微博), Douyin (抖音), Xiaohongshu (小红书)
- **Video**: Bilibili (B站), Kuaishou (快手)
- **News**: Toutiao (头条), 36Kr, IFeng (凤凰网), Sina News
- **Q&A**: Zhihu (知乎), Baidu Tieba (百度贴吧)
- **Tech**: CSDN, V2EX, Juejin (掘金)
- **Finance**: Xueqiu (雪球)

## Troubleshooting

### Crawler Issues

**Problem**: Spiders fail to start or timeout

```bash
# Check browser driver
chromedriver --version

# Test individual spider
python runspider-test.py --spider weibo --verbose

# Check crawler logs
tail -f hotsearchcrawler/logs/crawler.log
```

**Problem**: Anti-crawler detection

```python
# Add cookies in hotsearchcrawler/settings.py
COOKIES = {
    'weibo': 'your_cookie_string_here'
}

# Increase download delay
DOWNLOAD_DELAY = 5
RANDOMIZE_DOWNLOAD_DELAY = True
```

### LLM Analysis Issues

**Problem**: Analysis slow or fails

```python
# Switch to faster model
# In .env:
MODEL_NAME=gpt-3.5-turbo

# Or use local Pangu model
# Download from: https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model
PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model
```

**Problem**: Context length exceeded

```python
# Truncate content before analysis
from hotsearch_analysis_agent.utils import truncate_text

content = truncate_text(long_content, max_tokens=4000)
result = analyzer.analyze(content)
```

### Database Issues

**Problem**: Connection errors

```bash
# Verify MySQL is running
sudo systemctl status mysql

# Test connection
mysql -h localhost -u your_username -p opinion_analytics

# Check character set
SHOW VARIABLES LIKE 'character_set%';
# Should be utf8mb4 for Chinese support
```

**Problem**: Duplicate entries

```sql
-- Add unique constraint
ALTER TABLE hot_topics 
ADD UNIQUE KEY unique_topic (platform, title, DATE(created_at));
```

### Push Notification Issues

**Problem**: Email not sending

```python
# Test SMTP connection
import smtplib

try:
    server = smtplib.SMTP('smtp.gmail.com', 587)
    server.starttls()
    server.login('your_email@gmail.com', 'your_app_password')
    print("SMTP connection successful")
except Exception as e:
    print(f"SMTP error: {e}")
```

**Problem**: WeChat Work webhook fails

```bash
# Test webhook directly
curl -X POST 'https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY' \
  -H 'Content-Type: application/json' \
  -d '{"msgtype":"text","text":{"content":"Test message"}}'
```

## Advanced Configuration

### Custom Spider for New Platform

```python
# hotsearchcrawler/spiders/custom_spider.py
import scrapy
from hotsearchcrawler.items import HotTopicItem

class CustomSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://example.com/trending']
    
    def parse(self, response):
        for topic in response.css('.topic-item'):
            item = HotTopicItem()
            item['platform'] = 'custom_platform'
            item['title'] = topic.css('.title::text').get()
            item['url'] = topic.css('a::attr(href)').get()
            item['heat_score'] = topic.css('.heat::text').get()
            yield item
```

### Custom Analysis Prompt

```python
# hotsearch_analysis_agent/prompts.py
CUSTOM_ANALYSIS_PROMPT = """
你是一位专业的舆情分析专家。请分析以下话题:

话题: {topic}
内容: {content}
平台: {platform}

请提供:
1. 核心事件概要
2. 情感倾向(积极/消极/中性)
3. 关键利益相关方
4. 潜在影响范围
5. 发展趋势预测

输出JSON格式。
"""

# Use in analysis
from hotsearch_analysis_agent.analyzer import Analyzer

analyzer = Analyzer(custom_prompt=CUSTOM_ANALYSIS_PROMPT)
result = analyzer.analyze(topic_data)
```

This skill enables AI agents to help developers deploy and use a comprehensive Chinese public opinion monitoring system with LLM-powered analytics.
