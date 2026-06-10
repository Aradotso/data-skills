---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for museum artifact data
  - create a data engineering app with Harvard Art Museums API
  - set up analytics dashboard for art collection data
  - implement artifact metadata extraction and visualization
  - build a Streamlit app for museum data analytics
  - create SQL analytics queries for art museum collections
  - design ETL pipeline with pandas and MySQL
  - visualize museum artifact data with Plotly
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is an end-to-end data engineering application that demonstrates real-world ETL pipelines using the Harvard Art Museums API. It extracts artifact metadata, transforms nested JSON into relational structures, loads data into SQL databases, and provides interactive analytics through a Streamlit dashboard.

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

Obtain an API key from [Harvard Art Museums](https://www.harvardartmuseums.org/collections/api) and configure it:

```python
# Use environment variables (recommended)
import os
API_KEY = os.getenv('HARVARD_API_KEY')

# Or store in .env file
# HARVARD_API_KEY=your_api_key_here
# DB_HOST=your_database_host
# DB_USER=your_db_user
# DB_PASSWORD=your_db_password
# DB_NAME=your_db_name
```

### Database Setup

**MySQL/TiDB Schema:**
```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    dated VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    primaryimageurl TEXT,
    imagecount INT,
    totalpageviews INT,
    totaluniquepageviews INT
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    mediaid INT,
    baseimageurl TEXT,
    format VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
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
# Start the Streamlit app
streamlit run app.py
```

The app will be available at `http://localhost:8501`

## Core Components

### 1. API Data Collection

**Fetching Artifacts with Pagination:**
```python
import requests
import pandas as pd

def fetch_artifacts(api_key, num_records=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_records = []
    page = 1
    size = 100  # Max per page
    
    while len(all_records) < num_records:
        params = {
            'apikey': api_key,
            'page': page,
            'size': min(size, num_records - len(all_records))
        }
        
        response = requests.get(base_url, params=params)
        if response.status_code != 200:
            break
            
        data = response.json()
        records = data.get('records', [])
        if not records:
            break
            
        all_records.extend(records)
        page += 1
    
    return all_records[:num_records]
```

### 2. ETL Pipeline

**Extract and Transform:**
```python
def transform_artifacts(raw_data):
    """
    Transform nested JSON into normalized dataframes
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'dated': artifact.get('dated'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'primaryimageurl': artifact.get('primaryimageurl'),
            'imagecount': artifact.get('imagecount', 0),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        }
        metadata_list.append(metadata)
        
        # Extract media (images)
        for media in artifact.get('images', []):
            media_record = {
                'artifact_id': artifact.get('id'),
                'mediaid': media.get('imageid'),
                'baseimageurl': media.get('baseimageurl'),
                'format': media.get('format')
            }
            media_list.append(media_record)
        
        # Extract colors
        for color in artifact.get('colors', []):
            color_record = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            colors_list.append(color_record)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

**Load to Database:**
```python
import mysql.connector
from mysql.connector import Error

def load_to_database(metadata_df, media_df, colors_df, db_config):
    """
    Batch insert dataframes into SQL database
    """
    try:
        connection = mysql.connector.connect(**db_config)
        cursor = connection.cursor()
        
        # Insert metadata
        metadata_insert = """
        INSERT INTO artifactmetadata 
        (id, title, culture, dated, century, classification, department, 
         division, primaryimageurl, imagecount, totalpageviews, totaluniquepageviews)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_insert, metadata_df.values.tolist())
        
        # Insert media
        media_insert = """
        INSERT INTO artifactmedia (artifact_id, mediaid, baseimageurl, format)
        VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(media_insert, media_df.values.tolist())
        
        # Insert colors
        colors_insert = """
        INSERT INTO artifactcolors (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
        """
        cursor.executemany(colors_insert, colors_df.values.tolist())
        
        connection.commit()
        return True
        
    except Error as e:
        print(f"Database error: {e}")
        return False
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

### 3. SQL Analytics Queries

**Sample Analytical Queries:**
```python
# Top 10 cultures by artifact count
query_culture = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10
"""

# Artifacts by century distribution
query_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC
"""

# Most viewed artifacts
query_pageviews = """
SELECT title, culture, totalpageviews
FROM artifactmetadata
WHERE totalpageviews > 0
ORDER BY totalpageviews DESC
LIMIT 20
"""

# Color distribution analysis
query_colors = """
SELECT c.color, c.spectrum, COUNT(*) as usage_count, AVG(c.percent) as avg_percent
FROM artifactcolors c
GROUP BY c.color, c.spectrum
ORDER BY usage_count DESC
LIMIT 15
"""

# Department-wise artifact and image count
query_department = """
SELECT department, 
       COUNT(*) as artifact_count,
       SUM(imagecount) as total_images
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY artifact_count DESC
"""
```

### 4. Streamlit Dashboard

**Main Application Structure:**
```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Choose a page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        st.header("📥 Collect Artifact Data")
        
        num_records = st.number_input("Number of records", 10, 1000, 100)
        
        if st.button("Fetch Data"):
            with st.spinner("Fetching artifacts..."):
                api_key = os.getenv('HARVARD_API_KEY')
                raw_data = fetch_artifacts(api_key, num_records)
                metadata_df, media_df, colors_df = transform_artifacts(raw_data)
                
                st.success(f"Collected {len(metadata_df)} artifacts")
                st.dataframe(metadata_df.head(10))
                
                # Load to database
                if st.button("Load to Database"):
                    db_config = {
                        'host': os.getenv('DB_HOST'),
                        'user': os.getenv('DB_USER'),
                        'password': os.getenv('DB_PASSWORD'),
                        'database': os.getenv('DB_NAME')
                    }
                    success = load_to_database(metadata_df, media_df, colors_df, db_config)
                    if success:
                        st.success("✅ Data loaded successfully!")
    
    elif page == "SQL Analytics":
        st.header("📊 Run SQL Queries")
        
        queries = {
            "Top Cultures": query_culture,
            "Century Distribution": query_century,
            "Most Viewed": query_pageviews,
            "Color Analysis": query_colors,
            "Department Stats": query_department
        }
        
        selected_query = st.selectbox("Select Analysis", list(queries.keys()))
        
        if st.button("Execute Query"):
            result_df = execute_query(queries[selected_query])
            st.dataframe(result_df)
            
            # Auto-generate visualization
            if len(result_df.columns) >= 2:
                fig = px.bar(result_df, x=result_df.columns[0], y=result_df.columns[1])
                st.plotly_chart(fig)

def execute_query(query):
    """Execute SQL query and return dataframe"""
    db_config = {
        'host': os.getenv('DB_HOST'),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    connection = mysql.connector.connect(**db_config)
    df = pd.read_sql(query, connection)
    connection.close()
    return df

if __name__ == "__main__":
    main()
```

## Common Patterns

### Rate Limiting API Calls
```python
import time

def fetch_with_rate_limit(api_key, delay=0.5):
    """Add delay between API calls to respect rate limits"""
    all_records = []
    for page in range(1, 11):  # 10 pages
        response = requests.get(
            "https://api.harvardartmuseums.org/object",
            params={'apikey': api_key, 'page': page}
        )
        all_records.extend(response.json().get('records', []))
        time.sleep(delay)  # Rate limiting
    return all_records
```

### Error Handling in ETL
```python
def safe_transform(raw_data):
    """Transform with error handling"""
    processed = []
    errors = []
    
    for idx, artifact in enumerate(raw_data):
        try:
            record = {
                'id': artifact['id'],
                'title': artifact.get('title', 'Unknown'),
                'culture': artifact.get('culture', 'Unknown')
            }
            processed.append(record)
        except Exception as e:
            errors.append((idx, str(e)))
    
    return pd.DataFrame(processed), errors
```

## Troubleshooting

**API Connection Issues:**
- Verify API key is valid and not expired
- Check rate limits (Harvard API allows ~2500 requests/day)
- Ensure network connectivity to api.harvardartmuseums.org

**Database Connection Errors:**
- Confirm database credentials in environment variables
- Check if database tables exist (run schema creation first)
- Verify firewall/network access to database host

**Memory Issues with Large Datasets:**
```python
# Process in chunks
def fetch_in_batches(api_key, total=1000, batch_size=100):
    for offset in range(0, total, batch_size):
        batch = fetch_artifacts(api_key, batch_size)
        metadata_df, media_df, colors_df = transform_artifacts(batch)
        load_to_database(metadata_df, media_df, colors_df, db_config)
```

**Streamlit Caching for Performance:**
```python
@st.cache_data
def load_cached_data(query):
    """Cache query results to avoid redundant DB calls"""
    return execute_query(query)
```
