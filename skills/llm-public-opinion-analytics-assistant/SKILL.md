---
name: llm-public-opinion-analytics-assistant
description: A Chinese public opinion analytics system combining 26 real-time hot search lists from 15 platforms with LLM analysis for sentiment, clustering, and multi-channel alerting
triggers:
  - how do I set up the public opinion analytics assistant
  - analyze hot topics across Chinese social media platforms
  - configure crawler for Weibo, Bilibili, and Douyin hot searches
  - set up multi-channel notifications for trending topics
  - use LLM to analyze sentiment and cluster news topics
  - deploy hot search monitoring system with database
  - integrate Pangu model for Chinese text analysis
  - create scheduled reports for trending topics
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is a comprehensive Chinese public opinion monitoring system that aggregates real-time hot search data from 26 lists across 15 major platforms (Weibo, Bilibili, Douyin, Zhihu, Baidu, etc.) and combines it with LLM analysis capabilities. It provides conversational hot search queries, topic clustering, sentiment analysis, and multi-channel push notifications (WeChat Work, Telegram, Email).

**Key Features:**
- Multi-platform crawler system (Scrapy-based, independent from analysis system)
- LLM-powered analysis (supports Pangu model, OpenAI-compatible APIs)
- Natural language query interface with vector search
- Automated report generation with topic clustering
- Multi-channel alerting (WeChat Work Robot/App, Telegram, SMTP)
- Browser automation for extracting video/news detail content

## Installation

### Prerequisites

**1. Browser Driver Setup (Required for content extraction):**

For Chrome:
```bash
# Check Chrome version
google-chrome --version  # Linux
# or open chrome://settings/help

# Download matching ChromeDriver from https://chromedriver.chromium.org/
# Place in system PATH or project directory
export PATH=$PATH:/path/to/chromedriver
```

For Edge:
```bash
# Download EdgeDriver from https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/
# Verify installation
msedgedriver --version
```

**2. MySQL Database:**
```bash
# Install MySQL
sudo apt-get install mysql-server  # Ubuntu/Debian

# Create database and tables (refer to init.py)
mysql -u root -p
CREATE DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

**3. Python Environment:**
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

### Configuration

**1. Environment Variables (`.env`):**
```env
# Database Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch

# LLM API Configuration (OpenAI-compatible format)
OPENAI_API_KEY=your_api_key_here
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Or use Pangu model (local deployment)
# OPENAI_API_BASE=http://localhost:8000/v1
# MODEL_NAME=openpangu-embedded-7b

# Notification Channels
WECHAT_WORK_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY
TELEGRAM_BOT_TOKEN=your_telegram_bot_token
TELEGRAM_CHAT_ID=your_chat_id
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
```

**2. Crawler Configuration (`hotsearchcrawler/settings.py`):**
```python
# MySQL connection for crawler
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'root'
MYSQL_PASSWORD = 'your_password'
MYSQL_DB = 'hotsearch'

# Optional: Platform-specific cookies for authenticated access
COOKIES = {
    'weibo': 'your_weibo_cookie',
    'bilibili': 'your_bilibili_cookie',
}

# Crawler settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
```

**3. Database Initialization:**
```python
# Reference init.py structure
import mysql.connector

conn = mysql.connector.connect(
    host='localhost',
    user='root',
    password='your_password',
    database='hotsearch'
)

cursor = conn.cursor()

# Create hot search table
cursor.execute("""
CREATE TABLE IF NOT EXISTS hot_searches (
    id INT AUTO_INCREMENT PRIMARY KEY,
    platform VARCHAR(50),
    title VARCHAR(500),
    url VARCHAR(1000),
    hot_value VARCHAR(100),
    rank_position INT,
    crawl_time DATETIME,
    content TEXT,
    INDEX idx_platform (platform),
    INDEX idx_crawl_time (crawl_time)
)
""")

# Create analysis results table
cursor.execute("""
CREATE TABLE IF NOT EXISTS analysis_results (
    id INT AUTO_INCREMENT PRIMARY KEY,
    query VARCHAR(500),
    result_text TEXT,
    created_at DATETIME,
    INDEX idx_query (query)
)
""")

conn.commit()
```

## Core Commands & API Usage

### Starting the System

**1. Launch Main Application:**
```bash
# Start web interface and analysis system
python app.py
```

**2. Start Crawler System (via frontend or manually):**
```bash
# Manual crawler startup for testing
python run_spiders.py

# Test specific crawler
python runspider-test.py
```

**3. Test Push Notifications:**
```bash
# Test all configured notification channels
python test_push_task.py
```

### Analysis System API

**Query Hot Searches:**
```python
from hotsearch_analysis_agent import HotSearchAgent

# Initialize agent
agent = HotSearchAgent(
    api_key=os.getenv('OPENAI_API_KEY'),
    api_base=os.getenv('OPENAI_API_BASE'),
    model_name=os.getenv('MODEL_NAME')
)

# Natural language query
result = agent.query("最近关于人工智能的热点新闻有哪些？")
print(result)

# Get specific platform hot searches
weibo_results = agent.get_platform_data('weibo', limit=20)

# Topic clustering analysis
clusters = agent.cluster_topics(
    query="科技创新",
    num_clusters=3,
    days_back=7
)
```

**Sentiment Analysis:**
```python
# Analyze sentiment of specific topic
sentiment = agent.analyze_sentiment(
    topic="华为盘古大模型",
    platforms=['weibo', 'zhihu', 'bilibili']
)

# Output: {"positive": 0.65, "neutral": 0.25, "negative": 0.10}
```

**Generate Report:**
```python
# Create comprehensive analysis report
report = agent.generate_report(
    query="人工智能与前沿科技",
    include_clustering=True,
    include_sentiment=True,
    days_back=3
)

# report contains markdown-formatted analysis with:
# - Core findings
# - News details with links
# - Sentiment distribution
# - Topic clusters
```

### Crawler System API

**Run Specific Spiders:**
```python
from scrapy.crawler import CrawlerProcess
from hotsearchcrawler.spiders.weibo_spider import WeiboSpider

process = CrawlerProcess({
    'USER_AGENT': 'Mozilla/5.0...',
})

# Start Weibo hot search crawler
process.crawl(WeiboSpider)
process.start()
```

**Access Crawled Data:**
```python
import mysql.connector

conn = mysql.connector.connect(
    host=os.getenv('MYSQL_HOST'),
    user=os.getenv('MYSQL_USER'),
    password=os.getenv('MYSQL_PASSWORD'),
    database=os.getenv('MYSQL_DATABASE')
)

cursor = conn.cursor(dictionary=True)

# Get latest hot searches from all platforms
cursor.execute("""
    SELECT platform, title, url, hot_value, rank_position
    FROM hot_searches
    WHERE crawl_time >= DATE_SUB(NOW(), INTERVAL 1 HOUR)
    ORDER BY platform, rank_position
""")

results = cursor.fetchall()
```

### Push Notification API

**Configure Push Task:**
```python
from hotsearch_analysis_agent.push_service import PushService

push_service = PushService()

# Schedule automated report push
task = push_service.create_task(
    query="人工智能",
    channels=['wechat_work', 'telegram', 'email'],
    schedule="0 12 * * *",  # Daily at 12:00 PM
    recipients={
        'email': ['team@example.com'],
        'telegram': [os.getenv('TELEGRAM_CHAT_ID')]
    }
)

# Manual push
push_service.send_report(
    report_content=report,
    channels=['wechat_work'],
    title="AI热点分析报告"
)
```

**WeChat Work Robot:**
```python
import requests

webhook_url = os.getenv('WECHAT_WORK_WEBHOOK')

payload = {
    "msgtype": "markdown",
    "markdown": {
        "content": f"# 舆情分析报告\n\n{report_content}"
    }
}

response = requests.post(webhook_url, json=payload)
```

**Telegram Bot:**
```python
import telegram

bot = telegram.Bot(token=os.getenv('TELEGRAM_BOT_TOKEN'))

bot.send_message(
    chat_id=os.getenv('TELEGRAM_CHAT_ID'),
    text=report_content,
    parse_mode='Markdown'
)
```

## Common Patterns

### Pattern 1: Automated Daily Monitoring

```python
import schedule
import time
from hotsearch_analysis_agent import HotSearchAgent
from hotsearch_analysis_agent.push_service import PushService

agent = HotSearchAgent(
    api_key=os.getenv('OPENAI_API_KEY'),
    api_base=os.getenv('OPENAI_API_BASE')
)
push_service = PushService()

def daily_report():
    # Generate report
    report = agent.generate_report(
        query="关键词:科技,AI,政策",
        days_back=1
    )
    
    # Push to all channels
    push_service.send_report(
        report_content=report,
        channels=['wechat_work', 'email'],
        title="每日科技舆情报告"
    )

# Schedule daily at 9 AM
schedule.every().day.at("09:00").do(daily_report)

while True:
    schedule.run_pending()
    time.sleep(60)
```

### Pattern 2: Real-time Alert on Specific Keywords

```python
import time

MONITOR_KEYWORDS = ['突发', '危机', '紧急']
CHECK_INTERVAL = 300  # 5 minutes

def check_alerts():
    cursor.execute("""
        SELECT * FROM hot_searches
        WHERE crawl_time >= DATE_SUB(NOW(), INTERVAL 5 MINUTE)
    """)
    
    for row in cursor.fetchall():
        if any(kw in row['title'] for kw in MONITOR_KEYWORDS):
            # Analyze and send alert
            analysis = agent.analyze_sentiment(
                topic=row['title'],
                platforms=[row['platform']]
            )
            
            alert_msg = f"⚠️ 关键词触发警报\n\n" \
                       f"平台: {row['platform']}\n" \
                       f"标题: {row['title']}\n" \
                       f"热度: {row['hot_value']}\n" \
                       f"情感倾向: {analysis}"
            
            push_service.send_alert(
                message=alert_msg,
                channels=['telegram', 'wechat_work']
            )

while True:
    check_alerts()
    time.sleep(CHECK_INTERVAL)
```

### Pattern 3: Topic Clustering and Comparison

```python
# Compare topic trends across platforms
platforms = ['weibo', 'zhihu', 'bilibili', 'douyin']

topic_distribution = {}
for platform in platforms:
    results = agent.cluster_topics(
        query="新能源汽车",
        platform=platform,
        num_clusters=5,
        days_back=7
    )
    topic_distribution[platform] = results

# Generate comparative analysis
comparison_report = agent.compare_platforms(
    topic_distribution=topic_distribution,
    metric='sentiment_trend'
)
```

### Pattern 4: Vector Search for Similar Topics

```python
from hotsearch_analysis_agent.vector_search import VectorSearchEngine

# Initialize vector search
vector_engine = VectorSearchEngine(
    model_name='text-embedding-3-small',
    api_key=os.getenv('OPENAI_API_KEY')
)

# Find similar historical topics
query_embedding = vector_engine.embed_text("ChatGPT最新更新")

similar_topics = vector_engine.search(
    query_embedding=query_embedding,
    top_k=10,
    time_range_days=30
)

for topic in similar_topics:
    print(f"相似度: {topic['similarity']:.2f} - {topic['title']}")
```

## Configuration Best Practices

### Using Pangu Model (Local Deployment)

```bash
# Download Pangu model
# https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

# Start model service (example using vLLM)
python -m vllm.entrypoints.openai.api_server \
    --model /path/to/openpangu-embedded-7b-model \
    --host 0.0.0.0 \
    --port 8000

# Update .env
# OPENAI_API_BASE=http://localhost:8000/v1
# MODEL_NAME=openpangu-embedded-7b
```

### Crawler Rate Limiting

```python
# hotsearchcrawler/settings.py

# Respect platform rate limits
AUTOTHROTTLE_ENABLED = True
AUTOTHROTTLE_START_DELAY = 1
AUTOTHROTTLE_MAX_DELAY = 10
AUTOTHROTTLE_TARGET_CONCURRENCY = 2.0

# Retry configuration
RETRY_TIMES = 3
RETRY_HTTP_CODES = [500, 502, 503, 504, 408, 429]
```

### Database Optimization

```sql
-- Add indexes for common queries
CREATE INDEX idx_title_fulltext ON hot_searches(title(255));
CREATE INDEX idx_platform_time ON hot_searches(platform, crawl_time);

-- Partition by month for large datasets
ALTER TABLE hot_searches
PARTITION BY RANGE (YEAR(crawl_time) * 100 + MONTH(crawl_time)) (
    PARTITION p202601 VALUES LESS THAN (202602),
    PARTITION p202602 VALUES LESS THAN (202603)
);
```

## Troubleshooting

### Crawler Issues

**Problem: Selenium driver not found**
```bash
# Verify driver in PATH
which chromedriver  # Linux/macOS
where chromedriver  # Windows

# If not found, explicitly set path in code
from selenium import webdriver
from selenium.webdriver.chrome.service import Service

service = Service('/path/to/chromedriver')
driver = webdriver.Chrome(service=service)
```

**Problem: Platform blocking/CAPTCHA**
```python
# Add random delays
import random
time.sleep(random.uniform(2, 5))

# Rotate user agents
from scrapy.downloadermiddlewares.useragent import UserAgentMiddleware
# Configure in settings.py

# Use cookies for authenticated access
# Set in hotsearchcrawler/settings.py COOKIES dict
```

**Problem: Database connection pool exhausted**
```python
# Increase pool size in settings
MYSQL_POOL_SIZE = 10
MYSQL_MAX_OVERFLOW = 20

# Close connections properly
try:
    cursor.execute(query)
    conn.commit()
finally:
    cursor.close()
    conn.close()
```

### LLM Analysis Issues

**Problem: Rate limit exceeded**
```python
# Add retry with exponential backoff
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=4, max=10))
def call_llm_api(prompt):
    return agent.query(prompt)
```

**Problem: Token limit exceeded**
```python
# Truncate long inputs
def truncate_text(text, max_tokens=3000):
    # Simple character-based truncation (adjust ratio for Chinese)
    max_chars = max_tokens * 2
    return text[:max_chars] + "..." if len(text) > max_chars else text

# Use summarization for long content
summary = agent.summarize(long_content, max_length=500)
```

**Problem: Pangu model local deployment memory issues**
```bash
# Use quantization
python -m vllm.entrypoints.openai.api_server \
    --model /path/to/model \
    --quantization awq \
    --tensor-parallel-size 2  # Use multiple GPUs
```

### Push Notification Issues

**Problem: WeChat Work webhook 403**
```python
# Verify webhook URL is correct
# Check IP whitelist in WeChat Work admin panel
# Ensure message format is valid

# Test with minimal payload
test_payload = {
    "msgtype": "text",
    "text": {"content": "测试消息"}
}
response = requests.post(webhook_url, json=test_payload)
print(response.text)
```

**Problem: Email SMTP authentication failed**
```python
# For Gmail, use App Password instead of account password
# Enable 2FA and generate app password at:
# https://myaccount.google.com/apppasswords

import smtplib
from email.mime.text import MIMEText

try:
    server = smtplib.SMTP(os.getenv('SMTP_SERVER'), int(os.getenv('SMTP_PORT')))
    server.starttls()
    server.login(os.getenv('SMTP_USER'), os.getenv('SMTP_PASSWORD'))
    # Send email
except smtplib.SMTPAuthenticationError as e:
    print(f"Authentication failed: {e}")
```

### Vector Search Issues

**Problem: Poor search relevance**
```python
# Tune similarity threshold
results = vector_engine.search(
    query_embedding=embedding,
    top_k=20,
    similarity_threshold=0.7  # Increase for stricter matching
)

# Use hybrid search (vector + keyword)
results = vector_engine.hybrid_search(
    query_text="人工智能",
    keywords=["AI", "模型", "技术"],
    vector_weight=0.7,
    keyword_weight=0.3
)
```

**Problem: Embedding API timeout**
```python
# Batch embed with retry
def batch_embed(texts, batch_size=100):
    embeddings = []
    for i in range(0, len(texts), batch_size):
        batch = texts[i:i+batch_size]
        try:
            batch_embeddings = vector_engine.embed_batch(batch)
            embeddings.extend(batch_embeddings)
        except Exception as e:
            print(f"Batch {i} failed: {e}")
            time.sleep(5)
            # Retry logic
    return embeddings
```

## Advanced Usage

### Custom Spider Development

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
                'hot_value': item.css('.hot-value::text').get(),
                'rank_position': item.css('.rank::text').get(),
            }
```

### Custom Analysis Pipeline

```python
from hotsearch_analysis_agent import AnalysisPipeline

# Define custom analysis steps
pipeline = AnalysisPipeline([
    ('extract', agent.extract_keywords),
    ('cluster', agent.cluster_topics),
    ('sentiment', agent.analyze_sentiment),
    ('summarize', agent.generate_summary)
])

# Run pipeline on query results
results = agent.query("科技新闻")
analysis = pipeline.run(results)
```

This skill provides comprehensive coverage of the LLM-Based Intelligent Public Opinion Analytics Assistant for AI coding agents to effectively assist developers in deployment, configuration, and usage of the system.
