---
name: harvard-artifacts-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - extract and transform Harvard API data to SQL
  - set up artifact collection data pipeline
  - query and visualize museum artifact data
  - implement Harvard museums data engineering workflow
  - analyze art museum collections with SQL
  - build Streamlit app for artifact analytics
---

# Harvard Artifacts Collection Data Engineering & Analytics App

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is an end-to-end data engineering and analytics application that demonstrates real-world ETL pipelines using the Harvard Art Museums API. It extracts artifact data, transforms it into relational tables, loads it into MySQL/TiDB, and provides interactive SQL analytics through a Streamlit dashboard.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

**Core Components:**
- API integration with Harvard Art Museums
- ETL pipeline for nested JSON to relational data
- MySQL/TiDB database with normalized schema
- 20+ predefined analytical SQL queries
- Interactive Streamlit dashboard with Plotly visualizations

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

### Environment Variables

Create a `.env` file in the project root:

```bash
# Harvard API Configuration
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_db_host
DB_PORT=4000
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Schema

The application uses three main tables with foreign key relationships:

```sql
-- Artifact Metadata (main table)
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(255),
    dated VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    division VARCHAR(255),
    contact VARCHAR(255),
    description TEXT,
    provenance TEXT,
    copyright TEXT,
    totalpageviews INT,
    totaluniquepageviews INT,
    accessionyear INT,
    url TEXT
);

-- Artifact Media (one-to-many)
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    iiifbaseuri VARCHAR(500),
    baseimageurl VARCHAR(500),
    primaryurl VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors (one-to-many)
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
# Start Streamlit app
streamlit run app.py

# Access the dashboard at http://localhost:8501
```

## Key Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
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

def collect_all_artifacts(total_records=500):
    """Collect multiple pages of artifact data"""
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < total_records:
        data = fetch_artifacts(page=page, size=size)
        all_artifacts.extend(data.get('records', []))
        
        if not data.get('info', {}).get('next'):
            break
        
        page += 1
    
    return all_artifacts[:total_records]
```

### 2. ETL Transformation

```python
import pandas as pd

def transform_artifact_data(artifacts):
    """Transform nested JSON into relational dataframes"""
    
    # Main metadata
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'division': artifact.get('division'),
            'contact': artifact.get('contact'),
            'description': artifact.get('description'),
            'provenance': artifact.get('provenance'),
            'copyright': artifact.get('copyright'),
            'totalpageviews': artifact.get('totalpageviews'),
            'totaluniquepageviews': artifact.get('totaluniquepageviews'),
            'accessionyear': artifact.get('accessionyear'),
            'url': artifact.get('url')
        }
        metadata_records.append(metadata)
        
        # Extract media (nested)
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        for img in images:
            media = {
                'artifact_id': artifact_id,
                'iiifbaseuri': img.get('iiifbaseuri'),
                'baseimageurl': img.get('baseimageurl'),
                'primaryurl': img.get('primaryurl') if img.get('primary') else None
            }
            media_records.append(media)
        
        # Extract colors (nested)
        colors = artifact.get('colors', [])
        for color in colors:
            color_record = {
                'artifact_id': artifact_id,
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

### 3. Database Loading

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

def load_data_to_sql(metadata_df, media_df, colors_df):
    """Batch insert data into SQL tables"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Insert metadata (parent table first)
        for _, row in metadata_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmetadata 
                (id, title, culture, century, dated, classification, department, 
                 technique, medium, dimensions, creditline, division, contact, 
                 description, provenance, copyright, totalpageviews, 
                 totaluniquepageviews, accessionyear, url)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE title=VALUES(title)
            """, tuple(row))
        
        # Insert media
        for _, row in media_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmedia (artifact_id, iiifbaseuri, baseimageurl, primaryurl)
                VALUES (%s, %s, %s, %s)
            """, tuple(row))
        
        # Insert colors
        for _, row in colors_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactcolors (artifact_id, color, spectrum, hue, percent)
                VALUES (%s, %s, %s, %s, %s)
            """, tuple(row))
        
        conn.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Error: {e}")
        conn.rollback()
    finally:
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
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Media Availability": """
        SELECT 
            CASE WHEN m.artifact_id IS NOT NULL THEN 'Has Media' ELSE 'No Media' END as media_status,
            COUNT(DISTINCT a.id) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY media_status
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "Pageview Statistics": """
        SELECT 
            department,
            AVG(totalpageviews) as avg_views,
            MAX(totalpageviews) as max_views,
            COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE totalpageviews IS NOT NULL
        GROUP BY department
        ORDER BY avg_views DESC
    """
}

def execute_query(query_name):
    """Execute analytical query and return results"""
    conn = get_db_connection()
    query = ANALYTICS_QUERIES[query_name]
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Artifacts Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Queries")
    query_name = st.sidebar.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    # Execute query
    if st.sidebar.button("Run Query"):
        with st.spinner("Executing query..."):
            df = execute_query(query_name)
            
            # Display results
            st.subheader(f"Results: {query_name}")
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df.columns) >= 2:
                fig = px.bar(
                    df.head(20),
                    x=df.columns[0],
                    y=df.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig, use_container_width=True)
    
    # ETL Pipeline section
    st.sidebar.header("ETL Pipeline")
    num_records = st.sidebar.number_input("Records to Fetch", 100, 1000, 500)
    
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            artifacts = collect_all_artifacts(num_records)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifact_data(artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            load_data_to_sql(metadata_df, media_df, colors_df)
            st.success("Data loaded to SQL database")

if __name__ == "__main__":
    main()
```

## Common Patterns

### Error Handling for API Calls

```python
import time

def fetch_with_retry(page, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page)
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
            wait_time = 2 ** attempt
            print(f"Retry {attempt + 1} after {wait_time}s")
            time.sleep(wait_time)
```

### Batch Processing

```python
def batch_insert(df, table_name, batch_size=100):
    """Insert data in batches for performance"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    for i in range(0, len(df), batch_size):
        batch = df.iloc[i:i+batch_size]
        # Insert batch
        conn.commit()
    
    cursor.close()
    conn.close()
```

## Troubleshooting

**API Rate Limiting:**
- Implement delays between requests: `time.sleep(0.5)`
- Use smaller page sizes
- Cache responses locally

**Database Connection Issues:**
- Verify environment variables are loaded
- Check firewall/network access to DB
- Ensure TiDB Cloud IP whitelist is configured

**Memory Issues with Large Datasets:**
- Process data in chunks
- Use generators instead of loading all data
- Clear DataFrames after loading: `del df`

**Missing Data in Visualizations:**
- Handle NULL values in SQL queries with `WHERE column IS NOT NULL`
- Use `COALESCE()` for default values
- Filter out empty results before charting
