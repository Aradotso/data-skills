---
name: llm-public-opinion-analytics-assistant
description: Multi-platform hot topic crawler and LLM-powered public opinion analysis system with sentiment analysis, clustering, and multi-channel alerting
triggers:
  - set up public opinion monitoring system
  - analyze social media hot topics with LLM
  - crawl trending news from multiple platforms
  - configure sentiment analysis for public opinion
  - create hot topic alert notifications
  - deploy Chinese language opinion analytics
  - integrate Pangu LLM for text analysis
  - monitor trending topics across platforms
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

An intelligent public opinion analysis system that combines real-time data crawling from 15 mainstream Chinese platforms (26 trending lists) with LLM-powered analytics. Features conversational hot topic queries, theme-based search, topic clustering, sentiment analysis, and multi-channel alert notifications (Email, WeChat, Enterprise WeChat, Telegram).

## What It Does

- **Multi-Platform Crawling**: Collects trending topics from Weibo, Bilibili, Zhihu, Baidu, Toutiao, and 10+ other platforms
- **LLM Analysis**: Uses Huawei Pangu (or OpenAI-compatible models) for sentiment analysis, topic clustering, and content summarization
- **Video Content Extraction**: Extracts insights even from video-based news content
- **Multi-Channel Alerts**: Pushes analytical reports via Email, WeChat Work, Telegram, or personal WeChat
- **Conversational Interface**: Natural language queries for trending topics and analysis
- **Hotkey Controls**: Start/stop crawlers via keyboard shortcuts

## Architecture

```
├── hotsearch_analysis_agent/    # Analysis system (LLM-powered)
├── hotsearchcrawler/            # Scrapy-based crawler cluster
├── app.py                       # Main application entry
├── run_spiders.py              # Crawler orchestration
├── test_push_task.py           # Test notification channels
└── init.py                      # Database initialization
```

## Installation

### Prerequisites

**1. Browser Driver Setup**

The system requires a WebDriver for content extraction:

```bash
# Check your Chrome/Edge version first
# Chrome: chrome://settings/help
# Edge: edge://settings/help

# Download matching driver:
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Add to PATH (Linux/macOS)
sudo mv chromedriver /usr/local/bin/
chmod +x /usr/local/bin/chromedriver

# Verify installation
chromedriver --version
```

**2. MySQL Database**

```bash
# Install MySQL 5.7+
# Create database and tables using init.py as reference

mysql -u root -p
CREATE DATABASE public_opinion CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

**3. Python Environment**

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Configuration

**1. Environment Variables (`.env`)**

```bash
# Database Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=public_opinion

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=${OPENAI_API_KEY}
OPENAI_API_BASE=https://api.openai.com/v1  # Or local Pangu endpoint
MODEL_NAME=gpt-4  # Or pangu-embedded-7b

# Notification Channels
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=${EMAIL_USER}
SMTP_PASSWORD=${EMAIL_PASSWORD}

WECHAT_WORK_WEBHOOK=${WECHAT_WEBHOOK}
TELEGRAM_BOT_TOKEN=${TELEGRAM_TOKEN}
TELEGRAM_CHAT_ID=${TELEGRAM_CHAT_ID}
```

**2. Crawler Settings (`hotsearchcrawler/settings.py`)**

```python
# MySQL Connection
MYSQL_CONFIG = {
    'host': os.getenv('MYSQL_HOST', 'localhost'),
    'user': os.getenv('MYSQL_USER'),
    'password': os.getenv('MYSQL_PASSWORD'),
    'database': os.getenv('MYSQL_DATABASE'),
    'charset': 'utf8mb4'
}

# Optional: Platform-specific cookies
COOKIES = {
    'weibo': 'your_cookie_string',  # Optional for some platforms
    'bilibili': 'your_cookie_string'
}

# Crawler Behavior
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 2
ROBOTSTXT_OBEY = False
```

## Database Initialization

```python
# init.py - Create required tables
import pymysql

connection = pymysql.connect(
    host=os.getenv('MYSQL_HOST'),
    user=os.getenv('MYSQL_USER'),
    password=os.getenv('MYSQL_PASSWORD'),
    database=os.getenv('MYSQL_DATABASE')
)

cursor = connection.cursor()

# Hot topics table
cursor.execute("""
    CREATE TABLE IF NOT EXISTS hot_topics (
        id INT AUTO_INCREMENT PRIMARY KEY,
        platform VARCHAR(50),
        title VARCHAR(500),
        url TEXT,
        rank INT,
        heat_value VARCHAR(100),
        detail_content TEXT,
        crawled_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        INDEX idx_platform (platform),
        INDEX idx_crawled_at (crawled_at)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
""")

# Analysis results table
cursor.execute("""
    CREATE TABLE IF NOT EXISTS analysis_results (
        id INT AUTO_INCREMENT PRIMARY KEY,
        query_text TEXT,
        sentiment VARCHAR(50),
        cluster_topics JSON,
        summary TEXT,
        analyzed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
""")

connection.commit()
connection.close()
```

## Running the System

### Start the Analysis System

```bash
# Main application with web interface
python app.py
# Access at http://localhost:5000
```

### Run Crawlers

```bash
# Test individual crawler
python runspider-test.py

# Start all crawlers (triggered from web UI or directly)
python run_spiders.py

# Run specific platform crawler
cd hotsearchcrawler
scrapy crawl weibo_spider
scrapy crawl bilibili_spider
scrapy crawl zhihu_spider
```

### Test Notification Channels

```bash
# Test all configured push channels
python test_push_task.py
```

## Key Usage Patterns

### 1. Query Hot Topics via API

```python
import requests

# Query trending topics
response = requests.post('http://localhost:5000/api/query', json={
    'query': '人工智能相关的热点新闻',
    'platforms': ['weibo', 'zhihu', 'toutiao'],
    'limit': 20
})

results = response.json()
for topic in results['topics']:
    print(f"{topic['platform']}: {topic['title']} (热度: {topic['heat_value']})")
```

### 2. Sentiment Analysis with LLM

```python
from hotsearch_analysis_agent.llm_client import LLMAnalyzer

analyzer = LLMAnalyzer(
    api_key=os.getenv('OPENAI_API_KEY'),
    base_url=os.getenv('OPENAI_API_BASE'),
    model=os.getenv('MODEL_NAME')
)

# Analyze sentiment
text = "这个政策真是太好了,解决了民生大问题"
sentiment = analyzer.analyze_sentiment(text)
print(f"情感倾向: {sentiment['sentiment']}")  # positive/negative/neutral
print(f"置信度: {sentiment['confidence']}")

# Topic clustering
topics = ["AI发展", "芯片技术", "量子计算", "AI伦理", "算力"]
clusters = analyzer.cluster_topics(topics)
# Output: {"技术类": ["AI发展", "芯片技术", "量子计算"], "社会类": ["AI伦理"]}
```

### 3. Custom Crawler for New Platform

```python
# hotsearchcrawler/spiders/custom_spider.py
import scrapy
from ..items import HotTopicItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://example.com/trending']
    
    def parse(self, response):
        for item in response.css('.trending-item'):
            topic = HotTopicItem()
            topic['platform'] = 'custom_platform'
            topic['title'] = item.css('.title::text').get()
            topic['url'] = item.css('a::attr(href)').get()
            topic['rank'] = item.css('.rank::text').get()
            topic['heat_value'] = item.css('.heat::text').get()
            
            # Fetch detail page content
            yield scrapy.Request(
                topic['url'],
                callback=self.parse_detail,
                meta={'topic': topic}
            )
    
    def parse_detail(self, response):
        topic = response.meta['topic']
        topic['detail_content'] = response.css('.content::text').getall()
        yield topic
```

### 4. Generate Analysis Report

```python
from hotsearch_analysis_agent.report_generator import ReportGenerator

generator = ReportGenerator(llm_analyzer=analyzer)

# Generate comprehensive report
report = generator.create_report(
    query="人工智能与前沿科技",
    time_range="2026-04-01 to 2026-04-07",
    platforms=['weibo', 'zhihu', 'bilibili', 'toutiao']
)

print(report['markdown'])  # Markdown-formatted report
```

### 5. Configure Push Notifications

```python
# hotsearch_analysis_agent/push_manager.py
from .push_channels import EmailPusher, WeChatWorkPusher, TelegramPusher

# Email notification
email_pusher = EmailPusher(
    smtp_host=os.getenv('SMTP_HOST'),
    smtp_port=int(os.getenv('SMTP_PORT')),
    username=os.getenv('SMTP_USER'),
    password=os.getenv('SMTP_PASSWORD')
)

email_pusher.send_report(
    to=['analyst@company.com'],
    subject='舆情分析报告 - AI热点',
    content=report['markdown']
)

# WeChat Work webhook
wechat_pusher = WeChatWorkPusher(
    webhook_url=os.getenv('WECHAT_WORK_WEBHOOK')
)
wechat_pusher.send_markdown(report['markdown'])

# Telegram bot
telegram_pusher = TelegramPusher(
    bot_token=os.getenv('TELEGRAM_BOT_TOKEN'),
    chat_id=os.getenv('TELEGRAM_CHAT_ID')
)
telegram_pusher.send_message(report['summary'])
```

### 6. Schedule Periodic Analysis Tasks

```python
import schedule
import time

def daily_analysis_task():
    # Crawl latest data
    os.system('python run_spiders.py')
    
    # Generate report
    report = generator.create_report(
        query="今日热点",
        time_range="last_24_hours"
    )
    
    # Push to all channels
    email_pusher.send_report(to=['team@company.com'], 
                            subject='每日舆情简报', 
                            content=report['markdown'])
    wechat_pusher.send_markdown(report['markdown'])

# Run daily at 9 AM
schedule.every().day.at("09:00").do(daily_analysis_task)

while True:
    schedule.run_pending()
    time.sleep(60)
```

## LLM Integration (Pangu Model)

### Local Deployment

```bash
# Download Pangu model from GitCode
wget https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

# Serve with vLLM or similar OpenAI-compatible server
python -m vllm.entrypoints.openai.api_server \
    --model /path/to/openpangu-embedded-7b-model \
    --host 0.0.0.0 \
    --port 8000
```

### Configure for Pangu

```bash
# .env
OPENAI_API_BASE=http://localhost:8000/v1
MODEL_NAME=openpangu-embedded-7b
```

### Why Pangu?

- **Strong Chinese Language Understanding**: Better handling of sarcasm, implicit sentiment
- **Domain Adaptation**: Superior performance on tech, finance, policy domains
- **Privacy**: Local deployment for sensitive opinion data
- **Long Context**: Handles lengthy news articles and forum posts effectively

## Troubleshooting

### Crawler Issues

**Problem**: Crawlers fail to collect data

```bash
# Check WebDriver is accessible
which chromedriver

# Test individual spider with verbose logging
cd hotsearchcrawler
scrapy crawl weibo_spider -L DEBUG

# Verify database connection
python -c "import pymysql; pymysql.connect(host='localhost', user='root', password='pass')"
```

**Problem**: Rate limiting or blocked requests

```python
# Adjust settings.py
DOWNLOAD_DELAY = 5  # Increase delay
CONCURRENT_REQUESTS = 4  # Reduce concurrency
RETRY_TIMES = 3

# Add random user agents
USER_AGENT_LIST = [
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64)...',
    'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)...'
]
```

### LLM Analysis Issues

**Problem**: Model responses are poor quality

```python
# Increase temperature for creativity
analyzer = LLMAnalyzer(model='pangu', temperature=0.7)

# Use explicit prompts
prompt = """
分析以下新闻的情感倾向(积极/消极/中性):

新闻: {text}

请仅返回JSON格式: {{"sentiment": "积极", "confidence": 0.95}}
"""
```

**Problem**: Timeout on long texts

```python
# Implement chunking for long content
def analyze_long_text(text, max_length=4000):
    chunks = [text[i:i+max_length] for i in range(0, len(text), max_length)]
    sentiments = [analyzer.analyze_sentiment(chunk) for chunk in chunks]
    # Aggregate results
    return aggregate_sentiments(sentiments)
```

### Database Performance

```sql
-- Add indexes for common queries
CREATE INDEX idx_platform_time ON hot_topics(platform, crawled_at);
CREATE INDEX idx_title_fulltext ON hot_topics(title) USING FULLTEXT;

-- Archive old data
DELETE FROM hot_topics WHERE crawled_at < DATE_SUB(NOW(), INTERVAL 30 DAY);
```

### Notification Failures

```python
# Test each channel separately
python test_push_task.py --channel email
python test_push_task.py --channel wechat
python test_push_task.py --channel telegram

# Enable debug logging
import logging
logging.basicConfig(level=logging.DEBUG)
```

## Advanced Patterns

### Multi-Query Topic Clustering

```python
# Compare multiple time periods
topics_today = fetch_topics(date='2026-04-07')
topics_yesterday = fetch_topics(date='2026-04-06')

# Detect trending shifts
trends = analyzer.compare_topic_sets(topics_today, topics_yesterday)
print(f"新兴话题: {trends['emerging']}")
print(f"热度下降: {trends['declining']}")
```

### Cross-Platform Topic Correlation

```python
# Find same topics across platforms
from collections import defaultdict

topic_map = defaultdict(list)
for topic in all_topics:
    normalized = analyzer.normalize_topic(topic['title'])
    topic_map[normalized].append(topic)

# Identify viral topics
viral = {k: v for k, v in topic_map.items() if len(v) >= 3}
```

### Custom Alert Conditions

```python
# Alert on specific sentiment + keyword combination
def check_alert_conditions(topics):
    alerts = []
    for topic in topics:
        sentiment = analyzer.analyze_sentiment(topic['detail_content'])
        if '公司名' in topic['title'] and sentiment['sentiment'] == 'negative':
            alerts.append({
                'topic': topic['title'],
                'severity': 'HIGH',
                'platforms': [topic['platform']]
            })
    return alerts

# Push urgent alerts
if alerts:
    telegram_pusher.send_urgent_alert(alerts)
```

This skill provides comprehensive coverage of the LLM-based public opinion analytics system, from installation to advanced usage patterns for AI coding agents assisting developers.
