---
name: llm-public-opinion-analytics-assistant
description: Multi-platform public opinion analysis assistant with web scraping, LLM-based sentiment analysis, topic clustering, and multi-channel alert pushing (WeChat, Telegram, Email)
triggers:
  - set up public opinion monitoring system
  - analyze hot topics from multiple platforms
  - configure sentiment analysis with LLM
  - scrape trending news from social media
  - create automated alert push notifications
  - build Chinese news aggregation crawler
  - implement topic clustering analysis
  - monitor social media trends with AI
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive public opinion monitoring system that aggregates trending topics from 26 ranking lists across 15 mainstream Chinese platforms. Features LLM-powered sentiment analysis, topic clustering, conversational querying, and multi-channel push notifications (Enterprise WeChat, Telegram, Email).

## What This Project Does

- **Multi-Platform Data Collection**: Scrapy-based crawler cluster for Weibo, Bilibili, Baidu, Toutiao, and 11+ other platforms
- **LLM Analysis**: Sentiment analysis, topic clustering, and natural language querying using Huawei Pangu or OpenAI-compatible models
- **Conversational Interface**: Chat-based UI for querying hot topics, analyzing trends, and generating reports
- **Video Content Extraction**: Extracts information even from video-based news sources
- **Multi-Channel Alerts**: Push analysis reports via Enterprise WeChat, Telegram, or email with scheduled tasks
- **Real-Time Control**: Keyboard shortcuts to start/stop crawlers, quick platform data lookup

## Installation

### Prerequisites

**1. Browser Driver Setup** (Required for news detail scraping)

Download the appropriate driver for your browser:
- **Chrome**: https://chromedriver.chromium.org/
- **Edge**: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

Place driver in system PATH or browser installation directory.

Verify installation:
```bash
chromedriver --version  # or msedgedriver --version
```

**2. Database Setup**

```bash
# Install MySQL and create database
mysql -u root -p
CREATE DATABASE public_opinion CHARACTER SET utf8mb4;
```

**3. Python Environment**

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

**4. Database Initialization**

```python
# Run init.py to create tables
from hotsearch_analysis_agent.database import create_tables
create_tables()
```

## Configuration

### Environment Variables (.env)

Create `.env` file in project root:

```bash
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=public_opinion

# LLM Configuration (OpenAI-compatible API)
LLM_API_BASE=https://api.openai.com/v1
LLM_API_KEY=your_api_key_here
LLM_MODEL=gpt-4

# Or Huawei Pangu Model (local deployment)
# LLM_API_BASE=http://localhost:8000/v1
# LLM_MODEL=pangu-embedded-7b

# Push Notification Settings
# Enterprise WeChat Robot
WECHAT_WEBHOOK_URL=your_webhook_url

# Enterprise WeChat App (push to personal WeChat)
WECHAT_CORP_ID=your_corp_id
WECHAT_AGENT_ID=your_agent_id
WECHAT_SECRET=your_secret

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
SMTP_RECIPIENT=recipient@example.com
```

### Crawler Settings (hotsearchcrawler/settings.py)

```python
# MySQL connection for crawler
MYSQL_CONFIG = {
    'host': 'localhost',
    'port': 3306,
    'user': 'your_user',
    'password': 'your_password',
    'database': 'public_opinion',
    'charset': 'utf8mb4'
}

# Optional: Platform-specific cookies
COOKIES = {
    'weibo': 'your_weibo_cookie_string',
    'bilibili': 'your_bilibili_cookie_string'
}

# Concurrent requests
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
```

## Running the System

### Start Web Application

```bash
# Launch main application
python app.py
```

Access web interface at `http://localhost:5000`

### Control Crawlers

**Via Web Interface**: Use keyboard shortcuts or UI controls to start/stop crawlers

**Manual Crawler Test**:

```bash
# Test individual spider
cd hotsearchcrawler
python runspider-test.py --spider weibo

# Run all spiders
python run_spiders.py
```

### Test Push Notifications

```bash
# Test notification channels
python test_push_task.py
```

## Core API Usage

### Database Operations

```python
from hotsearch_analysis_agent.database import Database

# Initialize database connection
db = Database()

# Query hot topics from specific platform
topics = db.query(
    "SELECT * FROM hot_topics WHERE platform = %s ORDER BY rank LIMIT 10",
    ("weibo",)
)

# Insert crawled data
db.execute(
    """INSERT INTO hot_topics 
       (platform, title, rank, heat_value, url, crawled_at) 
       VALUES (%s, %s, %s, %s, %s, NOW())""",
    ("weibo", "AI技术突破", 1, 1250000, "https://weibo.com/...")
)

# Close connection
db.close()
```

### LLM Analysis

```python
from hotsearch_analysis_agent.llm_analyzer import LLMAnalyzer

# Initialize analyzer
analyzer = LLMAnalyzer(
    api_base=os.getenv("LLM_API_BASE"),
    api_key=os.getenv("LLM_API_KEY"),
    model=os.getenv("LLM_MODEL")
)

# Sentiment analysis
sentiment = analyzer.analyze_sentiment(
    "华为发布新款AI芯片,性能大幅提升,国产算力迎来突破"
)
# Returns: {"sentiment": "positive", "score": 0.85, "keywords": ["AI芯片", "突破"]}

# Topic clustering
topics_list = [
    "GPT-6遭提前曝光,2M超长上下文来了",
    "DeepSeek V4采用华为算力",
    "中国大模型周调用量连续五周超越美国"
]
clusters = analyzer.cluster_topics(topics_list)
# Returns: [{"cluster": "AI模型发展", "topics": [...], "summary": "..."}]

# Generate analysis report
report = analyzer.generate_report(
    query="人工智能与前沿科技",
    topics=topics_list,
    detail_contents=["content1", "content2"]
)
```

### Web Scraping (Crawler Spiders)

```python
# Example spider structure (hotsearchcrawler/spiders/weibo_spider.py)
import scrapy
from hotsearchcrawler.items import HotTopicItem

class WeiboSpider(scrapy.Spider):
    name = 'weibo'
    start_urls = ['https://s.weibo.com/top/summary']
    
    def parse(self, response):
        for item in response.css('.td-02'):
            topic = HotTopicItem()
            topic['platform'] = 'weibo'
            topic['title'] = item.css('a::text').get()
            topic['url'] = response.urljoin(item.css('a::attr(href)').get())
            topic['rank'] = item.css('.td-01::text').get()
            topic['heat_value'] = item.css('.td-03 span::text').get()
            yield topic
            
            # Follow link for detail content
            yield scrapy.Request(
                topic['url'],
                callback=self.parse_detail,
                meta={'item': topic}
            )
    
    def parse_detail(self, response):
        item = response.meta['item']
        item['content'] = response.css('.WB_detail::text').getall()
        yield item
```

### Push Notifications

```python
from hotsearch_analysis_agent.push_service import PushService

# Initialize push service
push = PushService()

# Send Enterprise WeChat notification
push.send_wechat_robot(
    webhook_url=os.getenv("WECHAT_WEBHOOK_URL"),
    message="## 热点分析报告\n\n今日舆情摘要...",
    markdown=True
)

# Send Telegram message
push.send_telegram(
    bot_token=os.getenv("TELEGRAM_BOT_TOKEN"),
    chat_id=os.getenv("TELEGRAM_CHAT_ID"),
    message="📊 热点分析报告\n\n..."
)

# Send email report
push.send_email(
    subject="每日舆情分析 - 2026-05-21",
    body="<h1>热点分析报告</h1><p>...</p>",
    html=True
)

# Schedule periodic push task
from apscheduler.schedulers.background import BackgroundScheduler

scheduler = BackgroundScheduler()
scheduler.add_job(
    func=push.send_daily_report,
    trigger='cron',
    hour=12,
    minute=30,
    args=["人工智能", "科技"]
)
scheduler.start()
```

## Common Usage Patterns

### Pattern 1: Automated Daily Report

```python
from hotsearch_analysis_agent.analyzer import OpinionAnalyzer

# Create analyzer instance
analyzer = OpinionAnalyzer()

# Query hot topics from last 24 hours
hot_topics = analyzer.get_hot_topics(
    platforms=["weibo", "bilibili", "zhihu"],
    hours=24,
    min_rank=10
)

# Perform clustering analysis
clusters = analyzer.cluster_and_analyze(hot_topics)

# Generate markdown report
report = analyzer.generate_markdown_report(
    title="关于人工智能与前沿科技的热点分析",
    clusters=clusters,
    timestamp="2026-05-21 12:30:00"
)

# Push to all channels
analyzer.push_to_all_channels(report)
```

### Pattern 2: Keyword-Based Monitoring

```python
# Monitor specific keywords
keywords = ["AI芯片", "大模型", "算力"]

# Search across all platforms
results = analyzer.search_topics(
    keywords=keywords,
    date_range=("2026-05-01", "2026-05-21")
)

# Analyze sentiment distribution
sentiment_stats = analyzer.analyze_sentiment_distribution(results)

# Alert if negative sentiment exceeds threshold
if sentiment_stats['negative_ratio'] > 0.3:
    push.send_alert(
        "⚠️ 负面舆情预警",
        f"关键词 '{keywords}' 负面情绪比例达到 {sentiment_stats['negative_ratio']:.1%}"
    )
```

### Pattern 3: Custom Spider Integration

```python
# Add new platform spider (hotsearchcrawler/spiders/custom_spider.py)
class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    
    custom_settings = {
        'ITEM_PIPELINES': {
            'hotsearchcrawler.pipelines.MySQLPipeline': 300,
        }
    }
    
    def start_requests(self):
        yield scrapy.Request(
            'https://example.com/hot',
            headers={'User-Agent': 'Mozilla/5.0...'}
        )
    
    def parse(self, response):
        # Extract data using CSS or XPath
        for item in response.css('.hot-item'):
            yield HotTopicItem(
                platform='custom_platform',
                title=item.css('.title::text').get(),
                url=item.css('a::attr(href)').get(),
                rank=item.css('.rank::text').get()
            )
```

## Troubleshooting

### Crawler Issues

**Problem**: ChromeDriver version mismatch
```bash
# Check Chrome version
google-chrome --version  # or microsoft-edge --version

# Download matching driver version
# Update PATH to include driver location
```

**Problem**: Anti-crawler blocking

```python
# Add cookies in settings.py
COOKIES = {
    'platform_name': 'your_cookie_string'
}

# Adjust request delay
DOWNLOAD_DELAY = 2
RANDOMIZE_DOWNLOAD_DELAY = True
```

### Database Connection Errors

```python
# Test connection
from hotsearch_analysis_agent.database import Database

try:
    db = Database()
    result = db.query("SELECT 1")
    print("Database connection successful")
except Exception as e:
    print(f"Connection failed: {e}")
    # Check .env file settings
```

### LLM API Issues

**Problem**: API timeout or rate limiting

```python
# Add retry logic
import time
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=4, max=10))
def call_llm_with_retry(prompt):
    return analyzer.analyze_sentiment(prompt)
```

**Problem**: Context length exceeded

```python
# Truncate input text
MAX_TOKENS = 4000

def truncate_text(text, max_tokens=MAX_TOKENS):
    # Simple character-based truncation (adjust based on tokenizer)
    max_chars = max_tokens * 4  # Rough estimate for Chinese
    return text[:max_chars] if len(text) > max_chars else text

truncated_content = truncate_text(long_article)
analysis = analyzer.analyze_sentiment(truncated_content)
```

### Push Notification Failures

```bash
# Test each channel individually
python test_push_task.py --channel wechat
python test_push_task.py --channel telegram
python test_push_task.py --channel email

# Check webhook URLs and tokens in .env file
# Verify network connectivity to external services
```

## Advanced Configuration

### Using Huawei Pangu Model (Local Deployment)

```bash
# Download model from GitCode
wget https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

# Start local inference server (example with vLLM)
python -m vllm.entrypoints.openai.api_server \
    --model /path/to/openpangu-embedded-7b-model \
    --host 0.0.0.0 \
    --port 8000

# Update .env
LLM_API_BASE=http://localhost:8000/v1
LLM_MODEL=openpangu-embedded-7b
```

### Distributed Crawler Deployment

```python
# Configure Scrapy with Redis scheduler
# Install scrapy-redis
pip install scrapy-redis

# Update hotsearchcrawler/settings.py
SCHEDULER = "scrapy_redis.scheduler.Scheduler"
DUPEFILTER_CLASS = "scrapy_redis.dupefilter.RFPDupeFilter"
REDIS_URL = 'redis://localhost:6379'

# Run multiple crawler instances
# Instance 1
python run_spiders.py --spider weibo

# Instance 2
python run_spiders.py --spider bilibili
```

This skill enables AI coding agents to help developers deploy, configure, and extend this comprehensive Chinese public opinion monitoring system with LLM-powered analysis capabilities.
