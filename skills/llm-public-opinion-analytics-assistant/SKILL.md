---
name: llm-public-opinion-analytics-assistant
description: Chinese public opinion monitoring system with 26 real-time hot lists from 15 platforms, LLM-powered sentiment analysis, topic clustering, and multi-channel alerts
triggers:
  - set up public opinion monitoring system
  - analyze Chinese social media sentiment
  - monitor hot search trends across platforms
  - configure Weibo Bilibili hot list crawler
  - implement LLM-based topic clustering
  - send hot topic alerts to WeChat or Telegram
  - deploy multi-platform sentiment analysis
  - crawl Chinese news and video content
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is a comprehensive Chinese public opinion monitoring system that aggregates real-time data from **26 hot lists across 15 mainstream platforms** (Weibo, Bilibili, Douyin, Zhihu, etc.) and combines it with large language model analysis capabilities. The system provides conversational hot search queries, topic clustering, sentiment analysis, and multi-channel push notifications via Email, WeChat, Enterprise WeChat, and Telegram.

**Key Features:**
- Real-time crawler cluster for 15 Chinese platforms
- LLM-powered sentiment and trend analysis
- Topic clustering and correlation detection
- Video content extraction and analysis
- Multi-channel alert system (WeChat, Telegram, Email)
- Web interface with keyboard shortcuts for crawler control

## Installation

### Prerequisites

```bash
# System requirements
- Python 3.8+
- MySQL 5.7+
- Chrome/Edge browser (for Selenium)
```

### Step 1: Browser Driver Setup

Download the appropriate driver for your browser:

**Chrome:**
```bash
# Visit https://chromedriver.chromium.org/
# Download driver matching your Chrome version
# Place in /usr/local/bin/ (macOS/Linux) or C:\Windows\System32\ (Windows)
```

**Edge:**
```bash
# Visit https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/
# Download driver matching your Edge version
```

Verify installation:
```bash
chromedriver --version
# or
msedgedriver --version
```

### Step 2: Install Dependencies

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

### Step 3: Database Setup

```bash
# Install MySQL and create database
mysql -u root -p

CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

Initialize tables by referencing `init.py`:

```python
import pymysql

connection = pymysql.connect(
    host='localhost',
    user='root',
    password='your_password',
    database='hotsearch_db',
    charset='utf8mb4'
)

# Create core tables
with connection.cursor() as cursor:
    # Hot search items table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS hot_searches (
            id INT AUTO_INCREMENT PRIMARY KEY,
            platform VARCHAR(50) NOT NULL,
            rank INT,
            title VARCHAR(500) NOT NULL,
            url VARCHAR(1000),
            hot_score VARCHAR(100),
            content TEXT,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            INDEX idx_platform (platform),
            INDEX idx_created_at (created_at)
        )
    """)
    
    # Analysis results table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS analysis_results (
            id INT AUTO_INCREMENT PRIMARY KEY,
            query TEXT NOT NULL,
            result LONGTEXT NOT NULL,
            sentiment VARCHAR(50),
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    
    connection.commit()
```

### Step 4: Configuration

Create `.env` file in project root:

```bash
# Database configuration
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=your_mysql_password
DB_NAME=hotsearch_db

# LLM API configuration (OpenAI-compatible endpoint)
OPENAI_API_KEY=your_api_key
OPENAI_BASE_URL=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Or use Huawei Pangu model (recommended for Chinese content)
PANGU_API_KEY=your_pangu_key
PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model

# Push notification services
# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password

# WeChat Work (Enterprise WeChat)
WECHAT_WEBHOOK_URL=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=your_key
WECHAT_CORP_ID=your_corp_id
WECHAT_AGENT_ID=your_agent_id
WECHAT_SECRET=your_secret

# Telegram
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

Configure crawler settings in `hotsearchcrawler/settings.py`:

```python
# MySQL connection
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'root'
MYSQL_PASSWORD = 'your_password'
MYSQL_DB = 'hotsearch_db'

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

## Running the System

### Start the Main Application

```bash
python app.py
```

The web interface will be available at `http://localhost:5000`

### Manual Crawler Execution

Test individual platform crawlers:

```bash
# Test specific platform
cd hotsearchcrawler
scrapy crawl weibo_hot
scrapy crawl bilibili_hot
scrapy crawl douyin_hot

# Or use test script
python runspider-test.py
```

Run all crawlers:

```bash
python run_spiders.py
```

From the web interface, use keyboard shortcuts:
- **Ctrl+S**: Start all crawlers
- **Ctrl+E**: Stop all crawlers

## Core API Usage

### Query Hot Searches

```python
from hotsearch_analysis_agent.analyzer import HotSearchAnalyzer

# Initialize analyzer
analyzer = HotSearchAnalyzer(
    db_config={
        'host': 'localhost',
        'user': 'root',
        'password': 'your_password',
        'database': 'hotsearch_db'
    }
)

# Query hot searches by platform
weibo_trends = analyzer.query_hot_searches(
    platform='weibo',
    limit=10
)

for item in weibo_trends:
    print(f"Rank {item['rank']}: {item['title']} (Score: {item['hot_score']})")
```

### Conversational Query

```python
# Natural language query
response = analyzer.chat_query(
    user_input="最近关于人工智能的热点有哪些?",
    context_window=7  # days
)

print(response['answer'])
print(f"Sentiment: {response['sentiment']}")
```

### Topic Clustering

```python
# Cluster related topics
clusters = analyzer.cluster_topics(
    keyword="人工智能",
    days_back=7,
    min_cluster_size=3
)

for cluster in clusters:
    print(f"\nCluster: {cluster['theme']}")
    print(f"Size: {cluster['size']} items")
    for topic in cluster['topics']:
        print(f"  - {topic['title']} ({topic['platform']})")
```

### Sentiment Analysis

```python
# Analyze sentiment for specific topic
sentiment_report = analyzer.analyze_sentiment(
    topic="GPT-6",
    platforms=['weibo', 'zhihu', 'bilibili']
)

print(f"Overall Sentiment: {sentiment_report['overall']}")
print(f"Positive: {sentiment_report['positive_pct']}%")
print(f"Negative: {sentiment_report['negative_pct']}%")
print(f"Neutral: {sentiment_report['neutral_pct']}%")
```

### Deep Content Analysis (Including Videos)

```python
# Fetch and analyze detailed content (even video transcripts)
detailed_analysis = analyzer.analyze_topic_deep(
    topic="DeepSeek V4",
    fetch_content=True,  # Crawl article/video content
    include_videos=True   # Extract video info
)

print(detailed_analysis['summary'])
print(f"\nSources analyzed: {len(detailed_analysis['sources'])}")
```

## Push Notification System

### Configure Push Task

```python
from hotsearch_analysis_agent.push_manager import PushManager

push_mgr = PushManager()

# Create scheduled task
task_id = push_mgr.create_task(
    query="人工智能 AND 前沿科技",
    channels=['wechat', 'telegram', 'email'],
    schedule='0 9,18 * * *',  # Cron: 9 AM and 6 PM daily
    min_hot_score=500000,  # Only items with score > 500k
    sentiment_filter=['positive', 'neutral']  # Exclude negative
)

print(f"Task created: {task_id}")
```

### Test Push Immediately

```python
# Test push notification
python test_push_task.py --query "人工智能" --channel telegram

# Or in code
from hotsearch_analysis_agent.push import send_push

report = analyzer.generate_report(
    query="人工智能与前沿科技",
    days=7
)

# Send via Telegram
send_push(
    content=report,
    channel='telegram',
    chat_id=os.getenv('TELEGRAM_CHAT_ID'),
    bot_token=os.getenv('TELEGRAM_BOT_TOKEN')
)

# Send via WeChat Work
send_push(
    content=report,
    channel='wechat_work',
    webhook_url=os.getenv('WECHAT_WEBHOOK_URL')
)

# Send via Email
send_push(
    content=report,
    channel='email',
    recipients=['analyst@company.com'],
    subject='AI热点分析报告 - 2026-04-07'
)
```

### Report Generation Format

Generated reports follow this structure:

```markdown
## Report - 关于人工智能与前沿科技的热点分析
**Time**: 2026-04-07 12:32:00

> 本报告旨在梳理近期人工智能与硬件生态领域的关键进展...

### 🔍 核心发现与数据亮点
🔹 **大模型上下文能力迈入新阶段**：网传GPT-6提前曝光...

### 📰 详细新闻内容梳理
#### 1️⃣ 大模型上下文能力突破：GPT-6技术细节提前曝光
**标题**：GPT-6遭提前曝光, 2M超长上下文来了
**URL**：[🎬 点击观看视频](https://www.bilibili.com/video/BV13pSoBBEvX)
**摘要**：社区流传信息显示，下一代GPT模型或支持200万token上下文窗口...

### 📊 分析与总结
✅ **技术能力向"超长上下文"演进**
✅ **国产生态进入"协同验证"深水区**
```

## Common Patterns

### Multi-Platform Data Aggregation

```python
from hotsearch_analysis_agent.aggregator import MultiPlatformAggregator

aggregator = MultiPlatformAggregator()

# Get trending topics across all platforms
all_trends = aggregator.aggregate(
    platforms=['weibo', 'bilibili', 'douyin', 'zhihu'],
    hours_back=24,
    deduplicate=True  # Merge similar topics
)

# Find cross-platform hot topics
cross_platform = aggregator.find_cross_platform_topics(
    min_platforms=3,  # Must appear on at least 3 platforms
    similarity_threshold=0.8
)

for topic in cross_platform:
    print(f"\n{topic['title']}")
    print(f"Platforms: {', '.join(topic['platforms'])}")
    print(f"Total reach: {topic['total_hot_score']}")
```

### LLM Integration (Pangu Model)

```python
from hotsearch_analysis_agent.llm import PanguAnalyzer

# Using Huawei Pangu model for Chinese content
pangu = PanguAnalyzer(
    model_path=os.getenv('PANGU_MODEL_PATH'),
    device='cuda'  # or 'cpu'
)

# Long text analysis
long_article = "..." # Full article content

analysis = pangu.analyze(
    text=long_article,
    tasks=['sentiment', 'summary', 'keywords', 'entities']
)

print(f"Sentiment: {analysis['sentiment']}")
print(f"Summary: {analysis['summary']}")
print(f"Keywords: {', '.join(analysis['keywords'])}")
print(f"Entities: {analysis['entities']}")
```

### Video Content Extraction

```python
from hotsearch_analysis_agent.video_parser import VideoContentParser

parser = VideoContentParser()

# Extract content from Bilibili video
video_url = "https://www.bilibili.com/video/BV13pSoBBEvX"

video_data = parser.extract(video_url)

print(f"Title: {video_data['title']}")
print(f"Views: {video_data['views']}")
print(f"Description: {video_data['description']}")
print(f"Comments: {len(video_data['comments'])}")

# If transcript available
if video_data.get('transcript'):
    print(f"Transcript: {video_data['transcript'][:500]}...")
```

## Troubleshooting

### Crawler Issues

**Problem**: Crawler fails with "driver not found"
```bash
# Solution: Verify driver in PATH
which chromedriver  # macOS/Linux
where chromedriver  # Windows

# Add to PATH if missing
export PATH=$PATH:/path/to/driver
```

**Problem**: Getting blocked by platform anti-crawling
```python
# Solution: Add delays and rotate user agents
# In hotsearchcrawler/settings.py
DOWNLOAD_DELAY = 2
RANDOMIZE_DOWNLOAD_DELAY = True
USER_AGENT_LIST = [
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64)...',
    'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)...'
]
```

**Problem**: Cookie expired for authenticated platforms
```python
# Solution: Update cookies in settings.py
# Get fresh cookies from browser DevTools (Application > Cookies)
COOKIES = {
    'weibo': 'SUB=new_cookie_value; SUBP=...'
}
```

### Database Issues

**Problem**: Connection refused
```bash
# Check MySQL is running
sudo systemctl status mysql  # Linux
brew services list  # macOS

# Verify credentials
mysql -u root -p -h localhost
```

**Problem**: Character encoding errors
```sql
-- Ensure database uses utf8mb4
ALTER DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
ALTER TABLE hot_searches CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### LLM Analysis Issues

**Problem**: Out of memory with Pangu model
```python
# Solution: Use smaller batch size or CPU
pangu = PanguAnalyzer(
    model_path=os.getenv('PANGU_MODEL_PATH'),
    device='cpu',
    batch_size=1
)
```

**Problem**: Poor analysis quality for specific domains
```python
# Solution: Add domain-specific context
analyzer.chat_query(
    user_input="分析芯片行业趋势",
    domain_context="半导体、集成电路、芯片制造",
    temperature=0.3  # Lower for factual analysis
)
```

### Push Notification Issues

**Problem**: Telegram bot not sending
```bash
# Test bot token
curl https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/getMe

# Get chat ID
curl https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/getUpdates
```

**Problem**: WeChat Work webhook failing
```python
# Verify webhook URL format
# Should be: https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=XXX

# Test with curl
curl -X POST 'https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY' \
  -H 'Content-Type: application/json' \
  -d '{"msgtype": "text", "text": {"content": "Test"}}'
```

**Problem**: Email not sending via SMTP
```python
# For Gmail, use App Password (not regular password)
# Enable 2FA, then generate app password at:
# https://myaccount.google.com/apppasswords

# Test SMTP connection
import smtplib
server = smtplib.SMTP('smtp.gmail.com', 587)
server.starttls()
server.login(os.getenv('SMTP_USER'), os.getenv('SMTP_PASSWORD'))
print("SMTP connection successful")
```

## Performance Optimization

### Crawler Performance

```python
# In hotsearchcrawler/settings.py
CONCURRENT_REQUESTS = 32  # Increase for faster crawling
REACTOR_THREADPOOL_MAXSIZE = 20
DOWNLOAD_TIMEOUT = 30

# Enable HTTP caching
HTTPCACHE_ENABLED = True
HTTPCACHE_EXPIRATION_SECS = 3600
```

### Database Indexing

```sql
-- Add indexes for common queries
CREATE INDEX idx_platform_date ON hot_searches(platform, created_at);
CREATE INDEX idx_title_fulltext ON hot_searches(title) USING FULLTEXT;
CREATE INDEX idx_hot_score ON hot_searches(hot_score);
```

### LLM Batch Processing

```python
# Process multiple queries in batch
queries = ["AI发展", "芯片技术", "新能源汽车"]

results = analyzer.batch_analyze(
    queries=queries,
    max_workers=3  # Parallel processing
)

for query, result in zip(queries, results):
    print(f"{query}: {result['summary']}")
```

This skill enables AI coding agents to effectively deploy, configure, and utilize the LLM-Based Intelligent Public Opinion Analytics Assistant for comprehensive Chinese social media monitoring and sentiment analysis.
