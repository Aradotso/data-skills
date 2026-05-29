---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Streamlit, MySQL, and Plotly
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - set up analytics dashboard for museum artifact data
  - create data engineering workflow with Streamlit and MySQL
  - extract and analyze Harvard art collection data
  - build artifact data visualization with Plotly
  - implement museum data pipeline with Python
  - query and visualize Harvard museum artifacts
  - create end-to-end data engineering app with Streamlit
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates building end-to-end data engineering and analytics applications using the Harvard Art Museums API. It implements ETL pipelines to extract artifact data, transform it into relational structures, load it into MySQL/TiDB Cloud, and visualize insights through interactive Streamlit dashboards.

## What It Does

- **Extracts** artifact metadata, media, and color data from Harvard Art Museums API
- **Transforms** nested JSON into normalized relational tables
- **Loads** data into MySQL with proper foreign key relationships
- **Analyzes** data using predefined SQL queries
- **Visualizes** results with interactive Plotly charts in Streamlit

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required dependencies:**
```
streamlit
pandas
requests
mysql-connector-python
plotly
```

## Configuration

### API Key Setup

Get a free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api):

```python
# Store in environment variable
export HARVARD_API_KEY="your_api_key_here"
```

### Database Configuration

Configure MySQL/TiDB connection:

```python
import mysql.connector
import os

db_config = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}

conn = mysql.connector.connect(**db_config)
```

## Database Schema

The project uses three main tables:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    period VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accessionyear INT
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    imagepermissionlevel INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API
    
    Args:
        api_key: Harvard API key
        page: Page number for pagination
        size: Number of records per page
    
    Returns:
        dict: JSON response with artifact data
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    
    return response.json()

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=100)

print(f"Total records: {data['info']['totalrecords']}")
print(f"Pages: {data['info']['pages']}")
```

### Transform: Process JSON to DataFrames

```python
import pandas as pd

def transform_artifacts(api_response):
    """
    Transform API response into normalized DataFrames
    
    Returns:
        tuple: (metadata_df, media_df, colors_df)
    """
    records = api_response.get('records', [])
    
    # Extract metadata
    metadata_list = []
    media_list = []
    colors_list = []
    
    for record in records:
        # Metadata
        metadata_list.append({
            'id': record.get('id'),
            'title': record.get('title'),
            'culture': record.get('culture'),
            'century': record.get('century'),
            'classification': record.get('classification'),
            'department': record.get('department'),
            'dated': record.get('dated'),
            'period': record.get('period'),
            'technique': record.get('technique'),
            'medium': record.get('medium'),
            'dimensions': record.get('dimensions'),
            'creditline': record.get('creditline'),
            'accessionyear': record.get('accessionyear')
        })
        
        # Media
        if 'primaryimageurl' in record:
            media_list.append({
                'artifact_id': record.get('id'),
                'baseimageurl': record.get('baseimageurl'),
                'primaryimageurl': record.get('primaryimageurl'),
                'imagepermissionlevel': record.get('imagepermissionlevel', 0)
            })
        
        # Colors
        colors = record.get('colors', [])
        for color in colors:
            colors_list.append({
                'artifact_id': record.get('id'),
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

### Load: Insert into MySQL

```python
def load_to_mysql(metadata_df, media_df, colors_df, db_config):
    """
    Load transformed data into MySQL database
    
    Args:
        metadata_df: DataFrame with artifact metadata
        media_df: DataFrame with media information
        colors_df: DataFrame with color data
        db_config: Database connection configuration
    """
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    
    # Insert metadata (batch insert for performance)
    metadata_sql = """
        INSERT IGNORE INTO artifactmetadata 
        (id, title, culture, century, classification, department, 
         dated, period, technique, medium, dimensions, creditline, accessionyear)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    """
    
    metadata_values = [tuple(row) for row in metadata_df.values]
    cursor.executemany(metadata_sql, metadata_values)
    
    # Insert media
    media_sql = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, primaryimageurl, imagepermissionlevel)
        VALUES (%s, %s, %s, %s)
    """
    
    media_values = [tuple(row) for row in media_df.values]
    cursor.executemany(media_sql, media_values)
    
    # Insert colors
    colors_sql = """
        INSERT INTO artifactcolors 
        (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
    """
    
    colors_values = [tuple(row) for row in colors_df.values]
    cursor.executemany(colors_sql, colors_values)
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print(f"Loaded {len(metadata_df)} artifacts successfully")
```

## Analytics Queries

### Common SQL Patterns

```python
# Top 10 cultures by artifact count
query_cultures = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Artifacts by century
query_centuries = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY count DESC
"""

# Color distribution
query_colors = """
    SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
    FROM artifactcolors
    GROUP BY color
    ORDER BY usage_count DESC
    LIMIT 15
"""

# Department-wise classification
query_dept_class = """
    SELECT department, classification, COUNT(*) as count
    FROM artifactmetadata
    WHERE department IS NOT NULL AND classification IS NOT NULL
    GROUP BY department, classification
    ORDER BY department, count DESC
"""

# Artifacts with images vs without
query_media_stats = """
    SELECT 
        COUNT(DISTINCT am.id) as total_artifacts,
        COUNT(DISTINCT media.artifact_id) as with_images,
        COUNT(DISTINCT am.id) - COUNT(DISTINCT media.artifact_id) as without_images
    FROM artifactmetadata am
    LEFT JOIN artifactmedia media ON am.id = media.artifact_id
"""
```

## Streamlit Dashboard

### Basic App Structure

```python
import streamlit as st
import pandas as pd
import plotly.express as px
import mysql.connector

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Database connection
@st.cache_resource
def get_db_connection():
    return mysql.connector.connect(**db_config)

# Execute query
def run_query(query):
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    return df

# Sidebar for query selection
st.sidebar.header("Analytics Queries")

queries = {
    "Top Cultures": query_cultures,
    "Artifacts by Century": query_centuries,
    "Color Distribution": query_colors,
    "Department Classification": query_dept_class,
    "Media Statistics": query_media_stats
}

selected_query = st.sidebar.selectbox("Select Analysis", list(queries.keys()))

# Execute and display
if st.button("Run Analysis"):
    with st.spinner("Executing query..."):
        result_df = run_query(queries[selected_query])
        
        # Display table
        st.subheader("Query Results")
        st.dataframe(result_df)
        
        # Auto-generate visualization
        if len(result_df.columns) >= 2:
            st.subheader("Visualization")
            
            fig = px.bar(
                result_df.head(10),
                x=result_df.columns[0],
                y=result_df.columns[1],
                title=f"{selected_query} - Top 10"
            )
            
            st.plotly_chart(fig, use_container_width=True)
```

### ETL Pipeline in Streamlit

```python
st.sidebar.header("ETL Pipeline")

if st.sidebar.button("Run ETL"):
    api_key = os.getenv('HARVARD_API_KEY')
    
    progress_bar = st.progress(0)
    status_text = st.empty()
    
    # Extract
    status_text.text("Extracting data from API...")
    data = fetch_artifacts(api_key, page=1, size=100)
    progress_bar.progress(33)
    
    # Transform
    status_text.text("Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts(data)
    progress_bar.progress(66)
    
    # Load
    status_text.text("Loading to database...")
    load_to_mysql(metadata_df, media_df, colors_df, db_config)
    progress_bar.progress(100)
    
    status_text.text("ETL completed successfully!")
    st.success(f"Processed {len(metadata_df)} artifacts")
```

## Common Patterns

### Pagination Handling

```python
def fetch_all_artifacts(api_key, max_pages=10):
    """Fetch multiple pages of artifacts"""
    all_records = []
    
    for page in range(1, max_pages + 1):
        data = fetch_artifacts(api_key, page=page, size=100)
        all_records.extend(data.get('records', []))
        
        if page >= data['info']['pages']:
            break
    
    return all_records
```

### Error Handling

```python
import time

def fetch_with_retry(api_key, page, retries=3):
    """Fetch with retry logic for rate limiting"""
    for attempt in range(retries):
        try:
            return fetch_artifacts(api_key, page=page)
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:  # Rate limit
                wait_time = 2 ** attempt
                time.sleep(wait_time)
            else:
                raise
    
    raise Exception("Max retries exceeded")
```

## Troubleshooting

**API Key Issues:**
- Ensure `HARVARD_API_KEY` environment variable is set
- Verify key is valid at https://www.harvardartmuseums.org/collections/api

**Database Connection:**
```python
# Test connection
try:
    conn = mysql.connector.connect(**db_config)
    print("Connection successful")
    conn.close()
except mysql.connector.Error as err:
    print(f"Error: {err}")
```

**Rate Limiting:**
- Harvard API has rate limits; implement delays between requests
- Use `time.sleep(1)` between pagination calls

**Missing Data:**
- Not all artifacts have all fields; use `.get()` with defaults
- Check for NULL values in SQL queries with `IS NOT NULL`

**Streamlit Caching:**
```python
# Clear cache if data seems stale
st.cache_resource.clear()
```
