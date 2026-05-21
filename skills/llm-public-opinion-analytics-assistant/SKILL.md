---
name: llm-public-opinion-analytics-assistant
description: Multi-platform hot search crawler with LLM-powered sentiment analysis, topic clustering, and multi-channel alert push system
triggers:
  - set up Chinese social media monitoring with AI analysis
  - crawl hot search rankings from Weibo Bilibili Douyin
  - analyze public opinion sentiment across multiple platforms
  - configure sentiment analysis push notifications to WeChat
  - cluster trending topics using large language models
  - build a real-time hot topic monitoring dashboard
  - scrape news details from video platforms for analysis
  - deploy public opinion analytics system with Huawei Pangu
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is an intelligent public opinion analytics assistant that integrates **15 mainstream Chinese platforms** with **26 different ranking lists**. It combines real-time web scraping with large language model (LLM) analysis capabilities to provide conversational hot search queries, topic clustering, sentiment analysis, and multi-channel push notifications (email, WeChat, Enterprise WeChat, Telegram).

Key features:
- Multi-platform crawler system (Weibo, Bilibili, Douyin, Zhihu, etc.)
- LLM-powered sentiment analysis and topic clustering
- Conversational interface for hot search queries
- Video content extraction and analysis
- Multi-channel alert push system
- Keyboard shortcut-controlled crawler management

## Installation

### Prerequisites

**1. Browser Driver Setup**

Download the appropriate driver for your browser:
- **Chrome**: [ChromeDriver](https://chromedriver.chromium.org/)
- **Edge**: [EdgeDriver](https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/)

Match driver version to your browser version, then add to system PATH:

```bash
# Linux/macOS - add to ~/.bashrc or ~/.zshrc
export PATH=$PATH:/path/to/driver/directory

# Verify installation
chromedriver --version
```

**2. Database Setup**

Install MySQL and create the database:

```bash
mysql -u root -p
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

**3. Python Environment**

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Configuration

**1. Environment Variables**

Create `.env` file in project root:

```env
# Database Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch_db

# OpenAI-Compatible LLM API (Huawei Pangu or alternative)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://your-llm-endpoint.com/v1

# Push Notification Channels (optional)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password

WECHAT_CORP_ID=your_corp_id
WECHAT_CORP_SECRET=your_corp_secret
WECHAT_AGENT_ID=your_agent_id

TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

**2. Crawler Configuration**

Edit `hotsearchcrawler/settings.py`:

```python
# MySQL Connection
MYSQL_CONFIG = {
    'host': os.getenv('MYSQL_HOST', 'localhost'),
    'port': int(os.getenv('MYSQL_PORT', 3306)),
    'user': os.getenv('MYSQL_USER', 'root'),
    'password': os.getenv('MYSQL_PASSWORD'),
    'database': os.getenv('MYSQL_DATABASE'),
    'charset': 'utf8mb4'
}

# Optional: Platform-specific cookies
COOKIES = {
    'weibo': 'your_weibo_cookies',
    'bilibili': 'your_bilibili_cookies'
}
```

**3. Database Initialization**

Run the initialization script:

```python
# init.py
import pymysql
from dotenv import load_dotenv
import os

load_dotenv()

conn = pymysql.connect(
    host=os.getenv('MYSQL_HOST'),
    port=int(os.getenv('MYSQL_PORT')),
    user=os.getenv('MYSQL_USER'),
    password=os.getenv('MYSQL_PASSWORD'),
    database=os.getenv('MYSQL_DATABASE'),
    charset='utf8mb4'
)

cursor = conn.cursor()

# Create hot search table
cursor.execute("""
CREATE TABLE IF NOT EXISTS hot_search (
    id INT AUTO_INCREMENT PRIMARY KEY,
    platform VARCHAR(50) NOT NULL,
    title VARCHAR(500) NOT NULL,
    url VARCHAR(1000),
    hot_value VARCHAR(50),
    rank_position INT,
    crawl_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    content TEXT,
    sentiment VARCHAR(20),
    INDEX idx_platform (platform),
    INDEX idx_crawl_time (crawl_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
""")

# Create analysis results table
cursor.execute("""
CREATE TABLE IF NOT EXISTS analysis_results (
    id INT AUTO_INCREMENT PRIMARY KEY,
    query_text TEXT NOT NULL,
    analysis_type VARCHAR(50),
    result_data JSON,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
""")

conn.commit()
cursor.close()
conn.close()
```

## Starting the System

### Launch Web Interface

```bash
python app.py
```

Access the dashboard at `http://localhost:5000`

### Run Crawler System

**Option 1: Via Web Interface**
- Use keyboard shortcuts in the web UI to start/stop crawlers

**Option 2: Manual Execution**

```bash
# Test single spider
python runspider-test.py

# Run all spiders
python run_spiders.py
```

## Core API Usage

### 1. Crawler System

The crawler module uses Scrapy framework. Example spider structure:

```python
# hotsearchcrawler/spiders/weibo_spider.py
import scrapy
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
import time

class WeiboSpider(scrapy.Spider):
    name = 'weibo'
    start_urls = ['https://s.weibo.com/top/summary']
    
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        options = webdriver.ChromeOptions()
        options.add_argument('--headless')
        self.driver = webdriver.Chrome(options=options)
    
    def parse(self, response):
        self.driver.get(response.url)
        time.sleep(3)
        
        # Extract hot search items
        items = self.driver.find_elements(By.CSS_SELECTOR, 'tr td.td-02 a')
        
        for idx, item in enumerate(items[:50], 1):
            yield {
                'platform': 'weibo',
                'title': item.text,
                'url': item.get_attribute('href'),
                'rank_position': idx,
                'hot_value': self.extract_hot_value(item),
                'crawl_time': time.strftime('%Y-%m-%d %H:%M:%S')
            }
    
    def extract_hot_value(self, element):
        try:
            hot_span = element.find_element(By.CSS_SELECTOR, 'span.hot')
            return hot_span.text
        except:
            return '0'
    
    def closed(self, reason):
        self.driver.quit()
```

### 2. LLM Analysis Agent

The analysis system uses OpenAI-compatible API:

```python
# hotsearch_analysis_agent/llm_analyzer.py
import openai
import os
from typing import List, Dict
import json

class LLMAnalyzer:
    def __init__(self):
        openai.api_key = os.getenv('OPENAI_API_KEY')
        openai.api_base = os.getenv('OPENAI_API_BASE')
        self.model = os.getenv('LLM_MODEL', 'pangu-embedded-7b')
    
    def sentiment_analysis(self, texts: List[str]) -> List[Dict]:
        """Analyze sentiment of multiple texts"""
        prompt = f"""分析以下新闻标题的情感倾向,返回JSON格式:
        [{{"text": "标题", "sentiment": "positive/negative/neutral", "score": 0-1}}]
        
        标题列表:
        {json.dumps(texts, ensure_ascii=False)}
        """
        
        response = openai.ChatCompletion.create(
            model=self.model,
            messages=[
                {"role": "system", "content": "你是专业的舆情分析助手"},
                {"role": "user", "content": prompt}
            ],
            temperature=0.3
        )
        
        return json.loads(response.choices[0].message.content)
    
    def topic_clustering(self, items: List[Dict]) -> Dict:
        """Cluster related topics"""
        titles = [item['title'] for item in items]
        prompt = f"""将以下热搜话题进行聚类分析,识别关联话题并分组。
        返回JSON格式: {{"clusters": [{{"theme": "主题名", "items": ["话题1", "话题2"]}}]}}
        
        话题列表:
        {json.dumps(titles, ensure_ascii=False)}
        """
        
        response = openai.ChatCompletion.create(
            model=self.model,
            messages=[
                {"role": "system", "content": "你是专业的话题聚类分析专家"},
                {"role": "user", "content": prompt}
            ],
            temperature=0.5
        )
        
        return json.loads(response.choices[0].message.content)
    
    def generate_report(self, query: str, data: List[Dict]) -> str:
        """Generate analytical report based on query"""
        prompt = f"""基于以下热搜数据,回答用户查询并生成分析报告:
        
        用户查询: {query}
        
        数据:
        {json.dumps(data, ensure_ascii=False, indent=2)}
        
        请提供:
        1. 核心发现与数据亮点
        2. 详细新闻内容梳理
        3. 分析与总结
        4. 信息传播特点
        """
        
        response = openai.ChatCompletion.create(
            model=self.model,
            messages=[
                {"role": "system", "content": "你是专业的舆情分析报告撰写专家"},
                {"role": "user", "content": prompt}
            ],
            temperature=0.7
        )
        
        return response.choices[0].message.content
```

### 3. Query Interface

```python
# hotsearch_analysis_agent/query_handler.py
import pymysql
from .llm_analyzer import LLMAnalyzer

class QueryHandler:
    def __init__(self):
        self.analyzer = LLMAnalyzer()
        self.db_config = {
            'host': os.getenv('MYSQL_HOST'),
            'user': os.getenv('MYSQL_USER'),
            'password': os.getenv('MYSQL_PASSWORD'),
            'database': os.getenv('MYSQL_DATABASE'),
            'charset': 'utf8mb4'
        }
    
    def handle_query(self, query: str) -> str:
        """Handle natural language query"""
        # Fetch relevant data from database
        data = self.fetch_relevant_data(query)
        
        # Generate analysis report
        report = self.analyzer.generate_report(query, data)
        
        return report
    
    def fetch_relevant_data(self, query: str) -> List[Dict]:
        """Fetch data from MySQL based on query"""
        conn = pymysql.connect(**self.db_config)
        cursor = conn.cursor(pymysql.cursors.DictCursor)
        
        # Use LLM to extract keywords
        keywords = self.extract_keywords(query)
        
        # Build SQL query
        conditions = " OR ".join([f"title LIKE '%{kw}%'" for kw in keywords])
        sql = f"""
        SELECT platform, title, url, hot_value, crawl_time, content
        FROM hot_search
        WHERE {conditions}
        ORDER BY crawl_time DESC
        LIMIT 100
        """
        
        cursor.execute(sql)
        results = cursor.fetchall()
        
        cursor.close()
        conn.close()
        
        return results
    
    def extract_keywords(self, query: str) -> List[str]:
        """Extract keywords from query using LLM"""
        # Implementation using LLM to extract key search terms
        pass
```

### 4. Push Notification System

```python
# test_push_task.py
import os
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
import requests
import json

class PushNotifier:
    def __init__(self):
        self.smtp_config = {
            'host': os.getenv('SMTP_HOST'),
            'port': int(os.getenv('SMTP_PORT')),
            'user': os.getenv('SMTP_USER'),
            'password': os.getenv('SMTP_PASSWORD')
        }
        self.wechat_config = {
            'corp_id': os.getenv('WECHAT_CORP_ID'),
            'corp_secret': os.getenv('WECHAT_CORP_SECRET'),
            'agent_id': os.getenv('WECHAT_AGENT_ID')
        }
        self.telegram_config = {
            'bot_token': os.getenv('TELEGRAM_BOT_TOKEN'),
            'chat_id': os.getenv('TELEGRAM_CHAT_ID')
        }
    
    def send_email(self, subject: str, content: str, to_addr: str):
        """Send email notification"""
        msg = MIMEMultipart()
        msg['From'] = self.smtp_config['user']
        msg['To'] = to_addr
        msg['Subject'] = subject
        
        msg.attach(MIMEText(content, 'html', 'utf-8'))
        
        with smtplib.SMTP(self.smtp_config['host'], self.smtp_config['port']) as server:
            server.starttls()
            server.login(self.smtp_config['user'], self.smtp_config['password'])
            server.send_message(msg)
    
    def send_wechat(self, content: str):
        """Send Enterprise WeChat notification"""
        # Get access token
        token_url = f"https://qyapi.weixin.qq.com/cgi-bin/gettoken?corpid={self.wechat_config['corp_id']}&corpsecret={self.wechat_config['corp_secret']}"
        token_resp = requests.get(token_url).json()
        access_token = token_resp['access_token']
        
        # Send message
        send_url = f"https://qyapi.weixin.qq.com/cgi-bin/message/send?access_token={access_token}"
        data = {
            "touser": "@all",
            "msgtype": "text",
            "agentid": self.wechat_config['agent_id'],
            "text": {"content": content}
        }
        requests.post(send_url, json=data)
    
    def send_telegram(self, content: str):
        """Send Telegram notification"""
        url = f"https://api.telegram.org/bot{self.telegram_config['bot_token']}/sendMessage"
        data = {
            "chat_id": self.telegram_config['chat_id'],
            "text": content,
            "parse_mode": "Markdown"
        }
        requests.post(url, json=data)
    
    def push_hot_topics(self, report: str, channels: List[str] = ['email']):
        """Push hot topic report to multiple channels"""
        for channel in channels:
            if channel == 'email':
                self.send_email(
                    subject="舆情热点分析报告",
                    content=report,
                    to_addr="recipient@example.com"
                )
            elif channel == 'wechat':
                self.send_wechat(report)
            elif channel == 'telegram':
                self.send_telegram(report)

# Usage example
if __name__ == '__main__':
    notifier = PushNotifier()
    report = "生成的分析报告内容..."
    notifier.push_hot_topics(report, channels=['email', 'wechat', 'telegram'])
```

## Common Patterns

### Pattern 1: Scheduled Crawler Execution

```python
# scheduled_crawler.py
import schedule
import time
from run_spiders import run_all_spiders

def job():
    print("Starting scheduled crawl...")
    run_all_spiders()
    print("Crawl completed.")

# Run every 30 minutes
schedule.every(30).minutes.do(job)

while True:
    schedule.run_pending()
    time.sleep(60)
```

### Pattern 2: Real-time Query Processing

```python
# app.py (Flask integration)
from flask import Flask, request, jsonify
from hotsearch_analysis_agent.query_handler import QueryHandler

app = Flask(__name__)
handler = QueryHandler()

@app.route('/query', methods=['POST'])
def query():
    user_query = request.json.get('query')
    result = handler.handle_query(user_query)
    return jsonify({'result': result})

@app.route('/sentiment', methods=['POST'])
def sentiment():
    texts = request.json.get('texts', [])
    from hotsearch_analysis_agent.llm_analyzer import LLMAnalyzer
    analyzer = LLMAnalyzer()
    results = analyzer.sentiment_analysis(texts)
    return jsonify(results)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

### Pattern 3: Custom Platform Spider

```python
# Add new platform crawler
import scrapy

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://example.com/hot']
    
    custom_settings = {
        'DOWNLOAD_DELAY': 2,
        'CONCURRENT_REQUESTS': 1
    }
    
    def parse(self, response):
        # Extract data using CSS or XPath selectors
        for item in response.css('.hot-item'):
            yield {
                'platform': 'custom_platform',
                'title': item.css('.title::text').get(),
                'url': item.css('a::attr(href)').get(),
                'hot_value': item.css('.hot-value::text').get(),
                'rank_position': item.css('.rank::text').get()
            }
```

## Troubleshooting

### Issue: ChromeDriver version mismatch

**Error**: `SessionNotCreatedException: Message: session not created: This version of ChromeDriver only supports Chrome version XX`

**Solution**:
```bash
# Check Chrome version
google-chrome --version

# Download matching ChromeDriver from:
# https://chromedriver.chromium.org/downloads

# Update PATH or replace driver binary
```

### Issue: MySQL connection refused

**Error**: `pymysql.err.OperationalError: (2003, "Can't connect to MySQL server")`

**Solution**:
```bash
# Check MySQL service status
systemctl status mysql  # Linux
brew services list  # macOS

# Verify credentials in .env
# Check firewall allows port 3306
sudo ufw allow 3306/tcp
```

### Issue: LLM API timeout

**Error**: `openai.error.Timeout: Request timed out`

**Solution**:
```python
# Increase timeout in LLM analyzer
response = openai.ChatCompletion.create(
    model=self.model,
    messages=messages,
    timeout=60  # Increase to 60 seconds
)

# Or implement retry logic
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=4, max=10))
def call_llm(self, prompt):
    # LLM call implementation
    pass
```

### Issue: Encoding errors with Chinese text

**Error**: `UnicodeEncodeError: 'latin-1' codec can't encode characters`

**Solution**:
```python
# Ensure UTF-8 encoding throughout pipeline
# MySQL connection
conn = pymysql.connect(charset='utf8mb4')

# Scrapy settings
FEED_EXPORT_ENCODING = 'utf-8'

# File operations
with open('output.txt', 'w', encoding='utf-8') as f:
    f.write(chinese_text)
```

### Issue: Crawler blocked by platform

**Solution**:
```python
# Add user agent rotation and delays
USER_AGENT_LIST = [
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
    'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36'
]

# In spider settings
DOWNLOAD_DELAY = 3
RANDOMIZE_DOWNLOAD_DELAY = True
CONCURRENT_REQUESTS_PER_DOMAIN = 1

# Use cookies from browser session
COOKIES_ENABLED = True
```

## Advanced Configuration

### Deploy with Huawei Pangu Model

Download and configure the Huawei openPangu-Embedded-7B model:

```bash
# Clone model repository
git clone https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model.git

# Configure API endpoint
# Update .env with local model server
OPENAI_API_BASE=http://localhost:8000/v1
LLM_MODEL=openpangu-embedded-7b
```

### Production Deployment

```bash
# Use Gunicorn for production
pip install gunicorn

# Run with multiple workers
gunicorn -w 4 -b 0.0.0.0:5000 app:app

# Use supervisor for process management
[program:hotsearch]
command=/path/to/venv/bin/gunicorn -w 4 app:app
directory=/path/to/project
autostart=true
autorestart=true
```
