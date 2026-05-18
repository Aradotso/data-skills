---
name: llm-public-opinion-analytics-assistant
description: LLM-powered multi-platform hot search crawler and sentiment analysis assistant with push notifications
triggers:
  - "analyze public opinion from Chinese social media platforms"
  - "crawl hot search rankings from Weibo, Bilibili, and other platforms"
  - "set up sentiment analysis on trending topics"
  - "configure multi-channel push notifications for trending news"
  - "cluster and analyze trending topics with LLM"
  - "scrape and analyze video content from social platforms"
  - "deploy public opinion monitoring system"
  - "query hot search data with natural language"
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

LLM-Based Intelligent Public Opinion Analytics Assistant is a comprehensive Chinese social media monitoring system that:

- **Crawls 26 hot search rankings** from 15 mainstream Chinese platforms (Weibo, Bilibili, Douyin, Zhihu, etc.)
- **Analyzes sentiment and trends** using LLM (supports Huawei PanGu model and OpenAI-compatible APIs)
- **Clusters related topics** across platforms for unified insight
- **Extracts content from news detail pages** including video transcriptions
- **Pushes reports** via email, WeChat Work, Enterprise WeChat, and Telegram
- **Provides conversational interface** for querying hot search data
- **Supports keyboard shortcuts** for crawler control

The project has two main components:
- **Crawler cluster** (`hotsearchcrawler`): Scrapy-based distributed crawlers
- **Analysis system** (`hotsearch_analysis_agent`): LLM-powered analytics and API

## Installation

### Prerequisites

1. **Python 3.8+**
2. **MySQL 5.7+**
3. **Browser Driver** (Chrome/Edge WebDriver)

### Browser Driver Setup

**Step 1: Check browser version**
```bash
# Chrome
google-chrome --version
# Edge
microsoft-edge --version
```

**Step 2: Download matching driver**
- Chrome: https://chromedriver.chromium.org/
- Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

**Step 3: Place driver in PATH**
```bash
# Linux/macOS
sudo mv chromedriver /usr/local/bin/
sudo chmod +x /usr/local/bin/chromedriver

# Verify
chromedriver --version
```

### Install Dependencies

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
# Create MySQL database
mysql -u root -p
```

```sql
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

Use `init.py` as reference for table schemas:

```python
# Initialize database tables
python init.py
```

## Configuration

### 1. Crawler Configuration

Edit `hotsearchcrawler/settings.py`:

```python
# MySQL connection
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'root'
MYSQL_PASSWORD = 'your_password'
MYSQL_DB = 'hotsearch_db'

# Optional: Platform cookies for authenticated crawling
COOKIES_WEIBO = 'your_weibo_cookies'
COOKIES_BILIBILI = 'your_bilibili_cookies'
```

### 2. Analysis System Configuration

Create `.env` file in project root:

```bash
# Database
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=your_password
MYSQL_DB=hotsearch_db

# LLM API (OpenAI-compatible format)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Or use local Huawei PanGu model
# MODEL_TYPE=pangu
# PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model

# Push notification channels (optional)
# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
EMAIL_TO=recipient@example.com

# WeChat Work Robot
WECHAT_WORK_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=your_key

# Enterprise WeChat App
ENTERPRISE_WECHAT_CORP_ID=your_corp_id
ENTERPRISE_WECHAT_AGENT_ID=your_agent_id
ENTERPRISE_WECHAT_SECRET=your_secret

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

## Running the System

### Start the Analysis System (Web UI + API)

```bash
# Main application
python app.py
```

The web interface will be available at `http://localhost:5000`

### Run Crawlers

**Option 1: Via Web UI**
- Access the web interface
- Use keyboard shortcuts or buttons to start/stop crawlers
- Monitor crawler status in real-time

**Option 2: Command Line**

```bash
# Test single crawler
python runspider-test.py

# Run all crawlers
python run_spiders.py
```

**Option 3: Run specific platform crawler**

```bash
cd hotsearchcrawler
scrapy crawl weibo_spider
scrapy crawl bilibili_spider
scrapy crawl douyin_spider
```

## Key Features and API Usage

### 1. Conversational Hot Search Query

```python
# Natural language query via chat interface
# Example queries:
# - "What's trending on Weibo today?"
# - "Show me technology-related hot searches"
# - "Analyze sentiment for topic: AI development"
# - "Cluster topics about electric vehicles"
```

### 2. Topic Clustering Analysis

```python
from hotsearch_analysis_agent.clustering import TopicClusterer

clusterer = TopicClusterer(model_name="gpt-4")

# Fetch hot search data
hot_searches = fetch_from_db("SELECT * FROM hot_searches WHERE date = CURDATE()")

# Cluster related topics
clusters = clusterer.cluster_topics(hot_searches)

# Output example:
# {
#   "cluster_1": {
#     "theme": "人工智能技术突破",
#     "items": [{"title": "GPT-6曝光", "platform": "bilibili"}, ...]
#   }
# }
```

### 3. Sentiment Analysis

```python
from hotsearch_analysis_agent.sentiment import SentimentAnalyzer

analyzer = SentimentAnalyzer()

# Analyze single topic
result = analyzer.analyze_topic("DeepSeek V4采用华为算力")

# Output:
# {
#   "sentiment": "positive",
#   "score": 0.85,
#   "keywords": ["国产", "算力", "突破"],
#   "summary": "舆论整体正面,体现国产技术进步..."
# }
```

### 4. Detail Page Content Extraction

```python
from hotsearchcrawler.detail_extractor import DetailExtractor

extractor = DetailExtractor()

# Extract from news page (including video content)
content = extractor.extract("https://www.bilibili.com/video/BV13pSoBBEvX/")

# Returns:
# {
#   "title": "GPT-6遭提前曝光, 2M超长上下文来了",
#   "content": "视频转录文本...",
#   "publish_time": "2026-04-07 10:30:00",
#   "author": "...",
#   "tags": [...]
# }
```

### 5. Push Notification Tasks

```python
from hotsearch_analysis_agent.push import PushTaskManager

manager = PushTaskManager()

# Create scheduled push task
task = manager.create_task(
    name="AI Tech Weekly Report",
    keywords=["人工智能", "前沿科技"],
    channels=["email", "wechat_work", "telegram"],
    schedule="0 9 * * 1",  # Every Monday 9 AM
    template="weekly_report"
)

# Test push immediately
manager.test_push(task_id=task.id)
```

**Test push tasks:**

```bash
# Test all configured channels
python test_push_task.py
```

## Common Patterns

### Pattern 1: Daily Hot Search Monitoring

```python
import schedule
import time
from hotsearchcrawler.scheduler import CrawlerScheduler
from hotsearch_analysis_agent.analyzer import DailyReportGenerator

def daily_monitor():
    # Run all crawlers
    scheduler = CrawlerScheduler()
    scheduler.run_all_spiders()
    
    # Wait for data collection
    time.sleep(300)  # 5 minutes
    
    # Generate daily report
    generator = DailyReportGenerator()
    report = generator.generate(date="today")
    
    # Push to channels
    push_manager.send_report(report, channels=["email", "telegram"])

# Schedule daily at 8 AM
schedule.every().day.at("08:00").do(daily_monitor)

while True:
    schedule.run_pending()
    time.sleep(60)
```

### Pattern 2: Keyword-Based Alerting

```python
from hotsearch_analysis_agent.monitor import KeywordMonitor

monitor = KeywordMonitor(
    keywords=["企业名称", "品牌危机", "负面舆情"],
    sentiment_threshold=-0.3,  # Alert on negative sentiment
    platforms=["weibo", "zhihu", "douyin"]
)

# Continuous monitoring
while True:
    alerts = monitor.check_keywords()
    
    if alerts:
        for alert in alerts:
            # Send immediate notification
            push_manager.send_alert(
                title=f"负面舆情预警: {alert['keyword']}",
                content=alert['summary'],
                channels=["wechat_work", "telegram"],
                priority="high"
            )
    
    time.sleep(600)  # Check every 10 minutes
```

### Pattern 3: Multi-Platform Topic Aggregation

```python
from hotsearch_analysis_agent.aggregator import CrossPlatformAggregator

aggregator = CrossPlatformAggregator()

# Get unified view of topic across platforms
topic_analysis = aggregator.aggregate_topic(
    keyword="新能源汽车",
    platforms=["weibo", "bilibili", "toutiao", "douyin"],
    days=7
)

# Output structure:
# {
#   "keyword": "新能源汽车",
#   "total_mentions": 1523,
#   "platform_distribution": {"weibo": 450, "bilibili": 320, ...},
#   "sentiment_trend": [0.2, 0.3, 0.5, ...],
#   "hot_sub_topics": ["特斯拉降价", "比亚迪销量", ...],
#   "summary": "LLM生成的综合分析..."
# }
```

### Pattern 4: Using Local PanGu Model

```python
# Update .env
# MODEL_TYPE=pangu
# PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model

from hotsearch_analysis_agent.llm import PanguAnalyzer

analyzer = PanguAnalyzer(model_path=os.getenv("PANGU_MODEL_PATH"))

# Analyze with local model (no API calls)
result = analyzer.analyze_sentiment(
    text="DeepSeek V4采用华为算力，国产芯片生态走到哪一步了？",
    context="科技新闻情感分析"
)
```

## Database Schema Reference

### Main Tables

**hot_searches** (Hot search entries)
```sql
CREATE TABLE hot_searches (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    platform VARCHAR(50),
    rank INT,
    title VARCHAR(500),
    url VARCHAR(1000),
    heat_value VARCHAR(100),
    crawled_at DATETIME,
    INDEX idx_platform_date (platform, crawled_at)
);
```

**topic_analysis** (Analysis results)
```sql
CREATE TABLE topic_analysis (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    topic_keyword VARCHAR(200),
    sentiment_score FLOAT,
    cluster_id VARCHAR(100),
    summary TEXT,
    analyzed_at DATETIME
);
```

**push_tasks** (Push notification tasks)
```sql
CREATE TABLE push_tasks (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    task_name VARCHAR(200),
    keywords JSON,
    channels JSON,
    schedule_cron VARCHAR(100),
    last_run DATETIME,
    status VARCHAR(20)
);
```

## Troubleshooting

### Issue: Crawler fails with "WebDriver not found"

**Solution:**
```bash
# Verify driver is in PATH
which chromedriver  # Linux/macOS
where chromedriver  # Windows

# Or set explicit path in settings.py
CHROME_DRIVER_PATH = '/usr/local/bin/chromedriver'
```

### Issue: MySQL connection refused

**Solution:**
```bash
# Check MySQL is running
sudo systemctl status mysql

# Verify credentials
mysql -u root -p -e "SHOW DATABASES;"

# Check .env file matches MySQL config
cat .env | grep MYSQL
```

### Issue: LLM API timeout or rate limit

**Solution:**
```python
# Add retry logic in .env
LLM_MAX_RETRIES=3
LLM_TIMEOUT=60

# Or switch to local PanGu model
MODEL_TYPE=pangu
PANGU_MODEL_PATH=/path/to/model
```

### Issue: Video content extraction fails

**Solution:**
```bash
# Ensure ffmpeg is installed (for video processing)
sudo apt-get install ffmpeg  # Ubuntu
brew install ffmpeg          # macOS

# Check browser driver supports headless mode
# Edit hotsearchcrawler/settings.py
HEADLESS_BROWSER = True
```

### Issue: Push notifications not working

**Solution:**
```bash
# Test individual channels
python test_push_task.py --channel email
python test_push_task.py --channel wechat_work

# Verify webhook URLs are accessible
curl -X POST $WECHAT_WORK_WEBHOOK -d '{"msgtype":"text","text":{"content":"test"}}'

# Check SMTP credentials for email
python -c "import smtplib; smtplib.SMTP('$SMTP_HOST', $SMTP_PORT).connect()"
```

### Issue: Out of memory during clustering

**Solution:**
```python
# Reduce batch size in clustering config
# hotsearch_analysis_agent/clustering.py
CLUSTER_BATCH_SIZE = 100  # Default: 500

# Or enable incremental clustering
INCREMENTAL_CLUSTERING = True
```

## Advanced Configuration

### Custom Spider for New Platform

```python
# hotsearchcrawler/spiders/custom_platform.py
import scrapy

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://example.com/hot']
    
    def parse(self, response):
        for item in response.css('.hot-item'):
            yield {
                'platform': 'custom_platform',
                'rank': item.css('.rank::text').get(),
                'title': item.css('.title::text').get(),
                'url': item.css('a::attr(href)').get(),
                'heat_value': item.css('.heat::text').get()
            }
```

### Custom Analysis Template

```python
# hotsearch_analysis_agent/templates/custom_report.py
from jinja2 import Template

CUSTOM_TEMPLATE = Template("""
## {{ title }}
**Date**: {{ date }}

### Top Trends
{% for trend in trends %}
- **{{ trend.title }}** ({{ trend.platform }})
  Sentiment: {{ trend.sentiment }}
  {{ trend.summary }}
{% endfor %}

### Analysis
{{ analysis }}
""")

# Use in report generation
report = CUSTOM_TEMPLATE.render(
    title="Custom Report",
    date="2026-05-17",
    trends=trend_data,
    analysis=llm_analysis
)
```

This skill enables AI coding agents to help developers deploy, configure, and extend the LLM-Based Intelligent Public Opinion Analytics Assistant for comprehensive Chinese social media monitoring and sentiment analysis.
