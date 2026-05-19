---
name: llm-public-opinion-analytics-assistant
description: A multi-platform public opinion analysis assistant integrating 26 hot search lists from 15 platforms with LLM analysis capabilities for sentiment analysis, topic clustering, and multi-channel alert pushing.
triggers:
  - analyze public opinion from Chinese social media platforms
  - set up hot topic monitoring and alerting system
  - crawl trending topics from Weibo Bilibili and other platforms
  - implement sentiment analysis for Chinese news content
  - configure multi-channel push notifications for trending topics
  - build real-time public opinion dashboard with LLM analysis
  - extract and analyze content from video news sources
  - create automated reports for trending topics and clusters
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is a comprehensive public opinion monitoring and analysis system that integrates **26 hot search lists from 15 mainstream Chinese platforms** (Weibo, Bilibili, Zhihu, Douyin, etc.) with large language model capabilities. It provides:

- Real-time web scraping of trending topics across platforms
- Conversational interface for topic queries and searches
- LLM-powered sentiment analysis and topic clustering
- Multi-channel push notifications (Email, WeChat, Telegram, Enterprise WeChat)
- Video content extraction and analysis capabilities
- Keyboard shortcuts for crawler control

The system architecture separates the crawler cluster (`hotsearchcrawler`) from the analysis system (`hotsearch_analysis_agent`), storing data in MySQL for processing.

## Installation

### Prerequisites

1. **Browser Driver Setup** (required for Selenium-based crawlers):

```bash
# Check your Chrome/Edge version first
# Chrome: chrome://version
# Edge: edge://version

# Download matching driver from:
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Place driver in PATH or project directory
# Verify installation:
chromedriver --version
```

2. **MySQL Database**:

```bash
# Install MySQL and create database
mysql -u root -p
CREATE DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

3. **Python Environment**:

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

### Database Initialization

```python
# Reference init.py for table schemas
# Example structure for hot search table:
import pymysql

connection = pymysql.connect(
    host='localhost',
    user='your_user',
    password='your_password',
    database='hotsearch',
    charset='utf8mb4'
)

with connection.cursor() as cursor:
    # Create hot search items table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS hot_search_items (
            id INT AUTO_INCREMENT PRIMARY KEY,
            platform VARCHAR(50) NOT NULL,
            title VARCHAR(500) NOT NULL,
            url VARCHAR(1000),
            rank INT,
            heat_value VARCHAR(100),
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            INDEX idx_platform (platform),
            INDEX idx_created_at (created_at)
        ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
    """)
    
    # Create analysis results table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS analysis_results (
            id INT AUTO_INCREMENT PRIMARY KEY,
            query TEXT NOT NULL,
            result LONGTEXT,
            sentiment VARCHAR(50),
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
    """)
    
    connection.commit()
```

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```bash
# Database Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Huawei Pangu Model (alternative)
PANGU_API_KEY=your_pangu_key
PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b

# Push Notification Channels
# Email (SMTP)
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
EMAIL_RECIPIENTS=recipient1@example.com,recipient2@example.com

# Enterprise WeChat Robot
WECHAT_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Browser Driver Path (optional if in PATH)
CHROME_DRIVER_PATH=/usr/local/bin/chromedriver
```

### Crawler Configuration

Edit `hotsearchcrawler/settings.py`:

```python
# MySQL connection settings
MYSQL_SETTINGS = {
    'host': 'localhost',
    'port': 3306,
    'user': 'your_user',
    'password': 'your_password',
    'database': 'hotsearch',
    'charset': 'utf8mb4'
}

# Scrapy settings
BOT_NAME = 'hotsearchcrawler'
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
COOKIES_ENABLED = True

# Optional: Platform-specific cookies for authenticated access
WEIBO_COOKIES = {
    'SUB': 'your_weibo_cookie'
}
```

## Core Usage Patterns

### Starting the Web Application

```python
# Start the Flask-based analysis interface
python app.py

# Access at http://localhost:5000
# Features:
# - Conversational query interface
# - Real-time hot search display
# - Crawler control shortcuts
# - Analysis visualization
```

### Running Crawlers

```python
# Test individual crawler
python runspider-test.py --platform weibo

# Run all crawlers via API (triggered from frontend)
# or manually:
python run_spiders.py
```

**Crawler example** (Weibo hot search):

```python
import scrapy
from hotsearchcrawler.items import HotSearchItem

class WeiboHotSearchSpider(scrapy.Spider):
    name = 'weibo_hot'
    start_urls = ['https://s.weibo.com/top/summary']
    
    def parse(self, response):
        for rank, item in enumerate(response.css('td.td-02'), start=1):
            hot_item = HotSearchItem()
            hot_item['platform'] = 'Weibo'
            hot_item['rank'] = rank
            hot_item['title'] = item.css('a::text').get()
            hot_item['url'] = response.urljoin(item.css('a::attr(href)').get())
            hot_item['heat_value'] = item.css('span::text').get()
            
            # Fetch detail page for full content analysis
            yield scrapy.Request(
                hot_item['url'],
                callback=self.parse_detail,
                meta={'item': hot_item}
            )
    
    def parse_detail(self, response):
        item = response.meta['item']
        # Extract content including video transcripts
        item['content'] = self.extract_content(response)
        yield item
    
    def extract_content(self, response):
        # Handle text and video content
        text = response.css('article.main-text::text').getall()
        video_desc = response.css('div.video-desc::text').get()
        return ' '.join(text) + (video_desc or '')
```

### Conversational Query Interface

```python
# Example API endpoint in app.py
from flask import Flask, request, jsonify
from hotsearch_analysis_agent import AnalysisAgent

app = Flask(__name__)
agent = AnalysisAgent()

@app.route('/api/query', methods=['POST'])
def query_analysis():
    """Handle natural language queries"""
    user_query = request.json.get('query')
    
    # Examples of natural language queries:
    # "最近人工智能领域有什么热点?"
    # "分析今天微博上关于科技的舆情"
    # "情感分析: 华为新产品发布的网络反应"
    
    result = agent.process_query(user_query)
    return jsonify(result)

# Agent implementation
class AnalysisAgent:
    def process_query(self, query):
        # 1. Retrieve relevant hot search data
        data = self.fetch_relevant_data(query)
        
        # 2. LLM-based analysis
        analysis = self.analyze_with_llm(query, data)
        
        # 3. Generate structured response
        return {
            'query': query,
            'topics': self.extract_topics(data),
            'sentiment': analysis['sentiment'],
            'summary': analysis['summary'],
            'clusters': self.cluster_topics(data),
            'sources': [item['url'] for item in data]
        }
```

### LLM Analysis Implementation

```python
import openai
import os
from typing import List, Dict

class LLMAnalyzer:
    def __init__(self):
        openai.api_key = os.getenv('OPENAI_API_KEY')
        openai.api_base = os.getenv('OPENAI_API_BASE')
        self.model = os.getenv('MODEL_NAME', 'gpt-4')
    
    def sentiment_analysis(self, texts: List[str]) -> Dict:
        """Analyze sentiment of multiple texts"""
        prompt = f"""分析以下新闻标题和内容的情感倾向,返回JSON格式:
        {{
            "overall_sentiment": "positive/negative/neutral",
            "sentiment_score": 0.0-1.0,
            "key_emotions": ["emotion1", "emotion2"],
            "reasoning": "分析理由"
        }}
        
        内容:
        {chr(10).join(texts[:10])}
        """
        
        response = openai.ChatCompletion.create(
            model=self.model,
            messages=[
                {"role": "system", "content": "你是专业的舆情分析专家"},
                {"role": "user", "content": prompt}
            ],
            temperature=0.3
        )
        
        return eval(response.choices[0].message.content)
    
    def topic_clustering(self, items: List[Dict]) -> List[Dict]:
        """Cluster related topics"""
        titles = [item['title'] for item in items]
        
        prompt = f"""将以下热搜话题进行聚类,识别关联主题:
        {chr(10).join(f"{i+1}. {t}" for i, t in enumerate(titles))}
        
        返回JSON数组,每个元素包含:
        {{
            "cluster_name": "主题名称",
            "topic_indices": [1, 3, 5],
            "summary": "聚类摘要"
        }}
        """
        
        response = openai.ChatCompletion.create(
            model=self.model,
            messages=[{"role": "user", "content": prompt}],
            temperature=0.5
        )
        
        return eval(response.choices[0].message.content)
    
    def generate_report(self, query: str, data: List[Dict], 
                       sentiment: Dict, clusters: List[Dict]) -> str:
        """Generate comprehensive analysis report"""
        prompt = f"""基于以下数据生成舆情分析报告:
        
        查询: {query}
        情感分析: {sentiment}
        话题聚类: {clusters}
        数据样本: {data[:5]}
        
        报告格式:
        # Report - {query}
        **Time**: {{当前时间}}
        
        > 执行摘要
        
        ### 核心发现与数据亮点
        ### 详细新闻内容梳理
        ### 分析与总结
        ### 信息传播特点
        ### 综上所述
        """
        
        response = openai.ChatCompletion.create(
            model=self.model,
            messages=[
                {"role": "system", "content": "你是资深舆情分析师"},
                {"role": "user", "content": prompt}
            ],
            temperature=0.7,
            max_tokens=4000
        )
        
        return response.choices[0].message.content
```

### Multi-Channel Push Notifications

```python
import requests
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
import os

class PushNotificationService:
    def __init__(self):
        self.smtp_server = os.getenv('SMTP_SERVER')
        self.smtp_port = int(os.getenv('SMTP_PORT', 587))
        self.smtp_user = os.getenv('SMTP_USER')
        self.smtp_password = os.getenv('SMTP_PASSWORD')
        self.wechat_webhook = os.getenv('WECHAT_WEBHOOK')
        self.telegram_token = os.getenv('TELEGRAM_BOT_TOKEN')
        self.telegram_chat_id = os.getenv('TELEGRAM_CHAT_ID')
    
    def send_email(self, subject: str, content: str, recipients: List[str]):
        """Send report via email"""
        msg = MIMEMultipart('alternative')
        msg['Subject'] = subject
        msg['From'] = self.smtp_user
        msg['To'] = ', '.join(recipients)
        
        html_part = MIMEText(content, 'html', 'utf-8')
        msg.attach(html_part)
        
        with smtplib.SMTP(self.smtp_server, self.smtp_port) as server:
            server.starttls()
            server.login(self.smtp_user, self.smtp_password)
            server.send_message(msg)
    
    def send_wechat(self, content: str):
        """Push to Enterprise WeChat group"""
        payload = {
            "msgtype": "markdown",
            "markdown": {
                "content": content
            }
        }
        requests.post(self.wechat_webhook, json=payload)
    
    def send_telegram(self, message: str):
        """Send to Telegram channel"""
        url = f"https://api.telegram.org/bot{self.telegram_token}/sendMessage"
        payload = {
            "chat_id": self.telegram_chat_id,
            "text": message,
            "parse_mode": "Markdown"
        }
        requests.post(url, json=payload)
    
    def broadcast_report(self, report: str, channels: List[str] = None):
        """Broadcast report to configured channels"""
        channels = channels or ['email', 'wechat', 'telegram']
        
        if 'email' in channels:
            recipients = os.getenv('EMAIL_RECIPIENTS', '').split(',')
            self.send_email(
                subject="舆情分析报告",
                content=report,
                recipients=recipients
            )
        
        if 'wechat' in channels:
            self.send_wechat(report)
        
        if 'telegram' in channels:
            self.send_telegram(report)

# Test push notifications
# python test_push_task.py
if __name__ == '__main__':
    service = PushNotificationService()
    test_report = "# 测试报告\n本次监测到3个热点话题..."
    service.broadcast_report(test_report, channels=['telegram'])
```

### Scheduled Push Tasks

```python
from apscheduler.schedulers.background import BackgroundScheduler
from datetime import datetime

class PushTaskScheduler:
    def __init__(self, analyzer: LLMAnalyzer, pusher: PushNotificationService):
        self.analyzer = analyzer
        self.pusher = pusher
        self.scheduler = BackgroundScheduler()
    
    def create_monitoring_task(self, query: str, cron: str, channels: List[str]):
        """Create scheduled monitoring task
        
        Example cron expressions:
        - "0 9 * * *" - Daily at 9 AM
        - "0 9,15 * * *" - Daily at 9 AM and 3 PM
        - "0 */6 * * *" - Every 6 hours
        """
        job_id = f"monitor_{query}_{datetime.now().timestamp()}"
        
        def task_func():
            # Fetch latest data
            data = self.fetch_data_for_query(query)
            
            # Analyze
            sentiment = self.analyzer.sentiment_analysis(
                [item['title'] for item in data]
            )
            clusters = self.analyzer.topic_clustering(data)
            
            # Generate report
            report = self.analyzer.generate_report(
                query, data, sentiment, clusters
            )
            
            # Push to channels
            self.pusher.broadcast_report(report, channels)
        
        # Parse cron and schedule
        cron_parts = cron.split()
        self.scheduler.add_job(
            task_func,
            trigger='cron',
            minute=cron_parts[0],
            hour=cron_parts[1],
            day=cron_parts[2],
            month=cron_parts[3],
            day_of_week=cron_parts[4],
            id=job_id
        )
        
        return job_id
    
    def start(self):
        self.scheduler.start()
    
    def stop(self):
        self.scheduler.shutdown()

# Usage
scheduler = PushTaskScheduler(analyzer, pusher)
scheduler.create_monitoring_task(
    query="人工智能与前沿科技",
    cron="0 9,21 * * *",  # 9 AM and 9 PM daily
    channels=['email', 'wechat']
)
scheduler.start()
```

## Common Workflows

### Complete Analysis Pipeline

```python
from hotsearch_analysis_agent import AnalysisAgent, LLMAnalyzer, PushNotificationService
import pymysql

def analyze_and_push(query: str, days: int = 1):
    """Complete workflow: fetch, analyze, and push"""
    
    # 1. Connect to database
    conn = pymysql.connect(
        host=os.getenv('MYSQL_HOST'),
        user=os.getenv('MYSQL_USER'),
        password=os.getenv('MYSQL_PASSWORD'),
        database=os.getenv('MYSQL_DATABASE'),
        charset='utf8mb4'
    )
    
    # 2. Fetch relevant data
    with conn.cursor(pymysql.cursors.DictCursor) as cursor:
        cursor.execute("""
            SELECT platform, title, url, heat_value, created_at
            FROM hot_search_items
            WHERE created_at >= DATE_SUB(NOW(), INTERVAL %s DAY)
            AND (title LIKE %s OR content LIKE %s)
            ORDER BY heat_value DESC
            LIMIT 100
        """, (days, f'%{query}%', f'%{query}%'))
        data = cursor.fetchall()
    
    # 3. LLM analysis
    analyzer = LLMAnalyzer()
    sentiment = analyzer.sentiment_analysis([item['title'] for item in data])
    clusters = analyzer.topic_clustering(data)
    report = analyzer.generate_report(query, data, sentiment, clusters)
    
    # 4. Push notification
    pusher = PushNotificationService()
    pusher.broadcast_report(report, channels=['email', 'wechat'])
    
    # 5. Save analysis result
    with conn.cursor() as cursor:
        cursor.execute("""
            INSERT INTO analysis_results (query, result, sentiment)
            VALUES (%s, %s, %s)
        """, (query, report, sentiment['overall_sentiment']))
        conn.commit()
    
    conn.close()
    return report

# Execute
report = analyze_and_push("人工智能", days=7)
print(report)
```

### Video Content Extraction

```python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

def extract_video_content(url: str) -> str:
    """Extract text content from video news pages"""
    options = webdriver.ChromeOptions()
    options.add_argument('--headless')
    options.add_argument('--no-sandbox')
    
    driver = webdriver.Chrome(options=options)
    driver.get(url)
    
    try:
        # Wait for video description/transcript
        wait = WebDriverWait(driver, 10)
        
        # Bilibili example
        if 'bilibili.com' in url:
            desc = wait.until(EC.presence_of_element_located(
                (By.CSS_SELECTOR, '.video-desc')
            ))
            comments = driver.find_elements(By.CSS_SELECTOR, '.reply-content')
            content = desc.text + '\n' + '\n'.join([c.text for c in comments[:20]])
        
        # Douyin example
        elif 'douyin.com' in url:
            desc = wait.until(EC.presence_of_element_located(
                (By.CSS_SELECTOR, '#video-desc')
            ))
            content = desc.text
        
        else:
            content = driver.find_element(By.TAG_NAME, 'body').text
        
        return content
    
    finally:
        driver.quit()
```

## Troubleshooting

### Browser Driver Issues

```bash
# Error: chromedriver executable needs to be in PATH
# Solution: Verify driver location and PATH
which chromedriver  # macOS/Linux
where chromedriver  # Windows

# If not found, add to PATH:
export PATH=$PATH:/path/to/driver/directory
# Or set in .env:
CHROME_DRIVER_PATH=/usr/local/bin/chromedriver
```

### Database Connection Failures

```python
# Test connection
import pymysql
try:
    conn = pymysql.connect(
        host=os.getenv('MYSQL_HOST'),
        user=os.getenv('MYSQL_USER'),
        password=os.getenv('MYSQL_PASSWORD'),
        database=os.getenv('MYSQL_DATABASE')
    )
    print("Database connected successfully")
    conn.close()
except Exception as e:
    print(f"Connection failed: {e}")
    # Check MySQL service: systemctl status mysql
    # Verify credentials and database exists
```

### Crawler Rate Limiting

```python
# In hotsearchcrawler/settings.py
# Adjust these settings:
DOWNLOAD_DELAY = 2  # Increase delay between requests
CONCURRENT_REQUESTS_PER_DOMAIN = 4  # Reduce concurrency
AUTOTHROTTLE_ENABLED = True
AUTOTHROTTLE_START_DELAY = 1
AUTOTHROTTLE_MAX_DELAY = 10

# Use rotating proxies if needed:
DOWNLOADER_MIDDLEWARES = {
    'scrapy.downloadermiddlewares.httpproxy.HttpProxyMiddleware': 110,
    'hotsearchcrawler.middlewares.ProxyMiddleware': 100,
}
```

### LLM API Errors

```python
# Handle rate limits and timeouts
import time
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=4, max=10))
def call_llm_with_retry(prompt: str):
    try:
        response = openai.ChatCompletion.create(
            model=os.getenv('MODEL_NAME'),
            messages=[{"role": "user", "content": prompt}],
            timeout=30
        )
        return response
    except openai.error.RateLimitError:
        print("Rate limit hit, retrying...")
        raise
    except openai.error.APIError as e:
        print(f"API error: {e}")
        raise
```

### Memory Issues with Large Reports

```python
# Process data in chunks
def process_large_dataset(query: str, chunk_size: int = 50):
    offset = 0
    all_results = []
    
    while True:
        with conn.cursor(pymysql.cursors.DictCursor) as cursor:
            cursor.execute("""
                SELECT * FROM hot_search_items
                WHERE title LIKE %s
                LIMIT %s OFFSET %s
            """, (f'%{query}%', chunk_size, offset))
            
            chunk = cursor.fetchall()
            if not chunk:
                break
            
            # Process chunk
            chunk_analysis = analyzer.analyze(chunk)
            all_results.append(chunk_analysis)
            
            offset += chunk_size
    
    # Merge results
    return merge_analysis_results(all_results)
```

## Advanced Features

### Custom Platform Crawler

```python
# Add new platform in hotsearchcrawler/spiders/
class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://example.com/trending']
    
    custom_settings = {
        'DOWNLOAD_DELAY': 2,
        'USER_AGENT': 'Mozilla/5.0...'
    }
    
    def parse(self, response):
        for item in response.css('div.trending-item'):
            yield {
                'platform': 'CustomPlatform',
                'title': item.css('h3::text').get(),
                'url': item.css('a::attr(href)').get(),
                'rank': item.css('.rank::text').get(),
                'heat_value': item.css('.heat::text').get()
            }
```

### Real-time WebSocket Updates

```python
from flask_socketio import SocketIO, emit

socketio = SocketIO(app)

@socketio.on('subscribe_topic')
def handle_subscription(data):
    topic = data['topic']
    
    # Stream real-time updates
    def send_updates():
        while True:
            new_items = fetch_new_items(topic)
            if new_items:
                emit('topic_update', {
                    'topic': topic,
                    'items': new_items
                })
            time.sleep(30)
    
    # Start background task
    socketio.start_background_task(send_updates)
```

This skill provides comprehensive guidance for deploying and using the LLM-Based Public Opinion Analytics Assistant for monitoring Chinese social media platforms, performing sentiment analysis, and automating alert notifications.
