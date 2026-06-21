---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with MySQL and Streamlit
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - set up data engineering project with museum artifacts
  - create analytics dashboard for Harvard Art Museums
  - extract and transform Harvard museum data
  - build Streamlit app for artifact analytics
  - connect Harvard API to MySQL database
  - analyze art museum collection data
  - visualize Harvard artifacts with Plotly
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

This project provides an end-to-end data engineering and analytics application using the Harvard Art Museums API. It demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualization with Streamlit.

## What It Does

- **Extracts** artifact data from Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into normalized relational tables (metadata, media, colors)
- **Loads** data into MySQL/TiDB Cloud with batch inserts
- **Analyzes** data using 20+ predefined SQL queries
- **Visualizes** results with interactive Plotly charts in a Streamlit dashboard

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

### Environment Variables

Create a `.env` file or set environment variables:

```bash
HARVARD_API_KEY=your_api_key_here
MYSQL_HOST=your_mysql_host
MYSQL_PORT=3306
MYSQL_USER=your_username
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=harvard_artifacts
```

### Get Harvard API Key

1. Visit https://docs.google.com/forms/d/e/1FAIpQLSfkmEBqH76HLMMiCC-GPPnhcvHC9aJS86E32dOd0Z8MpY2rvQ/viewform
2. Register for an API key
3. Store in environment variable `HARVARD_API_KEY`

### Database Setup

```sql
CREATE DATABASE harvard_artifacts;

USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    url VARCHAR(500)
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    iiifbaseuri VARCHAR(500),
    baseimageurl VARCHAR(500),
    format VARCHAR(50),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## Key Components

### 1. ETL Pipeline

**Extract from API:**

```python
import requests
import os

def fetch_artifacts(api_key, num_records=100, page=1):
    """Fetch artifacts from Harvard Art Museums API"""
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'size': min(num_records, 100),  # API limit per request
        'page': page,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    return response.json()

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, num_records=50)
artifacts = data.get('records', [])
```

**Transform to DataFrames:**

```python
import pandas as pd

def transform_artifacts(artifacts):
    """Transform nested JSON to normalized dataframes"""
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        
        # Metadata
        metadata_list.append({
            'objectid': objectid,
            'title': artifact.get('title', ''),
            'culture': artifact.get('culture', ''),
            'period': artifact.get('period', ''),
            'century': artifact.get('century', ''),
            'classification': artifact.get('classification', ''),
            'department': artifact.get('department', ''),
            'dated': artifact.get('dated', ''),
            'url': artifact.get('url', '')
        })
        
        # Media
        primary_image = artifact.get('primaryimageurl')
        if primary_image:
            media_list.append({
                'objectid': objectid,
                'iiifbaseuri': artifact.get('images', [{}])[0].get('iiifbaseuri', '') if artifact.get('images') else '',
                'baseimageurl': artifact.get('images', [{}])[0].get('baseimageurl', '') if artifact.get('images') else '',
                'format': artifact.get('images', [{}])[0].get('format', '') if artifact.get('images') else ''
            })
        
        # Colors
        for color_info in artifact.get('colors', []):
            colors_list.append({
                'objectid': objectid,
                'color': color_info.get('color', ''),
                'spectrum': color_info.get('spectrum', ''),
                'hue': color_info.get('hue', ''),
                'percent': color_info.get('percent', 0.0)
            })
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

**Load to MySQL:**

```python
import mysql.connector
from mysql.connector import Error

def load_to_mysql(metadata_df, media_df, colors_df):
    """Batch insert dataframes into MySQL"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('MYSQL_HOST'),
            port=int(os.getenv('MYSQL_PORT', 3306)),
            user=os.getenv('MYSQL_USER'),
            password=os.getenv('MYSQL_PASSWORD'),
            database=os.getenv('MYSQL_DATABASE')
        )
        
        cursor = connection.cursor()
        
        # Insert metadata
        metadata_sql = """
            INSERT IGNORE INTO artifactmetadata 
            (objectid, title, culture, period, century, classification, department, dated, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
        """
        cursor.executemany(metadata_sql, metadata_df.values.tolist())
        
        # Insert media
        media_sql = """
            INSERT INTO artifactmedia (objectid, iiifbaseuri, baseimageurl, format)
            VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(media_sql, media_df.values.tolist())
        
        # Insert colors
        colors_sql = """
            INSERT INTO artifactcolors (objectid, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """
        cursor.executemany(colors_sql, colors_df.values.tolist())
        
        connection.commit()
        print(f"Inserted {cursor.rowcount} records successfully")
        
    except Error as e:
        print(f"Error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

### 2. Analytics Queries

**Sample analytical queries:**

```python
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Top Color Distributions": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 10
    """,
    
    "Artifacts with Media": """
        SELECT 
            COUNT(DISTINCT m.objectid) as with_media,
            (SELECT COUNT(*) FROM artifactmetadata) as total,
            ROUND(COUNT(DISTINCT m.objectid) * 100.0 / (SELECT COUNT(*) FROM artifactmetadata), 2) as percentage
        FROM artifactmedia m
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL AND department != ''
        GROUP BY department
        ORDER BY count DESC
    """
}

def execute_query(query_name):
    """Execute analytical query and return DataFrame"""
    connection = mysql.connector.connect(
        host=os.getenv('MYSQL_HOST'),
        user=os.getenv('MYSQL_USER'),
        password=os.getenv('MYSQL_PASSWORD'),
        database=os.getenv('MYSQL_DATABASE')
    )
    
    df = pd.read_sql(ANALYTICS_QUERIES[query_name], connection)
    connection.close()
    return df
```

### 3. Streamlit Dashboard

**Main application:**

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Artifacts Analytics Dashboard")
    
    # Sidebar for ETL
    st.sidebar.header("ETL Pipeline")
    
    api_key = st.sidebar.text_input("Harvard API Key", type="password", value=os.getenv('HARVARD_API_KEY', ''))
    num_records = st.sidebar.slider("Number of Records", 10, 500, 100)
    
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Extracting data from API..."):
            data = fetch_artifacts(api_key, num_records)
            artifacts = data.get('records', [])
            
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            
        with st.spinner("Loading to MySQL..."):
            load_to_mysql(metadata_df, media_df, colors_df)
            
        st.success(f"✅ ETL Complete! Processed {len(artifacts)} artifacts")
    
    # Analytics section
    st.header("📊 Analytics")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        with st.spinner("Executing query..."):
            result_df = execute_query(query_name)
            
        st.dataframe(result_df)
        
        # Auto-generate visualization
        if len(result_df.columns) >= 2:
            fig = px.bar(
                result_df,
                x=result_df.columns[0],
                y=result_df.columns[1],
                title=query_name
            )
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

**Run the app:**

```bash
streamlit run app.py
```

## Common Patterns

### Pagination for Large Datasets

```python
def fetch_all_artifacts(api_key, total_records=1000):
    """Fetch artifacts with pagination"""
    all_artifacts = []
    page = 1
    per_page = 100
    
    while len(all_artifacts) < total_records:
        data = fetch_artifacts(api_key, num_records=per_page, page=page)
        artifacts = data.get('records', [])
        
        if not artifacts:
            break
            
        all_artifacts.extend(artifacts)
        page += 1
        
        # Rate limiting
        import time
        time.sleep(0.5)
    
    return all_artifacts[:total_records]
```

### Incremental Data Loading

```python
def get_latest_objectid():
    """Get the highest objectid already in database"""
    connection = mysql.connector.connect(...)
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(objectid) FROM artifactmetadata")
    result = cursor.fetchone()[0]
    connection.close()
    return result or 0

def incremental_load(api_key):
    """Only load new artifacts"""
    latest_id = get_latest_objectid()
    # Fetch artifacts with objectid > latest_id
    # Transform and load
```

## Troubleshooting

**API Rate Limiting:**
- Add `time.sleep(0.5)` between requests
- Use `hasimage=1` parameter to reduce dataset size

**Database Connection Issues:**
- Verify environment variables are set correctly
- Check MySQL service is running: `mysql -u root -p`
- Ensure database exists: `CREATE DATABASE IF NOT EXISTS harvard_artifacts`

**Missing Data in Tables:**
- Some artifacts lack certain fields (culture, period, colors)
- Use `IS NOT NULL` and `!= ''` filters in queries
- Handle None values during transformation

**Streamlit Port Conflicts:**
```bash
streamlit run app.py --server.port 8502
```

**Large Dataset Performance:**
- Use batch inserts with `executemany()`
- Add database indexes on frequently queried columns
- Limit initial ETL to 100-500 records for testing
