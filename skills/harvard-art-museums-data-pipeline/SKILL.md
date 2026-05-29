---
name: harvard-art-museums-data-pipeline
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Streamlit, MySQL, and Plotly
triggers:
  - how do I build a data pipeline with the Harvard Art Museums API
  - create an ETL process for museum artifact data
  - set up analytics dashboard for Harvard art collection
  - extract and visualize Harvard museum data
  - build a Streamlit app for art museum analytics
  - process Harvard Art Museums API data with Python
  - create SQL analytics for museum artifacts
  - visualize art collection data with Plotly
---

# Harvard Art Museums Data Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering solution for collecting, processing, and analyzing artifact data from the Harvard Art Museums API. It demonstrates ETL pipelines, SQL database design, analytical queries, and interactive visualizations using Streamlit.

## What It Does

The Harvard Art Museums Data Pipeline:
- Fetches artifact data from the Harvard Art Museums API with pagination and rate limiting
- Transforms nested JSON responses into relational database tables
- Loads structured data into MySQL/TiDB Cloud databases
- Executes analytical SQL queries for insights
- Visualizes results through interactive Streamlit dashboards with Plotly charts

Architecture: **API → ETL → SQL → Analytics → Visualization**

## Installation

### Prerequisites

- Python 3.8+
- MySQL or TiDB Cloud database
- Harvard Art Museums API key (get from https://harvardartmuseums.org/collections/api)

### Setup

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_db_host"
export DB_USER="your_db_user"
export DB_PASSWORD="your_db_password"
export DB_NAME="harvard_artifacts"
```

### Requirements

```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Database Schema

The pipeline creates three main tables:

**artifactmetadata**
- `objectid` (PRIMARY KEY)
- `title`
- `culture`
- `century`
- `department`
- `division`
- `classification`
- `totalpageviews`
- `totaluniquepageviews`

**artifactmedia**
- `mediaid` (PRIMARY KEY)
- `objectid` (FOREIGN KEY)
- `baseimageurl`
- `mediatype`
- `format`
- `height`
- `width`

**artifactcolors**
- `colorid` (AUTO_INCREMENT PRIMARY KEY)
- `objectid` (FOREIGN KEY)
- `color`
- `percentage`
- `hue`
- `saturation`

## ETL Pipeline Usage

### Extract Data from API

```python
import requests
import os

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts with pagination"""
    params = {
        'apikey': API_KEY,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(BASE_URL, params=params)
    response.raise_for_status()
    return response.json()

# Fetch first page
data = fetch_artifacts(page=1, size=100)
artifacts = data.get('records', [])
total_records = data.get('info', {}).get('totalrecords', 0)

print(f"Total artifacts available: {total_records}")
print(f"Fetched: {len(artifacts)} artifacts")
```

### Transform and Load Data

```python
import pandas as pd
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

def transform_metadata(artifacts):
    """Transform artifact data to metadata DataFrame"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata_list.append({
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'division': artifact.get('division', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        })
    
    return pd.DataFrame(metadata_list)

def transform_media(artifacts):
    """Transform media data to DataFrame"""
    media_list = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        images = artifact.get('images', [])
        
        for img in images:
            media_list.append({
                'mediaid': img.get('imageid'),
                'objectid': objectid,
                'baseimageurl': img.get('baseimageurl'),
                'mediatype': img.get('mediatype', 'Image'),
                'format': img.get('format'),
                'height': img.get('height', 0),
                'width': img.get('width', 0)
            })
    
    return pd.DataFrame(media_list)

def transform_colors(artifacts):
    """Transform color data to DataFrame"""
    color_list = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_list.append({
                'objectid': objectid,
                'color': color.get('color'),
                'percentage': color.get('percent', 0),
                'hue': color.get('hue'),
                'saturation': color.get('saturation')
            })
    
    return pd.DataFrame(color_list)

def load_to_database(df, table_name, connection):
    """Load DataFrame to MySQL table"""
    cursor = connection.cursor()
    
    # Create insert query
    cols = ','.join(df.columns)
    placeholders = ','.join(['%s'] * len(df.columns))
    query = f"INSERT INTO {table_name} ({cols}) VALUES ({placeholders})"
    
    # Batch insert
    data_tuples = [tuple(row) for row in df.values]
    cursor.executemany(query, data_tuples)
    connection.commit()
    
    print(f"Inserted {cursor.rowcount} rows into {table_name}")
    cursor.close()
```

### Complete ETL Workflow

```python
def run_etl_pipeline(num_pages=5, page_size=100):
    """Run complete ETL pipeline"""
    connection = create_database_connection()
    if not connection:
        return
    
    all_artifacts = []
    
    # Extract
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(page=page, size=page_size)
        artifacts = data.get('records', [])
        all_artifacts.extend(artifacts)
    
    print(f"Total artifacts extracted: {len(all_artifacts)}")
    
    # Transform
    metadata_df = transform_metadata(all_artifacts)
    media_df = transform_media(all_artifacts)
    colors_df = transform_colors(all_artifacts)
    
    # Load
    load_to_database(metadata_df, 'artifactmetadata', connection)
    load_to_database(media_df, 'artifactmedia', connection)
    load_to_database(colors_df, 'artifactcolors', connection)
    
    connection.close()
    print("ETL pipeline completed successfully!")

# Run the pipeline
run_etl_pipeline(num_pages=10, page_size=100)
```

## SQL Analytics Queries

### Common Analytical Queries

```python
# Query 1: Artifacts by culture
query_culture = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10;
"""

# Query 2: Artifacts by century
query_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL AND century != 'Unknown'
GROUP BY century
ORDER BY count DESC;
"""

# Query 3: Average pageviews by department
query_pageviews = """
SELECT department, 
       AVG(totalpageviews) as avg_views,
       COUNT(*) as artifact_count
FROM artifactmetadata
GROUP BY department
ORDER BY avg_views DESC;
"""

# Query 4: Most common colors
query_colors = """
SELECT color, COUNT(*) as frequency,
       AVG(percentage) as avg_percentage
FROM artifactcolors
GROUP BY color
ORDER BY frequency DESC
LIMIT 15;
"""

# Query 5: Media format distribution
query_media = """
SELECT format, mediatype, COUNT(*) as count
FROM artifactmedia
GROUP BY format, mediatype
ORDER BY count DESC;
"""

# Query 6: Artifacts with most images
query_most_images = """
SELECT m.objectid, m.title, COUNT(am.mediaid) as image_count
FROM artifactmetadata m
JOIN artifactmedia am ON m.objectid = am.objectid
GROUP BY m.objectid, m.title
ORDER BY image_count DESC
LIMIT 10;
"""

def execute_query(query, connection):
    """Execute SQL query and return DataFrame"""
    return pd.read_sql(query, connection)
```

## Streamlit Dashboard

### Basic App Structure

```python
import streamlit as st
import plotly.express as px

# Page configuration
st.set_page_config(
    page_title="Harvard Art Museums Analytics",
    page_icon="🎨",
    layout="wide"
)

st.title("🎨 Harvard Art Museums Analytics Dashboard")
st.markdown("---")

# Sidebar for query selection
st.sidebar.title("Analytics Queries")
query_options = {
    "Artifacts by Culture": query_culture,
    "Artifacts by Century": query_century,
    "Pageviews by Department": query_pageviews,
    "Color Distribution": query_colors,
    "Media Format Analysis": query_media,
    "Most Photographed Artifacts": query_most_images
}

selected_query = st.sidebar.selectbox(
    "Select Analysis",
    options=list(query_options.keys())
)

# Execute query
if st.sidebar.button("Run Analysis"):
    connection = create_database_connection()
    
    if connection:
        query = query_options[selected_query]
        df_result = execute_query(query, connection)
        
        # Display results
        st.subheader(f"Results: {selected_query}")
        st.dataframe(df_result, use_container_width=True)
        
        # Visualization
        if len(df_result.columns) >= 2:
            x_col = df_result.columns[0]
            y_col = df_result.columns[1]
            
            fig = px.bar(
                df_result,
                x=x_col,
                y=y_col,
                title=f"{selected_query} Visualization",
                labels={x_col: x_col.replace('_', ' ').title(),
                       y_col: y_col.replace('_', ' ').title()}
            )
            st.plotly_chart(fig, use_container_width=True)
        
        connection.close()
```

### ETL Control Panel

```python
# ETL section
st.sidebar.markdown("---")
st.sidebar.title("ETL Pipeline")

num_pages = st.sidebar.number_input("Pages to fetch", min_value=1, max_value=50, value=5)
page_size = st.sidebar.number_input("Records per page", min_value=10, max_value=100, value=100)

if st.sidebar.button("Run ETL Pipeline"):
    with st.spinner("Running ETL pipeline..."):
        try:
            run_etl_pipeline(num_pages=int(num_pages), page_size=int(page_size))
            st.success(f"Successfully loaded {num_pages * page_size} artifacts!")
        except Exception as e:
            st.error(f"ETL Error: {str(e)}")
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Configuration

Create a `.env` file:

```env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_mysql_host
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

Load in Python:

```python
from dotenv import load_dotenv
load_dotenv()
```

## Common Patterns

### Rate Limiting

```python
import time

def fetch_with_rate_limit(page, size, delay=0.5):
    """Fetch with rate limiting"""
    data = fetch_artifacts(page, size)
    time.sleep(delay)  # Respect API limits
    return data
```

### Error Handling

```python
def safe_etl_pipeline(num_pages, page_size):
    """ETL with error handling"""
    try:
        connection = create_database_connection()
        for page in range(1, num_pages + 1):
            try:
                data = fetch_artifacts(page, page_size)
                # Process data...
            except requests.RequestException as e:
                print(f"API error on page {page}: {e}")
                continue
    except Error as db_error:
        print(f"Database error: {db_error}")
    finally:
        if connection and connection.is_connected():
            connection.close()
```

## Troubleshooting

**API Key Issues**: Ensure `HARVARD_API_KEY` is set correctly. Test with:
```python
response = requests.get(f"{BASE_URL}?apikey={API_KEY}&size=1")
print(response.status_code)  # Should be 200
```

**Database Connection Errors**: Verify credentials and network access. Check with:
```python
connection = mysql.connector.connect(host=DB_HOST, user=DB_USER, password=DB_PASSWORD)
print(connection.is_connected())
```

**Missing Data**: Some artifacts may lack images/colors. Always use `.get()` with defaults:
```python
color = artifact.get('colors', [])  # Returns [] if missing
```

**Streamlit Performance**: For large datasets, use pagination or caching:
```python
@st.cache_data
def load_query_results(query):
    connection = create_database_connection()
    df = pd.read_sql(query, connection)
    connection.close()
    return df
```
