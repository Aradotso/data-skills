---
name: llm-public-opinion-analytics-assistant
description: Multi-platform hot search crawler and LLM-powered public opinion analysis system with sentiment analysis, topic clustering, and multi-channel push notifications
triggers:
  - set up public opinion monitoring system
  - analyze hot topics from social media platforms
  - configure multi-platform crawler for trending news
  - implement sentiment analysis on hot search data
  - create automated hot topic push notifications
  - build social media opinion analysis dashboard
  - aggregate trending data from multiple platforms
  - setup LLM-based news clustering and analysis
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is a comprehensive public opinion analytics assistant that integrates real-time data from **26 ranking lists across 15 mainstream platforms** (Weibo, Bilibili, Douyin, Baidu, Zhihu, etc.) with large language model analysis capabilities. It provides conversational hot search queries, topic-specific searches, topic clustering analysis, and sentiment analysis through a web interface.

**Key capabilities:**
- Multi-platform crawler system (Scrapy-based)
- LLM-powered content analysis (supports Pangu, OpenAI-compatible APIs)
- Video content extraction and analysis
- Multi-channel push notifications (Email, WeChat Work, Telegram)
- Topic clustering and sentiment analysis
- Keyboard shortcuts for crawler control

## Installation

### Prerequisites

**1. Browser Driver Setup**

Download and configure ChromeDriver or EdgeDriver:

```bash
# Check your browser version first
# Chrome: Settings → About Chrome
# Edge: Settings → About Microsoft Edge

# Download matching driver version
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Linux/macOS - move to system path
sudo mv chromedriver /usr/local/bin/
sudo chmod +x /usr/local/bin/chromedriver

# Windows - add driver directory to PATH environment variable
# Verify installation
chromedriver --version
```

**2. MySQL Database**

```bash
# Install MySQL 5.7+ or MariaDB
# Create database
mysql -u root -p

CREATE DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

**3. Python Environment**

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

Run the initialization script to create tables:

```python
# Reference init.py for table structure
# Key tables: hot_search, news_detail, analysis_result, push_task

import pymysql

connection = pymysql.connect(
    host='localhost',
    user='your_user',
    password='your_password',
    database='hotsearch',
    charset='utf8mb4'
)

# Execute table creation SQL from init.py
```

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```bash
# Database Configuration
DB_HOST=localhost
DB_PORT=3306
DB_USER=your_user
DB_PASSWORD=your_password
DB_NAME=hotsearch

# LLM API Configuration (OpenAI-compatible)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-3.5-turbo

# Or use Huawei Pangu Model (local deployment)
# PANGU_MODEL_PATH=/path/to/pangu/model

# Email Push Configuration (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
EMAIL_RECIPIENTS=recipient1@example.com,recipient2@example.com

# WeChat Work Bot
WECHAT_WORK_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=your_key

# WeChat Work App
WECHAT_WORK_CORPID=your_corp_id
WECHAT_WORK_CORPSECRET=your_corp_secret
WECHAT_WORK_AGENTID=your_agent_id

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

### Crawler Configuration

Edit `hotsearchcrawler/settings.py`:

```python
# MySQL Pipeline Configuration
MYSQL_CONFIG = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME'),
    'charset': 'utf8mb4'
}

# Enable pipelines
ITEM_PIPELINES = {
    'hotsearchcrawler.pipelines.MySQLPipeline': 300,
}

# Optional: Platform-specific cookies (if required)
COOKIES = {
    'weibo': 'your_weibo_cookies',
    'bilibili': 'your_bilibili_cookies'
}
```

## Usage

### Starting the System

**1. Launch Web Application**

```bash
python app.py
# Access at http://localhost:5000
```

**2. Start Crawlers (via Web UI)**

Use the keyboard shortcut in the web interface to start/stop crawlers, or run manually:

```bash
# Start all spiders
python run_spiders.py

# Test single spider
python runspider-test.py --spider weibo
```

### Core Components

#### Crawler System (`hotsearchcrawler/`)

The crawler cluster is completely separated from the analysis system:

```python
# Example: Custom spider for new platform
# hotsearchcrawler/spiders/custom_platform.py

import scrapy
from hotsearchcrawler.items import HotSearchItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://platform.com/trending']
    
    def parse(self, response):
        for item in response.css('.trending-item'):
            hot_search = HotSearchItem()
            hot_search['platform'] = 'custom_platform'
            hot_search['title'] = item.css('.title::text').get()
            hot_search['url'] = item.css('a::attr(href)').get()
            hot_search['rank'] = item.css('.rank::text').get()
            hot_search['hot_value'] = item.css('.hot::text').get()
            yield hot_search
```

#### Analysis System (`hotsearch_analysis_agent/`)

**Query Hot Topics**

```python
from hotsearch_analysis_agent.agent import AnalysisAgent

agent = AnalysisAgent()

# Query latest hot searches
results = agent.query_hot_topics(
    platform='weibo',
    limit=10,
    time_range='24h'
)

for topic in results:
    print(f"{topic['rank']}. {topic['title']} - {topic['hot_value']}")
```

**Sentiment Analysis**

```python
# Analyze sentiment for specific topic
sentiment = agent.analyze_sentiment(
    topic_id=12345,
    include_comments=True
)

print(f"Overall Sentiment: {sentiment['overall']}")
print(f"Positive: {sentiment['positive_ratio']}%")
print(f"Negative: {sentiment['negative_ratio']}%")
print(f"Neutral: {sentiment['neutral_ratio']}%")
```

**Topic Clustering**

```python
# Cluster related topics
clusters = agent.cluster_topics(
    keywords=['人工智能', 'AI', '大模型'],
    time_range='7d',
    min_cluster_size=3
)

for cluster in clusters:
    print(f"\nCluster: {cluster['theme']}")
    print(f"Topics: {len(cluster['topics'])}")
    for topic in cluster['topics']:
        print(f"  - {topic['title']}")
```

**LLM-Powered Analysis**

```python
# Generate comprehensive analysis report
report = agent.generate_report(
    query="人工智能与前沿科技",
    platforms=['weibo', 'zhihu', 'bilibili'],
    analysis_depth='detailed',
    include_video_content=True
)

print(report['summary'])
print(report['key_findings'])
print(report['sentiment_trend'])
```

### Push Notification System

**Configure Push Task**

```python
from hotsearch_analysis_agent.push import PushTaskManager

push_manager = PushTaskManager()

# Create scheduled push task
task = push_manager.create_task(
    name="AI Technology Daily Report",
    query_keywords=['人工智能', '大模型', '芯片'],
    schedule='daily',  # daily, hourly, weekly
    schedule_time='12:00',
    channels=['email', 'wechat_work', 'telegram'],
    analysis_enabled=True,
    cluster_enabled=True
)

# Start task
push_manager.start_task(task.id)
```

**Test Push Channels**

```bash
# Run push task tests
python test_push_task.py --channel email
python test_push_task.py --channel wechat_work
python test_push_task.py --channel telegram
```

**Manual Push**

```python
from hotsearch_analysis_agent.push import PushService

push_service = PushService()

# Email push
push_service.send_email(
    subject="Hot Topic Alert",
    content=report_html,
    recipients=['user@example.com']
)

# WeChat Work Robot
push_service.send_wechat_robot(
    content=report_markdown,
    mentioned_list=['@all']
)

# Telegram
push_service.send_telegram(
    message=report_text,
    parse_mode='Markdown'
)
```

## Common Patterns

### Real-Time Monitoring Dashboard

```python
# app.py - Flask integration example
from flask import Flask, jsonify, render_template
from hotsearch_analysis_agent.agent import AnalysisAgent

app = Flask(__name__)
agent = AnalysisAgent()

@app.route('/api/hot-topics')
def get_hot_topics():
    platform = request.args.get('platform', 'weibo')
    limit = int(request.args.get('limit', 20))
    
    topics = agent.query_hot_topics(
        platform=platform,
        limit=limit,
        time_range='1h'
    )
    return jsonify(topics)

@app.route('/api/analyze')
def analyze_topic():
    topic_id = request.args.get('topic_id')
    
    analysis = agent.analyze_topic(
        topic_id=topic_id,
        include_sentiment=True,
        include_clustering=True,
        fetch_detail_content=True
    )
    return jsonify(analysis)
```

### Custom Analysis Workflow

```python
# Combine multiple analysis steps
class CustomAnalysisPipeline:
    def __init__(self):
        self.agent = AnalysisAgent()
    
    def run_analysis(self, keywords, platforms):
        # Step 1: Fetch hot topics
        topics = []
        for platform in platforms:
            platform_topics = self.agent.query_hot_topics(
                platform=platform,
                keywords=keywords,
                time_range='24h'
            )
            topics.extend(platform_topics)
        
        # Step 2: Cluster related topics
        clusters = self.agent.cluster_topics(topics)
        
        # Step 3: Sentiment analysis per cluster
        for cluster in clusters:
            cluster['sentiment'] = self.agent.analyze_sentiment(
                topic_ids=[t['id'] for t in cluster['topics']]
            )
        
        # Step 4: Generate comprehensive report
        report = self.agent.generate_report(
            clusters=clusters,
            include_charts=True,
            output_format='html'
        )
        
        return report

# Usage
pipeline = CustomAnalysisPipeline()
report = pipeline.run_analysis(
    keywords=['科技', '创新'],
    platforms=['weibo', 'zhihu', 'toutiao']
)
```

### Video Content Analysis

```python
# Extract and analyze video-based trending topics
video_topics = agent.query_hot_topics(
    platform='bilibili',
    content_type='video',
    limit=10
)

for topic in video_topics:
    # Fetch video metadata and subtitles
    detail = agent.fetch_video_detail(topic['url'])
    
    # Analyze video content using LLM
    analysis = agent.analyze_video_content(
        title=detail['title'],
        description=detail['description'],
        subtitles=detail['subtitles'],
        comments=detail['comments'][:100]
    )
    
    print(f"Video: {detail['title']}")
    print(f"Main Topic: {analysis['main_topic']}")
    print(f"Sentiment: {analysis['sentiment']}")
    print(f"Key Points: {analysis['key_points']}")
```

## Troubleshooting

### Crawler Issues

**Problem: Browser driver not found**

```bash
# Verify driver is in PATH
which chromedriver  # Linux/macOS
where chromedriver  # Windows

# Add to PATH if missing
export PATH=$PATH:/path/to/driver  # Add to ~/.bashrc or ~/.zshrc
```

**Problem: Platform blocking requests**

```python
# Add delays and retry logic in settings.py
DOWNLOAD_DELAY = 3
RANDOMIZE_DOWNLOAD_DELAY = True
RETRY_TIMES = 5

# Use rotating user agents
DOWNLOADER_MIDDLEWARES = {
    'scrapy.downloadermiddlewares.useragent.UserAgentMiddleware': None,
    'scrapy_user_agents.middlewares.RandomUserAgentMiddleware': 400,
}
```

**Problem: Cookie expiration**

```python
# Implement cookie refresh mechanism
def refresh_cookies(platform):
    # Use selenium to login and get fresh cookies
    from selenium import webdriver
    driver = webdriver.Chrome()
    driver.get(f'https://{platform}.com/login')
    # ... login process ...
    cookies = driver.get_cookies()
    return cookies
```

### Database Issues

**Problem: Connection pool exhausted**

```python
# Increase pool size in settings
MYSQL_CONFIG = {
    'pool_size': 20,
    'max_overflow': 10,
    'pool_recycle': 3600
}
```

**Problem: Character encoding errors**

```sql
-- Ensure database uses utf8mb4
ALTER DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
ALTER TABLE hot_search CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### LLM Analysis Issues

**Problem: API rate limiting**

```python
# Implement request throttling
from time import sleep
import functools

def rate_limit(calls_per_minute=10):
    min_interval = 60.0 / calls_per_minute
    def decorator(func):
        last_called = [0.0]
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            wait = min_interval - elapsed
            if wait > 0:
                sleep(wait)
            result = func(*args, **kwargs)
            last_called[0] = time.time()
            return result
        return wrapper
    return decorator

@rate_limit(calls_per_minute=20)
def call_llm_api(prompt):
    # API call logic
    pass
```

**Problem: Context length exceeded**

```python
# Implement chunking for long texts
def analyze_long_content(content, max_tokens=3000):
    chunks = split_into_chunks(content, max_tokens)
    summaries = []
    
    for chunk in chunks:
        summary = agent.analyze_text(chunk)
        summaries.append(summary)
    
    # Synthesize final analysis
    final_analysis = agent.synthesize_summaries(summaries)
    return final_analysis
```

### Push Notification Issues

**Problem: Email delivery failures**

```python
# Use TLS/SSL properly
import smtplib
from email.mime.text import MIMEText

server = smtplib.SMTP(SMTP_HOST, SMTP_PORT)
server.starttls()  # Enable TLS
server.login(SMTP_USER, SMTP_PASSWORD)

# For Gmail, use app-specific password instead of account password
```

**Problem: WeChat Work webhook signature invalid**

```python
# Ensure correct content format
import json
import requests

def send_wechat_message(webhook_url, content):
    payload = {
        "msgtype": "markdown",
        "markdown": {
            "content": content
        }
    }
    response = requests.post(
        webhook_url,
        json=payload,
        headers={'Content-Type': 'application/json'}
    )
    return response.json()
```

## Advanced Usage

### Custom LLM Integration

```python
# Use local Pangu model instead of OpenAI API
from hotsearch_analysis_agent.llm import PanguModel

class CustomAgent(AnalysisAgent):
    def __init__(self):
        super().__init__()
        self.llm = PanguModel(
            model_path=os.getenv('PANGU_MODEL_PATH'),
            device='cuda'  # or 'cpu'
        )
    
    def analyze_with_pangu(self, prompt):
        response = self.llm.generate(
            prompt=prompt,
            max_length=2048,
            temperature=0.7,
            top_p=0.9
        )
        return response
```

### Batch Processing

```python
# Process multiple queries in parallel
from concurrent.futures import ThreadPoolExecutor

def batch_analyze(queries, max_workers=5):
    agent = AnalysisAgent()
    
    def analyze_single(query):
        return agent.analyze_topic(query)
    
    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        results = list(executor.map(analyze_single, queries))
    
    return results

# Usage
queries = ['AI技术', '芯片产业', '新能源汽车']
results = batch_analyze(queries)
```

This skill provides comprehensive coverage of the LLM-Based Public Opinion Analytics Assistant, enabling AI coding agents to effectively assist developers in deploying, configuring, and extending this multi-platform social media monitoring and analysis system.
