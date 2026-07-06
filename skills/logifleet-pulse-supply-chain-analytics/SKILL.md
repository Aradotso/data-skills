---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics intelligence platform for warehouse, fleet, and supply chain data modeling and visualization
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "deploy logicore analytics warehouse schema"
  - "configure power bi logistics dashboard"
  - "create multi-fact star schema for fleet data"
  - "build warehouse gravity zone analytics"
  - "implement cross-modal supply chain reporting"
  - "optimize fleet telemetry data model"
  - "set up real-time logistics KPI dashboard"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

LogiFleet Pulse is an advanced MS SQL Server data warehousing solution combined with Power BI dashboards for real-time logistics intelligence. It provides:

- **Multi-fact star schema** linking warehouse operations, fleet telemetry, and supply chain data
- **Time-phased dimensions** (15-minute granularity) for temporal analysis
- **Cross-modal KPI harmonization** between inventory, fleet, and supplier metrics
- **Predictive analytics** for bottleneck detection and maintenance triage
- **Warehouse Gravity Zones** for spatial optimization based on pick frequency and value

The platform integrates WMS, TMS, telematics feeds, and external APIs into a unified semantic layer.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (2022 recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Appropriate permissions for database creation and external data sources

### Step 1: Clone Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### Step 2: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance via SSMS or sqlcmd
-- Execute the main schema deployment script

USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute schema creation scripts in order
:r ./sql/01_create_dimensions.sql
:r ./sql/02_create_facts.sql
:r ./sql/03_create_bridge_tables.sql
:r ./sql/04_create_views.sql
:r ./sql/05_create_stored_procedures.sql
:r ./sql/06_create_indexes.sql
```

### Step 3: Configure Data Sources

Create a configuration file for connection strings:

```json
{
  "connections": {
    "wms_connection": "Server=${WMS_SQL_SERVER};Database=${WMS_DB};Trusted_Connection=True;",
    "telemetry_api": "${TELEMETRY_API_ENDPOINT}",
    "erp_connection": "Server=${ERP_SQL_SERVER};Database=${ERP_DB};User Id=${ERP_USER};Password=${ERP_PASSWORD};",
    "weather_api_key": "${WEATHER_API_KEY}"
  },
  "refresh_intervals": {
    "warehouse_operations": 900,
    "fleet_telemetry": 300,
    "supplier_data": 3600
  }
}
```

### Step 4: Import Power BI Template

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. Enter your SQL Server connection details when prompted
3. Configure row-level security roles via "Manage Roles"
4. Publish to Power BI Service for team access

## Core Data Model Structure

### Fact Tables

#### FactWarehouseOperations

Captures micro-operations at the warehouse level:

```sql
CREATE TABLE FactWarehouseOperations (
    OperationID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeID INT FOREIGN KEY REFERENCES DimTime(TimeID),
    ProductID INT FOREIGN KEY REFERENCES DimProduct(ProductID),
    WarehouseID INT FOREIGN KEY REFERENCES DimWarehouse(WarehouseID),
    EmployeeID INT FOREIGN KEY REFERENCES DimEmployee(EmployeeID),
    OperationType VARCHAR(20), -- 'RECEIVE', 'PUTAWAY', 'PICK', 'PACK', 'SHIP'
    Quantity DECIMAL(18,3),
    DwellTimeMinutes INT,
    GravityZoneID INT FOREIGN KEY REFERENCES DimGravityZone(GravityZoneID),
    CycleTimeSeconds INT,
    CreatedAt DATETIME2 DEFAULT SYSUTCDATETIME()
);

-- Columnstore index for analytical queries
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_WarehouseOps 
ON FactWarehouseOperations (TimeID, ProductID, OperationType, Quantity, DwellTimeMinutes);
```

#### FactFleetTrips

Tracks fleet movements and telemetry:

```sql
CREATE TABLE FactFleetTrips (
    TripID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeID INT FOREIGN KEY REFERENCES DimTime(TimeID),
    VehicleID INT FOREIGN KEY REFERENCES DimVehicle(VehicleID),
    RouteID INT FOREIGN KEY REFERENCES DimRoute(RouteID),
    DriverID INT FOREIGN KEY REFERENCES DimDriver(DriverID),
    OriginGeographyID INT FOREIGN KEY REFERENCES DimGeography(GeographyID),
    DestinationGeographyID INT FOREIGN KEY REFERENCES DimGeography(GeographyID),
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,3),
    IdleTimeMinutes INT,
    AverageSpeedKPH DECIMAL(5,2),
    LoadWeightKG DECIMAL(10,2),
    DelayMinutes INT,
    DelayReasonID INT FOREIGN KEY REFERENCES DimDelayReason(DelayReasonID),
    CreatedAt DATETIME2 DEFAULT SYSUTCDATETIME()
);

CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FleetTrips 
ON FactFleetTrips (TimeID, VehicleID, RouteID, FuelConsumedLiters, IdleTimeMinutes);
```

### Dimension Tables

#### DimTime (Time-Phased Dimension)

```sql
CREATE TABLE DimTime (
    TimeID INT PRIMARY KEY,
    DateTimeValue DATETIME2,
    Date DATE,
    TimeOfDay TIME,
    Hour TINYINT,
    Minute TINYINT,
    QuarterHourBucket TINYINT, -- 0, 15, 30, 45
    DayOfWeek TINYINT,
    DayName VARCHAR(10),
    WeekOfYear TINYINT,
    Month TINYINT,
    Quarter TINYINT,
    Year SMALLINT,
    FiscalPeriod VARCHAR(10),
    IsWeekend BIT,
    IsHoliday BIT
);

-- Populate with 15-minute granularity
EXEC sp_PopulateDimTime @StartDate = '2025-01-01', @EndDate = '2027-12-31';
```

#### DimProductGravity (Warehouse Gravity Zones)

```sql
CREATE TABLE DimProductGravity (
    ProductID INT PRIMARY KEY,
    SKU VARCHAR(50),
    ProductName VARCHAR(200),
    CategoryHierarchy VARCHAR(500), -- Dept > Category > Subcategory
    GravityScore DECIMAL(5,2), -- Composite score: velocity * value * (1/fragility)
    VelocityClass VARCHAR(10), -- 'FAST', 'MEDIUM', 'SLOW'
    ValueTier VARCHAR(10), -- 'HIGH', 'MEDIUM', 'LOW'
    FragilityIndex DECIMAL(3,2),
    OptimalZoneID INT FOREIGN KEY REFERENCES DimGravityZone(GravityZoneID),
    LastRecalculatedAt DATETIME2
);

-- Stored procedure to recalculate gravity scores
CREATE PROCEDURE sp_RecalculateProductGravity
AS
BEGIN
    UPDATE DimProductGravity
    SET GravityScore = (
        (PickFrequency / NULLIF(MaxPickFrequency, 0)) * 0.4 +
        (UnitValue / NULLIF(MaxUnitValue, 0)) * 0.4 +
        (1.0 / NULLIF(FragilityIndex, 0)) * 0.2
    ),
    LastRecalculatedAt = SYSUTCDATETIME()
    FROM DimProductGravity p
    CROSS APPLY (
        SELECT MAX(PickFrequency) AS MaxPickFrequency, 
               MAX(UnitValue) AS MaxUnitValue
        FROM ProductMetrics
    ) m;
END;
```

#### DimGeography (Hierarchical)

```sql
CREATE TABLE DimGeography (
    GeographyID INT PRIMARY KEY IDENTITY(1,1),
    LocationCode VARCHAR(20) UNIQUE,
    LocationName VARCHAR(100),
    LocationType VARCHAR(20), -- 'WAREHOUSE', 'DEPOT', 'CUSTOMER', 'ROUTE_NODE'
    ParentGeographyID INT FOREIGN KEY REFERENCES DimGeography(GeographyID),
    Continent VARCHAR(50),
    Country VARCHAR(50),
    Region VARCHAR(50),
    City VARCHAR(100),
    PostalCode VARCHAR(20),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    HierarchyPath AS (CAST(GeographyID AS VARCHAR(MAX)) + 
        CASE WHEN ParentGeographyID IS NOT NULL 
             THEN '/' + CAST(ParentGeographyID AS VARCHAR(MAX)) 
             ELSE '' END) PERSISTED
);
```

## Common Operations & Patterns

### Loading Warehouse Operations Data

```sql
-- Incremental load from staging table
CREATE PROCEDURE sp_LoadWarehouseOperations
    @BatchStartTime DATETIME2,
    @BatchEndTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactWarehouseOperations (
        TimeID, ProductID, WarehouseID, EmployeeID, 
        OperationType, Quantity, DwellTimeMinutes, GravityZoneID, CycleTimeSeconds
    )
    SELECT 
        t.TimeID,
        p.ProductID,
        w.WarehouseID,
        e.EmployeeID,
        s.OperationType,
        s.Quantity,
        DATEDIFF(MINUTE, s.StartTime, s.EndTime) AS DwellTimeMinutes,
        p.OptimalZoneID AS GravityZoneID,
        DATEDIFF(SECOND, s.StartTime, s.EndTime) AS CycleTimeSeconds
    FROM StagingWarehouseOps s
    INNER JOIN DimTime t ON DATEADD(MINUTE, 
        (DATEDIFF(MINUTE, 0, s.StartTime) / 15) * 15, 0) = t.DateTimeValue
    INNER JOIN DimProduct p ON s.SKU = p.SKU
    INNER JOIN DimWarehouse w ON s.WarehouseCode = w.WarehouseCode
    INNER JOIN DimEmployee e ON s.EmployeeID = e.ExternalEmployeeID
    WHERE s.StartTime >= @BatchStartTime AND s.StartTime < @BatchEndTime
    AND s.ProcessedFlag = 0;
    
    UPDATE StagingWarehouseOps SET ProcessedFlag = 1
    WHERE StartTime >= @BatchStartTime AND StartTime < @BatchEndTime;
END;
GO
```

### Cross-Fact KPI Query: Dwell Time vs Fleet Idle Time

```sql
-- Find correlation between warehouse dwell time and fleet idle time
-- for shipments originating from the same location
CREATE VIEW vw_DwellVsIdleCorrelation AS
SELECT 
    g.Region,
    p.CategoryHierarchy,
    AVG(wo.DwellTimeMinutes) AS AvgWarehouseDwellMin,
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdleMin,
    COUNT(DISTINCT wo.OperationID) AS WarehouseOperations,
    COUNT(DISTINCT ft.TripID) AS FleetTrips,
    -- Correlation coefficient (simplified)
    (COUNT(*) * SUM(wo.DwellTimeMinutes * ft.IdleTimeMinutes) - 
     SUM(wo.DwellTimeMinutes) * SUM(ft.IdleTimeMinutes)) /
    NULLIF(SQRT((COUNT(*) * SUM(POWER(wo.DwellTimeMinutes, 2)) - POWER(SUM(wo.DwellTimeMinutes), 2)) *
                (COUNT(*) * SUM(POWER(ft.IdleTimeMinutes, 2)) - POWER(SUM(ft.IdleTimeMinutes), 2))), 0) 
    AS CorrelationCoefficient
FROM FactWarehouseOperations wo
INNER JOIN DimWarehouse w ON wo.WarehouseID = w.WarehouseID
INNER JOIN DimGeography g ON w.GeographyID = g.GeographyID
INNER JOIN DimProduct p ON wo.ProductID = p.ProductID
INNER JOIN FactFleetTrips ft ON wo.TimeID = ft.TimeID 
    AND g.GeographyID = ft.OriginGeographyID
WHERE wo.OperationType = 'SHIP'
GROUP BY g.Region, p.CategoryHierarchy
HAVING COUNT(*) > 100; -- Statistical significance threshold
GO
```

### Predictive Bottleneck Detection

```sql
-- Identify potential bottlenecks using trend analysis
CREATE PROCEDURE sp_PredictBottlenecks
    @HoursAhead INT = 4
AS
BEGIN
    -- Calculate trend for next N hours based on last 24 hours
    WITH HourlyMetrics AS (
        SELECT 
            w.WarehouseID,
            t.Hour,
            AVG(wo.CycleTimeSeconds) AS AvgCycleTime,
            COUNT(*) AS OperationCount,
            STDEV(wo.CycleTimeSeconds) AS StdDevCycleTime
        FROM FactWarehouseOperations wo
        INNER JOIN DimTime t ON wo.TimeID = t.TimeID
        INNER JOIN DimWarehouse w ON wo.WarehouseID = w.WarehouseID
        WHERE t.DateTimeValue >= DATEADD(HOUR, -24, SYSUTCDATETIME())
        GROUP BY w.WarehouseID, t.Hour
    ),
    Forecast AS (
        SELECT 
            WarehouseID,
            DATEPART(HOUR, DATEADD(HOUR, @HoursAhead, SYSUTCDATETIME())) AS TargetHour,
            AVG(AvgCycleTime) AS PredictedCycleTime,
            AVG(StdDevCycleTime) AS PredictedVariability
        FROM HourlyMetrics
        WHERE Hour = DATEPART(HOUR, DATEADD(HOUR, @HoursAhead, SYSUTCDATETIME()))
        GROUP BY WarehouseID
    )
    SELECT 
        w.WarehouseName,
        f.TargetHour,
        f.PredictedCycleTime,
        f.PredictedVariability,
        CASE 
            WHEN f.PredictedCycleTime > w.MaxCapacityCycleTime THEN 'CRITICAL'
            WHEN f.PredictedCycleTime > w.MaxCapacityCycleTime * 0.8 THEN 'WARNING'
            ELSE 'NORMAL'
        END AS BottleneckRisk,
        w.MaxCapacityCycleTime
    FROM Forecast f
    INNER JOIN DimWarehouse w ON f.WarehouseID = w.WarehouseID
    ORDER BY BottleneckRisk DESC, f.PredictedCycleTime DESC;
END;
GO
```

## Power BI Configuration

### DAX Measures for Cross-Fact KPIs

```dax
// Composite Efficiency Score (Warehouse + Fleet)
CompositeEfficiency = 
VAR WarehouseEff = 
    DIVIDE(
        CALCULATE(SUM(FactWarehouseOperations[Quantity])),
        CALCULATE(SUM(FactWarehouseOperations[CycleTimeSeconds])) / 3600
    )
VAR FleetEff = 
    DIVIDE(
        CALCULATE(SUM(FactFleetTrips[DistanceKM])),
        CALCULATE(SUM(FactFleetTrips[FuelConsumedLiters]))
    )
RETURN
    (WarehouseEff * 0.5) + (FleetEff * 0.5)

// Gravity Zone Adherence %
GravityZoneAdherence = 
DIVIDE(
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[GravityZoneID] = DimProduct[OptimalZoneID]
    ),
    COUNTROWS(FactWarehouseOperations)
)

// Predictive Delay Risk (uses forecast table)
DelayRisk = 
VAR CurrentHour = HOUR(NOW())
VAR ForecastedDelay = 
    CALCULATE(
        AVERAGE(ForecastDelays[PredictedDelayMinutes]),
        ForecastDelays[ForecastHour] = CurrentHour
    )
RETURN
    IF(ForecastedDelay > 30, "HIGH", IF(ForecastedDelay > 15, "MEDIUM", "LOW"))
```

### Row-Level Security (RLS)

```dax
// Role: Regional Manager (only see their region)
[DimGeography[Region]] = USERNAME()

// Role: Warehouse Supervisor (only their warehouse)
[DimWarehouse[WarehouseCode]] IN 
    LOOKUPVALUE(
        EmployeeWarehouseAccess[WarehouseCode],
        EmployeeWarehouseAccess[Email],
        USERPRINCIPALNAME()
    )
```

## Automated Alerting

### SQL Server Agent Job for KPI Threshold Alerts

```sql
-- Create alert stored procedure
CREATE PROCEDURE sp_SendKPIAlerts
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    -- Check for high fleet idle time
    IF EXISTS (
        SELECT 1 FROM FactFleetTrips ft
        INNER JOIN DimTime t ON ft.TimeID = t.TimeID
        WHERE t.DateTimeValue >= DATEADD(HOUR, -1, SYSUTCDATETIME())
        GROUP BY ft.VehicleID
        HAVING AVG(ft.IdleTimeMinutes) > 0.15 * AVG(DATEDIFF(MINUTE, t.DateTimeValue, LEAD(t.DateTimeValue) OVER (ORDER BY t.TimeID)))
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Fleet idle time exceeds 15% threshold in last hour';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL_RECIPIENTS}',
            @subject = 'LogiFleet Pulse - High Fleet Idle Time',
            @body = @AlertMessage;
    END;
    
    -- Check for warehouse capacity bottleneck
    IF EXISTS (
        SELECT 1 FROM FactWarehouseOperations wo
        INNER JOIN DimTime t ON wo.TimeID = t.TimeID
        WHERE t.DateTimeValue >= DATEADD(MINUTE, -15, SYSUTCDATETIME())
        GROUP BY wo.WarehouseID
        HAVING AVG(wo.CycleTimeSeconds) > (
            SELECT MaxCapacityCycleTime FROM DimWarehouse WHERE WarehouseID = wo.WarehouseID
        )
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Warehouse cycle time exceeds capacity threshold';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL_RECIPIENTS}',
            @subject = 'LogiFleet Pulse - Warehouse Capacity Warning',
            @body = @AlertMessage;
    END;
END;
GO

-- Schedule via SQL Server Agent (run every 15 minutes)
EXEC msdb.dbo.sp_add_job @job_name = 'LogiFleet_KPI_Alerts';
EXEC msdb.dbo.sp_add_jobstep 
    @job_name = 'LogiFleet_KPI_Alerts',
    @step_name = 'Check_Thresholds',
    @command = 'EXEC sp_SendKPIAlerts';
EXEC msdb.dbo.sp_add_schedule 
    @schedule_name = 'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;
EXEC msdb.dbo.sp_attach_schedule @job_name = 'LogiFleet_KPI_Alerts', @schedule_name = 'Every15Minutes';
EXEC msdb.dbo.sp_start_job @job_name = 'LogiFleet_KPI_Alerts';
```

## Integration Patterns

### Connecting External APIs (Weather, Traffic)

```sql
-- Create external table for weather data (requires PolyBase/External Data Source)
CREATE EXTERNAL DATA SOURCE WeatherAPI
WITH (
    TYPE = BLOB_STORAGE,
    LOCATION = '${WEATHER_API_BLOB_ENDPOINT}',
    CREDENTIAL = WeatherAPICredential
);

CREATE EXTERNAL FILE FORMAT JSONFormat
WITH (FORMAT_TYPE = JSON);

CREATE EXTERNAL TABLE ExtWeatherData (
    LocationCode VARCHAR(20),
    Timestamp DATETIME2,
    TemperatureCelsius DECIMAL(5,2),
    PrecipitationMM DECIMAL(5,2),
    WindSpeedKPH DECIMAL(5,2)
)
WITH (
    LOCATION = '/weather/',
    DATA_SOURCE = WeatherAPI,
    FILE_FORMAT = JSONFormat
);

-- Join with fleet data to analyze weather impact
SELECT 
    ft.TripID,
    ft.DelayMinutes,
    wd.PrecipitationMM,
    wd.WindSpeedKPH,
    CASE 
        WHEN wd.PrecipitationMM > 10 THEN 'HEAVY_RAIN'
        WHEN wd.PrecipitationMM > 5 THEN 'RAIN'
        ELSE 'CLEAR'
    END AS WeatherCondition
FROM FactFleetTrips ft
INNER JOIN DimGeography g ON ft.OriginGeographyID = g.GeographyID
INNER JOIN ExtWeatherData wd ON g.LocationCode = wd.LocationCode
    AND DATEADD(MINUTE, (DATEDIFF(MINUTE, 0, wd.Timestamp) / 15) * 15, 0) = 
        (SELECT DateTimeValue FROM DimTime WHERE TimeID = ft.TimeID);
```

### WMS Integration via Stored Procedure

```sql
-- Sync from external WMS system
CREATE PROCEDURE sp_SyncFromWMS
AS
BEGIN
    -- Pull from linked server or staging table
    INSERT INTO StagingWarehouseOps (
        WarehouseCode, SKU, OperationType, Quantity, 
        StartTime, EndTime, EmployeeID
    )
    SELECT 
        w.LocationCode AS WarehouseCode,
        p.ItemNumber AS SKU,
        t.TransactionType AS OperationType,
        t.Quantity,
        t.TransactionDate AS StartTime,
        t.CompletionDate AS EndTime,
        t.UserID AS EmployeeID
    FROM [WMS_LinkedServer].WarehouseDB.dbo.Transactions t
    INNER JOIN [WMS_LinkedServer].WarehouseDB.dbo.Locations w ON t.LocationID = w.LocationID
    INNER JOIN [WMS_LinkedServer].WarehouseDB.dbo.Products p ON t.ProductID = p.ProductID
    WHERE t.TransactionDate >= DATEADD(MINUTE, -20, SYSUTCDATETIME())
    AND t.SyncedToAnalytics = 0;
    
    -- Mark as synced
    UPDATE [WMS_LinkedServer].WarehouseDB.dbo.Transactions
    SET SyncedToAnalytics = 1
    WHERE TransactionDate >= DATEADD(MINUTE, -20, SYSUTCDATETIME());
END;
GO
```

## Troubleshooting

### Slow Dashboard Refresh

**Symptom:** Power BI dashboards taking >2 minutes to refresh

**Solutions:**
1. Check columnstore index fragmentation:
```sql
SELECT 
    OBJECT_NAME(i.object_id) AS TableName,
    i.name AS IndexName,
    dmv.state_description,
    dmv.total_rows,
    dmv.deleted_rows,
    (dmv.deleted_rows * 100.0 / NULLIF(dmv.total_rows, 0)) AS FragmentationPercent
FROM sys.dm_db_column_store_row_group_physical_stats dmv
INNER JOIN sys.indexes i ON dmv.object_id = i.object_id AND dmv.index_id = i.index_id
WHERE OBJECT_NAME(i.object_id) IN ('FactWarehouseOperations', 'FactFleetTrips')
ORDER BY FragmentationPercent DESC;

-- Rebuild if >20% fragmentation
ALTER INDEX NCCI_WarehouseOps ON FactWarehouseOperations REBUILD;
```

2. Implement aggregation tables:
```sql
CREATE TABLE AggWarehouseDaily (
    Date DATE,
    WarehouseID INT,
    ProductID INT,
    TotalQuantity DECIMAL(18,3),
    AvgCycleTimeSeconds INT,
    TotalOperations INT
);

-- Populate via scheduled job
INSERT INTO AggWarehouseDaily
SELECT 
    CAST(t.DateTimeValue AS DATE) AS Date,
    wo.WarehouseID,
    wo.ProductID,
    SUM(wo.Quantity) AS TotalQuantity,
    AVG(wo.CycleTimeSeconds) AS AvgCycleTimeSeconds,
    COUNT(*) AS TotalOperations
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeID = t.TimeID
GROUP BY CAST(t.DateTimeValue AS DATE), wo.WarehouseID, wo.ProductID;
```

### Incorrect Time Grain

**Symptom:** Metrics not aligning to 15-minute buckets

**Solution:** Verify TimeID generation logic:
```sql
-- Correct time bucketing function
CREATE FUNCTION dbo.fn_GetTimeID (@InputDateTime DATETIME2)
RETURNS INT
AS
BEGIN
    DECLARE @Bucketed DATETIME2 = DATEADD(MINUTE, 
        (DATEDIFF(MINUTE, 0, @InputDateTime) / 15) * 15, 0);
    
    RETURN (SELECT TimeID FROM DimTime WHERE DateTimeValue = @Bucketed);
END;
GO

-- Test
SELECT dbo.fn_GetTimeID('2026-07-01 14:37:22'); -- Should bucket to 14:30
```

### Gravity Score Not Updating

**Symptom:** DimProductGravity.GravityScore remains static

**Solution:** Ensure recalculation job is running:
```sql
-- Check last run time
SELECT LastRecalculatedAt FROM DimProductGravity ORDER BY LastRecalculatedAt DESC;

-- Manually trigger
EXEC sp_RecalculateProductGravity;

-- Verify SQL Agent job schedule
SELECT 
    j.name AS JobName,
    s.name AS ScheduleName,
    s.enabled,
    CASE s.freq_type
        WHEN 1 THEN 'Once'
        WHEN 4 THEN 'Daily'
        WHEN 8 THEN 'Weekly'
    END AS Frequency
FROM msdb.dbo.sysjobs j
INNER JOIN msdb.dbo.sysjobschedules js ON j.job_id = js.job_id
INNER JOIN msdb.dbo.sysschedules s ON js.schedule_id = s.schedule_id
WHERE j.name = 'LogiFleet_GravityRecalc';
```

## Performance Optimization

### Partitioning Strategy for Large Fact Tables

```sql
-- Partition FactWarehouseOperations by month
CREATE PARTITION FUNCTION pf_MonthlyPartition (DATETIME2)
AS RANGE RIGHT FOR VALUES (
    '2026-01-01', '2026-02-01', '2026-03-01', '2026-04-01',
    '2026-05-01', '2026-06-01', '2026-07-01', '2026-08-01',
    '2026-09-01', '2026-10-01', '2026-11-01', '2026-12-01'
);

CREATE PARTITION SCHEME ps_MonthlyPartition
AS PARTITION pf_MonthlyPartition
ALL TO ([PRIMARY]);

-- Recreate fact table with partitioning
CREATE TABLE FactWarehouseOperations_Partitioned (
    OperationID BIGINT IDENTITY(1,1),
    TimeID INT,
    CreatedAt DATETIME2,
    -- ... other columns
    CONSTRAINT PK_WarehouseOps_Part PRIMARY KEY (OperationID, CreatedAt)
) ON ps_MonthlyPartition(CreatedAt);

-- Migrate data
INSERT INTO FactWarehouseOperations_Partitioned
SELECT * FROM FactWarehouseOperations;

-- Switch tables
EXEC sp_rename 'FactWarehouseOperations', 'FactWarehouseOperations_Old';
EXEC sp_rename 'FactWarehouseOperations_Partitioned', 'FactWarehouseOperations';
```

### Query Performance Tuning

```sql
-- Enable query store for pattern analysis
ALTER DATABASE LogiFleetPulse SET QUERY_STORE = ON;
ALTER DATABASE LogiFleetPulse SET QUERY_STORE (
    OPERATION_MODE = READ_WRITE,
    DATA_FLUSH_INTERVAL_SECONDS = 900,
    INTERVAL_LENGTH_MINUTES = 15
);

-- Identify slow queries
SELECT 
    qsq.query_id,
    qsqt.query_sql_text,
    qrs.avg_duration / 1000 AS avg_duration_ms,
    qrs.avg_logical_io_reads,
    qrs.count_executions
FROM sys.query_store_query qsq
INNER JOIN sys.query_store_query_text qsqt ON qsq.query_text_id = qsqt.query_text_id
INNER JOIN sys.query_store_plan qsp ON qsq.query_id = qsp.query_id
INNER JOIN sys.query_store_runtime_stats qrs ON qsp.plan_id = qrs.plan_id
WHERE qrs.avg_duration > 5000000 -- >5 seconds
ORDER BY qrs.avg_duration DESC;
```

## Best Practices

1. **Incremental Loading:** Always
