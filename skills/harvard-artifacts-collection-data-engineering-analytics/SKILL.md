---
name: harvard-artifacts-collection-data-engineering-analytics
description: End-to-end data engineering and analytics application using Harvard Art Museums API with ETL pipelines, SQL analytics, and Streamlit visualization
triggers:
  - build a data pipeline for museum artifacts
  - create an ETL workflow for Harvard art data
  - set up artifact collection analytics with Streamlit
  - integrate Harvard Art Museums API with SQL database
  - build a museum data engineering application
  - create interactive dashboards for art collection data
  - implement ETL pipeline for cultural heritage data
  - analyze museum artifacts with SQL and visualization
---

# Harvard Artifacts Collection Data Engineering Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project demonstrates a complete data engineering workflow using the Harvard Art Museums API. It implements:
- **Data Collection**: Paginated API requests to Harvard Art Museums
- **ETL Pipeline**: Extract, Transform, Load operations on artifact data
- **SQL Storage**: Relational database design for artifacts, media, and color data
- **Analytics**: Predefined SQL queries for insights
- **Visualization**: Interactive Streamlit dashboards with Plotly charts

The application showcases real-world patterns for API integration, data transformation, database design, and analytics visualization.

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages typically include:
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
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Database Setup

The project uses three main tables with a relational structure:

```sql
-- Artifacts metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    medium VARCHAR(500),
    technique VARCHAR(500),
    period VARCHAR(200),
    provenance TEXT,
    description TEXT,
    copyright TEXT,
    creditline TEXT,
    totalpageviews INT,
    totaluniquepageviews INT
);

-- Artifact media/images table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    imagepermissionlevel INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact color analysis table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(100),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# The app will open in your browser at http://localhost:8501
```

## Key Components

### 1. API Integration

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
        """Fetch artifacts with pagination support"""
        params = {
            'apikey': self.api_key,
            'page': page,
            'size': size
        }
        response = requests.get(self.base_url, params=params)
        response.raise_for_status()
        return response.json()
    
    def collect_batch(self, num_pages=5):
        """Collect multiple pages of data"""
        all_artifacts = []
        for page in range(1, num_pages + 1):
            data = self.fetch_artifacts(page=page)
            all_artifacts.extend(data.get('records', []))
        return all_artifacts
```

### 2. ETL Pipeline

```python
import pandas as pd
from typing import List, Dict, Tuple

class ArtifactETL:
    def __init__(self, raw_data: List[Dict]):
        self.raw_data = raw_data
    
    def extract_metadata(self) -> pd.DataFrame:
        """Extract core artifact metadata"""
        metadata = []
        for artifact in self.raw_data:
            metadata.append({
                'id': artifact.get('id'),
                'title': artifact.get('title'),
                'culture': artifact.get('culture'),
                'century': artifact.get('century'),
                'dated': artifact.get('dated'),
                'classification': artifact.get('classification'),
                'department': artifact.get('department'),
                'division': artifact.get('division'),
                'medium': artifact.get('medium'),
                'technique': artifact.get('technique'),
                'period': artifact.get('period'),
                'provenance': artifact.get('provenance'),
                'description': artifact.get('description'),
                'copyright': artifact.get('copyright'),
                'creditline': artifact.get('creditline'),
                'totalpageviews': artifact.get('totalpageviews', 0),
                'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
            })
        return pd.DataFrame(metadata)
    
    def extract_media(self) -> pd.DataFrame:
        """Extract media/image information"""
        media = []
        for artifact in self.raw_data:
            artifact_id = artifact.get('id')
            images = artifact.get('images', [])
            if images:
                primary_image = images[0] if images else {}
                media.append({
                    'artifact_id': artifact_id,
                    'baseimageurl': primary_image.get('baseimageurl'),
                    'primaryimageurl': artifact.get('primaryimageurl'),
                    'imagepermissionlevel': artifact.get('imagepermissionlevel', 0)
                })
        return pd.DataFrame(media)
    
    def extract_colors(self) -> pd.DataFrame:
        """Extract color analysis data"""
        colors = []
        for artifact in self.raw_data:
            artifact_id = artifact.get('id')
            color_data = artifact.get('colors', [])
            for color_entry in color_data:
                colors.append({
                    'artifact_id': artifact_id,
                    'color': color_entry.get('color'),
                    'spectrum': color_entry.get('spectrum'),
                    'hue': color_entry.get('hue'),
                    'percent': color_entry.get('percent', 0.0)
                })
        return pd.DataFrame(colors)
    
    def transform(self) -> Tuple[pd.DataFrame, pd.DataFrame, pd.DataFrame]:
        """Complete ETL transformation"""
        metadata_df = self.extract_metadata()
        media_df = self.extract_media()
        colors_df = self.extract_colors()
        
        # Clean and transform
        metadata_df.fillna('Unknown', inplace=True)
        media_df.fillna('', inplace=True)
        colors_df.fillna(0, inplace=True)
        
        return metadata_df, media_df, colors_df
```

### 3. Database Loader

```python
import mysql.connector
from mysql.connector import Error

class DatabaseLoader:
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
            print(f"Database connection error: {e}")
            return None
    
    def load_metadata(self, df: pd.DataFrame):
        """Batch insert artifact metadata"""
        cursor = self.connection.cursor()
        insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, dated, classification, department, 
         division, medium, technique, period, provenance, description, 
         copyright, creditline, totalpageviews, totaluniquepageviews)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        data = [tuple(row) for row in df.values]
        cursor.executemany(insert_query, data)
        self.connection.commit()
        cursor.close()
    
    def load_media(self, df: pd.DataFrame):
        """Batch insert media data"""
        cursor = self.connection.cursor()
        insert_query = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, primaryimageurl, imagepermissionlevel)
        VALUES (%s, %s, %s, %s)
        """
        data = [tuple(row) for row in df.values]
        cursor.executemany(insert_query, data)
        self.connection.commit()
        cursor.close()
    
    def load_colors(self, df: pd.DataFrame):
        """Batch insert color data"""
        cursor = self.connection.cursor()
        insert_query = """
        INSERT INTO artifactcolors 
        (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
        """
        data = [tuple(row) for row in df.values]
        cursor.executemany(insert_query, data)
        self.connection.commit()
        cursor.close()
```

### 4. Analytics Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture != 'Unknown'
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
    
    "popular_artifacts": """
        SELECT title, culture, totalpageviews
        FROM artifactmetadata
        WHERE totalpageviews > 0
        ORDER BY totalpageviews DESC
        LIMIT 20
    """,
    
    "color_distribution": """
        SELECT spectrum, COUNT(*) as count, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY spectrum
        ORDER BY count DESC
    """,
    
    "artifacts_with_images": """
        SELECT 
            a.department,
            COUNT(DISTINCT a.id) as total_artifacts,
            COUNT(DISTINCT m.artifact_id) as artifacts_with_images,
            ROUND(COUNT(DISTINCT m.artifact_id) * 100.0 / COUNT(DISTINCT a.id), 2) as percentage
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY a.department
        ORDER BY percentage DESC
    """
}
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def run_dashboard():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    query_selection = st.sidebar.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    # Database connection
    db = DatabaseLoader()
    
    # Execute query
    if st.button("Run Analysis"):
        query = ANALYTICS_QUERIES[query_selection]
        cursor = db.connection.cursor()
        cursor.execute(query)
        results = cursor.fetchall()
        columns = [desc[0] for desc in cursor.description]
        
        # Display results
        df_results = pd.DataFrame(results, columns=columns)
        st.dataframe(df_results)
        
        # Visualization
        if len(columns) >= 2:
            fig = px.bar(
                df_results,
                x=columns[0],
                y=columns[1],
                title=f"Analysis: {query_selection}"
            )
            st.plotly_chart(fig, use_container_width=True)
        
        cursor.close()

if __name__ == "__main__":
    run_dashboard()
```

## Common Patterns

### Complete ETL Workflow

```python
# Full pipeline execution
def run_etl_pipeline(num_pages=5):
    # 1. Extract from API
    client = HarvardAPIClient()
    raw_data = client.collect_batch(num_pages=num_pages)
    
    # 2. Transform
    etl = ArtifactETL(raw_data)
    metadata_df, media_df, colors_df = etl.transform()
    
    # 3. Load to database
    db = DatabaseLoader()
    db.load_metadata(metadata_df)
    db.load_media(media_df)
    db.load_colors(colors_df)
    
    print(f"Loaded {len(metadata_df)} artifacts successfully")
```

### Rate Limiting for API Calls

```python
import time

def fetch_with_rate_limit(client, num_pages, delay=1):
    """Fetch data with rate limiting"""
    all_data = []
    for page in range(1, num_pages + 1):
        data = client.fetch_artifacts(page=page)
        all_data.extend(data.get('records', []))
        time.sleep(delay)  # Respect API rate limits
    return all_data
```

## Troubleshooting

**API Key Issues**: Ensure `HARVARD_API_KEY` is set in `.env` and valid. Get keys at: https://www.harvardartmuseums.org/collections/api

**Database Connection Errors**: Verify all `DB_*` environment variables are correct. Test connection separately before running ETL.

**Missing Data in Queries**: Some artifacts may not have all fields populated. Use `COALESCE` or `IS NOT NULL` filters in SQL.

**Streamlit Port Conflicts**: If port 8501 is in use, specify a different port:
```bash
streamlit run app.py --server.port 8502
```

**Memory Issues with Large Batches**: Process data in smaller chunks or use pagination more effectively:
```python
# Process in batches of 100 records
for i in range(0, len(raw_data), 100):
    batch = raw_data[i:i+100]
    # Process batch
```
