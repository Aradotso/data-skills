---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - create a data engineering workflow with Harvard artifacts API
  - set up analytics dashboard for museum collection data
  - extract and analyze Harvard Art Museums API data
  - build a Streamlit app for art museum analytics
  - process Harvard artifacts data into SQL database
  - create visualization dashboard for museum collection
  - implement ETL for Harvard Art Museums collection
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. It demonstrates ETL pipelines, SQL analytics, and interactive visualization with Streamlit.

## What It Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Connects to Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts artifact metadata, transforms nested JSON, loads into relational SQL database
- **SQL Analytics**: Executes predefined analytical queries on artifact data
- **Visualization**: Creates interactive dashboards with Plotly and Streamlit
- **Database Design**: Implements normalized schema for artifacts, media, and color data

## Installation

```bash
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App
pip install -r requirements.txt
```

### Required Dependencies

```txt
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
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit [Harvard Art Museums API](https://harvardartmuseums.org/collections/api)
2. Register for a free API key
3. Add to your `.env` file

### Database Setup

The application supports MySQL or TiDB Cloud. Create the database:

```sql
CREATE DATABASE harvard_artifacts;
```

## Database Schema

The ETL pipeline creates three main tables:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    dated VARCHAR(200),
    accession_number VARCHAR(100),
    people TEXT,
    url VARCHAR(500)
);

-- Artifact media table
CREATE TABLE artifactmedia (
    artifact_id INT,
    media_id INT,
    base_url VARCHAR(500),
    format VARCHAR(50),
    height INT,
    width INT,
    PRIMARY KEY (artifact_id, media_id),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percent FLOAT,
    PRIMARY KEY (artifact_id, color),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key API Patterns

### Fetching Artifacts with Pagination

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
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
        return {
            'records': data.get('records', []),
            'total': data.get('info', {}).get('totalrecords', 0),
            'pages': data.get('info', {}).get('pages', 0)
        }
    else:
        raise Exception(f"API Error: {response.status_code}")

# Example usage
result = fetch_artifacts(page=1, size=50)
print(f"Total artifacts: {result['total']}")
print(f"Fetched: {len(result['records'])} artifacts")
```

### Rate Limiting and Batch Collection

```python
import time

def collect_artifacts_batch(num_pages=5, delay=1):
    """Collect multiple pages with rate limiting"""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        result = fetch_artifacts(page=page, size=100)
        all_artifacts.extend(result['records'])
        
        # Rate limiting
        if page < num_pages:
            time.sleep(delay)
    
    return all_artifacts
```

## ETL Pipeline Implementation

### Extract: Parse API Response

```python
def extract_artifact_metadata(artifact):
    """Extract metadata from artifact JSON"""
    return {
        'id': artifact.get('id'),
        'title': artifact.get('title'),
        'culture': artifact.get('culture'),
        'period': artifact.get('period'),
        'century': artifact.get('century'),
        'classification': artifact.get('classification'),
        'department': artifact.get('department'),
        'division': artifact.get('division'),
        'dated': artifact.get('dated'),
        'accession_number': artifact.get('accessionyear'),
        'people': ', '.join([p.get('name', '') for p in artifact.get('people', [])]),
        'url': artifact.get('url')
    }

def extract_artifact_media(artifact):
    """Extract media data from artifact JSON"""
    media_list = []
    artifact_id = artifact.get('id')
    
    for image in artifact.get('images', []):
        media_list.append({
            'artifact_id': artifact_id,
            'media_id': image.get('imageid'),
            'base_url': image.get('baseimageurl'),
            'format': image.get('format'),
            'height': image.get('height'),
            'width': image.get('width')
        })
    
    return media_list

def extract_artifact_colors(artifact):
    """Extract color data from artifact JSON"""
    colors_list = []
    artifact_id = artifact.get('id')
    
    for color in artifact.get('colors', []):
        colors_list.append({
            'artifact_id': artifact_id,
            'color': color.get('color'),
            'spectrum': color.get('spectrum'),
            'percent': color.get('percent')
        })
    
    return colors_list
```

### Transform and Load: Insert into SQL

```python
import mysql.connector
from mysql.connector import Error
import pandas as pd

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_artifacts_to_db(artifacts):
    """Load artifacts into SQL database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    # Prepare data
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        metadata_records.append(extract_artifact_metadata(artifact))
        media_records.extend(extract_artifact_media(artifact))
        color_records.extend(extract_artifact_colors(artifact))
    
    # Insert metadata
    metadata_df = pd.DataFrame(metadata_records)
    for _, row in metadata_df.iterrows():
        cursor.execute("""
            INSERT IGNORE INTO artifactmetadata 
            (id, title, culture, period, century, classification, 
             department, division, dated, accession_number, people, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """, tuple(row))
    
    # Insert media
    for media in media_records:
        cursor.execute("""
            INSERT IGNORE INTO artifactmedia 
            (artifact_id, media_id, base_url, format, height, width)
            VALUES (%s, %s, %s, %s, %s, %s)
        """, (media['artifact_id'], media['media_id'], media['base_url'],
              media['format'], media['height'], media['width']))
    
    # Insert colors
    for color in color_records:
        cursor.execute("""
            INSERT IGNORE INTO artifactcolors 
            (artifact_id, color, spectrum, percent)
            VALUES (%s, %s, %s, %s)
        """, (color['artifact_id'], color['color'], 
              color['spectrum'], color['percent']))
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print(f"Loaded {len(metadata_records)} artifacts to database")
```

## SQL Analytics Queries

### Sample Analytical Queries

```python
# Top 10 cultures by artifact count
query_cultures = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10;
"""

# Artifacts by century
query_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC;
"""

# Media format distribution
query_media = """
SELECT format, COUNT(*) as count
FROM artifactmedia
GROUP BY format
ORDER BY count DESC;
"""

# Most common colors
query_colors = """
SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
FROM artifactcolors
GROUP BY color
ORDER BY frequency DESC
LIMIT 15;
"""

# Department statistics
query_departments = """
SELECT department, 
       COUNT(*) as total_artifacts,
       COUNT(DISTINCT culture) as unique_cultures
FROM artifactmetadata
GROUP BY department
ORDER BY total_artifacts DESC;
"""

def execute_query(query):
    """Execute SQL query and return DataFrame"""
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

## Streamlit Dashboard

### Basic App Structure

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Sidebar for navigation
page = st.sidebar.selectbox("Select Page", [
    "Data Collection",
    "Analytics Dashboard",
    "Visualizations"
])

if page == "Data Collection":
    st.header("📥 Collect Artifact Data")
    
    num_pages = st.number_input("Number of pages to fetch", 
                                  min_value=1, max_value=100, value=5)
    
    if st.button("Start Collection"):
        with st.spinner("Fetching data from API..."):
            artifacts = collect_artifacts_batch(num_pages=num_pages)
            load_artifacts_to_db(artifacts)
            st.success(f"Successfully loaded {len(artifacts)} artifacts!")

elif page == "Analytics Dashboard":
    st.header("📊 SQL Analytics")
    
    query_options = {
        "Top Cultures": query_cultures,
        "Century Distribution": query_century,
        "Media Formats": query_media,
        "Color Analysis": query_colors,
        "Department Stats": query_departments
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
    
    if st.button("Run Query"):
        df = execute_query(query_options[selected_query])
        
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)
```

### Interactive Visualizations

```python
def create_culture_chart(df):
    """Create interactive culture distribution chart"""
    fig = px.bar(df, 
                 x='culture', 
                 y='artifact_count',
                 title='Top 10 Cultures by Artifact Count',
                 labels={'artifact_count': 'Number of Artifacts'},
                 color='artifact_count',
                 color_continuous_scale='Viridis')
    
    fig.update_layout(xaxis_tickangle=-45)
    return fig

def create_color_pie_chart(df):
    """Create pie chart for color distribution"""
    fig = px.pie(df, 
                 values='frequency', 
                 names='color',
                 title='Color Distribution Across Artifacts',
                 hole=0.3)
    return fig

# In Streamlit app
st.plotly_chart(create_culture_chart(df), use_container_width=True)
```

## Running the Application

### Start the Streamlit App

```bash
streamlit run app.py
```

The app will be available at `http://localhost:8501`

### Command Line ETL (Alternative)

```python
# etl_script.py
if __name__ == "__main__":
    import argparse
    
    parser = argparse.ArgumentParser()
    parser.add_argument('--pages', type=int, default=5, 
                       help='Number of pages to fetch')
    parser.add_argument('--delay', type=float, default=1.0,
                       help='Delay between requests')
    
    args = parser.parse_args()
    
    print(f"Fetching {args.pages} pages...")
    artifacts = collect_artifacts_batch(num_pages=args.pages, 
                                       delay=args.delay)
    
    print("Loading to database...")
    load_artifacts_to_db(artifacts)
    
    print("ETL complete!")
```

Run from command line:

```bash
python etl_script.py --pages 10 --delay 1.5
```

## Common Patterns

### Error Handling in API Calls

```python
def safe_api_fetch(page, retries=3):
    """Fetch with retry logic"""
    for attempt in range(retries):
        try:
            return fetch_artifacts(page=page)
        except Exception as e:
            if attempt < retries - 1:
                print(f"Retry {attempt + 1}/{retries}")
                time.sleep(2 ** attempt)  # Exponential backoff
            else:
                raise e
```

### Incremental Data Loading

```python
def get_latest_artifact_id():
    """Get the most recent artifact ID in database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()[0]
    cursor.close()
    conn.close()
    return result or 0

def incremental_load():
    """Load only new artifacts"""
    latest_id = get_latest_artifact_id()
    # Fetch artifacts with id > latest_id
    # Implementation depends on API capabilities
```

## Troubleshooting

### API Rate Limiting

If you encounter rate limit errors:

```python
import time
from functools import wraps

def rate_limit(calls_per_minute=60):
    def decorator(func):
        last_called = [0.0]
        
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            wait_time = 60.0 / calls_per_minute
            
            if elapsed < wait_time:
                time.sleep(wait_time - elapsed)
            
            result = func(*args, **kwargs)
            last_called[0] = time.time()
            return result
        
        return wrapper
    return decorator

@rate_limit(calls_per_minute=30)
def fetch_artifacts_limited(page):
    return fetch_artifacts(page=page)
```

### Database Connection Issues

```python
def safe_db_connection(max_retries=3):
    """Create database connection with retries"""
    for attempt in range(max_retries):
        try:
            conn = get_db_connection()
            conn.ping(reconnect=True)
            return conn
        except Error as e:
            if attempt < max_retries - 1:
                time.sleep(2)
            else:
                raise Exception(f"Database connection failed: {e}")
```

### Memory Management for Large Datasets

```python
def batch_insert(records, batch_size=1000):
    """Insert records in batches to manage memory"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    for i in range(0, len(records), batch_size):
        batch = records[i:i + batch_size]
        # Insert batch
        for record in batch:
            cursor.execute("INSERT IGNORE INTO ...", record)
        conn.commit()
        print(f"Processed {min(i + batch_size, len(records))}/{len(records)}")
    
    cursor.close()
    conn.close()
```

### Handling Missing Data

```python
def safe_extract(artifact, field, default=None):
    """Safely extract field with default value"""
    return artifact.get(field, default) or default

def clean_metadata(artifact):
    """Clean and validate metadata"""
    return {
        'id': artifact.get('id'),
        'title': safe_extract(artifact, 'title', 'Unknown'),
        'culture': safe_extract(artifact, 'culture', 'Not Specified'),
        'century': safe_extract(artifact, 'century', 'Unknown'),
        # ... other fields
    }
```

This skill covers the complete workflow for building ETL pipelines and analytics dashboards with the Harvard Art Museums API, from data collection through visualization.
