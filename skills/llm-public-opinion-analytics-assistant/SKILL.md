---
name: llm-public-opinion-analytics-assistant
description: Multi-platform hot search monitoring and LLM-powered sentiment analysis system with web UI and multi-channel alerting
triggers:
  - how do I set up the public opinion analytics assistant
  - configure multi-platform hot search crawler with LLM analysis
  - integrate sentiment analysis with hot topic monitoring
  - set up automated sentiment reports with push notifications
  - crawl trending topics from multiple Chinese platforms
  - analyze public opinion with large language models
  - deploy hot search monitoring system with web interface
  - configure Pangu LLM for sentiment analysis
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

This is a comprehensive public opinion monitoring system that:
- Crawls hot search/trending topics from **15 Chinese platforms** (26 ranking lists total)
- Analyzes content using large language models (Pangu, OpenAI-compatible APIs)
- Provides conversational web UI for querying trends, clustering topics, and sentiment analysis
- Extracts content from news detail pages (including video transcripts)
- Sends automated reports via Email, WeChat Work, Enterprise WeChat, or Telegram
- Supports keyboard shortcuts for crawler control

The system is split into two main components:
- **Analysis System** (`hotsearch_analysis_agent/`) - LLM-powered analytics and web UI
- **Crawler Cluster** (`hotsearchcrawler/`) - Scrapy-based distributed crawlers

## Installation

### Prerequisites

1. **Python 3.8+** with virtual environment
2. **MySQL 5.7+** database
3. **Browser Driver** (ChromeDriver or EdgeDriver)

### Browser Driver Setup

```bash
# Check your Chrome/Edge version first
# Chrome: Settings → About Chrome
# Edge: Settings → About Microsoft Edge

# Download matching driver version:
# Chrome: https://chromedriver.chromium.org/
# Edge: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Place driver in system PATH or project directory
# Verify installation:
chromedriver --version
# or
msedgedriver --version
```

### Project Installation

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

### Database Configuration

```python
# Reference init.py for schema setup
import pymysql

connection = pymysql.connect(
    host='localhost',
    user='root',
    password='your_password',
    database='hotsearch_db',
    charset='utf8mb4'
)

# Create tables (see init.py for full schema)
cursor = connection.cursor()
cursor.execute("""
    CREATE TABLE IF NOT EXISTS hot_search_items (
        id INT AUTO_INCREMENT PRIMARY KEY,
        platform VARCHAR(50),
        title VARCHAR(500),
        url VARCHAR(1000),
        rank INT,
        heat_score VARCHAR(100),
        content TEXT,
        sentiment VARCHAR(50),
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
""")
connection.commit()
```

## Configuration

### Environment Variables

Create `.env` file in project root:

```bash
# Database Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key_here
OPENAI_API_BASE=https://api.openai.com/v1  # Or custom endpoint
OPENAI_MODEL=gpt-4

# Or use Pangu model (local deployment)
PANGU_API_BASE=http://localhost:8000
PANGU_MODEL=openpangu-embedded-7b

# Push Notification Channels
# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
SMTP_TO=recipient@example.com

# WeChat Work Bot
WECHAT_WORK_BOT_KEY=your_bot_webhook_key

# Enterprise WeChat App
WECHAT_CORP_ID=your_corp_id
WECHAT_AGENT_ID=your_agent_id
WECHAT_SECRET=your_secret

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

### Crawler Settings

Edit `hotsearchcrawler/settings.py`:

```python
# MySQL Connection
MYSQL_HOST = os.getenv('MYSQL_HOST', 'localhost')
MYSQL_PORT = int(os.getenv('MYSQL_PORT', 3306))
MYSQL_USER = os.getenv('MYSQL_USER', 'root')
MYSQL_PASSWORD = os.getenv('MYSQL_PASSWORD')
MYSQL_DATABASE = os.getenv('MYSQL_DATABASE', 'hotsearch_db')

# Crawl Settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 2
COOKIES_ENABLED = True

# Optional: Platform-specific cookies
WEIBO_COOKIES = {
    'cookie_name': 'cookie_value'
}
```

## Running the System

### Start the Web Application

```bash
# Main application entry point
python app.py
```

The web UI will be available at `http://localhost:5000` (or configured port).

### Start Crawlers

```bash
# Test single crawler
python runspider-test.py

# Start all crawlers (can also be triggered from web UI)
python run_spiders.py
```

### Test Push Notifications

```bash
# Test all configured push channels
python test_push_task.py
```

## Key Usage Patterns

### Querying Hot Topics via Web UI

The web interface supports natural language queries:

```
# Example queries users can make:
"Show me today's Weibo hot search"
"What topics are trending on Bilibili?"
"Search for articles about artificial intelligence"
"Cluster analysis of recent tech news"
"Sentiment analysis for [specific topic]"
```

### Programmatic Data Access

```python
# Access hot search data directly
from hotsearch_analysis_agent.database import get_db_connection

def get_latest_trends(platform='weibo', limit=10):
    """Fetch latest hot search items from database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    query = """
        SELECT title, url, rank, heat_score, created_at
        FROM hot_search_items
        WHERE platform = %s
        ORDER BY created_at DESC, rank ASC
        LIMIT %s
    """
    cursor.execute(query, (platform, limit))
    results = cursor.fetchall()
    cursor.close()
    conn.close()
    
    return results

# Get trending topics
trends = get_latest_trends('weibo', 20)
for trend in trends:
    print(f"#{trend[2]} {trend[0]} - Heat: {trend[3]}")
```

### LLM Analysis Integration

```python
# Use LLM for sentiment analysis
from hotsearch_analysis_agent.llm_client import analyze_sentiment

def analyze_topic_sentiment(topic_title, content):
    """Analyze sentiment of a hot topic"""
    prompt = f"""
    Analyze the sentiment and public opinion for this topic:
    
    Title: {topic_title}
    Content: {content}
    
    Provide:
    1. Overall sentiment (positive/negative/neutral)
    2. Key emotional drivers
    3. Potential risks or opportunities
    """
    
    result = analyze_sentiment(prompt)
    return result

# Example usage
content = get_topic_content("某热点话题")
sentiment = analyze_topic_sentiment("某热点话题", content)
print(sentiment)
```

### Topic Clustering

```python
# Cluster related topics
from hotsearch_analysis_agent.clustering import cluster_topics

def find_related_topics(keywords, days=7):
    """Find and cluster related topics from last N days"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    query = """
        SELECT id, title, content, platform
        FROM hot_search_items
        WHERE created_at >= DATE_SUB(NOW(), INTERVAL %s DAY)
        AND (title LIKE %s OR content LIKE %s)
    """
    
    keyword_pattern = f"%{keywords}%"
    cursor.execute(query, (days, keyword_pattern, keyword_pattern))
    topics = cursor.fetchall()
    
    cursor.close()
    conn.close()
    
    # Cluster similar topics
    clusters = cluster_topics(topics)
    return clusters

# Find AI-related topics
ai_clusters = find_related_topics("人工智能", days=7)
```

### Setting Up Push Tasks

```python
# Configure automated report push
from hotsearch_analysis_agent.push_service import PushService

def setup_daily_report():
    """Set up daily sentiment report push"""
    push_service = PushService()
    
    # Configure report parameters
    report_config = {
        'topic_keywords': ['人工智能', '前沿科技'],
        'platforms': ['weibo', 'zhihu', 'bilibili'],
        'analysis_depth': 'detailed',
        'schedule': '08:00',  # Daily at 8 AM
        'channels': ['email', 'wechat_work']
    }
    
    # Register push task
    push_service.create_scheduled_task(report_config)
    print("Daily report task created successfully")

setup_daily_report()
```

### Manual Report Generation

```python
# Generate on-demand analysis report
from hotsearch_analysis_agent.report_generator import generate_report

def create_custom_report(topic, start_date, end_date):
    """Generate custom analysis report"""
    report = generate_report(
        topic=topic,
        start_date=start_date,
        end_date=end_date,
        platforms=['weibo', 'zhihu', 'toutiao'],
        include_sentiment=True,
        include_clustering=True,
        output_format='markdown'
    )
    
    return report

# Generate report
report = create_custom_report(
    topic="人工智能与前沿科技",
    start_date="2026-04-01",
    end_date="2026-04-07"
)
print(report)
```

## Crawler Development

### Adding a New Platform Crawler

```python
# hotsearchcrawler/spiders/new_platform_spider.py
import scrapy
from hotsearchcrawler.items import HotSearchItem

class NewPlatformSpider(scrapy.Spider):
    name = 'new_platform'
    allowed_domains = ['newplatform.com']
    start_urls = ['https://newplatform.com/trending']
    
    def parse(self, response):
        """Parse hot search list"""
        for idx, item in enumerate(response.css('.trend-item')):
            hot_search = HotSearchItem()
            hot_search['platform'] = 'new_platform'
            hot_search['title'] = item.css('.title::text').get()
            hot_search['url'] = item.css('a::attr(href)').get()
            hot_search['rank'] = idx + 1
            hot_search['heat_score'] = item.css('.heat::text').get()
            
            # Fetch detail page content
            yield scrapy.Request(
                hot_search['url'],
                callback=self.parse_detail,
                meta={'item': hot_search}
            )
    
    def parse_detail(self, response):
        """Parse detail page content"""
        item = response.meta['item']
        item['content'] = response.css('.article-content::text').getall()
        yield item
```

### Pipeline for Data Processing

```python
# hotsearchcrawler/pipelines.py
class MySQLPipeline:
    def process_item(self, item, spider):
        """Store item to MySQL database"""
        query = """
            INSERT INTO hot_search_items 
            (platform, title, url, rank, heat_score, content)
            VALUES (%s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE
            rank=VALUES(rank), heat_score=VALUES(heat_score)
        """
        
        self.cursor.execute(query, (
            item['platform'],
            item['title'],
            item['url'],
            item['rank'],
            item['heat_score'],
            item.get('content', '')
        ))
        self.connection.commit()
        
        return item
```

## Troubleshooting

### Crawler Issues

**Problem: Browser driver not found**
```bash
# Verify driver is in PATH
which chromedriver  # macOS/Linux
where chromedriver  # Windows

# Or specify driver path in settings
CHROME_DRIVER_PATH = '/usr/local/bin/chromedriver'
```

**Problem: Platform returns 403/blocked**
```python
# Add user agent and headers in settings.py
USER_AGENT = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'
DEFAULT_REQUEST_HEADERS = {
    'Accept': 'text/html,application/xhtml+xml',
    'Accept-Language': 'zh-CN,zh;q=0.9',
    'Referer': 'https://platform.com'
}

# Enable cookies and add delays
COOKIES_ENABLED = True
DOWNLOAD_DELAY = 3
```

### Database Connection Issues

```python
# Test database connection
import pymysql
from pymysql.err import OperationalError

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
except OperationalError as e:
    print(f"Connection failed: {e}")
```

### LLM API Issues

**Problem: API timeout or rate limit**
```python
# Add retry logic and timeout configuration
import openai
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(min=1, max=10))
def call_llm_with_retry(prompt):
    response = openai.ChatCompletion.create(
        model=os.getenv('OPENAI_MODEL'),
        messages=[{"role": "user", "content": prompt}],
        timeout=30
    )
    return response.choices[0].message.content
```

### Push Notification Failures

```python
# Test individual push channels
def test_email_push():
    import smtplib
    from email.mime.text import MIMEText
    
    msg = MIMEText('Test message')
    msg['Subject'] = 'Test'
    msg['From'] = os.getenv('SMTP_USER')
    msg['To'] = os.getenv('SMTP_TO')
    
    with smtplib.SMTP(os.getenv('SMTP_HOST'), int(os.getenv('SMTP_PORT'))) as server:
        server.starttls()
        server.login(os.getenv('SMTP_USER'), os.getenv('SMTP_PASSWORD'))
        server.send_message(msg)
    print("Email test successful")

test_email_push()
```

## Best Practices

1. **Rate Limiting**: Set appropriate delays between crawler requests to avoid IP blocks
2. **Data Validation**: Always validate and sanitize crawled content before LLM analysis
3. **Error Handling**: Implement comprehensive error handling for network failures and API errors
4. **Resource Management**: Monitor database size and implement data archival for old records
5. **LLM Cost Control**: Cache common analysis results and use rate limiting for API calls
6. **Security**: Never commit API keys or credentials; always use environment variables
