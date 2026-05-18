---
name: llm-public-opinion-analytics-assistant
description: Multi-platform real-time hot search data collection and LLM-powered sentiment analysis assistant with multi-channel push notifications
triggers:
  - how do i set up a public opinion monitoring system
  - scrape trending topics from multiple platforms with sentiment analysis
  - build a hot search analyzer with llm integration
  - create a social media trend monitoring dashboard
  - analyze sentiment across weibo bilibili and other platforms
  - set up automated hot topic notifications to telegram or wechat
  - monitor public opinion with huawei pangu llm
  - aggregate trending news with emotion analysis
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is a comprehensive public opinion analysis assistant that integrates real-time data from **26 trending lists across 15 mainstream platforms** (Weibo, Bilibili, Douyin, Zhihu, etc.) with large language model analysis capabilities. It provides conversational hot search queries, topic clustering, sentiment analysis, and multi-channel hot topic push notifications (Email, WeChat, Enterprise WeChat, Telegram).

## What It Does

- **Multi-Platform Data Scraping**: Crawls trending topics from 15+ Chinese platforms using Scrapy
- **LLM-Powered Analysis**: Sentiment analysis, topic clustering, and trend summarization using Huawei Pangu or OpenAI-compatible models
- **Conversational Interface**: Natural language queries for hot search data and analysis
- **Multi-Channel Notifications**: Automated push to WeChat Work, Telegram, and email
- **Video Content Extraction**: Can extract and analyze content even from video-based news
- **Hotkey Control**: Keyboard shortcuts for crawler start/stop operations

## Installation

### Prerequisites

1. **Browser Driver Setup**:
   ```bash
   # Download ChromeDriver or EdgeDriver matching your browser version
   # Chrome: https://chromedriver.chromium.org/
   # Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/
   
   # Place driver in system PATH or browser directory
   # Verify installation:
   chromedriver --version
   ```

2. **MySQL Database**:
   ```bash
   # Install MySQL and create database
   mysql -u root -p
   CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
   ```

3. **Python Environment**:
   ```bash
   # Clone repository
   git clone https://github.com/hmmnxkl/LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant.git
   cd LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant
   
   # Create virtual environment
   python -m venv venv
   source venv/bin/activate  # On Windows: venv\Scripts\activate
   
   # Install dependencies
   pip install -r requirements.txt
   ```

### Database Initialization

```python
# Reference init.py for database schema
# Key tables: hot_search, news_detail, analysis_results, push_tasks

# Example table structure
"""
hot_search: id, platform, title, url, rank, heat_value, create_time
news_detail: id, hot_search_id, content, summary, sentiment, keywords
analysis_results: id, query, result_type, content, create_time
push_tasks: id, task_name, platforms, keywords, channels, schedule
"""
```

## Configuration

### Environment Variables (.env)

```bash
# Database Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
LLM_API_BASE=https://your-llm-endpoint/v1
LLM_API_KEY=your_api_key
LLM_MODEL=pangu-embedded-7b  # or gpt-4, etc.

# Push Notification Channels
# WeChat Work Robot
WECHAT_WORK_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# WeChat Work App (for personal WeChat push)
WECHAT_WORK_CORPID=your_corp_id
WECHAT_WORK_CORPSECRET=your_corp_secret
WECHAT_WORK_AGENTID=your_agent_id

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
EMAIL_RECIPIENTS=recipient1@example.com,recipient2@example.com
```

### Crawler Configuration (hotsearchcrawler/settings.py)

```python
# MySQL settings
MYSQL_HOST = os.getenv('MYSQL_HOST', 'localhost')
MYSQL_PORT = int(os.getenv('MYSQL_PORT', 3306))
MYSQL_USER = os.getenv('MYSQL_USER')
MYSQL_PASSWORD = os.getenv('MYSQL_PASSWORD')
MYSQL_DATABASE = os.getenv('MYSQL_DATABASE')

# Optional: Platform-specific cookies
COOKIES = {
    'weibo': 'your_weibo_cookies',  # For accessing login-required content
    'bilibili': 'your_bilibili_cookies'
}

# Crawler settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
ROBOTSTXT_OBEY = False
```

## Running the Application

### Start the Analysis System

```bash
# Main application with web interface
python app.py

# Access at http://localhost:5000 (default port)
```

### Start Crawlers

```python
# Method 1: Via web interface (recommended)
# Click "Start Crawler" button in the frontend

# Method 2: Direct script execution
python run_spiders.py

# Method 3: Test individual crawler
python runspider-test.py --spider weibo
```

### Test Push Notifications

```bash
# Test all configured push channels
python test_push_task.py
```

## Key API Usage

### Query Hot Search Data

```python
from hotsearch_analysis_agent.query import HotSearchQuery

# Initialize query engine
query_engine = HotSearchQuery()

# Get trending topics from specific platform
weibo_trends = query_engine.get_platform_trends(
    platform='weibo',
    limit=20,
    time_range='24h'
)

# Search by keyword
ai_topics = query_engine.search_topics(
    keyword='人工智能',
    platforms=['weibo', 'zhihu', 'bilibili'],
    date_from='2026-05-01'
)

# Get topic clusters
clusters = query_engine.cluster_topics(
    topics=ai_topics,
    num_clusters=5
)
```

### Sentiment Analysis

```python
from hotsearch_analysis_agent.analyzer import SentimentAnalyzer

analyzer = SentimentAnalyzer(
    model_name=os.getenv('LLM_MODEL'),
    api_base=os.getenv('LLM_API_BASE'),
    api_key=os.getenv('LLM_API_KEY')
)

# Analyze single topic
result = analyzer.analyze_sentiment(
    text="DeepSeek V4采用华为算力，国产芯片生态走到哪一步了？",
    include_keywords=True
)

print(result)
# {
#   'sentiment': 'positive',
#   'score': 0.78,
#   'keywords': ['DeepSeek', '华为', '国产芯片', '算力'],
#   'summary': '关于国产AI芯片生态发展的正面报道'
# }

# Batch analysis
topics = query_engine.search_topics(keyword='科技')
batch_results = analyzer.batch_analyze([t['title'] for t in topics])
```

### Topic Clustering

```python
from hotsearch_analysis_agent.clustering import TopicClusterer

clusterer = TopicClusterer(llm_analyzer=analyzer)

# Get recent topics and cluster them
recent_topics = query_engine.get_platform_trends(
    platform='all',
    limit=100,
    time_range='24h'
)

# Perform clustering
clustered = clusterer.cluster_and_summarize(
    topics=recent_topics,
    num_clusters=5,
    generate_summary=True
)

for cluster in clustered:
    print(f"Cluster: {cluster['theme']}")
    print(f"Topics: {len(cluster['topics'])}")
    print(f"Summary: {cluster['summary']}")
    print(f"Sentiment: {cluster['avg_sentiment']}\n")
```

### Set Up Push Tasks

```python
from hotsearch_analysis_agent.push import PushTaskManager

push_manager = PushTaskManager()

# Create scheduled push task
task = push_manager.create_task(
    task_name='AI Technology Daily Report',
    keywords=['人工智能', '大模型', 'AI', 'DeepSeek'],
    platforms=['weibo', 'zhihu', 'bilibili', 'toutiao'],
    channels=['wechat_work', 'telegram', 'email'],
    schedule='daily',  # 'daily', 'hourly', or cron expression
    time='09:00',
    include_sentiment=True,
    include_clustering=True,
    min_heat=1000  # Minimum heat value threshold
)

# Manual push
push_manager.execute_push(
    task_id=task['id'],
    force=True  # Force push even if no new data
)

# List active tasks
active_tasks = push_manager.list_tasks(status='active')
```

### Conversational Query (Chat Interface)

```python
from hotsearch_analysis_agent.chat import ChatAgent

chat_agent = ChatAgent(
    query_engine=query_engine,
    analyzer=analyzer,
    clusterer=clusterer
)

# Natural language queries
response = chat_agent.chat("最近人工智能领域有什么热点话题?")
print(response)

response = chat_agent.chat("分析一下关于DeepSeek的舆情")
print(response)

response = chat_agent.chat("帮我聚类分析今天科技类新闻")
print(response)
```

## Crawler Development

### Add New Platform Spider

```python
# hotsearchcrawler/spiders/new_platform.py
import scrapy
from hotsearchcrawler.items import HotSearchItem

class NewPlatformSpider(scrapy.Spider):
    name = 'new_platform'
    allowed_domains = ['example.com']
    start_urls = ['https://example.com/trending']
    
    def parse(self, response):
        for item in response.css('.trending-item'):
            hot_search = HotSearchItem()
            hot_search['platform'] = 'new_platform'
            hot_search['title'] = item.css('.title::text').get()
            hot_search['url'] = item.css('a::attr(href)').get()
            hot_search['rank'] = item.css('.rank::text').get()
            hot_search['heat_value'] = item.css('.heat::text').get()
            
            # Follow link to get detailed content
            yield response.follow(
                hot_search['url'],
                callback=self.parse_detail,
                meta={'hot_search': hot_search}
            )
    
    def parse_detail(self, response):
        hot_search = response.meta['hot_search']
        hot_search['content'] = response.css('.content::text').getall()
        hot_search['summary'] = response.css('.summary::text').get()
        yield hot_search
```

## Common Patterns

### Daily Monitoring Workflow

```python
# Complete workflow for daily hot topic monitoring
from hotsearch_analysis_agent import (
    HotSearchQuery, SentimentAnalyzer, 
    TopicClusterer, PushTaskManager
)

def daily_monitoring_pipeline(keywords, platforms):
    # 1. Initialize components
    query = HotSearchQuery()
    analyzer = SentimentAnalyzer()
    clusterer = TopicClusterer(llm_analyzer=analyzer)
    push_manager = PushTaskManager()
    
    # 2. Collect data
    topics = query.search_topics(
        keyword=keywords,
        platforms=platforms,
        time_range='24h'
    )
    
    # 3. Sentiment analysis
    analyzed = analyzer.batch_analyze(
        [t['title'] + ' ' + t.get('summary', '') for t in topics]
    )
    
    # 4. Clustering
    clustered = clusterer.cluster_and_summarize(
        topics=topics,
        num_clusters=4
    )
    
    # 5. Generate report
    report = generate_report(clustered, analyzed)
    
    # 6. Push to channels
    push_manager.push_report(
        report=report,
        channels=['wechat_work', 'telegram', 'email'],
        title=f"Daily Hot Topics Report - {datetime.now().strftime('%Y-%m-%d')}"
    )
    
    return report

# Schedule daily execution
daily_monitoring_pipeline(
    keywords=['人工智能', '前沿科技'],
    platforms=['weibo', 'zhihu', 'bilibili']
)
```

### Video Content Extraction

```python
# Extract content from video-based news (e.g., Bilibili)
from hotsearch_analysis_agent.extractors import VideoContentExtractor

extractor = VideoContentExtractor()

# Extract from video URL
video_content = extractor.extract_from_url(
    url='https://www.bilibili.com/video/BV13pSoBBEvX',
    extract_comments=True,
    extract_danmaku=True  # Bullet comments
)

print(video_content)
# {
#   'title': 'GPT-6遭提前曝光, 2M超长上下文来了',
#   'description': '...',
#   'tags': ['GPT', '人工智能', 'OpenAI'],
#   'comments_summary': '...',
#   'sentiment': 'positive'
# }
```

### Custom LLM Integration

```python
# Use custom LLM model (Huawei Pangu example)
from hotsearch_analysis_agent.llm import LLMClient

# Initialize with Pangu model
llm = LLMClient(
    model_name='pangu-embedded-7b',
    api_base='http://localhost:8000/v1',  # Local deployment
    model_type='pangu'
)

# Custom prompt for opinion analysis
prompt = """
分析以下新闻标题的舆情特征:
标题: {title}

请从以下维度分析:
1. 情感倾向 (positive/negative/neutral)
2. 关键实体和主题
3. 潜在影响和趋势
4. 公众关注度预测

以JSON格式返回结果。
"""

result = llm.analyze(
    prompt=prompt.format(title='DeepSeek V4采用华为算力'),
    temperature=0.3
)
```

## Troubleshooting

### Browser Driver Issues

```bash
# Error: "chromedriver executable needs to be in PATH"
# Solution 1: Add driver to PATH
export PATH=$PATH:/path/to/chromedriver

# Solution 2: Specify driver path in settings
# hotsearchcrawler/settings.py
CHROME_DRIVER_PATH = '/usr/local/bin/chromedriver'

# Verify driver works
chromedriver --version
```

### Database Connection Errors

```python
# Test MySQL connection
import pymysql

try:
    conn = pymysql.connect(
        host=os.getenv('MYSQL_HOST'),
        port=int(os.getenv('MYSQL_PORT')),
        user=os.getenv('MYSQL_USER'),
        password=os.getenv('MYSQL_PASSWORD'),
        database=os.getenv('MYSQL_DATABASE'),
        charset='utf8mb4'
    )
    print("Database connection successful")
    conn.close()
except Exception as e:
    print(f"Connection failed: {e}")
```

### Crawler Blocked or Rate Limited

```python
# Adjust crawler settings
# hotsearchcrawler/settings.py

# Increase delay between requests
DOWNLOAD_DELAY = 3

# Use random user agents
DOWNLOADER_MIDDLEWARES = {
    'scrapy.downloadermiddlewares.useragent.UserAgentMiddleware': None,
    'scrapy_user_agents.middlewares.RandomUserAgentMiddleware': 400,
}

# Add proxy support if needed
DOWNLOADER_MIDDLEWARES.update({
    'scrapy.downloadermiddlewares.httpproxy.HttpProxyMiddleware': 110,
})
```

### LLM API Timeout

```python
# Increase timeout and add retry logic
from hotsearch_analysis_agent.llm import LLMClient

llm = LLMClient(
    model_name=os.getenv('LLM_MODEL'),
    api_base=os.getenv('LLM_API_BASE'),
    api_key=os.getenv('LLM_API_KEY'),
    timeout=120,  # Increase timeout to 120 seconds
    max_retries=3,
    retry_delay=5
)
```

### Push Notification Failures

```bash
# Test individual push channels

# WeChat Work
curl -X POST "https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"msgtype":"text","text":{"content":"Test message"}}'

# Telegram
curl -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
  -d "chat_id=${TELEGRAM_CHAT_ID}&text=Test message"

# SMTP Email
python -c "
import smtplib
from email.mime.text import MIMEText
msg = MIMEText('Test')
msg['Subject'] = 'Test'
msg['From'] = '${SMTP_USER}'
msg['To'] = '${EMAIL_RECIPIENTS}'
with smtplib.SMTP('${SMTP_HOST}', ${SMTP_PORT}) as s:
    s.starttls()
    s.login('${SMTP_USER}', '${SMTP_PASSWORD}')
    s.send_message(msg)
print('Email sent successfully')
"
```

## Advanced Features

### Custom Report Templates

```python
# Define custom report format
report_template = """
## Report - {title}
**Time**: {time}

> {summary}

---

### 🔍 Core Findings

{findings}

---

### 📰 Detailed News

{news_items}

---

### 📊 Analysis

{analysis}
"""

# Generate formatted report
from hotsearch_analysis_agent.reports import ReportGenerator

generator = ReportGenerator(template=report_template)
report = generator.generate(
    title='人工智能与前沿科技热点分析',
    data=clustered_topics,
    include_charts=True
)
```

This skill provides comprehensive coverage of the LLM-Based Intelligent Public Opinion Analytics Assistant for real-world deployment and usage by AI coding agents.
