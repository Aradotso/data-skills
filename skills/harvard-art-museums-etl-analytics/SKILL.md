---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for museum data
  - create analytics dashboard with Harvard Art Museums API
  - set up data pipeline with Streamlit visualization
  - extract and transform museum artifact data
  - build SQL analytics for art collection data
  - create museum data engineering pipeline
  - visualize Harvard Art Museums data with Plotly
  - implement ETL workflow for cultural heritage data
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides a complete data engineering and analytics solution for the Harvard Art Museums collection. It demonstrates:

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON into normalized relational tables
- **SQL Analytics**: Stores data in MySQL/TiDB with predefined analytical queries
- **Visualization**: Interactive Streamlit dashboard with Plotly charts

The application extracts artifact metadata, media details, and color information, then enables SQL-based analytics with real-time visualization.

## Installation

```bash
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_database_host"
export DB_PORT="4000"
export DB_USER="your_db_user"
export DB_PASSWORD="your_db_password"
export DB_NAME="artifacts_db"
```

### Requirements

```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Database Schema

The project uses three main tables with relational structure:

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    dated VARCHAR(200),
    medium VARCHAR(500),
    dimensions VARCHAR(500)
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    baseimageurl VARCHAR(1000),
    primaryimageurl VARCHAR(1000),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## Core Components

### 1. API Data Collection

```python
import requests
import pandas as pd
import os

class HarvardAPICollector:
    def __init__(self, api_key):
        self.api_key = api_key
        self.base_url = "https://api.harvardartmuseums.org/object"
    
    def fetch_artifacts(self, num_pages=5, page_size=100):
        """Fetch artifacts with pagination"""
        all_artifacts = []
        
        for page in range(1, num_pages + 1):
            params = {
                'apikey': self.api_key,
                'size': page_size,
                'page': page,
                'hasimage': 1  # Only artifacts with images
            }
            
            response = requests.get(self.base_url, params=params)
            
            if response.status_code == 200:
                data = response.json()
                all_artifacts.extend(data.get('records', []))
            else:
                print(f"Error fetching page {page}: {response.status_code}")
                break
        
        return all_artifacts

# Usage
api_key = os.getenv('HARVARD_API_KEY')
collector = HarvardAPICollector(api_key)
artifacts = collector.fetch_artifacts(num_pages=3)
```

### 2. ETL Transformation

```python
import pandas as pd

def transform_artifacts(raw_data):
    """Transform nested JSON to normalized dataframes"""
    
    # Metadata transformation
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title', ''),
            'culture': artifact.get('culture', ''),
            'century': artifact.get('century', ''),
            'classification': artifact.get('classification', ''),
            'department': artifact.get('department', ''),
            'division': artifact.get('division', ''),
            'dated': artifact.get('dated', ''),
            'medium': artifact.get('medium', ''),
            'dimensions': artifact.get('dimensions', '')
        }
        metadata_records.append(metadata)
        
        # Extract media
        media = {
            'objectid': artifact.get('objectid'),
            'baseimageurl': artifact.get('baseimageurl', ''),
            'primaryimageurl': artifact.get('primaryimageurl', '')
        }
        media_records.append(media)
        
        # Extract colors
        colors = artifact.get('colors', [])
        for color in colors:
            color_record = {
                'objectid': artifact.get('objectid'),
                'color': color.get('color', ''),
                'spectrum': color.get('spectrum', ''),
                'percent': color.get('percent', 0.0)
            }
            color_records.append(color_record)
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )

# Usage
metadata_df, media_df, colors_df = transform_artifacts(artifacts)
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

class DatabaseLoader:
    def __init__(self, host, port, user, password, database):
        self.config = {
            'host': host,
            'port': port,
            'user': user,
            'password': password,
            'database': database
        }
    
    def get_connection(self):
        """Create database connection"""
        try:
            conn = mysql.connector.connect(**self.config)
            return conn
        except Error as e:
            print(f"Error connecting to database: {e}")
            return None
    
    def load_metadata(self, df):
        """Batch insert metadata"""
        conn = self.get_connection()
        if not conn:
            return False
        
        cursor = conn.cursor()
        
        # Use INSERT IGNORE to skip duplicates
        insert_query = """
            INSERT IGNORE INTO artifactmetadata 
            (objectid, title, culture, century, classification, 
             department, division, dated, medium, dimensions)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """
        
        records = df.to_records(index=False)
        cursor.executemany(insert_query, records.tolist())
        
        conn.commit()
        cursor.close()
        conn.close()
        return True
    
    def load_media(self, df):
        """Batch insert media data"""
        conn = self.get_connection()
        if not conn:
            return False
        
        cursor = conn.cursor()
        
        insert_query = """
            INSERT INTO artifactmedia (objectid, baseimageurl, primaryimageurl)
            VALUES (%s, %s, %s)
        """
        
        records = df.to_records(index=False)
        cursor.executemany(insert_query, records.tolist())
        
        conn.commit()
        cursor.close()
        conn.close()
        return True
    
    def load_colors(self, df):
        """Batch insert color data"""
        conn = self.get_connection()
        if not conn:
            return False
        
        cursor = conn.cursor()
        
        insert_query = """
            INSERT INTO artifactcolors (objectid, color, spectrum, percent)
            VALUES (%s, %s, %s, %s)
        """
        
        records = df.to_records(index=False)
        cursor.executemany(insert_query, records.tolist())
        
        conn.commit()
        cursor.close()
        conn.close()
        return True

# Usage
loader = DatabaseLoader(
    host=os.getenv('DB_HOST'),
    port=int(os.getenv('DB_PORT')),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

loader.load_metadata(metadata_df)
loader.load_media(media_df)
loader.load_colors(colors_df)
```

### 4. SQL Analytics Queries

```python
# Example analytical queries

ANALYTICS_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "artifacts_by_department": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "color_distribution": """
        SELECT color, COUNT(*) as color_count, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY color_count DESC
        LIMIT 20
    """,
    
    "artifacts_with_images": """
        SELECT 
            CASE WHEN primaryimageurl != '' THEN 'Has Image' ELSE 'No Image' END as image_status,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY image_status
    """,
    
    "top_classifications": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL AND classification != ''
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "culture_century_matrix": """
        SELECT culture, century, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND century IS NOT NULL
        GROUP BY culture, century
        HAVING count > 5
        ORDER BY count DESC
        LIMIT 20
    """
}

def execute_analytics_query(query_name):
    """Execute analytics query and return results"""
    loader = DatabaseLoader(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT')),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    conn = loader.get_connection()
    if not conn:
        return None
    
    query = ANALYTICS_QUERIES.get(query_name)
    df = pd.read_sql(query, conn)
    conn.close()
    
    return df
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Art Analytics", layout="wide")
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "SQL Analytics":
        show_sql_analytics()
    elif page == "Visualizations":
        show_visualizations()

def show_data_collection():
    """Data collection interface"""
    st.header("📥 Collect Artifact Data")
    
    api_key = st.text_input("Harvard API Key", type="password", 
                            value=os.getenv('HARVARD_API_KEY', ''))
    num_pages = st.slider("Number of Pages to Fetch", 1, 10, 3)
    
    if st.button("Fetch & Load Data"):
        with st.spinner("Fetching data from API..."):
            collector = HarvardAPICollector(api_key)
            artifacts = collector.fetch_artifacts(num_pages=num_pages)
            
            st.success(f"Fetched {len(artifacts)} artifacts")
            
            # Transform
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            
            st.write("### Preview")
            st.dataframe(metadata_df.head())
            
            # Load to database
            if st.button("Load to Database"):
                loader = DatabaseLoader(
                    host=os.getenv('DB_HOST'),
                    port=int(os.getenv('DB_PORT')),
                    user=os.getenv('DB_USER'),
                    password=os.getenv('DB_PASSWORD'),
                    database=os.getenv('DB_NAME')
                )
                
                loader.load_metadata(metadata_df)
                loader.load_media(media_df)
                loader.load_colors(colors_df)
                
                st.success("Data loaded successfully!")

def show_sql_analytics():
    """SQL analytics interface"""
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    st.code(ANALYTICS_QUERIES[query_name], language='sql')
    
    if st.button("Execute Query"):
        df = execute_analytics_query(query_name)
        
        if df is not None and not df.empty:
            st.dataframe(df)
            
            # Auto-generate chart
            if len(df.columns) == 2:
                fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                           title=f"Results: {query_name}")
                st.plotly_chart(fig, use_container_width=True)

def show_visualizations():
    """Custom visualizations"""
    st.header("📈 Visualizations")
    
    viz_type = st.selectbox(
        "Select Visualization",
        ["Culture Distribution", "Century Timeline", "Color Analysis"]
    )
    
    if viz_type == "Culture Distribution":
        df = execute_analytics_query("artifacts_by_culture")
        fig = px.bar(df, x='culture', y='artifact_count',
                    title="Artifacts by Culture")
        st.plotly_chart(fig, use_container_width=True)
    
    elif viz_type == "Century Timeline":
        df = execute_analytics_query("artifacts_by_century")
        fig = px.line(df, x='century', y='artifact_count',
                     title="Artifacts by Century")
        st.plotly_chart(fig, use_container_width=True)
    
    elif viz_type == "Color Analysis":
        df = execute_analytics_query("color_distribution")
        fig = px.scatter(df, x='color', y='color_count', size='avg_percent',
                        title="Color Distribution")
        st.plotly_chart(fig, use_container_width=True)

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

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(collector, num_pages, delay=1):
    """Fetch data with rate limiting"""
    artifacts = []
    
    for page in range(1, num_pages + 1):
        page_data = collector.fetch_artifacts(num_pages=1, page_size=100)
        artifacts.extend(page_data)
        
        if page < num_pages:
            time.sleep(delay)  # Wait between requests
    
    return artifacts
```

### Error Handling in ETL

```python
def safe_transform(raw_data):
    """Transform with error handling"""
    metadata_records = []
    
    for artifact in raw_data:
        try:
            metadata = {
                'objectid': artifact.get('objectid'),
                'title': artifact.get('title', '')[:500],  # Truncate long titles
                'culture': artifact.get('culture', ''),
                # ... other fields
            }
            metadata_records.append(metadata)
        except Exception as e:
            print(f"Error processing artifact {artifact.get('objectid')}: {e}")
            continue
    
    return pd.DataFrame(metadata_records)
```

### Incremental Data Loading

```python
def get_existing_objectids(loader):
    """Get already loaded object IDs"""
    conn = loader.get_connection()
    query = "SELECT DISTINCT objectid FROM artifactmetadata"
    df = pd.read_sql(query, conn)
    conn.close()
    return set(df['objectid'].tolist())

def incremental_load(artifacts, loader):
    """Load only new artifacts"""
    existing_ids = get_existing_objectids(loader)
    new_artifacts = [a for a in artifacts if a['objectid'] not in existing_ids]
    
    if new_artifacts:
        metadata_df, media_df, colors_df = transform_artifacts(new_artifacts)
        loader.load_metadata(metadata_df)
        loader.load_media(media_df)
        loader.load_colors(colors_df)
        
    return len(new_artifacts)
```

## Troubleshooting

### API Key Issues

```python
# Verify API key works
import requests

def test_api_key(api_key):
    response = requests.get(
        "https://api.harvardartmuseums.org/object",
        params={'apikey': api_key, 'size': 1}
    )
    
    if response.status_code == 200:
        print("✓ API key is valid")
        return True
    else:
        print(f"✗ API error: {response.status_code}")
        return False
```

### Database Connection Problems

```python
# Test database connection
def test_db_connection():
    try:
        loader = DatabaseLoader(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT')),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        conn = loader.get_connection()
        
        if conn:
            print("✓ Database connection successful")
            conn.close()
            return True
    except Exception as e:
        print(f"✗ Database error: {e}")
        return False
```

### Memory Issues with Large Datasets

```python
# Process in batches
def batch_process(artifacts, batch_size=100):
    """Process artifacts in batches"""
    for i in range(0, len(artifacts), batch_size):
        batch = artifacts[i:i+batch_size]
        metadata_df, media_df, colors_df = transform_artifacts(batch)
        
        loader.load_metadata(metadata_df)
        loader.load_media(media_df)
        loader.load_colors(colors_df)
        
        print(f"Processed batch {i//batch_size + 1}")
```

## Best Practices

1. **Always use environment variables** for credentials
2. **Implement rate limiting** when calling external APIs
3. **Use batch inserts** for better database performance
4. **Handle missing data gracefully** in transformations
5. **Test queries** on small datasets before scaling
6. **Create indexes** on frequently queried columns
7. **Log errors** for debugging ETL failures
