---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - extract art museum data from Harvard API
  - set up data engineering pipeline for art collections
  - analyze Harvard museum artifacts with SQL
  - build Streamlit app for museum data visualization
  - create data warehouse for art museum collections
  - implement ETL for cultural heritage data
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build production-ready ETL pipelines and analytics applications using the Harvard Art Museums API. The project demonstrates how to extract artifact data, transform nested JSON into relational schemas, load into SQL databases, and create interactive visualizations with Streamlit.

## What This Project Does

The Harvard Art Museums ETL Analytics App provides:
- **API Integration**: Paginated data extraction from Harvard Art Museums API with rate limiting
- **ETL Pipeline**: Transform nested JSON artifacts into normalized relational tables
- **Database Design**: Three-table schema (metadata, media, colors) with foreign key relationships
- **SQL Analytics**: 20+ predefined analytical queries for cultural heritage insights
- **Interactive Dashboard**: Streamlit-based visualization layer with Plotly charts

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
echo "HARVARD_API_KEY=your_api_key_here" > .env
echo "DB_HOST=your_db_host" >> .env
echo "DB_USER=your_db_user" >> .env
echo "DB_PASSWORD=your_db_password" >> .env
echo "DB_NAME=harvard_artifacts" >> .env
```

### Required Dependencies

```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
sqlalchemy
```

## Configuration

### Harvard API Key Setup

1. Get API key from: https://docs.google.com/forms/d/e/1FAIpQLSfkmEBqH76HLMMiCC-GPPnhcvHC9aJS86E32dAd8x9aHze-BA/viewform
2. Store in environment variables:

```python
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"
```

### Database Configuration

The app supports MySQL and TiDB Cloud. Configure connection:

```python
import mysql.connector
import os

def get_db_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME'),
        ssl_disabled=False
    )
```

## Database Schema

### Create Tables

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    classification VARCHAR(200),
    department VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accessionyear INT,
    provenance TEXT,
    copyright TEXT,
    url VARCHAR(500),
    totalpageviews INT,
    totaluniquepageviews INT
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    mediaid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    primaryimageurl VARCHAR(500),
    imagename VARCHAR(500),
    imagetype VARCHAR(100),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    colorid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    colorhex VARCHAR(10),
    colorpercent DECIMAL(5,2),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import time

def extract_artifacts(api_key, num_pages=10, page_size=100):
    """Extract artifact data from Harvard Art Museums API"""
    all_artifacts = []
    base_url = "https://api.harvardartmuseums.org/object"
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page
        }
        
        try:
            response = requests.get(base_url, params=params)
            response.raise_for_status()
            data = response.json()
            
            if 'records' in data:
                all_artifacts.extend(data['records'])
                print(f"Fetched page {page}: {len(data['records'])} artifacts")
            
            # Rate limiting
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return all_artifacts
```

### Transform: Normalize JSON to Relational Format

```python
import pandas as pd

def transform_artifacts(artifacts):
    """Transform nested JSON into normalized dataframes"""
    
    # Metadata transformation
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:200],
            'period': artifact.get('period', '')[:200],
            'century': artifact.get('century', '')[:100],
            'dated': artifact.get('dated', '')[:200],
            'classification': artifact.get('classification', '')[:200],
            'department': artifact.get('department', '')[:200],
            'technique': artifact.get('technique', '')[:500],
            'medium': artifact.get('medium', '')[:500],
            'dimensions': artifact.get('dimensions', '')[:500],
            'creditline': artifact.get('creditline', ''),
            'accessionyear': artifact.get('accessionyear'),
            'provenance': artifact.get('provenance', ''),
            'copyright': artifact.get('copyright', ''),
            'url': artifact.get('url', '')[:500],
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        }
        metadata_records.append(metadata)
        
        # Extract media (images)
        objectid = artifact.get('objectid')
        images = artifact.get('images', [])
        if images:
            for img in images:
                media = {
                    'objectid': objectid,
                    'baseimageurl': img.get('baseimageurl', '')[:500],
                    'iiifbaseuri': img.get('iiifbaseuri', '')[:500],
                    'primaryimageurl': artifact.get('primaryimageurl', '')[:500],
                    'imagename': img.get('imagename', '')[:500],
                    'imagetype': img.get('format', '')[:100]
                }
                media_records.append(media)
        
        # Extract colors
        colors = artifact.get('colors', [])
        if colors:
            for color in colors:
                color_record = {
                    'objectid': objectid,
                    'colorhex': color.get('hex', '')[:10],
                    'colorpercent': color.get('percent', 0)
                }
                color_records.append(color_record)
    
    df_metadata = pd.DataFrame(metadata_records)
    df_media = pd.DataFrame(media_records) if media_records else pd.DataFrame()
    df_colors = pd.DataFrame(color_records) if color_records else pd.DataFrame()
    
    return df_metadata, df_media, df_colors
```

### Load: Batch Insert into SQL

```python
def load_to_database(df_metadata, df_media, df_colors):
    """Load transformed data into SQL database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Load metadata
        metadata_sql = """
        INSERT INTO artifactmetadata 
        (objectid, title, culture, period, century, dated, classification, 
         department, technique, medium, dimensions, creditline, accessionyear, 
         provenance, copyright, url, totalpageviews, totaluniquepageviews)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        
        metadata_values = df_metadata.fillna('').values.tolist()
        cursor.executemany(metadata_sql, metadata_values)
        
        # Load media
        if not df_media.empty:
            media_sql = """
            INSERT INTO artifactmedia 
            (objectid, baseimageurl, iiifbaseuri, primaryimageurl, imagename, imagetype)
            VALUES (%s, %s, %s, %s, %s, %s)
            """
            media_values = df_media.fillna('').values.tolist()
            cursor.executemany(media_sql, media_values)
        
        # Load colors
        if not df_colors.empty:
            colors_sql = """
            INSERT INTO artifactcolors 
            (objectid, colorhex, colorpercent)
            VALUES (%s, %s, %s)
            """
            color_values = df_colors.fillna('').values.tolist()
            cursor.executemany(colors_sql, color_values)
        
        conn.commit()
        print(f"Loaded {len(df_metadata)} artifacts successfully")
        
    except Exception as e:
        conn.rollback()
        print(f"Error loading data: {e}")
    finally:
        cursor.close()
        conn.close()
```

## Running the ETL Pipeline

```python
# Complete ETL workflow
def run_etl_pipeline(api_key, num_pages=10):
    """Execute complete ETL pipeline"""
    
    # Extract
    print("Starting extraction...")
    artifacts = extract_artifacts(api_key, num_pages=num_pages)
    print(f"Extracted {len(artifacts)} artifacts")
    
    # Transform
    print("Starting transformation...")
    df_metadata, df_media, df_colors = transform_artifacts(artifacts)
    print(f"Transformed into {len(df_metadata)} metadata, {len(df_media)} media, {len(df_colors)} color records")
    
    # Load
    print("Starting load...")
    load_to_database(df_metadata, df_media, df_colors)
    print("ETL pipeline complete!")
    
    return df_metadata, df_media, df_colors

# Execute
if __name__ == "__main__":
    API_KEY = os.getenv('HARVARD_API_KEY')
    run_etl_pipeline(API_KEY, num_pages=5)
```

## SQL Analytics Queries

### Sample Analytical Queries

```python
# Query 1: Artifacts by Culture
QUERY_CULTURE_DISTRIBUTION = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL AND culture != ''
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 20;
"""

# Query 2: Artifacts by Century
QUERY_CENTURY_DISTRIBUTION = """
SELECT century, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE century IS NOT NULL AND century != ''
GROUP BY century
ORDER BY artifact_count DESC;
"""

# Query 3: Department Analysis
QUERY_DEPARTMENT_STATS = """
SELECT department, 
       COUNT(*) as total_artifacts,
       AVG(totalpageviews) as avg_pageviews
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY total_artifacts DESC;
"""

# Query 4: Media Availability
QUERY_MEDIA_AVAILABILITY = """
SELECT 
    COUNT(DISTINCT m.objectid) as artifacts_with_images,
    (SELECT COUNT(*) FROM artifactmetadata) as total_artifacts,
    ROUND(COUNT(DISTINCT m.objectid) * 100.0 / 
          (SELECT COUNT(*) FROM artifactmetadata), 2) as percentage_with_images
FROM artifactmedia m;
"""

# Query 5: Top Colors Used
QUERY_TOP_COLORS = """
SELECT colorhex, 
       COUNT(*) as usage_count,
       AVG(colorpercent) as avg_percentage
FROM artifactcolors
GROUP BY colorhex
ORDER BY usage_count DESC
LIMIT 15;
"""

# Query 6: Classification by Period
QUERY_CLASSIFICATION_PERIOD = """
SELECT classification, period, COUNT(*) as count
FROM artifactmetadata
WHERE classification IS NOT NULL AND period IS NOT NULL
GROUP BY classification, period
ORDER BY count DESC
LIMIT 20;
"""
```

### Execute Query in Streamlit

```python
import streamlit as st
import pandas as pd

def execute_query(query):
    """Execute SQL query and return results as DataFrame"""
    conn = get_db_connection()
    try:
        df = pd.read_sql(query, conn)
        return df
    except Exception as e:
        st.error(f"Query error: {e}")
        return pd.DataFrame()
    finally:
        conn.close()
```

## Streamlit Dashboard Implementation

### Main App Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Art Museums Analytics", layout="wide")
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    st.markdown("**Data Engineering & Analytics Application**")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Navigate",
        ["Data Collection", "Analytics Dashboard", "ETL Pipeline Status"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "Analytics Dashboard":
        show_analytics_dashboard()
    else:
        show_etl_status()

def show_data_collection():
    st.header("📡 Data Collection from Harvard API")
    
    col1, col2 = st.columns(2)
    with col1:
        num_pages = st.number_input("Number of pages", min_value=1, max_value=50, value=5)
    with col2:
        page_size = st.number_input("Records per page", min_value=10, max_value=100, value=100)
    
    if st.button("Start ETL Pipeline"):
        with st.spinner("Running ETL pipeline..."):
            api_key = os.getenv('HARVARD_API_KEY')
            df_metadata, df_media, df_colors = run_etl_pipeline(api_key, num_pages=num_pages)
            
            st.success(f"✅ Loaded {len(df_metadata)} artifacts!")
            st.metric("Metadata Records", len(df_metadata))
            st.metric("Media Records", len(df_media))
            st.metric("Color Records", len(df_colors))

def show_analytics_dashboard():
    st.header("📊 SQL Analytics Dashboard")
    
    queries = {
        "Culture Distribution": QUERY_CULTURE_DISTRIBUTION,
        "Century Distribution": QUERY_CENTURY_DISTRIBUTION,
        "Department Statistics": QUERY_DEPARTMENT_STATS,
        "Media Availability": QUERY_MEDIA_AVAILABILITY,
        "Top Colors": QUERY_TOP_COLORS,
        "Classification by Period": QUERY_CLASSIFICATION_PERIOD
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        df = execute_query(queries[selected_query])
        
        if not df.empty:
            st.subheader("Query Results")
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df.columns) >= 2:
                fig = px.bar(
                    df.head(20), 
                    x=df.columns[0], 
                    y=df.columns[1],
                    title=selected_query
                )
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Pattern 1: Incremental Data Loading

```python
def incremental_load(api_key, last_updated_date):
    """Load only new artifacts since last update"""
    params = {
        'apikey': api_key,
        'hasimage': 1,
        'after': last_updated_date
    }
    response = requests.get(BASE_URL, params=params)
    return response.json().get('records', [])
```

### Pattern 2: Error Handling with Retry Logic

```python
from time import sleep

def fetch_with_retry(url, params, max_retries=3):
    """Fetch with exponential backoff retry"""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
            sleep(2 ** attempt)
```

### Pattern 3: Data Quality Validation

```python
def validate_artifact_data(df):
    """Validate data quality before loading"""
    issues = []
    
    # Check for required fields
    if df['objectid'].isnull().any():
        issues.append("Missing objectid values")
    
    # Check data types
    if not pd.api.types.is_numeric_dtype(df['objectid']):
        issues.append("objectid must be numeric")
    
    # Check string length constraints
    if (df['title'].str.len() > 500).any():
        issues.append("Title exceeds 500 characters")
    
    return issues
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# Run ETL pipeline standalone
python etl_pipeline.py

# Run specific query
python -c "from analytics import execute_query; print(execute_query('SELECT COUNT(*) FROM artifactmetadata'))"
```

## Troubleshooting

### Issue: API Rate Limiting

```python
# Add delay between requests
import time

for page in range(1, num_pages + 1):
    response = requests.get(url, params=params)
    time.sleep(1)  # 1 second delay
```

### Issue: Database Connection Timeout

```python
# Increase connection timeout
conn = mysql.connector.connect(
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME'),
    connection_timeout=30
)
```

### Issue: Large Dataset Memory Error

```python
# Use chunked processing
def load_in_chunks(df, chunk_size=1000):
    for i in range(0, len(df), chunk_size):
        chunk = df.iloc[i:i+chunk_size]
        load_to_database(chunk, None, None)
```

### Issue: Unicode Characters in Artifacts

```python
# Handle encoding properly
df_metadata['title'] = df_metadata['title'].apply(
    lambda x: x.encode('utf-8', 'ignore').decode('utf-8') if isinstance(x, str) else x
)
```

This skill provides complete coverage for building ETL pipelines and analytics dashboards using the Harvard Art Museums API with production-ready code patterns.
