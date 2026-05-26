---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for the Harvard Art Museums API
  - show me how to extract and transform Harvard museum data
  - help me set up analytics on Harvard Art Museums collection
  - create a data pipeline for art museum artifacts
  - build a Streamlit dashboard for museum collection data
  - how do I structure SQL tables for art museum metadata
  - integrate Harvard Art Museums API into my data pipeline
  - visualize museum artifact data with SQL analytics
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill helps you build end-to-end data engineering pipelines using the Harvard Art Museums API. The project demonstrates ETL operations, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact collections.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract nested JSON, transform into relational data, load into SQL databases
- **SQL Analytics**: 20+ predefined analytical queries for museum collection insights
- **Interactive Dashboards**: Streamlit-based UI with Plotly visualizations
- **Database Design**: Normalized schema for artifact metadata, media, and color data

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

Required packages:
```
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

```env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_username
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

Register at: https://docs.api.harvardartmuseums.org/

### Database Setup

The application supports MySQL or TiDB Cloud. Create the database:

```sql
CREATE DATABASE harvard_artifacts;
```

## Database Schema

The ETL pipeline creates three main tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    objectnumber VARCHAR(100),
    dated VARCHAR(255),
    division VARCHAR(255)
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(50),
    media_url TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
streamlit run app.py
```

The Streamlit interface will open at `http://localhost:8501`

## Core Usage Patterns

### 1. API Data Collection

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_pages=5):
    """Fetch artifacts from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = 'https://api.harvardartmuseums.org/object'
    
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': 100,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            all_artifacts.extend(data.get('records', []))
        else:
            print(f"Error fetching page {page}: {response.status_code}")
            break
    
    return all_artifacts
```

### 2. ETL Pipeline - Extract and Transform

```python
import pandas as pd

def transform_artifacts(raw_data):
    """Transform raw API data into structured DataFrames"""
    
    # Extract metadata
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        # Artifact metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'objectnumber': artifact.get('objectnumber'),
            'dated': artifact.get('dated'),
            'division': artifact.get('division')
        }
        metadata_list.append(metadata)
        
        # Media/images
        images = artifact.get('images', [])
        for img in images:
            media_list.append({
                'artifact_id': artifact.get('id'),
                'media_type': 'image',
                'media_url': img.get('baseimageurl')
            })
        
        # Color data
        colors = artifact.get('colors', [])
        for color in colors:
            colors_list.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'percentage': color.get('percent')
            })
    
    df_metadata = pd.DataFrame(metadata_list)
    df_media = pd.DataFrame(media_list)
    df_colors = pd.DataFrame(colors_list)
    
    return df_metadata, df_media, df_colors
```

### 3. ETL Pipeline - Load to SQL

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_to_database(df_metadata, df_media, df_colors):
    """Load transformed data into SQL database"""
    
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Insert metadata
        for _, row in df_metadata.iterrows():
            cursor.execute("""
                INSERT INTO artifactmetadata 
                (id, title, culture, period, century, classification, 
                 department, objectnumber, dated, division)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE title=VALUES(title)
            """, tuple(row))
        
        # Insert media
        for _, row in df_media.iterrows():
            cursor.execute("""
                INSERT INTO artifactmedia (artifact_id, media_type, media_url)
                VALUES (%s, %s, %s)
            """, tuple(row))
        
        # Insert colors
        for _, row in df_colors.iterrows():
            cursor.execute("""
                INSERT INTO artifactcolors (artifact_id, color, percentage)
                VALUES (%s, %s, %s)
            """, tuple(row))
        
        conn.commit()
        print(f"Loaded {len(df_metadata)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()
```

### 4. Analytical SQL Queries

```python
def run_analytics_query(query_name):
    """Execute predefined analytical queries"""
    
    queries = {
        'artifacts_by_culture': """
            SELECT culture, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE culture IS NOT NULL
            GROUP BY culture
            ORDER BY artifact_count DESC
            LIMIT 20
        """,
        
        'artifacts_by_century': """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL
            GROUP BY century
            ORDER BY count DESC
        """,
        
        'color_distribution': """
            SELECT color, COUNT(*) as frequency, 
                   ROUND(AVG(percentage), 2) as avg_percentage
            FROM artifactcolors
            GROUP BY color
            ORDER BY frequency DESC
            LIMIT 15
        """,
        
        'department_stats': """
            SELECT department, COUNT(*) as total_artifacts,
                   COUNT(DISTINCT culture) as unique_cultures
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY total_artifacts DESC
        """,
        
        'media_availability': """
            SELECT am.classification, COUNT(DISTINCT am.id) as artifact_count,
                   COUNT(media.id) as media_count
            FROM artifactmetadata am
            LEFT JOIN artifactmedia media ON am.id = media.artifact_id
            GROUP BY am.classification
            HAVING artifact_count > 10
            ORDER BY media_count DESC
        """
    }
    
    conn = get_db_connection()
    cursor = conn.cursor(dictionary=True)
    
    cursor.execute(queries[query_name])
    results = cursor.fetchall()
    
    cursor.close()
    conn.close()
    
    return pd.DataFrame(results)
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def create_dashboard():
    """Create interactive Streamlit dashboard"""
    
    st.title("🎨 Harvard Art Museums Analytics")
    st.markdown("---")
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Queries")
    
    query_options = {
        "Artifacts by Culture": "artifacts_by_culture",
        "Artifacts by Century": "artifacts_by_century",
        "Color Distribution": "color_distribution",
        "Department Statistics": "department_stats",
        "Media Availability": "media_availability"
    }
    
    selected_query = st.sidebar.selectbox(
        "Select Analysis",
        list(query_options.keys())
    )
    
    # Execute query
    if st.sidebar.button("Run Analysis"):
        with st.spinner("Executing query..."):
            df_results = run_analytics_query(query_options[selected_query])
            
            # Display results
            st.subheader(f"Results: {selected_query}")
            st.dataframe(df_results)
            
            # Visualization
            if len(df_results) > 0:
                st.subheader("Visualization")
                
                x_col = df_results.columns[0]
                y_col = df_results.columns[1]
                
                fig = px.bar(
                    df_results.head(20),
                    x=x_col,
                    y=y_col,
                    title=f"{selected_query}",
                    labels={x_col: x_col.replace('_', ' ').title(),
                           y_col: y_col.replace('_', ' ').title()}
                )
                
                st.plotly_chart(fig, use_container_width=True)
```

## Complete ETL Workflow

```python
def run_complete_etl_pipeline():
    """Execute full ETL pipeline"""
    
    print("Step 1: Extracting data from API...")
    raw_artifacts = fetch_artifacts(num_pages=10)
    print(f"Extracted {len(raw_artifacts)} artifacts")
    
    print("\nStep 2: Transforming data...")
    df_metadata, df_media, df_colors = transform_artifacts(raw_artifacts)
    print(f"Transformed into {len(df_metadata)} metadata records")
    print(f"Generated {len(df_media)} media records")
    print(f"Generated {len(df_colors)} color records")
    
    print("\nStep 3: Loading data to database...")
    load_to_database(df_metadata, df_media, df_colors)
    
    print("\nETL pipeline completed successfully!")

# Run the pipeline
if __name__ == "__main__":
    run_complete_etl_pipeline()
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(url, params, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        response = requests.get(url, params=params)
        
        if response.status_code == 200:
            return response.json()
        elif response.status_code == 429:  # Rate limited
            wait_time = 2 ** attempt
            print(f"Rate limited. Waiting {wait_time}s...")
            time.sleep(wait_time)
        else:
            print(f"Error {response.status_code}")
            break
    
    return None
```

### Database Connection Issues

```python
def test_db_connection():
    """Test database connectivity"""
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        cursor.close()
        conn.close()
        print("✓ Database connection successful")
        return True
    except Error as e:
        print(f"✗ Database connection failed: {e}")
        return False
```

### Handling NULL Values

```python
def clean_dataframe(df):
    """Clean DataFrame before database insertion"""
    # Replace None with empty strings for VARCHAR columns
    string_columns = df.select_dtypes(include=['object']).columns
    df[string_columns] = df[string_columns].fillna('')
    
    # Replace None with 0 for numeric columns
    numeric_columns = df.select_dtypes(include=['number']).columns
    df[numeric_columns] = df[numeric_columns].fillna(0)
    
    return df
```

## Common Patterns

### Incremental Updates

```python
def get_last_artifact_id():
    """Get the last loaded artifact ID"""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    conn.close()
    return result[0] if result[0] else 0

def incremental_etl():
    """Load only new artifacts"""
    last_id = get_last_artifact_id()
    # Fetch artifacts with ID > last_id
    # Process and load new data
```

### Data Quality Checks

```python
def validate_data_quality(df_metadata):
    """Run data quality checks"""
    checks = {
        'null_titles': df_metadata['title'].isnull().sum(),
        'duplicate_ids': df_metadata['id'].duplicated().sum(),
        'missing_culture': df_metadata['culture'].isnull().sum()
    }
    
    for check, count in checks.items():
        if count > 0:
            print(f"Warning: {count} records with {check}")
    
    return checks
```

This skill provides everything needed to build production-ready ETL pipelines for museum collection data with modern data engineering practices.
