---
name: harvard-artifacts-data-engineering-pipeline
description: Build end-to-end ETL pipelines for Harvard Art Museums API with SQL analytics and Streamlit visualization
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - create a data engineering workflow with Harvard API
  - set up SQL analytics for museum artifacts
  - build a Streamlit dashboard for art collection data
  - extract and transform Harvard museum API data
  - create artifact data pipeline with visualization
  - analyze Harvard art collection with SQL queries
  - build museum data analytics application
---

# Harvard Artifacts Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill helps you build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What It Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:
- **API Integration**: Collect artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color data into SQL databases
- **SQL Analytics**: 20+ predefined analytical queries for insights on artifacts, cultures, departments, and media
- **Interactive Visualization**: Streamlit dashboards with Plotly charts for real-time analytics
- **Database Design**: Relational schema with proper foreign keys across artifact metadata, media, and color tables

## Installation

### Prerequisites

```bash
# Python 3.8+
python --version

# Install required packages
pip install streamlit pandas requests sqlalchemy pymysql plotly python-dotenv
```

### Clone and Setup

```bash
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Create environment file
touch .env
```

### Environment Configuration

Create a `.env` file with your credentials:

```bash
# Harvard API Key (get from https://www.harvardartmuseums.org/collections/api)
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=4000
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

## Database Setup

### SQL Schema

The project uses three main tables with relational design:

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accessionyear INT,
    provenance TEXT,
    url TEXT
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    imagetotal INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

### Start Streamlit Dashboard

```bash
streamlit run app.py
```

The dashboard will open at `http://localhost:8501`

### Application Flow

1. **API Key Configuration**: Enter your Harvard API key in the sidebar
2. **Data Collection**: Specify number of artifacts to fetch (with pagination)
3. **ETL Execution**: Extract, transform, and load data into SQL tables
4. **Analytics Dashboard**: Run predefined SQL queries and visualize results

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, num_artifacts=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    base_url = "https://api.harvardartmuseums.org/object"
    artifacts = []
    page = 1
    size = 100  # Max per page
    
    while len(artifacts) < num_artifacts:
        params = {
            'apikey': api_key,
            'page': page,
            'size': min(size, num_artifacts - len(artifacts))
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            artifacts.extend(data.get('records', []))
            
            if len(data.get('records', [])) == 0:
                break
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_artifacts]
```

### 2. ETL Transformation

```python
import pandas as pd

def transform_artifact_data(artifacts):
    """Transform nested JSON into relational DataFrames"""
    
    # Metadata transformation
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'accessionyear': artifact.get('accessionyear'),
            'provenance': artifact.get('provenance'),
            'url': artifact.get('url')
        }
        metadata_list.append(metadata)
        
        # Extract media
        media = {
            'artifact_id': artifact.get('id'),
            'baseimageurl': artifact.get('baseimageurl'),
            'primaryimageurl': artifact.get('primaryimageurl'),
            'imagetotal': artifact.get('totalpageviews', 0)
        }
        media_list.append(media)
        
        # Extract colors
        colors = artifact.get('colors', [])
        for color in colors:
            color_data = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            colors_list.append(color_data)
    
    df_metadata = pd.DataFrame(metadata_list)
    df_media = pd.DataFrame(media_list)
    df_colors = pd.DataFrame(colors_list)
    
    return df_metadata, df_media, df_colors
```

### 3. SQL Database Loading

```python
from sqlalchemy import create_engine
import os

def load_to_database(df_metadata, df_media, df_colors):
    """Load DataFrames into SQL database"""
    
    # Create database connection
    connection_string = f"mysql+pymysql://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}@{os.getenv('DB_HOST')}:{os.getenv('DB_PORT')}/{os.getenv('DB_NAME')}"
    engine = create_engine(connection_string)
    
    try:
        # Load with replace strategy (or use 'append' for incremental)
        df_metadata.to_sql('artifactmetadata', engine, if_exists='replace', index=False)
        df_media.to_sql('artifactmedia', engine, if_exists='replace', index=False)
        df_colors.to_sql('artifactcolors', engine, if_exists='replace', index=False)
        
        print("Data loaded successfully!")
        return True
    except Exception as e:
        print(f"Error loading data: {e}")
        return False
    finally:
        engine.dispose()
```

### 4. SQL Analytics Queries

```python
def execute_sql_query(query):
    """Execute SQL query and return results as DataFrame"""
    connection_string = f"mysql+pymysql://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}@{os.getenv('DB_HOST')}:{os.getenv('DB_PORT')}/{os.getenv('DB_NAME')}"
    engine = create_engine(connection_string)
    
    try:
        df = pd.read_sql(query, engine)
        return df
    except Exception as e:
        print(f"Query error: {e}")
        return None
    finally:
        engine.dispose()

# Example analytical queries
SAMPLE_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Artifacts with Images": """
        SELECT 
            CASE WHEN primaryimageurl IS NOT NULL THEN 'With Image' ELSE 'No Image' END as image_status,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY image_status
    """,
    
    "Top Color Distribution": """
        SELECT color, COUNT(*) as frequency
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 10
    """,
    
    "Department Analysis": """
        SELECT department, COUNT(*) as total_artifacts,
               AVG(accessionyear) as avg_year
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY total_artifacts DESC
    """
}
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Artifacts Analytics Dashboard")
    
    # Sidebar configuration
    with st.sidebar:
        st.header("Configuration")
        api_key = st.text_input("Harvard API Key", type="password", value=os.getenv('HARVARD_API_KEY', ''))
        num_artifacts = st.number_input("Number of Artifacts", min_value=10, max_value=1000, value=100)
        
        if st.button("Run ETL Pipeline"):
            with st.spinner("Fetching data from API..."):
                artifacts = fetch_artifacts(api_key, num_artifacts)
            
            with st.spinner("Transforming data..."):
                df_metadata, df_media, df_colors = transform_artifact_data(artifacts)
            
            with st.spinner("Loading to database..."):
                success = load_to_database(df_metadata, df_media, df_colors)
            
            if success:
                st.success("ETL Pipeline completed!")
    
    # Analytics section
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox("Select Analysis", list(SAMPLE_QUERIES.keys()))
    
    if st.button("Run Query"):
        query = SAMPLE_QUERIES[query_name]
        
        with st.spinner("Executing query..."):
            df_result = execute_sql_query(query)
        
        if df_result is not None and not df_result.empty:
            st.subheader("Query Results")
            st.dataframe(df_result)
            
            # Auto-generate visualization
            if len(df_result.columns) >= 2:
                fig = px.bar(df_result, x=df_result.columns[0], y=df_result.columns[1],
                           title=query_name)
                st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Batch Processing for Large Datasets

```python
def batch_insert(df, table_name, batch_size=1000):
    """Insert data in batches for better performance"""
    connection_string = f"mysql+pymysql://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}@{os.getenv('DB_HOST')}:{os.getenv('DB_PORT')}/{os.getenv('DB_NAME')}"
    engine = create_engine(connection_string)
    
    total_rows = len(df)
    for i in range(0, total_rows, batch_size):
        batch = df.iloc[i:i+batch_size]
        batch.to_sql(table_name, engine, if_exists='append', index=False)
        print(f"Inserted {min(i+batch_size, total_rows)}/{total_rows} rows")
    
    engine.dispose()
```

### Error Handling and Retry Logic

```python
import time

def fetch_with_retry(api_key, max_retries=3):
    """Fetch data with retry logic for API failures"""
    for attempt in range(max_retries):
        try:
            artifacts = fetch_artifacts(api_key, 100)
            return artifacts
        except requests.exceptions.RequestException as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt  # Exponential backoff
                print(f"Retry {attempt + 1}/{max_retries} after {wait_time}s")
                time.sleep(wait_time)
            else:
                raise e
```

### Data Quality Validation

```python
def validate_data(df_metadata, df_media, df_colors):
    """Validate data quality before loading"""
    issues = []
    
    # Check for nulls in critical fields
    if df_metadata['id'].isnull().any():
        issues.append("Null IDs found in metadata")
    
    # Check referential integrity
    metadata_ids = set(df_metadata['id'])
    media_ids = set(df_media['artifact_id'])
    if not media_ids.issubset(metadata_ids):
        issues.append("Media references non-existent artifacts")
    
    # Check data types
    if df_metadata['accessionyear'].dtype != 'int64':
        issues.append("Accession year should be integer")
    
    return issues
```

## Troubleshooting

### API Rate Limiting

If you encounter rate limit errors:
```python
import time

def fetch_with_rate_limit(api_key, num_artifacts, delay=1):
    """Add delay between requests"""
    artifacts = []
    for page in range(1, (num_artifacts // 100) + 2):
        # Fetch page
        response = requests.get(base_url, params={'apikey': api_key, 'page': page})
        artifacts.extend(response.json().get('records', []))
        time.sleep(delay)  # Wait between requests
    return artifacts
```

### Database Connection Issues

```python
# Test database connection
def test_connection():
    try:
        connection_string = f"mysql+pymysql://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}@{os.getenv('DB_HOST')}:{os.getenv('DB_PORT')}/{os.getenv('DB_NAME')}"
        engine = create_engine(connection_string)
        connection = engine.connect()
        connection.close()
        print("✓ Database connection successful")
        return True
    except Exception as e:
        print(f"✗ Database connection failed: {e}")
        return False
```

### Memory Management for Large Datasets

```python
def process_in_chunks(artifacts, chunk_size=500):
    """Process large datasets in chunks"""
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i+chunk_size]
        df_metadata, df_media, df_colors = transform_artifact_data(chunk)
        load_to_database(df_metadata, df_media, df_colors)
        print(f"Processed chunk {i//chunk_size + 1}")
```

## Advanced Usage

### Custom SQL Analytics

```python
def create_custom_query(filters):
    """Build dynamic SQL queries based on user filters"""
    base_query = "SELECT * FROM artifactmetadata WHERE 1=1"
    
    if filters.get('culture'):
        base_query += f" AND culture = '{filters['culture']}'"
    if filters.get('century'):
        base_query += f" AND century = '{filters['century']}'"
    if filters.get('department'):
        base_query += f" AND department = '{filters['department']}'"
    
    return base_query
```

### Export Results

```python
def export_results(df, format='csv'):
    """Export query results to various formats"""
    if format == 'csv':
        df.to_csv('results.csv', index=False)
    elif format == 'excel':
        df.to_excel('results.xlsx', index=False)
    elif format == 'json':
        df.to_json('results.json', orient='records')
```

This skill provides everything needed to build production-ready ETL pipelines for museum data analytics with Harvard's API.
