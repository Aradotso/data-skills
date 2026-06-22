---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with SQL and Streamlit
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - create analytics dashboard for museum artifact data
  - extract and transform Harvard museum collection data
  - build data engineering project with art museum API
  - query Harvard artifacts database with SQL analytics
  - visualize museum collection data with Streamlit
  - set up artifact metadata ETL workflow
  - analyze Harvard art museum data with Python
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

Harvard-Artifacts-Collection-Data-Engineering-Analytics-App is an end-to-end data engineering and analytics platform that demonstrates real-world ETL pipelines using the Harvard Art Museums API. It extracts artifact metadata, transforms nested JSON into relational tables, loads data into SQL databases (MySQL/TiDB), and provides interactive analytics dashboards via Streamlit with Plotly visualizations.

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Dependencies typically include:**
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### 1. Harvard API Key Setup

Get your free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

Create a `.env` file or configure via Streamlit secrets:

```python
# .env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### 2. Database Configuration

Set up MySQL or TiDB Cloud database with connection parameters:

```python
import os
from dotenv import load_dotenv

load_dotenv()

DB_CONFIG = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}
```

## Database Schema

The project uses three main tables with relationships:

```sql
-- Artifact Metadata (main table)
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dated VARCHAR(255),
    url TEXT
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url TEXT,
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### Extract: Fetch Data from Harvard API

```python
import requests
import os

def extract_artifacts(api_key, num_records=100):
    """Extract artifact data from Harvard Art Museums API"""
    base_url = "https://api.harvardartmuseums.org/object"
    artifacts = []
    page = 1
    size = 100  # Max per page
    
    while len(artifacts) < num_records:
        params = {
            'apikey': api_key,
            'size': min(size, num_records - len(artifacts)),
            'page': page
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            artifacts.extend(data.get('records', []))
            
            if len(data.get('records', [])) == 0:
                break
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_records]
```

### Transform: Process Nested JSON

```python
import pandas as pd

def transform_artifacts(raw_data):
    """Transform raw API data into structured dataframes"""
    
    # Metadata transformation
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'technique': artifact.get('technique', 'Unknown'),
            'medium': artifact.get('medium', 'Unknown'),
            'dated': artifact.get('dated', 'Unknown'),
            'url': artifact.get('url', '')
        })
        
        # Extract media
        for media in artifact.get('images', []):
            media_records.append({
                'artifact_id': artifact.get('id'),
                'image_url': media.get('baseimageurl', ''),
                'media_type': 'image'
            })
        
        # Extract colors
        for color in artifact.get('colors', []):
            color_records.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color', ''),
                'percentage': color.get('percent', 0)
            })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### Load: Insert into SQL Database

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(df_metadata, df_media, df_colors, db_config):
    """Load transformed data into SQL database"""
    try:
        connection = mysql.connector.connect(**db_config)
        cursor = connection.cursor()
        
        # Load metadata
        metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, technique, medium, dated, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_query, df_metadata.values.tolist())
        
        # Load media
        media_query = """
        INSERT INTO artifactmedia (artifact_id, image_url, media_type)
        VALUES (%s, %s, %s)
        """
        cursor.executemany(media_query, df_media.values.tolist())
        
        # Load colors
        color_query = """
        INSERT INTO artifactcolors (artifact_id, color, percentage)
        VALUES (%s, %s, %s)
        """
        cursor.executemany(color_query, df_colors.values.tolist())
        
        connection.commit()
        print(f"Loaded {len(df_metadata)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

## Streamlit Dashboard Implementation

### Main Application Structure

```python
import streamlit as st
import pandas as pd
import plotly.express as px
from etl_functions import extract_artifacts, transform_artifacts, load_to_database
from sql_queries import execute_query, ANALYTICS_QUERIES

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

# Sidebar for configuration
st.sidebar.title("Configuration")
api_key = st.sidebar.text_input("Harvard API Key", type="password", 
                                 value=os.getenv('HARVARD_API_KEY', ''))

# Main tabs
tab1, tab2, tab3 = st.tabs(["ETL Pipeline", "SQL Analytics", "Visualizations"])

with tab1:
    st.header("Extract, Transform, Load")
    
    num_records = st.number_input("Number of records to extract", 
                                   min_value=10, max_value=1000, value=100)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Extracting data from API..."):
            raw_data = extract_artifacts(api_key, num_records)
            st.success(f"Extracted {len(raw_data)} records")
        
        with st.spinner("Transforming data..."):
            df_meta, df_media, df_colors = transform_artifacts(raw_data)
            st.success("Transformation complete")
            
            col1, col2, col3 = st.columns(3)
            col1.metric("Metadata Records", len(df_meta))
            col2.metric("Media Records", len(df_media))
            col3.metric("Color Records", len(df_colors))
        
        with st.spinner("Loading to database..."):
            load_to_database(df_meta, df_media, df_colors, DB_CONFIG)
            st.success("Data loaded successfully!")

with tab2:
    st.header("SQL Analytics Dashboard")
    
    query_name = st.selectbox("Select Analysis Query", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Execute Query"):
        query = ANALYTICS_QUERIES[query_name]
        st.code(query, language='sql')
        
        result_df = execute_query(query, DB_CONFIG)
        st.dataframe(result_df)
        
        # Auto-generate visualization
        if len(result_df) > 0 and len(result_df.columns) >= 2:
            fig = px.bar(result_df, x=result_df.columns[0], y=result_df.columns[1],
                        title=query_name)
            st.plotly_chart(fig, use_container_width=True)
```

### Sample Analytics Queries

```python
ANALYTICS_QUERIES = {
    "Top 10 Cultures by Artifact Count": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century Distribution": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != 'Unknown'
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Department-wise Classification": """
        SELECT department, classification, COUNT(*) as count
        FROM artifactmetadata
        GROUP BY department, classification
        ORDER BY department, count DESC
    """,
    
    "Media Availability Analysis": """
        SELECT 
            CASE WHEN media_count > 0 THEN 'With Media' ELSE 'No Media' END as media_status,
            COUNT(*) as artifact_count
        FROM (
            SELECT m.id, COUNT(img.media_id) as media_count
            FROM artifactmetadata m
            LEFT JOIN artifactmedia img ON m.id = img.artifact_id
            GROUP BY m.id
        ) as media_stats
        GROUP BY media_status
    """,
    
    "Top 15 Colors Across Artifacts": """
        SELECT color, COUNT(DISTINCT artifact_id) as artifact_count,
               AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY artifact_count DESC
        LIMIT 15
    """
}

def execute_query(query, db_config):
    """Execute SQL query and return results as DataFrame"""
    connection = mysql.connector.connect(**db_config)
    df = pd.read_sql(query, connection)
    connection.close()
    return df
```

## Running the Application

```bash
# Set environment variables
export HARVARD_API_KEY=your_api_key
export DB_HOST=your_host
export DB_USER=your_user
export DB_PASSWORD=your_password
export DB_NAME=harvard_artifacts

# Run Streamlit app
streamlit run app.py
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def extract_with_rate_limit(api_key, num_records, delay=0.5):
    """Extract with rate limiting to avoid API throttling"""
    artifacts = []
    
    for page in range(1, (num_records // 100) + 2):
        # Make request
        response = requests.get(url, params=params)
        artifacts.extend(response.json().get('records', []))
        
        # Rate limit
        time.sleep(delay)
        
    return artifacts
```

### Batch Insert Optimization

```python
def batch_insert(data, query, db_config, batch_size=1000):
    """Insert data in batches for better performance"""
    connection = mysql.connector.connect(**db_config)
    cursor = connection.cursor()
    
    for i in range(0, len(data), batch_size):
        batch = data[i:i+batch_size]
        cursor.executemany(query, batch)
        connection.commit()
    
    cursor.close()
    connection.close()
```

## Troubleshooting

**API Key Issues:**
```python
# Verify API key is loaded
if not os.getenv('HARVARD_API_KEY'):
    raise ValueError("HARVARD_API_KEY not set in environment")
```

**Database Connection Errors:**
```python
# Test connection
try:
    connection = mysql.connector.connect(**DB_CONFIG)
    print("Database connection successful")
except Error as e:
    print(f"Connection failed: {e}")
```

**Empty Results:**
- Check API response status codes
- Verify pagination logic
- Ensure database tables exist before loading
- Check for NULL handling in SQL queries

**Performance Issues:**
- Use batch inserts for large datasets
- Add database indexes on frequently queried columns
- Implement connection pooling for multiple queries
