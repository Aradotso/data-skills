---
name: llm-public-opinion-analytics-assistant
description: Multi-platform real-time public opinion analysis assistant with web crawlers, LLM-powered sentiment analysis, topic clustering, and multi-channel alerts
triggers:
  - set up public opinion monitoring system
  - analyze hot topics from social media platforms
  - create sentiment analysis for trending news
  - configure multi-platform web crawler for hot searches
  - send hot topic alerts via email or telegram
  - cluster and analyze trending topics with LLM
  - scrape real-time data from weibo douyin bilibili
  - build AI-powered news sentiment dashboard
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is an intelligent public opinion analysis assistant that combines real-time data from **15 mainstream platforms** (26 trending lists) with large language model analysis capabilities. It provides conversational hot search queries, topic clustering, sentiment analysis, and multi-channel alert notifications (Email, WeChat, Enterprise WeChat, Telegram).

**Key Features:**
- Multi-platform web crawlers (Weibo, Douyin, Bilibili, Zhihu, etc.)
- LLM-powered content analysis (supports Huawei Pangu, OpenAI-compatible models)
- Sentiment analysis and topic clustering
- Multi-channel push notifications
- Web dashboard with natural language query interface
- Automated video content extraction

## Installation

### Prerequisites

**1. Browser Driver Setup (Required for content extraction)**

Download the appropriate driver for your browser:
- **Chrome**: [ChromeDriver](https://chromedriver.chromium.org/)
- **Edge**: [EdgeDriver](https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/)

Place the driver in your system PATH or browser installation directory:
```bash
# Linux/macOS - add to PATH
export PATH=$PATH:/path/to/driver/directory

# Verify installation
chromedriver --version
```

**2. MySQL Database**

Install MySQL and create the database:
```sql
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### Install Dependencies

```bash
# Clone the repository
git clone https://github.com/hmmnxkl/LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant.git
cd LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant

# Create virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install requirements
pip install -r requirements.txt
```

### Database Initialization

Run the initialization script to create tables:
```bash
python init.py
```

## Configuration

### 1. Environment Variables (`.env`)

Create a `.env` file in the project root:

```env
# Database Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_username
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Alternative: Huawei Pangu Model (local deployment)
# PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model

# Push Notification Channels
# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
SMTP_RECIPIENTS=recipient1@example.com,recipient2@example.com

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Enterprise WeChat
WECHAT_WEBHOOK_URL=your_webhook_url
WECHAT_APP_ID=your_app_id
WECHAT_APP_SECRET=your_app_secret
```

### 2. Crawler Configuration (`hotsearchcrawler/settings.py`)

```python
# MySQL settings for crawler
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_username'
MYSQL_PASSWORD = 'your_password'
MYSQL_DATABASE = 'hotsearch_db'

# Optional: Platform-specific cookies
COOKIES = {
    'weibo': 'your_weibo_cookie',
    'douyin': 'your_douyin_cookie',
}

# Crawler settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
ROBOTSTXT_OBEY = False
```

## Usage

### Starting the System

**1. Launch the Web Application:**
```bash
python app.py
```
Access the dashboard at `http://localhost:5000`

**2. Start Crawlers (via Web Interface or CLI):**

Via CLI:
```bash
# Start all crawlers
python run_spiders.py

# Test specific crawler
python runspider-test.py --spider=weibo
```

Via web interface: Use the hotkey control in the dashboard to start/stop crawlers.

### Core API Examples

#### Query Hot Topics

```python
from hotsearch_analysis_agent import HotSearchAnalyzer

# Initialize analyzer
analyzer = HotSearchAnalyzer(
    db_host='localhost',
    db_user='user',
    db_password='password',
    db_name='hotsearch_db'
)

# Query recent hot topics from specific platform
topics = analyzer.query_topics(
    platform='weibo',
    limit=10,
    time_range='24h'
)

for topic in topics:
    print(f"{topic['rank']}. {topic['title']} - {topic['hot_score']}")
```

#### Perform Sentiment Analysis

```python
from hotsearch_analysis_agent import SentimentAnalyzer

analyzer = SentimentAnalyzer(model_name='gpt-4')

# Analyze sentiment of a topic
result = analyzer.analyze_sentiment(
    text="今天的新闻真是振奋人心，科技发展太快了！",
    context={'platform': 'weibo', 'topic_id': 123}
)

print(f"Sentiment: {result['sentiment']}")  # positive/negative/neutral
print(f"Confidence: {result['confidence']}")
print(f"Key phrases: {result['key_phrases']}")
```

#### Topic Clustering

```python
from hotsearch_analysis_agent import TopicClusterer

clusterer = TopicClusterer(llm_model='gpt-4')

# Fetch topics from last 24 hours
topics = analyzer.query_topics(time_range='24h', all_platforms=True)

# Perform clustering
clusters = clusterer.cluster_topics(
    topics=topics,
    num_clusters=5,
    algorithm='semantic'  # or 'keyword'
)

for cluster in clusters:
    print(f"\nCluster: {cluster['theme']}")
    print(f"Topics: {len(cluster['topics'])}")
    for topic in cluster['topics'][:3]:
        print(f"  - {topic['title']}")
```

#### Natural Language Query

```python
from hotsearch_analysis_agent import NLQueryEngine

query_engine = NLQueryEngine(llm_model='gpt-4')

# Ask questions in natural language
response = query_engine.query(
    "最近关于人工智能的热点话题有哪些?",
    db_connection=analyzer.db
)

print(response['answer'])
print(f"Related topics: {response['topics']}")
```

### Web Scraper Implementation

#### Custom Spider Example

```python
import scrapy
from hotsearchcrawler.items import HotSearchItem

class WeiboSpider(scrapy.Spider):
    name = 'weibo'
    start_urls = ['https://s.weibo.com/top/summary']
    
    def parse(self, response):
        for rank, item in enumerate(response.css('tbody tr'), 1):
            hot_item = HotSearchItem()
            hot_item['platform'] = 'weibo'
            hot_item['rank'] = rank
            hot_item['title'] = item.css('td.td-02 a::text').get()
            hot_item['hot_score'] = item.css('td.td-02 span::text').get()
            hot_item['url'] = item.css('td.td-02 a::attr(href)').get()
            hot_item['category'] = item.css('td.td-03 i::attr(class)').re_first(r'icon-(\w+)')
            
            # Fetch detail page content
            if hot_item['url']:
                yield scrapy.Request(
                    hot_item['url'],
                    callback=self.parse_detail,
                    meta={'item': hot_item}
                )
    
    def parse_detail(self, response):
        item = response.meta['item']
        item['content'] = response.css('article.content::text').getall()
        item['comments_count'] = response.css('.comment-count::text').get()
        yield item
```

### Push Notification Setup

#### Configure Alert Task

```python
from hotsearch_analysis_agent import PushTaskManager

task_manager = PushTaskManager()

# Create a scheduled push task
task = task_manager.create_task(
    name="AI Tech Daily Report",
    query="人工智能 科技",
    platforms=['weibo', 'zhihu', 'bilibili'],
    schedule="0 12 * * *",  # Daily at 12:00 PM
    channels=['email', 'telegram'],
    analysis_depth='detailed',  # basic/detailed/comprehensive
    min_hot_score=10000
)

# Test push task
task_manager.test_task(task_id=task['id'])
```

#### Email Report Template

```python
from hotsearch_analysis_agent import ReportGenerator

generator = ReportGenerator(llm_model='gpt-4')

report = generator.generate_report(
    topics=topics,
    theme="AI and Technology Trends",
    format='html',  # or 'markdown', 'text'
    include_sentiment=True,
    include_clusters=True
)

# Send via email
generator.send_report(
    report=report,
    recipients=['user@example.com'],
    subject="Daily AI Opinion Report"
)
```

## Common Patterns

### Full Analysis Pipeline

```python
from hotsearch_analysis_agent import AnalysisPipeline

# Create end-to-end pipeline
pipeline = AnalysisPipeline(
    db_config={'host': 'localhost', 'user': 'user', 'password': 'pass'},
    llm_config={'model': 'gpt-4', 'api_key': os.getenv('OPENAI_API_KEY')}
)

# Execute full analysis
results = pipeline.run(
    query="新能源汽车",
    platforms=['weibo', 'douyin', 'bilibili'],
    time_range='7d',
    steps=[
        'fetch_topics',
        'extract_content',
        'sentiment_analysis',
        'topic_clustering',
        'trend_analysis',
        'generate_report'
    ]
)

# Access results
print(f"Total topics analyzed: {results['total_topics']}")
print(f"Sentiment distribution: {results['sentiment_distribution']}")
print(f"Top clusters: {results['clusters'][:3]}")
print(f"Report: {results['report_url']}")
```

### Video Content Extraction

```python
from hotsearch_analysis_agent import VideoContentExtractor

extractor = VideoContentExtractor(browser='chrome')

# Extract content from video news
content = extractor.extract_video_info(
    url='https://www.bilibili.com/video/BV13pSoBBEvX',
    extract_comments=True,
    max_comments=100
)

print(f"Title: {content['title']}")
print(f"Description: {content['description']}")
print(f"Transcript: {content['transcript']}")  # If available
print(f"Top comments: {content['comments'][:5]}")
```

### Database Query Utilities

```python
from hotsearch_analysis_agent import DBQuery

db = DBQuery(host='localhost', user='user', password='pass')

# Get trending keywords
keywords = db.get_trending_keywords(
    time_range='24h',
    top_n=20,
    platforms=['weibo', 'zhihu']
)

# Get topic history
history = db.get_topic_history(
    topic_id=12345,
    time_range='30d'
)

# Get platform statistics
stats = db.get_platform_stats(
    platforms=['weibo', 'douyin'],
    metrics=['total_topics', 'avg_hot_score', 'sentiment_ratio']
)
```

## Troubleshooting

### Browser Driver Issues

**Error: `chromedriver not found`**
```bash
# Verify driver is in PATH
which chromedriver  # Linux/macOS
where chromedriver  # Windows

# Or specify driver path explicitly in settings
CHROME_DRIVER_PATH=/usr/local/bin/chromedriver
```

**Error: Browser version mismatch**
- Download driver matching your browser version exactly
- Check browser version: `google-chrome --version` or in browser settings

### Database Connection Errors

**Error: `Access denied for user`**
```python
# Verify credentials
import pymysql
connection = pymysql.connect(
    host='localhost',
    user='your_user',
    password='your_password',
    database='hotsearch_db'
)
connection.close()
```

**Error: `Table doesn't exist`**
- Ensure `init.py` was run successfully
- Check table creation: `SHOW TABLES;` in MySQL

### LLM API Issues

**Error: Rate limit exceeded**
```python
# Add retry logic with exponential backoff
from tenacity import retry, wait_exponential, stop_after_attempt

@retry(wait=wait_exponential(multiplier=1, min=4, max=60), 
       stop=stop_after_attempt(5))
def call_llm_api(prompt):
    return analyzer.analyze(prompt)
```

**Error: Token limit exceeded**
```python
# Chunk long content
from hotsearch_analysis_agent import TextChunker

chunker = TextChunker(max_tokens=4000)
chunks = chunker.split_text(long_content)

results = []
for chunk in chunks:
    result = analyzer.analyze(chunk)
    results.append(result)

# Merge results
final_result = chunker.merge_results(results)
```

### Crawler Issues

**Error: `403 Forbidden` or blocked by anti-scraping**
```python
# Rotate user agents
USER_AGENT_LIST = [
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64)...',
    'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)...',
]

# Add delays
DOWNLOAD_DELAY = 3
RANDOMIZE_DOWNLOAD_DELAY = True

# Use cookies (get from browser)
COOKIES = {'session_id': 'your_session_cookie'}
```

**Error: Incomplete data extraction**
```python
# Debug selector
scrapy shell 'https://example.com'
response.css('your::selector').getall()  # Test in shell first

# Add wait for dynamic content
from selenium.webdriver.support.ui import WebDriverWait
element = WebDriverWait(driver, 10).until(
    EC.presence_of_element_located((By.CSS_SELECTOR, '.content'))
)
```

### Memory Issues with Large Datasets

```python
# Process in batches
BATCH_SIZE = 1000
offset = 0

while True:
    topics = db.fetch_topics(limit=BATCH_SIZE, offset=offset)
    if not topics:
        break
    
    # Process batch
    results = analyzer.analyze_batch(topics)
    db.save_results(results)
    
    offset += BATCH_SIZE
```

## Advanced Configuration

### Using Huawei Pangu Model (Local)

```python
from hotsearch_analysis_agent import PanguAnalyzer

# Initialize with local Pangu model
analyzer = PanguAnalyzer(
    model_path='/path/to/openpangu-embedded-7b-model',
    device='cuda',  # or 'cpu'
    max_length=2048
)

# Use same interface as OpenAI analyzer
result = analyzer.analyze_sentiment(text="...")
```

### Custom Analysis Rules

```python
from hotsearch_analysis_agent import RuleEngine

engine = RuleEngine()

# Add custom sentiment rules
engine.add_rule(
    name='tech_positive_keywords',
    pattern=['突破', '创新', '领先', '成功'],
    action='boost_positive_score',
    weight=1.5
)

# Apply to analysis
result = analyzer.analyze_with_rules(text, rules=engine)
```
