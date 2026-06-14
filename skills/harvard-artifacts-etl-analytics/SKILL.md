---
name: harvard-artifacts-etl-analytics
description: Build end-to-end data pipelines with Harvard Art Museums API, including ETL, SQL storage, and Streamlit analytics dashboards
triggers:
  - how do I build an ETL pipeline for the Harvard Art Museums API
  - show me how to extract and load museum artifact data
  - help me create analytics dashboards with artifact collection data
  - how do I set up SQL database for Harvard museum artifacts
  - build a Streamlit app for art museum data visualization
  - create data engineering pipeline with Harvard API
  - analyze Harvard Art Museums collection data with SQL
  - process and visualize museum artifact metadata
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides a complete data engineering solution for the Harvard Art Museums API, demonstrating ETL pipelines, SQL analytics, and interactive Streamlit visualizations. It extracts artifact metadata, media details, and color information, stores them in relational databases, and enables complex analytical queries with visual outputs.

## What It Does

- **API Integration**: Fetches paginated data from Harvard Art Museums API with rate limiting
- **ETL Pipeline**: Transforms nested JSON into normalized relational tables
- **SQL Storage**: Stores artifacts, media, and color data in MySQL/TiDB Cloud
- **Analytics**: Executes 20+ predefined analytical queries
- **Visualization**: Interactive Plotly charts in Streamlit dashboards

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages:**
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### API Key Setup

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

Set environment variable:
```bash
export HARVARD_API_KEY="your_api_key_here"
```

Or use `.env` file:
```
HARVARD_API_KEY=your_api_key_here
```

### Database Configuration

Configure MySQL/TiDB connection in your code:
```python
import os
import mysql.connector

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
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    dated VARCHAR(255),
    department VARCHAR(255),
    classification VARCHAR(255),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    url VARCHAR(500)
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    format VARCHAR(50),
    width INT,
    height INT,
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

## Key Components

### 1. API Data Extraction

```python
import requests
import pandas as pd
import os

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    response.raise_for_status()
    return response.json()

def collect_all_artifacts(api_key, max_pages=10):
    """Collect multiple pages of artifacts"""
    all_records = []
    
    for page in range(1, max_pages + 1):
        data = fetch_artifacts(api_key, page=page)
        all_records.extend(data.get('records', []))
        
        # Check if more pages available
        if page >= data.get('info', {}).get('pages', 0):
            break
    
    return all_records
```

### 2. ETL Transformation

```python
def transform_artifacts(records):
    """Transform JSON records into normalized dataframes"""
    
    # Extract metadata
    metadata_list = []
    media_list = []
    colors_list = []
    
    for record in records:
        artifact_id = record.get('id')
        
        # Metadata
        metadata_list.append({
            'id': artifact_id,
            'title': record.get('title'),
            'culture': record.get('culture'),
            'century': record.get('century'),
            'dated': record.get('dated'),
            'department': record.get('department'),
            'classification': record.get('classification'),
            'medium': record.get('medium'),
            'dimensions': record.get('dimensions'),
            'creditline': record.get('creditline'),
            'url': record.get('url')
        })
        
        # Media (nested array)
        for img in record.get('images', []):
            media_list.append({
                'artifact_id': artifact_id,
                'baseimageurl': img.get('baseimageurl'),
                'iiifbaseuri': img.get('iiifbaseuri'),
                'format': img.get('format'),
                'width': img.get('width'),
                'height': img.get('height')
            })
        
        # Colors (nested array)
        for color in record.get('colors', []):
            colors_list.append({
                'artifact_id': artifact_id,
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

### 3. SQL Loading

```python
def load_to_database(df_metadata, df_media, df_colors, db_config):
    """Load transformed data into MySQL database"""
    import mysql.connector
    
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    
    # Load metadata
    for _, row in df_metadata.iterrows():
        sql = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, dated, department, classification, 
         medium, dimensions, creditline, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.execute(sql, tuple(row))
    
    # Load media
    for _, row in df_media.iterrows():
        sql = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, iiifbaseuri, format, width, height)
        VALUES (%s, %s, %s, %s, %s, %s)
        """
        cursor.execute(sql, tuple(row))
    
    # Load colors
    for _, row in df_colors.iterrows():
        sql = """
        INSERT INTO artifactcolors 
        (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
        """
        cursor.execute(sql, tuple(row))
    
    conn.commit()
    cursor.close()
    conn.close()
```

### 4. Complete ETL Pipeline

```python
import os
from dotenv import load_dotenv

def run_etl_pipeline():
    """Execute complete ETL pipeline"""
    load_dotenv()
    
    # Extract
    api_key = os.getenv('HARVARD_API_KEY')
    records = collect_all_artifacts(api_key, max_pages=5)
    
    # Transform
    df_metadata, df_media, df_colors = transform_artifacts(records)
    
    # Load
    db_config = {
        'host': os.getenv('DB_HOST'),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    
    load_to_database(df_metadata, df_media, df_colors, db_config)
    
    print(f"Loaded {len(df_metadata)} artifacts")
    print(f"Loaded {len(df_media)} media records")
    print(f"Loaded {len(df_colors)} color records")

if __name__ == "__main__":
    run_etl_pipeline()
```

## Analytical SQL Queries

### Sample Analytics Queries

```python
# Query 1: Top 10 Cultures by Artifact Count
query_1 = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10
"""

# Query 2: Artifacts by Century
query_2 = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC
"""

# Query 3: Department Distribution
query_3 = """
SELECT department, COUNT(*) as total_artifacts
FROM artifactmetadata
GROUP BY department
ORDER BY total_artifacts DESC
"""

# Query 4: Most Common Colors
query_4 = """
SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
FROM artifactcolors
GROUP BY color
ORDER BY frequency DESC
LIMIT 15
"""

# Query 5: Media Formats
query_5 = """
SELECT format, COUNT(*) as count
FROM artifactmedia
WHERE format IS NOT NULL
GROUP BY format
ORDER BY count DESC
"""

# Query 6: Artifacts with Multiple Images
query_6 = """
SELECT a.title, a.culture, COUNT(m.media_id) as image_count
FROM artifactmetadata a
JOIN artifactmedia m ON a.id = m.artifact_id
GROUP BY a.id, a.title, a.culture
HAVING image_count > 1
ORDER BY image_count DESC
LIMIT 20
"""
```

## Streamlit Dashboard

### Basic Dashboard Structure

```python
import streamlit as st
import pandas as pd
import plotly.express as px
import mysql.connector
import os

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def run_query(query):
    """Execute SQL query and return DataFrame"""
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# Title
st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Sidebar
st.sidebar.header("Query Selection")

queries = {
    "Top Cultures": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    "Century Distribution": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    "Color Analysis": """
        SELECT color, COUNT(*) as frequency
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """
}

selected_query = st.sidebar.selectbox("Choose Analysis", list(queries.keys()))

# Execute query
if st.button("Run Analysis"):
    with st.spinner("Executing query..."):
        df = run_query(queries[selected_query])
        
        # Display table
        st.subheader("Query Results")
        st.dataframe(df)
        
        # Visualization
        if len(df) > 0:
            st.subheader("Visualization")
            
            # Auto-detect columns for plotting
            x_col = df.columns[0]
            y_col = df.columns[1]
            
            fig = px.bar(df, x=x_col, y=y_col, 
                        title=f"{selected_query} Analysis")
            st.plotly_chart(fig, use_container_width=True)
```

### Advanced Dashboard with Multiple Tabs

```python
# Create tabs
tab1, tab2, tab3 = st.tabs(["Overview", "Detailed Analytics", "Color Analysis"])

with tab1:
    st.header("Collection Overview")
    
    col1, col2, col3 = st.columns(3)
    
    # Total artifacts
    total_query = "SELECT COUNT(*) as total FROM artifactmetadata"
    total = run_query(total_query)['total'][0]
    col1.metric("Total Artifacts", total)
    
    # Total images
    images_query = "SELECT COUNT(*) as total FROM artifactmedia"
    total_images = run_query(images_query)['total'][0]
    col2.metric("Total Images", total_images)
    
    # Unique cultures
    cultures_query = "SELECT COUNT(DISTINCT culture) as total FROM artifactmetadata"
    total_cultures = run_query(cultures_query)['total'][0]
    col3.metric("Unique Cultures", total_cultures)

with tab2:
    st.header("Department Analysis")
    
    dept_query = """
    SELECT department, COUNT(*) as count
    FROM artifactmetadata
    GROUP BY department
    ORDER BY count DESC
    """
    dept_df = run_query(dept_query)
    
    fig = px.pie(dept_df, values='count', names='department',
                 title='Artifacts by Department')
    st.plotly_chart(fig, use_container_width=True)

with tab3:
    st.header("Color Spectrum Analysis")
    
    color_query = """
    SELECT spectrum, COUNT(*) as count, AVG(percent) as avg_percent
    FROM artifactcolors
    WHERE spectrum IS NOT NULL
    GROUP BY spectrum
    ORDER BY count DESC
    """
    color_df = run_query(color_query)
    
    fig = px.bar(color_df, x='spectrum', y='count',
                 color='avg_percent',
                 title='Color Spectrum Distribution')
    st.plotly_chart(fig, use_container_width=True)
```

## Running the Application

```bash
# Set environment variables
export HARVARD_API_KEY="your_api_key"
export DB_HOST="localhost"
export DB_USER="root"
export DB_PASSWORD="your_password"
export DB_NAME="harvard_artifacts"

# Run Streamlit app
streamlit run app.py
```

## Common Patterns

### Batch Processing with Rate Limiting

```python
import time

def batch_collect_artifacts(api_key, total_records=1000, batch_size=100):
    """Collect artifacts in batches with rate limiting"""
    all_records = []
    pages_needed = (total_records // batch_size) + 1
    
    for page in range(1, pages_needed + 1):
        try:
            data = fetch_artifacts(api_key, page=page, size=batch_size)
            records = data.get('records', [])
            all_records.extend(records)
            
            print(f"Fetched page {page}: {len(records)} records")
            
            # Rate limiting: sleep between requests
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error on page {page}: {e}")
            continue
    
    return all_records
```

### Incremental Loading

```python
def get_latest_artifact_id(db_config):
    """Get the latest artifact ID from database"""
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()[0]
    cursor.close()
    conn.close()
    return result or 0

def incremental_etl(api_key, db_config):
    """Only load new artifacts"""
    latest_id = get_latest_artifact_id(db_config)
    
    # Fetch new records (API specific filtering needed)
    records = collect_all_artifacts(api_key, max_pages=5)
    
    # Filter records newer than latest_id
    new_records = [r for r in records if r.get('id', 0) > latest_id]
    
    if new_records:
        df_metadata, df_media, df_colors = transform_artifacts(new_records)
        load_to_database(df_metadata, df_media, df_colors, db_config)
        print(f"Loaded {len(new_records)} new artifacts")
    else:
        print("No new artifacts to load")
```

## Troubleshooting

### API Rate Limiting
```python
# Add exponential backoff
import time
from requests.exceptions import HTTPError

def fetch_with_retry(api_key, page, max_retries=3):
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except HTTPError as e:
            if e.response.status_code == 429:  # Rate limited
                wait_time = 2 ** attempt
                print(f"Rate limited, waiting {wait_time}s")
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues
```python
# Use connection pooling
from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    **db_config
)

def get_connection():
    return db_pool.get_connection()
```

### Memory Management for Large Datasets
```python
# Process in chunks
def chunked_load(df, chunk_size=1000):
    for start in range(0, len(df), chunk_size):
        chunk = df.iloc[start:start + chunk_size]
        # Load chunk to database
        yield chunk
```
