---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - extract data from Harvard Art Museums API
  - set up data engineering pipeline with Streamlit
  - analyze art collection data with SQL
  - visualize museum artifact metadata
  - implement batch data ingestion from art API
  - design relational schema for museum artifacts
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build production-grade ETL pipelines that extract artifact data from the Harvard Art Museums API, transform it into relational structures, load it into SQL databases, and create interactive analytics dashboards using Streamlit.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App demonstrates:
- **API Integration**: Paginated data collection from Harvard Art Museums API with rate limiting
- **ETL Pipeline**: Transform nested JSON into normalized relational tables
- **SQL Storage**: Store artifacts, media, and color data in MySQL/TiDB Cloud
- **Analytics**: Execute 20+ predefined analytical queries
- **Visualization**: Interactive Plotly charts via Streamlit interface

**Architecture**: API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_db_host"
export DB_USER="your_db_user"
export DB_PASSWORD="your_db_password"
export DB_NAME="harvard_artifacts"
```

**Requirements** (typical):
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### API Setup

1. Obtain API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Store in environment variable or `.env` file:

```python
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = 'https://api.harvardartmuseums.org/object'
```

### Database Schema

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    department VARCHAR(200),
    classification VARCHAR(200),
    dated VARCHAR(200),
    accession_number VARCHAR(100),
    url TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    base_image_url TEXT,
    image_count INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percentage DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Components

### 1. API Data Extraction

```python
import requests
import time

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    params = {
        'apikey': api_key,
        'page': page,
        'size': size
    }
    
    response = requests.get(BASE_URL, params=params)
    
    if response.status_code == 200:
        data = response.json()
        return data.get('records', []), data.get('info', {})
    else:
        raise Exception(f"API Error: {response.status_code}")

def collect_all_artifacts(api_key, max_records=1000):
    """
    Collect artifacts with rate limiting and pagination
    """
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        records, info = fetch_artifacts(api_key, page=page, size=size)
        all_artifacts.extend(records)
        
        # Rate limiting
        time.sleep(0.5)
        
        if len(records) < size:
            break
        
        page += 1
    
    return all_artifacts[:max_records]
```

### 2. ETL Transformation

```python
import pandas as pd

def transform_artifacts(raw_data):
    """
    Transform nested JSON into relational DataFrames
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
            'century': artifact.get('century'),
            'department': artifact.get('department'),
            'classification': artifact.get('classification'),
            'dated': artifact.get('dated'),
            'accession_number': artifact.get('accessionnumber'),
            'url': artifact.get('url')
        }
        metadata_list.append(metadata)
        
        # Extract media
        if artifact.get('images'):
            media = {
                'artifact_id': artifact.get('id'),
                'base_image_url': artifact.get('primaryimageurl'),
                'image_count': len(artifact.get('images', []))
            }
            media_list.append(media)
        
        # Extract colors
        for color_obj in artifact.get('colors', []):
            color = {
                'artifact_id': artifact.get('id'),
                'color': color_obj.get('color'),
                'spectrum': color_obj.get('spectrum'),
                'percentage': color_obj.get('percent')
            }
            colors_list.append(color)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### 3. Database Loading

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

def batch_insert_metadata(df_metadata):
    """
    Batch insert artifact metadata
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, department, classification, dated, accession_number, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    values = [tuple(row) for row in df_metadata.values]
    
    try:
        cursor.executemany(insert_query, values)
        conn.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
    except Error as e:
        print(f"Error: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()

def batch_insert_colors(df_colors):
    """
    Batch insert color data
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactcolors (artifact_id, color, spectrum, percentage)
        VALUES (%s, %s, %s, %s)
    """
    
    values = [tuple(row) for row in df_colors.values]
    
    try:
        cursor.executemany(insert_query, values)
        conn.commit()
        print(f"Inserted {cursor.rowcount} color records")
    except Error as e:
        print(f"Error: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()
```

### 4. Analytics Queries

```python
def get_artifacts_by_century():
    """
    Analytical query: Count artifacts by century
    """
    query = """
        SELECT century, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY artifact_count DESC
        LIMIT 20
    """
    
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df

def get_color_distribution():
    """
    Analytical query: Color usage across artifacts
    """
    query = """
        SELECT c.spectrum, COUNT(DISTINCT c.artifact_id) as artifact_count,
               AVG(c.percentage) as avg_percentage
        FROM artifactcolors c
        GROUP BY c.spectrum
        ORDER BY artifact_count DESC
    """
    
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df

def get_department_statistics():
    """
    Analytical query: Department-wise artifact distribution
    """
    query = """
        SELECT m.department, 
               COUNT(DISTINCT m.id) as total_artifacts,
               COUNT(DISTINCT med.media_id) as artifacts_with_media
        FROM artifactmetadata m
        LEFT JOIN artifactmedia med ON m.id = med.artifact_id
        WHERE m.department IS NOT NULL
        GROUP BY m.department
        ORDER BY total_artifacts DESC
    """
    
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for query selection
    query_options = {
        "Artifacts by Century": get_artifacts_by_century,
        "Color Distribution": get_color_distribution,
        "Department Statistics": get_department_statistics
    }
    
    selected_query = st.sidebar.selectbox(
        "Select Analysis",
        list(query_options.keys())
    )
    
    # Execute query
    if st.button("Run Analysis"):
        with st.spinner("Executing query..."):
            df_result = query_options[selected_query]()
            
            # Display results
            st.subheader("Query Results")
            st.dataframe(df_result)
            
            # Visualization
            if len(df_result) > 0:
                st.subheader("Visualization")
                
                if selected_query == "Artifacts by Century":
                    fig = px.bar(
                        df_result,
                        x='century',
                        y='artifact_count',
                        title='Artifact Distribution by Century'
                    )
                elif selected_query == "Color Distribution":
                    fig = px.bar(
                        df_result,
                        x='spectrum',
                        y='artifact_count',
                        title='Color Spectrum Distribution'
                    )
                else:
                    fig = px.bar(
                        df_result,
                        x='department',
                        y='total_artifacts',
                        title='Artifacts by Department'
                    )
                
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Complete ETL Pipeline

```python
def run_etl_pipeline(api_key, max_records=1000):
    """
    Execute complete ETL pipeline
    """
    # Extract
    print("Extracting data from API...")
    raw_artifacts = collect_all_artifacts(api_key, max_records)
    
    # Transform
    print("Transforming data...")
    df_metadata, df_media, df_colors = transform_artifacts(raw_artifacts)
    
    # Load
    print("Loading data to database...")
    batch_insert_metadata(df_metadata)
    batch_insert_colors(df_colors)
    
    print("ETL pipeline completed successfully!")
    
    return {
        'metadata_count': len(df_metadata),
        'media_count': len(df_media),
        'colors_count': len(df_colors)
    }
```

### Error Handling and Retry Logic

```python
import time
from functools import wraps

def retry_on_failure(max_retries=3, delay=2):
    """
    Decorator for retry logic
    """
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_retries - 1:
                        raise
                    print(f"Attempt {attempt + 1} failed: {e}. Retrying...")
                    time.sleep(delay)
        return wrapper
    return decorator

@retry_on_failure(max_retries=3)
def fetch_with_retry(api_key, page):
    """
    Fetch with automatic retry
    """
    return fetch_artifacts(api_key, page)
```

## Troubleshooting

### API Rate Limiting
```python
# Implement exponential backoff
def fetch_with_backoff(api_key, page, max_wait=60):
    wait_time = 1
    while wait_time <= max_wait:
        try:
            return fetch_artifacts(api_key, page)
        except Exception as e:
            if "429" in str(e):  # Rate limit error
                time.sleep(wait_time)
                wait_time *= 2
            else:
                raise
```

### Database Connection Issues
```python
# Test connection before operations
def test_db_connection():
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        cursor.close()
        conn.close()
        return True
    except Error as e:
        print(f"Database connection failed: {e}")
        return False
```

### Handling Missing Data
```python
# Clean and validate data
def clean_metadata(df):
    """
    Handle null values and data quality issues
    """
    df['title'] = df['title'].fillna('Untitled')
    df['culture'] = df['culture'].fillna('Unknown')
    df['century'] = df['century'].fillna('Unknown Period')
    
    # Remove duplicates
    df = df.drop_duplicates(subset=['id'])
    
    return df
```

This skill provides everything needed to build, deploy, and extend ETL pipelines for museum artifact data using modern Python data engineering tools.
