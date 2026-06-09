---
name: harvard-artifacts-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, Streamlit, and SQL
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - extract Harvard museum API data to SQL database
  - set up artifact collection data pipeline
  - visualize Harvard art museum data with Streamlit
  - query and analyze museum artifact metadata
  - implement museum data engineering workflow
  - transform Harvard API JSON to relational tables
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build complete data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for cultural artifact data.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Collects artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads nested JSON data into relational SQL tables
- **Database Design**: Structures data into normalized tables (metadata, media, colors) with foreign keys
- **SQL Analytics**: Provides 20+ predefined analytical queries for insights
- **Interactive Dashboard**: Streamlit-based UI with Plotly visualizations

**Architecture Flow**: API → ETL → SQL Database → Analytics Queries → Streamlit Dashboard → Plotly Visualizations

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

### Get Harvard API Key

1. Visit https://docs.harvardartmuseums.org/
2. Register for a free API key
3. Add to `.env` file

### Database Setup

The application supports MySQL or TiDB Cloud. Create the database:

```sql
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;
```

## Key Components

### 1. API Data Collection

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

class HarvardAPIClient:
    def __init__(self):
        self.api_key = os.getenv('HARVARD_API_KEY')
        self.base_url = "https://api.harvardartmuseums.org/object"
    
    def fetch_artifacts(self, page=1, size=100):
        """Fetch artifacts with pagination"""
        params = {
            'apikey': self.api_key,
            'page': page,
            'size': size
        }
        response = requests.get(self.base_url, params=params)
        response.raise_for_status()
        return response.json()
    
    def fetch_all_artifacts(self, total_records=1000):
        """Fetch multiple pages of artifacts"""
        all_artifacts = []
        page = 1
        size = 100
        
        while len(all_artifacts) < total_records:
            data = self.fetch_artifacts(page=page, size=size)
            records = data.get('records', [])
            if not records:
                break
            all_artifacts.extend(records)
            page += 1
        
        return all_artifacts[:total_records]
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

class ArtifactETL:
    def __init__(self):
        self.connection = self._create_connection()
    
    def _create_connection(self):
        """Create database connection"""
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
            print(f"Error connecting to database: {e}")
            return None
    
    def create_tables(self):
        """Create normalized tables"""
        cursor = self.connection.cursor()
        
        # Metadata table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactmetadata (
                objectid INT PRIMARY KEY,
                title VARCHAR(500),
                culture VARCHAR(200),
                century VARCHAR(100),
                classification VARCHAR(200),
                department VARCHAR(200),
                division VARCHAR(200),
                dated VARCHAR(200),
                period VARCHAR(200),
                technique VARCHAR(500),
                medium VARCHAR(500),
                dimensions VARCHAR(500),
                creditline TEXT,
                accessionyear INT,
                provenance TEXT,
                url VARCHAR(500)
            )
        """)
        
        # Media table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactmedia (
                mediaid INT AUTO_INCREMENT PRIMARY KEY,
                objectid INT,
                iiifbaseuri VARCHAR(500),
                baseimageurl VARCHAR(500),
                primaryimageurl VARCHAR(500),
                imagetype VARCHAR(100),
                FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
            )
        """)
        
        # Colors table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactcolors (
                colorid INT AUTO_INCREMENT PRIMARY KEY,
                objectid INT,
                color VARCHAR(50),
                spectrum VARCHAR(50),
                hue VARCHAR(50),
                percent FLOAT,
                FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
            )
        """)
        
        self.connection.commit()
        cursor.close()
    
    def transform_metadata(self, artifacts):
        """Transform JSON to metadata DataFrame"""
        metadata_records = []
        
        for artifact in artifacts:
            record = {
                'objectid': artifact.get('objectid'),
                'title': artifact.get('title'),
                'culture': artifact.get('culture'),
                'century': artifact.get('century'),
                'classification': artifact.get('classification'),
                'department': artifact.get('department'),
                'division': artifact.get('division'),
                'dated': artifact.get('dated'),
                'period': artifact.get('period'),
                'technique': artifact.get('technique'),
                'medium': artifact.get('medium'),
                'dimensions': artifact.get('dimensions'),
                'creditline': artifact.get('creditline'),
                'accessionyear': artifact.get('accessionyear'),
                'provenance': artifact.get('provenance'),
                'url': artifact.get('url')
            }
            metadata_records.append(record)
        
        return pd.DataFrame(metadata_records)
    
    def transform_media(self, artifacts):
        """Transform JSON to media DataFrame"""
        media_records = []
        
        for artifact in artifacts:
            objectid = artifact.get('objectid')
            images = artifact.get('images', [])
            
            if images:
                for img in images:
                    media_records.append({
                        'objectid': objectid,
                        'iiifbaseuri': img.get('iiifbaseuri'),
                        'baseimageurl': img.get('baseimageurl'),
                        'primaryimageurl': artifact.get('primaryimageurl'),
                        'imagetype': 'primary' if img.get('baseimageurl') == artifact.get('primaryimageurl') else 'additional'
                    })
            else:
                media_records.append({
                    'objectid': objectid,
                    'iiifbaseuri': None,
                    'baseimageurl': None,
                    'primaryimageurl': artifact.get('primaryimageurl'),
                    'imagetype': None
                })
        
        return pd.DataFrame(media_records)
    
    def transform_colors(self, artifacts):
        """Transform JSON to colors DataFrame"""
        color_records = []
        
        for artifact in artifacts:
            objectid = artifact.get('objectid')
            colors = artifact.get('colors', [])
            
            for color in colors:
                color_records.append({
                    'objectid': objectid,
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'hue': color.get('hue'),
                    'percent': color.get('percent')
                })
        
        return pd.DataFrame(color_records)
    
    def load_data(self, df, table_name):
        """Batch insert data into SQL table"""
        if df.empty:
            return
        
        cursor = self.connection.cursor()
        cols = ",".join(df.columns)
        placeholders = ",".join(["%s"] * len(df.columns))
        
        insert_query = f"INSERT IGNORE INTO {table_name} ({cols}) VALUES ({placeholders})"
        
        data_tuples = [tuple(x) for x in df.to_numpy()]
        cursor.executemany(insert_query, data_tuples)
        
        self.connection.commit()
        cursor.close()
```

### 3. Streamlit Application

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🏛️",
        layout="wide"
    )
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar navigation
    menu = st.sidebar.selectbox(
        "Navigation",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if menu == "Data Collection":
        show_data_collection()
    elif menu == "SQL Analytics":
        show_sql_analytics()
    else:
        show_visualizations()

def show_data_collection():
    st.header("📥 Data Collection")
    
    num_records = st.number_input("Number of records to fetch", 
                                   min_value=10, max_value=5000, value=100)
    
    if st.button("Fetch and Load Data"):
        with st.spinner("Fetching data from Harvard API..."):
            client = HarvardAPIClient()
            artifacts = client.fetch_all_artifacts(total_records=num_records)
            
            etl = ArtifactETL()
            etl.create_tables()
            
            # Transform and load
            metadata_df = etl.transform_metadata(artifacts)
            etl.load_data(metadata_df, 'artifactmetadata')
            
            media_df = etl.transform_media(artifacts)
            etl.load_data(media_df, 'artifactmedia')
            
            colors_df = etl.transform_colors(artifacts)
            etl.load_data(colors_df, 'artifactcolors')
            
            st.success(f"✅ Successfully loaded {len(artifacts)} artifacts!")

def show_sql_analytics():
    st.header("📊 SQL Analytics")
    
    queries = {
        "Artifacts by Culture": """
            SELECT culture, COUNT(*) as count 
            FROM artifactmetadata 
            WHERE culture IS NOT NULL 
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
        "Most Common Colors": """
            SELECT color, COUNT(*) as frequency 
            FROM artifactcolors 
            WHERE color IS NOT NULL 
            GROUP BY color 
            ORDER BY frequency DESC 
            LIMIT 15
        """,
        "Artifacts with Images": """
            SELECT 
                CASE WHEN primaryimageurl IS NOT NULL THEN 'With Image' ELSE 'No Image' END as status,
                COUNT(*) as count
            FROM artifactmetadata
            GROUP BY status
        """
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        etl = ArtifactETL()
        cursor = etl.connection.cursor()
        cursor.execute(queries[selected_query])
        
        results = cursor.fetchall()
        columns = [desc[0] for desc in cursor.description]
        
        df = pd.DataFrame(results, columns=columns)
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Pattern 1: Incremental Data Loading

```python
def incremental_load(self, last_update_date=None):
    """Load only new artifacts since last update"""
    if last_update_date:
        params = {
            'apikey': self.api_key,
            'lastupdate': last_update_date
        }
    else:
        params = {'apikey': self.api_key}
    
    response = requests.get(self.base_url, params=params)
    return response.json()
```

### Pattern 2: Error Handling in ETL

```python
def safe_load_data(self, df, table_name):
    """Load data with error handling"""
    try:
        self.load_data(df, table_name)
        return True, f"Successfully loaded {len(df)} records"
    except Error as e:
        return False, f"Error loading data: {str(e)}"
```

### Pattern 3: Data Quality Checks

```python
def validate_artifacts(self, artifacts):
    """Validate artifact data before loading"""
    valid_artifacts = []
    
    for artifact in artifacts:
        if artifact.get('objectid') and artifact.get('title'):
            valid_artifacts.append(artifact)
    
    return valid_artifacts
```

## Running the Application

```bash
# Run Streamlit app
streamlit run app.py

# Run with specific port
streamlit run app.py --server.port 8501

# Run with custom config
streamlit run app.py --server.headless true
```

## Troubleshooting

**Issue**: API rate limit exceeded
```python
import time

def fetch_with_retry(self, page, max_retries=3):
    for attempt in range(max_retries):
        try:
            return self.fetch_artifacts(page=page)
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:
                time.sleep(2 ** attempt)  # Exponential backoff
            else:
                raise
```

**Issue**: Database connection timeout
```python
def reconnect_if_needed(self):
    """Reconnect to database if connection is lost"""
    if not self.connection or not self.connection.is_connected():
        self.connection = self._create_connection()
```

**Issue**: Memory issues with large datasets
```python
def batch_fetch_artifacts(self, total_records, batch_size=100):
    """Fetch and process in batches"""
    for start in range(0, total_records, batch_size):
        page = (start // batch_size) + 1
        artifacts = self.fetch_artifacts(page=page, size=batch_size)
        yield artifacts['records']
```

**Issue**: NULL values in SQL inserts
```python
def clean_dataframe(self, df):
    """Replace None with appropriate defaults"""
    return df.fillna({
        'culture': 'Unknown',
        'century': 'Unknown',
        'accessionyear': 0
    })
```

This skill provides a complete framework for building museum data pipelines with ETL, SQL analytics, and interactive dashboards.
