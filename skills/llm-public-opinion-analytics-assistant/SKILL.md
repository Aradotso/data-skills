---
name: llm-public-opinion-analytics-assistant
description: Chinese public opinion analytics platform integrating 26 trending lists from 15 platforms with LLM-powered sentiment analysis, topic clustering, and multi-channel alert push
triggers:
  - "set up public opinion monitoring system"
  - "analyze Chinese social media trends"
  - "configure hotspot crawler for Weibo and Bilibili"
  - "implement sentiment analysis with LLM"
  - "create trending topic alerts for Telegram"
  - "deploy multi-platform news aggregation"
  - "build opinion analytics dashboard"
  - "extract insights from Chinese video platforms"
---

# LLM Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive Chinese public opinion monitoring platform that aggregates real-time trending data from 26 lists across 15 mainstream platforms (Weibo, Bilibili, Douyin, Zhihu, etc.) and provides LLM-powered analysis including sentiment classification, topic clustering, and automated alerts via email, WeChat Work, and Telegram.

## What This Project Does

- **Multi-Platform Crawling**: Scrapes trending lists from 15 Chinese platforms with Scrapy-based distributed crawlers
- **Deep Content Extraction**: Retrieves full article/video content from detail pages using Selenium
- **LLM Analysis**: Performs sentiment analysis, topic clustering, and trend summarization using large language models (supports Huawei Pangu, OpenAI-compatible APIs)
- **Conversational Interface**: Natural language queries for trending topics, theme searches, and analytical reports
- **Alert System**: Multi-channel push notifications (Enterprise WeChat, Telegram, email) with scheduled analysis reports
- **Shortcut Controls**: Keyboard shortcuts to start/stop crawlers and trigger analysis tasks

## Installation

### Prerequisites

**1. Browser Driver Setup**

Download and configure browser drivers for content extraction:

```bash
# For Chrome
wget https://chromedriver.chromium.org/downloads
# Match your Chrome version (check chrome://version)

# For Edge
wget https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Place driver in PATH or project directory
# Linux/macOS example:
sudo mv chromedriver /usr/local/bin/
chmod +x /usr/local/bin/chromedriver

# Verify installation
chromedriver --version
```

**2. MySQL Database**

```bash
# Install MySQL 8.0+
sudo apt-get install mysql-server  # Ubuntu/Debian
# OR
brew install mysql  # macOS

# Create database
mysql -u root -p
CREATE DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
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

Initialize database schema using the reference script:

```python
# init.py - Database setup reference
import pymysql

connection = pymysql.connect(
    host='localhost',
    user='root',
    password='your_password',
    database='hotsearch',
    charset='utf8mb4'
)

with connection.cursor() as cursor:
    # Create trending lists table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS trending_topics (
            id INT AUTO_INCREMENT PRIMARY KEY,
            platform VARCHAR(50) NOT NULL,
            title VARCHAR(500) NOT NULL,
            url VARCHAR(1000),
            rank INT,
            hot_value VARCHAR(100),
            detail_content TEXT,
            crawl_time DATETIME,
            INDEX idx_platform (platform),
            INDEX idx_crawl_time (crawl_time)
        )
    """)
    
    # Create analysis results table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS analysis_results (
            id INT AUTO_INCREMENT PRIMARY KEY,
            query_text VARCHAR(500),
            analysis_type VARCHAR(50),
            result_content TEXT,
            sentiment_score FLOAT,
            created_at DATETIME,
            INDEX idx_query (query_text),
            INDEX idx_type (analysis_type)
        )
    """)
    
    connection.commit()
```

## Configuration

### Environment Variables

Create `.env` file in project root:

```bash
# Database Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=your_secure_password
MYSQL_DATABASE=hotsearch

# LLM API Configuration (OpenAI-compatible format)
OPENAI_API_KEY=your_api_key_here
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-4

# OR use Huawei Pangu Model (local deployment)
# PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model
# USE_LOCAL_MODEL=true

# Push Notification Channels
# Enterprise WeChat Robot
WECHAT_ROBOT_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email (SMTP)
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
ALERT_EMAIL_TO=recipient@example.com

# Browser Driver Configuration
CHROME_DRIVER_PATH=/usr/local/bin/chromedriver
EDGE_DRIVER_PATH=/usr/local/bin/msedgedriver
```

### Crawler Settings

Configure `hotsearchcrawler/settings.py`:

```python
# MySQL pipeline settings
MYSQL_HOST = 'localhost'
MYSQL_DATABASE = 'hotsearch'
MYSQL_USER = 'root'
MYSQL_PASSWORD = 'your_password'

# Crawler behavior
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1  # Politeness delay between requests
ROBOTSTXT_OBEY = False  # Many Chinese platforms don't have robots.txt

# Optional: Platform-specific cookies for authenticated content
COOKIES = {
    'weibo': 'SUB=your_cookie_value',
    'bilibili': 'SESSDATA=your_session_data'
}

# User agent rotation
USER_AGENT = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'
```

## Starting the System

### Launch Analysis System

```bash
# Start the main application
python app.py

# Access web interface at http://localhost:5000
```

### Start Crawlers

**Via Web Interface:**
- Use keyboard shortcut (configured in UI) to start/stop crawlers

**Via Command Line:**

```bash
# Start all platform crawlers
python run_spiders.py

# Test specific platform
cd hotsearchcrawler
scrapy crawl weibo_hot  # Weibo trending
scrapy crawl bilibili_hot  # Bilibili trending
scrapy crawl douyin_hot  # Douyin trending
scrapy crawl zhihu_hot  # Zhihu trending
```

## Key Usage Patterns

### Conversational Query Interface

```python
# Example API call to analysis endpoint
import requests

response = requests.post('http://localhost:5000/api/query', json={
    'query': '最近关于人工智能的热点有哪些？',  # "What AI-related trending topics recently?"
    'analysis_type': 'theme_search'
})

result = response.json()
print(result['topics'])  # List of AI-related trending topics
print(result['analysis'])  # LLM-generated summary
```

### Sentiment Analysis

```python
# Analyze sentiment for a specific topic
response = requests.post('http://localhost:5000/api/analyze', json={
    'query': 'GPT-6模型发布',
    'analysis_type': 'sentiment'
})

sentiment = response.json()
# Returns: {
#   'sentiment': 'positive',
#   'score': 0.85,
#   'reasoning': '大多数评论表达了期待和兴奋...'
# }
```

### Topic Clustering

```python
# Cluster related topics
response = requests.post('http://localhost:5000/api/analyze', json={
    'query': '科技创新',
    'analysis_type': 'clustering',
    'time_range': '7d'  # Last 7 days
})

clusters = response.json()['clusters']
# Returns grouped topics:
# {
#   'AI模型发展': [topic1, topic2, ...],
#   '硬件生态': [topic3, topic4, ...],
#   '商业应用': [topic5, topic6, ...]
# }
```

### Scheduled Alert Tasks

```python
# test_push_task.py - Configure automated reports
from hotsearch_analysis_agent.push_service import PushService

push_service = PushService()

# Create scheduled alert
push_service.create_task({
    'name': 'AI技术日报',
    'query': '人工智能 OR 大模型 OR AI',
    'schedule': 'daily',  # daily, weekly, hourly
    'time': '09:00',
    'channels': ['wechat', 'telegram', 'email'],
    'analysis_types': ['sentiment', 'clustering', 'summary']
})

# Test immediate push
push_service.send_report(
    title='人工智能与前沿科技热点分析',
    content=analysis_result,
    channels=['telegram']
)
```

### Direct Crawler Usage

```python
# Custom crawler for specific platform
from hotsearchcrawler.spiders.weibo_spider import WeiboHotSpider
from scrapy.crawler import CrawlerProcess

process = CrawlerProcess({
    'USER_AGENT': 'Mozilla/5.0...',
    'MYSQL_HOST': 'localhost',
    'MYSQL_DATABASE': 'hotsearch'
})

process.crawl(WeiboHotSpider)
process.start()
```

### LLM Integration (Pangu Model)

```python
# Using Huawei Pangu model locally
from hotsearch_analysis_agent.llm_engine import PanguAnalyzer

analyzer = PanguAnalyzer(
    model_path='/path/to/openpangu-embedded-7b-model'
)

# Analyze topic
result = analyzer.analyze_topic(
    topic='DeepSeek V4采用华为算力',
    context=related_articles,
    task='sentiment_and_summary'
)

print(result['sentiment'])  # positive/neutral/negative
print(result['summary'])  # Chinese summary
print(result['key_points'])  # Extracted insights
```

## Common Workflows

### Daily Monitoring Setup

```python
# 1. Start crawlers (runs continuously)
# Run in background: nohup python run_spiders.py &

# 2. Configure daily report
from hotsearch_analysis_agent.scheduler import AnalysisScheduler

scheduler = AnalysisScheduler()
scheduler.add_daily_task(
    query='科技 OR 互联网 OR AI',
    time='08:00',
    recipients=['wechat_group', 'email'],
    report_format='detailed'  # detailed or summary
)

# 3. Start scheduler
scheduler.run()
```

### Real-Time Alert for Keywords

```python
# Monitor specific keywords with immediate alerts
from hotsearch_analysis_agent.realtime_monitor import RealtimeMonitor

monitor = RealtimeMonitor()
monitor.add_keyword_alert(
    keywords=['华为', '盘古', 'DeepSeek'],
    threshold_rank=10,  # Alert if keyword appears in top 10
    channels=['telegram'],
    callback=custom_alert_handler
)

monitor.start()
```

### Export Analysis Report

```python
# Generate and export comprehensive report
from hotsearch_analysis_agent.report_generator import ReportGenerator

generator = ReportGenerator()
report = generator.generate(
    query='人工智能',
    date_range=('2026-04-01', '2026-04-07'),
    include_sentiment=True,
    include_clusters=True,
    format='markdown'  # markdown, html, or json
)

# Save to file
with open('ai_report_20260407.md', 'w', encoding='utf-8') as f:
    f.write(report)

# Or push directly
generator.push_report(report, channels=['email', 'wechat'])
```

## Platform Coverage

Supported platforms (26 trending lists from 15 platforms):

- **Social Media**: Weibo, Douyin, Kuaishou
- **Video**: Bilibili, Xigua Video
- **News**: Toutiao, NetEase, Sina
- **Q&A/Forums**: Zhihu, Tieba
- **Tech**: 36Kr, iFeng Tech, IT Home
- **Finance**: Caijing, Eastmoney
- **Others**: Baidu Hot Search, Sogou

Each platform may have multiple lists (综合榜, 热搜榜, 视频榜, etc.)

## Troubleshooting

### Crawler Fails to Extract Content

```python
# Check browser driver compatibility
from selenium import webdriver
from selenium.webdriver.chrome.service import Service

service = Service('/usr/local/bin/chromedriver')
driver = webdriver.Chrome(service=service)
driver.get('https://www.bilibili.com')
print(driver.page_source[:500])  # Should show HTML

# If fails, update driver to match browser version
```

### Database Connection Issues

```python
# Test MySQL connection
import pymysql

try:
    conn = pymysql.connect(
        host='localhost',
        user='root',
        password='your_password',
        database='hotsearch'
    )
    print("Connected successfully")
except pymysql.Error as e:
    print(f"Error: {e}")
    # Common fixes:
    # 1. Check MySQL service: sudo systemctl status mysql
    # 2. Verify credentials in .env
    # 3. Grant permissions: GRANT ALL ON hotsearch.* TO 'root'@'localhost';
```

### LLM API Timeout

```python
# Increase timeout for large context
import openai
openai.api_key = os.getenv('OPENAI_API_KEY')

response = openai.ChatCompletion.create(
    model='gpt-4',
    messages=[{'role': 'user', 'content': long_text}],
    timeout=120  # Increase from default 30s
)
```

### Push Notification Not Sending

```bash
# Test Enterprise WeChat webhook
curl -X POST 'https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY' \
  -H 'Content-Type: application/json' \
  -d '{"msgtype":"text","text":{"content":"测试消息"}}'

# Test Telegram bot
curl "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage?chat_id=${TELEGRAM_CHAT_ID}&text=Test"
```

### Memory Issues with Large Datasets

```python
# Use batch processing for large queries
from hotsearch_analysis_agent.db_utils import query_topics_batch

for batch in query_topics_batch(
    query='人工智能',
    batch_size=1000,
    date_range=('2026-01-01', '2026-04-07')
):
    analysis = analyzer.process_batch(batch)
    save_results(analysis)
```

## Advanced Configuration

### Custom Platform Spider

```python
# hotsearchcrawler/spiders/custom_spider.py
import scrapy
from hotsearchcrawler.items import TrendingItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://custom-platform.com/trending']
    
    def parse(self, response):
        for item in response.css('.trending-item'):
            yield TrendingItem(
                platform='CustomPlatform',
                title=item.css('.title::text').get(),
                url=item.css('a::attr(href)').get(),
                rank=item.css('.rank::text').get(),
                hot_value=item.css('.hot::text').get()
            )
```

### Multi-Language Support Extension

```python
# Add translation layer for non-Chinese queries
from hotsearch_analysis_agent.translator import QueryTranslator

translator = QueryTranslator()
en_query = "artificial intelligence trends"
zh_query = translator.translate(en_query, target='zh')
# Returns: "人工智能趋势"

results = analyzer.query(zh_query)
translated_results = translator.translate_results(results, target='en')
```

This skill enables AI agents to help developers deploy and utilize a comprehensive Chinese public opinion monitoring system with LLM-powered analytics and multi-channel alerting.
