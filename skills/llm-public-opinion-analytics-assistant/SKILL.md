---
name: llm-public-opinion-analytics-assistant
description: Multi-platform public opinion analytics assistant with web scraping, LLM analysis, and multi-channel alert system
triggers:
  - how do I set up the public opinion analytics assistant
  - crawl hot search data from multiple platforms
  - analyze sentiment and cluster topics with LLM
  - configure push notifications for trending topics
  - how to use the opinion analytics dashboard
  - set up multi-platform hot search monitoring
  - create sentiment analysis reports with this tool
  - configure telegram and wechat alerts for trends
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This is a comprehensive public opinion analytics system that combines real-time data crawling from 15 major platforms (26 different trending lists) with LLM-powered analysis capabilities. The system provides:

- **Data Collection**: Scrapy-based crawler cluster for multi-platform hot search data
- **LLM Analysis**: Topic clustering, sentiment analysis, and trend detection using large language models
- **Interactive Interface**: Conversational query interface for exploring trending topics
- **Multi-Channel Alerts**: Push notifications via email, WeChat, Enterprise WeChat, and Telegram
- **Video Content Analysis**: Extracts insights even from video-based news content

The project consists of two main components:
1. **Analysis System** (`hotsearch_analysis_agent`) - Flask web application with LLM integration
2. **Crawler Cluster** (`hotsearchcrawler`) - Distributed Scrapy spiders for data collection

## Installation

### Prerequisites

**1. Browser Driver Setup**

The system requires a WebDriver for content extraction. Follow these steps:

```bash
# Check your Chrome/Edge version first
# Chrome: Settings → About Chrome
# Edge: Settings → About Microsoft Edge

# Download matching driver:
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# On macOS/Linux, move driver to PATH:
sudo mv chromedriver /usr/local/bin/
sudo chmod +x /usr/local/bin/chromedriver

# On Windows, add driver location to PATH environment variable

# Verify installation:
chromedriver --version
```

**2. Database Setup**

```bash
# Install MySQL
# Create database and tables using init.py as reference

mysql -u root -p
CREATE DATABASE opinion_analytics CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

**3. Python Environment**

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Configuration

**1. Crawler Settings** (`hotsearchcrawler/settings.py`)

```python
# MySQL configuration
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_user'
MYSQL_PASSWORD = 'your_password'
MYSQL_DATABASE = 'opinion_analytics'

# Optional: Platform cookies for authenticated access
# Add cookies in settings.py for platforms that require login
```

**2. Analysis System** (`.env` file in project root)

```env
# Database
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=opinion_analytics

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Or use local Huawei Pangu model
# PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model

# Push Notification Settings
# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
ALERT_EMAIL_RECIPIENTS=recipient1@example.com,recipient2@example.com

# Telegram
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# WeChat Work (Enterprise WeChat)
WECHAT_WORK_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=your_key
WECHAT_WORK_AGENT_ID=your_agent_id
WECHAT_WORK_CORP_ID=your_corp_id
WECHAT_WORK_SECRET=your_secret
```

## Running the System

### Start the Analysis Web Application

```bash
# Start the Flask app
python app.py

# Access the dashboard at http://localhost:5000
```

### Run the Crawler Cluster

**Option 1: Via Web Interface**

Use the dashboard's built-in controls (keyboard shortcuts available) to start/stop crawlers.

**Option 2: Command Line**

```bash
# Run all spiders
python run_spiders.py

# Test individual spider
cd hotsearchcrawler
scrapy crawl weibo_hot  # Example: Weibo hot search spider
scrapy crawl bilibili_hot  # Example: Bilibili trending
scrapy crawl zhihu_hot  # Example: Zhihu hot questions
```

### Test Push Notifications

```bash
# Test notification channels
python test_push_task.py
```

## Core Usage Patterns

### 1. Conversational Trending Topic Query

```python
# Example: Natural language query via the web interface
# User asks: "What's trending about AI and technology?"

# The system:
# 1. Searches across all 26 trending lists
# 2. Filters relevant topics using LLM
# 3. Clusters related discussions
# 4. Analyzes sentiment
# 5. Returns formatted response with source links
```

### 2. Custom Crawler Development

```python
# hotsearchcrawler/spiders/custom_platform.py
import scrapy
from hotsearchcrawler.items import HotSearchItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    allowed_domains = ['example.com']
    start_urls = ['https://example.com/trending']

    def parse(self, response):
        # Extract trending items
        for item in response.css('.trending-item'):
            hot_search = HotSearchItem()
            hot_search['title'] = item.css('.title::text').get()
            hot_search['url'] = item.css('a::attr(href)').get()
            hot_search['rank'] = item.css('.rank::text').get()
            hot_search['heat_score'] = item.css('.heat::text').get()
            hot_search['platform'] = 'custom_platform'
            hot_search['category'] = 'general'
            
            yield hot_search
            
            # Follow link to get detail content
            if hot_search['url']:
                yield scrapy.Request(
                    hot_search['url'],
                    callback=self.parse_detail,
                    meta={'item': hot_search}
                )
    
    def parse_detail(self, response):
        item = response.meta['item']
        item['content'] = response.css('.article-content::text').getall()
        item['publish_time'] = response.css('.publish-time::text').get()
        yield item
```

### 3. LLM-Powered Analysis

```python
# hotsearch_analysis_agent/analyzers/sentiment_analyzer.py
from openai import OpenAI
import os

class SentimentAnalyzer:
    def __init__(self):
        self.client = OpenAI(
            api_key=os.getenv('OPENAI_API_KEY'),
            base_url=os.getenv('OPENAI_API_BASE')
        )
    
    def analyze_sentiment(self, topics_data):
        """Analyze sentiment of trending topics"""
        prompt = f"""
        Analyze the sentiment of these trending topics:
        
        {topics_data}
        
        Provide:
        1. Overall sentiment (positive/negative/neutral)
        2. Key emotional drivers
        3. Potential risks or opportunities
        """
        
        response = self.client.chat.completions.create(
            model=os.getenv('MODEL_NAME', 'gpt-4'),
            messages=[
                {"role": "system", "content": "You are a public opinion analyst."},
                {"role": "user", "content": prompt}
            ],
            temperature=0.3
        )
        
        return response.choices[0].message.content
    
    def cluster_topics(self, topics_list):
        """Cluster related topics together"""
        prompt = f"""
        Cluster these topics by theme:
        
        {topics_list}
        
        Return JSON format:
        {{
            "cluster_name": ["topic1", "topic2"],
            ...
        }}
        """
        
        response = self.client.chat.completions.create(
            model=os.getenv('MODEL_NAME', 'gpt-4'),
            messages=[{"role": "user", "content": prompt}],
            temperature=0.3,
            response_format={"type": "json_object"}
        )
        
        return response.choices[0].message.content
```

### 4. Setting Up Push Alerts

```python
# hotsearch_analysis_agent/push/notification_manager.py
import requests
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
import os

class NotificationManager:
    
    def send_telegram(self, message):
        """Send alert via Telegram"""
        bot_token = os.getenv('TELEGRAM_BOT_TOKEN')
        chat_id = os.getenv('TELEGRAM_CHAT_ID')
        
        url = f"https://api.telegram.org/bot{bot_token}/sendMessage"
        payload = {
            'chat_id': chat_id,
            'text': message,
            'parse_mode': 'Markdown'
        }
        
        response = requests.post(url, json=payload)
        return response.json()
    
    def send_email(self, subject, body):
        """Send alert via SMTP email"""
        msg = MIMEMultipart()
        msg['From'] = os.getenv('SMTP_USER')
        msg['To'] = os.getenv('ALERT_EMAIL_RECIPIENTS')
        msg['Subject'] = subject
        
        msg.attach(MIMEText(body, 'html'))
        
        with smtplib.SMTP(os.getenv('SMTP_HOST'), int(os.getenv('SMTP_PORT'))) as server:
            server.starttls()
            server.login(os.getenv('SMTP_USER'), os.getenv('SMTP_PASSWORD'))
            server.send_message(msg)
    
    def send_wechat_work(self, message):
        """Send alert to WeChat Work group"""
        webhook = os.getenv('WECHAT_WORK_WEBHOOK')
        
        payload = {
            'msgtype': 'markdown',
            'markdown': {
                'content': message
            }
        }
        
        response = requests.post(webhook, json=payload)
        return response.json()
    
    def create_alert_task(self, keywords, channels=['email', 'telegram']):
        """Create monitoring task for specific keywords"""
        return {
            'keywords': keywords,
            'channels': channels,
            'frequency': 'hourly',  # or 'daily', 'realtime'
            'threshold': 100  # heat score threshold
        }
```

### 5. Querying the Database

```python
# hotsearch_analysis_agent/db/query_helper.py
import pymysql
import os

class HotSearchDB:
    def __init__(self):
        self.conn = pymysql.connect(
            host=os.getenv('MYSQL_HOST'),
            port=int(os.getenv('MYSQL_PORT')),
            user=os.getenv('MYSQL_USER'),
            password=os.getenv('MYSQL_PASSWORD'),
            database=os.getenv('MYSQL_DATABASE'),
            charset='utf8mb4'
        )
    
    def get_trending_by_platform(self, platform, limit=50):
        """Get latest trending topics from specific platform"""
        with self.conn.cursor(pymysql.cursors.DictCursor) as cursor:
            sql = """
                SELECT title, url, rank, heat_score, category, created_at
                FROM hot_search_items
                WHERE platform = %s
                ORDER BY created_at DESC, rank ASC
                LIMIT %s
            """
            cursor.execute(sql, (platform, limit))
            return cursor.fetchall()
    
    def search_topics(self, keyword):
        """Full-text search for topics"""
        with self.conn.cursor(pymysql.cursors.DictCursor) as cursor:
            sql = """
                SELECT * FROM hot_search_items
                WHERE title LIKE %s OR content LIKE %s
                ORDER BY created_at DESC
                LIMIT 100
            """
            pattern = f'%{keyword}%'
            cursor.execute(sql, (pattern, pattern))
            return cursor.fetchall()
    
    def get_trending_stats(self, hours=24):
        """Get trending statistics for the past N hours"""
        with self.conn.cursor(pymysql.cursors.DictCursor) as cursor:
            sql = """
                SELECT platform, COUNT(*) as count, AVG(heat_score) as avg_heat
                FROM hot_search_items
                WHERE created_at >= DATE_SUB(NOW(), INTERVAL %s HOUR)
                GROUP BY platform
            """
            cursor.execute(sql, (hours,))
            return cursor.fetchall()
```

## Common Workflows

### Workflow 1: Daily Trend Report Generation

```python
# scripts/generate_daily_report.py
from hotsearch_analysis_agent.db.query_helper import HotSearchDB
from hotsearch_analysis_agent.analyzers.sentiment_analyzer import SentimentAnalyzer
from hotsearch_analysis_agent.push.notification_manager import NotificationManager

def generate_daily_report():
    db = HotSearchDB()
    analyzer = SentimentAnalyzer()
    notifier = NotificationManager()
    
    # Get top topics from past 24 hours
    topics = db.get_trending_stats(hours=24)
    
    # Analyze sentiment and cluster
    sentiment = analyzer.analyze_sentiment(topics)
    clusters = analyzer.cluster_topics(topics)
    
    # Format report
    report = f"""
    # Daily Trend Report - {datetime.now().strftime('%Y-%m-%d')}
    
    ## Top Platforms
    {format_platform_stats(topics)}
    
    ## Sentiment Analysis
    {sentiment}
    
    ## Topic Clusters
    {clusters}
    """
    
    # Push to all channels
    notifier.send_email('Daily Trend Report', report)
    notifier.send_telegram(report)
    notifier.send_wechat_work(report)

if __name__ == '__main__':
    generate_daily_report()
```

### Workflow 2: Real-time Alert on Keywords

```python
# scripts/keyword_monitor.py
import time
from hotsearch_analysis_agent.db.query_helper import HotSearchDB
from hotsearch_analysis_agent.push.notification_manager import NotificationManager

def monitor_keywords(keywords, check_interval=300):
    """Monitor for specific keywords and send alerts"""
    db = HotSearchDB()
    notifier = NotificationManager()
    seen_items = set()
    
    while True:
        for keyword in keywords:
            results = db.search_topics(keyword)
            
            for item in results:
                item_id = f"{item['platform']}_{item['title']}"
                
                if item_id not in seen_items:
                    seen_items.add(item_id)
                    
                    # Check if heat score exceeds threshold
                    if item.get('heat_score', 0) > 1000:
                        alert_message = f"""
                        🔥 High Heat Alert!
                        
                        Keyword: {keyword}
                        Platform: {item['platform']}
                        Title: {item['title']}
                        Heat Score: {item['heat_score']}
                        URL: {item['url']}
                        """
                        
                        notifier.send_telegram(alert_message)
        
        time.sleep(check_interval)

if __name__ == '__main__':
    monitor_keywords(['AI', '人工智能', '科技'], check_interval=300)
```

## Troubleshooting

### WebDriver Issues

```bash
# Error: selenium.common.exceptions.WebDriverException
# Solution: Ensure driver version matches browser version

# Check Chrome version
google-chrome --version  # Linux
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome --version  # macOS

# Download matching driver and verify:
chromedriver --version

# If PATH issues persist, specify driver path explicitly in code:
from selenium import webdriver
driver = webdriver.Chrome(executable_path='/usr/local/bin/chromedriver')
```

### MySQL Connection Errors

```bash
# Error: pymysql.err.OperationalError: (2003, "Can't connect to MySQL server")
# Solution: Verify MySQL is running and credentials are correct

# Check MySQL status
sudo systemctl status mysql  # Linux
brew services list  # macOS

# Test connection manually
mysql -h localhost -u your_user -p

# Verify .env file has correct credentials
cat .env | grep MYSQL
```

### Crawler Rate Limiting

```python
# If getting 429 errors or blocks, adjust settings in hotsearchcrawler/settings.py

# Reduce concurrent requests
CONCURRENT_REQUESTS = 8  # Lower this value

# Add delays between requests
DOWNLOAD_DELAY = 3  # seconds
RANDOMIZE_DOWNLOAD_DELAY = True

# Use rotating user agents
DOWNLOADER_MIDDLEWARES = {
    'scrapy.downloadermiddlewares.useragent.UserAgentMiddleware': None,
    'scrapy_user_agents.middlewares.RandomUserAgentMiddleware': 400,
}

# Enable AutoThrottle
AUTOTHROTTLE_ENABLED = True
AUTOTHROTTLE_START_DELAY = 5
AUTOTHROTTLE_MAX_DELAY = 60
```

### LLM API Timeout

```python
# Increase timeout in OpenAI client
from openai import OpenAI

client = OpenAI(
    api_key=os.getenv('OPENAI_API_KEY'),
    timeout=60.0,  # Increase from default
    max_retries=3
)

# For long analyses, use streaming
response = client.chat.completions.create(
    model='gpt-4',
    messages=[...],
    stream=True
)

for chunk in response:
    print(chunk.choices[0].delta.content, end='')
```

### Database Performance

```sql
-- Add indexes for faster queries
CREATE INDEX idx_platform_time ON hot_search_items(platform, created_at);
CREATE INDEX idx_heat_score ON hot_search_items(heat_score);
CREATE FULLTEXT INDEX idx_content ON hot_search_items(title, content);

-- Clean old data periodically
DELETE FROM hot_search_items WHERE created_at < DATE_SUB(NOW(), INTERVAL 30 DAY);
```

## Advanced Configuration

### Using Huawei Pangu Model Locally

```python
# hotsearch_analysis_agent/llm/pangu_client.py
from transformers import AutoTokenizer, AutoModelForCausalLM
import os

class PanguAnalyzer:
    def __init__(self):
        model_path = os.getenv('PANGU_MODEL_PATH')
        self.tokenizer = AutoTokenizer.from_pretrained(model_path)
        self.model = AutoModelForCausalLM.from_pretrained(
            model_path,
            device_map='auto',
            torch_dtype='auto'
        )
    
    def analyze(self, prompt):
        inputs = self.tokenizer(prompt, return_tensors='pt')
        outputs = self.model.generate(
            **inputs,
            max_length=2048,
            temperature=0.3,
            top_p=0.9
        )
        return self.tokenizer.decode(outputs[0], skip_special_tokens=True)
```

### Distributed Crawler Deployment

```bash
# Deploy Scrapy with Scrapyd for distributed crawling

# Install scrapyd
pip install scrapyd scrapyd-client

# Start scrapyd server
scrapyd

# Deploy project
scrapyd-deploy

# Schedule spiders remotely
curl http://localhost:6800/schedule.json -d project=hotsearchcrawler -d spider=weibo_hot
```

This skill covers the essential operations for deploying and using the LLM-based public opinion analytics system, from initial setup through advanced monitoring workflows.
