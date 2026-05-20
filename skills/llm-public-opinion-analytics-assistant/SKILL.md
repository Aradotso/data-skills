---
name: llm-public-opinion-analytics-assistant
description: Multi-platform public opinion analytics assistant that aggregates hot search data from 15 platforms and provides LLM-powered sentiment analysis, topic clustering, and multi-channel alert notifications
triggers:
  - "set up public opinion monitoring for Chinese social media platforms"
  - "analyze sentiment and trends across Weibo, Bilibili, and other platforms"
  - "configure hot topic push notifications to WeChat or Telegram"
  - "crawl and analyze trending topics from multiple Chinese platforms"
  - "implement conversational interface for hot search queries"
  - "deploy LLM-based sentiment analysis for social media monitoring"
  - "build real-time public opinion analytics dashboard"
  - "aggregate and cluster trending topics across platforms"
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an intelligent public opinion analytics assistant that aggregates real-time data from 26 hot search lists across 15 mainstream Chinese platforms (Weibo, Bilibili, Douyin, etc.). It combines web crawling, LLM-powered analysis (sentiment, clustering, topic extraction), and multi-channel push notifications (email, WeChat, Enterprise WeChat, Telegram).

## What It Does

- **Data Collection**: Crawls 26 hot search lists from 15 platforms using Scrapy
- **Conversational Interface**: Chat-based queries for hot topics, trends, and platform-specific searches
- **LLM Analysis**: Topic clustering, sentiment analysis, and trend detection using Huawei Pangu or OpenAI-compatible models
- **Content Extraction**: Deep-dive into news detail pages, including video content analysis
- **Multi-Channel Alerts**: Push notifications via email (SMTP), WeChat (Enterprise), Telegram bots
- **Keyword Control**: Start/stop crawlers via hotkeys, quick platform data lookup

## Installation

### Prerequisites

**1. Browser Driver Setup (Required for Detail Page Scraping)**

The system uses Selenium to extract content from detail pages. You need Chrome/Chromium or Edge driver:

```bash
# Check your browser version first
# Chrome: Settings → About Chrome
# Edge: Settings → About Microsoft Edge

# Download matching driver:
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Place driver in PATH or project directory
# Linux/macOS example:
sudo mv chromedriver /usr/local/bin/
chmod +x /usr/local/bin/chromedriver

# Verify installation
chromedriver --version
```

**2. MySQL Database**

```bash
# Install MySQL 5.7+ or compatible (MariaDB)
# Create database and tables using init.py as reference

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
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Configuration

**1. Create `.env` file in project root:**

```bash
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_db_user
MYSQL_PASSWORD=your_db_password
MYSQL_DATABASE=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key_here
OPENAI_API_BASE=https://api.openai.com/v1  # Or your custom endpoint

# For Huawei Pangu model (if using local deployment)
PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model

# Push Notification Channels (optional)
# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
SMTP_RECIPIENT=recipient@example.com

# Enterprise WeChat Robot
WECHAT_WEBHOOK_URL=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

**2. Configure Crawler Settings** (`hotsearchcrawler/settings.py`):

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

# Optional: Platform-specific cookies (some platforms require login)
COOKIES_ENABLED = True
COOKIES = {
    'weibo': 'your_weibo_cookies_here',
    'bilibili': 'your_bilibili_cookies_here'
}
```

**3. Initialize Database:**

```python
# Reference init.py to create required tables
# Example schema for hot_search_items table:
from hotsearch_analysis_agent.database import init_database

init_database()  # Creates all required tables
```

## Core Usage

### Starting the System

```bash
# Start web interface and analysis agent
python app.py

# System will start on http://localhost:5000 (default)
# Access the conversational interface in browser
```

### Running Crawlers

**Option 1: Via Web Interface**

- Open http://localhost:5000
- Use hotkey controls to start/stop specific platform crawlers
- Monitor crawl status in real-time

**Option 2: Command Line**

```bash
# Run specific crawler
cd hotsearchcrawler
scrapy crawl weibo_hot

# Run all crawlers
python run_spiders.py

# Test single crawler
python runspider-test.py --spider=bilibili_hot
```

### Conversational Queries

Example queries in the web interface:

```
User: "今天微博热搜前10是什么？"
Assistant: [Returns top 10 Weibo trending topics with links]

User: "帮我分析一下最近关于人工智能的舆情"
Assistant: [Performs clustering, sentiment analysis, generates report]

User: "B站科技区有什么热点？"
Assistant: [Filters Bilibili tech section trending videos]
```

### Programmatic API Usage

```python
from hotsearch_analysis_agent.analyzer import HotSearchAnalyzer
from hotsearch_analysis_agent.database import HotSearchDB

# Initialize analyzer with LLM
analyzer = HotSearchAnalyzer(
    api_key=os.getenv('OPENAI_API_KEY'),
    api_base=os.getenv('OPENAI_API_BASE')
)

# Query hot topics by keyword
db = HotSearchDB()
results = db.search_by_keyword(
    keyword="人工智能",
    platforms=["weibo", "zhihu", "bilibili"],
    date_range=("2026-05-01", "2026-05-19")
)

# Perform sentiment analysis
sentiment_report = analyzer.analyze_sentiment(results)
print(sentiment_report['overall_sentiment'])  # 'positive', 'negative', 'neutral'
print(sentiment_report['score'])  # -1.0 to 1.0

# Topic clustering
clusters = analyzer.cluster_topics(results, num_clusters=5)
for cluster in clusters:
    print(f"Cluster: {cluster['label']}")
    print(f"Topics: {cluster['topics']}")
    print(f"Sentiment: {cluster['avg_sentiment']}")
```

### Setting Up Push Notifications

```python
from hotsearch_analysis_agent.push_service import PushTaskManager

# Create push task
push_manager = PushTaskManager()

# Configure alert for specific keywords
push_manager.create_task(
    name="AI热点监控",
    keywords=["人工智能", "ChatGPT", "大模型"],
    platforms=["weibo", "zhihu", "bilibili"],
    channels=["wechat", "telegram", "email"],
    schedule="0 */4 * * *",  # Every 4 hours
    sentiment_threshold=0.5,  # Only positive sentiment
    min_heat_score=5000  # Minimum engagement threshold
)

# Test push (sends immediately)
push_manager.test_push(
    task_name="AI热点监控",
    channel="telegram"
)
```

### Analyzing Video Content

```python
from hotsearch_analysis_agent.content_extractor import VideoContentExtractor

extractor = VideoContentExtractor()

# Extract content from video news (Bilibili, Douyin, etc.)
video_url = "https://www.bilibili.com/video/BV13pSoBBEvX"
content = extractor.extract(video_url)

print(content['title'])
print(content['description'])
print(content['transcript'])  # AI-extracted transcript if available
print(content['comments_summary'])  # Top comments analysis
```

## Common Patterns

### Pattern 1: Daily Trend Report Generation

```python
from hotsearch_analysis_agent.report_generator import TrendReportGenerator
from datetime import datetime, timedelta

generator = TrendReportGenerator()

# Generate daily report
report = generator.generate_daily_report(
    date=datetime.now().date(),
    topics=["科技", "金融", "政策"],
    platforms="all"
)

# Report includes:
# - Top trending topics with clustering
# - Sentiment analysis
# - Platform-specific insights
# - Links to original content

print(report.to_markdown())  # Markdown format
report.save_html("daily_report.html")
report.send_email(recipients=[os.getenv('SMTP_RECIPIENT')])
```

### Pattern 2: Custom Crawler for New Platform

```python
# In hotsearchcrawler/spiders/custom_spider.py
import scrapy
from hotsearchcrawler.items import HotSearchItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    allowed_domains = ['example.com']
    start_urls = ['https://example.com/trending']

    def parse(self, response):
        for item in response.css('.trending-item'):
            yield HotSearchItem(
                platform='custom_platform',
                title=item.css('.title::text').get(),
                url=item.css('a::attr(href)').get(),
                rank=item.css('.rank::text').get(),
                heat_score=item.css('.heat::text').get(),
                timestamp=datetime.now()
            )
```

### Pattern 3: Multi-Platform Comparison

```python
from hotsearch_analysis_agent.comparator import PlatformComparator

comparator = PlatformComparator()

# Compare trending topics across platforms
comparison = comparator.compare_platforms(
    platforms=["weibo", "zhihu", "douyin"],
    date_range=("2026-05-18", "2026-05-19"),
    topic="人工智能"
)

print(f"Cross-platform topics: {comparison['common_topics']}")
print(f"Platform-specific topics: {comparison['unique_topics']}")
print(f"Sentiment differences: {comparison['sentiment_variance']}")
```

### Pattern 4: Real-time Alert System

```python
from hotsearch_analysis_agent.alert_system import AlertMonitor
import asyncio

monitor = AlertMonitor()

# Define alert rules
monitor.add_rule(
    rule_id="breaking_news",
    keywords=["突发", "紧急", "重大"],
    heat_threshold=10000,
    platforms=["weibo", "toutiao"],
    callback=lambda event: monitor.send_urgent_alert(
        event,
        channels=["telegram", "wechat"]
    )
)

# Start monitoring (runs in background)
asyncio.run(monitor.start())
```

## Advanced Configuration

### LLM Model Selection

```python
# In hotsearch_analysis_agent/config.py

# Option 1: OpenAI-compatible API
LLM_CONFIG = {
    'provider': 'openai',
    'model': 'gpt-4-turbo',
    'api_key': os.getenv('OPENAI_API_KEY'),
    'temperature': 0.7,
    'max_tokens': 4000
}

# Option 2: Huawei Pangu (local deployment)
LLM_CONFIG = {
    'provider': 'pangu',
    'model_path': os.getenv('PANGU_MODEL_PATH'),
    'device': 'cuda',  # or 'cpu'
    'precision': 'fp16',
    'max_context_length': 8192
}

# Option 3: Custom OpenAI-compatible endpoint
LLM_CONFIG = {
    'provider': 'custom',
    'api_base': 'http://localhost:8000/v1',
    'model': 'custom-model-name',
    'api_key': 'not-needed-for-local'
}
```

### Database Optimization

```python
# In hotsearch_analysis_agent/database.py

# Enable query caching
CACHE_CONFIG = {
    'enabled': True,
    'backend': 'redis',
    'redis_url': os.getenv('REDIS_URL', 'redis://localhost:6379/0'),
    'ttl': 3600  # Cache for 1 hour
}

# Configure connection pool
MYSQL_POOL_CONFIG = {
    'pool_size': 10,
    'max_overflow': 20,
    'pool_recycle': 3600,
    'pool_pre_ping': True
}
```

## Troubleshooting

### Crawler Issues

**Problem**: Crawler returns empty results
```bash
# Check if target website structure changed
scrapy shell "https://weibo.com/hot/search"
# Test CSS/XPath selectors in shell

# Enable debug logging
export SCRAPY_LOG_LEVEL=DEBUG
python run_spiders.py
```

**Problem**: 403 Forbidden errors
```python
# Update user agent and headers in settings.py
USER_AGENT = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'
DEFAULT_REQUEST_HEADERS = {
    'Accept': 'text/html,application/xhtml+xml',
    'Accept-Language': 'zh-CN,zh;q=0.9,en;q=0.8',
    'Referer': 'https://www.google.com'
}

# Add cookies for authenticated platforms
COOKIES_ENABLED = True
```

### Database Connection

**Problem**: MySQL connection timeout
```python
# Increase timeout in .env
MYSQL_CONNECT_TIMEOUT=60

# Check MySQL max_connections
# mysql> SHOW VARIABLES LIKE 'max_connections';

# Verify credentials
from hotsearch_analysis_agent.database import test_connection
test_connection()  # Should print success message
```

### LLM Analysis

**Problem**: Slow response times
```python
# Reduce context window
LLM_CONFIG['max_tokens'] = 2000

# Batch similar queries
analyzer.analyze_batch(topics, batch_size=10)

# Use streaming for long responses
for chunk in analyzer.analyze_stream(query):
    print(chunk, end='', flush=True)
```

**Problem**: Inaccurate sentiment analysis
```python
# Fine-tune prompts in hotsearch_analysis_agent/prompts.py
SENTIMENT_PROMPT = """
分析以下中文文本的情感倾向,考虑:
1. 反讽和隐晦表达
2. 网络用语和表情符号
3. 上下文语境

文本: {text}

返回JSON: {{"sentiment": "positive/negative/neutral", "confidence": 0.0-1.0, "reasoning": "..."}}
"""

# Increase temperature for nuanced analysis
LLM_CONFIG['temperature'] = 0.9
```

### Push Notifications

**Problem**: WeChat webhook not working
```bash
# Test webhook directly
curl -X POST "YOUR_WEBHOOK_URL" \
  -H "Content-Type: application/json" \
  -d '{"msgtype": "text", "text": {"content": "测试消息"}}'

# Check webhook key expiration (Enterprise WeChat webhooks expire)
# Regenerate key in Enterprise WeChat admin panel
```

**Problem**: Email delivery failures
```python
# Test SMTP connection
from hotsearch_analysis_agent.push_service import test_smtp

test_smtp(
    host=os.getenv('SMTP_HOST'),
    port=int(os.getenv('SMTP_PORT')),
    user=os.getenv('SMTP_USER'),
    password=os.getenv('SMTP_PASSWORD')
)

# For Gmail, use app-specific password
# https://support.google.com/accounts/answer/185833
```

### Browser Driver

**Problem**: ChromeDriver version mismatch
```bash
# Check Chrome version
google-chrome --version  # Linux
# Settings → About Chrome (GUI)

# Download exact matching version
# https://chromedriver.chromium.org/downloads

# Or use webdriver-manager
pip install webdriver-manager
```

```python
# Auto-manage driver in code
from selenium import webdriver
from webdriver_manager.chrome import ChromeDriverManager

driver = webdriver.Chrome(ChromeDriverManager().install())
```

## Performance Tips

1. **Optimize Crawl Frequency**: Don't crawl too frequently (respect rate limits)
   ```python
   # In settings.py
   DOWNLOAD_DELAY = 2  # 2 seconds between requests
   CONCURRENT_REQUESTS_PER_DOMAIN = 2
   ```

2. **Database Indexing**: Add indexes on frequently queried columns
   ```sql
   CREATE INDEX idx_platform_timestamp ON hot_search_items(platform, timestamp);
   CREATE INDEX idx_keyword ON hot_search_items(title);
   ```

3. **Cache LLM Results**: Avoid re-analyzing identical content
   ```python
   from functools import lru_cache
   
   @lru_cache(maxsize=1000)
   def analyze_cached(text_hash):
       return analyzer.analyze_sentiment(text)
   ```

4. **Use Async for Push Notifications**:
   ```python
   import asyncio
   
   async def push_all_channels(message):
       await asyncio.gather(
           push_wechat(message),
           push_telegram(message),
           push_email(message)
       )
   ```
