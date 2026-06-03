---
name: harvard-artifacts-data-engineering-app
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - how do I build a data pipeline with Harvard Art Museums API
  - set up ETL for museum artifact data
  - create analytics dashboard for Harvard artifacts
  - implement museum data collection and visualization
  - build data engineering app with Streamlit and SQL
  - analyze Harvard Art Museums collection data
  - create artifact metadata ETL pipeline
  - visualize museum data with Plotly and SQL
---

# Harvard Artifacts Data Engineering App

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is an end-to-end data engineering and analytics application that demonstrates real-world ETL pipelines, SQL analytics, and interactive visualization using the Harvard Art Museums API. It extracts artifact metadata, transforms nested JSON into relational tables, loads data into SQL databases, and provides interactive Streamlit dashboards with 20+ analytical queries.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

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
export DB_NAME="harvard_artifacts"
```

**Required packages:**
- streamlit
- pandas
- requests
- mysql-connector-python (or tidb connector)
- plotly
- python-dotenv

## Configuration

### API Setup

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

```python
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"
```

### Database Configuration

```python
import mysql.connector

db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

connection = mysql.connector.connect(**db_config)
```

## Database Schema

The project uses three main tables with proper foreign key relationships:

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    dated VARCHAR(200),
    period VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    primaryimageurl TEXT,
    url TEXT,
    creditline TEXT,
    accession_number VARCHAR(100),
    lastupdate DATETIME
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl TEXT,
    caption TEXT,
    copyright TEXT,
    format VARCHAR(50),
    height INT,
    width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline

### Extract: Fetch Data from API

```python
import requests
import time

def fetch_artifacts(api_key, num_pages=10, page_size=100):
    """
    Extract artifact data from Harvard Art Museums API with pagination
    """
    artifacts = []
    base_url = "https://api.harvardartmuseums.org/object"
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page,
            'hasimage': 1  # Only get artifacts with images
        }
        
        try:
            response = requests.get(base_url, params=params)
            response.raise_for_status()
            data = response.json()
            
            artifacts.extend(data.get('records', []))
            print(f"Fetched page {page}: {len(data.get('records', []))} artifacts")
            
            # Rate limiting
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return artifacts
```

### Transform: Process JSON to DataFrames

```python
import pandas as pd

def transform_artifacts(raw_data):
    """
    Transform nested JSON into relational dataframes
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'period': artifact.get('period'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'primaryimageurl': artifact.get('primaryimageurl'),
            'url': artifact.get('url'),
            'creditline': artifact.get('creditline'),
            'accession_number': artifact.get('accessionnumber'),
            'lastupdate': artifact.get('lastupdate')
        }
        metadata_list.append(metadata)
        
        # Extract media (nested)
        for media in artifact.get('images', []):
            media_list.append({
                'artifact_id': artifact.get('id'),
                'baseimageurl': media.get('baseimageurl'),
                'caption': media.get('caption'),
                'copyright': media.get('copyright'),
                'format': media.get('format'),
                'height': media.get('height'),
                'width': media.get('width')
            })
        
        # Extract colors (nested)
        for color in artifact.get('colors', []):
            colors_list.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load: Batch Insert to SQL

```python
def load_to_database(df_metadata, df_media, df_colors, connection):
    """
    Load transformed data into SQL database with batch inserts
    """
    cursor = connection.cursor()
    
    # Load metadata
    metadata_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, classification, department, division, 
     dated, period, technique, medium, dimensions, primaryimageurl, 
     url, creditline, accession_number, lastupdate)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    metadata_values = df_metadata.values.tolist()
    cursor.executemany(metadata_query, metadata_values)
    
    # Load media
    if not df_media.empty:
        media_query = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, caption, copyright, format, height, width)
        VALUES (%s, %s, %s, %s, %s, %s, %s)
        """
        cursor.executemany(media_query, df_media.values.tolist())
    
    # Load colors
    if not df_colors.empty:
        colors_query = """
        INSERT INTO artifactcolors 
        (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
        """
        cursor.executemany(colors_query, df_colors.values.tolist())
    
    connection.commit()
    print(f"Loaded {len(df_metadata)} artifacts successfully")
```

## Analytics Queries

### Sample Analytical Queries

```python
# Query 1: Top 10 cultures by artifact count
query_cultures = """
SELECT culture, COUNT(*) as count 
FROM artifactmetadata 
WHERE culture IS NOT NULL 
GROUP BY culture 
ORDER BY count DESC 
LIMIT 10
"""

# Query 2: Artifacts by century
query_century = """
SELECT century, COUNT(*) as count 
FROM artifactmetadata 
WHERE century IS NOT NULL 
GROUP BY century 
ORDER BY count DESC
"""

# Query 3: Top colors in collection
query_colors = """
SELECT c.color, COUNT(*) as usage_count, AVG(c.percent) as avg_percent
FROM artifactcolors c
GROUP BY c.color
ORDER BY usage_count DESC
LIMIT 10
"""

# Query 4: Media availability by department
query_media = """
SELECT m.department, COUNT(DISTINCT me.media_id) as media_count
FROM artifactmetadata m
LEFT JOIN artifactmedia me ON m.id = me.artifact_id
GROUP BY m.department
ORDER BY media_count DESC
"""

# Query 5: Artifacts without images
query_no_images = """
SELECT department, COUNT(*) as count
FROM artifactmetadata
WHERE primaryimageurl IS NULL OR primaryimageurl = ''
GROUP BY department
ORDER BY count DESC
"""
```

## Streamlit Application

### Main App Structure

```python
import streamlit as st
import pandas as pd
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar for configuration
    with st.sidebar:
        st.header("Configuration")
        api_key = st.text_input("Harvard API Key", type="password", 
                                value=os.getenv('HARVARD_API_KEY', ''))
        
        st.header("Data Collection")
        num_pages = st.slider("Number of pages to fetch", 1, 50, 10)
        
        if st.button("Run ETL Pipeline"):
            with st.spinner("Fetching data from API..."):
                raw_data = fetch_artifacts(api_key, num_pages)
                
            with st.spinner("Transforming data..."):
                df_meta, df_media, df_colors = transform_artifacts(raw_data)
                
            with st.spinner("Loading to database..."):
                connection = mysql.connector.connect(**db_config)
                load_to_database(df_meta, df_media, df_colors, connection)
                connection.close()
                
            st.success(f"✅ Loaded {len(df_meta)} artifacts successfully!")
    
    # Analytics Dashboard
    st.header("📊 Analytics Dashboard")
    
    connection = mysql.connector.connect(**db_config)
    
    # Query selector
    query_options = {
        "Top Cultures": query_cultures,
        "Artifacts by Century": query_century,
        "Popular Colors": query_colors,
        "Media by Department": query_media,
        "Artifacts Without Images": query_no_images
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
    
    # Execute and visualize
    df_result = pd.read_sql(query_options[selected_query], connection)
    
    col1, col2 = st.columns([1, 2])
    
    with col1:
        st.dataframe(df_result, use_container_width=True)
    
    with col2:
        # Auto-generate visualization
        if len(df_result.columns) >= 2:
            fig = px.bar(df_result, 
                        x=df_result.columns[0], 
                        y=df_result.columns[1],
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)
    
    connection.close()

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Run Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Pattern 1: Incremental Data Loading

```python
def incremental_load(api_key, connection, last_update_date):
    """
    Load only artifacts updated after a certain date
    """
    params = {
        'apikey': api_key,
        'updatedafter': last_update_date,
        'size': 100
    }
    
    response = requests.get(BASE_URL, params=params)
    new_artifacts = response.json().get('records', [])
    
    df_meta, df_media, df_colors = transform_artifacts(new_artifacts)
    load_to_database(df_meta, df_media, df_colors, connection)
```

### Pattern 2: Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_etl_pipeline(api_key, num_pages, connection):
    """
    ETL pipeline with comprehensive error handling
    """
    try:
        logger.info("Starting ETL pipeline...")
        raw_data = fetch_artifacts(api_key, num_pages)
        
        if not raw_data:
            logger.warning("No data fetched from API")
            return
        
        df_meta, df_media, df_colors = transform_artifacts(raw_data)
        load_to_database(df_meta, df_media, df_colors, connection)
        
        logger.info(f"Successfully processed {len(df_meta)} artifacts")
        
    except Exception as e:
        logger.error(f"ETL pipeline failed: {e}")
        raise
```

## Troubleshooting

### Issue: API Rate Limiting

**Solution:** Add delays between requests and implement exponential backoff:

```python
import time
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

session = requests.Session()
retry = Retry(total=3, backoff_factor=1)
adapter = HTTPAdapter(max_retries=retry)
session.mount('https://', adapter)

time.sleep(0.5)  # Add delay between requests
```

### Issue: Database Connection Timeout

**Solution:** Use connection pooling:

```python
from mysql.connector import pooling

connection_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    **db_config
)

connection = connection_pool.get_connection()
```

### Issue: Large Data Memory Issues

**Solution:** Process in smaller batches:

```python
def batch_load(df, connection, batch_size=1000):
    for i in range(0, len(df), batch_size):
        batch = df.iloc[i:i+batch_size]
        load_to_database(batch, connection)
        print(f"Loaded batch {i//batch_size + 1}")
```

### Issue: Missing or Null Values

**Solution:** Handle nulls during transformation:

```python
df_metadata = df_metadata.fillna({
    'culture': 'Unknown',
    'century': 'Unknown',
    'primaryimageurl': ''
})

# Or drop rows with critical nulls
df_metadata = df_metadata.dropna(subset=['id', 'title'])
```

## Best Practices

1. **Always use environment variables** for sensitive data (API keys, passwords)
2. **Implement pagination** for large API responses
3. **Use batch inserts** for better database performance
4. **Add indexes** on frequently queried columns (culture, century, department)
5. **Cache query results** in Streamlit using `@st.cache_data`
6. **Validate data** before loading to database
7. **Log all operations** for debugging and monitoring
