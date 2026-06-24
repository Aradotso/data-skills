---
name: harvard-artifacts-data-engineering-pipeline
description: End-to-end ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data engineering pipeline with museum artifacts
  - set up Harvard API data collection and analytics
  - analyze Harvard Art Museums collection data
  - build artifact data warehouse with visualization
  - create museum data pipeline with Streamlit dashboard
  - extract and transform Harvard museum API data
  - design SQL analytics for art collection data
---

# Harvard Artifacts Collection Data Engineering & Analytics App

> Skill by [ara.so](https://ara.so) — Data Skills collection.

An end-to-end data engineering and analytics application that demonstrates production-grade ETL pipelines using the Harvard Art Museums API. The project extracts artifact metadata, transforms nested JSON into relational structures, loads data into SQL databases, and provides interactive analytics through a Streamlit dashboard.

## What This Project Does

This application showcases a complete data engineering workflow:
- **Extract**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **Transform**: Converts nested JSON into normalized relational tables
- **Load**: Batch inserts into MySQL/TiDB Cloud with proper relationships
- **Analyze**: Executes 20+ analytical SQL queries for insights
- **Visualize**: Interactive Plotly charts in Streamlit dashboard

The architecture follows: `API → ETL → SQL → Analytics → Visualization`

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

Required packages:
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Database Setup

The project uses three main tables with foreign key relationships:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255),
    period VARCHAR(255),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accession_number VARCHAR(100),
    PRIMARY KEY (id)
);

-- Artifact media table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(50),
    base_url VARCHAR(500),
    image_id VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Configuration

### API Configuration

The Harvard Art Museums API requires registration at https://www.harvardartmuseums.org/collections/api

Store credentials in `.env`:
```env
HARVARD_API_KEY=your_api_key_here
API_BASE_URL=https://api.harvardartmuseums.org/object
```

### Database Configuration

For MySQL/TiDB Cloud connection:
```env
DB_HOST=gateway01.your-region.prod.aws.tidbcloud.com
DB_PORT=4000
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
DB_SSL=true
```

## ETL Pipeline Implementation

### Extract: API Data Collection

```python
import requests
import pandas as pd
import os
from time import sleep

def fetch_artifacts(api_key, num_pages=10, page_size=100):
    """
    Fetch artifact data from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_records = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        try:
            response = requests.get(base_url, params=params)
            response.raise_for_status()
            data = response.json()
            
            if 'records' in data:
                all_records.extend(data['records'])
                print(f"Fetched page {page}/{num_pages}: {len(data['records'])} records")
            
            # Rate limiting - respect API limits
            sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            continue
    
    return all_records
```

### Transform: JSON to Relational Structure

```python
def transform_artifacts(raw_data):
    """
    Transform nested JSON into normalized dataframes for SQL insertion
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:255],
            'department': artifact.get('department', '')[:255],
            'division': artifact.get('division', '')[:255],
            'dated': artifact.get('dated', '')[:255],
            'period': artifact.get('period', '')[:255],
            'medium': artifact.get('medium', '')[:500],
            'dimensions': artifact.get('dimensions', '')[:500],
            'creditline': artifact.get('creditline', ''),
            'accession_number': artifact.get('accessionnumber', '')[:100]
        }
        metadata_records.append(metadata)
        
        # Extract media/images
        if 'images' in artifact and artifact['images']:
            for image in artifact['images']:
                media = {
                    'artifact_id': artifact.get('id'),
                    'media_type': 'image',
                    'base_url': image.get('baseimageurl', '')[:500],
                    'image_id': image.get('imageid', '')[:100]
                }
                media_records.append(media)
        
        # Extract colors
        if 'colors' in artifact and artifact['colors']:
            for color in artifact['colors']:
                color_record = {
                    'artifact_id': artifact.get('id'),
                    'color_hex': color.get('hex', '')[:10],
                    'color_percent': float(color.get('percent', 0))
                }
                color_records.append(color_record)
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### Load: Batch Insert into SQL

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(metadata_df, media_df, colors_df, db_config):
    """
    Load transformed dataframes into SQL database with error handling
    """
    try:
        connection = mysql.connector.connect(**db_config)
        cursor = connection.cursor()
        
        # Insert metadata (parent table first)
        metadata_query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, department, 
             division, dated, period, medium, dimensions, creditline, accession_number)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        
        metadata_values = metadata_df.values.tolist()
        cursor.executemany(metadata_query, metadata_values)
        print(f"Inserted {cursor.rowcount} metadata records")
        
        # Insert media (child table)
        if not media_df.empty:
            media_query = """
                INSERT INTO artifactmedia (artifact_id, media_type, base_url, image_id)
                VALUES (%s, %s, %s, %s)
            """
            media_values = media_df.values.tolist()
            cursor.executemany(media_query, media_values)
            print(f"Inserted {cursor.rowcount} media records")
        
        # Insert colors (child table)
        if not colors_df.empty:
            colors_query = """
                INSERT INTO artifactcolors (artifact_id, color_hex, color_percent)
                VALUES (%s, %s, %s)
            """
            colors_values = colors_df.values.tolist()
            cursor.executemany(colors_query, colors_values)
            print(f"Inserted {cursor.rowcount} color records")
        
        connection.commit()
        
    except Error as e:
        print(f"Database error: {e}")
        connection.rollback()
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

## Streamlit Application Structure

### Main Application Entry Point

```python
import streamlit as st
import plotly.express as px
from dotenv import load_dotenv

load_dotenv()

def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museums - Data Engineering & Analytics")
    st.markdown("---")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Navigation",
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

if __name__ == "__main__":
    main()
```

### Data Collection Page

```python
def show_data_collection():
    st.header("📥 API Data Collection")
    
    api_key = st.text_input("Harvard API Key", type="password", 
                            value=os.getenv("HARVARD_API_KEY", ""))
    
    col1, col2 = st.columns(2)
    with col1:
        num_pages = st.number_input("Number of Pages", min_value=1, max_value=100, value=5)
    with col2:
        page_size = st.number_input("Records per Page", min_value=10, max_value=100, value=50)
    
    if st.button("Fetch Data"):
        with st.spinner("Fetching artifacts from API..."):
            raw_data = fetch_artifacts(api_key, num_pages, page_size)
            st.session_state['raw_data'] = raw_data
            st.success(f"✅ Fetched {len(raw_data)} artifacts successfully!")
            
            # Preview
            st.subheader("Data Preview")
            st.json(raw_data[0] if raw_data else {})
```

### SQL Analytics Page

```python
def show_sql_analytics():
    st.header("📊 SQL Analytics Dashboard")
    
    # Predefined analytical queries
    queries = {
        "Artifacts by Culture": """
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
        """,
        "Department Distribution": """
            SELECT department, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY artifact_count DESC
        """,
        "Top 10 Colors Used": """
            SELECT color_hex, COUNT(*) as usage_count,
                   AVG(color_percent) as avg_percent
            FROM artifactcolors
            GROUP BY color_hex
            ORDER BY usage_count DESC
            LIMIT 10
        """,
        "Media Availability": """
            SELECT 
                CASE WHEN media_count > 0 THEN 'Has Media' ELSE 'No Media' END as media_status,
                COUNT(*) as artifact_count
            FROM (
                SELECT m.id, COUNT(med.id) as media_count
                FROM artifactmetadata m
                LEFT JOIN artifactmedia med ON m.id = med.artifact_id
                GROUP BY m.id
            ) as media_summary
            GROUP BY media_status
        """
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        db_config = {
            'host': os.getenv('DB_HOST'),
            'user': os.getenv('DB_USER'),
            'password': os.getenv('DB_PASSWORD'),
            'database': os.getenv('DB_NAME')
        }
        
        results_df = execute_query(queries[selected_query], db_config)
        
        if not results_df.empty:
            st.dataframe(results_df, use_container_width=True)
            
            # Auto-generate visualization
            if len(results_df.columns) == 2:
                fig = px.bar(results_df, 
                           x=results_df.columns[0], 
                           y=results_df.columns[1],
                           title=selected_query)
                st.plotly_chart(fig, use_container_width=True)

def execute_query(query, db_config):
    """Execute SQL query and return results as DataFrame"""
    try:
        connection = mysql.connector.connect(**db_config)
        df = pd.read_sql(query, connection)
        connection.close()
        return df
    except Error as e:
        st.error(f"Query error: {e}")
        return pd.DataFrame()
```

## Common Patterns

### Complete ETL Workflow

```python
def run_complete_etl():
    """
    Execute complete ETL pipeline from API to Database
    """
    # 1. Extract
    api_key = os.getenv('HARVARD_API_KEY')
    raw_data = fetch_artifacts(api_key, num_pages=10, page_size=100)
    
    # 2. Transform
    metadata_df, media_df, colors_df = transform_artifacts(raw_data)
    
    # 3. Load
    db_config = {
        'host': os.getenv('DB_HOST'),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME'),
        'port': int(os.getenv('DB_PORT', 3306))
    }
    
    load_to_database(metadata_df, media_df, colors_df, db_config)
    
    return {
        'metadata_count': len(metadata_df),
        'media_count': len(media_df),
        'colors_count': len(colors_df)
    }
```

### Incremental Data Updates

```python
def incremental_load(last_update_timestamp):
    """
    Fetch only new/updated artifacts since last run
    """
    params = {
        'apikey': os.getenv('HARVARD_API_KEY'),
        'updatedafter': last_update_timestamp,
        'size': 100
    }
    
    response = requests.get(
        "https://api.harvardartmuseums.org/object",
        params=params
    )
    
    new_records = response.json().get('records', [])
    
    if new_records:
        metadata_df, media_df, colors_df = transform_artifacts(new_records)
        load_to_database(metadata_df, media_df, colors_df, db_config)
    
    return len(new_records)
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# Run ETL pipeline standalone
python etl_pipeline.py

# Execute specific analytics query
python run_query.py --query "artifacts_by_culture"
```

## Troubleshooting

### API Rate Limiting
```python
# Add exponential backoff for API calls
from time import sleep
import random

def fetch_with_retry(url, params, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params)
            if response.status_code == 429:  # Rate limited
                wait_time = (2 ** attempt) + random.random()
                sleep(wait_time)
                continue
            response.raise_for_status()
            return response.json()
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            sleep(2 ** attempt)
```

### Database Connection Issues
```python
# Add SSL configuration for TiDB Cloud
db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME'),
    'ssl_ca': '/path/to/ca-cert.pem',
    'ssl_verify_cert': True,
    'ssl_verify_identity': True
}
```

### Memory Management for Large Datasets
```python
# Process data in chunks to avoid memory issues
def load_in_chunks(df, chunk_size=1000):
    for start in range(0, len(df), chunk_size):
        chunk = df.iloc[start:start + chunk_size]
        load_to_database(chunk, db_config)
        print(f"Loaded chunk {start//chunk_size + 1}")
```

### Data Quality Validation
```python
def validate_data(metadata_df):
    """
    Check for data quality issues before loading
    """
    issues = []
    
    # Check for null IDs
    if metadata_df['id'].isnull().any():
        issues.append("Found null IDs in metadata")
    
    # Check for duplicates
    if metadata_df['id'].duplicated().any():
        issues.append("Found duplicate artifact IDs")
    
    # Check string length constraints
    if (metadata_df['title'].str.len() > 500).any():
        issues.append("Title exceeds 500 characters")
    
    return issues
```

This skill provides comprehensive guidance for building production-grade ETL pipelines with the Harvard Art Museums API, including data extraction, transformation, SQL storage, and interactive analytics visualization.
