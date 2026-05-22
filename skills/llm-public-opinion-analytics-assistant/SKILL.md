---
name: llm-public-opinion-analytics-assistant
description: Real-time multi-platform hot topic crawler and sentiment analysis system with LLM-powered clustering and multi-channel push notifications
triggers:
  - analyze public opinion trends across social platforms
  - set up hot topic monitoring and alerts
  - crawl trending topics from weibo douyin and bilibili
  - perform sentiment analysis on news and social media
  - cluster related topics using large language models
  - configure multi-channel notifications for trending topics
  - build a public opinion monitoring dashboard
  - extract content from video news for analysis
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive public opinion monitoring system that crawls 26 trending lists from 15 major Chinese platforms, performs LLM-powered analysis (clustering, sentiment analysis), and sends multi-channel notifications (Email, WeChat, Enterprise WeChat, Telegram).

## What This Project Does

- **Multi-Platform Crawling**: Scrapes trending topics from Weibo, Douyin, Bilibili, Zhihu, Baidu, and 10+ other platforms
- **LLM Analysis**: Uses Huawei Pangu or OpenAI-compatible models for topic clustering and sentiment analysis
- **Content Extraction**: Extracts detailed content from news pages, including video-based news
- **Multi-Channel Push**: Sends alerts via Email (SMTP), Enterprise WeChat (bot/app), Telegram, and personal WeChat
- **Conversational Interface**: Natural language queries for trending topics, theme searches, and analysis
- **Hotkey Control**: Start/stop crawlers via keyboard shortcuts

## Installation

### Prerequisites

**1. Browser Driver Setup**

The project requires ChromeDriver or EdgeDriver for content extraction:

```bash
# Check your browser version
google-chrome --version  # or microsoft-edge --version

# Download matching driver from:
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Place driver in PATH or project root
# Verify installation:
chromedriver --version
```

**2. Database Setup**

Install MySQL and create database:

```sql
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Tables are auto-created by init.py, but key tables include:
-- hot_search_items: stores crawled trending topics
-- news_details: stores extracted news content
-- push_tasks: scheduled notification tasks
```

**3. Python Environment**

```bash
# Clone the repository
git clone https://github.com/hmmnxkl/LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant.git
cd LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant

# Create virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Configuration

**1. Environment Variables (`.env`)**

```bash
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1  # or local endpoint
MODEL_NAME=gpt-4  # or pangu model name

# Crawler Settings
CHROME_DRIVER_PATH=/usr/local/bin/chromedriver
EDGE_DRIVER_PATH=/usr/local/bin/msedgedriver
```

**2. Crawler Settings (`hotsearchcrawler/settings.py`)**

```python
# MySQL connection for crawler
MYSQL_CONFIG = {
    'host': os.getenv('MYSQL_HOST', 'localhost'),
    'port': int(os.getenv('MYSQL_PORT', 3306)),
    'user': os.getenv('MYSQL_USER', 'root'),
    'password': os.getenv('MYSQL_PASSWORD'),
    'database': os.getenv('MYSQL_DATABASE'),
}

# Optional: Platform-specific cookies
COOKIES = {
    'weibo': 'your_weibo_cookies',  # Optional, for better access
    'douyin': 'your_douyin_cookies',
}
```

**3. Push Notification Settings**

Configure in the web interface or via database:

```python
# Email (SMTP)
EMAIL_CONFIG = {
    'smtp_server': 'smtp.gmail.com',
    'smtp_port': 587,
    'sender': os.getenv('EMAIL_SENDER'),
    'password': os.getenv('EMAIL_PASSWORD'),
}

# Enterprise WeChat Bot
WECOM_WEBHOOK = os.getenv('WECOM_WEBHOOK_URL')

# Telegram Bot
TELEGRAM_BOT_TOKEN = os.getenv('TELEGRAM_BOT_TOKEN')
TELEGRAM_CHAT_ID = os.getenv('TELEGRAM_CHAT_ID')
```

## Running the System

### 1. Initialize Database

```bash
python init.py
```

### 2. Start the Web Application

```bash
python app.py
# Access at http://localhost:5000
```

### 3. Run Crawlers

**Via Web Interface:**
- Navigate to the dashboard
- Use hotkeys to start/stop crawlers (configured in UI)

**Manual Execution:**

```bash
# Test single platform crawler
python runspider-test.py --platform weibo

# Run all crawlers
python run_spiders.py
```

### 4. Test Push Notifications

```bash
python test_push_task.py --channel email --topic "AI技术"
```

## Core API & Usage Patterns

### Crawler Architecture

The crawler system (`hotsearchcrawler/`) is completely separate from the analysis system:

```python
# Example: Custom crawler for new platform
from scrapy import Spider
from hotsearchcrawler.items import HotSearchItem

class CustomPlatformSpider(Spider):
    name = 'custom_platform'
    
    def start_requests(self):
        yield scrapy.Request(
            url='https://platform.com/trending',
            callback=self.parse
        )
    
    def parse(self, response):
        for item in response.css('.trending-item'):
            yield HotSearchItem(
                platform='custom_platform',
                title=item.css('.title::text').get(),
                url=item.css('a::attr(href)').get(),
                rank=item.css('.rank::text').get(),
                heat_score=item.css('.heat::text').get(),
                timestamp=datetime.now(),
            )
```

### Analysis Agent Usage

The analysis system (`hotsearch_analysis_agent/`) uses LLMs for intelligent analysis:

```python
from hotsearch_analysis_agent.analyzer import OpinionAnalyzer

# Initialize analyzer
analyzer = OpinionAnalyzer(
    model_name=os.getenv('MODEL_NAME'),
    api_key=os.getenv('OPENAI_API_KEY'),
)

# Query trending topics
results = analyzer.query("过去24小时内人工智能相关的热点")
# Returns: List of relevant trending topics with analysis

# Perform topic clustering
clusters = analyzer.cluster_topics(
    query="科技行业",
    time_range="7d",
)
# Returns: Grouped topics with cluster labels

# Sentiment analysis
sentiment = analyzer.analyze_sentiment(
    topic_id=12345,
    include_details=True,
)
# Returns: {
#     'overall': 'positive',
#     'score': 0.72,
#     'breakdown': {'positive': 0.65, 'neutral': 0.25, 'negative': 0.10},
#     'key_opinions': [...]
# }
```

### Database Interaction

```python
from hotsearch_analysis_agent.database import Database

db = Database()

# Fetch trending topics by platform
topics = db.query("""
    SELECT * FROM hot_search_items 
    WHERE platform = %s 
    AND created_at >= NOW() - INTERVAL 1 DAY
    ORDER BY rank ASC
""", ('weibo',))

# Get detailed news content
details = db.query("""
    SELECT content, extracted_video_info 
    FROM news_details 
    WHERE topic_id = %s
""", (topic_id,))

# Insert custom analysis result
db.execute("""
    INSERT INTO analysis_results 
    (topic_id, sentiment, cluster_label, summary)
    VALUES (%s, %s, %s, %s)
""", (topic_id, 'positive', 'AI Technology', summary_text))
```

### Push Notification System

```python
from hotsearch_analysis_agent.notifier import PushNotifier

notifier = PushNotifier()

# Create scheduled push task
task_id = notifier.create_task(
    query="人工智能 AND 前沿科技",
    channels=['email', 'wecom_bot', 'telegram'],
    schedule='0 9,18 * * *',  # Cron format: 9 AM and 6 PM daily
    recipients=['user@example.com'],
)

# Send immediate notification
report = analyzer.generate_report(
    query="AI热点分析",
    format='markdown',
)

notifier.send(
    channels=['wecom_app', 'telegram'],
    content=report,
    title="每日AI热点分析报告",
)
```

### Content Extraction from Video News

```python
from hotsearch_analysis_agent.extractor import ContentExtractor

extractor = ContentExtractor(
    driver_type='chrome',  # or 'edge'
    headless=True,
)

# Extract content including video metadata
content = extractor.extract(url='https://www.bilibili.com/video/BV1xxxxx')

# Returns:
# {
#     'title': '视频标题',
#     'text_content': '视频描述和字幕',
#     'video_info': {
#         'duration': 300,
#         'views': 100000,
#         'likes': 5000,
#         'tags': ['AI', '科技']
#     },
#     'comments': [...],  # Top comments for sentiment analysis
# }
```

## Common Workflows

### 1. Monitor Specific Topic with Alerts

```python
from hotsearch_analysis_agent import OpinionAnalyzer, PushNotifier

analyzer = OpinionAnalyzer()
notifier = PushNotifier()

# Create monitoring task
task = notifier.create_task(
    query="华为 OR 盘古大模型",
    channels=['email', 'wecom_bot'],
    schedule='0 */3 * * *',  # Every 3 hours
    threshold={
        'min_heat': 50000,  # Only alert if heat score > 50k
        'sentiment': 'negative',  # Alert on negative sentiment
    },
    recipients=['monitor@company.com'],
)

print(f"Created monitoring task: {task['id']}")
```

### 2. Generate Daily Report

```python
from datetime import datetime, timedelta
from hotsearch_analysis_agent import OpinionAnalyzer

analyzer = OpinionAnalyzer()

# Analyze past 24 hours
report = analyzer.generate_report(
    query="人工智能与前沿科技",
    time_range=(
        datetime.now() - timedelta(days=1),
        datetime.now()
    ),
    include_sections=['summary', 'news_list', 'sentiment', 'clusters'],
    format='markdown',
)

# Report structure matches the example in README
# Contains: 核心发现、详细新闻内容、分析总结、信息传播特点
```

### 3. Cross-Platform Topic Tracking

```python
from hotsearch_analysis_agent.database import Database

db = Database()

# Track topic propagation across platforms
query = """
    SELECT platform, COUNT(*) as mentions, AVG(heat_score) as avg_heat
    FROM hot_search_items
    WHERE title LIKE %s
    AND created_at >= NOW() - INTERVAL 12 HOUR
    GROUP BY platform
    ORDER BY mentions DESC
"""

results = db.query(query, ('%DeepSeek%',))

# Analyze cross-platform spread
for row in results:
    print(f"{row['platform']}: {row['mentions']} mentions, avg heat: {row['avg_heat']}")
```

## Configuration Tips

### Using Huawei Pangu Model (Recommended)

```python
# .env configuration for local Pangu deployment
OPENAI_API_BASE=http://localhost:8000/v1  # Local inference server
MODEL_NAME=openpangu-embedded-7b
OPENAI_API_KEY=sk-local  # Dummy key for local deployment

# Download model from:
# https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model
```

### Crawler Optimization

```python
# hotsearchcrawler/settings.py

# Rate limiting to avoid IP blocks
DOWNLOAD_DELAY = 2
CONCURRENT_REQUESTS_PER_DOMAIN = 2

# Retry failed requests
RETRY_TIMES = 3
RETRY_HTTP_CODES = [500, 502, 503, 504, 408, 429]

# User agent rotation
USER_AGENTS = [
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36...',
    'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36...',
]
```

## Troubleshooting

### Crawler Issues

**Problem**: ChromeDriver version mismatch
```bash
# Check versions match
google-chrome --version
chromedriver --version

# Download matching version from https://chromedriver.chromium.org/
```

**Problem**: Platform returns 403/blocked
```python
# Add platform-specific cookies in hotsearchcrawler/settings.py
COOKIES = {
    'weibo': 'SUB=xxx; SUBP=xxx',  # Export from browser DevTools
}

# Or reduce request frequency
DOWNLOAD_DELAY = 5
```

### Database Connection

**Problem**: MySQL connection refused
```bash
# Check MySQL is running
sudo systemctl status mysql

# Verify credentials
mysql -u root -p -e "SHOW DATABASES;"

# Check .env file has correct credentials
cat .env | grep MYSQL
```

### LLM Analysis Issues

**Problem**: OpenAI API rate limit exceeded
```python
# Add retry logic in analyzer configuration
from tenacity import retry, wait_exponential

@retry(wait=wait_exponential(multiplier=1, min=4, max=60))
def analyze_with_retry(self, text):
    return self.llm.analyze(text)
```

**Problem**: Pangu model out of memory
```bash
# Reduce batch size in analysis config
ANALYSIS_BATCH_SIZE=5  # Default is 10

# Or use smaller context window
MAX_CONTEXT_LENGTH=4096  # Reduce from 8192
```

### Push Notification Failures

**Problem**: Email not sending
```python
# Test SMTP connection
import smtplib

server = smtplib.SMTP('smtp.gmail.com', 587)
server.starttls()
server.login(os.getenv('EMAIL_SENDER'), os.getenv('EMAIL_PASSWORD'))
# If this fails, check: 2FA settings, app-specific passwords, firewall
```

**Problem**: Enterprise WeChat webhook 400 error
```python
# Verify webhook URL format
# Should be: https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxxxx

# Test with minimal payload
import requests
requests.post(os.getenv('WECOM_WEBHOOK_URL'), json={
    'msgtype': 'text',
    'text': {'content': 'Test message'}
})
```

## Advanced Usage

### Custom Analysis Prompts

```python
# Customize LLM prompts for specific domains
custom_prompt = """
分析以下科技新闻的技术深度和商业影响:
{news_content}

请从以下维度评估:
1. 技术创新程度 (1-10)
2. 商业应用前景 (1-10)
3. 对行业的影响范围
4. 潜在风险因素
"""

analyzer.set_custom_prompt('tech_analysis', custom_prompt)
result = analyzer.analyze(topic_id, prompt_type='tech_analysis')
```

### Extending Platform Support

```python
# Add new platform crawler
# 1. Create spider in hotsearchcrawler/spiders/new_platform.py
# 2. Define item structure in hotsearchcrawler/items.py
# 3. Add pipeline processing if needed
# 4. Register in run_spiders.py

SPIDER_MODULES = ['hotsearchcrawler.spiders']
# System will auto-discover new spiders
```

This skill provides comprehensive coverage of the LLM-Based Public Opinion Analytics Assistant for AI coding agents to effectively help developers deploy, configure, and extend the system.
