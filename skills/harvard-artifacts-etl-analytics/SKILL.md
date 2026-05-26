---
name: harvard-artifacts-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using Harvard Art Museums API with Streamlit, Pandas, and SQL
triggers:
  - how do I build an ETL pipeline with the Harvard Art Museums API
  - create a data engineering app with Harvard artifacts collection
  - set up analytics dashboard for museum artifact data
  - extract and transform Harvard museum data into SQL
  - build a Streamlit app for art collection analytics
  - query and visualize Harvard Art Museums API data
  - implement ETL workflow for museum artifacts
  - analyze Harvard art collection with Python and SQL
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualization using Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Collect artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color data into SQL databases
- **SQL Analytics**: Execute predefined analytical queries on structured artifact data
- **Interactive Dashboards**: Visualize query results using Streamlit and Plotly
- **Database Design**: Relational schema with proper foreign key relationships

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required Dependencies:**
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### 1. Harvard Art Museums API Key

Get your API key from: https://docs.google.com/forms/d/e/1FAIpQLSfkmEBqH76HLMMiCC-GPPnhcvHC9aJS86E32dkEM0ZKWZ82tTg/viewform

Store it in environment variables:

```bash
# .env file
HARVARD_API_KEY=your_api_key_here
```

### 2. Database Configuration

Set up MySQL or TiDB Cloud connection:

```python
# Database credentials in .env
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
DB_PORT=3306
```

### 3. Database Schema

Create the required tables:

```sql
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    dated VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    medium VARCHAR(500),
    technique VARCHAR(500),
    dimensionsummary VARCHAR(500)
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    iiifbaseuri VARCHAR(500),
    baseimageurl VARCHAR(500),
    imagetype VARCHAR(100),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    colorpercent DECIMAL(5,2),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## Key Components

### ETL Pipeline Implementation

**Extract: Fetch data from Harvard API**

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only fetch artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Fetch multiple pages with pagination
def collect_artifacts(total_records=1000):
    """Collect artifacts with pagination handling"""
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < total_records:
        data = fetch_artifacts(page=page, size=size)
        artifacts = data.get('records', [])
        
        if not artifacts:
            break
            
        all_artifacts.extend(artifacts)
        page += 1
        
    return all_artifacts[:total_records]
```

**Transform: Process nested JSON into relational format**

```python
import pandas as pd

def transform_metadata(artifacts):
    """Transform artifact data into metadata table format"""
    metadata_records = []
    
    for artifact in artifacts:
        record = {
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'century': artifact.get('century', '')[:100],
            'dated': artifact.get('dated', '')[:255],
            'classification': artifact.get('classification', '')[:255],
            'department': artifact.get('department', '')[:255],
            'division': artifact.get('division', '')[:255],
            'medium': artifact.get('medium', '')[:500],
            'technique': artifact.get('technique', '')[:500],
            'dimensionsummary': artifact.get('dimensions', '')[:500]
        }
        metadata_records.append(record)
    
    return pd.DataFrame(metadata_records)

def transform_media(artifacts):
    """Transform media/image data"""
    media_records = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        images = artifact.get('images', [])
        
        for img in images:
            record = {
                'objectid': objectid,
                'iiifbaseuri': img.get('iiifbaseuri', '')[:500],
                'baseimageurl': img.get('baseimageurl', '')[:500],
                'imagetype': img.get('imagetype', '')[:100]
            }
            media_records.append(record)
    
    return pd.DataFrame(media_records)

def transform_colors(artifacts):
    """Transform color data"""
    color_records = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        colors = artifact.get('colors', [])
        
        for color_data in colors:
            record = {
                'objectid': objectid,
                'color': color_data.get('color', '')[:50],
                'spectrum': color_data.get('spectrum', '')[:50],
                'colorpercent': color_data.get('percent', 0)
            }
            color_records.append(record)
    
    return pd.DataFrame(color_records)
```

**Load: Insert data into SQL database**

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME'),
            port=int(os.getenv('DB_PORT', 3306))
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_metadata(df_metadata):
    """Load metadata into database with batch insert"""
    connection = get_db_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (objectid, title, culture, century, dated, classification, 
     department, division, medium, technique, dimensionsummary)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    records = df_metadata.to_records(index=False)
    data = [tuple(record) for record in records]
    
    try:
        cursor.executemany(insert_query, data)
        connection.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
        return True
    except Error as e:
        print(f"Error inserting metadata: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()

def load_media(df_media):
    """Load media data into database"""
    connection = get_db_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (objectid, iiifbaseuri, baseimageurl, imagetype)
    VALUES (%s, %s, %s, %s)
    """
    
    records = df_media.to_records(index=False)
    data = [tuple(record) for record in records]
    
    try:
        cursor.executemany(insert_query, data)
        connection.commit()
        print(f"Inserted {cursor.rowcount} media records")
        return True
    except Error as e:
        print(f"Error inserting media: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()
```

### Analytical SQL Queries

```python
# Example analytical queries
ANALYTICAL_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "Media Availability": """
        SELECT 
            CASE WHEN m.objectid IS NOT NULL THEN 'With Images' 
                 ELSE 'Without Images' END as media_status,
            COUNT(*) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.objectid = m.objectid
        GROUP BY media_status
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as frequency,
               AVG(colorpercent) as avg_percentage
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 10
    """
}

def execute_query(query):
    """Execute analytical query and return results"""
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

### Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def create_dashboard():
    """Create Streamlit analytics dashboard"""
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Select Analysis")
    query_name = st.sidebar.selectbox(
        "Choose Query",
        list(ANALYTICAL_QUERIES.keys())
    )
    
    # Execute selected query
    if st.sidebar.button("Run Analysis"):
        with st.spinner("Executing query..."):
            query = ANALYTICAL_QUERIES[query_name]
            df_result = execute_query(query)
            
            if df_result is not None and not df_result.empty:
                st.subheader(f"Results: {query_name}")
                
                # Display data table
                st.dataframe(df_result)
                
                # Auto-generate visualization
                if len(df_result.columns) == 2:
                    fig = px.bar(
                        df_result,
                        x=df_result.columns[0],
                        y=df_result.columns[1],
                        title=query_name
                    )
                    st.plotly_chart(fig)
            else:
                st.error("No results found")

if __name__ == "__main__":
    create_dashboard()
```

## Common Patterns

### Complete ETL Pipeline Execution

```python
def run_etl_pipeline(num_artifacts=500):
    """Execute complete ETL workflow"""
    print("Starting ETL Pipeline...")
    
    # Extract
    print("1. Extracting data from API...")
    artifacts = collect_artifacts(total_records=num_artifacts)
    
    # Transform
    print("2. Transforming data...")
    df_metadata = transform_metadata(artifacts)
    df_media = transform_media(artifacts)
    df_colors = transform_colors(artifacts)
    
    # Load
    print("3. Loading data into database...")
    load_metadata(df_metadata)
    load_media(df_media)
    load_colors(df_colors)
    
    print("ETL Pipeline completed successfully!")
```

### Running the Streamlit App

```bash
# Start the dashboard
streamlit run app.py

# Run on custom port
streamlit run app.py --server.port 8501
```

## Troubleshooting

**API Rate Limiting:**
```python
import time

def fetch_with_retry(page, max_retries=3):
    """Fetch with retry logic for rate limiting"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page=page)
        except Exception as e:
            if "429" in str(e):  # Too Many Requests
                wait_time = (attempt + 1) * 5
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

**Database Connection Issues:**
```python
# Test database connection
def test_connection():
    """Verify database connectivity"""
    try:
        conn = get_db_connection()
        if conn and conn.is_connected():
            print("✓ Database connection successful")
            conn.close()
            return True
        else:
            print("✗ Database connection failed")
            return False
    except Error as e:
        print(f"✗ Connection error: {e}")
        return False
```

**Handling NULL Values:**
```python
# Clean data before insert
df_metadata = df_metadata.fillna({
    'title': 'Unknown',
    'culture': 'Unknown',
    'century': 'Unknown'
})
```

This skill provides everything needed to build production-ready ETL pipelines and analytics dashboards for museum artifact data using the Harvard Art Museums API.
