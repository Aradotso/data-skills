---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline using Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for museum data
  - create a data engineering project with Harvard API
  - set up artifact collection analytics dashboard
  - implement museum data warehouse with SQL
  - build Streamlit app for art museum analytics
  - create Harvard Art Museums data pipeline
  - develop end-to-end museum data engineering solution
  - analyze Harvard artifacts with SQL queries
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is an end-to-end data engineering and analytics application that demonstrates real-world ETL pipelines using the Harvard Art Museums API. It extracts artifact metadata, transforms nested JSON into relational structures, loads data into SQL databases (MySQL/TiDB Cloud), and provides interactive analytics through a Streamlit dashboard with 20+ predefined analytical queries.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

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

### API Setup

1. Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api
2. Store it securely in environment variables or `.env` file:

```python
# .env file
HARVARD_API_KEY=your_actual_api_key
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Database Schema

The project uses three main tables with proper relationships:

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    object_id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    division VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    object_number VARCHAR(100),
    primary_image_url TEXT,
    total_unique_pages INT,
    total_page_views INT,
    verification_level INT
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    object_id INT,
    image_id VARCHAR(100),
    base_image_url TEXT,
    caption TEXT,
    height INT,
    width INT,
    format VARCHAR(50),
    FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    object_id INT,
    color_hex VARCHAR(7),
    color_percentage DECIMAL(5,2),
    css3_color VARCHAR(50),
    hue VARCHAR(50),
    FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
);
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

class HarvardAPIClient:
    def __init__(self):
        self.api_key = os.getenv('HARVARD_API_KEY')
        self.base_url = "https://api.harvardartmuseums.org/object"
    
    def fetch_artifacts(self, size=100, page=1):
        """Fetch artifacts with pagination support"""
        params = {
            'apikey': self.api_key,
            'size': size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
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
            data = self.fetch_artifacts(size=size, page=page)
            artifacts = data.get('records', [])
            
            if not artifacts:
                break
                
            all_artifacts.extend(artifacts)
            page += 1
        
        return all_artifacts[:total_records]
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

class ArtifactETL:
    def __init__(self, db_config):
        self.db_config = db_config
        self.connection = None
    
    def connect_db(self):
        """Establish database connection"""
        self.connection = mysql.connector.connect(**self.db_config)
        return self.connection
    
    def transform_metadata(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Transform artifact data to metadata DataFrame"""
        metadata = []
        
        for artifact in artifacts:
            metadata.append({
                'object_id': artifact.get('objectid'),
                'title': artifact.get('title', '')[:500],
                'culture': artifact.get('culture', '')[:200],
                'period': artifact.get('period', '')[:200],
                'century': artifact.get('century', '')[:100],
                'classification': artifact.get('classification', '')[:200],
                'division': artifact.get('division', '')[:200],
                'department': artifact.get('department', '')[:200],
                'dated': artifact.get('dated', '')[:200],
                'object_number': artifact.get('objectnumber', '')[:100],
                'primary_image_url': artifact.get('primaryimageurl'),
                'total_unique_pages': artifact.get('totalpageviews', 0),
                'total_page_views': artifact.get('totaluniquepageviews', 0),
                'verification_level': artifact.get('verificationlevel', 0)
            })
        
        return pd.DataFrame(metadata)
    
    def transform_media(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Transform nested media/images data"""
        media_records = []
        
        for artifact in artifacts:
            object_id = artifact.get('objectid')
            images = artifact.get('images', [])
            
            for img in images:
                media_records.append({
                    'object_id': object_id,
                    'image_id': img.get('imageid'),
                    'base_image_url': img.get('baseimageurl'),
                    'caption': img.get('caption', '')[:500] if img.get('caption') else None,
                    'height': img.get('height'),
                    'width': img.get('width'),
                    'format': img.get('format', '')[:50]
                })
        
        return pd.DataFrame(media_records)
    
    def transform_colors(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Transform color analysis data"""
        color_records = []
        
        for artifact in artifacts:
            object_id = artifact.get('objectid')
            colors = artifact.get('colors', [])
            
            for color in colors:
                color_records.append({
                    'object_id': object_id,
                    'color_hex': color.get('hex'),
                    'color_percentage': color.get('percent'),
                    'css3_color': color.get('css3', '')[:50],
                    'hue': color.get('hue', '')[:50]
                })
        
        return pd.DataFrame(color_records)
    
    def load_to_sql(self, df: pd.DataFrame, table_name: str):
        """Batch insert DataFrame into SQL table"""
        cursor = self.connection.cursor()
        
        cols = ','.join(df.columns)
        placeholders = ','.join(['%s'] * len(df.columns))
        insert_query = f"INSERT IGNORE INTO {table_name} ({cols}) VALUES ({placeholders})"
        
        # Batch insert for performance
        data_tuples = [tuple(x) for x in df.to_numpy()]
        cursor.executemany(insert_query, data_tuples)
        
        self.connection.commit()
        cursor.close()
```

### 3. SQL Analytics Queries

```python
class AnalyticsQueries:
    """Predefined analytical SQL queries"""
    
    QUERIES = {
        "Artifacts by Culture": """
            SELECT culture, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE culture IS NOT NULL AND culture != ''
            GROUP BY culture
            ORDER BY artifact_count DESC
            LIMIT 15;
        """,
        
        "Artifacts by Century": """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL
            GROUP BY century
            ORDER BY count DESC;
        """,
        
        "Top Departments": """
            SELECT department, COUNT(*) as total_artifacts
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY total_artifacts DESC
            LIMIT 10;
        """,
        
        "Media Format Distribution": """
            SELECT format, COUNT(*) as image_count
            FROM artifactmedia
            WHERE format IS NOT NULL
            GROUP BY format
            ORDER BY image_count DESC;
        """,
        
        "Most Popular Colors": """
            SELECT css3_color, COUNT(*) as usage_count,
                   AVG(color_percentage) as avg_percentage
            FROM artifactcolors
            WHERE css3_color IS NOT NULL
            GROUP BY css3_color
            ORDER BY usage_count DESC
            LIMIT 20;
        """,
        
        "High Pageview Artifacts": """
            SELECT title, culture, total_page_views, primary_image_url
            FROM artifactmetadata
            WHERE total_page_views > 0
            ORDER BY total_page_views DESC
            LIMIT 20;
        """,
        
        "Artifacts with Most Images": """
            SELECT am.title, am.culture, COUNT(ai.media_id) as image_count
            FROM artifactmetadata am
            JOIN artifactmedia ai ON am.object_id = ai.object_id
            GROUP BY am.object_id, am.title, am.culture
            ORDER BY image_count DESC
            LIMIT 15;
        """,
        
        "Color Diversity by Culture": """
            SELECT am.culture, COUNT(DISTINCT ac.css3_color) as unique_colors
            FROM artifactmetadata am
            JOIN artifactcolors ac ON am.object_id = ac.object_id
            WHERE am.culture IS NOT NULL AND am.culture != ''
            GROUP BY am.culture
            ORDER BY unique_colors DESC
            LIMIT 10;
        """
    }
    
    @staticmethod
    def execute_query(connection, query_name: str):
        """Execute analytical query and return results"""
        query = AnalyticsQueries.QUERIES.get(query_name)
        
        if not query:
            raise ValueError(f"Query '{query_name}' not found")
        
        df = pd.read_sql(query, connection)
        return df
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px
import plotly.graph_objects as go

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar configuration
    with st.sidebar:
        st.header("⚙️ Configuration")
        
        # Database connection
        db_config = {
            'host': os.getenv('DB_HOST'),
            'user': os.getenv('DB_USER'),
            'password': os.getenv('DB_PASSWORD'),
            'database': os.getenv('DB_NAME')
        }
        
        if st.button("🔄 Refresh Data from API"):
            with st.spinner("Fetching data from Harvard API..."):
                client = HarvardAPIClient()
                artifacts = client.fetch_all_artifacts(total_records=500)
                
                etl = ArtifactETL(db_config)
                etl.connect_db()
                
                # Transform and load
                metadata_df = etl.transform_metadata(artifacts)
                media_df = etl.transform_media(artifacts)
                colors_df = etl.transform_colors(artifacts)
                
                etl.load_to_sql(metadata_df, 'artifactmetadata')
                etl.load_to_sql(media_df, 'artifactmedia')
                etl.load_to_sql(colors_df, 'artifactcolors')
                
                st.success(f"✅ Loaded {len(artifacts)} artifacts!")
    
    # Main analytics section
    connection = mysql.connector.connect(**db_config)
    
    query_options = list(AnalyticsQueries.QUERIES.keys())
    selected_query = st.selectbox("📊 Select Analytics Query", query_options)
    
    if st.button("Run Query"):
        with st.spinner("Executing query..."):
            df = AnalyticsQueries.execute_query(connection, selected_query)
            
            st.subheader("Query Results")
            st.dataframe(df, use_container_width=True)
            
            # Auto-visualization
            if len(df.columns) >= 2:
                fig = px.bar(
                    df, 
                    x=df.columns[0], 
                    y=df.columns[1],
                    title=selected_query,
                    labels={df.columns[0]: df.columns[0].replace('_', ' ').title(),
                            df.columns[1]: df.columns[1].replace('_', ' ').title()}
                )
                st.plotly_chart(fig, use_container_width=True)
    
    connection.close()

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# The app will open at http://localhost:8501
```

## Common Patterns

### Pattern 1: Complete ETL Workflow

```python
from dotenv import load_dotenv
import os

load_dotenv()

# 1. Extract
client = HarvardAPIClient()
artifacts = client.fetch_all_artifacts(total_records=1000)

# 2. Transform & Load
db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

etl = ArtifactETL(db_config)
etl.connect_db()

metadata_df = etl.transform_metadata(artifacts)
media_df = etl.transform_media(artifacts)
colors_df = etl.transform_colors(artifacts)

etl.load_to_sql(metadata_df, 'artifactmetadata')
etl.load_to_sql(media_df, 'artifactmedia')
etl.load_to_sql(colors_df, 'artifactcolors')

print(f"ETL Complete: {len(artifacts)} artifacts processed")
```

### Pattern 2: Custom Analytics Query

```python
import pandas as pd

def run_custom_query(connection, query):
    """Execute custom SQL query"""
    df = pd.read_sql(query, connection)
    return df

# Example: Find artifacts with dominant red colors
custom_query = """
    SELECT am.title, am.culture, ac.css3_color, ac.color_percentage
    FROM artifactmetadata am
    JOIN artifactcolors ac ON am.object_id = ac.object_id
    WHERE ac.hue = 'red' AND ac.color_percentage > 30
    ORDER BY ac.color_percentage DESC
    LIMIT 20;
"""

connection = mysql.connector.connect(**db_config)
results = run_custom_query(connection, custom_query)
print(results)
```

### Pattern 3: Incremental Data Updates

```python
def incremental_update(last_update_id=0, batch_size=100):
    """Fetch only new artifacts since last update"""
    client = HarvardAPIClient()
    
    # Fetch artifacts modified after last_update_id
    params = {
        'apikey': os.getenv('HARVARD_API_KEY'),
        'size': batch_size,
        'sort': 'objectid',
        'sortorder': 'asc',
        'q': f'objectid:>{last_update_id}'
    }
    
    response = requests.get(client.base_url, params=params)
    new_artifacts = response.json().get('records', [])
    
    # Process only new records
    if new_artifacts:
        etl = ArtifactETL(db_config)
        etl.connect_db()
        # ... transform and load
        
    return len(new_artifacts)
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(client, max_retries=3, delay=2):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            return client.fetch_artifacts()
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:  # Rate limit
                wait_time = delay * (2 ** attempt)
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
def safe_db_connect(db_config, retries=3):
    """Connect with retry logic"""
    for i in range(retries):
        try:
            connection = mysql.connector.connect(**db_config)
            return connection
        except mysql.connector.Error as err:
            print(f"Connection attempt {i+1} failed: {err}")
            time.sleep(2)
    raise Exception("Could not connect to database")
```

### Handling NULL Values

```python
def clean_dataframe(df):
    """Clean DataFrame before SQL insert"""
    # Replace None with empty string for VARCHAR fields
    string_cols = df.select_dtypes(include=['object']).columns
    df[string_cols] = df[string_cols].fillna('')
    
    # Replace NaN with 0 for numeric fields
    numeric_cols = df.select_dtypes(include=['number']).columns
    df[numeric_cols] = df[numeric_cols].fillna(0)
    
    return df
```

### Large Dataset Memory Management

```python
def process_in_chunks(artifacts, chunk_size=100):
    """Process large datasets in chunks"""
    etl = ArtifactETL(db_config)
    etl.connect_db()
    
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i+chunk_size]
        
        metadata_df = etl.transform_metadata(chunk)
        etl.load_to_sql(metadata_df, 'artifactmetadata')
        
        print(f"Processed {i+len(chunk)}/{len(artifacts)} artifacts")
```

This skill enables AI agents to help developers build complete ETL pipelines with museum data, implement SQL analytics, and create interactive dashboards using modern data engineering practices.
