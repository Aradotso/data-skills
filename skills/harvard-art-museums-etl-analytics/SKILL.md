---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for museum artifact data
  - create a Harvard Art Museums analytics dashboard
  - set up artifact collection data engineering workflow
  - integrate Harvard Art Museums API with SQL database
  - build a Streamlit app for museum data visualization
  - create an end-to-end data pipeline with museum artifacts
  - analyze Harvard Art Museums collection data with SQL
  - visualize museum artifact metadata with Plotly
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color data into relational SQL databases
- **SQL Analytics**: Execute 20+ predefined analytical queries on structured museum data
- 
- **Interactive Visualization**: Build Streamlit dashboards with Plotly charts for data exploration
- **Database Design**: Implement normalized schema with proper foreign key relationships

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

## Installation

### Prerequisites

- Python 3.8+
- MySQL or TiDB Cloud account
- Harvard Art Museums API key (get from [https://harvardartmuseums.org/collections/api](https://harvardartmuseums.org/collections/api))

### Setup

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_database_host"
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="harvard_artifacts"
```

### Required Dependencies

```python
# requirements.txt
streamlit>=1.20.0
pandas>=1.5.0
requests>=2.28.0
mysql-connector-python>=8.0.0
plotly>=5.11.0
python-dotenv>=0.19.0
```

## Key Components

### 1. API Integration

Fetch artifact data from Harvard Art Museums API:

```python
import requests
import os
from typing import Dict, List

class HarvardAPIClient:
    def __init__(self, api_key: str):
        self.api_key = api_key
        self.base_url = "https://api.harvardartmuseums.org"
    
    def fetch_artifacts(self, page: int = 1, size: int = 100) -> Dict:
        """Fetch artifacts with pagination"""
        endpoint = f"{self.base_url}/object"
        params = {
            "apikey": self.api_key,
            "page": page,
            "size": size,
            "hasimage": 1  # Only artifacts with images
        }
        
        response = requests.get(endpoint, params=params)
        response.raise_for_status()
        return response.json()
    
    def collect_artifacts_batch(self, total_records: int = 500) -> List[Dict]:
        """Collect multiple pages of artifacts"""
        artifacts = []
        page = 1
        size = 100
        
        while len(artifacts) < total_records:
            data = self.fetch_artifacts(page=page, size=size)
            artifacts.extend(data.get("records", []))
            
            if not data.get("info", {}).get("next"):
                break
            
            page += 1
        
        return artifacts[:total_records]

# Usage
api_key = os.getenv("HARVARD_API_KEY")
client = HarvardAPIClient(api_key)
artifacts = client.collect_artifacts_batch(total_records=500)
```

### 2. ETL Pipeline

Transform nested JSON into relational tables:

```python
import pandas as pd
from typing import List, Dict, Tuple

class ArtifactETL:
    def __init__(self, artifacts: List[Dict]):
        self.artifacts = artifacts
    
    def extract_metadata(self) -> pd.DataFrame:
        """Extract core artifact metadata"""
        metadata = []
        
        for artifact in self.artifacts:
            metadata.append({
                "artifact_id": artifact.get("id"),
                "title": artifact.get("title"),
                "culture": artifact.get("culture"),
                "century": artifact.get("century"),
                "classification": artifact.get("classification"),
                "department": artifact.get("department"),
                "division": artifact.get("division"),
                "dated": artifact.get("dated"),
                "period": artifact.get("period"),
                "technique": artifact.get("technique"),
                "medium": artifact.get("medium"),
                "dimensions": artifact.get("dimensions"),
                "description": artifact.get("description")
            })
        
        return pd.DataFrame(metadata)
    
    def extract_media(self) -> pd.DataFrame:
        """Extract media/image data"""
        media_records = []
        
        for artifact in self.artifacts:
            artifact_id = artifact.get("id")
            images = artifact.get("images", [])
            
            for img in images:
                media_records.append({
                    "artifact_id": artifact_id,
                    "image_id": img.get("imageid"),
                    "base_url": img.get("baseimageurl"),
                    "width": img.get("width"),
                    "height": img.get("height"),
                    "format": img.get("format"),
                    "copyright": img.get("copyright")
                })
        
        return pd.DataFrame(media_records)
    
    def extract_colors(self) -> pd.DataFrame:
        """Extract color palette data"""
        color_records = []
        
        for artifact in self.artifacts:
            artifact_id = artifact.get("id")
            colors = artifact.get("colors", [])
            
            for color in colors:
                color_records.append({
                    "artifact_id": artifact_id,
                    "color_hex": color.get("hex"),
                    "color_name": color.get("color"),
                    "percentage": color.get("percent"),
                    "hue": color.get("hue"),
                    "saturation": color.get("saturation")
                })
        
        return pd.DataFrame(color_records)
    
    def run_etl(self) -> Tuple[pd.DataFrame, pd.DataFrame, pd.DataFrame]:
        """Execute full ETL pipeline"""
        metadata_df = self.extract_metadata()
        media_df = self.extract_media()
        colors_df = self.extract_colors()
        
        return metadata_df, media_df, colors_df

# Usage
etl = ArtifactETL(artifacts)
metadata_df, media_df, colors_df = etl.run_etl()
```

### 3. SQL Database Loading

Create schema and load data:

```python
import mysql.connector
from mysql.connector import Error
import pandas as pd
import os

class DatabaseLoader:
    def __init__(self):
        self.connection = mysql.connector.connect(
            host=os.getenv("DB_HOST"),
            user=os.getenv("DB_USER"),
            password=os.getenv("DB_PASSWORD"),
            database=os.getenv("DB_NAME")
        )
        self.cursor = self.connection.cursor()
    
    def create_schema(self):
        """Create database tables"""
        
        # Metadata table
        self.cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactmetadata (
                artifact_id INT PRIMARY KEY,
                title VARCHAR(500),
                culture VARCHAR(255),
                century VARCHAR(100),
                classification VARCHAR(255),
                department VARCHAR(255),
                division VARCHAR(255),
                dated VARCHAR(255),
                period VARCHAR(255),
                technique TEXT,
                medium TEXT,
                dimensions TEXT,
                description TEXT
            )
        """)
        
        # Media table
        self.cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactmedia (
                id INT AUTO_INCREMENT PRIMARY KEY,
                artifact_id INT,
                image_id INT,
                base_url VARCHAR(500),
                width INT,
                height INT,
                format VARCHAR(50),
                copyright TEXT,
                FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
            )
        """)
        
        # Colors table
        self.cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactcolors (
                id INT AUTO_INCREMENT PRIMARY KEY,
                artifact_id INT,
                color_hex VARCHAR(10),
                color_name VARCHAR(100),
                percentage FLOAT,
                hue VARCHAR(50),
                saturation FLOAT,
                FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
            )
        """)
        
        self.connection.commit()
    
    def load_dataframe(self, df: pd.DataFrame, table_name: str):
        """Batch insert DataFrame into SQL table"""
        
        # Replace NaN with None for SQL NULL
        df = df.where(pd.notnull(df), None)
        
        columns = ", ".join(df.columns)
        placeholders = ", ".join(["%s"] * len(df.columns))
        query = f"INSERT INTO {table_name} ({columns}) VALUES ({placeholders})"
        
        # Batch insert
        data = [tuple(row) for row in df.values]
        self.cursor.executemany(query, data)
        self.connection.commit()
    
    def close(self):
        self.cursor.close()
        self.connection.close()

# Usage
loader = DatabaseLoader()
loader.create_schema()
loader.load_dataframe(metadata_df, "artifactmetadata")
loader.load_dataframe(media_df, "artifactmedia")
loader.load_dataframe(colors_df, "artifactcolors")
loader.close()
```

### 4. SQL Analytics Queries

Predefined analytical queries:

```python
class ArtifactAnalytics:
    def __init__(self, connection):
        self.connection = connection
    
    def get_artifacts_by_culture(self) -> pd.DataFrame:
        """Count artifacts by culture"""
        query = """
            SELECT culture, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE culture IS NOT NULL
            GROUP BY culture
            ORDER BY artifact_count DESC
            LIMIT 20
        """
        return pd.read_sql(query, self.connection)
    
    def get_century_distribution(self) -> pd.DataFrame:
        """Artifact distribution by century"""
        query = """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL
            GROUP BY century
            ORDER BY count DESC
        """
        return pd.read_sql(query, self.connection)
    
    def get_top_colors(self) -> pd.DataFrame:
        """Most common colors across artifacts"""
        query = """
            SELECT color_name, COUNT(*) as frequency, AVG(percentage) as avg_percentage
            FROM artifactcolors
            WHERE color_name IS NOT NULL
            GROUP BY color_name
            ORDER BY frequency DESC
            LIMIT 15
        """
        return pd.read_sql(query, self.connection)
    
    def get_media_stats(self) -> pd.DataFrame:
        """Image format statistics"""
        query = """
            SELECT format, COUNT(*) as image_count,
                   AVG(width) as avg_width, AVG(height) as avg_height
            FROM artifactmedia
            WHERE format IS NOT NULL
            GROUP BY format
        """
        return pd.read_sql(query, self.connection)
    
    def get_department_breakdown(self) -> pd.DataFrame:
        """Artifacts by department"""
        query = """
            SELECT department, COUNT(*) as total_artifacts
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY total_artifacts DESC
        """
        return pd.read_sql(query, self.connection)

# Usage
analytics = ArtifactAnalytics(connection)
culture_df = analytics.get_artifacts_by_culture()
```

### 5. Streamlit Dashboard

Build interactive analytics dashboard:

```python
import streamlit as st
import plotly.express as px
import pandas as pd

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums - Analytics Dashboard")
    st.markdown("End-to-end data engineering and analytics pipeline")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    # Initialize connection
    connection = mysql.connector.connect(
        host=os.getenv("DB_HOST"),
        user=os.getenv("DB_USER"),
        password=os.getenv("DB_PASSWORD"),
        database=os.getenv("DB_NAME")
    )
    
    analytics = ArtifactAnalytics(connection)
    
    # Query selection
    query_options = {
        "Artifacts by Culture": analytics.get_artifacts_by_culture,
        "Century Distribution": analytics.get_century_distribution,
        "Top Colors": analytics.get_top_colors,
        "Media Statistics": analytics.get_media_stats,
        "Department Breakdown": analytics.get_department_breakdown
    }
    
    selected_query = st.sidebar.selectbox("Select Analysis", list(query_options.keys()))
    
    # Execute query
    if st.sidebar.button("Run Query"):
        with st.spinner("Executing query..."):
            result_df = query_options[selected_query]()
            
            # Display results
            st.subheader(f"Results: {selected_query}")
            st.dataframe(result_df)
            
            # Visualization
            if len(result_df) > 0:
                st.subheader("Visualization")
                
                # Auto-generate chart based on data
                if len(result_df.columns) >= 2:
                    fig = px.bar(
                        result_df,
                        x=result_df.columns[0],
                        y=result_df.columns[1],
                        title=selected_query
                    )
                    st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Full Pipeline Execution

```python
import os
from dotenv import load_dotenv

def run_full_pipeline():
    """Execute complete ETL and analytics pipeline"""
    
    # Load environment variables
    load_dotenv()
    
    # Step 1: Fetch data from API
    api_key = os.getenv("HARVARD_API_KEY")
    client = HarvardAPIClient(api_key)
    artifacts = client.collect_artifacts_batch(total_records=1000)
    
    # Step 2: Run ETL
    etl = ArtifactETL(artifacts)
    metadata_df, media_df, colors_df = etl.run_etl()
    
    # Step 3: Load to database
    loader = DatabaseLoader()
    loader.create_schema()
    loader.load_dataframe(metadata_df, "artifactmetadata")
    loader.load_dataframe(media_df, "artifactmedia")
    loader.load_dataframe(colors_df, "artifactcolors")
    loader.close()
    
    print("Pipeline completed successfully!")

run_full_pipeline()
```

### Incremental Data Loading

```python
def incremental_load(last_artifact_id: int):
    """Load only new artifacts since last run"""
    
    connection = mysql.connector.connect(...)
    cursor = connection.cursor()
    
    # Fetch new artifacts
    client = HarvardAPIClient(os.getenv("HARVARD_API_KEY"))
    artifacts = client.collect_artifacts_batch(total_records=100)
    
    # Filter artifacts newer than last ID
    new_artifacts = [a for a in artifacts if a.get("id", 0) > last_artifact_id]
    
    if new_artifacts:
        etl = ArtifactETL(new_artifacts)
        metadata_df, media_df, colors_df = etl.run_etl()
        
        loader = DatabaseLoader()
        loader.load_dataframe(metadata_df, "artifactmetadata")
        loader.load_dataframe(media_df, "artifactmedia")
        loader.load_dataframe(colors_df, "artifactcolors")
        loader.close()
```

## Configuration

### Environment Variables

Create a `.env` file:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_from_harvard

# Database Configuration
DB_HOST=your_mysql_or_tidb_host
DB_PORT=3306
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts

# Application Settings
ARTIFACTS_PER_BATCH=100
MAX_ARTIFACTS=1000
```

### Streamlit Configuration

Create `.streamlit/config.toml`:

```toml
[theme]
primaryColor = "#A51C30"
backgroundColor = "#FFFFFF"
secondaryBackgroundColor = "#F0F2F6"
textColor = "#262730"
font = "sans serif"

[server]
port = 8501
enableCORS = false
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(client, page, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            return client.fetch_artifacts(page=page)
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:
                wait_time = 2 ** attempt
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
def get_connection_with_retry():
    """Retry database connection"""
    max_attempts = 5
    for attempt in range(max_attempts):
        try:
            connection = mysql.connector.connect(
                host=os.getenv("DB_HOST"),
                user=os.getenv("DB_USER"),
                password=os.getenv("DB_PASSWORD"),
                database=os.getenv("DB_NAME")
            )
            return connection
        except Error as e:
            if attempt < max_attempts - 1:
                time.sleep(2 ** attempt)
            else:
                raise
```

### Handling Missing Data

```python
def clean_artifact_data(artifact: Dict) -> Dict:
    """Clean and validate artifact data"""
    
    # Ensure required fields exist
    required_fields = ["id", "title"]
    for field in required_fields:
        if field not in artifact or artifact[field] is None:
            artifact[field] = "Unknown"
    
    # Truncate long text fields
    if artifact.get("description") and len(artifact["description"]) > 5000:
        artifact["description"] = artifact["description"][:5000]
    
    return artifact
```

## Running the Application

```bash
# Run Streamlit dashboard
streamlit run app.py

# Execute ETL pipeline only
python etl_pipeline.py

# Run analytics queries
python analytics_queries.py
```

This skill provides AI agents with comprehensive knowledge to help developers build end-to-end data engineering pipelines using museum data, from API integration through visualization.
