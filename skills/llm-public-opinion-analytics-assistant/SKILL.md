---
name: llm-public-opinion-analytics-assistant
description: Intelligent public opinion analytics system that aggregates 26 trending lists from 15 platforms with LLM-powered sentiment analysis, topic clustering, and multi-channel alerting
triggers:
  - how do I set up a public opinion monitoring system
  - analyze trending topics across multiple platforms
  - deploy a sentiment analysis crawler system
  - create a hot topic aggregation dashboard
  - build an LLM-powered news analysis tool
  - configure multi-platform trending data scraper
  - set up automated topic clustering and alerts
  - implement Chinese social media sentiment analysis
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive public opinion analytics system that crawls and analyzes trending topics from 15 major Chinese platforms (26 ranking lists total). Features conversational querying, topic clustering, sentiment analysis, and multi-channel push notifications (Email, WeChat, Enterprise WeChat, Telegram).

## What It Does

- **Real-time Data Collection**: Crawls trending topics from Weibo, Bilibili, Zhihu, Baidu, Douyin, and 10+ other platforms
- **LLM-Powered Analysis**: Uses Huawei Pangu or OpenAI-compatible models for sentiment analysis and topic clustering
- **Conversational Interface**: Natural language queries for trending topics and analysis
- **Deep Content Extraction**: Fetches full article/video content using Selenium for comprehensive analysis
- **Multi-Channel Alerts**: Push reports via Enterprise WeChat, Telegram, or email
- **Hotkey Control**: Start/stop crawlers via keyboard shortcuts

## Installation

### Prerequisites

```bash
# System requirements
- Python 3.8+
- MySQL 5.7+ or 8.0+
- Chrome/Edge browser + matching WebDriver
```

### Browser Driver Setup

1. **Check browser version**:
   - Chrome: `chrome://settings/help`
   - Edge: `edge://settings/help`

2. **Download matching driver**:
   - ChromeDriver: https://chromedriver.chromium.org/
   - EdgeDriver: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

3. **Add to PATH**:
   ```bash
   # Linux/macOS
   export PATH=$PATH:/path/to/driver
   
   # Windows: Add driver directory to System PATH via Environment Variables
   ```

4. **Verify**:
   ```bash
   chromedriver --version
   ```

### Python Environment

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

### Database Setup

```python
# Reference init.py for database schema
import pymysql

# Create database
connection = pymysql.connect(
    host='localhost',
    user='root',
    password='${MYSQL_PASSWORD}',
    charset='utf8mb4'
)

with connection.cursor() as cursor:
    cursor.execute("CREATE DATABASE IF NOT EXISTS hotsearch DEFAULT CHARSET utf8mb4 COLLATE utf8mb4_unicode_ci")
    
# Run init.py to create tables
python init.py
```

### Configuration

Create `.env` file in project root:

```env
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=${MYSQL_PASSWORD}
MYSQL_DATABASE=hotsearch

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=${OPENAI_API_KEY}
OPENAI_API_BASE=https://api.openai.com/v1
OPENAI_MODEL=gpt-4

# Or use Huawei Pangu
# PANGU_API_KEY=${PANGU_API_KEY}
# PANGU_API_BASE=https://pangu-api.huaweicloud.com

# Push Notification Channels (optional)
# Enterprise WeChat Bot
WECHAT_WEBHOOK=${WECHAT_WEBHOOK_URL}

# Telegram Bot
TELEGRAM_BOT_TOKEN=${TELEGRAM_BOT_TOKEN}
TELEGRAM_CHAT_ID=${TELEGRAM_CHAT_ID}

# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=${EMAIL_ADDRESS}
SMTP_PASSWORD=${EMAIL_APP_PASSWORD}
SMTP_TO=${RECIPIENT_EMAIL}
```

Configure crawler settings in `hotsearchcrawler/settings.py`:

```python
# MySQL settings
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'root'
MYSQL_PASSWORD = '${MYSQL_PASSWORD}'
MYSQL_DB = 'hotsearch'

# Scrapy settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
COOKIES_ENABLED = True

# User agents rotation
USER_AGENTS = [
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
    # Add more user agents
]
```

## Usage

### Starting the System

```bash
# Start web interface
python app.py

# Access at http://localhost:5000
```

### Running Crawlers

**Via Web Interface** (recommended):
- Click "Start Crawler" button in UI
- Use keyboard shortcuts (configured in frontend)

**Via Command Line**:
```bash
# Test specific spider
python runspider-test.py

# Run all spiders
python run_spiders.py
```

### Querying Trending Topics

**Python API**:

```python
from hotsearch_analysis_agent.query_engine import QueryEngine

# Initialize query engine
engine = QueryEngine()

# Natural language query
results = engine.query("今天关于人工智能的热点有哪些")

# Get specific platform trends
weibo_trends = engine.get_platform_trends("weibo", limit=10)

# Search by keyword
ai_topics = engine.search_topics(keyword="人工智能", days=7)

for topic in ai_topics:
    print(f"{topic['title']} - {topic['platform']} - {topic['hot_value']}")
```

### Sentiment Analysis

```python
from hotsearch_analysis_agent.sentiment_analyzer import SentimentAnalyzer

analyzer = SentimentAnalyzer()

# Analyze single topic
sentiment = analyzer.analyze_topic(topic_id=12345)
print(f"Sentiment: {sentiment['label']}")  # positive/negative/neutral
print(f"Score: {sentiment['score']}")

# Batch analysis
topic_ids = [123, 456, 789]
results = analyzer.batch_analyze(topic_ids)

for result in results:
    print(f"{result['title']}: {result['sentiment']} ({result['confidence']:.2%})")
```

### Topic Clustering

```python
from hotsearch_analysis_agent.clustering import TopicClusterer

clusterer = TopicClusterer()

# Cluster recent topics
clusters = clusterer.cluster_topics(
    start_date="2026-05-01",
    end_date="2026-05-17",
    min_cluster_size=3
)

for cluster in clusters:
    print(f"Cluster: {cluster['theme']}")
    print(f"Topics: {len(cluster['topics'])}")
    for topic in cluster['topics']:
        print(f"  - {topic['title']} ({topic['platform']})")
```

### Setting Up Push Notifications

**Create Push Task**:

```python
from hotsearch_analysis_agent.push_task import PushTaskManager

manager = PushTaskManager()

# Create daily AI news digest
task = manager.create_task(
    name="AI Daily Digest",
    query="人工智能 OR 大模型 OR ChatGPT",
    schedule="0 12 * * *",  # Daily at noon
    channels=["wechat", "telegram", "email"],
    min_hot_value=5000,
    sentiment_filter="all"  # or "positive", "negative"
)

# Test task manually
manager.execute_task(task.id)
```

**Test Push Task**:

```bash
# Test all configured channels
python test_push_task.py
```

### Deep Content Extraction

The system automatically extracts full content (including video transcripts) when analyzing topics:

```python
from hotsearch_analysis_agent.content_extractor import ContentExtractor

extractor = ContentExtractor()

# Extract article content
content = extractor.extract_article(url="https://example.com/news/12345")
print(content['title'])
print(content['text'])
print(content['publish_time'])

# Extract video content (uses Selenium + OCR/ASR)
video_content = extractor.extract_video(url="https://bilibili.com/video/BV1234567890")
print(video_content['transcript'])
print(video_content['comments'])
```

## Common Patterns

### Custom Spider for New Platform

```python
# hotsearchcrawler/spiders/custom_platform.py
import scrapy
from hotsearchcrawler.items import HotSearchItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://example.com/trending']
    
    def parse(self, response):
        for rank, item in enumerate(response.css('.trending-item'), 1):
            yield HotSearchItem(
                platform='CustomPlatform',
                title=item.css('.title::text').get(),
                hot_value=item.css('.hot-value::text').get(),
                url=item.css('a::attr(href)').get(),
                rank=rank,
                category=item.css('.category::text').get()
            )
```

### Scheduled Analysis Reports

```python
from apscheduler.schedulers.background import BackgroundScheduler
from hotsearch_analysis_agent.report_generator import ReportGenerator

def generate_weekly_report():
    generator = ReportGenerator()
    report = generator.create_report(
        query="科技 AND (人工智能 OR 芯片)",
        period_days=7,
        include_sentiment=True,
        include_clustering=True
    )
    generator.send_report(report, channels=["email", "wechat"])

scheduler = BackgroundScheduler()
scheduler.add_job(
    generate_weekly_report,
    'cron',
    day_of_week='mon',
    hour=9,
    minute=0
)
scheduler.start()
```

### Custom LLM Integration

```python
# hotsearch_analysis_agent/llm_client.py
from openai import OpenAI

class CustomLLMClient:
    def __init__(self):
        self.client = OpenAI(
            api_key="${CUSTOM_LLM_API_KEY}",
            base_url="${CUSTOM_LLM_BASE_URL}"
        )
    
    def analyze_sentiment(self, text):
        response = self.client.chat.completions.create(
            model="custom-model",
            messages=[
                {"role": "system", "content": "你是一个专业的舆情分析助手"},
                {"role": "user", "content": f"分析以下文本的情感倾向:\n{text}"}
            ]
        )
        return response.choices[0].message.content
    
    def cluster_topics(self, topics):
        prompt = f"将以下{len(topics)}个话题进行聚类分析:\n"
        for i, topic in enumerate(topics, 1):
            prompt += f"{i}. {topic['title']}\n"
        
        response = self.client.chat.completions.create(
            model="custom-model",
            messages=[{"role": "user", "content": prompt}]
        )
        return response.choices[0].message.content
```

### Database Query Patterns

```python
import pymysql
from contextlib import contextmanager

@contextmanager
def get_db_connection():
    connection = pymysql.connect(
        host='${MYSQL_HOST}',
        user='${MYSQL_USER}',
        password='${MYSQL_PASSWORD}',
        database='hotsearch',
        charset='utf8mb4',
        cursorclass=pymysql.cursors.DictCursor
    )
    try:
        yield connection
    finally:
        connection.close()

# Get top trending topics
with get_db_connection() as conn:
    with conn.cursor() as cursor:
        cursor.execute("""
            SELECT platform, title, hot_value, url, created_at
            FROM trending_topics
            WHERE created_at >= DATE_SUB(NOW(), INTERVAL 24 HOUR)
            ORDER BY hot_value DESC
            LIMIT 50
        """)
        topics = cursor.fetchall()

# Get platform statistics
with get_db_connection() as conn:
    with conn.cursor() as cursor:
        cursor.execute("""
            SELECT 
                platform,
                COUNT(*) as topic_count,
                AVG(hot_value) as avg_hot_value,
                MAX(hot_value) as max_hot_value
            FROM trending_topics
            WHERE created_at >= DATE_SUB(NOW(), INTERVAL 7 DAY)
            GROUP BY platform
            ORDER BY topic_count DESC
        """)
        stats = cursor.fetchall()
```

## Troubleshooting

### Crawler Issues

**Spider not collecting data**:
```python
# Enable debug logging in settings.py
LOG_LEVEL = 'DEBUG'

# Test spider individually
scrapy crawl weibo -o output.json
```

**Selenium WebDriver errors**:
```bash
# Check driver version matches browser
chromedriver --version
google-chrome --version

# Use headless mode if GUI not available
# In content_extractor.py:
options.add_argument('--headless')
options.add_argument('--no-sandbox')
options.add_argument('--disable-dev-shm-usage')
```

**Anti-crawler detection**:
```python
# Increase delays in settings.py
DOWNLOAD_DELAY = 3
RANDOMIZE_DOWNLOAD_DELAY = True

# Add cookies for specific platforms
# hotsearchcrawler/settings.py
PLATFORM_COOKIES = {
    'weibo': 'your_cookie_string'
}
```

### Database Connection Issues

```python
# Test connection
import pymysql

try:
    connection = pymysql.connect(
        host='${MYSQL_HOST}',
        user='${MYSQL_USER}',
        password='${MYSQL_PASSWORD}',
        database='hotsearch'
    )
    print("Database connection successful")
    connection.close()
except Exception as e:
    print(f"Connection failed: {e}")
```

**Character encoding errors**:
```sql
-- Ensure database uses utf8mb4
ALTER DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
ALTER TABLE trending_topics CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### LLM API Issues

**Timeout errors**:
```python
# Increase timeout in client
from openai import OpenAI

client = OpenAI(
    api_key="${OPENAI_API_KEY}",
    timeout=60.0  # Increase from default 30s
)
```

**Rate limiting**:
```python
import time
from functools import wraps

def rate_limit(calls_per_minute=10):
    min_interval = 60.0 / calls_per_minute
    last_called = [0.0]
    
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            if elapsed < min_interval:
                time.sleep(min_interval - elapsed)
            result = func(*args, **kwargs)
            last_called[0] = time.time()
            return result
        return wrapper
    return decorator

@rate_limit(calls_per_minute=10)
def analyze_with_llm(text):
    # Your LLM call here
    pass
```

### Push Notification Issues

**Enterprise WeChat webhook not working**:
```python
# Test webhook
import requests

response = requests.post(
    '${WECHAT_WEBHOOK}',
    json={
        "msgtype": "text",
        "text": {"content": "Test message"}
    }
)
print(response.status_code, response.text)
```

**Email delivery failures**:
```python
# Test SMTP connection
import smtplib

try:
    server = smtplib.SMTP('${SMTP_HOST}', ${SMTP_PORT})
    server.starttls()
    server.login('${SMTP_USER}', '${SMTP_PASSWORD}')
    print("SMTP connection successful")
    server.quit()
except Exception as e:
    print(f"SMTP error: {e}")
```

### Performance Optimization

**Slow queries**:
```sql
-- Add indexes
CREATE INDEX idx_created_at ON trending_topics(created_at);
CREATE INDEX idx_platform ON trending_topics(platform);
CREATE INDEX idx_hot_value ON trending_topics(hot_value DESC);

-- Composite index for common queries
CREATE INDEX idx_platform_date ON trending_topics(platform, created_at);
```

**Memory usage**:
```python
# Process topics in batches
def process_in_batches(items, batch_size=100):
    for i in range(0, len(items), batch_size):
        batch = items[i:i + batch_size]
        yield batch

# Usage
for batch in process_in_batches(all_topics):
    results = analyzer.batch_analyze(batch)
```
