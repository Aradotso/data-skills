---
name: harvard-artifacts-etl-analytics
description: End-to-end ETL pipeline and analytics for Harvard Art Museums API with Streamlit dashboards
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - set up data engineering workflow with Harvard API
  - visualize Harvard Art Museums collection data
  - implement SQL analytics for artifact metadata
  - extract and transform museum API data
  - build Streamlit app for artifact analysis
  - create data pipeline with Harvard Museums API
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, analytical queries, and interactive Streamlit visualizations.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load nested JSON into relational SQL tables
- **Database Design**: Structured schema with `artifactmetadata`, `artifactmedia`, and `artifactcolors` tables
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Visualization**: Interactive Plotly charts via Streamlit dashboard

**Architecture**: API → ETL → SQL → Analytics → Visualization

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

### Environment Variables

Create a `.env` file or set environment variables:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Register at [Harvard Art Museums API](https://harvardartmuseums.org/collections/api)
2. Generate an API key
3. Store in environment variable `HARVARD_API_KEY`

## Core Components

### 1. API Data Collection

```python
import requests
import pandas as pd
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_records=100, page_size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    all_artifacts = []
    page = 1
    
    while len(all_artifacts) < num_records:
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code != 200:
            print(f"Error: {response.status_code}")
            break
            
        data = response.json()
        records = data.get('records', [])
        
        if not records:
            break
            
        all_artifacts.extend(records)
        page += 1
        
        # Respect rate limits
        import time
        time.sleep(0.5)
    
    return all_artifacts[:num_records]

# Usage
artifacts = fetch_artifacts(num_records=500)
print(f"Fetched {len(artifacts)} artifacts")
```

### 2. ETL Pipeline

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection():
    """Create MySQL database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def create_tables(connection):
    """Create database schema"""
    cursor = connection.cursor()
    
    # Artifact Metadata Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            objectid INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            period VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            department VARCHAR(200),
            dated VARCHAR(200),
            accessionyear INT,
            technique VARCHAR(500),
            medium VARCHAR(500),
            dimensions VARCHAR(500),
            creditline TEXT,
            description TEXT,
            provenance TEXT,
            url VARCHAR(500)
        )
    """)
    
    # Artifact Media Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            mediaid INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            iiifbaseuri VARCHAR(500),
            baseimageurl VARCHAR(500),
            primaryimageurl VARCHAR(500),
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    # Artifact Colors Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            colorid INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            color VARCHAR(50),
            spectrum VARCHAR(50),
            hue VARCHAR(50),
            percent DECIMAL(5,2),
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    connection.commit()
    cursor.close()

def transform_and_load(artifacts, connection):
    """Transform JSON data and load into SQL tables"""
    cursor = connection.cursor()
    
    for artifact in artifacts:
        # Insert metadata
        metadata_query = """
            INSERT IGNORE INTO artifactmetadata 
            (objectid, title, culture, period, century, classification, 
             department, dated, accessionyear, technique, medium, 
             dimensions, creditline, description, provenance, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """
        
        metadata_values = (
            artifact.get('objectid'),
            artifact.get('title'),
            artifact.get('culture'),
            artifact.get('period'),
            artifact.get('century'),
            artifact.get('classification'),
            artifact.get('department'),
            artifact.get('dated'),
            artifact.get('accessionyear'),
            artifact.get('technique'),
            artifact.get('medium'),
            artifact.get('dimensions'),
            artifact.get('creditline'),
            artifact.get('description'),
            artifact.get('provenance'),
            artifact.get('url')
        )
        
        cursor.execute(metadata_query, metadata_values)
        
        # Insert media data
        objectid = artifact.get('objectid')
        media_query = """
            INSERT INTO artifactmedia 
            (objectid, iiifbaseuri, baseimageurl, primaryimageurl)
            VALUES (%s, %s, %s, %s)
        """
        
        media_values = (
            objectid,
            artifact.get('iiifbaseuri'),
            artifact.get('baseimageurl'),
            artifact.get('primaryimageurl')
        )
        
        cursor.execute(media_query, media_values)
        
        # Insert color data
        colors = artifact.get('colors', [])
        for color in colors:
            color_query = """
                INSERT INTO artifactcolors 
                (objectid, color, spectrum, hue, percent)
                VALUES (%s, %s, %s, %s, %s)
            """
            
            color_values = (
                objectid,
                color.get('color'),
                color.get('spectrum'),
                color.get('hue'),
                color.get('percent')
            )
            
            cursor.execute(color_query, color_values)
    
    connection.commit()
    cursor.close()

# Complete ETL workflow
connection = create_database_connection()
if connection:
    create_tables(connection)
    artifacts = fetch_artifacts(num_records=500)
    transform_and_load(artifacts, connection)
    print(f"Loaded {len(artifacts)} artifacts into database")
    connection.close()
```

### 3. SQL Analytics Queries

```python
def execute_analytical_query(connection, query_name):
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
        
        'department_distribution': """
            SELECT department, COUNT(*) as artifact_count 
            FROM artifactmetadata 
            GROUP BY department 
            ORDER BY artifact_count DESC
        """,
        
        'top_colors': """
            SELECT color, COUNT(*) as usage_count, 
                   AVG(percent) as avg_percent
            FROM artifactcolors 
            GROUP BY color 
            ORDER BY usage_count DESC 
            LIMIT 10
        """,
        
        'artifacts_with_images': """
            SELECT 
                COUNT(DISTINCT m.objectid) as with_images,
                (SELECT COUNT(*) FROM artifactmetadata) as total,
                ROUND(COUNT(DISTINCT m.objectid) * 100.0 / 
                      (SELECT COUNT(*) FROM artifactmetadata), 2) as percentage
            FROM artifactmedia m
            WHERE m.primaryimageurl IS NOT NULL
        """,
        
        'accession_year_trends': """
            SELECT accessionyear, COUNT(*) as acquisitions
            FROM artifactmetadata
            WHERE accessionyear IS NOT NULL
            GROUP BY accessionyear
            ORDER BY accessionyear DESC
            LIMIT 20
        """
    }
    
    if query_name not in queries:
        return None
    
    cursor = connection.cursor(dictionary=True)
    cursor.execute(queries[query_name])
    results = cursor.fetchall()
    cursor.close()
    
    return pd.DataFrame(results)

# Usage
connection = create_database_connection()
df_cultures = execute_analytical_query(connection, 'artifacts_by_culture')
print(df_cultures)
connection.close()
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    st.markdown("Interactive analytics for artifact collection data")
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Queries")
    
    query_options = {
        "Artifacts by Culture": "artifacts_by_culture",
        "Artifacts by Century": "artifacts_by_century",
        "Department Distribution": "department_distribution",
        "Top Colors Used": "top_colors",
        "Image Availability": "artifacts_with_images",
        "Accession Year Trends": "accession_year_trends"
    }
    
    selected_query = st.sidebar.selectbox(
        "Select Analysis",
        list(query_options.keys())
    )
    
    # Database connection
    connection = create_database_connection()
    
    if connection:
        # Execute query
        query_key = query_options[selected_query]
        df = execute_analytical_query(connection, query_key)
        
        # Display results
        st.subheader(selected_query)
        st.dataframe(df, use_container_width=True)
        
        # Visualization
        if not df.empty and len(df.columns) >= 2:
            st.subheader("Visualization")
            
            x_col = df.columns[0]
            y_col = df.columns[1]
            
            fig = px.bar(
                df, 
                x=x_col, 
                y=y_col,
                title=f"{selected_query} - Bar Chart",
                labels={x_col: x_col.title(), y_col: y_col.title()}
            )
            
            st.plotly_chart(fig, use_container_width=True)
        
        connection.close()
    else:
        st.error("Database connection failed. Check configuration.")

if __name__ == "__main__":
    main()
```

## Running the Application

### Local Development

```bash
# Run Streamlit app
streamlit run app.py

# The app will open at http://localhost:8501
```

### ETL Pipeline Execution

```python
# Run complete ETL pipeline
python etl_pipeline.py

# Or programmatically
from etl_pipeline import run_etl

run_etl(num_artifacts=1000)
```

## Common Patterns

### Batch Processing with Error Handling

```python
def batch_insert_artifacts(artifacts, connection, batch_size=100):
    """Insert artifacts in batches for better performance"""
    cursor = connection.cursor()
    
    for i in range(0, len(artifacts), batch_size):
        batch = artifacts[i:i + batch_size]
        
        try:
            for artifact in batch:
                # Insert logic here
                pass
            
            connection.commit()
            print(f"Processed batch {i//batch_size + 1}")
            
        except Error as e:
            print(f"Error in batch {i//batch_size + 1}: {e}")
            connection.rollback()
    
    cursor.close()
```

### Incremental Data Loading

```python
def get_latest_objectid(connection):
    """Get the highest objectid already in database"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(objectid) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    
    return result[0] if result[0] else 0

def incremental_load(connection):
    """Load only new artifacts"""
    latest_id = get_latest_objectid(connection)
    
    # Fetch artifacts with objectid > latest_id
    params = {
        'apikey': os.getenv('HARVARD_API_KEY'),
        'size': 100,
        'sort': 'objectid',
        'sortorder': 'asc'
    }
    
    # Add filter for objectid > latest_id
    # Transform and load new data
```

## Troubleshooting

### API Rate Limiting

```python
import time
from functools import wraps

def rate_limit(calls_per_second=2):
    """Decorator to rate limit API calls"""
    min_interval = 1.0 / calls_per_second
    last_called = [0.0]
    
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            wait_time = min_interval - elapsed
            
            if wait_time > 0:
                time.sleep(wait_time)
            
            result = func(*args, **kwargs)
            last_called[0] = time.time()
            return result
        
        return wrapper
    return decorator

@rate_limit(calls_per_second=2)
def fetch_page(page):
    # API call here
    pass
```

### Database Connection Issues

```python
from mysql.connector import pooling

# Use connection pooling for better performance
connection_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    pool_reset_session=True,
    host=os.getenv('DB_HOST'),
    database=os.getenv('DB_NAME'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD')
)

def get_connection():
    """Get connection from pool"""
    return connection_pool.get_connection()
```

### Handling Missing Data

```python
def safe_get(data, key, default=None):
    """Safely extract nested data"""
    try:
        return data.get(key, default)
    except AttributeError:
        return default

# Usage in ETL
culture = safe_get(artifact, 'culture', 'Unknown')
accession_year = safe_get(artifact, 'accessionyear', 0)
```

## Best Practices

1. **Environment Variables**: Always use `.env` for sensitive credentials
2. **Error Logging**: Implement comprehensive logging for ETL failures
3. **Data Validation**: Validate API responses before inserting into database
4. **Indexing**: Create indexes on frequently queried columns (culture, century, department)
5. **Pagination**: Handle API pagination properly to avoid missing data
6. **Connection Management**: Use connection pooling and properly close connections
7. **Incremental Updates**: Implement incremental loading to avoid full reloads
