---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard with Harvard artifacts
  - extract and analyze museum collection data
  - implement data engineering pipeline with museum API
  - visualize Harvard Art Museums collection insights
  - set up SQL database for artifacts metadata
  - query and analyze art museum collection data
  - build Streamlit app for museum data visualization
---

# Harvard Art Museums ETL & Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build complete data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualizations with Streamlit.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load museum data into relational SQL databases
- **Database Design**: Structured schema with `artifactmetadata`, `artifactmedia`, and `artifactcolors` tables
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Interactive Dashboards**: Streamlit-based visualization with Plotly charts

**Architecture Flow**: API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Core Dependencies**:
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### 1. Harvard Art Museums API Key

Get your free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

### 2. Database Setup

Configure MySQL or TiDB Cloud connection. Create a `.env` file:

```bash
# .env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### 3. Initialize Database Schema

```sql
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    url TEXT,
    technique VARCHAR(500),
    medium VARCHAR(500)
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    baseimageurl TEXT,
    primaryimageurl TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    return response.json()

def collect_multiple_pages(num_pages=5):
    """Collect artifacts from multiple pages with pagination"""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(page=page, size=100)
        all_artifacts.extend(data.get('records', []))
    
    return all_artifacts
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

def transform_artifacts(raw_data):
    """Transform raw API data into structured DataFrames"""
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium')
        })
        
        # Extract media information
        if artifact.get('images'):
            for img in artifact['images']:
                media_records.append({
                    'artifact_id': artifact.get('id'),
                    'media_type': 'image',
                    'baseimageurl': img.get('baseimageurl'),
                    'primaryimageurl': artifact.get('primaryimageurl')
                })
        
        # Extract color information
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_records.append({
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'percentage': color.get('percent')
                })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )

def load_to_database(df_metadata, df_media, df_colors):
    """Load transformed data into SQL database"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=os.getenv('DB_PORT'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
        if connection.is_connected():
            cursor = connection.cursor()
            
            # Insert metadata
            for _, row in df_metadata.iterrows():
                cursor.execute("""
                    INSERT INTO artifactmetadata 
                    (id, title, culture, period, century, classification, 
                     department, dated, url, technique, medium)
                    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                    ON DUPLICATE KEY UPDATE title=VALUES(title)
                """, tuple(row))
            
            # Insert media
            for _, row in df_media.iterrows():
                cursor.execute("""
                    INSERT INTO artifactmedia 
                    (artifact_id, media_type, baseimageurl, primaryimageurl)
                    VALUES (%s, %s, %s, %s)
                """, tuple(row))
            
            # Insert colors
            for _, row in df_colors.iterrows():
                cursor.execute("""
                    INSERT INTO artifactcolors 
                    (artifact_id, color, spectrum, percentage)
                    VALUES (%s, %s, %s, %s)
                """, tuple(row))
            
            connection.commit()
            print(f"Loaded {len(df_metadata)} artifacts successfully")
            
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

### 3. Analytical SQL Queries

```python
ANALYTICS_QUERIES = {
    "Top 10 Cultures by Artifact Count": """
        SELECT culture, COUNT(*) as artifact_count 
        FROM artifactmetadata 
        WHERE culture IS NOT NULL 
        GROUP BY culture 
        ORDER BY artifact_count DESC 
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE century IS NOT NULL 
        GROUP BY century 
        ORDER BY century
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as total 
        FROM artifactmetadata 
        GROUP BY department 
        ORDER BY total DESC
    """,
    
    "Artifacts with Images": """
        SELECT 
            COUNT(DISTINCT a.id) as with_images,
            (SELECT COUNT(*) FROM artifactmetadata) as total,
            ROUND(COUNT(DISTINCT a.id) * 100.0 / 
                  (SELECT COUNT(*) FROM artifactmetadata), 2) as percentage
        FROM artifactmetadata a
        JOIN artifactmedia m ON a.id = m.artifact_id
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as frequency 
        FROM artifactcolors 
        GROUP BY color 
        ORDER BY frequency DESC 
        LIMIT 10
    """,
    
    "Average Color Percentage by Spectrum": """
        SELECT spectrum, AVG(percentage) as avg_percentage 
        FROM artifactcolors 
        WHERE spectrum IS NOT NULL 
        GROUP BY spectrum 
        ORDER BY avg_percentage DESC
    """
}

def execute_query(query):
    """Execute SQL query and return results as DataFrame"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    df = pd.read_sql(query, connection)
    connection.close()
    return df
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar for data collection
    with st.sidebar:
        st.header("📥 Data Collection")
        num_pages = st.number_input("Number of pages to fetch", 1, 10, 5)
        
        if st.button("Fetch & Load Data"):
            with st.spinner("Fetching artifacts..."):
                artifacts = collect_multiple_pages(num_pages)
                df_meta, df_media, df_colors = transform_artifacts(artifacts)
                load_to_database(df_meta, df_media, df_colors)
                st.success(f"Loaded {len(df_meta)} artifacts!")
    
    # Main analytics section
    st.header("📊 Analytics Queries")
    
    query_name = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Run Query"):
        with st.spinner("Executing query..."):
            query = ANALYTICS_QUERIES[query_name]
            results = execute_query(query)
            
            st.subheader("Results")
            st.dataframe(results)
            
            # Auto-generate visualization
            if len(results.columns) >= 2:
                fig = px.bar(
                    results,
                    x=results.columns[0],
                    y=results.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Pattern 1: Incremental Data Loading

```python
def get_latest_artifact_id():
    """Get the most recent artifact ID in database"""
    connection = mysql.connector.connect(...)
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    max_id = cursor.fetchone()[0]
    connection.close()
    return max_id or 0

def incremental_load():
    """Load only new artifacts"""
    last_id = get_latest_artifact_id()
    # Fetch artifacts with id > last_id
    # Transform and load
```

### Pattern 2: Error Handling for API Limits

```python
import time

def safe_api_call(page, retries=3, delay=2):
    """API call with retry logic"""
    for attempt in range(retries):
        try:
            return fetch_artifacts(page)
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:  # Rate limit
                wait_time = delay * (2 ** attempt)
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Pattern 3: Custom Analytics

```python
def create_custom_query(filters):
    """Build dynamic SQL query based on filters"""
    base_query = "SELECT * FROM artifactmetadata WHERE 1=1"
    
    if filters.get('culture'):
        base_query += f" AND culture = '{filters['culture']}'"
    if filters.get('century'):
        base_query += f" AND century = '{filters['century']}'"
    
    return base_query
```

## Troubleshooting

**API Key Issues**:
- Ensure `.env` file contains valid `HARVARD_API_KEY`
- Check API rate limits (default: 2500 requests/day)

**Database Connection Errors**:
- Verify database credentials in `.env`
- Ensure database and tables are created
- Check firewall rules for remote databases (TiDB Cloud)

**Data Loading Failures**:
- Check for NULL values in required fields
- Ensure artifact IDs are unique
- Verify foreign key constraints

**Streamlit Performance**:
- Use `@st.cache_data` for expensive operations
- Limit initial data loads
- Add pagination for large result sets

```python
@st.cache_data(ttl=3600)
def cached_query(query):
    return execute_query(query)
```
