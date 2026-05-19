---
name: llm-public-opinion-analytics-assistant
description: Deploy and use the LLM-based public opinion analytics assistant with 15 platforms, 26 rankings, multi-channel alerting, and sentiment analysis
triggers:
  - how do I set up the public opinion analytics assistant
  - configure hotsearch crawler for multiple platforms
  - analyze hot topics with LLM sentiment analysis
  - set up multi-channel push notifications for trending topics
  - deploy Chinese social media monitoring system
  - use Pangu model for opinion analytics
  - crawl Weibo Bilibili Douyin trending rankings
  - create public opinion monitoring dashboard
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is an intelligent public opinion analytics assistant that integrates real-time data from **15 mainstream platforms** across **26 ranking lists** with LLM analysis capabilities. It provides conversational hot search queries, topic clustering, sentiment analysis, and multi-channel alerting (email, WeChat, Enterprise WeChat, Telegram).

## What It Does

- **Real-time Data Collection**: Crawls hot search rankings from Weibo, Bilibili, Douyin, Zhihu, and 11+ other Chinese platforms
- **LLM-Powered Analysis**: Topic clustering, sentiment analysis, and trend detection using Pangu or OpenAI-compatible models
- **Multi-Channel Alerts**: Push hot topic reports via email, WeChat Work, Telegram, or Enterprise WeChat apps
- **Video Content Analysis**: Extracts insights even from video-based news content
- **Interactive Dashboard**: Chat-based interface for querying trends and analyzing topics

## Installation

### Prerequisites

1. **Browser Driver Setup** (for detailed content scraping):
   ```bash
   # Check your Chrome/Edge version first
   # Download matching driver from:
   # Chrome: https://chromedriver.chromium.org/
   # Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/
   
   # Place driver in PATH or project directory
   # Verify installation:
   chromedriver --version
   ```

2. **MySQL Database**:
   ```sql
   CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
   ```

3. **Python Environment**:
   ```bash
   git clone https://github.com/hmmnxkl/LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant.git
   cd LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant
   python -m venv venv
   source venv/bin/activate  # On Windows: venv\Scripts\activate
   pip install -r requirements.txt
   ```

### Configuration

1. **Create `.env` file in project root**:
   ```env
   # Database
   MYSQL_HOST=localhost
   MYSQL_PORT=3306
   MYSQL_USER=root
   MYSQL_PASSWORD=your_password
   MYSQL_DATABASE=hotsearch_db
   
   # LLM Model (OpenAI-compatible)
   OPENAI_API_KEY=your_api_key
   OPENAI_API_BASE=https://api.openai.com/v1
   MODEL_NAME=gpt-4
   
   # Or use Pangu model (local deployment)
   # PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model
   
   # Push Notifications (optional)
   SMTP_HOST=smtp.gmail.com
   SMTP_PORT=587
   SMTP_USER=your_email@gmail.com
   SMTP_PASSWORD=your_app_password
   
   WECHAT_WORK_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxx
   TELEGRAM_BOT_TOKEN=your_bot_token
   TELEGRAM_CHAT_ID=your_chat_id
   ```

2. **Crawler Settings** (`hotsearchcrawler/settings.py`):
   ```python
   # MySQL connection for crawler
   MYSQL_SETTINGS = {
       'host': 'localhost',
       'port': 3306,
       'user': 'root',
       'password': 'your_password',
       'database': 'hotsearch_db'
   }
   
   # Optional: Add cookies for authenticated platforms
   COOKIES = {
       'weibo': 'your_weibo_cookie',
       'bilibili': 'your_bilibili_cookie'
   }
   ```

3. **Initialize Database**:
   ```python
   # Run init.py to create tables
   python init.py
   ```

## Key Components

### 1. Crawler System (`hotsearchcrawler/`)

Start crawler via web interface or directly:

```python
# Direct crawler test
python runspider-test.py

# Or start all crawlers
python run_spiders.py
```

**Supported Platforms**:
- Social: Weibo (3 lists), Zhihu, Douban
- Video: Bilibili (4 lists), Douyin, Kuaishou
- News: Toutiao, Baidu, 36Kr, IT Home
- E-commerce: Taobao, JD
- Others: NetEase News, Sina News, etc.

### 2. Analysis System (`hotsearch_analysis_agent/`)

```python
from hotsearch_analysis_agent.analyzer import OpinionAnalyzer

# Initialize analyzer
analyzer = OpinionAnalyzer(
    model_name="gpt-4",  # or "pangu"
    api_key=os.getenv("OPENAI_API_KEY")
)

# Query hot topics
results = analyzer.query_topics(
    query="人工智能",  # AI
    platforms=["weibo", "bilibili", "zhihu"],
    time_range="24h"
)

# Perform sentiment analysis
sentiment = analyzer.analyze_sentiment(results)
print(f"Positive: {sentiment['positive']}, Negative: {sentiment['negative']}")

# Topic clustering
clusters = analyzer.cluster_topics(results, num_clusters=5)
for cluster in clusters:
    print(f"Cluster: {cluster['keywords']}")
    print(f"Articles: {len(cluster['articles'])}")
```

### 3. Web Application

```bash
# Start the web interface
python app.py

# Access at http://localhost:5000
# Use chat interface to query trends, analyze topics
```

**Web Interface Features**:
- Conversational topic queries
- Real-time hot search display
- Sentiment visualization
- One-click jump to original content
- Crawler start/stop controls (keyboard shortcuts)

### 4. Push Notification System

```python
from hotsearch_analysis_agent.push_service import PushService

# Initialize push service
pusher = PushService()

# Create push task
task = {
    "query": "人工智能",
    "frequency": "daily",  # or "hourly", "weekly"
    "channels": ["email", "wechat_work", "telegram"],
    "filters": {
        "min_heat": 1000,
        "sentiment": "all"  # or "positive", "negative"
    }
}

# Register task
pusher.create_task(task)

# Test push manually
python test_push_task.py
```

**Push Report Format**:
- Executive summary with key findings
- Detailed news items with URLs
- Data visualization (trending charts)
- Sentiment distribution analysis
- Related topic clustering

## Common Patterns

### Pattern 1: Real-time Monitoring Dashboard

```python
from hotsearch_analysis_agent import RealTimeMonitor

monitor = RealTimeMonitor(
    platforms=["weibo", "bilibili", "douyin"],
    refresh_interval=300  # 5 minutes
)

# Start monitoring
monitor.start()

# Get current hot topics
topics = monitor.get_top_topics(limit=20)
for topic in topics:
    print(f"{topic['platform']}: {topic['title']} (热度: {topic['heat']})")
```

### Pattern 2: Competitor Monitoring

```python
# Monitor specific keywords across platforms
keywords = ["竞争对手A", "竞争对手B"]

analyzer = OpinionAnalyzer()
results = analyzer.monitor_keywords(
    keywords=keywords,
    platforms="all",
    alert_threshold=5000  # Alert when heat > 5000
)

# Get sentiment breakdown
for keyword in keywords:
    sentiment = results[keyword]['sentiment']
    print(f"{keyword}: {sentiment['score']}/10")
```

### Pattern 3: Crisis Detection

```python
# Detect negative sentiment spikes
crisis_detector = analyzer.detect_crisis(
    brand_keywords=["公司名"],
    sentiment_threshold=-0.6,
    heat_threshold=10000,
    time_window="6h"
)

if crisis_detector.is_crisis:
    # Trigger immediate alert
    pusher.send_alert(
        priority="high",
        message=crisis_detector.summary,
        channels=["telegram", "wechat_work"]
    )
```

### Pattern 4: Using Pangu Model (Local)

```python
# Configure Pangu model for local deployment
from hotsearch_analysis_agent.models import PanguModel

pangu = PanguModel(
    model_path="/path/to/openpangu-embedded-7b-model",
    device="cuda"  # or "cpu"
)

# Use for analysis (same interface as OpenAI)
analysis = pangu.analyze_text(
    text="长文本新闻内容...",
    task="sentiment"  # or "summary", "clustering"
)
```

## Configuration Files

### Crawler Platform Configuration

```python
# hotsearchcrawler/spiders/weibo_spider.py
class WeiboHotSearchSpider(scrapy.Spider):
    name = 'weibo_hot'
    
    custom_settings = {
        'DOWNLOAD_DELAY': 2,
        'COOKIES_ENABLED': True,
        'USER_AGENT': 'Mozilla/5.0...'
    }
    
    def parse(self, response):
        # Extract hot search items
        for item in response.css('.hot-item'):
            yield {
                'title': item.css('.title::text').get(),
                'heat': item.css('.heat::text').get(),
                'url': item.css('a::attr(href)').get(),
                'platform': 'weibo',
                'timestamp': datetime.now()
            }
```

### Analysis Agent Configuration

```python
# hotsearch_analysis_agent/config.py
ANALYSIS_CONFIG = {
    'clustering': {
        'algorithm': 'kmeans',
        'min_similarity': 0.7,
        'max_clusters': 10
    },
    'sentiment': {
        'model': 'bert-base-chinese',
        'threshold': 0.5
    },
    'summarization': {
        'max_length': 200,
        'min_length': 50
    }
}
```

## Troubleshooting

### Issue: Crawler fails with 403 errors

```python
# Add retry middleware in settings.py
RETRY_TIMES = 5
RETRY_HTTP_CODES = [403, 500, 502, 503, 504]

# Rotate user agents
USER_AGENT_LIST = [
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
    'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36',
]

# Enable proxy pool (if needed)
PROXIES = [
    'http://proxy1:8000',
    'http://proxy2:8000'
]
```

### Issue: LLM analysis timeout

```python
# Adjust timeout settings
OPENAI_TIMEOUT = 120  # seconds

# Use streaming for long responses
response = analyzer.analyze_stream(text, task="summary")
for chunk in response:
    print(chunk, end='')
```

### Issue: Database connection pool exhausted

```python
# Increase pool size in settings
MYSQL_POOL_SIZE = 20
MYSQL_MAX_OVERFLOW = 10
MYSQL_POOL_RECYCLE = 3600
```

### Issue: Push notifications not working

```python
# Test each channel individually
python test_push_task.py --channel email
python test_push_task.py --channel wechat_work
python test_push_task.py --channel telegram

# Check credentials in .env
# Verify webhook URLs are accessible
# Check firewall/proxy settings
```

### Issue: Video content extraction fails

```python
# Ensure browser driver is installed and in PATH
import subprocess
result = subprocess.run(['chromedriver', '--version'], capture_output=True)
print(result.stdout)

# Enable headless mode for stability
CHROME_OPTIONS = [
    '--headless',
    '--no-sandbox',
    '--disable-dev-shm-usage'
]
```

## Advanced Usage

### Custom Platform Spider

```python
# hotsearchcrawler/spiders/custom_spider.py
import scrapy
from hotsearchcrawler.items import HotSearchItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://example.com/hot']
    
    def parse(self, response):
        for item in response.css('.trending-item'):
            yield HotSearchItem(
                title=item.css('.title::text').get(),
                url=item.css('a::attr(href)').get(),
                heat=self.parse_heat(item.css('.score::text').get()),
                platform='custom',
                category=item.css('.category::text').get()
            )
    
    def parse_heat(self, heat_str):
        # Convert "1.2万" to 12000
        if '万' in heat_str:
            return int(float(heat_str.replace('万', '')) * 10000)
        return int(heat_str)
```

### Custom Analysis Task

```python
from hotsearch_analysis_agent.base import BaseAnalyzer

class TrendPredictor(BaseAnalyzer):
    def predict_trend(self, topic, historical_data):
        # Time series analysis
        trend = self.model.predict(
            data=historical_data,
            horizon=24  # predict next 24 hours
        )
        
        return {
            'topic': topic,
            'prediction': trend,
            'confidence': trend['confidence_score'],
            'peak_time': trend['expected_peak']
        }
```

This skill enables AI coding agents to help developers deploy, configure, and use the LLM-based public opinion analytics assistant for comprehensive social media monitoring and analysis across Chinese platforms.
