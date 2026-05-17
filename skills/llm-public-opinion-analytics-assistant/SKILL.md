---
name: llm-public-opinion-analytics-assistant
description: Build real-time public opinion analytics systems with multi-platform hot topic crawling, LLM-powered sentiment analysis, and multi-channel alert distribution
triggers:
  - analyze public opinion from social media platforms
  - crawl hot topics from weibo and bilibili
  - set up sentiment analysis for trending topics
  - create public opinion monitoring dashboard
  - send alerts via wechat telegram for trending news
  - cluster and analyze social media trends
  - build a hot topic tracking system
  - deploy opinion analytics with pangu llm
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is a comprehensive public opinion analytics assistant that integrates real-time data from 15 mainstream platforms (26 hot lists) with large language model analysis capabilities. It enables conversational hot search queries, topic clustering, sentiment analysis, and multi-channel alert distribution (email, WeChat, Enterprise WeChat, Telegram).

## What It Does

- **Multi-Platform Data Crawling**: Scrapes hot topics from 15 platforms including Weibo, Bilibili, Douyin, Zhihu, etc.
- **LLM-Powered Analysis**: Uses Huawei Pangu or OpenAI-compatible models for sentiment analysis, topic clustering, and trend identification
- **Conversational Interface**: Natural language queries for hot search data and analytics
- **Content Extraction**: Extracts detailed content from news pages, including video-based content
- **Multi-Channel Alerts**: Push notifications via Enterprise WeChat, Telegram, and email
- **Real-Time Control**: Keyboard shortcuts to start/stop crawlers and query data

## Project Structure

```
.
├── hotsearch_analysis_agent/    # Analysis system with LLM integration
├── hotsearchcrawler/            # Scrapy-based crawler cluster
├── app.py                       # Main application entry point
├── run_spiders.py              # Crawler launcher
├── test_push_task/             # Push notification testing
└── init.py                     # Database initialization reference
```

## Installation

### Prerequisites

1. **Python Environment**
```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

2. **Browser Driver Setup**
   
Download ChromeDriver or EdgeDriver matching your browser version:
- Chrome: https://chromedriver.chromium.org/
- Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

Place driver in system PATH or project directory.

Verify installation:
```bash
chromedriver --version
```

3. **MySQL Database**
```sql
-- Create database
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Example table structure (see init.py for complete schema)
CREATE TABLE hot_topics (
    id INT AUTO_INCREMENT PRIMARY KEY,
    platform VARCHAR(50),
    title VARCHAR(500),
    url TEXT,
    rank INT,
    heat_value VARCHAR(100),
    fetch_time DATETIME,
    content TEXT,
    sentiment VARCHAR(50),
    INDEX idx_platform (platform),
    INDEX idx_fetch_time (fetch_time)
);
```

### Configuration

1. **Crawler Settings** (`hotsearchcrawler/settings.py`):
```python
# MySQL connection
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_user'
MYSQL_PASSWORD = 'your_password'
MYSQL_DATABASE = 'hotsearch_db'

# Optional: Platform cookies for authenticated access
COOKIES = {
    'weibo': 'your_cookie_string',
    # Add other platforms as needed
}
```

2. **Analysis System** (`.env` file in project root):
```bash
# Database
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DB=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_BASE_URL=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Or use Huawei Pangu model
# PANGU_API_KEY=your_pangu_key
# PANGU_BASE_URL=https://pangu-api-endpoint

# Push Notification Channels
# Enterprise WeChat
WECHAT_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Telegram
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
```

## Usage

### Starting the System

```bash
# Initialize database
python init.py

# Start the main application (web interface)
python app.py
```

The web interface will be available at `http://localhost:5000` (default).

### Running Crawlers

```python
# Start specific platform crawler
from hotsearchcrawler import run_spider

# Crawl Weibo hot topics
run_spider('weibo')

# Crawl Bilibili trending videos
run_spider('bilibili')

# Run all crawlers
python run_spiders.py --all

# Run crawlers for specific platforms
python run_spiders.py --platforms weibo,bilibili,zhihu
```

### Programmatic Usage

#### Query Hot Topics

```python
from hotsearch_analysis_agent.data_access import get_hot_topics

# Get latest hot topics from all platforms
topics = get_hot_topics(limit=50)

# Filter by platform
weibo_topics = get_hot_topics(platform='weibo', limit=20)

# Filter by time range
from datetime import datetime, timedelta
recent_topics = get_hot_topics(
    start_time=datetime.now() - timedelta(hours=24),
    end_time=datetime.now()
)

for topic in topics:
    print(f"{topic['platform']}: {topic['title']} (Rank: {topic['rank']})")
```

#### Sentiment Analysis

```python
from hotsearch_analysis_agent.analyzer import analyze_sentiment

# Analyze sentiment of a topic
topic_content = "用户对新政策的讨论内容..."
sentiment = analyze_sentiment(topic_content)

# Returns: {'sentiment': 'positive', 'score': 0.85, 'keywords': [...]}
print(f"Sentiment: {sentiment['sentiment']}, Score: {sentiment['score']}")
```

#### Topic Clustering

```python
from hotsearch_analysis_agent.clustering import cluster_topics

# Cluster similar topics
topics_data = get_hot_topics(limit=100)
clusters = cluster_topics(topics_data)

# clusters is a list of topic groups
for i, cluster in enumerate(clusters):
    print(f"Cluster {i+1}:")
    for topic in cluster['topics']:
        print(f"  - {topic['title']}")
    print(f"  Theme: {cluster['theme']}")
```

#### Generate Analysis Report

```python
from hotsearch_analysis_agent.report import generate_report

# Generate comprehensive analysis report
report = generate_report(
    query="人工智能与前沿科技",
    time_range=24,  # Last 24 hours
    platforms=['weibo', 'zhihu', 'bilibili']
)

print(report['markdown'])  # Full markdown report
print(report['summary'])   # Executive summary
```

### Setting Up Push Notifications

```python
from hotsearch_analysis_agent.push import PushTaskManager

# Create push task manager
push_mgr = PushTaskManager()

# Create a push task
task = push_mgr.create_task(
    name="AI Tech Monitoring",
    query="人工智能",
    platforms=['weibo', 'zhihu'],
    schedule="0 */6 * * *",  # Every 6 hours (cron format)
    channels=['wechat', 'telegram', 'email'],
    threshold={'heat_value': 1000000}  # Only topics with heat > 1M
)

# Start the task
push_mgr.start_task(task.id)

# List all active tasks
active_tasks = push_mgr.list_tasks(status='active')
```

### Web Interface Interactions

The frontend provides conversational queries:

```javascript
// Example: Natural language query through web API
fetch('/api/query', {
  method: 'POST',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({
    query: "最近关于人工智能的热点有哪些?",
    platforms: ["weibo", "zhihu"],
    analysis_type: "sentiment"
  })
})
.then(res => res.json())
.then(data => {
  console.log(data.topics);
  console.log(data.analysis);
});
```

## Common Patterns

### Daily Hot Topic Monitoring

```python
from hotsearch_analysis_agent import HotSearchAgent
from datetime import datetime

agent = HotSearchAgent()

# Monitor daily hot topics
def daily_monitor():
    topics = agent.get_trending_topics(
        platforms=['weibo', 'zhihu', 'douyin'],
        min_heat=500000,
        time_range=24
    )
    
    # Cluster related topics
    clusters = agent.cluster_topics(topics)
    
    # Analyze sentiment for each cluster
    for cluster in clusters:
        sentiment = agent.analyze_cluster_sentiment(cluster)
        
        if sentiment['score'] < -0.5:  # Negative sentiment alert
            agent.send_alert(
                title=f"Negative Sentiment Detected: {cluster['theme']}",
                content=cluster,
                channels=['wechat', 'email']
            )
    
    return clusters

# Schedule daily execution
from apscheduler.schedulers.blocking import BlockingScheduler
scheduler = BlockingScheduler()
scheduler.add_job(daily_monitor, 'cron', hour=9, minute=0)
scheduler.start()
```

### Custom Platform Crawler

```python
# Add custom platform crawler in hotsearchcrawler/spiders/
import scrapy
from hotsearchcrawler.items import HotTopicItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://custom-platform.com/hot']
    
    def parse(self, response):
        for item in response.css('.hot-item'):
            yield HotTopicItem(
                platform='custom_platform',
                title=item.css('.title::text').get(),
                url=item.css('a::attr(href)').get(),
                rank=item.css('.rank::text').get(),
                heat_value=item.css('.heat::text').get(),
                fetch_time=datetime.now()
            )
```

### LLM-Powered Content Extraction

```python
from hotsearch_analysis_agent.content_extractor import extract_content

# Extract detailed content from URL (including video content)
url = "https://www.bilibili.com/video/BV13pSoBBEvX"
content = extract_content(url, use_llm=True)

# content includes:
# - text: Extracted text content
# - video_transcript: Transcribed audio (for video content)
# - summary: LLM-generated summary
# - entities: Named entities extracted
# - sentiment: Sentiment analysis result

print(f"Summary: {content['summary']}")
print(f"Key entities: {content['entities']}")
```

## Troubleshooting

### Driver Not Found Error

```
selenium.common.exceptions.WebDriverException: Message: 'chromedriver' executable needs to be in PATH.
```

**Solution**: Ensure ChromeDriver/EdgeDriver is in system PATH or specify explicit path:

```python
from selenium import webdriver
from selenium.webdriver.chrome.service import Service

service = Service('/path/to/chromedriver')
driver = webdriver.Chrome(service=service)
```

### Database Connection Error

```
pymysql.err.OperationalError: (2003, "Can't connect to MySQL server")
```

**Solution**: Verify MySQL is running and credentials in `.env` are correct:

```bash
# Test connection
mysql -h localhost -u your_user -p hotsearch_db
```

### LLM API Rate Limiting

```
openai.error.RateLimitError: Rate limit exceeded
```

**Solution**: Implement retry logic with exponential backoff:

```python
from tenacity import retry, wait_exponential, stop_after_attempt

@retry(wait=wait_exponential(min=1, max=60), stop=stop_after_attempt(5))
def call_llm_api(prompt):
    # Your LLM API call
    return client.chat.completions.create(...)
```

### Crawler Getting Blocked

**Solution**: Add delays and user agents in `hotsearchcrawler/settings.py`:

```python
DOWNLOAD_DELAY = 3
RANDOMIZE_DOWNLOAD_DELAY = True

USER_AGENT = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'

# Use rotating proxies if needed
DOWNLOADER_MIDDLEWARES = {
    'scrapy.downloadermiddlewares.useragent.UserAgentMiddleware': None,
    'scrapy_user_agents.middlewares.RandomUserAgentMiddleware': 400,
}
```

### Memory Issues with Large Datasets

**Solution**: Use batching and pagination:

```python
from hotsearch_analysis_agent.data_access import get_hot_topics_batch

# Process in batches
batch_size = 1000
offset = 0

while True:
    batch = get_hot_topics_batch(limit=batch_size, offset=offset)
    if not batch:
        break
    
    # Process batch
    for topic in batch:
        process_topic(topic)
    
    offset += batch_size
```

## Advanced Features

### Custom Analysis Prompts

```python
from hotsearch_analysis_agent.llm_client import LLMClient

client = LLMClient()

# Custom analysis prompt
custom_prompt = """
Analyze the following hot topics and identify:
1. Emerging trends
2. Potential risks
3. Recommended actions

Topics: {topics}
"""

result = client.analyze(
    prompt=custom_prompt,
    topics=topics_data,
    temperature=0.3
)
```

### Multi-Language Support

```python
# Configure language for analysis
from hotsearch_analysis_agent.config import set_language

set_language('en')  # English analysis output
topics = get_hot_topics(platform='twitter')
analysis = analyze_sentiment(topics, language='en')
```

This skill enables AI agents to help developers build comprehensive public opinion monitoring and analysis systems with minimal configuration.
