---
name: harvard-art-museums-data-engineering-pipeline
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build a data pipeline for museum artifacts
  - create an ETL pipeline with Harvard Art Museums API
  - set up analytics dashboard for art collection data
  - extract and analyze Harvard museum artifacts
  - build Streamlit app for art collection analytics
  - implement SQL analytics for museum data
  - create data engineering pipeline for cultural artifacts
  - visualize Harvard Art Museums collection data
---

# Harvard Art Museums Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides a complete data engineering and analytics pipeline for the Harvard Art Museums API. It demonstrates professional ETL patterns, relational database design, SQL analytics, and interactive visualization using Streamlit. The architecture follows: **API → ETL → SQL → Analytics → Visualization**.

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

# Database Configuration (MySQL/TiDB Cloud)
DB_HOST=localhost
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Request an API key
3. Add to `.env` file

## Database Schema

The pipeline creates three relational tables:

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    url TEXT,
    PRIMARY KEY (id)
);

-- Artifact Media
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(50),
    media_url TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Components

### 1. API Data Collection

```python
import requests
import pandas as pd
from typing import Dict, List
import os
from dotenv import load_dotenv

load_dotenv()

class HarvardAPIClient:
    def __init__(self):
        self.api_key = os.getenv('HARVARD_API_KEY')
        self.base_url = 'https://api.harvardartmuseums.org/object'
    
    def fetch_artifacts(self, num_pages: int = 10, size: int = 100) -> List[Dict]:
        """Fetch artifacts with pagination handling"""
        artifacts = []
        
        for page in range(1, num_pages + 1):
            params = {
                'apikey': self.api_key,
                'size': size,
                'page': page
            }
            
            try:
                response = requests.get(self.base_url, params=params)
                response.raise_for_status()
                data = response.json()
                artifacts.extend(data.get('records', []))
                
                # Rate limiting
                import time
                time.sleep(0.5)
                
            except requests.exceptions.RequestException as e:
                print(f"Error fetching page {page}: {e}")
                continue
        
        return artifacts
    
    def get_artifact_by_id(self, artifact_id: int) -> Dict:
        """Fetch single artifact by ID"""
        url = f"{self.base_url}/{artifact_id}"
        params = {'apikey': self.api_key}
        
        response = requests.get(url, params=params)
        response.raise_for_status()
        return response.json()
```

### 2. ETL Pipeline

```python
import mysql.connector
from typing import List, Dict
import pandas as pd

class ETLPipeline:
    def __init__(self):
        self.db_config = {
            'host': os.getenv('DB_HOST'),
            'port': int(os.getenv('DB_PORT', 3306)),
            'user': os.getenv('DB_USER'),
            'password': os.getenv('DB_PASSWORD'),
            'database': os.getenv('DB_NAME')
        }
    
    def extract(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Extract and flatten artifact data"""
        metadata = []
        
        for artifact in artifacts:
            metadata.append({
                'id': artifact.get('id'),
                'title': artifact.get('title', 'Untitled'),
                'culture': artifact.get('culture'),
                'century': artifact.get('century'),
                'classification': artifact.get('classification'),
                'department': artifact.get('department'),
                'dated': artifact.get('dated'),
                'url': artifact.get('url')
            })
        
        return pd.DataFrame(metadata)
    
    def transform_media(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Extract media information"""
        media_data = []
        
        for artifact in artifacts:
            artifact_id = artifact.get('id')
            images = artifact.get('images', [])
            
            for image in images:
                media_data.append({
                    'artifact_id': artifact_id,
                    'media_type': 'image',
                    'media_url': image.get('baseimageurl')
                })
        
        return pd.DataFrame(media_data)
    
    def transform_colors(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Extract color information"""
        color_data = []
        
        for artifact in artifacts:
            artifact_id = artifact.get('id')
            colors = artifact.get('colors', [])
            
            for color in colors:
                color_data.append({
                    'artifact_id': artifact_id,
                    'color': color.get('color'),
                    'percentage': color.get('percent')
                })
        
        return pd.DataFrame(color_data)
    
    def load(self, df: pd.DataFrame, table_name: str):
        """Load data into SQL database"""
        conn = mysql.connector.connect(**self.db_config)
        cursor = conn.cursor()
        
        if table_name == 'artifactmetadata':
            query = """
                INSERT INTO artifactmetadata 
                (id, title, culture, century, classification, department, dated, url)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE
                title=VALUES(title), culture=VALUES(culture)
            """
            data = df[['id', 'title', 'culture', 'century', 
                      'classification', 'department', 'dated', 'url']].values.tolist()
        
        elif table_name == 'artifactmedia':
            query = """
                INSERT INTO artifactmedia (artifact_id, media_type, media_url)
                VALUES (%s, %s, %s)
            """
            data = df[['artifact_id', 'media_type', 'media_url']].values.tolist()
        
        elif table_name == 'artifactcolors':
            query = """
                INSERT INTO artifactcolors (artifact_id, color, percentage)
                VALUES (%s, %s, %s)
            """
            data = df[['artifact_id', 'color', 'percentage']].values.tolist()
        
        cursor.executemany(query, data)
        conn.commit()
        cursor.close()
        conn.close()
        
        print(f"Loaded {len(data)} records into {table_name}")
```

### 3. SQL Analytics Queries

```python
class AnalyticsEngine:
    def __init__(self, db_config: Dict):
        self.db_config = db_config
    
    def execute_query(self, query: str) -> pd.DataFrame:
        """Execute SQL query and return DataFrame"""
        conn = mysql.connector.connect(**self.db_config)
        df = pd.read_sql(query, conn)
        conn.close()
        return df
    
    # Sample Analytical Queries
    
    def artifacts_by_culture(self) -> pd.DataFrame:
        """Count artifacts by culture"""
        query = """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture IS NOT NULL
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 20
        """
        return self.execute_query(query)
    
    def artifacts_by_century(self) -> pd.DataFrame:
        """Artifacts distribution by century"""
        query = """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL
            GROUP BY century
            ORDER BY century
        """
        return self.execute_query(query)
    
    def top_colors(self) -> pd.DataFrame:
        """Most common colors across artifacts"""
        query = """
            SELECT color, COUNT(*) as frequency, AVG(percentage) as avg_percentage
            FROM artifactcolors
            WHERE color IS NOT NULL
            GROUP BY color
            ORDER BY frequency DESC
            LIMIT 15
        """
        return self.execute_query(query)
    
    def artifacts_with_images(self) -> pd.DataFrame:
        """Artifacts with media availability"""
        query = """
            SELECT 
                CASE 
                    WHEN m.artifact_id IS NOT NULL THEN 'With Images'
                    ELSE 'No Images'
                END as image_status,
                COUNT(*) as count
            FROM artifactmetadata a
            LEFT JOIN artifactmedia m ON a.id = m.artifact_id
            GROUP BY image_status
        """
        return self.execute_query(query)
    
    def department_classification(self) -> pd.DataFrame:
        """Classification breakdown by department"""
        query = """
            SELECT department, classification, COUNT(*) as count
            FROM artifactmetadata
            WHERE department IS NOT NULL AND classification IS NOT NULL
            GROUP BY department, classification
            ORDER BY department, count DESC
        """
        return self.execute_query(query)
```

### 4. Streamlit Application

```python
import streamlit as st
import plotly.express as px
from dotenv import load_dotenv

load_dotenv()

# Configure page
st.set_page_config(
    page_title="Harvard Art Museums Analytics",
    page_icon="🎨",
    layout="wide"
)

def main():
    st.title("🎨 Harvard Art Museums Data Analytics")
    st.markdown("End-to-end data engineering pipeline with interactive visualizations")
    
    # Initialize components
    db_config = {
        'host': os.getenv('DB_HOST'),
        'port': int(os.getenv('DB_PORT', 3306)),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    
    analytics = AnalyticsEngine(db_config)
    
    # Sidebar navigation
    st.sidebar.title("Navigation")
    page = st.sidebar.radio(
        "Choose a section",
        ["Data Collection", "Analytics Dashboard", "Custom Query"]
    )
    
    if page == "Data Collection":
        st.header("Data Collection from Harvard API")
        
        num_pages = st.number_input("Number of pages to fetch", 1, 50, 10)
        
        if st.button("Start ETL Pipeline"):
            with st.spinner("Fetching data from API..."):
                client = HarvardAPIClient()
                artifacts = client.fetch_artifacts(num_pages=num_pages)
                st.success(f"Fetched {len(artifacts)} artifacts")
            
            with st.spinner("Running ETL pipeline..."):
                etl = ETLPipeline()
                
                # Metadata
                metadata_df = etl.extract(artifacts)
                etl.load(metadata_df, 'artifactmetadata')
                
                # Media
                media_df = etl.transform_media(artifacts)
                etl.load(media_df, 'artifactmedia')
                
                # Colors
                colors_df = etl.transform_colors(artifacts)
                etl.load(colors_df, 'artifactcolors')
                
                st.success("ETL pipeline completed successfully!")
    
    elif page == "Analytics Dashboard":
        st.header("📊 Analytics Dashboard")
        
        col1, col2 = st.columns(2)
        
        with col1:
            st.subheader("Artifacts by Culture")
            culture_df = analytics.artifacts_by_culture()
            fig = px.bar(culture_df, x='culture', y='count', 
                        title="Top 20 Cultures")
            st.plotly_chart(fig, use_container_width=True)
            st.dataframe(culture_df)
        
        with col2:
            st.subheader("Color Distribution")
            colors_df = analytics.top_colors()
            fig = px.bar(colors_df, x='color', y='frequency',
                        title="Most Common Colors")
            st.plotly_chart(fig, use_container_width=True)
            st.dataframe(colors_df)
        
        st.subheader("Artifacts by Century")
        century_df = analytics.artifacts_by_century()
        fig = px.line(century_df, x='century', y='count',
                     title="Temporal Distribution")
        st.plotly_chart(fig, use_container_width=True)
        
        st.subheader("Media Availability")
        media_df = analytics.artifacts_with_images()
        fig = px.pie(media_df, names='image_status', values='count',
                    title="Artifacts with/without Images")
        st.plotly_chart(fig, use_container_width=True)
    
    elif page == "Custom Query":
        st.header("Custom SQL Query")
        
        query = st.text_area(
            "Enter your SQL query",
            height=150,
            placeholder="SELECT * FROM artifactmetadata LIMIT 10"
        )
        
        if st.button("Execute Query"):
            try:
                result_df = analytics.execute_query(query)
                st.success(f"Query returned {len(result_df)} rows")
                st.dataframe(result_df)
                
                # Auto-visualization for numeric columns
                numeric_cols = result_df.select_dtypes(include=['int64', 'float64']).columns
                if len(numeric_cols) > 0 and len(result_df) > 0:
                    st.subheader("Automatic Visualization")
                    fig = px.bar(result_df, x=result_df.columns[0], y=numeric_cols[0])
                    st.plotly_chart(fig)
                    
            except Exception as e:
                st.error(f"Query error: {e}")

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# The app will open at http://localhost:8501
```

## Common Patterns

### Complete Pipeline Execution

```python
from dotenv import load_dotenv
import os

load_dotenv()

def run_complete_pipeline():
    """Run full ETL pipeline"""
    # 1. Fetch data
    client = HarvardAPIClient()
    artifacts = client.fetch_artifacts(num_pages=10)
    print(f"Fetched {len(artifacts)} artifacts")
    
    # 2. Transform and Load
    etl = ETLPipeline()
    
    metadata_df = etl.extract(artifacts)
    etl.load(metadata_df, 'artifactmetadata')
    
    media_df = etl.transform_media(artifacts)
    etl.load(media_df, 'artifactmedia')
    
    colors_df = etl.transform_colors(artifacts)
    etl.load(colors_df, 'artifactcolors')
    
    # 3. Run analytics
    db_config = {
        'host': os.getenv('DB_HOST'),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    
    analytics = AnalyticsEngine(db_config)
    results = analytics.artifacts_by_culture()
    print(results.head())

if __name__ == "__main__":
    run_complete_pipeline()
```

### Incremental Data Loading

```python
def incremental_load(last_artifact_id: int):
    """Load only new artifacts"""
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    max_id = cursor.fetchone()[0] or 0
    
    client = HarvardAPIClient()
    # Fetch artifacts with ID > max_id
    # Process and load
    
    cursor.close()
    conn.close()
```

## Troubleshooting

### API Rate Limiting
```python
# Add delays between requests
import time
time.sleep(0.5)  # 500ms delay

# Handle rate limit errors
try:
    response = requests.get(url, params=params)
    if response.status_code == 429:
        time.sleep(5)  # Wait before retry
except requests.exceptions.HTTPError as e:
    print(f"Rate limit hit: {e}")
```

### Database Connection Issues
```python
# Test database connection
def test_db_connection():
    try:
        conn = mysql.connector.connect(**db_config)
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        cursor.close()
        conn.close()
        print("Database connection successful")
        return True
    except mysql.connector.Error as e:
        print(f"Database error: {e}")
        return False
```

### Missing Data Handling
```python
# Handle null values in ETL
def safe_extract(artifact: Dict, key: str, default='Unknown'):
    """Safely extract values with defaults"""
    value = artifact.get(key)
    return value if value is not None else default

# In transform
metadata = {
    'title': safe_extract(artifact, 'title', 'Untitled'),
    'culture': safe_extract(artifact, 'culture', 'Unknown'),
}
```

### Streamlit Caching for Performance
```python
@st.cache_data(ttl=3600)
def load_analytics_data():
    """Cache expensive queries for 1 hour"""
    analytics = AnalyticsEngine(db_config)
    return analytics.artifacts_by_culture()
```

## Key SQL Queries

```sql
-- Artifacts without images
SELECT a.id, a.title, a.culture
FROM artifactmetadata a
LEFT JOIN artifactmedia m ON a.id = m.artifact_id
WHERE m.artifact_id IS NULL;

-- Color diversity by culture
SELECT a.culture, COUNT(DISTINCT c.color) as color_diversity
FROM artifactmetadata a
JOIN artifactcolors c ON a.id = c.artifact_id
GROUP BY a.culture
ORDER BY color_diversity DESC;

-- Recent artifacts by department
SELECT department, COUNT(*) as count
FROM artifactmetadata
WHERE century LIKE '%21st century%'
GROUP BY department;
```

This skill provides everything needed to build, deploy, and extend the Harvard Art Museums data engineering pipeline.
