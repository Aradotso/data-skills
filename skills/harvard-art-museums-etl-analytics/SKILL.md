---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - analyze Harvard Art Museums collection with SQL
  - create a museum data analytics dashboard
  - extract and transform art museum API data
  - build a Streamlit app for art collection analytics
  - set up a data pipeline for museum artifacts
  - query Harvard Art Museums database
  - visualize museum collection data with Plotly
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is an end-to-end data engineering application that:
- Extracts artifact data from the Harvard Art Museums API
- Transforms nested JSON into relational database structures
- Loads data into MySQL/TiDB Cloud
- Provides SQL analytics queries
- Visualizes results through an interactive Streamlit dashboard

**Architecture:** API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required dependencies:**
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

Create a `.env` file or configure through Streamlit secrets:

```python
# .env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
DB_PORT=4000
```

### Streamlit Secrets (`.streamlit/secrets.toml`)

```toml
HARVARD_API_KEY = "your_api_key_here"
DB_HOST = "gateway01.us-east-1.prod.aws.tidbcloud.com"
DB_USER = "your_user"
DB_PASSWORD = "your_password"
DB_NAME = "harvard_artifacts"
DB_PORT = 4000
```

### Database Setup

The application uses three main tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    dated VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    medium VARCHAR(500),
    department VARCHAR(255),
    division VARCHAR(255),
    technique VARCHAR(500),
    period VARCHAR(255),
    provenance TEXT,
    description TEXT,
    url VARCHAR(500)
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    imagepermissionlevel INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
streamlit run app.py
```

The Streamlit app will launch on `http://localhost:8501`

## Core Components

### 1. API Data Extraction

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API
    
    Args:
        api_key: Harvard API key
        page: Page number for pagination
        size: Number of records per page (max 100)
    
    Returns:
        JSON response with artifact data
    """
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

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=100)
total_records = data['info']['totalrecords']
artifacts = data['records']
```

### 2. ETL Pipeline

```python
import pandas as pd

def transform_artifact_metadata(records):
    """
    Transform artifact records into metadata DataFrame
    """
    metadata = []
    for record in records:
        metadata.append({
            'id': record.get('id'),
            'title': record.get('title'),
            'culture': record.get('culture'),
            'dated': record.get('dated'),
            'century': record.get('century'),
            'classification': record.get('classification'),
            'medium': record.get('medium'),
            'department': record.get('department'),
            'division': record.get('division'),
            'technique': record.get('technique'),
            'period': record.get('period'),
            'provenance': record.get('provenance'),
            'description': record.get('description'),
            'url': record.get('url')
        })
    return pd.DataFrame(metadata)

def transform_artifact_media(records):
    """
    Transform media information from artifacts
    """
    media = []
    for record in records:
        media.append({
            'artifact_id': record.get('id'),
            'baseimageurl': record.get('baseimageurl'),
            'primaryimageurl': record.get('primaryimageurl'),
            'imagepermissionlevel': record.get('imagepermissionlevel', 0)
        })
    return pd.DataFrame(media)

def transform_artifact_colors(records):
    """
    Transform color data from artifacts
    """
    colors = []
    for record in records:
        artifact_id = record.get('id')
        color_data = record.get('colors', [])
        for color in color_data:
            colors.append({
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    return pd.DataFrame(colors)
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """
    Create database connection
    """
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME'),
        port=int(os.getenv('DB_PORT', 3306))
    )

def load_metadata(df, connection):
    """
    Batch insert metadata into database
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, dated, century, classification, medium, 
     department, division, technique, period, provenance, description, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()

def load_media(df, connection):
    """
    Insert media records
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (artifact_id, baseimageurl, primaryimageurl, imagepermissionlevel)
    VALUES (%s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()

def load_colors(df, connection):
    """
    Insert color records
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors 
    (artifact_id, color, spectrum, hue, percent)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()
```

### 4. SQL Analytics Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 20
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Top Classifications": """
        SELECT classification, COUNT(*) as total
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY total DESC
        LIMIT 15
    """,
    
    "Color Distribution": """
        SELECT color, COUNT(*) as frequency
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 20
    """,
    
    "Artifacts with Images": """
        SELECT 
            CASE 
                WHEN primaryimageurl IS NOT NULL THEN 'Has Image'
                ELSE 'No Image'
            END as image_status,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY image_status
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
    """
}

def execute_query(query, connection):
    """
    Execute SQL query and return DataFrame
    """
    cursor = connection.cursor()
    cursor.execute(query)
    columns = [desc[0] for desc in cursor.description]
    results = cursor.fetchall()
    cursor.close()
    return pd.DataFrame(results, columns=columns)
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for API key
    api_key = st.sidebar.text_input("Harvard API Key", 
                                     value=os.getenv('HARVARD_API_KEY', ''),
                                     type="password")
    
    # ETL Section
    st.header("📥 Data Collection")
    
    num_pages = st.number_input("Number of pages to fetch", 
                                 min_value=1, max_value=50, value=5)
    
    if st.button("Fetch and Load Data"):
        with st.spinner("Fetching data from API..."):
            all_records = []
            for page in range(1, num_pages + 1):
                data = fetch_artifacts(api_key, page=page)
                all_records.extend(data['records'])
                st.progress(page / num_pages)
            
            st.success(f"Fetched {len(all_records)} artifacts")
        
        with st.spinner("Transforming and loading data..."):
            conn = get_db_connection()
            
            # Transform
            metadata_df = transform_artifact_metadata(all_records)
            media_df = transform_artifact_media(all_records)
            colors_df = transform_artifact_colors(all_records)
            
            # Load
            load_metadata(metadata_df, conn)
            load_media(media_df, conn)
            load_colors(colors_df, conn)
            
            conn.close()
            st.success("Data loaded successfully!")
    
    # Analytics Section
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        conn = get_db_connection()
        query = ANALYTICS_QUERIES[query_name]
        
        st.code(query, language='sql')
        
        results_df = execute_query(query, conn)
        conn.close()
        
        st.dataframe(results_df)
        
        # Auto-visualization
        if len(results_df.columns) == 2:
            fig = px.bar(results_df, 
                        x=results_df.columns[0], 
                        y=results_df.columns[1],
                        title=query_name)
            st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Pagination Handling

```python
def fetch_all_artifacts(api_key, max_pages=10):
    """
    Fetch multiple pages of artifacts
    """
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        try:
            data = fetch_artifacts(api_key, page=page, size=100)
            all_artifacts.extend(data['records'])
            
            # Check if we've reached the end
            if page >= data['info']['pages']:
                break
                
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return all_artifacts
```

### Error Handling

```python
def safe_db_operation(operation, *args, **kwargs):
    """
    Wrapper for database operations with error handling
    """
    conn = None
    try:
        conn = get_db_connection()
        result = operation(conn, *args, **kwargs)
        return result
    except Error as e:
        st.error(f"Database error: {e}")
        return None
    finally:
        if conn and conn.is_connected():
            conn.close()
```

### Data Validation

```python
def validate_artifact_data(record):
    """
    Validate artifact record before processing
    """
    required_fields = ['id', 'title']
    
    for field in required_fields:
        if field not in record or record[field] is None:
            return False
    
    return True

# Usage in ETL
valid_records = [r for r in records if validate_artifact_data(r)]
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_rate_limit(api_key, page, delay=1):
    """
    Fetch with delay to respect rate limits
    """
    time.sleep(delay)
    return fetch_artifacts(api_key, page)
```

### Database Connection Issues

```python
def test_db_connection():
    """
    Test database connectivity
    """
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        cursor.close()
        conn.close()
        return True
    except Error as e:
        print(f"Connection failed: {e}")
        return False
```

### Missing Data Handling

```python
def safe_get(record, key, default=''):
    """
    Safely extract values from API response
    """
    value = record.get(key, default)
    return value if value is not None else default
```

### Large Dataset Processing

```python
def batch_process(records, batch_size=1000):
    """
    Process records in batches to avoid memory issues
    """
    for i in range(0, len(records), batch_size):
        batch = records[i:i + batch_size]
        yield batch

# Usage
for batch in batch_process(all_records, batch_size=500):
    metadata_df = transform_artifact_metadata(batch)
    load_metadata(metadata_df, conn)
```

## Best Practices

1. **Always use environment variables** for API keys and database credentials
2. **Implement pagination** when fetching large datasets
3. **Use batch inserts** for better database performance
4. **Validate data** before transformation and loading
5. **Handle API rate limits** with appropriate delays
6. **Close database connections** properly using try/finally blocks
7. **Log ETL operations** for debugging and monitoring
