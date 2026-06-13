```markdown
---
name: harvard-artifacts-data-pipeline
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I extract data from Harvard Art Museums API
  - build an ETL pipeline for museum artifacts
  - create analytics dashboard with Streamlit and SQL
  - set up Harvard artifacts data engineering project
  - query and visualize museum collection data
  - implement batch data pipeline for art collections
  - connect Python to TiDB for museum data
  - analyze artifact metadata with SQL queries
---

# Harvard Artifacts Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards.

## What This Project Does

The Harvard Artifacts Collection Data Engineering & Analytics App provides:

- **API Integration**: Fetch artifact metadata, media, and color data from Harvard Art Museums API
- **ETL Pipeline**: Extract, transform, and load nested JSON into relational SQL tables
- **SQL Analytics**: Execute predefined analytical queries on structured artifact data
- **Interactive Visualization**: Display query results in Streamlit with Plotly charts
- **Database Design**: Properly normalized tables with foreign key relationships

**Architecture Flow**: API → ETL → SQL (MySQL/TiDB) → Analytics → Streamlit Dashboard

## Installation

### Prerequisites

```bash
# Python 3.8+
python --version

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

### API Key Setup

1. Get your Harvard Art Museums API key: https://www.harvardartmuseums.org/collections/api
2. Create environment configuration:

```bash
# .env file
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_PORT=4000
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Setup

```sql
-- Create database
CREATE DATABASE harvard_artifacts;

-- Artifact metadata table
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
    period VARCHAR(200)
);

-- Artifact media table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Key Components and Usage

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
    url = f"https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    
    if response.status_code == 200:
        data = response.json()
        return data['records'], data['info']
    else:
        raise Exception(f"API Error: {response.status_code}")

# Fetch first page
artifacts, info = fetch_artifacts(page=1, size=100)
print(f"Total artifacts: {info['totalrecords']}")
print(f"Fetched: {len(artifacts)}")
```

### 2. ETL Pipeline Implementation

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

def transform_artifacts(raw_data: List[Dict]) -> tuple:
    """Transform nested JSON into relational dataframes"""
    
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata_records.append({
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
            'period': artifact.get('period')
        })
        
        # Extract media information
        if artifact.get('primaryimageurl'):
            media_records.append({
                'artifact_id': artifact.get('id'),
                'baseimageurl': artifact.get('baseimageurl'),
                'primaryimageurl': artifact.get('primaryimageurl'),
                'iiifbaseuri': artifact.get('images', [{}])[0].get('iiifbaseuri') if artifact.get('images') else None
            })
        
        # Extract color data
        colors = artifact.get('colors', [])
        for color in colors:
            color_records.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    df_metadata = pd.DataFrame(metadata_records)
    df_media = pd.DataFrame(media_records)
    df_colors = pd.DataFrame(color_records)
    
    return df_metadata, df_media, df_colors
```

### 3. Database Loading

```python
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
    """Batch insert dataframes into SQL database"""
    
    conn = get_db_connection()
    cursor = conn.cursor()
    
    # Insert metadata
    metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, dated, division, medium, technique, period)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    cursor.executemany(metadata_query, df_metadata.values.tolist())
    
    # Insert media
    media_query = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, primaryimageurl, iiifbaseuri)
        VALUES (%s, %s, %s, %s)
    """
    cursor.executemany(media_query, df_media.values.tolist())
    
    # Insert colors
    colors_query = """
        INSERT INTO artifactcolors 
        (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
    """
    cursor.executemany(colors_query, df_colors.values.tolist())
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print(f"Loaded {len(df_metadata)} artifacts to database")
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Artifacts Analytics Dashboard")
    
    # Sidebar for ETL operations
    with st.sidebar:
        st.header("Data Collection")
        
        num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=10, value=1)
        
        if st.button("Run ETL Pipeline"):
            with st.spinner("Fetching data from API..."):
                all_artifacts = []
                for page in range(1, num_pages + 1):
                    artifacts, _ = fetch_artifacts(page=page, size=100)
                    all_artifacts.extend(artifacts)
                
                st.success(f"Fetched {len(all_artifacts)} artifacts")
                
                # Transform
                df_metadata, df_media, df_colors = transform_artifacts(all_artifacts)
                
                # Load
                load_to_database(df_metadata, df_media, df_colors)
                st.success("Data loaded to database!")
    
    # Main content - Analytics
    st.header("SQL Analytics")
    
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
        "Color Distribution": """
            SELECT color, COUNT(*) as count 
            FROM artifactcolors 
            GROUP BY color 
            ORDER BY count DESC 
            LIMIT 15
        """,
        "Department Analysis": """
            SELECT department, COUNT(*) as artifact_count,
                   COUNT(DISTINCT am.artifact_id) as with_images
            FROM artifactmetadata a
            LEFT JOIN artifactmedia am ON a.id = am.artifact_id
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY artifact_count DESC
        """
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
    
    if st.button("Run Query"):
        conn = get_db_connection()
        df_result = pd.read_sql(query_options[selected_query], conn)
        conn.close()
        
        st.dataframe(df_result)
        
        # Visualize
        if len(df_result.columns) >= 2:
            fig = px.bar(df_result, x=df_result.columns[0], y=df_result.columns[1],
                        title=selected_query)
            st.plotly_chart(fig)
```

## Common Analytical Queries

### Top Artifacts by Classification

```sql
SELECT classification, COUNT(*) as count
FROM artifactmetadata
WHERE classification IS NOT NULL
GROUP BY classification
ORDER BY count DESC
LIMIT 10;
```

### Color Dominance Analysis

```sql
SELECT c.color, AVG(c.percent) as avg_percent, COUNT(*) as occurrences
FROM artifactcolors c
GROUP BY c.color
HAVING occurrences > 10
ORDER BY avg_percent DESC;
```

### Media Availability Rate

```sql
SELECT 
    COUNT(DISTINCT a.id) as total_artifacts,
    COUNT(DISTINCT m.artifact_id) as with_images,
    ROUND(COUNT(DISTINCT m.artifact_id) * 100.0 / COUNT(DISTINCT a.id), 2) as image_coverage_pct
FROM artifactmetadata a
LEFT JOIN artifactmedia m ON a.id = m.artifact_id;
```

### Artifacts by Period and Culture

```sql
SELECT period, culture, COUNT(*) as count
FROM artifactmetadata
WHERE period IS NOT NULL AND culture IS NOT NULL
GROUP BY period, culture
ORDER BY count DESC
LIMIT 20;
```

## Configuration Patterns

### Rate Limiting API Calls

```python
import time

def fetch_with_rate_limit(pages: int, delay: float = 0.5):
    """Fetch multiple pages with rate limiting"""
    all_data = []
    
    for page in range(1, pages + 1):
        artifacts, info = fetch_artifacts(page=page)
        all_data.extend(artifacts)
        
        print(f"Page {page}/{pages}: {len(artifacts)} artifacts")
        time.sleep(delay)  # Respect API rate limits
    
    return all_data
```

### Environment Configuration

```python
from dataclasses import dataclass
from dotenv import load_dotenv
import os

@dataclass
class Config:
    """Application configuration"""
    api_key: str
    db_host: str
    db_port: int
    db_user: str
    db_password: str
    db_name: str
    
    @classmethod
    def from_env(cls):
        load_dotenv()
        return cls(
            api_key=os.getenv('HARVARD_API_KEY'),
            db_host=os.getenv('DB_HOST'),
            db_port=int(os.getenv('DB_PORT', 3306)),
            db_user=os.getenv('DB_USER'),
            db_password=os.getenv('DB_PASSWORD'),
            db_name=os.getenv('DB_NAME', 'harvard_artifacts')
        )

# Usage
config = Config.from_env()
```

## Troubleshooting

### API Authentication Errors

```python
# Verify API key is loaded
import os
from dotenv import load_dotenv

load_dotenv()
api_key = os.getenv('HARVARD_API_KEY')

if not api_key:
    raise ValueError("HARVARD_API_KEY not found in environment")

print(f"API Key loaded: {api_key[:10]}...")
```

### Database Connection Issues

```python
# Test database connection
try:
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD')
    )
    print("Database connection successful")
    conn.close()
except mysql.connector.Error as err:
    print(f"Database error: {err}")
```

### Handling Missing Data

```python
# Safe data extraction with defaults
def safe_extract(artifact: Dict, field: str, default=None):
    """Safely extract field with default value"""
    return artifact.get(field, default)

# Apply to transformation
metadata_records.append({
    'id': artifact.get('id'),
    'title': safe_extract(artifact, 'title', 'Untitled'),
    'culture': safe_extract(artifact, 'culture', 'Unknown'),
    'century': safe_extract(artifact, 'century', 'Unknown')
})
```

### Pagination Edge Cases

```python
def fetch_all_artifacts(max_pages: int = None):
    """Fetch all available artifacts with pagination"""
    all_artifacts = []
    page = 1
    
    while True:
        artifacts, info = fetch_artifacts(page=page)
        
        if not artifacts:
            break
        
        all_artifacts.extend(artifacts)
        
        if max_pages and page >= max_pages:
            break
            
        if page >= info['pages']:
            break
        
        page += 1
    
    return all_artifacts
```

## Best Practices

1. **Always use environment variables** for API keys and database credentials
2. **Implement rate limiting** when fetching data to respect API quotas
3. **Use batch inserts** for better database performance
4. **Handle NULL values** explicitly in SQL queries and transformations
5. **Create indexes** on foreign keys for faster joins
6. **Log ETL operations** for debugging and monitoring
7. **Validate data** before insertion to prevent constraint violations

This skill provides comprehensive guidance for building data engineering pipelines with the Harvard Art Museums API, suitable for analytics, visualization, and educational purposes.
```
