---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI multi-fact star schema for real-time logistics intelligence, warehouse operations, and fleet optimization analytics
triggers:
  - set up logistics analytics warehouse
  - implement supply chain KPI dashboard
  - create multi-fact star schema for fleet data
  - build warehouse operations data model
  - deploy LogiFleet Pulse analytics
  - configure Power BI logistics intelligence
  - implement cross-modal supply chain reporting
  - set up real-time fleet optimization dashboard
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

LogiFleet Pulse is an advanced MS SQL Server data warehouse and Power BI visualization platform for logistics and supply chain analytics. It implements a multi-fact star schema that unifies:

- **Warehouse operations** (receiving, putaway, picking, packing, shipping, dwell time)
- **Fleet telemetry** (route tracking, fuel consumption, idle time, maintenance)
- **Inventory management** (aging curves, SKU velocity, gravity zones)
- **External signals** (weather, traffic, supplier reliability)

The platform uses time-phased dimensions, bridge tables for many-to-many relationships, and composite KPIs to enable cross-fact analytics like "dwell time per SKU vs. fleet idling cost per route."

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to data sources: WMS, TMS, GPS/telematics APIs

### Deployment Steps

1. **Clone the repository:**

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**

```sql
-- Connect to your SQL Server instance
-- Execute the schema creation script
USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Run the full schema script (typically named schema.sql or similar)
-- This creates all fact tables, dimensions, views, and stored procedures
```

3. **Configure data source connections:**

```json
// config.json (example structure - adapt to actual file)
{
  "sql_server": {
    "server": "localhost",
    "database": "LogiFleetPulse",
    "authentication": "integrated"
  },
  "data_sources": {
    "wms_api": "${WMS_API_ENDPOINT}",
    "telematics_api": "${TELEMATICS_API_ENDPOINT}",
    "weather_api": "${WEATHER_API_KEY}"
  },
  "refresh_interval_minutes": 15
}
```

4. **Import Power BI template:**

- Open Power BI Desktop
- File → Import → Power BI Template (.pbit)
- Select `LogiFleet_Pulse_Master.pbit`
- Enter SQL Server connection parameters
- The model will auto-configure relationships

## Core Data Model Architecture

### Fact Tables

#### FactWarehouseOperations

```sql
CREATE TABLE FactWarehouseOperations (
    OperationID INT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    QuantityHandled INT,
    DwellTimeMinutes INT,
    CycleTimeSeconds INT,
    WorkerID INT,
    ZoneGravityScore DECIMAL(5,2),
    CONSTRAINT FK_WarehouseOps_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WarehouseOps_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_WarehouseOps_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
);

-- Recommended indexes
CREATE NONCLUSTERED INDEX IX_FactWarehouseOps_Time 
    ON FactWarehouseOperations(TimeKey) 
    INCLUDE (DwellTimeMinutes, CycleTimeSeconds);

CREATE NONCLUSTERED INDEX IX_FactWarehouseOps_Product 
    ON FactWarehouseOperations(ProductKey, TimeKey) 
    INCLUDE (QuantityHandled);
```

#### FactFleetTrips

```sql
CREATE TABLE FactFleetTrips (
    TripID INT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    DistanceKm DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    DelayMinutes INT,
    DelayReasonKey INT,
    DriverID INT,
    CONSTRAINT FK_FleetTrips_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FleetTrips_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_FleetTrips_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FleetTrips_Destination FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
);

-- Recommended indexes
CREATE NONCLUSTERED INDEX IX_FactFleetTrips_Route 
    ON FactFleetTrips(RouteKey, TimeKey) 
    INCLUDE (FuelConsumedLiters, IdleTimeMinutes);
```

### Dimension Tables

#### DimTime (15-minute granularity)

```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    Date DATE NOT NULL,
    Year INT NOT NULL,
    Quarter INT NOT NULL,
    Month INT NOT NULL,
    DayOfMonth INT NOT NULL,
    DayOfWeek INT NOT NULL,
    Hour INT NOT NULL,
    Minute INT NOT NULL,
    IsWeekend BIT NOT NULL,
    FiscalYear INT,
    FiscalQuarter INT
);

-- Populate with 15-minute intervals
INSERT INTO DimTime (TimeKey, FullDateTime, Date, Year, Quarter, Month, DayOfMonth, DayOfWeek, Hour, Minute, IsWeekend)
SELECT 
    CAST(FORMAT(dt, 'yyyyMMddHHmm') AS INT) AS TimeKey,
    dt AS FullDateTime,
    CAST(dt AS DATE) AS Date,
    YEAR(dt) AS Year,
    DATEPART(QUARTER, dt) AS Quarter,
    MONTH(dt) AS Month,
    DAY(dt) AS DayOfMonth,
    DATEPART(WEEKDAY, dt) AS DayOfWeek,
    DATEPART(HOUR, dt) AS Hour,
    DATEPART(MINUTE, dt) AS Minute,
    CASE WHEN DATEPART(WEEKDAY, dt) IN (1, 7) THEN 1 ELSE 0 END AS IsWeekend
FROM (
    SELECT DATEADD(MINUTE, 15 * ROW_NUMBER() OVER (ORDER BY (SELECT NULL)), '2025-01-01') AS dt
    FROM sys.all_columns a CROSS JOIN sys.all_columns b
) AS DateTable
WHERE dt < '2028-01-01';
```

#### DimProductGravity

```sql
CREATE TABLE DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    ProductSKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName VARCHAR(255),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    VelocityClass VARCHAR(20), -- 'Fast', 'Medium', 'Slow'
    ValueTier VARCHAR(20), -- 'High', 'Medium', 'Low'
    FragilityScore DECIMAL(3,2), -- 0.00 to 1.00
    GravityScore DECIMAL(5,2), -- Composite: velocity * value * (1/fragility)
    RecommendedZone VARCHAR(50)
);

-- Calculate gravity score
UPDATE DimProductGravity
SET GravityScore = 
    (CASE VelocityClass 
        WHEN 'Fast' THEN 3 
        WHEN 'Medium' THEN 2 
        ELSE 1 END) *
    (CASE ValueTier 
        WHEN 'High' THEN 3 
        WHEN 'Medium' THEN 2 
        ELSE 1 END) *
    (1.0 / NULLIF(FragilityScore, 0));
```

## Key SQL Queries and Patterns

### Cross-Fact Analysis: Dwell Time vs. Fleet Idle Cost

```sql
-- Correlate warehouse dwell time with fleet idle time for same products
WITH WarehouseDwell AS (
    SELECT 
        p.ProductSKU,
        p.Category,
        AVG(w.DwellTimeMinutes) AS AvgDwellMinutes,
        SUM(w.QuantityHandled) AS TotalQuantity
    FROM FactWarehouseOperations w
    JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.Date >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY p.ProductSKU, p.Category
),
FleetIdle AS (
    SELECT 
        p.ProductSKU,
        AVG(f.IdleTimeMinutes) AS AvgIdleMinutes,
        SUM(f.FuelConsumedLiters * 1.5) AS IdleCostUSD -- $1.50/liter fuel cost
    FROM FactFleetTrips f
    JOIN BridgeRouteProduct br ON f.TripID = br.TripID
    JOIN DimProductGravity p ON br.ProductKey = p.ProductKey
    JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.Date >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY p.ProductSKU
)
SELECT 
    wd.ProductSKU,
    wd.Category,
    wd.AvgDwellMinutes,
    fi.AvgIdleMinutes,
    fi.IdleCostUSD,
    (wd.AvgDwellMinutes * fi.AvgIdleMinutes) AS FrictionScore
FROM WarehouseDwell wd
LEFT JOIN FleetIdle fi ON wd.ProductSKU = fi.ProductSKU
ORDER BY FrictionScore DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify products that should be moved to higher gravity zones
SELECT 
    p.ProductSKU,
    p.ProductName,
    p.GravityScore,
    p.RecommendedZone AS CurrentZone,
    AVG(w.CycleTimeSeconds) AS AvgCycleTime,
    COUNT(*) AS PickFrequency,
    CASE 
        WHEN p.GravityScore > 20 AND p.RecommendedZone != 'Zone_A' THEN 'MOVE_TO_A'
        WHEN p.GravityScore BETWEEN 10 AND 20 AND p.RecommendedZone = 'Zone_C' THEN 'MOVE_TO_B'
        ELSE 'KEEP_CURRENT'
    END AS RecommendedAction
FROM FactWarehouseOperations w
JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
JOIN DimTime t ON w.TimeKey = t.TimeKey
WHERE 
    w.OperationType = 'Picking'
    AND t.Date >= DATEADD(MONTH, -1, GETDATE())
GROUP BY 
    p.ProductKey, p.ProductSKU, p.ProductName, 
    p.GravityScore, p.RecommendedZone
HAVING COUNT(*) > 50
ORDER BY PickFrequency DESC;
```

### Predictive Bottleneck Detection

```sql
-- Stored procedure for real-time bottleneck alerts
CREATE PROCEDURE sp_DetectBottlenecks
    @ThresholdMultiplier DECIMAL(3,2) = 1.5
AS
BEGIN
    -- Detect warehouse operations exceeding normal cycle times
    WITH NormalBaseline AS (
        SELECT 
            OperationType,
            AVG(CycleTimeSeconds) AS AvgCycleTime,
            STDEV(CycleTimeSeconds) AS StdDevCycleTime
        FROM FactWarehouseOperations
        WHERE TimeKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm') AS INT)
        GROUP BY OperationType
    ),
    CurrentOperations AS (
        SELECT 
            w.OperationID,
            w.OperationType,
            w.CycleTimeSeconds,
            w.WarehouseKey,
            w.ProductKey,
            t.FullDateTime
        FROM FactWarehouseOperations w
        JOIN DimTime t ON w.TimeKey = t.TimeKey
        WHERE t.FullDateTime >= DATEADD(HOUR, -2, GETDATE())
    )
    SELECT 
        co.OperationID,
        co.OperationType,
        co.CycleTimeSeconds,
        nb.AvgCycleTime,
        (co.CycleTimeSeconds - nb.AvgCycleTime) AS DeviationSeconds,
        (co.CycleTimeSeconds / NULLIF(nb.AvgCycleTime, 0)) AS DeviationRatio,
        g.WarehouseName,
        p.ProductSKU,
        co.FullDateTime,
        'BOTTLENECK_ALERT' AS AlertType
    FROM CurrentOperations co
    JOIN NormalBaseline nb ON co.OperationType = nb.OperationType
    JOIN DimGeography g ON co.WarehouseKey = g.GeographyKey
    JOIN DimProductGravity p ON co.ProductKey = p.ProductKey
    WHERE co.CycleTimeSeconds > (nb.AvgCycleTime + (@ThresholdMultiplier * nb.StdDevCycleTime))
    ORDER BY DeviationRatio DESC;
END;
GO

-- Execute bottleneck detection
EXEC sp_DetectBottlenecks @ThresholdMultiplier = 1.5;
```

## Power BI DAX Measures

### Fleet Utilization Rate

```dax
Fleet Utilization % = 
VAR TotalTime = SUM(FactFleetTrips[DistanceKm]) / AVERAGE(FactFleetTrips[DistanceKm]) * 60
VAR ActiveTime = TotalTime - SUM(FactFleetTrips[IdleTimeMinutes])
RETURN
DIVIDE(ActiveTime, TotalTime, 0) * 100
```

### Composite Friction Score

```dax
Friction Score = 
VAR WarehouseDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR FleetIdle = AVERAGE(FactFleetTrips[IdleTimeMinutes])
RETURN
WarehouseDwell * FleetIdle / 100
```

### Dynamic Gravity Zone Assignment

```dax
Optimal Zone = 
SWITCH(
    TRUE(),
    [Product Gravity Score] > 20, "Zone A - High Priority",
    [Product Gravity Score] > 10, "Zone B - Medium Priority",
    "Zone C - Low Priority"
)
```

## Incremental Data Loading

### Stored Procedure for ETL

```sql
CREATE PROCEDURE sp_IncrementalLoad_WarehouseOps
    @LastLoadDateTime DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Get last load time if not provided
    IF @LastLoadDateTime IS NULL
        SELECT @LastLoadDateTime = MAX(LoadDateTime) 
        FROM ETL_LoadLog 
        WHERE TableName = 'FactWarehouseOperations';
    
    -- Insert new records from staging
    INSERT INTO FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductKey, OperationType,
        QuantityHandled, DwellTimeMinutes, CycleTimeSeconds,
        WorkerID, ZoneGravityScore
    )
    SELECT 
        CAST(FORMAT(s.OperationDateTime, 'yyyyMMddHHmm') AS INT) AS TimeKey,
        g.GeographyKey AS WarehouseKey,
        p.ProductKey,
        s.OperationType,
        s.Quantity,
        DATEDIFF(MINUTE, s.StartTime, s.EndTime) AS DwellTimeMinutes,
        DATEDIFF(SECOND, s.StartTime, s.EndTime) AS CycleTimeSeconds,
        s.WorkerID,
        p.GravityScore
    FROM Staging_WarehouseOperations s
    JOIN DimGeography g ON s.WarehouseCode = g.WarehouseCode
    JOIN DimProductGravity p ON s.SKU = p.ProductSKU
    WHERE s.OperationDateTime > @LastLoadDateTime;
    
    -- Log completion
    INSERT INTO ETL_LoadLog (TableName, LoadDateTime, RowsInserted)
    VALUES ('FactWarehouseOperations', GETDATE(), @@ROWCOUNT);
END;
GO
```

## Configuration and Security

### Row-Level Security Setup

```sql
-- Create security predicate function
CREATE FUNCTION dbo.fn_SecurityPredicate_Geography(@GeographyKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS SecurityCheck
WHERE @GeographyKey IN (
    SELECT GeographyKey 
    FROM dbo.UserGeographyAccess 
    WHERE UserName = USER_NAME()
);
GO

-- Apply security policy to fact tables
CREATE SECURITY POLICY GeographySecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate_Geography(WarehouseKey)
    ON dbo.FactWarehouseOperations,
ADD FILTER PREDICATE dbo.fn_SecurityPredicate_Geography(OriginGeographyKey)
    ON dbo.FactFleetTrips
WITH (STATE = ON);
GO
```

### Environment Variables for Connections

```sql
-- Use SQL Server configuration or external config files
-- Never hardcode credentials in scripts

-- Example: Using linked server for external data
EXEC sp_addlinkedserver 
    @server = 'WMS_EXTERNAL',
    @srvproduct = '',
    @provider = 'SQLNCLI',
    @datasrc = '${WMS_SERVER}',
    @catalog = '${WMS_DATABASE}';

EXEC sp_addlinkedsrvlogin 
    @rmtsrvname = 'WMS_EXTERNAL',
    @useself = 'FALSE',
    @rmtuser = '${WMS_USERNAME}',
    @rmtpassword = '${WMS_PASSWORD}';
```

## Automated Alerting

### SQL Server Agent Job for Threshold Monitoring

```sql
-- Create alert job
USE msdb;
GO

EXEC sp_add_job
    @job_name = 'LogiFleet_Bottleneck_Monitor',
    @enabled = 1,
    @description = 'Detect and alert on logistics bottlenecks';

EXEC sp_add_jobstep
    @job_name = 'LogiFleet_Bottleneck_Monitor',
    @step_name = 'Run Bottleneck Detection',
    @subsystem = 'TSQL',
    @command = '
        DECLARE @Alerts TABLE (
            OperationID INT,
            AlertMessage VARCHAR(500)
        );
        
        INSERT INTO @Alerts
        SELECT 
            OperationID,
            ''Cycle time exceeded by '' + CAST(DeviationRatio AS VARCHAR(10)) + ''x''
        FROM (
            EXEC sp_DetectBottlenecks @ThresholdMultiplier = 1.5
        ) AS Bottlenecks;
        
        -- Send email alerts (configure Database Mail first)
        IF EXISTS (SELECT 1 FROM @Alerts)
        BEGIN
            EXEC msdb.dbo.sp_send_dbmail
                @profile_name = ''LogiFleet_Mail'',
                @recipients = ''${ALERT_EMAIL}'',
                @subject = ''LogiFleet Bottleneck Alert'',
                @body = ''Critical bottlenecks detected. Check dashboard.'';
        END;
    ',
    @database_name = 'LogiFleetPulse';

-- Schedule every 15 minutes
EXEC sp_add_schedule
    @schedule_name = 'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;

EXEC sp_attach_schedule
    @job_name = 'LogiFleet_Bottleneck_Monitor',
    @schedule_name = 'Every15Minutes';

EXEC sp_add_jobserver
    @job_name = 'LogiFleet_Bottleneck_Monitor';
GO
```

## Troubleshooting

### Performance Issues

**Symptom:** Slow Power BI dashboard refresh

**Solutions:**
```sql
-- 1. Check missing indexes
SELECT 
    DB_NAME(database_id) AS DatabaseName,
    OBJECT_NAME(object_id) AS TableName,
    equality_columns,
    inequality_columns,
    included_columns
FROM sys.dm_db_missing_index_details
WHERE database_id = DB_ID('LogiFleetPulse');

-- 2. Update statistics
EXEC sp_updatestats;

-- 3. Rebuild fragmented indexes
SELECT 
    OBJECT_NAME(ps.object_id) AS TableName,
    i.name AS IndexName,
    ps.avg_fragmentation_in_percent
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'SAMPLED') ps
JOIN sys.indexes i ON ps.object_id = i.object_id AND ps.index_id = i.index_id
WHERE ps.avg_fragmentation_in_percent > 30;

-- Rebuild specific index
ALTER INDEX IX_FactWarehouseOps_Time 
    ON FactWarehouseOperations REBUILD;
```

### Data Quality Issues

**Symptom:** Missing relationships or null foreign keys

**Solution:**
```sql
-- Audit orphaned records
SELECT 
    'FactWarehouseOperations' AS TableName,
    COUNT(*) AS OrphanedRecords
FROM FactWarehouseOperations w
LEFT JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
WHERE p.ProductKey IS NULL

UNION ALL

SELECT 
    'FactFleetTrips',
    COUNT(*)
FROM FactFleetTrips f
LEFT JOIN DimTime t ON f.TimeKey = t.TimeKey
WHERE t.TimeKey IS NULL;

-- Fix by inserting missing dimension records or updating fact table
```

### Power BI Connection Issues

**Symptom:** "Cannot connect to data source"

**Solutions:**
1. Verify SQL Server allows remote connections:
```sql
EXEC sp_configure 'remote access', 1;
RECONFIGURE;
```

2. Check firewall rules (port 1433 for default SQL Server)
3. Use SQL Server authentication if Windows auth fails:
```
Server: localhost\SQLEXPRESS
Database: LogiFleetPulse
Username: ${SQL_USERNAME}
Password: ${SQL_PASSWORD}
```

### Incremental Load Failures

**Symptom:** Duplicate records or skipped data

**Solution:**
```sql
-- Add unique constraint to prevent duplicates
ALTER TABLE FactWarehouseOperations
ADD CONSTRAINT UQ_WarehouseOps_SourceID 
    UNIQUE (SourceSystemID, OperationDateTime);

-- Implement upsert logic
MERGE FactWarehouseOperations AS Target
USING Staging_WarehouseOperations AS Source
ON Target.SourceSystemID = Source.SourceSystemID
WHEN MATCHED THEN
    UPDATE SET 
        Target.DwellTimeMinutes = Source.DwellTimeMinutes,
        Target.UpdatedDateTime = GETDATE()
WHEN NOT MATCHED THEN
    INSERT (TimeKey, WarehouseKey, ProductKey, OperationType, QuantityHandled)
    VALUES (Source.TimeKey, Source.WarehouseKey, Source.ProductKey, 
            Source.OperationType, Source.Quantity);
```

## Common Workflow Patterns

### Daily Operations Dashboard Refresh

1. SQL Agent job runs incremental ETL every 15 minutes
2. Power BI dataset configured for DirectQuery or scheduled refresh
3. Dashboard displays real-time KPIs with 15-minute latency
4. Alerts triggered via email/Teams for threshold breaches

### Monthly Gravity Zone Re-optimization

```sql
-- Run at end of each month
EXEC sp_RecalculateGravityScores;
EXEC sp_GenerateZoneRecommendations;

-- Export recommendations to CSV for operations team
bcp "SELECT * FROM ZoneRecommendations" queryout C:\Reports\gravity_changes.csv -c -t, -S localhost -d LogiFleetPulse -T
```

### Quarterly Capacity Planning Analysis

```sql
-- Temporal elasticity simulation
WITH HistoricalLoad AS (
    SELECT 
        YEAR(t.Date) AS Year,
        MONTH(t.Date) AS Month,
        AVG(w.DwellTimeMinutes) AS AvgDwell,
        SUM(w.QuantityHandled) AS TotalVolume
    FROM FactWarehouseOperations w
    JOIN DimTime t ON w.TimeKey = t.TimeKey
    GROUP BY YEAR(t.Date), MONTH(t.Date)
)
SELECT 
    Year, Month, AvgDwell, TotalVolume,
    TotalVolume * 1.25 AS Scenario_125Capacity,
    AvgDwell * 1.15 AS Projected_Dwell_125
FROM HistoricalLoad
ORDER BY Year, Month;
```

## Best Practices

1. **Partitioning**: Partition fact tables by date for better query performance
2. **Compression**: Use page compression on historical data
3. **Archiving**: Move data older than 2 years to archive tables
4. **Monitoring**: Set up SQL Server Profiler traces for long-running queries
5. **Backups**: Daily full backups, hourly transaction log backups
6. **Documentation**: Maintain data dictionary with business definitions for all KPIs

## Additional Resources

- Use SQL Server Dynamic Management Views (DMVs) for performance tuning
- Configure Power BI Premium for larger datasets (>10GB)
- Implement Azure Synapse for big data scenarios (>500GB)
- Use Power Automate to trigger alerts in Microsoft Teams
