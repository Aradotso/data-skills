---
name: llm-public-opinion-analytics-assistant
description: Multi-platform hot search crawler and LLM-powered public opinion analysis assistant with 26 lists from 15 platforms, sentiment analysis, topic clustering, and multi-channel alert pushing
triggers:
  - analyze public opinion from social media platforms
  - crawl hot search trends from weibo bilibili and other platforms
  - set up sentiment analysis for news topics
  - configure public opinion monitoring and alerts
  - implement topic clustering for social media data
  - build a hot search aggregation system
  - create automated opinion reports with llm
  - deploy multi-platform news crawler with analysis
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is an intelligent public opinion analytics assistant that combines real-time data from **26 lists across 15 mainstream platforms** (Weibo, Bilibili, Toutiao, Baidu, etc.) with large language model analysis capabilities. It provides conversational hot search queries, topic-specific searches, topic clustering analysis, sentiment analysis, and multi-channel alert pushing (email, WeChat, Enterprise WeChat, Telegram).

Key features:
- Multi-platform hot search data crawling (15 platforms, 26 lists)
- LLM-powered analysis (sentiment, clustering, trending)
- Browser-based content extraction (including video metadata)
- Multi-channel push notifications
- Web UI with conversational interface
- Hotkey-controlled crawler management

## Installation

### Prerequisites

1. **Browser Driver Setup** (required for content extraction):
   ```bash
   # Check your Chrome/Edge version first
   # Chrome: Settings → About Chrome
   # Edge: Settings → About Microsoft Edge
   
   # Download matching ChromeDriver from:
   # https://chromedriver.chromium.org/
   # Or EdgeDriver from:
   # https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/
   
   # Place driver in system PATH or browser directory
   # Verify installation:
   chromedriver --version
   ```

2. **MySQL Database**:
   ```sql
   -- Create database
   CREATE DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
   
   -- Create main tables (reference init.py for full schema)
   CREATE TABLE hot_search_data (
       id INT AUTO_INCREMENT PRIMARY KEY,
       platform VARCHAR(50),
       title VARCHAR(500),
       url VARCHAR(1000),
       hot_value VARCHAR(100),
       rank_position INT,
       category VARCHAR(100),
       crawl_time DATETIME,
       INDEX idx_platform (platform),
       INDEX idx_crawl_time (crawl_time)
   );
   
   CREATE TABLE analysis_results (
       id INT AUTO_INCREMENT PRIMARY KEY,
       query_text TEXT,
       analysis_result LONGTEXT,
       sentiment VARCHAR(50),
       created_at DATETIME
   );
   
   CREATE TABLE push_tasks (
       id INT AUTO_INCREMENT PRIMARY KEY,
       task_name VARCHAR(200),
       query_keywords TEXT,
       push_channels VARCHAR(200),
       schedule_time VARCHAR(100),
       is_active BOOLEAN DEFAULT TRUE
   );
   ```

3. **Python Environment**:
   ```bash
   # Create virtual environment
   python -m venv venv
   source venv/bin/activate  # On Windows: venv\Scripts\activate
   
   # Install dependencies
   pip install -r requirements.txt
   ```

### Configuration

Create `.env` file in project root:

```bash
# Database Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1  # or your custom endpoint
OPENAI_MODEL=gpt-4  # or pangu model endpoint

# Optional: Platform Cookies (for authenticated access)
WEIBO_COOKIE=your_cookie_string
BILIBILI_COOKIE=your_cookie_string

# Push Notification Channels
# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
SMTP_FROM=your_email@gmail.com

# Enterprise WeChat Robot
WECHAT_ROBOT_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=your_key

# Enterprise WeChat App (for personal push)
WECHAT_APP_CORPID=your_corp_id
WECHAT_APP_SECRET=your_app_secret
WECHAT_APP_AGENTID=your_agent_id

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

Configure crawler settings in `hotsearchcrawler/settings.py`:

```python
# MySQL connection for crawler
MYSQL_CONFIG = {
    'host': os.getenv('MYSQL_HOST', 'localhost'),
    'port': int(os.getenv('MYSQL_PORT', 3306)),
    'user': os.getenv('MYSQL_USER', 'root'),
    'password': os.getenv('MYSQL_PASSWORD'),
    'database': os.getenv('MYSQL_DATABASE', 'hotsearch'),
}

# Crawler settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
ROBOTSTXT_OBEY = False

# User agent rotation
USER_AGENT = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'
```

## Running the Application

### Start the Web Application

```bash
# Main application (analysis system + web UI)
python app.py

# Access web interface at: http://localhost:5000
```

### Running Crawlers

```bash
# Test individual spider
python runspider-test.py

# Run all spiders (production)
python run_spiders.py

# Or use the web UI hotkey to start/stop crawlers
```

### Test Push Notifications

```bash
# Test push task configuration
python test_push_task.py
```

## Core Usage Patterns

### 1. Crawling Hot Search Data

```python
from hotsearchcrawler.spiders.weibo_spider import WeiboSpider
from scrapy.crawler import CrawlerProcess
from scrapy.utils.project import get_project_settings

# Run single spider
settings = get_project_settings()
process = CrawlerProcess(settings)
process.crawl(WeiboSpider)
process.start()

# Data is automatically stored in MySQL
```

### 2. Querying Hot Search Data

```python
import mysql.connector
from datetime import datetime, timedelta

# Connect to database
conn = mysql.connector.connect(
    host=os.getenv('MYSQL_HOST'),
    user=os.getenv('MYSQL_USER'),
    password=os.getenv('MYSQL_PASSWORD'),
    database=os.getenv('MYSQL_DATABASE')
)
cursor = conn.cursor(dictionary=True)

# Query recent hot searches from specific platform
query = """
SELECT platform, title, url, hot_value, rank_position, crawl_time
FROM hot_search_data
WHERE platform = %s 
AND crawl_time >= %s
ORDER BY rank_position ASC
LIMIT 50
"""
cursor.execute(query, ('weibo', datetime.now() - timedelta(hours=1)))
results = cursor.fetchall()

for row in results:
    print(f"#{row['rank_position']} {row['title']} (热度: {row['hot_value']})")
```

### 3. LLM-Powered Analysis

```python
import openai
import os

# Configure OpenAI-compatible client
openai.api_key = os.getenv('OPENAI_API_KEY')
openai.api_base = os.getenv('OPENAI_API_BASE')

def analyze_sentiment(text_list):
    """Analyze sentiment of hot search topics"""
    prompt = f"""
    分析以下热搜话题的整体情感倾向,并进行分类聚类:
    
    {chr(10).join(f"{i+1}. {text}" for i, text in enumerate(text_list))}
    
    请提供:
    1. 整体情感倾向(正面/中性/负面)及占比
    2. 主要话题聚类(至少3个类别)
    3. 每个类别的代表性话题
    """
    
    response = openai.ChatCompletion.create(
        model=os.getenv('OPENAI_MODEL', 'gpt-4'),
        messages=[
            {"role": "system", "content": "你是一个专业的舆情分析助手"},
            {"role": "user", "content": prompt}
        ],
        temperature=0.7
    )
    
    return response.choices[0].message.content

# Example usage
hot_topics = [row['title'] for row in results]
analysis = analyze_sentiment(hot_topics)
print(analysis)
```

### 4. Topic Clustering

```python
def cluster_topics(topics, query_theme):
    """Cluster topics by theme using LLM"""
    prompt = f"""
    请对以下话题进行聚类分析,主题是"{query_theme}":
    
    {chr(10).join(f"- {topic}" for topic in topics)}
    
    要求:
    1. 识别3-5个主要聚类
    2. 为每个聚类命名
    3. 列出每个聚类包含的话题
    4. 分析各聚类之间的关联性
    """
    
    response = openai.ChatCompletion.create(
        model=os.getenv('OPENAI_MODEL'),
        messages=[
            {"role": "system", "content": "你是舆情分析专家,擅长话题聚类"},
            {"role": "user", "content": prompt}
        ]
    )
    
    return response.choices[0].message.content

# Usage
clusters = cluster_topics(hot_topics, "人工智能")
```

### 5. Multi-Channel Push Notifications

```python
import requests
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart

class PushService:
    """Multi-channel notification service"""
    
    @staticmethod
    def push_to_wechat_robot(content, webhook_url):
        """Push to Enterprise WeChat Robot"""
        payload = {
            "msgtype": "markdown",
            "markdown": {
                "content": content
            }
        }
        response = requests.post(webhook_url, json=payload)
        return response.json()
    
    @staticmethod
    def push_to_telegram(content, bot_token, chat_id):
        """Push to Telegram"""
        url = f"https://api.telegram.org/bot{bot_token}/sendMessage"
        payload = {
            "chat_id": chat_id,
            "text": content,
            "parse_mode": "Markdown"
        }
        response = requests.post(url, json=payload)
        return response.json()
    
    @staticmethod
    def push_to_email(subject, content, recipients):
        """Push to Email via SMTP"""
        msg = MIMEMultipart()
        msg['From'] = os.getenv('SMTP_FROM')
        msg['To'] = ', '.join(recipients)
        msg['Subject'] = subject
        
        msg.attach(MIMEText(content, 'html', 'utf-8'))
        
        with smtplib.SMTP(os.getenv('SMTP_HOST'), int(os.getenv('SMTP_PORT'))) as server:
            server.starttls()
            server.login(os.getenv('SMTP_USER'), os.getenv('SMTP_PASSWORD'))
            server.send_message(msg)

# Example: Push analysis report
push_service = PushService()

report = f"""
## 舆情分析报告 - {datetime.now().strftime('%Y-%m-%d %H:%M')}

{analysis}

---
数据来源: 15个主流平台, 26个榜单
"""

# Push to WeChat
push_service.push_to_wechat_robot(
    report, 
    os.getenv('WECHAT_ROBOT_WEBHOOK')
)

# Push to Telegram
push_service.push_to_telegram(
    report,
    os.getenv('TELEGRAM_BOT_TOKEN'),
    os.getenv('TELEGRAM_CHAT_ID')
)

# Push to Email
push_service.push_to_email(
    "每日舆情分析报告",
    report,
    ['recipient@example.com']
)
```

### 6. Scheduled Push Tasks

```python
from apscheduler.schedulers.background import BackgroundScheduler
from datetime import datetime

class PushTaskManager:
    """Manage scheduled push tasks"""
    
    def __init__(self):
        self.scheduler = BackgroundScheduler()
        self.scheduler.start()
    
    def create_task(self, task_name, keywords, channels, cron_expression):
        """Create scheduled push task"""
        def job():
            # Query related hot searches
            results = self.query_by_keywords(keywords)
            
            # Generate analysis
            analysis = analyze_sentiment([r['title'] for r in results])
            
            # Push to channels
            if 'wechat' in channels:
                push_service.push_to_wechat_robot(analysis, os.getenv('WECHAT_ROBOT_WEBHOOK'))
            if 'telegram' in channels:
                push_service.push_to_telegram(analysis, os.getenv('TELEGRAM_BOT_TOKEN'), os.getenv('TELEGRAM_CHAT_ID'))
            if 'email' in channels:
                push_service.push_to_email(f"{task_name} - 分析报告", analysis, ['admin@example.com'])
        
        # Parse cron (e.g., "0 9 * * *" for daily 9am)
        self.scheduler.add_job(job, 'cron', **self._parse_cron(cron_expression))
    
    def query_by_keywords(self, keywords):
        """Query hot searches by keywords"""
        conn = mysql.connector.connect(**MYSQL_CONFIG)
        cursor = conn.cursor(dictionary=True)
        
        keyword_conditions = ' OR '.join([f"title LIKE %s" for _ in keywords])
        query = f"""
        SELECT * FROM hot_search_data 
        WHERE ({keyword_conditions})
        AND crawl_time >= DATE_SUB(NOW(), INTERVAL 24 HOUR)
        ORDER BY crawl_time DESC
        """
        cursor.execute(query, [f"%{kw}%" for kw in keywords])
        return cursor.fetchall()

# Usage
task_manager = PushTaskManager()
task_manager.create_task(
    task_name="AI热点监控",
    keywords=["人工智能", "ChatGPT", "大模型"],
    channels=['wechat', 'email'],
    cron_expression="0 9,21 * * *"  # Daily at 9am and 9pm
)
```

## Common Patterns

### Extract Content from News Detail Pages

```python
from selenium import webdriver
from selenium.webdriver.chrome.options import Options

def extract_article_content(url):
    """Extract full content from news page (including video metadata)"""
    options = Options()
    options.add_argument('--headless')
    options.add_argument('--disable-gpu')
    
    driver = webdriver.Chrome(options=options)
    driver.get(url)
    
    # Wait for content to load
    driver.implicitly_wait(10)
    
    # Extract text content
    content = driver.find_element_by_tag_name('body').text
    
    # Extract video metadata if present
    video_elements = driver.find_elements_by_tag_name('video')
    video_info = []
    for video in video_elements:
        video_info.append({
            'src': video.get_attribute('src'),
            'poster': video.get_attribute('poster'),
            'duration': video.get_attribute('duration')
        })
    
    driver.quit()
    
    return {
        'text': content,
        'videos': video_info
    }
```

### Aggregate Multi-Platform Data

```python
def aggregate_hot_searches(platforms=None, hours=24):
    """Aggregate hot searches from multiple platforms"""
    conn = mysql.connector.connect(**MYSQL_CONFIG)
    cursor = conn.cursor(dictionary=True)
    
    platform_filter = ""
    params = []
    
    if platforms:
        platform_filter = "AND platform IN (" + ','.join(['%s'] * len(platforms)) + ")"
        params.extend(platforms)
    
    query = f"""
    SELECT platform, title, url, hot_value, rank_position, crawl_time
    FROM hot_search_data
    WHERE crawl_time >= DATE_SUB(NOW(), INTERVAL %s HOUR)
    {platform_filter}
    ORDER BY crawl_time DESC, rank_position ASC
    """
    params.append(hours)
    
    cursor.execute(query, params)
    results = cursor.fetchall()
    
    # Group by platform
    by_platform = {}
    for row in results:
        platform = row['platform']
        if platform not in by_platform:
            by_platform[platform] = []
        by_platform[platform].append(row)
    
    return by_platform

# Get hot searches from Weibo, Bilibili, Zhihu
data = aggregate_hot_searches(['weibo', 'bilibili', 'zhihu'], hours=12)
```

## Troubleshooting

### ChromeDriver Version Mismatch
```bash
# Error: "This version of ChromeDriver only supports Chrome version XX"
# Solution: Download matching driver version
# Check Chrome version: chrome://version
# Download from: https://chromedriver.chromium.org/downloads
```

### MySQL Connection Errors
```python
# Error: "Access denied for user"
# Check .env configuration:
MYSQL_USER=root
MYSQL_PASSWORD=your_actual_password

# Error: "Unknown database 'hotsearch'"
# Create database first:
# mysql -u root -p
# CREATE DATABASE hotsearch CHARACTER SET utf8mb4;
```

### LLM API Timeout
```python
# Increase timeout for long analysis tasks
response = openai.ChatCompletion.create(
    model=os.getenv('OPENAI_MODEL'),
    messages=messages,
    timeout=120  # 120 seconds
)
```

### Cookie Expiration for Authenticated Crawling
```python
# Some platforms require valid cookies
# Update in .env or hotsearchcrawler/settings.py:
WEIBO_COOKIE='your_fresh_cookie_string'

# Get cookies from browser DevTools (F12) → Network → Request Headers
```

### Crawler Rate Limiting
```python
# In hotsearchcrawler/settings.py
DOWNLOAD_DELAY = 3  # Increase delay between requests
CONCURRENT_REQUESTS = 8  # Reduce concurrent requests
AUTOTHROTTLE_ENABLED = True
```

### Database Table Not Found
```bash
# Run initialization script
python init.py

# Or manually create tables from SQL schema in init.py
```

## Advanced Configuration

### Custom Spider for New Platform

```python
# hotsearchcrawler/spiders/custom_spider.py
import scrapy

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://example.com/hot']
    
    def parse(self, response):
        for item in response.css('.hot-item'):
            yield {
                'platform': 'custom_platform',
                'title': item.css('.title::text').get(),
                'url': item.css('a::attr(href)').get(),
                'hot_value': item.css('.hot-score::text').get(),
                'rank_position': item.css('.rank::text').get(),
                'crawl_time': datetime.now()
            }
```

### Custom Analysis Prompts

```python
# Customize analysis style in hotsearch_analysis_agent/prompts.py
SENTIMENT_ANALYSIS_PROMPT = """
你是一个专业的舆情分析师。分析以下话题的情感倾向:

{topics}

输出格式:
- 整体情感: [正面/中性/负面]
- 情感强度: [1-10]
- 关键词: [列出5个最重要的关键词]
- 风险评估: [低/中/高]
"""
```

This skill provides comprehensive guidance for deploying and using the LLM-based public opinion analytics assistant for multi-platform hot search monitoring and intelligent analysis.
