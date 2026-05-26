---
name: harvard-artifacts-collection-data-engineering-analytics
description: End-to-end data engineering and analytics application using Harvard Art Museums API with ETL pipelines, SQL analytics, and Streamlit visualization
triggers:
  - build a data pipeline for museum artifacts
  - create an ETL process with Harvard Art Museums API
  - set up artifact collection analytics dashboard
  - implement museum data engineering workflow
  - analyze Harvard art museum collections with SQL
  - visualize artifact metadata using Streamlit
  - extract and transform museum API data
  - build analytics app for art collections
---

# Harvard Artifacts Collection Data Engineering Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is a complete data engineering and analytics application that demonstrates real-world ETL (Extract, Transform, Load) pipelines. It collects artifact data from the Harvard Art Museums API, processes it into relational database tables, performs SQL analytics, and visualizes results through an interactive Streamlit dashboard.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

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

Set up your environment variables for secure configuration:

```bash
# .env file
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_PORT=4000
DB_USER=your_db_username
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Setup

The application uses three main tables:

```sql
-- Artifact Metadata Table
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
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(1000),
    primaryimageurl VARCHAR(1000),
    imagepermissionlevel INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Key Components

### 1. API Data Collection

```python
import requests
import pandas as pd
import os

def fetch_artifacts(api_key, num_records=100):
    """
    Fetch artifact data from Harvard Art Museums API
    """
    base_url = "https://api.harvardartmuseums.org/object"
    artifacts = []
    page = 1
    size = 100  # API pagination size
    
    while len(artifacts) < num_records:
        params = {
            'apikey': api_key,
            'size': size,
            'page': page
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            
            if not records:
                break
                
            artifacts.extend(records)
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_records]

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts = fetch_artifacts(api_key, num_records=500)
```

### 2. ETL Pipeline

```python
import pandas as pd
from sqlalchemy import create_engine

def transform_artifacts(artifacts):
    """
    Transform raw API data into structured dataframes
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:200],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:200],
            'department': artifact.get('department', '')[:200],
            'division': artifact.get('division', '')[:200],
            'dated': artifact.get('dated', '')[:200],
            'accessionyear': artifact.get('accessionyear'),
            'technique': artifact.get('technique', '')[:500],
            'medium': artifact.get('medium', '')[:500],
            'dimensions': artifact.get('dimensions', '')[:500]
        }
        metadata_list.append(metadata)
        
        # Extract media
        media = {
            'artifact_id': artifact.get('id'),
            'baseimageurl': artifact.get('baseimageurl', '')[:1000],
            'primaryimageurl': artifact.get('primaryimageurl', '')[:1000],
            'imagepermissionlevel': artifact.get('imagepermissionlevel', 0)
        }
        media_list.append(media)
        
        # Extract colors
        colors = artifact.get('colors', [])
        for color in colors:
            color_data = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color', '')[:50],
                'spectrum': color.get('spectrum', '')[:50],
                'hue': color.get('hue', '')[:50],
                'percent': color.get('percent', 0.0)
            }
            colors_list.append(color_data)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )

def load_to_database(df_metadata, df_media, df_colors, connection_string):
    """
    Load dataframes into SQL database
    """
    engine = create_engine(connection_string)
    
    # Load with batch inserts
    df_metadata.to_sql('artifactmetadata', engine, 
                       if_exists='append', index=False, method='multi')
    df_media.to_sql('artifactmedia', engine, 
                    if_exists='append', index=False, method='multi')
    df_colors.to_sql('artifactcolors', engine, 
                     if_exists='append', index=False, method='multi')
    
    print(f"Loaded {len(df_metadata)} metadata records")
    print(f"Loaded {len(df_media)} media records")
    print(f"Loaded {len(df_colors)} color records")

# Usage
df_metadata, df_media, df_colors = transform_artifacts(artifacts)

connection_string = f"mysql+mysqlconnector://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}@{os.getenv('DB_HOST')}:{os.getenv('DB_PORT')}/{os.getenv('DB_NAME')}"

load_to_database(df_metadata, df_media, df_colors, connection_string)
```

### 3. SQL Analytics Queries

```python
import mysql.connector

def execute_query(query, db_config):
    """
    Execute SQL query and return results as dataframe
    """
    conn = mysql.connector.connect(
        host=db_config['host'],
        port=db_config['port'],
        user=db_config['user'],
        password=db_config['password'],
        database=db_config['database']
    )
    
    df = pd.read_sql(query, conn)
    conn.close()
    
    return df

# Sample analytical queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Media Availability": """
        SELECT 
            CASE 
                WHEN primaryimageurl IS NOT NULL AND primaryimageurl != '' THEN 'Has Image'
                ELSE 'No Image'
            END as image_status,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY image_status
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Accession Year": """
        SELECT accessionyear, COUNT(*) as count
        FROM artifactmetadata
        WHERE accessionyear IS NOT NULL
        GROUP BY accessionyear
        ORDER BY accessionyear DESC
        LIMIT 20
    """,
    
    "Classification Analysis": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 15
    """
}
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    st.markdown("End-to-end Data Engineering & Analytics Application")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    # Database connection
    db_config = {
        'host': os.getenv('DB_HOST'),
        'port': int(os.getenv('DB_PORT', 4000)),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    
    # Query selection
    query_name = st.sidebar.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.sidebar.button("Run Analysis"):
        with st.spinner("Executing query..."):
            query = ANALYTICS_QUERIES[query_name]
            df_result = execute_query(query, db_config)
            
            st.subheader(f"📊 {query_name}")
            st.dataframe(df_result)
            
            # Auto-generate visualization
            if len(df_result.columns) >= 2:
                fig = px.bar(
                    df_result,
                    x=df_result.columns[0],
                    y=df_result.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig, use_container_width=True)
    
    # ETL Section
    st.sidebar.header("ETL Pipeline")
    
    num_records = st.sidebar.number_input(
        "Number of records to fetch",
        min_value=10,
        max_value=1000,
        value=100,
        step=10
    )
    
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner(f"Fetching {num_records} records..."):
            api_key = os.getenv('HARVARD_API_KEY')
            artifacts = fetch_artifacts(api_key, num_records)
            
            df_metadata, df_media, df_colors = transform_artifacts(artifacts)
            
            connection_string = f"mysql+mysqlconnector://{db_config['user']}:{db_config['password']}@{db_config['host']}:{db_config['port']}/{db_config['database']}"
            
            load_to_database(df_metadata, df_media, df_colors, connection_string)
            
            st.success(f"✅ Successfully loaded {num_records} artifacts!")

if __name__ == "__main__":
    main()
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(api_key, delay=1):
    """Fetch data with rate limiting"""
    artifacts = []
    
    for page in range(1, 11):  # Fetch 10 pages
        response = requests.get(
            "https://api.harvardartmuseums.org/object",
            params={'apikey': api_key, 'page': page, 'size': 100}
        )
        
        if response.status_code == 200:
            artifacts.extend(response.json()['records'])
        
        time.sleep(delay)  # Respect API rate limits
    
    return artifacts
```

### Incremental Data Loading

```python
def load_incremental(df, table_name, engine):
    """Load only new records not already in database"""
    existing_ids = pd.read_sql(
        f"SELECT id FROM {table_name}",
        engine
    )['id'].tolist()
    
    new_records = df[~df['id'].isin(existing_ids)]
    
    if len(new_records) > 0:
        new_records.to_sql(table_name, engine, 
                           if_exists='append', index=False)
        print(f"Loaded {len(new_records)} new records")
    else:
        print("No new records to load")
```

## Troubleshooting

### API Key Issues

```python
# Verify API key is loaded
import os
api_key = os.getenv('HARVARD_API_KEY')
if not api_key:
    raise ValueError("HARVARD_API_KEY not found in environment variables")

# Test API connection
response = requests.get(
    "https://api.harvardartmuseums.org/object",
    params={'apikey': api_key, 'size': 1}
)
print(f"API Status: {response.status_code}")
```

### Database Connection Issues

```python
# Test database connectivity
try:
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT')),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    print("✅ Database connection successful")
    conn.close()
except Exception as e:
    print(f"❌ Database connection failed: {e}")
```

### Data Type Mismatches

```python
# Handle null values and data types
df_metadata['accessionyear'] = pd.to_numeric(
    df_metadata['accessionyear'], 
    errors='coerce'
).astype('Int64')  # Nullable integer

df_colors['percent'] = pd.to_numeric(
    df_colors['percent'], 
    errors='coerce'
).fillna(0.0)
```

### Memory Management for Large Datasets

```python
# Process in chunks for large datasets
def load_large_dataset(artifacts, chunk_size=1000):
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i+chunk_size]
        df_metadata, df_media, df_colors = transform_artifacts(chunk)
        load_to_database(df_metadata, df_media, df_colors, connection_string)
        print(f"Processed chunk {i//chunk_size + 1}")
```

This skill enables AI coding agents to effectively work with the Harvard Artifacts Collection Data Engineering Analytics application, covering the complete ETL pipeline, SQL analytics, and visualization workflow.
