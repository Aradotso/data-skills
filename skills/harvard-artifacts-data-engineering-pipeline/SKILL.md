---
name: harvard-artifacts-data-engineering-pipeline
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums data
  - setup data engineering pipeline for museum artifacts
  - create analytics dashboard with Harvard API
  - process Harvard Art Museums API data into SQL
  - build Streamlit app for museum data visualization
  - extract and transform Harvard artifacts collection data
  - query and analyze museum artifact metadata
  - integrate Harvard Art Museums API with database
---

# Harvard Artifacts Collection Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL analytics, and interactive data visualization using Streamlit.

## What It Does

The Harvard Artifacts Collection Data Engineering App provides:
- Dynamic data collection from Harvard Art Museums API with pagination and rate limiting
- ETL pipeline that transforms nested JSON into relational database tables
- SQL database storage (MySQL/TiDB Cloud) with proper schema design
- 20+ predefined analytical SQL queries for artifact insights
- Interactive Streamlit dashboard with Plotly visualizations
- Analysis of artifact metadata, media availability, and color patterns

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly sqlalchemy
```

## Configuration

### Harvard API Key Setup

Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api):

```python
# Set as environment variable
export HARVARD_API_KEY="your_api_key_here"

# Or configure in Streamlit secrets (.streamlit/secrets.toml)
[api]
harvard_key = "your_api_key_here"

[database]
host = "your_db_host"
port = 4000
user = "your_db_user"
password = "your_db_password"
database = "your_db_name"
```

### Database Setup

```sql
-- Create database
CREATE DATABASE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    url TEXT,
    objectnumber VARCHAR(100),
    accessionyear INT
);

-- Artifact media table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_url TEXT,
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key API Usage Patterns

### Fetching Data from Harvard API

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts with pagination"""
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=50)
print(f"Total records: {data['info']['totalrecords']}")
print(f"Pages: {data['info']['pages']}")
```

### ETL Pipeline Implementation

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

class HarvardETL:
    def __init__(self, api_key, db_config):
        self.api_key = api_key
        self.db_config = db_config
        
    def extract(self, num_pages=5, page_size=100):
        """Extract data from Harvard API"""
        all_records = []
        
        for page in range(1, num_pages + 1):
            try:
                data = fetch_artifacts(self.api_key, page=page, size=page_size)
                all_records.extend(data['records'])
                print(f"Extracted page {page}: {len(data['records'])} records")
            except Exception as e:
                print(f"Error on page {page}: {e}")
                break
                
        return all_records
    
    def transform(self, records):
        """Transform nested JSON to relational format"""
        metadata_list = []
        media_list = []
        colors_list = []
        
        for record in records:
            # Metadata
            metadata = {
                'id': record.get('id'),
                'title': record.get('title', '')[:500],
                'culture': record.get('culture', '')[:200],
                'period': record.get('period', '')[:200],
                'century': record.get('century', '')[:100],
                'dated': record.get('dated', '')[:200],
                'classification': record.get('classification', '')[:200],
                'department': record.get('department', '')[:200],
                'division': record.get('division', '')[:200],
                'url': record.get('url', ''),
                'objectnumber': record.get('objectnumber', '')[:100],
                'accessionyear': record.get('accessionyear')
            }
            metadata_list.append(metadata)
            
            # Media
            if 'images' in record and record['images']:
                for img in record['images']:
                    media_list.append({
                        'artifact_id': record.get('id'),
                        'media_url': img.get('baseimageurl', ''),
                        'media_type': 'image'
                    })
            
            # Colors
            if 'colors' in record and record['colors']:
                for color in record['colors']:
                    colors_list.append({
                        'artifact_id': record.get('id'),
                        'color': color.get('color', ''),
                        'spectrum': color.get('spectrum', ''),
                        'percent': color.get('percent', 0.0)
                    })
        
        return {
            'metadata': pd.DataFrame(metadata_list),
            'media': pd.DataFrame(media_list),
            'colors': pd.DataFrame(colors_list)
        }
    
    def load(self, dataframes):
        """Load data into MySQL database"""
        conn = mysql.connector.connect(**self.db_config)
        cursor = conn.cursor()
        
        # Load metadata
        for _, row in dataframes['metadata'].iterrows():
            cursor.execute("""
                INSERT INTO artifactmetadata 
                (id, title, culture, period, century, dated, classification, 
                 department, division, url, objectnumber, accessionyear)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE title=VALUES(title)
            """, tuple(row))
        
        # Load media
        for _, row in dataframes['media'].iterrows():
            cursor.execute("""
                INSERT INTO artifactmedia (artifact_id, media_url, media_type)
                VALUES (%s, %s, %s)
            """, tuple(row))
        
        # Load colors
        for _, row in dataframes['colors'].iterrows():
            cursor.execute("""
                INSERT INTO artifactcolors (artifact_id, color, spectrum, percent)
                VALUES (%s, %s, %s, %s)
            """, tuple(row))
        
        conn.commit()
        cursor.close()
        conn.close()
        print("Data loaded successfully")

# Usage
db_config = {
    'host': os.getenv('DB_HOST'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

etl = HarvardETL(api_key=os.getenv('HARVARD_API_KEY'), db_config=db_config)
records = etl.extract(num_pages=3, page_size=100)
dataframes = etl.transform(records)
etl.load(dataframes)
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px
import pandas as pd

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

# Sidebar configuration
with st.sidebar:
    st.title("🎨 Harvard Artifacts")
    st.write("Data Engineering & Analytics")
    
    # API Configuration
    api_key = st.text_input("Harvard API Key", type="password", 
                            value=os.getenv('HARVARD_API_KEY', ''))
    
    # Database Configuration
    db_host = st.text_input("Database Host", value=os.getenv('DB_HOST', ''))
    db_user = st.text_input("Database User", value=os.getenv('DB_USER', ''))
    db_password = st.text_input("Database Password", type="password")
    db_name = st.text_input("Database Name", value="harvard_artifacts")

# Main page
st.title("Harvard Artifacts Collection Analytics Dashboard")

# ETL Section
if st.button("Run ETL Pipeline"):
    with st.spinner("Running ETL pipeline..."):
        try:
            db_config = {
                'host': db_host,
                'user': db_user,
                'password': db_password,
                'database': db_name
            }
            
            etl = HarvardETL(api_key, db_config)
            records = etl.extract(num_pages=2)
            dataframes = etl.transform(records)
            etl.load(dataframes)
            
            st.success(f"✅ ETL Complete! Processed {len(records)} records")
        except Exception as e:
            st.error(f"❌ ETL Failed: {str(e)}")

# Analytics Queries
st.header("📊 Analytics Queries")

queries = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE culture IS NOT NULL 
        GROUP BY culture 
        ORDER BY count DESC 
        LIMIT 15
    """,
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE century IS NOT NULL 
        GROUP BY century 
        ORDER BY count DESC
    """,
    "Color Distribution": """
        SELECT color, COUNT(*) as count, AVG(percent) as avg_percent
        FROM artifactcolors 
        GROUP BY color 
        ORDER BY count DESC 
        LIMIT 10
    """,
    "Department Distribution": """
        SELECT department, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE department IS NOT NULL 
        GROUP BY department 
        ORDER BY count DESC
    """,
    "Media Availability": """
        SELECT 
            COUNT(DISTINCT am.artifact_id) as artifacts_with_media,
            (SELECT COUNT(*) FROM artifactmetadata) as total_artifacts
        FROM artifactmedia am
    """
}

selected_query = st.selectbox("Select Analysis", list(queries.keys()))

if st.button("Run Query"):
    try:
        conn = mysql.connector.connect(
            host=db_host,
            user=db_user,
            password=db_password,
            database=db_name
        )
        
        df = pd.read_sql(queries[selected_query], conn)
        conn.close()
        
        st.subheader("Query Results")
        st.dataframe(df)
        
        # Visualization
        if len(df.columns) == 2 and 'count' in df.columns:
            fig = px.bar(df, x=df.columns[0], y='count', 
                        title=f"{selected_query} Analysis")
            st.plotly_chart(fig, use_container_width=True)
            
    except Exception as e:
        st.error(f"Query failed: {str(e)}")
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Analytical Queries

```sql
-- Top 10 cultures by artifact count
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10;

-- Color usage across artifacts
SELECT c.color, c.spectrum, 
       COUNT(*) as usage_count,
       AVG(c.percent) as avg_coverage
FROM artifactcolors c
GROUP BY c.color, c.spectrum
ORDER BY usage_count DESC;

-- Artifacts with complete metadata
SELECT 
    COUNT(*) as total,
    SUM(CASE WHEN culture IS NOT NULL THEN 1 ELSE 0 END) as with_culture,
    SUM(CASE WHEN century IS NOT NULL THEN 1 ELSE 0 END) as with_century,
    SUM(CASE WHEN department IS NOT NULL THEN 1 ELSE 0 END) as with_department
FROM artifactmetadata;

-- Media richness analysis
SELECT 
    COUNT(DISTINCT artifact_id) as artifacts_with_images,
    COUNT(*) as total_images,
    AVG(images_per_artifact) as avg_images
FROM (
    SELECT artifact_id, COUNT(*) as images_per_artifact
    FROM artifactmedia
    GROUP BY artifact_id
) as subquery;
```

## Troubleshooting

**API Rate Limiting**
```python
import time

def fetch_with_retry(api_key, page, max_retries=3):
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except Exception as e:
            if "429" in str(e):  # Rate limit
                wait_time = (attempt + 1) * 5
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

**Database Connection Issues**
```python
def test_db_connection(db_config):
    try:
        conn = mysql.connector.connect(**db_config)
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        cursor.close()
        conn.close()
        return result[0] == 1
    except Exception as e:
        print(f"Connection failed: {e}")
        return False
```

**Memory Issues with Large Datasets**
```python
def batch_load(dataframes, batch_size=1000):
    """Load data in batches to avoid memory issues"""
    for df_name, df in dataframes.items():
        for i in range(0, len(df), batch_size):
            batch = df.iloc[i:i+batch_size]
            # Load batch to database
            print(f"Loaded {i+len(batch)}/{len(df)} rows of {df_name}")
```

This skill enables AI agents to help developers build complete data engineering pipelines from API extraction through SQL analytics to interactive visualization dashboards.
