---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline with the Harvard Art Museums API
  - create a data pipeline for museum artifact analytics
  - set up Harvard Art Museums data engineering project
  - build a Streamlit dashboard for artifact data
  - extract and transform Harvard museum collection data
  - create SQL analytics for art museum artifacts
  - integrate Harvard Art Museums API with database
  - visualize museum artifact data with Plotly
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualization using Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering & Analytics App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color data into relational databases
- **SQL Analytics**: Pre-built analytical queries for artifact insights
- **Interactive Dashboards**: Streamlit-based UI with Plotly visualizations
- **Database Support**: MySQL and TiDB Cloud integration

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

Create a `.env` file in the project root:

```env
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_username
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Setup

Create the required tables:

```sql
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    description TEXT,
    technique VARCHAR(500),
    division VARCHAR(255)
);

CREATE TABLE artifactmedia (
    artifact_id INT,
    media_type VARCHAR(100),
    base_url VARCHAR(500),
    format VARCHAR(50),
    width INT,
    height INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
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

Extract artifacts from Harvard Art Museums API:

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
    url = f"https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    response.raise_for_status()
    return response.json()

def collect_multiple_pages(num_pages=5):
    """Collect artifacts from multiple pages"""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(page=page)
        all_artifacts.extend(data.get('records', []))
    
    return all_artifacts
```

### 2. ETL Pipeline

Transform and load data into SQL database:

```python
import mysql.connector
import pandas as pd
from typing import List, Dict

def create_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def transform_artifact_metadata(artifacts: List[Dict]) -> pd.DataFrame:
    """Transform artifact data into metadata DataFrame"""
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'period': artifact.get('period', '')[:255],
            'century': artifact.get('century', '')[:255],
            'classification': artifact.get('classification', '')[:255],
            'department': artifact.get('department', '')[:255],
            'dated': artifact.get('dated', '')[:255],
            'description': artifact.get('description', ''),
            'technique': artifact.get('technique', '')[:500],
            'division': artifact.get('division', '')[:255]
        })
    
    return pd.DataFrame(metadata)

def transform_artifact_media(artifacts: List[Dict]) -> pd.DataFrame:
    """Transform media data into DataFrame"""
    media_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for image in images:
            media_records.append({
                'artifact_id': artifact_id,
                'media_type': 'image',
                'base_url': image.get('baseimageurl', ''),
                'format': image.get('format', ''),
                'width': image.get('width'),
                'height': image.get('height')
            })
    
    return pd.DataFrame(media_records)

def transform_artifact_colors(artifacts: List[Dict]) -> pd.DataFrame:
    """Transform color data into DataFrame"""
    color_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_records.append({
                'artifact_id': artifact_id,
                'color': color.get('color', ''),
                'spectrum': color.get('spectrum', ''),
                'hue': color.get('hue', ''),
                'percent': color.get('percent', 0.0)
            })
    
    return pd.DataFrame(color_records)

def load_data_to_db(df: pd.DataFrame, table_name: str, if_exists='append'):
    """Load DataFrame to database table"""
    conn = create_db_connection()
    cursor = conn.cursor()
    
    # Convert DataFrame to list of tuples
    cols = ','.join(df.columns)
    placeholders = ','.join(['%s'] * len(df.columns))
    
    if if_exists == 'replace':
        cursor.execute(f"TRUNCATE TABLE {table_name}")
    
    insert_query = f"INSERT INTO {table_name} ({cols}) VALUES ({placeholders})"
    
    # Batch insert
    values = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, values)
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print(f"Loaded {len(df)} records into {table_name}")

def run_etl_pipeline(num_pages=5):
    """Run complete ETL pipeline"""
    # Extract
    print("Extracting artifacts...")
    artifacts = collect_multiple_pages(num_pages)
    
    # Transform
    print("Transforming data...")
    metadata_df = transform_artifact_metadata(artifacts)
    media_df = transform_artifact_media(artifacts)
    colors_df = transform_artifact_colors(artifacts)
    
    # Load
    print("Loading to database...")
    load_data_to_db(metadata_df, 'artifactmetadata', if_exists='replace')
    load_data_to_db(media_df, 'artifactmedia', if_exists='replace')
    load_data_to_db(colors_df, 'artifactcolors', if_exists='replace')
    
    print("ETL pipeline completed!")
```

### 3. SQL Analytics Queries

Common analytical queries:

```python
# Query 1: Artifacts by culture
query_by_culture = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL AND culture != ''
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 20
"""

# Query 2: Artifacts by century
query_by_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC
"""

# Query 3: Top colors used
query_top_colors = """
SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
FROM artifactcolors
GROUP BY color
ORDER BY usage_count DESC
LIMIT 15
"""

# Query 4: Media availability
query_media_stats = """
SELECT 
    COUNT(DISTINCT am.id) as total_artifacts,
    COUNT(DISTINCT med.artifact_id) as artifacts_with_media,
    ROUND(COUNT(DISTINCT med.artifact_id) * 100.0 / COUNT(DISTINCT am.id), 2) as media_percentage
FROM artifactmetadata am
LEFT JOIN artifactmedia med ON am.id = med.artifact_id
"""

# Query 5: Artifacts by department
query_by_department = """
SELECT department, COUNT(*) as count
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY count DESC
"""

def execute_query(query: str) -> pd.DataFrame:
    """Execute SQL query and return DataFrame"""
    conn = create_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

### 4. Streamlit Dashboard

Build interactive dashboard:

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Choose Analysis",
        ["ETL Pipeline", "Analytics Queries", "Visualizations"]
    )
    
    if page == "ETL Pipeline":
        st.header("ETL Pipeline Management")
        
        num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=20, value=5)
        
        if st.button("Run ETL Pipeline"):
            with st.spinner("Running ETL pipeline..."):
                try:
                    run_etl_pipeline(num_pages)
                    st.success("ETL pipeline completed successfully!")
                except Exception as e:
                    st.error(f"Error: {str(e)}")
    
    elif page == "Analytics Queries":
        st.header("SQL Analytics")
        
        query_options = {
            "Artifacts by Culture": query_by_culture,
            "Artifacts by Century": query_by_century,
            "Top Colors": query_top_colors,
            "Media Statistics": query_media_stats,
            "Department Distribution": query_by_department
        }
        
        selected_query = st.selectbox("Select Query", list(query_options.keys()))
        
        if st.button("Execute Query"):
            with st.spinner("Executing query..."):
                df = execute_query(query_options[selected_query])
                st.dataframe(df)
                
                # Auto-generate visualization
                if len(df.columns) == 2 and df.shape[0] > 0:
                    fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                                title=selected_query)
                    st.plotly_chart(fig, use_container_width=True)
    
    elif page == "Visualizations":
        st.header("Interactive Visualizations")
        
        # Color distribution
        st.subheader("Color Distribution in Artifacts")
        color_df = execute_query(query_top_colors)
        fig = px.pie(color_df, values='usage_count', names='color', 
                    title='Top Colors in Collection')
        st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Batch Processing with Rate Limiting

```python
import time

def batch_fetch_artifacts(total_pages=10, batch_size=5, delay=1):
    """Fetch artifacts in batches to respect rate limits"""
    all_artifacts = []
    
    for batch_start in range(1, total_pages + 1, batch_size):
        batch_end = min(batch_start + batch_size, total_pages + 1)
        
        for page in range(batch_start, batch_end):
            artifacts = fetch_artifacts(page=page)
            all_artifacts.extend(artifacts.get('records', []))
            time.sleep(delay)  # Rate limiting
        
        print(f"Completed batch {batch_start}-{batch_end-1}")
    
    return all_artifacts
```

### Error Handling in ETL

```python
def safe_etl_pipeline(num_pages=5):
    """ETL pipeline with error handling"""
    try:
        artifacts = collect_multiple_pages(num_pages)
        
        if not artifacts:
            raise ValueError("No artifacts fetched")
        
        metadata_df = transform_artifact_metadata(artifacts)
        
        if metadata_df.empty:
            raise ValueError("Transformation resulted in empty DataFrame")
        
        load_data_to_db(metadata_df, 'artifactmetadata')
        
        return {"status": "success", "records": len(artifacts)}
        
    except requests.RequestException as e:
        return {"status": "error", "message": f"API Error: {str(e)}"}
    except mysql.connector.Error as e:
        return {"status": "error", "message": f"Database Error: {str(e)}"}
    except Exception as e:
        return {"status": "error", "message": f"Unexpected Error: {str(e)}"}
```

## Troubleshooting

### API Key Issues
- Ensure `HARVARD_API_KEY` is set in `.env`
- Get API key from: https://www.harvardartmuseums.org/collections/api
- Verify key is valid: `curl "https://api.harvardartmuseums.org/object?apikey=YOUR_KEY&size=1"`

### Database Connection Errors
- Check database credentials in `.env`
- Verify database server is running
- Ensure firewall allows connection to DB_PORT

### Empty Results
- API may return no results for certain filters
- Check pagination parameters (page, size)
- Verify `hasimage=1` filter if you need images

### Memory Issues with Large Datasets
- Process in smaller batches
- Use chunked inserts: `cursor.executemany()` with batch size
- Clear DataFrames after loading: `del df; gc.collect()`
