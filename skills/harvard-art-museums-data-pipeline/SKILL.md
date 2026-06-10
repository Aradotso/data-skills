---
name: harvard-art-museums-data-pipeline
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - how do I build a data pipeline with Harvard Art Museums API
  - set up ETL for museum artifact data
  - create analytics dashboard for Harvard art collection
  - integrate Harvard Art Museums API with SQL database
  - build Streamlit app for museum data visualization
  - extract and analyze Harvard artifacts data
  - design museum collection data engineering pipeline
  - query and visualize art museum metadata
---

# Harvard Art Museums Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. It demonstrates ETL pipeline construction, SQL database design, analytical querying, and interactive visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering & Analytics App provides:

- **API Integration**: Fetches artifact metadata from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads artifact data into relational SQL databases
- **Database Design**: Structures data across multiple tables (metadata, media, colors) with proper relationships
- **Analytics Queries**: Executes predefined SQL queries for insights on artifacts, cultures, centuries, and media
- **Visualization Dashboard**: Interactive Streamlit interface with Plotly charts for data exploration

**Architecture Flow**: API → ETL → SQL → Analytics → Visualization

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
export DB_NAME="your_database_name"
```

**Requirements** (typical setup):
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py
```

The app will launch at `http://localhost:8501` with tabs for data collection, ETL processing, and analytics.

## Core Components

### 1. API Data Collection

Fetch artifact data from Harvard Art Museums API:

```python
import requests
import os

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts with pagination"""
    params = {
        'apikey': API_KEY,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(BASE_URL, params=params)
    
    if response.status_code == 200:
        data = response.json()
        return {
            'records': data.get('records', []),
            'total_pages': data['info']['pages'],
            'total_records': data['info']['totalrecords']
        }
    else:
        raise Exception(f"API Error: {response.status_code}")

# Fetch multiple pages
def collect_artifacts(num_pages=5):
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        result = fetch_artifacts(page=page)
        all_artifacts.extend(result['records'])
    
    return all_artifacts
```

### 2. ETL Pipeline

Transform and load data into SQL database:

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

def create_database_connection():
    """Establish MySQL/TiDB connection"""
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

def create_tables(connection):
    """Create database schema"""
    cursor = connection.cursor()
    
    # Artifact Metadata Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(255),
            century VARCHAR(100),
            classification VARCHAR(255),
            department VARCHAR(255),
            division VARCHAR(255),
            dated VARCHAR(255),
            medium VARCHAR(500),
            technique VARCHAR(500),
            period VARCHAR(255),
            accession_year INT,
            url TEXT
        )
    """)
    
    # Media Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            media_type VARCHAR(50),
            base_url TEXT,
            image_url TEXT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    # Colors Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(50),
            spectrum VARCHAR(100),
            percentage FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    connection.commit()
    cursor.close()

def transform_artifacts(artifacts):
    """Transform raw API data into structured dataframes"""
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'medium': artifact.get('medium'),
            'technique': artifact.get('technique'),
            'period': artifact.get('period'),
            'accession_year': artifact.get('accessionyear'),
            'url': artifact.get('url')
        }
        metadata_list.append(metadata)
        
        # Extract media
        if artifact.get('images'):
            for img in artifact['images']:
                media = {
                    'artifact_id': artifact['id'],
                    'media_type': 'image',
                    'base_url': img.get('baseimageurl'),
                    'image_url': img.get('iiifbaseuri')
                }
                media_list.append(media)
        
        # Extract colors
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_data = {
                    'artifact_id': artifact['id'],
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'percentage': color.get('percent')
                }
                colors_list.append(color_data)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )

def load_data(connection, metadata_df, media_df, colors_df):
    """Load dataframes into SQL database"""
    cursor = connection.cursor()
    
    # Insert metadata
    for _, row in metadata_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, department, 
             division, dated, medium, technique, period, accession_year, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """, tuple(row))
    
    # Insert media
    for _, row in media_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmedia 
            (artifact_id, media_type, base_url, image_url)
            VALUES (%s, %s, %s, %s)
        """, tuple(row))
    
    # Insert colors
    for _, row in colors_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, percentage)
            VALUES (%s, %s, %s, %s)
        """, tuple(row))
    
    connection.commit()
    cursor.close()
```

### 3. Analytics Queries

Execute analytical SQL queries:

```python
def execute_query(connection, query):
    """Execute SQL query and return results as DataFrame"""
    try:
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        print(f"Query execution error: {e}")
        return None

# Sample analytical queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY artifact_count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "Color Distribution": """
        SELECT color, COUNT(*) as frequency, 
               AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "Media Availability": """
        SELECT 
            COUNT(DISTINCT m.artifact_id) as artifacts_with_media,
            COUNT(*) as total_media_items
        FROM artifactmedia m
    """,
    
    "Artifacts by Classification": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Accession Year Trends": """
        SELECT accession_year, COUNT(*) as acquisitions
        FROM artifactmetadata
        WHERE accession_year IS NOT NULL
        GROUP BY accession_year
        ORDER BY accession_year DESC
        LIMIT 20
    """
}
```

### 4. Streamlit Dashboard

Build interactive visualization interface:

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Data Analytics")
    st.markdown("*End-to-End Data Engineering Pipeline*")
    
    # Tabs
    tab1, tab2, tab3 = st.tabs(["📥 Data Collection", "🔄 ETL Pipeline", "📊 Analytics"])
    
    with tab1:
        st.header("Collect Artifact Data")
        num_pages = st.slider("Number of pages to fetch", 1, 10, 5)
        
        if st.button("Fetch Data"):
            with st.spinner("Collecting artifacts..."):
                artifacts = collect_artifacts(num_pages)
                st.success(f"Collected {len(artifacts)} artifacts")
                st.session_state.artifacts = artifacts
    
    with tab2:
        st.header("ETL Processing")
        
        if st.button("Run ETL Pipeline"):
            if 'artifacts' in st.session_state:
                with st.spinner("Running ETL..."):
                    connection = create_database_connection()
                    create_tables(connection)
                    
                    metadata_df, media_df, colors_df = transform_artifacts(
                        st.session_state.artifacts
                    )
                    
                    load_data(connection, metadata_df, media_df, colors_df)
                    connection.close()
                    
                    st.success("ETL pipeline completed!")
                    st.write(f"Loaded {len(metadata_df)} artifacts")
            else:
                st.warning("Please collect data first")
    
    with tab3:
        st.header("Analytics Dashboard")
        
        query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
        
        if st.button("Run Query"):
            connection = create_database_connection()
            df = execute_query(connection, ANALYTICS_QUERIES[query_name])
            connection.close()
            
            if df is not None and not df.empty:
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

if __name__ == "__main__":
    main()
```

## Configuration

### Environment Variables

Create a `.env` file:

```env
HARVARD_API_KEY=your_api_key_from_harvard
DB_HOST=gateway01.us-east-1.prod.aws.tidbcloud.com
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
DB_PORT=4000
```

### Getting Harvard API Key

1. Visit [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Register for a free API key
3. Add to environment variables

## Common Patterns

### Batch Processing with Rate Limiting

```python
import time

def batch_collect_with_rate_limit(total_pages, batch_size=5, delay=1):
    """Collect data in batches with rate limiting"""
    all_artifacts = []
    
    for i in range(0, total_pages, batch_size):
        batch_end = min(i + batch_size, total_pages)
        
        for page in range(i + 1, batch_end + 1):
            artifacts = fetch_artifacts(page=page)
            all_artifacts.extend(artifacts['records'])
            time.sleep(delay)  # Rate limiting
        
        print(f"Processed {len(all_artifacts)} artifacts so far...")
    
    return all_artifacts
```

### Error Handling in ETL

```python
def safe_etl_pipeline(artifacts):
    """ETL with error handling and logging"""
    try:
        connection = create_database_connection()
        if not connection:
            raise Exception("Database connection failed")
        
        create_tables(connection)
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        
        # Validate data
        if metadata_df.empty:
            raise ValueError("No metadata extracted")
        
        load_data(connection, metadata_df, media_df, colors_df)
        connection.close()
        
        return {
            'status': 'success',
            'metadata_count': len(metadata_df),
            'media_count': len(media_df),
            'colors_count': len(colors_df)
        }
    
    except Exception as e:
        print(f"ETL Error: {e}")
        return {'status': 'failed', 'error': str(e)}
```

## Troubleshooting

### API Rate Limiting
- Harvard API has rate limits; add delays between requests
- Use `time.sleep()` between pagination calls
- Cache responses to avoid redundant API calls

### Database Connection Issues
- Verify environment variables are set correctly
- Check firewall rules for TiDB Cloud or MySQL
- Ensure SSL/TLS settings if using cloud databases

### Data Quality Issues
- Some fields may be `None` - handle gracefully with `.get()`
- Use `ON DUPLICATE KEY UPDATE` for idempotent inserts
- Validate data types before SQL insertion

### Memory Management
- Process large datasets in chunks
- Use generators for streaming data
- Clear session state in Streamlit to prevent memory leaks

```python
# Chunk processing example
def process_in_chunks(artifacts, chunk_size=100):
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i + chunk_size]
        yield chunk
```
