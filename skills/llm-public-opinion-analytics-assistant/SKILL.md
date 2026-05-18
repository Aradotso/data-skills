---
name: llm-public-opinion-analytics-assistant
description: Multi-platform hot topic crawler and LLM-powered sentiment analysis system with push notifications
triggers:
  - how do I set up the public opinion analytics assistant
  - crawl hot topics from weibo bilibili douyin
  - analyze sentiment and cluster topics with LLM
  - configure push notifications for trending topics
  - set up hot search crawler with database
  - use pangu model for opinion analysis
  - integrate multi-platform news monitoring
  - deploy sentiment analysis with LLM agent
---

# LLM-Based Intelligent Public Opinion Analytics Assistant

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is a comprehensive public opinion monitoring system that combines real-time data crawling from 15 mainstream Chinese platforms (26 ranking lists) with LLM-powered analysis. It provides:

- **Multi-platform hot topic crawling** (Weibo, Bilibili, Douyin, etc.)
- **LLM-driven analysis**: topic clustering, sentiment analysis, conversational queries
- **Content extraction**: Including video-based news content
- **Multi-channel push notifications**: Email, WeChat Work, Telegram
- **Web interface**: Real-time dashboard with hotkey controls

The system consists of two main components:
- `hotsearch_analysis_agent`: Analysis system with LLM integration
- `hotsearchcrawler`: Distributed crawler cluster (fully decoupled)

## Installation

### Prerequisites

**1. Browser Driver Setup (Critical)**

The system uses Selenium to extract detailed news content. You must install browser drivers:

```bash
# For Chrome/Chromium
# Download ChromeDriver from: https://chromedriver.chromium.org/
# Match your Chrome version (check in chrome://version/)

# For Edge
# Download from: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/

# Place driver in system PATH or project directory
# Verify installation:
chromedriver --version  # or msedgedriver --version
```

**2. MySQL Database**

```sql
-- Create database
CREATE DATABASE hotsearch CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Refer to init.py for complete table schemas
-- Key tables: hot_topics, news_details, analysis_results, push_tasks
```

**3. Python Environment**

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Configuration

**Environment Variables (`.env`)**

```bash
# Database Configuration
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=your_user
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=hotsearch

# LLM Configuration (OpenAI-compatible API)
OPENAI_API_KEY=your_api_key
OPENAI_API_BASE=https://api.your-provider.com/v1
MODEL_NAME=gpt-4  # or pangu-embedded-7b if local

# Pangu Model (Local Deployment)
PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model
PANGU_API_URL=http://localhost:8000  # If using API server

# Push Notification Services
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASSWORD=your_app_password

WECHAT_WORK_WEBHOOK=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY
WECHAT_WORK_CORPID=your_corp_id
WECHAT_WORK_SECRET=your_secret
WECHAT_WORK_AGENTID=your_agent_id

TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

**Crawler Configuration (`hotsearchcrawler/settings.py`)**

```python
# MySQL Connection
MYSQL_SETTINGS = {
    'host': os.getenv('MYSQL_HOST', 'localhost'),
    'port': int(os.getenv('MYSQL_PORT', 3306)),
    'user': os.getenv('MYSQL_USER'),
    'password': os.getenv('MYSQL_PASSWORD'),
    'database': os.getenv('MYSQL_DATABASE'),
}

# Optional: Platform cookies for authenticated crawling
COOKIES = {
    'weibo': 'your_weibo_cookie',
    'bilibili': 'your_bilibili_cookie',
}

# Crawler settings
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 1
ROBOTSTXT_OBEY = False
```

## Core Usage

### Starting the System

**1. Launch Web Application**

```bash
# Main application entry point
python app.py

# Access at http://localhost:5000
```

**2. Control Crawlers via Web Interface**

The web interface provides hotkey controls:
- **Start Crawlers**: Triggers `run_spiders.py` in background
- **Stop Crawlers**: Gracefully terminates crawler processes
- View real-time crawler status and logs

**3. Manual Crawler Testing**

```bash
# Test individual spider
cd hotsearchcrawler
python runspider-test.py

# Or run specific spider
scrapy crawl weibo_hot
scrapy crawl bilibili_hot
```

### LLM-Powered Analysis

**Conversational Query Interface**

```python
# In hotsearch_analysis_agent/agent.py
from langchain.agents import initialize_agent, Tool
from langchain.chat_models import ChatOpenAI
from langchain.memory import ConversationBufferMemory

class OpinionAnalysisAgent:
    def __init__(self):
        self.llm = ChatOpenAI(
            model_name=os.getenv('MODEL_NAME', 'gpt-4'),
            temperature=0.7
        )
        self.memory = ConversationBufferMemory()
        
        tools = [
            Tool(
                name="SearchHotTopics",
                func=self.search_hot_topics,
                description="Search hot topics by keyword, platform, or time range"
            ),
            Tool(
                name="AnalyzeSentiment",
                func=self.analyze_sentiment,
                description="Analyze sentiment of specific topic or news"
            ),
            Tool(
                name="ClusterTopics",
                func=self.cluster_topics,
                description="Cluster related topics and identify trends"
            )
        ]
        
        self.agent = initialize_agent(
            tools, self.llm, 
            agent="conversational-react-description",
            memory=self.memory,
            verbose=True
        )
    
    def query(self, user_input):
        return self.agent.run(user_input)

# Usage
agent = OpinionAnalysisAgent()
result = agent.query("分析最近关于人工智能的热点话题")
```

**Topic Clustering**

```python
from sklearn.cluster import DBSCAN
from sentence_transformers import SentenceTransformer

class TopicClusterAnalyzer:
    def __init__(self):
        self.model = SentenceTransformer('paraphrase-multilingual-MiniLM-L12-v2')
    
    def cluster_topics(self, topics):
        # topics: List[Dict] with keys: title, content, platform
        texts = [f"{t['title']} {t['content']}" for t in topics]
        embeddings = self.model.encode(texts)
        
        clustering = DBSCAN(eps=0.3, min_samples=2).fit(embeddings)
        
        clusters = {}
        for idx, label in enumerate(clustering.labels_):
            if label not in clusters:
                clusters[label] = []
            clusters[label].append(topics[idx])
        
        return self.summarize_clusters(clusters)
    
    def summarize_clusters(self, clusters):
        summaries = {}
        for label, topics in clusters.items():
            if label == -1:  # Noise
                continue
            
            # Use LLM to generate cluster summary
            prompt = f"总结以下相关话题的共同主题:\n" + \
                     "\n".join([t['title'] for t in topics[:5]])
            
            summary = self.llm_summarize(prompt)
            summaries[label] = {
                'summary': summary,
                'topics': topics,
                'count': len(topics)
            }
        
        return summaries
```

**Sentiment Analysis**

```python
from transformers import pipeline

class SentimentAnalyzer:
    def __init__(self):
        # Use Chinese sentiment model
        self.classifier = pipeline(
            "sentiment-analysis",
            model="uer/roberta-base-finetuned-jd-binary-chinese"
        )
    
    def analyze_news(self, news_content):
        result = self.classifier(news_content[:512])  # Truncate for model
        
        return {
            'label': result[0]['label'],  # POSITIVE/NEGATIVE
            'score': result[0]['score'],
            'interpretation': self.interpret_sentiment(result[0])
        }
    
    def batch_analyze(self, news_list):
        results = []
        for news in news_list:
            sentiment = self.analyze_news(news['content'])
            results.append({
                'title': news['title'],
                'url': news['url'],
                'sentiment': sentiment
            })
        return results
```

### Push Notification System

**Configure Push Tasks**

```python
# In test_push_task.py
from hotsearch_analysis_agent.push import PushTaskManager

push_manager = PushTaskManager()

# Create push task
task = push_manager.create_task(
    name="AI科技热点监控",
    keywords=["人工智能", "大模型", "芯片"],
    platforms=["weibo", "zhihu", "36kr"],
    channels=["email", "wechat_work", "telegram"],
    frequency="daily",  # or "realtime", "hourly"
    threshold={
        "heat_score": 1000,  # Minimum heat score
        "sentiment_alert": "negative"  # Alert on negative sentiment
    }
)

# Test push
push_manager.test_push(task_id=task.id)
```

**Email Push Implementation**

```python
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart

class EmailPusher:
    def __init__(self):
        self.smtp_host = os.getenv('SMTP_HOST')
        self.smtp_port = int(os.getenv('SMTP_PORT'))
        self.smtp_user = os.getenv('SMTP_USER')
        self.smtp_password = os.getenv('SMTP_PASSWORD')
    
    def send_report(self, recipients, subject, content):
        msg = MIMEMultipart('alternative')
        msg['From'] = self.smtp_user
        msg['To'] = ', '.join(recipients)
        msg['Subject'] = subject
        
        html_content = self.generate_html_report(content)
        msg.attach(MIMEText(html_content, 'html', 'utf-8'))
        
        with smtplib.SMTP(self.smtp_host, self.smtp_port) as server:
            server.starttls()
            server.login(self.smtp_user, self.smtp_password)
            server.send_message(msg)
    
    def generate_html_report(self, content):
        # content: Dict with clusters, sentiment, top_topics
        html = f"""
        <html>
        <body>
            <h1>舆情分析报告</h1>
            <h2>时间: {content['timestamp']}</h2>
            
            <h3>核心发现</h3>
            <ul>
                {''.join([f"<li>{finding}</li>" for finding in content['findings']])}
            </ul>
            
            <h3>热点话题</h3>
            {''.join([self.format_topic(t) for t in content['top_topics']])}
        </body>
        </html>
        """
        return html
```

**WeChat Work Push**

```python
import requests

class WeChatWorkPusher:
    def __init__(self):
        self.webhook = os.getenv('WECHAT_WORK_WEBHOOK')
    
    def send_markdown(self, title, content):
        payload = {
            "msgtype": "markdown",
            "markdown": {
                "content": f"# {title}\n\n{content}"
            }
        }
        
        response = requests.post(self.webhook, json=payload)
        return response.json()
    
    def send_news_card(self, articles):
        payload = {
            "msgtype": "news",
            "news": {
                "articles": [
                    {
                        "title": article['title'],
                        "description": article['summary'],
                        "url": article['url'],
                        "picurl": article.get('image', '')
                    }
                    for article in articles[:8]  # Max 8 articles
                ]
            }
        }
        
        response = requests.post(self.webhook, json=payload)
        return response.json()
```

### Database Operations

**Query Hot Topics**

```python
import pymysql
from contextlib import contextmanager

class HotSearchDB:
    def __init__(self):
        self.config = {
            'host': os.getenv('MYSQL_HOST'),
            'port': int(os.getenv('MYSQL_PORT')),
            'user': os.getenv('MYSQL_USER'),
            'password': os.getenv('MYSQL_PASSWORD'),
            'database': os.getenv('MYSQL_DATABASE'),
            'charset': 'utf8mb4'
        }
    
    @contextmanager
    def get_connection(self):
        conn = pymysql.connect(**self.config)
        try:
            yield conn
        finally:
            conn.close()
    
    def get_hot_topics(self, platform=None, limit=50, hours=24):
        with self.get_connection() as conn:
            cursor = conn.cursor(pymysql.cursors.DictCursor)
            
            query = """
                SELECT * FROM hot_topics
                WHERE created_at > DATE_SUB(NOW(), INTERVAL %s HOUR)
            """
            params = [hours]
            
            if platform:
                query += " AND platform = %s"
                params.append(platform)
            
            query += " ORDER BY heat_score DESC LIMIT %s"
            params.append(limit)
            
            cursor.execute(query, params)
            return cursor.fetchall()
    
    def search_topics(self, keywords, date_range=None):
        with self.get_connection() as conn:
            cursor = conn.cursor(pymysql.cursors.DictCursor)
            
            query = """
                SELECT * FROM hot_topics
                WHERE title LIKE %s OR content LIKE %s
            """
            keyword_pattern = f"%{keywords}%"
            params = [keyword_pattern, keyword_pattern]
            
            if date_range:
                query += " AND created_at BETWEEN %s AND %s"
                params.extend(date_range)
            
            cursor.execute(query, params)
            return cursor.fetchall()
```

## Common Patterns

### Complete Analysis Pipeline

```python
from hotsearch_analysis_agent import OpinionAnalysisAgent, TopicClusterAnalyzer
from hotsearchcrawler import get_latest_topics

def analyze_trending_topics(keywords, platforms):
    # Step 1: Fetch latest data
    db = HotSearchDB()
    topics = []
    for platform in platforms:
        topics.extend(db.search_topics(keywords, platform=platform))
    
    # Step 2: Cluster related topics
    cluster_analyzer = TopicClusterAnalyzer()
    clusters = cluster_analyzer.cluster_topics(topics)
    
    # Step 3: Sentiment analysis
    sentiment_analyzer = SentimentAnalyzer()
    for cluster in clusters.values():
        cluster['sentiment'] = sentiment_analyzer.batch_analyze(cluster['topics'])
    
    # Step 4: LLM-powered summary
    agent = OpinionAnalysisAgent()
    summary = agent.query(
        f"基于以下聚类结果生成分析报告: {clusters}"
    )
    
    # Step 5: Push to channels
    push_manager = PushTaskManager()
    push_manager.send_report(
        channels=['email', 'wechat_work'],
        report={
            'clusters': clusters,
            'summary': summary,
            'timestamp': datetime.now()
        }
    )
    
    return summary
```

### Custom Spider Development

```python
# In hotsearchcrawler/spiders/custom_spider.py
import scrapy
from hotsearchcrawler.items import HotTopicItem

class CustomPlatformSpider(scrapy.Spider):
    name = 'custom_platform'
    allowed_domains = ['example.com']
    start_urls = ['https://example.com/hot']
    
    def parse(self, response):
        for item in response.css('.hot-item'):
            hot_topic = HotTopicItem()
            hot_topic['platform'] = 'custom_platform'
            hot_topic['title'] = item.css('.title::text').get()
            hot_topic['url'] = item.css('a::attr(href)').get()
            hot_topic['heat_score'] = int(item.css('.score::text').get())
            hot_topic['rank'] = item.css('.rank::text').get()
            
            # Follow to detail page
            yield response.follow(
                hot_topic['url'],
                callback=self.parse_detail,
                meta={'item': hot_topic}
            )
    
    def parse_detail(self, response):
        item = response.meta['item']
        item['content'] = response.css('.content::text').getall()
        item['author'] = response.css('.author::text').get()
        item['publish_time'] = response.css('.time::text').get()
        
        yield item
```

## Troubleshooting

### Browser Driver Issues

```bash
# Error: "chromedriver executable needs to be in PATH"
# Solution: Verify driver location
which chromedriver  # Linux/Mac
where chromedriver  # Windows

# Add to PATH if needed
export PATH=$PATH:/path/to/driver  # Linux/Mac
```

### MySQL Connection Errors

```python
# Error: "2003: Can't connect to MySQL server"
# Check connection in Python
import pymysql
try:
    conn = pymysql.connect(
        host=os.getenv('MYSQL_HOST'),
        port=int(os.getenv('MYSQL_PORT')),
        user=os.getenv('MYSQL_USER'),
        password=os.getenv('MYSQL_PASSWORD')
    )
    print("Connection successful")
except Exception as e:
    print(f"Error: {e}")
```

### LLM API Timeout

```python
# Increase timeout for slow LLM responses
from langchain.chat_models import ChatOpenAI

llm = ChatOpenAI(
    model_name=os.getenv('MODEL_NAME'),
    request_timeout=120,  # 2 minutes
    max_retries=3
)
```

### Crawler Rate Limiting

```python
# In hotsearchcrawler/settings.py
# Adjust these settings if getting blocked
DOWNLOAD_DELAY = 3  # Increase delay between requests
CONCURRENT_REQUESTS_PER_DOMAIN = 1  # Reduce concurrency
AUTOTHROTTLE_ENABLED = True
AUTOTHROTTLE_START_DELAY = 5
```

### Memory Issues with Large Datasets

```python
# Process topics in batches
def process_large_dataset(topics, batch_size=100):
    for i in range(0, len(topics), batch_size):
        batch = topics[i:i+batch_size]
        results = analyze_batch(batch)
        save_to_db(results)
        # Clear memory
        del results
```

## Advanced Configuration

### Using Pangu Model Locally

```bash
# Download model from: https://ai.gitcode.com/ascend-tribe/openpangu-embedded-7b-model

# Set environment variable
export PANGU_MODEL_PATH=/path/to/openpangu-embedded-7b-model

# In Python code
from transformers import AutoModelForCausalLM, AutoTokenizer

model = AutoModelForCausalLM.from_pretrained(
    os.getenv('PANGU_MODEL_PATH'),
    device_map="auto"
)
tokenizer = AutoTokenizer.from_pretrained(os.getenv('PANGU_MODEL_PATH'))

def query_pangu(prompt):
    inputs = tokenizer(prompt, return_tensors="pt")
    outputs = model.generate(**inputs, max_length=512)
    return tokenizer.decode(outputs[0])
```

### Custom Analysis Prompt Templates

```python
ANALYSIS_PROMPTS = {
    'sentiment': """
    分析以下新闻的情感倾向(正面/负面/中性):
    标题: {title}
    内容: {content}
    
    请提供:
    1. 情感分类
    2. 置信度
    3. 关键情感词
    """,
    
    'cluster_summary': """
    以下是{count}条相关话题,请生成简洁的主题总结:
    {topics}
    
    要求:
    - 100字以内
    - 突出核心事件
    - 识别争议点
    """
}
```

This skill enables AI agents to help developers deploy and use the complete public opinion monitoring and analysis system with multi-platform crawling, LLM analysis, and automated push notifications.
