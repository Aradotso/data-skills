---
name: harvard-art-museums-etl-pipeline
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for museum data
  - connect to Harvard Art Museums API
  - create a data engineering pipeline with Streamlit
  - set up artifact collection analytics
  - build museum data visualization dashboard
  - implement art collection data warehouse
  - analyze Harvard museum artifacts with SQL
  - create interactive museum data dashboard
---

# Harvard Art Museums ETL Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering pipelines using the Harvard Art Museums API. The project demonstrates ETL operations, SQL database design, analytical queries, and interactive visualization using Streamlit.

## What It Does

The Harvard Artifacts Collection app provides:
- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads museum data into relational databases
- **SQL Analytics**: Predefined analytical queries for artifact insights
- **Interactive Dashboard**: Streamlit-based UI with Plotly visualizations
- **Database Design**: Normalized tables for artifact metadata, media, and color data

Architecture: `API → ETL → SQL → Analytics → Visualization`

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

Register at: https://www.harvardartmuseums.org/collections/api

### Database Setup

The application supports MySQL or TiDB Cloud. Create the database:

```sql
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

-- Tables are auto-created by the ETL pipeline
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API
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
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

def extract_artifact_metadata(artifacts):
    """
    Transform API response into structured dataframe
    """
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'object_id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'accession_year': artifact.get('accessionyear'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def extract_artifact_media(artifacts):
    """
    Extract media/image data from artifacts
    """
    media_list = []
    
    for artifact in artifacts:
        object_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            media = {
                'object_id': object_id,
                'image_id': img.get('imageid'),
                'base_url': img.get('baseimageurl'),
                'width': img.get('width'),
                'height': img.get('height'),
                'format': img.get('format'),
                'copyright': img.get('copyright')
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def extract_artifact_colors(artifacts):
    """
    Extract color data from artifacts
    """
    colors_list = []
    
    for artifact in artifacts:
        object_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data = {
                'object_id': object_id,
                'color_name': color.get('color'),
                'hex_value': color.get('hex'),
                'percentage': color.get('percent')
            }
            colors_list.append(color_data)
    
    return pd.DataFrame(colors_list)
```

### 3. Database Loading

```python
def create_database_connection():
    """
    Create MySQL database connection
    """
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def create_tables(connection):
    """
    Create database schema
    """
    cursor = connection.cursor()
    
    # Artifact metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            object_id INT PRIMARY KEY,
            title TEXT,
            culture VARCHAR(255),
            period VARCHAR(255),
            century VARCHAR(100),
            classification VARCHAR(255),
            department VARCHAR(255),
            division VARCHAR(255),
            dated VARCHAR(255),
            accession_year INT,
            technique TEXT,
            medium TEXT
        )
    """)
    
    # Artifact media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            object_id INT,
            image_id INT,
            base_url TEXT,
            width INT,
            height INT,
            format VARCHAR(50),
            copyright TEXT,
            FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
        )
    """)
    
    # Artifact colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            object_id INT,
            color_name VARCHAR(100),
            hex_value VARCHAR(10),
            percentage FLOAT,
            FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
        )
    """)
    
    connection.commit()
    cursor.close()

def load_dataframe_to_sql(df, table_name, connection):
    """
    Batch insert dataframe into SQL table
    """
    cursor = connection.cursor()
    
    # Prepare column names and placeholders
    cols = ','.join(df.columns)
    placeholders = ','.join(['%s'] * len(df.columns))
    
    insert_query = f"""
        INSERT IGNORE INTO {table_name} ({cols})
        VALUES ({placeholders})
    """
    
    # Batch insert
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    
    connection.commit()
    cursor.close()
    print(f"Loaded {len(df)} records into {table_name}")
```

### 4. SQL Analytics Queries

```python
# Sample analytical queries

ANALYTICS_QUERIES = {
    "Total Artifacts": """
        SELECT COUNT(*) as total_artifacts 
        FROM artifactmetadata
    """,
    
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Top Departments": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "Artifacts with Images": """
        SELECT 
            CASE WHEN m.object_id IS NOT NULL THEN 'With Images' 
                 ELSE 'Without Images' 
            END as image_status,
            COUNT(*) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.object_id = m.object_id
        GROUP BY image_status
    """,
    
    "Top Colors Used": """
        SELECT color_name, COUNT(*) as usage_count, AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color_name
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Accession Year": """
        SELECT accession_year, COUNT(*) as count
        FROM artifactmetadata
        WHERE accession_year IS NOT NULL
        GROUP BY accession_year
        ORDER BY accession_year DESC
        LIMIT 20
    """
}

def execute_analytics_query(query, connection):
    """
    Execute SQL query and return DataFrame
    """
    df = pd.read_sql(query, connection)
    return df
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Art Museums Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museums Data Analytics")
    st.markdown("End-to-end ETL and Analytics Pipeline")
    
    # Sidebar configuration
    with st.sidebar:
        st.header("Configuration")
        
        # API key input
        api_key = st.text_input(
            "Harvard API Key",
            value=os.getenv('HARVARD_API_KEY', ''),
            type="password"
        )
        
        # Data collection
        if st.button("Fetch New Data"):
            with st.spinner("Fetching artifacts..."):
                data = fetch_artifacts(api_key, page=1, size=100)
                artifacts = data.get('records', [])
                
                # ETL process
                metadata_df = extract_artifact_metadata(artifacts)
                media_df = extract_artifact_media(artifacts)
                colors_df = extract_artifact_colors(artifacts)
                
                # Load to database
                connection = create_database_connection()
                create_tables(connection)
                
                load_dataframe_to_sql(metadata_df, 'artifactmetadata', connection)
                load_dataframe_to_sql(media_df, 'artifactmedia', connection)
                load_dataframe_to_sql(colors_df, 'artifactcolors', connection)
                
                connection.close()
                st.success(f"Loaded {len(artifacts)} artifacts!")
    
    # Analytics section
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Run Query"):
        connection = create_database_connection()
        
        if connection:
            query = ANALYTICS_QUERIES[query_name]
            
            # Show SQL query
            with st.expander("View SQL Query"):
                st.code(query, language='sql')
            
            # Execute and display results
            df = execute_analytics_query(query, connection)
            
            col1, col2 = st.columns([1, 2])
            
            with col1:
                st.dataframe(df, use_container_width=True)
            
            with col2:
                if len(df.columns) == 2:
                    # Create visualization
                    fig = px.bar(
                        df,
                        x=df.columns[0],
                        y=df.columns[1],
                        title=query_name
                    )
                    st.plotly_chart(fig, use_container_width=True)
            
            connection.close()

if __name__ == "__main__":
    main()
```

## Common Patterns

### Full ETL Pipeline Execution

```python
def run_etl_pipeline(api_key, num_pages=5):
    """
    Complete ETL pipeline for multiple pages
    """
    connection = create_database_connection()
    create_tables(connection)
    
    all_metadata = []
    all_media = []
    all_colors = []
    
    for page in range(1, num_pages + 1):
        print(f"Processing page {page}...")
        
        # Extract
        data = fetch_artifacts(api_key, page=page, size=100)
        artifacts = data.get('records', [])
        
        # Transform
        metadata_df = extract_artifact_metadata(artifacts)
        media_df = extract_artifact_media(artifacts)
        colors_df = extract_artifact_colors(artifacts)
        
        all_metadata.append(metadata_df)
        all_media.append(media_df)
        all_colors.append(colors_df)
    
    # Combine all pages
    combined_metadata = pd.concat(all_metadata, ignore_index=True)
    combined_media = pd.concat(all_media, ignore_index=True)
    combined_colors = pd.concat(all_colors, ignore_index=True)
    
    # Load
    load_dataframe_to_sql(combined_metadata, 'artifactmetadata', connection)
    load_dataframe_to_sql(combined_media, 'artifactmedia', connection)
    load_dataframe_to_sql(combined_colors, 'artifactcolors', connection)
    
    connection.close()
    print(f"ETL complete: {len(combined_metadata)} artifacts processed")
```

### Data Quality Checks

```python
def validate_data_quality(connection):
    """
    Run data quality checks
    """
    checks = {
        "Orphaned Media Records": """
            SELECT COUNT(*) FROM artifactmedia 
            WHERE object_id NOT IN (SELECT object_id FROM artifactmetadata)
        """,
        "Null Titles": """
            SELECT COUNT(*) FROM artifactmetadata WHERE title IS NULL
        """,
        "Duplicate Object IDs": """
            SELECT object_id, COUNT(*) as count 
            FROM artifactmetadata 
            GROUP BY object_id 
            HAVING count > 1
        """
    }
    
    for check_name, query in checks.items():
        df = pd.read_sql(query, connection)
        print(f"{check_name}: {df.iloc[0, 0]}")
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(api_key, page, max_retries=3):
    """
    Fetch with exponential backoff
    """
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except Exception as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt
                print(f"Retry in {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise e
```

### Database Connection Issues

```python
def test_database_connection():
    """
    Verify database connectivity
    """
    try:
        connection = create_database_connection()
        if connection and connection.is_connected():
            print("✓ Database connected")
            connection.close()
            return True
        else:
            print("✗ Connection failed")
            return False
    except Exception as e:
        print(f"✗ Error: {e}")
        return False
```

### Missing Dependencies

```bash
# If Streamlit fails to start
pip install --upgrade streamlit

# If MySQL connector issues
pip install mysql-connector-python --force-reinstall

# For plotting issues
pip install plotly --upgrade
```

## Advanced Usage

### Custom Query Builder

```python
def build_custom_query(filters):
    """
    Dynamic query builder based on user filters
    """
    base_query = "SELECT * FROM artifactmetadata WHERE 1=1"
    
    if filters.get('culture'):
        base_query += f" AND culture = '{filters['culture']}'"
    
    if filters.get('century'):
        base_query += f" AND century = '{filters['century']}'"
    
    if filters.get('department'):
        base_query += f" AND department = '{filters['department']}'"
    
    return base_query
```

### Export Results

```python
def export_to_csv(query, connection, filename):
    """
    Export query results to CSV
    """
    df = pd.read_sql(query, connection)
    df.to_csv(filename, index=False)
    print(f"Exported to {filename}")
```

This skill provides comprehensive coverage for building museum data pipelines with Harvard's API, suitable for data engineering portfolios and real-world analytics applications.
