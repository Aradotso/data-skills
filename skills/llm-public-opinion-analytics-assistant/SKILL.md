---
name: llm-public-opinion-analytics-assistant
description: A real-time public opinion analytics system integrating 15 platforms' 26 ranking lists with LLM analysis for sentiment analysis, topic clustering, and multi-channel alerts
triggers:
  - set up public opinion monitoring with LLM analysis
  - how do I scrape hot search rankings from multiple platforms
  - configure sentiment analysis and topic clustering for news
  - send public opinion reports via WeChat or Telegram
  - analyze trending topics across Weibo Bilibili and other platforms
  - implement real-time hot search monitoring with push notifications
  - deploy a Chinese public opinion analytics assistant
  - integrate Pangu LLM for sentiment analysis
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is a comprehensive public opinion analytics assistant that combines real-time data from **15 mainstream platforms** (26 ranking lists) with large language model analysis capabilities. It provides conversational hot search queries, topic clustering, sentiment analysis, and multi-channel push notifications (Email, WeChat, Enterprise WeChat, Telegram).

## What This Project Does

- **Multi-Platform Data Collection**: Scrapes hot search rankings from Weibo, Bilibili, Zhihu, Douyin, and 11+ other Chinese platforms
- **LLM-Powered Analysis**: Uses Pangu or OpenAI-compatible models for sentiment analysis, topic clustering, and content summarization
- **Conversational Interface**: Natural language queries for hot search data
- **Content Deep Dive**: Extracts full article/video content from news detail pages
- **Multi-Channel Alerts**: Push reports via Email (SMTP), Enterprise WeChat, Telegram, WeChat
- **Hotkey Control**: Start/stop crawlers with keyboard shortcuts

## Architecture Overview

```
project/
├── hotsearch_analysis_agent/    # Analysis system (LLM backend)
├── hotsearchcrawler/            # Scrapy crawler cluster
├── app.py                       # Main Flask application
├── run_spiders.py               # Crawler launcher
├── test_push_task/              # Push notification tests
└── init.py                      # Database schema reference
```

## Installation

### Prerequisites

1. **Browser Driver Setup** (for news detail extraction):

```bash
# Download ChromeDriver matching your Chrome version
# From: https://chromedriver.chromium.org/
# Or EdgeDriver: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Linux/macOS: Place in /usr/local/bin/
sudo mv chromedriver /usr/local/bin/
sudo chmod +x /usr/local/bin/chromedriver

# Windows: Add driver directory to PATH
# Verify installation:
chromedriver --version
```

2. **MySQL Database**:

```bash
# Install MySQL 5.7+ or 8.0
# Create database and tables using init.py as reference
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

### Configuration

**1. Database Configuration** (`hotsearchcrawler/settings.py`):

```python
# MySQL settings for crawler
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_user'
MYSQL_PASSWORD = 'your_password'
MYSQL_DATABASE = 'hotsearch_db'
```

**2. Environment Variables** (`.env` file in project root):

```bash
# Database
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1  # Or Pangu endpoint
OPENAI_MODEL=gpt-4  # Or pangu-embedded-7b

# Push Notification Settings
# Enterprise WeChat Bot
WEWORK_WEBHOOK_URL=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
SMTP_FROM=your_email@gmail.com
SMTP_TO=recipient@example.com
```

**3. Platform Cookies** (Optional, for platforms requiring auth):

Edit `hotsearchcrawler/settings.py` or create `cookies.json`:

```python
# Example for specific platforms
COOKIES = {
    'weibo': 'your_weibo_cookie_string',
    'bilibili': 'your_bilibili_cookie_string'
}
```

## Database Schema

Reference `init.py` for table structures. Key tables:

```sql
-- Hot search entries
CREATE TABLE hot_searches (
    id INT PRIMARY KEY AUTO_INCREMENT,
    platform VARCHAR(50),
    title VARCHAR(500),
    url VARCHAR(1000),
    rank INT,
    heat_value VARCHAR(100),
    content TEXT,  -- Full article/video content
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Analysis results
CREATE TABLE analysis_results (
    id INT PRIMARY KEY AUTO_INCREMENT,
    query TEXT,
    sentiment VARCHAR(50),  -- positive/negative/neutral
    topics JSON,  -- Clustered topics
    summary TEXT,
    created_at TIMESTAMP
);

-- Push tasks
CREATE TABLE push_tasks (
    id INT PRIMARY KEY AUTO_INCREMENT,
    task_name VARCHAR(200),
    query_keywords TEXT,
    channels JSON,  -- ['wework', 'telegram', 'email']
    schedule VARCHAR(100),  -- Cron expression
    is_active BOOLEAN DEFAULT TRUE
);
```

## Running the System

### Start the Analysis Application

```bash
# Start Flask web interface
python app.py

# Access at http://localhost:5000
```

### Control Crawlers

**Via Web Interface**:
- Navigate to the dashboard
- Use hotkeys (configured in UI) to start/stop crawlers

**Via Command Line**:

```bash
# Test single crawler
python runspider-test.py weibo

# Run all crawlers
python run_spiders.py

# Run specific platform group
python run_spiders.py --platforms weibo,bilibili,zhihu
```

### Test Push Notifications

```bash
# Test all configured channels
cd test_push_task
python test_push.py

# Test specific channel
python test_push.py --channel wework
python test_push.py --channel telegram
python test_push.py --channel email
```

## Key API Usage Patterns

### Conversational Query Interface

```python
from hotsearch_analysis_agent.agent import AnalysisAgent

# Initialize agent with LLM
agent = AnalysisAgent(
    model_name=os.getenv('OPENAI_MODEL'),
    api_key=os.getenv('OPENAI_API_KEY'),
    api_base=os.getenv('OPENAI_API_BASE')
)

# Natural language query
response = agent.query("最近人工智能领域有什么热点新闻?")
print(response['answer'])
print(response['sources'])  # Source platforms and links

# Topic clustering
clusters = agent.cluster_topics(
    query="科技新闻",
    time_range="24h",
    platforms=["weibo", "zhihu", "bilibili"]
)
```

### Sentiment Analysis

```python
from hotsearch_analysis_agent.sentiment import SentimentAnalyzer

analyzer = SentimentAnalyzer(model_name=os.getenv('OPENAI_MODEL'))

# Analyze single text
result = analyzer.analyze("这个新政策对民生有很大帮助")
print(result['sentiment'])  # 'positive'
print(result['confidence'])  # 0.92

# Batch analysis
texts = [item['title'] for item in hot_searches]
sentiments = analyzer.batch_analyze(texts)
```

### Topic Clustering

```python
from hotsearch_analysis_agent.clustering import TopicClusterer

clusterer = TopicClusterer(model_name=os.getenv('OPENAI_MODEL'))

# Fetch recent hot searches
hot_searches = fetch_hot_searches(platform='all', hours=24)

# Cluster by semantic similarity
clusters = clusterer.cluster(
    texts=[item['title'] for item in hot_searches],
    num_clusters=5,
    method='semantic'  # or 'keyword'
)

for cluster in clusters:
    print(f"Topic: {cluster['topic_name']}")
    print(f"Items: {len(cluster['items'])}")
    print(f"Keywords: {cluster['keywords']}")
```

### Scraper Integration

```python
from hotsearchcrawler.spiders import WeiboSpider, BilibiliSpider

# Run spider programmatically
from scrapy.crawler import CrawlerProcess
from scrapy.utils.project import get_project_settings

process = CrawlerProcess(get_project_settings())

# Run single spider
process.crawl(WeiboSpider)
process.start()

# Run multiple spiders
process.crawl(WeiboSpider)
process.crawl(BilibiliSpider)
process.start()
```

### Push Notification Setup

```python
from hotsearch_analysis_agent.push import PushManager

push_manager = PushManager()

# Create push task
task = push_manager.create_task(
    task_name="AI Tech Weekly Report",
    query_keywords=["人工智能", "科技创新", "大模型"],
    channels=['wework', 'telegram', 'email'],
    schedule="0 12 * * 1",  # Every Monday at 12:00
    template="weekly_report"
)

# Execute task immediately
report = push_manager.execute_task(task_id=task['id'])

# Manual push
push_manager.push_to_wework(
    content="# Hot Topic Alert\n\n紧急舆情监测...",
    webhook_url=os.getenv('WEWORK_WEBHOOK_URL')
)

push_manager.push_to_telegram(
    content="🔥 Trending: AI breakthroughs...",
    bot_token=os.getenv('TELEGRAM_BOT_TOKEN'),
    chat_id=os.getenv('TELEGRAM_CHAT_ID')
)
```

## Common Patterns

### Pattern 1: Daily Hot Search Report

```python
from hotsearch_analysis_agent import AnalysisAgent, PushManager
from datetime import datetime, timedelta

# Fetch last 24h hot searches
agent = AnalysisAgent()
report = agent.generate_report(
    time_range=(datetime.now() - timedelta(days=1), datetime.now()),
    platforms=['weibo', 'zhihu', 'bilibili', 'douyin'],
    include_sentiment=True,
    include_clustering=True
)

# Push to multiple channels
push_manager = PushManager()
push_manager.push_to_wework(report['markdown'])
push_manager.push_to_email(
    subject=f"Daily Hot Search Report - {datetime.now().strftime('%Y-%m-%d')}",
    body=report['html']
)
```

### Pattern 2: Keyword Monitoring with Alerts

```python
from hotsearch_analysis_agent import KeywordMonitor

monitor = KeywordMonitor(
    keywords=["疫情", "政策", "房价", "股市"],
    platforms=['weibo', 'toutiao', 'baidu'],
    check_interval=300  # 5 minutes
)

@monitor.on_match
def handle_keyword_match(match):
    print(f"Keyword '{match['keyword']}' found in {match['platform']}")
    print(f"Title: {match['title']}")
    print(f"Heat: {match['heat_value']}")
    
    # Send alert if heat exceeds threshold
    if int(match['heat_value']) > 1000000:
        push_manager.push_to_telegram(
            content=f"🚨 High Heat Alert\n\n{match['title']}\n{match['url']}"
        )

monitor.start()
```

### Pattern 3: Custom Platform Scraper

```python
import scrapy
from hotsearchcrawler.items import HotSearchItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://example.com/hot-rankings']
    
    def parse(self, response):
        for idx, item in enumerate(response.css('.hot-item')):
            hot_search = HotSearchItem()
            hot_search['platform'] = 'custom_platform'
            hot_search['title'] = item.css('.title::text').get()
            hot_search['url'] = item.css('a::attr(href)').get()
            hot_search['rank'] = idx + 1
            hot_search['heat_value'] = item.css('.heat::text').get()
            
            # Fetch detail page content
            yield scrapy.Request(
                url=hot_search['url'],
                callback=self.parse_detail,
                meta={'item': hot_search}
            )
    
    def parse_detail(self, response):
        item = response.meta['item']
        item['content'] = response.css('.article-content').get()
        yield item
```

## LLM Model Configuration

### Using Pangu Model (Recommended)

```python
# Download Pangu model from:
# https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

# Local deployment
from hotsearch_analysis_agent.llm import PanguModel

model = PanguModel(
    model_path='/path/to/openpangu-embedded-7b',
    device='cuda'  # or 'cpu'
)

# Use in agent
agent = AnalysisAgent(llm=model)
```

### Using OpenAI-Compatible APIs

```python
# .env configuration
OPENAI_API_BASE=https://api.openai.com/v1
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4

# Or use local vLLM server
OPENAI_API_BASE=http://localhost:8000/v1
OPENAI_MODEL=pangu-7b
```

## Troubleshooting

### Crawler Issues

**Problem**: Crawler returns empty results
```python
# Check if platform changed HTML structure
from hotsearchcrawler.spiders import WeiboSpider
spider = WeiboSpider()

# Enable debug logging
import logging
logging.basicConfig(level=logging.DEBUG)

# Test selectors
response = spider.parse(test_response)
```

**Problem**: Anti-scraping detection
```python
# Add cookies in settings.py
DEFAULT_REQUEST_HEADERS = {
    'User-Agent': 'Mozilla/5.0...',
    'Cookie': 'your_cookie_string'
}

# Use proxy rotation
DOWNLOADER_MIDDLEWARES = {
    'scrapy.downloadermiddlewares.httpproxy.HttpProxyMiddleware': 1,
}
```

### Database Connection Errors

```python
# Test connection
import pymysql

conn = pymysql.connect(
    host=os.getenv('MYSQL_HOST'),
    port=int(os.getenv('MYSQL_PORT')),
    user=os.getenv('MYSQL_USER'),
    password=os.getenv('MYSQL_PASSWORD'),
    database=os.getenv('MYSQL_DATABASE'),
    charset='utf8mb4'
)
print("Connection successful")
conn.close()
```

### Push Notification Failures

**Enterprise WeChat**:
```python
# Verify webhook URL
import requests

response = requests.post(
    os.getenv('WEWORK_WEBHOOK_URL'),
    json={
        "msgtype": "text",
        "text": {"content": "Test message"}
    }
)
print(response.json())
```

**Telegram**:
```python
# Test bot token
import requests

url = f"https://api.telegram.org/bot{os.getenv('TELEGRAM_BOT_TOKEN')}/getMe"
response = requests.get(url)
print(response.json())
```

### LLM Model Issues

**Problem**: Model loading fails
```python
# Check model path and format
import os
model_path = '/path/to/openpangu-embedded-7b'
print(os.path.exists(model_path))
print(os.listdir(model_path))  # Should contain config.json, pytorch_model.bin

# Check GPU availability
import torch
print(torch.cuda.is_available())
```

**Problem**: Analysis quality poor
```python
# Adjust prompt templates in hotsearch_analysis_agent/prompts.py
# Increase temperature for more creative analysis
agent = AnalysisAgent(temperature=0.8, max_tokens=2000)
```

### Browser Driver Issues

```bash
# Chrome version mismatch
google-chrome --version
chromedriver --version  # Should match major version

# Download correct version
# From: https://chromedriver.chromium.org/downloads

# Set explicit path in code
from selenium import webdriver
driver = webdriver.Chrome(executable_path='/usr/local/bin/chromedriver')
```

## Performance Optimization

```python
# Enable concurrent scraping
CONCURRENT_REQUESTS = 16
CONCURRENT_REQUESTS_PER_DOMAIN = 8

# Use Redis for deduplication
DUPEFILTER_CLASS = 'scrapy_redis.dupefilter.RFPDupeFilter'
SCHEDULER = 'scrapy_redis.scheduler.Scheduler'

# Cache LLM results
from functools import lru_cache

@lru_cache(maxsize=1000)
def cached_sentiment_analysis(text):
    return analyzer.analyze(text)
```
