---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums data using Python, SQL, and Streamlit
triggers:
  - how do I fetch data from Harvard Art Museums API
  - build an ETL pipeline for museum artifacts
  - create analytics dashboard with Streamlit and SQL
  - visualize Harvard museum collection data
  - set up TiDB cloud database for art data
  - query Harvard artifacts by culture and century
  - implement pagination for museum API requests
  - transform nested JSON museum data to SQL tables
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. It demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualizations with Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color data into SQL databases
- **SQL Analytics**: Execute predefined analytical queries on museum collection data
- **Interactive Dashboards**: Visualize insights using Streamlit and Plotly
- **Database Design**: Relational schema with proper foreign key relationships

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

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
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_PORT=4000
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Register for a free API key
3. Add to your `.env` file

### Database Setup

The application supports MySQL or TiDB Cloud:

```python
import mysql.connector
from dotenv import load_dotenv
import os

load_dotenv()

def get_db_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
```

## Database Schema

### Create Tables

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    objectnumber VARCHAR(100),
    accessionyear INT,
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500)
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(1000),
    primaryimageurl VARCHAR(1000),
    imagespresent BOOLEAN,
    totalimagecount INT,
    totalpageviews INT,
    totaluniquepageviews INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import time

def fetch_artifacts(api_key, size=100, max_records=1000):
    """Fetch artifacts with pagination"""
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    page = 1
    
    while len(all_artifacts) < max_records:
        params = {
            'apikey': api_key,
            'size': size,
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
        
        all_artifacts.extend(records)
        print(f"Fetched page {page}: {len(records)} artifacts")
        
        page += 1
        time.sleep(0.5)  # Rate limiting
    
    return all_artifacts[:max_records]
```

### Transform: Process JSON to DataFrames

```python
import pandas as pd

def transform_artifacts(artifacts):
    """Transform nested JSON to relational dataframes"""
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Metadata
        metadata_list.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'objectnumber': artifact.get('objectnumber'),
            'accessionyear': artifact.get('accessionyear'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions')
        })
        
        # Media
        media_list.append({
            'artifact_id': artifact.get('id'),
            'baseimageurl': artifact.get('baseimageurl'),
            'primaryimageurl': artifact.get('primaryimageurl'),
            'imagespresent': artifact.get('images') is not None and len(artifact.get('images', [])) > 0,
            'totalimagecount': artifact.get('totalpageviews', 0),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        })
        
        # Colors
        for color in artifact.get('colors', []):
            colors_list.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load: Insert into Database

```python
def load_to_database(metadata_df, media_df, colors_df):
    """Batch insert dataframes to SQL database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    # Insert metadata
    for _, row in metadata_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmetadata 
            (id, title, culture, period, century, classification, department, 
             dated, objectnumber, accessionyear, technique, medium, dimensions)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """, tuple(row))
    
    # Insert media
    for _, row in media_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmedia 
            (artifact_id, baseimageurl, primaryimageurl, imagespresent, 
             totalimagecount, totalpageviews, totaluniquepageviews)
            VALUES (%s, %s, %s, %s, %s, %s, %s)
        """, tuple(row))
    
    # Insert colors
    for _, row in colors_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """, tuple(row))
    
    conn.commit()
    cursor.close()
    conn.close()
    print("Data loaded successfully!")
```

## Analytical SQL Queries

### Common Analytics Patterns

```python
# Top 10 cultures by artifact count
query1 = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10;
"""

# Artifacts by century
query2 = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC;
"""

# Media availability analysis
query3 = """
SELECT 
    imagespresent,
    COUNT(*) as count,
    ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM artifactmedia), 2) as percentage
FROM artifactmedia
GROUP BY imagespresent;
"""

# Color distribution
query4 = """
SELECT color, COUNT(*) as frequency
FROM artifactcolors
WHERE color IS NOT NULL
GROUP BY color
ORDER BY frequency DESC
LIMIT 15;
"""

# Department analysis
query5 = """
SELECT department, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY artifact_count DESC;
"""
```

## Streamlit Dashboard

### Main Application

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar navigation
    page = st.sidebar.selectbox("Select Page", [
        "Data Collection",
        "ETL Pipeline",
        "SQL Analytics",
        "Visualizations"
    ])
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "ETL Pipeline":
        show_etl_pipeline()
    elif page == "SQL Analytics":
        show_analytics()
    elif page == "Visualizations":
        show_visualizations()

def show_analytics():
    st.header("SQL Analytics")
    
    queries = {
        "Top Cultures": """SELECT culture, COUNT(*) as count 
                          FROM artifactmetadata 
                          WHERE culture IS NOT NULL 
                          GROUP BY culture 
                          ORDER BY count DESC LIMIT 10""",
        "Century Distribution": """SELECT century, COUNT(*) as count 
                                   FROM artifactmetadata 
                                   WHERE century IS NOT NULL 
                                   GROUP BY century""",
        "Color Analysis": """SELECT color, COUNT(*) as frequency 
                            FROM artifactcolors 
                            GROUP BY color 
                            ORDER BY frequency DESC LIMIT 15"""
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        conn = get_db_connection()
        df = pd.read_sql(queries[selected_query], conn)
        conn.close()
        
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                        title=selected_query)
            st.plotly_chart(fig)

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

### Error Handling for API Requests

```python
def safe_api_request(url, params, max_retries=3):
    """API request with retry logic"""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)
    return None
```

### Incremental Data Loading

```python
def get_last_artifact_id():
    """Get the last loaded artifact ID"""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()[0]
    cursor.close()
    conn.close()
    return result or 0
```

## Troubleshooting

**API Rate Limiting**: Add `time.sleep(0.5)` between requests or use exponential backoff.

**Database Connection Errors**: Verify `.env` credentials and ensure database is accessible. For TiDB Cloud, check SSL requirements.

**Memory Issues with Large Datasets**: Process data in batches using `chunksize` parameter in pandas.

**Missing Color Data**: Some artifacts don't have color information. Filter with `WHERE color IS NOT NULL`.

**Streamlit Caching**: Use `@st.cache_data` for expensive operations:

```python
@st.cache_data
def load_analytics_data(query):
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```
