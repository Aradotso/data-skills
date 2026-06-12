---
name: harvard-art-museums-etl-analytics
description: Build end-to-end data pipelines with the Harvard Art Museums API, ETL processing, SQL storage, and Streamlit analytics dashboards
triggers:
  - "build a data pipeline for museum artifacts"
  - "create an ETL pipeline with Harvard Art Museums API"
  - "set up analytics dashboard with Streamlit and SQL"
  - "extract and transform art collection data"
  - "build museum artifact analytics application"
  - "create data engineering pipeline for API data"
  - "visualize Harvard art collection data"
  - "implement batch ETL for museum API"
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build complete data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, relational database design, SQL analytics, and interactive visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color information
- **SQL Database**: Store structured data in MySQL/TiDB Cloud with proper relational design
- **Analytics Queries**: 20+ predefined SQL queries for artifact analysis
- **Interactive Dashboards**: Streamlit-based visualization using Plotly charts

## Installation

### Prerequisites

```bash
# Python 3.8+
python --version

# Install dependencies
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Project Setup

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install requirements
pip install -r requirements.txt

# Create environment file
cat > .env << EOF
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
EOF
```

### Get Harvard API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Request an API key (free)
3. Add to your `.env` file

## Project Architecture

```
API → Extract → Transform → Load → SQL Database → Analytics → Visualization

Components:
- API Source: Harvard Art Museums API
- ETL: Python (requests, pandas)
- Database: MySQL/TiDB Cloud
- Backend: Python
- Frontend: Streamlit
- Visualization: Plotly
```

## Database Schema

The application uses three main tables:

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    technique VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255)
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
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

## Key Code Patterns

### 1. API Data Collection

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'fields': 'id,title,culture,period,century,classification,department,technique,division,dated,primaryimageurl,baseimageurl,colors,images'
    }
    
    response = requests.get(url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=50)
artifacts = data['records']
total_records = data['info']['totalrecords']
```

### 2. ETL Pipeline Implementation

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

def extract_artifacts(api_key: str, num_pages: int = 5) -> List[Dict]:
    """Extract artifacts with pagination handling"""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        try:
            data = fetch_artifacts(api_key, page=page, size=100)
            all_artifacts.extend(data['records'])
            print(f"Extracted page {page}: {len(data['records'])} artifacts")
        except Exception as e:
            print(f"Error on page {page}: {e}")
            break
    
    return all_artifacts

def transform_metadata(artifacts: List[Dict]) -> pd.DataFrame:
    """Transform artifacts into metadata DataFrame"""
    metadata_records = []
    
    for artifact in artifacts:
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],  # Truncate to fit schema
            'culture': artifact.get('culture', ''),
            'period': artifact.get('period', ''),
            'century': artifact.get('century', ''),
            'classification': artifact.get('classification', ''),
            'department': artifact.get('department', ''),
            'technique': artifact.get('technique', ''),
            'division': artifact.get('division', ''),
            'dated': artifact.get('dated', '')
        })
    
    return pd.DataFrame(metadata_records)

def transform_media(artifacts: List[Dict]) -> pd.DataFrame:
    """Transform image/media information"""
    media_records = []
    
    for artifact in artifacts:
        if artifact.get('images') or artifact.get('primaryimageurl'):
            media_records.append({
                'artifact_id': artifact.get('id'),
                'baseimageurl': artifact.get('baseimageurl', ''),
                'primaryimageurl': artifact.get('primaryimageurl', ''),
                'iiifbaseuri': artifact.get('images', [{}])[0].get('iiifbaseuri', '') if artifact.get('images') else ''
            })
    
    return pd.DataFrame(media_records)

def transform_colors(artifacts: List[Dict]) -> pd.DataFrame:
    """Transform color information"""
    color_records = []
    
    for artifact in artifacts:
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_records.append({
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color', ''),
                    'spectrum': color.get('spectrum', ''),
                    'hue': color.get('hue', ''),
                    'percent': color.get('percent', 0.0)
                })
    
    return pd.DataFrame(color_records)
```

### 3. Database Loading

```python
def get_db_connection():
    """Create database connection from environment variables"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_metadata(df: pd.DataFrame):
    """Load metadata with batch insert"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    # Use REPLACE to handle duplicates
    insert_query = """
    REPLACE INTO artifactmetadata 
    (id, title, culture, period, century, classification, department, technique, division, dated)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    """
    
    records = df.to_records(index=False).tolist()
    cursor.executemany(insert_query, records)
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print(f"Loaded {len(records)} metadata records")

def load_media(df: pd.DataFrame):
    """Load media information"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (artifact_id, baseimageurl, primaryimageurl, iiifbaseuri)
    VALUES (%s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE 
    baseimageurl=VALUES(baseimageurl),
    primaryimageurl=VALUES(primaryimageurl)
    """
    
    records = df.to_records(index=False).tolist()
    cursor.executemany(insert_query, records)
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print(f"Loaded {len(records)} media records")

def load_colors(df: pd.DataFrame):
    """Load color information"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    # Clear existing colors for these artifacts to avoid duplicates
    artifact_ids = df['artifact_id'].unique().tolist()
    if artifact_ids:
        cursor.execute(
            f"DELETE FROM artifactcolors WHERE artifact_id IN ({','.join(['%s']*len(artifact_ids))})",
            artifact_ids
        )
    
    insert_query = """
    INSERT INTO artifactcolors 
    (artifact_id, color, spectrum, hue, percent)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    records = df.to_records(index=False).tolist()
    cursor.executemany(insert_query, records)
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print(f"Loaded {len(records)} color records")
```

### 4. Complete ETL Orchestration

```python
def run_etl_pipeline(api_key: str, num_pages: int = 5):
    """
    Run complete ETL pipeline
    """
    print("Starting ETL Pipeline...")
    
    # Extract
    print("\n1. Extracting data from API...")
    artifacts = extract_artifacts(api_key, num_pages)
    print(f"Extracted {len(artifacts)} total artifacts")
    
    # Transform
    print("\n2. Transforming data...")
    metadata_df = transform_metadata(artifacts)
    media_df = transform_media(artifacts)
    colors_df = transform_colors(artifacts)
    
    print(f"  - Metadata records: {len(metadata_df)}")
    print(f"  - Media records: {len(media_df)}")
    print(f"  - Color records: {len(colors_df)}")
    
    # Load
    print("\n3. Loading data to database...")
    load_metadata(metadata_df)
    load_media(media_df)
    load_colors(colors_df)
    
    print("\n✓ ETL Pipeline completed successfully!")
    
    return {
        'artifacts': len(artifacts),
        'metadata': len(metadata_df),
        'media': len(media_df),
        'colors': len(colors_df)
    }
```

## Analytics Queries

### Example SQL Analytics

```python
def get_artifacts_by_culture():
    """Get artifact count by culture"""
    query = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL AND culture != ''
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 15
    """
    
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    
    return df

def get_artifacts_by_century():
    """Get artifact distribution by century"""
    query = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL AND century != ''
    GROUP BY century
    ORDER BY count DESC
    """
    
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    
    return df

def get_color_distribution():
    """Get most common colors across artifacts"""
    query = """
    SELECT color, COUNT(*) as color_count, AVG(percent) as avg_percent
    FROM artifactcolors
    WHERE color IS NOT NULL
    GROUP BY color
    ORDER BY color_count DESC
    LIMIT 20
    """
    
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    
    return df

def get_artifacts_with_images():
    """Count artifacts with available images"""
    query = """
    SELECT 
        CASE 
            WHEN primaryimageurl IS NOT NULL AND primaryimageurl != '' THEN 'With Images'
            ELSE 'Without Images'
        END as image_status,
        COUNT(*) as count
    FROM artifactmedia
    GROUP BY image_status
    """
    
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    
    return df

def get_department_breakdown():
    """Get artifact count by department"""
    query = """
    SELECT department, COUNT(*) as count
    FROM artifactmetadata
    WHERE department IS NOT NULL AND department != ''
    GROUP BY department
    ORDER BY count DESC
    """
    
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    
    return df
```

## Streamlit Dashboard

### Basic Streamlit App Structure

```python
import streamlit as st
import plotly.express as px
from dotenv import load_dotenv

load_dotenv()

st.set_page_config(
    page_title="Harvard Art Museums Analytics",
    page_icon="🎨",
    layout="wide"
)

st.title("🎨 Harvard Art Museums Analytics Dashboard")
st.markdown("End-to-end data pipeline and analytics for Harvard Art Museums collection")

# Sidebar for ETL controls
st.sidebar.header("ETL Pipeline")

if st.sidebar.button("Run ETL Pipeline"):
    with st.spinner("Running ETL pipeline..."):
        api_key = os.getenv('HARVARD_API_KEY')
        results = run_etl_pipeline(api_key, num_pages=3)
        st.sidebar.success(f"✓ Pipeline complete! Processed {results['artifacts']} artifacts")

# Main dashboard
st.header("📊 Analytics Queries")

query_option = st.selectbox(
    "Select Analysis",
    [
        "Artifacts by Culture",
        "Artifacts by Century",
        "Color Distribution",
        "Image Availability",
        "Department Breakdown"
    ]
)

if st.button("Run Query"):
    with st.spinner("Executing query..."):
        if query_option == "Artifacts by Culture":
            df = get_artifacts_by_culture()
            st.dataframe(df)
            
            fig = px.bar(df, x='culture', y='artifact_count', 
                        title='Artifact Count by Culture')
            st.plotly_chart(fig, use_container_width=True)
            
        elif query_option == "Artifacts by Century":
            df = get_artifacts_by_century()
            st.dataframe(df)
            
            fig = px.bar(df, x='century', y='count',
                        title='Artifact Distribution by Century')
            st.plotly_chart(fig, use_container_width=True)
            
        elif query_option == "Color Distribution":
            df = get_color_distribution()
            st.dataframe(df)
            
            fig = px.bar(df, x='color', y='color_count',
                        title='Most Common Colors in Collection',
                        color='avg_percent')
            st.plotly_chart(fig, use_container_width=True)
            
        elif query_option == "Image Availability":
            df = get_artifacts_with_images()
            st.dataframe(df)
            
            fig = px.pie(df, values='count', names='image_status',
                        title='Image Availability Status')
            st.plotly_chart(fig, use_container_width=True)
            
        elif query_option == "Department Breakdown":
            df = get_department_breakdown()
            st.dataframe(df)
            
            fig = px.bar(df, x='department', y='count',
                        title='Artifacts by Department')
            st.plotly_chart(fig, use_container_width=True)
```

### Running the Streamlit App

```bash
# Start the dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Configuration

### Environment Variables

```bash
# .env file structure
HARVARD_API_KEY=your_harvard_api_key
DB_HOST=your_database_host
DB_USER=your_db_username
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
DB_PORT=3306  # Optional, defaults to 3306
```

### Database Setup

```python
def create_database_schema():
    """Initialize database schema"""
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD')
    )
    cursor = conn.cursor()
    
    # Create database
    cursor.execute(f"CREATE DATABASE IF NOT EXISTS {os.getenv('DB_NAME')}")
    cursor.execute(f"USE {os.getenv('DB_NAME')}")
    
    # Create tables
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS artifactmetadata (
        id INT PRIMARY KEY,
        title VARCHAR(500),
        culture VARCHAR(255),
        period VARCHAR(255),
        century VARCHAR(255),
        classification VARCHAR(255),
        department VARCHAR(255),
        technique VARCHAR(255),
        division VARCHAR(255),
        dated VARCHAR(255)
    )
    """)
    
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS artifactmedia (
        media_id INT AUTO_INCREMENT PRIMARY KEY,
        artifact_id INT,
        baseimageurl VARCHAR(500),
        primaryimageurl VARCHAR(500),
        iiifbaseuri VARCHAR(500),
        FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id) ON DELETE CASCADE
    )
    """)
    
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS artifactcolors (
        color_id INT AUTO_INCREMENT PRIMARY KEY,
        artifact_id INT,
        color VARCHAR(50),
        spectrum VARCHAR(50),
        hue VARCHAR(50),
        percent FLOAT,
        FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id) ON DELETE CASCADE
    )
    """)
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print("✓ Database schema created successfully")
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_artifacts_with_rate_limit(api_key, num_pages, delay=0.5):
    """Fetch artifacts with rate limiting to avoid API throttling"""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        try:
            data = fetch_artifacts(api_key, page=page)
            all_artifacts.extend(data['records'])
            print(f"Page {page}/{num_pages}: {len(data['records'])} artifacts")
            
            # Rate limiting
            if page < num_pages:
                time.sleep(delay)
                
        except Exception as e:
            print(f"Error on page {page}: {e}")
            break
    
    return all_artifacts
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)

def safe_etl_pipeline(api_key, num_pages=5):
    """ETL pipeline with comprehensive error handling"""
    try:
        logging.info("Starting ETL pipeline...")
        
        # Extract
        artifacts = extract_artifacts(api_key, num_pages)
        logging.info(f"Extracted {len(artifacts)} artifacts")
        
        # Transform
        metadata_df = transform_metadata(artifacts)
        media_df = transform_media(artifacts)
        colors_df = transform_colors(artifacts)
        
        # Load
        load_metadata(metadata_df)
        load_media(media_df)
        load_colors(colors_df)
        
        logging.info("ETL pipeline completed successfully")
        return True
        
    except requests.exceptions.RequestException as e:
        logging.error(f"API request failed: {e}")
        return False
    except mysql.connector.Error as e:
        logging.error(f"Database error: {e}")
        return False
    except Exception as e:
        logging.error(f"Unexpected error: {e}")
        return False
```

### Incremental Data Loading

```python
def get_last_loaded_id():
    """Get the highest artifact ID already in database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    
    cursor.close()
    conn.close()
    
    return result[0] if result[0] else 0

def incremental_etl(api_key, num_pages=5):
    """Only load artifacts newer than what's in database"""
    last_id = get_last_loaded_id()
    print(f"Last loaded artifact ID: {last_id}")
    
    artifacts = extract_artifacts(api_key, num_pages)
    
    # Filter to only new artifacts
    new_artifacts = [a for a in artifacts if a.get('id', 0) > last_id]
    print(f"Found {len(new_artifacts)} new artifacts")
    
    if new_artifacts:
        metadata_df = transform_metadata(new_artifacts)
        media_df = transform_media(new_artifacts)
        colors_df = transform_colors(new_artifacts)
        
        load_metadata(metadata_df)
        load_media(media_df)
        load_colors(colors_df)
```

## Troubleshooting

### API Connection Issues

```python
# Test API connectivity
def test_api_connection(api_key):
    """Verify API key and connection"""
    try:
        response = requests.get(
            "https://api.harvardartmuseums.org/object",
            params={'apikey': api_key, 'size': 1},
            timeout=10
        )
        
        if response.status_code == 200:
            print("✓ API connection successful")
            return True
        elif response.status_code == 401:
            print("✗ Invalid API key")
            return False
        else:
            print(f"✗ API error: {response.status_code}")
            return False
            
    except requests.exceptions.Timeout:
        print("✗ API request timed out")
        return False
    except Exception as e:
        print(f"✗ Connection error: {e}")
        return False
```

### Database Connection Issues

```python
def test_db_connection():
    """Verify database connectivity"""
    try:
        conn = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME'),
            connect_timeout=10
        )
        
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        
        cursor.close()
        conn.close()
        
        print("✓ Database connection successful")
        return True
        
    except mysql.connector.Error as e:
        print(f"✗ Database error: {e}")
        return False
```

### Common Issues

**Issue: "API rate limit exceeded"**
```python
# Solution: Add delays between requests
time.sleep(1)  # 1 second delay between API calls
```

**Issue: "Foreign key constraint fails"**
```python
# Solution: Load metadata before media/colors
load_metadata(metadata_df)  # Load parent table first
load_media(media_df)        # Then child tables
load_colors(colors_df)
```

**Issue: "Data too long for column"**
```python
# Solution: Truncate long strings during transform
'title': artifact.get('title', '')[:500]  # Limit to column size
```

**Issue: "Duplicate entry for key PRIMARY"**
```python
# Solution: Use REPLACE or ON DUPLICATE KEY UPDATE
cursor.execute("REPLACE INTO artifactmetadata ...")
# or
cursor.execute("INSERT ... ON DUPLICATE KEY UPDATE ...")
```

This skill provides complete guidance for building data engineering pipelines with the Harvard Art Museums API, from extraction through visualization.
