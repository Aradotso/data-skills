---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing platform for logistics, fleet management, and warehouse optimization with multi-fact star schema modeling
triggers:
  - set up logistics analytics dashboard
  - configure supply chain data warehouse
  - implement fleet tracking data model
  - build warehouse operations reporting
  - create multi-fact star schema for logistics
  - deploy Power BI logistics intelligence platform
  - optimize warehouse gravity zones
  - integrate fleet telemetry with SQL Server
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is an advanced data warehousing and analytics platform that combines MS SQL Server backend with Power BI visualization for comprehensive supply chain, fleet, and warehouse management intelligence. It implements a multi-fact star schema architecture that harmonizes warehouse operations, fleet telemetry, inventory aging, and external data sources into a unified semantic layer.

**Key Capabilities:**
- Multi-fact star schema with time-phased dimensions
- Cross-fact KPI harmonization (warehouse + fleet metrics)
- Real-time operational dashboards (15-minute refresh)
- Predictive bottleneck detection
- Warehouse gravity zone optimization
- Fleet maintenance triage engine
- Role-based access with row-level security

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise Edition)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to data sources: WMS, TMS, telematics APIs, ERP

### Database Deployment

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**
```sql
-- Connect to your SQL Server instance
-- Create the database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Execute the schema deployment script
-- Assumes schema.sql is in the repo root
:r schema.sql
GO

-- Verify table creation
SELECT TABLE_SCHEMA, TABLE_NAME 
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_TYPE = 'BASE TABLE'
ORDER BY TABLE_SCHEMA, TABLE_NAME
```

3. **Configure connection strings:**
```json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_USERNAME}",
    "password": "${SQL_PASSWORD}",
    "trusted_connection": false
  },
  "data_sources": {
    "wms_api": "${WMS_API_ENDPOINT}",
    "fleet_telemetry": "${FLEET_TELEMETRY_ENDPOINT}",
    "weather_api": "${WEATHER_API_ENDPOINT}"
  }
}
```

## Core Data Model Architecture

### Fact Tables

**FactWarehouseOperations** - Tracks all warehouse activities
```sql
CREATE TABLE FactWarehouseOperations (
    OperationID INT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    QuantityProcessed DECIMAL(18,2),
    DwellTimeMinutes INT,
    StorageZoneKey INT,
    EmployeeKey INT,
    ProcessingTimeMinutes DECIMAL(10,2),
    QualityScore DECIMAL(5,2),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
)
GO

-- Create clustered columnstore index for optimal performance
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactWarehouseOperations 
ON FactWarehouseOperations
GO
```

**FactFleetTrips** - Fleet and route performance
```sql
CREATE TABLE FactFleetTrips (
    TripID INT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT NOT NULL,
    TripStartTime DATETIME2,
    TripEndTime DATETIME2,
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadWeightKG DECIMAL(10,2),
    DeliveryOnTime BIT,
    WeatherConditionKey INT,
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey)
)
GO

CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactFleetTrips 
ON FactFleetTrips
GO
```

**FactCrossDock** - Direct transfer operations
```sql
CREATE TABLE FactCrossDock (
    CrossDockID INT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    InboundTripID INT,
    OutboundTripID INT,
    ProductKey INT NOT NULL,
    Quantity DECIMAL(18,2),
    TransferTimeMinutes INT,
    DockBayKey INT,
    TemperatureCompliance BIT,
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
)
GO
```

### Dimension Tables

**DimTime** - Time-phased dimension with 15-minute granularity
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2,
    Year INT,
    Quarter INT,
    Month INT,
    DayOfMonth INT,
    DayOfWeek INT,
    DayName VARCHAR(10),
    HourOfDay INT,
    MinuteBucket INT, -- 0, 15, 30, 45
    IsWeekend BIT,
    IsHoliday BIT,
    FiscalPeriod INT,
    FiscalYear INT
)
GO

-- Populate DimTime with procedure
CREATE PROCEDURE sp_PopulateDimTime
    @StartDate DATETIME2,
    @EndDate DATETIME2
AS
BEGIN
    DECLARE @CurrentDate DATETIME2 = @StartDate
    
    WHILE @CurrentDate <= @EndDate
    BEGIN
        -- 15-minute intervals
        DECLARE @Minute INT = 0
        WHILE @Minute < 60
        BEGIN
            INSERT INTO DimTime (TimeKey, FullDateTime, Year, Quarter, Month, DayOfMonth, 
                                 DayOfWeek, DayName, HourOfDay, MinuteBucket, IsWeekend)
            VALUES (
                CAST(FORMAT(@CurrentDate, 'yyyyMMddHHmm') AS INT),
                DATEADD(MINUTE, @Minute, @CurrentDate),
                YEAR(@CurrentDate),
                DATEPART(QUARTER, @CurrentDate),
                MONTH(@CurrentDate),
                DAY(@CurrentDate),
                DATEPART(WEEKDAY, @CurrentDate),
                DATENAME(WEEKDAY, @CurrentDate),
                DATEPART(HOUR, @CurrentDate),
                @Minute,
                CASE WHEN DATEPART(WEEKDAY, @CurrentDate) IN (1,7) THEN 1 ELSE 0 END
            )
            
            SET @Minute = @Minute + 15
        END
        
        SET @CurrentDate = DATEADD(DAY, 1, @CurrentDate)
    END
END
GO
```

**DimProductGravity** - Product classification with gravity scoring
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL,
    ProductName VARCHAR(255),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    VelocityClass VARCHAR(20), -- 'Fast', 'Medium', 'Slow'
    ValueClass VARCHAR(20), -- 'High', 'Medium', 'Low'
    FragilityScore DECIMAL(5,2), -- 0-10 scale
    GravityScore DECIMAL(8,4), -- Computed composite score
    RecommendedZoneType VARCHAR(50),
    WeightKG DECIMAL(10,2),
    DimensionsCM VARCHAR(50),
    ExpirationSensitive BIT,
    TemperatureControlled BIT
)
GO

-- Stored procedure to calculate gravity scores
CREATE PROCEDURE sp_UpdateProductGravityScores
AS
BEGIN
    UPDATE DimProductGravity
    SET GravityScore = (
        -- Weighted formula combining velocity, value, and fragility
        (CASE VelocityClass 
            WHEN 'Fast' THEN 10.0 
            WHEN 'Medium' THEN 5.0 
            WHEN 'Slow' THEN 1.0 
            ELSE 0 END) * 0.5 +
        (CASE ValueClass 
            WHEN 'High' THEN 10.0 
            WHEN 'Medium' THEN 5.0 
            WHEN 'Low' THEN 1.0 
            ELSE 0 END) * 0.3 +
        FragilityScore * 0.2
    )
END
GO
```

## Key Analytical Queries

### Cross-Fact KPI: Fleet Efficiency vs. Warehouse Dwell Time
```sql
-- Analyze correlation between warehouse dwell time and fleet idle time
CREATE VIEW vw_CrossFactEfficiency AS
SELECT 
    t.Year,
    t.Month,
    w.WarehouseCode,
    p.Category,
    AVG(wh.DwellTimeMinutes) AS AvgWarehouseDwellTime,
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdleTime,
    SUM(wh.QuantityProcessed) AS TotalUnitsProcessed,
    SUM(ft.FuelConsumedLiters) AS TotalFuelConsumed,
    -- Efficiency ratio
    CASE 
        WHEN AVG(ft.IdleTimeMinutes) > 0 
        THEN SUM(wh.QuantityProcessed) / AVG(ft.IdleTimeMinutes)
        ELSE 0 
    END AS EfficiencyRatio
FROM FactWarehouseOperations wh
INNER JOIN DimTime t ON wh.TimeKey = t.TimeKey
INNER JOIN DimWarehouse w ON wh.WarehouseKey = w.WarehouseKey
INNER JOIN DimProduct p ON wh.ProductKey = p.ProductKey
LEFT JOIN FactFleetTrips ft ON t.TimeKey = ft.TimeKey AND w.GeographyKey = ft.OriginGeographyKey
WHERE wh.OperationType = 'Shipping'
GROUP BY t.Year, t.Month, w.WarehouseCode, p.Category
GO
```

### Predictive Bottleneck Detection
```sql
-- Identify potential bottlenecks by analyzing historical patterns
CREATE PROCEDURE sp_DetectBottlenecks
    @ThresholdPercentile DECIMAL(5,2) = 90.0
AS
BEGIN
    WITH HistoricalPerformance AS (
        SELECT 
            WarehouseKey,
            OperationType,
            AVG(ProcessingTimeMinutes) AS AvgTime,
            STDEV(ProcessingTimeMinutes) AS StdDevTime,
            PERCENTILE_CONT(@ThresholdPercentile/100.0) 
                WITHIN GROUP (ORDER BY ProcessingTimeMinutes) 
                OVER (PARTITION BY WarehouseKey, OperationType) AS PercentileTime
        FROM FactWarehouseOperations
        WHERE TimeKey >= CAST(FORMAT(DATEADD(DAY, -90, GETDATE()), 'yyyyMMdd') AS INT)
        GROUP BY WarehouseKey, OperationType
    ),
    RecentPerformance AS (
        SELECT 
            WarehouseKey,
            OperationType,
            AVG(ProcessingTimeMinutes) AS RecentAvgTime
        FROM FactWarehouseOperations
        WHERE TimeKey >= CAST(FORMAT(DATEADD(DAY, -7, GETDATE()), 'yyyyMMdd') AS INT)
        GROUP BY WarehouseKey, OperationType
    )
    SELECT 
        w.WarehouseName,
        hp.OperationType,
        hp.AvgTime AS HistoricalAverage,
        rp.RecentAvgTime,
        hp.PercentileTime AS BottleneckThreshold,
        CASE 
            WHEN rp.RecentAvgTime > hp.PercentileTime 
            THEN 'ALERT: Bottleneck Detected'
            WHEN rp.RecentAvgTime > hp.AvgTime + hp.StdDevTime 
            THEN 'WARNING: Performance Degrading'
            ELSE 'OK'
        END AS Status
    FROM HistoricalPerformance hp
    INNER JOIN RecentPerformance rp 
        ON hp.WarehouseKey = rp.WarehouseKey 
        AND hp.OperationType = rp.OperationType
    INNER JOIN DimWarehouse w ON hp.WarehouseKey = w.WarehouseKey
    WHERE rp.RecentAvgTime > hp.AvgTime
    ORDER BY rp.RecentAvgTime - hp.AvgTime DESC
END
GO
```

### Warehouse Gravity Zone Optimization
```sql
-- Recommend zone reassignments based on product gravity scores
CREATE PROCEDURE sp_OptimizeGravityZones
AS
BEGIN
    WITH ProductActivity AS (
        SELECT 
            p.ProductKey,
            p.GravityScore,
            sz.ZoneType AS CurrentZone,
            COUNT(*) AS PickFrequency,
            AVG(wh.ProcessingTimeMinutes) AS AvgProcessingTime
        FROM FactWarehouseOperations wh
        INNER JOIN DimProductGravity p ON wh.ProductKey = p.ProductKey
        INNER JOIN DimStorageZone sz ON wh.StorageZoneKey = sz.ZoneKey
        WHERE wh.OperationType = 'Picking'
          AND wh.TimeKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMdd') AS INT)
        GROUP BY p.ProductKey, p.GravityScore, sz.ZoneType
    ),
    ZoneRecommendations AS (
        SELECT 
            ProductKey,
            GravityScore,
            CurrentZone,
            PickFrequency,
            AvgProcessingTime,
            CASE 
                WHEN GravityScore > 8.0 THEN 'High-Gravity-Zone'
                WHEN GravityScore > 5.0 THEN 'Medium-Gravity-Zone'
                ELSE 'Low-Gravity-Zone'
            END AS RecommendedZone
        FROM ProductActivity
    )
    SELECT 
        p.SKU,
        p.ProductName,
        zr.CurrentZone,
        zr.RecommendedZone,
        zr.GravityScore,
        zr.PickFrequency,
        zr.AvgProcessingTime,
        CASE 
            WHEN zr.CurrentZone != zr.RecommendedZone 
            THEN 'RECOMMEND RELOCATION'
            ELSE 'Optimal Placement'
        END AS Action
    FROM ZoneRecommendations zr
    INNER JOIN DimProductGravity p ON zr.ProductKey = p.ProductKey
    WHERE zr.CurrentZone != zr.RecommendedZone
    ORDER BY zr.GravityScore DESC, zr.PickFrequency DESC
END
GO
```

## Data Loading Patterns

### Incremental ETL for Warehouse Operations
```sql
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadTimeKey INT
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Extract from source WMS (assumes linked server or staging table)
    INSERT INTO FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductKey, OperationType, 
        QuantityProcessed, DwellTimeMinutes, StorageZoneKey, 
        EmployeeKey, ProcessingTimeMinutes, QualityScore
    )
    SELECT 
        CAST(FORMAT(s.OperationDateTime, 'yyyyMMddHHmm') AS INT) AS TimeKey,
        w.WarehouseKey,
        p.ProductKey,
        s.OperationType,
        s.Quantity,
        DATEDIFF(MINUTE, s.ArrivalDateTime, s.ProcessedDateTime) AS DwellTimeMinutes,
        sz.ZoneKey,
        e.EmployeeKey,
        s.ProcessingTimeMinutes,
        s.QualityScore
    FROM StagingWarehouseOperations s
    INNER JOIN DimWarehouse w ON s.WarehouseCode = w.WarehouseCode
    INNER JOIN DimProductGravity p ON s.SKU = p.SKU
    LEFT JOIN DimStorageZone sz ON s.ZoneCode = sz.ZoneCode
    LEFT JOIN DimEmployee e ON s.EmployeeID = e.EmployeeID
    WHERE CAST(FORMAT(s.OperationDateTime, 'yyyyMMddHHmm') AS INT) > @LastLoadTimeKey
    
    -- Log the load
    INSERT INTO ETLLog (ProcedureName, RecordsProcessed, LoadDateTime)
    VALUES ('sp_LoadWarehouseOperations', @@ROWCOUNT, GETDATE())
END
GO
```

### Fleet Telemetry Integration
```sql
CREATE PROCEDURE sp_LoadFleetTelemetry
    @BatchDateTime DATETIME2
AS
BEGIN
    -- Assumes external table or API staging layer
    MERGE INTO FactFleetTrips AS target
    USING (
        SELECT 
            CAST(FORMAT(TripStartTime, 'yyyyMMddHHmm') AS INT) AS TimeKey,
            v.VehicleKey,
            r.RouteKey,
            d.DriverKey,
            TripStartTime,
            TripEndTime,
            DistanceKM,
            FuelConsumed,
            IdleTimeMinutes,
            LoadWeight,
            CASE WHEN ActualArrival <= ScheduledArrival THEN 1 ELSE 0 END AS OnTime,
            wc.WeatherConditionKey
        FROM StagingFleetTelemetry s
        INNER JOIN DimVehicle v ON s.VehicleID = v.VehicleID
        INNER JOIN DimRoute r ON s.RouteID = r.RouteID
        INNER JOIN DimDriver d ON s.DriverID = d.DriverID
        LEFT JOIN DimWeatherCondition wc ON s.WeatherCode = wc.WeatherCode
        WHERE s.BatchDateTime = @BatchDateTime
    ) AS source
    ON target.VehicleKey = source.VehicleKey 
        AND target.TripStartTime = source.TripStartTime
    WHEN NOT MATCHED THEN
        INSERT (TimeKey, VehicleKey, RouteKey, DriverKey, TripStartTime, 
                TripEndTime, DistanceKM, FuelConsumedLiters, IdleTimeMinutes, 
                LoadWeightKG, DeliveryOnTime, WeatherConditionKey)
        VALUES (source.TimeKey, source.VehicleKey, source.RouteKey, 
                source.DriverKey, source.TripStartTime, source.TripEndTime, 
                source.DistanceKM, source.FuelConsumed, source.IdleTimeMinutes, 
                source.LoadWeight, source.OnTime, source.WeatherConditionKey);
END
GO
```

## Power BI Integration

### Connect to SQL Server from Power BI
```m
// Power Query M code for connection
let
    Source = Sql.Database(
        Environment.GetEnvironmentVariable("SQL_SERVER_HOST"), 
        "LogiFleetPulse",
        [Query="SELECT * FROM vw_CrossFactEfficiency"]
    ),
    #"Changed Type" = Table.TransformColumnTypes(Source,{
        {"Year", Int64.Type}, 
        {"Month", Int64.Type}, 
        {"AvgWarehouseDwellTime", type number}, 
        {"AvgFleetIdleTime", type number},
        {"EfficiencyRatio", type number}
    })
in
    #"Changed Type"
```

### DAX Measures for Cross-Fact Analysis
```dax
// Composite KPI: Logistics Efficiency Score
LogisticsEfficiencyScore = 
VAR WarehouseThroughput = 
    DIVIDE(
        SUM(FactWarehouseOperations[QuantityProcessed]),
        SUM(FactWarehouseOperations[ProcessingTimeMinutes]),
        0
    )
VAR FleetUtilization = 
    DIVIDE(
        SUM(FactFleetTrips[DistanceKM]),
        SUM(FactFleetTrips[DistanceKM]) + SUM(FactFleetTrips[IdleTimeMinutes]) * 0.5,
        0
    )
VAR OnTimeDeliveryRate = 
    DIVIDE(
        CALCULATE(COUNT(FactFleetTrips[TripID]), FactFleetTrips[DeliveryOnTime] = TRUE()),
        COUNT(FactFleetTrips[TripID]),
        0
    )
RETURN
    (WarehouseThroughput * 0.4 + FleetUtilization * 0.3 + OnTimeDeliveryRate * 0.3) * 100

// Time Intelligence: YoY Comparison
WarehouseDwellTime_YoY = 
VAR CurrentPeriod = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR PreviousYear = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        SAMEPERIODLASTYEAR(DimTime[FullDateTime])
    )
RETURN
    DIVIDE(CurrentPeriod - PreviousYear, PreviousYear, 0)
```

## Automated Alerting System

### SQL Server Agent Job for Threshold Monitoring
```sql
CREATE PROCEDURE sp_MonitorKPIThresholds
AS
BEGIN
    DECLARE @AlertMessages TABLE (
        AlertType VARCHAR(50),
        Message NVARCHAR(MAX),
        Severity VARCHAR(20)
    )
    
    -- Check fleet idle time threshold (>15% of trip time)
    INSERT INTO @AlertMessages
    SELECT 
        'Fleet Idle Alert',
        'Vehicle ' + v.VehicleCode + ' idle time: ' + 
        CAST(ROUND((ft.IdleTimeMinutes * 100.0) / 
            DATEDIFF(MINUTE, ft.TripStartTime, ft.TripEndTime), 2) AS VARCHAR) + '%',
        'HIGH'
    FROM FactFleetTrips ft
    INNER JOIN DimVehicle v ON ft.VehicleKey = v.VehicleKey
    WHERE ft.TimeKey >= CAST(FORMAT(DATEADD(HOUR, -24, GETDATE()), 'yyyyMMdd') AS INT)
      AND (ft.IdleTimeMinutes * 100.0) / 
          DATEDIFF(MINUTE, ft.TripStartTime, ft.TripEndTime) > 15
    
    -- Check warehouse dwell time (>72 hours)
    INSERT INTO @AlertMessages
    SELECT 
        'Warehouse Dwell Alert',
        'SKU ' + p.SKU + ' at ' + w.WarehouseName + 
        ' dwell time: ' + CAST(wh.DwellTimeMinutes/60 AS VARCHAR) + ' hours',
        'MEDIUM'
    FROM FactWarehouseOperations wh
    INNER JOIN DimWarehouse w ON wh.WarehouseKey = w.WarehouseKey
    INNER JOIN DimProductGravity p ON wh.ProductKey = p.ProductKey
    WHERE wh.TimeKey >= CAST(FORMAT(DATEADD(HOUR, -24, GETDATE()), 'yyyyMMdd') AS INT)
      AND wh.DwellTimeMinutes > 4320 -- 72 hours
    
    -- Send alerts via database mail
    IF EXISTS (SELECT 1 FROM @AlertMessages)
    BEGIN
        DECLARE @Body NVARCHAR(MAX) = ''
        
        SELECT @Body = @Body + AlertType + ' [' + Severity + ']: ' + Message + CHAR(13) + CHAR(10)
        FROM @AlertMessages
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL_RECIPIENTS}',
            @subject = 'LogiFleet Pulse: KPI Threshold Breaches',
            @body = @Body
    END
END
GO
```

## Row-Level Security Implementation

### Configure Role-Based Access
```sql
-- Create security predicate function
CREATE FUNCTION fn_SecurityPredicate(@WarehouseKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS SecurityCheck
WHERE 
    @WarehouseKey IN (
        SELECT WarehouseKey 
        FROM dbo.UserWarehouseAccess 
        WHERE UserName = USER_NAME()
    )
    OR IS_MEMBER('db_owner') = 1
GO

-- Apply security policy to fact tables
CREATE SECURITY POLICY WarehouseAccessPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(WarehouseKey)
ON dbo.FactWarehouseOperations,
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(WarehouseKey)
ON dbo.FactFleetTrips
WITH (STATE = ON)
GO

-- Grant user warehouse access
CREATE TABLE UserWarehouseAccess (
    UserName NVARCHAR(128),
    WarehouseKey INT,
    FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey)
)
GO

-- Example: Grant access
INSERT INTO UserWarehouseAccess (UserName, WarehouseKey)
VALUES ('DOMAIN\JohnDoe', 1), ('DOMAIN\JohnDoe', 3)
GO
```

## Performance Optimization

### Partitioning Strategy
```sql
-- Create partition function for time-based partitioning
CREATE PARTITION FUNCTION pf_MonthlyPartition (INT)
AS RANGE RIGHT FOR VALUES (
    202601, 202602, 202603, 202604, 202605, 202606,
    202607, 202608, 202609, 202610, 202611, 202612,
    202701, 202702 -- Extend as needed
)
GO

-- Create partition scheme
CREATE PARTITION SCHEME ps_MonthlyPartition
AS PARTITION pf_MonthlyPartition
ALL TO ([PRIMARY])
GO

-- Apply to fact table
CREATE TABLE FactWarehouseOperations_Partitioned (
    OperationID INT IDENTITY(1,1),
    TimeKey INT NOT NULL,
    -- ... other columns ...
    CONSTRAINT PK_FactWarehouseOperations_Partitioned 
        PRIMARY KEY (OperationID, TimeKey)
) ON ps_MonthlyPartition(TimeKey)
GO
```

### Indexing Strategy
```sql
-- Covering index for common fleet queries
CREATE NONCLUSTERED INDEX IX_FactFleetTrips_Coverage
ON FactFleetTrips (VehicleKey, TimeKey)
INCLUDE (DistanceKM, FuelConsumedLiters, IdleTimeMinutes, DeliveryOnTime)
GO

-- Filtered index for on-time delivery analysis
CREATE NONCLUSTERED INDEX IX_FactFleetTrips_OnTime
ON FactFleetTrips (RouteKey, TimeKey)
WHERE DeliveryOnTime = 1
GO

-- Columnstore index for aggregation queries
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactWarehouseOperations_CS
ON FactWarehouseOperations (
    TimeKey, WarehouseKey, ProductKey, OperationType, 
    QuantityProcessed, DwellTimeMinutes
)
GO
```

## Troubleshooting

### Common Issues

**Issue: Slow cross-fact queries**
```sql
-- Check execution plan and statistics
SET STATISTICS IO ON
SET STATISTICS TIME ON

EXEC sp_DetectBottlenecks @ThresholdPercentile = 90.0

-- Update statistics on large tables
UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN
UPDATE STATISTICS FactFleetTrips WITH FULLSCAN
```

**Issue: TimeKey mismatch between fact tables**
```sql
-- Validate time dimension coverage
SELECT 
    'FactWarehouseOperations' AS TableName,
    MIN(TimeKey) AS MinKey,
    MAX(TimeKey) AS MaxKey,
    COUNT(DISTINCT TimeKey) AS UniqueKeys
FROM FactWarehouseOperations
UNION ALL
SELECT 
    'FactFleetTrips',
    MIN(TimeKey),
    MAX(TimeKey),
    COUNT(DISTINCT TimeKey)
FROM FactFleetTrips
UNION ALL
SELECT 
    'DimTime',
    MIN(TimeKey),
    MAX(TimeKey),
    COUNT(TimeKey)
FROM DimTime

-- Populate missing time keys
EXEC sp_PopulateDimTime 
    @StartDate = '2026-01-01', 
    @EndDate = '2027-12-31'
```

**Issue: Power BI refresh failures**
```powershell
# Test connection from Power BI gateway
Test-NetConnection -ComputerName ${SQL_SERVER_HOST} -Port 1433

# Verify service account permissions
sqlcmd -S ${SQL_SERVER_HOST} -d LogiFleetPulse -E -Q "SELECT USER_NAME()"
```

**Issue: Gravity scores not updating**
```sql
-- Manually trigger gravity score recalculation
EXEC sp_UpdateProductGravityScores

-- Check for missing velocity/value classifications
SELECT COUNT(*) 
FROM DimProductGravity 
WHERE VelocityClass IS NULL OR ValueClass IS NULL
```

### Monitoring Query Performance
```sql
-- Identify top resource-consuming queries
SELECT TOP 10
    qs.total_elapsed_time / qs.execution_count AS AvgDuration,
    qs.execution_count,
    SUBSTRING(st.text, (qs.statement_start_offset/2)+1,   
        ((CASE qs.statement_end_offset  
            WHEN -1 THEN DATALENGTH(st.text)  
            ELSE qs.statement_end_offset  
        END - qs.statement_start_offset)/2) + 1) AS QueryText
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
WHERE st.text LIKE '%FactWarehouse%' OR st.text LIKE '%FactFleet%'
ORDER BY qs.total_elapsed_time / qs.execution_count DESC
```
