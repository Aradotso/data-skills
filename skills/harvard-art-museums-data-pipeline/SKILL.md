---
name: harvard-art-museums-data-pipeline
description: Build end-to-end data engineering pipelines with the Harvard Art Museums API using ETL, SQL analytics, and Streamlit visualization
triggers:
  - build a data pipeline for museum artifacts
  - create ETL pipeline with Harvard Art Museums API
  - set up analytics dashboard for art collections
  - implement SQL analytics with museum data
  - build streamlit app for artifact visualization
  - create data engineering pipeline for art museums
  - analyze Harvard art collection with Python
  - develop museum artifact analytics application
---

# Harvard Art Museums Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Harvard Artifacts Collection Data Engineering & Analytics App is an end-to-end data pipeline that demonstrates production-grade ETL processes, SQL analytics, and interactive visualization. It fetches artifact data from the Harvard Art Museums API, transforms it into relational structures, stores it in SQL databases (MySQL/TiDB), and provides analytical insights through a Streamlit dashboard.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required Dependencies:**
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### API Key Setup

1. Obtain a Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api
2. Create environment configuration:

```bash
# .env file
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Database Schema

The pipeline uses three main tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    department VARCHAR(200),
    division VARCHAR(200),
    classification VARCHAR(200),
    technique VARCHAR(500),
    dated VARCHAR(200),
    url TEXT
);

CREATE TABLE artifactmedia (
    media_id INT PRIMARY KEY,
    artifact_id INT,
    baseimageurl TEXT,
    format VARCHAR(50),
    height INT,
    width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Usage Patterns

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
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

# Fetch first 100 artifacts
data = fetch_artifacts(page=1, size=100)
artifacts = data['records']
total_pages = data['info']['pages']
```

### 2. ETL Pipeline Implementation

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

class ArtifactETL:
    def __init__(self, db_config):
        self.db_config = db_config
        self.connection = None
    
    def connect_db(self):
        """Establish database connection"""
        self.connection = mysql.connector.connect(
            host=self.db_config['host'],
            user=self.db_config['user'],
            password=self.db_config['password'],
            database=self.db_config['database']
        )
        return self.connection.cursor()
    
    def extract(self, num_pages=5):
        """Extract data from API"""
        all_artifacts = []
        
        for page in range(1, num_pages + 1):
            data = fetch_artifacts(page=page, size=100)
            all_artifacts.extend(data['records'])
        
        return all_artifacts
    
    def transform_metadata(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Transform artifact metadata"""
        metadata = []
        
        for artifact in artifacts:
            metadata.append({
                'id': artifact.get('id'),
                'title': artifact.get('title', '')[:500],
                'culture': artifact.get('culture', '')[:200],
                'century': artifact.get('century', '')[:100],
                'department': artifact.get('department', '')[:200],
                'division': artifact.get('division', '')[:200],
                'classification': artifact.get('classification', '')[:200],
                'technique': artifact.get('technique', '')[:500],
                'dated': artifact.get('dated', '')[:200],
                'url': artifact.get('url', '')
            })
        
        return pd.DataFrame(metadata)
    
    def transform_media(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Transform media information"""
        media = []
        
        for artifact in artifacts:
            if 'primaryimageurl' in artifact and artifact['primaryimageurl']:
                media.append({
                    'media_id': artifact['id'],
                    'artifact_id': artifact['id'],
                    'baseimageurl': artifact.get('primaryimageurl'),
                    'format': 'image',
                    'height': artifact.get('height'),
                    'width': artifact.get('width')
                })
        
        return pd.DataFrame(media)
    
    def transform_colors(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Transform color data"""
        colors = []
        
        for artifact in artifacts:
            if 'colors' in artifact and artifact['colors']:
                for color in artifact['colors']:
                    colors.append({
                        'artifact_id': artifact['id'],
                        'color': color.get('color'),
                        'spectrum': color.get('spectrum'),
                        'percentage': color.get('percent')
                    })
        
        return pd.DataFrame(colors)
    
    def load(self, df: pd.DataFrame, table_name: str):
        """Load data into SQL database"""
        cursor = self.connect_db()
        
        # Batch insert for performance
        columns = ', '.join(df.columns)
        placeholders = ', '.join(['%s'] * len(df.columns))
        
        insert_query = f"""
            INSERT INTO {table_name} ({columns})
            VALUES ({placeholders})
            ON DUPLICATE KEY UPDATE
            {', '.join([f"{col}=VALUES({col})" for col in df.columns if col not in ['id', 'media_id', 'color_id']])}
        """
        
        data_tuples = [tuple(row) for row in df.values]
        cursor.executemany(insert_query, data_tuples)
        self.connection.commit()
        cursor.close()

# Usage example
db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

etl = ArtifactETL(db_config)

# Run ETL pipeline
artifacts = etl.extract(num_pages=10)
metadata_df = etl.transform_metadata(artifacts)
media_df = etl.transform_media(artifacts)
colors_df = etl.transform_colors(artifacts)

etl.load(metadata_df, 'artifactmetadata')
etl.load(media_df, 'artifactmedia')
etl.load(colors_df, 'artifactcolors')
```

### 3. SQL Analytics Queries

```python
def execute_query(query: str, db_config: Dict) -> pd.DataFrame:
    """Execute SQL query and return results as DataFrame"""
    connection = mysql.connector.connect(**db_config)
    df = pd.read_sql(query, connection)
    connection.close()
    return df

# Example analytical queries
queries = {
    'artifacts_by_culture': """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    
    'artifacts_by_century': """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY century
    """,
    
    'color_distribution': """
        SELECT color, COUNT(*) as frequency, AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    'media_availability': """
        SELECT 
            COUNT(DISTINCT am.id) as total_artifacts,
            COUNT(DISTINCT media.artifact_id) as artifacts_with_media,
            ROUND(COUNT(DISTINCT media.artifact_id) * 100.0 / COUNT(DISTINCT am.id), 2) as media_percentage
        FROM artifactmetadata am
        LEFT JOIN artifactmedia media ON am.id = media.artifact_id
    """,
    
    'artifacts_by_department': """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """
}

# Execute a query
result_df = execute_query(queries['artifacts_by_culture'], db_config)
print(result_df)
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def create_dashboard():
    """Create interactive analytics dashboard"""
    st.title("Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Select Analysis")
    query_options = {
        "Artifacts by Culture": "artifacts_by_culture",
        "Artifacts by Century": "artifacts_by_century",
        "Color Distribution": "color_distribution",
        "Media Availability": "media_availability",
        "Department Analysis": "artifacts_by_department"
    }
    
    selected_query = st.sidebar.selectbox(
        "Choose an analysis:",
        options=list(query_options.keys())
    )
    
    # Execute selected query
    query_key = query_options[selected_query]
    df = execute_query(queries[query_key], db_config)
    
    # Display results
    st.subheader(f"Results: {selected_query}")
    st.dataframe(df)
    
    # Visualizations
    if query_key in ['artifacts_by_culture', 'artifacts_by_century', 'artifacts_by_department']:
        fig = px.bar(
            df,
            x=df.columns[0],
            y='count',
            title=selected_query,
            labels={'count': 'Number of Artifacts'}
        )
        st.plotly_chart(fig, use_container_width=True)
    
    elif query_key == 'color_distribution':
        fig = px.bar(
            df,
            x='color',
            y='frequency',
            title='Most Common Colors in Artifacts',
            color='avg_percentage',
            labels={'frequency': 'Count', 'avg_percentage': 'Avg %'}
        )
        st.plotly_chart(fig, use_container_width=True)
    
    elif query_key == 'media_availability':
        fig = px.pie(
            values=[df['artifacts_with_media'][0], 
                   df['total_artifacts'][0] - df['artifacts_with_media'][0]],
            names=['With Media', 'Without Media'],
            title='Media Availability'
        )
        st.plotly_chart(fig, use_container_width=True)

# Run the app
if __name__ == "__main__":
    create_dashboard()
```

### 5. Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Advanced Patterns

### Rate Limiting and Batch Processing

```python
import time

def batch_fetch_artifacts(total_pages: int, batch_size: int = 10, delay: float = 1.0):
    """Fetch artifacts in batches with rate limiting"""
    all_artifacts = []
    
    for batch_start in range(1, total_pages + 1, batch_size):
        batch_end = min(batch_start + batch_size, total_pages + 1)
        
        for page in range(batch_start, batch_end):
            try:
                data = fetch_artifacts(page=page)
                all_artifacts.extend(data['records'])
                time.sleep(delay)  # Rate limiting
            except Exception as e:
                print(f"Error fetching page {page}: {e}")
                continue
        
        print(f"Completed batch: pages {batch_start}-{batch_end-1}")
    
    return all_artifacts
```

### Data Quality Checks

```python
def validate_data(df: pd.DataFrame, table_name: str) -> bool:
    """Validate data before loading"""
    checks = {
        'no_nulls_in_id': df['id'].notna().all() if 'id' in df.columns else True,
        'no_duplicates': df.duplicated().sum() == 0,
        'valid_row_count': len(df) > 0
    }
    
    for check, result in checks.items():
        if not result:
            print(f"Validation failed for {table_name}: {check}")
            return False
    
    return True
```

## Troubleshooting

**API Rate Limiting:**
```python
# Add exponential backoff
import time
from functools import wraps

def retry_with_backoff(max_retries=3, backoff_factor=2):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_retries - 1:
                        raise
                    wait_time = backoff_factor ** attempt
                    time.sleep(wait_time)
        return wrapper
    return decorator

@retry_with_backoff()
def fetch_artifacts_safe(page=1, size=100):
    return fetch_artifacts(page, size)
```

**Database Connection Issues:**
```python
# Use connection pooling
from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    **db_config
)

def get_connection():
    return db_pool.get_connection()
```

**Memory Management for Large Datasets:**
```python
# Process data in chunks
def load_in_chunks(df: pd.DataFrame, table_name: str, chunk_size: int = 1000):
    """Load large DataFrame in chunks"""
    for i in range(0, len(df), chunk_size):
        chunk = df.iloc[i:i + chunk_size]
        etl.load(chunk, table_name)
        print(f"Loaded chunk {i//chunk_size + 1}")
```

This skill enables AI coding agents to build production-ready data pipelines using the Harvard Art Museums API, implementing ETL best practices, SQL analytics, and interactive visualizations.
