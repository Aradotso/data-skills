---
name: logifleet-pulse-supply-chain-analytics
description: Multi-modal logistics intelligence engine with MS SQL Server star schema and Power BI dashboards for warehouse, fleet, and supply chain analytics
triggers:
  - set up logifleet pulse supply chain analytics
  - configure warehouse and fleet analytics database
  - deploy power bi logistics dashboard
  - create multi-fact star schema for supply chain
  - implement warehouse gravity zone optimization
  - build real-time fleet tracking analytics
  - design logistics data warehouse schema
  - integrate supply chain kpi dashboards
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI agents to help developers deploy and configure LogiFleet Pulse, an advanced logistics intelligence platform that combines MS SQL Server data warehousing with Power BI visualization for unified warehouse, fleet, and supply chain analytics.

## What LogiFleet Pulse Does

LogiFleet Pulse is a **multi-modal logistics intelligence engine** that:

- Unifies warehouse operations, fleet telemetry, inventory, and external data into a single semantic layer
- Implements a custom multi-fact star schema with time-phased dimensions
- Provides real-time Power BI dashboards (15-minute refresh cycles)
- Enables cross-fact KPI analysis (e.g., warehouse dwell time vs. fleet idle costs)
- Offers predictive bottleneck detection and adaptive fleet maintenance triage
- Supports warehouse "gravity zone" optimization based on pick frequency and item value

**Core Components:**
- SQL Server database schema (multi-fact star schema)
- Power BI template (`.pbit` file)
- Data ingestion pipelines (external tables, stored procedures)
- Role-based security and alerting system

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Data sources: WMS, TMS/telemetry APIs, ERP system

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Create the database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Deploy dimension tables first
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    DateTimeUTC DATETIME NOT NULL,
    TimeSlot15Min INT NOT NULL, -- 0-95 (96 slots per day)
    HourOfDay INT NOT NULL,
    DayOfWeek NVARCHAR(20) NOT NULL,
    FiscalPeriod NVARCHAR(10) NOT NULL,
    IsBusinessHour BIT DEFAULT 1
)

CREATE CLUSTERED INDEX IX_DimTime_DateTime ON DimTime(DateTimeUTC)
GO

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    NodeID NVARCHAR(50) NOT NULL UNIQUE,
    NodeType NVARCHAR(20) NOT NULL, -- 'Warehouse', 'Route', 'Supplier'
    LocationName NVARCHAR(200) NOT NULL,
    Region NVARCHAR(100),
    Country NVARCHAR(100),
    Continent NVARCHAR(50),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7)
)
GO

CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU NVARCHAR(50) NOT NULL UNIQUE,
    ProductName NVARCHAR(200),
    Category NVARCHAR(100),
    GravityScore DECIMAL(5,2) DEFAULT 0, -- Calculated: velocity * value / fragility
    VelocityRank INT, -- 1=fastest moving
    ValueTier NVARCHAR(20), -- 'High', 'Medium', 'Low'
    FragilityIndex DECIMAL(3,2) DEFAULT 1.0,
    LastUpdated DATETIME DEFAULT GETDATE()
)

CREATE INDEX IX_Product_Gravity ON DimProductGravity(GravityScore DESC)
GO

CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID NVARCHAR(50) NOT NULL UNIQUE,
    SupplierName NVARCHAR(200),
    LeadTimeMean INT, -- Days
    LeadTimeVariance DECIMAL(5,2),
    DefectPercentage DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2) DEFAULT 100.0,
    LastAuditDate DATE
)
GO

-- Fact table: Warehouse Operations
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType NVARCHAR(50) NOT NULL, -- 'Putaway', 'Pick', 'Pack', 'Ship', 'Receiving'
    OperationStartUTC DATETIME NOT NULL,
    OperationEndUTC DATETIME,
    DurationMinutes AS DATEDIFF(MINUTE, OperationStartUTC, OperationEndUTC),
    Quantity INT NOT NULL,
    ZoneAssignment NVARCHAR(50),
    DwellTimeHours DECIMAL(8,2), -- Hours between receiving and shipping
    OperatorID NVARCHAR(50),
    BatchID NVARCHAR(50)
)

CREATE CLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey, OperationStartUTC)
CREATE INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey)
CREATE INDEX IX_FactWarehouse_Geography ON FactWarehouseOperations(GeographyKey)
GO

-- Fact table: Fleet Operations
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID NVARCHAR(50) NOT NULL,
    DriverID NVARCHAR(50),
    TripStartUTC DATETIME NOT NULL,
    TripEndUTC DATETIME,
    TripDurationMinutes AS DATEDIFF(MINUTE, TripStartUTC, TripEndUTC),
    DistanceKM DECIMAL(10,2),
    FuelConsumptionLiters DECIMAL(8,2),
    IdleTimeMinutes INT DEFAULT 0,
    LoadWeightKG DECIMAL(10,2),
    RouteSegmentID NVARCHAR(50),
    DelayReasonCode NVARCHAR(50) -- 'Weather', 'Traffic', 'Mechanical', NULL
)

CREATE CLUSTERED INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey, TripStartUTC)
CREATE INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleID)
CREATE INDEX IX_FactFleet_Origin ON FactFleetTrips(OriginGeographyKey)
GO

-- Cross-Dock bridge table (many-to-many: trips to warehouse operations)
CREATE TABLE BridgeCrossDock (
    BridgeKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TripKey BIGINT NOT NULL FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OperationKey BIGINT NOT NULL FOREIGN KEY REFERENCES FactWarehouseOperations(OperationKey),
    TransferTimeUTC DATETIME NOT NULL,
    CONSTRAINT UQ_CrossDock UNIQUE (TripKey, OperationKey)
)
GO
```

### Step 2: Create Time Dimension Population Procedure

```sql
CREATE PROCEDURE uspPopulateTimeDimension
    @StartDate DATETIME,
    @EndDate DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentDateTime DATETIME = @StartDate;
    
    WHILE @CurrentDateTime <= @EndDate
    BEGIN
        INSERT INTO DimTime (DateTimeUTC, TimeSlot15Min, HourOfDay, DayOfWeek, FiscalPeriod, IsBusinessHour)
        VALUES (
            @CurrentDateTime,
            (DATEPART(HOUR, @CurrentDateTime) * 4) + (DATEPART(MINUTE, @CurrentDateTime) / 15),
            DATEPART(HOUR, @CurrentDateTime),
            DATENAME(WEEKDAY, @CurrentDateTime),
            CONCAT('FY', YEAR(@CurrentDateTime), 'Q', DATEPART(QUARTER, @CurrentDateTime)),
            CASE 
                WHEN DATEPART(HOUR, @CurrentDateTime) BETWEEN 6 AND 18 
                     AND DATEPART(WEEKDAY, @CurrentDateTime) NOT IN (1, 7) 
                THEN 1 
                ELSE 0 
            END
        );
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END
GO

-- Populate time dimension for 2 years
EXEC uspPopulateTimeDimension '2025-01-01', '2027-01-01'
GO
```

### Step 3: Create Data Ingestion Stored Procedure

```sql
CREATE PROCEDURE uspIngestWarehouseOperations
    @SourceTable NVARCHAR(200) -- External table or staging table name
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Merge from staging into fact table with dimension lookups
    MERGE FactWarehouseOperations AS Target
    USING (
        SELECT 
            t.TimeKey,
            g.GeographyKey,
            p.ProductKey,
            s.OperationType,
            s.OperationStartUTC,
            s.OperationEndUTC,
            s.Quantity,
            s.ZoneAssignment,
            s.DwellTimeHours,
            s.OperatorID,
            s.BatchID
        FROM (SELECT * FROM OPENQUERY([LinkedWMS], 'SELECT * FROM ' + @SourceTable)) AS s
        INNER JOIN DimTime t ON t.DateTimeUTC = DATEADD(MINUTE, (DATEPART(MINUTE, s.OperationStartUTC) / 15) * 15, 
                                                         DATEADD(HOUR, DATEDIFF(HOUR, 0, s.OperationStartUTC), 0))
        INNER JOIN DimGeography g ON g.NodeID = s.WarehouseID
        INNER JOIN DimProductGravity p ON p.SKU = s.SKU
    ) AS Source
    ON Target.OperationStartUTC = Source.OperationStartUTC 
       AND Target.ProductKey = Source.ProductKey 
       AND Target.OperatorID = Source.OperatorID
    WHEN NOT MATCHED THEN
        INSERT (TimeKey, GeographyKey, ProductKey, OperationType, OperationStartUTC, OperationEndUTC, 
                Quantity, ZoneAssignment, DwellTimeHours, OperatorID, BatchID)
        VALUES (Source.TimeKey, Source.GeographyKey, Source.ProductKey, Source.OperationType, 
                Source.OperationStartUTC, Source.OperationEndUTC, Source.Quantity, Source.ZoneAssignment, 
                Source.DwellTimeHours, Source.OperatorID, Source.BatchID);
END
GO
```

### Step 4: Configure Power BI Template

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. When prompted, enter your SQL Server connection string:
   - **Server:** `your-sql-server.database.windows.net` or `localhost`
   - **Database:** `LogiFleetPulse`
   - **Authentication:** Windows or SQL Server credentials (use env vars)

```powerquery
// Power Query M code for SQL connection
let
    Server = Environment.GetEnvironmentVariable("LOGIFLEET_SQL_SERVER"),
    Database = Environment.GetEnvironmentVariable("LOGIFLEET_DATABASE"),
    Source = Sql.Database(Server, Database, [Query="
        SELECT 
            o.OperationKey,
            t.DateTimeUTC,
            t.HourOfDay,
            g.LocationName AS Warehouse,
            g.Region,
            p.SKU,
            p.ProductName,
            p.GravityScore,
            o.OperationType,
            o.DurationMinutes,
            o.DwellTimeHours,
            o.Quantity
        FROM FactWarehouseOperations o
        INNER JOIN DimTime t ON o.TimeKey = t.TimeKey
        INNER JOIN DimGeography g ON o.GeographyKey = g.GeographyKey
        INNER JOIN DimProductGravity p ON o.ProductKey = p.ProductKey
        WHERE t.DateTimeUTC >= DATEADD(DAY, -90, GETDATE())
    "])
in
    Source
```

3. Configure scheduled refresh (Power BI Service):
   - Refresh frequency: Every 15 minutes (requires Premium capacity)
   - Incremental refresh: Enable for fact tables (retain 2 years, refresh last 7 days)

## Key SQL Queries for Analytics

### Cross-Fact KPI: Dwell Time vs Fleet Idle Time

```sql
-- Find correlation between warehouse dwell time and fleet delays
SELECT 
    w.LocationName AS Warehouse,
    AVG(wo.DwellTimeHours) AS AvgDwellTimeHours,
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdleMinutes,
    COUNT(DISTINCT ft.VehicleID) AS VehicleCount,
    SUM(ft.FuelConsumptionLiters) AS TotalFuelLiters
FROM FactWarehouseOperations wo
INNER JOIN DimGeography w ON wo.GeographyKey = w.GeographyKey
INNER JOIN BridgeCrossDock b ON wo.OperationKey = b.OperationKey
INNER JOIN FactFleetTrips ft ON b.TripKey = ft.TripKey
WHERE wo.OperationStartUTC >= DATEADD(DAY, -30, GETDATE())
GROUP BY w.LocationName
HAVING AVG(wo.DwellTimeHours) > 48 -- Products sitting more than 2 days
ORDER BY AvgFleetIdleMinutes DESC
```

### Warehouse Gravity Zone Recommendations

```sql
-- Identify products that should be moved to higher-gravity zones
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    wo.ZoneAssignment AS CurrentZone,
    CASE 
        WHEN p.GravityScore > 75 THEN 'Zone A (High Gravity)'
        WHEN p.GravityScore > 50 THEN 'Zone B (Medium Gravity)'
        ELSE 'Zone C (Low Gravity)'
    END AS RecommendedZone,
    COUNT(*) AS PickCount,
    AVG(wo.DurationMinutes) AS AvgPickTimeMinutes
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
WHERE wo.OperationType = 'Pick'
  AND wo.OperationStartUTC >= DATEADD(DAY, -14, GETDATE())
GROUP BY p.SKU, p.ProductName, p.GravityScore, wo.ZoneAssignment
HAVING COUNT(*) > 10 -- Only products with significant activity
ORDER BY p.GravityScore DESC
```

### Predictive Bottleneck Detection

```sql
-- Identify time slots and locations with high congestion risk
WITH OperationCounts AS (
    SELECT 
        t.TimeSlot15Min,
        t.HourOfDay,
        g.LocationName,
        COUNT(*) AS OperationVolume,
        AVG(wo.DurationMinutes) AS AvgDuration
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
    WHERE t.DateTimeUTC >= DATEADD(DAY, -7, GETDATE())
    GROUP BY t.TimeSlot15Min, t.HourOfDay, g.LocationName
)
SELECT 
    TimeSlot15Min,
    HourOfDay,
    LocationName,
    OperationVolume,
    AvgDuration,
    CASE 
        WHEN OperationVolume > 100 AND AvgDuration > 20 THEN 'Critical'
        WHEN OperationVolume > 75 AND AvgDuration > 15 THEN 'High'
        WHEN OperationVolume > 50 THEN 'Medium'
        ELSE 'Normal'
    END AS BottleneckRisk
FROM OperationCounts
WHERE OperationVolume > 25
ORDER BY OperationVolume DESC, AvgDuration DESC
```

### Fleet Maintenance Priority Queue

```sql
-- Generate maintenance queue based on revenue impact
WITH VehicleMetrics AS (
    SELECT 
        ft.VehicleID,
        SUM(ft.IdleTimeMinutes) AS TotalIdleMinutes,
        AVG(ft.FuelConsumptionLiters / NULLIF(ft.DistanceKM, 0)) AS AvgFuelEfficiency,
        COUNT(*) AS TripCount,
        SUM(ft.LoadWeightKG) AS TotalCargoHandled,
        MAX(ft.TripEndUTC) AS LastTripDate
    FROM FactFleetTrips ft
    WHERE ft.TripStartUTC >= DATEADD(DAY, -30, GETDATE())
    GROUP BY ft.VehicleID
)
SELECT 
    VehicleID,
    TotalIdleMinutes,
    AvgFuelEfficiency,
    TripCount,
    TotalCargoHandled,
    LastTripDate,
    CASE 
        WHEN TotalIdleMinutes > 500 AND AvgFuelEfficiency > 0.15 THEN 'Priority 1 - Engine Diagnostics'
        WHEN TotalIdleMinutes > 300 THEN 'Priority 2 - Route Optimization Review'
        WHEN AvgFuelEfficiency > 0.12 THEN 'Priority 3 - Tire Pressure Check'
        ELSE 'Standard Maintenance Schedule'
    END AS MaintenanceAction,
    (TotalCargoHandled / 1000.0) * 50 AS EstimatedRevenueAtRisk -- $50 per ton at risk
FROM VehicleMetrics
ORDER BY TotalIdleMinutes DESC, AvgFuelEfficiency DESC
```

## Configuration Patterns

### Role-Based Security

```sql
-- Create security table
CREATE TABLE SecurityRoles (
    RoleID INT PRIMARY KEY IDENTITY(1,1),
    RoleName NVARCHAR(50) NOT NULL UNIQUE,
    GeographyFilter NVARCHAR(MAX), -- JSON array of allowed GeographyKeys
    CanViewFinancials BIT DEFAULT 0
)

INSERT INTO SecurityRoles (RoleName, GeographyFilter, CanViewFinancials)
VALUES 
    ('GlobalExecutive', NULL, 1), -- Full access
    ('RegionalManager', '[10, 11, 12]', 1), -- Specific regions
    ('WarehouseSupervisor', '[10]', 0) -- Single warehouse
GO

-- Row-level security function
CREATE FUNCTION fnSecurityPredicate(@GeographyKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN (
    SELECT 1 AS AccessGranted
    WHERE 
        IS_MEMBER('GlobalExecutive') = 1 
        OR EXISTS (
            SELECT 1 
            FROM dbo.SecurityRoles sr
            INNER JOIN dbo.UserRoleMapping urm ON sr.RoleID = urm.RoleID
            WHERE urm.UserName = USER_NAME()
              AND (sr.GeographyFilter IS NULL OR @GeographyKey IN (SELECT value FROM STRING_SPLIT(sr.GeographyFilter, ',')))
        )
)
GO

-- Apply security policy
CREATE SECURITY POLICY GeographySecurityPolicy
ADD FILTER PREDICATE dbo.fnSecurityPredicate(GeographyKey) ON dbo.FactWarehouseOperations,
ADD FILTER PREDICATE dbo.fnSecurityPredicate(OriginGeographyKey) ON dbo.FactFleetTrips
WITH (STATE = ON)
GO
```

### Automated Alerting

```sql
CREATE PROCEDURE uspCheckKPIThresholds
AS
BEGIN
    -- Check for excessive dwell time
    INSERT INTO AlertLog (AlertType, Severity, Message, DetectedUTC)
    SELECT 
        'ExcessiveDwellTime' AS AlertType,
        'High' AS Severity,
        CONCAT('SKU ', p.SKU, ' has avg dwell time of ', CAST(AVG(wo.DwellTimeHours) AS NVARCHAR(10)), ' hours') AS Message,
        GETDATE() AS DetectedUTC
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    WHERE wo.OperationStartUTC >= DATEADD(HOUR, -24, GETDATE())
    GROUP BY p.SKU
    HAVING AVG(wo.DwellTimeHours) > 72 -- 3 days threshold
    
    -- Check for fleet idle time anomalies
    INSERT INTO AlertLog (AlertType, Severity, Message, DetectedUTC)
    SELECT 
        'FleetIdleAnomaly' AS AlertType,
        'Medium' AS Severity,
        CONCAT('Vehicle ', ft.VehicleID, ' idle time ', CAST(SUM(ft.IdleTimeMinutes) AS NVARCHAR(10)), ' min in last 24h') AS Message,
        GETDATE() AS DetectedUTC
    FROM FactFleetTrips ft
    WHERE ft.TripStartUTC >= DATEADD(HOUR, -24, GETDATE())
    GROUP BY ft.VehicleID
    HAVING SUM(ft.IdleTimeMinutes) > (SELECT AVG(TotalIdle) * 1.5 
                                       FROM (SELECT SUM(IdleTimeMinutes) AS TotalIdle 
                                             FROM FactFleetTrips 
                                             WHERE TripStartUTC >= DATEADD(DAY, -7, GETDATE())
                                             GROUP BY VehicleID) AS AvgData)
END
GO

-- Schedule with SQL Agent
-- CREATE JOB 'LogiFleetAlerts' TO RUN uspCheckKPIThresholds every 15 minutes
```

### Environment Variables Configuration

Create a `.env` file (not committed to repo):

```bash
# SQL Server Connection
LOGIFLEET_SQL_SERVER=your-server.database.windows.net
LOGIFLEET_DATABASE=LogiFleetPulse
LOGIFLEET_SQL_USER=logifleet_app
LOGIFLEET_SQL_PASSWORD=your-secure-password

# External API Keys (for weather/traffic enrichment)
WEATHER_API_KEY=your-weather-api-key
TRAFFIC_API_KEY=your-traffic-api-key

# Power BI Service
POWERBI_WORKSPACE_ID=your-workspace-guid
POWERBI_DATASET_ID=your-dataset-guid
```

Load in SQL scripts via linked server or application layer:

```sql
-- Example: External table for weather data
CREATE EXTERNAL DATA SOURCE WeatherAPI
WITH (
    TYPE = REST,
    LOCATION = 'https://api.weatherprovider.com',
    CREDENTIAL = WeatherAPICredential -- Stored in SQL Server securely
)
```

## Common Patterns

### Pattern 1: Temporal Elasticity Simulation

```sql
-- Simulate impact of warehouse capacity changes on fleet utilization
DECLARE @CapacityMultiplier DECIMAL(3,2) = 0.95; -- 95% capacity

WITH SimulatedOperations AS (
    SELECT 
        wo.*,
        wo.DurationMinutes * @CapacityMultiplier AS SimulatedDuration
    FROM FactWarehouseOperations wo
    WHERE wo.OperationStartUTC >= DATEADD(DAY, -30, GETDATE())
),
ImpactedTrips AS (
    SELECT 
        ft.TripKey,
        ft.VehicleID,
        ft.TripDurationMinutes,
        ft.TripDurationMinutes + (sim.SimulatedDuration - wo.DurationMinutes) AS ProjectedDuration
    FROM FactFleetTrips ft
    INNER JOIN BridgeCrossDock b ON ft.TripKey = b.TripKey
    INNER JOIN SimulatedOperations sim ON b.OperationKey = sim.OperationKey
    INNER JOIN FactWarehouseOperations wo ON b.OperationKey = wo.OperationKey
)
SELECT 
    AVG(TripDurationMinutes) AS CurrentAvgTripMinutes,
    AVG(ProjectedDuration) AS ProjectedAvgTripMinutes,
    AVG(ProjectedDuration - TripDurationMinutes) AS AvgImpactMinutes,
    COUNT(DISTINCT VehicleID) AS AffectedVehicles
FROM ImpactedTrips
```

### Pattern 2: Natural Language Query (Power BI Q&A)

Configure synonyms in Power BI model:

```
// In Power BI Desktop > Modeling > Q&A Setup
Synonyms for "DwellTimeHours": storage time, time in warehouse, holding period
Synonyms for "IdleTimeMinutes": waiting time, parked time, non-moving time
Synonyms for "GravityScore": importance, priority, pull factor
```

Users can then ask:
- "Show me products with high storage time last month"
- "Which vehicles had the most waiting time this week?"
- "Compare priority scores across regions"

### Pattern 3: Incremental Data Refresh

```sql
-- Create view for incremental refresh (Power BI)
CREATE VIEW vw_FactWarehouseOperations_Incremental
AS
SELECT 
    o.*,
    t.DateTimeUTC AS RefreshDate
FROM FactWarehouseOperations o
INNER JOIN DimTime t ON o.TimeKey = t.TimeKey
WHERE t.DateTimeUTC >= DATEADD(DAY, -7, GETDATE()) -- Only last 7 days
GO
```

In Power BI:
1. Right-click table > Incremental Refresh
2. Archive data: 2 years
3. Refresh data: Last 7 days
4. Detect data changes: Use `RefreshDate` column

## Troubleshooting

### Issue: Slow Cross-Fact Queries

**Solution:** Ensure proper indexing on bridge tables

```sql
-- Add covering indexes
CREATE INDEX IX_Bridge_TripOperation 
ON BridgeCrossDock (TripKey, OperationKey) 
INCLUDE (TransferTimeUTC)

-- Update statistics
UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN
UPDATE STATISTICS FactFleetTrips WITH FULLSCAN
```

### Issue: Power BI Refresh Failures

**Solution:** Check connection timeout and query folding

```powerquery
// Increase timeout in Power Query
let
    Source = Sql.Database(Server, Database, [
        CommandTimeout=#duration(0, 1, 0, 0) // 1 hour timeout
    ])
in
    Source
```

Verify query folding (View > Query Folding Indicators) — if broken, optimize M code.

### Issue: Time Dimension Gaps

**Solution:** Re-run population procedure for missing ranges

```sql
-- Find gaps
SELECT 
    DATEADD(MINUTE, 15, t1.DateTimeUTC) AS MissingStart,
    t2.DateTimeUTC AS NextExisting
FROM DimTime t1
LEFT JOIN DimTime t2 ON t2.DateTimeUTC = DATEADD(MINUTE, 15, t1.DateTimeUTC)
WHERE t2.TimeKey IS NULL
  AND t1.DateTimeUTC >= '2026-01-01'
ORDER BY t1.DateTimeUTC

-- Fill gaps
EXEC uspPopulateTimeDimension '2026-07-01', '2026-08-01'
```

### Issue: Gravity Score Not Updating

**Solution:** Create scheduled job to recalculate

```sql
CREATE PROCEDURE uspRecalculateGravityScores
AS
BEGIN
    UPDATE p
    SET 
        p.VelocityRank = ranked.RowNum,
        p.GravityScore = (ranked.PickFrequency * ranked.AvgValue) / NULLIF(p.FragilityIndex, 0),
        p.LastUpdated = GETDATE()
    FROM DimProductGravity p
    INNER JOIN (
        SELECT 
            wo.ProductKey,
            COUNT(*) AS PickFrequency,
            AVG(p2.ValueTier) AS AvgValue, -- Assume ValueTier mapped to numeric
            ROW_NUMBER() OVER (ORDER BY COUNT(*) DESC) AS RowNum
        FROM FactWarehouseOperations wo
        INNER JOIN DimProductGravity p2 ON wo.ProductKey = p2.ProductKey
        WHERE wo.OperationType = 'Pick'
          AND wo.OperationStartUTC >= DATEADD(DAY, -30, GETDATE())
        GROUP BY wo.ProductKey
    ) ranked ON p.ProductKey = ranked.ProductKey
END
GO

-- Schedule with SQL Agent (daily at 2 AM)
```

### Issue: Row-Level Security Not Working in Power BI Service

**Solution:** Ensure roles are defined in both SQL and Power BI

```sql
-- Verify user mapping
SELECT 
    urm.UserName,
    sr.RoleName,
    sr.GeographyFilter
FROM UserRoleMapping urm
INNER JOIN SecurityRoles sr ON urm.RoleID = sr.RoleID
WHERE urm.UserName = 'domain\username'
```

In Power BI Service:
1. Dataset Settings > Row-level security
2. Add members to roles (match SQL usernames)
3. Test with "View as Role"

## Advanced: External API Integration

### Weather Delay Correlation

```sql
-- Create external table for weather data (requires PolyBase or REST API)
CREATE EXTERNAL TABLE ExtWeatherData (
    LocationID NVARCHAR(50),
    EventTimeUTC DATETIME,
    WeatherCondition NVARCHAR(50),
    SeverityScore INT
)
WITH (
    LOCATION = '/weather/delays',
    DATA_SOURCE = WeatherAPI,
    FILE_FORMAT = JSONFormat
)

-- Correlate with fleet delays
SELECT 
    ft.VehicleID,
    ft.TripStartUTC,
    ft.DelayReasonCode,
    w.WeatherCondition,
    w.SeverityScore,
    ft.IdleTimeMinutes
FROM FactFleetTrips ft
INNER JOIN DimGeography g ON ft.OriginGeographyKey = g.GeographyKey
INNER JOIN ExtWeatherData w ON w.LocationID = g.NodeID
    AND w.EventTimeUTC BETWEEN ft.TripStartUTC AND ft.TripEndUTC
WHERE ft.DelayReasonCode = 'Weather'
  AND w.SeverityScore > 5
