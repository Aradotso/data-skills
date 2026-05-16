---
name: llm-public-opinion-analytics-assistant
description: Multi-platform real-time public opinion analytics assistant with web scraping, LLM analysis, clustering, sentiment analysis, and multi-channel alerting
triggers:
  - set up public opinion monitoring system
  - analyze hot search trends across platforms
  - scrape news from multiple social platforms
  - configure sentiment analysis with LLM
  - create topic clustering from social media
  - send public opinion alerts to wechat telegram
  - deploy chinese social media monitoring
  - integrate pangu model for text analysis
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is an intelligent public opinion analytics assistant that combines real-time data from **26 trending lists across 15 mainstream platforms** with large language model (LLM) analysis capabilities. It provides conversational hot search queries, topic-specific searches, topic clustering analysis, and sentiment analysis. The system supports crawler control via shortcuts, multi-platform data querying with direct navigation, and multi-channel alerting (email, WeChat, Enterprise WeChat, Telegram) based on accumulated analysis results from news detail pages (including video content extraction).

**Key Features:**
- Real-time data collection from 15+ Chinese platforms (Weibo, Bilibili, Toutiao, etc.)
- LLM-powered analysis using Pangu model or OpenAI-compatible APIs
- Topic clustering and sentiment analysis
- Multi-channel push notifications (Enterprise WeChat, Telegram, Email)
- Web-based conversational interface
- Separate crawler cluster architecture

## Installation

### Prerequisites

1. **Python 3.8+** with virtual environment
2. **MySQL database** (5.7+ or 8.0+)
3. **Browser driver** (Chrome/Edge) for Selenium-based crawlers
4. **LLM API access** (Huawei Pangu model or OpenAI-compatible endpoint)

### Step 1: Browser Driver Setup

Download the appropriate driver for your browser:

- **Chrome**: https://chromedriver.chromium.org/
- **Edge**: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

Place the driver in your system PATH or in the project directory. Verify installation:

```bash
chromedriver --version
# or
msedgedriver --version
```

### Step 2: Clone and Install Dependencies

```bash
git clone https://github.com/hmmnxkl/LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant.git
cd LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Step 3: Database Setup

Create MySQL database and tables:

```python
# Reference init.py for table schemas
import pymysql

connection = pymysql.connect(
    host='localhost',
    user='your_user',
    password='your_password',
    database='hotsearch_db',
    charset='utf8mb4'
)

# Run SQL from init.py to create tables:
# - hot_search_items (trending topics)
# - news_details (detailed content)
# - push_tasks (notification configs)
# - analysis_results (LLM outputs)
```

### Step 4: Environment Configuration

Create `.env` file in the project root:

```env
# Database Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_mysql_user
MYSQL_PASSWORD=your_mysql_password
MYSQL_DATABASE=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key_here
OPENAI_BASE_URL=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Or use Pangu model (local deployment)
# PANGU_MODEL_PATH=/path/to/pangu/model
# PANGU_API_URL=http://localhost:8000/v1

# Push Notification Configs
ENTERPRISE_WECHAT_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY
TELEGRAM_BOT_TOKEN=your_telegram_bot_token
TELEGRAM_CHAT_ID=your_chat_id
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_email_password
```

### Step 5: Crawler Configuration

Edit `hotsearchcrawler/settings.py`:

```python
# MySQL connection for crawler
MYSQL_CONFIG = {
    'host': 'localhost',
    'user': 'your_user',
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

## Usage

### Starting the Application

```bash
# Start the web interface and analysis system
python app.py
```

Access the web UI at `http://localhost:5000`

### Running Crawlers

**Option 1: Via Web Interface**
- Use the web UI to start/stop crawlers with shortcut keys
- Monitor crawler status in real-time

**Option 2: Command Line**

```bash
# Test individual crawler
python runspider-test.py --platform weibo

# Run all crawlers
python run_spiders.py
```

### Conversational Query Examples

Through the web interface, you can ask:

```
"Show me today's Weibo hot searches"
"What are people saying about AI technology?"
"Analyze sentiment on recent tech news"
"Find trending topics related to artificial intelligence"
"Cluster topics from the last 24 hours"
```

### Programmatic Analysis API

```python
from hotsearch_analysis_agent.analyzer import OpinionAnalyzer
from hotsearch_analysis_agent.models import HotSearchQuery

# Initialize analyzer
analyzer = OpinionAnalyzer(
    model_name=os.getenv('MODEL_NAME'),
    api_key=os.getenv('OPENAI_API_KEY'),
    base_url=os.getenv('OPENAI_BASE_URL')
)

# Query hot searches
query = HotSearchQuery(
    keywords=["人工智能", "AI"],
    platforms=["weibo", "bilibili", "toutiao"],
    time_range="24h"
)

results = analyzer.search_topics(query)

# Perform sentiment analysis
sentiment = analyzer.analyze_sentiment(results)
print(f"Positive: {sentiment['positive_ratio']:.2%}")
print(f"Negative: {sentiment['negative_ratio']:.2%}")

# Topic clustering
clusters = analyzer.cluster_topics(results, num_clusters=5)
for i, cluster in enumerate(clusters):
    print(f"Cluster {i}: {cluster['keywords']}")
    print(f"  Items: {len(cluster['items'])}")
```

### Setting Up Push Notifications

```python
from test_push_task import create_push_task

# Create a push task for AI/tech monitoring
task = create_push_task(
    task_name="AI Tech Monitor",
    keywords=["人工智能", "大模型", "深度学习"],
    channels=["enterprise_wechat", "telegram", "email"],
    frequency="daily",  # or "hourly", "realtime"
    threshold={"热度": 10000}  # Only push if heat > 10000
)

# The task will automatically generate reports like the example in README
```

### Working with Pangu Model (Local Deployment)

```python
from hotsearch_analysis_agent.llm_client import PanguClient

# Initialize Pangu client
pangu = PanguClient(
    model_path="/path/to/openpangu-embedded-7b-model",
    device="cuda"  # or "cpu"
)

# Analyze long-form content
news_content = """
[Long news article or social media post in Chinese]
"""

analysis = pangu.analyze_text(
    text=news_content,
    task="sentiment_and_summary",
    max_length=512
)

print(f"Sentiment: {analysis['sentiment']}")
print(f"Summary: {analysis['summary']}")
print(f"Key entities: {analysis['entities']}")
```

## Key Components

### 1. Crawler System (`hotsearchcrawler/`)

Scrapy-based crawlers for 15 platforms:

```python
# Example: Custom spider for a new platform
from hotsearchcrawler.base_spider import BaseHotSearchSpider

class NewPlatformSpider(BaseHotSearchSpider):
    name = 'new_platform'
    start_urls = ['https://platform.com/trending']
    
    def parse(self, response):
        for item in response.css('.trending-item'):
            yield {
                'title': item.css('.title::text').get(),
                'heat': item.css('.heat::text').get(),
                'url': item.css('a::attr(href)').get(),
                'platform': self.name,
                'rank': item.css('.rank::text').get()
            }
```

### 2. Analysis Agent (`hotsearch_analysis_agent/`)

LLM-powered analysis engine:

```python
from hotsearch_analysis_agent.clustering import TopicClusterer
from hotsearch_analysis_agent.sentiment import SentimentAnalyzer

# Topic clustering with embeddings
clusterer = TopicClusterer(embedding_model='text-embedding-ada-002')
topics = clusterer.fit_transform(news_items, n_clusters=5)

# Sentiment analysis
sentiment_analyzer = SentimentAnalyzer(llm_client=pangu)
sentiments = sentiment_analyzer.batch_analyze([
    item['title'] + ' ' + item['content'] 
    for item in news_items
])
```

### 3. Database Schema

Key tables:

```sql
-- Hot search items
CREATE TABLE hot_search_items (
    id INT AUTO_INCREMENT PRIMARY KEY,
    platform VARCHAR(50),
    title VARCHAR(500),
    heat BIGINT,
    rank INT,
    url TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_platform_time (platform, created_at)
);

-- Detailed news content
CREATE TABLE news_details (
    id INT AUTO_INCREMENT PRIMARY KEY,
    hot_search_id INT,
    content TEXT,
    video_transcript TEXT,  -- For video content
    images JSON,
    author VARCHAR(200),
    publish_time TIMESTAMP,
    FOREIGN KEY (hot_search_id) REFERENCES hot_search_items(id)
);

-- Push tasks
CREATE TABLE push_tasks (
    id INT AUTO_INCREMENT PRIMARY KEY,
    task_name VARCHAR(200),
    keywords JSON,
    channels JSON,
    frequency VARCHAR(20),
    filters JSON,
    is_active BOOLEAN DEFAULT TRUE,
    last_run TIMESTAMP
);
```

## Common Patterns

### Pattern 1: Real-time Monitoring with Alerts

```python
from hotsearch_analysis_agent.monitor import RealTimeMonitor
from hotsearch_analysis_agent.notifier import MultiChannelNotifier

# Setup monitor
monitor = RealTimeMonitor(
    keywords=["重大事件", "突发新闻"],
    platforms=["weibo", "toutiao"],
    check_interval=300  # 5 minutes
)

# Setup notifier
notifier = MultiChannelNotifier(
    channels={
        'enterprise_wechat': os.getenv('ENTERPRISE_WECHAT_WEBHOOK'),
        'telegram': {
            'bot_token': os.getenv('TELEGRAM_BOT_TOKEN'),
            'chat_id': os.getenv('TELEGRAM_CHAT_ID')
        }
    }
)

# Start monitoring
@monitor.on_match
def handle_match(items):
    report = analyzer.generate_report(items)
    notifier.send(report, channels=['enterprise_wechat', 'telegram'])

monitor.start()
```

### Pattern 2: Batch Historical Analysis

```python
from datetime import datetime, timedelta
from hotsearch_analysis_agent.database import Database

db = Database()

# Query historical data
start_date = datetime.now() - timedelta(days=7)
items = db.query_hot_searches(
    platforms=['weibo', 'bilibili'],
    start_time=start_date,
    keywords=['科技', '人工智能']
)

# Generate trend analysis
trend = analyzer.analyze_trend(items, group_by='day')

# Visualize (if using plotting library)
import matplotlib.pyplot as plt
plt.plot(trend['dates'], trend['volumes'])
plt.title('AI Topic Trend - Last 7 Days')
plt.savefig('trend.png')
```

### Pattern 3: Custom Report Generation

```python
def generate_custom_report(keywords, time_range='24h'):
    """Generate formatted report like the example in README"""
    
    # Fetch data
    items = analyzer.search_topics(
        HotSearchQuery(keywords=keywords, time_range=time_range)
    )
    
    # Cluster topics
    clusters = analyzer.cluster_topics(items, num_clusters=4)
    
    # Analyze sentiment
    sentiments = analyzer.analyze_sentiment(items)
    
    # Format report
    report = f"""
## Report - {', '.join(keywords)} Analysis
**Time**: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}

### 🔍 Core Findings

"""
    
    for i, cluster in enumerate(clusters, 1):
        report += f"🔹 **{cluster['theme']}**: {cluster['summary']}\n\n"
    
    report += "### 📰 Detailed News\n\n"
    
    for item in items[:5]:  # Top 5
        report += f"#### {item['title']}\n"
        report += f"**URL**: [{item['platform']}]({item['url']})\n"
        report += f"**Summary**: {item['summary']}\n\n"
    
    report += f"### 📊 Sentiment Distribution\n\n"
    report += f"✅ Positive: {sentiments['positive_ratio']:.1%}\n"
    report += f"⚠️ Neutral: {sentiments['neutral_ratio']:.1%}\n"
    report += f"❌ Negative: {sentiments['negative_ratio']:.1%}\n"
    
    return report
```

## Troubleshooting

### Crawler Issues

**Problem**: Selenium driver not found
```bash
# Solution: Verify driver in PATH
which chromedriver  # Linux/Mac
where chromedriver  # Windows

# Or set explicit path in settings.py
CHROME_DRIVER_PATH = '/usr/local/bin/chromedriver'
```

**Problem**: Platform blocking/rate limiting
```python
# Add delays and rotate user agents in settings.py
DOWNLOAD_DELAY = 2
RANDOMIZE_DOWNLOAD_DELAY = True
USER_AGENT_LIST = [
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64)...',
    'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)...'
]
```

### Database Connection Issues

```python
# Test connection
import pymysql

try:
    conn = pymysql.connect(**MYSQL_CONFIG)
    print("Connection successful")
except Exception as e:
    print(f"Error: {e}")
    # Check: MySQL running? Credentials correct? Database exists?
```

### LLM API Errors

```python
# Implement retry logic
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=4, max=10))
def call_llm(prompt):
    response = llm_client.chat(prompt)
    return response

# Handle rate limits
import time

def safe_analyze_batch(items, batch_size=10):
    results = []
    for i in range(0, len(items), batch_size):
        batch = items[i:i+batch_size]
        results.extend(analyzer.analyze(batch))
        time.sleep(1)  # Rate limiting
    return results
```

### Memory Issues with Large Datasets

```python
# Use generators and batch processing
def process_large_dataset(query, batch_size=100):
    offset = 0
    while True:
        batch = db.query_paginated(query, limit=batch_size, offset=offset)
        if not batch:
            break
        
        yield analyzer.process_batch(batch)
        offset += batch_size

# Clear cache periodically
import gc
gc.collect()
```

### Character Encoding Issues

```python
# Ensure UTF-8 encoding throughout
# In MySQL connection:
MYSQL_CONFIG = {
    'charset': 'utf8mb4',
    'use_unicode': True
}

# In file operations:
with open('report.txt', 'w', encoding='utf-8') as f:
    f.write(report)
```

## Advanced Configuration

### Custom Platform Integration

```python
# Add new platform in hotsearchcrawler/spiders/
from hotsearchcrawler.base_spider import BaseHotSearchSpider

class CustomPlatformSpider(BaseHotSearchSpider):
    name = 'custom_platform'
    
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.api_endpoint = 'https://api.platform.com/trending'
    
    def start_requests(self):
        yield scrapy.Request(
            self.api_endpoint,
            callback=self.parse_api,
            headers={'Authorization': f'Bearer {self.api_token}'}
        )
    
    def parse_api(self, response):
        data = response.json()
        for item in data['trending']:
            yield self.create_item(
                title=item['title'],
                heat=item['engagement'],
                url=item['link']
            )
```

### Fine-tuning Analysis Prompts

```python
# Customize analysis prompts in hotsearch_analysis_agent/prompts.py
SENTIMENT_PROMPT = """
分析以下中文新闻的情感倾向。考虑以下因素:
1. 明确的情感词汇
2. 隐含的语气和态度
3. 讽刺或反讽表达

新闻内容: {content}

请返回JSON格式: {{"sentiment": "positive/neutral/negative", "confidence": 0-1, "reasoning": "..."}}
"""

CLUSTERING_PROMPT = """
将以下新闻标题聚类为{num_clusters}个主题。每个主题提供:
- 主题名称
- 核心关键词
- 简短描述

标题列表: {titles}
"""
```

This skill provides comprehensive coverage of the LLM-Based Intelligent Public Opinion Analytics Assistant for AI coding agents to effectively help developers deploy, configure, and utilize the system for Chinese social media monitoring and analysis.
