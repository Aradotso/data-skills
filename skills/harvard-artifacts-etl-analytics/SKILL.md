---
name: harvard-artifacts-etl-analytics
description: End-to-end ETL pipeline and analytics for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for museum artifacts
  - integrate Harvard Art Museums API
  - create artifact data analytics dashboard
  - set up museum collection data engineering
  - query Harvard artifacts with SQL
  - visualize art museum data with Streamlit
  - build end-to-end data pipeline for artifacts
  - analyze art collection metadata
---

# Harvard Artifacts Collection Data Engineering & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides a complete data engineering solution for the Harvard Art Museums API. It demonstrates:

- **API Integration**: Fetching artifact data from Harvard Art Museums with pagination and rate limiting
- **ETL Pipeline**: Extracting, transforming, and loading artifact metadata, media, and color data
- **SQL Storage**: Relational database design with proper foreign key relationships
- **Analytics**: 20+ predefined SQL queries for artifact insights
- **Visualization**: Interactive Streamlit dashboards with Plotly charts

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

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

### API Key Setup

1. Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Create a `.env` file or configure in Streamlit secrets:

```python
# .env file
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_db_host
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Setup

The application uses MySQL or TiDB Cloud. Create three tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    period VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dated VARCHAR(200),
    division VARCHAR(200)
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(1000),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py
```

The app will launch on `http://localhost:8501`

## Core Components

### 1. API Data Extraction

```python
import requests
import os

def fetch_artifacts(api_key, size=100, page=1):
    """
    Fetch artifacts from Harvard Art Museums API
    
    Args:
        api_key: Harvard API key
        size: Number of records per page (max 100)
        page: Page number for pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'size': size,
        'page': page,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    return response.json()

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, size=100, page=1)
artifacts = data['records']
total_records = data['info']['totalrecords']
```

### 2. ETL Pipeline

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """
    Transform raw API data into structured metadata
    """
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'period': artifact.get('period'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dated': artifact.get('dated'),
            'division': artifact.get('division')
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
        
        for image in images:
            media_list.append({
                'artifact_id': artifact_id,
                'image_url': image.get('baseimageurl')
            })
    
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
            color_list.append({
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'percentage': color.get('percent')
            })
    
    return pd.DataFrame(color_list)
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_to_sql(df, table_name, connection):
    """
    Batch insert DataFrame to SQL table
    
    Args:
        df: Pandas DataFrame
        table_name: Target table name
        connection: MySQL connection object
    """
    cursor = connection.cursor()
    
    # Prepare column names and placeholders
    cols = ','.join(df.columns)
    placeholders = ','.join(['%s'] * len(df.columns))
    
    insert_query = f"INSERT INTO {table_name} ({cols}) VALUES ({placeholders})"
    
    # Batch insert
    data_tuples = [tuple(x) for x in df.to_numpy()]
    cursor.executemany(insert_query, data_tuples)
    connection.commit()
    cursor.close()
    
    return cursor.rowcount

# Usage
conn = get_db_connection()
metadata_df = transform_artifact_metadata(artifacts)
rows_inserted = load_to_sql(metadata_df, 'artifactmetadata', conn)
conn.close()
```

### 4. Analytics Queries

```python
def execute_query(query, connection):
    """
    Execute SQL query and return results as DataFrame
    """
    cursor = connection.cursor(dictionary=True)
    cursor.execute(query)
    results = cursor.fetchall()
    cursor.close()
    return pd.DataFrame(results)

# Sample analytical queries
QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "top_colors": """
        SELECT color, SUM(percentage) as total_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY total_percentage DESC
        LIMIT 10
    """,
    
    "artifacts_with_images": """
        SELECT 
            CASE WHEN image_count > 0 THEN 'With Images' ELSE 'No Images' END as category,
            COUNT(*) as count
        FROM (
            SELECT m.id, COUNT(a.id) as image_count
            FROM artifactmetadata m
            LEFT JOIN artifactmedia a ON m.id = a.artifact_id
            GROUP BY m.id
        ) as subquery
        GROUP BY category
    """,
    
    "department_distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """
}

# Execute query
conn = get_db_connection()
df_results = execute_query(QUERIES['artifacts_by_culture'], conn)
conn.close()
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Artifacts Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Options")
    query_name = st.sidebar.selectbox(
        "Select Analysis",
        list(QUERIES.keys())
    )
    
    # Execute query
    if st.button("Run Analysis"):
        with st.spinner("Executing query..."):
            conn = get_db_connection()
            df = execute_query(QUERIES[query_name], conn)
            conn.close()
            
            # Display results
            st.subheader("Query Results")
            st.dataframe(df)
            
            # Visualization
            if len(df) > 0 and len(df.columns) >= 2:
                st.subheader("Visualization")
                fig = px.bar(
                    df,
                    x=df.columns[0],
                    y=df.columns[1],
                    title=f"{query_name.replace('_', ' ').title()}"
                )
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Pagination for Large Datasets

```python
def fetch_all_artifacts(api_key, total_records=1000):
    """
    Fetch multiple pages of artifacts
    """
    all_artifacts = []
    page_size = 100
    total_pages = (total_records + page_size - 1) // page_size
    
    for page in range(1, total_pages + 1):
        data = fetch_artifacts(api_key, size=page_size, page=page)
        all_artifacts.extend(data['records'])
        
        # Rate limiting
        time.sleep(0.5)
    
    return all_artifacts
```

### Error Handling

```python
def safe_etl_pipeline(api_key, batch_size=100):
    """
    ETL with error handling
    """
    try:
        # Extract
        artifacts = fetch_artifacts(api_key, size=batch_size)
        
        # Transform
        metadata_df = transform_artifact_metadata(artifacts)
        media_df = transform_artifact_media(artifacts)
        colors_df = transform_artifact_colors(artifacts)
        
        # Load
        conn = get_db_connection()
        load_to_sql(metadata_df, 'artifactmetadata', conn)
        load_to_sql(media_df, 'artifactmedia', conn)
        load_to_sql(colors_df, 'artifactcolors', conn)
        conn.close()
        
        return True
        
    except requests.RequestException as e:
        st.error(f"API Error: {e}")
        return False
    except Error as e:
        st.error(f"Database Error: {e}")
        return False
    except Exception as e:
        st.error(f"Unexpected Error: {e}")
        return False
```

## Troubleshooting

### API Rate Limits

If you encounter 429 errors:

```python
import time

def fetch_with_retry(api_key, max_retries=3):
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key)
        except requests.HTTPError as e:
            if e.response.status_code == 429:
                wait_time = 2 ** attempt
                time.sleep(wait_time)
            else:
                raise
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
        return True
    except Error as e:
        st.error(f"Cannot connect to database: {e}")
        return False
```

### Missing Data Handling

```python
def safe_get(artifact, key, default=None):
    """Safely extract nested values"""
    return artifact.get(key, default) or default

# In transformation
'culture': safe_get(artifact, 'culture', 'Unknown'),
```

## Advanced Usage

### Custom Query Builder

```python
def build_dynamic_query(filters):
    """
    Build SQL query from user filters
    """
    query = "SELECT * FROM artifactmetadata WHERE 1=1"
    
    if filters.get('culture'):
        query += f" AND culture = '{filters['culture']}'"
    
    if filters.get('century'):
        query += f" AND century = '{filters['century']}'"
    
    if filters.get('department'):
        query += f" AND department = '{filters['department']}'"
    
    return query
```

### Export Results

```python
def export_to_csv(df, filename):
    """Export DataFrame to CSV"""
    df.to_csv(filename, index=False)
    st.success(f"Data exported to {filename}")

# In Streamlit
if st.button("Export Results"):
    export_to_csv(df, "artifacts_export.csv")
```
