---
name: harvard-art-museums-etl-analytics
description: End-to-end ETL pipeline and analytics app using Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for museum data
  - fetch and analyze Harvard Art Museums artifacts
  - create a data engineering pipeline with API integration
  - set up Streamlit analytics dashboard for art collections
  - extract transform load museum artifact data
  - analyze art museum collections with SQL queries
  - visualize museum artifact data with interactive dashboards
  - implement batch ETL with pandas and MySQL
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

This is a complete data engineering and analytics application that demonstrates real-world ETL patterns using the Harvard Art Museums API. It extracts artifact data, transforms nested JSON into relational tables, loads into MySQL/TiDB, executes analytical SQL queries, and visualizes results through an interactive Streamlit dashboard.

**Architecture Flow**: API → ETL → SQL → Analytics → Visualization

**Key Components**:
- API integration with pagination and rate limiting
- ETL pipeline transforming nested JSON to relational schema
- SQL database with three core tables (metadata, media, colors)
- 20+ predefined analytical queries
- Interactive Plotly visualizations

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Dependencies** (typical requirements.txt):
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### 1. Harvard Art Museums API Key

Sign up at [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api) to get your API key.

```python
# Store in environment variable
export HARVARD_API_KEY="your_api_key_here"
```

### 2. Database Connection

Configure MySQL/TiDB Cloud connection:

```python
import os
import mysql.connector

# Database configuration from environment
db_config = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}

# Create connection
conn = mysql.connector.connect(**db_config)
cursor = conn.cursor()
```

### 3. Environment Variables

Create a `.env` file:

```bash
HARVARD_API_KEY=your_harvard_api_key
DB_HOST=your_db_host
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
DB_PORT=3306
```

## Database Schema

### Core Tables

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    artifact_id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    medium VARCHAR(500),
    department VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255),
    period VARCHAR(255),
    technique VARCHAR(500),
    INDEX idx_culture (culture),
    INDEX idx_century (century),
    INDEX idx_classification (classification)
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    color_percentage DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id),
    INDEX idx_color (color)
);
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import time

def fetch_artifacts(api_key, num_pages=10, size=100):
    """
    Fetch artifact data from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        try:
            response = requests.get(base_url, params=params, timeout=30)
            response.raise_for_status()
            data = response.json()
            
            if 'records' in data:
                all_artifacts.extend(data['records'])
                print(f"Fetched page {page}: {len(data['records'])} records")
            
            # Rate limiting: respect API limits
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return all_artifacts
```

### Transform: Process Nested JSON

```python
import pandas as pd

def transform_artifacts(artifacts):
    """
    Transform nested JSON into structured DataFrames
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        
        # Extract metadata
        metadata = {
            'artifact_id': artifact_id,
            'title': artifact.get('title', ''),
            'culture': artifact.get('culture', ''),
            'century': artifact.get('century', ''),
            'classification': artifact.get('classification', ''),
            'medium': artifact.get('medium', ''),
            'department': artifact.get('department', ''),
            'division': artifact.get('division', ''),
            'dated': artifact.get('dated', ''),
            'period': artifact.get('period', ''),
            'technique': artifact.get('technique', '')
        }
        metadata_list.append(metadata)
        
        # Extract media information
        media = {
            'artifact_id': artifact_id,
            'baseimageurl': artifact.get('baseimageurl', ''),
            'primaryimageurl': artifact.get('primaryimageurl', ''),
            'iiifbaseuri': artifact.get('images', [{}])[0].get('iiifbaseuri', '') if artifact.get('images') else ''
        }
        media_list.append(media)
        
        # Extract color data
        if artifact.get('colors'):
            for color_data in artifact['colors']:
                color = {
                    'artifact_id': artifact_id,
                    'color': color_data.get('color', ''),
                    'color_percentage': color_data.get('percent', 0)
                }
                colors_list.append(color)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load: Batch Insert into SQL

```python
def load_to_database(conn, metadata_df, media_df, colors_df):
    """
    Load transformed data into MySQL database using batch inserts
    """
    cursor = conn.cursor()
    
    # Insert metadata (batch)
    metadata_query = """
        INSERT INTO artifactmetadata 
        (artifact_id, title, culture, century, classification, medium, 
         department, division, dated, period, technique)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    metadata_values = metadata_df.values.tolist()
    cursor.executemany(metadata_query, metadata_values)
    
    # Insert media
    media_query = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, primaryimageurl, iiifbaseuri)
        VALUES (%s, %s, %s, %s)
    """
    media_values = media_df.values.tolist()
    cursor.executemany(media_query, media_values)
    
    # Insert colors
    if not colors_df.empty:
        colors_query = """
            INSERT INTO artifactcolors 
            (artifact_id, color, color_percentage)
            VALUES (%s, %s, %s)
        """
        colors_values = colors_df.values.tolist()
        cursor.executemany(colors_query, colors_values)
    
    conn.commit()
    print(f"Loaded {len(metadata_df)} artifacts successfully")
```

## Analytical SQL Queries

### Example Queries

```python
# Query 1: Artifact distribution by culture
query_by_culture = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL AND culture != ''
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 15
"""

# Query 2: Artifacts by century
query_by_century = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL AND century != ''
    GROUP BY century
    ORDER BY century
"""

# Query 3: Media availability analysis
query_media_availability = """
    SELECT 
        COUNT(CASE WHEN primaryimageurl != '' THEN 1 END) as with_primary_image,
        COUNT(CASE WHEN primaryimageurl = '' THEN 1 END) as without_primary_image,
        COUNT(*) as total_artifacts
    FROM artifactmedia
"""

# Query 4: Top colors across artifacts
query_top_colors = """
    SELECT color, COUNT(*) as usage_count, AVG(color_percentage) as avg_percentage
    FROM artifactcolors
    GROUP BY color
    ORDER BY usage_count DESC
    LIMIT 10
"""

# Query 5: Classification distribution
query_classification = """
    SELECT classification, COUNT(*) as count
    FROM artifactmetadata
    WHERE classification IS NOT NULL AND classification != ''
    GROUP BY classification
    ORDER BY count DESC
    LIMIT 10
"""

# Query 6: Department-wise artifacts
query_by_department = """
    SELECT department, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE department IS NOT NULL AND department != ''
    GROUP BY department
    ORDER BY artifact_count DESC
"""
```

## Streamlit Application

### Main Application Structure

```python
import streamlit as st
import pandas as pd
import plotly.express as px
from dotenv import load_dotenv
import os

load_dotenv()

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")

# Sidebar for navigation
page = st.sidebar.selectbox(
    "Navigation",
    ["Data Collection", "SQL Analytics", "Visualizations"]
)

if page == "Data Collection":
    st.header("📥 Data Collection & ETL Pipeline")
    
    api_key = st.text_input("Enter Harvard API Key", type="password")
    num_pages = st.slider("Number of pages to fetch", 1, 50, 10)
    
    if st.button("Start ETL Process"):
        with st.spinner("Fetching artifacts..."):
            artifacts = fetch_artifacts(api_key, num_pages)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            conn = mysql.connector.connect(**db_config)
            load_to_database(conn, metadata_df, media_df, colors_df)
            conn.close()
            st.success("ETL Complete!")

elif page == "SQL Analytics":
    st.header("📊 SQL Query Analytics")
    
    queries = {
        "Artifacts by Culture": query_by_culture,
        "Artifacts by Century": query_by_century,
        "Media Availability": query_media_availability,
        "Top Colors": query_top_colors,
        "Classification Distribution": query_classification,
        "Department Distribution": query_by_department
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        conn = mysql.connector.connect(**db_config)
        df = pd.read_sql(queries[selected_query], conn)
        conn.close()
        
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)

elif page == "Visualizations":
    st.header("📈 Interactive Visualizations")
    # Custom visualization logic
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at: http://localhost:8501
```

## Common Patterns

### Error Handling in ETL

```python
def safe_etl_pipeline(api_key, num_pages, db_config):
    """
    ETL with comprehensive error handling
    """
    try:
        # Extract
        artifacts = fetch_artifacts(api_key, num_pages)
        if not artifacts:
            raise ValueError("No artifacts fetched")
        
        # Transform
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        
        # Validate data
        assert not metadata_df.empty, "Metadata is empty"
        
        # Load
        conn = mysql.connector.connect(**db_config)
        try:
            load_to_database(conn, metadata_df, media_df, colors_df)
        finally:
            conn.close()
        
        return True, f"Successfully processed {len(artifacts)} artifacts"
        
    except Exception as e:
        return False, f"ETL Failed: {str(e)}"
```

### Incremental Data Loading

```python
def get_last_artifact_id(conn):
    """Get the last loaded artifact ID for incremental loads"""
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(artifact_id) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def incremental_load(api_key, conn, start_page=1):
    """Load only new artifacts"""
    last_id = get_last_artifact_id(conn)
    artifacts = fetch_artifacts(api_key, num_pages=5)
    
    # Filter new artifacts
    new_artifacts = [a for a in artifacts if a.get('id', 0) > last_id]
    
    if new_artifacts:
        metadata_df, media_df, colors_df = transform_artifacts(new_artifacts)
        load_to_database(conn, metadata_df, media_df, colors_df)
        return len(new_artifacts)
    return 0
```

## Troubleshooting

### API Rate Limiting
```python
# Add exponential backoff
import time
from requests.exceptions import HTTPError

def fetch_with_retry(url, params, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=30)
            response.raise_for_status()
            return response.json()
        except HTTPError as e:
            if e.response.status_code == 429:  # Rate limited
                wait_time = 2 ** attempt
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues
```python
# Connection pooling for Streamlit
from mysql.connector import pooling

@st.cache_resource
def get_connection_pool():
    return pooling.MySQLConnectionPool(
        pool_name="harvard_pool",
        pool_size=5,
        **db_config
    )

def execute_query(query):
    pool = get_connection_pool()
    conn = pool.get_connection()
    try:
        df = pd.read_sql(query, conn)
        return df
    finally:
        conn.close()
```

### Memory Management for Large Datasets
```python
def chunked_etl(api_key, total_pages, chunk_size=5):
    """Process data in chunks to manage memory"""
    for start_page in range(1, total_pages + 1, chunk_size):
        end_page = min(start_page + chunk_size, total_pages + 1)
        artifacts = fetch_artifacts(api_key, num_pages=end_page - start_page)
        
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        
        conn = mysql.connector.connect(**db_config)
        load_to_database(conn, metadata_df, media_df, colors_df)
        conn.close()
        
        # Clear memory
        del artifacts, metadata_df, media_df, colors_df
```
