---
name: harvard-artifacts-data-pipeline
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API data with Python, SQL, and Streamlit
triggers:
  - build a data pipeline with harvard art museums api
  - create etl workflow for museum artifacts
  - analyze harvard art collection data
  - set up streamlit dashboard for art museum data
  - extract and transform harvard api data
  - query harvard artifacts with sql analytics
  - visualize museum collection metadata
  - build end-to-end data engineering pipeline
---

# Harvard Artifacts Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Harvard Artifacts Collection Data Engineering & Analytics App is an end-to-end data pipeline that demonstrates real-world ETL patterns using the Harvard Art Museums API. It extracts artifact metadata, transforms nested JSON into relational tables, loads data into SQL databases, and provides interactive analytics dashboards using Streamlit and Plotly.

**Architecture Flow**: API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

### Required Dependencies

```python
# requirements.txt
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

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### API Key Setup

1. Register at [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Obtain your API key
3. Store in environment variable: `HARVARD_API_KEY`

## Database Schema

The pipeline creates three main tables:

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    objectnumber VARCHAR(100),
    dated VARCHAR(200),
    period VARCHAR(200)
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percent FLOAT,
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

class HarvardAPIClient:
    def __init__(self):
        self.api_key = os.getenv('HARVARD_API_KEY')
        self.base_url = "https://api.harvardartmuseums.org"
    
    def fetch_objects(self, page=1, size=100):
        """Fetch artifacts with pagination"""
        url = f"{self.base_url}/object"
        params = {
            'apikey': self.api_key,
            'page': page,
            'size': size
        }
        response = requests.get(url, params=params)
        response.raise_for_status()
        return response.json()
    
    def extract_batch(self, num_pages=5):
        """Extract multiple pages of data"""
        all_records = []
        for page in range(1, num_pages + 1):
            data = self.fetch_objects(page=page)
            all_records.extend(data.get('records', []))
        return all_records
```

### 2. ETL Transformation

```python
import pandas as pd

class ArtifactTransformer:
    @staticmethod
    def transform_metadata(raw_data):
        """Transform raw API data to metadata dataframe"""
        metadata = []
        for record in raw_data:
            metadata.append({
                'id': record.get('id'),
                'title': record.get('title'),
                'culture': record.get('culture'),
                'century': record.get('century'),
                'classification': record.get('classification'),
                'department': record.get('department'),
                'objectnumber': record.get('objectnumber'),
                'dated': record.get('dated'),
                'period': record.get('period')
            })
        return pd.DataFrame(metadata)
    
    @staticmethod
    def transform_media(raw_data):
        """Extract media/image information"""
        media = []
        for record in raw_data:
            artifact_id = record.get('id')
            media.append({
                'artifact_id': artifact_id,
                'baseimageurl': record.get('baseimageurl'),
                'primaryimageurl': record.get('primaryimageurl')
            })
        return pd.DataFrame(media)
    
    @staticmethod
    def transform_colors(raw_data):
        """Extract color palette data"""
        colors = []
        for record in raw_data:
            artifact_id = record.get('id')
            color_data = record.get('colors', [])
            for color in color_data:
                colors.append({
                    'artifact_id': artifact_id,
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'percent': color.get('percent')
                })
        return pd.DataFrame(colors)
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

class DatabaseLoader:
    def __init__(self):
        self.config = {
            'host': os.getenv('DB_HOST'),
            'port': os.getenv('DB_PORT', 3306),
            'user': os.getenv('DB_USER'),
            'password': os.getenv('DB_PASSWORD'),
            'database': os.getenv('DB_NAME')
        }
    
    def get_connection(self):
        """Create database connection"""
        return mysql.connector.connect(**self.config)
    
    def load_metadata(self, df):
        """Batch insert metadata"""
        conn = self.get_connection()
        cursor = conn.cursor()
        
        query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, objectnumber, dated, period)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        
        data = [tuple(row) for row in df.values]
        cursor.executemany(query, data)
        conn.commit()
        cursor.close()
        conn.close()
    
    def load_media(self, df):
        """Batch insert media data"""
        conn = self.get_connection()
        cursor = conn.cursor()
        
        query = """
        INSERT INTO artifactmedia (artifact_id, baseimageurl, primaryimageurl)
        VALUES (%s, %s, %s)
        """
        
        data = [tuple(row) for row in df.values]
        cursor.executemany(query, data)
        conn.commit()
        cursor.close()
        conn.close()
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("API Key", type="password", 
                                      value=os.getenv('HARVARD_API_KEY', ''))
    
    # ETL Controls
    st.header("📥 Data Collection")
    num_pages = st.slider("Number of pages to fetch", 1, 10, 5)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Extracting data..."):
            client = HarvardAPIClient()
            raw_data = client.extract_batch(num_pages=num_pages)
            
        with st.spinner("Transforming data..."):
            transformer = ArtifactTransformer()
            metadata_df = transformer.transform_metadata(raw_data)
            media_df = transformer.transform_media(raw_data)
            colors_df = transformer.transform_colors(raw_data)
            
        with st.spinner("Loading to database..."):
            loader = DatabaseLoader()
            loader.load_metadata(metadata_df)
            loader.load_media(media_df)
            loader.load_colors(colors_df)
            
        st.success(f"✅ Loaded {len(metadata_df)} artifacts successfully!")
    
    # Analytics Section
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
        "Department Distribution": """
            SELECT department, COUNT(*) as count 
            FROM artifactmetadata 
            GROUP BY department 
            ORDER BY count DESC
        """,
        "Color Analysis": """
            SELECT color, COUNT(*) as frequency 
            FROM artifactcolors 
            GROUP BY color 
            ORDER BY frequency DESC 
            LIMIT 15
        """
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        loader = DatabaseLoader()
        conn = loader.get_connection()
        df = pd.read_sql(queries[selected_query], conn)
        conn.close()
        
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                        title=selected_query)
            st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common SQL Analytics Queries

```sql
-- Top 10 cultures with most artifacts
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10;

-- Artifacts with images
SELECT 
    COUNT(*) as total_artifacts,
    SUM(CASE WHEN primaryimageurl IS NOT NULL THEN 1 ELSE 0 END) as with_images,
    ROUND(100.0 * SUM(CASE WHEN primaryimageurl IS NOT NULL THEN 1 ELSE 0 END) / COUNT(*), 2) as image_percentage
FROM artifactmedia;

-- Most common color palettes
SELECT color, spectrum, AVG(percent) as avg_percent
FROM artifactcolors
GROUP BY color, spectrum
ORDER BY avg_percent DESC
LIMIT 20;

-- Artifacts by classification and century
SELECT classification, century, COUNT(*) as count
FROM artifactmetadata
WHERE classification IS NOT NULL AND century IS NOT NULL
GROUP BY classification, century
ORDER BY count DESC;
```

## Advanced Patterns

### Rate Limiting API Requests

```python
import time

class RateLimitedClient(HarvardAPIClient):
    def __init__(self, requests_per_minute=60):
        super().__init__()
        self.delay = 60.0 / requests_per_minute
    
    def fetch_objects(self, page=1, size=100):
        time.sleep(self.delay)
        return super().fetch_objects(page, size)
```

### Incremental ETL

```python
def incremental_etl():
    """Only fetch new artifacts since last run"""
    loader = DatabaseLoader()
    conn = loader.get_connection()
    cursor = conn.cursor()
    
    # Get max ID from database
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    max_id = cursor.fetchone()[0] or 0
    
    # Fetch only newer artifacts
    client = HarvardAPIClient()
    url = f"{client.base_url}/object"
    params = {
        'apikey': client.api_key,
        'q': f'id:>{max_id}',
        'size': 100
    }
    
    response = requests.get(url, params=params)
    new_data = response.json().get('records', [])
    
    # Transform and load
    transformer = ArtifactTransformer()
    metadata_df = transformer.transform_metadata(new_data)
    loader.load_metadata(metadata_df)
```

## Troubleshooting

### API Rate Limit Errors
- Implement exponential backoff
- Reduce batch size
- Add delays between requests

### Database Connection Issues
```python
# Test connection
try:
    loader = DatabaseLoader()
    conn = loader.get_connection()
    print("✅ Database connected")
    conn.close()
except Error as e:
    print(f"❌ Connection failed: {e}")
```

### Missing Data in Queries
- Check for NULL values in WHERE clauses
- Use COALESCE for default values
- Verify foreign key relationships

### Streamlit Performance
```python
# Cache expensive operations
@st.cache_data
def load_analytics_data(query):
    loader = DatabaseLoader()
    conn = loader.get_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

## Best Practices

1. **Always use environment variables** for credentials
2. **Implement pagination** for large datasets
3. **Use batch inserts** for performance
4. **Add error handling** around API calls
5. **Create indexes** on frequently queried columns
6. **Validate data** before insertion
7. **Log ETL runs** for monitoring
