---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server + Power BI logistics intelligence platform for warehouse, fleet, and supply chain analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "deploy the logistics data warehouse schema"
  - "configure Power BI for fleet and warehouse dashboards"
  - "create cross-fact KPI queries for supply chain"
  - "implement warehouse gravity zone optimization"
  - "build real-time fleet telemetry analytics"
  - "set up multi-modal logistics intelligence"
  - "integrate warehouse and fleet data models"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

LogiFleet Pulse is a multi-modal logistics intelligence engine that combines MS SQL Server data warehousing with Power BI visualization to unify warehouse operations, fleet telemetry, inventory management, and external signals into a single semantic layer. It features a multi-fact star schema with time-phased dimensions, cross-fact KPI harmonization, and predictive analytics for supply chain optimization.

**Key capabilities:**
- Unified data model linking warehouse, fleet, and supplier data
- Multi-fact star schema with 15-minute granularity time dimensions
- Real-time dashboards with predictive bottleneck detection
- Warehouse Gravity Zones™ for spatial optimization
- Fleet triage engine with proactive maintenance scoring
- Role-based access control and multilingual support

## Installation

### Prerequisites

- MS SQL Server 2019+ (supports polybase and external tables)
- Power BI Desktop (latest version)
- Access to data sources: WMS, TMS, telemetry APIs, ERP systems

### Database Setup

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**
```sql
-- Connect to your SQL Server instance
-- Run the main schema deployment script
:r schema/01_create_dimensions.sql
:r schema/02_create_facts.sql
:r schema/03_create_views.sql
:r schema/04_create_procedures.sql
:r schema/05_create_indexes.sql
```

3. **Configure data source connections:**
```json
// config_sample.json - rename to config.json
{
  "connections": {
    "wms_api": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "auth_token": "${WMS_API_TOKEN}"
    },
    "fleet_telemetry": {
      "endpoint": "${FLEET_TELEMETRY_ENDPOINT}",
      "auth_token": "${FLEET_TELEMETRY_TOKEN}"
    },
    "erp_database": {
      "server": "${ERP_SQL_SERVER}",
      "database": "${ERP_DATABASE}",
      "username": "${ERP_USERNAME}",
      "password": "${ERP_PASSWORD}"
    }
  }
}
```

4. **Set up incremental loading:**
```sql
-- Execute the incremental load stored procedure
EXEC dbo.sp_LoadWarehouseOperations @IncrementalFlag = 1;
EXEC dbo.sp_LoadFleetTrips @IncrementalFlag = 1;
EXEC dbo.sp_LoadCrossDockOperations @IncrementalFlag = 1;
```

### Power BI Configuration

1. **Open the template:**
```powershell
# Open the Power BI template file
Start-Process "LogiFleet_Pulse_Master.pbit"
```

2. **Configure connection parameters:**
- Server: Your SQL Server instance
- Database: LogiFleetPulse (or your database name)
- Data Connectivity mode: DirectQuery (for real-time) or Import (for performance)

3. **Set up scheduled refresh (Power BI Service):**
```json
{
  "refreshSchedule": {
    "frequency": "every_15_minutes",
    "timeZone": "UTC",
    "enabled": true
  }
}
```

## Core Data Model

### Dimension Tables

```sql
-- DimTime: 15-minute granularity time dimension
CREATE TABLE dbo.DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2,
    TimeOfDay TIME,
    Hour TINYINT,
    QuarterHour TINYINT,
    DayOfWeek VARCHAR(20),
    IsBusinessDay BIT,
    FiscalPeriod VARCHAR(10)
);

-- DimGeography: Hierarchical location dimension
CREATE TABLE dbo.DimGeography (
    GeographyKey INT PRIMARY KEY,
    LocationCode VARCHAR(50),
    LocationName VARCHAR(200),
    LocationType VARCHAR(50), -- Warehouse, Distribution Center, Route Node
    Region VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(10,6),
    Longitude DECIMAL(10,6)
);

-- DimProductGravity: Product with velocity scoring
CREATE TABLE dbo.DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(100),
    ProductName VARCHAR(500),
    Category VARCHAR(100),
    GravityScore DECIMAL(5,2), -- Composite: velocity + value + fragility
    VelocityRank INT,
    ValueTier VARCHAR(20),
    FragilityIndex DECIMAL(3,2),
    OptimalZone VARCHAR(50)
);

-- DimSupplierReliability: Supplier performance metrics
CREATE TABLE dbo.DimSupplierReliability (
    SupplierKey INT PRIMARY KEY,
    SupplierCode VARCHAR(50),
    SupplierName VARCHAR(200),
    LeadTimeVariance DECIMAL(5,2),
    DefectPercentage DECIMAL(5,4),
    ComplianceScore DECIMAL(5,2),
    ReliabilityTier VARCHAR(20)
);
```

### Fact Tables

```sql
-- FactWarehouseOperations: Core warehouse activity
CREATE TABLE dbo.FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType VARCHAR(50), -- Receiving, Putaway, Picking, Packing, Shipping
    QuantityHandled INT,
    DwellTimeMinutes INT,
    ProcessingTimeMinutes INT,
    ZoneCode VARCHAR(50),
    EmployeeID VARCHAR(50),
    BatchNumber VARCHAR(100)
);

-- FactFleetTrips: Vehicle telemetry and trip data
CREATE TABLE dbo.FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID VARCHAR(50),
    DriverID VARCHAR(50),
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    LoadWeightKG INT,
    TripDurationMinutes INT,
    DelayMinutes INT,
    DelayReason VARCHAR(200)
);

-- FactCrossDock: Cross-docking operations
CREATE TABLE dbo.FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    QuantityTransferred INT,
    DockTimeMinutes INT,
    CrossDockType VARCHAR(50) -- Direct, Temporary Hold
);
```

## Key SQL Queries

### Cross-Fact KPI: Fleet Idling Cost per Route with Warehouse Dwell Time

```sql
-- Correlate fleet idle time with warehouse dwell time to identify bottlenecks
WITH WarehouseDwell AS (
    SELECT 
        w.GeographyKey,
        g.LocationName AS Warehouse,
        AVG(w.DwellTimeMinutes) AS AvgDwellTime,
        COUNT(*) AS OperationCount
    FROM dbo.FactWarehouseOperations w
    JOIN dbo.DimGeography g ON w.GeographyKey = g.GeographyKey
    WHERE w.TimeKey >= (SELECT MAX(TimeKey) - 672 FROM DimTime) -- Last week
    GROUP BY w.GeographyKey, g.LocationName
),
FleetIdle AS (
    SELECT 
        f.OriginGeographyKey,
        AVG(f.IdleTimeMinutes) AS AvgIdleTime,
        SUM(f.FuelConsumedLiters) AS TotalFuel,
        COUNT(*) AS TripCount
    FROM dbo.FactFleetTrips f
    WHERE f.TimeKey >= (SELECT MAX(TimeKey) - 672 FROM DimTime)
    GROUP BY f.OriginGeographyKey
)
SELECT 
    wd.Warehouse,
    wd.AvgDwellTime,
    fi.AvgIdleTime,
    fi.TotalFuel,
    (fi.AvgIdleTime / NULLIF(wd.AvgDwellTime, 0)) AS IdleToDwellRatio,
    CASE 
        WHEN (fi.AvgIdleTime / NULLIF(wd.AvgDwellTime, 0)) > 0.5 THEN 'High Friction'
        WHEN (fi.AvgIdleTime / NULLIF(wd.AvgDwellTime, 0)) > 0.3 THEN 'Moderate Friction'
        ELSE 'Low Friction'
    END AS FrictionCategory
FROM WarehouseDwell wd
JOIN FleetIdle fi ON wd.GeographyKey = fi.OriginGeographyKey
ORDER BY IdleToDwellRatio DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify products in suboptimal zones based on gravity score
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.OptimalZone AS RecommendedZone,
    w.ZoneCode AS CurrentZone,
    AVG(w.DwellTimeMinutes) AS AvgDwellTime,
    COUNT(*) AS PickFrequency,
    CASE 
        WHEN p.OptimalZone <> w.ZoneCode THEN 'Needs Relocation'
        ELSE 'Optimal'
    END AS ZoneStatus
FROM dbo.FactWarehouseOperations w
JOIN dbo.DimProductGravity p ON w.ProductKey = p.ProductKey
WHERE w.OperationType = 'Picking'
    AND w.TimeKey >= (SELECT MAX(TimeKey) - 2880 FROM DimTime) -- Last 30 days
GROUP BY p.SKU, p.ProductName, p.GravityScore, p.OptimalZone, w.ZoneCode
HAVING AVG(w.DwellTimeMinutes) > 60 -- Focus on slow-moving items
ORDER BY p.GravityScore DESC, AvgDwellTime DESC;
```

### Predictive Bottleneck Detection

```sql
-- Identify probable congestion points using temporal patterns
WITH HourlyLoad AS (
    SELECT 
        t.Hour,
        t.DayOfWeek,
        g.LocationName,
        COUNT(*) AS OperationCount,
        AVG(w.ProcessingTimeMinutes) AS AvgProcessingTime
    FROM dbo.FactWarehouseOperations w
    JOIN dbo.DimTime t ON w.TimeKey = t.TimeKey
    JOIN dbo.DimGeography g ON w.GeographyKey = g.GeographyKey
    WHERE t.FullDateTime >= DATEADD(DAY, -90, GETDATE())
    GROUP BY t.Hour, t.DayOfWeek, g.LocationName
)
SELECT 
    LocationName,
    DayOfWeek,
    Hour,
    OperationCount,
    AvgProcessingTime,
    AVG(OperationCount) OVER (PARTITION BY LocationName) AS LocationAvg,
    (OperationCount - AVG(OperationCount) OVER (PARTITION BY LocationName)) / 
        NULLIF(STDEV(OperationCount) OVER (PARTITION BY LocationName), 0) AS ZScore,
    CASE 
        WHEN (OperationCount - AVG(OperationCount) OVER (PARTITION BY LocationName)) / 
             NULLIF(STDEV(OperationCount) OVER (PARTITION BY LocationName), 0) > 2 THEN 'High Risk'
        WHEN (OperationCount - AVG(OperationCount) OVER (PARTITION BY LocationName)) / 
             NULLIF(STDEV(OperationCount) OVER (PARTITION BY LocationName), 0) > 1 THEN 'Moderate Risk'
        ELSE 'Normal'
    END AS BottleneckRisk
FROM HourlyLoad
WHERE (OperationCount - AVG(OperationCount) OVER (PARTITION BY LocationName)) / 
      NULLIF(STDEV(OperationCount) OVER (PARTITION BY LocationName), 0) > 1
ORDER BY ZScore DESC;
```

### Fleet Maintenance Triage

```sql
-- Prioritize fleet maintenance by revenue impact
WITH FleetHealth AS (
    SELECT 
        VehicleID,
        AVG(IdleTimeMinutes * 1.0 / NULLIF(TripDurationMinutes, 0)) AS IdleRatio,
        SUM(FuelConsumedLiters) AS TotalFuel,
        COUNT(*) AS TripCount,
        AVG(DelayMinutes) AS AvgDelay
    FROM dbo.FactFleetTrips
    WHERE TimeKey >= (SELECT MAX(TimeKey) - 2880 FROM DimTime) -- Last 30 days
    GROUP BY VehicleID
),
LoadValue AS (
    SELECT 
        f.VehicleID,
        SUM(p.GravityScore * w.QuantityHandled) AS TotalValueTransported
    FROM dbo.FactFleetTrips f
    JOIN dbo.FactWarehouseOperations w ON f.TimeKey = w.TimeKey
    JOIN dbo.DimProductGravity p ON w.ProductKey = p.ProductKey
    WHERE f.TimeKey >= (SELECT MAX(TimeKey) - 2880 FROM DimTime)
    GROUP BY f.VehicleID
)
SELECT 
    fh.VehicleID,
    fh.IdleRatio,
    fh.TotalFuel,
    fh.AvgDelay,
    ISNULL(lv.TotalValueTransported, 0) AS ValueAtRisk,
    (fh.IdleRatio * 100) + (fh.AvgDelay * 0.5) + 
        (ISNULL(lv.TotalValueTransported, 0) / 1000) AS MaintenancePriorityScore
FROM FleetHealth fh
LEFT JOIN LoadValue lv ON fh.VehicleID = lv.VehicleID
WHERE fh.IdleRatio > 0.15 OR fh.AvgDelay > 10
ORDER BY MaintenancePriorityScore DESC;
```

## Stored Procedures

### Incremental Data Loading

```sql
-- Incremental warehouse operations load
CREATE PROCEDURE dbo.sp_LoadWarehouseOperations
    @IncrementalFlag BIT = 1,
    @LastLoadTime DATETIME2 = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Get last successful load time if not provided
    IF @LastLoadTime IS NULL
        SELECT @LastLoadTime = ISNULL(MAX(LastLoadTime), '2020-01-01')
        FROM dbo.ETL_LoadLog 
        WHERE TableName = 'FactWarehouseOperations' AND Status = 'Success';
    
    -- Insert new records from staging
    INSERT INTO dbo.FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        QuantityHandled, DwellTimeMinutes, ProcessingTimeMinutes,
        ZoneCode, EmployeeID, BatchNumber
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.OperationType,
        s.QuantityHandled,
        DATEDIFF(MINUTE, s.StartTime, s.EndTime) AS DwellTimeMinutes,
        s.ProcessingTimeMinutes,
        s.ZoneCode,
        s.EmployeeID,
        s.BatchNumber
    FROM staging.WarehouseOperations s
    JOIN dbo.DimTime t ON CAST(s.StartTime AS DATETIME2) = t.FullDateTime
    JOIN dbo.DimGeography g ON s.LocationCode = g.LocationCode
    JOIN dbo.DimProductGravity p ON s.SKU = p.SKU
    WHERE s.LastModified > @LastLoadTime;
    
    -- Log the load
    INSERT INTO dbo.ETL_LoadLog (TableName, LastLoadTime, RecordsLoaded, Status)
    VALUES ('FactWarehouseOperations', GETDATE(), @@ROWCOUNT, 'Success');
END;
```

### Automated Alerting

```sql
-- Create alerts for KPI threshold breaches
CREATE PROCEDURE dbo.sp_GenerateAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Fleet idle time alert (>15% of trip duration)
    INSERT INTO dbo.Alerts (AlertType, Severity, Message, CreatedAt)
    SELECT 
        'Fleet Idle Time',
        'High',
        'Vehicle ' + VehicleID + ' has idle time >15%: ' + 
            CAST(CAST(AvgIdleRatio * 100 AS DECIMAL(5,2)) AS VARCHAR) + '%',
        GETDATE()
    FROM (
        SELECT 
            VehicleID,
            AVG(IdleTimeMinutes * 1.0 / NULLIF(TripDurationMinutes, 0)) AS AvgIdleRatio
        FROM dbo.FactFleetTrips
        WHERE TimeKey >= (SELECT MAX(TimeKey) - 96 FROM DimTime) -- Last 24 hours
        GROUP BY VehicleID
    ) f
    WHERE AvgIdleRatio > 0.15;
    
    -- Warehouse dwell time alert (>72 hours)
    INSERT INTO dbo.Alerts (AlertType, Severity, Message, CreatedAt)
    SELECT 
        'Warehouse Dwell Time',
        'Medium',
        'SKU ' + p.SKU + ' at ' + g.LocationName + ' has dwell time ' + 
            CAST(DwellTimeMinutes AS VARCHAR) + ' minutes',
        GETDATE()
    FROM dbo.FactWarehouseOperations w
    JOIN dbo.DimProductGravity p ON w.ProductKey = p.ProductKey
    JOIN dbo.DimGeography g ON w.GeographyKey = g.GeographyKey
    WHERE w.DwellTimeMinutes > 4320 -- 72 hours
        AND w.TimeKey >= (SELECT MAX(TimeKey) - 96 FROM DimTime);
END;
```

## Power BI DAX Measures

### Cross-Fact KPIs

```dax
// Total Fleet Cost per Warehouse
Fleet Cost per Warehouse = 
VAR FuelCostPerLiter = 1.50
VAR IdleCostPerMinute = 0.75
RETURN
CALCULATE(
    SUM(FactFleetTrips[FuelConsumedLiters]) * FuelCostPerLiter +
    SUM(FactFleetTrips[IdleTimeMinutes]) * IdleCostPerMinute,
    USERELATIONSHIP(FactFleetTrips[OriginGeographyKey], DimGeography[GeographyKey])
)

// Warehouse Efficiency Score
Warehouse Efficiency Score = 
VAR AvgProcessingTime = AVERAGE(FactWarehouseOperations[ProcessingTimeMinutes])
VAR BenchmarkTime = 15
VAR AvgDwellTime = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR BenchmarkDwell = 120
RETURN
((BenchmarkTime / NULLIF(AvgProcessingTime, 0)) * 0.5 +
 (BenchmarkDwell / NULLIF(AvgDwellTime, 0)) * 0.5) * 100

// Predictive Bottleneck Index
Bottleneck Index = 
VAR CurrentLoad = COUNTROWS(FactWarehouseOperations)
VAR HistoricalAvg = CALCULATE(
    COUNTROWS(FactWarehouseOperations),
    DATEADD(DimTime[FullDateTime], -7, DAY)
)
VAR StdDev = STDEV.P(FactWarehouseOperations[ProcessingTimeMinutes])
RETURN
(CurrentLoad - HistoricalAvg) / NULLIF(StdDev, 0)

// Fleet Utilization Rate
Fleet Utilization % = 
DIVIDE(
    SUM(FactFleetTrips[TripDurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[TripDurationMinutes])
) * 100
```

### Time Intelligence

```dax
// Rolling 30-Day Dwell Time Average
Rolling 30D Dwell Time = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    DATESINPERIOD(DimTime[FullDateTime], LASTDATE(DimTime[FullDateTime]), -30, DAY)
)

// Year-over-Year Fleet Fuel Efficiency
YoY Fuel Efficiency Change = 
VAR CurrentYearFuel = 
    CALCULATE(
        DIVIDE(SUM(FactFleetTrips[DistanceKM]), SUM(FactFleetTrips[FuelConsumedLiters])),
        DATESYTD(DimTime[FullDateTime])
    )
VAR PriorYearFuel = 
    CALCULATE(
        DIVIDE(SUM(FactFleetTrips[DistanceKM]), SUM(FactFleetTrips[FuelConsumedLiters])),
        SAMEPERIODLASTYEAR(DATESYTD(DimTime[FullDateTime]))
    )
RETURN
DIVIDE(CurrentYearFuel - PriorYearFuel, PriorYearFuel) * 100
```

## Configuration & Security

### Row-Level Security

```sql
-- Create roles for departmental access
CREATE TABLE dbo.SecurityRoles (
    RoleID INT PRIMARY KEY IDENTITY,
    RoleName VARCHAR(100),
    Department VARCHAR(100),
    GeographyFilter VARCHAR(500), -- Comma-separated geography keys
    ProductFilter VARCHAR(500)     -- Comma-separated product categories
);

-- Sample role definitions
INSERT INTO dbo.SecurityRoles (RoleName, Department, GeographyFilter, ProductFilter)
VALUES 
    ('Warehouse_Manager_East', 'Operations', '1,2,3', NULL),
    ('Fleet_Supervisor_West', 'Logistics', '4,5,6', NULL),
    ('Executive_View', 'Executive', NULL, NULL);
```

**Power BI RLS DAX:**
```dax
// Apply in Power BI Model > Manage Roles
[GeographyKey] IN {1, 2, 3}  // For Warehouse_Manager_East
```

### Environment Variables

```bash
# .env file for configuration
export SQL_SERVER="your-server.database.windows.net"
export SQL_DATABASE="LogiFleetPulse"
export SQL_USERNAME="logifleet_admin"
export SQL_PASSWORD="${SQL_ADMIN_PASSWORD}"
export WMS_API_ENDPOINT="https://wms-api.company.com/v2"
export WMS_API_TOKEN="${WMS_API_KEY}"
export FLEET_TELEMETRY_ENDPOINT="https://fleet-telemetry.company.com/api"
export FLEET_TELEMETRY_TOKEN="${FLEET_TELEMETRY_KEY}"
export POWERBI_WORKSPACE_ID="${POWERBI_WORKSPACE_GUID}"
export ALERT_EMAIL="logistics-alerts@company.com"
export ALERT_TEAMS_WEBHOOK="${TEAMS_WEBHOOK_URL}"
```

## Common Patterns

### Pattern 1: Adding a New Data Source

```sql
-- 1. Create staging table
CREATE TABLE staging.NewDataSource (
    RecordID BIGINT PRIMARY KEY,
    SourceTimestamp DATETIME2,
    LocationCode VARCHAR(50),
    MetricValue DECIMAL(18,2),
    LastModified DATETIME2
);

-- 2. Create dimension/fact table
CREATE TABLE dbo.FactNewMetric (
    MetricKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    MetricValue DECIMAL(18,2),
    SourceSystem VARCHAR(50)
);

-- 3. Create ETL stored procedure
CREATE PROCEDURE dbo.sp_LoadNewMetric
AS
BEGIN
    INSERT INTO dbo.FactNewMetric (TimeKey, GeographyKey, MetricValue, SourceSystem)
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        s.MetricValue,
        'NewSource'
    FROM staging.NewDataSource s
    JOIN dbo.DimTime t ON s.SourceTimestamp = t.FullDateTime
    JOIN dbo.DimGeography g ON s.LocationCode = g.LocationCode;
END;
```

### Pattern 2: Creating Composite KPIs

```sql
-- Create a view for complex cross-fact calculations
CREATE VIEW dbo.vw_CompositeSupplyChainScore AS
WITH WarehouseMetrics AS (
    SELECT 
        GeographyKey,
        AVG(ProcessingTimeMinutes) AS AvgProcessing,
        AVG(DwellTimeMinutes) AS AvgDwell
    FROM dbo.FactWarehouseOperations
    GROUP BY GeographyKey
),
FleetMetrics AS (
    SELECT 
        OriginGeographyKey,
        AVG(IdleTimeMinutes * 1.0 / NULLIF(TripDurationMinutes, 0)) AS AvgIdleRatio,
        AVG(DelayMinutes) AS AvgDelay
    FROM dbo.FactFleetTrips
    GROUP BY OriginGeographyKey
)
SELECT 
    g.LocationName,
    -- Composite score: lower is better
    (wm.AvgProcessing / 15.0) * 30 +           -- Processing efficiency (30% weight)
    (wm.AvgDwell / 120.0) * 30 +               -- Dwell efficiency (30% weight)
    (fm.AvgIdleRatio * 100) * 20 +             -- Fleet idle (20% weight)
    (fm.AvgDelay / 10.0) * 20                  -- Delay impact (20% weight)
    AS CompositeScore
FROM dbo.DimGeography g
JOIN WarehouseMetrics wm ON g.GeographyKey = wm.GeographyKey
JOIN FleetMetrics fm ON g.GeographyKey = fm.OriginGeographyKey;
```

### Pattern 3: Temporal Simulation Queries

```sql
-- Simulate impact of capacity change on fleet utilization
DECLARE @CapacityIncrease DECIMAL(5,2) = 0.15; -- 15% increase

WITH SimulatedLoad AS (
    SELECT 
        VehicleID,
        TripDurationMinutes,
        LoadWeightKG * (1 + @CapacityIncrease) AS SimulatedLoadKG,
        FuelConsumedLiters * (1 + (@CapacityIncrease * 0.7)) AS SimulatedFuel -- Assume 70% fuel correlation
    FROM dbo.FactFleetTrips
    WHERE TimeKey >= (SELECT MAX(TimeKey) - 2880 FROM DimTime)
)
SELECT 
    'Current' AS Scenario,
    AVG(LoadWeightKG) AS AvgLoad,
    AVG(FuelConsumedLiters) AS AvgFuel,
    SUM(FuelConsumedLiters) * 1.50 AS TotalFuelCost
FROM dbo.FactFleetTrips
WHERE TimeKey >= (SELECT MAX(TimeKey) - 2880 FROM DimTime)
UNION ALL
SELECT 
    'Simulated +15%' AS Scenario,
    AVG(SimulatedLoadKG) AS AvgLoad,
    AVG(SimulatedFuel) AS AvgFuel,
    SUM(SimulatedFuel) * 1.50 AS TotalFuelCost
FROM SimulatedLoad;
```

## Troubleshooting

### Issue: Power BI Reports Timing Out

**Symptom:** DirectQuery dashboards take >30 seconds to load

**Solution:**
```sql
-- Create indexed views for common aggregations
CREATE VIEW dbo.vw_DailyWarehouseSummary
WITH SCHEMABINDING
AS
SELECT 
    w.GeographyKey,
    t.TimeKey,
    COUNT_BIG(*) AS OperationCount,
    SUM(CAST(w.QuantityHandled AS BIGINT)) AS TotalQuantity,
    AVG(CAST(w.DwellTimeMinutes AS DECIMAL(18,2))) AS AvgDwellTime
FROM dbo.FactWarehouseOperations w
JOIN dbo.DimTime t ON w.TimeKey = t.TimeKey
GROUP BY w.GeographyKey, t.TimeKey;

CREATE UNIQUE CLUSTERED INDEX IX_DailyWarehouseSummary
ON dbo.vw_DailyWarehouseSummary (GeographyKey, TimeKey);
```

### Issue: Incremental Load Duplicates

**Symptom:** Duplicate records after ETL runs

**Solution:**
```sql
-- Add deduplication logic to stored procedure
CREATE PROCEDURE dbo.sp_LoadWarehouseOperations_Safe
AS
BEGIN
    -- Use MERGE for upsert behavior
    MERGE dbo.FactWarehouseOperations AS target
    USING (
        SELECT DISTINCT
            t.TimeKey,
            g.GeographyKey,
            p.ProductKey,
            s.OperationType,
            s.QuantityHandled,
            DATEDIFF(MINUTE, s.StartTime, s.EndTime) AS DwellTimeMinutes,
            s.ProcessingTimeMinutes,
            s.ZoneCode,
            s.EmployeeID,
            s.BatchNumber
        FROM staging.War
