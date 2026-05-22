---
name: llm-public-opinion-analytics-assistant
description: Chinese public opinion analytics assistant with multi-platform hot search crawling, LLM-based analysis, clustering, sentiment analysis, and multi-channel push notifications
triggers:
  - how do I set up a public opinion monitoring system
  - crawl hot search data from chinese social platforms
  - analyze trending topics with sentiment analysis
  - set up multi-platform news crawler with LLM analysis
  - configure weixin telegram email push notifications for trending topics
  - build a chinese social media analytics dashboard
  - cluster and analyze hot topics from weibo douyin bilibili
  - create automated public opinion reports with ai
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is a comprehensive public opinion analytics system that crawls hot search data from **15 Chinese platforms** (26 lists total), performs LLM-based analysis, and delivers insights via multiple channels (email, WeChat, Telegram). It combines distributed crawling with Pangu/OpenAI-compatible LLM models for topic clustering, sentiment analysis, and automated reporting.

## What It Does

- **Multi-platform crawling**: Real-time hot search data from Weibo, Douyin, Bilibili, Zhihu, Baidu, Toutiao, and 9+ other platforms
- **LLM-powered analysis**: Topic clustering, sentiment analysis, and trend detection using Huawei Pangu or OpenAI-compatible models
- **Interactive web UI**: Conversational interface for querying trends, searching topics, and viewing analysis
- **Automated push notifications**: Schedule reports to WeChat Work, Telegram, or email
- **Video content extraction**: Extracts information even from video-based news items
- **Keyboard shortcuts**: Quick control of crawler start/stop

## Installation

### Prerequisites

**1. Browser Driver Setup (Required for crawler)**

The project uses Selenium for dynamic content. You need Chrome/Edge driver:

```bash
# macOS with Chrome
brew install chromedriver

# Or download manually from:
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Verify installation
chromedriver --version
```

Add driver to PATH or place in project directory.

**2. Python Environment**

```bash
# Clone the repository
git clone https://github.com/hmmnxkl/LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant.git
cd LLM-Based-Intelligent-Public-Opinion-Analytics-Assistant

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

**3. MySQL Database**

```bash
# Install MySQL 5.7+ or 8.0+
# Create database and tables using init.py as reference

# Example database setup
mysql -u root -p
CREATE DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

Run `init.py` to initialize schema:

```python
# Review and execute init.py to create tables
python init.py
```

## Configuration

### Environment Variables

Create `.env` file in project root:

```bash
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_username
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1  # Or your custom endpoint

# For Huawei Pangu Model (if using local deployment)
PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model
PANGU_API_URL=http://localhost:8000/v1

# Push Notification Channels
# WeChat Work Bot
WECHAT_WORK_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=your_key

# WeChat Work App (for personal WeChat push)
WECHAT_WORK_CORPID=your_corp_id
WECHAT_WORK_CORPSECRET=your_corp_secret
WECHAT_WORK_AGENTID=your_agent_id

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# SMTP Email
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
EMAIL_RECIPIENTS=recipient1@example.com,recipient2@example.com
```

### Crawler Configuration

Edit `hotsearchcrawler/settings.py`:

```python
# MySQL connection for crawler
MYSQL_HOST = 'localhost'
MYSQL_PORT = 3306
MYSQL_USER = 'your_username'
MYSQL_PASSWORD = 'your_password'
MYSQL_DATABASE = 'hotsearch'

# Optional: Platform-specific cookies for authenticated access
COOKIES = {
    'weibo': 'your_weibo_cookie',
    'douyin': 'your_douyin_cookie',
    # Add others as needed
}

# Crawler concurrency
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
```

## Running the Application

### Start the Web Interface

```bash
# From project root
python app.py
```

Access the web UI at `http://localhost:5000`

### Start Crawlers

**Option 1: Via Web UI**
- Click "Start Crawler" button in the interface
- Use keyboard shortcuts (Ctrl+S to start, Ctrl+E to stop)

**Option 2: Command Line**

```bash
# Run all crawlers
python run_spiders.py

# Test individual spider
python runspider-test.py --spider=weibo
python runspider-test.py --spider=douyin
```

Available spiders:
- `weibo` - Weibo hot search
- `douyin` - Douyin trending
- `bilibili` - Bilibili hot videos
- `zhihu` - Zhihu hot questions
- `baidu` - Baidu trending
- `toutiao` - Toutiao news
- And 9+ more platforms

### Test Push Notifications

```bash
# Test all configured push channels
python test_push_task.py

# The script will send a test message to all enabled channels
```

## API Usage Examples

### Query Hot Search Data

```python
from hotsearch_analysis_agent.database import get_db_connection

def get_platform_trending(platform='weibo', limit=10):
    """Get trending topics from a specific platform"""
    conn = get_db_connection()
    cursor = conn.cursor(dictionary=True)
    
    query = """
        SELECT title, rank, hot_value, url, created_at
        FROM hot_search
        WHERE platform = %s
        ORDER BY created_at DESC, rank ASC
        LIMIT %s
    """
    
    cursor.execute(query, (platform, limit))
    results = cursor.fetchall()
    cursor.close()
    conn.close()
    
    return results

# Usage
trending = get_platform_trending('weibo', 20)
for item in trending:
    print(f"#{item['rank']}: {item['title']} (热度: {item['hot_value']})")
```

### LLM-Based Topic Clustering

```python
from hotsearch_analysis_agent.llm_analyzer import cluster_topics
import os

def analyze_trending_clusters(keywords, days=1):
    """Cluster related trending topics using LLM"""
    
    # Fetch related news
    conn = get_db_connection()
    cursor = conn.cursor(dictionary=True)
    
    query = """
        SELECT title, content, platform, url
        FROM hot_search
        WHERE created_at >= DATE_SUB(NOW(), INTERVAL %s DAY)
        AND (title LIKE %s OR content LIKE %s)
    """
    
    search_pattern = f'%{keywords}%'
    cursor.execute(query, (days, search_pattern, search_pattern))
    news_items = cursor.fetchall()
    cursor.close()
    conn.close()
    
    # Use LLM to cluster
    clusters = cluster_topics(news_items, api_key=os.getenv('OPENAI_API_KEY'))
    
    return clusters

# Usage
clusters = analyze_trending_clusters('人工智能')
for cluster in clusters:
    print(f"主题: {cluster['theme']}")
    print(f"相关新闻: {len(cluster['items'])} 条")
    print(f"情感倾向: {cluster['sentiment']}")
    print("---")
```

### Sentiment Analysis

```python
from hotsearch_analysis_agent.llm_analyzer import analyze_sentiment
import os

def get_topic_sentiment(topic_title):
    """Analyze sentiment for a specific trending topic"""
    
    # Get topic details and comments
    conn = get_db_connection()
    cursor = conn.cursor(dictionary=True)
    
    query = """
        SELECT title, content, platform, url, hot_value
        FROM hot_search
        WHERE title = %s
        ORDER BY created_at DESC
        LIMIT 1
    """
    
    cursor.execute(query, (topic_title,))
    topic = cursor.fetchone()
    cursor.close()
    conn.close()
    
    if not topic:
        return None
    
    # Perform sentiment analysis
    analysis = analyze_sentiment(
        text=topic['content'],
        api_key=os.getenv('OPENAI_API_KEY')
    )
    
    return {
        'topic': topic['title'],
        'sentiment': analysis['sentiment'],  # positive/negative/neutral
        'score': analysis['score'],  # -1.0 to 1.0
        'summary': analysis['summary']
    }

# Usage
sentiment = get_topic_sentiment('GPT-6遭提前曝光')
print(f"话题: {sentiment['topic']}")
print(f"情感: {sentiment['sentiment']} (分数: {sentiment['score']})")
print(f"摘要: {sentiment['summary']}")
```

### Schedule Automated Push Notifications

```python
from hotsearch_analysis_agent.push_service import PushService
import os

def create_push_task(keywords, channels=['wechat', 'telegram', 'email']):
    """Create scheduled push notification for specific topics"""
    
    push_service = PushService(
        wechat_webhook=os.getenv('WECHAT_WORK_WEBHOOK'),
        telegram_token=os.getenv('TELEGRAM_BOT_TOKEN'),
        telegram_chat=os.getenv('TELEGRAM_CHAT_ID'),
        smtp_config={
            'host': os.getenv('SMTP_HOST'),
            'port': int(os.getenv('SMTP_PORT')),
            'user': os.getenv('SMTP_USER'),
            'password': os.getenv('SMTP_PASSWORD'),
            'recipients': os.getenv('EMAIL_RECIPIENTS').split(',')
        }
    )
    
    # Analyze and generate report
    clusters = analyze_trending_clusters(keywords)
    
    # Format report
    report = format_analysis_report(
        title=f"关于{keywords}的热点分析",
        clusters=clusters,
        timestamp=datetime.now()
    )
    
    # Push to selected channels
    results = {}
    if 'wechat' in channels:
        results['wechat'] = push_service.push_to_wechat(report)
    if 'telegram' in channels:
        results['telegram'] = push_service.push_to_telegram(report)
    if 'email' in channels:
        results['email'] = push_service.push_to_email(
            subject=f"舆情分析报告: {keywords}",
            body=report
        )
    
    return results

# Usage
push_results = create_push_task(
    keywords='人工智能与前沿科技',
    channels=['wechat', 'telegram', 'email']
)
```

### Custom Spider for New Platform

```python
# hotsearchcrawler/spiders/custom_platform.py
import scrapy
from hotsearchcrawler.items import HotSearchItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    start_urls = ['https://example.com/trending']
    
    def parse(self, response):
        # Extract trending items
        for item in response.css('.trending-item'):
            hot_item = HotSearchItem()
            hot_item['platform'] = 'custom_platform'
            hot_item['title'] = item.css('.title::text').get()
            hot_item['rank'] = item.css('.rank::text').get()
            hot_item['hot_value'] = item.css('.hot-value::text').get()
            hot_item['url'] = response.urljoin(item.css('a::attr(href)').get())
            
            # Follow detail page for full content
            yield scrapy.Request(
                hot_item['url'],
                callback=self.parse_detail,
                meta={'item': hot_item}
            )
    
    def parse_detail(self, response):
        item = response.meta['item']
        item['content'] = response.css('.article-content::text').getall()
        item['publish_time'] = response.css('.publish-time::text').get()
        
        yield item
```

## Common Patterns

### Daily Automated Report

```python
import schedule
import time

def daily_ai_report():
    """Generate and push daily AI trending report"""
    keywords = ['人工智能', '大模型', '机器学习', 'AI']
    
    for keyword in keywords:
        create_push_task(
            keywords=keyword,
            channels=['wechat', 'email']
        )
        time.sleep(300)  # 5 minute delay between reports

# Schedule daily at 9 AM
schedule.every().day.at("09:00").do(daily_ai_report)

while True:
    schedule.run_pending()
    time.sleep(60)
```

### Multi-Platform Topic Tracking

```python
def track_topic_across_platforms(topic, platforms=None):
    """Track how a topic trends across different platforms"""
    
    if platforms is None:
        platforms = ['weibo', 'douyin', 'bilibili', 'zhihu', 'toutiao']
    
    results = {}
    
    conn = get_db_connection()
    cursor = conn.cursor(dictionary=True)
    
    for platform in platforms:
        query = """
            SELECT AVG(rank) as avg_rank, 
                   MAX(hot_value) as peak_hot,
                   COUNT(*) as appearances
            FROM hot_search
            WHERE platform = %s 
            AND title LIKE %s
            AND created_at >= DATE_SUB(NOW(), INTERVAL 7 DAY)
        """
        
        cursor.execute(query, (platform, f'%{topic}%'))
        results[platform] = cursor.fetchone()
    
    cursor.close()
    conn.close()
    
    return results

# Usage
trend_data = track_topic_across_platforms('DeepSeek V4')
for platform, stats in trend_data.items():
    if stats['appearances'] > 0:
        print(f"{platform}: 平均排名 {stats['avg_rank']:.1f}, "
              f"峰值热度 {stats['peak_hot']}, "
              f"上榜 {stats['appearances']} 次")
```

## Troubleshooting

### Crawler Issues

**Crawlers not collecting data:**
```bash
# Check if browser driver is accessible
which chromedriver  # macOS/Linux
where chromedriver  # Windows

# Test individual spider with verbose output
scrapy crawl weibo -s LOG_LEVEL=DEBUG

# Verify database connection
python -c "from hotsearch_analysis_agent.database import get_db_connection; print('DB OK' if get_db_connection() else 'DB FAIL')"
```

**Anti-scraping measures:**
- Add delays in `settings.py`: `DOWNLOAD_DELAY = 3`
- Configure cookies for authenticated platforms
- Use proxy rotation if needed

### LLM Analysis Issues

**API connection errors:**
```python
# Test OpenAI-compatible endpoint
import openai
import os

openai.api_key = os.getenv('OPENAI_API_KEY')
openai.api_base = os.getenv('OPENAI_API_BASE')

try:
    response = openai.ChatCompletion.create(
        model="gpt-3.5-turbo",
        messages=[{"role": "user", "content": "测试"}]
    )
    print("API OK:", response.choices[0].message.content)
except Exception as e:
    print("API Error:", e)
```

**Using Huawei Pangu locally:**
```bash
# Download model from https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model
# Deploy with vLLM or compatible server
python -m vllm.entrypoints.openai.api_server \
    --model /path/to/openpangu-embedded-7b-model \
    --port 8000

# Update .env
OPENAI_API_BASE=http://localhost:8000/v1
```

### Push Notification Failures

**WeChat Work webhook not working:**
- Verify webhook URL is correct and active
- Check if bot is added to target group
- Test with curl:
```bash
curl -X POST "https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"msgtype":"text","text":{"content":"测试消息"}}'
```

**Telegram bot issues:**
- Verify bot token with: `https://api.telegram.org/bot<TOKEN>/getMe`
- Check chat ID is correct (use `/getUpdates` endpoint)

**Email SMTP errors:**
- For Gmail, use App Password instead of regular password
- Enable "Less secure app access" or use OAuth2
- Check firewall allows outbound SMTP ports (587/465)

### Database Performance

**Slow queries:**
```sql
-- Add indexes for common query patterns
CREATE INDEX idx_platform_time ON hot_search(platform, created_at);
CREATE INDEX idx_title_search ON hot_search(title);
CREATE INDEX idx_hot_value ON hot_search(hot_value DESC);

-- Optimize table
OPTIMIZE TABLE hot_search;
```

**Disk space issues:**
```sql
-- Archive old data (older than 90 days)
CREATE TABLE hot_search_archive LIKE hot_search;
INSERT INTO hot_search_archive 
SELECT * FROM hot_search 
WHERE created_at < DATE_SUB(NOW(), INTERVAL 90 DAY);

DELETE FROM hot_search 
WHERE created_at < DATE_SUB(NOW(), INTERVAL 90 DAY);
```

This skill provides comprehensive coverage of the public opinion analytics system, from setup through advanced usage patterns for multi-platform monitoring and LLM-powered analysis.
