---
name: llm-public-opinion-analytics-assistant
description: Chinese public opinion analysis assistant integrating 15 platforms, 26 hot search lists, LLM analysis, sentiment detection, topic clustering, and multi-channel alerts
triggers:
  - how do I set up the public opinion analytics assistant
  - analyze hot topics from Chinese social media platforms
  - configure sentiment analysis with LLM models
  - crawl hot search data from Weibo Bilibili Douyin
  - set up multi-channel alerts for trending topics
  - deploy opinion monitoring with Pangu model
  - cluster and analyze social media sentiment
  - integrate hot search crawlers with LLM analysis
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is an intelligent Chinese public opinion monitoring system that combines real-time data from **15 mainstream platforms** (26 hot search lists) with LLM analysis capabilities. It provides conversational queries, topic clustering, sentiment analysis, and multi-channel alerting (email, WeChat, Enterprise WeChat, Telegram).

## What It Does

- **Multi-platform data collection**: Crawls hot search lists from Weibo, Bilibili, Douyin, Zhihu, and 11 other Chinese platforms
- **LLM-powered analysis**: Uses Pangu or OpenAI-compatible models for sentiment analysis, topic clustering, and report generation
- **Natural language interface**: Conversational queries for hot topics, theme searches, and trend analysis
- **Detail extraction**: Parses news article content including video-based content for deep analysis
- **Multi-channel alerts**: Pushes reports via Enterprise WeChat, Telegram, email (SMTP)
- **Keyboard-controlled crawlers**: Start/stop data collection with hotkeys

## Installation

### Prerequisites

**1. Browser Driver Setup**

Download the appropriate WebDriver for your browser:

- **Chrome**: [ChromeDriver](https://chromedriver.chromium.org/)
- **Edge**: [EdgeDriver](https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/)

Place the driver in your system PATH or browser installation directory.

Verify installation:
```bash
chromedriver --version
# or
msedgedriver --version
```

**2. MySQL Database**

Install MySQL and create the database:

```python
# Reference: init.py
import pymysql

connection = pymysql.connect(
    host='localhost',
    user='your_user',
    password='your_password',
    charset='utf8mb4'
)

cursor = connection.cursor()
cursor.execute("CREATE DATABASE IF NOT EXISTS hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci")
cursor.execute("USE hotsearch_db")

# Create hot search table
cursor.execute("""
CREATE TABLE IF NOT EXISTS hot_searches (
    id INT AUTO_INCREMENT PRIMARY KEY,
    platform VARCHAR(50),
    title VARCHAR(500),
    url TEXT,
    hot_value VARCHAR(100),
    rank_position INT,
    collected_at DATETIME,
    content TEXT,
    sentiment VARCHAR(20),
    INDEX idx_platform (platform),
    INDEX idx_collected_at (collected_at)
)
""")

connection.commit()
connection.close()
```

**3. Python Environment**

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Configuration

**1. Environment Variables (`.env`)**

```bash
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch_db

# LLM Model Configuration
OPENAI_API_BASE=http://localhost:8000/v1  # For Pangu or compatible models
OPENAI_API_KEY=your_api_key
MODEL_NAME=openpangu-embedded-7b

# Push Notification Channels
# Enterprise WeChat Bot
WECHAT_WEBHOOK_URL=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
ALERT_EMAIL=recipient@example.com
```

**2. Crawler Settings (`hotsearchcrawler/settings.py`)**

```python
# MySQL connection for crawler
MYSQL_CONFIG = {
    'host': 'localhost',
    'port': 3306,
    'user': 'your_user',
    'password': 'your_password',
    'database': 'hotsearch_db',
    'charset': 'utf8mb4'
}

# Optional: Platform-specific cookies
COOKIES = {
    'weibo': 'your_weibo_cookies',  # Optional for extended access
    'bilibili': 'your_bilibili_cookies'
}

# Browser settings
BROWSER_TYPE = 'chrome'  # or 'edge'
HEADLESS_MODE = True
```

## Starting the System

### 1. Launch the Web Application

```bash
python app.py
```

Access at `http://localhost:5000`

### 2. Start Crawlers

**Via Web Interface:**
- Use hotkey controls in the frontend

**Via Command Line:**
```bash
# Start all crawlers
python run_spiders.py

# Test specific crawler
cd hotsearchcrawler
scrapy crawl weibo_spider
scrapy crawl bilibili_spider
scrapy crawl douyin_spider
```

## Key API and Usage Patterns

### Querying Hot Search Data

```python
from hotsearch_analysis_agent.database import get_hot_searches

# Get latest hot searches from all platforms
hot_topics = get_hot_searches(limit=50)

# Filter by platform
weibo_topics = get_hot_searches(platform='weibo', limit=20)

# Filter by time range
from datetime import datetime, timedelta
recent = get_hot_searches(
    start_time=datetime.now() - timedelta(hours=6),
    end_time=datetime.now()
)
```

### LLM Analysis Integration

```python
from hotsearch_analysis_agent.llm_analyzer import OpinionAnalyzer

analyzer = OpinionAnalyzer(
    api_base=os.getenv('OPENAI_API_BASE'),
    api_key=os.getenv('OPENAI_API_KEY'),
    model=os.getenv('MODEL_NAME')
)

# Sentiment analysis
sentiment = analyzer.analyze_sentiment(
    text="今天的新闻真是令人震惊,没想到会这样发展"
)
# Returns: {'sentiment': 'negative', 'confidence': 0.87}

# Topic clustering
topics = [
    "GPT-6遭提前曝光, 2M超长上下文来了",
    "DeepSeek V4采用华为算力",
    "中国主流大模型周调用量连续五周超越美国"
]
clusters = analyzer.cluster_topics(topics)
# Returns: [{'cluster': 'AI模型发展', 'topics': [...], 'keywords': ['大模型', '算力']}]

# Generate report
report = analyzer.generate_report(
    query="人工智能与前沿科技",
    hot_searches=recent,
    include_sentiment=True,
    include_clustering=True
)
```

### Multi-Channel Push Notifications

```python
from hotsearch_analysis_agent.push_service import PushService

push = PushService()

# Enterprise WeChat
push.send_wechat(
    webhook_url=os.getenv('WECHAT_WEBHOOK_URL'),
    content=report,
    mentioned_list=['@all']
)

# Telegram
push.send_telegram(
    bot_token=os.getenv('TELEGRAM_BOT_TOKEN'),
    chat_id=os.getenv('TELEGRAM_CHAT_ID'),
    message=report,
    parse_mode='Markdown'
)

# Email
push.send_email(
    smtp_host=os.getenv('SMTP_HOST'),
    smtp_port=int(os.getenv('SMTP_PORT')),
    sender=os.getenv('SMTP_USER'),
    password=os.getenv('SMTP_PASSWORD'),
    recipients=[os.getenv('ALERT_EMAIL')],
    subject="舆情分析报告 - 2026-04-07",
    body=report,
    html=True
)
```

### Scheduled Push Tasks

```python
from hotsearch_analysis_agent.scheduler import ScheduledPushTask

# Create daily report task
task = ScheduledPushTask(
    name="daily_ai_report",
    query="人工智能与前沿科技",
    schedule="0 12 * * *",  # Daily at 12:00 PM
    channels=['wechat', 'telegram', 'email'],
    min_hot_value=5000  # Only topics with >5000 热度
)

task.start()
```

## Common Patterns

### Pattern 1: Real-time Monitoring with Alerts

```python
from hotsearch_analysis_agent.monitor import HotTopicMonitor

monitor = HotTopicMonitor(
    keywords=['人工智能', 'AI', '大模型'],
    platforms=['weibo', 'zhihu', 'bilibili'],
    check_interval=300,  # 5 minutes
    alert_threshold=10000  # Alert if hot_value > 10000
)

@monitor.on_alert
def handle_alert(topic):
    """Triggered when matching topic exceeds threshold"""
    analysis = analyzer.analyze_sentiment(topic['title'])
    
    push.send_telegram(
        bot_token=os.getenv('TELEGRAM_BOT_TOKEN'),
        chat_id=os.getenv('TELEGRAM_CHAT_ID'),
        message=f"🔥 热点预警\n\n{topic['title']}\n热度: {topic['hot_value']}\n情感: {analysis['sentiment']}"
    )

monitor.start()
```

### Pattern 2: Batch Analysis Pipeline

```python
from datetime import datetime, timedelta

# Collect data from last 24 hours
yesterday = datetime.now() - timedelta(days=1)
topics = get_hot_searches(start_time=yesterday, limit=200)

# Group by platform
from collections import defaultdict
by_platform = defaultdict(list)
for topic in topics:
    by_platform[topic['platform']].append(topic)

# Analyze each platform
results = {}
for platform, items in by_platform.items():
    titles = [item['title'] for item in items]
    
    # Cluster topics
    clusters = analyzer.cluster_topics(titles)
    
    # Sentiment distribution
    sentiments = [analyzer.analyze_sentiment(t) for t in titles]
    
    results[platform] = {
        'clusters': clusters,
        'sentiment_stats': {
            'positive': sum(1 for s in sentiments if s['sentiment'] == 'positive'),
            'negative': sum(1 for s in sentiments if s['sentiment'] == 'negative'),
            'neutral': sum(1 for s in sentiments if s['sentiment'] == 'neutral')
        }
    }

# Generate comparative report
comparative_report = analyzer.generate_comparative_report(results)
```

### Pattern 3: Custom Crawler Extension

```python
# hotsearchcrawler/spiders/custom_platform_spider.py
import scrapy
from hotsearchcrawler.items import HotSearchItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://example.com/hot-topics']
    
    def parse(self, response):
        for topic in response.css('.topic-item'):
            item = HotSearchItem()
            item['platform'] = 'custom_platform'
            item['title'] = topic.css('.title::text').get()
            item['url'] = topic.css('a::attr(href)').get()
            item['hot_value'] = topic.css('.hot::text').get()
            item['rank_position'] = topic.css('.rank::text').get()
            item['collected_at'] = datetime.now()
            
            # Follow detail page
            yield response.follow(
                item['url'],
                callback=self.parse_detail,
                meta={'item': item}
            )
    
    def parse_detail(self, response):
        item = response.meta['item']
        item['content'] = response.css('.content::text').getall()
        yield item
```

## Configuration Tips

### Using Pangu Model Locally

Download Pangu model from: https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

```bash
# Serve with vLLM or compatible server
python -m vllm.entrypoints.openai.api_server \
    --model /path/to/openpangu-embedded-7b \
    --port 8000 \
    --dtype auto \
    --api-key your_local_key
```

Update `.env`:
```bash
OPENAI_API_BASE=http://localhost:8000/v1
OPENAI_API_KEY=your_local_key
MODEL_NAME=openpangu-embedded-7b
```

### Optimizing Crawler Performance

```python
# hotsearchcrawler/settings.py

# Concurrent requests
CONCURRENT_REQUESTS = 16
CONCURRENT_REQUESTS_PER_DOMAIN = 4

# Download delay
DOWNLOAD_DELAY = 1
RANDOMIZE_DOWNLOAD_DELAY = True

# Retry settings
RETRY_TIMES = 3
RETRY_HTTP_CODES = [500, 502, 503, 504, 408, 429]

# Cache (for development)
HTTPCACHE_ENABLED = True
HTTPCACHE_EXPIRATION_SECS = 3600
```

## Troubleshooting

### WebDriver Not Found

```
selenium.common.exceptions.WebDriverException: Message: 'chromedriver' executable needs to be in PATH
```

**Solution:**
```bash
# Download driver and add to PATH
export PATH=$PATH:/path/to/driver/directory

# Or specify driver location in code
from selenium import webdriver
from selenium.webdriver.chrome.service import Service

service = Service('/path/to/chromedriver')
driver = webdriver.Chrome(service=service)
```

### MySQL Connection Errors

```
pymysql.err.OperationalError: (2003, "Can't connect to MySQL server")
```

**Solution:**
```bash
# Check MySQL is running
sudo systemctl status mysql

# Verify credentials
mysql -u your_user -p -h localhost

# Update .env with correct credentials
# Ensure database exists
mysql -u root -p -e "CREATE DATABASE IF NOT EXISTS hotsearch_db"
```

### LLM API Timeout

```
openai.error.Timeout: Request timed out
```

**Solution:**
```python
# Increase timeout in analyzer initialization
analyzer = OpinionAnalyzer(
    api_base=os.getenv('OPENAI_API_BASE'),
    api_key=os.getenv('OPENAI_API_KEY'),
    model=os.getenv('MODEL_NAME'),
    timeout=120  # Increase from default 30s
)

# Or implement retry logic
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(min=1, max=10))
def analyze_with_retry(text):
    return analyzer.analyze_sentiment(text)
```

### Crawler Cookie Expiration

```
403 Forbidden or missing content
```

**Solution:**
```python
# Update cookies in settings.py
# Export from browser DevTools > Application > Cookies
COOKIES = {
    'weibo': 'SUB=...; SUBP=...',  # Copy from browser
}

# Or implement dynamic cookie refresh
from selenium import webdriver

def get_fresh_cookies(url):
    driver = webdriver.Chrome()
    driver.get(url)
    input("Login manually, then press Enter...")
    cookies = driver.get_cookies()
    driver.quit()
    return {c['name']: c['value'] for c in cookies}
```

### Memory Issues with Large Datasets

```
MemoryError: Unable to allocate array
```

**Solution:**
```python
# Process in batches
def batch_analyze(topics, batch_size=50):
    results = []
    for i in range(0, len(topics), batch_size):
        batch = topics[i:i + batch_size]
        batch_results = [analyzer.analyze_sentiment(t) for t in batch]
        results.extend(batch_results)
        # Clear memory between batches
        import gc
        gc.collect()
    return results

# Or use database pagination
def get_hot_searches_paginated(page=1, page_size=100):
    offset = (page - 1) * page_size
    # Add LIMIT and OFFSET to SQL query
    return query_with_pagination(offset, page_size)
```
