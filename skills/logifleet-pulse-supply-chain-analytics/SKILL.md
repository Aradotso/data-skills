---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server data warehousing and Power BI analytics engine for logistics, fleet management, and supply chain intelligence
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "create logistics data warehouse with SQL Server"
  - "build Power BI dashboard for fleet management"
  - "implement multi-fact star schema for warehousing"
  - "configure cross-modal supply chain analytics"
  - "deploy LogiCore warehouse and fleet intelligence"
  - "integrate logistics KPI tracking and visualization"
  - "design real-time supply chain monitoring system"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

LogiFleet Pulse is an advanced data warehousing and analytics solution that unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a single semantic layer. It provides:

- **Multi-fact star schema** for warehouse operations, fleet trips, and cross-dock activities
- **Power BI dashboards** with 15-minute refresh intervals
- **Cross-fact KPI harmonization** linking inventory with fleet performance
- **Predictive analytics** for bottleneck detection and maintenance prioritization
- **Time-phased dimensions** for temporal analysis and simulation
- **Role-based access control** with row-level security

The platform is built on MS SQL Server (2019+) and Power BI, designed for 3PLs, retail chains, distributors, and logistics operators.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019 or later
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Power BI Desktop (latest version)
- Access to data sources: WMS, TMS, telematics APIs, or sample data files

### Step 1: Deploy SQL Schema

Clone the repository and navigate to the SQL scripts directory:

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse/sql
```

Execute the schema creation script in SSMS:

```sql
-- Run this first to create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute the main schema script
-- (Assuming the file is named schema_create.sql)
:r schema_create.sql
GO
```

### Step 2: Configure Data Sources

Create a configuration file from the sample:

```bash
cp config_sample.json config.json
```

Edit `config.json` with your connection strings:

```json
{
  "sql_server": {
    "server": "YOUR_SQL_SERVER",
    "database": "LogiFleetPulse",
    "auth": "windows"
  },
  "data_sources": {
    "wms_api": "${WMS_API_ENDPOINT}",
    "fleet_telemetry": "${FLEET_API_ENDPOINT}",
    "weather_api_key": "${WEATHER_API_KEY}"
  },
  "refresh_interval_minutes": 15
}
```

### Step 3: Load Initial Data

If using sample data:

```sql
-- Load sample dimension data
EXEC sp_LoadSampleDimensions;

-- Load sample fact data
EXEC sp_LoadSampleFacts @StartDate = '2026-01-01', @EndDate = '2026-06-01';
GO
```

### Step 4: Import Power BI Template

1. Open Power BI Desktop
2. File → Import → Power BI Template
3. Select `LogiFleet_Pulse_Master.pbit`
4. Enter SQL Server connection details when prompted
5. Verify data model relationships load correctly

## Core Data Model Structure

### Key Dimension Tables

```sql
-- DimTime: 15-minute granularity time dimension
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    TimeStamp DATETIME NOT NULL,
    HourOfDay TINYINT,
    DayOfWeek TINYINT,
    IsWeekend BIT,
    FiscalPeriod VARCHAR(10),
    INDEX IX_TimeStamp (TimeStamp)
);

-- DimGeography: Hierarchical location dimension
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    Continent VARCHAR(50),
    Country VARCHAR(100),
    Region VARCHAR(100),
    City VARCHAR(100),
    WarehouseCode VARCHAR(20),
    RouteNodeID VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6)
);

-- DimProductGravity: Product classification with velocity scoring
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    VelocityScore DECIMAL(5,2), -- Picks per day
    ValueScore DECIMAL(10,2),   -- Unit value
    FragilityScore TINYINT,     -- 1-10 scale
    GravityZone VARCHAR(20),    -- High/Medium/Low
    INDEX IX_GravityZone (GravityZone)
);
```

### Key Fact Tables

```sql
-- FactWarehouseOperations: Core warehouse activity metrics
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    OperationType VARCHAR(20), -- Putaway/Pick/Pack/Ship
    Quantity INT,
    DwellTimeMinutes INT,
    CycleTimeSeconds INT,
    ZoneCode VARCHAR(20),
    OperatorID VARCHAR(50),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey)
);

-- FactFleetTrips: Vehicle telemetry and route performance
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    DistanceMiles DECIMAL(8,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedGallons DECIMAL(6,2),
    LoadWeightLbs INT,
    DriverID VARCHAR(50),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
);
```

## Common SQL Queries & Patterns

### Cross-Fact KPI: Warehouse Velocity vs Fleet Efficiency

```sql
-- Correlate SKU movement speed with delivery route efficiency
WITH WarehouseVelocity AS (
    SELECT 
        p.SKU,
        p.GravityZone,
        AVG(w.DwellTimeMinutes) AS AvgDwellTime,
        COUNT(*) AS TotalOperations
    FROM FactWarehouseOperations w
    JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    WHERE w.OperationType = 'Pick'
    GROUP BY p.SKU, p.GravityZone
),
FleetEfficiency AS (
    SELECT 
        g.WarehouseCode,
        AVG(f.IdleTimeMinutes * 100.0 / NULLIF(f.DurationMinutes, 0)) AS AvgIdlePercent,
        AVG(f.FuelConsumedGallons / NULLIF(f.DistanceMiles, 0)) AS AvgFuelPerMile
    FROM FactFleetTrips f
    JOIN DimGeography g ON f.OriginGeographyKey = g.GeographyKey
    GROUP BY g.WarehouseCode
)
SELECT 
    wv.GravityZone,
    AVG(wv.AvgDwellTime) AS WarehouseAvgDwell,
    AVG(fe.AvgIdlePercent) AS FleetAvgIdlePercent,
    AVG(fe.AvgFuelPerMile) AS FleetAvgFuelPerMile
FROM WarehouseVelocity wv
CROSS JOIN FleetEfficiency fe
GROUP BY wv.GravityZone
ORDER BY wv.GravityZone;
```

### Predictive Bottleneck Detection

```sql
-- Identify warehouses with increasing dwell time trends
WITH DwellTrends AS (
    SELECT 
        g.WarehouseCode,
        t.FiscalPeriod,
        AVG(w.DwellTimeMinutes) AS AvgDwell,
        LAG(AVG(w.DwellTimeMinutes)) OVER (
            PARTITION BY g.WarehouseCode 
            ORDER BY t.FiscalPeriod
        ) AS PrevPeriodDwell
    FROM FactWarehouseOperations w
    JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
    JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE w.OperationType IN ('Pick', 'Pack')
    GROUP BY g.WarehouseCode, t.FiscalPeriod
)
SELECT 
    WarehouseCode,
    FiscalPeriod,
    AvgDwell,
    PrevPeriodDwell,
    ((AvgDwell - PrevPeriodDwell) / NULLIF(PrevPeriodDwell, 0)) * 100 AS PercentIncrease
FROM DwellTrends
WHERE PrevPeriodDwell IS NOT NULL
    AND ((AvgDwell - PrevPeriodDwell) / NULLIF(PrevPeriodDwell, 0)) > 0.15
ORDER BY PercentIncrease DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Recommend SKU zone reassignments based on velocity changes
WITH CurrentVelocity AS (
    SELECT 
        p.ProductKey,
        p.SKU,
        p.GravityZone AS CurrentZone,
        COUNT(*) AS RecentPicks,
        AVG(w.CycleTimeSeconds) AS AvgCycleTime
    FROM FactWarehouseOperations w
    JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE w.OperationType = 'Pick'
        AND t.TimeStamp >= DATEADD(DAY, -30, GETDATE())
    GROUP BY p.ProductKey, p.SKU, p.GravityZone
),
RecommendedZone AS (
    SELECT 
        ProductKey,
        SKU,
        CurrentZone,
        RecentPicks,
        AvgCycleTime,
        CASE 
            WHEN RecentPicks > 100 THEN 'High'
            WHEN RecentPicks > 30 THEN 'Medium'
            ELSE 'Low'
        END AS RecommendedZone
    FROM CurrentVelocity
)
SELECT *
FROM RecommendedZone
WHERE CurrentZone <> RecommendedZone
ORDER BY RecentPicks DESC;
```

## Stored Procedures for ETL

### Incremental Data Loading

```sql
-- Incremental load from staging to fact table
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LoadDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRANSACTION;
    
    -- Insert new records from staging
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, GeographyKey, OperationType,
        Quantity, DwellTimeMinutes, CycleTimeSeconds, ZoneCode, OperatorID
    )
    SELECT 
        t.TimeKey,
        p.ProductKey,
        g.GeographyKey,
        s.OperationType,
        s.Quantity,
        s.DwellTimeMinutes,
        s.CycleTimeSeconds,
        s.ZoneCode,
        s.OperatorID
    FROM StagingWarehouseOperations s
    JOIN DimTime t ON s.OperationTimestamp = t.TimeStamp
    JOIN DimProductGravity p ON s.SKU = p.SKU
    JOIN DimGeography g ON s.WarehouseCode = g.WarehouseCode
    WHERE CAST(s.OperationTimestamp AS DATE) = @LoadDate
        AND NOT EXISTS (
            SELECT 1 FROM FactWarehouseOperations f
            WHERE f.TimeKey = t.TimeKey
                AND f.ProductKey = p.ProductKey
                AND f.ZoneCode = s.ZoneCode
        );
    
    COMMIT TRANSACTION;
END;
GO
```

### Automated Alert Generation

```sql
-- Threshold-based alert generation
CREATE PROCEDURE sp_GenerateOperationalAlerts
AS
BEGIN
    DECLARE @AlertThreshold_IdlePercent DECIMAL(5,2) = 15.0;
    DECLARE @AlertThreshold_DwellHours INT = 72;
    
    -- Fleet idle time alerts
    INSERT INTO AlertLog (AlertType, Severity, Message, GeneratedAt)
    SELECT 
        'FleetIdleTime',
        'High',
        'Vehicle ' + CAST(f.VehicleKey AS VARCHAR(10)) + 
        ' exceeded idle threshold: ' + 
        CAST(f.IdleTimeMinutes * 100.0 / f.DurationMinutes AS VARCHAR(10)) + '%',
        GETDATE()
    FROM FactFleetTrips f
    JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.TimeStamp >= DATEADD(HOUR, -1, GETDATE())
        AND (f.IdleTimeMinutes * 100.0 / NULLIF(f.DurationMinutes, 0)) > @AlertThreshold_IdlePercent;
    
    -- Warehouse dwell time alerts
    INSERT INTO AlertLog (AlertType, Severity, Message, GeneratedAt)
    SELECT 
        'ExcessiveDwell',
        'Medium',
        'SKU ' + p.SKU + ' in warehouse ' + g.WarehouseCode + 
        ' exceeded dwell threshold: ' + 
        CAST(w.DwellTimeMinutes / 60.0 AS VARCHAR(10)) + ' hours',
        GETDATE()
    FROM FactWarehouseOperations w
    JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
    JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.TimeStamp >= DATEADD(HOUR, -1, GETDATE())
        AND w.DwellTimeMinutes > (@AlertThreshold_DwellHours * 60);
END;
GO
```

## Power BI DAX Measures

### Cross-Fact Composite KPI

```dax
// Warehouse-Fleet Efficiency Score
Efficiency_Score = 
VAR WarehouseTurnover = 
    DIVIDE(
        SUM(FactWarehouseOperations[Quantity]),
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
    )
VAR FleetUtilization = 
    DIVIDE(
        SUM(FactFleetTrips[DurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
        SUM(FactFleetTrips[DurationMinutes])
    )
RETURN
    (WarehouseTurnover * 0.6 + FleetUtilization * 0.4) * 100
```

### Time Intelligence with Comparison

```dax
// Dwell Time YoY Change
DwellTime_YoY_Change = 
VAR CurrentPeriod = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR PreviousYear = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        SAMEPERIODLASTYEAR(DimTime[TimeStamp])
    )
RETURN
    DIVIDE(CurrentPeriod - PreviousYear, PreviousYear)
```

### Predictive Bottleneck Index

```dax
// Bottleneck Risk Score (0-100)
Bottleneck_Risk = 
VAR DwellTrend = 
    DIVIDE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        CALCULATE(
            AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
            DATEADD(DimTime[TimeStamp], -30, DAY)
        )
    ) - 1
VAR CapacityUtilization = 
    DIVIDE(
        SUM(FactWarehouseOperations[Quantity]),
        MAX(DimGeography[MaxCapacity])
    )
RETURN
    MINX(
        {100},
        (DwellTrend * 40 + CapacityUtilization * 60) * 100
    )
```

## Configuration & Customization

### Row-Level Security Setup

```sql
-- Create role-based security table
CREATE TABLE SecurityRoles (
    RoleID INT PRIMARY KEY IDENTITY(1,1),
    RoleName VARCHAR(50),
    UserEmail VARCHAR(200),
    AllowedWarehouses VARCHAR(MAX), -- Comma-separated codes
    AllowedRegions VARCHAR(MAX)
);

-- Sample data
INSERT INTO SecurityRoles (RoleName, UserEmail, AllowedWarehouses, AllowedRegions)
VALUES 
    ('RegionalManager', 'manager@company.com', 'WH001,WH002', 'WEST,CENTRAL'),
    ('Executive', 'exec@company.com', NULL, NULL); -- NULL = all access
```

Power BI RLS DAX filter:

```dax
// Apply to DimGeography table
[WarehouseCode] IN 
    FILTER(
        VALUES(SecurityRoles[AllowedWarehouses]),
        SecurityRoles[UserEmail] = USERPRINCIPALNAME()
    )
    || ISBLANK(
        LOOKUPVALUE(
            SecurityRoles[AllowedWarehouses],
            SecurityRoles[UserEmail], USERPRINCIPALNAME()
        )
    )
```

### Scheduled Refresh Configuration

In Power BI Service:

1. Navigate to workspace → Dataset settings
2. Configure gateway connection to SQL Server
3. Set refresh schedule:
   - Frequency: Every 15 minutes (requires Premium capacity)
   - Or: Hourly for standard capacity
4. Add failure notification email: `${ALERT_EMAIL}`

For SQL Server Agent scheduled jobs:

```sql
-- Create job to refresh aggregates every 15 minutes
EXEC sp_add_job 
    @job_name = 'LogiFleet_RefreshAggregates',
    @enabled = 1;

EXEC sp_add_jobstep
    @job_name = 'LogiFleet_RefreshAggregates',
    @step_name = 'Load_Warehouse_Operations',
    @subsystem = 'TSQL',
    @command = 'EXEC sp_LoadWarehouseOperations @LoadDate = CAST(GETDATE() AS DATE)';

EXEC sp_add_schedule
    @schedule_name = 'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;

EXEC sp_attach_schedule
    @job_name = 'LogiFleet_RefreshAggregates',
    @schedule_name = 'Every15Minutes';
```

## Integration Patterns

### REST API Data Ingestion

```sql
-- Example: Load fleet telemetry from API into staging
CREATE PROCEDURE sp_IngestFleetTelemetry
AS
BEGIN
    DECLARE @JSON NVARCHAR(MAX);
    
    -- Call external API (requires SQL Server 2022+ or custom CLR)
    -- In practice, use Azure Data Factory or SSIS for API calls
    
    -- Parse JSON into staging table
    INSERT INTO StagingFleetTrips (
        VehicleID, Timestamp, OriginCode, DestinationCode,
        DistanceMiles, DurationMinutes, IdleTimeMinutes, FuelConsumed
    )
    SELECT 
        JSON_VALUE(value, '$.vehicleId'),
        CAST(JSON_VALUE(value, '$.timestamp') AS DATETIME),
        JSON_VALUE(value, '$.origin'),
        JSON_VALUE(value, '$.destination'),
        CAST(JSON_VALUE(value, '$.distance') AS DECIMAL(8,2)),
        CAST(JSON_VALUE(value, '$.duration') AS INT),
        CAST(JSON_VALUE(value, '$.idleTime') AS INT),
        CAST(JSON_VALUE(value, '$.fuel') AS DECIMAL(6,2))
    FROM OPENJSON(@JSON);
END;
GO
```

### External Data Enrichment (Weather API)

```sql
-- Enrich trips with weather conditions
CREATE TABLE ExternalWeatherData (
    WeatherKey INT PRIMARY KEY IDENTITY(1,1),
    GeographyKey INT,
    TimeKey INT,
    Temperature DECIMAL(5,2),
    Precipitation DECIMAL(5,2),
    WindSpeed DECIMAL(5,2),
    Condition VARCHAR(50)
);

-- Join pattern for weather impact analysis
SELECT 
    f.TripKey,
    f.DurationMinutes,
    f.IdleTimeMinutes,
    w.Condition,
    w.Precipitation,
    CASE 
        WHEN w.Precipitation > 0.5 THEN 'HighRisk'
        WHEN w.Precipitation > 0.1 THEN 'MediumRisk'
        ELSE 'LowRisk'
    END AS WeatherRiskLevel
FROM FactFleetTrips f
JOIN ExternalWeatherData w 
    ON f.OriginGeographyKey = w.GeographyKey
    AND f.TimeKey = w.TimeKey;
```

## Troubleshooting

### Issue: Power BI Refresh Failures

**Symptom**: Dataset refresh times out or fails with "Query timeout expired"

**Solution**:
1. Check SQL Server query execution plans:
```sql
-- Identify slow-running queries
SELECT 
    qs.execution_count,
    SUBSTRING(qt.text, qs.statement_start_offset/2 + 1,
        (CASE WHEN qs.statement_end_offset = -1
            THEN LEN(CONVERT(NVARCHAR(MAX), qt.text)) * 2
            ELSE qs.statement_end_offset
        END - qs.statement_start_offset)/2) AS query_text,
    qs.total_elapsed_time / 1000000.0 AS total_elapsed_time_sec
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
WHERE qt.text LIKE '%FactWarehouseOperations%'
ORDER BY qs.total_elapsed_time DESC;
```

2. Add covering indexes:
```sql
-- Index for common filtering patterns
CREATE NONCLUSTERED INDEX IX_FactWarehouse_TimeProdGeo
ON FactWarehouseOperations (TimeKey, ProductKey, GeographyKey)
INCLUDE (Quantity, DwellTimeMinutes, CycleTimeSeconds);
```

3. Enable Power BI query folding by avoiding calculated columns in Power Query

### Issue: Incorrect Cross-Fact Relationships

**Symptom**: DAX measures return blank or unexpected values when joining warehouse and fleet facts

**Solution**:
1. Verify bridge table exists and is properly configured:
```sql
-- Create many-to-many bridge if missing
CREATE TABLE BridgeWarehouseFleet (
    BridgeKey BIGINT PRIMARY KEY IDENTITY(1,1),
    OperationKey BIGINT,
    TripKey BIGINT,
    LinkageType VARCHAR(20) -- 'Outbound', 'Inbound', 'CrossDock'
);
```

2. In Power BI, set cardinality to "Many to Many" and cross-filter direction to "Both"

### Issue: Slow Aggregation Performance

**Symptom**: Dashboards take 30+ seconds to render

**Solution**:
1. Create indexed views for common aggregations:
```sql
CREATE VIEW vw_DailyWarehouseSummary
WITH SCHEMABINDING
AS
SELECT 
    CAST(t.TimeStamp AS DATE) AS OperationDate,
    w.GeographyKey,
    w.ProductKey,
    COUNT_BIG(*) AS OperationCount,
    SUM(w.Quantity) AS TotalQuantity,
    AVG(w.DwellTimeMinutes) AS AvgDwell
FROM dbo.FactWarehouseOperations w
JOIN dbo.DimTime t ON w.TimeKey = t.TimeKey
GROUP BY CAST(t.TimeStamp AS DATE), w.GeographyKey, w.ProductKey;
GO

CREATE UNIQUE CLUSTERED INDEX IX_DailyWarehouseSummary
ON vw_DailyWarehouseSummary (OperationDate, GeographyKey, ProductKey);
```

2. Use aggregation tables in Power BI:
   - Right-click fact table → Manage aggregations
   - Map to summary view
   - Set detail level to "Daily" instead of "15-minute"

### Issue: Row-Level Security Not Working

**Symptom**: Users see all data despite RLS rules

**Solution**:
1. Verify RLS is applied to dimension tables, not fact tables
2. Test using "View as role" in Power BI Desktop
3. Check for blank/null values in security filter columns:
```sql
-- Find users without security assignments
SELECT DISTINCT w.OperatorID
FROM FactWarehouseOperations w
LEFT JOIN SecurityRoles s ON w.OperatorID = s.UserEmail
WHERE s.UserEmail IS NULL;
```

## Best Practices

1. **Partitioning Strategy**: Partition fact tables by month for easier maintenance:
```sql
ALTER TABLE FactWarehouseOperations
ADD CONSTRAINT PK_FactWarehouse_Partitioned
PRIMARY KEY CLUSTERED (OperationKey, TimeKey)
ON MonthlyPartitionScheme(TimeKey);
```

2. **Incremental Loading**: Always use watermark columns to avoid full reloads:
```sql
DECLARE @LastLoadTime DATETIME = (SELECT MAX(LoadTimestamp) FROM ETLLog);
SELECT * FROM StagingWarehouseOperations WHERE ModifiedDate > @LastLoadTime;
```

3. **Data Validation**: Implement constraint checks before loading:
```sql
-- Validate no negative dwell times
IF EXISTS (SELECT 1 FROM StagingWarehouseOperations WHERE DwellTimeMinutes < 0)
    RAISERROR('Invalid dwell time detected', 16, 1);
```

4. **Monitoring**: Log all ETL operations:
```sql
INSERT INTO ETLLog (ProcedureName, RecordsProcessed, Duration, Status)
VALUES ('sp_LoadWarehouseOperations', @@ROWCOUNT, DATEDIFF(SECOND, @StartTime, GETDATE()), 'Success');
```

This skill provides comprehensive guidance for deploying and operating LogiFleet Pulse as a production supply chain analytics platform.
