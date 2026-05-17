---
name: llm-public-opinion-analytics-assistant
description: Use this Chinese LLM-powered public opinion monitoring system to collect, analyze and push trending topics from 15+ platforms with sentiment analysis
triggers:
  - set up public opinion monitoring system
  - analyze trending topics from Chinese social media
  - create sentiment analysis dashboard for news
  - configure hot topic crawler with notifications
  - build multi-platform trend aggregator
  - set up automated trend push notifications
  - monitor weibo douyin bilibili trending topics
  - create Chinese social media analytics assistant
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is an intelligent public opinion monitoring system that integrates real-time data from **26 trending lists across 15 major Chinese platforms** (Weibo, Douyin, Bilibili, Zhihu, etc.) with LLM analysis capabilities. It provides conversational trending topic queries, topic clustering, sentiment analysis, and multi-channel push notifications (WeChat, Telegram, Email).

## What This Project Does

- **Multi-Platform Data Collection**: Crawls trending topics from 15 Chinese platforms including social media, video platforms, and news sites
- **LLM-Powered Analysis**: Uses large language models (supports Huawei Pangu, OpenAI-compatible APIs) for topic clustering and sentiment analysis
- **Conversational Interface**: Chat-based UI for querying trends, searching specific topics, and analyzing sentiment
- **Content Deep Dive**: Extracts full content from news detail pages (including video transcripts)
- **Multi-Channel Notifications**: Push reports via Enterprise WeChat, Telegram, or Email
- **Separated Architecture**: Crawler cluster (`hotsearchcrawler`) runs independently from analysis system (`hotsearch_analysis_agent`)

## Installation

### Prerequisites

1. **Browser Driver Setup** (required for detail page scraping):

```bash
# For Chrome - download ChromeDriver matching your Chrome version
# https://chromedriver.chromium.org/
# For Edge - download EdgeDriver
# https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Place driver in PATH or browser installation directory
# Verify installation:
chromedriver --version
```

2. **MySQL Database**:

```sql
-- Create database
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Reference init.py for table schemas (main tables):
-- - trending_topics (stores crawled trending items)
-- - analysis_results (stores LLM analysis results)
-- - push_tasks (stores notification task configs)
```

3. **Python Environment**:

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Configuration

**1. Crawler Configuration** (`hotsearchcrawler/settings.py`):

```python
# MySQL connection
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_user'
MYSQL_PASSWORD = 'your_password'
MYSQL_DATABASE = 'hotsearch_db'

# Optional: Platform-specific cookies (for authenticated content)
# COOKIES = {
#     'weibo': 'your_cookie_string',
#     'douyin': 'your_cookie_string'
# }
```

**2. Analysis System Configuration** (`.env` file in project root):

```env
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key_here
OPENAI_API_BASE=https://api.openai.com/v1
OPENAI_MODEL=gpt-4

# Or use Huawei Pangu (local deployment)
# PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model
# USE_PANGU=true

# Push Notification Configs (optional)
# Enterprise WeChat Robot
WECHAT_WEBHOOK_URL=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email SMTP
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
```

## Key Commands and Usage

### Starting the Crawler

```bash
# From project root
python run_spiders.py

# Or test individual spider
cd hotsearchcrawler
scrapy crawl weibo_hot  # Crawl Weibo trending
scrapy crawl douyin_hot # Crawl Douyin trending
scrapy crawl bilibili_hot # Crawl Bilibili trending
```

**Crawler Architecture**:
- Each platform has a dedicated spider in `hotsearchcrawler/spiders/`
- Spiders store data directly to MySQL
- Can be controlled via frontend UI with keyboard shortcuts

### Starting the Analysis System

```bash
# Start web interface and API
python app.py

# Default runs on http://localhost:5000
```

### Testing Push Notifications

```bash
# Test push task configuration
python test_push_task.py
```

## API and Code Examples

### Using the Analysis Agent (Python)

```python
from hotsearch_analysis_agent.agent import OpinionAnalysisAgent
from hotsearch_analysis_agent.database import get_db_connection

# Initialize agent
agent = OpinionAnalysisAgent(
    model="gpt-4",  # or use Pangu model
    api_key=os.getenv("OPENAI_API_KEY")
)

# Query trending topics
db = get_db_connection()
cursor = db.cursor(dictionary=True)
cursor.execute("""
    SELECT title, platform, hot_value, url 
    FROM trending_topics 
    WHERE created_at > NOW() - INTERVAL 1 HOUR
    ORDER BY hot_value DESC 
    LIMIT 20
""")
trends = cursor.fetchall()

# Perform topic clustering
query = "人工智能与前沿科技"
clustered_topics = agent.cluster_topics(trends, query)

# Sentiment analysis
for topic in clustered_topics:
    sentiment = agent.analyze_sentiment(topic['title'])
    topic['sentiment'] = sentiment
    print(f"{topic['title']}: {sentiment}")

# Generate analysis report
report = agent.generate_report(
    topic_clusters=clustered_topics,
    query=query,
    include_details=True
)
print(report)
```

### Direct Database Queries

```python
import pymysql
from datetime import datetime, timedelta

# Connect to database
conn = pymysql.connect(
    host='localhost',
    user='your_user',
    password='your_password',
    database='hotsearch_db',
    charset='utf8mb4'
)

# Get top trends from last 24 hours
with conn.cursor() as cursor:
    sql = """
        SELECT platform, title, hot_value, url, created_at
        FROM trending_topics
        WHERE created_at > %s
        ORDER BY hot_value DESC
        LIMIT 50
    """
    cursor.execute(sql, (datetime.now() - timedelta(days=1),))
    results = cursor.fetchall()
    
    for row in results:
        print(f"[{row[0]}] {row[1]} - 热度: {row[2]}")

# Search specific topics
with conn.cursor() as cursor:
    keyword = "人工智能"
    sql = """
        SELECT platform, title, url
        FROM trending_topics
        WHERE title LIKE %s
        ORDER BY created_at DESC
        LIMIT 20
    """
    cursor.execute(sql, (f"%{keyword}%",))
    results = cursor.fetchall()
```

### Setting Up Push Tasks

```python
from hotsearch_analysis_agent.push_service import PushTaskManager

# Initialize push task manager
manager = PushTaskManager()

# Create a push task
task = manager.create_task(
    name="AI科技热点日报",
    query_keywords=["人工智能", "大模型", "芯片"],
    schedule="0 9 * * *",  # Daily at 9 AM (cron format)
    channels=["wechat", "telegram"],
    min_hot_value=5000,  # Minimum hotness threshold
    sentiment_filter=None  # Options: "positive", "negative", None
)

# Manual trigger
manager.execute_task(task_id=task['id'])

# List all tasks
tasks = manager.list_tasks()
for t in tasks:
    print(f"{t['name']}: {t['schedule']} -> {t['channels']}")
```

### Custom Crawler Integration

```python
# hotsearchcrawler/spiders/custom_platform.py
import scrapy
from hotsearchcrawler.items import TrendingItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://example.com/trending']
    
    def parse(self, response):
        for trend in response.css('.trend-item'):
            item = TrendingItem()
            item['platform'] = 'CustomPlatform'
            item['title'] = trend.css('.title::text').get()
            item['url'] = trend.css('a::attr(href)').get()
            item['hot_value'] = trend.css('.hot-value::text').get()
            item['rank'] = trend.css('.rank::text').get()
            
            # Optional: Crawl detail page for full content
            detail_url = item['url']
            yield response.follow(
                detail_url,
                callback=self.parse_detail,
                meta={'item': item}
            )
    
    def parse_detail(self, response):
        item = response.meta['item']
        item['content'] = response.css('.article-content::text').getall()
        item['author'] = response.css('.author::text').get()
        yield item
```

## Common Patterns

### Pattern 1: Scheduled Analysis with Push

```python
from apscheduler.schedulers.blocking import BlockingScheduler
from hotsearch_analysis_agent.agent import OpinionAnalysisAgent
from hotsearch_analysis_agent.push_service import push_to_wechat

def daily_analysis_job():
    agent = OpinionAnalysisAgent()
    
    # Get today's top trends
    trends = agent.get_recent_trends(hours=24, limit=100)
    
    # Cluster by topics
    clusters = agent.cluster_topics(trends, query="今日热点")
    
    # Generate report
    report = agent.generate_report(clusters, format="markdown")
    
    # Push to WeChat
    push_to_wechat(
        webhook_url=os.getenv("WECHAT_WEBHOOK_URL"),
        content=report,
        msg_type="markdown"
    )

# Schedule daily at 8 AM
scheduler = BlockingScheduler()
scheduler.add_job(daily_analysis_job, 'cron', hour=8, minute=0)
scheduler.start()
```

### Pattern 2: Real-Time Sentiment Monitoring

```python
import time
from hotsearch_analysis_agent.agent import OpinionAnalysisAgent

agent = OpinionAnalysisAgent()
keywords = ["公司名称", "产品名称"]
alert_threshold = -0.5  # Negative sentiment threshold

while True:
    for keyword in keywords:
        trends = agent.search_topics(keyword, hours=1)
        
        for trend in trends:
            sentiment_score = agent.analyze_sentiment(trend['title'])
            
            if sentiment_score < alert_threshold:
                # Alert: negative sentiment detected
                alert_message = f"⚠️ 负面舆情预警\n主题: {trend['title']}\n平台: {trend['platform']}\n情感分数: {sentiment_score}\n链接: {trend['url']}"
                
                # Send immediate notification
                send_telegram_alert(alert_message)
    
    time.sleep(300)  # Check every 5 minutes
```

### Pattern 3: Multi-Platform Comparison

```python
from collections import defaultdict
from hotsearch_analysis_agent.database import get_db_connection

def compare_platform_coverage(keyword, hours=24):
    db = get_db_connection()
    cursor = db.cursor(dictionary=True)
    
    sql = """
        SELECT platform, COUNT(*) as count, AVG(hot_value) as avg_hot
        FROM trending_topics
        WHERE title LIKE %s 
        AND created_at > NOW() - INTERVAL %s HOUR
        GROUP BY platform
        ORDER BY count DESC
    """
    
    cursor.execute(sql, (f"%{keyword}%", hours))
    results = cursor.fetchall()
    
    print(f"\n话题 '{keyword}' 在各平台的覆盖情况:")
    for row in results:
        print(f"  {row['platform']}: {row['count']}条, 平均热度: {row['avg_hot']:.0f}")
    
    return results

# Usage
compare_platform_coverage("春节", hours=24)
```

## Troubleshooting

### Issue: ChromeDriver version mismatch

```bash
# Error: "This version of ChromeDriver only supports Chrome version XX"
# Solution: Download matching version
# Check Chrome version: chrome://version
# Download from: https://chromedriver.chromium.org/downloads
```

### Issue: MySQL connection errors

```python
# Error: "Access denied for user"
# Solution: Check credentials and grant permissions

# Connect to MySQL as root
# GRANT ALL PRIVILEGES ON hotsearch_db.* TO 'your_user'@'localhost';
# FLUSH PRIVILEGES;

# Verify connection
import pymysql
try:
    conn = pymysql.connect(
        host='localhost',
        user='your_user',
        password='your_password',
        database='hotsearch_db'
    )
    print("Connection successful!")
except Exception as e:
    print(f"Connection failed: {e}")
```

### Issue: Crawler returns empty results

```python
# Debug spider output
# cd hotsearchcrawler
# scrapy crawl weibo_hot -o output.json

# Check if platform changed HTML structure
# Update selectors in spider file:
# hotsearchcrawler/spiders/weibo_hot.py

# Common fix: Update CSS selectors
def parse(self, response):
    # Old selector (may be outdated)
    # trends = response.css('.trend-item')
    
    # Use scrapy shell to test new selectors
    # scrapy shell 'https://weibo.com/hot/search'
    # response.css('new-selector')
    pass
```

### Issue: LLM analysis returns poor results

```python
# Solution 1: Improve prompt engineering
agent = OpinionAnalysisAgent()
agent.set_system_prompt("""
你是一个专业的舆情分析专家。
分析时注意:
1. 识别情感倾向的细微差别
2. 考虑中文的反讽和隐喻表达
3. 按话题聚类时注重内容相关性
4. 输出格式要清晰易读
""")

# Solution 2: Use local Pangu model for Chinese text
# Download from: https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model
# Update .env:
# USE_PANGU=true
# PANGU_MODEL_PATH=/path/to/model
```

### Issue: Push notifications not working

```python
# Test each channel independently
from hotsearch_analysis_agent.push_service import (
    test_wechat_push,
    test_telegram_push,
    test_email_push
)

# Test WeChat
test_wechat_push(
    webhook_url=os.getenv("WECHAT_WEBHOOK_URL"),
    test_message="测试消息"
)

# Test Telegram
test_telegram_push(
    bot_token=os.getenv("TELEGRAM_BOT_TOKEN"),
    chat_id=os.getenv("TELEGRAM_CHAT_ID")
)

# Test Email
test_email_push(
    smtp_host=os.getenv("SMTP_HOST"),
    smtp_user=os.getenv("SMTP_USER"),
    smtp_password=os.getenv("SMTP_PASSWORD"),
    recipient="test@example.com"
)
```

## Additional Resources

- **Huawei Pangu Model**: https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model
- **Scrapy Documentation**: https://docs.scrapy.org/
- **MySQL Python Connector**: https://dev.mysql.com/doc/connector-python/en/
- **APScheduler**: https://apscheduler.readthedocs.io/ (for task scheduling)
