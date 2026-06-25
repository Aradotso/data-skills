---
name: harvard-artifacts-etl-streamlit-app
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I fetch data from Harvard Art Museums API
  - build an ETL pipeline for museum artifacts
  - create a Streamlit dashboard for Harvard artifacts
  - set up SQL database for artifact metadata
  - query Harvard museum collection data
  - visualize art museum data with Plotly
  - implement pagination for Harvard API
  - design schema for artifact data
---

# Harvard Artifacts ETL & Analytics App

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is an end-to-end data engineering application that demonstrates real-world ETL pipelines using the Harvard Art Museums API. It extracts artifact data, transforms it into relational structures, loads it into SQL databases, and provides interactive analytics through a Streamlit dashboard.

## What This Project Does

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON into normalized relational tables
- **SQL Storage**: Stores data in MySQL/TiDB with proper schema design
- **Analytics**: Executes 20+ predefined SQL queries for insights
- **Visualization**: Creates interactive dashboards with Plotly charts

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
export DB_USER="your_db_user"
export DB_PASSWORD="your_db_password"
export DB_NAME="harvard_artifacts"
```

## Configuration

### API Key Setup

Get your free API key from [Harvard Art Museums API](https://harvardartmuseums.org/collections/api):

```python
import os

# In your code, always reference the environment variable
API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = 'https://api.harvardartmuseums.org/object'
```

### Database Configuration

```python
import mysql.connector
import os

db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

conn = mysql.connector.connect(**db_config)
```

## Database Schema

The application uses three main tables:

```sql
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    accessionyear INT,
    technique VARCHAR(500),
    medium VARCHAR(500),
    verificationlevel INT,
    totalpageviews INT,
    totaluniquepageviews INT
);

CREATE TABLE artifactmedia (
    mediaid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

CREATE TABLE artifactcolors (
    colorid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## Core ETL Pipeline

### Extract: Fetch Data from API

```python
import requests
import os

def fetch_artifacts(num_records=100, page_size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    API_KEY = os.getenv('HARVARD_API_KEY')
    BASE_URL = 'https://api.harvardartmuseums.org/object'
    
    all_records = []
    page = 1
    
    while len(all_records) < num_records:
        params = {
            'apikey': API_KEY,
            'size': page_size,
            'page': page,
            'hasimage': 1  # Only get artifacts with images
        }
        
        response = requests.get(BASE_URL, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            
            if not records:
                break
                
            all_records.extend(records)
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return all_records[:num_records]
```

### Transform: Process JSON Data

```python
import pandas as pd

def transform_metadata(artifacts):
    """
    Transform artifact JSON into metadata dataframe
    """
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'dated': artifact.get('dated', 'Unknown'),
            'accessionyear': artifact.get('accessionyear'),
            'technique': artifact.get('technique', 'Unknown'),
            'medium': artifact.get('medium', 'Unknown'),
            'verificationlevel': artifact.get('verificationlevel', 0),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_media(artifacts):
    """
    Extract media/image information
    """
    media_list = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        media = {
            'objectid': objectid,
            'baseimageurl': artifact.get('baseimageurl'),
            'primaryimageurl': artifact.get('primaryimageurl'),
            'iiifbaseuri': artifact.get('iiifbaseuri')
        }
        media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_colors(artifacts):
    """
    Extract color information from nested structure
    """
    colors_list = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        colors = artifact.get('colors', [])
        
        for color_data in colors:
            color_record = {
                'objectid': objectid,
                'color': color_data.get('color'),
                'spectrum': color_data.get('spectrum'),
                'percentage': color_data.get('percent')
            }
            colors_list.append(color_record)
    
    return pd.DataFrame(colors_list)
```

### Load: Insert into SQL Database

```python
def load_to_database(metadata_df, media_df, colors_df, db_config):
    """
    Batch load dataframes into SQL database
    """
    import mysql.connector
    
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    
    # Load metadata
    metadata_tuples = [tuple(x) for x in metadata_df.to_numpy()]
    metadata_sql = """
        INSERT INTO artifactmetadata 
        (objectid, title, culture, century, classification, department, 
         dated, accessionyear, technique, medium, verificationlevel, 
         totalpageviews, totaluniquepageviews)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE objectid=objectid
    """
    cursor.executemany(metadata_sql, metadata_tuples)
    
    # Load media
    media_tuples = [tuple(x) for x in media_df.to_numpy()]
    media_sql = """
        INSERT INTO artifactmedia 
        (objectid, baseimageurl, primaryimageurl, iiifbaseuri)
        VALUES (%s, %s, %s, %s)
    """
    cursor.executemany(media_sql, media_tuples)
    
    # Load colors
    colors_tuples = [tuple(x) for x in colors_df.to_numpy()]
    colors_sql = """
        INSERT INTO artifactcolors 
        (objectid, color, spectrum, percentage)
        VALUES (%s, %s, %s, %s)
    """
    cursor.executemany(colors_sql, colors_tuples)
    
    conn.commit()
    cursor.close()
    conn.close()
```

## Streamlit Application

### Main App Structure

```python
import streamlit as st
import pandas as pd
import plotly.express as px
import os

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Sidebar for configuration
with st.sidebar:
    st.header("Configuration")
    num_records = st.number_input("Number of artifacts to fetch", 
                                   min_value=10, max_value=1000, 
                                   value=100)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching data..."):
            artifacts = fetch_artifacts(num_records)
            
        with st.spinner("Transforming data..."):
            metadata_df = transform_metadata(artifacts)
            media_df = transform_media(artifacts)
            colors_df = transform_colors(artifacts)
            
        with st.spinner("Loading to database..."):
            db_config = {
                'host': os.getenv('DB_HOST'),
                'user': os.getenv('DB_USER'),
                'password': os.getenv('DB_PASSWORD'),
                'database': os.getenv('DB_NAME')
            }
            load_to_database(metadata_df, media_df, colors_df, db_config)
            
        st.success(f"✅ Successfully loaded {len(metadata_df)} artifacts!")

# Analytics Section
st.header("📊 Analytics Queries")

queries = {
    "Top 10 Cultures by Artifact Count": """
        SELECT culture, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE culture != 'Unknown'
        GROUP BY culture 
        ORDER BY count DESC 
        LIMIT 10
    """,
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE century != 'Unknown'
        GROUP BY century 
        ORDER BY count DESC
    """,
    "Most Common Color Spectrum": """
        SELECT spectrum, COUNT(*) as count, AVG(percentage) as avg_percentage
        FROM artifactcolors 
        WHERE spectrum IS NOT NULL
        GROUP BY spectrum 
        ORDER BY count DESC
    """,
    "Department Distribution": """
        SELECT department, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE department != 'Unknown'
        GROUP BY department 
        ORDER BY count DESC
    """,
    "Most Viewed Artifacts": """
        SELECT title, culture, totalpageviews 
        FROM artifactmetadata 
        WHERE totalpageviews > 0
        ORDER BY totalpageviews DESC 
        LIMIT 10
    """
}

selected_query = st.selectbox("Select Analysis", list(queries.keys()))

if st.button("Execute Query"):
    import mysql.connector
    
    db_config = {
        'host': os.getenv('DB_HOST'),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    
    conn = mysql.connector.connect(**db_config)
    df = pd.read_sql(queries[selected_query], conn)
    conn.close()
    
    st.dataframe(df, use_container_width=True)
    
    # Auto-generate visualization
    if len(df.columns) >= 2:
        fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                     title=selected_query)
        st.plotly_chart(fig, use_container_width=True)
```

## Common Patterns

### Rate Limiting and Error Handling

```python
import time

def fetch_with_retry(url, params, max_retries=3):
    """
    Fetch with exponential backoff retry logic
    """
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            
            if response.status_code == 429:  # Rate limit
                wait_time = 2 ** attempt
                time.sleep(wait_time)
                continue
                
            response.raise_for_status()
            return response.json()
            
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)
    
    return None
```

### Incremental ETL

```python
def get_last_processed_id(conn):
    """
    Get the last processed object ID for incremental loads
    """
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(objectid) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    return result[0] if result[0] else 0

def fetch_new_artifacts(last_id):
    """
    Fetch only artifacts after the last processed ID
    """
    params = {
        'apikey': os.getenv('HARVARD_API_KEY'),
        'size': 100,
        'sort': 'objectid',
        'sortorder': 'asc',
        'objectid': f'>{last_id}'
    }
    
    response = requests.get(BASE_URL, params=params)
    return response.json().get('records', [])
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

## Troubleshooting

### API Key Issues

```python
# Verify API key is loaded
import os

api_key = os.getenv('HARVARD_API_KEY')
if not api_key:
    raise ValueError("HARVARD_API_KEY environment variable not set")

# Test API connection
response = requests.get(
    'https://api.harvardartmuseums.org/object',
    params={'apikey': api_key, 'size': 1}
)
print(f"API Status: {response.status_code}")
```

### Database Connection Issues

```python
# Test database connection
try:
    conn = mysql.connector.connect(**db_config)
    print("✅ Database connected successfully")
    conn.close()
except mysql.connector.Error as e:
    print(f"❌ Database connection failed: {e}")
```

### Missing Data Fields

```python
# Handle missing nested fields safely
def safe_get(dictionary, *keys, default=None):
    """
    Safely access nested dictionary keys
    """
    for key in keys:
        try:
            dictionary = dictionary[key]
        except (KeyError, TypeError):
            return default
    return dictionary

# Usage
color_data = safe_get(artifact, 'colors', 0, 'color', default='Unknown')
```

### Memory Issues with Large Datasets

```python
def fetch_in_batches(total_records, batch_size=100):
    """
    Process large datasets in batches to avoid memory issues
    """
    for offset in range(0, total_records, batch_size):
        artifacts = fetch_artifacts_page(offset, batch_size)
        
        # Transform and load immediately
        metadata_df = transform_metadata(artifacts)
        load_to_database(metadata_df, media_df, colors_df, db_config)
        
        # Clear memory
        del artifacts, metadata_df
```

This skill enables AI agents to help developers build complete ETL pipelines with the Harvard Art Museums API, including data extraction, transformation, SQL storage, and interactive visualization dashboards.
