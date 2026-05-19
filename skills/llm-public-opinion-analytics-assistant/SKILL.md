---
name: llm-public-opinion-analytics-assistant
description: Multi-platform hot search data crawler and LLM-powered sentiment analysis system with real-time monitoring and multi-channel alerting
triggers:
  - how do I set up the public opinion analytics assistant
  - crawl hot search data from multiple platforms
  - analyze sentiment trends with LLM models
  - configure push notifications for trending topics
  - set up hotsearch crawler for weibo douyin bilibili
  - deploy the opinion analytics system with pangu model
  - analyze and cluster trending topics using AI
  - configure multi-channel alerts for hot topics
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive public opinion monitoring system that crawls hot search data from 15 mainstream platforms (26 total ranking lists) and provides LLM-powered analysis including sentiment analysis, topic clustering, and multi-channel alerting via email, WeChat, Enterprise WeChat, and Telegram.

## What This Project Does

- **Multi-Platform Crawling**: Scrapes trending topics from Weibo, Douyin, Bilibili, Baidu, Zhihu, and 10+ other platforms
- **LLM Analysis**: Uses Huawei Pangu or OpenAI-compatible models for sentiment analysis, topic clustering, and trend identification
- **Interactive Query**: Natural language interface for searching and analyzing trending topics
- **Detail Extraction**: Extracts content from news detail pages (including video transcripts)
- **Multi-Channel Alerts**: Push notifications via Enterprise WeChat, Telegram, email (SMTP)
- **Hotkey Control**: Start/stop crawlers via keyboard shortcuts in the frontend

## Installation

### Prerequisites

```bash
# Python 3.8+
# MySQL 5.7+
# Chrome/Edge browser with matching WebDriver
```

### Browser Driver Setup

1. **Check browser version**:
   - Chrome: Settings → About → Note version (e.g., `115.0.5790.102`)
   - Edge: Settings → About → Note version

2. **Download matching driver**:
   - Chrome: https://chromedriver.chromium.org/
   - Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

3. **Install driver**:
   ```bash
   # Linux/macOS
   sudo mv chromedriver /usr/local/bin/
   sudo chmod +x /usr/local/bin/chromedriver
   
   # Windows - add to PATH or place in C:\Windows\System32\
   ```

4. **Verify installation**:
   ```bash
   chromedriver --version
   ```

### Project Setup

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

### Database Configuration

```python
# Reference init.py for database schema
import pymysql

connection = pymysql.connect(
    host='localhost',
    user='your_user',
    password='your_password',
    database='hotsearch_db',
    charset='utf8mb4'
)

# Create required tables (see init.py for full schema)
cursor = connection.cursor()
cursor.execute("""
CREATE TABLE IF NOT EXISTS hot_search (
    id INT AUTO_INCREMENT PRIMARY KEY,
    platform VARCHAR(50),
    title VARCHAR(500),
    url TEXT,
    rank_position INT,
    heat_value VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)
""")
connection.commit()
```

## Configuration

### Environment Variables (.env)

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

# Or use Huawei Pangu (local deployment)
USE_PANGU=true
PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model

# Push Notification Channels
# Enterprise WeChat Bot
WECHAT_WEBHOOK_URL=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
SMTP_TO=recipient@example.com
```

### Crawler Settings (hotsearchcrawler/settings.py)

```python
# MySQL Configuration
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_user'
MYSQL_PASSWORD = 'your_password'
MYSQL_DATABASE = 'hotsearch_db'

# Optional: Platform-specific cookies (for authenticated scraping)
COOKIES = {
    'weibo': 'your_weibo_cookies',
    'bilibili': 'your_bilibili_cookies',
}

# Crawler Settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
RETRY_TIMES = 3
```

## Key Commands

### Start the Application

```bash
# Start main web interface
python app.py

# Access at http://localhost:5000
```

### Manual Crawler Control

```bash
# Test single crawler
python runspider-test.py

# Run all crawlers (normally triggered via web UI)
python run_spiders.py
```

### Test Push Notifications

```bash
# Test all configured push channels
python test_push_task.py
```

## Core API Usage

### Query Hot Searches

```python
from hotsearch_analysis_agent.api import search_hot_topics

# Natural language query
results = search_hot_topics(
    query="人工智能相关的新闻",
    platforms=["weibo", "douyin", "bilibili"],
    limit=20
)

for item in results:
    print(f"{item['platform']}: {item['title']} (热度: {item['heat_value']})")
```

### Sentiment Analysis

```python
from hotsearch_analysis_agent.analyzer import analyze_sentiment

# Analyze a topic's sentiment
topic = "GPT-6提前曝光"
sentiment = analyze_sentiment(topic)

print(f"情感倾向: {sentiment['polarity']}")  # positive/negative/neutral
print(f"情感得分: {sentiment['score']}")      # -1.0 to 1.0
print(f"关键词: {sentiment['keywords']}")
```

### Topic Clustering

```python
from hotsearch_analysis_agent.clustering import cluster_topics

# Group related topics
topics = [
    "GPT-6遭提前曝光",
    "DeepSeek V4采用华为算力",
    "Anthropic收入超越OpenAI",
    "国产大模型调用量领先"
]

clusters = cluster_topics(topics)

for cluster_id, cluster_topics in clusters.items():
    print(f"聚类 {cluster_id}:")
    for t in cluster_topics:
        print(f"  - {t}")
```

### Configure Push Task

```python
from hotsearch_analysis_agent.push import create_push_task

# Create scheduled push task
task = create_push_task(
    name="AI技术热点推送",
    query_keywords=["人工智能", "大模型", "AI"],
    platforms=["weibo", "36kr", "iheima"],
    channels=["wechat", "telegram", "email"],
    schedule="0 12 * * *",  # Daily at 12:00
    min_heat_threshold=10000
)

task.save()
```

## Common Patterns

### Complete Analysis Workflow

```python
from hotsearch_analysis_agent import (
    search_hot_topics,
    fetch_detail_content,
    analyze_sentiment,
    cluster_topics,
    generate_report,
    push_to_channels
)

# 1. Search topics
topics = search_hot_topics(
    query="前沿科技",
    platforms=["weibo", "douyin", "bilibili", "36kr"],
    days_back=1
)

# 2. Fetch detail content (including video transcripts)
enriched_topics = []
for topic in topics:
    detail = fetch_detail_content(topic['url'])
    topic['content'] = detail['text']
    topic['sentiment'] = analyze_sentiment(detail['text'])
    enriched_topics.append(topic)

# 3. Cluster related topics
clusters = cluster_topics(enriched_topics)

# 4. Generate analysis report
report = generate_report(
    clusters=clusters,
    analysis_type="trend_analysis",
    template="tech_focus"
)

# 5. Push to configured channels
push_to_channels(
    content=report,
    channels=["wechat", "telegram", "email"],
    priority="high"
)
```

### Custom LLM Integration

```python
from hotsearch_analysis_agent.llm import LLMClient

# Initialize with custom model
llm = LLMClient(
    api_base="http://localhost:8000/v1",
    api_key="local-key",
    model="pangu-7b"
)

# Analyze with custom prompt
analysis = llm.chat(
    messages=[
        {"role": "system", "content": "你是舆情分析专家"},
        {"role": "user", "content": f"分析以下话题的情感倾向和潜在影响:\n{topic_text}"}
    ],
    temperature=0.3
)

print(analysis['choices'][0]['message']['content'])
```

### Platform-Specific Scraping

```python
from hotsearchcrawler.spiders import WeiboSpider, DouyinSpider

# Weibo hot search
weibo_spider = WeiboSpider()
weibo_results = weibo_spider.parse_hot_search()

# Douyin trending
douyin_spider = DouyinSpider()
douyin_results = douyin_spider.parse_trending()

# Save to database
for result in weibo_results + douyin_results:
    save_to_db(result)
```

### Advanced Filtering

```python
from hotsearch_analysis_agent.filters import (
    filter_by_heat,
    filter_by_keywords,
    filter_by_sentiment,
    deduplicate
)

# Multi-stage filtering
topics = search_hot_topics(platforms=["all"])

# Filter by heat value
hot_topics = filter_by_heat(topics, min_heat=50000)

# Filter by keywords
relevant = filter_by_keywords(
    hot_topics,
    include_keywords=["科技", "AI", "芯片"],
    exclude_keywords=["娱乐", "八卦"]
)

# Filter by sentiment
positive_topics = filter_by_sentiment(
    relevant,
    sentiment_range=["positive", "neutral"]
)

# Remove duplicates
unique_topics = deduplicate(
    positive_topics,
    similarity_threshold=0.85
)
```

## Troubleshooting

### WebDriver Issues

```bash
# Error: chromedriver executable needs to be in PATH
# Solution: Verify driver location
which chromedriver  # Linux/macOS
where chromedriver  # Windows

# Update PATH if needed
export PATH=$PATH:/path/to/driver/directory
```

### Database Connection Errors

```python
# Test connection
import pymysql

try:
    conn = pymysql.connect(
        host=MYSQL_HOST,
        port=MYSQL_PORT,
        user=MYSQL_USER,
        password=MYSQL_PASSWORD,
        database=MYSQL_DATABASE
    )
    print("✓ Database connected")
except Exception as e:
    print(f"✗ Connection failed: {e}")
```

### Crawler Blocking

```python
# Add retry logic and user agents
from hotsearchcrawler.middlewares import RetryMiddleware

# In settings.py
DOWNLOADER_MIDDLEWARES = {
    'hotsearchcrawler.middlewares.RetryMiddleware': 550,
    'scrapy.downloadermiddlewares.useragent.UserAgentMiddleware': None,
    'scrapy_user_agents.middlewares.RandomUserAgentMiddleware': 400,
}

RETRY_TIMES = 5
RETRY_HTTP_CODES = [500, 502, 503, 504, 408, 429]
```

### LLM API Rate Limits

```python
from hotsearch_analysis_agent.llm import RateLimiter

# Add rate limiting
limiter = RateLimiter(max_calls=60, period=60)  # 60 calls/minute

@limiter.limit
def analyze_with_llm(text):
    return llm.chat([{"role": "user", "content": text}])
```

### Push Notification Failures

```python
# Test individual channels
from hotsearch_analysis_agent.push import test_channel

# Test WeChat webhook
test_channel("wechat", message="测试消息")

# Test Telegram
test_channel("telegram", message="Test message")

# Test SMTP
test_channel("email", subject="Test", body="Test email")

# Check logs for detailed error messages
tail -f logs/push_service.log
```

### Memory Issues with Large Datasets

```python
# Use batch processing
from hotsearch_analysis_agent.utils import batch_process

topics = search_hot_topics(limit=10000)

# Process in batches
for batch in batch_process(topics, batch_size=100):
    results = analyze_sentiment_batch(batch)
    save_results(results)
    # Clear memory
    del results
```

## Performance Optimization

```python
# Enable caching for repeated queries
from hotsearch_analysis_agent.cache import enable_cache

enable_cache(
    backend="redis",
    host="localhost",
    port=6379,
    ttl=3600  # 1 hour
)

# Parallel processing for multiple platforms
from concurrent.futures import ThreadPoolExecutor

platforms = ["weibo", "douyin", "bilibili", "zhihu"]

with ThreadPoolExecutor(max_workers=4) as executor:
    futures = [
        executor.submit(search_hot_topics, platforms=[p])
        for p in platforms
    ]
    results = [f.result() for f in futures]
```
