---
name: harvard-art-museums-data-pipeline
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Streamlit, SQL, and Plotly
triggers:
  - how do I build a data pipeline with Harvard Art Museums API
  - create ETL for museum artifacts data
  - build analytics dashboard with Streamlit and SQL
  - fetch and analyze Harvard museum collection data
  - how to structure museum API data in SQL database
  - visualize art artifacts data with Plotly
  - create museum data engineering pipeline
  - query Harvard Art Museums API and store in database
---

# Harvard Art Museums Data Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, analytics queries, and interactive Streamlit dashboards with Plotly visualizations.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Secure data collection from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract artifact metadata, transform nested JSON into relational structure, load into SQL
- **Database Design**: Three-table schema (metadata, media, colors) with foreign key relationships
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Interactive Dashboard**: Streamlit-based UI with Plotly visualizations

**Architecture Flow**: API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages**:
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### 1. Harvard Art Museums API Key

Get your API key from: https://www.harvardartmuseums.org/collections/api

```python
# Store in environment variable
import os
os.environ['HARVARD_API_KEY'] = 'your_api_key_here'

# Or use .env file
# .env
HARVARD_API_KEY=your_api_key_here
```

### 2. Database Configuration

```python
# MySQL/TiDB connection config
DB_CONFIG = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}
```

## Core Components

### 1. API Data Collection

```python
import requests
import os

class HarvardAPIClient:
    def __init__(self, api_key=None):
        self.api_key = api_key or os.getenv('HARVARD_API_KEY')
        self.base_url = "https://api.harvardartmuseums.org"
    
    def fetch_objects(self, size=100, page=1):
        """Fetch artifact objects with pagination"""
        url = f"{self.base_url}/object"
        params = {
            'apikey': self.api_key,
            'size': size,
            'page': page
        }
        response = requests.get(url, params=params)
        response.raise_for_status()
        return response.json()
    
    def collect_artifacts(self, total_records=1000):
        """Collect artifacts with pagination handling"""
        artifacts = []
        page = 1
        size = 100
        
        while len(artifacts) < total_records:
            data = self.fetch_objects(size=size, page=page)
            records = data.get('records', [])
            if not records:
                break
            artifacts.extend(records)
            page += 1
        
        return artifacts[:total_records]
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector

class ArtifactETL:
    def __init__(self, db_config):
        self.db_config = db_config
        self.conn = None
    
    def connect(self):
        """Establish database connection"""
        self.conn = mysql.connector.connect(**self.db_config)
        return self.conn.cursor()
    
    def create_tables(self):
        """Create database schema"""
        cursor = self.connect()
        
        # Metadata table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactmetadata (
                objectid INT PRIMARY KEY,
                title VARCHAR(500),
                culture VARCHAR(200),
                century VARCHAR(100),
                classification VARCHAR(200),
                department VARCHAR(200),
                dated VARCHAR(200),
                period VARCHAR(200),
                url TEXT
            )
        """)
        
        # Media table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactmedia (
                id INT AUTO_INCREMENT PRIMARY KEY,
                objectid INT,
                baseimageurl TEXT,
                primaryimageurl TEXT,
                FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
            )
        """)
        
        # Colors table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactcolors (
                id INT AUTO_INCREMENT PRIMARY KEY,
                objectid INT,
                color VARCHAR(50),
                spectrum VARCHAR(50),
                percent FLOAT,
                FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
            )
        """)
        
        self.conn.commit()
        cursor.close()
    
    def transform_metadata(self, artifacts):
        """Transform artifacts to metadata DataFrame"""
        metadata = []
        for obj in artifacts:
            metadata.append({
                'objectid': obj.get('objectid'),
                'title': obj.get('title'),
                'culture': obj.get('culture'),
                'century': obj.get('century'),
                'classification': obj.get('classification'),
                'department': obj.get('department'),
                'dated': obj.get('dated'),
                'period': obj.get('period'),
                'url': obj.get('url')
            })
        return pd.DataFrame(metadata)
    
    def transform_media(self, artifacts):
        """Transform artifacts to media DataFrame"""
        media = []
        for obj in artifacts:
            if obj.get('primaryimageurl') or obj.get('baseimageurl'):
                media.append({
                    'objectid': obj.get('objectid'),
                    'baseimageurl': obj.get('baseimageurl'),
                    'primaryimageurl': obj.get('primaryimageurl')
                })
        return pd.DataFrame(media)
    
    def transform_colors(self, artifacts):
        """Transform artifacts to colors DataFrame"""
        colors = []
        for obj in artifacts:
            object_colors = obj.get('colors', [])
            for color_obj in object_colors:
                colors.append({
                    'objectid': obj.get('objectid'),
                    'color': color_obj.get('color'),
                    'spectrum': color_obj.get('spectrum'),
                    'percent': color_obj.get('percent')
                })
        return pd.DataFrame(colors)
    
    def load_data(self, df, table_name):
        """Batch insert data into SQL table"""
        cursor = self.connect()
        
        # Replace NaN with None
        df = df.where(pd.notnull(df), None)
        
        # Create INSERT statement
        cols = ', '.join(df.columns)
        placeholders = ', '.join(['%s'] * len(df.columns))
        sql = f"INSERT IGNORE INTO {table_name} ({cols}) VALUES ({placeholders})"
        
        # Batch insert
        data = [tuple(row) for row in df.values]
        cursor.executemany(sql, data)
        self.conn.commit()
        cursor.close()
    
    def run_pipeline(self, artifacts):
        """Execute full ETL pipeline"""
        self.create_tables()
        
        # Transform
        metadata_df = self.transform_metadata(artifacts)
        media_df = self.transform_media(artifacts)
        colors_df = self.transform_colors(artifacts)
        
        # Load
        self.load_data(metadata_df, 'artifactmetadata')
        self.load_data(media_df, 'artifactmedia')
        self.load_data(colors_df, 'artifactcolors')
        
        return {
            'metadata_rows': len(metadata_df),
            'media_rows': len(media_df),
            'colors_rows': len(colors_df)
        }
```

### 3. SQL Analytics Queries

```python
class ArtifactAnalytics:
    def __init__(self, db_config):
        self.db_config = db_config
    
    def execute_query(self, query):
        """Execute SQL query and return DataFrame"""
        conn = mysql.connector.connect(**self.db_config)
        df = pd.read_sql(query, conn)
        conn.close()
        return df
    
    # Sample analytical queries
    QUERIES = {
        'artifacts_by_culture': """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture IS NOT NULL
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 10
        """,
        
        'artifacts_by_century': """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL
            GROUP BY century
            ORDER BY count DESC
        """,
        
        'artifacts_with_images': """
            SELECT 
                COUNT(DISTINCT m.objectid) as with_images,
                (SELECT COUNT(*) FROM artifactmetadata) as total,
                ROUND(COUNT(DISTINCT m.objectid) * 100.0 / 
                    (SELECT COUNT(*) FROM artifactmetadata), 2) as percentage
            FROM artifactmedia m
            WHERE m.primaryimageurl IS NOT NULL
        """,
        
        'top_colors': """
            SELECT color, COUNT(*) as frequency
            FROM artifactcolors
            GROUP BY color
            ORDER BY frequency DESC
            LIMIT 10
        """,
        
        'artifacts_by_department': """
            SELECT department, COUNT(*) as count
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY count DESC
        """,
        
        'color_distribution': """
            SELECT 
                spectrum,
                AVG(percent) as avg_percent,
                COUNT(*) as artifact_count
            FROM artifactcolors
            GROUP BY spectrum
            ORDER BY avg_percent DESC
        """
    }
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def create_dashboard():
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Initialize components
    api_client = HarvardAPIClient()
    etl = ArtifactETL(DB_CONFIG)
    analytics = ArtifactAnalytics(DB_CONFIG)
    
    # Sidebar
    st.sidebar.header("Configuration")
    action = st.sidebar.selectbox(
        "Select Action",
        ["Collect Data", "View Analytics", "Run Custom Query"]
    )
    
    if action == "Collect Data":
        st.header("📥 Data Collection")
        num_records = st.number_input("Number of artifacts", 100, 5000, 1000)
        
        if st.button("Fetch & Load Data"):
            with st.spinner("Collecting data..."):
                artifacts = api_client.collect_artifacts(num_records)
                st.success(f"Collected {len(artifacts)} artifacts")
            
            with st.spinner("Running ETL pipeline..."):
                results = etl.run_pipeline(artifacts)
                st.success("ETL Complete!")
                st.json(results)
    
    elif action == "View Analytics":
        st.header("📊 Analytics Dashboard")
        
        query_name = st.selectbox(
            "Select Analysis",
            list(ArtifactAnalytics.QUERIES.keys())
        )
        
        if st.button("Run Analysis"):
            query = ArtifactAnalytics.QUERIES[query_name]
            df = analytics.execute_query(query)
            
            # Display table
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df.columns) == 2 and len(df) > 0:
                fig = px.bar(
                    df,
                    x=df.columns[0],
                    y=df.columns[1],
                    title=query_name.replace('_', ' ').title()
                )
                st.plotly_chart(fig)
    
    elif action == "Run Custom Query":
        st.header("🔍 Custom SQL Query")
        custom_query = st.text_area("Enter SQL Query", height=150)
        
        if st.button("Execute"):
            try:
                df = analytics.execute_query(custom_query)
                st.dataframe(df)
            except Exception as e:
                st.error(f"Query Error: {str(e)}")

if __name__ == "__main__":
    create_dashboard()
```

## Running the Application

```bash
# Run Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Pattern 1: Incremental Data Loading

```python
def incremental_load(last_objectid):
    """Load only new artifacts since last run"""
    cursor = etl.connect()
    cursor.execute("SELECT MAX(objectid) FROM artifactmetadata")
    max_id = cursor.fetchone()[0] or 0
    
    # Fetch artifacts with objectid > max_id
    new_artifacts = api_client.fetch_objects_filter(f"objectid:>{max_id}")
    etl.run_pipeline(new_artifacts)
```

### Pattern 2: Error Handling with Rate Limiting

```python
import time

def fetch_with_retry(url, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = requests.get(url)
            if response.status_code == 429:  # Rate limited
                wait_time = int(response.headers.get('Retry-After', 60))
                time.sleep(wait_time)
                continue
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)
```

## Troubleshooting

**API Key Issues**:
- Verify key is active at Harvard Art Museums dashboard
- Check environment variable is loaded: `echo $HARVARD_API_KEY`
- Ensure `.env` file is in project root

**Database Connection Errors**:
```python
# Test connection
try:
    conn = mysql.connector.connect(**DB_CONFIG)
    print("Connection successful!")
    conn.close()
except mysql.connector.Error as err:
    print(f"Error: {err}")
```

**Rate Limiting**:
- Default limit: 2500 requests/day
- Implement exponential backoff
- Cache responses locally

**Memory Issues with Large Datasets**:
```python
# Process in chunks
def process_in_chunks(artifacts, chunk_size=100):
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i+chunk_size]
        etl.run_pipeline(chunk)
```

**Streamlit Performance**:
```python
# Use caching for expensive operations
@st.cache_data(ttl=3600)
def load_analytics_data(query):
    return analytics.execute_query(query)
```
