---
name: llm-public-opinion-analytics-assistant
description: Use this multi-platform hot search crawler and LLM-powered public opinion analysis system with real-time data collection, sentiment analysis, and multi-channel push notifications
triggers:
  - how do I set up the public opinion analytics assistant
  - crawl hot search data from multiple platforms
  - analyze sentiment and cluster topics with LLM
  - configure push notifications for hot topics
  - use the hotsearch crawler system
  - set up Pangu model for opinion analysis
  - query and analyze trending topics across platforms
  - how to start the hotsearch spider cluster
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is a comprehensive public opinion analysis assistant that combines real-time data from **15 mainstream platforms** (26 ranking lists) with large language model analysis capabilities. It provides:

- **Multi-platform hot search crawling** via Scrapy-based distributed crawler cluster
- **Conversational query interface** for hot topic exploration
- **LLM-powered analysis**: topic clustering, sentiment analysis, theme detection
- **Multi-channel push notifications**: Email, WeChat Work, Telegram
- **Video content extraction** from news detail pages
- **Hotkey-controlled crawler** management

The system uses **Huawei Pangu LLM** (or OpenAI-compatible models) for text analysis and supports local deployment for data privacy.

## Architecture

```
├── hotsearchcrawler/          # Scrapy crawler cluster (15 platforms)
│   ├── spiders/               # Individual platform spiders
│   ├── settings.py            # Crawler configuration
│   └── middlewares.py         # Browser automation middleware
├── hotsearch_analysis_agent/  # LLM analysis system
│   ├── agent/                 # Analysis agents
│   ├── services/              # Push services
│   └── models/                # Database models
├── app.py                     # Main application entry
├── run_spiders.py             # Crawler launcher
└── .env                       # Configuration file
```

## Installation

### Prerequisites

1. **Python 3.8+**
2. **MySQL 5.7+** database
3. **Browser driver** (ChromeDriver or EdgeDriver)

### Browser Driver Setup

```bash
# For Chrome - check version first
google-chrome --version  # or chrome://version

# Download matching ChromeDriver from:
# https://chromedriver.chromium.org/

# For Edge
msedge --version

# Download matching EdgeDriver from:
# https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Place driver in PATH (Linux/Mac example)
sudo mv chromedriver /usr/local/bin/
sudo chmod +x /usr/local/bin/chromedriver

# Verify installation
chromedriver --version
```

### Environment Setup

```bash
# Clone repository
git clone https://github.com/hmmnxkl/LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant.git
cd LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Database Configuration

```bash
# Create MySQL database
mysql -u root -p

# In MySQL shell:
CREATE DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

# Initialize tables (reference init.py)
python init.py
```

### Configuration Files

Create `.env` file in project root:

```bash
# Database Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_username
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Or for Huawei Pangu Model
# PANGU_API_KEY=your_pangu_key
# PANGU_API_BASE=https://pangu-api-endpoint

# Push Service Configuration
# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password

# WeChat Work Robot
WECHAT_ROBOT_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxx

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

Update `hotsearchcrawler/settings.py`:

```python
# MySQL Configuration for Crawler
MYSQL_CONFIG = {
    'host': 'localhost',
    'port': 3306,
    'user': 'your_username',
    'password': 'your_password',
    'database': 'hotsearch',
    'charset': 'utf8mb4'
}

# Optional: Platform-specific cookies
COOKIES_DICT = {
    'weibo': 'your_weibo_cookie',
    'zhihu': 'your_zhihu_cookie'
}
```

## Usage

### Starting the Main Application

```python
# app.py - Main application entry
from hotsearch_analysis_agent import create_app

app = create_app()

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
```

```bash
# Start the web interface
python app.py

# Access at http://localhost:5000
```

### Running Crawlers

```python
# run_spiders.py - Execute all platform crawlers
from scrapy.crawler import CrawlerProcess
from scrapy.utils.project import get_project_settings
from hotsearchcrawler.spiders import *

def run_all_spiders():
    """Start all configured platform spiders"""
    settings = get_project_settings()
    process = CrawlerProcess(settings)
    
    # Add spiders
    spider_classes = [
        WeiboHotSearchSpider,
        ZhihuHotSpider,
        BilibiliHotSpider,
        DouyinHotSpider,
        ToutiaoHotSpider,
        BaiduHotSpider
        # ... 15 platforms total
    ]
    
    for spider_cls in spider_classes:
        process.crawl(spider_cls)
    
    process.start()

if __name__ == '__main__':
    run_all_spiders()
```

```bash
# Run crawlers manually
python run_spiders.py

# Or via web interface hotkey (configured in frontend)
```

### Query and Analysis API

```python
# Example: Query hot topics with LLM analysis
from hotsearch_analysis_agent.agent import OpinionAnalysisAgent
from hotsearch_analysis_agent.services import DatabaseService

# Initialize services
db_service = DatabaseService()
agent = OpinionAnalysisAgent(
    model_name=os.getenv('MODEL_NAME'),
    api_key=os.getenv('OPENAI_API_KEY')
)

# Query hot topics by keyword
def query_hot_topics(keyword: str, platform: str = None):
    """
    Query hot search data from database
    
    Args:
        keyword: Search keyword
        platform: Optional platform filter (weibo, zhihu, bilibili, etc.)
    """
    query = f"""
    SELECT title, url, hot_value, platform, created_at, content
    FROM hot_search_items
    WHERE title LIKE %s
    """
    params = [f'%{keyword}%']
    
    if platform:
        query += " AND platform = %s"
        params.append(platform)
    
    query += " ORDER BY hot_value DESC LIMIT 50"
    
    return db_service.execute_query(query, params)

# Perform sentiment analysis
def analyze_sentiment(topics: list):
    """
    Analyze sentiment of topic list using LLM
    
    Args:
        topics: List of topic dictionaries with 'title' and 'content'
    """
    prompt = f"""
    分析以下热搜话题的情感倾向,对每个话题给出:
    1. 情感极性: 正面/中性/负面
    2. 情感强度: 1-10分
    3. 关键情感词
    
    话题列表:
    {[t['title'] for t in topics]}
    """
    
    result = agent.analyze(prompt, context=topics)
    return result

# Topic clustering
def cluster_topics(topics: list):
    """
    Group related topics using LLM-based clustering
    """
    prompt = f"""
    将以下热搜话题进行聚类分组,识别共同主题:
    
    {[{'title': t['title'], 'platform': t['platform']} for t in topics]}
    
    输出JSON格式:
    {{
        "clusters": [
            {{
                "theme": "主题名称",
                "topics": ["话题1", "话题2"],
                "summary": "该聚类的简要描述"
            }}
        ]
    }}
    """
    
    return agent.analyze(prompt, output_format='json')
```

### Push Notification Configuration

```python
# test_push_task.py - Test push services
from hotsearch_analysis_agent.services import (
    EmailPushService,
    WeChatRobotService,
    TelegramPushService
)

# Email push example
email_service = EmailPushService(
    smtp_host=os.getenv('SMTP_HOST'),
    smtp_port=int(os.getenv('SMTP_PORT')),
    username=os.getenv('SMTP_USER'),
    password=os.getenv('SMTP_PASSWORD')
)

report = {
    'title': '人工智能与前沿科技热点分析',
    'time': '2026-04-07 12:32:00',
    'summary': '本报告梳理近期AI领域关键进展...',
    'topics': [...],  # Analysis results
    'clusters': [...]  # Clustered topics
}

# Send email report
email_service.send_report(
    to_addresses=['recipient@example.com'],
    subject='舆情分析报告',
    report=report
)

# WeChat Work Robot push
wechat_service = WeChatRobotService(
    webhook_url=os.getenv('WECHAT_ROBOT_WEBHOOK')
)

wechat_service.send_markdown(
    title='🔥 热点预警',
    content=f"""
    ## {report['title']}
    
    **核心发现**:
    {report['summary']}
    
    [查看完整报告](http://your-domain.com/report/{report['id']})
    """
)

# Telegram push
telegram_service = TelegramPushService(
    bot_token=os.getenv('TELEGRAM_BOT_TOKEN'),
    chat_id=os.getenv('TELEGRAM_CHAT_ID')
)

telegram_service.send_message(
    text=f"📊 *{report['title']}*\n\n{report['summary']}",
    parse_mode='Markdown'
)
```

### Custom Spider Example

```python
# hotsearchcrawler/spiders/custom_spider.py
import scrapy
from hotsearchcrawler.items import HotSearchItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    platform = 'CustomSite'
    
    def start_requests(self):
        url = 'https://example.com/hot-topics'
        yield scrapy.Request(
            url=url,
            callback=self.parse,
            dont_filter=True
        )
    
    def parse(self, response):
        """Parse hot topic list"""
        for rank, topic in enumerate(response.css('.hot-item'), 1):
            item = HotSearchItem()
            item['rank'] = rank
            item['title'] = topic.css('.title::text').get()
            item['url'] = response.urljoin(topic.css('a::attr(href)').get())
            item['hot_value'] = topic.css('.hot-score::text').get()
            item['platform'] = self.platform
            
            # Fetch detail page for content extraction
            yield scrapy.Request(
                url=item['url'],
                callback=self.parse_detail,
                meta={'item': item}
            )
    
    def parse_detail(self, response):
        """Extract content from detail page"""
        item = response.meta['item']
        
        # Extract text content
        item['content'] = ' '.join(
            response.css('article p::text').getall()
        )
        
        # Extract video info if present
        if response.css('video'):
            item['video_url'] = response.css('video source::attr(src)').get()
            item['video_title'] = response.css('video::attr(title)').get()
        
        yield item
```

## Common Patterns

### Scheduled Crawling

```python
# Using APScheduler for periodic crawling
from apscheduler.schedulers.background import BackgroundScheduler
from run_spiders import run_all_spiders

scheduler = BackgroundScheduler()

# Run crawlers every 30 minutes
scheduler.add_job(
    run_all_spiders,
    'interval',
    minutes=30,
    id='hotsearch_crawl'
)

scheduler.start()
```

### Real-time Query with Caching

```python
from functools import lru_cache
from datetime import datetime, timedelta

@lru_cache(maxsize=128)
def get_cached_topics(platform: str, time_key: str):
    """Cache topics for 5 minutes (time_key changes every 5 min)"""
    return db_service.execute_query(
        "SELECT * FROM hot_search_items WHERE platform = %s AND created_at > %s",
        [platform, datetime.now() - timedelta(hours=1)]
    )

# Generate time key that changes every 5 minutes
time_key = datetime.now().strftime('%Y%m%d%H%M')[:-1] + '0'
topics = get_cached_topics('weibo', time_key)
```

### Batch Analysis Pipeline

```python
def analyze_daily_trends():
    """Analyze full day's data and generate report"""
    # 1. Fetch all topics from last 24 hours
    topics = db_service.execute_query("""
        SELECT * FROM hot_search_items 
        WHERE created_at > DATE_SUB(NOW(), INTERVAL 24 HOUR)
    """)
    
    # 2. Cluster topics
    clusters = cluster_topics(topics)
    
    # 3. Sentiment analysis per cluster
    for cluster in clusters['clusters']:
        cluster_topics = [t for t in topics if t['title'] in cluster['topics']]
        cluster['sentiment'] = analyze_sentiment(cluster_topics)
    
    # 4. Generate and push report
    report = generate_report(clusters)
    push_to_all_channels(report)
    
    return report
```

## Troubleshooting

### Crawler Issues

**Problem**: Browser driver not found
```bash
# Solution: Verify driver in PATH
which chromedriver  # Linux/Mac
where chromedriver  # Windows

# Or set explicit path in settings.py
SELENIUM_DRIVER_EXECUTABLE_PATH = '/usr/local/bin/chromedriver'
```

**Problem**: Anti-scraping blocks (403/429 errors)
```python
# Solution: Add delays and randomization in settings.py
DOWNLOAD_DELAY = 3
RANDOMIZE_DOWNLOAD_DELAY = True
CONCURRENT_REQUESTS_PER_DOMAIN = 1

# Use rotating user agents
USER_AGENT_LIST = [
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64)...',
    'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)...'
]
```

**Problem**: Cookies expired
```python
# Solution: Update cookies in settings.py
# Extract from browser DevTools -> Application -> Cookies
COOKIES_DICT = {
    'weibo': 'SUB=xxx; SUBP=yyy',  # Update from browser
}
```

### LLM Analysis Issues

**Problem**: API rate limits
```python
# Solution: Implement exponential backoff
import time
from tenacity import retry, wait_exponential, stop_after_attempt

@retry(wait=wait_exponential(min=1, max=60), stop=stop_after_attempt(5))
def call_llm_with_retry(prompt: str):
    return agent.analyze(prompt)
```

**Problem**: Context length exceeded
```python
# Solution: Chunk large inputs
def analyze_large_dataset(topics: list, chunk_size: int = 20):
    results = []
    for i in range(0, len(topics), chunk_size):
        chunk = topics[i:i + chunk_size]
        result = agent.analyze_batch(chunk)
        results.extend(result)
    return results
```

### Database Performance

**Problem**: Slow queries on large datasets
```sql
-- Solution: Add indexes
CREATE INDEX idx_platform_time ON hot_search_items(platform, created_at);
CREATE INDEX idx_title_search ON hot_search_items(title(100));
CREATE FULLTEXT INDEX idx_content_fulltext ON hot_search_items(content);
```

### Push Service Failures

**Problem**: WeChat Work webhook fails
```python
# Solution: Verify webhook format
response = wechat_service.send({
    "msgtype": "markdown",
    "markdown": {
        "content": "content here"
    }
})
if response.status_code != 200:
    logger.error(f"WeChat push failed: {response.text}")
```

## Advanced Configuration

### Using Huawei Pangu Model Locally

```python
# Download model from: https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

from transformers import AutoModelForCausalLM, AutoTokenizer

model_path = "/path/to/openpangu-embedded-7b-model"
tokenizer = AutoTokenizer.from_pretrained(model_path)
model = AutoModelForCausalLM.from_pretrained(
    model_path,
    device_map="auto",
    torch_dtype="auto"
)

# Use in analysis agent
class PanguAnalysisAgent:
    def __init__(self, model_path: str):
        self.tokenizer = AutoTokenizer.from_pretrained(model_path)
        self.model = AutoModelForCausalLM.from_pretrained(model_path)
    
    def analyze(self, prompt: str):
        inputs = self.tokenizer(prompt, return_tensors="pt")
        outputs = self.model.generate(**inputs, max_length=2048)
        return self.tokenizer.decode(outputs[0])
```

This skill provides comprehensive coverage of the LLM-based public opinion analytics assistant for AI coding agents to effectively help developers deploy and use the system.
