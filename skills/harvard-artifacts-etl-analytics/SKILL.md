---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics for Harvard Art Museums API data with SQL storage and Streamlit visualization
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - set up Harvard artifacts data engineering pipeline
  - create analytics dashboard for museum artifact data
  - extract and analyze Harvard Art Museums API data
  - build SQL database from Harvard Art Museums API
  - visualize artifact collection data with Streamlit
  - process Harvard museum data with Python ETL
  - query and analyze art museum metadata
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

An end-to-end data engineering and analytics application for building ETL pipelines from the Harvard Art Museums API. This project demonstrates real-world patterns for API data extraction, SQL transformation/storage, and interactive visualization using Streamlit.

## What It Does

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color data into relational SQL tables
- **SQL Analytics**: Run predefined analytical queries on structured artifact data
- **Interactive Dashboards**: Visualize query results using Streamlit and Plotly
- **Database Design**: Proper relational schema with foreign keys (artifactmetadata, artifactmedia, artifactcolors)

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

### 1. Harvard Art Museums API Key

Get your free API key from https://harvardartmuseums.org/collections/api

```python
# Store in .env file
HARVARD_API_KEY=your_api_key_here
```

### 2. Database Connection

Configure MySQL/TiDB connection:

```python
# .env file
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### 3. Load Environment Variables

```python
import os
from dotenv import load_dotenv

load_dotenv()

API_KEY = os.getenv('HARVARD_API_KEY')
DB_CONFIG = {
    'host': os.getenv('DB_HOST'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}
```

## Database Schema

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    dated VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    medium VARCHAR(500),
    technique VARCHAR(500),
    accessionyear INT,
    primaryimageurl TEXT
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    iiifbaseuri TEXT,
    baseimageurl TEXT,
    publiccaption TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
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

## Core ETL Pipeline

### Extract: Fetch Data from API

```python
import requests
import time

def fetch_artifacts(api_key, page=1, size=100, max_pages=10):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    
    for page_num in range(1, max_pages + 1):
        params = {
            'apikey': api_key,
            'page': page_num,
            'size': size,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            all_artifacts.extend(data.get('records', []))
            
            # Check if more pages exist
            if page_num >= data.get('info', {}).get('pages', 0):
                break
        else:
            print(f"Error fetching page {page_num}: {response.status_code}")
            break
        
        # Rate limiting
        time.sleep(0.5)
    
    return all_artifacts
```

### Transform: Process JSON to DataFrames

```python
import pandas as pd

def transform_artifacts(artifacts):
    """
    Transform nested JSON into relational DataFrames
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Metadata extraction
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'dated': artifact.get('dated'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'medium': artifact.get('medium'),
            'technique': artifact.get('technique'),
            'accessionyear': artifact.get('accessionyear'),
            'primaryimageurl': artifact.get('primaryimageurl')
        }
        metadata_records.append(metadata)
        
        # Media extraction
        for image in artifact.get('images', []):
            media = {
                'artifact_id': artifact.get('id'),
                'iiifbaseuri': image.get('iiifbaseuri'),
                'baseimageurl': image.get('baseimageurl'),
                'publiccaption': image.get('publiccaption')
            }
            media_records.append(media)
        
        # Color extraction
        for color in artifact.get('colors', []):
            color_record = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            color_records.append(color_record)
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### Load: Insert into SQL Database

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(metadata_df, media_df, colors_df, db_config):
    """
    Load DataFrames into MySQL database with batch inserts
    """
    try:
        connection = mysql.connector.connect(**db_config)
        cursor = connection.cursor()
        
        # Insert metadata (batch insert for performance)
        metadata_insert = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, dated, 
         department, division, medium, technique, accessionyear, primaryimageurl)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_insert, metadata_df.values.tolist())
        
        # Insert media
        media_insert = """
        INSERT INTO artifactmedia 
        (artifact_id, iiifbaseuri, baseimageurl, publiccaption)
        VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(media_insert, media_df.values.tolist())
        
        # Insert colors
        colors_insert = """
        INSERT INTO artifactcolors 
        (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
        """
        cursor.executemany(colors_insert, colors_df.values.tolist())
        
        connection.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

## Streamlit Application

### Main App Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Choose Operation",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        data_collection_page()
    elif page == "SQL Analytics":
        analytics_page()
    elif page == "Visualizations":
        visualization_page()

def data_collection_page():
    st.header("📥 Data Collection from API")
    
    api_key = st.text_input("Harvard API Key", type="password")
    num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=50, value=5)
    
    if st.button("Start ETL Pipeline"):
        with st.spinner("Fetching artifacts..."):
            artifacts = fetch_artifacts(api_key, max_pages=num_pages)
            st.success(f"Fetched {len(artifacts)} artifacts")
            
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            st.success("Data transformed")
            
        with st.spinner("Loading to database..."):
            load_to_database(metadata_df, media_df, colors_df, DB_CONFIG)
            st.success("ETL Complete!")

def analytics_page():
    st.header("📊 SQL Analytics")
    
    queries = {
        "Top 10 Cultures": "SELECT culture, COUNT(*) as count FROM artifactmetadata GROUP BY culture ORDER BY count DESC LIMIT 10",
        "Artifacts by Century": "SELECT century, COUNT(*) as count FROM artifactmetadata WHERE century IS NOT NULL GROUP BY century ORDER BY century",
        "Department Distribution": "SELECT department, COUNT(*) as count FROM artifactmetadata GROUP BY department ORDER BY count DESC",
        "Top Colors Used": "SELECT color, COUNT(*) as usage_count FROM artifactcolors GROUP BY color ORDER BY usage_count DESC LIMIT 15",
        "Media Availability": "SELECT COUNT(DISTINCT artifact_id) as artifacts_with_media FROM artifactmedia"
    }
    
    selected_query = st.selectbox("Select Query", list(queries.keys()))
    
    if st.button("Run Query"):
        connection = mysql.connector.connect(**DB_CONFIG)
        result_df = pd.read_sql(queries[selected_query], connection)
        connection.close()
        
        st.dataframe(result_df)
        
        # Auto-visualization
        if len(result_df.columns) == 2:
            fig = px.bar(result_df, x=result_df.columns[0], y=result_df.columns[1])
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Analytics Queries

```python
# Top cultures by artifact count
SELECT culture, COUNT(*) as artifact_count 
FROM artifactmetadata 
WHERE culture IS NOT NULL 
GROUP BY culture 
ORDER BY artifact_count DESC 
LIMIT 10;

# Artifacts with images vs without
SELECT 
    CASE WHEN primaryimageurl IS NOT NULL THEN 'With Image' ELSE 'No Image' END as has_image,
    COUNT(*) as count
FROM artifactmetadata
GROUP BY has_image;

# Color analysis - most dominant colors
SELECT color, AVG(percent) as avg_dominance, COUNT(*) as occurrences
FROM artifactcolors
GROUP BY color
ORDER BY avg_dominance DESC
LIMIT 10;

# Accession trends over time
SELECT accessionyear, COUNT(*) as acquisitions
FROM artifactmetadata
WHERE accessionyear BETWEEN 1900 AND 2024
GROUP BY accessionyear
ORDER BY accessionyear;
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Troubleshooting

**API Rate Limiting**
```python
# Add exponential backoff
import time
from requests.exceptions import RequestException

def fetch_with_retry(url, params, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params)
            if response.status_code == 429:  # Too Many Requests
                wait_time = 2 ** attempt
                time.sleep(wait_time)
                continue
            return response
        except RequestException as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)
```

**Database Connection Issues**
```python
# Test connection before ETL
def test_db_connection(db_config):
    try:
        connection = mysql.connector.connect(**db_config)
        if connection.is_connected():
            print("Database connection successful")
            connection.close()
            return True
    except Error as e:
        print(f"Connection failed: {e}")
        return False
```

**Memory Management for Large Datasets**
```python
# Process in chunks
def load_in_chunks(df, table_name, db_config, chunk_size=1000):
    connection = mysql.connector.connect(**db_config)
    for start in range(0, len(df), chunk_size):
        chunk = df.iloc[start:start + chunk_size]
        chunk.to_sql(table_name, connection, if_exists='append', index=False)
    connection.close()
```
