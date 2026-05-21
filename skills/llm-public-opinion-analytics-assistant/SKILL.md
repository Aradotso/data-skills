---
name: llm-public-opinion-analytics-assistant
description: Multi-platform public opinion analytics system with LLM-powered analysis, real-time hot search tracking, sentiment analysis, and multi-channel alerts
triggers:
  - "scrape hot search data from Chinese platforms"
  - "analyze public opinion trends with LLM"
  - "set up sentiment analysis for social media topics"
  - "configure hot topic push notifications to WeChat"
  - "cluster and analyze trending news topics"
  - "build a public opinion monitoring dashboard"
  - "extract content from video news pages"
  - "deploy Pangu model for Chinese text analysis"
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is a comprehensive public opinion analytics system that monitors 26 hot search lists across 15 major Chinese platforms (Weibo, Bilibili, Baidu, Toutiao, etc.). It combines web scraping with LLM-powered analysis to provide conversational queries, topic clustering, sentiment analysis, and multi-channel alerts (Email, WeChat, Enterprise WeChat, Telegram).

## What It Does

- **Real-time Data Collection**: Scrapy-based crawler cluster for 15 platforms
- **LLM Analysis**: Topic clustering, sentiment analysis, and conversational queries using Pangu or OpenAI-compatible models
- **Content Extraction**: Extracts full content from news detail pages, including video transcripts
- **Multi-channel Alerts**: Push reports via Email (SMTP), WeChat, Enterprise WeChat, or Telegram
- **Web Dashboard**: Frontend interface for querying, browsing, and controlling crawlers

## Installation

### Prerequisites

**1. Browser Driver Setup**

The system uses Selenium for content extraction. Install ChromeDriver or EdgeDriver:

```bash
# Check your browser version first
google-chrome --version  # or microsoft-edge --version

# Download matching driver from:
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Place driver in PATH (Linux/Mac example)
sudo mv chromedriver /usr/local/bin/
chmod +x /usr/local/bin/chromedriver

# Verify installation
chromedriver --version
```

**2. MySQL Database**

```sql
-- Create database and user
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'hotsearch_user'@'localhost' IDENTIFIED BY 'your_password';
GRANT ALL PRIVILEGES ON hotsearch_db.* TO 'hotsearch_user'@'localhost';
FLUSH PRIVILEGES;
```

**3. Python Environment**

```bash
# Clone repository
git clone https://github.com/hmmnxkl/LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant.git
cd LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant

# Create virtual environment
python3 -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Configuration

**1. Database Initialization**

```python
# Reference init.py for table schemas
# Run initialization script
python init.py
```

**2. Environment Variables (`.env`)**

```bash
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=hotsearch_user
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
OPENAI_MODEL=gpt-4

# Or use Pangu model (local deployment)
USE_PANGU=true
PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model

# Push Notification Channels
# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
SMTP_FROM=your_email@gmail.com
SMTP_TO=recipient@example.com

# Enterprise WeChat Bot
WEWORK_BOT_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

**3. Crawler Settings (`hotsearchcrawler/settings.py`)**

```python
# MySQL connection for crawler
MYSQL_SETTINGS = {
    'host': 'localhost',
    'port': 3306,
    'user': 'hotsearch_user',
    'password': 'your_password',
    'database': 'hotsearch_db',
    'charset': 'utf8mb4'
}

# Optional: Platform-specific cookies
COOKIES = {
    'weibo': 'your_weibo_cookie',
    'bilibili': 'your_bilibili_cookie'
}
```

## Running the System

### Start the Web Application

```bash
# Launch main application
python app.py

# Access dashboard at http://localhost:5000
```

### Manual Crawler Control

```bash
# Test individual spider
python runspider-test.py

# Run all spiders (usually triggered from web UI)
python run_spiders.py
```

### Testing Push Notifications

```bash
# Test notification channels
python test_push_task.py
```

## Key Usage Patterns

### 1. Data Collection

**Starting Crawlers from Web UI:**

The frontend provides keyboard shortcuts to control crawlers:

```javascript
// Triggered via web dashboard
// Starts all configured spiders
// Data stored in MySQL tables: weibo_hot, bilibili_hot, baidu_hot, etc.
```

**Manual Spider Execution:**

```python
# Run specific spider
from scrapy.crawler import CrawlerProcess
from hotsearchcrawler.spiders.weibo_spider import WeiboSpider

process = CrawlerProcess()
process.crawl(WeiboSpider)
process.start()
```

### 2. LLM-Powered Analysis

**Conversational Query:**

```python
from hotsearch_analysis_agent.analyzer import OpinionAnalyzer

analyzer = OpinionAnalyzer()

# Natural language query
query = "最近关于人工智能的热点话题有哪些？"
result = analyzer.query(query)

print(result['analysis'])  # LLM-generated analysis
print(result['related_news'])  # Relevant news items
```

**Topic Clustering:**

```python
# Cluster related topics across platforms
clusters = analyzer.cluster_topics(
    keywords=["人工智能", "大模型"],
    days=7,
    min_cluster_size=3
)

for cluster in clusters:
    print(f"Cluster: {cluster['theme']}")
    print(f"Items: {len(cluster['items'])}")
    print(f"Platforms: {cluster['platforms']}")
```

**Sentiment Analysis:**

```python
# Analyze sentiment for specific topic
sentiment = analyzer.analyze_sentiment(
    topic="GPT-6发布",
    platforms=["weibo", "bilibili", "zhihu"]
)

print(f"Overall sentiment: {sentiment['overall']}")  # positive/negative/neutral
print(f"Confidence: {sentiment['confidence']}")
print(f"Distribution: {sentiment['distribution']}")  # {'positive': 0.6, 'negative': 0.2, ...}
```

### 3. Content Extraction from Detail Pages

```python
from hotsearch_analysis_agent.content_extractor import ContentExtractor

extractor = ContentExtractor()

# Extract full article/video content
url = "https://www.bilibili.com/video/BV13pSoBBEvX"
content = extractor.extract(url)

print(content['title'])
print(content['text'])  # Extracted text, even from video pages
print(content['metadata'])  # Views, publish time, author, etc.
```

### 4. Setting Up Push Tasks

**Email Push:**

```python
from hotsearch_analysis_agent.push import EmailPusher

pusher = EmailPusher(
    smtp_host=os.getenv('SMTP_HOST'),
    smtp_port=int(os.getenv('SMTP_PORT')),
    username=os.getenv('SMTP_USER'),
    password=os.getenv('SMTP_PASSWORD')
)

# Push daily report
report = analyzer.generate_report(
    topics=["人工智能", "前沿科技"],
    days=1
)

pusher.send(
    to=os.getenv('SMTP_TO'),
    subject="每日舆情分析报告",
    content=report
)
```

**Enterprise WeChat Push:**

```python
from hotsearch_analysis_agent.push import WeComPusher

pusher = WeComPusher(webhook=os.getenv('WEWORK_BOT_WEBHOOK'))

pusher.send_markdown(
    content=f"# 热点预警\n\n{report}",
    mentioned_list=["@all"]  # Mention all members
)
```

**Telegram Push:**

```python
from hotsearch_analysis_agent.push import TelegramPusher

pusher = TelegramPusher(
    bot_token=os.getenv('TELEGRAM_BOT_TOKEN'),
    chat_id=os.getenv('TELEGRAM_CHAT_ID')
)

pusher.send_message(
    text=report,
    parse_mode='Markdown'
)
```

### 5. Scheduled Analysis Tasks

```python
import schedule
import time

def daily_analysis():
    analyzer = OpinionAnalyzer()
    report = analyzer.generate_report(
        topics=["AI", "芯片", "新能源"],
        days=1,
        include_sentiment=True,
        include_clustering=True
    )
    
    # Push to multiple channels
    email_pusher.send(to=os.getenv('SMTP_TO'), subject="每日报告", content=report)
    wecom_pusher.send_markdown(content=report)

# Schedule daily at 12:00
schedule.every().day.at("12:00").do(daily_analysis)

while True:
    schedule.run_pending()
    time.sleep(60)
```

## Database Schema Reference

**Hot Search Tables (per platform):**

```sql
CREATE TABLE weibo_hot (
    id INT AUTO_INCREMENT PRIMARY KEY,
    rank INT,
    title VARCHAR(500),
    url VARCHAR(1000),
    heat_value BIGINT,
    category VARCHAR(50),
    collected_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_collected_at (collected_at),
    INDEX idx_title (title(255))
);

-- Similar tables: bilibili_hot, baidu_hot, toutiao_hot, etc.
```

**Detail Content Cache:**

```sql
CREATE TABLE content_cache (
    id INT AUTO_INCREMENT PRIMARY KEY,
    url VARCHAR(1000) UNIQUE,
    title VARCHAR(500),
    content TEXT,
    author VARCHAR(200),
    publish_time DATETIME,
    platform VARCHAR(50),
    metadata JSON,
    extracted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_url (url(255)),
    INDEX idx_platform (platform)
);
```

**Analysis Results:**

```sql
CREATE TABLE analysis_results (
    id INT AUTO_INCREMENT PRIMARY KEY,
    query_hash VARCHAR(64) UNIQUE,
    query TEXT,
    result JSON,
    model VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_query_hash (query_hash),
    INDEX idx_created_at (created_at)
);
```

## API Endpoints (Web Dashboard)

```python
# Start/Stop crawlers
POST /api/crawler/start
POST /api/crawler/stop

# Query hot searches
GET /api/hotsearch?platform=weibo&limit=50

# LLM analysis
POST /api/analyze
{
    "query": "分析最近AI话题的舆情趋势",
    "platforms": ["weibo", "bilibili"],
    "days": 7
}

# Get topic clusters
POST /api/cluster
{
    "keywords": ["人工智能"],
    "days": 7,
    "min_cluster_size": 3
}

# Sentiment analysis
POST /api/sentiment
{
    "topic": "GPT-6",
    "platforms": ["weibo", "zhihu"]
}
```

## Pangu Model Integration

For enhanced Chinese text understanding, deploy Huawei Pangu model locally:

```python
# Download model from:
# https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

from transformers import AutoModelForCausalLM, AutoTokenizer

model_path = "/path/to/openpangu-embedded-7b-model"
tokenizer = AutoTokenizer.from_pretrained(model_path, trust_remote_code=True)
model = AutoModelForCausalLM.from_pretrained(model_path, trust_remote_code=True)

# Use in analyzer
analyzer = OpinionAnalyzer(
    model=model,
    tokenizer=tokenizer,
    use_local=True
)
```

## Troubleshooting

**ChromeDriver Version Mismatch:**

```bash
# Error: "This version of ChromeDriver only supports Chrome version XX"
# Solution: Download matching driver version
chromedriver --version
google-chrome --version  # Versions must match major release
```

**MySQL Connection Failed:**

```python
# Error: "Access denied for user"
# Check credentials in .env
# Verify MySQL user privileges
GRANT ALL PRIVILEGES ON hotsearch_db.* TO 'hotsearch_user'@'localhost';
FLUSH PRIVILEGES;
```

**LLM API Timeout:**

```python
# Increase timeout in analyzer
analyzer = OpinionAnalyzer(
    api_timeout=120,  # seconds
    max_retries=3
)
```

**Empty Crawler Results:**

```bash
# Check if cookies are required for platform
# Update hotsearchcrawler/settings.py with valid cookies
# Or use rotating proxies for rate-limited platforms
```

**Video Content Extraction Fails:**

```python
# Ensure browser driver is properly configured
# Check if platform requires login for video access
# Verify network connectivity to CDN servers
```

**Push Notification Not Received:**

```python
# Test individual channels
python test_push_task.py

# Verify webhook URLs and tokens
# Check firewall rules for outbound SMTP/HTTPS
```

## Best Practices

1. **Rate Limiting**: Configure delays between crawler requests to avoid IP bans
2. **Data Deduplication**: Use `title` and `url` hash to prevent duplicate entries
3. **Cache Management**: Implement TTL for `content_cache` table to manage storage
4. **Model Selection**: Use Pangu for Chinese-specific analysis, GPT-4 for multilingual
5. **Monitoring**: Set up alerts for crawler failures and LLM API errors
6. **Backup**: Regular MySQL backups for historical trend analysis
