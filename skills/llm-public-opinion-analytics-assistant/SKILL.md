---
name: llm-public-opinion-analytics-assistant
description: Multi-platform hot search monitoring and sentiment analysis assistant with 15 platforms, 26 leaderboards, LLM-powered clustering, and multi-channel alerts
triggers:
  - set up public opinion monitoring system
  - analyze social media sentiment with LLM
  - monitor hot search trends across platforms
  - create sentiment analysis crawler for Chinese platforms
  - build multi-platform trending topic aggregator
  - set up automated hot topic alerts and reports
  - implement opinion mining with large language models
  - configure Weibo Douyin Bilibili hot search monitoring
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is an intelligent public opinion analytics assistant that aggregates real-time data from **15 mainstream platforms** across **26 leaderboards** (Weibo, Douyin, Bilibili, Zhihu, Baidu, etc.) and combines it with large language model analysis capabilities. It provides conversational hot search queries, topic clustering, sentiment analysis, and multi-channel alert pushes (email, WeChat, Enterprise WeChat, Telegram).

**Key capabilities:**
- Real-time web scraping of hot search rankings from Chinese social media platforms
- LLM-powered sentiment analysis and topic clustering
- Browser automation for extracting content from news detail pages (including video transcripts)
- Multi-channel push notifications for hot topics
- Conversational interface for querying trends and insights
- Keyboard shortcuts for crawler control

## Installation

### Prerequisites

1. **Python 3.8+** with virtual environment
2. **MySQL database** for storing scraped data
3. **Browser driver** (Chrome/Edge) for web scraping
4. **Large Language Model** API access (OpenAI-compatible format, or local deployment like Huawei Pangu)

### Browser Driver Setup

Download the appropriate driver for your browser:

- **ChromeDriver**: https://chromedriver.chromium.org/
- **EdgeDriver**: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

Place the driver in your system PATH or browser installation directory. Verify installation:

```bash
chromedriver --version
# or
msedgedriver --version
```

### Project Setup

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

### Database Configuration

Create MySQL database and tables using the reference in `init.py`:

```python
import pymysql

connection = pymysql.connect(
    host='localhost',
    user='your_user',
    password='your_password',
    database='hotsearch_db',
    charset='utf8mb4'
)

# Create tables for each platform
with connection.cursor() as cursor:
    # Weibo hot search table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS weibo_hotsearch (
            id INT AUTO_INCREMENT PRIMARY KEY,
            rank INT,
            title VARCHAR(255),
            url VARCHAR(512),
            hot_value VARCHAR(50),
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            INDEX idx_created_at (created_at)
        ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
    """)
    
    # Add similar tables for other platforms
    # douyin_hotsearch, bilibili_hotsearch, zhihu_hotsearch, etc.
    
connection.commit()
connection.close()
```

### Environment Variables

Create a `.env` file in the project root:

```env
# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch_db

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.openai.com/v1
LLM_MODEL=gpt-4

# Or for local Huawei Pangu model
# OPENAI_API_BASE=http://localhost:8000/v1
# LLM_MODEL=pangu-7b

# Push Notification Settings
# Email (SMTP)
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password
EMAIL_RECIPIENTS=recipient1@example.com,recipient2@example.com

# Enterprise WeChat
WECHAT_WEBHOOK_URL=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=your_key
WECHAT_CORP_ID=your_corp_id
WECHAT_AGENT_ID=your_agent_id
WECHAT_SECRET=your_secret

# Telegram
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

## Configuration

### Crawler Settings

Edit `hotsearchcrawler/settings.py`:

```python
# MySQL Connection
MYSQL_HOST = os.getenv('MYSQL_HOST', 'localhost')
MYSQL_PORT = int(os.getenv('MYSQL_PORT', 3306))
MYSQL_USER = os.getenv('MYSQL_USER', 'root')
MYSQL_PASSWORD = os.getenv('MYSQL_PASSWORD', '')
MYSQL_DATABASE = os.getenv('MYSQL_DATABASE', 'hotsearch_db')

# Scrapy Settings
ROBOTSTXT_OBEY = False
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 2
COOKIES_ENABLED = True

# Optional: Platform-specific cookies for authenticated access
COOKIES = {
    'weibo': 'your_weibo_cookie_string',
    'douyin': 'your_douyin_cookie_string',
}

# User Agent Rotation
USER_AGENT_LIST = [
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
    'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36',
]
```

### Analysis System Configuration

The analysis system reads from `.env` automatically. Customize LLM prompts in `hotsearch_analysis_agent/prompts.py`:

```python
SENTIMENT_ANALYSIS_PROMPT = """
请分析以下新闻标题的情感倾向，返回"正面"、"负面"或"中性"，并给出1-10的情感强度分数。

标题：{title}
内容：{content}

输出格式：
情感倾向：[正面/负面/中性]
情感强度：[1-10]
理由：[简要说明]
"""

TOPIC_CLUSTERING_PROMPT = """
请对以下热搜话题进行聚类分析，识别出主要的话题类别。

热搜列表：
{hot_topics}

输出格式：
1. 话题类别1
   - 相关热搜：[列表]
   - 摘要：[说明]

2. 话题类别2
   ...
"""
```

## Usage

### Starting the System

```bash
# Start the web interface and analysis system
python app.py
```

The web interface will be available at `http://localhost:5000`

### Running Crawlers

**Option 1: Through Web Interface**
- Navigate to the crawler control panel
- Use keyboard shortcuts to start/stop crawlers for specific platforms

**Option 2: Command Line**

```bash
# Test individual spider
python runspider-test.py weibo

# Run all spiders
python run_spiders.py
```

**Option 3: Scheduled Execution**

```python
# Example cron job configuration
# Edit crontab: crontab -e
# Run every hour
0 * * * * cd /path/to/project && /path/to/venv/bin/python run_spiders.py

# Or use Python scheduler in app.py
from apscheduler.schedulers.background import BackgroundScheduler

scheduler = BackgroundScheduler()
scheduler.add_job(
    func=run_all_spiders,
    trigger='interval',
    hours=1,
    id='crawler_job'
)
scheduler.start()
```

### Conversational Query Interface

Use natural language to query hot topics:

```python
# In the web interface chat or via API
# Examples:
"今天微博热搜前十是什么？"
"帮我分析关于人工智能的舆情"
"最近三天科技类话题的情感倾向如何？"
"聚类分析今天的所有热搜话题"
```

### API Usage

```python
from hotsearch_analysis_agent import AnalysisAgent

# Initialize agent
agent = AnalysisAgent(
    llm_model=os.getenv('LLM_MODEL'),
    api_key=os.getenv('OPENAI_API_KEY'),
    api_base=os.getenv('OPENAI_API_BASE')
)

# Query hot topics
results = agent.query_hot_topics(
    platform='weibo',
    limit=20,
    date_range='today'
)

# Perform sentiment analysis
sentiment = agent.analyze_sentiment(
    topic="人工智能技术突破",
    content="DeepSeek V4采用华为算力..."
)
print(f"Sentiment: {sentiment['polarity']}, Score: {sentiment['score']}")

# Topic clustering
clusters = agent.cluster_topics(
    topics=results,
    num_clusters=5
)

# Generate report
report = agent.generate_report(
    query="人工智能与前沿科技",
    date_range="last_7_days",
    include_sentiment=True,
    include_clustering=True
)
```

### Setting Up Push Notifications

Test push functionality:

```bash
python test_push_task.py
```

Configure automated push tasks:

```python
from hotsearch_analysis_agent.push_task import PushTaskManager

manager = PushTaskManager()

# Create a push task
task = manager.create_task(
    name="AI热点日报",
    query="人工智能 OR 大模型 OR 芯片",
    schedule="daily",  # daily, hourly, weekly
    time="12:00",
    channels=['email', 'wechat', 'telegram'],
    sentiment_filter='all',  # all, positive, negative, neutral
    min_hot_value=10000
)

# Start task
manager.start_task(task.id)

# View active tasks
active_tasks = manager.list_active_tasks()

# Stop task
manager.stop_task(task.id)
```

## Common Patterns

### Custom Spider for New Platform

```python
# hotsearchcrawler/spiders/new_platform_spider.py
import scrapy
from hotsearchcrawler.items import HotSearchItem

class NewPlatformSpider(scrapy.Spider):
    name = 'new_platform'
    allowed_domains = ['newplatform.com']
    start_urls = ['https://www.newplatform.com/hot']
    
    def parse(self, response):
        # Extract hot search items
        for item in response.css('.hot-item'):
            hot_search = HotSearchItem()
            hot_search['rank'] = item.css('.rank::text').get()
            hot_search['title'] = item.css('.title::text').get()
            hot_search['url'] = item.css('a::attr(href)').get()
            hot_search['hot_value'] = item.css('.hot-value::text').get()
            hot_search['platform'] = 'new_platform'
            
            yield hot_search
```

### Extracting Content from Detail Pages

```python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

def extract_article_content(url):
    """Extract content from news detail page, including video transcripts"""
    options = webdriver.ChromeOptions()
    options.add_argument('--headless')
    driver = webdriver.Chrome(options=options)
    
    try:
        driver.get(url)
        
        # Wait for content to load
        WebDriverWait(driver, 10).until(
            EC.presence_of_element_located((By.CLASS_NAME, 'article-content'))
        )
        
        # Extract text content
        content = driver.find_element(By.CLASS_NAME, 'article-content').text
        
        # Check for video and extract transcript if available
        try:
            video_transcript = driver.find_element(
                By.CLASS_NAME, 'video-transcript'
            ).text
            content += f"\n\n视频文字稿：\n{video_transcript}"
        except:
            pass
        
        return content
        
    finally:
        driver.quit()
```

### Custom Sentiment Analysis with Local LLM

```python
import requests

def analyze_with_pangu(text):
    """Use local Huawei Pangu model for sentiment analysis"""
    api_url = os.getenv('OPENAI_API_BASE', 'http://localhost:8000/v1')
    
    response = requests.post(
        f"{api_url}/chat/completions",
        json={
            "model": "pangu-7b",
            "messages": [
                {
                    "role": "system",
                    "content": "你是一个专业的舆情分析专家。"
                },
                {
                    "role": "user",
                    "content": f"分析以下文本的情感倾向：\n\n{text}"
                }
            ],
            "temperature": 0.3,
            "max_tokens": 500
        },
        headers={"Content-Type": "application/json"}
    )
    
    return response.json()['choices'][0]['message']['content']
```

### Batch Analysis Workflow

```python
def daily_analysis_workflow():
    """Complete daily analysis and report generation"""
    agent = AnalysisAgent()
    
    # 1. Fetch today's hot topics from all platforms
    all_topics = []
    for platform in ['weibo', 'douyin', 'bilibili', 'zhihu', 'baidu']:
        topics = agent.query_hot_topics(
            platform=platform,
            limit=50,
            date_range='today'
        )
        all_topics.extend(topics)
    
    # 2. Perform topic clustering
    clusters = agent.cluster_topics(all_topics, num_clusters=8)
    
    # 3. Analyze sentiment for each cluster
    for cluster in clusters:
        cluster['sentiment'] = agent.analyze_sentiment_batch(
            cluster['topics']
        )
    
    # 4. Generate comprehensive report
    report = agent.generate_comprehensive_report(
        clusters=clusters,
        date=datetime.now().strftime('%Y-%m-%d')
    )
    
    # 5. Push to configured channels
    push_manager = PushTaskManager()
    push_manager.send_report(
        report=report,
        channels=['email', 'wechat']
    )
    
    return report
```

## Troubleshooting

### Crawler Issues

**Problem**: Spider fails to extract data
- Check if website structure has changed (update CSS selectors)
- Verify cookies are still valid for authenticated platforms
- Increase `DOWNLOAD_DELAY` to avoid rate limiting

```python
# Debug spider with verbose logging
scrapy crawl weibo -L DEBUG
```

**Problem**: Database connection errors
- Verify MySQL service is running: `systemctl status mysql`
- Check credentials in `.env` file
- Ensure database and tables exist

```python
# Test database connection
import pymysql
connection = pymysql.connect(
    host=os.getenv('MYSQL_HOST'),
    user=os.getenv('MYSQL_USER'),
    password=os.getenv('MYSQL_PASSWORD'),
    database=os.getenv('MYSQL_DATABASE')
)
print("Connection successful!")
connection.close()
```

### LLM Analysis Issues

**Problem**: LLM API timeout or errors
- Check API key validity
- Verify API endpoint URL
- Reduce batch size for large text inputs

```python
# Test LLM connection
import openai
openai.api_key = os.getenv('OPENAI_API_KEY')
openai.api_base = os.getenv('OPENAI_API_BASE')

response = openai.ChatCompletion.create(
    model=os.getenv('LLM_MODEL'),
    messages=[{"role": "user", "content": "测试连接"}],
    max_tokens=50
)
print(response.choices[0].message.content)
```

**Problem**: Poor sentiment analysis accuracy
- Refine prompts in `prompts.py`
- Use examples in few-shot learning
- Consider fine-tuning local model on domain-specific data

### Browser Driver Issues

**Problem**: ChromeDriver version mismatch
- Update ChromeDriver to match your Chrome version
- Or specify exact driver path:

```python
from selenium import webdriver
driver = webdriver.Chrome(executable_path='/path/to/chromedriver')
```

**Problem**: Selenium timeouts on slow networks
- Increase wait times
- Use explicit waits instead of implicit waits

```python
from selenium.webdriver.support.ui import WebDriverWait

wait = WebDriverWait(driver, 30)  # Increase to 30 seconds
element = wait.until(EC.presence_of_element_located((By.ID, 'content')))
```

### Push Notification Issues

**Problem**: WeChat webhook not receiving messages
- Verify webhook URL is correct
- Check message format matches WeChat API requirements
- Ensure webhook is not rate-limited (max 20 messages/minute)

**Problem**: Email push fails
- Check SMTP credentials and server settings
- Enable "Less secure app access" or use app-specific passwords
- Verify firewall allows SMTP port (usually 587 or 465)

```python
# Test email sending
import smtplib
from email.mime.text import MIMEText

msg = MIMEText("Test message")
msg['Subject'] = "Test"
msg['From'] = os.getenv('SMTP_USER')
msg['To'] = 'recipient@example.com'

with smtplib.SMTP(os.getenv('SMTP_SERVER'), int(os.getenv('SMTP_PORT'))) as server:
    server.starttls()
    server.login(os.getenv('SMTP_USER'), os.getenv('SMTP_PASSWORD'))
    server.send_message(msg)
    print("Email sent successfully!")
```

## Performance Optimization

### Crawler Performance

```python
# Increase concurrency in settings.py
CONCURRENT_REQUESTS = 32
CONCURRENT_REQUESTS_PER_DOMAIN = 8

# Enable caching to avoid redundant requests
HTTPCACHE_ENABLED = True
HTTPCACHE_EXPIRATION_SECS = 3600
HTTPCACHE_DIR = 'httpcache'
```

### Database Optimization

```sql
-- Add indexes for faster queries
CREATE INDEX idx_platform_created ON weibo_hotsearch(platform, created_at);
CREATE INDEX idx_hot_value ON weibo_hotsearch(hot_value);

-- Partition tables by date for large datasets
ALTER TABLE weibo_hotsearch PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION p2026 VALUES LESS THAN (2027)
);
```

### LLM Batch Processing

```python
# Process multiple items in parallel
from concurrent.futures import ThreadPoolExecutor

def analyze_batch(topics, batch_size=10):
    with ThreadPoolExecutor(max_workers=5) as executor:
        futures = []
        for i in range(0, len(topics), batch_size):
            batch = topics[i:i+batch_size]
            future = executor.submit(agent.analyze_sentiment_batch, batch)
            futures.append(future)
        
        results = [f.result() for f in futures]
    
    return results
```
