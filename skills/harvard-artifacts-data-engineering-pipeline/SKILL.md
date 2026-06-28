---
name: harvard-artifacts-data-engineering-pipeline
description: Build end-to-end data engineering pipelines with the Harvard Art Museums API, ETL workflows, SQL analytics, and Streamlit visualization
triggers:
  - how do I build a data pipeline with Harvard Art Museums API
  - show me ETL pipeline for museum artifacts data
  - create analytics dashboard for Harvard artifacts collection
  - how to use Harvard Art Museums API for data engineering
  - build Streamlit app with museum data and SQL analytics
  - extract and analyze Harvard museum artifacts data
  - set up ETL workflow for art collection data
  - visualize museum artifacts with SQL queries
---

# Harvard Artifacts Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates a complete data engineering workflow: extracting artifact data from the Harvard Art Museums API, transforming it into relational structures, loading into SQL databases, running analytics queries, and visualizing results through an interactive Streamlit dashboard.

## What This Project Does

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads nested JSON into normalized SQL tables
- **SQL Analytics**: Executes 20+ predefined analytical queries on artifact metadata, media, and color data
- **Interactive Visualization**: Displays query results and auto-generates Plotly charts in Streamlit

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages:**
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### 1. Harvard Art Museums API Key

Get your free API key from: https://www.harvardartmuseums.org/collections/api

Set it as an environment variable:
```bash
export HARVARD_API_KEY="your_api_key_here"
```

Or use a `.env` file:
```
HARVARD_API_KEY=your_api_key_here
```

### 2. Database Configuration

The project supports MySQL or TiDB Cloud. Configure connection parameters:

```python
import os
from dotenv import load_dotenv

load_dotenv()

db_config = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts')
}
```

Environment variables needed:
```
DB_HOST=your_db_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

## Key Components

### ETL Pipeline Architecture

The pipeline follows this flow:
1. **Extract**: Fetch artifact data from API with pagination
2. **Transform**: Parse JSON, normalize nested structures, create relational records
3. **Load**: Batch insert into three main tables with foreign key relationships

### Database Schema

```sql
-- Main artifact metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dated VARCHAR(200),
    url TEXT,
    accession_number VARCHAR(100)
);

-- Media/images associated with artifacts
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url TEXT,
    image_format VARCHAR(50),
    image_width INT,
    image_height INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Color data extracted from artifact images
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percent DECIMAL(5,2),
    color_spectrum VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Usage Patterns

### 1. Fetching Data from Harvard API

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
data = fetch_artifacts(api_key, page=1, size=50)
artifacts = data['records']
total_pages = data['info']['pages']
```

### 2. Transform Nested JSON to Relational Records

```python
import pandas as pd

def transform_artifact_data(artifacts):
    """Transform API response into normalized dataframes"""
    
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url'),
            'accession_number': artifact.get('accessionmethod')
        })
        
        # Extract media/images
        if artifact.get('images'):
            for img in artifact['images']:
                media_records.append({
                    'artifact_id': artifact.get('id'),
                    'image_url': img.get('baseimageurl'),
                    'image_format': img.get('format'),
                    'image_width': img.get('width'),
                    'image_height': img.get('height')
                })
        
        # Extract color data
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_records.append({
                    'artifact_id': artifact.get('id'),
                    'color_hex': color.get('hex'),
                    'color_percent': color.get('percent'),
                    'color_spectrum': color.get('spectrum')
                })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### 3. Load Data into SQL Database

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(metadata_df, media_df, colors_df, db_config):
    """Batch insert dataframes into MySQL database"""
    
    try:
        connection = mysql.connector.connect(**db_config)
        cursor = connection.cursor()
        
        # Insert metadata
        insert_metadata = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, 
         department, technique, medium, dated, url, accession_number)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(insert_metadata, metadata_df.values.tolist())
        
        # Insert media
        insert_media = """
        INSERT INTO artifactmedia 
        (artifact_id, image_url, image_format, image_width, image_height)
        VALUES (%s, %s, %s, %s, %s)
        """
        cursor.executemany(insert_media, media_df.values.tolist())
        
        # Insert colors
        insert_colors = """
        INSERT INTO artifactcolors 
        (artifact_id, color_hex, color_percent, color_spectrum)
        VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(insert_colors, colors_df.values.tolist())
        
        connection.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

### 4. Run Analytics Queries

```python
def execute_analytics_query(query, db_config):
    """Execute SQL query and return results as DataFrame"""
    
    connection = mysql.connector.connect(**db_config)
    df = pd.read_sql(query, connection)
    connection.close()
    return df

# Example analytical queries
queries = {
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE century IS NOT NULL 
        GROUP BY century 
        ORDER BY count DESC 
        LIMIT 10
    """,
    
    "Top Cultures": """
        SELECT culture, COUNT(*) as artifact_count 
        FROM artifactmetadata 
        WHERE culture IS NOT NULL 
        GROUP BY culture 
        ORDER BY artifact_count DESC 
        LIMIT 15
    """,
    
    "Color Distribution": """
        SELECT color_spectrum, COUNT(*) as color_count,
               AVG(color_percent) as avg_percent
        FROM artifactcolors 
        GROUP BY color_spectrum 
        ORDER BY color_count DESC
    """,
    
    "Artifacts with Most Images": """
        SELECT m.title, m.culture, COUNT(img.media_id) as image_count
        FROM artifactmetadata m
        JOIN artifactmedia img ON m.id = img.artifact_id
        GROUP BY m.id, m.title, m.culture
        ORDER BY image_count DESC
        LIMIT 20
    """
}

# Execute query
result_df = execute_analytics_query(queries["Top Cultures"], db_config)
```

### 5. Streamlit Dashboard Integration

```python
import streamlit as st
import plotly.express as px

def create_dashboard():
    st.title("Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for query selection
    query_name = st.sidebar.selectbox(
        "Select Analysis",
        list(queries.keys())
    )
    
    # Execute selected query
    if st.button("Run Analysis"):
        with st.spinner("Executing query..."):
            result_df = execute_analytics_query(queries[query_name], db_config)
            
            # Display results
            st.subheader(f"Results: {query_name}")
            st.dataframe(result_df)
            
            # Auto-generate visualization
            if len(result_df.columns) >= 2:
                fig = px.bar(
                    result_df, 
                    x=result_df.columns[0], 
                    y=result_df.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig)

# Run the app
if __name__ == "__main__":
    create_dashboard()
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Workflows

### Complete ETL Pipeline Execution

```python
def run_etl_pipeline(api_key, db_config, num_pages=5):
    """Run complete ETL pipeline"""
    
    all_metadata = []
    all_media = []
    all_colors = []
    
    # Extract
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, page=page, size=100)
        artifacts = data['records']
        
        # Transform
        metadata_df, media_df, colors_df = transform_artifact_data(artifacts)
        all_metadata.append(metadata_df)
        all_media.append(media_df)
        all_colors.append(colors_df)
    
    # Combine all dataframes
    final_metadata = pd.concat(all_metadata, ignore_index=True)
    final_media = pd.concat(all_media, ignore_index=True)
    final_colors = pd.concat(all_colors, ignore_index=True)
    
    # Load
    load_to_database(final_metadata, final_media, final_colors, db_config)
    
    return len(final_metadata)

# Execute pipeline
total_artifacts = run_etl_pipeline(
    api_key=os.getenv('HARVARD_API_KEY'),
    db_config=db_config,
    num_pages=10
)
print(f"Pipeline complete: {total_artifacts} artifacts processed")
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(api_key, page, max_retries=3):
    """Fetch with exponential backoff for rate limiting"""
    for attempt in range(max_retries):
        try:
            data = fetch_artifacts(api_key, page)
            return data
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
def test_db_connection(db_config):
    """Test database connectivity"""
    try:
        connection = mysql.connector.connect(**db_config)
        if connection.is_connected():
            print("✓ Database connection successful")
            cursor = connection.cursor()
            cursor.execute("SELECT DATABASE();")
            db_name = cursor.fetchone()
            print(f"✓ Connected to database: {db_name[0]}")
            return True
    except Error as e:
        print(f"✗ Connection failed: {e}")
        return False
    finally:
        if connection.is_connected():
            connection.close()
```

### Handling Missing Data

```python
def safe_transform(artifacts):
    """Transform with null handling"""
    metadata_records = []
    
    for artifact in artifacts:
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title') or 'Unknown',
            'culture': artifact.get('culture') or None,
            'period': artifact.get('period') or None,
            # Use .get() with defaults for all fields
        })
    
    df = pd.DataFrame(metadata_records)
    # Replace empty strings with None for SQL NULL
    df = df.replace('', None)
    return df
```

## Best Practices

1. **Batch Processing**: Load data in batches to avoid memory issues with large datasets
2. **Error Handling**: Wrap API calls and DB operations in try-except blocks
3. **Environment Variables**: Never hardcode credentials; use `.env` files
4. **Incremental Updates**: Track last sync timestamp to avoid re-processing artifacts
5. **Data Validation**: Validate API responses before database insertion
6. **Indexing**: Create indexes on frequently queried columns (culture, century, classification)
