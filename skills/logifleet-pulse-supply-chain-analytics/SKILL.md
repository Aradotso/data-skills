---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing template for logistics, fleet, and supply chain intelligence with multi-fact star schema modeling
triggers:
  - "set up LogiFleet Pulse analytics"
  - "deploy supply chain data warehouse"
  - "configure Power BI logistics dashboard"
  - "implement warehouse gravity zones"
  - "create fleet analytics schema"
  - "build cross-modal supply chain reports"
  - "optimize logistics data model"
  - "configure real-time fleet tracking dashboard"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive **data warehousing and analytics template** for logistics operations, combining MS SQL Server backend with Power BI visualization. It provides:

- **Multi-fact star schema** linking warehouse operations, fleet telemetry, and supply chain data
- **Cross-fact KPI harmonization** for unified logistics intelligence
- **Warehouse Gravity Zones** spatial optimization framework
- **Real-time dashboards** refreshed every 15 minutes
- **Predictive bottleneck detection** using temporal patterns
- **Role-based access control** with row-level security

The platform integrates data from WMS, telematics, supplier portals, and external APIs into a unified semantic layer.

## Installation & Prerequisites

### Requirements
- **MS SQL Server 2019+** (Standard or Enterprise edition recommended)
- **Power BI Desktop** (latest version)
- **SQL Server Management Studio** (SSMS) or Azure Data Studio
- Sufficient storage for fact table partitioning (estimate 100GB+ for production scale)

### Initial Setup

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**
```sql
-- Connect to your SQL Server instance via SSMS
-- Run the schema creation script
:r schema/01_create_database.sql
:r schema/02_create_dimensions.sql
:r schema/03_create_facts.sql
:r schema/04_create_bridges.sql
:r schema/05_create_views.sql
:r schema/06_create_stored_procedures.sql
:r schema/07_create_indexes.sql
```

3. **Configure data sources:**
```json
// config_sample.json - rename to config.json and populate
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_USER}",
    "password": "${SQL_PASSWORD}"
  },
  "data_sources": {
    "wms_api": "${WMS_API_ENDPOINT}",
    "telematics_feed": "${TELEMATICS_API_URL}",
    "weather_api_key": "${WEATHER_API_KEY}"
  }
}
```

4. **Open Power BI template:**
```bash
# Open the .pbit template file
# When prompted, enter your SQL Server connection details
# Template location: powerbi/LogiFleet_Pulse_Master.pbit
```

## Core Database Schema

### Dimension Tables

**DimTime** - Time intelligence with 15-minute granularity:
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME2,
    Date DATE,
    Year INT,
    Quarter INT,
    Month INT,
    Week INT,
    DayOfWeek INT,
    Hour INT,
    Minute15Bucket INT,
    IsBusinessHour BIT,
    FiscalPeriod VARCHAR(10)
);

-- Create index for range queries
CREATE NONCLUSTERED INDEX IX_DimTime_DateTime 
ON DimTime(DateTime) INCLUDE (TimeKey);
```

**DimProductGravity** - Warehouse gravity zone assignments:
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL,
    ProductName NVARCHAR(200),
    Category NVARCHAR(100),
    GravityScore DECIMAL(5,2), -- Composite metric
    VelocityClass VARCHAR(20), -- Fast/Medium/Slow
    ValueClass VARCHAR(20),    -- High/Medium/Low
    FragilityScore INT,
    OptimalZone VARCHAR(10),
    LastRecalculated DATETIME2
);

-- Index for gravity-based queries
CREATE NONCLUSTERED INDEX IX_ProductGravity_Score 
ON DimProductGravity(GravityScore DESC, OptimalZone);
```

**DimGeography** - Hierarchical location structure:
```sql
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY,
    LocationCode VARCHAR(20),
    LocationName NVARCHAR(100),
    LocationType VARCHAR(20), -- Warehouse/Route/Customer
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    RegionKey INT,
    CountryKey INT,
    ParentGeographyKey INT
);
```

### Fact Tables

**FactWarehouseOperations** - Core warehouse activity:
```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    OperationType VARCHAR(20), -- Receiving/Putaway/Picking/Packing/Shipping
    Quantity INT,
    DwellTimeMinutes INT,
    ProcessingTimeSeconds INT,
    OperatorID INT,
    ZoneCode VARCHAR(10),
    BatchNumber VARCHAR(50),
    TransactionTimestamp DATETIME2
);

-- Partitioning by month for performance
CREATE PARTITION FUNCTION PF_WarehouseOps (DATETIME2)
AS RANGE RIGHT FOR VALUES 
('2025-01-01', '2025-02-01', '2025-03-01', '2025-04-01');

-- Clustered index on time dimension
CREATE CLUSTERED INDEX CIX_FactWarehouseOps_Time 
ON FactWarehouseOperations(TimeKey, OperationKey);
```

**FactFleetTrips** - Vehicle telemetry and routing:
```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TripID VARCHAR(50) NOT NULL,
    VehicleKey INT NOT NULL,
    DriverKey INT NOT NULL,
    StartTimeKey INT NOT NULL,
    EndTimeKey INT,
    OriginGeographyKey INT,
    DestinationGeographyKey INT,
    PlannedDistanceKm DECIMAL(10,2),
    ActualDistanceKm DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadWeightKg DECIMAL(10,2),
    DelayMinutes INT,
    DelayReason VARCHAR(100),
    TripStatus VARCHAR(20)
);

-- Index for route analysis
CREATE NONCLUSTERED INDEX IX_FleetTrips_Route 
ON FactFleetTrips(OriginGeographyKey, DestinationGeographyKey)
INCLUDE (ActualDistanceKm, FuelConsumedLiters, IdleTimeMinutes);
```

**FactCrossDock** - Direct transfers without storage:
```sql
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    GeographyKey INT,
    Quantity INT,
    DockDwellMinutes INT,
    TransferOperatorID INT
);
```

## Key Stored Procedures

### Incremental Data Loading

```sql
CREATE PROCEDURE usp_LoadWarehouseOperations
    @StartDateTime DATETIME2,
    @EndDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Merge from staging to fact table
    MERGE FactWarehouseOperations AS target
    USING (
        SELECT 
            dt.TimeKey,
            dp.ProductKey,
            dg.GeographyKey,
            stg.OperationType,
            stg.Quantity,
            DATEDIFF(MINUTE, stg.StartTime, stg.EndTime) AS DwellTimeMinutes,
            stg.ProcessingTimeSeconds,
            stg.OperatorID,
            stg.ZoneCode,
            stg.BatchNumber,
            stg.TransactionTimestamp
        FROM StagingWarehouseOps stg
        INNER JOIN DimTime dt ON stg.TransactionTimestamp = dt.DateTime
        INNER JOIN DimProductGravity dp ON stg.SKU = dp.SKU
        INNER JOIN DimGeography dg ON stg.LocationCode = dg.LocationCode
        WHERE stg.TransactionTimestamp BETWEEN @StartDateTime AND @EndDateTime
    ) AS source
    ON target.BatchNumber = source.BatchNumber 
        AND target.TransactionTimestamp = source.TransactionTimestamp
    WHEN NOT MATCHED THEN
        INSERT (TimeKey, ProductKey, GeographyKey, OperationType, Quantity,
                DwellTimeMinutes, ProcessingTimeSeconds, OperatorID, ZoneCode,
                BatchNumber, TransactionTimestamp)
        VALUES (source.TimeKey, source.ProductKey, source.GeographyKey, 
                source.OperationType, source.Quantity, source.DwellTimeMinutes,
                source.ProcessingTimeSeconds, source.OperatorID, source.ZoneCode,
                source.BatchNumber, source.TransactionTimestamp);
    
    -- Log the load
    INSERT INTO ETL_LoadLog (ProcedureName, StartTime, EndTime, RowsAffected)
    VALUES ('usp_LoadWarehouseOperations', @StartDateTime, @EndDateTime, @@ROWCOUNT);
END;
```

### Gravity Score Calculation

```sql
CREATE PROCEDURE usp_RecalculateProductGravity
    @LookbackDays INT = 90
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Update gravity scores based on recent activity
    WITH ActivityMetrics AS (
        SELECT 
            ProductKey,
            COUNT(*) AS PickCount,
            AVG(CAST(ProcessingTimeSeconds AS FLOAT)) AS AvgProcessingTime,
            SUM(Quantity) AS TotalVolume
        FROM FactWarehouseOperations
        WHERE OperationType = 'Picking'
            AND TransactionTimestamp >= DATEADD(DAY, -@LookbackDays, GETDATE())
        GROUP BY ProductKey
    ),
    GravityCalculation AS (
        SELECT 
            am.ProductKey,
            -- Weighted score: 40% velocity, 30% value, 30% fragility
            (
                (CASE 
                    WHEN am.PickCount > PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY am.PickCount) OVER() THEN 40
                    WHEN am.PickCount > PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY am.PickCount) OVER() THEN 25
                    ELSE 10
                END) +
                (CASE 
                    WHEN dp.ValueClass = 'High' THEN 30
                    WHEN dp.ValueClass = 'Medium' THEN 20
                    ELSE 10
                END) +
                (dp.FragilityScore * 3)
            ) AS NewGravityScore,
            CASE 
                WHEN am.PickCount > PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY am.PickCount) OVER() THEN 'Fast'
                WHEN am.PickCount > PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY am.PickCount) OVER() THEN 'Medium'
                ELSE 'Slow'
            END AS NewVelocityClass
        FROM ActivityMetrics am
        INNER JOIN DimProductGravity dp ON am.ProductKey = dp.ProductKey
    )
    UPDATE dp
    SET 
        dp.GravityScore = gc.NewGravityScore,
        dp.VelocityClass = gc.NewVelocityClass,
        dp.OptimalZone = CASE 
            WHEN gc.NewGravityScore >= 70 THEN 'A'
            WHEN gc.NewGravityScore >= 50 THEN 'B'
            WHEN gc.NewGravityScore >= 30 THEN 'C'
            ELSE 'D'
        END,
        dp.LastRecalculated = GETDATE()
    FROM DimProductGravity dp
    INNER JOIN GravityCalculation gc ON dp.ProductKey = gc.ProductKey;
END;
```

## Cross-Fact KPI Queries

### Dwell Time vs Fleet Idle Time Correlation

```sql
-- Find products with high warehouse dwell correlated with fleet delays
WITH WarehouseDwell AS (
    SELECT 
        fwo.ProductKey,
        dp.SKU,
        dp.ProductName,
        AVG(fwo.DwellTimeMinutes) AS AvgDwellMinutes,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations fwo
    INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
    WHERE fwo.OperationType IN ('Receiving', 'Putaway')
        AND fwo.TimeKey >= (SELECT TimeKey FROM DimTime WHERE Date = DATEADD(DAY, -30, CAST(GETDATE() AS DATE)))
    GROUP BY fwo.ProductKey, dp.SKU, dp.ProductName
),
FleetImpact AS (
    SELECT 
        dp.ProductKey,
        AVG(fft.IdleTimeMinutes) AS AvgFleetIdleMinutes,
        AVG(fft.DelayMinutes) AS AvgDelayMinutes
    FROM FactFleetTrips fft
    INNER JOIN FactWarehouseOperations fwo ON fft.OriginGeographyKey = fwo.GeographyKey
    INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
    WHERE fft.StartTimeKey >= (SELECT TimeKey FROM DimTime WHERE Date = DATEADD(DAY, -30, CAST(GETDATE() AS DATE)))
    GROUP BY dp.ProductKey
)
SELECT 
    wd.SKU,
    wd.ProductName,
    wd.AvgDwellMinutes,
    fi.AvgFleetIdleMinutes,
    fi.AvgDelayMinutes,
    -- Correlation strength indicator
    CASE 
        WHEN wd.AvgDwellMinutes > 120 AND fi.AvgDelayMinutes > 30 THEN 'High Risk'
        WHEN wd.AvgDwellMinutes > 60 AND fi.AvgDelayMinutes > 15 THEN 'Medium Risk'
        ELSE 'Low Risk'
    END AS CorrelationRisk
FROM WarehouseDwell wd
LEFT JOIN FleetImpact fi ON wd.ProductKey = fi.ProductKey
WHERE wd.AvgDwellMinutes > 60
ORDER BY wd.AvgDwellMinutes DESC, fi.AvgDelayMinutes DESC;
```

### Route Density by Product Gravity

```sql
-- Analyze optimal route density for high-gravity products
SELECT 
    dg_origin.LocationName AS OriginWarehouse,
    dg_dest.LocationName AS DestinationZone,
    dp.OptimalZone AS ProductGravityZone,
    COUNT(DISTINCT fft.TripID) AS TripCount,
    SUM(fft.LoadWeightKg) AS TotalWeightKg,
    AVG(fft.FuelConsumedLiters) AS AvgFuelLiters,
    AVG(fft.ActualDistanceKm / NULLIF(fft.PlannedDistanceKm, 0)) AS RouteEfficiencyRatio,
    SUM(CASE WHEN fft.DelayMinutes > 30 THEN 1 ELSE 0 END) AS DelayedTrips
FROM FactFleetTrips fft
INNER JOIN DimGeography dg_origin ON fft.OriginGeographyKey = dg_origin.GeographyKey
INNER JOIN DimGeography dg_dest ON fft.DestinationGeographyKey = dg_dest.GeographyKey
INNER JOIN FactWarehouseOperations fwo ON fft.OriginGeographyKey = fwo.GeographyKey
INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
WHERE dp.GravityScore >= 70 -- High-gravity products only
    AND fft.TripStatus = 'Completed'
    AND fft.StartTimeKey >= (SELECT TimeKey FROM DimTime WHERE Date = DATEADD(DAY, -90, CAST(GETDATE() AS DATE)))
GROUP BY dg_origin.LocationName, dg_dest.LocationName, dp.OptimalZone
HAVING COUNT(DISTINCT fft.TripID) >= 5
ORDER BY AvgFuelLiters ASC, RouteEfficiencyRatio DESC;
```

## Power BI Configuration

### Connection String Template

```powerquery
// M Query for Power BI data source
let
    Source = Sql.Database(
        Environment.GetEnvironmentVariable("SQL_SERVER_HOST"),
        "LogiFleetPulse",
        [
            Query = "SELECT * FROM vw_UnifiedLogisticsView",
            CommandTimeout = #duration(0, 0, 5, 0)
        ]
    )
in
    Source
```

### DAX Measures for Cross-Fact Analysis

**Composite KPI: Logistics Efficiency Score**
```dax
Logistics Efficiency Score = 
VAR WarehouseEfficiency = 
    DIVIDE(
        SUM(FactWarehouseOperations[Quantity]),
        SUM(FactWarehouseOperations[DwellTimeMinutes]),
        0
    )
VAR FleetEfficiency = 
    DIVIDE(
        SUM(FactFleetTrips[PlannedDistanceKm]),
        SUM(FactFleetTrips[ActualDistanceKm]),
        0
    )
VAR WeightedScore = (WarehouseEfficiency * 0.6) + (FleetEfficiency * 0.4)
RETURN
    WeightedScore * 100
```

**Predictive Bottleneck Index**
```dax
Bottleneck Risk Index = 
VAR CurrentDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR HistoricalAvg = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        DATEADD(DimTime[Date], -30, DAY)
    )
VAR DwellDelta = DIVIDE(CurrentDwell - HistoricalAvg, HistoricalAvg, 0)
VAR FleetIdleRate = 
    DIVIDE(
        SUM(FactFleetTrips[IdleTimeMinutes]),
        SUM(FactFleetTrips[ActualDistanceKm]) * 60,
        0
    )
VAR RiskScore = (DwellDelta * 70) + (FleetIdleRate * 30)
RETURN
    IF(RiskScore > 0.5, "High", IF(RiskScore > 0.25, "Medium", "Low"))
```

### Automated Refresh Configuration

```powershell
# PowerShell script for scheduled refresh via Power BI REST API
$tenantId = $env:POWERBI_TENANT_ID
$clientId = $env:POWERBI_CLIENT_ID
$clientSecret = $env:POWERBI_CLIENT_SECRET
$workspaceId = $env:POWERBI_WORKSPACE_ID
$datasetId = $env:POWERBI_DATASET_ID

# Get access token
$tokenBody = @{
    grant_type    = "client_credentials"
    client_id     = $clientId
    client_secret = $clientSecret
    resource      = "https://analysis.windows.net/powerbi/api"
}
$tokenResponse = Invoke-RestMethod -Method Post -Uri "https://login.microsoftonline.com/$tenantId/oauth2/token" -Body $tokenBody

# Trigger refresh
$headers = @{
    "Authorization" = "Bearer $($tokenResponse.access_token)"
}
Invoke-RestMethod -Method Post -Uri "https://api.powerbi.com/v1.0/myorg/groups/$workspaceId/datasets/$datasetId/refreshes" -Headers $headers
```

## Automated Alerting

### Threshold-Based Alert Procedure

```sql
CREATE PROCEDURE usp_MonitorKPIThresholds
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertMessages TABLE (
        AlertType VARCHAR(50),
        Severity VARCHAR(20),
        Message NVARCHAR(500),
        MetricValue DECIMAL(18,2),
        Threshold DECIMAL(18,2)
    );
    
    -- Check fleet idle time threshold (> 15% of trip duration)
    INSERT INTO @AlertMessages
    SELECT 
        'Fleet Idle Time' AS AlertType,
        'High' AS Severity,
        CONCAT('Vehicle ', v.VehicleID, ' exceeded idle threshold on route ', 
               fft.TripID, ': ', CAST(fft.IdleTimeMinutes AS VARCHAR), ' minutes') AS Message,
        CAST(fft.IdleTimeMinutes AS DECIMAL(18,2)) AS MetricValue,
        CAST(DATEDIFF(MINUTE, dt_start.DateTime, dt_end.DateTime) * 0.15 AS DECIMAL(18,2)) AS Threshold
    FROM FactFleetTrips fft
    INNER JOIN DimTime dt_start ON fft.StartTimeKey = dt_start.TimeKey
    INNER JOIN DimTime dt_end ON fft.EndTimeKey = dt_end.TimeKey
    INNER JOIN DimVehicle v ON fft.VehicleKey = v.VehicleKey
    WHERE fft.IdleTimeMinutes > (DATEDIFF(MINUTE, dt_start.DateTime, dt_end.DateTime) * 0.15)
        AND dt_start.DateTime >= DATEADD(HOUR, -4, GETDATE());
    
    -- Check warehouse dwell time (> 72 hours for non-bulk items)
    INSERT INTO @AlertMessages
    SELECT 
        'Warehouse Dwell' AS AlertType,
        'Medium' AS Severity,
        CONCAT('SKU ', dp.SKU, ' exceeded 72h dwell in zone ', fwo.ZoneCode,
               ': ', CAST(fwo.DwellTimeMinutes / 60.0 AS DECIMAL(10,1)), ' hours') AS Message,
        CAST(fwo.DwellTimeMinutes AS DECIMAL(18,2)) AS MetricValue,
        4320 AS Threshold -- 72 hours in minutes
    FROM FactWarehouseOperations fwo
    INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
    WHERE fwo.DwellTimeMinutes > 4320
        AND dp.VelocityClass <> 'Slow'
        AND fwo.TransactionTimestamp >= DATEADD(HOUR, -24, GETDATE());
    
    -- Send alerts if any found
    IF EXISTS (SELECT 1 FROM @AlertMessages)
    BEGIN
        -- Log to alert table
        INSERT INTO AlertLog (AlertType, Severity, Message, MetricValue, Threshold, AlertTimestamp)
        SELECT AlertType, Severity, Message, MetricValue, Threshold, GETDATE()
        FROM @AlertMessages;
        
        -- Send email notifications (requires Database Mail configured)
        DECLARE @EmailBody NVARCHAR(MAX);
        SET @EmailBody = (
            SELECT 
                CONCAT(Severity, ' - ', AlertType, ': ', Message, CHAR(13))
            FROM @AlertMessages
            FOR XML PATH('')
        );
        
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = 'logistics-ops@yourcompany.com',
            @subject = 'LogiFleet Pulse - KPI Threshold Alerts',
            @body = @EmailBody;
    END;
END;
```

## Row-Level Security Implementation

```sql
-- Create security role table
CREATE TABLE SecurityRoles (
    RoleID INT PRIMARY KEY,
    RoleName VARCHAR(50),
    GeographyFilter VARCHAR(200), -- JSON or delimited list
    CanViewFleetData BIT,
    CanViewFinancials BIT
);

-- Create RLS function
CREATE FUNCTION dbo.fn_SecurityPredicate(@GeographyKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN (
    SELECT 1 AS AccessAllowed
    FROM dbo.SecurityRoles sr
    WHERE sr.RoleName = USER_NAME()
        AND (
            sr.GeographyFilter IS NULL -- Full access
            OR @GeographyKey IN (SELECT value FROM STRING_SPLIT(sr.GeographyFilter, ','))
        )
);

-- Apply security policy to fact tables
CREATE SECURITY POLICY FleetSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(OriginGeographyKey) ON FactFleetTrips,
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(GeographyKey) ON FactWarehouseOperations
WITH (STATE = ON);
```

## Common Patterns

### Pattern 1: Time-Phased Dimension Join

```sql
-- Always join facts to time dimension for temporal analysis
SELECT 
    dt.Date,
    dt.DayOfWeek,
    dp.OptimalZone,
    COUNT(*) AS OperationCount,
    AVG(fwo.ProcessingTimeSeconds) AS AvgProcessingTime
FROM FactWarehouseOperations fwo
INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
WHERE dt.Date BETWEEN '2025-01-01' AND '2025-01-31'
    AND dt.IsBusinessHour = 1
GROUP BY dt.Date, dt.DayOfWeek, dp.OptimalZone
ORDER BY dt.Date, dp.OptimalZone;
```

### Pattern 2: Bridge Table for Many-to-Many

```sql
-- Use bridge tables to resolve complex relationships
-- Example: Routes serving multiple warehouse zones
SELECT 
    r.RouteID,
    STRING_AGG(wz.ZoneCode, ', ') AS ServedZones,
    COUNT(DISTINCT wz.ZoneCode) AS ZoneCount
FROM DimRoute r
INNER JOIN BridgeRouteZone brz ON r.RouteKey = brz.RouteKey
INNER JOIN DimWarehouseZone wz ON brz.ZoneKey = wz.ZoneKey
GROUP BY r.RouteID
HAVING COUNT(DISTINCT wz.ZoneCode) > 1;
```

### Pattern 3: Temporal Elasticity Simulation

```sql
-- Simulate "what-if" scenario: 95% warehouse capacity utilization
WITH CurrentState AS (
    SELECT 
        GeographyKey,
        SUM(Quantity) AS CurrentVolume,
        MAX(dg.StorageCapacity) AS MaxCapacity
    FROM FactWarehouseOperations fwo
    INNER JOIN DimGeography dg ON fwo.GeographyKey = dg.GeographyKey
    WHERE TransactionTimestamp >= DATEADD(DAY, -7, GETDATE())
    GROUP BY GeographyKey
),
SimulatedLoad AS (
    SELECT 
        GeographyKey,
        CurrentVolume,
        MaxCapacity,
        CurrentVolume / NULLIF(MaxCapacity, 0) AS CurrentUtilization,
        (MaxCapacity * 0.95) AS TargetVolume,
        ((MaxCapacity * 0.95) - CurrentVolume) AS AdditionalCapacity
    FROM CurrentState
)
SELECT 
    dg.LocationName,
    sl.CurrentUtilization,
    sl.AdditionalCapacity,
    -- Estimate impact on processing time (non-linear relationship)
    CASE 
        WHEN sl.CurrentUtilization < 0.80 THEN 1.0
        WHEN sl.CurrentUtilization < 0.90 THEN 1.15
        ELSE 1.35
    END AS EstimatedProcessingMultiplier
FROM SimulatedLoad sl
INNER JOIN DimGeography dg ON sl.GeographyKey = dg.GeographyKey
ORDER BY sl.CurrentUtilization DESC;
```

## Troubleshooting

### Issue: Slow Cross-Fact Queries

**Symptom:** Queries joining multiple fact tables timeout or take > 30 seconds

**Solution:**
```sql
-- Ensure covering indexes exist on join keys
CREATE NONCLUSTERED INDEX IX_FactWarehouseOps_ProductTime
ON FactWarehouseOperations(ProductKey, TimeKey)
INCLUDE (Quantity, DwellTimeMinutes);

-- Use indexed views for common aggregations
CREATE VIEW vw_DailyWarehouseMetrics
WITH SCHEMABINDING
AS
SELECT 
    fwo.GeographyKey,
    dt.Date,
    COUNT_BIG(*) AS OperationCount,
    SUM(fwo.Quantity) AS TotalQuantity,
    AVG(fwo.DwellTimeMinutes) AS AvgDwellMinutes
FROM dbo.FactWarehouseOperations fwo
INNER JOIN dbo.DimTime dt ON fwo.TimeKey = dt.TimeKey
GROUP BY fwo.GeographyKey, dt.Date;

CREATE UNIQUE CLUSTERED INDEX UCIX_DailyMetrics
ON vw_DailyWarehouseMetrics(GeographyKey, Date);
```

### Issue: Power BI Dataset Refresh Failures

**Symptom:** Scheduled refresh fails with timeout errors

**Solution:**
```sql
-- Implement incremental refresh with watermark table
CREATE TABLE ETL_Watermark (
    TableName VARCHAR(100) PRIMARY KEY,
    LastRefreshTimestamp DATETIME2
);

-- Modify load procedures to use watermark
ALTER PROCEDURE usp_LoadWarehouseOperations
AS
BEGIN
    DECLARE @LastRefresh DATETIME2;
    SELECT @LastRefresh = LastRefreshTimestamp 
    FROM ETL_Watermark 
    WHERE TableName = 'FactWarehouseOperations';
    
    -- Load only new records
    INSERT INTO FactWarehouseOperations (...)
    SELECT ... FROM StagingWarehouseOps
    WHERE TransactionTimestamp > @LastRefresh;
    
    -- Update watermark
    UPDATE ETL_Watermark
    SET LastRefreshTimestamp = GETDATE()
    WHERE TableName = 'FactWarehouseOperations';
END;
```

### Issue: Gravity Score Calculation Inconsistencies

**Symptom:** Products frequently change gravity zones, causing operational confusion

**Solution:**
```sql
-- Add damping factor to prevent oscillation
ALTER PROCEDURE usp_RecalculateProductGravity
AS
BEGIN
    -- Use exponential moving average instead of raw scores
    UPDATE dp
    SET dp.GravityScore = (dp.GravityScore * 0.7) + (gc.New
