---
name: llm-public-opinion-analytics-assistant
description: Deploy and use the LLM-Based Intelligent Public Opinion Analytics Assistant for real-time multi-platform hotspot analysis, clustering, sentiment analysis, and automated alerts
triggers:
  - set up the public opinion analytics assistant
  - analyze hot topics from Chinese social platforms
  - configure multi-platform crawler for trending topics
  - implement sentiment analysis with LLM for news
  - set up automated hotspot push notifications
  - query and cluster trending topics across platforms
  - deploy opinion monitoring system with crawler
  - analyze Weibo Douyin Bilibili trending data
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is an intelligent public opinion analytics assistant that combines real-time data from **26 trending lists across 15 mainstream Chinese platforms** with LLM analysis capabilities. It enables conversational hotspot queries, topic clustering, sentiment analysis, multi-platform data crawling, and multi-channel alert pushing (email, WeChat, Telegram).

## What This Project Does

- **Real-time Data Collection**: Crawls trending topics from 15+ platforms (Weibo, Douyin, Bilibili, Baidu, Zhihu, etc.)
- **LLM-Powered Analysis**: Uses large language models (supports Huawei Pangu, OpenAI-compatible APIs) for topic clustering and sentiment analysis
- **Conversational Interface**: Natural language queries for hotspot data and analysis
- **Detailed Content Extraction**: Fetches full article/video content for deep analysis
- **Multi-Channel Alerts**: Pushes analysis reports via Enterprise WeChat, Telegram, Email
- **Web Dashboard**: Frontend interface for data exploration and crawler control

## Installation & Setup

### Prerequisites

1. **Python 3.8+** with virtual environment
2. **MySQL Database** (5.7+ or 8.0+)
3. **Browser Driver** (ChromeDriver or EdgeDriver for content scraping)

### Browser Driver Setup

```bash
# Check your Chrome/Edge version first
google-chrome --version  # or microsoft-edge --version

# Download matching driver version from:
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Place driver in PATH (example for Linux/macOS)
sudo mv chromedriver /usr/local/bin/
sudo chmod +x /usr/local/bin/chromedriver

# Verify installation
chromedriver --version
```

### Project Installation

```bash
# Clone repository
git clone https://github.com/hmmnxkl/LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant.git
cd LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Database Configuration

```python
# Reference init.py for database schema
# Create database and tables in MySQL

import pymysql

connection = pymysql.connect(
    host='localhost',
    user='your_mysql_user',
    password='your_mysql_password',
    charset='utf8mb4'
)

with connection.cursor() as cursor:
    cursor.execute("CREATE DATABASE IF NOT EXISTS hotsearch CHARACTER SET utf8mb4")
    # Run table creation SQL from init.py
    
connection.close()
```

### Environment Configuration

Create `.env` file in project root:

```env
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_mysql_user
MYSQL_PASSWORD=your_mysql_password
MYSQL_DATABASE=hotsearch

# LLM API Configuration (OpenAI-compatible)
OPENAI_API_KEY=your_api_key_here
OPENAI_API_BASE=https://api.openai.com/v1
OPENAI_MODEL=gpt-4

# Or use Huawei Pangu Model (local deployment)
# PANGU_MODEL_PATH=/path/to/pangu/model

# Push Notification Configuration (optional)
# Enterprise WeChat Bot
WECHAT_BOT_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_telegram_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email SMTP
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
```

Crawler configuration in `hotsearchcrawler/settings.py`:

```python
# MySQL settings for crawler
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_mysql_user'
MYSQL_PASSWORD = 'your_mysql_password'
MYSQL_DATABASE = 'hotsearch'

# Optional: Platform-specific cookies for better access
COOKIES = {
    'weibo': 'your_weibo_cookies',
    'douyin': 'your_douyin_cookies'
}
```

## Running the System

### Start the Web Application

```bash
# Main application entry point
python app.py
```

Access the web interface at `http://localhost:5000`

### Running Crawlers

**Via Web Interface** (Recommended):
- Use keyboard shortcuts in the web UI to start/stop crawlers

**Manual Testing**:

```bash
# Test individual crawler
python runspider-test.py

# Run all crawlers
python run_spiders.py
```

### Test Push Notifications

```bash
# Test configured push channels
python test_push_task.py
```

## Key Components & Usage

### 1. Conversational Query Interface

```python
# Example: Query hot topics via natural language
# In the web interface, type queries like:

"Show me today's top trending topics on Weibo"
"What are people talking about AI on Bilibili?"
"Analyze sentiment around '华为' topic"
"Cluster related topics about technology"
```

### 2. Programmatic Data Access

```python
from hotsearch_analysis_agent.database import get_db_connection

# Query trending topics
def get_platform_trending(platform='weibo', limit=10):
    conn = get_db_connection()
    cursor = conn.cursor()
    
    query = """
        SELECT title, hot_value, url, created_at 
        FROM hot_topics 
        WHERE platform = %s 
        ORDER BY hot_value DESC, created_at DESC 
        LIMIT %s
    """
    
    cursor.execute(query, (platform, limit))
    results = cursor.fetchall()
    cursor.close()
    conn.close()
    
    return results

# Get topics
weibo_trends = get_platform_trending('weibo', 20)
for topic in weibo_trends:
    print(f"{topic[0]} - Heat: {topic[1]}")
```

### 3. LLM Analysis Integration

```python
from hotsearch_analysis_agent.llm_analyzer import analyze_topics, sentiment_analysis

# Topic clustering
topics = ["华为发布新手机", "华为AI芯片突破", "华为鸿蒙系统更新"]
clusters = analyze_topics(topics)

# Example output structure:
# {
#   'cluster_name': '华为科技进展',
#   'topics': [...],
#   'summary': 'LLM生成的聚类摘要',
#   'key_themes': ['手机', 'AI', '操作系统']
# }

# Sentiment analysis on news content
news_content = "华为发布最新AI芯片,性能提升显著..."
sentiment = sentiment_analysis(news_content)

# Returns: {'sentiment': 'positive', 'score': 0.85, 'explanation': '...'}
```

### 4. Crawler Control

```python
from hotsearchcrawler.run_spiders import start_all_spiders, stop_all_spiders

# Start all platform crawlers
start_all_spiders()

# Stop crawlers
stop_all_spiders()

# Run specific platform crawler
from scrapy.crawler import CrawlerProcess
from hotsearchcrawler.spiders.weibo_spider import WeiboSpider

process = CrawlerProcess()
process.crawl(WeiboSpider)
process.start()
```

### 5. Content Detail Extraction

```python
from hotsearch_analysis_agent.content_fetcher import fetch_article_content

# Fetch full content from news URL (including video content)
url = "https://www.bilibili.com/video/BV13pSoBBEvX"
content = fetch_article_content(url)

# Returns structured content:
# {
#   'title': '视频标题',
#   'content': '视频描述和字幕文本',
#   'author': '作者',
#   'publish_time': '2026-04-07',
#   'tags': ['AI', '科技']
# }
```

### 6. Automated Push Tasks

```python
from hotsearch_analysis_agent.push_service import create_push_task, send_report

# Create scheduled push task
task_config = {
    'name': '科技热点日报',
    'keywords': ['AI', '人工智能', '芯片'],
    'platforms': ['weibo', 'zhihu', 'bilibili'],
    'channels': ['wechat', 'telegram', 'email'],
    'schedule': 'daily',
    'time': '09:00'
}

create_push_task(task_config)

# Manual push of analysis report
report_data = {
    'title': '关于人工智能的热点分析',
    'summary': '...',
    'topics': [...],
    'analysis': '...'
}

send_report(
    report_data, 
    channels=['wechat', 'email'],
    recipients=['admin@example.com']
)
```

## Common Patterns

### Pattern 1: Daily Trending Report

```python
from datetime import datetime, timedelta
from hotsearch_analysis_agent.database import get_db_connection
from hotsearch_analysis_agent.llm_analyzer import generate_report

def daily_trending_report():
    """Generate comprehensive daily trending report"""
    
    conn = get_db_connection()
    cursor = conn.cursor()
    
    # Get last 24 hours data
    yesterday = datetime.now() - timedelta(days=1)
    
    query = """
        SELECT platform, title, hot_value, url, content 
        FROM hot_topics 
        WHERE created_at >= %s 
        ORDER BY hot_value DESC 
        LIMIT 100
    """
    
    cursor.execute(query, (yesterday,))
    topics = cursor.fetchall()
    
    # Use LLM to analyze and cluster
    report = generate_report(topics, analysis_type='comprehensive')
    
    cursor.close()
    conn.close()
    
    return report
```

### Pattern 2: Keyword Monitoring

```python
def monitor_keywords(keywords, platforms=['weibo', 'douyin', 'zhihu']):
    """Monitor specific keywords across platforms"""
    
    conn = get_db_connection()
    cursor = conn.cursor()
    
    results = {}
    
    for keyword in keywords:
        query = """
            SELECT platform, title, hot_value, url, created_at
            FROM hot_topics
            WHERE title LIKE %s AND platform IN %s
            ORDER BY created_at DESC
            LIMIT 50
        """
        
        cursor.execute(query, (f'%{keyword}%', platforms))
        results[keyword] = cursor.fetchall()
    
    cursor.close()
    conn.close()
    
    return results

# Usage
tech_topics = monitor_keywords(
    ['华为', 'AI芯片', 'DeepSeek'],
    platforms=['weibo', 'zhihu', 'bilibili']
)
```

### Pattern 3: Sentiment Trend Analysis

```python
from hotsearch_analysis_agent.llm_analyzer import batch_sentiment_analysis

def analyze_sentiment_trend(topic_keyword, days=7):
    """Analyze sentiment trend for a topic over time"""
    
    conn = get_db_connection()
    cursor = conn.cursor()
    
    query = """
        SELECT DATE(created_at) as date, title, content
        FROM hot_topics
        WHERE title LIKE %s AND created_at >= DATE_SUB(NOW(), INTERVAL %s DAY)
        ORDER BY created_at
    """
    
    cursor.execute(query, (f'%{topic_keyword}%', days))
    rows = cursor.fetchall()
    
    # Group by date
    by_date = {}
    for row in rows:
        date, title, content = row
        if date not in by_date:
            by_date[date] = []
        by_date[date].append({'title': title, 'content': content})
    
    # Analyze sentiment per day
    trend = {}
    for date, items in by_date.items():
        sentiments = batch_sentiment_analysis([item['content'] for item in items])
        avg_score = sum(s['score'] for s in sentiments) / len(sentiments)
        trend[date] = {
            'average_sentiment': avg_score,
            'count': len(items),
            'items': items
        }
    
    cursor.close()
    conn.close()
    
    return trend
```

## Troubleshooting

### Issue: Crawler Fails to Extract Data

**Problem**: Spider returns empty results or errors

**Solutions**:

```python
# 1. Check if cookies are needed (in hotsearchcrawler/settings.py)
COOKIES = {
    'weibo': 'your_valid_cookies_here'
}

# 2. Verify browser driver is accessible
import subprocess
result = subprocess.run(['chromedriver', '--version'], capture_output=True)
print(result.stdout.decode())

# 3. Enable verbose logging in spider
import logging
logging.basicConfig(level=logging.DEBUG)

# 4. Test with individual spider
from hotsearchcrawler.spiders.weibo_spider import WeiboSpider
spider = WeiboSpider()
# Add debug breakpoints in parse methods
```

### Issue: LLM Analysis Returns Poor Results

**Problem**: Topic clustering or sentiment analysis is inaccurate

**Solutions**:

```python
# 1. Check if model supports Chinese text
# For OpenAI models, ensure using gpt-4 or gpt-3.5-turbo
OPENAI_MODEL = 'gpt-4'

# 2. Adjust prompt templates in hotsearch_analysis_agent/llm_analyzer.py
CLUSTER_PROMPT = """
你是一个专业的中文舆情分析助手。请分析以下热点话题,识别主题并分类。
确保理解中文语境和网络用语。

话题列表:
{topics}

请返回JSON格式的聚类结果。
"""

# 3. Increase context length for long content
MAX_TOKENS = 4000  # Adjust based on model limits
```

### Issue: Database Connection Errors

**Problem**: MySQL connection timeouts or authentication failures

**Solutions**:

```python
# 1. Test connection manually
import pymysql
try:
    conn = pymysql.connect(
        host='localhost',
        user='your_user',
        password='your_password',
        database='hotsearch',
        charset='utf8mb4',
        connect_timeout=10
    )
    print("Connection successful")
    conn.close()
except Exception as e:
    print(f"Connection failed: {e}")

# 2. Check MySQL user permissions
# In MySQL console:
# GRANT ALL PRIVILEGES ON hotsearch.* TO 'your_user'@'localhost';
# FLUSH PRIVILEGES;

# 3. Increase connection pool size in settings
MYSQL_POOL_SIZE = 10
MYSQL_MAX_OVERFLOW = 20
```

### Issue: Push Notifications Not Sending

**Problem**: WeChat/Telegram/Email alerts fail silently

**Solutions**:

```python
# Test each channel individually

# 1. Test Enterprise WeChat
import requests
webhook = 'https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY'
data = {
    "msgtype": "text",
    "text": {"content": "Test message"}
}
response = requests.post(webhook, json=data)
print(response.json())

# 2. Test Telegram
from telegram import Bot
bot = Bot(token='YOUR_TOKEN')
bot.send_message(chat_id='YOUR_CHAT_ID', text='Test')

# 3. Test Email
import smtplib
from email.mime.text import MIMEText

msg = MIMEText('Test')
msg['Subject'] = 'Test'
msg['From'] = 'your_email@gmail.com'
msg['To'] = 'recipient@example.com'

with smtplib.SMTP('smtp.gmail.com', 587) as server:
    server.starttls()
    server.login('your_email@gmail.com', 'your_app_password')
    server.send_message(msg)
```

### Issue: High Memory Usage

**Problem**: Application consumes excessive memory during crawling

**Solutions**:

```python
# 1. Limit concurrent requests in hotsearchcrawler/settings.py
CONCURRENT_REQUESTS = 8  # Reduce from default 16
CONCURRENT_REQUESTS_PER_DOMAIN = 4

# 2. Enable item pipelines to write data incrementally
ITEM_PIPELINES = {
    'hotsearchcrawler.pipelines.MysqlPipeline': 300,
}

# 3. Add memory limits in crawler process
import resource
resource.setrlimit(resource.RLIMIT_AS, (2 * 1024 ** 3, -1))  # 2GB limit
```

## Platform Support

Currently supports 15 platforms with 26 trending lists:

- **Social Media**: Weibo, Douyin, Kuaishou
- **Video**: Bilibili, Xigua Video
- **Q&A**: Zhihu, Baidu Zhidao
- **News**: Toutiao, Baidu News, Sina News
- **E-commerce**: JD.com, Taobao
- **Search**: Baidu Hot Search
- **Tech**: IT Home, 36Kr, iHeima

Each platform has dedicated spiders in `hotsearchcrawler/spiders/`.
