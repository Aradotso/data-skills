---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for art collection data
  - extract and transform Harvard museum artifacts
  - set up data engineering pipeline with Streamlit
  - query and visualize museum artifact data
  - implement SQL analytics for art collections
  - build end-to-end data pipeline with museum API
  - create interactive art data visualization
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is an end-to-end data engineering and analytics application that demonstrates how to build ETL pipelines using the Harvard Art Museums API. It extracts artifact data, transforms it into relational structures, loads it into SQL databases, and provides interactive analytics dashboards using Streamlit.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages (typical setup)
pip install streamlit pandas requests mysql-connector-python plotly sqlalchemy
```

## Configuration

### API Key Setup

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

Configure using environment variables:

```bash
export HARVARD_API_KEY="your_api_key_here"
```

### Database Configuration

Set up your MySQL/TiDB Cloud credentials:

```bash
export DB_HOST="your_db_host"
export DB_PORT="3306"
export DB_USER="your_db_user"
export DB_PASSWORD="your_db_password"
export DB_NAME="harvard_artifacts"
```

## Project Structure

```
project/
├── app.py                 # Main Streamlit application
├── etl_pipeline.py        # ETL logic
├── database.py            # Database connections and queries
├── requirements.txt       # Python dependencies
└── config.py             # Configuration management
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    
    return response.json()

def extract_all_artifacts(api_key, max_records=1000):
    """Extract artifacts with pagination handling"""
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        data = fetch_artifacts(api_key, page, size)
        records = data.get('records', [])
        
        if not records:
            break
            
        all_artifacts.extend(records)
        page += 1
        
        # Rate limiting
        import time
        time.sleep(0.5)
    
    return all_artifacts[:max_records]
```

### 2. Data Transformation

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """Transform artifact data into metadata table"""
    metadata_records = []
    
    for artifact in artifacts:
        record = {
            'artifact_id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline')
        }
        metadata_records.append(record)
    
    return pd.DataFrame(metadata_records)

def transform_artifact_media(artifacts):
    """Transform nested media data"""
    media_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            record = {
                'artifact_id': artifact_id,
                'image_id': img.get('imageid'),
                'base_url': img.get('baseimageurl'),
                'format': img.get('format'),
                'width': img.get('width'),
                'height': img.get('height'),
                'copyright': img.get('copyright')
            }
            media_records.append(record)
    
    return pd.DataFrame(media_records)

def transform_artifact_colors(artifacts):
    """Transform color data"""
    color_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            record = {
                'artifact_id': artifact_id,
                'color_name': color.get('color'),
                'percentage': color.get('percent'),
                'hex_value': color.get('hex'),
                'css3': color.get('css3')
            }
            color_records.append(record)
    
    return pd.DataFrame(color_records)
```

### 3. Database Schema

```python
import mysql.connector
from mysql.connector import Error

def create_database_schema(connection):
    """Create database tables"""
    cursor = connection.cursor()
    
    # Artifact Metadata Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            artifact_id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            period VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            department VARCHAR(200),
            division VARCHAR(200),
            dated VARCHAR(200),
            medium TEXT,
            dimensions TEXT,
            creditline TEXT,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    
    # Artifact Media Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            image_id INT,
            base_url VARCHAR(500),
            format VARCHAR(50),
            width INT,
            height INT,
            copyright TEXT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    # Artifact Colors Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color_name VARCHAR(100),
            percentage FLOAT,
            hex_value VARCHAR(20),
            css3 VARCHAR(100),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    connection.commit()
    cursor.close()

def get_db_connection():
    """Create database connection"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=os.getenv('DB_PORT', 3306),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    return connection
```

### 4. Data Loading

```python
def load_data_to_db(df, table_name, connection):
    """Batch insert data into database"""
    cursor = connection.cursor()
    
    if df.empty:
        return
    
    # Prepare column names and placeholders
    columns = ', '.join(df.columns)
    placeholders = ', '.join(['%s'] * len(df.columns))
    
    insert_query = f"""
        INSERT INTO {table_name} ({columns})
        VALUES ({placeholders})
        ON DUPLICATE KEY UPDATE
        {', '.join([f'{col}=VALUES({col})' for col in df.columns if col != 'id'])}
    """
    
    # Convert DataFrame to list of tuples
    data = [tuple(x) for x in df.to_numpy()]
    
    # Batch insert
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()
    
    print(f"Loaded {len(df)} records into {table_name}")
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for configuration
    with st.sidebar:
        st.header("Configuration")
        
        api_key = st.text_input("Harvard API Key", type="password", 
                                value=os.getenv('HARVARD_API_KEY', ''))
        
        if st.button("Run ETL Pipeline"):
            with st.spinner("Extracting data from API..."):
                artifacts = extract_all_artifacts(api_key)
                st.success(f"Extracted {len(artifacts)} artifacts")
            
            with st.spinner("Transforming data..."):
                metadata_df = transform_artifact_metadata(artifacts)
                media_df = transform_artifact_media(artifacts)
                colors_df = transform_artifact_colors(artifacts)
                st.success("Data transformed successfully")
            
            with st.spinner("Loading to database..."):
                conn = get_db_connection()
                create_database_schema(conn)
                load_data_to_db(metadata_df, 'artifactmetadata', conn)
                load_data_to_db(media_df, 'artifactmedia', conn)
                load_data_to_db(colors_df, 'artifactcolors', conn)
                conn.close()
                st.success("Data loaded to database")
    
    # Analytics section
    st.header("📊 Analytics")
    
    query_options = {
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
            ORDER BY century
        """,
        "Top Colors Used": """
            SELECT color_name, ROUND(AVG(percentage), 2) as avg_percentage, COUNT(*) as frequency
            FROM artifactcolors
            GROUP BY color_name
            ORDER BY frequency DESC
            LIMIT 10
        """,
        "Department Distribution": """
            SELECT department, COUNT(*) as count
            FROM artifactmetadata
            GROUP BY department
            ORDER BY count DESC
        """
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
    
    if st.button("Run Query"):
        conn = get_db_connection()
        df = pd.read_sql(query_options[selected_query], conn)
        conn.close()
        
        st.dataframe(df)
        
        # Visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Analytics Queries

```sql
-- Artifacts with most images
SELECT 
    m.artifact_id,
    m.title,
    COUNT(a.image_id) as image_count
FROM artifactmetadata m
JOIN artifactmedia a ON m.artifact_id = a.artifact_id
GROUP BY m.artifact_id, m.title
ORDER BY image_count DESC
LIMIT 10;

-- Color distribution analysis
SELECT 
    color_name,
    COUNT(DISTINCT artifact_id) as artifact_count,
    ROUND(AVG(percentage), 2) as avg_percentage
FROM artifactcolors
GROUP BY color_name
HAVING artifact_count > 5
ORDER BY artifact_count DESC;

-- Artifacts by classification and period
SELECT 
    classification,
    period,
    COUNT(*) as count
FROM artifactmetadata
WHERE classification IS NOT NULL AND period IS NOT NULL
GROUP BY classification, period
ORDER BY count DESC;
```

## Running the Application

```bash
# Set environment variables
export HARVARD_API_KEY="your_api_key"
export DB_HOST="your_host"
export DB_USER="your_user"
export DB_PASSWORD="your_password"
export DB_NAME="harvard_artifacts"

# Run Streamlit app
streamlit run app.py
```

## Troubleshooting

**API Rate Limiting:**
```python
import time
from functools import wraps

def rate_limit(calls_per_second=2):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            time.sleep(1.0 / calls_per_second)
            return func(*args, **kwargs)
        return wrapper
    return decorator

@rate_limit(calls_per_second=2)
def fetch_artifacts(api_key, page):
    # Your fetch logic
    pass
```

**Database Connection Issues:**
```python
def get_db_connection_with_retry(max_retries=3):
    for attempt in range(max_retries):
        try:
            connection = mysql.connector.connect(
                host=os.getenv('DB_HOST'),
                user=os.getenv('DB_USER'),
                password=os.getenv('DB_PASSWORD'),
                database=os.getenv('DB_NAME')
            )
            return connection
        except Error as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)
```

**Handling Missing Data:**
```python
def safe_get(dictionary, key, default=None):
    """Safely extract nested values"""
    value = dictionary.get(key, default)
    return value if value else default

# Use in transformations
record = {
    'culture': safe_get(artifact, 'culture', 'Unknown'),
    'period': safe_get(artifact, 'period', 'Unknown')
}
```
