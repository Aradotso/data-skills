---
name: llm-public-opinion-analytics-assistant
description: Multi-platform hot search crawler and LLM-powered sentiment analysis system with automated push notifications
triggers:
  - set up public opinion monitoring system
  - crawl hot search data from multiple platforms
  - analyze sentiment of trending topics
  - configure automated sentiment reports
  - integrate LLM for opinion analytics
  - push hot topic notifications to WeChat
  - cluster and analyze trending news
  - deploy opinion monitoring assistant
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is a comprehensive public opinion analytics system that combines:
- **Real-time data crawler** for 26 trending lists across 15 major platforms (Weibo, Bilibili, Douyin, etc.)
- **LLM-powered analysis** for topic clustering, sentiment analysis, and trend detection
- **Multi-channel push notifications** (Email, WeChat, Enterprise WeChat, Telegram)
- **Interactive chat interface** for conversational queries about hot topics
- **Video content extraction** capability for comprehensive news analysis

The system separates concerns between crawler cluster (`hotsearchcrawler`) and analysis engine (`hotsearch_analysis_agent`), supporting local LLM deployment for data privacy.

## Installation

### Prerequisites

```bash
# Python 3.8+
python --version

# MySQL 5.7+ or 8.0+
mysql --version

# Chrome/Edge browser driver
chromedriver --version  # or msedgedriver --version
```

### Browser Driver Setup

1. **Check browser version**: Open Chrome/Edge → Settings → About
2. **Download matching driver**:
   - Chrome: https://chromedriver.chromium.org/
   - Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/
3. **Add to PATH**:
   ```bash
   # Linux/macOS
   export PATH=$PATH:/path/to/driver/directory
   
   # Windows: Add driver directory to System Environment Variables > Path
   ```
4. **Verify**:
   ```bash
   chromedriver --version
   ```

### Project Setup

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
# Reference init.py for schema creation
import pymysql

connection = pymysql.connect(
    host='localhost',
    user='root',
    password='your_password',
    charset='utf8mb4'
)

with connection.cursor() as cursor:
    # Create database
    cursor.execute("CREATE DATABASE IF NOT EXISTS hotsearch_db CHARACTER SET utf8mb4")
    cursor.execute("USE hotsearch_db")
    
    # Create hot search table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS hot_search (
            id INT AUTO_INCREMENT PRIMARY KEY,
            platform VARCHAR(50) NOT NULL,
            title TEXT NOT NULL,
            url TEXT,
            rank INT,
            heat_value VARCHAR(50),
            crawl_time DATETIME,
            content LONGTEXT,
            sentiment VARCHAR(20),
            INDEX idx_platform (platform),
            INDEX idx_crawl_time (crawl_time)
        ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
    """)
    
    # Create push task table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS push_tasks (
            id INT AUTO_INCREMENT PRIMARY KEY,
            task_name VARCHAR(100) NOT NULL,
            query_keywords TEXT NOT NULL,
            push_channels JSON,
            schedule_cron VARCHAR(50),
            is_active BOOLEAN DEFAULT TRUE,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
    """)

connection.commit()
connection.close()
```

### Configuration

Create `.env` file in project root:

```env
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key_or_pangu_key
OPENAI_API_BASE=https://api.openai.com/v1
OPENAI_MODEL=gpt-4

# Or use Huawei Pangu model locally
# PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model

# Push Notification Channels
# WeChat Work Bot
WECHAT_WORK_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email (SMTP)
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USERNAME=your_email@gmail.com
SMTP_PASSWORD=your_app_password
SMTP_FROM=your_email@gmail.com
SMTP_TO=recipient@example.com
```

Configure crawler settings in `hotsearchcrawler/settings.py`:

```python
# MySQL settings
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'root'
MYSQL_PASSWORD = 'your_password'
MYSQL_DATABASE = 'hotsearch_db'

# Optional: Platform-specific cookies for enhanced access
COOKIES = {
    'weibo': 'your_weibo_cookie',
    'bilibili': 'your_bilibili_cookie'
}
```

## Core Components

### 1. Web Application (app.py)

Start the main application server:

```python
# app.py
from flask import Flask, render_template, jsonify, request
from hotsearch_analysis_agent import AnalysisAgent
import threading

app = Flask(__name__)
agent = AnalysisAgent()

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/api/query', methods=['POST'])
def query_hotscarch():
    data = request.json
    query = data.get('query', '')
    result = agent.analyze_query(query)
    return jsonify(result)

@app.route('/api/start_crawler', methods=['POST'])
def start_crawler():
    from run_spiders import start_all_spiders
    thread = threading.Thread(target=start_all_spiders)
    thread.start()
    return jsonify({'status': 'started'})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
```

Run the application:

```bash
python app.py
# Access at http://localhost:5000
```

### 2. Crawler System

#### Run All Crawlers

```python
# run_spiders.py
from scrapy.crawler import CrawlerProcess
from scrapy.utils.project import get_project_settings
from hotsearchcrawler.spiders import *

def start_all_spiders():
    settings = get_project_settings()
    process = CrawlerProcess(settings)
    
    # Add all spider classes
    spiders = [
        'weibo_spider',
        'douyin_spider',
        'bilibili_spider',
        'zhihu_spider',
        # ... add other 22 platform spiders
    ]
    
    for spider_name in spiders:
        process.crawl(spider_name)
    
    process.start()

if __name__ == '__main__':
    start_all_spiders()
```

#### Test Individual Spider

```python
# runspider-test.py
from scrapy.crawler import CrawlerRunner
from twisted.internet import reactor
from hotsearchcrawler.spiders.weibo_spider import WeiboSpider

runner = CrawlerRunner()

d = runner.crawl(WeiboSpider)
d.addBoth(lambda _: reactor.stop())
reactor.run()
```

### 3. Analysis Agent

```python
# hotsearch_analysis_agent/agent.py
from openai import OpenAI
import pymysql
import os

class AnalysisAgent:
    def __init__(self):
        self.client = OpenAI(
            api_key=os.getenv('OPENAI_API_KEY'),
            base_url=os.getenv('OPENAI_API_BASE')
        )
        self.db = self._init_db()
    
    def _init_db(self):
        return pymysql.connect(
            host=os.getenv('MYSQL_HOST'),
            user=os.getenv('MYSQL_USER'),
            password=os.getenv('MYSQL_PASSWORD'),
            database=os.getenv('MYSQL_DATABASE'),
            charset='utf8mb4'
        )
    
    def query_hotsearch(self, keywords, platforms=None, limit=50):
        """Query hot search data from database"""
        with self.db.cursor(pymysql.cursors.DictCursor) as cursor:
            sql = """
                SELECT * FROM hot_search 
                WHERE title LIKE %s 
                AND crawl_time > DATE_SUB(NOW(), INTERVAL 24 HOUR)
            """
            params = [f'%{keywords}%']
            
            if platforms:
                placeholders = ','.join(['%s'] * len(platforms))
                sql += f" AND platform IN ({placeholders})"
                params.extend(platforms)
            
            sql += " ORDER BY crawl_time DESC LIMIT %s"
            params.append(limit)
            
            cursor.execute(sql, params)
            return cursor.fetchall()
    
    def sentiment_analysis(self, text):
        """Analyze sentiment using LLM"""
        response = self.client.chat.completions.create(
            model=os.getenv('OPENAI_MODEL', 'gpt-4'),
            messages=[
                {"role": "system", "content": "你是舆情分析专家。分析文本的情感倾向,返回:正面/负面/中性"},
                {"role": "user", "content": f"分析这条新闻的情感倾向:\n{text}"}
            ],
            temperature=0.3
        )
        return response.choices[0].message.content.strip()
    
    def cluster_topics(self, news_list):
        """Cluster related topics"""
        titles = [news['title'] for news in news_list]
        prompt = f"""
        对以下热搜话题进行聚类分析,识别主要主题:
        {chr(10).join(f'{i+1}. {t}' for i, t in enumerate(titles))}
        
        返回JSON格式: {{"clusters": [{{"theme": "主题名", "items": [索引列表]}}]}}
        """
        
        response = self.client.chat.completions.create(
            model=os.getenv('OPENAI_MODEL', 'gpt-4'),
            messages=[
                {"role": "system", "content": "你是话题聚类分析专家"},
                {"role": "user", "content": prompt}
            ],
            temperature=0.5
        )
        
        import json
        return json.loads(response.choices[0].message.content)
    
    def analyze_query(self, query):
        """Main analysis entry point"""
        # Extract keywords from natural language query
        keywords = self._extract_keywords(query)
        
        # Query database
        news = self.query_hotsearch(keywords)
        
        # Perform clustering
        clusters = self.cluster_topics(news)
        
        # Sentiment analysis on representative items
        for cluster in clusters['clusters']:
            sample_idx = cluster['items'][0]
            sentiment = self.sentiment_analysis(news[sample_idx]['title'])
            cluster['sentiment'] = sentiment
        
        return {
            'query': query,
            'total_results': len(news),
            'clusters': clusters,
            'raw_data': news
        }
    
    def _extract_keywords(self, query):
        """Use LLM to extract search keywords"""
        response = self.client.chat.completions.create(
            model=os.getenv('OPENAI_MODEL', 'gpt-4'),
            messages=[
                {"role": "system", "content": "从用户查询中提取关键词,只返回关键词,不要其他内容"},
                {"role": "user", "content": query}
            ],
            temperature=0.1
        )
        return response.choices[0].message.content.strip()
```

### 4. Push Notification System

```python
# test_push_task.py
import requests
import os
from datetime import datetime

class PushNotifier:
    @staticmethod
    def push_to_wechat_work(content):
        """Push to WeChat Work group bot"""
        webhook = os.getenv('WECHAT_WORK_WEBHOOK')
        data = {
            "msgtype": "markdown",
            "markdown": {
                "content": content
            }
        }
        response = requests.post(webhook, json=data)
        return response.json()
    
    @staticmethod
    def push_to_telegram(content):
        """Push to Telegram bot"""
        token = os.getenv('TELEGRAM_BOT_TOKEN')
        chat_id = os.getenv('TELEGRAM_CHAT_ID')
        url = f'https://api.telegram.org/bot{token}/sendMessage'
        data = {
            'chat_id': chat_id,
            'text': content,
            'parse_mode': 'Markdown'
        }
        response = requests.post(url, data=data)
        return response.json()
    
    @staticmethod
    def push_to_email(subject, content):
        """Push via SMTP email"""
        import smtplib
        from email.mime.text import MIMEText
        
        msg = MIMEText(content, 'html', 'utf-8')
        msg['Subject'] = subject
        msg['From'] = os.getenv('SMTP_FROM')
        msg['To'] = os.getenv('SMTP_TO')
        
        server = smtplib.SMTP(os.getenv('SMTP_SERVER'), int(os.getenv('SMTP_PORT')))
        server.starttls()
        server.login(os.getenv('SMTP_USERNAME'), os.getenv('SMTP_PASSWORD'))
        server.send_message(msg)
        server.quit()
        return True

# Example: Create automated push task
def create_push_task(keywords, channels, schedule='0 9,18 * * *'):
    """
    Create scheduled push task
    
    Args:
        keywords: Search keywords (e.g., "人工智能,AI,大模型")
        channels: List of push channels ['wechat', 'telegram', 'email']
        schedule: Cron expression for scheduling
    """
    import pymysql
    
    db = pymysql.connect(
        host=os.getenv('MYSQL_HOST'),
        user=os.getenv('MYSQL_USER'),
        password=os.getenv('MYSQL_PASSWORD'),
        database=os.getenv('MYSQL_DATABASE')
    )
    
    with db.cursor() as cursor:
        sql = """
            INSERT INTO push_tasks (task_name, query_keywords, push_channels, schedule_cron)
            VALUES (%s, %s, %s, %s)
        """
        import json
        cursor.execute(sql, (
            f'Daily AI News Alert',
            keywords,
            json.dumps(channels),
            schedule
        ))
    
    db.commit()
    db.close()

# Test push notification
if __name__ == '__main__':
    notifier = PushNotifier()
    
    report = """
    ## 今日AI热点分析
    
    **时间**: 2026-05-18
    
    ### 核心发现
    - GPT-6曝光,上下文窗口达200万token
    - DeepSeek V4采用华为昇腾算力
    - 国内大模型调用量连续领先
    
    [查看详细报告](http://localhost:5000/report/latest)
    """
    
    # Push to all channels
    notifier.push_to_wechat_work(report)
    notifier.push_to_telegram(report)
    notifier.push_to_email('AI热点日报', report)
```

## Common Usage Patterns

### Pattern 1: Real-time Topic Monitoring

```python
from hotsearch_analysis_agent import AnalysisAgent

agent = AnalysisAgent()

# Monitor specific keywords
result = agent.analyze_query("最近关于人工智能的热点新闻")

# Access clustered topics
for cluster in result['clusters']:
    print(f"主题: {cluster['theme']}")
    print(f"情感: {cluster['sentiment']}")
    print(f"相关新闻数: {len(cluster['items'])}")
```

### Pattern 2: Scheduled Report Generation

```python
from apscheduler.schedulers.background import BackgroundScheduler
from hotsearch_analysis_agent import AnalysisAgent
from test_push_task import PushNotifier

def generate_daily_report():
    agent = AnalysisAgent()
    notifier = PushNotifier()
    
    # Query and analyze
    result = agent.analyze_query("今日科技热点")
    
    # Format report
    report = f"""
    ## 舆情日报 - {datetime.now().strftime('%Y-%m-%d')}
    
    **总计热搜**: {result['total_results']}条
    
    ### 主题分布
    """
    
    for cluster in result['clusters']:
        report += f"\n- **{cluster['theme']}** ({cluster['sentiment']})"
    
    # Push to channels
    notifier.push_to_wechat_work(report)

# Schedule daily at 9 AM and 6 PM
scheduler = BackgroundScheduler()
scheduler.add_job(generate_daily_report, 'cron', hour='9,18')
scheduler.start()
```

### Pattern 3: Platform-Specific Analysis

```python
# Analyze sentiment across different platforms
def compare_platforms(keywords):
    agent = AnalysisAgent()
    
    platforms = ['weibo', 'zhihu', 'bilibili', 'douyin']
    results = {}
    
    for platform in platforms:
        news = agent.query_hotsearch(keywords, platforms=[platform])
        
        sentiments = [agent.sentiment_analysis(item['title']) for item in news[:10]]
        
        positive = sentiments.count('正面')
        negative = sentiments.count('负面')
        neutral = sentiments.count('中性')
        
        results[platform] = {
            'total': len(news),
            'positive': positive,
            'negative': negative,
            'neutral': neutral
        }
    
    return results

# Usage
comparison = compare_platforms("新能源汽车")
for platform, stats in comparison.items():
    print(f"{platform}: {stats}")
```

## Troubleshooting

### Issue: ChromeDriver not found

```bash
# Verify driver in PATH
which chromedriver  # Linux/macOS
where chromedriver  # Windows

# If not found, add manually
export PATH=$PATH:/usr/local/bin/chromedriver
```

### Issue: MySQL connection refused

```python
# Check MySQL service
# Linux
sudo systemctl status mysql

# macOS
brew services list

# Test connection
import pymysql
try:
    conn = pymysql.connect(host='localhost', user='root', password='password')
    print("Connection successful")
except Exception as e:
    print(f"Error: {e}")
```

### Issue: LLM API timeout

```python
# Increase timeout in agent initialization
from openai import OpenAI

client = OpenAI(
    api_key=os.getenv('OPENAI_API_KEY'),
    timeout=120.0,  # Increase to 120 seconds
    max_retries=3
)
```

### Issue: Empty crawler results

```python
# Check crawler logs
tail -f logs/crawler.log

# Test individual spider with verbose output
scrapy crawl weibo_spider -L DEBUG

# Verify database insertion
import pymysql
conn = pymysql.connect(host='localhost', user='root', password='password', database='hotsearch_db')
cursor = conn.cursor()
cursor.execute("SELECT COUNT(*) FROM hot_search WHERE crawl_time > NOW() - INTERVAL 1 HOUR")
print(f"Recent entries: {cursor.fetchone()[0]}")
```

### Issue: Push notification fails

```python
# Test webhook connectivity
import requests

webhook = os.getenv('WECHAT_WORK_WEBHOOK')
response = requests.post(webhook, json={
    "msgtype": "text",
    "text": {"content": "Test message"}
})

print(f"Status: {response.status_code}")
print(f"Response: {response.json()}")

# Verify environment variables
import os
required_vars = ['WECHAT_WORK_WEBHOOK', 'TELEGRAM_BOT_TOKEN', 'SMTP_USERNAME']
for var in required_vars:
    print(f"{var}: {'✓' if os.getenv(var) else '✗'}")
```

## Advanced Configuration

### Using Huawei Pangu Model Locally

```python
# Download model from: https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

from transformers import AutoModelForCausalLM, AutoTokenizer

model_path = "/path/to/openpangu-embedded-7b-model"
tokenizer = AutoTokenizer.from_pretrained(model_path)
model = AutoModelForCausalLM.from_pretrained(model_path, device_map="auto")

def analyze_with_pangu(text):
    inputs = tokenizer(text, return_tensors="pt")
    outputs = model.generate(**inputs, max_length=512)
    return tokenizer.decode(outputs[0], skip_special_tokens=True)
```

### Custom Spider Development

```python
# hotsearchcrawler/spiders/custom_platform_spider.py
import scrapy

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://example.com/hot-topics']
    
    def parse(self, response):
        for item in response.css('.hot-item'):
            yield {
                'platform': self.name,
                'title': item.css('.title::text').get(),
                'url': item.css('a::attr(href)').get(),
                'rank': item.css('.rank::text').get(),
                'heat_value': item.css('.heat::text').get(),
                'crawl_time': datetime.now()
            }
```

This skill provides comprehensive guidance for deploying and using the LLM-Based Intelligent Public Opinion Analytics Assistant for monitoring, analyzing, and reporting on trending topics across multiple Chinese social platforms.
