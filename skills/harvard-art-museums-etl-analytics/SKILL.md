---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline using Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for museum artifact data
  - fetch and analyze Harvard Art Museums collection
  - create a data engineering app with Streamlit
  - extract museum data from Harvard API
  - set up SQL analytics for art collection
  - visualize artifact metadata with interactive dashboards
  - implement batch data ingestion for museum collections
  - analyze art collection data by culture and century
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

This project implements a complete data engineering pipeline that:
- Fetches artifact data from the Harvard Art Museums API with pagination
- Performs ETL operations to transform nested JSON into relational tables
- Loads data into MySQL/TiDB Cloud databases
- Provides 20+ predefined SQL analytics queries
- Visualizes results through an interactive Streamlit dashboard

The application demonstrates real-world patterns for API integration, data transformation, SQL database design, and business intelligence visualization.

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Install core dependencies if requirements.txt is unavailable
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Register for a free API key
3. Add to your `.env` file

### Database Setup

```python
import mysql.connector
from dotenv import load_dotenv
import os

load_dotenv()

# Create database connection
conn = mysql.connector.connect(
    host=os.getenv('DB_HOST'),
    port=int(os.getenv('DB_PORT', 3306)),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD')
)

cursor = conn.cursor()

# Create database
cursor.execute(f"CREATE DATABASE IF NOT EXISTS {os.getenv('DB_NAME')}")
cursor.execute(f"USE {os.getenv('DB_NAME')}")

# Create tables
cursor.execute("""
    CREATE TABLE IF NOT EXISTS artifactmetadata (
        id INT PRIMARY KEY,
        title VARCHAR(500),
        culture VARCHAR(200),
        century VARCHAR(100),
        classification VARCHAR(200),
        department VARCHAR(200),
        dated VARCHAR(200),
        medium VARCHAR(500),
        creditline TEXT,
        provenance TEXT,
        url VARCHAR(500)
    )
""")

cursor.execute("""
    CREATE TABLE IF NOT EXISTS artifactmedia (
        media_id INT AUTO_INCREMENT PRIMARY KEY,
        artifact_id INT,
        baseimageurl VARCHAR(500),
        iiifbaseuri VARCHAR(500),
        imagetype VARCHAR(100),
        FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
    )
""")

cursor.execute("""
    CREATE TABLE IF NOT EXISTS artifactcolors (
        color_id INT AUTO_INCREMENT PRIMARY KEY,
        artifact_id INT,
        color VARCHAR(50),
        spectrum VARCHAR(100),
        percentage FLOAT,
        FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
    )
""")

conn.commit()
cursor.close()
conn.close()
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py
```

The app will open in your browser at `http://localhost:8501`

## Core ETL Pipeline

### 1. Extract Data from API

```python
import requests
import pandas as pd

def fetch_artifacts(api_key, num_records=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    base_url = "https://api.harvardartmuseums.org/object"
    artifacts = []
    page = 1
    size = 100  # Max per page
    
    while len(artifacts) < num_records:
        params = {
            'apikey': api_key,
            'size': min(size, num_records - len(artifacts)),
            'page': page
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code != 200:
            print(f"Error: {response.status_code}")
            break
            
        data = response.json()
        records = data.get('records', [])
        
        if not records:
            break
            
        artifacts.extend(records)
        page += 1
        
    return artifacts[:num_records]
```

### 2. Transform Data

```python
def transform_artifacts(artifacts):
    """Transform nested JSON into relational dataframes"""
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
            'dated': artifact.get('dated'),
            'medium': artifact.get('medium'),
            'creditline': artifact.get('creditline'),
            'provenance': artifact.get('provenance'),
            'url': artifact.get('url')
        }
        metadata_list.append(metadata)
        
        # Extract media/images
        primary_image = artifact.get('primaryimageurl')
        if primary_image:
            media = {
                'artifact_id': artifact.get('id'),
                'baseimageurl': primary_image,
                'iiifbaseuri': artifact.get('iiifbaseuri'),
                'imagetype': 'primary'
            }
            media_list.append(media)
        
        # Extract colors
        colors = artifact.get('colors', [])
        for color in colors:
            color_entry = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'percentage': color.get('percent')
            }
            colors_list.append(color_entry)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### 3. Load Data to SQL

```python
def load_to_sql(metadata_df, media_df, colors_df, conn):
    """Batch insert dataframes into SQL database"""
    cursor = conn.cursor()
    
    # Load metadata
    for _, row in metadata_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, department, 
             dated, medium, creditline, provenance, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE
            title=VALUES(title), culture=VALUES(culture)
        """, tuple(row))
    
    # Load media
    for _, row in media_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmedia 
            (artifact_id, baseimageurl, iiifbaseuri, imagetype)
            VALUES (%s, %s, %s, %s)
        """, tuple(row))
    
    # Load colors
    for _, row in colors_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, percentage)
            VALUES (%s, %s, %s, %s)
        """, tuple(row))
    
    conn.commit()
    cursor.close()
```

## Key Analytics Queries

### Query 1: Artifacts by Culture

```python
def get_artifacts_by_culture(conn):
    """Count artifacts grouped by culture"""
    query = """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 20
    """
    return pd.read_sql(query, conn)
```

### Query 2: Artifacts with Media

```python
def get_artifacts_with_media(conn):
    """Find artifacts that have associated media"""
    query = """
        SELECT m.title, m.culture, m.century, a.baseimageurl
        FROM artifactmetadata m
        INNER JOIN artifactmedia a ON m.id = a.artifact_id
        WHERE a.baseimageurl IS NOT NULL
        LIMIT 50
    """
    return pd.read_sql(query, conn)
```

### Query 3: Top Colors Analysis

```python
def get_color_distribution(conn):
    """Analyze color usage across artifacts"""
    query = """
        SELECT c.color, COUNT(DISTINCT c.artifact_id) as artifact_count,
               AVG(c.percentage) as avg_percentage
        FROM artifactcolors c
        GROUP BY c.color
        ORDER BY artifact_count DESC
        LIMIT 15
    """
    return pd.read_sql(query, conn)
```

### Query 4: Artifacts by Century

```python
def get_artifacts_by_century(conn):
    """Count artifacts by time period"""
    query = """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """
    return pd.read_sql(query, conn)
```

### Query 5: Department Analysis

```python
def get_department_stats(conn):
    """Artifact counts by museum department"""
    query = """
        SELECT department, 
               COUNT(*) as total_artifacts,
               COUNT(DISTINCT culture) as unique_cultures
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY total_artifacts DESC
    """
    return pd.read_sql(query, conn)
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Data Analytics")
    st.markdown("---")
    
    # Sidebar for ETL operations
    with st.sidebar:
        st.header("⚙️ ETL Pipeline")
        
        api_key = st.text_input("Harvard API Key", type="password")
        num_records = st.number_input("Records to Fetch", min_value=10, max_value=1000, value=100)
        
        if st.button("🚀 Run ETL Pipeline"):
            with st.spinner("Fetching data..."):
                artifacts = fetch_artifacts(api_key, num_records)
                
            with st.spinner("Transforming data..."):
                metadata_df, media_df, colors_df = transform_artifacts(artifacts)
                
            with st.spinner("Loading to database..."):
                conn = get_db_connection()
                load_to_sql(metadata_df, media_df, colors_df, conn)
                conn.close()
                
            st.success(f"✅ Loaded {len(artifacts)} artifacts!")
    
    # Main analytics section
    st.header("📊 Analytics Dashboard")
    
    query_option = st.selectbox(
        "Select Analysis",
        [
            "Artifacts by Culture",
            "Color Distribution",
            "Artifacts by Century",
            "Department Statistics",
            "Media Coverage"
        ]
    )
    
    conn = get_db_connection()
    
    if query_option == "Artifacts by Culture":
        df = get_artifacts_by_culture(conn)
        st.dataframe(df)
        
        fig = px.bar(df, x='culture', y='count', 
                     title='Artifact Count by Culture',
                     labels={'count': 'Number of Artifacts'})
        st.plotly_chart(fig, use_container_width=True)
        
    elif query_option == "Color Distribution":
        df = get_color_distribution(conn)
        st.dataframe(df)
        
        fig = px.bar(df, x='color', y='artifact_count',
                     title='Color Usage Across Artifacts',
                     color='avg_percentage')
        st.plotly_chart(fig, use_container_width=True)
    
    conn.close()

if __name__ == "__main__":
    main()
```

## Common Patterns

### Rate Limiting and Error Handling

```python
import time

def fetch_with_retry(url, params, max_retries=3):
    """Fetch data with exponential backoff"""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            
            if response.status_code == 429:  # Rate limit
                wait_time = 2 ** attempt
                time.sleep(wait_time)
                continue
                
            response.raise_for_status()
            return response.json()
            
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)
    
    return None
```

### Batch Processing for Large Datasets

```python
def batch_insert(df, table_name, conn, batch_size=1000):
    """Insert dataframe in batches for better performance"""
    cursor = conn.cursor()
    
    for i in range(0, len(df), batch_size):
        batch = df.iloc[i:i+batch_size]
        
        placeholders = ', '.join(['%s'] * len(batch.columns))
        columns = ', '.join(batch.columns)
        
        query = f"INSERT INTO {table_name} ({columns}) VALUES ({placeholders})"
        
        cursor.executemany(query, batch.values.tolist())
        conn.commit()
        
    cursor.close()
```

## Troubleshooting

### API Key Issues

```python
# Test API connection
def test_api_connection(api_key):
    """Verify API key is valid"""
    url = "https://api.harvardartmuseums.org/object"
    params = {'apikey': api_key, 'size': 1}
    
    response = requests.get(url, params=params)
    
    if response.status_code == 401:
        return False, "Invalid API key"
    elif response.status_code == 200:
        return True, "Connection successful"
    else:
        return False, f"Error: {response.status_code}"
```

### Database Connection Issues

```python
def verify_db_connection():
    """Test database connectivity"""
    try:
        conn = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        conn.close()
        return True, "Database connected"
    except mysql.connector.Error as e:
        return False, str(e)
```

### Memory Optimization for Large Datasets

```python
def fetch_artifacts_chunked(api_key, total_records=10000, chunk_size=1000):
    """Process large datasets in chunks to avoid memory issues"""
    for offset in range(0, total_records, chunk_size):
        artifacts = fetch_artifacts(api_key, chunk_size)
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        
        conn = get_db_connection()
        load_to_sql(metadata_df, media_df, colors_df, conn)
        conn.close()
        
        yield len(artifacts)  # Report progress
```
