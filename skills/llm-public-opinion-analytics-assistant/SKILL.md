---
name: llm-public-opinion-analytics-assistant
description: Multi-platform public opinion analytics assistant with real-time hot search crawling, sentiment analysis, topic clustering, and multi-channel alert push using LLM capabilities
triggers:
  - analyze public opinion from multiple platforms
  - set up hot topic monitoring and alerts
  - crawl trending searches from social media
  - perform sentiment analysis on news articles
  - configure multi-channel push notifications for trending topics
  - cluster and analyze related topics across platforms
  - monitor real-time hot searches from weibo douyin bilibili
  - extract content from video news pages for analysis
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A Python-based public opinion analytics platform that integrates **15 mainstream platforms** with **26 real-time ranking lists**. Combines web crawling, LLM analysis, and multi-channel push notifications to monitor trending topics, perform sentiment analysis, topic clustering, and deliver actionable insights through email, WeChat, Enterprise WeChat, and Telegram.

## What It Does

- **Multi-Platform Crawling**: Collects real-time hot search data from 15+ platforms (Weibo, Douyin, Bilibili, etc.)
- **LLM-Powered Analysis**: Uses Huawei Pangu or OpenAI-compatible models for sentiment analysis, topic clustering, and content extraction
- **Content Extraction**: Extracts detailed content even from video-based news pages
- **Interactive Query**: Natural language interface for hot search queries and topic-specific searches
- **Multi-Channel Push**: Automated alerts via Enterprise WeChat, Telegram, and email (SMTP)
- **Keyboard-Controlled Crawlers**: Start/stop crawlers using hotkeys

## Installation

### Prerequisites

**Browser Driver Setup** (required for content extraction):

1. **Identify browser version**:
   - Open Chrome/Edge → Settings → About
   - Note the version number (e.g., `115.0.5790.102`)

2. **Download matching driver**:
   - Chrome: https://chromedriver.chromium.org/
   - Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

3. **Place driver in system PATH**:
   ```bash
   # Linux/macOS - move to /usr/local/bin
   sudo mv chromedriver /usr/local/bin/
   sudo chmod +x /usr/local/bin/chromedriver
   
   # Windows - add to PATH or place in C:\Windows\System32\
   ```

4. **Verify installation**:
   ```bash
   chromedriver --version
   ```

### Python Environment

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

### Database Setup

```bash
# Install MySQL (example for Ubuntu)
sudo apt-get install mysql-server

# Create database and tables
mysql -u root -p
```

```sql
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE hotsearch_db;

-- Reference init.py for complete schema
CREATE TABLE hot_searches (
    id INT AUTO_INCREMENT PRIMARY KEY,
    platform VARCHAR(50) NOT NULL,
    title VARCHAR(500) NOT NULL,
    url VARCHAR(1000),
    rank INT,
    heat_value BIGINT,
    fetch_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    content TEXT,
    sentiment VARCHAR(20),
    INDEX idx_platform (platform),
    INDEX idx_fetch_time (fetch_time)
);
```

## Configuration

### Environment Variables (.env)

Create a `.env` file in the project root:

```bash
# Database Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_db_user
MYSQL_PASSWORD=your_db_password
MYSQL_DATABASE=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key_here
OPENAI_API_BASE=https://api.openai.com/v1  # Or Pangu endpoint

# Push Notification Channels
# Enterprise WeChat Bot
WECHAT_WEBHOOK_URL=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_telegram_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email (SMTP)
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USERNAME=your_email@gmail.com
SMTP_PASSWORD=your_email_app_password
EMAIL_TO=recipient@example.com
```

### Crawler Configuration (hotsearchcrawler/settings.py)

```python
# MySQL Settings
MYSQL_HOST = os.getenv('MYSQL_HOST', 'localhost')
MYSQL_PORT = int(os.getenv('MYSQL_PORT', 3306))
MYSQL_USER = os.getenv('MYSQL_USER')
MYSQL_PASSWORD = os.getenv('MYSQL_PASSWORD')
MYSQL_DATABASE = os.getenv('MYSQL_DATABASE')

# Optional: Platform-specific cookies
COOKIES = {
    'weibo': 'YOUR_WEIBO_COOKIES',  # Optional for auth-required content
}

# Browser Configuration
BROWSER_TYPE = 'chrome'  # or 'edge'
HEADLESS_MODE = True
```

## Key Commands

### Start the Web Application

```bash
python app.py
```

Access the web interface at `http://localhost:5000`

### Run Crawlers Manually

```bash
# Test single platform crawler
python runspider-test.py --platform weibo

# Run all crawlers
python run_spiders.py

# Run specific crawler group
cd hotsearchcrawler
scrapy crawl weibo_spider
scrapy crawl douyin_spider
scrapy crawl bilibili_spider
```

### Test Push Notifications

```bash
python test_push_task.py --channel wechat --message "Test Alert"
python test_push_task.py --channel telegram --message "Hot topic detected"
python test_push_task.py --channel email --subject "Alert" --message "Content"
```

## API Usage Patterns

### Crawler Integration

```python
from hotsearchcrawler.spiders import WeiboSpider
from scrapy.crawler import CrawlerProcess
from scrapy.utils.project import get_project_settings

# Initialize crawler
process = CrawlerProcess(get_project_settings())

# Start Weibo crawler
process.crawl(WeiboSpider)
process.start()
```

### Database Operations

```python
import pymysql
from datetime import datetime

# Connect to database
conn = pymysql.connect(
    host=os.getenv('MYSQL_HOST'),
    port=int(os.getenv('MYSQL_PORT')),
    user=os.getenv('MYSQL_USER'),
    password=os.getenv('MYSQL_PASSWORD'),
    database=os.getenv('MYSQL_DATABASE'),
    charset='utf8mb4'
)

# Insert hot search data
def save_hot_search(platform, title, url, rank, heat_value, content=None):
    with conn.cursor() as cursor:
        sql = """
        INSERT INTO hot_searches 
        (platform, title, url, rank, heat_value, content, fetch_time)
        VALUES (%s, %s, %s, %s, %s, %s, %s)
        """
        cursor.execute(sql, (platform, title, url, rank, heat_value, content, datetime.now()))
    conn.commit()

# Query recent hot searches
def get_recent_searches(platform, hours=24):
    with conn.cursor(pymysql.cursors.DictCursor) as cursor:
        sql = """
        SELECT * FROM hot_searches 
        WHERE platform = %s AND fetch_time > DATE_SUB(NOW(), INTERVAL %s HOUR)
        ORDER BY rank ASC
        """
        cursor.execute(sql, (platform, hours))
        return cursor.fetchall()
```

### LLM Analysis Integration

```python
import openai
import os

# Configure OpenAI-compatible client
openai.api_key = os.getenv('OPENAI_API_KEY')
openai.api_base = os.getenv('OPENAI_API_BASE')

def analyze_sentiment(text):
    """Analyze sentiment of news article"""
    response = openai.ChatCompletion.create(
        model="gpt-4",  # or "pangu-7b" for local deployment
        messages=[
            {"role": "system", "content": "你是一个舆情分析专家，擅长情感倾向分析。"},
            {"role": "user", "content": f"分析以下新闻的情感倾向（正面/负面/中性）:\n\n{text}"}
        ],
        temperature=0.3
    )
    return response.choices[0].message.content

def cluster_topics(articles):
    """Cluster related topics"""
    titles = "\n".join([f"{i+1}. {a['title']}" for i, a in enumerate(articles)])
    
    response = openai.ChatCompletion.create(
        model="gpt-4",
        messages=[
            {"role": "system", "content": "你是话题聚类分析专家。"},
            {"role": "user", "content": f"将以下新闻标题进行聚类，识别主要话题:\n\n{titles}"}
        ]
    )
    return response.choices[0].message.content

# Example usage
articles = get_recent_searches('weibo', 24)
for article in articles[:5]:
    sentiment = analyze_sentiment(article['content'])
    print(f"{article['title']}: {sentiment}")
```

### Content Extraction from Video Pages

```python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

def extract_video_content(url):
    """Extract content from video-based news pages (Bilibili, Douyin)"""
    options = webdriver.ChromeOptions()
    options.add_argument('--headless')
    options.add_argument('--no-sandbox')
    
    driver = webdriver.Chrome(options=options)
    driver.get(url)
    
    try:
        # Wait for video description to load
        wait = WebDriverWait(driver, 10)
        
        # Bilibili example
        if 'bilibili.com' in url:
            desc = wait.until(
                EC.presence_of_element_located((By.CLASS_NAME, "desc-info-text"))
            ).text
            
            # Extract comments
            comments = driver.find_elements(By.CLASS_NAME, "reply-content")
            comment_text = "\n".join([c.text for c in comments[:10]])
            
            content = f"视频描述: {desc}\n\n热门评论:\n{comment_text}"
        
        return content
    finally:
        driver.quit()
```

### Multi-Channel Push Notifications

```python
import requests
import json
import smtplib
from email.mime.text import MIMEText

def push_to_wechat(webhook_url, content):
    """Push to Enterprise WeChat bot"""
    data = {
        "msgtype": "markdown",
        "markdown": {
            "content": content
        }
    }
    response = requests.post(webhook_url, json=data)
    return response.json()

def push_to_telegram(bot_token, chat_id, message):
    """Push to Telegram"""
    url = f"https://api.telegram.org/bot{bot_token}/sendMessage"
    data = {
        "chat_id": chat_id,
        "text": message,
        "parse_mode": "Markdown"
    }
    response = requests.post(url, json=data)
    return response.json()

def push_to_email(smtp_server, smtp_port, username, password, to_email, subject, body):
    """Push via email"""
    msg = MIMEText(body, 'html', 'utf-8')
    msg['Subject'] = subject
    msg['From'] = username
    msg['To'] = to_email
    
    with smtplib.SMTP(smtp_server, smtp_port) as server:
        server.starttls()
        server.login(username, password)
        server.send_message(msg)

# Example: Create alert report
def create_alert_report(topic, articles):
    """Generate formatted alert report"""
    report = f"# 热点话题分析: {topic}\n\n"
    report += f"**时间**: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}\n\n"
    
    # Perform clustering
    cluster_result = cluster_topics(articles)
    report += f"## 话题聚类\n{cluster_result}\n\n"
    
    # Add top articles
    report += "## 热门内容\n\n"
    for i, article in enumerate(articles[:5], 1):
        sentiment = analyze_sentiment(article.get('content', article['title']))
        report += f"{i}. **{article['title']}**\n"
        report += f"   - 平台: {article['platform']} | 热度: {article['heat_value']}\n"
        report += f"   - 情感: {sentiment}\n"
        report += f"   - [查看原文]({article['url']})\n\n"
    
    return report

# Push alert to all channels
topic = "人工智能"
articles = get_recent_searches('weibo', 24)
report = create_alert_report(topic, articles)

push_to_wechat(os.getenv('WECHAT_WEBHOOK_URL'), report)
push_to_telegram(os.getenv('TELEGRAM_BOT_TOKEN'), os.getenv('TELEGRAM_CHAT_ID'), report)
push_to_email(
    os.getenv('SMTP_SERVER'),
    int(os.getenv('SMTP_PORT')),
    os.getenv('SMTP_USERNAME'),
    os.getenv('SMTP_PASSWORD'),
    os.getenv('EMAIL_TO'),
    f"热点话题: {topic}",
    report
)
```

## Common Patterns

### Scheduled Monitoring Task

```python
import schedule
import time

def monitor_and_alert(keywords, platforms=['weibo', 'douyin', 'bilibili']):
    """Monitor specific keywords and send alerts"""
    for platform in platforms:
        articles = get_recent_searches(platform, 1)  # Last hour
        
        # Filter by keywords
        relevant = [a for a in articles if any(kw in a['title'] for kw in keywords)]
        
        if relevant:
            report = create_alert_report(f"关键词: {', '.join(keywords)}", relevant)
            push_to_wechat(os.getenv('WECHAT_WEBHOOK_URL'), report)

# Schedule monitoring every hour
schedule.every(1).hours.do(
    monitor_and_alert, 
    keywords=['人工智能', 'AI', 'DeepSeek']
)

while True:
    schedule.run_pending()
    time.sleep(60)
```

### Daily Summary Report

```python
def generate_daily_summary():
    """Generate daily hot topic summary"""
    platforms = ['weibo', 'douyin', 'bilibili', 'zhihu']
    all_articles = []
    
    for platform in platforms:
        articles = get_recent_searches(platform, 24)
        all_articles.extend(articles)
    
    # Sort by heat value
    top_articles = sorted(all_articles, key=lambda x: x['heat_value'], reverse=True)[:20]
    
    # Generate report
    report = create_alert_report("今日热点汇总", top_articles)
    
    # Push to all channels
    push_to_email(
        os.getenv('SMTP_SERVER'),
        int(os.getenv('SMTP_PORT')),
        os.getenv('SMTP_USERNAME'),
        os.getenv('SMTP_PASSWORD'),
        os.getenv('EMAIL_TO'),
        "舆情日报",
        report
    )

# Schedule daily at 8 AM
schedule.every().day.at("08:00").do(generate_daily_summary)
```

## Troubleshooting

### Browser Driver Issues

**Error**: `selenium.common.exceptions.WebDriverException: 'chromedriver' executable needs to be in PATH`

**Solution**:
```bash
# Verify driver is executable and in PATH
which chromedriver  # Linux/macOS
where chromedriver  # Windows

# If not found, add to PATH or specify path in code
from selenium.webdriver.chrome.service import Service
service = Service('/path/to/chromedriver')
driver = webdriver.Chrome(service=service)
```

### Database Connection Errors

**Error**: `pymysql.err.OperationalError: (2003, "Can't connect to MySQL server")`

**Solution**:
```bash
# Check MySQL is running
sudo systemctl status mysql  # Linux
brew services list | grep mysql  # macOS

# Verify credentials in .env
mysql -h localhost -u your_user -p
```

### LLM API Rate Limits

**Error**: `openai.error.RateLimitError: Rate limit exceeded`

**Solution**:
```python
import time
from tenacity import retry, wait_exponential, stop_after_attempt

@retry(wait=wait_exponential(multiplier=1, min=4, max=60), stop=stop_after_attempt(5))
def analyze_with_retry(text):
    return analyze_sentiment(text)
```

### Crawler Blocked by Platform

**Solution**: Add delays and user agents:
```python
# In hotsearchcrawler/settings.py
DOWNLOAD_DELAY = 2
RANDOMIZE_DOWNLOAD_DELAY = True
USER_AGENT = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'
COOKIES_ENABLED = True
```

### Missing Content from Video Pages

**Solution**: Increase wait time for dynamic content:
```python
wait = WebDriverWait(driver, 20)  # Increase from 10 to 20 seconds
driver.execute_script("window.scrollTo(0, document.body.scrollHeight);")  # Scroll to load
time.sleep(3)  # Additional wait
```
