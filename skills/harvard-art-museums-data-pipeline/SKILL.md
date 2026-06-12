---
name: harvard-art-museums-data-pipeline
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build a data pipeline for Harvard Art Museums
  - create ETL pipeline with Harvard API
  - analyze Harvard art collection data
  - set up museum artifacts data engineering app
  - implement SQL analytics dashboard for art data
  - extract transform load Harvard museums data
  - visualize museum collection with Streamlit
  - query Harvard Art Museums API
---

# Harvard Art Museums Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is an end-to-end data engineering application that demonstrates production ETL pipelines, SQL analytics, and interactive visualization using the Harvard Art Museums API. It extracts artifact metadata, transforms nested JSON into relational tables, loads data into MySQL/TiDB, and provides a Streamlit dashboard for analytics.

**Architecture**: API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

### Dependencies

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

```bash
# Harvard API Configuration
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Request an API key (free)
3. Add to `.env` file

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
    culture VARCHAR(255),
    classification VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    dated VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    medium VARCHAR(500),
    technique VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accessionyear INT,
    url TEXT,
    totalpageviews INT,
    totaluniquepageviews INT
)
""")

cursor.execute("""
CREATE TABLE IF NOT EXISTS artifactmedia (
    artifact_id INT,
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    primaryimageurl VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
)
""")

cursor.execute("""
CREATE TABLE IF NOT EXISTS artifactcolors (
    artifact_id INT,
    color VARCHAR(100),
    spectrum VARCHAR(100),
    hue VARCHAR(100),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
)
""")

conn.commit()
cursor.close()
conn.close()
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
    url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size
    }
    
    response = requests.get(url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

def collect_multiple_pages(num_pages=5, page_size=100):
    """Collect artifacts across multiple pages"""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(page=page, size=page_size)
        all_artifacts.extend(data.get('records', []))
    
    return all_artifacts
```

### 2. ETL Transformation

```python
import pandas as pd

def transform_artifacts(raw_artifacts):
    """Transform nested JSON into relational dataframes"""
    
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'classification': artifact.get('classification'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'medium': artifact.get('medium'),
            'technique': artifact.get('technique'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'accessionyear': artifact.get('accessionyear'),
            'url': artifact.get('url'),
            'totalpageviews': artifact.get('totalpageviews'),
            'totaluniquepageviews': artifact.get('totaluniquepageviews')
        }
        metadata_records.append(metadata)
        
        # Extract media information
        if artifact.get('primaryimageurl'):
            media = {
                'artifact_id': artifact.get('id'),
                'baseimageurl': artifact.get('baseimageurl'),
                'iiifbaseuri': artifact.get('iiifbaseuri'),
                'primaryimageurl': artifact.get('primaryimageurl')
            }
            media_records.append(media)
        
        # Extract color data
        colors = artifact.get('colors', [])
        for color in colors:
            color_record = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            color_records.append(color_record)
    
    return {
        'metadata': pd.DataFrame(metadata_records),
        'media': pd.DataFrame(media_records),
        'colors': pd.DataFrame(color_records)
    }
```

### 3. SQL Loading

```python
import mysql.connector
from dotenv import load_dotenv
import os

def load_to_database(dataframes):
    """Batch insert dataframes into SQL tables"""
    load_dotenv()
    
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    cursor = conn.cursor()
    
    # Load metadata
    metadata_df = dataframes['metadata']
    for _, row in metadata_df.iterrows():
        sql = """
        INSERT INTO artifactmetadata 
        (id, title, culture, classification, period, century, dated, 
         department, division, medium, technique, dimensions, creditline, 
         accessionyear, url, totalpageviews, totaluniquepageviews)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.execute(sql, tuple(row))
    
    # Load media
    media_df = dataframes['media']
    for _, row in media_df.iterrows():
        sql = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, iiifbaseuri, primaryimageurl)
        VALUES (%s, %s, %s, %s)
        """
        cursor.execute(sql, tuple(row))
    
    # Load colors
    colors_df = dataframes['colors']
    for _, row in colors_df.iterrows():
        sql = """
        INSERT INTO artifactcolors 
        (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
        """
        cursor.execute(sql, tuple(row))
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print(f"Loaded {len(metadata_df)} artifacts to database")
```

### 4. SQL Analytics Queries

```python
def get_analytics_queries():
    """Predefined analytical SQL queries"""
    return {
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
        
        "Most Viewed Artifacts": """
            SELECT title, culture, totalpageviews
            FROM artifactmetadata
            WHERE totalpageviews IS NOT NULL
            ORDER BY totalpageviews DESC
            LIMIT 20
        """,
        
        "Media Availability": """
            SELECT 
                CASE WHEN m.artifact_id IS NOT NULL THEN 'Has Image' ELSE 'No Image' END as media_status,
                COUNT(*) as count
            FROM artifactmetadata a
            LEFT JOIN artifactmedia m ON a.id = m.artifact_id
            GROUP BY media_status
        """,
        
        "Color Distribution": """
            SELECT color, COUNT(*) as count, AVG(percent) as avg_percent
            FROM artifactcolors
            GROUP BY color
            ORDER BY count DESC
            LIMIT 15
        """,
        
        "Department Analysis": """
            SELECT department, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY artifact_count DESC
        """,
        
        "Accession Year Trends": """
            SELECT accessionyear, COUNT(*) as count
            FROM artifactmetadata
            WHERE accessionyear IS NOT NULL AND accessionyear > 1800
            GROUP BY accessionyear
            ORDER BY accessionyear
        """
    }

def execute_query(query_name):
    """Execute a specific analytics query"""
    load_dotenv()
    
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    queries = get_analytics_queries()
    query = queries.get(query_name)
    
    if not query:
        raise ValueError(f"Query '{query_name}' not found")
    
    df = pd.read_sql(query, conn)
    conn.close()
    
    return df
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px
import pandas as pd

def main():
    st.set_page_config(page_title="Harvard Art Analytics", layout="wide")
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Choose a page",
        ["Data Collection", "Analytics Dashboard", "Custom Query"]
    )
    
    if page == "Data Collection":
        st.header("📥 Collect Artifact Data")
        
        num_pages = st.number_input("Number of pages to fetch", 
                                     min_value=1, max_value=50, value=5)
        
        if st.button("Start Data Collection"):
            with st.spinner("Fetching data from API..."):
                artifacts = collect_multiple_pages(num_pages=num_pages)
                st.success(f"Collected {len(artifacts)} artifacts")
                
                st.write("Transforming data...")
                dataframes = transform_artifacts(artifacts)
                
                st.write("Loading to database...")
                load_to_database(dataframes)
                st.success("ETL pipeline completed!")
    
    elif page == "Analytics Dashboard":
        st.header("📊 SQL Analytics")
        
        queries = get_analytics_queries()
        query_name = st.selectbox("Select Analysis", list(queries.keys()))
        
        if st.button("Run Query"):
            with st.spinner("Executing query..."):
                df = execute_query(query_name)
                
                st.subheader("Results")
                st.dataframe(df)
                
                # Auto-generate visualization
                if len(df.columns) == 2:
                    fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                                title=query_name)
                    st.plotly_chart(fig, use_container_width=True)
    
    elif page == "Custom Query":
        st.header("🔍 Custom SQL Query")
        
        custom_query = st.text_area("Enter your SQL query", height=200)
        
        if st.button("Execute"):
            try:
                load_dotenv()
                conn = mysql.connector.connect(
                    host=os.getenv('DB_HOST'),
                    port=int(os.getenv('DB_PORT', 3306)),
                    user=os.getenv('DB_USER'),
                    password=os.getenv('DB_PASSWORD'),
                    database=os.getenv('DB_NAME')
                )
                df = pd.read_sql(custom_query, conn)
                conn.close()
                st.dataframe(df)
            except Exception as e:
                st.error(f"Query failed: {str(e)}")

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Complete ETL Pipeline

```python
from dotenv import load_dotenv

def run_full_pipeline(num_pages=5):
    """Execute complete ETL pipeline"""
    load_dotenv()
    
    # Extract
    print("Extracting data from API...")
    artifacts = collect_multiple_pages(num_pages=num_pages)
    
    # Transform
    print("Transforming data...")
    dataframes = transform_artifacts(artifacts)
    
    # Load
    print("Loading to database...")
    load_to_database(dataframes)
    
    print(f"Pipeline complete! Processed {len(artifacts)} artifacts")
    
    return dataframes

# Run pipeline
if __name__ == "__main__":
    run_full_pipeline(num_pages=10)
```

### Incremental Data Loading

```python
def get_latest_artifact_id():
    """Get the highest artifact ID in database"""
    load_dotenv()
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    conn.close()
    return result[0] if result[0] else 0

def incremental_load():
    """Load only new artifacts"""
    latest_id = get_latest_artifact_id()
    print(f"Latest artifact ID: {latest_id}")
    
    # Fetch new artifacts (implement API filtering)
    # Transform and load only new records
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(page, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page=page)
        except Exception as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt
                print(f"Retry in {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise e
```

### Database Connection Issues

```python
def test_db_connection():
    """Test database connectivity"""
    try:
        load_dotenv()
        conn = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME'),
            connect_timeout=10
        )
        print("✅ Database connection successful")
        conn.close()
        return True
    except Exception as e:
        print(f"❌ Connection failed: {str(e)}")
        return False
```

### Missing API Key

```python
def validate_config():
    """Validate required environment variables"""
    load_dotenv()
    
    required = ['HARVARD_API_KEY', 'DB_HOST', 'DB_USER', 'DB_PASSWORD', 'DB_NAME']
    missing = [var for var in required if not os.getenv(var)]
    
    if missing:
        raise ValueError(f"Missing required environment variables: {', '.join(missing)}")
    
    print("✅ Configuration validated")
```

## Best Practices

- **Batch Processing**: Load data in batches of 100-500 records for optimal performance
- **Error Handling**: Wrap API calls in try-except blocks with retry logic
- **Data Validation**: Check for null values and data types before database insertion
- **Indexing**: Add indexes on frequently queried columns (culture, century, department)
- **Logging**: Implement logging for production ETL pipelines

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('etl_pipeline.log'),
        logging.StreamHandler()
    ]
)

logger = logging.getLogger(__name__)

def safe_etl_run():
    try:
        logger.info("Starting ETL pipeline")
        artifacts = collect_multiple_pages(num_pages=5)
        logger.info(f"Extracted {len(artifacts)} artifacts")
        # ... rest of pipeline
    except Exception as e:
        logger.error(f"Pipeline failed: {str(e)}", exc_info=True)
        raise
```
