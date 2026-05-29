---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data with SQL and Streamlit
triggers:
  - how do I fetch data from Harvard Art Museums API
  - build an ETL pipeline for museum artifacts
  - create analytics dashboard with Streamlit and SQL
  - visualize Harvard museum collection data
  - set up TiDB Cloud database for artifact data
  - run SQL analytics on museum API data
  - implement pagination for Harvard Arts API
  - create interactive visualizations with Plotly and museum data
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. It covers ETL pipeline construction, SQL database design, analytical query execution, and interactive visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App demonstrates:
- **API Integration**: Fetching paginated artifact data from Harvard Art Museums API
- **ETL Pipeline**: Extracting, transforming, and loading nested JSON into relational SQL tables
- **Database Design**: Structured storage with proper foreign key relationships
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Interactive Visualization**: Streamlit dashboards with Plotly charts

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_db_host"
export DB_USER="your_db_user"
export DB_PASSWORD="your_db_password"
export DB_NAME="harvard_artifacts"
```

**Required packages:**
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Database Schema

The project uses three main tables:

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
    medium VARCHAR(500),
    technique VARCHAR(500),
    primaryimageurl TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl TEXT,
    format VARCHAR(50),
    height INT,
    width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent DECIMAL(5,2),
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
        JSON response with artifact records
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    
    return response.json()

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=100)

print(f"Total records: {data['info']['totalrecords']}")
print(f"Pages: {data['info']['pages']}")
print(f"Records fetched: {len(data['records'])}")
```

### Handling Pagination

```python
def fetch_all_artifacts(api_key: str, max_records: int = 1000) -> List[Dict]:
    """
    Fetch multiple pages of artifacts with rate limiting.
    
    Args:
        api_key: Harvard API key
        max_records: Maximum number of records to fetch
    
    Returns:
        List of all artifact records
    """
    import time
    
    all_records = []
    page = 1
    size = 100
    
    while len(all_records) < max_records:
        try:
            data = fetch_artifacts(api_key, page=page, size=size)
            records = data.get('records', [])
            
            if not records:
                break
            
            all_records.extend(records)
            print(f"Fetched page {page}: {len(records)} records")
            
            page += 1
            time.sleep(0.5)  # Rate limiting
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return all_records[:max_records]
```

## ETL Pipeline

### Extract and Transform

```python
import pandas as pd

def transform_artifacts(records: List[Dict]) -> tuple:
    """
    Transform nested JSON records into relational dataframes.
    
    Returns:
        Tuple of (metadata_df, media_df, colors_df)
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for record in records:
        # Extract metadata
        metadata = {
            'id': record.get('id'),
            'title': record.get('title'),
            'culture': record.get('culture'),
            'century': record.get('century'),
            'classification': record.get('classification'),
            'department': record.get('department'),
            'dated': record.get('dated'),
            'medium': record.get('medium'),
            'technique': record.get('technique'),
            'primaryimageurl': record.get('primaryimageurl')
        }
        metadata_list.append(metadata)
        
        # Extract media (nested array)
        for media in record.get('images', []):
            media_list.append({
                'artifact_id': record.get('id'),
                'baseimageurl': media.get('baseimageurl'),
                'format': media.get('format'),
                'height': media.get('height'),
                'width': media.get('width')
            })
        
        # Extract colors (nested array)
        for color in record.get('colors', []):
            colors_list.append({
                'artifact_id': record.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load to SQL Database

```python
import mysql.connector
from typing import Tuple

def load_to_database(
    metadata_df: pd.DataFrame,
    media_df: pd.DataFrame,
    colors_df: pd.DataFrame,
    db_config: Dict
) -> None:
    """
    Load transformed dataframes into MySQL/TiDB Cloud.
    
    Args:
        metadata_df: Artifact metadata dataframe
        media_df: Media information dataframe
        colors_df: Color data dataframe
        db_config: Database connection configuration
    """
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    
    # Insert metadata (batch insert for performance)
    metadata_query = """
        INSERT IGNORE INTO artifactmetadata 
        (id, title, culture, century, classification, department, 
         dated, medium, technique, primaryimageurl)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    """
    
    metadata_values = [
        tuple(row) for row in metadata_df.values
    ]
    cursor.executemany(metadata_query, metadata_values)
    
    # Insert media
    media_query = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, format, height, width)
        VALUES (%s, %s, %s, %s, %s)
    """
    
    if not media_df.empty:
        media_values = [tuple(row) for row in media_df.values]
        cursor.executemany(media_query, media_values)
    
    # Insert colors
    colors_query = """
        INSERT INTO artifactcolors 
        (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
    """
    
    if not colors_df.empty:
        colors_values = [tuple(row) for row in colors_df.values]
        cursor.executemany(colors_query, colors_values)
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print(f"Loaded {len(metadata_df)} artifacts to database")

# Usage
db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

load_to_database(metadata_df, media_df, colors_df, db_config)
```

## SQL Analytics Queries

### Sample Analytical Queries

```python
# Query 1: Artifact distribution by culture
QUERY_BY_CULTURE = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 15
"""

# Query 2: Artifacts by century
QUERY_BY_CENTURY = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY count DESC
"""

# Query 3: Media availability analysis
QUERY_MEDIA_STATS = """
    SELECT 
        COUNT(DISTINCT a.id) as total_artifacts,
        COUNT(DISTINCT m.artifact_id) as artifacts_with_media,
        COUNT(m.media_id) as total_media_files
    FROM artifactmetadata a
    LEFT JOIN artifactmedia m ON a.id = m.artifact_id
"""

# Query 4: Top colors across artifacts
QUERY_TOP_COLORS = """
    SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
    FROM artifactcolors
    GROUP BY color
    ORDER BY usage_count DESC
    LIMIT 10
"""

# Query 5: Department breakdown
QUERY_BY_DEPARTMENT = """
    SELECT department, COUNT(*) as count
    FROM artifactmetadata
    WHERE department IS NOT NULL
    GROUP BY department
    ORDER BY count DESC
"""
```

### Execute Queries

```python
def execute_analytical_query(query: str, db_config: Dict) -> pd.DataFrame:
    """
    Execute SQL query and return results as dataframe.
    
    Args:
        query: SQL query string
        db_config: Database connection configuration
    
    Returns:
        Query results as pandas DataFrame
    """
    conn = mysql.connector.connect(**db_config)
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# Usage
results = execute_analytical_query(QUERY_BY_CULTURE, db_config)
print(results)
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
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Select Function",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "SQL Analytics":
        show_sql_analytics()
    elif page == "Visualizations":
        show_visualizations()

def show_data_collection():
    st.header("📥 Data Collection from Harvard API")
    
    api_key = st.text_input("Enter Harvard API Key", type="password")
    num_records = st.number_input("Number of records", min_value=10, max_value=1000, value=100)
    
    if st.button("Fetch Data"):
        with st.spinner("Fetching artifacts..."):
            records = fetch_all_artifacts(api_key, max_records=num_records)
            
            metadata_df, media_df, colors_df = transform_artifacts(records)
            
            st.success(f"Fetched {len(records)} artifacts!")
            st.dataframe(metadata_df.head())
            
            # Load to database
            db_config = {
                'host': os.getenv('DB_HOST'),
                'user': os.getenv('DB_USER'),
                'password': os.getenv('DB_PASSWORD'),
                'database': os.getenv('DB_NAME')
            }
            load_to_database(metadata_df, media_df, colors_df, db_config)
            st.success("Data loaded to database!")

def show_sql_analytics():
    st.header("📊 SQL Analytics")
    
    # Query selector
    queries = {
        "Artifacts by Culture": QUERY_BY_CULTURE,
        "Artifacts by Century": QUERY_BY_CENTURY,
        "Media Statistics": QUERY_MEDIA_STATS,
        "Top Colors": QUERY_TOP_COLORS,
        "Department Breakdown": QUERY_BY_DEPARTMENT
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        db_config = {
            'host': os.getenv('DB_HOST'),
            'user': os.getenv('DB_USER'),
            'password': os.getenv('DB_PASSWORD'),
            'database': os.getenv('DB_NAME')
        }
        
        results = execute_analytical_query(queries[selected_query], db_config)
        
        st.subheader("Query Results")
        st.dataframe(results)
        
        # Auto-generate visualization
        if len(results.columns) >= 2:
            fig = px.bar(
                results,
                x=results.columns[0],
                y=results.columns[1],
                title=selected_query
            )
            st.plotly_chart(fig, use_container_width=True)

def show_visualizations():
    st.header("📈 Interactive Visualizations")
    
    db_config = {
        'host': os.getenv('DB_HOST'),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    
    # Culture distribution
    culture_df = execute_analytical_query(QUERY_BY_CULTURE, db_config)
    
    fig1 = px.bar(
        culture_df,
        x='culture',
        y='artifact_count',
        title='Artifact Distribution by Culture',
        color='artifact_count',
        color_continuous_scale='viridis'
    )
    st.plotly_chart(fig1, use_container_width=True)
    
    # Color analysis
    colors_df = execute_analytical_query(QUERY_TOP_COLORS, db_config)
    
    fig2 = px.pie(
        colors_df,
        values='usage_count',
        names='color',
        title='Top 10 Colors in Collection'
    )
    st.plotly_chart(fig2, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Configuration

### Environment Variables

Create a `.env` file:

```env
HARVARD_API_KEY=your_api_key_here
DB_HOST=gateway01.your-region.prod.aws.tidbcloud.com
DB_PORT=4000
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Load Configuration

```python
from dotenv import load_dotenv
import os

load_dotenv()

def get_db_config() -> Dict:
    """Get database configuration from environment variables."""
    return {
        'host': os.getenv('DB_HOST'),
        'port': int(os.getenv('DB_PORT', 3306)),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME'),
        'ssl_ca': os.getenv('DB_SSL_CA')  # For TiDB Cloud
    }
```

## Running the Application

```bash
# Run Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Complete ETL Workflow

```python
def run_etl_pipeline(api_key: str, num_records: int = 500):
    """Complete ETL pipeline from API to database."""
    
    # Extract
    print("Extracting data from API...")
    records = fetch_all_artifacts(api_key, max_records=num_records)
    
    # Transform
    print("Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts(records)
    
    # Load
    print("Loading to database...")
    db_config = get_db_config()
    load_to_database(metadata_df, media_df, colors_df, db_config)
    
    print(f"ETL complete! Processed {len(records)} artifacts.")
    
    return metadata_df, media_df, colors_df
```

### Error Handling

```python
def safe_api_fetch(api_key: str, page: int) -> Dict:
    """Fetch with retry logic and error handling."""
    import time
    
    max_retries = 3
    retry_delay = 2
    
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page=page)
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:  # Rate limited
                print(f"Rate limited. Waiting {retry_delay}s...")
                time.sleep(retry_delay)
                retry_delay *= 2
            else:
                raise
        except requests.exceptions.RequestException as e:
            print(f"Attempt {attempt + 1} failed: {e}")
            if attempt == max_retries - 1:
                raise
            time.sleep(retry_delay)
    
    return {}
```

## Troubleshooting

### API Rate Limiting

```python
# Add delay between requests
import time

for page in range(1, 11):
    data = fetch_artifacts(api_key, page=page)
    time.sleep(1)  # 1 second delay
```

### Database Connection Issues

```python
# Test connection
def test_db_connection(db_config: Dict) -> bool:
    """Test database connectivity."""
    try:
        conn = mysql.connector.connect(**db_config)
        conn.close()
        return True
    except mysql.connector.Error as e:
        print(f"Database connection error: {e}")
        return False

# For TiDB Cloud, ensure SSL is configured
db_config['ssl_verify_cert'] = True
db_config['ssl_verify_identity'] = True
```

### Handling Missing Data

```python
def safe_transform(record: Dict) -> Dict:
    """Transform with default values for missing fields."""
    return {
        'id': record.get('id', 0),
        'title': record.get('title', 'Untitled'),
        'culture': record.get('culture', 'Unknown'),
        'century': record.get('century', 'Unknown'),
        'classification': record.get('classification', 'Unclassified'),
        'department': record.get('department', 'Unknown'),
        'dated': record.get('dated', ''),
        'medium': record.get('medium', ''),
        'technique': record.get('technique', ''),
        'primaryimageurl': record.get('primaryimageurl', '')
    }
```

This skill provides comprehensive guidance for building ETL pipelines and analytics dashboards using the Harvard Art Museums API with modern data engineering tools.
