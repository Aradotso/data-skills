---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard from museum artifacts API
  - extract and visualize Harvard Art Museums collection data
  - set up data engineering pipeline with Streamlit
  - query and analyze museum artifacts with SQL
  - transform Harvard API data into relational database
  - build museum collection analytics application
  - create art museum data visualization dashboard
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates extracting artifact data via API, transforming it into relational structures, loading into SQL databases, and creating interactive Streamlit dashboards with SQL analytics and visualizations.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides:
- **API Integration**: Fetch artifact metadata from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load nested JSON data into normalized SQL tables
- **SQL Analytics**: Execute predefined analytical queries on artifact collections
- **Interactive Dashboard**: Streamlit-based UI with Plotly visualizations
- **Database Design**: Relational schema for artifacts, media, and color data

**Architecture Flow**: API → ETL → SQL Database → Analytics Queries → Streamlit Visualization

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

### Environment Variables

Set up your credentials using environment variables:

```bash
# Harvard Art Museums API
export HARVARD_API_KEY="your_api_key_here"

# MySQL/TiDB Database
export DB_HOST="your_db_host"
export DB_USER="your_db_user"
export DB_PASSWORD="your_db_password"
export DB_NAME="harvard_artifacts"
export DB_PORT="3306"
```

### Database Setup

Create the required tables:

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    technique VARCHAR(255),
    dated VARCHAR(255),
    url VARCHAR(500),
    creditline TEXT,
    division VARCHAR(255)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    primaryimageurl VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## API Integration

### Fetching Data from Harvard Art Museums API

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    
    Args:
        api_key: Harvard API key from environment
        page: Page number for pagination
        size: Number of records per page (max 100)
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=100)
artifacts = data.get('records', [])
total_records = data.get('info', {}).get('totalrecords', 0)
```

### Handling Pagination

```python
import time

def fetch_all_artifacts(api_key, max_records=1000):
    """Fetch multiple pages of artifacts with rate limiting"""
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        try:
            data = fetch_artifacts(api_key, page=page, size=size)
            records = data.get('records', [])
            
            if not records:
                break
                
            all_artifacts.extend(records)
            page += 1
            
            # Rate limiting - be respectful to API
            time.sleep(0.5)
            
        except Exception as e:
            print(f"Error on page {page}: {e}")
            break
    
    return all_artifacts[:max_records]
```

## ETL Pipeline

### Extract: Parse API Response

```python
import pandas as pd

def extract_metadata(artifacts):
    """Extract metadata from API response"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'technique': artifact.get('technique'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url'),
            'creditline': artifact.get('creditline'),
            'division': artifact.get('division')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)
```

### Transform: Normalize Nested Data

```python
def extract_media(artifacts):
    """Extract media information into separate table"""
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        if images:
            media = {
                'artifact_id': artifact_id,
                'baseimageurl': images[0].get('baseimageurl'),
                'iiifbaseuri': images[0].get('iiifbaseuri'),
                'primaryimageurl': artifact.get('primaryimageurl')
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def extract_colors(artifacts):
    """Extract color data into separate table"""
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'percentage': color.get('percent')
            }
            color_list.append(color_data)
    
    return pd.DataFrame(color_list)
```

### Load: Insert into Database

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection using environment variables"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME'),
        port=int(os.getenv('DB_PORT', 3306))
    )

def load_metadata(df_metadata):
    """Load metadata into database with batch insert"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, department, 
         technique, dated, url, creditline, division)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    data = [tuple(row) for row in df_metadata.values]
    cursor.executemany(insert_query, data)
    conn.commit()
    
    cursor.close()
    conn.close()

def load_media(df_media):
    """Load media data into database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, iiifbaseuri, primaryimageurl)
        VALUES (%s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df_media.values]
    cursor.executemany(insert_query, data)
    conn.commit()
    
    cursor.close()
    conn.close()
```

### Complete ETL Function

```python
def run_etl_pipeline(num_records=500):
    """Run complete ETL pipeline"""
    print("Starting ETL Pipeline...")
    
    # Extract
    api_key = os.getenv('HARVARD_API_KEY')
    artifacts = fetch_all_artifacts(api_key, max_records=num_records)
    print(f"Extracted {len(artifacts)} artifacts")
    
    # Transform
    df_metadata = extract_metadata(artifacts)
    df_media = extract_media(artifacts)
    df_colors = extract_colors(artifacts)
    print("Data transformation complete")
    
    # Load
    load_metadata(df_metadata)
    load_media(df_media)
    load_colors(df_colors)
    print("Data loaded into database")
    
    return {
        'metadata_count': len(df_metadata),
        'media_count': len(df_media),
        'colors_count': len(df_colors)
    }
```

## SQL Analytics Queries

### Common Analytical Queries

```python
# Top 10 cultures by artifact count
query_cultures = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Artifacts by century
query_century = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY count DESC
"""

# Department distribution
query_departments = """
    SELECT department, COUNT(*) as total_artifacts
    FROM artifactmetadata
    GROUP BY department
    ORDER BY total_artifacts DESC
"""

# Color analysis
query_colors = """
    SELECT color, 
           COUNT(*) as occurrence_count,
           AVG(percentage) as avg_percentage
    FROM artifactcolors
    GROUP BY color
    ORDER BY occurrence_count DESC
    LIMIT 15
"""

# Artifacts with images
query_media_availability = """
    SELECT 
        COUNT(DISTINCT m.artifact_id) as with_media,
        COUNT(DISTINCT a.id) as total_artifacts,
        ROUND(COUNT(DISTINCT m.artifact_id) * 100.0 / COUNT(DISTINCT a.id), 2) as percentage
    FROM artifactmetadata a
    LEFT JOIN artifactmedia m ON a.id = m.artifact_id
"""
```

### Execute Query Function

```python
def execute_query(query):
    """Execute SQL query and return results as DataFrame"""
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# Usage
df_cultures = execute_query(query_cultures)
print(df_cultures)
```

## Streamlit Dashboard

### Basic Dashboard Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Art Analytics", layout="wide")
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar for configuration
    with st.sidebar:
        st.header("Configuration")
        
        # ETL Controls
        if st.button("Run ETL Pipeline"):
            with st.spinner("Running ETL..."):
                results = run_etl_pipeline(num_records=500)
                st.success(f"Loaded {results['metadata_count']} artifacts")
    
    # Main content
    tab1, tab2, tab3 = st.tabs(["Analytics", "Visualizations", "Data Explorer"])
    
    with tab1:
        display_analytics()
    
    with tab2:
        display_visualizations()
    
    with tab3:
        display_data_explorer()

if __name__ == "__main__":
    main()
```

### Analytics Tab

```python
def display_analytics():
    st.header("SQL Analytics Queries")
    
    queries = {
        "Top Cultures": query_cultures,
        "Century Distribution": query_century,
        "Department Analysis": query_departments,
        "Color Patterns": query_colors
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        with st.spinner("Executing query..."):
            df = execute_query(queries[selected_query])
            
            st.subheader("Results")
            st.dataframe(df, use_container_width=True)
            
            # Auto-generate visualization
            if len(df.columns) >= 2:
                fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                            title=f"{selected_query} Analysis")
                st.plotly_chart(fig, use_container_width=True)
```

### Visualization Tab

```python
def display_visualizations():
    st.header("Interactive Visualizations")
    
    col1, col2 = st.columns(2)
    
    with col1:
        st.subheader("Artifacts by Culture")
        df_culture = execute_query(query_cultures)
        fig = px.bar(df_culture, x='culture', y='artifact_count',
                    title="Top 10 Cultures")
        st.plotly_chart(fig, use_container_width=True)
    
    with col2:
        st.subheader("Color Distribution")
        df_colors = execute_query(query_colors)
        fig = px.pie(df_colors, names='color', values='occurrence_count',
                    title="Color Usage Across Artifacts")
        st.plotly_chart(fig, use_container_width=True)
    
    # Time series
    st.subheader("Artifacts by Century")
    df_century = execute_query(query_century)
    fig = px.line(df_century, x='century', y='count',
                 title="Artifact Distribution Over Time")
    st.plotly_chart(fig, use_container_width=True)
```

### Data Explorer Tab

```python
def display_data_explorer():
    st.header("Raw Data Explorer")
    
    table_name = st.selectbox("Select Table", 
                             ["artifactmetadata", "artifactmedia", "artifactcolors"])
    
    limit = st.slider("Number of rows", 10, 500, 100)
    
    query = f"SELECT * FROM {table_name} LIMIT {limit}"
    df = execute_query(query)
    
    st.dataframe(df, use_container_width=True)
    
    # Download option
    csv = df.to_csv(index=False)
    st.download_button(
        label="Download CSV",
        data=csv,
        file_name=f"{table_name}_export.csv",
        mime="text/csv"
    )
```

## Running the Application

```bash
# Set environment variables first
export HARVARD_API_KEY="your_key"
export DB_HOST="localhost"
export DB_USER="root"
export DB_PASSWORD="password"
export DB_NAME="harvard_artifacts"

# Run Streamlit app
streamlit run app.py
```

## Common Patterns

### Rate-Limited API Calls

```python
from functools import wraps
import time

def rate_limit(calls=10, period=1):
    """Decorator to rate limit API calls"""
    def decorator(func):
        last_called = [0.0]
        
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            wait_time = period / calls
            
            if elapsed < wait_time:
                time.sleep(wait_time - elapsed)
            
            last_called[0] = time.time()
            return func(*args, **kwargs)
        
        return wrapper
    return decorator

@rate_limit(calls=5, period=1)
def fetch_artifact_details(artifact_id):
    """Fetch single artifact with rate limiting"""
    api_key = os.getenv('HARVARD_API_KEY')
    url = f"https://api.harvardartmuseums.org/object/{artifact_id}"
    response = requests.get(url, params={'apikey': api_key})
    return response.json()
```

### Incremental ETL Updates

```python
def get_last_processed_id():
    """Get the highest artifact ID already in database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    conn.close()
    return result[0] if result[0] else 0

def incremental_etl():
    """Only fetch and load new artifacts"""
    last_id = get_last_processed_id()
    
    api_key = os.getenv('HARVARD_API_KEY')
    params = {
        'apikey': api_key,
        'size': 100,
        'after': last_id  # Only get artifacts after this ID
    }
    
    response = requests.get(
        "https://api.harvardartmuseums.org/object",
        params=params
    )
    
    new_artifacts = response.json().get('records', [])
    
    if new_artifacts:
        df_metadata = extract_metadata(new_artifacts)
        load_metadata(df_metadata)
        print(f"Loaded {len(new_artifacts)} new artifacts")
```

## Troubleshooting

### API Connection Issues

```python
def test_api_connection():
    """Verify API key and connection"""
    api_key = os.getenv('HARVARD_API_KEY')
    
    if not api_key:
        return "ERROR: HARVARD_API_KEY not set in environment"
    
    try:
        response = requests.get(
            "https://api.harvardartmuseums.org/object",
            params={'apikey': api_key, 'size': 1},
            timeout=10
        )
        
        if response.status_code == 200:
            return "✓ API connection successful"
        elif response.status_code == 401:
            return "ERROR: Invalid API key"
        else:
            return f"ERROR: HTTP {response.status_code}"
            
    except requests.exceptions.Timeout:
        return "ERROR: Request timeout"
    except Exception as e:
        return f"ERROR: {str(e)}"
```

### Database Connection Issues

```python
def test_db_connection():
    """Verify database connection and tables"""
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        
        # Check tables exist
        cursor.execute("SHOW TABLES")
        tables = [table[0] for table in cursor.fetchall()]
        
        required_tables = ['artifactmetadata', 'artifactmedia', 'artifactcolors']
        missing = [t for t in required_tables if t not in tables]
        
        if missing:
            return f"ERROR: Missing tables: {', '.join(missing)}"
        
        cursor.close()
        conn.close()
        return "✓ Database connection successful"
        
    except Error as e:
        return f"ERROR: {str(e)}"
```

### Handling Missing Data

```python
def safe_extract_metadata(artifact):
    """Extract metadata with defaults for missing fields"""
    return {
        'id': artifact.get('id'),
        'title': artifact.get('title', 'Untitled'),
        'culture': artifact.get('culture', 'Unknown'),
        'period': artifact.get('period'),
        'century': artifact.get('century', 'Unknown'),
        'classification': artifact.get('classification', 'Unclassified'),
        'department': artifact.get('department', 'Unknown'),
        'technique': artifact.get('technique'),
        'dated': artifact.get('dated'),
        'url': artifact.get('url'),
        'creditline': artifact.get('creditline'),
        'division': artifact.get('division')
    }
```

This skill provides comprehensive coverage of building ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit.
