---
name: llm-public-opinion-analytics-assistant
description: A Chinese-language public opinion analytics system combining 15 platforms, 26 hot search lists, web crawlers, LLM analysis, and multi-channel push notifications
triggers:
  - set up public opinion monitoring system
  - analyze Chinese social media trends
  - crawl hot search rankings from Weibo Bilibili
  - configure sentiment analysis with LLM
  - push trending topics to WeChat Telegram
  - cluster and analyze news topics
  - deploy opinion analytics assistant
  - integrate Pangu model for sentiment analysis
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is a comprehensive public opinion analytics system designed for Chinese platforms. It integrates:

- **15 mainstream platforms** with **26 hot search lists** (Weibo, Bilibili, Douyin, Zhihu, Baidu, etc.)
- **Distributed crawler cluster** (Scrapy-based) for real-time data collection
- **LLM-powered analysis** (supports OpenAI API format, recommends Huawei Pangu model)
- **Conversational query interface** for hot search, topic clustering, sentiment analysis
- **Multi-channel push notifications** (WeChat Work, Telegram, Email)
- **Video content extraction** from news detail pages

The system separates crawler (`hotsearchcrawler`) and analysis (`hotsearch_analysis_agent`) components for independent scaling.

## Installation

### Prerequisites

**Browser Driver Setup** (required for news detail scraping):

1. Install Chrome or Edge browser
2. Check browser version: `Settings` → `About`
3. Download matching driver:
   - Chrome: https://chromedriver.chromium.org/
   - Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/
4. Place driver in system PATH or project directory
5. Verify: `chromedriver --version`

**Database Setup**:

```bash
# Install MySQL
mysql -u root -p

# Create database (see init.py for full schema)
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

**Python Environment**:

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

## Configuration

### 1. Crawler Settings (`hotsearchcrawler/settings.py`)

```python
# MySQL Configuration
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_user'
MYSQL_PASSWORD = 'your_password'
MYSQL_DATABASE = 'hotsearch_db'

# Optional: Platform Cookies (for authenticated content)
COOKIES_WEIBO = {
    'SUB': 'your_weibo_cookie',
    # ... additional cookies
}
```

### 2. Analysis System Settings (`.env`)

```bash
# MySQL
DB_HOST=localhost
DB_PORT=3306
DB_USER=your_user
DB_PASSWORD=your_password
DB_NAME=hotsearch_db

# LLM API (OpenAI-compatible format)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-4  # or local Pangu model endpoint

# Push Notifications (optional)
WECHAT_WORK_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
```

### 3. Database Initialization

```python
# Reference init.py for table schemas
# Key tables: hot_search_records, news_details, analysis_results, push_tasks

from hotsearch_analysis_agent.database import init_database

init_database()  # Creates tables if not exists
```

## Core Usage

### Starting the Crawler Cluster

```python
# Method 1: Via Web Interface (recommended)
python app.py
# Navigate to http://localhost:5000
# Use keyboard shortcuts to start/stop crawlers

# Method 2: Direct CLI
cd hotsearchcrawler
python run_spiders.py --platforms weibo bilibili zhihu

# Method 3: Test single spider
python runspider-test.py --spider weibo
```

### Analysis System API

```python
from hotsearch_analysis_agent import AnalysisAgent

# Initialize agent
agent = AnalysisAgent(
    model_name="gpt-4",  # or Pangu model endpoint
    db_config={
        "host": "localhost",
        "user": "root",
        "password": "password",
        "database": "hotsearch_db"
    }
)

# Query hot search rankings
results = agent.query_hot_search(
    platform="weibo",
    limit=20,
    time_range="today"
)

# Topic-based search
topics = agent.search_topics(
    keywords="人工智能 AI",
    platforms=["weibo", "zhihu", "bilibili"]
)

# Clustering analysis
clusters = agent.cluster_topics(
    time_range="last_24h",
    min_cluster_size=3
)

# Sentiment analysis
sentiment = agent.analyze_sentiment(
    topic_id="topic_12345",
    include_comments=True
)
```

### Conversational Query Interface

```python
# Natural language queries via web UI or API
user_query = "最近关于人工智能的热门话题有哪些?"

response = agent.conversational_query(user_query)
# Returns: Structured response with rankings, trends, sentiment
```

### Push Notification Tasks

```python
from hotsearch_analysis_agent.push import PushTaskManager

# Create push task
push_manager = PushTaskManager()

task = push_manager.create_task(
    name="AI热点监控",
    query="人工智能 AND 前沿科技",
    schedule="0 9,18 * * *",  # Cron format: 9 AM & 6 PM daily
    channels=["wechat", "telegram", "email"],
    threshold={
        "min_heat": 1000,  # Minimum heat score
        "sentiment_filter": "all"  # or "positive", "negative"
    }
)

# Test push (see test_push_task.py)
push_manager.test_push(
    task_id=task.id,
    channels=["telegram"]
)
```

## Common Patterns

### Pattern 1: Real-Time Hot Search Monitoring

```python
import asyncio
from hotsearch_analysis_agent import HotSearchMonitor

async def monitor_trends():
    monitor = HotSearchMonitor(platforms=["weibo", "douyin"])
    
    async for event in monitor.stream_updates(interval=300):  # 5-min intervals
        if event.heat_score > 5000:
            # Trigger alert
            await push_manager.send_alert(
                title=f"突发热点: {event.title}",
                content=event.summary,
                channels=["wechat"]
            )

asyncio.run(monitor_trends())
```

### Pattern 2: Topic Clustering & Report Generation

```python
from hotsearch_analysis_agent import ReportGenerator

# Analyze and generate report
report = ReportGenerator(
    time_range="2026-04-07 00:00:00 to 2026-04-07 23:59:59",
    topics_filter="人工智能|芯片|大模型"
)

markdown_report = report.generate(
    include_clusters=True,
    include_sentiment=True,
    include_source_links=True
)

# Output format matches example in README
with open("report_20260407.md", "w", encoding="utf-8") as f:
    f.write(markdown_report)
```

### Pattern 3: Local Pangu Model Integration

```python
# For privacy-sensitive deployments
from hotsearch_analysis_agent.models import PanguModel

# Download model from https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model
pangu = PanguModel(
    model_path="/path/to/openpangu-embedded-7b",
    device="cuda"  # or "cpu", "npu" (Ascend)
)

# Use in analysis agent
agent = AnalysisAgent(
    model=pangu,
    db_config=db_config
)

# Long text analysis
result = agent.analyze_article(
    content=long_article_text,
    task="sentiment_and_summary"
)
```

### Pattern 4: Custom Platform Spider

```python
# Add new platform to hotsearchcrawler/spiders/
import scrapy

class CustomPlatformSpider(scrapy.Spider):
    name = "custom_platform"
    
    def start_requests(self):
        yield scrapy.Request(
            url="https://custom-platform.com/hot",
            callback=self.parse
        )
    
    def parse(self, response):
        for item in response.css(".hot-item"):
            yield {
                "platform": "custom_platform",
                "title": item.css(".title::text").get(),
                "url": item.css("a::attr(href)").get(),
                "heat_score": int(item.css(".heat::text").get()),
                "timestamp": datetime.now()
            }
```

## Troubleshooting

### Issue: ChromeDriver Version Mismatch

```bash
# Error: "ChromeDriver only supports Chrome version X"
# Solution: Update driver to match browser version
wget https://chromedriver.storage.googleapis.com/LATEST_RELEASE
# Download matching version
```

### Issue: MySQL Connection Refused

```python
# Verify connection parameters
import pymysql

try:
    conn = pymysql.connect(
        host=os.getenv("DB_HOST"),
        user=os.getenv("DB_USER"),
        password=os.getenv("DB_PASSWORD"),
        database=os.getenv("DB_NAME")
    )
    print("Connected successfully")
except Exception as e:
    print(f"Connection failed: {e}")
```

### Issue: Empty Crawler Results

```python
# Check spider logs
cd hotsearchcrawler
scrapy crawl weibo -L DEBUG

# Common causes:
# 1. Missing cookies for authenticated content
# 2. Platform anti-crawling measures (add delays in settings.py)
# 3. Selector changes (update CSS/XPath in spider)
```

### Issue: LLM API Rate Limits

```python
# Implement rate limiting
from hotsearch_analysis_agent.utils import RateLimiter

limiter = RateLimiter(max_requests=10, time_window=60)  # 10 req/min

@limiter.limit
def call_llm_api(prompt):
    return agent.model.generate(prompt)
```

### Issue: Video Content Not Extracted

```bash
# Ensure ffmpeg installed for video processing
sudo apt-get install ffmpeg  # Linux
brew install ffmpeg  # macOS

# Verify in Python
import subprocess
subprocess.run(["ffmpeg", "-version"])
```

## Advanced Configuration

### Multi-Instance Crawler Deployment

```yaml
# docker-compose.yml
version: '3.8'
services:
  crawler-weibo:
    build: ./hotsearchcrawler
    environment:
      - SPIDER_NAME=weibo
      - MYSQL_HOST=db
    depends_on:
      - db
  
  crawler-bilibili:
    build: ./hotsearchcrawler
    environment:
      - SPIDER_NAME=bilibili
      - MYSQL_HOST=db
    depends_on:
      - db
  
  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
      MYSQL_DATABASE: hotsearch_db
```

### Custom Analysis Workflow

```python
# Chain multiple analysis steps
from hotsearch_analysis_agent import Pipeline

pipeline = Pipeline([
    ("fetch", FetchStep(platforms=["weibo", "zhihu"])),
    ("filter", FilterStep(min_heat=1000)),
    ("cluster", ClusterStep(algorithm="dbscan")),
    ("sentiment", SentimentStep(model="pangu")),
    ("summarize", SummarizeStep(max_length=500)),
    ("push", PushStep(channels=["telegram"]))
])

results = pipeline.run(query="AI芯片")
```

This skill enables AI agents to help developers deploy and customize a production-ready Chinese public opinion monitoring system with LLM-powered analytics.
