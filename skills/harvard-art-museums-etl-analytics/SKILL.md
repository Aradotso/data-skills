---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do i build an etl pipeline for the harvard art museums api
  - show me how to extract and transform harvard museum data
  - create a data analytics dashboard with streamlit and museum artifacts
  - how to load harvard api data into sql database
  - build an end-to-end data engineering project with museum data
  - visualize harvard art collection data with plotly
  - set up mysql database for artifact metadata
  - query and analyze art museum data with sql
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for cultural heritage data.

## What It Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load museum data into relational SQL databases
- **SQL Analytics**: Pre-built analytical queries for insights on artifacts, cultures, periods, and media
- **Interactive Dashboards**: Streamlit-based visualization with Plotly charts
- **Database Design**: Normalized schema with metadata, media, and color tables

## Installation

### Prerequisites

```bash
# Python 3.8+
# MySQL or TiDB Cloud database instance
```

### Setup

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests plotly mysql-connector-python python-dotenv
```

### Configuration

Create a `.env` file in the project root:

```env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_db_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

## Database Schema

### Create Tables

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dated VARCHAR(255),
    accession_year INT,
    url TEXT
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url TEXT,
    base_image_url TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API
    
    Args:
        api_key: Harvard API key
        page: Page number for pagination
        size: Number of records per page (max 100)
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=100)
print(f"Total records: {data['info']['totalrecords']}")
print(f"Fetched: {len(data['records'])} artifacts")
```

### Transform: Process JSON to DataFrames

```python
import pandas as pd

def transform_artifacts(api_response):
    """
    Transform API JSON response into structured DataFrames
    
    Returns:
        metadata_df, media_df, colors_df
    """
    records = api_response['records']
    
    # Metadata DataFrame
    metadata = []
    for record in records:
        metadata.append({
            'id': record.get('id'),
            'title': record.get('title'),
            'culture': record.get('culture'),
            'period': record.get('period'),
            'century': record.get('century'),
            'classification': record.get('classification'),
            'department': record.get('department'),
            'technique': record.get('technique'),
            'medium': record.get('medium'),
            'dated': record.get('dated'),
            'accession_year': record.get('accessionyear'),
            'url': record.get('url')
        })
    
    metadata_df = pd.DataFrame(metadata)
    
    # Media DataFrame
    media = []
    for record in records:
        artifact_id = record.get('id')
        images = record.get('images', [])
        for img in images:
            media.append({
                'artifact_id': artifact_id,
                'image_url': img.get('iiifbaseuri'),
                'base_image_url': img.get('baseimageurl')
            })
    
    media_df = pd.DataFrame(media)
    
    # Colors DataFrame
    colors = []
    for record in records:
        artifact_id = record.get('id')
        color_list = record.get('colors', [])
        for color in color_list:
            colors.append({
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'percentage': color.get('percent')
            })
    
    colors_df = pd.DataFrame(colors)
    
    return metadata_df, media_df, colors_df

# Usage
metadata_df, media_df, colors_df = transform_artifacts(data)
print(f"Metadata rows: {len(metadata_df)}")
print(f"Media rows: {len(media_df)}")
print(f"Colors rows: {len(colors_df)}")
```

### Load: Insert into Database

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_metadata(df):
    """Load metadata into artifactmetadata table"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, 
         department, technique, medium, dated, accession_year, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    data_tuples = [tuple(x) for x in df.values]
    cursor.executemany(insert_query, data_tuples)
    
    conn.commit()
    cursor.close()
    conn.close()
    print(f"Loaded {len(df)} metadata records")

def load_media(df):
    """Load media into artifactmedia table"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia (artifact_id, image_url, base_image_url)
        VALUES (%s, %s, %s)
    """
    
    data_tuples = [tuple(x) for x in df.values]
    cursor.executemany(insert_query, data_tuples)
    
    conn.commit()
    cursor.close()
    conn.close()
    print(f"Loaded {len(df)} media records")

def load_colors(df):
    """Load colors into artifactcolors table"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactcolors (artifact_id, color, percentage)
        VALUES (%s, %s, %s)
    """
    
    data_tuples = [tuple(x) for x in df.values]
    cursor.executemany(insert_query, data_tuples)
    
    conn.commit()
    cursor.close()
    conn.close()
    print(f"Loaded {len(df)} color records")

# Usage
load_metadata(metadata_df)
load_media(media_df)
load_colors(colors_df)
```

## Streamlit Dashboard

### Basic Dashboard Structure

```python
# app.py
import streamlit as st
import pandas as pd
import plotly.express as px
from dotenv import load_dotenv
import os

load_dotenv()

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Sidebar configuration
st.sidebar.header("Configuration")

# Database connection test
@st.cache_resource
def init_connection():
    return get_db_connection()

# Main tabs
tab1, tab2, tab3 = st.tabs(["📊 Analytics", "🔄 ETL Pipeline", "📈 Visualizations"])

with tab1:
    st.header("SQL Analytics Queries")
    
    queries = {
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
            ORDER BY count DESC
        """,
        "Department Distribution": """
            SELECT department, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY artifact_count DESC
        """,
        "Top Colors Used": """
            SELECT color, 
                   COUNT(*) as frequency,
                   AVG(percentage) as avg_percentage
            FROM artifactcolors
            GROUP BY color
            ORDER BY frequency DESC
            LIMIT 10
        """
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        conn = init_connection()
        df = pd.read_sql(queries[selected_query], conn)
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)

with tab2:
    st.header("ETL Pipeline Execution")
    
    num_pages = st.number_input("Number of pages to fetch", 1, 10, 1)
    
    if st.button("Run ETL"):
        with st.spinner("Fetching data from API..."):
            api_key = os.getenv('HARVARD_API_KEY')
            all_records = []
            
            for page in range(1, num_pages + 1):
                data = fetch_artifacts(api_key, page=page)
                all_records.extend(data['records'])
                st.write(f"Fetched page {page}")
            
            st.success(f"Fetched {len(all_records)} artifacts")
            
            # Transform
            st.write("Transforming data...")
            metadata_df, media_df, colors_df = transform_artifacts({'records': all_records})
            
            # Load
            st.write("Loading into database...")
            load_metadata(metadata_df)
            load_media(media_df)
            load_colors(colors_df)
            
            st.success("ETL Pipeline completed successfully!")

with tab3:
    st.header("Advanced Visualizations")
    
    conn = init_connection()
    
    # Artifacts over time
    query = """
        SELECT accession_year, COUNT(*) as count
        FROM artifactmetadata
        WHERE accession_year IS NOT NULL
        GROUP BY accession_year
        ORDER BY accession_year
    """
    df = pd.read_sql(query, conn)
    
    fig = px.line(df, x='accession_year', y='count',
                  title='Artifact Acquisitions Over Time')
    st.plotly_chart(fig, use_container_width=True)
```

## Common Analytical Queries

### Artifacts with Most Images

```sql
SELECT m.id, m.title, COUNT(med.media_id) as image_count
FROM artifactmetadata m
JOIN artifactmedia med ON m.id = med.artifact_id
GROUP BY m.id, m.title
ORDER BY image_count DESC
LIMIT 20;
```

### Color Diversity by Department

```sql
SELECT m.department, COUNT(DISTINCT c.color) as unique_colors
FROM artifactmetadata m
JOIN artifactcolors c ON m.id = c.artifact_id
WHERE m.department IS NOT NULL
GROUP BY m.department
ORDER BY unique_colors DESC;
```

### Artifacts by Period and Classification

```sql
SELECT period, classification, COUNT(*) as count
FROM artifactmetadata
WHERE period IS NOT NULL AND classification IS NOT NULL
GROUP BY period, classification
ORDER BY count DESC
LIMIT 15;
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(api_key, page, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except Exception as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt
                print(f"Retry {attempt + 1} after {wait_time}s")
                time.sleep(wait_time)
            else:
                raise e
```

### Database Connection Pooling

```python
from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    host=os.getenv('DB_HOST'),
    database=os.getenv('DB_NAME'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD')
)

def get_pooled_connection():
    return db_pool.get_connection()
```

### Handling NULL Values

```python
# Clean DataFrames before loading
metadata_df = metadata_df.where(pd.notnull(metadata_df), None)
metadata_df['accession_year'] = pd.to_numeric(metadata_df['accession_year'], errors='coerce')
```

## Best Practices

1. **Incremental Loading**: Track last processed artifact ID to avoid duplicates
2. **Batch Processing**: Process API data in batches of 100 records
3. **Error Logging**: Implement comprehensive logging for ETL failures
4. **Data Validation**: Validate foreign key relationships before insertion
5. **Caching**: Use Streamlit caching for database connections and query results
