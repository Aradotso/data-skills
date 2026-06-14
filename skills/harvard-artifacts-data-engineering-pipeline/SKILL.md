---
name: harvard-artifacts-data-engineering-pipeline
description: Build end-to-end data engineering pipelines using the Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build a data pipeline with harvard art museums api
  - create etl pipeline for museum artifact data
  - analyze harvard art collection with sql
  - visualize museum data using streamlit
  - fetch and transform harvard api data
  - build analytics dashboard for art museum data
  - setup tidb pipeline for artifact metadata
  - query harvard art museums collection
---

# Harvard Artifacts Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates building production-ready data engineering pipelines using the Harvard Art Museums API. It showcases ETL workflows, relational database design, SQL analytics, and interactive visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering App provides:

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads nested JSON into normalized SQL tables
- **Database Management**: Stores data in MySQL/TiDB Cloud with proper schema design
- **SQL Analytics**: 20+ predefined analytical queries for artifact insights
- **Visualization**: Interactive Plotly charts via Streamlit dashboard

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Dependencies** (typical setup):
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

Create a `.env` file:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_db_host
DB_PORT=4000
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Schema

The pipeline creates three main tables:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    object_id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    description TEXT,
    accession_number VARCHAR(100),
    primary_image_url TEXT
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    object_id INT,
    media_url TEXT,
    media_type VARCHAR(100),
    FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    object_id INT,
    color_hex VARCHAR(10),
    color_percent DECIMAL(5,2),
    FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
);
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Key Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
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

# Fetch multiple pages with pagination
def fetch_all_artifacts(total_pages=5):
    all_artifacts = []
    for page in range(1, total_pages + 1):
        data = fetch_artifacts(page=page)
        all_artifacts.extend(data.get('records', []))
        print(f"Fetched page {page}, total records: {len(all_artifacts)}")
    return all_artifacts
```

### 2. ETL Pipeline - Transform

```python
import pandas as pd

def transform_artifact_data(raw_artifacts):
    """Transform nested JSON into relational dataframes"""
    
    # Metadata transformation
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_artifacts:
        # Extract metadata
        metadata = {
            'object_id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'description': artifact.get('description'),
            'accession_number': artifact.get('accessionyear'),
            'primary_image_url': artifact.get('primaryimageurl')
        }
        metadata_records.append(metadata)
        
        # Extract media (multiple images per artifact)
        for image in artifact.get('images', []):
            media_records.append({
                'object_id': artifact.get('id'),
                'media_url': image.get('baseimageurl'),
                'media_type': 'image'
            })
        
        # Extract colors
        for color in artifact.get('colors', []):
            color_records.append({
                'object_id': artifact.get('id'),
                'color_hex': color.get('hex'),
                'color_percent': color.get('percent')
            })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### 3. Load to Database

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

def load_metadata(df_metadata):
    """Batch insert metadata into database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (object_id, title, culture, period, century, classification, 
         department, dated, description, accession_number, primary_image_url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    # Prepare data tuples
    data = [tuple(row) for row in df_metadata.to_numpy()]
    
    try:
        cursor.executemany(insert_query, data)
        conn.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
    except Error as e:
        print(f"Error: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()

def load_media(df_media):
    """Load media data"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia (object_id, media_url, media_type)
        VALUES (%s, %s, %s)
    """
    
    data = [tuple(row) for row in df_media.to_numpy()]
    cursor.executemany(insert_query, data)
    conn.commit()
    cursor.close()
    conn.close()
```

### 4. SQL Analytics Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 20
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
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Top Colors Used": """
        SELECT color_hex, SUM(color_percent) as total_usage
        FROM artifactcolors
        GROUP BY color_hex
        ORDER BY total_usage DESC
        LIMIT 15
    """,
    
    "Artifacts with Most Images": """
        SELECT m.object_id, m.title, COUNT(a.media_id) as image_count
        FROM artifactmetadata m
        JOIN artifactmedia a ON m.object_id = a.object_id
        GROUP BY m.object_id, m.title
        ORDER BY image_count DESC
        LIMIT 10
    """
}

def execute_analytics_query(query_name):
    """Execute analytics query and return dataframe"""
    conn = get_db_connection()
    df = pd.read_sql(ANALYTICS_QUERIES[query_name], conn)
    conn.close()
    return df
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Options")
    query_name = st.sidebar.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    # Execute query
    if st.sidebar.button("Run Query"):
        with st.spinner("Executing query..."):
            df_results = execute_analytics_query(query_name)
            
            # Display results
            st.subheader(f"Results: {query_name}")
            st.dataframe(df_results)
            
            # Auto-generate visualization
            if len(df_results.columns) == 2:
                fig = px.bar(
                    df_results,
                    x=df_results.columns[0],
                    y=df_results.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Complete ETL Workflow

```python
def run_etl_pipeline(num_pages=5):
    """Run complete ETL pipeline"""
    
    # Extract
    print("Extracting data from API...")
    raw_data = fetch_all_artifacts(total_pages=num_pages)
    
    # Transform
    print("Transforming data...")
    df_metadata, df_media, df_colors = transform_artifact_data(raw_data)
    
    # Load
    print("Loading to database...")
    load_metadata(df_metadata)
    load_media(df_media)
    load_colors(df_colors)
    
    print("ETL pipeline completed successfully!")

# Run pipeline
run_etl_pipeline(num_pages=10)
```

### Rate Limiting API Calls

```python
import time

def fetch_with_rate_limit(page, delay=0.5):
    """Fetch with rate limiting"""
    data = fetch_artifacts(page=page)
    time.sleep(delay)  # Respect API rate limits
    return data
```

## Troubleshooting

**API Key Issues**
```python
# Verify API key is loaded
if not os.getenv('HARVARD_API_KEY'):
    raise ValueError("HARVARD_API_KEY not found in environment")
```

**Database Connection Errors**
```python
# Test connection
try:
    conn = get_db_connection()
    print("Database connected successfully!")
    conn.close()
except Error as e:
    print(f"Connection failed: {e}")
```

**Empty Results**
```python
# Check if API returns data
response = fetch_artifacts(page=1)
if not response.get('records'):
    print("No records found. Check API parameters.")
```

**Memory Issues with Large Datasets**
```python
# Process in chunks
def load_in_chunks(df, chunk_size=1000):
    for i in range(0, len(df), chunk_size):
        chunk = df.iloc[i:i+chunk_size]
        load_metadata(chunk)
```
