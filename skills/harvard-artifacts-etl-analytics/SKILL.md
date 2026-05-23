---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API data with MySQL and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - set up analytics dashboard for museum artifacts
  - query Harvard artifacts API and store in database
  - create data engineering pipeline for art collection
  - analyze Harvard museum data with SQL
  - build Streamlit app for artifact analytics
  - extract and visualize museum collection data
  - implement ETL workflow for art museum API
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering and analytics solution for the Harvard Art Museums API. It demonstrates ETL pipelines, SQL analytics, and interactive Streamlit dashboards for museum artifact data.

## What It Does

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads artifact metadata, media, and color data into relational tables
- **SQL Analytics**: Runs 20+ predefined analytical queries on artifact collections
- **Visualization**: Creates interactive Plotly dashboards in Streamlit
- **Database Design**: Implements normalized schema with proper foreign key relationships

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

### Required Dependencies

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

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# MySQL/TiDB Connection
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Database Setup

Create the required tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dated VARCHAR(200),
    url VARCHAR(500)
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
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

## Key Usage Patterns

### Running the Streamlit App

```bash
streamlit run app.py
```

### ETL Pipeline Implementation

#### Extract: Fetch Data from API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
    url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Fetch multiple pages
all_artifacts = []
for page in range(1, 6):  # First 5 pages
    data = fetch_artifacts(page=page)
    all_artifacts.extend(data['records'])
    print(f"Fetched page {page}: {len(data['records'])} artifacts")
```

#### Transform: Process JSON to DataFrames

```python
import pandas as pd

def transform_artifacts(raw_data):
    """Transform raw API data into structured DataFrames"""
    
    # Metadata DataFrame
    metadata = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url')
        })
        
        # Extract media
        if artifact.get('images'):
            for img in artifact['images']:
                media_list.append({
                    'artifact_id': artifact.get('id'),
                    'media_type': 'image',
                    'media_url': img.get('baseimageurl')
                })
        
        # Extract colors
        if artifact.get('colors'):
            for color in artifact['colors']:
                colors_list.append({
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color'),
                    'percentage': color.get('percent')
                })
    
    df_metadata = pd.DataFrame(metadata)
    df_media = pd.DataFrame(media_list)
    df_colors = pd.DataFrame(colors_list)
    
    return df_metadata, df_media, df_colors
```

#### Load: Insert into Database

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=os.getenv('DB_PORT'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_to_database(df_metadata, df_media, df_colors):
    """Load DataFrames into MySQL database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Insert metadata (batch insert)
        metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, technique, medium, dated, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_query, df_metadata.values.tolist())
        
        # Insert media
        media_query = """
        INSERT INTO artifactmedia (artifact_id, media_type, media_url)
        VALUES (%s, %s, %s)
        """
        cursor.executemany(media_query, df_media.values.tolist())
        
        # Insert colors
        colors_query = """
        INSERT INTO artifactcolors (artifact_id, color, percentage)
        VALUES (%s, %s, %s)
        """
        cursor.executemany(colors_query, df_colors.values.tolist())
        
        conn.commit()
        print(f"Loaded {len(df_metadata)} artifacts successfully")
        
    except Error as e:
        print(f"Error loading data: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()
```

### SQL Analytics Queries

```python
def run_analytics_query(query_name):
    """Execute predefined analytics queries"""
    
    queries = {
        'artifacts_by_culture': """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture IS NOT NULL
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 10
        """,
        
        'artifacts_by_century': """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL
            GROUP BY century
            ORDER BY count DESC
        """,
        
        'top_colors': """
            SELECT color, AVG(percentage) as avg_percentage, COUNT(*) as usage_count
            FROM artifactcolors
            GROUP BY color
            ORDER BY usage_count DESC
            LIMIT 10
        """,
        
        'media_availability': """
            SELECT 
                m.department,
                COUNT(DISTINCT m.id) as total_artifacts,
                COUNT(DISTINCT med.artifact_id) as artifacts_with_media,
                ROUND(COUNT(DISTINCT med.artifact_id) * 100.0 / COUNT(DISTINCT m.id), 2) as media_percentage
            FROM artifactmetadata m
            LEFT JOIN artifactmedia med ON m.id = med.artifact_id
            GROUP BY m.department
            ORDER BY media_percentage DESC
        """,
        
        'artifacts_by_technique': """
            SELECT technique, COUNT(*) as count
            FROM artifactmetadata
            WHERE technique IS NOT NULL
            GROUP BY technique
            ORDER BY count DESC
            LIMIT 15
        """
    }
    
    conn = get_db_connection()
    df = pd.read_sql(queries[query_name], conn)
    conn.close()
    
    return df
```

### Streamlit Dashboard Integration

```python
import streamlit as st
import plotly.express as px

def create_dashboard():
    """Create interactive analytics dashboard"""
    
    st.title("Harvard Artifacts Analytics Dashboard")
    
    # Query selector
    query_options = {
        'Artifacts by Culture': 'artifacts_by_culture',
        'Artifacts by Century': 'artifacts_by_century',
        'Top Colors Used': 'top_colors',
        'Media Availability': 'media_availability',
        'Artifacts by Technique': 'artifacts_by_technique'
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
    
    if st.button("Run Analysis"):
        with st.spinner("Executing query..."):
            df = run_analytics_query(query_options[selected_query])
            
            # Display results
            st.dataframe(df)
            
            # Create visualization
            if len(df.columns) >= 2:
                fig = px.bar(
                    df,
                    x=df.columns[0],
                    y=df.columns[1],
                    title=selected_query
                )
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    create_dashboard()
```

## Complete ETL Workflow

```python
def full_etl_pipeline(num_pages=5):
    """Execute complete ETL workflow"""
    
    print("Starting ETL Pipeline...")
    
    # EXTRACT
    print("Step 1: Extracting data from API...")
    all_artifacts = []
    for page in range(1, num_pages + 1):
        data = fetch_artifacts(page=page)
        all_artifacts.extend(data['records'])
        print(f"  - Fetched page {page}: {len(data['records'])} records")
    
    # TRANSFORM
    print("\nStep 2: Transforming data...")
    df_metadata, df_media, df_colors = transform_artifacts(all_artifacts)
    print(f"  - Metadata records: {len(df_metadata)}")
    print(f"  - Media records: {len(df_media)}")
    print(f"  - Color records: {len(df_colors)}")
    
    # LOAD
    print("\nStep 3: Loading data to database...")
    load_to_database(df_metadata, df_media, df_colors)
    
    print("\nETL Pipeline completed successfully!")

# Run the pipeline
full_etl_pipeline(num_pages=5)
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(page, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page=page)
        except Exception as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt
                print(f"Retry {attempt + 1} after {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise e
```

### Database Connection Issues

```python
def verify_db_connection():
    """Test database connectivity"""
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        cursor.close()
        conn.close()
        print("Database connection successful!")
        return True
    except Error as e:
        print(f"Database connection failed: {e}")
        return False
```

### Handling Missing Data

```python
def clean_artifact_data(artifact):
    """Clean and validate artifact data"""
    return {
        'id': artifact.get('id', 0),
        'title': artifact.get('title', 'Unknown')[:500],
        'culture': artifact.get('culture', 'Unknown')[:200],
        'century': artifact.get('century', 'Unknown')[:100],
        'classification': artifact.get('classification', 'Unclassified')[:200],
        'department': artifact.get('department', 'Unknown')[:200],
        'technique': artifact.get('technique', '')[:500],
        'medium': artifact.get('medium', '')[:500],
        'dated': artifact.get('dated', '')[:200],
        'url': artifact.get('url', '')[:500]
    }
```

## Best Practices

- Always use environment variables for sensitive credentials
- Implement batch inserts for better database performance
- Add proper error handling and logging for production use
- Use connection pooling for high-volume ETL operations
- Validate data before insertion to prevent constraint violations
- Monitor API rate limits and implement backoff strategies
