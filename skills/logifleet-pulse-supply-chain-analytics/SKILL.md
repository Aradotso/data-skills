---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing and analytics platform for logistics, fleet management, and supply chain intelligence with multi-fact star schema modeling
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure Power BI logistics dashboard"
  - "implement multi-fact star schema for warehouse data"
  - "create fleet optimization analytics database"
  - "build supply chain intelligence dashboard"
  - "deploy LogiCore Analytics data warehouse"
  - "integrate warehouse and fleet telemetry data"
  - "query cross-fact logistics KPIs"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is an advanced data warehousing and analytics platform for logistics and supply chain management. It provides:

- **Multi-fact star schema** architecture linking warehouse operations, fleet telemetry, and inventory data
- **MS SQL Server** backend with time-phased dimensions (15-minute granularity)
- **Power BI dashboards** for real-time logistics intelligence
- **Cross-modal KPI harmonization** between warehouse, fleet, and supplier metrics
- **Predictive analytics** for bottleneck detection and maintenance prioritization

The platform integrates warehouse management systems (WMS), GPS/telematics feeds, supplier portals, weather APIs, and order history into a unified semantic layer.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to data sources: WMS, telemetry APIs, ERP systems

### Database Deployment

1. **Clone the repository:**

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy SQL schema:**

```sql
-- Connect to your SQL Server instance
-- Run the main schema deployment script
:r .\sql\schema\01_create_database.sql
:r .\sql\schema\02_create_dimensions.sql
:r .\sql\schema\03_create_facts.sql
:r .\sql\schema\04_create_views.sql
:r .\sql\schema\05_create_procedures.sql
```

3. **Configure data source connections:**

```json
// config.json (copy from config_sample.json)
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_USER}",
    "password": "${SQL_PASSWORD}"
  },
  "data_sources": {
    "wms_api": "${WMS_API_ENDPOINT}",
    "telematics_feed": "${TELEMATICS_URL}",
    "weather_api_key": "${WEATHER_API_KEY}"
  }
}
```

4. **Import Power BI template:**

- Open `LogiFleet_Pulse_Master.pbit`
- Enter SQL Server connection string when prompted
- Configure refresh schedule in Power BI Service (optional)

## Core Data Model

### Star Schema Architecture

The platform uses a **multi-fact star schema** with shared dimensions:

```sql
-- Core Fact Tables
FactWarehouseOperations (
    OperationID INT PRIMARY KEY,
    TimeID INT FOREIGN KEY REFERENCES DimTime,
    ProductID INT FOREIGN KEY REFERENCES DimProductGravity,
    WarehouseZoneID INT FOREIGN KEY REFERENCES DimGeography,
    OperationType VARCHAR(50), -- 'PUTAWAY', 'PICK', 'PACK', 'SHIP'
    DwellTimeMinutes INT,
    PickRate DECIMAL(10,2),
    PackingTimeSeconds INT,
    BatchID VARCHAR(50)
)

FactFleetTrips (
    TripID INT PRIMARY KEY,
    TimeID INT FOREIGN KEY REFERENCES DimTime,
    VehicleID INT FOREIGN KEY REFERENCES DimVehicle,
    RouteID INT FOREIGN KEY REFERENCES DimGeography,
    FuelConsumed DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    DistanceKM DECIMAL(10,2),
    AverageMPH DECIMAL(5,2)
)

FactCrossDock (
    CrossDockID INT PRIMARY KEY,
    TimeID INT FOREIGN KEY REFERENCES DimTime,
    InboundShipmentID VARCHAR(50),
    OutboundShipmentID VARCHAR(50),
    TransferTimeMinutes INT,
    ProductID INT FOREIGN KEY REFERENCES DimProductGravity
)

-- Shared Dimensions
DimTime (
    TimeID INT PRIMARY KEY,
    DateTime DATETIME,
    Hour INT,
    DayOfWeek VARCHAR(20),
    FiscalPeriod VARCHAR(20),
    Is15MinuteBucket BIT
)

DimProductGravity (
    ProductID INT PRIMARY KEY,
    SKU VARCHAR(50),
    ProductName VARCHAR(200),
    GravityScore DECIMAL(5,2), -- Velocity + Value + Fragility
    Category VARCHAR(100),
    ValueTier VARCHAR(20)
)

DimGeography (
    GeographyID INT PRIMARY KEY,
    LocationType VARCHAR(50), -- 'WAREHOUSE', 'ROUTE_NODE', 'ZONE'
    Continent VARCHAR(50),
    Country VARCHAR(100),
    Region VARCHAR(100),
    WarehouseName VARCHAR(200),
    ZoneName VARCHAR(100)
)
```

## Key Queries & Patterns

### Cross-Fact KPI Analysis

**Correlate warehouse dwell time with fleet delivery delays:**

```sql
-- Find products with high warehouse dwell time causing fleet bottlenecks
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    AVG(wo.DwellTimeMinutes) AS AvgWarehouseDwell,
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdle,
    COUNT(DISTINCT ft.TripID) AS AffectedTrips
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON wo.ProductID = p.ProductID
INNER JOIN FactFleetTrips ft ON wo.TimeID = ft.TimeID
    AND wo.BatchID = ft.LoadingBatchID -- Link via batch
WHERE wo.OperationType = 'PICK'
    AND wo.DwellTimeMinutes > 72 -- More than 3 days
    AND ft.IdleTimeMinutes > 30
GROUP BY p.SKU, p.ProductName, p.GravityScore
HAVING COUNT(DISTINCT ft.TripID) > 5
ORDER BY AvgWarehouseDwell DESC;
```

### Warehouse Gravity Zone Optimization

**Identify products that should be reclassified to different gravity zones:**

```sql
-- Products with velocity mismatch to their current zone assignment
WITH ProductVelocity AS (
    SELECT 
        ProductID,
        COUNT(*) AS PickCount,
        AVG(CAST(DwellTimeMinutes AS FLOAT)) AS AvgDwellTime
    FROM FactWarehouseOperations
    WHERE OperationType = 'PICK'
        AND TimeID >= DATEADD(MONTH, -3, GETDATE()) -- Last 3 months
    GROUP BY ProductID
)
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore AS CurrentGravity,
    pv.PickCount,
    pv.AvgDwellTime,
    CASE 
        WHEN pv.PickCount > 100 AND p.GravityScore < 7 THEN 'INCREASE_GRAVITY'
        WHEN pv.PickCount < 20 AND p.GravityScore > 5 THEN 'DECREASE_GRAVITY'
        ELSE 'OPTIMAL'
    END AS RecommendedAction,
    g.ZoneName AS CurrentZone
FROM ProductVelocity pv
INNER JOIN DimProductGravity p ON pv.ProductID = p.ProductID
INNER JOIN FactWarehouseOperations wo ON p.ProductID = wo.ProductID
INNER JOIN DimGeography g ON wo.WarehouseZoneID = g.GeographyID
WHERE g.LocationType = 'ZONE'
ORDER BY pv.PickCount DESC;
```

### Fleet Maintenance Triage

**Generate proactive maintenance queue ranked by revenue impact:**

```sql
-- Prioritize vehicle maintenance by combining telemetry anomalies with load value
WITH VehicleIssues AS (
    SELECT 
        VehicleID,
        AVG(FuelConsumed / NULLIF(DistanceKM, 0)) AS AvgFuelEfficiency,
        SUM(IdleTimeMinutes) AS TotalIdleTime,
        COUNT(*) AS TripCount
    FROM FactFleetTrips
    WHERE TimeID >= DATEADD(DAY, -7, GETDATE())
    GROUP BY VehicleID
),
LoadValue AS (
    SELECT 
        ft.VehicleID,
        SUM(p.GravityScore * wo.PickRate) AS TotalLoadValue
    FROM FactFleetTrips ft
    INNER JOIN FactWarehouseOperations wo ON ft.LoadingBatchID = wo.BatchID
    INNER JOIN DimProductGravity p ON wo.ProductID = p.ProductID
    WHERE ft.TimeID >= DATEADD(DAY, -7, GETDATE())
    GROUP BY ft.VehicleID
)
SELECT 
    v.VehicleID,
    v.LicensePlate,
    vi.AvgFuelEfficiency,
    vi.TotalIdleTime,
    lv.TotalLoadValue,
    -- Priority score: higher for inefficient vehicles carrying high-value loads
    (vi.AvgFuelEfficiency * 0.4 + (vi.TotalIdleTime / 60.0) * 0.3 + (lv.TotalLoadValue / 1000.0) * 0.3) AS PriorityScore
FROM VehicleIssues vi
INNER JOIN DimVehicle v ON vi.VehicleID = v.VehicleID
LEFT JOIN LoadValue lv ON vi.VehicleID = lv.VehicleID
WHERE vi.AvgFuelEfficiency > 0.15 -- Threshold for concern
    OR vi.TotalIdleTime > 120 -- More than 2 hours idle
ORDER BY PriorityScore DESC;
```

### Temporal Elasticity Simulation

**Simulate impact of warehouse capacity changes on fleet utilization:**

```sql
-- Stored procedure for scenario analysis
CREATE PROCEDURE sp_SimulateCapacityImpact
    @NewCapacityPercent DECIMAL(5,2),
    @SimulationDays INT = 30
AS
BEGIN
    -- Create temporary table with simulated data
    SELECT 
        ft.TripID,
        ft.VehicleID,
        ft.DistanceKM,
        ft.IdleTimeMinutes,
        wo.DwellTimeMinutes,
        -- Simulate: higher capacity reduces dwell time, increases fleet efficiency
        wo.DwellTimeMinutes * (100.0 / @NewCapacityPercent) AS SimulatedDwellTime,
        ft.IdleTimeMinutes * (100.0 / @NewCapacityPercent) AS SimulatedIdleTime
    INTO #SimulationResults
    FROM FactFleetTrips ft
    INNER JOIN FactWarehouseOperations wo ON ft.LoadingBatchID = wo.BatchID
    WHERE ft.TimeID >= DATEADD(DAY, -@SimulationDays, GETDATE());
    
    -- Compare baseline vs simulated
    SELECT 
        'BASELINE' AS Scenario,
        AVG(IdleTimeMinutes) AS AvgIdleTime,
        AVG(DwellTimeMinutes) AS AvgDwellTime,
        SUM(DistanceKM) AS TotalDistance
    FROM #SimulationResults
    UNION ALL
    SELECT 
        'SIMULATED_' + CAST(@NewCapacityPercent AS VARCHAR) + '%' AS Scenario,
        AVG(SimulatedIdleTime) AS AvgIdleTime,
        AVG(SimulatedDwellTime) AS AvgDwellTime,
        SUM(DistanceKM) AS TotalDistance
    FROM #SimulationResults;
    
    DROP TABLE #SimulationResults;
END;

-- Execute simulation
EXEC sp_SimulateCapacityImpact @NewCapacityPercent = 95.0, @SimulationDays = 30;
```

## Data Loading Procedures

### Incremental ETL Pattern

```sql
-- Incremental load for warehouse operations
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadTimestamp DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Default to last successful load
    IF @LastLoadTimestamp IS NULL
        SELECT @LastLoadTimestamp = MAX(LoadTimestamp) FROM ETL_LoadLog WHERE TableName = 'FactWarehouseOperations';
    
    -- Insert new records from staging
    INSERT INTO FactWarehouseOperations (
        OperationID, TimeID, ProductID, WarehouseZoneID, 
        OperationType, DwellTimeMinutes, PickRate, PackingTimeSeconds, BatchID
    )
    SELECT 
        s.OperationID,
        t.TimeID,
        p.ProductID,
        g.GeographyID,
        s.OperationType,
        s.DwellTimeMinutes,
        s.PickRate,
        s.PackingTimeSeconds,
        s.BatchID
    FROM StagingWarehouseOps s
    INNER JOIN DimTime t ON CAST(s.OperationTimestamp AS DATE) = CAST(t.DateTime AS DATE)
        AND DATEPART(HOUR, s.OperationTimestamp) = t.Hour
    INNER JOIN DimProductGravity p ON s.SKU = p.SKU
    INNER JOIN DimGeography g ON s.ZoneName = g.ZoneName
    WHERE s.OperationTimestamp > @LastLoadTimestamp
        AND NOT EXISTS (
            SELECT 1 FROM FactWarehouseOperations 
            WHERE OperationID = s.OperationID
        );
    
    -- Log the load
    INSERT INTO ETL_LoadLog (TableName, LoadTimestamp, RowsInserted)
    VALUES ('FactWarehouseOperations', GETDATE(), @@ROWCOUNT);
END;
```

## Power BI Integration

### DAX Measures for Cross-Fact Analysis

```dax
// Measure: Warehouse-Fleet Efficiency Score
WarehouseFleetEfficiency = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR AvgIdle = AVERAGE(FactFleetTrips[IdleTimeMinutes])
VAR NormalizedDwell = DIVIDE(AvgDwell, 72) // 72 hours baseline
VAR NormalizedIdle = DIVIDE(AvgIdle, 30) // 30 minutes baseline
RETURN 
    1 - ((NormalizedDwell * 0.5) + (NormalizedIdle * 0.5)) // Higher = better

// Measure: Bottleneck Risk Index
BottleneckRisk = 
CALCULATE(
    COUNTROWS(FactWarehouseOperations),
    FactWarehouseOperations[DwellTimeMinutes] > 72
) * 
CALCULATE(
    COUNTROWS(FactFleetTrips),
    FactFleetTrips[IdleTimeMinutes] > 30
) / 100.0

// Calculated Column: Gravity Zone Assignment
GravityZone = 
SWITCH(
    TRUE(),
    [GravityScore] >= 8, "HIGH_VELOCITY",
    [GravityScore] >= 5, "MEDIUM_VELOCITY",
    "LOW_VELOCITY"
)
```

### Power BI Report Configuration

```json
// Dataset refresh settings (via Power BI REST API or Service)
{
  "refreshSchedule": {
    "enabled": true,
    "frequency": "Hourly",
    "times": ["00:00", "01:00", "02:00"], // Every hour
    "localTimeZoneId": "UTC"
  },
  "incremental_refresh": {
    "enabled": true,
    "archive_years": 2,
    "rolling_window_days": 90
  }
}
```

## Alerting & Automation

### Threshold-Based Alerts

```sql
-- Stored procedure for automated alerting
CREATE PROCEDURE sp_CheckKPIThresholds
AS
BEGIN
    -- Fleet idle time breach
    INSERT INTO Alerts (AlertType, Severity, Message, CreatedAt)
    SELECT 
        'FLEET_IDLE_THRESHOLD',
        'WARNING',
        'Vehicle ' + CAST(VehicleID AS VARCHAR) + ' exceeded 15% idle time threshold',
        GETDATE()
    FROM FactFleetTrips
    WHERE TimeID >= DATEADD(HOUR, -1, GETDATE())
        AND (CAST(IdleTimeMinutes AS FLOAT) / NULLIF(LoadingTimeMinutes + IdleTimeMinutes, 0)) > 0.15
    GROUP BY VehicleID
    HAVING COUNT(*) > 3;
    
    -- High dwell time alert
    INSERT INTO Alerts (AlertType, Severity, Message, CreatedAt)
    SELECT 
        'DWELL_TIME_CRITICAL',
        'CRITICAL',
        'SKU ' + p.SKU + ' in zone ' + g.ZoneName + ' has ' + CAST(AVG(wo.DwellTimeMinutes) AS VARCHAR) + ' min dwell time',
        GETDATE()
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductID = p.ProductID
    INNER JOIN DimGeography g ON wo.WarehouseZoneID = g.GeographyID
    WHERE wo.TimeID >= DATEADD(HOUR, -4, GETDATE())
        AND wo.DwellTimeMinutes > 72
    GROUP BY p.SKU, g.ZoneName
    HAVING AVG(wo.DwellTimeMinutes) > 96; -- 4 days
END;

-- Schedule via SQL Agent Job (run every 15 minutes)
```

## Configuration Best Practices

### Indexing Strategy

```sql
-- Clustered indexes on fact tables (time-based)
CREATE CLUSTERED INDEX IX_FactWarehouse_Time 
ON FactWarehouseOperations(TimeID, ProductID);

CREATE CLUSTERED INDEX IX_FactFleet_Time 
ON FactFleetTrips(TimeID, VehicleID);

-- Non-clustered indexes for common filters
CREATE NONCLUSTERED INDEX IX_FactWarehouse_DwellTime 
ON FactWarehouseOperations(DwellTimeMinutes)
INCLUDE (ProductID, WarehouseZoneID);

CREATE NONCLUSTERED INDEX IX_FactFleet_IdleTime 
ON FactFleetTrips(IdleTimeMinutes)
INCLUDE (VehicleID, RouteID);

-- Columnstore for analytical queries
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouse
ON FactWarehouseOperations (TimeID, ProductID, DwellTimeMinutes, PickRate);
```

### Partitioning for Large Datasets

```sql
-- Partition fact tables by month for better performance
CREATE PARTITION FUNCTION PF_ByMonth (DATETIME)
AS RANGE RIGHT FOR VALUES (
    '2026-01-01', '2026-02-01', '2026-03-01', 
    '2026-04-01', '2026-05-01', '2026-06-01'
);

CREATE PARTITION SCHEME PS_ByMonth
AS PARTITION PF_ByMonth ALL TO ([PRIMARY]);

-- Apply to fact table
CREATE TABLE FactWarehouseOperations (
    OperationID INT,
    TimeID INT,
    OperationTimestamp DATETIME,
    -- ... other columns
) ON PS_ByMonth(OperationTimestamp);
```

## Troubleshooting

### Common Issues

**Power BI refresh failures:**

```sql
-- Check for orphaned records in fact tables
SELECT 'Orphaned Products' AS Issue, COUNT(*) AS Count
FROM FactWarehouseOperations wo
LEFT JOIN DimProductGravity p ON wo.ProductID = p.ProductID
WHERE p.ProductID IS NULL

UNION ALL

SELECT 'Orphaned TimeIDs', COUNT(*)
FROM FactFleetTrips ft
LEFT JOIN DimTime t ON ft.TimeID = t.TimeID
WHERE t.TimeID IS NULL;
```

**Slow cross-fact queries:**

```sql
-- Verify statistics are up-to-date
UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS FactFleetTrips WITH FULLSCAN;

-- Check for missing indexes
SELECT 
    migs.avg_total_user_cost * (migs.avg_user_impact / 100.0) * (migs.user_seeks + migs.user_scans) AS improvement_measure,
    mid.statement AS table_name,
    mid.equality_columns,
    mid.inequality_columns,
    mid.included_columns
FROM sys.dm_db_missing_index_group_stats AS migs
INNER JOIN sys.dm_db_missing_index_groups AS mig ON migs.group_handle = mig.index_group_handle
INNER JOIN sys.dm_db_missing_index_details AS mid ON mig.index_handle = mid.index_handle
WHERE migs.avg_total_user_cost * (migs.avg_user_impact / 100.0) * (migs.user_seeks + migs.user_scans) > 1000
ORDER BY improvement_measure DESC;
```

**Data quality issues:**

```sql
-- Run data validation checks
EXEC sp_ValidateDataQuality;

-- Custom validation procedure
CREATE PROCEDURE sp_ValidateDataQuality
AS
BEGIN
    -- Check for negative values
    SELECT 'Negative dwell time' AS Issue, COUNT(*) FROM FactWarehouseOperations WHERE DwellTimeMinutes < 0;
    
    -- Check for missing dimensions
    SELECT 'Missing product gravity scores' AS Issue, COUNT(*) FROM DimProductGravity WHERE GravityScore IS NULL;
    
    -- Check for duplicate records
    SELECT 'Duplicate operations' AS Issue, OperationID, COUNT(*) 
    FROM FactWarehouseOperations 
    GROUP BY OperationID 
    HAVING COUNT(*) > 1;
END;
```

## Advanced Patterns

### Many-to-Many Bridge Tables

```sql
-- Bridge table for routes accessing multiple warehouse zones
CREATE TABLE BridgeRouteZone (
    RouteID INT FOREIGN KEY REFERENCES DimGeography(GeographyID),
    ZoneID INT FOREIGN KEY REFERENCES DimGeography(GeographyID),
    AccessFrequency INT,
    AverageAccessTimeMinutes DECIMAL(10,2)
);

-- Query using bridge table
SELECT 
    r.RouteName,
    z.ZoneName,
    brz.AccessFrequency,
    AVG(ft.IdleTimeMinutes) AS AvgIdleTime
FROM FactFleetTrips ft
INNER JOIN BridgeRouteZone brz ON ft.RouteID = brz.RouteID
INNER JOIN DimGeography z ON brz.ZoneID = z.GeographyID
INNER JOIN DimGeography r ON ft.RouteID = r.GeographyID
WHERE z.LocationType = 'ZONE'
    AND r.LocationType = 'ROUTE_NODE'
GROUP BY r.RouteName, z.ZoneName, brz.AccessFrequency;
```

### Role-Playing Dimensions

```sql
-- DimTime used multiple times with different roles
SELECT 
    t1.DateTime AS PickupTime,
    t2.DateTime AS DeliveryTime,
    DATEDIFF(MINUTE, t1.DateTime, t2.DateTime) AS TotalTripMinutes,
    ft.DistanceKM
FROM FactFleetTrips ft
INNER JOIN DimTime t1 ON ft.PickupTimeID = t1.TimeID -- Pickup role
INNER JOIN DimTime t2 ON ft.DeliveryTimeID = t2.TimeID -- Delivery role
WHERE t1.DateTime >= '2026-01-01';
```

This skill provides comprehensive guidance for deploying and using LogiFleet Pulse for supply chain analytics, covering database setup, query patterns, Power BI integration, and operational best practices.
