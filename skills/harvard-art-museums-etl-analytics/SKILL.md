---
name: harvard-art-museums-etl-analytics
description: Build end-to-end data engineering pipelines with Harvard Art Museums API, ETL workflows, SQL analytics, and Streamlit dashboards
triggers:
  - build an ETL pipeline for museum artifact data
  - create a data engineering workflow with Harvard Art Museums API
  - set up SQL analytics for art collection data
  - build a Streamlit dashboard for museum artifacts
  - extract and transform Harvard Art Museums data
  - create artifact analytics with SQL queries
  - build a museum data pipeline with Python
  - analyze art collection data with ETL
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load museum data into relational SQL databases
- **Database Design**: Structured tables for artifact metadata, media, and color information
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Interactive Visualization**: Streamlit dashboards with Plotly charts

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

### Environment Variables

Create a `.env` file in the project root:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration (MySQL/TiDB)
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Request an API key (free for non-commercial use)
3. Add to your `.env` file

## Key Components

### 1. API Data Collection

Fetch artifact data with pagination:

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    params = {
        'apikey': API_KEY,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(BASE_URL, params=params)
    response.raise_for_status()
    return response.json()

# Fetch multiple pages
all_artifacts = []
for page in range(1, 6):  # First 5 pages
    data = fetch_artifacts(page=page)
    all_artifacts.extend(data['records'])
    print(f"Fetched page {page}: {len(data['records'])} artifacts")
```

### 2. ETL Pipeline

Transform and load data into SQL:

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

def create_database_connection():
    """Create MySQL database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=os.getenv('DB_PORT'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Error connecting to database: {e}")
        return None

def transform_artifact_metadata(artifacts):
    """Transform artifact data to metadata table format"""
    metadata_records = []
    
    for artifact in artifacts:
        record = {
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'period': artifact.get('period', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'medium': artifact.get('medium', 'Unknown'),
            'dimensions': artifact.get('dimensions', 'Unknown'),
            'creditline': artifact.get('creditline', 'Unknown'),
            'division': artifact.get('division', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'dated': artifact.get('dated', 'Unknown')
        }
        metadata_records.append(record)
    
    return pd.DataFrame(metadata_records)

def transform_artifact_media(artifacts):
    """Transform media/image data"""
    media_records = []
    
    for artifact in artifacts:
        object_id = artifact.get('objectid')
        images = artifact.get('images', [])
        
        for img in images:
            record = {
                'objectid': object_id,
                'imageid': img.get('imageid'),
                'baseimageurl': img.get('baseimageurl', ''),
                'width': img.get('width', 0),
                'height': img.get('height', 0),
                'format': img.get('format', 'Unknown')
            }
            media_records.append(record)
    
    return pd.DataFrame(media_records)

def transform_artifact_colors(artifacts):
    """Transform color data"""
    color_records = []
    
    for artifact in artifacts:
        object_id = artifact.get('objectid')
        colors = artifact.get('colors', [])
        
        for color in colors:
            record = {
                'objectid': object_id,
                'color': color.get('color', 'Unknown'),
                'spectrum': color.get('spectrum', 'Unknown'),
                'hue': color.get('hue', 'Unknown'),
                'percent': color.get('percent', 0.0)
            }
            color_records.append(record)
    
    return pd.DataFrame(color_records)

def load_to_database(df, table_name, connection):
    """Load DataFrame to SQL table"""
    cursor = connection.cursor()
    
    # Create insert query
    columns = ', '.join(df.columns)
    placeholders = ', '.join(['%s'] * len(df.columns))
    query = f"INSERT INTO {table_name} ({columns}) VALUES ({placeholders})"
    
    # Batch insert
    data = [tuple(row) for row in df.values]
    cursor.executemany(query, data)
    connection.commit()
    
    print(f"Loaded {len(df)} records into {table_name}")
    cursor.close()
```

### 3. Database Schema

Create the SQL tables:

```python
def create_tables(connection):
    """Create database schema"""
    cursor = connection.cursor()
    
    # Artifact Metadata Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            objectid INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            period VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            medium VARCHAR(500),
            dimensions VARCHAR(500),
            creditline TEXT,
            division VARCHAR(200),
            department VARCHAR(200),
            dated VARCHAR(200)
        )
    """)
    
    # Artifact Media Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            imageid INT,
            baseimageurl VARCHAR(500),
            width INT,
            height INT,
            format VARCHAR(50),
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    # Artifact Colors Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            color VARCHAR(100),
            spectrum VARCHAR(100),
            hue VARCHAR(100),
            percent FLOAT,
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    connection.commit()
    cursor.close()
    print("Database schema created successfully")
```

### 4. SQL Analytics Queries

Example analytical queries:

```python
ANALYTICAL_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != 'Unknown'
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Top Departments": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        GROUP BY department
        ORDER BY total_artifacts DESC
        LIMIT 10
    """,
    
    "Media Availability": """
        SELECT 
            CASE WHEN imageid IS NOT NULL THEN 'Has Image' ELSE 'No Image' END as media_status,
            COUNT(*) as count
        FROM artifactmetadata m
        LEFT JOIN artifactmedia am ON m.objectid = am.objectid
        GROUP BY media_status
    """,
    
    "Color Distribution": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 10
    """,
    
    "Artifacts by Classification and Period": """
        SELECT classification, period, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification != 'Unknown' AND period != 'Unknown'
        GROUP BY classification, period
        ORDER BY count DESC
        LIMIT 20
    """
}

def execute_query(connection, query_name):
    """Execute analytical query and return results"""
    cursor = connection.cursor(dictionary=True)
    query = ANALYTICAL_QUERIES[query_name]
    
    cursor.execute(query)
    results = cursor.fetchall()
    cursor.close()
    
    return pd.DataFrame(results)
```

### 5. Streamlit Dashboard

Build the interactive app:

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Art Museums Analytics", layout="wide")
    st.title("🎨 Harvard Art Museums - Data Analytics Dashboard")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "ETL Pipeline", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "ETL Pipeline":
        show_etl_pipeline()
    elif page == "SQL Analytics":
        show_sql_analytics()
    elif page == "Visualizations":
        show_visualizations()

def show_sql_analytics():
    """SQL Analytics page"""
    st.header("📊 SQL Analytics Dashboard")
    
    connection = create_database_connection()
    if not connection:
        st.error("Database connection failed")
        return
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICAL_QUERIES.keys()))
    
    if st.button("Run Query"):
        with st.spinner("Executing query..."):
            df = execute_query(connection, query_name)
            
            st.subheader("Query Results")
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df) > 0:
                st.subheader("Visualization")
                
                # Determine chart type based on columns
                numeric_cols = df.select_dtypes(include=['int64', 'float64']).columns
                categorical_cols = df.select_dtypes(include=['object']).columns
                
                if len(categorical_cols) > 0 and len(numeric_cols) > 0:
                    fig = px.bar(
                        df, 
                        x=categorical_cols[0], 
                        y=numeric_cols[0],
                        title=query_name
                    )
                    st.plotly_chart(fig, use_container_width=True)
    
    connection.close()

def show_data_collection():
    """Data collection page"""
    st.header("🔌 API Data Collection")
    
    num_pages = st.number_input("Number of pages to fetch", 1, 10, 5)
    
    if st.button("Fetch Artifacts"):
        with st.spinner(f"Fetching {num_pages} pages..."):
            artifacts = []
            for page in range(1, num_pages + 1):
                data = fetch_artifacts(page=page)
                artifacts.extend(data['records'])
                st.write(f"Page {page}: {len(data['records'])} artifacts")
            
            st.success(f"Total artifacts fetched: {len(artifacts)}")
            
            # Preview data
            if artifacts:
                preview_df = pd.DataFrame([
                    {
                        'ID': a.get('objectid'),
                        'Title': a.get('title', 'Unknown')[:50],
                        'Culture': a.get('culture', 'Unknown'),
                        'Century': a.get('century', 'Unknown')
                    }
                    for a in artifacts[:10]
                ])
                st.dataframe(preview_df)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Complete ETL Workflow

```python
def run_complete_etl(num_pages=5):
    """Run complete ETL pipeline"""
    # Extract
    print("Step 1: Extracting data from API...")
    artifacts = []
    for page in range(1, num_pages + 1):
        data = fetch_artifacts(page=page)
        artifacts.extend(data['records'])
    
    # Transform
    print("Step 2: Transforming data...")
    metadata_df = transform_artifact_metadata(artifacts)
    media_df = transform_artifact_media(artifacts)
    colors_df = transform_artifact_colors(artifacts)
    
    # Load
    print("Step 3: Loading to database...")
    connection = create_database_connection()
    create_tables(connection)
    
    load_to_database(metadata_df, 'artifactmetadata', connection)
    load_to_database(media_df, 'artifactmedia', connection)
    load_to_database(colors_df, 'artifactcolors', connection)
    
    connection.close()
    print("ETL pipeline completed successfully!")

# Execute
run_complete_etl(num_pages=10)
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(page, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page=page)
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:  # Rate limit
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
def test_database_connection():
    """Test database connectivity"""
    try:
        connection = create_database_connection()
        if connection.is_connected():
            print("✓ Database connection successful")
            cursor = connection.cursor()
            cursor.execute("SELECT DATABASE()")
            db_name = cursor.fetchone()[0]
            print(f"✓ Connected to database: {db_name}")
            cursor.close()
            connection.close()
            return True
    except Error as e:
        print(f"✗ Database connection failed: {e}")
        return False
```

### Handle Missing Data

```python
def safe_transform(artifacts):
    """Handle missing or malformed data"""
    clean_records = []
    
    for artifact in artifacts:
        try:
            record = {
                'objectid': artifact.get('objectid'),
                'title': artifact.get('title') or 'Untitled',
                'culture': artifact.get('culture') or 'Unknown',
                # Use .get() with defaults for all fields
            }
            clean_records.append(record)
        except Exception as e:
            print(f"Error processing artifact {artifact.get('objectid')}: {e}")
            continue
    
    return pd.DataFrame(clean_records)
```

This skill provides AI coding agents with comprehensive knowledge to build, modify, and troubleshoot Harvard Art Museums ETL analytics applications.
