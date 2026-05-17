---
name: llm-public-opinion-analytics-assistant
description: Intelligent public opinion analytics platform aggregating 26 hot lists from 15 platforms with LLM analysis for sentiment, clustering, and multi-channel alerts
triggers:
  - "set up public opinion monitoring system"
  - "analyze hot topics across multiple platforms"
  - "configure sentiment analysis for social media trends"
  - "create hot topic crawler with LLM analytics"
  - "implement multi-platform hot search aggregation"
  - "build opinion analytics with push notifications"
  - "deploy Chinese social media monitoring system"
  - "integrate Pangu model for sentiment analysis"
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is an intelligent public opinion analytics platform that aggregates real-time data from **26 hot lists** across **15 mainstream Chinese platforms** (Weibo, Bilibili, Douyin, Zhihu, etc.) and combines it with Large Language Model analysis capabilities. It provides conversational hot topic queries, theme-based searching, topic clustering, sentiment analysis, and multi-channel push notifications (WeChat Work, Telegram, Email).

## Core Capabilities

- **Multi-platform data crawling**: 15 platforms including Weibo, Bilibili, Douyin, Baidu, Zhihu, etc.
- **LLM-powered analysis**: Topic clustering, sentiment analysis, trend detection using Pangu or OpenAI-compatible models
- **Conversational interface**: Natural language queries for hot topics and analytics
- **Content extraction**: Deep parsing of news details including video content
- **Multi-channel alerts**: Push reports via WeChat Work, Telegram, Email (SMTP)
- **Hotkey controls**: Start/stop crawlers via keyboard shortcuts

## Installation

### Prerequisites

```bash
# Python 3.8+
# MySQL 5.7+
# Chrome/Edge browser + matching WebDriver
```

### Browser Driver Setup

1. **Check browser version**: Settings → About (e.g., Chrome 115.0.5790.102)

2. **Download matching driver**:
   - Chrome: https://chromedriver.chromium.org/
   - Edge: https://developer.microsoft.com/microsoft-edge/tools/webdriver/

3. **Add driver to PATH**:
   ```bash
   # Linux/macOS
   sudo mv chromedriver /usr/local/bin/
   chmod +x /usr/local/bin/chromedriver
   
   # Windows: Add driver folder to System PATH environment variable
   ```

4. **Verify**:
   ```bash
   chromedriver --version
   ```

### Install Dependencies

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install requirements
pip install -r requirements.txt
```

### Database Setup

```bash
# Create MySQL database
mysql -u root -p

CREATE DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE hotsearch;
```

Reference `init.py` for table schemas:

```python
# Example from init.py (adapt as needed)
import pymysql

conn = pymysql.connect(
    host='localhost',
    user='root',
    password='your_password',
    database='hotsearch',
    charset='utf8mb4'
)

cursor = conn.cursor()

# Create hot_topics table
cursor.execute("""
CREATE TABLE IF NOT EXISTS hot_topics (
    id INT AUTO_INCREMENT PRIMARY KEY,
    platform VARCHAR(50) NOT NULL,
    title VARCHAR(500) NOT NULL,
    url VARCHAR(1000),
    rank INT,
    heat_value VARCHAR(100),
    category VARCHAR(100),
    fetch_time DATETIME,
    content TEXT,
    INDEX idx_platform (platform),
    INDEX idx_fetch_time (fetch_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
""")

# Create analysis_results table
cursor.execute("""
CREATE TABLE IF NOT EXISTS analysis_results (
    id INT AUTO_INCREMENT PRIMARY KEY,
    query VARCHAR(500),
    result_type VARCHAR(50),
    content TEXT,
    created_at DATETIME,
    INDEX idx_query (query),
    INDEX idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
""")

conn.commit()
cursor.close()
conn.close()
```

## Configuration

### Crawler Configuration

Edit `hotsearchcrawler/settings.py`:

```python
# MySQL connection
MYSQL_CONFIG = {
    'host': 'localhost',
    'port': 3306,
    'user': 'root',
    'password': 'your_password',
    'database': 'hotsearch',
    'charset': 'utf8mb4'
}

# Optional: Platform-specific cookies for authenticated access
COOKIES = {
    'weibo': 'your_weibo_cookies_string',
    'bilibili': 'your_bilibili_cookies_string'
}

# Browser settings
SELENIUM_DRIVER_NAME = 'chrome'  # or 'edge'
SELENIUM_DRIVER_EXECUTABLE_PATH = '/usr/local/bin/chromedriver'
```

### Analysis System Configuration

Create `.env` file in project root:

```bash
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_BASE=https://your-llm-endpoint/v1
OPENAI_API_KEY=your_api_key_here
OPENAI_MODEL=pangu-embedded-7b  # or gpt-3.5-turbo, etc.

# Or use Huawei Pangu directly
PANGU_API_URL=http://localhost:8000/v1/chat/completions
PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b

# Push Notification Configuration
# WeChat Work Robot
WECHAT_WORK_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# WeChat Work Application
WECHAT_WORK_CORPID=your_corp_id
WECHAT_WORK_CORPSECRET=your_corp_secret
WECHAT_WORK_AGENTID=your_agent_id

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
EMAIL_FROM=your_email@gmail.com
EMAIL_TO=recipient@example.com
```

## Running the System

### Start Analysis System (Web Interface)

```bash
python app.py
```

Access at `http://localhost:5000`

### Start Crawlers

**Option 1: Via Web Interface**
- Click "Start Crawler" button in the web UI
- Use keyboard shortcuts to control (configured in settings)

**Option 2: Command Line**

```bash
# Test specific crawler
python runspider-test.py --spider weibo

# Run all crawlers
python run_spiders.py
```

**Option 3: Scheduled Crawling**

```python
# Add to crontab for periodic execution
# Crawl every hour
0 * * * * cd /path/to/project && /path/to/venv/bin/python run_spiders.py
```

## API Usage Patterns

### Query Hot Topics

```python
from hotsearch_analysis_agent.query_engine import QueryEngine

engine = QueryEngine()

# Get hot topics from specific platform
weibo_hot = engine.query_platform("weibo", limit=20)
print(weibo_hot)

# Search by keyword
ai_topics = engine.search_topics("人工智能", platforms=["weibo", "zhihu"])
print(ai_topics)

# Time-range query
from datetime import datetime, timedelta
yesterday = datetime.now() - timedelta(days=1)
recent_topics = engine.query_by_time(
    start_time=yesterday,
    platforms=["bilibili", "douyin"]
)
```

### LLM Analysis

```python
from hotsearch_analysis_agent.llm_analyzer import LLMAnalyzer

analyzer = LLMAnalyzer()

# Sentiment analysis
topics = ["话题1", "话题2", "话题3"]
sentiment_result = analyzer.analyze_sentiment(topics)
print(sentiment_result)

# Topic clustering
all_topics = engine.query_platform("weibo", limit=100)
clusters = analyzer.cluster_topics(all_topics)
print(clusters)

# Generate analytical report
report = analyzer.generate_report(
    query="人工智能与前沿科技",
    topics=ai_topics,
    include_sentiment=True,
    include_clustering=True
)
print(report)
```

### Content Extraction

```python
from hotsearch_analysis_agent.content_extractor import ContentExtractor

extractor = ContentExtractor()

# Extract article content
content = extractor.extract_from_url(
    "https://www.example.com/news/article123"
)
print(content['title'])
print(content['text'])

# Extract video metadata (Bilibili, Douyin, etc.)
video_info = extractor.extract_video_info(
    "https://www.bilibili.com/video/BV1234567890"
)
print(video_info['title'])
print(video_info['description'])
print(video_info['tags'])
```

### Push Notifications

```python
from hotsearch_analysis_agent.push_service import PushService

pusher = PushService()

# Send to WeChat Work
pusher.send_wechat_work(
    title="热点分析报告",
    content=report,
    mentioned_list=["@all"]
)

# Send to Telegram
pusher.send_telegram(
    message=f"📊 {report}",
    parse_mode="Markdown"
)

# Send email
pusher.send_email(
    subject="舆情分析 - 人工智能热点",
    body=report,
    html=True
)
```

### Scheduled Push Tasks

```python
from hotsearch_analysis_agent.task_scheduler import TaskScheduler

scheduler = TaskScheduler()

# Create daily report task
task = scheduler.create_task(
    name="AI热点日报",
    query="人工智能",
    schedule="0 18 * * *",  # Daily at 6 PM
    channels=["wechat_work", "email"],
    analysis_types=["sentiment", "clustering"]
)

# Test push task
from test_push_task import test_push
test_push(
    query="人工智能",
    channels=["wechat_work"],
    dry_run=False
)
```

## Common Patterns

### Building a Custom Monitoring Dashboard

```python
from flask import Flask, jsonify, request
from hotsearch_analysis_agent.query_engine import QueryEngine
from hotsearch_analysis_agent.llm_analyzer import LLMAnalyzer

app = Flask(__name__)
query_engine = QueryEngine()
analyzer = LLMAnalyzer()

@app.route('/api/hot-topics')
def get_hot_topics():
    platform = request.args.get('platform', 'weibo')
    limit = int(request.args.get('limit', 20))
    
    topics = query_engine.query_platform(platform, limit=limit)
    return jsonify(topics)

@app.route('/api/analyze', methods=['POST'])
def analyze_topics():
    data = request.json
    query = data.get('query')
    
    # Fetch relevant topics
    topics = query_engine.search_topics(query)
    
    # Perform analysis
    sentiment = analyzer.analyze_sentiment(topics)
    clusters = analyzer.cluster_topics(topics)
    report = analyzer.generate_report(
        query=query,
        topics=topics,
        sentiment=sentiment,
        clusters=clusters
    )
    
    return jsonify({
        'sentiment': sentiment,
        'clusters': clusters,
        'report': report
    })

if __name__ == '__main__':
    app.run(debug=True, port=5001)
```

### Automated Alert System

```python
import schedule
import time
from hotsearch_analysis_agent.query_engine import QueryEngine
from hotsearch_analysis_agent.llm_analyzer import LLMAnalyzer
from hotsearch_analysis_agent.push_service import PushService

def monitor_keywords():
    """Monitor specific keywords and alert on threshold breach"""
    keywords = ["产品召回", "安全事故", "负面舆情"]
    threshold = 5  # Alert if keyword appears in top 5
    
    engine = QueryEngine()
    pusher = PushService()
    
    for keyword in keywords:
        topics = engine.search_topics(keyword, limit=50)
        
        # Check if any topic ranks in top 5
        for topic in topics:
            if topic['rank'] <= threshold:
                alert_message = f"""
                ⚠️ 关键词预警
                
                关键词: {keyword}
                话题: {topic['title']}
                平台: {topic['platform']}
                排名: #{topic['rank']}
                热度: {topic['heat_value']}
                链接: {topic['url']}
                """
                
                pusher.send_wechat_work(
                    title="舆情预警",
                    content=alert_message
                )
                break

# Schedule monitoring every 30 minutes
schedule.every(30).minutes.do(monitor_keywords)

while True:
    schedule.run_pending()
    time.sleep(60)
```

### Multi-Platform Comparison

```python
from collections import defaultdict
from hotsearch_analysis_agent.query_engine import QueryEngine

def compare_platforms(query, platforms=None):
    """Compare how a topic performs across platforms"""
    if platforms is None:
        platforms = ["weibo", "zhihu", "bilibili", "douyin"]
    
    engine = QueryEngine()
    results = defaultdict(list)
    
    for platform in platforms:
        topics = engine.search_topics(query, platforms=[platform])
        results[platform] = topics
    
    # Generate comparison report
    print(f"\n{'='*60}")
    print(f"话题对比分析: {query}")
    print(f"{'='*60}\n")
    
    for platform, topics in results.items():
        print(f"\n{platform.upper()}:")
        print(f"  相关话题数: {len(topics)}")
        
        if topics:
            top_topic = topics[0]
            print(f"  最热话题: {top_topic['title']}")
            print(f"  排名: #{top_topic['rank']}")
            print(f"  热度: {top_topic['heat_value']}")
    
    return results

# Usage
compare_platforms("人工智能")
```

## Troubleshooting

### Crawler Issues

**Problem**: WebDriver not found
```bash
# Solution: Verify driver path
which chromedriver  # Linux/macOS
where chromedriver  # Windows

# Reinstall driver
wget https://chromedriver.storage.googleapis.com/LATEST_RELEASE
# Download matching version
```

**Problem**: Cookies expired / Access denied
```python
# Solution: Update cookies in settings.py
# 1. Login to platform in browser
# 2. Open DevTools → Application → Cookies
# 3. Copy cookie string
# 4. Update COOKIES dict in settings.py

COOKIES = {
    'weibo': 'SUB=your_new_cookie_value; ...'
}
```

**Problem**: Crawler timeout
```python
# Adjust timeout in spider settings
# hotsearchcrawler/spiders/base_spider.py

SELENIUM_TIMEOUT = 30  # Increase timeout
RETRY_TIMES = 3
DOWNLOAD_DELAY = 2  # Add delay between requests
```

### Database Issues

**Problem**: Connection refused
```bash
# Check MySQL status
sudo systemctl status mysql

# Test connection
mysql -u root -p -h localhost
```

**Problem**: Character encoding errors
```sql
-- Ensure database uses utf8mb4
ALTER DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
ALTER TABLE hot_topics CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### LLM Analysis Issues

**Problem**: API connection timeout
```python
# Increase timeout in .env or code
import os
os.environ['OPENAI_API_TIMEOUT'] = '120'  # seconds

# Or use retry logic
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=4, max=10))
def call_llm(prompt):
    return analyzer.analyze(prompt)
```

**Problem**: Poor analysis quality
```python
# Improve prompts with more context
prompt = f"""
你是一个专业的舆情分析师。请分析以下话题的情感倾向:

话题列表:
{topics}

要求:
1. 识别整体情感倾向(正面/负面/中性)
2. 提供具体证据支撑判断
3. 分析潜在风险点
4. 给出应对建议

输出格式: JSON
"""

result = analyzer.analyze_with_prompt(prompt)
```

### Push Notification Issues

**Problem**: WeChat Work webhook returns error 93000
```python
# Solution: Check webhook URL format
# Correct format: https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Verify message format
import json
payload = {
    "msgtype": "text",
    "text": {
        "content": "Test message"
    }
}
print(json.dumps(payload, ensure_ascii=False))
```

**Problem**: Email not sending
```python
# Enable Gmail "Less secure app access" or use App Password
# For Gmail: Use port 587 with STARTTLS

import smtplib
from email.mime.text import MIMEText

# Test connection
try:
    server = smtplib.SMTP('smtp.gmail.com', 587)
    server.starttls()
    server.login('your_email@gmail.com', 'your_app_password')
    print("SMTP connection successful")
    server.quit()
except Exception as e:
    print(f"Error: {e}")
```

## Performance Optimization

```python
# Use connection pooling for database
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

engine = create_engine(
    'mysql+pymysql://user:pass@localhost/hotsearch',
    poolclass=QueuePool,
    pool_size=10,
    max_overflow=20
)

# Cache LLM results
from functools import lru_cache

@lru_cache(maxsize=100)
def cached_analysis(query):
    return analyzer.analyze(query)

# Async crawling
import asyncio
from concurrent.futures import ThreadPoolExecutor

async def crawl_all_platforms():
    platforms = ['weibo', 'zhihu', 'bilibili', 'douyin']
    with ThreadPoolExecutor(max_workers=4) as executor:
        loop = asyncio.get_event_loop()
        tasks = [
            loop.run_in_executor(executor, crawl_platform, platform)
            for platform in platforms
        ]
        results = await asyncio.gather(*tasks)
    return results
```
