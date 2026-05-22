---
name: llm-public-opinion-analytics-assistant
description: Multi-platform hot search crawler with LLM-powered sentiment analysis, topic clustering, and multi-channel alert system for Chinese social media monitoring
triggers:
  - set up public opinion monitoring system
  - scrape hot search trends from weibo bilibili douyin
  - analyze sentiment and cluster topics with LLM
  - configure hot topic push notifications
  - deploy chinese social media crawler
  - build sentiment analysis pipeline for news
  - monitor trending topics across platforms
  - set up pangu model for opinion analytics
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive public opinion monitoring system that crawls 26 trending lists from 15 major Chinese platforms (Weibo, Bilibili, Douyin, Zhihu, etc.), analyzes content using large language models, and provides multi-channel alerting (WeChat Work, Telegram, Email). The system features conversational querying, topic clustering, sentiment analysis, and automated report generation.

## What It Does

- **Multi-Platform Crawling**: Scrapes real-time trending data from Weibo, Bilibili, Douyin, Zhihu, Baidu, Toutiao, and 9 other platforms
- **LLM-Powered Analysis**: Topic clustering, sentiment analysis, and summarization using Pangu or OpenAI-compatible models
- **Conversational Interface**: Natural language queries for hot search analysis
- **Content Extraction**: Deep crawling of news detail pages including video content
- **Multi-Channel Alerts**: Push notifications via WeChat Work, Telegram, and email
- **Web Dashboard**: Real-time monitoring interface with quick links to source content

## Installation

### Prerequisites

**1. Browser Driver Setup**

The crawler requires ChromeDriver or EdgeDriver for content extraction:

```bash
# Check your Chrome/Edge version first
google-chrome --version  # or microsoft-edge --version

# Download matching driver from:
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Place driver in system PATH
sudo mv chromedriver /usr/local/bin/  # Linux/macOS
# or add to PATH on Windows: C:\WebDriver\
```

**2. MySQL Database**

```bash
# Install MySQL 5.7+
sudo apt-get install mysql-server  # Ubuntu
brew install mysql  # macOS

# Create database
mysql -u root -p
CREATE DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
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

### Database Initialization

```python
# Reference init.py for table schema
# Key tables:
# - hotsearch_data: Raw crawled trending items
# - news_content: Detailed news content
# - analysis_results: LLM analysis outputs
# - push_tasks: Alert configurations

# Run initialization
python init.py
```

## Configuration

### Environment Variables

Create `.env` file in project root:

```bash
# Database Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_username
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
OPENAI_MODEL=gpt-4

# Or use Pangu Model (local deployment)
PANGU_API_BASE=http://localhost:8000
PANGU_MODEL=pangu-embedded-7b

# Push Notification Settings
WECHAT_WORK_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxx
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email
SMTP_PASSWORD=your_email_password
```

### Crawler Configuration

Edit `hotsearchcrawler/settings.py`:

```python
# MySQL connection for crawler
MYSQL_CONFIG = {
    'host': os.getenv('MYSQL_HOST', 'localhost'),
    'port': int(os.getenv('MYSQL_PORT', 3306)),
    'user': os.getenv('MYSQL_USER'),
    'password': os.getenv('MYSQL_PASSWORD'),
    'database': os.getenv('MYSQL_DATABASE'),
    'charset': 'utf8mb4'
}

# Crawler settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
COOKIES_ENABLED = True

# Optional: Platform-specific cookies
# Add cookies in hotsearchcrawler/cookies/ for authenticated access
```

## Running the System

### Start Web Interface

```bash
# Launch Flask application
python app.py

# Access at http://localhost:5000
```

### Manual Crawler Execution

```bash
# Test individual spider
python runspider-test.py weibo

# Run all spiders
python run_spiders.py

# Available spiders:
# weibo, bilibili, douyin, zhihu, baidu, toutiao, 
# 36kr, thepaper, iheima, tencent_news, etc.
```

### Programmatic Usage

```python
from hotsearch_analysis_agent.analyzer import TopicAnalyzer
from hotsearch_analysis_agent.database import DatabaseManager

# Initialize components
db = DatabaseManager()
analyzer = TopicAnalyzer(model='pangu-embedded-7b')

# Query hot searches
query = "人工智能相关热搜"
results = db.search_hot_topics(query, platform='weibo', limit=20)

# Perform topic clustering
clustered = analyzer.cluster_topics(results)
print(f"Found {len(clustered)} topic clusters")

# Sentiment analysis
for cluster in clustered:
    sentiment = analyzer.analyze_sentiment(cluster['content'])
    print(f"Topic: {cluster['title']}")
    print(f"Sentiment: {sentiment['score']} ({sentiment['label']})")
    
# Generate report
report = analyzer.generate_report(
    clustered, 
    query="AI和科技前沿",
    format='markdown'
)
```

## Key Features & Patterns

### 1. Conversational Query

```python
from hotsearch_analysis_agent.chat import ChatInterface

chat = ChatInterface()

# Natural language queries
response = chat.query("最近三天关于新能源汽车的热搜")
# Returns: Structured data + LLM summary

response = chat.query("分析一下今天微博热搜的情感倾向")
# Returns: Sentiment distribution + key topics

response = chat.query("哪些科技话题在多个平台同时热门?")
# Returns: Cross-platform trending analysis
```

### 2. Automated Push Notifications

```python
from hotsearch_analysis_agent.push import PushManager

push_mgr = PushManager()

# Create monitoring task
task = push_mgr.create_task(
    name="AI热点监控",
    keywords=["人工智能", "ChatGPT", "大模型"],
    platforms=["weibo", "zhihu", "bilibili"],
    threshold={"heat_index": 100000},
    channels=["wechat_work", "telegram"],
    schedule="0 */2 * * *"  # Every 2 hours
)

# Manual trigger
push_mgr.execute_task(task['id'])
```

### 3. Custom Analysis Pipeline

```python
from hotsearch_analysis_agent.pipeline import AnalysisPipeline
from hotsearch_analysis_agent.extractors import ContentExtractor

pipeline = AnalysisPipeline()

# Define analysis stages
pipeline.add_stage('extract', ContentExtractor(deep_crawl=True))
pipeline.add_stage('cluster', TopicClusterer(min_similarity=0.7))
pipeline.add_stage('sentiment', SentimentAnalyzer())
pipeline.add_stage('summarize', LLMSummarizer(max_length=500))

# Process batch
hot_searches = db.get_recent_hot_searches(hours=24)
results = pipeline.run(hot_searches)

# Export results
pipeline.export(results, format='json', output='analysis_results.json')
```

### 4. Deep Content Extraction

```python
from hotsearch_analysis_agent.extractors import DetailPageExtractor

extractor = DetailPageExtractor()

# Extract from news URL (including video content)
content = extractor.extract("https://weibo.com/1234567890/abcdefg")

print(content['title'])
print(content['text'])
print(content['images'])
print(content['video_transcript'])  # AI-extracted from video
print(content['comments'][:5])
```

### 5. Topic Clustering

```python
from hotsearch_analysis_agent.clustering import SemanticClusterer

clusterer = SemanticClusterer(
    model='pangu-embedded-7b',
    similarity_threshold=0.75
)

# Cluster hot searches
topics = db.get_hot_topics(date='2026-05-21', platforms=['weibo', 'zhihu'])
clusters = clusterer.fit_predict(topics)

# Analyze clusters
for cluster_id, items in clusters.items():
    print(f"\n=== Cluster {cluster_id} ===")
    print(f"Size: {len(items)}")
    print(f"Keywords: {clusterer.extract_keywords(items)}")
    print(f"Platforms: {set(item['platform'] for item in items)}")
```

## Dashboard Integration

### Frontend API Calls

```python
from flask import Flask, jsonify, request
from hotsearch_analysis_agent.api import APIRouter

app = Flask(__name__)
api = APIRouter()

@app.route('/api/search', methods=['POST'])
def search():
    query = request.json['query']
    filters = request.json.get('filters', {})
    
    results = api.search_topics(
        query=query,
        platforms=filters.get('platforms'),
        date_range=filters.get('date_range'),
        limit=50
    )
    
    return jsonify(results)

@app.route('/api/analyze', methods=['POST'])
def analyze():
    topic_ids = request.json['topic_ids']
    analysis_type = request.json['type']  # 'sentiment', 'cluster', 'summary'
    
    analysis = api.analyze(topic_ids, analysis_type)
    return jsonify(analysis)

@app.route('/api/crawler/control', methods=['POST'])
def crawler_control():
    action = request.json['action']  # 'start', 'stop'
    spiders = request.json.get('spiders', 'all')
    
    if action == 'start':
        api.start_crawlers(spiders)
    else:
        api.stop_crawlers(spiders)
    
    return jsonify({'status': 'success'})
```

## Report Generation

```python
from hotsearch_analysis_agent.reports import ReportGenerator

generator = ReportGenerator()

# Generate comprehensive report
report = generator.create_report(
    query="人工智能与前沿科技",
    date_range=("2026-05-15", "2026-05-21"),
    platforms=["all"],
    include_sections=[
        'overview',
        'key_findings',
        'detailed_news',
        'sentiment_analysis',
        'trend_forecast'
    ]
)

# Export formats
generator.export_markdown(report, 'report.md')
generator.export_html(report, 'report.html')
generator.export_pdf(report, 'report.pdf')

# Push report
push_mgr.send_report(
    report,
    channels=['email', 'wechat_work'],
    recipients=['team@example.com']
)
```

## Platform-Specific Crawlers

```python
# Available spider names and their targets:

spiders = {
    'weibo': 'Weibo Hot Search',
    'bilibili': 'Bilibili Trending Videos',
    'douyin': 'Douyin Hot Topics',
    'zhihu': 'Zhihu Hot Questions',
    'baidu': 'Baidu Hot Search',
    'toutiao': 'Toutiao Hot News',
    '36kr': '36Kr Tech News',
    'thepaper': 'The Paper News',
    'iheima': 'iheima Startup News',
    'tencent_news': 'Tencent News',
    'netease_news': 'NetEase News',
    'sina_news': 'Sina News',
    'ifeng': 'ifeng News',
    'cankaoxiaoxi': 'Cankaoxiaoxi',
    'huanqiu': 'Huanqiu News'
}

# Run specific spiders
from hotsearchcrawler.runner import SpiderRunner

runner = SpiderRunner()
runner.run_spiders(['weibo', 'zhihu', 'bilibili'])
```

## Troubleshooting

### ChromeDriver Issues

```bash
# Version mismatch error
# Solution: Download exact version matching your browser

# Permission denied
chmod +x /usr/local/bin/chromedriver

# PATH not found
export PATH=$PATH:/path/to/chromedriver/directory
```

### Database Connection Errors

```python
# Test connection
from hotsearch_analysis_agent.database import DatabaseManager

db = DatabaseManager()
if db.test_connection():
    print("Database connected successfully")
else:
    print("Connection failed - check credentials in .env")
```

### LLM API Errors

```python
# Fallback to local model if API fails
from hotsearch_analysis_agent.analyzer import TopicAnalyzer

analyzer = TopicAnalyzer(
    primary_model='gpt-4',
    fallback_model='pangu-embedded-7b',
    timeout=30
)
```

### Memory Issues with Large Datasets

```python
# Process in batches
from hotsearch_analysis_agent.utils import batch_processor

def process_large_dataset(data, batch_size=100):
    for batch in batch_processor(data, batch_size):
        results = analyzer.process_batch(batch)
        db.save_results(results)
        # Results saved incrementally
```

### Cookie Expiration

```python
# Auto-refresh cookies for authenticated platforms
from hotsearchcrawler.middleware import CookieRefreshMiddleware

# Enable in settings.py:
DOWNLOADER_MIDDLEWARES = {
    'hotsearchcrawler.middleware.CookieRefreshMiddleware': 543,
}

# Manual cookie update
db.update_cookies('weibo', new_cookies_dict)
```

## Advanced Configuration

### Custom LLM Integration

```python
from hotsearch_analysis_agent.llm import BaseLLM

class CustomLLM(BaseLLM):
    def __init__(self, api_endpoint):
        self.endpoint = api_endpoint
    
    def generate(self, prompt, **kwargs):
        # Implement your LLM call
        response = requests.post(
            self.endpoint,
            json={'prompt': prompt, **kwargs}
        )
        return response.json()['text']

# Register custom LLM
analyzer = TopicAnalyzer(llm=CustomLLM('http://localhost:9000'))
```

### Scheduler Configuration

```python
from apscheduler.schedulers.background import BackgroundScheduler

scheduler = BackgroundScheduler()

# Schedule crawlers
scheduler.add_job(
    func=runner.run_all_spiders,
    trigger='cron',
    hour='*/1'  # Every hour
)

# Schedule analysis
scheduler.add_job(
    func=analyzer.run_daily_analysis,
    trigger='cron',
    hour=8,
    minute=0
)

scheduler.start()
```

This system is production-ready for Chinese social media monitoring, with proven stability handling 15 platforms and 26 trending lists simultaneously.
