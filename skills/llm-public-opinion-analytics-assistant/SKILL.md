---
name: llm-public-opinion-analytics-assistant
description: LLM-based real-time public opinion monitoring system that aggregates 26 trending lists from 15 platforms with AI-powered sentiment analysis and multi-channel alerts
triggers:
  - set up public opinion monitoring system
  - analyze trending topics across multiple platforms
  - configure hot topic push notifications
  - crawl trending lists from social media
  - perform sentiment analysis on news articles
  - cluster and analyze public opinion data
  - deploy LLM-based opinion analytics
  - integrate pangu model for sentiment analysis
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

This is a comprehensive public opinion monitoring and analysis system that:

- **Aggregates trending data** from 26 trending lists across 15 major Chinese platforms (Weibo, Bilibili, Zhihu, Douyin, etc.)
- **AI-powered analysis** using LLM (Huawei Pangu model recommended) for topic clustering, sentiment analysis, and trend detection
- **Multi-channel alerts** via email, WeChat, Enterprise WeChat, and Telegram
- **Conversational interface** for querying trending topics and analyzing public sentiment
- **Real-time crawling** with keyboard shortcuts to start/stop data collection
- **Video content analysis** - extracts insights even from video-based news

The system separates crawling (`hotsearchcrawler`) from analysis (`hotsearch_analysis_agent`) for scalable deployment.

## Installation

### Prerequisites

**1. Browser Driver Setup**

The system requires ChromeDriver or EdgeDriver for dynamic content scraping:

```bash
# Check your browser version first
google-chrome --version  # or msedge --version

# Download matching driver:
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/microsoft-edge/tools/webdriver/

# Place driver in PATH (Linux/macOS example)
sudo mv chromedriver /usr/local/bin/
sudo chmod +x /usr/local/bin/chromedriver

# Verify installation
chromedriver --version
```

**2. MySQL Database**

```bash
# Install MySQL
sudo apt-get install mysql-server  # Ubuntu/Debian
# or brew install mysql              # macOS

# Create database
mysql -u root -p
```

```sql
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

**3. Python Environment**

```bash
# Clone repository
git clone https://github.com/hmmnxkl/LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant.git
cd LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant

# Create virtual environment
python3 -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Database Initialization

Run the initialization script to create required tables:

```python
# Reference init.py structure - adapt database connection settings
from hotsearch_analysis_agent.database import init_database

init_database(
    host=os.getenv('MYSQL_HOST', 'localhost'),
    port=int(os.getenv('MYSQL_PORT', 3306)),
    user=os.getenv('MYSQL_USER'),
    password=os.getenv('MYSQL_PASSWORD'),
    database=os.getenv('MYSQL_DATABASE', 'hotsearch_db')
)
```

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```bash
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_mysql_user
MYSQL_PASSWORD=your_mysql_password
MYSQL_DATABASE=hotsearch_db

# LLM API Configuration (OpenAI-compatible format)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://your-llm-endpoint.com/v1
LLM_MODEL_NAME=pangu-embedded-7b

# Push Notification Settings
# Email (SMTP)
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
EMAIL_RECIPIENTS=recipient1@example.com,recipient2@example.com

# Enterprise WeChat Robot
WECHAT_ROBOT_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=your_key

# Enterprise WeChat App
WECHAT_CORP_ID=your_corp_id
WECHAT_AGENT_ID=your_agent_id
WECHAT_CORP_SECRET=your_corp_secret

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

### Crawler Configuration

Edit `hotsearchcrawler/settings.py`:

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
    'weibo': 'your_weibo_cookie',  # For accessing login-required content
    'bilibili': 'your_bilibili_cookie'
}

# Crawl interval (seconds)
DOWNLOAD_DELAY = 2
CONCURRENT_REQUESTS = 16
```

## Running the System

### Start the Web Interface

```bash
# Main application entry point
python app.py
```

The web UI will be available at `http://localhost:5000` (default).

### Manual Crawler Testing

```bash
# Test individual platform crawlers
python runspider-test.py --platform weibo
python runspider-test.py --platform bilibili

# Run all crawlers
python run_spiders.py
```

### Test Push Notifications

```bash
# Test configured notification channels
python test_push_task.py
```

## Key Components and Usage

### 1. Data Crawling

The crawler system uses Scrapy to collect trending data:

```python
# Example: Custom spider for a new platform
from hotsearchcrawler.base_spider import BaseHotSearchSpider

class MyPlatformSpider(BaseHotSearchSpider):
    name = 'myplatform'
    start_urls = ['https://myplatform.com/trending']
    
    def parse(self, response):
        for item in response.css('.trending-item'):
            yield {
                'title': item.css('.title::text').get(),
                'url': item.css('a::attr(href)').get(),
                'heat_value': item.css('.heat::text').get(),
                'platform': self.name,
                'crawl_time': datetime.now()
            }
```

### 2. LLM-Powered Analysis

Query trending topics with natural language:

```python
from hotsearch_analysis_agent.agent import OpinionAnalysisAgent

# Initialize agent with LLM backend
agent = OpinionAnalysisAgent(
    api_key=os.getenv('OPENAI_API_KEY'),
    api_base=os.getenv('OPENAI_API_BASE'),
    model=os.getenv('LLM_MODEL_NAME')
)

# Natural language query
response = agent.query("人工智能相关的热点新闻")
print(response)

# Topic clustering
clusters = agent.cluster_topics(min_similarity=0.7)

# Sentiment analysis
sentiment = agent.analyze_sentiment("科技领域")
# Returns: {'positive': 0.65, 'neutral': 0.25, 'negative': 0.10}
```

### 3. Setting Up Push Tasks

Configure automated alerts for trending topics:

```python
from hotsearch_analysis_agent.push_task import PushTaskManager

task_manager = PushTaskManager()

# Create a push task
task_config = {
    'name': 'AI Technology Monitoring',
    'query': '人工智能 OR 大模型',
    'platforms': ['weibo', 'zhihu', 'bilibili'],
    'channels': ['email', 'wechat_robot', 'telegram'],
    'interval': 3600,  # Check every hour
    'threshold': 10000,  # Heat value threshold
    'sentiment_filter': 'all'  # 'positive', 'negative', or 'all'
}

task_manager.create_task(task_config)
task_manager.start_task(task_config['name'])
```

### 4. Multi-Channel Notifications

```python
from hotsearch_analysis_agent.notifiers import (
    EmailNotifier,
    WeChatRobotNotifier,
    TelegramNotifier
)

# Email notification
email_notifier = EmailNotifier(
    smtp_server=os.getenv('SMTP_SERVER'),
    smtp_port=int(os.getenv('SMTP_PORT')),
    username=os.getenv('SMTP_USER'),
    password=os.getenv('SMTP_PASSWORD')
)

report_content = """
## AI Technology Trending Report
**Time**: 2026-05-19 10:00

### Top Trending Topics:
1. GPT-6 200万上下文窗口 (Heat: 2.5M)
2. DeepSeek V4采用华为昇腾 (Heat: 1.8M)
...
"""

email_notifier.send(
    recipients=os.getenv('EMAIL_RECIPIENTS').split(','),
    subject='Daily AI Trending Report',
    content=report_content
)

# WeChat Robot
wechat_notifier = WeChatRobotNotifier(
    webhook_url=os.getenv('WECHAT_ROBOT_WEBHOOK')
)
wechat_notifier.send_markdown(report_content)

# Telegram
telegram_notifier = TelegramNotifier(
    bot_token=os.getenv('TELEGRAM_BOT_TOKEN'),
    chat_id=os.getenv('TELEGRAM_CHAT_ID')
)
telegram_notifier.send(report_content)
```

### 5. Database Queries

```python
from hotsearch_analysis_agent.database import DatabaseManager

db = DatabaseManager()

# Get latest trending topics
trending = db.query("""
    SELECT platform, title, heat_value, url, crawl_time
    FROM trending_data
    WHERE crawl_time > DATE_SUB(NOW(), INTERVAL 1 HOUR)
    ORDER BY heat_value DESC
    LIMIT 50
""")

# Search specific keywords
results = db.search_topics(
    keyword="人工智能",
    platforms=["weibo", "zhihu"],
    start_date="2026-05-01",
    end_date="2026-05-19"
)

# Get topic detail with full content
detail = db.get_topic_detail(topic_id=12345)
# Includes scraped article/video content
```

## Common Patterns

### Pattern 1: Daily Trending Report Generation

```python
from hotsearch_analysis_agent.report import ReportGenerator

generator = ReportGenerator(agent=agent, db=db)

# Generate comprehensive report
report = generator.generate_daily_report(
    topics=["人工智能", "科技前沿", "商业动态"],
    platforms=["weibo", "zhihu", "bilibili", "toutiao"],
    date="2026-05-19"
)

# Auto-push to all configured channels
generator.push_report(report, channels=['email', 'wechat', 'telegram'])
```

### Pattern 2: Real-Time Topic Monitoring

```python
import schedule
import time

def monitor_hot_topics():
    """Monitor and alert on emerging hot topics"""
    recent_topics = db.get_trending(minutes=30)
    
    for topic in recent_topics:
        # Check if topic heat exceeds threshold
        if topic['heat_value'] > 500000:
            # Analyze sentiment
            sentiment = agent.analyze_sentiment(topic['title'])
            
            # Generate alert
            if sentiment['negative'] > 0.6:
                alert = f"⚠️ High-heat negative topic detected:\n{topic['title']}\nHeat: {topic['heat_value']}"
                wechat_notifier.send_text(alert)

# Schedule monitoring
schedule.every(30).minutes.do(monitor_hot_topics)

while True:
    schedule.run_pending()
    time.sleep(60)
```

### Pattern 3: Topic Clustering and Trend Analysis

```python
# Cluster related topics over the past week
topics = db.get_trending_range(days=7, min_heat=100000)

clusters = agent.cluster_topics(
    topics,
    method='semantic',  # or 'keyword'
    min_cluster_size=3
)

for cluster_id, cluster_topics in clusters.items():
    print(f"\n📊 Cluster {cluster_id}:")
    print(f"Main theme: {cluster_topics['theme']}")
    print(f"Topic count: {len(cluster_topics['items'])}")
    
    # Generate cluster summary
    summary = agent.summarize_cluster(cluster_topics)
    print(summary)
```

## Troubleshooting

### ChromeDriver Issues

```bash
# Error: "ChromeDriver version mismatch"
# Solution: Download exact version matching your Chrome
chrome --version
# Download from https://chromedriver.chromium.org/downloads

# Error: "ChromeDriver not found in PATH"
export PATH=$PATH:/path/to/chromedriver/directory
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
        password=os.getenv('MYSQL_PASSWORD'),
        database=os.getenv('MYSQL_DATABASE')
    )
    print("✅ Database connection successful")
    conn.close()
except Exception as e:
    print(f"❌ Connection failed: {e}")
```

### LLM API Issues

```python
# Test LLM connectivity
import openai

openai.api_key = os.getenv('OPENAI_API_KEY')
openai.api_base = os.getenv('OPENAI_API_BASE')

try:
    response = openai.ChatCompletion.create(
        model=os.getenv('LLM_MODEL_NAME'),
        messages=[{"role": "user", "content": "测试连接"}],
        max_tokens=10
    )
    print("✅ LLM API working:", response.choices[0].message.content)
except Exception as e:
    print(f"❌ LLM API error: {e}")
```

### Crawler Not Collecting Data

```bash
# Check crawler logs
tail -f hotsearchcrawler/logs/spider.log

# Test individual spider
scrapy crawl weibo -o test_output.json

# Check for blocked requests (HTTP 403/429)
# Solution: Add delays or use proxies in settings.py
DOWNLOAD_DELAY = 3
CONCURRENT_REQUESTS = 8
```

### Push Notifications Failing

```python
# Test each notification channel individually

# Email
from hotsearch_analysis_agent.notifiers import EmailNotifier
email_notifier = EmailNotifier(...)
email_notifier.test_connection()

# WeChat - verify webhook URL is active
import requests
response = requests.post(
    os.getenv('WECHAT_ROBOT_WEBHOOK'),
    json={"msgtype": "text", "text": {"content": "Test message"}}
)
print(response.status_code, response.text)

# Telegram - verify bot token and chat ID
from telegram import Bot
bot = Bot(token=os.getenv('TELEGRAM_BOT_TOKEN'))
print(bot.get_me())
```

### Memory Issues with Large Datasets

```python
# Use batch processing for large queries
from hotsearch_analysis_agent.database import DatabaseManager

db = DatabaseManager()

# Instead of loading all data at once:
# topics = db.get_all_topics()  # ❌ May cause OOM

# Use pagination:
batch_size = 1000
offset = 0

while True:
    batch = db.get_topics_batch(limit=batch_size, offset=offset)
    if not batch:
        break
    
    # Process batch
    process_topics(batch)
    offset += batch_size
```

## Platform-Specific Notes

**Supported Platforms** (26 trending lists from 15 platforms):
- Weibo, Zhihu, Bilibili, Douyin, Toutiao
- Baidu, 36kr, IT之家, Hacker News
- V2EX, GitHub Trending, Product Hunt
- And more...

Each platform may require specific cookies or headers for full access. Configure in `hotsearchcrawler/settings.py`.
