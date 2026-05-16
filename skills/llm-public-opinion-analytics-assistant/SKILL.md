---
name: llm-public-opinion-analytics-assistant
description: Chinese multi-platform hot topic crawler with LLM-powered sentiment analysis, clustering, and multi-channel push notifications
triggers:
  - analyze public opinion from Chinese social platforms
  - crawl hot topics from Weibo Bilibili and other platforms
  - set up sentiment analysis for trending news
  - configure multi-platform hot search monitoring
  - create automated opinion report with LLM
  - implement topic clustering and sentiment detection
  - setup hot topic push notifications to WeChat
  - build a Chinese public opinion monitoring system
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to use the LLM-Based Intelligent Public Opinion Analytics Assistant, a comprehensive Chinese public opinion monitoring system that crawls hot topics from 15 mainstream platforms (26 ranking lists), performs LLM-powered analysis including sentiment detection and topic clustering, and sends automated reports via email, WeChat, Enterprise WeChat, or Telegram.

## What This Project Does

The system consists of two main components:

1. **Crawler Cluster** (`hotsearchcrawler`): Scraped data from 15 Chinese platforms including Weibo, Bilibili, Douyin, Zhihu, Baidu, etc.
2. **Analysis System** (`hotsearch_analysis_agent`): LLM-powered conversational interface for querying hot topics, clustering analysis, sentiment analysis, and automated report generation

Key capabilities:
- Real-time multi-platform hot topic monitoring
- Natural language queries for trending topics
- Automatic topic clustering and sentiment analysis
- Multi-channel push notifications (Email, WeChat, Enterprise WeChat, Telegram)
- Video content analysis (extracts information even from video-based news)
- Keyboard shortcuts for crawler control

## Installation

### Prerequisites

**1. Browser Driver Setup**

The project requires a browser driver (Chrome/Edge) for scraping news detail pages:

```bash
# Check your Chrome/Edge version first
# Chrome: Settings → About
# Edge: Settings → About

# Download matching ChromeDriver from:
# https://chromedriver.chromium.org/

# Or EdgeDriver from:
# https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Place driver in system PATH or project root
# Verify installation:
chromedriver --version
```

**2. Database Setup**

```bash
# Install MySQL
# Create database and tables using init.py as reference

# Example database initialization
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
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Configuration

**1. Environment Variables (`.env` file)**

```bash
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=${DB_PASSWORD}
MYSQL_DATABASE=hotsearch

# LLM API Configuration (OpenAI-compatible format)
OPENAI_API_KEY=${YOUR_LLM_API_KEY}
OPENAI_API_BASE=https://api.openai.com/v1  # Or your LLM endpoint

# Recommended: Huawei Pangu model endpoint
# OPENAI_API_BASE=https://your-pangu-endpoint.com/v1

# Push Notification Channels (optional)
# Email
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=${YOUR_EMAIL}
SMTP_PASSWORD=${YOUR_EMAIL_PASSWORD}
EMAIL_FROM=${YOUR_EMAIL}
EMAIL_TO=recipient@example.com

# Enterprise WeChat Bot
WECHAT_WEBHOOK=${WECHAT_BOT_WEBHOOK_URL}

# Enterprise WeChat App
WECHAT_CORP_ID=${CORP_ID}
WECHAT_AGENT_ID=${AGENT_ID}
WECHAT_SECRET=${CORP_SECRET}

# Telegram Bot
TELEGRAM_BOT_TOKEN=${BOT_TOKEN}
TELEGRAM_CHAT_ID=${CHAT_ID}
```

**2. Crawler Settings (`hotsearchcrawler/settings.py`)**

```python
# MySQL connection for crawler
MYSQL_CONFIG = {
    'host': 'localhost',
    'port': 3306,
    'user': 'root',
    'password': os.getenv('DB_PASSWORD'),
    'database': 'hotsearch',
    'charset': 'utf8mb4'
}

# Optional: Platform-specific cookies
COOKIES = {
    'weibo': 'your_weibo_cookie_string',
    'bilibili': 'your_bilibili_cookie_string'
}

# Browser driver path (if not in system PATH)
CHROMEDRIVER_PATH = '/path/to/chromedriver'
```

## Running the System

### Start the Analysis Web Interface

```bash
# Main application entry point
python app.py
```

This launches the web interface at `http://localhost:5000` with:
- Conversational query interface
- Real-time hot topic monitoring dashboard
- Crawler control via keyboard shortcuts
- Push task configuration panel

### Run Crawlers

**Via Web Interface:**
- Use keyboard shortcuts to start/stop crawlers
- Monitor crawling status in real-time

**Via Command Line:**

```bash
# Run all crawlers
python run_spiders.py

# Test specific crawler
python runspider-test.py --spider=weibo

# Available spiders: weibo, bilibili, douyin, zhihu, baidu, etc.
```

### Test Push Notifications

```bash
# Test all configured push channels
python test_push_task.py

# This will send test reports via:
# - Email (if configured)
# - WeChat bot (if configured)
# - Enterprise WeChat app (if configured)
# - Telegram (if configured)
```

## Core API and Usage Patterns

### Conversational Query Examples

The system supports natural language queries in Chinese:

```python
# Example queries (sent via web interface):

# 1. Get hot topics from specific platform
"给我看看微博热搜前10条"
# "Show me top 10 Weibo hot topics"

# 2. Search for specific topic
"搜索关于人工智能的新闻"
# "Search for news about artificial intelligence"

# 3. Topic clustering
"分析最近关于科技的话题聚类"
# "Analyze recent technology topic clusters"

# 4. Sentiment analysis
"分析这个话题的情感倾向"
# "Analyze sentiment for this topic"
```

### Programmatic API Usage

**Query Hot Topics:**

```python
from hotsearch_analysis_agent.query import HotSearchQuery

# Initialize query engine
query = HotSearchQuery(db_config={
    'host': 'localhost',
    'user': 'root',
    'password': os.getenv('DB_PASSWORD'),
    'database': 'hotsearch'
})

# Get top 10 from specific platform
results = query.get_platform_hot_topics(
    platform='weibo',
    limit=10
)

# Search by keyword
results = query.search_topics(
    keyword='人工智能',
    platforms=['weibo', 'zhihu', 'bilibili'],
    days=7  # Last 7 days
)

# Get topic details including news content
detail = query.get_topic_detail(topic_id=12345)
```

**Topic Clustering:**

```python
from hotsearch_analysis_agent.clustering import TopicCluster

cluster = TopicCluster(llm_api_key=os.getenv('OPENAI_API_KEY'))

# Cluster topics by keyword
clusters = cluster.analyze(
    keyword='人工智能',
    days=7,
    min_cluster_size=3
)

# Result structure:
# {
#     'clusters': [
#         {
#             'name': '大模型技术迭代',
#             'topics': [...],
#             'summary': '...'
#         },
#         ...
#     ]
# }
```

**Sentiment Analysis:**

```python
from hotsearch_analysis_agent.sentiment import SentimentAnalyzer

analyzer = SentimentAnalyzer(llm_api_key=os.getenv('OPENAI_API_KEY'))

# Analyze single topic
sentiment = analyzer.analyze_topic(topic_id=12345)
# Returns: {'sentiment': 'positive', 'score': 0.85, 'reasoning': '...'}

# Batch sentiment analysis
sentiments = analyzer.analyze_batch(topic_ids=[123, 456, 789])
```

**Generate Analysis Report:**

```python
from hotsearch_analysis_agent.report import ReportGenerator

generator = ReportGenerator(
    llm_api_key=os.getenv('OPENAI_API_KEY'),
    db_config={...}
)

# Generate comprehensive report
report = generator.generate(
    keyword='人工智能',
    days=7,
    include_clustering=True,
    include_sentiment=True,
    include_source_links=True
)

# report contains markdown formatted analysis
# with structured sections, data highlights, and source URLs
```

**Setup Push Tasks:**

```python
from hotsearch_analysis_agent.push import PushTaskManager

manager = PushTaskManager(db_config={...})

# Create scheduled push task
task = manager.create_task(
    name='AI Tech Daily Digest',
    keyword='人工智能',
    schedule='0 12 * * *',  # Daily at 12:00 PM (cron format)
    channels=['email', 'wechat'],
    config={
        'email': {
            'recipients': ['user@example.com'],
            'subject': 'AI热点日报'
        },
        'wechat': {
            'webhook': os.getenv('WECHAT_WEBHOOK')
        }
    }
)

# List active tasks
tasks = manager.list_tasks()

# Execute task manually
manager.execute_task(task_id=task['id'])

# Delete task
manager.delete_task(task_id=task['id'])
```

### Crawler Customization

**Add New Platform Spider:**

```python
# In hotsearchcrawler/spiders/new_platform.py
import scrapy
from ..items import HotSearchItem

class NewPlatformSpider(scrapy.Spider):
    name = 'newplatform'
    allowed_domains = ['newplatform.com']
    start_urls = ['https://newplatform.com/hot']
    
    def parse(self, response):
        for item in response.css('.hot-item'):
            yield HotSearchItem(
                platform='newplatform',
                title=item.css('.title::text').get(),
                url=item.css('.link::attr(href)').get(),
                rank=item.css('.rank::text').get(),
                heat=item.css('.heat::text').get(),
                category=item.css('.category::text').get(),
                timestamp=datetime.now()
            )
    
    def parse_detail(self, response):
        # Extract news detail page content
        content = response.css('.article-content::text').getall()
        return {
            'content': '\n'.join(content),
            'author': response.css('.author::text').get(),
            'publish_time': response.css('.time::text').get()
        }
```

## Common Workflows

### Daily Hot Topic Monitoring

```python
# 1. Start crawlers (via web UI or command line)
python run_spiders.py

# 2. Query aggregated hot topics
from hotsearch_analysis_agent.query import HotSearchQuery

query = HotSearchQuery(db_config={...})
daily_hot = query.get_aggregated_hot_topics(
    platforms=['weibo', 'zhihu', 'bilibili'],
    limit=50,
    hours=24
)

# 3. Generate analysis report
from hotsearch_analysis_agent.report import ReportGenerator

generator = ReportGenerator(llm_api_key=os.getenv('OPENAI_API_KEY'))
report = generator.generate_daily_digest(topics=daily_hot)

# 4. Push to configured channels
from hotsearch_analysis_agent.push import PushService

pusher = PushService()
pusher.send_report(
    report=report,
    channels=['email', 'wechat', 'telegram']
)
```

### Topic Tracking and Alert

```python
# Track specific keywords and get alerts
from hotsearch_analysis_agent.alert import TopicAlert

alert = TopicAlert(
    keywords=['重大事件', '突发新闻'],
    platforms=['weibo', 'toutiao'],
    threshold=10000,  # Heat threshold
    db_config={...}
)

# Check for new high-heat topics
new_alerts = alert.check_and_notify(
    channels=['telegram'],  # Immediate notification
    interval=300  # Check every 5 minutes
)
```

### Sentiment Trend Analysis

```python
from hotsearch_analysis_agent.sentiment import SentimentTrendAnalyzer

analyzer = SentimentTrendAnalyzer(llm_api_key=os.getenv('OPENAI_API_KEY'))

# Analyze sentiment trend over time
trend = analyzer.analyze_trend(
    keyword='某品牌',
    days=30,
    granularity='daily'
)

# Result: time series of sentiment scores
# [{
#     'date': '2026-05-01',
#     'sentiment': 'positive',
#     'score': 0.75,
#     'topic_count': 123
# }, ...]
```

## Troubleshooting

### Browser Driver Issues

```bash
# Error: chromedriver not found
# Solution: Add driver to PATH or specify path in settings.py

export PATH=$PATH:/path/to/chromedriver

# Or in hotsearchcrawler/settings.py:
CHROMEDRIVER_PATH = '/absolute/path/to/chromedriver'

# Error: version mismatch
# Solution: Download matching driver version
# Check Chrome version: chrome://version/
# Download from: https://chromedriver.chromium.org/
```

### Database Connection Failures

```python
# Error: Can't connect to MySQL server
# Check connection parameters:

import pymysql

try:
    conn = pymysql.connect(
        host='localhost',
        user='root',
        password=os.getenv('DB_PASSWORD'),
        database='hotsearch'
    )
    print("Connection successful")
except Exception as e:
    print(f"Connection failed: {e}")

# Common fixes:
# 1. Verify MySQL is running: sudo systemctl status mysql
# 2. Check firewall rules
# 3. Verify user permissions: GRANT ALL ON hotsearch.* TO 'user'@'localhost';
```

### Crawler Rate Limiting

```python
# Error: 429 Too Many Requests
# Solution: Add delays in hotsearchcrawler/settings.py

DOWNLOAD_DELAY = 3  # Seconds between requests
CONCURRENT_REQUESTS_PER_DOMAIN = 1
AUTOTHROTTLE_ENABLED = True
AUTOTHROTTLE_START_DELAY = 2
AUTOTHROTTLE_MAX_DELAY = 10

# For specific platforms requiring cookies:
# Update COOKIES dict in settings.py
```

### LLM API Timeout

```python
# Error: Request timeout during analysis
# Solution: Increase timeout in analysis client

from openai import OpenAI

client = OpenAI(
    api_key=os.getenv('OPENAI_API_KEY'),
    base_url=os.getenv('OPENAI_API_BASE'),
    timeout=60.0  # Increase from default 30s
)

# For Pangu model deployment:
# Ensure model is properly loaded and endpoint is responsive
# Check model health: curl http://your-endpoint/health
```

### Push Notification Failures

```python
# Email not sending
# Check SMTP configuration:

import smtplib
from email.mime.text import MIMEText

try:
    msg = MIMEText('Test')
    msg['Subject'] = 'Test'
    msg['From'] = os.getenv('SMTP_USER')
    msg['To'] = 'recipient@example.com'
    
    with smtplib.SMTP(os.getenv('SMTP_HOST'), int(os.getenv('SMTP_PORT'))) as server:
        server.starttls()
        server.login(os.getenv('SMTP_USER'), os.getenv('SMTP_PASSWORD'))
        server.send_message(msg)
    print("Email sent successfully")
except Exception as e:
    print(f"Email failed: {e}")

# WeChat webhook not working
# Verify webhook URL is active:
import requests

response = requests.post(
    os.getenv('WECHAT_WEBHOOK'),
    json={'msgtype': 'text', 'text': {'content': 'Test'}}
)
print(response.status_code, response.text)
```

## Advanced Configuration

### Using Huawei Pangu Model

The project recommends Huawei's openPangu-Embedded-7B model for superior Chinese language understanding:

```bash
# Download model from:
# https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

# Deploy locally (example with vLLM):
python -m vllm.entrypoints.openai.api_server \
    --model /path/to/openpangu-embedded-7b \
    --host 0.0.0.0 \
    --port 8000

# Update .env:
OPENAI_API_BASE=http://localhost:8000/v1
OPENAI_API_KEY=EMPTY  # Not required for local deployment
```

### Performance Optimization

```python
# For large-scale monitoring (hotsearchcrawler/settings.py):

# Increase crawler concurrency
CONCURRENT_REQUESTS = 32
CONCURRENT_REQUESTS_PER_DOMAIN = 8

# Enable caching
HTTPCACHE_ENABLED = True
HTTPCACHE_EXPIRATION_SECS = 3600

# Database connection pooling (in analysis system)
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

engine = create_engine(
    f"mysql+pymysql://{os.getenv('MYSQL_USER')}:{os.getenv('MYSQL_PASSWORD')}@{os.getenv('MYSQL_HOST')}/{os.getenv('MYSQL_DATABASE')}",
    poolclass=QueuePool,
    pool_size=10,
    max_overflow=20
)
```

This skill covers the complete workflow for deploying and using the LLM-Based Intelligent Public Opinion Analytics Assistant for comprehensive Chinese social media monitoring and analysis.
