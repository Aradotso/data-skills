---
name: llm-public-opinion-analytics-assistant
description: A comprehensive Chinese public opinion monitoring system with 26 real-time rankings from 15 platforms, LLM-powered analysis, and multi-channel alerting
triggers:
  - how do I set up the public opinion monitoring system
  - analyze hot topics from Chinese social media platforms
  - configure multi-platform web scraping for trending news
  - set up sentiment analysis with LLM models
  - create automated alerts for trending topics
  - monitor weibo douyin bilibili hot searches
  - deploy opinion analytics with Pangu model
  - build a trending topics dashboard with scrapy
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This is a comprehensive Chinese public opinion monitoring and analysis system that integrates:

- **26 real-time ranking lists** from **15 major Chinese platforms** (Weibo, Douyin, Bilibili, Baidu, Zhihu, Toutiao, etc.)
- **LLM-powered analysis** (optimized for Huawei Pangu model, supports OpenAI-compatible APIs)
- **Conversational interface** for natural language queries about trending topics
- **Topic clustering and sentiment analysis**
- **Multi-channel push notifications** (Email, WeChat Work, Telegram)
- **Scrapy-based distributed crawler cluster**
- **Video content extraction** capabilities

## Project Structure

```
project/
├── app.py                          # Main application entry point
├── hotsearch_analysis_agent/       # Analysis system (LLM integration)
│   ├── agent/                      # LLM agent logic
│   ├── database/                   # Database models
│   ├── utils/                      # Helper utilities
│   └── .env                        # Configuration file
├── hotsearchcrawler/               # Crawler cluster (completely separate)
│   ├── hotsearchcrawler/
│   │   ├── spiders/               # Platform-specific spiders
│   │   ├── settings.py            # Scrapy settings
│   │   └── pipelines.py           # Data processing pipelines
│   └── run_spiders.py             # Crawler launcher
├── init.py                         # Database initialization
├── test_push_task.py              # Push notification testing
└── requirements.txt
```

## Installation

### 1. Prerequisites

#### Browser Driver Setup (Required for Detail Scraping)

1. **Check your browser version**:
   - Open Chrome/Edge → Settings → About
   - Note the version number (e.g., `115.0.5790.102`)

2. **Download matching driver**:
   - Chrome: https://chromedriver.chromium.org/
   - Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

3. **Install driver**:
   ```bash
   # macOS/Linux: Move to system PATH
   sudo mv chromedriver /usr/local/bin/
   sudo chmod +x /usr/local/bin/chromedriver
   
   # Windows: Add driver folder to PATH environment variable
   # Or place in C:\Windows\System32\
   ```

4. **Verify installation**:
   ```bash
   chromedriver --version
   # Should output: ChromeDriver 115.0.5790.102
   ```

#### Database Setup

```bash
# Install MySQL 5.7+
# Create database
mysql -u root -p
CREATE DATABASE hotsearch_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### 2. Python Environment

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### 3. Database Initialization

```python
# Run init.py to create tables
python init.py
```

The script creates these key tables:
- `hot_search_items`: Stores trending topics
- `news_details`: Detailed content from crawled pages
- `analysis_results`: LLM analysis outputs
- `push_tasks`: Scheduled notification tasks

## Configuration

### Analysis System Configuration (`hotsearch_analysis_agent/.env`)

```bash
# Database
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch_db

# LLM Configuration (OpenAI-compatible format)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1  # Or your custom endpoint
MODEL_NAME=gpt-4  # Or pangu-embedded-7b for local deployment

# Push Notifications
# Email (SMTP)
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=${EMAIL_USER}
SMTP_PASSWORD=${EMAIL_PASSWORD}

# WeChat Work Bot
WECHAT_WORK_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Telegram Bot
TELEGRAM_BOT_TOKEN=${TELEGRAM_TOKEN}
TELEGRAM_CHAT_ID=${TELEGRAM_CHAT_ID}
```

### Crawler Configuration (`hotsearchcrawler/hotsearchcrawler/settings.py`)

```python
# Database connection
MYSQL_CONFIG = {
    'host': 'localhost',
    'port': 3306,
    'user': 'your_user',
    'password': 'your_password',
    'database': 'hotsearch_db',
    'charset': 'utf8mb4'
}

# Scrapy settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
RANDOMIZE_DOWNLOAD_DELAY = True

# Optional: Platform cookies for authenticated access
COOKIES_WEIBO = {
    'SUB': 'your_cookie_here'
}
```

## Usage

### Starting the Application

```bash
# Start main application
python app.py

# Application runs at http://localhost:5000
```

### Starting Crawlers

```python
# Via web interface: Click "Start Crawlers" button
# Or manually:
python hotsearchcrawler/run_spiders.py

# Test single spider
cd hotsearchcrawler
scrapy crawl weibo_hot  # For Weibo hot search
scrapy crawl douyin_hot  # For Douyin trending
scrapy crawl bilibili_hot  # For Bilibili rankings
```

### Supported Platforms

The crawler cluster includes spiders for:
- **Social Media**: Weibo, Douyin, Xiaohongshu
- **Video**: Bilibili, Xigua, Kuaishou
- **News**: Toutiao, Baidu, Zhihu, Sina
- **Tech**: 36Kr, iHeima, Tencent News
- **Others**: Baidu Tieba, Weibo Trending, etc.

### API Integration Examples

#### Querying Hot Topics

```python
from hotsearch_analysis_agent.database.db_manager import DBManager

db = DBManager()

# Get latest hot topics from all platforms
topics = db.get_latest_hot_topics(limit=50)
for topic in topics:
    print(f"{topic['platform']}: {topic['title']} (热度: {topic['hot_value']})")

# Query specific platform
weibo_topics = db.get_hot_topics_by_platform('weibo', hours=24)

# Search by keyword
ai_topics = db.search_topics(keyword='人工智能', days=7)
```

#### LLM Analysis

```python
from hotsearch_analysis_agent.agent.llm_client import LLMClient
from hotsearch_analysis_agent.agent.analyzer import OpinionAnalyzer

# Initialize LLM client
llm = LLMClient(
    api_key=os.getenv('OPENAI_API_KEY'),
    base_url=os.getenv('OPENAI_API_BASE'),
    model=os.getenv('MODEL_NAME')
)

# Initialize analyzer
analyzer = OpinionAnalyzer(llm_client=llm, db_manager=db)

# Analyze topics by keyword
analysis = analyzer.analyze_by_keyword(
    keyword="人工智能",
    days=3,
    include_sentiment=True,
    include_clustering=True
)

print(analysis['summary'])
print(f"Sentiment: {analysis['sentiment']['overall']}")
print(f"Topic clusters: {len(analysis['clusters'])}")
```

#### Sentiment Analysis

```python
# Analyze sentiment of specific topic
result = analyzer.analyze_sentiment(topic_id=12345)

print(f"Overall sentiment: {result['sentiment']}")  # positive/negative/neutral
print(f"Confidence: {result['confidence']}")
print(f"Key phrases: {result['key_phrases']}")
```

#### Topic Clustering

```python
# Cluster related topics
clusters = analyzer.cluster_topics(
    keyword="科技",
    days=7,
    min_cluster_size=3
)

for cluster in clusters:
    print(f"\nCluster: {cluster['label']}")
    print(f"Topics: {len(cluster['topics'])}")
    for topic in cluster['topics']:
        print(f"  - {topic['title']}")
```

### Setting Up Push Notifications

```python
from hotsearch_analysis_agent.utils.push_manager import PushManager

push_mgr = PushManager()

# Create scheduled push task
task = push_mgr.create_task(
    name="AI技术热点监控",
    keyword="人工智能",
    schedule_time="12:00",  # Daily at 12:00
    channels=["email", "wechat_work", "telegram"],
    recipients={
        "email": ["user@example.com"],
        "wechat_work": "webhook_key",
        "telegram": "chat_id"
    }
)

# Test push immediately
push_mgr.test_push(task_id=task['id'])
```

#### Email Push Configuration

```python
from hotsearch_analysis_agent.utils.email_sender import EmailSender

sender = EmailSender(
    smtp_server=os.getenv('SMTP_SERVER'),
    smtp_port=int(os.getenv('SMTP_PORT')),
    username=os.getenv('SMTP_USER'),
    password=os.getenv('SMTP_PASSWORD')
)

# Send analysis report
sender.send_report(
    recipients=["user@example.com"],
    subject="舆情分析报告 - AI与前沿科技",
    analysis_result=analysis,
    include_charts=True
)
```

#### WeChat Work Push

```python
from hotsearch_analysis_agent.utils.wechat_pusher import WeChatPusher

wechat = WeChatPusher(webhook_url=os.getenv('WECHAT_WORK_WEBHOOK'))

# Send markdown message
wechat.send_markdown(
    title="热点预警",
    content=f"""
## 🔥 AI技术新突破
> **GPT-6提前曝光,200万超长上下文**

[查看详情](https://example.com/news/123)
    """
)
```

#### Telegram Push

```python
from hotsearch_analysis_agent.utils.telegram_pusher import TelegramPusher

telegram = TelegramPusher(
    bot_token=os.getenv('TELEGRAM_BOT_TOKEN'),
    chat_id=os.getenv('TELEGRAM_CHAT_ID')
)

# Send alert
telegram.send_message(
    text="⚠️ 检测到突发热点:\n\n人工智能相关话题热度激增300%",
    parse_mode="Markdown"
)
```

## Working with Huawei Pangu Model

### Local Deployment

```python
# Download model from: https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

from transformers import AutoModelForCausalLM, AutoTokenizer

model_path = "/path/to/openpangu-embedded-7b"

tokenizer = AutoTokenizer.from_pretrained(model_path, trust_remote_code=True)
model = AutoModelForCausalLM.from_pretrained(
    model_path,
    trust_remote_code=True,
    device_map="auto"
)

# Use with OpenAI-compatible wrapper
from hotsearch_analysis_agent.agent.local_llm_server import LocalLLMServer

server = LocalLLMServer(model=model, tokenizer=tokenizer)
server.start(host="127.0.0.1", port=8000)

# Configure .env to use local endpoint
# OPENAI_API_BASE=http://127.0.0.1:8000/v1
# MODEL_NAME=pangu-embedded-7b
```

### Analysis Prompt Optimization for Pangu

```python
# Pangu performs better with structured Chinese prompts
PANGU_ANALYSIS_PROMPT = """
请分析以下舆情数据:

【数据来源】: {platforms}
【时间范围】: {time_range}
【关键词】: {keyword}

【热搜话题】:
{topics}

请从以下维度进行分析:
1. 核心事件梳理
2. 情感倾向判断(正面/负面/中性)
3. 话题关联性分析
4. 传播特征总结
5. 风险评估与建议

输出格式: JSON
"""

response = llm.chat(
    messages=[{"role": "user", "content": PANGU_ANALYSIS_PROMPT.format(**params)}],
    temperature=0.3
)
```

## Common Patterns

### Real-time Monitoring Dashboard

```python
import streamlit as st
from datetime import datetime, timedelta

# Refresh every 5 minutes
st.set_page_config(page_title="舆情监控", layout="wide")

@st.cache_data(ttl=300)
def load_latest_topics():
    db = DBManager()
    return db.get_latest_hot_topics(limit=100)

topics = load_latest_topics()

# Display by platform
platforms = set(t['platform'] for t in topics)
for platform in platforms:
    st.subheader(f"📊 {platform}")
    platform_topics = [t for t in topics if t['platform'] == platform]
    for topic in platform_topics[:10]:
        st.write(f"**{topic['rank']}. {topic['title']}** - 热度: {topic['hot_value']}")
```

### Automated Alert System

```python
from apscheduler.schedulers.background import BackgroundScheduler

scheduler = BackgroundScheduler()

def check_breaking_news():
    db = DBManager()
    analyzer = OpinionAnalyzer(llm, db)
    
    # Check for sudden spikes
    trending = db.get_trending_topics(spike_threshold=300)
    
    if trending:
        analysis = analyzer.quick_analysis(trending)
        
        # Send alerts
        push_mgr = PushManager()
        push_mgr.send_alert(
            title="突发热点预警",
            content=analysis['summary'],
            urgency="high"
        )

# Run every 10 minutes
scheduler.add_job(check_breaking_news, 'interval', minutes=10)
scheduler.start()
```

### Custom Spider Development

```python
# hotsearchcrawler/hotsearchcrawler/spiders/custom_spider.py
import scrapy
from ..items import HotSearchItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    
    def start_requests(self):
        yield scrapy.Request(
            url='https://api.example.com/trending',
            callback=self.parse
        )
    
    def parse(self, response):
        data = response.json()
        
        for idx, item in enumerate(data['items'], 1):
            yield HotSearchItem(
                platform='custom_platform',
                title=item['title'],
                url=item['url'],
                hot_value=item.get('heat', 0),
                rank=idx,
                collected_at=datetime.now()
            )
```

## Troubleshooting

### Crawler Issues

**Problem**: Spiders fail with timeout errors

```python
# Increase timeout in settings.py
DOWNLOAD_TIMEOUT = 30
RETRY_TIMES = 5

# Use proxy if blocked
DOWNLOADER_MIDDLEWARES = {
    'scrapy.downloadermiddlewares.httpproxy.HttpProxyMiddleware': 110,
}
PROXIES = ['http://proxy1:8080', 'http://proxy2:8080']
```

**Problem**: Platform returns empty results

```bash
# Check if cookies are required
# Update cookies in settings.py or use selenium middleware
```

### LLM Analysis Issues

**Problem**: Analysis results are inconsistent

```python
# Lower temperature for more deterministic outputs
llm_client = LLMClient(
    model=MODEL_NAME,
    temperature=0.1,  # More deterministic
    top_p=0.9
)

# Add system prompt for consistent formatting
SYSTEM_PROMPT = "你是一个专业的舆情分析助手,请用JSON格式输出分析结果。"
```

**Problem**: Out of memory with Pangu model

```python
# Use 4-bit quantization
from transformers import BitsAndBytesConfig

quantization_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_compute_dtype=torch.float16
)

model = AutoModelForCausalLM.from_pretrained(
    model_path,
    quantization_config=quantization_config,
    device_map="auto"
)
```

### Database Performance

**Problem**: Slow queries with large dataset

```sql
-- Add indexes
CREATE INDEX idx_platform_time ON hot_search_items(platform, collected_at);
CREATE INDEX idx_keyword ON hot_search_items(title) USING FULLTEXT;
CREATE INDEX idx_hot_value ON hot_search_items(hot_value DESC);

-- Archive old data
DELETE FROM hot_search_items WHERE collected_at < DATE_SUB(NOW(), INTERVAL 90 DAY);
```

### Push Notification Failures

**Problem**: WeChat Work webhook returns 400

```python
# Ensure content is properly encoded
import json

payload = {
    "msgtype": "markdown",
    "markdown": {
        "content": content.encode('utf-8').decode('utf-8')
    }
}
```

**Problem**: Email stuck in spam

```python
# Add proper headers
headers = {
    'From': f"舆情系统 <{SMTP_USER}>",
    'Reply-To': SMTP_USER,
    'X-Mailer': 'HotSearch Analytics System'
}
```

## Advanced Features

### Video Content Extraction

```python
# The system can extract text from video news
from hotsearch_analysis_agent.utils.video_processor import VideoProcessor

processor = VideoProcessor()

# Extract subtitles and audio transcription
video_content = processor.extract_content(video_url)
print(video_content['title'])
print(video_content['description'])
print(video_content['transcript'])  # From ASR or subtitles
```

### Multi-keyword Monitoring

```python
# Monitor multiple keywords simultaneously
keywords = ["人工智能", "新能源", "半导体"]
results = {}

for keyword in keywords:
    results[keyword] = analyzer.analyze_by_keyword(
        keyword=keyword,
        days=1,
        min_relevance=0.7
    )

# Generate comparative report
report = analyzer.generate_comparison_report(results)
```

This skill provides comprehensive coverage of the LLM-Based Intelligent Public Opinion Analytics Assistant, enabling AI coding agents to help developers deploy, configure, and extend this sophisticated Chinese social media monitoring system.
