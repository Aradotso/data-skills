---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - create a data analytics dashboard with museum artifact data
  - integrate Harvard Art Museums API into my data pipeline
  - build a Streamlit app for art collection analytics
  - design SQL schema for museum artifact metadata
  - extract and transform Harvard museum API data
  - visualize art museum data with Plotly and Streamlit
  - set up an end-to-end data engineering project with museum data
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive data visualization using Streamlit.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides:
- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color data into SQL databases
- **SQL Analytics**: Run predefined analytical queries on structured museum data
- - **Interactive Dashboards**: Visualize query results using Streamlit and Plotly
- **Database Design**: Relational schema for artifact metadata, media, and color information

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

### 1. Harvard Art Museums API Key

Get your free API key from [Harvard Art Museums API](https://harvardartmuseums.org/collections/api).

### 2. Database Setup

Configure your MySQL or TiDB Cloud connection:

```python
# Create .env file
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### 3. Database Schema

Create the required tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    objectnumber VARCHAR(255),
    dated VARCHAR(255),
    period VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    description TEXT,
    provenance TEXT
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    imagepermissionlevel INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    saturation FLOAT,
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
streamlit run app.py
```

Access the dashboard at `http://localhost:8501`

## Key Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Fetch multiple pages with pagination
def fetch_all_artifacts(api_key, max_pages=10):
    all_artifacts = []
    for page in range(1, max_pages + 1):
        data = fetch_artifacts(api_key, page=page)
        all_artifacts.extend(data.get('records', []))
        
        # Respect rate limits
        import time
        time.sleep(0.5)
    
    return all_artifacts
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector

def transform_artifacts(raw_data):
    """Transform nested JSON into relational dataframes"""
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'objectnumber': artifact.get('objectnumber'),
            'dated': artifact.get('dated'),
            'period': artifact.get('period'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'description': artifact.get('description'),
            'provenance': artifact.get('provenance')
        })
        
        # Extract media data
        if artifact.get('primaryimageurl'):
            media_records.append({
                'artifact_id': artifact.get('id'),
                'baseimageurl': artifact.get('baseimageurl'),
                'primaryimageurl': artifact.get('primaryimageurl'),
                'iiifbaseuri': artifact.get('iiifbaseuri'),
                'imagepermissionlevel': artifact.get('imagepermissionlevel', 0)
            })
        
        # Extract color data
        for color in artifact.get('colors', []):
            color_records.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'saturation': color.get('saturation'),
                'percent': color.get('percent')
            })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )

def load_to_database(metadata_df, media_df, colors_df, db_config):
    """Load transformed data into MySQL database"""
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    
    # Insert metadata (batch insert)
    for _, row in metadata_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, department, 
             objectnumber, dated, period, technique, medium, dimensions, 
             creditline, description, provenance)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """, tuple(row))
    
    # Insert media
    for _, row in media_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmedia 
            (artifact_id, baseimageurl, primaryimageurl, iiifbaseuri, imagepermissionlevel)
            VALUES (%s, %s, %s, %s, %s)
        """, tuple(row))
    
    # Insert colors
    for _, row in colors_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, hue, saturation, percent)
            VALUES (%s, %s, %s, %s, %s, %s)
        """, tuple(row))
    
    conn.commit()
    cursor.close()
    conn.close()
```

### 3. Analytical SQL Queries

```python
ANALYTICAL_QUERIES = {
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
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count 
        FROM artifactmetadata 
        GROUP BY department 
        ORDER BY artifact_count DESC
    """,
    
    "Media Availability": """
        SELECT 
            COUNT(DISTINCT m.artifact_id) as artifacts_with_media,
            COUNT(DISTINCT a.id) as total_artifacts,
            ROUND(COUNT(DISTINCT m.artifact_id) * 100.0 / COUNT(DISTINCT a.id), 2) as percentage
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.id = m.artifact_id
    """,
    
    "Top 10 Color Usage": """
        SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "Artifacts with Complete Data": """
        SELECT a.id, a.title, a.culture, a.century, 
               COUNT(DISTINCT m.media_id) as media_count,
               COUNT(DISTINCT c.color_id) as color_count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.id = m.artifact_id
        LEFT JOIN artifactcolors c ON a.id = c.artifact_id
        GROUP BY a.id, a.title, a.culture, a.century
        HAVING media_count > 0 AND color_count > 0
        ORDER BY media_count DESC, color_count DESC
        LIMIT 20
    """
}

def run_analytics_query(query, db_config):
    """Execute analytical query and return results as DataFrame"""
    conn = mysql.connector.connect(**db_config)
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password", 
                                     value=os.getenv('HARVARD_API_KEY', ''))
    
    # Database connection
    db_config = {
        'host': os.getenv('DB_HOST'),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    
    # ETL Section
    st.header("📥 Data Collection & ETL")
    if st.button("Fetch & Load Artifacts"):
        with st.spinner("Fetching data from API..."):
            raw_data = fetch_all_artifacts(api_key, max_pages=5)
            st.success(f"Fetched {len(raw_data)} artifacts")
            
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(raw_data)
            st.success("Data transformed successfully")
            
        with st.spinner("Loading to database..."):
            load_to_database(metadata_df, media_df, colors_df, db_config)
            st.success("Data loaded to database")
    
    # Analytics Section
    st.header("📊 SQL Analytics")
    query_name = st.selectbox("Select Analysis", list(ANALYTICAL_QUERIES.keys()))
    
    if st.button("Run Query"):
        query = ANALYTICAL_QUERIES[query_name]
        st.code(query, language='sql')
        
        df = run_analytics_query(query, db_config)
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                        title=query_name)
            st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Incremental Data Loading

```python
def get_last_artifact_id(db_config):
    """Get the last loaded artifact ID"""
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()[0]
    conn.close()
    return result or 0

def fetch_new_artifacts(api_key, last_id):
    """Fetch only new artifacts since last load"""
    url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'size': 100,
        'sort': 'id',
        'sortorder': 'asc',
        'id': f'>{last_id}'
    }
    response = requests.get(url, params=params)
    return response.json().get('records', [])
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_fetch_artifacts(api_key, page=1, max_retries=3):
    """Fetch artifacts with retry logic"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except Exception as e:
            logger.error(f"Attempt {attempt + 1} failed: {e}")
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

## Troubleshooting

### API Rate Limiting
- Harvard API has rate limits; add delays between requests
- Use `time.sleep(0.5)` between pagination calls

### Database Connection Issues
- Verify credentials in `.env` file
- Ensure database exists and user has proper permissions
- Check firewall rules for remote database connections

### Missing Data Fields
- API responses may have null/missing fields
- Use `.get()` method with defaults: `artifact.get('field', 'N/A')`
- Handle null values in SQL with `WHERE field IS NOT NULL`

### Memory Issues with Large Datasets
- Process data in batches instead of loading everything at once
- Use `cursor.executemany()` for bulk inserts
- Clear DataFrames after loading: `del df`

### Streamlit Performance
- Cache database queries with `@st.cache_data`
- Limit result set sizes in SQL queries
- Use pagination for large result tables
