---
name: llm-public-opinion-analytics-assistant
description: Multi-platform hot search crawler with LLM-powered sentiment analysis, topic clustering, and automated push notifications across 15 platforms and 26 ranking lists
triggers:
  - build a public opinion monitoring system with LLM analysis
  - crawl hot search data from multiple Chinese platforms
  - analyze sentiment and cluster topics from social media trends
  - set up automated hot topic push notifications
  - deploy a real-time trending news analytics dashboard
  - integrate bilibili weibo douyin hot search crawlers
  - create an AI-powered public sentiment analysis tool
  - monitor and analyze trending topics across platforms
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive public opinion monitoring system that combines real-time web scraping from 15 Chinese platforms (26 ranking lists) with LLM-powered analysis. Provides conversational queries, topic clustering, sentiment analysis, and multi-channel push notifications (WeChat, Telegram, email).

## What This Project Does

- **Multi-Platform Crawling**: Scrapes hot search/trending data from Weibo, Bilibili, Douyin, Zhihu, Baidu, and 10+ other platforms
- **LLM Analysis**: Uses large language models (supports Huawei Pangu, OpenAI-compatible APIs) for:
  - Topic clustering and correlation detection
  - Sentiment/emotion analysis
  - Deep content extraction (including video transcripts)
- **Conversational Interface**: Natural language queries via web UI
- **Automated Notifications**: Push reports via WeChat Enterprise, Telegram, or email
- **Hot Key Control**: Keyboard shortcuts for crawler start/stop
- **Direct Navigation**: Click-through to original news sources

## Installation

### Prerequisites

**1. Browser Driver Setup**

Download ChromeDriver or EdgeDriver matching your browser version:

- Chrome: https://chromedriver.chromium.org/
- Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

Place driver in system PATH or project root.

**2. MySQL Database**

Install MySQL and create database:

```sql
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

**3. Python Environment**

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

### Database Initialization

Reference `init.py` for table schemas. Key tables:

```python
# Example initialization (adapt from init.py)
import mysql.connector

conn = mysql.connector.connect(
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

cursor = conn.cursor()

# Hot search entries table
cursor.execute('''
CREATE TABLE IF NOT EXISTS hot_search (
    id INT AUTO_INCREMENT PRIMARY KEY,
    platform VARCHAR(50),
    title VARCHAR(500),
    url VARCHAR(1000),
    hot_value VARCHAR(100),
    rank_position INT,
    crawl_time DATETIME,
    content TEXT,
    sentiment VARCHAR(50),
    INDEX idx_platform_time (platform, crawl_time)
)
''')

conn.commit()
```

## Configuration

### 1. Environment Variables (.env)

Create `.env` file in project root:

```env
# Database
DB_HOST=localhost
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=hotsearch_db

# LLM API (OpenAI-compatible format)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-3.5-turbo

# Or use Huawei Pangu (local deployment)
# PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model

# Push Notifications (optional)
WECHAT_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxx
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_email_password
```

### 2. Crawler Settings (hotsearchcrawler/settings.py)

```python
# MySQL connection
MYSQL_CONFIG = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

# Optional: Platform cookies for authenticated content
COOKIES = {
    'weibo': 'your_weibo_cookie_string',
    'bilibili': 'your_bilibili_cookie_string'
}

# Crawl interval (seconds)
DOWNLOAD_DELAY = 2
CONCURRENT_REQUESTS = 16
```

## Running the System

### Start Analysis System (Web UI)

```bash
python app.py
```

Access web interface at `http://localhost:5000`

### Run Crawlers

**Option 1: Via Web UI**
- Navigate to crawler control panel
- Use hotkeys (configured in UI) to start/stop crawlers

**Option 2: Command Line**

```bash
# Run all platform crawlers
python run_spiders.py

# Test specific platform
python runspider-test.py --platform weibo

# Run specific spider
cd hotsearchcrawler
scrapy crawl weibo_hot
```

### Test Push Notifications

```bash
python test_push_task.py
```

## Key API Usage

### Conversational Query Example

```python
from hotsearch_analysis_agent.agent import HotSearchAgent

agent = HotSearchAgent(
    db_config={
        'host': os.getenv('DB_HOST'),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    },
    llm_api_key=os.getenv('OPENAI_API_KEY'),
    llm_base_url=os.getenv('OPENAI_API_BASE')
)

# Natural language query
response = agent.query("今天关于人工智能的热点有哪些?")
print(response['answer'])
print(response['related_news'])
```

### Manual Sentiment Analysis

```python
from hotsearch_analysis_agent.analyzer import SentimentAnalyzer

analyzer = SentimentAnalyzer(model_name=os.getenv('MODEL_NAME'))

text = "这款产品太棒了!完全超出预期!"
sentiment = analyzer.analyze(text)
# Returns: {'label': 'positive', 'score': 0.95}
```

### Topic Clustering

```python
from hotsearch_analysis_agent.clusterer import TopicClusterer
import mysql.connector

conn = mysql.connector.connect(**db_config)
cursor = conn.cursor(dictionary=True)

# Fetch recent hot topics
cursor.execute("""
    SELECT title, platform, hot_value, url 
    FROM hot_search 
    WHERE crawl_time >= DATE_SUB(NOW(), INTERVAL 24 HOUR)
""")
topics = cursor.fetchall()

clusterer = TopicClusterer()
clusters = clusterer.cluster([t['title'] for t in topics])

for cluster_id, items in clusters.items():
    print(f"Cluster {cluster_id}:")
    for item in items:
        print(f"  - {item}")
```

### Push Notification Setup

```python
from hotsearch_analysis_agent.pusher import NotificationPusher

pusher = NotificationPusher(
    wechat_webhook=os.getenv('WECHAT_WEBHOOK'),
    telegram_token=os.getenv('TELEGRAM_BOT_TOKEN'),
    telegram_chat_id=os.getenv('TELEGRAM_CHAT_ID')
)

# Push analysis report
report = """
# AI 热点分析报告
时间: 2026-05-19 10:00

核心发现:
- GPT-6 上下文能力突破
- 国产算力生态协同
...
"""

pusher.push_wechat(report)
pusher.push_telegram(report)
```

## Common Patterns

### Pattern 1: Scheduled Monitoring Task

```python
import schedule
import time
from hotsearch_analysis_agent.monitor import HotTopicMonitor

monitor = HotTopicMonitor(
    db_config=db_config,
    keywords=['人工智能', '芯片', 'AI'],
    notification_channels=['wechat', 'telegram']
)

def check_hot_topics():
    results = monitor.check_keywords()
    if results['matches']:
        monitor.send_alert(results)

# Run every 30 minutes
schedule.every(30).minutes.do(check_hot_topics)

while True:
    schedule.run_pending()
    time.sleep(60)
```

### Pattern 2: Multi-Platform Data Aggregation

```python
import mysql.connector
from datetime import datetime, timedelta

def aggregate_platform_data(hours=24):
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor(dictionary=True)
    
    time_threshold = datetime.now() - timedelta(hours=hours)
    
    cursor.execute("""
        SELECT platform, COUNT(*) as count, AVG(rank_position) as avg_rank
        FROM hot_search
        WHERE crawl_time >= %s
        GROUP BY platform
        ORDER BY count DESC
    """, (time_threshold,))
    
    return cursor.fetchall()

stats = aggregate_platform_data()
for stat in stats:
    print(f"{stat['platform']}: {stat['count']} topics, avg rank {stat['avg_rank']:.1f}")
```

### Pattern 3: Custom Spider Integration

```python
# Add new platform spider in hotsearchcrawler/spiders/
import scrapy

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://example.com/hot']
    
    def parse(self, response):
        for item in response.css('.hot-item'):
            yield {
                'platform': 'CustomPlatform',
                'title': item.css('.title::text').get(),
                'url': item.css('a::attr(href)').get(),
                'hot_value': item.css('.hot-score::text').get(),
                'rank_position': item.css('.rank::text').get(),
                'crawl_time': scrapy.Field()
            }
```

## Troubleshooting

### Browser Driver Issues

**Error**: `selenium.common.exceptions.WebDriverException: chromedriver executable needs to be in PATH`

**Solution**:
```bash
# Linux/Mac
export PATH=$PATH:/path/to/chromedriver/directory

# Windows (PowerShell)
$env:PATH += ";C:\path\to\chromedriver"

# Or specify in code
from selenium import webdriver
driver = webdriver.Chrome(executable_path='/path/to/chromedriver')
```

### Database Connection Failures

**Error**: `mysql.connector.errors.ProgrammingError: Access denied`

**Solution**:
```sql
-- Grant privileges
GRANT ALL PRIVILEGES ON hotsearch_db.* TO 'your_user'@'localhost';
FLUSH PRIVILEGES;
```

### Crawler Rate Limiting

**Error**: `HTTP 429 Too Many Requests`

**Solution**: Adjust `hotsearchcrawler/settings.py`:
```python
DOWNLOAD_DELAY = 5  # Increase delay
CONCURRENT_REQUESTS = 8  # Reduce concurrency
AUTOTHROTTLE_ENABLED = True
AUTOTHROTTLE_TARGET_CONCURRENCY = 2.0
```

### LLM API Timeout

**Error**: `openai.error.Timeout: Request timed out`

**Solution**:
```python
import openai

openai.api_key = os.getenv('OPENAI_API_KEY')
openai.request_timeout = 120  # Increase timeout

# Or use retry wrapper
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=4, max=10))
def call_llm(prompt):
    return openai.ChatCompletion.create(...)
```

### Missing Video Content

**Error**: Video-based news returning empty content

**Solution**: Ensure FFmpeg is installed for video processing:
```bash
# Ubuntu/Debian
sudo apt-get install ffmpeg

# macOS
brew install ffmpeg

# Verify
ffmpeg -version
```

### WeChat Push Failures

**Error**: `{"errcode":93000,"errmsg":"invalid webhook"}`

**Solution**:
- Verify webhook URL is not expired (regenerate in WeChat Enterprise admin panel)
- Check message format adheres to WeChat's markdown spec:
```python
import requests

def push_wechat_safe(webhook, content):
    payload = {
        "msgtype": "markdown",
        "markdown": {
            "content": content.replace('#', '###')  # Adjust heading levels
        }
    }
    response = requests.post(webhook, json=payload)
    response.raise_for_status()
```

## Advanced Configuration

### Using Huawei Pangu Model (Local)

```python
# Download model from https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model
# Update .env
PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model
USE_LOCAL_MODEL=true

# In analysis code
from hotsearch_analysis_agent.llm_wrapper import LocalLLM

llm = LocalLLM(model_path=os.getenv('PANGU_MODEL_PATH'))
response = llm.generate("分析以下热点的情感倾向: ...")
```

### Custom Keyword Monitoring

```python
# Add to hotsearch_analysis_agent/config.py
MONITOR_KEYWORDS = {
    '科技': ['人工智能', 'AI', '芯片', '5G', '量子计算'],
    '金融': ['股市', '加密货币', '央行', '利率'],
    '社会': ['教育', '医疗', '房价', '就业']
}

# Use in monitoring script
from hotsearch_analysis_agent.monitor import KeywordMonitor

monitor = KeywordMonitor(keywords=MONITOR_KEYWORDS['科技'])
alerts = monitor.scan_and_alert(threshold=1000)  # Hot value threshold
```

This skill provides comprehensive coverage for deploying and using the LLM-based public opinion analytics system, from basic setup to advanced monitoring patterns.
