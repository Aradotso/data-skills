---
name: llm-public-opinion-analytics-assistant
description: Multi-platform hot topic crawler and LLM-powered public opinion analysis system with sentiment analysis, topic clustering, and multi-channel push notifications
triggers:
  - analyze hot topics from social media platforms
  - scrape trending news from multiple sources
  - set up public opinion monitoring with LLM analysis
  - cluster and analyze sentiment of trending topics
  - configure multi-channel hot topic push notifications
  - build a real-time hot search analytics dashboard
  - deploy distributed web crawlers for trending content
  - create automated opinion analysis reports
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is a comprehensive public opinion analytics platform that combines real-time data crawling from 15 mainstream platforms (26 trending lists) with LLM-powered analysis capabilities. It provides conversational hot search queries, topic clustering, sentiment analysis, and multi-channel push notifications (Email, WeChat, Enterprise WeChat, Telegram).

**Key Features:**
- Multi-platform crawler cluster (Weibo, Bilibili, Douyin, etc.)
- LLM-powered analysis (using Huawei PanGu or OpenAI-compatible models)
- Natural language query interface
- Sentiment analysis and topic clustering
- Multi-channel hot topic push notifications
- Browser-based content extraction (including video content)

**Project Structure:**
- `hotsearch_analysis_agent/` - Analysis system with LLM integration
- `hotsearchcrawler/` - Distributed crawler cluster (fully separated)
- `app.py` - Main application entry point
- `run_spiders` - Crawler launcher (triggered via frontend)
- `test_push_task` - Push notification testing

## Installation

### Prerequisites

1. **Browser Driver Setup:**
```bash
# Download ChromeDriver or EdgeDriver matching your browser version
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Place driver in system PATH, e.g.:
# Windows: C:\Windows\System32\
# Linux/macOS: /usr/local/bin/

# Verify installation
chromedriver --version
```

2. **Database Setup:**
```bash
# Install MySQL
# Create database and tables (refer to init.py)
mysql -u root -p < init.sql
```

3. **Python Environment:**
```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Configuration

**1. Environment Variables (`.env`):**
```bash
# Database Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_db_user
MYSQL_PASSWORD=your_db_password
MYSQL_DATABASE=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Or use Huawei PanGu (local deployment)
PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model
USE_LOCAL_MODEL=true

# Push Notification Channels
# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password

# Enterprise WeChat Bot
WECHAT_WEBHOOK_URL=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxx

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

**2. Crawler Settings (`hotsearchcrawler/settings.py`):**
```python
# MySQL Configuration
MYSQL_CONFIG = {
    'host': os.getenv('MYSQL_HOST', 'localhost'),
    'port': int(os.getenv('MYSQL_PORT', 3306)),
    'user': os.getenv('MYSQL_USER'),
    'password': os.getenv('MYSQL_PASSWORD'),
    'database': os.getenv('MYSQL_DATABASE'),
    'charset': 'utf8mb4'
}

# Optional: Platform-specific cookies
COOKIES = {
    'weibo': 'your_weibo_cookies',  # Optional for better access
    'bilibili': 'your_bilibili_cookies'
}

# Crawler concurrency
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
```

## Core Usage

### Starting the Application

```python
# Launch the main application
python app.py

# The web interface will be available at http://localhost:5000
```

### Starting Crawlers

```python
# Method 1: Via frontend interface (recommended)
# Navigate to web UI and use hotkey control

# Method 2: Manual start
python run_spiders

# Method 3: Test specific spider
cd hotsearchcrawler
python runspider-test --spider=weibo_hot
```

### Crawler Management

```python
# hotsearchcrawler/run_spiders - Main crawler orchestrator
import subprocess
from scrapy.crawler import CrawlerProcess
from scrapy.utils.project import get_project_settings

def start_all_crawlers():
    """Start all configured crawlers"""
    settings = get_project_settings()
    process = CrawlerProcess(settings)
    
    # Add spiders
    spiders = [
        'weibo_hot',
        'bilibili_hot',
        'douyin_hot',
        'zhihu_hot',
        # ... more spiders
    ]
    
    for spider_name in spiders:
        process.crawl(spider_name)
    
    process.start()

# Start crawlers in background
if __name__ == '__main__':
    start_all_crawlers()
```

### LLM-Powered Analysis

```python
# hotsearch_analysis_agent/llm_analyzer.py
import openai
from typing import List, Dict

class OpinionAnalyzer:
    def __init__(self, api_key: str, api_base: str, model: str):
        self.client = openai.OpenAI(
            api_key=api_key,
            base_url=api_base
        )
        self.model = model
    
    def analyze_topics(self, topics: List[Dict]) -> Dict:
        """Analyze trending topics with LLM"""
        prompt = self._build_analysis_prompt(topics)
        
        response = self.client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": "You are a public opinion analyst."},
                {"role": "user", "content": prompt}
            ],
            temperature=0.7
        )
        
        return self._parse_analysis(response.choices[0].message.content)
    
    def _build_analysis_prompt(self, topics: List[Dict]) -> str:
        """Build analysis prompt from topics"""
        topic_text = "\n".join([
            f"{i+1}. {t['title']} (热度: {t['hot_value']}, 来源: {t['platform']})"
            for i, t in enumerate(topics)
        ])
        
        return f"""
        请分析以下热搜话题，提供:
        1. 话题聚类分析
        2. 情感倾向判断
        3. 热点趋势预测
        4. 关键信息提取
        
        热搜数据:
        {topic_text}
        """
    
    def cluster_topics(self, topics: List[Dict]) -> List[List[Dict]]:
        """Cluster related topics using LLM"""
        prompt = f"""
        将以下话题按主题相似度聚类，返回JSON格式:
        {topics}
        """
        
        response = self.client.chat.completions.create(
            model=self.model,
            messages=[{"role": "user", "content": prompt}],
            response_format={"type": "json_object"}
        )
        
        return json.loads(response.choices[0].message.content)

# Usage
analyzer = OpinionAnalyzer(
    api_key=os.getenv('OPENAI_API_KEY'),
    api_base=os.getenv('OPENAI_API_BASE'),
    model=os.getenv('MODEL_NAME')
)

topics = fetch_hot_topics_from_db()
analysis = analyzer.analyze_topics(topics)
clusters = analyzer.cluster_topics(topics)
```

### Sentiment Analysis

```python
# hotsearch_analysis_agent/sentiment.py
from typing import List, Dict

class SentimentAnalyzer:
    def __init__(self, llm_client):
        self.llm = llm_client
    
    def analyze_sentiment(self, content: str) -> Dict:
        """Analyze sentiment of news content"""
        prompt = f"""
        分析以下新闻内容的情感倾向，返回JSON格式:
        {{
            "sentiment": "positive/neutral/negative",
            "score": 0.0-1.0,
            "keywords": ["关键词1", "关键词2"],
            "summary": "简短摘要"
        }}
        
        内容: {content}
        """
        
        response = self.llm.chat.completions.create(
            model=os.getenv('MODEL_NAME'),
            messages=[{"role": "user", "content": prompt}],
            response_format={"type": "json_object"}
        )
        
        return json.loads(response.choices[0].message.content)
    
    def batch_analyze(self, items: List[Dict]) -> List[Dict]:
        """Batch sentiment analysis"""
        results = []
        for item in items:
            sentiment = self.analyze_sentiment(item['content'])
            results.append({
                **item,
                'sentiment': sentiment
            })
        return results

# Usage
sentiment_analyzer = SentimentAnalyzer(analyzer.client)
news_items = fetch_news_from_db()
analyzed = sentiment_analyzer.batch_analyze(news_items)
```

### Multi-Channel Push Notifications

```python
# test_push_task - Push notification system
import smtplib
import requests
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart

class PushNotifier:
    def __init__(self):
        self.smtp_config = {
            'host': os.getenv('SMTP_HOST'),
            'port': int(os.getenv('SMTP_PORT')),
            'user': os.getenv('SMTP_USER'),
            'password': os.getenv('SMTP_PASSWORD')
        }
        self.wechat_webhook = os.getenv('WECHAT_WEBHOOK_URL')
        self.telegram_token = os.getenv('TELEGRAM_BOT_TOKEN')
        self.telegram_chat_id = os.getenv('TELEGRAM_CHAT_ID')
    
    def push_email(self, subject: str, content: str, recipients: List[str]):
        """Send email notification"""
        msg = MIMEMultipart()
        msg['From'] = self.smtp_config['user']
        msg['To'] = ', '.join(recipients)
        msg['Subject'] = subject
        
        msg.attach(MIMEText(content, 'html', 'utf-8'))
        
        with smtplib.SMTP(self.smtp_config['host'], self.smtp_config['port']) as server:
            server.starttls()
            server.login(self.smtp_config['user'], self.smtp_config['password'])
            server.send_message(msg)
    
    def push_wechat(self, content: str):
        """Push to Enterprise WeChat"""
        data = {
            "msgtype": "markdown",
            "markdown": {
                "content": content
            }
        }
        response = requests.post(self.wechat_webhook, json=data)
        return response.json()
    
    def push_telegram(self, message: str):
        """Push to Telegram"""
        url = f"https://api.telegram.org/bot{self.telegram_token}/sendMessage"
        data = {
            "chat_id": self.telegram_chat_id,
            "text": message,
            "parse_mode": "Markdown"
        }
        response = requests.post(url, json=data)
        return response.json()
    
    def push_all_channels(self, report: Dict):
        """Push to all configured channels"""
        formatted_content = self._format_report(report)
        
        # Email
        self.push_email(
            subject=f"热点分析报告 - {report['time']}",
            content=formatted_content['html'],
            recipients=['recipient@example.com']
        )
        
        # WeChat
        self.push_wechat(formatted_content['markdown'])
        
        # Telegram
        self.push_telegram(formatted_content['markdown'])
    
    def _format_report(self, report: Dict) -> Dict:
        """Format report for different channels"""
        return {
            'html': self._to_html(report),
            'markdown': self._to_markdown(report)
        }

# Usage
notifier = PushNotifier()

# Generate analysis report
report = {
    'time': '2026-04-07 12:32:00',
    'topic': '人工智能与前沿科技',
    'findings': analysis_results,
    'news_items': top_news
}

# Push to all channels
notifier.push_all_channels(report)
```

### Database Operations

```python
# hotsearch_analysis_agent/database.py
import pymysql
from typing import List, Dict

class HotSearchDB:
    def __init__(self):
        self.config = {
            'host': os.getenv('MYSQL_HOST'),
            'port': int(os.getenv('MYSQL_PORT')),
            'user': os.getenv('MYSQL_USER'),
            'password': os.getenv('MYSQL_PASSWORD'),
            'database': os.getenv('MYSQL_DATABASE'),
            'charset': 'utf8mb4'
        }
    
    def fetch_hot_topics(self, limit: int = 50) -> List[Dict]:
        """Fetch recent hot topics"""
        conn = pymysql.connect(**self.config)
        cursor = conn.cursor(pymysql.cursors.DictCursor)
        
        query = """
        SELECT id, title, hot_value, platform, url, created_at
        FROM hot_topics
        ORDER BY created_at DESC, hot_value DESC
        LIMIT %s
        """
        
        cursor.execute(query, (limit,))
        results = cursor.fetchall()
        
        cursor.close()
        conn.close()
        
        return results
    
    def search_topics(self, keyword: str, platforms: List[str] = None) -> List[Dict]:
        """Search topics by keyword"""
        conn = pymysql.connect(**self.config)
        cursor = conn.cursor(pymysql.cursors.DictCursor)
        
        query = """
        SELECT id, title, hot_value, platform, url, content, created_at
        FROM hot_topics
        WHERE title LIKE %s OR content LIKE %s
        """
        params = [f'%{keyword}%', f'%{keyword}%']
        
        if platforms:
            query += " AND platform IN (" + ",".join(['%s'] * len(platforms)) + ")"
            params.extend(platforms)
        
        query += " ORDER BY hot_value DESC LIMIT 100"
        
        cursor.execute(query, params)
        results = cursor.fetchall()
        
        cursor.close()
        conn.close()
        
        return results
    
    def save_analysis(self, topic_ids: List[int], analysis: Dict):
        """Save analysis results"""
        conn = pymysql.connect(**self.config)
        cursor = conn.cursor()
        
        query = """
        INSERT INTO topic_analysis (topic_ids, sentiment, clusters, summary, created_at)
        VALUES (%s, %s, %s, %s, NOW())
        """
        
        cursor.execute(query, (
            json.dumps(topic_ids),
            analysis['sentiment'],
            json.dumps(analysis['clusters']),
            analysis['summary']
        ))
        
        conn.commit()
        cursor.close()
        conn.close()

# Usage
db = HotSearchDB()

# Fetch hot topics
topics = db.fetch_hot_topics(limit=50)

# Search by keyword
ai_topics = db.search_topics('人工智能', platforms=['weibo', 'zhihu'])

# Save analysis
db.save_analysis(
    topic_ids=[t['id'] for t in topics],
    analysis=analysis_results
)
```

## Common Patterns

### Scheduled Analysis Task

```python
# Create scheduled analysis job
import schedule
import time

def analyze_and_push():
    """Scheduled analysis and push"""
    db = HotSearchDB()
    analyzer = OpinionAnalyzer(
        api_key=os.getenv('OPENAI_API_KEY'),
        api_base=os.getenv('OPENAI_API_BASE'),
        model=os.getenv('MODEL_NAME')
    )
    notifier = PushNotifier()
    
    # Fetch topics
    topics = db.fetch_hot_topics(limit=100)
    
    # Analyze
    analysis = analyzer.analyze_topics(topics)
    clusters = analyzer.cluster_topics(topics)
    
    # Generate report
    report = {
        'time': datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
        'topic': '每日热点分析',
        'analysis': analysis,
        'clusters': clusters,
        'topics': topics[:20]
    }
    
    # Save and push
    db.save_analysis([t['id'] for t in topics], analysis)
    notifier.push_all_channels(report)

# Schedule daily at 9:00 AM and 6:00 PM
schedule.every().day.at("09:00").do(analyze_and_push)
schedule.every().day.at("18:00").do(analyze_and_push)

while True:
    schedule.run_pending()
    time.sleep(60)
```

### Custom Spider Creation

```python
# hotsearchcrawler/spiders/custom_spider.py
import scrapy
from hotsearchcrawler.items import HotTopicItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    allowed_domains = ['example.com']
    start_urls = ['https://example.com/hot']
    
    def parse(self, response):
        """Parse hot topic list"""
        for item in response.css('.hot-item'):
            hot_topic = HotTopicItem()
            hot_topic['title'] = item.css('.title::text').get()
            hot_topic['hot_value'] = item.css('.hot-value::text').get()
            hot_topic['url'] = item.css('a::attr(href)').get()
            hot_topic['platform'] = self.name
            
            # Request detail page
            yield scrapy.Request(
                url=hot_topic['url'],
                callback=self.parse_detail,
                meta={'item': hot_topic}
            )
    
    def parse_detail(self, response):
        """Parse detail page content"""
        item = response.meta['item']
        item['content'] = response.css('.content::text').getall()
        item['author'] = response.css('.author::text').get()
        item['publish_time'] = response.css('.time::text').get()
        
        yield item
```

## Troubleshooting

**Chrome Driver Issues:**
```bash
# Error: chromedriver not found
# Solution: Ensure driver is in PATH and executable
chmod +x /path/to/chromedriver
export PATH=$PATH:/path/to/chromedriver

# Version mismatch
# Solution: Download exact version matching your browser
chromedriver --version
google-chrome --version  # Should match
```

**Database Connection Failed:**
```python
# Check connection
import pymysql
try:
    conn = pymysql.connect(
        host=os.getenv('MYSQL_HOST'),
        user=os.getenv('MYSQL_USER'),
        password=os.getenv('MYSQL_PASSWORD')
    )
    print("Connection successful")
except Exception as e:
    print(f"Connection failed: {e}")
```

**LLM API Rate Limits:**
```python
# Add retry logic with backoff
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=4, max=10))
def call_llm_with_retry(prompt):
    return analyzer.client.chat.completions.create(
        model=os.getenv('MODEL_NAME'),
        messages=[{"role": "user", "content": prompt}]
    )
```

**Crawler Blocked:**
```python
# Add random user agents and delays
# In hotsearchcrawler/settings.py
USER_AGENT_LIST = [
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
    'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36'
]

DOWNLOAD_DELAY = 2  # Increase delay
RANDOMIZE_DOWNLOAD_DELAY = True
```

**Push Notification Failures:**
```python
# Test each channel separately
notifier = PushNotifier()

# Test email
notifier.push_email("Test", "Test content", ["test@example.com"])

# Test WeChat
notifier.push_wechat("Test message")

# Test Telegram
notifier.push_telegram("Test message")
```

This skill provides comprehensive coverage for deploying and using the LLM-Based Intelligent Public Opinion Analytics Assistant for real-time trending topic analysis and monitoring.
