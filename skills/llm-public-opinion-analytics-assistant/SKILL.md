---
name: llm-public-opinion-analytics-assistant
description: A comprehensive public opinion analytics assistant that aggregates 26 real-time hot search lists from 15 platforms with LLM analysis for sentiment, clustering, and multi-channel alerts
triggers:
  - analyze public opinion trends across multiple platforms
  - set up hot topic monitoring and push notifications
  - crawl hot search rankings from social media platforms
  - perform sentiment analysis on news and social topics
  - configure multi-channel alerts for trending topics
  - cluster and analyze related public opinion topics
  - extract content from video news using web scraping
  - build a real-time public opinion monitoring system
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

This project is an intelligent public opinion analytics assistant that combines real-time data from **26 hot search lists across 15 mainstream platforms** (Weibo, Bilibili, Douyin, Baidu, etc.) with large language model analysis capabilities. It provides:

- **Real-time data collection**: Multi-platform hot search crawlers with keyboard shortcut controls
- **Conversational queries**: Natural language interface for searching trends and topics
- **Advanced analytics**: Topic clustering, sentiment analysis, and theme-based searches
- **Content extraction**: Detailed news page scraping (including video content metadata)
- **Multi-channel alerts**: Push notifications via Email, WeChat Work, Enterprise WeChat, and Telegram

The system is split into two independent components:
1. **Analysis System**: `hotsearch_analysis_agent` - LLM-powered analytics and API
2. **Crawler Cluster**: `hotsearchcrawler` - Scrapy-based multi-platform data collection

## Installation

### Prerequisites

**1. Browser Driver Setup** (required for news detail page scraping):

```bash
# Verify your Chrome/Edge version first
google-chrome --version  # or microsoft-edge --version

# Download matching ChromeDriver from:
# https://chromedriver.chromium.org/
# or EdgeDriver from:
# https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Place driver in system PATH, e.g.:
# Linux/macOS:
sudo mv chromedriver /usr/local/bin/
sudo chmod +x /usr/local/bin/chromedriver

# Verify installation:
chromedriver --version
```

**2. MySQL Database**:

```bash
# Install MySQL and create database
mysql -u root -p
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

**3. Python Environment**:

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

### Database Initialization

```python
# Reference init.py for table creation
# Key tables:
# - hot_search_items: Stores crawled hot search data
# - analysis_results: Stores LLM analysis outputs
# - push_tasks: Manages scheduled alert tasks

# Run initialization (adapt from init.py):
python init.py
```

## Configuration

### Environment Variables (.env)

Create a `.env` file in the project root:

```env
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_username
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key_here
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Or for Huawei Pangu model (recommended):
PANGU_API_KEY=your_pangu_api_key
PANGU_API_ENDPOINT=your_pangu_endpoint

# Push Notification Channels (optional)
# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password

# WeChat Work Bot
WEWORK_BOT_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxx

# WeChat Work Application
WEWORK_CORP_ID=your_corp_id
WEWORK_AGENT_ID=your_agent_id
WEWORK_SECRET=your_secret

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

### Crawler Configuration (hotsearchcrawler/settings.py)

```python
# MySQL Pipeline Settings
MYSQL_CONFIG = {
    'host': os.getenv('MYSQL_HOST'),
    'port': int(os.getenv('MYSQL_PORT')),
    'user': os.getenv('MYSQL_USER'),
    'password': os.getenv('MYSQL_PASSWORD'),
    'database': os.getenv('MYSQL_DATABASE'),
}

# Optional: Platform-specific cookies for authenticated access
COOKIES = {
    'weibo': 'your_weibo_cookies',
    'bilibili': 'your_bilibili_cookies',
    # Add as needed
}

# Browser driver path (if not in system PATH)
CHROME_DRIVER_PATH = '/path/to/chromedriver'
EDGE_DRIVER_PATH = '/path/to/msedgedriver'
```

## Key Components and Usage

### 1. Starting the Analysis System

```bash
# Main application entry point
python app.py

# The web interface will be available at:
# http://localhost:5000 (or configured port)
```

### 2. Running Crawlers

**Via Web Interface** (recommended):
- Navigate to crawler control panel
- Use keyboard shortcuts to start/stop crawlers

**Via Command Line**:

```bash
# Test individual spider
python runspider-test.py

# Run all spiders
python run_spiders.py
```

**Programmatic Control**:

```python
from hotsearchcrawler.runner import CrawlerRunner

# Initialize crawler manager
runner = CrawlerRunner()

# Start specific platform crawlers
runner.start_crawler('weibo_hot')
runner.start_crawler('bilibili_hot')

# Start all crawlers
runner.start_all()

# Stop crawlers
runner.stop_all()

# Check crawler status
status = runner.get_status()
print(f"Active crawlers: {status['active']}")
```

### 3. Querying Hot Search Data

```python
from hotsearch_analysis_agent.data_service import HotSearchService

service = HotSearchService()

# Get latest hot searches from all platforms
hot_searches = service.get_latest_hot_searches(limit=50)

# Query specific platform
weibo_trends = service.get_platform_hot_searches('weibo', hours=24)

# Search by keyword
ai_topics = service.search_hot_searches(keyword='人工智能')

# Get trending topics in time range
from datetime import datetime, timedelta
yesterday = datetime.now() - timedelta(days=1)
recent_trends = service.get_hot_searches_by_time(
    start_time=yesterday,
    platform='bilibili'
)

for item in recent_trends:
    print(f"{item['title']} - 热度: {item['heat_score']}")
```

### 4. LLM-Powered Analysis

**Sentiment Analysis**:

```python
from hotsearch_analysis_agent.analyzer import SentimentAnalyzer

analyzer = SentimentAnalyzer()

# Analyze sentiment for a topic
topic_id = 12345
result = analyzer.analyze_sentiment(topic_id)

print(f"情感倾向: {result['sentiment']}")  # positive/negative/neutral
print(f"置信度: {result['confidence']}")
print(f"关键词: {result['keywords']}")
```

**Topic Clustering**:

```python
from hotsearch_analysis_agent.analyzer import TopicClusterer

clusterer = TopicClusterer()

# Cluster related topics from last 24 hours
clusters = clusterer.cluster_recent_topics(hours=24, num_clusters=5)

for i, cluster in enumerate(clusters):
    print(f"\n聚类 {i+1}: {cluster['theme']}")
    print(f"相关话题数: {len(cluster['topics'])}")
    for topic in cluster['topics'][:3]:
        print(f"  - {topic['title']}")
```

**Theme-Based Search and Analysis**:

```python
from hotsearch_analysis_agent.analyzer import ThemeAnalyzer

analyzer = ThemeAnalyzer()

# Search and analyze specific theme
theme = "人工智能与前沿科技"
report = analyzer.generate_theme_report(
    theme=theme,
    days=7,
    include_video_content=True
)

print(report['summary'])
print(f"核心发现: {len(report['key_findings'])}")
print(f"相关新闻: {len(report['related_news'])}")
```

### 5. Setting Up Push Tasks

**Test Push Channels**:

```bash
# Test all configured push channels
python test_push_task.py
```

**Create Scheduled Push Task**:

```python
from hotsearch_analysis_agent.push_service import PushTaskManager

manager = PushTaskManager()

# Create daily AI trend report
task = manager.create_push_task(
    name="AI热点日报",
    theme="人工智能",
    schedule_type="daily",  # daily/hourly/weekly
    schedule_time="09:00",
    channels=["wework_app", "email"],  # wework_bot, wework_app, telegram, email
    min_heat_threshold=5000,  # Minimum heat score to trigger
    enable_clustering=True,
    enable_sentiment=True
)

# List all tasks
tasks = manager.list_tasks()

# Update task
manager.update_task(task_id=task['id'], channels=["telegram"])

# Delete task
manager.delete_task(task_id=task['id'])
```

**Manual Push**:

```python
from hotsearch_analysis_agent.push_service import PushService

pusher = PushService()

# Generate and push report immediately
pusher.push_theme_report(
    theme="科技热点",
    channels=["email"],
    recipient_override="analyst@company.com"
)
```

### 6. Web API Usage

The analysis system exposes a RESTful API:

**Query Hot Searches**:

```bash
# Get latest trends
curl http://localhost:5000/api/hot_searches?limit=20&platform=weibo

# Search by keyword
curl http://localhost:5000/api/search?q=人工智能&days=3
```

**Request Analysis**:

```bash
# Analyze specific topic
curl -X POST http://localhost:5000/api/analyze/sentiment \
  -H "Content-Type: application/json" \
  -d '{"topic_id": 12345}'

# Cluster topics
curl -X POST http://localhost:5000/api/analyze/cluster \
  -H "Content-Type: application/json" \
  -d '{"hours": 24, "num_clusters": 5}'
```

**Manage Push Tasks**:

```bash
# Create push task
curl -X POST http://localhost:5000/api/push_tasks \
  -H "Content-Type: application/json" \
  -d '{
    "name": "每日科技简报",
    "theme": "科技",
    "schedule_type": "daily",
    "schedule_time": "08:00",
    "channels": ["email"]
  }'

# List tasks
curl http://localhost:5000/api/push_tasks

# Delete task
curl -X DELETE http://localhost:5000/api/push_tasks/123
```

## Common Patterns

### Pattern 1: Real-Time Monitoring Dashboard

```python
import time
from hotsearch_analysis_agent.data_service import HotSearchService
from hotsearch_analysis_agent.analyzer import SentimentAnalyzer

service = HotSearchService()
sentiment = SentimentAnalyzer()

def monitor_trending_topics():
    """Monitor and analyze trending topics in real-time"""
    while True:
        # Get top 10 trending topics
        trends = service.get_latest_hot_searches(limit=10)
        
        for topic in trends:
            # Quick sentiment check
            analysis = sentiment.analyze_sentiment(topic['id'])
            
            # Alert on negative sentiment with high heat
            if analysis['sentiment'] == 'negative' and topic['heat_score'] > 10000:
                print(f"⚠️  负面热点预警: {topic['title']}")
                print(f"   热度: {topic['heat_score']}, 情感: {analysis['confidence']}")
        
        time.sleep(300)  # Check every 5 minutes

monitor_trending_topics()
```

### Pattern 2: Cross-Platform Topic Tracking

```python
from hotsearch_analysis_agent.data_service import HotSearchService

service = HotSearchService()

def track_topic_across_platforms(keyword):
    """Track how a topic trends across different platforms"""
    platforms = ['weibo', 'bilibili', 'douyin', 'baidu', 'zhihu']
    
    results = {}
    for platform in platforms:
        items = service.search_hot_searches(
            keyword=keyword,
            platform=platform,
            hours=24
        )
        
        if items:
            results[platform] = {
                'count': len(items),
                'max_heat': max(item['heat_score'] for item in items),
                'titles': [item['title'] for item in items[:3]]
            }
    
    return results

# Example usage
topic_spread = track_topic_across_platforms('GPT-6')
for platform, data in topic_spread.items():
    print(f"{platform}: {data['count']}条, 最高热度{data['max_heat']}")
```

### Pattern 3: Automated Weekly Report

```python
from hotsearch_analysis_agent.analyzer import ThemeAnalyzer
from hotsearch_analysis_agent.push_service import PushService
from datetime import datetime

def generate_weekly_report():
    """Generate comprehensive weekly AI trend report"""
    analyzer = ThemeAnalyzer()
    pusher = PushService()
    
    # Generate report
    report = analyzer.generate_theme_report(
        theme="人工智能与前沿科技",
        days=7,
        include_video_content=True
    )
    
    # Format report
    formatted_report = f"""
    # 周报 - {datetime.now().strftime('%Y年%m月%d日')}
    
    ## 核心发现
    {chr(10).join(f"- {finding}" for finding in report['key_findings'])}
    
    ## Top 10 热点新闻
    {chr(10).join(f"{i+1}. {news['title']}" for i, news in enumerate(report['related_news'][:10]))}
    
    ## 情感分析
    - 正面: {report['sentiment_distribution']['positive']}%
    - 中性: {report['sentiment_distribution']['neutral']}%
    - 负面: {report['sentiment_distribution']['negative']}%
    """
    
    # Push to multiple channels
    pusher.push_custom_report(
        content=formatted_report,
        channels=["email", "wework_app"],
        title="AI周报"
    )

# Schedule this function to run weekly
generate_weekly_report()
```

## Troubleshooting

### Crawler Issues

**Problem**: Crawler fails to start or returns no data

```python
# Check browser driver
import subprocess
result = subprocess.run(['chromedriver', '--version'], capture_output=True)
print(result.stdout.decode())

# Verify driver is in PATH
import shutil
driver_path = shutil.which('chromedriver')
print(f"Driver location: {driver_path}")

# Test with explicit driver path
from hotsearchcrawler.settings import CHROME_DRIVER_PATH
print(f"Configured path: {CHROME_DRIVER_PATH}")
```

**Problem**: Platform-specific crawler blocked

```bash
# Some platforms require cookies for access
# Export cookies from browser using extension like "EditThisCookie"
# Add to hotsearchcrawler/settings.py:
COOKIES = {
    'weibo': 'SUB=xxx; SUBP=yyy; ...',
}
```

### Database Connection Issues

```python
# Test MySQL connection
import mysql.connector
from dotenv import load_dotenv
import os

load_dotenv()

try:
    conn = mysql.connector.connect(
        host=os.getenv('MYSQL_HOST'),
        user=os.getenv('MYSQL_USER'),
        password=os.getenv('MYSQL_PASSWORD'),
        database=os.getenv('MYSQL_DATABASE')
    )
    print("✓ Database connection successful")
    conn.close()
except Exception as e:
    print(f"✗ Connection failed: {e}")
```

### LLM Analysis Errors

**Problem**: Model returns poor quality analysis

```python
# For Huawei Pangu model (recommended for Chinese content):
# Ensure you're using the correct model endpoint
# Download model from: https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

# Test LLM connection
from hotsearch_analysis_agent.llm_client import LLMClient

client = LLMClient()
response = client.test_connection()
print(response)
```

**Problem**: Rate limiting or timeout

```python
# Implement retry logic with exponential backoff
import time
from functools import wraps

def retry_on_failure(max_retries=3, delay=2):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_retries - 1:
                        raise
                    wait_time = delay * (2 ** attempt)
                    print(f"Retry {attempt + 1}/{max_retries} after {wait_time}s")
                    time.sleep(wait_time)
        return wrapper
    return decorator

@retry_on_failure(max_retries=3)
def analyze_with_retry(topic_id):
    return analyzer.analyze_sentiment(topic_id)
```

### Push Notification Failures

```bash
# Test individual channels
python test_push_task.py --channel email
python test_push_task.py --channel wework_bot
python test_push_task.py --channel telegram

# Check credentials
python -c "from dotenv import load_dotenv; import os; load_dotenv(); print('SMTP_USER:', os.getenv('SMTP_USER'))"
```

## Performance Optimization

**Database Indexing**:

```sql
-- Add indexes for common queries
CREATE INDEX idx_platform_time ON hot_search_items(platform, crawl_time);
CREATE INDEX idx_heat_score ON hot_search_items(heat_score DESC);
CREATE INDEX idx_title_fulltext ON hot_search_items(title) USING FULLTEXT;
```

**Crawler Rate Limiting**:

```python
# In hotsearchcrawler/settings.py
DOWNLOAD_DELAY = 2  # Seconds between requests
CONCURRENT_REQUESTS_PER_DOMAIN = 1
AUTOTHROTTLE_ENABLED = True
```

**Caching Analysis Results**:

```python
from functools import lru_cache
from datetime import datetime, timedelta

@lru_cache(maxsize=128)
def get_cached_sentiment(topic_id, cache_hours=24):
    """Cache sentiment analysis for 24 hours"""
    return analyzer.analyze_sentiment(topic_id)
```

This skill provides comprehensive coverage of the LLM-Based Public Opinion Analytics Assistant, enabling AI agents to help developers deploy, configure, and utilize this powerful multi-platform monitoring and analysis system.
