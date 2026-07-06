---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing solution for logistics, fleet management, and supply chain analytics with multi-fact star schema
triggers:
  - "set up logifleet pulse supply chain analytics"
  - "deploy logistics data warehouse with power bi"
  - "configure fleet and warehouse analytics dashboard"
  - "implement multi-fact star schema for logistics"
  - "create supply chain intelligence reporting"
  - "build warehouse gravity zones analysis"
  - "setup cross-modal logistics KPI tracking"
  - "integrate fleet telemetry with warehouse operations"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

LogiFleet Pulse is a comprehensive data warehousing and business intelligence solution for logistics operations. It provides:

- **Multi-fact star schema** linking warehouse operations, fleet telemetry, and supply chain data
- **Power BI dashboards** for real-time logistics intelligence
- **Cross-fact KPI harmonization** between inventory, fleet, and delivery metrics
- **Predictive analytics** for bottleneck detection and fleet maintenance triage
- **Warehouse gravity zones** for optimized storage placement
- **Time-phased dimensions** with 15-minute granularity for operational tracking

The platform integrates data from WMS, GPS/telematics, supplier portals, and external APIs to provide unified supply chain visibility.

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Appropriate database permissions (db_owner for initial setup)

### Step 1: Clone Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### Step 2: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance via SSMS or Azure Data Studio
-- Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Run the schema deployment script
-- Execute schema/01_create_dimensions.sql
-- Execute schema/02_create_facts.sql
-- Execute schema/03_create_bridges.sql
-- Execute schema/04_create_views.sql
-- Execute schema/05_create_procedures.sql
```

### Step 3: Configure Data Sources

Update `config_sample.json` with your connection details:

```json
{
  "sql_server": {
    "server": "YOUR_SQL_SERVER",
    "database": "LogiFleetPulse",
    "auth_mode": "windows|sql",
    "username": "${SQL_USERNAME}",
    "password": "${SQL_PASSWORD}"
  },
  "data_sources": {
    "wms_api": "${WMS_API_ENDPOINT}",
    "telematics_api": "${FLEET_API_ENDPOINT}",
    "weather_api": "${WEATHER_API_KEY}"
  }
}
```

### Step 4: Import Power BI Template

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. Enter SQL Server connection details when prompted
3. Configure row-level security roles if needed
4. Publish to Power BI Service (optional)

## Core Data Model Structure

### Dimension Tables

```sql
-- DimTime: 15-minute granularity time tracking
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2(0) NOT NULL,
    Date DATE NOT NULL,
    HourOfDay TINYINT,
    DayOfWeek TINYINT,
    WeekOfYear TINYINT,
    FiscalQuarter TINYINT,
    IsWeekend BIT,
    IsHoliday BIT
);

-- DimGeography: Hierarchical location tracking
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID NVARCHAR(50) NOT NULL,
    LocationName NVARCHAR(100),
    LocationType NVARCHAR(20), -- 'Warehouse', 'Route_Node', 'Dock'
    ParentGeographyKey INT,
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    Region NVARCHAR(50),
    Country NVARCHAR(50),
    CONSTRAINT FK_Geography_Parent FOREIGN KEY (ParentGeographyKey) 
        REFERENCES DimGeography(GeographyKey)
);

-- DimProductGravity: Warehouse velocity tracking
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU NVARCHAR(50) NOT NULL UNIQUE,
    ProductName NVARCHAR(200),
    Category NVARCHAR(50),
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value / fragility
    VelocityClass NVARCHAR(20), -- 'Fast', 'Medium', 'Slow', 'Obsolete'
    FragilityScore TINYINT, -- 1-10 scale
    UnitValue DECIMAL(10,2),
    OptimalZone NVARCHAR(20),
    LastRecalcDate DATE
);
```

### Fact Tables

```sql
-- FactWarehouseOperations: Granular warehouse activity tracking
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    GeographyKey INT NOT NULL, -- Warehouse/Zone
    OperationType NVARCHAR(20), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    QuantityHandled INT,
    DurationMinutes DECIMAL(8,2),
    OperatorID NVARCHAR(50),
    BatchID NVARCHAR(50),
    DwellTimeHours DECIMAL(8,2), -- Time item spent in location
    QualityFlags NVARCHAR(100), -- Damage, defect indicators
    CONSTRAINT FK_WH_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WH_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    CONSTRAINT FK_WH_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey)
);

-- FactFleetTrips: Fleet telemetry and route performance
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    StartTimeKey INT NOT NULL,
    EndTimeKey INT NOT NULL,
    VehicleID NVARCHAR(50) NOT NULL,
    DriverID NVARCHAR(50),
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    PlannedDistanceKM DECIMAL(8,2),
    ActualDistanceKM DECIMAL(8,2),
    FuelConsumedLiters DECIMAL(8,2),
    IdleTimeMinutes DECIMAL(8,2),
    LoadWeightKG DECIMAL(10,2),
    TripStatus NVARCHAR(20), -- 'Completed', 'Delayed', 'InProgress'
    DelayReasonCode NVARCHAR(50),
    CONSTRAINT FK_Fleet_StartTime FOREIGN KEY (StartTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_EndTime FOREIGN KEY (EndTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_Fleet_Destination FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
);

-- Bridge table for many-to-many relationships
CREATE TABLE BridgeTripShipments (
    TripKey BIGINT NOT NULL,
    OperationKey BIGINT NOT NULL,
    LoadSequence TINYINT,
    PRIMARY KEY (TripKey, OperationKey),
    CONSTRAINT FK_Bridge_Trip FOREIGN KEY (TripKey) REFERENCES FactFleetTrips(TripKey),
    CONSTRAINT FK_Bridge_Operation FOREIGN KEY (OperationKey) REFERENCES FactWarehouseOperations(OperationKey)
);
```

## Key Stored Procedures

### Loading Data

```sql
-- Incremental load procedure for warehouse operations
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LoadDate DATE = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    IF @LoadDate IS NULL SET @LoadDate = CAST(GETDATE() AS DATE);
    
    -- Merge from staging into fact table
    MERGE INTO FactWarehouseOperations AS target
    USING (
        SELECT 
            t.TimeKey,
            p.ProductKey,
            g.GeographyKey,
            stg.OperationType,
            stg.QuantityHandled,
            stg.DurationMinutes,
            stg.OperatorID,
            stg.BatchID,
            stg.DwellTimeHours,
            stg.QualityFlags
        FROM StagingWarehouseOps stg
        INNER JOIN DimTime t ON stg.OperationTimestamp = t.FullDateTime
        INNER JOIN DimProductGravity p ON stg.SKU = p.SKU
        INNER JOIN DimGeography g ON stg.LocationID = g.LocationID
        WHERE CAST(stg.OperationTimestamp AS DATE) = @LoadDate
    ) AS source
    ON 1=0 -- Always insert for append-only fact
    WHEN NOT MATCHED THEN
        INSERT (TimeKey, ProductKey, GeographyKey, OperationType, 
                QuantityHandled, DurationMinutes, OperatorID, BatchID, 
                DwellTimeHours, QualityFlags)
        VALUES (source.TimeKey, source.ProductKey, source.GeographyKey, 
                source.OperationType, source.QuantityHandled, 
                source.DurationMinutes, source.OperatorID, source.BatchID,
                source.DwellTimeHours, source.QualityFlags);
    
    -- Recalculate gravity scores for affected products
    EXEC usp_RecalculateGravityScores @LoadDate;
END;
GO
```

### Calculating Gravity Zones

```sql
CREATE PROCEDURE usp_RecalculateGravityScores
    @AsOfDate DATE = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    IF @AsOfDate IS NULL SET @AsOfDate = CAST(GETDATE() AS DATE);
    
    UPDATE p
    SET 
        GravityScore = (velocity.PicksPerDay * p.UnitValue) / NULLIF(p.FragilityScore, 0),
        VelocityClass = CASE 
            WHEN velocity.PicksPerDay >= 20 THEN 'Fast'
            WHEN velocity.PicksPerDay >= 5 THEN 'Medium'
            WHEN velocity.PicksPerDay >= 1 THEN 'Slow'
            ELSE 'Obsolete'
        END,
        OptimalZone = CASE 
            WHEN (velocity.PicksPerDay * p.UnitValue) / NULLIF(p.FragilityScore, 0) >= 100 THEN 'HighGravity_A'
            WHEN (velocity.PicksPerDay * p.UnitValue) / NULLIF(p.FragilityScore, 0) >= 50 THEN 'MediumGravity_B'
            ELSE 'LowGravity_C'
        END,
        LastRecalcDate = @AsOfDate
    FROM DimProductGravity p
    CROSS APPLY (
        SELECT 
            COUNT(*) * 1.0 / 30 AS PicksPerDay
        FROM FactWarehouseOperations wo
        INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
        WHERE wo.ProductKey = p.ProductKey
            AND wo.OperationType = 'Picking'
            AND t.Date >= DATEADD(DAY, -30, @AsOfDate)
    ) velocity;
END;
GO
```

## Power BI DAX Measures

### Cross-Fact KPIs

```dax
// Total Warehouse Throughput (units/hour)
Warehouse Throughput = 
VAR TotalUnits = SUM(FactWarehouseOperations[QuantityHandled])
VAR TotalHours = SUM(FactWarehouseOperations[DurationMinutes]) / 60
RETURN
DIVIDE(TotalUnits, TotalHours, 0)

// Fleet Fuel Efficiency (km/liter)
Fleet Fuel Efficiency = 
DIVIDE(
    SUM(FactFleetTrips[ActualDistanceKM]),
    SUM(FactFleetTrips[FuelConsumedLiters]),
    0
)

// Cross-Fact: Cost per Delivered Unit
Cost Per Unit = 
VAR FuelCost = SUM(FactFleetTrips[FuelConsumedLiters]) * 1.5 // Assuming $1.50/liter
VAR LaborHours = SUM(FactWarehouseOperations[DurationMinutes]) / 60
VAR LaborCost = LaborHours * 25 // Assuming $25/hour
VAR TotalUnitsDelivered = 
    CALCULATE(
        SUM(FactWarehouseOperations[QuantityHandled]),
        FactWarehouseOperations[OperationType] = "Shipping"
    )
RETURN
DIVIDE(FuelCost + LaborCost, TotalUnitsDelivered, 0)

// Predictive Bottleneck Index
Bottleneck Risk Score = 
VAR AvgDwellTime = AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR StdDevDwell = STDEV.P(FactWarehouseOperations[DwellTimeHours])
VAR CurrentDwell = MAX(FactWarehouseOperations[DwellTimeHours])
VAR IdleTimeRatio = DIVIDE(SUM(FactFleetTrips[IdleTimeMinutes]), SUM(FactFleetTrips[ActualDistanceKM]) * 60, 0)
RETURN
(CurrentDwell - AvgDwellTime) / NULLIF(StdDevDwell, 0) * (1 + IdleTimeRatio)

// Warehouse Gravity Zone Compliance
Gravity Compliance % = 
VAR ItemsInOptimalZone = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[OperationType] = "Putaway",
        DimGeography[LocationType] = DimProductGravity[OptimalZone]
    )
VAR TotalPutaways = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[OperationType] = "Putaway"
    )
RETURN
DIVIDE(ItemsInOptimalZone, TotalPutaways, 0)
```

## Common Usage Patterns

### Pattern 1: Daily Operations Dashboard

```sql
-- View for Power BI: Today's operations summary
CREATE VIEW vw_TodayOperationsSummary AS
SELECT 
    t.HourOfDay,
    g.LocationName AS Warehouse,
    p.VelocityClass,
    COUNT(DISTINCT wo.OperationKey) AS TotalOperations,
    SUM(wo.QuantityHandled) AS UnitsProcessed,
    AVG(wo.DurationMinutes) AS AvgDurationMinutes,
    AVG(wo.DwellTimeHours) AS AvgDwellHours
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
WHERE t.Date = CAST(GETDATE() AS DATE)
GROUP BY t.HourOfDay, g.LocationName, p.VelocityClass;
GO
```

### Pattern 2: Fleet Performance Analysis

```sql
-- Identify underperforming routes
SELECT 
    o.LocationName AS Origin,
    d.LocationName AS Destination,
    COUNT(ft.TripKey) AS TotalTrips,
    AVG(ft.ActualDistanceKM - ft.PlannedDistanceKM) AS AvgDistanceVariance,
    AVG(ft.IdleTimeMinutes) AS AvgIdleMinutes,
    SUM(CASE WHEN ft.TripStatus = 'Delayed' THEN 1 ELSE 0 END) * 100.0 / COUNT(*) AS DelayRate
FROM FactFleetTrips ft
INNER JOIN DimGeography o ON ft.OriginGeographyKey = o.GeographyKey
INNER JOIN DimGeography d ON ft.DestinationGeographyKey = d.GeographyKey
INNER JOIN DimTime st ON ft.StartTimeKey = st.TimeKey
WHERE st.Date >= DATEADD(DAY, -30, CAST(GETDATE() AS DATE))
GROUP BY o.LocationName, d.LocationName
HAVING AVG(ft.IdleTimeMinutes) > 30
ORDER BY DelayRate DESC;
```

### Pattern 3: Inventory Aging Alert

```sql
-- Create alert procedure for stale inventory
CREATE PROCEDURE usp_CheckInventoryAging
    @ThresholdDays INT = 90
AS
BEGIN
    SELECT 
        p.SKU,
        p.ProductName,
        g.LocationName,
        MAX(wo.DwellTimeHours) / 24.0 AS DaysInStorage,
        p.VelocityClass,
        p.UnitValue * SUM(wo.QuantityHandled) AS ValueAtRisk
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
    WHERE wo.OperationType IN ('Receiving', 'Putaway')
        AND wo.DwellTimeHours > (@ThresholdDays * 24)
    GROUP BY p.SKU, p.ProductName, g.LocationName, p.VelocityClass, p.UnitValue
    ORDER BY ValueAtRisk DESC;
END;
GO

-- Schedule via SQL Agent job for daily email alerts
```

## Configuration

### Row-Level Security Setup

```sql
-- Create security roles in Power BI
CREATE TABLE SecurityRoles (
    RoleID INT PRIMARY KEY IDENTITY(1,1),
    RoleName NVARCHAR(50) NOT NULL,
    UserEmail NVARCHAR(100) NOT NULL,
    GeographyFilter NVARCHAR(200), -- e.g., 'Region = "North America"'
    StartDate DATE NOT NULL,
    EndDate DATE
);

-- Example RLS filter in Power BI (DAX)
-- Apply to DimGeography table
[Region] IN VALUES('SecurityRoles'[GeographyFilter])
```

### Automated Refresh Configuration

Power BI Service dataset settings:
- **Scheduled Refresh**: Every 15 minutes during business hours (6 AM - 10 PM)
- **Incremental Refresh**: Partition by Date, keep 2 years, refresh last 7 days
- **DirectQuery**: Enable for real-time dashboards (FactWarehouseOperations, FactFleetTrips)

```json
// dataset.json excerpt for incremental refresh
{
  "incrementalRefresh": {
    "table": "FactWarehouseOperations",
    "rangeStart": "2024-01-01",
    "rangeEnd": "GETDATE()",
    "refreshGranularity": "Day",
    "archiveStorage": true
  }
}
```

## Troubleshooting

### Issue: Power BI Connection Timeout

```sql
-- Add indexes for common filter patterns
CREATE NONCLUSTERED INDEX IX_WH_TimeProduct 
ON FactWarehouseOperations(TimeKey, ProductKey) 
INCLUDE (QuantityHandled, DurationMinutes);

CREATE NONCLUSTERED INDEX IX_Fleet_TimeRoute
ON FactFleetTrips(StartTimeKey, OriginGeographyKey, DestinationGeographyKey)
INCLUDE (ActualDistanceKM, FuelConsumedLiters);

-- Enable table partitioning for large fact tables
CREATE PARTITION FUNCTION pf_Date (DATE)
AS RANGE RIGHT FOR VALUES 
    ('2024-01-01', '2024-02-01', '2024-03-01', /* ... monthly partitions ... */);
```

### Issue: Gravity Score Anomalies

```sql
-- Validate gravity calculations
SELECT 
    SKU,
    ProductName,
    GravityScore,
    VelocityClass,
    UnitValue,
    FragilityScore,
    (SELECT COUNT(*) FROM FactWarehouseOperations wo 
     WHERE wo.ProductKey = p.ProductKey 
       AND wo.OperationType = 'Picking'
       AND wo.TimeKey IN (SELECT TimeKey FROM DimTime WHERE Date >= DATEADD(DAY, -30, GETDATE()))
    ) AS RecentPicks
FROM DimProductGravity p
WHERE GravityScore IS NULL OR GravityScore < 0
ORDER BY SKU;

-- Recalculate if needed
EXEC usp_RecalculateGravityScores;
```

### Issue: Cross-Fact Query Performance

```sql
-- Optimize bridge table queries with filtered indexes
CREATE NONCLUSTERED INDEX IX_Bridge_Trip_Filtered
ON BridgeTripShipments(TripKey)
INCLUDE (OperationKey)
WHERE LoadSequence = 1; -- First load only

-- Use query hints in views for complex joins
CREATE VIEW vw_CrossFactAnalysis WITH SCHEMABINDING AS
SELECT 
    t.Date,
    COUNT(DISTINCT ft.TripKey) AS TotalTrips,
    SUM(wo.QuantityHandled) AS UnitsShipped
FROM dbo.FactFleetTrips ft
INNER JOIN dbo.BridgeTripShipments bts ON ft.TripKey = bts.TripKey
INNER JOIN dbo.FactWarehouseOperations wo ON bts.OperationKey = wo.OperationKey
INNER JOIN dbo.DimTime t ON ft.StartTimeKey = t.TimeKey
GROUP BY t.Date
OPTION (HASH JOIN, MAXDOP 4);
GO
```

## Environment Variables

Required environment variables for automated ETL:

```bash
# SQL Server connection
export SQL_SERVER="your-server.database.windows.net"
export SQL_DATABASE="LogiFleetPulse"
export SQL_USERNAME="${SQL_USERNAME}"
export SQL_PASSWORD="${SQL_PASSWORD}"

# External API keys
export WMS_API_ENDPOINT="${WMS_API_ENDPOINT}"
export WMS_API_KEY="${WMS_API_KEY}"
export FLEET_API_ENDPOINT="${FLEET_API_ENDPOINT}"
export FLEET_API_KEY="${FLEET_API_KEY}"
export WEATHER_API_KEY="${WEATHER_API_KEY}"

# Power BI service
export POWERBI_WORKSPACE_ID="${POWERBI_WORKSPACE_ID}"
export POWERBI_CLIENT_ID="${POWERBI_CLIENT_ID}"
export POWERBI_CLIENT_SECRET="${POWERBI_CLIENT_SECRET}"
```

## Advanced: Python Integration for Predictive Analytics

```python
import pyodbc
import pandas as pd
from sklearn.ensemble import RandomForestRegressor
import os

# Connect to SQL Server
conn = pyodbc.connect(
    f"Driver={{ODBC Driver 17 for SQL Server}};"
    f"Server={os.getenv('SQL_SERVER')};"
    f"Database={os.getenv('SQL_DATABASE')};"
    f"UID={os.getenv('SQL_USERNAME')};"
    f"PWD={os.getenv('SQL_PASSWORD')}"
)

# Load training data for bottleneck prediction
query = """
SELECT 
    wo.DwellTimeHours,
    p.GravityScore,
    g.LocationType,
    t.HourOfDay,
    t.DayOfWeek,
    COUNT(*) OVER (PARTITION BY wo.GeographyKey, t.Date) AS DailyVolume,
    LEAD(wo.DwellTimeHours) OVER (PARTITION BY wo.ProductKey ORDER BY t.FullDateTime) AS NextDwellTime
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE t.Date >= DATEADD(DAY, -90, GETDATE())
"""

df = pd.read_sql(query, conn)
df = df.dropna(subset=['NextDwellTime'])

# Train model
features = ['DwellTimeHours', 'GravityScore', 'HourOfDay', 'DayOfWeek', 'DailyVolume']
X = pd.get_dummies(df[features + ['LocationType']], drop_first=True)
y = df['NextDwellTime']

model = RandomForestRegressor(n_estimators=100, random_state=42)
model.fit(X, y)

# Score current inventory for bottleneck risk
current_query = """
SELECT 
    wo.OperationKey,
    wo.DwellTimeHours,
    p.GravityScore,
    g.LocationType,
    t.HourOfDay,
    t.DayOfWeek,
    COUNT(*) OVER (PARTITION BY wo.GeographyKey, t.Date) AS DailyVolume
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE wo.OperationType = 'Putaway' AND t.Date = CAST(GETDATE() AS DATE)
"""

current_df = pd.read_sql(current_query, conn)
X_current = pd.get_dummies(current_df[features + ['LocationType']], drop_first=True)
predictions = model.predict(X_current)

# Write back to SQL
current_df['PredictedDwellTime'] = predictions
current_df['BottleneckRisk'] = (current_df['PredictedDwellTime'] > 72).astype(int)

cursor = conn.cursor()
for _, row in current_df.iterrows():
    cursor.execute(
        "UPDATE FactWarehouseOperations SET PredictedDwellTime = ?, BottleneckRisk = ? WHERE OperationKey = ?",
        row['PredictedDwellTime'], row['BottleneckRisk'], row['OperationKey']
    )
conn.commit()
conn.close()
```

## Resources

- **SQL Scripts**: `/schema/` directory contains all DDL and stored procedures
- **Power BI Template**: `LogiFleet_Pulse_Master.pbit` in repository root
- **Sample Data Generator**: `/tools/data_generator.py` for testing
- **Documentation**: `/docs/` for detailed architecture diagrams

This skill enables AI agents to help developers deploy, configure, and extend the LogiFleet Pulse supply chain analytics platform effectively.
