---
name: harvard-art-museums-data-pipeline
description: Build end-to-end ETL pipelines and analytics applications using the Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build a data pipeline for Harvard Art Museums API
  - create ETL workflow for museum artifacts data
  - set up Harvard Art Museums analytics dashboard
  - extract and visualize museum collection data
  - implement artifact data engineering pipeline
  - query Harvard Art Museums API with SQL analytics
  - develop Streamlit app for museum data visualization
  - design relational database for museum artifacts
---

# Harvard Art Museums Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Harvard Artifacts Collection Data Engineering Analytics App is a complete ETL pipeline and analytics solution for museum artifact data. It demonstrates production-grade data engineering patterns including API pagination, relational database design, batch processing, and interactive visualization using Streamlit.

**Core capabilities:**
- Extract artifact metadata, media, and color data from Harvard Art Museums API
- Transform nested JSON into normalized relational tables
- Load data into MySQL/TiDB Cloud with proper foreign key relationships
- Execute analytical SQL queries and visualize results
- Build interactive dashboards with Plotly charts

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

### Environment Variables

Create a `.env` file in the project root:

```env
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_db_host
DB_PORT=4000
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

**Get your Harvard API key:**
- Register at https://www.harvardartmuseums.org/collections/api
- Free tier allows 2500 requests/day

### Database Setup

```sql
-- Create database
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    accession_number VARCHAR(100),
    url VARCHAR(500)
);

-- Artifact media table
CREATE TABLE artifactmedia (
    artifact_id INT,
    media_id INT,
    image_url VARCHAR(500),
    alt_text TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    artifact_id INT,
    color_hex VARCHAR(10),
    color_name VARCHAR(100),
    color_percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core API Integration

### Fetching Artifacts with Pagination

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_records=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    
    Args:
        num_records: Total number of records to fetch
        
    Returns:
        List of artifact dictionaries
    """
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    artifacts = []
    page = 1
    size = 100  # API allows max 100 per request
    
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

### Rate Limiting and Error Handling

```python
import time
from requests.exceptions import RequestException

def fetch_with_retry(url, params, max_retries=3, delay=2):
    """Fetch data with exponential backoff retry logic"""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            
            if response.status_code == 429:  # Rate limit
                wait_time = delay * (2 ** attempt)
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
                continue
                
            response.raise_for_status()
            return response.json()
            
        except RequestException as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(delay)
    
    return None
```

## ETL Pipeline Implementation

### Extract Phase

```python
def extract_artifact_data(artifact):
    """Extract relevant fields from raw artifact JSON"""
    return {
        'id': artifact.get('id'),
        'title': artifact.get('title', 'Untitled'),
        'culture': artifact.get('culture'),
        'period': artifact.get('period'),
        'century': artifact.get('century'),
        'classification': artifact.get('classification'),
        'department': artifact.get('department'),
        'dated': artifact.get('dated'),
        'accession_number': artifact.get('accessionnumber'),
        'url': artifact.get('url')
    }

def extract_media_data(artifact):
    """Extract media information from artifact"""
    media_records = []
    images = artifact.get('images', [])
    
    for img in images:
        media_records.append({
            'artifact_id': artifact.get('id'),
            'media_id': img.get('imageid'),
            'image_url': img.get('baseimageurl'),
            'alt_text': img.get('alttext')
        })
    
    return media_records

def extract_color_data(artifact):
    """Extract color information from artifact"""
    color_records = []
    colors = artifact.get('colors', [])
    
    for color in colors:
        color_records.append({
            'artifact_id': artifact.get('id'),
            'color_hex': color.get('hex'),
            'color_name': color.get('color'),
            'color_percentage': color.get('percent')
        })
    
    return color_records
```

### Transform Phase

```python
import pandas as pd

def transform_artifacts_to_dataframes(artifacts):
    """
    Transform raw artifact list into structured DataFrames
    
    Returns:
        Tuple of (metadata_df, media_df, colors_df)
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Extract and append metadata
        metadata_list.append(extract_artifact_data(artifact))
        
        # Extract and append media
        media_list.extend(extract_media_data(artifact))
        
        # Extract and append colors
        colors_list.extend(extract_color_data(artifact))
    
    # Create DataFrames
    metadata_df = pd.DataFrame(metadata_list)
    media_df = pd.DataFrame(media_list)
    colors_df = pd.DataFrame(colors_list)
    
    # Clean and normalize
    metadata_df['title'] = metadata_df['title'].str[:500]  # Truncate
    metadata_df = metadata_df.fillna('')  # Handle nulls
    
    return metadata_df, media_df, colors_df
```

### Load Phase

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_metadata(df, connection):
    """Batch insert metadata records"""
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, 
         department, dated, accession_number, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title = VALUES(title),
        culture = VALUES(culture),
        period = VALUES(period)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    
    print(f"Loaded {cursor.rowcount} metadata records")

def load_media(df, connection):
    """Batch insert media records"""
    cursor = connection.cursor()
    
    # Clear existing media for these artifacts
    artifact_ids = df['artifact_id'].unique().tolist()
    placeholders = ','.join(['%s'] * len(artifact_ids))
    cursor.execute(
        f"DELETE FROM artifactmedia WHERE artifact_id IN ({placeholders})",
        artifact_ids
    )
    
    insert_query = """
        INSERT INTO artifactmedia 
        (artifact_id, media_id, image_url, alt_text)
        VALUES (%s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    
    print(f"Loaded {cursor.rowcount} media records")

def load_colors(df, connection):
    """Batch insert color records"""
    cursor = connection.cursor()
    
    # Clear existing colors for these artifacts
    artifact_ids = df['artifact_id'].unique().tolist()
    placeholders = ','.join(['%s'] * len(artifact_ids))
    cursor.execute(
        f"DELETE FROM artifactcolors WHERE artifact_id IN ({placeholders})",
        artifact_ids
    )
    
    insert_query = """
        INSERT INTO artifactcolors 
        (artifact_id, color_hex, color_name, color_percentage)
        VALUES (%s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    
    print(f"Loaded {cursor.rowcount} color records")
```

### Complete ETL Workflow

```python
def run_etl_pipeline(num_records=100):
    """Execute complete ETL pipeline"""
    print(f"Starting ETL for {num_records} artifacts...")
    
    # Extract
    print("Extracting data from API...")
    artifacts = fetch_artifacts(num_records)
    print(f"Extracted {len(artifacts)} artifacts")
    
    # Transform
    print("Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts_to_dataframes(artifacts)
    print(f"Transformed: {len(metadata_df)} metadata, {len(media_df)} media, {len(colors_df)} colors")
    
    # Load
    print("Loading data to database...")
    connection = get_db_connection()
    
    try:
        load_metadata(metadata_df, connection)
        load_media(media_df, connection)
        load_colors(colors_df, connection)
        print("ETL completed successfully!")
    finally:
        connection.close()
```

## SQL Analytics Queries

### Analytical Query Examples

```python
# Top 10 cultures by artifact count
query_cultures = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL AND culture != ''
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Artifacts with most colors
query_colorful = """
    SELECT m.id, m.title, COUNT(c.color_hex) as color_count
    FROM artifactmetadata m
    LEFT JOIN artifactcolors c ON m.id = c.artifact_id
    GROUP BY m.id, m.title
    ORDER BY color_count DESC
    LIMIT 10
"""

# Department distribution
query_departments = """
    SELECT department, COUNT(*) as count
    FROM artifactmetadata
    WHERE department IS NOT NULL
    GROUP BY department
    ORDER BY count DESC
"""

# Dominant color analysis
query_dominant_colors = """
    SELECT color_name, COUNT(*) as usage_count,
           AVG(color_percentage) as avg_percentage
    FROM artifactcolors
    WHERE color_name IS NOT NULL
    GROUP BY color_name
    ORDER BY usage_count DESC
    LIMIT 15
"""

# Media availability rate
query_media_stats = """
    SELECT 
        COUNT(DISTINCT m.id) as total_artifacts,
        COUNT(DISTINCT med.artifact_id) as artifacts_with_media,
        ROUND(COUNT(DISTINCT med.artifact_id) * 100.0 / COUNT(DISTINCT m.id), 2) as media_percentage
    FROM artifactmetadata m
    LEFT JOIN artifactmedia med ON m.id = med.artifact_id
"""
```

### Query Execution Function

```python
def execute_analytics_query(query, connection):
    """Execute SQL query and return DataFrame"""
    try:
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        print(f"Query error: {e}")
        return pd.DataFrame()
```

## Streamlit Dashboard Implementation

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Art Museums Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museums Data Pipeline")
    st.markdown("---")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Navigation",
        ["ETL Pipeline", "SQL Analytics", "Data Exploration"]
    )
    
    if page == "ETL Pipeline":
        show_etl_page()
    elif page == "SQL Analytics":
        show_analytics_page()
    else:
        show_exploration_page()

if __name__ == "__main__":
    main()
```

### ETL Dashboard Page

```python
def show_etl_page():
    st.header("Data Collection & ETL")
    
    col1, col2 = st.columns(2)
    
    with col1:
        num_records = st.number_input(
            "Number of artifacts to collect",
            min_value=10,
            max_value=1000,
            value=100,
            step=10
        )
    
    with col2:
        st.write("")
        st.write("")
        if st.button("Run ETL Pipeline", type="primary"):
            with st.spinner("Running ETL..."):
                try:
                    run_etl_pipeline(num_records)
                    st.success(f"Successfully processed {num_records} artifacts!")
                except Exception as e:
                    st.error(f"ETL failed: {str(e)}")
```

### Analytics Dashboard Page

```python
def show_analytics_page():
    st.header("SQL Analytics Dashboard")
    
    # Query selector
    queries = {
        "Top Cultures": query_cultures,
        "Most Colorful Artifacts": query_colorful,
        "Department Distribution": query_departments,
        "Dominant Colors": query_dominant_colors,
        "Media Statistics": query_media_stats
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Execute Query"):
        connection = get_db_connection()
        
        try:
            df = execute_analytics_query(queries[selected_query], connection)
            
            # Display results
            st.subheader("Query Results")
            st.dataframe(df, use_container_width=True)
            
            # Auto-generate visualization
            if len(df) > 0 and len(df.columns) >= 2:
                st.subheader("Visualization")
                
                # Determine chart type based on data
                x_col = df.columns[0]
                y_col = df.columns[1]
                
                if df[y_col].dtype in ['int64', 'float64']:
                    fig = px.bar(df, x=x_col, y=y_col, 
                                title=selected_query)
                    st.plotly_chart(fig, use_container_width=True)
                    
        finally:
            connection.close()
```

### Interactive Visualization

```python
def create_interactive_chart(df, chart_type, x_col, y_col, title):
    """Create Plotly interactive charts"""
    if chart_type == "bar":
        fig = px.bar(df, x=x_col, y=y_col, title=title,
                    color=y_col, color_continuous_scale='Viridis')
    elif chart_type == "pie":
        fig = px.pie(df, names=x_col, values=y_col, title=title)
    elif chart_type == "scatter":
        fig = px.scatter(df, x=x_col, y=y_col, title=title,
                        size=y_col, hover_data=df.columns)
    
    fig.update_layout(
        hovermode='x unified',
        template='plotly_white'
    )
    
    return fig
```

## Common Patterns

### Incremental Data Loading

```python
def get_last_artifact_id(connection):
    """Get the highest artifact ID already in database"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def fetch_new_artifacts_only():
    """Fetch only artifacts newer than what's in database"""
    connection = get_db_connection()
    last_id = get_last_artifact_id(connection)
    connection.close()
    
    # Fetch with filter
    params = {
        'apikey': os.getenv('HARVARD_API_KEY'),
        'size': 100,
        'sort': 'id',
        'sortorder': 'asc',
        'after': last_id
    }
    
    # Continue fetching logic...
```

### Data Quality Validation

```python
def validate_dataframe(df, table_name):
    """Validate DataFrame before loading"""
    issues = []
    
    # Check for required columns
    required_cols = {
        'artifactmetadata': ['id', 'title'],
        'artifactmedia': ['artifact_id', 'image_url'],
        'artifactcolors': ['artifact_id', 'color_hex']
    }
    
    for col in required_cols.get(table_name, []):
        if col not in df.columns:
            issues.append(f"Missing required column: {col}")
    
    # Check for duplicates
    if 'id' in df.columns:
        dupes = df['id'].duplicated().sum()
        if dupes > 0:
            issues.append(f"Found {dupes} duplicate IDs")
    
    # Check data types
    if 'id' in df.columns and df['id'].dtype not in ['int64', 'Int64']:
        issues.append("ID column must be integer")
    
    return issues
```

## Troubleshooting

### API Rate Limiting

```python
# Implement exponential backoff
import time

def safe_api_call(url, params, max_wait=60):
    """Handle rate limiting gracefully"""
    wait_time = 1
    
    while wait_time <= max_wait:
        response = requests.get(url, params=params)
        
        if response.status_code == 200:
            return response.json()
        elif response.status_code == 429:
            st.warning(f"Rate limited. Waiting {wait_time}s...")
            time.sleep(wait_time)
            wait_time *= 2
        else:
            raise Exception(f"API error: {response.status_code}")
    
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
def test_database_connection():
    """Test and diagnose database connectivity"""
    try:
        connection = get_db_connection()
        cursor = connection.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        connection.close()
        
        if result[0] == 1:
            st.success("✅ Database connection successful")
            return True
    except Error as e:
        st.error(f"❌ Database connection failed: {e}")
        st.info("Check your .env file and database credentials")
        return False
```

### Memory Management for Large Datasets

```python
def process_in_batches(artifacts, batch_size=50):
    """Process large datasets in batches to avoid memory issues"""
    connection = get_db_connection()
    
    for i in range(0, len(artifacts), batch_size):
        batch = artifacts[i:i+batch_size]
        
        # Transform batch
        metadata_df, media_df, colors_df = transform_artifacts_to_dataframes(batch)
        
        # Load batch
        load_metadata(metadata_df, connection)
        load_media(media_df, connection)
        load_colors(colors_df, connection)
        
        print(f"Processed batch {i//batch_size + 1}")
    
    connection.close()
```

### Handling Missing Data

```python
def clean_artifact_data(df):
    """Handle missing and malformed data"""
    # Replace empty strings with None for database NULL
    df = df.replace('', None)
    
    # Handle specific fields
    if 'title' in df.columns:
        df['title'] = df['title'].fillna('Untitled')
    
    if 'century' in df.columns:
        df['century'] = df['century'].str.replace('century', '').str.strip()
    
    # Remove records with missing critical fields
    if 'id' in df.columns:
        df = df.dropna(subset=['id'])
    
    return df
```

This skill provides comprehensive coverage of building production-grade data pipelines using the Harvard Art Museums API, including ETL workflows, SQL analytics, and interactive Streamlit dashboards.
