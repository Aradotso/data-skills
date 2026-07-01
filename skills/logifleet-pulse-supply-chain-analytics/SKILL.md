---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing template for logistics, fleet management, and supply chain analytics with multi-fact star schema
triggers:
  - "set up logistics analytics dashboard"
  - "create supply chain data warehouse"
  - "build fleet management reporting system"
  - "implement warehouse operations analytics"
  - "configure Power BI logistics template"
  - "design multi-fact star schema for supply chain"
  - "deploy logifleet pulse analytics"
  - "integrate warehouse and fleet data"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive data warehousing and analytics template for logistics operations. It provides a multi-fact star schema architecture that unifies warehouse operations, fleet telemetry, and supply chain data into a single semantic layer using MS SQL Server and Power BI.

**Core Components:**
- SQL schema with fact tables for warehouse operations, fleet trips, and cross-dock activities
- Time-phased dimensions for temporal analysis
- Power BI template (`.pbit`) with pre-built dashboards
- Stored procedures for incremental loading and alerting
- Role-based access control and row-level security

**Primary Use Cases:**
- Real-time logistics operations monitoring
- Fleet optimization and maintenance prioritization
- Warehouse space utilization and gravity zone analysis
- Cross-functional supply chain KPI harmonization
- Predictive bottleneck detection

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to source systems (WMS, TMS, telemetry APIs)

### Database Setup

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**
```sql
-- Connect to your SQL Server instance
-- Execute the main schema creation script
:r deploy_schema.sql

-- Verify table creation
SELECT TABLE_SCHEMA, TABLE_NAME, TABLE_TYPE
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'dbo'
AND TABLE_NAME LIKE 'Fact%' OR TABLE_NAME LIKE 'Dim%'
ORDER BY TABLE_NAME;
```

3. **Configure connection strings:**
```json
// config.json (create from config_sample.json)
{
  "sqlServer": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "SqlPassword",
    "username": "${SQL_USERNAME}",
    "password": "${SQL_PASSWORD}"
  },
  "dataSources": {
    "wmsApi": "${WMS_API_ENDPOINT}",
    "telemetryApi": "${FLEET_TELEMETRY_ENDPOINT}",
    "erpConnection": "${ERP_CONNECTION_STRING}"
  }
}
```

## Core Data Model

### Fact Tables

**FactWarehouseOperations** - Warehouse activity transactions:
```sql
CREATE TABLE FactWarehouseOperations (
    OperationID INT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    LocationKey INT NOT NULL,
    OperationType VARCHAR(20), -- Putaway, Pick, Pack, Ship
    QuantityHandled DECIMAL(10,2),
    DwellTimeMinutes INT,
    CycleTimeSeconds INT,
    EmployeeID INT,
    LoadedTimestamp DATETIME2 DEFAULT GETDATE(),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    FOREIGN KEY (LocationKey) REFERENCES DimGeography(LocationKey)
);

-- Create clustered columnstore index for analytics
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactWarehouseOps 
ON FactWarehouseOperations;
```

**FactFleetTrips** - Fleet and route performance:
```sql
CREATE TABLE FactFleetTrips (
    TripID INT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    OriginLocationKey INT NOT NULL,
    DestinationLocationKey INT NOT NULL,
    DistanceKM DECIMAL(10,2),
    FuelLiters DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    DelayMinutes INT,
    DelayReason VARCHAR(100),
    LoadedTimestamp DATETIME2 DEFAULT GETDATE(),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    FOREIGN KEY (OriginLocationKey) REFERENCES DimGeography(LocationKey),
    FOREIGN KEY (DestinationLocationKey) REFERENCES DimGeography(LocationKey)
);
```

### Dimension Tables

**DimTime** - 15-minute granularity time dimension:
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    DateKey INT, -- YYYYMMDD format
    TimeOfDay TIME,
    HourOfDay TINYINT,
    MinuteOfHour TINYINT,
    DayOfWeek TINYINT,
    DayName VARCHAR(10),
    IsWeekend BIT,
    WeekOfYear TINYINT,
    MonthNumber TINYINT,
    MonthName VARCHAR(10),
    Quarter TINYINT,
    Year SMALLINT,
    FiscalPeriod VARCHAR(10)
);

-- Populate time dimension (example for one day)
DECLARE @StartDate DATETIME2 = '2026-01-01 00:00:00';
DECLARE @EndDate DATETIME2 = '2026-12-31 23:45:00';

WITH TimeSeries AS (
    SELECT @StartDate AS TimeValue
    UNION ALL
    SELECT DATEADD(MINUTE, 15, TimeValue)
    FROM TimeSeries
    WHERE DATEADD(MINUTE, 15, TimeValue) <= @EndDate
)
INSERT INTO DimTime (TimeKey, FullDateTime, DateKey, TimeOfDay, HourOfDay, MinuteOfHour, DayOfWeek, DayName, IsWeekend)
SELECT 
    CAST(FORMAT(TimeValue, 'yyyyMMddHHmm') AS INT) AS TimeKey,
    TimeValue,
    CAST(FORMAT(TimeValue, 'yyyyMMdd') AS INT) AS DateKey,
    CAST(TimeValue AS TIME) AS TimeOfDay,
    DATEPART(HOUR, TimeValue),
    DATEPART(MINUTE, TimeValue),
    DATEPART(WEEKDAY, TimeValue),
    DATENAME(WEEKDAY, TimeValue),
    CASE WHEN DATEPART(WEEKDAY, TimeValue) IN (1, 7) THEN 1 ELSE 0 END
FROM TimeSeries
OPTION (MAXRECURSION 0);
```

**DimProductGravity** - Product classification with gravity scoring:
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    ProductSKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName VARCHAR(200),
    Category VARCHAR(50),
    Subcategory VARCHAR(50),
    UnitValue DECIMAL(10,2),
    WeightKG DECIMAL(8,3),
    FragilityScore TINYINT, -- 1-10 scale
    VelocityScore TINYINT, -- 1-10 (calculated from pick frequency)
    GravityScore AS (VelocityScore * 0.5 + (UnitValue / NULLIF(WeightKG, 0)) * 0.3 + FragilityScore * 0.2) PERSISTED,
    RecommendedZone VARCHAR(20), -- High-Gravity, Mid-Gravity, Low-Gravity
    IsCurrent BIT DEFAULT 1,
    EffectiveDate DATE,
    ExpirationDate DATE
);
```

## Key Stored Procedures

### Incremental Data Loading

```sql
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadTimestamp DATETIME2 = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Default to last load if not specified
    IF @LastLoadTimestamp IS NULL
        SELECT @LastLoadTimestamp = MAX(LoadedTimestamp) 
        FROM FactWarehouseOperations;
    
    -- Load new records from staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, LocationKey, OperationType, 
        QuantityHandled, DwellTimeMinutes, CycleTimeSeconds, EmployeeID
    )
    SELECT 
        CAST(FORMAT(s.OperationTimestamp, 'yyyyMMddHHmm') AS INT) AS TimeKey,
        p.ProductKey,
        g.LocationKey,
        s.OperationType,
        s.Quantity,
        DATEDIFF(MINUTE, s.StartTime, s.EndTime) AS DwellTimeMinutes,
        DATEDIFF(SECOND, s.StartTime, s.EndTime) AS CycleTimeSeconds,
        s.EmployeeID
    FROM StagingWarehouseOps s
    INNER JOIN DimProductGravity p ON s.ProductSKU = p.ProductSKU AND p.IsCurrent = 1
    INNER JOIN DimGeography g ON s.LocationCode = g.LocationCode
    WHERE s.OperationTimestamp > @LastLoadTimestamp
        AND s.IsProcessed = 0;
    
    -- Mark staging records as processed
    UPDATE StagingWarehouseOps
    SET IsProcessed = 1
    WHERE OperationTimestamp > @LastLoadTimestamp;
    
    PRINT 'Loaded ' + CAST(@@ROWCOUNT AS VARCHAR) + ' warehouse operations';
END;
GO
```

### Alert Generation

```sql
CREATE PROCEDURE usp_GenerateFleetAlerts
    @ThresholdIdlePercent DECIMAL(5,2) = 15.0,
    @EmailRecipient VARCHAR(255) = '${ALERT_EMAIL}'
AS
BEGIN
    -- Find vehicles exceeding idle time threshold
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    WITH FleetMetrics AS (
        SELECT 
            v.VehicleID,
            v.VehiclePlate,
            SUM(f.IdleTimeMinutes) AS TotalIdleMinutes,
            SUM(DATEDIFF(MINUTE, t.FullDateTime, 
                DATEADD(MINUTE, f.LoadingTimeMinutes + f.UnloadingTimeMinutes, t.FullDateTime))) AS TotalTripMinutes,
            (SUM(f.IdleTimeMinutes) * 100.0 / 
                NULLIF(SUM(DATEDIFF(MINUTE, t.FullDateTime, 
                    DATEADD(MINUTE, f.LoadingTimeMinutes + f.UnloadingTimeMinutes, t.FullDateTime))), 0)) AS IdlePercent
        FROM FactFleetTrips f
        INNER JOIN DimVehicle v ON f.VehicleKey = v.VehicleKey
        INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
        WHERE t.FullDateTime >= DATEADD(DAY, -7, GETDATE())
        GROUP BY v.VehicleID, v.VehiclePlate
        HAVING (SUM(f.IdleTimeMinutes) * 100.0 / 
                NULLIF(SUM(DATEDIFF(MINUTE, t.FullDateTime, 
                    DATEADD(MINUTE, f.LoadingTimeMinutes + f.UnloadingTimeMinutes, t.FullDateTime))), 0)) > @ThresholdIdlePercent
    )
    SELECT @AlertMessage = STRING_AGG(
        'Vehicle ' + VehiclePlate + ': ' + 
        CAST(IdlePercent AS VARCHAR(10)) + '% idle time', 
        CHAR(13) + CHAR(10)
    )
    FROM FleetMetrics;
    
    -- Log alert to table
    IF @AlertMessage IS NOT NULL
    BEGIN
        INSERT INTO AlertLog (AlertType, AlertMessage, GeneratedAt)
        VALUES ('FleetIdleExceeded', @AlertMessage, GETDATE());
        
        -- Send email via database mail (configure separately)
        -- EXEC msdb.dbo.sp_send_dbmail ...
        
        PRINT 'Alert generated: ' + @AlertMessage;
    END
END;
GO
```

## Power BI Integration

### Connecting to SQL Server

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. Enter connection parameters when prompted:
   - Server: Your SQL Server instance
   - Database: LogiFleetPulse

### Key DAX Measures

**Fleet Utilization Rate:**
```dax
Fleet Utilization % = 
VAR TotalTime = SUM(FactFleetTrips[LoadingTimeMinutes]) + 
                SUM(FactFleetTrips[UnloadingTimeMinutes]) + 
                SUM(FactFleetTrips[IdleTimeMinutes])
VAR ActiveTime = SUM(FactFleetTrips[LoadingTimeMinutes]) + 
                 SUM(FactFleetTrips[UnloadingTimeMinutes])
RETURN
DIVIDE(ActiveTime, TotalTime, 0) * 100
```

**Warehouse Dwell Time Average by Gravity Zone:**
```dax
Avg Dwell by Zone = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    ALLEXCEPT(DimProductGravity, DimProductGravity[RecommendedZone])
)
```

**Cross-Fact KPI: Cost per Delivered SKU:**
```dax
Cost per Delivered SKU = 
VAR TotalFuelCost = SUMX(FactFleetTrips, FactFleetTrips[FuelLiters] * 1.50)
VAR TotalDeliveredQty = SUM(FactWarehouseOperations[QuantityHandled])
RETURN
DIVIDE(TotalFuelCost, TotalDeliveredQty, 0)
```

**Time Intelligence: YoY Growth:**
```dax
Warehouse Ops YoY = 
VAR CurrentPeriod = SUM(FactWarehouseOperations[QuantityHandled])
VAR PriorYear = CALCULATE(
    SUM(FactWarehouseOperations[QuantityHandled]),
    SAMEPERIODLASTYEAR(DimTime[FullDateTime])
)
RETURN
DIVIDE(CurrentPeriod - PriorYear, PriorYear, 0) * 100
```

## Common Patterns

### Pattern 1: Temporal Analysis with 15-Minute Buckets

```sql
-- Analyze peak warehouse activity hours
SELECT 
    t.HourOfDay,
    t.DayName,
    COUNT(*) AS OperationCount,
    AVG(w.CycleTimeSeconds) AS AvgCycleTime,
    SUM(w.QuantityHandled) AS TotalQuantity
FROM FactWarehouseOperations w
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
WHERE t.FullDateTime >= DATEADD(MONTH, -1, GETDATE())
GROUP BY t.HourOfDay, t.DayName, t.DayOfWeek
ORDER BY t.DayOfWeek, t.HourOfDay;
```

### Pattern 2: Cross-Fact Analysis

```sql
-- Correlate warehouse dwell time with fleet delays
WITH WarehouseDwell AS (
    SELECT 
        t.DateKey,
        g.LocationKey,
        AVG(w.DwellTimeMinutes) AS AvgDwell
    FROM FactWarehouseOperations w
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    INNER JOIN DimGeography g ON w.LocationKey = g.LocationKey
    WHERE w.OperationType = 'Ship'
    GROUP BY t.DateKey, g.LocationKey
),
FleetDelays AS (
    SELECT 
        t.DateKey,
        f.OriginLocationKey,
        AVG(f.DelayMinutes) AS AvgDelay
    FROM FactFleetTrips f
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    GROUP BY t.DateKey, f.OriginLocationKey
)
SELECT 
    wd.DateKey,
    l.LocationName,
    wd.AvgDwell,
    ISNULL(fd.AvgDelay, 0) AS AvgDelay,
    CASE 
        WHEN wd.AvgDwell > 60 AND fd.AvgDelay > 30 THEN 'High Risk'
        WHEN wd.AvgDwell > 60 OR fd.AvgDelay > 30 THEN 'Medium Risk'
        ELSE 'Low Risk'
    END AS RiskLevel
FROM WarehouseDwell wd
LEFT JOIN FleetDelays fd ON wd.DateKey = fd.DateKey AND wd.LocationKey = fd.OriginLocationKey
INNER JOIN DimGeography l ON wd.LocationKey = l.LocationKey
ORDER BY wd.DateKey DESC, RiskLevel DESC;
```

### Pattern 3: Gravity Zone Optimization

```sql
-- Identify products that should be reassigned to different gravity zones
WITH ProductPerformance AS (
    SELECT 
        p.ProductKey,
        p.ProductSKU,
        p.ProductName,
        p.RecommendedZone,
        COUNT(*) AS PickFrequency,
        AVG(w.CycleTimeSeconds) AS AvgCycleTime,
        p.GravityScore
    FROM FactWarehouseOperations w
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    WHERE w.OperationType = 'Pick'
        AND w.TimeKey >= CAST(FORMAT(DATEADD(MONTH, -3, GETDATE()), 'yyyyMMddHHmm') AS INT)
    GROUP BY p.ProductKey, p.ProductSKU, p.ProductName, p.RecommendedZone, p.GravityScore
)
SELECT 
    ProductSKU,
    ProductName,
    RecommendedZone AS CurrentZone,
    PickFrequency,
    AvgCycleTime,
    GravityScore,
    CASE 
        WHEN PickFrequency > 100 AND RecommendedZone != 'High-Gravity' THEN 'Move to High-Gravity'
        WHEN PickFrequency < 10 AND RecommendedZone = 'High-Gravity' THEN 'Move to Low-Gravity'
        ELSE 'Keep Current'
    END AS Recommendation
FROM ProductPerformance
WHERE (PickFrequency > 100 AND RecommendedZone != 'High-Gravity')
   OR (PickFrequency < 10 AND RecommendedZone = 'High-Gravity')
ORDER BY PickFrequency DESC;
```

## Configuration

### Row-Level Security Setup

```sql
-- Create security roles
CREATE ROLE WarehouseViewer;
CREATE ROLE FleetManager;
CREATE ROLE ExecutiveAccess;

-- Create security predicate function
CREATE FUNCTION dbo.fn_SecurityPredicate(@LocationKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS SecurityCheck
WHERE 
    -- Executives see everything
    IS_MEMBER('ExecutiveAccess') = 1
    OR
    -- Warehouse viewers see only their locations
    (IS_MEMBER('WarehouseViewer') = 1 AND @LocationKey IN (
        SELECT LocationKey FROM dbo.UserLocationAccess 
        WHERE Username = USER_NAME()
    ))
    OR
    -- Fleet managers see fleet data
    IS_MEMBER('FleetManager') = 1;
GO

-- Apply security policy to warehouse operations
CREATE SECURITY POLICY WarehouseSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(LocationKey)
ON dbo.FactWarehouseOperations
WITH (STATE = ON);
```

### Scheduled Refresh Configuration

```sql
-- Create SQL Agent job for incremental loads (T-SQL example)
USE msdb;
GO

EXEC dbo.sp_add_job
    @job_name = N'LogiFleet_IncrementalLoad';

EXEC sp_add_jobstep
    @job_name = N'LogiFleet_IncrementalLoad',
    @step_name = N'Load Warehouse Data',
    @subsystem = N'TSQL',
    @command = N'EXEC dbo.usp_LoadWarehouseOperations;',
    @database_name = N'LogiFleetPulse';

EXEC sp_add_jobstep
    @job_name = N'LogiFleet_IncrementalLoad',
    @step_name = N'Load Fleet Data',
    @subsystem = N'TSQL',
    @command = N'EXEC dbo.usp_LoadFleetTrips;',
    @database_name = N'LogiFleetPulse';

-- Schedule every 15 minutes
EXEC sp_add_schedule
    @schedule_name = N'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;

EXEC sp_attach_schedule
    @job_name = N'LogiFleet_IncrementalLoad',
    @schedule_name = N'Every15Minutes';

EXEC sp_add_jobserver
    @job_name = N'LogiFleet_IncrementalLoad';
```

## Troubleshooting

### Issue: Power BI Refresh Fails

**Symptoms:** "Unable to connect to data source" error

**Solutions:**
```sql
-- Verify SQL Server connection
SELECT @@SERVERNAME AS ServerName, DB_NAME() AS DatabaseName;

-- Check user permissions
SELECT 
    dp.name AS UserName,
    dp.type_desc AS UserType,
    o.name AS ObjectName,
    p.permission_name
FROM sys.database_permissions p
INNER JOIN sys.database_principals dp ON p.grantee_principal_id = dp.principal_id
LEFT JOIN sys.objects o ON p.major_id = o.object_id
WHERE dp.name = '${POWERBI_SERVICE_ACCOUNT}';

-- Grant necessary permissions
GRANT SELECT ON SCHEMA::dbo TO [PowerBIServiceAccount];
```

### Issue: Slow Query Performance

**Symptoms:** Dashboards take >30 seconds to load

**Solutions:**
```sql
-- Check missing indexes
SELECT 
    migs.avg_total_user_cost * (migs.avg_user_impact / 100.0) * (migs.user_seeks + migs.user_scans) AS ImprovementMeasure,
    mid.statement AS TableName,
    mid.equality_columns,
    mid.inequality_columns,
    mid.included_columns
FROM sys.dm_db_missing_index_group_stats AS migs
INNER JOIN sys.dm_db_missing_index_groups AS mig ON migs.group_handle = mig.index_group_handle
INNER JOIN sys.dm_db_missing_index_details AS mid ON mig.index_handle = mid.index_handle
WHERE migs.avg_total_user_cost * (migs.avg_user_impact / 100.0) * (migs.user_seeks + migs.user_scans) > 10
ORDER BY ImprovementMeasure DESC;

-- Create recommended indexes
CREATE NONCLUSTERED INDEX IX_FactWarehouseOps_TimeKey_ProductKey 
ON FactWarehouseOperations (TimeKey, ProductKey)
INCLUDE (QuantityHandled, DwellTimeMinutes);

-- Update statistics
UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS FactFleetTrips WITH FULLSCAN;
```

### Issue: Data Not Loading from Source Systems

**Symptoms:** Staging tables empty after ETL run

**Solutions:**
```sql
-- Check external table connectivity (if using PolyBase)
SELECT * FROM sys.external_data_sources;

-- Test API connectivity (if using stored procedure with CLR)
EXEC sp_OACreate 'MSXML2.ServerXMLHTTP', @obj OUT;
EXEC sp_OAMethod @obj, 'open', NULL, 'GET', '${WMS_API_ENDPOINT}/health', false;
EXEC sp_OAMethod @obj, 'send';
EXEC sp_OAGetProperty @obj, 'status', @status OUT;
SELECT @status AS ResponseStatus; -- Should be 200

-- Review staging table for errors
SELECT TOP 100 * 
FROM StagingWarehouseOps 
WHERE IsProcessed = 0 
ORDER BY ReceivedTimestamp DESC;
```

### Issue: Gravity Score Calculation Issues

**Symptoms:** Products assigned to wrong zones

**Solutions:**
```sql
-- Recalculate gravity scores based on recent activity
UPDATE p
SET 
    p.VelocityScore = CASE 
        WHEN PickCount > 100 THEN 10
        WHEN PickCount > 50 THEN 8
        WHEN PickCount > 20 THEN 6
        WHEN PickCount > 10 THEN 4
        ELSE 2
    END,
    p.RecommendedZone = CASE 
        WHEN GravityScore > 7 THEN 'High-Gravity'
        WHEN GravityScore > 4 THEN 'Mid-Gravity'
        ELSE 'Low-Gravity'
    END
FROM DimProductGravity p
INNER JOIN (
    SELECT 
        ProductKey,
        COUNT(*) AS PickCount
    FROM FactWarehouseOperations
    WHERE OperationType = 'Pick'
        AND TimeKey >= CAST(FORMAT(DATEADD(MONTH, -3, GETDATE()), 'yyyyMMddHHmm') AS INT)
    GROUP BY ProductKey
) agg ON p.ProductKey = agg.ProductKey;

-- Verify zone distribution
SELECT 
    RecommendedZone,
    COUNT(*) AS ProductCount,
    AVG(VelocityScore) AS AvgVelocity,
    AVG(GravityScore) AS AvgGravity
FROM DimProductGravity
WHERE IsCurrent = 1
GROUP BY RecommendedZone;
```

## Advanced Usage

### External Data Integration (Weather API)

```sql
-- Create external table for weather delays
CREATE EXTERNAL DATA SOURCE WeatherAPI
WITH (
    TYPE = REST_API,
    LOCATION = '${WEATHER_API_ENDPOINT}',
    CREDENTIAL = WeatherAPICredential
);

-- Load weather data into dimension
INSERT INTO DimWeatherConditions (DateKey, LocationKey, Condition, Temperature, Precipitation)
SELECT 
    CAST(FORMAT(w.ObservationDate, 'yyyyMMdd') AS INT),
    l.LocationKey,
    w.WeatherCode,
    w.TempCelsius,
    w.RainfallMM
FROM EXTERNAL_TABLE_WeatherFeed w
INNER JOIN DimGeography l ON w.StationCode = l.WeatherStationCode;
```

### Predictive Maintenance Scoring

```sql
-- Calculate vehicle maintenance priority
WITH VehicleHealth AS (
    SELECT 
        v.VehicleKey,
        v.VehiclePlate,
        SUM(f.DistanceKM) AS TotalDistance,
        AVG(f.FuelLiters / NULLIF(f.DistanceKM, 0)) AS AvgFuelEfficiency,
        COUNT(CASE WHEN f.DelayReason LIKE '%mechanical%' THEN 1 END) AS MechanicalDelays
    FROM FactFleetTrips f
    INNER JOIN DimVehicle v ON f.VehicleKey = v.VehicleKey
    WHERE f.TimeKey >= CAST(FORMAT(DATEADD(MONTH, -6, GETDATE()), 'yyyyMMddHHmm') AS INT)
    GROUP BY v.VehicleKey, v.VehiclePlate
)
SELECT 
    VehiclePlate,
    TotalDistance,
    AvgFuelEfficiency,
    MechanicalDelays,
    (TotalDistance / 1000) * 0.4 + 
    (CASE WHEN AvgFuelEfficiency > 0.12 THEN 10 ELSE 0 END) * 0.3 +
    (MechanicalDelays * 5) * 0.3 AS MaintenancePriorityScore
FROM VehicleHealth
ORDER BY MaintenancePriorityScore DESC;
```

This skill provides the foundation for implementing and customizing LogiFleet Pulse for logistics analytics. Adjust schema, DAX measures, and stored procedures based on specific business requirements and data sources.
