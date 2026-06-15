---
name: harvard-artifacts-etl-analytics
description: End-to-end ETL pipeline and analytics for Harvard Art Museums API with Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums
  - create analytics dashboard for museum artifacts
  - extract data from Harvard Art Museums API
  - set up artifact data warehouse with SQL
  - visualize museum collection data with Streamlit
  - analyze Harvard artifacts by culture and century
  - build data engineering pipeline for art museums
  - create museum artifact analytics application
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build and work with end-to-end data engineering pipelines using the Harvard Art Museums API. The project demonstrates ETL operations, SQL database design, analytics queries, and interactive Streamlit visualization for museum artifact data.

## What It Does

The Harvard Artifacts Collection application:
- Extracts artifact data from the Harvard Art Museums API with pagination and rate limiting
- Transforms nested JSON into relational database tables
- Loads data into MySQL/TiDB Cloud with proper schema design
- Executes analytical SQL queries for insights
- Visualizes results using Plotly charts in a Streamlit dashboard

**Architecture Flow**: API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages**:
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

Set up environment variables for API and database credentials:

```bash
# .env file
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

## Database Schema

The application uses three main tables with foreign key relationships:

```sql
-- Artifact Metadata (main table)
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    period VARCHAR(200),
    division VARCHAR(200),
    rank INT,
    description TEXT
);

-- Artifact Media (1-to-many relationship)
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    media_url VARCHAR(1000),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors (1-to-many relationship)
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    color_percent FLOAT,
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

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        data = response.json()
        return data['records'], data['info']
    else:
        raise Exception(f"API Error: {response.status_code}")

# Example: Fetch multiple pages with rate limiting
import time

def fetch_all_artifacts(max_pages=10):
    """Fetch multiple pages of artifacts with rate limiting"""
    all_records = []
    
    for page in range(1, max_pages + 1):
        records, info = fetch_artifacts(page=page, size=100)
        all_records.extend(records)
        
        print(f"Fetched page {page}/{info['pages']} - {len(records)} records")
        
        # Rate limiting: 1 second between requests
        time.sleep(1)
        
        if page >= info['pages']:
            break
    
    return all_records
```

### 2. ETL Transformation

```python
import pandas as pd

def transform_artifact_metadata(records):
    """Transform API records into metadata DataFrame"""
    metadata = []
    
    for record in records:
        metadata.append({
            'id': record.get('id'),
            'title': record.get('title', 'Untitled'),
            'culture': record.get('culture'),
            'century': record.get('century'),
            'classification': record.get('classification'),
            'department': record.get('department'),
            'dated': record.get('dated'),
            'period': record.get('period'),
            'division': record.get('division'),
            'rank': record.get('rank', 0),
            'description': record.get('description')
        })
    
    return pd.DataFrame(metadata)

def transform_artifact_media(records):
    """Extract media/images from nested JSON"""
    media_list = []
    
    for record in records:
        artifact_id = record.get('id')
        images = record.get('images', [])
        
        for img in images:
            media_list.append({
                'artifact_id': artifact_id,
                'media_type': 'image',
                'media_url': img.get('baseimageurl')
            })
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(records):
    """Extract color data from nested JSON"""
    colors_list = []
    
    for record in records:
        artifact_id = record.get('id')
        colors = record.get('colors', [])
        
        for color in colors:
            colors_list.append({
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'color_percent': color.get('percent', 0.0)
            })
    
    return pd.DataFrame(colors_list)
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_to_database(metadata_df, media_df, colors_df):
    """Batch load DataFrames into SQL database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Load metadata
        for _, row in metadata_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmetadata 
                (id, title, culture, century, classification, department, 
                 dated, period, division, rank, description)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE
                title=VALUES(title), culture=VALUES(culture)
            """, tuple(row))
        
        # Load media
        for _, row in media_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmedia (artifact_id, media_type, media_url)
                VALUES (%s, %s, %s)
            """, tuple(row))
        
        # Load colors
        for _, row in colors_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactcolors (artifact_id, color, color_percent)
                VALUES (%s, %s, %s)
            """, tuple(row))
        
        conn.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()
```

### 4. Analytics Queries

```python
def run_analytics_query(query):
    """Execute analytical SQL query and return DataFrame"""
    conn = get_db_connection()
    
    try:
        df = pd.read_sql(query, conn)
        return df
    finally:
        conn.close()

# Example analytics queries
ANALYTICS_QUERIES = {
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
    
    "Top Colors in Collection": """
        SELECT color, COUNT(*) as artifact_count, 
               AVG(color_percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "Artifacts with Most Images": """
        SELECT a.id, a.title, COUNT(m.id) as image_count
        FROM artifactmetadata a
        JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY a.id, a.title
        ORDER BY image_count DESC
        LIMIT 20
    """
}
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Artifacts Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Queries")
    selected_query = st.sidebar.selectbox(
        "Choose Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    # Execute query
    if st.button("Run Analysis"):
        with st.spinner("Executing query..."):
            query = ANALYTICS_QUERIES[selected_query]
            df = run_analytics_query(query)
            
            # Display results
            st.subheader("Query Results")
            st.dataframe(df)
            
            # Visualization
            if len(df) > 0 and len(df.columns) >= 2:
                st.subheader("Visualization")
                
                fig = px.bar(
                    df,
                    x=df.columns[0],
                    y=df.columns[1],
                    title=selected_query,
                    labels={df.columns[0]: df.columns[0].title(),
                           df.columns[1]: df.columns[1].title()}
                )
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Common Patterns

### Complete ETL Pipeline Execution

```python
def run_etl_pipeline(num_pages=5):
    """Complete ETL pipeline execution"""
    
    # Extract
    print("Extracting data from API...")
    records = fetch_all_artifacts(max_pages=num_pages)
    
    # Transform
    print("Transforming data...")
    metadata_df = transform_artifact_metadata(records)
    media_df = transform_artifact_media(records)
    colors_df = transform_artifact_colors(records)
    
    # Load
    print("Loading to database...")
    load_to_database(metadata_df, media_df, colors_df)
    
    print("ETL pipeline completed successfully!")
    return metadata_df, media_df, colors_df
```

### Error Handling and Retry Logic

```python
from time import sleep

def fetch_with_retry(page, max_retries=3):
    """Fetch with exponential backoff retry"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page=page)
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            wait_time = 2 ** attempt
            print(f"Retry {attempt + 1} after {wait_time}s: {e}")
            sleep(wait_time)
```

## Troubleshooting

**API Rate Limiting**: Add sleep between requests (1-2 seconds recommended)
```python
import time
time.sleep(1)  # Between API calls
```

**Database Connection Issues**: Verify credentials and network access
```python
# Test connection
try:
    conn = get_db_connection()
    print("Database connected successfully")
    conn.close()
except Error as e:
    print(f"Connection failed: {e}")
```

**Memory Issues with Large Datasets**: Use batch processing
```python
# Process in chunks
chunk_size = 100
for i in range(0, len(df), chunk_size):
    chunk = df.iloc[i:i+chunk_size]
    load_to_database(chunk, media_df, colors_df)
```

**Missing API Key**: Ensure `.env` file is in project root
```python
if not os.getenv('HARVARD_API_KEY'):
    raise ValueError("HARVARD_API_KEY not found in environment variables")
```
