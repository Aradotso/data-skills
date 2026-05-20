---
name: llm-public-opinion-analytics-assistant
description: Build real-time public opinion monitoring systems with multi-platform hot search crawlers, LLM-powered sentiment analysis, topic clustering, and multi-channel alerting (WeChat, Telegram, Email)
triggers:
  - set up public opinion monitoring system
  - analyze hot topics from weibo bilibili douyin
  - create sentiment analysis for social media trends
  - build multi-platform hot search crawler
  - implement topic clustering with LLM
  - configure automated trend alerts to telegram
  - monitor real-time public opinion across platforms
  - analyze social media sentiment with AI
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive public opinion monitoring system that integrates real-time data from **15 mainstream platforms** (26 ranking lists total) with LLM analysis capabilities. The system provides conversational hot search queries, topic clustering, sentiment analysis, and multi-channel alerting (Enterprise WeChat, Telegram, Email).

## What It Does

- **Multi-Platform Crawling**: Collects hot search data from Weibo, Bilibili, Douyin, Zhihu, Baidu, and 10+ other Chinese platforms
- **LLM-Powered Analysis**: Uses Huawei Pangu (or OpenAI-compatible models) for sentiment analysis and topic clustering
- **Natural Language Queries**: Conversational interface for querying trends and topics
- **Deep Content Extraction**: Extracts content from news detail pages, including video transcripts
- **Automated Alerting**: Pushes trend reports via Enterprise WeChat, Telegram, or SMTP email
- **Web Dashboard**: Frontend interface for data exploration and crawler control

## Installation

### Prerequisites

**1. Browser Driver Setup**

For Chrome:
```bash
# Download ChromeDriver matching your Chrome version
# From: https://chromedriver.chromium.org/

# Linux/macOS - place in PATH
sudo mv chromedriver /usr/local/bin/
chmod +x /usr/local/bin/chromedriver

# Verify
chromedriver --version
```

For Edge:
```bash
# Download from: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/
# Place in system PATH similarly
```

**2. MySQL Database**

```bash
# Install MySQL 5.7+ or 8.0+
# Create database
mysql -u root -p
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

**3. Python Environment**

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
# Reference init.py for table schemas
# Key tables:
# - hot_search_items: Stores trending topics
# - platform_data: Raw platform data
# - analysis_results: LLM analysis outputs
# - push_tasks: Alert configurations

# Run initialization
python init.py
```

## Configuration

### Crawler Configuration

**`hotsearchcrawler/settings.py`**:

```python
# MySQL Configuration
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_user'
MYSQL_PASSWORD = 'your_password'
MYSQL_DB = 'hotsearch_db'

# Optional: Platform-specific cookies for authenticated access
COOKIES = {
    'weibo': 'your_weibo_cookie',
    'bilibili': 'your_bilibili_cookie'
}

# Crawler settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
USER_AGENT = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'
```

### Analysis System Configuration

**`.env` file** (create in project root):

```env
# MySQL Database
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DB=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key_here
OPENAI_API_BASE=https://api.openai.com/v1
OPENAI_MODEL=gpt-4

# Or use Huawei Pangu local deployment
USE_LOCAL_PANGU=true
PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model

# Push Notification Channels
# Enterprise WeChat Robot
WECHAT_WEBHOOK_URL=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
EMAIL_FROM=your_email@gmail.com
EMAIL_TO=recipient@example.com
```

## Running the System

### Start the Analysis Web Interface

```bash
python app.py
# Access at http://localhost:5000
```

### Run Crawlers

**Via Web Interface**:
- Use keyboard shortcut to start/stop crawlers from the dashboard

**Via Command Line**:

```bash
# Test single spider
python runspider-test.py weibo

# Run all spiders
python run_spiders.py
```

### Test Push Notifications

```bash
python test_push_task.py
```

## Key API Usage

### Crawler Development Pattern

```python
# hotsearchcrawler/spiders/example_spider.py
import scrapy
from hotsearchcrawler.items import HotSearchItem

class ExampleSpider(scrapy.Spider):
    name = 'example_platform'
    
    def start_requests(self):
        url = 'https://example.com/trending'
        yield scrapy.Request(url, callback=self.parse)
    
    def parse(self, response):
        for item in response.css('.trending-item'):
            hot_item = HotSearchItem()
            hot_item['platform'] = 'example'
            hot_item['title'] = item.css('.title::text').get()
            hot_item['url'] = item.css('a::attr(href)').get()
            hot_item['rank'] = item.css('.rank::text').get()
            hot_item['heat_value'] = item.css('.heat::text').get()
            yield hot_item
```

### LLM Analysis Integration

```python
from hotsearch_analysis_agent.analyzer import TopicAnalyzer

# Initialize analyzer
analyzer = TopicAnalyzer(
    api_key=os.getenv('OPENAI_API_KEY'),
    model=os.getenv('OPENAI_MODEL', 'gpt-4')
)

# Analyze sentiment
sentiment = analyzer.analyze_sentiment(
    text="这个政策真的太好了,支持!",
    platform="weibo"
)
# Returns: {"sentiment": "positive", "confidence": 0.92, "keywords": ["政策", "支持"]}

# Topic clustering
topics = analyzer.cluster_topics(
    items=[
        {"title": "AI技术突破", "content": "..."},
        {"title": "人工智能新进展", "content": "..."},
        {"title": "机器学习应用", "content": "..."}
    ]
)
# Returns clustered topic groups with labels
```

### Database Query Patterns

```python
import pymysql
import os

# Connect to database
conn = pymysql.connect(
    host=os.getenv('MYSQL_HOST'),
    port=int(os.getenv('MYSQL_PORT')),
    user=os.getenv('MYSQL_USER'),
    password=os.getenv('MYSQL_PASSWORD'),
    database=os.getenv('MYSQL_DB'),
    charset='utf8mb4'
)

# Query hot searches
with conn.cursor() as cursor:
    sql = """
        SELECT platform, title, url, rank, heat_value, created_at
        FROM hot_search_items
        WHERE platform = %s AND created_at > NOW() - INTERVAL 1 HOUR
        ORDER BY rank ASC
    """
    cursor.execute(sql, ('weibo',))
    results = cursor.fetchall()

# Search by keyword
with conn.cursor() as cursor:
    sql = """
        SELECT * FROM hot_search_items
        WHERE title LIKE %s OR content LIKE %s
        ORDER BY created_at DESC
        LIMIT 50
    """
    keyword = '%人工智能%'
    cursor.execute(sql, (keyword, keyword))
    results = cursor.fetchall()

conn.close()
```

### Push Task Configuration

```python
from hotsearch_analysis_agent.push import PushTaskManager

# Initialize push manager
push_manager = PushTaskManager()

# Create push task
task = push_manager.create_task(
    name="AI热点监测",
    query_keywords=["人工智能", "AI", "机器学习"],
    platforms=["weibo", "zhihu", "bilibili"],
    schedule="0 */6 * * *",  # Every 6 hours
    channels={
        "telegram": {
            "enabled": True,
            "bot_token": os.getenv('TELEGRAM_BOT_TOKEN'),
            "chat_id": os.getenv('TELEGRAM_CHAT_ID')
        },
        "wechat": {
            "enabled": True,
            "webhook_url": os.getenv('WECHAT_WEBHOOK_URL')
        },
        "email": {
            "enabled": False
        }
    },
    analysis_config={
        "sentiment_analysis": True,
        "topic_clustering": True,
        "min_heat_threshold": 100000
    }
)
```

### Natural Language Query Interface

```python
from hotsearch_analysis_agent.chat import ChatInterface

chat = ChatInterface()

# Query hot searches
response = chat.query("今天微博上有什么热搜?")
# Returns formatted list of Weibo trending topics

# Topic search
response = chat.query("最近关于人工智能的热点有哪些?")
# Returns clustered AI-related topics across platforms

# Sentiment analysis
response = chat.query("分析一下华为新产品的舆情倾向")
# Returns sentiment distribution and key opinions
```

## Common Patterns

### Building Custom Analysis Pipeline

```python
from hotsearch_analysis_agent.pipeline import AnalysisPipeline

# Define custom pipeline
pipeline = AnalysisPipeline([
    ('fetch', 'multi_platform_fetcher'),
    ('filter', 'keyword_filter', {'keywords': ['科技', 'AI']}),
    ('analyze', 'sentiment_analyzer'),
    ('cluster', 'topic_clusterer', {'min_cluster_size': 3}),
    ('summarize', 'llm_summarizer'),
    ('push', 'multi_channel_pusher')
])

# Execute pipeline
results = pipeline.run(
    platforms=['weibo', 'zhihu', 'bilibili'],
    time_range='24h'
)
```

### Video Content Extraction

```python
from hotsearch_analysis_agent.extractors import VideoExtractor

# Extract content from video news (Bilibili, Douyin, etc.)
extractor = VideoExtractor()

content = extractor.extract(
    url='https://www.bilibili.com/video/BV13pSoBBEvX',
    include_transcript=True,
    include_comments=True
)

# Returns:
# {
#   'title': '...',
#   'transcript': '...',  # AI-extracted speech-to-text
#   'description': '...',
#   'top_comments': [...],
#   'metadata': {...}
# }
```

### Trend Report Generation

```python
from hotsearch_analysis_agent.reports import ReportGenerator
from datetime import datetime

generator = ReportGenerator()

report = generator.generate(
    topic="人工智能与前沿科技",
    time_range="7d",
    platforms=["weibo", "zhihu", "36kr", "bilibili"],
    analysis_depth="comprehensive",
    include_charts=True
)

# Save as markdown
with open(f"report_{datetime.now().strftime('%Y%m%d')}.md", 'w') as f:
    f.write(report.to_markdown())

# Send via configured channels
report.push(channels=['telegram', 'wechat', 'email'])
```

## Troubleshooting

### Browser Driver Issues

```bash
# Error: ChromeDriver not found
# Solution: Verify driver is in PATH
which chromedriver  # Linux/macOS
where chromedriver  # Windows

# Error: Version mismatch
# Solution: Download matching version
google-chrome --version  # Check browser version
# Download exact matching driver version
```

### Database Connection Errors

```python
# Test connection
import pymysql

try:
    conn = pymysql.connect(
        host=os.getenv('MYSQL_HOST'),
        port=int(os.getenv('MYSQL_PORT')),
        user=os.getenv('MYSQL_USER'),
        password=os.getenv('MYSQL_PASSWORD')
    )
    print("Connection successful")
    conn.close()
except Exception as e:
    print(f"Connection failed: {e}")
```

### LLM API Timeout

```python
# Increase timeout in analyzer
analyzer = TopicAnalyzer(
    api_key=os.getenv('OPENAI_API_KEY'),
    timeout=60,  # seconds
    max_retries=3
)
```

### Crawler Being Blocked

```python
# In hotsearchcrawler/settings.py
# Adjust delay and concurrent requests
DOWNLOAD_DELAY = 3  # Increase delay
CONCURRENT_REQUESTS = 8  # Reduce concurrency

# Add random user agent rotation
DOWNLOADER_MIDDLEWARES = {
    'scrapy.downloadermiddlewares.useragent.UserAgentMiddleware': None,
    'scrapy_user_agents.middlewares.RandomUserAgentMiddleware': 400,
}
```

### Push Notification Not Sending

```bash
# Test each channel independently
python test_push_task.py --channel telegram
python test_push_task.py --channel wechat
python test_push_task.py --channel email

# Check logs
tail -f logs/push_tasks.log
```

## Project Structure Reference

```
.
├── app.py                          # Main web application entry
├── hotsearch_analysis_agent/       # Analysis system
│   ├── analyzer.py                 # LLM analysis core
│   ├── chat.py                     # Chat interface
│   ├── pipeline.py                 # Analysis pipeline
│   ├── push.py                     # Push task manager
│   └── reports.py                  # Report generator
├── hotsearchcrawler/               # Crawler cluster
│   ├── spiders/                    # Platform-specific spiders
│   ├── items.py                    # Data models
│   ├── pipelines.py                # Data processing
│   └── settings.py                 # Crawler configuration
├── run_spiders.py                  # Crawler orchestrator
├── test_push_task.py               # Push testing utility
├── init.py                         # Database initialization
└── requirements.txt                # Python dependencies
```

Use this skill to build comprehensive public opinion monitoring systems with automated analysis and alerting across multiple Chinese social media platforms.
