---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for art museum artifacts
  - extract and transform Harvard museum API data
  - visualize art museum collection data with Streamlit
  - set up SQL database for artifact metadata
  - analyze Harvard Art Museums collection data
  - build data engineering pipeline for museum artifacts
  - create interactive visualization for art collection data
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL analytics, and interactive data visualization using Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering & Analytics App provides:

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads artifact metadata, media, and color data into relational SQL databases
- **SQL Analytics**: Executes predefined analytical queries on structured artifact data
- **Interactive Dashboards**: Visualizes query results using Streamlit and Plotly

**Architecture Flow**: API → ETL → SQL → Analytics → Visualization

## Installation

### Prerequisites

```bash
# Python 3.8 or higher required
python --version

# Install required packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Project Setup

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

### Environment Configuration

Create a `.env` file in the project root:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# MySQL/TiDB Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

## Database Setup

### Creating Tables

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    division VARCHAR(200),
    medium VARCHAR(500),
    technique VARCHAR(500),
    period VARCHAR(200),
    primaryimageurl TEXT,
    verificationlevel INT,
    accessionyear INT
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl TEXT,
    format VARCHAR(50),
    height INT,
    width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Fetch multiple pages with pagination
def fetch_all_artifacts(total_records=500):
    """Fetch artifacts with pagination handling"""
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < total_records:
        data = fetch_artifacts(page=page, size=size)
        records = data.get('records', [])
        
        if not records:
            break
            
        all_artifacts.extend(records)
        page += 1
        
        # Rate limiting
        import time
        time.sleep(0.5)
    
    return all_artifacts[:total_records]
```

### 2. ETL Transformation

```python
import pandas as pd

def transform_artifacts(raw_data):
    """Transform nested JSON into relational dataframes"""
    
    # Metadata transformation
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'division': artifact.get('division'),
            'medium': artifact.get('medium'),
            'technique': artifact.get('technique'),
            'period': artifact.get('period'),
            'primaryimageurl': artifact.get('primaryimageurl'),
            'verificationlevel': artifact.get('verificationlevel'),
            'accessionyear': artifact.get('accessionyear')
        }
        metadata_records.append(metadata)
        
        # Extract media
        images = artifact.get('images', [])
        for image in images:
            media = {
                'artifact_id': artifact.get('id'),
                'baseimageurl': image.get('baseimageurl'),
                'format': image.get('format'),
                'height': image.get('height'),
                'width': image.get('width')
            }
            media_records.append(media)
        
        # Extract colors
        colors = artifact.get('colors', [])
        for color in colors:
            color_record = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            color_records.append(color_record)
    
    return {
        'metadata': pd.DataFrame(metadata_records),
        'media': pd.DataFrame(media_records),
        'colors': pd.DataFrame(color_records)
    }
```

### 3. SQL Loading

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

def load_to_sql(dataframes):
    """Batch insert dataframes into SQL tables"""
    connection = get_db_connection()
    cursor = connection.cursor()
    
    try:
        # Load metadata
        for _, row in dataframes['metadata'].iterrows():
            cursor.execute("""
                INSERT INTO artifactmetadata 
                (id, title, culture, century, classification, department, 
                 dated, division, medium, technique, period, primaryimageurl, 
                 verificationlevel, accessionyear)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE title=VALUES(title)
            """, tuple(row))
        
        # Load media
        for _, row in dataframes['media'].iterrows():
            cursor.execute("""
                INSERT INTO artifactmedia 
                (artifact_id, baseimageurl, format, height, width)
                VALUES (%s, %s, %s, %s, %s)
            """, tuple(row))
        
        # Load colors
        for _, row in dataframes['colors'].iterrows():
            cursor.execute("""
                INSERT INTO artifactcolors 
                (artifact_id, color, spectrum, hue, percent)
                VALUES (%s, %s, %s, %s, %s)
            """, tuple(row))
        
        connection.commit()
        print(f"Loaded {len(dataframes['metadata'])} artifacts successfully")
        
    except Error as e:
        print(f"Error loading data: {e}")
        connection.rollback()
    finally:
        cursor.close()
        connection.close()
```

### 4. Analytics Queries

```python
def execute_analytics_query(query_name):
    """Execute predefined analytics queries"""
    
    queries = {
        "artifacts_by_culture": """
            SELECT culture, COUNT(*) as count 
            FROM artifactmetadata 
            WHERE culture IS NOT NULL 
            GROUP BY culture 
            ORDER BY count DESC 
            LIMIT 10
        """,
        
        "artifacts_by_century": """
            SELECT century, COUNT(*) as count 
            FROM artifactmetadata 
            WHERE century IS NOT NULL 
            GROUP BY century 
            ORDER BY count DESC
        """,
        
        "top_colors": """
            SELECT color, COUNT(*) as frequency 
            FROM artifactcolors 
            GROUP BY color 
            ORDER BY frequency DESC 
            LIMIT 15
        """,
        
        "media_availability": """
            SELECT 
                CASE WHEN primaryimageurl IS NOT NULL 
                     THEN 'With Image' 
                     ELSE 'No Image' 
                END as image_status,
                COUNT(*) as count
            FROM artifactmetadata
            GROUP BY image_status
        """,
        
        "artifacts_by_department": """
            SELECT department, COUNT(*) as count
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY count DESC
        """
    }
    
    connection = get_db_connection()
    cursor = connection.cursor(dictionary=True)
    
    try:
        cursor.execute(queries[query_name])
        results = cursor.fetchall()
        return pd.DataFrame(results)
    finally:
        cursor.close()
        connection.close()
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("Harvard Art Museums Analytics Dashboard")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "Analytics Dashboard", "Visualizations"]
    )
    
    if page == "Data Collection":
        st.header("Collect Artifact Data")
        
        num_records = st.number_input(
            "Number of records to fetch", 
            min_value=10, 
            max_value=1000, 
            value=100
        )
        
        if st.button("Fetch and Load Data"):
            with st.spinner("Fetching data from API..."):
                raw_data = fetch_all_artifacts(total_records=num_records)
                
            with st.spinner("Transforming data..."):
                dataframes = transform_artifacts(raw_data)
                
            with st.spinner("Loading to database..."):
                load_to_sql(dataframes)
                
            st.success(f"Successfully loaded {num_records} artifacts!")
    
    elif page == "Analytics Dashboard":
        st.header("SQL Analytics")
        
        query_options = [
            "artifacts_by_culture",
            "artifacts_by_century",
            "top_colors",
            "media_availability",
            "artifacts_by_department"
        ]
        
        selected_query = st.selectbox("Select Analysis", query_options)
        
        if st.button("Run Query"):
            results_df = execute_analytics_query(selected_query)
            st.dataframe(results_df)
            
            # Auto-generate visualization
            if len(results_df.columns) >= 2:
                fig = px.bar(
                    results_df,
                    x=results_df.columns[0],
                    y=results_df.columns[1],
                    title=selected_query.replace('_', ' ').title()
                )
                st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# The app will open in your browser at http://localhost:8501
```

## Common Patterns

### Full ETL Pipeline Execution

```python
# Complete pipeline from API to visualization
def run_full_pipeline(num_artifacts=500):
    # Extract
    print("Extracting data from API...")
    raw_data = fetch_all_artifacts(total_records=num_artifacts)
    
    # Transform
    print("Transforming data...")
    dataframes = transform_artifacts(raw_data)
    
    # Load
    print("Loading to database...")
    load_to_sql(dataframes)
    
    # Validate
    print("Running validation query...")
    connection = get_db_connection()
    cursor = connection.cursor()
    cursor.execute("SELECT COUNT(*) FROM artifactmetadata")
    count = cursor.fetchone()[0]
    cursor.close()
    connection.close()
    
    print(f"Pipeline complete. Total artifacts in database: {count}")
    return count
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_api_fetch(page, size, max_retries=3):
    """Fetch with retry logic"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page, size)
        except Exception as e:
            logger.warning(f"Attempt {attempt + 1} failed: {e}")
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

## Troubleshooting

### API Rate Limiting
```python
# Add delays between requests
import time
time.sleep(1)  # Wait 1 second between API calls
```

### Database Connection Issues
```python
# Test connection
try:
    connection = get_db_connection()
    print("Database connection successful")
    connection.close()
except Error as e:
    print(f"Connection failed: {e}")
    # Check DB_HOST, DB_PORT, DB_USER, DB_PASSWORD in .env
```

### Missing API Key
```python
# Verify API key is loaded
api_key = os.getenv('HARVARD_API_KEY')
if not api_key:
    raise ValueError("HARVARD_API_KEY not found in environment variables")
```

### Duplicate Key Errors
```python
# Use ON DUPLICATE KEY UPDATE for idempotent inserts
cursor.execute("""
    INSERT INTO artifactmetadata (id, title, ...) 
    VALUES (%s, %s, ...)
    ON DUPLICATE KEY UPDATE title=VALUES(title)
""", data)
```

This skill enables AI coding agents to build production-ready ETL pipelines and analytics dashboards for museum artifact data using modern Python data engineering tools.
