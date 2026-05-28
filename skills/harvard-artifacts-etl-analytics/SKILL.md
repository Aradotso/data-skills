---
name: harvard-artifacts-etl-analytics
description: End-to-end ETL pipeline and analytics app for Harvard Art Museums API data with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums API
  - create a data engineering project with museum artifacts
  - set up Harvard Art Museums data analytics dashboard
  - extract and analyze Harvard museum collection data
  - build a Streamlit app for art museum data
  - create SQL analytics for Harvard artifacts
  - design an ETL workflow for museum API data
  - visualize Harvard Art Museums collection insights
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is an end-to-end data engineering and analytics application that demonstrates real-world ETL pipelines using the Harvard Art Museums API. It extracts artifact data, transforms it into relational structures, loads it into SQL databases (MySQL/TiDB Cloud), and provides interactive analytics dashboards via Streamlit.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

### Required Dependencies

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

Get your API key from: https://docs.harvardartmuseums.org/

Store it in environment variables or `.env` file:

```bash
export HARVARD_API_KEY="your_api_key_here"
```

### 2. Database Configuration

Set up your MySQL/TiDB Cloud credentials:

```python
# .env file
DB_HOST="your_host"
DB_PORT="4000"
DB_USER="your_username"
DB_PASSWORD="your_password"
DB_NAME="harvard_artifacts"
```

### 3. Database Schema Setup

```sql
-- Create artifactmetadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    dated VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    classification VARCHAR(255),
    medium VARCHAR(500),
    technique VARCHAR(500),
    period VARCHAR(255),
    accession_number VARCHAR(100),
    total_unique_pages INT,
    rank INT,
    url VARCHAR(500),
    image_count INT,
    last_updated DATETIME
);

-- Create artifactmedia table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    base_url VARCHAR(500),
    format VARCHAR(50),
    height INT,
    width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Create artifactcolors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_name VARCHAR(100),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core ETL Pipeline

### Extract Phase

```python
import requests
import os
from typing import Dict, List

def fetch_artifacts(api_key: str, page: int = 1, size: int = 100) -> Dict:
    """
    Fetch artifacts from Harvard Art Museums API
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    
    return response.json()

def collect_artifacts(api_key: str, total_records: int = 500) -> List[Dict]:
    """
    Collect multiple pages of artifacts with pagination
    """
    artifacts = []
    page = 1
    size = 100
    
    while len(artifacts) < total_records:
        data = fetch_artifacts(api_key, page=page, size=size)
        records = data.get('records', [])
        
        if not records:
            break
            
        artifacts.extend(records)
        page += 1
    
    return artifacts[:total_records]
```

### Transform Phase

```python
import pandas as pd

def transform_artifact_metadata(artifacts: List[Dict]) -> pd.DataFrame:
    """
    Transform raw artifact data into structured metadata
    """
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'classification': artifact.get('classification'),
            'medium': artifact.get('medium'),
            'technique': artifact.get('technique'),
            'period': artifact.get('period'),
            'accession_number': artifact.get('accessionyear'),
            'total_unique_pages': artifact.get('totalpageviews', 0),
            'rank': artifact.get('rank', 0),
            'url': artifact.get('url'),
            'image_count': len(artifact.get('images', [])),
            'last_updated': artifact.get('lastupdate')
        })
    
    return pd.DataFrame(metadata)

def transform_artifact_media(artifacts: List[Dict]) -> pd.DataFrame:
    """
    Extract media/image data from artifacts
    """
    media_data = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for image in images:
            media_data.append({
                'artifact_id': artifact_id,
                'base_url': image.get('baseimageurl'),
                'format': image.get('format'),
                'height': image.get('height'),
                'width': image.get('width')
            })
    
    return pd.DataFrame(media_data)

def transform_artifact_colors(artifacts: List[Dict]) -> pd.DataFrame:
    """
    Extract color information from artifacts
    """
    color_data = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data.append({
                'artifact_id': artifact_id,
                'color_hex': color.get('hex'),
                'color_name': color.get('color'),
                'percentage': color.get('percent')
            })
    
    return pd.DataFrame(color_data)
```

### Load Phase

```python
import mysql.connector
from typing import Optional

class DatabaseLoader:
    def __init__(self, host: str, port: int, user: str, password: str, database: str):
        self.config = {
            'host': host,
            'port': port,
            'user': user,
            'password': password,
            'database': database
        }
        self.connection: Optional[mysql.connector.connection] = None
    
    def connect(self):
        """Establish database connection"""
        self.connection = mysql.connector.connect(**self.config)
        return self.connection
    
    def load_metadata(self, df: pd.DataFrame):
        """Load artifact metadata into database"""
        cursor = self.connection.cursor()
        
        insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, dated, department, division, 
         classification, medium, technique, period, accession_number, 
         total_unique_pages, rank, url, image_count, last_updated)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
        """
        
        data = [tuple(row) for row in df.values]
        cursor.executemany(insert_query, data)
        self.connection.commit()
        cursor.close()
    
    def load_media(self, df: pd.DataFrame):
        """Load media data into database"""
        cursor = self.connection.cursor()
        
        insert_query = """
        INSERT INTO artifactmedia (artifact_id, base_url, format, height, width)
        VALUES (%s, %s, %s, %s, %s)
        """
        
        data = [tuple(row) for row in df.values]
        cursor.executemany(insert_query, data)
        self.connection.commit()
        cursor.close()
    
    def load_colors(self, df: pd.DataFrame):
        """Load color data into database"""
        cursor = self.connection.cursor()
        
        insert_query = """
        INSERT INTO artifactcolors (artifact_id, color_hex, color_name, percentage)
        VALUES (%s, %s, %s, %s)
        """
        
        data = [tuple(row) for row in df.values]
        cursor.executemany(insert_query, data)
        self.connection.commit()
        cursor.close()
    
    def close(self):
        """Close database connection"""
        if self.connection:
            self.connection.close()
```

## Complete ETL Workflow

```python
import os
from dotenv import load_dotenv

def run_etl_pipeline():
    """
    Execute complete ETL pipeline
    """
    # Load environment variables
    load_dotenv()
    
    # Extract
    api_key = os.getenv('HARVARD_API_KEY')
    print("Extracting data from Harvard API...")
    artifacts = collect_artifacts(api_key, total_records=500)
    
    # Transform
    print("Transforming data...")
    metadata_df = transform_artifact_metadata(artifacts)
    media_df = transform_artifact_media(artifacts)
    colors_df = transform_artifact_colors(artifacts)
    
    # Load
    print("Loading data into database...")
    db_loader = DatabaseLoader(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    db_loader.connect()
    db_loader.load_metadata(metadata_df)
    db_loader.load_media(media_df)
    db_loader.load_colors(colors_df)
    db_loader.close()
    
    print("ETL pipeline completed successfully!")

if __name__ == "__main__":
    run_etl_pipeline()
```

## SQL Analytics Queries

### Example Analytical Queries

```python
# Sample analytics queries dictionary
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE culture IS NOT NULL 
        GROUP BY culture 
        ORDER BY count DESC 
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE century IS NOT NULL 
        GROUP BY century 
        ORDER BY count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count 
        FROM artifactmetadata 
        GROUP BY department 
        ORDER BY artifact_count DESC
    """,
    
    "Media Availability": """
        SELECT 
            CASE WHEN image_count > 0 THEN 'With Images' ELSE 'No Images' END as media_status,
            COUNT(*) as count
        FROM artifactmetadata
        GROUP BY media_status
    """,
    
    "Top Colors Used": """
        SELECT color_name, COUNT(*) as usage_count, AVG(percentage) as avg_percentage
        FROM artifactcolors
        WHERE color_name IS NOT NULL
        GROUP BY color_name
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "Artifacts with Most Images": """
        SELECT title, culture, image_count
        FROM artifactmetadata
        WHERE image_count > 0
        ORDER BY image_count DESC
        LIMIT 10
    """,
    
    "Classification Breakdown": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 15
    """
}

def execute_query(connection, query: str) -> pd.DataFrame:
    """
    Execute SQL query and return results as DataFrame
    """
    return pd.read_sql(query, connection)
```

## Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def create_analytics_dashboard():
    """
    Create interactive Streamlit dashboard
    """
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Options")
    query_name = st.sidebar.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    # Database connection
    db_loader = DatabaseLoader(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    try:
        connection = db_loader.connect()
        
        # Execute selected query
        query = ANALYTICS_QUERIES[query_name]
        df_results = execute_query(connection, query)
        
        # Display results
        st.subheader(f"📊 {query_name}")
        st.dataframe(df_results, use_container_width=True)
        
        # Auto-generate visualization
        if len(df_results.columns) >= 2:
            fig = px.bar(
                df_results.head(15),
                x=df_results.columns[0],
                y=df_results.columns[1],
                title=query_name
            )
            st.plotly_chart(fig, use_container_width=True)
        
    finally:
        db_loader.close()

if __name__ == "__main__":
    create_analytics_dashboard()
```

## Running the Application

```bash
# Run ETL pipeline
python etl_pipeline.py

# Launch Streamlit dashboard
streamlit run app.py
```

## Common Patterns

### Pattern 1: Incremental Data Loading

```python
def get_last_updated_timestamp(connection):
    """Get the most recent update timestamp from database"""
    query = "SELECT MAX(last_updated) as last_update FROM artifactmetadata"
    df = pd.read_sql(query, connection)
    return df['last_update'][0]

def fetch_new_artifacts(api_key: str, since: str) -> List[Dict]:
    """Fetch only artifacts updated since last ETL run"""
    base_url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'updatedafter': since
    }
    response = requests.get(base_url, params=params)
    return response.json().get('records', [])
```

### Pattern 2: Error Handling for API Calls

```python
import time
from requests.exceptions import RequestException

def fetch_with_retry(url: str, params: dict, max_retries: int = 3) -> Dict:
    """Fetch data with retry logic and exponential backoff"""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except RequestException as e:
            if attempt == max_retries - 1:
                raise
            wait_time = 2 ** attempt
            print(f"Request failed. Retrying in {wait_time}s...")
            time.sleep(wait_time)
```

### Pattern 3: Data Quality Validation

```python
def validate_artifact_data(df: pd.DataFrame) -> pd.DataFrame:
    """Validate and clean artifact data"""
    # Remove duplicates
    df = df.drop_duplicates(subset=['id'])
    
    # Handle missing critical fields
    df = df[df['id'].notna()]
    
    # Standardize text fields
    text_columns = ['culture', 'department', 'classification']
    for col in text_columns:
        if col in df.columns:
            df[col] = df[col].str.strip().str.title()
    
    # Validate numeric fields
    if 'image_count' in df.columns:
        df['image_count'] = df['image_count'].fillna(0).astype(int)
    
    return df
```

## Troubleshooting

### Issue: API Rate Limiting

```python
# Add rate limiting to API calls
import time

def rate_limited_fetch(api_key: str, delay: float = 0.5):
    """Fetch with rate limiting"""
    artifacts = []
    for page in range(1, 11):
        data = fetch_artifacts(api_key, page=page)
        artifacts.extend(data.get('records', []))
        time.sleep(delay)  # Delay between requests
    return artifacts
```

### Issue: Database Connection Timeout

```python
# Use connection pooling
from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

def get_connection():
    return db_pool.get_connection()
```

### Issue: Large Dataset Memory Issues

```python
def batch_load_data(df: pd.DataFrame, batch_size: int = 1000):
    """Load data in batches to avoid memory issues"""
    db_loader = DatabaseLoader(...)
    db_loader.connect()
    
    for i in range(0, len(df), batch_size):
        batch = df.iloc[i:i+batch_size]
        db_loader.load_metadata(batch)
        print(f"Loaded batch {i//batch_size + 1}")
    
    db_loader.close()
```

### Issue: Missing API Key

```python
def get_api_key() -> str:
    """Safely retrieve API key with validation"""
    api_key = os.getenv('HARVARD_API_KEY')
    if not api_key:
        raise ValueError(
            "Harvard API key not found. "
            "Set HARVARD_API_KEY environment variable or add to .env file"
        )
    return api_key
```

## Best Practices

1. **Always use environment variables** for credentials
2. **Implement proper error handling** for API calls and database operations
3. **Use batch processing** for large datasets to optimize performance
4. **Add logging** to track ETL pipeline progress
5. **Validate data** before loading into database
6. **Create indexes** on frequently queried columns
7. **Schedule regular ETL runs** using cron or task schedulers
8. **Monitor API quota usage** to avoid service interruptions
