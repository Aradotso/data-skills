---
name: llm-public-opinion-analytics-assistant
description: Multi-platform hot search data crawler and LLM-powered public opinion analysis system with sentiment analysis, topic clustering, and multi-channel alerts
triggers:
  - analyze public opinion from social media platforms
  - set up hot topic monitoring and alerts
  - crawl trending news from chinese platforms
  - perform sentiment analysis on social media trends
  - cluster and analyze trending topics
  - send hot topic alerts to wechat or telegram
  - scrape and analyze weibo douyin bilibili trends
  - build a public opinion monitoring system
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive public opinion analysis system that combines real-time data crawling from 15 mainstream Chinese platforms (26 ranking lists) with LLM-powered analysis capabilities. The system provides conversational hot search queries, topic clustering, sentiment analysis, and multi-channel push notifications (Email, WeChat, Enterprise WeChat, Telegram).

## Overview

This project consists of two main components:
1. **Crawler Cluster** (`hotsearchcrawler/`) - Scrapy-based distributed crawlers for 15 platforms
2. **Analysis Agent** (`hotsearch_analysis_agent/`) - LLM-powered analysis system with web interface

**Supported Platforms**: Weibo, Douyin, Bilibili, Baidu, Zhihu, Toutiao, Kuaishou, Xiaohongshu, 36Kr, and more.

## Installation

### Prerequisites

```bash
# System requirements
- Python 3.8+
- MySQL 5.7+
- Chrome/Edge browser + WebDriver
```

### WebDriver Setup

1. **Check browser version**: Open Chrome/Edge → Settings → About
2. **Download matching driver**:
   - Chrome: https://chromedriver.chromium.org/
   - Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/
3. **Add to PATH**:
   ```bash
   # Linux/macOS
   export PATH=$PATH:/path/to/driver
   
   # Windows
   # Add driver directory to System Environment Variables → Path
   ```
4. **Verify**:
   ```bash
   chromedriver --version
   ```

### Project Setup

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

### Database Configuration

```python
# Reference: init.py
import pymysql

connection = pymysql.connect(
    host='localhost',
    user='root',
    password='your_password',
    database='hotsearch_db',
    charset='utf8mb4'
)

# Create main tables
cursor = connection.cursor()
cursor.execute("""
CREATE TABLE IF NOT EXISTS hot_searches (
    id INT AUTO_INCREMENT PRIMARY KEY,
    platform VARCHAR(50),
    title VARCHAR(500),
    url VARCHAR(1000),
    hot_value VARCHAR(100),
    rank_position INT,
    content TEXT,
    sentiment VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
""")
connection.commit()
```

## Configuration

### Crawler Settings

Edit `hotsearchcrawler/settings.py`:

```python
# MySQL Configuration
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'root'
MYSQL_PASSWORD = 'your_password'
MYSQL_DB = 'hotsearch_db'

# Crawler Settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
ROBOTSTXT_OBEY = False

# User Agent Rotation
USER_AGENT_LIST = [
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
    # Add more user agents
]
```

### Analysis System Configuration

Create `.env` file in project root:

```bash
# Database
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Or use Huawei Pangu model
PANGU_API_KEY=your_pangu_key
PANGU_MODEL_PATH=/path/to/pangu/model

# Push Notification Channels
# Email
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password

# WeChat Work (Enterprise WeChat)
WECHAT_CORP_ID=your_corp_id
WECHAT_AGENT_ID=your_agent_id
WECHAT_SECRET=your_secret

# Telegram
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

## Running the System

### Start Crawlers

```bash
# Test single crawler
python runspider-test.py

# Start all crawlers (via web interface preferred)
python run_spiders.py
```

### Start Analysis System

```bash
# Launch web application
python app.py

# Access at http://localhost:5000
```

### Web Interface Features

- **Shortcut Controls**: Use keyboard shortcuts to start/stop crawlers
- **Query Interface**: Natural language queries like "今天微博热搜有什么关于AI的话题?"
- **Platform Jump**: Click any result to navigate to original source
- **Analysis Dashboard**: View sentiment trends and topic clusters

## Key API Patterns

### Query Hot Searches

```python
from hotsearch_analysis_agent.database import HotSearchDB

db = HotSearchDB()

# Query specific platform
weibo_trends = db.query_by_platform('weibo', limit=50)
for item in weibo_trends:
    print(f"{item['rank']}: {item['title']} (热度: {item['hot_value']})")

# Search by keyword
ai_topics = db.search_by_keyword('人工智能')

# Get time range data
recent = db.query_by_time_range(
    start_date='2026-05-01',
    end_date='2026-05-18'
)
```

### Sentiment Analysis

```python
from hotsearch_analysis_agent.analyzer import SentimentAnalyzer

analyzer = SentimentAnalyzer()

# Analyze single text
result = analyzer.analyze_text("这个产品真的很好用,强烈推荐!")
# Returns: {'sentiment': 'positive', 'score': 0.92, 'keywords': [...]}

# Batch analysis
texts = [item['title'] for item in weibo_trends]
sentiments = analyzer.batch_analyze(texts)

# Aggregate sentiment for topic
topic_sentiment = analyzer.aggregate_sentiment(
    platform='weibo',
    keyword='GPT-6'
)
```

### Topic Clustering

```python
from hotsearch_analysis_agent.clustering import TopicClusterer

clusterer = TopicClusterer()

# Cluster recent hot searches
topics = db.get_recent_topics(hours=24)
clusters = clusterer.cluster_topics(topics, n_clusters=5)

for cluster_id, items in clusters.items():
    print(f"\n簇 {cluster_id}:")
    for item in items:
        print(f"  - {item['title']}")
```

### LLM-Powered Analysis

```python
from hotsearch_analysis_agent.llm import AnalysisAgent

agent = AnalysisAgent()

# Conversational query
response = agent.query("分析一下今天关于人工智能的热搜趋势")
print(response)

# Generate report
report = agent.generate_report(
    keyword='人工智能',
    platforms=['weibo', 'zhihu', 'bilibili'],
    time_range='24h'
)
print(report)  # Markdown-formatted analysis
```

### Push Notifications

```python
from hotsearch_analysis_agent.pusher import NotificationPusher

pusher = NotificationPusher()

# Create push task
task = {
    'keyword': 'DeepSeek',
    'platforms': ['weibo', 'zhihu'],
    'threshold': 100000,  # Hot value threshold
    'channels': ['wechat', 'telegram'],
    'schedule': '*/30 * * * *'  # Every 30 minutes
}

task_id = pusher.create_task(task)

# Manual trigger
pusher.trigger_push(
    title='AI热点快报',
    content=report,
    channels=['email', 'wechat']
)
```

## Crawler Development

### Add New Platform Crawler

```python
# hotsearchcrawler/spiders/new_platform.py
import scrapy
from hotsearchcrawler.items import HotSearchItem

class NewPlatformSpider(scrapy.Spider):
    name = 'new_platform'
    allowed_domains = ['example.com']
    start_urls = ['https://example.com/hot']
    
    def parse(self, response):
        for item in response.css('.hot-item'):
            yield HotSearchItem(
                platform='new_platform',
                title=item.css('.title::text').get(),
                url=item.css('a::attr(href)').get(),
                hot_value=item.css('.hot::text').get(),
                rank_position=item.css('.rank::text').get()
            )
```

### Extract Detail Page Content

```python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

def extract_content(url):
    options = webdriver.ChromeOptions()
    options.add_argument('--headless')
    driver = webdriver.Chrome(options=options)
    
    try:
        driver.get(url)
        wait = WebDriverWait(driver, 10)
        
        # Wait for content to load
        content = wait.until(
            EC.presence_of_element_located((By.CLASS_NAME, 'article-content'))
        ).text
        
        # Handle video pages
        video_title = driver.find_element(By.CLASS_NAME, 'video-title').text
        
        return {
            'content': content,
            'video_title': video_title
        }
    finally:
        driver.quit()
```

## Testing Push Tasks

```bash
# Test all notification channels
python test_push_task.py

# Test specific channel
python -c "
from hotsearch_analysis_agent.pusher import NotificationPusher
pusher = NotificationPusher()
pusher.test_channel('wechat')
"
```

## Common Patterns

### Daily Monitoring Workflow

```python
# monitor.py
import schedule
import time
from hotsearch_analysis_agent.monitor import HotSearchMonitor

monitor = HotSearchMonitor()

def daily_analysis():
    # Collect data from all platforms
    monitor.collect_all_platforms()
    
    # Generate analysis report
    report = monitor.analyze_trends(keywords=['AI', '科技', '创新'])
    
    # Push to configured channels
    monitor.push_report(report)

# Schedule daily at 9 AM
schedule.every().day.at("09:00").do(daily_analysis)

while True:
    schedule.run_pending()
    time.sleep(60)
```

### Custom Analysis Pipeline

```python
from hotsearch_analysis_agent.pipeline import AnalysisPipeline

pipeline = AnalysisPipeline()

# Define custom analysis steps
pipeline.add_step('fetch', platforms=['weibo', 'zhihu'])
pipeline.add_step('filter', keyword='AI', min_hot_value=50000)
pipeline.add_step('sentiment', method='llm')
pipeline.add_step('cluster', n_clusters=3)
pipeline.add_step('report', format='markdown')
pipeline.add_step('push', channels=['email'])

# Execute pipeline
results = pipeline.run()
```

## Troubleshooting

### Crawler Issues

**Problem**: 403 Forbidden or blocked requests
```python
# Solution: Add cookies and better user agents
# hotsearchcrawler/settings.py
COOKIES_ENABLED = True
DEFAULT_REQUEST_HEADERS = {
    'Accept': 'text/html,application/xhtml+xml,application/xml',
    'Accept-Language': 'zh-CN,zh;q=0.9,en;q=0.8',
    'Referer': 'https://www.weibo.com/'
}
```

**Problem**: WebDriver not found
```bash
# Verify PATH configuration
which chromedriver  # Linux/macOS
where chromedriver  # Windows

# Or specify driver path explicitly
export CHROMEDRIVER_PATH=/usr/local/bin/chromedriver
```

### Database Issues

**Problem**: Connection errors
```python
# Check MySQL service status
# Linux: sudo systemctl status mysql
# Windows: Check Services panel

# Test connection
import pymysql
try:
    conn = pymysql.connect(host='localhost', user='root', password='pwd')
    print("Connected!")
except Exception as e:
    print(f"Error: {e}")
```

**Problem**: Encoding issues with Chinese text
```python
# Ensure utf8mb4 charset
ALTER DATABASE hotsearch_db CHARACTER SET = utf8mb4 COLLATE = utf8mb4_unicode_ci;
ALTER TABLE hot_searches CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### LLM Analysis Issues

**Problem**: Rate limiting or timeout
```python
# Add retry logic
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=4, max=10))
def query_llm(prompt):
    return agent.query(prompt)
```

**Problem**: Poor Chinese analysis quality
```python
# Use specialized Chinese model or adjust prompt
agent = AnalysisAgent(
    model='gpt-4',  # Better Chinese understanding
    temperature=0.3,  # More focused outputs
    system_prompt="你是一个专业的中文舆情分析专家..."
)
```

### Push Notification Issues

**Problem**: WeChat push fails
```python
# Verify credentials
from hotsearch_analysis_agent.pusher import WeChatPusher

pusher = WeChatPusher()
token = pusher.get_access_token()
print(f"Token valid: {token is not None}")

# Check webhook URL
pusher.test_webhook()
```

## Advanced Usage

### Custom LLM Integration

```python
# Use Huawei Pangu model locally
from hotsearch_analysis_agent.llm import PanguAnalyzer

analyzer = PanguAnalyzer(
    model_path='/path/to/openpangu-embedded-7b',
    device='cuda'  # or 'cpu'
)

response = analyzer.analyze(
    prompt="分析以下热搜趋势...",
    max_length=2000
)
```

### Real-time Monitoring Dashboard

```python
# app.py extension
from flask import Flask, jsonify
from flask_socketio import SocketIO

app = Flask(__name__)
socketio = SocketIO(app)

@socketio.on('subscribe')
def handle_subscription(data):
    keyword = data['keyword']
    # Stream updates to client
    for update in monitor.stream_updates(keyword):
        socketio.emit('update', update)
```

This skill enables AI coding agents to help developers build comprehensive public opinion monitoring and analysis systems with multi-platform data collection, LLM-powered insights, and automated alerting capabilities.
