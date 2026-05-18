---
name: llm-public-opinion-analytics-assistant
description: Deploy and use LLM-based public opinion analytics assistant for real-time hot search monitoring, topic clustering, sentiment analysis, and multi-channel alert pushing across 15 platforms and 26 trending lists.
triggers:
  - set up public opinion monitoring system
  - configure hot search crawler and analysis
  - implement sentiment analysis with LLM
  - deploy multi-platform trending topic tracker
  - create automated opinion analytics reports
  - setup hot topic push notifications
  - build real-time social media monitoring
  - configure chinese public opinion assistant
---

# LLM Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is an intelligent public opinion analytics assistant that combines real-time data from **15 mainstream platforms** (26 trending lists total) with LLM analysis capabilities. It provides conversational hot search queries, topic clustering, sentiment analysis, and multi-channel push notifications (WeChat, Enterprise WeChat, Telegram, Email).

Key capabilities:
- Real-time crawler cluster for 15+ platforms (Weibo, Bilibili, Douyin, Zhihu, Baidu, etc.)
- LLM-powered analysis (sentiment, clustering, summarization)
- Natural language query interface
- Video content extraction and analysis
- Multi-channel hot topic alerts
- Hotkey-controlled crawler management

## Installation

### Prerequisites

1. **Browser Driver Setup** (required for content extraction):

```bash
# Download ChromeDriver matching your Chrome version
# Place in system PATH or project directory
# Verify installation
chromedriver --version

# Or for Edge
msedgedriver --version
```

2. **Python Environment**:

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

3. **MySQL Database**:

```bash
# Create database and tables
mysql -u root -p < init.sql
# Or use the init.py reference file
python init.py
```

### Project Structure

```
├── hotsearch_analysis_agent/  # Analysis system
├── hotsearchcrawler/          # Crawler cluster (decoupled)
├── app.py                     # Main application entry
├── run_spiders.py             # Crawler launcher
├── test_push_task/            # Push notification tests
└── runspider-test/            # Crawler tests
```

## Configuration

### Environment Variables

Create `.env` file in project root:

```bash
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch_db

# OpenAI-compatible LLM API (supports Pangu, OpenAI, etc.)
LLM_API_BASE=http://your-llm-endpoint/v1
LLM_API_KEY=your_api_key
LLM_MODEL=pangu-embedded-7b

# Push Notification Channels
# WeChat Work Bot
WECHAT_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email SMTP
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
SMTP_RECIPIENTS=recipient1@example.com,recipient2@example.com
```

### Crawler Configuration

Edit `hotsearchcrawler/settings.py`:

```python
# MySQL settings for crawler data storage
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_user'
MYSQL_PASSWORD = 'your_password'
MYSQL_DATABASE = 'hotsearch_db'

# Optional: Platform-specific cookies
# For platforms requiring authentication
COOKIES = {
    'weibo': 'your_weibo_cookies',
    # Add others as needed
}

# Crawler settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
ROBOTSTXT_OBEY = False
```

## Usage

### Starting the Application

```bash
# Launch main web interface
python app.py

# Access at http://localhost:5000
```

### Crawler Management

**Via Web Interface**:
- Use hotkeys to start/stop crawlers
- Monitor real-time data collection status

**Via Command Line**:

```bash
# Test single spider
python runspider-test/test_weibo.py

# Start all crawlers
python run_spiders.py

# Start specific platform crawler
scrapy crawl weibo_hot -s JOBDIR=crawls/weibo_hot
```

### Analysis API Examples

```python
from hotsearch_analysis_agent.analyzer import HotSearchAnalyzer

# Initialize analyzer
analyzer = HotSearchAnalyzer(
    llm_api_base=os.getenv("LLM_API_BASE"),
    llm_api_key=os.getenv("LLM_API_KEY"),
    model=os.getenv("LLM_MODEL")
)

# Query hot topics
results = analyzer.query_topics(
    query="人工智能",
    platforms=["weibo", "zhihu", "bilibili"],
    time_range="24h"
)

# Perform sentiment analysis
sentiment = analyzer.analyze_sentiment(
    topic_id="12345",
    detail_level="comprehensive"
)

# Topic clustering
clusters = analyzer.cluster_topics(
    keywords=["AI", "大模型", "芯片"],
    min_cluster_size=5
)

# Generate report
report = analyzer.generate_report(
    query="前沿科技",
    include_sentiment=True,
    include_clustering=True,
    format="markdown"
)
```

### Push Notification Setup

```python
from hotsearch_analysis_agent.push import PushManager

# Initialize push manager
push_mgr = PushManager()

# Create scheduled push task
task = push_mgr.create_task(
    name="AI热点监控",
    query="人工智能 OR 大模型",
    schedule="0 9,18 * * *",  # Cron format: 9 AM and 6 PM daily
    channels=["wechat", "telegram", "email"],
    threshold={"热度": 100000}  # Only push if heat > 100k
)

# Test push immediately
push_mgr.test_push(
    task_id=task.id,
    channel="wechat"
)

# List active tasks
tasks = push_mgr.list_tasks()

# Disable task
push_mgr.disable_task(task_id)
```

### Database Query Patterns

```python
from hotsearch_analysis_agent.database import HotSearchDB

db = HotSearchDB()

# Get trending topics
trends = db.get_trending(
    platform="weibo",
    limit=50,
    time_range="1h"
)

# Search by keyword
results = db.search_topics(
    keyword="AI",
    platforms=["all"],
    start_date="2026-05-01",
    end_date="2026-05-18"
)

# Get topic details with content extraction
detail = db.get_topic_detail(
    topic_id="12345",
    extract_video=True  # Extract content even from video
)

# Topic history tracking
history = db.get_topic_history(
    topic_id="12345",
    metrics=["热度", "排名", "评论数"]
)
```

## Common Patterns

### Conversational Query Interface

```python
# Natural language queries supported
queries = [
    "显示微博今天的热搜前10",
    "搜索关于人工智能的所有话题",
    "分析最近一周科技类新闻的情感倾向",
    "找出与'芯片'相关的聚类话题",
    "推送关于'深度学习'的热点到企业微信"
]

for query in queries:
    response = analyzer.process_natural_query(query)
    print(response)
```

### Multi-Platform Data Aggregation

```python
# Aggregate data from multiple platforms
aggregated = analyzer.aggregate_platforms(
    query="GPT-6",
    platforms=["weibo", "zhihu", "douyin", "bilibili"],
    deduplicate=True,
    sort_by="heat"
)

# Cross-platform topic correlation
correlation = analyzer.find_correlations(
    topic="人工智能",
    platforms=["weibo", "zhihu"],
    time_window="7d"
)
```

### Custom Analysis Pipeline

```python
# Build custom analysis workflow
pipeline = analyzer.create_pipeline([
    ("fetch", {"platforms": ["weibo", "zhihu"], "limit": 100}),
    ("filter", {"min_heat": 50000, "keywords": ["AI", "科技"]}),
    ("sentiment", {"model": "pangu-7b"}),
    ("cluster", {"algorithm": "dbscan", "min_samples": 3}),
    ("summarize", {"max_length": 500}),
    ("push", {"channels": ["wechat"]})
])

result = pipeline.execute()
```

### Video Content Extraction

```python
# Extract content from video-based hot topics
video_analysis = analyzer.extract_video_content(
    url="https://www.bilibili.com/video/BV13pSoBBEvX",
    extract_comments=True,
    extract_transcript=True,
    sentiment_analysis=True
)

print(video_analysis["title"])
print(video_analysis["transcript"])
print(video_analysis["sentiment"])
print(video_analysis["top_comments"])
```

## Troubleshooting

### Browser Driver Issues

```bash
# Error: chromedriver not found
# Solution: Verify driver in PATH
echo $PATH  # Check if driver directory is listed
which chromedriver  # Should show path

# Error: version mismatch
# Solution: Download matching version
google-chrome --version
# Download corresponding chromedriver version
```

### Database Connection Errors

```python
# Test database connectivity
from hotsearch_analysis_agent.database import test_connection

if test_connection():
    print("Database connected successfully")
else:
    print("Check MySQL credentials in .env")
    # Verify: MYSQL_HOST, MYSQL_USER, MYSQL_PASSWORD
```

### LLM API Timeout

```python
# Increase timeout for long analysis tasks
analyzer = HotSearchAnalyzer(
    llm_api_base=os.getenv("LLM_API_BASE"),
    llm_api_key=os.getenv("LLM_API_KEY"),
    timeout=120,  # 2 minutes
    max_retries=3
)
```

### Crawler Rate Limiting

```python
# Adjust crawler settings if blocked
# In hotsearchcrawler/settings.py
DOWNLOAD_DELAY = 3  # Increase delay between requests
CONCURRENT_REQUESTS = 8  # Reduce concurrent requests
AUTOTHROTTLE_ENABLED = True
AUTOTHROTTLE_START_DELAY = 2
AUTOTHROTTLE_MAX_DELAY = 10
```

### Memory Issues with Large Datasets

```python
# Use batch processing for large queries
def process_large_dataset(query, batch_size=1000):
    offset = 0
    while True:
        batch = db.search_topics(
            keyword=query,
            limit=batch_size,
            offset=offset
        )
        if not batch:
            break
        
        # Process batch
        analyzer.analyze_batch(batch)
        offset += batch_size
```

### Push Notification Failures

```python
# Debug push channels
push_mgr.test_channel("wechat")  # Returns detailed error
push_mgr.test_channel("telegram")
push_mgr.test_channel("email")

# Check webhook validity
# WeChat: webhook must be active in WeChat Work admin
# Telegram: verify bot token with @BotFather
# Email: ensure SMTP settings and app password correct
```

## Platform Support

Supported platforms (15 total, 26 lists):
- **Weibo** (微博热搜)
- **Zhihu** (知乎热榜)
- **Bilibili** (B站热门)
- **Douyin** (抖音热榜)
- **Baidu** (百度热搜)
- **Toutiao** (头条热榜)
- **36Kr** (36氪)
- **ITHome** (IT之家)
- And more...

Each platform supports direct jump to original page from query results.
