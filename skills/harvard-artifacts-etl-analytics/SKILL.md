---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline with the Harvard Art Museums API
  - help me create analytics dashboards for museum artifact data
  - show me how to extract and transform Harvard museum data
  - build a data pipeline for Harvard Art Museums artifacts
  - create visualizations from Harvard museum API data
  - set up SQL analytics for art museum collections
  - how to use the Harvard artifacts collection app
  - build end-to-end data engineering pipeline for museum data
---

# Harvard Artifacts Collection ETL & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill helps you build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, analytics queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides:
- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color data into relational databases
- **SQL Analytics**: Pre-built analytical queries for insights on artifacts, cultures, centuries, and media
- **Interactive Dashboards**: Streamlit-based UI with Plotly visualizations

**Architecture Flow**: API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required Dependencies**:
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### API Key Setup

Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

Create a `.env` file:
```bash
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Database Setup

Create the database schema:

```sql
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    accessionyear INT,
    url VARCHAR(500)
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    baseimageurl VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
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

def fetch_artifacts(num_records=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = 'https://api.harvardartmuseums.org/object'
    
    all_records = []
    page = 1
    size = 100  # Max records per request
    
    while len(all_records) < num_records:
        params = {
            'apikey': api_key,
            'size': min(size, num_records - len(all_records)),
            'page': page,
            'hasimage': 1  # Only fetch artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            
            if not records:
                break
                
            all_records.extend(records)
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return all_records[:num_records]
```

### 2. ETL Transformation

```python
import pandas as pd

def transform_artifacts(raw_data):
    """Transform nested JSON into relational dataframes"""
    
    # Metadata transformation
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', ''),
            'culture': artifact.get('culture', ''),
            'century': artifact.get('century', ''),
            'classification': artifact.get('classification', ''),
            'department': artifact.get('department', ''),
            'dated': artifact.get('dated', ''),
            'accessionyear': artifact.get('accessionyear'),
            'url': artifact.get('url', '')
        }
        metadata_list.append(metadata)
        
        # Extract media
        for image in artifact.get('images', []):
            media = {
                'artifact_id': artifact.get('id'),
                'media_type': 'image',
                'baseimageurl': image.get('baseimageurl', '')
            }
            media_list.append(media)
        
        # Extract colors
        for color in artifact.get('colors', []):
            color_data = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color', ''),
                'hue': color.get('hue', ''),
                'percent': color.get('percent', 0.0)
            }
            colors_list.append(color_data)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### 3. SQL Loading

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(metadata_df, media_df, colors_df):
    """Batch insert data into MySQL database"""
    
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
        cursor = connection.cursor()
        
        # Insert metadata (batch insert)
        metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, dated, accessionyear, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        metadata_values = metadata_df.values.tolist()
        cursor.executemany(metadata_query, metadata_values)
        
        # Insert media
        media_query = """
        INSERT INTO artifactmedia (artifact_id, media_type, baseimageurl)
        VALUES (%s, %s, %s)
        """
        media_values = media_df.values.tolist()
        cursor.executemany(media_query, media_values)
        
        # Insert colors
        colors_query = """
        INSERT INTO artifactcolors (artifact_id, color, hue, percent)
        VALUES (%s, %s, %s, %s)
        """
        colors_values = colors_df.values.tolist()
        cursor.executemany(colors_query, colors_values)
        
        connection.commit()
        print(f"Successfully loaded {len(metadata_df)} artifacts")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

### 4. Analytics Queries

```python
# Sample analytical SQL queries

ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Most Common Colors": """
        SELECT color, COUNT(*) as count, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Artifacts with Images": """
        SELECT a.classification, COUNT(DISTINCT a.id) as artifact_count
        FROM artifactmetadata a
        JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY a.classification
        ORDER BY artifact_count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "Accession Year Trends": """
        SELECT accessionyear, COUNT(*) as count
        FROM artifactmetadata
        WHERE accessionyear IS NOT NULL
        GROUP BY accessionyear
        ORDER BY accessionyear DESC
        LIMIT 20
    """
}

def execute_query(query):
    """Execute analytical query and return results"""
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

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for data collection
    st.sidebar.header("ETL Pipeline")
    num_records = st.sidebar.number_input("Records to fetch", 100, 1000, 500)
    
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            raw_data = fetch_artifacts(num_records)
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(raw_data)
        
        with st.spinner("Loading to database..."):
            load_to_database(metadata_df, media_df, colors_df)
        
        st.success(f"Successfully loaded {len(metadata_df)} artifacts!")
    
    # Analytics section
    st.header("Analytics Queries")
    
    query_name = st.selectbox("Select Query", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Execute Query"):
        query = ANALYTICS_QUERIES[query_name]
        
        with st.spinner("Running query..."):
            results = execute_query(query)
        
        st.dataframe(results)
        
        # Auto-generate visualization
        if len(results.columns) == 2:
            fig = px.bar(
                results,
                x=results.columns[0],
                y=results.columns[1],
                title=query_name
            )
            st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# The app will open at http://localhost:8501
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(num_records, delay=0.5):
    """Fetch data with rate limiting to avoid API throttling"""
    all_records = []
    
    for page in range(1, (num_records // 100) + 2):
        # Fetch page
        records = fetch_page(page)
        all_records.extend(records)
        
        # Rate limit
        time.sleep(delay)
        
        if len(all_records) >= num_records:
            break
    
    return all_records[:num_records]
```

### Incremental Data Loading

```python
def get_last_loaded_id():
    """Get the last artifact ID loaded"""
    connection = mysql.connector.connect(...)
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    max_id = cursor.fetchone()[0]
    connection.close()
    return max_id or 0

def incremental_load():
    """Load only new artifacts"""
    last_id = get_last_loaded_id()
    
    # Fetch artifacts with ID > last_id
    params = {
        'apikey': os.getenv('HARVARD_API_KEY'),
        'size': 100,
        'sort': 'id',
        'sortorder': 'asc',
        'idafter': last_id
    }
    
    response = requests.get(base_url, params=params)
    # Process and load...
```

## Troubleshooting

**API Rate Limits**: If you get 429 errors, increase the delay between requests:
```python
time.sleep(1)  # Wait 1 second between requests
```

**Database Connection Errors**: Verify your `.env` credentials and ensure the database is accessible:
```python
# Test connection
try:
    connection = mysql.connector.connect(...)
    print("Connection successful!")
except Error as e:
    print(f"Error: {e}")
```

**Memory Issues with Large Datasets**: Use chunked loading:
```python
def load_in_chunks(df, chunk_size=1000):
    for start in range(0, len(df), chunk_size):
        chunk = df[start:start + chunk_size]
        # Load chunk to database
```

**Missing Data Fields**: Handle None values in transformations:
```python
metadata = {
    'title': artifact.get('title') or 'Unknown',
    'culture': artifact.get('culture') or 'Not Specified',
    # ...
}
```

This skill enables AI agents to help developers build complete ETL pipelines and analytics applications using museum API data with Python, SQL, and modern visualization tools.
