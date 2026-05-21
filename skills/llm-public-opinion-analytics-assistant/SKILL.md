---
name: llm-public-opinion-analytics-assistant
description: Deploy and use an intelligent public opinion analytics system that aggregates 26 hot lists from 15 platforms with LLM-powered sentiment analysis, topic clustering, and multi-channel push notifications
triggers:
  - how do I set up the public opinion analytics assistant
  - analyze trending topics from multiple social platforms
  - configure hot topic push notifications to WeChat or Telegram
  - scrape and analyze trending news with sentiment detection
  - set up LLM-based opinion monitoring system
  - create automated reports from social media hot lists
  - deploy Chinese social media analytics with AI
  - monitor trending topics across Weibo Bilibili Douyin
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The LLM-Based Intelligent Public Opinion Analytics Assistant is a comprehensive Chinese social media monitoring system that:

- **Aggregates data** from 15 major Chinese platforms (Weibo, Bilibili, Douyin, etc.) covering 26 different hot lists
- **Analyzes content** using LLM capabilities for sentiment analysis, topic clustering, and trend identification
- **Extracts insights** from news detail pages including video content
- **Pushes reports** via multiple channels (Email, WeChat, Enterprise WeChat, Telegram)
- **Provides conversational interface** for querying hot topics and generating analysis reports

The system consists of two independent components:
1. **Crawler cluster** (`hotsearchcrawler`) - Data collection from platforms
2. **Analysis system** (`hotsearch_analysis_agent`) - LLM-powered analytics and reporting

## Installation

### Prerequisites

**1. Browser Driver Setup (Required for news detail scraping)**

Download the appropriate driver for your browser:

- **Chrome**: https://chromedriver.chromium.org/
- **Edge**: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

Match driver version to your browser version:

```bash
# Check browser version
google-chrome --version  # Linux
# or check in browser: Settings → About

# Verify driver installation
chromedriver --version
```

Place driver in system PATH or project directory.

**2. Python Environment**

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # Linux/Mac
# or venv\Scripts\activate on Windows

# Install dependencies
pip install -r requirements.txt
```

**3. MySQL Database**

Install MySQL and create database:

```python
# Reference from init.py
import pymysql

connection = pymysql.connect(
    host='localhost',
    user='root',
    password='your_password',
    charset='utf8mb4'
)

cursor = connection.cursor()
cursor.execute("CREATE DATABASE IF NOT EXISTS hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci")
cursor.execute("USE hotsearch_db")

# Create tables for hot search data
cursor.execute("""
CREATE TABLE IF NOT EXISTS hot_searches (
    id INT AUTO_INCREMENT PRIMARY KEY,
    platform VARCHAR(50) NOT NULL,
    title VARCHAR(500) NOT NULL,
    url TEXT,
    heat_value VARCHAR(100),
    rank_position INT,
    detail_content TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_platform (platform),
    INDEX idx_created_at (created_at)
)
""")

connection.commit()
```

### Configuration

**1. Crawler Settings** (`hotsearchcrawler/settings.py`)

```python
# MySQL Configuration
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'root'
MYSQL_PASSWORD = os.getenv('MYSQL_PASSWORD')
MYSQL_DB = 'hotsearch_db'

# Optional: Platform cookies for authenticated access
COOKIES = {
    'weibo': 'your_weibo_cookie',  # Optional
    'bilibili': 'your_bilibili_cookie'  # Optional
}
```

**2. Analysis System** (`.env` file)

```bash
# Database
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=your_password
MYSQL_DB=hotsearch_db

# LLM Configuration - Pangu Model (recommended) or OpenAI-compatible API
LLM_API_BASE=http://localhost:8000/v1  # Local Pangu deployment
LLM_API_KEY=your_api_key
LLM_MODEL=pangu-embedded-7b

# Push Notifications
# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
NOTIFICATION_EMAIL=recipient@example.com

# Enterprise WeChat Bot
WECHAT_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

## Running the System

### Start the Analysis System

```bash
python app.py
```

Access web interface at `http://localhost:5000`

### Control Crawlers

**From Web Interface:**
Use keyboard shortcuts or UI buttons to start/stop crawlers.

**Manual Crawler Execution:**

```bash
# Test single spider
python runspider-test.py

# Run all spiders (typically triggered from UI)
python run_spiders.py
```

## Core Usage Patterns

### 1. Conversational Hot Topic Query

```python
# Example: Query hot topics via chat interface
# User input: "Show me top 10 trending topics on Weibo today"

from hotsearch_analysis_agent.query_engine import QueryEngine

engine = QueryEngine()
response = engine.process_query(
    query="Show me top 10 trending topics on Weibo today",
    platforms=["weibo"],
    limit=10
)

# Response includes:
# - List of topics with heat values
# - Direct links to original posts
# - Quick sentiment summary
```

### 2. Topic Clustering Analysis

```python
# Analyze related topics across platforms
from hotsearch_analysis_agent.clustering import TopicCluster

cluster = TopicCluster()
results = cluster.analyze(
    keywords=["人工智能", "AI"],  # "Artificial Intelligence", "AI"
    platforms=["weibo", "zhihu", "bilibili"],
    time_range="today"
)

# Returns:
# - Clustered topics by similarity
# - Sentiment distribution
# - Platform-specific trends
```

### 3. Sentiment Analysis

```python
# Analyze sentiment for specific topic
from hotsearch_analysis_agent.sentiment import SentimentAnalyzer

analyzer = SentimentAnalyzer(llm_client=llm_client)
sentiment = analyzer.analyze_topic(
    topic="DeepSeek V4采用华为算力",
    detail_content=news_content
)

# Output:
# {
#   "overall_sentiment": "positive",
#   "confidence": 0.87,
#   "key_opinions": [...],
#   "emotional_keywords": [...]
# }
```

### 4. Automated Report Generation

```python
# Generate comprehensive analysis report
from hotsearch_analysis_agent.report_generator import ReportGenerator

generator = ReportGenerator()
report = generator.create_report(
    query="人工智能与前沿科技",  # AI and cutting-edge tech
    platforms=["all"],
    analysis_depth="detailed"
)

# Report structure:
# - Core findings with data highlights
# - Detailed news content with sources
# - Analysis and trends
# - Information propagation characteristics
```

### 5. Multi-Channel Push Notifications

```python
# Configure push task
from hotsearch_analysis_agent.push_manager import PushManager

push_manager = PushManager()

# Create push task
task = push_manager.create_task(
    name="AI技术热点监控",
    keywords=["GPT", "大模型", "DeepSeek"],
    platforms=["weibo", "zhihu"],
    channels=["wechat", "telegram", "email"],
    schedule="0 9,18 * * *",  # Cron format: 9 AM and 6 PM daily
    analysis_type="clustering"
)

# Test push manually
push_manager.test_push(task_id=task.id, channel="telegram")
```

### 6. Custom Spider Development

```python
# Add new platform spider in hotsearchcrawler/spiders/
import scrapy
from hotsearchcrawler.items import HotSearchItem

class NewPlatformSpider(scrapy.Spider):
    name = 'new_platform'
    
    def start_requests(self):
        url = 'https://newplatform.com/hot'
        yield scrapy.Request(url, callback=self.parse)
    
    def parse(self, response):
        for item in response.css('.hot-item'):
            hot_item = HotSearchItem()
            hot_item['platform'] = 'new_platform'
            hot_item['title'] = item.css('.title::text').get()
            hot_item['url'] = item.css('a::attr(href)').get()
            hot_item['heat_value'] = item.css('.heat::text').get()
            hot_item['rank_position'] = item.css('.rank::text').get()
            
            # Fetch detail page content
            if hot_item['url']:
                yield scrapy.Request(
                    hot_item['url'],
                    callback=self.parse_detail,
                    meta={'item': hot_item}
                )
    
    def parse_detail(self, response):
        item = response.meta['item']
        item['detail_content'] = response.css('.content::text').getall()
        yield item
```

## LLM Integration: Pangu Model

The project recommends Huawei's Pangu model for Chinese text analysis:

```python
# Local deployment reference
from hotsearch_analysis_agent.llm_client import LLMClient

client = LLMClient(
    api_base=os.getenv('LLM_API_BASE'),
    api_key=os.getenv('LLM_API_KEY'),
    model='pangu-embedded-7b'
)

# Use for analysis tasks
response = client.chat(
    messages=[
        {"role": "system", "content": "你是一个专业的舆情分析助手"},
        {"role": "user", "content": f"分析以下话题的情感倾向:\n{topic_content}"}
    ],
    temperature=0.7,
    max_tokens=2000
)
```

Download Pangu model: https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

## Common Workflows

### Daily Monitoring Workflow

```python
# 1. Schedule crawler runs (via cron or UI)
# 2. System automatically collects data every N hours
# 3. LLM analyzes new entries in background
# 4. Push notifications sent based on configured tasks

# Example: Morning briefing setup
from hotsearch_analysis_agent.scheduler import TaskScheduler

scheduler = TaskScheduler()
scheduler.add_task(
    name="morning_briefing",
    action="generate_report",
    keywords=["all"],
    channels=["email", "wechat"],
    cron="0 8 * * *"  # 8 AM daily
)
```

### Incident Tracking Workflow

```python
# Track specific event across platforms
from hotsearch_analysis_agent.tracker import EventTracker

tracker = EventTracker()
event = tracker.create_event(
    name="DeepSeek V4发布",
    keywords=["DeepSeek", "V4", "华为算力"],
    start_date="2026-04-01",
    platforms=["all"]
)

# Get timeline and sentiment evolution
timeline = tracker.get_timeline(event.id)
sentiment_trend = tracker.get_sentiment_trend(event.id)
```

## Troubleshooting

### Crawler Issues

**Problem**: Spider fails to extract data

```python
# Enable verbose logging in settings.py
LOG_LEVEL = 'DEBUG'

# Test spider in isolation
scrapy crawl weibo_spider -o output.json

# Check if CSS selectors need updating (platforms change frequently)
```

**Problem**: Detail page content missing (especially videos)

```bash
# Verify browser driver is accessible
which chromedriver

# Check if Selenium is properly configured
python -c "from selenium import webdriver; driver = webdriver.Chrome(); driver.quit()"
```

### Database Issues

**Problem**: Connection errors

```python
# Test connection manually
import pymysql
try:
    conn = pymysql.connect(
        host=os.getenv('MYSQL_HOST'),
        user=os.getenv('MYSQL_USER'),
        password=os.getenv('MYSQL_PASSWORD'),
        db=os.getenv('MYSQL_DB')
    )
    print("Connection successful")
except Exception as e:
    print(f"Error: {e}")
```

**Problem**: Character encoding issues with Chinese text

```sql
-- Ensure database uses utf8mb4
ALTER DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
ALTER TABLE hot_searches CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### LLM Analysis Issues

**Problem**: Analysis quality poor

```python
# Try adjusting temperature and prompt
response = client.chat(
    messages=[...],
    temperature=0.3,  # Lower for more focused analysis
    max_tokens=3000   # Increase for detailed reports
)

# Add few-shot examples to system prompt
system_prompt = """你是舆情分析专家。分析示例:
输入: [新闻标题]
输出: 
- 情感: 正面/中立/负面
- 关键点: [要点列表]
- 趋势: [判断]
"""
```

### Push Notification Issues

**Problem**: WeChat/Telegram push failing

```python
# Test push endpoints separately
from hotsearch_analysis_agent.push_channels import WeChatPusher

pusher = WeChatPusher(webhook_url=os.getenv('WECHAT_WEBHOOK'))
result = pusher.send_text("测试消息")
if not result.success:
    print(f"Error: {result.error}")
```

## Advanced Configuration

### Custom Analysis Prompts

```python
# Modify prompts in hotsearch_analysis_agent/prompts.py
SENTIMENT_ANALYSIS_PROMPT = """
作为舆情分析专家,分析以下内容的情感倾向:

内容: {content}

要求:
1. 判断整体情感(正面/中立/负面)
2. 提取关键情感词
3. 分析潜在影响
4. 给出置信度评分

输出JSON格式。
"""

# Use in analyzer
analyzer.set_prompt(SENTIMENT_ANALYSIS_PROMPT)
```

### Platform-Specific Configurations

```python
# Configure platform priorities and weights
PLATFORM_CONFIG = {
    'weibo': {'weight': 1.5, 'priority': 'high'},
    'zhihu': {'weight': 1.2, 'priority': 'medium'},
    'bilibili': {'weight': 1.0, 'priority': 'medium'},
    'douyin': {'weight': 1.3, 'priority': 'high'}
}

# Apply in clustering algorithm
cluster.set_platform_weights(PLATFORM_CONFIG)
```

## Testing

```bash
# Test push notifications
python test_push_task.py

# Test individual spider
python runspider-test.py

# Full system integration test
python -m pytest tests/
```

## Project Stats

- **15 platforms**: Weibo, Zhihu, Bilibili, Douyin, Toutiao, Baidu, etc.
- **26 hot lists**: Multiple ranking lists per platform
- **4 push channels**: Email, WeChat, Enterprise WeChat, Telegram
- **LLM-powered**: Sentiment analysis, clustering, report generation

This skill enables AI coding agents to help developers deploy and customize a production-ready Chinese social media monitoring system with advanced LLM analytics capabilities.
