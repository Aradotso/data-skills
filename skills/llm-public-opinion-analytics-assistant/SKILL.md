---
name: llm-public-opinion-analytics-assistant
description: Multi-platform hot topic crawler and LLM-powered public opinion analysis system with sentiment analysis, topic clustering, and multi-channel alerting
triggers:
  - analyze hot topics from social media platforms
  - set up public opinion monitoring and alerts
  - crawl trending topics from multiple platforms
  - perform sentiment analysis on news and social posts
  - cluster related topics using LLM
  - configure hot topic push notifications
  - scrape and analyze Chinese social media trends
  - build a public opinion analytics dashboard
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is a comprehensive public opinion monitoring system that crawls 26 trending lists from 15 major Chinese platforms (Weibo, Bilibili, Toutiao, etc.), analyzes content using large language models, and provides conversational queries, sentiment analysis, topic clustering, and multi-channel alerts (email, WeChat, Telegram).

## What It Does

- **Multi-Platform Crawling**: Scrapes trending topics from 15 platforms including Weibo, Bilibili, Zhihu, Douyin, Baidu, etc.
- **LLM-Powered Analysis**: Uses models like Pangu (recommended) or OpenAI-compatible APIs for deep content analysis
- **Video Content Extraction**: Extracts text from video-based news using browser automation
- **Sentiment Analysis**: Detects positive/negative trends in public opinion
- **Topic Clustering**: Groups related topics across platforms
- **Conversational Interface**: Natural language queries for trend exploration
- **Multi-Channel Alerts**: Push reports via email, WeChat Work, Telegram
- **Hotkey Control**: Start/stop crawlers with keyboard shortcuts

## Installation

### Prerequisites

**1. Browser Driver Setup** (required for video content extraction):

```bash
# Check your Chrome/Edge version first
google-chrome --version  # or microsoft-edge --version

# Download matching ChromeDriver from:
# https://chromedriver.chromium.org/
# Or EdgeDriver from:
# https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Place driver in system PATH or project directory
# Example for Linux/macOS:
sudo mv chromedriver /usr/local/bin/
sudo chmod +x /usr/local/bin/chromedriver

# Verify installation
chromedriver --version
```

**2. MySQL Database**:

```bash
# Install MySQL 5.7+ or 8.0+
# Create database and tables using init.py as reference

mysql -u root -p
CREATE DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

**3. Python Environment**:

```bash
# Clone repository
git clone https://github.com/hmmnxkl/LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant.git
cd LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Database Initialization

Reference `init.py` to create tables:

```python
import pymysql

# Connect to MySQL
conn = pymysql.connect(
    host='localhost',
    user='your_user',
    password='your_password',
    database='hotsearch',
    charset='utf8mb4'
)

# Create tables for each platform
# Example table structure:
cursor = conn.cursor()
cursor.execute("""
CREATE TABLE IF NOT EXISTS weibo_hot (
    id INT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(500),
    url VARCHAR(1000),
    hot_value VARCHAR(100),
    rank INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_created (created_at),
    INDEX idx_title (title(255))
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
""")
conn.commit()
```

## Configuration

### 1. Crawler Configuration

Edit `hotsearchcrawler/settings.py`:

```python
# MySQL connection
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_user'
MYSQL_PASSWORD = 'your_password'
MYSQL_DATABASE = 'hotsearch'

# Optional: Platform-specific cookies for authenticated content
COOKIES = {
    'weibo': 'your_cookie_string',
    'bilibili': 'your_cookie_string'
}

# Spider settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 2
ROBOTSTXT_OBEY = False
```

### 2. Analysis System Configuration

Create `.env` file in project root:

```bash
# Database
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_BASE_URL=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Or use Pangu model (recommended for Chinese content)
# PANGU_MODEL_PATH=/path/to/pangu-model
# USE_LOCAL_MODEL=true

# Email Notifications
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
NOTIFICATION_EMAIL=recipient@example.com

# WeChat Work Bot
WECHAT_WEBHOOK_URL=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=your_key

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

## Running the System

### Start Web Interface

```bash
# Launch Flask application
python app.py

# Access at http://localhost:5000
# Features:
# - Conversational topic queries
# - Platform-specific trend browsing
# - Real-time crawler control via hotkeys
# - Topic clustering visualization
```

### Manual Crawler Execution

```bash
# Test individual platform crawler
python runspider-test.py --platform weibo

# Run all crawlers
python run_spiders.py
```

### Test Push Notifications

```bash
# Test notification channels
python test_push_task.py

# This will send a sample report to configured channels
```

## Key API Patterns

### Querying Hot Topics

```python
from hotsearch_analysis_agent.database import get_recent_topics

# Get latest topics from specific platform
topics = get_recent_topics(
    platform='weibo',
    hours=24,
    limit=50
)

for topic in topics:
    print(f"{topic['rank']}. {topic['title']} ({topic['hot_value']})")
```

### Sentiment Analysis

```python
from hotsearch_analysis_agent.analyzer import analyze_sentiment

# Analyze sentiment of a topic
result = analyze_sentiment(
    text="人工智能技术取得重大突破",
    platform='weibo',
    topic_id=12345
)

print(f"Sentiment: {result['sentiment']}")  # positive/negative/neutral
print(f"Score: {result['score']}")  # -1.0 to 1.0
print(f"Keywords: {result['keywords']}")
```

### Topic Clustering

```python
from hotsearch_analysis_agent.clustering import cluster_topics

# Find related topics across platforms
clusters = cluster_topics(
    query="人工智能",
    platforms=['weibo', 'zhihu', 'toutiao'],
    time_range_hours=48
)

for cluster in clusters:
    print(f"\nCluster: {cluster['theme']}")
    print(f"Topics: {len(cluster['topics'])}")
    for topic in cluster['topics']:
        print(f"  - [{topic['platform']}] {topic['title']}")
```

### LLM-Powered Analysis

```python
from hotsearch_analysis_agent.llm import generate_report

# Generate comprehensive analysis report
report = generate_report(
    topic="人工智能与前沿科技",
    platforms=['weibo', 'bilibili', 'zhihu'],
    time_range_hours=72,
    include_sentiment=True,
    include_clustering=True
)

print(report['summary'])
print(report['key_findings'])
print(report['trending_topics'])
```

### Scheduling Push Tasks

```python
from hotsearch_analysis_agent.push import schedule_push_task

# Configure automated alerts
task = schedule_push_task(
    topic="科技热点",
    platforms=['weibo', 'toutiao'],
    schedule='daily',  # or 'hourly', 'weekly'
    time='09:00',
    channels=['email', 'wechat', 'telegram'],
    min_hot_value=100000,  # Only alert if topic exceeds threshold
    sentiment_filter='negative'  # Only negative sentiment topics
)
```

## Common Patterns

### Custom Platform Spider

```python
# In hotsearchcrawler/spiders/
import scrapy
from hotsearchcrawler.items import HotTopicItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://platform.com/trending']
    
    def parse(self, response):
        for topic in response.css('.topic-item'):
            item = HotTopicItem()
            item['title'] = topic.css('.title::text').get()
            item['url'] = topic.css('a::attr(href)').get()
            item['hot_value'] = topic.css('.hot::text').get()
            item['rank'] = topic.css('.rank::text').get()
            item['platform'] = self.name
            yield item
```

### Video Content Extraction

```python
from hotsearchcrawler.video_extractor import extract_video_content

# Extract text from video-based news
content = extract_video_content(
    url='https://www.bilibili.com/video/BV13pSoBBEvX',
    method='selenium'  # Uses browser driver
)

print(f"Title: {content['title']}")
print(f"Description: {content['description']}")
print(f"Comments: {content['top_comments']}")
```

### Multi-Platform Query

```python
from hotsearch_analysis_agent.query import conversational_query

# Natural language query across platforms
response = conversational_query(
    user_input="最近关于人工智能的热点有哪些?",
    platforms='all',
    time_range='3d'
)

print(response['answer'])
print(f"Found {len(response['topics'])} related topics")
```

## Troubleshooting

### Browser Driver Issues

```bash
# If "chromedriver not found" error:
which chromedriver  # Check if in PATH
export PATH=$PATH:/path/to/driver/directory

# If version mismatch:
google-chrome --version
# Download matching driver version from chromedriver.chromium.org
```

### Database Connection Errors

```python
# Test MySQL connection
import pymysql
try:
    conn = pymysql.connect(
        host='localhost',
        user='your_user',
        password='your_password',
        database='hotsearch'
    )
    print("Connection successful")
except Exception as e:
    print(f"Error: {e}")
```

### LLM API Timeout

```python
# Increase timeout in analysis agent
from hotsearch_analysis_agent.llm import LLMAnalyzer

analyzer = LLMAnalyzer(
    api_key=os.getenv('OPENAI_API_KEY'),
    timeout=120,  # Increase to 120 seconds
    max_retries=3
)
```

### Crawler Rate Limiting

```python
# In hotsearchcrawler/settings.py
DOWNLOAD_DELAY = 5  # Increase delay between requests
CONCURRENT_REQUESTS_PER_DOMAIN = 4  # Reduce concurrency
RETRY_TIMES = 5
RETRY_HTTP_CODES = [500, 502, 503, 504, 408, 429]
```

### Memory Issues with Large Datasets

```python
# Process in batches
from hotsearch_analysis_agent.database import batch_process

topics = get_recent_topics(platform='weibo', hours=168)
for batch in batch_process(topics, batch_size=100):
    analyze_sentiment_batch(batch)
```

## Advanced Usage

### Custom Analysis Pipeline

```python
from hotsearch_analysis_agent import AnalysisPipeline

pipeline = AnalysisPipeline()
pipeline.add_stage('fetch', platforms=['weibo', 'zhihu'])
pipeline.add_stage('filter', min_hot_value=50000)
pipeline.add_stage('sentiment', model='pangu')
pipeline.add_stage('cluster', algorithm='kmeans', n_clusters=5)
pipeline.add_stage('summarize', max_length=500)
pipeline.add_stage('push', channels=['email', 'telegram'])

results = pipeline.run(topic="人工智能")
```

### Real-time Monitoring Dashboard

```python
# Use Flask-SocketIO for real-time updates
from flask import Flask
from flask_socketio import SocketIO, emit

app = Flask(__name__)
socketio = SocketIO(app)

@socketio.on('monitor_topic')
def monitor(data):
    topic = data['topic']
    while True:
        updates = get_latest_updates(topic)
        emit('topic_update', updates)
        time.sleep(60)
```

This skill provides comprehensive coverage of the public opinion analytics system's capabilities for crawling, analyzing, and monitoring trending topics across Chinese social media platforms using LLM-powered insights.
