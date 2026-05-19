---
name: llm-public-opinion-analytics-assistant
description: LLM-powered public opinion analytics system that aggregates hot topics from 15 platforms, performs sentiment analysis, and delivers multi-channel alerts
triggers:
  - set up public opinion monitoring system
  - analyze social media sentiment and hot topics
  - create multi-platform hot search crawler
  - build LLM-based sentiment analysis dashboard
  - configure hot topic push notifications
  - integrate Huawei Pangu model for Chinese text analysis
  - deploy real-time opinion analytics with clustering
  - scrape and analyze trending topics from multiple platforms
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is an intelligent public opinion analytics assistant that combines real-time data from **26 ranking lists across 15 mainstream platforms** with large language model capabilities. It provides conversational hot topic queries, theme searches, topic clustering analysis, and sentiment analysis through a web interface. The system supports hotkey-controlled crawlers, multi-platform data aggregation, detailed content extraction (including video transcripts), and multi-channel push notifications (Email, WeChat, Enterprise WeChat, Telegram).

## Architecture

The project consists of two main components:

1. **Analysis System** (`hotsearch_analysis_agent/`) - LLM-powered analytics backend
2. **Crawler Cluster** (`hotsearchcrawler/`) - Distributed web scraping system using Scrapy

Both systems are fully decoupled and communicate via MySQL database.

## Installation

### Prerequisites

```bash
# Python 3.8+ required
python --version

# MySQL 8.0+ required
mysql --version
```

### Browser Driver Setup

The system requires a browser driver for content extraction:

1. **Check browser version**: Chrome/Edge → Settings → About
2. **Download matching driver**:
   - Chrome: https://chromedriver.chromium.org/
   - Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/
3. **Add to PATH**:
   ```bash
   # Linux/macOS
   export PATH=$PATH:/path/to/driver/directory
   
   # Windows: Add to System PATH via Environment Variables
   ```
4. **Verify**:
   ```bash
   chromedriver --version
   ```

### Project Setup

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

```bash
# Create database and tables (reference init.py for schema)
mysql -u root -p < init.sql

# Or use Python initialization script
python init.py
```

## Configuration

### Environment Variables

Create `.env` file in project root:

```env
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_username
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=opinion_analytics

# LLM Configuration (OpenAI-compatible API)
LLM_API_KEY=your_api_key
LLM_API_BASE=https://api.your-llm-provider.com/v1
LLM_MODEL=pangu-7b  # Or other model

# Push Notification Channels
# Enterprise WeChat Robot
WECHAT_ROBOT_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Enterprise WeChat App
WECHAT_CORP_ID=your_corp_id
WECHAT_AGENT_ID=your_agent_id
WECHAT_SECRET=your_secret

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
EMAIL_RECIPIENTS=recipient1@example.com,recipient2@example.com
```

### Crawler Configuration

Edit `hotsearchcrawler/settings.py`:

```python
# MySQL connection for crawler
MYSQL_CONFIG = {
    'host': 'localhost',
    'port': 3306,
    'user': 'your_username',
    'password': 'your_password',
    'database': 'opinion_analytics',
    'charset': 'utf8mb4'
}

# Optional: Platform-specific cookies
COOKIES = {
    'weibo': 'your_weibo_cookie',
    'bilibili': 'your_bilibili_cookie'
}

# Crawler settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 2
ROBOTSTXT_OBEY = False
```

## Running the System

### Start Analysis System

```bash
# Launch web interface and API
python app.py

# Access at http://localhost:5000
```

### Start Crawler Cluster

```bash
# Run all spiders
python run_spiders.py

# Or start from web interface using hotkey
# Default: Ctrl+Shift+S (configurable in frontend)
```

### Test Individual Components

```bash
# Test crawler for specific platform
python runspider-test.py --platform weibo

# Test push notification
python test_push_task.py --channel wechat --topic "AI技术"
```

## Core Features and API

### 1. Hot Topic Query

```python
from hotsearch_analysis_agent import AnalysisAgent

agent = AnalysisAgent()

# Query hot topics from specific platform
topics = agent.query_hot_topics(
    platform="weibo",
    limit=10,
    time_range="today"
)

for topic in topics:
    print(f"{topic['rank']}. {topic['title']} - {topic['heat']}")
```

### 2. Theme-Based Search

```python
# Search topics matching specific keywords
results = agent.search_by_theme(
    keywords=["人工智能", "ChatGPT"],
    platforms=["weibo", "zhihu", "bilibili"],
    date_from="2026-05-01",
    date_to="2026-05-19"
)

print(f"Found {len(results)} related topics")
```

### 3. Topic Clustering Analysis

```python
# Cluster related topics using LLM
clusters = agent.cluster_topics(
    topics=results,
    num_clusters=5,
    model="pangu-7b"
)

for cluster in clusters:
    print(f"Cluster: {cluster['theme']}")
    print(f"Topics: {len(cluster['topics'])}")
    print(f"Representative: {cluster['representative']}")
```

### 4. Sentiment Analysis

```python
# Analyze sentiment of topic discussions
sentiment = agent.analyze_sentiment(
    topic_id="weibo_12345",
    include_comments=True,
    depth="deep"  # Uses detailed content extraction
)

print(f"Overall: {sentiment['overall']} ({sentiment['score']})")
print(f"Positive: {sentiment['positive_ratio']}%")
print(f"Negative: {sentiment['negative_ratio']}%")
print(f"Neutral: {sentiment['neutral_ratio']}%")
```

### 5. Multi-Channel Push Notification

```python
from hotsearch_analysis_agent.push import PushManager

push_mgr = PushManager()

# Create scheduled push task
task = push_mgr.create_task(
    name="AI Tech Monitoring",
    keywords=["人工智能", "前沿科技"],
    schedule="0 9,18 * * *",  # Cron: 9AM and 6PM daily
    channels=["wechat", "telegram", "email"],
    analysis_depth="detailed",
    min_heat_threshold=10000
)

# Manual trigger
report = push_mgr.generate_report(
    query="人工智能与前沿科技",
    platforms=["all"],
    analysis_type="comprehensive"
)

push_mgr.send(
    report=report,
    channels=["wechat", "email"]
)
```

## Common Patterns

### Pattern 1: Daily Hot Topic Digest

```python
from hotsearch_analysis_agent import AnalysisAgent
from datetime import datetime

agent = AnalysisAgent()

# Aggregate top topics from all platforms
daily_digest = agent.get_daily_digest(
    date=datetime.now().date(),
    top_n=50,
    platforms=["weibo", "zhihu", "douyin", "bilibili"]
)

# Generate analysis report
report = agent.generate_report(
    topics=daily_digest,
    include_sentiment=True,
    include_clustering=True,
    format="markdown"
)

print(report)
```

### Pattern 2: Brand Monitoring

```python
# Monitor brand mentions across platforms
brand_mentions = agent.monitor_brand(
    brand_name="华为",
    keywords=["盘古", "昇腾", "鸿蒙"],
    alert_threshold=5000,  # Alert if heat > 5000
    sentiment_alert="negative",  # Alert on negative sentiment
    platforms=["weibo", "zhihu", "toutiao"]
)

if brand_mentions['alert']:
    # Trigger immediate notification
    push_mgr.send_alert(
        title="Brand Alert: Negative Sentiment Spike",
        content=brand_mentions['summary'],
        channels=["wechat", "email"],
        priority="high"
    )
```

### Pattern 3: Topic Trend Analysis

```python
# Track topic evolution over time
trend = agent.analyze_trend(
    topic_keyword="DeepSeek V4",
    days=7,
    metrics=["heat", "sentiment", "spread_platforms"]
)

# Visualize trend data
import matplotlib.pyplot as plt

plt.plot(trend['dates'], trend['heat'])
plt.title("Topic Heat Trend")
plt.xlabel("Date")
plt.ylabel("Heat Score")
plt.savefig("trend.png")
```

### Pattern 4: Cross-Platform Comparison

```python
# Compare same topic across platforms
comparison = agent.compare_platforms(
    topic="GPT-6",
    platforms=["weibo", "zhihu", "bilibili", "douyin"],
    metrics=["heat", "sentiment", "comment_count", "spread_speed"]
)

for platform, data in comparison.items():
    print(f"{platform}: Heat={data['heat']}, Sentiment={data['sentiment']}")
```

## Database Schema Reference

Key tables used by the system:

```sql
-- Hot topics table
CREATE TABLE hot_topics (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    platform VARCHAR(50) NOT NULL,
    title VARCHAR(500) NOT NULL,
    url VARCHAR(1000),
    heat BIGINT DEFAULT 0,
    rank INT,
    content TEXT,
    sentiment VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_platform_created (platform, created_at),
    INDEX idx_heat (heat DESC)
);

-- Push tasks table
CREATE TABLE push_tasks (
    id INT AUTO_INCREMENT PRIMARY KEY,
    task_name VARCHAR(200) NOT NULL,
    keywords JSON,
    schedule VARCHAR(100),
    channels JSON,
    status VARCHAR(20) DEFAULT 'active',
    last_run TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Analysis results cache
CREATE TABLE analysis_cache (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    query_hash VARCHAR(64) UNIQUE,
    query_params JSON,
    result JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP,
    INDEX idx_expires (expires_at)
);
```

## Huawei Pangu Model Integration

The project recommends Huawei Pangu model for Chinese text analysis:

```python
from hotsearch_analysis_agent.llm import PanguModel

# Initialize Pangu model
model = PanguModel(
    model_path="/path/to/openpangu-embedded-7b",
    device="cuda",  # or "cpu"
    max_length=2048
)

# Perform sentiment analysis
result = model.analyze_sentiment(
    text="今天看到DeepSeek V4采用华为昇腾算力的新闻，感觉国产AI生态越来越完善了！",
    context="科技新闻评论"
)

print(result['sentiment'])  # 'positive'
print(result['confidence'])  # 0.92

# Generate summary
summary = model.summarize(
    text=long_article_content,
    max_summary_length=200,
    style="professional"
)

print(summary)
```

Download Pangu model:
- https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

## Troubleshooting

### Issue: Crawler Not Collecting Data

```bash
# Check crawler logs
tail -f hotsearchcrawler/logs/spider.log

# Verify MySQL connection
python -c "from hotsearchcrawler.settings import MYSQL_CONFIG; import pymysql; pymysql.connect(**MYSQL_CONFIG)"

# Test individual spider
scrapy crawl weibo_spider -s LOG_LEVEL=DEBUG
```

### Issue: Browser Driver Not Found

```bash
# Linux: Install system-wide
sudo apt-get install chromium-chromedriver

# macOS: Use Homebrew
brew install chromedriver

# Verify PATH
which chromedriver
```

### Issue: LLM API Timeout

```python
# Increase timeout in config
from hotsearch_analysis_agent.llm import LLMConfig

config = LLMConfig(
    api_timeout=120,  # seconds
    max_retries=3,
    retry_delay=5
)

agent = AnalysisAgent(llm_config=config)
```

### Issue: Push Notification Failed

```bash
# Test each channel individually
python test_push_task.py --channel email --debug
python test_push_task.py --channel wechat --debug
python test_push_task.py --channel telegram --debug

# Check environment variables
env | grep -E '(WECHAT|TELEGRAM|SMTP)'
```

### Issue: Memory Usage Too High

```python
# Optimize batch processing
agent = AnalysisAgent(
    batch_size=10,  # Process 10 topics at a time
    cache_enabled=True,
    cache_ttl=3600  # 1 hour cache
)

# Clear old cache periodically
agent.clear_cache(older_than_days=7)
```

## Advanced Configuration

### Custom Spider Development

```python
# Create custom spider in hotsearchcrawler/spiders/
from scrapy import Spider
from hotsearchcrawler.items import HotTopicItem

class MyPlatformSpider(Spider):
    name = 'myplatform'
    start_urls = ['https://myplatform.com/hot']
    
    def parse(self, response):
        for topic in response.css('.topic-item'):
            item = HotTopicItem()
            item['platform'] = 'myplatform'
            item['title'] = topic.css('.title::text').get()
            item['url'] = topic.css('a::attr(href)').get()
            item['heat'] = int(topic.css('.heat::text').get())
            item['rank'] = topic.css('.rank::text').get()
            yield item
```

### Custom Analysis Pipeline

```python
from hotsearch_analysis_agent.pipeline import AnalysisPipeline

class CustomAnalyzer(AnalysisPipeline):
    def process(self, topics):
        # Add custom processing logic
        enriched = self.enrich_with_external_data(topics)
        clustered = self.custom_clustering(enriched)
        scored = self.custom_scoring(clustered)
        return scored

# Register custom analyzer
agent.register_pipeline(CustomAnalyzer())
```

This skill provides comprehensive guidance for deploying and using the LLM-based public opinion analytics system in production environments.
