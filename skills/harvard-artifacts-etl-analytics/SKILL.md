---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Streamlit, SQL, and Plotly
triggers:
  - how do I fetch data from Harvard Art Museums API
  - build an ETL pipeline for museum artifacts
  - create analytics dashboard with Streamlit and SQL
  - process Harvard museum collection data
  - design SQL schema for artifact metadata
  - visualize museum data with Plotly
  - implement batch data insertion for museum artifacts
  - query and analyze art collection database
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. It covers ETL pipeline design, SQL database schema creation, batch data processing, analytical query implementation, and interactive visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection app demonstrates a complete data engineering workflow:

1. **Extract**: Fetch artifact data from Harvard Art Museums API with pagination
2. **Transform**: Parse nested JSON into relational structures (metadata, media, colors)
3. **Load**: Batch insert into MySQL/TiDB Cloud with proper schema design
4. **Analyze**: Execute 20+ predefined analytical SQL queries
5. **Visualize**: Display results through interactive Plotly charts in Streamlit

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Expected packages:
# streamlit
# pandas
# requests
# mysql-connector-python
# plotly
# python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file or configure via Streamlit secrets:

```python
# .env file
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_PORT=4000
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Streamlit Secrets Configuration

```toml
# .streamlit/secrets.toml
[api]
harvard_key = "your_api_key_here"

[database]
host = "your_database_host"
port = 4000
user = "your_username"
password = "your_password"
database = "harvard_artifacts"
```

## Database Schema Design

### Create Tables

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    dated VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accessionyear INT,
    url VARCHAR(500)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    publiccaption TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key API Patterns

### API Data Collection

```python
import requests
import time

def fetch_artifacts(api_key, num_records=100, page_size=100):
    """
    Fetch artifact data from Harvard Art Museums API with pagination.
    
    Args:
        api_key: Harvard API key from environment
        num_records: Total records to fetch
        page_size: Records per API call (max 100)
    
    Returns:
        List of artifact dictionaries
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    
    pages_needed = (num_records + page_size - 1) // page_size
    
    for page in range(1, pages_needed + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            all_artifacts.extend(records)
            
            # Rate limiting - be respectful to API
            time.sleep(0.5)
        else:
            print(f"Error on page {page}: {response.status_code}")
            break
    
    return all_artifacts[:num_records]
```

### ETL Transform Functions

```python
import pandas as pd

def transform_metadata(artifacts):
    """Extract and transform artifact metadata into DataFrame."""
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', ''),
            'culture': artifact.get('culture', ''),
            'century': artifact.get('century', ''),
            'dated': artifact.get('dated', ''),
            'classification': artifact.get('classification', ''),
            'department': artifact.get('department', ''),
            'division': artifact.get('division', ''),
            'technique': artifact.get('technique', ''),
            'medium': artifact.get('medium', ''),
            'dimensions': artifact.get('dimensions', ''),
            'creditline': artifact.get('creditline', ''),
            'accessionyear': artifact.get('accessionyear'),
            'url': artifact.get('url', '')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_media(artifacts):
    """Extract media/image data with artifact relationships."""
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for image in images:
            media = {
                'artifact_id': artifact_id,
                'baseimageurl': image.get('baseimageurl', ''),
                'iiifbaseuri': image.get('iiifbaseuri', ''),
                'publiccaption': image.get('publiccaption', '')
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_colors(artifacts):
    """Extract color analysis data."""
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color': color.get('color', ''),
                'spectrum': color.get('spectrum', ''),
                'hue': color.get('hue', ''),
                'percent': color.get('percent', 0)
            }
            color_list.append(color_data)
    
    return pd.DataFrame(color_list)
```

### Batch Database Loading

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection from environment variables."""
    import os
    
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def batch_insert_metadata(df_metadata):
    """Batch insert artifact metadata with conflict handling."""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, dated, classification, department, 
     division, technique, medium, dimensions, creditline, accessionyear, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    # Convert DataFrame to list of tuples
    data_tuples = [tuple(x) for x in df_metadata.to_numpy()]
    
    try:
        cursor.executemany(insert_query, data_tuples)
        conn.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
    except Error as e:
        print(f"Error inserting metadata: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()

def batch_insert_media(df_media):
    """Batch insert media data."""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (artifact_id, baseimageurl, iiifbaseuri, publiccaption)
    VALUES (%s, %s, %s, %s)
    """
    
    data_tuples = [tuple(x) for x in df_media.to_numpy()]
    
    try:
        cursor.executemany(insert_query, data_tuples)
        conn.commit()
        print(f"Inserted {cursor.rowcount} media records")
    except Error as e:
        print(f"Error inserting media: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()
```

## Analytical SQL Queries

### Common Analytics Patterns

```python
# Top 10 cultures by artifact count
QUERY_CULTURES = """
SELECT culture, COUNT(*) as count
FROM artifactmetadata
WHERE culture IS NOT NULL AND culture != ''
GROUP BY culture
ORDER BY count DESC
LIMIT 10
"""

# Artifacts with images vs without
QUERY_IMAGE_AVAILABILITY = """
SELECT 
    CASE WHEN media_id IS NOT NULL THEN 'With Images' ELSE 'Without Images' END as image_status,
    COUNT(DISTINCT am.id) as count
FROM artifactmetadata am
LEFT JOIN artifactmedia med ON am.id = med.artifact_id
GROUP BY image_status
"""

# Color distribution analysis
QUERY_COLOR_DISTRIBUTION = """
SELECT spectrum, COUNT(*) as count
FROM artifactcolors
WHERE spectrum IS NOT NULL
GROUP BY spectrum
ORDER BY count DESC
"""

# Century-wise artifact distribution
QUERY_CENTURY_DISTRIBUTION = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL AND century != ''
GROUP BY century
ORDER BY count DESC
LIMIT 15
"""

# Department analytics
QUERY_DEPARTMENT_STATS = """
SELECT 
    department,
    COUNT(*) as total_artifacts,
    COUNT(DISTINCT classification) as unique_classifications
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY total_artifacts DESC
"""
```

### Execute and Visualize Queries

```python
import pandas as pd
import plotly.express as px

def execute_query(query):
    """Execute SQL query and return DataFrame."""
    conn = get_db_connection()
    try:
        df = pd.read_sql(query, conn)
        return df
    finally:
        conn.close()

def create_bar_chart(df, x_col, y_col, title):
    """Create interactive Plotly bar chart."""
    fig = px.bar(
        df,
        x=x_col,
        y=y_col,
        title=title,
        labels={x_col: x_col.replace('_', ' ').title(), 
                y_col: y_col.replace('_', ' ').title()}
    )
    fig.update_layout(xaxis_tickangle=-45)
    return fig
```

## Streamlit Application Patterns

### Main App Structure

```python
import streamlit as st

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for configuration
    with st.sidebar:
        st.header("Configuration")
        api_key = st.text_input("Harvard API Key", type="password")
        
        if st.button("Fetch New Data"):
            with st.spinner("Fetching artifacts..."):
                artifacts = fetch_artifacts(api_key, num_records=500)
                st.success(f"Fetched {len(artifacts)} artifacts")
    
    # Main analytics section
    st.header("📊 Analytics Queries")
    
    query_options = {
        "Top Cultures": QUERY_CULTURES,
        "Image Availability": QUERY_IMAGE_AVAILABILITY,
        "Color Distribution": QUERY_COLOR_DISTRIBUTION,
        "Century Distribution": QUERY_CENTURY_DISTRIBUTION
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
    
    if st.button("Run Query"):
        df_result = execute_query(query_options[selected_query])
        
        col1, col2 = st.columns([1, 2])
        
        with col1:
            st.dataframe(df_result)
        
        with col2:
            if len(df_result.columns) >= 2:
                fig = create_bar_chart(
                    df_result, 
                    df_result.columns[0], 
                    df_result.columns[1],
                    selected_query
                )
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Complete ETL Workflow

```python
def run_complete_etl(api_key, num_records=100):
    """Execute full ETL pipeline."""
    
    # Extract
    st.write("### Step 1: Extracting data from API")
    artifacts = fetch_artifacts(api_key, num_records)
    st.success(f"✓ Extracted {len(artifacts)} artifacts")
    
    # Transform
    st.write("### Step 2: Transforming data")
    df_metadata = transform_metadata(artifacts)
    df_media = transform_media(artifacts)
    df_colors = transform_colors(artifacts)
    st.success(f"✓ Transformed into {len(df_metadata)} metadata, "
               f"{len(df_media)} media, {len(df_colors)} color records")
    
    # Load
    st.write("### Step 3: Loading into database")
    batch_insert_metadata(df_metadata)
    batch_insert_media(df_media)
    batch_insert_colors(df_colors)
    st.success("✓ Data loaded successfully")
    
    return df_metadata, df_media, df_colors
```

## Troubleshooting

### API Rate Limiting
```python
# Add exponential backoff for rate limits
import time
from requests.exceptions import RequestException

def fetch_with_retry(url, params, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            if response.status_code == 429:  # Rate limited
                wait_time = 2 ** attempt
                time.sleep(wait_time)
                continue
            return response
        except RequestException as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)
```

### Database Connection Issues
```python
# Test database connection
def test_db_connection():
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        cursor.close()
        conn.close()
        return True
    except Error as e:
        st.error(f"Database connection failed: {e}")
        return False
```

### Handling NULL Values
```python
# Clean data before insertion
def clean_dataframe(df):
    """Replace None/NaN with empty strings or appropriate defaults."""
    df = df.fillna('')
    df = df.replace({pd.NA: '', None: ''})
    return df
```

This skill provides comprehensive coverage of building ETL pipelines and analytics dashboards using the Harvard Art Museums API with modern data engineering tools.
