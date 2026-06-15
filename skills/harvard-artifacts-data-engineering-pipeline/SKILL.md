---
name: harvard-artifacts-data-engineering-pipeline
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums API
  - create a data engineering project with Harvard artifacts
  - set up analytics dashboard for museum artifacts data
  - extract and load Harvard museum data to SQL database
  - build Streamlit app for art museum data visualization
  - analyze Harvard Art Museums collection with SQL queries
  - create ETL workflow for museum artifacts API
  - visualize Harvard art collection data with Plotly
---

# Harvard Artifacts Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables building end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What It Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:
- API integration with Harvard Art Museums to collect artifact data
- ETL pipeline to extract, transform, and load data into SQL databases
- Relational database schema for artifacts, media, and color metadata
- 20+ predefined analytical SQL queries for insights
- Interactive Streamlit dashboard with Plotly visualizations
- Batch processing and pagination for large datasets

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

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Architecture Overview

**Pipeline Flow:** API → Extract → Transform → Load → SQL Analytics → Visualization

- **Data Source:** Harvard Art Museums API
- **ETL:** Python with Requests and Pandas
- **Storage:** MySQL/TiDB Cloud
- **Analytics:** SQL queries
- **Visualization:** Streamlit + Plotly

## Database Schema

The ETL pipeline creates three main tables:

### artifactmetadata
```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    classification VARCHAR(255),
    dated VARCHAR(255),
    century VARCHAR(255),
    division VARCHAR(255),
    department VARCHAR(255),
    accession_number VARCHAR(255),
    description TEXT
);
```

### artifactmedia
```sql
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    media_url TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

### artifactcolors
```sql
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### API Data Extraction

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
        'size': size
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=100)
artifacts = data['records']
total_records = data['info']['totalrecords']
```

### Data Transformation

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """
    Transform raw API data into structured metadata
    """
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown')[:500],
            'culture': artifact.get('culture', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'dated': artifact.get('dated', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'division': artifact.get('division', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'accession_number': artifact.get('accessionyear', 'Unknown'),
            'description': artifact.get('description', '')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(artifacts):
    """
    Extract media URLs from artifacts
    """
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for image in images:
            media = {
                'artifact_id': artifact_id,
                'media_type': 'image',
                'media_url': image.get('baseimageurl', '')
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(artifacts):
    """
    Extract color data from artifacts
    """
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color_hex': color.get('hex', ''),
                'color_percent': color.get('percent', 0.0)
            }
            color_list.append(color_data)
    
    return pd.DataFrame(color_list)
```

### Database Loading

```python
import mysql.connector
from mysql.connector import Error

def create_db_connection():
    """
    Create MySQL database connection
    """
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_metadata_to_db(df_metadata, connection):
    """
    Batch insert metadata into database
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, classification, dated, century, 
     division, department, accession_number, description)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    values = df_metadata.to_records(index=False).tolist()
    cursor.executemany(insert_query, values)
    connection.commit()
    
    print(f"Inserted {cursor.rowcount} metadata records")

def load_media_to_db(df_media, connection):
    """
    Batch insert media data into database
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (artifact_id, media_type, media_url)
    VALUES (%s, %s, %s)
    """
    
    values = df_media.to_records(index=False).tolist()
    cursor.executemany(insert_query, values)
    connection.commit()
    
    print(f"Inserted {cursor.rowcount} media records")
```

## Complete ETL Workflow

```python
def run_etl_pipeline(num_pages=5):
    """
    Execute complete ETL pipeline
    """
    api_key = os.getenv('HARVARD_API_KEY')
    connection = create_db_connection()
    
    if not connection:
        print("Failed to connect to database")
        return
    
    all_artifacts = []
    
    # Extract
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, page=page, size=100)
        all_artifacts.extend(data['records'])
    
    print(f"Total artifacts extracted: {len(all_artifacts)}")
    
    # Transform
    df_metadata = transform_artifact_metadata(all_artifacts)
    df_media = transform_artifact_media(all_artifacts)
    df_colors = transform_artifact_colors(all_artifacts)
    
    # Load
    load_metadata_to_db(df_metadata, connection)
    load_media_to_db(df_media, connection)
    # Similar for colors...
    
    connection.close()
    print("ETL pipeline completed successfully")

# Execute
run_etl_pipeline(num_pages=10)
```

## SQL Analytics Queries

### Query 1: Artifacts by Culture
```python
def get_artifacts_by_culture(connection):
    query = """
    SELECT culture, COUNT(*) as count
    FROM artifactmetadata
    WHERE culture != 'Unknown'
    GROUP BY culture
    ORDER BY count DESC
    LIMIT 10
    """
    return pd.read_sql(query, connection)
```

### Query 2: Media Availability
```python
def get_media_availability(connection):
    query = """
    SELECT 
        CASE WHEN m.artifact_id IS NOT NULL THEN 'Has Media' ELSE 'No Media' END as status,
        COUNT(*) as count
    FROM artifactmetadata a
    LEFT JOIN artifactmedia m ON a.id = m.artifact_id
    GROUP BY status
    """
    return pd.read_sql(query, connection)
```

### Query 3: Top Colors
```python
def get_top_colors(connection):
    query = """
    SELECT color_hex, SUM(color_percent) as total_percent
    FROM artifactcolors
    GROUP BY color_hex
    ORDER BY total_percent DESC
    LIMIT 15
    """
    return pd.read_sql(query, connection)
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Queries")
    query_option = st.sidebar.selectbox(
        "Select Analysis",
        [
            "Artifacts by Culture",
            "Media Availability",
            "Top Colors",
            "Artifacts by Century",
            "Department Distribution"
        ]
    )
    
    # Database connection
    connection = create_db_connection()
    
    # Execute selected query
    if query_option == "Artifacts by Culture":
        df = get_artifacts_by_culture(connection)
        st.subheader("Top 10 Cultures by Artifact Count")
        st.dataframe(df)
        
        # Visualization
        fig = px.bar(df, x='culture', y='count', 
                     title="Artifacts by Culture",
                     labels={'count': 'Number of Artifacts', 'culture': 'Culture'})
        st.plotly_chart(fig)
    
    elif query_option == "Top Colors":
        df = get_top_colors(connection)
        st.subheader("Most Common Colors in Collection")
        st.dataframe(df)
        
        fig = px.bar(df, x='color_hex', y='total_percent',
                     color='color_hex',
                     title="Color Distribution")
        st.plotly_chart(fig)
    
    connection.close()

if __name__ == "__main__":
    main()
```

## Configuration

### Environment Variables

```bash
# Required API key from Harvard Art Museums
HARVARD_API_KEY=your_api_key_here

# Database configuration
DB_HOST=localhost
DB_USER=root
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
DB_PORT=3306
```

### requirements.txt

```
streamlit>=1.20.0
pandas>=1.5.0
requests>=2.28.0
mysql-connector-python>=8.0.0
plotly>=5.13.0
python-dotenv>=0.21.0
```

## Common Patterns

### Pagination Handling

```python
def fetch_all_artifacts(api_key, max_records=1000):
    """
    Fetch artifacts with automatic pagination
    """
    artifacts = []
    page = 1
    size = 100
    
    while len(artifacts) < max_records:
        data = fetch_artifacts(api_key, page=page, size=size)
        artifacts.extend(data['records'])
        
        if len(data['records']) < size:
            break
        
        page += 1
    
    return artifacts[:max_records]
```

### Rate Limiting

```python
import time

def fetch_with_rate_limit(api_key, pages, delay=1):
    """
    Fetch data with rate limiting to avoid API throttling
    """
    artifacts = []
    
    for page in range(1, pages + 1):
        data = fetch_artifacts(api_key, page=page)
        artifacts.extend(data['records'])
        time.sleep(delay)  # Wait between requests
    
    return artifacts
```

### Error Handling

```python
def safe_etl_execution():
    """
    ETL with comprehensive error handling
    """
    try:
        connection = create_db_connection()
        api_key = os.getenv('HARVARD_API_KEY')
        
        if not api_key:
            raise ValueError("HARVARD_API_KEY not set")
        
        artifacts = fetch_all_artifacts(api_key, max_records=500)
        
        df_metadata = transform_artifact_metadata(artifacts)
        load_metadata_to_db(df_metadata, connection)
        
    except Error as db_error:
        print(f"Database error: {db_error}")
    except requests.RequestException as api_error:
        print(f"API error: {api_error}")
    except Exception as e:
        print(f"Unexpected error: {e}")
    finally:
        if connection and connection.is_connected():
            connection.close()
```

## Troubleshooting

### API Key Issues
- Obtain API key from https://www.harvardartmuseums.org/collections/api
- Verify key is set in environment: `echo $HARVARD_API_KEY`
- Check API rate limits (default: 2500 requests/day)

### Database Connection Errors
```python
# Test connection
def test_db_connection():
    try:
        conn = create_db_connection()
        if conn.is_connected():
            print("Database connection successful")
            conn.close()
        else:
            print("Connection failed")
    except Error as e:
        print(f"Error: {e}")
```

### Empty Data Returns
- Check if API endpoint is accessible
- Verify pagination parameters are correct
- Ensure data fields exist in API response

### Streamlit Port Conflicts
```bash
# Use custom port
streamlit run app.py --server.port 8502
```

### Memory Issues with Large Datasets
```python
# Process in chunks
def process_in_chunks(artifacts, chunk_size=100):
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i+chunk_size]
        df = transform_artifact_metadata(chunk)
        load_metadata_to_db(df, connection)
```
