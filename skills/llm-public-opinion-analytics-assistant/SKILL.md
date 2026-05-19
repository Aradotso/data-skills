---
name: llm-public-opinion-analytics-assistant
description: Multi-platform real-time public opinion monitoring and analysis system with LLM-powered insights, sentiment analysis, and multi-channel alert notifications
triggers:
  - set up public opinion monitoring for trending topics
  - analyze sentiment across multiple social media platforms
  - configure hot topic push notifications
  - scrape trending lists from Chinese platforms
  - implement LLM-based public opinion analysis
  - monitor and cluster social media trends
  - build a real-time hot search crawler
  - create sentiment analysis reports with AI
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive public opinion monitoring system that aggregates trending topics from 15 Chinese platforms (26 ranking lists), performs LLM-powered analysis including clustering and sentiment analysis, and delivers multi-channel alerts (email, WeChat, Telegram). Built with Python, Scrapy, and integrated with Huawei Pangu or OpenAI-compatible LLMs.

## What This Project Does

- **Multi-Platform Crawling**: Scrapes real-time trending topics from Weibo, Bilibili, Douyin, Zhihu, Baidu, Toutiao, and 9 other Chinese platforms
- **LLM Analysis**: Performs topic clustering, sentiment analysis, and generates comprehensive reports using large language models
- **Detail Extraction**: Retrieves full article content including video transcripts for deeper analysis
- **Multi-Channel Alerts**: Pushes analysis reports via Enterprise WeChat, Telegram, and email
- **Web Interface**: Provides conversational interface for querying trends, searching topics, and viewing analysis
- **Scheduled Tasks**: Configurable push notifications based on custom topics and time intervals

## Installation & Setup

### Prerequisites

1. **Browser Driver Setup** (required for content scraping):

```bash
# Check your Chrome/Edge version first (Settings → About)

# Download matching driver:
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Linux/macOS - move to PATH:
sudo mv chromedriver /usr/local/bin/
sudo chmod +x /usr/local/bin/chromedriver

# Windows - add to PATH or place in project root
# Verify installation:
chromedriver --version
```

2. **MySQL Database**:

```sql
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

3. **Python Environment**:

```bash
git clone https://github.com/hmmnxkl/LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant.git
cd LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### Configuration

1. **Database Schema** - Reference `init.py` to create tables:

```python
# Key tables structure:
# - hot_search_items: Stores trending topic entries
# - news_details: Full article content and metadata
# - push_tasks: Scheduled notification configurations
# - analysis_results: LLM-generated analysis cache
```

2. **Crawler Settings** - Edit `hotsearchcrawler/settings.py`:

```python
# MySQL connection
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_user'
MYSQL_PASSWORD = 'your_password'
MYSQL_DATABASE = 'hotsearch_db'

# Optional: Platform cookies for authenticated content
COOKIES = {
    'weibo': 'your_weibo_cookie',  # Optional
    'bilibili': 'your_bilibili_cookie'  # Optional
}
```

3. **Analysis System** - Create `.env` file:

```env
# Database
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key_here
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Or use Huawei Pangu (local deployment)
# PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model
# USE_LOCAL_MODEL=true

# Push Notification Channels
# Enterprise WeChat Robot
WECHAT_ROBOT_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Enterprise WeChat App (for personal notifications)
WECHAT_CORP_ID=your_corp_id
WECHAT_CORP_SECRET=your_corp_secret
WECHAT_AGENT_ID=your_agent_id

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email (SMTP)
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
EMAIL_RECIPIENTS=recipient1@example.com,recipient2@example.com
```

## Running the System

### Start the Analysis System

```bash
# Main application with web interface
python app.py

# Access web UI at http://localhost:5000
```

### Run Crawlers

```bash
# Test single spider
python runspider-test.py

# Start all spiders (26 platform lists)
# Option 1: Via web UI (recommended) - use hotkey controls in browser
# Option 2: Direct execution
python run_spiders.py

# Crawlers run independently and populate MySQL database
```

### Test Push Notifications

```bash
# Test all configured channels
python test_push_task.py
```

## Core API Patterns

### Crawler Architecture

Each platform spider extends Scrapy's base spider:

```python
# hotsearchcrawler/spiders/weibo_spider.py
import scrapy
from hotsearchcrawler.items import HotSearchItem

class WeiboHotSpider(scrapy.Spider):
    name = 'weibo_hot'
    
    def start_requests(self):
        url = 'https://s.weibo.com/top/summary'
        yield scrapy.Request(url, callback=self.parse)
    
    def parse(self, response):
        for item in response.css('tr.td-03'):
            hot_item = HotSearchItem()
            hot_item['platform'] = 'weibo'
            hot_item['rank'] = item.css('td.td-01::text').get()
            hot_item['title'] = item.css('td.td-02 a::text').get()
            hot_item['hot_value'] = item.css('td.td-02 span::text').get()
            hot_item['url'] = response.urljoin(item.css('td.td-02 a::attr(href)').get())
            hot_item['category'] = item.css('td.td-03 i::attr(class)').get()
            yield hot_item
```

### Database Pipeline

Items are automatically stored via MySQL pipeline:

```python
# hotsearchcrawler/pipelines.py
import pymysql
from datetime import datetime

class MySQLPipeline:
    def open_spider(self, spider):
        self.conn = pymysql.connect(
            host=spider.settings.get('MYSQL_HOST'),
            port=spider.settings.get('MYSQL_PORT'),
            user=spider.settings.get('MYSQL_USER'),
            password=spider.settings.get('MYSQL_PASSWORD'),
            database=spider.settings.get('MYSQL_DATABASE'),
            charset='utf8mb4'
        )
        self.cursor = self.conn.cursor()
    
    def process_item(self, item, spider):
        sql = """
        INSERT INTO hot_search_items 
        (platform, rank, title, hot_value, url, category, crawl_time)
        VALUES (%s, %s, %s, %s, %s, %s, %s)
        """
        self.cursor.execute(sql, (
            item['platform'],
            item['rank'],
            item['title'],
            item['hot_value'],
            item['url'],
            item.get('category', ''),
            datetime.now()
        ))
        self.conn.commit()
        return item
```

### LLM Analysis Integration

```python
# hotsearch_analysis_agent/analyzer.py
import os
from openai import OpenAI

class OpinionAnalyzer:
    def __init__(self):
        self.client = OpenAI(
            api_key=os.getenv('OPENAI_API_KEY'),
            base_url=os.getenv('OPENAI_API_BASE')
        )
        self.model = os.getenv('MODEL_NAME', 'gpt-4')
    
    def analyze_sentiment(self, articles):
        """Analyze sentiment of multiple articles"""
        prompt = f"""
        分析以下新闻标题和内容的情感倾向,输出正面、负面或中性:
        
        {self._format_articles(articles)}
        
        输出格式:
        标题: [情感倾向] 简要理由
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
    
    def cluster_topics(self, topics, keyword):
        """Cluster related topics around a keyword"""
        prompt = f"""
        根据关键词"{keyword}"对以下热搜话题进行聚类分析:
        
        {self._format_topics(topics)}
        
        识别主要子主题,并为每个类别生成摘要。
        """
        
        response = self.client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": "你是专业的信息聚类分析师"},
                {"role": "user", "content": prompt}
            ],
            temperature=0.5
        )
        return response.choices[0].message.content
    
    def generate_report(self, keyword, articles, sentiment, clusters):
        """Generate comprehensive opinion analysis report"""
        prompt = f"""
        基于以下信息生成舆情分析报告:
        
        主题: {keyword}
        相关新闻数量: {len(articles)}
        情感分析结果: {sentiment}
        话题聚类: {clusters}
        
        报告应包含:
        1. 核心发现与数据亮点
        2. 详细新闻内容梳理(带链接)
        3. 分析与总结
        4. 信息传播特点
        5. 综合结论
        
        使用markdown格式,专业且易读。
        """
        
        response = self.client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": "你是资深舆情分析报告撰写专家"},
                {"role": "user", "content": prompt}
            ],
            temperature=0.7,
            max_tokens=4000
        )
        return response.choices[0].message.content
```

### Push Notification System

```python
# hotsearch_analysis_agent/push_manager.py
import requests
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart

class PushManager:
    def __init__(self, config):
        self.config = config
    
    def push_to_wechat_robot(self, report):
        """Push to Enterprise WeChat robot webhook"""
        webhook = self.config.get('WECHAT_ROBOT_WEBHOOK')
        if not webhook:
            return False
        
        data = {
            "msgtype": "markdown",
            "markdown": {
                "content": report
            }
        }
        response = requests.post(webhook, json=data)
        return response.status_code == 200
    
    def push_to_telegram(self, report):
        """Push to Telegram bot"""
        token = self.config.get('TELEGRAM_BOT_TOKEN')
        chat_id = self.config.get('TELEGRAM_CHAT_ID')
        if not token or not chat_id:
            return False
        
        url = f"https://api.telegram.org/bot{token}/sendMessage"
        data = {
            "chat_id": chat_id,
            "text": report,
            "parse_mode": "Markdown"
        }
        response = requests.post(url, json=data)
        return response.status_code == 200
    
    def push_to_email(self, subject, report):
        """Push via SMTP email"""
        smtp_server = self.config.get('SMTP_SERVER')
        smtp_user = self.config.get('SMTP_USER')
        smtp_password = self.config.get('SMTP_PASSWORD')
        recipients = self.config.get('EMAIL_RECIPIENTS', '').split(',')
        
        if not all([smtp_server, smtp_user, smtp_password, recipients]):
            return False
        
        msg = MIMEMultipart('alternative')
        msg['Subject'] = subject
        msg['From'] = smtp_user
        msg['To'] = ', '.join(recipients)
        
        html_content = self._markdown_to_html(report)
        msg.attach(MIMEText(html_content, 'html'))
        
        with smtplib.SMTP(smtp_server, self.config.get('SMTP_PORT', 587)) as server:
            server.starttls()
            server.login(smtp_user, smtp_password)
            server.send_message(msg)
        return True
```

### Scheduled Task Management

```python
# hotsearch_analysis_agent/task_scheduler.py
from apscheduler.schedulers.background import BackgroundScheduler
from datetime import datetime

class TaskScheduler:
    def __init__(self, analyzer, push_manager):
        self.scheduler = BackgroundScheduler()
        self.analyzer = analyzer
        self.push_manager = push_manager
    
    def add_push_task(self, task_config):
        """
        Add scheduled push task
        task_config: {
            'name': 'AI trends monitoring',
            'keywords': ['人工智能', '大模型'],
            'schedule': '0 9,18 * * *',  # Cron format
            'channels': ['wechat', 'telegram', 'email']
        }
        """
        def execute_task():
            # Query recent articles matching keywords
            articles = self._query_articles(task_config['keywords'])
            
            # Perform analysis
            sentiment = self.analyzer.analyze_sentiment(articles)
            clusters = self.analyzer.cluster_topics(articles, task_config['keywords'][0])
            report = self.analyzer.generate_report(
                task_config['keywords'][0], 
                articles, 
                sentiment, 
                clusters
            )
            
            # Push to configured channels
            for channel in task_config['channels']:
                if channel == 'wechat':
                    self.push_manager.push_to_wechat_robot(report)
                elif channel == 'telegram':
                    self.push_manager.push_to_telegram(report)
                elif channel == 'email':
                    subject = f"舆情分析报告: {task_config['name']}"
                    self.push_manager.push_to_email(subject, report)
        
        # Parse cron schedule
        cron_parts = task_config['schedule'].split()
        self.scheduler.add_job(
            execute_task,
            'cron',
            minute=cron_parts[0],
            hour=cron_parts[1],
            day=cron_parts[2],
            month=cron_parts[3],
            day_of_week=cron_parts[4],
            id=task_config['name']
        )
    
    def start(self):
        self.scheduler.start()
    
    def stop(self):
        self.scheduler.shutdown()
```

## Common Workflows

### 1. Query Trending Topics

```python
# Query API endpoint or database directly
import pymysql
from datetime import datetime, timedelta

def get_platform_trends(platform, hours=24):
    """Get recent trends from specific platform"""
    conn = pymysql.connect(
        host=os.getenv('MYSQL_HOST'),
        user=os.getenv('MYSQL_USER'),
        password=os.getenv('MYSQL_PASSWORD'),
        database=os.getenv('MYSQL_DATABASE')
    )
    
    cutoff_time = datetime.now() - timedelta(hours=hours)
    
    with conn.cursor() as cursor:
        sql = """
        SELECT rank, title, hot_value, url, category, crawl_time
        FROM hot_search_items
        WHERE platform = %s AND crawl_time >= %s
        ORDER BY crawl_time DESC, rank ASC
        LIMIT 50
        """
        cursor.execute(sql, (platform, cutoff_time))
        results = cursor.fetchall()
    
    conn.close()
    return results
```

### 2. Search Topics by Keyword

```python
def search_topics(keyword, platforms=None, days=7):
    """Search topics across platforms containing keyword"""
    conn = pymysql.connect(
        host=os.getenv('MYSQL_HOST'),
        user=os.getenv('MYSQL_USER'),
        password=os.getenv('MYSQL_PASSWORD'),
        database=os.getenv('MYSQL_DATABASE')
    )
    
    cutoff_time = datetime.now() - timedelta(days=days)
    
    with conn.cursor() as cursor:
        if platforms:
            platform_filter = f"AND platform IN ({','.join(['%s']*len(platforms))})"
            params = [f"%{keyword}%", cutoff_time] + platforms
        else:
            platform_filter = ""
            params = [f"%{keyword}%", cutoff_time]
        
        sql = f"""
        SELECT platform, rank, title, hot_value, url, crawl_time
        FROM hot_search_items
        WHERE title LIKE %s AND crawl_time >= %s {platform_filter}
        ORDER BY crawl_time DESC
        """
        cursor.execute(sql, params)
        results = cursor.fetchall()
    
    conn.close()
    return results
```

### 3. Create Monitoring Task

```python
# Via web UI or API
from hotsearch_analysis_agent.task_scheduler import TaskScheduler

scheduler = TaskScheduler(analyzer, push_manager)

# Add daily AI trends monitoring
scheduler.add_push_task({
    'name': 'AI_daily_report',
    'keywords': ['人工智能', 'AI', '大模型', 'GPT'],
    'schedule': '0 9 * * *',  # Daily at 9 AM
    'channels': ['wechat', 'email']
})

# Add breaking news alerts (every 2 hours)
scheduler.add_push_task({
    'name': 'breaking_news',
    'keywords': ['突发', '重大', '紧急'],
    'schedule': '0 */2 * * *',
    'channels': ['telegram']
})

scheduler.start()
```

## Supported Platforms

The crawler supports these 15 Chinese platforms with 26 ranking lists:

- **Weibo** (微博): Hot search, real-time hot
- **Bilibili** (哔哩哔哩): Trending videos, hot rankings
- **Douyin** (抖音): Hot search, trending videos
- **Zhihu** (知乎): Hot questions, hot list
- **Baidu** (百度): Real-time hot, hot search
- **Toutiao** (今日头条): Hot list
- **Douban** (豆瓣): Group discussions
- **Tieba** (贴吧): Hot topics
- **36Kr** (36氪): News trending
- **iHeima** (黑马): Tech news
- **Sina News** (新浪新闻): Breaking news
- **NetEase** (网易): Hot news
- **Tencent** (腾讯): News hot list
- **Huxiu** (虎嗅): Article trending
- **IT Home** (IT之家): Tech news

## Troubleshooting

### Crawler Issues

**Problem**: ChromeDriver version mismatch
```bash
# Solution: Download exact version matching your browser
chromedriver --version
google-chrome --version  # or microsoft-edge --version

# If mismatch, download correct version and replace
```

**Problem**: Cookie expiration causing 403 errors
```python
# Solution: Update cookies in settings.py
# Get fresh cookies from browser DevTools → Application → Cookies
COOKIES = {
    'weibo': 'updated_cookie_string'
}
```

**Problem**: Rate limiting or IP blocking
```python
# Solution: Add delays and user agents in settings.py
DOWNLOAD_DELAY = 3
RANDOMIZE_DOWNLOAD_DELAY = True
USER_AGENT = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'

# Consider using proxy pool
DOWNLOADER_MIDDLEWARES = {
    'scrapy.downloadermiddlewares.httpproxy.HttpProxyMiddleware': 110,
}
```

### LLM Analysis Issues

**Problem**: API timeout or rate limits
```python
# Solution: Add retry logic and exponential backoff
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=4, max=10))
def analyze_with_retry(self, content):
    return self.client.chat.completions.create(...)
```

**Problem**: Token limit exceeded for long articles
```python
# Solution: Chunk content or summarize first
def chunk_text(text, max_tokens=3000):
    # Simple character-based chunking (1 token ≈ 2-4 chars for Chinese)
    max_chars = max_tokens * 2
    chunks = [text[i:i+max_chars] for i in range(0, len(text), max_chars)]
    return chunks

# Analyze each chunk separately then aggregate
```

### Database Issues

**Problem**: Connection pool exhausted
```python
# Solution: Use connection pooling in settings.py
MYSQL_POOL_SIZE = 10
MYSQL_POOL_RECYCLE = 3600

# Or close connections properly
try:
    cursor.execute(sql)
finally:
    cursor.close()
    conn.close()
```

**Problem**: Character encoding errors
```sql
-- Solution: Ensure database and tables use utf8mb4
ALTER DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
ALTER TABLE hot_search_items CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### Push Notification Issues

**Problem**: WeChat robot webhook returns 93000
```python
# Solution: Check webhook URL and message format
# Ensure markdown content doesn't exceed limits (max 4096 bytes)
def truncate_content(content, max_bytes=4000):
    encoded = content.encode('utf-8')
    if len(encoded) <= max_bytes:
        return content
    return encoded[:max_bytes].decode('utf-8', errors='ignore') + '...'
```

**Problem**: Email not delivered
```python
# Solution: Check SMTP credentials and enable app passwords
# For Gmail: https://myaccount.google.com/apppasswords
# Test connection:
import smtplib
server = smtplib.SMTP('smtp.gmail.com', 587)
server.starttls()
server.login(os.getenv('SMTP_USER'), os.getenv('SMTP_PASSWORD'))
server.quit()  # Should succeed without error
```

## Performance Optimization

### Crawler Optimization

```python
# settings.py - Increase concurrency
CONCURRENT_REQUESTS = 32
CONCURRENT_REQUESTS_PER_DOMAIN = 8
REACTOR_THREADPOOL_MAXSIZE = 20

# Enable HTTP caching to reduce redundant requests
HTTPCACHE_ENABLED = True
HTTPCACHE_EXPIRATION_SECS = 3600
HTTPCACHE_DIR = 'httpcache'
```

### Database Optimization

```sql
-- Add indexes for common queries
CREATE INDEX idx_platform_time ON hot_search_items(platform, crawl_time);
CREATE INDEX idx_title_search ON hot_search_items(title(100));
CREATE INDEX idx_keyword_search ON news_details(title(100), content(100));

-- Partition large tables by date
ALTER TABLE hot_search_items PARTITION BY RANGE (TO_DAYS(crawl_time)) (
    PARTITION p_old VALUES LESS THAN (TO_DAYS('2026-01-01')),
    PARTITION p_2026_q1 VALUES LESS THAN (TO_DAYS('2026-04-01')),
    PARTITION p_2026_q2 VALUES LESS THAN (TO_DAYS('2026-07-01'))
);
```

### LLM Analysis Caching

```python
# Cache analysis results to avoid redundant API calls
import hashlib
import json

class CachedAnalyzer(OpinionAnalyzer):
    def analyze_sentiment(self, articles):
        cache_key = hashlib.md5(
            json.dumps(articles, sort_keys=True).encode()
        ).hexdigest()
        
        # Check cache in database
        cached = self._get_cached_result(cache_key)
        if cached:
            return cached
        
        # Perform analysis
        result = super().analyze_sentiment(articles)
        
        # Store in cache
        self._cache_result(cache_key, result)
        return result
```

This skill provides comprehensive coverage for deploying and using the LLM-based public opinion analytics assistant for monitoring Chinese social media platforms and generating AI-powered insights.
