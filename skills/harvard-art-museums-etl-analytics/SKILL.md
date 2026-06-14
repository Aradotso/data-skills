---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard with museum artifacts data
  - set up Harvard Art Museums API data engineering project
  - analyze art museum collection data with SQL
  - build Streamlit app for artifact visualization
  - extract and transform Harvard museum API data
  - create data pipeline for art collection analytics
  - visualize museum artifacts with Plotly and Streamlit
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit visualizations for museum artifact data.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides:

- **API Integration**: Secure data collection from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract artifact metadata, transform nested JSON into relational tables, and load into SQL databases
- **SQL Analytics**: 20+ predefined analytical queries for artifact insights
- **Interactive Dashboards**: Streamlit-based visualization using Plotly for real-time analytics
- **Database Design**: Properly structured relational schema with foreign key relationships

## Installation

### Prerequisites

```bash
# Python 3.8+
# MySQL or TiDB Cloud account
# Harvard Art Museums API key (free from https://www.harvardartmuseums.org/collections/api)
```

### Setup

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Expected requirements.txt contents:
# streamlit
# pandas
# requests
# mysql-connector-python
# plotly
# python-dotenv
```

### Environment Configuration

Create a `.env` file in the project root:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_username
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

## Database Schema Setup

The project uses three main tables with proper relationships:

```sql
-- Artifact Metadata (main table)
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    division VARCHAR(255),
    department VARCHAR(255),
    technique VARCHAR(500),
    period VARCHAR(255),
    dated VARCHAR(255),
    url TEXT,
    accession_year INT
);

-- Artifact Media (images/files)
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    base_url TEXT,
    format VARCHAR(50),
    height INT,
    width INT,
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

## Core Architecture Pattern

**API → ETL → SQL → Analytics → Visualization**

```python
import os
import requests
import pandas as pd
import mysql.connector
from dotenv import load_dotenv

# Load environment variables
load_dotenv()

class HarvardArtETL:
    def __init__(self):
        self.api_key = os.getenv('HARVARD_API_KEY')
        self.base_url = 'https://api.harvardartmuseums.org/object'
        self.db_config = {
            'host': os.getenv('DB_HOST'),
            'port': int(os.getenv('DB_PORT', 3306)),
            'user': os.getenv('DB_USER'),
            'password': os.getenv('DB_PASSWORD'),
            'database': os.getenv('DB_NAME')
        }
    
    def extract_artifacts(self, page=1, size=100):
        """Extract artifact data from Harvard API with pagination"""
        params = {
            'apikey': self.api_key,
            'page': page,
            'size': size
        }
        
        try:
            response = requests.get(self.base_url, params=params)
            response.raise_for_status()
            data = response.json()
            return data.get('records', []), data.get('info', {})
        except requests.exceptions.RequestException as e:
            print(f"API Error: {e}")
            return [], {}
    
    def transform_metadata(self, records):
        """Transform artifact records into metadata DataFrame"""
        metadata = []
        for record in records:
            metadata.append({
                'id': record.get('id'),
                'title': record.get('title', '')[:500],
                'culture': record.get('culture', '')[:255],
                'century': record.get('century', '')[:100],
                'classification': record.get('classification', '')[:255],
                'division': record.get('division', '')[:255],
                'department': record.get('department', '')[:255],
                'technique': record.get('technique', '')[:500],
                'period': record.get('period', '')[:255],
                'dated': record.get('dated', '')[:255],
                'url': record.get('url', ''),
                'accession_year': record.get('accessionyear')
            })
        return pd.DataFrame(metadata)
    
    def transform_media(self, records):
        """Extract and transform media data from nested JSON"""
        media_data = []
        for record in records:
            artifact_id = record.get('id')
            images = record.get('images', [])
            
            for image in images:
                media_data.append({
                    'artifact_id': artifact_id,
                    'base_url': image.get('baseimageurl', ''),
                    'format': image.get('format', ''),
                    'height': image.get('height'),
                    'width': image.get('width')
                })
        return pd.DataFrame(media_data)
    
    def transform_colors(self, records):
        """Extract and transform color data"""
        color_data = []
        for record in records:
            artifact_id = record.get('id')
            colors = record.get('colors', [])
            
            for color in colors:
                color_data.append({
                    'artifact_id': artifact_id,
                    'color': color.get('color', ''),
                    'spectrum': color.get('spectrum', ''),
                    'hue': color.get('hue', ''),
                    'percent': color.get('percent')
                })
        return pd.DataFrame(color_data)
    
    def load_to_sql(self, df, table_name, if_exists='append'):
        """Load DataFrame to SQL database"""
        try:
            conn = mysql.connector.connect(**self.db_config)
            cursor = conn.cursor()
            
            # Prepare batch insert
            if not df.empty:
                columns = ', '.join(df.columns)
                placeholders = ', '.join(['%s'] * len(df.columns))
                
                if if_exists == 'replace':
                    cursor.execute(f"DELETE FROM {table_name}")
                
                insert_query = f"INSERT IGNORE INTO {table_name} ({columns}) VALUES ({placeholders})"
                cursor.executemany(insert_query, df.values.tolist())
                
                conn.commit()
                print(f"Loaded {cursor.rowcount} rows into {table_name}")
            
            cursor.close()
            conn.close()
        except mysql.connector.Error as e:
            print(f"Database Error: {e}")
    
    def run_etl_pipeline(self, num_pages=5):
        """Execute complete ETL pipeline"""
        for page in range(1, num_pages + 1):
            print(f"Processing page {page}...")
            
            # Extract
            records, info = self.extract_artifacts(page=page)
            
            if not records:
                break
            
            # Transform
            metadata_df = self.transform_metadata(records)
            media_df = self.transform_media(records)
            colors_df = self.transform_colors(records)
            
            # Load
            self.load_to_sql(metadata_df, 'artifactmetadata')
            self.load_to_sql(media_df, 'artifactmedia')
            self.load_to_sql(colors_df, 'artifactcolors')
            
            print(f"Page {page} completed. Total pages: {info.get('pages', 'unknown')}")
```

## Streamlit Application Pattern

```python
import streamlit as st
import pandas as pd
import plotly.express as px
from harvard_etl import HarvardArtETL

def main():
    st.set_page_config(page_title="Harvard Art Museums Analytics", layout="wide")
    st.title("🎨 Harvard Art Museums - Data Analytics Dashboard")
    
    # Initialize ETL
    etl = HarvardArtETL()
    
    # Sidebar for ETL controls
    st.sidebar.header("⚙️ ETL Pipeline")
    
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Extracting data from API..."):
            num_pages = st.sidebar.number_input("Number of pages", 1, 50, 5)
            etl.run_etl_pipeline(num_pages=num_pages)
            st.success(f"ETL completed for {num_pages} pages!")
    
    # Analytics Section
    st.header("📊 SQL Analytics")
    
    # Query selector
    queries = {
        "Artifacts by Culture": """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture IS NOT NULL AND culture != ''
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 15
        """,
        "Artifacts by Century": """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL AND century != ''
            GROUP BY century
            ORDER BY count DESC
        """,
        "Department Distribution": """
            SELECT department, COUNT(*) as count
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY count DESC
        """,
        "Media Availability": """
            SELECT 
                CASE WHEN media_count > 0 THEN 'Has Media' ELSE 'No Media' END as media_status,
                COUNT(*) as artifact_count
            FROM (
                SELECT a.id, COUNT(m.media_id) as media_count
                FROM artifactmetadata a
                LEFT JOIN artifactmedia m ON a.id = m.artifact_id
                GROUP BY a.id
            ) as media_summary
            GROUP BY media_status
        """,
        "Top Colors Used": """
            SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
            FROM artifactcolors
            WHERE color IS NOT NULL
            GROUP BY color
            ORDER BY usage_count DESC
            LIMIT 10
        """,
        "Accession Year Trends": """
            SELECT accession_year, COUNT(*) as count
            FROM artifactmetadata
            WHERE accession_year IS NOT NULL
            GROUP BY accession_year
            ORDER BY accession_year DESC
            LIMIT 20
        """
    }
    
    selected_query = st.selectbox("Select Analysis Query", list(queries.keys()))
    
    if st.button("Execute Query"):
        try:
            conn = mysql.connector.connect(**etl.db_config)
            df = pd.read_sql(queries[selected_query], conn)
            conn.close()
            
            # Display results
            st.subheader("Query Results")
            st.dataframe(df, use_container_width=True)
            
            # Visualization
            if len(df.columns) >= 2:
                st.subheader("Visualization")
                fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                            title=selected_query)
                st.plotly_chart(fig, use_container_width=True)
        
        except Exception as e:
            st.error(f"Query Error: {e}")

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Common Analytical Queries

### Complex Join Analysis

```python
# Artifacts with color analysis
query = """
SELECT 
    a.title,
    a.culture,
    a.century,
    GROUP_CONCAT(DISTINCT c.color ORDER BY c.percent DESC) as dominant_colors,
    COUNT(DISTINCT m.media_id) as image_count
FROM artifactmetadata a
LEFT JOIN artifactcolors c ON a.id = c.artifact_id
LEFT JOIN artifactmedia m ON a.id = m.artifact_id
WHERE a.culture IS NOT NULL
GROUP BY a.id, a.title, a.culture, a.century
HAVING image_count > 0
ORDER BY image_count DESC
LIMIT 50
"""
```

### Time-based Analytics

```python
# Artifact acquisition patterns over time
query = """
SELECT 
    accession_year,
    department,
    COUNT(*) as acquisitions,
    COUNT(DISTINCT classification) as classification_diversity
FROM artifactmetadata
WHERE accession_year BETWEEN 1900 AND 2024
GROUP BY accession_year, department
ORDER BY accession_year DESC, acquisitions DESC
"""
```

## Troubleshooting

### API Rate Limiting

```python
import time

def extract_artifacts_with_retry(self, page=1, size=100, max_retries=3):
    """Extract with exponential backoff"""
    for attempt in range(max_retries):
        try:
            records, info = self.extract_artifacts(page, size)
            return records, info
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:  # Rate limit
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    return [], {}
```

### Database Connection Issues

```python
def get_db_connection(self):
    """Get database connection with error handling"""
    try:
        conn = mysql.connector.connect(
            **self.db_config,
            connect_timeout=10,
            autocommit=False
        )
        return conn
    except mysql.connector.Error as e:
        print(f"Connection failed: {e}")
        print("Check DB_HOST, DB_USER, DB_PASSWORD in .env")
        return None
```

### Handling Missing Data

```python
def transform_metadata(self, records):
    """Transform with null handling"""
    metadata = []
    for record in records:
        metadata.append({
            'id': record.get('id'),
            'title': (record.get('title') or 'Untitled')[:500],
            'culture': (record.get('culture') or 'Unknown')[:255],
            # Truncate strings to prevent SQL errors
            'century': str(record.get('century', ''))[:100] if record.get('century') else None,
            # Handle numeric fields
            'accession_year': int(record['accessionyear']) if record.get('accessionyear') else None
        })
    return pd.DataFrame(metadata)
```

## Best Practices

1. **Batch Processing**: Process API data in batches to avoid memory issues
2. **Error Logging**: Log API errors and database issues for debugging
3. **Data Validation**: Validate data types before SQL insertion
4. **Connection Pooling**: Reuse database connections for better performance
5. **Incremental Loads**: Track processed pages to avoid duplicate data
6. **Environment Variables**: Never hardcode credentials; always use `.env` files

This skill provides comprehensive knowledge for building production-ready ETL pipelines and analytics dashboards using museum API data, SQL databases, and modern Python visualization tools.
