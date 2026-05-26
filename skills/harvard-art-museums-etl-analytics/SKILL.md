---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard with Harvard artifacts
  - fetch and analyze Harvard museum collection data
  - set up data engineering project with museum API
  - visualize Harvard Art Museums data with Streamlit
  - extract transform load Harvard artifacts into SQL
  - query and analyze art museum collection data
  - create interactive museum data visualization
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualization using Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides a complete data pipeline architecture:

- **Extract**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **Transform**: Converts nested JSON into normalized relational data structures
- **Load**: Batch inserts data into SQL databases (MySQL/TiDB Cloud)
- **Analyze**: Executes predefined SQL queries for insights
- **Visualize**: Creates interactive dashboards with Plotly charts in Streamlit

The architecture follows: `API → ETL → SQL → Analytics → Visualization`

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
DB_USER=your_db_username
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Schema

The application uses three main tables:

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    dated VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    url TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    iiifbaseuri TEXT,
    baseimageurl TEXT,
    primaryimageurl TEXT,
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

## Key API Patterns

### Fetching Data from Harvard Art Museums API

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
    base_url = 'https://api.harvardartmuseums.org/object'
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only fetch artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        data = response.json()
        return data['records'], data['info']
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Example usage
artifacts, metadata = fetch_artifacts(page=1, size=50)
print(f"Total records: {metadata['totalrecords']}")
print(f"Pages: {metadata['pages']}")
```

### ETL Pipeline Implementation

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

class HarvardArtifactETL:
    def __init__(self, db_config: Dict):
        self.db_config = db_config
        self.connection = None
    
    def connect_db(self):
        """Establish database connection"""
        self.connection = mysql.connector.connect(
            host=self.db_config['host'],
            port=self.db_config['port'],
            user=self.db_config['user'],
            password=self.db_config['password'],
            database=self.db_config['database']
        )
    
    def extract(self, num_pages: int = 5) -> List[Dict]:
        """Extract data from Harvard API"""
        all_artifacts = []
        
        for page in range(1, num_pages + 1):
            artifacts, _ = fetch_artifacts(page=page, size=100)
            all_artifacts.extend(artifacts)
            print(f"Extracted page {page}: {len(artifacts)} records")
        
        return all_artifacts
    
    def transform(self, artifacts: List[Dict]) -> tuple:
        """Transform artifacts into normalized tables"""
        metadata_records = []
        media_records = []
        color_records = []
        
        for artifact in artifacts:
            # Extract metadata
            metadata = {
                'id': artifact.get('id'),
                'title': artifact.get('title', '')[:500],
                'culture': artifact.get('culture', '')[:255],
                'dated': artifact.get('dated', '')[:255],
                'century': artifact.get('century', '')[:100],
                'classification': artifact.get('classification', '')[:255],
                'department': artifact.get('department', '')[:255],
                'division': artifact.get('division', '')[:255],
                'technique': artifact.get('technique', '')[:500],
                'medium': artifact.get('medium', '')[:500],
                'url': artifact.get('url', '')
            }
            metadata_records.append(metadata)
            
            # Extract media information
            if 'images' in artifact and artifact['images']:
                media = {
                    'artifact_id': artifact.get('id'),
                    'iiifbaseuri': artifact.get('images', [{}])[0].get('iiifbaseuri', ''),
                    'baseimageurl': artifact.get('images', [{}])[0].get('baseimageurl', ''),
                    'primaryimageurl': artifact.get('primaryimageurl', '')
                }
                media_records.append(media)
            
            # Extract color information
            if 'colors' in artifact and artifact['colors']:
                for color in artifact['colors']:
                    color_record = {
                        'artifact_id': artifact.get('id'),
                        'color': color.get('color', ''),
                        'spectrum': color.get('spectrum', ''),
                        'hue': color.get('hue', ''),
                        'percent': color.get('percent', 0.0)
                    }
                    color_records.append(color_record)
        
        return (
            pd.DataFrame(metadata_records),
            pd.DataFrame(media_records),
            pd.DataFrame(color_records)
        )
    
    def load(self, df_metadata: pd.DataFrame, df_media: pd.DataFrame, 
             df_colors: pd.DataFrame):
        """Load data into SQL database"""
        cursor = self.connection.cursor()
        
        # Insert metadata
        for _, row in df_metadata.iterrows():
            query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, dated, century, classification, 
             department, division, technique, medium, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
            """
            cursor.execute(query, tuple(row))
        
        # Insert media
        for _, row in df_media.iterrows():
            query = """
            INSERT INTO artifactmedia 
            (artifact_id, iiifbaseuri, baseimageurl, primaryimageurl)
            VALUES (%s, %s, %s, %s)
            """
            cursor.execute(query, tuple(row))
        
        # Insert colors
        for _, row in df_colors.iterrows():
            query = """
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
            """
            cursor.execute(query, tuple(row))
        
        self.connection.commit()
        cursor.close()
        print(f"Loaded {len(df_metadata)} artifacts, {len(df_media)} media, {len(df_colors)} colors")

# Usage
db_config = {
    'host': os.getenv('DB_HOST'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

etl = HarvardArtifactETL(db_config)
etl.connect_db()

artifacts = etl.extract(num_pages=10)
df_meta, df_media, df_colors = etl.transform(artifacts)
etl.load(df_meta, df_media, df_colors)
```

## Streamlit Dashboard Application

```python
import streamlit as st
import pandas as pd
import plotly.express as px
import mysql.connector

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

# Database connection
@st.cache_resource
def get_db_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def execute_query(query: str) -> pd.DataFrame:
    """Execute SQL query and return results as DataFrame"""
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    return df

# Sidebar navigation
st.sidebar.title("Navigation")
page = st.sidebar.radio("Select Page", ["ETL Pipeline", "Analytics Dashboard", "Visualizations"])

if page == "ETL Pipeline":
    st.title("🔄 ETL Pipeline")
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Running ETL..."):
            etl = HarvardArtifactETL(db_config)
            etl.connect_db()
            artifacts = etl.extract(num_pages=5)
            df_meta, df_media, df_colors = etl.transform(artifacts)
            etl.load(df_meta, df_media, df_colors)
            st.success(f"✅ Loaded {len(df_meta)} artifacts successfully!")

elif page == "Analytics Dashboard":
    st.title("📊 Analytics Dashboard")
    
    queries = {
        "Top 10 Cultures by Artifact Count": """
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
            SELECT color, COUNT(*) as count, AVG(percent) as avg_percent
            FROM artifactcolors
            GROUP BY color
            ORDER BY count DESC
            LIMIT 15
        """
    }
    
    selected_query = st.selectbox("Select Query", list(queries.keys()))
    
    if st.button("Execute Query"):
        df = execute_query(queries[selected_query])
        st.dataframe(df, use_container_width=True)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)

elif page == "Visualizations":
    st.title("📈 Interactive Visualizations")
    
    # Color spectrum analysis
    query = """
        SELECT spectrum, COUNT(*) as count
        FROM artifactcolors
        GROUP BY spectrum
        ORDER BY count DESC
    """
    df = execute_query(query)
    fig = px.pie(df, values='count', names='spectrum', 
                 title='Color Spectrum Distribution')
    st.plotly_chart(fig, use_container_width=True)
```

## Common Analytical Queries

```sql
-- 1. Artifacts with images vs without
SELECT 
    CASE WHEN primaryimageurl IS NOT NULL THEN 'With Image' ELSE 'Without Image' END as image_status,
    COUNT(*) as count
FROM artifactmetadata m
LEFT JOIN artifactmedia med ON m.id = med.artifact_id
GROUP BY image_status;

-- 2. Top techniques used
SELECT technique, COUNT(*) as count
FROM artifactmetadata
WHERE technique IS NOT NULL AND technique != ''
GROUP BY technique
ORDER BY count DESC
LIMIT 20;

-- 3. Color diversity per artifact
SELECT artifact_id, COUNT(DISTINCT color) as color_count
FROM artifactcolors
GROUP BY artifact_id
ORDER BY color_count DESC
LIMIT 10;

-- 4. Artifacts by classification and century
SELECT classification, century, COUNT(*) as count
FROM artifactmetadata
WHERE classification IS NOT NULL AND century IS NOT NULL
GROUP BY classification, century
ORDER BY count DESC;
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# The app will open in your browser at http://localhost:8501
```

## Troubleshooting

**API Rate Limiting**: The Harvard API has rate limits. Add delays between requests:
```python
import time
time.sleep(0.5)  # 500ms delay between requests
```

**Database Connection Errors**: Verify credentials and network access:
```python
try:
    conn = mysql.connector.connect(**db_config)
    print("✅ Database connected")
except mysql.connector.Error as err:
    print(f"❌ Error: {err}")
```

**Large Dataset Issues**: Use batch processing:
```python
batch_size = 1000
for i in range(0, len(df), batch_size):
    batch = df.iloc[i:i+batch_size]
    # Process batch
```

**Missing Data Fields**: Always use `.get()` with defaults when accessing API response fields to avoid KeyErrors.
