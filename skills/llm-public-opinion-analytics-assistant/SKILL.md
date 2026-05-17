---
name: llm-public-opinion-analytics-assistant
description: Multi-platform hot topic crawler and LLM-powered sentiment analysis system with push notifications
triggers:
  - "analyze social media sentiment and trending topics"
  - "set up a public opinion monitoring system"
  - "crawl hot search rankings from multiple platforms"
  - "build a sentiment analysis assistant with LLM"
  - "create automated topic clustering and alerts"
  - "monitor trending news across Chinese social platforms"
  - "implement multi-channel hot topic notifications"
  - "analyze public opinion with conversational interface"
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive public opinion monitoring system that aggregates 26 trending lists from 15 major Chinese platforms (Weibo, Bilibili, Toutiao, etc.), performs LLM-powered sentiment analysis, topic clustering, and delivers insights via multiple channels (Email, WeChat, Enterprise WeChat, Telegram).

## What This Project Does

- **Multi-Platform Crawling**: Scrapes real-time trending topics from 15 platforms including Weibo, Bilibili, Zhihu, Toutiao, Douyin, and more
- **LLM Analysis**: Uses large language models (supports Pangu, OpenAI-compatible APIs) for sentiment analysis, topic clustering, and trend detection
- **Conversational Interface**: Natural language queries for hot topics, theme searches, and analysis results
- **Multi-Channel Alerts**: Push notifications via Email (SMTP), Enterprise WeChat (Robot/App), Telegram, and personal WeChat
- **Detail Extraction**: Deep content analysis including video transcripts and news article content
- **Hotkey Controls**: Keyboard shortcuts to start/stop crawlers and manage tasks

## Installation

### Prerequisites

1. **Browser Driver Setup** (Required for detail page scraping):

   ```bash
   # Download ChromeDriver or EdgeDriver matching your browser version
   # Chrome: https://chromedriver.chromium.org/
   # Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/
   
   # Place driver in system PATH or project directory
   # Verify installation:
   chromedriver --version
   # or
   msedgedriver --version
   ```

2. **MySQL Database**:

   ```bash
   # Install MySQL 5.7+ or MariaDB
   # Create database
   mysql -u root -p -e "CREATE DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
   ```

3. **Python Environment**:

   ```bash
   # Create virtual environment
   python -m venv venv
   source venv/bin/activate  # On Windows: venv\Scripts\activate
   
   # Install dependencies
   pip install -r requirements.txt
   ```

### Initial Setup

1. **Initialize Database Tables**:

   ```python
   # Reference init.py for table schemas
   # Key tables: hot_searches, analysis_results, push_tasks, platform_configs
   
   # Example minimal schema creation
   import pymysql
   
   conn = pymysql.connect(
       host=os.getenv('MYSQL_HOST', 'localhost'),
       user=os.getenv('MYSQL_USER'),
       password=os.getenv('MYSQL_PASSWORD'),
       database=os.getenv('MYSQL_DATABASE'),
       charset='utf8mb4'
   )
   
   # Run SQL from init.py to create tables
   ```

2. **Configure Environment Variables** (`.env` file):

   ```env
   # MySQL Configuration
   MYSQL_HOST=localhost
   MYSQL_PORT=3306
   MYSQL_USER=your_user
   MYSQL_PASSWORD=your_password
   MYSQL_DATABASE=hotsearch
   
   # LLM Configuration (OpenAI-compatible API)
   OPENAI_API_KEY=your_api_key
   OPENAI_API_BASE=https://api.openai.com/v1
   OPENAI_MODEL=gpt-4
   
   # Or Pangu Model (local deployment)
   PANGU_API_BASE=http://localhost:8000
   PANGU_MODEL=openpangu-embedded-7b
   
   # Push Notification Channels
   SMTP_HOST=smtp.gmail.com
   SMTP_PORT=587
   SMTP_USER=your_email@gmail.com
   SMTP_PASSWORD=your_app_password
   
   WECHAT_CORP_ID=your_corp_id
   WECHAT_CORP_SECRET=your_corp_secret
   WECHAT_AGENT_ID=your_agent_id
   
   TELEGRAM_BOT_TOKEN=your_bot_token
   TELEGRAM_CHAT_ID=your_chat_id
   ```

3. **Configure Crawler Settings** (`hotsearchcrawler/settings.py`):

   ```python
   # MySQL connection for crawler
   MYSQL_CONFIG = {
       'host': os.getenv('MYSQL_HOST', 'localhost'),
       'port': int(os.getenv('MYSQL_PORT', 3306)),
       'user': os.getenv('MYSQL_USER'),
       'password': os.getenv('MYSQL_PASSWORD'),
       'database': os.getenv('MYSQL_DATABASE'),
       'charset': 'utf8mb4'
   }
   
   # Optional: Platform-specific cookies
   COOKIES = {
       'weibo': 'your_weibo_cookie',  # Optional for login-protected content
       'bilibili': 'your_bilibili_cookie'
   }
   
   # Crawler settings
   CONCURRENT_REQUESTS = 16
   DOWNLOAD_DELAY = 1
   ROBOTSTXT_OBEY = False
   ```

## Running the Application

### Start Web Interface

```bash
# Start main application
python app.py

# Access web interface at http://localhost:5000
```

### Manual Crawler Control

```bash
# Test single crawler
python runspider-test.py

# Start all crawlers (normally triggered from web UI)
python run_spiders.py
```

### Test Push Notifications

```bash
# Test configured notification channels
python test_push_task.py
```

## Key API & Usage Patterns

### 1. Conversational Query Interface

```python
from hotsearch_analysis_agent.agent import AnalysisAgent

# Initialize agent with LLM config
agent = AnalysisAgent(
    model_name=os.getenv('OPENAI_MODEL', 'gpt-4'),
    api_key=os.getenv('OPENAI_API_KEY'),
    api_base=os.getenv('OPENAI_API_BASE')
)

# Query hot topics
response = agent.query("今天微博热搜有哪些关于AI的话题?")
# Returns structured response with trending AI topics from Weibo

# Theme-based search
response = agent.query("最近关于芯片的舆情分析")
# Returns sentiment analysis and clustering for chip-related topics

# Sentiment analysis
response = agent.query("分析华为昇腾的舆论倾向")
# Returns sentiment breakdown (positive/negative/neutral) with evidence
```

### 2. Crawler Management

```python
from hotsearchcrawler.crawler_manager import CrawlerManager

# Initialize crawler manager
manager = CrawlerManager()

# Start specific platform crawler
manager.start_crawler('weibo')  # Options: weibo, bilibili, zhihu, toutiao, etc.

# Start all crawlers
manager.start_all()

# Stop crawlers
manager.stop_all()

# Check status
status = manager.get_status()
# Returns: {'weibo': 'running', 'bilibili': 'stopped', ...}
```

### 3. Topic Clustering & Analysis

```python
from hotsearch_analysis_agent.clustering import TopicClusterer

clusterer = TopicClusterer(
    model_name=os.getenv('OPENAI_MODEL'),
    api_key=os.getenv('OPENAI_API_KEY')
)

# Fetch recent hot topics
topics = clusterer.fetch_topics(
    platforms=['weibo', 'bilibili', 'zhihu'],
    hours=24  # Last 24 hours
)

# Perform clustering
clusters = clusterer.cluster_topics(
    topics=topics,
    num_clusters=5,
    theme='人工智能'  # Optional theme filter
)

# Generate summary report
report = clusterer.generate_report(clusters)
print(report)
```

### 4. Sentiment Analysis

```python
from hotsearch_analysis_agent.sentiment import SentimentAnalyzer

analyzer = SentimentAnalyzer(
    model_name=os.getenv('OPENAI_MODEL'),
    api_key=os.getenv('OPENAI_API_KEY')
)

# Analyze single topic
result = analyzer.analyze_topic(
    topic_id=12345,
    include_details=True  # Fetch detail page content
)

# Returns:
# {
#   'sentiment': 'positive',
#   'confidence': 0.87,
#   'key_points': ['技术突破', '市场认可'],
#   'evidence': [...],
#   'detail_content': '...'  # Extracted from news/video
# }

# Batch analysis
results = analyzer.analyze_batch(topic_ids=[123, 456, 789])
```

### 5. Push Notification Setup

```python
from hotsearch_analysis_agent.push import PushManager

push_manager = PushManager()

# Create email push task
push_manager.create_task(
    task_name="AI热点日报",
    channel="email",
    recipients=["admin@example.com"],
    query="人工智能 AND 芯片",
    schedule="0 9 * * *",  # Daily at 9 AM (cron format)
    config={
        'smtp_host': os.getenv('SMTP_HOST'),
        'smtp_port': int(os.getenv('SMTP_PORT')),
        'smtp_user': os.getenv('SMTP_USER'),
        'smtp_password': os.getenv('SMTP_PASSWORD')
    }
)

# Create Enterprise WeChat push
push_manager.create_task(
    task_name="舆情警报",
    channel="wechat_corp",
    recipients=["@all"],  # Send to all members
    query="负面 AND (公司名 OR 产品名)",
    threshold={'sentiment': 'negative', 'count': 5},
    config={
        'corp_id': os.getenv('WECHAT_CORP_ID'),
        'corp_secret': os.getenv('WECHAT_CORP_SECRET'),
        'agent_id': os.getenv('WECHAT_AGENT_ID')
    }
)

# Create Telegram push
push_manager.create_task(
    task_name="实时热搜",
    channel="telegram",
    recipients=[os.getenv('TELEGRAM_CHAT_ID')],
    query="热搜前10",
    schedule="*/30 * * * *",  # Every 30 minutes
    config={
        'bot_token': os.getenv('TELEGRAM_BOT_TOKEN')
    }
)

# Execute task manually
push_manager.execute_task(task_id=1)
```

### 6. Detail Page Content Extraction

```python
from hotsearchcrawler.detail_extractor import DetailExtractor

extractor = DetailExtractor()

# Extract content from news article
content = extractor.extract(
    url="https://news.example.com/article/123",
    content_type="article"
)

# Extract transcript from video (e.g., Bilibili)
content = extractor.extract(
    url="https://www.bilibili.com/video/BV13pSoBBEvX",
    content_type="video"
)

# Returns:
# {
#   'title': '...',
#   'content': '...',  # Article text or video transcript
#   'author': '...',
#   'publish_time': '2026-04-07 12:30:00',
#   'images': [...],
#   'metadata': {...}
# }
```

## Common Workflow Patterns

### Pattern 1: Daily Hot Topic Digest

```python
from hotsearch_analysis_agent import AnalysisAgent, PushManager
from datetime import datetime, timedelta

# Fetch yesterday's top topics
agent = AnalysisAgent()
topics = agent.query(
    f"过去24小时内各平台热搜前20的话题",
    time_range=(datetime.now() - timedelta(days=1), datetime.now())
)

# Cluster by theme
clusters = agent.cluster_topics(topics['data'], num_clusters=5)

# Generate report
report = agent.generate_report(
    clusters=clusters,
    format='html',
    include_sentiment=True
)

# Push via email
push_manager = PushManager()
push_manager.send_email(
    recipients=['team@company.com'],
    subject=f"舆情日报 - {datetime.now().strftime('%Y-%m-%d')}",
    body=report
)
```

### Pattern 2: Real-Time Sentiment Monitoring

```python
from hotsearch_analysis_agent import SentimentAnalyzer, PushManager
import time

analyzer = SentimentAnalyzer()
push_manager = PushManager()

# Monitor specific keywords
keywords = ['品牌名', '产品名']

while True:
    # Fetch recent mentions
    topics = analyzer.fetch_topics_by_keywords(
        keywords=keywords,
        hours=1  # Last hour
    )
    
    # Analyze sentiment
    for topic in topics:
        result = analyzer.analyze_topic(topic['id'], include_details=True)
        
        # Alert on negative sentiment
        if result['sentiment'] == 'negative' and result['confidence'] > 0.8:
            push_manager.send_alert(
                channel='wechat_corp',
                message=f"⚠️ 负面舆情警报\n话题: {topic['title']}\n置信度: {result['confidence']}\n详情: {result['detail_url']}",
                urgency='high'
            )
    
    time.sleep(300)  # Check every 5 minutes
```

### Pattern 3: Competitive Intelligence Tracking

```python
from hotsearch_analysis_agent import AnalysisAgent

agent = AnalysisAgent()

# Define competitors
competitors = ['竞品A', '竞品B', '竞品C']

# Fetch competitive mentions
intelligence = {}
for competitor in competitors:
    result = agent.query(
        f"最近7天关于{competitor}的热搜和讨论",
        time_range=7,
        platforms=['weibo', 'zhihu', 'toutiao']
    )
    
    intelligence[competitor] = {
        'mention_count': result['total_count'],
        'sentiment': result['sentiment_summary'],
        'top_topics': result['data'][:5],
        'trend': result['trend']  # Increasing/decreasing
    }

# Generate comparative report
report = agent.generate_competitive_report(intelligence)
```

## Configuration Tips

### LLM Model Selection

```python
# Using OpenAI-compatible API
OPENAI_API_BASE=https://api.openai.com/v1
OPENAI_MODEL=gpt-4

# Using local Pangu model (recommended for Chinese text)
PANGU_API_BASE=http://localhost:8000
PANGU_MODEL=openpangu-embedded-7b

# Using other compatible providers
OPENAI_API_BASE=https://api.deepseek.com/v1
OPENAI_MODEL=deepseek-chat
```

### Crawler Rate Limiting

```python
# In hotsearchcrawler/settings.py
DOWNLOAD_DELAY = 2  # Seconds between requests
CONCURRENT_REQUESTS = 8  # Simultaneous requests
AUTOTHROTTLE_ENABLED = True
AUTOTHROTTLE_TARGET_CONCURRENCY = 1.0
```

### Platform Selection

```python
# Enable/disable specific platforms in config
ENABLED_PLATFORMS = [
    'weibo',      # Weibo hot search
    'bilibili',   # Bilibili trending
    'zhihu',      # Zhihu hot questions
    'toutiao',    # Toutiao hot news
    'douyin',     # Douyin trending
    # ... 10 more platforms
]
```

## Troubleshooting

### Issue: Browser driver not found

```bash
# Verify driver is in PATH
which chromedriver  # macOS/Linux
where chromedriver  # Windows

# If not found, add to PATH or specify in settings
CHROME_DRIVER_PATH=/usr/local/bin/chromedriver
```

### Issue: Crawler fails with 403/blocked

```python
# Add cookies for authenticated platforms
# In hotsearchcrawler/settings.py
COOKIES = {
    'weibo': 'SUB=your_cookie; ...',
}

# Or use rotating proxies
DOWNLOADER_MIDDLEWARES = {
    'scrapy.downloadermiddlewares.httpproxy.HttpProxyMiddleware': 110,
}
HTTP_PROXY = 'http://proxy.example.com:8080'
```

### Issue: LLM analysis timeout

```python
# Increase timeout in agent config
agent = AnalysisAgent(
    timeout=120,  # seconds
    max_retries=3
)

# Or use streaming for long responses
response = agent.query(query, stream=True)
for chunk in response:
    print(chunk, end='', flush=True)
```

### Issue: MySQL connection errors

```python
# Verify connection settings
import pymysql

try:
    conn = pymysql.connect(
        host=os.getenv('MYSQL_HOST'),
        port=int(os.getenv('MYSQL_PORT')),
        user=os.getenv('MYSQL_USER'),
        password=os.getenv('MYSQL_PASSWORD'),
        database=os.getenv('MYSQL_DATABASE')
    )
    print("Connection successful!")
except Exception as e:
    print(f"Error: {e}")
```

### Issue: Push notifications not sending

```python
# Test each channel individually
from hotsearch_analysis_agent.push import PushManager

manager = PushManager()

# Test email
manager.test_channel('email', {
    'smtp_host': os.getenv('SMTP_HOST'),
    'smtp_user': os.getenv('SMTP_USER'),
    'smtp_password': os.getenv('SMTP_PASSWORD')
})

# Test Telegram
manager.test_channel('telegram', {
    'bot_token': os.getenv('TELEGRAM_BOT_TOKEN'),
    'chat_id': os.getenv('TELEGRAM_CHAT_ID')
})
```

### Issue: Detail extraction fails for videos

```python
# Ensure video platform API access
# Some platforms require authentication or have rate limits

# For Bilibili videos, ensure valid cookies
BILIBILI_COOKIES = 'your_bilibili_cookies'

# Or use official API if available
BILIBILI_API_KEY = os.getenv('BILIBILI_API_KEY')
```

## Performance Optimization

```python
# Use database indexing
CREATE INDEX idx_platform_time ON hot_searches(platform, publish_time);
CREATE INDEX idx_keywords ON hot_searches(keywords(100));

# Enable result caching
from functools import lru_cache

@lru_cache(maxsize=100)
def get_topic_analysis(topic_id):
    # ... analysis logic
    pass

# Batch processing for large datasets
from hotsearch_analysis_agent import BatchProcessor

processor = BatchProcessor(batch_size=50)
results = processor.analyze_topics(topic_ids, parallel=True)
```
