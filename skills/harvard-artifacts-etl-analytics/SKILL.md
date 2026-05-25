---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I fetch data from Harvard Art Museums API
  - build an ETL pipeline for museum artifact data
  - create analytics dashboard with Streamlit for artifact collections
  - how to structure Harvard museum data in SQL database
  - visualize art collection data with Plotly
  - set up data engineering pipeline for museum artifacts
  - query and analyze Harvard Art Museums collection data
  - build end-to-end data pipeline with API and database
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project demonstrates an end-to-end data engineering pipeline that:
- Extracts artifact data from the Harvard Art Museums API
- Transforms nested JSON into relational database structures
- Loads data into MySQL/TiDB Cloud
- Provides SQL-based analytics queries
- Visualizes insights using Streamlit and Plotly

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

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
export DB_NAME="your_database_name"
```

## Core Dependencies

```python
import streamlit as st
import pandas as pd
import requests
import mysql.connector
import plotly.express as px
import plotly.graph_objects as go
from typing import List, Dict, Optional
import time
```

## Database Schema

The ETL pipeline creates three main tables:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    dated VARCHAR(255),
    url TEXT,
    accession_number VARCHAR(100),
    PRIMARY KEY (id)
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url TEXT,
    thumbnail_url TEXT,
    has_image BOOLEAN,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(7),
    color_percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## API Integration

### Fetching Artifact Data

```python
import requests
import os
from typing import Dict, List

def fetch_artifacts(api_key: str, page: int = 1, size: int = 100) -> Dict:
    """
    Fetch artifacts from Harvard Art Museums API with pagination.
    
    Args:
        api_key: Harvard API key from environment
        page: Page number for pagination
        size: Number of records per page (max 100)
    
    Returns:
        Dictionary containing artifact data and metadata
    """
    base_url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only fetch artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    return response.json()

def collect_artifacts_batch(api_key: str, num_pages: int = 10) -> List[Dict]:
    """
    Collect multiple pages of artifacts with rate limiting.
    
    Args:
        api_key: Harvard API key
        num_pages: Number of pages to fetch
    
    Returns:
        List of artifact records
    """
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        try:
            data = fetch_artifacts(api_key, page=page)
            all_artifacts.extend(data.get('records', []))
            
            # Rate limiting - respect API limits
            time.sleep(0.5)
            
            print(f"Fetched page {page}/{num_pages} - {len(data.get('records', []))} records")
        except Exception as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return all_artifacts
```

### Usage Example

```python
# In your application or script
api_key = os.getenv('HARVARD_API_KEY')
artifacts = collect_artifacts_batch(api_key, num_pages=5)
print(f"Total artifacts collected: {len(artifacts)}")
```

## ETL Pipeline

### Extract and Transform

```python
import pandas as pd
from typing import Tuple

def transform_artifacts(artifacts: List[Dict]) -> Tuple[pd.DataFrame, pd.DataFrame, pd.DataFrame]:
    """
    Transform raw artifact JSON into three relational DataFrames.
    
    Args:
        artifacts: List of artifact dictionaries from API
    
    Returns:
        Tuple of (metadata_df, media_df, colors_df)
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url'),
            'accession_number': artifact.get('accessionNumber')
        })
        
        # Extract media information
        primary_image = artifact.get('primaryimageurl')
        media_records.append({
            'artifact_id': artifact.get('id'),
            'image_url': primary_image,
            'thumbnail_url': artifact.get('primaryImageThumbnailUrl'),
            'has_image': 1 if primary_image else 0
        })
        
        # Extract color data
        colors = artifact.get('colors', [])
        for color in colors:
            color_records.append({
                'artifact_id': artifact.get('id'),
                'color_hex': color.get('hex'),
                'color_percent': color.get('percent')
            })
    
    metadata_df = pd.DataFrame(metadata_records)
    media_df = pd.DataFrame(media_records)
    colors_df = pd.DataFrame(color_records)
    
    return metadata_df, media_df, colors_df
```

### Load to Database

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection(host: str, user: str, password: str, database: str):
    """Create MySQL database connection."""
    try:
        connection = mysql.connector.connect(
            host=host,
            user=user,
            password=password,
            database=database
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_to_database(metadata_df: pd.DataFrame, 
                     media_df: pd.DataFrame, 
                     colors_df: pd.DataFrame,
                     connection):
    """
    Load DataFrames into SQL database using batch inserts.
    
    Args:
        metadata_df: Artifact metadata DataFrame
        media_df: Artifact media DataFrame
        colors_df: Artifact colors DataFrame
        connection: MySQL connection object
    """
    cursor = connection.cursor()
    
    try:
        # Insert metadata
        metadata_query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, department, 
             technique, medium, dimensions, dated, url, accession_number)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        metadata_values = metadata_df.values.tolist()
        cursor.executemany(metadata_query, metadata_values)
        
        # Insert media
        media_query = """
            INSERT INTO artifactmedia 
            (artifact_id, image_url, thumbnail_url, has_image)
            VALUES (%s, %s, %s, %s)
        """
        media_values = media_df.values.tolist()
        cursor.executemany(media_query, media_values)
        
        # Insert colors
        if not colors_df.empty:
            colors_query = """
                INSERT INTO artifactcolors 
                (artifact_id, color_hex, color_percent)
                VALUES (%s, %s, %s)
            """
            colors_values = colors_df.values.tolist()
            cursor.executemany(colors_query, colors_values)
        
        connection.commit()
        print(f"Successfully loaded {len(metadata_df)} artifacts to database")
        
    except Error as e:
        connection.rollback()
        print(f"Error loading data: {e}")
    finally:
        cursor.close()
```

## Streamlit Analytics Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    # Database connection
    connection = create_database_connection(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    # ETL Section
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            api_key = os.getenv('HARVARD_API_KEY')
            artifacts = collect_artifacts_batch(api_key, num_pages=5)
            
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            load_to_database(metadata_df, media_df, colors_df, connection)
            
            st.success(f"ETL Complete! Loaded {len(metadata_df)} artifacts")
    
    # Analytics Section
    st.header("📊 SQL Analytics")
    
    query_options = {
        "Artifacts by Culture": """
            SELECT culture, COUNT(*) as count 
            FROM artifactmetadata 
            WHERE culture IS NOT NULL 
            GROUP BY culture 
            ORDER BY count DESC 
            LIMIT 15
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
        "Most Common Colors": """
            SELECT color_hex, COUNT(*) as frequency, 
                   AVG(color_percent) as avg_percent
            FROM artifactcolors 
            GROUP BY color_hex 
            ORDER BY frequency DESC 
            LIMIT 20
        """
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
    
    if st.button("Run Query"):
        query = query_options[selected_query]
        df = pd.read_sql(query, connection)
        
        st.subheader("Results")
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Analytics Queries

### Artifact Distribution Analysis

```python
def get_classification_distribution(connection) -> pd.DataFrame:
    """Get artifact counts by classification."""
    query = """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 20
    """
    return pd.read_sql(query, connection)

def get_artifacts_with_images(connection) -> pd.DataFrame:
    """Analyze image availability."""
    query = """
        SELECT 
            CASE WHEN has_image = 1 THEN 'With Image' ELSE 'No Image' END as status,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY has_image
    """
    return pd.read_sql(query, connection)

def get_color_palette_analysis(connection, limit: int = 10) -> pd.DataFrame:
    """Get most common color palettes across artifacts."""
    query = f"""
        SELECT 
            color_hex,
            COUNT(DISTINCT artifact_id) as artifact_count,
            AVG(color_percent) as avg_percentage,
            MAX(color_percent) as max_percentage
        FROM artifactcolors
        GROUP BY color_hex
        ORDER BY artifact_count DESC
        LIMIT {limit}
    """
    return pd.read_sql(query, connection)
```

## Visualization Patterns

### Creating Interactive Charts

```python
def create_culture_timeline(connection):
    """Create timeline visualization of artifacts by culture and century."""
    query = """
        SELECT culture, century, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND century IS NOT NULL
        GROUP BY culture, century
        HAVING count > 5
        ORDER BY century
    """
    df = pd.read_sql(query, connection)
    
    fig = px.scatter(df, x='century', y='culture', size='count',
                     title='Artifact Distribution: Culture vs Century',
                     hover_data=['count'])
    return fig

def create_color_visualization(colors_df: pd.DataFrame):
    """Visualize color distribution with actual color swatches."""
    fig = go.Figure(data=[go.Bar(
        x=colors_df['color_hex'],
        y=colors_df['artifact_count'],
        marker_color=colors_df['color_hex'],
        text=colors_df['artifact_count'],
        textposition='auto'
    )])
    
    fig.update_layout(
        title='Most Common Colors in Collection',
        xaxis_title='Color',
        yaxis_title='Number of Artifacts'
    )
    return fig
```

## Configuration

### Environment Variables

Create a `.env` file or set environment variables:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
DB_PORT=3306
```

### Streamlit Configuration

Create `.streamlit/config.toml`:

```toml
[theme]
primaryColor = "#A51C30"
backgroundColor = "#FFFFFF"
secondaryBackgroundColor = "#F0F2F6"
textColor = "#262730"

[server]
maxUploadSize = 200
```

## Troubleshooting

### API Rate Limiting

```python
import time
from requests.exceptions import HTTPError

def fetch_with_retry(url: str, params: dict, max_retries: int = 3):
    """Fetch with exponential backoff for rate limiting."""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params)
            response.raise_for_status()
            return response.json()
        except HTTPError as e:
            if e.response.status_code == 429:  # Too Many Requests
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
def test_database_connection(host: str, user: str, password: str, database: str):
    """Test database connectivity."""
    try:
        connection = mysql.connector.connect(
            host=host,
            user=user,
            password=password,
            database=database,
            connect_timeout=10
        )
        if connection.is_connected():
            print("✓ Database connection successful")
            connection.close()
            return True
    except Error as e:
        print(f"✗ Database connection failed: {e}")
        return False
```

### Handling Missing Data

```python
def clean_artifact_data(df: pd.DataFrame) -> pd.DataFrame:
    """Clean and validate artifact data."""
    # Replace None/NaN with appropriate defaults
    df['title'] = df['title'].fillna('Untitled')
    df['culture'] = df['culture'].fillna('Unknown')
    df['century'] = df['century'].fillna('Unknown')
    
    # Remove duplicates based on artifact ID
    df = df.drop_duplicates(subset=['id'], keep='first')
    
    # Validate data types
    df['id'] = df['id'].astype(int)
    
    return df
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# Run ETL pipeline only
python etl_pipeline.py

# Run with custom configuration
streamlit run app.py --server.port 8080
```
