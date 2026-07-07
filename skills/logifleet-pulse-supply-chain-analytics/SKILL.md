---
name: logifleet-pulse-supply-chain-analytics
description: Multi-modal logistics intelligence platform with MS SQL Server data warehousing and Power BI dashboards for supply chain optimization
triggers:
  - set up supply chain analytics dashboard
  - implement logistics data warehouse
  - create fleet management power bi dashboard
  - build warehouse operations star schema
  - integrate logistics telemetry data
  - deploy logifleet pulse analytics
  - configure cross-modal supply chain reporting
  - optimize warehouse gravity zones
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

LogiFleet Pulse is an advanced logistics intelligence platform that unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a single semantic data model. Built on MS SQL Server with Power BI visualization, it provides real-time cross-fact analytics for supply chain optimization.

## What It Does

- **Multi-fact star schema** linking warehouse, fleet, and supplier data
- **Real-time dashboards** with 15-minute refresh granularity
- **Predictive bottleneck detection** using cross-fact KPI harmonization
- **Warehouse gravity zone optimization** based on pick frequency and item value
- **Fleet triage engine** prioritizing maintenance by revenue impact
- **Temporal elasticity modeling** for scenario simulation

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio

### Step 1: Clone Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### Step 2: Deploy SQL Schema

Connect to your SQL Server instance and run the schema deployment:

```sql
-- Create database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute schema scripts in order
:r .\sql\01_create_dimensions.sql
:r .\sql\02_create_facts.sql
:r .\sql\03_create_views.sql
:r .\sql\04_create_procedures.sql
:r .\sql\05_create_indexes.sql
GO
```

### Step 3: Configure Data Sources

Create a configuration file for connection strings:

```json
{
  "connections": {
    "sql_server": "Server=${SQL_SERVER};Database=LogiFleetPulse;Trusted_Connection=True;",
    "wms_api": "${WMS_API_ENDPOINT}",
    "fleet_telemetry_api": "${FLEET_API_ENDPOINT}",
    "weather_api_key": "${WEATHER_API_KEY}"
  },
  "refresh_interval_minutes": 15,
  "alert_thresholds": {
    "fleet_idle_percent": 15,
    "dwell_time_hours": 72,
    "temperature_variance_celsius": 2
  }
}
```

### Step 4: Import Power BI Template

1. Open Power BI Desktop
2. File → Import → Power BI Template
3. Select `LogiFleet_Pulse_Master.pbit`
4. Enter connection string when prompted
5. Configure row-level security for user roles

## Core Data Model

### Fact Tables

#### FactWarehouseOperations

```sql
CREATE TABLE FactWarehouseOperations (
    OperationID INT PRIMARY KEY IDENTITY(1,1),
    TimeID INT NOT NULL,
    WarehouseID INT NOT NULL,
    ProductID INT NOT NULL,
    OperationType VARCHAR(20), -- 'Putaway', 'Pick', 'Pack', 'Ship'
    DwellTimeMinutes INT,
    PickRateUnitsPerHour DECIMAL(10,2),
    GravityZoneID INT,
    FOREIGN KEY (TimeID) REFERENCES DimTime(TimeID),
    FOREIGN KEY (WarehouseID) REFERENCES DimGeography(GeographyID),
    FOREIGN KEY (ProductID) REFERENCES DimProductGravity(ProductID)
);

-- Partitioning for performance
CREATE PARTITION SCHEME PS_WarehouseOps
AS PARTITION PF_MonthlyRange
TO ([FG_2025Q1], [FG_2025Q2], [FG_2025Q3], [FG_2025Q4], [FG_2026Q1]);
```

#### FactFleetTrips

```sql
CREATE TABLE FactFleetTrips (
    TripID INT PRIMARY KEY IDENTITY(1,1),
    VehicleID INT NOT NULL,
    TimeStartID INT NOT NULL,
    TimeEndID INT NOT NULL,
    RouteID INT NOT NULL,
    OriginGeographyID INT NOT NULL,
    DestinationGeographyID INT NOT NULL,
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadWeightKg DECIMAL(12,2),
    MaintenanceScore INT, -- 0-100
    WeatherDelayMinutes INT,
    FOREIGN KEY (TimeStartID) REFERENCES DimTime(TimeID),
    FOREIGN KEY (TimeEndID) REFERENCES DimTime(TimeID),
    FOREIGN KEY (OriginGeographyID) REFERENCES DimGeography(GeographyID),
    FOREIGN KEY (DestinationGeographyID) REFERENCES DimGeography(GeographyID)
);
```

### Dimension Tables

#### DimTime (Time-Phased Dimension)

```sql
CREATE TABLE DimTime (
    TimeID INT PRIMARY KEY,
    DateTimeFull DATETIME NOT NULL,
    DateKey INT NOT NULL, -- YYYYMMDD
    FiscalYear INT,
    FiscalQuarter INT,
    FiscalMonth INT,
    DayOfWeek INT,
    HourOfDay INT,
    QuarterHourBucket INT, -- 0-95 (96 buckets per day)
    IsBusinessHour BIT,
    IsWeekend BIT
);

-- Populate time dimension
EXEC sp_PopulateTimeDimension 
    @StartDate = '2025-01-01', 
    @EndDate = '2027-12-31', 
    @GranularityMinutes = 15;
```

#### DimProductGravity (Warehouse Gravity Zones)

```sql
CREATE TABLE DimProductGravity (
    ProductID INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName NVARCHAR(200),
    CategoryL1 VARCHAR(100),
    CategoryL2 VARCHAR(100),
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value * fragility_weight
    PickFrequencyRank INT,
    UnitValue DECIMAL(10,2),
    FragilityMultiplier DECIMAL(3,2),
    OptimalZone VARCHAR(20), -- 'High', 'Medium', 'Low', 'Bulk'
    LastGravityCalculation DATETIME
);

-- Stored procedure to recalculate gravity scores
CREATE PROCEDURE sp_RecalculateProductGravity
AS
BEGIN
    UPDATE DimProductGravity
    SET GravityScore = (
        SELECT 
            (COUNT(*) * 0.4) + -- Pick frequency weight
            (AVG(UnitValue) * 0.4) + -- Value weight
            (AVG(FragilityMultiplier) * 0.2) -- Fragility weight
        FROM FactWarehouseOperations
        WHERE ProductID = DimProductGravity.ProductID
        AND TimeID >= (SELECT MAX(TimeID) - 8640 FROM DimTime) -- Last 90 days
    ),
    OptimalZone = CASE
        WHEN GravityScore > 75 THEN 'High'
        WHEN GravityScore > 50 THEN 'Medium'
        WHEN GravityScore > 25 THEN 'Low'
        ELSE 'Bulk'
    END,
    LastGravityCalculation = GETDATE();
END;
```

## Key Stored Procedures

### Cross-Fact KPI Query

```sql
CREATE PROCEDURE sp_GetCrossFactKPIs
    @StartDate DATE,
    @EndDate DATE,
    @WarehouseID INT = NULL
AS
BEGIN
    WITH WarehouseDwell AS (
        SELECT 
            w.WarehouseID,
            p.CategoryL1,
            AVG(w.DwellTimeMinutes) AS AvgDwellMinutes,
            SUM(CASE WHEN w.DwellTimeMinutes > 4320 THEN 1 ELSE 0 END) AS ItemsOver72Hours
        FROM FactWarehouseOperations w
        INNER JOIN DimTime t ON w.TimeID = t.TimeID
        INNER JOIN DimProductGravity p ON w.ProductID = p.ProductID
        WHERE t.DateKey BETWEEN CONVERT(INT, FORMAT(@StartDate, 'yyyyMMdd'))
                            AND CONVERT(INT, FORMAT(@EndDate, 'yyyyMMdd'))
            AND (@WarehouseID IS NULL OR w.WarehouseID = @WarehouseID)
        GROUP BY w.WarehouseID, p.CategoryL1
    ),
    FleetPerformance AS (
        SELECT
            f.OriginGeographyID AS WarehouseID,
            AVG(f.IdleTimeMinutes * 1.0 / DATEDIFF(MINUTE, ts.DateTimeFull, te.DateTimeFull)) AS IdlePercent,
            AVG(f.FuelConsumedLiters) AS AvgFuelPerTrip
        FROM FactFleetTrips f
        INNER JOIN DimTime ts ON f.TimeStartID = ts.TimeID
        INNER JOIN DimTime te ON f.TimeEndID = te.TimeID
        WHERE ts.DateKey BETWEEN CONVERT(INT, FORMAT(@StartDate, 'yyyyMMdd'))
                              AND CONVERT(INT, FORMAT(@EndDate, 'yyyyMMdd'))
            AND (@WarehouseID IS NULL OR f.OriginGeographyID = @WarehouseID)
        GROUP BY f.OriginGeographyID
    )
    SELECT 
        wd.WarehouseID,
        g.WarehouseName,
        wd.CategoryL1,
        wd.AvgDwellMinutes,
        wd.ItemsOver72Hours,
        fp.IdlePercent * 100 AS IdlePercentage,
        fp.AvgFuelPerTrip,
        -- Cross-fact composite metric
        (wd.AvgDwellMinutes / 60.0) * fp.IdlePercent * fp.AvgFuelPerTrip AS LogisticsFrictionIndex
    FROM WarehouseDwell wd
    INNER JOIN FleetPerformance fp ON wd.WarehouseID = fp.WarehouseID
    INNER JOIN DimGeography g ON wd.WarehouseID = g.GeographyID
    ORDER BY LogisticsFrictionIndex DESC;
END;
```

### Predictive Bottleneck Detection

```sql
CREATE PROCEDURE sp_PredictBottlenecks
    @ForecastHours INT = 24
AS
BEGIN
    -- Historical pattern analysis
    WITH HistoricalPatterns AS (
        SELECT
            t.DayOfWeek,
            t.HourOfDay,
            w.WarehouseID,
            AVG(w.DwellTimeMinutes) AS AvgHistoricalDwell,
            STDEV(w.DwellTimeMinutes) AS StdDevDwell
        FROM FactWarehouseOperations w
        INNER JOIN DimTime t ON w.TimeID = t.TimeID
        WHERE t.DateTimeFull >= DATEADD(DAY, -90, GETDATE())
        GROUP BY t.DayOfWeek, t.HourOfDay, w.WarehouseID
    ),
    FutureTimeSlots AS (
        SELECT 
            TimeID,
            DayOfWeek,
            HourOfDay,
            DateTimeFull
        FROM DimTime
        WHERE DateTimeFull BETWEEN GETDATE() 
                               AND DATEADD(HOUR, @ForecastHours, GETDATE())
    )
    SELECT 
        fts.DateTimeFull AS PredictedTime,
        hp.WarehouseID,
        g.WarehouseName,
        hp.AvgHistoricalDwell AS ExpectedDwellMinutes,
        hp.StdDevDwell,
        -- Bottleneck probability score (0-100)
        CASE 
            WHEN hp.AvgHistoricalDwell > 4320 THEN 95
            WHEN hp.AvgHistoricalDwell > 2880 THEN 75
            WHEN hp.AvgHistoricalDwell > 1440 THEN 50
            ELSE 20
        END AS BottleneckProbability,
        -- Recommended action
        CASE 
            WHEN hp.AvgHistoricalDwell > 4320 THEN 'URGENT: Reallocate to high-gravity zone'
            WHEN hp.AvgHistoricalDwell > 2880 THEN 'WARNING: Monitor pick rate closely'
            ELSE 'Normal operations expected'
        END AS RecommendedAction
    FROM FutureTimeSlots fts
    INNER JOIN HistoricalPatterns hp 
        ON fts.DayOfWeek = hp.DayOfWeek 
        AND fts.HourOfDay = hp.HourOfDay
    INNER JOIN DimGeography g ON hp.WarehouseID = g.GeographyID
    WHERE hp.AvgHistoricalDwell > 1440 -- Only show potential issues
    ORDER BY BottleneckProbability DESC, fts.DateTimeFull;
END;
```

## Power BI DAX Measures

### Fleet Efficiency Composite

```dax
Fleet Efficiency Index = 
VAR TotalTripTime = 
    SUMX(
        FactFleetTrips,
        DATEDIFF(
            RELATED(DimTime[TimeStart][DateTimeFull]),
            RELATED(DimTime[TimeEnd][DateTimeFull]),
            MINUTE
        )
    )
VAR TotalIdleTime = SUM(FactFleetTrips[IdleTimeMinutes])
VAR IdlePercent = DIVIDE(TotalIdleTime, TotalTripTime, 0)
VAR AvgFuelEfficiency = DIVIDE(
    SUM(FactFleetTrips[FuelConsumedLiters]),
    SUM(FactFleetTrips[LoadWeightKg]) / 1000,
    0
)
RETURN
    (1 - IdlePercent) * 0.6 + 
    (1 / (1 + AvgFuelEfficiency)) * 0.4
```

### Warehouse Gravity Compliance

```dax
Gravity Zone Compliance % = 
VAR ItemsInOptimalZone = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[CurrentZone] = RELATED(DimProductGravity[OptimalZone])
    )
VAR TotalItems = COUNTROWS(FactWarehouseOperations)
RETURN
    DIVIDE(ItemsInOptimalZone, TotalItems, 0) * 100
```

### Cross-Fact Temporal Elasticity

```dax
Logistics Friction Index = 
VAR WarehouseDwell = 
    AVERAGEX(
        FactWarehouseOperations,
        FactWarehouseOperations[DwellTimeMinutes] / 60
    )
VAR FleetIdle = 
    AVERAGEX(
        FactFleetTrips,
        FactFleetTrips[IdleTimeMinutes] / 
        DATEDIFF(
            RELATED(DimTime[TimeStart][DateTimeFull]),
            RELATED(DimTime[TimeEnd][DateTimeFull]),
            MINUTE
        )
    )
VAR FuelCost = AVERAGE(FactFleetTrips[FuelConsumedLiters]) * 1.5 // Assume $1.5/L
RETURN
    WarehouseDwell * FleetIdle * FuelCost
```

## Data Loading Patterns

### Incremental Load with Change Tracking

```sql
CREATE PROCEDURE sp_IncrementalLoadWarehouseOps
    @LastLoadDateTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Enable change tracking on source table
    -- ALTER DATABASE WMS_Source SET CHANGE_TRACKING = ON;
    -- ALTER TABLE WMS_Source.dbo.Operations ENABLE CHANGE_TRACKING;
    
    MERGE FactWarehouseOperations AS target
    USING (
        SELECT 
            op.OperationID,
            dt.TimeID,
            op.WarehouseID,
            op.ProductID,
            op.OperationType,
            DATEDIFF(MINUTE, op.StartTime, op.EndTime) AS DwellTimeMinutes,
            op.PickRate AS PickRateUnitsPerHour,
            pg.GravityZoneID
        FROM WMS_Source.dbo.Operations op
        INNER JOIN DimTime dt ON DATEPART(HOUR, op.StartTime) = dt.HourOfDay
            AND DATEPART(MINUTE, op.StartTime) / 15 = dt.QuarterHourBucket / 4
            AND CAST(op.StartTime AS DATE) = CAST(dt.DateTimeFull AS DATE)
        INNER JOIN DimProductGravity pg ON op.SKU = pg.SKU
        WHERE op.LastModified > @LastLoadDateTime
    ) AS source
    ON target.OperationID = source.OperationID
    WHEN MATCHED THEN
        UPDATE SET 
            target.DwellTimeMinutes = source.DwellTimeMinutes,
            target.PickRateUnitsPerHour = source.PickRateUnitsPerHour
    WHEN NOT MATCHED THEN
        INSERT (TimeID, WarehouseID, ProductID, OperationType, 
                DwellTimeMinutes, PickRateUnitsPerHour, GravityZoneID)
        VALUES (source.TimeID, source.WarehouseID, source.ProductID, 
                source.OperationType, source.DwellTimeMinutes, 
                source.PickRateUnitsPerHour, source.GravityZoneID);
    
    -- Update last load timestamp
    UPDATE ETL_Metadata
    SET LastLoadDateTime = GETDATE()
    WHERE TableName = 'FactWarehouseOperations';
END;
```

### External API Integration (Fleet Telemetry)

```sql
CREATE PROCEDURE sp_LoadFleetTelemetry
AS
BEGIN
    -- Assumes external table configured via PolyBase or Linked Server
    INSERT INTO FactFleetTrips (
        VehicleID, TimeStartID, TimeEndID, RouteID,
        OriginGeographyID, DestinationGeographyID,
        FuelConsumedLiters, IdleTimeMinutes, LoadWeightKg,
        MaintenanceScore, WeatherDelayMinutes
    )
    SELECT 
        v.VehicleID,
        ts.TimeID AS TimeStartID,
        te.TimeID AS TimeEndID,
        r.RouteID,
        og.GeographyID AS OriginGeographyID,
        dg.GeographyID AS DestinationGeographyID,
        t.FuelUsed AS FuelConsumedLiters,
        t.IdleTime AS IdleTimeMinutes,
        t.LoadWeight AS LoadWeightKg,
        v.MaintenanceScore,
        COALESCE(w.DelayMinutes, 0) AS WeatherDelayMinutes
    FROM OPENQUERY(
        FLEET_API_LINKEDSERVER,
        'SELECT * FROM telemetry WHERE timestamp > DATEADD(MINUTE, -15, GETDATE())'
    ) AS t
    INNER JOIN DimVehicle v ON t.vehicle_id = v.ExternalVehicleID
    INNER JOIN DimRoute r ON t.route_code = r.RouteCode
    INNER JOIN DimTime ts ON ts.DateTimeFull = t.start_timestamp
    INNER JOIN DimTime te ON te.DateTimeFull = t.end_timestamp
    INNER JOIN DimGeography og ON t.origin_lat = og.Latitude AND t.origin_lon = og.Longitude
    INNER JOIN DimGeography dg ON t.dest_lat = dg.Latitude AND t.dest_lon = dg.Longitude
    LEFT JOIN WeatherDelays w ON t.route_code = w.RouteCode 
        AND t.start_timestamp BETWEEN w.StartTime AND w.EndTime
    WHERE NOT EXISTS (
        SELECT 1 FROM FactFleetTrips ft
        WHERE ft.VehicleID = v.VehicleID
        AND ft.TimeStartID = ts.TimeID
    );
END;
```

## Automated Alerting

### Critical Threshold Monitoring

```sql
CREATE PROCEDURE sp_CheckAlertsAndNotify
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    DECLARE @Recipients VARCHAR(500) = '${ALERT_EMAIL_LIST}';
    
    -- Fleet idle time breach
    IF EXISTS (
        SELECT 1 
        FROM FactFleetTrips f
        INNER JOIN DimTime ts ON f.TimeStartID = ts.TimeID
        WHERE ts.DateTimeFull >= DATEADD(HOUR, -1, GETDATE())
        AND (f.IdleTimeMinutes * 1.0 / 
             DATEDIFF(MINUTE, ts.DateTimeFull, 
                      (SELECT DateTimeFull FROM DimTime WHERE TimeID = f.TimeEndID))
            ) > 0.15
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Fleet idle time exceeded 15% in the last hour. Review routing optimization.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = @Recipients,
            @subject = 'LogiFleet Pulse: Fleet Idle Alert',
            @body = @AlertMessage,
            @importance = 'High';
    END;
    
    -- Warehouse dwell time breach
    IF EXISTS (
        SELECT 1
        FROM FactWarehouseOperations w
        INNER JOIN DimTime t ON w.TimeID = t.TimeID
        WHERE t.DateTimeFull >= DATEADD(HOUR, -4, GETDATE())
        AND w.DwellTimeMinutes > 4320
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Items with >72hr dwell time detected. Inventory rebalancing recommended.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = @Recipients,
            @subject = 'LogiFleet Pulse: Warehouse Dwell Alert',
            @body = @AlertMessage,
            @importance = 'High';
    END;
END;
```

### SQL Agent Job Configuration

```sql
-- Schedule alerting job every 15 minutes
USE msdb;
GO

EXEC sp_add_job
    @job_name = 'LogiFleet_Alert_Monitor',
    @enabled = 1,
    @description = 'Monitor KPI thresholds and send alerts';

EXEC sp_add_jobstep
    @job_name = 'LogiFleet_Alert_Monitor',
    @step_name = 'Check Alerts',
    @subsystem = 'TSQL',
    @database_name = 'LogiFleetPulse',
    @command = 'EXEC sp_CheckAlertsAndNotify;';

EXEC sp_add_schedule
    @schedule_name = 'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;

EXEC sp_attach_schedule
    @job_name = 'LogiFleet_Alert_Monitor',
    @schedule_name = 'Every15Minutes';

EXEC sp_add_jobserver
    @job_name = 'LogiFleet_Alert_Monitor';
```

## Configuration Best Practices

### Indexing Strategy

```sql
-- Covering index for cross-fact queries
CREATE NONCLUSTERED INDEX IX_WarehouseOps_CrossFact
ON FactWarehouseOperations (WarehouseID, ProductID, TimeID)
INCLUDE (DwellTimeMinutes, PickRateUnitsPerHour)
WITH (FILLFACTOR = 90, PAD_INDEX = ON);

-- Filtered index for slow-moving items
CREATE NONCLUSTERED INDEX IX_WarehouseOps_SlowMovers
ON FactWarehouseOperations (ProductID, TimeID)
WHERE DwellTimeMinutes > 4320;

-- Columnstore for analytical queries
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FleetTrips_Analytics
ON FactFleetTrips (VehicleID, TimeStartID, TimeEndID, FuelConsumedLiters, IdleTimeMinutes);
```

### Row-Level Security

```sql
-- Create security policy for regional access
CREATE FUNCTION fn_SecurityPredicateWarehouse(@WarehouseID INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS AccessAllowed
WHERE @WarehouseID IN (
    SELECT w.WarehouseID 
    FROM dbo.DimGeography w
    INNER JOIN dbo.UserWarehouseAccess u ON w.RegionID = u.RegionID
    WHERE u.Username = USER_NAME()
);
GO

CREATE SECURITY POLICY WarehouseSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicateWarehouse(WarehouseID)
ON dbo.FactWarehouseOperations,
ADD FILTER PREDICATE dbo.fn_SecurityPredicateWarehouse(OriginGeographyID)
ON dbo.FactFleetTrips
WITH (STATE = ON);
```

## Troubleshooting

### Performance Issues

**Symptom**: Cross-fact queries taking >10 seconds

```sql
-- Check query execution plan
SET STATISTICS TIME ON;
SET STATISTICS IO ON;

EXEC sp_GetCrossFactKPIs 
    @StartDate = '2026-01-01', 
    @EndDate = '2026-01-31';

-- Update statistics if outdated
UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS FactFleetTrips WITH FULLSCAN;

-- Consider table partitioning
ALTER TABLE FactWarehouseOperations 
SWITCH PARTITION 1 TO ArchiveFactWarehouseOperations PARTITION 1;
```

### Power BI Refresh Failures

**Symptom**: "Gateway timeout" errors

```sql
-- Optimize views used by Power BI
CREATE VIEW vw_WarehouseKPIs_Optimized
WITH SCHEMABINDING
AS
SELECT 
    w.WarehouseID,
    t.DateKey,
    COUNT_BIG(*) AS OperationCount,
    AVG(w.DwellTimeMinutes) AS AvgDwell,
    SUM(CASE WHEN w.OperationType = 'Pick' THEN 1 ELSE 0 END) AS PickCount
FROM dbo.FactWarehouseOperations w
INNER JOIN dbo.DimTime t ON w.TimeID = t.TimeID
GROUP BY w.WarehouseID, t.DateKey;

-- Create indexed view
CREATE UNIQUE CLUSTERED INDEX IX_WarehouseKPIs_Optimized
ON vw_WarehouseKPIs_Optimized (WarehouseID, DateKey);
```

### Data Quality Issues

**Symptom**: Null or inconsistent gravity scores

```sql
-- Validate product gravity assignments
SELECT 
    p.ProductID,
    p.SKU,
    p.GravityScore,
    p.OptimalZone,
    COUNT(w.OperationID) AS RecentOperations,
    AVG(w.DwellTimeMinutes) AS ActualDwell
FROM DimProductGravity p
LEFT JOIN FactWarehouseOperations w ON p.ProductID = w.ProductID
    AND w.TimeID >= (SELECT MAX(TimeID) - 8640 FROM DimTime)
WHERE p.GravityScore IS NULL OR p.LastGravityCalculation < DATEADD(DAY, -7, GETDATE())
GROUP BY p.ProductID, p.SKU, p.GravityScore, p.OptimalZone;

-- Recalculate missing scores
EXEC sp_RecalculateProductGravity;
```

## Common Usage Patterns

### Daily Operations Dashboard Query

```sql
-- Executive summary for today
DECLARE @Today DATE = CAST(GETDATE() AS DATE);

SELECT 
    'Warehouse Operations' AS MetricCategory,
    COUNT(DISTINCT w.WarehouseID) AS ActiveLocations,
    COUNT(*) AS TotalOperations,
    AVG(w.DwellTimeMinutes) / 60.0 AS AvgDwellHours,
    SUM(CASE WHEN w.DwellTimeMinutes > 4320 THEN 1 ELSE 0 END) AS CriticalItems
FROM FactWarehouseOperations w
INNER JOIN DimTime t ON w.TimeID = t.TimeID
WHERE CAST(t.DateTimeFull AS DATE) = @Today

UNION ALL

SELECT 
    'Fleet Performance',
    COUNT(DISTINCT f.VehicleID),
    COUNT(*),
    AVG(f.IdleTimeMinutes) / 60.0,
    SUM(CASE WHEN f.MaintenanceScore < 50 THEN 1 ELSE 0 END)
FROM FactFleetTrips f
INNER JOIN DimTime ts ON f.TimeStartID = ts.TimeID
WHERE CAST(ts.DateTimeFull AS DATE) = @Today;
```

### Scenario Simulation

```sql
-- What-if analysis: Impact of increasing warehouse capacity by 20%
WITH BaselineMetrics AS (
    SELECT 
        AVG(DwellTimeMinutes) AS CurrentDwell,
        COUNT(*) AS CurrentVolume
    FROM FactWarehouseOperations
    WHERE TimeID >= (SELECT MAX(TimeID) - 8640 FROM DimTime)
)
SELECT 
    CurrentDwell,
    CurrentVolume,
    CurrentDwell * 0.8 AS ProjectedDwell_20PctIncrease,
    CurrentVolume * 1.2 AS ProjectedVolume_20PctIncrease,
    (CurrentDwell * 0.8) / CurrentDwell AS EfficiencyGain
FROM BaselineMetrics;
```

This skill provides AI agents with the complete technical foundation to implement, configure, and troubleshoot LogiFleet Pulse supply chain analytics platform.
