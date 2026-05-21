---
name: llm-public-opinion-analytics-assistant
description: Comprehensive guide for deploying and using the LLM-Based Intelligent Public Opinion Analytics Assistant with multi-platform hot search data crawling, sentiment analysis, and alert pushing
triggers:
  - how do I set up the public opinion analytics assistant
  - deploy hot search crawler with LLM analysis
  - configure sentiment analysis for social media platforms
  - set up multi-channel alert pushing for trending topics
  - analyze hot topics from weibo bilibili and other platforms
  - crawl and analyze trending news with large language models
  - implement topic clustering and sentiment detection
  - configure telegram wechat email alerts for hot searches
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

The LLM-Based Intelligent Public Opinion Analytics Assistant is a comprehensive public opinion monitoring system that:

- **Crawls 26 hot search lists** from 15 major Chinese platforms (Weibo, Bilibili, Toutiao, Zhihu, etc.)
- **Analyzes trending topics** using large language models (LLM) with support for long-context analysis
- **Performs topic clustering** and sentiment analysis on news content, including video-based content
- **Provides conversational interface** for natural language queries about trending topics
- **Sends multi-channel alerts** via Email, WeChat Work, Enterprise WeChat, and Telegram
- **Supports browser automation** for deep content extraction from detail pages

The system consists of two main components:
1. **Crawler cluster** (`hotsearchcrawler`) - Scrapy-based distributed crawler
2. **Analysis agent** (`hotsearch_analysis_agent`) - LLM-powered analytics backend with web UI

## Installation and Setup

### Prerequisites

1. **Python Environment**
```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

2. **MySQL Database**
```bash
# Install MySQL 5.7+ or 8.0+
# Create database
mysql -u root -p
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

3. **Browser Driver Setup (Critical)**

For Edge:
```bash
# Check your Edge version: edge://settings/help
# Download matching EdgeDriver from:
# https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Linux/macOS - place in PATH
sudo mv msedgedriver /usr/local/bin/
sudo chmod +x /usr/local/bin/msedgedriver

# Windows - add to PATH or place in project root
# Verify installation
msedgedriver --version
```

For Chrome:
```bash
# Download from https://chromedriver.chromium.org/
# Match your Chrome version: chrome://version/

# Linux/macOS
sudo mv chromedriver /usr/local/bin/
sudo chmod +x /usr/local/bin/chromedriver

# Verify
chromedriver --version
```

### Database Initialization

```python
# Reference: init.py
import pymysql

# Database connection
connection = pymysql.connect(
    host='localhost',
    user='root',
    password='${DB_PASSWORD}',
    database='hotsearch_db',
    charset='utf8mb4'
)

# Create hot search table
create_table_sql = """
CREATE TABLE IF NOT EXISTS hot_searches (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    platform VARCHAR(50) NOT NULL,
    rank_index INT,
    title VARCHAR(500) NOT NULL,
    url TEXT,
    hot_value VARCHAR(100),
    category VARCHAR(100),
    description TEXT,
    detail_content LONGTEXT,
    crawl_time DATETIME NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_platform_time (platform, crawl_time),
    INDEX idx_title (title(255)),
    INDEX idx_crawl_time (crawl_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
"""

with connection.cursor() as cursor:
    cursor.execute(create_table_sql)
connection.commit()
```

### Configuration Files

#### 1. Crawler Settings (`hotsearchcrawler/settings.py`)

```python
# MySQL Configuration
MYSQL_CONFIG = {
    'host': 'localhost',
    'port': 3306,
    'user': 'root',
    'password': '${DB_PASSWORD}',
    'database': 'hotsearch_db',
    'charset': 'utf8mb4'
}

# Concurrent settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
CONCURRENT_REQUESTS_PER_DOMAIN = 8

# Browser automation (for detail page crawling)
SELENIUM_DRIVER_NAME = 'edge'  # or 'chrome'
SELENIUM_DRIVER_EXECUTABLE_PATH = '/usr/local/bin/msedgedriver'
SELENIUM_BROWSER_EXECUTABLE_PATH = '/usr/bin/microsoft-edge'

# Optional: Platform cookies for authenticated access
COOKIES_CONFIG = {
    'weibo': 'your_weibo_cookie_string',
    'bilibili': 'your_bilibili_cookie_string'
}
```

#### 2. Analysis Agent Configuration (`.env`)

```bash
# Database
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=${DB_PASSWORD}
DB_NAME=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=${YOUR_LLM_API_KEY}
OPENAI_API_BASE=https://your-llm-endpoint.com/v1
MODEL_NAME=pangu-7b  # or gpt-4, deepseek-v4, etc.

# Huawei Pangu Model (recommended for Chinese text)
PANGU_API_KEY=${PANGU_API_KEY}
PANGU_ENDPOINT=https://pangu-api.huaweicloud.com

# Push Notification Channels
# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=${EMAIL_ADDRESS}
SMTP_PASSWORD=${EMAIL_APP_PASSWORD}
SMTP_FROM=${EMAIL_ADDRESS}

# WeChat Work Robot
WECHAT_WORK_WEBHOOK=${WECHAT_WEBHOOK_URL}

# Enterprise WeChat App
ENTERPRISE_WECHAT_CORP_ID=${CORP_ID}
ENTERPRISE_WECHAT_APP_SECRET=${APP_SECRET}
ENTERPRISE_WECHAT_AGENT_ID=${AGENT_ID}

# Telegram Bot
TELEGRAM_BOT_TOKEN=${BOT_TOKEN}
TELEGRAM_CHAT_ID=${CHAT_ID}
```

## Running the System

### Start Crawler Cluster

```python
# Test single spider
python runspider-test.py

# Run all spiders via web UI
python app.py
# Navigate to http://localhost:5000
# Use keyboard shortcuts in UI to start/stop crawlers
```

### Start Analysis Agent

```python
# Main application
python app.py

# The web UI provides:
# - Real-time hot search monitoring
# - Conversational query interface
# - Topic clustering visualization
# - Sentiment analysis dashboard
# - Push task management
```

### Test Push Notifications

```python
# test_push_task.py
from hotsearch_analysis_agent.push_manager import PushManager

push_mgr = PushManager()

# Test email push
push_mgr.send_email(
    subject="Hot Topic Alert: AI Technology Trends",
    body=analysis_report,
    recipients=["user@example.com"]
)

# Test WeChat Work
push_mgr.send_wechat_work(
    content=analysis_report,
    mentioned_list=["@all"]
)

# Test Telegram
push_mgr.send_telegram(
    message=analysis_report
)
```

## Key API and Usage Patterns

### Crawler Spider Example

```python
# hotsearchcrawler/spiders/weibo_spider.py
import scrapy
from hotsearchcrawler.items import HotSearchItem

class WeiboHotSpider(scrapy.Spider):
    name = 'weibo_hot'
    start_urls = ['https://s.weibo.com/top/summary']
    
    def parse(self, response):
        for rank, item in enumerate(response.css('tbody tr'), 1):
            title = item.css('td.td-02 a::text').get()
            url = item.css('td.td-02 a::attr(href)').get()
            hot_value = item.css('td.td-02 span::text').get()
            
            if title:
                hot_item = HotSearchItem()
                hot_item['platform'] = 'weibo'
                hot_item['rank_index'] = rank
                hot_item['title'] = title.strip()
                hot_item['url'] = response.urljoin(url) if url else None
                hot_item['hot_value'] = hot_value
                hot_item['crawl_time'] = datetime.now()
                
                # Request detail page for deep content
                if hot_item['url']:
                    yield scrapy.Request(
                        hot_item['url'],
                        callback=self.parse_detail,
                        meta={'item': hot_item}
                    )
    
    def parse_detail(self, response):
        item = response.meta['item']
        # Extract article content or video info
        item['detail_content'] = response.css('article.article::text').getall()
        yield item
```

### LLM Analysis Pipeline

```python
# hotsearch_analysis_agent/analyzer.py
from openai import OpenAI
import os

class HotSearchAnalyzer:
    def __init__(self):
        self.client = OpenAI(
            api_key=os.getenv('OPENAI_API_KEY'),
            base_url=os.getenv('OPENAI_API_BASE')
        )
        self.model = os.getenv('MODEL_NAME', 'gpt-4')
    
    def analyze_sentiment(self, text: str) -> dict:
        """Analyze sentiment of hot search topic"""
        prompt = f"""
        分析以下热搜话题的情感倾向,输出JSON格式:
        {{
            "sentiment": "positive/negative/neutral",
            "confidence": 0.0-1.0,
            "key_emotions": ["emotion1", "emotion2"],
            "reasoning": "分析理由"
        }}
        
        话题: {text}
        """
        
        response = self.client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": "你是专业的舆情分析专家"},
                {"role": "user", "content": prompt}
            ],
            temperature=0.3
        )
        
        return response.choices[0].message.content
    
    def cluster_topics(self, topics: list) -> dict:
        """Cluster related hot search topics"""
        topics_text = "\n".join([f"{i+1}. {t['title']}" for i, t in enumerate(topics)])
        
        prompt = f"""
        对以下热搜话题进行聚类分析,识别核心主题:
        
        {topics_text}
        
        输出JSON格式:
        {{
            "clusters": [
                {{
                    "theme": "主题名称",
                    "topics": [1, 3, 5],
                    "summary": "主题总结"
                }}
            ]
        }}
        """
        
        response = self.client.chat.completions.create(
            model=self.model,
            messages=[{"role": "user", "content": prompt}],
            temperature=0.5
        )
        
        return response.choices[0].message.content
    
    def generate_report(self, query: str, topics: list) -> str:
        """Generate comprehensive analysis report"""
        context = "\n\n".join([
            f"[{i+1}] {t['title']}\n{t['url']}\n摘要: {t.get('description', 'N/A')}"
            for i, t in enumerate(topics)
        ])
        
        prompt = f"""
        基于以下热搜数据,生成关于"{query}"的舆情分析报告:
        
        {context}
        
        报告应包含:
        1. 核心发现与数据亮点
        2. 详细新闻内容梳理
        3. 分析与总结
        4. 信息传播特点
        5. 综合结论
        """
        
        response = self.client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": "你是资深舆情分析师,擅长撰写专业分析报告"},
                {"role": "user", "content": prompt}
            ],
            temperature=0.7,
            max_tokens=4000
        )
        
        return response.choices[0].message.content
```

### Query Interface Usage

```python
# hotsearch_analysis_agent/query_handler.py
from analyzer import HotSearchAnalyzer
from database import HotSearchDB

class QueryHandler:
    def __init__(self):
        self.db = HotSearchDB()
        self.analyzer = HotSearchAnalyzer()
    
    def handle_query(self, user_query: str) -> dict:
        """Process natural language query"""
        
        # Extract search intent and keywords
        keywords = self._extract_keywords(user_query)
        time_range = self._extract_time_range(user_query)
        
        # Query database
        results = self.db.search_hot_topics(
            keywords=keywords,
            time_range=time_range,
            platforms=None  # All platforms
        )
        
        # Perform analysis
        if "情感" in user_query or "sentiment" in user_query.lower():
            analysis = [self.analyzer.analyze_sentiment(r['title']) for r in results]
        elif "聚类" in user_query or "cluster" in user_query.lower():
            analysis = self.analyzer.cluster_topics(results)
        else:
            analysis = self.analyzer.generate_report(user_query, results)
        
        return {
            'query': user_query,
            'results': results,
            'analysis': analysis,
            'timestamp': datetime.now().isoformat()
        }
```

### Push Task Configuration

```python
# hotsearch_analysis_agent/push_scheduler.py
from apscheduler.schedulers.background import BackgroundScheduler
from push_manager import PushManager
from query_handler import QueryHandler

class PushScheduler:
    def __init__(self):
        self.scheduler = BackgroundScheduler()
        self.push_mgr = PushManager()
        self.query_handler = QueryHandler()
    
    def add_push_task(self, task_config: dict):
        """
        Add scheduled push task
        
        task_config = {
            'name': '人工智能热点监控',
            'query': '人工智能 OR AI OR 大模型',
            'schedule': 'cron',
            'cron_expr': '0 9,18 * * *',  # 9AM and 6PM daily
            'channels': ['email', 'wechat_work'],
            'recipients': ['team@company.com']
        }
        """
        
        def execute_task():
            # Execute query and analysis
            result = self.query_handler.handle_query(task_config['query'])
            report = result['analysis']
            
            # Send via configured channels
            if 'email' in task_config['channels']:
                self.push_mgr.send_email(
                    subject=f"热点分析报告: {task_config['name']}",
                    body=report,
                    recipients=task_config['recipients']
                )
            
            if 'wechat_work' in task_config['channels']:
                self.push_mgr.send_wechat_work(content=report)
            
            if 'telegram' in task_config['channels']:
                self.push_mgr.send_telegram(message=report)
        
        # Add to scheduler
        self.scheduler.add_job(
            execute_task,
            trigger='cron',
            **self._parse_cron(task_config['cron_expr']),
            id=task_config['name']
        )
    
    def start(self):
        self.scheduler.start()
```

## Configuration Patterns

### Multi-Platform Crawler Configuration

```python
# hotsearchcrawler/settings.py
SPIDER_MODULES = ['hotsearchcrawler.spiders']

# Platform-specific settings
PLATFORM_CONFIGS = {
    'weibo': {
        'enabled': True,
        'schedule': '*/15 * * * *',  # Every 15 minutes
        'priority': 10
    },
    'bilibili': {
        'enabled': True,
        'schedule': '*/30 * * * *',
        'priority': 8
    },
    'zhihu': {
        'enabled': True,
        'schedule': '0 */1 * * *',  # Hourly
        'priority': 7
    },
    'toutiao': {
        'enabled': True,
        'schedule': '*/20 * * * *',
        'priority': 9
    }
}

# Middleware pipeline
DOWNLOADER_MIDDLEWARES = {
    'hotsearchcrawler.middlewares.SeleniumMiddleware': 543,
    'hotsearchcrawler.middlewares.RotateUserAgentMiddleware': 400,
    'hotsearchcrawler.middlewares.RetryMiddleware': 550,
}

ITEM_PIPELINES = {
    'hotsearchcrawler.pipelines.MySQLPipeline': 300,
    'hotsearchcrawler.pipelines.DuplicateFilterPipeline': 200,
}
```

### LLM Model Selection

```python
# For Huawei Pangu (recommended for Chinese)
# .env
MODEL_NAME=pangu-embedded-7b
OPENAI_API_BASE=https://ai.gitcode.com/api/v1

# For OpenAI
MODEL_NAME=gpt-4
OPENAI_API_BASE=https://api.openai.com/v1

# For DeepSeek
MODEL_NAME=deepseek-chat
OPENAI_API_BASE=https://api.deepseek.com/v1

# For local deployment
MODEL_NAME=your-local-model
OPENAI_API_BASE=http://localhost:8000/v1
```

## Troubleshooting

### Browser Driver Issues

```bash
# Error: selenium.common.exceptions.WebDriverException
# Solution: Verify driver and browser versions match

# Check browser version
google-chrome --version  # or microsoft-edge --version

# Download matching driver
# Chrome: https://chromedriver.chromium.org/downloads
# Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Verify driver is executable
chmod +x /usr/local/bin/chromedriver
chromedriver --version
```

### Database Connection Errors

```python
# Error: pymysql.err.OperationalError: (2003, "Can't connect to MySQL")
# Solution: Check MySQL service and credentials

import pymysql

try:
    conn = pymysql.connect(
        host='localhost',
        user='root',
        password=os.getenv('DB_PASSWORD'),
        database='hotsearch_db'
    )
    print("Database connection successful")
except pymysql.err.OperationalError as e:
    print(f"Connection failed: {e}")
    # Check: MySQL running? Password correct? Database exists?
```

### LLM API Timeout

```python
# Error: openai.error.Timeout
# Solution: Increase timeout and implement retry logic

from openai import OpenAI
from tenacity import retry, stop_after_attempt, wait_exponential

class RobustAnalyzer:
    def __init__(self):
        self.client = OpenAI(
            api_key=os.getenv('OPENAI_API_KEY'),
            timeout=60.0  # Increase timeout to 60 seconds
        )
    
    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=4, max=10)
    )
    def analyze_with_retry(self, text: str):
        return self.client.chat.completions.create(
            model=os.getenv('MODEL_NAME'),
            messages=[{"role": "user", "content": text}],
            timeout=60
        )
```

### Crawler Rate Limiting

```python
# Error: 429 Too Many Requests or IP blocked
# Solution: Implement rate limiting and proxy rotation

# hotsearchcrawler/settings.py
AUTOTHROTTLE_ENABLED = True
AUTOTHROTTLE_START_DELAY = 2
AUTOTHROTTLE_MAX_DELAY = 10
AUTOTHROTTLE_TARGET_CONCURRENCY = 1.0

# Use proxy pool
DOWNLOADER_MIDDLEWARES = {
    'hotsearchcrawler.middlewares.ProxyMiddleware': 410,
}

# Implement proxy middleware
class ProxyMiddleware:
    def __init__(self):
        self.proxies = [
            'http://proxy1:8080',
            'http://proxy2:8080',
        ]
        self.current = 0
    
    def process_request(self, request, spider):
        request.meta['proxy'] = self.proxies[self.current]
        self.current = (self.current + 1) % len(self.proxies)
```

### Memory Issues with Long Context

```python
# Error: CUDA out of memory or system memory exhausted
# Solution: Implement chunking for long documents

def analyze_long_content(self, content: str, max_chunk_size: int = 4000):
    """Analyze long content by chunking"""
    chunks = [content[i:i+max_chunk_size] 
              for i in range(0, len(content), max_chunk_size)]
    
    summaries = []
    for chunk in chunks:
        summary = self.client.chat.completions.create(
            model=self.model,
            messages=[{
                "role": "user",
                "content": f"总结以下内容:\n{chunk}"
            }],
            max_tokens=500
        )
        summaries.append(summary.choices[0].message.content)
    
    # Combine summaries
    final_summary = self.client.chat.completions.create(
        model=self.model,
        messages=[{
            "role": "user",
            "content": f"整合以下摘要:\n\n" + "\n\n".join(summaries)
        }]
    )
    
    return final_summary.choices[0].message.content
```

## Best Practices

1. **Use environment variables** for all sensitive configuration
2. **Implement proper error handling** in both crawler and analysis pipelines
3. **Monitor crawler health** via web UI keyboard shortcuts
4. **Cache LLM responses** for repeated queries to reduce costs
5. **Schedule push tasks** during off-peak hours to avoid rate limits
6. **Regular database cleanup** of old hot search data to maintain performance
7. **Test push channels** individually before production deployment
