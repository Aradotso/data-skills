---
name: llm-public-opinion-analytics-assistant
description: AI-powered public opinion analysis assistant aggregating 26 hot search lists from 15 platforms with sentiment analysis and multi-channel alerting
triggers:
  - how do I set up the public opinion analytics assistant
  - crawl hot search data from Chinese platforms
  - analyze sentiment trends across social media platforms
  - set up hot topic push notifications to WeChat or Telegram
  - cluster related news topics with LLM analysis
  - query trending topics from Weibo Bilibili or Douyin
  - configure the opinion monitoring crawler system
  - deploy the hotsearch analysis agent
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is an intelligent public opinion analysis assistant that combines real-time data from **26 hot search lists** across **15 mainstream Chinese platforms** (Weibo, Bilibili, Douyin, Baidu, Zhihu, etc.) with LLM analysis capabilities. It provides:

- **Conversational queries** for hot search rankings and topic searches
- **Topic clustering** and sentiment analysis using LLM models
- **Web crawlers** for multi-platform data collection (including video content extraction)
- **Multi-channel push notifications** (Enterprise WeChat, Telegram, Email)
- **Real-time monitoring** with keyboard shortcuts for crawler control

The system architecture separates the crawler cluster (`hotsearchcrawler`) from the analysis system (`hotsearch_analysis_agent`), storing data in MySQL for querying and analysis.

## Installation

### Prerequisites

1. **Python 3.8+** with virtual environment support
2. **MySQL 5.7+** database server
3. **Browser driver** (ChromeDriver or EdgeDriver) for content extraction

### Browser Driver Setup

**Step 1: Check browser version**
```bash
# Chrome: chrome://version
# Edge: edge://version
```

**Step 2: Download matching driver**
- Chrome: https://chromedriver.chromium.org/
- Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

**Step 3: Add driver to PATH**
```bash
# Linux/macOS
export PATH=$PATH:/path/to/driver/directory
source ~/.bashrc

# Windows: Add to System Environment Variables PATH
# C:\WebDriver\
```

**Step 4: Verify installation**
```bash
chromedriver --version
# or
msedgedriver --version
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

### Database Initialization

```python
# Reference init.py for database schema
import pymysql

connection = pymysql.connect(
    host='localhost',
    user='your_user',
    password='your_password',
    database='hotsearch_db',
    charset='utf8mb4'
)

# Create tables for hot search data
cursor = connection.cursor()
cursor.execute("""
CREATE TABLE IF NOT EXISTS hot_search (
    id INT AUTO_INCREMENT PRIMARY KEY,
    platform VARCHAR(50),
    title VARCHAR(500),
    url VARCHAR(1000),
    rank INT,
    heat_value BIGINT,
    content TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_platform (platform),
    INDEX idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
""")
connection.commit()
```

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```bash
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch_db

# LLM API Configuration (OpenAI-compatible format)
OPENAI_API_KEY=your_api_key_here
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Or use Huawei Pangu model (recommended for Chinese text)
PANGU_API_KEY=your_pangu_key
PANGU_API_BASE=https://your-pangu-endpoint

# Push Notification Channels
WECHAT_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxx
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_email_password
```

### Crawler Settings

Edit `hotsearchcrawler/settings.py`:

```python
# MySQL connection for crawler
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_user'
MYSQL_PASSWORD = 'your_password'
MYSQL_DATABASE = 'hotsearch_db'

# Optional: Add cookies for platforms requiring authentication
COOKIES = {
    'weibo': 'your_weibo_cookies',
    'bilibili': 'your_bilibili_cookies'
}

# Crawler settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
RETRY_TIMES = 3
```

## Running the System

### Start the Analysis Web Interface

```bash
python app.py
```

Access the web interface at `http://localhost:5000`

### Run Crawlers

**From web interface:**
- Use keyboard shortcuts to start/stop crawlers

**From command line (testing):**
```bash
# Test single platform crawler
python runspider-test.py --platform weibo

# Run all crawlers
python run_spiders.py
```

### Test Push Notifications

```bash
python test_push_task.py
```

## API Usage Examples

### Query Hot Search Rankings

```python
from hotsearch_analysis_agent import HotSearchAPI

api = HotSearchAPI()

# Get Weibo hot search
weibo_hot = api.get_hot_search(platform='weibo', limit=50)
for item in weibo_hot:
    print(f"#{item['rank']} {item['title']} - {item['heat_value']}")

# Search specific topic across all platforms
results = api.search_topic(keyword='人工智能', platforms=['weibo', 'zhihu', 'bilibili'])
```

### Perform Sentiment Analysis

```python
from hotsearch_analysis_agent import SentimentAnalyzer

analyzer = SentimentAnalyzer()

# Analyze sentiment of a topic
topic = "GPT-6遭提前曝光"
sentiment = analyzer.analyze(topic)
print(f"Sentiment: {sentiment['polarity']} (score: {sentiment['score']})")
# Output: Sentiment: neutral (score: 0.12)

# Batch analysis
topics = api.get_hot_search(platform='weibo', limit=10)
sentiments = analyzer.batch_analyze([t['title'] for t in topics])
```

### Topic Clustering

```python
from hotsearch_analysis_agent import TopicCluster

cluster = TopicCluster()

# Get recent news
recent_news = api.get_news(hours=24, platforms=['all'])

# Cluster related topics
clusters = cluster.group_topics(recent_news, method='semantic')
for cluster_id, topics in clusters.items():
    print(f"\nCluster {cluster_id}:")
    for topic in topics:
        print(f"  - {topic['title']} ({topic['platform']})")
```

### Generate Analysis Report

```python
from hotsearch_analysis_agent import ReportGenerator

generator = ReportGenerator()

# Generate report for specific query
report = generator.create_report(
    query="人工智能与前沿科技",
    time_range="24h",
    include_sentiment=True,
    include_clustering=True
)

print(report.to_markdown())
# Outputs formatted report with sections:
# - Core Findings
# - News Summary
# - Sentiment Analysis
# - Topic Clusters
```

### Setup Push Notifications

```python
from hotsearch_analysis_agent import PushTask

# Create push task
task = PushTask(
    name="AI Technology Monitoring",
    query="人工智能 OR 大模型",
    channels=["wechat", "telegram"],
    schedule="0 */2 * * *",  # Every 2 hours
    threshold={"heat_value": 100000}
)

task.save()
task.activate()

# Test immediate push
task.execute_now()
```

## Web Interface Usage

### Conversational Query

```
User: 查询微博热搜前10
Assistant: [Displays top 10 Weibo trends with links]

User: 搜索关于人工智能的新闻
Assistant: [Shows AI-related news across platforms]

User: 对这些新闻进行情感分析
Assistant: [Provides sentiment breakdown]
```

### Keyboard Shortcuts

- `Ctrl+S`: Start all crawlers
- `Ctrl+E`: Stop all crawlers
- `Ctrl+R`: Refresh data
- `Ctrl+P`: Open push task settings

## Common Patterns

### Custom Crawler for New Platform

```python
# hotsearchcrawler/spiders/new_platform.py
import scrapy
from hotsearchcrawler.items import HotSearchItem

class NewPlatformSpider(scrapy.Spider):
    name = 'new_platform'
    start_urls = ['https://newplatform.com/trending']
    
    def parse(self, response):
        for idx, item in enumerate(response.css('.trend-item')):
            yield HotSearchItem(
                platform='new_platform',
                title=item.css('.title::text').get(),
                url=item.css('a::attr(href)').get(),
                rank=idx + 1,
                heat_value=int(item.css('.heat::text').get()),
                content=None  # Will be fetched by detail spider
            )
```

### Custom LLM Analysis Function

```python
from hotsearch_analysis_agent import LLMAnalyzer

class CustomAnalyzer(LLMAnalyzer):
    def analyze_controversy(self, topic):
        prompt = f"""
        Analyze the controversy level of this topic: {topic}
        Consider: public debate intensity, polarization, media coverage.
        Output: controversy_score (0-10) and explanation.
        """
        response = self.call_llm(prompt)
        return self.parse_response(response)
```

### Scheduled Monitoring Task

```python
from apscheduler.schedulers.background import BackgroundScheduler
from hotsearch_analysis_agent import MonitoringTask

def monitor_keywords():
    task = MonitoringTask(keywords=['疫情', '经济政策'])
    alerts = task.check_for_alerts()
    if alerts:
        task.push_alerts(alerts, channels=['wechat', 'email'])

scheduler = BackgroundScheduler()
scheduler.add_job(monitor_keywords, 'cron', hour='*/3')
scheduler.start()
```

## Troubleshooting

### Crawler Issues

**Problem**: Crawler fails to extract content from video platforms
```python
# Solution: Ensure browser driver is properly configured
# Check hotsearchcrawler/middlewares.py
from selenium import webdriver
from selenium.webdriver.chrome.options import Options

options = Options()
options.add_argument('--headless')
options.add_argument('--disable-gpu')
driver = webdriver.Chrome(options=options)
```

**Problem**: Rate limiting / IP blocking
```python
# Add to settings.py
DOWNLOADER_MIDDLEWARES = {
    'scrapy.downloadermiddlewares.useragent.UserAgentMiddleware': None,
    'scrapy_user_agents.middlewares.RandomUserAgentMiddleware': 400,
    'scrapy.downloadermiddlewares.retry.RetryMiddleware': 550,
}
RETRY_HTTP_CODES = [500, 502, 503, 504, 408, 429]
```

### Database Connection Issues

```python
# Test connection
import pymysql

try:
    conn = pymysql.connect(
        host='localhost',
        user='user',
        password='pass',
        database='hotsearch_db'
    )
    print("Connection successful")
except Exception as e:
    print(f"Connection failed: {e}")
    # Check: MySQL service running, credentials, firewall
```

### LLM API Issues

```python
# Test API connection
import openai
import os

openai.api_key = os.getenv('OPENAI_API_KEY')
openai.api_base = os.getenv('OPENAI_API_BASE')

try:
    response = openai.ChatCompletion.create(
        model="gpt-4",
        messages=[{"role": "user", "content": "test"}]
    )
    print("API working")
except Exception as e:
    print(f"API error: {e}")
    # Check: API key validity, endpoint URL, network connectivity
```

### Push Notification Failures

```python
# Test WeChat webhook
import requests

webhook_url = os.getenv('WECHAT_WEBHOOK')
response = requests.post(webhook_url, json={
    "msgtype": "text",
    "text": {"content": "Test message"}
})
print(response.status_code, response.json())

# Test Telegram
from telegram import Bot
bot = Bot(token=os.getenv('TELEGRAM_BOT_TOKEN'))
bot.send_message(chat_id=os.getenv('TELEGRAM_CHAT_ID'), text="Test")
```

## Advanced Configuration

### Using Huawei Pangu Model (Recommended for Chinese)

The project recommends Huawei's OpenPangu model for better Chinese language understanding:

```python
# Download model: https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

# Configure in .env
PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model
USE_LOCAL_MODEL=true

# In code
from hotsearch_analysis_agent import PanguAnalyzer

analyzer = PanguAnalyzer(model_path=os.getenv('PANGU_MODEL_PATH'))
result = analyzer.analyze_sentiment("这个政策对经济影响深远")
```

### Multi-Language Support

```python
# Add language detection
from langdetect import detect

def analyze_with_language_detection(text):
    lang = detect(text)
    if lang == 'zh-cn':
        analyzer = PanguAnalyzer()
    else:
        analyzer = OpenAIAnalyzer()
    return analyzer.analyze(text)
```

This skill covers the core functionality needed to deploy and use the LLM-Based Public Opinion Analytics Assistant for monitoring Chinese social media platforms and generating intelligent analysis reports.
