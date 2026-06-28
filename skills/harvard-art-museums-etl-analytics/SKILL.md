---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - create an ETL pipeline for Harvard Art Museums data
  - build a Streamlit dashboard for museum artifact analytics
  - extract and transform Harvard Art Museums API data
  - set up a data engineering pipeline for art collection data
  - implement SQL analytics on museum artifacts
  - visualize Harvard Art Museums collection data
  - create a data pipeline from Harvard API to SQL database
  - build an interactive analytics app for museum collections
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. It covers ETL pipeline development, SQL database design, analytical query execution, and interactive visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App demonstrates:

- **API Data Collection**: Fetches artifact metadata from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts nested JSON, transforms into relational format, loads into SQL databases
- **SQL Database Design**: Creates normalized tables for artifact metadata, media, and color data
- **Analytics Queries**: Executes 20+ predefined analytical SQL queries
- **Interactive Dashboards**: Visualizes query results using Streamlit and Plotly

**Architecture Flow**: API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly sqlalchemy
```

## Configuration

### API Key Setup

Obtain a Harvard Art Museums API key from [https://www.harvardartmuseums.org/collections/api](https://www.harvardartmuseums.org/collections/api)

Store the API key in environment variables or Streamlit secrets:

```bash
# .env file
HARVARD_API_KEY=your_api_key_here
```

### Database Configuration

Set up MySQL or TiDB Cloud connection:

```python
# config.py or environment variables
DB_HOST = "your_db_host"
DB_PORT = 3306
DB_USER = "your_db_user"
DB_PASSWORD = "your_db_password"
DB_NAME = "harvard_artifacts"
```

For Streamlit secrets (`.streamlit/secrets.toml`):

```toml
[database]
host = "your_db_host"
port = 3306
user = "your_db_user"
password = "your_db_password"
database = "harvard_artifacts"

[api]
harvard_api_key = "your_api_key_here"
```

## Database Schema

### Create Tables

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255),
    url TEXT,
    totalpageviews INT,
    totaluniquepageviews INT
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl TEXT,
    iiifbaseuri TEXT,
    publiccaption TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percent DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### Extract Data from API

```python
import requests
import pandas as pd
import os

def fetch_artifacts(api_key, num_pages=10, page_size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        try:
            response = requests.get(base_url, params=params, timeout=30)
            response.raise_for_status()
            data = response.json()
            
            if 'records' in data:
                all_artifacts.extend(data['records'])
            
            # Rate limiting
            import time
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            continue
    
    return all_artifacts
```

### Transform Data

```python
def transform_artifacts(artifacts):
    """
    Transform nested JSON into relational DataFrames
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url'),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        }
        metadata_list.append(metadata)
        
        # Extract media
        if artifact.get('images'):
            for image in artifact['images']:
                media = {
                    'artifact_id': artifact.get('id'),
                    'baseimageurl': image.get('baseimageurl'),
                    'iiifbaseuri': image.get('iiifbaseuri'),
                    'publiccaption': image.get('publiccaption')
                }
                media_list.append(media)
        
        # Extract colors
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_data = {
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color'),
                    'percent': color.get('percent')
                }
                colors_list.append(color_data)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load Data to SQL

```python
import mysql.connector
from sqlalchemy import create_engine

def load_to_database(metadata_df, media_df, colors_df, db_config):
    """
    Load DataFrames into MySQL database using batch inserts
    """
    # Create SQLAlchemy engine
    engine = create_engine(
        f"mysql+mysqlconnector://{db_config['user']}:{db_config['password']}"
        f"@{db_config['host']}:{db_config['port']}/{db_config['database']}"
    )
    
    # Load data with error handling
    try:
        # Use replace to handle duplicates
        metadata_df.to_sql('artifactmetadata', engine, 
                          if_exists='append', index=False, method='multi')
        media_df.to_sql('artifactmedia', engine, 
                       if_exists='append', index=False, method='multi')
        colors_df.to_sql('artifactcolors', engine, 
                        if_exists='append', index=False, method='multi')
        
        print("Data loaded successfully!")
        
    except Exception as e:
        print(f"Error loading data: {e}")
    finally:
        engine.dispose()
```

## Streamlit Application

### Main App Structure

```python
import streamlit as st
import pandas as pd
import plotly.express as px
from sqlalchemy import create_engine

st.set_page_config(page_title="Harvard Art Analytics", layout="wide")

# Sidebar configuration
st.sidebar.title("Configuration")
api_key = st.sidebar.text_input("Harvard API Key", type="password",
                                value=st.secrets.get("api", {}).get("harvard_api_key", ""))

# Database connection
@st.cache_resource
def get_database_connection():
    db_config = st.secrets["database"]
    engine = create_engine(
        f"mysql+mysqlconnector://{db_config['user']}:{db_config['password']}"
        f"@{db_config['host']}:{db_config['port']}/{db_config['database']}"
    )
    return engine

# ETL Section
st.title("🎨 Harvard Art Museums Analytics")

tab1, tab2, tab3 = st.tabs(["Data Collection", "Analytics", "Visualizations"])

with tab1:
    st.header("ETL Pipeline")
    
    num_pages = st.number_input("Number of pages to fetch", 1, 100, 10)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            artifacts = fetch_artifacts(api_key, num_pages)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            engine = get_database_connection()
            db_config = st.secrets["database"]
            load_to_database(metadata_df, media_df, colors_df, db_config)
            st.success("Data loaded to database")
```

## Analytical SQL Queries

### Example Queries

```python
# Query definitions
ANALYTICAL_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Top Viewed Artifacts": """
        SELECT title, culture, totalpageviews
        FROM artifactmetadata
        ORDER BY totalpageviews DESC
        LIMIT 20
    """,
    
    "Color Distribution": """
        SELECT color, ROUND(AVG(percent), 2) as avg_percent, COUNT(*) as usage_count
        FROM artifactcolors
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "Artifacts with Media": """
        SELECT 
            CASE WHEN media_count > 0 THEN 'Has Media' ELSE 'No Media' END as media_status,
            COUNT(*) as count
        FROM (
            SELECT a.id, COUNT(m.id) as media_count
            FROM artifactmetadata a
            LEFT JOIN artifactmedia m ON a.id = m.artifact_id
            GROUP BY a.id
        ) subquery
        GROUP BY media_status
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """
}
```

### Execute and Visualize Queries

```python
with tab2:
    st.header("SQL Analytics Dashboard")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICAL_QUERIES.keys()))
    
    if st.button("Run Query"):
        engine = get_database_connection()
        
        try:
            df_result = pd.read_sql(ANALYTICAL_QUERIES[query_name], engine)
            
            st.subheader("Query Results")
            st.dataframe(df_result, use_container_width=True)
            
            # Auto-generate visualization
            if len(df_result.columns) >= 2:
                st.subheader("Visualization")
                fig = px.bar(df_result, 
                           x=df_result.columns[0], 
                           y=df_result.columns[1],
                           title=query_name)
                st.plotly_chart(fig, use_container_width=True)
                
        except Exception as e:
            st.error(f"Query error: {e}")
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Common Patterns

### Handling API Rate Limits

```python
import time
from requests.exceptions import HTTPError

def fetch_with_retry(url, params, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=30)
            response.raise_for_status()
            return response.json()
        except HTTPError as e:
            if e.response.status_code == 429:  # Rate limited
                wait_time = 2 ** attempt  # Exponential backoff
                time.sleep(wait_time)
            else:
                raise
    return None
```

### Incremental Data Loading

```python
def get_last_artifact_id(engine):
    """Get the highest artifact ID already in database"""
    query = "SELECT MAX(id) as max_id FROM artifactmetadata"
    result = pd.read_sql(query, engine)
    return result['max_id'].iloc[0] or 0

def load_new_artifacts_only(artifacts, engine):
    """Load only artifacts newer than what's in database"""
    last_id = get_last_artifact_id(engine)
    new_artifacts = [a for a in artifacts if a['id'] > last_id]
    return new_artifacts
```

## Troubleshooting

### API Connection Issues

```python
# Test API connectivity
def test_api_connection(api_key):
    url = "https://api.harvardartmuseums.org/object"
    params = {'apikey': api_key, 'size': 1}
    
    try:
        response = requests.get(url, params=params, timeout=10)
        response.raise_for_status()
        return True, "API connection successful"
    except requests.exceptions.RequestException as e:
        return False, f"API connection failed: {e}"
```

### Database Connection Issues

```python
# Test database connectivity
def test_db_connection(db_config):
    try:
        conn = mysql.connector.connect(**db_config)
        conn.close()
        return True, "Database connection successful"
    except mysql.connector.Error as e:
        return False, f"Database connection failed: {e}"
```

### Empty Results

- **Check API key validity**: Ensure the API key is active
- **Verify pagination**: Some pages may return empty results
- **Check filters**: `hasimage=1` filter may limit results
- **Database constraints**: Foreign key violations on insert

### Performance Optimization

```python
# Use connection pooling for better performance
from sqlalchemy.pool import QueuePool

engine = create_engine(
    connection_string,
    poolclass=QueuePool,
    pool_size=10,
    max_overflow=20
)

# Batch inserts for faster loading
def batch_insert(df, table_name, engine, batch_size=1000):
    for i in range(0, len(df), batch_size):
        batch = df.iloc[i:i+batch_size]
        batch.to_sql(table_name, engine, if_exists='append', 
                    index=False, method='multi')
```

This skill provides comprehensive guidance for building ETL pipelines and analytics dashboards with the Harvard Art Museums API using modern data engineering practices.
