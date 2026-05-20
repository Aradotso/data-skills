---
name: llm-public-opinion-analytics-assistant
description: Multi-platform hot search data crawler and LLM-powered public opinion analysis system with sentiment analysis, topic clustering, and multi-channel alerting
triggers:
  - "set up public opinion monitoring system"
  - "analyze hot topics from multiple platforms"
  - "crawl trending news from social media"
  - "configure sentiment analysis for news"
  - "send hot topic alerts to wechat or telegram"
  - "cluster related news topics automatically"
  - "extract content from video news pages"
  - "deploy hotsearch crawler with LLM analysis"
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive public opinion monitoring system that crawls **26 trending lists** from **15 major platforms** (Weibo, Bilibili, Douyin, etc.), performs LLM-powered analysis (sentiment, clustering, summarization), and sends multi-channel alerts (Email, WeChat, Telegram). Features conversational querying, video content extraction, and real-time hot topic tracking.

## What This Project Does

- **Multi-Platform Crawling**: Scrapes hot search/trending data from 15 Chinese platforms using Scrapy
- **LLM Analysis**: Uses Huawei Pangu model (or OpenAI-compatible APIs) for sentiment analysis, topic clustering, and summarization
- **Content Extraction**: Retrieves full article/video content from detail pages for deeper analysis
- **Conversational Interface**: Natural language queries for trending topics and analysis
- **Multi-Channel Alerts**: Push reports to WeChat Work, Telegram, Email based on custom rules
- **Hotkey Control**: Start/stop crawlers via keyboard shortcuts

## Installation

### Prerequisites

**1. Browser Driver Setup** (required for content extraction):

Download the appropriate driver for your browser:
- **Chrome**: https://chromedriver.chromium.org/
- **Edge**: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

Place driver in system PATH:
```bash
# Linux/macOS - move to /usr/local/bin
sudo mv chromedriver /usr/local/bin/
sudo chmod +x /usr/local/bin/chromedriver

# Windows - add driver directory to PATH environment variable
# or place in C:\Windows\System32\
```

Verify installation:
```bash
chromedriver --version
```

**2. MySQL Database**:
```bash
# Install MySQL 8.0+
# Create database
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

Use `init.py` as reference to create tables:

```python
import pymysql

connection = pymysql.connect(
    host='localhost',
    user='root',
    password='your_password',
    database='hotsearch',
    charset='utf8mb4'
)

with connection.cursor() as cursor:
    # Create hot search table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS hot_search (
            id INT AUTO_INCREMENT PRIMARY KEY,
            platform VARCHAR(50) NOT NULL,
            rank_num INT,
            title VARCHAR(500),
            hot_value VARCHAR(100),
            url TEXT,
            crawl_time DATETIME,
            INDEX idx_platform (platform),
            INDEX idx_crawl_time (crawl_time)
        ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
    """)
    
    # Create detail content table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS news_detail (
            id INT AUTO_INCREMENT PRIMARY KEY,
            source_id INT,
            platform VARCHAR(50),
            title VARCHAR(500),
            content LONGTEXT,
            summary TEXT,
            sentiment VARCHAR(20),
            created_at DATETIME,
            INDEX idx_source_id (source_id)
        ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
    """)
    
    connection.commit()
```

## Configuration

### 1. Crawler Configuration

Edit `hotsearchcrawler/settings.py`:

```python
# MySQL connection
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'root'
MYSQL_PASSWORD = 'your_password'
MYSQL_DB = 'hotsearch'

# Crawler settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
ROBOTSTXT_OBEY = False

# Browser automation
SELENIUM_DRIVER_NAME = 'chrome'  # or 'edge'
SELENIUM_DRIVER_EXECUTABLE_PATH = '/usr/local/bin/chromedriver'
SELENIUM_BROWSER_EXECUTABLE_PATH = '/usr/bin/google-chrome'
```

### 2. Analysis System Configuration

Create `.env` file in project root:

```env
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch

# LLM API Configuration (OpenAI-compatible format)
OPENAI_API_KEY=your_api_key_here
OPENAI_API_BASE=https://api.openai.com/v1
LLM_MODEL=gpt-4

# Or use Huawei Pangu model locally
# PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model

# Push Notification Configuration
# WeChat Work Bot
WECHAT_WEBHOOK_URL=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxxxx

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
EMAIL_RECIPIENTS=recipient1@example.com,recipient2@example.com

# Server Configuration
FLASK_HOST=0.0.0.0
FLASK_PORT=5000
```

## Usage

### Starting the System

**1. Launch Analysis Backend**:
```bash
python app.py
```

**2. Start Crawlers** (via web interface or CLI):
```bash
# Test individual spider
python runspider-test.py

# Run all spiders
python run_spiders.py
```

**3. Access Web Interface**:
Open `http://localhost:5000` in browser

### Core Python API

#### Querying Hot Search Data

```python
from hotsearch_analysis_agent.database import get_hot_search_data

# Get latest hot searches from specific platform
results = get_hot_search_data(
    platform='weibo',
    limit=20,
    time_range='today'
)

for item in results:
    print(f"{item['rank_num']}. {item['title']} - {item['hot_value']}")
```

#### LLM-Powered Analysis

```python
from hotsearch_analysis_agent.llm_analyzer import analyze_sentiment, cluster_topics

# Sentiment analysis
content = "这个新闻真是太令人震惊了..."
sentiment = analyze_sentiment(content)
print(f"Sentiment: {sentiment['label']} ({sentiment['score']:.2f})")

# Topic clustering
news_items = [
    {"id": 1, "title": "AI技术突破", "content": "..."},
    {"id": 2, "title": "大模型应用", "content": "..."},
    {"id": 3, "title": "体育赛事", "content": "..."}
]
clusters = cluster_topics(news_items)
for cluster_id, items in clusters.items():
    print(f"Cluster {cluster_id}: {[item['title'] for item in items]}")
```

#### Content Extraction from News Pages

```python
from hotsearch_analysis_agent.content_extractor import extract_detail

# Extract content including video transcripts
url = "https://www.bilibili.com/video/BV13pSoBBEvX"
detail = extract_detail(url, platform='bilibili')

print(f"Title: {detail['title']}")
print(f"Content: {detail['content'][:200]}...")
print(f"Has Video: {detail['has_video']}")
```

#### Setting Up Push Tasks

```python
from hotsearch_analysis_agent.push_manager import create_push_task

# Create daily hot topic report task
task = create_push_task(
    name="AI Tech Daily Report",
    query_keywords=["人工智能", "大模型", "前沿科技"],
    platforms=["weibo", "zhihu", "36kr"],
    channels=["wechat", "telegram", "email"],
    schedule="0 12 * * *",  # Daily at 12:00
    min_hot_value=10000,
    sentiment_filter="all"
)

print(f"Task created: {task['id']}")
```

### Writing Custom Crawlers

Add new spider in `hotsearchcrawler/spiders/`:

```python
import scrapy
from hotsearchcrawler.items import HotSearchItem

class NewPlatformSpider(scrapy.Spider):
    name = 'newplatform'
    allowed_domains = ['newplatform.com']
    start_urls = ['https://newplatform.com/trending']
    
    custom_settings = {
        'ITEM_PIPELINES': {
            'hotsearchcrawler.pipelines.MySQLPipeline': 300,
        }
    }
    
    def parse(self, response):
        for idx, item in enumerate(response.css('.trending-item')):
            hot_item = HotSearchItem()
            hot_item['platform'] = 'newplatform'
            hot_item['rank_num'] = idx + 1
            hot_item['title'] = item.css('.title::text').get()
            hot_item['hot_value'] = item.css('.hot-value::text').get()
            hot_item['url'] = item.css('a::attr(href)').get()
            
            yield hot_item
```

## Common Patterns

### Pattern 1: Multi-Platform Topic Tracking

```python
from datetime import datetime, timedelta
from hotsearch_analysis_agent.aggregator import track_cross_platform_topic

# Track how a topic evolves across platforms
topic = "GPT-6"
start_time = datetime.now() - timedelta(days=7)

timeline = track_cross_platform_topic(
    keyword=topic,
    platforms=['weibo', 'zhihu', 'bilibili', '36kr'],
    start_time=start_time
)

for platform, entries in timeline.items():
    print(f"\n{platform.upper()}:")
    for entry in entries:
        print(f"  {entry['crawl_time']}: {entry['title']} ({entry['hot_value']})")
```

### Pattern 2: Automated Alert with Custom Rules

```python
from hotsearch_analysis_agent.alert_engine import create_alert_rule

# Alert when tech topics exceed threshold
rule = create_alert_rule(
    name="High Impact Tech News",
    conditions={
        "keywords": ["AI", "芯片", "突破"],
        "hot_value_min": 500000,
        "platforms": ["weibo", "toutiao"],
        "sentiment": ["positive", "neutral"]
    },
    actions=[
        {"type": "wechat", "template": "urgent"},
        {"type": "telegram", "mention": True}
    ],
    cooldown_minutes=60
)

# Rule runs automatically in background
```

### Pattern 3: Scheduled Report Generation

```python
from hotsearch_analysis_agent.reporter import generate_report
from apscheduler.schedulers.blocking import BlockingScheduler

def daily_report_job():
    report = generate_report(
        title="Daily AI & Tech Hot Topics",
        query={
            "keywords": ["人工智能", "大模型", "前沿科技"],
            "time_range": "24h",
            "min_rank": 50
        },
        analysis_options={
            "include_sentiment": True,
            "include_clustering": True,
            "max_clusters": 5
        },
        output_format="markdown"
    )
    
    # Push to configured channels
    report.push(channels=["wechat", "email"])

# Schedule daily at 12:00
scheduler = BlockingScheduler()
scheduler.add_job(daily_report_job, 'cron', hour=12, minute=0)
scheduler.start()
```

### Pattern 4: Video Content Analysis

```python
from hotsearch_analysis_agent.video_analyzer import analyze_video_news

# Analyze trending video news
video_url = "https://www.bilibili.com/video/BV1BFDwBZEJq"

analysis = analyze_video_news(
    url=video_url,
    extract_transcript=True,  # Extract video speech/captions
    analyze_comments=True      # Include top comments in analysis
)

print(f"Video: {analysis['title']}")
print(f"Transcript Summary: {analysis['transcript_summary']}")
print(f"Sentiment: {analysis['overall_sentiment']}")
print(f"Key Topics: {', '.join(analysis['key_topics'])}")
```

## Troubleshooting

### Issue: ChromeDriver version mismatch

**Error**: `SessionNotCreatedException: Message: session not created: This version of ChromeDriver only supports Chrome version XX`

**Solution**:
```bash
# Check Chrome version
google-chrome --version

# Download matching ChromeDriver from:
# https://chromedriver.chromium.org/downloads

# Update driver path in settings.py
```

### Issue: MySQL connection refused

**Error**: `pymysql.err.OperationalError: (2003, "Can't connect to MySQL server")`

**Solution**:
```bash
# Check MySQL is running
sudo systemctl status mysql

# Verify credentials
mysql -u root -p -e "SHOW DATABASES;"

# Update .env file with correct credentials
```

### Issue: Crawler getting blocked

**Symptoms**: 403 errors, empty results, CAPTCHAs

**Solutions**:
```python
# In hotsearchcrawler/settings.py

# 1. Increase download delay
DOWNLOAD_DELAY = 3

# 2. Enable AutoThrottle
AUTOTHROTTLE_ENABLED = True
AUTOTHROTTLE_START_DELAY = 2
AUTOTHROTTLE_MAX_DELAY = 10

# 3. Rotate user agents
DOWNLOADER_MIDDLEWARES = {
    'scrapy.downloadermiddlewares.useragent.UserAgentMiddleware': None,
    'scrapy_user_agents.middlewares.RandomUserAgentMiddleware': 400,
}

# 4. Use proxy (optional)
# Add to individual spider:
custom_settings = {
    'HTTPPROXY_ENABLED': True,
    'HTTPPROXY_AUTH_ENCODING': 'utf8'
}
```

### Issue: LLM API rate limits

**Error**: `Rate limit exceeded` or slow responses

**Solution**:
```python
# Implement request queuing in hotsearch_analysis_agent/llm_analyzer.py

import time
from functools import wraps

def rate_limit(calls_per_minute=20):
    min_interval = 60.0 / calls_per_minute
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

@rate_limit(calls_per_minute=15)
def call_llm_api(prompt):
    # Your LLM API call here
    pass
```

### Issue: Memory issues with large crawls

**Symptoms**: Process killed, slow performance

**Solution**:
```python
# Implement batch processing in pipelines

class MySQLPipeline:
    def __init__(self):
        self.items_buffer = []
        self.buffer_size = 100
    
    def process_item(self, item, spider):
        self.items_buffer.append(item)
        
        if len(self.items_buffer) >= self.buffer_size:
            self.flush_buffer()
        
        return item
    
    def flush_buffer(self):
        if not self.items_buffer:
            return
        
        # Batch insert
        self.cursor.executemany(
            "INSERT INTO hot_search (...) VALUES (%s, %s, ...)",
            [tuple(item.values()) for item in self.items_buffer]
        )
        self.connection.commit()
        self.items_buffer = []
```

## Key Commands Reference

```bash
# Start main application
python app.py

# Run all crawlers
python run_spiders.py

# Test specific spider
scrapy crawl weibo -s LOG_LEVEL=DEBUG

# Test push notifications
python test_push_task.py

# Database initialization
python init.py

# Run with custom config
python app.py --config custom_config.yml
```

## Advanced: Huawei Pangu Local Deployment

For local LLM deployment without external APIs:

```python
# Download model from:
# https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

# In .env:
USE_LOCAL_LLM=true
PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model

# Load model in llm_analyzer.py:
from transformers import AutoTokenizer, AutoModelForCausalLM

tokenizer = AutoTokenizer.from_pretrained(
    os.getenv('PANGU_MODEL_PATH'),
    trust_remote_code=True
)

model = AutoModelForCausalLM.from_pretrained(
    os.getenv('PANGU_MODEL_PATH'),
    trust_remote_code=True,
    device_map='auto'
)

def analyze_with_pangu(text):
    inputs = tokenizer(text, return_tensors="pt")
    outputs = model.generate(**inputs, max_length=512)
    return tokenizer.decode(outputs[0])
```
