---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server data warehouse and Power BI dashboard template for multi-modal logistics intelligence with cross-fact KPI harmonization
triggers:
  - "set up LogiFleet Pulse logistics analytics"
  - "deploy supply chain data warehouse schema"
  - "configure Power BI logistics dashboard"
  - "implement warehouse gravity zones"
  - "create multi-fact star schema for fleet management"
  - "build cross-modal supply chain analytics"
  - "set up real-time logistics intelligence platform"
  - "configure fleet and warehouse KPI tracking"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive data warehousing and analytics template for logistics operations. It provides:

- **Multi-fact star schema** for warehouse operations, fleet trips, and cross-dock transfers
- **Power BI dashboard templates** for real-time logistics visualization
- **Cross-fact KPI harmonization** linking inventory, fleet, and supplier metrics
- **Time-phased dimensions** for temporal analysis at 15-minute granularity
- **Predictive bottleneck detection** using composite aggregation bridges
- **Warehouse Gravity Zones™** for spatial optimization based on pick frequency and value

The platform integrates data from WMS, GPS/telematics, supplier portals, weather APIs, and customer orders into a unified semantic layer.

## Installation

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to source systems (WMS, TMS, ERP) via SQL, REST API, or flat files

### Deployment Steps

1. **Clone the repository:**

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**

```sql
-- Connect to your SQL Server instance and execute the schema script
-- Assumes database name: LogiFleetDB

USE master;
GO

CREATE DATABASE LogiFleetDB;
GO

USE LogiFleetDB;
GO

-- Execute the main schema creation script
-- This creates dimension and fact tables with proper indexing
```

3. **Configure data sources:**

Update connection strings in configuration (use environment variables for credentials):

```json
{
  "connections": {
    "wms_source": {
      "server": "${WMS_SQL_SERVER}",
      "database": "${WMS_DATABASE}",
      "username": "${WMS_USER}",
      "password": "${WMS_PASSWORD}"
    },
    "fleet_api": {
      "endpoint": "${FLEET_API_ENDPOINT}",
      "api_key": "${FLEET_API_KEY}"
    }
  }
}
```

4. **Import Power BI template:**

```powershell
# Open the .pbit template
Start-Process "LogiFleet_Pulse_Master.pbit"
```

## Core Schema Structure

### Key Fact Tables

**FactWarehouseOperations** - Core warehouse activity metrics:

```sql
CREATE TABLE FactWarehouseOperations (
    OperationID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    DateKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(50) NOT NULL, -- 'PUTAWAY', 'PICK', 'PACK', 'SHIP'
    QuantityUnits INT NOT NULL,
    DwellTimeMinutes INT,
    CycleTimeSeconds INT,
    ZoneGravityScore DECIMAL(5,2),
    OperatorID INT,
    LoadTimestamp DATETIME2 DEFAULT GETDATE(),
    CONSTRAINT FK_WH_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WH_Date FOREIGN KEY (DateKey) REFERENCES DimDate(DateKey),
    CONSTRAINT FK_WH_Geo FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_WH_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
);

-- Columnstore index for analytical queries
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWH 
ON FactWarehouseOperations (TimeKey, DateKey, GeographyKey, ProductKey, QuantityUnits, DwellTimeMinutes);

-- Rowstore index for operational queries
CREATE INDEX IX_FactWH_DateTimeProduct 
ON FactWarehouseOperations (DateKey, TimeKey, ProductKey) 
INCLUDE (QuantityUnits, DwellTimeMinutes);
```

**FactFleetTrips** - Vehicle and route telemetry:

```sql
CREATE TABLE FactFleetTrips (
    TripID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    DateKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadWeightKG INT,
    TripDurationMinutes INT,
    AverageSpeedKPH DECIMAL(5,2),
    WeatherCondition VARCHAR(50),
    OnTimeFlag BIT,
    LoadTimestamp DATETIME2 DEFAULT GETDATE(),
    CONSTRAINT FK_Fleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_Fleet_Route FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey)
);

CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactFleet 
ON FactFleetTrips (DateKey, TimeKey, VehicleKey, RouteKey, DistanceKM, FuelConsumedLiters);
```

### Key Dimension Tables

**DimTime** - 15-minute granularity time dimension:

```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    TimeValue TIME(0) NOT NULL,
    HourOfDay TINYINT NOT NULL,
    MinuteBucket TINYINT NOT NULL, -- 0, 15, 30, 45
    ShiftPeriod VARCHAR(20), -- 'Morning', 'Afternoon', 'Night'
    IsBusinessHours BIT,
    QuarterHour VARCHAR(10) -- '08:00-08:15', etc.
);

-- Populate time dimension (0-95 for 24 hours * 4 intervals)
WITH TimeGenerator AS (
    SELECT 0 AS TimeKey
    UNION ALL
    SELECT TimeKey + 1 
    FROM TimeGenerator 
    WHERE TimeKey < 95
)
INSERT INTO DimTime (TimeKey, TimeValue, HourOfDay, MinuteBucket, ShiftPeriod, IsBusinessHours, QuarterHour)
SELECT 
    TimeKey,
    TIMEFROMPARTS(TimeKey / 4, (TimeKey % 4) * 15, 0, 0, 0) AS TimeValue,
    TimeKey / 4 AS HourOfDay,
    (TimeKey % 4) * 15 AS MinuteBucket,
    CASE 
        WHEN TimeKey / 4 BETWEEN 6 AND 13 THEN 'Morning'
        WHEN TimeKey / 4 BETWEEN 14 AND 21 THEN 'Afternoon'
        ELSE 'Night'
    END AS ShiftPeriod,
    CASE WHEN TimeKey / 4 BETWEEN 8 AND 17 THEN 1 ELSE 0 END AS IsBusinessHours,
    FORMAT(TIMEFROMPARTS(TimeKey / 4, (TimeKey % 4) * 15, 0, 0, 0), 'HH:mm') + '-' +
    FORMAT(DATEADD(MINUTE, 15, TIMEFROMPARTS(TimeKey / 4, (TimeKey % 4) * 15, 0, 0, 0)), 'HH:mm') AS QuarterHour
FROM TimeGenerator
OPTION (MAXRECURSION 100);
```

**DimProductGravity** - Product classification with gravity scoring:

```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName VARCHAR(255) NOT NULL,
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    UnitValue DECIMAL(10,2),
    FragilityScore TINYINT, -- 1-10 scale
    PickFrequencyScore DECIMAL(5,2), -- Calculated from historical data
    GravityZone VARCHAR(20), -- 'High', 'Medium', 'Low'
    GravityScore DECIMAL(5,2), -- Composite score for optimization
    WeightKG DECIMAL(8,2),
    DimensionsLWH VARCHAR(50),
    EffectiveDate DATE NOT NULL,
    ExpiryDate DATE,
    IsCurrent BIT DEFAULT 1
);

-- Calculate gravity score trigger
CREATE TRIGGER TR_CalculateGravity ON DimProductGravity
AFTER INSERT, UPDATE
AS
BEGIN
    UPDATE dg
    SET GravityScore = 
        (dg.PickFrequencyScore * 0.5) + 
        (dg.UnitValue / 100 * 0.3) + 
        (dg.FragilityScore * 0.2),
        GravityZone = 
        CASE 
            WHEN (dg.PickFrequencyScore * 0.5 + dg.UnitValue / 100 * 0.3 + dg.FragilityScore * 0.2) >= 7 THEN 'High'
            WHEN (dg.PickFrequencyScore * 0.5 + dg.UnitValue / 100 * 0.3 + dg.FragilityScore * 0.2) >= 4 THEN 'Medium'
            ELSE 'Low'
        END
    FROM DimProductGravity dg
    INNER JOIN inserted i ON dg.ProductKey = i.ProductKey;
END;
```

## Data Loading Patterns

### Incremental ETL for Warehouse Operations

```sql
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LoadFromDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Load from WMS staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey, DateKey, GeographyKey, ProductKey, 
        OperationType, QuantityUnits, DwellTimeMinutes, 
        CycleTimeSeconds, ZoneGravityScore, OperatorID
    )
    SELECT 
        -- Map to time dimension (15-minute buckets)
        (DATEPART(HOUR, s.OperationTimestamp) * 4) + (DATEPART(MINUTE, s.OperationTimestamp) / 15) AS TimeKey,
        CONVERT(INT, FORMAT(s.OperationTimestamp, 'yyyyMMdd')) AS DateKey,
        g.GeographyKey,
        p.ProductKey,
        s.OperationType,
        s.Quantity,
        DATEDIFF(MINUTE, s.ArrivalTimestamp, s.DepartureTimestamp) AS DwellTimeMinutes,
        s.CycleTimeSeconds,
        p.GravityScore,
        s.OperatorID
    FROM Staging_WMS_Operations s
    INNER JOIN DimGeography g ON s.WarehouseID = g.WarehouseID AND g.IsCurrent = 1
    INNER JOIN DimProductGravity p ON s.SKU = p.SKU AND p.IsCurrent = 1
    WHERE s.OperationTimestamp >= @LoadFromDateTime
        AND s.ProcessedFlag = 0;
    
    -- Mark as processed
    UPDATE Staging_WMS_Operations
    SET ProcessedFlag = 1
    WHERE OperationTimestamp >= @LoadFromDateTime
        AND ProcessedFlag = 0;
END;
GO

-- Schedule incremental load every 15 minutes
-- (Use SQL Server Agent or external orchestration)
```

### Cross-Fact KPI Query Example

```sql
-- Find SKUs with high dwell time that are also on inefficient routes
CREATE VIEW vw_CrossFactDwellFleetAnalysis AS
WITH WarehouseDwell AS (
    SELECT 
        p.SKU,
        p.ProductName,
        p.GravityZone,
        AVG(wh.DwellTimeMinutes) AS AvgDwellMinutes,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations wh
    INNER JOIN DimProductGravity p ON wh.ProductKey = p.ProductKey
    INNER JOIN DimDate d ON wh.DateKey = d.DateKey
    WHERE d.CalendarDate >= DATEADD(DAY, -30, GETDATE())
        AND wh.OperationType IN ('PUTAWAY', 'PICK')
    GROUP BY p.SKU, p.ProductName, p.GravityZone
    HAVING AVG(wh.DwellTimeMinutes) > 240 -- More than 4 hours
),
FleetEfficiency AS (
    SELECT 
        f.RouteKey,
        r.RouteName,
        AVG(f.IdleTimeMinutes * 1.0 / NULLIF(f.TripDurationMinutes, 0)) AS IdleTimeRatio,
        AVG(f.FuelConsumedLiters / NULLIF(f.DistanceKM, 0)) AS FuelEfficiencyLPK
    FROM FactFleetTrips f
    INNER JOIN DimRoute r ON f.RouteKey = r.RouteKey
    INNER JOIN DimDate d ON f.DateKey = d.DateKey
    WHERE d.CalendarDate >= DATEADD(DAY, -30, GETDATE())
    GROUP BY f.RouteKey, r.RouteName
    HAVING AVG(f.IdleTimeMinutes * 1.0 / NULLIF(f.TripDurationMinutes, 0)) > 0.15
)
SELECT 
    wd.SKU,
    wd.ProductName,
    wd.GravityZone,
    wd.AvgDwellMinutes,
    fe.RouteName,
    fe.IdleTimeRatio,
    fe.FuelEfficiencyLPK,
    -- Composite inefficiency score
    (wd.AvgDwellMinutes / 60.0) * fe.IdleTimeRatio * 100 AS InefficiencyIndex
FROM WarehouseDwell wd
CROSS JOIN FleetEfficiency fe
WHERE wd.GravityZone = 'High' -- Focus on high-priority items
ORDER BY InefficiencyIndex DESC;
```

## Power BI Integration

### Connecting to SQL Server

```dax
// In Power Query (M language)
let
    Source = Sql.Database(
        Environment.GetEnvironmentVariable("LOGIFLEET_SQL_SERVER"), 
        "LogiFleetDB",
        [Query="SELECT * FROM FactWarehouseOperations WHERE DateKey >= " & Number.ToText(Date.Year(DateTime.LocalNow()) * 10000 + Date.Month(DateTime.LocalNow()) * 100 + 01)]
    )
in
    Source
```

### Key DAX Measures

**Average Dwell Time by Gravity Zone:**

```dax
AvgDwellTime_ByGravityZone = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    ALLEXCEPT(DimProductGravity, DimProductGravity[GravityZone])
)
```

**Fleet Utilization Rate:**

```dax
FleetUtilizationRate = 
DIVIDE(
    SUM(FactFleetTrips[TripDurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[TripDurationMinutes]),
    0
)
```

**Cross-Fact Bottleneck Index:**

```dax
BottleneckIndex = 
VAR HighDwellCount = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[DwellTimeMinutes] > 240
    )
VAR HighIdleCount = 
    CALCULATE(
        COUNTROWS(FactFleetTrips),
        DIVIDE(FactFleetTrips[IdleTimeMinutes], FactFleetTrips[TripDurationMinutes], 0) > 0.15
    )
RETURN
    HighDwellCount + (HighIdleCount * 1.5) // Weight fleet issues higher
```

**Time Intelligence - Rolling 7-Day Average:**

```dax
DwellTime_7DayMA = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    DATESINPERIOD(
        DimDate[CalendarDate],
        LASTDATE(DimDate[CalendarDate]),
        -7,
        DAY
    )
)
```

## Automated Alerting

### SQL Server Agent Alert Procedure

```sql
CREATE PROCEDURE usp_CheckKPIThresholds
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    DECLARE @EmailRecipient VARCHAR(255) = 'logistics-ops@company.com';
    
    -- Check 1: Warehouse dwell time exceeds threshold
    IF EXISTS (
        SELECT 1 
        FROM FactWarehouseOperations wh
        INNER JOIN DimDate d ON wh.DateKey = d.DateKey
        WHERE d.CalendarDate = CAST(GETDATE() AS DATE)
            AND wh.DwellTimeMinutes > 480 -- 8 hours
        GROUP BY wh.GeographyKey
        HAVING COUNT(*) > 10
    )
    BEGIN
        SET @AlertMessage = 'ALERT: More than 10 items have exceeded 8-hour dwell time today. Check warehouse capacity.';
        
        -- Log to alert table
        INSERT INTO AlertLog (AlertType, AlertMessage, Severity, CreatedAt)
        VALUES ('DWELL_TIME_BREACH', @AlertMessage, 'HIGH', GETDATE());
        
        -- Send email (requires Database Mail configuration)
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = @EmailRecipient,
            @subject = 'LogiFleet Alert: High Dwell Time',
            @body = @AlertMessage;
    END;
    
    -- Check 2: Fleet idle time ratio exceeds 20%
    IF EXISTS (
        SELECT 1
        FROM FactFleetTrips f
        INNER JOIN DimDate d ON f.DateKey = d.DateKey
        WHERE d.CalendarDate = CAST(GETDATE() AS DATE)
        GROUP BY f.VehicleKey
        HAVING AVG(CAST(f.IdleTimeMinutes AS FLOAT) / NULLIF(f.TripDurationMinutes, 0)) > 0.2
    )
    BEGIN
        SET @AlertMessage = 'ALERT: One or more vehicles have idle time exceeding 20% of trip duration.';
        
        INSERT INTO AlertLog (AlertType, AlertMessage, Severity, CreatedAt)
        VALUES ('FLEET_IDLE_HIGH', @AlertMessage, 'MEDIUM', GETDATE());
        
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = @EmailRecipient,
            @subject = 'LogiFleet Alert: High Fleet Idle Time',
            @body = @AlertMessage;
    END;
END;
GO

-- Create SQL Agent Job to run every hour
USE msdb;
GO

EXEC sp_add_job 
    @job_name = 'LogiFleet_KPI_Monitoring',
    @enabled = 1,
    @description = 'Monitors KPIs and sends alerts';
GO

EXEC sp_add_jobstep
    @job_name = 'LogiFleet_KPI_Monitoring',
    @step_name = 'Execute KPI Check',
    @subsystem = 'TSQL',
    @database_name = 'LogiFleetDB',
    @command = 'EXEC usp_CheckKPIThresholds';
GO

EXEC sp_add_schedule
    @schedule_name = 'HourlyCheck',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 8, -- Hours
    @freq_subday_interval = 1;
GO

EXEC sp_attach_schedule
    @job_name = 'LogiFleet_KPI_Monitoring',
    @schedule_name = 'HourlyCheck';
GO
```

## Configuration Patterns

### Row-Level Security Setup

```sql
-- Create security predicate function
CREATE FUNCTION fn_SecurityPredicate(@GeographyKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN
    SELECT 1 AS result
    WHERE @GeographyKey IN (
        SELECT GeographyKey 
        FROM dbo.UserGeographyAccess 
        WHERE UserName = USER_NAME()
    )
    OR IS_MEMBER('LogisticsAdmin') = 1; -- Admins see all
GO

-- Apply security policy to fact tables
CREATE SECURITY POLICY GeographySecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(GeographyKey)
ON dbo.FactWarehouseOperations,
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(OriginGeographyKey)
ON dbo.FactFleetTrips
WITH (STATE = ON);
GO

-- Grant user access to specific regions
INSERT INTO UserGeographyAccess (UserName, GeographyKey)
VALUES 
    ('region_manager_east', 1),
    ('region_manager_east', 2),
    ('region_manager_west', 3);
```

### External Data Source Integration (Weather API)

```sql
-- Create external table for weather data ingestion
CREATE MASTER KEY ENCRYPTION BY PASSWORD = '${EXTERNAL_DATA_KEY}';
GO

CREATE DATABASE SCOPED CREDENTIAL WeatherAPICredential
WITH IDENTITY = 'SHARED ACCESS SIGNATURE',
SECRET = '${WEATHER_API_TOKEN}';
GO

CREATE EXTERNAL DATA SOURCE WeatherDataFeed
WITH (
    TYPE = BLOB_STORAGE,
    LOCATION = 'https://weatherapi.blob.core.windows.net/feeds',
    CREDENTIAL = WeatherAPICredential
);
GO

-- Stored procedure to load weather data
CREATE PROCEDURE usp_LoadWeatherData
AS
BEGIN
    -- Load JSON from external source
    DECLARE @WeatherJSON NVARCHAR(MAX);
    
    -- Parse and insert into staging
    INSERT INTO Staging_WeatherData (GeographyKey, ObservationTime, Condition, TempCelsius, Precipitation)
    SELECT 
        g.GeographyKey,
        JSON_VALUE(value, '$.timestamp') AS ObservationTime,
        JSON_VALUE(value, '$.condition') AS Condition,
        JSON_VALUE(value, '$.temp_c') AS TempCelsius,
        JSON_VALUE(value, '$.precip_mm') AS Precipitation
    FROM OPENROWSET(
        BULK 'weather_feed.json',
        DATA_SOURCE = 'WeatherDataFeed',
        SINGLE_CLOB
    ) AS WeatherFile
    CROSS APPLY OPENJSON(WeatherFile.BulkColumn) AS WeatherRecords
    INNER JOIN DimGeography g ON JSON_VALUE(WeatherRecords.value, '$.location') = g.City;
END;
GO
```

## Common Troubleshooting

### Performance Optimization

**Issue: Slow cross-fact queries**

```sql
-- Add covering indexes for common join patterns
CREATE INDEX IX_FactWH_ProductGeo 
ON FactWarehouseOperations (ProductKey, GeographyKey) 
INCLUDE (DwellTimeMinutes, QuantityUnits);

CREATE INDEX IX_FactFleet_VehicleRoute
ON FactFleetTrips (VehicleKey, RouteKey)
INCLUDE (IdleTimeMinutes, FuelConsumedLiters);

-- Update statistics after bulk loads
UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS FactFleetTrips WITH FULLSCAN;
```

**Issue: Power BI refresh timeout**

```dax
// Use incremental refresh in Power BI
// Create RangeStart and RangeEnd parameters
// Filter table with:
Table.SelectRows(
    Source, 
    each [OperationTimestamp] >= RangeStart and [OperationTimestamp] < RangeEnd
)

// Configure incremental refresh policy:
// - Refresh last 7 days
// - Archive data older than 365 days
```

### Data Quality Checks

```sql
-- Create data quality monitoring view
CREATE VIEW vw_DataQualityMetrics AS
SELECT 
    'WarehouseOps' AS TableName,
    COUNT(*) AS TotalRecords,
    SUM(CASE WHEN DwellTimeMinutes IS NULL THEN 1 ELSE 0 END) AS NullDwellTime,
    SUM(CASE WHEN DwellTimeMinutes < 0 THEN 1 ELSE 0 END) AS NegativeDwellTime,
    SUM(CASE WHEN ProductKey NOT IN (SELECT ProductKey FROM DimProductGravity) THEN 1 ELSE 0 END) AS OrphanedProductKeys
FROM FactWarehouseOperations
WHERE DateKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -7, GETDATE()), 'yyyyMMdd'))

UNION ALL

SELECT 
    'FleetTrips' AS TableName,
    COUNT(*) AS TotalRecords,
    SUM(CASE WHEN IdleTimeMinutes IS NULL THEN 1 ELSE 0 END) AS NullIdleTime,
    SUM(CASE WHEN IdleTimeMinutes > TripDurationMinutes THEN 1 ELSE 0 END) AS IdleExceedsTrip,
    SUM(CASE WHEN FuelConsumedLiters <= 0 THEN 1 ELSE 0 END) AS InvalidFuel
FROM FactFleetTrips
WHERE DateKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -7, GETDATE()), 'yyyyMMdd'));
```

### Backup and Recovery

```sql
-- Full backup with compression
BACKUP DATABASE LogiFleetDB
TO DISK = 'D:\SQLBackups\LogiFleetDB_Full.bak'
WITH COMPRESSION, INIT, STATS = 10;

-- Transaction log backup (every 15 minutes)
BACKUP LOG LogiFleetDB
TO DISK = 'D:\SQLBackups\LogiFleetDB_Log.trn'
WITH COMPRESSION, NOINIT;

-- Restore procedure
RESTORE DATABASE LogiFleetDB
FROM DISK = 'D:\SQLBackups\LogiFleetDB_Full.bak'
WITH NORECOVERY;

RESTORE LOG LogiFleetDB
FROM DISK = 'D:\SQLBackups\LogiFleetDB_Log.trn'
WITH RECOVERY;
```

## Best Practices

1. **Always use parameterized queries** to prevent SQL injection when integrating external data sources
2. **Partition large fact tables** by DateKey for improved query performance and maintenance
3. **Implement slowly changing dimensions (SCD Type 2)** for DimProductGravity to track historical gravity score changes
4. **Use columnstore indexes** for analytical queries, rowstore for transactional operations
5. **Schedule incremental loads** during off-peak hours; use 15-minute intervals for real-time dashboards
6. **Enable Query Store** for performance monitoring and troubleshooting
7. **Document all custom DAX measures** with business logic comments
8. **Test row-level security** thoroughly before production deployment
9. **Monitor Power BI dataset refresh history** and set up failure alerts
10. **Version control** all SQL scripts and Power BI templates using Git

## Advanced Patterns

### Warehouse Gravity Zone Rebalancing

```sql
-- Stored procedure to recommend storage zone reassignments
CREATE PROCEDURE usp_RecommendGravityRebalancing
AS
BEGIN
    WITH CurrentPlacement AS (
        SELECT 
            p.ProductKey,
            p.SKU,
            p.GravityZone AS CurrentZone,
            AVG(wh.DwellTimeMinutes) AS AvgDwell,
            COUNT(*) AS PickCount,
            -- Recalculate gravity based on last 30 days
            (COUNT(*) * 1.0 / 30) * 0.5 + 
            (p.UnitValue / 100 * 0.3) + 
            (p.FragilityScore * 0.2) AS RecalculatedGravity
        FROM DimProductGravity p
        INNER JOIN FactWarehouseOperations wh ON p.ProductKey = wh.ProductKey
        INNER JOIN DimDate d ON wh.DateKey = d.DateKey
        WHERE d.CalendarDate >= DATEADD(DAY, -30, GETDATE())
            AND wh.OperationType = 'PICK'
        GROUP BY p.ProductKey, p.SKU, p.GravityZone, p.UnitValue, p.FragilityScore
    )
    SELECT 
        SKU,
        CurrentZone,
        CASE 
            WHEN RecalculatedGravity >= 7 THEN 'High'
            WHEN RecalculatedGravity >= 4 THEN 'Medium'
            ELSE 'Low'
        END AS RecommendedZone,
        RecalculatedGravity,
        AvgDwell,
        PickCount,
        CASE 
            WHEN CurrentZone <> CASE 
                WHEN RecalculatedGravity >= 7 THEN 'High'
                WHEN RecalculatedGravity >= 4 THEN 'Medium'
                ELSE 'Low'
            END THEN 'REASSIGN'
            ELSE 'MAINTAIN'
        END AS Action
    FROM CurrentPlacement
    WHERE CurrentZone <> CASE 
            WHEN RecalculatedGravity >= 7 THEN 'High'
            WHEN RecalculatedGravity >= 4 THEN 'Medium'
            ELSE 'Low'
        END
    ORDER BY ABS(RecalculatedGravity - 
        CASE CurrentZone 
            WHEN 'High' THEN 8.5 
            WHEN 'Medium' THEN 5.5 
            ELSE 2.5 
        END) DESC;
END;
```

This skill provides comprehensive coverage of LogiFleet Pulse for AI coding agents to effectively assist developers in deploying, configuring, and utilizing this supply chain analytics platform.
