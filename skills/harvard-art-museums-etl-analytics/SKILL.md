---
name: harvard-art-museums-etl-analytics
description: End-to-end ETL pipeline and analytics app using Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifact data
  - set up data pipeline with Streamlit and SQL
  - extract and analyze Harvard Art Museums API data
  - implement artifact data collection and visualization
  - build museum data engineering application
  - create SQL analytics for art collections
  - design ETL workflow for museum artifacts
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides a complete data engineering pipeline for collecting, processing, and analyzing artifact data from the Harvard Art Museums API. It demonstrates real-world ETL patterns, relational database design, SQL analytics, and interactive visualization using Streamlit.

The application architecture follows: **API → ETL → SQL → Analytics → Visualization**

Key capabilities:
- API data extraction with pagination and rate limiting
- Transform nested JSON into normalized relational tables
- SQL database storage (MySQL/TiDB Cloud)
- 20+ predefined analytical queries
- Interactive Plotly visualizations

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_database_host"
export DB_USER="your_db_user"
export DB_PASSWORD="your_db_password"
export DB_NAME="harvard_artifacts"

# Run the application
streamlit run app.py
```

### Required Dependencies

```python
# requirements.txt
streamlit>=1.20.0
pandas>=1.5.0
requests>=2.28.0
mysql-connector-python>=8.0.0
plotly>=5.10.0
python-dotenv>=0.20.0
```

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_from_harvard

# Database Configuration
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts

# API Settings
API_BASE_URL=https://api.harvardartmuseums.org
RECORDS_PER_PAGE=100
MAX_PAGES=10
```

### Database Setup

```sql
-- Create database
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE IF NOT EXISTS artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    technique VARCHAR(255),
    dated VARCHAR(255),
    accession_number VARCHAR(100),
    url TEXT,
    INDEX idx_culture (culture),
    INDEX idx_century (century),
    INDEX idx_classification (classification)
);

-- Artifact media table
CREATE TABLE IF NOT EXISTS artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    media_url TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id),
    INDEX idx_artifact (artifact_id)
);

-- Artifact colors table
CREATE TABLE IF NOT EXISTS artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percent DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id),
    INDEX idx_artifact (artifact_id)
);
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from typing import List, Dict

def fetch_artifacts(api_key: str, page: int = 1, size: int = 100) -> Dict:
    """
    Fetch artifacts from Harvard Art Museums API with pagination.
    
    Args:
        api_key: Harvard API key
        page: Page number to fetch
        size: Records per page
    
    Returns:
        Dict containing records and pagination info
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        "apikey": api_key,
        "page": page,
        "size": size,
        "hasimage": 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    
    return response.json()

def collect_all_artifacts(api_key: str, max_pages: int = 10) -> List[Dict]:
    """
    Collect multiple pages of artifact data.
    
    Args:
        api_key: Harvard API key
        max_pages: Maximum number of pages to collect
    
    Returns:
        List of artifact records
    """
    all_records = []
    
    for page in range(1, max_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, page=page)
        
        records = data.get("records", [])
        all_records.extend(records)
        
        # Check if more pages available
        info = data.get("info", {})
        if page >= info.get("pages", 0):
            break
    
    return all_records
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

class ArtifactETL:
    """ETL pipeline for Harvard Art Museums data."""
    
    def __init__(self, db_config: Dict):
        """Initialize with database configuration."""
        self.db_config = db_config
        self.conn = None
    
    def connect_db(self):
        """Establish database connection."""
        self.conn = mysql.connector.connect(**self.db_config)
        return self.conn
    
    def extract_metadata(self, records: List[Dict]) -> pd.DataFrame:
        """
        Extract artifact metadata from API records.
        
        Args:
            records: List of artifact records from API
        
        Returns:
            DataFrame with artifact metadata
        """
        metadata = []
        
        for record in records:
            metadata.append({
                "id": record.get("id"),
                "title": record.get("title", "")[:500],
                "culture": record.get("culture", "")[:255],
                "period": record.get("period", "")[:255],
                "century": record.get("century", "")[:100],
                "classification": record.get("classification", "")[:255],
                "department": record.get("department", "")[:255],
                "technique": record.get("technique", "")[:255],
                "dated": record.get("dated", "")[:255],
                "accession_number": record.get("accessionyear", "")[:100],
                "url": record.get("url", "")
            })
        
        return pd.DataFrame(metadata)
    
    def extract_media(self, records: List[Dict]) -> pd.DataFrame:
        """Extract media/image data from artifacts."""
        media_data = []
        
        for record in records:
            artifact_id = record.get("id")
            images = record.get("images", [])
            
            for image in images:
                media_data.append({
                    "artifact_id": artifact_id,
                    "media_type": "image",
                    "media_url": image.get("baseimageurl", "")
                })
        
        return pd.DataFrame(media_data)
    
    def extract_colors(self, records: List[Dict]) -> pd.DataFrame:
        """Extract color data from artifacts."""
        color_data = []
        
        for record in records:
            artifact_id = record.get("id")
            colors = record.get("colors", [])
            
            for color in colors:
                color_data.append({
                    "artifact_id": artifact_id,
                    "color_hex": color.get("hex", ""),
                    "color_percent": color.get("percent", 0.0)
                })
        
        return pd.DataFrame(color_data)
    
    def load_to_sql(self, df: pd.DataFrame, table_name: str, if_exists: str = "append"):
        """
        Load DataFrame to SQL table.
        
        Args:
            df: DataFrame to load
            table_name: Target table name
            if_exists: How to behave if table exists ('append', 'replace', 'fail')
        """
        if self.conn is None:
            self.connect_db()
        
        cursor = self.conn.cursor()
        
        # Prepare insert statement
        cols = ", ".join(df.columns)
        placeholders = ", ".join(["%s"] * len(df.columns))
        
        if if_exists == "replace":
            sql = f"REPLACE INTO {table_name} ({cols}) VALUES ({placeholders})"
        else:
            sql = f"INSERT INTO {table_name} ({cols}) VALUES ({placeholders})"
        
        # Batch insert
        data = [tuple(row) for row in df.values]
        cursor.executemany(sql, data)
        self.conn.commit()
        
        print(f"Loaded {len(df)} records into {table_name}")
    
    def run_etl(self, records: List[Dict]):
        """Run complete ETL pipeline."""
        print("Starting ETL pipeline...")
        
        # Extract
        print("Extracting metadata...")
        metadata_df = self.extract_metadata(records)
        
        print("Extracting media...")
        media_df = self.extract_media(records)
        
        print("Extracting colors...")
        colors_df = self.extract_colors(records)
        
        # Load
        print("Loading to database...")
        self.load_to_sql(metadata_df, "artifactmetadata", if_exists="replace")
        self.load_to_sql(media_df, "artifactmedia")
        self.load_to_sql(colors_df, "artifactcolors")
        
        print("ETL pipeline completed!")
```

### 3. SQL Analytics Queries

```python
class ArtifactAnalytics:
    """Predefined analytical queries for artifact data."""
    
    QUERIES = {
        "artifacts_by_culture": """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture IS NOT NULL AND culture != ''
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 20
        """,
        
        "artifacts_by_century": """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL AND century != ''
            GROUP BY century
            ORDER BY count DESC
        """,
        
        "artifacts_by_classification": """
            SELECT classification, COUNT(*) as count
            FROM artifactmetadata
            WHERE classification IS NOT NULL AND classification != ''
            GROUP BY classification
            ORDER BY count DESC
            LIMIT 15
        """,
        
        "artifacts_with_images": """
            SELECT 
                CASE WHEN COUNT(m.id) > 0 THEN 'Has Images' ELSE 'No Images' END as image_status,
                COUNT(DISTINCT a.id) as artifact_count
            FROM artifactmetadata a
            LEFT JOIN artifactmedia m ON a.id = m.artifact_id
            GROUP BY image_status
        """,
        
        "top_colors": """
            SELECT color_hex, COUNT(*) as usage_count, AVG(color_percent) as avg_percent
            FROM artifactcolors
            GROUP BY color_hex
            ORDER BY usage_count DESC
            LIMIT 10
        """,
        
        "artifacts_by_department": """
            SELECT department, COUNT(*) as count
            FROM artifactmetadata
            WHERE department IS NOT NULL AND department != ''
            GROUP BY department
            ORDER BY count DESC
        """
    }
    
    @staticmethod
    def execute_query(conn, query_name: str) -> pd.DataFrame:
        """Execute a predefined query and return results."""
        query = ArtifactAnalytics.QUERIES.get(query_name)
        if not query:
            raise ValueError(f"Query '{query_name}' not found")
        
        return pd.read_sql(query, conn)
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px
import os
from dotenv import load_dotenv

# Load environment variables
load_dotenv()

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    api_key = os.getenv("HARVARD_API_KEY")
    db_config = {
        "host": os.getenv("DB_HOST"),
        "user": os.getenv("DB_USER"),
        "password": os.getenv("DB_PASSWORD"),
        "database": os.getenv("DB_NAME")
    }
    
    # Data collection section
    if st.sidebar.button("Collect New Data"):
        with st.spinner("Fetching data from API..."):
            records = collect_all_artifacts(api_key, max_pages=5)
            st.success(f"Collected {len(records)} artifacts")
            
            etl = ArtifactETL(db_config)
            etl.run_etl(records)
            st.success("ETL completed successfully!")
    
    # Analytics section
    st.header("📊 Analytics")
    
    query_choice = st.selectbox(
        "Select Analysis",
        list(ArtifactAnalytics.QUERIES.keys())
    )
    
    if st.button("Run Query"):
        conn = mysql.connector.connect(**db_config)
        
        try:
            df = ArtifactAnalytics.execute_query(conn, query_choice)
            
            # Display results
            st.subheader("Query Results")
            st.dataframe(df)
            
            # Visualize if applicable
            if len(df.columns) >= 2 and df.shape[0] > 0:
                fig = px.bar(
                    df,
                    x=df.columns[0],
                    y=df.columns[1],
                    title=query_choice.replace("_", " ").title()
                )
                st.plotly_chart(fig, use_container_width=True)
        
        finally:
            conn.close()

if __name__ == "__main__":
    main()
```

## Common Patterns

### Pattern 1: Incremental Data Loading

```python
def incremental_load(etl: ArtifactETL, api_key: str):
    """Load only new artifacts not in database."""
    # Get max ID from database
    conn = etl.connect_db()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    max_id = cursor.fetchone()[0] or 0
    
    # Fetch only newer records
    records = fetch_artifacts(api_key)
    new_records = [r for r in records if r.get("id", 0) > max_id]
    
    if new_records:
        etl.run_etl(new_records)
        print(f"Loaded {len(new_records)} new artifacts")
    else:
        print("No new artifacts found")
```

### Pattern 2: Custom Query Builder

```python
def build_custom_query(
    table: str = "artifactmetadata",
    columns: List[str] = ["*"],
    filters: Dict = None,
    group_by: str = None,
    order_by: str = None,
    limit: int = None
) -> str:
    """Build SQL query dynamically."""
    query = f"SELECT {', '.join(columns)} FROM {table}"
    
    if filters:
        conditions = [f"{k} = '{v}'" for k, v in filters.items()]
        query += f" WHERE {' AND '.join(conditions)}"
    
    if group_by:
        query += f" GROUP BY {group_by}"
    
    if order_by:
        query += f" ORDER BY {order_by}"
    
    if limit:
        query += f" LIMIT {limit}"
    
    return query
```

## Troubleshooting

### API Rate Limiting

```python
import time
from requests.exceptions import HTTPError

def fetch_with_retry(api_key: str, page: int, max_retries: int = 3):
    """Fetch with exponential backoff on rate limit."""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except HTTPError as e:
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
def test_db_connection(db_config: Dict) -> bool:
    """Test database connectivity."""
    try:
        conn = mysql.connector.connect(**db_config)
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        cursor.fetchone()
        conn.close()
        return True
    except Exception as e:
        print(f"Database connection failed: {e}")
        return False
```

### Memory Management for Large Datasets

```python
def process_in_chunks(records: List[Dict], chunk_size: int = 1000):
    """Process large datasets in chunks."""
    etl = ArtifactETL(db_config)
    
    for i in range(0, len(records), chunk_size):
        chunk = records[i:i + chunk_size]
        etl.run_etl(chunk)
        print(f"Processed chunk {i//chunk_size + 1}")
```

## Advanced Usage

### Custom Transformations

```python
def add_custom_fields(df: pd.DataFrame) -> pd.DataFrame:
    """Add derived fields to artifact metadata."""
    # Extract decade from century
    df['decade'] = df['century'].str.extract(r'(\d{2})\d{2}')[0] + '0s'
    
    # Categorize by age
    df['age_category'] = pd.cut(
        df['century'].str.extract(r'(\d+)')[0].astype(float),
        bins=[0, 500, 1000, 1500, 2000],
        labels=['Ancient', 'Medieval', 'Renaissance', 'Modern']
    )
    
    return df
```

### Export Analytics Results

```python
def export_results(df: pd.DataFrame, format: str = "csv"):
    """Export query results to file."""
    timestamp = pd.Timestamp.now().strftime("%Y%m%d_%H%M%S")
    
    if format == "csv":
        filename = f"analytics_{timestamp}.csv"
        df.to_csv(filename, index=False)
    elif format == "excel":
        filename = f"analytics_{timestamp}.xlsx"
        df.to_excel(filename, index=False)
    elif format == "json":
        filename = f"analytics_{timestamp}.json"
        df.to_json(filename, orient="records", indent=2)
    
    return filename
```
