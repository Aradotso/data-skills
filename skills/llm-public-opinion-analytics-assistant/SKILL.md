---
name: llm-public-opinion-analytics-assistant
description: Multi-platform hot topic crawler and LLM-powered public opinion analytics system with sentiment analysis and multi-channel alerts
triggers:
  - "set up public opinion monitoring system"
  - "crawl hot topics from multiple platforms"
  - "analyze sentiment and cluster news topics"
  - "configure hot topic push notifications"
  - "deploy Chinese social media crawler"
  - "integrate LLM for opinion analysis"
  - "build real-time trending topic dashboard"
  - "set up Weibo Douyin Bilibili crawler"
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive public opinion analytics platform that crawls 26 trending lists from 15 mainstream Chinese platforms (Weibo, Douyin, Bilibili, Zhihu, etc.) and provides LLM-powered analysis including topic clustering, sentiment analysis, and multi-channel alerting (Email, WeChat, Telegram).

## What This Project Does

- **Multi-Platform Crawling**: Collects real-time trending topics from 15 platforms including Weibo, Douyin, Bilibili, Zhihu, Baidu, Toutiao, etc.
- **Deep Content Analysis**: Extracts full news details including video content metadata
- **LLM-Powered Analytics**: Uses Huawei Pangu or OpenAI-compatible models for:
  - Natural language querying of trending topics
  - Topic clustering and correlation analysis
  - Sentiment analysis (positive/negative/neutral)
  - Automated report generation
- **Multi-Channel Alerts**: Push notifications via Enterprise WeChat, Telegram, Email
- **Interactive Dashboard**: Chat-based interface for querying and analyzing trends

## Installation

### Prerequisites

**1. Browser Driver Setup** (for content crawling):

```bash
# Download ChromeDriver or EdgeDriver matching your browser version
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Place driver in system PATH, e.g.:
# Windows: C:\Windows\System32\
# macOS/Linux: /usr/local/bin/

# Verify installation
chromedriver --version
```

**2. MySQL Database**:

```bash
# Install MySQL 5.7+ and create database
mysql -u root -p
CREATE DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

**3. Python Environment**:

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

```python
# Reference init.py for table structure
# Key tables:
# - hotsearch_data: Raw trending topic data
# - news_details: Extracted news content
# - analysis_results: LLM analysis outputs
# - push_tasks: Notification schedules

# Run initialization (adjust connection params in init.py)
python init.py
```

### Environment Configuration

Create `.env` file in project root:

```bash
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1  # or custom endpoint
OPENAI_MODEL=gpt-4  # or pangu-embedded-7b for local

# Push Notification Channels (optional)
# Email
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
NOTIFICATION_EMAIL=recipient@example.com

# Enterprise WeChat
WECHAT_WEBHOOK_URL=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxx
WECHAT_CORP_ID=your_corp_id
WECHAT_AGENT_ID=your_agent_id
WECHAT_SECRET=your_secret

# Telegram
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

### Crawler Configuration

Edit `hotsearchcrawler/settings.py`:

```python
# MySQL connection for crawler
MYSQL_CONFIG = {
    'host': 'localhost',
    'port': 3306,
    'user': 'root',
    'password': 'your_password',
    'database': 'hotsearch',
    'charset': 'utf8mb4'
}

# Optional: Platform cookies for authenticated content
# Some platforms may require login cookies
COOKIES = {
    'weibo': 'your_weibo_cookie',  # Optional
    'douyin': 'your_douyin_cookie'  # Optional
}

# Crawler settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
ROBOTSTXT_OBEY = False
```

## Running the System

### Start the Analysis Dashboard

```bash
# Launch web interface
python app.py

# Access at http://localhost:5000
# Default port can be changed in app.py
```

### Run Crawlers

**Via Web Interface** (Recommended):
- Navigate to dashboard
- Use hotkey controls to start/stop specific platform crawlers
- Monitor crawl status in real-time

**Via Command Line**:

```bash
# Test single platform crawler
python runspider-test.py

# Run all platform crawlers
python run_spiders.py

# The crawler cluster is in hotsearchcrawler/ directory
cd hotsearchcrawler
scrapy crawl weibo_hot  # Weibo trending
scrapy crawl douyin_hot  # Douyin trending
scrapy crawl bilibili_hot  # Bilibili trending
# ... etc for 26 different trending lists
```

## Key Usage Patterns

### 1. Querying Trending Topics via Chat Interface

```python
# In the web dashboard chat:
# User: "Show me top 10 trending topics on Weibo today"
# User: "Analyze sentiment for topics about AI"
# User: "Cluster news about technology in the past week"

# The system uses LLM to:
# 1. Parse natural language query
# 2. Query database for matching topics
# 3. Perform clustering/sentiment analysis
# 4. Return formatted results
```

### 2. Programmatic Data Access

```python
import mysql.connector
from datetime import datetime, timedelta

# Connect to database
conn = mysql.connector.connect(
    host=os.getenv('MYSQL_HOST'),
    user=os.getenv('MYSQL_USER'),
    password=os.getenv('MYSQL_PASSWORD'),
    database=os.getenv('MYSQL_DATABASE')
)
cursor = conn.cursor(dictionary=True)

# Get trending topics from last 24 hours
yesterday = datetime.now() - timedelta(days=1)
query = """
    SELECT platform, title, hot_value, url, crawl_time
    FROM hotsearch_data
    WHERE crawl_time >= %s
    ORDER BY hot_value DESC
    LIMIT 50
"""
cursor.execute(query, (yesterday,))
trending = cursor.fetchall()

for topic in trending:
    print(f"[{topic['platform']}] {topic['title']} - {topic['hot_value']}")
```

### 3. LLM Analysis Integration

```python
import openai
import os

# Configure OpenAI-compatible client
openai.api_key = os.getenv('OPENAI_API_KEY')
openai.api_base = os.getenv('OPENAI_API_BASE')

def analyze_sentiment(topics):
    """Analyze sentiment of trending topics using LLM"""
    prompt = f"""
    Analyze the sentiment (positive/negative/neutral) of these trending topics:
    
    {chr(10).join([f"- {t['title']}" for t in topics])}
    
    Return JSON format: [{{"title": "...", "sentiment": "positive/negative/neutral", "confidence": 0.0-1.0}}]
    """
    
    response = openai.ChatCompletion.create(
        model=os.getenv('OPENAI_MODEL'),
        messages=[
            {"role": "system", "content": "You are a sentiment analysis expert for Chinese social media."},
            {"role": "user", "content": prompt}
        ],
        temperature=0.3
    )
    
    return response.choices[0].message.content

# Usage
trending_topics = [{"title": "新款手机发布"}, {"title": "环保政策实施"}]
sentiment_analysis = analyze_sentiment(trending_topics)
print(sentiment_analysis)
```

### 4. Topic Clustering

```python
def cluster_topics(topics, query_theme):
    """Cluster related topics using LLM"""
    prompt = f"""
    Given these trending topics, identify clusters related to: {query_theme}
    
    Topics:
    {chr(10).join([f"{i+1}. {t['title']}" for i, t in enumerate(topics)])}
    
    Group related topics into clusters and provide:
    - Cluster name
    - Member topic IDs
    - Common theme summary
    """
    
    response = openai.ChatCompletion.create(
        model=os.getenv('OPENAI_MODEL'),
        messages=[
            {"role": "system", "content": "You are an expert at identifying topic relationships and clusters."},
            {"role": "user", "content": prompt}
        ],
        temperature=0.5
    )
    
    return response.choices[0].message.content

# Usage
ai_topics = cursor.execute(
    "SELECT * FROM hotsearch_data WHERE title LIKE '%人工智能%' OR title LIKE '%AI%'"
).fetchall()
clusters = cluster_topics(ai_topics, "人工智能")
```

### 5. Setting Up Push Notifications

```python
# Test push task configuration
# Edit test_push_task.py

from hotsearch_analysis_agent.push_service import PushService

# Create push service
pusher = PushService()

# Configure push task
task_config = {
    'query_keywords': ['人工智能', '科技', '芯片'],
    'platforms': ['weibo', 'douyin', 'zhihu'],
    'channels': ['email', 'wechat'],  # 'telegram' also supported
    'schedule': '0 9,18 * * *',  # Cron format: 9 AM and 6 PM daily
    'min_hot_value': 100000,  # Minimum hotness threshold
    'sentiment_filter': None  # or 'positive', 'negative', 'neutral'
}

# Create scheduled push task
pusher.create_task(
    name='AI Tech Daily Digest',
    config=task_config
)

# Manual push (for testing)
pusher.send_report(
    query='人工智能',
    channels=['email']
)
```

### 6. Extracting Video Content Details

```python
# The crawler automatically extracts video metadata
# even for video-based platforms like Douyin, Bilibili

def get_video_details(url):
    """Retrieve detailed content from video news"""
    query = """
        SELECT title, description, tags, view_count, 
               comment_count, like_count, video_duration
        FROM news_details
        WHERE source_url = %s
    """
    cursor.execute(query, (url,))
    details = cursor.fetchone()
    return details

# Example: Get Bilibili video trending details
video_url = 'https://www.bilibili.com/video/BV13pSoBBEvX'
details = get_video_details(video_url)
print(f"Title: {details['title']}")
print(f"Views: {details['view_count']}")
print(f"Description: {details['description']}")
```

## Platform Coverage

The system supports 26 trending lists across 15 platforms:

| Platform | Lists Supported |
|----------|----------------|
| 微博 (Weibo) | 热搜榜, 要闻榜 |
| 抖音 (Douyin) | 热点榜, 娱乐榜, 社会榜 |
| B站 (Bilibili) | 综合热门, 每周必看 |
| 知乎 (Zhihu) | 热榜 |
| 百度 (Baidu) | 热搜榜, 实时热点 |
| 头条 (Toutiao) | 热榜 |
| 贴吧 (Tieba) | 热议榜 |
| 小红书 (Xiaohongshu) | 热搜榜 |
| 36氪 (36Kr) | 快讯 |
| 虎扑 (Hupu) | 热帖 |
| IT之家 | 热榜 |
| 澎湃新闻 | 热榜 |
| 少数派 (SSPAI) | 热文 |
| 网易新闻 | 热点 |
| 腾讯新闻 | 热点 |

## Report Generation Example

The system can generate comprehensive analytical reports:

```python
def generate_opinion_report(query, days=7):
    """Generate comprehensive opinion analysis report"""
    
    # 1. Fetch relevant data
    cutoff = datetime.now() - timedelta(days=days)
    cursor.execute("""
        SELECT * FROM hotsearch_data 
        WHERE (title LIKE %s OR content LIKE %s)
        AND crawl_time >= %s
        ORDER BY hot_value DESC
    """, (f'%{query}%', f'%{query}%', cutoff))
    
    topics = cursor.fetchall()
    
    # 2. Fetch detailed content for top topics
    detailed_content = []
    for topic in topics[:20]:  # Top 20
        cursor.execute("""
            SELECT * FROM news_details 
            WHERE source_url = %s
        """, (topic['url'],))
        detail = cursor.fetchone()
        if detail:
            detailed_content.append(detail)
    
    # 3. LLM-based comprehensive analysis
    prompt = f"""
    Generate a comprehensive public opinion analysis report about: {query}
    
    Data summary:
    - Total topics found: {len(topics)}
    - Top platforms: {', '.join(set([t['platform'] for t in topics[:10]]))}
    - Time range: Past {days} days
    
    Top trending topics:
    {chr(10).join([f"{i+1}. [{t['platform']}] {t['title']} (热度: {t['hot_value']})" for i, t in enumerate(topics[:10])])}
    
    Detailed content available for {len(detailed_content)} topics.
    
    Please provide:
    1. Core findings and data highlights (🔹 bullet points)
    2. Detailed news content summary
    3. Analysis and conclusions (✅ checkmarks)
    4. Information dissemination characteristics
    5. Overall summary
    
    Format as a professional Chinese report with markdown formatting.
    """
    
    response = openai.ChatCompletion.create(
        model=os.getenv('OPENAI_MODEL'),
        messages=[
            {"role": "system", "content": "You are an expert public opinion analyst specializing in Chinese social media trends."},
            {"role": "user", "content": prompt}
        ],
        temperature=0.7,
        max_tokens=4000
    )
    
    report = response.choices[0].message.content
    
    # 4. Save report
    report_time = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
    full_report = f"""
# Report - {query} 热点分析
**Time**: {report_time}

{report}
    """
    
    return full_report

# Usage
report = generate_opinion_report('人工智能与前沿科技', days=7)
print(report)
```

## Troubleshooting

### Crawler Issues

**Problem**: ChromeDriver not found
```bash
# Solution: Verify driver in PATH
which chromedriver  # macOS/Linux
where chromedriver  # Windows

# Or set explicit path in crawler settings
CHROME_DRIVER_PATH = '/usr/local/bin/chromedriver'
```

**Problem**: Platform blocking or rate limiting
```python
# In hotsearchcrawler/settings.py
# Increase delays
DOWNLOAD_DELAY = 3  # seconds between requests
CONCURRENT_REQUESTS = 8  # reduce concurrency

# Add user agent rotation
USER_AGENT_LIST = [
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64)...',
    'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)...'
]

# Enable cookies for authenticated access
# Some platforms require login cookies
```

**Problem**: Missing content from video platforms
```python
# Ensure video extraction middleware is enabled
# Check hotsearchcrawler/middlewares.py

# For platforms like Douyin/Bilibili
# May need platform-specific API credentials
# Add to settings.py:
DOUYIN_API_KEY = os.getenv('DOUYIN_API_KEY')  # if available
```

### Database Issues

**Problem**: Character encoding errors
```sql
-- Ensure database and tables use utf8mb4
ALTER DATABASE hotsearch CHARACTER SET = utf8mb4 COLLATE = utf8mb4_unicode_ci;
ALTER TABLE hotsearch_data CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

**Problem**: Slow queries on large datasets
```sql
-- Add indexes on commonly queried fields
CREATE INDEX idx_crawl_time ON hotsearch_data(crawl_time);
CREATE INDEX idx_platform ON hotsearch_data(platform);
CREATE INDEX idx_hot_value ON hotsearch_data(hot_value);
CREATE FULLTEXT INDEX idx_title_content ON hotsearch_data(title, content);
```

### LLM Integration Issues

**Problem**: Local Pangu model not loading
```bash
# Download Pangu model
# From: https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

# Place in models/ directory
# Update .env:
OPENAI_API_BASE=http://localhost:8000/v1  # local inference server
OPENAI_MODEL=pangu-embedded-7b

# Start local inference server (if using vLLM or similar)
python -m vllm.entrypoints.openai.api_server \
    --model /path/to/pangu-embedded-7b \
    --host 0.0.0.0 \
    --port 8000
```

**Problem**: API rate limits or costs
```python
# Implement caching for repeated queries
from functools import lru_cache
import hashlib
import json

@lru_cache(maxsize=1000)
def cached_llm_analysis(query_hash):
    # Check database cache first
    cursor.execute("""
        SELECT result FROM analysis_cache 
        WHERE query_hash = %s 
        AND created_at > DATE_SUB(NOW(), INTERVAL 1 HOUR)
    """, (query_hash,))
    
    cached = cursor.fetchone()
    if cached:
        return json.loads(cached['result'])
    
    # Otherwise call LLM
    result = openai.ChatCompletion.create(...)
    
    # Cache result
    cursor.execute("""
        INSERT INTO analysis_cache (query_hash, result) 
        VALUES (%s, %s)
    """, (query_hash, json.dumps(result)))
    
    return result
```

### Push Notification Issues

**Problem**: WeChat webhook not working
```python
# Verify webhook URL format
# Should be: https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=XXXXX

# Test webhook
import requests
resp = requests.post(
    os.getenv('WECHAT_WEBHOOK_URL'),
    json={
        "msgtype": "text",
        "text": {"content": "Test message"}
    }
)
print(resp.status_code, resp.text)
```

**Problem**: Email SMTP authentication failed
```bash
# For Gmail, use App Password (not account password)
# Enable 2FA, then generate app password at:
# https://myaccount.google.com/apppasswords

# Update .env:
SMTP_PASSWORD=your_16_char_app_password
```

**Problem**: Telegram bot not receiving messages
```python
# Verify bot token and chat ID
import requests

# Get bot info
resp = requests.get(f'https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/getMe')
print(resp.json())

# Get updates to find chat_id
resp = requests.get(f'https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/getUpdates')
print(resp.json())  # Look for chat.id in results
```

## Advanced Configuration

### Custom Platform Crawlers

Add new platforms by creating spider in `hotsearchcrawler/spiders/`:

```python
# hotsearchcrawler/spiders/custom_platform.py
import scrapy
from ..items import HotSearchItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://example.com/trending']
    
    def parse(self, response):
        for item in response.css('.trending-item'):
            yield HotSearchItem(
                platform='CustomPlatform',
                title=item.css('.title::text').get(),
                url=item.css('a::attr(href)').get(),
                hot_value=item.css('.hot-value::text').get(),
                rank=item.css('.rank::text').get()
            )
```

### Custom Analysis Pipelines

```python
# Create custom analyzer in hotsearch_analysis_agent/analyzers/

class CustomAnalyzer:
    def __init__(self, llm_client):
        self.llm = llm_client
    
    def analyze_trends(self, topics, metric='growth_rate'):
        """Calculate trending velocity"""
        # Compare current vs historical data
        # Return topics with highest growth rates
        pass
    
    def detect_emerging_topics(self, topics, threshold=0.8):
        """Identify suddenly popular new topics"""
        # Use LLM to detect novelty + velocity
        pass
```

This skill provides comprehensive guidance for deploying and using the LLM-Based Public Opinion Analytics Assistant for multi-platform trending topic monitoring and analysis.
