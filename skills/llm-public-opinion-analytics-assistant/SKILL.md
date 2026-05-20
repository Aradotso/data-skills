---
name: llm-public-opinion-analytics-assistant
description: Multi-platform public opinion monitoring system with 26 hot lists from 15 platforms, LLM-powered sentiment analysis, topic clustering, and multi-channel alerts
triggers:
  - how do i set up public opinion monitoring
  - analyze hot topics across social media platforms
  - configure multi-platform sentiment analysis
  - create hot topic alert tasks
  - crawl weibo bilibili trending lists
  - analyze public sentiment with llm
  - set up telegram wechat opinion alerts
  - cluster related topics from news feeds
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is an intelligent public opinion analytics assistant that integrates real-time data from **26 hot lists across 15 mainstream platforms** (Weibo, Bilibili, Douyin, Baidu, Zhihu, etc.) with LLM analysis capabilities. It provides conversational hot search queries, topic-specific searches, topic clustering analysis, and sentiment analysis. The system supports hotkey-controlled crawler management, multi-platform data querying with direct links, and multi-channel push notifications (Email, WeChat, Enterprise WeChat, Telegram).

## Installation

### Prerequisites

1. **Browser Driver Setup** (required for content extraction):
   ```bash
   # For Chrome
   # Download from https://chromedriver.chromium.org/
   # Match your Chrome version
   
   # For Edge
   # Download from https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/
   
   # Add driver to PATH
   export PATH=$PATH:/path/to/driver/directory
   
   # Verify installation
   chromedriver --version
   ```

2. **MySQL Database**:
   ```sql
   CREATE DATABASE public_opinion CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
   ```

3. **Python Environment**:
   ```bash
   # Create virtual environment
   python -m venv venv
   source venv/bin/activate  # On Windows: venv\Scripts\activate
   
   # Install dependencies
   pip install -r requirements.txt
   ```

### Project Structure

```
├── hotsearch_analysis_agent/  # Analysis system
├── hotsearchcrawler/          # Crawler cluster (independent)
├── app.py                     # Main application entry
├── run_spiders.py            # Crawler launcher
├── test_push_task.py         # Push notification testing
├── runspider-test.py         # Crawler testing
└── init.py                   # Database initialization reference
```

## Configuration

### Database Configuration

Create `.env` file in project root:

```bash
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=public_opinion

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
OPENAI_MODEL=gpt-4

# Or use Huawei Pangu Model (recommended for Chinese text)
PANGU_API_KEY=your_pangu_key
PANGU_API_BASE=https://pangu.huaweicloud.com/v1
```

### Crawler Settings

Edit `hotsearchcrawler/settings.py`:

```python
# MySQL settings
MYSQL_HOST = os.getenv('MYSQL_HOST', 'localhost')
MYSQL_PORT = int(os.getenv('MYSQL_PORT', 3306))
MYSQL_USER = os.getenv('MYSQL_USER', 'root')
MYSQL_PASSWORD = os.getenv('MYSQL_PASSWORD', '')
MYSQL_DATABASE = os.getenv('MYSQL_DATABASE', 'public_opinion')

# Optional: Platform-specific cookies
COOKIES = {
    'weibo': 'your_cookie_string',
    'bilibili': 'your_cookie_string'
}

# Crawler settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
ROBOTSTXT_OBEY = False
```

### Push Notification Configuration

Configure in `hotsearch_analysis_agent/config/push_config.py`:

```python
# Email configuration
EMAIL_CONFIG = {
    'smtp_server': 'smtp.gmail.com',
    'smtp_port': 587,
    'sender': os.getenv('EMAIL_SENDER'),
    'password': os.getenv('EMAIL_PASSWORD'),
    'use_tls': True
}

# Enterprise WeChat Bot
WECHAT_WORK_BOT = {
    'webhook_url': os.getenv('WECHAT_WEBHOOK_URL')
}

# Enterprise WeChat Application
WECHAT_WORK_APP = {
    'corp_id': os.getenv('WECHAT_CORP_ID'),
    'agent_id': os.getenv('WECHAT_AGENT_ID'),
    'secret': os.getenv('WECHAT_SECRET')
}

# Telegram Bot
TELEGRAM_CONFIG = {
    'bot_token': os.getenv('TELEGRAM_BOT_TOKEN'),
    'chat_id': os.getenv('TELEGRAM_CHAT_ID')
}
```

## Database Initialization

```python
# Reference: init.py
from hotsearch_analysis_agent.database import init_database

# Initialize database schema
init_database()

# Tables created:
# - hot_search_items: Stores crawled hot search data
# - analysis_results: LLM analysis results
# - topic_clusters: Topic clustering results
# - sentiment_analysis: Sentiment scores
# - push_tasks: Alert task configurations
# - push_logs: Push notification history
```

## Core Usage

### Starting the Application

```bash
# Start main application (includes web UI)
python app.py

# Application will run on http://localhost:5000
```

### Running Crawlers

```python
# Method 1: Via web UI
# Navigate to http://localhost:5000 and use hotkeys to start/stop crawlers

# Method 2: Direct execution
python run_spiders.py

# Method 3: Test specific spider
python runspider-test.py --spider=weibo
```

### Crawler Usage Examples

```python
# hotsearchcrawler/spiders/weibo_spider.py
import scrapy
from hotsearchcrawler.items import HotSearchItem

class WeiboSpider(scrapy.Spider):
    name = 'weibo'
    
    def start_requests(self):
        url = 'https://s.weibo.com/top/summary'
        yield scrapy.Request(url, callback=self.parse)
    
    def parse(self, response):
        for item in response.css('tr.item'):
            hot_item = HotSearchItem()
            hot_item['platform'] = 'weibo'
            hot_item['title'] = item.css('a::text').get()
            hot_item['url'] = item.css('a::attr(href)').get()
            hot_item['hot_value'] = item.css('.hot-value::text').get()
            hot_item['rank'] = item.css('.rank::text').get()
            yield hot_item
```

### Analysis System Usage

```python
from hotsearch_analysis_agent.agent import OpinionAnalysisAgent
from hotsearch_analysis_agent.database import get_session

# Initialize agent
agent = OpinionAnalysisAgent()

# Query hot topics
results = agent.query_hot_topics(
    platform='weibo',
    limit=10,
    date_range='today'
)

# Topic-specific search
search_results = agent.search_topics(
    keywords=['人工智能', 'AI'],
    platforms=['weibo', 'bilibili', 'zhihu']
)

# Topic clustering analysis
clusters = agent.cluster_topics(
    topic_ids=[1, 2, 3, 4, 5],
    min_similarity=0.7
)

# Sentiment analysis
sentiment = agent.analyze_sentiment(
    topic_id=123,
    include_comments=True
)
```

### LLM-Powered Analysis

```python
from hotsearch_analysis_agent.llm import LLMAnalyzer
import os

# Initialize LLM analyzer
analyzer = LLMAnalyzer(
    api_key=os.getenv('OPENAI_API_KEY'),
    api_base=os.getenv('OPENAI_API_BASE'),
    model=os.getenv('OPENAI_MODEL', 'gpt-4')
)

# Analyze topic cluster
cluster_analysis = analyzer.analyze_cluster(
    topics=[
        {'title': 'GPT-6提前曝光', 'content': '...', 'platform': 'bilibili'},
        {'title': 'DeepSeek V4采用华为算力', 'content': '...', 'platform': 'weibo'}
    ]
)

# Generate report
report = analyzer.generate_report(
    query='人工智能与前沿科技',
    topics=cluster_analysis,
    include_sentiment=True
)

print(report)
```

### Push Notification Tasks

```python
from hotsearch_analysis_agent.push import PushTaskManager
import os

# Initialize push manager
push_manager = PushTaskManager()

# Create push task
task = push_manager.create_task(
    name='AI热点监控',
    keywords=['人工智能', 'GPT', 'DeepSeek'],
    platforms=['weibo', 'bilibili', 'zhihu'],
    channels=['email', 'telegram', 'wechat_work'],
    schedule='0 9,18 * * *',  # Cron format: 9 AM and 6 PM daily
    min_hot_value=100000
)

# Send immediate push
push_manager.send_push(
    channels=['telegram'],
    content=report,
    task_id=task.id
)

# Email push example
push_manager.send_email(
    recipients=['user@example.com'],
    subject='AI热点分析报告',
    body=report,
    html=True
)

# Telegram push example
push_manager.send_telegram(
    message=report,
    parse_mode='Markdown'
)

# WeChat Work bot push
push_manager.send_wechat_bot(
    content=report,
    mentioned_list=['@all']
)
```

### Testing Push Notifications

```python
# test_push_task.py
from hotsearch_analysis_agent.push import test_all_channels
import os

# Test all configured channels
test_all_channels()

# Test specific channel
from hotsearch_analysis_agent.push import send_test_email

send_test_email(
    recipient=os.getenv('TEST_EMAIL'),
    subject='测试推送'
)
```

## API Endpoints

The web application exposes these endpoints:

```python
# GET /api/hot_topics
# Query hot topics
# Parameters: platform, limit, date_range

# POST /api/search
# Search topics by keywords
# Body: {"keywords": ["AI"], "platforms": ["weibo"]}

# POST /api/analyze/cluster
# Cluster related topics
# Body: {"topic_ids": [1, 2, 3], "min_similarity": 0.7}

# POST /api/analyze/sentiment
# Analyze sentiment
# Body: {"topic_id": 123}

# POST /api/push/create
# Create push task
# Body: {"name": "...", "keywords": [...], "schedule": "..."}

# POST /api/crawler/start
# Start crawlers
# Body: {"spiders": ["weibo", "bilibili"]}

# POST /api/crawler/stop
# Stop crawlers
```

## Common Patterns

### Complete Monitoring Workflow

```python
from hotsearch_analysis_agent import OpinionMonitor
import os

# Initialize monitor
monitor = OpinionMonitor()

# 1. Start crawlers for selected platforms
monitor.start_crawlers(['weibo', 'bilibili', 'zhihu', 'douyin'])

# 2. Set up keyword monitoring
monitor.add_keywords(['人工智能', 'ChatGPT', 'DeepSeek'])

# 3. Configure clustering
monitor.configure_clustering(
    min_similarity=0.7,
    update_interval=3600  # 1 hour
)

# 4. Set up sentiment analysis
monitor.enable_sentiment_analysis(
    platforms=['weibo', 'zhihu'],
    analyze_comments=True
)

# 5. Create push task
monitor.create_push_task(
    name='AI Daily Digest',
    channels=['email', 'telegram'],
    schedule='0 18 * * *',  # 6 PM daily
    min_hot_value=50000
)

# 6. Start monitoring
monitor.run()
```

### Custom Analysis Pipeline

```python
from hotsearch_analysis_agent.pipeline import AnalysisPipeline
from hotsearch_analysis_agent.llm import LLMAnalyzer

# Create custom pipeline
pipeline = AnalysisPipeline()

# Add processing steps
pipeline.add_step('fetch', lambda: fetch_hot_topics(limit=50))
pipeline.add_step('filter', lambda topics: filter_by_keywords(topics, ['AI', '科技']))
pipeline.add_step('cluster', lambda topics: cluster_topics(topics, min_sim=0.7))
pipeline.add_step('analyze', lambda clusters: analyze_with_llm(clusters))
pipeline.add_step('report', lambda analysis: generate_report(analysis))
pipeline.add_step('push', lambda report: send_to_channels(report, ['email']))

# Execute pipeline
result = pipeline.execute()
```

## Troubleshooting

### Crawler Issues

```python
# Issue: Browser driver not found
# Solution: Verify driver in PATH
import shutil
print(shutil.which('chromedriver'))  # Should return path

# Issue: Cookie expired for platform
# Solution: Update cookies in settings.py
# Visit platform in browser, export cookies, update COOKIES dict

# Issue: Rate limiting
# Solution: Adjust DOWNLOAD_DELAY in settings.py
DOWNLOAD_DELAY = 3  # Increase delay between requests
```

### Database Issues

```python
# Issue: Connection refused
# Solution: Check MySQL service and credentials
from hotsearch_analysis_agent.database import test_connection
test_connection()  # Returns True if successful

# Issue: Table doesn't exist
# Solution: Reinitialize database
from hotsearch_analysis_agent.database import init_database
init_database(force=True)
```

### LLM Analysis Issues

```python
# Issue: API timeout
# Solution: Increase timeout or use streaming
analyzer = LLMAnalyzer(
    timeout=60,
    use_streaming=True
)

# Issue: Context too long
# Solution: Summarize topics before analysis
from hotsearch_analysis_agent.utils import summarize_topics
summarized = summarize_topics(topics, max_length=2000)
result = analyzer.analyze_cluster(summarized)

# Issue: Chinese text garbled
# Solution: Ensure UTF-8 encoding
# Recommended: Use Huawei Pangu model for Chinese content
analyzer = LLMAnalyzer(
    api_key=os.getenv('PANGU_API_KEY'),
    api_base=os.getenv('PANGU_API_BASE'),
    model='pangu-7b'
)
```

### Push Notification Issues

```python
# Issue: Email not sending
# Solution: Check SMTP configuration and credentials
from hotsearch_analysis_agent.push import diagnose_email
diagnose_email()

# Issue: Telegram bot not responding
# Solution: Verify bot token and chat_id
from hotsearch_analysis_agent.push import test_telegram
test_telegram()

# Issue: WeChat webhook fails
# Solution: Check webhook URL validity
import requests
response = requests.post(
    os.getenv('WECHAT_WEBHOOK_URL'),
    json={'msgtype': 'text', 'text': {'content': 'test'}}
)
print(response.status_code)  # Should be 200
```

## Advanced Features

### Video Content Extraction

The system can extract content from video-based news:

```python
from hotsearch_analysis_agent.extractors import VideoContentExtractor

extractor = VideoContentExtractor()

# Extract from Bilibili video
content = extractor.extract(
    url='https://www.bilibili.com/video/BV13pSoBBEvX',
    include_comments=True
)

# Returns: title, description, tags, top comments
```

### Custom Report Templates

```python
from hotsearch_analysis_agent.reports import ReportTemplate

template = ReportTemplate()
template.add_section('summary', weight=0.3)
template.add_section('key_findings', weight=0.3)
template.add_section('detailed_news', weight=0.2)
template.add_section('analysis', weight=0.2)

report = template.generate(analysis_data)
```

This skill enables AI coding agents to effectively use the public opinion analytics system for multi-platform monitoring, LLM-powered analysis, and automated alerting.
