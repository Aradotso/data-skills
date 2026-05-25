---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for museum artifacts
  - create a data analytics dashboard with Streamlit
  - fetch data from Harvard Art Museums API
  - design SQL schema for artifact collections
  - visualize museum collection data with Plotly
  - set up data engineering pipeline with Python
  - analyze art museum metadata with SQL
  - implement batch data ingestion for artifacts
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering and analytics application for the Harvard Art Museums API. It demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualization using Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App enables you to:

- **Extract** artifact data from the Harvard Art Museums API with pagination and rate limiting
- **Transform** nested JSON responses into normalized relational tables
- **Load** structured data into MySQL/TiDB Cloud databases
- **Analyze** collections using predefined SQL queries
- **Visualize** insights through interactive Plotly dashboards in Streamlit

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

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

### Environment Variables

Create a `.env` file in the project root:

```env
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Getting a Harvard API Key

1. Visit [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Request an API key (free for non-commercial use)
3. Add the key to your `.env` file

## Database Schema

The application uses three main tables:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    dated VARCHAR(200),
    period VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    url VARCHAR(500)
);

-- Media/images table
CREATE TABLE artifactmedia (
    mediaid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    totalimages INT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

-- Color analysis table
CREATE TABLE artifactcolors (
    colorid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## Running the Application

```bash
streamlit run app.py
```

The Streamlit interface will launch at `http://localhost:8501`

## Key Components and Usage

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifact data from Harvard Art Museums API
    
    Args:
        api_key: Harvard API key
        page: Page number for pagination
        size: Number of records per page (max 100)
    
    Returns:
        JSON response with artifact records
    """
    url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    response.raise_for_status()
    return response.json()

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=100)
print(f"Total records: {data['info']['totalrecords']}")
print(f"Pages: {data['info']['pages']}")
```

### 2. ETL Pipeline with Pagination

```python
import pandas as pd
import time

def extract_all_artifacts(api_key, max_pages=10):
    """
    Extract artifacts with pagination and rate limiting
    """
    all_records = []
    
    for page in range(1, max_pages + 1):
        try:
            data = fetch_artifacts(api_key, page=page, size=100)
            records = data.get('records', [])
            all_records.extend(records)
            
            print(f"Fetched page {page}: {len(records)} records")
            
            # Rate limiting - respect API limits
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error on page {page}: {e}")
            break
    
    return all_records

def transform_metadata(records):
    """
    Transform nested JSON into normalized metadata DataFrame
    """
    metadata_list = []
    
    for record in records:
        metadata = {
            'objectid': record.get('objectid'),
            'title': record.get('title'),
            'culture': record.get('culture'),
            'century': record.get('century'),
            'classification': record.get('classification'),
            'department': record.get('department'),
            'division': record.get('division'),
            'dated': record.get('dated'),
            'period': record.get('period'),
            'technique': record.get('technique'),
            'medium': record.get('medium'),
            'dimensions': record.get('dimensions'),
            'url': record.get('url')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_media(records):
    """
    Extract media/image information
    """
    media_list = []
    
    for record in records:
        media = {
            'objectid': record.get('objectid'),
            'baseimageurl': record.get('baseimageurl'),
            'primaryimageurl': record.get('primaryimageurl'),
            'totalimages': record.get('totalpageviews', 0)
        }
        media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_colors(records):
    """
    Extract color analysis data (nested array)
    """
    color_list = []
    
    for record in records:
        objectid = record.get('objectid')
        colors = record.get('colors', [])
        
        for color in colors:
            color_data = {
                'objectid': objectid,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            color_list.append(color_data)
    
    return pd.DataFrame(color_list)
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """
    Create MySQL database connection
    """
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=os.getenv('DB_PORT', 3306),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_metadata(df_metadata):
    """
    Batch insert metadata into database
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (objectid, title, culture, century, classification, department, 
     division, dated, period, technique, medium, dimensions, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    # Convert DataFrame to list of tuples
    data_tuples = [tuple(row) for row in df_metadata.values]
    
    try:
        cursor.executemany(insert_query, data_tuples)
        conn.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
    except Error as e:
        print(f"Error inserting metadata: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()

def load_media(df_media):
    """
    Batch insert media data
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (objectid, baseimageurl, primaryimageurl, totalimages)
    VALUES (%s, %s, %s, %s)
    """
    
    data_tuples = [tuple(row) for row in df_media.values]
    
    try:
        cursor.executemany(insert_query, data_tuples)
        conn.commit()
        print(f"Inserted {cursor.rowcount} media records")
    except Error as e:
        print(f"Error inserting media: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()
```

### 4. Analytical SQL Queries

```python
# Sample analytical queries
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
        ORDER BY count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "Artifacts with Most Images": """
        SELECT m.title, m.culture, a.totalimages
        FROM artifactmetadata m
        JOIN artifactmedia a ON m.objectid = a.objectid
        ORDER BY a.totalimages DESC
        LIMIT 10
    """,
    
    "Media Availability": """
        SELECT 
            CASE 
                WHEN totalimages > 0 THEN 'Has Images'
                ELSE 'No Images'
            END as image_status,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY image_status
    """
}

def execute_query(query_name):
    """
    Execute analytical query and return results as DataFrame
    """
    conn = get_db_connection()
    
    try:
        df = pd.read_sql(ANALYTICAL_QUERIES[query_name], conn)
        return df
    except Error as e:
        print(f"Query error: {e}")
        return pd.DataFrame()
    finally:
        conn.close()
```

### 5. Streamlit Visualization

```python
import streamlit as st
import plotly.express as px

def create_dashboard():
    """
    Main Streamlit dashboard
    """
    st.title("🏛️ Harvard Art Museums Analytics")
    st.markdown("Data Engineering & Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Options")
    query_choice = st.sidebar.selectbox(
        "Select Analysis",
        list(ANALYTICAL_QUERIES.keys())
    )
    
    # Execute query button
    if st.sidebar.button("Run Analysis"):
        with st.spinner("Executing query..."):
            df = execute_query(query_choice)
            
            if not df.empty:
                st.subheader(f"Results: {query_choice}")
                st.dataframe(df)
                
                # Auto-generate visualization
                if len(df.columns) == 2:
                    fig = px.bar(
                        df,
                        x=df.columns[0],
                        y=df.columns[1],
                        title=query_choice
                    )
                    st.plotly_chart(fig, use_container_width=True)
            else:
                st.warning("No results found")

if __name__ == "__main__":
    create_dashboard()
```

## Common Patterns

### Complete ETL Workflow

```python
def run_etl_pipeline(api_key, max_pages=5):
    """
    Complete ETL workflow
    """
    # EXTRACT
    print("Extracting data from API...")
    records = extract_all_artifacts(api_key, max_pages)
    
    # TRANSFORM
    print("Transforming data...")
    df_metadata = transform_metadata(records)
    df_media = transform_media(records)
    df_colors = transform_colors(records)
    
    # LOAD
    print("Loading data to database...")
    load_metadata(df_metadata)
    load_media(df_media)
    load_colors(df_colors)
    
    print("ETL pipeline completed successfully!")

# Execute
api_key = os.getenv('HARVARD_API_KEY')
run_etl_pipeline(api_key, max_pages=10)
```

## Troubleshooting

### API Rate Limiting
```python
# Add exponential backoff for rate limit errors
import time

def fetch_with_retry(api_key, page, max_retries=3):
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:  # Rate limit
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues
```python
# Test database connection
def test_db_connection():
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        print("Database connection successful!")
        cursor.close()
        conn.close()
        return True
    except Error as e:
        print(f"Database connection failed: {e}")
        return False
```

### Handling Missing Data
```python
# Clean data before loading
df_metadata = df_metadata.fillna({
    'title': 'Untitled',
    'culture': 'Unknown',
    'century': 'Unknown'
})

# Or drop rows with critical missing values
df_metadata = df_metadata.dropna(subset=['objectid'])
```

This skill enables AI agents to help developers build complete ETL pipelines for museum artifact data with professional-grade data engineering practices.
