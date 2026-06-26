---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Streamlit, MySQL, and Plotly
triggers:
  - how do I fetch data from Harvard Art Museums API
  - build an ETL pipeline for museum artifacts
  - create analytics dashboard with Streamlit
  - query Harvard museum collection data
  - set up TiDB database for artifact analytics
  - visualize museum data with Plotly
  - extract and transform Harvard API JSON data
  - create SQL schema for museum artifacts
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering solution for collecting, transforming, storing, and analyzing artifact data from the Harvard Art Museums API. It demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualization using Streamlit.

## What It Does

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON into relational database tables
- **SQL Storage**: Stores data in MySQL/TiDB with proper schema design
- **Analytics**: Executes predefined SQL queries for insights
- **Visualization**: Creates interactive dashboards using Streamlit and Plotly

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

### 1. Harvard API Key

Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)

Store in environment variable or `.env` file:
```bash
HARVARD_API_KEY=your_api_key_here
```

### 2. Database Configuration

Set up MySQL or TiDB Cloud credentials:
```bash
DB_HOST=your_db_host
DB_PORT=4000
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### 3. Streamlit Secrets (Optional)

Create `.streamlit/secrets.toml`:
```toml
[api]
harvard_api_key = "your_api_key"

[database]
host = "your_db_host"
port = 4000
user = "your_username"
password = "your_password"
database = "harvard_artifacts"
```

## Database Schema

The project uses three main tables:

```sql
-- Artifact metadata
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accessionyear INT
);

-- Artifact media/images
CREATE TABLE artifactmedia (
    mediaid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

-- Artifact colors
CREATE TABLE artifactcolors (
    colorid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## Key API Patterns

### Fetching Artifacts with Pagination

```python
import requests
import os

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = 'https://api.harvardartmuseums.org/object'

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard API with pagination"""
    params = {
        'apikey': API_KEY,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(BASE_URL, params=params)
    response.raise_for_status()
    
    data = response.json()
    return data['records'], data['info']

# Fetch multiple pages
all_artifacts = []
for page in range(1, 6):  # Get first 5 pages
    artifacts, info = fetch_artifacts(page=page)
    all_artifacts.extend(artifacts)
    print(f"Fetched page {page}/{info['pages']}")
```

### ETL: Extract and Transform

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """Transform raw API data into structured metadata"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'accessionyear': artifact.get('accessionyear')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(artifacts):
    """Extract media/image URLs"""
    media_list = []
    
    for artifact in artifacts:
        media = {
            'objectid': artifact.get('objectid'),
            'baseimageurl': artifact.get('baseimageurl'),
            'primaryimageurl': artifact.get('primaryimageurl')
        }
        media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(artifacts):
    """Extract color information"""
    colors_list = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        colors = artifact.get('colors', [])
        
        for color_info in colors:
            color_record = {
                'objectid': objectid,
                'color': color_info.get('color'),
                'spectrum': color_info.get('spectrum'),
                'percent': color_info.get('percent')
            }
            colors_list.append(color_record)
    
    return pd.DataFrame(colors_list)
```

### Load: Database Operations

```python
import mysql.connector
import os

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_metadata(df):
    """Batch insert artifact metadata"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (objectid, title, culture, period, century, classification, 
     department, dated, medium, dimensions, creditline, accessionyear)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    data = df.to_records(index=False).tolist()
    cursor.executemany(insert_query, data)
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print(f"Inserted {len(df)} metadata records")

def load_colors(df):
    """Batch insert color data"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors (objectid, color, spectrum, percent)
    VALUES (%s, %s, %s, %s)
    """
    
    data = df[['objectid', 'color', 'spectrum', 'percent']].to_records(index=False).tolist()
    cursor.executemany(insert_query, data)
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print(f"Inserted {len(df)} color records")
```

## Streamlit Application Patterns

### Basic App Structure

```python
import streamlit as st
import pandas as pd

st.set_page_config(
    page_title="Harvard Artifacts Analytics",
    page_icon="🏛️",
    layout="wide"
)

st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Sidebar for navigation
page = st.sidebar.selectbox(
    "Select Page",
    ["Data Collection", "Analytics", "Visualizations"]
)

if page == "Data Collection":
    st.header("📥 Collect Artifact Data")
    
    num_pages = st.number_input("Number of pages to fetch", 1, 100, 5)
    
    if st.button("Fetch Data"):
        with st.spinner("Fetching artifacts..."):
            all_artifacts = []
            progress_bar = st.progress(0)
            
            for i in range(num_pages):
                artifacts, _ = fetch_artifacts(page=i+1)
                all_artifacts.extend(artifacts)
                progress_bar.progress((i + 1) / num_pages)
            
            st.success(f"Fetched {len(all_artifacts)} artifacts!")
            
            # Transform and display
            df = transform_artifact_metadata(all_artifacts)
            st.dataframe(df.head(20))
```

### SQL Analytics Dashboard

```python
def run_sql_query(query):
    """Execute SQL query and return DataFrame"""
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# Predefined queries
QUERIES = {
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
    "Most Common Colors": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    "Artifacts with Images": """
        SELECT 
            COUNT(DISTINCT m.objectid) as total_artifacts,
            COUNT(DISTINCT CASE WHEN am.primaryimageurl IS NOT NULL THEN m.objectid END) as with_images,
            ROUND(COUNT(DISTINCT CASE WHEN am.primaryimageurl IS NOT NULL THEN m.objectid END) * 100.0 / COUNT(DISTINCT m.objectid), 2) as image_percentage
        FROM artifactmetadata m
        LEFT JOIN artifactmedia am ON m.objectid = am.objectid
    """
}

st.header("📊 SQL Analytics")

query_name = st.selectbox("Select Analysis", list(QUERIES.keys()))

if st.button("Run Query"):
    query = QUERIES[query_name]
    
    with st.expander("View SQL Query"):
        st.code(query, language="sql")
    
    df = run_sql_query(query)
    
    st.dataframe(df)
    
    # Auto-generate visualization
    if len(df.columns) >= 2:
        st.plotly_chart(
            px.bar(df, x=df.columns[0], y=df.columns[1], title=query_name)
        )
```

### Visualization with Plotly

```python
import plotly.express as px
import plotly.graph_objects as go

def create_culture_distribution():
    """Create interactive bar chart of artifacts by culture"""
    query = """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 20
    """
    df = run_sql_query(query)
    
    fig = px.bar(
        df,
        x='culture',
        y='count',
        title='Top 20 Cultures in Collection',
        labels={'culture': 'Culture', 'count': 'Number of Artifacts'},
        color='count',
        color_continuous_scale='Viridis'
    )
    
    fig.update_layout(xaxis_tickangle=-45)
    return fig

def create_color_spectrum_pie():
    """Create pie chart of color spectrum distribution"""
    query = """
        SELECT spectrum, COUNT(*) as count
        FROM artifactcolors
        WHERE spectrum IS NOT NULL
        GROUP BY spectrum
    """
    df = run_sql_query(query)
    
    fig = px.pie(
        df,
        values='count',
        names='spectrum',
        title='Color Spectrum Distribution'
    )
    
    return fig

# Display in Streamlit
st.plotly_chart(create_culture_distribution(), use_container_width=True)
st.plotly_chart(create_color_spectrum_pie(), use_container_width=True)
```

## Common Patterns

### Complete ETL Pipeline

```python
def run_etl_pipeline(num_pages=5):
    """Run complete ETL pipeline"""
    # Extract
    print("Extracting data from API...")
    all_artifacts = []
    for page in range(1, num_pages + 1):
        artifacts, _ = fetch_artifacts(page=page)
        all_artifacts.extend(artifacts)
    
    # Transform
    print("Transforming data...")
    metadata_df = transform_artifact_metadata(all_artifacts)
    media_df = transform_artifact_media(all_artifacts)
    colors_df = transform_artifact_colors(all_artifacts)
    
    # Load
    print("Loading data into database...")
    load_metadata(metadata_df)
    load_metadata(media_df)  # Assuming similar function for media
    load_colors(colors_df)
    
    print("ETL pipeline completed successfully!")
    
    return {
        'metadata_count': len(metadata_df),
        'media_count': len(media_df),
        'colors_count': len(colors_df)
    }
```

### Error Handling and Rate Limiting

```python
import time
from requests.exceptions import RequestException

def fetch_artifacts_safe(page=1, size=100, retry=3):
    """Fetch artifacts with retry logic and rate limiting"""
    for attempt in range(retry):
        try:
            params = {
                'apikey': os.getenv('HARVARD_API_KEY'),
                'page': page,
                'size': size
            }
            
            response = requests.get(BASE_URL, params=params, timeout=10)
            response.raise_for_status()
            
            # Rate limiting: wait between requests
            time.sleep(0.5)
            
            return response.json()['records']
            
        except RequestException as e:
            print(f"Attempt {attempt + 1} failed: {e}")
            if attempt < retry - 1:
                time.sleep(2 ** attempt)  # Exponential backoff
            else:
                raise
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Troubleshooting

**API Rate Limiting:**
- Add delays between requests using `time.sleep()`
- Implement exponential backoff for retries
- Monitor API response headers for rate limit info

**Database Connection Issues:**
- Verify credentials in environment variables
- Check firewall/security group settings for TiDB Cloud
- Ensure SSL requirements are met if needed

**Memory Issues with Large Datasets:**
- Process data in smaller batches
- Use database cursors for large result sets
- Implement pagination in Streamlit displays

**Missing Data:**
- Handle None values in transformations with `.get()` and defaults
- Use `ON DUPLICATE KEY UPDATE` for idempotent inserts
- Validate data before insertion

**Streamlit Performance:**
- Use `@st.cache_data` for expensive computations
- Cache database connections with `@st.cache_resource`
- Limit initial data loads and use pagination

```python
@st.cache_data(ttl=3600)
def cached_query(query):
    """Cache query results for 1 hour"""
    return run_sql_query(query)
```
