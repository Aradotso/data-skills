---
name: harvard-art-museum-etl-analytics
description: End-to-end ETL pipeline and analytics application for Harvard Art Museums API data with SQL storage and Streamlit visualization
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums API
  - set up Harvard artifacts data engineering project
  - create analytics dashboard for museum collection data
  - extract and analyze Harvard Art Museums data
  - build streamlit app with museum API data
  - how to process Harvard Art Museums artifacts with SQL
  - visualize museum collection data with plotly
  - implement ETL for art museum metadata
---

# Harvard Art Museum ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL analytics, and interactive data visualization using Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load museum data into relational databases
- **SQL Database Design**: Structured storage with proper relationships (metadata, media, colors)
- **Analytics Queries**: 20+ predefined SQL queries for museum collection insights
- **Interactive Visualization**: Streamlit dashboard with Plotly charts

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

### API Key Setup

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

Create a `.env` file:

```bash
HARVARD_API_KEY=your_api_key_here
DB_HOST=localhost
DB_USER=your_db_user
DB_PASSWORD=your_db_password
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
    division VARCHAR(200),
    accession_number VARCHAR(100),
    UNIQUE KEY (id)
);

-- Artifact media table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url TEXT,
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id) ON DELETE CASCADE
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id) ON DELETE CASCADE
);
```

## Key API Patterns

### Extracting Data from Harvard API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org"

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts with pagination"""
    url = f"{BASE_URL}/object"
    params = {
        'apikey': API_KEY,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    response.raise_for_status()
    return response.json()

# Fetch multiple pages
def fetch_all_artifacts(max_pages=5):
    all_artifacts = []
    for page in range(1, max_pages + 1):
        data = fetch_artifacts(page=page)
        artifacts = data.get('records', [])
        all_artifacts.extend(artifacts)
        print(f"Fetched page {page}: {len(artifacts)} artifacts")
    return all_artifacts
```

### ETL Pipeline Implementation

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

class HarvardArtifactETL:
    def __init__(self, db_config):
        self.db_config = db_config
        self.connection = None
    
    def connect_db(self):
        """Establish database connection"""
        try:
            self.connection = mysql.connector.connect(
                host=self.db_config['host'],
                user=self.db_config['user'],
                password=self.db_config['password'],
                database=self.db_config['database']
            )
            return self.connection
        except Error as e:
            print(f"Database connection error: {e}")
            return None
    
    def extract(self, max_pages=5):
        """Extract data from API"""
        return fetch_all_artifacts(max_pages)
    
    def transform(self, artifacts):
        """Transform nested JSON to relational format"""
        metadata_list = []
        media_list = []
        colors_list = []
        
        for artifact in artifacts:
            # Transform metadata
            metadata = {
                'id': artifact.get('id'),
                'title': artifact.get('title', '')[:500],
                'culture': artifact.get('culture', '')[:200],
                'century': artifact.get('century', '')[:100],
                'dated': artifact.get('dated', '')[:200],
                'department': artifact.get('department', '')[:200],
                'classification': artifact.get('classification', '')[:200],
                'division': artifact.get('division', '')[:200],
                'accession_number': artifact.get('accessionyear', '')
            }
            metadata_list.append(metadata)
            
            # Transform media
            images = artifact.get('images', [])
            for img in images:
                media = {
                    'artifact_id': artifact.get('id'),
                    'image_url': img.get('baseimageurl'),
                    'media_type': 'image'
                }
                media_list.append(media)
            
            # Transform colors
            colors = artifact.get('colors', [])
            for color in colors:
                color_data = {
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color'),
                    'percentage': color.get('percent')
                }
                colors_list.append(color_data)
        
        return {
            'metadata': pd.DataFrame(metadata_list),
            'media': pd.DataFrame(media_list),
            'colors': pd.DataFrame(colors_list)
        }
    
    def load(self, dataframes):
        """Load data into SQL database"""
        if not self.connection:
            self.connect_db()
        
        cursor = self.connection.cursor()
        
        # Load metadata
        for _, row in dataframes['metadata'].iterrows():
            query = """
                INSERT INTO artifactmetadata 
                (id, title, culture, century, dated, department, classification, division, accession_number)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE title=VALUES(title)
            """
            cursor.execute(query, tuple(row))
        
        # Load media
        for _, row in dataframes['media'].iterrows():
            query = """
                INSERT INTO artifactmedia (artifact_id, image_url, media_type)
                VALUES (%s, %s, %s)
            """
            cursor.execute(query, tuple(row))
        
        # Load colors
        for _, row in dataframes['colors'].iterrows():
            query = """
                INSERT INTO artifactcolors (artifact_id, color, percentage)
                VALUES (%s, %s, %s)
            """
            cursor.execute(query, tuple(row))
        
        self.connection.commit()
        cursor.close()
        print("Data loaded successfully")

# Usage
db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

etl = HarvardArtifactETL(db_config)
artifacts = etl.extract(max_pages=3)
transformed = etl.transform(artifacts)
etl.load(transformed)
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import pandas as pd
import plotly.express as px
import mysql.connector

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

# Database connection
@st.cache_resource
def get_db_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

# Analytics queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
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
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as occurrences, AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY occurrences DESC
        LIMIT 10
    """,
    
    "Artifacts with Media": """
        SELECT 
            CASE WHEN m.artifact_id IS NOT NULL THEN 'With Media' ELSE 'Without Media' END as media_status,
            COUNT(DISTINCT a.id) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY media_status
    """
}

def execute_query(query):
    """Execute SQL query and return DataFrame"""
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# Streamlit UI
st.title("🎨 Harvard Art Museums Analytics Dashboard")

st.sidebar.header("Analytics Options")
selected_query = st.sidebar.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))

if st.sidebar.button("Run Analysis"):
    with st.spinner("Executing query..."):
        query = ANALYTICS_QUERIES[selected_query]
        df = execute_query(query)
        
        st.subheader(f"Results: {selected_query}")
        st.dataframe(df, use_container_width=True)
        
        # Visualization
        if len(df.columns) >= 2:
            fig = px.bar(
                df, 
                x=df.columns[0], 
                y=df.columns[1],
                title=selected_query,
                labels={df.columns[0]: df.columns[0].replace('_', ' ').title(),
                        df.columns[1]: df.columns[1].replace('_', ' ').title()}
            )
            st.plotly_chart(fig, use_container_width=True)

# ETL Controls
st.sidebar.header("ETL Pipeline")
if st.sidebar.button("Run ETL Pipeline"):
    with st.spinner("Running ETL..."):
        etl = HarvardArtifactETL(db_config)
        artifacts = etl.extract(max_pages=2)
        transformed = etl.transform(artifacts)
        etl.load(transformed)
        st.sidebar.success(f"Loaded {len(artifacts)} artifacts")
```

## Common Analytics Queries

```python
# Culture analysis with classification
culture_classification_query = """
    SELECT culture, classification, COUNT(*) as count
    FROM artifactmetadata
    WHERE culture IS NOT NULL AND classification IS NOT NULL
    GROUP BY culture, classification
    ORDER BY count DESC
    LIMIT 20
"""

# Color diversity analysis
color_diversity_query = """
    SELECT 
        a.culture,
        COUNT(DISTINCT c.color) as unique_colors,
        COUNT(*) as total_color_entries
    FROM artifactmetadata a
    JOIN artifactcolors c ON a.id = c.artifact_id
    GROUP BY a.culture
    HAVING unique_colors > 5
    ORDER BY unique_colors DESC
"""

# Media-rich artifacts
media_rich_query = """
    SELECT 
        a.id,
        a.title,
        a.culture,
        COUNT(m.id) as media_count
    FROM artifactmetadata a
    JOIN artifactmedia m ON a.id = m.artifact_id
    GROUP BY a.id, a.title, a.culture
    HAVING media_count > 3
    ORDER BY media_count DESC
"""
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_rate_limit(page, delay=1):
    """Add delay between API calls"""
    time.sleep(delay)
    return fetch_artifacts(page)
```

### Database Connection Issues

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
        if conn.is_connected():
            print("✓ Database connection successful")
            conn.close()
            return True
    except Error as e:
        print(f"✗ Database connection failed: {e}")
        return False
```

### Handling NULL Values

```python
def safe_transform(artifact):
    """Handle missing data during transformation"""
    return {
        'id': artifact.get('id'),
        'title': artifact.get('title') or 'Unknown',
        'culture': artifact.get('culture') or 'Unknown',
        'century': artifact.get('century') or 'Unknown'
    }
```

## Best Practices

1. **Batch Processing**: Process API data in batches to manage memory
2. **Error Handling**: Wrap API calls in try-except blocks
3. **Data Validation**: Validate data before loading into SQL
4. **Incremental Loading**: Use ON DUPLICATE KEY UPDATE for idempotent loads
5. **Caching**: Use Streamlit's `@st.cache_data` for query results

This skill enables building production-ready ETL pipelines with museum collection data, SQL analytics, and interactive dashboards.
