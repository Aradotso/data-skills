---
name: harvard-art-museum-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Streamlit, SQL, and Python
triggers:
  - how do I build a data pipeline with Harvard Art Museums API
  - set up ETL workflow for museum artifact data
  - create analytics dashboard for Harvard art collection
  - extract and analyze Harvard museum data
  - build Streamlit app with Harvard Art Museums API
  - query and visualize museum artifact metadata
  - implement SQL analytics for art collection data
  - process Harvard Art Museums API with Python
---

# Harvard Art Museum ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Art Museum ETL Analytics application:
- Fetches artifact data from the Harvard Art Museums API with pagination
- Transforms nested JSON into relational database tables
- Loads structured data into MySQL/TiDB Cloud databases
- Executes analytical SQL queries on artifact metadata, media, and colors
- Visualizes insights through interactive Plotly charts in Streamlit

Architecture: **API → ETL → SQL → Analytics → Visualization**

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Requirements:**
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file or configure environment variables:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=4000
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Setup

The application uses three main tables:

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    dated VARCHAR(255),
    url VARCHAR(500)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Components

### 1. API Data Extraction

```python
import requests
import os

def fetch_artifacts_from_api(api_key, num_records=100):
    """
    Fetch artifact data from Harvard Art Museums API with pagination.
    
    Args:
        api_key: Harvard API key from environment
        num_records: Total number of records to fetch
    
    Returns:
        List of artifact dictionaries
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    page = 1
    size = 100  # API limit per page
    
    while len(all_artifacts) < num_records:
        params = {
            'apikey': api_key,
            'page': page,
            'size': min(size, num_records - len(all_artifacts))
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            
            if not records:
                break
                
            all_artifacts.extend(records)
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return all_artifacts[:num_records]
```

### 2. ETL Pipeline

```python
import pandas as pd

def transform_artifacts_to_dataframes(artifacts):
    """
    Transform nested JSON artifacts into relational dataframes.
    
    Returns:
        Tuple of (metadata_df, media_df, colors_df)
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'period': artifact.get('period', '')[:255],
            'century': artifact.get('century', '')[:255],
            'classification': artifact.get('classification', '')[:255],
            'department': artifact.get('department', '')[:255],
            'technique': artifact.get('technique', '')[:500],
            'medium': artifact.get('medium', '')[:500],
            'dimensions': artifact.get('dimensions', '')[:500],
            'dated': artifact.get('dated', '')[:255],
            'url': artifact.get('url', '')[:500]
        }
        metadata_list.append(metadata)
        
        # Extract media information
        primary_image = artifact.get('primaryimageurl')
        if primary_image:
            media = {
                'artifact_id': artifact.get('id'),
                'baseimageurl': primary_image[:500],
                'iiifbaseuri': artifact.get('images', [{}])[0].get('iiifbaseuri', '')[:500] if artifact.get('images') else ''
            }
            media_list.append(media)
        
        # Extract color data
        colors = artifact.get('colors', [])
        for color in colors:
            color_record = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color', '')[:50],
                'spectrum': color.get('spectrum', '')[:50],
                'percent': color.get('percent', 0.0)
            }
            colors_list.append(color_record)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def load_data_to_database(metadata_df, media_df, colors_df, db_config):
    """
    Batch insert dataframes into MySQL database.
    
    Args:
        metadata_df: Artifact metadata DataFrame
        media_df: Media information DataFrame
        colors_df: Color data DataFrame
        db_config: Database connection dictionary
    """
    try:
        connection = mysql.connector.connect(**db_config)
        cursor = connection.cursor()
        
        # Insert metadata
        metadata_sql = """
            INSERT INTO artifactmetadata 
            (id, title, culture, period, century, classification, 
             department, technique, medium, dimensions, dated, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        metadata_values = metadata_df.values.tolist()
        cursor.executemany(metadata_sql, metadata_values)
        
        # Insert media
        media_sql = """
            INSERT INTO artifactmedia (artifact_id, baseimageurl, iiifbaseuri)
            VALUES (%s, %s, %s)
        """
        media_values = media_df.values.tolist()
        cursor.executemany(media_sql, media_values)
        
        # Insert colors
        colors_sql = """
            INSERT INTO artifactcolors (artifact_id, color, spectrum, percent)
            VALUES (%s, %s, %s, %s)
        """
        colors_values = colors_df.values.tolist()
        cursor.executemany(colors_sql, colors_values)
        
        connection.commit()
        print(f"Inserted {len(metadata_df)} metadata, {len(media_df)} media, {len(colors_df)} color records")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

### 4. Analytics Queries

```python
# Sample analytical SQL queries

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
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Top Colors by Frequency": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "Media Availability": """
        SELECT 
            COUNT(DISTINCT m.artifact_id) as artifacts_with_media,
            (SELECT COUNT(*) FROM artifactmetadata) as total_artifacts,
            ROUND(COUNT(DISTINCT m.artifact_id) * 100.0 / 
                  (SELECT COUNT(*) FROM artifactmetadata), 2) as percentage
        FROM artifactmedia m
    """,
    
    "Artifacts with Multiple Colors": """
        SELECT a.title, a.culture, COUNT(c.color) as color_count
        FROM artifactmetadata a
        JOIN artifactcolors c ON a.id = c.artifact_id
        GROUP BY a.id, a.title, a.culture
        HAVING color_count > 5
        ORDER BY color_count DESC
        LIMIT 20
    """
}

def execute_analytics_query(query, db_config):
    """Execute analytical query and return results as DataFrame."""
    try:
        connection = mysql.connector.connect(**db_config)
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        print(f"Query error: {e}")
        return pd.DataFrame()
    finally:
        if connection.is_connected():
            connection.close()
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password", 
                                     value=os.getenv("HARVARD_API_KEY", ""))
    
    # Database config from environment
    db_config = {
        'host': os.getenv('DB_HOST'),
        'port': int(os.getenv('DB_PORT', 3306)),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    
    # ETL Section
    st.header("📥 Data Collection & ETL")
    num_records = st.number_input("Number of records to fetch", 
                                   min_value=10, max_value=1000, value=100)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            artifacts = fetch_artifacts_from_api(api_key, num_records)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts_to_dataframes(artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            load_data_to_database(metadata_df, media_df, colors_df, db_config)
            st.success("Data loaded to database")
    
    # Analytics Section
    st.header("📊 SQL Analytics")
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        query = ANALYTICS_QUERIES[query_name]
        st.code(query, language="sql")
        
        results_df = execute_analytics_query(query, db_config)
        
        if not results_df.empty:
            st.dataframe(results_df)
            
            # Auto-generate visualization
            if len(results_df.columns) >= 2:
                fig = px.bar(results_df, 
                            x=results_df.columns[0], 
                            y=results_df.columns[1],
                            title=query_name)
                st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Pattern 1: Incremental Data Loading

```python
def incremental_load(api_key, db_config, batch_size=100):
    """Load data incrementally to avoid overwhelming the database."""
    connection = mysql.connector.connect(**db_config)
    cursor = connection.cursor()
    
    # Get last loaded artifact ID
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    last_id = cursor.fetchone()[0] or 0
    
    # Fetch only new artifacts
    artifacts = fetch_artifacts_from_api(api_key, num_records=batch_size)
    new_artifacts = [a for a in artifacts if a.get('id', 0) > last_id]
    
    if new_artifacts:
        metadata_df, media_df, colors_df = transform_artifacts_to_dataframes(new_artifacts)
        load_data_to_database(metadata_df, media_df, colors_df, db_config)
    
    connection.close()
    return len(new_artifacts)
```

### Pattern 2: Error Handling with API Rate Limits

```python
import time

def fetch_with_retry(api_key, max_retries=3):
    """Fetch data with exponential backoff on rate limit errors."""
    for attempt in range(max_retries):
        try:
            artifacts = fetch_artifacts_from_api(api_key, num_records=100)
            return artifacts
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:  # Rate limit
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    return []
```

### Pattern 3: Custom Analytics Query

```python
def custom_analytics(db_config, culture_filter=None):
    """Run custom analytics with dynamic filtering."""
    base_query = """
        SELECT a.culture, a.century, COUNT(*) as count,
               AVG(LENGTH(a.title)) as avg_title_length
        FROM artifactmetadata a
        WHERE 1=1
    """
    
    if culture_filter:
        base_query += f" AND a.culture = '{culture_filter}'"
    
    base_query += " GROUP BY a.culture, a.century ORDER BY count DESC"
    
    return execute_analytics_query(base_query, db_config)
```

## Troubleshooting

### API Connection Issues

```python
# Test API connectivity
def test_api_connection(api_key):
    try:
        response = requests.get(
            "https://api.harvardartmuseums.org/object",
            params={'apikey': api_key, 'size': 1}
        )
        if response.status_code == 200:
            print("✓ API connection successful")
            return True
        elif response.status_code == 401:
            print("✗ Invalid API key")
        else:
            print(f"✗ API error: {response.status_code}")
        return False
    except Exception as e:
        print(f"✗ Connection error: {e}")
        return False
```

### Database Connection Issues

```python
def test_database_connection(db_config):
    try:
        connection = mysql.connector.connect(**db_config)
        if connection.is_connected():
            print("✓ Database connection successful")
            connection.close()
            return True
    except Error as e:
        print(f"✗ Database error: {e}")
        return False
```

### Data Quality Validation

```python
def validate_data_quality(metadata_df):
    """Check for data quality issues before loading."""
    issues = []
    
    # Check for duplicates
    duplicates = metadata_df['id'].duplicated().sum()
    if duplicates > 0:
        issues.append(f"Found {duplicates} duplicate IDs")
    
    # Check for missing critical fields
    missing_titles = metadata_df['title'].isna().sum()
    if missing_titles > 0:
        issues.append(f"{missing_titles} artifacts missing titles")
    
    # Check for data type issues
    if not pd.api.types.is_integer_dtype(metadata_df['id']):
        issues.append("ID column is not integer type")
    
    if issues:
        print("Data quality issues:", issues)
    else:
        print("✓ Data quality validation passed")
    
    return len(issues) == 0
```

## Running the Application

```bash
# Set environment variables
export HARVARD_API_KEY=your_api_key
export DB_HOST=your_host
export DB_USER=your_user
export DB_PASSWORD=your_password
export DB_NAME=harvard_artifacts

# Run Streamlit app
streamlit run app.py
```

The dashboard will be available at `http://localhost:8501`
