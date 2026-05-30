---
name: harvard-art-museums-etl-analytics
description: End-to-end ETL pipeline and analytics for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for museum artifact data
  - query the Harvard Art Museums API
  - create analytics dashboards for cultural heritage data
  - set up artifact metadata database with MySQL
  - visualize museum collection data with Streamlit
  - implement data engineering pipelines for art collections
  - analyze artifact color and media metadata
  - design SQL schemas for museum artifacts
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides a complete data engineering workflow for the Harvard Art Museums API: data extraction with pagination, transformation into relational tables, loading into MySQL/TiDB, analytical SQL queries, and interactive Streamlit dashboards with Plotly visualizations.

## What It Does

- **Extracts** artifact data from Harvard Art Museums API with rate limiting and pagination
- **Transforms** nested JSON into normalized relational tables (metadata, media, colors)
- **Loads** structured data into MySQL/TiDB Cloud databases
- **Analyzes** collections with 20+ predefined SQL analytics queries
- **Visualizes** results through interactive Streamlit dashboards

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

1. Get a free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Configure in Streamlit app or environment variable:

```python
# In .env file
HARVARD_API_KEY=your_api_key_here
```

### Database Configuration

Configure MySQL/TiDB connection:

```python
import mysql.connector
import os

db_config = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}

conn = mysql.connector.connect(**db_config)
```

## Database Schema

### Core Tables

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    primaryimageurl TEXT,
    url TEXT
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl TEXT,
    format VARCHAR(50),
    description TEXT,
    technique VARCHAR(200),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
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

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import pandas as pd
import time

def fetch_artifacts(api_key, size=100, max_pages=10):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        params = {
            'apikey': api_key,
            'size': size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            all_artifacts.extend(data.get('records', []))
            
            # Respect rate limits
            time.sleep(0.5)
        else:
            print(f"Error fetching page {page}: {response.status_code}")
            break
    
    return all_artifacts
```

### Transform: Normalize JSON to Tables

```python
def transform_artifacts(artifacts):
    """
    Transform nested JSON into normalized dataframes
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Metadata table
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'primaryimageurl': artifact.get('primaryimageurl'),
            'url': artifact.get('url')
        })
        
        # Media table
        for media in artifact.get('images', []):
            media_records.append({
                'artifact_id': artifact.get('id'),
                'baseimageurl': media.get('baseimageurl'),
                'format': media.get('format'),
                'description': media.get('description'),
                'technique': media.get('technique')
            })
        
        # Colors table
        for color in artifact.get('colors', []):
            color_records.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### Load: Batch Insert to Database

```python
def load_to_database(df_metadata, df_media, df_colors, db_config):
    """
    Load dataframes into MySQL database with batch inserts
    """
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    
    # Load metadata
    metadata_cols = ['id', 'title', 'culture', 'century', 'dated', 
                     'classification', 'department', 'division', 'technique',
                     'medium', 'dimensions', 'primaryimageurl', 'url']
    
    insert_metadata = f"""
        INSERT INTO artifactmetadata ({', '.join(metadata_cols)})
        VALUES ({', '.join(['%s'] * len(metadata_cols))})
        ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    cursor.executemany(insert_metadata, df_metadata[metadata_cols].values.tolist())
    
    # Load media
    if not df_media.empty:
        media_cols = ['artifact_id', 'baseimageurl', 'format', 'description', 'technique']
        insert_media = f"""
            INSERT INTO artifactmedia ({', '.join(media_cols)})
            VALUES ({', '.join(['%s'] * len(media_cols))})
        """
        cursor.executemany(insert_media, df_media[media_cols].values.tolist())
    
    # Load colors
    if not df_colors.empty:
        color_cols = ['artifact_id', 'color', 'spectrum', 'hue', 'percent']
        insert_colors = f"""
            INSERT INTO artifactcolors ({', '.join(color_cols)})
            VALUES ({', '.join(['%s'] * len(color_cols))})
        """
        cursor.executemany(insert_colors, df_colors[color_cols].values.tolist())
    
    conn.commit()
    cursor.close()
    conn.close()
```

## Analytical SQL Queries

### Sample Analytics Queries

```python
# Top 10 cultures by artifact count
query_cultures = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Artifacts by century
query_centuries = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY count DESC
"""

# Color distribution
query_colors = """
    SELECT spectrum, COUNT(*) as color_count
    FROM artifactcolors
    GROUP BY spectrum
    ORDER BY color_count DESC
"""

# Department-wise artifact classification
query_dept_class = """
    SELECT department, classification, COUNT(*) as count
    FROM artifactmetadata
    WHERE department IS NOT NULL AND classification IS NOT NULL
    GROUP BY department, classification
    ORDER BY department, count DESC
"""

# Media availability analysis
query_media = """
    SELECT 
        COUNT(DISTINCT am.id) as total_artifacts,
        COUNT(DISTINCT media.artifact_id) as artifacts_with_media,
        ROUND(COUNT(DISTINCT media.artifact_id) * 100.0 / COUNT(DISTINCT am.id), 2) as media_percentage
    FROM artifactmetadata am
    LEFT JOIN artifactmedia media ON am.id = media.artifact_id
"""
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("API Key", type="password")
    
    db_config = {
        'host': st.sidebar.text_input("DB Host", "localhost"),
        'user': st.sidebar.text_input("DB User", "root"),
        'password': st.sidebar.text_input("DB Password", type="password"),
        'database': st.sidebar.text_input("Database", "harvard_artifacts")
    }
    
    # ETL Section
    st.header("ETL Pipeline")
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching data..."):
            artifacts = fetch_artifacts(api_key, size=100, max_pages=5)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_meta, df_media, df_colors = transform_artifacts(artifacts)
            st.success("Data transformed")
        
        with st.spinner("Loading to database..."):
            load_to_database(df_meta, df_media, df_colors, db_config)
            st.success("Data loaded successfully")
    
    # Analytics Section
    st.header("Analytics Queries")
    
    query_options = {
        "Top Cultures": query_cultures,
        "Artifacts by Century": query_centuries,
        "Color Distribution": query_colors,
        "Department Classification": query_dept_class,
        "Media Availability": query_media
    }
    
    selected_query = st.selectbox("Select Query", list(query_options.keys()))
    
    if st.button("Execute Query"):
        conn = mysql.connector.connect(**db_config)
        df_result = pd.read_sql(query_options[selected_query], conn)
        conn.close()
        
        st.dataframe(df_result)
        
        # Auto-generate visualizations
        if len(df_result.columns) == 2:
            fig = px.bar(df_result, x=df_result.columns[0], y=df_result.columns[1])
            st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Rate Limiting for API Calls

```python
import time
from functools import wraps

def rate_limit(calls_per_second=2):
    """Decorator to rate limit API calls"""
    min_interval = 1.0 / calls_per_second
    last_called = [0.0]
    
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            wait_time = min_interval - elapsed
            if wait_time > 0:
                time.sleep(wait_time)
            result = func(*args, **kwargs)
            last_called[0] = time.time()
            return result
        return wrapper
    return decorator

@rate_limit(calls_per_second=2)
def api_call(url, params):
    return requests.get(url, params=params)
```

### Error Handling in ETL

```python
def safe_etl_pipeline(api_key, db_config):
    """ETL with comprehensive error handling"""
    try:
        # Extract
        artifacts = fetch_artifacts(api_key)
        if not artifacts:
            raise ValueError("No artifacts fetched")
        
        # Transform
        df_meta, df_media, df_colors = transform_artifacts(artifacts)
        
        # Validate
        assert not df_meta.empty, "Metadata is empty"
        
        # Load
        load_to_database(df_meta, df_media, df_colors, db_config)
        
        return True, f"Successfully processed {len(artifacts)} artifacts"
        
    except requests.RequestException as e:
        return False, f"API Error: {str(e)}"
    except mysql.connector.Error as e:
        return False, f"Database Error: {str(e)}"
    except Exception as e:
        return False, f"Unexpected Error: {str(e)}"
```

## Troubleshooting

### API Issues

**Rate Limit Exceeded:**
```python
# Increase sleep time between requests
time.sleep(1)  # Wait 1 second between calls
```

**Invalid API Key:**
- Verify key at [Harvard Art Museums API portal](https://www.harvardartmuseums.org/collections/api)
- Check environment variable is correctly set

### Database Connection

**Connection Timeout:**
```python
db_config = {
    'host': 'localhost',
    'user': 'root',
    'password': os.getenv('DB_PASSWORD'),
    'database': 'harvard_artifacts',
    'connection_timeout': 30  # Increase timeout
}
```

**Foreign Key Constraint Errors:**
- Ensure `artifactmetadata` is loaded before `artifactmedia` and `artifactcolors`
- Use `ON DUPLICATE KEY UPDATE` for re-runs

### Streamlit Performance

**Large Dataset Rendering:**
```python
# Use pagination for large results
@st.cache_data
def load_data(query, limit=1000):
    return pd.read_sql(f"{query} LIMIT {limit}", conn)
```

**Memory Issues:**
```python
# Process in chunks
chunk_size = 1000
for chunk in pd.read_sql(query, conn, chunksize=chunk_size):
    # Process each chunk
    pass
```
