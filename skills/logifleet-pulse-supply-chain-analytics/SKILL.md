---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehouse with multi-fact star schema for fleet, warehouse, and cross-dock analytics
triggers:
  - set up logifleet pulse supply chain analytics
  - configure logistics data warehouse with power bi
  - implement multi-fact star schema for fleet operations
  - deploy warehouse gravity zones analytics
  - create supply chain kpi dashboard
  - integrate fleet telemetry with warehouse operations
  - build cross-modal logistics intelligence platform
  - optimize warehouse and fleet data modeling
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

LogiFleet Pulse is an advanced MS SQL Server data warehouse and Power BI analytics platform for supply chain logistics. It implements a multi-fact star schema that unifies warehouse operations, fleet telemetry, cross-dock transfers, and external data sources into a single semantic layer for real-time logistics intelligence.

## What It Does

- **Multi-Fact Data Warehouse**: Star schema with FactWarehouseOperations, FactFleetTrips, and FactCrossDock
- **Time-Phased Dimensions**: 15-minute granularity time buckets with role-playing dimensions
- **Warehouse Gravity Zones**: Spatial optimization based on pick frequency, value, and fragility
- **Fleet Triage Engine**: Prioritized maintenance queue based on telemetry and revenue impact
- **Cross-Fact KPI Harmonization**: Links inventory turnover with fuel burn rates and dwell times
- **Power BI Dashboards**: Real-time visualization with role-based security and natural language queries

## Installation

### Prerequisites

- MS SQL Server 2019+ (Enterprise/Standard for production)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio

### Database Setup

1. **Clone the repository**:
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema**:
```sql
-- Connect to your SQL Server instance
-- Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute the schema creation script
-- (Assumes schema.sql exists in the repository)
:r schema.sql
```

3. **Configure data source connections**:
```json
{
  "warehouse_api": {
    "connection_string": "${WMS_CONNECTION_STRING}",
    "refresh_interval": 900
  },
  "fleet_telemetry": {
    "endpoint": "${FLEET_API_ENDPOINT}",
    "api_key": "${FLEET_API_KEY}"
  },
  "weather_service": {
    "endpoint": "${WEATHER_API_ENDPOINT}",
    "api_key": "${WEATHER_API_KEY}"
  }
}
```

## Core Data Model

### Fact Tables

**FactWarehouseOperations** - Tracks all warehouse activities:
```sql
CREATE TABLE FactWarehouseOperations (
    OperationID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    OperationType VARCHAR(50) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    DwellTimeMinutes INT,
    QuantityHandled DECIMAL(18,2),
    OperationCost DECIMAL(18,2),
    GravityZoneKey INT,
    CONSTRAINT FK_WH_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WH_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_WH_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimGeography(GeographyKey)
);

-- Columnstore index for fast aggregation
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouse 
ON FactWarehouseOperations (TimeKey, ProductKey, QuantityHandled, DwellTimeMinutes);
```

**FactFleetTrips** - Captures vehicle telemetry and route data:
```sql
CREATE TABLE FactFleetTrips (
    TripID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadWeightKG DECIMAL(18,2),
    DelayMinutes INT,
    DelayReasonKey INT,
    CONSTRAINT FK_Fleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey)
);
```

### Dimension Tables

**DimTime** - Time dimension with 15-minute granularity:
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME NOT NULL,
    HourOfDay INT,
    MinuteOfHour INT,
    DayOfWeek INT,
    DayName VARCHAR(20),
    WeekOfYear INT,
    MonthOfYear INT,
    QuarterOfYear INT,
    FiscalPeriod VARCHAR(20),
    IsWeekend BIT,
    IsHoliday BIT
);

-- Populate time dimension
DECLARE @StartDate DATETIME = '2024-01-01';
DECLARE @EndDate DATETIME = '2027-12-31';
DECLARE @CurrentDate DATETIME = @StartDate;

WHILE @CurrentDate < @EndDate
BEGIN
    INSERT INTO DimTime (TimeKey, FullDateTime, DateKey, TimeOfDay, HourOfDay, MinuteOfHour, DayOfWeek, DayName, WeekOfYear, MonthOfYear, QuarterOfYear, IsWeekend)
    VALUES (
        CONVERT(INT, FORMAT(@CurrentDate, 'yyyyMMddHHmm')),
        @CurrentDate,
        CONVERT(INT, FORMAT(@CurrentDate, 'yyyyMMdd')),
        CONVERT(TIME, @CurrentDate),
        DATEPART(HOUR, @CurrentDate),
        DATEPART(MINUTE, @CurrentDate),
        DATEPART(WEEKDAY, @CurrentDate),
        DATENAME(WEEKDAY, @CurrentDate),
        DATEPART(WEEK, @CurrentDate),
        DATEPART(MONTH, @CurrentDate),
        DATEPART(QUARTER, @CurrentDate),
        CASE WHEN DATEPART(WEEKDAY, @CurrentDate) IN (1,7) THEN 1 ELSE 0 END
    );
    
    SET @CurrentDate = DATEADD(MINUTE, 15, @CurrentDate);
END;
```

**DimProductGravity** - Product categorization with gravity scoring:
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    UnitValue DECIMAL(18,2),
    Fragility DECIMAL(3,2), -- 0.00 to 1.00
    PickFrequencyScore DECIMAL(5,2), -- Calculated from historical picks
    ReplenishmentLeadDays INT,
    GravityScore AS (PickFrequencyScore * UnitValue * (1 + Fragility)) PERSISTED,
    GravityZone AS (
        CASE 
            WHEN (PickFrequencyScore * UnitValue * (1 + Fragility)) > 1000 THEN 'High'
            WHEN (PickFrequencyScore * UnitValue * (1 + Fragility)) > 500 THEN 'Medium'
            ELSE 'Low'
        END
    ) PERSISTED
);
```

## Key Stored Procedures

### Incremental Data Loading

**Load Warehouse Operations**:
```sql
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Load from staging with transformation
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, WarehouseKey, OperationType, 
        DwellTimeMinutes, QuantityHandled, OperationCost, GravityZoneKey
    )
    SELECT 
        CONVERT(INT, FORMAT(s.OperationTimestamp, 'yyyyMMddHHmm')) AS TimeKey,
        p.ProductKey,
        g.GeographyKey AS WarehouseKey,
        s.OperationType,
        DATEDIFF(MINUTE, s.StartTime, s.EndTime) AS DwellTimeMinutes,
        s.Quantity,
        s.LaborCost + s.EquipmentCost AS OperationCost,
        p.GravityZone
    FROM StagingWarehouseOps s
    INNER JOIN DimProductGravity p ON s.SKU = p.SKU
    INNER JOIN DimGeography g ON s.WarehouseCode = g.LocationCode
    WHERE s.OperationTimestamp > @LastLoadDateTime
        AND s.IsProcessed = 0;
    
    -- Mark as processed
    UPDATE StagingWarehouseOps 
    SET IsProcessed = 1 
    WHERE OperationTimestamp > @LastLoadDateTime;
    
    -- Log completion
    INSERT INTO LoadLog (ProcedureName, RowsProcessed, LoadDateTime)
    VALUES ('usp_LoadWarehouseOperations', @@ROWCOUNT, GETDATE());
END;
```

### Cross-Fact KPI Calculation

**Dwell Time vs Fleet Idle Time Correlation**:
```sql
CREATE PROCEDURE usp_CalculateDwellFleetCorrelation
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    WITH WarehouseDwell AS (
        SELECT 
            t.DateKey,
            p.Category,
            g.Region,
            AVG(w.DwellTimeMinutes) AS AvgDwellMinutes,
            SUM(w.QuantityHandled) AS TotalQuantity
        FROM FactWarehouseOperations w
        INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
        INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
        INNER JOIN DimGeography g ON w.WarehouseKey = g.GeographyKey
        WHERE t.FullDateTime BETWEEN @StartDate AND @EndDate
            AND w.OperationType = 'Shipping'
        GROUP BY t.DateKey, p.Category, g.Region
    ),
    FleetIdle AS (
        SELECT 
            t.DateKey,
            g.Region,
            AVG(f.IdleTimeMinutes) AS AvgIdleMinutes,
            SUM(f.FuelConsumedLiters) AS TotalFuel
        FROM FactFleetTrips f
        INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
        INNER JOIN DimGeography g ON f.OriginGeographyKey = g.GeographyKey
        WHERE t.FullDateTime BETWEEN @StartDate AND @EndDate
        GROUP BY t.DateKey, g.Region
    )
    SELECT 
        wd.DateKey,
        wd.Category,
        wd.Region,
        wd.AvgDwellMinutes,
        fi.AvgIdleMinutes,
        wd.TotalQuantity,
        fi.TotalFuel,
        -- Calculate correlation coefficient (simplified)
        (wd.AvgDwellMinutes / NULLIF(fi.AvgIdleMinutes, 0)) AS DwellIdleRatio
    FROM WarehouseDwell wd
    INNER JOIN FleetIdle fi ON wd.DateKey = fi.DateKey AND wd.Region = fi.Region
    ORDER BY wd.DateKey, wd.Region;
END;
```

### Predictive Bottleneck Detection

**Identify High-Risk Zones**:
```sql
CREATE PROCEDURE usp_PredictBottlenecks
    @ForecastHours INT = 24
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentTime DATETIME = GETDATE();
    DECLARE @ForecastEnd DATETIME = DATEADD(HOUR, @ForecastHours, @CurrentTime);
    
    -- Analyze historical patterns for same time window
    WITH HistoricalPatterns AS (
        SELECT 
            DATEPART(HOUR, t.FullDateTime) AS HourOfDay,
            DATEPART(WEEKDAY, t.FullDateTime) AS DayOfWeek,
            g.GeographyKey,
            g.LocationName,
            AVG(w.DwellTimeMinutes) AS AvgDwellTime,
            STDEV(w.DwellTimeMinutes) AS StdDevDwellTime,
            COUNT(*) AS OperationCount
        FROM FactWarehouseOperations w
        INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
        INNER JOIN DimGeography g ON w.WarehouseKey = g.GeographyKey
        WHERE t.FullDateTime >= DATEADD(DAY, -90, @CurrentTime)
        GROUP BY DATEPART(HOUR, t.FullDateTime), DATEPART(WEEKDAY, t.FullDateTime), 
                 g.GeographyKey, g.LocationName
    ),
    CurrentLoad AS (
        SELECT 
            g.GeographyKey,
            COUNT(*) AS CurrentOperations,
            AVG(w.DwellTimeMinutes) AS CurrentAvgDwell
        FROM FactWarehouseOperations w
        INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
        INNER JOIN DimGeography g ON w.WarehouseKey = g.GeographyKey
        WHERE t.FullDateTime >= DATEADD(HOUR, -2, @CurrentTime)
        GROUP BY g.GeographyKey
    )
    SELECT 
        hp.LocationName,
        hp.HourOfDay,
        hp.DayOfWeek,
        hp.AvgDwellTime AS HistoricalAvgDwell,
        cl.CurrentAvgDwell,
        cl.CurrentOperations,
        hp.OperationCount AS HistoricalOperations,
        -- Bottleneck risk score
        CASE 
            WHEN cl.CurrentAvgDwell > (hp.AvgDwellTime + 2 * hp.StdDevDwellTime) 
                AND cl.CurrentOperations > hp.OperationCount * 1.2
            THEN 'CRITICAL'
            WHEN cl.CurrentAvgDwell > (hp.AvgDwellTime + hp.StdDevDwellTime)
            THEN 'WARNING'
            ELSE 'NORMAL'
        END AS BottleneckRisk
    FROM HistoricalPatterns hp
    INNER JOIN CurrentLoad cl ON hp.GeographyKey = cl.GeographyKey
    WHERE DATEPART(HOUR, @CurrentTime) = hp.HourOfDay
        AND DATEPART(WEEKDAY, @CurrentTime) = hp.DayOfWeek
    ORDER BY BottleneckRisk DESC, cl.CurrentAvgDwell DESC;
END;
```

## Power BI Integration

### Connection Setup

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Enter server name and database: `LogiFleetPulse`
4. Select DirectQuery mode for real-time data or Import for better performance

### DAX Measures

**Total Dwell Time with Gravity Weight**:
```dax
Total Weighted Dwell Time = 
SUMX(
    FactWarehouseOperations,
    FactWarehouseOperations[DwellTimeMinutes] * 
    RELATED(DimProductGravity[GravityScore]) / 100
)
```

**Fleet Efficiency Ratio**:
```dax
Fleet Efficiency % = 
DIVIDE(
    SUM(FactFleetTrips[DistanceKM]),
    SUM(FactFleetTrips[DistanceKM]) + 
    (SUM(FactFleetTrips[IdleTimeMinutes]) * 0.5), -- Assume 0.5 km per idle minute
    0
) * 100
```

**Cross-Fact Throughput Score**:
```dax
Supply Chain Throughput = 
VAR WarehouseVelocity = 
    DIVIDE(
        SUM(FactWarehouseOperations[QuantityHandled]),
        SUM(FactWarehouseOperations[DwellTimeMinutes]),
        0
    )
VAR FleetVelocity = 
    DIVIDE(
        SUM(FactFleetTrips[DistanceKM]),
        SUM(FactFleetTrips[IdleTimeMinutes]) + 
        CALCULATE(SUM(FactFleetTrips[DistanceKM]) / 60), -- Assume 60 km/h average
        0
    )
RETURN
    WarehouseVelocity * FleetVelocity
```

**Time Intelligence - YoY Comparison**:
```dax
Dwell Time YoY Change = 
VAR CurrentDwell = SUM(FactWarehouseOperations[DwellTimeMinutes])
VAR PreviousYearDwell = 
    CALCULATE(
        SUM(FactWarehouseOperations[DwellTimeMinutes]),
        SAMEPERIODLASTYEAR(DimTime[FullDateTime])
    )
RETURN
    DIVIDE(CurrentDwell - PreviousYearDwell, PreviousYearDwell, 0)
```

### Row-Level Security

**Implement regional access control**:
```dax
-- Create role: Regional Manager
[Region] = USERPRINCIPALNAME()

-- Or for specific regions
[Region] IN {"North", "East", "South", "West"}
```

In SQL, create security mapping:
```sql
CREATE TABLE SecurityRoles (
    UserEmail VARCHAR(200) PRIMARY KEY,
    AllowedRegions VARCHAR(500),
    AccessLevel VARCHAR(50)
);

-- Example data
INSERT INTO SecurityRoles VALUES 
('manager.north@company.com', 'North', 'Manager'),
('analyst.east@company.com', 'East', 'Analyst');
```

## Common Patterns

### Pattern 1: Daily ETL Pipeline

```sql
-- Master ETL procedure
CREATE PROCEDURE usp_DailyETL
AS
BEGIN
    BEGIN TRY
        BEGIN TRANSACTION;
        
        DECLARE @LastLoad DATETIME;
        SELECT @LastLoad = MAX(LoadDateTime) FROM LoadLog;
        
        -- Load dimensions first
        EXEC usp_LoadDimProducts;
        EXEC usp_LoadDimGeography;
        EXEC usp_LoadDimVehicles;
        
        -- Load facts
        EXEC usp_LoadWarehouseOperations @LastLoad;
        EXEC usp_LoadFleetTrips @LastLoad;
        
        -- Update gravity scores
        EXEC usp_RecalculateGravityScores;
        
        -- Generate alerts
        EXEC usp_PredictBottlenecks @ForecastHours = 24;
        
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        
        INSERT INTO ErrorLog (ProcedureName, ErrorMessage, ErrorDateTime)
        VALUES ('usp_DailyETL', ERROR_MESSAGE(), GETDATE());
        
        THROW;
    END CATCH
END;
```

**Schedule with SQL Agent**:
```sql
USE msdb;
GO

EXEC sp_add_job 
    @job_name = N'LogiFleet Daily ETL',
    @enabled = 1;

EXEC sp_add_jobstep 
    @job_name = N'LogiFleet Daily ETL',
    @step_name = N'Run ETL',
    @subsystem = N'TSQL',
    @database_name = N'LogiFleetPulse',
    @command = N'EXEC usp_DailyETL';

EXEC sp_add_schedule 
    @schedule_name = N'Daily at 2 AM',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @active_start_time = 020000;

EXEC sp_attach_schedule 
    @job_name = N'LogiFleet Daily ETL',
    @schedule_name = N'Daily at 2 AM';

EXEC sp_add_jobserver 
    @job_name = N'LogiFleet Daily ETL';
```

### Pattern 2: Real-Time Alerting

```sql
CREATE PROCEDURE usp_CheckKPIThresholds
AS
BEGIN
    DECLARE @AlertRecipients VARCHAR(500) = '${ALERT_EMAIL_RECIPIENTS}';
    
    -- Check for critical dwell time
    DECLARE @CriticalDwell TABLE (
        WarehouseName VARCHAR(200),
        ProductCategory VARCHAR(100),
        AvgDwellMinutes INT
    );
    
    INSERT INTO @CriticalDwell
    SELECT TOP 10
        g.LocationName,
        p.Category,
        AVG(w.DwellTimeMinutes) AS AvgDwell
    FROM FactWarehouseOperations w
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    INNER JOIN DimGeography g ON w.WarehouseKey = g.GeographyKey
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    WHERE t.FullDateTime >= DATEADD(HOUR, -4, GETDATE())
        AND p.GravityZone = 'High'
    GROUP BY g.LocationName, p.Category
    HAVING AVG(w.DwellTimeMinutes) > 120
    ORDER BY AvgDwell DESC;
    
    IF EXISTS (SELECT 1 FROM @CriticalDwell)
    BEGIN
        DECLARE @AlertBody NVARCHAR(MAX);
        
        SELECT @AlertBody = COALESCE(@AlertBody + CHAR(13), '') + 
            WarehouseName + ' - ' + ProductCategory + ': ' + 
            CAST(AvgDwellMinutes AS VARCHAR) + ' minutes'
        FROM @CriticalDwell;
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleet Alerts',
            @recipients = @AlertRecipients,
            @subject = 'CRITICAL: High Gravity Zone Dwell Time Alert',
            @body = @AlertBody;
    END;
END;
```

### Pattern 3: Warehouse Gravity Zone Optimization

```sql
CREATE PROCEDURE usp_OptimizeGravityZones
    @SimulationDays INT = 90
AS
BEGIN
    -- Calculate optimal zone assignments based on historical data
    WITH ProductMetrics AS (
        SELECT 
            p.ProductKey,
            p.SKU,
            p.Category,
            COUNT(*) AS TotalPicks,
            AVG(w.DwellTimeMinutes) AS AvgDwell,
            SUM(w.QuantityHandled * p.UnitValue) AS TotalValue
        FROM FactWarehouseOperations w
        INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
        INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
        WHERE t.FullDateTime >= DATEADD(DAY, -@SimulationDays, GETDATE())
            AND w.OperationType IN ('Picking', 'Shipping')
        GROUP BY p.ProductKey, p.SKU, p.Category
    ),
    OptimalZones AS (
        SELECT 
            ProductKey,
            SKU,
            Category,
            TotalPicks,
            AvgDwell,
            TotalValue,
            -- Calculate new gravity score
            (TotalPicks / CAST(@SimulationDays AS DECIMAL)) * TotalValue AS NewGravityScore,
            CASE 
                WHEN (TotalPicks / CAST(@SimulationDays AS DECIMAL)) * TotalValue > 1000 THEN 'High'
                WHEN (TotalPicks / CAST(@SimulationDays AS DECIMAL)) * TotalValue > 500 THEN 'Medium'
                ELSE 'Low'
            END AS RecommendedZone
        FROM ProductMetrics
    )
    SELECT 
        o.SKU,
        o.Category,
        p.GravityZone AS CurrentZone,
        o.RecommendedZone,
        o.NewGravityScore,
        o.TotalPicks,
        CASE 
            WHEN p.GravityZone <> o.RecommendedZone THEN 'REASSIGN'
            ELSE 'KEEP'
        END AS Action
    FROM OptimalZones o
    INNER JOIN DimProductGravity p ON o.ProductKey = p.ProductKey
    WHERE p.GravityZone <> o.RecommendedZone
    ORDER BY o.NewGravityScore DESC;
END;
```

## Configuration

### Connection Strings

Store in environment variables or Azure Key Vault:

```bash
# SQL Server connection
export LOGIFLEET_DB_SERVER="your-server.database.windows.net"
export LOGIFLEET_DB_NAME="LogiFleetPulse"
export LOGIFLEET_DB_USER="${SQL_ADMIN_USER}"
export LOGIFLEET_DB_PASSWORD="${SQL_ADMIN_PASSWORD}"

# External APIs
export WMS_CONNECTION_STRING="${WMS_API_CONNECTION}"
export FLEET_API_ENDPOINT="https://api.fleetprovider.com/v2"
export FLEET_API_KEY="${FLEET_TELEMETRY_KEY}"
export WEATHER_API_ENDPOINT="https://api.weather.com/v3"
export WEATHER_API_KEY="${WEATHER_SERVICE_KEY}"
```

### Power BI Connection String

In Power BI, use M query for parameterized connections:

```m
let
    Server = #"SERVER_NAME",
    Database = "LogiFleetPulse",
    Source = Sql.Database(Server, Database, [
        Query = "SELECT * FROM vw_WarehouseFleetUnified",
        CommandTimeout = #duration(0, 0, 10, 0)
    ])
in
    Source
```

## Troubleshooting

### Issue: Slow Query Performance

**Diagnosis**:
```sql
-- Check missing indexes
SELECT 
    migs.avg_user_impact * (migs.user_seeks + migs.user_scans) AS Impact,
    mid.statement AS TableName,
    mid.equality_columns,
    mid.inequality_columns,
    mid.included_columns
FROM sys.dm_db_missing_index_groups mig
INNER JOIN sys.dm_db_missing_index_group_stats migs ON mig.index_group_handle = migs.group_handle
INNER JOIN sys.dm_db_missing_index_details mid ON mig.index_handle = mid.index_handle
WHERE mid.database_id = DB_ID('LogiFleetPulse')
ORDER BY Impact DESC;
```

**Solution**: Add recommended indexes or partition large fact tables:
```sql
-- Partition FactWarehouseOperations by month
CREATE PARTITION FUNCTION pf_Monthly (INT)
AS RANGE RIGHT FOR VALUES (
    20240101, 20240201, 20240301, 20240401, 20240501, 20240601,
    20240701, 20240801, 20240901, 20241001, 20241101, 20241201
);

CREATE PARTITION SCHEME ps_Monthly
AS PARTITION pf_Monthly
ALL TO ([PRIMARY]);

-- Rebuild table on partition scheme
CREATE TABLE FactWarehouseOperations_Partitioned (
    -- Same schema as original
) ON ps_Monthly(TimeKey);
```

### Issue: Power BI Refresh Failures

**Diagnosis**: Check gateway logs and SQL Server execution:
```sql
-- View active Power BI queries
SELECT 
    session_id,
    start_time,
    status,
    command,
    wait_type,
    wait_time,
    cpu_time,
    reads,
    writes,
    TEXT
FROM sys.dm_exec_requests
CROSS APPLY sys.dm_exec_sql_text(sql_handle)
WHERE database_id = DB_ID('LogiFleetPulse')
ORDER BY start_time DESC;
```

**Solution**: Optimize DirectQuery or switch to Import mode for aggregated views:
```sql
-- Create aggregated view for Power BI
CREATE VIEW vw_DailyWarehouseMetrics
WITH SCHEMABINDING
AS
SELECT 
    t.DateKey,
    g.Region,
    p.Category,
    COUNT_BIG(*) AS OperationCount,
    SUM(w.DwellTimeMinutes) AS TotalDwell,
    SUM(w.QuantityHandled) AS TotalQuantity
FROM dbo.FactWarehouseOperations w
INNER JOIN dbo.DimTime t ON w.TimeKey = t.TimeKey
INNER JOIN dbo.DimGeography g ON w.WarehouseKey = g.GeographyKey
INNER JOIN dbo.DimProductGravity p ON w.ProductKey = p.ProductKey
GROUP BY t.DateKey, g.Region, p.Category;

-- Create indexed view for fast aggregation
CREATE UNIQUE CLUSTERED INDEX UCI_DailyMetrics 
ON vw_DailyWarehouseMetrics (DateKey, Region, Category);
```

### Issue: Gravity Score Calculation Errors

**Diagnosis**:
```sql
-- Find products with NULL or zero gravity scores
SELECT 
    ProductKey,
    SKU,
    PickFrequencyScore,
    UnitValue,
    Fragility,
    GravityScore
FROM DimProductGravity
WHERE GravityScore IS NULL OR GravityScore = 0;
```

**Solution**: Recalculate with default values:
```sql
UPDATE DimProductGravity
SET PickFrequencyScore = 1.0
WHERE PickFrequencyScore IS NULL OR PickFrequencyScore = 0;

UPDATE DimProductGravity
SET Fragility = 0.1
WHERE Fragility IS NULL;

-- Force recalculation of computed columns
ALTER TABLE DimProductGravity
DROP COLUMN GravityScore;

ALTER TABLE DimProductGravity
ADD GravityScore AS (PickFrequencyScore * UnitValue * (1 + Fragility)) PERSISTED;
```

### Issue: ETL Job Failures

**Check error logs**:
```sql
SELECT TOP 20
    ErrorID,
    ProcedureName,
    ErrorMessage,
    ErrorDateTime
FROM ErrorLog
ORDER BY ErrorDateTime DESC;
```

**Enable retry logic**:
```sql
CREATE PROCEDURE usp_DailyETL_WithRetry
AS
BEGIN
    DECLARE @RetryCount INT = 0;
    DECLARE @MaxRetries INT = 3;
    DECLARE @Success BIT = 0;
    
    WHILE @RetryCount < @MaxRet
