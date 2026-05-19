---
name: llm-public-opinion-analytics-assistant
description: Chinese multi-platform hot search crawler and LLM-powered public opinion analysis system with sentiment analysis, topic clustering, and multi-channel alerting
triggers:
  - scrape hot search data from Chinese platforms
  - analyze public opinion sentiment with LLM
  - set up hot topic push notifications
  - cluster trending topics from multiple sources
  - extract insights from Chinese news and video content
  - configure public opinion monitoring dashboard
  - deploy real-time sentiment analysis system
  - integrate Pangu LLM for Chinese text analysis
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is an intelligent public opinion monitoring system that crawls data from 15 Chinese platforms (26 hot search lists), performs LLM-powered analysis including sentiment detection and topic clustering, and pushes alerts via email, WeChat, Enterprise WeChat, and Telegram. It separates crawling (`hotsearchcrawler`) from analysis (`hotsearch_analysis_agent`) for scalable deployment.

## Core Capabilities

- **Multi-Platform Crawling**: Real-time data from Weibo, Bilibili, Baidu, Zhihu, Douyin, and 10+ other platforms
- **LLM Analysis**: Sentiment analysis, topic clustering, and trend detection using Huawei Pangu or OpenAI-compatible models
- **Content Extraction**: Deep scraping of article and video content for comprehensive analysis
- **Multi-Channel Alerts**: Email (SMTP), WeChat Work, Telegram bot integration
- **Conversational UI**: Natural language queries for hot search data
- **Keyboard Controls**: Quick start/stop crawler operations

## Installation

### Prerequisites

```bash
# Python 3.8+
python --version

# MySQL database
mysql --version
```

### Browser Driver Setup

**1. Check browser version:**
- Edge: `edge://settings/help`
- Chrome: `chrome://settings/help`

**2. Download matching driver:**
- Chrome: https://chromedriver.chromium.org/
- Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

**3. Add to PATH:**

Linux/macOS:
```bash
sudo mv chromedriver /usr/local/bin/
chmod +x /usr/local/bin/chromedriver
chromedriver --version  # Verify
```

Windows:
```powershell
# Move to C:\WebDriver\
# Add to PATH: System Properties > Environment Variables > Path > Add "C:\WebDriver"
chromedriver.exe --version  # Verify
```

### Install Dependencies

```bash
# Clone repository
git clone https://github.com/hmmnxkl/LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant.git
cd LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant

# Create virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install requirements
pip install -r requirements.txt
```

### Database Setup

```python
# Reference init.py for schema
# Create database
mysql -u root -p
```

```sql
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Example tables (see init.py for complete schema)
CREATE TABLE hot_searches (
    id INT AUTO_INCREMENT PRIMARY KEY,
    platform VARCHAR(50) NOT NULL,
    rank INT,
    title VARCHAR(500),
    url TEXT,
    heat VARCHAR(100),
    label VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_platform (platform),
    INDEX idx_created_at (created_at)
);

CREATE TABLE news_content (
    id INT AUTO_INCREMENT PRIMARY KEY,
    url TEXT NOT NULL,
    title VARCHAR(500),
    content TEXT,
    author VARCHAR(200),
    publish_time DATETIME,
    platform VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE push_tasks (
    id INT AUTO_INCREMENT PRIMARY KEY,
    task_name VARCHAR(200),
    query_keywords TEXT,
    push_channels JSON,
    schedule_cron VARCHAR(100),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## Configuration

### Crawler Settings

Edit `hotsearchcrawler/settings.py`:

```python
# MySQL Configuration
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_user'
MYSQL_PASSWORD = 'your_password'
MYSQL_DB = 'hotsearch_db'

# Optional: Platform cookies for authenticated scraping
COOKIES = {
    'weibo': 'your_weibo_cookies',
    'bilibili': 'your_bilibili_cookies'
}

# Scrapy settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
RANDOMIZE_DOWNLOAD_DELAY = True
```

### Analysis System Configuration

Create `.env` file in project root:

```bash
# Database
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DB=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_BASE_URL=https://api.openai.com/v1  # Or Pangu endpoint
MODEL_NAME=gpt-4  # Or openpangu-embedded-7b

# Push Notifications
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password

WECHAT_WORK_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxx
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

### Pangu Model Setup (Optional)

For local deployment with Huawei Pangu:

```bash
# Download model from:
# https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

# Configure in .env
OPENAI_BASE_URL=http://localhost:8000/v1  # Your Pangu serving endpoint
MODEL_NAME=openpangu-embedded-7b
```

## Usage

### Starting the Crawler

**Method 1: Direct execution**
```bash
cd hotsearchcrawler
python run_spiders.py
```

**Method 2: Via web UI**
```bash
# Start main application
python app.py

# Access http://localhost:5000
# Use keyboard shortcut to start crawler (documented in UI)
```

**Test individual spider:**
```bash
cd hotsearchcrawler
scrapy crawl weibo  # Test Weibo crawler
scrapy crawl bilibili  # Test Bilibili crawler
```

### Starting the Analysis System

```bash
# Run main application
python app.py

# Access web interface
# Default: http://localhost:5000
```

### API Examples

**Query hot searches:**
```python
import requests

# Natural language query
response = requests.post('http://localhost:5000/api/query', json={
    'query': '最近关于人工智能的热搜'  # Recent AI hot searches
})
results = response.json()
```

**Create push task:**
```python
import requests

task_config = {
    'task_name': 'AI Technology Monitoring',
    'query_keywords': ['人工智能', 'ChatGPT', '大模型'],
    'push_channels': ['email', 'telegram'],
    'schedule_cron': '0 9 * * *',  # Daily at 9 AM
    'email_recipients': ['alerts@company.com']
}

response = requests.post('http://localhost:5000/api/tasks', json=task_config)
```

**Sentiment analysis:**
```python
from hotsearch_analysis_agent.analyzer import SentimentAnalyzer

analyzer = SentimentAnalyzer()
result = analyzer.analyze_sentiment(
    text='这款产品真的太好用了,非常推荐大家购买!'
)
print(result)
# Output: {'sentiment': 'positive', 'score': 0.92, 'confidence': 0.87}
```

### Common Patterns

**1. Topic Clustering Analysis**

```python
from hotsearch_analysis_agent.clustering import TopicCluster
import mysql.connector

# Fetch recent hot searches
conn = mysql.connector.connect(
    host=os.getenv('MYSQL_HOST'),
    user=os.getenv('MYSQL_USER'),
    password=os.getenv('MYSQL_PASSWORD'),
    database=os.getenv('MYSQL_DB')
)

cursor = conn.cursor(dictionary=True)
cursor.execute("""
    SELECT title, platform, heat, url 
    FROM hot_searches 
    WHERE created_at >= DATE_SUB(NOW(), INTERVAL 24 HOUR)
    ORDER BY heat DESC LIMIT 100
""")
hot_searches = cursor.fetchall()

# Perform clustering
clusterer = TopicCluster()
clusters = clusterer.cluster_topics([item['title'] for item in hot_searches])

for cluster_id, topics in clusters.items():
    print(f"Cluster {cluster_id}: {topics['summary']}")
    print(f"Topics: {topics['items']}\n")
```

**2. Deep Content Extraction**

```python
from hotsearchcrawler.content_extractor import ContentExtractor

extractor = ContentExtractor()

# Extract article content
article_data = extractor.extract_article(
    url='https://example.com/news/article.html'
)
print(article_data)
# {'title': '...', 'content': '...', 'author': '...', 'publish_time': '...'}

# Extract video content (including subtitles/descriptions)
video_data = extractor.extract_video(
    url='https://www.bilibili.com/video/BV1xx411c7mD'
)
print(video_data)
# {'title': '...', 'description': '...', 'transcript': '...'}
```

**3. Multi-Channel Push Implementation**

```python
from hotsearch_analysis_agent.pusher import MultiChannelPusher

pusher = MultiChannelPusher()

report_content = """
## AI Technology Hot Topics - 2026-05-19

### Top Trending
1. GPT-6 context window reaches 2M tokens
2. DeepSeek V4 adopts Huawei Ascend chips
3. China's LLM API calls exceed US for 5 weeks

### Analysis
...
"""

# Send via multiple channels
pusher.send(
    content=report_content,
    channels=['email', 'wechat_work', 'telegram'],
    recipients={
        'email': ['team@company.com'],
        'telegram': [os.getenv('TELEGRAM_CHAT_ID')]
    }
)
```

**4. Scheduled Task Setup**

```python
from hotsearch_analysis_agent.scheduler import TaskScheduler

scheduler = TaskScheduler()

# Create monitoring task
task = scheduler.create_task(
    name='Tech Industry Monitoring',
    keywords=['科技', '芯片', '人工智能'],
    platforms=['weibo', 'zhihu', 'toutiao'],
    analysis_type='sentiment_and_clustering',
    push_channels=['email', 'wechat_work'],
    cron='0 */6 * * *',  # Every 6 hours
    filters={
        'min_heat': 5000,
        'sentiment_threshold': 0.3
    }
)

# Start scheduler
scheduler.start()
```

## Spider Development

To add a new platform spider:

```python
# hotsearchcrawler/spiders/new_platform.py
import scrapy

class NewPlatformSpider(scrapy.Spider):
    name = 'new_platform'
    start_urls = ['https://new-platform.com/hot']

    def parse(self, response):
        for item in response.css('.hot-item'):
            yield {
                'platform': 'new_platform',
                'rank': item.css('.rank::text').get(),
                'title': item.css('.title::text').get(),
                'url': response.urljoin(item.css('a::attr(href)').get()),
                'heat': item.css('.heat::text').get(),
                'label': item.css('.label::text').get()
            }
```

Register in `run_spiders.py`:

```python
SPIDERS = [
    'weibo', 'bilibili', 'zhihu', 'douyin', 
    'baidu', 'toutiao', 'new_platform'  # Add here
]
```

## Troubleshooting

**Crawler not extracting content:**
- Verify ChromeDriver/EdgeDriver version matches browser
- Check if target site requires authentication (add cookies in settings)
- Inspect site's anti-bot measures (adjust `DOWNLOAD_DELAY`, add user-agent rotation)

**LLM analysis returning errors:**
- Verify `OPENAI_API_KEY` and `OPENAI_BASE_URL` in `.env`
- Check API rate limits and quota
- For Pangu model: ensure local serving endpoint is running

**Database connection failures:**
```python
# Test connection
import mysql.connector
conn = mysql.connector.connect(
    host=os.getenv('MYSQL_HOST'),
    user=os.getenv('MYSQL_USER'),
    password=os.getenv('MYSQL_PASSWORD')
)
print(conn.is_connected())
```

**Push notifications not sending:**
- SMTP: Enable "App Passwords" for Gmail (https://myaccount.google.com/apppasswords)
- WeChat Work: Verify webhook URL is active
- Telegram: Test bot token with `https://api.telegram.org/bot<TOKEN>/getMe`

**Encoding issues with Chinese text:**
```python
# Ensure UTF-8 encoding in MySQL
ALTER DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

# In Python
import sys
sys.stdout.reconfigure(encoding='utf-8')
```

## Testing

```bash
# Test crawler
cd hotsearchcrawler
python runspider-test.py

# Test push task
python test_push_task.py

# Run specific spider test
scrapy crawl weibo -o output.json
```

## Performance Optimization

**For large-scale deployment:**

```python
# hotsearchcrawler/settings.py
CONCURRENT_REQUESTS = 32
CONCURRENT_REQUESTS_PER_DOMAIN = 8
AUTOTHROTTLE_ENABLED = True
AUTOTHROTTLE_TARGET_CONCURRENCY = 2.0

# Use Redis for distributed crawling
SCHEDULER = "scrapy_redis.scheduler.Scheduler"
DUPEFILTER_CLASS = "scrapy_redis.dupefilter.RFPDupeFilter"
REDIS_URL = 'redis://localhost:6379'
```

**Database indexing:**

```sql
CREATE INDEX idx_platform_time ON hot_searches(platform, created_at);
CREATE INDEX idx_keywords ON news_content(title(100));
CREATE FULLTEXT INDEX idx_content ON news_content(content);
```

This skill enables AI agents to configure, deploy, and extend a production-grade Chinese public opinion monitoring system with LLM-powered analysis capabilities.
