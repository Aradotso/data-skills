---
name: llm-public-opinion-analytics-assistant
description: Deploy and operate an LLM-powered public opinion analytics system that crawls 26 trending lists from 15 platforms, performs sentiment analysis, topic clustering, and sends multi-channel alerts
triggers:
  - how do I set up the public opinion analytics assistant
  - configure the hot search crawler and LLM analysis system
  - how to analyze trending topics with LLM sentiment analysis
  - set up multi-channel alerts for trending news
  - deploy the Chinese public opinion monitoring system
  - configure crawler for Weibo Bilibili and other platforms
  - how to use Pangu model for sentiment analysis
  - integrate trending topic alerts with WeChat and Telegram
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is an intelligent public opinion analytics assistant that combines real-time data from 26 trending lists across 15 mainstream Chinese platforms with LLM analysis capabilities. It supports conversational hot search queries, topic-specific searches, topic clustering, sentiment analysis, and multi-channel alert pushes (email, WeChat, Enterprise WeChat, Telegram).

## What It Does

- **Multi-Platform Crawling**: Scrapes trending topics from 15 platforms including Weibo, Bilibili, Douyin, Baidu, Zhihu, etc.
- **LLM-Powered Analysis**: Uses large language models (supports Pangu, OpenAI-compatible APIs) for sentiment analysis, topic clustering, and report generation
- **Conversational Interface**: Natural language queries for trending topics, theme searches, and analytics
- **Video Content Extraction**: Extracts insights from video-based news content
- **Multi-Channel Alerts**: Automated push notifications via Enterprise WeChat, Telegram, Email (SMTP)
- **Real-Time Control**: Hotkey-based crawler start/stop control

## Installation

### Prerequisites

1. **Python 3.8+** with virtual environment
2. **MySQL Database** for storing crawled data
3. **Browser Driver** (Chrome/Edge) for dynamic content scraping
4. **LLM Access** (Pangu model or OpenAI-compatible API)

### Browser Driver Setup

```bash
# Download ChromeDriver matching your Chrome version
# Visit https://chromedriver.chromium.org/

# For Linux/macOS, move to PATH
sudo mv chromedriver /usr/local/bin/
chmod +x /usr/local/bin/chromedriver

# Verify installation
chromedriver --version
```

### Project Installation

```bash
# Clone the repository
git clone https://github.com/hmmnxkl/LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant.git
cd LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Database Setup

```bash
# Create MySQL database
mysql -u root -p

# In MySQL shell:
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

```python
# Reference init.py for table structure
# Key tables:
# - hot_search_items: Stores trending topics
# - analysis_results: Stores LLM analysis outputs
# - push_tasks: Manages alert schedules
```

## Configuration

### 1. Crawler Configuration

Edit `hotsearchcrawler/settings.py`:

```python
# MySQL connection
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'root'
MYSQL_PASSWORD = 'your_password'
MYSQL_DB = 'hotsearch_db'

# Optional: Platform-specific cookies for authenticated access
COOKIES = {
    'weibo': 'your_weibo_cookie',
    'bilibili': 'your_bilibili_cookie'
}
```

### 2. Analysis System Configuration

Create `.env` file in project root:

```env
# Database
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=your_password
MYSQL_DB=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Or use Pangu model locally
# PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model

# Push Notifications
# Enterprise WeChat Robot
WECHAT_ROBOT_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=your_key

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
```

### 3. Pangu Model Setup (Optional)

```bash
# Download Pangu model
# Visit: https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

# Extract and configure path in .env
PANGU_MODEL_PATH=/opt/models/openpangu-embedded-7b-model
```

## Running the Application

### Start the Web Application

```bash
# From project root
python app.py
```

Access the web interface at `http://localhost:5000`

### Start Crawlers

```python
# Method 1: Via web interface (recommended)
# Use the UI to start/stop crawlers with hotkeys

# Method 2: Direct script execution
python run_spiders.py
```

### Test Crawlers

```bash
# Test individual spider
python runspider-test.py
```

### Test Push Notifications

```bash
# Test push task configuration
python test_push_task.py
```

## Core API Usage

### Querying Trending Topics

```python
from hotsearch_analysis_agent.query_engine import QueryEngine

# Initialize query engine
engine = QueryEngine()

# Natural language query
results = engine.query("今天微博热搜有哪些关于人工智能的话题?")
# Returns: List of AI-related trending topics from Weibo

# Platform-specific query
results = engine.query_platform("bilibili", limit=20)
# Returns: Top 20 trending topics from Bilibili
```

### Sentiment Analysis

```python
from hotsearch_analysis_agent.sentiment_analyzer import SentimentAnalyzer

analyzer = SentimentAnalyzer(model_name="pangu")

# Analyze single topic
topic_text = "GPT-6遭提前曝光, 2M超长上下文来了"
sentiment = analyzer.analyze(topic_text)
# Returns: {'sentiment': 'positive', 'score': 0.85, 'keywords': ['GPT-6', '上下文']}

# Batch analysis
topics = ["新闻标题1", "新闻标题2", "新闻标题3"]
results = analyzer.batch_analyze(topics)
```

### Topic Clustering

```python
from hotsearch_analysis_agent.cluster_analyzer import ClusterAnalyzer

analyzer = ClusterAnalyzer()

# Cluster recent topics by theme
clusters = analyzer.cluster_topics(
    query="人工智能",
    time_range="7d",
    min_cluster_size=3
)

# Returns:
# [
#   {
#     'theme': '大模型技术进展',
#     'topics': [...],
#     'sentiment': 'positive',
#     'trend': 'rising'
#   },
#   ...
# ]
```

### Generate Analytics Report

```python
from hotsearch_analysis_agent.report_generator import ReportGenerator

generator = ReportGenerator()

# Generate comprehensive report
report = generator.generate(
    query="人工智能与前沿科技",
    time_range="7d",
    include_sentiment=True,
    include_clustering=True
)

# Export report
generator.export_markdown(report, "report_2026-05-19.md")
generator.export_html(report, "report_2026-05-19.html")
```

### Configure Push Tasks

```python
from hotsearch_analysis_agent.push_manager import PushManager

manager = PushManager()

# Create scheduled push task
task = manager.create_task(
    name="AI热点日报",
    query="人工智能",
    schedule="0 12 * * *",  # Daily at 12:00
    channels=["wechat", "telegram", "email"],
    recipients={
        "email": ["team@example.com"],
        "telegram": ["@channel_id"]
    }
)

# Test push immediately
manager.test_push(task_id=task.id)

# List active tasks
tasks = manager.list_tasks()

# Stop task
manager.stop_task(task_id=task.id)
```

## Common Patterns

### Daily Monitoring Workflow

```python
from hotsearch_analysis_agent import HotSearchAnalyzer

# Initialize analyzer
analyzer = HotSearchAnalyzer(
    db_config={
        'host': 'localhost',
        'user': 'root',
        'password': 'your_password',
        'database': 'hotsearch_db'
    },
    llm_config={
        'model': 'pangu',
        'model_path': '/opt/models/openpangu-embedded-7b-model'
    }
)

# Daily analysis routine
def daily_analysis():
    # Get today's trending topics
    topics = analyzer.get_trending(time_range="24h", platforms="all")
    
    # Cluster by theme
    clusters = analyzer.cluster(topics, method="semantic")
    
    # Sentiment analysis
    for cluster in clusters:
        cluster['sentiment'] = analyzer.analyze_sentiment(cluster['topics'])
    
    # Generate report
    report = analyzer.generate_report(clusters, template="daily")
    
    # Push to channels
    analyzer.push(report, channels=["wechat", "email"])

# Schedule daily
from apscheduler.schedulers.blocking import BlockingScheduler
scheduler = BlockingScheduler()
scheduler.add_job(daily_analysis, 'cron', hour=12, minute=0)
scheduler.start()
```

### Custom Platform Integration

```python
# Extend crawler for new platform
from hotsearchcrawler.spiders.base import BaseSpider

class CustomPlatformSpider(BaseSpider):
    name = 'custom_platform'
    allowed_domains = ['custom.com']
    start_urls = ['https://custom.com/trending']
    
    def parse(self, response):
        for item in response.css('.trending-item'):
            yield {
                'title': item.css('.title::text').get(),
                'url': item.css('a::attr(href)').get(),
                'hot_score': item.css('.score::text').get(),
                'platform': self.name,
                'timestamp': datetime.now()
            }
```

### Multi-Source Topic Aggregation

```python
from hotsearch_analysis_agent.aggregator import TopicAggregator

aggregator = TopicAggregator()

# Aggregate same topic from different platforms
topics = aggregator.aggregate(
    query="iPhone 16",
    platforms=["weibo", "douyin", "bilibili", "zhihu"],
    time_range="3d"
)

# Cross-platform sentiment comparison
for topic in topics:
    print(f"{topic['platform']}: {topic['sentiment']} (热度: {topic['hot_score']})")
```

## Troubleshooting

### Crawler Issues

**Problem**: Browser driver not found
```bash
# Solution: Verify driver in PATH
which chromedriver  # Linux/macOS
where chromedriver  # Windows

# Or specify driver path in settings.py
CHROME_DRIVER_PATH = '/usr/local/bin/chromedriver'
```

**Problem**: Platform blocking requests
```python
# Solution: Add delays and user agents in settings.py
DOWNLOAD_DELAY = 3
RANDOMIZE_DOWNLOAD_DELAY = True
USER_AGENT = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'

# Use cookies for authenticated access
COOKIES_ENABLED = True
```

### LLM Analysis Issues

**Problem**: Out of memory when loading Pangu model
```python
# Solution: Use quantized model or reduce batch size
MODEL_QUANTIZATION = "int8"  # or "int4"
BATCH_SIZE = 4  # Reduce from default 16
```

**Problem**: Slow analysis performance
```python
# Solution: Enable caching and parallel processing
ENABLE_CACHE = True
CACHE_TTL = 3600  # 1 hour
MAX_WORKERS = 4  # Parallel analysis threads
```

### Database Issues

**Problem**: Connection pool exhausted
```python
# Solution: Adjust connection pool settings
MYSQL_POOL_SIZE = 20
MYSQL_MAX_OVERFLOW = 10
MYSQL_POOL_RECYCLE = 3600
```

**Problem**: Duplicate entries
```sql
-- Solution: Add unique constraint
ALTER TABLE hot_search_items 
ADD UNIQUE KEY unique_topic (platform, title, DATE(created_at));
```

### Push Notification Issues

**Problem**: Enterprise WeChat webhook fails
```python
# Solution: Verify webhook URL and test with curl
import requests

webhook = "https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY"
test_msg = {
    "msgtype": "text",
    "text": {"content": "测试消息"}
}
response = requests.post(webhook, json=test_msg)
print(response.json())  # Should return {"errcode":0,"errmsg":"ok"}
```

**Problem**: Email not sending
```python
# Solution: Check SMTP settings and enable less secure apps
SMTP_USE_TLS = True
SMTP_TIMEOUT = 30

# Test SMTP connection
import smtplib
server = smtplib.SMTP('smtp.gmail.com', 587)
server.starttls()
server.login('your_email@gmail.com', 'app_password')
```

## Advanced Features

### Custom Analysis Prompts

```python
# Customize LLM analysis prompts
from hotsearch_analysis_agent.config import AnalysisConfig

config = AnalysisConfig()
config.sentiment_prompt = """
分析以下新闻标题的情感倾向。请返回JSON格式:
{"sentiment": "positive/negative/neutral", "confidence": 0.0-1.0, "reason": "简要说明"}

新闻标题: {title}
"""

config.cluster_prompt = """
将以下主题进行聚类分析，识别共同主题和趋势...
"""

analyzer = SentimentAnalyzer(config=config)
```

### Real-Time Monitoring

```python
# WebSocket-based real-time updates
from hotsearch_analysis_agent.realtime import RealtimeMonitor

monitor = RealtimeMonitor()

@monitor.on_new_topic
def handle_new_topic(topic):
    print(f"新热搜: {topic['title']} (热度: {topic['hot_score']})")
    
    # Immediate sentiment analysis
    sentiment = analyzer.analyze(topic['title'])
    
    # Alert if negative sentiment above threshold
    if sentiment['sentiment'] == 'negative' and sentiment['score'] > 0.8:
        push_manager.immediate_alert(topic, channels=['telegram'])

monitor.start(platforms=['weibo', 'douyin'])
```

This skill provides comprehensive coverage of the LLM-Based Public Opinion Analytics Assistant, enabling AI coding agents to effectively deploy, configure, and operate this sophisticated trending topic monitoring and analysis system.
