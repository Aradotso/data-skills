---
name: llm-public-opinion-analytics-assistant
description: A Python-based public opinion analytics assistant that integrates 26 real-time hot search lists from 15 mainstream platforms with LLM analysis capabilities for sentiment analysis, topic clustering, and multi-channel alerts.
triggers:
  - how do I analyze public opinion trends with LLM
  - set up a hot search monitoring system
  - crawl and analyze weibo hot topics with AI
  - build sentiment analysis for social media trends
  - configure multi-platform hot search aggregation
  - deploy public opinion analytics with push notifications
  - analyze trending topics across multiple platforms
  - set up automated hot topic alerts
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is an intelligent public opinion analytics system that combines real-time data from **26 hot search lists** across **15 mainstream platforms** (Weibo, Bilibili, Douyin, Baidu, etc.) with large language model analysis capabilities. It provides conversational hot search queries, topic clustering, sentiment analysis, and multi-channel push notifications (Email, WeChat, Enterprise WeChat, Telegram).

**Key capabilities:**
- Multi-platform hot search data crawling with Scrapy
- LLM-powered sentiment analysis and topic clustering
- Conversational interface for querying trends
- Automated hot topic push to multiple channels
- Video content extraction and analysis
- Keyboard shortcut control for crawler management

## Installation

### Prerequisites

```bash
# Python 3.8+
# MySQL database
# Edge or Chrome browser + corresponding WebDriver
```

### 1. Browser Driver Setup

**Download the appropriate driver:**
- Chrome: https://chromedriver.chromium.org/
- Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

**Place driver in system PATH or browser directory:**

```bash
# Linux/macOS - move to PATH
sudo mv chromedriver /usr/local/bin/
sudo chmod +x /usr/local/bin/chromedriver

# Verify installation
chromedriver --version
```

### 2. Install Dependencies

```bash
# Clone repository
git clone https://github.com/hmmnxkl/LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant.git
cd LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant

# Create virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install requirements
pip install -r requirements.txt
```

### 3. Database Setup

```bash
# Create MySQL database
mysql -u root -p

CREATE DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

**Initialize tables using `init.py`:**

```python
# Reference structure from init.py
# Tables: hot_search_data, analysis_results, push_tasks, etc.
python init.py
```

### 4. Configuration

**Create `.env` file in project root:**

```bash
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_username
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
OPENAI_MODEL=gpt-4

# Or use Huawei Pangu model
PANGU_API_KEY=your_pangu_key
PANGU_API_BASE=https://pangu-api-endpoint

# Push Notification Configuration
# Email (SMTP)
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
SMTP_RECIPIENT=recipient@example.com

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Enterprise WeChat
WECHAT_CORP_ID=your_corp_id
WECHAT_CORP_SECRET=your_corp_secret
WECHAT_AGENT_ID=your_agent_id
WECHAT_WEBHOOK_URL=your_webhook_url
```

**Configure crawler settings in `hotsearchcrawler/settings.py`:**

```python
# Database connection
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_username'
MYSQL_PASSWORD = 'your_password'
MYSQL_DATABASE = 'hotsearch'

# Optional: Add cookies for specific platforms
COOKIES = {
    'weibo': 'your_weibo_cookies',
    'bilibili': 'your_bilibili_cookies'
}

# User Agent
USER_AGENT = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'

# Concurrent requests
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
```

## Running the System

### Start the Web Application

```bash
# Run main application
python app.py

# Access at http://localhost:5000
```

### Start Crawlers

**Via Web Interface:**
- Use keyboard shortcuts in the web UI to start/stop crawlers

**Via Command Line:**

```bash
# Test individual crawler
python runspider-test.py

# Run all spiders
python run_spiders.py
```

### Test Push Notifications

```bash
# Test push task configuration
python test_push_task.py
```

## Core Usage Patterns

### 1. Querying Hot Search Data

```python
from hotsearch_analysis_agent.database import DatabaseManager
from hotsearch_analysis_agent.analyzer import TrendAnalyzer

# Initialize database connection
db = DatabaseManager(
    host='localhost',
    user='your_user',
    password='your_password',
    database='hotsearch'
)

# Query recent hot searches from Weibo
weibo_trends = db.query("""
    SELECT title, hot_value, url, platform, create_time
    FROM hot_search_data
    WHERE platform = 'weibo'
    ORDER BY create_time DESC
    LIMIT 10
""")

for trend in weibo_trends:
    print(f"{trend['title']} - 热度: {trend['hot_value']}")
```

### 2. LLM-Based Sentiment Analysis

```python
from hotsearch_analysis_agent.llm_client import LLMClient
from hotsearch_analysis_agent.analyzer import SentimentAnalyzer

# Initialize LLM client
llm = LLMClient(
    api_key=os.getenv('OPENAI_API_KEY'),
    api_base=os.getenv('OPENAI_API_BASE'),
    model=os.getenv('OPENAI_MODEL', 'gpt-4')
)

# Analyze sentiment of a topic
analyzer = SentimentAnalyzer(llm)

topic_title = "人工智能技术突破"
topic_content = """
DeepSeek V4采用华为算力,国产芯片生态走到哪一步了?
相关讨论内容...
"""

sentiment_result = analyzer.analyze_sentiment(
    title=topic_title,
    content=topic_content
)

print(f"情感倾向: {sentiment_result['sentiment']}")  # positive/negative/neutral
print(f"置信度: {sentiment_result['confidence']}")
print(f"关键观点: {sentiment_result['key_points']}")
```

### 3. Topic Clustering

```python
from hotsearch_analysis_agent.analyzer import TopicClusterer

# Get recent hot searches
recent_trends = db.query("""
    SELECT title, content, platform
    FROM hot_search_data
    WHERE create_time >= DATE_SUB(NOW(), INTERVAL 24 HOUR)
""")

# Cluster related topics
clusterer = TopicClusterer(llm)
clusters = clusterer.cluster_topics(recent_trends)

for cluster in clusters:
    print(f"\n主题: {cluster['theme']}")
    print(f"相关条目数: {len(cluster['items'])}")
    for item in cluster['items']:
        print(f"  - {item['title']} ({item['platform']})")
```

### 4. Conversational Query Interface

```python
from hotsearch_analysis_agent.chat_interface import ChatBot

chatbot = ChatBot(db, llm)

# User query
user_query = "最近关于人工智能的热点有哪些?"

response = chatbot.process_query(user_query)
print(response)

# Example output:
# 根据最近24小时的数据,人工智能相关热点包括:
# 1. GPT-6遭提前曝光, 2M超长上下文来了 (Bilibili, 热度: 125万)
# 2. DeepSeek V4采用华为算力 (今日头条, 热度: 89万)
# 3. Anthropic年化收入暴涨至300亿美元 (新浪科技, 热度: 67万)
```

### 5. Configuring Push Tasks

```python
from hotsearch_analysis_agent.push_manager import PushTaskManager

push_manager = PushTaskManager(db)

# Create a push task for AI-related topics
task = push_manager.create_task(
    name="AI热点推送",
    keywords=["人工智能", "大模型", "AI", "GPT"],
    channels=["email", "telegram", "wechat"],
    schedule="0 9,18 * * *",  # Twice daily at 9 AM and 6 PM
    sentiment_filter="all",  # or "positive", "negative"
    min_hot_value=100000
)

print(f"任务已创建: {task['id']}")

# Test the push task
push_manager.test_push(task['id'])
```

### 6. Multi-Channel Push Implementation

```python
from hotsearch_analysis_agent.push_channels import (
    EmailPusher, TelegramPusher, WeChatPusher
)

# Email push
email_pusher = EmailPusher(
    smtp_server=os.getenv('SMTP_SERVER'),
    smtp_port=int(os.getenv('SMTP_PORT')),
    username=os.getenv('SMTP_USER'),
    password=os.getenv('SMTP_PASSWORD')
)

email_pusher.send_report(
    recipient=os.getenv('SMTP_RECIPIENT'),
    subject="AI与前沿科技热点分析",
    content=report_html
)

# Telegram push
telegram_pusher = TelegramPusher(
    bot_token=os.getenv('TELEGRAM_BOT_TOKEN'),
    chat_id=os.getenv('TELEGRAM_CHAT_ID')
)

telegram_pusher.send_message(
    text=report_markdown,
    parse_mode='Markdown'
)

# WeChat enterprise push
wechat_pusher = WeChatPusher(
    corp_id=os.getenv('WECHAT_CORP_ID'),
    corp_secret=os.getenv('WECHAT_CORP_SECRET'),
    agent_id=os.getenv('WECHAT_AGENT_ID')
)

wechat_pusher.send_text_card(
    title="舆情分析报告",
    description=report_summary,
    url=report_url
)
```

### 7. Custom Crawler Development

```python
# hotsearchcrawler/spiders/custom_spider.py
import scrapy
from hotsearchcrawler.items import HotSearchItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    allowed_domains = ['example.com']
    start_urls = ['https://example.com/hot-list']
    
    def parse(self, response):
        for item in response.css('.hot-item'):
            hot_search = HotSearchItem()
            hot_search['title'] = item.css('.title::text').get()
            hot_search['hot_value'] = item.css('.hot-value::text').get()
            hot_search['url'] = item.css('a::attr(href)').get()
            hot_search['platform'] = 'custom_platform'
            hot_search['rank'] = item.css('.rank::text').get()
            
            # Fetch detail page for content extraction
            if hot_search['url']:
                yield response.follow(
                    hot_search['url'],
                    callback=self.parse_detail,
                    meta={'item': hot_search}
                )
    
    def parse_detail(self, response):
        item = response.meta['item']
        item['content'] = response.css('.article-content::text').getall()
        item['author'] = response.css('.author::text').get()
        yield item
```

## Configuration Files

### Scrapy Settings (`hotsearchcrawler/settings.py`)

```python
BOT_NAME = 'hotsearchcrawler'

SPIDER_MODULES = ['hotsearchcrawler.spiders']
NEWSPIDER_MODULE = 'hotsearchcrawler.spiders'

# Pipelines
ITEM_PIPELINES = {
    'hotsearchcrawler.pipelines.MySQLPipeline': 300,
}

# Retry settings
RETRY_TIMES = 3
RETRY_HTTP_CODES = [500, 502, 503, 504, 408, 429]

# AutoThrottle
AUTOTHROTTLE_ENABLED = True
AUTOTHROTTLE_START_DELAY = 1
AUTOTHROTTLE_MAX_DELAY = 10
```

### Analysis System Config (`hotsearch_analysis_agent/config.py`)

```python
import os
from dotenv import load_dotenv

load_dotenv()

# Database
DB_CONFIG = {
    'host': os.getenv('MYSQL_HOST', 'localhost'),
    'port': int(os.getenv('MYSQL_PORT', 3306)),
    'user': os.getenv('MYSQL_USER'),
    'password': os.getenv('MYSQL_PASSWORD'),
    'database': os.getenv('MYSQL_DATABASE'),
    'charset': 'utf8mb4'
}

# LLM
LLM_CONFIG = {
    'api_key': os.getenv('OPENAI_API_KEY'),
    'api_base': os.getenv('OPENAI_API_BASE'),
    'model': os.getenv('OPENAI_MODEL', 'gpt-4'),
    'temperature': 0.7,
    'max_tokens': 4000
}

# Supported platforms
PLATFORMS = [
    'weibo', 'bilibili', 'douyin', 'baidu', 'zhihu',
    'toutiao', 'sina', 'tencent', 'netease', 'ifeng',
    'thepaper', 'xiaohongshu', 'kuaishou', 'douban', 'tieba'
]
```

## Common Patterns

### Pattern 1: Scheduled Analysis Reports

```python
from apscheduler.schedulers.background import BackgroundScheduler
from hotsearch_analysis_agent.report_generator import ReportGenerator

scheduler = BackgroundScheduler()
report_gen = ReportGenerator(db, llm)

def generate_daily_report():
    """Generate and push daily hot topic report"""
    report = report_gen.generate_report(
        keywords=["人工智能", "科技", "经济"],
        time_range="24h",
        include_sentiment=True,
        include_clustering=True
    )
    
    # Push to all configured channels
    push_manager.push_report(report)

# Schedule daily at 9 AM
scheduler.add_job(generate_daily_report, 'cron', hour=9, minute=0)
scheduler.start()
```

### Pattern 2: Real-time Keyword Monitoring

```python
from hotsearch_analysis_agent.monitor import KeywordMonitor

monitor = KeywordMonitor(db, llm)

# Monitor specific keywords
keywords = ["突发", "重大", "紧急"]

def on_keyword_detected(trend):
    """Callback when keyword is detected"""
    print(f"检测到关键词: {trend['title']}")
    
    # Immediate analysis
    analysis = analyzer.quick_analyze(trend)
    
    # Push alert
    if analysis['urgency'] == 'high':
        push_manager.push_alert(
            title=f"紧急舆情: {trend['title']}",
            content=analysis['summary'],
            channels=['telegram', 'wechat']
        )

monitor.watch(keywords, callback=on_keyword_detected, interval=300)
```

### Pattern 3: Video Content Extraction

```python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

def extract_video_content(url):
    """Extract content from video pages (Bilibili, Douyin, etc.)"""
    options = webdriver.ChromeOptions()
    options.add_argument('--headless')
    driver = webdriver.Chrome(options=options)
    
    try:
        driver.get(url)
        
        # Wait for content to load
        wait = WebDriverWait(driver, 10)
        
        # Extract title, description, comments
        title = wait.until(
            EC.presence_of_element_located((By.CSS_SELECTOR, '.video-title'))
        ).text
        
        description = driver.find_element(By.CSS_SELECTOR, '.desc-info').text
        
        # Extract comments for sentiment analysis
        comments = [
            el.text for el in driver.find_elements(By.CSS_SELECTOR, '.comment-text')
        ]
        
        return {
            'title': title,
            'description': description,
            'comments': comments[:50]  # Top 50 comments
        }
    finally:
        driver.quit()
```

## Troubleshooting

### Issue: WebDriver Not Found

```bash
# Error: selenium.common.exceptions.WebDriverException: 'chromedriver' executable needs to be in PATH

# Solution: Verify driver installation
which chromedriver  # Linux/macOS
where chromedriver  # Windows

# If not found, reinstall driver to PATH location
```

### Issue: MySQL Connection Failed

```python
# Error: pymysql.err.OperationalError: (2003, "Can't connect to MySQL server")

# Check MySQL service
# Linux:
sudo systemctl status mysql

# Verify credentials in .env
# Test connection
import pymysql
conn = pymysql.connect(
    host=os.getenv('MYSQL_HOST'),
    user=os.getenv('MYSQL_USER'),
    password=os.getenv('MYSQL_PASSWORD')
)
```

### Issue: LLM API Rate Limit

```python
from tenacity import retry, wait_exponential, stop_after_attempt

@retry(wait=wait_exponential(min=1, max=60), stop=stop_after_attempt(5))
def call_llm_with_retry(prompt):
    """Retry LLM calls with exponential backoff"""
    return llm.generate(prompt)

# Use in analysis
result = call_llm_with_retry("Analyze this topic...")
```

### Issue: Crawler Blocked by Platform

```python
# Add random delays and rotate user agents
import random

USER_AGENTS = [
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64)...',
    'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)...',
    # Add more user agents
]

# In spider settings
DOWNLOADER_MIDDLEWARES = {
    'scrapy.downloadermiddlewares.useragent.UserAgentMiddleware': None,
    'scrapy_user_agents.middlewares.RandomUserAgentMiddleware': 400,
}

RANDOM_UA_PER_PROXY = True
DOWNLOAD_DELAY = random.uniform(1, 3)
```

### Issue: Push Notification Failed

```python
# Email: Check SMTP settings and app password
# Telegram: Verify bot token and chat ID
# WeChat: Ensure corp_id, secret, and agent_id are correct

# Test individual channels
from hotsearch_analysis_agent.push_channels import test_all_channels

results = test_all_channels()
for channel, status in results.items():
    if not status['success']:
        print(f"{channel} failed: {status['error']}")
```

## Best Practices

1. **Rate Limiting**: Implement delays between crawler requests to avoid IP bans
2. **Data Freshness**: Run crawlers at optimal intervals (15-30 minutes for hot searches)
3. **LLM Caching**: Cache similar queries to reduce API costs
4. **Error Logging**: Use comprehensive logging for debugging crawler and analysis issues
5. **Backup Strategy**: Regular database backups before schema changes
6. **API Key Rotation**: Use multiple API keys for high-volume LLM requests

## Additional Resources

- Huawei Pangu Model: https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model
- Scrapy Documentation: https://docs.scrapy.org/
- Selenium WebDriver: https://www.selenium.dev/documentation/webdriver/
