---
name: llm-public-opinion-analytics-assistant
description: Multi-platform hot topic crawler and LLM-powered sentiment analysis system with real-time monitoring and multi-channel alerting
triggers:
  - how do I set up public opinion monitoring
  - crawl hot topics from multiple platforms
  - analyze sentiment of trending news
  - configure hot topic push notifications
  - cluster similar topics from social media
  - run multi-platform hot search crawler
  - set up LLM-based sentiment analysis
  - monitor and analyze public opinion trends
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What It Does

This is a comprehensive public opinion analytics system that combines web crawling with LLM-powered analysis. It monitors **15 platforms** across **26 hot topic lists** (Weibo, Bilibili, Baidu, Douyin, etc.), performs sentiment analysis, topic clustering, and sends alerts via WeChat, Telegram, or email.

**Key capabilities:**
- Real-time crawling of trending topics from major Chinese platforms
- LLM-based sentiment analysis and topic clustering
- Natural language query interface for hot topic insights
- Multi-channel push notifications (WeChat Work, Telegram, Email)
- Video content analysis extraction
- Keyboard shortcut control for crawler start/stop

## Installation

### Prerequisites

1. **Browser Driver Setup** (required for detail page crawling):

```bash
# Check your Chrome/Edge version first
google-chrome --version
# or
microsoft-edge --version

# Download matching ChromeDriver from:
# https://chromedriver.chromium.org/
# Or EdgeDriver from:
# https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Place driver in PATH (Linux/macOS example)
sudo mv chromedriver /usr/local/bin/
sudo chmod +x /usr/local/bin/chromedriver

# Verify installation
chromedriver --version
```

2. **MySQL Database**:

```bash
# Install MySQL
sudo apt-get install mysql-server  # Ubuntu/Debian
# or
brew install mysql  # macOS

# Create database
mysql -u root -p
```

```sql
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

3. **Python Environment**:

```bash
# Clone the repository
git clone https://github.com/hmmnxkl/LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant.git
cd LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant

# Create virtual environment
python -m venv venv
source venv/bin/activate  # Linux/macOS
# or
venv\Scripts\activate  # Windows

# Install dependencies
pip install -r requirements.txt
```

### Database Initialization

Reference the `init.py` file to create necessary tables:

```python
import pymysql

connection = pymysql.connect(
    host='localhost',
    user='your_user',
    password='your_password',
    database='hotsearch_db',
    charset='utf8mb4'
)

# Create tables for hot topics, analysis results, push tasks, etc.
# See init.py for full schema
```

## Configuration

### 1. Crawler Configuration

Edit `hotsearchcrawler/settings.py`:

```python
# MySQL connection
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_user'
MYSQL_PASSWORD = 'your_password'
MYSQL_DATABASE = 'hotsearch_db'

# Optional: Platform-specific cookies
COOKIES = {
    'weibo': 'your_weibo_cookie',
    'bilibili': 'your_bilibili_cookie',
}

# Crawler settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
ROBOTSTXT_OBEY = False
```

### 2. LLM Analysis Configuration

Create `.env` file in project root:

```bash
# Database
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch_db

# OpenAI-compatible API (supports Pangu, DeepSeek, etc.)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.your-llm-provider.com/v1
OPENAI_MODEL=gpt-4

# Push notification channels
# Email
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password

# WeChat Work Bot
WECHAT_WEBHOOK_URL=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxx

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

### 3. Using Huawei Pangu Model (Recommended)

The project recommends using Pangu for better Chinese text understanding:

```python
# Download Pangu model
# URL: https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

# Configure in .env
OPENAI_API_BASE=http://localhost:8000/v1  # Local Pangu service
OPENAI_MODEL=pangu-embedded-7b
```

## Running the System

### Start the Web Interface

```bash
# Start the analysis system and web UI
python app.py
```

Access the UI at `http://localhost:5000`

### Manual Crawler Execution

```bash
# Test single spider
python runspider-test.py

# Run all spiders
python run_spiders.py
```

The crawler will populate hot topics into the database, which the analysis system queries.

## Key Usage Patterns

### 1. Natural Language Queries

Through the web interface, ask questions like:

```python
# Example queries (in Chinese):
# "今天微博热搜榜前10是什么?"
# "搜索关于人工智能的热点"
# "分析最近科技类新闻的情感倾向"
# "哪些话题正在多个平台同时热门?"
```

### 2. Programmatic Topic Retrieval

```python
from hotsearch_analysis_agent.database import get_db_connection

def get_hot_topics(platform='weibo', limit=10):
    """Fetch latest hot topics from database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    query = """
        SELECT title, url, hot_value, platform, created_at
        FROM hot_topics
        WHERE platform = %s
        ORDER BY hot_value DESC, created_at DESC
        LIMIT %s
    """
    
    cursor.execute(query, (platform, limit))
    topics = cursor.fetchall()
    cursor.close()
    conn.close()
    
    return topics

# Usage
topics = get_hot_topics('weibo', 20)
for topic in topics:
    print(f"{topic['title']} - {topic['hot_value']}")
```

### 3. Sentiment Analysis with LLM

```python
from hotsearch_analysis_agent.llm_analyzer import analyze_sentiment

def analyze_topic_sentiment(topic_text, detail_content):
    """Analyze sentiment of a hot topic"""
    prompt = f"""
    分析以下新闻的情感倾向(正面/负面/中性):
    
    标题: {topic_text}
    内容: {detail_content}
    
    请输出JSON格式:
    {{
        "sentiment": "positive/negative/neutral",
        "confidence": 0.95,
        "keywords": ["关键词1", "关键词2"],
        "summary": "简短总结"
    }}
    """
    
    result = analyze_sentiment(prompt)
    return result

# Usage
sentiment = analyze_topic_sentiment(
    "GPT-6遭提前曝光, 2M超长上下文来了",
    "社区流传信息显示，下一代GPT模型或支持200万token上下文窗口..."
)
print(sentiment['sentiment'])  # "positive"
```

### 4. Topic Clustering

```python
from hotsearch_analysis_agent.clustering import cluster_similar_topics

def find_related_topics(keywords, time_range='24h'):
    """Cluster topics by semantic similarity"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    # Fetch topics from last 24h
    query = """
        SELECT id, title, content, platform
        FROM hot_topics
        WHERE created_at > DATE_SUB(NOW(), INTERVAL 1 DAY)
    """
    
    cursor.execute(query)
    topics = cursor.fetchall()
    
    # Use LLM to cluster
    clusters = cluster_similar_topics(topics, keywords)
    
    return clusters

# Usage
ai_clusters = find_related_topics(['人工智能', 'AI', '大模型'])
for cluster in ai_clusters:
    print(f"Cluster: {cluster['theme']}")
    for topic in cluster['topics']:
        print(f"  - {topic['title']} ({topic['platform']})")
```

### 5. Setting Up Push Tasks

```python
from hotsearch_analysis_agent.push_manager import create_push_task

def setup_ai_monitoring():
    """Create push task for AI-related hot topics"""
    task = create_push_task(
        name="AI热点监控",
        keywords=["人工智能", "大模型", "ChatGPT", "AI"],
        platforms=["weibo", "zhihu", "bilibili"],
        channels=["wechat", "telegram"],
        frequency="hourly",
        threshold=50000  # Hot value threshold
    )
    
    return task

# Test push manually
from test_push_task import test_push

test_push(
    channel='telegram',
    title='AI热点汇总',
    content='过去24小时内检测到5个相关热点...'
)
```

### 6. Video Content Analysis

For video-based hot topics (Bilibili, Douyin):

```python
from hotsearch_analysis_agent.video_extractor import extract_video_info

def analyze_video_topic(video_url):
    """Extract and analyze video content"""
    # Uses browser driver to load page and extract info
    video_info = extract_video_info(video_url)
    
    # video_info contains:
    # - title
    # - description
    # - comments (top N)
    # - view count
    # - danmaku samples
    
    # Analyze with LLM
    analysis_prompt = f"""
    视频标题: {video_info['title']}
    简介: {video_info['description']}
    热门评论: {video_info['comments'][:5]}
    
    请分析该视频的主要话题和公众反应。
    """
    
    return analyze_sentiment(analysis_prompt)

# Usage
analysis = analyze_video_topic('https://www.bilibili.com/video/BV13pSoBBEvX/')
```

## Common Workflows

### Workflow 1: Daily Hot Topic Report

```python
import schedule
import time
from datetime import datetime

def generate_daily_report():
    """Generate and send daily hot topic summary"""
    platforms = ['weibo', 'zhihu', 'bilibili', 'toutiao']
    all_topics = []
    
    for platform in platforms:
        topics = get_hot_topics(platform, 10)
        all_topics.extend(topics)
    
    # Cluster and analyze
    clusters = cluster_similar_topics(all_topics, auto_detect_themes=True)
    
    # Generate report
    report = f"""
    # 每日热点分析报告
    **时间**: {datetime.now().strftime('%Y-%m-%d %H:%M')}
    
    ## 核心话题
    """
    
    for cluster in clusters[:5]:
        report += f"\n### {cluster['theme']}\n"
        sentiment = analyze_sentiment(cluster['summary'])
        report += f"情感倾向: {sentiment['sentiment']}\n"
        report += f"相关热度: {cluster['total_heat']}\n\n"
    
    # Push via configured channels
    test_push('wechat', '每日热点报告', report)
    test_push('email', '每日热点报告', report)

# Schedule daily at 8 AM
schedule.every().day.at("08:00").do(generate_daily_report)

while True:
    schedule.run_pending()
    time.sleep(60)
```

### Workflow 2: Real-time Crisis Monitoring

```python
def monitor_crisis_keywords():
    """Monitor for crisis-related keywords in real-time"""
    crisis_keywords = ['事故', '泄漏', '召回', '安全隐患', '投诉']
    
    while True:
        recent_topics = get_hot_topics_last_hours(1)
        
        for topic in recent_topics:
            if any(kw in topic['title'] for kw in crisis_keywords):
                # Immediate analysis
                detail = fetch_topic_detail(topic['url'])
                sentiment = analyze_sentiment(topic['title'], detail)
                
                if sentiment['sentiment'] == 'negative':
                    alert_message = f"""
                    ⚠️ 危机预警
                    
                    话题: {topic['title']}
                    平台: {topic['platform']}
                    热度: {topic['hot_value']}
                    情感: {sentiment['sentiment']}
                    
                    详情: {topic['url']}
                    """
                    
                    # Immediate push to all channels
                    test_push('telegram', '危机预警', alert_message)
                    test_push('wechat', '危机预警', alert_message)
        
        time.sleep(300)  # Check every 5 minutes
```

## Troubleshooting

### Browser Driver Issues

```bash
# If "chromedriver not found" error:
which chromedriver  # Should show path
echo $PATH  # Check if driver location is in PATH

# If version mismatch:
chromedriver --version
google-chrome --version
# Download matching version from https://chromedriver.chromium.org/
```

### Database Connection Errors

```python
# Test MySQL connection
import pymysql

try:
    conn = pymysql.connect(
        host='localhost',
        user='your_user',
        password='your_password',
        database='hotsearch_db'
    )
    print("Connection successful")
    conn.close()
except Exception as e:
    print(f"Connection failed: {e}")
```

### Crawler Not Collecting Data

```bash
# Test individual spider
cd hotsearchcrawler
scrapy crawl weibo_spider -L DEBUG

# Check for:
# - Cookie expiration (some platforms require login)
# - IP blocking (add delays or proxies)
# - Site structure changes (update selectors)
```

### LLM API Errors

```python
# Test API connectivity
import openai
import os

openai.api_key = os.getenv('OPENAI_API_KEY')
openai.api_base = os.getenv('OPENAI_API_BASE')

try:
    response = openai.ChatCompletion.create(
        model=os.getenv('OPENAI_MODEL'),
        messages=[{"role": "user", "content": "测试"}]
    )
    print("API working:", response.choices[0].message.content)
except Exception as e:
    print(f"API error: {e}")
```

### Push Notifications Not Sending

```python
# Test each channel individually
from test_push_task import test_push

# Test email
test_push('email', 'Test', 'This is a test message')

# Test WeChat Work
test_push('wechat', 'Test', 'This is a test message')

# Test Telegram
test_push('telegram', 'Test', 'This is a test message')

# Check .env configuration and network connectivity
```

## Performance Tips

1. **Crawler optimization**: Adjust `CONCURRENT_REQUESTS` and `DOWNLOAD_DELAY` in settings.py based on your server capacity

2. **Database indexing**: Add indexes on frequently queried columns:

```sql
CREATE INDEX idx_platform_created ON hot_topics(platform, created_at);
CREATE INDEX idx_hot_value ON hot_topics(hot_value DESC);
CREATE INDEX idx_keywords ON hot_topics(title);  -- Full-text search
```

3. **LLM caching**: Cache analysis results to avoid re-analyzing same content:

```python
import hashlib
import json

def cached_analysis(content, cache_hours=24):
    cache_key = hashlib.md5(content.encode()).hexdigest()
    # Check cache in database/redis
    cached = get_from_cache(cache_key)
    if cached:
        return json.loads(cached)
    
    result = analyze_sentiment(content)
    save_to_cache(cache_key, json.dumps(result), expire=cache_hours*3600)
    return result
```
