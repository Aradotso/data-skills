---
name: harvard-artifacts-etl-analytics
description: End-to-end ETL pipeline and analytics for Harvard Art Museums API with Streamlit visualization
triggers:
  - build etl pipeline for harvard art museums
  - create analytics dashboard with streamlit
  - fetch harvard museum artifacts data
  - setup sql database for artifact analytics
  - visualize museum collection data
  - implement harvard api data engineering workflow
  - analyze art museum metadata with sql
  - create museum artifacts collection pipeline
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build end-to-end data engineering pipelines using the Harvard Art Museums API. The project demonstrates ETL operations, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact collections.

## What This Project Does

Harvard-Artifacts-Collection-Data-Engineering-Analytics-App is a complete data pipeline that:
- Extracts artifact data from Harvard Art Museums API with pagination and rate limiting
- Transforms nested JSON into normalized relational database schema
- Loads data into MySQL/TiDB Cloud with batch inserts
- Provides 20+ analytical SQL queries for insights
- Visualizes results through interactive Streamlit dashboards with Plotly charts

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

## Installation

### Prerequisites
```bash
# Python 3.8+
python --version

# Install dependencies
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Project Setup
```bash
# Clone repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install requirements
pip install -r requirements.txt

# Set up environment variables
cat > .env << EOF
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
EOF
```

### Get Harvard API Key
1. Visit https://www.harvardartmuseums.org/collections/api
2. Request an API key
3. Store in `.env` file as `HARVARD_API_KEY`

## Database Schema

The ETL pipeline creates three normalized tables:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    dated VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    medium VARCHAR(500),
    department VARCHAR(255),
    division VARCHAR(255),
    primaryimageurl TEXT,
    copyright VARCHAR(500),
    accession_year INT
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    media_type VARCHAR(100),
    baseimageurl TEXT,
    format VARCHAR(50),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color_hex VARCHAR(10),
    color_name VARCHAR(100),
    color_percent FLOAT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## Key Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_records=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    all_artifacts = []
    page = 1
    size = 100  # API limit per page
    
    while len(all_artifacts) < num_records:
        params = {
            'apikey': api_key,
            'size': min(size, num_records - len(all_artifacts)),
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            
            if not records:
                break
                
            all_artifacts.extend(records)
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return all_artifacts[:num_records]
```

### 2. ETL Transformation

```python
import pandas as pd

def transform_artifacts(raw_data):
    """Transform nested JSON to normalized dataframes"""
    
    # Main metadata extraction
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title', ''),
            'culture': artifact.get('culture', ''),
            'dated': artifact.get('dated', ''),
            'period': artifact.get('period', ''),
            'century': artifact.get('century', ''),
            'classification': artifact.get('classification', ''),
            'medium': artifact.get('medium', ''),
            'department': artifact.get('department', ''),
            'division': artifact.get('division', ''),
            'primaryimageurl': artifact.get('primaryimageurl', ''),
            'copyright': artifact.get('copyright', ''),
            'accession_year': artifact.get('accessionyear')
        }
        metadata_list.append(metadata)
        
        # Extract media (images)
        objectid = artifact.get('objectid')
        for image in artifact.get('images', []):
            media = {
                'objectid': objectid,
                'media_type': 'image',
                'baseimageurl': image.get('baseimageurl', ''),
                'format': image.get('format', '')
            }
            media_list.append(media)
        
        # Extract colors
        for color in artifact.get('colors', []):
            color_data = {
                'objectid': objectid,
                'color_hex': color.get('hex', ''),
                'color_name': color.get('color', ''),
                'color_percent': color.get('percent', 0.0)
            }
            colors_list.append(color_data)
    
    return {
        'metadata': pd.DataFrame(metadata_list),
        'media': pd.DataFrame(media_list),
        'colors': pd.DataFrame(colors_list)
    }
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection():
    """Create MySQL database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_to_database(dataframes):
    """Batch insert dataframes into SQL tables"""
    connection = create_database_connection()
    
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    try:
        # Load metadata
        metadata_df = dataframes['metadata']
        for _, row in metadata_df.iterrows():
            sql = """
            INSERT INTO artifactmetadata 
            (objectid, title, culture, dated, period, century, 
             classification, medium, department, division, 
             primaryimageurl, copyright, accession_year)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE 
            title=VALUES(title), culture=VALUES(culture)
            """
            cursor.execute(sql, tuple(row))
        
        # Load media
        media_df = dataframes['media']
        for _, row in media_df.iterrows():
            sql = """
            INSERT INTO artifactmedia 
            (objectid, media_type, baseimageurl, format)
            VALUES (%s, %s, %s, %s)
            """
            cursor.execute(sql, tuple(row))
        
        # Load colors
        colors_df = dataframes['colors']
        for _, row in colors_df.iterrows():
            sql = """
            INSERT INTO artifactcolors 
            (objectid, color_hex, color_name, color_percent)
            VALUES (%s, %s, %s, %s)
            """
            cursor.execute(sql, tuple(row))
        
        connection.commit()
        return True
        
    except Error as e:
        print(f"Load error: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()
```

### 4. Analytical Queries

```python
ANALYTICAL_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture 
        ORDER BY count DESC 
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY artifact_count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "Top Colors Used": """
        SELECT color_name, COUNT(*) as usage_count,
               AVG(color_percent) as avg_percentage
        FROM artifactcolors
        WHERE color_name IS NOT NULL
        GROUP BY color_name
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "Media Format Analysis": """
        SELECT format, COUNT(*) as image_count
        FROM artifactmedia
        WHERE format IS NOT NULL
        GROUP BY format
        ORDER BY image_count DESC
    """,
    
    "Artifacts with Multiple Images": """
        SELECT a.objectid, a.title, COUNT(m.media_id) as image_count
        FROM artifactmetadata a
        JOIN artifactmedia m ON a.objectid = m.objectid
        GROUP BY a.objectid, a.title
        HAVING image_count > 1
        ORDER BY image_count DESC
        LIMIT 20
    """
}

def execute_analytical_query(query_name):
    """Execute predefined analytical query"""
    connection = create_database_connection()
    
    if not connection:
        return None
    
    query = ANALYTICAL_QUERIES.get(query_name)
    
    if not query:
        return None
    
    try:
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        print(f"Query error: {e}")
        return None
    finally:
        connection.close()
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🏛️",
        layout="wide"
    )
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar - ETL Controls
    with st.sidebar:
        st.header("ETL Pipeline")
        
        num_records = st.number_input(
            "Number of artifacts to fetch",
            min_value=10,
            max_value=1000,
            value=100,
            step=10
        )
        
        if st.button("Run ETL Pipeline"):
            with st.spinner("Fetching artifacts..."):
                raw_data = fetch_artifacts(num_records)
                st.success(f"Fetched {len(raw_data)} artifacts")
            
            with st.spinner("Transforming data..."):
                transformed = transform_artifacts(raw_data)
                st.success("Data transformed")
            
            with st.spinner("Loading to database..."):
                success = load_to_database(transformed)
                if success:
                    st.success("Data loaded successfully!")
                else:
                    st.error("Failed to load data")
    
    # Main area - Analytics
    st.header("📊 Analytics Dashboard")
    
    query_choice = st.selectbox(
        "Select Analysis",
        list(ANALYTICAL_QUERIES.keys())
    )
    
    if st.button("Run Analysis"):
        with st.spinner("Executing query..."):
            results = execute_analytical_query(query_choice)
            
            if results is not None and not results.empty:
                st.subheader("Query Results")
                st.dataframe(results, use_container_width=True)
                
                # Auto-visualization
                if len(results.columns) >= 2:
                    fig = px.bar(
                        results,
                        x=results.columns[0],
                        y=results.columns[1],
                        title=query_choice
                    )
                    st.plotly_chart(fig, use_container_width=True)
            else:
                st.warning("No results found")

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Rate Limiting for API Calls

```python
import time

def fetch_with_rate_limit(url, params, delay=0.5):
    """Fetch with rate limiting to avoid API throttling"""
    response = requests.get(url, params=params)
    time.sleep(delay)  # 500ms delay between requests
    return response
```

### Incremental ETL Updates

```python
def incremental_etl(last_update_timestamp):
    """Fetch only new/updated artifacts since last ETL run"""
    params = {
        'apikey': os.getenv('HARVARD_API_KEY'),
        'updatedafter': last_update_timestamp
    }
    # Continue with ETL pipeline
```

### Error Handling for API Failures

```python
def robust_fetch(max_retries=3):
    """Fetch with retry logic"""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

## Troubleshooting

### API Key Issues
```python
# Verify API key is loaded
import os
from dotenv import load_dotenv

load_dotenv()
api_key = os.getenv('HARVARD_API_KEY')
if not api_key:
    raise ValueError("HARVARD_API_KEY not found in environment variables")
```

### Database Connection Failures
```python
# Test connection
def test_db_connection():
    try:
        conn = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD')
        )
        print("Connection successful")
        conn.close()
    except Error as e:
        print(f"Connection failed: {e}")
```

### Memory Issues with Large Datasets
```python
# Process in chunks
def batch_etl(total_records, batch_size=100):
    for offset in range(0, total_records, batch_size):
        batch = fetch_artifacts(batch_size, offset)
        transformed = transform_artifacts(batch)
        load_to_database(transformed)
```

### Duplicate Key Errors
```python
# Use UPSERT pattern
sql = """
INSERT INTO artifactmetadata (...) 
VALUES (...) 
ON DUPLICATE KEY UPDATE 
    title=VALUES(title),
    culture=VALUES(culture)
"""
```

This skill provides complete ETL pipeline implementation for museum artifact data with production-ready patterns for data engineering workflows.
