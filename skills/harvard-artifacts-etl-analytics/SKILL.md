---
name: harvard-artifacts-etl-analytics
description: ETL pipeline and analytics for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - query Harvard artifacts collection with SQL analytics
  - create a data engineering pipeline for museum artifacts
  - visualize Harvard museum data with Streamlit
  - extract and transform Harvard Art Museums API data
  - set up artifact collection analytics dashboard
  - implement museum data warehouse with SQL
  - analyze Harvard art collection by culture and century
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Harvard Artifacts Collection Data Engineering Analytics App is an end-to-end data pipeline that:
- Extracts artifact data from the Harvard Art Museums API
- Transforms nested JSON into normalized relational tables
- Loads data into MySQL/TiDB Cloud databases
- Provides 20+ analytical SQL queries for insights
- Visualizes results through an interactive Streamlit dashboard

**Architecture**: API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_database_host"
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="harvard_artifacts"
```

**Required packages**:
- streamlit
- pandas
- requests
- mysql-connector-python (or pymysql)
- plotly
- python-dotenv

## Getting Harvard Art Museums API Key

1. Visit https://harvardartmuseums.org/collections/api
2. Request an API key (free for non-commercial use)
3. Store in environment variable or `.env` file

## Database Schema

The ETL pipeline creates three normalized tables:

```sql
-- Main artifact metadata
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    department VARCHAR(200),
    classification VARCHAR(200),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accessionyear INT
);

-- Media/images associated with artifacts
CREATE TABLE artifactmedia (
    mediaid INT PRIMARY KEY AUTO_INCREMENT,
    objectid INT,
    baseimageurl VARCHAR(500),
    format VARCHAR(50),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

-- Color palette data
CREATE TABLE artifactcolors (
    colorid INT PRIMARY KEY AUTO_INCREMENT,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## ETL Pipeline Code Examples

### Extract: Fetch Data from API

```python
import requests
import os
from typing import Dict, List

def fetch_artifacts(api_key: str, page: int = 1, size: int = 100) -> Dict:
    """Fetch artifacts from Harvard Art Museums API with pagination."""
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only fetch artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    
    return response.json()

def collect_all_artifacts(api_key: str, max_records: int = 1000) -> List[Dict]:
    """Collect multiple pages of artifacts with rate limiting."""
    import time
    
    all_records = []
    page = 1
    size = 100
    
    while len(all_records) < max_records:
        data = fetch_artifacts(api_key, page=page, size=size)
        records = data.get('records', [])
        
        if not records:
            break
            
        all_records.extend(records)
        page += 1
        
        # Rate limiting (API typically allows 2500 requests/day)
        time.sleep(0.5)
    
    return all_records[:max_records]
```

### Transform: Normalize JSON to Relational Format

```python
import pandas as pd
from typing import List, Dict, Tuple

def transform_artifacts(raw_data: List[Dict]) -> Tuple[pd.DataFrame, pd.DataFrame, pd.DataFrame]:
    """Transform nested JSON into normalized dataframes."""
    
    # Extract metadata
    metadata_records = []
    for artifact in raw_data:
        metadata_records.append({
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title', ''),
            'culture': artifact.get('culture', ''),
            'century': artifact.get('century', ''),
            'dated': artifact.get('dated', ''),
            'department': artifact.get('department', ''),
            'classification': artifact.get('classification', ''),
            'medium': artifact.get('medium', ''),
            'dimensions': artifact.get('dimensions', ''),
            'creditline': artifact.get('creditline', ''),
            'accessionyear': artifact.get('accessionyear')
        })
    
    # Extract media data
    media_records = []
    for artifact in raw_data:
        objectid = artifact.get('objectid')
        images = artifact.get('images', [])
        
        for img in images:
            media_records.append({
                'objectid': objectid,
                'baseimageurl': img.get('baseimageurl'),
                'format': img.get('format')
            })
    
    # Extract color data
    color_records = []
    for artifact in raw_data:
        objectid = artifact.get('objectid')
        colors = artifact.get('colors', [])
        
        for color_obj in colors:
            color_records.append({
                'objectid': objectid,
                'color': color_obj.get('color'),
                'spectrum': color_obj.get('spectrum'),
                'hue': color_obj.get('hue')
            })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### Load: Insert into MySQL Database

```python
import mysql.connector
from mysql.connector import Error
import pandas as pd

def get_db_connection():
    """Create database connection from environment variables."""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_metadata(df: pd.DataFrame, connection):
    """Batch insert metadata with duplicate handling."""
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (objectid, title, culture, century, dated, department, 
         classification, medium, dimensions, creditline, accessionyear)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    # Convert DataFrame to list of tuples
    records = df.to_records(index=False).tolist()
    
    cursor.executemany(insert_query, records)
    connection.commit()
    cursor.close()

def load_all_data(metadata_df: pd.DataFrame, 
                  media_df: pd.DataFrame, 
                  colors_df: pd.DataFrame):
    """Load all transformed data into database."""
    connection = get_db_connection()
    
    try:
        load_metadata(metadata_df, connection)
        load_media(media_df, connection)
        load_colors(colors_df, connection)
        print(f"Loaded {len(metadata_df)} artifacts successfully")
    except Error as e:
        print(f"Database error: {e}")
        connection.rollback()
    finally:
        connection.close()
```

## Analytics Queries

### Sample SQL Queries for Insights

```python
# Top 10 cultures with most artifacts
QUERY_CULTURE_DISTRIBUTION = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL AND culture != ''
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 10;
"""

# Artifacts by century
QUERY_CENTURY_ANALYSIS = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY count DESC;
"""

# Most common color palettes
QUERY_COLOR_ANALYSIS = """
    SELECT color, spectrum, COUNT(*) as frequency
    FROM artifactcolors
    GROUP BY color, spectrum
    ORDER BY frequency DESC
    LIMIT 15;
"""

# Departments with image coverage
QUERY_MEDIA_COVERAGE = """
    SELECT 
        m.department,
        COUNT(DISTINCT m.objectid) as total_artifacts,
        COUNT(DISTINCT med.objectid) as with_images,
        ROUND(COUNT(DISTINCT med.objectid) * 100.0 / COUNT(DISTINCT m.objectid), 2) as coverage_pct
    FROM artifactmetadata m
    LEFT JOIN artifactmedia med ON m.objectid = med.objectid
    WHERE m.department IS NOT NULL
    GROUP BY m.department
    ORDER BY total_artifacts DESC;
"""

def execute_query(query: str) -> pd.DataFrame:
    """Execute SQL query and return results as DataFrame."""
    connection = get_db_connection()
    df = pd.read_sql(query, connection)
    connection.close()
    return df
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("Harvard Art Museums Analytics Dashboard")
    st.markdown("### ETL Pipeline & SQL Analytics")
    
    # Sidebar for query selection
    query_options = {
        "Culture Distribution": QUERY_CULTURE_DISTRIBUTION,
        "Century Analysis": QUERY_CENTURY_ANALYSIS,
        "Color Palette": QUERY_COLOR_ANALYSIS,
        "Media Coverage": QUERY_MEDIA_COVERAGE
    }
    
    selected_query = st.sidebar.selectbox(
        "Select Analysis",
        list(query_options.keys())
    )
    
    # Execute query
    if st.button("Run Analysis"):
        with st.spinner("Executing query..."):
            df = execute_query(query_options[selected_query])
            
            # Display results
            st.subheader("Query Results")
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df.columns) >= 2:
                fig = px.bar(
                    df, 
                    x=df.columns[0], 
                    y=df.columns[1],
                    title=f"{selected_query} Visualization"
                )
                st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Run Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Complete ETL Workflow

```python
import os
from dotenv import load_dotenv

def run_etl_pipeline():
    """Execute complete ETL pipeline."""
    # Load configuration
    load_dotenv()
    api_key = os.getenv('HARVARD_API_KEY')
    
    # Extract
    print("Extracting data from API...")
    raw_data = collect_all_artifacts(api_key, max_records=500)
    
    # Transform
    print("Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts(raw_data)
    
    # Load
    print("Loading into database...")
    load_all_data(metadata_df, media_df, colors_df)
    
    print("ETL pipeline completed successfully!")

if __name__ == "__main__":
    run_etl_pipeline()
```

### Incremental Data Loading

```python
def get_latest_objectid(connection) -> int:
    """Get the highest objectid already in database."""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(objectid) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    return result[0] if result[0] else 0

def incremental_etl():
    """Only load new artifacts since last run."""
    connection = get_db_connection()
    latest_id = get_latest_objectid(connection)
    connection.close()
    
    # Fetch only new records
    api_key = os.getenv('HARVARD_API_KEY')
    # Add filtering logic based on latest_id
    # ... continue ETL
```

## Troubleshooting

**API Rate Limiting**
- Harvard API allows ~2500 requests/day
- Implement `time.sleep()` between requests
- Use `hasimage=1` parameter to reduce result set

**Database Connection Issues**
```python
# Test connection
try:
    conn = get_db_connection()
    print("Database connected successfully")
    conn.close()
except Error as e:
    print(f"Connection failed: {e}")
```

**Missing Data Handling**
```python
# Handle null/missing values during transform
df['culture'] = df['culture'].fillna('Unknown')
df['accessionyear'] = pd.to_numeric(df['accessionyear'], errors='coerce')
```

**Memory Issues with Large Datasets**
```python
# Process in chunks
def process_in_chunks(records, chunk_size=100):
    for i in range(0, len(records), chunk_size):
        chunk = records[i:i+chunk_size]
        metadata, media, colors = transform_artifacts(chunk)
        load_all_data(metadata, media, colors)
```

## Configuration Best Practices

Create a `.env` file:
```env
HARVARD_API_KEY=your_api_key_here
DB_HOST=localhost
DB_USER=root
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
DB_PORT=3306
```

Never commit `.env` to version control — add to `.gitignore`.
