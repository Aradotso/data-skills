---
name: llm-public-opinion-analytics-assistant
description: Multi-platform hot search crawler and LLM-powered public opinion analysis system with sentiment analysis, topic clustering, and multi-channel alerts
triggers:
  - set up public opinion monitoring system
  - analyze social media trends with LLM
  - crawl hot search rankings from Chinese platforms
  - configure sentiment analysis for news topics
  - create topic clustering dashboard for trending news
  - set up automated opinion reports with push notifications
  - monitor Weibo Bilibili trending topics
  - build real-time public opinion analytics
---

# LLM Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive public opinion monitoring system that crawls 26 trending lists from 15 major Chinese platforms (Weibo, Bilibili, Douyin, Baidu, etc.) and uses LLM analysis for sentiment detection, topic clustering, and automated reporting. Features conversational query interface, multi-channel push notifications (WeChat, Telegram, email), and real-time trend analysis.

## What It Does

- **Multi-Platform Crawling**: Scrapes real-time trending topics from 15 platforms including Weibo, Bilibili, Douyin, Zhihu, Baidu Hot Search
- **LLM Analysis**: Uses large language models (supports Huawei Pangu, OpenAI-compatible APIs) for sentiment analysis, topic clustering, and content summarization
- **Conversational Interface**: Natural language queries for trend analysis ("Show me AI-related trending topics")
- **Content Extraction**: Extracts full article content and even video transcripts for deep analysis
- **Multi-Channel Alerts**: Push reports via email, WeChat Work, Telegram, or WeChat personal
- **Clustering & Visualization**: Groups related topics and presents sentiment trends

## Installation

### Prerequisites

1. **Browser Driver Setup** (required for content extraction):

```bash
# Download ChromeDriver or EdgeDriver matching your browser version
# ChromeDriver: https://chromedriver.chromium.org/
# EdgeDriver: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# macOS/Linux: Place driver in PATH
sudo mv chromedriver /usr/local/bin/
chmod +x /usr/local/bin/chromedriver

# Windows: Add driver directory to PATH environment variable
# Verify installation
chromedriver --version
```

2. **MySQL Database**:

```bash
# Install MySQL 8.0+
# Create database
mysql -u root -p -e "CREATE DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
```

3. **Python Environment**:

```bash
git clone https://github.com/hmmnxkl/LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant.git
cd LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### Database Initialization

```python
# Reference init.py for table schema
# Key tables: hot_searches, news_content, analysis_results, push_tasks

import pymysql

connection = pymysql.connect(
    host='localhost',
    user='root',
    password='your_password',
    database='hotsearch',
    charset='utf8mb4'
)

# Create tables (see init.py for full schema)
with connection.cursor() as cursor:
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS hot_searches (
            id INT AUTO_INCREMENT PRIMARY KEY,
            platform VARCHAR(50),
            title VARCHAR(500),
            url TEXT,
            hot_value VARCHAR(100),
            rank_position INT,
            crawl_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            INDEX idx_platform_time (platform, crawl_time)
        ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
    """)
    connection.commit()
```

## Configuration

### Core Configuration Files

**1. Crawler Settings** (`hotsearchcrawler/settings.py`):

```python
# MySQL connection
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'root'
MYSQL_PASSWORD = os.getenv('MYSQL_PASSWORD')
MYSQL_DB = 'hotsearch'

# Platform-specific cookies (optional, for some platforms)
COOKIES = {
    'weibo': 'your_weibo_cookie_string',  # Optional
    'bilibili': 'your_bilibili_cookie'     # Optional
}

# Concurrent requests
CONCURRENT_REQUESTS = 8
DOWNLOAD_DELAY = 1
```

**2. Analysis System** (`.env` file in project root):

```env
# Database
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=your_db_password
MYSQL_DB=hotsearch

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Or use Huawei Pangu (local deployment recommended)
# PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model

# Push Notification Settings
# WeChat Work Robot
WECHAT_WORK_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
EMAIL_RECIPIENTS=recipient1@example.com,recipient2@example.com

# WeChat Work Application (personal push)
WECHAT_CORP_ID=your_corp_id
WECHAT_APP_SECRET=your_app_secret
WECHAT_AGENT_ID=your_agent_id
```

## Running the System

### Start Crawler

```bash
# Test individual spider
python runspider-test.py

# Run all spiders (usually triggered from web UI)
python run_spiders.py
```

### Start Analysis Web Interface

```bash
# Launch Flask application
python app.py

# Access at http://localhost:5000
# Features:
# - Conversational query interface
# - Real-time trending list display
# - Start/stop crawler with hotkeys
# - Configure push notification tasks
```

### Testing Push Notifications

```bash
# Test push task configuration
python test_push_task.py
```

## Key Usage Patterns

### 1. Querying Trends via API

```python
from hotsearch_analysis_agent.query_engine import QueryEngine

engine = QueryEngine()

# Natural language query
results = engine.query("Show me trending topics about artificial intelligence in the last 24 hours")

# Results include:
# - Matched trending items
# - Sentiment analysis
# - Topic clusters
# - Related news content
```

### 2. Topic Clustering Analysis

```python
from hotsearch_analysis_agent.clustering import TopicCluster

cluster = TopicCluster()

# Cluster topics from multiple platforms
topics = [
    {"title": "GPT-6 leaked with 2M context", "platform": "bilibili"},
    {"title": "DeepSeek V4 uses Huawei chips", "platform": "weibo"},
    {"title": "AI model usage surpasses US", "platform": "toutiao"}
]

clusters = cluster.analyze(topics)
# Returns: grouped topics with common themes
```

### 3. Sentiment Analysis

```python
from hotsearch_analysis_agent.sentiment import SentimentAnalyzer

analyzer = SentimentAnalyzer()

content = """
DeepSeek V4确认采用华为昇腾算力底座,标志着国产芯片-框架-模型全栈生态
进入规模化验证阶段。这是国产AI芯片的重要里程碑。
"""

sentiment = analyzer.analyze(content)
# Returns: {"score": 0.85, "label": "positive", "confidence": 0.92}
```

### 4. Content Extraction (Including Video)

```python
from hotsearch_analysis_agent.content_extractor import ContentExtractor

extractor = ContentExtractor()

# Extract article content
url = "https://www.bilibili.com/video/BV13pSoBBEvX/"
content = extractor.extract(url)

# For video URLs, extracts:
# - Video title and description
# - Subtitle/transcript if available
# - Comment sentiment analysis
# - Related metadata
```

### 5. Creating Scheduled Push Tasks

```python
from hotsearch_analysis_agent.push_manager import PushManager

manager = PushManager()

# Create daily AI trends report
task = manager.create_task(
    name="AI Daily Report",
    query="人工智能 OR 大模型 OR AI",
    schedule="0 12 * * *",  # Daily at 12:00
    channels=["wechat_work", "email"],
    analysis_depth="detailed"  # Options: summary, detailed, comprehensive
)

# Task automatically:
# 1. Queries matching trends
# 2. Performs LLM analysis
# 3. Generates formatted report
# 4. Sends via configured channels
```

### 6. Custom Report Generation

```python
from hotsearch_analysis_agent.report_generator import ReportGenerator
from datetime import datetime, timedelta

generator = ReportGenerator()

# Generate report for specific topic
report = generator.generate(
    topic="人工智能与前沿科技",
    start_time=datetime.now() - timedelta(days=1),
    end_time=datetime.now(),
    platforms=["weibo", "bilibili", "zhihu"],
    include_sentiment=True,
    include_clustering=True
)

# Report contains:
# - Executive summary
# - Key findings with data points
# - Detailed news items with URLs
# - Sentiment trends
# - Information propagation analysis
# - Actionable insights
```

## Working with Scrapy Spiders

### Platform Spider Structure

```python
# Example: hotsearchcrawler/spiders/weibo_spider.py
import scrapy
from ..items import HotSearchItem

class WeiboHotSpider(scrapy.Spider):
    name = 'weibo_hot'
    allowed_domains = ['weibo.com']
    start_urls = ['https://s.weibo.com/top/summary']
    
    def parse(self, response):
        for item in response.css('tr.item'):
            yield HotSearchItem(
                platform='weibo',
                title=item.css('a::text').get(),
                url=item.css('a::attr(href)').get(),
                hot_value=item.css('span.hot::text').get(),
                rank_position=item.css('td.num::text').get()
            )
```

### Pipeline for Database Storage

```python
# hotsearchcrawler/pipelines.py
class MySQLPipeline:
    def process_item(self, item, spider):
        self.cursor.execute("""
            INSERT INTO hot_searches 
            (platform, title, url, hot_value, rank_position)
            VALUES (%s, %s, %s, %s, %s)
        """, (
            item['platform'],
            item['title'],
            item['url'],
            item['hot_value'],
            item['rank_position']
        ))
        self.conn.commit()
        return item
```

## LLM Integration

### Using Huawei Pangu Model (Recommended for Local Deployment)

```python
from hotsearch_analysis_agent.llm.pangu_client import PanguClient

# Initialize local Pangu model
client = PanguClient(
    model_path="/path/to/openpangu-embedded-7b-model"
)

# Sentiment analysis
prompt = f"""
分析以下新闻的情感倾向(positive/negative/neutral):

新闻内容: {news_content}

输出格式: {{"sentiment": "positive", "confidence": 0.95, "reasoning": "..."}}
"""

response = client.generate(prompt, max_tokens=500)
```

### Using OpenAI-Compatible API

```python
from hotsearch_analysis_agent.llm.openai_client import OpenAIClient

client = OpenAIClient(
    api_key=os.getenv('OPENAI_API_KEY'),
    base_url=os.getenv('OPENAI_API_BASE'),
    model=os.getenv('MODEL_NAME', 'gpt-4')
)

# Topic clustering
messages = [
    {"role": "system", "content": "你是一个舆情分析专家"},
    {"role": "user", "content": f"将以下话题分组: {topics_list}"}
]

result = client.chat_completion(messages)
```

## Common Workflows

### Daily Monitoring Workflow

```python
# 1. Start scheduled crawler (via cron or systemd)
# crontab -e
# 0 */4 * * * cd /path/to/project && /path/to/venv/bin/python run_spiders.py

# 2. Configure analysis task
from hotsearch_analysis_agent.scheduler import TaskScheduler

scheduler = TaskScheduler()
scheduler.add_daily_task(
    time="08:00",
    query="科技 OR 互联网",
    channels=["email"]
)

# 3. Query recent trends
from hotsearch_analysis_agent.api import get_recent_trends

trends = get_recent_trends(
    hours=24,
    platforms=["weibo", "zhihu", "bilibili"],
    min_hot_value=100000
)
```

### Event Monitoring Workflow

```python
# Monitor specific keywords with alerts
from hotsearch_analysis_agent.alert_manager import AlertManager

alert = AlertManager()

# Alert when keyword appears in top 10
alert.create_rule(
    keywords=["产品名称", "公司名"],
    rank_threshold=10,
    platforms=["weibo", "douyin"],
    notification_channels=["telegram", "wechat_work"],
    immediate=True  # Push immediately when detected
)
```

## Troubleshooting

### Crawler Issues

**Problem**: Spider fails with 403/blocked

```python
# Solution: Rotate user agents, add delays
# In hotsearchcrawler/settings.py
DOWNLOAD_DELAY = 3
RANDOMIZE_DOWNLOAD_DELAY = True

USER_AGENT_LIST = [
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64)...',
    'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)...'
]
```

**Problem**: ChromeDriver not found

```bash
# Verify driver is in PATH
which chromedriver  # macOS/Linux
where chromedriver  # Windows

# Or specify driver path explicitly in code
from selenium import webdriver
driver = webdriver.Chrome(executable_path='/path/to/chromedriver')
```

### Database Issues

**Problem**: Connection timeout

```python
# Increase connection pool settings
MYSQL_POOL_SIZE = 10
MYSQL_POOL_RECYCLE = 3600
MYSQL_CONNECT_TIMEOUT = 30
```

**Problem**: Character encoding errors

```sql
-- Ensure database uses utf8mb4
ALTER DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
ALTER TABLE hot_searches CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### LLM Analysis Issues

**Problem**: Analysis too slow with large models

```python
# Use lighter model for real-time queries
# Reserve heavy model for scheduled reports
REALTIME_MODEL = "gpt-3.5-turbo"
REPORT_MODEL = "gpt-4"
```

**Problem**: Context length exceeded

```python
# Truncate long content before analysis
def truncate_content(content, max_tokens=6000):
    # Simple token estimation: ~1.5 chars per token for Chinese
    max_chars = int(max_tokens * 1.5)
    return content[:max_chars] if len(content) > max_chars else content
```

### Push Notification Issues

**Problem**: WeChat Work webhook returns error

```python
# Verify webhook format
import requests

webhook_url = os.getenv('WECHAT_WORK_WEBHOOK')
payload = {
    "msgtype": "markdown",
    "markdown": {
        "content": "# Test Message\nIf you see this, webhook works!"
    }
}
response = requests.post(webhook_url, json=payload)
print(response.json())  # Check for error details
```

**Problem**: Email sending fails

```python
# Test SMTP connection
import smtplib
from email.mime.text import MIMEText

try:
    server = smtplib.SMTP(os.getenv('SMTP_HOST'), int(os.getenv('SMTP_PORT')))
    server.starttls()
    server.login(os.getenv('SMTP_USER'), os.getenv('SMTP_PASSWORD'))
    print("SMTP connection successful")
    server.quit()
except Exception as e:
    print(f"SMTP error: {e}")
```

## Performance Optimization

### Crawler Optimization

```python
# Enable concurrent requests
CONCURRENT_REQUESTS = 16
CONCURRENT_REQUESTS_PER_DOMAIN = 4

# Use asyncio for faster crawling
TWISTED_REACTOR = 'twisted.internet.asyncioreactor.AsyncioSelectorReactor'

# Cache DNS lookups
DNSCACHE_ENABLED = True
```

### Database Query Optimization

```python
# Use indexes for common queries
# Add to init.py or run manually
CREATE INDEX idx_platform_time ON hot_searches(platform, crawl_time);
CREATE INDEX idx_title_fulltext ON hot_searches(title) USING FULLTEXT;
CREATE INDEX idx_hot_value ON hot_searches(hot_value DESC);

# Query with pagination
SELECT * FROM hot_searches 
WHERE platform = 'weibo' 
ORDER BY crawl_time DESC 
LIMIT 100 OFFSET 0;
```

## Advanced Features

### Custom Platform Spider

```python
# Add new platform in hotsearchcrawler/spiders/
class NewPlatformSpider(scrapy.Spider):
    name = 'new_platform'
    start_urls = ['https://example.com/trending']
    
    def parse(self, response):
        for item in response.css('.trending-item'):
            yield HotSearchItem(
                platform='new_platform',
                title=item.css('h3::text').get(),
                url=response.urljoin(item.css('a::attr(href)').get()),
                hot_value=item.css('.popularity::text').get(),
                rank_position=item.css('.rank::text').get()
            )

# Register in settings.py
SPIDER_MODULES = ['hotsearchcrawler.spiders']
```

### Multi-Language Support

```python
# Add translation for international platforms
from hotsearch_analysis_agent.translator import Translator

translator = Translator()

# Translate before analysis
if detect_language(content) != 'zh':
    content_zh = translator.translate(content, target='zh')
    sentiment = analyzer.analyze(content_zh)
```

This skill provides comprehensive coverage of the LLM Public Opinion Analytics Assistant for monitoring Chinese social media trends with AI-powered analysis.
