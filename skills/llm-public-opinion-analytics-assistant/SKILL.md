---
name: llm-public-opinion-analytics-assistant
description: Intelligent public opinion analytics assistant that aggregates 26 hot search lists from 15 platforms with LLM-powered sentiment analysis, topic clustering, and multi-channel alert push
triggers:
  - how do I set up a public opinion monitoring system
  - analyze hot topics across Chinese social media platforms
  - configure multi-platform hot search crawler with LLM analysis
  - implement sentiment analysis for social media trends
  - create automated topic clustering and alerts for trending news
  - build a real-time opinion analytics dashboard with AI
  - monitor and analyze Weibo Douyin Bilibili trending topics
  - set up automated hot topic push notifications
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is a comprehensive public opinion analytics platform that:

- **Crawls 26 hot search lists** from 15 mainstream Chinese platforms (Weibo, Douyin, Bilibili, Baidu, Zhihu, etc.)
- **Analyzes content using LLMs** (supports Huawei Pangu, OpenAI-compatible APIs) for sentiment analysis, topic clustering, and trend detection
- **Extracts detailed content** from news pages including video-based content
- **Provides conversational interface** for natural language queries about trending topics
- **Pushes automated reports** via email, WeChat Work, Enterprise WeChat, and Telegram
- **Supports keyboard shortcuts** for crawler control and multi-platform data quick search

The architecture separates crawlers (`hotsearchcrawler`) from the analysis system (`hotsearch_analysis_agent`), enabling independent scaling and maintenance.

## Installation

### Prerequisites

1. **Browser Driver Setup** (required for content extraction):

```bash
# Check your Chrome/Edge version first
google-chrome --version  # or microsoft-edge --version

# Download matching ChromeDriver/EdgeDriver
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/microsoft-edge/tools/webdriver/

# Add to PATH (Linux/macOS example)
sudo mv chromedriver /usr/local/bin/
sudo chmod +x /usr/local/bin/chromedriver

# Verify installation
chromedriver --version
```

2. **MySQL Database**:

```bash
# Install MySQL 8.0+
sudo apt-get install mysql-server  # Ubuntu/Debian
brew install mysql                  # macOS

# Create database and tables using init.py reference
mysql -u root -p
CREATE DATABASE public_opinion CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

3. **Python Environment**:

```bash
# Clone repository
git clone https://github.com/hmmnxkl/LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant.git
cd LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant

# Create virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Configuration

1. **Database Configuration** (`hotsearchcrawler/settings.py`):

```python
# MySQL connection settings
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_user'
MYSQL_PASSWORD = 'your_password'
MYSQL_DB = 'public_opinion'
```

2. **Analysis System Configuration** (`.env` file in project root):

```bash
# Database
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DB=public_opinion

# LLM API (OpenAI-compatible endpoint)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Or use local Huawei Pangu model
# PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model
# USE_LOCAL_MODEL=true

# Push notifications
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password

WECHAT_WORK_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=your_key
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

3. **Optional Platform Cookies** (for accessing restricted content):

Edit `hotsearchcrawler/settings.py`:

```python
# Example: Weibo cookies
WEIBO_COOKIES = {
    'SUB': 'your_sub_cookie',
    'SUBP': 'your_subp_cookie'
}
```

## Running the System

### Start the Web Application

```bash
# Launch the main application server
python app.py

# Access the frontend at http://localhost:5000
```

### Control Crawlers

**From Frontend UI** (recommended):
- Use the keyboard shortcut to start/stop crawlers
- View real-time crawling status

**Manual Crawler Execution**:

```bash
# Test individual spider
python runspider-test.py

# Run all spiders (production)
python run_spiders.py
```

### Initialize Database

```bash
# Create tables and initial schema
python init.py
```

## Key Features & Usage

### 1. Conversational Hot Search Query

Query trending topics using natural language:

```python
from hotsearch_analysis_agent.query_engine import QueryEngine

engine = QueryEngine()

# Ask about specific topics
response = engine.query("最近人工智能领域有什么热点?")
print(response['answer'])
print(response['sources'])  # Source articles with links

# Query specific platforms
response = engine.query("微博今天热搜前10条是什么?")

# Sentiment analysis
response = engine.query("分析关于某某事件的情感倾向")
```

### 2. Topic Clustering Analysis

Automatically cluster related topics:

```python
from hotsearch_analysis_agent.clustering import TopicClusterer

clusterer = TopicClusterer()

# Cluster today's hot topics
clusters = clusterer.analyze_daily_topics()

for cluster in clusters:
    print(f"Cluster: {cluster['theme']}")
    print(f"Topics: {len(cluster['topics'])}")
    print(f"Platforms: {cluster['platforms']}")
    print(f"Sentiment: {cluster['sentiment_score']}")
```

### 3. Automated Push Notifications

Configure and test push tasks:

```python
from hotsearch_analysis_agent.push_manager import PushManager

manager = PushManager()

# Create a monitoring task
task = manager.create_task(
    name="AI Tech Monitoring",
    keywords=["人工智能", "大模型", "GPT"],
    platforms=["weibo", "zhihu", "bilibili"],
    push_channels=["email", "wechat_work"],
    schedule="0 9,18 * * *",  # 9 AM and 6 PM daily
    min_hot_score=1000
)

# Test push manually
manager.test_push(task_id=task.id, channel="email")
```

**Push Task Configuration**:

```python
# Example push task in database or config
{
    "task_name": "Tech Trends Alert",
    "query": "前沿科技 OR 人工智能",
    "filters": {
        "platforms": ["weibo", "douyin", "bilibili"],
        "min_engagement": 10000,
        "sentiment": ["positive", "neutral"]
    },
    "analysis_depth": "detailed",  # or "summary"
    "push_config": {
        "email": {
            "recipients": ["analyst@company.com"],
            "subject_template": "Daily Tech Trends - {date}"
        },
        "wechat_work": {
            "webhook_url": "${WECHAT_WORK_WEBHOOK}",
            "mention_users": ["@all"]
        },
        "telegram": {
            "bot_token": "${TELEGRAM_BOT_TOKEN}",
            "chat_id": "${TELEGRAM_CHAT_ID}"
        }
    },
    "schedule": "0 12 * * *"  # Daily at noon
}
```

### 4. Content Extraction from News Pages

Extract full content including video transcripts:

```python
from hotsearchcrawler.content_extractor import ContentExtractor

extractor = ContentExtractor()

# Extract from article URL
content = extractor.extract(
    url="https://www.bilibili.com/video/BV13pSoBBEvX/",
    content_type="video"
)

print(content['title'])
print(content['text'])  # Extracted transcript or description
print(content['metadata'])  # Views, likes, publish time, etc.
```

### 5. Crawler Management

```python
from hotsearchcrawler.spider_manager import SpiderManager

manager = SpiderManager()

# List available spiders
spiders = manager.list_spiders()
print(spiders)  # ['weibo', 'douyin', 'bilibili', 'zhihu', ...]

# Run specific spider
manager.run_spider('weibo', settings={
    'DOWNLOAD_DELAY': 2,
    'CONCURRENT_REQUESTS': 8
})

# Get crawler statistics
stats = manager.get_stats('weibo')
print(f"Items scraped: {stats['item_scraped_count']}")
print(f"Error count: {stats['error_count']}")
```

## Common Patterns

### Pattern 1: Daily Hot Topic Report Generation

```python
from hotsearch_analysis_agent.report_generator import ReportGenerator
from datetime import datetime

generator = ReportGenerator()

# Generate comprehensive daily report
report = generator.generate_daily_report(
    date=datetime.now(),
    topic_filter="人工智能 OR 前沿科技",
    platforms=["weibo", "zhihu", "bilibili", "toutiao"],
    include_sentiment=True,
    include_clustering=True,
    language="zh-CN"
)

# Save to file
with open(f"report_{datetime.now().strftime('%Y%m%d')}.md", "w") as f:
    f.write(report.to_markdown())

# Or push directly
generator.push_report(report, channels=["email", "wechat_work"])
```

### Pattern 2: Real-time Event Monitoring

```python
from hotsearch_analysis_agent.event_monitor import EventMonitor

monitor = EventMonitor()

# Define monitoring rules
monitor.add_rule(
    name="突发事件监控",
    conditions={
        "keywords": ["突发", "紧急", "重大"],
        "rapid_rise": True,  # Detect rapid ranking increases
        "multi_platform": 3,  # Appear on 3+ platforms
        "time_window": 3600  # Within 1 hour
    },
    actions={
        "alert": ["email", "telegram"],
        "analyze": True,
        "archive": True
    }
)

# Start monitoring (blocking)
monitor.start()
```

### Pattern 3: Sentiment Trend Analysis

```python
from hotsearch_analysis_agent.sentiment_analyzer import SentimentAnalyzer
import pandas as pd

analyzer = SentimentAnalyzer()

# Analyze sentiment for specific topic over time
topic = "ChatGPT"
results = analyzer.analyze_trend(
    topic=topic,
    start_date="2025-01-01",
    end_date="2025-01-31",
    platforms=["weibo", "zhihu"],
    granularity="daily"
)

# Convert to DataFrame for visualization
df = pd.DataFrame(results)
df.plot(x='date', y=['positive_ratio', 'negative_ratio', 'neutral_ratio'])
```

### Pattern 4: Multi-Platform Data Aggregation

```python
from hotsearch_analysis_agent.aggregator import HotSearchAggregator

aggregator = HotSearchAggregator()

# Get unified hot search ranking across platforms
unified_ranking = aggregator.get_unified_ranking(
    platforms=["weibo", "douyin", "bilibili", "baidu"],
    top_n=50,
    time_range="today",
    deduplication=True,  # Merge similar topics
    boost_multi_platform=True  # Boost topics appearing on multiple platforms
)

for rank, item in enumerate(unified_ranking, 1):
    print(f"{rank}. {item['title']}")
    print(f"   Platforms: {', '.join(item['platforms'])}")
    print(f"   Heat Score: {item['heat_score']}")
    print(f"   Sentiment: {item['sentiment']}")
```

## API Reference

### Core Classes

**QueryEngine** - Natural language query interface:
```python
engine = QueryEngine(model_name="gpt-4", temperature=0.7)
result = engine.query(question: str, context_limit: int = 10)
# Returns: {'answer': str, 'sources': List[Dict], 'confidence': float}
```

**TopicClusterer** - Topic clustering and analysis:
```python
clusterer = TopicClusterer(algorithm="kmeans", n_clusters="auto")
clusters = clusterer.analyze_daily_topics(min_cluster_size: int = 3)
# Returns: List[Cluster] with themes, topics, sentiment
```

**PushManager** - Multi-channel notification management:
```python
manager = PushManager()
task = manager.create_task(name: str, keywords: List[str], **config)
manager.execute_task(task_id: int)
manager.test_push(task_id: int, channel: str)
```

**ContentExtractor** - Deep content extraction:
```python
extractor = ContentExtractor(driver_path="/usr/local/bin/chromedriver")
content = extractor.extract(url: str, content_type: str = "auto")
# Returns: {'title': str, 'text': str, 'metadata': Dict}
```

## Troubleshooting

### Issue: ChromeDriver Version Mismatch

```bash
# Error: "This version of ChromeDriver only supports Chrome version XX"
# Solution: Download matching version
google-chrome --version  # Check Chrome version
# Download from https://chromedriver.chromium.org/downloads
```

### Issue: MySQL Connection Errors

```python
# Error: "Can't connect to MySQL server"
# Check MySQL service status
sudo systemctl status mysql

# Verify credentials
mysql -u your_user -p -h localhost

# Grant permissions if needed
GRANT ALL PRIVILEGES ON public_opinion.* TO 'your_user'@'localhost';
FLUSH PRIVILEGES;
```

### Issue: LLM API Rate Limits

```python
# Add retry logic and rate limiting
from hotsearch_analysis_agent.llm_client import LLMClient

client = LLMClient(
    api_key="${OPENAI_API_KEY}",
    max_retries=5,
    retry_delay=2,
    requests_per_minute=20
)
```

### Issue: Crawler Getting Blocked

```python
# Increase delays and use rotating user agents
# In hotsearchcrawler/settings.py
DOWNLOAD_DELAY = 3
RANDOMIZE_DOWNLOAD_DELAY = True
USER_AGENT_LIST = [...]  # Add multiple user agents

# Use middleware for cookie refresh
DOWNLOADER_MIDDLEWARES = {
    'hotsearchcrawler.middlewares.CookieRefreshMiddleware': 400,
}
```

### Issue: Out of Memory During Analysis

```python
# Process in batches
from hotsearch_analysis_agent.batch_processor import BatchProcessor

processor = BatchProcessor(batch_size=100)
results = processor.process_topics(
    topic_ids=large_topic_list,
    on_batch_complete=save_intermediate_results
)
```

### Issue: Missing Dependencies for Video Extraction

```bash
# Install FFmpeg for video processing
sudo apt-get install ffmpeg  # Ubuntu/Debian
brew install ffmpeg          # macOS

# Verify installation
ffmpeg -version
```

## Advanced Configuration

### Using Local Huawei Pangu Model

```python
# Download model from https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

# Configure in .env
USE_LOCAL_MODEL=true
PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model
PANGU_DEVICE=cuda  # or cpu

# Initialize in code
from hotsearch_analysis_agent.llm_client import PanguClient

client = PanguClient(
    model_path="${PANGU_MODEL_PATH}",
    device="${PANGU_DEVICE}",
    max_length=2048
)
```

### Custom Spider Development

```python
# Create new spider in hotsearchcrawler/spiders/
import scrapy
from hotsearchcrawler.items import HotSearchItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://platform.com/trending']
    
    def parse(self, response):
        for item in response.css('.trending-item'):
            yield HotSearchItem(
                title=item.css('.title::text').get(),
                rank=item.css('.rank::text').get(),
                heat_score=item.css('.heat::text').get(),
                url=item.css('a::attr(href)').get(),
                platform='custom_platform',
                category='综合'
            )
```

### Scheduled Crawler Execution

```bash
# Use crontab for automated crawling
crontab -e

# Run crawlers every hour
0 * * * * cd /path/to/project && /path/to/venv/bin/python run_spiders.py

# Generate morning report at 8 AM
0 8 * * * cd /path/to/project && /path/to/venv/bin/python -c "from hotsearch_analysis_agent.report_generator import ReportGenerator; ReportGenerator().generate_and_push_daily_report()"
```

This skill enables AI coding agents to help developers deploy, configure, and extend this comprehensive public opinion analytics system with LLM-powered insights and multi-channel alerting capabilities.
