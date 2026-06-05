---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API data with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - show me how to create data analytics dashboard with museum artifacts
  - how to extract and load Harvard museum data into SQL
  - build a Streamlit app for art collection analytics
  - query and visualize Harvard Art Museums data
  - create an end-to-end data engineering pipeline for museum artifacts
  - how to process Harvard API data with pandas and SQL
  - build interactive analytics for art museum collections
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL analytics, and interactive visualization using Streamlit.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides:

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads nested JSON data into relational SQL tables
- **SQL Analytics**: Predefined analytical queries for insights on artifacts, media, colors, and cultural patterns
- **Interactive Dashboards**: Streamlit-based UI with Plotly visualizations
- **Database Design**: Normalized schema with `artifactmetadata`, `artifactmedia`, and `artifactcolors` tables

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required Dependencies**:
```txt
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

### 2. Environment Variables
Create a `.env` file in the project root:

```bash
# Harvard API Configuration
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_db_host
DB_PORT=4000
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### 3. Database Setup
```sql
-- Create database
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

-- Create tables
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
    url VARCHAR(500)
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    media_url VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core API Usage

### Fetching Data from Harvard API

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
        return data['records'], data['info']
    else:
        raise Exception(f"API Error: {response.status_code}")

# Usage
artifacts, info = fetch_artifacts(page=1, size=50)
print(f"Total artifacts: {info['totalrecords']}")
print(f"Pages: {info['pages']}")
```

### Pagination Handling

```python
def fetch_all_artifacts(max_pages=10):
    """Fetch multiple pages of artifacts"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        try:
            artifacts, info = fetch_artifacts(page=page)
            all_artifacts.extend(artifacts)
            print(f"Fetched page {page}/{info['pages']}")
            
            if page >= info['pages']:
                break
                
        except Exception as e:
            print(f"Error on page {page}: {e}")
            break
    
    return all_artifacts
```

## ETL Pipeline Implementation

### Extract and Transform

```python
import pandas as pd

def extract_metadata(artifacts):
    """Extract artifact metadata"""
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'period': artifact.get('period'),
            'technique': artifact.get('technique'),
            'url': artifact.get('url')
        })
    
    return pd.DataFrame(metadata)

def extract_media(artifacts):
    """Extract media information"""
    media = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        
        if 'images' in artifact and artifact['images']:
            for image in artifact['images']:
                media.append({
                    'artifact_id': artifact_id,
                    'media_type': 'image',
                    'media_url': image.get('baseimageurl')
                })
        
        if 'primaryimageurl' in artifact:
            media.append({
                'artifact_id': artifact_id,
                'media_type': 'primary_image',
                'media_url': artifact.get('primaryimageurl')
            })
    
    return pd.DataFrame(media)

def extract_colors(artifacts):
    """Extract color information"""
    colors = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        
        if 'colors' in artifact and artifact['colors']:
            for color in artifact['colors']:
                colors.append({
                    'artifact_id': artifact_id,
                    'color_hex': color.get('hex'),
                    'color_percentage': color.get('percent')
                })
    
    return pd.DataFrame(colors)
```

### Load to Database

```python
import mysql.connector
from mysql.connector import Error

def create_connection():
    """Create database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_metadata(df, connection):
    """Load metadata into database"""
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, dated, period, technique, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture), century=VALUES(century)
    """
    
    records = df.to_records(index=False)
    data = [tuple(record) for record in records]
    
    cursor.executemany(insert_query, data)
    connection.commit()
    print(f"Inserted {cursor.rowcount} metadata records")
    cursor.close()

def load_media(df, connection):
    """Load media data"""
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia (artifact_id, media_type, media_url)
        VALUES (%s, %s, %s)
    """
    
    records = df.to_records(index=False)
    data = [tuple(record) for record in records]
    
    cursor.executemany(insert_query, data)
    connection.commit()
    print(f"Inserted {cursor.rowcount} media records")
    cursor.close()

def load_colors(df, connection):
    """Load color data"""
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactcolors (artifact_id, color_hex, color_percentage)
        VALUES (%s, %s, %s)
    """
    
    records = df.to_records(index=False)
    data = [tuple(record) for record in records]
    
    cursor.executemany(insert_query, data)
    connection.commit()
    print(f"Inserted {cursor.rowcount} color records")
    cursor.close()
```

### Complete ETL Pipeline

```python
def run_etl_pipeline(max_pages=5):
    """Run complete ETL pipeline"""
    print("Starting ETL Pipeline...")
    
    # Extract
    print("Extracting data from API...")
    artifacts = fetch_all_artifacts(max_pages=max_pages)
    
    # Transform
    print("Transforming data...")
    metadata_df = extract_metadata(artifacts)
    media_df = extract_media(artifacts)
    colors_df = extract_colors(artifacts)
    
    # Load
    print("Loading data to database...")
    connection = create_connection()
    
    if connection:
        load_metadata(metadata_df, connection)
        load_media(media_df, connection)
        load_colors(colors_df, connection)
        connection.close()
        print("ETL Pipeline completed successfully!")
    else:
        print("Failed to connect to database")

# Run the pipeline
if __name__ == "__main__":
    run_etl_pipeline(max_pages=10)
```

## SQL Analytics Queries

### Common Analytical Queries

```python
def execute_query(query, connection):
    """Execute SQL query and return DataFrame"""
    return pd.read_sql(query, connection)

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
query_centuries = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC;
"""

# Department distribution
query_departments = """
SELECT department, COUNT(*) as total_artifacts
FROM artifactmetadata
GROUP BY department
ORDER BY total_artifacts DESC;
"""

# Media availability
query_media = """
SELECT 
    COUNT(DISTINCT artifact_id) as artifacts_with_media,
    COUNT(*) as total_media_items,
    media_type
FROM artifactmedia
GROUP BY media_type;
"""

# Color analysis
query_colors = """
SELECT color_hex, COUNT(*) as usage_count, AVG(color_percentage) as avg_percentage
FROM artifactcolors
GROUP BY color_hex
ORDER BY usage_count DESC
LIMIT 20;
"""

# Usage
connection = create_connection()
if connection:
    cultures_df = execute_query(query_cultures, connection)
    print(cultures_df)
    connection.close()
```

## Streamlit Dashboard

### Basic Streamlit App Structure

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

def main():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Select Analysis",
        ["Overview", "Culture Analysis", "Century Distribution", 
         "Color Patterns", "Media Analysis"]
    )
    
    # Database connection
    connection = create_connection()
    
    if not connection:
        st.error("Failed to connect to database")
        return
    
    if page == "Culture Analysis":
        show_culture_analysis(connection)
    elif page == "Century Distribution":
        show_century_distribution(connection)
    elif page == "Color Patterns":
        show_color_patterns(connection)
    elif page == "Media Analysis":
        show_media_analysis(connection)
    
    connection.close()

def show_culture_analysis(connection):
    st.header("Artifact Distribution by Culture")
    
    query = """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 15;
    """
    
    df = pd.read_sql(query, connection)
    
    # Display data table
    st.dataframe(df)
    
    # Create visualization
    fig = px.bar(df, x='culture', y='count', 
                 title='Top 15 Cultures by Artifact Count',
                 labels={'count': 'Number of Artifacts', 'culture': 'Culture'})
    st.plotly_chart(fig, use_container_width=True)

def show_color_patterns(connection):
    st.header("Color Usage Patterns")
    
    query = """
        SELECT color_hex, COUNT(*) as frequency, 
               AVG(color_percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color_hex
        ORDER BY frequency DESC
        LIMIT 20;
    """
    
    df = pd.read_sql(query, connection)
    
    # Add color swatches
    st.dataframe(df)
    
    fig = px.scatter(df, x='frequency', y='avg_percentage', 
                     color='color_hex',
                     title='Color Frequency vs Average Percentage',
                     labels={'frequency': 'Usage Frequency', 
                            'avg_percentage': 'Average Percentage in Artifacts'})
    st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

### Interactive Query Builder

```python
def query_builder_page():
    st.header("Custom Query Builder")
    
    # Predefined queries
    queries = {
        "Artifacts by Classification": """
            SELECT classification, COUNT(*) as count
            FROM artifactmetadata
            GROUP BY classification
            ORDER BY count DESC;
        """,
        "Artifacts with Images": """
            SELECT am.*, COUNT(m.id) as image_count
            FROM artifactmetadata am
            LEFT JOIN artifactmedia m ON am.id = m.artifact_id
            GROUP BY am.id
            HAVING image_count > 0;
        """,
        "Most Common Techniques": """
            SELECT technique, COUNT(*) as count
            FROM artifactmetadata
            WHERE technique IS NOT NULL
            GROUP BY technique
            ORDER BY count DESC
            LIMIT 10;
        """
    }
    
    selected_query = st.selectbox("Select a query", list(queries.keys()))
    
    st.code(queries[selected_query], language='sql')
    
    if st.button("Execute Query"):
        connection = create_connection()
        if connection:
            df = pd.read_sql(queries[selected_query], connection)
            st.dataframe(df)
            connection.close()
```

## Common Patterns

### Rate Limiting for API Calls

```python
import time

def fetch_with_rate_limit(page, size=100, delay=0.5):
    """Fetch data with rate limiting"""
    artifacts, info = fetch_artifacts(page, size)
    time.sleep(delay)  # Respect API rate limits
    return artifacts, info
```

### Batch Processing

```python
def process_in_batches(artifacts, batch_size=100):
    """Process artifacts in batches"""
    connection = create_connection()
    
    for i in range(0, len(artifacts), batch_size):
        batch = artifacts[i:i+batch_size]
        
        metadata_df = extract_metadata(batch)
        media_df = extract_media(batch)
        colors_df = extract_colors(batch)
        
        load_metadata(metadata_df, connection)
        load_media(media_df, connection)
        load_colors(colors_df, connection)
        
        print(f"Processed batch {i//batch_size + 1}")
    
    connection.close()
```

### Error Handling

```python
def safe_etl_pipeline(max_pages=5):
    """ETL pipeline with error handling"""
    try:
        artifacts = []
        
        for page in range(1, max_pages + 1):
            try:
                batch, info = fetch_artifacts(page)
                artifacts.extend(batch)
            except Exception as e:
                st.warning(f"Failed to fetch page {page}: {e}")
                continue
        
        if artifacts:
            connection = create_connection()
            if connection:
                metadata_df = extract_metadata(artifacts)
                load_metadata(metadata_df, connection)
                connection.close()
                st.success(f"Loaded {len(artifacts)} artifacts")
        else:
            st.error("No artifacts fetched")
            
    except Exception as e:
        st.error(f"ETL Pipeline failed: {e}")
```

## Troubleshooting

### API Key Issues
- Verify API key is valid at Harvard Art Museums API portal
- Check `.env` file is in project root
- Ensure `python-dotenv` is installed: `pip install python-dotenv`

### Database Connection Errors
```python
# Test connection
try:
    connection = create_connection()
    if connection.is_connected():
        print("Database connected successfully")
        connection.close()
except Error as e:
    print(f"Connection error: {e}")
```

### Missing Data Handling
```python
def safe_extract(artifact, key, default=None):
    """Safely extract data with default values"""
    return artifact.get(key, default)

# Use in extraction
metadata.append({
    'id': safe_extract(artifact, 'id'),
    'title': safe_extract(artifact, 'title', 'Untitled'),
    'culture': safe_extract(artifact, 'culture', 'Unknown')
})
```

### Streamlit Cache for Performance
```python
@st.cache_data(ttl=3600)
def load_cached_data(query):
    """Cache query results for 1 hour"""
    connection = create_connection()
    df = pd.read_sql(query, connection)
    connection.close()
    return df
```

### Memory Issues with Large Datasets
```python
# Use chunked reading
def load_large_dataset(query, chunksize=1000):
    connection = create_connection()
    chunks = []
    
    for chunk in pd.read_sql(query, connection, chunksize=chunksize):
        chunks.append(chunk)
    
    connection.close()
    return pd.concat(chunks, ignore_index=True)
```

This skill provides everything needed to build, deploy, and extend the Harvard Artifacts ETL analytics application with proper data engineering practices.
