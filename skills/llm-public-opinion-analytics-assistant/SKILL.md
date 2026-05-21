---
name: llm-public-opinion-analytics-assistant
description: Multi-platform real-time hot topic crawler and LLM-powered sentiment analysis system with automated alerting
triggers:
  - how do I set up the public opinion analytics assistant
  - help me analyze hot topics across social platforms
  - how to configure the hot search crawler system
  - show me how to use the sentiment analysis agent
  - set up multi-channel push notifications for trending topics
  - how do I deploy the hotsearch analysis system
  - configure browser drivers for news content extraction
  - analyze and cluster trending topics with LLM
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive system that crawls 26 real-time trending lists from 15 major platforms, performs LLM-based sentiment and clustering analysis, and pushes insights via email, WeChat, Telegram, or corporate messaging systems.

## What It Does

This project combines distributed web crawling with large language model analysis to:

- **Crawl trending topics** from platforms like Weibo, Bilibili, Toutiao, and 12+ others
- **Extract content** from news detail pages (including video metadata)
- **Analyze sentiment** and cluster related topics using LLM (Huawei Pangu or OpenAI-compatible models)
- **Generate reports** with insights on trending themes
- **Push notifications** via Enterprise WeChat, Telegram, email (SMTP)
- **Provide conversational interface** for querying and analyzing topics

## Installation

### Prerequisites

**Browser Driver Setup** (required for content extraction):

1. Install Chrome or Edge browser
2. Download matching driver:
   - Chrome: https://chromedriver.chromium.org/
   - Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/
3. Place driver in system PATH or browser directory
4. Verify: `chromedriver --version`

**Database Setup**:

```bash
# Install MySQL and create database
mysql -u root -p
CREATE DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

**Python Environment**:

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Initialize Database

```python
# Reference init.py for table schemas
from hotsearch_analysis_agent.database import init_database

init_database()
```

## Configuration

### 1. Crawler Configuration

Edit `hotsearchcrawler/settings.py`:

```python
# MySQL connection
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_user'
MYSQL_PASSWORD = 'your_password'
MYSQL_DATABASE = 'hotsearch'

# Optional: Platform cookies for authenticated access
COOKIES = {
    'weibo': 'your_cookie_string',
    'bilibili': 'your_cookie_string'
}
```

### 2. Analysis System Configuration

Create `.env` file in `hotsearch_analysis_agent/`:

```bash
# Database
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Or use local Huawei Pangu model
# MODEL_TYPE=pangu
# PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b

# Push Notifications
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password

TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

WECHAT_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxx
```

## Core Usage Patterns

### Running the Crawler

```python
# Test individual spider
from hotsearchcrawler.spiders.weibo_spider import WeiboSpider
from scrapy.crawler import CrawlerProcess

process = CrawlerProcess()
process.crawl(WeiboSpider)
process.start()
```

```bash
# Run all spiders (from project root)
python run_spiders.py

# Or via web interface
python app.py
# Access http://localhost:5000 and use keyboard shortcuts to start/stop
```

### Querying Trending Topics

```python
from hotsearch_analysis_agent.query_engine import QueryEngine

engine = QueryEngine()

# Query specific platform
results = engine.query_platform('weibo', limit=10)
for item in results:
    print(f"{item['rank']}. {item['title']} - {item['hotness']}")

# Search by keyword
results = engine.search_topics('人工智能')
print(f"Found {len(results)} related topics")

# Time-based query
from datetime import datetime, timedelta
yesterday = datetime.now() - timedelta(days=1)
results = engine.query_by_time(start_time=yesterday)
```

### LLM-Based Analysis

```python
from hotsearch_analysis_agent.analyzer import TopicAnalyzer

analyzer = TopicAnalyzer(model_type='openai')  # or 'pangu'

# Sentiment analysis
sentiment = analyzer.analyze_sentiment(
    "GPT-6遭提前曝光, 2M超长上下文来了"
)
print(f"Sentiment: {sentiment['label']}, Score: {sentiment['score']}")

# Topic clustering
topics = [
    "DeepSeek V4采用华为算力",
    "中国主流大模型周调用量连续五周超越美国",
    "Anthropic年化收入暴涨至300亿美元"
]
clusters = analyzer.cluster_topics(topics)
for cluster_id, items in clusters.items():
    print(f"Cluster {cluster_id}: {items}")

# Generate analysis report
report = analyzer.generate_report(
    query="人工智能与前沿科技",
    topics=topics,
    time_range='2026-04-01 to 2026-04-07'
)
print(report)
```

### Setting Up Push Tasks

```python
from hotsearch_analysis_agent.push_service import PushService

push = PushService()

# Email notification
push.send_email(
    subject="热点分析报告 - AI领域",
    content=report,
    recipients=['user@example.com']
)

# Telegram push
push.send_telegram(
    message=f"🔥 新热点发现\n\n{report[:500]}..."
)

# Enterprise WeChat (Robot webhook)
push.send_wechat_robot(
    content=report,
    mentioned_list=['@all']
)

# Schedule periodic push
from hotsearch_analysis_agent.scheduler import TaskScheduler

scheduler = TaskScheduler()
scheduler.add_task(
    name='daily_ai_report',
    query='人工智能',
    schedule='0 9 * * *',  # 9 AM daily
    push_channels=['email', 'telegram']
)
```

### Conversational Query Interface

```python
from hotsearch_analysis_agent.chat_agent import ChatAgent

agent = ChatAgent()

# Natural language queries
response = agent.query("今天微博热搜前10是什么?")
print(response)

response = agent.query("分析一下最近关于AI的热点话题")
print(response)

response = agent.query("有哪些话题涉及华为?")
print(response)
```

## Web Interface Usage

```bash
# Start Flask app
python app.py
```

**Key Features**:
- Dashboard showing all platform trends
- Keyboard shortcuts for crawler control
- Click-through to original news pages
- Interactive chat for LLM analysis
- Task configuration for scheduled reports

## Database Schema Reference

```python
# Hot topics table (populated by crawlers)
CREATE TABLE hot_topics (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    platform VARCHAR(50),
    title VARCHAR(500),
    url VARCHAR(1000),
    hotness BIGINT,
    rank INT,
    crawl_time DATETIME,
    content TEXT,  # Extracted detail content
    INDEX idx_platform_time (platform, crawl_time)
);

# Analysis results
CREATE TABLE analysis_results (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    topic_id BIGINT,
    sentiment VARCHAR(20),
    sentiment_score FLOAT,
    cluster_id INT,
    keywords JSON,
    created_at DATETIME
);

# Push tasks
CREATE TABLE push_tasks (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100),
    query VARCHAR(500),
    schedule VARCHAR(100),
    channels JSON,
    enabled BOOLEAN,
    last_run DATETIME
);
```

## Common Patterns

### Extract Content from News Pages

```python
from hotsearch_analysis_agent.content_extractor import ContentExtractor

extractor = ContentExtractor(driver_type='chrome')

# Extract article content
content = extractor.extract(url='https://www.example.com/news/123')
print(content['title'])
print(content['text'])
print(content['publish_time'])

# Extract video metadata
video_content = extractor.extract(url='https://www.bilibili.com/video/BV13pSoBBEvX')
print(video_content['description'])
print(video_content['tags'])
```

### Batch Process Topics

```python
from hotsearch_analysis_agent.batch_processor import BatchProcessor

processor = BatchProcessor()

# Fetch unprocessed topics
topics = processor.get_pending_topics(platform='weibo', hours=24)

# Analyze in batch
for topic in topics:
    # Extract content
    if not topic.get('content'):
        topic['content'] = extractor.extract(topic['url'])
    
    # Analyze sentiment
    sentiment = analyzer.analyze_sentiment(topic['title'])
    
    # Save results
    processor.save_analysis(topic['id'], sentiment)
    
print(f"Processed {len(topics)} topics")
```

### Custom LLM Prompt Templates

```python
from hotsearch_analysis_agent.prompts import PromptTemplate

# Define custom analysis prompt
custom_prompt = PromptTemplate(
    template="""
    分析以下热点话题的舆情倾向和潜在影响:
    
    话题: {title}
    内容: {content}
    平台: {platform}
    热度: {hotness}
    
    请从以下维度分析:
    1. 情感倾向(正面/负面/中性)
    2. 主要关注点
    3. 潜在舆论风险
    4. 建议应对策略
    """
)

result = analyzer.analyze_with_template(
    template=custom_prompt,
    title=topic['title'],
    content=topic['content'],
    platform=topic['platform'],
    hotness=topic['hotness']
)
```

## Troubleshooting

### Browser Driver Issues

```bash
# Error: 'chromedriver' executable needs to be in PATH
# Solution: Verify driver location
which chromedriver  # Linux/Mac
where chromedriver  # Windows

# Add to PATH (Linux/Mac)
export PATH=$PATH:/path/to/driver/directory

# Windows: Add driver directory to System Environment Variables
```

### MySQL Connection Errors

```python
# Error: Access denied for user
# Check credentials in settings.py or .env

# Error: Database doesn't exist
# Create database first:
# CREATE DATABASE hotsearch CHARACTER SET utf8mb4;
```

### LLM API Timeout

```python
# Increase timeout in analyzer config
from hotsearch_analysis_agent.analyzer import TopicAnalyzer

analyzer = TopicAnalyzer(
    timeout=120,  # seconds
    max_retries=3
)
```

### Crawler Gets Blocked

```python
# Add delays and headers in spider settings
DOWNLOAD_DELAY = 2
RANDOMIZE_DOWNLOAD_DELAY = True

DEFAULT_REQUEST_HEADERS = {
    'User-Agent': 'Mozilla/5.0 ...',
    'Accept': 'text/html,application/json',
}

# Use cookies for authenticated platforms
COOKIES_ENABLED = True
```

### Memory Issues with Large Batches

```python
# Process topics in smaller chunks
def process_in_batches(topics, batch_size=100):
    for i in range(0, len(topics), batch_size):
        batch = topics[i:i+batch_size]
        process_batch(batch)
        # Clear memory
        import gc
        gc.collect()
```

## Advanced Configuration

### Using Huawei Pangu Model Locally

```python
# Download model from:
# https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

# Configure in .env
MODEL_TYPE=pangu
PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model
PANGU_DEVICE=cuda  # or cpu

# Use in code
from hotsearch_analysis_agent.analyzer import TopicAnalyzer

analyzer = TopicAnalyzer(model_type='pangu')
result = analyzer.analyze_sentiment("DeepSeek V4采用华为算力")
```

### Multi-Platform Concurrent Crawling

```python
from scrapy.crawler import CrawlerProcess
from scrapy.utils.project import get_project_settings

settings = get_project_settings()
settings.set('CONCURRENT_REQUESTS', 16)
settings.set('CONCURRENT_REQUESTS_PER_DOMAIN', 4)

process = CrawlerProcess(settings)

# Add all spiders
from hotsearchcrawler.spiders import *
process.crawl(WeiboSpider)
process.crawl(BilibiliSpider)
process.crawl(ToutiaoSpider)
# ... add more

process.start()
```
