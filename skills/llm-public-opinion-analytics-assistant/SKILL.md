---
name: llm-public-opinion-analytics-assistant
description: Multi-platform hot search data crawler and LLM-powered public opinion analysis assistant with sentiment analysis, topic clustering, and multi-channel alerts
triggers:
  - how do I set up the public opinion analytics assistant
  - scrape hot search data from multiple platforms
  - analyze sentiment and cluster topics with LLM
  - configure push notifications for trending topics
  - use the opinion analysis assistant API
  - set up pangu model for sentiment analysis
  - crawl weibo bilibili douyin hot search data
  - create automated opinion monitoring tasks
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive public opinion monitoring system that integrates 26 hot search ranking lists from 15 mainstream Chinese platforms (Weibo, Bilibili, Douyin, etc.) with LLM-powered analysis capabilities. Features conversational queries, topic clustering, sentiment analysis, and multi-channel push notifications (WeChat, Telegram, Email).

## What It Does

- **Multi-Platform Data Collection**: Crawls hot search data from 15 platforms (26 ranking lists total)
- **LLM-Powered Analysis**: Uses Pangu or OpenAI-compatible models for sentiment analysis and topic clustering
- **Conversational Interface**: Natural language queries for hot search data and trend analysis
- **Deep Content Extraction**: Extracts content from news detail pages, including video transcripts
- **Multi-Channel Alerts**: Push notifications via Enterprise WeChat, Telegram, and email
- **Hotkey Control**: Start/stop crawlers with keyboard shortcuts

## Project Structure

```
.
├── app.py                          # Main application entry point
├── hotsearch_analysis_agent/       # Analysis system
│   ├── config/                     # Configuration files
│   ├── models/                     # Database models
│   ├── services/                   # Business logic services
│   └── utils/                      # Utility functions
├── hotsearchcrawler/               # Scrapy crawler cluster
│   ├── spiders/                    # Platform-specific spiders
│   ├── settings.py                 # Crawler settings
│   └── pipelines.py                # Data processing pipelines
├── run_spiders.py                  # Crawler launcher
├── test_push_task.py               # Push notification testing
└── init.py                         # Database initialization
```

## Installation

### Prerequisites

1. **Python 3.8+** with virtual environment
2. **MySQL 5.7+** database
3. **Browser Driver** (Chrome/Edge WebDriver)

### Step 1: Browser Driver Setup

Download the appropriate WebDriver for your browser:

- **Chrome**: https://chromedriver.chromium.org/
- **Edge**: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

Place the driver in your system PATH or browser installation directory.

Verify installation:
```bash
chromedriver --version
# or
msedgedriver --version
```

### Step 2: Install Dependencies

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

### Step 3: Database Setup

Create MySQL database and tables:

```python
# Reference init.py for schema
import mysql.connector

db = mysql.connector.connect(
    host="localhost",
    user="root",
    password=os.getenv("MYSQL_PASSWORD"),
    database="opinion_db"
)

cursor = db.cursor()

# Create hot search table
cursor.execute("""
CREATE TABLE IF NOT EXISTS hot_search (
    id INT AUTO_INCREMENT PRIMARY KEY,
    platform VARCHAR(50),
    title VARCHAR(500),
    url VARCHAR(1000),
    hot_value VARCHAR(100),
    rank_position INT,
    content TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_platform (platform),
    INDEX idx_created_at (created_at)
)
""")

# Create analysis results table
cursor.execute("""
CREATE TABLE IF NOT EXISTS analysis_results (
    id INT AUTO_INCREMENT PRIMARY KEY,
    query TEXT,
    result LONGTEXT,
    sentiment VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)
""")

db.commit()
```

### Step 4: Configuration

Create `.env` file in project root:

```bash
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=opinion_db

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
MODEL_NAME=gpt-4

# Or use Pangu model (local deployment)
USE_PANGU=true
PANGU_MODEL_PATH=/path/to/pangu/model

# Push Notification Configuration
# Enterprise WeChat Robot
WECHAT_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email SMTP
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
RECIPIENT_EMAIL=recipient@example.com
```

Configure crawler settings in `hotsearchcrawler/settings.py`:

```python
# MySQL Pipeline Settings
MYSQL_CONFIG = {
    'host': os.getenv('MYSQL_HOST'),
    'port': int(os.getenv('MYSQL_PORT')),
    'user': os.getenv('MYSQL_USER'),
    'password': os.getenv('MYSQL_PASSWORD'),
    'database': os.getenv('MYSQL_DATABASE'),
}

# Optional: Platform-specific cookies
COOKIES = {
    'weibo': 'your_weibo_cookie',
    'bilibili': 'your_bilibili_cookie',
}
```

## Running the Application

### Start the Web Interface

```bash
python app.py
```

Access at `http://localhost:5000`

### Run Crawlers Manually

```bash
# Test individual spider
python runspider-test.py

# Run all spiders
python run_spiders.py
```

### Test Push Notifications

```bash
python test_push_task.py
```

## Key Usage Patterns

### 1. Crawling Hot Search Data

```python
from hotsearchcrawler.spiders import WeiboSpider, BilibiliSpider
from scrapy.crawler import CrawlerProcess
from scrapy.utils.project import get_project_settings

# Initialize crawler process
process = CrawlerProcess(get_project_settings())

# Add spiders
process.crawl(WeiboSpider)
process.crawl(BilibiliSpider)

# Start crawling
process.start()
```

### 2. Querying Hot Search Data

```python
from hotsearch_analysis_agent.services.query_service import QueryService

query_service = QueryService()

# Query by platform
weibo_trends = query_service.get_hot_search(platform='weibo', limit=10)

# Query by keyword
ai_topics = query_service.search_by_keyword('人工智能')

# Query by time range
today_trends = query_service.get_hot_search_by_date(
    start_date='2026-05-01',
    end_date='2026-05-19'
)
```

### 3. LLM-Powered Analysis

```python
from hotsearch_analysis_agent.services.analysis_service import AnalysisService

analysis_service = AnalysisService()

# Sentiment analysis
sentiment = analysis_service.analyze_sentiment(
    text="这个产品真的太好用了,强烈推荐!"
)
# Returns: {'sentiment': 'positive', 'confidence': 0.95}

# Topic clustering
topics = analysis_service.cluster_topics(
    keywords=['人工智能', 'AI', '深度学习', '机器学习']
)
# Returns: [{'cluster': 0, 'keywords': ['人工智能', 'AI'], 'count': 150}, ...]

# Generate analysis report
report = analysis_service.generate_report(
    query="人工智能与前沿科技",
    time_range={'start': '2026-04-01', 'end': '2026-04-07'}
)
```

### 4. Setting Up Push Tasks

```python
from hotsearch_analysis_agent.services.push_service import PushService

push_service = PushService()

# Configure push task
task_config = {
    'name': 'AI热点监控',
    'keywords': ['人工智能', 'GPT', 'DeepSeek'],
    'platforms': ['weibo', 'bilibili', 'zhihu'],
    'threshold': 100000,  # Hot value threshold
    'interval': 3600,  # Check every hour (seconds)
    'channels': ['wechat', 'telegram', 'email']
}

# Create push task
task_id = push_service.create_task(task_config)

# Send test notification
push_service.send_notification(
    channel='wechat',
    title='测试推送',
    content='这是一条测试消息',
    url='https://example.com'
)
```

### 5. Conversational Query Interface

```python
from hotsearch_analysis_agent.services.chat_service import ChatService

chat_service = ChatService()

# Natural language query
response = chat_service.process_query(
    user_input="今天微博上最火的科技新闻是什么?"
)

# Multi-turn conversation
conversation_id = chat_service.start_conversation()
chat_service.add_message(conversation_id, "最近AI领域有什么新闻?")
response1 = chat_service.get_response(conversation_id)

chat_service.add_message(conversation_id, "能详细说说DeepSeek吗?")
response2 = chat_service.get_response(conversation_id)
```

### 6. Custom Spider Example

```python
# hotsearchcrawler/spiders/custom_spider.py
import scrapy
from hotsearchcrawler.items import HotSearchItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://example.com/hot']
    
    def parse(self, response):
        for item in response.css('.hot-item'):
            hot_item = HotSearchItem()
            hot_item['platform'] = 'custom_platform'
            hot_item['title'] = item.css('.title::text').get()
            hot_item['url'] = item.css('a::attr(href)').get()
            hot_item['hot_value'] = item.css('.hot-value::text').get()
            hot_item['rank_position'] = item.css('.rank::text').get()
            
            # Extract detail content
            if hot_item['url']:
                yield scrapy.Request(
                    hot_item['url'],
                    callback=self.parse_detail,
                    meta={'item': hot_item}
                )
    
    def parse_detail(self, response):
        item = response.meta['item']
        item['content'] = response.css('.article-content::text').getall()
        yield item
```

## API Reference

### QueryService

```python
class QueryService:
    def get_hot_search(self, platform=None, limit=50):
        """Get latest hot search data"""
        
    def search_by_keyword(self, keyword, platform=None):
        """Search by keyword across platforms"""
        
    def get_hot_search_by_date(self, start_date, end_date, platform=None):
        """Query hot search data by date range"""
        
    def get_trending_keywords(self, days=7, limit=20):
        """Get trending keywords in recent days"""
```

### AnalysisService

```python
class AnalysisService:
    def analyze_sentiment(self, text):
        """Analyze sentiment of text"""
        
    def cluster_topics(self, keywords, num_clusters=5):
        """Cluster related topics"""
        
    def generate_report(self, query, time_range):
        """Generate comprehensive analysis report"""
        
    def detect_trends(self, keyword, time_range):
        """Detect trend changes for keyword"""
```

### PushService

```python
class PushService:
    def create_task(self, config):
        """Create automated push task"""
        
    def send_notification(self, channel, title, content, url=None):
        """Send notification to specified channel"""
        
    def send_wechat(self, content):
        """Send to Enterprise WeChat"""
        
    def send_telegram(self, message):
        """Send to Telegram"""
        
    def send_email(self, subject, body):
        """Send email notification"""
```

## Configuration Examples

### Multi-Platform Crawler Configuration

```python
# hotsearchcrawler/settings.py

# Supported platforms
SPIDERS = {
    'weibo': 'WeiboSpider',
    'bilibili': 'BilibiliSpider',
    'zhihu': 'ZhihuSpider',
    'douyin': 'DouyinSpider',
    'toutiao': 'ToutiaoSpider',
    # ... 15 platforms total
}

# Crawler settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 2
RANDOMIZE_DOWNLOAD_DELAY = True

# User agent rotation
USER_AGENTS = [
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
    'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36',
]
```

### LLM Model Configuration

```python
# Using Pangu model (local deployment)
from hotsearch_analysis_agent.models.pangu_model import PanguModel

model = PanguModel(
    model_path=os.getenv('PANGU_MODEL_PATH'),
    max_length=2048,
    temperature=0.7
)

# Or use OpenAI-compatible API
from openai import OpenAI

client = OpenAI(
    api_key=os.getenv('OPENAI_API_KEY'),
    base_url=os.getenv('OPENAI_API_BASE')
)
```

## Troubleshooting

### Crawler Issues

**Problem**: WebDriver not found
```bash
# Verify driver in PATH
which chromedriver  # Linux/Mac
where chromedriver  # Windows

# Add to PATH if needed
export PATH=$PATH:/path/to/driver
```

**Problem**: Platform returns 403/blocked
```python
# Add cookies in hotsearchcrawler/settings.py
COOKIES = {
    'weibo': 'your_cookie_string',
}

# Or reduce request rate
DOWNLOAD_DELAY = 5
CONCURRENT_REQUESTS = 8
```

**Problem**: Content extraction fails
```python
# Enable debug logging
import logging
logging.basicConfig(level=logging.DEBUG)

# Check response in spider
def parse(self, response):
    self.logger.debug(f"Response: {response.text[:500]}")
```

### Database Issues

**Problem**: Connection refused
```bash
# Check MySQL is running
sudo systemctl status mysql  # Linux
brew services list  # Mac

# Verify credentials
mysql -u root -p -h localhost
```

**Problem**: Table doesn't exist
```python
# Re-run initialization
python init.py

# Or create manually
python -c "from init import create_tables; create_tables()"
```

### LLM Analysis Issues

**Problem**: Model response is empty
```python
# Check API connection
import openai
response = openai.ChatCompletion.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "test"}]
)
print(response)

# For Pangu: verify model path
ls -lh /path/to/pangu/model
```

**Problem**: Sentiment analysis inaccurate
```python
# Adjust temperature and prompt
analysis_service = AnalysisService(
    temperature=0.3,  # Lower for more consistent results
    system_prompt="你是一个专业的舆情分析专家,擅长中文情感分析..."
)
```

### Push Notification Issues

**Problem**: WeChat webhook fails
```python
# Test webhook directly
import requests
requests.post(
    os.getenv('WECHAT_WEBHOOK'),
    json={'msgtype': 'text', 'text': {'content': 'test'}}
)
```

**Problem**: Email not sending
```python
# Verify SMTP settings
import smtplib
server = smtplib.SMTP(os.getenv('SMTP_HOST'), int(os.getenv('SMTP_PORT')))
server.starttls()
server.login(os.getenv('SMTP_USER'), os.getenv('SMTP_PASSWORD'))
```

## Common Workflows

### Daily Monitoring Setup

```python
# 1. Configure crawler schedule (cron)
# 0 */2 * * * cd /path/to/project && python run_spiders.py

# 2. Set up automated analysis
from hotsearch_analysis_agent.tasks import AnalysisTask

task = AnalysisTask()
task.schedule_daily_analysis(
    keywords=['人工智能', '科技'],
    time='09:00'
)

# 3. Configure push notifications
push_service.create_task({
    'name': '每日科技热点',
    'keywords': ['AI', '科技', '创新'],
    'interval': 86400,  # Daily
    'channels': ['email']
})
```

### Custom Analysis Pipeline

```python
# Create custom analysis workflow
from hotsearch_analysis_agent.pipeline import AnalysisPipeline

pipeline = AnalysisPipeline()

# Add processing steps
pipeline.add_step('fetch_data', query_service.get_hot_search)
pipeline.add_step('clean_data', lambda x: [i for i in x if i['hot_value'] > 10000])
pipeline.add_step('analyze_sentiment', analysis_service.analyze_sentiment)
pipeline.add_step('cluster_topics', analysis_service.cluster_topics)
pipeline.add_step('generate_report', analysis_service.generate_report)

# Execute pipeline
result = pipeline.run(platform='weibo', limit=100)
```
