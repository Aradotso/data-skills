---
name: llm-public-opinion-analytics-assistant
description: Build AI-powered public opinion analytics systems with multi-platform hot search crawling, sentiment analysis, and intelligent reporting
triggers:
  - how do I set up a public opinion monitoring system
  - integrate multi-platform hot search data crawling
  - analyze trending topics with sentiment analysis
  - create automated opinion analytics reports
  - build a hot search aggregation dashboard
  - implement intelligent news clustering and alerting
  - configure multi-channel opinion push notifications
  - crawl and analyze social media trends
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is an intelligent public opinion analytics assistant that combines real-time data from **15 mainstream platforms** and **26 ranking lists** with large language model analysis capabilities. It provides conversational hot search queries, topic clustering, sentiment analysis, and multi-channel alert pushing (email, WeChat, Telegram).

**Key capabilities:**
- Multi-platform web scraping (Weibo, Bilibili, Baidu, Zhihu, Douyin, etc.)
- Natural language query interface for trending topics
- AI-powered sentiment analysis and topic clustering
- Automated report generation with Huawei PanGu or OpenAI-compatible models
- Multi-channel push notifications (enterprise WeChat, Telegram, email)
- Video content extraction and analysis

## Installation

### Prerequisites

1. **Python 3.8+** with virtual environment
2. **MySQL 5.7+** database
3. **Browser driver** (Chrome/Edge) for web scraping

### Browser Driver Setup

```bash
# Download ChromeDriver matching your Chrome version
# From: https://chromedriver.chromium.org/

# Or EdgeDriver for Microsoft Edge
# From: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Add to system PATH or place in project directory
export PATH=$PATH:/path/to/chromedriver

# Verify installation
chromedriver --version
```

### Install Dependencies

```bash
# Clone the repository
git clone https://github.com/hmmnxkl/LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant.git
cd LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install requirements
pip install -r requirements.txt
```

### Database Setup

```python
# Reference init.py for database schema
import pymysql

# Create database connection
connection = pymysql.connect(
    host='localhost',
    user='your_user',
    password='your_password',
    charset='utf8mb4'
)

# Create database and tables (see init.py for full schema)
cursor = connection.cursor()
cursor.execute("CREATE DATABASE IF NOT EXISTS hotsearch_db CHARACTER SET utf8mb4")
cursor.execute("USE hotsearch_db")

# Main tables: hot_searches, news_details, analysis_results, push_tasks
```

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```bash
# Database configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch_db

# LLM API configuration (OpenAI-compatible)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Or use local PanGu model
USE_LOCAL_MODEL=true
PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b

# Push notification channels
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password

WECHAT_WEBHOOK_URL=your_webhook_url
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

### Crawler Settings

Edit `hotsearchcrawler/settings.py`:

```python
# MySQL connection for crawler
MYSQL_CONFIG = {
    'host': 'localhost',
    'port': 3306,
    'user': 'your_user',
    'password': 'your_password',
    'database': 'hotsearch_db',
    'charset': 'utf8mb4'
}

# Optional: Platform-specific cookies
COOKIES = {
    'weibo': 'your_weibo_cookies',
    'bilibili': 'your_bilibili_cookies'
}

# Scraping intervals
DOWNLOAD_DELAY = 3
CONCURRENT_REQUESTS = 8
```

## Core Usage

### Starting the Web Application

```bash
# Launch the main Flask application
python app.py

# Access at http://localhost:5000
```

### Running Crawlers

```python
# Method 1: Via web interface (recommended)
# Use the frontend dashboard to start/stop crawlers with hotkeys

# Method 2: Command line for testing
python runspider-test.py

# Method 3: Programmatic control
from hotsearchcrawler.run_spiders import SpiderManager

manager = SpiderManager()
manager.start_all_spiders()  # Start all platform crawlers
manager.stop_all_spiders()   # Stop all crawlers
```

### Querying Hot Searches

```python
from hotsearch_analysis_agent.query_engine import QueryEngine

# Initialize query engine
engine = QueryEngine(
    model_name="gpt-4",
    api_key=os.getenv("OPENAI_API_KEY")
)

# Natural language query
result = engine.query("最近关于人工智能的热点新闻有哪些?")
print(result['answer'])
print(result['sources'])  # Original URLs and metadata

# Platform-specific query
weibo_trends = engine.query("微博热搜前10", platform="weibo")

# Topic clustering
clusters = engine.cluster_topics(
    query="科技新闻",
    min_similarity=0.7
)
for cluster in clusters:
    print(f"Cluster: {cluster['topic']}")
    print(f"Articles: {len(cluster['items'])}")
```

### Sentiment Analysis

```python
from hotsearch_analysis_agent.sentiment_analyzer import SentimentAnalyzer

analyzer = SentimentAnalyzer()

# Analyze single article
sentiment = analyzer.analyze_text(
    "华为发布新款芯片,性能大幅提升,国产芯片迎来突破"
)
print(sentiment)  # {'label': 'positive', 'score': 0.92}

# Batch sentiment analysis
articles = [
    {"id": 1, "title": "...", "content": "..."},
    {"id": 2, "title": "...", "content": "..."}
]
results = analyzer.analyze_batch(articles)

# Trend analysis over time
trend = analyzer.get_sentiment_trend(
    topic="人工智能",
    start_date="2026-04-01",
    end_date="2026-04-07"
)
```

### Automated Report Generation

```python
from hotsearch_analysis_agent.report_generator import ReportGenerator

generator = ReportGenerator(
    model_name="gpt-4",
    template="analytical"  # or "summary", "detailed"
)

# Generate report for specific topic
report = generator.generate(
    query="人工智能与前沿科技",
    start_date="2026-04-01",
    end_date="2026-04-07",
    platforms=["weibo", "zhihu", "bilibili"],
    include_sentiment=True,
    include_clustering=True
)

# Report structure
print(report['title'])
print(report['summary'])
print(report['key_findings'])  # Bullet points
print(report['detailed_news'])  # Article list with URLs
print(report['analysis'])  # AI-generated insights
print(report['trend_chart_data'])  # For visualization
```

### Multi-Channel Push Notifications

```python
from hotsearch_analysis_agent.push_manager import PushManager

# Initialize push manager
push_mgr = PushManager()

# Create push task
task = {
    'query': '人工智能热点',
    'schedule': 'daily',  # or 'hourly', 'weekly'
    'time': '09:00',
    'channels': ['email', 'wechat', 'telegram'],
    'threshold': 100,  # Minimum hot search rank
    'keywords': ['AI', '大模型', '芯片']
}

# Register task
task_id = push_mgr.create_task(task)

# Manual push for testing
push_mgr.push_report(
    report=report,
    channels=['email'],
    recipients=['user@example.com']
)

# Test push configuration
python test_push_task.py
```

### Email Push Example

```python
from hotsearch_analysis_agent.push.email_pusher import EmailPusher

pusher = EmailPusher(
    smtp_host=os.getenv('SMTP_HOST'),
    smtp_port=int(os.getenv('SMTP_PORT')),
    username=os.getenv('SMTP_USER'),
    password=os.getenv('SMTP_PASSWORD')
)

pusher.send(
    to=['recipient@example.com'],
    subject='每日舆情报告 - 人工智能',
    html_content=report['html'],
    attachments=[
        {'filename': 'trend_chart.png', 'path': '/tmp/chart.png'}
    ]
)
```

### Enterprise WeChat Push

```python
from hotsearch_analysis_agent.push.wechat_pusher import WeChatPusher

# Webhook robot
pusher = WeChatPusher(
    webhook_url=os.getenv('WECHAT_WEBHOOK_URL')
)

pusher.send_markdown(
    title="热点分析报告",
    content=report['markdown']
)

# Enterprise WeChat app (推送到个人微信)
from hotsearch_analysis_agent.push.wechat_app_pusher import WeChatAppPusher

app_pusher = WeChatAppPusher(
    corp_id=os.getenv('WECHAT_CORP_ID'),
    agent_id=os.getenv('WECHAT_AGENT_ID'),
    secret=os.getenv('WECHAT_SECRET')
)

app_pusher.send_text(
    content=report['summary'],
    touser='UserID1|UserID2'
)
```

### Telegram Push

```python
from hotsearch_analysis_agent.push.telegram_pusher import TelegramPusher

pusher = TelegramPusher(
    bot_token=os.getenv('TELEGRAM_BOT_TOKEN')
)

pusher.send_message(
    chat_id=os.getenv('TELEGRAM_CHAT_ID'),
    text=report['markdown'],
    parse_mode='Markdown'
)
```

## Advanced Patterns

### Custom Crawler for New Platform

```python
# Create new spider in hotsearchcrawler/spiders/
import scrapy
from hotsearchcrawler.items import HotSearchItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://platform.com/trending']
    
    def parse(self, response):
        for item in response.css('.trend-item'):
            yield HotSearchItem(
                platform='custom_platform',
                title=item.css('.title::text').get(),
                url=item.css('a::attr(href)').get(),
                rank=item.css('.rank::text').get(),
                hot_value=item.css('.hot::text').get(),
                crawl_time=datetime.now()
            )
```

### Using Local PanGu Model

```python
from hotsearch_analysis_agent.models.pangu_model import PanGuModel

# Load local PanGu model
model = PanGuModel(
    model_path=os.getenv('PANGU_MODEL_PATH'),
    device='cuda'  # or 'cpu'
)

# Analysis with PanGu
analysis = model.analyze(
    prompt="分析以下新闻的情感倾向和关键信息:",
    content=news_content,
    max_length=2048
)

print(analysis['sentiment'])
print(analysis['key_points'])
print(analysis['summary'])
```

### Combining Multiple Data Sources

```python
from hotsearch_analysis_agent.data_fusion import DataFusion

fusion = DataFusion()

# Merge cross-platform data for same topic
merged = fusion.merge_topic(
    topic="DeepSeek V4",
    platforms=["weibo", "zhihu", "bilibili", "toutiao"],
    time_window_hours=24
)

# Deduplicate similar articles
unique_articles = fusion.deduplicate(
    articles=merged,
    similarity_threshold=0.85
)

# Cross-reference and enrich
enriched = fusion.cross_reference(
    articles=unique_articles,
    sources=['news_api', 'official_statements']
)
```

## Troubleshooting

### Crawler Issues

```python
# Browser driver not found
# Solution: Verify chromedriver is in PATH
import subprocess
result = subprocess.run(['which', 'chromedriver'], capture_output=True)
print(result.stdout.decode())

# Anti-scraping measures triggered
# Solution: Add delays and rotate user agents in settings.py
DOWNLOAD_DELAY = 5
USER_AGENT_LIST = [
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64)...',
    'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)...'
]
```

### Database Connection

```python
# Connection pool exhausted
# Solution: Increase pool size in settings
MYSQL_CONFIG = {
    'pool_size': 20,
    'max_overflow': 10,
    'pool_recycle': 3600
}

# Encoding issues
# Ensure all connections use utf8mb4
connection = pymysql.connect(
    charset='utf8mb4',
    collation='utf8mb4_unicode_ci'
)
```

### Model API Errors

```python
# Rate limiting
# Solution: Implement exponential backoff
from tenacity import retry, wait_exponential, stop_after_attempt

@retry(wait=wait_exponential(min=1, max=60), stop=stop_after_attempt(5))
def call_llm_api(prompt):
    response = client.chat.completions.create(
        model="gpt-4",
        messages=[{"role": "user", "content": prompt}]
    )
    return response.choices[0].message.content

# Context length exceeded
# Solution: Chunk long articles
def chunk_text(text, max_tokens=4000):
    # Split on paragraphs, keeping under token limit
    chunks = []
    current_chunk = ""
    for paragraph in text.split('\n\n'):
        if len(current_chunk) + len(paragraph) < max_tokens * 4:  # rough estimate
            current_chunk += paragraph + '\n\n'
        else:
            chunks.append(current_chunk)
            current_chunk = paragraph + '\n\n'
    if current_chunk:
        chunks.append(current_chunk)
    return chunks
```

### Performance Optimization

```python
# Slow queries
# Solution: Add database indexes
CREATE INDEX idx_platform_time ON hot_searches(platform, crawl_time);
CREATE INDEX idx_keywords ON hot_searches(title);
CREATE FULLTEXT INDEX ft_content ON news_details(content);

# High memory usage during analysis
# Solution: Process in batches
def analyze_large_dataset(articles, batch_size=50):
    results = []
    for i in range(0, len(articles), batch_size):
        batch = articles[i:i+batch_size]
        batch_results = analyzer.analyze_batch(batch)
        results.extend(batch_results)
        # Clear GPU cache if using local model
        if torch.cuda.is_available():
            torch.cuda.empty_cache()
    return results
```

## Common Workflows

### Daily Opinion Monitoring

```python
# Complete workflow for daily monitoring
from datetime import datetime, timedelta

# 1. Run crawlers
manager.start_all_spiders()

# 2. Wait for completion
time.sleep(1800)  # 30 minutes

# 3. Generate daily report
yesterday = datetime.now() - timedelta(days=1)
report = generator.generate(
    query="全部热点",
    start_date=yesterday.strftime("%Y-%m-%d"),
    platforms="all"
)

# 4. Push to configured channels
push_mgr.push_report(report, channels=['email', 'wechat'])

# 5. Archive results
archive_report(report, f"reports/{datetime.now().strftime('%Y%m%d')}.html")
```

This skill covers the essential operations for deploying and using the LLM-based public opinion analytics assistant in real-world monitoring scenarios.
