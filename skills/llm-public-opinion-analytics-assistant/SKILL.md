---
name: llm-public-opinion-analytics-assistant
description: Multi-platform public opinion analytics system with 26 real-time ranking lists, LLM-powered clustering, sentiment analysis, and multi-channel notifications
triggers:
  - how do I set up the public opinion analytics assistant
  - analyze trending topics across multiple platforms
  - configure sentiment analysis with LLM models
  - scrape hot search rankings from social platforms
  - set up multi-channel notifications for trending topics
  - cluster and analyze social media trends
  - deploy the opinion analytics dashboard
  - integrate Pangu model for Chinese text analysis
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is an intelligent public opinion analytics assistant that integrates **15 mainstream platforms** with **26 ranking lists**. It combines real-time web scraping with LLM-powered analysis for conversational topic queries, clustering, sentiment analysis, and multi-channel notifications (email, WeChat, enterprise WeChat, Telegram). The system supports video content extraction and can analyze detailed news pages.

**Key Capabilities:**
- Real-time hot search data from 15 platforms (Weibo, Bilibili, Douyin, Baidu, etc.)
- LLM-powered topic clustering and sentiment analysis
- Natural language query interface
- Multi-channel push notifications with scheduled reports
- Keyboard shortcuts for crawler control
- Video content analysis support

## Installation

### Prerequisites

1. **Browser Driver Setup** (for web scraping):

```bash
# Check your Chrome/Edge version first
google-chrome --version  # or: microsoft-edge --version

# Download matching driver:
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Place driver in PATH (Linux/macOS example):
sudo mv chromedriver /usr/local/bin/
chmod +x /usr/local/bin/chromedriver

# Verify installation:
chromedriver --version
```

2. **Database Setup**:

```bash
# Install MySQL 5.7+
# Create database and tables using init.py as reference
mysql -u root -p
```

```sql
CREATE DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE hotsearch;

-- Main hot search table
CREATE TABLE hot_search (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    platform VARCHAR(50) NOT NULL,
    title VARCHAR(500) NOT NULL,
    url VARCHAR(1000),
    rank_position INT,
    hot_score VARCHAR(50),
    category VARCHAR(100),
    crawl_time DATETIME,
    detail_content TEXT,
    sentiment VARCHAR(20),
    INDEX idx_platform_time (platform, crawl_time),
    INDEX idx_title (title(255))
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

3. **Python Environment**:

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Core Dependencies

```python
# requirements.txt key packages:
# scrapy>=2.11.0
# flask>=2.3.0
# selenium>=4.15.0
# openai>=1.0.0
# mysql-connector-python>=8.2.0
# transformers>=4.35.0  # for local LLM
# torch>=2.0.0  # for Pangu model
```

## Configuration

### 1. Environment Variables (.env)

```bash
# Database Configuration
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=${MYSQL_PASSWORD}
DB_NAME=hotsearch

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=${YOUR_OPENAI_API_KEY}
OPENAI_API_BASE=https://api.openai.com/v1
OPENAI_MODEL=gpt-4

# Or use local Pangu model
USE_LOCAL_MODEL=true
LOCAL_MODEL_PATH=/path/to/openpangu-embedded-7b-model

# Notification Channels
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=${EMAIL_USER}
SMTP_PASSWORD=${EMAIL_PASSWORD}

WECHAT_WEBHOOK=${WECHAT_ROBOT_WEBHOOK}
TELEGRAM_BOT_TOKEN=${TELEGRAM_BOT_TOKEN}
TELEGRAM_CHAT_ID=${TELEGRAM_CHAT_ID}
```

### 2. Crawler Configuration (hotsearchcrawler/settings.py)

```python
# MySQL Pipeline Settings
MYSQL_CONFIG = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME'),
    'charset': 'utf8mb4'
}

# Concurrent Settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1

# User Agent Rotation
USER_AGENT_LIST = [
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36...',
    # Add more user agents
]

# Optional: Platform Cookies (for restricted content)
COOKIES_WEIBO = {
    'SUB': 'your_cookie_value_here'  # Get from browser
}
```

### 3. Analysis System Configuration (hotsearch_analysis_agent/config.py)

```python
# LLM Configuration
LLM_CONFIG = {
    'provider': 'openai',  # or 'pangu' for local model
    'model': 'gpt-4',
    'temperature': 0.7,
    'max_tokens': 2000
}

# Analysis Parameters
CLUSTERING_CONFIG = {
    'min_similarity': 0.65,  # Topic similarity threshold
    'min_cluster_size': 3,
    'max_clusters': 10
}

SENTIMENT_CONFIG = {
    'labels': ['positive', 'neutral', 'negative'],
    'threshold': 0.6
}
```

## Usage

### Starting the System

```bash
# 1. Initialize database (first time only)
python init.py

# 2. Start the web application
python app.py
# Access dashboard at http://localhost:5000

# 3. Start crawlers (can be triggered via web UI or CLI)
python run_spiders.py --platforms weibo,douyin,bilibili
```

### Using the Web Interface

```python
# The Flask app provides these endpoints:

# GET /api/rankings?platform=weibo&limit=50
# Returns latest hot search rankings

# POST /api/query
# Natural language query interface
{
    "query": "分析最近关于AI的热点新闻",
    "platforms": ["weibo", "douyin", "zhihu"],
    "days": 7
}

# POST /api/cluster
# Topic clustering analysis
{
    "keywords": ["人工智能", "AI"],
    "min_items": 5,
    "time_range": "7d"
}

# POST /api/sentiment
# Sentiment analysis
{
    "topic": "华为盘古大模型",
    "platforms": ["all"]
}
```

### Core API Usage

#### 1. Database Operations

```python
from hotsearch_analysis_agent.db_manager import DatabaseManager

# Initialize database connection
db = DatabaseManager()

# Query hot search data
results = db.query_hot_search(
    platform='weibo',
    start_date='2026-05-01',
    end_date='2026-05-19',
    keyword='人工智能'
)

# Insert new hot search item
db.insert_hot_search({
    'platform': 'weibo',
    'title': 'GPT-6遭提前曝光',
    'url': 'https://weibo.com/...',
    'rank_position': 1,
    'hot_score': '4580000',
    'category': 'tech',
    'crawl_time': datetime.now()
})

# Batch operations
db.batch_insert_hot_search(items_list)
```

#### 2. LLM Analysis

```python
from hotsearch_analysis_agent.llm_analyzer import LLMAnalyzer

# Initialize with Pangu model (local)
analyzer = LLMAnalyzer(
    model_type='pangu',
    model_path='/path/to/openpangu-embedded-7b-model'
)

# Or use OpenAI-compatible API
analyzer = LLMAnalyzer(
    model_type='openai',
    api_key=os.getenv('OPENAI_API_KEY'),
    model='gpt-4'
)

# Topic clustering
news_items = [
    {'title': 'GPT-6遭提前曝光', 'content': '...'},
    {'title': 'DeepSeek V4采用华为算力', 'content': '...'},
    # more items
]

clusters = analyzer.cluster_topics(
    news_items,
    min_similarity=0.65
)
# Returns: [
#   {'topic': 'AI模型技术突破', 'items': [...], 'summary': '...'},
#   {'topic': '国产算力生态', 'items': [...], 'summary': '...'}
# ]

# Sentiment analysis
sentiment = analyzer.analyze_sentiment(
    text="DeepSeek V4采用华为算力，国产芯片生态走到哪一步了？",
    context=news_items
)
# Returns: {'label': 'positive', 'score': 0.82, 'reasoning': '...'}

# Generate analysis report
report = analyzer.generate_report(
    query="分析人工智能与前沿科技的热点",
    clusters=clusters,
    time_range='7d'
)
```

#### 3. Web Scraping

```python
from hotsearchcrawler.spiders import WeiboSpider, DouyinSpider

# Run single spider (Scrapy framework)
from scrapy.crawler import CrawlerProcess
from scrapy.utils.project import get_project_settings

process = CrawlerProcess(get_project_settings())

# Scrape Weibo hot search
process.crawl(WeiboSpider)

# Scrape with custom settings
process.crawl(
    DouyinSpider,
    max_pages=5,
    categories=['tech', 'entertainment']
)

process.start()

# Or use the scheduler for multiple platforms
from hotsearchcrawler.scheduler import CrawlerScheduler

scheduler = CrawlerScheduler()
scheduler.run_crawlers(['weibo', 'douyin', 'bilibili'])
scheduler.schedule_periodic(interval_minutes=30)
```

#### 4. Notification System

```python
from hotsearch_analysis_agent.notifier import NotificationManager

notifier = NotificationManager()

# Send email report
notifier.send_email(
    to='user@example.com',
    subject='每日舆情分析报告',
    body=report_html,
    attachment_path='report.pdf'
)

# WeChat Work webhook
notifier.send_wechat(
    webhook_url=os.getenv('WECHAT_WEBHOOK'),
    content={
        'msgtype': 'markdown',
        'markdown': {
            'content': report_markdown
        }
    }
)

# Telegram notification
notifier.send_telegram(
    bot_token=os.getenv('TELEGRAM_BOT_TOKEN'),
    chat_id=os.getenv('TELEGRAM_CHAT_ID'),
    message=report_text,
    parse_mode='Markdown'
)

# Schedule periodic push task
from hotsearch_analysis_agent.push_task import PushTaskManager

task_manager = PushTaskManager()
task_manager.create_task(
    name='daily_ai_report',
    query='人工智能 AI 大模型',
    schedule='0 9 * * *',  # Daily at 9 AM
    channels=['email', 'wechat', 'telegram'],
    recipients={
        'email': ['team@example.com'],
        'wechat': [os.getenv('WECHAT_WEBHOOK')],
        'telegram': [os.getenv('TELEGRAM_CHAT_ID')]
    }
)
```

## Common Patterns

### Pattern 1: Custom Platform Spider

```python
# hotsearchcrawler/spiders/custom_platform.py
import scrapy
from hotsearchcrawler.items import HotSearchItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://example.com/trending']
    
    def parse(self, response):
        for idx, item in enumerate(response.css('.trending-item')):
            hot_item = HotSearchItem()
            hot_item['platform'] = self.name
            hot_item['title'] = item.css('.title::text').get()
            hot_item['url'] = item.css('a::attr(href)').get()
            hot_item['rank_position'] = idx + 1
            hot_item['hot_score'] = item.css('.score::text').get()
            hot_item['crawl_time'] = datetime.now()
            
            # Fetch detail page
            yield scrapy.Request(
                hot_item['url'],
                callback=self.parse_detail,
                meta={'item': hot_item}
            )
    
    def parse_detail(self, response):
        item = response.meta['item']
        item['detail_content'] = response.css('.article-content').get()
        yield item
```

### Pattern 2: Custom Analysis Query

```python
from hotsearch_analysis_agent import AnalysisAgent

agent = AnalysisAgent()

# Complex multi-step analysis
results = agent.analyze(
    query="对比分析国内外大模型的发展趋势",
    steps=[
        {
            'type': 'data_retrieval',
            'params': {
                'keywords': ['GPT', 'Claude', 'DeepSeek', '盘古'],
                'time_range': '30d',
                'platforms': ['all']
            }
        },
        {
            'type': 'clustering',
            'params': {'method': 'semantic', 'min_cluster_size': 5}
        },
        {
            'type': 'sentiment',
            'params': {'granularity': 'aspect'}
        },
        {
            'type': 'comparison',
            'params': {
                'dimensions': ['技术能力', '商业化', '生态建设'],
                'groups': ['国内模型', '国外模型']
            }
        }
    ]
)

# Generate visual report
agent.export_report(
    results,
    format='html',
    include_charts=True,
    output_path='analysis_report.html'
)
```

### Pattern 3: Real-time Monitoring

```python
from hotsearch_analysis_agent.monitor import TrendMonitor

monitor = TrendMonitor()

# Define monitoring rules
monitor.add_rule(
    name='breaking_tech_news',
    keywords=['人工智能', 'AI', '芯片', '大模型'],
    threshold_score=1000000,  # Hot score threshold
    platforms=['weibo', 'douyin', 'zhihu'],
    callback=lambda item: print(f"Breaking: {item['title']}"),
    notify_channels=['telegram']
)

# Start monitoring (runs in background)
monitor.start(interval_seconds=60)

# Or check manually
trending_items = monitor.check_trends(
    time_window='1h',
    min_growth_rate=2.0  # 200% growth
)
```

## Troubleshooting

### Issue: Browser Driver Not Found

```bash
# Error: selenium.common.exceptions.WebDriverException: 'chromedriver' not in PATH

# Solution 1: Add to PATH
export PATH=$PATH:/path/to/driver/directory

# Solution 2: Specify driver path in code
from selenium import webdriver
from selenium.webdriver.chrome.service import Service

service = Service(executable_path='/usr/local/bin/chromedriver')
driver = webdriver.Chrome(service=service)
```

### Issue: Database Connection Failed

```python
# Error: mysql.connector.errors.DatabaseError: Access denied

# Check .env configuration
import os
print(f"DB_HOST: {os.getenv('DB_HOST')}")
print(f"DB_USER: {os.getenv('DB_USER')}")

# Test connection manually
import mysql.connector
conn = mysql.connector.connect(
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)
print("Connection successful!" if conn.is_connected() else "Failed")
```

### Issue: LLM Analysis Timeout

```python
# For large text analysis, use chunking

def analyze_large_content(content, analyzer):
    chunk_size = 4000  # tokens
    chunks = [content[i:i+chunk_size] for i in range(0, len(content), chunk_size)]
    
    results = []
    for chunk in chunks:
        result = analyzer.analyze_sentiment(
            text=chunk,
            timeout=30  # seconds
        )
        results.append(result)
    
    # Aggregate results
    avg_score = sum(r['score'] for r in results) / len(results)
    return {'label': results[0]['label'], 'score': avg_score}
```

### Issue: Crawler Being Blocked

```python
# Increase download delay and use proxy rotation

# In hotsearchcrawler/settings.py
DOWNLOAD_DELAY = 3
RANDOMIZE_DOWNLOAD_DELAY = True

DOWNLOADER_MIDDLEWARES = {
    'scrapy.downloadermiddlewares.useragent.UserAgentMiddleware': None,
    'scrapy_user_agents.middlewares.RandomUserAgentMiddleware': 400,
    'scrapy.downloadermiddlewares.httpproxy.HttpProxyMiddleware': 110,
    'hotsearchcrawler.middlewares.ProxyMiddleware': 100,
}

# Add proxy pool
PROXY_POOL = [
    'http://proxy1.example.com:8080',
    'http://proxy2.example.com:8080',
]
```

### Issue: Pangu Model Loading Error

```python
# Error: Out of memory when loading local model

# Solution: Use quantization or smaller batch size
from transformers import AutoModelForCausalLM, AutoTokenizer
import torch

model = AutoModelForCausalLM.from_pretrained(
    "/path/to/openpangu-embedded-7b-model",
    torch_dtype=torch.float16,  # Use FP16
    device_map="auto",  # Automatic device placement
    load_in_8bit=True  # 8-bit quantization
)

# Reduce batch size in analysis
analyzer = LLMAnalyzer(
    model_type='pangu',
    model_path='/path/to/model',
    batch_size=1  # Process one item at a time
)
```

## Testing

```bash
# Test crawler
python runspider-test.py --spider weibo

# Test push notifications
python test_push_task.py --channel email --recipient test@example.com

# Test full analysis pipeline
python -m pytest tests/test_analysis_pipeline.py -v
```

## Advanced: Custom LLM Integration

```python
# hotsearch_analysis_agent/llm_custom.py
from hotsearch_analysis_agent.llm_analyzer import BaseLLMAnalyzer

class CustomLLMAnalyzer(BaseLLMAnalyzer):
    def __init__(self, api_endpoint, api_key):
        self.endpoint = api_endpoint
        self.api_key = api_key
    
    def analyze_sentiment(self, text, **kwargs):
        import requests
        response = requests.post(
            f"{self.endpoint}/sentiment",
            headers={"Authorization": f"Bearer {self.api_key}"},
            json={"text": text}
        )
        return response.json()
    
    def cluster_topics(self, items, **kwargs):
        # Implement custom clustering logic
        embeddings = self._get_embeddings([item['title'] for item in items])
        clusters = self._kmeans_clustering(embeddings)
        return clusters

# Use custom analyzer
analyzer = CustomLLMAnalyzer(
    api_endpoint=os.getenv('CUSTOM_LLM_ENDPOINT'),
    api_key=os.getenv('CUSTOM_LLM_KEY')
)
```
