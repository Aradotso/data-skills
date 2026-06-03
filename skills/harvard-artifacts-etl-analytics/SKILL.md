---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I fetch data from Harvard Art Museums API
  - build an ETL pipeline for museum artifacts
  - create analytics dashboard with Streamlit and SQL
  - query Harvard artifacts collection data
  - set up museum data engineering pipeline
  - visualize art collection data with Plotly
  - connect to TiDB for artifact storage
  - paginate through Harvard API results
---

# Harvard Artifacts ETL & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

An end-to-end data engineering and analytics application for the Harvard Art Museums API. This project demonstrates ETL pipelines, SQL analytics, and interactive visualization using Streamlit. It extracts artifact metadata, media, and color data, transforms it into relational tables, and provides 20+ analytical queries with visual dashboards.

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
export DB_NAME="your_database_name"
```

**Required packages:**
- `streamlit`
- `pandas`
- `requests`
- `mysql-connector-python` or `pymysql`
- `plotly`

## Getting Harvard Art Museums API Key

1. Visit [Harvard Art Museums API Portal](https://www.harvardartmuseums.org/collections/api)
2. Register for a free API key
3. Store it securely as an environment variable

## Core Architecture

**Data Flow:** `API → Extract → Transform → Load → SQL → Analytics → Visualization`

### Database Schema

```python
# artifactmetadata table
"""
id (INT, PRIMARY KEY)
title (VARCHAR)
culture (VARCHAR)
dated (VARCHAR)
century (VARCHAR)
classification (VARCHAR)
department (VARCHAR)
technique (TEXT)
medium (TEXT)
dimensions (TEXT)
creditline (TEXT)
"""

# artifactmedia table
"""
id (INT, AUTO_INCREMENT, PRIMARY KEY)
artifact_id (INT, FOREIGN KEY)
media_url (TEXT)
media_type (VARCHAR)
"""

# artifactcolors table
"""
id (INT, AUTO_INCREMENT, PRIMARY KEY)
artifact_id (INT, FOREIGN KEY)
color (VARCHAR)
spectrum (VARCHAR)
percent (FLOAT)
"""
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts with pagination support"""
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

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts, info = fetch_artifacts(api_key, page=1, size=50)
print(f"Total records: {info['totalrecords']}")
print(f"Pages: {info['pages']}")
```

### Transform: Process JSON to DataFrames

```python
import pandas as pd

def transform_artifacts(records):
    """Transform API response into structured DataFrames"""
    
    # Metadata DataFrame
    metadata_list = []
    media_list = []
    colors_list = []
    
    for record in records:
        # Extract metadata
        metadata = {
            'id': record.get('id'),
            'title': record.get('title', 'Unknown'),
            'culture': record.get('culture', 'Unknown'),
            'dated': record.get('dated', 'Unknown'),
            'century': record.get('century', 'Unknown'),
            'classification': record.get('classification', 'Unknown'),
            'department': record.get('department', 'Unknown'),
            'technique': record.get('technique', ''),
            'medium': record.get('medium', ''),
            'dimensions': record.get('dimensions', ''),
            'creditline': record.get('creditline', '')
        }
        metadata_list.append(metadata)
        
        # Extract media
        images = record.get('images', [])
        for img in images:
            media_list.append({
                'artifact_id': record.get('id'),
                'media_url': img.get('baseimageurl', ''),
                'media_type': 'image'
            })
        
        # Extract colors
        colors = record.get('colors', [])
        for color in colors:
            colors_list.append({
                'artifact_id': record.get('id'),
                'color': color.get('color', ''),
                'spectrum': color.get('spectrum', ''),
                'percent': color.get('percent', 0.0)
            })
    
    df_metadata = pd.DataFrame(metadata_list)
    df_media = pd.DataFrame(media_list)
    df_colors = pd.DataFrame(colors_list)
    
    return df_metadata, df_media, df_colors
```

### Load: Insert into SQL Database

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

def create_tables(conn):
    """Create database schema"""
    cursor = conn.cursor()
    
    # Metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(255),
            dated VARCHAR(255),
            century VARCHAR(255),
            classification VARCHAR(255),
            department VARCHAR(255),
            technique TEXT,
            medium TEXT,
            dimensions TEXT,
            creditline TEXT
        )
    """)
    
    # Media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            media_url TEXT,
            media_type VARCHAR(50),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    # Colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(100),
            spectrum VARCHAR(100),
            percent FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    conn.commit()

def load_data(df_metadata, df_media, df_colors):
    """Batch insert DataFrames into database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    # Insert metadata
    for _, row in df_metadata.iterrows():
        cursor.execute("""
            INSERT IGNORE INTO artifactmetadata 
            (id, title, culture, dated, century, classification, 
             department, technique, medium, dimensions, creditline)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """, tuple(row))
    
    # Insert media
    for _, row in df_media.iterrows():
        cursor.execute("""
            INSERT INTO artifactmedia (artifact_id, media_url, media_type)
            VALUES (%s, %s, %s)
        """, tuple(row))
    
    # Insert colors
    for _, row in df_colors.iterrows():
        cursor.execute("""
            INSERT INTO artifactcolors (artifact_id, color, spectrum, percent)
            VALUES (%s, %s, %s, %s)
        """, tuple(row))
    
    conn.commit()
    conn.close()
```

## Analytical SQL Queries

### Key Analytics Patterns

```python
# Top 10 cultures by artifact count
query_cultures = """
SELECT culture, COUNT(*) as count
FROM artifactmetadata
WHERE culture != 'Unknown'
GROUP BY culture
ORDER BY count DESC
LIMIT 10
"""

# Artifacts by century
query_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC
"""

# Department distribution
query_departments = """
SELECT department, COUNT(*) as artifact_count
FROM artifactmetadata
GROUP BY department
ORDER BY artifact_count DESC
"""

# Color distribution across artifacts
query_colors = """
SELECT spectrum, COUNT(*) as frequency, AVG(percent) as avg_percent
FROM artifactcolors
GROUP BY spectrum
ORDER BY frequency DESC
LIMIT 10
"""

# Artifacts with most media
query_media = """
SELECT am.id, am.title, COUNT(media.id) as media_count
FROM artifactmetadata am
JOIN artifactmedia media ON am.id = media.artifact_id
GROUP BY am.id, am.title
ORDER BY media_count DESC
LIMIT 10
"""

# Classification by culture
query_classification = """
SELECT culture, classification, COUNT(*) as count
FROM artifactmetadata
WHERE culture != 'Unknown'
GROUP BY culture, classification
ORDER BY count DESC
LIMIT 15
"""

def execute_query(query):
    """Execute SQL query and return DataFrame"""
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

## Streamlit Dashboard

### Complete App Structure

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

def main():
    st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")
    
    # Sidebar for navigation
    page = st.sidebar.radio(
        "Select Module",
        ["ETL Pipeline", "SQL Analytics", "Visualizations"]
    )
    
    if page == "ETL Pipeline":
        show_etl_page()
    elif page == "SQL Analytics":
        show_analytics_page()
    else:
        show_visualization_page()

def show_etl_page():
    st.header("📥 ETL Pipeline")
    
    api_key = st.text_input("Harvard API Key", type="password")
    num_pages = st.number_input("Number of pages to fetch", 1, 10, 1)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching data..."):
            all_records = []
            for page in range(1, num_pages + 1):
                records, info = fetch_artifacts(api_key, page=page)
                all_records.extend(records)
            
            st.success(f"Fetched {len(all_records)} artifacts")
            
            # Transform
            df_meta, df_media, df_colors = transform_artifacts(all_records)
            st.write(f"Transformed into {len(df_meta)} metadata, "
                    f"{len(df_media)} media, {len(df_colors)} color records")
            
            # Load
            load_data(df_meta, df_media, df_colors)
            st.success("✅ Data loaded into database!")

def show_analytics_page():
    st.header("📊 SQL Analytics")
    
    queries = {
        "Top Cultures": query_cultures,
        "Century Distribution": query_century,
        "Department Stats": query_departments,
        "Color Analysis": query_colors,
        "Media Rich Artifacts": query_media
    }
    
    selected = st.selectbox("Select Query", list(queries.keys()))
    
    if st.button("Execute Query"):
        df = execute_query(queries[selected])
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                        title=selected)
            st.plotly_chart(fig, use_container_width=True)

def show_visualization_page():
    st.header("📈 Interactive Visualizations")
    
    # Example: Culture distribution pie chart
    df = execute_query(query_cultures)
    fig = px.pie(df, names='culture', values='count', 
                 title='Artifact Distribution by Culture')
    st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Configuration Best Practices

### Environment Variables (.env)

```bash
# API Configuration
HARVARD_API_KEY=your_api_key_here

# Database Configuration (MySQL/TiDB)
DB_HOST=gateway01.us-west-2.prod.aws.tidbcloud.com
DB_PORT=4000
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=artifacts_db

# App Configuration
STREAMLIT_SERVER_PORT=8501
STREAMLIT_SERVER_ADDRESS=localhost
```

### Load Environment Variables

```python
from dotenv import load_dotenv
import os

load_dotenv()

# Access configuration
API_KEY = os.getenv('HARVARD_API_KEY')
DB_CONFIG = {
    'host': os.getenv('DB_HOST'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}
```

## Common Patterns & Use Cases

### Incremental Data Loading

```python
def get_last_artifact_id(conn):
    """Get the highest artifact ID already in database"""
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def incremental_load(api_key):
    """Load only new artifacts since last update"""
    conn = get_db_connection()
    last_id = get_last_artifact_id(conn)
    conn.close()
    
    # Fetch artifacts with ID > last_id
    params = {
        'apikey': api_key,
        'q': f'id:>{last_id}',
        'size': 100
    }
    # Process and load...
```

### Rate Limiting & Error Handling

```python
import time
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def create_session_with_retry():
    """Create requests session with retry logic"""
    session = requests.Session()
    retry = Retry(
        total=5,
        backoff_factor=1,
        status_forcelist=[429, 500, 502, 503, 504]
    )
    adapter = HTTPAdapter(max_retries=retry)
    session.mount('https://', adapter)
    return session

def fetch_with_rate_limit(api_key, page, delay=0.5):
    """Fetch with rate limiting"""
    session = create_session_with_retry()
    time.sleep(delay)  # Respect API rate limits
    
    try:
        response = session.get(
            "https://api.harvardartmuseums.org/object",
            params={'apikey': api_key, 'page': page}
        )
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestException as e:
        st.error(f"API Error: {e}")
        return None
```

## Troubleshooting

### API Issues

**Problem:** 401 Unauthorized
```python
# Verify API key is set correctly
api_key = os.getenv('HARVARD_API_KEY')
if not api_key:
    raise ValueError("HARVARD_API_KEY not set in environment")
```

**Problem:** Rate limiting (429 Too Many Requests)
```python
# Add exponential backoff
import time
for attempt in range(3):
    response = requests.get(url, params=params)
    if response.status_code == 429:
        wait_time = 2 ** attempt
        time.sleep(wait_time)
    else:
        break
```

### Database Connection Issues

**Problem:** Connection timeout
```python
# Increase timeout and add connection pooling
import mysql.connector.pooling

dbconfig = {
    "pool_name": "mypool",
    "pool_size": 5,
    "host": os.getenv('DB_HOST'),
    "user": os.getenv('DB_USER'),
    "password": os.getenv('DB_PASSWORD'),
    "database": os.getenv('DB_NAME'),
    "connect_timeout": 30
}

connection_pool = mysql.connector.pooling.MySQLConnectionPool(**dbconfig)
```

### Data Quality Issues

**Problem:** NULL/missing values
```python
# Handle missing data during transform
df_metadata['culture'] = df_metadata['culture'].fillna('Unknown')
df_metadata['century'] = df_metadata['century'].fillna('Unknown')
df_colors['percent'] = pd.to_numeric(df_colors['percent'], errors='coerce').fillna(0.0)
```

**Problem:** Duplicate records
```python
# Use INSERT IGNORE or ON DUPLICATE KEY UPDATE
cursor.execute("""
    INSERT INTO artifactmetadata (id, title, culture, ...) 
    VALUES (%s, %s, %s, ...)
    ON DUPLICATE KEY UPDATE 
    title=VALUES(title), culture=VALUES(culture)
""", data)
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# With custom port
streamlit run app.py --server.port 8080

# Access at http://localhost:8501
```
