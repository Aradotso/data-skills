---
name: llm-public-opinion-analytics-assistant
description: Python-based public opinion analytics system integrating 15 platforms, 26 trending lists, web scraping, LLM analysis, sentiment detection, and multi-channel alerts
triggers:
  - set up public opinion monitoring system
  - analyze trending topics across platforms
  - scrape hot search data from social media
  - configure sentiment analysis with LLM
  - send automated trend alerts via WeChat
  - cluster news topics using AI
  - deploy Chinese social media crawler
  - integrate Pangu model for opinion analysis
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is a comprehensive public opinion analytics assistant that combines real-time data from **15 mainstream platforms** (26 trending lists total) with large language model (LLM) analysis capabilities. It enables conversational hot search queries, topic-specific searches, topic clustering, and sentiment analysis through a web interface.

**Key capabilities:**
- Multi-platform web scraping (Weibo, Bilibili, Douyin, Baidu, etc.)
- LLM-powered analysis using Huawei Pangu or OpenAI-compatible models
- Topic clustering and sentiment analysis
- Multi-channel push notifications (Email, WeChat, Enterprise WeChat, Telegram)
- Video content extraction and analysis
- Keyboard shortcut crawler control

## Installation

### Prerequisites

**1. Browser Driver Setup**

For Edge:
```bash
# Download EdgeDriver matching your Edge version
# From: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Place driver in system PATH or project root
# Verify installation:
msedgedriver --version
```

For Chrome:
```bash
# Download ChromeDriver matching your Chrome version
# From: https://chromedriver.chromium.org/

# Verify installation:
chromedriver --version
```

**2. MySQL Database**

```sql
-- Create database (refer to init.py for schema)
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

**3. Python Environment**

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Core Dependencies

```txt
scrapy>=2.11.0
selenium>=4.15.0
openai>=1.0.0
pymysql>=1.1.0
flask>=3.0.0
requests>=2.31.0
beautifulsoup4>=4.12.0
lxml>=4.9.0
python-dotenv>=1.0.0
```

## Configuration

### Environment Variables (.env)

Create a `.env` file in the project root:

```bash
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=your_password
MYSQL_DB=hotsearch_db

# LLM API Configuration (OpenAI-compatible format)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://your-api-endpoint.com/v1
OPENAI_MODEL=gpt-4

# Or use Huawei Pangu model endpoint
# OPENAI_API_BASE=http://localhost:8000/v1
# OPENAI_MODEL=openpangu-embedded-7b

# Push Notification Channels
# Email (SMTP)
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
EMAIL_RECIPIENTS=recipient1@example.com,recipient2@example.com

# Enterprise WeChat Robot
WECHAT_ROBOT_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

### Crawler Configuration (hotsearchcrawler/settings.py)

```python
# MySQL Pipeline Settings
MYSQL_CONFIG = {
    'host': os.getenv('MYSQL_HOST', 'localhost'),
    'port': int(os.getenv('MYSQL_PORT', 3306)),
    'user': os.getenv('MYSQL_USER', 'root'),
    'password': os.getenv('MYSQL_PASSWORD'),
    'database': os.getenv('MYSQL_DB'),
    'charset': 'utf8mb4'
}

# Scrapy Settings
ROBOTSTXT_OBEY = False
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
COOKIES_ENABLED = True

# User Agent Rotation
USER_AGENT = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'
```

## Database Initialization

```python
# Reference: init.py
import pymysql
from dotenv import load_dotenv
import os

load_dotenv()

connection = pymysql.connect(
    host=os.getenv('MYSQL_HOST'),
    user=os.getenv('MYSQL_USER'),
    password=os.getenv('MYSQL_PASSWORD'),
    database=os.getenv('MYSQL_DB'),
    charset='utf8mb4'
)

# Create tables for trending data
with connection.cursor() as cursor:
    # Hot search table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS hot_search (
            id INT AUTO_INCREMENT PRIMARY KEY,
            platform VARCHAR(50),
            title VARCHAR(500),
            url VARCHAR(1000),
            hot_value VARCHAR(100),
            rank_position INT,
            category VARCHAR(100),
            collected_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            INDEX idx_platform (platform),
            INDEX idx_collected_at (collected_at)
        ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
    """)
    
    # Detailed content table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS content_detail (
            id INT AUTO_INCREMENT PRIMARY KEY,
            hot_search_id INT,
            content TEXT,
            summary TEXT,
            sentiment VARCHAR(50),
            keywords VARCHAR(500),
            analyzed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY (hot_search_id) REFERENCES hot_search(id)
        ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
    """)
    
    connection.commit()
```

## Running the Application

### Start the Analysis System

```bash
# Start Flask web application
python app.py
```

The web interface will be available at `http://localhost:5000`

### Run Crawlers

**Via Web Interface:**
- Access the dashboard
- Use keyboard shortcuts to start/stop crawlers
- Monitor real-time scraping status

**Via Command Line:**

```bash
# Run specific spider
cd hotsearchcrawler
scrapy crawl weibo_spider

# Run all spiders
python run_spiders.py
```

**Test Individual Spider:**

```bash
# Test spider functionality
python runspider-test.py --spider=bilibili
```

## Key Usage Patterns

### 1. Conversational Query Interface

```python
# hotsearch_analysis_agent/query_handler.py
from openai import OpenAI
import os

client = OpenAI(
    api_key=os.getenv('OPENAI_API_KEY'),
    base_url=os.getenv('OPENAI_API_BASE')
)

def analyze_trending_topic(user_query: str, trending_data: list) -> str:
    """Analyze trending topics using LLM"""
    
    # Prepare context from scraped data
    context = "\n".join([
        f"- {item['title']} (Platform: {item['platform']}, Rank: {item['rank_position']})"
        for item in trending_data[:20]
    ])
    
    prompt = f"""
    Based on the following trending topics from multiple platforms:
    
    {context}
    
    User query: {user_query}
    
    Please provide a comprehensive analysis including:
    1. Key themes and patterns
    2. Sentiment trends
    3. Notable events
    4. Cross-platform correlations
    """
    
    response = client.chat.completions.create(
        model=os.getenv('OPENAI_MODEL'),
        messages=[
            {"role": "system", "content": "You are a public opinion analysis expert."},
            {"role": "user", "content": prompt}
        ],
        temperature=0.7,
        max_tokens=2000
    )
    
    return response.choices[0].message.content
```

### 2. Topic Clustering

```python
# hotsearch_analysis_agent/clustering.py
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.cluster import KMeans
import jieba

def cluster_topics(titles: list[str], n_clusters: int = 5) -> dict:
    """Cluster trending topics using TF-IDF and K-Means"""
    
    # Tokenize Chinese text
    tokenized = [' '.join(jieba.cut(title)) for title in titles]
    
    # Vectorize
    vectorizer = TfidfVectorizer(max_features=100)
    X = vectorizer.fit_transform(tokenized)
    
    # Cluster
    kmeans = KMeans(n_clusters=n_clusters, random_state=42)
    labels = kmeans.fit_predict(X)
    
    # Group by cluster
    clusters = {}
    for idx, label in enumerate(labels):
        if label not in clusters:
            clusters[label] = []
        clusters[label].append({
            'title': titles[idx],
            'index': idx
        })
    
    return clusters
```

### 3. Sentiment Analysis with LLM

```python
# hotsearch_analysis_agent/sentiment.py
def analyze_sentiment_batch(contents: list[str]) -> list[dict]:
    """Batch sentiment analysis for trending content"""
    
    results = []
    for content in contents:
        prompt = f"""
        Analyze the sentiment of this news content:
        
        {content[:500]}  # Limit length
        
        Classify as: positive, negative, or neutral
        Provide confidence score (0-1) and brief reasoning.
        
        Response format:
        {{"sentiment": "...", "confidence": 0.0, "reason": "..."}}
        """
        
        response = client.chat.completions.create(
            model=os.getenv('OPENAI_MODEL'),
            messages=[{"role": "user", "content": prompt}],
            temperature=0.3
        )
        
        import json
        result = json.loads(response.choices[0].message.content)
        results.append(result)
    
    return results
```

### 4. Multi-Platform Scraping

```python
# hotsearchcrawler/spiders/weibo_spider.py
import scrapy
from selenium import webdriver
from selenium.webdriver.common.by import By

class WeiboSpider(scrapy.Spider):
    name = 'weibo_spider'
    start_urls = ['https://s.weibo.com/top/summary']
    
    def __init__(self):
        options = webdriver.EdgeOptions()
        options.add_argument('--headless')
        self.driver = webdriver.Edge(options=options)
    
    def parse(self, response):
        self.driver.get(response.url)
        
        # Wait for dynamic content
        import time
        time.sleep(2)
        
        items = self.driver.find_elements(By.CSS_SELECTOR, 'tr.td-01')
        
        for idx, item in enumerate(items[:50]):
            try:
                title_elem = item.find_element(By.CSS_SELECTOR, 'a')
                hot_value_elem = item.find_element(By.CSS_SELECTOR, 'span.td-02')
                
                yield {
                    'platform': 'weibo',
                    'title': title_elem.text,
                    'url': title_elem.get_attribute('href'),
                    'hot_value': hot_value_elem.text,
                    'rank_position': idx + 1,
                    'category': self.extract_category(item)
                }
            except Exception as e:
                self.logger.error(f"Error parsing item: {e}")
    
    def extract_category(self, element):
        try:
            icon = element.find_element(By.CSS_SELECTOR, 'span.ico-icon')
            return icon.get_attribute('alt') or 'general'
        except:
            return 'general'
    
    def closed(self, reason):
        self.driver.quit()
```

### 5. Push Notification System

```python
# hotsearch_analysis_agent/push_task.py
import requests
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart

class PushTaskManager:
    """Manage multi-channel push notifications"""
    
    def send_email_report(self, subject: str, content: str):
        """Send analysis report via email"""
        
        msg = MIMEMultipart()
        msg['From'] = os.getenv('SMTP_USER')
        msg['To'] = os.getenv('EMAIL_RECIPIENTS')
        msg['Subject'] = subject
        
        msg.attach(MIMEText(content, 'html', 'utf-8'))
        
        with smtplib.SMTP(os.getenv('SMTP_SERVER'), int(os.getenv('SMTP_PORT'))) as server:
            server.starttls()
            server.login(os.getenv('SMTP_USER'), os.getenv('SMTP_PASSWORD'))
            server.send_message(msg)
    
    def send_wechat_alert(self, content: str):
        """Send alert to Enterprise WeChat group"""
        
        webhook_url = os.getenv('WECHAT_ROBOT_WEBHOOK')
        data = {
            "msgtype": "markdown",
            "markdown": {
                "content": content
            }
        }
        
        response = requests.post(webhook_url, json=data)
        return response.status_code == 200
    
    def send_telegram_message(self, text: str):
        """Send message via Telegram bot"""
        
        url = f"https://api.telegram.org/bot{os.getenv('TELEGRAM_BOT_TOKEN')}/sendMessage"
        data = {
            "chat_id": os.getenv('TELEGRAM_CHAT_ID'),
            "text": text,
            "parse_mode": "Markdown"
        }
        
        response = requests.post(url, json=data)
        return response.json()
```

### 6. Automated Report Generation

```python
# test_push_task.py - Scheduled report generation
from datetime import datetime
from hotsearch_analysis_agent.query_handler import analyze_trending_topic
from hotsearch_analysis_agent.push_task import PushTaskManager

def generate_daily_report(query_topic: str = "人工智能与前沿科技"):
    """Generate and push daily analysis report"""
    
    # Fetch trending data from database
    import pymysql
    connection = pymysql.connect(
        host=os.getenv('MYSQL_HOST'),
        user=os.getenv('MYSQL_USER'),
        password=os.getenv('MYSQL_PASSWORD'),
        database=os.getenv('MYSQL_DB')
    )
    
    with connection.cursor(pymysql.cursors.DictCursor) as cursor:
        cursor.execute("""
            SELECT * FROM hot_search 
            WHERE collected_at >= DATE_SUB(NOW(), INTERVAL 24 HOUR)
            ORDER BY rank_position ASC
            LIMIT 100
        """)
        trending_data = cursor.fetchall()
    
    # Generate analysis report
    report_content = analyze_trending_topic(query_topic, trending_data)
    
    # Format report
    report_html = f"""
    <h2>Report - {query_topic} Daily Analysis</h2>
    <p><strong>Time</strong>: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}</p>
    <hr>
    {report_content}
    """
    
    # Push to multiple channels
    push_manager = PushTaskManager()
    push_manager.send_email_report(
        subject=f"Public Opinion Report - {query_topic}",
        content=report_html
    )
    push_manager.send_wechat_alert(report_content[:1000])  # WeChat has length limit
    
    print("Report generated and pushed successfully")

if __name__ == "__main__":
    generate_daily_report()
```

## Troubleshooting

### Browser Driver Issues

```python
# Verify driver is accessible
import subprocess

try:
    result = subprocess.run(['chromedriver', '--version'], 
                          capture_output=True, text=True)
    print(f"ChromeDriver found: {result.stdout}")
except FileNotFoundError:
    print("ChromeDriver not in PATH. Add to system PATH or specify location.")
```

### MySQL Connection Errors

```python
# Test database connection
import pymysql

try:
    connection = pymysql.connect(
        host=os.getenv('MYSQL_HOST'),
        user=os.getenv('MYSQL_USER'),
        password=os.getenv('MYSQL_PASSWORD')
    )
    print("MySQL connection successful")
    connection.close()
except pymysql.Error as e:
    print(f"MySQL connection failed: {e}")
    print("Check MYSQL_* environment variables in .env")
```

### LLM API Timeout

```python
# Add timeout and retry logic
from openai import OpenAI
import time

def llm_call_with_retry(prompt: str, max_retries: int = 3):
    client = OpenAI(
        api_key=os.getenv('OPENAI_API_KEY'),
        base_url=os.getenv('OPENAI_API_BASE'),
        timeout=30.0  # 30 second timeout
    )
    
    for attempt in range(max_retries):
        try:
            response = client.chat.completions.create(
                model=os.getenv('OPENAI_MODEL'),
                messages=[{"role": "user", "content": prompt}]
            )
            return response.choices[0].message.content
        except Exception as e:
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)  # Exponential backoff
                continue
            raise e
```

### Crawler Getting Blocked

```python
# Rotate user agents and add delays
import random

USER_AGENTS = [
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
    'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36',
    'Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36'
]

# In spider settings
DOWNLOADER_MIDDLEWARES = {
    'scrapy.downloadermiddlewares.useragent.UserAgentMiddleware': None,
    'scrapy_user_agents.middlewares.RandomUserAgentMiddleware': 400,
}

# Add random delays
DOWNLOAD_DELAY = 2
RANDOMIZE_DOWNLOAD_DELAY = True
```

## Common Workflows

**Daily monitoring setup:**
```bash
# Schedule with cron (Linux/macOS)
# Run crawlers every 2 hours
0 */2 * * * cd /path/to/project && python run_spiders.py

# Generate reports daily at 8 AM
0 8 * * * cd /path/to/project && python test_push_task.py
```

**Query trending topics:**
```python
# Via API or web interface
query = "最近人工智能领域有什么新闻"
results = analyze_trending_topic(query, get_recent_trends())
```

**Export analysis results:**
```python
# Export to CSV/JSON
import pandas as pd

df = pd.read_sql("SELECT * FROM hot_search", connection)
df.to_csv('trending_export.csv', index=False, encoding='utf-8-sig')
```
