---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehouse with multi-fact star schema for fleet, warehouse, and supply chain analytics
triggers:
  - set up logifleet pulse supply chain analytics
  - deploy logistics data warehouse with power bi
  - configure warehouse and fleet tracking schema
  - implement multi-fact star schema for logistics
  - create supply chain kpi dashboard
  - integrate warehouse and fleet telemetry data
  - build real-time logistics intelligence platform
  - connect power bi to logistics data warehouse
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

LogiFleet Pulse is an advanced MS SQL Server data warehouse and Power BI visualization platform for logistics and supply chain analytics. It uses a multi-fact star schema to integrate warehouse operations, fleet telemetry, inventory management, and external data sources into a unified semantic layer for real-time decision-making.

## What It Does

- **Multi-Modal Data Integration**: Combines warehouse management, fleet tracking, supplier data, and external APIs (weather, traffic)
- **Multi-Fact Star Schema**: Time-phased dimensional modeling with cross-fact KPI reconciliation
- **Real-Time Dashboards**: Power BI reports refreshed every 15 minutes with role-based access
- **Predictive Analytics**: Bottleneck detection, fleet triage, and temporal elasticity modeling
- **Warehouse Optimization**: Gravity zone mapping based on pick frequency, value, and fragility
- **Fleet Intelligence**: Route optimization, fuel consumption analysis, and proactive maintenance scheduling

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Enterprise or Standard edition recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to source systems (WMS, TMS, telemetry feeds)

### Deploy the SQL Schema

1. **Clone the repository**:
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Create the database**:
```sql
-- Execute in SSMS or Azure Data Studio
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Set recovery model for performance
ALTER DATABASE LogiFleetPulse SET RECOVERY SIMPLE
GO
```

3. **Deploy schema objects** (execute scripts in order):
```sql
-- 01_create_dimensions.sql
-- Creates DimTime, DimGeography, DimProduct, DimSupplier, etc.

-- 02_create_facts.sql
-- Creates FactWarehouseOperations, FactFleetTrips, FactCrossDock

-- 03_create_bridges.sql
-- Creates bridge tables for many-to-many relationships

-- 04_create_views.sql
-- Creates semantic layer views for Power BI

-- 05_create_stored_procedures.sql
-- Creates ETL and alerting procedures
```

### Configure Data Sources

Create a configuration file for connection strings:

```json
{
  "connections": {
    "wms_api": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "api_key": "${WMS_API_KEY}"
    },
    "fleet_telemetry": {
      "endpoint": "${FLEET_TELEMETRY_ENDPOINT}",
      "auth_token": "${FLEET_AUTH_TOKEN}"
    },
    "erp_system": {
      "server": "${ERP_SQL_SERVER}",
      "database": "${ERP_DATABASE}",
      "username": "${ERP_USERNAME}",
      "password": "${ERP_PASSWORD}"
    },
    "weather_api": {
      "endpoint": "https://api.weatherapi.com/v1",
      "api_key": "${WEATHER_API_KEY}"
    }
  },
  "refresh_intervals": {
    "warehouse_operations": 900,
    "fleet_trips": 900,
    "supplier_data": 3600,
    "external_feeds": 1800
  }
}
```

## Core Schema Structure

### Dimension Tables

**DimTime** - Time dimension with 15-minute granularity:
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME NOT NULL,
    HourOfDay TINYINT NOT NULL,
    DayOfWeek TINYINT NOT NULL,
    DayName VARCHAR(10),
    FiscalPeriod VARCHAR(10),
    IsWeekend BIT,
    IsHoliday BIT
)

-- Create clustered index
CREATE CLUSTERED INDEX IX_DimTime_FullDateTime ON DimTime(FullDateTime)
```

**DimGeography** - Hierarchical location dimension:
```sql
CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationID VARCHAR(50) UNIQUE NOT NULL,
    LocationName VARCHAR(200) NOT NULL,
    LocationType VARCHAR(50), -- Warehouse, Route Node, Distribution Center
    AddressLine1 VARCHAR(200),
    City VARCHAR(100),
    StateProvince VARCHAR(100),
    Country VARCHAR(100),
    PostalCode VARCHAR(20),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    ParentLocationID VARCHAR(50),
    RegionName VARCHAR(100),
    Continent VARCHAR(50),
    ValidFrom DATETIME2 NOT NULL,
    ValidTo DATETIME2 NOT NULL,
    IsCurrent BIT DEFAULT 1
)
```

**DimProductGravity** - Product dimension with gravity scoring:
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU VARCHAR(100) UNIQUE NOT NULL,
    ProductName VARCHAR(500),
    Category VARCHAR(100),
    Subcategory VARCHAR(100),
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    IsFragile BIT,
    RequiresRefrigeration BIT,
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value / lead_time
    GravityZone VARCHAR(20), -- High, Medium, Low
    PickFrequencyRank INT,
    UnitValue DECIMAL(10,2),
    ValidFrom DATETIME2 NOT NULL,
    ValidTo DATETIME2 NOT NULL,
    IsCurrent BIT DEFAULT 1
)

-- Index for gravity zone queries
CREATE NONCLUSTERED INDEX IX_ProductGravity_Zone ON DimProductGravity(GravityZone, GravityScore DESC)
```

### Fact Tables

**FactWarehouseOperations**:
```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    DateKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(50), -- Receiving, Putaway, Picking, Packing, Shipping
    OperationID VARCHAR(100) UNIQUE NOT NULL,
    OrderID VARCHAR(100),
    QuantityHandled INT,
    DwellTimeMinutes INT,
    CycleTimeSeconds INT,
    DistanceTraveled DECIMAL(10,2), -- in warehouse
    OperatorID VARCHAR(50),
    EquipmentID VARCHAR(50),
    ZoneFrom VARCHAR(50),
    ZoneTo VARCHAR(50),
    ErrorCount INT DEFAULT 0,
    RequiredRework BIT DEFAULT 0,
    CONSTRAINT FK_WH_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WH_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_WH_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
)

-- Columnstore index for analytical queries
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactWarehouseOperations_CS 
ON FactWarehouseOperations (TimeKey, GeographyKey, ProductKey, OperationType, QuantityHandled, DwellTimeMinutes)
```

**FactFleetTrips**:
```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKeyDeparture INT NOT NULL,
    TimeKeyArrival INT NOT NULL,
    GeographyKeyOrigin INT NOT NULL,
    GeographyKeyDestination INT NOT NULL,
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50),
    TripID VARCHAR(100) UNIQUE NOT NULL,
    LoadWeight DECIMAL(10,2),
    LoadVolume DECIMAL(10,2),
    PlannedDistanceKM DECIMAL(10,2),
    ActualDistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    DelayTimeMinutes INT,
    DelayReason VARCHAR(200),
    WeatherCondition VARCHAR(50),
    TrafficLevel VARCHAR(20),
    MaintenanceAlert BIT DEFAULT 0,
    CONSTRAINT FK_Fleet_TimeDep FOREIGN KEY (TimeKeyDeparture) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_TimeArr FOREIGN KEY (TimeKeyArrival) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Origin FOREIGN KEY (GeographyKeyOrigin) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_Fleet_Dest FOREIGN KEY (GeographyKeyDestination) REFERENCES DimGeography(GeographyKey)
)

-- Columnstore for fleet analytics
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactFleetTrips_CS 
ON FactFleetTrips (TimeKeyDeparture, VehicleID, FuelConsumedLiters, IdleTimeMinutes, ActualDistanceKM)
```

## Key Stored Procedures

### ETL Procedure for Warehouse Operations

```sql
CREATE PROCEDURE dbo.usp_LoadWarehouseOperations
    @BatchStartTime DATETIME2,
    @BatchEndTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert warehouse operations from staging
    INSERT INTO FactWarehouseOperations (
        TimeKey, DateKey, GeographyKey, ProductKey,
        OperationType, OperationID, OrderID,
        QuantityHandled, DwellTimeMinutes, CycleTimeSeconds
    )
    SELECT 
        t.TimeKey,
        CAST(CONVERT(VARCHAR(8), s.OperationTimestamp, 112) AS INT) AS DateKey,
        g.GeographyKey,
        p.ProductKey,
        s.OperationType,
        s.OperationID,
        s.OrderID,
        s.QuantityHandled,
        DATEDIFF(MINUTE, s.StartTime, s.EndTime) AS DwellTimeMinutes,
        DATEDIFF(SECOND, s.StartTime, s.EndTime) AS CycleTimeSeconds
    FROM staging.WarehouseOperations s
    INNER JOIN DimTime t ON 
        CAST(s.OperationTimestamp AS DATETIME2) >= t.FullDateTime AND
        CAST(s.OperationTimestamp AS DATETIME2) < DATEADD(MINUTE, 15, t.FullDateTime)
    INNER JOIN DimGeography g ON s.LocationID = g.LocationID AND g.IsCurrent = 1
    INNER JOIN DimProductGravity p ON s.SKU = p.SKU AND p.IsCurrent = 1
    WHERE s.OperationTimestamp >= @BatchStartTime
      AND s.OperationTimestamp < @BatchEndTime
      AND NOT EXISTS (
          SELECT 1 FROM FactWarehouseOperations f 
          WHERE f.OperationID = s.OperationID
      )
    
    -- Log batch execution
    INSERT INTO dbo.ETLLog (ProcedureName, BatchStart, BatchEnd, RowsProcessed)
    VALUES ('usp_LoadWarehouseOperations', @BatchStartTime, @BatchEndTime, @@ROWCOUNT)
END
GO
```

### Calculate Product Gravity Scores

```sql
CREATE PROCEDURE dbo.usp_UpdateProductGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Calculate gravity score based on recent activity
    WITH ProductMetrics AS (
        SELECT 
            p.ProductKey,
            p.SKU,
            COUNT(*) AS PickCount,
            AVG(wo.CycleTimeSeconds) AS AvgCycleTime,
            SUM(wo.QuantityHandled) AS TotalQuantity,
            p.UnitValue
        FROM DimProductGravity p
        INNER JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
        WHERE wo.OperationType = 'Picking'
          AND wo.TimeKey >= (SELECT MAX(TimeKey) - 2880 FROM DimTime) -- Last 30 days
        GROUP BY p.ProductKey, p.SKU, p.UnitValue
    )
    UPDATE p
    SET 
        p.GravityScore = (pm.PickCount * pm.TotalQuantity * p.UnitValue) / NULLIF(pm.AvgCycleTime, 0),
        p.GravityZone = CASE 
            WHEN (pm.PickCount * pm.TotalQuantity * p.UnitValue) / NULLIF(pm.AvgCycleTime, 0) > 1000 THEN 'High'
            WHEN (pm.PickCount * pm.TotalQuantity * p.UnitValue) / NULLIF(pm.AvgCycleTime, 0) > 100 THEN 'Medium'
            ELSE 'Low'
        END,
        p.PickFrequencyRank = RANK() OVER (ORDER BY pm.PickCount DESC)
    FROM DimProductGravity p
    INNER JOIN ProductMetrics pm ON p.ProductKey = pm.ProductKey
    WHERE p.IsCurrent = 1
END
GO
```

### Automated Alert System

```sql
CREATE PROCEDURE dbo.usp_CheckFleetIdlingThresholds
    @ThresholdPercentage DECIMAL(5,2) = 15.0
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Identify trips with excessive idling
    DECLARE @Alerts TABLE (
        VehicleID VARCHAR(50),
        TripID VARCHAR(100),
        IdlePercentage DECIMAL(5,2),
        AlertMessage VARCHAR(500)
    )
    
    INSERT INTO @Alerts
    SELECT 
        ft.VehicleID,
        ft.TripID,
        (CAST(ft.IdleTimeMinutes AS DECIMAL) / NULLIF(DATEDIFF(MINUTE, td.FullDateTime, ta.FullDateTime), 0)) * 100 AS IdlePercentage,
        'Vehicle ' + ft.VehicleID + ' exceeded idle threshold on trip ' + ft.TripID + 
        ' (' + CAST(ROUND((CAST(ft.IdleTimeMinutes AS DECIMAL) / NULLIF(DATEDIFF(MINUTE, td.FullDateTime, ta.FullDateTime), 0)) * 100, 1) AS VARCHAR) + '%)'
    FROM FactFleetTrips ft
    INNER JOIN DimTime td ON ft.TimeKeyDeparture = td.TimeKey
    INNER JOIN DimTime ta ON ft.TimeKeyArrival = ta.TimeKey
    WHERE ft.TimeKeyArrival >= (SELECT MAX(TimeKey) - 96 FROM DimTime) -- Last 24 hours
      AND (CAST(ft.IdleTimeMinutes AS DECIMAL) / NULLIF(DATEDIFF(MINUTE, td.FullDateTime, ta.FullDateTime), 0)) * 100 > @ThresholdPercentage
    
    -- Insert alerts into notification queue
    IF EXISTS (SELECT 1 FROM @Alerts)
    BEGIN
        INSERT INTO dbo.AlertQueue (AlertType, VehicleID, TripID, AlertMessage, CreatedAt)
        SELECT 'FleetIdling', VehicleID, TripID, AlertMessage, GETDATE()
        FROM @Alerts
    END
END
GO
```

## Power BI Configuration

### Connect to SQL Server

1. **Open Power BI Desktop** and select "Get Data" → "SQL Server"

2. **Enter connection details**:
   - Server: `${SQL_SERVER_INSTANCE}`
   - Database: `LogiFleetPulse`
   - Data Connectivity mode: `DirectQuery` (for real-time) or `Import` (for performance)

3. **Import semantic layer views**:
```sql
-- Views to import into Power BI
SELECT * FROM vw_WarehousePerformance
SELECT * FROM vw_FleetEfficiency
SELECT * FROM vw_CrossFactKPIs
SELECT * FROM vw_ProductGravityZones
```

### Key DAX Measures

**Warehouse Efficiency**:
```dax
AvgDwellTime = 
AVERAGE(FactWarehouseOperations[DwellTimeMinutes])

DwellTimeByGravityZone = 
CALCULATE(
    [AvgDwellTime],
    ALLEXCEPT(DimProductGravity, DimProductGravity[GravityZone])
)

PickingEfficiency = 
DIVIDE(
    SUM(FactWarehouseOperations[QuantityHandled]),
    SUM(FactWarehouseOperations[CycleTimeSeconds]) / 3600,
    0
) // Units per hour
```

**Fleet Performance**:
```dax
AvgFuelEfficiency = 
DIVIDE(
    SUM(FactFleetTrips[ActualDistanceKM]),
    SUM(FactFleetTrips[FuelConsumedLiters]),
    0
) // KM per liter

IdleTimePercentage = 
DIVIDE(
    SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[IdleTimeMinutes]) + 
    SUMX(
        FactFleetTrips,
        DATEDIFF(
            RELATED(DimTime[FullDateTime]),
            RELATED(DimTime[FullDateTime]),
            MINUTE
        )
    ),
    0
) * 100

DelayImpactCost = 
SUMX(
    FactFleetTrips,
    FactFleetTrips[DelayTimeMinutes] * 2.5 // $2.50 per minute delay cost
)
```

**Cross-Fact KPIs**:
```dax
DwellToIdleCorrelation = 
VAR HighDwellSKUs = 
    CALCULATETABLE(
        VALUES(DimProductGravity[SKU]),
        FactWarehouseOperations[DwellTimeMinutes] > 72 * 60
    )
RETURN
    CALCULATE(
        AVERAGE(FactFleetTrips[IdleTimeMinutes]),
        FILTER(
            FactFleetTrips,
            // Join logic - trips containing high dwell SKUs
            TRUE
        )
    )

RevenuePerFleetHour = 
DIVIDE(
    [TotalShipmentValue],
    SUMX(
        FactFleetTrips,
        DATEDIFF(
            RELATED(DimTime[FullDateTime]),
            RELATEDTABLE(DimTime)[FullDateTime],
            HOUR
        )
    ),
    0
)
```

### Row-Level Security

```dax
-- Create role: RegionalManager
[GeographyKey] IN (
    SELECT g.GeographyKey
    FROM DimGeography g
    WHERE g.RegionName = USERPRINCIPALNAME()
)

-- Create role: WarehouseOperator
[GeographyKey] IN (
    SELECT g.GeographyKey
    FROM DimGeography g
    INNER JOIN SecurityUserMapping s ON g.LocationID = s.LocationID
    WHERE s.UserEmail = USERPRINCIPALNAME()
)
```

## Common Patterns & Use Cases

### Pattern 1: Identify Gravity Zone Optimization Opportunities

```sql
-- Find products in wrong gravity zone
WITH ProductPerformance AS (
    SELECT 
        p.SKU,
        p.ProductName,
        p.GravityZone AS CurrentZone,
        COUNT(*) AS PickCount,
        AVG(wo.CycleTimeSeconds) AS AvgCycleTime,
        AVG(wo.DistanceTraveled) AS AvgDistance
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    WHERE wo.OperationType = 'Picking'
      AND wo.TimeKey >= (SELECT MAX(TimeKey) - 672 FROM DimTime) -- Last 7 days
    GROUP BY p.SKU, p.ProductName, p.GravityZone
)
SELECT 
    SKU,
    ProductName,
    CurrentZone,
    PickCount,
    AvgCycleTime,
    AvgDistance,
    CASE 
        WHEN CurrentZone = 'Low' AND PickCount > 50 THEN 'Move to High Gravity'
        WHEN CurrentZone = 'High' AND PickCount < 10 THEN 'Move to Low Gravity'
        ELSE 'Optimal'
    END AS Recommendation
FROM ProductPerformance
WHERE CurrentZone IN ('High', 'Low')
ORDER BY PickCount DESC
```

### Pattern 2: Fleet Route Efficiency Analysis

```sql
-- Compare planned vs actual routes with delay attribution
SELECT 
    ft.VehicleID,
    ft.TripID,
    go.LocationName AS Origin,
    gd.LocationName AS Destination,
    ft.PlannedDistanceKM,
    ft.ActualDistanceKM,
    ft.ActualDistanceKM - ft.PlannedDistanceKM AS DistanceDeviation,
    ft.DelayTimeMinutes,
    ft.DelayReason,
    ft.WeatherCondition,
    ft.FuelConsumedLiters,
    (ft.FuelConsumedLiters / NULLIF(ft.ActualDistanceKM, 0)) AS LitersPer100KM,
    ft.IdleTimeMinutes,
    (CAST(ft.IdleTimeMinutes AS DECIMAL) / NULLIF(DATEDIFF(MINUTE, td.FullDateTime, ta.FullDateTime), 0)) * 100 AS IdlePercentage
FROM FactFleetTrips ft
INNER JOIN DimGeography go ON ft.GeographyKeyOrigin = go.GeographyKey
INNER JOIN DimGeography gd ON ft.GeographyKeyDestination = gd.GeographyKey
INNER JOIN DimTime td ON ft.TimeKeyDeparture = td.TimeKey
INNER JOIN DimTime ta ON ft.TimeKeyArrival = ta.TimeKey
WHERE ft.TimeKeyDeparture >= (SELECT MAX(TimeKey) - 2016 FROM DimTime) -- Last 3 weeks
  AND ft.DelayTimeMinutes > 30
ORDER BY ft.DelayTimeMinutes DESC
```

### Pattern 3: Cross-Fact Bottleneck Detection

```sql
-- Correlate warehouse dwell spikes with fleet delays
WITH WarehouseDwell AS (
    SELECT 
        wo.DateKey,
        AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations wo
    WHERE wo.OperationType IN ('Receiving', 'Putaway')
    GROUP BY wo.DateKey
),
FleetDelays AS (
    SELECT 
        CAST(CONVERT(VARCHAR(8), td.FullDateTime, 112) AS INT) AS DateKey,
        AVG(ft.DelayTimeMinutes) AS AvgDelay,
        COUNT(*) AS TripCount
    FROM FactFleetTrips ft
    INNER JOIN DimTime td ON ft.TimeKeyDeparture = td.TimeKey
    GROUP BY CAST(CONVERT(VARCHAR(8), td.FullDateTime, 112) AS INT)
)
SELECT 
    wd.DateKey,
    wd.AvgDwellTime,
    wd.OperationCount,
    fd.AvgDelay,
    fd.TripCount,
    CASE 
        WHEN wd.AvgDwellTime > 120 AND fd.AvgDelay > 45 THEN 'Critical Bottleneck'
        WHEN wd.AvgDwellTime > 90 OR fd.AvgDelay > 30 THEN 'Warning'
        ELSE 'Normal'
    END AS BottleneckStatus
FROM WarehouseDwell wd
LEFT JOIN FleetDelays fd ON wd.DateKey = fd.DateKey
WHERE wd.DateKey >= CAST(CONVERT(VARCHAR(8), DATEADD(DAY, -30, GETDATE()), 112) AS INT)
ORDER BY wd.DateKey DESC
```

### Pattern 4: Automated Scheduled Refresh

```sql
-- SQL Server Agent job or Azure Data Factory pipeline
CREATE PROCEDURE dbo.usp_DailyRefresh
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @StartTime DATETIME2 = DATEADD(DAY, -1, CAST(CAST(GETDATE() AS DATE) AS DATETIME2))
    DECLARE @EndTime DATETIME2 = CAST(CAST(GETDATE() AS DATE) AS DATETIME2)
    
    -- Refresh warehouse data
    EXEC dbo.usp_LoadWarehouseOperations @StartTime, @EndTime
    
    -- Refresh fleet data
    EXEC dbo.usp_LoadFleetTrips @StartTime, @EndTime
    
    -- Update gravity scores
    EXEC dbo.usp_UpdateProductGravityScores
    
    -- Check for alerts
    EXEC dbo.usp_CheckFleetIdlingThresholds 15.0
    EXEC dbo.usp_CheckWarehouseDwellThresholds 120
    
    -- Update summary tables
    EXEC dbo.usp_UpdateDailySummaries
END
GO
```

## Troubleshooting

### Issue: Power BI Report Refresh Fails

**Symptoms**: Error "Unable to connect to the data source" or timeout

**Solutions**:
1. Verify SQL Server firewall rules allow Power BI service IP ranges
2. Check connection string in Power BI Desktop:
   ```
   Server=tcp:yourserver.database.windows.net,1433;
   Initial Catalog=LogiFleetPulse;
   Persist Security Info=False;
   MultipleActiveResultSets=False;
   Encrypt=True;
   TrustServerCertificate=False;
   Connection Timeout=30;
   ```
3. Switch from DirectQuery to Import mode for large fact tables
4. Create indexed views for complex queries:
   ```sql
   CREATE VIEW vw_WarehousePerformance WITH SCHEMABINDING AS
   SELECT 
       wo.DateKey,
       wo.GeographyKey,
       SUM(wo.QuantityHandled) AS TotalQuantity,
       AVG(wo.DwellTimeMinutes) AS AvgDwell
   FROM dbo.FactWarehouseOperations wo
   GROUP BY wo.DateKey, wo.GeographyKey
   
   CREATE UNIQUE CLUSTERED INDEX IX_WHPerf ON vw_WarehousePerformance(DateKey, GeographyKey)
   ```

### Issue: Slow Query Performance on Cross-Fact KPIs

**Symptoms**: Queries joining FactWarehouseOperations and FactFleetTrips take >10 seconds

**Solutions**:
1. Create bridge table for common query patterns:
   ```sql
   CREATE TABLE BridgeWarehouseFleet (
       BridgeKey BIGINT IDENTITY(1,1) PRIMARY KEY,
       OperationKey BIGINT,
       TripKey BIGINT,
       DateKey INT,
       GeographyKey INT,
       CONSTRAINT FK_Bridge_WH FOREIGN KEY (OperationKey) REFERENCES FactWarehouseOperations(OperationKey),
       CONSTRAINT FK_Bridge_Fleet FOREIGN KEY (TripKey) REFERENCES FactFleetTrips(TripKey)
   )
   
   CREATE NONCLUSTERED INDEX IX_Bridge_Date_Geo ON BridgeWarehouseFleet(DateKey, GeographyKey)
   ```

2. Partition large fact tables by date:
   ```sql
   CREATE PARTITION FUNCTION pf_DateKey (INT)
   AS RANGE RIGHT FOR VALUES (20260101, 20260201, 20260301, 20260401, 20260501, 20260601)
   
   CREATE PARTITION SCHEME ps_DateKey
   AS PARTITION pf_DateKey ALL TO ([PRIMARY])
   
   -- Recreate table on partition scheme
   CREATE TABLE FactWarehouseOperations_Partitioned (
       ...columns...
   ) ON ps_DateKey(DateKey)
   ```

### Issue: Gravity Score Calculation Returns NULL

**Symptoms**: `GravityScore` column shows NULL values after running update procedure

**Solutions**:
1. Ensure sufficient historical data exists (minimum 7 days):
   ```sql
   SELECT COUNT(*), MIN(TimeKey), MAX(TimeKey)
   FROM FactWarehouseOperations
   ```

2. Handle division by zero in calculation:
   ```sql
   UPDATE p
   SET p.GravityScore = 
       CASE 
           WHEN pm.AvgCycleTime > 0 THEN 
               (pm.PickCount * pm.TotalQuantity * p.UnitValue) / pm.AvgCycleTime
           ELSE 0
       END
   ```

3. Verify product dimension is populated with unit values:
   ```sql
   SELECT COUNT(*) 
   FROM DimProductGravity 
   WHERE UnitValue IS NULL OR UnitValue = 0
   ```

### Issue: Row-Level Security Not Filtering Data

**Symptoms**: Users see data outside their authorized region/warehouse

**Solutions**:
1. Verify RLS roles are applied in Power BI Service (not just Desktop)
2. Test RLS in Power BI Desktop using "View as Role" feature
3. Check user mapping table exists and is current:
   ```sql
   SELECT * FROM SecurityUserMapping WHERE UserEmail = 'user@company.com'
   ```
4. Ensure RLS filter uses correct identifier:
   ```dax
   -- Use email, not display name
   USERPRINCIPALNAME() -- Returns email
   USERNAME() -- Returns domain\username (for Windows auth)
   ```

### Issue: External API Data Not Loading

**Symptoms**: Weather or traffic data missing from reports

**Solutions**:
1. Test API connectivity from SQL Server:
   ```sql
   -- Enable external scripts if using Python/R
   EXEC sp_configure 'external scripts enabled', 1
   RECONFIGURE
   
   -- Test API call via stored procedure
   EXEC dbo.usp_TestWeatherAPI
   ```

2. Implement retry logic with exponential backoff:
   ```sql
   CREATE PROCEDURE dbo.usp_LoadWeatherData
   AS
   BEGIN
       DECLARE @RetryCount INT = 0
       DECLARE @MaxRetries INT = 3
       
       WHILE @RetryCount < @MaxRetries
       BEGIN
           BEGIN TRY
               -- API call logic
               BREAK
           END TRY
           BEGIN CATCH
               SET @Ret
