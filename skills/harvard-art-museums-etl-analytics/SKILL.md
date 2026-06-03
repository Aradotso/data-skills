---
name: harvard-art-museums-etl-analytics
description: End-to-end ETL pipeline and analytics app using Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - fetch and analyze Harvard art collection data
  - set up data engineering pipeline with Streamlit
  - query Harvard Art Museums API and store in database
  - visualize museum artifact data with SQL analytics
  - process art museum collection data into structured format
  - implement artifact metadata ETL workflow
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides a complete data engineering solution for working with the Harvard Art Museums API. It implements an ETL pipeline that extracts artifact data, transforms nested JSON structures into relational tables, loads data into SQL databases (MySQL/TiDB Cloud), and provides interactive analytics dashboards using Streamlit.

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required dependencies:**
- streamlit
- pandas
- requests
- mysql-connector-python (or pymysql)
- plotly
- python-dotenv

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Obtaining Harvard API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Request an API key (free for non-commercial use)
3. Add to your `.env` file

### Database Setup

The application creates three main tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    classification VARCHAR(200),
    objectnumber VARCHAR(100),
    accessionyear INT,
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    primaryimageurl TEXT,
    totalpageviews INT,
    totaluniquepageviews INT
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    iiifbaseuri VARCHAR(500),
    baseimageurl VARCHAR(500),
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

The Streamlit app will open in your browser at `http://localhost:8501`

## Core Components

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
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Fetch with pagination
def fetch_all_artifacts(max_records=1000):
    """Fetch multiple pages of artifacts"""
    all_records = []
    page = 1
    records_per_page = 100
    
    while len(all_records) < max_records:
        data = fetch_artifacts(page=page, size=records_per_page)
        records = data.get('records', [])
        
        if not records:
            break
            
        all_records.extend(records)
        page += 1
        
    return all_records[:max_records]
```

### 2. ETL Transformation

```python
import pandas as pd

def transform_artifact_metadata(records):
    """Transform raw API data into structured metadata"""
    metadata_list = []
    
    for record in records:
        metadata = {
            'id': record.get('id'),
            'title': record.get('title'),
            'culture': record.get('culture'),
            'period': record.get('period'),
            'century': record.get('century'),
            'dated': record.get('dated'),
            'department': record.get('department'),
            'division': record.get('division'),
            'classification': record.get('classification'),
            'objectnumber': record.get('objectnumber'),
            'accessionyear': record.get('accessionyear'),
            'technique': record.get('technique'),
            'medium': record.get('medium'),
            'dimensions': record.get('dimensions'),
            'creditline': record.get('creditline'),
            'primaryimageurl': record.get('primaryimageurl'),
            'totalpageviews': record.get('totalpageviews', 0),
            'totaluniquepageviews': record.get('totaluniquepageviews', 0)
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(records):
    """Extract media information"""
    media_list = []
    
    for record in records:
        artifact_id = record.get('id')
        images = record.get('images', [])
        
        for image in images:
            media = {
                'artifact_id': artifact_id,
                'iiifbaseuri': image.get('iiifbaseuri'),
                'baseimageurl': image.get('baseimageurl')
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(records):
    """Extract color information"""
    colors_list = []
    
    for record in records:
        artifact_id = record.get('id')
        colors = record.get('colors', [])
        
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            colors_list.append(color_data)
    
    return pd.DataFrame(colors_list)
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=os.getenv('DB_PORT', 3306),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_metadata(df):
    """Load metadata into database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, period, century, dated, department, division, 
     classification, objectnumber, accessionyear, technique, medium, 
     dimensions, creditline, primaryimageurl, totalpageviews, totaluniquepageviews)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    data = df.values.tolist()
    cursor.executemany(insert_query, data)
    conn.commit()
    
    cursor.close()
    conn.close()

def load_media(df):
    """Load media data into database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (artifact_id, iiifbaseuri, baseimageurl)
    VALUES (%s, %s, %s)
    """
    
    data = df.values.tolist()
    cursor.executemany(insert_query, data)
    conn.commit()
    
    cursor.close()
    conn.close()

def load_colors(df):
    """Load color data into database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors (artifact_id, color, spectrum, hue, percent)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    data = df.values.tolist()
    cursor.executemany(insert_query, data)
    conn.commit()
    
    cursor.close()
    conn.close()
```

### 4. SQL Analytics Queries

```python
# Example analytical queries

ANALYTICS_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 20
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "most_viewed_artifacts": """
        SELECT title, culture, totalpageviews
        FROM artifactmetadata
        WHERE totalpageviews > 0
        ORDER BY totalpageviews DESC
        LIMIT 20
    """,
    
    "artifacts_by_department": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "color_distribution": """
        SELECT spectrum, COUNT(*) as count, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY spectrum
        ORDER BY count DESC
    """,
    
    "artifacts_with_media": """
        SELECT 
            (SELECT COUNT(*) FROM artifactmetadata) as total_artifacts,
            (SELECT COUNT(DISTINCT artifact_id) FROM artifactmedia) as artifacts_with_media,
            ROUND((SELECT COUNT(DISTINCT artifact_id) FROM artifactmedia) * 100.0 / 
                  (SELECT COUNT(*) FROM artifactmetadata), 2) as percentage_with_media
    """,
    
    "accession_year_distribution": """
        SELECT accessionyear, COUNT(*) as count
        FROM artifactmetadata
        WHERE accessionyear IS NOT NULL AND accessionyear > 1800
        GROUP BY accessionyear
        ORDER BY accessionyear DESC
    """
}

def execute_query(query_name):
    """Execute analytical query and return results"""
    conn = get_db_connection()
    query = ANALYTICS_QUERIES.get(query_name)
    
    df = pd.read_sql(query, conn)
    conn.close()
    
    return df
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "Analytics", "Visualization"]
    )
    
    if page == "Data Collection":
        st.header("Data Collection from API")
        
        num_records = st.number_input("Number of records to fetch", 
                                      min_value=10, 
                                      max_value=1000, 
                                      value=100)
        
        if st.button("Fetch Data"):
            with st.spinner("Fetching data from Harvard API..."):
                records = fetch_all_artifacts(max_records=num_records)
                st.success(f"Fetched {len(records)} records")
                
                # Transform
                metadata_df = transform_artifact_metadata(records)
                media_df = transform_artifact_media(records)
                colors_df = transform_artifact_colors(records)
                
                # Load
                load_metadata(metadata_df)
                load_media(media_df)
                load_colors(colors_df)
                
                st.success("Data loaded into database!")
    
    elif page == "Analytics":
        st.header("SQL Analytics")
        
        query_name = st.selectbox(
            "Select Query",
            list(ANALYTICS_QUERIES.keys())
        )
        
        if st.button("Run Query"):
            df = execute_query(query_name)
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df.columns) == 2 and 'count' in df.columns:
                fig = px.bar(df, x=df.columns[0], y='count',
                            title=f"Distribution: {query_name}")
                st.plotly_chart(fig)
    
    elif page == "Visualization":
        st.header("Interactive Visualizations")
        
        # Custom visualizations
        viz_type = st.selectbox(
            "Select Visualization",
            ["Culture Distribution", "Century Timeline", "Color Analysis"]
        )
        
        if viz_type == "Culture Distribution":
            df = execute_query("artifacts_by_culture")
            fig = px.bar(df, x='culture', y='count',
                        title="Artifacts by Culture")
            st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Full ETL Pipeline Execution

```python
def run_etl_pipeline(max_records=500):
    """Complete ETL pipeline execution"""
    print("Starting ETL pipeline...")
    
    # Extract
    print("1. Extracting data from API...")
    records = fetch_all_artifacts(max_records=max_records)
    print(f"   Extracted {len(records)} records")
    
    # Transform
    print("2. Transforming data...")
    metadata_df = transform_artifact_metadata(records)
    media_df = transform_artifact_media(records)
    colors_df = transform_artifact_colors(records)
    print(f"   Metadata records: {len(metadata_df)}")
    print(f"   Media records: {len(media_df)}")
    print(f"   Color records: {len(colors_df)}")
    
    # Load
    print("3. Loading data to database...")
    load_metadata(metadata_df)
    load_media(media_df)
    load_colors(colors_df)
    print("   Data loaded successfully!")
    
    return metadata_df, media_df, colors_df
```

### Incremental Data Updates

```python
def fetch_recent_artifacts(since_date):
    """Fetch only recently updated artifacts"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'lastupdate': since_date,
        'hasimage': 1
    }
    
    response = requests.get(base_url, params=params)
    return response.json().get('records', [])
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_rate_limit(page, size=100, delay=1):
    """Fetch with delay to respect rate limits"""
    time.sleep(delay)
    return fetch_artifacts(page=page, size=size)
```

### Database Connection Issues

```python
def get_db_connection_with_retry(max_retries=3):
    """Database connection with retry logic"""
    for attempt in range(max_retries):
        try:
            conn = get_db_connection()
            return conn
        except Error as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)
```

### Handling Missing Data

```python
def safe_transform(record, field, default=None):
    """Safely extract field with default value"""
    value = record.get(field, default)
    if isinstance(value, list) and len(value) > 0:
        return value[0]
    return value if value else default
```

### Memory Management for Large Datasets

```python
def batch_process_artifacts(total_records, batch_size=100):
    """Process artifacts in batches to manage memory"""
    for i in range(0, total_records, batch_size):
        batch = fetch_artifacts(page=(i // batch_size) + 1, size=batch_size)
        
        # Transform and load immediately
        metadata_df = transform_artifact_metadata(batch)
        load_metadata(metadata_df)
        
        print(f"Processed batch {i // batch_size + 1}")
```

## Key Analytics Queries

The project includes 20+ predefined analytics queries for insights like:
- Artifact distribution by culture, century, department
- Media availability analysis
- Color usage patterns
- Pageview statistics
- Accession year trends
- Classification breakdowns

Access these through the Streamlit dashboard or execute directly via `execute_query()` function.
