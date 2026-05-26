---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Streamlit, MySQL, and Plotly
triggers:
  - how do I fetch Harvard Art Museum artifacts data
  - build an ETL pipeline for Harvard museum collection
  - analyze Harvard Art Museums API with SQL
  - create artifact analytics dashboard with Streamlit
  - extract and transform museum artifact metadata
  - visualize Harvard museum data with Plotly
  - set up Harvard Art Museums data pipeline
  - query Harvard artifacts database
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering solution for the Harvard Art Museums API. It implements a complete ETL pipeline that extracts artifact metadata, transforms nested JSON into relational structures, loads data into SQL databases (MySQL/TiDB), and provides interactive analytics dashboards using Streamlit and Plotly.

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

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

### 1. Harvard Art Museums API Key

Get your API key from: https://www.harvardartmuseums.org/collections/api

```python
# Store in environment variable
export HARVARD_API_KEY="your_api_key_here"
```

### 2. Database Configuration

```python
# MySQL/TiDB connection (use environment variables)
import os
import mysql.connector

db_config = {
    'host': os.getenv('DB_HOST'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

conn = mysql.connector.connect(**db_config)
```

### 3. Environment Variables Setup

Create `.env` file:

```bash
HARVARD_API_KEY=your_api_key
DB_HOST=your_db_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

## Core ETL Pipeline

### Extract: Fetch Artifacts from API

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    try:
        response = requests.get(base_url, params=params, timeout=30)
        response.raise_for_status()
        data = response.json()
        
        return {
            'records': data.get('records', []),
            'total': data.get('info', {}).get('totalrecords', 0),
            'pages': data.get('info', {}).get('pages', 0)
        }
    except requests.exceptions.RequestException as e:
        print(f"API Error: {e}")
        return None

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts = fetch_artifacts(api_key, page=1, size=50)
print(f"Total artifacts: {artifacts['total']}")
```

### Transform: Parse Nested JSON

```python
import pandas as pd

def transform_artifacts(records):
    """
    Transform nested JSON into normalized dataframes
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for record in records:
        artifact_id = record.get('id')
        
        # Artifact Metadata
        metadata = {
            'artifact_id': artifact_id,
            'title': record.get('title'),
            'culture': record.get('culture'),
            'period': record.get('period'),
            'century': record.get('century'),
            'classification': record.get('classification'),
            'division': record.get('division'),
            'department': record.get('department'),
            'dated': record.get('dated'),
            'medium': record.get('medium'),
            'dimensions': record.get('dimensions'),
            'creditline': record.get('creditline'),
            'url': record.get('url')
        }
        metadata_list.append(metadata)
        
        # Artifact Media
        for media in record.get('images', []):
            media_data = {
                'artifact_id': artifact_id,
                'image_id': media.get('imageid'),
                'base_url': media.get('baseimageurl'),
                'format': media.get('format'),
                'width': media.get('width'),
                'height': media.get('height')
            }
            media_list.append(media_data)
        
        # Artifact Colors
        for color in record.get('colors', []):
            color_data = {
                'artifact_id': artifact_id,
                'color_name': color.get('color'),
                'hex': color.get('hex'),
                'percentage': color.get('percent')
            }
            colors_list.append(color_data)
    
    return {
        'metadata': pd.DataFrame(metadata_list),
        'media': pd.DataFrame(media_list),
        'colors': pd.DataFrame(colors_list)
    }

# Usage
transformed = transform_artifacts(artifacts['records'])
print(f"Metadata records: {len(transformed['metadata'])}")
print(f"Media records: {len(transformed['media'])}")
print(f"Color records: {len(transformed['colors'])}")
```

### Load: Insert into SQL Database

```python
import mysql.connector
from mysql.connector import Error

def create_tables(conn):
    """
    Create normalized database schema
    """
    cursor = conn.cursor()
    
    # Artifact Metadata Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            artifact_id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            period VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            division VARCHAR(200),
            department VARCHAR(200),
            dated VARCHAR(200),
            medium TEXT,
            dimensions VARCHAR(500),
            creditline TEXT,
            url VARCHAR(500)
        )
    """)
    
    # Artifact Media Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            image_id INT,
            base_url VARCHAR(500),
            format VARCHAR(50),
            width INT,
            height INT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    # Artifact Colors Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color_name VARCHAR(100),
            hex VARCHAR(10),
            percentage FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    conn.commit()
    cursor.close()

def load_to_database(conn, dataframes):
    """
    Batch insert dataframes into database
    """
    cursor = conn.cursor()
    
    # Insert Metadata
    metadata_df = dataframes['metadata']
    for _, row in metadata_df.iterrows():
        cursor.execute("""
            INSERT IGNORE INTO artifactmetadata 
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """, tuple(row))
    
    # Insert Media
    media_df = dataframes['media']
    for _, row in media_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmedia (artifact_id, image_id, base_url, format, width, height)
            VALUES (%s, %s, %s, %s, %s, %s)
        """, tuple(row))
    
    # Insert Colors
    colors_df = dataframes['colors']
    for _, row in colors_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactcolors (artifact_id, color_name, hex, percentage)
            VALUES (%s, %s, %s, %s)
        """, tuple(row))
    
    conn.commit()
    cursor.close()

# Usage
conn = mysql.connector.connect(**db_config)
create_tables(conn)
load_to_database(conn, transformed)
conn.close()
```

## Streamlit Dashboard

### Basic App Structure

```python
import streamlit as st
import pandas as pd
import plotly.express as px
import mysql.connector
import os

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

# Database connection
@st.cache_resource
def get_database_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

# Main dashboard
st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Sidebar navigation
page = st.sidebar.selectbox("Select Page", 
    ["Data Collection", "SQL Analytics", "Visualizations"])

if page == "Data Collection":
    st.header("API Data Collection")
    
    api_key = st.text_input("Harvard API Key", type="password", 
                            value=os.getenv('HARVARD_API_KEY', ''))
    num_pages = st.number_input("Pages to Fetch", 1, 10, 1)
    
    if st.button("Fetch Artifacts"):
        with st.spinner("Fetching data..."):
            all_records = []
            for page_num in range(1, num_pages + 1):
                result = fetch_artifacts(api_key, page=page_num)
                if result:
                    all_records.extend(result['records'])
            
            st.success(f"Fetched {len(all_records)} artifacts")
            transformed = transform_artifacts(all_records)
            st.dataframe(transformed['metadata'].head())

elif page == "SQL Analytics":
    st.header("SQL Query Analytics")
    
    queries = {
        "Top 10 Cultures": """
            SELECT culture, COUNT(*) as count 
            FROM artifactmetadata 
            WHERE culture IS NOT NULL 
            GROUP BY culture 
            ORDER BY count DESC 
            LIMIT 10
        """,
        "Artifacts by Century": """
            SELECT century, COUNT(*) as count 
            FROM artifactmetadata 
            WHERE century IS NOT NULL 
            GROUP BY century 
            ORDER BY count DESC
        """,
        "Most Common Colors": """
            SELECT color_name, COUNT(*) as occurrences, 
                   AVG(percentage) as avg_percentage
            FROM artifactcolors 
            GROUP BY color_name 
            ORDER BY occurrences DESC 
            LIMIT 10
        """,
        "Department Distribution": """
            SELECT department, COUNT(*) as artifacts
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY artifacts DESC
        """
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        conn = get_database_connection()
        df = pd.read_sql(queries[selected_query], conn)
        
        st.subheader("Query Results")
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)
        
        conn.close()

elif page == "Visualizations":
    st.header("Interactive Visualizations")
    
    conn = get_database_connection()
    
    # Color distribution
    color_query = """
        SELECT color_name, SUM(percentage) as total_percentage
        FROM artifactcolors
        GROUP BY color_name
        ORDER BY total_percentage DESC
        LIMIT 15
    """
    color_df = pd.read_sql(color_query, conn)
    
    fig = px.pie(color_df, names='color_name', values='total_percentage',
                 title='Color Distribution Across Artifacts')
    st.plotly_chart(fig, use_container_width=True)
    
    conn.close()
```

### Run the Dashboard

```bash
streamlit run app.py
```

## Common Analytical Queries

```python
# Artifacts with most images
"""
SELECT m.artifact_id, m.title, COUNT(med.id) as image_count
FROM artifactmetadata m
JOIN artifactmedia med ON m.artifact_id = med.artifact_id
GROUP BY m.artifact_id, m.title
ORDER BY image_count DESC
LIMIT 20
"""

# Average image dimensions by classification
"""
SELECT m.classification, 
       AVG(med.width) as avg_width, 
       AVG(med.height) as avg_height,
       COUNT(*) as total_images
FROM artifactmetadata m
JOIN artifactmedia med ON m.artifact_id = med.artifact_id
WHERE m.classification IS NOT NULL
GROUP BY m.classification
ORDER BY total_images DESC
"""

# Color palette for specific culture
"""
SELECT c.color_name, c.hex, AVG(c.percentage) as avg_pct
FROM artifactcolors c
JOIN artifactmetadata m ON c.artifact_id = m.artifact_id
WHERE m.culture = 'Japanese'
GROUP BY c.color_name, c.hex
ORDER BY avg_pct DESC
LIMIT 10
"""
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(api_key, page, retries=3):
    for attempt in range(retries):
        try:
            result = fetch_artifacts(api_key, page)
            return result
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:  # Too Many Requests
                wait_time = 2 ** attempt
                time.sleep(wait_time)
            else:
                raise
    return None
```

### Database Connection Issues

```python
def safe_db_connection():
    try:
        conn = mysql.connector.connect(**db_config)
        conn.ping(reconnect=True)
        return conn
    except Error as e:
        st.error(f"Database connection failed: {e}")
        return None
```

### Missing Data Handling

```python
# Handle null values in transformation
metadata = {
    'artifact_id': artifact_id,
    'title': record.get('title', 'Unknown'),
    'culture': record.get('culture') or 'Not Specified',
    'century': record.get('century') or 'Unknown Period'
}
```

## Best Practices

1. **Pagination**: Always implement pagination for large datasets
2. **Error Handling**: Wrap API calls in try-except blocks
3. **Data Validation**: Validate artifact IDs before foreign key inserts
4. **Caching**: Use `@st.cache_data` for expensive operations
5. **Environment Variables**: Never hardcode credentials
6. **Batch Processing**: Insert data in batches for performance
