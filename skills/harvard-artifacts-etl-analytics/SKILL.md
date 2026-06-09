---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - how do I build an ETL pipeline with the Harvard Art Museums API
  - create a data engineering project with museum artifacts
  - set up SQL analytics for Harvard art collection data
  - build a Streamlit dashboard for museum artifact data
  - extract and transform Harvard museum API data
  - create artifact analytics with SQL queries
  - implement a data pipeline for art museum collections
  - visualize Harvard artifacts data with Plotly
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering and analytics application using the Harvard Art Museums API. It demonstrates ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What It Does

- **API Integration**: Collects artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads artifact metadata, media, and color data into SQL databases
- **SQL Analytics**: Stores data in relational tables (MySQL/TiDB Cloud) with proper relationships
- **Interactive Dashboard**: Provides 20+ predefined analytical queries with Plotly visualizations
- **Real-time Insights**: Culture distribution, century analysis, media availability, color patterns

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Requirements typically include**:
```
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
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_username
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Request an API key (free)
3. Add to your `.env` file

### Database Setup

```sql
CREATE DATABASE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    technique VARCHAR(500),
    dated VARCHAR(200),
    url TEXT,
    accession_number VARCHAR(100)
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    base_url TEXT,
    format VARCHAR(50),
    height INT,
    width INT,
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

def fetch_artifacts(num_records=100, page=1):
    """Fetch artifacts from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'size': min(num_records, 100),  # API limit per page
        'page': page,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Fetch with pagination
def fetch_all_artifacts(total_records=500):
    """Fetch multiple pages of artifacts"""
    all_artifacts = []
    page = 1
    per_page = 100
    
    while len(all_artifacts) < total_records:
        data = fetch_artifacts(num_records=per_page, page=page)
        artifacts = data.get('records', [])
        all_artifacts.extend(artifacts)
        
        if not artifacts or len(artifacts) < per_page:
            break
            
        page += 1
    
    return all_artifacts[:total_records]
```

### 2. ETL Transformation

```python
import pandas as pd

def transform_artifacts(raw_artifacts):
    """Transform raw API data into structured dataframes"""
    
    # Metadata transformation
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_artifacts:
        # Extract metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'technique': artifact.get('technique', 'Unknown'),
            'dated': artifact.get('dated', 'Unknown'),
            'url': artifact.get('url', ''),
            'accession_number': artifact.get('accessionNumber', '')
        })
        
        # Extract media information
        images = artifact.get('images', [])
        for image in images:
            media_records.append({
                'artifact_id': artifact.get('id'),
                'base_url': image.get('baseimageurl', ''),
                'format': image.get('format', ''),
                'height': image.get('height', 0),
                'width': image.get('width', 0)
            })
        
        # Extract color information
        colors = artifact.get('colors', [])
        for color in colors:
            color_records.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color', ''),
                'spectrum': color.get('spectrum', ''),
                'percentage': color.get('percent', 0.0)
            })
    
    metadata_df = pd.DataFrame(metadata_records)
    media_df = pd.DataFrame(media_records)
    colors_df = pd.DataFrame(color_records)
    
    return metadata_df, media_df, colors_df
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
        raise Exception(f"Database connection error: {e}")

def load_to_database(metadata_df, media_df, colors_df):
    """Batch insert dataframes into SQL database"""
    connection = get_db_connection()
    cursor = connection.cursor()
    
    try:
        # Load metadata
        metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, technique, dated, url, accession_number)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        metadata_values = metadata_df.values.tolist()
        cursor.executemany(metadata_query, metadata_values)
        
        # Load media
        media_query = """
        INSERT INTO artifactmedia (artifact_id, base_url, format, height, width)
        VALUES (%s, %s, %s, %s, %s)
        """
        media_values = media_df.values.tolist()
        cursor.executemany(media_query, media_values)
        
        # Load colors
        colors_query = """
        INSERT INTO artifactcolors (artifact_id, color, spectrum, percentage)
        VALUES (%s, %s, %s, %s)
        """
        colors_values = colors_df.values.tolist()
        cursor.executemany(colors_query, colors_values)
        
        connection.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        connection.rollback()
        raise Exception(f"Database load error: {e}")
    finally:
        cursor.close()
        connection.close()
```

### 4. SQL Analytics Queries

```python
def execute_query(query):
    """Execute SQL query and return results as dataframe"""
    connection = get_db_connection()
    try:
        df = pd.read_sql(query, connection)
        return df
    finally:
        connection.close()

# Sample analytical queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE culture != 'Unknown'
        GROUP BY culture 
        ORDER BY count DESC 
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE century != 'Unknown'
        GROUP BY century 
        ORDER BY count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as count 
        FROM artifactmetadata 
        GROUP BY department 
        ORDER BY count DESC
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as frequency, AVG(percentage) as avg_percentage
        FROM artifactcolors 
        GROUP BY color 
        ORDER BY frequency DESC 
        LIMIT 10
    """,
    
    "Media Format Distribution": """
        SELECT format, COUNT(*) as count 
        FROM artifactmedia 
        WHERE format IS NOT NULL 
        GROUP BY format 
        ORDER BY count DESC
    """,
    
    "Artifacts with Most Images": """
        SELECT m.title, m.culture, COUNT(am.id) as image_count
        FROM artifactmetadata m
        JOIN artifactmedia am ON m.id = am.artifact_id
        GROUP BY m.id, m.title, m.culture
        ORDER BY image_count DESC
        LIMIT 10
    """
}
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "Analytics Dashboard", "Custom Query"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "Analytics Dashboard":
        show_analytics_dashboard()
    else:
        show_custom_query()

def show_data_collection():
    """ETL interface"""
    st.header("📥 Data Collection & ETL")
    
    num_records = st.number_input("Number of records to collect", 
                                  min_value=10, max_value=1000, value=100)
    
    if st.button("Start ETL Process"):
        with st.spinner("Fetching data from API..."):
            artifacts = fetch_all_artifacts(num_records)
            st.success(f"✅ Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            st.success("✅ Data transformed")
        
        with st.spinner("Loading to database..."):
            load_to_database(metadata_df, media_df, colors_df)
            st.success("✅ Data loaded to database")

def show_analytics_dashboard():
    """Display analytics with visualizations"""
    st.header("📊 Analytics Dashboard")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        query = ANALYTICS_QUERIES[query_name]
        
        with st.spinner("Executing query..."):
            df = execute_query(query)
        
        st.subheader("Query Results")
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                        title=query_name)
            st.plotly_chart(fig, use_container_width=True)

def show_custom_query():
    """Custom SQL query interface"""
    st.header("🔍 Custom SQL Query")
    
    query = st.text_area("Enter SQL Query", height=150)
    
    if st.button("Execute"):
        try:
            df = execute_query(query)
            st.success(f"✅ Query returned {len(df)} rows")
            st.dataframe(df)
        except Exception as e:
            st.error(f"❌ Query error: {e}")

if __name__ == "__main__":
    main()
```

## Common Patterns

### Complete ETL Pipeline

```python
def run_full_pipeline(num_records=500):
    """Execute complete ETL pipeline"""
    print("Starting ETL pipeline...")
    
    # Extract
    print(f"Extracting {num_records} records...")
    artifacts = fetch_all_artifacts(num_records)
    
    # Transform
    print("Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts(artifacts)
    
    # Load
    print("Loading to database...")
    load_to_database(metadata_df, media_df, colors_df)
    
    print("Pipeline completed successfully!")
    return {
        'metadata': len(metadata_df),
        'media': len(media_df),
        'colors': len(colors_df)
    }
```

### Error Handling & Rate Limiting

```python
import time

def fetch_with_retry(num_records, max_retries=3):
    """Fetch with retry logic and rate limiting"""
    for attempt in range(max_retries):
        try:
            time.sleep(1)  # Rate limiting
            return fetch_artifacts(num_records)
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            print(f"Retry {attempt + 1}/{max_retries} after error: {e}")
            time.sleep(2 ** attempt)  # Exponential backoff
```

## Troubleshooting

### API Key Issues
- Ensure `HARVARD_API_KEY` is set in `.env`
- Verify API key is active at https://www.harvardartmuseums.org/collections/api
- Check rate limits (typically 2,500 requests/day)

### Database Connection Errors
- Verify database credentials in `.env`
- Ensure database exists: `CREATE DATABASE harvard_artifacts;`
- Check firewall rules for remote database connections
- For TiDB Cloud, ensure SSL/TLS settings are configured

### Memory Issues with Large Datasets
- Process data in smaller batches (100-500 records)
- Use chunked inserts instead of loading entire dataframes
- Clear unused dataframes with `del df` and `gc.collect()`

### Streamlit Performance
- Use `@st.cache_data` for expensive operations
- Limit query result sizes with SQL `LIMIT` clauses
- Use connection pooling for database operations

```python
@st.cache_data(ttl=3600)
def cached_query(query):
    """Cache query results for 1 hour"""
    return execute_query(query)
```
