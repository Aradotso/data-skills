---
name: llm-public-opinion-analytics-assistant
description: Chinese public opinion analytics assistant integrating 15 platforms with 26 trending lists, LLM analysis, sentiment detection, topic clustering, and multi-channel push notifications
triggers:
  - how do I set up the public opinion analytics assistant
  - analyze trending topics from Chinese social media platforms
  - configure hot topic crawlers for Weibo and Bilibili
  - set up sentiment analysis with Pangu LLM model
  - create push notifications for trending news to WeChat
  - implement topic clustering for Chinese news data
  - scrape and analyze hot search rankings from multiple platforms
  - configure multi-channel alerts for public opinion monitoring
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive Chinese public opinion monitoring system that combines real-time data from 15 major platforms (26 trending lists) with large language model analysis capabilities. Provides conversational hot search queries, topic clustering, sentiment analysis, and multi-channel push notifications (email, WeChat, Enterprise WeChat, Telegram).

## What This Project Does

- **Multi-Platform Data Collection**: Crawls trending topics from 15 Chinese platforms including Weibo, Bilibili, Douyin, Zhihu, Baidu, etc.
- **LLM-Powered Analysis**: Uses Huawei Pangu or compatible OpenAI-format models for sentiment analysis, topic clustering, and content understanding
- **Conversational Interface**: Natural language queries for hot search rankings and specific topic searches
- **Multi-Channel Push**: Automated alerts via Enterprise WeChat, Telegram, and email with customizable triggers
- **Video Content Analysis**: Extracts insights even from video-based news content

## Installation

### Prerequisites

1. **Browser Driver Setup** (required for content extraction):

```bash
# Download ChromeDriver or EdgeDriver matching your browser version
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Place driver in PATH or project directory
# Verify installation:
chromedriver --version
```

2. **Database Setup**:

```bash
# Install MySQL and create database
mysql -u root -p
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

3. **Python Environment**:

```bash
# Clone repository
git clone https://github.com/hmmnxkl/LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant.git
cd LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Configuration

Create `.env` file in project root:

```bash
# Database Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_username
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch_db

# LLM Configuration (OpenAI-compatible format)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1  # Or local Pangu endpoint

# Push Notification Channels (optional)
WECHAT_WEBHOOK_URL=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxx
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
```

Configure crawler settings in `hotsearchcrawler/settings.py`:

```python
# MySQL connection for crawler
MYSQL_CONFIG = {
    'host': 'localhost',
    'port': 3306,
    'user': 'your_username',
    'password': 'your_password',
    'database': 'hotsearch_db',
    'charset': 'utf8mb4'
}

# Optional: Add cookies for specific platforms
COOKIES = {
    'weibo': 'your_weibo_cookies',
    'bilibili': 'your_bilibili_cookies'
}
```

Initialize database schema:

```python
# Run database initialization
python init.py
```

## Key Commands

### Start the Web Application

```bash
# Launch main application
python app.py

# Access web interface at http://localhost:5000
```

### Run Crawlers

```bash
# Test individual crawler
python runspider-test.py

# Start all crawlers (or use web interface shortcut)
python run_spiders.py

# Crawlers run continuously and update database
```

### Test Push Notifications

```bash
# Test notification channels
python test_push_task.py
```

## Core Usage Patterns

### 1. Database Models and Query Patterns

```python
from hotsearch_analysis_agent.models import HotSearchItem, AnalysisResult
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
import os

# Database connection
DATABASE_URL = f"mysql+pymysql://{os.getenv('MYSQL_USER')}:{os.getenv('MYSQL_PASSWORD')}@{os.getenv('MYSQL_HOST')}:{os.getenv('MYSQL_PORT')}/{os.getenv('MYSQL_DATABASE')}"
engine = create_engine(DATABASE_URL)
Session = sessionmaker(bind=engine)
session = Session()

# Query recent hot topics from specific platform
recent_items = session.query(HotSearchItem).filter(
    HotSearchItem.platform == 'weibo',
    HotSearchItem.rank <= 10
).order_by(HotSearchItem.created_at.desc()).limit(10).all()

for item in recent_items:
    print(f"#{item.rank} {item.title} - {item.hot_value}")
    print(f"URL: {item.url}")

# Query by keyword
keyword_results = session.query(HotSearchItem).filter(
    HotSearchItem.title.like('%人工智能%')
).all()
```

### 2. LLM Analysis Integration

```python
from openai import OpenAI
import os

# Initialize LLM client (works with Pangu or OpenAI)
client = OpenAI(
    api_key=os.getenv('OPENAI_API_KEY'),
    base_url=os.getenv('OPENAI_API_BASE')
)

def analyze_sentiment(text):
    """Perform sentiment analysis on text"""
    response = client.chat.completions.create(
        model="pangu-embedded-7b",  # or gpt-4
        messages=[
            {"role": "system", "content": "你是一个专业的舆情分析助手,负责分析文本的情感倾向。请判断文本是正面、负面还是中性,并给出置信度。"},
            {"role": "user", "content": f"请分析以下文本的情感倾向:\n\n{text}"}
        ],
        temperature=0.3
    )
    return response.choices[0].message.content

def cluster_topics(titles):
    """Cluster related topics"""
    titles_text = "\n".join([f"{i+1}. {t}" for i, t in enumerate(titles)])
    
    response = client.chat.completions.create(
        model="pangu-embedded-7b",
        messages=[
            {"role": "system", "content": "你是一个话题聚类专家。请将相关的话题归类到一起,识别主要主题。"},
            {"role": "user", "content": f"请对以下热搜话题进行聚类分析:\n\n{titles_text}"}
        ],
        temperature=0.5
    )
    return response.choices[0].message.content

# Example usage
hot_titles = [item.title for item in recent_items]
cluster_result = cluster_topics(hot_titles)
print("话题聚类结果:", cluster_result)
```

### 3. Push Notification System

```python
import requests
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
import os

class NotificationManager:
    """Manage multi-channel push notifications"""
    
    @staticmethod
    def send_wechat(content, webhook_url=None):
        """Send to Enterprise WeChat"""
        url = webhook_url or os.getenv('WECHAT_WEBHOOK_URL')
        data = {
            "msgtype": "markdown",
            "markdown": {
                "content": content
            }
        }
        response = requests.post(url, json=data)
        return response.json()
    
    @staticmethod
    def send_telegram(content, bot_token=None, chat_id=None):
        """Send to Telegram"""
        bot_token = bot_token or os.getenv('TELEGRAM_BOT_TOKEN')
        chat_id = chat_id or os.getenv('TELEGRAM_CHAT_ID')
        url = f"https://api.telegram.org/bot{bot_token}/sendMessage"
        data = {
            "chat_id": chat_id,
            "text": content,
            "parse_mode": "Markdown"
        }
        response = requests.post(url, json=data)
        return response.json()
    
    @staticmethod
    def send_email(subject, content, to_email):
        """Send email notification"""
        msg = MIMEMultipart()
        msg['From'] = os.getenv('SMTP_USER')
        msg['To'] = to_email
        msg['Subject'] = subject
        
        msg.attach(MIMEText(content, 'html', 'utf-8'))
        
        with smtplib.SMTP(os.getenv('SMTP_HOST'), int(os.getenv('SMTP_PORT'))) as server:
            server.starttls()
            server.login(os.getenv('SMTP_USER'), os.getenv('SMTP_PASSWORD'))
            server.send_message(msg)

# Example: Push hot topic analysis
notifier = NotificationManager()

# Format analysis report
report = f"""## 热点分析报告
**时间**: 2026-05-17 12:00:00

### 核心发现
{cluster_result}

### 情感倾向
整体情感: 中性偏正面
"""

# Send to multiple channels
notifier.send_wechat(report)
notifier.send_telegram(report)
notifier.send_email("每日热点分析", report, "analyst@example.com")
```

### 4. Custom Crawler Development

```python
import scrapy
from hotsearchcrawler.items import HotSearchItem
from datetime import datetime

class CustomPlatformSpider(scrapy.Spider):
    """Template for adding new platform crawlers"""
    name = 'custom_platform'
    allowed_domains = ['example.com']
    start_urls = ['https://example.com/hot']
    
    def parse(self, response):
        # Extract hot search items
        for idx, item_selector in enumerate(response.css('.hot-item')):
            item = HotSearchItem()
            item['platform'] = 'custom_platform'
            item['rank'] = idx + 1
            item['title'] = item_selector.css('.title::text').get()
            item['url'] = item_selector.css('a::attr(href)').get()
            item['hot_value'] = item_selector.css('.hot-value::text').get()
            item['created_at'] = datetime.now()
            
            # Follow to detail page for content extraction
            yield scrapy.Request(
                item['url'],
                callback=self.parse_detail,
                meta={'item': item}
            )
    
    def parse_detail(self, response):
        """Extract detailed content"""
        item = response.meta['item']
        item['content'] = response.css('.article-content::text').getall()
        item['summary'] = response.css('.summary::text').get()
        yield item
```

### 5. Topic Analysis Workflow

```python
from datetime import datetime, timedelta

def analyze_trending_topics(keyword, days=7):
    """Comprehensive analysis of trending topics"""
    
    # 1. Query historical data
    start_date = datetime.now() - timedelta(days=days)
    items = session.query(HotSearchItem).filter(
        HotSearchItem.title.like(f'%{keyword}%'),
        HotSearchItem.created_at >= start_date
    ).all()
    
    if not items:
        return {"error": "No data found"}
    
    # 2. Aggregate by platform
    platform_stats = {}
    for item in items:
        if item.platform not in platform_stats:
            platform_stats[item.platform] = []
        platform_stats[item.platform].append(item)
    
    # 3. Sentiment analysis
    all_content = " ".join([item.content or item.title for item in items])
    sentiment = analyze_sentiment(all_content)
    
    # 4. Trend detection
    daily_counts = {}
    for item in items:
        date_key = item.created_at.strftime('%Y-%m-%d')
        daily_counts[date_key] = daily_counts.get(date_key, 0) + 1
    
    # 5. Generate report
    report = {
        "keyword": keyword,
        "period": f"{days} days",
        "total_mentions": len(items),
        "platforms": list(platform_stats.keys()),
        "sentiment": sentiment,
        "daily_trend": daily_counts,
        "top_items": sorted(items, key=lambda x: x.hot_value or 0, reverse=True)[:5]
    }
    
    return report

# Example usage
analysis = analyze_trending_topics("人工智能", days=7)
print(f"Found {analysis['total_mentions']} mentions across {len(analysis['platforms'])} platforms")
```

## Configuration Best Practices

### Crawler Scheduling

```python
# In hotsearchcrawler/settings.py
DOWNLOAD_DELAY = 3  # Respectful crawling
CONCURRENT_REQUESTS = 8
AUTOTHROTTLE_ENABLED = True

# Retry configuration
RETRY_TIMES = 3
RETRY_HTTP_CODES = [500, 502, 503, 504, 408, 429]

# User agent rotation
USER_AGENT_LIST = [
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
    'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36'
]
```

### LLM Model Selection

```python
# For Huawei Pangu local deployment
OPENAI_API_BASE = "http://localhost:8000/v1"
MODEL_NAME = "pangu-embedded-7b"

# For cloud API
OPENAI_API_BASE = "https://api.openai.com/v1"
MODEL_NAME = "gpt-4"

# Temperature settings
ANALYSIS_TEMPERATURE = 0.3  # More deterministic for sentiment
CLUSTERING_TEMPERATURE = 0.5  # More creative for grouping
```

## Troubleshooting

### Browser Driver Issues

```bash
# Error: "ChromeDriver version mismatch"
# Solution: Update driver to match browser version
chrome --version
# Download matching driver from https://chromedriver.chromium.org/

# Error: "Driver not found in PATH"
# Solution: Add driver location to PATH
export PATH=$PATH:/path/to/driver  # Linux/Mac
set PATH=%PATH%;C:\path\to\driver  # Windows
```

### Database Connection Issues

```python
# Test database connection
from sqlalchemy import create_engine, text

engine = create_engine(DATABASE_URL, echo=True)
with engine.connect() as conn:
    result = conn.execute(text("SELECT 1"))
    print("Database connected:", result.fetchone())

# Common fixes:
# 1. Check MySQL service is running: systemctl status mysql
# 2. Verify credentials in .env file
# 3. Ensure database exists: CREATE DATABASE IF NOT EXISTS hotsearch_db
```

### LLM API Errors

```python
# Handle API timeouts and retries
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=4, max=10))
def call_llm_with_retry(prompt):
    try:
        response = client.chat.completions.create(
            model="pangu-embedded-7b",
            messages=[{"role": "user", "content": prompt}],
            timeout=30
        )
        return response.choices[0].message.content
    except Exception as e:
        print(f"LLM API error: {e}")
        raise

# For local Pangu deployment issues:
# 1. Check model is loaded: curl http://localhost:8000/v1/models
# 2. Monitor GPU memory: nvidia-smi
# 3. Adjust batch size if OOM errors occur
```

### Crawler Blocking

```python
# Add proxy support in settings.py
DOWNLOADER_MIDDLEWARES = {
    'scrapy.downloadermiddlewares.httpproxy.HttpProxyMiddleware': 110,
    'hotsearchcrawler.middlewares.ProxyMiddleware': 100,
}

# Implement proxy rotation
class ProxyMiddleware:
    def process_request(self, request, spider):
        proxy_list = os.getenv('PROXY_LIST', '').split(',')
        if proxy_list:
            request.meta['proxy'] = random.choice(proxy_list)
```

### Push Notification Failures

```python
# Verify webhook URLs
import requests

def test_webhook(url):
    try:
        response = requests.post(url, json={"test": "message"}, timeout=5)
        print(f"Status: {response.status_code}, Response: {response.text}")
    except Exception as e:
        print(f"Webhook error: {e}")

# Test all channels
test_webhook(os.getenv('WECHAT_WEBHOOK_URL'))
```

## Advanced Features

### Video Content Extraction

```python
# The system can analyze video-based news using transcription
def extract_video_content(video_url):
    """Extract text from video content"""
    # Uses browser automation to get video description and comments
    from selenium import webdriver
    
    driver = webdriver.Chrome()
    driver.get(video_url)
    
    # Extract video description
    description = driver.find_element_by_css_selector('.video-desc').text
    
    # Get top comments (user sentiment indicator)
    comments = driver.find_elements_by_css_selector('.comment-item')
    comment_text = "\n".join([c.text for c in comments[:10]])
    
    driver.quit()
    
    return {
        "description": description,
        "comments": comment_text
    }
```

### Custom Analysis Pipelines

```python
# Create specialized analysis for specific domains
def financial_news_analysis(items):
    """Specialized analysis for financial topics"""
    
    # Extract financial keywords
    financial_keywords = ['股市', '基金', '投资', '上市', '融资']
    relevant_items = [
        item for item in items 
        if any(kw in item.title for kw in financial_keywords)
    ]
    
    # Perform sentiment with financial context
    prompt = f"""作为金融分析师,分析以下新闻标题的市场情绪:
    {[item.title for item in relevant_items]}
    
    请评估:
    1. 市场情绪(看涨/看跌/中性)
    2. 影响程度(高/中/低)
    3. 关键风险因素
    """
    
    analysis = call_llm_with_retry(prompt)
    return analysis
```

This skill enables AI agents to help developers deploy and customize a comprehensive Chinese public opinion monitoring system with LLM-powered analysis capabilities.
