---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehouse for logistics, fleet, and warehouse analytics with multi-fact star schema
triggers:
  - set up supply chain analytics dashboard
  - configure logifleet pulse warehouse
  - implement logistics data warehouse
  - create power bi fleet tracking dashboard
  - build warehouse operations analytics
  - deploy sql server logistics schema
  - integrate fleet telemetry with warehouse data
  - setup cross-modal supply chain intelligence
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a multi-modal logistics intelligence platform combining MS SQL Server data warehousing with Power BI visualization. It integrates warehouse operations, fleet telemetry, inventory management, and external data sources into a unified semantic layer using a multi-fact star schema architecture.

**Core capabilities:**
- Multi-fact star schema with time-phased dimensions
- Cross-fact KPI harmonization (warehouse + fleet metrics)
- Real-time dashboard updates (15-minute granularity)
- Predictive bottleneck detection
- Warehouse gravity zone optimization
- Fleet maintenance triage based on revenue impact

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) recommended
- Access to source systems: WMS, TMS, telematics APIs

### Deployment Steps

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy SQL schema:**
```sql
-- Connect to your SQL Server instance via SSMS
-- Execute the schema creation script
:r schema/01_create_database.sql
:r schema/02_create_dimensions.sql
:r schema/03_create_facts.sql
:r schema/04_create_views.sql
:r schema/05_create_stored_procedures.sql
```

3. **Configure data source connections:**
```json
-- Edit config_sample.json and rename to config.json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_USER}",
    "password": "${SQL_PASSWORD}"
  },
  "wms_api": {
    "endpoint": "${WMS_API_ENDPOINT}",
    "api_key": "${WMS_API_KEY}"
  },
  "telematics_api": {
    "endpoint": "${TELEMATICS_API_ENDPOINT}",
    "api_key": "${TELEMATICS_API_KEY}"
  }
}
```

4. **Import Power BI template:**
- Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
- Enter SQL Server connection details when prompted
- Verify data model relationships auto-generated correctly

## Database Schema Architecture

### Core Fact Tables

**FactWarehouseOperations** - Warehouse activity tracking:
```sql
CREATE TABLE FactWarehouseOperations (
    OperationID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    OperationType NVARCHAR(50), -- 'Putaway', 'Pick', 'Pack', 'Ship'
    QuantityHandled INT,
    DwellTimeMinutes INT,
    OperationDurationSeconds INT,
    EmployeeKey INT,
    ZoneGravityScore DECIMAL(5,2),
    RecordedAt DATETIME2 DEFAULT GETDATE()
);

-- Create clustered columnstore index for analytics
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactWarehouseOps 
ON FactWarehouseOperations;
```

**FactFleetTrips** - Fleet telemetry and routing:
```sql
CREATE TABLE FactFleetTrips (
    TripID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT,
    OriginGeographyKey INT,
    DestinationGeographyKey INT,
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadWeightKG INT,
    TripDurationMinutes INT,
    OnTimeStatus BIT,
    RecordedAt DATETIME2 DEFAULT GETDATE()
);

CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactFleetTrips 
ON FactFleetTrips;
```

**FactCrossDock** - Direct transfer operations:
```sql
CREATE TABLE FactCrossDock (
    CrossDockID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    DockDwellMinutes INT,
    QuantityTransferred INT,
    RecordedAt DATETIME2 DEFAULT GETDATE()
);
```

### Core Dimension Tables

**DimTime** - 15-minute granularity time dimension:
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2,
    Date DATE,
    Year INT,
    Quarter INT,
    Month INT,
    Week INT,
    DayOfYear INT,
    DayOfMonth INT,
    DayOfWeek INT,
    Hour INT,
    Minute15Bucket INT, -- 0, 15, 30, 45
    IsWeekend BIT,
    FiscalYear INT,
    FiscalQuarter INT,
    ShiftType NVARCHAR(20) -- 'Morning', 'Afternoon', 'Night'
);
```

**DimProductGravity** - Products with velocity scoring:
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU NVARCHAR(50) UNIQUE NOT NULL,
    ProductName NVARCHAR(200),
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    GravityScore DECIMAL(5,2), -- Calculated: velocity + value + fragility
    AverageVelocity DECIMAL(10,2), -- Units per week
    UnitValue DECIMAL(10,2),
    FragilityIndex INT, -- 1-10
    OptimalZoneType NVARCHAR(50),
    LastUpdated DATETIME2
);
```

**DimGeography** - Hierarchical location dimension:
```sql
CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationID NVARCHAR(50) UNIQUE,
    LocationName NVARCHAR(200),
    LocationType NVARCHAR(50), -- 'Warehouse', 'Route Node', 'Customer Site'
    Continent NVARCHAR(50),
    Country NVARCHAR(100),
    Region NVARCHAR(100),
    City NVARCHAR(100),
    PostalCode NVARCHAR(20),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6)
);
```

## Key Stored Procedures

### Incremental Data Loading

**Load warehouse operations from WMS:**
```sql
CREATE PROCEDURE usp_LoadWarehouseOperations
    @StartDate DATETIME2,
    @EndDate DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new operations from staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, WarehouseKey, OperationType,
        QuantityHandled, DwellTimeMinutes, OperationDurationSeconds,
        EmployeeKey, ZoneGravityScore
    )
    SELECT 
        t.TimeKey,
        p.ProductKey,
        w.WarehouseKey,
        s.OperationType,
        s.Quantity,
        DATEDIFF(MINUTE, s.StartTime, s.EndTime) AS DwellTime,
        DATEDIFF(SECOND, s.StartTime, s.EndTime) AS Duration,
        e.EmployeeKey,
        p.GravityScore
    FROM Staging_WarehouseOps s
    INNER JOIN DimTime t ON CAST(s.StartTime AS DATE) = t.Date 
        AND DATEPART(HOUR, s.StartTime) = t.Hour
        AND (DATEPART(MINUTE, s.StartTime) / 15) * 15 = t.Minute15Bucket
    INNER JOIN DimProductGravity p ON s.SKU = p.SKU
    INNER JOIN DimWarehouse w ON s.WarehouseCode = w.WarehouseCode
    LEFT JOIN DimEmployee e ON s.EmployeeID = e.EmployeeID
    WHERE s.StartTime >= @StartDate AND s.StartTime < @EndDate
    AND NOT EXISTS (
        SELECT 1 FROM FactWarehouseOperations f 
        WHERE f.RecordedAt = s.RecordedAt AND f.ProductKey = p.ProductKey
    );
    
    -- Update gravity scores based on new velocity
    EXEC usp_RecalculateGravityScores;
END;
```

### Cross-Fact KPI Calculation

**Calculate warehouse-fleet efficiency correlation:**
```sql
CREATE PROCEDURE usp_CalculateWarehouseFleetEfficiency
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    SELECT 
        t.Date,
        w.WarehouseName,
        -- Warehouse metrics
        AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
        SUM(wo.QuantityHandled) AS TotalUnitsHandled,
        COUNT(DISTINCT wo.ProductKey) AS DistinctSKUs,
        -- Fleet metrics (correlated by time and destination)
        AVG(ft.IdleTimeMinutes) AS AvgFleetIdleTime,
        AVG(ft.FuelConsumedLiters / NULLIF(ft.DistanceKM, 0)) AS AvgFuelEfficiency,
        SUM(CASE WHEN ft.OnTimeStatus = 1 THEN 1 ELSE 0 END) * 100.0 / COUNT(ft.TripID) AS OnTimePercentage,
        -- Cross-fact KPI
        CASE 
            WHEN AVG(wo.DwellTimeMinutes) > 120 AND AVG(ft.IdleTimeMinutes) > 30 
            THEN 'High Friction'
            WHEN AVG(wo.DwellTimeMinutes) < 60 AND AVG(ft.IdleTimeMinutes) < 15 
            THEN 'Optimized'
            ELSE 'Moderate'
        END AS EfficiencyStatus
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    INNER JOIN DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
    LEFT JOIN FactFleetTrips ft ON t.TimeKey = ft.TimeKey 
        AND w.GeographyKey = ft.OriginGeographyKey
    WHERE t.Date >= @StartDate AND t.Date <= @EndDate
    GROUP BY t.Date, w.WarehouseName, w.WarehouseKey
    ORDER BY t.Date, w.WarehouseName;
END;
```

### Predictive Bottleneck Detection

**Identify potential congestion points:**
```sql
CREATE PROCEDURE usp_PredictBottlenecks
    @PredictionHorizonHours INT = 4
AS
BEGIN
    WITH RecentTrends AS (
        SELECT 
            WarehouseKey,
            ProductKey,
            AVG(DwellTimeMinutes) AS AvgRecentDwell,
            STDEV(DwellTimeMinutes) AS StdDevDwell,
            COUNT(*) AS RecentOperations
        FROM FactWarehouseOperations
        WHERE RecordedAt >= DATEADD(HOUR, -@PredictionHorizonHours * 2, GETDATE())
        GROUP BY WarehouseKey, ProductKey
    ),
    CurrentState AS (
        SELECT 
            WarehouseKey,
            ProductKey,
            AVG(DwellTimeMinutes) AS CurrentAvgDwell,
            COUNT(*) AS CurrentOperations
        FROM FactWarehouseOperations
        WHERE RecordedAt >= DATEADD(HOUR, -1, GETDATE())
        GROUP BY WarehouseKey, ProductKey
    )
    SELECT 
        w.WarehouseName,
        p.ProductName,
        p.SKU,
        c.CurrentAvgDwell,
        r.AvgRecentDwell,
        r.StdDevDwell,
        -- Bottleneck risk score
        CASE 
            WHEN c.CurrentAvgDwell > (r.AvgRecentDwell + 2 * r.StdDevDwell) THEN 'CRITICAL'
            WHEN c.CurrentAvgDwell > (r.AvgRecentDwell + r.StdDevDwell) THEN 'WARNING'
            ELSE 'NORMAL'
        END AS RiskLevel,
        (c.CurrentAvgDwell - r.AvgRecentDwell) / NULLIF(r.StdDevDwell, 0) AS ZScore
    FROM CurrentState c
    INNER JOIN RecentTrends r ON c.WarehouseKey = r.WarehouseKey 
        AND c.ProductKey = r.ProductKey
    INNER JOIN DimWarehouse w ON c.WarehouseKey = w.WarehouseKey
    INNER JOIN DimProductGravity p ON c.ProductKey = p.ProductKey
    WHERE r.RecentOperations >= 10 -- Minimum data points
    ORDER BY ZScore DESC;
END;
```

## Power BI Integration

### Connection String Configuration

In Power BI Desktop, use this M query for SQL connection:
```m
let
    Source = Sql.Database(
        Environment.GetEnvironmentVariable("SQL_SERVER_HOST"), 
        "LogiFleetPulse",
        [
            Query="EXEC usp_GetWarehouseFleetSummary @StartDate='2026-01-01', @EndDate='2026-12-31'",
            CommandTimeout=#duration(0, 0, 10, 0)
        ]
    )
in
    Source
```

### DAX Measures for Cross-Fact Analysis

**Warehouse Efficiency Score:**
```dax
WarehouseEfficiency = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR TargetDwell = 60
VAR UnitsPerHour = DIVIDE(
    SUM(FactWarehouseOperations[QuantityHandled]),
    SUM(FactWarehouseOperations[OperationDurationSeconds]) / 3600
)
VAR TargetUnitsPerHour = 100
RETURN
    (1 - (AvgDwell / TargetDwell)) * 0.5 + 
    (UnitsPerHour / TargetUnitsPerHour) * 0.5
```

**Fleet Revenue Impact Score:**
```dax
FleetRevenueImpact = 
VAR TripRevenue = 
    SUMX(
        FactFleetTrips,
        FactFleetTrips[LoadWeightKG] * 
        RELATED(DimProductGravity[UnitValue])
    )
VAR IdleCost = 
    SUM(FactFleetTrips[IdleTimeMinutes]) * 2.5 -- $2.50 per idle minute
VAR FuelCost = 
    SUM(FactFleetTrips[FuelConsumedLiters]) * 1.5 -- $1.50 per liter
RETURN
    TripRevenue - IdleCost - FuelCost
```

**Cross-Modal Friction Index:**
```dax
FrictionIndex = 
VAR WarehouseDwellZ = 
    DIVIDE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]) - 60,
        STDEV.P(FactWarehouseOperations[DwellTimeMinutes])
    )
VAR FleetIdleZ = 
    DIVIDE(
        AVERAGE(FactFleetTrips[IdleTimeMinutes]) - 15,
        STDEV.P(FactFleetTrips[IdleTimeMinutes])
    )
RETURN
    (WarehouseDwellZ + FleetIdleZ) / 2
```

## Data Loading Patterns

### Incremental ETL from External APIs

**Python script for telematics ingestion:**
```python
import pyodbc
import requests
import os
from datetime import datetime, timedelta

# Database connection
conn_string = (
    f"DRIVER={{ODBC Driver 17 for SQL Server}};"
    f"SERVER={os.getenv('SQL_SERVER_HOST')};"
    f"DATABASE=LogiFleetPulse;"
    f"UID={os.getenv('SQL_USER')};"
    f"PWD={os.getenv('SQL_PASSWORD')}"
)

def load_fleet_telemetry(start_date, end_date):
    """Load fleet trip data from telematics API to SQL Server"""
    
    # Fetch data from API
    api_endpoint = os.getenv('TELEMATICS_API_ENDPOINT')
    api_key = os.getenv('TELEMATICS_API_KEY')
    
    response = requests.get(
        f"{api_endpoint}/trips",
        headers={"Authorization": f"Bearer {api_key}"},
        params={"start": start_date.isoformat(), "end": end_date.isoformat()}
    )
    
    trips = response.json()['data']
    
    # Insert into staging table
    conn = pyodbc.connect(conn_string)
    cursor = conn.cursor()
    
    # Truncate staging
    cursor.execute("TRUNCATE TABLE Staging_FleetTrips")
    
    # Bulk insert
    insert_query = """
        INSERT INTO Staging_FleetTrips (
            VehicleID, RouteID, DriverID, OriginLocationID, 
            DestinationLocationID, DistanceKM, FuelConsumedLiters,
            IdleTimeMinutes, LoadWeightKG, TripStartTime, TripEndTime, OnTime
        ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
    """
    
    batch = []
    for trip in trips:
        batch.append((
            trip['vehicle_id'],
            trip['route_id'],
            trip['driver_id'],
            trip['origin'],
            trip['destination'],
            trip['distance_km'],
            trip['fuel_liters'],
            trip['idle_minutes'],
            trip['load_kg'],
            trip['start_time'],
            trip['end_time'],
            trip['on_time']
        ))
        
        if len(batch) >= 1000:
            cursor.executemany(insert_query, batch)
            conn.commit()
            batch = []
    
    if batch:
        cursor.executemany(insert_query, batch)
        conn.commit()
    
    # Call stored procedure to load into fact table
    cursor.execute(
        "EXEC usp_LoadFleetTrips @StartDate=?, @EndDate=?",
        start_date, end_date
    )
    conn.commit()
    
    cursor.close()
    conn.close()
    
    print(f"Loaded {len(trips)} fleet trips")

if __name__ == "__main__":
    # Load last 24 hours
    end_date = datetime.now()
    start_date = end_date - timedelta(hours=24)
    load_fleet_telemetry(start_date, end_date)
```

### Scheduled Refresh with Power Automate

Use this template for automated Power BI refresh:
```json
{
  "trigger": {
    "type": "Recurrence",
    "recurrence": {
      "frequency": "Minute",
      "interval": 15
    }
  },
  "actions": {
    "RefreshDataset": {
      "type": "ApiConnection",
      "inputs": {
        "host": {
          "connection": {
            "name": "@parameters('$connections')['powerbi']['connectionId']"
          }
        },
        "method": "post",
        "path": "/v1.0/myorg/groups/@{parameters('WorkspaceId')}/datasets/@{parameters('DatasetId')}/refreshes"
      }
    }
  }
}
```

## Common Analytical Queries

### Top Bottleneck Products by Gravity Zone

```sql
SELECT TOP 20
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.OptimalZoneType,
    w.WarehouseName,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
    COUNT(*) AS OperationCount,
    RANK() OVER (PARTITION BY w.WarehouseKey ORDER BY AVG(wo.DwellTimeMinutes) DESC) AS DwellRank
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
INNER JOIN DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE t.Date >= DATEADD(DAY, -30, GETDATE())
GROUP BY p.SKU, p.ProductName, p.GravityScore, p.OptimalZoneType, w.WarehouseName, w.WarehouseKey
HAVING COUNT(*) >= 10
ORDER BY AvgDwellTime DESC;
```

### Fleet Idle Time Correlation with Warehouse Load

```sql
WITH WarehouseLoad AS (
    SELECT 
        wo.TimeKey,
        wo.WarehouseKey,
        COUNT(*) AS OperationsCount,
        AVG(wo.DwellTimeMinutes) AS AvgDwell
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.Date >= DATEADD(DAY, -7, GETDATE())
    GROUP BY wo.TimeKey, wo.WarehouseKey
),
FleetPerformance AS (
    SELECT 
        ft.TimeKey,
        ft.OriginGeographyKey,
        AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
        COUNT(*) AS TripCount
    FROM FactFleetTrips ft
    INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
    WHERE t.Date >= DATEADD(DAY, -7, GETDATE())
    GROUP BY ft.TimeKey, ft.OriginGeographyKey
)
SELECT 
    t.Date,
    t.Hour,
    w.WarehouseName,
    wl.OperationsCount,
    wl.AvgDwell,
    fp.AvgIdleTime,
    fp.TripCount,
    -- Correlation indicator
    CASE 
        WHEN wl.AvgDwell > 90 AND fp.AvgIdleTime > 25 THEN 'Correlated Delay'
        WHEN wl.AvgDwell < 45 AND fp.AvgIdleTime < 10 THEN 'Optimal Flow'
        ELSE 'Mixed Performance'
    END AS PerformancePattern
FROM WarehouseLoad wl
INNER JOIN FleetPerformance fp ON wl.TimeKey = fp.TimeKey
INNER JOIN DimWarehouse w ON wl.WarehouseKey = w.WarehouseKey
    AND w.GeographyKey = fp.OriginGeographyKey
INNER JOIN DimTime t ON wl.TimeKey = t.TimeKey
ORDER BY t.Date, t.Hour, w.WarehouseName;
```

### Revenue Impact of Delayed Shipments

```sql
SELECT 
    t.Date,
    g.Country,
    g.Region,
    COUNT(CASE WHEN ft.OnTimeStatus = 0 THEN 1 END) AS DelayedShipments,
    COUNT(*) AS TotalShipments,
    SUM(CASE WHEN ft.OnTimeStatus = 0 
        THEN ft.LoadWeightKG * p.UnitValue * 0.05 -- 5% penalty
        ELSE 0 
    END) AS EstimatedRevenueLoss,
    AVG(CASE WHEN ft.OnTimeStatus = 0 THEN ft.IdleTimeMinutes ELSE NULL END) AS AvgDelayIdleTime
FROM FactFleetTrips ft
INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON ft.DestinationGeographyKey = g.GeographyKey
LEFT JOIN FactCrossDock cd ON ft.TripID = cd.OutboundTripKey
LEFT JOIN DimProductGravity p ON cd.ProductKey = p.ProductKey
WHERE t.Date >= DATEADD(MONTH, -1, GETDATE())
GROUP BY t.Date, g.Country, g.Region
HAVING COUNT(CASE WHEN ft.OnTimeStatus = 0 THEN 1 END) > 0
ORDER BY EstimatedRevenueLoss DESC;
```

## Troubleshooting

### Performance Issues

**Slow cross-fact queries:**
```sql
-- Verify columnstore index health
SELECT 
    OBJECT_NAME(object_id) AS TableName,
    state_description,
    total_rows,
    deleted_rows,
    100.0 * deleted_rows / NULLIF(total_rows, 0) AS FragmentationPercent
FROM sys.dm_db_column_store_row_group_physical_stats
WHERE OBJECT_NAME(object_id) IN ('FactWarehouseOperations', 'FactFleetTrips')
ORDER BY FragmentationPercent DESC;

-- Rebuild if fragmentation > 10%
ALTER INDEX CCI_FactWarehouseOps ON FactWarehouseOperations REBUILD;
ALTER INDEX CCI_FactFleetTrips ON FactFleetTrips REBUILD;
```

**Power BI refresh timeouts:**
- Increase timeout in Power Query: `CommandTimeout=#duration(0, 0, 30, 0)`
- Partition fact tables by date range
- Use incremental refresh in Power BI settings

### Data Quality Issues

**Missing dimension lookups:**
```sql
-- Find orphaned fact records
SELECT 'FactWarehouseOperations' AS TableName, COUNT(*) AS OrphanedRecords
FROM FactWarehouseOperations wo
WHERE NOT EXISTS (SELECT 1 FROM DimProductGravity p WHERE p.ProductKey = wo.ProductKey)

UNION ALL

SELECT 'FactFleetTrips', COUNT(*)
FROM FactFleetTrips ft
WHERE NOT EXISTS (SELECT 1 FROM DimVehicle v WHERE v.VehicleKey = ft.VehicleKey);

-- Add missing dimension entries
INSERT INTO DimProductGravity (SKU, ProductName, GravityScore)
SELECT DISTINCT s.SKU, 'Unknown Product', 0.0
FROM Staging_WarehouseOps s
WHERE NOT EXISTS (SELECT 1 FROM DimProductGravity p WHERE p.SKU = s.SKU);
```

### Incorrect Gravity Scores

Recalculate based on recent velocity:
```sql
CREATE PROCEDURE usp_RecalculateGravityScores
AS
BEGIN
    WITH VelocityStats AS (
        SELECT 
            ProductKey,
            COUNT(*) / 7.0 AS UnitsPerWeek, -- Last 7 days
            AVG(DwellTimeMinutes) AS AvgDwell
        FROM FactWarehouseOperations
        WHERE RecordedAt >= DATEADD(DAY, -7, GETDATE())
        GROUP BY ProductKey
    )
    UPDATE p
    SET 
        p.AverageVelocity = ISNULL(v.UnitsPerWeek, 0),
        p.GravityScore = 
            (ISNULL(v.UnitsPerWeek, 0) * 0.4) + -- Velocity weight
            (p.UnitValue / 100.0 * 0.4) +        -- Value weight
            (p.FragilityIndex * 0.2),            -- Fragility weight
        p.OptimalZoneType = CASE
            WHEN ISNULL(v.UnitsPerWeek, 0) > 50 THEN 'High-Velocity'
            WHEN p.UnitValue > 500 THEN 'High-Value'
            WHEN p.FragilityIndex > 7 THEN 'Fragile'
            ELSE 'Standard'
        END,
        p.LastUpdated = GETDATE()
    FROM DimProductGravity p
    LEFT JOIN VelocityStats v ON p.ProductKey = v.ProductKey;
END;
```

## Advanced Patterns

### Temporal Elasticity Simulation

Simulate impact of warehouse capacity change:
```sql
CREATE PROCEDURE usp_SimulateCapacityChange
    @WarehouseKey INT,
    @CapacityChangePercent DECIMAL(5,2), -- e.g., 15.0 for +15%
    @SimulationDays INT = 30
AS
BEGIN
    -- Get baseline metrics
    DECLARE @BaselineAvgDwell DECIMAL(10,2);
    DECLARE @BaselineOperationsPerDay INT;
    
    SELECT 
        @BaselineAvgDwell = AVG(DwellTimeMinutes),
        @BaselineOperationsPerDay = COUNT(*) / 30
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE wo.WarehouseKey = @WarehouseKey
    AND t.Date >= DATEADD(DAY, -30, GETDATE());
    
    -- Simulate new state (simplified linear model)
    SELECT 
        'Baseline' AS Scenario,
        @BaselineAvgDwell AS AvgDwellTime,
        @BaselineOperationsPerDay AS DailyOperations,
        @BaselineOperationsPerDay * 30 AS MonthlyThroughput
    
    UNION ALL
    
    SELECT 
        'Simulated (+' + CAST(@CapacityChangePercent AS VARCHAR) + '%)' AS Scenario,
        @BaselineAvgDwell * (1 - (@CapacityChangePercent / 100 * 0.3)) AS AvgDwellTime,
        @BaselineOperationsPerDay * (1 + @CapacityChangePercent / 100) AS DailyOperations,
        @BaselineOperationsPerDay * (1 + @CapacityChangePercent / 100) * 30 AS MonthlyThroughput;
END;
```

### Multi-User Anomaly Tagging

```sql
CREATE TABLE AnomalyLog (
    AnomalyID BIGINT IDENTITY(1,1) PRIMARY KEY,
    FactTable NVARCHAR(100),
    FactRecordID BIGINT,
