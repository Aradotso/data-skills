---
name: harvard-art-museums-data-pipeline
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - create a data engineering project with museum artifacts
  - set up Harvard Art Museums data analytics dashboard
  - extract and visualize art museum collection data
  - build Streamlit app for museum API data
  - create SQL analytics for Harvard artifacts
  - develop end-to-end data pipeline for art collections
  - analyze Harvard museum data with Python
---

# Harvard Art Museums Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipeline creation, SQL database design, analytical queries, and interactive visualization using Streamlit.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides a complete data pipeline that:
- Fetches artifact data from Harvard Art Museums API with pagination handling
- Transforms nested JSON responses into normalized relational data
- Loads data into SQL databases (MySQL/TiDB Cloud)
- Executes analytical SQL queries on the collected data
- Visualizes insights through interactive Streamlit dashboards with Plotly charts

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

Required packages:
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### 1. API Key Setup

Obtain an API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

Create a `.env` file:
```env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### 2. Database Setup

Create the database schema:

```sql
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    dated VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    classification VARCHAR(255),
    medium VARCHAR(500),
    technique VARCHAR(500),
    period VARCHAR(255),
    dated_begin INT,
    dated_end INT,
    accession_year INT
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(1000),
    media_type VARCHAR(50),
    caption TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(7),
    color_name VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Components

### ETL Pipeline Implementation

#### Extract: Fetch Data from API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, num_records=100, page=1):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'size': min(num_records, 100),  # API limit is 100 per request
        'page': page,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, num_records=50)
artifacts = data['records']
```

#### Transform: Process Nested JSON

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """
    Transform artifact records into structured metadata
    """
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'classification': artifact.get('classification'),
            'medium': artifact.get('medium'),
            'technique': artifact.get('technique'),
            'period': artifact.get('period'),
            'dated_begin': artifact.get('datebegin'),
            'dated_end': artifact.get('dateend'),
            'accession_year': artifact.get('accessionyear')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(artifacts):
    """
    Extract media/image data from artifacts
    """
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            media = {
                'artifact_id': artifact_id,
                'image_url': img.get('baseimageurl'),
                'media_type': 'image',
                'caption': img.get('caption')
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(artifacts):
    """
    Extract color data from artifacts
    """
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color_hex': color.get('hex'),
                'color_name': color.get('color'),
                'percentage': color.get('percent')
            }
            color_list.append(color_data)
    
    return pd.DataFrame(color_list)
```

#### Load: Insert into SQL Database

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
        database=os.getenv('DB_NAME')
    )

def load_metadata(df_metadata):
    """
    Load artifact metadata into database
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, dated, department, division, 
     classification, medium, technique, period, dated_begin, dated_end, accession_year)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    data_tuples = [tuple(row) for row in df_metadata.values]
    cursor.executemany(insert_query, data_tuples)
    
    conn.commit()
    cursor.close()
    conn.close()

def load_media(df_media):
    """
    Load artifact media data into database
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (artifact_id, image_url, media_type, caption)
    VALUES (%s, %s, %s, %s)
    """
    
    data_tuples = [tuple(row) for row in df_media.values]
    cursor.executemany(insert_query, data_tuples)
    
    conn.commit()
    cursor.close()
    conn.close()

def load_colors(df_colors):
    """
    Load artifact color data into database
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors (artifact_id, color_hex, color_name, percentage)
    VALUES (%s, %s, %s, %s)
    """
    
    data_tuples = [tuple(row) for row in df_colors.values]
    cursor.executemany(insert_query, data_tuples)
    
    conn.commit()
    cursor.close()
    conn.close()
```

### Complete ETL Pipeline

```python
def run_etl_pipeline(num_records=100):
    """
    Execute complete ETL pipeline
    """
    api_key = os.getenv('HARVARD_API_KEY')
    
    # Extract
    print("Extracting data from API...")
    data = fetch_artifacts(api_key, num_records=num_records)
    artifacts = data['records']
    
    # Transform
    print("Transforming data...")
    df_metadata = transform_artifact_metadata(artifacts)
    df_media = transform_artifact_media(artifacts)
    df_colors = transform_artifact_colors(artifacts)
    
    # Load
    print("Loading data into database...")
    load_metadata(df_metadata)
    load_media(df_media)
    load_colors(df_colors)
    
    print(f"ETL pipeline completed: {len(artifacts)} artifacts processed")
    
    return {
        'artifacts_count': len(artifacts),
        'media_count': len(df_media),
        'colors_count': len(df_colors)
    }
```

## Analytical SQL Queries

### Example Analytics Queries

```python
def execute_query(query):
    """
    Execute SQL query and return results as DataFrame
    """
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# Query 1: Artifacts by Culture
query_culture = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10
"""

# Query 2: Artifacts by Century
query_century = """
SELECT century, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY artifact_count DESC
"""

# Query 3: Department Distribution
query_department = """
SELECT department, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY artifact_count DESC
"""

# Query 4: Most Common Colors
query_colors = """
SELECT color_name, COUNT(*) as color_count, AVG(percentage) as avg_percentage
FROM artifactcolors
WHERE color_name IS NOT NULL
GROUP BY color_name
ORDER BY color_count DESC
LIMIT 10
"""

# Query 5: Artifacts with Most Images
query_images = """
SELECT m.id, m.title, COUNT(med.media_id) as image_count
FROM artifactmetadata m
JOIN artifactmedia med ON m.id = med.artifact_id
GROUP BY m.id, m.title
ORDER BY image_count DESC
LIMIT 10
"""

# Query 6: Artifacts by Time Period
query_timeline = """
SELECT 
    FLOOR(dated_begin / 100) * 100 as century_start,
    COUNT(*) as artifact_count
FROM artifactmetadata
WHERE dated_begin IS NOT NULL
GROUP BY century_start
ORDER BY century_start
"""
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Data Analytics Dashboard")
    
    # Sidebar for ETL control
    st.sidebar.header("ETL Pipeline")
    num_records = st.sidebar.number_input("Number of records to fetch", 
                                          min_value=10, max_value=1000, value=100)
    
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Running ETL pipeline..."):
            result = run_etl_pipeline(num_records)
            st.sidebar.success(f"✅ Processed {result['artifacts_count']} artifacts")
    
    # Analytics Section
    st.header("📊 Analytics Dashboard")
    
    analysis_type = st.selectbox(
        "Select Analysis",
        ["Artifacts by Culture", "Artifacts by Century", "Department Distribution",
         "Most Common Colors", "Artifacts with Most Images", "Timeline Analysis"]
    )
    
    queries = {
        "Artifacts by Culture": query_culture,
        "Artifacts by Century": query_century,
        "Department Distribution": query_department,
        "Most Common Colors": query_colors,
        "Artifacts with Most Images": query_images,
        "Timeline Analysis": query_timeline
    }
    
    if st.button("Run Analysis"):
        query = queries[analysis_type]
        df_result = execute_query(query)
        
        # Display results
        st.subheader("Query Results")
        st.dataframe(df_result)
        
        # Visualization
        if len(df_result) > 0:
            st.subheader("Visualization")
            x_col = df_result.columns[0]
            y_col = df_result.columns[1]
            
            fig = px.bar(df_result, x=x_col, y=y_col, 
                        title=f"{analysis_type}")
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Common Patterns

### Pagination Handling

```python
def fetch_all_artifacts(api_key, total_records=500):
    """
    Fetch multiple pages of artifacts
    """
    all_artifacts = []
    page = 1
    records_per_page = 100
    
    while len(all_artifacts) < total_records:
        data = fetch_artifacts(api_key, num_records=records_per_page, page=page)
        artifacts = data['records']
        
        if not artifacts:
            break
            
        all_artifacts.extend(artifacts)
        page += 1
    
    return all_artifacts[:total_records]
```

### Error Handling for API Requests

```python
import time

def fetch_artifacts_with_retry(api_key, num_records=100, max_retries=3):
    """
    Fetch artifacts with retry logic
    """
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, num_records)
        except Exception as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt  # Exponential backoff
                print(f"Retry {attempt + 1}/{max_retries} after {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise e
```

### Incremental Data Loading

```python
def get_last_artifact_id():
    """
    Get the last loaded artifact ID
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()[0]
    cursor.close()
    conn.close()
    return result or 0

def incremental_etl():
    """
    Load only new artifacts
    """
    last_id = get_last_artifact_id()
    # Fetch artifacts with ID > last_id
    # Transform and load new data only
    pass
```

## Troubleshooting

### API Rate Limiting

If you encounter rate limiting errors:
```python
import time

def fetch_with_rate_limit(api_key, requests_per_minute=60):
    delay = 60 / requests_per_minute
    data = fetch_artifacts(api_key)
    time.sleep(delay)
    return data
```

### Database Connection Issues

```python
def test_db_connection():
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        cursor.close()
        conn.close()
        print("✅ Database connection successful")
        return True
    except Error as e:
        print(f"❌ Database connection failed: {e}")
        return False
```

### Missing API Response Fields

```python
def safe_get(dictionary, key, default=None):
    """
    Safely get nested dictionary values
    """
    return dictionary.get(key, default)

# Usage in transformation
metadata = {
    'id': safe_get(artifact, 'id', 0),
    'title': safe_get(artifact, 'title', 'Untitled'),
    # ... other fields
}
```

### Large Dataset Memory Management

```python
def batch_process_artifacts(artifacts, batch_size=50):
    """
    Process artifacts in batches to manage memory
    """
    for i in range(0, len(artifacts), batch_size):
        batch = artifacts[i:i + batch_size]
        
        df_metadata = transform_artifact_metadata(batch)
        df_media = transform_artifact_media(batch)
        df_colors = transform_artifact_colors(batch)
        
        load_metadata(df_metadata)
        load_media(df_media)
        load_colors(df_colors)
        
        print(f"Processed batch {i//batch_size + 1}")
```
