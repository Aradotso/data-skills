---
name: llm-public-opinion-analytics-assistant
description: Multi-platform hot search crawler and LLM-powered sentiment analysis system with push notifications
triggers:
  - "help me analyze public opinion from Chinese social platforms"
  - "set up a hot search monitoring system with sentiment analysis"
  - "create a multi-platform news crawler with LLM analysis"
  - "configure automated sentiment reports with push notifications"
  - "scrape trending topics from Weibo, Bilibili, and other platforms"
  - "build a public opinion analytics dashboard with AI"
  - "implement hot search aggregation with emotion detection"
  - "deploy a Chinese social media monitoring system"
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive public opinion monitoring system that aggregates real-time data from 15 Chinese platforms (26 different ranking lists), performs LLM-powered analysis including topic clustering and sentiment analysis, and delivers automated reports via email, WeChat, Enterprise WeChat, and Telegram.

## What This Project Does

- **Multi-Platform Crawling**: Scrapes hot search data from Weibo, Bilibili, Douyin, Zhihu, Baidu, and 10+ other platforms
- **LLM Analysis**: Uses Huawei PanGu or OpenAI-compatible models for sentiment analysis, topic clustering, and trend summarization
- **Conversational Interface**: Web UI for natural language queries about trending topics
- **Deep Content Extraction**: Fetches full article/video content from detail pages using browser automation
- **Multi-Channel Notifications**: Push reports to email (SMTP), Enterprise WeChat, Telegram, and personal WeChat
- **Hotkey Control**: Start/stop crawlers via keyboard shortcuts

## Installation

### Prerequisites

1. **Python Environment**:
```bash
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
pip install -r requirements.txt
```

2. **MySQL Database**:
```bash
# Install MySQL 5.7+ and create database
mysql -u root -p
CREATE DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

3. **Browser Driver** (for detail page scraping):
```bash
# Download ChromeDriver/EdgeDriver matching your browser version
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/microsoft-edge/tools/webdriver/

# Place in system PATH or project root
# Verify installation:
chromedriver --version
```

### Database Setup

```python
# Run init.py to create tables
import mysql.connector

conn = mysql.connector.connect(
    host=os.getenv('MYSQL_HOST', 'localhost'),
    user=os.getenv('MYSQL_USER'),
    password=os.getenv('MYSQL_PASSWORD'),
    database=os.getenv('MYSQL_DATABASE', 'hotsearch')
)

# Tables created:
# - hot_search_items (trending topics)
# - news_details (scraped content)
# - analysis_results (LLM outputs)
# - push_tasks (notification jobs)
```

## Configuration

### Environment Variables

Create `.env` in project root:

```bash
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch

# LLM Configuration (OpenAI-compatible API)
LLM_API_KEY=your_api_key
LLM_API_BASE=https://api.openai.com/v1
LLM_MODEL=gpt-4

# Alternative: Huawei PanGu (local deployment)
# PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model

# Push Notification Settings
# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
EMAIL_RECIPIENTS=recipient1@example.com,recipient2@example.com

# Enterprise WeChat Webhook
WECHAT_WEBHOOK_URL=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Browser Driver Path (optional, if not in PATH)
CHROMEDRIVER_PATH=/usr/local/bin/chromedriver
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
}

# Optional: Add cookies for platforms requiring authentication
COOKIES = {
    'weibo': 'your_weibo_cookie_string',
    'zhihu': 'your_zhihu_cookie_string',
}

# Crawl settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1  # Seconds between requests
```

## Running the System

### Start the Web Application

```bash
python app.py
```

Access UI at `http://localhost:5000`

### Run Crawlers Manually

```bash
# Test single spider
python runspider-test.py --spider weibo

# Run all spiders (use UI hotkey in production)
python run_spiders.py
```

### Test Push Notifications

```bash
python test_push_task.py
```

## Key Usage Patterns

### 1. Querying Hot Search Data

```python
from hotsearch_analysis_agent.database import get_latest_trends

# Get top 10 trending topics from all platforms
trends = get_latest_trends(limit=10)

for trend in trends:
    print(f"{trend['platform']}: {trend['title']} (热度: {trend['heat']})")
    print(f"URL: {trend['url']}\n")
```

### 2. Performing LLM Analysis

```python
from hotsearch_analysis_agent.llm_analyzer import analyze_sentiment, cluster_topics

# Sentiment analysis on a single topic
topic_text = "某明星恋情曝光引发网友热议"
sentiment = analyze_sentiment(topic_text)
print(f"情感倾向: {sentiment['emotion']} (置信度: {sentiment['confidence']})")

# Topic clustering across multiple trends
trends_list = [
    "GPT-6提前曝光",
    "DeepSeek V4采用华为算力",
    "中国大模型调用量超美国"
]
clusters = cluster_topics(trends_list)
for cluster in clusters:
    print(f"主题: {cluster['theme']}")
    print(f"相关话题: {', '.join(cluster['topics'])}\n")
```

### 3. Creating Custom Crawlers

```python
# hotsearchcrawler/spiders/custom_spider.py
import scrapy
from hotsearchcrawler.items import HotSearchItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://example.com/trending']
    
    def parse(self, response):
        for item in response.css('.trending-item'):
            yield HotSearchItem(
                platform='CustomPlatform',
                title=item.css('.title::text').get(),
                heat=item.css('.heat::text').get(),
                url=item.css('a::attr(href)').get(),
                rank=item.css('.rank::text').get(),
                timestamp=datetime.now()
            )
```

### 4. Setting Up Automated Reports

```python
from hotsearch_analysis_agent.push_service import schedule_report

# Schedule daily AI report at 12:00
schedule_report(
    query="人工智能与前沿科技",
    channels=['email', 'wechat', 'telegram'],
    schedule_time="12:00",
    frequency="daily"
)
```

### 5. Extracting Video Content Details

```python
from hotsearch_analysis_agent.content_extractor import extract_video_info

# Extract transcript/metadata from Bilibili video
video_url = "https://www.bilibili.com/video/BV13pSoBBEvX"
content = extract_video_info(video_url)

print(f"标题: {content['title']}")
print(f"描述: {content['description']}")
print(f"字幕文本: {content['transcript'][:200]}...")
```

## API Reference

### Core Analysis Functions

```python
# hotsearch_analysis_agent/llm_analyzer.py

def analyze_sentiment(text: str) -> dict:
    """
    Analyze sentiment of text using LLM
    
    Returns:
        {
            'emotion': str,  # 'positive', 'negative', 'neutral'
            'confidence': float,
            'keywords': list[str]
        }
    """

def cluster_topics(topics: list[str], num_clusters: int = 5) -> list[dict]:
    """
    Cluster related topics using LLM
    
    Returns list of:
        {
            'theme': str,
            'topics': list[str],
            'summary': str
        }
    """

def generate_report(query: str, time_range: str = '24h') -> str:
    """
    Generate markdown report for query
    
    Args:
        query: Search keywords (e.g., "人工智能")
        time_range: '24h', '7d', '30d'
    
    Returns:
        Markdown-formatted analysis report
    """
```

### Database Query Helpers

```python
# hotsearch_analysis_agent/database.py

def get_trends_by_platform(platform: str, limit: int = 20) -> list[dict]:
    """Get latest trends from specific platform"""

def search_topics(keyword: str, platforms: list[str] = None) -> list[dict]:
    """Search across platforms for keyword"""

def get_trend_history(topic_id: int, days: int = 7) -> list[dict]:
    """Get historical heat data for topic"""
```

## Common Workflows

### Workflow 1: Daily Monitoring Setup

```python
# 1. Start crawlers (via UI or CLI)
python run_spiders.py --platforms weibo,zhihu,bilibili --interval 3600

# 2. Configure analysis keywords
from hotsearch_analysis_agent.config import add_monitor_keyword
add_monitor_keyword("科技创新", alert_threshold=10000)

# 3. Set up push notification
from hotsearch_analysis_agent.push_service import create_push_task
create_push_task(
    name="Daily Tech Report",
    query="科技创新 OR 人工智能",
    channels=['email', 'wechat'],
    trigger_time="09:00"
)
```

### Workflow 2: Ad-Hoc Analysis

```python
from hotsearch_analysis_agent.analyzer import analyze_query

# Conversational query via API
result = analyze_query(
    user_input="最近有哪些关于新能源汽车的热点?",
    context_days=3
)

print(result['summary'])
for item in result['related_topics']:
    print(f"- {item['title']} ({item['platform']}) - {item['url']}")
```

### Workflow 3: Custom Report Generation

```python
from hotsearch_analysis_agent.report_generator import ReportGenerator

generator = ReportGenerator()
report = generator.create_custom_report(
    title="周度科技热点分析",
    keywords=["人工智能", "芯片", "新能源"],
    time_range="7d",
    include_sentiment=True,
    include_clustering=True
)

# Save to file
with open("weekly_report.md", "w", encoding="utf-8") as f:
    f.write(report)

# Or push directly
generator.push_report(report, channels=['email'])
```

## Troubleshooting

### ChromeDriver Version Mismatch

```bash
# Error: "ChromeDriver only supports Chrome version X"

# Solution 1: Update ChromeDriver to match browser
wget https://chromedriver.storage.googleapis.com/LATEST_RELEASE
# Download corresponding version

# Solution 2: Use webdriver-manager (auto-updates)
pip install webdriver-manager

# In your code:
from selenium import webdriver
from webdriver_manager.chrome import ChromeDriverManager

driver = webdriver.Chrome(ChromeDriverManager().install())
```

### MySQL Connection Errors

```python
# Error: "Can't connect to MySQL server"

# Check connection settings
import mysql.connector

try:
    conn = mysql.connector.connect(
        host=os.getenv('MYSQL_HOST'),
        user=os.getenv('MYSQL_USER'),
        password=os.getenv('MYSQL_PASSWORD')
    )
    print("Connected successfully")
except mysql.connector.Error as err:
    print(f"Error: {err}")
    # Common fixes:
    # 1. Verify MySQL is running: sudo systemctl status mysql
    # 2. Check firewall: sudo ufw allow 3306
    # 3. Grant privileges: GRANT ALL ON hotsearch.* TO 'user'@'localhost';
```

### LLM API Rate Limits

```python
# Error: "Rate limit exceeded"

# Solution: Implement retry with backoff
import time
from functools import wraps

def retry_with_backoff(max_retries=3, base_delay=2):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if "rate_limit" in str(e).lower():
                        delay = base_delay * (2 ** attempt)
                        print(f"Rate limited, retrying in {delay}s...")
                        time.sleep(delay)
                    else:
                        raise
            raise Exception("Max retries exceeded")
        return wrapper
    return decorator

@retry_with_backoff()
def call_llm_api(prompt):
    # Your LLM call here
    pass
```

### Scrapy Spider Not Finding Items

```python
# Debugging scraped data

# 1. Enable verbose logging
# In settings.py:
LOG_LEVEL = 'DEBUG'

# 2. Use Scrapy shell for testing selectors
scrapy shell 'https://weibo.com/hot/search'
# Then test:
response.css('.trending-item .title::text').getall()

# 3. Check for dynamic content
# If page uses JavaScript rendering:
from scrapy_selenium import SeleniumRequest

def start_requests(self):
    yield SeleniumRequest(url=self.start_urls[0], callback=self.parse)
```

### Push Notification Not Sending

```python
# Test individual channels

# Email test
from hotsearch_analysis_agent.push_service import send_email
send_email(
    subject="Test",
    body="Test message",
    recipients=[os.getenv('EMAIL_RECIPIENTS').split(',')[0]]
)

# Enterprise WeChat test
import requests
webhook_url = os.getenv('WECHAT_WEBHOOK_URL')
response = requests.post(webhook_url, json={
    "msgtype": "text",
    "text": {"content": "Test message"}
})
print(response.json())

# Telegram test
from telegram import Bot
bot = Bot(token=os.getenv('TELEGRAM_BOT_TOKEN'))
bot.send_message(
    chat_id=os.getenv('TELEGRAM_CHAT_ID'),
    text="Test message"
)
```

## Advanced Configuration

### Using Local PanGu Model

```python
# Download model from:
# https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

# Configure in .env:
PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model
USE_LOCAL_MODEL=true

# Load model
from transformers import AutoModelForCausalLM, AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained(os.getenv('PANGU_MODEL_PATH'))
model = AutoModelForCausalLM.from_pretrained(
    os.getenv('PANGU_MODEL_PATH'),
    device_map="auto",
    torch_dtype="auto"
)

# Use for analysis
def analyze_with_pangu(prompt: str) -> str:
    inputs = tokenizer(prompt, return_tensors="pt")
    outputs = model.generate(**inputs, max_length=512)
    return tokenizer.decode(outputs[0])
```

### Custom Prompt Engineering

```python
# Customize analysis prompts in hotsearch_analysis_agent/prompts.py

SENTIMENT_PROMPT = """
分析以下文本的情感倾向,并返回JSON格式:
文本: {text}

返回格式:
{{"emotion": "positive/negative/neutral", "confidence": 0.0-1.0, "keywords": []}}
"""

CLUSTERING_PROMPT = """
将以下话题进行聚类分析,识别主要主题:
{topics}

请按主题归类并生成摘要。
"""
```

This skill provides comprehensive coverage of the LLM-Based Public Opinion Analytics Assistant for AI coding agents to effectively assist developers in deploying and customizing the system.
