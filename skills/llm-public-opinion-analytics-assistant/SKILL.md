---
name: llm-public-opinion-analytics-assistant
description: Deploy and use an LLM-powered public opinion analytics assistant that crawls 26 trending lists from 15 platforms, performs sentiment analysis, topic clustering, and pushes reports via email/WeChat/Telegram
triggers:
  - set up public opinion monitoring system
  - analyze trending topics across social platforms
  - configure hot search crawler with sentiment analysis
  - implement multi-platform trending data aggregation
  - create automated opinion report push notifications
  - deploy LLM-based sentiment analysis for news
  - build real-time hot topic tracking system
  - analyze public sentiment on trending topics
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is a comprehensive public opinion analytics platform that combines real-time data from **26 trending lists across 15 major platforms** (Weibo, Bilibili, Douyin, Baidu, Zhihu, etc.) with LLM analysis capabilities. It provides conversational querying of trending topics, topic clustering, sentiment analysis, and automated report pushing through multiple channels (email, WeChat, enterprise WeChat, Telegram).

**Key Features:**
- Multi-platform web crawlers for trending data (Scrapy-based)
- LLM-powered analysis (supports OpenAI-compatible APIs, with Huawei PanGu model recommended)
- Natural language query interface for trending data
- Automated sentiment analysis and topic clustering
- Multi-channel push notifications for hot topics
- Keyboard shortcuts for crawler control
- Detail page content extraction (including video content)

## Installation

### Prerequisites

1. **Browser Driver Setup** (required for detail page scraping):

```bash
# Check your Chrome/Edge browser version first
# Chrome: Settings → About Chrome
# Edge: Settings → About Microsoft Edge

# Download matching driver:
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Linux/macOS - place driver in PATH
sudo mv chromedriver /usr/local/bin/
chmod +x /usr/local/bin/chromedriver

# Verify installation
chromedriver --version
```

2. **MySQL Database Setup**:

```bash
# Install MySQL 5.7+ or 8.0+
# Create database
mysql -u root -p

CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

3. **Python Environment**:

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
# Reference init.py for table schemas
# Key tables: trending_data, analysis_results, push_tasks

import pymysql

connection = pymysql.connect(
    host='localhost',
    user='your_user',
    password='your_password',
    database='hotsearch_db',
    charset='utf8mb4'
)

# Execute table creation scripts from init.py
# Tables include: platforms, trending_lists, news_details, sentiment_analysis, etc.
```

## Configuration

### 1. Crawler Configuration

Edit `hotsearchcrawler/settings.py`:

```python
# MySQL connection
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_user'
MYSQL_PASSWORD = 'your_password'
MYSQL_DATABASE = 'hotsearch_db'

# Browser driver path (if not in PATH)
CHROME_DRIVER_PATH = '/usr/local/bin/chromedriver'

# Optional: Platform cookies for authentication
PLATFORM_COOKIES = {
    'weibo': 'your_weibo_cookie',
    'bilibili': 'your_bilibili_cookie'
}

# Crawler settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 0.5
ROBOTSTXT_OBEY = False
```

### 2. Analysis System Configuration

Create `.env` file in project root:

```bash
# Database
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch_db

# LLM API (OpenAI-compatible)
LLM_API_BASE=https://api.openai.com/v1
LLM_API_KEY=your_api_key
LLM_MODEL=gpt-4

# Or use Huawei PanGu (recommended for Chinese text)
# LLM_API_BASE=http://localhost:8000/v1
# LLM_MODEL=openpangu-embedded-7b

# Push notifications
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password

WECHAT_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Frontend
FRONTEND_PORT=5000
```

## Running the System

### Start Crawlers

```bash
# From project root
python run_spiders.py

# Or test individual spider
cd hotsearchcrawler
scrapy crawl weibo_hot -a test=1
scrapy crawl bilibili_hot
scrapy crawl douyin_hot
```

### Start Analysis Server

```bash
# Start main application
python app.py

# Access frontend
# Open browser: http://localhost:5000
```

## Key Usage Patterns

### 1. Query Trending Topics via API

```python
import requests

# Query all trending topics
response = requests.get('http://localhost:5000/api/trending')
data = response.json()

for item in data['items']:
    print(f"{item['platform']}: {item['title']} (热度: {item['heat']})")

# Query specific platform
response = requests.get('http://localhost:5000/api/trending?platform=weibo')

# Search by keyword
response = requests.post(
    'http://localhost:5000/api/search',
    json={'keyword': '人工智能', 'days': 7}
)
```

### 2. Perform Sentiment Analysis

```python
import requests

# Analyze sentiment for a topic
response = requests.post(
    'http://localhost:5000/api/analyze/sentiment',
    json={
        'topic_id': 12345,
        'content': '新闻内容或评论文本...'
    }
)

result = response.json()
print(f"情感: {result['sentiment']}")  # positive/negative/neutral
print(f"置信度: {result['confidence']}")
print(f"关键词: {result['keywords']}")
```

### 3. Topic Clustering

```python
# Cluster related topics
response = requests.post(
    'http://localhost:5000/api/analyze/cluster',
    json={
        'topic_ids': [123, 456, 789],
        'method': 'lda',  # or 'kmeans'
        'num_clusters': 3
    }
)

clusters = response.json()['clusters']
for cluster in clusters:
    print(f"\n聚类 {cluster['id']}:")
    print(f"主题: {cluster['theme']}")
    print(f"话题: {', '.join(cluster['topics'])}")
```

### 4. Configure Push Tasks

```python
import requests

# Create automated push task
task_config = {
    'name': '每日科技热点推送',
    'keywords': ['人工智能', '前沿科技', '芯片'],
    'platforms': ['weibo', 'zhihu', 'bilibili'],
    'schedule': '0 12 * * *',  # Daily at 12:00
    'channels': ['email', 'wechat'],
    'email_to': 'recipient@example.com',
    'wechat_webhook': 'YOUR_WEBHOOK_URL',
    'analysis_enabled': True,
    'sentiment_filter': 'all'  # or 'positive'/'negative'
}

response = requests.post(
    'http://localhost:5000/api/push/task',
    json=task_config
)

task_id = response.json()['task_id']
print(f"推送任务创建成功: {task_id}")
```

### 5. Direct Database Access

```python
import pymysql
from contextlib import contextmanager

@contextmanager
def get_db_connection():
    conn = pymysql.connect(
        host='localhost',
        user='your_user',
        password='your_password',
        database='hotsearch_db',
        charset='utf8mb4',
        cursorclass=pymysql.cursors.DictCursor
    )
    try:
        yield conn
    finally:
        conn.close()

# Query trending data
with get_db_connection() as conn:
    with conn.cursor() as cursor:
        sql = """
        SELECT platform, title, heat, url, created_at
        FROM trending_data
        WHERE created_at >= DATE_SUB(NOW(), INTERVAL 1 DAY)
        ORDER BY heat DESC
        LIMIT 50
        """
        cursor.execute(sql)
        results = cursor.fetchall()
        
        for row in results:
            print(f"{row['title']} ({row['platform']}) - {row['heat']}")
```

### 6. Custom Analysis with LLM

```python
from hotsearch_analysis_agent.llm_client import LLMClient

# Initialize LLM client
llm = LLMClient(
    api_base='http://localhost:8000/v1',
    api_key='your_key',
    model='openpangu-embedded-7b'
)

# Analyze news content
news_content = """
华为发布新一代昇腾AI芯片,性能提升300%...
"""

prompt = f"""
请分析以下新闻的核心观点、情感倾向和潜在影响:

{news_content}

输出格式:
1. 核心观点: 
2. 情感倾向: [正面/负面/中性]
3. 潜在影响:
4. 关键词:
"""

response = llm.generate(prompt)
print(response)
```

### 7. Generate Analysis Report

```python
from hotsearch_analysis_agent.report_generator import generate_report

# Generate comprehensive report
report = generate_report(
    keywords=['人工智能', '前沿科技'],
    date_range=('2026-04-01', '2026-04-07'),
    platforms=['weibo', 'zhihu', 'bilibili'],
    include_sentiment=True,
    include_clustering=True,
    output_format='markdown'  # or 'html', 'json'
)

print(report['title'])
print(report['summary'])
for section in report['sections']:
    print(f"\n## {section['heading']}")
    print(section['content'])
```

## Testing Push Notifications

```bash
# Test email push
python test_push_task.py --channel email --to recipient@example.com

# Test WeChat push
python test_push_task.py --channel wechat

# Test Telegram push
python test_push_task.py --channel telegram

# Test all channels
python test_push_task.py --all
```

## Common Patterns

### Pattern 1: Daily Trending Report Automation

```python
# scripts/daily_report.py
from hotsearch_analysis_agent.report_generator import generate_report
from hotsearch_analysis_agent.push_client import PushClient
import schedule
import time

def daily_report_task():
    # Generate report
    report = generate_report(
        keywords=['热点', '科技', '财经'],
        date_range='today',
        include_sentiment=True
    )
    
    # Push to channels
    pusher = PushClient()
    pusher.push_email(
        to='team@company.com',
        subject=f"每日舆情报告 - {report['date']}",
        content=report['html']
    )
    pusher.push_wechat(report['markdown'])

# Schedule daily at 9 AM
schedule.every().day.at("09:00").do(daily_report_task)

while True:
    schedule.run_pending()
    time.sleep(60)
```

### Pattern 2: Real-time Alert Monitoring

```python
# Monitor specific keywords and alert immediately
from hotsearch_analysis_agent.monitor import KeywordMonitor

monitor = KeywordMonitor(
    keywords=['公司名称', '产品名称'],
    alert_threshold=10000,  # Heat threshold
    check_interval=300  # 5 minutes
)

@monitor.on_alert
def handle_alert(trending_item):
    print(f"🔥 热点预警: {trending_item['title']}")
    print(f"热度: {trending_item['heat']}")
    
    # Perform immediate sentiment analysis
    sentiment = analyze_sentiment(trending_item['id'])
    
    if sentiment['score'] < -0.5:
        # Negative sentiment - urgent alert
        push_urgent_alert(trending_item, sentiment)

monitor.start()
```

### Pattern 3: Competitive Intelligence Tracking

```python
# Track competitors' mention trends
competitors = ['竞争对手A', '竞争对手B', '竞争对手C']

for competitor in competitors:
    results = search_trending(
        keyword=competitor,
        days=30,
        platforms=['weibo', 'zhihu', 'toutiao']
    )
    
    # Analyze trend
    heat_trend = [item['heat'] for item in results]
    sentiment_trend = [analyze_sentiment(item['id'])['score'] for item in results]
    
    # Generate competitor report
    print(f"\n=== {competitor} ===")
    print(f"提及次数: {len(results)}")
    print(f"平均热度: {sum(heat_trend)/len(heat_trend):.0f}")
    print(f"情感均值: {sum(sentiment_trend)/len(sentiment_trend):.2f}")
```

## Troubleshooting

### Issue: Crawler fails with timeout errors

```bash
# Increase timeout in hotsearchcrawler/settings.py
DOWNLOAD_TIMEOUT = 30
CONCURRENT_REQUESTS = 8  # Reduce concurrency

# Check browser driver
chromedriver --version
which chromedriver

# Test spider individually
cd hotsearchcrawler
scrapy crawl weibo_hot -L DEBUG
```

### Issue: Database connection errors

```python
# Verify MySQL connection
import pymysql

try:
    conn = pymysql.connect(
        host='localhost',
        user='your_user',
        password='your_password',
        database='hotsearch_db'
    )
    print("✅ Database connection successful")
    conn.close()
except Exception as e:
    print(f"❌ Connection failed: {e}")

# Check MySQL is running
# Linux: systemctl status mysql
# macOS: brew services list
# Windows: services.msc
```

### Issue: LLM API errors

```bash
# Test API connectivity
curl -X POST http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "openpangu-embedded-7b",
    "messages": [{"role": "user", "content": "测试"}]
  }'

# Check .env configuration
grep LLM_ .env

# Verify model is loaded (for local deployment)
# Check model files in expected directory
ls -lh /path/to/openpangu-embedded-7b-model/
```

### Issue: Push notifications not working

```python
# Test email settings
from hotsearch_analysis_agent.push_client import test_email

test_email(
    smtp_host='smtp.gmail.com',
    smtp_port=587,
    user='your_email@gmail.com',
    password='your_app_password',
    to='recipient@example.com'
)

# Test WeChat webhook
import requests

response = requests.post(
    'YOUR_WECHAT_WEBHOOK_URL',
    json={
        "msgtype": "text",
        "text": {"content": "测试消息"}
    }
)
print(response.status_code, response.text)
```

### Issue: Frontend not loading

```bash
# Check if port is already in use
lsof -i :5000  # macOS/Linux
netstat -ano | findstr :5000  # Windows

# Start with different port
FRONTEND_PORT=8080 python app.py

# Check logs
tail -f logs/app.log
```

## Advanced Configuration

### Deploy PanGu Model Locally

```bash
# Download model from Huawei GitCode
# https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

# Install vLLM or compatible serving framework
pip install vllm

# Start model server
python -m vllm.entrypoints.openai.api_server \
    --model /path/to/openpangu-embedded-7b-model \
    --host 0.0.0.0 \
    --port 8000

# Update .env to use local endpoint
LLM_API_BASE=http://localhost:8000/v1
LLM_MODEL=openpangu-embedded-7b
```

### Scale Crawler Cluster

```bash
# Use Scrapy distributed crawling
# Install scrapyd
pip install scrapyd

# Start scrapyd server
scrapyd

# Deploy spiders
cd hotsearchcrawler
scrapyd-deploy

# Schedule crawl jobs
curl http://localhost:6800/schedule.json \
    -d project=hotsearch \
    -d spider=weibo_hot
```

This skill provides comprehensive guidance for deploying and using the LLM-Based Public Opinion Analytics Assistant to monitor trending topics, analyze sentiment, and automate intelligence reporting across Chinese social media platforms.
