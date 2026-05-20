---
name: llm-public-opinion-analytics-assistant
description: AI-powered public opinion monitoring system that crawls 26 trending lists from 15 platforms, performs sentiment analysis, topic clustering, and multi-channel alerts
triggers:
  - "set up public opinion monitoring system"
  - "crawl trending topics from multiple platforms"
  - "analyze sentiment and cluster hot topics"
  - "configure multi-platform news crawler"
  - "send trending alerts to WeChat or Telegram"
  - "deploy Chinese social media sentiment analysis"
  - "integrate Pangu LLM for opinion analytics"
  - "monitor Weibo Douyin Bilibili trending topics"
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive Chinese social media monitoring and analysis system that combines real-time web crawling from 15 mainstream platforms (26 trending lists) with large language model capabilities. Supports conversational queries, topic clustering, sentiment analysis, and multi-channel push notifications (WeChat, Telegram, Email).

## What This Project Does

- **Multi-Platform Data Crawling**: Automatically scrapes trending topics from Weibo, Douyin, Bilibili, Zhihu, Baidu, and 10+ other Chinese platforms
- **LLM-Powered Analysis**: Uses Huawei Pangu model (or OpenAI-compatible APIs) for sentiment analysis, topic clustering, and trend summarization
- **Conversational Interface**: Natural language queries for trending topics, specific themes, and historical analysis
- **Content Extraction**: Extracts full article/video content from detail pages for deep analysis
- **Multi-Channel Alerts**: Push reports via Enterprise WeChat, Telegram, or email based on custom rules
- **Hotkey Control**: Start/stop crawlers with keyboard shortcuts from the frontend

## Installation

### Prerequisites

1. **Browser Driver Setup** (required for content extraction):

```bash
# Chrome/Chromium users
# Download ChromeDriver matching your Chrome version from:
# https://chromedriver.chromium.org/

# Edge users
# Download EdgeDriver from:
# https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Place driver in system PATH or project root
# Verify installation:
chromedriver --version  # or msedgedriver --version
```

2. **MySQL Database**:

```bash
# Install MySQL 5.7+ and create database
mysql -u root -p
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

3. **Python Environment**:

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Database Initialization

```python
# Reference init.py to create tables
# Key tables: hot_searches, news_details, analysis_results, push_tasks

# Example table structure:
"""
CREATE TABLE hot_searches (
    id INT PRIMARY KEY AUTO_INCREMENT,
    platform VARCHAR(50),
    title VARCHAR(500),
    url VARCHAR(1000),
    hot_value VARCHAR(100),
    rank INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE news_details (
    id INT PRIMARY KEY AUTO_INCREMENT,
    search_id INT,
    content TEXT,
    summary TEXT,
    sentiment VARCHAR(50),
    FOREIGN KEY (search_id) REFERENCES hot_searches(id)
);
"""
```

## Configuration

### Environment Variables

Create `.env` file in project root:

```bash
# Database Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch_db

# LLM Configuration (Pangu or OpenAI-compatible)
LLM_API_BASE=http://localhost:8000/v1  # Pangu local deployment
LLM_API_KEY=your_api_key_or_token
LLM_MODEL=openpangu-embedded-7b

# Alternative: OpenAI API
# LLM_API_BASE=https://api.openai.com/v1
# LLM_API_KEY=sk-...
# LLM_MODEL=gpt-4

# Push Notification Channels
WEWORK_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=your_key
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
```

### Crawler Settings

Edit `hotsearchcrawler/settings.py`:

```python
# MySQL connection for crawler
MYSQL_CONFIG = {
    'host': os.getenv('MYSQL_HOST', 'localhost'),
    'port': int(os.getenv('MYSQL_PORT', 3306)),
    'user': os.getenv('MYSQL_USER'),
    'password': os.getenv('MYSQL_PASSWORD'),
    'database': os.getenv('MYSQL_DATABASE'),
}

# Optional: Platform-specific cookies for authenticated access
COOKIES = {
    'weibo': 'your_weibo_cookie',  # Optional, for protected content
    'douyin': 'your_douyin_cookie',
}

# Crawl interval (seconds)
CRAWL_INTERVAL = 300  # 5 minutes

# Concurrent requests
CONCURRENT_REQUESTS = 16
```

## Core Usage Patterns

### Starting the Application

```bash
# Launch main application (web interface + API)
python app.py

# Access web interface at:
# http://localhost:5000
```

### Manual Crawler Testing

```python
# Test individual platform crawler
# runspider-test.py example

from hotsearchcrawler.spiders.weibo_spider import WeiboSpider
from scrapy.crawler import CrawlerProcess

process = CrawlerProcess()
process.crawl(WeiboSpider)
process.start()
```

### Programmatic API Usage

```python
from hotsearch_analysis_agent.database import get_hot_searches, save_analysis_result
from hotsearch_analysis_agent.llm_client import analyze_sentiment, cluster_topics

# Query recent hot searches
searches = get_hot_searches(platform='weibo', limit=50)

# Perform sentiment analysis
for search in searches:
    sentiment = analyze_sentiment(search['title'])
    save_analysis_result(search['id'], sentiment_score=sentiment['score'])

# Cluster related topics
topics = cluster_topics([s['title'] for s in searches])
print(f"Found {len(topics)} topic clusters")
```

### LLM Analysis Integration

```python
# hotsearch_analysis_agent/llm_client.py pattern

import openai
import os

class LLMAnalyzer:
    def __init__(self):
        openai.api_base = os.getenv('LLM_API_BASE')
        openai.api_key = os.getenv('LLM_API_KEY')
        self.model = os.getenv('LLM_MODEL')
    
    def analyze_sentiment(self, text):
        """Analyze sentiment of trending topic"""
        prompt = f"""分析以下热搜话题的情感倾向(正面/负面/中性):
        
话题: {text}

请返回JSON格式: {{"sentiment": "正面/负面/中性", "confidence": 0.95, "reason": "简要理由"}}"""
        
        response = openai.ChatCompletion.create(
            model=self.model,
            messages=[{"role": "user", "content": prompt}],
            temperature=0.3
        )
        
        return response.choices[0].message.content
    
    def cluster_topics(self, titles):
        """Cluster related trending topics"""
        prompt = f"""将以下热搜话题进行聚类分组:

{chr(10).join(f'{i+1}. {t}' for i, t in enumerate(titles))}

请识别相关话题并分组,返回JSON格式: {{"clusters": [{{"theme": "主题", "topics": [1, 3, 5]}}]}}"""
        
        response = openai.ChatCompletion.create(
            model=self.model,
            messages=[{"role": "user", "content": prompt}],
            temperature=0.5
        )
        
        return response.choices[0].message.content
    
    def generate_report(self, query, relevant_news):
        """Generate analytical report for specific query"""
        news_text = "\n\n".join([
            f"标题: {n['title']}\nURL: {n['url']}\n摘要: {n['summary']}"
            for n in relevant_news
        ])
        
        prompt = f"""基于以下新闻数据,生成关于"{query}"的舆情分析报告:

{news_text}

报告要求:
1. 核心发现与数据亮点
2. 详细新闻内容梳理
3. 分析与总结
4. 信息传播特点"""
        
        response = openai.ChatCompletion.create(
            model=self.model,
            messages=[{"role": "user", "content": prompt}],
            temperature=0.7,
            max_tokens=2000
        )
        
        return response.choices[0].message.content
```

### Push Notification Setup

```python
# test_push_task.py pattern

from hotsearch_analysis_agent.push_service import PushService

push_service = PushService()

# Configure push task
task_config = {
    'name': 'AI Technology Monitoring',
    'query': '人工智能',  # Keyword to monitor
    'platforms': ['weibo', 'zhihu', 'bilibili'],
    'schedule': '0 9,18 * * *',  # Cron format: 9 AM and 6 PM daily
    'channels': ['wework', 'telegram'],
    'min_hot_value': 1000000,  # Minimum trending score threshold
}

# Test push to Enterprise WeChat
report = generate_report('人工智能', recent_news)
push_service.send_wework(report)

# Test push to Telegram
push_service.send_telegram(report)

# Test email push
push_service.send_email(
    subject='舆情报告 - 人工智能',
    body=report,
    recipients=['analyst@company.com']
)
```

### Content Extraction from Detail Pages

```python
# Using Selenium for dynamic content extraction

from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

class ContentExtractor:
    def __init__(self):
        options = Options()
        options.add_argument('--headless')
        options.add_argument('--no-sandbox')
        options.add_argument('--disable-dev-shm-usage')
        self.driver = webdriver.Chrome(options=options)
    
    def extract_article(self, url):
        """Extract full article content from news detail page"""
        self.driver.get(url)
        
        # Wait for content to load
        wait = WebDriverWait(self.driver, 10)
        
        try:
            # Adjust selectors based on platform
            if 'weibo.com' in url:
                content = wait.until(
                    EC.presence_of_element_located((By.CLASS_NAME, 'Feed_body'))
                ).text
            elif 'bilibili.com' in url:
                # Extract video description and comments
                title = self.driver.find_element(By.CLASS_NAME, 'video-title').text
                desc = self.driver.find_element(By.CLASS_NAME, 'video-desc').text
                content = f"{title}\n\n{desc}"
            else:
                content = self.driver.find_element(By.TAG_NAME, 'article').text
            
            return content
        except Exception as e:
            print(f"Failed to extract content: {e}")
            return ""
    
    def close(self):
        self.driver.quit()
```

### Web API Endpoints

```python
# Typical Flask routes in app.py

from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/api/search', methods=['POST'])
def search_topics():
    """Search trending topics by query"""
    data = request.json
    query = data.get('query')
    platform = data.get('platform', 'all')
    
    results = search_hot_searches(query, platform)
    return jsonify(results)

@app.route('/api/analyze', methods=['POST'])
def analyze_topic():
    """Analyze specific topic with LLM"""
    data = request.json
    topic_id = data.get('topic_id')
    
    topic_data = get_topic_details(topic_id)
    analysis = llm_analyzer.analyze_sentiment(topic_data['content'])
    
    return jsonify(analysis)

@app.route('/api/crawler/start', methods=['POST'])
def start_crawler():
    """Start crawler for specified platforms"""
    data = request.json
    platforms = data.get('platforms', [])
    
    # Start crawler subprocess
    subprocess.Popen(['python', 'run_spiders.py', '--platforms', ','.join(platforms)])
    
    return jsonify({'status': 'started'})

@app.route('/api/report/generate', methods=['POST'])
def generate_report_api():
    """Generate analytical report"""
    data = request.json
    query = data.get('query')
    time_range = data.get('time_range', '24h')
    
    relevant_news = fetch_relevant_news(query, time_range)
    report = llm_analyzer.generate_report(query, relevant_news)
    
    return jsonify({'report': report})
```

## Common Workflows

### Setting Up Automated Monitoring

1. **Configure crawler targets** in `hotsearchcrawler/settings.py`
2. **Start crawler service**: `python run_spiders.py`
3. **Set up push tasks** with cron schedules
4. **Monitor via web dashboard** at `http://localhost:5000`

### Custom Platform Integration

```python
# Add new platform spider in hotsearchcrawler/spiders/

import scrapy
from hotsearchcrawler.items import HotSearchItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://customplatform.com/trending']
    
    def parse(self, response):
        for item in response.css('.trending-item'):
            yield HotSearchItem(
                platform='custom_platform',
                title=item.css('.title::text').get(),
                url=item.css('a::attr(href)').get(),
                hot_value=item.css('.hot-score::text').get(),
                rank=item.css('.rank::text').get(),
            )
```

### Scheduled Report Generation

```python
# Use APScheduler for periodic tasks

from apscheduler.schedulers.background import BackgroundScheduler

scheduler = BackgroundScheduler()

def daily_report_job():
    """Generate and send daily trending report"""
    analyzer = LLMAnalyzer()
    push_service = PushService()
    
    # Get top topics from last 24 hours
    topics = get_hot_searches(hours=24, limit=100)
    
    # Cluster and analyze
    clusters = analyzer.cluster_topics([t['title'] for t in topics])
    report = analyzer.generate_report('每日舆情总结', topics)
    
    # Push to configured channels
    push_service.send_wework(report)
    push_service.send_email('Daily Opinion Report', report)

# Schedule daily at 9 AM
scheduler.add_job(daily_report_job, 'cron', hour=9)
scheduler.start()
```

## Troubleshooting

### Browser Driver Issues

```bash
# Error: "chromedriver not found in PATH"
# Solution: Verify driver location
which chromedriver  # Linux/macOS
where chromedriver  # Windows

# Add to PATH temporarily
export PATH=$PATH:/path/to/driver  # Linux/macOS
set PATH=%PATH%;C:\path\to\driver  # Windows

# Or specify driver path in code
from selenium import webdriver
driver = webdriver.Chrome(executable_path='/path/to/chromedriver')
```

### MySQL Connection Errors

```python
# Error: "Access denied for user"
# Check .env credentials match MySQL user

# Error: "Unknown database"
# Ensure database exists:
# mysql -u root -p -e "CREATE DATABASE hotsearch_db;"

# Test connection
import pymysql
conn = pymysql.connect(
    host=os.getenv('MYSQL_HOST'),
    user=os.getenv('MYSQL_USER'),
    password=os.getenv('MYSQL_PASSWORD'),
    database=os.getenv('MYSQL_DATABASE')
)
print("Connection successful!")
```

### LLM API Timeouts

```python
# Increase timeout for slow models
openai.api_request_timeout = 60  # seconds

# Retry logic
from tenacity import retry, stop_after_attempt, wait_fixed

@retry(stop=stop_after_attempt(3), wait=wait_fixed(2))
def analyze_with_retry(text):
    return llm_analyzer.analyze_sentiment(text)
```

### Platform Anti-Scraping

```python
# Add delays and user agents
# In settings.py:
DOWNLOAD_DELAY = 2  # seconds between requests
USER_AGENT = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'

# Rotate proxies if needed
ROTATING_PROXY_LIST = [
    'http://proxy1:8080',
    'http://proxy2:8080',
]
```

### Crawler Not Collecting Data

```bash
# Check crawler logs
tail -f logs/crawler.log

# Test individual spider
scrapy crawl weibo -o output.json

# Verify selectors still work (platforms change HTML frequently)
scrapy shell "https://s.weibo.com/top/summary"
>>> response.css('.td-02 a::text').getall()
```

### Push Notifications Not Sending

```python
# Test webhook connectivity
import requests

response = requests.post(
    os.getenv('WEWORK_WEBHOOK'),
    json={'msgtype': 'text', 'text': {'content': 'Test message'}}
)
print(response.status_code, response.text)

# Check Telegram bot
from telegram import Bot
bot = Bot(token=os.getenv('TELEGRAM_BOT_TOKEN'))
bot.send_message(chat_id=os.getenv('TELEGRAM_CHAT_ID'), text='Test')
```

## Key Files Reference

- `app.py` - Main Flask application entry point
- `run_spiders.py` - Crawler orchestration script (called from web UI)
- `hotsearchcrawler/settings.py` - Scrapy crawler configuration
- `hotsearch_analysis_agent/llm_client.py` - LLM integration layer
- `hotsearch_analysis_agent/database.py` - Database operations
- `hotsearch_analysis_agent/push_service.py` - Multi-channel notification service
- `test_push_task.py` - Push notification testing utilities
- `init.py` - Database schema initialization reference

## Performance Tips

- Use connection pooling for MySQL (SQLAlchemy recommended)
- Cache LLM responses for frequently queried topics
- Implement rate limiting on API endpoints
- Run crawlers in separate processes/containers
- Use Redis for task queue if scaling beyond single server
- Monitor ChromeDriver memory usage (restart periodically)
