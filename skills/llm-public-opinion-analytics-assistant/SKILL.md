---
name: llm-public-opinion-analytics-assistant
description: Deploy and use an LLM-based public opinion analytics system that crawls 26 hot lists from 15 platforms, provides conversational sentiment analysis, topic clustering, and multi-channel alert pushing.
triggers:
  - set up public opinion monitoring system
  - configure multi-platform hot search crawler
  - analyze trending topics with LLM
  - implement sentiment analysis for social media
  - create hot topic alert notifications
  - build opinion analytics dashboard
  - deploy Chinese social media monitoring
  - analyze weibo douyin bilibili trends
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

This is a comprehensive public opinion analytics system that combines real-time data crawling from 15 major Chinese platforms (26 hot lists total) with LLM-powered analysis. It provides:

- Real-time hot search data collection from platforms like Weibo, Bilibili, Douyin, Baidu, etc.
- Conversational query interface for hot list exploration
- Topic clustering and sentiment analysis using LLM (Huawei Pangu model recommended)
- Multi-channel alert pushing (Email, WeChat Work, Telegram, Enterprise WeChat)
- Video content extraction and analysis capabilities
- Hotkey-controlled crawler management

The system architecture separates crawlers (`hotsearchcrawler`) from the analysis system (`hotsearch_analysis_agent`), enabling independent deployment and scaling.

## Installation

### Prerequisites

1. **Browser Driver Setup** (required for content extraction):

```bash
# Check your Chrome/Edge version first
google-chrome --version  # or microsoft-edge --version

# Download matching ChromeDriver from:
# https://chromedriver.chromium.org/

# On Linux/macOS, move to PATH:
sudo mv chromedriver /usr/local/bin/
chmod +x /usr/local/bin/chromedriver

# Verify installation:
chromedriver --version
```

2. **MySQL Database**:

```bash
# Install MySQL 5.7+ or 8.0+
# Create database
mysql -u root -p
CREATE DATABASE public_opinion CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

3. **Python Environment**:

```bash
# Clone the repository
git clone https://github.com/hmmnxkl/LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant.git
cd LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant

# Create virtual environment
python3 -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Database Initialization

Reference `init.py` to create required tables:

```python
import pymysql

connection = pymysql.connect(
    host='localhost',
    user='root',
    password='your_password',
    database='public_opinion',
    charset='utf8mb4'
)

# Execute table creation SQL from init.py
# Tables include: hot_search_data, analysis_results, push_tasks, etc.
```

### Configuration

1. **Crawler Configuration** (`hotsearchcrawler/settings.py`):

```python
# MySQL connection
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'root'
MYSQL_PASSWORD = 'your_password'
MYSQL_DATABASE = 'public_opinion'

# Optional: Platform-specific cookies for authenticated access
COOKIES = {
    'weibo': 'your_weibo_cookie',
    'douyin': 'your_douyin_cookie'
}
```

2. **Analysis System Configuration** (`.env` file):

```bash
# MySQL Configuration
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=your_password
DB_NAME=public_opinion

# LLM Configuration (OpenAI-compatible API)
LLM_API_BASE=http://localhost:8000/v1
LLM_API_KEY=your_api_key
LLM_MODEL=pangu-7b

# Push Notification Configs
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password

WECHAT_WORK_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY
TELEGRAM_BOT_TOKEN=your_telegram_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

## Running the System

### Start the Analysis System

```bash
# Main application
python app.py
```

Access the web interface at `http://localhost:5000`

### Run Crawlers

```bash
# Test individual crawler
python runspider-test.py

# Start all crawlers (can also be triggered from web UI)
python run_spiders.py
```

### Test Push Notifications

```bash
# Test push task configuration
python test_push_task.py
```

## Core Usage Patterns

### 1. Querying Hot Search Data

```python
from hotsearch_analysis_agent.database import get_hot_searches

# Query recent hot searches from specific platform
hot_searches = get_hot_searches(
    platform='weibo',
    limit=20,
    time_range='24h'
)

for item in hot_searches:
    print(f"{item['rank']}. {item['title']} - {item['heat_score']}")
```

### 2. Topic Clustering Analysis

```python
from hotsearch_analysis_agent.analyzer import TopicCluster

# Cluster related topics
clusterer = TopicCluster(llm_model='pangu-7b')

topics = [
    "GPT-6遭提前曝光",
    "DeepSeek V4采用华为算力",
    "中国大模型调用量超美国"
]

clusters = clusterer.analyze(topics)
print(clusters['summary'])
# Output: 聚类主题: AI大模型技术进展与国产化突破
```

### 3. Sentiment Analysis

```python
from hotsearch_analysis_agent.analyzer import SentimentAnalyzer

analyzer = SentimentAnalyzer(llm_model='pangu-7b')

# Analyze sentiment of a topic
result = analyzer.analyze_sentiment(
    title="首次超越OpenAI，Anthropic年化收入暴涨至300亿美元",
    content="市场研究机构披露，Anthropic凭借企业级安全合规方案..."
)

print(result['sentiment'])  # 'positive', 'negative', 'neutral'
print(result['confidence'])  # 0.92
print(result['keywords'])    # ['突破', '领先', '增长']
```

### 4. Creating Push Tasks

```python
from hotsearch_analysis_agent.push_manager import PushTask

# Create email alert for AI-related topics
task = PushTask.create(
    name="AI热点监控",
    keywords=["人工智能", "大模型", "GPT"],
    platforms=["weibo", "zhihu", "36kr"],
    frequency="daily",  # 'realtime', 'hourly', 'daily'
    channels=["email", "wechat_work"],
    threshold_heat_score=5000
)

task.save()
```

### 5. Custom Crawler for New Platform

```python
# hotsearchcrawler/spiders/new_platform.py
import scrapy
from hotsearchcrawler.items import HotSearchItem

class NewPlatformSpider(scrapy.Spider):
    name = 'new_platform'
    start_urls = ['https://newplatform.com/trending']
    
    def parse(self, response):
        for item in response.css('.hot-item'):
            hot_item = HotSearchItem()
            hot_item['platform'] = 'new_platform'
            hot_item['title'] = item.css('.title::text').get()
            hot_item['rank'] = item.css('.rank::text').get()
            hot_item['heat_score'] = item.css('.heat::text').get()
            hot_item['url'] = item.css('a::attr(href)').get()
            
            # Extract detail page content
            yield scrapy.Request(
                hot_item['url'],
                callback=self.parse_detail,
                meta={'item': hot_item}
            )
    
    def parse_detail(self, response):
        item = response.meta['item']
        item['content'] = response.css('.content::text').getall()
        yield item
```

### 6. LLM-Powered Report Generation

```python
from hotsearch_analysis_agent.report_generator import ReportGenerator

generator = ReportGenerator(llm_model='pangu-7b')

# Generate comprehensive analysis report
report = generator.generate_report(
    query="人工智能与前沿科技",
    time_range="7d",
    include_clustering=True,
    include_sentiment=True
)

print(report['markdown_content'])
# Outputs formatted markdown report with:
# - Core findings
# - Detailed news summaries
# - Trend analysis
# - Information propagation patterns
```

### 7. Multi-Channel Push Configuration

```python
from hotsearch_analysis_agent.push_channels import (
    EmailPusher, WeChatWorkPusher, TelegramPusher
)

# Email push
email_pusher = EmailPusher(
    smtp_host=os.getenv('SMTP_HOST'),
    smtp_port=int(os.getenv('SMTP_PORT')),
    username=os.getenv('SMTP_USER'),
    password=os.getenv('SMTP_PASSWORD')
)

email_pusher.send(
    to_addresses=['analyst@company.com'],
    subject='AI热点分析报告',
    content=report['markdown_content'],
    content_type='html'
)

# WeChat Work webhook
wechat_pusher = WeChatWorkPusher(
    webhook_url=os.getenv('WECHAT_WORK_WEBHOOK')
)

wechat_pusher.send_markdown(
    content=report['markdown_content']
)

# Telegram bot
telegram_pusher = TelegramPusher(
    bot_token=os.getenv('TELEGRAM_BOT_TOKEN'),
    chat_id=os.getenv('TELEGRAM_CHAT_ID')
)

telegram_pusher.send_message(
    text=report['text_summary']
)
```

## Common Workflows

### Daily Monitoring Setup

```python
# Start crawlers at scheduled intervals
from apscheduler.schedulers.background import BackgroundScheduler

scheduler = BackgroundScheduler()

# Crawl hot searches every 30 minutes
scheduler.add_job(
    func=run_all_crawlers,
    trigger='interval',
    minutes=30
)

# Generate daily report at 8 AM
scheduler.add_job(
    func=generate_daily_report,
    trigger='cron',
    hour=8,
    minute=0
)

scheduler.start()
```

### Emergency Alert System

```python
from hotsearch_analysis_agent.monitor import EmergencyMonitor

monitor = EmergencyMonitor(
    keywords=['危机', '事故', '紧急'],
    heat_threshold=10000,
    sentiment_threshold='negative',
    platforms=['weibo', 'douyin', 'toutiao']
)

# Real-time monitoring with immediate push
monitor.start(
    callback=lambda event: push_emergency_alert(event),
    interval_seconds=60
)
```

## Configuration Files

### Supported Platforms

The system supports crawling from these 15 platforms (26 lists):
- Weibo (微博热搜)
- Bilibili (B站热门)
- Douyin (抖音热榜)
- Zhihu (知乎热榜)
- Baidu (百度热搜)
- 36Kr (36氪热榜)
- Toutiao (今日头条)
- WeChat (微信热榜)
- Tieba (百度贴吧)
- And more...

### LLM Model Configuration

The project recommends Huawei Pangu model but supports any OpenAI-compatible API:

```python
# Download Pangu model from:
# https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

# Configure local inference server
LLM_CONFIG = {
    'model_path': '/path/to/pangu-7b',
    'api_base': 'http://localhost:8000/v1',
    'max_tokens': 2000,
    'temperature': 0.7
}
```

## Troubleshooting

### Browser Driver Issues

```bash
# Error: "chromedriver not found"
# Solution: Add to PATH or specify full path

export PATH=$PATH:/usr/local/bin

# Or in code:
from selenium import webdriver
options = webdriver.ChromeOptions()
driver = webdriver.Chrome(
    executable_path='/usr/local/bin/chromedriver',
    options=options
)
```

### MySQL Connection Failed

```python
# Check connection parameters
import pymysql

try:
    conn = pymysql.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT')),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    print("Database connected successfully")
except Exception as e:
    print(f"Connection error: {e}")
    # Check: 1) MySQL service running 2) Credentials correct 3) Database exists
```

### Crawler Returns Empty Data

```python
# Debug crawler with logging
import logging
from scrapy.utils.log import configure_logging

configure_logging(install_root_handler=False)
logging.basicConfig(
    filename='crawler.log',
    format='%(levelname)s: %(message)s',
    level=logging.DEBUG
)

# Check: 1) Network connectivity 2) Platform HTML structure changes 3) Rate limiting
```

### LLM Analysis Timeout

```python
# Increase timeout for long text analysis
from hotsearch_analysis_agent.llm_client import LLMClient

client = LLMClient(
    api_base=os.getenv('LLM_API_BASE'),
    timeout=300  # 5 minutes for long documents
)

# Or use streaming for long responses
response = client.generate_stream(
    prompt=long_content,
    max_tokens=4000
)

for chunk in response:
    print(chunk, end='')
```

### Push Notification Not Received

```python
# Test each channel independently
from hotsearch_analysis_agent.push_channels import test_all_channels

test_results = test_all_channels()
for channel, result in test_results.items():
    if not result['success']:
        print(f"{channel} failed: {result['error']}")
        # Check: 1) API credentials 2) Network access 3) Webhook URLs
```

## Advanced Features

### Video Content Extraction

The system can extract text from video-based hot topics:

```python
from hotsearch_analysis_agent.video_analyzer import VideoContentExtractor

extractor = VideoContentExtractor()

# Extract content from Bilibili video
content = extractor.extract(
    url='https://www.bilibili.com/video/BV13pSoBBEvX',
    methods=['ocr', 'speech_to_text', 'subtitle']
)

print(content['transcript'])
print(content['on_screen_text'])
```

### Custom Analysis Rules

```python
from hotsearch_analysis_agent.rules import AnalysisRule

# Define custom detection rule
rule = AnalysisRule(
    name="tech_breakthrough",
    conditions={
        'keywords': ['突破', '首次', '领先'],
        'platforms': ['36kr', 'zhihu'],
        'heat_growth_rate': 0.5  # 50% increase in 1 hour
    },
    action='immediate_push',
    priority='high'
)

rule.save()
```

This skill provides comprehensive coverage for deploying and operating an LLM-powered public opinion analytics system with focus on Chinese social media platforms.
