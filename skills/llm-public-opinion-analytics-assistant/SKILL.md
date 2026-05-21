---
name: llm-public-opinion-analytics-assistant
description: Multi-platform hot search monitoring and LLM-based sentiment analysis system with automated reporting and multi-channel notifications
triggers:
  - set up public opinion monitoring system
  - analyze hot topics from chinese social platforms
  - create sentiment analysis crawler for weibo douyin bilibili
  - monitor trending topics across multiple platforms
  - build llm-powered news analysis tool
  - deploy multi-platform hot search aggregator
  - implement automated sentiment reporting system
  - configure public opinion analytics with push notifications
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is a comprehensive public opinion analytics system that aggregates hot search data from **15 mainstream Chinese platforms** (26 ranking lists total) and combines them with LLM analysis capabilities. It provides conversational hot search queries, topic clustering, sentiment analysis, and automated multi-channel reporting (Email, WeChat, Enterprise WeChat, Telegram).

**Key capabilities:**
- Real-time crawling of hot topics from Weibo, Douyin, Bilibili, Baidu, Zhihu, etc.
- LLM-powered topic clustering and sentiment analysis
- Natural language query interface for hot search data
- Automated report generation with customizable push schedules
- Multi-channel notification support (4 channels)
- Video content extraction and analysis

## Installation

### Prerequisites

1. **Browser Driver Setup** (required for news detail scraping):

```bash
# For Chrome
# Download ChromeDriver matching your Chrome version from:
# https://chromedriver.chromium.org/

# For Edge
# Download EdgeDriver from:
# https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Place driver in system PATH or browser installation directory
# Verify installation:
chromedriver --version
# or
msedgedriver --version
```

2. **MySQL Database**:

```bash
# Install MySQL 5.7+ and create database
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

Use the reference `init.py` to create tables:

```python
# Example database schema initialization
import pymysql

connection = pymysql.connect(
    host='localhost',
    user='root',
    password='your_password',
    database='hotsearch_db',
    charset='utf8mb4'
)

with connection.cursor() as cursor:
    # Hot search data table
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS hot_searches (
            id INT AUTO_INCREMENT PRIMARY KEY,
            platform VARCHAR(50),
            rank_list VARCHAR(100),
            title VARCHAR(500),
            url VARCHAR(1000),
            hot_value VARCHAR(100),
            fetch_time DATETIME,
            detail_content TEXT,
            INDEX idx_platform (platform),
            INDEX idx_fetch_time (fetch_time)
        )
    ''')
    
    # Analysis results table
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS analysis_results (
            id INT AUTO_INCREMENT PRIMARY KEY,
            query_text TEXT,
            cluster_result TEXT,
            sentiment_result TEXT,
            created_at DATETIME,
            INDEX idx_created_at (created_at)
        )
    ''')
    
    connection.commit()
```

## Configuration

### 1. Crawler Configuration

Edit `hotsearchcrawler/settings.py`:

```python
# MySQL Configuration
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'root'
MYSQL_PASSWORD = 'your_password'
MYSQL_DB = 'hotsearch_db'

# Optional: Platform-specific cookies (for platforms requiring authentication)
COOKIES = {
    'weibo': 'your_weibo_cookie_string',
    'douyin': 'your_douyin_cookie_string',
}

# Crawler settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
ROBOTSTXT_OBEY = False
```

### 2. Analysis System Configuration

Create `.env` file in project root:

```bash
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=your_password
MYSQL_DB=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
LLM_API_BASE=https://your-llm-endpoint.com/v1
LLM_API_KEY=your_api_key_here
LLM_MODEL=pangu-embedded-7b  # or gpt-4, deepseek-v4, etc.

# Push Notification Channels
# Enterprise WeChat Robot
WECHAT_ROBOT_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=your_key

# Enterprise WeChat Application
WECHAT_CORP_ID=your_corp_id
WECHAT_APP_SECRET=your_app_secret
WECHAT_AGENT_ID=your_agent_id

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
SMTP_RECIPIENT=recipient@example.com
```

## Usage

### Starting the System

```bash
# Start the web application (includes crawler control interface)
python app.py

# Or test crawler independently
python runspider-test.py

# Or run specific spider
python run_spiders.py --spider weibo_hot
```

The web interface will be available at `http://localhost:5000`

### Conversational Query Interface

```python
# Example: Query hot topics via Python API
from hotsearch_analysis_agent import AnalysisAgent

agent = AnalysisAgent()

# Natural language query
response = agent.query("最近关于人工智能的热点新闻有哪些?")
print(response)

# Topic clustering
clusters = agent.cluster_topics(
    query="科技新闻",
    days=7,
    min_cluster_size=3
)

# Sentiment analysis
sentiment = agent.analyze_sentiment(
    query="华为盘古大模型",
    platforms=["weibo", "zhihu", "bilibili"]
)
```

### Crawler Control

```python
# Start all crawlers
from hotsearchcrawler import SpiderManager

manager = SpiderManager()

# Start specific platforms
manager.start_spiders(['weibo_hot', 'douyin_hot', 'bilibili_hot'])

# Stop all crawlers
manager.stop_all()

# Get crawler status
status = manager.get_status()
```

### Automated Push Tasks

Create and schedule push tasks:

```python
# test_push_task.py
from hotsearch_analysis_agent.push import PushTaskManager

task_manager = PushTaskManager()

# Create daily report task
task_manager.create_task(
    name="daily_ai_report",
    query="人工智能与前沿科技",
    schedule="0 12 * * *",  # Daily at 12:00 PM
    channels=["wechat_robot", "telegram", "email"],
    platforms=["weibo", "zhihu", "bilibili", "toutiao"],
    days_range=1,
    min_relevance_score=0.7
)

# Create weekly industry report
task_manager.create_task(
    name="weekly_tech_digest",
    query="科技行业动态",
    schedule="0 18 * * 5",  # Every Friday at 6:00 PM
    channels=["wechat_app", "email"],
    enable_clustering=True,
    enable_sentiment=True
)

# Start task scheduler
task_manager.start_scheduler()
```

### LLM Analysis Integration

```python
# Using Pangu model for analysis (recommended)
from hotsearch_analysis_agent.llm import PanguAnalyzer

analyzer = PanguAnalyzer(
    api_base=os.getenv('LLM_API_BASE'),
    api_key=os.getenv('LLM_API_KEY'),
    model='pangu-embedded-7b'
)

# Generate comprehensive report
report = analyzer.generate_report(
    hot_searches=query_results,
    topic="人工智能发展趋势",
    include_sentiment=True,
    include_clustering=True
)

# Extract key insights
insights = analyzer.extract_insights(
    content=news_detail_content,
    focus_areas=["技术突破", "商业应用", "行业影响"]
)
```

## Common Patterns

### Pattern 1: Multi-Platform Hot Search Aggregation

```python
from hotsearch_analysis_agent import HotSearchAggregator

aggregator = HotSearchAggregator()

# Get top 10 from all platforms
all_hot = aggregator.get_top_n(
    n=10,
    platforms=["weibo", "douyin", "bilibili", "zhihu", "baidu"],
    time_range_hours=1
)

# Deduplicate similar topics
unique_topics = aggregator.deduplicate(all_hot, similarity_threshold=0.85)

# Rank by cross-platform popularity
ranked = aggregator.rank_by_cross_platform_heat(unique_topics)
```

### Pattern 2: Topic Clustering with LLM

```python
from hotsearch_analysis_agent import TopicClusterer

clusterer = TopicClusterer(llm_analyzer=analyzer)

# Cluster topics from last 3 days
clusters = clusterer.cluster(
    query="科技新闻",
    days=3,
    method="semantic",  # or "keyword"
    min_cluster_size=2
)

for cluster_id, topics in clusters.items():
    print(f"\n集群 {cluster_id}: {topics['summary']}")
    for topic in topics['items']:
        print(f"  - {topic['title']} ({topic['platform']})")
```

### Pattern 3: Sentiment Trend Analysis

```python
from hotsearch_analysis_agent import SentimentAnalyzer

sentiment_analyzer = SentimentAnalyzer(llm_analyzer=analyzer)

# Analyze sentiment trend over time
trend = sentiment_analyzer.analyze_trend(
    topic="华为盘古大模型",
    platforms=["weibo", "zhihu"],
    days=7,
    granularity="daily"
)

# Output: {
#   "2026-05-12": {"positive": 0.65, "neutral": 0.25, "negative": 0.10},
#   "2026-05-13": {"positive": 0.70, "neutral": 0.22, "negative": 0.08},
#   ...
# }
```

### Pattern 4: Custom Report Generation

```python
from hotsearch_analysis_agent import ReportGenerator

generator = ReportGenerator(llm_analyzer=analyzer)

report = generator.generate(
    template="industry_analysis",  # or "daily_digest", "weekly_summary"
    query="人工智能行业动态",
    platforms=["weibo", "zhihu", "bilibili", "toutiao"],
    days=7,
    sections=[
        "核心发现与数据亮点",
        "详细新闻内容梳理",
        "分析与总结",
        "信息传播特点"
    ],
    max_news_items=5,
    include_charts=True
)

# Save report
with open("report.md", "w", encoding="utf-8") as f:
    f.write(report)
```

## Platform-Specific Crawlers

Available spiders (26 ranking lists from 15 platforms):

```python
AVAILABLE_SPIDERS = {
    'weibo': ['weibo_hot', 'weibo_realtime'],
    'douyin': ['douyin_hot', 'douyin_entertainment'],
    'bilibili': ['bilibili_hot', 'bilibili_weekly'],
    'zhihu': ['zhihu_hot'],
    'baidu': ['baidu_hot', 'baidu_realtime'],
    'toutiao': ['toutiao_hot'],
    'kuaishou': ['kuaishou_hot'],
    'xiaohongshu': ['xiaohongshu_hot'],
    'tieba': ['tieba_hot'],
    '36kr': ['36kr_hot'],
    'ithome': ['ithome_hot'],
    'cls': ['cls_telegraph'],
    'eastmoney': ['eastmoney_hot'],
    'gelonghui': ['gelonghui_hot'],
    'wallstreetcn': ['wallstreetcn_hot']
}
```

## Troubleshooting

### Browser Driver Issues

```bash
# Error: ChromeDriver version mismatch
# Solution: Verify browser and driver versions match
google-chrome --version
chromedriver --version

# Error: Driver not found in PATH
# Solution: Add to PATH or specify explicitly
export PATH=$PATH:/path/to/driver/directory
```

### Database Connection Issues

```python
# Test MySQL connection
import pymysql

try:
    conn = pymysql.connect(
        host='localhost',
        user='root',
        password='your_password',
        database='hotsearch_db'
    )
    print("Database connection successful")
    conn.close()
except Exception as e:
    print(f"Connection failed: {e}")
```

### LLM API Issues

```python
# Test LLM endpoint
import requests

response = requests.post(
    f"{os.getenv('LLM_API_BASE')}/chat/completions",
    headers={"Authorization": f"Bearer {os.getenv('LLM_API_KEY')}"},
    json={
        "model": os.getenv('LLM_MODEL'),
        "messages": [{"role": "user", "content": "测试"}]
    }
)
print(f"Status: {response.status_code}")
print(f"Response: {response.json()}")
```

### Cookie Expiration (Platform Authentication)

```python
# Refresh cookies manually or use automated cookie refresh
from hotsearchcrawler.utils import CookieRefresher

refresher = CookieRefresher()
new_cookies = refresher.refresh_platform_cookies('weibo')
# Update settings.py with new cookies
```

### Memory Issues with Large Datasets

```python
# Use pagination for large queries
from hotsearch_analysis_agent import DataLoader

loader = DataLoader()
for batch in loader.load_in_batches(
    query="科技新闻",
    days=30,
    batch_size=1000
):
    process_batch(batch)
```

## Advanced Configuration

### Custom LLM Model Integration

```python
# Add support for custom LLM models
from hotsearch_analysis_agent.llm import BaseLLMAnalyzer

class CustomLLMAnalyzer(BaseLLMAnalyzer):
    def __init__(self, model_path):
        self.model = self.load_model(model_path)
    
    def analyze(self, text, task_type):
        # Implement custom analysis logic
        return self.model.generate(text, task=task_type)

# Use custom analyzer
analyzer = CustomLLMAnalyzer(model_path="/path/to/your/model")
```

### Performance Optimization

```python
# Enable caching for frequent queries
from hotsearch_analysis_agent import CacheManager

cache = CacheManager(backend='redis', ttl=3600)

@cache.cached(key_prefix='hot_search')
def get_hot_searches(platform, hours=1):
    return fetch_from_db(platform, hours)
```
