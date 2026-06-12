---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with SQL and Streamlit
triggers:
  - how do I extract data from Harvard Art Museums API
  - build an ETL pipeline for museum artifacts
  - create analytics dashboard with Streamlit and SQL
  - process Harvard museum collection data
  - set up artifact data visualization with Plotly
  - how to paginate Harvard Art Museums API requests
  - query museum artifacts by culture or century
  - visualize art collection data with interactive charts
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipeline construction, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection app provides:
- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads nested JSON data into relational SQL tables
- **SQL Analytics**: 20+ predefined analytical queries for artifact insights
- **Interactive Dashboards**: Streamlit frontend with Plotly visualizations

**Architecture Flow**: `API → ETL → SQL → Analytics → Visualization`

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
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="artifacts_db"
```

## Required Dependencies

```python
# requirements.txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### API Key Setup

Get your Harvard Art Museums API key from: https://harvardartmuseums.org/collections/api

```python
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"
```

### Database Configuration

```python
import mysql.connector

db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

connection = mysql.connector.connect(**db_config)
```

## Database Schema

Create the following tables for the ETL pipeline:

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255),
    accessionyear INT,
    period VARCHAR(255),
    technique VARCHAR(500)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(1000),
    primaryimageurl VARCHAR(1000),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## API Data Extraction

### Fetch Artifacts with Pagination

```python
import requests
import time

def fetch_artifacts(api_key, num_records=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    page = 1
    size = 100  # API max per page
    all_records = []
    
    while len(all_records) < num_records:
        params = {
            'apikey': api_key,
            'page': page,
            'size': min(size, num_records - len(all_records))
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            
            if not records:
                break
                
            all_records.extend(records)
            page += 1
            
            # Rate limiting - be respectful to API
            time.sleep(0.5)
        else:
            print(f"Error: {response.status_code}")
            break
    
    return all_records[:num_records]
```

### Extract Specific Fields

```python
def extract_artifact_data(records):
    """
    Extract and flatten artifact data for SQL insertion
    """
    artifacts = []
    
    for record in records:
        artifact = {
            'id': record.get('id'),
            'title': record.get('title', 'Unknown')[:500],
            'culture': record.get('culture', 'Unknown'),
            'century': record.get('century', 'Unknown'),
            'classification': record.get('classification', 'Unknown'),
            'department': record.get('department', 'Unknown'),
            'division': record.get('division', 'Unknown'),
            'dated': record.get('dated', 'Unknown'),
            'accessionyear': record.get('accessionyear'),
            'period': record.get('period', 'Unknown'),
            'technique': record.get('technique', 'Unknown')
        }
        artifacts.append(artifact)
    
    return artifacts
```

## ETL Pipeline Implementation

### Complete ETL Function

```python
import pandas as pd
import mysql.connector

def etl_artifacts_to_sql(api_key, db_config, num_records=100):
    """
    Complete ETL pipeline: Extract from API, Transform, Load to SQL
    """
    # EXTRACT
    print("Extracting data from API...")
    raw_records = fetch_artifacts(api_key, num_records)
    
    # TRANSFORM - Metadata
    print("Transforming metadata...")
    metadata = extract_artifact_data(raw_records)
    df_metadata = pd.DataFrame(metadata)
    
    # TRANSFORM - Media
    print("Transforming media data...")
    media_data = []
    for record in raw_records:
        if record.get('id'):
            media_data.append({
                'artifact_id': record.get('id'),
                'baseimageurl': record.get('baseimageurl', ''),
                'primaryimageurl': record.get('primaryimageurl', '')
            })
    df_media = pd.DataFrame(media_data)
    
    # TRANSFORM - Colors
    print("Transforming color data...")
    color_data = []
    for record in raw_records:
        artifact_id = record.get('id')
        colors = record.get('colors', [])
        
        for color in colors:
            color_data.append({
                'artifact_id': artifact_id,
                'color': color.get('color', ''),
                'spectrum': color.get('spectrum', ''),
                'hue': color.get('hue', ''),
                'percent': color.get('percent', 0.0)
            })
    df_colors = pd.DataFrame(color_data)
    
    # LOAD
    print("Loading data to SQL...")
    connection = mysql.connector.connect(**db_config)
    cursor = connection.cursor()
    
    # Load metadata
    for _, row in df_metadata.iterrows():
        cursor.execute("""
            INSERT IGNORE INTO artifactmetadata 
            (id, title, culture, century, classification, department, 
             division, dated, accessionyear, period, technique)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """, tuple(row))
    
    # Load media
    for _, row in df_media.iterrows():
        cursor.execute("""
            INSERT INTO artifactmedia (artifact_id, baseimageurl, primaryimageurl)
            VALUES (%s, %s, %s)
        """, tuple(row))
    
    # Load colors
    for _, row in df_colors.iterrows():
        cursor.execute("""
            INSERT INTO artifactcolors (artifact_id, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """, tuple(row))
    
    connection.commit()
    cursor.close()
    connection.close()
    
    print(f"ETL Complete: {len(df_metadata)} artifacts loaded")
```

## SQL Analytics Queries

### Sample Analytical Queries

```python
# Query 1: Artifacts by Culture
query_culture = """
SELECT culture, COUNT(*) as count
FROM artifactmetadata
WHERE culture != 'Unknown'
GROUP BY culture
ORDER BY count DESC
LIMIT 10
"""

# Query 2: Artifacts by Century
query_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC
"""

# Query 3: Media Availability
query_media = """
SELECT 
    SUM(CASE WHEN primaryimageurl != '' THEN 1 ELSE 0 END) as has_image,
    SUM(CASE WHEN primaryimageurl = '' THEN 1 ELSE 0 END) as no_image
FROM artifactmedia
"""

# Query 4: Color Distribution
query_colors = """
SELECT color, COUNT(*) as frequency
FROM artifactcolors
GROUP BY color
ORDER BY frequency DESC
LIMIT 10
"""

# Query 5: Department Distribution
query_dept = """
SELECT department, COUNT(*) as count
FROM artifactmetadata
WHERE department != 'Unknown'
GROUP BY department
ORDER BY count DESC
"""
```

### Execute Query Function

```python
def execute_query(query, db_config):
    """
    Execute SQL query and return results as DataFrame
    """
    connection = mysql.connector.connect(**db_config)
    df = pd.read_sql(query, connection)
    connection.close()
    return df
```

## Streamlit Dashboard

### Main App Structure

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums Analytics Dashboard")
st.markdown("Explore artifact data through interactive SQL analytics")

# Sidebar for configuration
st.sidebar.header("Configuration")
api_key = st.sidebar.text_input("Harvard API Key", type="password", 
                                 value=os.getenv('HARVARD_API_KEY', ''))

# ETL Section
st.header("📥 Data Collection (ETL)")
num_records = st.number_input("Number of artifacts to fetch", 
                               min_value=10, max_value=1000, value=100)

if st.button("Run ETL Pipeline"):
    with st.spinner("Running ETL..."):
        try:
            etl_artifacts_to_sql(api_key, db_config, num_records)
            st.success(f"✅ Successfully loaded {num_records} artifacts")
        except Exception as e:
            st.error(f"❌ ETL Error: {str(e)}")

# Analytics Section
st.header("📊 SQL Analytics")

queries = {
    "Artifacts by Culture": query_culture,
    "Artifacts by Century": query_century,
    "Media Availability": query_media,
    "Color Distribution": query_colors,
    "Department Distribution": query_dept
}

selected_query = st.selectbox("Select Analysis", list(queries.keys()))

if st.button("Run Query"):
    with st.spinner("Executing query..."):
        try:
            df_result = execute_query(queries[selected_query], db_config)
            
            # Display table
            st.subheader("Query Results")
            st.dataframe(df_result)
            
            # Auto-generate visualization
            if len(df_result.columns) == 2:
                fig = px.bar(df_result, 
                            x=df_result.columns[0], 
                            y=df_result.columns[1],
                            title=selected_query)
                st.plotly_chart(fig, use_container_width=True)
                
        except Exception as e:
            st.error(f"Query Error: {str(e)}")
```

## Common Patterns

### Batch Insert Optimization

```python
def batch_insert_artifacts(cursor, artifacts, batch_size=100):
    """
    Insert artifacts in batches for better performance
    """
    for i in range(0, len(artifacts), batch_size):
        batch = artifacts[i:i+batch_size]
        
        values = [tuple(art.values()) for art in batch]
        
        cursor.executemany("""
            INSERT IGNORE INTO artifactmetadata 
            (id, title, culture, century, classification, department, 
             division, dated, accessionyear, period, technique)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """, values)
```

### Error Handling for API Requests

```python
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
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
    return None
```

## Troubleshooting

### API Rate Limiting
If you encounter 429 errors, increase the delay between requests:
```python
time.sleep(1)  # Increase from 0.5 to 1 second
```

### Database Connection Issues
Verify credentials and network access:
```python
try:
    connection = mysql.connector.connect(**db_config)
    print("✅ Database connected")
except mysql.connector.Error as err:
    print(f"❌ Database error: {err}")
```

### Missing Data Fields
Handle missing fields gracefully:
```python
artifact = {
    'id': record.get('id'),
    'title': (record.get('title') or 'Unknown')[:500],
    'culture': record.get('culture') or 'Unknown'
}
```

### Streamlit Performance
For large datasets, use caching:
```python
@st.cache_data
def load_data_cached(query, db_config):
    return execute_query(query, db_config)
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```
