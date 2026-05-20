---
name: llm-public-opinion-analytics-assistant
description: A comprehensive public opinion analytics assistant that crawls 26 trending lists from 15 platforms and uses LLMs to analyze sentiment, cluster topics, and deliver multi-channel alerts
triggers:
  - set up public opinion monitoring system
  - configure trending topic crawler and analyzer
  - analyze social media sentiment with LLM
  - create multi-platform hot topic tracker
  - build sentiment analysis and clustering pipeline
  - deploy real-time trending news analytics
  - set up automated opinion report pushing
  - configure Chinese social media monitoring
---

# LLM Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive system that combines real-time data from 26 trending lists across 15 mainstream Chinese platforms with LLM analysis capabilities. The system enables conversational hot search queries, topic clustering, sentiment analysis, and multi-channel push notifications (WeChat, Email, Telegram).

## What This Project Does

- **Multi-Platform Crawling**: Collects trending topics from 15 platforms including Weibo, Bilibili, Baidu, Toutiao, etc.
- **LLM-Powered Analysis**: Uses models like Huawei PanGu for sentiment analysis, topic clustering, and summarization
- **Conversational Interface**: Natural language queries for trending topics and analysis
- **Content Extraction**: Extracts full content even from video news sources
- **Multi-Channel Notifications**: Supports WeChat Work, Telegram, and Email push notifications
- **Hotkey Control**: Quick crawler start/stop via keyboard shortcuts

## Installation

### Prerequisites

**1. Browser Driver Setup**

Download and configure browser driver (Chrome or Edge):

```bash
# Verify Chrome/Edge version first
google-chrome --version  # or microsoft-edge --version

# Download matching ChromeDriver from https://chromedriver.chromium.org/
# or EdgeDriver from https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Add to PATH (Linux/macOS)
sudo mv chromedriver /usr/local/bin/
sudo chmod +x /usr/local/bin/chromedriver

# Verify installation
chromedriver --version
```

**2. MySQL Database**

```bash
# Install MySQL
sudo apt-get install mysql-server  # Ubuntu/Debian
# or
brew install mysql  # macOS

# Start MySQL service
sudo systemctl start mysql

# Create database
mysql -u root -p
CREATE DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

**3. Python Environment**

```bash
# Clone repository
git clone https://github.com/hmmnxkl/LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant.git
cd LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant

# Create virtual environment
python -m venv venv
source venv/bin/activate  # or venv\Scripts\activate on Windows

# Install dependencies
pip install -r requirements.txt
```

### Database Initialization

Reference the `init.py` file to create required tables:

```python
# Example database schema initialization
import pymysql

conn = pymysql.connect(
    host='localhost',
    user='your_user',
    password='your_password',
    database='hotsearch',
    charset='utf8mb4'
)

cursor = conn.cursor()

# Create trending topics table
cursor.execute("""
CREATE TABLE IF NOT EXISTS trending_topics (
    id INT AUTO_INCREMENT PRIMARY KEY,
    platform VARCHAR(50),
    title VARCHAR(500),
    url VARCHAR(1000),
    rank INT,
    heat_score VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_platform (platform),
    INDEX idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
""")

# Create analysis results table
cursor.execute("""
CREATE TABLE IF NOT EXISTS analysis_results (
    id INT AUTO_INCREMENT PRIMARY KEY,
    query TEXT,
    result LONGTEXT,
    sentiment VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
""")

conn.commit()
conn.close()
```

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```bash
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_username
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch

# LLM Configuration (OpenAI-compatible format)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://your-llm-endpoint.com/v1
MODEL_NAME=pangu-7b  # or other model

# Push Notification Configuration
# WeChat Work Robot
WECHAT_WEBHOOK_URL=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=your_key

# WeChat Work App
WECHAT_CORP_ID=your_corp_id
WECHAT_CORP_SECRET=your_corp_secret
WECHAT_AGENT_ID=your_agent_id

# Telegram
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
EMAIL_RECIPIENTS=recipient1@example.com,recipient2@example.com
```

### Crawler Settings

Edit `hotsearchcrawler/settings.py`:

```python
# MySQL Connection
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_user'
MYSQL_PASSWORD = 'your_password'
MYSQL_DATABASE = 'hotsearch'

# Optional: Platform-specific cookies
COOKIES = {
    'weibo': 'your_weibo_cookie',
    'bilibili': 'your_bilibili_cookie'
}

# Crawler settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
ROBOTSTXT_OBEY = False
```

## Usage

### Starting the Analysis System

```python
# app.py - Main application entry point
from hotsearch_analysis_agent import AnalysisAgent
from flask import Flask, render_template, request, jsonify

app = Flask(__name__)
agent = AnalysisAgent()

@app.route('/query', methods=['POST'])
def query_analysis():
    user_query = request.json.get('query')
    result = agent.analyze(user_query)
    return jsonify(result)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

Run the web interface:

```bash
python app.py
```

### Running the Crawler

**Manual Start:**

```python
# run_spiders.py
from hotsearchcrawler.spider_manager import SpiderManager

manager = SpiderManager()

# Start all spiders
manager.start_all()

# Start specific platform
manager.start_spider('weibo')
manager.start_spider('bilibili')

# Stop all spiders
manager.stop_all()
```

**Command Line:**

```bash
# Run specific spider
cd hotsearchcrawler
scrapy crawl weibo

# Run all spiders
python run_spiders.py
```

**Via Web Interface:**

Use keyboard shortcuts in the web UI to start/stop crawlers.

### LLM Analysis Examples

**Sentiment Analysis:**

```python
from hotsearch_analysis_agent.llm_analyzer import LLMAnalyzer

analyzer = LLMAnalyzer(
    api_key=os.getenv('OPENAI_API_KEY'),
    api_base=os.getenv('OPENAI_API_BASE'),
    model=os.getenv('MODEL_NAME')
)

# Analyze sentiment
text = "今天的AI技术发展真是太令人兴奋了!"
sentiment = analyzer.analyze_sentiment(text)
print(sentiment)  # Output: {"sentiment": "positive", "score": 0.92}
```

**Topic Clustering:**

```python
# Cluster related trending topics
topics = [
    "GPT-6提前曝光 200万超长上下文",
    "DeepSeek V4采用华为算力",
    "中国大模型周调用量超越美国"
]

clusters = analyzer.cluster_topics(topics)
print(clusters)
# Output: {
#   "cluster_1": {
#     "theme": "大模型技术进展",
#     "topics": [...],
#     "summary": "..."
#   }
# }
```

**Conversational Query:**

```python
from hotsearch_analysis_agent.agent import HotSearchAgent

agent = HotSearchAgent()

# Natural language query
response = agent.query("最近关于人工智能的热点有哪些?")
print(response)

# With context awareness
response = agent.query("分析一下这些话题的情感倾向")
print(response)
```

### Push Notification Setup

**Test Push Configuration:**

```python
# test_push_task.py
from hotsearch_analysis_agent.push_manager import PushManager

push_mgr = PushManager()

# Test WeChat Work Robot
push_mgr.test_wechat_robot(
    webhook_url=os.getenv('WECHAT_WEBHOOK_URL'),
    message="测试推送消息"
)

# Test Telegram
push_mgr.test_telegram(
    bot_token=os.getenv('TELEGRAM_BOT_TOKEN'),
    chat_id=os.getenv('TELEGRAM_CHAT_ID'),
    message="Test push notification"
)

# Test Email
push_mgr.test_email(
    smtp_config={
        'host': os.getenv('SMTP_HOST'),
        'port': int(os.getenv('SMTP_PORT')),
        'user': os.getenv('SMTP_USER'),
        'password': os.getenv('SMTP_PASSWORD')
    },
    recipients=os.getenv('EMAIL_RECIPIENTS').split(','),
    subject="测试邮件",
    content="测试推送内容"
)
```

**Create Scheduled Push Task:**

```python
from hotsearch_analysis_agent.scheduler import TaskScheduler

scheduler = TaskScheduler()

# Schedule daily AI trend report
scheduler.add_task(
    name="ai_trends_daily",
    query="人工智能与前沿科技的热点",
    schedule="0 12 * * *",  # Daily at 12:00
    channels=['wechat', 'email', 'telegram'],
    analysis_type='comprehensive'  # or 'sentiment', 'clustering'
)

scheduler.start()
```

## Common Patterns

### Custom Spider for New Platform

```python
# hotsearchcrawler/spiders/custom_platform.py
import scrapy
from hotsearchcrawler.items import TrendingItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://custom-platform.com/trending']
    
    def parse(self, response):
        for item in response.css('.trending-item'):
            trending = TrendingItem()
            trending['platform'] = 'custom_platform'
            trending['title'] = item.css('.title::text').get()
            trending['url'] = item.css('a::attr(href)').get()
            trending['rank'] = item.css('.rank::text').get()
            trending['heat_score'] = item.css('.heat::text').get()
            yield trending
```

### Custom Analysis Pipeline

```python
from hotsearch_analysis_agent.base_analyzer import BaseAnalyzer

class CustomAnalyzer(BaseAnalyzer):
    def analyze(self, data):
        # Preprocess data
        cleaned_data = self.preprocess(data)
        
        # Call LLM
        prompt = f"分析以下舆情数据:\n{cleaned_data}"
        response = self.llm_client.chat(prompt)
        
        # Post-process results
        result = self.parse_response(response)
        
        # Store to database
        self.save_results(result)
        
        return result
    
    def preprocess(self, data):
        # Custom preprocessing logic
        return data
    
    def parse_response(self, response):
        # Parse LLM output
        return {
            'summary': response.get('summary'),
            'sentiment': response.get('sentiment'),
            'key_topics': response.get('topics')
        }
```

### Real-time Monitoring Dashboard

```python
# Real-time trending topic monitor
from hotsearch_analysis_agent.monitor import TrendingMonitor

monitor = TrendingMonitor()

# Monitor specific keywords
monitor.add_keywords(['人工智能', 'AI', '大模型'])

# Set alert threshold
monitor.set_threshold(heat_score=10000)

# Callback on alert
def on_alert(topic):
    print(f"热点预警: {topic['title']} (热度: {topic['heat_score']})")
    # Trigger immediate analysis and push
    agent.analyze_and_push(topic)

monitor.on_alert(on_alert)
monitor.start()
```

## Troubleshooting

**Browser Driver Issues:**

```python
# Verify driver path
import shutil
driver_path = shutil.which('chromedriver')
print(f"Driver location: {driver_path}")

# If not found, set explicitly
from selenium import webdriver
from selenium.webdriver.chrome.service import Service

service = Service('/path/to/chromedriver')
driver = webdriver.Chrome(service=service)
```

**MySQL Connection Errors:**

```python
# Test connection
import pymysql

try:
    conn = pymysql.connect(
        host=os.getenv('MYSQL_HOST'),
        port=int(os.getenv('MYSQL_PORT')),
        user=os.getenv('MYSQL_USER'),
        password=os.getenv('MYSQL_PASSWORD'),
        database=os.getenv('MYSQL_DATABASE')
    )
    print("Database connection successful")
    conn.close()
except Exception as e:
    print(f"Connection failed: {e}")
```

**LLM API Timeout:**

```python
# Increase timeout and add retry logic
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(min=1, max=10))
def call_llm_with_retry(prompt):
    response = analyzer.llm_client.chat(
        prompt,
        timeout=60  # Increase timeout
    )
    return response
```

**Cookie Expiration:**

```python
# Refresh cookies for authenticated platforms
from hotsearchcrawler.cookie_manager import CookieManager

cookie_mgr = CookieManager()

# Update cookies
cookie_mgr.update_cookie('weibo', 'new_cookie_value')

# Auto-refresh on 401/403 errors
def handle_auth_error(response):
    if response.status in [401, 403]:
        cookie_mgr.refresh_cookie(response.meta['platform'])
        return response.request.replace(cookies=cookie_mgr.get_cookie())
```

**Encoding Issues with Chinese Text:**

```python
# Ensure proper UTF-8 handling
import sys
import io

# Set default encoding
sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')

# Database connection with charset
conn = pymysql.connect(
    charset='utf8mb4',
    use_unicode=True
)

# Scrapy settings
FEED_EXPORT_ENCODING = 'utf-8'
```

**Memory Issues with Large Datasets:**

```python
# Process data in batches
def process_large_dataset(query_date):
    batch_size = 1000
    offset = 0
    
    while True:
        batch = db.query(
            f"SELECT * FROM trending_topics WHERE date = %s LIMIT %s OFFSET %s",
            (query_date, batch_size, offset)
        )
        
        if not batch:
            break
            
        # Process batch
        analyzer.analyze_batch(batch)
        
        offset += batch_size
```

This skill provides comprehensive guidance for deploying and using the LLM-based public opinion analytics system, with focus on practical implementation for AI coding agents assisting developers.
