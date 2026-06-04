---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for art museum data
  - create analytics dashboard with Harvard Art Museums API
  - set up data engineering pipeline for museum artifacts
  - extract and transform art collection data
  - build Streamlit app for museum data visualization
  - query and analyze Harvard art museums collection
  - create SQL database for artifact metadata
  - visualize museum artifact statistics
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualization using Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides a complete data pipeline:

1. **Extract**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
2. **Transform**: Convert nested JSON into relational database structures
3. **Load**: Batch insert data into MySQL/TiDB Cloud databases
4. **Analyze**: Execute predefined SQL analytics queries
5. **Visualize**: Display results in interactive Streamlit dashboards with Plotly charts

The application handles three main data entities:
- **Artifact Metadata**: Core information about museum objects
- **Artifact Media**: Images and media files associated with artifacts
- **Artifact Colors**: Color palette data extracted from artifact images

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
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Request an API key (free)
3. Add to your `.env` file

## Database Setup

### Create Database Schema

```sql
-- Create database
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
USE harvard_artifacts;

-- Artifact Metadata table
CREATE TABLE IF NOT EXISTS artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    people TEXT,
    creditline TEXT,
    accession_number VARCHAR(100),
    url VARCHAR(500),
    rank INT,
    verification_level INT
);

-- Artifact Media table
CREATE TABLE IF NOT EXISTS artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    base_image_url VARCHAR(500),
    primary_image_url VARCHAR(500),
    image_count INT,
    video_count INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors table
CREATE TABLE IF NOT EXISTS artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(7),
    color_name VARCHAR(100),
    color_percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_records=100, page_size=100):
    """
    Fetch artifact data from Harvard Art Museums API
    
    Args:
        num_records: Total number of records to fetch
        page_size: Number of records per API call (max 100)
    
    Returns:
        List of artifact dictionaries
    """
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    all_artifacts = []
    page = 1
    
    while len(all_artifacts) < num_records:
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page,
            'hasimage': 1  # Only fetch artifacts with images
        }
        
        try:
            response = requests.get(base_url, params=params)
            response.raise_for_status()
            data = response.json()
            
            records = data.get('records', [])
            if not records:
                break
                
            all_artifacts.extend(records)
            page += 1
            
            # Rate limiting
            import time
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching data: {e}")
            break
    
    return all_artifacts[:num_records]
```

### 2. Data Transformation

```python
import pandas as pd

def transform_metadata(artifacts):
    """Transform raw API data into metadata DataFrame"""
    metadata = []
    
    for artifact in artifacts:
        # Extract people names
        people = ', '.join([p.get('name', '') for p in artifact.get('people', [])])
        
        record = {
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', ''),
            'century': artifact.get('century', ''),
            'dated': artifact.get('dated', ''),
            'classification': artifact.get('classification', ''),
            'department': artifact.get('department', ''),
            'division': artifact.get('division', ''),
            'technique': artifact.get('technique', '')[:500],
            'medium': artifact.get('medium', '')[:500],
            'dimensions': artifact.get('dimensions', '')[:500],
            'people': people,
            'creditline': artifact.get('creditline', ''),
            'accession_number': artifact.get('accessionnumber', ''),
            'url': artifact.get('url', ''),
            'rank': artifact.get('rank', 0),
            'verification_level': artifact.get('verificationlevel', 0)
        }
        metadata.append(record)
    
    return pd.DataFrame(metadata)

def transform_media(artifacts):
    """Transform media data into DataFrame"""
    media = []
    
    for artifact in artifacts:
        record = {
            'artifact_id': artifact.get('id'),
            'base_image_url': artifact.get('baseimageurl', ''),
            'primary_image_url': artifact.get('primaryimageurl', ''),
            'image_count': artifact.get('totalpageviews', 0),
            'video_count': artifact.get('totalvideos', 0)
        }
        media.append(record)
    
    return pd.DataFrame(media)

def transform_colors(artifacts):
    """Transform color data into DataFrame"""
    colors = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        color_data = artifact.get('colors', [])
        
        for color in color_data:
            record = {
                'artifact_id': artifact_id,
                'color_hex': color.get('hex', ''),
                'color_name': color.get('color', ''),
                'color_percentage': color.get('percent', 0.0)
            }
            colors.append(record)
    
    return pd.DataFrame(colors)
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
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

def load_metadata(df):
    """Batch insert metadata into database"""
    connection = get_db_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, dated, classification, department, 
     division, technique, medium, dimensions, people, creditline, 
     accession_number, url, rank, verification_level)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    try:
        data = [tuple(row) for row in df.values]
        cursor.executemany(insert_query, data)
        connection.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
        return True
    except Error as e:
        print(f"Error loading metadata: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()

def load_media(df):
    """Batch insert media data"""
    connection = get_db_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (artifact_id, base_image_url, primary_image_url, image_count, video_count)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    try:
        data = [tuple(row) for row in df.values]
        cursor.executemany(insert_query, data)
        connection.commit()
        print(f"Inserted {cursor.rowcount} media records")
        return True
    except Error as e:
        print(f"Error loading media: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()

def load_colors(df):
    """Batch insert color data"""
    connection = get_db_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors 
    (artifact_id, color_hex, color_name, color_percentage)
    VALUES (%s, %s, %s, %s)
    """
    
    try:
        data = [tuple(row) for row in df.values]
        cursor.executemany(insert_query, data)
        connection.commit()
        print(f"Inserted {cursor.rowcount} color records")
        return True
    except Error as e:
        print(f"Error loading colors: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()
```

### 4. Analytics Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "Artifacts by Department": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL AND department != ''
        GROUP BY department
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Top Cultures": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
        LIMIT 20
    """,
    
    "Media Availability": """
        SELECT 
            CASE 
                WHEN image_count > 0 THEN 'With Images'
                ELSE 'No Images'
            END as media_status,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY media_status
    """,
    
    "Top Colors": """
        SELECT color_name, COUNT(*) as usage_count, 
               AVG(color_percentage) as avg_percentage
        FROM artifactcolors
        WHERE color_name IS NOT NULL
        GROUP BY color_name
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "Classification Distribution": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL AND classification != ''
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 15
    """
}

def execute_query(query):
    """Execute SQL query and return results as DataFrame"""
    connection = get_db_connection()
    if not connection:
        return None
    
    try:
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        print(f"Query execution error: {e}")
        return None
    finally:
        connection.close()
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
    
    st.title("🎨 Harvard Art Museums Data Engineering & Analytics")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["ETL Pipeline", "Analytics Dashboard", "Data Explorer"]
    )
    
    if page == "ETL Pipeline":
        show_etl_page()
    elif page == "Analytics Dashboard":
        show_analytics_page()
    else:
        show_explorer_page()

def show_etl_page():
    st.header("📥 ETL Pipeline")
    
    num_records = st.number_input(
        "Number of records to fetch",
        min_value=10,
        max_value=1000,
        value=100,
        step=10
    )
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            artifacts = fetch_artifacts(num_records)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df = transform_metadata(artifacts)
            media_df = transform_media(artifacts)
            colors_df = transform_colors(artifacts)
            st.success("Data transformation complete")
        
        with st.spinner("Loading to database..."):
            load_metadata(metadata_df)
            load_media(media_df)
            load_colors(colors_df)
            st.success("Data loaded successfully!")
        
        # Display sample data
        st.subheader("Sample Metadata")
        st.dataframe(metadata_df.head())

def show_analytics_page():
    st.header("📊 Analytics Dashboard")
    
    query_name = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Run Query"):
        query = ANALYTICS_QUERIES[query_name]
        
        with st.spinner("Executing query..."):
            df = execute_query(query)
        
        if df is not None and not df.empty:
            st.subheader("Results")
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df.columns) == 2:
                fig = px.bar(
                    df,
                    x=df.columns[0],
                    y=df.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig, use_container_width=True)
        else:
            st.warning("No data returned from query")

def show_explorer_page():
    st.header("🔍 Data Explorer")
    
    # Custom SQL query interface
    custom_query = st.text_area(
        "Enter SQL Query",
        height=150,
        placeholder="SELECT * FROM artifactmetadata LIMIT 10"
    )
    
    if st.button("Execute Custom Query"):
        if custom_query.strip():
            df = execute_query(custom_query)
            if df is not None:
                st.dataframe(df)
                st.download_button(
                    "Download CSV",
                    df.to_csv(index=False),
                    "query_results.csv",
                    "text/csv"
                )

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# The app will open at http://localhost:8501
```

## Common Patterns

### Complete ETL Workflow

```python
# Full pipeline execution
def run_complete_etl(num_records=100):
    """Execute complete ETL pipeline"""
    # Extract
    print("Step 1: Extracting data...")
    artifacts = fetch_artifacts(num_records)
    
    # Transform
    print("Step 2: Transforming data...")
    metadata_df = transform_metadata(artifacts)
    media_df = transform_media(artifacts)
    colors_df = transform_colors(artifacts)
    
    # Load
    print("Step 3: Loading data...")
    success = (
        load_metadata(metadata_df) and
        load_media(media_df) and
        load_colors(colors_df)
    )
    
    if success:
        print("ETL pipeline completed successfully!")
        return True
    else:
        print("ETL pipeline failed")
        return False
```

### Incremental Data Loading

```python
def incremental_load(last_loaded_id=0):
    """Load only new artifacts since last run"""
    connection = get_db_connection()
    cursor = connection.cursor()
    
    # Get max ID from database
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    max_id = result[0] if result[0] else 0
    
    cursor.close()
    connection.close()
    
    # Fetch only newer records
    # Implementation depends on API filtering capabilities
    print(f"Last loaded ID: {max_id}")
```

### Data Quality Checks

```python
def validate_data_quality():
    """Run data quality checks"""
    connection = get_db_connection()
    cursor = connection.cursor()
    
    checks = {
        "Total Artifacts": "SELECT COUNT(*) FROM artifactmetadata",
        "Null Titles": "SELECT COUNT(*) FROM artifactmetadata WHERE title IS NULL",
        "Missing Media": """
            SELECT COUNT(*) 
            FROM artifactmetadata m 
            LEFT JOIN artifactmedia med ON m.id = med.artifact_id 
            WHERE med.artifact_id IS NULL
        """,
        "Orphaned Colors": """
            SELECT COUNT(*) 
            FROM artifactcolors c 
            LEFT JOIN artifactmetadata m ON c.artifact_id = m.id 
            WHERE m.id IS NULL
        """
    }
    
    results = {}
    for check_name, query in checks.items():
        cursor.execute(query)
        results[check_name] = cursor.fetchone()[0]
    
    cursor.close()
    connection.close()
    
    return results
```

## Troubleshooting

### API Rate Limiting

```python
# Add exponential backoff for rate limiting
import time
from functools import wraps

def retry_with_backoff(max_retries=3):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except requests.exceptions.HTTPError as e:
                    if e.response.status_code == 429:
                        wait_time = 2 ** attempt
                        print(f"Rate limited. Waiting {wait_time}s...")
                        time.sleep(wait_time)
                    else:
                        raise
            raise Exception("Max retries exceeded")
        return wrapper
    return decorator

@retry_with_backoff(max_retries=5)
def fetch_with_retry(url, params):
    response = requests.get(url, params=params)
    response.raise_for_status()
    return response.json()
```

### Database Connection Issues

```python
# Test database connectivity
def test_database_connection():
    """Verify database connection and schema"""
    try:
        connection = get_db_connection()
        if not connection:
            return False
        
        cursor = connection.cursor()
        
        # Check if tables exist
        cursor.execute("SHOW TABLES")
        tables = [table[0] for table in cursor.fetchall()]
        
        required_tables = ['artifactmetadata', 'artifactmedia', 'artifactcolors']
        missing_tables = [t for t in required_tables if t not in tables]
        
        if missing_tables:
            print(f"Missing tables: {missing_tables}")
            return False
        
        print("Database connection and schema verified")
        return True
    except Exception as e:
        print(f"Database test failed: {e}")
        return False
    finally:
        if connection:
            cursor.close()
            connection.close()
```

### Memory Management for Large Datasets

```python
def chunked_load(artifacts, chunk_size=100):
    """Load data in chunks to avoid memory issues"""
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i + chunk_size]
        
        metadata_df = transform_metadata(chunk)
        media_df = transform_media(chunk)
        colors_df = transform_colors(chunk)
        
        load_metadata(metadata_df)
        load_media(media_df)
        load_colors(colors_df)
        
        print(f"Loaded chunk {i//chunk_size + 1}")
```

This skill provides comprehensive coverage of building ETL pipelines and analytics dashboards with the Harvard Art Museums API, including data extraction, transformation, loading, SQL analytics, and Streamlit visualization.
