---
name: harvard-art-museums-data-pipeline
description: Build end-to-end data engineering pipelines with Harvard Art Museums API, ETL, SQL analytics, and Streamlit visualization
triggers:
  - create a data pipeline for museum artifacts
  - build ETL pipeline with Harvard Art Museums API
  - set up artifact collection analytics dashboard
  - implement museum data engineering workflow
  - create streamlit dashboard for art museum data
  - build SQL analytics for Harvard artifacts
  - extract and analyze museum collection data
  - develop artifact data visualization pipeline
---

# Harvard Art Museums Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Harvard Artifacts Collection Data Engineering & Analytics App is an end-to-end data pipeline that demonstrates professional ETL practices using the Harvard Art Museums API. It extracts artifact metadata, transforms nested JSON into relational structures, loads data into SQL databases (MySQL/TiDB Cloud), and provides interactive analytics through Streamlit dashboards with Plotly visualizations.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

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
export DB_NAME="harvard_artifacts"
```

## Key Components

### 1. API Integration

The Harvard Art Museums API provides access to artifact data with pagination support:

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    
    return response.json()

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=100)
print(f"Total records: {data['info']['totalrecords']}")
print(f"Total pages: {data['info']['pages']}")
```

### 2. ETL Pipeline

Transform nested JSON into relational table structures:

```python
import pandas as pd

def extract_artifact_metadata(artifacts_json):
    """Extract and flatten artifact metadata"""
    records = []
    
    for artifact in artifacts_json['records']:
        record = {
            'object_id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'accession_year': artifact.get('accessionyear'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium')
        }
        records.append(record)
    
    return pd.DataFrame(records)

def extract_artifact_media(artifacts_json):
    """Extract media/image data"""
    media_records = []
    
    for artifact in artifacts_json['records']:
        object_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            media_record = {
                'object_id': object_id,
                'image_id': img.get('imageid'),
                'base_url': img.get('baseimageurl'),
                'width': img.get('width'),
                'height': img.get('height'),
                'format': img.get('format')
            }
            media_records.append(media_record)
    
    return pd.DataFrame(media_records)

def extract_artifact_colors(artifacts_json):
    """Extract color palette data"""
    color_records = []
    
    for artifact in artifacts_json['records']:
        object_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_record = {
                'object_id': object_id,
                'color_name': color.get('color'),
                'hex_code': color.get('hex'),
                'percentage': color.get('percent')
            }
            color_records.append(color_record)
    
    return pd.DataFrame(color_records)
```

### 3. Database Schema

Create relational tables with proper foreign keys:

```python
import mysql.connector
from mysql.connector import Error

def create_database_schema(connection):
    """Create database tables for artifact data"""
    
    cursor = connection.cursor()
    
    # Artifact Metadata Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            object_id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(255),
            period VARCHAR(255),
            century VARCHAR(100),
            classification VARCHAR(255),
            department VARCHAR(255),
            dated VARCHAR(255),
            accession_year INT,
            technique TEXT,
            medium TEXT,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    
    # Artifact Media Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            object_id INT,
            image_id INT,
            base_url VARCHAR(500),
            width INT,
            height INT,
            format VARCHAR(50),
            FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
        )
    """)
    
    # Artifact Colors Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            object_id INT,
            color_name VARCHAR(100),
            hex_code VARCHAR(10),
            percentage DECIMAL(5,2),
            FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
        )
    """)
    
    connection.commit()
    cursor.close()

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
```

### 4. Data Loading

Batch insert data for performance:

```python
def load_metadata_to_db(df, connection):
    """Load artifact metadata to database"""
    cursor = connection.cursor()
    
    # Prepare batch insert
    insert_query = """
        INSERT INTO artifactmetadata 
        (object_id, title, culture, period, century, classification, 
         department, dated, accession_year, technique, medium)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE 
        title=VALUES(title), culture=VALUES(culture)
    """
    
    # Convert DataFrame to list of tuples
    data = [tuple(x) for x in df.to_numpy()]
    
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()
    
    return len(data)

def load_media_to_db(df, connection):
    """Load artifact media to database"""
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia 
        (object_id, image_id, base_url, width, height, format)
        VALUES (%s, %s, %s, %s, %s, %s)
    """
    
    data = [tuple(x) for x in df.to_numpy()]
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()
    
    return len(data)
```

### 5. SQL Analytics Queries

Common analytical queries for insights:

```python
ANALYTICAL_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 20
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "artifacts_by_department": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "media_availability": """
        SELECT 
            am.classification,
            COUNT(DISTINCT am.object_id) as total_artifacts,
            COUNT(DISTINCT med.object_id) as artifacts_with_images,
            ROUND(COUNT(DISTINCT med.object_id) * 100.0 / COUNT(DISTINCT am.object_id), 2) as image_percentage
        FROM artifactmetadata am
        LEFT JOIN artifactmedia med ON am.object_id = med.object_id
        WHERE am.classification IS NOT NULL
        GROUP BY am.classification
        ORDER BY total_artifacts DESC
        LIMIT 15
    """,
    
    "color_distribution": """
        SELECT color_name, COUNT(*) as usage_count, AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color_name
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "artifacts_by_accession_year": """
        SELECT accession_year, COUNT(*) as artifacts_acquired
        FROM artifactmetadata
        WHERE accession_year IS NOT NULL
        GROUP BY accession_year
        ORDER BY accession_year DESC
        LIMIT 20
    """
}

def run_analytical_query(query_name, connection):
    """Execute analytical query and return DataFrame"""
    query = ANALYTICAL_QUERIES.get(query_name)
    
    if not query:
        raise ValueError(f"Query '{query_name}' not found")
    
    df = pd.read_sql(query, connection)
    return df
```

### 6. Streamlit Dashboard

Create interactive analytics dashboard:

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    st.markdown("### Data Engineering & Analytics Pipeline")
    
    # Sidebar for configuration
    with st.sidebar:
        st.header("Configuration")
        
        api_key = st.text_input("Harvard API Key", 
                                type="password", 
                                value=os.getenv('HARVARD_API_KEY', ''))
        
        st.header("ETL Controls")
        num_pages = st.number_input("Pages to fetch", min_value=1, max_value=100, value=5)
        
        if st.button("Run ETL Pipeline"):
            run_etl_pipeline(api_key, num_pages)
    
    # Main dashboard
    tab1, tab2, tab3 = st.tabs(["📊 Analytics", "🗄️ Data Explorer", "📈 Visualizations"])
    
    with tab1:
        st.header("SQL Analytics Queries")
        
        query_name = st.selectbox(
            "Select Analysis",
            list(ANALYTICAL_QUERIES.keys())
        )
        
        if st.button("Run Query"):
            conn = get_db_connection()
            df = run_analytical_query(query_name, conn)
            
            st.subheader("Query Results")
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df.columns) >= 2:
                fig = px.bar(df, 
                            x=df.columns[0], 
                            y=df.columns[1],
                            title=query_name.replace('_', ' ').title())
                st.plotly_chart(fig, use_container_width=True)
            
            conn.close()
    
    with tab2:
        st.header("Data Explorer")
        
        conn = get_db_connection()
        
        table = st.selectbox("Select Table", 
                            ["artifactmetadata", "artifactmedia", "artifactcolors"])
        
        query = f"SELECT * FROM {table} LIMIT 100"
        df = pd.read_sql(query, conn)
        
        st.dataframe(df)
        st.info(f"Showing first 100 rows from {table}")
        
        conn.close()

def run_etl_pipeline(api_key, num_pages):
    """Execute full ETL pipeline"""
    with st.spinner("Running ETL Pipeline..."):
        conn = get_db_connection()
        
        total_artifacts = 0
        
        for page in range(1, num_pages + 1):
            # Extract
            data = fetch_artifacts(api_key, page=page)
            
            # Transform
            metadata_df = extract_artifact_metadata(data)
            media_df = extract_artifact_media(data)
            colors_df = extract_artifact_colors(data)
            
            # Load
            load_metadata_to_db(metadata_df, conn)
            load_media_to_db(media_df, conn)
            
            total_artifacts += len(metadata_df)
            
            st.progress(page / num_pages)
        
        conn.close()
        st.success(f"✅ ETL Complete! Processed {total_artifacts} artifacts")

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Configuration

Create a `.env` file for environment variables:

```bash
HARVARD_API_KEY=your_api_key_here
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

## Common Patterns

### Complete ETL Script

```python
def complete_etl_workflow():
    """Full ETL workflow example"""
    
    # 1. Setup
    api_key = os.getenv('HARVARD_API_KEY')
    conn = get_db_connection()
    create_database_schema(conn)
    
    # 2. Extract
    print("Extracting data from API...")
    artifacts_data = fetch_artifacts(api_key, page=1, size=100)
    
    # 3. Transform
    print("Transforming data...")
    metadata_df = extract_artifact_metadata(artifacts_data)
    media_df = extract_artifact_media(artifacts_data)
    colors_df = extract_artifact_colors(artifacts_data)
    
    # 4. Load
    print("Loading to database...")
    load_metadata_to_db(metadata_df, conn)
    load_media_to_db(media_df, conn)
    
    # 5. Analytics
    print("Running analytics...")
    results = run_analytical_query('artifacts_by_culture', conn)
    print(results.head())
    
    conn.close()
    print("ETL complete!")
```

### Pagination Handler

```python
def fetch_all_artifacts(api_key, max_pages=10):
    """Fetch multiple pages of artifacts"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        try:
            data = fetch_artifacts(api_key, page=page)
            all_artifacts.extend(data['records'])
            
            print(f"Fetched page {page}/{max_pages}")
            
            # Respect rate limits
            time.sleep(0.5)
            
        except requests.exceptions.HTTPError as e:
            print(f"Error on page {page}: {e}")
            break
    
    return all_artifacts
```

## Troubleshooting

### API Key Issues

```python
# Verify API key is working
def test_api_connection(api_key):
    """Test Harvard API connection"""
    try:
        response = requests.get(
            "https://api.harvardartmuseums.org/object",
            params={'apikey': api_key, 'size': 1}
        )
        response.raise_for_status()
        print("✅ API connection successful")
        return True
    except requests.exceptions.HTTPError as e:
        print(f"❌ API error: {e}")
        return False
```

### Database Connection Issues

```python
def test_db_connection():
    """Test database connectivity"""
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        cursor.close()
        conn.close()
        print("✅ Database connection successful")
        return True
    except Error as e:
        print(f"❌ Database error: {e}")
        return False
```

### Handle Missing Data

```python
def safe_extract_artifact_metadata(artifacts_json):
    """Extract metadata with null handling"""
    records = []
    
    for artifact in artifacts_json.get('records', []):
        record = {
            'object_id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', None),
            'period': artifact.get('period', None),
            'century': artifact.get('century', None),
            'classification': artifact.get('classification', 'Unclassified'),
            'department': artifact.get('department', None),
            'dated': artifact.get('dated', None),
            'accession_year': artifact.get('accessionyear', None),
            'technique': artifact.get('technique', None),
            'medium': artifact.get('medium', None)
        }
        records.append(record)
    
    return pd.DataFrame(records)
```

## Best Practices

1. **Always use environment variables** for sensitive credentials
2. **Implement rate limiting** when fetching from the API
3. **Use batch inserts** for better database performance
4. **Handle pagination** properly for large datasets
5. **Validate data** before loading to database
6. **Create indexes** on frequently queried columns
7. **Log ETL progress** for debugging and monitoring
