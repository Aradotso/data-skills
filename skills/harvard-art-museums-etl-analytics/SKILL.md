---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - create an ETL pipeline for Harvard Art Museums API
  - build analytics dashboard for museum artifacts data
  - set up Harvard museums data engineering project
  - analyze art collections with SQL and visualization
  - integrate Harvard Art Museums API with database
  - create Streamlit app for museum data analytics
  - build artifact collection data pipeline
  - query and visualize Harvard museum artifacts
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL analytics, and interactive data visualization with Streamlit.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load museum data into relational databases
- **SQL Analytics**: Execute analytical queries on artifact metadata, media, and color data
- **Interactive Dashboards**: Visualize insights using Streamlit and Plotly
- **Database Design**: Structured relational schema for artifact storage and analysis

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
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

### API Key Setup

1. Get your free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Create a `.env` file:

```bash
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Database Setup

Create the required tables in MySQL/TiDB:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dated VARCHAR(255),
    period VARCHAR(255),
    url TEXT,
    creditline TEXT,
    division VARCHAR(255),
    imagepermissionlevel INT
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl TEXT,
    format VARCHAR(50),
    iiifbaseuri TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

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

## Key Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_records=100):
    """Fetch artifact data from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    artifacts = []
    page = 1
    per_page = 100
    
    while len(artifacts) < num_records:
        params = {
            'apikey': api_key,
            'size': per_page,
            'page': page
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            artifacts.extend(data['records'])
            
            if len(data['records']) == 0:
                break
            
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_records]
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

def transform_artifacts(raw_data: List[Dict]) -> tuple:
    """Transform raw API data into relational structure"""
    
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
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dated': artifact.get('dated'),
            'period': artifact.get('period'),
            'url': artifact.get('url'),
            'creditline': artifact.get('creditline'),
            'division': artifact.get('division'),
            'imagepermissionlevel': artifact.get('imagepermissionlevel', 0)
        }
        metadata_records.append(metadata)
        
        # Extract media information
        if artifact.get('primaryimageurl'):
            media = {
                'artifact_id': artifact.get('id'),
                'baseimageurl': artifact.get('primaryimageurl'),
                'format': 'image',
                'iiifbaseuri': artifact.get('iiifbaseuri')
            }
            media_records.append(media)
        
        # Extract color information
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_record = {
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'hue': color.get('hue'),
                    'percent': color.get('percent', 0)
                }
                color_records.append(color_record)
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )

def load_to_database(metadata_df, media_df, colors_df):
    """Load transformed data into MySQL database"""
    
    config = {
        'host': os.getenv('DB_HOST'),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    
    conn = mysql.connector.connect(**config)
    cursor = conn.cursor()
    
    # Insert metadata
    for _, row in metadata_df.iterrows():
        query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, department, 
             technique, medium, dated, period, url, creditline, 
             division, imagepermissionlevel)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.execute(query, tuple(row))
    
    # Insert media
    for _, row in media_df.iterrows():
        query = """
            INSERT INTO artifactmedia 
            (artifact_id, baseimageurl, format, iiifbaseuri)
            VALUES (%s, %s, %s, %s)
        """
        cursor.execute(query, tuple(row))
    
    # Insert colors
    for _, row in colors_df.iterrows():
        query = """
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """
        cursor.execute(query, tuple(row))
    
    conn.commit()
    cursor.close()
    conn.close()
```

### 3. SQL Analytics Queries

```python
# Sample analytical queries

ANALYTICS_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "top_departments": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "color_distribution": """
        SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "artifacts_with_images": """
        SELECT 
            CASE WHEN m.artifact_id IS NOT NULL THEN 'With Images' 
                 ELSE 'No Images' END as image_status,
            COUNT(*) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY image_status
    """,
    
    "classification_analysis": """
        SELECT classification, COUNT(*) as count,
               COUNT(DISTINCT culture) as cultures
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 15
    """
}

def execute_query(query_name: str) -> pd.DataFrame:
    """Execute analytical query and return results"""
    config = {
        'host': os.getenv('DB_HOST'),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    
    conn = mysql.connector.connect(**config)
    query = ANALYTICS_QUERIES.get(query_name)
    
    if query:
        df = pd.read_sql(query, conn)
        conn.close()
        return df
    else:
        conn.close()
        raise ValueError(f"Query '{query_name}' not found")
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for ETL operations
    with st.sidebar:
        st.header("Data Pipeline")
        
        num_records = st.number_input("Number of Records", 
                                      min_value=10, 
                                      max_value=1000, 
                                      value=100)
        
        if st.button("Run ETL Pipeline"):
            with st.spinner("Fetching data from API..."):
                raw_data = fetch_artifacts(num_records)
            
            with st.spinner("Transforming data..."):
                metadata_df, media_df, colors_df = transform_artifacts(raw_data)
            
            with st.spinner("Loading to database..."):
                load_to_database(metadata_df, media_df, colors_df)
            
            st.success(f"✅ Loaded {len(metadata_df)} artifacts!")
    
    # Main dashboard area
    st.header("📊 Analytics")
    
    query_option = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Run Query"):
        df = execute_query(query_option)
        
        st.subheader("Query Results")
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(df, 
                        x=df.columns[0], 
                        y=df.columns[1],
                        title=f"Visualization: {query_option}")
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Batch Processing with Pagination

```python
def fetch_with_pagination(total_records=1000, batch_size=100):
    """Fetch large datasets in batches"""
    all_records = []
    
    for offset in range(0, total_records, batch_size):
        params = {
            'apikey': os.getenv('HARVARD_API_KEY'),
            'size': batch_size,
            'page': (offset // batch_size) + 1
        }
        
        response = requests.get(
            "https://api.harvardartmuseums.org/object",
            params=params
        )
        
        if response.status_code == 200:
            data = response.json()
            all_records.extend(data['records'])
        else:
            break
    
    return all_records
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_etl_pipeline(num_records):
    """ETL pipeline with error handling"""
    try:
        logger.info(f"Starting ETL for {num_records} records")
        
        # Extract
        raw_data = fetch_artifacts(num_records)
        logger.info(f"Extracted {len(raw_data)} records")
        
        # Transform
        metadata_df, media_df, colors_df = transform_artifacts(raw_data)
        logger.info("Transformation complete")
        
        # Load
        load_to_database(metadata_df, media_df, colors_df)
        logger.info("Data loaded successfully")
        
        return True
        
    except requests.RequestException as e:
        logger.error(f"API request failed: {e}")
        return False
    except mysql.connector.Error as e:
        logger.error(f"Database error: {e}")
        return False
    except Exception as e:
        logger.error(f"Unexpected error: {e}")
        return False
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Troubleshooting

**API Rate Limiting**: The Harvard Art Museums API has rate limits. Add delays between requests:

```python
import time

def fetch_with_rate_limit(num_records, delay=0.5):
    artifacts = []
    page = 1
    
    while len(artifacts) < num_records:
        # Make request
        response = requests.get(base_url, params=params)
        artifacts.extend(response.json()['records'])
        
        time.sleep(delay)  # Wait between requests
        page += 1
    
    return artifacts
```

**Database Connection Issues**: Verify credentials and network access:

```python
def test_db_connection():
    """Test database connectivity"""
    try:
        conn = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        conn.close()
        print("✅ Database connection successful")
        return True
    except mysql.connector.Error as e:
        print(f"❌ Database connection failed: {e}")
        return False
```

**Missing Data Fields**: Handle null values in API responses:

```python
def safe_get(data, key, default=None):
    """Safely extract nested data"""
    try:
        return data.get(key, default)
    except (KeyError, AttributeError):
        return default
```
