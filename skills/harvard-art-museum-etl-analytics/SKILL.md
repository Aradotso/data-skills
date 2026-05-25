---
name: harvard-art-museum-etl-analytics
description: ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - extract and transform Harvard museum API data
  - set up artifact collection data engineering pipeline
  - query and visualize museum collection data
  - implement SQL analytics for art museum datasets
  - create Streamlit app for museum data visualization
  - build end-to-end data pipeline for art collections
---

# Harvard Art Museum ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Harvard Artifacts Collection Data Engineering Analytics App is an end-to-end data engineering solution that extracts artifact data from the Harvard Art Museums API, transforms it into relational tables, loads it into SQL databases, and provides interactive analytics dashboards using Streamlit. The project demonstrates real-world ETL patterns, SQL analytics, and data visualization techniques.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Register for free API access
3. Add key to `.env` file

### Database Setup

```sql
-- Create database
CREATE DATABASE harvard_artifacts;

-- Create tables
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    accessionyear INT,
    medium VARCHAR(500),
    dimensions VARCHAR(500)
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    media_url TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Access the dashboard at http://localhost:8501
```

## Core Components

### 1. API Data Collection

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
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
        raise Exception(f"API request failed: {response.status_code}")

def collect_multiple_pages(num_pages=5):
    """Collect artifacts from multiple pages"""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(page=page)
        all_artifacts.extend(data.get('records', []))
    
    return all_artifacts
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

class ArtifactETL:
    def __init__(self, db_config):
        self.db_config = db_config
        self.conn = None
        
    def connect_db(self):
        """Establish database connection"""
        self.conn = mysql.connector.connect(
            host=self.db_config['host'],
            port=self.db_config['port'],
            user=self.db_config['user'],
            password=self.db_config['password'],
            database=self.db_config['database']
        )
        return self.conn
    
    def extract(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Extract relevant fields from API response"""
        extracted_data = []
        
        for artifact in artifacts:
            extracted_data.append({
                'id': artifact.get('id'),
                'title': artifact.get('title'),
                'culture': artifact.get('culture'),
                'century': artifact.get('century'),
                'classification': artifact.get('classification'),
                'department': artifact.get('department'),
                'dated': artifact.get('dated'),
                'accessionyear': artifact.get('accessionyear'),
                'medium': artifact.get('medium'),
                'dimensions': artifact.get('dimensions'),
                'images': artifact.get('images', []),
                'colors': artifact.get('colors', [])
            })
        
        return pd.DataFrame(extracted_data)
    
    def transform(self, df: pd.DataFrame) -> Dict[str, pd.DataFrame]:
        """Transform data into normalized tables"""
        # Metadata table
        metadata_df = df[['id', 'title', 'culture', 'century', 
                          'classification', 'department', 'dated', 
                          'accessionyear', 'medium', 'dimensions']].copy()
        
        # Media table (flatten images)
        media_records = []
        for _, row in df.iterrows():
            for img in row['images']:
                media_records.append({
                    'artifact_id': row['id'],
                    'media_type': 'image',
                    'media_url': img.get('baseimageurl')
                })
        media_df = pd.DataFrame(media_records)
        
        # Colors table (flatten colors)
        color_records = []
        for _, row in df.iterrows():
            for color in row['colors']:
                color_records.append({
                    'artifact_id': row['id'],
                    'color': color.get('color'),
                    'percentage': color.get('percent')
                })
        colors_df = pd.DataFrame(color_records)
        
        return {
            'metadata': metadata_df,
            'media': media_df,
            'colors': colors_df
        }
    
    def load(self, transformed_data: Dict[str, pd.DataFrame]):
        """Load data into SQL database"""
        cursor = self.conn.cursor()
        
        # Load metadata
        metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, 
         dated, accessionyear, medium, dimensions)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        
        for _, row in transformed_data['metadata'].iterrows():
            cursor.execute(metadata_query, tuple(row))
        
        # Load media
        media_query = """
        INSERT INTO artifactmedia (artifact_id, media_type, media_url)
        VALUES (%s, %s, %s)
        """
        
        for _, row in transformed_data['media'].iterrows():
            cursor.execute(media_query, tuple(row))
        
        # Load colors
        colors_query = """
        INSERT INTO artifactcolors (artifact_id, color, percentage)
        VALUES (%s, %s, %s)
        """
        
        for _, row in transformed_data['colors'].iterrows():
            cursor.execute(colors_query, tuple(row))
        
        self.conn.commit()
        cursor.close()
```

### 3. SQL Analytics Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "top_colors": """
        SELECT color, COUNT(*) as usage_count, AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "department_distribution": """
        SELECT department, COUNT(*) as artifacts
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifacts DESC
    """,
    
    "media_availability": """
        SELECT 
            COUNT(DISTINCT m.artifact_id) as artifacts_with_media,
            COUNT(*) as total_media_items,
            AVG(images_per_artifact) as avg_images
        FROM artifactmedia m
        JOIN (
            SELECT artifact_id, COUNT(*) as images_per_artifact
            FROM artifactmedia
            GROUP BY artifact_id
        ) sub ON m.artifact_id = sub.artifact_id
    """
}

def execute_analytics_query(conn, query_name):
    """Execute a predefined analytics query"""
    cursor = conn.cursor(dictionary=True)
    query = ANALYTICS_QUERIES.get(query_name)
    
    if not query:
        raise ValueError(f"Query '{query_name}' not found")
    
    cursor.execute(query)
    results = cursor.fetchall()
    cursor.close()
    
    return pd.DataFrame(results)
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "ETL Pipeline", "Analytics Dashboard"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "ETL Pipeline":
        show_etl_pipeline()
    elif page == "Analytics Dashboard":
        show_analytics()

def show_data_collection():
    st.header("📡 Data Collection")
    
    num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=50, value=5)
    
    if st.button("Fetch Artifacts"):
        with st.spinner("Fetching data from API..."):
            artifacts = collect_multiple_pages(num_pages)
            st.success(f"Collected {len(artifacts)} artifacts")
            st.json(artifacts[0])  # Show sample

def show_etl_pipeline():
    st.header("⚙️ ETL Pipeline")
    
    if st.button("Run ETL"):
        with st.spinner("Running ETL pipeline..."):
            # Fetch data
            artifacts = collect_multiple_pages(5)
            
            # Initialize ETL
            etl = ArtifactETL(db_config)
            etl.connect_db()
            
            # Extract
            df = etl.extract(artifacts)
            st.write(f"Extracted {len(df)} records")
            
            # Transform
            transformed = etl.transform(df)
            st.write("Transformed into tables:")
            for table_name, table_df in transformed.items():
                st.write(f"- {table_name}: {len(table_df)} rows")
            
            # Load
            etl.load(transformed)
            st.success("Data loaded successfully!")

def show_analytics():
    st.header("📊 Analytics Dashboard")
    
    # Query selector
    query_name = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Run Query"):
        conn = mysql.connector.connect(**db_config)
        df = execute_analytics_query(conn, query_name)
        
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1])
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(page, delay=0.5):
    """Fetch artifacts with rate limiting"""
    data = fetch_artifacts(page)
    time.sleep(delay)  # Respect API rate limits
    return data
```

### Batch Processing

```python
def batch_insert(cursor, query, data, batch_size=1000):
    """Insert data in batches for better performance"""
    for i in range(0, len(data), batch_size):
        batch = data[i:i + batch_size]
        cursor.executemany(query, batch)
        cursor.connection.commit()
```

### Error Handling

```python
def safe_api_call(func, *args, max_retries=3, **kwargs):
    """Retry API calls on failure"""
    for attempt in range(max_retries):
        try:
            return func(*args, **kwargs)
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

## Troubleshooting

### API Key Issues
- Ensure `HARVARD_API_KEY` is set in `.env`
- Verify API key is active at harvardartmuseums.org
- Check rate limits (typically 2500 requests/day)

### Database Connection
- Verify MySQL/TiDB credentials in `.env`
- Check firewall allows connections on port 3306
- Ensure database and tables exist

### Missing Data
- API may return null values for some fields
- Use `.get()` method with defaults: `artifact.get('culture', 'Unknown')`
- Filter records: `'hasimage': 1` parameter

### Memory Issues with Large Datasets
- Process data in smaller batches
- Use pagination with smaller page sizes
- Clear DataFrames after loading: `del df`

### Streamlit Performance
- Cache database connections: `@st.cache_resource`
- Cache query results: `@st.cache_data`
- Limit initial data display sizes
