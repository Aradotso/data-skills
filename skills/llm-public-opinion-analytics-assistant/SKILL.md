---
name: llm-public-opinion-analytics-assistant
description: Deploy and use an intelligent public opinion analytics assistant that aggregates 26 trending lists from 15 platforms with LLM analysis for sentiment, clustering, and multi-channel push notifications
triggers:
  - set up public opinion monitoring system
  - analyze trending topics across platforms
  - configure sentiment analysis for social media
  - deploy Chinese social media analytics with LLM
  - monitor and cluster hot search trends
  - send trending topic alerts to wechat
  - scrape and analyze weibo bilibili trends
  - build multi-platform opinion monitoring dashboard
---

# LLM-Based Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is an intelligent public opinion analytics assistant that combines real-time data from 26 trending lists across 15 mainstream Chinese platforms (Weibo, Bilibili, Zhihu, Douyin, etc.) with large language model analysis capabilities. It provides conversational queries, topic clustering, sentiment analysis, and multi-channel push notifications (WeChat, Telegram, Email).

## What This Project Does

- **Multi-Platform Data Aggregation**: Scrapes 26 trending lists from 15 platforms including Weibo, Bilibili, Zhihu, Douyin, Baidu, Toutiao
- **LLM-Powered Analysis**: Uses large language models (supports Huawei Pangu, OpenAI-compatible APIs) for:
  - Topic clustering and correlation analysis
  - Sentiment tendency detection
  - Detailed content extraction (including video content)
- **Interactive Web Interface**: Natural language queries for trending topics, platform-specific searches, and analysis
- **Multi-Channel Notifications**: Push alerts via Enterprise WeChat, Telegram, Email (SMTP)
- **Crawler Control**: Start/stop web scrapers via keyboard shortcuts

## Installation

### Prerequisites

1. **Python Environment**: Python 3.8+
2. **MySQL Database**: For storing scraped data
3. **Browser Driver**: ChromeDriver or EdgeDriver for detailed content scraping

#### Browser Driver Setup

Download the driver matching your browser version:

- **Chrome**: [ChromeDriver Downloads](https://chromedriver.chromium.org/)
- **Edge**: [EdgeDriver Downloads](https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/)

Place the driver in your system PATH or browser installation directory.

Verify installation:

```bash
chromedriver --version
# or
msedgedriver --version
```

### Installation Steps

```bash
# Clone the repository
git clone https://github.com/hmmnxkl/LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant.git
cd LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant

# Create and activate virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Database Setup

```python
# Reference init.py for database schema
import mysql.connector

# Create database connection
conn = mysql.connector.connect(
    host=os.getenv("MYSQL_HOST", "localhost"),
    user=os.getenv("MYSQL_USER", "root"),
    password=os.getenv("MYSQL_PASSWORD"),
    database=os.getenv("MYSQL_DATABASE", "hotsearch")
)

# Execute schema creation from init.py
# Tables include: hot_search_items, analysis_results, push_tasks, etc.
```

## Configuration

### Environment Variables (.env)

Create a `.env` file in the project root:

```env
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=your_mysql_password
MYSQL_DATABASE=hotsearch

# LLM API Configuration (OpenAI-compatible format)
OPENAI_API_KEY=your_api_key_here
OPENAI_API_BASE=https://api.openai.com/v1
OPENAI_MODEL=gpt-4

# Alternative: Huawei Pangu Model (if using local deployment)
# PANGU_MODEL_PATH=/path/to/pangu/model

# Push Notification Channels
# Enterprise WeChat Robot
WECHAT_ROBOT_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_telegram_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
SMTP_TO=recipient@example.com
```

### Crawler Configuration (hotsearchcrawler/settings.py)

```python
# MySQL connection for scrapers
MYSQL_CONFIG = {
    'host': os.getenv('MYSQL_HOST', 'localhost'),
    'port': int(os.getenv('MYSQL_PORT', 3306)),
    'user': os.getenv('MYSQL_USER', 'root'),
    'password': os.getenv('MYSQL_PASSWORD'),
    'database': os.getenv('MYSQL_DATABASE', 'hotsearch'),
}

# Optional: Platform-specific cookies for authenticated scraping
# Add cookies for platforms requiring login (Weibo, Bilibili, etc.)
COOKIES = {
    'weibo': 'your_weibo_cookies_here',
    # Add other platforms as needed
}
```

## Running the Application

### Start the Web Interface

```bash
python app.py
```

The web interface will be available at `http://localhost:5000`

### Start Crawlers

#### Via Web Interface
Use the keyboard shortcut or button in the UI to start/stop crawlers.

#### Via Command Line

```bash
# Test single spider
python runspider-test.py

# Run all spiders
python run_spiders.py
```

### Test Push Notifications

```bash
python test_push_task.py
```

## Key Usage Patterns

### Query Trending Topics via Web Interface

The web interface supports natural language queries:

- "Show me today's Weibo trending topics"
- "What's trending on Bilibili about AI?"
- "Analyze sentiment for topics about technology"
- "Cluster news about specific keywords"

### Programmatic Access to Data

```python
import mysql.connector
from datetime import datetime

# Connect to database
conn = mysql.connector.connect(
    host=os.getenv("MYSQL_HOST"),
    user=os.getenv("MYSQL_USER"),
    password=os.getenv("MYSQL_PASSWORD"),
    database=os.getenv("MYSQL_DATABASE")
)
cursor = conn.cursor(dictionary=True)

# Query recent trending topics from specific platform
cursor.execute("""
    SELECT title, rank, hot_value, url, platform, created_at
    FROM hot_search_items
    WHERE platform = 'weibo'
    AND created_at >= DATE_SUB(NOW(), INTERVAL 1 DAY)
    ORDER BY rank ASC
    LIMIT 10
""")

trending_topics = cursor.fetchall()
for topic in trending_topics:
    print(f"#{topic['rank']} {topic['title']} (热度: {topic['hot_value']})")
```

### LLM-Based Topic Analysis

```python
from hotsearch_analysis_agent.llm_client import LLMClient
from hotsearch_analysis_agent.analyzer import TopicAnalyzer

# Initialize LLM client
llm_client = LLMClient(
    api_key=os.getenv("OPENAI_API_KEY"),
    base_url=os.getenv("OPENAI_API_BASE"),
    model=os.getenv("OPENAI_MODEL", "gpt-4")
)

# Analyze topics
analyzer = TopicAnalyzer(llm_client)

# Get topics about specific keyword
topics = fetch_topics_by_keyword("人工智能")

# Perform sentiment analysis
sentiment_result = analyzer.analyze_sentiment(topics)
print(f"Overall sentiment: {sentiment_result['tendency']}")
print(f"Positive: {sentiment_result['positive_ratio']}%")
print(f"Negative: {sentiment_result['negative_ratio']}%")

# Topic clustering
clusters = analyzer.cluster_topics(topics)
for cluster_name, cluster_topics in clusters.items():
    print(f"\nCluster: {cluster_name}")
    for t in cluster_topics:
        print(f"  - {t['title']}")
```

### Configure Push Notifications

```python
from hotsearch_analysis_agent.push_service import PushService

# Initialize push service
push_service = PushService()

# Create push task for trending topics
push_service.create_task(
    name="AI Technology Trends",
    keywords=["人工智能", "AI", "大模型"],
    platforms=["weibo", "zhihu", "bilibili"],
    channels=["wechat", "telegram"],
    schedule="0 9,12,18 * * *",  # Cron format: 9am, 12pm, 6pm daily
    threshold={
        "min_hot_value": 100000,  # Minimum trending score
        "min_topics": 3  # Minimum number of topics to trigger
    }
)

# Manual push execution
report = push_service.generate_report(
    keyword="人工智能",
    time_range="24h"
)
push_service.send_to_wechat(report)
push_service.send_to_telegram(report)
push_service.send_to_email(report)
```

### Custom Spider Integration

```python
import scrapy
from hotsearchcrawler.items import HotSearchItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    allowed_domains = ['example.com']
    start_urls = ['https://example.com/trending']

    def parse(self, response):
        for item in response.css('.trending-item'):
            yield HotSearchItem(
                title=item.css('.title::text').get(),
                rank=item.css('.rank::text').get(),
                hot_value=item.css('.hot-value::text').get(),
                url=item.css('a::attr(href)').get(),
                platform='custom_platform',
                category=item.css('.category::text').get(),
            )
```

## Analysis Report Generation

The system generates detailed analysis reports like this:

```python
from hotsearch_analysis_agent.report_generator import ReportGenerator

generator = ReportGenerator()

# Generate report for specific query
report = generator.generate(
    query="人工智能与前沿科技",
    time_range="7d",
    include_sentiment=True,
    include_clustering=True,
    include_sources=True
)

# Report structure
print(report.summary)  # Executive summary
print(report.key_findings)  # Bulleted insights
print(report.detailed_news)  # News items with URLs
print(report.sentiment_analysis)  # Sentiment breakdown
print(report.recommendations)  # Actionable recommendations
```

## Common Troubleshooting

### Browser Driver Issues

**Error**: `selenium.common.exceptions.WebDriverException: Message: 'chromedriver' executable needs to be in PATH`

**Solution**: Ensure ChromeDriver/EdgeDriver is installed and in system PATH:

```bash
# Verify driver is accessible
which chromedriver  # macOS/Linux
where chromedriver  # Windows

# Add to PATH if needed (Linux/macOS)
export PATH=$PATH:/path/to/driver/directory

# Or move driver to standard location
sudo mv chromedriver /usr/local/bin/  # macOS/Linux
```

### MySQL Connection Errors

**Error**: `mysql.connector.errors.ProgrammingError: Access denied`

**Solution**: Verify MySQL credentials in `.env`:

```bash
# Test connection manually
mysql -h localhost -u root -p
# Then enter password

# Grant privileges if needed
GRANT ALL PRIVILEGES ON hotsearch.* TO 'root'@'localhost';
FLUSH PRIVILEGES;
```

### LLM API Rate Limits

**Error**: `openai.error.RateLimitError`

**Solution**: Implement retry logic or use local models:

```python
# Switch to local Pangu model deployment
# Download from: https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

# Update .env
PANGU_MODEL_PATH=/path/to/pangu/model
USE_LOCAL_MODEL=true
```

### Crawler Timeout Issues

**Error**: `TimeoutError` when scraping platforms

**Solution**: Adjust timeout settings in `hotsearchcrawler/settings.py`:

```python
# Increase timeout values
DOWNLOAD_TIMEOUT = 30  # Default: 15
CONCURRENT_REQUESTS_PER_DOMAIN = 8  # Reduce from 16

# Add retry middleware
RETRY_TIMES = 5
RETRY_HTTP_CODES = [500, 502, 503, 504, 408, 429]
```

### Empty Scraping Results

**Issue**: Scrapers return no data for certain platforms

**Solution**: Update cookies for authenticated platforms:

```python
# In hotsearchcrawler/settings.py or platform-specific spider
# Export cookies from browser using extension like EditThisCookie
COOKIES = {
    'weibo': 'SUB=your_cookie_here; SUBP=...',
    'bilibili': 'SESSDATA=your_session_data; bili_jct=...',
}
```

## Platform Coverage

Supported platforms (26 lists across 15 platforms):
- Weibo (微博热搜)
- Bilibili (B站热门)
- Zhihu (知乎热榜)
- Douyin/TikTok (抖音热点)
- Baidu (百度热搜)
- Toutiao (今日头条)
- Kuaishou (快手)
- Xiaohongshu (小红书)
- 36Kr (36氪)
- IT之家
- And 5 more platforms

Each platform may provide multiple lists (e.g., general trending, video trending, topic-specific lists).
