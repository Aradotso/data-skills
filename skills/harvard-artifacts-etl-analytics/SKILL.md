---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Streamlit, SQL, and interactive visualizations
triggers:
  - how do I build an ETL pipeline for museum data
  - create a data engineering app with Harvard Art Museums API
  - set up artifact collection analytics with Streamlit
  - build SQL analytics dashboard for museum artifacts
  - extract and transform Harvard museum API data
  - create interactive visualizations for art collection data
  - implement batch data ingestion from Harvard API
  - design relational database schema for artifact metadata
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Harvard Artifacts Collection Data Engineering Analytics App is an end-to-end data pipeline that extracts artifact data from the Harvard Art Museums API, transforms it into relational tables, loads it into SQL databases (MySQL/TiDB Cloud), and provides interactive analytics dashboards using Streamlit and Plotly.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

Key capabilities:
- Paginated API data collection with rate limiting
- ETL pipeline for nested JSON to relational tables
- SQL database design with foreign key relationships
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
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="harvard_artifacts"
```

Required packages:
- `streamlit`
- `pandas`
- `requests`
- `mysql-connector-python` or `pymysql`
- `plotly`
- `python-dotenv`

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Access the dashboard at http://localhost:8501
```

## Database Schema

The application uses three main tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    accession_number VARCHAR(100),
    url TEXT
);

CREATE TABLE artifactmedia (
    media_id INT PRIMARY KEY AUTO_INCREMENT,
    artifact_id INT,
    base_image_url TEXT,
    image_width INT,
    image_height INT,
    format VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    color_id INT PRIMARY KEY AUTO_INCREMENT,
    artifact_id INT,
    color_hex VARCHAR(7),
    color_percent FLOAT,
    color_hue VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## API Integration

### Fetching Artifacts

```python
import requests
import os
from typing import List, Dict

def fetch_artifacts(api_key: str, page: int = 1, size: int = 100) -> Dict:
    """
    Fetch artifacts from Harvard Art Museums API with pagination.
    
    Args:
        api_key: Harvard API key from environment
        page: Page number (default: 1)
        size: Results per page (max: 100)
    
    Returns:
        Dictionary with records and pagination info
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        "apikey": api_key,
        "page": page,
        "size": size,
        "hasimage": 1  # Only artifacts with images
    }
    
    try:
        response = requests.get(base_url, params=params, timeout=30)
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestException as e:
        print(f"API request failed: {e}")
        return {}

# Usage
api_key = os.getenv("HARVARD_API_KEY")
data = fetch_artifacts(api_key, page=1, size=50)
artifacts = data.get("records", [])
```

### Handling Pagination

```python
def fetch_all_artifacts(api_key: str, max_pages: int = 10) -> List[Dict]:
    """Fetch multiple pages of artifacts."""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, page=page)
        
        records = data.get("records", [])
        if not records:
            break
            
        all_artifacts.extend(records)
        
        # Check if more pages exist
        info = data.get("info", {})
        if page >= info.get("pages", 0):
            break
    
    return all_artifacts
```

## ETL Pipeline

### Extract and Transform

```python
import pandas as pd

def extract_metadata(artifacts: List[Dict]) -> pd.DataFrame:
    """Extract artifact metadata into a DataFrame."""
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            "id": artifact.get("id"),
            "title": artifact.get("title", "")[:500],
            "culture": artifact.get("culture", "")[:200],
            "period": artifact.get("period", "")[:200],
            "century": artifact.get("century", "")[:100],
            "classification": artifact.get("classification", "")[:200],
            "department": artifact.get("department", "")[:200],
            "dated": artifact.get("dated", "")[:200],
            "accession_number": artifact.get("accessionyear", ""),
            "url": artifact.get("url", "")
        })
    
    return pd.DataFrame(metadata)

def extract_media(artifacts: List[Dict]) -> pd.DataFrame:
    """Extract media/image data."""
    media_data = []
    
    for artifact in artifacts:
        artifact_id = artifact.get("id")
        primary_image = artifact.get("primaryimageurl")
        
        if primary_image:
            media_data.append({
                "artifact_id": artifact_id,
                "base_image_url": primary_image,
                "image_width": artifact.get("width"),
                "image_height": artifact.get("height"),
                "format": "jpg"
            })
    
    return pd.DataFrame(media_data)

def extract_colors(artifacts: List[Dict]) -> pd.DataFrame:
    """Extract color palette data."""
    color_data = []
    
    for artifact in artifacts:
        artifact_id = artifact.get("id")
        colors = artifact.get("colors", [])
        
        for color in colors:
            color_data.append({
                "artifact_id": artifact_id,
                "color_hex": color.get("hex"),
                "color_percent": color.get("percent"),
                "color_hue": color.get("hue")
            })
    
    return pd.DataFrame(color_data)
```

### Load to Database

```python
import mysql.connector
from typing import List

def get_db_connection():
    """Create database connection."""
    return mysql.connector.connect(
        host=os.getenv("DB_HOST"),
        user=os.getenv("DB_USER"),
        password=os.getenv("DB_PASSWORD"),
        database=os.getenv("DB_NAME")
    )

def batch_insert_metadata(df: pd.DataFrame, batch_size: int = 500):
    """Batch insert artifact metadata."""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, department, dated, accession_number, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    for i in range(0, len(df), batch_size):
        batch = df.iloc[i:i + batch_size]
        values = [tuple(row) for row in batch.values]
        
        try:
            cursor.executemany(insert_query, values)
            conn.commit()
            print(f"Inserted batch {i // batch_size + 1}")
        except Exception as e:
            conn.rollback()
            print(f"Error inserting batch: {e}")
    
    cursor.close()
    conn.close()

def batch_insert_media(df: pd.DataFrame):
    """Batch insert media data."""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia 
        (artifact_id, base_image_url, image_width, image_height, format)
        VALUES (%s, %s, %s, %s, %s)
    """
    
    values = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, values)
    conn.commit()
    
    cursor.close()
    conn.close()
```

## SQL Analytics Queries

### Common Analytics Patterns

```python
def execute_query(query: str) -> pd.DataFrame:
    """Execute SQL query and return results as DataFrame."""
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# Top 10 cultures by artifact count
query_1 = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL AND culture != ''
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Artifacts by century
query_2 = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY count DESC
"""

# Most common colors
query_3 = """
    SELECT color_hue, COUNT(*) as frequency,
           AVG(color_percent) as avg_percent
    FROM artifactcolors
    WHERE color_hue IS NOT NULL
    GROUP BY color_hue
    ORDER BY frequency DESC
    LIMIT 10
"""

# Department distribution
query_4 = """
    SELECT department, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE department IS NOT NULL
    GROUP BY department
    ORDER BY artifact_count DESC
"""

# Artifacts with media
query_5 = """
    SELECT 
        a.classification,
        COUNT(DISTINCT a.id) as total_artifacts,
        COUNT(DISTINCT m.artifact_id) as with_media,
        ROUND(COUNT(DISTINCT m.artifact_id) * 100.0 / COUNT(DISTINCT a.id), 2) as media_percentage
    FROM artifactmetadata a
    LEFT JOIN artifactmedia m ON a.id = m.artifact_id
    WHERE a.classification IS NOT NULL
    GROUP BY a.classification
    HAVING total_artifacts > 10
    ORDER BY media_percentage DESC
"""
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🏛️",
        layout="wide"
    )
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar navigation
    page = st.sidebar.radio(
        "Navigation",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "SQL Analytics":
        show_sql_analytics()
    else:
        show_visualizations()

def show_data_collection():
    """ETL interface."""
    st.header("📥 Data Collection & ETL")
    
    api_key = st.text_input("Harvard API Key", type="password", 
                            value=os.getenv("HARVARD_API_KEY", ""))
    
    num_pages = st.slider("Number of pages to fetch", 1, 20, 5)
    
    if st.button("Start ETL Pipeline"):
        with st.spinner("Fetching artifacts..."):
            artifacts = fetch_all_artifacts(api_key, max_pages=num_pages)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_metadata = extract_metadata(artifacts)
            df_media = extract_media(artifacts)
            df_colors = extract_colors(artifacts)
            st.success("Data transformed")
        
        with st.spinner("Loading to database..."):
            batch_insert_metadata(df_metadata)
            batch_insert_media(df_media)
            st.success("Data loaded successfully!")

def show_sql_analytics():
    """SQL query interface."""
    st.header("📊 SQL Analytics")
    
    queries = {
        "Top Cultures": query_1,
        "Artifacts by Century": query_2,
        "Color Analysis": query_3,
        "Department Distribution": query_4,
        "Media Coverage": query_5
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        df = execute_query(queries[selected_query])
        st.dataframe(df, use_container_width=True)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                        title=f"{selected_query} Analysis")
            st.plotly_chart(fig, use_container_width=True)

def show_visualizations():
    """Pre-built visualizations."""
    st.header("📈 Interactive Visualizations")
    
    # Culture distribution
    df = execute_query(query_1)
    fig = px.bar(df, x='culture', y='artifact_count',
                title="Top 10 Cultures by Artifact Count")
    st.plotly_chart(fig, use_container_width=True)
    
    # Color heatmap
    df_colors = execute_query(query_3)
    fig = px.pie(df_colors, values='frequency', names='color_hue',
                title="Color Distribution Across Artifacts")
    st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Configuration

### Environment Variables

Create a `.env` file:

```env
HARVARD_API_KEY=your_api_key_here
DB_HOST=localhost
DB_USER=root
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
DB_PORT=3306
```

### Loading Configuration

```python
from dotenv import load_dotenv

load_dotenv()

CONFIG = {
    "api_key": os.getenv("HARVARD_API_KEY"),
    "db_config": {
        "host": os.getenv("DB_HOST"),
        "user": os.getenv("DB_USER"),
        "password": os.getenv("DB_PASSWORD"),
        "database": os.getenv("DB_NAME"),
        "port": int(os.getenv("DB_PORT", 3306))
    }
}
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(api_key: str, page: int, max_retries: int = 3):
    """Fetch with exponential backoff on rate limit."""
    for attempt in range(max_retries):
        try:
            response = fetch_artifacts(api_key, page)
            return response
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:  # Rate limited
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    return {}
```

### Database Connection Issues

```python
def test_db_connection():
    """Test database connectivity."""
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        cursor.close()
        conn.close()
        print("✅ Database connection successful")
        return True
    except Exception as e:
        print(f"❌ Database connection failed: {e}")
        return False
```

### Handling Missing Data

```python
def clean_artifact_data(artifact: Dict) -> Dict:
    """Clean and validate artifact data."""
    return {
        "id": artifact.get("id", 0),
        "title": (artifact.get("title") or "Untitled")[:500],
        "culture": (artifact.get("culture") or "Unknown")[:200],
        "century": (artifact.get("century") or "Unknown")[:100],
        # Use .get() with defaults for all fields
    }
```

### Memory Management for Large Datasets

```python
def process_in_chunks(artifacts: List[Dict], chunk_size: int = 1000):
    """Process large artifact lists in chunks."""
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i + chunk_size]
        df = extract_metadata(chunk)
        batch_insert_metadata(df)
        print(f"Processed {i + len(chunk)}/{len(artifacts)}")
```

## Best Practices

1. **Always use environment variables** for API keys and credentials
2. **Implement batch inserts** for performance (500-1000 records per batch)
3. **Use ON DUPLICATE KEY UPDATE** to handle re-runs without errors
4. **Add retry logic** for API calls with rate limiting
5. **Validate data types** before inserting into SQL
6. **Create indexes** on foreign keys and frequently queried columns
7. **Use connection pooling** for production deployments
