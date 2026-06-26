---
name: harvard-art-museums-data-engineering-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build a data pipeline with Harvard Art Museums API
  - set up ETL for Harvard artifacts collection
  - create analytics dashboard for museum data
  - extract and analyze Harvard Art Museums data
  - build Streamlit app for artifact analytics
  - query Harvard Art Museums API with Python
  - implement ETL pipeline for museum artifacts
  - visualize museum collection data with Plotly
---

# Harvard Art Museums Data Engineering & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates a complete data engineering workflow: extracting data from the Harvard Art Museums API, transforming it into relational tables, loading into SQL databases, and building interactive analytics dashboards with Streamlit.

## What This Project Does

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON into structured relational tables (metadata, media, colors)
- **SQL Storage**: Loads data into MySQL/TiDB Cloud with proper schema design
- **Analytics**: Executes 20+ predefined analytical queries
- **Visualization**: Interactive Plotly dashboards for data exploration

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
export HARVARD_API_KEY="your_api_key"
export DB_HOST="your_db_host"
export DB_USER="your_db_user"
export DB_PASSWORD="your_db_password"
export DB_NAME="harvard_artifacts"
```

**Requirements typically include:**
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## API Integration

### Getting API Key

1. Register at [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Store key in environment variable or `.env` file

### Fetching Artifacts

```python
import requests
import os

API_KEY = os.getenv("HARVARD_API_KEY")
BASE_URL = "https://api.harvardartmuseums.org/object"

def fetch_artifacts(size=100, page=1):
    """Fetch artifacts with pagination"""
    params = {
        "apikey": API_KEY,
        "size": size,
        "page": page,
        "hasimage": 1  # Only artifacts with images
    }
    
    response = requests.get(BASE_URL, params=params)
    response.raise_for_status()
    return response.json()

# Example usage
data = fetch_artifacts(size=50, page=1)
artifacts = data.get("records", [])
total_records = data.get("info", {}).get("totalrecords", 0)
print(f"Fetched {len(artifacts)} of {total_records} artifacts")
```

### Handling Pagination

```python
def fetch_all_artifacts(max_records=500):
    """Fetch multiple pages of artifacts"""
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        data = fetch_artifacts(size=size, page=page)
        records = data.get("records", [])
        
        if not records:
            break
            
        all_artifacts.extend(records)
        page += 1
        
        # Respect rate limits
        import time
        time.sleep(0.5)
    
    return all_artifacts[:max_records]
```

## ETL Pipeline

### Extract: Parse API Response

```python
import pandas as pd

def extract_metadata(artifacts):
    """Extract core artifact metadata"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            "object_id": artifact.get("objectid"),
            "title": artifact.get("title"),
            "culture": artifact.get("culture"),
            "period": artifact.get("period"),
            "century": artifact.get("century"),
            "classification": artifact.get("classification"),
            "department": artifact.get("department"),
            "dated": artifact.get("dated"),
            "division": artifact.get("division")
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def extract_media(artifacts):
    """Extract media/image information"""
    media_list = []
    
    for artifact in artifacts:
        object_id = artifact.get("objectid")
        images = artifact.get("images", [])
        
        for img in images:
            media = {
                "object_id": object_id,
                "image_id": img.get("imageid"),
                "base_url": img.get("baseimageurl"),
                "width": img.get("width"),
                "height": img.get("height"),
                "format": img.get("format")
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def extract_colors(artifacts):
    """Extract color palette information"""
    color_list = []
    
    for artifact in artifacts:
        object_id = artifact.get("objectid")
        colors = artifact.get("colors", [])
        
        for color in colors:
            color_data = {
                "object_id": object_id,
                "color": color.get("color"),
                "spectrum": color.get("spectrum"),
                "hue": color.get("hue"),
                "percent": color.get("percent")
            }
            color_list.append(color_data)
    
    return pd.DataFrame(color_list)
```

### Transform: Clean and Validate

```python
def transform_metadata(df):
    """Clean and transform metadata"""
    # Remove duplicates
    df = df.drop_duplicates(subset=["object_id"])
    
    # Handle nulls
    df = df.fillna("")
    
    # Truncate long text fields
    df["title"] = df["title"].str[:255]
    df["culture"] = df["culture"].str[:100]
    
    return df

def transform_media(df):
    """Clean media data"""
    # Remove rows without image_id
    df = df.dropna(subset=["image_id"])
    
    # Convert dimensions to integers
    df["width"] = pd.to_numeric(df["width"], errors="coerce").fillna(0).astype(int)
    df["height"] = pd.to_numeric(df["height"], errors="coerce").fillna(0).astype(int)
    
    return df
```

### Load: Insert into SQL Database

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv("DB_HOST"),
        user=os.getenv("DB_USER"),
        password=os.getenv("DB_PASSWORD"),
        database=os.getenv("DB_NAME")
    )

def create_tables(connection):
    """Create database schema"""
    cursor = connection.cursor()
    
    # Metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            object_id INT PRIMARY KEY,
            title VARCHAR(255),
            culture VARCHAR(100),
            period VARCHAR(100),
            century VARCHAR(50),
            classification VARCHAR(100),
            department VARCHAR(100),
            dated VARCHAR(100),
            division VARCHAR(100)
        )
    """)
    
    # Media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            object_id INT,
            image_id INT,
            base_url TEXT,
            width INT,
            height INT,
            format VARCHAR(50),
            FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
        )
    """)
    
    # Colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            object_id INT,
            color VARCHAR(50),
            spectrum VARCHAR(50),
            hue VARCHAR(50),
            percent FLOAT,
            FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
        )
    """)
    
    connection.commit()
    cursor.close()

def load_to_database(df, table_name, connection):
    """Batch insert dataframe into database"""
    cursor = connection.cursor()
    
    # Prepare insert statement
    cols = ", ".join(df.columns)
    placeholders = ", ".join(["%s"] * len(df.columns))
    insert_sql = f"INSERT IGNORE INTO {table_name} ({cols}) VALUES ({placeholders})"
    
    # Convert dataframe to list of tuples
    data = [tuple(row) for row in df.values]
    
    # Batch insert
    cursor.executemany(insert_sql, data)
    connection.commit()
    cursor.close()
    
    print(f"Inserted {cursor.rowcount} rows into {table_name}")
```

## Streamlit Analytics Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Select Feature",
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
    
    num_records = st.number_input("Number of records to fetch", 10, 1000, 100)
    
    if st.button("Fetch Data"):
        with st.spinner("Fetching artifacts..."):
            artifacts = fetch_all_artifacts(max_records=num_records)
            
            # ETL process
            metadata_df = transform_metadata(extract_metadata(artifacts))
            media_df = transform_media(extract_media(artifacts))
            colors_df = extract_colors(artifacts)
            
            # Load to database
            conn = get_db_connection()
            create_tables(conn)
            load_to_database(metadata_df, "artifactmetadata", conn)
            load_to_database(media_df, "artifactmedia", conn)
            load_to_database(colors_df, "artifactcolors", conn)
            conn.close()
            
            st.success(f"✅ Loaded {len(metadata_df)} artifacts into database")

if __name__ == "__main__":
    main()
```

### SQL Analytics Queries

```python
ANALYTICS_QUERIES = {
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century != ''
        GROUP BY century
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Top Cultures": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department != ''
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "Color Palette Analysis": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 20
    """,
    
    "Image Dimensions Analysis": """
        SELECT 
            CASE 
                WHEN width < 500 THEN 'Small'
                WHEN width < 1000 THEN 'Medium'
                ELSE 'Large'
            END as size_category,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY size_category
    """
}

def show_sql_analytics():
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        conn = get_db_connection()
        query = ANALYTICS_QUERIES[query_name]
        
        df = pd.read_sql(query, conn)
        conn.close()
        
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                        title=query_name)
            st.plotly_chart(fig, use_container_width=True)
```

## Common Patterns

### Environment Configuration

```python
from dotenv import load_dotenv
import os

load_dotenv()

CONFIG = {
    "api_key": os.getenv("HARVARD_API_KEY"),
    "db_host": os.getenv("DB_HOST"),
    "db_user": os.getenv("DB_USER"),
    "db_password": os.getenv("DB_PASSWORD"),
    "db_name": os.getenv("DB_NAME", "harvard_artifacts")
}
```

### Error Handling

```python
def safe_api_call(url, params, max_retries=3):
    """API call with retry logic"""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                st.error(f"API call failed: {e}")
                return None
            time.sleep(2 ** attempt)  # Exponential backoff
```

## Troubleshooting

**API Rate Limiting**: Add delays between requests
```python
import time
time.sleep(0.5)  # 500ms between requests
```

**Database Connection Issues**: Check credentials and network
```python
try:
    conn = get_db_connection()
    conn.ping(reconnect=True)
except Error as e:
    st.error(f"Database connection failed: {e}")
```

**Missing Data in API Response**: Always use `.get()` with defaults
```python
title = artifact.get("title", "Untitled")
images = artifact.get("images", [])
```

**Memory Issues with Large Datasets**: Use chunking
```python
chunk_size = 100
for i in range(0, len(data), chunk_size):
    chunk = data[i:i+chunk_size]
    load_to_database(pd.DataFrame(chunk), table_name, conn)
```
