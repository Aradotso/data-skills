---
name: llm-public-opinion-analytics-assistant
description: Chinese public opinion analytics system with multi-platform hot search crawler, LLM-based sentiment analysis, and multi-channel alert push (WeChat, Telegram, Email)
triggers:
  - analyze public opinion trends from Chinese social media
  - crawl hot search rankings from Weibo Zhihu Bilibili
  - set up sentiment analysis for trending topics
  - configure multi-platform hot search monitoring
  - send opinion alerts via WeChat or Telegram
  - analyze news sentiment with LLM
  - cluster trending topics across platforms
  - deploy opinion monitoring system
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is an intelligent public opinion monitoring system that crawls hot search rankings from **15 major Chinese platforms** (26 ranking lists total), analyzes sentiment and trends using large language models, and sends multi-channel alerts. It supports conversational queries, topic clustering, sentiment analysis, and automated push notifications via WeChat Work, Telegram, and email.

## What It Does

- **Multi-Platform Data Collection**: Scrapes hot search rankings from Weibo, Zhihu, Bilibili, Douyin, Baidu, and 10+ other platforms
- **LLM-Powered Analysis**: Sentiment analysis, topic clustering, and trend identification using OpenAI-compatible LLM APIs
- **Conversational Interface**: Natural language queries for trending topics and analytics
- **Multi-Channel Alerts**: Push notifications via WeChat Work Robot/App, Telegram, and email (SMTP)
- **Video Content Analysis**: Extracts text from video-based news for comprehensive analysis
- **Web UI**: Frontend dashboard for crawler control, data visualization, and query management

## Installation

### Prerequisites

**1. Browser Driver Setup**

The crawler requires Edge or Chrome WebDriver for dynamic content:

```bash
# Check your browser version first
google-chrome --version  # or microsoft-edge --version

# Download matching driver version:
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/microsoft-edge/tools/webdriver/

# Add driver to PATH (Linux/macOS example)
sudo mv chromedriver /usr/local/bin/
sudo chmod +x /usr/local/bin/chromedriver

# Verify installation
chromedriver --version
```

**2. MySQL Database**

```bash
# Install MySQL 5.7+ or 8.0+
# Create database and tables using init.py as reference
mysql -u root -p
CREATE DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
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

### Configuration

**1. Database Configuration**

Edit `hotsearchcrawler/settings.py`:

```python
# MySQL connection settings
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_db_user'
MYSQL_PASSWORD = 'your_db_password'
MYSQL_DATABASE = 'hotsearch'
```

**2. LLM Configuration**

Create `.env` file in project root:

```bash
# OpenAI-compatible API (supports Pangu, DeepSeek, etc.)
OPENAI_API_KEY=your_api_key_here
OPENAI_BASE_URL=https://api.openai.com/v1  # or custom endpoint

# MySQL credentials
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_db_user
MYSQL_PASSWORD=your_db_password
MYSQL_DATABASE=hotsearch
```

**3. Push Notification Configuration**

Add to `.env` for desired channels:

```bash
# WeChat Work Robot
WECHAT_WORK_ROBOT_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# WeChat Work App
WECHAT_WORK_CORP_ID=your_corp_id
WECHAT_WORK_AGENT_ID=your_agent_id
WECHAT_WORK_SECRET=your_secret

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
EMAIL_RECIPIENTS=recipient1@example.com,recipient2@example.com
```

## Key Commands

### Start the System

```bash
# Launch main application (web UI + API)
python app.py

# Access web interface at http://localhost:5000
```

### Manual Crawler Operations

```bash
# Test single spider
python runspider-test.py

# Run all spiders (normally triggered from web UI)
python run_spiders.py
```

### Test Push Notifications

```bash
# Test configured push channels
python test_push_task.py
```

## Core API Usage

### Crawler Module

The crawler system uses Scrapy framework. Example spider structure:

```python
# hotsearchcrawler/spiders/weibo_spider.py
import scrapy
from hotsearchcrawler.items import HotSearchItem

class WeiboSpider(scrapy.Spider):
    name = 'weibo'
    allowed_domains = ['s.weibo.com']
    start_urls = ['https://s.weibo.com/top/summary']
    
    def parse(self, response):
        for rank, item in enumerate(response.css('td.td-02 a'), start=1):
            hot_search = HotSearchItem()
            hot_search['platform'] = 'weibo'
            hot_search['rank'] = rank
            hot_search['title'] = item.css('::text').get().strip()
            hot_search['url'] = response.urljoin(item.css('::attr(href)').get())
            hot_search['heat'] = item.css('span::text').get()
            yield hot_search
```

### Analysis Module

Query hot search data with LLM analysis:

```python
from hotsearch_analysis_agent.analyzer import OpinionAnalyzer

# Initialize analyzer
analyzer = OpinionAnalyzer()

# Query trending topics
result = analyzer.query_trends(
    platform='weibo',
    topic_keywords='人工智能',
    time_range='24h'
)

# Perform sentiment analysis
sentiment = analyzer.analyze_sentiment(
    news_ids=[1001, 1002, 1003],
    aggregate=True
)

# Cluster related topics
clusters = analyzer.cluster_topics(
    platform='all',
    min_similarity=0.7,
    time_window='7d'
)
```

### Push Notification System

```python
from hotsearch_analysis_agent.push import PushService

# Initialize push service
push_service = PushService()

# Create alert task
task = {
    'title': '关于人工智能与前沿科技的热点分析',
    'content': '分析报告内容...',
    'channels': ['wechat_work', 'telegram', 'email'],
    'priority': 'high'
}

# Send push notification
result = push_service.send_alert(task)
```

## Common Patterns

### 1. Scheduled Crawling with Analysis

```python
from apscheduler.schedulers.background import BackgroundScheduler
from hotsearch_analysis_agent.crawler_controller import CrawlerController
from hotsearch_analysis_agent.analyzer import OpinionAnalyzer

scheduler = BackgroundScheduler()
controller = CrawlerController()
analyzer = OpinionAnalyzer()

def crawl_and_analyze():
    # Run crawlers
    controller.start_all_spiders()
    
    # Wait for completion and analyze
    time.sleep(300)  # 5 minutes
    
    # Analyze new data
    insights = analyzer.get_daily_insights()
    
    # Send to stakeholders
    push_service.send_alert({
        'title': '每日舆情简报',
        'content': insights,
        'channels': ['email']
    })

# Schedule every 6 hours
scheduler.add_job(crawl_and_analyze, 'interval', hours=6)
scheduler.start()
```

### 2. Topic Monitoring with Alerts

```python
from hotsearch_analysis_agent.monitor import TopicMonitor

monitor = TopicMonitor()

# Define monitoring rules
monitor.add_rule(
    keywords=['数据泄露', '隐私风险', '安全漏洞'],
    platforms=['weibo', 'zhihu', 'toutiao'],
    threshold_rank=20,  # Alert if in top 20
    callback=lambda event: push_service.send_alert({
        'title': f'安全舆情预警: {event.title}',
        'content': event.summary,
        'channels': ['wechat_work', 'telegram'],
        'priority': 'urgent'
    })
)

monitor.start()  # Runs in background
```

### 3. Custom LLM Analysis

```python
import os
from openai import OpenAI

# Initialize OpenAI-compatible client
client = OpenAI(
    api_key=os.getenv('OPENAI_API_KEY'),
    base_url=os.getenv('OPENAI_BASE_URL')
)

def analyze_news_batch(news_items):
    """Batch analyze news sentiment and topics"""
    
    combined_text = "\n\n".join([
        f"【{item['platform']}】{item['title']}: {item['content']}"
        for item in news_items
    ])
    
    response = client.chat.completions.create(
        model="gpt-4",  # or pangu, deepseek-chat
        messages=[
            {
                "role": "system",
                "content": "你是舆情分析专家,擅长识别新闻情感倾向和关键话题"
            },
            {
                "role": "user",
                "content": f"分析以下新闻的整体情感倾向和核心话题:\n\n{combined_text}"
            }
        ],
        temperature=0.3
    )
    
    return response.choices[0].message.content
```

### 4. Database Query Examples

```python
import pymysql
import os

# Connect to database
conn = pymysql.connect(
    host=os.getenv('MYSQL_HOST'),
    port=int(os.getenv('MYSQL_PORT', 3306)),
    user=os.getenv('MYSQL_USER'),
    password=os.getenv('MYSQL_PASSWORD'),
    database=os.getenv('MYSQL_DATABASE'),
    charset='utf8mb4'
)

# Get top trending topics across all platforms
def get_cross_platform_trends(hours=24):
    query = """
    SELECT title, platform, rank, heat, url, created_at
    FROM hot_search_items
    WHERE created_at >= DATE_SUB(NOW(), INTERVAL %s HOUR)
    AND rank <= 10
    ORDER BY platform, rank
    """
    
    with conn.cursor(pymysql.cursors.DictCursor) as cursor:
        cursor.execute(query, (hours,))
        return cursor.fetchall()

# Track topic evolution over time
def track_topic_evolution(keyword, days=7):
    query = """
    SELECT DATE(created_at) as date, 
           MIN(rank) as best_rank,
           AVG(rank) as avg_rank,
           COUNT(*) as appearances
    FROM hot_search_items
    WHERE title LIKE %s
    AND created_at >= DATE_SUB(NOW(), INTERVAL %s DAY)
    GROUP BY DATE(created_at)
    ORDER BY date DESC
    """
    
    with conn.cursor(pymysql.cursors.DictCursor) as cursor:
        cursor.execute(query, (f'%{keyword}%', days))
        return cursor.fetchall()
```

## Configuration Best Practices

### Crawler Settings

Edit `hotsearchcrawler/settings.py`:

```python
# Concurrent requests (adjust based on server capacity)
CONCURRENT_REQUESTS = 16
CONCURRENT_REQUESTS_PER_DOMAIN = 4

# Download delays to avoid blocking
DOWNLOAD_DELAY = 2
RANDOMIZE_DOWNLOAD_DELAY = True

# User agent rotation
USER_AGENT_LIST = [
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
    'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36',
    # Add more user agents
]

# Retry settings
RETRY_TIMES = 3
RETRY_HTTP_CODES = [500, 502, 503, 504, 408, 429]

# Pipeline for data storage
ITEM_PIPELINES = {
    'hotsearchcrawler.pipelines.MySQLPipeline': 300,
    'hotsearchcrawler.pipelines.DeduplicationPipeline': 200,
}
```

### LLM Model Selection

For Chinese content analysis, recommended models:

```bash
# Huawei Pangu (recommended by project)
OPENAI_BASE_URL=https://ai.gitcode.com/api/v1
# Model: openpangu-embedded-7b

# DeepSeek (good performance on Chinese)
OPENAI_BASE_URL=https://api.deepseek.com/v1
# Model: deepseek-chat

# OpenAI (general purpose)
OPENAI_BASE_URL=https://api.openai.com/v1
# Model: gpt-4 or gpt-3.5-turbo
```

## Troubleshooting

### Crawler Issues

**Problem**: WebDriver not found
```bash
# Solution: Verify driver in PATH
which chromedriver  # Linux/macOS
where chromedriver  # Windows

# Or specify driver path in code
from selenium import webdriver
driver = webdriver.Chrome(executable_path='/path/to/chromedriver')
```

**Problem**: Anti-crawler blocking (403, CAPTCHA)
```python
# Solution: Add cookies for specific platforms
# In hotsearchcrawler/settings.py
COOKIES_WEIBO = {
    'SUB': 'your_cookie_value',
    'SUBP': 'your_cookie_value'
}

# Use in spider
def start_requests(self):
    for url in self.start_urls:
        yield scrapy.Request(url, cookies=COOKIES_WEIBO)
```

### Database Issues

**Problem**: Connection timeout
```bash
# Check MySQL is running
systemctl status mysql  # Linux
# or
brew services list  # macOS

# Verify connection parameters
mysql -h localhost -u your_user -p hotsearch
```

**Problem**: Character encoding errors
```sql
-- Ensure database and tables use utf8mb4
ALTER DATABASE hotsearch CHARACTER SET = utf8mb4 COLLATE = utf8mb4_unicode_ci;
ALTER TABLE hot_search_items CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### LLM Analysis Issues

**Problem**: API rate limiting
```python
# Solution: Implement retry with exponential backoff
import time
from openai import RateLimitError

def call_llm_with_retry(messages, max_retries=3):
    for attempt in range(max_retries):
        try:
            return client.chat.completions.create(
                model="gpt-4",
                messages=messages
            )
        except RateLimitError:
            if attempt < max_retries - 1:
                wait_time = (2 ** attempt) * 5
                time.sleep(wait_time)
            else:
                raise
```

**Problem**: Context length exceeded
```python
# Solution: Truncate or summarize long texts
def truncate_content(text, max_tokens=3000):
    # Rough estimate: 1 token ≈ 1.5 Chinese characters
    max_chars = int(max_tokens * 1.5)
    if len(text) > max_chars:
        return text[:max_chars] + "...[内容已截断]"
    return text
```

### Push Notification Issues

**Problem**: WeChat Work webhook fails
```python
# Verify webhook URL format and key
# Test with curl:
curl -X POST 'https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY' \
  -H 'Content-Type: application/json' \
  -d '{"msgtype": "text", "text": {"content": "测试消息"}}'
```

**Problem**: Email SMTP authentication fails
```bash
# For Gmail, use App Password (not regular password)
# Enable 2FA, then generate App Password at:
# https://myaccount.google.com/apppasswords

# Test SMTP connection
python -c "import smtplib; smtplib.SMTP('smtp.gmail.com', 587).starttls()"
```

## Web UI Access

After running `python app.py`, access the web interface:

```
http://localhost:5000
```

Key features:
- **Dashboard**: Overview of trending topics across platforms
- **Search**: Natural language queries (e.g., "过去24小时微博上关于AI的热搜")
- **Crawler Control**: Start/stop crawlers with keyboard shortcuts
- **Analytics**: View sentiment analysis, topic clusters, and trend charts
- **Alerts**: Configure and manage push notification tasks

## Performance Optimization

```python
# Use connection pooling for database
from dbutils.pooled_db import PooledDB
import pymysql

db_pool = PooledDB(
    creator=pymysql,
    maxconnections=10,
    host=os.getenv('MYSQL_HOST'),
    user=os.getenv('MYSQL_USER'),
    password=os.getenv('MYSQL_PASSWORD'),
    database=os.getenv('MYSQL_DATABASE'),
    charset='utf8mb4'
)

# Batch insert for crawler data
def batch_insert_items(items, batch_size=100):
    conn = db_pool.connection()
    cursor = conn.cursor()
    
    for i in range(0, len(items), batch_size):
        batch = items[i:i+batch_size]
        query = """
        INSERT INTO hot_search_items 
        (platform, title, rank, heat, url, created_at)
        VALUES (%s, %s, %s, %s, %s, NOW())
        """
        cursor.executemany(query, batch)
    
    conn.commit()
    cursor.close()
    conn.close()
```

This system provides comprehensive public opinion monitoring for Chinese platforms with AI-powered analysis and flexible alerting.
