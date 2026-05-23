---
name: harvard-art-museums-data-pipeline
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Streamlit, MySQL, and Plotly
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - create a data engineering project with museum artifacts
  - set up Streamlit analytics dashboard for Harvard API
  - extract and transform Harvard Art Museums data
  - build SQL analytics for museum artifact collections
  - visualize Harvard museum data with Plotly
  - implement artifact metadata ETL pipeline
  - create museum data engineering application
---

# Harvard Art Museums Data Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines with API integration, SQL database design, analytical queries, and interactive Streamlit visualizations.

## What It Does

The Harvard Art Museums Data Pipeline provides:
- **API Integration**: Fetch artifact data from Harvard Art Museums with pagination and rate limiting
- **ETL Processing**: Extract nested JSON, transform to relational format, load into SQL databases
- **Database Design**: Structured tables for artifact metadata, media, and color information
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Interactive Dashboard**: Streamlit-based UI with Plotly visualizations

## Installation

### Prerequisites

```bash
# Python 3.8+
python --version

# Install dependencies
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Database Setup

The application supports MySQL or TiDB Cloud. Create the following schema:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    technique VARCHAR(500),
    dated VARCHAR(255),
    url VARCHAR(500),
    verificationlevel INT,
    accession_number VARCHAR(255),
    division VARCHAR(255)
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    imagepermissionlevel INT,
    totalimages INT,
    totalpageimagepreviews INT,
    totaluniquepageimagepreviews INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
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

### Configuration

Create a `.env` file in the project root:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

Get your API key from: https://docs.google.com/forms/d/e/1FAIpQLSfkmEBqH76HLMMiCC-GPPnhcvHC9aJS86E32dOd0Z8MpY2rvQ/viewform

## Project Structure

```
Harvard-Artifacts-Collection-Data-Engineering-Analytics-App/
├── app.py                  # Main Streamlit application
├── etl_pipeline.py         # ETL logic and data processing
├── database.py             # Database connection and operations
├── queries.py              # SQL analytics queries
├── .env                    # Environment variables
├── requirements.txt        # Python dependencies
└── README.md
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_records=100):
    """Fetch artifact data from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    artifacts = []
    page = 1
    size = 100  # Max per page
    
    while len(artifacts) < num_records:
        params = {
            'apikey': api_key,
            'size': min(size, num_records - len(artifacts)),
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            artifacts.extend(data.get('records', []))
            
            if not data.get('records'):
                break
                
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_records]
```

### 2. ETL Transform

```python
import pandas as pd

def transform_artifacts(artifacts_json):
    """Transform nested JSON to relational format"""
    
    # Extract metadata
    metadata = []
    media = []
    colors = []
    
    for artifact in artifacts_json:
        # Metadata table
        metadata.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', ''),
            'period': artifact.get('period', ''),
            'century': artifact.get('century', ''),
            'classification': artifact.get('classification', ''),
            'department': artifact.get('department', ''),
            'technique': artifact.get('technique', '')[:500],
            'dated': artifact.get('dated', ''),
            'url': artifact.get('url', ''),
            'verificationlevel': artifact.get('verificationlevel', 0),
            'accession_number': artifact.get('accessionyear', ''),
            'division': artifact.get('division', '')
        })
        
        # Media table
        media.append({
            'artifact_id': artifact.get('id'),
            'baseimageurl': artifact.get('baseimageurl', ''),
            'primaryimageurl': artifact.get('primaryimageurl', ''),
            'imagepermissionlevel': artifact.get('imagepermissionlevel', 0),
            'totalimages': artifact.get('totalimages', 0),
            'totalpageimagepreviews': artifact.get('totalpageimagepreviews', 0),
            'totaluniquepageimagepreviews': artifact.get('totaluniquepageimagepreviews', 0)
        })
        
        # Colors table (nested data)
        if artifact.get('colors'):
            for color_obj in artifact.get('colors', []):
                colors.append({
                    'artifact_id': artifact.get('id'),
                    'color': color_obj.get('color', ''),
                    'spectrum': color_obj.get('spectrum', ''),
                    'hue': color_obj.get('hue', ''),
                    'saturation': color_obj.get('saturation', 0.0),
                    'percent': color_obj.get('percent', 0.0)
                })
    
    return (
        pd.DataFrame(metadata),
        pd.DataFrame(media),
        pd.DataFrame(colors)
    )
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            database=os.getenv('DB_NAME'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD')
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_to_database(metadata_df, media_df, colors_df):
    """Batch insert data into SQL tables"""
    connection = get_db_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    try:
        # Load metadata
        metadata_query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, period, century, classification, 
             department, technique, dated, url, verificationlevel, 
             accession_number, division)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_query, metadata_df.values.tolist())
        
        # Load media
        media_query = """
            INSERT INTO artifactmedia 
            (artifact_id, baseimageurl, primaryimageurl, 
             imagepermissionlevel, totalimages, totalpageimagepreviews, 
             totaluniquepageimagepreviews)
            VALUES (%s, %s, %s, %s, %s, %s, %s)
        """
        cursor.executemany(media_query, media_df.values.tolist())
        
        # Load colors
        if not colors_df.empty:
            colors_query = """
                INSERT INTO artifactcolors 
                (artifact_id, color, spectrum, hue, saturation, percent)
                VALUES (%s, %s, %s, %s, %s, %s)
            """
            cursor.executemany(colors_query, colors_df.values.tolist())
        
        connection.commit()
        return True
        
    except Error as e:
        print(f"Database load error: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()
```

### 4. SQL Analytics Queries

```python
# queries.py
ANALYTICS_QUERIES = {
    "Artifact Count by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "Top Departments by Artifacts": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY total_artifacts DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "Color Distribution Across Artifacts": """
        SELECT color, COUNT(*) as frequency
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 10
    """,
    
    "Average Images per Classification": """
        SELECT m.classification, AVG(a.totalimages) as avg_images
        FROM artifactmetadata m
        JOIN artifactmedia a ON m.id = a.artifact_id
        WHERE m.classification IS NOT NULL
        GROUP BY m.classification
        ORDER BY avg_images DESC
        LIMIT 10
    """,
    
    "Artifacts with Most Images": """
        SELECT m.title, m.culture, a.totalimages, a.primaryimageurl
        FROM artifactmetadata m
        JOIN artifactmedia a ON m.id = a.artifact_id
        ORDER BY a.totalimages DESC
        LIMIT 10
    """,
    
    "Dominant Color Analysis": """
        SELECT c.color, c.spectrum, 
               AVG(c.percent) as avg_percent,
               COUNT(DISTINCT c.artifact_id) as artifact_count
        FROM artifactcolors c
        GROUP BY c.color, c.spectrum
        HAVING artifact_count > 5
        ORDER BY avg_percent DESC
        LIMIT 15
    """
}

def execute_query(query_name):
    """Execute analytics query and return results"""
    connection = get_db_connection()
    if not connection:
        return None
    
    try:
        query = ANALYTICS_QUERIES[query_name]
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        print(f"Query execution error: {e}")
        return None
    finally:
        connection.close()
```

### 5. Streamlit Dashboard

```python
# app.py
import streamlit as st
import plotly.express as px

st.set_page_config(
    page_title="Harvard Art Museums Analytics",
    page_icon="🎨",
    layout="wide"
)

st.title("🎨 Harvard Art Museums Data Analytics")
st.markdown("*ETL Pipeline & SQL Analytics Dashboard*")

# Sidebar for ETL
with st.sidebar:
    st.header("⚙️ ETL Pipeline")
    
    num_records = st.number_input(
        "Number of records to fetch",
        min_value=10,
        max_value=1000,
        value=100,
        step=10
    )
    
    if st.button("🔄 Run ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            artifacts = fetch_artifacts(num_records)
            st.success(f"✓ Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            st.success("✓ Data transformed")
        
        with st.spinner("Loading to database..."):
            success = load_to_database(metadata_df, media_df, colors_df)
            if success:
                st.success("✓ Data loaded to SQL")
            else:
                st.error("✗ Database load failed")

# Main analytics section
st.header("📊 SQL Analytics")

query_name = st.selectbox(
    "Select Analytics Query",
    list(ANALYTICS_QUERIES.keys())
)

if st.button("Execute Query"):
    with st.spinner("Running query..."):
        results = execute_query(query_name)
        
        if results is not None and not results.empty:
            st.subheader("Query Results")
            st.dataframe(results, use_container_width=True)
            
            # Auto-generate visualization
            if len(results.columns) >= 2:
                fig = px.bar(
                    results,
                    x=results.columns[0],
                    y=results.columns[1],
                    title=query_name,
                    labels={results.columns[0]: results.columns[0].title(),
                           results.columns[1]: results.columns[1].title()}
                )
                st.plotly_chart(fig, use_container_width=True)
        else:
            st.warning("No results found")
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# The app will open at http://localhost:8501
```

## Common Patterns

### Pattern 1: Incremental ETL

```python
def fetch_new_artifacts_only(last_id):
    """Fetch only artifacts newer than last_id"""
    params = {
        'apikey': os.getenv('HARVARD_API_KEY'),
        'size': 100,
        'sort': 'id',
        'sortorder': 'asc',
        'id': f'>{last_id}'
    }
    response = requests.get(base_url, params=params)
    return response.json().get('records', [])
```

### Pattern 2: Error Handling in ETL

```python
def safe_etl_pipeline(num_records):
    """ETL with comprehensive error handling"""
    try:
        artifacts = fetch_artifacts(num_records)
        if not artifacts:
            return {"status": "error", "message": "No data fetched"}
        
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        
        success = load_to_database(metadata_df, media_df, colors_df)
        
        return {
            "status": "success" if success else "error",
            "records_processed": len(artifacts),
            "metadata_count": len(metadata_df),
            "media_count": len(media_df),
            "colors_count": len(colors_df)
        }
    except Exception as e:
        return {"status": "error", "message": str(e)}
```

### Pattern 3: Custom Analytics

```python
def create_custom_query(filters):
    """Build dynamic SQL based on user filters"""
    base_query = "SELECT * FROM artifactmetadata WHERE 1=1"
    
    if filters.get('culture'):
        base_query += f" AND culture = '{filters['culture']}'"
    
    if filters.get('century'):
        base_query += f" AND century = '{filters['century']}'"
    
    if filters.get('department'):
        base_query += f" AND department = '{filters['department']}'"
    
    return base_query
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_rate_limit(num_records, delay=0.5):
    """Add delay between API calls"""
    artifacts = []
    for page in range(1, (num_records // 100) + 2):
        # Fetch page
        time.sleep(delay)  # Rate limit
        # ... rest of logic
    return artifacts
```

### Database Connection Issues

```python
def verify_db_connection():
    """Test database connectivity"""
    try:
        connection = get_db_connection()
        if connection and connection.is_connected():
            cursor = connection.cursor()
            cursor.execute("SELECT 1")
            cursor.close()
            connection.close()
            return True
    except Error as e:
        print(f"Connection test failed: {e}")
        return False
```

### Memory Management for Large Datasets

```python
def batch_load(dataframe, batch_size=1000):
    """Load large dataframes in batches"""
    total_rows = len(dataframe)
    for start in range(0, total_rows, batch_size):
        end = min(start + batch_size, total_rows)
        batch = dataframe.iloc[start:end]
        load_to_database(batch, batch, pd.DataFrame())
```

This skill provides everything needed to build production-ready data engineering pipelines with the Harvard Art Museums API, focusing on practical ETL patterns, SQL analytics, and interactive visualization.
