---
name: llm-public-opinion-analytics-assistant
description: Multi-platform hot topic crawler with LLM-powered sentiment analysis, clustering, and multi-channel push notifications for public opinion monitoring
triggers:
  - how do I analyze public opinion from social media platforms
  - set up a hot topic monitoring system with sentiment analysis
  - crawl trending topics from multiple Chinese platforms
  - configure sentiment analysis with Pangu LLM model
  - create automated public opinion reports with email and WeChat push
  - monitor real-time hot searches across Weibo Bilibili Douyin
  - build a topic clustering system for social media trends
  - integrate LLM analysis for news sentiment and opinion mining
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is a comprehensive public opinion analytics assistant that integrates real-time data from **15 mainstream platforms** across **26 trending lists** with large language model analysis capabilities. It provides conversational hot topic queries, theme-based searches, topic clustering, sentiment analysis, and multi-channel push notifications (email, WeChat, Enterprise WeChat, Telegram).

## What This Project Does

- **Multi-Platform Data Collection**: Crawls trending topics from Weibo, Bilibili, Douyin, Zhihu, Baidu, and 10+ other Chinese platforms
- **LLM-Powered Analysis**: Uses Pangu or OpenAI-compatible models for sentiment analysis, topic clustering, and report generation
- **Conversational Interface**: Natural language queries for hot topic exploration
- **Deep Content Extraction**: Fetches full article/video content from detail pages for comprehensive analysis
- **Multi-Channel Push**: Automated reports via email (SMTP), WeChat, Enterprise WeChat, Telegram
- **Crawler Control**: Keyboard shortcuts to start/stop crawlers via web UI

## Project Architecture

```
project/
├── hotsearchcrawler/          # Scrapy-based crawler cluster
│   ├── spiders/               # Platform-specific spiders
│   ├── settings.py            # Crawler configuration
│   └── pipelines.py           # Data pipeline to MySQL
├── hotsearch_analysis_agent/  # LLM analysis system
│   ├── agent.py               # Main agent logic
│   ├── models/                # Database models
│   └── utils/                 # Helper functions
├── app.py                     # Flask web application
├── run_spiders.py             # Crawler launcher
└── .env                       # Configuration file
```

## Installation

### Prerequisites

1. **Browser Driver Setup** (for detail page scraping):
```bash
# Download ChromeDriver or EdgeDriver matching your browser version
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Place driver in system PATH or project directory
# Verify installation:
chromedriver --version
```

2. **Python Environment**:
```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # Linux/Mac
# or
venv\Scripts\activate  # Windows

# Install dependencies
pip install -r requirements.txt
```

3. **MySQL Database**:
```bash
# Install MySQL 5.7+ and create database
mysql -u root -p
CREATE DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

4. **Initialize Database Tables**:
```python
# Reference init.py for table schemas
# Key tables: hot_searches, analysis_results, push_tasks
python init.py
```

## Configuration

### 1. Environment Variables (.env)

```bash
# Database Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_username
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
OPENAI_MODEL=gpt-4

# Or use Pangu model (recommended)
PANGU_API_BASE=http://localhost:8000/v1
PANGU_MODEL=openpangu-embedded-7b

# Push Notification Channels
# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
SMTP_FROM=your_email@gmail.com

# Enterprise WeChat Bot
WECHAT_BOT_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Web UI
FLASK_SECRET_KEY=your_random_secret_key
FLASK_PORT=5000
```

### 2. Crawler Configuration (hotsearchcrawler/settings.py)

```python
# MySQL connection for crawler
MYSQL_CONFIG = {
    'host': 'localhost',
    'port': 3306,
    'user': 'your_username',
    'password': 'your_password',
    'database': 'hotsearch',
    'charset': 'utf8mb4'
}

# Optional: Platform cookies for authenticated access
COOKIES = {
    'weibo': 'your_weibo_cookie',
    'bilibili': 'your_bilibili_cookie'
}

# Concurrent requests
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
```

## Core Usage

### Starting the Web Application

```bash
# Start Flask server
python app.py

# Access web UI at http://localhost:5000
```

### Running Crawlers

```python
# Method 1: Via web UI
# Navigate to UI, use keyboard shortcuts to start/stop crawlers

# Method 2: Direct execution
python run_spiders.py

# Method 3: Test specific spider
python runspider-test.py --spider weibo
```

### Programmatic API Usage

#### 1. Query Hot Topics

```python
from hotsearch_analysis_agent.agent import HotSearchAgent
from hotsearch_analysis_agent.models import get_db_session

# Initialize agent
agent = HotSearchAgent()
session = get_db_session()

# Query hot topics by platform
query = "帮我查询微博热搜前10条"
response = agent.process_query(query, session)
print(response)

# Query specific theme
query = "最近关于人工智能的热点有哪些"
response = agent.process_query(query, session)
print(response)
```

#### 2. Sentiment Analysis

```python
from hotsearch_analysis_agent.analyzer import SentimentAnalyzer

analyzer = SentimentAnalyzer()

# Analyze single topic
topic = "GPT-6遭提前曝光, 2M超长上下文来了"
content = "社区流传信息显示，下一代GPT模型或支持200万token上下文窗口..."
sentiment = analyzer.analyze_sentiment(topic, content)

print(f"Sentiment: {sentiment['sentiment']}")  # positive/negative/neutral
print(f"Score: {sentiment['score']}")  # -1.0 to 1.0
print(f"Keywords: {sentiment['keywords']}")
```

#### 3. Topic Clustering

```python
from hotsearch_analysis_agent.clustering import TopicClusterer
from hotsearch_analysis_agent.models import HotSearch

# Get recent hot searches
session = get_db_session()
topics = session.query(HotSearch).filter(
    HotSearch.created_at >= datetime.now() - timedelta(days=1)
).all()

# Perform clustering
clusterer = TopicClusterer()
clusters = clusterer.cluster_topics(topics)

for cluster_id, cluster_topics in clusters.items():
    print(f"Cluster {cluster_id}:")
    for topic in cluster_topics:
        print(f"  - {topic.title} ({topic.platform})")
```

#### 4. Generate Analysis Report

```python
from hotsearch_analysis_agent.report_generator import ReportGenerator

generator = ReportGenerator()

# Generate report for specific theme
theme = "人工智能与前沿科技"
report = generator.generate_report(
    theme=theme,
    days=7,  # Last 7 days
    platforms=['weibo', 'bilibili', 'zhihu'],
    include_sentiment=True,
    include_clustering=True
)

print(report['markdown'])  # Markdown formatted report
print(report['summary'])   # Executive summary
```

### Setting Up Push Tasks

```python
from hotsearch_analysis_agent.push_manager import PushTaskManager
from hotsearch_analysis_agent.models import PushTask

manager = PushTaskManager()

# Create email push task
task = PushTask(
    name="AI科技日报",
    theme="人工智能与前沿科技",
    channels=['email'],
    schedule_type='daily',  # daily/weekly/realtime
    schedule_time='08:00',
    recipients=['user@example.com'],
    enabled=True
)

session.add(task)
session.commit()

# Test push immediately
manager.execute_task(task.id)
```

### Multi-Channel Push Configuration

```python
# Email push
email_config = {
    'type': 'email',
    'recipients': ['user1@example.com', 'user2@example.com'],
    'subject_template': '舆情日报 - {date}',
    'html_template': True
}

# Enterprise WeChat Bot
wechat_bot_config = {
    'type': 'wechat_bot',
    'webhook': os.getenv('WECHAT_BOT_WEBHOOK'),
    'msgtype': 'markdown'  # text/markdown
}

# Telegram Bot
telegram_config = {
    'type': 'telegram',
    'bot_token': os.getenv('TELEGRAM_BOT_TOKEN'),
    'chat_id': os.getenv('TELEGRAM_CHAT_ID'),
    'parse_mode': 'Markdown'
}

# Register channels
task.push_config = {
    'channels': [email_config, wechat_bot_config, telegram_config]
}
```

## Working with Crawlers

### Custom Spider Example

```python
# hotsearchcrawler/spiders/custom_spider.py
import scrapy
from hotsearchcrawler.items import HotSearchItem

class CustomSpider(scrapy.Spider):
    name = 'custom_platform'
    allowed_domains = ['example.com']
    start_urls = ['https://example.com/trending']
    
    def parse(self, response):
        for item in response.css('.trending-item'):
            yield HotSearchItem(
                platform='custom_platform',
                title=item.css('.title::text').get(),
                url=item.css('a::attr(href)').get(),
                rank=item.css('.rank::text').get(),
                heat_value=item.css('.heat::text').get(),
                category=item.css('.category::text').get()
            )
```

### Extracting Detail Page Content

```python
from hotsearchcrawler.content_extractor import ContentExtractor

extractor = ContentExtractor()

# Extract text content
url = "https://www.toutiao.com/article/7625514106853818934/"
content = extractor.extract_content(url)

print(content['title'])
print(content['text'])
print(content['author'])
print(content['publish_time'])

# Extract video content (with subtitle extraction)
video_url = "https://www.bilibili.com/video/BV13pSoBBEvX/"
video_content = extractor.extract_video_content(video_url)

print(video_content['title'])
print(video_content['description'])
print(video_content['subtitles'])  # AI-generated or uploaded subtitles
```

## Common Patterns

### Real-Time Monitoring Loop

```python
import time
from datetime import datetime, timedelta

def monitor_hot_topics(keywords, interval_minutes=30):
    """Monitor hot topics matching keywords"""
    agent = HotSearchAgent()
    session = get_db_session()
    
    while True:
        # Query recent topics
        recent_topics = session.query(HotSearch).filter(
            HotSearch.created_at >= datetime.now() - timedelta(minutes=interval_minutes)
        ).all()
        
        # Filter by keywords
        matched = [t for t in recent_topics if any(kw in t.title for kw in keywords)]
        
        if matched:
            # Trigger analysis
            report = generator.generate_report(
                topics=matched,
                include_sentiment=True
            )
            
            # Push notification
            push_manager.push_immediate(report, channels=['wechat_bot'])
        
        time.sleep(interval_minutes * 60)

# Usage
monitor_hot_topics(['人工智能', 'AI', '大模型'], interval_minutes=30)
```

### Batch Analysis Pipeline

```python
from concurrent.futures import ThreadPoolExecutor

def analyze_platform_batch(platform, date_range):
    """Batch analyze topics from specific platform"""
    session = get_db_session()
    
    topics = session.query(HotSearch).filter(
        HotSearch.platform == platform,
        HotSearch.created_at.between(*date_range)
    ).all()
    
    analyzer = SentimentAnalyzer()
    
    with ThreadPoolExecutor(max_workers=5) as executor:
        futures = [
            executor.submit(analyzer.analyze_sentiment, t.title, t.content)
            for t in topics
        ]
        
        results = [f.result() for f in futures]
    
    return results

# Analyze all Weibo topics from last week
from datetime import datetime, timedelta
end_date = datetime.now()
start_date = end_date - timedelta(days=7)
results = analyze_platform_batch('weibo', (start_date, end_date))
```

### Custom Report Template

```python
def generate_custom_report(topics, format='markdown'):
    """Generate custom-formatted report"""
    
    report = f"# 舆情分析报告\n\n"
    report += f"**生成时间**: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}\n\n"
    
    # Group by platform
    by_platform = {}
    for topic in topics:
        by_platform.setdefault(topic.platform, []).append(topic)
    
    for platform, platform_topics in by_platform.items():
        report += f"## {platform.upper()} 热点\n\n"
        
        for idx, topic in enumerate(platform_topics[:10], 1):
            report += f"{idx}. **{topic.title}**\n"
            report += f"   - 热度: {topic.heat_value}\n"
            report += f"   - 链接: {topic.url}\n"
            
            if topic.sentiment_score:
                sentiment = "正面" if topic.sentiment_score > 0.2 else "负面" if topic.sentiment_score < -0.2 else "中性"
                report += f"   - 情感: {sentiment}\n"
            
            report += "\n"
    
    return report
```

## Troubleshooting

### Crawler Issues

**Problem**: Spider returns empty results
```python
# Check if cookies are needed
# Add cookies in hotsearchcrawler/settings.py
COOKIES = {
    'platform_name': 'cookie_string_from_browser'
}

# Test with scrapy shell
scrapy shell "https://target-url.com"
response.css('.selector').getall()
```

**Problem**: ChromeDriver version mismatch
```bash
# Check Chrome version
google-chrome --version  # Linux
# Or check in browser: chrome://version

# Download matching driver
# Place in PATH or specify in code:
from selenium import webdriver
driver = webdriver.Chrome(executable_path='/path/to/chromedriver')
```

### Database Connection Issues

```python
# Test connection
from sqlalchemy import create_engine
engine = create_engine(
    f"mysql+pymysql://{user}:{password}@{host}:{port}/{database}?charset=utf8mb4"
)
connection = engine.connect()
print("Connected successfully")
connection.close()
```

### LLM API Issues

```python
# Test OpenAI-compatible API
import openai
openai.api_base = os.getenv('OPENAI_API_BASE')
openai.api_key = os.getenv('OPENAI_API_KEY')

response = openai.ChatCompletion.create(
    model=os.getenv('OPENAI_MODEL'),
    messages=[{"role": "user", "content": "测试连接"}]
)
print(response.choices[0].message.content)
```

### Push Notification Failures

```python
# Test email
from hotsearch_analysis_agent.push_channels import EmailChannel
channel = EmailChannel()
channel.send_test_message('recipient@example.com')

# Test WeChat bot
import requests
webhook = os.getenv('WECHAT_BOT_WEBHOOK')
response = requests.post(webhook, json={
    "msgtype": "text",
    "text": {"content": "测试消息"}
})
print(response.json())
```

### Memory Issues with Large Datasets

```python
# Use pagination for large queries
from sqlalchemy import func

def process_large_dataset(batch_size=1000):
    session = get_db_session()
    total = session.query(func.count(HotSearch.id)).scalar()
    
    for offset in range(0, total, batch_size):
        batch = session.query(HotSearch).limit(batch_size).offset(offset).all()
        process_batch(batch)
        session.expunge_all()  # Clear session cache
```

## Advanced Configuration

### Using Pangu Model Locally

```bash
# Download model from GitCode
# https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

# Serve with vLLM or FastAPI
python -m vllm.entrypoints.openai.api_server \
    --model /path/to/openpangu-embedded-7b \
    --host 0.0.0.0 \
    --port 8000

# Update .env
OPENAI_API_BASE=http://localhost:8000/v1
OPENAI_MODEL=openpangu-embedded-7b
```

### Custom Analysis Prompts

```python
# Modify prompt templates in hotsearch_analysis_agent/prompts.py
SENTIMENT_PROMPT = """
分析以下舆情信息的情感倾向和关键观点:

标题: {title}
内容: {content}
来源: {platform}

请从以下维度分析:
1. 整体情感倾向(正面/负面/中性)
2. 情感强度(0-10分)
3. 关键观点提取
4. 潜在风险点
"""

# Use in analyzer
result = llm.complete(SENTIMENT_PROMPT.format(
    title=topic.title,
    content=topic.content,
    platform=topic.platform
))
```

This skill enables AI agents to effectively use the LLM-Based Intelligent Public Opinion Analytics Assistant for comprehensive social media monitoring and sentiment analysis in Chinese platforms.
