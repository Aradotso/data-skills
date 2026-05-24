---
name: harvard-art-museums-data-pipeline
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build a data pipeline for museum artifacts
  - extract and analyze Harvard Art Museums data
  - create ETL pipeline with Streamlit dashboard
  - visualize art collection data with SQL analytics
  - process Harvard API data into relational database
  - build analytics app for museum collection
  - query and visualize artifact metadata
  - implement art museum data engineering workflow
---

# Harvard Art Museums Data Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides a complete data engineering and analytics solution for the Harvard Art Museums API. It demonstrates:

- **API Integration**: Fetching paginated artifact data from Harvard Art Museums
- **ETL Pipeline**: Extracting, transforming, and loading nested JSON into relational tables
- **SQL Storage**: Structured storage in MySQL/TiDB with proper foreign key relationships
- **Analytics Dashboard**: 20+ predefined SQL queries for insights
- **Interactive Visualization**: Streamlit-based UI with Plotly charts

The architecture follows: `API → ETL → SQL → Analytics → Visualization`

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly sqlalchemy
```

## Configuration

### Environment Variables

Create a `.env` file or set environment variables:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_db_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### API Key Setup

1. Get your free API key from: https://www.harvardartmuseums.org/collections/api
2. Store it securely in environment variables
3. Never commit API keys to version control

## Database Schema

The pipeline creates three relational tables:

```sql
-- Main artifact metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    dated VARCHAR(200),
    accessionyear INT,
    url TEXT
);

-- Media/image information
CREATE TABLE artifactmedia (
    media_id INT PRIMARY KEY AUTO_INCREMENT,
    artifact_id INT,
    baseimageurl TEXT,
    primaryimageurl TEXT,
    iiifbaseuri TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Color palette data
CREATE TABLE artifactcolors (
    color_id INT PRIMARY KEY AUTO_INCREMENT,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(100),
    hue VARCHAR(100),
    percent DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Usage Patterns

### 1. API Data Extraction

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

def collect_batch_data(api_key, num_records=1000):
    """Collect multiple pages of artifact data"""
    all_artifacts = []
    page = 1
    records_per_page = 100
    
    while len(all_artifacts) < num_records:
        data = fetch_artifacts(api_key, page=page, size=records_per_page)
        artifacts = data.get('records', [])
        
        if not artifacts:
            break
            
        all_artifacts.extend(artifacts)
        page += 1
    
    return all_artifacts[:num_records]
```

### 2. ETL Transformation

```python
import pandas as pd

def transform_to_dataframes(artifacts_json):
    """Transform nested JSON to relational dataframes"""
    
    # Extract metadata
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts_json:
        # Main metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown')[:500],
            'culture': artifact.get('culture', 'Unknown')[:200],
            'century': artifact.get('century', 'Unknown')[:100],
            'classification': artifact.get('classification', 'Unknown')[:200],
            'department': artifact.get('department', 'Unknown')[:200],
            'division': artifact.get('division', 'Unknown')[:200],
            'dated': artifact.get('dated', 'Unknown')[:200],
            'accessionyear': artifact.get('accessionyear'),
            'url': artifact.get('url', '')
        })
        
        # Media information
        media_records.append({
            'artifact_id': artifact.get('id'),
            'baseimageurl': artifact.get('baseimageurl', ''),
            'primaryimageurl': artifact.get('primaryimageurl', ''),
            'iiifbaseuri': artifact.get('iiifbaseuri', '')
        })
        
        # Color data (nested)
        colors = artifact.get('colors', [])
        for color in colors:
            color_records.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color', ''),
                'spectrum': color.get('spectrum', ''),
                'hue': color.get('hue', ''),
                'percent': color.get('percent', 0.0)
            })
    
    return {
        'metadata': pd.DataFrame(metadata_records),
        'media': pd.DataFrame(media_records),
        'colors': pd.DataFrame(color_records)
    }
```

### 3. Database Loading

```python
import mysql.connector
from sqlalchemy import create_engine

def create_db_connection(host, user, password, database):
    """Create MySQL connection using SQLAlchemy"""
    connection_string = f"mysql+mysqlconnector://{user}:{password}@{host}/{database}"
    engine = create_engine(connection_string)
    return engine

def load_to_database(dataframes, engine):
    """Load transformed data to SQL database"""
    
    # Load metadata first (parent table)
    dataframes['metadata'].to_sql(
        'artifactmetadata', 
        con=engine, 
        if_exists='append', 
        index=False,
        method='multi'
    )
    
    # Load media (child table)
    dataframes['media'].to_sql(
        'artifactmedia', 
        con=engine, 
        if_exists='append', 
        index=False,
        method='multi'
    )
    
    # Load colors (child table)
    if not dataframes['colors'].empty:
        dataframes['colors'].to_sql(
            'artifactcolors', 
            con=engine, 
            if_exists='append', 
            index=False,
            method='multi'
        )
    
    print(f"Loaded {len(dataframes['metadata'])} artifacts successfully")
```

### 4. SQL Analytics Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "Top 10 Cultures by Artifact Count": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture != 'Unknown'
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century Distribution": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Department-wise Classification": """
        SELECT department, classification, COUNT(*) as count
        FROM artifactmetadata
        GROUP BY department, classification
        ORDER BY department, count DESC
    """,
    
    "Color Distribution Across Artifacts": """
        SELECT color, COUNT(DISTINCT artifact_id) as artifact_count,
               AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "Artifacts with Complete Media": """
        SELECT am.title, am.culture, am.century,
               amedia.primaryimageurl
        FROM artifactmetadata am
        JOIN artifactmedia amedia ON am.id = amedia.artifact_id
        WHERE amedia.primaryimageurl IS NOT NULL 
          AND amedia.primaryimageurl != ''
        LIMIT 20
    """,
    
    "Accession Year Trends": """
        SELECT accessionyear, COUNT(*) as count
        FROM artifactmetadata
        WHERE accessionyear IS NOT NULL
        GROUP BY accessionyear
        ORDER BY accessionyear DESC
    """
}

def execute_query(engine, query):
    """Execute SQL query and return DataFrame"""
    return pd.read_sql(query, con=engine)
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Art Analytics", layout="wide")
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    with st.sidebar:
        st.header("Configuration")
        api_key = st.text_input("Harvard API Key", type="password", 
                                value=os.getenv("HARVARD_API_KEY", ""))
        
        st.header("Database Connection")
        db_host = st.text_input("DB Host", value=os.getenv("DB_HOST", ""))
        db_user = st.text_input("DB User", value=os.getenv("DB_USER", ""))
        db_password = st.text_input("DB Password", type="password", 
                                   value=os.getenv("DB_PASSWORD", ""))
        db_name = st.text_input("DB Name", value=os.getenv("DB_NAME", ""))
    
    # ETL Section
    st.header("📥 Data Collection & ETL")
    col1, col2 = st.columns(2)
    
    with col1:
        num_records = st.number_input("Records to Fetch", 100, 5000, 500)
        if st.button("Start ETL Pipeline"):
            with st.spinner("Fetching data from API..."):
                artifacts = collect_batch_data(api_key, num_records)
                st.success(f"Fetched {len(artifacts)} artifacts")
            
            with st.spinner("Transforming data..."):
                dfs = transform_to_dataframes(artifacts)
                st.success("Data transformed successfully")
            
            with st.spinner("Loading to database..."):
                engine = create_db_connection(db_host, db_user, db_password, db_name)
                load_to_database(dfs, engine)
                st.success("ETL Pipeline completed!")
    
    # Analytics Section
    st.header("📊 SQL Analytics")
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Execute Query"):
        engine = create_db_connection(db_host, db_user, db_password, db_name)
        query = ANALYTICS_QUERIES[query_name]
        
        with st.spinner("Running query..."):
            results = execute_query(engine, query)
            
        st.dataframe(results)
        
        # Auto-visualization
        if len(results.columns) >= 2:
            fig = px.bar(results, x=results.columns[0], y=results.columns[1],
                        title=query_name)
            st.plotly_chart(fig, use_container_width=True)

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

### Batch Processing with Rate Limiting

```python
import time

def fetch_with_rate_limit(api_key, pages, delay=0.5):
    """Fetch multiple pages with rate limiting"""
    all_data = []
    
    for page in range(1, pages + 1):
        data = fetch_artifacts(api_key, page=page)
        all_data.extend(data.get('records', []))
        time.sleep(delay)  # Respect API rate limits
    
    return all_data
```

### Error Handling

```python
def safe_etl_pipeline(api_key, db_config, num_records):
    """ETL pipeline with error handling"""
    try:
        # Extract
        artifacts = collect_batch_data(api_key, num_records)
        if not artifacts:
            raise ValueError("No artifacts fetched")
        
        # Transform
        dfs = transform_to_dataframes(artifacts)
        
        # Load
        engine = create_db_connection(**db_config)
        load_to_database(dfs, engine)
        
        return True, "Pipeline completed successfully"
    
    except requests.exceptions.RequestException as e:
        return False, f"API Error: {str(e)}"
    except Exception as e:
        return False, f"Pipeline Error: {str(e)}"
```

## Troubleshooting

### API Issues
- **Invalid API Key**: Ensure your Harvard API key is valid and not expired
- **Rate Limiting**: Add delays between requests (0.5-1 second recommended)
- **Empty Results**: Check API parameters and endpoint availability

### Database Issues
- **Connection Failed**: Verify DB credentials and network connectivity
- **Foreign Key Errors**: Ensure parent records exist before inserting child records
- **Duplicate Keys**: Use `INSERT IGNORE` or check for existing records

### Streamlit Issues
- **Port Already in Use**: Change port with `streamlit run app.py --server.port 8502`
- **Environment Variables Not Loading**: Use `python-dotenv` or set variables explicitly

## Advanced Usage

### Custom Query Builder

```python
def build_dynamic_query(table, columns, filters=None, limit=None):
    """Build SQL query dynamically"""
    query = f"SELECT {', '.join(columns)} FROM {table}"
    
    if filters:
        where_clauses = [f"{k} = '{v}'" for k, v in filters.items()]
        query += " WHERE " + " AND ".join(where_clauses)
    
    if limit:
        query += f" LIMIT {limit}"
    
    return query
```

### Data Quality Checks

```python
def validate_data_quality(df):
    """Check data quality before loading"""
    issues = []
    
    # Check for nulls in required fields
    if df['id'].isnull().any():
        issues.append("Missing artifact IDs")
    
    # Check for duplicates
    if df['id'].duplicated().any():
        issues.append("Duplicate artifact IDs found")
    
    # Check data types
    if df['accessionyear'].dtype not in ['int64', 'Int64']:
        issues.append("Invalid accessionyear data type")
    
    return len(issues) == 0, issues
```

This skill enables AI agents to build complete data engineering pipelines for museum artifact data with proper ETL, SQL analytics, and interactive visualization.
