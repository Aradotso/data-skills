---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehouse for logistics, fleet tracking, and supply chain intelligence with multi-fact star schema
triggers:
  - set up logistics analytics dashboard
  - deploy supply chain data warehouse
  - configure fleet tracking in power bi
  - implement warehouse operations analytics
  - create multi-fact star schema for logistics
  - build real-time supply chain dashboard
  - optimize warehouse gravity zones
  - setup logifleet pulse analytics
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is an advanced logistics and supply chain intelligence platform built on MS SQL Server and Power BI. It provides a unified semantic layer that connects warehouse operations, fleet telemetry, inventory management, and external data sources through a multi-fact star schema architecture.

**Key Capabilities:**
- Real-time logistics dashboards with 15-minute refresh intervals
- Cross-fact KPI harmonization (warehouse + fleet + inventory)
- Predictive bottleneck detection using temporal elasticity modeling
- Warehouse Gravity Zones™ for spatial optimization
- Adaptive fleet triage engine with proactive maintenance scoring
- Role-based access control with row-level security

## Installation

### Prerequisites

- MS SQL Server 2019 or later
- Power BI Desktop (latest version)
- Access to data sources: WMS, telematics/GPS feeds, ERP systems

### Database Setup

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy SQL schema:**
```sql
-- Connect to your SQL Server instance
-- Run the schema deployment script
:r schema/01_create_database.sql
:r schema/02_create_dimensions.sql
:r schema/03_create_facts.sql
:r schema/04_create_views.sql
:r schema/05_create_procedures.sql
:r schema/06_create_indexes.sql
```

3. **Configure data sources:**
```json
// config.json (create from config_sample.json)
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_USER}",
    "password": "${SQL_PASSWORD}"
  },
  "data_sources": {
    "wms_connection": "${WMS_CONNECTION_STRING}",
    "telematics_api": "${TELEMATICS_API_URL}",
    "weather_api_key": "${WEATHER_API_KEY}"
  },
  "refresh_interval_minutes": 15
}
```

4. **Import Power BI template:**
- Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
- Update connection strings to point to your SQL Server
- Configure data refresh schedule

## Core Data Model

### Dimension Tables

```sql
-- DimTime: 15-minute granularity time dimension
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME2,
    Date DATE,
    TimeSlot15Min TIME,
    HourOfDay INT,
    DayOfWeek VARCHAR(10),
    FiscalPeriod VARCHAR(7),
    IsWeekend BIT,
    IsHoliday BIT
);

-- DimGeography: Hierarchical location dimension
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY,
    LocationID VARCHAR(50),
    LocationName VARCHAR(100),
    LocationType VARCHAR(20), -- Warehouse, Route Node, Cross-Dock
    ParentLocationKey INT,
    Continent VARCHAR(50),
    Country VARCHAR(50),
    Region VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6)
);

-- DimProductGravity: Product classification with gravity scoring
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50),
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    VelocityScore DECIMAL(5,2), -- Pick frequency
    ValueScore DECIMAL(5,2), -- Unit value
    FragilityScore DECIMAL(5,2), -- Handling sensitivity
    GravityScore AS (VelocityScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2),
    GravityZone VARCHAR(20) -- High, Medium, Low
);

-- DimSupplierReliability: Supplier performance metrics
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY,
    SupplierID VARCHAR(50),
    SupplierName VARCHAR(200),
    LeadTimeAvgDays DECIMAL(5,2),
    LeadTimeVariance DECIMAL(5,2),
    DefectPercentage DECIMAL(5,4),
    ComplianceScore DECIMAL(5,2),
    ReliabilityTier VARCHAR(10) -- A, B, C
);
```

### Fact Tables

```sql
-- FactWarehouseOperations: Core warehouse activity metrics
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType VARCHAR(20), -- Receiving, Putaway, Picking, Packing, Shipping
    OperationID VARCHAR(50),
    QuantityHandled INT,
    DurationMinutes DECIMAL(8,2),
    DwellTimeHours DECIMAL(8,2), -- Time in storage
    ZoneID VARCHAR(20),
    HandlerEmployeeID VARCHAR(50),
    OperationCost DECIMAL(10,2)
);

-- FactFleetTrips: Fleet movement and performance
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID VARCHAR(50),
    DriverID VARCHAR(50),
    TripID VARCHAR(50),
    DistanceKM DECIMAL(8,2),
    DurationMinutes DECIMAL(8,2),
    IdleTimeMinutes DECIMAL(8,2),
    FuelConsumedLiters DECIMAL(8,2),
    LoadWeightKG DECIMAL(10,2),
    DelayMinutes DECIMAL(8,2),
    DelayReason VARCHAR(100),
    TripCost DECIMAL(10,2)
);

-- FactCrossDock: Direct transfer operations
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    InboundTripID VARCHAR(50),
    OutboundTripID VARCHAR(50),
    QuantityTransferred INT,
    DockDwellMinutes DECIMAL(8,2),
    TransferCost DECIMAL(10,2)
);
```

## Key Stored Procedures

### Incremental Data Loading

```sql
-- Incremental load for warehouse operations
CREATE PROCEDURE usp_LoadWarehouseOperations
    @StartDateTime DATETIME2,
    @EndDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        OperationID, QuantityHandled, DurationMinutes,
        DwellTimeHours, ZoneID, HandlerEmployeeID, OperationCost
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        src.OperationType,
        src.OperationID,
        src.Quantity,
        DATEDIFF(MINUTE, src.StartTime, src.EndTime),
        DATEDIFF(HOUR, src.StartTime, COALESCE(src.ShippedTime, GETDATE())),
        src.ZoneID,
        src.EmployeeID,
        src.CalculatedCost
    FROM ExternalWarehouseData src
    INNER JOIN DimTime t ON CAST(src.StartTime AS DATE) = t.Date 
        AND DATEPART(HOUR, src.StartTime) = t.HourOfDay
        AND (DATEPART(MINUTE, src.StartTime) / 15) = (DATEPART(MINUTE, t.TimeSlot15Min) / 15)
    INNER JOIN DimGeography g ON src.LocationID = g.LocationID
    INNER JOIN DimProductGravity p ON src.SKU = p.SKU
    WHERE src.StartTime BETWEEN @StartDateTime AND @EndDateTime
        AND NOT EXISTS (
            SELECT 1 FROM FactWarehouseOperations 
            WHERE OperationID = src.OperationID
        );
END;
GO

-- Incremental load for fleet trips
CREATE PROCEDURE usp_LoadFleetTrips
    @StartDateTime DATETIME2,
    @EndDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactFleetTrips (
        TimeKey, OriginGeographyKey, DestinationGeographyKey,
        VehicleID, DriverID, TripID, DistanceKM, DurationMinutes,
        IdleTimeMinutes, FuelConsumedLiters, LoadWeightKG,
        DelayMinutes, DelayReason, TripCost
    )
    SELECT 
        t.TimeKey,
        go.GeographyKey AS OriginKey,
        gd.GeographyKey AS DestinationKey,
        src.VehicleID,
        src.DriverID,
        src.TripID,
        src.Distance,
        DATEDIFF(MINUTE, src.DepartureTime, src.ArrivalTime),
        src.IdleTime,
        src.FuelUsed,
        src.LoadWeight,
        CASE WHEN src.ExpectedArrival < src.ArrivalTime 
            THEN DATEDIFF(MINUTE, src.ExpectedArrival, src.ArrivalTime)
            ELSE 0 END,
        src.DelayReason,
        src.TotalCost
    FROM ExternalFleetData src
    INNER JOIN DimTime t ON CAST(src.DepartureTime AS DATE) = t.Date
    INNER JOIN DimGeography go ON src.OriginLocationID = go.LocationID
    INNER JOIN DimGeography gd ON src.DestinationLocationID = gd.LocationID
    WHERE src.DepartureTime BETWEEN @StartDateTime AND @EndDateTime
        AND NOT EXISTS (
            SELECT 1 FROM FactFleetTrips WHERE TripID = src.TripID
        );
END;
GO
```

### Cross-Fact KPI Queries

```sql
-- View: Fleet efficiency correlated with warehouse dwell time
CREATE VIEW vw_FleetEfficiencyByDwellTime AS
SELECT 
    t.Date,
    g.LocationName AS Warehouse,
    p.Category AS ProductCategory,
    AVG(w.DwellTimeHours) AS AvgDwellTimeHours,
    AVG(f.IdleTimeMinutes / NULLIF(f.DurationMinutes, 0)) AS FleetIdleTimeRatio,
    AVG(f.FuelConsumedLiters / NULLIF(f.DistanceKM, 0)) AS FuelEfficiency,
    COUNT(DISTINCT f.TripKey) AS TripCount,
    SUM(f.TripCost) AS TotalFleetCost
FROM FactWarehouseOperations w
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
INNER JOIN FactFleetTrips f ON f.OriginGeographyKey = w.GeographyKey 
    AND f.TimeKey = w.TimeKey
WHERE w.OperationType = 'Shipping'
GROUP BY t.Date, g.LocationName, p.Category;
GO

-- Stored Procedure: Predictive bottleneck detection
CREATE PROCEDURE usp_PredictBottlenecks
    @ForecastHorizonHours INT = 24,
    @BottleneckThresholdPct DECIMAL(5,2) = 85.0
AS
BEGIN
    -- Analyze current capacity utilization trends
    WITH CapacityTrends AS (
        SELECT 
            g.GeographyKey,
            g.LocationName,
            t.HourOfDay,
            COUNT(*) AS OperationCount,
            AVG(DurationMinutes) AS AvgDuration,
            MAX(DurationMinutes) AS MaxDuration,
            -- Capacity utilization based on historical max
            (COUNT(*) * AVG(DurationMinutes)) / NULLIF(MAX(DurationMinutes), 0) AS UtilizationScore
        FROM FactWarehouseOperations w
        INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
        INNER JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
        WHERE t.DateTime >= DATEADD(HOUR, -168, GETDATE()) -- Last 7 days
        GROUP BY g.GeographyKey, g.LocationName, t.HourOfDay
    )
    SELECT 
        LocationName,
        HourOfDay,
        UtilizationScore,
        CASE 
            WHEN UtilizationScore >= @BottleneckThresholdPct THEN 'High Risk'
            WHEN UtilizationScore >= 70 THEN 'Medium Risk'
            ELSE 'Low Risk'
        END AS BottleneckRisk,
        AvgDuration,
        OperationCount
    FROM CapacityTrends
    WHERE UtilizationScore >= 70
    ORDER BY UtilizationScore DESC;
END;
GO
```

### Automated Alerting

```sql
-- Alert configuration table
CREATE TABLE AlertThresholds (
    AlertID INT PRIMARY KEY IDENTITY,
    MetricName VARCHAR(100),
    ThresholdValue DECIMAL(10,2),
    ComparisonOperator VARCHAR(10), -- >, <, =, >=, <=
    AlertPriority VARCHAR(20), -- Critical, High, Medium, Low
    RecipientList VARCHAR(500), -- Comma-separated emails
    IsActive BIT DEFAULT 1
);

-- Sample alert thresholds
INSERT INTO AlertThresholds (MetricName, ThresholdValue, ComparisonOperator, AlertPriority, RecipientList)
VALUES 
    ('FleetIdleTimePercentage', 15.0, '>', 'High', '${FLEET_MANAGER_EMAIL}'),
    ('WarehouseDwellTimeHours', 72.0, '>', 'Critical', '${WAREHOUSE_MANAGER_EMAIL}'),
    ('FuelEfficiencyDropPercentage', 10.0, '>', 'Medium', '${OPERATIONS_EMAIL}'),
    ('CrossDockDwellMinutes', 30.0, '>', 'High', '${LOGISTICS_EMAIL}');

-- Alert checking procedure
CREATE PROCEDURE usp_CheckAndSendAlerts
AS
BEGIN
    DECLARE @AlertMessage VARCHAR(MAX);
    
    -- Check fleet idle time
    IF EXISTS (
        SELECT 1 
        FROM FactFleetTrips 
        WHERE TimeKey IN (SELECT TimeKey FROM DimTime WHERE DateTime >= DATEADD(HOUR, -1, GETDATE()))
        GROUP BY VehicleID
        HAVING AVG(IdleTimeMinutes / NULLIF(DurationMinutes, 0)) * 100 > 
            (SELECT ThresholdValue FROM AlertThresholds WHERE MetricName = 'FleetIdleTimePercentage')
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Fleet idle time exceeded threshold. Check fleet dashboard.';
        -- Integration with email/Teams notification system
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = (SELECT RecipientList FROM AlertThresholds WHERE MetricName = 'FleetIdleTimePercentage'),
            @subject = 'LogiFleet Pulse Alert: High Fleet Idle Time',
            @body = @AlertMessage;
    END;
    
    -- Check warehouse dwell time
    IF EXISTS (
        SELECT 1 
        FROM FactWarehouseOperations
        WHERE DwellTimeHours > (SELECT ThresholdValue FROM AlertThresholds WHERE MetricName = 'WarehouseDwellTimeHours')
            AND TimeKey IN (SELECT TimeKey FROM DimTime WHERE DateTime >= DATEADD(HOUR, -24, GETDATE()))
    )
    BEGIN
        SET @AlertMessage = 'CRITICAL: Products exceeding dwell time threshold detected.';
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = (SELECT RecipientList FROM AlertThresholds WHERE MetricName = 'WarehouseDwellTimeHours'),
            @subject = 'LogiFleet Pulse Alert: Critical Dwell Time',
            @body = @AlertMessage,
            @importance = 'High';
    END;
END;
GO

-- Schedule alert checking (SQL Server Agent Job)
-- Create job to run usp_CheckAndSendAlerts every 15 minutes
```

## Power BI Configuration

### DAX Measures for Cross-Fact Analysis

```dax
// Total Fleet Cost per Warehouse Dwell Hour
FleetCostPerDwellHour = 
DIVIDE(
    SUM(FactFleetTrips[TripCost]),
    AVERAGE(FactWarehouseOperations[DwellTimeHours]),
    0
)

// Warehouse Gravity Zone Efficiency
GravityZoneEfficiency = 
VAR HighGravityAvgDwell = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeHours]),
        DimProductGravity[GravityZone] = "High"
    )
VAR LowGravityAvgDwell = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeHours]),
        DimProductGravity[GravityZone] = "Low"
    )
RETURN
    DIVIDE(LowGravityAvgDwell - HighGravityAvgDwell, LowGravityAvgDwell, 0) * 100

// Fleet Idle Time Percentage
FleetIdleTimePercentage = 
DIVIDE(
    SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[DurationMinutes]),
    0
) * 100

// Predictive Bottleneck Score (0-100)
BottleneckScore = 
VAR CurrentUtilization = 
    DIVIDE(
        COUNTROWS(FactWarehouseOperations),
        CALCULATE(
            COUNTROWS(FactWarehouseOperations),
            DATESINPERIOD(DimTime[Date], MAX(DimTime[Date]), -7, DAY)
        ) / 7,
        0
    )
VAR DwellTrend = 
    DIVIDE(
        AVERAGE(FactWarehouseOperations[DwellTimeHours]),
        CALCULATE(
            AVERAGE(FactWarehouseOperations[DwellTimeHours]),
            DATESINPERIOD(DimTime[Date], MAX(DimTime[Date]), -30, DAY)
        ),
        1
    )
RETURN
    MINX({CurrentUtilization * DwellTrend * 100}, [Value])
```

### Row-Level Security

```dax
// RLS for regional managers - only see their region
[Region] = USERNAME()

// RLS for warehouse supervisors - only see their warehouse
[LocationName] = LOOKUPVALUE(
    Users[AssignedWarehouse],
    Users[Email], USERNAME()
)
```

## Common Patterns

### Pattern 1: Warehouse Gravity Zone Optimization

```sql
-- Analyze current gravity zone assignments vs. actual performance
WITH ZonePerformance AS (
    SELECT 
        p.SKU,
        p.ProductName,
        p.GravityZone AS CurrentZone,
        AVG(w.DwellTimeHours) AS AvgDwellTime,
        COUNT(*) AS PickCount,
        -- Calculate what zone it SHOULD be in based on actual metrics
        CASE 
            WHEN AVG(w.DwellTimeHours) < 24 AND COUNT(*) > 100 THEN 'High'
            WHEN AVG(w.DwellTimeHours) < 72 AND COUNT(*) > 20 THEN 'Medium'
            ELSE 'Low'
        END AS RecommendedZone
    FROM FactWarehouseOperations w
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.DateTime >= DATEADD(DAY, -30, GETDATE())
        AND w.OperationType = 'Picking'
    GROUP BY p.SKU, p.ProductName, p.GravityZone
)
SELECT 
    SKU,
    ProductName,
    CurrentZone,
    RecommendedZone,
    AvgDwellTime,
    PickCount
FROM ZonePerformance
WHERE CurrentZone <> RecommendedZone
ORDER BY PickCount DESC;
```

### Pattern 2: Fleet Route Optimization Analysis

```sql
-- Identify routes with high idle time and suggest consolidation
WITH RouteAnalysis AS (
    SELECT 
        CONCAT(go.LocationName, ' -> ', gd.LocationName) AS Route,
        COUNT(*) AS TripCount,
        AVG(f.IdleTimeMinutes / NULLIF(f.DurationMinutes, 0)) * 100 AS AvgIdlePct,
        AVG(f.LoadWeightKG / 1000.0) AS AvgLoadTonnes,
        SUM(f.TripCost) AS TotalCost,
        -- Vehicle capacity utilization (assuming 10 tonne max)
        AVG(f.LoadWeightKG / 1000.0) / 10.0 * 100 AS CapacityUtilization
    FROM FactFleetTrips f
    INNER JOIN DimGeography go ON f.OriginGeographyKey = go.GeographyKey
    INNER JOIN DimGeography gd ON f.DestinationGeographyKey = gd.GeographyKey
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.DateTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY CONCAT(go.LocationName, ' -> ', gd.LocationName)
)
SELECT 
    Route,
    TripCount,
    AvgIdlePct,
    CapacityUtilization,
    TotalCost,
    CASE 
        WHEN AvgIdlePct > 15 THEN 'Reduce idle time through driver training'
        WHEN CapacityUtilization < 60 THEN 'Consolidate trips to improve load efficiency'
        ELSE 'Optimize'
    END AS Recommendation
FROM RouteAnalysis
WHERE AvgIdlePct > 10 OR CapacityUtilization < 70
ORDER BY TotalCost DESC;
```

### Pattern 3: Supplier Reliability Impact on Operations

```sql
-- Correlate supplier reliability with warehouse and fleet performance
SELECT 
    s.SupplierName,
    s.ReliabilityTier,
    s.LeadTimeAvgDays,
    s.LeadTimeVariance,
    AVG(cd.DockDwellMinutes) AS AvgCrossDockDwell,
    AVG(w.DwellTimeHours) AS AvgWarehouseDwell,
    COUNT(DISTINCT f.TripKey) AS OutboundTrips,
    AVG(f.DelayMinutes) AS AvgDeliveryDelay,
    SUM(cd.TransferCost + w.OperationCost + f.TripCost) AS TotalLogisticsCost
FROM DimSupplierReliability s
INNER JOIN FactCrossDock cd ON s.SupplierKey = cd.SupplierKey
INNER JOIN FactWarehouseOperations w ON cd.ProductKey = w.ProductKey
INNER JOIN FactFleetTrips f ON cd.OutboundTripID = f.TripID
INNER JOIN DimTime t ON cd.TimeKey = t.TimeKey
WHERE t.DateTime >= DATEADD(DAY, -90, GETDATE())
GROUP BY s.SupplierName, s.ReliabilityTier, s.LeadTimeAvgDays, s.LeadTimeVariance
ORDER BY TotalLogisticsCost DESC;
```

## Troubleshooting

### Issue: Power BI report refresh fails

**Symptoms:** Error message "Unable to connect to data source"

**Solutions:**
```sql
-- Check database connection
SELECT @@SERVERNAME AS ServerName, DB_NAME() AS DatabaseName;

-- Verify service account has read access
SELECT 
    dp.name AS UserName,
    dp.type_desc AS UserType,
    o.name AS TableName,
    p.permission_name,
    p.state_desc
FROM sys.database_permissions p
INNER JOIN sys.database_principals dp ON p.grantee_principal_id = dp.principal_id
INNER JOIN sys.objects o ON p.major_id = o.object_id
WHERE dp.name = '${POWERBI_SERVICE_ACCOUNT}'
ORDER BY TableName;
```

### Issue: Slow query performance on cross-fact analysis

**Symptoms:** Queries joining FactWarehouseOperations and FactFleetTrips timeout

**Solutions:**
```sql
-- Create covering indexes for common join patterns
CREATE NONCLUSTERED INDEX IX_FactWarehouse_GeographyTime
ON FactWarehouseOperations(GeographyKey, TimeKey)
INCLUDE (ProductKey, DwellTimeHours, OperationCost);

CREATE NONCLUSTERED INDEX IX_FactFleet_OriginTime
ON FactFleetTrips(OriginGeographyKey, TimeKey)
INCLUDE (DestinationGeographyKey, IdleTimeMinutes, TripCost);

-- Enable query store for performance analysis
ALTER DATABASE LogiFleetPulse SET QUERY_STORE = ON;

-- Review execution plans for expensive queries
SELECT 
    qs.query_id,
    qt.query_sql_text,
    rs.avg_duration / 1000.0 AS avg_duration_ms,
    rs.avg_logical_io_reads,
    rs.count_executions
FROM sys.query_store_query_text qt
INNER JOIN sys.query_store_query q ON qt.query_text_id = q.query_text_id
INNER JOIN sys.query_store_plan qp ON q.query_id = qp.query_id
INNER JOIN sys.query_store_runtime_stats rs ON qp.plan_id = rs.plan_id
WHERE qt.query_sql_text LIKE '%FactWarehouse%FactFleet%'
ORDER BY rs.avg_duration DESC;
```

### Issue: Data freshness lag exceeds 15 minutes

**Symptoms:** Dashboard shows stale data

**Solutions:**
```sql
-- Check last successful load times
SELECT 
    'WarehouseOperations' AS FactTable,
    MAX(t.DateTime) AS LastLoadedDateTime,
    DATEDIFF(MINUTE, MAX(t.DateTime), GETDATE()) AS MinutesSinceLastLoad
FROM FactWarehouseOperations f
INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
UNION ALL
SELECT 
    'FleetTrips',
    MAX(t.DateTime),
    DATEDIFF(MINUTE, MAX(t.DateTime), GETDATE())
FROM FactFleetTrips f
INNER JOIN DimTime t ON f.TimeKey = t.TimeKey;

-- Force immediate incremental load
DECLARE @StartTime DATETIME2 = DATEADD(HOUR, -2, GETDATE());
DECLARE @EndTime DATETIME2 = GETDATE();

EXEC usp_LoadWarehouseOperations @StartTime, @EndTime;
EXEC usp_LoadFleetTrips @StartTime, @EndTime;

-- Check SQL Agent job status
SELECT 
    j.name AS JobName,
    h.run_date,
    h.run_time,
    h.run_duration,
    CASE h.run_status
        WHEN 0 THEN 'Failed'
        WHEN 1 THEN 'Succeeded'
        WHEN 2 THEN 'Retry'
        WHEN 3 THEN 'Canceled'
        WHEN 4 THEN 'In Progress'
    END AS Status,
    h.message
FROM msdb.dbo.sysjobs j
INNER JOIN msdb.dbo.sysjobhistory h ON j.job_id = h.job_id
WHERE j.name LIKE '%LogiFleet%'
ORDER BY h.run_date DESC, h.run_time DESC;
```

### Issue: Gravity zone recommendations not appearing

**Symptoms:** Zone optimization query returns no rows

**Solutions:**
```sql
-- Verify product gravity scores are calculated
SELECT 
    COUNT(*) AS TotalProducts,
    COUNT(CASE WHEN GravityScore IS NOT NULL THEN 1 END) AS WithGravityScore,
    COUNT(CASE WHEN GravityZone IS NOT NULL THEN 1 END) AS WithGravityZone
FROM DimProductGravity;

-- Recalculate gravity scores if needed
UPDATE DimProductGravity
SET 
    VelocityScore = (
        SELECT COUNT(*) 
        FROM FactWarehouseOperations w
        WHERE w.ProductKey = DimProductGravity.ProductKey
            AND w.OperationType = 'Picking'
    ) / 10.0, -- Normalize to 0-100 scale
    GravityZone = CASE 
        WHEN GravityScore >= 70 THEN 'High'
        WHEN GravityScore >= 40 THEN 'Medium'
        ELSE 'Low'
    END;
```

## Environment Variables Reference

```bash
# Database connection
export SQL_SERVER_HOST="your-sql-server.database.windows.net"
export SQL_USER="logifleet_admin"
export SQL_PASSWORD="your-secure-password"

# Data source connections
export WMS_CONNECTION_STRING="your-wms-connection-string"
export TELEMATICS_API_URL="https://api.telematics-provider.com/v1"
export WEATHER_API_KEY="your-weather-api-key"

# Email notifications
export FLEET_MANAGER_EMAIL="fleet.manager@company.com"
export WAREHOUSE_MANAGER_EMAIL="warehouse.manager@company.com"
export OPERATIONS_EMAIL="operations
