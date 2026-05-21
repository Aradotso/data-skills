---
name: llm-public-opinion-analytics-assistant
description: Multi-platform real-time public opinion analytics system with LLM-powered sentiment analysis, topic clustering, and multi-channel alerts
triggers:
  - analyze public opinion trends from social media
  - set up hot topic monitoring and alerts
  - crawl trending topics from multiple platforms
  - perform sentiment analysis on news articles
  - cluster related topics with LLM
  - configure multi-channel notifications for trending events
  - monitor Weibo Douyin and Bilibili hot searches
  - extract and analyze video content from news sources
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is a comprehensive public opinion analytics platform that combines real-time data crawling from 15 mainstream platforms (26 trending lists total) with LLM-powered analysis capabilities. It provides conversational hot search queries, topic-specific searches, topic clustering analysis, and sentiment analysis through a web interface. The system supports multi-channel push notifications (WeChat Work, Telegram, Email) and can extract insights even from video-based news content.

**Key Platforms Covered:** Weibo, Douyin, Bilibili, Zhihu, Baidu, Toutiao, and more.

**Core Capabilities:**
- Real-time trending topic crawling with Scrapy cluster
- LLM-powered sentiment and clustering analysis (uses Huawei Pangu model)
- Multi-channel alert system
- Video content extraction and analysis
- Natural language query interface
- Keyboard shortcuts for crawler control

## Installation

### Prerequisites

1. **Python 3.8+** with virtual environment
2. **MySQL Database** for data storage
3. **Browser Driver** (Chrome/Edge) for content extraction

### Browser Driver Setup

```bash
# Check your browser version first
google-chrome --version  # or microsoft-edge --version

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

### Database Configuration

```bash
# Create MySQL database
mysql -u root -p
```

```sql
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE hotsearch_db;

-- Run init.py to create tables
-- Or manually create required tables based on init.py structure
```

### Environment Configuration

Create `.env` file in project root:

```bash
# Database Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_mysql_user
MYSQL_PASSWORD=your_mysql_password
MYSQL_DATABASE=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://your-llm-endpoint.com/v1
OPENAI_MODEL=pangu-embedded-7b  # or your model name

# Push Notification Settings (optional)
# WeChat Work Robot
WECHAT_ROBOT_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=your_key

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email (SMTP)
EMAIL_HOST=smtp.gmail.com
EMAIL_PORT=587
EMAIL_USER=your_email@gmail.com
EMAIL_PASSWORD=your_app_password
EMAIL_RECIPIENTS=recipient1@example.com,recipient2@example.com
```

### Crawler Settings

Edit `hotsearchcrawler/settings.py`:

```python
# MySQL connection for crawler
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_mysql_user'
MYSQL_PASSWORD = 'your_mysql_password'
MYSQL_DATABASE = 'hotsearch_db'

# Optional: Platform-specific cookies
COOKIES = {
    'weibo': 'your_weibo_cookies',
    'douyin': 'your_douyin_cookies',
    # Add as needed
}
```

## Core Usage

### Starting the System

```bash
# Initialize database (first time only)
python init.py

# Start the main application
python app.py

# Access web interface at http://localhost:5000
```

### Running Crawlers

```python
# Programmatic crawler start
from hotsearchcrawler.run_spiders import start_all_crawlers

# Start all platform crawlers
start_all_crawlers()

# Or run specific platform
import subprocess
subprocess.run(['scrapy', 'crawl', 'weibo_hot'], cwd='hotsearchcrawler')
```

```bash
# CLI crawler execution
cd hotsearchcrawler

# Run single crawler
scrapy crawl weibo_hot

# Run all crawlers (defined in run_spiders.py)
python run_spiders.py
```

### Analysis Agent Usage

```python
from hotsearch_analysis_agent.agent import OpinionAnalysisAgent
from hotsearch_analysis_agent.database import DatabaseManager

# Initialize agent
db = DatabaseManager(
    host=os.getenv('MYSQL_HOST'),
    user=os.getenv('MYSQL_USER'),
    password=os.getenv('MYSQL_PASSWORD'),
    database=os.getenv('MYSQL_DATABASE')
)

agent = OpinionAnalysisAgent(
    llm_api_key=os.getenv('OPENAI_API_KEY'),
    llm_base_url=os.getenv('OPENAI_API_BASE'),
    model_name=os.getenv('OPENAI_MODEL')
)

# Query trending topics
query = "分析人工智能相关的热点新闻"
results = db.search_topics(query)

# Perform sentiment analysis
sentiment_report = agent.analyze_sentiment(results)
print(sentiment_report)

# Topic clustering
clusters = agent.cluster_topics(results, num_clusters=5)
for cluster_id, topics in clusters.items():
    print(f"Cluster {cluster_id}: {topics}")

# Generate comprehensive report
report = agent.generate_report(
    query=query,
    topics=results,
    include_sentiment=True,
    include_clustering=True
)
print(report)
```

### Setting Up Push Notifications

```python
from test_push_task import PushTaskManager

# Initialize push manager
push_manager = PushTaskManager()

# Create monitoring task
task_config = {
    'name': 'AI Technology Monitor',
    'keywords': ['人工智能', 'AI', 'GPT', 'DeepSeek'],
    'platforms': ['weibo', 'zhihu', 'toutiao'],
    'channels': ['wechat_work', 'telegram', 'email'],
    'interval_minutes': 30,  # Check every 30 minutes
    'min_heat_score': 5000,  # Minimum trending score
    'sentiment_threshold': 0.6  # Alert on strong sentiment
}

# Add task
push_manager.create_task(task_config)

# Test push to specific channel
push_manager.test_push(
    channel='telegram',
    message='Test alert: New trending topic detected',
    report_data={'title': 'GPT-6发布', 'score': 150000}
)
```

### Web Interface Query Examples

The web interface supports natural language queries:

```python
# Example conversational queries through the UI:
# 1. "查询今天微博热搜前10"
# 2. "搜索关于科技的热点话题"
# 3. "分析最近AI相关新闻的情感倾向"
# 4. "聚类分析娱乐板块的热门话题"
# 5. "设置关键词'新能源汽车'的推送任务"
```

## Common Patterns

### Custom Crawler for New Platform

```python
# hotsearchcrawler/spiders/custom_platform.py
import scrapy
from hotsearchcrawler.items import HotSearchItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://platform.com/trending']
    
    def parse(self, response):
        for topic in response.css('.trending-item'):
            item = HotSearchItem()
            item['title'] = topic.css('.title::text').get()
            item['url'] = topic.css('a::attr(href)').get()
            item['hot_score'] = topic.css('.score::text').get()
            item['platform'] = 'custom_platform'
            item['rank'] = topic.css('.rank::text').get()
            item['crawl_time'] = datetime.now()
            
            # Extract detail page content
            yield scrapy.Request(
                item['url'],
                callback=self.parse_detail,
                meta={'item': item}
            )
    
    def parse_detail(self, response):
        item = response.meta['item']
        item['content'] = response.css('.article-content::text').getall()
        yield item
```

### Sentiment Analysis with Custom Prompts

```python
from hotsearch_analysis_agent.llm_client import LLMClient

llm = LLMClient(
    api_key=os.getenv('OPENAI_API_KEY'),
    base_url=os.getenv('OPENAI_API_BASE')
)

# Custom sentiment analysis prompt
def analyze_with_custom_prompt(content):
    prompt = f"""
    分析以下新闻内容的情感倾向和舆论风险等级:
    
    内容: {content}
    
    请从以下维度分析:
    1. 情感极性(正面/中性/负面)及置信度
    2. 舆论风险等级(低/中/高)
    3. 关键情感触发词
    4. 建议应对策略
    
    以JSON格式返回结果。
    """
    
    response = llm.chat_completion(
        messages=[{'role': 'user', 'content': prompt}],
        temperature=0.3
    )
    return response
```

### Batch Processing with Database

```python
from hotsearch_analysis_agent.database import DatabaseManager

db = DatabaseManager.from_env()

# Get unprocessed topics
unprocessed = db.execute_query("""
    SELECT id, title, content, platform 
    FROM hot_topics 
    WHERE sentiment_score IS NULL 
    LIMIT 100
""")

# Batch analyze
for topic in unprocessed:
    sentiment = agent.analyze_sentiment(topic['content'])
    
    db.execute_update("""
        UPDATE hot_topics 
        SET sentiment_score = %s, 
            sentiment_label = %s,
            processed_at = NOW()
        WHERE id = %s
    """, (sentiment['score'], sentiment['label'], topic['id']))
```

### Multi-Channel Alert Orchestration

```python
class AlertOrchestrator:
    def __init__(self):
        self.channels = {
            'wechat_work': WeChatWorkNotifier(),
            'telegram': TelegramNotifier(),
            'email': EmailNotifier()
        }
    
    def send_alert(self, report, channels=['wechat_work']):
        """Send alert through multiple channels"""
        for channel_name in channels:
            if channel_name in self.channels:
                try:
                    self.channels[channel_name].send(
                        title=report['title'],
                        content=report['summary'],
                        urgency=report.get('urgency', 'normal')
                    )
                except Exception as e:
                    print(f"Failed to send via {channel_name}: {e}")
    
    def scheduled_monitoring(self, keywords, interval=1800):
        """Monitor and alert on keyword matches"""
        import time
        while True:
            results = db.search_topics_by_keywords(keywords)
            if results:
                report = agent.generate_report(results)
                self.send_alert(report, channels=['telegram', 'email'])
            time.sleep(interval)
```

## Configuration Tips

### Optimizing Crawler Performance

```python
# hotsearchcrawler/settings.py

# Concurrent requests
CONCURRENT_REQUESTS = 16
CONCURRENT_REQUESTS_PER_DOMAIN = 8

# Download delays
DOWNLOAD_DELAY = 1
RANDOMIZE_DOWNLOAD_DELAY = True

# AutoThrottle for adaptive crawling
AUTOTHROTTLE_ENABLED = True
AUTOTHROTTLE_START_DELAY = 1
AUTOTHROTTLE_MAX_DELAY = 10
AUTOTHROTTLE_TARGET_CONCURRENCY = 2.0

# Retry settings
RETRY_TIMES = 3
RETRY_HTTP_CODES = [500, 502, 503, 504, 522, 524, 408, 429]
```

### LLM Model Configuration

```python
# For Huawei Pangu model (local deployment)
LLM_CONFIG = {
    'model_path': '/path/to/openpangu-embedded-7b-model',
    'device': 'cuda',  # or 'cpu'
    'max_length': 4096,
    'temperature': 0.7,
    'top_p': 0.9
}

# For OpenAI-compatible API
LLM_CONFIG = {
    'api_key': os.getenv('OPENAI_API_KEY'),
    'base_url': os.getenv('OPENAI_API_BASE'),
    'model': 'pangu-embedded-7b',
    'timeout': 60
}
```

## Troubleshooting

### Browser Driver Issues

```bash
# Error: chromedriver not found
# Solution: Verify driver is in PATH
which chromedriver  # Linux/macOS
where chromedriver  # Windows

# Error: version mismatch
# Solution: Download exact matching version
google-chrome --version
# Download corresponding driver from official site

# Error: Permission denied
chmod +x /path/to/chromedriver
```

### Database Connection Errors

```python
# Test database connection
from hotsearch_analysis_agent.database import DatabaseManager

try:
    db = DatabaseManager.from_env()
    db.test_connection()
    print("Database connection successful")
except Exception as e:
    print(f"Connection failed: {e}")
    # Check: MySQL service running, credentials, firewall
```

### Crawler Not Collecting Data

```bash
# Enable Scrapy debug logging
export SCRAPY_LOG_LEVEL=DEBUG
scrapy crawl weibo_hot

# Check specific issues:
# 1. Cookies expired - update in settings.py
# 2. Platform changed HTML structure - update CSS selectors
# 3. IP blocked - add proxy middleware or reduce crawl rate
```

### LLM API Timeouts

```python
# Increase timeout and add retry logic
import openai
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=4, max=10))
def call_llm_with_retry(prompt):
    response = openai.ChatCompletion.create(
        model=os.getenv('OPENAI_MODEL'),
        messages=[{'role': 'user', 'content': prompt}],
        timeout=120  # Increased timeout
    )
    return response

# For extremely long content, split into chunks
def analyze_long_content(content, chunk_size=3000):
    chunks = [content[i:i+chunk_size] for i in range(0, len(content), chunk_size)]
    results = [call_llm_with_retry(chunk) for chunk in chunks]
    return aggregate_results(results)
```

### Push Notification Failures

```python
# Test each channel individually
from test_push_task import test_wechat_work, test_telegram, test_email

# WeChat Work webhook test
test_wechat_work()

# Telegram bot test (verify bot token and chat ID)
test_telegram()

# Email SMTP test (check app passwords for Gmail)
test_email()

# Common fixes:
# - Verify webhook URLs are active
# - Check firewall allows outbound connections
# - Validate API tokens haven't expired
```

## Advanced Usage

### Huawei Pangu Model Integration

```python
# Download model from: https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

from transformers import AutoModelForCausalLM, AutoTokenizer

model_path = "/path/to/openpangu-embedded-7b-model"
tokenizer = AutoTokenizer.from_pretrained(model_path, trust_remote_code=True)
model = AutoModelForCausalLM.from_pretrained(
    model_path,
    trust_remote_code=True,
    device_map="auto"
)

def analyze_with_pangu(text):
    prompt = f"请分析以下舆情内容的情感倾向:\n{text}\n\n分析结果:"
    inputs = tokenizer(prompt, return_tensors="pt")
    outputs = model.generate(**inputs, max_length=2048)
    return tokenizer.decode(outputs[0], skip_special_tokens=True)
```

This skill provides comprehensive guidance for using the LLM-Based Public Opinion Analytics Assistant for real-time social media monitoring, sentiment analysis, and automated alerting.
