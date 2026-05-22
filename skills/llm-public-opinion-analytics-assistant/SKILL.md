---
name: llm-public-opinion-analytics-assistant
description: Chinese public opinion analytics system with multi-platform hot search crawling, LLM-based sentiment analysis, and multi-channel alerting
triggers:
  - how do I set up the public opinion analytics assistant
  - crawl hot search data from Chinese platforms
  - analyze sentiment and cluster topics with LLM
  - configure multi-platform news crawlers
  - send hot topic alerts to WeChat or Telegram
  - query trending topics across Weibo Bilibili Douyin
  - set up opinion monitoring with push notifications
  - integrate Pangu LLM for Chinese text analysis
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This is a comprehensive Chinese public opinion monitoring system that:

- **Crawls 26 hot search lists** from 15 major Chinese platforms (Weibo, Bilibili, Douyin, Baidu, Zhihu, etc.)
- **Analyzes content** using LLM (supports Pangu, OpenAI-compatible models) for sentiment analysis, topic clustering, and trend detection
- **Extracts deep content** from news detail pages including video transcripts
- **Pushes alerts** via email, WeChat (personal/enterprise), and Telegram
- **Provides conversational UI** for querying trends and analyzing topics

The system is split into two main components:
- **Crawler cluster** (`hotsearchcrawler/`) - Scrapy-based multi-platform crawlers
- **Analysis system** (`hotsearch_analysis_agent/`) - Flask backend with LLM integration

## Installation

### Prerequisites

**1. Browser Driver Setup**

The system requires Chrome/Edge WebDriver for dynamic content extraction:

```bash
# Check your Chrome version
google-chrome --version  # or microsoft-edge --version

# Download matching ChromeDriver from:
# https://chromedriver.chromium.org/
# or EdgeDriver from:
# https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Place driver in system PATH, e.g.:
# Linux/macOS: /usr/local/bin/chromedriver
# Windows: C:\Windows\System32\chromedriver.exe

# Verify installation
chromedriver --version
```

**2. Python Environment**

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

**3. MySQL Database**

```bash
# Install MySQL 5.7+ or MariaDB
# Create database (refer to init.py for schema)
mysql -u root -p

CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

Run `init.py` to initialize tables:

```python
# init.py example structure
import pymysql

connection = pymysql.connect(
    host='localhost',
    user='root',
    password='your_password',
    database='hotsearch_db',
    charset='utf8mb4'
)

# Tables: hot_searches, news_content, analysis_results, push_tasks, etc.
```

### Configuration

**1. Crawler Settings** (`hotsearchcrawler/settings.py`):

```python
# MySQL connection
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'root'
MYSQL_PASSWORD = 'your_password'
MYSQL_DATABASE = 'hotsearch_db'

# Optional: Platform cookies for authenticated access
COOKIES = {
    'weibo': 'your_weibo_cookie',  # Optional, improves data quality
    'bilibili': 'your_bilibili_cookie'
}

# Concurrent requests
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 0.5
```

**2. Analysis System** (`.env` file in project root):

```bash
# MySQL
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1  # or custom endpoint
OPENAI_MODEL=gpt-4  # or pangu-7b if using local deployment

# For Huawei Pangu (local deployment)
PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model
PANGU_API_ENDPOINT=http://localhost:8000/v1

# Push Notification Channels
# WeChat Work Robot
WECHAT_WORK_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# WeChat Work App (personal push)
WECHAT_CORP_ID=your_corp_id
WECHAT_AGENT_ID=your_agent_id
WECHAT_SECRET=your_secret

# Telegram
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
EMAIL_FROM=your_email@gmail.com
EMAIL_TO=recipient@example.com
```

## Running the System

### Start Analysis Backend

```bash
# Main application entry point
python app.py

# Starts Flask server on http://localhost:5000
# Access web UI at http://localhost:5000
```

### Run Crawlers

**Option 1: Via Web UI**
- Navigate to web interface
- Use keyboard shortcut to start/stop crawlers
- Monitor crawler status in dashboard

**Option 2: Manual Execution**

```bash
# Test single spider
python runspider-test.py

# Run all spiders (production)
python run_spiders.py
```

**Example Spider Test** (`runspider-test.py`):

```python
from scrapy.crawler import CrawlerProcess
from scrapy.utils.project import get_project_settings
from hotsearchcrawler.spiders.weibo_spider import WeiboSpider

# Get Scrapy settings
settings = get_project_settings()

# Override specific settings
settings.set('LOG_LEVEL', 'INFO')

# Run spider
process = CrawlerProcess(settings)
process.crawl(WeiboSpider)
process.start()
```

## Key Features & Code Examples

### 1. Query Hot Searches via API

```python
import requests

# Get trending topics from specific platform
response = requests.get('http://localhost:5000/api/hot_searches', params={
    'platform': 'weibo',  # weibo, bilibili, douyin, baidu, zhihu, etc.
    'limit': 20
})

data = response.json()
for item in data['results']:
    print(f"{item['rank']}. {item['title']} - {item['heat_value']}")
    print(f"   URL: {item['url']}")
```

### 2. Conversational Topic Search

```python
# Natural language query
response = requests.post('http://localhost:5000/api/chat', json={
    'message': '最近关于人工智能的热点有哪些?',  # "Recent AI hot topics?"
    'session_id': 'user123'
})

reply = response.json()
print(reply['answer'])  # LLM-generated summary with sources
```

### 3. Topic Clustering Analysis

```python
# Cluster related topics across platforms
response = requests.post('http://localhost:5000/api/analyze/cluster', json={
    'keyword': '人工智能',  # "Artificial Intelligence"
    'platforms': ['weibo', 'bilibili', 'zhihu'],
    'time_range': '7d'  # Last 7 days
})

clusters = response.json()
for cluster in clusters['clusters']:
    print(f"\n主题: {cluster['theme']}")
    print(f"相关话题数: {len(cluster['topics'])}")
    for topic in cluster['topics']:
        print(f"  - {topic['title']} ({topic['platform']})")
```

### 4. Sentiment Analysis

```python
# Analyze sentiment for specific topic
response = requests.post('http://localhost:5000/api/analyze/sentiment', json={
    'topic_id': 12345,
    'include_comments': True  # Analyze detail page + comments
})

sentiment = response.json()
print(f"总体情感: {sentiment['overall']}")  # positive/negative/neutral
print(f"正面: {sentiment['positive_ratio']}%")
print(f"负面: {sentiment['negative_ratio']}%")
print(f"\n情感摘要:\n{sentiment['summary']}")  # LLM-generated insight
```

### 5. Set Up Push Tasks

```python
# Create alert for specific keywords
response = requests.post('http://localhost:5000/api/push_task/create', json={
    'name': 'AI Tech Monitor',
    'keywords': ['GPT-6', '盘古大模型', 'DeepSeek'],
    'platforms': ['all'],
    'channels': ['wechat_work', 'telegram'],  # Multi-channel
    'frequency': 'daily',  # or 'realtime', 'hourly'
    'threshold': {
        'min_heat': 100000,  # Minimum heat value
        'sentiment': 'any'  # or 'positive', 'negative'
    }
})

task_id = response.json()['task_id']
print(f"Push task created: {task_id}")
```

**Test Push Task**:

```bash
# Test notification delivery
python test_push_task.py
```

```python
# test_push_task.py example
import os
from hotsearch_analysis_agent.push.wechat_pusher import WeChatWorkPusher
from hotsearch_analysis_agent.push.telegram_pusher import TelegramPusher

# Test WeChat Work push
wechat_pusher = WeChatWorkPusher(
    webhook_url=os.getenv('WECHAT_WORK_WEBHOOK')
)

report = {
    'title': '人工智能热点分析',
    'summary': 'GPT-6遭提前曝光, 2M超长上下文来了...',
    'topics': [
        {'title': 'GPT-6曝光', 'heat': 500000, 'url': 'https://...'}
    ]
}

wechat_pusher.send_report(report)

# Test Telegram push
telegram_pusher = TelegramPusher(
    bot_token=os.getenv('TELEGRAM_BOT_TOKEN'),
    chat_id=os.getenv('TELEGRAM_CHAT_ID')
)
telegram_pusher.send_report(report)
```

### 6. Custom Spider Development

Add new platform crawler:

```python
# hotsearchcrawler/spiders/new_platform_spider.py
import scrapy
from hotsearchcrawler.items import HotSearchItem

class NewPlatformSpider(scrapy.Spider):
    name = 'newplatform'
    allowed_domains = ['newplatform.com']
    start_urls = ['https://newplatform.com/hot']
    
    custom_settings = {
        'DOWNLOAD_DELAY': 1,
    }
    
    def parse(self, response):
        # Extract hot search items
        for item in response.css('.hot-item'):
            hot_search = HotSearchItem()
            hot_search['platform'] = 'newplatform'
            hot_search['title'] = item.css('.title::text').get()
            hot_search['url'] = item.css('a::attr(href)').get()
            hot_search['heat_value'] = item.css('.heat::text').get()
            hot_search['rank'] = item.css('.rank::text').get()
            
            # Follow detail page for content extraction
            yield response.follow(
                hot_search['url'],
                callback=self.parse_detail,
                meta={'hot_search': hot_search}
            )
    
    def parse_detail(self, response):
        hot_search = response.meta['hot_search']
        
        # Extract full content (including video transcripts if present)
        hot_search['content'] = response.css('.article-content').get()
        hot_search['publish_time'] = response.css('.pub-time::text').get()
        
        yield hot_search
```

Register in `run_spiders.py`:

```python
from hotsearchcrawler.spiders.new_platform_spider import NewPlatformSpider

spiders = [
    WeiboSpider,
    BilibiliSpider,
    NewPlatformSpider,  # Add here
    # ...
]
```

## LLM Integration (Pangu Model)

### Local Deployment Setup

```bash
# Download Pangu model
# From: https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

# Install inference framework (example with vLLM)
pip install vllm

# Start model server
python -m vllm.entrypoints.openai.api_server \
    --model /path/to/openpangu-embedded-7b-model \
    --host 0.0.0.0 \
    --port 8000
```

### Configure Analysis System

```python
# hotsearch_analysis_agent/llm/pangu_client.py
import openai

class PanguClient:
    def __init__(self):
        openai.api_base = os.getenv('PANGU_API_ENDPOINT', 'http://localhost:8000/v1')
        openai.api_key = 'not-needed-for-local'
        self.model = 'pangu-7b'
    
    def analyze_sentiment(self, text: str) -> dict:
        prompt = f"""分析以下新闻的情感倾向,并给出正面、负面、中性的比例:

{text}

请以JSON格式返回: {{"positive": 0.0-1.0, "negative": 0.0-1.0, "neutral": 0.0-1.0, "summary": "简短分析"}}"""
        
        response = openai.ChatCompletion.create(
            model=self.model,
            messages=[
                {"role": "system", "content": "你是一个专业的舆情分析助手"},
                {"role": "user", "content": prompt}
            ],
            temperature=0.3
        )
        
        return eval(response.choices[0].message.content)
    
    def cluster_topics(self, topics: list) -> list:
        topics_text = "\n".join([f"{i+1}. {t['title']}" for i, t in enumerate(topics)])
        
        prompt = f"""对以下热点话题进行聚类分析,找出相关联的主题:

{topics_text}

请识别出3-5个主题集群,每个集群包含相关话题的编号和一个主题名称。"""
        
        response = openai.ChatCompletion.create(
            model=self.model,
            messages=[{"role": "user", "content": prompt}],
            temperature=0.5
        )
        
        # Parse LLM response into clusters
        return self._parse_clusters(response.choices[0].message.content)
```

## Common Patterns

### Scheduled Crawling

```python
# Use APScheduler for periodic crawling
from apscheduler.schedulers.background import BackgroundScheduler

scheduler = BackgroundScheduler()

def crawl_job():
    """Run all spiders"""
    import subprocess
    subprocess.run(['python', 'run_spiders.py'])

# Run every hour
scheduler.add_job(crawl_job, 'interval', hours=1)
scheduler.start()
```

### Real-time Alert Pipeline

```python
# hotsearch_analysis_agent/monitor/realtime_monitor.py
import time
from hotsearch_analysis_agent.db.mysql_client import MySQLClient
from hotsearch_analysis_agent.llm.pangu_client import PanguClient
from hotsearch_analysis_agent.push.multi_channel_pusher import MultiChannelPusher

class RealtimeMonitor:
    def __init__(self):
        self.db = MySQLClient()
        self.llm = PanguClient()
        self.pusher = MultiChannelPusher()
    
    def monitor_keywords(self, keywords: list, threshold: int = 100000):
        """Monitor for keyword spikes"""
        last_check = {}
        
        while True:
            for keyword in keywords:
                # Query recent hot searches
                results = self.db.query(
                    "SELECT * FROM hot_searches WHERE title LIKE %s AND heat_value > %s ORDER BY created_at DESC LIMIT 5",
                    (f'%{keyword}%', threshold)
                )
                
                for item in results:
                    if item['id'] not in last_check:
                        # New hot topic detected
                        analysis = self.llm.analyze_sentiment(item['content'])
                        
                        alert = {
                            'keyword': keyword,
                            'topic': item,
                            'sentiment': analysis,
                            'timestamp': time.time()
                        }
                        
                        # Push to all configured channels
                        self.pusher.send_alert(alert)
                        last_check[item['id']] = True
            
            time.sleep(300)  # Check every 5 minutes
```

### Data Export

```python
# Export analysis results
import pandas as pd

def export_analysis(start_date: str, end_date: str, output_file: str):
    """Export analysis results to Excel"""
    db = MySQLClient()
    
    query = """
    SELECT 
        h.platform, h.title, h.heat_value, h.url,
        a.sentiment, a.summary, a.created_at
    FROM hot_searches h
    LEFT JOIN analysis_results a ON h.id = a.hot_search_id
    WHERE h.created_at BETWEEN %s AND %s
    ORDER BY h.heat_value DESC
    """
    
    results = db.query(query, (start_date, end_date))
    df = pd.DataFrame(results)
    df.to_excel(output_file, index=False, engine='openpyxl')
    
    print(f"Exported {len(results)} records to {output_file}")
```

## Troubleshooting

### Crawler Issues

**Problem: Spiders not collecting data**

```bash
# Check spider logs
tail -f logs/scrapy.log

# Common issues:
# 1. Website structure changed - update CSS selectors
# 2. Rate limiting - increase DOWNLOAD_DELAY in settings.py
# 3. Missing cookies - add authentication cookies for logged-in content
```

**Problem: ChromeDriver crashes**

```python
# Add headless mode and disable GPU in spider settings
from selenium.webdriver.chrome.options import Options

chrome_options = Options()
chrome_options.add_argument('--headless')
chrome_options.add_argument('--no-sandbox')
chrome_options.add_argument('--disable-dev-shm-usage')
chrome_options.add_argument('--disable-gpu')
```

### Database Issues

**Problem: MySQL connection errors**

```python
# Verify connection
import pymysql

try:
    conn = pymysql.connect(
        host=os.getenv('MYSQL_HOST'),
        user=os.getenv('MYSQL_USER'),
        password=os.getenv('MYSQL_PASSWORD'),
        database=os.getenv('MYSQL_DATABASE')
    )
    print("✓ Database connected")
except Exception as e:
    print(f"✗ Connection failed: {e}")
```

**Problem: Character encoding errors**

```sql
-- Ensure UTF8MB4 encoding for Chinese characters
ALTER DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
ALTER TABLE hot_searches CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### LLM Issues

**Problem: Pangu model inference slow**

```bash
# Use quantized model (faster inference)
python -m vllm.entrypoints.openai.api_server \
    --model /path/to/pangu-7b \
    --quantization awq \
    --dtype half \
    --max-model-len 4096

# Or use smaller context window
--max-model-len 2048
```

**Problem: LLM hallucinations in sentiment analysis**

```python
# Add stricter prompt formatting and temperature control
def analyze_sentiment(self, text: str) -> dict:
    prompt = f"""严格基于以下文本进行情感分析,不要添加文中没有的信息:

{text[:1000]}  # Limit context to reduce hallucination

仅返回JSON格式: {{"positive": 数值, "negative": 数值, "neutral": 数值, "summary": "简短总结"}}
数值范围0-1,总和必须为1。"""
    
    response = openai.ChatCompletion.create(
        model=self.model,
        messages=[{"role": "user", "content": prompt}],
        temperature=0.1,  # Lower temperature for factual analysis
        max_tokens=200
    )
    # ...
```

### Push Notification Issues

**Problem: WeChat Work webhook fails**

```python
# Test webhook directly
import requests

response = requests.post(
    os.getenv('WECHAT_WORK_WEBHOOK'),
    json={
        "msgtype": "text",
        "text": {
            "content": "Test message"
        }
    }
)

if response.status_code == 200:
    print("✓ Webhook working")
else:
    print(f"✗ Failed: {response.text}")
```

**Problem: Telegram rate limiting**

```python
# Implement rate limiting for Telegram bot
import time
from functools import wraps

def rate_limit(min_interval=1):
    """Decorator to limit function call frequency"""
    last_called = [0.0]
    
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            if elapsed < min_interval:
                time.sleep(min_interval - elapsed)
            result = func(*args, **kwargs)
            last_called[0] = time.time()
            return result
        return wrapper
    return decorator

@rate_limit(min_interval=1)
def send_telegram_message(bot_token, chat_id, text):
    # Send message with rate limiting
    pass
```

## Platform Support

Currently supported platforms:
- **Social Media**: Weibo (微博), Douyin (抖音), Kuaishou (快手)
- **Video**: Bilibili (哔哩哔哩), iQIYI (爱奇艺), Tencent Video (腾讯视频)
- **News**: Toutiao (今日头条), Baidu News (百度新闻), NetEase (网易)
- **Forums**: Zhihu (知乎), Tieba (百度贴吧), Douban (豆瓣)
- **E-commerce**: Taobao (淘宝), JD (京东), Pinduoduo (拼多多)

Each platform crawler handles:
- List page parsing for hot topics
- Detail page content extraction
- Video transcript extraction (where applicable)
- Anti-scraping measures (cookies, headers, delays)
