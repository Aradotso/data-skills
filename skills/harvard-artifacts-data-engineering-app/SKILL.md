---
name: harvard-artifacts-data-engineering-app
description: End-to-end data engineering and analytics application using Harvard Art Museums API with ETL pipelines, SQL analytics, and Streamlit visualization
triggers:
  - build a data pipeline with Harvard Art Museums API
  - create ETL pipeline for museum artifacts data
  - set up Streamlit analytics dashboard for Harvard museums
  - implement artifact data collection and visualization
  - build SQL analytics for museum collections
  - create interactive dashboard for art museum data
  - develop data engineering app with museum API
  - visualize Harvard Art Museums data with Streamlit
---

# Harvard Artifacts Data Engineering App

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI coding agents to build and work with end-to-end data engineering applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Collects artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads nested JSON artifact data into relational SQL tables
- **Database Design**: Structured schema with `artifactmetadata`, `artifactmedia`, and `artifactcolors` tables
- **SQL Analytics**: 20+ predefined analytical queries for insights on artifacts, cultures, centuries, and media
- **Interactive Visualization**: Streamlit dashboard with Plotly charts for real-time data exploration

## Installation

### Prerequisites

```bash
# Python 3.8+
python --version

# Install dependencies
pip install streamlit pandas requests plotly mysql-connector-python sqlalchemy
```

### Environment Setup

Create a `.env` file or set environment variables:

```bash
# Harvard Art Museums API
export HARVARD_API_KEY="your-api-key-here"

# Database Configuration
export DB_HOST="your-db-host"
export DB_PORT="4000"
export DB_USER="your-username"
export DB_PASSWORD="your-password"
export DB_NAME="harvard_artifacts"
```

### Database Setup

```sql
-- Create database
CREATE DATABASE IF NOT EXISTS harvard_artifacts;

-- Create artifactmetadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    dated VARCHAR(200),
    url TEXT,
    creditline TEXT,
    copyright TEXT,
    description TEXT,
    provenance TEXT,
    technique TEXT
);

-- Create artifactmedia table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    total_images INT,
    has_images BOOLEAN,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Create artifactcolors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent DECIMAL(5, 2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Components and API

### 1. API Data Collection

```python
import requests
import os
from typing import Dict, List

class HarvardAPICollector:
    """Collect artifact data from Harvard Art Museums API"""
    
    def __init__(self, api_key: str = None):
        self.api_key = api_key or os.getenv('HARVARD_API_KEY')
        self.base_url = "https://api.harvardartmuseums.org/object"
    
    def fetch_artifacts(self, page: int = 1, size: int = 100) -> Dict:
        """Fetch artifacts with pagination"""
        params = {
            'apikey': self.api_key,
            'page': page,
            'size': size,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(self.base_url, params=params)
        response.raise_for_status()
        return response.json()
    
    def collect_multiple_pages(self, num_pages: int = 5) -> List[Dict]:
        """Collect data from multiple pages"""
        all_artifacts = []
        
        for page in range(1, num_pages + 1):
            try:
                data = self.fetch_artifacts(page=page)
                artifacts = data.get('records', [])
                all_artifacts.extend(artifacts)
                print(f"Collected page {page}: {len(artifacts)} artifacts")
            except Exception as e:
                print(f"Error on page {page}: {e}")
                break
        
        return all_artifacts
```

### 2. ETL Pipeline

```python
import pandas as pd
from typing import List, Dict, Tuple

class ArtifactETL:
    """Extract, Transform, Load artifact data"""
    
    @staticmethod
    def extract_metadata(artifacts: List[Dict]) -> pd.DataFrame:
        """Extract and transform metadata"""
        metadata = []
        
        for artifact in artifacts:
            metadata.append({
                'id': artifact.get('id'),
                'title': artifact.get('title'),
                'culture': artifact.get('culture'),
                'period': artifact.get('period'),
                'century': artifact.get('century'),
                'classification': artifact.get('classification'),
                'department': artifact.get('department'),
                'division': artifact.get('division'),
                'dated': artifact.get('dated'),
                'url': artifact.get('url'),
                'creditline': artifact.get('creditline'),
                'copyright': artifact.get('copyright'),
                'description': artifact.get('description'),
                'provenance': artifact.get('provenance'),
                'technique': artifact.get('technique')
            })
        
        return pd.DataFrame(metadata)
    
    @staticmethod
    def extract_media(artifacts: List[Dict]) -> pd.DataFrame:
        """Extract media information"""
        media = []
        
        for artifact in artifacts:
            media.append({
                'artifact_id': artifact.get('id'),
                'baseimageurl': artifact.get('baseimageurl'),
                'primaryimageurl': artifact.get('primaryimageurl'),
                'iiifbaseuri': artifact.get('iiifbaseuri'),
                'total_images': artifact.get('totalpageviews', 0),
                'has_images': 1 if artifact.get('primaryimageurl') else 0
            })
        
        return pd.DataFrame(media)
    
    @staticmethod
    def extract_colors(artifacts: List[Dict]) -> pd.DataFrame:
        """Extract color data from artifacts"""
        colors = []
        
        for artifact in artifacts:
            artifact_id = artifact.get('id')
            color_data = artifact.get('colors', [])
            
            for color in color_data:
                colors.append({
                    'artifact_id': artifact_id,
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'hue': color.get('hue'),
                    'percent': color.get('percent')
                })
        
        return pd.DataFrame(colors)
    
    @staticmethod
    def transform_all(artifacts: List[Dict]) -> Tuple[pd.DataFrame, pd.DataFrame, pd.DataFrame]:
        """Transform all artifact data into DataFrames"""
        metadata_df = ArtifactETL.extract_metadata(artifacts)
        media_df = ArtifactETL.extract_media(artifacts)
        colors_df = ArtifactETL.extract_colors(artifacts)
        
        return metadata_df, media_df, colors_df
```

### 3. Database Loader

```python
from sqlalchemy import create_engine
import pandas as pd

class DatabaseLoader:
    """Load data into SQL database"""
    
    def __init__(self, connection_string: str = None):
        if connection_string is None:
            connection_string = (
                f"mysql+mysqlconnector://{os.getenv('DB_USER')}:"
                f"{os.getenv('DB_PASSWORD')}@{os.getenv('DB_HOST')}:"
                f"{os.getenv('DB_PORT')}/{os.getenv('DB_NAME')}"
            )
        self.engine = create_engine(connection_string)
    
    def load_metadata(self, df: pd.DataFrame, if_exists: str = 'append'):
        """Load metadata to database"""
        df.to_sql('artifactmetadata', self.engine, 
                  if_exists=if_exists, index=False)
        print(f"Loaded {len(df)} metadata records")
    
    def load_media(self, df: pd.DataFrame, if_exists: str = 'append'):
        """Load media data to database"""
        df.to_sql('artifactmedia', self.engine, 
                  if_exists=if_exists, index=False, 
                  index_label='media_id')
        print(f"Loaded {len(df)} media records")
    
    def load_colors(self, df: pd.DataFrame, if_exists: str = 'append'):
        """Load color data to database"""
        df.to_sql('artifactcolors', self.engine, 
                  if_exists=if_exists, index=False,
                  index_label='color_id')
        print(f"Loaded {len(df)} color records")
    
    def execute_query(self, query: str) -> pd.DataFrame:
        """Execute SQL query and return results"""
        return pd.read_sql(query, self.engine)
```

### 4. Analytical Queries

```python
class AnalyticsQueries:
    """Predefined analytical SQL queries"""
    
    @staticmethod
    def get_queries() -> Dict[str, str]:
        return {
            "Artifacts by Culture": """
                SELECT culture, COUNT(*) as artifact_count
                FROM artifactmetadata
                WHERE culture IS NOT NULL
                GROUP BY culture
                ORDER BY artifact_count DESC
                LIMIT 15
            """,
            
            "Artifacts by Century": """
                SELECT century, COUNT(*) as count
                FROM artifactmetadata
                WHERE century IS NOT NULL
                GROUP BY century
                ORDER BY count DESC
                LIMIT 20
            """,
            
            "Department Distribution": """
                SELECT department, COUNT(*) as total_artifacts
                FROM artifactmetadata
                WHERE department IS NOT NULL
                GROUP BY department
                ORDER BY total_artifacts DESC
            """,
            
            "Media Availability": """
                SELECT 
                    CASE WHEN has_images = 1 THEN 'With Images' 
                         ELSE 'Without Images' END as image_status,
                    COUNT(*) as count
                FROM artifactmedia
                GROUP BY has_images
            """,
            
            "Top Color Distribution": """
                SELECT color, COUNT(*) as usage_count,
                       AVG(percent) as avg_percent
                FROM artifactcolors
                WHERE color IS NOT NULL
                GROUP BY color
                ORDER BY usage_count DESC
                LIMIT 15
            """,
            
            "Classification Analysis": """
                SELECT classification, COUNT(*) as count
                FROM artifactmetadata
                WHERE classification IS NOT NULL
                GROUP BY classification
                ORDER BY count DESC
                LIMIT 15
            """,
            
            "Artifacts with Most Images": """
                SELECT m.title, m.culture, media.total_images
                FROM artifactmetadata m
                JOIN artifactmedia media ON m.id = media.artifact_id
                WHERE media.total_images > 0
                ORDER BY media.total_images DESC
                LIMIT 20
            """,
            
            "Color Spectrum Analysis": """
                SELECT spectrum, COUNT(DISTINCT artifact_id) as artifacts,
                       AVG(percent) as avg_coverage
                FROM artifactcolors
                WHERE spectrum IS NOT NULL
                GROUP BY spectrum
                ORDER BY artifacts DESC
            """
        }
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def create_dashboard():
    """Main Streamlit dashboard"""
    
    st.set_page_config(page_title="Harvard Artifacts Analytics", 
                       layout="wide")
    
    st.title("🏛️ Harvard Art Museums Data Engineering & Analytics")
    st.markdown("---")
    
    # Initialize components
    collector = HarvardAPICollector()
    db_loader = DatabaseLoader()
    queries = AnalyticsQueries.get_queries()
    
    # Sidebar
    st.sidebar.header("⚙️ Configuration")
    
    # Data Collection Section
    if st.sidebar.button("🔄 Collect New Data"):
        with st.spinner("Collecting data from API..."):
            artifacts = collector.collect_multiple_pages(num_pages=3)
            metadata_df, media_df, colors_df = ArtifactETL.transform_all(artifacts)
            
            db_loader.load_metadata(metadata_df)
            db_loader.load_media(media_df)
            db_loader.load_colors(colors_df)
            
            st.success(f"✅ Loaded {len(artifacts)} artifacts")
    
    # Analytics Section
    st.header("📊 SQL Analytics Dashboard")
    
    query_name = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        query = queries[query_name]
        
        with st.spinner("Executing query..."):
            results = db_loader.execute_query(query)
            
            # Display results
            st.subheader("Query Results")
            st.dataframe(results)
            
            # Auto-generate visualization
            if len(results.columns) >= 2:
                fig = px.bar(results, 
                            x=results.columns[0], 
                            y=results.columns[1],
                            title=query_name)
                st.plotly_chart(fig, use_container_width=True)
    
    # Display raw query
    with st.expander("View SQL Query"):
        st.code(queries[query_name], language="sql")

if __name__ == "__main__":
    create_dashboard()
```

## Common Patterns

### Complete ETL Workflow

```python
def run_complete_pipeline():
    """Execute complete ETL pipeline"""
    
    # 1. Collect data
    collector = HarvardAPICollector()
    artifacts = collector.collect_multiple_pages(num_pages=5)
    
    # 2. Transform data
    metadata_df, media_df, colors_df = ArtifactETL.transform_all(artifacts)
    
    # 3. Load to database
    loader = DatabaseLoader()
    loader.load_metadata(metadata_df, if_exists='replace')
    loader.load_media(media_df, if_exists='replace')
    loader.load_colors(colors_df, if_exists='replace')
    
    # 4. Run analytics
    query = """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """
    results = loader.execute_query(query)
    print(results)
```

### Incremental Data Loading

```python
def incremental_load(start_page: int, end_page: int):
    """Load data incrementally without duplicates"""
    
    collector = HarvardAPICollector()
    loader = DatabaseLoader()
    
    for page in range(start_page, end_page + 1):
        artifacts = collector.fetch_artifacts(page=page)['records']
        
        metadata_df, media_df, colors_df = ArtifactETL.transform_all(artifacts)
        
        # Append only new records
        loader.load_metadata(metadata_df, if_exists='append')
        loader.load_media(media_df, if_exists='append')
        loader.load_colors(colors_df, if_exists='append')
```

## Configuration

### Streamlit Config

Create `.streamlit/config.toml`:

```toml
[theme]
primaryColor = "#8B0000"
backgroundColor = "#FFFFFF"
secondaryBackgroundColor = "#F0F0F0"
textColor = "#262730"
font = "sans serif"

[server]
port = 8501
enableCORS = false
```

### Requirements File

```txt
streamlit==1.28.0
pandas==2.1.0
requests==2.31.0
plotly==5.17.0
mysql-connector-python==8.1.0
sqlalchemy==2.0.20
python-dotenv==1.0.0
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(collector, page, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            return collector.fetch_artifacts(page=page)
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
from sqlalchemy import text

def test_connection():
    """Test database connectivity"""
    try:
        loader = DatabaseLoader()
        with loader.engine.connect() as conn:
            result = conn.execute(text("SELECT 1"))
            print("✅ Database connection successful")
            return True
    except Exception as e:
        print(f"❌ Connection failed: {e}")
        return False
```

### Missing Data Handling

```python
def clean_dataframe(df: pd.DataFrame) -> pd.DataFrame:
    """Clean and validate DataFrame before loading"""
    
    # Replace None with empty strings for text fields
    text_cols = df.select_dtypes(include=['object']).columns
    df[text_cols] = df[text_cols].fillna('')
    
    # Truncate long strings to fit database constraints
    for col in text_cols:
        if col in ['title', 'culture', 'period']:
            df[col] = df[col].str[:500]
    
    return df
```

### Streamlit Caching

```python
@st.cache_data(ttl=3600)
def cached_query_execution(query: str):
    """Cache query results for 1 hour"""
    loader = DatabaseLoader()
    return loader.execute_query(query)
```

This skill enables AI agents to build complete data engineering applications with museum APIs, implementing ETL pipelines, SQL analytics, and interactive visualizations using industry-standard tools and patterns.
