---
name: llm-public-opinion-analytics-assistant
description: A multi-platform hot topic monitoring and sentiment analysis system combining 26 real-time data sources with LLM-powered analytics, featuring automated crawling, clustering, and multi-channel alerting.
triggers:
  - Set up public opinion monitoring system
  - Configure hot topic crawler with sentiment analysis
  - Implement multi-platform news aggregation and alerts
  - Deploy LLM-based opinion analytics assistant
  - Create automated news sentiment monitoring
  - Build real-time hot search tracking system
  - Configure Weibo Douyin hot topic analysis
  - Set up enterprise WeChat news alerts with clustering
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is an intelligent public opinion monitoring system that aggregates real-time data from **15 mainstream platforms** across **26 different rankings/charts**, combining web scraping with LLM-powered analysis. It provides conversational hot topic queries, theme-based searches, topic clustering, sentiment analysis, and multi-channel push notifications (Email, WeChat, Enterprise WeChat, Telegram).

**Key capabilities:**
- Real-time crawling of hot topics from Weibo, Douyin, Bilibili, Zhihu, etc.
- LLM-powered content analysis (including video content extraction)
- Natural language query interface for topic research
- Automated sentiment and clustering analysis
- Multi-channel alert system with scheduled reports
- Keyboard shortcut control for crawler management

## Installation

### Prerequisites

**1. Browser Driver Setup**

Download the appropriate WebDriver for your browser:
- **Chrome**: https://chromedriver.chromium.org/
- **Edge**: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

Match the driver version to your browser version. Place the driver in your system PATH or project directory.

```bash
# Verify driver installation
chromedriver --version
# or
msedgedriver --version
```

**2. Python Environment**

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # Linux/Mac
# or
venv\Scripts\activate  # Windows

# Install dependencies
pip install -r requirements.txt
```

**3. Database Setup**

Install MySQL and create the database:

```sql
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Create main tables (refer to init.py for complete schema)
CREATE TABLE hot_topics (
    id INT AUTO_INCREMENT PRIMARY KEY,
    platform VARCHAR(50) NOT NULL,
    title VARCHAR(500) NOT NULL,
    url TEXT,
    rank INT,
    heat VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_platform (platform),
    INDEX idx_created_at (created_at)
);

CREATE TABLE analysis_results (
    id INT AUTO_INCREMENT PRIMARY KEY,
    query TEXT NOT NULL,
    result LONGTEXT,
    sentiment VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE push_tasks (
    id INT AUTO_INCREMENT PRIMARY KEY,
    task_name VARCHAR(200) NOT NULL,
    query TEXT NOT NULL,
    push_channels JSON,
    schedule_cron VARCHAR(100),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Configuration

**1. Environment Variables (`.env`)**

```bash
# Database Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_mysql_user
MYSQL_PASSWORD=your_mysql_password
MYSQL_DATABASE=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_openai_api_key
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Or use Huawei Pangu model (local deployment)
USE_PANGU_MODEL=false
PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model

# Push Notification Channels
# Email (SMTP)
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_email_app_password
EMAIL_RECIPIENTS=recipient1@example.com,recipient2@example.com

# Enterprise WeChat Robot
WECHAT_ROBOT_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Enterprise WeChat Application
WECHAT_CORP_ID=your_corp_id
WECHAT_AGENT_ID=your_agent_id
WECHAT_APP_SECRET=your_app_secret

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

**2. Crawler Settings (`hotsearchcrawler/settings.py`)**

```python
# MySQL connection for crawler
MYSQL_CONFIG = {
    'host': 'localhost',
    'port': 3306,
    'user': 'your_mysql_user',
    'password': 'your_mysql_password',
    'database': 'hotsearch_db',
    'charset': 'utf8mb4'
}

# Optional: Platform-specific cookies
COOKIES = {
    'weibo': 'your_weibo_cookies',
    'bilibili': 'your_bilibili_cookies'
}

# Crawler concurrency
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
```

## Core Components

### 1. Starting the System

```bash
# Start main application (includes web interface)
python app.py

# The web interface will be available at:
# http://localhost:5000
```

### 2. Running Crawlers

**Via Web Interface:**
- Access the dashboard and use keyboard shortcuts to control crawlers
- Crawlers run independently from the analysis system

**Manual Testing:**

```bash
# Test individual crawler
python runspider-test.py

# Run all crawlers
python run_spiders.py
```

**Crawler Control Example:**

```python
from hotsearchcrawler.crawler_manager import CrawlerManager

manager = CrawlerManager()

# Start crawling all platforms
manager.start_all()

# Start specific platform
manager.start_crawler('weibo')
manager.start_crawler('douyin')

# Stop crawlers
manager.stop_all()

# Check status
status = manager.get_status()
print(f"Active crawlers: {status['active']}")
```

### 3. Analysis System API

**Query Hot Topics:**

```python
from hotsearch_analysis_agent.analyzer import HotSearchAnalyzer

analyzer = HotSearchAnalyzer()

# Natural language query
result = analyzer.query("最近关于人工智能的热点有哪些?")
print(result['answer'])
print(result['sentiment'])  # positive/negative/neutral

# Query specific platform
result = analyzer.query("微博上今天最热的话题", platform="weibo")

# Topic clustering
clusters = analyzer.cluster_topics(
    query="科技新闻",
    time_range="24h",
    min_cluster_size=3
)

for cluster in clusters:
    print(f"Cluster: {cluster['theme']}")
    print(f"Topics: {cluster['topics']}")
    print(f"Sentiment: {cluster['avg_sentiment']}")
```

**Sentiment Analysis:**

```python
from hotsearch_analysis_agent.sentiment import SentimentAnalyzer

sentiment_analyzer = SentimentAnalyzer()

# Analyze single text
result = sentiment_analyzer.analyze("这个产品真是太好用了！")
print(result)  # {'sentiment': 'positive', 'score': 0.95}

# Batch analysis
topics = [
    "某公司产品质量问题频发",
    "新政策获得民众广泛支持",
    "股市今日震荡调整"
]
results = sentiment_analyzer.batch_analyze(topics)
```

### 4. Push Notification System

**Create Push Task:**

```python
from hotsearch_analysis_agent.push_manager import PushManager

push_mgr = PushManager()

# Create scheduled task
task = push_mgr.create_task(
    task_name="AI Tech Daily Report",
    query="人工智能和前沿科技的热点",
    channels=["email", "wechat_robot", "telegram"],
    schedule_cron="0 8 * * *",  # Daily at 8 AM
    analysis_options={
        "include_clustering": True,
        "include_sentiment": True,
        "min_heat_threshold": 10000
    }
)

# Test push immediately
push_mgr.test_push(task_id=task['id'])

# List all tasks
tasks = push_mgr.list_tasks(active_only=True)

# Disable task
push_mgr.toggle_task(task_id=1, active=False)
```

**Manual Push Example:**

```python
from hotsearch_analysis_agent.notifiers import (
    EmailNotifier,
    WeChatRobotNotifier,
    TelegramNotifier
)

# Email notification
email_notifier = EmailNotifier()
email_notifier.send(
    subject="热点分析报告 - 2024-01-15",
    content=analysis_result['formatted_report'],
    recipients=["analyst@company.com"]
)

# Enterprise WeChat Robot
wechat_notifier = WeChatRobotNotifier()
wechat_notifier.send_markdown(
    title="【舆情快报】AI领域热点",
    content=analysis_result['markdown_report']
)

# Telegram
telegram_notifier = TelegramNotifier()
telegram_notifier.send_message(
    text=analysis_result['summary'],
    parse_mode="Markdown"
)
```

### 5. Working with Crawled Data

**Query Database:**

```python
from hotsearch_analysis_agent.data_manager import DataManager

dm = DataManager()

# Get latest hot topics
topics = dm.get_hot_topics(
    platform="weibo",
    limit=50,
    time_range="6h"
)

# Get topic details with content
topic_detail = dm.get_topic_detail(
    topic_id=123,
    include_content=True  # Fetches full article/video content
)

# Search by keyword
results = dm.search_topics(
    keyword="人工智能",
    platforms=["weibo", "zhihu", "bilibili"],
    date_from="2024-01-01",
    date_to="2024-01-15"
)

# Get trending themes
themes = dm.get_trending_themes(
    time_window="7d",
    min_occurrence=5
)
```

### 6. Advanced Analysis Patterns

**Multi-Platform Comparative Analysis:**

```python
from hotsearch_analysis_agent.analyzer import HotSearchAnalyzer

analyzer = HotSearchAnalyzer()

# Compare same topic across platforms
comparison = analyzer.compare_platforms(
    query="新能源汽车",
    platforms=["weibo", "douyin", "zhihu"],
    analysis_type="sentiment"
)

for platform, data in comparison.items():
    print(f"{platform}: {data['sentiment']} (热度: {data['total_heat']})")
```

**Topic Evolution Tracking:**

```python
# Track how a topic evolves over time
evolution = analyzer.track_topic_evolution(
    keywords=["GPT-5", "大模型"],
    time_range="30d",
    interval="1d"
)

for day in evolution:
    print(f"{day['date']}: {day['mention_count']} mentions, "
          f"sentiment={day['avg_sentiment']}")
```

**Custom Report Generation:**

```python
from hotsearch_analysis_agent.report_generator import ReportGenerator

report_gen = ReportGenerator()

report = report_gen.generate(
    query="区块链技术应用",
    template="detailed",  # or "summary", "executive"
    format="markdown",    # or "html", "pdf"
    include_sections=[
        "overview",
        "key_findings",
        "sentiment_analysis",
        "topic_clusters",
        "source_distribution",
        "recommendations"
    ]
)

# Save report
report_gen.save(report, "blockchain_analysis_2024.md")
```

## Common Usage Patterns

### Pattern 1: Real-time Monitoring Dashboard

```python
import time
from hotsearch_analysis_agent.monitor import RealtimeMonitor

monitor = RealtimeMonitor()

# Start monitoring with auto-refresh
monitor.start(
    queries=["产品召回", "数据泄露", "安全漏洞"],
    refresh_interval=300,  # 5 minutes
    alert_threshold={
        "heat": 50000,
        "growth_rate": 0.5,  # 50% increase
        "sentiment": "negative"
    },
    on_alert=lambda event: push_mgr.send_alert(event)
)

# Dashboard will continuously update and trigger alerts
```

### Pattern 2: Scheduled Daily Reports

```python
from apscheduler.schedulers.blocking import BlockingScheduler

scheduler = BlockingScheduler()

@scheduler.scheduled_job('cron', hour=8, minute=0)
def daily_tech_report():
    result = analyzer.query(
        "科技领域的重要新闻和热点",
        analysis_depth="comprehensive"
    )
    
    report = report_gen.generate(
        query="Tech Daily Digest",
        content=result,
        template="executive"
    )
    
    email_notifier.send(
        subject=f"Tech Digest - {datetime.now().strftime('%Y-%m-%d')}",
        content=report,
        recipients=["team@company.com"]
    )

scheduler.start()
```

### Pattern 3: Crisis Detection

```python
from hotsearch_analysis_agent.crisis_detector import CrisisDetector

detector = CrisisDetector()

# Monitor for negative sentiment spikes
crisis_events = detector.detect(
    keywords=["公司名称", "品牌名称"],
    platforms=["weibo", "zhihu", "douyin"],
    detection_rules={
        "sentiment_threshold": -0.6,
        "volume_spike": 3.0,  # 3x normal volume
        "urgency_keywords": ["召回", "诉讼", "事故", "质量问题"]
    }
)

if crisis_events:
    for event in crisis_events:
        # Immediate notification
        wechat_notifier.send_alert(
            title="⚠️ 舆情预警",
            content=f"检测到负面舆情激增:\n{event['summary']}",
            mentioned_users=["@all"]
        )
```

## Troubleshooting

### Issue: Crawler Not Collecting Data

**Check browser driver:**
```bash
# Verify driver is in PATH
which chromedriver  # Linux/Mac
where chromedriver  # Windows

# Test driver manually
chromedriver --version
```

**Check crawler logs:**
```python
import logging
logging.basicConfig(level=logging.DEBUG)

# Run test crawler
python runspider-test.py
```

**Common fixes:**
- Update browser and driver to matching versions
- Check MySQL connection in `settings.py`
- Verify cookies for authenticated platforms
- Reduce `CONCURRENT_REQUESTS` if getting blocked

### Issue: LLM Analysis Failing

**Check API configuration:**
```python
from hotsearch_analysis_agent.llm_client import test_connection

# Test LLM connectivity
result = test_connection()
if not result['success']:
    print(f"Error: {result['error']}")
```

**Switch to local Pangu model:**
```bash
# Download model
# https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

# Update .env
USE_PANGU_MODEL=true
PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model
```

### Issue: Push Notifications Not Sending

**Test each channel individually:**
```python
# Test email
python test_push_task.py --channel email

# Test WeChat
python test_push_task.py --channel wechat

# Test Telegram
python test_push_task.py --channel telegram
```

**Verify credentials:**
- Email: Check SMTP settings and app password
- WeChat: Verify webhook URL is valid
- Telegram: Test bot token with `/getMe` command

### Issue: Database Performance

**Add indexes for common queries:**
```sql
CREATE INDEX idx_platform_created ON hot_topics(platform, created_at);
CREATE INDEX idx_title_search ON hot_topics(title(100));
CREATE INDEX idx_heat ON hot_topics(heat);

-- For full-text search
ALTER TABLE hot_topics ADD FULLTEXT INDEX ft_title(title);
```

**Query optimization:**
```python
# Use pagination for large result sets
topics = dm.get_hot_topics(
    platform="weibo",
    limit=100,
    offset=0,
    order_by="created_at DESC"
)

# Cache frequently accessed data
from functools import lru_cache

@lru_cache(maxsize=128)
def get_platform_stats(platform, date):
    return dm.get_statistics(platform, date)
```

## Advanced Configuration

### Custom Platform Crawler

```python
# Add new platform in hotsearchcrawler/spiders/

import scrapy
from hotsearchcrawler.items import HotTopicItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    allowed_domains = ['example.com']
    start_urls = ['https://example.com/hot']
    
    def parse(self, response):
        for topic in response.css('.hot-topic'):
            item = HotTopicItem()
            item['platform'] = 'custom_platform'
            item['title'] = topic.css('.title::text').get()
            item['url'] = topic.css('a::attr(href)').get()
            item['rank'] = topic.css('.rank::text').get()
            item['heat'] = topic.css('.heat::text').get()
            yield item
```

### Custom Analysis Pipeline

```python
from hotsearch_analysis_agent.base_analyzer import BaseAnalyzer

class CustomAnalyzer(BaseAnalyzer):
    def analyze(self, topics, query):
        # Custom analysis logic
        filtered = self.filter_relevant(topics, query)
        clustered = self.custom_clustering(filtered)
        sentiment = self.analyze_sentiment(clustered)
        
        return {
            'summary': self.generate_summary(clustered),
            'clusters': clustered,
            'sentiment': sentiment,
            'insights': self.extract_insights(clustered)
        }
```

This skill provides comprehensive guidance for deploying and operating the LLM-based public opinion analytics system across all major Chinese social platforms with intelligent analysis and alerting capabilities.
