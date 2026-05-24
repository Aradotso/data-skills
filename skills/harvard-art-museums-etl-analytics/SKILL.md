---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - extract and transform Harvard API data
  - set up museum collection data engineering project
  - visualize Harvard Art Museums analytics
  - implement artifact metadata pipeline
  - query Harvard museum database
  - analyze art collection data with SQL
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build and work with end-to-end data engineering pipelines using the Harvard Art Museums API. It covers ETL processes, SQL database design, analytical queries, and interactive Streamlit visualization dashboards.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App demonstrates real-world data engineering patterns:

- **Extract**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **Transform**: Convert nested JSON into relational database schema
- **Load**: Batch insert structured data into MySQL/TiDB Cloud
- **Analyze**: Execute 20+ predefined SQL analytical queries
- **Visualize**: Display results in interactive Streamlit dashboards with Plotly charts

Architecture: `API → ETL → SQL → Analytics → Visualization`

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly
```

## Configuration

### API Key Setup

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

```python
# Store in environment variables
export HARVARD_API_KEY="your_api_key_here"

# Or configure in Streamlit app
# API key input is provided in the UI
```

### Database Configuration

```python
import mysql.connector

# MySQL/TiDB Cloud connection
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

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    department VARCHAR(200),
    classification VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    url VARCHAR(500)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
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

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import time

def fetch_artifacts(api_key, max_records=100):
    """Extract artifacts from Harvard API with pagination"""
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    page = 1
    
    while len(all_artifacts) < max_records:
        params = {
            'apikey': api_key,
            'size': 100,  # Max per page
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            all_artifacts.extend(records)
            
            if not records or len(all_artifacts) >= max_records:
                break
                
            page += 1
            time.sleep(0.5)  # Rate limiting
        else:
            print(f"Error: {response.status_code}")
            break
    
    return all_artifacts[:max_records]
```

### Transform: Clean and Structure Data

```python
import pandas as pd

def transform_artifacts(raw_data):
    """Transform nested JSON to relational format"""
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        # Metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'department': artifact.get('department'),
            'classification': artifact.get('classification'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'url': artifact.get('url')
        }
        metadata_list.append(metadata)
        
        # Media
        if artifact.get('primaryimageurl'):
            media = {
                'artifact_id': artifact.get('id'),
                'baseimageurl': artifact.get('primaryimageurl'),
                'iiifbaseuri': artifact.get('images', [{}])[0].get('iiifbaseuri') if artifact.get('images') else None
            }
            media_list.append(media)
        
        # Colors
        for color in artifact.get('colors', []):
            color_record = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            colors_list.append(color_record)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load: Insert into Database

```python
def load_to_database(metadata_df, media_df, colors_df, connection):
    """Batch insert data into SQL database"""
    cursor = connection.cursor()
    
    # Insert metadata
    metadata_sql = """
    INSERT INTO artifactmetadata 
    (id, title, culture, period, century, dated, department, 
     classification, technique, medium, dimensions, creditline, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    metadata_values = [tuple(row) for row in metadata_df.values]
    cursor.executemany(metadata_sql, metadata_values)
    
    # Insert media
    if not media_df.empty:
        media_sql = """
        INSERT INTO artifactmedia (artifact_id, baseimageurl, iiifbaseuri)
        VALUES (%s, %s, %s)
        """
        media_values = [tuple(row) for row in media_df.values]
        cursor.executemany(media_sql, media_values)
    
    # Insert colors
    if not colors_df.empty:
        colors_sql = """
        INSERT INTO artifactcolors (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
        """
        colors_values = [tuple(row) for row in colors_df.values]
        cursor.executemany(colors_sql, colors_values)
    
    connection.commit()
    cursor.close()
```

## Analytics Queries

### Sample Analytical Queries

```python
# Query 1: Artifacts by Culture
query_culture = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10;
"""

# Query 2: Media Availability
query_media = """
SELECT 
    CASE 
        WHEN baseimageurl IS NOT NULL THEN 'Has Image'
        ELSE 'No Image'
    END as media_status,
    COUNT(*) as count
FROM artifactmedia
GROUP BY media_status;
"""

# Query 3: Century Distribution
query_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC
LIMIT 15;
"""

# Query 4: Color Usage
query_colors = """
SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
FROM artifactcolors
GROUP BY color
ORDER BY frequency DESC
LIMIT 10;
"""

# Query 5: Department Analysis
query_dept = """
SELECT department, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY artifact_count DESC;
"""

# Execute query
def execute_query(connection, query):
    """Execute SQL query and return DataFrame"""
    cursor = connection.cursor()
    cursor.execute(query)
    columns = [desc[0] for desc in cursor.description]
    results = cursor.fetchall()
    cursor.close()
    return pd.DataFrame(results, columns=columns)
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password")
    
    # Database connection inputs
    db_host = st.sidebar.text_input("Database Host", value=os.getenv('DB_HOST', ''))
    db_user = st.sidebar.text_input("Database User", value=os.getenv('DB_USER', ''))
    db_password = st.sidebar.text_input("Database Password", type="password")
    db_name = st.sidebar.text_input("Database Name", value=os.getenv('DB_NAME', ''))
    
    # ETL Section
    st.header("📥 Data Collection & ETL")
    num_records = st.number_input("Number of records to collect", 
                                  min_value=10, max_value=1000, value=100)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Extracting data from API..."):
            artifacts = fetch_artifacts(api_key, num_records)
            st.success(f"Extracted {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            st.success("Transformation complete")
        
        with st.spinner("Loading to database..."):
            connection = mysql.connector.connect(
                host=db_host, user=db_user, 
                password=db_password, database=db_name
            )
            load_to_database(metadata_df, media_df, colors_df, connection)
            connection.close()
            st.success("Data loaded successfully!")
    
    # Analytics Section
    st.header("📊 SQL Analytics")
    
    query_options = {
        "Artifacts by Culture": query_culture,
        "Media Availability": query_media,
        "Century Distribution": query_century,
        "Color Usage": query_colors,
        "Department Analysis": query_dept
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
    
    if st.button("Run Query"):
        connection = mysql.connector.connect(
            host=db_host, user=db_user, 
            password=db_password, database=db_name
        )
        
        results_df = execute_query(connection, query_options[selected_query])
        
        # Display results
        st.dataframe(results_df)
        
        # Visualization
        if len(results_df.columns) >= 2:
            fig = px.bar(results_df, 
                        x=results_df.columns[0], 
                        y=results_df.columns[1],
                        title=selected_query)
            st.plotly_chart(fig)
        
        connection.close()

if __name__ == "__main__":
    main()
```

## Common Patterns

### Incremental Loading

```python
def incremental_load(api_key, connection, last_id=0):
    """Load only new artifacts since last run"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    last_id = result[0] if result[0] else 0
    
    # Fetch artifacts with ID > last_id
    params = {
        'apikey': api_key,
        'size': 100,
        'sort': 'id',
        'sortorder': 'asc',
        'after': last_id
    }
    
    # Continue ETL process...
```

### Error Handling

```python
def safe_etl_pipeline(api_key, connection, max_records):
    """ETL with comprehensive error handling"""
    try:
        # Extract
        artifacts = fetch_artifacts(api_key, max_records)
        if not artifacts:
            raise ValueError("No artifacts fetched")
        
        # Transform
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        
        # Validate
        if metadata_df.empty:
            raise ValueError("Transformation resulted in empty dataframe")
        
        # Load
        load_to_database(metadata_df, media_df, colors_df, connection)
        
        return True, f"Successfully processed {len(artifacts)} artifacts"
        
    except requests.RequestException as e:
        return False, f"API Error: {str(e)}"
    except mysql.connector.Error as e:
        return False, f"Database Error: {str(e)}"
    except Exception as e:
        return False, f"Unexpected Error: {str(e)}"
```

## Running the Application

```bash
# Set environment variables
export HARVARD_API_KEY="your_api_key"
export DB_HOST="your_db_host"
export DB_USER="your_db_user"
export DB_PASSWORD="your_db_password"
export DB_NAME="harvard_artifacts"

# Run Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Troubleshooting

### API Rate Limiting
```python
# Add exponential backoff
import time
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def create_session():
    session = requests.Session()
    retry = Retry(total=5, backoff_factor=1, status_forcelist=[429, 500, 502, 503, 504])
    adapter = HTTPAdapter(max_retries=retry)
    session.mount('http://', adapter)
    session.mount('https://', adapter)
    return session
```

### Database Connection Issues
```python
# Test database connection
def test_db_connection(db_config):
    try:
        conn = mysql.connector.connect(**db_config)
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        cursor.close()
        conn.close()
        return True, "Connection successful"
    except Exception as e:
        return False, str(e)
```

### Memory Optimization for Large Datasets
```python
# Process in chunks
def chunked_etl(api_key, connection, total_records, chunk_size=100):
    for offset in range(0, total_records, chunk_size):
        artifacts = fetch_artifacts(api_key, chunk_size)
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        load_to_database(metadata_df, media_df, colors_df, connection)
        time.sleep(1)  # Rate limiting
```
