---
name: llm-public-opinion-analytics-assistant
description: Build and run intelligent public opinion analysis systems with real-time hot search data from 15+ platforms, LLM-powered sentiment analysis, and multi-channel alerts
triggers:
  - how do I set up the public opinion analytics assistant
  - analyze hot search trends across multiple platforms
  - configure sentiment analysis with LLM models
  - set up multi-channel push notifications for hot topics
  - crawl real-time data from Weibo Bilibili and other platforms
  - create topic clustering analysis with large language models
  - implement conversational hot search query interface
  - troubleshoot browser driver issues in hot search crawler
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What It Does

This project is an intelligent public opinion analysis assistant that combines real-time data from **26 hot search lists** across **15 mainstream platforms** (Weibo, Bilibili, Zhihu, Douyin, etc.) with large language model (LLM) analysis capabilities. It enables conversational hot search queries, topic clustering, sentiment analysis, and multi-channel push notifications (email, WeChat, Enterprise WeChat, Telegram).

**Key Features:**
- Real-time crawler cluster for 15+ Chinese social media/news platforms
- LLM-powered topic clustering and sentiment analysis
- Conversational interface for querying trends
- Video content analysis (extracts insights from video-based news)
- Multi-channel alert system with scheduled push tasks
- Keyboard shortcut control for crawler start/stop

## Installation

### Prerequisites

1. **Python 3.8+** with virtual environment
2. **MySQL 5.7+** database
3. **Browser Driver** (ChromeDriver or EdgeDriver)

### Step 1: Browser Driver Setup

```bash
# Check your Chrome/Edge version first
# Download matching driver from:
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Linux/macOS - place driver in PATH
sudo mv chromedriver /usr/local/bin/
sudo chmod +x /usr/local/bin/chromedriver

# Windows - add driver directory to System PATH
# Or place in project root
```

Verify installation:
```bash
chromedriver --version
```

### Step 2: Install Dependencies

```bash
# Clone the repository
git clone https://github.com/hmmnxkl/LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant.git
cd LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant

# Create virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install requirements
pip install -r requirements.txt
```

### Step 3: Database Setup

```bash
# Create MySQL database
mysql -u root -p -e "CREATE DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"

# Initialize tables (reference init.py for schema)
python init.py
```

### Step 4: Configuration

**Crawler Settings** (`hotsearchcrawler/settings.py`):
```python
# MySQL Configuration
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_mysql_user'
MYSQL_PASSWORD = 'your_mysql_password'
MYSQL_DB = 'hotsearch'

# Optional: Platform cookies for authenticated endpoints
COOKIES = {
    'weibo': 'your_weibo_cookies',
    'bilibili': 'your_bilibili_cookies'
}
```

**Analysis System** (`.env` file):
```bash
# MySQL
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_mysql_user
MYSQL_PASSWORD=your_mysql_password
MYSQL_DB=hotsearch

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://your-llm-endpoint.com/v1
MODEL_NAME=gpt-4  # or pangu-7b for local deployment

# Push Notification Channels
EMAIL_SMTP_HOST=smtp.gmail.com
EMAIL_SMTP_PORT=587
EMAIL_USERNAME=your_email@gmail.com
EMAIL_PASSWORD=your_app_password

WECHAT_WEBHOOK_URL=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=your_key
TELEGRAM_BOT_TOKEN=your_telegram_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

## Core Usage

### Starting the System

```bash
# Start the analysis web interface
python app.py

# Access at http://localhost:5000
```

### Running Crawlers

**Via Web Interface:**
- Use keyboard shortcuts in the web UI to start/stop crawlers

**Manual Testing:**
```bash
# Test specific platform crawler
python runspider-test.py weibo
python runspider-test.py bilibili

# Run all crawlers
python run_spiders.py
```

### Conversational Query Examples

Through the web interface, users can ask natural language questions:

```
"Show me today's Weibo hot search"
"What are people talking about regarding AI recently?"
"Analyze sentiment around topic X"
"Cluster related topics about technology"
```

### Programmatic API Usage

**Query Hot Search Data:**
```python
from hotsearch_analysis_agent.database import get_database_connection

# Connect to database
conn = get_database_connection()
cursor = conn.cursor()

# Query recent hot searches from specific platform
cursor.execute("""
    SELECT title, hot_value, url, platform, created_at
    FROM hot_searches
    WHERE platform = 'weibo' AND created_at >= DATE_SUB(NOW(), INTERVAL 1 DAY)
    ORDER BY hot_value DESC
    LIMIT 20
""")

results = cursor.fetchall()
for row in results:
    print(f"{row[0]} - Heat: {row[1]} - Platform: {row[3]}")

conn.close()
```

**LLM Analysis Integration:**
```python
import os
from openai import OpenAI

# Initialize LLM client (OpenAI-compatible)
client = OpenAI(
    api_key=os.getenv("OPENAI_API_KEY"),
    base_url=os.getenv("OPENAI_API_BASE")
)

def analyze_sentiment(text):
    """Analyze sentiment of hot search content"""
    response = client.chat.completions.create(
        model=os.getenv("MODEL_NAME", "gpt-4"),
        messages=[
            {"role": "system", "content": "你是一个舆情分析专家,擅长分析文本的情感倾向。"},
            {"role": "user", "content": f"分析以下内容的情感倾向(正面/负面/中性):\n\n{text}"}
        ],
        temperature=0.3
    )
    return response.choices[0].message.content

def cluster_topics(topics_list):
    """Cluster related topics using LLM"""
    topics_text = "\n".join([f"{i+1}. {t}" for i, t in enumerate(topics_list)])
    
    response = client.chat.completions.create(
        model=os.getenv("MODEL_NAME", "gpt-4"),
        messages=[
            {"role": "system", "content": "你是一个话题分析专家,擅长识别和聚类相关话题。"},
            {"role": "user", "content": f"请将以下话题进行聚类分析:\n\n{topics_text}"}
        ],
        temperature=0.5
    )
    return response.choices[0].message.content

# Example usage
hot_topics = ["GPT-6曝光", "AI发展", "华为芯片", "DeepSeek模型"]
clusters = cluster_topics(hot_topics)
print(clusters)
```

### Setting Up Push Notifications

**Test Push Configuration:**
```bash
python test_push_task.py
```

**Create Scheduled Push Task:**
```python
from hotsearch_analysis_agent.push_service import PushService
from datetime import datetime, timedelta

push_service = PushService()

# Create daily AI tech report
task_config = {
    "name": "AI Technology Daily Report",
    "query": "人工智能 OR AI OR 大模型",
    "platforms": ["weibo", "bilibili", "zhihu"],
    "channels": ["email", "wechat"],
    "schedule": "0 18 * * *",  # Daily at 6 PM
    "sentiment_filter": None,  # All sentiments
    "min_hot_value": 50000
}

# Register task
push_service.create_task(task_config)

# Manual trigger for testing
push_service.trigger_task(task_config["name"])
```

**Multi-Channel Push Example:**
```python
def send_alert(title, content, channels=["email", "telegram"]):
    """Send alert through multiple channels"""
    
    if "email" in channels:
        send_email(
            to=os.getenv("EMAIL_RECIPIENT"),
            subject=title,
            body=content
        )
    
    if "wechat" in channels:
        send_wechat_message(
            webhook_url=os.getenv("WECHAT_WEBHOOK_URL"),
            content=content
        )
    
    if "telegram" in channels:
        send_telegram_message(
            bot_token=os.getenv("TELEGRAM_BOT_TOKEN"),
            chat_id=os.getenv("TELEGRAM_CHAT_ID"),
            text=content
        )

# Usage
send_alert(
    title="突发舆情预警",
    content="检测到热度异常上升的话题:\n某重大事件详情...",
    channels=["email", "wechat", "telegram"]
)
```

## Common Patterns

### Custom Crawler for New Platform

```python
# Add to hotsearchcrawler/spiders/
import scrapy
from hotsearchcrawler.items import HotSearchItem

class NewPlatformSpider(scrapy.Spider):
    name = 'newplatform'
    start_urls = ['https://newplatform.com/hot']
    
    def parse(self, response):
        for item in response.css('.hot-item'):
            yield HotSearchItem(
                title=item.css('.title::text').get(),
                hot_value=item.css('.hot-value::text').get(),
                url=item.css('a::attr(href)').get(),
                platform='newplatform',
                rank=item.css('.rank::text').get()
            )
```

### Advanced Sentiment Analysis

```python
def detailed_sentiment_analysis(text, include_aspects=True):
    """Perform detailed sentiment analysis with aspect extraction"""
    
    prompt = f"""
    对以下文本进行详细的情感分析:
    
    文本: {text}
    
    请提供:
    1. 总体情感倾向(正面/负面/中性)及置信度
    2. 情感强度(1-10分)
    3. 关键情感词汇
    {"4. 不同方面的情感(如果适用)" if include_aspects else ""}
    
    以JSON格式返回结果。
    """
    
    response = client.chat.completions.create(
        model=os.getenv("MODEL_NAME"),
        messages=[
            {"role": "system", "content": "你是专业的情感分析系统。"},
            {"role": "user", "content": prompt}
        ],
        temperature=0.2
    )
    
    return response.choices[0].message.content
```

### Trend Detection

```python
def detect_trending_topics(time_window_hours=24, min_growth_rate=2.0):
    """Detect rapidly trending topics"""
    
    conn = get_database_connection()
    cursor = conn.cursor()
    
    query = """
    SELECT 
        title,
        platform,
        MAX(hot_value) as peak_value,
        MIN(hot_value) as initial_value,
        COUNT(*) as mention_count,
        MAX(hot_value) / MIN(hot_value) as growth_rate
    FROM hot_searches
    WHERE created_at >= DATE_SUB(NOW(), INTERVAL %s HOUR)
    GROUP BY title, platform
    HAVING growth_rate >= %s
    ORDER BY growth_rate DESC
    LIMIT 10
    """
    
    cursor.execute(query, (time_window_hours, min_growth_rate))
    trending = cursor.fetchall()
    
    conn.close()
    return trending
```

## Troubleshooting

**Browser Driver Issues:**
```bash
# Error: "chromedriver not found in PATH"
# Solution: Verify driver is in PATH
which chromedriver  # Linux/macOS
where chromedriver  # Windows

# Error: "session not created: This version of ChromeDriver only supports Chrome version XX"
# Solution: Download matching driver version for your browser
google-chrome --version  # Check Chrome version
# Download corresponding driver from https://chromedriver.chromium.org/
```

**Database Connection Errors:**
```python
# Error: "Access denied for user"
# Solution: Check MySQL credentials in .env
# Ensure user has proper privileges:
# GRANT ALL PRIVILEGES ON hotsearch.* TO 'your_user'@'localhost';

# Error: "Can't connect to MySQL server"
# Solution: Verify MySQL is running
# Linux: sudo systemctl status mysql
# macOS: brew services list
```

**LLM API Issues:**
```python
# Error: "Invalid API key"
# Solution: Verify environment variables are loaded
import os
from dotenv import load_dotenv
load_dotenv()
print(os.getenv("OPENAI_API_KEY"))  # Should not be None

# Error: "Rate limit exceeded"
# Solution: Implement exponential backoff
import time
from openai import RateLimitError

def call_llm_with_retry(prompt, max_retries=3):
    for attempt in range(max_retries):
        try:
            return client.chat.completions.create(...)
        except RateLimitError:
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)  # Exponential backoff
            else:
                raise
```

**Crawler Timeout/Blocking:**
```python
# In hotsearchcrawler/settings.py
# Adjust request delays to avoid blocking
DOWNLOAD_DELAY = 3  # Seconds between requests
CONCURRENT_REQUESTS_PER_DOMAIN = 1
RANDOMIZE_DOWNLOAD_DELAY = True

# Add User-Agent rotation
USER_AGENT_LIST = [
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36...',
    'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36...'
]
```

**Memory Issues with Large Analysis:**
```python
# Process hot searches in batches
def analyze_in_batches(items, batch_size=50):
    results = []
    for i in range(0, len(items), batch_size):
        batch = items[i:i+batch_size]
        batch_results = analyze_sentiment(batch)
        results.extend(batch_results)
        time.sleep(1)  # Rate limiting
    return results
```

## Project Structure Reference

```
├── app.py                          # Main web application entry
├── hotsearch_analysis_agent/       # Analysis system
│   ├── database.py                 # DB connection utilities
│   ├── llm_service.py              # LLM integration
│   ├── push_service.py             # Multi-channel notifications
│   └── query_engine.py             # Conversational query handler
├── hotsearchcrawler/               # Crawler cluster (separate)
│   ├── spiders/                    # Platform-specific spiders
│   │   ├── weibo_spider.py
│   │   ├── bilibili_spider.py
│   │   └── ...
│   ├── settings.py                 # Crawler configuration
│   └── pipelines.py                # Data processing pipelines
├── init.py                         # Database initialization
├── run_spiders.py                  # Crawler orchestration
├── test_push_task.py               # Push notification testing
└── requirements.txt                # Python dependencies
```

This skill enables AI coding agents to help developers deploy and customize this comprehensive public opinion analytics system with LLM-powered insights and real-time social media monitoring.
