---
name: harvard-artifacts-data-engineering-pipeline
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - "build an ETL pipeline for museum artifact data"
  - "create analytics dashboard for Harvard Art Museums API"
  - "set up data engineering workflow for art collection data"
  - "analyze Harvard artifacts using SQL and visualization"
  - "extract and transform museum API data into relational database"
  - "build Streamlit dashboard for art museum analytics"
  - "create end-to-end data pipeline for cultural heritage data"
  - "query and visualize Harvard Art Museums collection"
---

# Harvard Artifacts Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL analytics, and interactive data visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media details, and color data into relational databases
- **SQL Analytics**: Run complex analytical queries on structured art collection data
- **Interactive Dashboards**: Visualize insights using Streamlit and Plotly
- **Database Design**: Proper relational schema with foreign key relationships

**Architecture**: API → ETL → SQL → Analytics → Visualization

## Installation

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

Create a `.env` file in the project root:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Setup

The application uses three main tables:

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
    accessionyear INT,
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    division VARCHAR(200),
    totalpageviews INT,
    totaluniquepageviews INT
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    imagespresent BOOLEAN,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
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

```bash
# Start the Streamlit dashboard
streamlit run app.py
```

The app will be available at `http://localhost:8501`

## Core Components

### 1. API Data Collection

```python
import requests
import pandas as pd
from typing import List, Dict
import time

class HarvardAPIClient:
    def __init__(self, api_key: str):
        self.api_key = api_key
        self.base_url = "https://api.harvardartmuseums.org/object"
        
    def fetch_artifacts(self, size: int = 100, max_pages: int = 10) -> List[Dict]:
        """Fetch artifacts with pagination"""
        all_artifacts = []
        
        for page in range(1, max_pages + 1):
            params = {
                'apikey': self.api_key,
                'size': size,
                'page': page,
                'hasimage': 1  # Only artifacts with images
            }
            
            response = requests.get(self.base_url, params=params)
            
            if response.status_code == 200:
                data = response.json()
                all_artifacts.extend(data.get('records', []))
                
                # Rate limiting
                time.sleep(0.5)
            else:
                print(f"Error fetching page {page}: {response.status_code}")
                break
                
        return all_artifacts
```

### 2. ETL Pipeline

```python
import mysql.connector
from typing import List, Dict

class ArtifactETL:
    def __init__(self, db_config: Dict):
        self.db_config = db_config
        
    def extract(self, api_client: HarvardAPIClient, num_artifacts: int = 500):
        """Extract data from API"""
        artifacts = api_client.fetch_artifacts(size=100, max_pages=num_artifacts//100)
        return artifacts
    
    def transform(self, artifacts: List[Dict]) -> tuple:
        """Transform nested JSON to relational format"""
        metadata_list = []
        media_list = []
        colors_list = []
        
        for artifact in artifacts:
            # Extract metadata
            metadata = {
                'id': artifact.get('id'),
                'title': artifact.get('title', '')[:500],
                'culture': artifact.get('culture', '')[:200],
                'century': artifact.get('century', '')[:100],
                'classification': artifact.get('classification', '')[:200],
                'department': artifact.get('department', '')[:200],
                'dated': artifact.get('dated', '')[:200],
                'accessionyear': artifact.get('accessionyear'),
                'technique': artifact.get('technique', '')[:500],
                'medium': artifact.get('medium', '')[:500],
                'dimensions': artifact.get('dimensions', '')[:500],
                'creditline': artifact.get('creditline', ''),
                'division': artifact.get('division', '')[:200],
                'totalpageviews': artifact.get('totalpageviews', 0),
                'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
            }
            metadata_list.append(metadata)
            
            # Extract media
            media = {
                'artifact_id': artifact.get('id'),
                'baseimageurl': artifact.get('baseimageurl', '')[:500],
                'primaryimageurl': artifact.get('primaryimageurl', '')[:500],
                'iiifbaseuri': artifact.get('images', [{}])[0].get('iiifbaseuri', '')[:500] if artifact.get('images') else '',
                'imagespresent': 1 if artifact.get('images') else 0
            }
            media_list.append(media)
            
            # Extract colors
            for color_data in artifact.get('colors', []):
                color = {
                    'artifact_id': artifact.get('id'),
                    'color': color_data.get('color', '')[:50],
                    'spectrum': color_data.get('spectrum', '')[:50],
                    'hue': color_data.get('hue', '')[:50],
                    'percent': color_data.get('percent', 0.0)
                }
                colors_list.append(color)
        
        return (
            pd.DataFrame(metadata_list),
            pd.DataFrame(media_list),
            pd.DataFrame(colors_list)
        )
    
    def load(self, metadata_df, media_df, colors_df):
        """Load data into MySQL database"""
        conn = mysql.connector.connect(**self.db_config)
        cursor = conn.cursor()
        
        # Insert metadata (batch)
        metadata_query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, department, dated, 
             accessionyear, technique, medium, dimensions, creditline, 
             division, totalpageviews, totaluniquepageviews)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_query, metadata_df.values.tolist())
        
        # Insert media
        media_query = """
            INSERT INTO artifactmedia 
            (artifact_id, baseimageurl, primaryimageurl, iiifbaseuri, imagespresent)
            VALUES (%s, %s, %s, %s, %s)
        """
        cursor.executemany(media_query, media_df.values.tolist())
        
        # Insert colors
        if not colors_df.empty:
            colors_query = """
                INSERT INTO artifactcolors 
                (artifact_id, color, spectrum, hue, percent)
                VALUES (%s, %s, %s, %s, %s)
            """
            cursor.executemany(colors_query, colors_df.values.tolist())
        
        conn.commit()
        cursor.close()
        conn.close()
```

### 3. SQL Analytics Queries

```python
# Example analytical queries
ANALYTICS_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "artifacts_by_department": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "top_viewed_artifacts": """
        SELECT title, culture, century, totalpageviews
        FROM artifactmetadata
        ORDER BY totalpageviews DESC
        LIMIT 20
    """,
    
    "color_distribution": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "artifacts_with_images": """
        SELECT 
            SUM(CASE WHEN imagespresent = 1 THEN 1 ELSE 0 END) as with_images,
            SUM(CASE WHEN imagespresent = 0 THEN 1 ELSE 0 END) as without_images
        FROM artifactmedia
    """,
    
    "artifacts_by_accession_year": """
        SELECT accessionyear, COUNT(*) as count
        FROM artifactmetadata
        WHERE accessionyear IS NOT NULL
        GROUP BY accessionyear
        ORDER BY accessionyear DESC
        LIMIT 20
    """
}

def run_analytics_query(query: str, db_config: Dict) -> pd.DataFrame:
    """Execute SQL query and return results as DataFrame"""
    conn = mysql.connector.connect(**db_config)
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px
from dotenv import load_dotenv
import os

# Load environment variables
load_dotenv()

# Database configuration
DB_CONFIG = {
    'host': os.getenv('DB_HOST'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums Collection Analytics")
st.markdown("---")

# Sidebar - Data Collection
with st.sidebar:
    st.header("Data Collection")
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching and loading data..."):
            api_key = os.getenv('HARVARD_API_KEY')
            client = HarvardAPIClient(api_key)
            etl = ArtifactETL(DB_CONFIG)
            
            # Extract
            artifacts = etl.extract(client, num_artifacts=500)
            
            # Transform
            metadata_df, media_df, colors_df = etl.transform(artifacts)
            
            # Load
            etl.load(metadata_df, media_df, colors_df)
            
            st.success(f"Loaded {len(metadata_df)} artifacts!")

# Main content - Analytics
st.header("📊 Analytics Dashboard")

# Query selector
query_name = st.selectbox(
    "Select Analysis",
    list(ANALYTICS_QUERIES.keys())
)

if st.button("Run Query"):
    df = run_analytics_query(ANALYTICS_QUERIES[query_name], DB_CONFIG)
    
    # Display results
    st.subheader("Query Results")
    st.dataframe(df, use_container_width=True)
    
    # Auto-generate visualization
    if len(df.columns) >= 2:
        st.subheader("Visualization")
        
        x_col = df.columns[0]
        y_col = df.columns[1]
        
        fig = px.bar(
            df.head(20), 
            x=x_col, 
            y=y_col,
            title=f"{query_name.replace('_', ' ').title()}"
        )
        st.plotly_chart(fig, use_container_width=True)
```

## Common Patterns

### Incremental Data Loading

```python
def incremental_load(api_client: HarvardAPIClient, last_id: int):
    """Load only new artifacts since last sync"""
    params = {
        'apikey': api_client.api_key,
        'after': last_id,
        'size': 100
    }
    response = requests.get(api_client.base_url, params=params)
    return response.json().get('records', [])
```

### Data Quality Checks

```python
def validate_artifact_data(df: pd.DataFrame) -> bool:
    """Validate required fields before loading"""
    required_fields = ['id', 'title']
    
    for field in required_fields:
        if field not in df.columns or df[field].isnull().any():
            return False
    
    return True
```

## Troubleshooting

### API Rate Limiting
If you encounter rate limiting errors:
```python
# Increase delay between requests
time.sleep(1)  # Wait 1 second instead of 0.5
```

### Database Connection Issues
```python
# Test connection
try:
    conn = mysql.connector.connect(**DB_CONFIG)
    print("Connection successful!")
    conn.close()
except Exception as e:
    print(f"Connection failed: {e}")
```

### Large Dataset Handling
```python
# Process in smaller batches
def batch_insert(cursor, query, data, batch_size=1000):
    for i in range(0, len(data), batch_size):
        batch = data[i:i+batch_size]
        cursor.executemany(query, batch)
        conn.commit()
```

### Missing Color Data
Some artifacts may not have color information:
```python
# Handle empty colors gracefully
if not colors_df.empty:
    cursor.executemany(colors_query, colors_df.values.tolist())
```

This skill enables comprehensive museum data engineering workflows from API extraction through visualization.
