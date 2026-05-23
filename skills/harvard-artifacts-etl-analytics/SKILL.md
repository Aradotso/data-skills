---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with SQL and Streamlit
triggers:
  - how do I extract data from Harvard Art Museums API
  - build an ETL pipeline for museum artifact data
  - create analytics dashboard with Streamlit and SQL
  - how to transform Harvard API JSON to relational tables
  - query and visualize museum collection data
  - set up data engineering pipeline for artifacts
  - analyze art museum collections with Python
  - connect Harvard Art Museums API to SQL database
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Harvard Artifacts Collection Data Engineering Analytics App is an end-to-end data pipeline that demonstrates professional ETL processes, SQL analytics, and interactive visualization. It extracts artifact data from the Harvard Art Museums API, transforms nested JSON into relational database tables, loads into MySQL/TiDB Cloud, and provides 20+ analytical queries with Plotly visualizations through a Streamlit interface.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

## Configuration

### API Key Setup

1. Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api
2. Store in environment variable or Streamlit secrets:

```python
# .env file
HARVARD_API_KEY=your_api_key_here

# Or use Streamlit secrets (.streamlit/secrets.toml)
[api]
harvard_key = "your_api_key_here"

[database]
host = "your_db_host"
port = 4000
user = "your_db_user"
password = "your_db_password"
database = "harvard_artifacts"
```

### Database Setup

```sql
-- Create database
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    technique VARCHAR(300),
    medium VARCHAR(300),
    dated VARCHAR(200),
    period VARCHAR(200),
    url TEXT,
    creditline TEXT,
    description TEXT
);

-- Media/images table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    imagetotal INT,
    videototal INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Color analysis table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    hex_code VARCHAR(10),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core API Integration

### Fetching Artifacts with Pagination

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, max_records=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    page = 1
    size = 100  # Max per page
    
    while len(all_artifacts) < max_records:
        params = {
            'apikey': api_key,
            'size': size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            
            if not records:
                break
                
            all_artifacts.extend(records)
            page += 1
            
            # Rate limiting
            import time
            time.sleep(0.5)
        else:
            print(f"Error: {response.status_code}")
            break
    
    return all_artifacts[:max_records]

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts = fetch_artifacts(api_key, max_records=500)
print(f"Fetched {len(artifacts)} artifacts")
```

## ETL Pipeline

### Extract and Transform

```python
import pandas as pd

def extract_metadata(artifacts):
    """
    Extract and flatten artifact metadata
    """
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'division': artifact.get('division', 'Unknown'),
            'technique': artifact.get('technique', 'Unknown'),
            'medium': artifact.get('medium', 'Unknown'),
            'dated': artifact.get('dated', 'Unknown'),
            'period': artifact.get('period', 'Unknown'),
            'url': artifact.get('url', ''),
            'creditline': artifact.get('creditline', ''),
            'description': artifact.get('description', '')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def extract_media(artifacts):
    """
    Extract media/image information
    """
    media_list = []
    
    for artifact in artifacts:
        media = {
            'artifact_id': artifact.get('id'),
            'baseimageurl': artifact.get('baseimageurl', ''),
            'primaryimageurl': artifact.get('primaryimageurl', ''),
            'imagetotal': artifact.get('totalpageviews', 0),
            'videototal': artifact.get('totalvideoviews', 0)
        }
        media_list.append(media)
    
    return pd.DataFrame(media_list)

def extract_colors(artifacts):
    """
    Extract color data from artifacts
    """
    colors_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_record = {
                'artifact_id': artifact_id,
                'color': color.get('color', 'Unknown'),
                'hex_code': color.get('hex', '#000000'),
                'percentage': color.get('percent', 0.0)
            }
            colors_list.append(color_record)
    
    return pd.DataFrame(colors_list)

# Execute ETL
metadata_df = extract_metadata(artifacts)
media_df = extract_media(artifacts)
colors_df = extract_colors(artifacts)
```

### Load to SQL Database

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(df, table_name, db_config):
    """
    Batch insert DataFrame into SQL table
    """
    try:
        connection = mysql.connector.connect(
            host=db_config['host'],
            port=db_config['port'],
            user=db_config['user'],
            password=db_config['password'],
            database=db_config['database']
        )
        
        if connection.is_connected():
            cursor = connection.cursor()
            
            # Prepare columns and placeholders
            cols = ','.join(df.columns)
            placeholders = ','.join(['%s'] * len(df.columns))
            sql = f"INSERT INTO {table_name} ({cols}) VALUES ({placeholders})"
            
            # Batch insert
            data = [tuple(row) for row in df.values]
            cursor.executemany(sql, data)
            connection.commit()
            
            print(f"Inserted {cursor.rowcount} rows into {table_name}")
            
            cursor.close()
            connection.close()
            
    except Error as e:
        print(f"Database error: {e}")

# Database configuration
db_config = {
    'host': os.getenv('DB_HOST'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': 'harvard_artifacts'
}

# Load data
load_to_database(metadata_df, 'artifactmetadata', db_config)
load_to_database(media_df, 'artifactmedia', db_config)
load_to_database(colors_df, 'artifactcolors', db_config)
```

## Streamlit Application

### Main App Structure

```python
import streamlit as st
import pandas as pd
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

def main():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar navigation
    menu = st.sidebar.selectbox(
        "Navigation",
        ["Data Collection", "Analytics Dashboard", "Visualizations"]
    )
    
    if menu == "Data Collection":
        data_collection_page()
    elif menu == "Analytics Dashboard":
        analytics_page()
    elif menu == "Visualizations":
        visualization_page()

def data_collection_page():
    st.header("📥 Data Collection")
    
    api_key = st.text_input("Enter Harvard API Key", type="password")
    num_records = st.number_input("Number of Records", min_value=10, max_value=1000, value=100)
    
    if st.button("Fetch Data"):
        with st.spinner("Fetching artifacts..."):
            artifacts = fetch_artifacts(api_key, num_records)
            
            metadata_df = extract_metadata(artifacts)
            st.success(f"Fetched {len(metadata_df)} artifacts")
            st.dataframe(metadata_df.head(10))
            
            # Option to load to database
            if st.button("Load to Database"):
                load_to_database(metadata_df, 'artifactmetadata', db_config)
                st.success("Data loaded successfully!")

if __name__ == "__main__":
    main()
```

## SQL Analytics Queries

### Common Analytical Queries

```python
def run_analytics_query(query_name, connection):
    """
    Execute predefined analytical queries
    """
    queries = {
        'artifacts_by_culture': """
            SELECT culture, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE culture != 'Unknown'
            GROUP BY culture
            ORDER BY artifact_count DESC
            LIMIT 10
        """,
        
        'artifacts_by_century': """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century != 'Unknown'
            GROUP BY century
            ORDER BY count DESC
        """,
        
        'media_availability': """
            SELECT 
                CASE 
                    WHEN imagetotal > 0 THEN 'Has Images'
                    ELSE 'No Images'
                END as image_status,
                COUNT(*) as count
            FROM artifactmedia
            GROUP BY image_status
        """,
        
        'top_colors': """
            SELECT color, COUNT(*) as usage_count
            FROM artifactcolors
            GROUP BY color
            ORDER BY usage_count DESC
            LIMIT 10
        """,
        
        'classification_distribution': """
            SELECT classification, COUNT(*) as count
            FROM artifactmetadata
            WHERE classification != 'Unknown'
            GROUP BY classification
            ORDER BY count DESC
            LIMIT 15
        """,
        
        'department_analysis': """
            SELECT department, 
                   COUNT(*) as total_artifacts,
                   COUNT(DISTINCT culture) as cultures_represented
            FROM artifactmetadata
            WHERE department != 'Unknown'
            GROUP BY department
            ORDER BY total_artifacts DESC
        """
    }
    
    cursor = connection.cursor(dictionary=True)
    cursor.execute(queries[query_name])
    results = cursor.fetchall()
    cursor.close()
    
    return pd.DataFrame(results)
```

### Analytics Dashboard Page

```python
def analytics_page():
    st.header("📊 SQL Analytics Dashboard")
    
    # Connect to database
    connection = mysql.connector.connect(**db_config)
    
    query_options = [
        "Artifacts by Culture",
        "Artifacts by Century",
        "Media Availability",
        "Top Colors",
        "Classification Distribution",
        "Department Analysis"
    ]
    
    selected_query = st.selectbox("Select Analysis", query_options)
    
    # Map selection to query name
    query_map = {
        "Artifacts by Culture": "artifacts_by_culture",
        "Artifacts by Century": "artifacts_by_century",
        "Media Availability": "media_availability",
        "Top Colors": "top_colors",
        "Classification Distribution": "classification_distribution",
        "Department Analysis": "department_analysis"
    }
    
    if st.button("Run Query"):
        query_name = query_map[selected_query]
        df = run_analytics_query(query_name, connection)
        
        st.subheader("Query Results")
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)
    
    connection.close()
```

## Visualization Patterns

### Creating Interactive Charts

```python
import plotly.express as px
import plotly.graph_objects as go

def create_culture_chart(df):
    """
    Create horizontal bar chart for culture distribution
    """
    fig = px.bar(df, x='artifact_count', y='culture',
                 orientation='h',
                 title='Top 10 Cultures by Artifact Count',
                 labels={'artifact_count': 'Number of Artifacts', 'culture': 'Culture'})
    fig.update_layout(height=500)
    return fig

def create_color_pie_chart(df):
    """
    Create pie chart for color distribution
    """
    fig = px.pie(df, values='usage_count', names='color',
                 title='Color Distribution in Artifacts',
                 hole=0.3)
    return fig

def create_timeline_chart(df):
    """
    Create timeline visualization for artifacts by century
    """
    fig = px.line(df, x='century', y='count',
                  title='Artifact Timeline by Century',
                  markers=True)
    fig.update_layout(xaxis_title='Century', yaxis_title='Artifact Count')
    return fig

# Usage in Streamlit
def visualization_page():
    st.header("📈 Data Visualizations")
    
    connection = mysql.connector.connect(**db_config)
    
    viz_type = st.selectbox("Visualization Type", 
                            ["Culture Distribution", "Color Analysis", "Timeline"])
    
    if viz_type == "Culture Distribution":
        df = run_analytics_query('artifacts_by_culture', connection)
        fig = create_culture_chart(df)
        st.plotly_chart(fig, use_container_width=True)
    
    connection.close()
```

## Troubleshooting

### Common Issues

**API Rate Limiting:**
```python
# Add delays between requests
import time
time.sleep(0.5)  # 500ms delay

# Or use exponential backoff
import requests
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

session = requests.Session()
retry = Retry(total=3, backoff_factor=1)
adapter = HTTPAdapter(max_retries=retry)
session.mount('https://', adapter)
```

**Database Connection Errors:**
```python
# Test connection
try:
    connection = mysql.connector.connect(**db_config)
    if connection.is_connected():
        print("Connected to database")
        connection.close()
except Error as e:
    print(f"Error: {e}")
    # Check: host, port, credentials, firewall rules
```

**Missing Data Handling:**
```python
# Handle None values in DataFrames
df = df.fillna({
    'culture': 'Unknown',
    'century': 'Unknown',
    'classification': 'Unknown'
})

# Or drop rows with missing critical data
df = df.dropna(subset=['id', 'title'])
```

**Streamlit Secrets Access:**
```python
# Access secrets safely
try:
    api_key = st.secrets["api"]["harvard_key"]
    db_config = {
        'host': st.secrets["database"]["host"],
        'user': st.secrets["database"]["user"],
        'password': st.secrets["database"]["password"]
    }
except KeyError as e:
    st.error(f"Missing secret: {e}")
```

## Running the Application

```bash
# Local development
streamlit run app.py

# With custom port
streamlit run app.py --server.port 8501

# With environment variables
export HARVARD_API_KEY=your_key
export DB_HOST=your_host
streamlit run app.py
```
