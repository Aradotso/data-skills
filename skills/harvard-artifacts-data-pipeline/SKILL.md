---
name: harvard-artifacts-data-pipeline
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I build a data pipeline for Harvard Art Museums API
  - create an ETL pipeline for museum artifact data
  - analyze Harvard Art Museums data with SQL
  - build a Streamlit dashboard for art collection analytics
  - extract and visualize Harvard museum artifacts
  - set up database schema for art museum data
  - query Harvard Art Museums API with Python
  - create analytics for museum collection data
---

# Harvard Artifacts Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering pipelines using the Harvard Art Museums API. The project demonstrates ETL workflows, SQL database design, analytics queries, and interactive Streamlit visualizations for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:
- API integration with Harvard Art Museums to fetch artifact data
- ETL pipeline to extract, transform, and load data into relational databases
- SQL schema design for artifact metadata, media, and color information
- Pre-built analytical SQL queries for common insights
- Interactive Streamlit dashboard with Plotly visualizations

**Architecture Flow:** API → ETL → SQL Database → Analytics Queries → Streamlit Visualization

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

### API Key Setup

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

```python
# Create .env file
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Database Connection

```python
import mysql.connector
import os
from dotenv import load_dotenv

load_dotenv()

def get_db_connection():
    """Establish database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
```

## Database Schema

### Create Tables

```python
def create_tables(connection):
    """Create database schema for artifact data"""
    cursor = connection.cursor()
    
    # Artifact Metadata Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(255),
            classification VARCHAR(255),
            century VARCHAR(100),
            dated VARCHAR(255),
            department VARCHAR(255),
            division VARCHAR(255),
            period VARCHAR(255),
            technique VARCHAR(500),
            medium VARCHAR(500),
            dimensions VARCHAR(500),
            creditline TEXT,
            accessionyear INT,
            provenance TEXT,
            url VARCHAR(500)
        )
    """)
    
    # Artifact Media Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            baseimageurl VARCHAR(500),
            primaryimageurl VARCHAR(500),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    # Artifact Colors Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(50),
            spectrum VARCHAR(100),
            percent FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    connection.commit()
    cursor.close()
```

## ETL Pipeline

### Extract: Fetch Data from API

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """
    Extract artifact data from Harvard Art Museums API
    Handles pagination and rate limiting
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=100)
total_records = data['info']['totalrecords']
artifacts = data['records']
```

### Transform: Process and Clean Data

```python
import pandas as pd

def transform_artifact_data(raw_data):
    """
    Transform nested JSON into normalized DataFrames
    """
    artifacts = []
    media_records = []
    color_records = []
    
    for record in raw_data['records']:
        # Extract metadata
        artifact = {
            'id': record.get('id'),
            'title': record.get('title'),
            'culture': record.get('culture'),
            'classification': record.get('classification'),
            'century': record.get('century'),
            'dated': record.get('dated'),
            'department': record.get('department'),
            'division': record.get('division'),
            'period': record.get('period'),
            'technique': record.get('technique'),
            'medium': record.get('medium'),
            'dimensions': record.get('dimensions'),
            'creditline': record.get('creditline'),
            'accessionyear': record.get('accessionyear'),
            'provenance': record.get('provenance'),
            'url': record.get('url')
        }
        artifacts.append(artifact)
        
        # Extract media information
        if record.get('primaryimageurl'):
            media_records.append({
                'artifact_id': record.get('id'),
                'baseimageurl': record.get('baseimageurl'),
                'primaryimageurl': record.get('primaryimageurl')
            })
        
        # Extract color information
        if record.get('colors'):
            for color in record['colors']:
                color_records.append({
                    'artifact_id': record.get('id'),
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'percent': color.get('percent')
                })
    
    return (
        pd.DataFrame(artifacts),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### Load: Insert Data into SQL

```python
def load_artifact_data(connection, metadata_df, media_df, colors_df):
    """
    Load transformed data into SQL database with batch inserts
    """
    cursor = connection.cursor()
    
    # Insert metadata
    metadata_query = """
        INSERT IGNORE INTO artifactmetadata 
        (id, title, culture, classification, century, dated, department, 
         division, period, technique, medium, dimensions, creditline, 
         accessionyear, provenance, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    """
    cursor.executemany(metadata_query, metadata_df.values.tolist())
    
    # Insert media
    if not media_df.empty:
        media_query = """
            INSERT INTO artifactmedia 
            (artifact_id, baseimageurl, primaryimageurl)
            VALUES (%s, %s, %s)
        """
        cursor.executemany(media_query, media_df.values.tolist())
    
    # Insert colors
    if not colors_df.empty:
        colors_query = """
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, percent)
            VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(colors_query, colors_df.values.tolist())
    
    connection.commit()
    cursor.close()
```

## Streamlit Application

### Main App Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password")
    
    # Database connection
    if st.sidebar.button("Connect to Database"):
        try:
            conn = get_db_connection()
            st.sidebar.success("Database connected!")
        except Exception as e:
            st.sidebar.error(f"Connection failed: {e}")
    
    # ETL Pipeline Section
    st.header("📥 Data Collection")
    num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=50, value=5)
    
    if st.button("Run ETL Pipeline"):
        run_etl_pipeline(api_key, num_pages)
    
    # Analytics Section
    st.header("📊 Analytics Queries")
    query_options = {
        "Artifacts by Culture": "SELECT culture, COUNT(*) as count FROM artifactmetadata WHERE culture IS NOT NULL GROUP BY culture ORDER BY count DESC LIMIT 10",
        "Artifacts by Century": "SELECT century, COUNT(*) as count FROM artifactmetadata WHERE century IS NOT NULL GROUP BY century ORDER BY count DESC",
        "Top Departments": "SELECT department, COUNT(*) as count FROM artifactmetadata GROUP BY department ORDER BY count DESC LIMIT 10",
        "Color Distribution": "SELECT color, COUNT(*) as count FROM artifactcolors GROUP BY color ORDER BY count DESC LIMIT 15",
        "Media Availability": "SELECT COUNT(DISTINCT am.artifact_id) as with_media, (SELECT COUNT(*) FROM artifactmetadata) as total FROM artifactmedia am"
    }
    
    selected_query = st.selectbox("Select Analytics Query", list(query_options.keys()))
    
    if st.button("Run Query"):
        conn = get_db_connection()
        df = pd.read_sql(query_options[selected_query], conn)
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2 and df.shape[0] > 1:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                        title=selected_query)
            st.plotly_chart(fig)
        
        conn.close()

def run_etl_pipeline(api_key, num_pages):
    """Execute full ETL pipeline"""
    progress_bar = st.progress(0)
    conn = get_db_connection()
    
    for page in range(1, num_pages + 1):
        # Extract
        raw_data = fetch_artifacts(api_key, page=page)
        
        # Transform
        metadata_df, media_df, colors_df = transform_artifact_data(raw_data)
        
        # Load
        load_artifact_data(conn, metadata_df, media_df, colors_df)
        
        progress_bar.progress(page / num_pages)
        st.write(f"Processed page {page}/{num_pages}")
    
    conn.close()
    st.success("ETL Pipeline completed!")

if __name__ == "__main__":
    main()
```

### Run the Application

```bash
streamlit run app.py
```

## Common Analytics Queries

### Top 10 Cultures with Most Artifacts

```sql
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10;
```

### Artifacts with Complete Media

```sql
SELECT am.id, am.title, am.culture, med.primaryimageurl
FROM artifactmetadata am
INNER JOIN artifactmedia med ON am.id = med.artifact_id
WHERE med.primaryimageurl IS NOT NULL;
```

### Color Spectrum Analysis

```sql
SELECT spectrum, AVG(percent) as avg_percent, COUNT(*) as color_count
FROM artifactcolors
GROUP BY spectrum
ORDER BY avg_percent DESC;
```

### Artifacts by Century and Department

```sql
SELECT century, department, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL AND department IS NOT NULL
GROUP BY century, department
ORDER BY century, count DESC;
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(api_key, page, max_retries=3):
    """Fetch data with retry logic for rate limiting"""
    for attempt in range(max_retries):
        try:
            data = fetch_artifacts(api_key, page)
            return data
        except Exception as e:
            if "429" in str(e):  # Rate limit error
                wait_time = 2 ** attempt  # Exponential backoff
                time.sleep(wait_time)
            else:
                raise e
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
def get_db_connection_with_retry():
    """Database connection with error handling"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME'),
            connect_timeout=10
        )
        return connection
    except mysql.connector.Error as err:
        if err.errno == 2003:
            raise Exception("Cannot connect to database server")
        elif err.errno == 1045:
            raise Exception("Invalid database credentials")
        else:
            raise Exception(f"Database error: {err}")
```

### Handling Missing Data

```python
def safe_get(dictionary, key, default=None):
    """Safely extract values from nested dictionaries"""
    try:
        return dictionary.get(key, default)
    except (AttributeError, KeyError):
        return default

# Usage in transformation
artifact = {
    'id': safe_get(record, 'id'),
    'title': safe_get(record, 'title', 'Untitled'),
    'culture': safe_get(record, 'culture', 'Unknown')
}
```

## Best Practices

1. **Always use environment variables** for API keys and database credentials
2. **Implement pagination** when fetching large datasets from the API
3. **Use batch inserts** for better database performance
4. **Add proper error handling** for API failures and database issues
5. **Create indexes** on frequently queried columns (culture, century, department)
6. **Validate data** before inserting into the database
7. **Use connection pooling** for production deployments
