---
name: harvard-artifacts-data-engineering-app
description: End-to-end data engineering and analytics application for Harvard Art Museums API with ETL pipelines, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data engineering app with Streamlit and SQL
  - fetch and analyze Harvard artifacts collection data
  - set up analytics dashboard for museum API data
  - implement Harvard Art Museums data pipeline
  - build interactive visualization for art collection data
  - query and visualize Harvard museum artifacts
  - create end-to-end data workflow with API and database
---

# Harvard Artifacts Data Engineering App

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is an end-to-end data engineering and analytics application that:
- Extracts artifact data from the Harvard Art Museums API
- Transforms nested JSON into relational database structures
- Loads data into MySQL/TiDB Cloud databases
- Executes analytical SQL queries
- Visualizes results through interactive Streamlit dashboards

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

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

### Database Setup

Create the required database schema:

```sql
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
USE harvard_artifacts;

-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    technique VARCHAR(500),
    dated VARCHAR(100),
    description TEXT,
    provenance TEXT,
    copyright TEXT,
    url VARCHAR(500),
    last_updated DATETIME
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(500),
    base_url VARCHAR(500),
    alt_text TEXT,
    caption TEXT,
    format VARCHAR(50),
    height INT,
    width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(7),
    color_name VARCHAR(100),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    saturation DECIMAL(5,2),
    brightness DECIMAL(5,2),
    percentage DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Access the app at http://localhost:8501
```

## Key Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only fetch artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Example: Fetch multiple pages
def collect_artifacts(total_pages=5):
    all_artifacts = []
    
    for page in range(1, total_pages + 1):
        data = fetch_artifacts(page=page, size=100)
        all_artifacts.extend(data.get('records', []))
        print(f"Collected page {page}: {len(data.get('records', []))} artifacts")
    
    return all_artifacts
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from datetime import datetime

class ArtifactETL:
    def __init__(self, db_config):
        self.db_config = db_config
        self.connection = None
    
    def connect_db(self):
        """Establish database connection"""
        self.connection = mysql.connector.connect(
            host=self.db_config['host'],
            port=self.db_config['port'],
            user=self.db_config['user'],
            password=self.db_config['password'],
            database=self.db_config['database']
        )
        return self.connection.cursor()
    
    def extract_metadata(self, artifact):
        """Extract metadata from artifact JSON"""
        return {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'technique': artifact.get('technique'),
            'dated': artifact.get('dated'),
            'description': artifact.get('description'),
            'provenance': artifact.get('provenance'),
            'copyright': artifact.get('copyright'),
            'url': artifact.get('url'),
            'last_updated': datetime.now()
        }
    
    def extract_media(self, artifact):
        """Extract media/image data from artifact"""
        media_list = []
        images = artifact.get('images', [])
        
        for img in images:
            media_list.append({
                'artifact_id': artifact.get('id'),
                'image_url': img.get('iiifbaseuri'),
                'base_url': img.get('baseimageurl'),
                'alt_text': img.get('alttext'),
                'caption': img.get('caption'),
                'format': img.get('format'),
                'height': img.get('height'),
                'width': img.get('width')
            })
        
        return media_list
    
    def extract_colors(self, artifact):
        """Extract color data from artifact"""
        color_list = []
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_list.append({
                'artifact_id': artifact.get('id'),
                'color_hex': color.get('hex'),
                'color_name': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'saturation': color.get('saturation'),
                'brightness': color.get('brightness'),
                'percentage': color.get('percent')
            })
        
        return color_list
    
    def load_metadata(self, metadata_list):
        """Batch insert metadata into database"""
        cursor = self.connect_db()
        
        insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, division, 
         technique, dated, description, provenance, copyright, url, last_updated)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture), last_updated=VALUES(last_updated)
        """
        
        cursor.executemany(insert_query, [
            (m['id'], m['title'], m['culture'], m['century'], m['classification'],
             m['department'], m['division'], m['technique'], m['dated'],
             m['description'], m['provenance'], m['copyright'], m['url'], m['last_updated'])
            for m in metadata_list
        ])
        
        self.connection.commit()
        cursor.close()
        print(f"Loaded {len(metadata_list)} metadata records")
    
    def run_etl(self, artifacts):
        """Execute full ETL pipeline"""
        metadata_list = []
        media_list = []
        color_list = []
        
        # Extract
        for artifact in artifacts:
            metadata_list.append(self.extract_metadata(artifact))
            media_list.extend(self.extract_media(artifact))
            color_list.extend(self.extract_colors(artifact))
        
        # Load
        self.load_metadata(metadata_list)
        # Similar methods for load_media() and load_colors()
        
        print(f"ETL Complete: {len(metadata_list)} artifacts processed")
```

### 3. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    with st.sidebar:
        st.header("Configuration")
        api_key = st.text_input("Harvard API Key", type="password", 
                                value=os.getenv('HARVARD_API_KEY', ''))
        
        st.header("Data Collection")
        num_pages = st.slider("Pages to collect", 1, 10, 5)
        
        if st.button("🔄 Fetch & Load Data"):
            with st.spinner("Collecting artifacts..."):
                artifacts = collect_artifacts(total_pages=num_pages)
                
                db_config = {
                    'host': os.getenv('DB_HOST'),
                    'port': int(os.getenv('DB_PORT', 3306)),
                    'user': os.getenv('DB_USER'),
                    'password': os.getenv('DB_PASSWORD'),
                    'database': os.getenv('DB_NAME')
                }
                
                etl = ArtifactETL(db_config)
                etl.run_etl(artifacts)
                
                st.success(f"✅ Loaded {len(artifacts)} artifacts")
    
    # Analytics Queries
    st.header("📊 Analytics Queries")
    
    query_options = {
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
        "Color Spectrum Analysis": """
            SELECT spectrum, COUNT(*) as count, AVG(percentage) as avg_percentage
            FROM artifactcolors 
            GROUP BY spectrum 
            ORDER BY count DESC
        """
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
    
    if st.button("Run Query"):
        db_config = {
            'host': os.getenv('DB_HOST'),
            'port': int(os.getenv('DB_PORT', 3306)),
            'user': os.getenv('DB_USER'),
            'password': os.getenv('DB_PASSWORD'),
            'database': os.getenv('DB_NAME')
        }
        
        connection = mysql.connector.connect(**db_config)
        df = pd.read_sql(query_options[selected_query], connection)
        connection.close()
        
        # Display results
        col1, col2 = st.columns([1, 1])
        
        with col1:
            st.subheader("Query Results")
            st.dataframe(df)
        
        with col2:
            st.subheader("Visualization")
            if len(df.columns) >= 2:
                fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                           title=selected_query)
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common SQL Analytics Queries

```sql
-- Top 10 cultures by artifact count
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10;

-- Artifacts with media by classification
SELECT am.classification, COUNT(DISTINCT am.id) as artifacts_with_media
FROM artifactmetadata am
JOIN artifactmedia med ON am.id = med.artifact_id
GROUP BY am.classification
ORDER BY artifacts_with_media DESC;

-- Average color brightness by century
SELECT am.century, AVG(ac.brightness) as avg_brightness
FROM artifactmetadata am
JOIN artifactcolors ac ON am.id = ac.artifact_id
WHERE am.century IS NOT NULL
GROUP BY am.century
ORDER BY avg_brightness DESC;

-- Most common color spectrums
SELECT spectrum, COUNT(*) as occurrences, AVG(percentage) as avg_coverage
FROM artifactcolors
GROUP BY spectrum
ORDER BY occurrences DESC;

-- Department-wise media availability
SELECT department, 
       COUNT(DISTINCT am.id) as total_artifacts,
       COUNT(DISTINCT med.artifact_id) as artifacts_with_images,
       ROUND(COUNT(DISTINCT med.artifact_id) * 100.0 / COUNT(DISTINCT am.id), 2) as image_coverage_pct
FROM artifactmetadata am
LEFT JOIN artifactmedia med ON am.id = med.artifact_id
GROUP BY department
ORDER BY image_coverage_pct DESC;
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
                print(f"Retry {attempt + 1} after {wait_time}s: {e}")
                time.sleep(wait_time)
            else:
                raise
```

### Database Connection Issues

```python
def test_db_connection():
    """Test database connectivity"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        print("✅ Database connection successful")
        connection.close()
        return True
    except Exception as e:
        print(f"❌ Database connection failed: {e}")
        return False
```

### Handling Missing Data

```python
def safe_get(dictionary, key, default=None):
    """Safely extract nested dictionary values"""
    try:
        value = dictionary.get(key, default)
        return value if value else default
    except:
        return default

# Usage in extraction
metadata = {
    'id': safe_get(artifact, 'id'),
    'title': safe_get(artifact, 'title', 'Untitled'),
    'culture': safe_get(artifact, 'culture', 'Unknown')
}
```

## Best Practices

1. **Batch Processing**: Process artifacts in batches to optimize database inserts
2. **Error Handling**: Wrap API calls and database operations in try-except blocks
3. **Data Validation**: Validate API responses before loading into database
4. **Incremental Updates**: Use `last_updated` timestamp for incremental ETL
5. **Connection Pooling**: Reuse database connections for better performance
