---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Streamlit, SQL, and Plotly
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - create a data analytics dashboard with Harvard artifacts
  - extract and transform Harvard Art Museums API data
  - build a Streamlit app for art museum analytics
  - set up SQL database for Harvard artifacts collection
  - query and visualize Harvard Art Museums data
  - implement ETL pipeline with Pandas and MySQL
  - analyze Harvard art collection with Python
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipeline construction, SQL database design, and interactive visualization with Streamlit.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides:
- Dynamic artifact data collection from Harvard Art Museums API
- ETL (Extract, Transform, Load) operations on nested JSON data
- Relational database storage (MySQL/TiDB Cloud)
- Analytical SQL query execution
- Interactive Streamlit dashboards with Plotly visualizations

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Install core packages if requirements.txt is not available
pip install streamlit pandas requests mysql-connector-python plotly sqlalchemy
```

## Configuration

### API Key Setup

Get your Harvard Art Museums API key from https://docs.harvardartmuseums.org/

```python
# Store API key in environment variable
import os
os.environ['HARVARD_API_KEY'] = 'your-api-key-here'

# Or use Streamlit secrets
# Create .streamlit/secrets.toml
"""
[api]
harvard_key = "your-api-key-here"

[database]
host = "your-db-host"
user = "your-db-user"
password = "your-db-password"
database = "harvard_artifacts"
"""
```

### Database Configuration

```python
import mysql.connector
from sqlalchemy import create_engine

# Using mysql-connector-python
db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

connection = mysql.connector.connect(**db_config)

# Using SQLAlchemy
engine = create_engine(
    f"mysql+mysqlconnector://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}@{os.getenv('DB_HOST')}/{os.getenv('DB_NAME')}"
)
```

## Key Components

### 1. API Data Extraction

```python
import requests
import pandas as pd

def extract_artifacts(api_key, num_records=100):
    """Extract artifact data from Harvard Art Museums API with pagination."""
    base_url = "https://api.harvardartmuseums.org/object"
    artifacts = []
    page = 1
    per_page = 100
    
    while len(artifacts) < num_records:
        params = {
            'apikey': api_key,
            'page': page,
            'size': min(per_page, num_records - len(artifacts))
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            artifacts.extend(data.get('records', []))
            
            if len(data.get('records', [])) < per_page:
                break
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_records]

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts = extract_artifacts(api_key, num_records=500)
```

### 2. Data Transformation

```python
def transform_artifacts(artifacts):
    """Transform nested JSON into relational dataframes."""
    
    # Artifact Metadata
    metadata = []
    for artifact in artifacts:
        metadata.append({
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url')
        })
    
    df_metadata = pd.DataFrame(metadata)
    
    # Artifact Media
    media = []
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        for img in artifact.get('images', []):
            media.append({
                'objectid': objectid,
                'imageid': img.get('imageid'),
                'baseimageurl': img.get('baseimageurl'),
                'format': img.get('format'),
                'width': img.get('width'),
                'height': img.get('height')
            })
    
    df_media = pd.DataFrame(media)
    
    # Artifact Colors
    colors = []
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        for color in artifact.get('colors', []):
            colors.append({
                'objectid': objectid,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'percentage': color.get('percent')
            })
    
    df_colors = pd.DataFrame(colors)
    
    return df_metadata, df_media, df_colors
```

### 3. Database Schema Creation

```python
def create_database_schema(connection):
    """Create tables for artifact data."""
    cursor = connection.cursor()
    
    # Artifact Metadata Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            objectid INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(255),
            period VARCHAR(255),
            century VARCHAR(100),
            classification VARCHAR(255),
            department VARCHAR(255),
            dated VARCHAR(255),
            url VARCHAR(500)
        )
    """)
    
    # Artifact Media Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            imageid INT,
            baseimageurl VARCHAR(500),
            format VARCHAR(50),
            width INT,
            height INT,
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    # Artifact Colors Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            color VARCHAR(50),
            spectrum VARCHAR(50),
            percentage FLOAT,
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    connection.commit()
    cursor.close()
```

### 4. Data Loading

```python
def load_data_to_sql(df_metadata, df_media, df_colors, connection):
    """Load dataframes into SQL database with batch inserts."""
    
    # Load metadata
    df_metadata.to_sql(
        'artifactmetadata',
        con=connection,
        if_exists='append',
        index=False,
        method='multi',
        chunksize=1000
    )
    
    # Load media
    df_media.to_sql(
        'artifactmedia',
        con=connection,
        if_exists='append',
        index=False,
        method='multi',
        chunksize=1000
    )
    
    # Load colors
    df_colors.to_sql(
        'artifactcolors',
        con=connection,
        if_exists='append',
        index=False,
        method='multi',
        chunksize=1000
    )
```

### 5. Analytical SQL Queries

```python
analytical_queries = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Media Availability": """
        SELECT 
            CASE WHEN m.objectid IS NOT NULL THEN 'Has Media' ELSE 'No Media' END as media_status,
            COUNT(DISTINCT a.objectid) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.objectid = m.objectid
        GROUP BY media_status
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as usage_count, AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Classification Analysis": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 15
    """
}

def execute_query(connection, query):
    """Execute analytical query and return results as DataFrame."""
    return pd.read_sql(query, connection)
```

### 6. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("API Key", type="password", value=os.getenv('HARVARD_API_KEY', ''))
    num_records = st.sidebar.slider("Number of Records", 100, 1000, 500)
    
    # ETL Section
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Extracting data from API..."):
            artifacts = extract_artifacts(api_key, num_records)
            st.success(f"Extracted {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_metadata, df_media, df_colors = transform_artifacts(artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            connection = mysql.connector.connect(**db_config)
            create_database_schema(connection)
            load_data_to_sql(df_metadata, df_media, df_colors, connection)
            st.success("Data loaded to database")
    
    # Analytics Section
    st.header("Analytics Dashboard")
    
    query_name = st.selectbox("Select Analysis", list(analytical_queries.keys()))
    
    if st.button("Run Query"):
        connection = mysql.connector.connect(**db_config)
        df_result = execute_query(connection, analytical_queries[query_name])
        
        st.subheader("Results")
        st.dataframe(df_result)
        
        # Auto-generate visualization
        if len(df_result.columns) >= 2:
            fig = px.bar(
                df_result,
                x=df_result.columns[0],
                y=df_result.columns[1],
                title=query_name
            )
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Run the Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Rate Limiting API Calls

```python
import time

def extract_with_rate_limit(api_key, num_records, delay=0.1):
    """Extract data with rate limiting."""
    artifacts = []
    for page in range(1, (num_records // 100) + 2):
        response = requests.get(
            "https://api.harvardartmuseums.org/object",
            params={'apikey': api_key, 'page': page, 'size': 100}
        )
        if response.status_code == 200:
            artifacts.extend(response.json().get('records', []))
        time.sleep(delay)  # Rate limiting
    return artifacts[:num_records]
```

### Error Handling in ETL

```python
def safe_transform(artifacts):
    """Transform with error handling."""
    metadata = []
    for artifact in artifacts:
        try:
            metadata.append({
                'objectid': artifact.get('objectid'),
                'title': artifact.get('title', 'Unknown'),
                'culture': artifact.get('culture'),
                'century': artifact.get('century')
            })
        except Exception as e:
            st.warning(f"Skipping artifact {artifact.get('objectid')}: {e}")
            continue
    return pd.DataFrame(metadata)
```

### Incremental Data Loading

```python
def incremental_load(new_artifacts, connection):
    """Load only new artifacts not in database."""
    cursor = connection.cursor()
    cursor.execute("SELECT objectid FROM artifactmetadata")
    existing_ids = set(row[0] for row in cursor.fetchall())
    
    new_data = [a for a in new_artifacts if a.get('objectid') not in existing_ids]
    
    if new_data:
        df_metadata, df_media, df_colors = transform_artifacts(new_data)
        load_data_to_sql(df_metadata, df_media, df_colors, connection)
        return len(new_data)
    return 0
```

## Troubleshooting

**API Key Issues:**
```python
# Verify API key works
response = requests.get(
    "https://api.harvardartmuseums.org/object",
    params={'apikey': os.getenv('HARVARD_API_KEY'), 'size': 1}
)
print(f"Status: {response.status_code}")
```

**Database Connection Errors:**
```python
# Test connection
try:
    connection = mysql.connector.connect(**db_config)
    print("Database connected successfully")
    connection.close()
except mysql.connector.Error as err:
    print(f"Error: {err}")
```

**Memory Issues with Large Datasets:**
```python
# Use chunked processing
def chunked_extract(api_key, total_records, chunk_size=100):
    for offset in range(0, total_records, chunk_size):
        artifacts = extract_artifacts(api_key, chunk_size)
        df_metadata, df_media, df_colors = transform_artifacts(artifacts)
        load_data_to_sql(df_metadata, df_media, df_colors, connection)
```
