---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I use the Harvard Art Museums API for data engineering
  - build an ETL pipeline with Harvard artifacts data
  - create analytics dashboard for museum collection data
  - extract and transform Harvard Art Museums API data
  - set up SQL database for art museum artifacts
  - visualize Harvard museum data with Streamlit
  - implement data pipeline for cultural heritage collections
  - analyze art museum metadata with SQL queries
---

# Harvard Art Museums ETL & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

An end-to-end data engineering and analytics application that demonstrates real-world ETL pipelines using the Harvard Art Museums API. This project shows how to extract artifact data, transform nested JSON into relational tables, load into SQL databases, and build interactive analytics dashboards with Streamlit.

## What This Project Does

This application provides a complete data pipeline for museum artifact data:

- **Extract**: Fetch artifact metadata from Harvard Art Museums API with pagination and rate limiting
- **Transform**: Convert nested JSON responses into normalized relational data structures
- **Load**: Insert data into MySQL/TiDB Cloud with proper schema design
- **Analyze**: Run 20+ predefined SQL analytical queries
- **Visualize**: Display results in interactive Plotly charts via Streamlit

The architecture follows: `API → ETL → SQL → Analytics → Visualization`

## Installation

### Prerequisites

```bash
# Python 3.8+
python --version

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Project Setup

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Create environment file for credentials
touch .env
```

### Configuration

Create a `.env` file with your credentials:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# MySQL/TiDB Cloud Database
DB_HOST=your_host
DB_PORT=4000
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

Get your Harvard Art Museums API key at: https://www.harvardartmuseums.org/collections/api

## Database Schema

The project uses three main tables with proper relationships:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    department VARCHAR(200),
    classification VARCHAR(200),
    technique VARCHAR(300),
    period VARCHAR(200),
    accession_number VARCHAR(100)
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(1000),
    caption TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    hex_code VARCHAR(10),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key API Patterns

### Extracting Data from Harvard Art Museums API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    api_key = os.getenv('HARVARD_API_KEY')
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
data = fetch_artifacts(page=1, size=50)
artifacts = data['records']
total_records = data['info']['totalrecords']
```

### ETL Pipeline Implementation

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

class HarvardArtifactsETL:
    def __init__(self):
        self.db_config = {
            'host': os.getenv('DB_HOST'),
            'port': int(os.getenv('DB_PORT', 3306)),
            'user': os.getenv('DB_USER'),
            'password': os.getenv('DB_PASSWORD'),
            'database': os.getenv('DB_NAME')
        }
    
    def extract(self, num_pages=5):
        """Extract data from API"""
        all_artifacts = []
        
        for page in range(1, num_pages + 1):
            data = fetch_artifacts(page=page, size=100)
            all_artifacts.extend(data['records'])
        
        return all_artifacts
    
    def transform(self, artifacts: List[Dict]):
        """Transform nested JSON into relational data"""
        metadata_list = []
        media_list = []
        colors_list = []
        
        for artifact in artifacts:
            # Transform metadata
            metadata = {
                'id': artifact.get('id'),
                'title': artifact.get('title', 'Unknown')[:500],
                'culture': artifact.get('culture', 'Unknown')[:200],
                'century': artifact.get('century', 'Unknown')[:100],
                'dated': artifact.get('dated', 'Unknown')[:200],
                'department': artifact.get('department', 'Unknown')[:200],
                'classification': artifact.get('classification', 'Unknown')[:200],
                'technique': artifact.get('technique', 'Unknown')[:300],
                'period': artifact.get('period', 'Unknown')[:200],
                'accession_number': artifact.get('accessionyear', 'Unknown')[:100]
            }
            metadata_list.append(metadata)
            
            # Transform media/images
            if artifact.get('images'):
                for img in artifact['images']:
                    media = {
                        'artifact_id': artifact['id'],
                        'image_url': img.get('baseimageurl', ''),
                        'caption': img.get('caption', '')
                    }
                    media_list.append(media)
            
            # Transform colors
            if artifact.get('colors'):
                for color in artifact['colors']:
                    color_data = {
                        'artifact_id': artifact['id'],
                        'color': color.get('color', ''),
                        'hex_code': color.get('hex', ''),
                        'percentage': color.get('percent', 0.0)
                    }
                    colors_list.append(color_data)
        
        return (
            pd.DataFrame(metadata_list),
            pd.DataFrame(media_list),
            pd.DataFrame(colors_list)
        )
    
    def load(self, metadata_df, media_df, colors_df):
        """Load data into SQL database"""
        conn = mysql.connector.connect(**self.db_config)
        cursor = conn.cursor()
        
        # Insert metadata
        for _, row in metadata_df.iterrows():
            cursor.execute("""
                INSERT IGNORE INTO artifactmetadata 
                (id, title, culture, century, dated, department, 
                 classification, technique, period, accession_number)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            """, tuple(row))
        
        # Insert media
        for _, row in media_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmedia (artifact_id, image_url, caption)
                VALUES (%s, %s, %s)
            """, tuple(row))
        
        # Insert colors
        for _, row in colors_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactcolors (artifact_id, color, hex_code, percentage)
                VALUES (%s, %s, %s, %s)
            """, tuple(row))
        
        conn.commit()
        cursor.close()
        conn.close()
        
        return len(metadata_df)

# Run ETL pipeline
etl = HarvardArtifactsETL()
artifacts = etl.extract(num_pages=3)
metadata_df, media_df, colors_df = etl.transform(artifacts)
count = etl.load(metadata_df, media_df, colors_df)
print(f"Loaded {count} artifacts into database")
```

## Analytical SQL Queries

### Common Analytics Patterns

```python
def run_sql_query(query: str):
    """Execute SQL query and return results as DataFrame"""
    conn = mysql.connector.connect(**db_config)
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# Top 10 cultures by artifact count
query_cultures = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture != 'Unknown'
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10
"""

# Artifacts by century distribution
query_centuries = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century != 'Unknown'
GROUP BY century
ORDER BY count DESC
"""

# Most common colors across artifacts
query_colors = """
SELECT c.color, COUNT(DISTINCT c.artifact_id) as artifact_count,
       AVG(c.percentage) as avg_percentage
FROM artifactcolors c
GROUP BY c.color
ORDER BY artifact_count DESC
LIMIT 15
"""

# Media availability analysis
query_media = """
SELECT 
    COUNT(DISTINCT m.artifact_id) as artifacts_with_media,
    COUNT(*) as total_media_items,
    AVG(media_per_artifact) as avg_media_per_artifact
FROM artifactmedia m
JOIN (
    SELECT artifact_id, COUNT(*) as media_per_artifact
    FROM artifactmedia
    GROUP BY artifact_id
) subq ON m.artifact_id = subq.artifact_id
"""

# Execute queries
cultures_df = run_sql_query(query_cultures)
centuries_df = run_sql_query(query_centuries)
colors_df = run_sql_query(query_colors)
```

## Streamlit Dashboard

### Building the Analytics Interface

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
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Select Analysis",
        ["Overview", "Culture Analysis", "Time Period", "Color Analysis", "Media Stats"]
    )
    
    if page == "Culture Analysis":
        st.header("Artifact Distribution by Culture")
        
        query = """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture != 'Unknown'
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 20
        """
        
        df = run_sql_query(query)
        
        # Display table
        st.dataframe(df, use_container_width=True)
        
        # Visualize with Plotly
        fig = px.bar(
            df,
            x='culture',
            y='count',
            title='Top 20 Cultures in Collection',
            labels={'count': 'Number of Artifacts', 'culture': 'Culture'},
            color='count',
            color_continuous_scale='Viridis'
        )
        fig.update_layout(xaxis_tickangle=-45)
        st.plotly_chart(fig, use_container_width=True)
    
    elif page == "Color Analysis":
        st.header("Color Distribution Analysis")
        
        query = """
        SELECT color, COUNT(DISTINCT artifact_id) as artifacts,
               AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY artifacts DESC
        LIMIT 15
        """
        
        df = run_sql_query(query)
        
        col1, col2 = st.columns(2)
        
        with col1:
            fig = px.bar(
                df,
                x='color',
                y='artifacts',
                title='Most Common Colors',
                color='color'
            )
            st.plotly_chart(fig)
        
        with col2:
            fig = px.pie(
                df,
                values='artifacts',
                names='color',
                title='Color Distribution'
            )
            st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

### Running the Streamlit App

```bash
# Start the dashboard
streamlit run app.py

# The app will open at http://localhost:8501
```

## Common Workflows

### Complete Data Collection Pipeline

```python
# 1. Initialize ETL
etl = HarvardArtifactsETL()

# 2. Extract artifacts (control pagination)
artifacts = etl.extract(num_pages=10)  # 1000 artifacts

# 3. Transform data
metadata_df, media_df, colors_df = etl.transform(artifacts)

# 4. Validate data
print(f"Metadata records: {len(metadata_df)}")
print(f"Media records: {len(media_df)}")
print(f"Color records: {len(colors_df)}")

# 5. Load into database
records_loaded = etl.load(metadata_df, media_df, colors_df)
print(f"Successfully loaded {records_loaded} artifacts")
```

### Incremental Data Updates

```python
def get_latest_artifact_id():
    """Get the most recent artifact ID in database"""
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    max_id = cursor.fetchone()[0]
    cursor.close()
    conn.close()
    return max_id or 0

def incremental_load():
    """Load only new artifacts"""
    latest_id = get_latest_artifact_id()
    
    # Fetch only artifacts with ID > latest_id
    # (Implementation depends on API filtering capabilities)
    etl = HarvardArtifactsETL()
    new_artifacts = etl.extract(num_pages=2)
    
    # Filter new artifacts
    new_artifacts = [a for a in new_artifacts if a['id'] > latest_id]
    
    if new_artifacts:
        metadata_df, media_df, colors_df = etl.transform(new_artifacts)
        etl.load(metadata_df, media_df, colors_df)
        print(f"Added {len(new_artifacts)} new artifacts")
    else:
        print("No new artifacts to load")
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(page, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page=page)
        except Exception as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt
                print(f"Retry {attempt + 1} after {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise e
```

### Database Connection Issues

```python
def test_database_connection():
    """Verify database connectivity"""
    try:
        conn = mysql.connector.connect(**db_config)
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        cursor.close()
        conn.close()
        print("✓ Database connection successful")
        return True
    except Exception as e:
        print(f"✗ Database connection failed: {e}")
        return False
```

### Data Quality Checks

```python
def validate_data_quality(df):
    """Check data quality before loading"""
    issues = []
    
    # Check for nulls in required fields
    if df['id'].isnull().any():
        issues.append("NULL values in ID column")
    
    # Check for duplicates
    if df['id'].duplicated().any():
        issues.append(f"{df['id'].duplicated().sum()} duplicate IDs")
    
    # Check data types
    if not pd.api.types.is_integer_dtype(df['id']):
        issues.append("ID column must be integer")
    
    if issues:
        print("⚠️ Data quality issues:")
        for issue in issues:
            print(f"  - {issue}")
        return False
    
    print("✓ Data quality checks passed")
    return True
```

### Memory Management for Large Datasets

```python
def batch_load_artifacts(total_pages, batch_size=5):
    """Load artifacts in batches to manage memory"""
    etl = HarvardArtifactsETL()
    total_loaded = 0
    
    for start_page in range(1, total_pages + 1, batch_size):
        end_page = min(start_page + batch_size - 1, total_pages)
        print(f"Processing batch: pages {start_page}-{end_page}")
        
        # Extract and transform batch
        artifacts = []
        for page in range(start_page, end_page + 1):
            data = fetch_artifacts(page=page)
            artifacts.extend(data['records'])
        
        metadata_df, media_df, colors_df = etl.transform(artifacts)
        count = etl.load(metadata_df, media_df, colors_df)
        total_loaded += count
        
        # Clear memory
        del artifacts, metadata_df, media_df, colors_df
        
    print(f"Total artifacts loaded: {total_loaded}")
```

## Best Practices

1. **Always use environment variables** for sensitive credentials
2. **Implement retry logic** for API calls to handle transient failures
3. **Validate data** before loading into database
4. **Use batch processing** for large datasets to manage memory
5. **Create indexes** on frequently queried columns (culture, century, artifact_id)
6. **Log ETL operations** for audit trails and debugging
7. **Handle API pagination** properly to avoid missing data
8. **Normalize data types** during transformation to prevent SQL errors
