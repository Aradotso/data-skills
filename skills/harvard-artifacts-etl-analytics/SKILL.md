---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I extract data from the Harvard Art Museums API
  - build an ETL pipeline for museum artifacts
  - create analytics dashboard with Streamlit and museum data
  - set up SQL database for Harvard artifacts collection
  - visualize art museum data with Plotly
  - implement pagination for Harvard API requests
  - transform nested JSON artifacts into relational tables
  - query Harvard museum collection with SQL analytics
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. It covers ETL pipeline development, SQL database design, analytical queries, and interactive Streamlit dashboards with Plotly visualizations.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App demonstrates a complete data pipeline:

1. **Extract**: Fetches artifact data from Harvard Art Museums API with pagination
2. **Transform**: Converts nested JSON into normalized relational structures
3. **Load**: Batch inserts data into MySQL/TiDB Cloud databases
4. **Analyze**: Executes 20+ predefined SQL analytical queries
5. **Visualize**: Displays results in interactive Streamlit dashboards

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages:**
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

# Database Configuration
DB_HOST=your_database_host
DB_PORT=4000
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Database Setup

```sql
-- Create database
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    department VARCHAR(200),
    classification VARCHAR(200),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    provenance TEXT,
    description TEXT,
    url VARCHAR(500)
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
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

## Key API Integration Patterns

### Fetching Data with Pagination

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_artifacts=100):
    """Fetch artifacts from Harvard API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    artifacts = []
    page = 1
    size = 100  # Max per page
    
    while len(artifacts) < num_artifacts:
        params = {
            'apikey': api_key,
            'size': min(size, num_artifacts - len(artifacts)),
            'page': page
        }
        
        response = requests.get(base_url, params=params)
        response.raise_for_status()
        
        data = response.json()
        records = data.get('records', [])
        
        if not records:
            break
            
        artifacts.extend(records)
        page += 1
    
    return artifacts[:num_artifacts]
```

### ETL Pipeline Implementation

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

class HarvardArtifactsETL:
    def __init__(self, db_config: Dict):
        self.db_config = db_config
        
    def extract(self, num_records: int = 100) -> List[Dict]:
        """Extract artifacts from API"""
        return fetch_artifacts(num_records)
    
    def transform(self, raw_data: List[Dict]) -> tuple:
        """Transform nested JSON to relational format"""
        metadata_list = []
        media_list = []
        colors_list = []
        
        for artifact in raw_data:
            # Extract metadata
            metadata = {
                'id': artifact.get('id'),
                'title': artifact.get('title', 'Unknown')[:500],
                'culture': artifact.get('culture', 'Unknown')[:200],
                'century': artifact.get('century', 'Unknown')[:100],
                'dated': artifact.get('dated', 'Unknown')[:200],
                'department': artifact.get('department', 'Unknown')[:200],
                'classification': artifact.get('classification', 'Unknown')[:200],
                'medium': artifact.get('medium', 'Unknown')[:500],
                'dimensions': artifact.get('dimensions', 'Unknown')[:500],
                'creditline': artifact.get('creditline', ''),
                'provenance': artifact.get('provenance', ''),
                'description': artifact.get('description', ''),
                'url': artifact.get('url', '')[:500]
            }
            metadata_list.append(metadata)
            
            # Extract media
            if artifact.get('primaryimageurl'):
                media = {
                    'artifact_id': artifact.get('id'),
                    'baseimageurl': artifact.get('baseimageurl', '')[:500],
                    'primaryimageurl': artifact.get('primaryimageurl', '')[:500],
                    'iiifbaseuri': artifact.get('images', [{}])[0].get('iiifbaseuri', '')[:500] if artifact.get('images') else ''
                }
                media_list.append(media)
            
            # Extract colors
            for color_data in artifact.get('colors', []):
                color = {
                    'artifact_id': artifact.get('id'),
                    'color': color_data.get('color', '')[:50],
                    'spectrum': color_data.get('spectrum', '')[:50],
                    'hue': color_data.get('hue', '')[:50],
                    'percent': color_data.get('percent', 0.0)
                }
                colors_list.append(color)
        
        return (
            pd.DataFrame(metadata_list),
            pd.DataFrame(media_list),
            pd.DataFrame(colors_list)
        )
    
    def load(self, metadata_df: pd.DataFrame, media_df: pd.DataFrame, colors_df: pd.DataFrame):
        """Load data into SQL database"""
        conn = mysql.connector.connect(**self.db_config)
        cursor = conn.cursor()
        
        # Load metadata
        for _, row in metadata_df.iterrows():
            sql = """
                INSERT INTO artifactmetadata 
                (id, title, culture, century, dated, department, classification, 
                 medium, dimensions, creditline, provenance, description, url)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE title=VALUES(title)
            """
            cursor.execute(sql, tuple(row))
        
        # Load media
        for _, row in media_df.iterrows():
            sql = """
                INSERT INTO artifactmedia 
                (artifact_id, baseimageurl, primaryimageurl, iiifbaseuri)
                VALUES (%s, %s, %s, %s)
            """
            cursor.execute(sql, tuple(row))
        
        # Load colors
        for _, row in colors_df.iterrows():
            sql = """
                INSERT INTO artifactcolors 
                (artifact_id, color, spectrum, hue, percent)
                VALUES (%s, %s, %s, %s, %s)
            """
            cursor.execute(sql, tuple(row))
        
        conn.commit()
        cursor.close()
        conn.close()
    
    def run_pipeline(self, num_records: int = 100):
        """Execute full ETL pipeline"""
        print("Extracting data...")
        raw_data = self.extract(num_records)
        
        print("Transforming data...")
        metadata_df, media_df, colors_df = self.transform(raw_data)
        
        print("Loading data...")
        self.load(metadata_df, media_df, colors_df)
        
        print(f"ETL complete! Loaded {len(metadata_df)} artifacts.")
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px
import pandas as pd

def main():
    st.title("Harvard Art Museums Analytics Dashboard")
    
    # Database connection
    db_config = {
        'host': os.getenv('DB_HOST'),
        'port': int(os.getenv('DB_PORT', 3306)),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    
    # Sidebar for ETL
    st.sidebar.header("Data Collection")
    num_records = st.sidebar.number_input("Number of artifacts", min_value=10, max_value=1000, value=100)
    
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Running ETL pipeline..."):
            etl = HarvardArtifactsETL(db_config)
            etl.run_pipeline(num_records)
        st.sidebar.success("ETL completed successfully!")
    
    # Analytics queries
    st.header("SQL Analytics")
    
    queries = {
        "Top 10 Cultures by Artifact Count": """
            SELECT culture, COUNT(*) as count 
            FROM artifactmetadata 
            GROUP BY culture 
            ORDER BY count DESC 
            LIMIT 10
        """,
        "Artifacts by Century": """
            SELECT century, COUNT(*) as count 
            FROM artifactmetadata 
            WHERE century IS NOT NULL AND century != 'Unknown'
            GROUP BY century 
            ORDER BY century
        """,
        "Department Distribution": """
            SELECT department, COUNT(*) as count 
            FROM artifactmetadata 
            GROUP BY department 
            ORDER BY count DESC
        """,
        "Media Availability": """
            SELECT 
                COUNT(DISTINCT m.artifact_id) as with_media,
                (SELECT COUNT(*) FROM artifactmetadata) - COUNT(DISTINCT m.artifact_id) as without_media
            FROM artifactmedia m
        """,
        "Top Colors Used": """
            SELECT color, COUNT(*) as count, AVG(percent) as avg_percent
            FROM artifactcolors
            GROUP BY color
            ORDER BY count DESC
            LIMIT 10
        """
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Execute Query"):
        conn = mysql.connector.connect(**db_config)
        df = pd.read_sql(queries[selected_query], conn)
        conn.close()
        
        st.subheader("Results")
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2 and df.shape[0] > 1:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], title=selected_query)
            st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Common Analytical Queries

```python
# Artifacts with most color diversity
query = """
SELECT 
    m.title, 
    m.culture, 
    COUNT(DISTINCT c.color) as color_count
FROM artifactmetadata m
JOIN artifactcolors c ON m.id = c.artifact_id
GROUP BY m.id, m.title, m.culture
ORDER BY color_count DESC
LIMIT 20
"""

# Time period analysis
query = """
SELECT 
    century,
    department,
    COUNT(*) as artifact_count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century, department
ORDER BY century, artifact_count DESC
"""

# Image availability by classification
query = """
SELECT 
    m.classification,
    COUNT(DISTINCT m.id) as total_artifacts,
    COUNT(DISTINCT media.artifact_id) as with_images,
    ROUND(COUNT(DISTINCT media.artifact_id) * 100.0 / COUNT(DISTINCT m.id), 2) as image_percentage
FROM artifactmetadata m
LEFT JOIN artifactmedia media ON m.id = media.artifact_id
GROUP BY m.classification
ORDER BY total_artifacts DESC
"""
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Troubleshooting

**API Rate Limiting:**
```python
import time

def fetch_with_retry(url, params, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:
                time.sleep(2 ** attempt)
            else:
                raise
    raise Exception("Max retries exceeded")
```

**Database Connection Issues:**
```python
# Test connection
try:
    conn = mysql.connector.connect(**db_config)
    print("Database connected successfully!")
    conn.close()
except mysql.connector.Error as err:
    print(f"Error: {err}")
```

**Memory Issues with Large Datasets:**
```python
# Use batch processing
def load_in_batches(df, table_name, batch_size=1000):
    for i in range(0, len(df), batch_size):
        batch = df.iloc[i:i+batch_size]
        batch.to_sql(table_name, conn, if_exists='append', index=False)
```
