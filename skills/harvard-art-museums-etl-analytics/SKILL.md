---
name: harvard-art-museums-etl-analytics
description: ETL pipeline and analytics app for Harvard Art Museums API with Streamlit visualization and SQL database integration
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data engineering project with museum artifacts
  - fetch and analyze Harvard Art Museums API data
  - set up a Streamlit analytics dashboard for art collections
  - design SQL schema for museum artifact data
  - implement batch data loading from Harvard museums API
  - visualize art museum data with Plotly and Streamlit
  - create an end-to-end data pipeline for cultural artifacts
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering and analytics application for the Harvard Art Museums API. It demonstrates ETL pipeline implementation, SQL database design, analytical queries, and interactive visualization using Streamlit.

## What It Does

- **Data Collection**: Fetches artifact metadata, media, and color data from Harvard Art Museums API with pagination support
- **ETL Pipeline**: Transforms nested JSON API responses into normalized relational database tables
- **SQL Storage**: Stores data in MySQL/TiDB Cloud with proper foreign key relationships
- **Analytics**: Executes 20+ predefined analytical SQL queries for insights
- **Visualization**: Creates interactive dashboards with Plotly charts in Streamlit

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages:**
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### API Key Setup

Get your Harvard Art Museums API key from [https://www.harvardartmuseums.org/collections/api](https://www.harvardartmuseums.org/collections/api)

Create a `.env` file or configure in your app:

```python
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')
```

### Database Configuration

```python
import mysql.connector

db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME'),
    'port': int(os.getenv('DB_PORT', 3306))
}

conn = mysql.connector.connect(**db_config)
```

## Database Schema

### Create Tables

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    division VARCHAR(255),
    department VARCHAR(255),
    objectnumber VARCHAR(100),
    dated VARCHAR(255),
    description TEXT,
    period VARCHAR(255)
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(1000),
    imageid VARCHAR(100),
    renditionnumber VARCHAR(50),
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
import time

def fetch_artifacts(api_key, num_records=100, page_size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_records = []
    page = 1
    
    while len(all_records) < num_records:
        params = {
            'apikey': api_key,
            'size': min(page_size, num_records - len(all_records)),
            'page': page
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            all_records.extend(records)
            
            # Check if more pages available
            if not records or len(all_records) >= num_records:
                break
                
            page += 1
            time.sleep(0.5)  # Rate limiting
        else:
            print(f"Error: {response.status_code}")
            break
    
    return all_records[:num_records]
```

### Transform: Process JSON to DataFrames

```python
import pandas as pd

def transform_artifacts(raw_data):
    """
    Transform raw API data into normalized dataframes
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for record in raw_data:
        # Extract metadata
        metadata = {
            'id': record.get('id'),
            'title': record.get('title', '')[:500],
            'culture': record.get('culture', '')[:255],
            'century': record.get('century', '')[:100],
            'classification': record.get('classification', '')[:255],
            'division': record.get('division', '')[:255],
            'department': record.get('department', '')[:255],
            'objectnumber': record.get('objectnumber', '')[:100],
            'dated': record.get('dated', '')[:255],
            'description': record.get('description', ''),
            'period': record.get('period', '')[:255]
        }
        metadata_list.append(metadata)
        
        # Extract media
        images = record.get('images', [])
        for img in images:
            media = {
                'artifact_id': record.get('id'),
                'baseimageurl': img.get('baseimageurl', '')[:1000],
                'imageid': str(img.get('imageid', ''))[:100],
                'renditionnumber': img.get('renditionnumber', '')[:50]
            }
            media_list.append(media)
        
        # Extract colors
        colors = record.get('colors', [])
        for color in colors:
            color_data = {
                'artifact_id': record.get('id'),
                'color': color.get('color', '')[:50],
                'spectrum': color.get('spectrum', '')[:50],
                'hue': color.get('hue', '')[:50],
                'percent': color.get('percent', 0.0)
            }
            colors_list.append(color_data)
    
    df_metadata = pd.DataFrame(metadata_list)
    df_media = pd.DataFrame(media_list)
    df_colors = pd.DataFrame(colors_list)
    
    return df_metadata, df_media, df_colors
```

### Load: Insert into SQL Database

```python
def load_to_database(df_metadata, df_media, df_colors, db_config):
    """
    Batch insert dataframes into SQL database
    """
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    
    # Insert metadata
    for _, row in df_metadata.iterrows():
        sql = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, division, 
         department, objectnumber, dated, description, period)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.execute(sql, tuple(row))
    
    # Insert media
    for _, row in df_media.iterrows():
        sql = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, imageid, renditionnumber)
        VALUES (%s, %s, %s, %s)
        """
        cursor.execute(sql, tuple(row))
    
    # Insert colors
    for _, row in df_colors.iterrows():
        sql = """
        INSERT INTO artifactcolors 
        (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
        """
        cursor.execute(sql, tuple(row))
    
    conn.commit()
    cursor.close()
    conn.close()
```

## Streamlit Application

### Main App Structure

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Art Museums Analytics", layout="wide")

def main():
    st.title("🎨 Harvard Art Museums Data Analytics")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Navigation",
        ["Data Collection", "ETL Pipeline", "Analytics Dashboard", "Visualizations"]
    )
    
    if page == "Data Collection":
        data_collection_page()
    elif page == "ETL Pipeline":
        etl_pipeline_page()
    elif page == "Analytics Dashboard":
        analytics_page()
    elif page == "Visualizations":
        visualization_page()

if __name__ == "__main__":
    main()
```

### Data Collection Page

```python
def data_collection_page():
    st.header("📥 Data Collection")
    
    api_key = st.text_input("Harvard API Key", type="password", 
                            value=os.getenv('HARVARD_API_KEY', ''))
    num_records = st.number_input("Number of Records", min_value=10, 
                                  max_value=1000, value=100)
    
    if st.button("Fetch Data"):
        with st.spinner("Fetching data from API..."):
            raw_data = fetch_artifacts(api_key, num_records)
            st.success(f"Fetched {len(raw_data)} artifacts")
            
            # Preview data
            st.subheader("Sample Data")
            st.json(raw_data[0] if raw_data else {})
            
            # Store in session state
            st.session_state['raw_data'] = raw_data
```

### Analytics Queries

```python
ANALYTICAL_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture 
        ORDER BY count DESC 
        LIMIT 15
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century 
        ORDER BY count DESC
    """,
    
    "Media Availability": """
        SELECT 
            CASE WHEN m.artifact_id IS NOT NULL THEN 'Has Media' 
                 ELSE 'No Media' END as media_status,
            COUNT(*) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY media_status
    """,
    
    "Top Colors in Collection": """
        SELECT color, COUNT(*) as count, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Artifacts by Department": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL AND department != ''
        GROUP BY department
        ORDER BY count DESC
    """
}

def analytics_page():
    st.header("📊 SQL Analytics Dashboard")
    
    query_name = st.selectbox("Select Query", list(ANALYTICAL_QUERIES.keys()))
    
    if st.button("Execute Query"):
        conn = mysql.connector.connect(**db_config)
        df_result = pd.read_sql(ANALYTICAL_QUERIES[query_name], conn)
        conn.close()
        
        st.subheader("Query Results")
        st.dataframe(df_result)
        
        # Auto-generate visualization
        if len(df_result.columns) >= 2:
            fig = px.bar(df_result, x=df_result.columns[0], 
                        y=df_result.columns[1],
                        title=query_name)
            st.plotly_chart(fig, use_container_width=True)
```

## Common Patterns

### Rate Limiting API Calls

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
            if elapsed < min_interval:
                time.sleep(min_interval - elapsed)
            result = func(*args, **kwargs)
            last_called[0] = time.time()
            return result
        return wrapper
    return decorator

@rate_limit(calls_per_second=2)
def fetch_single_artifact(artifact_id, api_key):
    url = f"https://api.harvardartmuseums.org/object/{artifact_id}"
    response = requests.get(url, params={'apikey': api_key})
    return response.json()
```

### Error Handling in ETL

```python
def safe_transform(raw_data):
    """Transform with error handling"""
    successful = []
    failed = []
    
    for i, record in enumerate(raw_data):
        try:
            # Transform logic here
            transformed = transform_single_record(record)
            successful.append(transformed)
        except Exception as e:
            failed.append({'index': i, 'error': str(e), 'record_id': record.get('id')})
    
    return successful, failed
```

## Troubleshooting

**API Key Issues:**
- Ensure API key is valid and not expired
- Check rate limits (default is 2500 requests/day)
- Verify network connectivity

**Database Connection Errors:**
```python
try:
    conn = mysql.connector.connect(**db_config)
    st.success("Database connected successfully")
except mysql.connector.Error as err:
    st.error(f"Database error: {err}")
    st.info("Check DB_HOST, DB_USER, DB_PASSWORD in environment variables")
```

**Memory Issues with Large Datasets:**
```python
# Use chunking for large datasets
def load_in_chunks(df, chunk_size=1000):
    for start in range(0, len(df), chunk_size):
        chunk = df[start:start + chunk_size]
        # Load chunk to database
        load_to_database(chunk, db_config)
```

**Empty Results:**
- Check if tables are populated: `SELECT COUNT(*) FROM artifactmetadata`
- Verify foreign key relationships are correct
- Ensure data was committed to database after insertion
