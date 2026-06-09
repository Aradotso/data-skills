---
name: harvard-artifacts-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for museum artifacts
  - analyze Harvard Art Museums collection data
  - create a data engineering app with Streamlit
  - set up artifact collection analytics dashboard
  - implement museum API data pipeline
  - build SQL analytics for art collections
  - create interactive museum data visualizations
  - process Harvard museums API with ETL
---

# Harvard Artifacts Collection Data Engineering & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides a complete data engineering and analytics application for the Harvard Art Museums API. It demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact collections.

## What It Does

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads nested JSON into relational database tables
- **SQL Analytics**: Stores data in MySQL/TiDB with proper schema design and foreign keys
- **Interactive Dashboard**: Streamlit UI for data collection, query execution, and visualization
- **Analytics Queries**: 20+ predefined SQL queries for insights on artifacts, cultures, media, and colors

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages:**
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

Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api):

```python
# Store in .env file
HARVARD_API_KEY=your_api_key_here
```

### Database Configuration

```python
# MySQL/TiDB connection parameters
DB_CONFIG = {
    'host': 'your_host',
    'port': 4000,
    'user': 'your_username',
    'password': 'your_password',
    'database': 'artifacts_db'
}
```

**Using environment variables (recommended):**
```python
import os
from dotenv import load_dotenv

load_dotenv()

DB_CONFIG = {
    'host': os.getenv('DB_HOST'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}
```

## Database Schema

The project uses three main tables:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    classification VARCHAR(200),
    department VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    period VARCHAR(200),
    accession_year INT,
    technique VARCHAR(500),
    medium VARCHAR(500)
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(1000),
    has_image BOOLEAN,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Components

### 1. API Data Collection

```python
import requests
import pandas as pd

def fetch_artifacts(api_key, num_pages=5):
    """
    Fetch artifacts from Harvard Art Museums API
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': 100,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            all_artifacts.extend(data.get('records', []))
        else:
            print(f"Error fetching page {page}: {response.status_code}")
            break
    
    return all_artifacts
```

### 2. ETL Pipeline

```python
import mysql.connector
from mysql.connector import Error

def extract_metadata(artifacts):
    """Extract artifact metadata into DataFrame"""
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'period': artifact.get('period'),
            'accession_year': artifact.get('accessionyear'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium')
        })
    
    return pd.DataFrame(metadata)

def extract_media(artifacts):
    """Extract media information"""
    media = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        primary_image = artifact.get('primaryimageurl')
        has_image = 1 if primary_image else 0
        
        media.append({
            'artifact_id': artifact_id,
            'image_url': primary_image,
            'has_image': has_image
        })
    
    return pd.DataFrame(media)

def extract_colors(artifacts):
    """Extract color data"""
    colors = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        color_data = artifact.get('colors', [])
        
        for color_obj in color_data:
            colors.append({
                'artifact_id': artifact_id,
                'color': color_obj.get('color'),
                'percentage': color_obj.get('percent')
            })
    
    return pd.DataFrame(colors)

def load_to_database(df, table_name, db_config):
    """Load DataFrame to MySQL table"""
    try:
        connection = mysql.connector.connect(**db_config)
        cursor = connection.cursor()
        
        # Prepare insert statement
        cols = ','.join(df.columns)
        placeholders = ','.join(['%s'] * len(df.columns))
        insert_query = f"INSERT INTO {table_name} ({cols}) VALUES ({placeholders})"
        
        # Batch insert
        data_tuples = [tuple(row) for row in df.values]
        cursor.executemany(insert_query, data_tuples)
        
        connection.commit()
        print(f"Loaded {cursor.rowcount} rows into {table_name}")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

### 3. Complete ETL Workflow

```python
def run_etl_pipeline(api_key, db_config, num_pages=5):
    """
    Complete ETL pipeline execution
    """
    # Extract from API
    print("Extracting data from Harvard API...")
    artifacts = fetch_artifacts(api_key, num_pages)
    
    # Transform into DataFrames
    print("Transforming data...")
    metadata_df = extract_metadata(artifacts)
    media_df = extract_media(artifacts)
    colors_df = extract_colors(artifacts)
    
    # Load to database
    print("Loading to database...")
    load_to_database(metadata_df, 'artifactmetadata', db_config)
    load_to_database(media_df, 'artifactmedia', db_config)
    load_to_database(colors_df, 'artifactcolors', db_config)
    
    print("ETL pipeline completed successfully!")
```

## Analytical SQL Queries

### Sample Queries

```python
# Query 1: Artifacts by culture
query_by_culture = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10
"""

# Query 2: Artifacts by century
query_by_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC
"""

# Query 3: Media availability
query_media_stats = """
SELECT 
    has_image,
    COUNT(*) as count,
    ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM artifactmedia), 2) as percentage
FROM artifactmedia
GROUP BY has_image
"""

# Query 4: Top colors used
query_top_colors = """
SELECT 
    color,
    COUNT(*) as frequency,
    ROUND(AVG(percentage), 2) as avg_percentage
FROM artifactcolors
GROUP BY color
ORDER BY frequency DESC
LIMIT 15
"""

# Query 5: Artifacts by department
query_by_department = """
SELECT department, COUNT(*) as count
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY count DESC
"""

# Query 6: Artifacts with complete metadata
query_complete_metadata = """
SELECT COUNT(*) as complete_records
FROM artifactmetadata
WHERE culture IS NOT NULL
  AND century IS NOT NULL
  AND department IS NOT NULL
  AND medium IS NOT NULL
"""
```

### Execute Query Function

```python
def execute_query(query, db_config):
    """Execute SQL query and return results as DataFrame"""
    try:
        connection = mysql.connector.connect(**db_config)
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        print(f"Query error: {e}")
        return pd.DataFrame()
    finally:
        if connection.is_connected():
            connection.close()
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("API Key", type="password", 
                                     value=os.getenv('HARVARD_API_KEY', ''))
    
    # Database configuration
    db_config = {
        'host': st.sidebar.text_input("DB Host", os.getenv('DB_HOST', 'localhost')),
        'port': st.sidebar.number_input("DB Port", value=3306),
        'user': st.sidebar.text_input("DB User", os.getenv('DB_USER', '')),
        'password': st.sidebar.text_input("DB Password", type="password", 
                                          value=os.getenv('DB_PASSWORD', '')),
        'database': st.sidebar.text_input("Database Name", 
                                          os.getenv('DB_NAME', 'artifacts_db'))
    }
    
    # Main tabs
    tab1, tab2, tab3 = st.tabs(["📥 Data Collection", "📊 Analytics", "📈 Visualizations"])
    
    with tab1:
        data_collection_tab(api_key, db_config)
    
    with tab2:
        analytics_tab(db_config)
    
    with tab3:
        visualization_tab(db_config)

if __name__ == "__main__":
    main()
```

### Data Collection Tab

```python
def data_collection_tab(api_key, db_config):
    st.header("Data Collection from Harvard API")
    
    num_pages = st.slider("Number of pages to fetch", 1, 10, 5)
    
    if st.button("🚀 Run ETL Pipeline"):
        if not api_key:
            st.error("Please provide API key")
            return
        
        with st.spinner("Running ETL pipeline..."):
            try:
                run_etl_pipeline(api_key, db_config, num_pages)
                st.success("ETL pipeline completed successfully!")
            except Exception as e:
                st.error(f"Error: {str(e)}")
```

### Analytics Tab

```python
def analytics_tab(db_config):
    st.header("SQL Analytics Queries")
    
    queries = {
        "Artifacts by Culture": query_by_culture,
        "Artifacts by Century": query_by_century,
        "Media Availability": query_media_stats,
        "Top Colors": query_top_colors,
        "Artifacts by Department": query_by_department
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Execute Query"):
        with st.spinner("Executing query..."):
            df = execute_query(queries[selected_query], db_config)
            
            if not df.empty:
                st.dataframe(df, use_container_width=True)
                
                # Auto-generate visualization
                if len(df.columns) == 2:
                    fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                                title=selected_query)
                    st.plotly_chart(fig, use_container_width=True)
            else:
                st.warning("No data returned")
```

### Visualization Tab

```python
def visualization_tab(db_config):
    st.header("Interactive Visualizations")
    
    # Culture distribution
    df_culture = execute_query(query_by_culture, db_config)
    if not df_culture.empty:
        fig1 = px.bar(df_culture, x='culture', y='artifact_count',
                     title='Top 10 Cultures by Artifact Count',
                     labels={'artifact_count': 'Number of Artifacts'})
        st.plotly_chart(fig1, use_container_width=True)
    
    # Color distribution
    df_colors = execute_query(query_top_colors, db_config)
    if not df_colors.empty:
        fig2 = px.pie(df_colors, names='color', values='frequency',
                     title='Color Distribution in Artifacts')
        st.plotly_chart(fig2, use_container_width=True)
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(api_key, num_pages, delay=1):
    """Fetch with rate limiting"""
    artifacts = []
    
    for page in range(1, num_pages + 1):
        response = requests.get(base_url, params={'apikey': api_key, 'page': page})
        
        if response.status_code == 200:
            artifacts.extend(response.json().get('records', []))
        
        time.sleep(delay)  # Rate limiting
    
    return artifacts
```

### Error Handling in ETL

```python
def safe_extract_metadata(artifacts):
    """Extract with error handling"""
    metadata = []
    
    for artifact in artifacts:
        try:
            metadata.append({
                'id': artifact.get('id'),
                'title': artifact.get('title', 'Unknown'),
                'culture': artifact.get('culture', 'Unknown'),
                # ... other fields
            })
        except Exception as e:
            print(f"Error processing artifact {artifact.get('id')}: {e}")
            continue
    
    return pd.DataFrame(metadata)
```

### Incremental Data Loading

```python
def load_incremental(df, table_name, db_config):
    """Load only new records"""
    connection = mysql.connector.connect(**db_config)
    cursor = connection.cursor()
    
    # Get existing IDs
    cursor.execute(f"SELECT id FROM {table_name}")
    existing_ids = set(row[0] for row in cursor.fetchall())
    
    # Filter new records
    new_records = df[~df['id'].isin(existing_ids)]
    
    if not new_records.empty:
        load_to_database(new_records, table_name, db_config)
        print(f"Loaded {len(new_records)} new records")
    else:
        print("No new records to load")
```

## Troubleshooting

### API Connection Issues

```python
# Test API connection
def test_api_connection(api_key):
    try:
        response = requests.get(
            "https://api.harvardartmuseums.org/object",
            params={'apikey': api_key, 'size': 1}
        )
        return response.status_code == 200
    except Exception as e:
        print(f"API connection failed: {e}")
        return False
```

### Database Connection Issues

```python
# Test database connection
def test_db_connection(db_config):
    try:
        connection = mysql.connector.connect(**db_config)
        if connection.is_connected():
            print("Database connected successfully")
            connection.close()
            return True
    except Error as e:
        print(f"Database connection failed: {e}")
        return False
```

### Handle Missing Data

```python
# Clean DataFrame before loading
def clean_dataframe(df):
    # Replace None with appropriate defaults
    df = df.fillna({
        'culture': 'Unknown',
        'century': 'Unknown',
        'department': 'Unspecified',
        'accession_year': 0
    })
    
    # Truncate long strings
    for col in df.select_dtypes(include=['object']).columns:
        df[col] = df[col].astype(str).str[:500]
    
    return df
```

This skill provides everything needed to build, configure, and extend the Harvard Artifacts ETL and Analytics application with proper data engineering practices.
