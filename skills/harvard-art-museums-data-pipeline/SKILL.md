---
name: harvard-art-museums-data-pipeline
description: Build end-to-end data engineering pipelines for museum artifact data using ETL, SQL analytics, and Streamlit visualization
triggers:
  - how do I build a data pipeline for museum artifacts
  - set up ETL pipeline with Harvard Art Museums API
  - create analytics dashboard with Streamlit and SQL
  - fetch and analyze Harvard museum collection data
  - build data engineering project with museum API
  - create artifact data visualization pipeline
  - implement ETL workflow for art museum data
  - query and visualize museum artifact metadata
---

# Harvard Art Museums Data Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering and analytics application using the Harvard Art Museums API. It demonstrates real-world ETL pipelines, SQL database design, analytics queries, and interactive visualization with Streamlit.

## What It Does

The Harvard Art Museums Data Pipeline enables you to:

- **Extract** artifact data from the Harvard Art Museums API with pagination and rate limiting
- **Transform** nested JSON data into normalized relational tables
- **Load** structured data into SQL databases (MySQL/TiDB Cloud)
- **Analyze** data using predefined SQL queries for insights
- **Visualize** results through interactive Streamlit dashboards with Plotly charts

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

## Installation

### Prerequisites

- Python 3.7+
- MySQL or TiDB Cloud account
- Harvard Art Museums API key (get from https://www.harvardartmuseums.org/collections/api)

### Setup

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

### Environment Variables

Create a `.env` file or use Streamlit secrets:

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

### Streamlit Secrets

Alternatively, configure `.streamlit/secrets.toml`:

```toml
[api]
harvard_key = "your_api_key_here"

[database]
host = "your_database_host"
port = 3306
user = "your_username"
password = "your_password"
database = "harvard_artifacts"
```

### Database Schema

The pipeline creates three main tables:

```sql
-- Artifact metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255),
    period VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accessionyear INT,
    provenance TEXT,
    description TEXT,
    url VARCHAR(500)
);

-- Artifact media/images
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact color information
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

## Running the Application

### Launch Streamlit Dashboard

```bash
streamlit run app.py
```

The dashboard will open at `http://localhost:8501`

## Core Components

### 1. API Data Extraction

```python
import requests
import os

class HarvardAPIClient:
    def __init__(self, api_key):
        self.api_key = api_key
        self.base_url = "https://api.harvardartmuseums.org/object"
    
    def fetch_artifacts(self, size=100, page=1):
        """Fetch artifacts with pagination"""
        params = {
            'apikey': self.api_key,
            'size': size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(self.base_url, params=params)
        response.raise_for_status()
        return response.json()
    
    def fetch_all_pages(self, total_records=1000, page_size=100):
        """Fetch multiple pages of artifacts"""
        all_artifacts = []
        pages = (total_records // page_size) + 1
        
        for page in range(1, pages + 1):
            data = self.fetch_artifacts(size=page_size, page=page)
            all_artifacts.extend(data.get('records', []))
            
            if len(all_artifacts) >= total_records:
                break
        
        return all_artifacts[:total_records]

# Usage
api_key = os.getenv('HARVARD_API_KEY')
client = HarvardAPIClient(api_key)
artifacts = client.fetch_all_pages(total_records=500)
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

class ArtifactETL:
    def __init__(self, db_config):
        self.db_config = db_config
        self.conn = None
    
    def connect_db(self):
        """Establish database connection"""
        self.conn = mysql.connector.connect(**self.db_config)
        return self.conn
    
    def transform_metadata(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Transform artifact metadata"""
        metadata_records = []
        
        for artifact in artifacts:
            record = {
                'id': artifact.get('id'),
                'title': artifact.get('title'),
                'culture': artifact.get('culture'),
                'century': artifact.get('century'),
                'classification': artifact.get('classification'),
                'department': artifact.get('department'),
                'division': artifact.get('division'),
                'dated': artifact.get('dated'),
                'period': artifact.get('period'),
                'technique': artifact.get('technique'),
                'medium': artifact.get('medium'),
                'dimensions': artifact.get('dimensions'),
                'creditline': artifact.get('creditline'),
                'accessionyear': artifact.get('accessionyear'),
                'provenance': artifact.get('provenance'),
                'description': artifact.get('description'),
                'url': artifact.get('url')
            }
            metadata_records.append(record)
        
        return pd.DataFrame(metadata_records)
    
    def transform_media(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Transform media/image data"""
        media_records = []
        
        for artifact in artifacts:
            if artifact.get('primaryimageurl'):
                media = {
                    'artifact_id': artifact.get('id'),
                    'baseimageurl': artifact.get('baseimageurl'),
                    'primaryimageurl': artifact.get('primaryimageurl'),
                    'iiifbaseuri': artifact.get('images', [{}])[0].get('iiifbaseuri') if artifact.get('images') else None
                }
                media_records.append(media)
        
        return pd.DataFrame(media_records)
    
    def transform_colors(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Transform color data"""
        color_records = []
        
        for artifact in artifacts:
            colors = artifact.get('colors', [])
            for color in colors:
                record = {
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'hue': color.get('hue'),
                    'percent': color.get('percent')
                }
                color_records.append(record)
        
        return pd.DataFrame(color_records)
    
    def load_to_db(self, df: pd.DataFrame, table_name: str, if_exists='append'):
        """Load DataFrame to database"""
        from sqlalchemy import create_engine
        
        engine = create_engine(
            f"mysql+mysqlconnector://{self.db_config['user']}:{self.db_config['password']}"
            f"@{self.db_config['host']}:{self.db_config['port']}/{self.db_config['database']}"
        )
        
        df.to_sql(table_name, engine, if_exists=if_exists, index=False)
        print(f"Loaded {len(df)} records to {table_name}")

# Usage
db_config = {
    'host': os.getenv('DB_HOST'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

etl = ArtifactETL(db_config)
etl.connect_db()

# Transform and load
metadata_df = etl.transform_metadata(artifacts)
media_df = etl.transform_media(artifacts)
colors_df = etl.transform_colors(artifacts)

etl.load_to_db(metadata_df, 'artifactmetadata')
etl.load_to_db(media_df, 'artifactmedia')
etl.load_to_db(colors_df, 'artifactcolors')
```

### 3. SQL Analytics Queries

```python
class ArtifactAnalytics:
    def __init__(self, db_config):
        self.db_config = db_config
    
    def execute_query(self, query: str) -> pd.DataFrame:
        """Execute SQL query and return results as DataFrame"""
        conn = mysql.connector.connect(**self.db_config)
        df = pd.read_sql(query, conn)
        conn.close()
        return df
    
    # Sample analytical queries
    
    def artifacts_by_culture(self, limit=10):
        """Top cultures by artifact count"""
        query = f"""
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT {limit}
        """
        return self.execute_query(query)
    
    def artifacts_by_century(self):
        """Distribution by century"""
        query = """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
        """
        return self.execute_query(query)
    
    def media_availability(self):
        """Artifacts with vs without images"""
        query = """
        SELECT 
            CASE WHEN am.artifact_id IS NOT NULL THEN 'With Images' ELSE 'Without Images' END as media_status,
            COUNT(*) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia am ON a.id = am.artifact_id
        GROUP BY media_status
        """
        return self.execute_query(query)
    
    def color_distribution(self, limit=15):
        """Most common colors"""
        query = f"""
        SELECT color, COUNT(*) as frequency
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT {limit}
        """
        return self.execute_query(query)
    
    def artifacts_by_department(self):
        """Department-wise distribution"""
        query = """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
        """
        return self.execute_query(query)

# Usage
analytics = ArtifactAnalytics(db_config)
culture_data = analytics.artifacts_by_culture(limit=10)
print(culture_data)
```

### 4. Streamlit Visualization

```python
import streamlit as st
import plotly.express as px

def create_dashboard():
    st.set_page_config(page_title="Harvard Art Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    st.markdown("---")
    
    # Initialize analytics
    db_config = {
        'host': st.secrets['database']['host'],
        'port': st.secrets['database']['port'],
        'user': st.secrets['database']['user'],
        'password': st.secrets['database']['password'],
        'database': st.secrets['database']['database']
    }
    
    analytics = ArtifactAnalytics(db_config)
    
    # Sidebar for query selection
    st.sidebar.header("Select Analysis")
    analysis_type = st.sidebar.selectbox(
        "Choose Analysis",
        [
            "Artifacts by Culture",
            "Artifacts by Century",
            "Media Availability",
            "Color Distribution",
            "Department Distribution"
        ]
    )
    
    # Execute and display
    if analysis_type == "Artifacts by Culture":
        st.subheader("Top Cultures by Artifact Count")
        df = analytics.artifacts_by_culture(limit=15)
        
        # Display table
        st.dataframe(df)
        
        # Plot
        fig = px.bar(df, x='culture', y='artifact_count', 
                     title='Artifacts by Culture',
                     labels={'culture': 'Culture', 'artifact_count': 'Count'})
        st.plotly_chart(fig, use_container_width=True)
    
    elif analysis_type == "Color Distribution":
        st.subheader("Most Common Colors in Artifacts")
        df = analytics.color_distribution(limit=20)
        
        st.dataframe(df)
        
        fig = px.bar(df, x='color', y='frequency',
                     title='Color Distribution',
                     color='color',
                     labels={'color': 'Color', 'frequency': 'Frequency'})
        st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    create_dashboard()
```

## Common Patterns

### Batch Processing Large Datasets

```python
def batch_insert(etl, artifacts, batch_size=100):
    """Process artifacts in batches"""
    for i in range(0, len(artifacts), batch_size):
        batch = artifacts[i:i+batch_size]
        
        metadata_df = etl.transform_metadata(batch)
        media_df = etl.transform_media(batch)
        colors_df = etl.transform_colors(batch)
        
        etl.load_to_db(metadata_df, 'artifactmetadata')
        etl.load_to_db(media_df, 'artifactmedia')
        etl.load_to_db(colors_df, 'artifactcolors')
        
        print(f"Processed batch {i//batch_size + 1}")
```

### Error Handling in API Calls

```python
import time

def fetch_with_retry(client, max_retries=3, delay=2):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            return client.fetch_artifacts()
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
            wait_time = delay * (2 ** attempt)
            print(f"Retry {attempt + 1} after {wait_time}s")
            time.sleep(wait_time)
```

## Troubleshooting

### Database Connection Issues

```python
# Test connection
try:
    conn = mysql.connector.connect(**db_config)
    print("Database connected successfully")
    conn.close()
except mysql.connector.Error as e:
    print(f"Error: {e}")
```

### API Rate Limiting

Add delays between requests:

```python
import time

for page in range(1, num_pages + 1):
    data = client.fetch_artifacts(page=page)
    time.sleep(0.5)  # 500ms delay between requests
```

### Missing Environment Variables

```python
import os

required_vars = ['HARVARD_API_KEY', 'DB_HOST', 'DB_USER', 'DB_PASSWORD']
missing = [var for var in required_vars if not os.getenv(var)]

if missing:
    raise EnvironmentError(f"Missing environment variables: {', '.join(missing)}")
```

### Data Quality Checks

```python
def validate_artifacts(df):
    """Check data quality"""
    print(f"Total records: {len(df)}")
    print(f"Null values:\n{df.isnull().sum()}")
    print(f"Duplicate IDs: {df['id'].duplicated().sum()}")
    return df
```

This skill enables AI coding agents to help developers build complete data engineering pipelines using the Harvard Art Museums API, from extraction through visualization.
