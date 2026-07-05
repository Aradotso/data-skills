---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing platform for logistics intelligence with multi-fact star schema, fleet telemetry, and warehouse operations analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure Power BI logistics dashboard"
  - "create warehouse operations data model"
  - "implement fleet telemetry tracking system"
  - "build multi-fact star schema for logistics"
  - "optimize supply chain KPI reporting"
  - "deploy logistics data warehouse"
  - "integrate warehouse management system analytics"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

LogiFleet Pulse is an advanced logistics intelligence platform that combines MS SQL Server data warehousing with Power BI visualization. It provides a multi-modal analytics engine for warehouse operations, fleet telemetry, inventory management, and cross-dock operations using a sophisticated multi-fact star schema architecture.

## What This Project Does

LogiFleet Pulse creates a unified semantic layer that connects:
- **Warehouse Management**: receiving, putaway, picking, packing, shipping operations
- **Fleet Telemetry**: GPS tracking, fuel consumption, driver behavior, vehicle diagnostics
- **Inventory Analytics**: aging curves, SKU velocity, gravity zone optimization
- **Supply Chain KPIs**: cross-fact metrics linking warehouse efficiency to fleet performance

The platform uses time-phased dimensions (15-minute granularity) and bridge tables for many-to-many relationships, enabling complex queries across operational silos.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- Data sources: WMS, telemetry API, ERP system with SQL/REST access

### Database Deployment

1. **Clone the repository and deploy the SQL schema:**

```sql
-- Run the main schema creation script
-- This creates tables, relationships, views, and stored procedures

USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME NOT NULL,
    HourOfDay INT NOT NULL,
    MinuteBucket INT NOT NULL, -- 0, 15, 30, 45
    DayOfWeek NVARCHAR(20),
    IsWeekend BIT,
    FiscalPeriod NVARCHAR(20),
    INDEX IX_DimTime_FullDateTime (FullDateTime)
);

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID NVARCHAR(50) UNIQUE NOT NULL,
    LocationName NVARCHAR(100),
    LocationType NVARCHAR(50), -- Warehouse, Route Node, Cross-Dock
    Country NVARCHAR(100),
    Region NVARCHAR(100),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    INDEX IX_DimGeography_LocationID (LocationID)
);

CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU NVARCHAR(50) UNIQUE NOT NULL,
    ProductName NVARCHAR(200),
    Category NVARCHAR(100),
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value * (1/fragility)
    VelocityClass NVARCHAR(20), -- Fast, Medium, Slow
    ValueTier NVARCHAR(20), -- High, Medium, Low
    FragilityIndex DECIMAL(3,2),
    OptimalZone NVARCHAR(50),
    INDEX IX_DimProduct_SKU (SKU),
    INDEX IX_DimProduct_GravityScore (GravityScore DESC)
);

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    LocationKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType NVARCHAR(50), -- Receiving, Putaway, Picking, Packing, Shipping
    OperationStartTime DATETIME,
    OperationEndTime DATETIME,
    DurationMinutes AS DATEDIFF(MINUTE, OperationStartTime, OperationEndTime),
    Quantity INT,
    DwellTimeHours DECIMAL(10,2),
    PickPathDistance DECIMAL(10,2), -- In meters
    OperatorID NVARCHAR(50),
    INDEX IX_FactWarehouse_Time (TimeKey),
    INDEX IX_FactWarehouse_Product (ProductKey),
    INDEX IX_FactWarehouse_Location (LocationKey)
);

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginLocationKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationLocationKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID NVARCHAR(50),
    DriverID NVARCHAR(50),
    TripStartTime DATETIME,
    TripEndTime DATETIME,
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    LoadingTimeMinutes DECIMAL(10,2),
    UnloadingTimeMinutes DECIMAL(10,2),
    LoadWeight DECIMAL(10,2), -- In kg
    WeatherCondition NVARCHAR(50),
    TrafficDelay BIT,
    INDEX IX_FactFleet_Time (TimeKey),
    INDEX IX_FactFleet_Origin (OriginLocationKey),
    INDEX IX_FactFleet_Destination (DestinationLocationKey),
    INDEX IX_FactFleet_Vehicle (VehicleID)
);

-- Create bridge table for many-to-many relationships
CREATE TABLE BridgeRouteWarehouse (
    BridgeKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TripKey BIGINT NOT NULL FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OperationKey BIGINT NOT NULL FOREIGN KEY REFERENCES FactWarehouseOperations(OperationKey),
    ShipmentID NVARCHAR(50),
    INDEX IX_Bridge_Trip (TripKey),
    INDEX IX_Bridge_Operation (OperationKey)
);
```

2. **Create incremental loading stored procedure:**

```sql
CREATE PROCEDURE usp_IncrementalLoadWarehouseOperations
    @LastLoadDateTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new records from staging/source
    INSERT INTO FactWarehouseOperations (
        TimeKey, LocationKey, ProductKey, OperationType,
        OperationStartTime, OperationEndTime, Quantity,
        DwellTimeHours, PickPathDistance, OperatorID
    )
    SELECT 
        dt.TimeKey,
        dg.GeographyKey,
        dp.ProductKey,
        src.OperationType,
        src.OperationStartTime,
        src.OperationEndTime,
        src.Quantity,
        src.DwellTimeHours,
        src.PickPathDistance,
        src.OperatorID
    FROM StagingWarehouseOperations src
    INNER JOIN DimTime dt ON DATEADD(MINUTE, 
        (DATEPART(MINUTE, src.OperationStartTime) / 15) * 15, 
        CAST(CAST(src.OperationStartTime AS DATE) AS DATETIME) + 
        CAST(CAST(DATEPART(HOUR, src.OperationStartTime) AS VARCHAR) + ':00' AS TIME)
    ) = dt.FullDateTime
    INNER JOIN DimGeography dg ON src.LocationID = dg.LocationID
    INNER JOIN DimProductGravity dp ON src.SKU = dp.SKU
    WHERE src.OperationStartTime > @LastLoadDateTime;
    
    RETURN @@ROWCOUNT;
END;
GO
```

3. **Create views for common analytics:**

```sql
CREATE VIEW vw_CrossFactKPI AS
SELECT 
    dt.FullDateTime,
    dg.LocationName,
    dp.SKU,
    dp.ProductName,
    dp.GravityScore,
    -- Warehouse metrics
    AVG(fwo.DurationMinutes) AS AvgWarehouseOperationMinutes,
    SUM(fwo.Quantity) AS TotalQuantityProcessed,
    AVG(fwo.DwellTimeHours) AS AvgDwellTimeHours,
    -- Fleet metrics (joined via bridge)
    AVG(fft.IdleTimeMinutes) AS AvgFleetIdleMinutes,
    AVG(fft.FuelConsumedLiters / NULLIF(fft.DistanceKM, 0)) AS AvgFuelEfficiency,
    -- Cross-fact KPI: Cost per unit delivered
    (SUM(fft.FuelConsumedLiters) * 1.5 + SUM(fwo.DurationMinutes) * 0.5) / 
        NULLIF(SUM(fwo.Quantity), 0) AS EstimatedCostPerUnit
FROM FactWarehouseOperations fwo
INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
INNER JOIN DimGeography dg ON fwo.LocationKey = dg.GeographyKey
INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
LEFT JOIN BridgeRouteWarehouse brw ON fwo.OperationKey = brw.OperationKey
LEFT JOIN FactFleetTrips fft ON brw.TripKey = fft.TripKey
GROUP BY dt.FullDateTime, dg.LocationName, dp.SKU, dp.ProductName, dp.GravityScore;
GO
```

### Power BI Configuration

1. **Open the Power BI template:**
   - Download `LogiFleet_Pulse_Master.pbit` from the repository
   - Open in Power BI Desktop
   - When prompted, enter your SQL Server connection string

2. **Configure data source connection:**

```powerquery
let
    Source = Sql.Database(
        Environment.GetEnvironmentVariable("SQL_SERVER_HOST"),
        Environment.GetEnvironmentVariable("SQL_DATABASE_NAME"),
        [Query="SELECT * FROM vw_CrossFactKPI WHERE FullDateTime >= DATEADD(DAY, -90, GETDATE())"]
    ),
    ChangeTypes = Table.TransformColumnTypes(Source,{
        {"FullDateTime", type datetime},
        {"AvgDwellTimeHours", type number},
        {"EstimatedCostPerUnit", Currency.Type}
    })
in
    ChangeTypes
```

3. **Set up scheduled refresh:**
   - Publish to Power BI Service
   - Configure gateway connection
   - Set refresh schedule (every 15 minutes recommended)

## Key Features & Usage

### Warehouse Gravity Zones Calculation

```sql
-- Update gravity scores based on recent activity
UPDATE DimProductGravity
SET 
    GravityScore = (
        VelocityScore * ValueMultiplier * (1.0 / NULLIF(FragilityIndex, 0))
    ),
    OptimalZone = CASE 
        WHEN GravityScore > 75 THEN 'Zone-A-Fast'
        WHEN GravityScore BETWEEN 40 AND 75 THEN 'Zone-B-Medium'
        ELSE 'Zone-C-Slow'
    END
FROM DimProductGravity dp
CROSS APPLY (
    SELECT 
        AVG(Quantity / NULLIF(DurationMinutes, 0)) * 100 AS VelocityScore,
        CASE 
            WHEN dp.ValueTier = 'High' THEN 1.5
            WHEN dp.ValueTier = 'Medium' THEN 1.0
            ELSE 0.7
        END AS ValueMultiplier
    FROM FactWarehouseOperations
    WHERE ProductKey = dp.ProductKey
        AND OperationStartTime >= DATEADD(DAY, -30, GETDATE())
) calc;
```

### Fleet Triage Score Calculation

```sql
-- Generate proactive maintenance queue
SELECT 
    VehicleID,
    TripCount,
    AvgIdleTimePercent,
    TotalDistanceKM,
    AvgLoadWeight,
    TriageScore,
    RevenueImpact,
    MaintenancePriority
FROM (
    SELECT 
        VehicleID,
        COUNT(*) AS TripCount,
        AVG(IdleTimeMinutes / NULLIF(DATEDIFF(MINUTE, TripStartTime, TripEndTime), 0) * 100) AS AvgIdleTimePercent,
        SUM(DistanceKM) AS TotalDistanceKM,
        AVG(LoadWeight) AS AvgLoadWeight,
        -- Triage score: higher = more urgent
        (
            (SUM(DistanceKM) / 1000 * 0.3) + -- Distance factor
            (AVG(IdleTimeMinutes) * 0.4) + -- Idle time factor
            (AVG(LoadWeight) / 1000 * 0.3) -- Load factor
        ) AS TriageScore,
        SUM(LoadWeight) * 0.05 AS RevenueImpact, -- Estimate $0.05 per kg
        CASE 
            WHEN AVG(IdleTimeMinutes) > 60 THEN 'Critical'
            WHEN AVG(IdleTimeMinutes) > 30 THEN 'High'
            ELSE 'Normal'
        END AS MaintenancePriority
    FROM FactFleetTrips
    WHERE TripStartTime >= DATEADD(DAY, -7, GETDATE())
    GROUP BY VehicleID
) ranked
ORDER BY TriageScore DESC;
```

### Cross-Fact Query: Dwell Time vs Fleet Idling

```sql
-- Find shipments with high warehouse dwell time AND high fleet idle time
SELECT 
    brw.ShipmentID,
    dg_warehouse.LocationName AS WarehouseName,
    dp.SKU,
    dp.ProductName,
    fwo.DwellTimeHours,
    fft.IdleTimeMinutes,
    fft.VehicleID,
    fft.DistanceKM,
    (fwo.DwellTimeHours * 2.5) + (fft.IdleTimeMinutes * 0.8) AS TotalDelayScore
FROM BridgeRouteWarehouse brw
INNER JOIN FactWarehouseOperations fwo ON brw.OperationKey = fwo.OperationKey
INNER JOIN FactFleetTrips fft ON brw.TripKey = fft.TripKey
INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
INNER JOIN DimGeography dg_warehouse ON fwo.LocationKey = dg_warehouse.GeographyKey
WHERE fwo.DwellTimeHours > 72 -- More than 3 days
    AND fft.IdleTimeMinutes > 45 -- More than 45 minutes idle
    AND fwo.OperationStartTime >= DATEADD(QUARTER, -1, GETDATE())
ORDER BY TotalDelayScore DESC;
```

## Configuration

### Environment Variables

Set these environment variables before deploying:

```bash
# SQL Server connection
export SQL_SERVER_HOST="your-sql-server.database.windows.net"
export SQL_DATABASE_NAME="LogiFleetPulse"
export SQL_USERNAME="logifleet_app"
export SQL_PASSWORD="${SQL_PASSWORD}" # Use secure secret management

# Data source APIs
export WMS_API_ENDPOINT="https://your-wms.example.com/api/v1"
export WMS_API_KEY="${WMS_API_KEY}"
export TELEMETRY_API_ENDPOINT="https://fleet-telemetry.example.com/api"
export TELEMETRY_API_KEY="${TELEMETRY_API_KEY}"

# External enrichment
export WEATHER_API_KEY="${WEATHER_API_KEY}"
export TRAFFIC_API_KEY="${TRAFFIC_API_KEY}"

# Alert configuration
export ALERT_EMAIL_RECIPIENTS="logistics-team@example.com"
export ALERT_SLACK_WEBHOOK="${SLACK_WEBHOOK_URL}"
```

### Automated Alerting Setup

```sql
CREATE PROCEDURE usp_CheckKPIThresholds
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    -- Check for excessive fleet idling
    IF EXISTS (
        SELECT 1 FROM FactFleetTrips
        WHERE TripStartTime >= DATEADD(HOUR, -24, GETDATE())
            AND IdleTimeMinutes / NULLIF(DATEDIFF(MINUTE, TripStartTime, TripEndTime), 0) > 0.15
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Fleet idling time exceeded 15% of trip duration in last 24 hours';
        -- Send notification (integrate with email/SMS service)
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = 'logistics-team@example.com',
            @subject = 'LogiFleet Pulse Alert',
            @body = @AlertMessage;
    END;
    
    -- Check for high dwell time on high-gravity items
    IF EXISTS (
        SELECT 1 FROM FactWarehouseOperations fwo
        INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
        WHERE fwo.DwellTimeHours > 48
            AND dp.GravityScore > 70
            AND fwo.OperationStartTime >= DATEADD(HOUR, -24, GETDATE())
    )
    BEGIN
        SET @AlertMessage = 'ALERT: High-gravity SKUs have dwell time > 48 hours';
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = 'warehouse-mgr@example.com',
            @subject = 'LogiFleet Pulse Alert - Warehouse',
            @body = @AlertMessage;
    END;
END;
GO

-- Schedule the alert check to run every 15 minutes
EXEC msdb.dbo.sp_add_job @job_name = 'LogiFleet_KPI_Monitor';
EXEC msdb.dbo.sp_add_jobstep 
    @job_name = 'LogiFleet_KPI_Monitor',
    @step_name = 'Check_Thresholds',
    @command = 'EXEC usp_CheckKPIThresholds';
EXEC msdb.dbo.sp_add_schedule 
    @schedule_name = 'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;
```

## Common Patterns

### Pattern 1: Time-Phased Dimension Lookup

```sql
-- Get the correct TimeKey for a timestamp (rounds to nearest 15-minute bucket)
DECLARE @EventTime DATETIME = '2026-07-15 14:37:22';

SELECT TimeKey 
FROM DimTime
WHERE FullDateTime = DATEADD(MINUTE, 
    (DATEPART(MINUTE, @EventTime) / 15) * 15, 
    CAST(CAST(@EventTime AS DATE) AS DATETIME) + 
    CAST(CAST(DATEPART(HOUR, @EventTime) AS VARCHAR) + ':00' AS TIME)
);
```

### Pattern 2: Role-Based Row-Level Security

```sql
-- Create security predicate function
CREATE FUNCTION dbo.fn_SecurityPredicate(@LocationName NVARCHAR(100))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS AccessResult
    WHERE @LocationName IN (
        SELECT LocationName FROM dbo.UserLocationAccess
        WHERE Username = USER_NAME()
    );
GO

-- Apply security policy
CREATE SECURITY POLICY WarehouseSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(LocationName) ON dbo.vw_CrossFactKPI
WITH (STATE = ON);
```

### Pattern 3: Temporal Elasticity Simulation

```sql
-- What-if analysis: Impact of 95% warehouse capacity
WITH SimulationParameters AS (
    SELECT 0.95 AS TargetCapacity, 10000 AS MaxWarehouseVolume
),
CurrentUtilization AS (
    SELECT 
        LocationKey,
        SUM(Quantity) AS CurrentVolume,
        (SELECT MaxWarehouseVolume FROM SimulationParameters) AS MaxVolume
    FROM FactWarehouseOperations
    WHERE OperationStartTime >= DATEADD(DAY, -7, GETDATE())
    GROUP BY LocationKey
)
SELECT 
    LocationKey,
    CurrentVolume,
    MaxVolume,
    CAST(CurrentVolume AS FLOAT) / MaxVolume AS CurrentUtilization,
    (SELECT TargetCapacity FROM SimulationParameters) AS TargetUtilization,
    -- Projected change in fleet trips needed
    CASE 
        WHEN (CAST(CurrentVolume AS FLOAT) / MaxVolume) < (SELECT TargetCapacity FROM SimulationParameters)
        THEN 'Increase fleet capacity by ' + 
             CAST(((SELECT TargetCapacity FROM SimulationParameters) - 
                   (CAST(CurrentVolume AS FLOAT) / MaxVolume)) * 100 AS NVARCHAR) + '%'
        ELSE 'Sufficient capacity'
    END AS Recommendation
FROM CurrentUtilization;
```

## Power BI DAX Measures

### Key Performance Indicators

```dax
// Total Cost Per Unit Delivered
TotalCostPerUnit = 
DIVIDE(
    SUM(FactFleetTrips[FuelConsumedLiters]) * 1.5 + 
    SUM(FactWarehouseOperations[DurationMinutes]) * 0.5,
    SUM(FactWarehouseOperations[Quantity]),
    0
)

// Fleet Efficiency Score (0-100)
FleetEfficiencyScore = 
VAR IdlePercent = DIVIDE(
    AVERAGE(FactFleetTrips[IdleTimeMinutes]),
    AVERAGE(DATEDIFF(FactFleetTrips[TripStartTime], FactFleetTrips[TripEndTime], MINUTE)),
    0
)
VAR FuelEfficiency = DIVIDE(
    AVERAGE(FactFleetTrips[DistanceKM]),
    AVERAGE(FactFleetTrips[FuelConsumedLiters]),
    0
)
RETURN (1 - IdlePercent) * 50 + (FuelEfficiency / 15) * 50

// Warehouse Velocity Index
WarehouseVelocityIndex = 
AVERAGEX(
    SUMMARIZE(
        FactWarehouseOperations,
        DimProductGravity[SKU],
        "UnitsPerHour", DIVIDE(
            SUM(FactWarehouseOperations[Quantity]),
            SUM(FactWarehouseOperations[DurationMinutes]) / 60,
            0
        )
    ),
    [UnitsPerHour]
)

// Cross-Fact Bottleneck Score
BottleneckScore = 
VAR HighDwellCount = CALCULATE(
    COUNTROWS(FactWarehouseOperations),
    FactWarehouseOperations[DwellTimeHours] > 72
)
VAR HighIdleCount = CALCULATE(
    COUNTROWS(FactFleetTrips),
    FactFleetTrips[IdleTimeMinutes] > 45
)
RETURN (HighDwellCount * 2) + (HighIdleCount * 1.5)
```

## Troubleshooting

### Issue: Slow query performance on cross-fact queries

**Solution:** Ensure proper indexing on foreign keys and implement table partitioning:

```sql
-- Partition FactWarehouseOperations by month
CREATE PARTITION FUNCTION pf_MonthlyPartition (DATETIME)
AS RANGE RIGHT FOR VALUES (
    '2026-01-01', '2026-02-01', '2026-03-01', '2026-04-01',
    '2026-05-01', '2026-06-01', '2026-07-01', '2026-08-01'
);

CREATE PARTITION SCHEME ps_MonthlyPartition
AS PARTITION pf_MonthlyPartition ALL TO ([PRIMARY]);

-- Rebuild table with partitioning
CREATE TABLE FactWarehouseOperations_Partitioned (
    -- Same schema as original
    OperationKey BIGINT NOT NULL,
    TimeKey INT NOT NULL,
    -- ... other columns
    OperationStartTime DATETIME NOT NULL
) ON ps_MonthlyPartition(OperationStartTime);
```

### Issue: Power BI refresh timeout

**Solution:** Implement incremental refresh in Power BI:

```powerquery
let
    Source = Sql.Database(
        Environment.GetEnvironmentVariable("SQL_SERVER_HOST"),
        Environment.GetEnvironmentVariable("SQL_DATABASE_NAME")
    ),
    FilteredData = Table.SelectRows(Source, each [FullDateTime] >= RangeStart and [FullDateTime] < RangeEnd)
in
    FilteredData
```

Then configure incremental refresh policy in Power BI:
- Refresh last 7 days
- Store data for 2 years

### Issue: Gravity score not updating

**Solution:** Verify the calculation logic and check for division by zero:

```sql
-- Debug gravity score calculation
SELECT 
    SKU,
    VelocityClass,
    ValueTier,
    FragilityIndex,
    GravityScore,
    CASE 
        WHEN FragilityIndex = 0 THEN 'ERROR: FragilityIndex is zero'
        WHEN VelocityClass IS NULL THEN 'ERROR: VelocityClass not set'
        ELSE 'OK'
    END AS ValidationStatus
FROM DimProductGravity
WHERE GravityScore IS NULL OR GravityScore = 0;
```

### Issue: Duplicate records in fact tables

**Solution:** Implement unique constraint and merge logic:

```sql
-- Add unique constraint to prevent duplicates
ALTER TABLE FactWarehouseOperations
ADD CONSTRAINT UQ_Operation UNIQUE (
    LocationKey, ProductKey, OperationStartTime, OperatorID
);

-- Deduplicate existing data
WITH DuplicatesCTE AS (
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY LocationKey, ProductKey, OperationStartTime, OperatorID
            ORDER BY OperationKey
        ) AS RowNum
    FROM FactWarehouseOperations
)
DELETE FROM DuplicatesCTE WHERE RowNum > 1;
```

## Integration Examples

### Integrate with External Weather API

```sql
-- Create stored procedure to enrich fleet data with weather
CREATE PROCEDURE usp_EnrichFleetDataWithWeather
AS
BEGIN
    -- Assume weather data is loaded into a staging table
    UPDATE fft
    SET fft.WeatherCondition = ws.Condition
    FROM FactFleetTrips fft
    INNER JOIN DimGeography dg ON fft.OriginLocationKey = dg.GeographyKey
    INNER JOIN WeatherStaging ws ON 
        ws.Latitude = dg.Latitude AND
        ws.Longitude = dg.Longitude AND
        ws.Timestamp BETWEEN fft.TripStartTime AND fft.TripEndTime
    WHERE fft.WeatherCondition IS NULL;
END;
GO
```

### Export KPIs to Excel via Power Automate

Power BI template includes a button trigger that exports data:

1. Create a Power Automate flow
2. Trigger: Power BI button click
3. Action: Get data from SQL view `vw_CrossFactKPI`
4. Action: Create Excel file in SharePoint
5. Action: Send email with attachment

## Performance Optimization

### Columnstore Index for Large Fact Tables

```sql
-- Create columnstore index for analytical queries
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactWarehouse_Columnstore
ON FactWarehouseOperations (
    TimeKey, LocationKey, ProductKey, Quantity, DurationMinutes, DwellTimeHours
);

CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactFleet_Columnstore
ON FactFleetTrips (
    TimeKey, OriginLocationKey, DestinationLocationKey, 
    DistanceKM, FuelConsumedLiters, IdleTimeMinutes
);
```

### Statistics Update Schedule

```sql
-- Create job to update statistics weekly
CREATE PROCEDURE usp_UpdateStatistics
AS
BEGIN
    UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
    UPDATE STATISTICS FactFleetTrips WITH FULLSCAN;
    UPDATE STATISTICS BridgeRouteWarehouse WITH FULLSCAN;
END;
GO
```

This skill provides comprehensive guidance for deploying and using the LogiFleet Pulse analytics platform, enabling AI coding agents to assist developers in building advanced supply chain intelligence systems.
