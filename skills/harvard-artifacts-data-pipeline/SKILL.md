---
name: harvard-artifacts-data-pipeline
description: Build end-to-end data engineering pipelines using Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - how do I build a data pipeline with Harvard Art Museums API
  - set up ETL pipeline for museum artifact data
  - create analytics dashboard for Harvard art collection
  - integrate Harvard Art Museums API with SQL database
  - build Streamlit app for artifact data visualization
  - extract and transform museum data from Harvard API
  - analyze Harvard art collection with SQL queries
  - create interactive dashboard for museum artifacts
---

# Harvard Artifacts Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Harvard Artifacts Collection Data Engineering & Analytics App is an end-to-end data pipeline that demonstrates professional ETL workflows using the Harvard Art Museums API. It extracts artifact metadata, transforms nested JSON into relational structures, loads data into SQL databases, and provides interactive analytics through Streamlit dashboards.

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
export DB_HOST="your_database_host"
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="harvard_artifacts"
```

### Required Dependencies

```text
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### 1. Harvard Art Museums API Setup

Obtain an API key from: https://www.harvardartmuseums.org/collections/api

```python
import os
from dotenv import load_dotenv

load_dotenv()

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = 'https://api.harvardartmuseums.org/object'
```

### 2. Database Configuration

The application uses MySQL or TiDB Cloud. Configure connection parameters:

```python
import mysql.connector

db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME'),
    'port': int(os.getenv('DB_PORT', 3306))
}

connection = mysql.connector.connect(**db_config)
```

### 3. Database Schema

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    dated VARCHAR(255),
    classification VARCHAR(255),
    division VARCHAR(255),
    department VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accessionyear INT,
    url TEXT
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl TEXT,
    format VARCHAR(50),
    height INT,
    width INT,
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

## Core ETL Pipeline

### Extract: Fetch Data from API

```python
import requests
import time

def fetch_artifacts(api_key, num_pages=10, size=100):
    """
    Fetch artifact data from Harvard Art Museums API with pagination
    """
    artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': size,
            'page': page
        }
        
        try:
            response = requests.get(BASE_URL, params=params)
            response.raise_for_status()
            data = response.json()
            
            artifacts.extend(data.get('records', []))
            
            # Rate limiting
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return artifacts
```

### Transform: Process Nested JSON

```python
import pandas as pd

def transform_metadata(artifacts):
    """
    Transform artifact records into metadata dataframe
    """
    metadata_list = []
    
    for artifact in artifacts:
        metadata_list.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'classification': artifact.get('classification'),
            'division': artifact.get('division'),
            'department': artifact.get('department'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'accessionyear': artifact.get('accessionyear'),
            'url': artifact.get('url')
        })
    
    return pd.DataFrame(metadata_list)

def transform_media(artifacts):
    """
    Extract and transform media information
    """
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for image in images:
            media_list.append({
                'artifact_id': artifact_id,
                'baseimageurl': image.get('baseimageurl'),
                'format': image.get('format'),
                'height': image.get('height'),
                'width': image.get('width')
            })
    
    return pd.DataFrame(media_list)

def transform_colors(artifacts):
    """
    Extract and transform color information
    """
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_list.append({
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return pd.DataFrame(color_list)
```

### Load: Batch Insert to SQL

```python
def load_to_sql(df, table_name, connection):
    """
    Batch insert dataframe into SQL table
    """
    cursor = connection.cursor()
    
    # Replace NaN with None for SQL compatibility
    df = df.where(pd.notnull(df), None)
    
    # Create insert statement
    columns = ', '.join(df.columns)
    placeholders = ', '.join(['%s'] * len(df.columns))
    insert_query = f"INSERT IGNORE INTO {table_name} ({columns}) VALUES ({placeholders})"
    
    # Batch insert
    data_tuples = [tuple(x) for x in df.to_numpy()]
    cursor.executemany(insert_query, data_tuples)
    connection.commit()
    
    print(f"Inserted {cursor.rowcount} rows into {table_name}")
    cursor.close()
```

### Complete ETL Workflow

```python
def run_etl_pipeline():
    """
    Execute complete ETL pipeline
    """
    # Extract
    print("Extracting data from API...")
    artifacts = fetch_artifacts(API_KEY, num_pages=5, size=100)
    
    # Transform
    print("Transforming data...")
    metadata_df = transform_metadata(artifacts)
    media_df = transform_media(artifacts)
    colors_df = transform_colors(artifacts)
    
    # Load
    print("Loading data to database...")
    connection = mysql.connector.connect(**db_config)
    
    load_to_sql(metadata_df, 'artifactmetadata', connection)
    load_to_sql(media_df, 'artifactmedia', connection)
    load_to_sql(colors_df, 'artifactcolors', connection)
    
    connection.close()
    print("ETL pipeline completed!")
```

## SQL Analytics Queries

### Example Analytical Queries

```python
# Query 1: Top 10 cultures by artifact count
query_cultures = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10
"""

# Query 2: Artifacts by century
query_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC
"""

# Query 3: Media availability analysis
query_media = """
SELECT 
    CASE WHEN m.artifact_id IS NOT NULL THEN 'With Media' ELSE 'No Media' END as media_status,
    COUNT(*) as count
FROM artifactmetadata a
LEFT JOIN artifactmedia m ON a.id = m.artifact_id
GROUP BY media_status
"""

# Query 4: Top colors in collection
query_colors = """
SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
FROM artifactcolors
WHERE color IS NOT NULL
GROUP BY color
ORDER BY frequency DESC
LIMIT 10
"""

# Query 5: Department distribution
query_departments = """
SELECT department, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY artifact_count DESC
"""
```

### Execute Queries with Results

```python
def execute_query(query, connection):
    """
    Execute SQL query and return results as DataFrame
    """
    cursor = connection.cursor()
    cursor.execute(query)
    
    columns = [desc[0] for desc in cursor.description]
    results = cursor.fetchall()
    
    cursor.close()
    
    return pd.DataFrame(results, columns=columns)
```

## Streamlit Dashboard

### Basic App Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museums - Data Analytics Dashboard")
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Choose a page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "SQL Analytics":
        show_sql_analytics()
    elif page == "Visualizations":
        show_visualizations()

def show_data_collection():
    st.header("📥 Data Collection")
    
    num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=50, value=5)
    
    if st.button("Start ETL Pipeline"):
        with st.spinner("Running ETL pipeline..."):
            run_etl_pipeline()
        st.success("ETL pipeline completed successfully!")

def show_sql_analytics():
    st.header("📊 SQL Analytics")
    
    queries = {
        "Top Cultures": query_cultures,
        "Artifacts by Century": query_century,
        "Media Availability": query_media,
        "Top Colors": query_colors,
        "Department Distribution": query_departments
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        connection = mysql.connector.connect(**db_config)
        results = execute_query(queries[selected_query], connection)
        connection.close()
        
        st.dataframe(results)
        
        # Auto-generate visualization
        if len(results.columns) == 2:
            fig = px.bar(results, x=results.columns[0], y=results.columns[1])
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

### Interactive Visualizations

```python
def create_interactive_chart(df, x_col, y_col, chart_type="bar"):
    """
    Create interactive Plotly charts
    """
    if chart_type == "bar":
        fig = px.bar(df, x=x_col, y=y_col, 
                     title=f"{y_col} by {x_col}",
                     color=y_col,
                     color_continuous_scale='Viridis')
    elif chart_type == "pie":
        fig = px.pie(df, names=x_col, values=y_col,
                     title=f"Distribution of {y_col}")
    elif chart_type == "scatter":
        fig = px.scatter(df, x=x_col, y=y_col,
                        title=f"{y_col} vs {x_col}")
    
    fig.update_layout(
        template="plotly_white",
        hovermode='x unified'
    )
    
    return fig
```

## Common Patterns

### Rate-Limited API Calls

```python
from time import sleep
from functools import wraps

def rate_limit(calls_per_second=2):
    """
    Decorator for rate limiting API calls
    """
    min_interval = 1.0 / calls_per_second
    
    def decorator(func):
        last_called = [0.0]
        
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            left_to_wait = min_interval - elapsed
            
            if left_to_wait > 0:
                sleep(left_to_wait)
            
            result = func(*args, **kwargs)
            last_called[0] = time.time()
            return result
        
        return wrapper
    return decorator

@rate_limit(calls_per_second=2)
def fetch_artifact_by_id(artifact_id):
    response = requests.get(f"{BASE_URL}/{artifact_id}", params={'apikey': API_KEY})
    return response.json()
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)

def safe_etl_pipeline():
    """
    ETL pipeline with comprehensive error handling
    """
    try:
        artifacts = fetch_artifacts(API_KEY)
        logging.info(f"Fetched {len(artifacts)} artifacts")
        
        metadata_df = transform_metadata(artifacts)
        logging.info(f"Transformed {len(metadata_df)} metadata records")
        
        connection = mysql.connector.connect(**db_config)
        load_to_sql(metadata_df, 'artifactmetadata', connection)
        connection.close()
        
        logging.info("ETL pipeline completed successfully")
        return True
        
    except requests.exceptions.RequestException as e:
        logging.error(f"API request failed: {e}")
        return False
    except mysql.connector.Error as e:
        logging.error(f"Database error: {e}")
        return False
    except Exception as e:
        logging.error(f"Unexpected error: {e}")
        return False
```

## Running the Application

### Start Streamlit App

```bash
streamlit run app.py
```

### Command Line ETL Execution

```python
# etl_runner.py
if __name__ == "__main__":
    import argparse
    
    parser = argparse.ArgumentParser(description='Run Harvard Artifacts ETL Pipeline')
    parser.add_argument('--pages', type=int, default=5, help='Number of pages to fetch')
    parser.add_argument('--size', type=int, default=100, help='Records per page')
    
    args = parser.parse_args()
    
    artifacts = fetch_artifacts(API_KEY, num_pages=args.pages, size=args.size)
    run_etl_pipeline()
```

```bash
python etl_runner.py --pages 10 --size 50
```

## Troubleshooting

### API Connection Issues

- **Problem**: 401 Unauthorized
  - **Solution**: Verify API key is correctly set in environment variables
  
- **Problem**: Rate limiting errors
  - **Solution**: Implement exponential backoff or reduce request frequency

```python
def fetch_with_retry(url, params, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
            sleep(2 ** attempt)  # Exponential backoff
```

### Database Connection Issues

- **Problem**: Connection timeout
  - **Solution**: Check firewall settings and database host accessibility
  
- **Problem**: Foreign key constraint failures
  - **Solution**: Ensure parent records exist before inserting child records

```python
# Load in correct order
load_to_sql(metadata_df, 'artifactmetadata', connection)  # Parent first
load_to_sql(media_df, 'artifactmedia', connection)        # Then children
load_to_sql(colors_df, 'artifactcolors', connection)
```

### Memory Issues with Large Datasets

```python
def batch_process_artifacts(artifacts, batch_size=1000):
    """
    Process artifacts in batches to manage memory
    """
    for i in range(0, len(artifacts), batch_size):
        batch = artifacts[i:i+batch_size]
        
        metadata_df = transform_metadata(batch)
        connection = mysql.connector.connect(**db_config)
        load_to_sql(metadata_df, 'artifactmetadata', connection)
        connection.close()
        
        logging.info(f"Processed batch {i//batch_size + 1}")
```

## Best Practices

1. **Always use environment variables** for sensitive credentials
2. **Implement pagination** for large API responses
3. **Use batch inserts** for database operations
4. **Add logging** throughout the pipeline
5. **Handle NULL values** appropriately in transformations
6. **Create database indexes** on frequently queried columns
7. **Use connection pooling** for production deployments
8. **Validate data** before loading to database
