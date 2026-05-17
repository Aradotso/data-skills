---
name: llm-public-opinion-analytics-assistant
description: Multi-platform hot search aggregation and LLM-powered sentiment analysis assistant with 15 platforms, 26 ranking lists, clustering, and multi-channel alerts
triggers:
  - how do I set up a public opinion monitoring system
  - analyze hot topics across Chinese social platforms
  - configure sentiment analysis with LLM for trending news
  - deploy hot search crawler with WeChat alerts
  - integrate Pangu model for opinion analytics
  - build real-time social media monitoring dashboard
  - scrape Weibo Douyin Bilibili trending topics
  - set up multi-channel hot topic notifications
---

# LLM Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive public opinion monitoring system that aggregates 26 ranking lists from 15 mainstream Chinese platforms (Weibo, Douyin, Bilibili, Zhihu, etc.) and provides LLM-powered analysis including sentiment analysis, topic clustering, and multi-channel alerting via WeChat, Telegram, and email.

## What This Project Does

- **Multi-Platform Data Collection**: Crawls hot search rankings from 15 platforms including Weibo, Douyin, Bilibili, Zhihu, Baidu, Toutiao, etc.
- **LLM-Powered Analysis**: Uses large language models (supports Huawei Pangu, OpenAI-compatible APIs) for sentiment analysis, topic clustering, and trend identification
- **Conversational Interface**: Natural language query interface for searching topics, platforms, and analyzing trends
- **Deep Content Extraction**: Fetches full article/video content even for video-based news
- **Multi-Channel Alerts**: Push notifications via Enterprise WeChat, Telegram, SMTP email
- **Real-Time Dashboard**: Web UI with hotkey-controlled crawler management and instant platform jumping

## Installation

### Prerequisites

**Browser Driver Setup (Required for Content Extraction)**

1. Install Chrome or Edge browser
2. Check browser version: Settings → About (e.g., Chrome 115.0.5790.102)
3. Download matching driver:
   - Chrome: https://chromedriver.chromium.org/
   - Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/
4. Place driver in system PATH or browser directory
5. Verify: `chromedriver --version`

**Python Environment**

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # Linux/Mac
# venv\Scripts\activate  # Windows

# Install dependencies
pip install -r requirements.txt
```

**Database Setup**

```bash
# Install MySQL
# Create database and tables using init.py as reference

mysql -u root -p
CREATE DATABASE opinion_analytics CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### Configuration

**1. Environment Variables (`.env`)**

```env
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=opinion_analytics

# LLM API (OpenAI-compatible format)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
LLM_MODEL=gpt-3.5-turbo

# Or use Huawei Pangu (local deployment recommended)
# PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model

# Notification Channels (optional)
WECHAT_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxx
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
```

**2. Crawler Settings (`hotsearchcrawler/settings.py`)**

```python
# MySQL connection for crawler
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'root'
MYSQL_PASSWORD = 'your_password'
MYSQL_DATABASE = 'opinion_analytics'

# Optional: Platform-specific cookies for authenticated access
COOKIES = {
    'weibo': 'your_weibo_cookie',
    'douyin': 'your_douyin_cookie'
}

# Crawler behavior
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
ROBOTSTXT_OBEY = False
```

**3. Database Initialization**

```python
# Reference init.py for table schemas
from hotsearch_analysis_agent.database import init_database

init_database()
```

## Key Components

### Starting the System

```bash
# Start web application (analysis + UI)
python app.py

# Test crawler independently
python runspider-test.py

# Test push notifications
python test_push_task.py
```

### Project Structure

```
├── app.py                          # Main application entry
├── hotsearch_analysis_agent/       # Analysis system
│   ├── agent/                      # LLM agent logic
│   ├── database/                   # DB models and queries
│   ├── api/                        # REST API endpoints
│   └── static/                     # Frontend assets
├── hotsearchcrawler/               # Spider cluster (decoupled)
│   ├── spiders/                    # Platform-specific crawlers
│   ├── settings.py                 # Scrapy settings
│   └── run_spiders.py              # Spider orchestration
└── init.py                         # Database schema reference
```

## Core Usage Patterns

### 1. Querying Hot Topics via API

```python
import requests

# Natural language query
response = requests.post('http://localhost:5000/api/query', json={
    'query': '最近关于人工智能的热搜有哪些',  # "Recent AI hot searches"
    'platforms': ['weibo', 'zhihu', 'bilibili']
})

results = response.json()
for item in results['hot_topics']:
    print(f"{item['title']} - {item['platform']} - {item['heat_score']}")
```

### 2. Sentiment Analysis

```python
from hotsearch_analysis_agent.agent import OpinionAgent

agent = OpinionAgent()

# Analyze sentiment for a topic
analysis = agent.analyze_sentiment(
    topic="芯片技术突破",
    content_list=[
        "国产芯片取得重大突破,太棒了!",
        "还有很长的路要走...",
        "这是历史性的一刻"
    ]
)

print(f"Overall sentiment: {analysis['sentiment']}")  # positive/neutral/negative
print(f"Confidence: {analysis['confidence']}")
print(f"Key emotions: {analysis['emotions']}")
```

### 3. Topic Clustering

```python
from hotsearch_analysis_agent.agent import TopicCluster

cluster = TopicCluster()

# Get all hot topics from last 24 hours
topics = cluster.fetch_recent_topics(hours=24)

# Cluster related topics
clusters = cluster.cluster_topics(topics, min_similarity=0.7)

for cluster_id, group in clusters.items():
    print(f"\nCluster {cluster_id}:")
    for topic in group:
        print(f"  - {topic['title']} ({topic['platform']})")
```

### 4. Setting Up Push Notifications

```python
from hotsearch_analysis_agent.push import PushTask

# Create monitoring task
task = PushTask(
    keywords=['人工智能', '芯片', '科技'],
    platforms=['weibo', 'zhihu', 'toutiao'],
    channels=['wechat', 'telegram', 'email'],
    threshold=50000,  # Minimum heat score
    interval_minutes=30
)

# Start background monitoring
task.start()

# Or manual trigger
report = task.generate_report()
task.send_notification(report, channels=['wechat'])
```

### 5. Running Specific Crawlers

```python
from scrapy.crawler import CrawlerProcess
from hotsearchcrawler.spiders.weibo_spider import WeiboSpider

# Run single spider
process = CrawlerProcess()
process.crawl(WeiboSpider)
process.start()
```

```bash
# Or via command line
cd hotsearchcrawler
scrapy crawl weibo
scrapy crawl douyin
scrapy crawl bilibili
```

### 6. Custom LLM Integration

```python
from hotsearch_analysis_agent.agent import BaseLLMProvider

class CustomLLMProvider(BaseLLMProvider):
    def __init__(self, model_path):
        self.model = self.load_model(model_path)
    
    def chat_completion(self, messages):
        prompt = self.format_messages(messages)
        response = self.model.generate(prompt)
        return {
            'content': response,
            'model': 'custom-model',
            'usage': {'tokens': len(response.split())}
        }

# Use custom provider
from hotsearch_analysis_agent.config import set_llm_provider
set_llm_provider(CustomLLMProvider('/path/to/model'))
```

### 7. Web UI Interaction

```javascript
// Frontend query example (static/js/app.js)
async function queryHotTopics(userInput) {
    const response = await fetch('/api/chat', {
        method: 'POST',
        headers: {'Content-Type': 'application/json'},
        body: JSON.stringify({
            message: userInput,
            session_id: getCurrentSessionId()
        })
    });
    
    const data = await response.json();
    displayResults(data.topics);
    displayAnalysis(data.analysis);
}

// Hotkey-controlled crawler
document.addEventListener('keydown', (e) => {
    if (e.ctrlKey && e.key === 's') {
        fetch('/api/crawler/start', {method: 'POST'});
    }
    if (e.ctrlKey && e.key === 'e') {
        fetch('/api/crawler/stop', {method: 'POST'});
    }
});
```

## Advanced Configuration

### Pangu Model Deployment (Recommended for Chinese Text)

```python
# Download model from:
# https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

from hotsearch_analysis_agent.models import PanguModel

model = PanguModel(
    model_path='/path/to/openpangu-embedded-7b-model',
    device='cuda',  # or 'npu' for Huawei Ascend
    max_length=2048
)

# Use for long-form Chinese text analysis
result = model.analyze_sentiment(long_chinese_text)
```

### Multi-Platform Cookie Management

```python
# hotsearchcrawler/middlewares/cookie_middleware.py
class CookieRotationMiddleware:
    def __init__(self):
        self.cookies = {
            'weibo': [cookie1, cookie2, cookie3],
            'douyin': [cookie1, cookie2]
        }
    
    def process_request(self, request, spider):
        platform = spider.name
        cookie = random.choice(self.cookies.get(platform, []))
        request.cookies = cookie
```

### Custom Report Templates

```python
from hotsearch_analysis_agent.report import ReportGenerator

generator = ReportGenerator(template='custom_template.md')

report = generator.create_report(
    title='AI与科技热点分析',
    time_range='2026-04-07',
    topics=analyzed_topics,
    include_sentiment=True,
    include_charts=True
)

generator.export(report, format='markdown')  # or 'html', 'pdf'
```

## Common Workflows

### Daily Monitoring Setup

```python
# scheduled_tasks.py
from apscheduler.schedulers.background import BackgroundScheduler
from hotsearch_analysis_agent import OpinionAgent, PushTask

scheduler = BackgroundScheduler()

def daily_analysis_job():
    agent = OpinionAgent()
    topics = agent.query_hot_topics(platforms='all', hours=24)
    
    # Cluster and analyze
    clusters = agent.cluster_topics(topics)
    analysis = agent.generate_comprehensive_report(clusters)
    
    # Send to subscribed channels
    push = PushTask(channels=['wechat', 'email'])
    push.send_notification(analysis)

# Run every day at 8 AM
scheduler.add_job(daily_analysis_job, 'cron', hour=8)
scheduler.start()
```

### Platform-Specific Querying

```python
# Query specific platform with filters
from hotsearch_analysis_agent.database import HotTopicQuery

query = HotTopicQuery()

# Get Weibo hot topics about "AI" with heat > 100k
weibo_ai_topics = query.filter(
    platform='weibo',
    keywords=['人工智能', 'AI'],
    min_heat=100000,
    time_range='24h'
).order_by_heat().limit(20).execute()

for topic in weibo_ai_topics:
    print(f"{topic.rank}. {topic.title} - {topic.heat_score}")
    print(f"   URL: {topic.url}")
    print(f"   Content: {topic.content[:100]}...")
```

## Troubleshooting

### Browser Driver Issues

```bash
# Error: "chromedriver not found"
# Solution 1: Add to PATH
export PATH=$PATH:/path/to/driver/directory

# Solution 2: Specify explicitly in settings
CHROME_DRIVER_PATH=/usr/local/bin/chromedriver

# Verify driver version matches browser
chromedriver --version
google-chrome --version
```

### Database Connection Errors

```python
# Test connection
from hotsearch_analysis_agent.database import test_connection

if not test_connection():
    print("Check MySQL credentials in .env")
    print("Ensure database 'opinion_analytics' exists")
    print("Verify firewall allows port 3306")
```

### LLM API Timeouts

```python
# Increase timeout for large requests
from hotsearch_analysis_agent.config import LLMConfig

LLMConfig.set_timeout(120)  # seconds
LLMConfig.set_max_retries(3)

# Use streaming for long responses
response = agent.chat_stream(
    "分析最近100条关于AI的新闻",
    stream=True
)
for chunk in response:
    print(chunk, end='', flush=True)
```

### Crawler Rate Limiting

```python
# hotsearchcrawler/settings.py
# Adjust these if getting blocked:

DOWNLOAD_DELAY = 2  # Increase delay between requests
CONCURRENT_REQUESTS_PER_DOMAIN = 4  # Reduce concurrency
AUTOTHROTTLE_ENABLED = True
AUTOTHROTTLE_TARGET_CONCURRENCY = 2.0

# Use proxy rotation
DOWNLOADER_MIDDLEWARES = {
    'scrapy.downloadermiddlewares.httpproxy.HttpProxyMiddleware': 110,
    'hotsearchcrawler.middlewares.ProxyMiddleware': 100,
}
```

### Missing Video Content

```python
# Ensure video extractors are enabled
from hotsearch_analysis_agent.extractors import VideoContentExtractor

extractor = VideoContentExtractor(
    enable_transcript=True,  # Extract subtitles
    enable_ocr=True,         # Extract text from video frames
    enable_audio=True        # Convert audio to text
)

content = extractor.extract_from_url(video_url)
```

## Performance Optimization

```python
# Use connection pooling for database
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

engine = create_engine(
    f"mysql+pymysql://{user}:{password}@{host}/{db}",
    poolclass=QueuePool,
    pool_size=10,
    max_overflow=20
)

# Cache frequent queries
from functools import lru_cache

@lru_cache(maxsize=100)
def get_platform_hot_topics(platform, hours=24):
    # Expensive query
    return query_database(platform, hours)

# Batch process topics
def analyze_topics_batch(topics, batch_size=10):
    for i in range(0, len(topics), batch_size):
        batch = topics[i:i+batch_size]
        results = agent.analyze_batch(batch)
        save_results(results)
```
