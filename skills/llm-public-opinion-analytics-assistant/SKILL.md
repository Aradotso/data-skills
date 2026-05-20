---
name: llm-public-opinion-analytics-assistant
description: Multi-platform public opinion analytics assistant with web scraping, LLM analysis, and multi-channel alert push capabilities
triggers:
  - how do i set up the public opinion analytics assistant
  - crawl hot search data from multiple platforms
  - analyze sentiment and cluster topics using LLM
  - configure multi-channel push notifications for hot topics
  - integrate pangu or openai models for opinion analysis
  - scrape bilibili weibo toutiao trending data
  - set up mysql database for hot search trends
  - query and analyze public opinion with conversational interface
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive public opinion monitoring system that integrates **15 platforms** and **26 trending lists** with LLM-powered analysis. Supports conversational queries, topic clustering, sentiment analysis, and multi-channel alert push (Email, WeChat, Enterprise WeChat, Telegram).

## What It Does

- **Multi-Platform Data Collection**: Scrapes trending topics from Weibo, Bilibili, Toutiao, and 12+ other Chinese platforms
- **LLM-Powered Analysis**: Topic clustering, sentiment analysis, and trend summarization using OpenAI-compatible APIs or Huawei Pangu models
- **Conversational Interface**: Natural language queries for hot search lists and topic analysis
- **Multi-Channel Push**: Scheduled reports via email, WeChat Work, Telegram, and enterprise messaging
- **Video Content Extraction**: Extracts insights even from video-based news content

## Installation

### Prerequisites

**1. Browser Driver Setup**

Download ChromeDriver or EdgeDriver matching your browser version:

```bash
# Verify Chrome version
google-chrome --version  # Linux
# or check in browser: Settings → About

# Download driver from:
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/microsoft-edge/tools/webdriver/

# Add to PATH (Linux/macOS)
sudo mv chromedriver /usr/local/bin/
chmod +x /usr/local/bin/chromedriver

# Verify installation
chromedriver --version
```

**2. MySQL Database**

```sql
-- Create database
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Create main tables (simplified schema)
CREATE TABLE hot_search_items (
    id INT AUTO_INCREMENT PRIMARY KEY,
    platform VARCHAR(50),
    title VARCHAR(500),
    url VARCHAR(1000),
    rank INT,
    heat_value VARCHAR(100),
    category VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE analysis_results (
    id INT AUTO_INCREMENT PRIMARY KEY,
    query TEXT,
    result TEXT,
    sentiment VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
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

**1. Crawler Settings (`hotsearchcrawler/settings.py`)**

```python
# MySQL Configuration
MYSQL_CONFIG = {
    'host': 'localhost',
    'port': 3306,
    'user': 'your_user',
    'password': 'your_password',
    'database': 'hotsearch_db',
    'charset': 'utf8mb4'
}

# Optional: Platform cookies for authenticated access
COOKIES = {
    'weibo': 'your_weibo_cookies',
    'bilibili': 'your_bilibili_cookies'
}
```

**2. Analysis System (`.env` file)**

```bash
# Database
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Or use Huawei Pangu
# PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model
# USE_PANGU=true

# Push Notification Channels
# Email
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
EMAIL_RECIPIENTS=recipient1@example.com,recipient2@example.com

# Enterprise WeChat Robot
WECHAT_WORK_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=your_key

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

## Running the System

### Start Web Interface

```bash
# Run main application
python app.py

# Access at http://localhost:5000
```

### Run Crawlers

**Via Web Interface**: Use the UI to start/stop crawlers with keyboard shortcuts

**Via Command Line**:

```bash
# Test individual spider
python runspider-test.py --spider weibo

# Run all spiders
python run_spiders.py
```

### Test Push Notifications

```bash
# Test push task configuration
python test_push_task.py
```

## Key Usage Patterns

### 1. Programmatic Data Collection

```python
from hotsearchcrawler.spiders.weibo_spider import WeiboSpider
from hotsearchcrawler.db_handler import DBHandler

# Initialize database handler
db = DBHandler()

# Run spider
spider = WeiboSpider()
items = spider.parse()

# Save to database
for item in items:
    db.save_item({
        'platform': 'weibo',
        'title': item['title'],
        'url': item['url'],
        'rank': item['rank'],
        'heat_value': item['heat']
    })
```

### 2. LLM-Based Analysis

```python
from hotsearch_analysis_agent.analyzer import OpinionAnalyzer

analyzer = OpinionAnalyzer()

# Query hot topics
query = "分析关于人工智能的热点话题"
results = analyzer.query_topics(query)

# Sentiment analysis
sentiment = analyzer.analyze_sentiment(results)
print(f"Sentiment: {sentiment['overall']}")  # positive/negative/neutral

# Topic clustering
clusters = analyzer.cluster_topics(results)
for cluster in clusters:
    print(f"Cluster: {cluster['theme']}")
    print(f"Keywords: {', '.join(cluster['keywords'])}")
```

### 3. Multi-Channel Push Configuration

```python
from hotsearch_analysis_agent.push_manager import PushManager

push_mgr = PushManager()

# Configure push task
task = {
    'name': '科技热点监控',
    'query': '人工智能 OR 大模型',
    'schedule': '0 9,18 * * *',  # Cron format: 9 AM and 6 PM daily
    'channels': ['email', 'wechat_work', 'telegram'],
    'threshold': {
        'heat_value': 100000,
        'rank': 10
    }
}

push_mgr.create_task(task)
```

### 4. Data Retrieval API

```python
from hotsearch_analysis_agent.data_service import DataService

service = DataService()

# Get trending topics by platform
weibo_trends = service.get_trends(platform='weibo', limit=20)

# Search by keyword
results = service.search_topics(
    keyword='人工智能',
    platforms=['weibo', 'bilibili', 'toutiao'],
    date_range=('2026-05-01', '2026-05-19')
)

# Get detailed content (including video transcripts)
details = service.get_topic_details(url='https://www.bilibili.com/video/BV13pSoBBEvX/')
print(details['summary'])
```

### 5. Custom LLM Integration

```python
from hotsearch_analysis_agent.llm_client import LLMClient

# Use OpenAI-compatible API
client = LLMClient(
    api_key=os.getenv('OPENAI_API_KEY'),
    base_url=os.getenv('OPENAI_API_BASE'),
    model='gpt-4'
)

# Or use Huawei Pangu model (local)
# client = LLMClient(
#     model_type='pangu',
#     model_path='/path/to/openpangu-embedded-7b-model'
# )

# Generate analysis report
report = client.generate_report(
    topics=results,
    prompt_template="""
    分析以下热点话题,总结核心发现:
    {topics}
    
    重点关注技术进展和商业影响。
    """
)
```

## Common Workflows

### Daily Monitoring Workflow

```python
# 1. Automated crawling (scheduled via cron or system UI)
# 2. Query latest trends
latest = service.get_trends(hours=24)

# 3. Analyze with LLM
analysis = analyzer.analyze_trends(latest, focus='sentiment,clustering')

# 4. Push alerts if threshold met
if analysis['max_heat'] > 500000:
    push_mgr.send_alert(
        title='高热度舆情预警',
        content=analysis['summary'],
        channels=['wechat_work']
    )
```

### Custom Topic Monitoring

```python
# Define monitoring target
KEYWORDS = ['芯片', '半导体', '国产算力']

# Continuous monitoring
def monitor_topics():
    results = service.search_topics(
        keyword=' OR '.join(KEYWORDS),
        platforms=['all']
    )
    
    # Filter by engagement
    high_engagement = [
        r for r in results 
        if r.get('heat_value', 0) > 100000
    ]
    
    if high_engagement:
        # Generate detailed report
        report = analyzer.generate_detailed_report(
            topics=high_engagement,
            include_source_content=True
        )
        
        # Save and push
        db.save_report(report)
        push_mgr.send_report(report, channels=['email', 'telegram'])
```

## Troubleshooting

### ChromeDriver Issues

```bash
# Error: ChromeDriver version mismatch
# Solution: Download exact version matching your browser

chromedriver --version
google-chrome --version  # Should match major version

# Error: ChromeDriver not found in PATH
export PATH=$PATH:/path/to/chromedriver/directory
```

### Database Connection Errors

```python
# Test connection
from hotsearchcrawler.db_handler import DBHandler

try:
    db = DBHandler()
    db.test_connection()
    print("Database connection successful")
except Exception as e:
    print(f"Connection failed: {e}")
    # Check .env file and MySQL service status
```

### LLM API Rate Limits

```python
# Implement retry logic
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=4, max=10)
)
def analyze_with_retry(query):
    return analyzer.query_topics(query)
```

### Cookie Expiration (Platform Authentication)

```python
# Update cookies in settings.py
# Extract from browser DevTools → Application → Cookies

# For Weibo:
# 1. Login to weibo.com
# 2. Copy cookie string
# 3. Update COOKIES['weibo'] in settings.py
```

### Memory Issues with Large Datasets

```python
# Batch processing
def process_large_dataset(date_range, batch_size=1000):
    total_records = service.count_records(date_range)
    
    for offset in range(0, total_records, batch_size):
        batch = service.get_trends(
            date_range=date_range,
            limit=batch_size,
            offset=offset
        )
        
        # Process batch
        analysis = analyzer.analyze_trends(batch)
        db.save_analysis(analysis)
        
        # Clear memory
        del batch
```

## Advanced Features

### Video Content Analysis

```python
# Extract insights from video platforms (Bilibili, Douyin)
video_analyzer = VideoContentAnalyzer()

video_url = 'https://www.bilibili.com/video/BV13pSoBBEvX/'
transcript = video_analyzer.extract_transcript(video_url)
summary = analyzer.summarize_content(transcript)

# Include in opinion analysis
combined_analysis = analyzer.analyze_multimodal(
    text_sources=text_results,
    video_sources=[summary]
)
```

### Custom Report Templates

```python
# Define report structure
template = """
## 舆情分析报告 - {date}

### 核心发现
{key_findings}

### 热点话题TOP10
{top_topics}

### 情感分析
{sentiment_breakdown}

### 传播趋势
{trend_chart}
"""

report = analyzer.generate_custom_report(
    data=results,
    template=template,
    format='markdown'  # or 'html', 'pdf'
)
```

## Project Structure

```
├── app.py                          # Main web application
├── hotsearchcrawler/               # Spider cluster
│   ├── spiders/                    # Platform-specific spiders
│   │   ├── weibo_spider.py
│   │   ├── bilibili_spider.py
│   │   └── toutiao_spider.py
│   ├── settings.py                 # Crawler configuration
│   └── db_handler.py               # Database operations
├── hotsearch_analysis_agent/       # Analysis system
│   ├── analyzer.py                 # LLM analysis engine
│   ├── llm_client.py               # LLM API wrapper
│   ├── data_service.py             # Data retrieval service
│   └── push_manager.py             # Push notification manager
├── run_spiders.py                  # Crawler launcher
└── test_push_task.py               # Push notification tester
```

This skill enables AI agents to help developers deploy and operate a production-grade public opinion analytics system with minimal configuration.
