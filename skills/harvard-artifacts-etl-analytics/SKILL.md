---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums API
  - create a data pipeline with Harvard artifacts collection
  - set up analytics dashboard for museum artifact data
  - fetch and analyze Harvard museum data with SQL
  - build a Streamlit app for art collection analytics
  - implement ETL for Harvard Art Museums API
  - create data engineering pipeline for museum artifacts
  - visualize Harvard art collection data with Python
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards.

## What It Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color data into relational databases
- **SQL Analytics**: Execute predefined analytical queries on artifact collections
- **Interactive Dashboards**: Visualize query results using Streamlit and Plotly
- **Database Management**: Store structured data in MySQL/TiDB Cloud with proper relationships

## Installation

### Prerequisites

```bash
# Python 3.8+
python --version

# Install dependencies
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Environment Setup

Create a `.env` file in the project root:

```bash
# .env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

### Database Setup

```sql
-- Create database
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    dated VARCHAR(255),
    classification VARCHAR(255),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    department VARCHAR(255),
    division VARCHAR(255),
    credit_line TEXT,
    accession_number VARCHAR(100),
    provenance TEXT,
    copyright TEXT,
    url VARCHAR(500),
    last_updated DATETIME
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(1000),
    media_type VARCHAR(100),
    rank INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_name VARCHAR(100),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Components

### 1. API Data Collection

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

class HarvardAPIClient:
    """Client for Harvard Art Museums API"""
    
    def __init__(self):
        self.api_key = os.getenv('HARVARD_API_KEY')
        self.base_url = "https://api.harvardartmuseums.org/object"
    
    def fetch_artifacts(self, size=100, page=1):
        """Fetch artifacts with pagination"""
        params = {
            'apikey': self.api_key,
            'size': size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(self.base_url, params=params)
        response.raise_for_status()
        return response.json()
    
    def fetch_all_artifacts(self, total_records=1000):
        """Fetch multiple pages of artifacts"""
        all_records = []
        page = 1
        size = 100
        
        while len(all_records) < total_records:
            data = self.fetch_artifacts(size=size, page=page)
            records = data.get('records', [])
            
            if not records:
                break
                
            all_records.extend(records)
            page += 1
            
            # Respect rate limits
            import time
            time.sleep(0.5)
        
        return all_records[:total_records]
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from datetime import datetime

class ArtifactETL:
    """ETL pipeline for artifact data"""
    
    def __init__(self):
        self.db_config = {
            'host': os.getenv('DB_HOST'),
            'port': int(os.getenv('DB_PORT', 3306)),
            'user': os.getenv('DB_USER'),
            'password': os.getenv('DB_PASSWORD'),
            'database': os.getenv('DB_NAME')
        }
    
    def extract(self, api_client, num_records=1000):
        """Extract data from API"""
        print(f"Extracting {num_records} artifacts...")
        artifacts = api_client.fetch_all_artifacts(num_records)
        return artifacts
    
    def transform(self, artifacts):
        """Transform JSON to relational format"""
        metadata_records = []
        media_records = []
        color_records = []
        
        for artifact in artifacts:
            # Metadata
            metadata = {
                'id': artifact.get('id'),
                'title': artifact.get('title', '')[:500],
                'culture': artifact.get('culture', '')[:255],
                'period': artifact.get('period', '')[:255],
                'century': artifact.get('century', '')[:255],
                'dated': artifact.get('dated', '')[:255],
                'classification': artifact.get('classification', '')[:255],
                'medium': artifact.get('medium', '')[:500],
                'dimensions': artifact.get('dimensions', '')[:500],
                'department': artifact.get('department', '')[:255],
                'division': artifact.get('division', '')[:255],
                'credit_line': artifact.get('creditline', ''),
                'accession_number': artifact.get('accessionyear', '')[:100],
                'provenance': artifact.get('provenance', ''),
                'copyright': artifact.get('copyright', ''),
                'url': artifact.get('url', '')[:500],
                'last_updated': datetime.now()
            }
            metadata_records.append(metadata)
            
            # Media
            for idx, image in enumerate(artifact.get('images', [])):
                media = {
                    'artifact_id': artifact.get('id'),
                    'image_url': image.get('baseimageurl', '')[:1000],
                    'media_type': 'image',
                    'rank': idx
                }
                media_records.append(media)
            
            # Colors
            for color in artifact.get('colors', []):
                color_record = {
                    'artifact_id': artifact.get('id'),
                    'color_hex': color.get('hex', ''),
                    'color_name': color.get('color', '')[:100],
                    'percentage': color.get('percent', 0.0)
                }
                color_records.append(color_record)
        
        return (
            pd.DataFrame(metadata_records),
            pd.DataFrame(media_records),
            pd.DataFrame(color_records)
        )
    
    def load(self, metadata_df, media_df, colors_df):
        """Load data into SQL database"""
        conn = mysql.connector.connect(**self.db_config)
        cursor = conn.cursor()
        
        try:
            # Load metadata
            for _, row in metadata_df.iterrows():
                insert_query = """
                INSERT INTO artifactmetadata 
                (id, title, culture, period, century, dated, classification, 
                 medium, dimensions, department, division, credit_line, 
                 accession_number, provenance, copyright, url, last_updated)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE
                title=VALUES(title), last_updated=VALUES(last_updated)
                """
                cursor.execute(insert_query, tuple(row))
            
            # Load media
            for _, row in media_df.iterrows():
                insert_query = """
                INSERT INTO artifactmedia (artifact_id, image_url, media_type, rank)
                VALUES (%s, %s, %s, %s)
                """
                cursor.execute(insert_query, tuple(row))
            
            # Load colors
            for _, row in colors_df.iterrows():
                insert_query = """
                INSERT INTO artifactcolors (artifact_id, color_hex, color_name, percentage)
                VALUES (%s, %s, %s, %s)
                """
                cursor.execute(insert_query, tuple(row))
            
            conn.commit()
            print(f"Loaded {len(metadata_df)} artifacts successfully")
            
        finally:
            cursor.close()
            conn.close()
    
    def run_pipeline(self, api_client, num_records=1000):
        """Execute full ETL pipeline"""
        artifacts = self.extract(api_client, num_records)
        metadata_df, media_df, colors_df = self.transform(artifacts)
        self.load(metadata_df, media_df, colors_df)
```

### 3. SQL Analytics Queries

```python
class ArtifactAnalytics:
    """Analytical queries for artifact data"""
    
    QUERIES = {
        'artifacts_by_culture': """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture IS NOT NULL AND culture != ''
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 20
        """,
        
        'artifacts_by_century': """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL AND century != ''
            GROUP BY century
            ORDER BY century
        """,
        
        'artifacts_by_department': """
            SELECT department, COUNT(*) as count
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY count DESC
        """,
        
        'top_colors': """
            SELECT color_name, COUNT(*) as frequency, AVG(percentage) as avg_percentage
            FROM artifactcolors
            WHERE color_name IS NOT NULL
            GROUP BY color_name
            ORDER BY frequency DESC
            LIMIT 15
        """,
        
        'artifacts_with_media': """
            SELECT 
                (SELECT COUNT(*) FROM artifactmetadata) as total_artifacts,
                (SELECT COUNT(DISTINCT artifact_id) FROM artifactmedia) as artifacts_with_media,
                ROUND((SELECT COUNT(DISTINCT artifact_id) FROM artifactmedia) * 100.0 / 
                      (SELECT COUNT(*) FROM artifactmetadata), 2) as percentage_with_media
        """,
        
        'classification_distribution': """
            SELECT classification, COUNT(*) as count
            FROM artifactmetadata
            WHERE classification IS NOT NULL AND classification != ''
            GROUP BY classification
            ORDER BY count DESC
            LIMIT 15
        """
    }
    
    def __init__(self, db_config):
        self.db_config = db_config
    
    def execute_query(self, query_name):
        """Execute a predefined query"""
        query = self.QUERIES.get(query_name)
        if not query:
            raise ValueError(f"Query '{query_name}' not found")
        
        conn = mysql.connector.connect(**self.db_config)
        df = pd.read_sql(query, conn)
        conn.close()
        return df
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    st.markdown("Explore artifact collections through data")
    
    # Sidebar configuration
    with st.sidebar:
        st.header("⚙️ Configuration")
        
        if st.button("Run ETL Pipeline"):
            with st.spinner("Running ETL pipeline..."):
                api_client = HarvardAPIClient()
                etl = ArtifactETL()
                etl.run_pipeline(api_client, num_records=500)
                st.success("ETL completed successfully!")
    
    # Analytics section
    st.header("📊 Analytics")
    
    analytics = ArtifactAnalytics({
        'host': os.getenv('DB_HOST'),
        'port': int(os.getenv('DB_PORT', 3306)),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    })
    
    # Query selector
    query_options = list(analytics.QUERIES.keys())
    selected_query = st.selectbox("Select Analysis", query_options)
    
    if st.button("Run Query"):
        with st.spinner("Executing query..."):
            df = analytics.execute_query(selected_query)
            
            # Display results
            st.subheader("Query Results")
            st.dataframe(df)
            
            # Visualization
            if len(df.columns) >= 2:
                st.subheader("Visualization")
                fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                            title=selected_query.replace('_', ' ').title())
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Incremental ETL

```python
def incremental_load(self, api_client):
    """Load only new artifacts since last update"""
    conn = mysql.connector.connect(**self.db_config)
    cursor = conn.cursor()
    
    # Get last update timestamp
    cursor.execute("SELECT MAX(last_updated) FROM artifactmetadata")
    last_update = cursor.fetchone()[0]
    cursor.close()
    conn.close()
    
    # Fetch only newer artifacts
    artifacts = api_client.fetch_artifacts_since(last_update)
    # Process as normal...
```

### Error Handling

```python
def safe_extract(self, api_client, num_records):
    """Extract with retry logic"""
    max_retries = 3
    for attempt in range(max_retries):
        try:
            return api_client.fetch_all_artifacts(num_records)
        except requests.RequestException as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

## Troubleshooting

### API Rate Limiting

```python
# Add delay between requests
import time

def fetch_with_rate_limit(self, size=100, page=1):
    response = requests.get(self.base_url, params=params)
    time.sleep(1)  # 1 second delay
    return response.json()
```

### Database Connection Issues

```python
# Use connection pooling
from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="artifact_pool",
    pool_size=5,
    **db_config
)

conn = db_pool.get_connection()
```

### Memory Management for Large Datasets

```python
# Process in batches
def load_in_batches(self, df, batch_size=1000):
    for i in range(0, len(df), batch_size):
        batch = df.iloc[i:i+batch_size]
        self._insert_batch(batch)
```

### Missing API Key

```python
if not self.api_key:
    raise ValueError("HARVARD_API_KEY not found in environment variables")
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# Or run ETL pipeline directly
python -c "
from etl import HarvardAPIClient, ArtifactETL
client = HarvardAPIClient()
etl = ArtifactETL()
etl.run_pipeline(client, num_records=1000)
"
```

This skill enables comprehensive data engineering workflows with the Harvard Art Museums API, from data collection through visualization.
