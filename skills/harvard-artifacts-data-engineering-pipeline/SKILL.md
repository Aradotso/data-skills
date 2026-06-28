---
name: harvard-artifacts-data-engineering-pipeline
description: Build end-to-end data engineering pipelines with Harvard Art Museums API, ETL workflows, SQL analytics, and Streamlit visualization
triggers:
  - build a data pipeline for Harvard Art Museums API
  - create ETL workflow for museum artifact data
  - set up Harvard artifacts analytics dashboard
  - analyze Harvard museum collection data with SQL
  - visualize art museum data with Streamlit
  - extract and transform Harvard API data
  - design artifact data warehouse with SQL
  - build museum collection analytics app
---

# Harvard Artifacts Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Harvard Artifacts Collection Data Engineering Analytics App is an end-to-end data pipeline that demonstrates real-world ETL practices using the Harvard Art Museums API. It extracts artifact metadata, transforms nested JSON into relational tables, loads data into SQL databases (MySQL/TiDB Cloud), and provides interactive analytics through a Streamlit dashboard.

**Architecture**: API → ETL → SQL → Analytics → Visualization

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

### API Key Setup

1. Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api
2. Store it securely using environment variables:

```python
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')
```

### Database Configuration

```python
import mysql.connector

db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME'),
    'port': int(os.getenv('DB_PORT', 3306))
}

connection = mysql.connector.connect(**db_config)
```

## Database Schema

The project uses three main tables with foreign key relationships:

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    description TEXT,
    provenance TEXT
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    image_url VARCHAR(1000),
    image_height INT,
    image_width INT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color_hex VARCHAR(10),
    color_percent FLOAT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## ETL Pipeline Implementation

### 1. Extract: Fetch Data from API

```python
import requests
import pandas as pd

def fetch_artifacts(api_key, num_records=100):
    """
    Extract artifact data from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    artifacts = []
    page = 1
    size = 100  # API max per page
    
    while len(artifacts) < num_records:
        params = {
            'apikey': api_key,
            'size': min(size, num_records - len(artifacts)),
            'page': page
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            artifacts.extend(data.get('records', []))
            
            if len(data.get('records', [])) == 0:
                break
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts
```

### 2. Transform: Process Nested JSON

```python
def transform_artifacts(artifacts):
    """
    Transform nested JSON into relational dataframes
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'description': artifact.get('description'),
            'provenance': artifact.get('provenance')
        }
        metadata_list.append(metadata)
        
        # Extract media images
        images = artifact.get('images', [])
        for img in images:
            media = {
                'objectid': artifact.get('objectid'),
                'image_url': img.get('baseimageurl'),
                'image_height': img.get('height'),
                'image_width': img.get('width')
            }
            media_list.append(media)
        
        # Extract color data
        colors = artifact.get('colors', [])
        for color in colors:
            color_data = {
                'objectid': artifact.get('objectid'),
                'color_hex': color.get('hex'),
                'color_percent': color.get('percent')
            }
            colors_list.append(color_data)
    
    df_metadata = pd.DataFrame(metadata_list)
    df_media = pd.DataFrame(media_list)
    df_colors = pd.DataFrame(colors_list)
    
    return df_metadata, df_media, df_colors
```

### 3. Load: Insert into SQL Database

```python
def load_to_database(df_metadata, df_media, df_colors, connection):
    """
    Batch insert dataframes into SQL database
    """
    cursor = connection.cursor()
    
    # Load metadata
    for _, row in df_metadata.iterrows():
        query = """
        INSERT INTO artifactmetadata 
        (objectid, title, culture, period, century, classification, department, dated, description, provenance)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.execute(query, tuple(row))
    
    # Load media
    for _, row in df_media.iterrows():
        query = """
        INSERT INTO artifactmedia (objectid, image_url, image_height, image_width)
        VALUES (%s, %s, %s, %s)
        """
        cursor.execute(query, tuple(row))
    
    # Load colors
    for _, row in df_colors.iterrows():
        query = """
        INSERT INTO artifactcolors (objectid, color_hex, color_percent)
        VALUES (%s, %s, %s)
        """
        cursor.execute(query, tuple(row))
    
    connection.commit()
    cursor.close()
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

# Artifacts with images vs without
query_media_availability = """
SELECT 
    CASE WHEN m.objectid IS NOT NULL THEN 'With Images' ELSE 'Without Images' END as media_status,
    COUNT(*) as count
FROM artifactmetadata a
LEFT JOIN artifactmedia m ON a.objectid = m.objectid
GROUP BY media_status;
"""

# Most common colors across artifacts
query_colors = """
SELECT color_hex, COUNT(*) as usage_count, AVG(color_percent) as avg_percent
FROM artifactcolors
GROUP BY color_hex
ORDER BY usage_count DESC
LIMIT 15;
"""

# Department-wise artifact distribution
query_departments = """
SELECT department, COUNT(*) as total_artifacts
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY total_artifacts DESC;
"""

# Artifacts by century
query_centuries = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC
LIMIT 10;
"""
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password", value=os.getenv('HARVARD_API_KEY', ''))
    
    # ETL Pipeline Section
    st.header("1️⃣ ETL Pipeline")
    num_records = st.number_input("Number of records to fetch", min_value=10, max_value=1000, value=100)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            artifacts = fetch_artifacts(api_key, num_records)
            st.success(f"Extracted {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_metadata, df_media, df_colors = transform_artifacts(artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            connection = mysql.connector.connect(**db_config)
            load_to_database(df_metadata, df_media, df_colors, connection)
            connection.close()
            st.success("Data loaded to database")
    
    # Analytics Section
    st.header("2️⃣ SQL Analytics")
    
    query_options = {
        "Top 10 Cultures": query_cultures,
        "Media Availability": query_media_availability,
        "Color Distribution": query_colors,
        "Department Distribution": query_departments,
        "Artifacts by Century": query_centuries
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
    
    if st.button("Execute Query"):
        connection = mysql.connector.connect(**db_config)
        df_result = pd.read_sql(query_options[selected_query], connection)
        connection.close()
        
        st.dataframe(df_result)
        
        # Auto-generate visualization
        if len(df_result.columns) >= 2:
            fig = px.bar(df_result, x=df_result.columns[0], y=df_result.columns[1],
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Set environment variables
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_db_host"
export DB_USER="your_db_user"
export DB_PASSWORD="your_db_password"
export DB_NAME="harvard_artifacts"

# Run Streamlit app
streamlit run app.py
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(api_key, num_records, delay=0.5):
    artifacts = []
    page = 1
    
    while len(artifacts) < num_records:
        response = requests.get(base_url, params={'apikey': api_key, 'page': page})
        if response.status_code == 200:
            artifacts.extend(response.json().get('records', []))
            page += 1
            time.sleep(delay)  # Respect API rate limits
        else:
            break
    
    return artifacts
```

### Batch Insert Optimization

```python
def batch_insert(df, table_name, connection, batch_size=1000):
    cursor = connection.cursor()
    
    for i in range(0, len(df), batch_size):
        batch = df.iloc[i:i+batch_size]
        # Insert batch
        cursor.executemany(query, [tuple(row) for _, row in batch.iterrows()])
        connection.commit()
    
    cursor.close()
```

## Troubleshooting

**API Key Issues**:
- Ensure your Harvard API key is valid and not expired
- Check environment variable is properly loaded: `print(os.getenv('HARVARD_API_KEY'))`

**Database Connection Errors**:
- Verify database credentials and network connectivity
- For TiDB Cloud, ensure SSL settings if required
- Check firewall rules allow connections

**Empty Results**:
- API may return limited data for certain queries
- Check pagination logic and API response structure
- Validate API endpoint and parameters

**Memory Issues with Large Datasets**:
- Use chunked processing for large extracts
- Implement batch inserts instead of row-by-row
- Clear dataframes after loading: `del df_metadata`

**Streamlit Performance**:
- Cache database connections: `@st.cache_resource`
- Cache query results: `@st.cache_data`
- Limit visualization data points for large datasets
