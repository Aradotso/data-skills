---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for the Harvard Art Museums API
  - create a data analytics app with Harvard museum artifacts
  - set up a Streamlit dashboard for art collection data
  - extract and transform Harvard Art Museums data into SQL
  - build an end-to-end data engineering project with museum API
  - analyze Harvard artifact data with SQL queries
  - visualize art museum collection data with Plotly
  - create a data pipeline from Harvard Art Museums to database
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates how to build a complete data engineering and analytics application using the Harvard Art Museums API. It implements ETL pipelines to extract artifact data, transform it into relational structures, load it into SQL databases, and visualize insights through an interactive Streamlit dashboard.

## What It Does

The Harvard Art Museums ETL Analytics app:

- **Extracts** artifact data from the Harvard Art Museums API with pagination handling
- **Transforms** nested JSON responses into normalized relational tables
- **Loads** data into MySQL/TiDB databases with proper schema design
- **Analyzes** collections using 20+ predefined SQL queries
- **Visualizes** results through interactive Plotly charts in a Streamlit interface

## Installation

### Prerequisites

- Python 3.8+
- MySQL or TiDB Cloud account
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
export DB_HOST="your_database_host"
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="your_database_name"
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

## Configuration

### Database Connection

Create a `.env` file or configure environment variables:

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

### API Configuration

```python
import os

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = 'https://api.harvardartmuseums.org/object'
```

## Database Schema

The application uses three main tables:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    period VARCHAR(200),
    url VARCHAR(500)
);

-- Artifact media table
CREATE TABLE artifactmedia (
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    has_images BOOLEAN,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import pandas as pd

def fetch_artifacts(api_key, num_records=100):
    """
    Fetch artifact data from Harvard Art Museums API
    """
    artifacts = []
    page = 1
    size = 100  # Max per request
    
    while len(artifacts) < num_records:
        params = {
            'apikey': api_key,
            'size': min(size, num_records - len(artifacts)),
            'page': page
        }
        
        response = requests.get(BASE_URL, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            if not records:
                break
            artifacts.extend(records)
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_records]
```

### Transform: Process JSON to DataFrames

```python
def transform_artifacts(artifacts):
    """
    Transform raw API data into structured DataFrames
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'period': artifact.get('period'),
            'url': artifact.get('url')
        }
        metadata_list.append(metadata)
        
        # Extract media information
        media = {
            'artifact_id': artifact.get('id'),
            'baseimageurl': artifact.get('baseimageurl'),
            'primaryimageurl': artifact.get('primaryimageurl'),
            'has_images': 1 if artifact.get('images') else 0
        }
        media_list.append(media)
        
        # Extract color information
        colors = artifact.get('colors', [])
        for color in colors:
            color_entry = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'percent': color.get('percent')
            }
            colors_list.append(color_entry)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load: Insert Data into Database

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(metadata_df, media_df, colors_df, db_config):
    """
    Load transformed data into MySQL database
    """
    try:
        connection = mysql.connector.connect(**db_config)
        cursor = connection.cursor()
        
        # Insert metadata
        for _, row in metadata_df.iterrows():
            query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, century, dated, classification, 
             department, division, technique, medium, period, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
            """
            cursor.execute(query, tuple(row))
        
        # Insert media
        for _, row in media_df.iterrows():
            query = """
            INSERT INTO artifactmedia 
            (artifact_id, baseimageurl, primaryimageurl, has_images)
            VALUES (%s, %s, %s, %s)
            """
            cursor.execute(query, tuple(row))
        
        # Insert colors
        for _, row in colors_df.iterrows():
            query = """
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, percent)
            VALUES (%s, %s, %s, %s)
            """
            cursor.execute(query, tuple(row))
        
        connection.commit()
        print(f"Successfully loaded {len(metadata_df)} artifacts")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

## Running the Full ETL Pipeline

```python
def run_etl_pipeline(api_key, db_config, num_records=100):
    """
    Execute complete ETL pipeline
    """
    # Extract
    print("Extracting data from API...")
    artifacts = fetch_artifacts(api_key, num_records)
    
    # Transform
    print("Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts(artifacts)
    
    # Load
    print("Loading data to database...")
    load_to_database(metadata_df, media_df, colors_df, db_config)
    
    print("ETL pipeline completed successfully!")

# Execute
if __name__ == "__main__":
    run_etl_pipeline(API_KEY, DB_CONFIG, num_records=500)
```

## Analytics Queries

### Sample SQL Queries

```python
# Query 1: Top 10 cultures by artifact count
QUERY_CULTURE_DISTRIBUTION = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10;
"""

# Query 2: Century-wise artifact distribution
QUERY_CENTURY_DISTRIBUTION = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC
LIMIT 15;
"""

# Query 3: Artifacts with images vs without
QUERY_IMAGE_AVAILABILITY = """
SELECT 
    CASE WHEN has_images = 1 THEN 'With Images' ELSE 'Without Images' END as category,
    COUNT(*) as count
FROM artifactmedia
GROUP BY has_images;
"""

# Query 4: Top color usage
QUERY_COLOR_DISTRIBUTION = """
SELECT color, COUNT(*) as frequency
FROM artifactcolors
WHERE color IS NOT NULL
GROUP BY color
ORDER BY frequency DESC
LIMIT 10;
"""

# Query 5: Department-wise artifact count
QUERY_DEPARTMENT_STATS = """
SELECT department, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY artifact_count DESC;
"""
```

### Execute Queries

```python
def execute_query(query, db_config):
    """
    Execute SQL query and return results as DataFrame
    """
    try:
        connection = mysql.connector.connect(**db_config)
        df = pd.read_sql(query, connection)
        connection.close()
        return df
    except Error as e:
        print(f"Query error: {e}")
        return pd.DataFrame()

# Usage
results = execute_query(QUERY_CULTURE_DISTRIBUTION, DB_CONFIG)
print(results)
```

## Streamlit Dashboard

### Basic App Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Art Analytics", layout="wide")
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar for ETL controls
    with st.sidebar:
        st.header("Data Collection")
        api_key = st.text_input("API Key", type="password", 
                                value=os.getenv('HARVARD_API_KEY', ''))
        num_records = st.number_input("Records to Fetch", 
                                      min_value=10, max_value=1000, 
                                      value=100, step=10)
        
        if st.button("Run ETL Pipeline"):
            with st.spinner("Running ETL..."):
                run_etl_pipeline(api_key, DB_CONFIG, num_records)
                st.success("ETL completed!")
    
    # Main content area
    st.header("📊 Analytics")
    
    # Query selection
    query_options = {
        "Culture Distribution": QUERY_CULTURE_DISTRIBUTION,
        "Century Distribution": QUERY_CENTURY_DISTRIBUTION,
        "Image Availability": QUERY_IMAGE_AVAILABILITY,
        "Color Distribution": QUERY_COLOR_DISTRIBUTION,
        "Department Statistics": QUERY_DEPARTMENT_STATS
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
    
    if st.button("Run Query"):
        query = query_options[selected_query]
        df = execute_query(query, DB_CONFIG)
        
        if not df.empty:
            st.subheader("Results")
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df.columns) == 2:
                fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                           title=selected_query)
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

### Advanced Visualization

```python
def create_culture_chart(df):
    """
    Create interactive bar chart for culture distribution
    """
    fig = px.bar(
        df,
        x='culture',
        y='artifact_count',
        title='Top 10 Cultures by Artifact Count',
        labels={'culture': 'Culture', 'artifact_count': 'Number of Artifacts'},
        color='artifact_count',
        color_continuous_scale='Viridis'
    )
    fig.update_layout(xaxis_tickangle=-45)
    return fig

def create_color_pie_chart(df):
    """
    Create pie chart for color distribution
    """
    fig = px.pie(
        df,
        values='frequency',
        names='color',
        title='Color Usage Distribution',
        hole=0.3
    )
    return fig
```

## Common Patterns

### Batch Processing Large Datasets

```python
def batch_insert(df, table_name, db_config, batch_size=1000):
    """
    Insert data in batches for better performance
    """
    connection = mysql.connector.connect(**db_config)
    cursor = connection.cursor()
    
    for i in range(0, len(df), batch_size):
        batch = df.iloc[i:i+batch_size]
        # Insert batch
        connection.commit()
    
    cursor.close()
    connection.close()
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_api_request(url, params, max_retries=3):
    """
    API request with retry logic
    """
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            logger.warning(f"Attempt {attempt + 1} failed: {e}")
            if attempt == max_retries - 1:
                raise
    return None
```

## Troubleshooting

### API Rate Limiting

If you encounter rate limiting errors:

```python
import time

def fetch_with_rate_limit(api_key, num_records, delay=1):
    """
    Fetch data with rate limiting
    """
    artifacts = []
    page = 1
    
    while len(artifacts) < num_records:
        # Fetch data
        time.sleep(delay)  # Wait between requests
        # ... rest of logic
    
    return artifacts
```

### Database Connection Issues

```python
def test_db_connection(db_config):
    """
    Test database connectivity
    """
    try:
        connection = mysql.connector.connect(**db_config)
        if connection.is_connected():
            print("Database connection successful")
            connection.close()
            return True
    except Error as e:
        print(f"Connection failed: {e}")
        return False
```

### Empty API Results

```python
def validate_api_response(data):
    """
    Validate API response has records
    """
    if not data or 'records' not in data:
        raise ValueError("Invalid API response")
    
    records = data.get('records', [])
    if not records:
        raise ValueError("No records returned from API")
    
    return records
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# Run ETL pipeline from command line
python etl_pipeline.py

# Run with custom parameters
python etl_pipeline.py --records 500 --batch-size 100
```
