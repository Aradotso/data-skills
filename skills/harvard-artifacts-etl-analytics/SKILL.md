---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - extract and transform Harvard API data to SQL
  - build streamlit app for art museum analytics
  - query Harvard Art Museums API with pagination
  - visualize museum artifact data with plotly
  - design SQL schema for artifact metadata
  - implement batch insert for museum data
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering solution for the Harvard Art Museums API. It demonstrates ETL pipeline construction, SQL database design, analytical queries, and interactive visualization using Streamlit. The application extracts artifact metadata, transforms nested JSON into relational tables, loads data into MySQL/TiDB, and provides 20+ analytical queries with auto-generated visualizations.

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required dependencies:**
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file or configure environment variables:

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

```python
import mysql.connector
import os
from dotenv import load_dotenv

load_dotenv()

# Connection configuration
db_config = {
    'host': os.getenv('DB_HOST'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

# Create database connection
conn = mysql.connector.connect(**db_config)
cursor = conn.cursor()

# Create tables
cursor.execute("""
CREATE TABLE IF NOT EXISTS artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    dated VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    objectnumber VARCHAR(100),
    division VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    provenance TEXT,
    description TEXT,
    url VARCHAR(500)
)
""")

cursor.execute("""
CREATE TABLE IF NOT EXISTS artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    iiifbaseuri VARCHAR(500),
    baseimageurl VARCHAR(500),
    renditionnumber VARCHAR(50),
    format VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
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
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
)
""")

conn.commit()
cursor.close()
conn.close()
```

## API Integration

### Fetching Data from Harvard Art Museums API

```python
import requests
import os

class HarvardAPIClient:
    def __init__(self, api_key=None):
        self.api_key = api_key or os.getenv('HARVARD_API_KEY')
        self.base_url = "https://api.harvardartmuseums.org/object"
    
    def fetch_artifacts(self, size=100, page=1):
        """Fetch artifacts with pagination"""
        params = {
            'apikey': self.api_key,
            'size': size,
            'page': page
        }
        
        response = requests.get(self.base_url, params=params)
        response.raise_for_status()
        return response.json()
    
    def fetch_all_artifacts(self, max_records=1000):
        """Fetch multiple pages of artifacts"""
        all_records = []
        page = 1
        size = 100
        
        while len(all_records) < max_records:
            data = self.fetch_artifacts(size=size, page=page)
            records = data.get('records', [])
            
            if not records:
                break
            
            all_records.extend(records)
            page += 1
            
            # Respect rate limits
            import time
            time.sleep(0.5)
        
        return all_records[:max_records]
```

## ETL Pipeline

### Extract, Transform, Load

```python
import pandas as pd
import mysql.connector

class ArtifactETL:
    def __init__(self, db_config):
        self.db_config = db_config
    
    def extract(self, api_client, num_records=500):
        """Extract data from API"""
        print(f"Extracting {num_records} records...")
        return api_client.fetch_all_artifacts(max_records=num_records)
    
    def transform(self, raw_data):
        """Transform raw API data into structured dataframes"""
        metadata_list = []
        media_list = []
        colors_list = []
        
        for artifact in raw_data:
            # Metadata
            metadata_list.append({
                'id': artifact.get('id'),
                'title': artifact.get('title', '')[:500],
                'culture': artifact.get('culture', '')[:255],
                'century': artifact.get('century', '')[:100],
                'dated': artifact.get('dated', '')[:255],
                'classification': artifact.get('classification', '')[:255],
                'department': artifact.get('department', '')[:255],
                'objectnumber': artifact.get('objectnumber', '')[:100],
                'division': artifact.get('division', '')[:255],
                'technique': artifact.get('technique', '')[:500],
                'medium': artifact.get('medium', '')[:500],
                'provenance': artifact.get('provenance', ''),
                'description': artifact.get('description', ''),
                'url': artifact.get('url', '')[:500]
            })
            
            # Media
            for media in artifact.get('images', []):
                media_list.append({
                    'artifact_id': artifact.get('id'),
                    'iiifbaseuri': media.get('iiifbaseuri', '')[:500],
                    'baseimageurl': media.get('baseimageurl', '')[:500],
                    'renditionnumber': media.get('renditionnumber', '')[:50],
                    'format': media.get('format', '')[:50]
                })
            
            # Colors
            for color in artifact.get('colors', []):
                colors_list.append({
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color', '')[:50],
                    'spectrum': color.get('spectrum', '')[:50],
                    'hue': color.get('hue', '')[:50],
                    'percent': color.get('percent', 0.0)
                })
        
        return (
            pd.DataFrame(metadata_list),
            pd.DataFrame(media_list),
            pd.DataFrame(colors_list)
        )
    
    def load(self, metadata_df, media_df, colors_df):
        """Load data into SQL database"""
        conn = mysql.connector.connect(**self.db_config)
        cursor = conn.cursor()
        
        try:
            # Load metadata
            for _, row in metadata_df.iterrows():
                cursor.execute("""
                    INSERT INTO artifactmetadata 
                    (id, title, culture, century, dated, classification, department, 
                     objectnumber, division, technique, medium, provenance, description, url)
                    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                    ON DUPLICATE KEY UPDATE title=VALUES(title)
                """, tuple(row))
            
            # Load media
            for _, row in media_df.iterrows():
                cursor.execute("""
                    INSERT INTO artifactmedia 
                    (artifact_id, iiifbaseuri, baseimageurl, renditionnumber, format)
                    VALUES (%s, %s, %s, %s, %s)
                """, tuple(row))
            
            # Load colors
            for _, row in colors_df.iterrows():
                cursor.execute("""
                    INSERT INTO artifactcolors 
                    (artifact_id, color, spectrum, hue, percent)
                    VALUES (%s, %s, %s, %s, %s)
                """, tuple(row))
            
            conn.commit()
            print(f"Loaded {len(metadata_df)} artifacts, {len(media_df)} media items, {len(colors_df)} colors")
        
        finally:
            cursor.close()
            conn.close()
```

## Streamlit Application

### Main App Structure

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Sidebar navigation
page = st.sidebar.selectbox(
    "Navigation",
    ["Data Collection", "Analytics Queries", "Visualizations"]
)

if page == "Data Collection":
    st.header("📥 Data Collection")
    
    num_records = st.number_input("Number of records to fetch", 
                                   min_value=10, max_value=5000, value=100)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Extracting data from API..."):
            api_client = HarvardAPIClient()
            etl = ArtifactETL(db_config)
            
            # Extract
            raw_data = etl.extract(api_client, num_records)
            st.success(f"Extracted {len(raw_data)} records")
            
            # Transform
            metadata_df, media_df, colors_df = etl.transform(raw_data)
            st.success("Data transformed successfully")
            
            # Load
            etl.load(metadata_df, media_df, colors_df)
            st.success("Data loaded to database")

elif page == "Analytics Queries":
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
            SELECT color, AVG(percent) as avg_percent, COUNT(*) as frequency 
            FROM artifactcolors 
            GROUP BY color 
            ORDER BY frequency DESC 
            LIMIT 10
        """,
        "Media Availability": """
            SELECT 
                CASE WHEN m.artifact_id IS NULL THEN 'No Media' ELSE 'Has Media' END as media_status,
                COUNT(*) as count
            FROM artifactmetadata a
            LEFT JOIN artifactmedia m ON a.id = m.artifact_id
            GROUP BY media_status
        """
    }
    
    query_name = st.selectbox("Select Query", list(queries.keys()))
    
    if st.button("Execute Query"):
        conn = mysql.connector.connect(**db_config)
        df = pd.read_sql(queries[query_name], conn)
        conn.close()
        
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                        title=query_name)
            st.plotly_chart(fig, use_container_width=True)
```

## Common Analytics Queries

```python
# Top 10 classifications
"""
SELECT classification, COUNT(*) as count 
FROM artifactmetadata 
WHERE classification IS NOT NULL 
GROUP BY classification 
ORDER BY count DESC 
LIMIT 10
"""

# Artifacts with complete metadata
"""
SELECT 
    COUNT(*) as total,
    SUM(CASE WHEN culture IS NOT NULL THEN 1 ELSE 0 END) as with_culture,
    SUM(CASE WHEN century IS NOT NULL THEN 1 ELSE 0 END) as with_century,
    SUM(CASE WHEN medium IS NOT NULL THEN 1 ELSE 0 END) as with_medium
FROM artifactmetadata
"""

# Color spectrum distribution
"""
SELECT spectrum, COUNT(*) as count, AVG(percent) as avg_percent 
FROM artifactcolors 
GROUP BY spectrum 
ORDER BY count DESC
"""
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Troubleshooting

**API Rate Limiting:**
- Add delays between requests (`time.sleep(0.5)`)
- Reduce batch size
- Check API quota limits

**Database Connection Issues:**
```python
# Test connection
try:
    conn = mysql.connector.connect(**db_config)
    print("Connected successfully")
except mysql.connector.Error as err:
    print(f"Error: {err}")
```

**Memory Issues with Large Datasets:**
- Use batch processing
- Implement chunked inserts
- Clear dataframes after loading

**Missing Data Fields:**
- Always use `.get()` with defaults for nested JSON
- Implement null checks in SQL queries
- Validate data before inserting
