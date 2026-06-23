---
name: harvard-art-museums-data-pipeline
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build data pipeline for harvard art museums
  - create etl workflow for museum artifacts
  - analyze harvard art collection with sql
  - set up streamlit dashboard for art data
  - extract transform load museum api data
  - query harvard artifacts database
  - visualize museum collection analytics
  - configure tidb cloud for art data
---

# Harvard Art Museums Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering pipelines using the Harvard Art Museums API. The project demonstrates ETL workflows, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection Data Engineering & Analytics App:
- Fetches artifact data from the Harvard Art Museums API with pagination
- Transforms nested JSON into relational database tables
- Loads data into MySQL/TiDB Cloud with proper schema relationships
- Executes analytical SQL queries for insights
- Visualizes results through interactive Streamlit dashboards with Plotly

**Architecture Flow**: API → Extract → Transform → Load (SQL) → Analytics → Visualization

## Installation

### Prerequisites
```bash
# Install required packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Environment Setup
Create a `.env` file with required credentials:
```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration (MySQL or TiDB Cloud)
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Database Schema Setup
```sql
-- Create artifacts metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255),
    period VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accessionyear INT,
    provenance TEXT,
    copyright TEXT,
    url VARCHAR(500)
);

-- Create media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    iiifbaseuri VARCHAR(500),
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Create colors table
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

### 1. API Data Extraction

```python
import requests
import os

def fetch_artifacts(api_key, size=100, page=1):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    
    Args:
        api_key: Harvard API key from environment
        size: Number of records per page (max 100)
        page: Page number for pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'size': size,
        'page': page,
        'fields': 'id,title,culture,century,classification,department,division,dated,period,technique,medium,dimensions,creditline,accessionyear,provenance,copyright,url,primaryimageurl,baseimageurl,images,colors'
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    return response.json()

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, size=100, page=1)
artifacts = data.get('records', [])
total_records = data.get('info', {}).get('totalrecords', 0)
```

### 2. ETL Transform Functions

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """Transform artifact data into metadata DataFrame"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata_list.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:255],
            'department': artifact.get('department', '')[:255],
            'division': artifact.get('division', '')[:255],
            'dated': artifact.get('dated', '')[:255],
            'period': artifact.get('period', '')[:255],
            'technique': artifact.get('technique', '')[:500],
            'medium': artifact.get('medium', '')[:500],
            'dimensions': artifact.get('dimensions', '')[:500],
            'creditline': artifact.get('creditline', ''),
            'accessionyear': artifact.get('accessionyear'),
            'provenance': artifact.get('provenance', ''),
            'copyright': artifact.get('copyright', ''),
            'url': artifact.get('url', '')[:500]
        })
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(artifacts):
    """Transform nested media/image data"""
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        if images:
            for img in images:
                media_list.append({
                    'artifact_id': artifact_id,
                    'iiifbaseuri': img.get('iiifbaseuri', '')[:500],
                    'baseimageurl': img.get('baseimageurl', '')[:500],
                    'primaryimageurl': artifact.get('primaryimageurl', '')[:500]
                })
        else:
            media_list.append({
                'artifact_id': artifact_id,
                'iiifbaseuri': '',
                'baseimageurl': '',
                'primaryimageurl': artifact.get('primaryimageurl', '')[:500]
            })
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(artifacts):
    """Transform color data into separate table"""
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_list.append({
                'artifact_id': artifact_id,
                'color': color.get('color', '')[:50],
                'spectrum': color.get('spectrum', '')[:50],
                'hue': color.get('hue', '')[:50],
                'percent': color.get('percent', 0.0)
            })
    
    return pd.DataFrame(color_list)
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def create_db_connection():
    """Create database connection using environment variables"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=os.getenv('DB_PORT', 3306),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def batch_insert_metadata(df_metadata, connection):
    """Batch insert artifact metadata with error handling"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, classification, department, division, 
     dated, period, technique, medium, dimensions, creditline, 
     accessionyear, provenance, copyright, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture), century=VALUES(century)
    """
    
    data_tuples = [tuple(row) for row in df_metadata.values]
    
    try:
        cursor.executemany(insert_query, data_tuples)
        connection.commit()
        return cursor.rowcount
    except Error as e:
        connection.rollback()
        print(f"Insert error: {e}")
        return 0
    finally:
        cursor.close()
```

### 4. Analytical SQL Queries

```python
def execute_analytical_query(query_name, connection):
    """Execute predefined analytical queries"""
    
    queries = {
        "artifacts_by_culture": """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture IS NOT NULL AND culture != ''
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 20
        """,
        
        "artifacts_by_century": """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL AND century != ''
            GROUP BY century
            ORDER BY count DESC
        """,
        
        "top_departments": """
            SELECT department, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY artifact_count DESC
            LIMIT 15
        """,
        
        "media_availability": """
            SELECT 
                CASE 
                    WHEN primaryimageurl IS NOT NULL AND primaryimageurl != '' THEN 'Has Image'
                    ELSE 'No Image'
                END as image_status,
                COUNT(*) as count
            FROM artifactmedia
            GROUP BY image_status
        """,
        
        "color_distribution": """
            SELECT spectrum, COUNT(*) as count, AVG(percent) as avg_percent
            FROM artifactcolors
            WHERE spectrum IS NOT NULL
            GROUP BY spectrum
            ORDER BY count DESC
        """,
        
        "artifacts_with_provenance": """
            SELECT 
                CASE 
                    WHEN provenance IS NOT NULL AND provenance != '' THEN 'Has Provenance'
                    ELSE 'No Provenance'
                END as provenance_status,
                COUNT(*) as count
            FROM artifactmetadata
            GROUP BY provenance_status
        """
    }
    
    cursor = connection.cursor(dictionary=True)
    cursor.execute(queries[query_name])
    results = cursor.fetchall()
    cursor.close()
    
    return pd.DataFrame(results)
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Art Analytics", layout="wide")
    st.title("🏛️ Harvard Art Museums Data Analytics")
    
    # Sidebar for configuration
    st.sidebar.header("Configuration")
    
    # Data collection section
    with st.expander("📥 Data Collection"):
        num_pages = st.number_input("Pages to fetch (100 records/page)", 
                                     min_value=1, max_value=10, value=1)
        
        if st.button("Fetch & Load Data"):
            with st.spinner("Fetching data from API..."):
                api_key = os.getenv('HARVARD_API_KEY')
                all_artifacts = []
                
                for page in range(1, num_pages + 1):
                    data = fetch_artifacts(api_key, page=page)
                    all_artifacts.extend(data.get('records', []))
                
                # Transform
                df_metadata = transform_artifact_metadata(all_artifacts)
                df_media = transform_artifact_media(all_artifacts)
                df_colors = transform_artifact_colors(all_artifacts)
                
                # Load
                conn = create_db_connection()
                if conn:
                    rows = batch_insert_metadata(df_metadata, conn)
                    st.success(f"✅ Loaded {rows} artifacts")
                    conn.close()
    
    # Analytics section
    st.header("📊 Analytics Dashboard")
    
    query_options = {
        "Artifacts by Culture": "artifacts_by_culture",
        "Artifacts by Century": "artifacts_by_century",
        "Top Departments": "top_departments",
        "Media Availability": "media_availability",
        "Color Distribution": "color_distribution",
        "Provenance Status": "artifacts_with_provenance"
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
    
    if st.button("Run Query"):
        conn = create_db_connection()
        if conn:
            df_result = execute_analytical_query(
                query_options[selected_query], 
                conn
            )
            
            # Display results
            st.dataframe(df_result, use_container_width=True)
            
            # Visualize
            if len(df_result) > 0:
                x_col = df_result.columns[0]
                y_col = df_result.columns[1]
                
                fig = px.bar(df_result, x=x_col, y=y_col, 
                            title=selected_query,
                            color=y_col,
                            color_continuous_scale='Viridis')
                
                st.plotly_chart(fig, use_container_width=True)
            
            conn.close()

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Set environment variables
export HARVARD_API_KEY="your_key"
export DB_HOST="your_host"
export DB_USER="your_user"
export DB_PASSWORD="your_password"
export DB_NAME="harvard_artifacts"

# Run Streamlit app
streamlit run app.py
```

## Common Patterns

### Pagination Handling
```python
def fetch_all_artifacts(api_key, total_pages=5):
    """Fetch multiple pages with rate limiting"""
    import time
    
    all_artifacts = []
    for page in range(1, total_pages + 1):
        data = fetch_artifacts(api_key, page=page)
        all_artifacts.extend(data.get('records', []))
        time.sleep(0.5)  # Rate limiting
    
    return all_artifacts
```

### Error Handling in ETL
```python
def safe_transform(artifacts):
    """Transform with error handling for missing fields"""
    try:
        df = transform_artifact_metadata(artifacts)
        # Replace NaN with None for SQL compatibility
        df = df.where(pd.notnull(df), None)
        return df
    except Exception as e:
        st.error(f"Transform error: {e}")
        return pd.DataFrame()
```

## Troubleshooting

**API Key Issues**: Ensure `HARVARD_API_KEY` is set in `.env` and the key is valid from https://hvrd.art/o/305239

**Database Connection Failures**: Verify TiDB Cloud or MySQL credentials, check firewall rules, ensure database exists

**Rate Limiting**: Harvard API has rate limits; add `time.sleep()` between requests

**Data Type Errors**: Ensure string fields are truncated to match VARCHAR limits in schema

**Foreign Key Violations**: Always insert into `artifactmetadata` before `artifactmedia` or `artifactcolors`
