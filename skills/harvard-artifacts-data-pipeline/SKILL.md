---
name: harvard-artifacts-data-pipeline
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums data using Python, SQL, and Streamlit
triggers:
  - how do I fetch data from Harvard Art Museums API
  - build an ETL pipeline for museum artifacts
  - create analytics dashboard with Streamlit and museum data
  - query Harvard artifacts collection with SQL
  - visualize museum data with Plotly
  - set up data engineering project for art collections
  - extract and transform Harvard API artifact data
  - build SQL database for museum artifacts
---

# Harvard Artifacts Data Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project demonstrates an end-to-end data engineering pipeline that extracts artifact data from the Harvard Art Museums API, transforms it into structured relational data, loads it into SQL databases, and provides interactive analytics visualizations through Streamlit. It's designed to showcase real-world ETL processes, SQL analytics, and data visualization patterns.

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages include:
# - streamlit
# - pandas
# - requests
# - mysql-connector-python (or pymysql)
# - plotly
# - python-dotenv
```

## Configuration

### API Key Setup

Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api):

```python
# Store in .env file
HARVARD_API_KEY=your_api_key_here
DATABASE_HOST=your_db_host
DATABASE_USER=your_db_user
DATABASE_PASSWORD=your_db_password
DATABASE_NAME=your_db_name
```

### Database Setup

Create the required tables:

```sql
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    accessionyear INT,
    rank INT,
    verificationlevel INT,
    totalpageviews INT,
    totaluniquepageviews INT
);

CREATE TABLE artifactmedia (
    mediaid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    iiifbaseuri VARCHAR(500),
    publiccaption TEXT,
    baseimageurl VARCHAR(500),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

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

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        "apikey": api_key,
        "page": page,
        "size": size,
        "hasimage": 1  # Only artifacts with images
    }
    
    try:
        response = requests.get(base_url, params=params)
        response.raise_for_status()
        data = response.json()
        
        return {
            "records": data.get("records", []),
            "info": data.get("info", {}),
            "total_pages": data["info"]["pages"]
        }
    except requests.exceptions.RequestException as e:
        print(f"Error fetching data: {e}")
        return None

def fetch_all_artifacts(api_key, max_pages=10):
    """
    Fetch multiple pages of artifacts with rate limiting
    """
    import time
    all_records = []
    
    for page in range(1, max_pages + 1):
        print(f"Fetching page {page}...")
        result = fetch_artifacts(api_key, page=page)
        
        if result:
            all_records.extend(result["records"])
            time.sleep(0.5)  # Rate limiting
        else:
            break
    
    return all_records
```

### 2. ETL Pipeline

```python
import pandas as pd

def extract_metadata(records):
    """
    Extract artifact metadata from API response
    """
    metadata = []
    
    for record in records:
        metadata.append({
            "objectid": record.get("objectid"),
            "title": record.get("title"),
            "culture": record.get("culture"),
            "century": record.get("century"),
            "classification": record.get("classification"),
            "department": record.get("department"),
            "dated": record.get("dated"),
            "accessionyear": record.get("accessionyear"),
            "rank": record.get("rank"),
            "verificationlevel": record.get("verificationlevel"),
            "totalpageviews": record.get("totalpageviews"),
            "totaluniquepageviews": record.get("totaluniquepageviews")
        })
    
    return pd.DataFrame(metadata)

def extract_media(records):
    """
    Extract media information (images) from artifacts
    """
    media = []
    
    for record in records:
        objectid = record.get("objectid")
        images = record.get("images", [])
        
        for img in images:
            media.append({
                "objectid": objectid,
                "iiifbaseuri": img.get("iiifbaseuri"),
                "publiccaption": img.get("publiccaption"),
                "baseimageurl": img.get("baseimageurl")
            })
    
    return pd.DataFrame(media)

def extract_colors(records):
    """
    Extract color data from artifacts
    """
    colors = []
    
    for record in records:
        objectid = record.get("objectid")
        color_data = record.get("colors", [])
        
        for color in color_data:
            colors.append({
                "objectid": objectid,
                "color": color.get("color"),
                "spectrum": color.get("spectrum"),
                "hue": color.get("hue"),
                "percent": color.get("percent")
            })
    
    return pd.DataFrame(colors)
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """
    Create database connection
    """
    try:
        connection = mysql.connector.connect(
            host=os.getenv("DATABASE_HOST"),
            user=os.getenv("DATABASE_USER"),
            password=os.getenv("DATABASE_PASSWORD"),
            database=os.getenv("DATABASE_NAME")
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def batch_insert_metadata(df, connection):
    """
    Batch insert artifact metadata
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (objectid, title, culture, century, classification, department, 
     dated, accessionyear, rank, verificationlevel, 
     totalpageviews, totaluniquepageviews)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    data = [tuple(row) for row in df.values]
    
    try:
        cursor.executemany(insert_query, data)
        connection.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
    except Error as e:
        print(f"Error inserting metadata: {e}")
        connection.rollback()
    finally:
        cursor.close()

def batch_insert_media(df, connection):
    """
    Batch insert media data
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (objectid, iiifbaseuri, publiccaption, baseimageurl)
    VALUES (%s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    
    try:
        cursor.executemany(insert_query, data)
        connection.commit()
        print(f"Inserted {cursor.rowcount} media records")
    except Error as e:
        print(f"Error inserting media: {e}")
        connection.rollback()
    finally:
        cursor.close()
```

### 4. SQL Analytics Queries

```python
# Sample analytical queries

ANALYTICS_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 20
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "top_viewed_artifacts": """
        SELECT title, culture, century, totalpageviews
        FROM artifactmetadata
        ORDER BY totalpageviews DESC
        LIMIT 20
    """,
    
    "artifacts_with_images": """
        SELECT am.department, COUNT(DISTINCT am.objectid) as artifacts_with_images
        FROM artifactmetadata am
        JOIN artifactmedia amd ON am.objectid = amd.objectid
        GROUP BY am.department
        ORDER BY artifacts_with_images DESC
    """,
    
    "color_distribution": """
        SELECT color, COUNT(*) as count, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "artifacts_by_classification": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 20
    """
}

def execute_query(query, connection):
    """
    Execute SQL query and return DataFrame
    """
    try:
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        print(f"Query execution error: {e}")
        return None
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def create_dashboard():
    """
    Create interactive Streamlit dashboard
    """
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🏛️",
        layout="wide"
    )
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Queries")
    query_name = st.sidebar.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    # Database connection
    connection = get_db_connection()
    
    if connection:
        # Execute selected query
        if st.sidebar.button("Run Query"):
            with st.spinner("Executing query..."):
                df = execute_query(ANALYTICS_QUERIES[query_name], connection)
                
                if df is not None and not df.empty:
                    # Display results
                    st.subheader(f"Results: {query_name.replace('_', ' ').title()}")
                    st.dataframe(df)
                    
                    # Visualization
                    if len(df.columns) >= 2:
                        fig = px.bar(
                            df.head(20),
                            x=df.columns[0],
                            y=df.columns[1],
                            title=f"{query_name.replace('_', ' ').title()}"
                        )
                        st.plotly_chart(fig, use_container_width=True)
                else:
                    st.warning("No results found")
        
        connection.close()
    else:
        st.error("Database connection failed")

if __name__ == "__main__":
    create_dashboard()
```

## Common Usage Patterns

### Complete ETL Workflow

```python
import os
from dotenv import load_dotenv

load_dotenv()

def run_etl_pipeline():
    """
    Execute complete ETL pipeline
    """
    # 1. Extract
    api_key = os.getenv("HARVARD_API_KEY")
    print("Extracting data from API...")
    records = fetch_all_artifacts(api_key, max_pages=5)
    
    # 2. Transform
    print("Transforming data...")
    metadata_df = extract_metadata(records)
    media_df = extract_media(records)
    colors_df = extract_colors(records)
    
    # 3. Load
    print("Loading data to database...")
    connection = get_db_connection()
    
    if connection:
        batch_insert_metadata(metadata_df, connection)
        batch_insert_media(media_df, connection)
        batch_insert_colors(colors_df, connection)
        connection.close()
        print("ETL pipeline completed successfully!")
    else:
        print("ETL pipeline failed - database connection error")

# Run the pipeline
run_etl_pipeline()
```

## Troubleshooting

**API Rate Limiting**: Add delays between requests
```python
import time
time.sleep(0.5)  # Wait 500ms between requests
```

**Database Connection Issues**: Verify credentials and network access
```python
# Test connection
connection = get_db_connection()
if connection and connection.is_connected():
    print("Connection successful")
else:
    print("Check DATABASE_* environment variables")
```

**Empty Query Results**: Check data availability
```python
# Verify data exists
cursor = connection.cursor()
cursor.execute("SELECT COUNT(*) FROM artifactmetadata")
count = cursor.fetchone()[0]
print(f"Total records: {count}")
```

**Streamlit Port Conflicts**: Specify custom port
```bash
streamlit run app.py --server.port 8502
```
