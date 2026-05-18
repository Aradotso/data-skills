---
name: llm-public-opinion-analytics-assistant
description: Build intelligent public opinion monitoring systems with real-time multi-platform data crawling, LLM-powered sentiment analysis, topic clustering, and multi-channel alert pushes
triggers:
  - analyze social media sentiment trends
  - monitor hot topics across platforms
  - set up public opinion monitoring dashboard
  - create sentiment analysis pipeline with LLM
  - crawl trending topics from multiple sources
  - build opinion analytics with push notifications
  - aggregate news sentiment analysis
  - cluster related topics from social feeds
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is a comprehensive public opinion monitoring system that integrates **15 mainstream platforms** with **26 trending lists**, combining real-time web scraping with large language model analysis. It provides conversational hot search queries, topic clustering, sentiment analysis, and multi-channel push notifications (Email, WeChat, Telegram).

## What This Project Does

- **Multi-Platform Data Crawling**: Automated scrapers for 15 platforms (Weibo, Bilibili, Zhihu, Toutiao, etc.)
- **LLM-Powered Analysis**: Topic clustering, sentiment analysis, and trend detection using large language models
- **Conversational Interface**: Natural language queries for hot topics and trends
- **Content Extraction**: Deep analysis including video content metadata
- **Multi-Channel Alerts**: Push reports via Enterprise WeChat, Telegram, and email
- **Real-Time Monitoring**: Hotkey-controlled crawler management with live dashboard

## Installation

### Prerequisites

```bash
# Python 3.8+
# MySQL 5.7+
# Chrome or Edge browser
```

### Browser Driver Setup

1. **Check browser version** (Chrome/Edge → Settings → About)
2. **Download matching driver**:
   - Chrome: https://chromedriver.chromium.org/
   - Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/
3. **Add to PATH**:

```bash
# Linux/macOS
export PATH=$PATH:/path/to/driver/directory
source ~/.bashrc

# Windows: Add driver path to System PATH via Environment Variables
```

4. **Verify**:

```bash
chromedriver --version
```

### Install Dependencies

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install requirements
pip install -r requirements.txt
```

### Database Setup

```python
# Reference init.py for schema
import pymysql

connection = pymysql.connect(
    host='localhost',
    user='your_user',
    password='your_password',
    database='public_opinion_db',
    charset='utf8mb4'
)

# Create tables as defined in init.py
```

## Configuration

### Crawler Configuration (`hotsearchcrawler/settings.py`)

```python
# MySQL connection
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_user'
MYSQL_PASSWORD = 'your_password'
MYSQL_DATABASE = 'public_opinion_db'

# Spider settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
COOKIES_ENABLED = True

# Optional: Platform-specific cookies
COOKIES_DEBUG = {
    'weibo': 'your_weibo_cookie',
    'bilibili': 'your_bilibili_cookie'
}
```

### Analysis System Configuration (`.env`)

```bash
# Database
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=public_opinion_db

# LLM Configuration (OpenAI-compatible API)
LLM_API_BASE=http://your-llm-endpoint
LLM_API_KEY=your_api_key
LLM_MODEL=pangu-embedded-7b

# Push Notifications
# Enterprise WeChat
WECHAT_WEBHOOK_URL=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY
WECHAT_CORP_ID=your_corp_id
WECHAT_AGENT_ID=your_agent_id
WECHAT_SECRET=your_secret

# Telegram
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email (SMTP)
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_USER=your_email@example.com
SMTP_PASSWORD=your_password
EMAIL_TO=recipient@example.com
```

## Running the System

### Start the Web Application

```bash
python app.py
```

Access the dashboard at `http://localhost:5000`

### Manual Crawler Execution (Testing)

```bash
# Test single spider
cd hotsearchcrawler
python runspider-test.py

# Run specific platform
scrapy crawl weibo_hot
scrapy crawl bilibili_hot
scrapy crawl zhihu_hot
```

### Start All Crawlers (Production)

```bash
# Via UI: Use hotkey controls in dashboard
# Or manually:
python run_spiders.py
```

## Key API Patterns

### Query Hot Topics

```python
from hotsearch_analysis_agent.query_engine import QueryEngine

engine = QueryEngine()

# Natural language query
results = engine.query("最近人工智能领域的热点话题")
# Returns: Clustered topics with sentiment analysis

# Platform-specific query
results = engine.query("微博热搜前10", platform="weibo")

# Time-filtered query
results = engine.query(
    "昨天的科技新闻",
    time_range={"start": "2026-05-17", "end": "2026-05-18"}
)
```

### Sentiment Analysis

```python
from hotsearch_analysis_agent.sentiment_analyzer import SentimentAnalyzer

analyzer = SentimentAnalyzer()

# Analyze single article
sentiment = analyzer.analyze_text("""
    华为盘古大模型发布新版本,性能提升显著
""")
# Returns: {"sentiment": "positive", "score": 0.85, "keywords": [...]}

# Batch analysis
articles = [{"id": 1, "content": "..."}, {"id": 2, "content": "..."}]
results = analyzer.batch_analyze(articles)
```

### Topic Clustering

```python
from hotsearch_analysis_agent.topic_cluster import TopicCluster

cluster = TopicCluster()

# Cluster recent hot topics
topics = cluster.cluster_topics(
    query="人工智能",
    num_clusters=4,
    min_similarity=0.7
)
# Returns: [{
#   "cluster_id": 1,
#   "topic": "大模型技术突破",
#   "articles": [...],
#   "sentiment": "positive"
# }, ...]
```

### Generate Analysis Report

```python
from hotsearch_analysis_agent.report_generator import ReportGenerator

generator = ReportGenerator()

# Generate comprehensive report
report = generator.generate(
    query="人工智能与前沿科技",
    include_clusters=True,
    include_sentiment=True,
    include_sources=True
)

# Save as HTML/Markdown
generator.save(report, format="markdown", path="./report.md")
```

## Push Notification Setup

### Test Push Configurations

```python
# test_push_task.py
from hotsearch_analysis_agent.push_service import PushService

pusher = PushService()

# Test Enterprise WeChat
pusher.send_wechat_robot(
    webhook_url=os.getenv("WECHAT_WEBHOOK_URL"),
    content="测试推送",
    mentioned_list=["@all"]
)

# Test Telegram
pusher.send_telegram(
    bot_token=os.getenv("TELEGRAM_BOT_TOKEN"),
    chat_id=os.getenv("TELEGRAM_CHAT_ID"),
    message="测试推送",
    parse_mode="Markdown"
)

# Test Email
pusher.send_email(
    subject="舆情分析报告",
    body="<html>...</html>",
    to=os.getenv("EMAIL_TO"),
    html=True
)
```

### Schedule Push Tasks

```python
from hotsearch_analysis_agent.scheduler import TaskScheduler

scheduler = TaskScheduler()

# Daily report at 9 AM
scheduler.add_task(
    name="daily_ai_report",
    query="人工智能领域热点",
    schedule="0 9 * * *",  # Cron format
    channels=["wechat", "telegram", "email"],
    format="markdown"
)

# Real-time alerts for specific keywords
scheduler.add_alert(
    keywords=["安全漏洞", "数据泄露"],
    threshold_score=0.8,
    channels=["wechat", "telegram"]
)
```

## Data Extraction Patterns

### Extract Article Content (Including Video)

```python
from hotsearchcrawler.content_extractor import ContentExtractor

extractor = ContentExtractor()

# Extract from news URL
content = extractor.extract(url="https://example.com/news/12345")
# Returns: {
#   "title": "...",
#   "content": "...",
#   "publish_time": "2026-05-18",
#   "author": "...",
#   "media_type": "video",  # or "article"
#   "video_metadata": {...}  # if video
# }

# Extract video metadata (Bilibili, YouTube, etc.)
video_info = extractor.extract_video_info(url="https://bilibili.com/video/...")
# Returns: transcript, duration, views, comments
```

### Custom Spider Development

```python
# hotsearchcrawler/spiders/custom_spider.py
import scrapy
from ..items import HotSearchItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://custom-platform.com/trending']
    
    def parse(self, response):
        for item in response.css('.trending-item'):
            yield HotSearchItem(
                title=item.css('.title::text').get(),
                url=item.css('a::attr(href)').get(),
                rank=item.css('.rank::text').get(),
                heat_score=item.css('.score::text').get(),
                platform='custom_platform',
                category='tech'
            )
```

## LLM Integration (Pangu Model)

### Local Deployment

```bash
# Download Pangu model
# https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

# Configure model path in .env
LLM_MODEL_PATH=/path/to/pangu-model
LLM_DEVICE=cuda  # or cpu
```

### Use LLM for Analysis

```python
from hotsearch_analysis_agent.llm_client import LLMClient

llm = LLMClient()

# Sentiment analysis prompt
prompt = f"""
分析以下新闻的情感倾向和关键信息:

标题: {article['title']}
内容: {article['content']}

请返回JSON格式:
{{
    "sentiment": "positive/negative/neutral",
    "score": 0-1,
    "keywords": [...],
    "summary": "..."
}}
"""

response = llm.generate(
    prompt=prompt,
    max_tokens=500,
    temperature=0.3
)

# Topic clustering prompt
cluster_prompt = f"""
将以下{len(articles)}篇文章按主题聚类,返回3-5个主题簇:

{format_articles(articles)}

要求:
1. 每个簇包含标题和相关文章ID
2. 为每个簇生成简短描述
3. 返回JSON数组格式
"""
```

## Common Workflows

### Workflow 1: Real-Time Monitoring Dashboard

```python
# app.py route example
from flask import Flask, jsonify, request
from hotsearch_analysis_agent.query_engine import QueryEngine

app = Flask(__name__)
engine = QueryEngine()

@app.route('/api/hot-topics')
def get_hot_topics():
    platform = request.args.get('platform', 'all')
    limit = request.args.get('limit', 50)
    
    topics = engine.get_trending(
        platform=platform,
        limit=limit,
        time_window='24h'
    )
    
    return jsonify(topics)

@app.route('/api/analyze')
def analyze_query():
    query = request.args.get('q')
    
    analysis = engine.analyze(
        query=query,
        include_clustering=True,
        include_sentiment=True
    )
    
    return jsonify(analysis)
```

### Workflow 2: Automated Daily Reports

```python
# scheduled_report.py
from hotsearch_analysis_agent.report_generator import ReportGenerator
from hotsearch_analysis_agent.push_service import PushService
from datetime import datetime

def generate_daily_report():
    generator = ReportGenerator()
    pusher = PushService()
    
    # Generate report
    report = generator.generate(
        query="人工智能与前沿科技",
        time_range="yesterday",
        format="markdown"
    )
    
    # Push to all channels
    pusher.push_report(
        report=report,
        channels=["wechat", "telegram", "email"],
        title=f"舆情分析报告 - {datetime.now().strftime('%Y-%m-%d')}"
    )

# Schedule with cron: 0 9 * * *
if __name__ == '__main__':
    generate_daily_report()
```

### Workflow 3: Keyword Alert System

```python
from hotsearch_analysis_agent.alert_monitor import AlertMonitor

monitor = AlertMonitor()

# Define alert rules
monitor.add_rule(
    name="security_alert",
    keywords=["数据泄露", "安全漏洞", "黑客攻击"],
    platforms=["weibo", "zhihu", "toutiao"],
    threshold_heat=1000,  # Heat score threshold
    callback=lambda article: send_urgent_alert(article)
)

# Start monitoring
monitor.start(interval=300)  # Check every 5 minutes
```

## Troubleshooting

### Issue: Crawler Returns Empty Results

```python
# Check cookies validity
from hotsearchcrawler.cookie_manager import CookieManager

manager = CookieManager()
valid = manager.validate_cookies('weibo')

if not valid:
    # Refresh cookies manually or via browser automation
    manager.refresh_cookies('weibo')
```

### Issue: LLM Analysis Timeout

```python
# Increase timeout in .env
LLM_REQUEST_TIMEOUT=120

# Or reduce input length
def truncate_content(text, max_length=2000):
    return text[:max_length] + "..." if len(text) > max_length else text
```

### Issue: Database Connection Pool Exhausted

```python
# Adjust pool size in settings
MYSQL_POOL_SIZE = 20
MYSQL_MAX_OVERFLOW = 10
MYSQL_POOL_RECYCLE = 3600

# Or use connection retry
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

engine = create_engine(
    f"mysql+pymysql://{user}:{password}@{host}/{database}",
    poolclass=QueuePool,
    pool_size=20,
    max_overflow=10,
    pool_pre_ping=True  # Test connections
)
```

### Issue: Push Notifications Not Sending

```python
# Test each channel individually
from hotsearch_analysis_agent.push_service import PushService

pusher = PushService()

# Debug mode
pusher.set_debug(True)

try:
    pusher.send_wechat_robot(
        webhook_url=os.getenv("WECHAT_WEBHOOK_URL"),
        content="测试"
    )
except Exception as e:
    print(f"WeChat push failed: {e}")
    # Check webhook URL validity
    # Verify network connectivity
```

### Issue: Chrome Driver Version Mismatch

```bash
# Check Chrome version
google-chrome --version

# Download exact matching driver version
# Or use webdriver-manager for automatic management
pip install webdriver-manager
```

```python
from selenium import webdriver
from webdriver_manager.chrome import ChromeDriverManager

driver = webdriver.Chrome(ChromeDriverManager().install())
```

## Advanced Usage

### Custom LLM Backend

```python
# hotsearch_analysis_agent/llm_client.py
class CustomLLMClient(LLMClient):
    def __init__(self, api_base, api_key):
        self.api_base = api_base
        self.api_key = api_key
    
    def generate(self, prompt, **kwargs):
        response = requests.post(
            f"{self.api_base}/v1/completions",
            headers={"Authorization": f"Bearer {self.api_key}"},
            json={
                "prompt": prompt,
                "max_tokens": kwargs.get("max_tokens", 500),
                "temperature": kwargs.get("temperature", 0.7)
            }
        )
        return response.json()["choices"][0]["text"]
```

### Multi-Language Support

```python
# Add translation layer
from hotsearch_analysis_agent.translator import Translator

translator = Translator()

# Translate queries
en_query = translator.translate("最新科技新闻", target="en")

# Translate results
results_cn = translator.translate_batch(
    results,
    source="en",
    target="zh"
)
```

This skill provides comprehensive guidance for deploying and using the LLM-based public opinion analytics system, covering installation, configuration, API usage, and common troubleshooting scenarios.
