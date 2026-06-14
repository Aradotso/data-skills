---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Streamlit, SQL, and Plotly
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - create a data engineering project with art museum data
  - set up analytics dashboard for Harvard artifacts collection
  - extract and visualize Harvard museum artifact data
  - build streamlit app for museum collection analytics
  - implement SQL analytics on Harvard art data
  - create interactive visualizations for museum artifacts
  - design relational database for art collection metadata
---

# Harvard Artifacts Collection ETL & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

This project demonstrates end-to-end data engineering: extracting artifact data from Harvard Art Museums API, transforming it through ETL pipelines, loading into SQL databases, and visualizing insights through Streamlit dashboards.

## What It Does

- **API Integration**: Fetches artifact metadata from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON into normalized relational tables
- **SQL Storage**: Stores data in MySQL/TiDB Cloud with proper schema design
- **Analytics**: Executes 20+ predefined SQL queries for insights
- **Visualization**: Interactive Plotly dashboards via Streamlit

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests sqlalchemy pymysql plotly python-dotenv
```

## Configuration

### API Key Setup

Get your Harvard Art Museums API key from https://harvardartmuseums.org/collections/api

```python
# .env file
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Database Schema

The project uses three main tables:

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    url TEXT,
    primary_image_url TEXT
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    media_url TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
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

def fetch_artifacts(api_key, size=100, page=1):
    """Fetch artifacts from Harvard Art Museums API"""
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'size': size,
        'page': page,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, size=100, page=1)
artifacts = data.get('records', [])
```

### 2. ETL Pipeline

```python
import pandas as pd
from sqlalchemy import create_engine

def transform_artifact_data(artifacts):
    """Transform nested JSON to normalized DataFrames"""
    
    # Metadata table
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Untitled'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url'),
            'primary_image_url': artifact.get('primaryimageurl')
        }
        metadata_records.append(metadata)
        
        # Extract media (images)
        if 'images' in artifact and artifact['images']:
            for img in artifact['images']:
                media = {
                    'artifact_id': artifact.get('id'),
                    'media_type': 'image',
                    'media_url': img.get('baseimageurl')
                }
                media_records.append(media)
        
        # Extract colors
        if 'colors' in artifact and artifact['colors']:
            for color in artifact['colors']:
                color_record = {
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color'),
                    'percentage': color.get('percent')
                }
                color_records.append(color_record)
    
    return {
        'metadata': pd.DataFrame(metadata_records),
        'media': pd.DataFrame(media_records),
        'colors': pd.DataFrame(color_records)
    }

# Transform data
transformed = transform_artifact_data(artifacts)
```

### 3. Load to Database

```python
def load_to_database(dataframes, connection_string):
    """Load transformed data into SQL database"""
    
    engine = create_engine(connection_string)
    
    # Load metadata (main table first)
    dataframes['metadata'].to_sql(
        'artifactmetadata',
        con=engine,
        if_exists='append',
        index=False,
        method='multi'
    )
    
    # Load media
    if not dataframes['media'].empty:
        dataframes['media'].to_sql(
            'artifactmedia',
            con=engine,
            if_exists='append',
            index=False,
            method='multi'
        )
    
    # Load colors
    if not dataframes['colors'].empty:
        dataframes['colors'].to_sql(
            'artifactcolors',
            con=engine,
            if_exists='append',
            index=False,
            method='multi'
        )
    
    print(f"Loaded {len(dataframes['metadata'])} artifacts to database")

# Connection string
connection_string = f"mysql+pymysql://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}@{os.getenv('DB_HOST')}:{os.getenv('DB_PORT')}/{os.getenv('DB_NAME')}"

# Load data
load_to_database(transformed, connection_string)
```

### 4. Analytical Queries

```python
def get_artifacts_by_century():
    """Query artifacts distribution by century"""
    query = """
    SELECT century, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY artifact_count DESC
    LIMIT 10
    """
    return pd.read_sql(query, engine)

def get_color_distribution():
    """Query color usage across artifacts"""
    query = """
    SELECT color, 
           COUNT(DISTINCT artifact_id) as artifact_count,
           AVG(percentage) as avg_percentage
    FROM artifactcolors
    GROUP BY color
    ORDER BY artifact_count DESC
    LIMIT 15
    """
    return pd.read_sql(query, engine)

def get_artifacts_with_images():
    """Query artifacts with image availability"""
    query = """
    SELECT 
        CASE 
            WHEN primary_image_url IS NOT NULL THEN 'Has Image'
            ELSE 'No Image'
        END as image_status,
        COUNT(*) as count
    FROM artifactmetadata
    GROUP BY image_status
    """
    return pd.read_sql(query, engine)

def get_department_statistics():
    """Query artifacts by department"""
    query = """
    SELECT department, 
           COUNT(*) as total_artifacts,
           COUNT(DISTINCT culture) as unique_cultures
    FROM artifactmetadata
    WHERE department IS NOT NULL
    GROUP BY department
    ORDER BY total_artifacts DESC
    """
    return pd.read_sql(query, engine)
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px
from sqlalchemy import create_engine
import pandas as pd

# App configuration
st.set_page_config(
    page_title="Harvard Artifacts Analytics",
    page_icon="🏛️",
    layout="wide"
)

st.title("🏛️ Harvard Art Museums Collection Analytics")

# Database connection
@st.cache_resource
def get_engine():
    connection_string = f"mysql+pymysql://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}@{os.getenv('DB_HOST')}:{os.getenv('DB_PORT')}/{os.getenv('DB_NAME')}"
    return create_engine(connection_string)

engine = get_engine()

# Sidebar - Query Selection
st.sidebar.header("Analytics Queries")
query_options = {
    "Artifacts by Century": get_artifacts_by_century,
    "Color Distribution": get_color_distribution,
    "Image Availability": get_artifacts_with_images,
    "Department Statistics": get_department_statistics
}

selected_query = st.sidebar.selectbox(
    "Select Analysis",
    options=list(query_options.keys())
)

# Execute query
if st.sidebar.button("Run Analysis"):
    with st.spinner("Executing query..."):
        df = query_options[selected_query]()
        
        # Display results
        st.subheader(f"Results: {selected_query}")
        st.dataframe(df, use_container_width=True)
        
        # Visualize
        if len(df.columns) >= 2:
            fig = px.bar(
                df,
                x=df.columns[0],
                y=df.columns[1],
                title=selected_query
            )
            st.plotly_chart(fig, use_container_width=True)

# ETL Section
st.sidebar.header("ETL Operations")
if st.sidebar.button("Fetch New Data"):
    with st.spinner("Fetching from API..."):
        api_key = os.getenv('HARVARD_API_KEY')
        data = fetch_artifacts(api_key, size=50, page=1)
        artifacts = data.get('records', [])
        
        # Transform and load
        transformed = transform_artifact_data(artifacts)
        load_to_database(transformed, str(engine.url))
        
        st.success(f"Successfully loaded {len(artifacts)} new artifacts!")
```

### 6. Complete ETL Script

```python
# etl_pipeline.py
import os
import requests
import pandas as pd
from sqlalchemy import create_engine
from dotenv import load_dotenv
import time

load_dotenv()

class HarvardArtifactsETL:
    def __init__(self, api_key, db_connection_string):
        self.api_key = api_key
        self.engine = create_engine(db_connection_string)
        self.base_url = "https://api.harvardartmuseums.org/object"
    
    def extract(self, total_pages=5, size=100):
        """Extract data from API with pagination"""
        all_artifacts = []
        
        for page in range(1, total_pages + 1):
            print(f"Fetching page {page}...")
            
            params = {
                'apikey': self.api_key,
                'size': size,
                'page': page,
                'hasimage': 1
            }
            
            response = requests.get(self.base_url, params=params)
            
            if response.status_code == 200:
                data = response.json()
                all_artifacts.extend(data.get('records', []))
                time.sleep(0.5)  # Rate limiting
            else:
                print(f"Error on page {page}: {response.status_code}")
        
        return all_artifacts
    
    def transform(self, artifacts):
        """Transform artifacts into normalized tables"""
        metadata_records = []
        media_records = []
        color_records = []
        
        for artifact in artifacts:
            metadata_records.append({
                'id': artifact.get('id'),
                'title': artifact.get('title', 'Untitled')[:500],
                'culture': artifact.get('culture'),
                'century': artifact.get('century'),
                'classification': artifact.get('classification'),
                'department': artifact.get('department'),
                'dated': artifact.get('dated'),
                'url': artifact.get('url'),
                'primary_image_url': artifact.get('primaryimageurl')
            })
            
            for img in artifact.get('images', []):
                media_records.append({
                    'artifact_id': artifact.get('id'),
                    'media_type': 'image',
                    'media_url': img.get('baseimageurl')
                })
            
            for color in artifact.get('colors', []):
                color_records.append({
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color'),
                    'percentage': color.get('percent')
                })
        
        return {
            'metadata': pd.DataFrame(metadata_records),
            'media': pd.DataFrame(media_records),
            'colors': pd.DataFrame(color_records)
        }
    
    def load(self, dataframes):
        """Load data into database"""
        with self.engine.begin() as conn:
            dataframes['metadata'].to_sql(
                'artifactmetadata',
                con=conn,
                if_exists='append',
                index=False
            )
            
            if not dataframes['media'].empty:
                dataframes['media'].to_sql(
                    'artifactmedia',
                    con=conn,
                    if_exists='append',
                    index=False
                )
            
            if not dataframes['colors'].empty:
                dataframes['colors'].to_sql(
                    'artifactcolors',
                    con=conn,
                    if_exists='append',
                    index=False
                )
    
    def run_pipeline(self, total_pages=5):
        """Execute full ETL pipeline"""
        print("Starting ETL pipeline...")
        
        # Extract
        print("Extracting data from API...")
        artifacts = self.extract(total_pages=total_pages)
        print(f"Extracted {len(artifacts)} artifacts")
        
        # Transform
        print("Transforming data...")
        dataframes = self.transform(artifacts)
        print(f"Transformed into {len(dataframes['metadata'])} metadata records")
        
        # Load
        print("Loading into database...")
        self.load(dataframes)
        print("ETL pipeline completed successfully!")

# Usage
if __name__ == "__main__":
    api_key = os.getenv('HARVARD_API_KEY')
    db_connection = f"mysql+pymysql://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}@{os.getenv('DB_HOST')}:{os.getenv('DB_PORT')}/{os.getenv('DB_NAME')}"
    
    etl = HarvardArtifactsETL(api_key, db_connection)
    etl.run_pipeline(total_pages=5)
```

## Running the Application

```bash
# Run Streamlit dashboard
streamlit run app.py

# Run ETL pipeline separately
python etl_pipeline.py
```

## Common Patterns

### Incremental Loading

```python
def get_max_artifact_id(engine):
    """Get the highest artifact ID in database"""
    query = "SELECT MAX(id) as max_id FROM artifactmetadata"
    result = pd.read_sql(query, engine)
    return result['max_id'].iloc[0] or 0

# Only fetch artifacts newer than existing ones
max_id = get_max_artifact_id(engine)
# Use max_id in API filter
```

### Error Handling

```python
def safe_extract(api_key, page, retries=3):
    """Extract with retry logic"""
    for attempt in range(retries):
        try:
            response = requests.get(
                base_url,
                params={'apikey': api_key, 'page': page},
                timeout=10
            )
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            if attempt == retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

## Troubleshooting

**API Rate Limiting**: Add `time.sleep(0.5)` between requests

**Database Connection Issues**: Verify credentials in `.env` and check firewall rules

**Empty DataFrames**: Check API response structure, Harvard API may return null fields

**Primary Key Conflicts**: Use `if_exists='replace'` or check for duplicates before inserting

**Memory Issues**: Process data in smaller batches (reduce page size or total pages)
