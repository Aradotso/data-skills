---
name: llm-public-opinion-analytics-assistant
description: Intelligent public opinion analytics system aggregating 26 hot topic lists from 15 platforms with LLM-powered sentiment analysis, clustering, and multi-channel alerts
triggers:
  - set up public opinion monitoring system
  - analyze hot topics from chinese social media
  - configure sentiment analysis for news aggregation
  - implement multi-platform hot search crawler
  - create intelligent opinion analytics dashboard
  - deploy llm based news sentiment monitoring
  - build hot topic clustering and alert system
  - integrate pangu model for chinese text analysis
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive public opinion analytics system that aggregates real-time data from 26 hot topic lists across 15 major Chinese platforms (Weibo, Bilibili, Douyin, etc.). Combines web crawling, LLM-powered analysis (sentiment, clustering, topic extraction), and multi-channel push notifications (WeChat, Email, Telegram). Supports conversational queries, keyboard shortcuts for crawler control, and detailed content extraction from news pages including video content.

## What This Project Does

- **Multi-Platform Data Aggregation**: Crawls 26 hot topic lists from 15 platforms including Weibo, Bilibili, Douyin, Baidu, Toutiao, Zhihu, etc.
- **LLM-Powered Analysis**: Uses Huawei Pangu model (or OpenAI-compatible APIs) for sentiment analysis, topic clustering, and trend detection
- **Conversational Interface**: Natural language queries for hot topic search, theme-based filtering, and trend analysis
- **Deep Content Extraction**: Fetches full article/video content from detail pages using browser automation
- **Multi-Channel Alerts**: Push reports via Enterprise WeChat, WeChat Work App, Telegram, and Email
- **Crawler Management**: Start/stop crawlers via keyboard shortcuts in the web interface

## Installation

### Prerequisites

```bash
# Python 3.8+
# MySQL 5.7+ or 8.0+
# Chrome/Edge browser installed
```

### 1. Browser Driver Setup

**Download matching driver for your browser version:**

```bash
# Check Chrome version
google-chrome --version  # Linux
# Or check in browser: Settings → About

# Download ChromeDriver from:
# https://chromedriver.chromium.org/

# Or EdgeDriver from:
# https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Place driver in PATH or project directory
sudo mv chromedriver /usr/local/bin/  # macOS/Linux
# Windows: Add driver location to PATH environment variable

# Verify installation
chromedriver --version
```

### 2. Clone and Install Dependencies

```bash
git clone https://github.com/hmmnxkl/LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant.git
cd LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant

# Create virtual environment
python -m venv venv
source venv/bin/activate  # Linux/macOS
# venv\Scripts\activate  # Windows

# Install dependencies
pip install -r requirements.txt
```

### 3. Database Setup

```bash
# Install MySQL and create database
mysql -u root -p

CREATE DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

```python
# Reference init.py for table schema
# Run database initialization (modify connection params first)
python init.py
```

### 4. Configuration

**Create `.env` file in project root:**

```bash
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch

# LLM API Configuration (OpenAI-compatible format)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://your-api-endpoint.com/v1
OPENAI_MODEL=gpt-4

# Or use Pangu model (local deployment)
# PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model

# Push Notification Channels (optional)
# Enterprise WeChat Robot
WECHAT_ROBOT_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email SMTP
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
SMTP_RECIPIENT=recipient@example.com
```

**Configure crawler settings in `hotsearchcrawler/settings.py`:**

```python
# MySQL connection for crawler
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_user'
MYSQL_PASSWORD = 'your_password'
MYSQL_DATABASE = 'hotsearch'

# Optional: Add cookies for platforms requiring authentication
COOKIES = {
    'weibo': 'your_weibo_cookies',
    'bilibili': 'your_bilibili_cookies'
}
```

## Running the System

### Start the Main Application

```bash
# Activate virtual environment first
source venv/bin/activate

# Run the web interface
python app.py

# Access at http://localhost:5000
```

### Start Crawlers

**Option 1: Via Web Interface**
- Open browser to `http://localhost:5000`
- Use keyboard shortcuts to start/stop crawlers
- Monitor crawler status in real-time

**Option 2: Command Line**

```bash
# Test individual spider
python runspider-test.py

# Run all spiders
python run_spiders.py
```

## Key API and Usage Patterns

### Conversational Query Interface

```python
from hotsearch_analysis_agent import AnalysisAgent

# Initialize agent with LLM configuration
agent = AnalysisAgent(
    api_key=os.getenv('OPENAI_API_KEY'),
    api_base=os.getenv('OPENAI_API_BASE'),
    model=os.getenv('OPENAI_MODEL')
)

# Query hot topics by keyword
response = agent.query("查询人工智能相关的热搜")
# Returns clustered topics with sentiment analysis

# Theme-based search
response = agent.query("最近科技领域的热点话题")

# Sentiment analysis request
response = agent.query("分析华为新品发布的舆论情感倾向")
```

### Direct Database Query for Hot Topics

```python
import pymysql
from datetime import datetime, timedelta

connection = pymysql.connect(
    host=os.getenv('MYSQL_HOST'),
    user=os.getenv('MYSQL_USER'),
    password=os.getenv('MYSQL_PASSWORD'),
    database=os.getenv('MYSQL_DATABASE'),
    charset='utf8mb4'
)

def get_recent_hot_topics(platform='weibo', hours=24):
    """Fetch hot topics from specific platform"""
    with connection.cursor() as cursor:
        sql = """
        SELECT title, url, hot_value, platform, created_at 
        FROM hot_topics 
        WHERE platform = %s 
        AND created_at >= %s 
        ORDER BY hot_value DESC 
        LIMIT 50
        """
        time_threshold = datetime.now() - timedelta(hours=hours)
        cursor.execute(sql, (platform, time_threshold))
        return cursor.fetchall()

# Get Weibo hot topics from last 24 hours
topics = get_recent_hot_topics('weibo', 24)
for topic in topics:
    print(f"{topic[0]} - Hot Value: {topic[2]}")
```

### Topic Clustering Analysis

```python
from hotsearch_analysis_agent import ClusterAnalyzer

analyzer = ClusterAnalyzer(
    model_name=os.getenv('OPENAI_MODEL'),
    api_key=os.getenv('OPENAI_API_KEY')
)

# Fetch topics and perform clustering
topics = get_recent_hot_topics(platform='all', hours=48)
clusters = analyzer.cluster_topics(topics)

# Generate cluster report
for cluster_id, cluster_data in clusters.items():
    print(f"Cluster {cluster_id}: {cluster_data['theme']}")
    print(f"Topics count: {len(cluster_data['topics'])}")
    print(f"Sentiment: {cluster_data['sentiment']}")
    print("---")
```

### Sentiment Analysis on Extracted Content

```python
from hotsearch_analysis_agent import SentimentAnalyzer
from hotsearchcrawler.content_extractor import ContentExtractor

# Extract full content from detail page
extractor = ContentExtractor()
content = extractor.extract_from_url(
    url="https://weibo.com/detail/xxxxx",
    platform="weibo"
)

# Analyze sentiment
sentiment_analyzer = SentimentAnalyzer(
    model=os.getenv('OPENAI_MODEL'),
    api_key=os.getenv('OPENAI_API_KEY')
)

result = sentiment_analyzer.analyze(content['text'])
print(f"Sentiment: {result['sentiment']}")  # positive/negative/neutral
print(f"Score: {result['score']}")  # 0-1
print(f"Key phrases: {result['key_phrases']}")
```

### Setting Up Push Notifications

```python
from hotsearch_analysis_agent import PushTaskManager

push_manager = PushTaskManager()

# Create a push task for AI-related topics
task_config = {
    'name': '人工智能热点监控',
    'keywords': ['人工智能', 'AI', '大模型', '机器学习'],
    'platforms': ['weibo', 'toutiao', 'zhihu'],
    'schedule': '0 8,12,18 * * *',  # Cron: 8am, 12pm, 6pm daily
    'channels': ['wechat_work', 'email'],
    'min_hot_value': 100000,  # Minimum hot value threshold
    'sentiment_filter': None  # None, 'positive', 'negative'
}

task_id = push_manager.create_task(task_config)
print(f"Task created: {task_id}")
```

**Test push notification manually:**

```bash
python test_push_task.py --task_id 1
```

### WeChat Work Robot Push

```python
import requests
import json

def push_to_wechat_work(webhook_url, content):
    """Send markdown report to WeChat Work group"""
    headers = {'Content-Type': 'application/json'}
    data = {
        "msgtype": "markdown",
        "markdown": {
            "content": content
        }
    }
    response = requests.post(webhook_url, headers=headers, data=json.dumps(data))
    return response.json()

# Generate report and push
webhook = os.getenv('WECHAT_ROBOT_WEBHOOK')
report = """## 人工智能热点分析
**时间**: 2026-05-18 09:00

🔹 **GPT-6曝光**: 200万token上下文
🔹 **DeepSeek V4**: 采用华为昇腾算力
🔹 **中国大模型**: 周调用量连续5周全球第一

[查看详情](http://your-domain.com/report/123)
"""

push_to_wechat_work(webhook, report)
```

### Telegram Bot Push

```python
import requests

def push_to_telegram(bot_token, chat_id, message):
    """Send message via Telegram bot"""
    url = f"https://api.telegram.org/bot{bot_token}/sendMessage"
    payload = {
        'chat_id': chat_id,
        'text': message,
        'parse_mode': 'Markdown'
    }
    response = requests.post(url, json=payload)
    return response.json()

# Send report
bot_token = os.getenv('TELEGRAM_BOT_TOKEN')
chat_id = os.getenv('TELEGRAM_CHAT_ID')
message = "🔥 *AI领域热点* (2026-05-18)\n\n• GPT-6提前曝光\n• 中国大模型调用量领先"

push_to_telegram(bot_token, chat_id, message)
```

## Crawler Development Pattern

### Adding a New Platform Spider

```python
# Create new spider in hotsearchcrawler/spiders/
import scrapy
from ..items import HotTopicItem

class NewPlatformSpider(scrapy.Spider):
    name = 'newplatform'
    allowed_domains = ['newplatform.com']
    start_urls = ['https://newplatform.com/hot']

    def parse(self, response):
        # Extract hot topic list
        for topic in response.css('.hot-item'):
            item = HotTopicItem()
            item['title'] = topic.css('.title::text').get()
            item['url'] = topic.css('a::attr(href)').get()
            item['hot_value'] = topic.css('.hot-num::text').get()
            item['platform'] = 'newplatform'
            item['rank'] = topic.css('.rank::text').get()
            
            # Request detail page for full content
            yield response.follow(
                item['url'],
                callback=self.parse_detail,
                meta={'item': item}
            )

    def parse_detail(self, response):
        """Extract full content from detail page"""
        item = response.meta['item']
        item['content'] = response.css('.article-content::text').getall()
        item['author'] = response.css('.author::text').get()
        item['publish_time'] = response.css('.time::attr(datetime)').get()
        yield item
```

### Handling JavaScript-Rendered Pages

```python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

class JsRenderedSpider(scrapy.Spider):
    name = 'js_platform'
    
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        options = webdriver.ChromeOptions()
        options.add_argument('--headless')
        options.add_argument('--no-sandbox')
        self.driver = webdriver.Chrome(options=options)
    
    def parse(self, response):
        self.driver.get(response.url)
        
        # Wait for dynamic content
        wait = WebDriverWait(self.driver, 10)
        wait.until(EC.presence_of_element_located((By.CLASS_NAME, 'hot-item')))
        
        # Extract after JS rendering
        items = self.driver.find_elements(By.CLASS_NAME, 'hot-item')
        for element in items:
            item = HotTopicItem()
            item['title'] = element.find_element(By.CLASS_NAME, 'title').text
            item['url'] = element.find_element(By.TAG_NAME, 'a').get_attribute('href')
            yield item
    
    def closed(self, reason):
        self.driver.quit()
```

## Common Patterns

### Scheduling Automated Reports

```python
from apscheduler.schedulers.background import BackgroundScheduler
from hotsearch_analysis_agent import ReportGenerator

scheduler = BackgroundScheduler()

def generate_daily_report():
    """Generate and push daily opinion analysis report"""
    generator = ReportGenerator()
    report = generator.create_report(
        time_range=24,  # Last 24 hours
        min_topics=5,   # At least 5 topics per cluster
        include_sentiment=True
    )
    
    # Push to configured channels
    push_manager.send_report(report, channels=['wechat_work', 'email'])

# Schedule daily at 9 AM and 6 PM
scheduler.add_job(generate_daily_report, 'cron', hour='9,18')
scheduler.start()
```

### Filtering Topics by Keywords

```python
def filter_topics_by_keywords(keywords, platforms=None, hours=24):
    """Filter hot topics matching specific keywords"""
    with connection.cursor() as cursor:
        platform_condition = ""
        if platforms:
            platform_list = "','".join(platforms)
            platform_condition = f"AND platform IN ('{platform_list}')"
        
        sql = f"""
        SELECT * FROM hot_topics 
        WHERE created_at >= DATE_SUB(NOW(), INTERVAL %s HOUR)
        {platform_condition}
        AND (
            title LIKE %s OR 
            content LIKE %s
        )
        ORDER BY hot_value DESC
        """
        
        pattern = f"%{'%'.join(keywords)}%"
        cursor.execute(sql, (hours, pattern, pattern))
        return cursor.fetchall()

# Example: Find AI-related topics
ai_topics = filter_topics_by_keywords(
    keywords=['AI', '人工智能', '大模型'],
    platforms=['weibo', 'zhihu', 'toutiao'],
    hours=48
)
```

### Custom LLM Integration (Pangu Model)

```python
from transformers import AutoTokenizer, AutoModelForCausalLM
import torch

class PanguAnalyzer:
    def __init__(self, model_path):
        self.tokenizer = AutoTokenizer.from_pretrained(model_path)
        self.model = AutoModelForCausalLM.from_pretrained(
            model_path,
            torch_dtype=torch.float16,
            device_map="auto"
        )
    
    def analyze_sentiment(self, text):
        """Sentiment analysis using Pangu model"""
        prompt = f"""请分析以下文本的情感倾向,回答正面、负面或中性:

文本: {text}

情感倾向:"""
        
        inputs = self.tokenizer(prompt, return_tensors="pt").to(self.model.device)
        outputs = self.model.generate(**inputs, max_length=512)
        result = self.tokenizer.decode(outputs[0], skip_special_tokens=True)
        
        return self._parse_sentiment(result)
    
    def _parse_sentiment(self, response):
        if '正面' in response:
            return 'positive'
        elif '负面' in response:
            return 'negative'
        return 'neutral'

# Use Pangu instead of OpenAI
pangu = PanguAnalyzer('/path/to/openpangu-embedded-7b-model')
sentiment = pangu.analyze_sentiment("华为发布新一代AI芯片,性能提升显著")
```

## Troubleshooting

### ChromeDriver Version Mismatch

```bash
# Error: "ChromeDriver version mismatch"
# Solution: Download exact matching version

# Check Chrome version
google-chrome --version
# Chrome 115.0.5790.102

# Download ChromeDriver 115.0.5790.102 from:
# https://chromedriver.chromium.org/downloads

# Replace old driver
sudo rm /usr/local/bin/chromedriver
sudo mv ~/Downloads/chromedriver /usr/local/bin/
sudo chmod +x /usr/local/bin/chromedriver
```

### Database Connection Errors

```python
# Error: "Lost connection to MySQL server"
# Solution: Increase connection timeout

import pymysql

connection = pymysql.connect(
    host=os.getenv('MYSQL_HOST'),
    user=os.getenv('MYSQL_USER'),
    password=os.getenv('MYSQL_PASSWORD'),
    database=os.getenv('MYSQL_DATABASE'),
    charset='utf8mb4',
    connect_timeout=10,
    read_timeout=30,
    write_timeout=30
)
```

### Crawler Blocked by Platform

```python
# Add random delays and user agents
import random
import time

class RateLimitedSpider(scrapy.Spider):
    download_delay = 2
    
    custom_settings = {
        'USER_AGENT': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
        'DOWNLOAD_DELAY': 2,
        'RANDOMIZE_DOWNLOAD_DELAY': True,
        'CONCURRENT_REQUESTS_PER_DOMAIN': 1
    }
    
    def parse(self, response):
        # Add random delay between requests
        time.sleep(random.uniform(1, 3))
        # ... extract data
```

### LLM API Rate Limit

```python
import time
from functools import wraps

def rate_limited_api_call(max_retries=3, delay=60):
    """Decorator to handle API rate limits"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if 'rate_limit' in str(e).lower():
                        wait_time = delay * (attempt + 1)
                        print(f"Rate limited. Waiting {wait_time}s...")
                        time.sleep(wait_time)
                    else:
                        raise
            raise Exception("Max retries exceeded")
        return wrapper
    return decorator

@rate_limited_api_call(max_retries=3, delay=60)
def call_llm_api(prompt):
    # Your API call here
    pass
```

### Memory Issues with Large Datasets

```python
# Use batch processing for large queries
def process_topics_in_batches(batch_size=1000):
    """Process topics in batches to avoid memory issues"""
    offset = 0
    while True:
        with connection.cursor() as cursor:
            sql = """
            SELECT * FROM hot_topics 
            ORDER BY id 
            LIMIT %s OFFSET %s
            """
            cursor.execute(sql, (batch_size, offset))
            batch = cursor.fetchall()
            
            if not batch:
                break
            
            # Process batch
            for topic in batch:
                analyze_topic(topic)
            
            offset += batch_size
            print(f"Processed {offset} topics")
```

### Push Notification Failures

```python
# Implement retry logic with exponential backoff
import requests
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def create_retry_session():
    """Create session with automatic retries"""
    session = requests.Session()
    retry = Retry(
        total=3,
        backoff_factor=1,
        status_forcelist=[500, 502, 503, 504]
    )
    adapter = HTTPAdapter(max_retries=retry)
    session.mount('http://', adapter)
    session.mount('https://', adapter)
    return session

# Use in push functions
session = create_retry_session()
response = session.post(webhook_url, json=payload, timeout=10)
```

## Reference Resources

- **Pangu Model Download**: https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model
- **Supported Platforms**: Weibo, Bilibili, Douyin, Baidu, Toutiao, Zhihu, 36Kr, Baidu Tieba, Douban, Weibo Hot, IT Home, Sina Tech, NetEase News, Tencent News
- **Project Structure**: Crawler cluster (`hotsearchcrawler`) is completely separated from analysis system (`hotsearch_analysis_agent`)
