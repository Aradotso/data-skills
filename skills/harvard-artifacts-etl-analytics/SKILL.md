---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I fetch data from Harvard Art Museums API
  - build an ETL pipeline for museum artifacts
  - create analytics dashboard with Streamlit and SQL
  - extract artifact data and load into database
  - visualize museum collection data with Plotly
  - set up Harvard artifacts data engineering pipeline
  - query and analyze art museum data with SQL
  - transform nested JSON museum data into relational tables
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

This project provides an end-to-end data engineering and analytics application for the Harvard Art Museums API. It demonstrates production-grade ETL pipelines, SQL database design, and interactive visualization using Streamlit. The architecture follows: **API → ETL → SQL → Analytics → Visualization**.

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages:**
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

1. Get your API key from [Harvard Art Museums API](https://harvardartmuseums.org/collections/api)
2. Create a `.env` file or use Streamlit secrets:

```python
# .env file
HARVARD_API_KEY=your_api_key_here
```

Or in Streamlit secrets (`.streamlit/secrets.toml`):
```toml
HARVARD_API_KEY = "your_api_key_here"
```

### Database Configuration

Set up MySQL/TiDB Cloud connection:

```python
import os
from dotenv import load_dotenv

load_dotenv()

DB_CONFIG = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}
```

## Running the Application

```bash
# Run the Streamlit app
streamlit run app.py
```

The app will be available at `http://localhost:8501`

## Core Components

### 1. API Data Extraction

**Fetch artifacts with pagination:**

```python
import requests
import os

def fetch_artifacts(api_key, size=100, page=1):
    """
    Fetch artifact data from Harvard Art Museums API
    
    Args:
        api_key: Harvard API key from environment
        size: Number of records per page (max 100)
        page: Page number for pagination
    
    Returns:
        dict: JSON response containing artifact records
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'size': size,
        'page': page,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    
    return response.json()

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, size=100, page=1)

print(f"Total records: {data['info']['totalrecords']}")
print(f"Records fetched: {len(data['records'])}")
```

**Handle rate limiting and pagination:**

```python
import time

def fetch_all_artifacts(api_key, max_records=1000):
    """Fetch multiple pages with rate limiting"""
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        try:
            data = fetch_artifacts(api_key, size=size, page=page)
            records = data['records']
            
            if not records:
                break
                
            all_artifacts.extend(records)
            print(f"Fetched {len(all_artifacts)} artifacts...")
            
            page += 1
            time.sleep(0.5)  # Rate limiting
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return all_artifacts[:max_records]
```

### 2. ETL Pipeline

**Extract and transform artifact data:**

```python
import pandas as pd

def transform_artifacts(raw_data):
    """
    Transform raw API data into normalized dataframes
    
    Returns:
        tuple: (metadata_df, media_df, colors_df)
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata_records.append({
            'artifact_id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'medium': artifact.get('medium'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url')
        })
        
        # Extract media/images
        if artifact.get('images'):
            for img in artifact['images']:
                media_records.append({
                    'artifact_id': artifact.get('id'),
                    'image_id': img.get('imageid'),
                    'base_url': img.get('baseimageurl'),
                    'width': img.get('width'),
                    'height': img.get('height'),
                    'format': img.get('format')
                })
        
        # Extract color data
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_records.append({
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'hue': color.get('hue'),
                    'percent': color.get('percent')
                })
    
    metadata_df = pd.DataFrame(metadata_records)
    media_df = pd.DataFrame(media_records)
    colors_df = pd.DataFrame(color_records)
    
    return metadata_df, media_df, colors_df
```

### 3. Database Schema and Loading

**Create database tables:**

```python
import mysql.connector

def create_tables(connection):
    """Create normalized tables for artifact data"""
    cursor = connection.cursor()
    
    # Artifact metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            artifact_id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            department VARCHAR(200),
            division VARCHAR(200),
            medium TEXT,
            dated VARCHAR(200),
            url VARCHAR(500)
        )
    """)
    
    # Artifact media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            image_id INT,
            base_url VARCHAR(500),
            width INT,
            height INT,
            format VARCHAR(50),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    # Artifact colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(100),
            spectrum VARCHAR(100),
            hue VARCHAR(100),
            percent FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    connection.commit()
    cursor.close()
```

**Load data into database:**

```python
def load_to_database(metadata_df, media_df, colors_df, db_config):
    """Batch insert dataframes into MySQL"""
    connection = mysql.connector.connect(**db_config)
    cursor = connection.cursor()
    
    # Load metadata
    for _, row in metadata_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmetadata 
            (artifact_id, title, culture, century, classification, 
             department, division, medium, dated, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """, tuple(row))
    
    # Load media
    if not media_df.empty:
        for _, row in media_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmedia 
                (artifact_id, image_id, base_url, width, height, format)
                VALUES (%s, %s, %s, %s, %s, %s)
            """, tuple(row))
    
    # Load colors
    if not colors_df.empty:
        for _, row in colors_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactcolors 
                (artifact_id, color, spectrum, hue, percent)
                VALUES (%s, %s, %s, %s, %s)
            """, tuple(row))
    
    connection.commit()
    cursor.close()
    connection.close()
    
    print(f"Loaded {len(metadata_df)} artifacts to database")
```

### 4. SQL Analytics Queries

**Common analytical queries:**

```python
# Query 1: Artifacts by culture
"""
SELECT culture, COUNT(*) as count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY count DESC
LIMIT 10
"""

# Query 2: Artifacts by century
"""
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC
"""

# Query 3: Department distribution
"""
SELECT department, COUNT(*) as artifact_count
FROM artifactmetadata
GROUP BY department
ORDER BY artifact_count DESC
"""

# Query 4: Media availability
"""
SELECT 
    CASE 
        WHEN image_id IS NOT NULL THEN 'Has Image'
        ELSE 'No Image'
    END as media_status,
    COUNT(*) as count
FROM artifactmetadata m
LEFT JOIN artifactmedia med ON m.artifact_id = med.artifact_id
GROUP BY media_status
"""

# Query 5: Color distribution
"""
SELECT color, COUNT(*) as frequency
FROM artifactcolors
GROUP BY color
ORDER BY frequency DESC
LIMIT 15
"""

# Query 6: Artifacts with multiple images
"""
SELECT m.artifact_id, m.title, COUNT(med.image_id) as image_count
FROM artifactmetadata m
JOIN artifactmedia med ON m.artifact_id = med.artifact_id
GROUP BY m.artifact_id, m.title
HAVING image_count > 1
ORDER BY image_count DESC
LIMIT 10
"""
```

**Execute queries in Streamlit:**

```python
import streamlit as st
import pandas as pd

def execute_query(query, db_config):
    """Execute SQL query and return results as DataFrame"""
    connection = mysql.connector.connect(**db_config)
    df = pd.read_sql(query, connection)
    connection.close()
    return df

# In Streamlit app
st.title("Harvard Artifacts Analytics")

query_options = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    "Century Distribution": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """
}

selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
query = query_options[selected_query]

if st.button("Run Query"):
    results = execute_query(query, DB_CONFIG)
    st.dataframe(results)
```

### 5. Visualization with Plotly

```python
import plotly.express as px

def create_bar_chart(df, x_col, y_col, title):
    """Create interactive bar chart from query results"""
    fig = px.bar(
        df,
        x=x_col,
        y=y_col,
        title=title,
        labels={x_col: x_col.replace('_', ' ').title(), 
                y_col: y_col.replace('_', ' ').title()},
        color=y_col,
        color_continuous_scale='viridis'
    )
    
    fig.update_layout(
        xaxis_tickangle=-45,
        height=500,
        showlegend=False
    )
    
    return fig

# In Streamlit app
if not results.empty:
    st.dataframe(results)
    
    # Auto-detect columns for visualization
    x_col = results.columns[0]
    y_col = results.columns[1]
    
    chart = create_bar_chart(results, x_col, y_col, selected_query)
    st.plotly_chart(chart, use_container_width=True)
```

## Common Patterns

### Complete ETL Pipeline

```python
def run_etl_pipeline(api_key, db_config, max_records=500):
    """Complete ETL process"""
    # Extract
    print("Extracting data from API...")
    raw_data = fetch_all_artifacts(api_key, max_records)
    
    # Transform
    print("Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts(raw_data)
    
    # Load
    print("Loading to database...")
    connection = mysql.connector.connect(**db_config)
    create_tables(connection)
    connection.close()
    
    load_to_database(metadata_df, media_df, colors_df, db_config)
    
    print("ETL pipeline completed successfully!")
    return metadata_df, media_df, colors_df
```

### Streamlit App Structure

```python
import streamlit as st
import os

# Page config
st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

# Sidebar configuration
with st.sidebar:
    st.header("Configuration")
    api_key = st.text_input("Harvard API Key", type="password", 
                             value=os.getenv('HARVARD_API_KEY', ''))
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Running ETL..."):
            run_etl_pipeline(api_key, DB_CONFIG, max_records=500)
            st.success("Pipeline completed!")

# Main content
st.title("🎨 Harvard Art Museums Analytics Dashboard")

tab1, tab2, tab3 = st.tabs(["Analytics", "Data Explorer", "Visualizations"])

with tab1:
    # Query selection and execution
    pass

with tab2:
    # Raw data exploration
    pass

with tab3:
    # Custom visualizations
    pass
```

## Troubleshooting

### API Issues

**Rate limiting errors:**
```python
# Add exponential backoff
import time
from requests.exceptions import RequestException

def fetch_with_retry(api_key, size, page, max_retries=3):
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, size, page)
        except RequestException as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt
                print(f"Retry {attempt + 1}/{max_retries} after {wait_time}s")
                time.sleep(wait_time)
            else:
                raise e
```

### Database Connection Issues

**Connection timeout:**
```python
DB_CONFIG = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'connect_timeout': 10,  # Add timeout
    'autocommit': True
}
```

### Data Quality Issues

**Handle missing values:**
```python
def clean_dataframe(df):
    """Clean and validate dataframe"""
    # Replace empty strings with None
    df = df.replace('', None)
    
    # Handle numeric columns
    numeric_cols = ['width', 'height', 'percent']
    for col in numeric_cols:
        if col in df.columns:
            df[col] = pd.to_numeric(df[col], errors='coerce')
    
    return df

metadata_df = clean_dataframe(metadata_df)
```

### Memory Issues with Large Datasets

**Process in chunks:**
```python
def load_large_dataset_in_chunks(df, table_name, db_config, chunk_size=1000):
    """Load large datasets in batches"""
    connection = mysql.connector.connect(**db_config)
    
    for i in range(0, len(df), chunk_size):
        chunk = df.iloc[i:i+chunk_size]
        chunk.to_sql(table_name, connection, if_exists='append', index=False)
        print(f"Loaded {i+chunk_size}/{len(df)} records")
    
    connection.close()
```

## Best Practices

1. **Always use environment variables** for API keys and database credentials
2. **Implement error handling** for API requests and database operations
3. **Add logging** to track ETL pipeline progress
4. **Use parameterized queries** to prevent SQL injection
5. **Cache API responses** during development to avoid rate limits
6. **Validate data** before loading to database
7. **Create indexes** on frequently queried columns for performance
