---
name: harvard-artifacts-collection-data-engineering-analytics
description: End-to-end data engineering and analytics application for Harvard Art Museums API with ETL pipelines, SQL storage, and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data analytics app with Harvard artifacts
  - set up artifact collection from Harvard museums API
  - visualize Harvard art museum data with SQL queries
  - build a Streamlit dashboard for museum artifacts
  - implement data engineering pipeline for art collections
  - query and analyze Harvard Art Museums API data
  - create interactive museum data visualization
---

# Harvard Artifacts Collection Data Engineering Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides a complete data engineering pipeline for the Harvard Art Museums API, featuring ETL operations, SQL database storage, analytical queries, and interactive Streamlit visualizations.

## What It Does

- **Extracts** artifact data from Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into normalized relational database tables
- **Loads** data into MySQL/TiDB Cloud with optimized batch inserts
- **Analyzes** artifact collections using 20+ predefined SQL queries
- **Visualizes** results through interactive Plotly charts in Streamlit

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

### API Key Setup

1. Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api
2. Create a `.env` file:

```bash
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Database Schema

The application creates three main tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    division VARCHAR(200),
    department VARCHAR(200),
    classification VARCHAR(200),
    dated VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    people TEXT,
    url VARCHAR(500)
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    base_url VARCHAR(500),
    image_id VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
streamlit run app.py
```

The Streamlit interface will open at `http://localhost:8501`

## Key API Integration Patterns

### Fetching Artifacts with Pagination

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_records=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    artifacts = []
    page = 1
    size = 100  # Max per page
    
    while len(artifacts) < num_records:
        params = {
            'apikey': api_key,
            'size': min(size, num_records - len(artifacts)),
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            artifacts.extend(data['records'])
            
            if len(data['records']) < size:
                break  # No more records
                
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_records]
```

### Extracting and Transforming Data

```python
import pandas as pd

def transform_artifacts(artifacts):
    """Transform nested JSON to relational format"""
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:200],
            'century': artifact.get('century', '')[:100],
            'division': artifact.get('division', '')[:200],
            'department': artifact.get('department', '')[:200],
            'classification': artifact.get('classification', '')[:200],
            'dated': artifact.get('dated', '')[:200],
            'technique': artifact.get('technique', '')[:500],
            'medium': artifact.get('medium', '')[:500],
            'people': str(artifact.get('people', []))[:1000],
            'url': artifact.get('url', '')[:500]
        })
        
        # Extract media
        for image in artifact.get('images', []):
            media_records.append({
                'artifact_id': artifact.get('id'),
                'media_type': 'image',
                'base_url': image.get('baseimageurl', '')[:500],
                'image_id': image.get('imageid', '')[:100]
            })
        
        # Extract colors
        for color in artifact.get('colors', []):
            color_records.append({
                'artifact_id': artifact.get('id'),
                'color_hex': color.get('hex', '')[:10],
                'color_percent': color.get('percent', 0.0)
            })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

## Database Operations

### Loading Data to MySQL

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(metadata_df, media_df, colors_df):
    """Load transformed data into MySQL database"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
        cursor = connection.cursor()
        
        # Insert metadata (batch)
        metadata_query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, century, division, department, 
             classification, dated, technique, medium, people, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        
        metadata_values = [tuple(row) for row in metadata_df.values]
        cursor.executemany(metadata_query, metadata_values)
        
        # Insert media
        media_query = """
            INSERT INTO artifactmedia 
            (artifact_id, media_type, base_url, image_id)
            VALUES (%s, %s, %s, %s)
        """
        media_values = [tuple(row) for row in media_df.values]
        cursor.executemany(media_query, media_values)
        
        # Insert colors
        colors_query = """
            INSERT INTO artifactcolors 
            (artifact_id, color_hex, color_percent)
            VALUES (%s, %s, %s)
        """
        colors_values = [tuple(row) for row in colors_df.values]
        cursor.executemany(colors_query, colors_values)
        
        connection.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

## Analytical SQL Queries

### Example Analytics Queries

```python
# Top 10 cultures by artifact count
query_1 = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL AND culture != ''
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10
"""

# Artifacts with most images
query_2 = """
SELECT m.id, m.title, COUNT(im.id) as image_count
FROM artifactmetadata m
JOIN artifactmedia im ON m.id = im.artifact_id
GROUP BY m.id, m.title
ORDER BY image_count DESC
LIMIT 10
"""

# Most common colors
query_3 = """
SELECT color_hex, COUNT(*) as usage_count, 
       AVG(color_percent) as avg_percent
FROM artifactcolors
GROUP BY color_hex
ORDER BY usage_count DESC
LIMIT 15
"""

# Century distribution
query_4 = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC
"""
```

## Streamlit Dashboard Patterns

### Creating Interactive Analytics

```python
import streamlit as st
import plotly.express as px

def run_query_and_visualize(query, chart_title):
    """Execute SQL query and create visualization"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    # Execute query
    df = pd.read_sql(query, connection)
    connection.close()
    
    # Display table
    st.dataframe(df)
    
    # Create visualization if data has 2+ columns
    if len(df.columns) >= 2:
        fig = px.bar(
            df, 
            x=df.columns[0], 
            y=df.columns[1],
            title=chart_title
        )
        st.plotly_chart(fig, use_container_width=True)
    
    return df

# Example usage in Streamlit
st.title("Harvard Art Museums Analytics")

query_options = {
    "Top Cultures": query_1,
    "Most Photographed Artifacts": query_2,
    "Popular Colors": query_3,
    "Century Distribution": query_4
}

selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
run_query_and_visualize(query_options[selected_query], selected_query)
```

## Common Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(url, params, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        response = requests.get(url, params=params)
        
        if response.status_code == 200:
            return response.json()
        elif response.status_code == 429:  # Rate limited
            wait_time = 2 ** attempt
            time.sleep(wait_time)
        else:
            break
    
    return None
```

### Database Connection Errors

```python
def get_db_connection():
    """Get database connection with error handling"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME'),
            connect_timeout=10
        )
        return connection
    except Error as e:
        st.error(f"Database connection failed: {e}")
        return None
```

### Handling Missing Data

```python
def safe_get(data, key, default=''):
    """Safely extract data with default values"""
    value = data.get(key, default)
    return value if value is not None else default
```

This skill enables AI agents to help developers build complete data engineering pipelines using the Harvard Art Museums API with proper ETL patterns, SQL analytics, and interactive visualizations.
