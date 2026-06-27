---
name: harvard-artifacts-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using Harvard Art Museums API with Streamlit
triggers:
  - "build an ETL pipeline for Harvard Art Museums data"
  - "create analytics dashboard for art museum artifacts"
  - "extract and load Harvard museum data to SQL"
  - "visualize art museum data with Streamlit"
  - "set up Harvard Art Museums API integration"
  - "build data engineering pipeline for museum collections"
  - "create interactive art data analytics app"
  - "query and visualize museum artifact metadata"
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

An end-to-end data engineering and analytics application that demonstrates real-world ETL pipelines using the Harvard Art Museums API. This project extracts artifact metadata, transforms nested JSON into relational structures, loads data into SQL databases, and provides interactive analytics dashboards using Streamlit.

## What It Does

- **API Integration**: Connects to Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts artifact metadata, media details, and color data; transforms nested JSON into relational tables
- **SQL Storage**: Stores data in MySQL/TiDB Cloud with proper schema design and foreign key relationships
- **Analytics**: Executes 20+ predefined SQL queries for insights on artifacts, cultures, media, and colors
- **Visualization**: Interactive dashboards with Plotly charts and real-time query results

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
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

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Register for a free API key
3. Add to `.env` file

### Database Setup

The application supports MySQL and TiDB Cloud. Schema is auto-created on first run:

```sql
-- Main tables created by the ETL pipeline
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
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500)
);

CREATE TABLE artifactmedia (
    artifact_id INT,
    media_id INT,
    baseimageurl VARCHAR(500),
    format VARCHAR(50),
    height INT,
    width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
streamlit run app.py
```

Access the dashboard at `http://localhost:8501`

## Core Components

### 1. API Data Collection

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
    url = f"https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    response.raise_for_status()
    return response.json()

# Collect multiple pages
def collect_artifacts(num_pages=5):
    """Collect artifacts across multiple pages"""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        data = fetch_artifacts(page=page)
        artifacts = data.get('records', [])
        all_artifacts.extend(artifacts)
        print(f"Collected page {page}: {len(artifacts)} artifacts")
    
    return all_artifacts
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

def transform_artifacts(raw_data):
    """Transform nested JSON into relational dataframes"""
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions')
        }
        metadata_records.append(metadata)
        
        # Extract media
        for media in artifact.get('images', []):
            media_record = {
                'artifact_id': artifact.get('id'),
                'media_id': media.get('imageid'),
                'baseimageurl': media.get('baseimageurl'),
                'format': media.get('format'),
                'height': media.get('height'),
                'width': media.get('width')
            }
            media_records.append(media_record)
        
        # Extract colors
        for color in artifact.get('colors', []):
            color_record = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            color_records.append(color_record)
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )

def load_to_database(df_metadata, df_media, df_colors):
    """Load dataframes to SQL database"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=os.getenv('DB_PORT'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    cursor = connection.cursor()
    
    # Insert metadata (batch)
    metadata_tuples = [tuple(x) for x in df_metadata.to_numpy()]
    insert_metadata = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, 
         department, division, dated, technique, medium, dimensions)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    cursor.executemany(insert_metadata, metadata_tuples)
    
    # Insert media
    media_tuples = [tuple(x) for x in df_media.to_numpy()]
    insert_media = """
        INSERT INTO artifactmedia 
        (artifact_id, media_id, baseimageurl, format, height, width)
        VALUES (%s, %s, %s, %s, %s, %s)
    """
    cursor.executemany(insert_media, media_tuples)
    
    # Insert colors
    color_tuples = [tuple(x) for x in df_colors.to_numpy()]
    insert_colors = """
        INSERT INTO artifactcolors 
        (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
    """
    cursor.executemany(insert_colors, color_tuples)
    
    connection.commit()
    cursor.close()
    connection.close()
```

### 3. Analytics Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
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
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as usage_count
        FROM artifactcolors
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifacts
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifacts DESC
    """,
    
    "Media Availability": """
        SELECT 
            COUNT(DISTINCT artifact_id) as artifacts_with_media,
            COUNT(*) as total_media_files
        FROM artifactmedia
    """,
    
    "Average Color Diversity per Artifact": """
        SELECT AVG(color_count) as avg_colors
        FROM (
            SELECT artifact_id, COUNT(DISTINCT color) as color_count
            FROM artifactcolors
            GROUP BY artifact_id
        ) as subquery
    """
}

def execute_query(query_name):
    """Execute analytics query and return results"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    query = ANALYTICS_QUERIES[query_name]
    df = pd.read_sql(query, connection)
    connection.close()
    
    return df
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Art Museums Analytics")
    
    # Sidebar for configuration
    st.sidebar.header("Configuration")
    
    # Data Collection Section
    st.header("1. Data Collection")
    num_pages = st.number_input("Number of pages to collect", 1, 20, 5)
    
    if st.button("Collect Data from API"):
        with st.spinner("Fetching artifacts..."):
            artifacts = collect_artifacts(num_pages)
            st.success(f"Collected {len(artifacts)} artifacts")
            
            # ETL Process
            st.info("Transforming data...")
            df_meta, df_media, df_colors = transform_artifacts(artifacts)
            
            st.info("Loading to database...")
            load_to_database(df_meta, df_media, df_colors)
            
            st.success("ETL Complete!")
    
    # Analytics Section
    st.header("2. Analytics Dashboard")
    
    query_selection = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Run Query"):
        with st.spinner("Executing query..."):
            results = execute_query(query_selection)
            
            st.dataframe(results)
            
            # Auto-generate visualization
            if len(results.columns) == 2:
                fig = px.bar(
                    results,
                    x=results.columns[0],
                    y=results.columns[1],
                    title=query_selection
                )
                st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Rate Limiting API Calls

```python
import time

def fetch_with_rate_limit(page, delay=0.5):
    """Fetch with rate limiting to respect API limits"""
    data = fetch_artifacts(page=page)
    time.sleep(delay)  # 0.5 second delay between requests
    return data
```

### Error Handling in ETL

```python
def safe_transform(raw_data):
    """Transform with error handling"""
    try:
        return transform_artifacts(raw_data)
    except Exception as e:
        print(f"Transform error: {e}")
        # Return empty dataframes
        return (
            pd.DataFrame(),
            pd.DataFrame(),
            pd.DataFrame()
        )
```

### Incremental Data Loading

```python
def get_max_artifact_id():
    """Get highest artifact ID already in database"""
    connection = mysql.connector.connect(...)
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    max_id = cursor.fetchone()[0] or 0
    connection.close()
    return max_id

def fetch_new_artifacts_only():
    """Fetch only artifacts newer than what we have"""
    max_id = get_max_artifact_id()
    # Filter API results by ID > max_id
    # Implementation depends on API capabilities
```

## Troubleshooting

### API Key Issues

```python
# Verify API key is loaded
api_key = os.getenv('HARVARD_API_KEY')
if not api_key:
    raise ValueError("HARVARD_API_KEY not found in environment variables")
```

### Database Connection Errors

```python
try:
    connection = mysql.connector.connect(...)
except Error as e:
    print(f"Database connection failed: {e}")
    print("Check DB_HOST, DB_USER, DB_PASSWORD in .env file")
```

### Empty Results

```python
# Check if API returned data
data = fetch_artifacts()
if not data.get('records'):
    print("No records returned. Check API key and parameters")
```

### Memory Issues with Large Datasets

```python
# Process in chunks
def process_in_chunks(artifacts, chunk_size=100):
    """Process large datasets in chunks"""
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i+chunk_size]
        df_meta, df_media, df_colors = transform_artifacts(chunk)
        load_to_database(df_meta, df_media, df_colors)
        print(f"Processed chunk {i//chunk_size + 1}")
```

## Advanced Usage

### Custom Query Builder

```python
def custom_query(where_clause=""):
    """Execute custom analytics query"""
    base_query = f"""
        SELECT culture, century, COUNT(*) as count
        FROM artifactmetadata
        {where_clause}
        GROUP BY culture, century
        ORDER BY count DESC
    """
    return execute_raw_query(base_query)

# Usage
results = custom_query("WHERE department = 'Asian Art'")
```

### Export Analytics Results

```python
def export_results(df, filename):
    """Export query results to CSV"""
    df.to_csv(f"exports/{filename}.csv", index=False)
    print(f"Exported to exports/{filename}.csv")
```

This skill provides complete coverage of building ETL pipelines and analytics dashboards for museum data using modern Python data engineering tools.
