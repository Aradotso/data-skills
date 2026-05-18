---
name: llm-public-opinion-analytics
description: Chinese public opinion analytics assistant with multi-platform hot search crawling, LLM-powered analysis, and multi-channel push notifications
triggers:
  - crawl hot search data from Chinese platforms
  - analyze public sentiment and trending topics
  - set up opinion monitoring and alerts
  - configure weibo bilibili douyin crawlers
  - analyze news sentiment with pangu model
  - push trending reports to wechat telegram
  - cluster and analyze social media topics
  - monitor public opinion across platforms
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive Chinese public opinion monitoring system that crawls 26 hot search lists from 15 major platforms (Weibo, Bilibili, Douyin, etc.), performs LLM-powered sentiment and clustering analysis, and pushes reports via email, WeChat, Enterprise WeChat, and Telegram.

## What This Project Does

- **Multi-Platform Crawling**: Automated scrapers for Weibo, Bilibili, Douyin, Toutiao, Zhihu, Baidu, and 10+ other Chinese platforms
- **LLM Analysis**: Sentiment analysis, topic clustering, and trend detection using Huawei Pangu or OpenAI-compatible models
- **Content Extraction**: Extracts full content from news detail pages (including video-based news)
- **Multi-Channel Push**: Schedule and send reports via Enterprise WeChat, WeChat, Telegram, or email
- **Web Interface**: Chat-based query interface with keyboard shortcuts for crawler control

## Installation

### Prerequisites

**1. Browser Driver Setup** (required for content scraping):

```bash
# Download ChromeDriver or EdgeDriver matching your browser version
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Place driver in system PATH or project directory
# Verify installation:
chromedriver --version
```

**2. MySQL Database**:

```bash
# Create database and tables using init.py reference
mysql -u root -p
CREATE DATABASE public_opinion CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

**3. Python Environment**:

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

**1. Crawler Settings** (`hotsearchcrawler/settings.py`):

```python
# MySQL connection
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'root'
MYSQL_PASSWORD = 'your_password'
MYSQL_DB = 'public_opinion'

# Optional: Platform cookies for authenticated scraping
COOKIES = {
    'weibo': 'your_weibo_cookie',
    'bilibili': 'your_bilibili_cookie'
}
```

**2. Analysis System** (`.env` file):

```bash
# Database
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=your_password
MYSQL_DB=public_opinion

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_BASE_URL=https://api.openai.com/v1  # Or Pangu model endpoint

# Push Notifications
# Enterprise WeChat Robot
WECOM_WEBHOOK_URL=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=your_key

# Enterprise WeChat Application
WECOM_CORPID=your_corp_id
WECOM_CORPSECRET=your_corp_secret
WECOM_AGENTID=your_agent_id

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
SMTP_TO=recipient@example.com
```

## Running the System

### Start Web Interface

```bash
python app.py
```

Access the web UI at `http://localhost:5000`

### Manual Crawler Execution

**Test individual spider**:

```bash
# From runspider-test file
cd hotsearchcrawler
scrapy crawl weibo_spider
scrapy crawl bilibili_spider
scrapy crawl douyin_spider
```

**Run all spiders**:

```bash
# Execute run_spiders (normally triggered via web interface)
python run_spiders.py
```

### Test Push Notifications

```bash
# From test_push_task file
python test_push_task.py
```

## Key Usage Patterns

### 1. Querying Hot Search Data

**Via Web Interface** (natural language):

```python
# User input examples:
"显示微博热搜前10条"
"查找关于人工智能的话题"
"分析最近科技类新闻的情感倾向"
"对今日热点进行聚类分析"
```

**Direct Database Query**:

```python
import pymysql

connection = pymysql.connect(
    host='localhost',
    user='root',
    password='your_password',
    database='public_opinion',
    charset='utf8mb4'
)

with connection.cursor() as cursor:
    # Get latest hot searches from Weibo
    sql = """
        SELECT title, hot_value, url, crawl_time 
        FROM hot_search 
        WHERE platform = 'weibo' 
        ORDER BY crawl_time DESC 
        LIMIT 10
    """
    cursor.execute(sql)
    results = cursor.fetchall()
    for row in results:
        print(f"{row[0]} - 热度: {row[1]}")
```

### 2. LLM-Powered Analysis

**Sentiment Analysis**:

```python
from hotsearch_analysis_agent.analyzer import SentimentAnalyzer

analyzer = SentimentAnalyzer(
    api_key=os.getenv('OPENAI_API_KEY'),
    base_url=os.getenv('OPENAI_BASE_URL')
)

# Analyze news content
content = """
DeepSeek V4采用华为算力，国产芯片生态走到哪一步了？
国产大模型DeepSeek最新迭代版本确认基于华为昇腾910B集群训练...
"""

result = analyzer.analyze_sentiment(content)
print(f"情感倾向: {result['sentiment']}")  # positive/negative/neutral
print(f"置信度: {result['confidence']}")
print(f"关键词: {result['keywords']}")
```

**Topic Clustering**:

```python
from hotsearch_analysis_agent.analyzer import TopicClusterer

clusterer = TopicClusterer(
    api_key=os.getenv('OPENAI_API_KEY'),
    base_url=os.getenv('OPENAI_BASE_URL')
)

# Cluster related topics
topics = [
    "GPT-6遭提前曝光, 2M超长上下文来了",
    "DeepSeek V4采用华为算力",
    "Anthropic年化收入暴涨至300亿美元",
    "中国主流大模型周调用量连续五周超越美国"
]

clusters = clusterer.cluster_topics(topics)
for cluster_id, cluster_topics in clusters.items():
    print(f"话题簇 {cluster_id}: {cluster_topics}")
```

### 3. Setting Up Push Tasks

**Create Scheduled Report**:

```python
from hotsearch_analysis_agent.push_manager import PushManager

push_manager = PushManager()

# Configure push task
task_config = {
    'name': '人工智能领域舆情日报',
    'keywords': ['人工智能', '大模型', 'AI', '芯片'],
    'platforms': ['weibo', 'toutiao', 'zhihu'],
    'schedule': 'daily',  # daily/hourly/weekly
    'time': '12:00',
    'channels': ['wecom_robot', 'telegram', 'email'],
    'analysis_types': ['sentiment', 'clustering', 'trend']
}

push_manager.create_task(task_config)
```

**Enterprise WeChat Push Example**:

```python
import requests
import json

def push_to_wecom(content, webhook_url):
    """Push report to Enterprise WeChat group"""
    payload = {
        "msgtype": "markdown",
        "markdown": {
            "content": content
        }
    }
    
    response = requests.post(
        webhook_url,
        data=json.dumps(payload),
        headers={'Content-Type': 'application/json'}
    )
    
    return response.json()

# Generate report markdown
report = """
## 关于人工智能与前沿科技的热点分析
**Time**: 2026-05-18 12:00:00

### 🔍 核心发现
- GPT-6提前曝光，200万超长上下文
- DeepSeek V4采用华为昇腾算力
- Anthropic收入首超OpenAI

[查看详情](https://your-domain.com/report/123)
"""

push_to_wecom(report, os.getenv('WECOM_WEBHOOK_URL'))
```

### 4. Custom Spider Development

**Add new platform spider**:

```python
# hotsearchcrawler/spiders/custom_spider.py
import scrapy
from hotsearchcrawler.items import HotSearchItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://example.com/trending']
    
    def parse(self, response):
        for item_div in response.css('.trending-item'):
            item = HotSearchItem()
            item['platform'] = 'custom_platform'
            item['title'] = item_div.css('.title::text').get()
            item['hot_value'] = item_div.css('.hot-value::text').get()
            item['url'] = item_div.css('a::attr(href)').get()
            item['rank'] = item_div.css('.rank::text').get()
            
            # Fetch detail page content
            if item['url']:
                yield scrapy.Request(
                    item['url'],
                    callback=self.parse_detail,
                    meta={'item': item}
                )
    
    def parse_detail(self, response):
        item = response.meta['item']
        item['content'] = response.css('.article-content::text').getall()
        item['author'] = response.css('.author::text').get()
        yield item
```

## Common Workflows

### Daily Monitoring Setup

```python
# 1. Configure crawlers to run every 30 minutes
from apscheduler.schedulers.background import BackgroundScheduler
from hotsearchcrawler.run_spiders import run_all_spiders

scheduler = BackgroundScheduler()
scheduler.add_job(run_all_spiders, 'interval', minutes=30)
scheduler.start()

# 2. Set up automatic analysis on new data
def analyze_new_data():
    # Fetch unanalyzed records
    new_items = fetch_unanalyzed_items()
    
    for item in new_items:
        # Sentiment analysis
        sentiment = analyzer.analyze_sentiment(item['content'])
        
        # Update database
        update_analysis_result(item['id'], sentiment)
        
        # Trigger alert if negative sentiment on monitored keywords
        if sentiment['sentiment'] == 'negative' and contains_keywords(item['title']):
            send_alert(item)

scheduler.add_job(analyze_new_data, 'interval', minutes=10)
```

### Generate Custom Report

```python
from datetime import datetime, timedelta
from hotsearch_analysis_agent.report_generator import ReportGenerator

generator = ReportGenerator(
    api_key=os.getenv('OPENAI_API_KEY'),
    base_url=os.getenv('OPENAI_BASE_URL')
)

# Generate report for last 24 hours
start_time = datetime.now() - timedelta(days=1)
end_time = datetime.now()

report = generator.generate_report(
    keywords=['人工智能', '大模型'],
    platforms=['weibo', 'zhihu', 'toutiao'],
    start_time=start_time,
    end_time=end_time,
    include_sentiment=True,
    include_clustering=True
)

# Save as markdown
with open(f'report_{datetime.now().strftime("%Y%m%d")}.md', 'w') as f:
    f.write(report)
```

## Troubleshooting

### Crawler Issues

**Problem**: ChromeDriver version mismatch
```bash
# Solution: Update driver to match browser version
# Check browser version: chrome://version or edge://version
# Download matching driver from official sources
```

**Problem**: Cookies expired (HTTP 401/403)
```python
# Solution: Update cookies in settings.py
# 1. Login to platform in browser
# 2. Open DevTools (F12) > Application > Cookies
# 3. Copy cookie string and update COOKIES dict
```

**Problem**: Anti-scraping detection
```python
# hotsearchcrawler/settings.py
DOWNLOAD_DELAY = 3  # Increase delay between requests
RANDOMIZE_DOWNLOAD_DELAY = True
CONCURRENT_REQUESTS_PER_DOMAIN = 1  # Reduce concurrency

# Use proxy rotation
DOWNLOADER_MIDDLEWARES = {
    'scrapy.downloadermiddlewares.httpproxy.HttpProxyMiddleware': 110,
    'hotsearchcrawler.middlewares.ProxyMiddleware': 100,
}
```

### Database Issues

**Problem**: Connection timeout
```python
# Solution: Check MySQL service status and firewall
# Increase connection pool timeout
MYSQL_CONNECT_TIMEOUT = 60
```

**Problem**: Character encoding errors
```sql
-- Ensure database and tables use utf8mb4
ALTER DATABASE public_opinion CHARACTER SET = utf8mb4 COLLATE = utf8mb4_unicode_ci;
ALTER TABLE hot_search CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### LLM Analysis Issues

**Problem**: Pangu model memory overflow
```python
# Solution: Reduce batch size or use smaller model variant
# For local deployment, ensure sufficient GPU memory (recommend 16GB+)
```

**Problem**: API rate limiting
```python
# Implement exponential backoff
import time
from tenacity import retry, wait_exponential, stop_after_attempt

@retry(wait=wait_exponential(multiplier=1, min=4, max=60), 
       stop=stop_after_attempt(5))
def call_llm_api(prompt):
    # Your API call here
    pass
```

### Push Notification Issues

**Problem**: Enterprise WeChat webhook returns error
```python
# Verify webhook URL is correct and not expired
# Check message format matches API spec
# Test with minimal payload:
test_payload = {
    "msgtype": "text",
    "text": {"content": "Test message"}
}
```

**Problem**: Telegram bot not receiving messages
```bash
# Verify bot token and chat ID
# Get chat ID by sending /start to bot and checking:
curl https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/getUpdates
```

## Advanced Configuration

### Using Pangu Model Locally

```python
# Download model from https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model
# Configure model path in .env
PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model

# Load model
from hotsearch_analysis_agent.models import PanguModel

model = PanguModel(model_path=os.getenv('PANGU_MODEL_PATH'))

# Use for analysis
result = model.analyze(
    text="新闻内容...",
    task="sentiment"  # sentiment/clustering/summarization
)
```

### Custom Analysis Pipeline

```python
from hotsearch_analysis_agent.pipeline import AnalysisPipeline

pipeline = AnalysisPipeline()

# Add custom analysis steps
pipeline.add_step('preprocess', lambda x: x.strip())
pipeline.add_step('sentiment', SentimentAnalyzer())
pipeline.add_step('keywords', KeywordExtractor())
pipeline.add_step('clustering', TopicClusterer())

# Execute pipeline
results = pipeline.run(news_items)
```
