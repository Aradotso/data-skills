---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for museum data
  - analyze Harvard Art Museums collection
  - create a data engineering app with Streamlit
  - set up SQL analytics for artifact data
  - extract and load museum API data
  - visualize art collection analytics
  - design a museum data warehouse
  - implement batch data ingestion from APIs
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Harvard Artifacts Collection Data Engineering Analytics App is a production-ready ETL pipeline that extracts artifact data from the Harvard Art Museums API, transforms it into relational schemas, loads it into SQL databases (MySQL/TiDB), and provides interactive analytics through Streamlit dashboards with Plotly visualizations.

**Architecture**: API → ETL → SQL → Analytics → Visualization

**Key Components**:
- API integration with pagination and rate limiting
- Multi-table relational database design
- Batch insert optimization
- 20+ predefined analytical queries
- Interactive visualization dashboards

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

### Environment Setup

Create a `.env` file or set environment variables:

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

### Database Schema

The application uses three core tables:

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    department VARCHAR(200),
    classification VARCHAR(200),
    dated VARCHAR(200),
    url TEXT,
    description TEXT,
    accession_number VARCHAR(100),
    people TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    base_image_url TEXT,
    primary_image_url TEXT,
    image_count INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(7),
    color_name VARCHAR(100),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Core Functionality

### 1. API Data Extraction

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    
    data = response.json()
    return data['records'], data['info']

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts, info = fetch_artifacts(api_key, page=1)

print(f"Total records: {info['totalrecords']}")
print(f"Fetched: {len(artifacts)} artifacts")
```

### 2. Data Transformation

```python
import pandas as pd

def transform_artifact_metadata(raw_artifacts):
    """
    Transform raw API data into structured metadata
    """
    metadata = []
    
    for artifact in raw_artifacts:
        record = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'department': artifact.get('department'),
            'classification': artifact.get('classification'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url'),
            'description': artifact.get('description'),
            'accession_number': artifact.get('accessionmethod'),
            'people': ', '.join([p.get('name', '') for p in artifact.get('people', [])])
        }
        metadata.append(record)
    
    return pd.DataFrame(metadata)

def transform_artifact_colors(raw_artifacts):
    """
    Extract color data from artifacts
    """
    colors = []
    
    for artifact in raw_artifacts:
        artifact_id = artifact.get('id')
        color_data = artifact.get('colors', [])
        
        for color in color_data:
            colors.append({
                'artifact_id': artifact_id,
                'color_hex': color.get('hex'),
                'color_name': color.get('color'),
                'percentage': color.get('percent')
            })
    
    return pd.DataFrame(colors)

def transform_artifact_media(raw_artifacts):
    """
    Extract media and image data
    """
    media = []
    
    for artifact in raw_artifacts:
        media.append({
            'artifact_id': artifact.get('id'),
            'base_image_url': artifact.get('baseimageurl'),
            'primary_image_url': artifact.get('primaryimageurl'),
            'image_count': len(artifact.get('images', []))
        })
    
    return pd.DataFrame(media)
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

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

def batch_insert_metadata(connection, df_metadata):
    """
    Batch insert artifact metadata with optimization
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, department, classification, dated, url, description, accession_number, people)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    data = [tuple(row) for row in df_metadata.values]
    
    cursor.executemany(insert_query, data)
    connection.commit()
    
    print(f"Inserted {cursor.rowcount} metadata records")
    cursor.close()

def batch_insert_colors(connection, df_colors):
    """
    Batch insert color data
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors (artifact_id, color_hex, color_name, percentage)
    VALUES (%s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df_colors.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    
    print(f"Inserted {cursor.rowcount} color records")
    cursor.close()
```

### 4. Complete ETL Pipeline

```python
def run_etl_pipeline(api_key, num_pages=5):
    """
    Execute complete ETL pipeline
    """
    # Connect to database
    conn = create_database_connection()
    if not conn:
        return
    
    all_artifacts = []
    
    # Extract: Fetch data from API
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        artifacts, info = fetch_artifacts(api_key, page=page)
        all_artifacts.extend(artifacts)
    
    print(f"Total artifacts extracted: {len(all_artifacts)}")
    
    # Transform: Convert to structured data
    df_metadata = transform_artifact_metadata(all_artifacts)
    df_colors = transform_artifact_colors(all_artifacts)
    df_media = transform_artifact_media(all_artifacts)
    
    # Load: Insert into database
    batch_insert_metadata(conn, df_metadata)
    batch_insert_colors(conn, df_colors)
    batch_insert_media(conn, df_media)
    
    conn.close()
    print("ETL pipeline completed successfully!")

# Execute pipeline
api_key = os.getenv('HARVARD_API_KEY')
run_etl_pipeline(api_key, num_pages=10)
```

### 5. Analytics Queries

```python
def execute_analytics_query(connection, query_name):
    """
    Execute predefined analytics queries
    """
    queries = {
        'artifacts_by_culture': """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture IS NOT NULL
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 15
        """,
        
        'artifacts_by_century': """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL
            GROUP BY century
            ORDER BY count DESC
        """,
        
        'top_colors': """
            SELECT color_name, COUNT(*) as frequency, AVG(percentage) as avg_percentage
            FROM artifactcolors
            WHERE color_name IS NOT NULL
            GROUP BY color_name
            ORDER BY frequency DESC
            LIMIT 10
        """,
        
        'media_availability': """
            SELECT 
                CASE 
                    WHEN image_count > 0 THEN 'Has Images'
                    ELSE 'No Images'
                END as image_status,
                COUNT(*) as count
            FROM artifactmedia
            GROUP BY image_status
        """,
        
        'artifacts_by_department': """
            SELECT department, COUNT(*) as count
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY count DESC
        """
    }
    
    cursor = connection.cursor(dictionary=True)
    cursor.execute(queries[query_name])
    results = cursor.fetchall()
    cursor.close()
    
    return pd.DataFrame(results)
```

### 6. Streamlit Dashboard Integration

```python
import streamlit as st
import plotly.express as px

def create_dashboard():
    """
    Create interactive Streamlit dashboard
    """
    st.title("🏛️ Harvard Art Museums Analytics")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    # Database connection
    conn = create_database_connection()
    
    # Query selection
    query_options = [
        'artifacts_by_culture',
        'artifacts_by_century',
        'top_colors',
        'media_availability',
        'artifacts_by_department'
    ]
    
    selected_query = st.sidebar.selectbox("Select Analysis", query_options)
    
    # Execute query
    if st.sidebar.button("Run Analysis"):
        with st.spinner("Executing query..."):
            df_results = execute_analytics_query(conn, selected_query)
            
            # Display results
            st.subheader("Query Results")
            st.dataframe(df_results)
            
            # Visualization
            if len(df_results) > 0:
                st.subheader("Visualization")
                
                x_col = df_results.columns[0]
                y_col = df_results.columns[1]
                
                fig = px.bar(
                    df_results,
                    x=x_col,
                    y=y_col,
                    title=f"{selected_query.replace('_', ' ').title()}",
                    color=y_col,
                    color_continuous_scale='Viridis'
                )
                
                st.plotly_chart(fig, use_container_width=True)
    
    conn.close()

# Run dashboard
if __name__ == "__main__":
    create_dashboard()
```

## Common Patterns

### Rate Limiting API Calls

```python
import time

def fetch_with_rate_limit(api_key, total_pages, delay=1):
    """
    Fetch data with rate limiting to avoid API throttling
    """
    all_data = []
    
    for page in range(1, total_pages + 1):
        artifacts, _ = fetch_artifacts(api_key, page=page)
        all_data.extend(artifacts)
        
        time.sleep(delay)  # Rate limit
        
        if page % 10 == 0:
            print(f"Progress: {page}/{total_pages} pages")
    
    return all_data
```

### Error Handling in ETL

```python
def safe_etl_pipeline(api_key, num_pages):
    """
    ETL pipeline with comprehensive error handling
    """
    try:
        conn = create_database_connection()
        if not conn:
            raise Exception("Database connection failed")
        
        for page in range(1, num_pages + 1):
            try:
                artifacts, _ = fetch_artifacts(api_key, page=page)
                
                df_metadata = transform_artifact_metadata(artifacts)
                batch_insert_metadata(conn, df_metadata)
                
                print(f"Page {page} processed successfully")
                
            except requests.exceptions.RequestException as e:
                print(f"API error on page {page}: {e}")
                continue
            
            except Exception as e:
                print(f"Processing error on page {page}: {e}")
                continue
        
        conn.close()
        
    except Exception as e:
        print(f"Fatal error: {e}")
        raise
```

## Troubleshooting

### API Key Issues

```python
# Verify API key is valid
def test_api_connection(api_key):
    try:
        response = requests.get(
            "https://api.harvardartmuseums.org/object",
            params={'apikey': api_key, 'size': 1}
        )
        response.raise_for_status()
        print("✓ API connection successful")
        return True
    except Exception as e:
        print(f"✗ API connection failed: {e}")
        return False
```

### Database Connection Issues

```python
# Test database connectivity
def test_database_connection():
    try:
        conn = create_database_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        cursor.close()
        conn.close()
        print("✓ Database connection successful")
        return True
    except Exception as e:
        print(f"✗ Database connection failed: {e}")
        return False
```

### Memory Optimization for Large Datasets

```python
def chunked_etl_pipeline(api_key, total_pages, chunk_size=10):
    """
    Process data in chunks to manage memory
    """
    conn = create_database_connection()
    
    for chunk_start in range(1, total_pages + 1, chunk_size):
        chunk_end = min(chunk_start + chunk_size, total_pages + 1)
        
        chunk_artifacts = []
        for page in range(chunk_start, chunk_end):
            artifacts, _ = fetch_artifacts(api_key, page=page)
            chunk_artifacts.extend(artifacts)
        
        # Process chunk
        df_metadata = transform_artifact_metadata(chunk_artifacts)
        batch_insert_metadata(conn, df_metadata)
        
        # Clear memory
        del chunk_artifacts
        del df_metadata
        
        print(f"Processed chunk {chunk_start}-{chunk_end-1}")
    
    conn.close()
```

## Best Practices

1. **Always use environment variables** for API keys and database credentials
2. **Implement rate limiting** when fetching from APIs
3. **Use batch inserts** for database operations (100-1000 records per batch)
4. **Handle NULL values** during transformation
5. **Create indexes** on frequently queried columns (culture, century, department)
6. **Log ETL operations** for debugging and monitoring
7. **Use connection pooling** for high-throughput applications
8. **Validate data** before inserting into database
