---
name: harvard-artifacts-etl-analytics
description: End-to-end ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build etl pipeline for harvard art museums
  - connect to harvard art museums api
  - create analytics dashboard with streamlit
  - set up sql database for artifact data
  - extract harvard museum collection data
  - visualize art museum analytics
  - design artifact metadata pipeline
  - query harvard art collections
---

# Harvard Artifacts Collection Data Engineering & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build end-to-end data engineering pipelines using the Harvard Art Museums API. It covers ETL operations, SQL database design, analytics queries, and interactive Streamlit dashboards for artifact data visualization.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App demonstrates a complete data pipeline:

1. **Extract**: Fetch artifact data from Harvard Art Museums API with pagination
2. **Transform**: Parse nested JSON into normalized relational structures
3. **Load**: Batch insert data into MySQL/TiDB Cloud
4. **Analyze**: Execute SQL analytics queries on artifact metadata, media, and colors
5. **Visualize**: Display results in interactive Streamlit dashboards with Plotly charts

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```env
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=gateway01.us-east-1.prod.aws.tidbcloud.com
DB_PORT=4000
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### API Key Setup

Get your free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api):

```python
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')
```

### Database Setup

```sql
-- Create database
CREATE DATABASE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    objectnumber VARCHAR(100),
    dated VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    division VARCHAR(255),
    contact VARCHAR(255),
    description TEXT,
    provenance TEXT,
    verificationlevel INT,
    primaryimageurl TEXT,
    url TEXT
);

-- Artifact media table
CREATE TABLE artifactmedia (
    mediaid INT PRIMARY KEY AUTO_INCREMENT,
    artifactid INT,
    baseimageurl TEXT,
    format VARCHAR(50),
    height INT,
    width INT,
    FOREIGN KEY (artifactid) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    colorid INT PRIMARY KEY AUTO_INCREMENT,
    artifactid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent DECIMAL(5,2),
    FOREIGN KEY (artifactid) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import pandas as pd
from typing import List, Dict

def fetch_artifacts(api_key: str, num_pages: int = 5, page_size: int = 100) -> List[Dict]:
    """
    Fetch artifact data from Harvard Art Museums API with pagination.
    
    Args:
        api_key: Harvard API key
        num_pages: Number of pages to fetch
        page_size: Records per page (max 100)
    
    Returns:
        List of artifact dictionaries
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
            response = requests.get(base_url, params=params)
            response.raise_for_status()
            data = response.json()
            
            artifacts = data.get('records', [])
            all_artifacts.extend(artifacts)
            
            print(f"Fetched page {page}: {len(artifacts)} artifacts")
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return all_artifacts
```

### Transform: Normalize JSON to DataFrames

```python
def transform_artifacts(raw_data: List[Dict]) -> tuple:
    """
    Transform raw API data into normalized dataframes.
    
    Returns:
        (metadata_df, media_df, colors_df)
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'objectnumber': artifact.get('objectnumber'),
            'dated': artifact.get('dated'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'division': artifact.get('division'),
            'contact': artifact.get('contact'),
            'description': artifact.get('description'),
            'provenance': artifact.get('provenance'),
            'verificationlevel': artifact.get('verificationlevel'),
            'primaryimageurl': artifact.get('primaryimageurl'),
            'url': artifact.get('url')
        }
        metadata_records.append(metadata)
        
        # Extract media (images)
        artifact_id = artifact.get('id')
        for image in artifact.get('images', []):
            media = {
                'artifactid': artifact_id,
                'baseimageurl': image.get('baseimageurl'),
                'format': image.get('format'),
                'height': image.get('height'),
                'width': image.get('width')
            }
            media_records.append(media)
        
        # Extract colors
        for color_info in artifact.get('colors', []):
            color = {
                'artifactid': artifact_id,
                'color': color_info.get('color'),
                'spectrum': color_info.get('spectrum'),
                'hue': color_info.get('hue'),
                'percent': color_info.get('percent')
            }
            color_records.append(color)
    
    metadata_df = pd.DataFrame(metadata_records)
    media_df = pd.DataFrame(media_records)
    colors_df = pd.DataFrame(color_records)
    
    return metadata_df, media_df, colors_df
```

### Load: Batch Insert to SQL

```python
import mysql.connector
from typing import List

def get_db_connection():
    """Create database connection using environment variables."""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_metadata(df: pd.DataFrame, batch_size: int = 100):
    """Load artifact metadata with batch inserts."""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, department, 
         objectnumber, dated, technique, medium, dimensions, creditline,
         division, contact, description, provenance, verificationlevel,
         primaryimageurl, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    records = df.to_records(index=False)
    data = [tuple(row) for row in records]
    
    for i in range(0, len(data), batch_size):
        batch = data[i:i + batch_size]
        cursor.executemany(insert_query, batch)
        conn.commit()
        print(f"Inserted batch {i//batch_size + 1}")
    
    cursor.close()
    conn.close()

def load_media(df: pd.DataFrame):
    """Load artifact media data."""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia (artifactid, baseimageurl, format, height, width)
        VALUES (%s, %s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.to_records(index=False)]
    cursor.executemany(insert_query, data)
    conn.commit()
    
    cursor.close()
    conn.close()

def load_colors(df: pd.DataFrame):
    """Load artifact color data."""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactcolors (artifactid, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.to_records(index=False)]
    cursor.executemany(insert_query, data)
    conn.commit()
    
    cursor.close()
    conn.close()
```

## Analytics Queries

### Sample SQL Queries

```python
# Analytics query library
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 20
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY century
    """,
    
    "Artifacts with Images": """
        SELECT 
            CASE WHEN primaryimageurl IS NOT NULL THEN 'Has Image' ELSE 'No Image' END as image_status,
            COUNT(*) as count
        FROM artifactmetadata
        GROUP BY image_status
    """,
    
    "Most Common Colors": """
        SELECT color, COUNT(*) as frequency
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "Artifacts by Department": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "Average Color Distribution": """
        SELECT color, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY avg_percent DESC
        LIMIT 10
    """,
    
    "Media Format Distribution": """
        SELECT format, COUNT(*) as count
        FROM artifactmedia
        WHERE format IS NOT NULL
        GROUP BY format
        ORDER BY count DESC
    """
}

def execute_query(query: str) -> pd.DataFrame:
    """Execute SQL query and return results as DataFrame."""
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["ETL Pipeline", "Analytics Dashboard", "Data Explorer"]
    )
    
    if page == "ETL Pipeline":
        show_etl_page()
    elif page == "Analytics Dashboard":
        show_analytics_page()
    else:
        show_explorer_page()

def show_etl_page():
    """ETL pipeline control page."""
    st.header("📥 ETL Pipeline")
    
    col1, col2 = st.columns(2)
    
    with col1:
        num_pages = st.number_input("Number of Pages", min_value=1, max_value=50, value=5)
        page_size = st.number_input("Page Size", min_value=10, max_value=100, value=100)
    
    with col2:
        api_key = st.text_input("API Key", type="password", value=os.getenv('HARVARD_API_KEY', ''))
    
    if st.button("🚀 Run ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            raw_data = fetch_artifacts(api_key, num_pages, page_size)
            st.success(f"Fetched {len(raw_data)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(raw_data)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            load_metadata(metadata_df)
            load_media(media_df)
            load_colors(colors_df)
            st.success("Data loaded to database")
        
        st.balloons()

def show_analytics_page():
    """Analytics dashboard with predefined queries."""
    st.header("📊 Analytics Dashboard")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        query = ANALYTICS_QUERIES[query_name]
        
        with st.spinner("Executing query..."):
            df = execute_query(query)
        
        st.subheader("Query Results")
        st.dataframe(df, use_container_width=True)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(
                df,
                x=df.columns[0],
                y=df.columns[1],
                title=query_name,
                color=df.columns[1],
                color_continuous_scale='Viridis'
            )
            st.plotly_chart(fig, use_container_width=True)

def show_explorer_page():
    """Custom SQL query interface."""
    st.header("🔍 Data Explorer")
    
    custom_query = st.text_area(
        "Enter SQL Query",
        height=150,
        placeholder="SELECT * FROM artifactmetadata LIMIT 10"
    )
    
    if st.button("Execute"):
        try:
            df = execute_query(custom_query)
            st.dataframe(df, use_container_width=True)
            
            st.download_button(
                "Download CSV",
                df.to_csv(index=False),
                "query_results.csv",
                "text/csv"
            )
        except Exception as e:
            st.error(f"Query error: {e}")

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

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(api_key: str, num_pages: int, delay: float = 0.5):
    """Fetch data with rate limiting to avoid API throttling."""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        artifacts = fetch_single_page(api_key, page)
        all_artifacts.extend(artifacts)
        
        time.sleep(delay)  # Rate limit
    
    return all_artifacts
```

### Error Handling in ETL

```python
def safe_etl_pipeline(api_key: str, num_pages: int):
    """ETL pipeline with comprehensive error handling."""
    try:
        # Extract
        raw_data = fetch_artifacts(api_key, num_pages)
        if not raw_data:
            raise ValueError("No data fetched from API")
        
        # Transform
        metadata_df, media_df, colors_df = transform_artifacts(raw_data)
        
        # Validate
        assert len(metadata_df) > 0, "Metadata is empty"
        
        # Load
        load_metadata(metadata_df)
        load_media(media_df)
        load_colors(colors_df)
        
        return True
        
    except requests.exceptions.RequestException as e:
        print(f"API Error: {e}")
        return False
    except mysql.connector.Error as e:
        print(f"Database Error: {e}")
        return False
    except Exception as e:
        print(f"Unexpected Error: {e}")
        return False
```

## Troubleshooting

### Database Connection Issues

```python
# Test database connection
def test_db_connection():
    """Test if database is reachable."""
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        cursor.close()
        conn.close()
        print("✅ Database connection successful")
        return True
    except Exception as e:
        print(f"❌ Database connection failed: {e}")
        return False
```

### API Key Validation

```python
def validate_api_key(api_key: str) -> bool:
    """Validate Harvard API key."""
    test_url = f"https://api.harvardartmuseums.org/object?apikey={api_key}&size=1"
    
    try:
        response = requests.get(test_url)
        return response.status_code == 200
    except:
        return False
```

### Handle Missing Data

```python
def safe_get(dictionary: dict, key: str, default=None):
    """Safely extract values from nested dictionaries."""
    return dictionary.get(key, default)

# Use in transformation
metadata = {
    'id': safe_get(artifact, 'id'),
    'title': safe_get(artifact, 'title', 'Untitled'),
    'culture': safe_get(artifact, 'culture', 'Unknown')
}
```

This skill provides everything needed to build, deploy, and extend a production-ready ETL pipeline and analytics dashboard using the Harvard Art Museums API.
