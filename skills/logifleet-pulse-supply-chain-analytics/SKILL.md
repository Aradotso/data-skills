---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing engine for logistics, fleet management, and supply chain analytics with multi-fact star schema
triggers:
  - "set up logistics data warehouse with SQL Server"
  - "create Power BI dashboard for fleet tracking"
  - "build supply chain analytics data model"
  - "implement warehouse operations reporting"
  - "configure multi-fact star schema for logistics"
  - "deploy fleet telemetry and KPI dashboard"
  - "integrate warehouse and fleet data analytics"
  - "set up cross-modal supply chain intelligence"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

LogiFleet Pulse is a comprehensive data warehousing and analytics solution for supply chain, logistics, and fleet management built on MS SQL Server and Power BI. It provides a multi-fact star schema that harmonizes warehouse operations, fleet telemetry, inventory management, and external data sources into unified KPI dashboards with predictive analytics capabilities.

## What It Does

- **Multi-Fact Data Model**: Integrates warehouse operations, fleet trips, cross-dock transfers, and supplier data
- **Unified Semantic Layer**: Single source of truth linking inventory, fleet, and logistics KPIs
- **Real-Time Dashboards**: Power BI templates with 15-minute refresh cycles
- **Predictive Analytics**: Bottleneck detection, fleet triage scoring, and temporal elasticity modeling
- **Warehouse Gravity Zones**: Spatial optimization based on pick frequency, value, and fragility
- **Role-Based Security**: Row-level security for multi-tenant and multi-department access

## Installation

### Prerequisites

- MS SQL Server 2019 or higher
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Admin access to SQL Server instance

### Database Setup

1. **Clone the repository**:
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema**:
```sql
-- Connect to your SQL Server instance
-- Execute the master schema script
:r schema/01_create_database.sql
:r schema/02_create_dimensions.sql
:r schema/03_create_facts.sql
:r schema/04_create_views.sql
:r schema/05_create_stored_procedures.sql
:r schema/06_create_indexes.sql
```

3. **Configure data sources** in `config.json`:
```json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_USERNAME}",
    "password": "${SQL_PASSWORD}"
  },
  "data_sources": {
    "wms_connection": "${WMS_CONNECTION_STRING}",
    "telemetry_api": "${FLEET_API_ENDPOINT}",
    "weather_api_key": "${WEATHER_API_KEY}"
  },
  "refresh_interval_minutes": 15
}
```

## Core Data Model

### Fact Tables

#### FactWarehouseOperations
Tracks all warehouse micro-operations:

```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey INT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'RECEIVE', 'PUTAWAY', 'PICK', 'PACK', 'SHIP'
    Quantity DECIMAL(18,2),
    DwellTimeMinutes INT,
    OperatorID VARCHAR(50),
    ZoneID VARCHAR(20),
    CompletionTimestamp DATETIME2,
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey)
);

-- Create clustered columnstore index for analytics
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactWarehouseOperations 
ON FactWarehouseOperations;

-- Create non-clustered index for time-based queries
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time 
ON FactWarehouseOperations(TimeKey) INCLUDE (ProductKey, Quantity);
```

#### FactFleetTrips
Captures fleet telemetry and route performance:

```sql
CREATE TABLE FactFleetTrips (
    TripKey INT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT NOT NULL,
    TripStartTime DATETIME2,
    TripEndTime DATETIME2,
    DistanceMiles DECIMAL(10,2),
    FuelConsumedGallons DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadWeightLbs DECIMAL(10,2),
    DelayMinutes INT,
    DelayReason VARCHAR(100),
    WeatherCondition VARCHAR(50),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey)
);

CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactFleetTrips 
ON FactFleetTrips;
```

### Dimension Tables

#### DimTime (Time-Phased Dimension)
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2,
    Hour INT,
    Minute INT,
    DayOfWeek VARCHAR(10),
    DayOfMonth INT,
    Month INT,
    Quarter INT,
    FiscalYear INT,
    IsWeekend BIT,
    IsHoliday BIT,
    TimeSlot15Min VARCHAR(5) -- '00:00', '00:15', '00:30', '00:45'
);

-- Populate with 15-minute granularity
INSERT INTO DimTime (TimeKey, FullDateTime, Hour, Minute, DayOfWeek, Month, Quarter, FiscalYear, IsWeekend, TimeSlot15Min)
SELECT 
    CAST(FORMAT(dt, 'yyyyMMddHHmm') AS INT) AS TimeKey,
    dt AS FullDateTime,
    DATEPART(HOUR, dt) AS Hour,
    DATEPART(MINUTE, dt) AS Minute,
    DATENAME(WEEKDAY, dt) AS DayOfWeek,
    DATEPART(MONTH, dt) AS Month,
    DATEPART(QUARTER, dt) AS Quarter,
    YEAR(dt) AS FiscalYear,
    CASE WHEN DATEPART(WEEKDAY, dt) IN (1,7) THEN 1 ELSE 0 END AS IsWeekend,
    FORMAT(dt, 'HH:mm') AS TimeSlot15Min
FROM (
    SELECT DATEADD(MINUTE, 15 * n, '2024-01-01 00:00:00') AS dt
    FROM (SELECT TOP 105120 ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1 AS n
          FROM sys.all_objects a CROSS JOIN sys.all_objects b) nums
) dates
WHERE dt < '2027-01-01';
```

#### DimProductGravity (Warehouse Gravity Zones)
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value / replenishment_time
    PickFrequency DECIMAL(10,2), -- Picks per day
    UnitValue DECIMAL(10,2),
    Fragility VARCHAR(20), -- 'HIGH', 'MEDIUM', 'LOW'
    RecommendedZone VARCHAR(20), -- 'GRAVITY-A', 'GRAVITY-B', 'GRAVITY-C'
    LastGravityUpdate DATETIME2
);

-- Stored procedure to recalculate gravity scores
CREATE PROCEDURE sp_UpdateProductGravity
AS
BEGIN
    UPDATE DimProductGravity
    SET GravityScore = (
        SELECT 
            (COUNT(*) * AVG(UnitValue)) / NULLIF(AVG(ReplenishmentLeadTimeDays), 0)
        FROM FactWarehouseOperations fwo
        WHERE fwo.ProductKey = DimProductGravity.ProductKey
            AND fwo.TimeKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm') AS INT)
    ),
    RecommendedZone = CASE 
        WHEN GravityScore >= 75 THEN 'GRAVITY-A'
        WHEN GravityScore >= 50 THEN 'GRAVITY-B'
        ELSE 'GRAVITY-C'
    END,
    LastGravityUpdate = GETDATE();
END;
```

## Key Queries and Views

### Cross-Fact KPI Harmonization
```sql
-- View: Unified dwell time vs fleet efficiency
CREATE VIEW vw_DwellTimeFleetImpact AS
SELECT 
    dt.FiscalYear,
    dt.Quarter,
    dp.Category AS ProductCategory,
    AVG(fwo.DwellTimeMinutes) AS AvgDwellTimeMinutes,
    AVG(fft.IdleTimeMinutes) AS AvgFleetIdleMinutes,
    AVG(fft.FuelConsumedGallons / NULLIF(fft.DistanceMiles, 0)) AS AvgFuelEfficiency,
    COUNT(DISTINCT fwo.OperationKey) AS WarehouseOperations,
    COUNT(DISTINCT fft.TripKey) AS FleetTrips
FROM FactWarehouseOperations fwo
INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
INNER JOIN FactFleetTrips fft ON dt.TimeKey = fft.TimeKey
WHERE fwo.OperationType = 'SHIP'
    AND fft.TripEndTime IS NOT NULL
GROUP BY dt.FiscalYear, dt.Quarter, dp.Category;
```

### Predictive Bottleneck Index
```sql
CREATE VIEW vw_PredictiveBottleneck AS
WITH HourlyMetrics AS (
    SELECT 
        dt.TimeSlot15Min,
        dt.DayOfWeek,
        w.WarehouseName,
        COUNT(*) AS OperationCount,
        AVG(DwellTimeMinutes) AS AvgDwell,
        STDEV(DwellTimeMinutes) AS StdDevDwell
    FROM FactWarehouseOperations fwo
    INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    INNER JOIN DimWarehouse w ON fwo.WarehouseKey = w.WarehouseKey
    WHERE dt.TimeKey >= CAST(FORMAT(DATEADD(DAY, -90, GETDATE()), 'yyyyMMddHHmm') AS INT)
    GROUP BY dt.TimeSlot15Min, dt.DayOfWeek, w.WarehouseName
)
SELECT 
    TimeSlot15Min,
    DayOfWeek,
    WarehouseName,
    OperationCount,
    AvgDwell,
    -- Bottleneck score: high operation count + high dwell variance
    (OperationCount / 10.0) * (StdDevDwell / NULLIF(AvgDwell, 0)) AS BottleneckScore
FROM HourlyMetrics
WHERE OperationCount > 5 -- Minimum threshold
ORDER BY BottleneckScore DESC;
```

### Fleet Triage Engine
```sql
CREATE PROCEDURE sp_GenerateFleetTriage
    @DateFrom DATETIME2,
    @DateTo DATETIME2
AS
BEGIN
    SELECT 
        v.VehicleID,
        v.VehicleType,
        COUNT(DISTINCT fft.TripKey) AS TotalTrips,
        AVG(fft.IdleTimeMinutes) AS AvgIdleTime,
        SUM(fft.DelayMinutes) AS TotalDelayMinutes,
        AVG(fft.FuelConsumedGallons / NULLIF(fft.DistanceMiles, 0)) AS FuelEfficiency,
        -- Triage score: weighted by high-value cargo impact
        SUM(
            CASE 
                WHEN dp.GravityScore >= 75 THEN fft.DelayMinutes * 3
                WHEN dp.GravityScore >= 50 THEN fft.DelayMinutes * 2
                ELSE fft.DelayMinutes
            END
        ) AS TriageScore
    FROM FactFleetTrips fft
    INNER JOIN DimVehicle v ON fft.VehicleKey = v.VehicleKey
    INNER JOIN FactWarehouseOperations fwo ON fft.TimeKey = fwo.TimeKey
    INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
    WHERE fft.TripStartTime BETWEEN @DateFrom AND @DateTo
    GROUP BY v.VehicleID, v.VehicleType
    HAVING SUM(fft.DelayMinutes) > 30 -- Only flag vehicles with significant delays
    ORDER BY TriageScore DESC;
END;
```

## Power BI Integration

### Loading the Template
```bash
# Open Power BI Desktop
# File > Import > Power BI Template (.pbit)
# Navigate to: LogiFleet_Pulse_Master.pbit
```

### Connection Configuration
When prompted, enter connection parameters:
- **SQL Server**: Your SQL Server hostname
- **Database**: LogiFleetPulse
- **Authentication**: Windows or SQL Server authentication

### DAX Measures for Cross-Fact Analysis

```dax
// Measure: Composite Efficiency Score
CompositeEfficiencyScore = 
VAR WarehouseEfficiency = 
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]) / 60
VAR FleetEfficiency = 
    AVERAGE(FactFleetTrips[IdleTimeMinutes]) / 
    AVERAGE(FactFleetTrips[DistanceMiles])
RETURN
    (1 / WarehouseEfficiency) * (1 / FleetEfficiency) * 100

// Measure: Gravity Zone Compliance
GravityZoneCompliance = 
DIVIDE(
    COUNTROWS(
        FILTER(
            FactWarehouseOperations,
            RELATED(DimProductGravity[RecommendedZone]) = 
            FactWarehouseOperations[ZoneID]
        )
    ),
    COUNTROWS(FactWarehouseOperations),
    0
) * 100

// Measure: Predictive Delay Risk
PredictiveDelayRisk = 
VAR HistoricalDelayRate = 
    CALCULATE(
        COUNTROWS(FactFleetTrips),
        FactFleetTrips[DelayMinutes] > 15
    ) / COUNTROWS(FactFleetTrips)
VAR CurrentWeatherFactor = 
    IF(
        SELECTEDVALUE(FactFleetTrips[WeatherCondition]) IN {"Rain", "Snow", "Fog"},
        1.5,
        1.0
    )
RETURN
    HistoricalDelayRate * CurrentWeatherFactor * 100
```

### Row-Level Security
```dax
// Create role: WarehouseManager
[DimWarehouse[WarehouseName]] = USERNAME()

// Create role: RegionalDirector
[DimGeography[Region]] IN 
    LOOKUPVALUE(
        UserPermissions[AllowedRegions],
        UserPermissions[Username], USERNAME()
    )
```

## Data Loading Patterns

### Incremental Load Stored Procedure
```sql
CREATE PROCEDURE sp_IncrementalLoadWarehouseOps
AS
BEGIN
    DECLARE @LastLoadTime DATETIME2;
    SELECT @LastLoadTime = MAX(CompletionTimestamp) 
    FROM FactWarehouseOperations;

    -- Load from external WMS source
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, WarehouseKey, OperationType, 
        Quantity, DwellTimeMinutes, OperatorID, ZoneID, CompletionTimestamp
    )
    SELECT 
        CAST(FORMAT(wms.timestamp, 'yyyyMMddHHmm') AS INT) AS TimeKey,
        dp.ProductKey,
        dw.WarehouseKey,
        wms.operation_type,
        wms.quantity,
        DATEDIFF(MINUTE, wms.start_time, wms.end_time) AS DwellTimeMinutes,
        wms.operator_id,
        wms.zone_id,
        wms.timestamp
    FROM OPENROWSET(
        'SQLNCLI', 
        '${WMS_CONNECTION_STRING}',
        'SELECT * FROM wms.operations WHERE timestamp > ?', @LastLoadTime
    ) AS wms
    INNER JOIN DimProduct dp ON wms.sku = dp.SKU
    INNER JOIN DimWarehouse dw ON wms.warehouse_code = dw.WarehouseCode;
    
    -- Recalculate gravity scores
    EXEC sp_UpdateProductGravity;
END;
```

### Automated Scheduling (SQL Server Agent)
```sql
-- Create job for 15-minute refresh
EXEC msdb.dbo.sp_add_job 
    @job_name = N'LogiFleet_IncrementalLoad';

EXEC msdb.dbo.sp_add_jobstep
    @job_name = N'LogiFleet_IncrementalLoad',
    @step_name = N'Load_Warehouse_Data',
    @command = N'EXEC sp_IncrementalLoadWarehouseOps',
    @database_name = N'LogiFleetPulse';

EXEC msdb.dbo.sp_add_schedule
    @schedule_name = N'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;

EXEC msdb.dbo.sp_attach_schedule
    @job_name = N'LogiFleet_IncrementalLoad',
    @schedule_name = N'Every15Minutes';
```

## Alerting and Monitoring

### KPI Breach Alert
```sql
CREATE PROCEDURE sp_SendKPIAlert
    @AlertType VARCHAR(50),
    @Threshold DECIMAL(10,2),
    @CurrentValue DECIMAL(10,2),
    @Recipients VARCHAR(500)
AS
BEGIN
    DECLARE @Subject VARCHAR(200);
    DECLARE @Body VARCHAR(MAX);

    SET @Subject = 'LogiFleet Alert: ' + @AlertType + ' threshold breached';
    SET @Body = 'Current value: ' + CAST(@CurrentValue AS VARCHAR) + 
                ' exceeds threshold: ' + CAST(@Threshold AS VARCHAR);

    -- Use Database Mail
    EXEC msdb.dbo.sp_send_dbmail
        @profile_name = 'LogiFleetAlerts',
        @recipients = @Recipients,
        @subject = @Subject,
        @body = @Body;
END;

-- Trigger alert for high fleet idle time
CREATE PROCEDURE sp_CheckFleetIdleAlerts
AS
BEGIN
    DECLARE @AvgIdleTime DECIMAL(10,2);
    SELECT @AvgIdleTime = AVG(IdleTimeMinutes)
    FROM FactFleetTrips
    WHERE TimeKey >= CAST(FORMAT(DATEADD(HOUR, -1, GETDATE()), 'yyyyMMddHHmm') AS INT);

    IF @AvgIdleTime > 15
    BEGIN
        EXEC sp_SendKPIAlert 
            @AlertType = 'Fleet Idle Time',
            @Threshold = 15.0,
            @CurrentValue = @AvgIdleTime,
            @Recipients = '${ALERT_EMAIL_RECIPIENTS}';
    END;
END;
```

## Common Patterns

### Pattern 1: Temporal Elasticity Analysis
```sql
-- Simulate capacity change impact
WITH BaselineMetrics AS (
    SELECT 
        AVG(DwellTimeMinutes) AS BaselineDwell,
        COUNT(*) AS BaselineOps
    FROM FactWarehouseOperations
    WHERE TimeKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm') AS INT)
)
SELECT 
    'Current State' AS Scenario,
    BaselineDwell,
    BaselineOps,
    NULL AS ProjectedDwell,
    NULL AS ProjectedOps
FROM BaselineMetrics
UNION ALL
SELECT 
    '95% Capacity' AS Scenario,
    NULL,
    NULL,
    BaselineDwell * 1.25 AS ProjectedDwell, -- 25% increase assumption
    BaselineOps * 0.95 AS ProjectedOps
FROM BaselineMetrics;
```

### Pattern 2: Multi-Modal Correlation
```sql
-- Find correlation between weather and combined delays
SELECT 
    fft.WeatherCondition,
    AVG(fwo.DwellTimeMinutes) AS AvgWarehouseDwell,
    AVG(fft.DelayMinutes) AS AvgFleetDelay,
    COUNT(DISTINCT fft.TripKey) AS AffectedTrips,
    -- Combined delay score
    (AVG(fwo.DwellTimeMinutes) + AVG(fft.DelayMinutes)) AS TotalDelayImpact
FROM FactFleetTrips fft
INNER JOIN DimTime dt ON fft.TimeKey = dt.TimeKey
INNER JOIN FactWarehouseOperations fwo ON dt.TimeKey = fwo.TimeKey
WHERE fft.WeatherCondition IS NOT NULL
    AND dt.TimeKey >= CAST(FORMAT(DATEADD(DAY, -90, GETDATE()), 'yyyyMMddHHmm') AS INT)
GROUP BY fft.WeatherCondition
ORDER BY TotalDelayImpact DESC;
```

### Pattern 3: Self-Service Query via Natural Language
Configure Power BI Q&A with synonyms:
```json
{
  "synonyms": {
    "dwell": ["wait time", "holding time", "storage duration"],
    "fleet": ["vehicles", "trucks", "delivery fleet"],
    "gravity": ["priority", "importance", "velocity"],
    "bottleneck": ["congestion", "delay", "slowdown"]
  },
  "featured_questions": [
    "Show me high gravity products with long dwell times",
    "Which routes have the most delays this month",
    "Compare warehouse efficiency across regions"
  ]
}
```

## Troubleshooting

### Issue: Slow Cross-Fact Queries
**Solution**: Verify columnstore indexes and statistics are current
```sql
-- Rebuild columnstore indexes
ALTER INDEX CCI_FactWarehouseOperations 
ON FactWarehouseOperations REBUILD;

ALTER INDEX CCI_FactFleetTrips 
ON FactFleetTrips REBUILD;

-- Update statistics
UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS FactFleetTrips WITH FULLSCAN;
```

### Issue: Power BI Refresh Failures
**Solution**: Check connection string and gateway configuration
```sql
-- Test connection from SQL Server
SELECT @@SERVERNAME AS ServerName, DB_NAME() AS DatabaseName;

-- Verify user permissions
SELECT 
    dp.name AS UserName,
    dp.type_desc AS UserType,
    o.name AS ObjectName,
    p.permission_name
FROM sys.database_permissions p
INNER JOIN sys.database_principals dp ON p.grantee_principal_id = dp.principal_id
INNER JOIN sys.objects o ON p.major_id = o.object_id
WHERE dp.name = '${SQL_USERNAME}';
```

### Issue: Missing Time Dimension Records
**Solution**: Extend DimTime population
```sql
-- Add future time periods
INSERT INTO DimTime (TimeKey, FullDateTime, Hour, Minute, DayOfWeek, Month, Quarter, FiscalYear, IsWeekend, TimeSlot15Min)
SELECT 
    CAST(FORMAT(dt, 'yyyyMMddHHmm') AS INT) AS TimeKey,
    dt AS FullDateTime,
    DATEPART(HOUR, dt) AS Hour,
    DATEPART(MINUTE, dt) AS Minute,
    DATENAME(WEEKDAY, dt) AS DayOfWeek,
    DATEPART(MONTH, dt) AS Month,
    DATEPART(QUARTER, dt) AS Quarter,
    YEAR(dt) AS FiscalYear,
    CASE WHEN DATEPART(WEEKDAY, dt) IN (1,7) THEN 1 ELSE 0 END AS IsWeekend,
    FORMAT(dt, 'HH:mm') AS TimeSlot15Min
FROM (
    SELECT DATEADD(MINUTE, 15 * n, (SELECT MAX(FullDateTime) FROM DimTime)) AS dt
    FROM (SELECT TOP 35040 ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1 AS n
          FROM sys.all_objects a CROSS JOIN sys.all_objects b) nums
) dates
WHERE dt NOT IN (SELECT FullDateTime FROM DimTime);
```

### Issue: Gravity Score Not Updating
**Solution**: Check stored procedure execution and data freshness
```sql
-- Manually trigger gravity recalculation
EXEC sp_UpdateProductGravity;

-- Verify last update
SELECT ProductKey, SKU, GravityScore, LastGravityUpdate
FROM DimProductGravity
WHERE LastGravityUpdate < DATEADD(DAY, -1, GETDATE())
ORDER BY GravityScore DESC;
```

## Performance Optimization

### Partitioning Strategy
```sql
-- Partition FactWarehouseOperations by month
CREATE PARTITION FUNCTION pf_MonthlyPartition (INT)
AS RANGE RIGHT FOR VALUES (
    202401, 202402, 202403, 202404, 202405, 202406,
    202407, 202408, 202409, 202410, 202411, 202412
);

CREATE PARTITION SCHEME ps_MonthlyPartition
AS PARTITION pf_MonthlyPartition ALL TO ([PRIMARY]);

-- Recreate table on partition scheme
CREATE TABLE FactWarehouseOperations_Partitioned (
    OperationKey INT IDENTITY(1,1),
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    OperationType VARCHAR(50),
    Quantity DECIMAL(18,2),
    DwellTimeMinutes INT,
    OperatorID VARCHAR(50),
    ZoneID VARCHAR(20),
    CompletionTimestamp DATETIME2,
    MonthKey AS (TimeKey / 1000000) PERSISTED
) ON ps_MonthlyPartition(MonthKey);
```

### Indexing Best Practices
```sql
-- Cover queries filtering by time + product
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time_Product
ON FactWarehouseOperations(TimeKey, ProductKey)
INCLUDE (Quantity, DwellTimeMinutes, ZoneID);

-- Cover fleet queries by vehicle and route
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle_Route
ON FactFleetTrips(VehicleKey, RouteKey, TimeKey)
INCLUDE (IdleTimeMinutes, DelayMinutes);
```

## Environment Variables Reference

Required environment variables for configuration:

- `SQL_SERVER_HOST` - SQL Server hostname or IP
- `SQL_USERNAME` - SQL authentication username
- `SQL_PASSWORD` - SQL authentication password
- `WMS_CONNECTION_STRING` - Warehouse management system connection
- `FLEET_API_ENDPOINT` - Fleet telemetry API URL
- `WEATHER_API_KEY` - External weather service API key
- `ALERT_EMAIL_RECIPIENTS` - Comma-separated email list for alerts

## Additional Resources

- **Documentation**: See `/docs` folder for detailed schema diagrams
- **Sample Data**: Use `/sample_data/seed_data.sql` for testing
- **Power BI Templates**: Multiple dashboard templates in `/powerbi` folder
- **API Integration**: REST API examples in `/integrations` folder
