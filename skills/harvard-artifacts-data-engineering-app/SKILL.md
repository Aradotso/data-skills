---
name: harvard-artifacts-data-engineering-app
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - set up Harvard artifacts collection analytics dashboard
  - extract and analyze Harvard museum API data
  - create data pipeline with Harvard Art Museums API
  - build Streamlit app for museum artifact analytics
  - configure SQL database for Harvard artifacts ETL
  - visualize Harvard museum collection data
  - implement batch data ingestion for art museums
---

# Harvard Artifacts Data Engineering App

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is an end-to-end data engineering and analytics application that demonstrates real-world ETL pipelines using the Harvard Art Museums API. It extracts artifact data, transforms nested JSON into relational structures, loads data into SQL databases (MySQL/TiDB Cloud), and provides interactive analytics dashboards using Streamlit and Plotly.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

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

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Getting Harvard API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Register for a free API key
3. Add key to your `.env` file

### Database Schema

The application uses three main tables:

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    classification VARCHAR(255),
    century VARCHAR(100),
    department VARCHAR(255),
    division VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dated VARCHAR(255),
    period VARCHAR(255),
    accessionyear INT,
    dimensions VARCHAR(500),
    creditline TEXT,
    provenance TEXT,
    people TEXT,
    url VARCHAR(500)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    mediaid INT PRIMARY KEY AUTO_INCREMENT,
    artifactid INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    totalpagviews INT,
    totaluniquepageviews INT,
    FOREIGN KEY (artifactid) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    colorid INT PRIMARY KEY AUTO_INCREMENT,
    artifactid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent DECIMAL(5,2),
    FOREIGN KEY (artifactid) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Access the dashboard at http://localhost:8501
```

## Key Components

### 1. ETL Pipeline Implementation

#### Extract: API Data Collection

```python
import requests
import pandas as pd
import os

def fetch_artifacts(api_key, num_pages=10, page_size=100):
    """
    Extract artifact data from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        try:
            response = requests.get(base_url, params=params, timeout=30)
            response.raise_for_status()
            data = response.json()
            
            if 'records' in data:
                all_artifacts.extend(data['records'])
                print(f"Fetched page {page}: {len(data['records'])} records")
            else:
                print(f"No records found on page {page}")
                break
                
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return all_artifacts
```

#### Transform: Data Normalization

```python
def transform_artifacts_to_relational(artifacts):
    """
    Transform nested JSON into relational table structures
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'classification': artifact.get('classification', '')[:255],
            'century': artifact.get('century', '')[:100],
            'department': artifact.get('department', '')[:255],
            'division': artifact.get('division', '')[:255],
            'technique': artifact.get('technique', '')[:500],
            'medium': artifact.get('medium', '')[:500],
            'dated': artifact.get('dated', '')[:255],
            'period': artifact.get('period', '')[:255],
            'accessionyear': artifact.get('accessionyear'),
            'dimensions': artifact.get('dimensions', '')[:500],
            'creditline': artifact.get('creditline', ''),
            'provenance': artifact.get('provenance', ''),
            'people': str(artifact.get('people', [])),
            'url': artifact.get('url', '')[:500]
        }
        metadata_records.append(metadata)
        
        # Extract media information
        media = {
            'artifactid': artifact.get('id'),
            'baseimageurl': artifact.get('baseimageurl', '')[:500],
            'primaryimageurl': artifact.get('primaryimageurl', '')[:500],
            'totalpagviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        }
        media_records.append(media)
        
        # Extract color data
        if 'colors' in artifact and artifact['colors']:
            for color in artifact['colors']:
                color_record = {
                    'artifactid': artifact.get('id'),
                    'color': color.get('color', '')[:50],
                    'spectrum': color.get('spectrum', '')[:50],
                    'hue': color.get('hue', '')[:50],
                    'percent': color.get('percent', 0.0)
                }
                color_records.append(color_record)
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

#### Load: Batch Database Insertion

```python
import mysql.connector
from mysql.connector import Error

def load_data_to_database(metadata_df, media_df, colors_df, db_config):
    """
    Load transformed data into SQL database with batch inserts
    """
    try:
        connection = mysql.connector.connect(**db_config)
        cursor = connection.cursor()
        
        # Load metadata
        metadata_insert = """
        INSERT IGNORE INTO artifactmetadata 
        (id, title, culture, classification, century, department, 
         division, technique, medium, dated, period, accessionyear, 
         dimensions, creditline, provenance, people, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """
        metadata_values = metadata_df.values.tolist()
        cursor.executemany(metadata_insert, metadata_values)
        
        # Load media
        media_insert = """
        INSERT INTO artifactmedia 
        (artifactid, baseimageurl, primaryimageurl, totalpagviews, totaluniquepageviews)
        VALUES (%s, %s, %s, %s, %s)
        """
        media_values = media_df.values.tolist()
        cursor.executemany(media_insert, media_values)
        
        # Load colors
        if not colors_df.empty:
            colors_insert = """
            INSERT INTO artifactcolors 
            (artifactid, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
            """
            colors_values = colors_df.values.tolist()
            cursor.executemany(colors_insert, colors_values)
        
        connection.commit()
        print(f"Successfully loaded {len(metadata_df)} artifacts")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

### 2. SQL Analytics Queries

```python
# Sample analytical queries for the dashboard

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
        SELECT century, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY artifact_count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "Most Viewed Artifacts": """
        SELECT m.title, m.culture, m.century, 
               med.totalpagviews, med.primaryimageurl
        FROM artifactmetadata m
        JOIN artifactmedia med ON m.id = med.artifactid
        WHERE med.totalpagviews > 0
        ORDER BY med.totalpagviews DESC
        LIMIT 20
    """,
    
    "Color Distribution": """
        SELECT color, COUNT(*) as usage_count, 
               AVG(percent) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "Artifacts with Complete Metadata": """
        SELECT 
            COUNT(*) as total_artifacts,
            SUM(CASE WHEN culture IS NOT NULL THEN 1 ELSE 0 END) as with_culture,
            SUM(CASE WHEN century IS NOT NULL THEN 1 ELSE 0 END) as with_century,
            SUM(CASE WHEN technique IS NOT NULL THEN 1 ELSE 0 END) as with_technique
        FROM artifactmetadata
    """,
    
    "Accession Year Trends": """
        SELECT accessionyear, COUNT(*) as artifacts_acquired
        FROM artifactmetadata
        WHERE accessionyear IS NOT NULL AND accessionyear > 1800
        GROUP BY accessionyear
        ORDER BY accessionyear DESC
    """
}
```

### 3. Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px
import plotly.graph_objects as go

def execute_query(query, db_config):
    """Execute SQL query and return results as DataFrame"""
    connection = mysql.connector.connect(**db_config)
    df = pd.read_sql(query, connection)
    connection.close()
    return df

def create_visualization(df, title):
    """Create interactive Plotly visualization"""
    if len(df.columns) >= 2:
        x_col = df.columns[0]
        y_col = df.columns[1]
        
        fig = px.bar(
            df, 
            x=x_col, 
            y=y_col,
            title=title,
            labels={x_col: x_col.replace('_', ' ').title(),
                    y_col: y_col.replace('_', ' ').title()},
            color=y_col,
            color_continuous_scale='Viridis'
        )
        
        fig.update_layout(
            xaxis_tickangle=-45,
            height=500,
            showlegend=False
        )
        
        return fig
    return None

# Streamlit app structure
st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Sidebar for query selection
st.sidebar.header("Analytics Options")
query_name = st.sidebar.selectbox(
    "Select Analysis",
    list(ANALYTICAL_QUERIES.keys())
)

# Database configuration from environment
db_config = {
    'host': os.getenv('DB_HOST'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

# Execute and display results
if st.sidebar.button("Run Analysis"):
    with st.spinner("Executing query..."):
        query = ANALYTICAL_QUERIES[query_name]
        df_result = execute_query(query, db_config)
        
        st.subheader(f"📊 {query_name}")
        
        # Display data table
        st.dataframe(df_result, use_container_width=True)
        
        # Display visualization
        fig = create_visualization(df_result, query_name)
        if fig:
            st.plotly_chart(fig, use_container_width=True)
        
        # Display metrics
        col1, col2, col3 = st.columns(3)
        with col1:
            st.metric("Total Records", len(df_result))
        with col2:
            if len(df_result.columns) >= 2:
                st.metric("Max Value", df_result.iloc[:, 1].max())
        with col3:
            if len(df_result.columns) >= 2:
                st.metric("Average", f"{df_result.iloc[:, 1].mean():.2f}")
```

## Common Patterns

### ETL Pipeline Execution

```python
import os
from dotenv import load_dotenv

def run_etl_pipeline():
    """Complete ETL pipeline execution"""
    load_dotenv()
    
    # Extract
    api_key = os.getenv('HARVARD_API_KEY')
    artifacts = fetch_artifacts(api_key, num_pages=5, page_size=100)
    
    # Transform
    metadata_df, media_df, colors_df = transform_artifacts_to_relational(artifacts)
    
    # Load
    db_config = {
        'host': os.getenv('DB_HOST'),
        'port': int(os.getenv('DB_PORT', 3306)),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    
    load_data_to_database(metadata_df, media_df, colors_df, db_config)
    
    print(f"ETL Complete: {len(metadata_df)} artifacts processed")

if __name__ == "__main__":
    run_etl_pipeline()
```

### Incremental Data Updates

```python
def incremental_etl(last_update_date):
    """
    Fetch only new/updated artifacts since last run
    """
    base_url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': os.getenv('HARVARD_API_KEY'),
        'after': last_update_date,
        'size': 100
    }
    
    response = requests.get(base_url, params=params)
    new_artifacts = response.json().get('records', [])
    
    # Process only new records
    if new_artifacts:
        metadata_df, media_df, colors_df = transform_artifacts_to_relational(new_artifacts)
        load_data_to_database(metadata_df, media_df, colors_df, db_config)
        print(f"Updated {len(new_artifacts)} new artifacts")
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('etl_pipeline.log'),
        logging.StreamHandler()
    ]
)

def safe_api_fetch(api_key, page):
    """API fetch with error handling and retry logic"""
    max_retries = 3
    retry_delay = 5
    
    for attempt in range(max_retries):
        try:
            response = requests.get(
                "https://api.harvardartmuseums.org/object",
                params={'apikey': api_key, 'page': page, 'size': 100},
                timeout=30
            )
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            logging.error(f"Attempt {attempt + 1} failed: {e}")
            if attempt < max_retries - 1:
                time.sleep(retry_delay)
            else:
                logging.error(f"Failed to fetch page {page} after {max_retries} attempts")
                return None
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_rate_limiting(api_key, num_pages, rate_limit_delay=1):
    """
    Fetch data with rate limiting to avoid API throttling
    """
    artifacts = []
    for page in range(1, num_pages + 1):
        data = safe_api_fetch(api_key, page)
        if data and 'records' in data:
            artifacts.extend(data['records'])
        time.sleep(rate_limit_delay)  # Respect rate limits
    return artifacts
```

### Database Connection Issues

```python
def test_database_connection(db_config):
    """Test database connectivity"""
    try:
        connection = mysql.connector.connect(**db_config)
        if connection.is_connected():
            print("✓ Database connection successful")
            cursor = connection.cursor()
            cursor.execute("SELECT DATABASE();")
            db_name = cursor.fetchone()[0]
            print(f"Connected to database: {db_name}")
            return True
    except Error as e:
        print(f"✗ Database connection failed: {e}")
        return False
    finally:
        if connection.is_connected():
            connection.close()
```

### Data Quality Validation

```python
def validate_data_quality(df):
    """Validate data before loading to database"""
    issues = []
    
    # Check for missing IDs
    if df['id'].isnull().any():
        issues.append("Missing artifact IDs detected")
    
    # Check for duplicates
    if df['id'].duplicated().any():
        issues.append(f"Found {df['id'].duplicated().sum()} duplicate IDs")
    
    # Check data types
    if df['accessionyear'].dtype != 'int64':
        issues.append("Accession year has incorrect data type")
    
    if issues:
        print("⚠️ Data Quality Issues:")
        for issue in issues:
            print(f"  - {issue}")
        return False
    
    print("✓ Data quality validation passed")
    return True
```

### Missing Dependencies

```bash
# If imports fail, ensure all dependencies are installed
pip install --upgrade streamlit pandas requests mysql-connector-python plotly python-dotenv

# For TiDB Cloud connection
pip install mysqlclient  # Additional MySQL driver
```

## Advanced Usage

### Custom Query Builder

```python
def build_custom_query(filters):
    """
    Build dynamic SQL queries based on user filters
    """
    base_query = "SELECT * FROM artifactmetadata WHERE 1=1"
    
    if filters.get('culture'):
        base_query += f" AND culture = '{filters['culture']}'"
    
    if filters.get('century'):
        base_query += f" AND century = '{filters['century']}'"
    
    if filters.get('department'):
        base_query += f" AND department = '{filters['department']}'"
    
    base_query += " LIMIT 100"
    return base_query
```

### Export Functionality

```python
def export_query_results(df, format='csv'):
    """Export query results to various formats"""
    timestamp = pd.Timestamp.now().strftime('%Y%m%d_%H%M%S')
    
    if format == 'csv':
        filename = f'artifacts_export_{timestamp}.csv'
        df.to_csv(filename, index=False)
    elif format == 'excel':
        filename = f'artifacts_export_{timestamp}.xlsx'
        df.to_excel(filename, index=False)
    elif format == 'json':
        filename = f'artifacts_export_{timestamp}.json'
        df.to_json(filename, orient='records', indent=2)
    
    return filename
```

This skill provides comprehensive guidance for implementing end-to-end data engineering pipelines with the Harvard Art Museums API, including ETL processes, database management, and interactive analytics dashboards.
