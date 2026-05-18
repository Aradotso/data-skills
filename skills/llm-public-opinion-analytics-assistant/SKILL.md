---
name: llm-public-opinion-analytics-assistant
description: Use the LLM-based intelligent public opinion analytics assistant to crawl, analyze, and monitor hot topics from 15+ Chinese platforms with AI-powered clustering and sentiment analysis
triggers:
  - how do I set up the public opinion analytics assistant
  - crawl hot search data from Chinese social platforms
  - analyze trending topics with sentiment analysis
  - configure multi-platform hot topic monitoring
  - set up automated alerts for trending news
  - cluster and analyze social media discussions
  - integrate Pangu LLM for opinion analysis
  - push hot topic reports to WeChat or Telegram
---

# LLM Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The LLM-Based Intelligent Public Opinion Analytics Assistant is a Python-based system that combines real-time data crawling from 26 hot topic lists across 15 mainstream Chinese platforms with large language model analysis capabilities. It provides conversational hot search queries, topic clustering, sentiment analysis, and multi-channel alert pushing (email, WeChat, Enterprise WeChat, Telegram).

**Key Features:**
- Real-time data collection from Weibo, Bilibili, Zhihu, Douyin, Baidu, and 10+ other platforms
- AI-powered topic clustering and sentiment analysis using LLMs (supports Pangu, OpenAI-compatible models)
- Natural language query interface for hot topic exploration
- Video content analysis extraction
- Multi-channel push notifications (WeChat Work, Telegram, Email)
- Keyboard shortcuts for crawler control
- Separated crawler cluster and analysis system architecture

## Installation

### Prerequisites

1. **Browser Driver Setup** (required for news detail page scraping):

```bash
# Download ChromeDriver or EdgeDriver matching your browser version
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/microsoft-edge/tools/webdriver/

# Add driver to PATH or place in project directory
# Verify installation:
chromedriver --version
```

2. **MySQL Database**:

```bash
# Install MySQL 5.7+ and create database
mysql -u root -p
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4;
```

3. **Python Environment**:

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

```python
# Reference init.py to create tables
from hotsearch_analysis_agent.database import init_database

init_database()
```

## Configuration

### 1. Crawler Settings (`hotsearchcrawler/settings.py`)

```python
# MySQL connection
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_username'
MYSQL_PASSWORD = 'your_password'
MYSQL_DATABASE = 'hotsearch_db'

# Optional: Platform-specific cookies for authenticated access
COOKIES = {
    'weibo': 'your_weibo_cookie_string',
    'bilibili': 'your_bilibili_cookie_string'
}
```

### 2. Analysis System Settings (`.env` file)

```bash
# MySQL Configuration
DB_HOST=localhost
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Or use Pangu model (local deployment)
# Download from: https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model
PANGU_MODEL_PATH=/path/to/pangu/model

# Push Notification Channels
# Enterprise WeChat
WECHAT_WORK_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_telegram_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
SMTP_FROM=your_email@gmail.com
SMTP_TO=recipient@example.com
```

## Running the System

### Start the Web Application

```bash
# Main application entry point
python app.py
```

The web interface will be available at `http://localhost:5000`

### Manual Crawler Execution

```bash
# Test individual spider
python runspider-test.py

# Run all spiders (also triggered via web UI)
python run_spiders.py
```

### Test Push Notifications

```bash
# Test alert configuration
python test_push_task.py
```

## Core Usage Patterns

### 1. Crawler Data Collection

```python
# Example: Custom spider for new platform
from hotsearchcrawler.spiders.base_spider import BaseHotSearchSpider

class CustomPlatformSpider(BaseHotSearchSpider):
    name = 'custom_platform'
    
    def start_requests(self):
        yield scrapy.Request(
            url='https://api.example.com/hot/list',
            callback=self.parse
        )
    
    def parse(self, response):
        data = response.json()
        for item in data['list']:
            yield {
                'platform': 'CustomPlatform',
                'title': item['title'],
                'hot_value': item['heat'],
                'url': item['link'],
                'rank': item['rank'],
                'crawl_time': datetime.now()
            }
```

### 2. Query Hot Topics via API

```python
from hotsearch_analysis_agent.api import query_hot_topics

# Natural language query
results = query_hot_topics(
    query="Show me AI-related hot topics from Weibo today",
    platforms=['weibo'],
    date_range='today'
)

for topic in results:
    print(f"{topic['rank']}. {topic['title']} (热度: {topic['hot_value']})")
```

### 3. Topic Clustering Analysis

```python
from hotsearch_analysis_agent.analyzer import TopicClusterAnalyzer

analyzer = TopicClusterAnalyzer(model='pangu')

# Analyze topics from multiple platforms
topics = get_recent_topics(hours=24)
clusters = analyzer.cluster_topics(topics)

for cluster_id, cluster_data in clusters.items():
    print(f"\n主题簇 {cluster_id}: {cluster_data['theme']}")
    print(f"相关话题数: {len(cluster_data['topics'])}")
    print(f"情感倾向: {cluster_data['sentiment']}")
```

### 4. Sentiment Analysis

```python
from hotsearch_analysis_agent.analyzer import SentimentAnalyzer

analyzer = SentimentAnalyzer()

# Analyze sentiment for a topic
topic_data = {
    'title': 'GPT-6遭提前曝光, 2M超长上下文来了',
    'content': '社区流传信息显示，下一代GPT模型或支持200万token...',
    'comments': ['真的吗', '期待', '不敢相信']
}

sentiment = analyzer.analyze(topic_data)
print(f"情感倾向: {sentiment['polarity']}")  # positive/negative/neutral
print(f"置信度: {sentiment['confidence']}")
```

### 5. Automated Push Tasks

```python
from hotsearch_analysis_agent.push import PushTaskManager

task_manager = PushTaskManager()

# Create monitoring task
task_manager.create_task(
    name='AI Technology Monitor',
    keywords=['人工智能', 'GPT', '大模型', 'AI'],
    platforms=['weibo', 'zhihu', 'bilibili'],
    threshold={'hot_value': 100000},  # Trigger when heat > 100k
    channels=['wechat_work', 'telegram'],
    schedule='*/30 * * * *',  # Every 30 minutes
    analysis_options={
        'enable_clustering': True,
        'enable_sentiment': True,
        'generate_report': True
    }
)

# Start task scheduler
task_manager.start_scheduler()
```

### 6. Generate Analysis Report

```python
from hotsearch_analysis_agent.report import ReportGenerator

generator = ReportGenerator(llm_model='pangu')

# Generate comprehensive report
report = generator.generate(
    query='人工智能与前沿科技',
    time_range='last_24h',
    include_sections=[
        'key_findings',
        'detailed_news',
        'trend_analysis',
        'propagation_analysis'
    ]
)

# Export to markdown
with open('report.md', 'w', encoding='utf-8') as f:
    f.write(report.to_markdown())

# Push via configured channels
report.push_to_channels(['email', 'wechat_work'])
```

## Advanced Configurations

### Custom LLM Integration

```python
# hotsearch_analysis_agent/llm_client.py
from openai import OpenAI

class CustomLLMClient:
    def __init__(self):
        self.client = OpenAI(
            api_key=os.getenv('OPENAI_API_KEY'),
            base_url=os.getenv('OPENAI_API_BASE')
        )
    
    def analyze_topic(self, topic_text, analysis_type='sentiment'):
        prompt = self._build_prompt(topic_text, analysis_type)
        
        response = self.client.chat.completions.create(
            model=os.getenv('MODEL_NAME', 'gpt-4'),
            messages=[
                {"role": "system", "content": "你是舆情分析专家"},
                {"role": "user", "content": prompt}
            ],
            temperature=0.3
        )
        
        return response.choices[0].message.content
```

### Database Query Helpers

```python
from hotsearch_analysis_agent.database import DatabaseManager

db = DatabaseManager()

# Get trending topics with filters
topics = db.query_topics(
    platforms=['weibo', 'douyin'],
    min_hot_value=50000,
    keywords=['科技', '人工智能'],
    start_date='2026-05-01',
    end_date='2026-05-18',
    limit=100
)

# Get topic history for trend analysis
history = db.get_topic_history(
    topic_id='topic_123',
    time_range='7d'
)
```

## Troubleshooting

### Crawler Issues

**Problem:** Browser driver not found
```bash
# Solution: Verify driver installation and PATH
which chromedriver  # Unix
where chromedriver  # Windows

# Or specify driver path in settings
CHROME_DRIVER_PATH = '/usr/local/bin/chromedriver'
```

**Problem:** Platform returns 403/blocked
```bash
# Solution: Add valid cookies in settings.py or use proxy
DOWNLOADER_MIDDLEWARES = {
    'scrapy.downloadermiddlewares.useragent.UserAgentMiddleware': None,
    'scrapy_user_agents.middlewares.RandomUserAgentMiddleware': 400,
}
```

### Database Connection Errors

```python
# Verify connection
from hotsearch_analysis_agent.database import test_connection

if not test_connection():
    print("Check DB_HOST, DB_PORT, DB_USER, DB_PASSWORD in .env")
    print("Ensure MySQL service is running")
```

### LLM API Errors

```bash
# Test API connectivity
curl -X POST $OPENAI_API_BASE/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d '{"model":"gpt-3.5-turbo","messages":[{"role":"user","content":"test"}]}'

# For Pangu local model, check model path
ls -lh $PANGU_MODEL_PATH/*.bin
```

### Push Notification Failures

```python
# Test individual channels
from hotsearch_analysis_agent.push import test_push_channels

test_push_channels(['wechat_work', 'telegram', 'email'])
```

## Project Structure Reference

```
.
├── app.py                          # Main application entry
├── hotsearch_analysis_agent/       # Analysis system
│   ├── api.py                      # Query API
│   ├── analyzer.py                 # Clustering & sentiment
│   ├── database.py                 # Database operations
│   ├── llm_client.py              # LLM integration
│   ├── push.py                     # Push notifications
│   └── report.py                   # Report generation
├── hotsearchcrawler/              # Crawler cluster (separate)
│   ├── spiders/                   # Platform spiders
│   ├── settings.py                # Crawler config
│   └── pipelines.py               # Data pipelines
├── run_spiders.py                 # Crawler launcher
├── test_push_task.py              # Test push tasks
└── init.py                        # Database initialization
```

This skill enables AI coding agents to help developers deploy, configure, and extend the public opinion analytics system for monitoring Chinese social platforms with LLM-powered analysis.
