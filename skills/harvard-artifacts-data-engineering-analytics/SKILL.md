---
name: harvard-artifacts-data-engineering-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Streamlit, MySQL, and Plotly
triggers:
  - how do I build a data pipeline with Harvard Art Museums API
  - create ETL pipeline for museum artifacts data
  - visualize Harvard museum data with Streamlit
  - query Harvard artifacts database with SQL
  - build analytics dashboard for art museum data
  - extract and load Harvard API data to MySQL
  - analyze museum collection data with Python
  - create data engineering project with art museum API
---

# Harvard Artifacts Data Engineering Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project demonstrates end-to-end data engineering workflows using the Harvard Art Museums API. It includes:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color data
- **SQL Database**: Store structured data in MySQL/TiDB with relational schema
- **Analytics**: Run SQL queries for insights on artifacts, cultures, centuries, and media
- **Visualization**: Interactive Streamlit dashboards with Plotly charts

The architecture follows: `API → ETL → SQL → Analytics → Visualization`

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Requirements typically include:**
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```bash
# Harvard Art Museums API Key
HARVARD_API_KEY=your_api_key_here

# MySQL/TiDB Connection
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Register at [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Request an API key
3. Add to `.env` file

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
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    medium VARCHAR(500),
    technique VARCHAR(500),
    period VARCHAR(255),
    creditline TEXT,
    url VARCHAR(500)
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(1000),
    image_width INT,
    image_height INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# App will open at http://localhost:8501
```

## Key Components

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
    response.raise_for_status()
    
    data = response.json()
    return data['records'], data['info']

def fetch_all_artifacts(max_records=1000):
    """Fetch multiple pages of artifacts"""
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        records, info = fetch_artifacts(page=page, size=size)
        all_artifacts.extend(records)
        
        if page >= info['pages']:
            break
        
        page += 1
    
    return all_artifacts[:max_records]
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

def extract_metadata(artifacts):
    """Extract artifact metadata"""
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Untitled'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'medium': artifact.get('medium'),
            'technique': artifact.get('technique'),
            'period': artifact.get('period'),
            'creditline': artifact.get('creditline'),
            'url': artifact.get('url')
        })
    
    return pd.DataFrame(metadata)

def extract_media(artifacts):
    """Extract media/image data"""
    media = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            media.append({
                'artifact_id': artifact_id,
                'image_url': img.get('baseimageurl'),
                'image_width': img.get('width'),
                'image_height': img.get('height')
            })
    
    return pd.DataFrame(media)

def extract_colors(artifacts):
    """Extract color data"""
    colors = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        color_data = artifact.get('colors', [])
        
        for color in color_data:
            colors.append({
                'artifact_id': artifact_id,
                'color_hex': color.get('hex'),
                'color_percent': color.get('percent')
            })
    
    return pd.DataFrame(colors)

def load_to_database(df, table_name, db_config):
    """Load DataFrame to MySQL database"""
    try:
        connection = mysql.connector.connect(
            host=db_config['host'],
            port=db_config['port'],
            user=db_config['user'],
            password=db_config['password'],
            database=db_config['database']
        )
        
        cursor = connection.cursor()
        
        # Prepare batch insert
        cols = ",".join([str(i) for i in df.columns.tolist()])
        placeholders = ",".join(["%s"] * len(df.columns))
        
        sql = f"INSERT INTO {table_name} ({cols}) VALUES ({placeholders})"
        
        # Execute batch insert
        data = [tuple(x) for x in df.to_numpy()]
        cursor.executemany(sql, data)
        
        connection.commit()
        print(f"Loaded {cursor.rowcount} records into {table_name}")
        
    except Error as e:
        print(f"Error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

### 3. Complete ETL Execution

```python
def run_etl_pipeline(max_records=500):
    """Execute complete ETL pipeline"""
    # Database configuration
    db_config = {
        'host': os.getenv('DB_HOST'),
        'port': int(os.getenv('DB_PORT', 3306)),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    
    # Extract
    print("Fetching artifacts from API...")
    artifacts = fetch_all_artifacts(max_records=max_records)
    
    # Transform
    print("Transforming data...")
    metadata_df = extract_metadata(artifacts)
    media_df = extract_media(artifacts)
    colors_df = extract_colors(artifacts)
    
    # Load
    print("Loading to database...")
    load_to_database(metadata_df, 'artifactmetadata', db_config)
    load_to_database(media_df, 'artifactmedia', db_config)
    load_to_database(colors_df, 'artifactcolors', db_config)
    
    print("ETL pipeline completed!")
```

### 4. SQL Analytics Queries

```python
def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def execute_query(query):
    """Execute SQL query and return results as DataFrame"""
    connection = get_db_connection()
    df = pd.read_sql(query, connection)
    connection.close()
    return df

# Sample analytics queries
ANALYTICS_QUERIES = {
    "Top 10 Cultures by Artifact Count": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
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
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "Artifacts with Most Images": """
        SELECT a.title, a.culture, COUNT(m.media_id) as image_count
        FROM artifactmetadata a
        JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY a.id, a.title, a.culture
        ORDER BY image_count DESC
        LIMIT 10
    """,
    
    "Most Common Colors": """
        SELECT color_hex, COUNT(*) as usage_count,
               AVG(color_percent) as avg_percent
        FROM artifactcolors
        GROUP BY color_hex
        ORDER BY usage_count DESC
        LIMIT 10
    """
}
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Data Analytics")
    st.markdown("Interactive analytics dashboard for museum artifacts")
    
    # Sidebar
    st.sidebar.header("🔧 Configuration")
    query_name = st.sidebar.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    # Main content
    if st.button("Run Analysis"):
        with st.spinner("Executing query..."):
            query = ANALYTICS_QUERIES[query_name]
            df = execute_query(query)
            
            st.subheader(f"📊 {query_name}")
            
            # Display data table
            st.dataframe(df, use_container_width=True)
            
            # Visualization
            if len(df.columns) >= 2:
                fig = px.bar(
                    df,
                    x=df.columns[0],
                    y=df.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig, use_container_width=True)
    
    # ETL Controls
    st.sidebar.header("🔄 ETL Pipeline")
    max_records = st.sidebar.number_input("Max Records", 100, 10000, 500)
    
    if st.sidebar.button("Run ETL"):
        with st.spinner("Running ETL pipeline..."):
            run_etl_pipeline(max_records=max_records)
            st.success("ETL completed successfully!")

if __name__ == "__main__":
    main()
```

## Common Patterns

### Incremental Data Loading

```python
def get_max_artifact_id():
    """Get the highest artifact ID in database"""
    query = "SELECT MAX(id) as max_id FROM artifactmetadata"
    connection = get_db_connection()
    cursor = connection.cursor()
    cursor.execute(query)
    result = cursor.fetchone()
    connection.close()
    return result[0] if result[0] else 0

def incremental_etl():
    """Load only new artifacts"""
    max_id = get_max_artifact_id()
    
    # Fetch artifacts with ID > max_id
    # Implementation depends on API filtering capabilities
    pass
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_fetch_artifacts(page, size, retries=3):
    """Fetch with retry logic"""
    for attempt in range(retries):
        try:
            return fetch_artifacts(page, size)
        except requests.exceptions.RequestException as e:
            logger.warning(f"Attempt {attempt + 1} failed: {e}")
            if attempt == retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

## Troubleshooting

### API Rate Limits
```python
import time

def fetch_with_rate_limit(page, size, delay=1):
    """Add delay between requests"""
    result = fetch_artifacts(page, size)
    time.sleep(delay)
    return result
```

### Database Connection Issues
- Verify credentials in `.env` file
- Check database host accessibility
- Ensure database and tables exist
- Verify firewall rules for database port

### Missing Data Handling
```python
def safe_get(dictionary, key, default=''):
    """Safely extract nested values"""
    return dictionary.get(key, default) or default
```

### Memory Management for Large Datasets
```python
def batch_etl(total_records=10000, batch_size=500):
    """Process data in batches"""
    for offset in range(0, total_records, batch_size):
        artifacts = fetch_artifacts(page=offset//100 + 1, size=100)
        # Process and load batch
        pass
```
