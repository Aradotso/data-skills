---
name: llm-public-opinion-analytics-assistant
description: Chinese multi-platform hot search aggregator with LLM-powered sentiment analysis, topic clustering, and multi-channel alert pushing
triggers:
  - how do I set up a public opinion monitoring system
  - help me aggregate hot topics from Chinese social platforms
  - how to analyze sentiment trends across Weibo and Bilibili
  - set up automated hot topic alerts with LLM analysis
  - how do I configure multi-platform crawler for Chinese news
  - build a public opinion analytics dashboard with clustering
  - how to push hot topic reports to WeChat and Telegram
  - integrate Huawei Pangu model for sentiment analysis
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is a comprehensive Chinese public opinion monitoring system that aggregates real-time data from **15 mainstream platforms** across **26 hot search lists**. It combines web crawling, LLM-powered analysis, and multi-channel alerting to provide:

- **Real-time hot topic aggregation** from Weibo, Bilibili, Douyin, Zhihu, and 11+ other Chinese platforms
- **Conversational query interface** for natural language topic search and filtering
- **LLM-powered analysis** including topic clustering, sentiment analysis, and trend reports
- **Multi-channel push notifications** to Enterprise WeChat, Telegram, and email
- **Video content extraction** capability for analyzing video-based news

The architecture separates the crawler cluster (`hotsearchcrawler`) from the analysis system (`hotsearch_analysis_agent`), storing data in MySQL for robust querying.

## Installation

### Prerequisites

1. **Browser Driver Setup** (required for news detail extraction):

```bash
# Check your Chrome/Edge version first
google-chrome --version  # or microsoft-edge --version

# Download matching ChromeDriver from:
# https://chromedriver.chromium.org/

# macOS/Linux - move to PATH
sudo mv chromedriver /usr/local/bin/
sudo chmod +x /usr/local/bin/chromedriver

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
python3 -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Configuration

1. **Database Initialization**:

```python
# Reference init.py to create tables
# Key tables: hot_topics, news_details, analysis_results, push_tasks

# Run initialization
python init.py
```

2. **Environment Variables** (`.env` file):

```bash
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch_db

# LLM API Configuration (OpenAI-compatible format)
LLM_API_KEY=your_api_key
LLM_API_BASE=https://api.openai.com/v1
LLM_MODEL=gpt-4

# Or use Huawei Pangu Model (local deployment)
# PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model

# Push Notification Channels
# Enterprise WeChat Robot
WECHAT_WEBHOOK_URL=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=your_key

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email SMTP
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
```

3. **Crawler Settings** (`hotsearchcrawler/settings.py`):

```python
# MySQL connection for crawler
MYSQL_CONFIG = {
    'host': 'localhost',
    'port': 3306,
    'user': 'root',
    'password': 'your_password',
    'database': 'hotsearch_db'
}

# Optional: Platform-specific cookies
COOKIES = {
    'weibo': 'your_weibo_cookies',
    'bilibili': 'your_bilibili_cookies'
}

# Crawler concurrency
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 0.5
```

## Key Commands

### Starting the System

```bash
# 1. Start the web application (analysis system + frontend)
python app.py
# Default: http://localhost:5000

# 2. Start crawler cluster (can be triggered from frontend or manually)
python run_spiders.py

# 3. Test individual spider
python runspider-test.py --spider weibo
```

### Testing Push Notifications

```bash
# Test all configured push channels
python test_push_task.py

# Test specific channel
python test_push_task.py --channel wechat
python test_push_task.py --channel telegram
python test_push_task.py --channel email
```

## Core Usage Patterns

### 1. Querying Hot Topics via API

```python
from hotsearch_analysis_agent.database import get_db_connection

def query_hot_topics(platform=None, keyword=None, limit=50):
    """Query hot topics from database"""
    conn = get_db_connection()
    cursor = conn.cursor(dictionary=True)
    
    query = "SELECT * FROM hot_topics WHERE 1=1"
    params = []
    
    if platform:
        query += " AND platform = %s"
        params.append(platform)
    
    if keyword:
        query += " AND title LIKE %s"
        params.append(f"%{keyword}%")
    
    query += " ORDER BY rank ASC, crawl_time DESC LIMIT %s"
    params.append(limit)
    
    cursor.execute(query, params)
    results = cursor.fetchall()
    cursor.close()
    conn.close()
    
    return results

# Example usage
weibo_topics = query_hot_topics(platform='weibo', limit=10)
ai_topics = query_hot_topics(keyword='人工智能', limit=20)
```

### 2. LLM-Powered Topic Clustering

```python
from hotsearch_analysis_agent.llm_analyzer import TopicAnalyzer

def analyze_topics_with_clustering(topics, query="人工智能"):
    """Cluster topics and generate analysis report"""
    analyzer = TopicAnalyzer(
        api_key=os.getenv('LLM_API_KEY'),
        api_base=os.getenv('LLM_API_BASE'),
        model=os.getenv('LLM_MODEL')
    )
    
    # Prepare topic data
    topic_texts = [
        f"{t['title']} - {t['platform']} (热度:{t['heat']})"
        for t in topics
    ]
    
    # Perform clustering and analysis
    result = analyzer.cluster_and_analyze(
        topics=topic_texts,
        query=query,
        num_clusters=4
    )
    
    return {
        'clusters': result['clusters'],
        'summary': result['summary'],
        'sentiment': result['sentiment_distribution']
    }

# Example usage
topics = query_hot_topics(keyword='AI', limit=100)
analysis = analyze_topics_with_clustering(topics, query="AI技术进展")
print(analysis['summary'])
```

### 3. Sentiment Analysis

```python
from hotsearch_analysis_agent.llm_analyzer import SentimentAnalyzer

def analyze_sentiment_trends(topic_title):
    """Analyze sentiment for a specific topic across platforms"""
    analyzer = SentimentAnalyzer()
    
    # Get related news details
    conn = get_db_connection()
    cursor = conn.cursor(dictionary=True)
    cursor.execute("""
        SELECT content, platform, created_at 
        FROM news_details 
        WHERE topic_title = %s
    """, (topic_title,))
    news_items = cursor.fetchall()
    cursor.close()
    conn.close()
    
    # Analyze each item
    sentiments = []
    for item in news_items:
        sentiment = analyzer.analyze(item['content'])
        sentiments.append({
            'platform': item['platform'],
            'sentiment': sentiment,  # 'positive', 'negative', 'neutral'
            'time': item['created_at']
        })
    
    return sentiments
```

### 4. Setting Up Push Tasks

```python
from hotsearch_analysis_agent.push_manager import PushTaskManager

def create_monitoring_task(keywords, channels=['wechat', 'telegram']):
    """Create automated monitoring and push task"""
    manager = PushTaskManager()
    
    task_config = {
        'name': f'监控任务: {", ".join(keywords)}',
        'keywords': keywords,
        'platforms': ['weibo', 'zhihu', 'bilibili'],  # Target platforms
        'channels': channels,
        'frequency': '1h',  # Check every hour
        'threshold': {
            'min_heat': 100000,  # Minimum heat score
            'sentiment_alert': 'negative'  # Alert on negative sentiment
        },
        'report_format': 'detailed'  # or 'brief'
    }
    
    task_id = manager.create_task(task_config)
    manager.start_task(task_id)
    
    return task_id

# Example: Monitor AI-related negative sentiment
task_id = create_monitoring_task(
    keywords=['人工智能', 'AI', '机器学习'],
    channels=['wechat', 'email']
)
```

### 5. Extracting Video News Content

```python
from hotsearchcrawler.utils.content_extractor import VideoContentExtractor

def extract_video_news(url, platform='bilibili'):
    """Extract text content from video news (comments, description, subtitles)"""
    extractor = VideoContentExtractor()
    
    content = extractor.extract(url, platform)
    
    return {
        'title': content.get('title'),
        'description': content.get('description'),
        'comments': content.get('top_comments', []),
        'subtitle_text': content.get('subtitles'),
        'play_count': content.get('play_count'),
        'like_count': content.get('like_count')
    }

# Example usage
bilibili_url = "https://www.bilibili.com/video/BV13pSoBBEvX"
video_content = extract_video_news(bilibili_url, 'bilibili')
```

### 6. Generating HTML Reports

```python
from hotsearch_analysis_agent.report_generator import ReportGenerator

def generate_daily_report(date, topic_query="人工智能与前沿科技"):
    """Generate formatted HTML report for push notifications"""
    generator = ReportGenerator()
    
    # Query topics for the date
    topics = query_topics_by_date(date, keyword=topic_query)
    
    # Analyze and cluster
    analysis = analyze_topics_with_clustering(topics, query=topic_query)
    
    # Generate report
    report = generator.create_report(
        title=f"关于{topic_query}的热点分析",
        date=date,
        clusters=analysis['clusters'],
        summary=analysis['summary'],
        sentiment=analysis['sentiment'],
        news_items=topics[:20]  # Top 20 news
    )
    
    return report  # Returns HTML string

# Example usage
from datetime import datetime
today = datetime.now().strftime('%Y-%m-%d')
html_report = generate_daily_report(today, "AI技术")
```

## Configuration for Huawei Pangu Model

For local deployment with Huawei's Pangu model instead of OpenAI API:

```python
# In hotsearch_analysis_agent/llm_analyzer.py

from transformers import AutoModelForCausalLM, AutoTokenizer

class PanguAnalyzer:
    def __init__(self, model_path):
        self.tokenizer = AutoTokenizer.from_pretrained(model_path)
        self.model = AutoModelForCausalLM.from_pretrained(
            model_path,
            device_map="auto",
            torch_dtype="auto"
        )
    
    def analyze(self, prompt, max_length=2048):
        inputs = self.tokenizer(prompt, return_tensors="pt").to(self.model.device)
        outputs = self.model.generate(
            **inputs,
            max_length=max_length,
            temperature=0.7,
            top_p=0.9
        )
        return self.tokenizer.decode(outputs[0], skip_special_tokens=True)

# Usage
analyzer = PanguAnalyzer(
    model_path="/path/to/openpangu-embedded-7b-model"
)
result = analyzer.analyze("分析以下话题的舆情倾向：...")
```

Download Pangu model from: https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

## Troubleshooting

### ChromeDriver Version Mismatch

```bash
# Error: SessionNotCreatedException: Message: session not created: 
# This version of ChromeDriver only supports Chrome version X

# Solution: Download exact matching version
google-chrome --version  # Check version first
# Download from https://chromedriver.chromium.org/downloads
```

### MySQL Connection Errors

```python
# Error: Can't connect to MySQL server

# Check MySQL is running
sudo systemctl status mysql  # Linux
# brew services list  # macOS

# Verify credentials
mysql -u root -p -h localhost

# Update .env file with correct credentials
```

### Crawler Not Collecting Data

```python
# Check spider logs
tail -f hotsearchcrawler/logs/spider.log

# Test individual spider
scrapy crawl weibo -s LOG_LEVEL=DEBUG

# Verify platform cookies if needed (some platforms require authentication)
# Update COOKIES in hotsearchcrawler/settings.py
```

### Push Notifications Not Working

```bash
# Test push channels individually
python test_push_task.py --channel wechat --debug

# Verify webhook URLs and tokens
# Enterprise WeChat: Check webhook key in WECHAT_WEBHOOK_URL
# Telegram: Verify TELEGRAM_BOT_TOKEN and TELEGRAM_CHAT_ID
# Email: Test SMTP credentials with telnet
```

### LLM API Rate Limiting

```python
# Add retry logic and backoff
import time
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=4, max=10))
def call_llm_with_retry(prompt):
    analyzer = TopicAnalyzer()
    return analyzer.analyze(prompt)

# Or switch to local Pangu model to avoid API limits
```

### Memory Issues with Large Datasets

```python
# Use pagination for large queries
def query_topics_paginated(page_size=1000):
    offset = 0
    while True:
        topics = query_hot_topics_with_offset(limit=page_size, offset=offset)
        if not topics:
            break
        yield topics
        offset += page_size

# Process in batches
for batch in query_topics_paginated():
    process_batch(batch)
```

## Common Workflow Example

Complete workflow for setting up daily AI news monitoring:

```python
import os
from datetime import datetime, timedelta
from hotsearch_analysis_agent import *

def setup_ai_monitoring():
    """Complete workflow: crawl → analyze → report → push"""
    
    # 1. Start crawlers for target platforms
    from hotsearchcrawler.runner import SpiderRunner
    runner = SpiderRunner()
    runner.start_spiders(['weibo', 'zhihu', 'bilibili', 'toutiao'])
    
    # 2. Wait for data collection (or schedule via cron)
    time.sleep(300)  # 5 minutes
    
    # 3. Query collected topics
    topics = query_hot_topics(
        keyword='人工智能|AI|机器学习',
        limit=200
    )
    
    # 4. Perform LLM analysis
    analysis = analyze_topics_with_clustering(
        topics,
        query="人工智能技术进展与应用"
    )
    
    # 5. Generate report
    report = generate_daily_report(
        date=datetime.now().strftime('%Y-%m-%d'),
        topic_query="人工智能",
        clusters=analysis['clusters'],
        summary=analysis['summary']
    )
    
    # 6. Push to configured channels
    push_manager = PushTaskManager()
    push_manager.send_report(
        report=report,
        channels=['wechat', 'telegram', 'email'],
        recipients=['team@example.com']
    )
    
    print(f"✅ Monitoring cycle completed at {datetime.now()}")

# Run once or schedule with cron
# 0 */6 * * * /path/to/venv/bin/python /path/to/script.py
setup_ai_monitoring()
```

## Platform Support

Supported platforms (as of 2026):
- **Social**: Weibo, Zhihu, Douban
- **Video**: Bilibili, Douyin, Kuaishou
- **News**: Toutiao, Sina News, NetEase News
- **Tech**: 36Kr, IFanr, CSDN
- **Finance**: Xueqiu, JRJ
- **E-commerce**: Taobao Hot Search

Each platform spider can be enabled/disabled in `run_spiders.py`.
