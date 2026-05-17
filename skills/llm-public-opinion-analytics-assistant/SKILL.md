---
name: llm-public-opinion-analytics-assistant
description: Use this LLM-based multi-platform public opinion analytics assistant to collect, analyze, and push hot topic data from 15 platforms with 26 ranking lists
triggers:
  - set up public opinion monitoring system
  - analyze hot topics from weibo bilibili douyin
  - configure sentiment analysis for news feeds
  - create hot topic push notifications
  - scrape trending topics from multiple platforms
  - build opinion analytics dashboard
  - monitor social media sentiment analysis
  - deploy multi-platform trend crawler
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is an intelligent public opinion analytics assistant that combines real-time data from **15 mainstream platforms** (26 ranking lists) with large language model analysis capabilities. It enables conversational hot search queries, topic clustering, sentiment analysis, and multi-channel alert push notifications (Email, WeChat, Enterprise WeChat, Telegram).

## What It Does

- **Multi-Platform Data Collection**: Crawls trending topics from 15 platforms including Weibo, Bilibili, Douyin, Baidu, etc.
- **Conversational Query Interface**: Natural language dialog for hot search queries and topic analysis
- **AI-Powered Analysis**: Topic clustering, sentiment analysis, and trend detection using LLMs (supports Huawei Pangu model)
- **Content Deep Dive**: Extracts detailed content from news pages (including video transcripts)
- **Multi-Channel Push**: Configurable hot topic alerts via Email, WeChat, Enterprise WeChat, Telegram
- **Crawler Control**: Hotkey-based crawler start/stop management
- **Direct Jump Navigation**: Quick access to original platform pages

## Installation

### Prerequisites

1. **Browser Driver Setup** (required for content extraction):

```bash
# Chrome/Edge driver must match your browser version
# Download from:
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Verify installation
chromedriver --version  # or msedgedriver --version
```

2. **MySQL Database**:

```bash
# Create database
mysql -u root -p
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

3. **Python Environment**:

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Project Structure

```
.
├── app.py                          # Main application entry
├── hotsearch_analysis_agent/       # Analysis system
│   ├── config/                     # Configuration files
│   ├── models/                     # Database models
│   ├── agents/                     # AI analysis agents
│   └── push_service/               # Push notification services
├── hotsearchcrawler/               # Crawler cluster (separate)
│   ├── spiders/                    # Platform-specific crawlers
│   └── settings.py                 # Crawler configuration
├── run_spiders.py                  # Crawler launcher
├── test_push_task.py               # Push notification tester
└── init.py                         # Database initialization reference
```

## Configuration

### 1. Environment Variables (`.env`)

```bash
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_db_user
MYSQL_PASSWORD=your_db_password
MYSQL_DATABASE=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_openai_api_key
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Or use Huawei Pangu (local deployment)
# PANGU_MODEL_PATH=/path/to/pangu/model
# USE_LOCAL_MODEL=true

# Push Service Configuration
# Email
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_email_password

# Enterprise WeChat
WECHAT_WEBHOOK_URL=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxx
WECHAT_APP_CORPID=your_corp_id
WECHAT_APP_SECRET=your_app_secret
WECHAT_APP_AGENTID=your_agent_id

# Telegram
TELEGRAM_BOT_TOKEN=your_telegram_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

### 2. Crawler Configuration (`hotsearchcrawler/settings.py`)

```python
# MySQL connection for crawler
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_db_user'
MYSQL_PASSWORD = 'your_db_password'
MYSQL_DATABASE = 'hotsearch_db'

# Optional: Platform-specific cookies
COOKIES = {
    'weibo': 'your_weibo_cookie',
    'bilibili': 'your_bilibili_cookie',
    # Add cookies for platforms requiring authentication
}

# Crawler settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
RANDOMIZE_DOWNLOAD_DELAY = True
```

### 3. Database Initialization

```python
# Reference init.py for database schema
from hotsearch_analysis_agent.models import Base, HotSearchItem, AnalysisReport
from sqlalchemy import create_engine

# Create engine
engine = create_engine(
    f"mysql+pymysql://{MYSQL_USER}:{MYSQL_PASSWORD}@{MYSQL_HOST}:{MYSQL_PORT}/{MYSQL_DATABASE}"
)

# Create all tables
Base.metadata.create_all(engine)
```

## Usage

### Starting the Application

```bash
# Launch main application (web interface)
python app.py

# Access web interface at http://localhost:5000
```

### Running Crawlers

```python
# Method 1: Through web interface (recommended)
# Use the web UI to start/stop crawlers with hotkeys

# Method 2: Direct command line
python run_spiders.py

# Method 3: Test specific spider
cd hotsearchcrawler
scrapy crawl weibo_hot  # Available: weibo_hot, bilibili_hot, douyin_hot, etc.
```

### Conversational Query Examples

```python
from hotsearch_analysis_agent.agents import QueryAgent

agent = QueryAgent()

# Query hot topics
response = agent.query("Show me today's top 10 trending topics on Weibo")

# Topic search
response = agent.query("Search for news about artificial intelligence")

# Sentiment analysis
response = agent.query("Analyze sentiment for topics related to electric vehicles")

# Clustering analysis
response = agent.query("Cluster similar topics from the past 24 hours")
```

### Programmatic API Usage

```python
from hotsearch_analysis_agent.services import HotSearchService
from hotsearch_analysis_agent.analysis import SentimentAnalyzer, TopicClusterer

# Initialize services
service = HotSearchService()

# Fetch hot topics from specific platform
topics = service.get_hot_topics(platform='weibo', limit=20)

# Sentiment analysis
analyzer = SentimentAnalyzer()
for topic in topics:
    sentiment = analyzer.analyze(topic.title, topic.content)
    print(f"{topic.title}: {sentiment.score} ({sentiment.label})")

# Topic clustering
clusterer = TopicClusterer()
clusters = clusterer.cluster_topics(topics, num_clusters=5)
for cluster_id, cluster_topics in clusters.items():
    print(f"Cluster {cluster_id}: {len(cluster_topics)} topics")
```

### Setting Up Push Notifications

```python
from hotsearch_analysis_agent.push_service import PushTaskManager, PushConfig

# Create push configuration
config = PushConfig(
    channels=['email', 'telegram'],
    keywords=['人工智能', 'AI', '机器学习'],
    sentiment_threshold=0.7,
    frequency='daily',  # or 'hourly', 'realtime'
    time='12:00'
)

# Initialize push manager
manager = PushTaskManager()

# Create push task
task_id = manager.create_task(
    name="AI News Daily Digest",
    config=config
)

# Test push
manager.test_push(task_id)

# Start scheduled push
manager.start_task(task_id)
```

### Testing Push Services

```bash
# Test all configured push channels
python test_push_task.py

# Test specific channel
python test_push_task.py --channel email
python test_push_task.py --channel telegram
```

## Common Patterns

### Pattern 1: Real-Time Monitoring Dashboard

```python
from hotsearch_analysis_agent import RealtimeMonitor
from datetime import datetime, timedelta

monitor = RealtimeMonitor()

# Monitor specific keywords
keywords = ['华为', '盘古大模型', 'AI']
alert_threshold = 100  # Alert when mentions > 100/hour

while True:
    recent_data = monitor.get_recent_topics(
        keywords=keywords,
        time_range=timedelta(hours=1)
    )
    
    if len(recent_data) > alert_threshold:
        monitor.send_alert(
            message=f"High activity detected: {len(recent_data)} mentions",
            data=recent_data
        )
    
    time.sleep(300)  # Check every 5 minutes
```

### Pattern 2: Batch Analysis Report Generation

```python
from hotsearch_analysis_agent import ReportGenerator
from datetime import datetime, timedelta

generator = ReportGenerator()

# Generate weekly report
report = generator.generate_report(
    start_date=datetime.now() - timedelta(days=7),
    end_date=datetime.now(),
    topics=['人工智能', '前沿科技'],
    include_sentiment=True,
    include_clustering=True,
    include_trend_analysis=True
)

# Export report
generator.export(report, format='markdown', output_path='./reports/')
generator.export(report, format='html', output_path='./reports/')
```

### Pattern 3: Custom Platform Crawler

```python
# Add new spider in hotsearchcrawler/spiders/
import scrapy
from hotsearchcrawler.items import HotSearchItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://example.com/hot']
    
    def parse(self, response):
        for item_selector in response.css('.hot-item'):
            item = HotSearchItem()
            item['title'] = item_selector.css('.title::text').get()
            item['url'] = item_selector.css('a::attr(href)').get()
            item['rank'] = item_selector.css('.rank::text').get()
            item['heat_score'] = item_selector.css('.heat::text').get()
            item['platform'] = 'custom_platform'
            item['crawl_time'] = datetime.now()
            
            # Fetch detail page
            yield scrapy.Request(
                item['url'],
                callback=self.parse_detail,
                meta={'item': item}
            )
    
    def parse_detail(self, response):
        item = response.meta['item']
        item['content'] = response.css('.article-content::text').getall()
        yield item
```

### Pattern 4: Sentiment Trend Tracking

```python
from hotsearch_analysis_agent.analysis import SentimentTracker
import matplotlib.pyplot as plt

tracker = SentimentTracker()

# Track sentiment over time
topic = "电动汽车"
sentiment_history = tracker.track_sentiment(
    topic=topic,
    days=30,
    granularity='daily'
)

# Visualize trend
plt.plot(sentiment_history['dates'], sentiment_history['scores'])
plt.title(f'Sentiment Trend: {topic}')
plt.xlabel('Date')
plt.ylabel('Sentiment Score')
plt.savefig(f'{topic}_sentiment_trend.png')
```

## Troubleshooting

### Issue: Crawler Fails to Start

```bash
# Check if browser driver is properly installed
which chromedriver  # or: where chromedriver (Windows)

# Verify driver version matches browser
chromedriver --version
google-chrome --version

# Check crawler logs
tail -f hotsearchcrawler/logs/crawler.log
```

### Issue: Database Connection Errors

```python
# Test MySQL connection
from sqlalchemy import create_engine
engine = create_engine(
    f"mysql+pymysql://{MYSQL_USER}:{MYSQL_PASSWORD}@{MYSQL_HOST}:{MYSQL_PORT}/{MYSQL_DATABASE}"
)
try:
    connection = engine.connect()
    print("Database connection successful")
    connection.close()
except Exception as e:
    print(f"Connection failed: {e}")
```

### Issue: LLM API Rate Limiting

```python
# Implement retry with exponential backoff
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=4, max=10)
)
def call_llm_api(prompt):
    # Your LLM API call here
    pass
```

### Issue: Push Notifications Not Sending

```bash
# Test push services individually
python test_push_task.py --channel email --verbose

# Check push service logs
tail -f hotsearch_analysis_agent/logs/push_service.log

# Verify credentials
python -c "from hotsearch_analysis_agent.push_service import test_credentials; test_credentials()"
```

### Issue: Memory Issues with Large Dataset

```python
# Use batch processing for large queries
from hotsearch_analysis_agent.services import HotSearchService

service = HotSearchService()

# Process in batches
batch_size = 1000
offset = 0

while True:
    batch = service.get_hot_topics(limit=batch_size, offset=offset)
    if not batch:
        break
    
    # Process batch
    process_batch(batch)
    offset += batch_size
```

### Issue: Chinese Text Encoding Problems

```python
# Ensure UTF-8 encoding throughout
import sys
sys.stdout.reconfigure(encoding='utf-8')

# In MySQL configuration
# CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci
```

## Advanced Configuration

### Using Huawei Pangu Model (Local Deployment)

```python
from hotsearch_analysis_agent.models import PanguModel

# Initialize Pangu model
model = PanguModel(
    model_path="/path/to/openpangu-embedded-7b-model",
    device="cuda:0"  # or "cpu"
)

# Use in analysis agent
from hotsearch_analysis_agent.agents import QueryAgent

agent = QueryAgent(llm=model)
response = agent.query("分析今日热点话题")
```

### Custom Crawler Scheduling

```python
# In run_spiders.py
from apscheduler.schedulers.background import BackgroundScheduler

scheduler = BackgroundScheduler()

# Schedule different platforms at different intervals
scheduler.add_job(run_weibo_spider, 'interval', minutes=15)
scheduler.add_job(run_bilibili_spider, 'interval', minutes=30)
scheduler.add_job(run_douyin_spider, 'interval', hours=1)

scheduler.start()
```

This skill provides comprehensive coverage of the LLM-Based Intelligent Public Opinion Analytics Assistant, enabling AI coding agents to help developers set up, configure, and effectively use this multi-platform public opinion monitoring and analysis system.
