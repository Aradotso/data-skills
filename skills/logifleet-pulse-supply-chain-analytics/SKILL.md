---
name: logifleet-pulse-supply-chain-analytics
description: Multi-modal logistics intelligence platform using MS SQL Server star schema and Power BI for warehouse, fleet, and supply chain KPI harmonization
triggers:
  - "set up logistics data warehouse with SQL Server"
  - "create supply chain analytics dashboard in Power BI"
  - "implement multi-fact star schema for warehouse and fleet data"
  - "build logistics KPI tracking system"
  - "design fleet optimization and warehouse analytics"
  - "configure cross-modal supply chain intelligence"
  - "integrate warehouse management with fleet telemetry"
  - "deploy logistics business intelligence solution"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive logistics intelligence platform that unifies warehouse operations, fleet management, and supply chain analytics into a single semantic data model. Built on MS SQL Server with Power BI visualization, it implements a multi-fact star schema architecture that enables cross-modal KPI analysis (e.g., correlating warehouse dwell time with fleet fuel consumption).

**Key capabilities:**
- Multi-fact data warehouse with time-phased dimensions
- Real-time integration of warehouse, fleet, and external data sources
- Predictive bottleneck detection and maintenance triage
- Warehouse Gravity Zones™ spatial optimization
- Role-based dashboards with natural language querying

## Installation

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- SSMS (SQL Server Management Studio) for schema deployment
- Source system access: WMS, TMS, telematics APIs

### Database Schema Setup

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**
```sql
-- Connect to your SQL Server instance via SSMS
-- Execute the schema creation script
:r schema/01_create_database.sql
:r schema/02_create_dimensions.sql
:r schema/03_create_facts.sql
:r schema/04_create_views.sql
:r schema/05_create_stored_procedures.sql
```

3. **Configure data sources:**
```json
// Update config_sample.json with your connection strings
{
  "sql_server": {
    "host": "your-server.database.windows.net",
    "database": "LogiFleetPulse",
    "auth": "integrated"
  },
  "data_sources": {
    "wms_connection": "${WMS_CONNECTION_STRING}",
    "telemetry_api": "${FLEET_TELEMETRY_API_URL}",
    "weather_api": "${WEATHER_API_KEY}"
  }
}
```

4. **Import Power BI template:**
```powershell
# Open the .pbit template
Start-Process "LogiFleet_Pulse_Master.pbit"
# When prompted, enter your SQL Server connection details
```

## Core Data Model Architecture

### Dimension Tables

**DimTime** - 15-minute granularity time tracking:
```sql
CREATE TABLE dbo.DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2(0) NOT NULL,
    HourOfDay TINYINT,
    DayOfWeek TINYINT,
    IsBusinessHour BIT,
    FiscalQuarter TINYINT,
    FiscalYear SMALLINT
);

-- Example: Query operations during business hours
SELECT 
    f.OperationID,
    t.FullDateTime,
    f.DwellTimeMinutes
FROM FactWarehouseOperations f
JOIN DimTime t ON f.TimeKey = t.TimeKey
WHERE t.IsBusinessHour = 1
    AND t.FiscalQuarter = 2;
```

**DimProductGravity** - Velocity-based warehouse positioning:
```sql
CREATE TABLE dbo.DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU NVARCHAR(50) NOT NULL,
    ProductName NVARCHAR(200),
    GravityScore DECIMAL(5,2), -- Calculated: velocity × value × (1/fragility)
    VelocityCategory NVARCHAR(20), -- Fast/Medium/Slow
    OptimalZone NVARCHAR(10), -- A1, B2, etc.
    LastRecalcDate DATE
);

-- Example: Identify misplaced high-gravity items
SELECT 
    p.SKU,
    p.GravityScore,
    p.OptimalZone AS RecommendedZone,
    w.CurrentZone AS ActualZone,
    w.PickFrequency
FROM DimProductGravity p
JOIN FactWarehouseOperations w ON p.ProductKey = w.ProductKey
WHERE p.OptimalZone != w.CurrentZone
    AND p.GravityScore > 75
ORDER BY p.GravityScore DESC;
```

### Fact Tables

**FactWarehouseOperations** - Micro-operations tracking:
```sql
CREATE TABLE dbo.FactWarehouseOperations (
    OperationID BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    OperationType NVARCHAR(20), -- Putaway, Pick, Pack, Ship
    DwellTimeMinutes INT,
    PickCycleSeconds INT,
    CurrentZone NVARCHAR(10),
    OperatorID INT,
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
);

-- Example: Calculate average dwell time by product category
SELECT 
    p.ProductName,
    AVG(w.DwellTimeMinutes) AS AvgDwellMinutes,
    COUNT(*) AS OperationCount
FROM FactWarehouseOperations w
JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
WHERE w.OperationType = 'Putaway'
    AND w.TimeKey >= 20260701000 -- July 1, 2026
GROUP BY p.ProductName
HAVING AVG(w.DwellTimeMinutes) > 72 * 60 -- More than 72 hours
ORDER BY AvgDwellMinutes DESC;
```

**FactFleetTrips** - Vehicle telemetry and route segments:
```sql
CREATE TABLE dbo.FactFleetTrips (
    TripID BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT NOT NULL,
    FuelConsumedLiters DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadWeightKg INT,
    DistanceKm DECIMAL(10,2),
    TirePressureAvg DECIMAL(4,1),
    EngineHealthScore TINYINT, -- 0-100
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey)
);

-- Example: Identify high-idle routes
SELECT 
    r.RouteName,
    AVG(f.IdleTimeMinutes) AS AvgIdleMinutes,
    AVG(f.FuelConsumedLiters) AS AvgFuelLiters,
    COUNT(DISTINCT f.VehicleKey) AS VehicleCount
FROM FactFleetTrips f
JOIN DimRoute r ON f.RouteKey = r.RouteKey
GROUP BY r.RouteName
HAVING AVG(f.IdleTimeMinutes) > AVG(f.DistanceKm) * 0.15 -- Idle > 15% of trip time proxy
ORDER BY AvgIdleMinutes DESC;
```

## Key Stored Procedures

### Incremental Data Loading

**Load warehouse operations from WMS:**
```sql
CREATE PROCEDURE dbo.usp_LoadWarehouseOperations
    @LoadDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Truncate staging table
    TRUNCATE TABLE staging.WarehouseOperations;
    
    -- Load from external WMS source (via linked server or bulk insert)
    INSERT INTO staging.WarehouseOperations
    SELECT * FROM OPENROWSET(
        'SQLNCLI',
        'Server=${WMS_SERVER};Trusted_Connection=yes;',
        'SELECT * FROM WMS.dbo.Operations WHERE OperationDate = ''' + CAST(@LoadDate AS VARCHAR(10)) + ''''
    );
    
    -- Merge into fact table with deduplication
    MERGE dbo.FactWarehouseOperations AS target
    USING (
        SELECT 
            t.TimeKey,
            p.ProductKey,
            w.WarehouseKey,
            s.OperationType,
            DATEDIFF(MINUTE, s.StartTime, s.EndTime) AS DwellTimeMinutes,
            s.PickCycleSeconds,
            s.ZoneCode AS CurrentZone,
            s.OperatorID
        FROM staging.WarehouseOperations s
        JOIN DimTime t ON s.OperationDateTime = t.FullDateTime
        JOIN DimProductGravity p ON s.SKU = p.SKU
        JOIN DimWarehouse w ON s.WarehouseCode = w.WarehouseCode
    ) AS source
    ON target.OperationID = source.OperationID
    WHEN NOT MATCHED THEN
        INSERT (TimeKey, ProductKey, WarehouseKey, OperationType, DwellTimeMinutes, PickCycleSeconds, CurrentZone, OperatorID)
        VALUES (source.TimeKey, source.ProductKey, source.WarehouseKey, source.OperationType, source.DwellTimeMinutes, source.PickCycleSeconds, source.CurrentZone, source.OperatorID);
    
    -- Log completion
    INSERT INTO dbo.ETL_Log (ProcedureName, LoadDate, RowsAffected)
    VALUES ('usp_LoadWarehouseOperations', @LoadDate, @@ROWCOUNT);
END;
GO

-- Execute daily load
EXEC dbo.usp_LoadWarehouseOperations @LoadDate = '2026-07-10';
```

### Cross-Fact KPI Calculation

**Correlate warehouse dwell with fleet idle:**
```sql
CREATE PROCEDURE dbo.usp_CalculateCrossDockEfficiency
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    WITH WarehouseDwell AS (
        SELECT 
            w.ProductKey,
            w.WarehouseKey,
            AVG(w.DwellTimeMinutes) AS AvgDwellMinutes
        FROM FactWarehouseOperations w
        JOIN DimTime t ON w.TimeKey = t.TimeKey
        WHERE t.FullDateTime BETWEEN @StartDate AND @EndDate
            AND w.OperationType = 'CrossDock'
        GROUP BY w.ProductKey, w.WarehouseKey
    ),
    FleetIdle AS (
        SELECT 
            f.RouteKey,
            AVG(f.IdleTimeMinutes) AS AvgIdleMinutes,
            AVG(f.FuelConsumedLiters) AS AvgFuelLiters
        FROM FactFleetTrips f
        JOIN DimTime t ON f.TimeKey = t.TimeKey
        WHERE t.FullDateTime BETWEEN @StartDate AND @EndDate
        GROUP BY f.RouteKey
    )
    SELECT 
        p.ProductName,
        w.WarehouseKey,
        wd.AvgDwellMinutes,
        fi.AvgIdleMinutes,
        fi.AvgFuelLiters,
        -- Composite efficiency score (lower is better)
        (wd.AvgDwellMinutes * 0.6 + fi.AvgIdleMinutes * 0.4) AS CompositeScore
    FROM WarehouseDwell wd
    JOIN FleetIdle fi ON wd.WarehouseKey = fi.RouteKey -- Assumes route starts at warehouse
    JOIN DimProductGravity p ON wd.ProductKey = p.ProductKey
    ORDER BY CompositeScore DESC;
END;
GO

-- Execute analysis
EXEC dbo.usp_CalculateCrossDockEfficiency 
    @StartDate = '2026-07-01', 
    @EndDate = '2026-07-10';
```

## Power BI Integration

### DAX Measures for Cross-Fact Analysis

**Warehouse-Fleet Correlation Metric:**
```dax
// Create in Power BI Desktop
Fleet_Idle_Per_Dwell_Hour = 
VAR TotalDwellHours = SUM(FactWarehouseOperations[DwellTimeMinutes]) / 60
VAR TotalIdleMinutes = SUM(FactFleetTrips[IdleTimeMinutes])
RETURN
    DIVIDE(TotalIdleMinutes, TotalDwellHours, 0)
```

**Predictive Bottleneck Index:**
```dax
Bottleneck_Risk_Score = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR AvgIdle = AVERAGE(FactFleetTrips[IdleTimeMinutes])
VAR DwellThreshold = 4320 // 72 hours in minutes
VAR IdleThreshold = 45 // 15% of 5-hour trip
RETURN
    SWITCH(
        TRUE(),
        AvgDwell > DwellThreshold && AvgIdle > IdleThreshold, 100, // Critical
        AvgDwell > DwellThreshold || AvgIdle > IdleThreshold, 60,  // Warning
        30 // Normal
    )
```

### Report Configuration

**Connect Power BI to SQL Server:**
```powerquery
// M Query in Power BI Desktop
let
    Source = Sql.Database(
        "${SQL_SERVER_HOST}", 
        "LogiFleetPulse",
        [
            Query="SELECT * FROM dbo.vw_UnifiedLogisticsMetrics"
        ]
    ),
    FilteredRows = Table.SelectRows(
        Source, 
        each [FullDateTime] >= #datetime(2026, 7, 1, 0, 0, 0)
    )
in
    FilteredRows
```

## Automated Alerting

### Threshold-Based Notifications

**SQL Server Agent job for alert triggers:**
```sql
CREATE PROCEDURE dbo.usp_CheckThresholdsAndAlert
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    -- Check for excessive fleet idling
    IF EXISTS (
        SELECT 1 
        FROM FactFleetTrips f
        JOIN DimTime t ON f.TimeKey = t.TimeKey
        WHERE t.FullDateTime >= DATEADD(HOUR, -4, GETDATE())
        GROUP BY f.VehicleKey
        HAVING AVG(f.IdleTimeMinutes) > 60
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Fleet idle time exceeded 60 minutes average in last 4 hours. Check fleet dashboard.';
        
        -- Send email via Database Mail
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL}',
            @subject = 'LogiFleet Pulse: Fleet Idle Alert',
            @body = @AlertMessage;
    END;
    
    -- Check for warehouse dwell anomalies
    IF EXISTS (
        SELECT 1
        FROM FactWarehouseOperations w
        WHERE w.DwellTimeMinutes > 5760 -- 4 days
            AND w.TimeKey >= (SELECT MAX(TimeKey) FROM DimTime WHERE FullDateTime >= DATEADD(DAY, -1, GETDATE()))
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Products with dwell time >4 days detected. Review warehouse gravity zones.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL}',
            @subject = 'LogiFleet Pulse: Warehouse Dwell Alert',
            @body = @AlertMessage;
    END;
END;
GO

-- Schedule via SQL Server Agent
-- Job runs every 15 minutes
```

## Common Patterns

### Pattern 1: Gravity Zone Optimization

**Recalculate product gravity scores monthly:**
```sql
-- Stored procedure to update gravity scores
CREATE PROCEDURE dbo.usp_RecalculateGravityScores
AS
BEGIN
    UPDATE p
    SET 
        p.GravityScore = (
            (velocity.PicksPerDay * 10) +           -- Velocity weight
            (value.AvgValueUSD / 100) +             -- Value weight
            (100.0 / NULLIF(p.FragilityIndex, 0))   -- Inverse fragility
        ),
        p.VelocityCategory = CASE
            WHEN velocity.PicksPerDay >= 50 THEN 'Fast'
            WHEN velocity.PicksPerDay >= 10 THEN 'Medium'
            ELSE 'Slow'
        END,
        p.OptimalZone = CASE
            WHEN velocity.PicksPerDay >= 50 THEN 'A1'
            WHEN velocity.PicksPerDay >= 10 THEN 'B2'
            ELSE 'C3'
        END,
        p.LastRecalcDate = GETDATE()
    FROM DimProductGravity p
    CROSS APPLY (
        SELECT AVG(CAST(w.PickCycleSeconds AS FLOAT) / 86400) AS PicksPerDay
        FROM FactWarehouseOperations w
        WHERE w.ProductKey = p.ProductKey
            AND w.TimeKey >= (SELECT TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(MONTH, -1, GETDATE()))
    ) velocity
    CROSS APPLY (
        SELECT AVG(UnitPrice * OrderQuantity) AS AvgValueUSD
        FROM FactOrders o
        WHERE o.ProductKey = p.ProductKey
    ) value;
END;
GO
```

### Pattern 2: Predictive Maintenance Scoring

**Fleet vehicle health prediction:**
```sql
CREATE VIEW dbo.vw_FleetMaintenancePriority
AS
WITH VehicleHealth AS (
    SELECT 
        f.VehicleKey,
        v.VehiclePlate,
        AVG(f.EngineHealthScore) AS AvgHealthScore,
        AVG(f.TirePressureAvg) AS AvgTirePressure,
        SUM(f.IdleTimeMinutes) AS TotalIdleMinutes,
        MAX(m.LastServiceDate) AS LastServiceDate
    FROM FactFleetTrips f
    JOIN DimVehicle v ON f.VehicleKey = v.VehicleKey
    LEFT JOIN DimMaintenance m ON v.VehicleKey = m.VehicleKey
    WHERE f.TimeKey >= (SELECT TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(DAY, -30, GETDATE()))
    GROUP BY f.VehicleKey, v.VehiclePlate
),
LoadProfile AS (
    SELECT 
        f.VehicleKey,
        AVG(f.LoadWeightKg) AS AvgLoadWeight,
        MAX(o.MarginPercent) AS MaxMargin
    FROM FactFleetTrips f
    JOIN FactOrders o ON f.RouteKey = o.RouteKey
    GROUP BY f.VehicleKey
)
SELECT 
    vh.VehicleKey,
    vh.VehiclePlate,
    vh.AvgHealthScore,
    vh.AvgTirePressure,
    lp.AvgLoadWeight,
    lp.MaxMargin,
    -- Composite urgency score (0-100, higher = more urgent)
    (
        ((100 - vh.AvgHealthScore) * 0.4) +                    -- Health degradation
        ((100 - (vh.AvgTirePressure / 35 * 100)) * 0.2) +     -- Tire pressure drop
        (DATEDIFF(DAY, vh.LastServiceDate, GETDATE()) * 0.2) + -- Days since service
        (lp.MaxMargin * 0.2)                                   -- Revenue at risk
    ) AS MaintenanceUrgencyScore
FROM VehicleHealth vh
JOIN LoadProfile lp ON vh.VehicleKey = lp.VehicleKey
WHERE vh.AvgHealthScore < 80 OR vh.AvgTirePressure < 30;
GO

-- Query top 10 priority vehicles
SELECT TOP 10 * 
FROM dbo.vw_FleetMaintenancePriority
ORDER BY MaintenanceUrgencyScore DESC;
```

### Pattern 3: Temporal Elasticity Simulation

**Scenario modeling for capacity changes:**
```sql
CREATE PROCEDURE dbo.usp_SimulateCapacityChange
    @CapacityIncreasePercent DECIMAL(5,2),
    @SimulationDays INT = 30
AS
BEGIN
    -- Create temp table for scenario results
    CREATE TABLE #ScenarioResults (
        ScenarioName NVARCHAR(50),
        AvgDwellMinutes DECIMAL(10,2),
        AvgFleetUtilization DECIMAL(5,2),
        ProjectedCostSavings DECIMAL(12,2)
    );
    
    -- Baseline scenario
    INSERT INTO #ScenarioResults
    SELECT 
        'Baseline',
        AVG(w.DwellTimeMinutes),
        AVG(CAST(f.IdleTimeMinutes AS FLOAT) / NULLIF(f.DistanceKm * 1.5, 0)) * 100, -- Proxy utilization
        0
    FROM FactWarehouseOperations w
    JOIN FactFleetTrips f ON w.TimeKey = f.TimeKey
    WHERE w.TimeKey >= (SELECT TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(DAY, -@SimulationDays, GETDATE()));
    
    -- Increased capacity scenario (assumes linear reduction in dwell)
    INSERT INTO #ScenarioResults
    SELECT 
        'Capacity +' + CAST(@CapacityIncreasePercent AS NVARCHAR(10)) + '%',
        AVG(w.DwellTimeMinutes) * (1 - (@CapacityIncreasePercent / 100)),
        AVG(CAST(f.IdleTimeMinutes AS FLOAT) / NULLIF(f.DistanceKm * 1.5, 0)) * 100 * 0.95, -- 5% improvement
        -- Cost savings: reduced dwell = faster turnover = reduced holding cost
        (AVG(w.DwellTimeMinutes) * (@CapacityIncreasePercent / 100)) * 0.05 * COUNT(*) -- $0.05/minute holding cost
    FROM FactWarehouseOperations w
    JOIN FactFleetTrips f ON w.TimeKey = f.TimeKey
    WHERE w.TimeKey >= (SELECT TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(DAY, -@SimulationDays, GETDATE()));
    
    -- Return comparison
    SELECT * FROM #ScenarioResults;
    
    DROP TABLE #ScenarioResults;
END;
GO

-- Run simulation: 15% capacity increase
EXEC dbo.usp_SimulateCapacityChange @CapacityIncreasePercent = 15.0;
```

## Configuration

### Row-Level Security Setup

**Restrict data access by user role:**
```sql
-- Create security policy
CREATE SCHEMA security;
GO

CREATE FUNCTION security.fn_UserAccessPredicate(@WarehouseKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN (
    SELECT 1 AS AccessGranted
    WHERE 
        -- Executives see all warehouses
        IS_ROLEMEMBER('Executives') = 1
        OR
        -- Warehouse managers see only their warehouse
        @WarehouseKey IN (
            SELECT WarehouseKey 
            FROM dbo.UserWarehouseAccess 
            WHERE UserName = USER_NAME()
        )
);
GO

-- Apply policy to fact table
CREATE SECURITY POLICY security.WarehouseAccessPolicy
ADD FILTER PREDICATE security.fn_UserAccessPredicate(WarehouseKey)
ON dbo.FactWarehouseOperations
WITH (STATE = ON);
GO
```

### External Data Source Configuration

**Link weather API for delay correlation:**
```sql
-- Create external data source (requires PolyBase or Azure Synapse)
CREATE EXTERNAL DATA SOURCE WeatherAPI
WITH (
    TYPE = BLOB_STORAGE,
    LOCATION = 'https://${WEATHER_API_ENDPOINT}',
    CREDENTIAL = WeatherAPICredential
);
GO

-- Create external table
CREATE EXTERNAL TABLE ext.WeatherData (
    LocationKey INT,
    ForecastTime DATETIME2,
    Temperature DECIMAL(5,2),
    PrecipitationMM DECIMAL(5,2),
    WindSpeedKmh DECIMAL(5,2)
)
WITH (
    DATA_SOURCE = WeatherAPI,
    FILE_FORMAT = JSON_Format
);
GO

-- Join with fleet data to identify weather delays
SELECT 
    f.TripID,
    f.RouteKey,
    r.RouteName,
    w.PrecipitationMM,
    w.WindSpeedKmh,
    f.IdleTimeMinutes,
    CASE 
        WHEN w.PrecipitationMM > 10 OR w.WindSpeedKmh > 50 THEN 'Weather Delay'
        ELSE 'Normal'
    END AS DelayCategory
FROM FactFleetTrips f
JOIN DimRoute r ON f.RouteKey = r.RouteKey
JOIN ext.WeatherData w ON r.LocationKey = w.LocationKey
    AND ABS(DATEDIFF(MINUTE, f.TripStartTime, w.ForecastTime)) < 30;
```

## Troubleshooting

### Issue: Power BI Refresh Failures

**Diagnosis:**
```sql
-- Check for orphaned records in fact tables
SELECT 
    'FactWarehouseOperations' AS TableName,
    COUNT(*) AS OrphanedRecords
FROM dbo.FactWarehouseOperations f
LEFT JOIN DimTime t ON f.TimeKey = t.TimeKey
WHERE t.TimeKey IS NULL

UNION ALL

SELECT 
    'FactFleetTrips',
    COUNT(*)
FROM dbo.FactFleetTrips f
LEFT JOIN DimTime t ON f.TimeKey = t.TimeKey
WHERE t.TimeKey IS NULL;
```

**Solution:**
```sql
-- Clean up orphaned records
DELETE FROM dbo.FactWarehouseOperations
WHERE TimeKey NOT IN (SELECT TimeKey FROM DimTime);

DELETE FROM dbo.FactFleetTrips
WHERE TimeKey NOT IN (SELECT TimeKey FROM DimTime);

-- Recreate missing dimension entries
INSERT INTO DimTime (TimeKey, FullDateTime, HourOfDay, DayOfWeek, IsBusinessHour)
SELECT DISTINCT
    CAST(FORMAT(OperationDateTime, 'yyyyMMddHHmm') AS INT),
    OperationDateTime,
    DATEPART(HOUR, OperationDateTime),
    DATEPART(WEEKDAY, OperationDateTime),
    CASE WHEN DATEPART(HOUR, OperationDateTime) BETWEEN 8 AND 17 THEN 1 ELSE 0 END
FROM staging.WarehouseOperations
WHERE OperationDateTime NOT IN (SELECT FullDateTime FROM DimTime);
```

### Issue: Slow Cross-Fact Queries

**Diagnosis:**
```sql
-- Check index fragmentation
SELECT 
    OBJECT_NAME(ips.object_id) AS TableName,
    i.name AS IndexName,
    ips.avg_fragmentation_in_percent,
    ips.page_count
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'SAMPLED') ips
JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.avg_fragmentation_in_percent > 30
    AND ips.page_count > 1000
ORDER BY ips.avg_fragmentation_in_percent DESC;
```

**Solution:**
```sql
-- Rebuild fragmented indexes
ALTER INDEX ALL ON dbo.FactWarehouseOperations REBUILD WITH (ONLINE = ON);
ALTER INDEX ALL ON dbo.FactFleetTrips REBUILD WITH (ONLINE = ON);

-- Create covering index for common cross-fact join
CREATE NONCLUSTERED INDEX IX_WarehouseOps_TimeProduct
ON dbo.FactWarehouseOperations (TimeKey, ProductKey)
INCLUDE (DwellTimeMinutes, CurrentZone);

CREATE NONCLUSTERED INDEX IX_FleetTrips_TimeRoute
ON dbo.FactFleetTrips (TimeKey, RouteKey)
INCLUDE (IdleTimeMinutes, FuelConsumedLiters);
```

### Issue: Gravity Score Calculation Timeouts

**Diagnosis:**
```sql
-- Check row counts and data distribution
SELECT 
    'FactWarehouseOperations' AS TableName,
    COUNT(*) AS TotalRows,
    COUNT(DISTINCT ProductKey) AS DistinctProducts,
    MIN(TimeKey) AS OldestRecord,
    MAX(TimeKey) AS NewestRecord
FROM dbo.FactWarehouseOperations;
```

**Solution:**
```sql
-- Archive old data (partition or archive table)
-- Create archive table
SELECT * 
INTO dbo.FactWarehouseOperations_Archive_2025
FROM dbo.FactWarehouseOperations
WHERE TimeKey < (SELECT TimeKey FROM DimTime WHERE FullDateTime = '2026-01-01');

-- Delete archived records from main table
DELETE FROM dbo.FactWarehouseOperations
WHERE TimeKey < (SELECT TimeKey FROM DimTime WHERE FullDateTime = '2026-01-01');

-- Add computed column for gravity score (persisted)
ALTER TABLE dbo.DimProductGravity
ADD GravityScore_Computed AS (
    (VelocityScore * 10) + (ValueScore / 100) + (100.0 / NULLIF(FragilityIndex, 0))
) PERSISTED;
```

## Performance Optimization

### Partitioning Strategy

**Partition fact tables by time:**
```sql
-- Create partition function (monthly partitions)
CREATE PARTITION FUNCTION pf_MonthlyRange (INT)
AS RANGE RIGHT FOR VALUES (
    20260101, 20260201, 20260301, 20260401, 20260501, 20260601,
    20260701, 20260801, 20260901, 20261001, 20261101, 20261201
);
GO

-- Create partition scheme
CREATE PARTITION SCHEME ps_MonthlyRange
AS PARTITION pf_MonthlyRange
ALL TO ([PRIMARY]);
GO

-- Recreate fact table with partitioning
CREATE TABLE dbo.FactWarehouseOperations_Partitioned (
    OperationID BIGINT IDENTITY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    -- ... other columns
    CONSTRAINT PK_FactWarehouseOps_Part PRIMARY KEY
