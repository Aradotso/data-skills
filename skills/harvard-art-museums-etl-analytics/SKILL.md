---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - fetch and analyze Harvard Art Museums API data
  - set up data engineering pipeline with Streamlit
  - query and visualize museum collection data
  - extract transform load Harvard museum artifacts
  - build SQL analytics for art collections
  - create interactive museum data visualization
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

This project provides an end-to-end data engineering solution for collecting, processing, and analyzing Harvard Art Museums artifact data. It demonstrates production-grade ETL pipelines, SQL database design, and interactive analytics dashboards using Streamlit.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

## Configuration

### API Key Setup

1. Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Create a `.env` file in the project root:

```bash
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
DB_PORT=3306
```

### Database Setup

Create the required tables in your MySQL/TiDB database:

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    dated VARCHAR(200),
    period VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accessionyear INT,
    provenance TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    primaryimageurl VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
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

## Key Components

### 1. API Data Collection

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = 'https://api.harvardartmuseums.org/object'
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        data = response.json()
        return data['records'], data['info']
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Example: Fetch first 100 artifacts
artifacts, info = fetch_artifacts(page=1, size=100)
print(f"Total records available: {info['totalrecords']}")
print(f"Fetched {len(artifacts)} artifacts")
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

def transform_artifacts(raw_data: List[Dict]) -> tuple:
    """Transform raw API data into structured dataframes"""
    
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:200],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:200],
            'department': artifact.get('department', '')[:200],
            'division': artifact.get('division', '')[:200],
            'dated': artifact.get('dated', '')[:200],
            'period': artifact.get('period', '')[:200],
            'technique': artifact.get('technique', '')[:500],
            'medium': artifact.get('medium', '')[:500],
            'dimensions': artifact.get('dimensions', '')[:500],
            'creditline': artifact.get('creditline', ''),
            'accessionyear': artifact.get('accessionyear'),
            'provenance': artifact.get('provenance', '')
        }
        metadata_records.append(metadata)
        
        # Extract media information
        if 'primaryimageurl' in artifact:
            media = {
                'artifact_id': artifact.get('id'),
                'baseimageurl': artifact.get('baseimageurl', ''),
                'iiifbaseuri': artifact.get('iiifbaseuri', ''),
                'primaryimageurl': artifact.get('primaryimageurl', '')
            }
            media_records.append(media)
        
        # Extract color data
        if 'colors' in artifact and artifact['colors']:
            for color in artifact['colors']:
                color_record = {
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color', ''),
                    'spectrum': color.get('spectrum', ''),
                    'hue': color.get('hue', ''),
                    'percent': color.get('percent', 0.0)
                }
                color_records.append(color_record)
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )

# Example usage
artifacts, _ = fetch_artifacts(page=1, size=50)
metadata_df, media_df, colors_df = transform_artifacts(artifacts)
print(f"Transformed {len(metadata_df)} metadata records")
print(f"Transformed {len(media_df)} media records")
print(f"Transformed {len(colors_df)} color records")
```

### 3. Database Load Operations

```python
def get_db_connection():
    """Create database connection"""
    load_dotenv()
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME'),
        port=int(os.getenv('DB_PORT', 3306))
    )

def load_to_database(metadata_df, media_df, colors_df):
    """Batch insert data into SQL database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Insert metadata (handle duplicates)
        metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, 
         division, dated, period, technique, medium, dimensions, 
         creditline, accessionyear, provenance)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE 
        title=VALUES(title), culture=VALUES(culture)
        """
        cursor.executemany(metadata_query, metadata_df.values.tolist())
        
        # Insert media
        if not media_df.empty:
            media_query = """
            INSERT INTO artifactmedia 
            (artifact_id, baseimageurl, iiifbaseuri, primaryimageurl)
            VALUES (%s, %s, %s, %s)
            """
            cursor.executemany(media_query, media_df.values.tolist())
        
        # Insert colors
        if not colors_df.empty:
            colors_query = """
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
            """
            cursor.executemany(colors_query, colors_df.values.tolist())
        
        conn.commit()
        print(f"Successfully loaded {len(metadata_df)} artifacts to database")
        
    except Exception as e:
        conn.rollback()
        raise Exception(f"Database load failed: {str(e)}")
    finally:
        cursor.close()
        conn.close()
```

### 4. SQL Analytics Queries

```python
def execute_analytics_query(query: str) -> pd.DataFrame:
    """Execute analytical query and return results as DataFrame"""
    conn = get_db_connection()
    
    try:
        df = pd.read_sql(query, conn)
        return df
    finally:
        conn.close()

# Example analytics queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 20
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Top Departments": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        GROUP BY department
        ORDER BY total_artifacts DESC
        LIMIT 10
    """,
    
    "Color Distribution": """
        SELECT color, COUNT(*) as frequency,
               AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "Artifacts with Media": """
        SELECT 
            COUNT(DISTINCT m.id) as total_artifacts,
            COUNT(DISTINCT am.artifact_id) as with_media,
            ROUND(COUNT(DISTINCT am.artifact_id) * 100.0 / COUNT(DISTINCT m.id), 2) as media_percentage
        FROM artifactmetadata m
        LEFT JOIN artifactmedia am ON m.id = am.artifact_id
    """,
    
    "Classification Analysis": """
        SELECT classification, 
               COUNT(*) as count,
               COUNT(DISTINCT culture) as unique_cultures
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 15
    """
}

# Execute query example
df = execute_analytics_query(ANALYTICS_QUERIES["Artifacts by Culture"])
print(df.head())
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar for operations
    with st.sidebar:
        st.header("⚙️ Operations")
        
        # Data Collection
        if st.button("🔄 Fetch New Data"):
            with st.spinner("Fetching artifacts from API..."):
                try:
                    artifacts, info = fetch_artifacts(page=1, size=100)
                    metadata_df, media_df, colors_df = transform_artifacts(artifacts)
                    load_to_database(metadata_df, media_df, colors_df)
                    st.success(f"Successfully loaded {len(artifacts)} artifacts!")
                except Exception as e:
                    st.error(f"Error: {str(e)}")
    
    # Analytics Section
    st.header("📊 Analytics Dashboard")
    
    query_choice = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Run Analysis"):
        with st.spinner("Executing query..."):
            try:
                df = execute_analytics_query(ANALYTICS_QUERIES[query_choice])
                
                # Display results
                st.subheader("Query Results")
                st.dataframe(df, use_container_width=True)
                
                # Visualization
                if len(df) > 0 and len(df.columns) >= 2:
                    st.subheader("Visualization")
                    fig = px.bar(
                        df,
                        x=df.columns[0],
                        y=df.columns[1],
                        title=query_choice
                    )
                    st.plotly_chart(fig, use_container_width=True)
                    
            except Exception as e:
                st.error(f"Query failed: {str(e)}")

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Incremental Data Loading

```python
def get_last_artifact_id():
    """Get the highest artifact ID in database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    conn.close()
    return result[0] if result[0] else 0

def incremental_load(num_pages=5):
    """Load only new artifacts"""
    last_id = get_last_artifact_id()
    
    for page in range(1, num_pages + 1):
        artifacts, _ = fetch_artifacts(page=page, size=100)
        
        # Filter only new artifacts
        new_artifacts = [a for a in artifacts if a['id'] > last_id]
        
        if new_artifacts:
            metadata_df, media_df, colors_df = transform_artifacts(new_artifacts)
            load_to_database(metadata_df, media_df, colors_df)
            print(f"Page {page}: Loaded {len(new_artifacts)} new artifacts")
        else:
            print(f"Page {page}: No new artifacts")
```

### Error Handling and Retry Logic

```python
import time
from requests.exceptions import RequestException

def fetch_with_retry(page=1, max_retries=3):
    """Fetch artifacts with exponential backoff retry"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page=page)
        except RequestException as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt
                print(f"Attempt {attempt + 1} failed. Retrying in {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise Exception(f"Failed after {max_retries} attempts: {str(e)}")
```

## Troubleshooting

### API Rate Limiting

```python
# Add rate limiting to API calls
import time

def fetch_artifacts_with_rate_limit(page=1, size=100, delay=1):
    """Fetch artifacts with rate limiting"""
    time.sleep(delay)  # Wait between requests
    return fetch_artifacts(page, size)
```

### Database Connection Issues

```python
# Test database connection
def test_db_connection():
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        cursor.close()
        conn.close()
        print("✓ Database connection successful")
        return True
    except Exception as e:
        print(f"✗ Database connection failed: {str(e)}")
        return False
```

### Memory Management for Large Datasets

```python
def batch_process_artifacts(total_pages=100, batch_size=10):
    """Process artifacts in batches to manage memory"""
    for batch_start in range(1, total_pages + 1, batch_size):
        batch_end = min(batch_start + batch_size, total_pages + 1)
        
        for page in range(batch_start, batch_end):
            artifacts, _ = fetch_artifacts(page=page)
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            load_to_database(metadata_df, media_df, colors_df)
        
        print(f"Completed batch {batch_start}-{batch_end-1}")
        time.sleep(2)  # Rate limiting between batches
```

## Advanced Analytics Examples

### Time Series Analysis

```python
time_series_query = """
SELECT 
    accessionyear as year,
    COUNT(*) as acquisitions,
    COUNT(DISTINCT culture) as cultures_acquired
FROM artifactmetadata
WHERE accessionyear IS NOT NULL
    AND accessionyear BETWEEN 1900 AND 2024
GROUP BY accessionyear
ORDER BY accessionyear
"""

df = execute_analytics_query(time_series_query)
fig = px.line(df, x='year', y='acquisitions', 
              title='Artifact Acquisitions Over Time')
```

### Cross-Table Joins

```python
complex_query = """
SELECT 
    m.culture,
    m.classification,
    c.color,
    COUNT(*) as artifact_count,
    AVG(c.percent) as avg_color_percent
FROM artifactmetadata m
JOIN artifactcolors c ON m.id = c.artifact_id
WHERE m.culture IS NOT NULL 
    AND c.color IS NOT NULL
GROUP BY m.culture, m.classification, c.color
HAVING artifact_count > 5
ORDER BY artifact_count DESC
LIMIT 50
"""
```
