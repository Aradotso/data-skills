---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do i build an etl pipeline with harvard art museums api
  - create analytics dashboard for museum artifact data
  - extract and analyze harvard art museums collection
  - set up streamlit app for museum data visualization
  - build data engineering pipeline for art museum artifacts
  - analyze harvard art collection with sql queries
  - create museum artifact analytics with python
  - visualize art museum data with plotly and streamlit
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL analytics, and interactive data visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color data into SQL databases
- **SQL Analytics**: Run predefined analytical queries on structured artifact data
- **Visualization**: Create interactive dashboards using Streamlit and Plotly
- **Database Design**: Relational schema with proper foreign key relationships

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

### Required Dependencies

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

Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

### 2. Environment Variables

Create a `.env` file in the project root:

```bash
# Harvard API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### 3. Database Setup

Connect to MySQL or TiDB Cloud and create the database:

```sql
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    technique VARCHAR(300),
    medium VARCHAR(300),
    dimensions VARCHAR(300),
    creditline TEXT,
    accession_number VARCHAR(100),
    url VARCHAR(500)
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(500),
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Components

### API Data Collection

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = 'https://api.harvardartmuseums.org/object'
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    return response.json()

# Fetch first 100 artifacts
data = fetch_artifacts(page=1, size=100)
artifacts = data.get('records', [])
total_records = data.get('info', {}).get('totalrecords', 0)
```

### ETL Pipeline - Extract and Transform

```python
import pandas as pd

def transform_artifact_data(artifacts):
    """Transform raw API data into structured format"""
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'accession_number': artifact.get('accessionyear'),
            'url': artifact.get('url')
        }
        metadata_records.append(metadata)
        
        # Extract media/images
        images = artifact.get('images', [])
        for img in images:
            media_records.append({
                'artifact_id': artifact.get('id'),
                'image_url': img.get('baseimageurl'),
                'media_type': 'image'
            })
        
        # Extract colors
        colors = artifact.get('colors', [])
        for color in colors:
            color_records.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'percentage': color.get('percent')
            })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )

# Transform data
metadata_df, media_df, colors_df = transform_artifact_data(artifacts)
```

### ETL Pipeline - Load to Database

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

def load_to_database(metadata_df, media_df, colors_df):
    """Load transformed data into SQL database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Insert metadata (batch insert)
        metadata_query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, period, century, classification, 
             department, technique, medium, dimensions, creditline, 
             accession_number, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_query, metadata_df.values.tolist())
        
        # Insert media
        media_query = """
            INSERT INTO artifactmedia (artifact_id, image_url, media_type)
            VALUES (%s, %s, %s)
        """
        cursor.executemany(media_query, media_df.values.tolist())
        
        # Insert colors
        colors_query = """
            INSERT INTO artifactcolors (artifact_id, color, spectrum, percentage)
            VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(colors_query, colors_df.values.tolist())
        
        conn.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Error: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()

# Load data
load_to_database(metadata_df, media_df, colors_df)
```

### Analytics Queries

```python
def run_analytical_query(query_name):
    """Execute predefined analytical queries"""
    queries = {
        'artifacts_by_culture': """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture IS NOT NULL
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 10
        """,
        'artifacts_by_century': """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL
            GROUP BY century
            ORDER BY count DESC
        """,
        'top_colors': """
            SELECT color, COUNT(*) as frequency
            FROM artifactcolors
            GROUP BY color
            ORDER BY frequency DESC
            LIMIT 15
        """,
        'media_availability': """
            SELECT 
                CASE WHEN m.artifact_id IS NOT NULL THEN 'With Media' ELSE 'No Media' END as media_status,
                COUNT(DISTINCT a.id) as count
            FROM artifactmetadata a
            LEFT JOIN artifactmedia m ON a.id = m.artifact_id
            GROUP BY media_status
        """,
        'artifacts_by_department': """
            SELECT department, COUNT(*) as count
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY count DESC
        """
    }
    
    conn = get_db_connection()
    cursor = conn.cursor(dictionary=True)
    
    cursor.execute(queries[query_name])
    results = cursor.fetchall()
    
    cursor.close()
    conn.close()
    
    return pd.DataFrame(results)

# Run analytics
culture_data = run_analytical_query('artifacts_by_culture')
print(culture_data)
```

### Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    st.markdown("End-to-end ETL and Analytics Application")
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Queries")
    query_options = {
        "Artifacts by Culture": "artifacts_by_culture",
        "Artifacts by Century": "artifacts_by_century",
        "Top Colors Used": "top_colors",
        "Media Availability": "media_availability",
        "Artifacts by Department": "artifacts_by_department"
    }
    
    selected_query = st.sidebar.selectbox(
        "Select Analysis",
        list(query_options.keys())
    )
    
    # Execute query
    if st.sidebar.button("Run Analysis"):
        with st.spinner("Analyzing data..."):
            df = run_analytical_query(query_options[selected_query])
            
            # Display results
            st.subheader(f"Results: {selected_query}")
            st.dataframe(df)
            
            # Visualization
            if len(df.columns) == 2:
                fig = px.bar(
                    df,
                    x=df.columns[0],
                    y=df.columns[1],
                    title=selected_query,
                    labels={df.columns[0]: df.columns[0].title(), 
                           df.columns[1]: 'Count'}
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

### Pagination for Large Datasets

```python
def fetch_all_artifacts(max_pages=10):
    """Fetch multiple pages of artifacts"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        data = fetch_artifacts(page=page, size=100)
        artifacts = data.get('records', [])
        all_artifacts.extend(artifacts)
        
        # Respect rate limits
        import time
        time.sleep(0.5)
    
    return all_artifacts
```

### Incremental Data Updates

```python
def get_latest_artifact_id():
    """Get the most recent artifact ID in database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    conn.close()
    return result[0] if result[0] else 0

def incremental_etl():
    """Only fetch new artifacts"""
    latest_id = get_latest_artifact_id()
    # Implement logic to fetch only artifacts with id > latest_id
```

## Troubleshooting

### API Rate Limiting
- Add delays between requests: `time.sleep(0.5)`
- Implement exponential backoff for failed requests
- Cache API responses locally

### Database Connection Issues
- Verify environment variables are loaded correctly
- Check firewall rules for TiDB Cloud or remote MySQL
- Use connection pooling for better performance

### Memory Issues with Large Datasets
- Process data in batches
- Use chunked DataFrame operations
- Clear DataFrames after loading to database

### Missing or Null Data
- Always check for `None` values before inserting
- Use `ON DUPLICATE KEY UPDATE` for upserts
- Handle missing nested fields with `.get()` method
