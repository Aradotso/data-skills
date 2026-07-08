---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehouse for logistics, fleet, and supply chain analytics with multi-fact star schema
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "deploy the logistics data warehouse schema"
  - "configure Power BI for fleet and warehouse tracking"
  - "implement cross-fact KPI reporting for logistics"
  - "create supply chain intelligence dashboard"
  - "integrate warehouse and fleet telemetry data"
  - "build multi-modal logistics analytics"
  - "query cross-dock operations with star schema"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive data warehousing and analytics solution for supply chain operations. It combines MS SQL Server backend (multi-fact star schema) with Power BI dashboards to unify warehouse operations, fleet telemetry, inventory management, and external data sources (weather, traffic) into a single semantic layer.

**Core capabilities:**
- Multi-fact star schema with time-phased dimensions
- Cross-fact KPI harmonization (warehouse + fleet metrics)
- Real-time operational dashboards (15-minute refresh)
- Predictive bottleneck detection
- Warehouse gravity zone optimization
- Fleet maintenance triage engine

**Primary language:** SQL (MS SQL Server 2019+), DAX (Power BI)

## Installation

### Prerequisites

- MS SQL Server 2019 or later
- Power BI Desktop (latest version)
- Database user with CREATE TABLE, CREATE PROCEDURE, CREATE VIEW permissions
- Optional: Azure Synapse Analytics for big data enrichment

### Step 1: Deploy SQL Schema

```sql
-- Connect to your MS SQL Server instance
-- Ensure you have a dedicated database (e.g., LogiFleetPulse)

USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Deploy the schema files in order:
-- 1. Create dimension tables
-- 2. Create fact tables
-- 3. Create relationships and indexes
-- 4. Create stored procedures for incremental loading
-- 5. Create views for Power BI

-- Example: Create DimTime (time-phased dimension)
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    FullDateTime DATETIME NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME NOT NULL,
    HourOfDay TINYINT NOT NULL,
    MinuteBucket TINYINT NOT NULL, -- 0, 15, 30, 45
    DayOfWeek TINYINT NOT NULL,
    DayName VARCHAR(10) NOT NULL,
    IsWeekend BIT NOT NULL,
    FiscalYear INT NOT NULL,
    FiscalQuarter TINYINT NOT NULL,
    FiscalPeriod TINYINT NOT NULL
);
GO

CREATE UNIQUE INDEX IX_DimTime_DateTime ON DimTime(FullDateTime);
GO

-- Create DimGeography (hierarchical location dimension)
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) UNIQUE NOT NULL,
    LocationName VARCHAR(200) NOT NULL,
    LocationType VARCHAR(50) NOT NULL, -- Warehouse, RouteNode, CrossDock
    ParentLocationID VARCHAR(50),
    Region VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    IsActive BIT DEFAULT 1,
    ValidFrom DATETIME NOT NULL DEFAULT GETDATE(),
    ValidTo DATETIME NULL
);
GO

CREATE INDEX IX_DimGeography_LocationID ON DimGeography(LocationID);
CREATE INDEX IX_DimGeography_Type ON DimGeography(LocationType);
GO

-- Create DimProduct with Gravity Score
CREATE TABLE DimProduct (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName VARCHAR(200) NOT NULL,
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    IsFragile BIT DEFAULT 0,
    IsPerishable BIT DEFAULT 0,
    -- Gravity zone calculation inputs
    AvgPickFrequency DECIMAL(10,2), -- picks per day
    UnitValue DECIMAL(12,2),
    ReplenishmentLeadTimeDays INT,
    GravityScore AS (
        (AvgPickFrequency * 0.5) + 
        (UnitValue * 0.3) + 
        (CASE WHEN IsFragile = 1 THEN 20 ELSE 0 END) +
        (CASE WHEN IsPerishable = 1 THEN 25 ELSE 0 END) -
        (ReplenishmentLeadTimeDays * 0.5)
    ) PERSISTED,
    GravityZone AS (
        CASE 
            WHEN (
                (AvgPickFrequency * 0.5) + 
                (UnitValue * 0.3) + 
                (CASE WHEN IsFragile = 1 THEN 20 ELSE 0 END) +
                (CASE WHEN IsPerishable = 1 THEN 25 ELSE 0 END) -
                (ReplenishmentLeadTimeDays * 0.5)
            ) > 80 THEN 'High'
            WHEN (
                (AvgPickFrequency * 0.5) + 
                (UnitValue * 0.3) + 
                (CASE WHEN IsFragile = 1 THEN 20 ELSE 0 END) +
                (CASE WHEN IsPerishable = 1 THEN 25 ELSE 0 END) -
                (ReplenishmentLeadTimeDays * 0.5)
            ) > 40 THEN 'Medium'
            ELSE 'Low'
        END
    ) PERSISTED,
    ValidFrom DATETIME NOT NULL DEFAULT GETDATE(),
    ValidTo DATETIME NULL
);
GO

CREATE INDEX IX_DimProduct_SKU ON DimProduct(SKU);
CREATE INDEX IX_DimProduct_GravityZone ON DimProduct(GravityZone);
GO
```

### Step 2: Create Fact Tables

```sql
-- FactWarehouseOperations: Core warehouse activity tracking
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProduct(ProductKey),
    OperationType VARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    OperationID VARCHAR(100) UNIQUE NOT NULL,
    Quantity INT NOT NULL,
    DurationMinutes DECIMAL(10,2),
    StorageZone VARCHAR(50),
    DwellTimeHours DECIMAL(10,2), -- Time between operations
    EmployeeID VARCHAR(50),
    BatchNumber VARCHAR(100),
    OrderID VARCHAR(100),
    TemperatureAtOperation DECIMAL(5,2), -- For cold chain tracking
    CreatedAt DATETIME DEFAULT GETDATE()
);
GO

CREATE INDEX IX_FactWarehouseOps_Time ON FactWarehouseOperations(TimeKey);
CREATE INDEX IX_FactWarehouseOps_Product ON FactWarehouseOperations(ProductKey);
CREATE INDEX IX_FactWarehouseOps_Type ON FactWarehouseOperations(OperationType);
CREATE INDEX IX_FactWarehouseOps_OrderID ON FactWarehouseOperations(OrderID);
GO

-- FactFleetTrips: Vehicle and route tracking
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50),
    TripID VARCHAR(100) UNIQUE NOT NULL,
    DistanceKm DECIMAL(10,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelLiters DECIMAL(10,2),
    AverageSpeed DECIMAL(5,2),
    LoadWeight DECIMAL(10,2),
    LoadVolume DECIMAL(10,2),
    OnTimeDelivery BIT,
    DelayReasonCode VARCHAR(50), -- Weather, Traffic, Breakdown, etc.
    TirePressureAlert BIT DEFAULT 0,
    EngineDiagnosticCode VARCHAR(20),
    CreatedAt DATETIME DEFAULT GETDATE()
);
GO

CREATE INDEX IX_FactFleetTrips_Time ON FactFleetTrips(TimeKey);
CREATE INDEX IX_FactFleetTrips_Origin ON FactFleetTrips(OriginGeographyKey);
CREATE INDEX IX_FactFleetTrips_Destination ON FactFleetTrips(DestinationGeographyKey);
CREATE INDEX IX_FactFleetTrips_Vehicle ON FactFleetTrips(VehicleID);
GO

-- Bridge table for many-to-many: Trips to Warehouse Operations
CREATE TABLE BridgeTripOperations (
    TripKey BIGINT NOT NULL FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OperationKey BIGINT NOT NULL FOREIGN KEY REFERENCES FactWarehouseOperations(OperationKey),
    SequenceNumber INT,
    PRIMARY KEY (TripKey, OperationKey)
);
GO
```

### Step 3: Create Views for Power BI

```sql
-- View: Cross-fact KPI - Dwell Time vs Fleet Idling Cost
CREATE VIEW vw_DwellTimeVsFleetIdling AS
SELECT 
    t.FiscalYear,
    t.FiscalQuarter,
    g.Region,
    p.Category,
    AVG(wo.DwellTimeHours) AS AvgDwellTimeHours,
    AVG(ft.IdleTimeMinutes) AS AvgIdleTimeMinutes,
    AVG(ft.IdleTimeMinutes * 0.8) AS EstimatedIdleCostUSD, -- $0.8 per minute
    COUNT(DISTINCT wo.OperationKey) AS TotalOperations,
    COUNT(DISTINCT ft.TripKey) AS TotalTrips
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
INNER JOIN BridgeTripOperations bto ON wo.OperationKey = bto.OperationKey
INNER JOIN FactFleetTrips ft ON bto.TripKey = ft.TripKey
GROUP BY t.FiscalYear, t.FiscalQuarter, g.Region, p.Category;
GO

-- View: Warehouse Gravity Zone Performance
CREATE VIEW vw_GravityZonePerformance AS
SELECT 
    p.GravityZone,
    wo.StorageZone,
    COUNT(*) AS TotalPicks,
    AVG(wo.DurationMinutes) AS AvgPickDurationMin,
    AVG(wo.DwellTimeHours) AS AvgDwellHours,
    SUM(wo.Quantity) AS TotalQuantityPicked
FROM FactWarehouseOperations wo
INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
WHERE wo.OperationType = 'Picking'
GROUP BY p.GravityZone, wo.StorageZone;
GO

-- View: Fleet Maintenance Priority
CREATE VIEW vw_FleetMaintenancePriority AS
SELECT 
    ft.VehicleID,
    COUNT(CASE WHEN ft.TirePressureAlert = 1 THEN 1 END) AS TirePressureAlerts,
    COUNT(CASE WHEN ft.EngineDiagnosticCode IS NOT NULL THEN 1 END) AS EngineAlerts,
    AVG(ft.LoadWeight) AS AvgLoadWeight,
    SUM(CASE WHEN p.GravityZone = 'High' THEN 1 ELSE 0 END) AS HighValueTrips,
    -- Priority score: alerts + high-value cargo weight
    (
        COUNT(CASE WHEN ft.TirePressureAlert = 1 THEN 1 END) * 10 +
        COUNT(CASE WHEN ft.EngineDiagnosticCode IS NOT NULL THEN 1 END) * 15 +
        AVG(ft.LoadWeight) * 0.01 +
        SUM(CASE WHEN p.GravityZone = 'High' THEN 1 ELSE 0 END) * 5
    ) AS MaintenancePriorityScore
FROM FactFleetTrips ft
INNER JOIN BridgeTripOperations bto ON ft.TripKey = bto.TripKey
INNER JOIN FactWarehouseOperations wo ON bto.OperationKey = wo.OperationKey
INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
WHERE ft.CreatedAt >= DATEADD(MONTH, -1, GETDATE())
GROUP BY ft.VehicleID;
GO
```

### Step 4: Create Incremental Load Procedures

```sql
-- Stored procedure for incremental warehouse operations load
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Default to last 24 hours if not specified
    IF @LastLoadDateTime IS NULL
        SET @LastLoadDateTime = DATEADD(HOUR, -24, GETDATE());
    
    -- Insert new operations from staging/source system
    -- Replace [SourceDB].[dbo].[StagingWarehouseOps] with your actual source
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        OperationID, Quantity, DurationMinutes, StorageZone,
        DwellTimeHours, EmployeeID, BatchNumber, OrderID,
        TemperatureAtOperation
    )
    SELECT 
        dt.TimeKey,
        dg.GeographyKey,
        dp.ProductKey,
        src.OperationType,
        src.OperationID,
        src.Quantity,
        src.DurationMinutes,
        src.StorageZone,
        src.DwellTimeHours,
        src.EmployeeID,
        src.BatchNumber,
        src.OrderID,
        src.TemperatureAtOperation
    FROM [SourceDB].[dbo].[StagingWarehouseOps] src
    INNER JOIN DimTime dt ON CAST(src.OperationDateTime AS DATETIME) = dt.FullDateTime
    INNER JOIN DimGeography dg ON src.WarehouseID = dg.LocationID
    INNER JOIN DimProduct dp ON src.SKU = dp.SKU
    WHERE src.OperationDateTime > @LastLoadDateTime
        AND NOT EXISTS (
            SELECT 1 FROM FactWarehouseOperations 
            WHERE OperationID = src.OperationID
        );
    
    PRINT 'Loaded ' + CAST(@@ROWCOUNT AS VARCHAR) + ' warehouse operations';
END;
GO
```

### Step 5: Configure Power BI Template

Download the repository and open `LogiFleet_Pulse_Master.pbit`:

1. **Set up connection:**
   - File → Options → Data source settings
   - Add SQL Server connection: `Server=YOUR_SERVER;Database=LogiFleetPulse`
   - Authentication: Windows or SQL Server (use environment variable for password)

2. **Configure refresh schedule:**
   - In Power BI Service, set up refresh every 15 minutes
   - Enable incremental refresh for fact tables (1 year history, refresh last 7 days)

3. **Set up row-level security:**
   ```dax
   -- In Power BI Desktop, create role "RegionalManager"
   [Region] = USERPRINCIPALNAME()
   ```

## Key DAX Measures for Power BI

```dax
// Measure: Average Dwell Time (warehouse operations)
AvgDwellTime = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeHours]),
    FactWarehouseOperations[OperationType] IN {"Picking", "Packing"}
)

// Measure: Fleet Utilization Rate
FleetUtilizationRate = 
DIVIDE(
    SUM(FactFleetTrips[DurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[DurationMinutes]),
    0
) * 100

// Measure: Cross-Fact - Cost Per Pick (warehouse labor + fleet idle cost)
CostPerPick = 
VAR WarehousePickTime = 
    CALCULATE(
        SUM(FactWarehouseOperations[DurationMinutes]),
        FactWarehouseOperations[OperationType] = "Picking"
    )
VAR WarehouseLaborCost = WarehousePickTime * 0.5 // $0.5 per minute
VAR AssociatedIdleTime = 
    CALCULATE(
        SUM(FactFleetTrips[IdleTimeMinutes]),
        USERELATIONSHIP(FactFleetTrips[TripKey], BridgeTripOperations[TripKey])
    )
VAR FleetIdleCost = AssociatedIdleTime * 0.8 // $0.8 per minute
VAR TotalPicks = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[OperationType] = "Picking"
    )
RETURN
DIVIDE(WarehouseLaborCost + FleetIdleCost, TotalPicks, 0)

// Measure: Predictive Bottleneck Index (simple trend-based)
BottleneckIndex = 
VAR CurrentAvgDwell = [AvgDwellTime]
VAR PriorAvgDwell = 
    CALCULATE(
        [AvgDwellTime],
        DATEADD(DimTime[FullDateTime], -7, DAY)
    )
VAR DwellTrend = DIVIDE(CurrentAvgDwell - PriorAvgDwell, PriorAvgDwell, 0)
VAR CurrentFleetUtil = [FleetUtilizationRate]
VAR PriorFleetUtil = 
    CALCULATE(
        [FleetUtilizationRate],
        DATEADD(DimTime[FullDateTime], -7, DAY)
    )
VAR FleetUtilTrend = DIVIDE(PriorFleetUtil - CurrentFleetUtil, PriorFleetUtil, 0)
RETURN
(DwellTrend * 50) + (FleetUtilTrend * 50) // Normalized 0-100 scale
```

## Configuration

### Environment Variables

Set these in your deployment environment (Azure, on-prem server, etc.):

```bash
# SQL Server connection
export LOGIFLEET_SQL_SERVER="your-server.database.windows.net"
export LOGIFLEET_SQL_DB="LogiFleetPulse"
export LOGIFLEET_SQL_USER="logifleet_app"
export LOGIFLEET_SQL_PASSWORD="your-secure-password"

# External API endpoints (optional integrations)
export WEATHER_API_KEY="your-weather-api-key"
export TRAFFIC_API_ENDPOINT="https://api.traffic-provider.com/v1"

# Power BI Service (for automated refresh)
export POWERBI_WORKSPACE_ID="your-workspace-guid"
export POWERBI_DATASET_ID="your-dataset-guid"
export POWERBI_CLIENT_ID="your-azure-app-id"
export POWERBI_CLIENT_SECRET="your-azure-app-secret"
```

### Config File Example (config.json)

```json
{
  "database": {
    "server": "${LOGIFLEET_SQL_SERVER}",
    "database": "${LOGIFLEET_SQL_DB}",
    "authentication": "sql",
    "connectionTimeout": 30,
    "requestTimeout": 60
  },
  "etl": {
    "incrementalLoadInterval": "15min",
    "historicalRetentionDays": 730,
    "enablePartitioning": true
  },
  "powerbi": {
    "refreshSchedule": "*/15 * * * *",
    "incrementalRefreshDays": 7,
    "fullRefreshOnWeekend": true
  },
  "alerts": {
    "dwellTimeThresholdHours": 72,
    "fleetUtilizationMinPercent": 70,
    "maintenancePriorityScoreThreshold": 50
  }
}
```

## Common Patterns

### Pattern 1: Query Cross-Fact KPIs

```sql
-- Find SKUs with high dwell time that also cause fleet delays
SELECT TOP 20
    p.SKU,
    p.ProductName,
    p.GravityZone,
    AVG(wo.DwellTimeHours) AS AvgDwellHours,
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdleMin,
    COUNT(DISTINCT ft.TripKey) AS AffectedTrips
FROM FactWarehouseOperations wo
INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
INNER JOIN BridgeTripOperations bto ON wo.OperationKey = bto.OperationKey
INNER JOIN FactFleetTrips ft ON bto.TripKey = ft.TripKey
WHERE wo.DwellTimeHours > 72
    AND ft.DelayReasonCode IS NOT NULL
GROUP BY p.SKU, p.ProductName, p.GravityZone
ORDER BY AVG(wo.DwellTimeHours) DESC;
```

### Pattern 2: Detect Misplaced High-Gravity Items

```sql
-- Find high-gravity products stored in low-proximity zones
SELECT 
    p.SKU,
    p.GravityZone,
    wo.StorageZone,
    COUNT(*) AS OperationCount,
    AVG(wo.DurationMinutes) AS AvgPickTimeMin
FROM FactWarehouseOperations wo
INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
WHERE p.GravityZone = 'High'
    AND wo.StorageZone NOT LIKE '%A%' -- Assume A zones are near dock
    AND wo.OperationType = 'Picking'
GROUP BY p.SKU, p.GravityZone, wo.StorageZone
HAVING AVG(wo.DurationMinutes) > 5 -- Longer than optimal
ORDER BY AVG(wo.DurationMinutes) DESC;
```

### Pattern 3: Fleet Maintenance Priority Report

```sql
-- Generate prioritized maintenance list
SELECT 
    VehicleID,
    TirePressureAlerts,
    EngineAlerts,
    HighValueTrips,
    MaintenancePriorityScore,
    CASE 
        WHEN MaintenancePriorityScore > 100 THEN 'Critical'
        WHEN MaintenancePriorityScore > 50 THEN 'High'
        WHEN MaintenancePriorityScore > 20 THEN 'Medium'
        ELSE 'Low'
    END AS PriorityLevel
FROM vw_FleetMaintenancePriority
ORDER BY MaintenancePriorityScore DESC;
```

### Pattern 4: Temporal Elasticity Simulation (What-If)

```sql
-- Simulate impact of increasing warehouse capacity from 80% to 95%
DECLARE @CurrentCapacityPct DECIMAL(5,2) = 80.0;
DECLARE @NewCapacityPct DECIMAL(5,2) = 95.0;

WITH CurrentMetrics AS (
    SELECT 
        AVG(DwellTimeHours) AS AvgDwell,
        COUNT(*) AS TotalOps
    FROM FactWarehouseOperations
    WHERE CreatedAt >= DATEADD(MONTH, -1, GETDATE())
)
SELECT 
    AvgDwell AS CurrentAvgDwell,
    -- Simple linear extrapolation (replace with ML model in production)
    AvgDwell * (@NewCapacityPct / @CurrentCapacityPct) AS ProjectedAvgDwell,
    TotalOps,
    TotalOps * (@NewCapacityPct / @CurrentCapacityPct) AS ProjectedTotalOps
FROM CurrentMetrics;
```

### Pattern 5: Automated Alert via Stored Procedure

```sql
-- Stored procedure to check thresholds and send alerts
CREATE PROCEDURE sp_CheckAlerts
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    -- Check dwell time threshold
    IF EXISTS (
        SELECT 1 
        FROM FactWarehouseOperations wo
        INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
        WHERE wo.DwellTimeHours > 72
            AND p.GravityZone = 'High'
            AND wo.CreatedAt >= DATEADD(HOUR, -1, GETDATE())
    )
    BEGIN
        SET @AlertMessage = 'ALERT: High-gravity SKUs with dwell time > 72 hours detected';
        -- Insert into alert log table or trigger external notification
        INSERT INTO AlertLog (AlertType, Message, CreatedAt)
        VALUES ('DwellTime', @AlertMessage, GETDATE());
    END;
    
    -- Check fleet utilization
    DECLARE @AvgUtilization DECIMAL(5,2);
    SELECT @AvgUtilization = AVG(
        CAST(DurationMinutes - IdleTimeMinutes AS DECIMAL) / 
        NULLIF(DurationMinutes, 0) * 100
    )
    FROM FactFleetTrips
    WHERE CreatedAt >= DATEADD(HOUR, -1, GETDATE());
    
    IF @AvgUtilization < 70
    BEGIN
        SET @AlertMessage = 'ALERT: Fleet utilization dropped below 70% (' + 
                            CAST(@AvgUtilization AS VARCHAR) + '%)';
        INSERT INTO AlertLog (AlertType, Message, CreatedAt)
        VALUES ('FleetUtilization', @AlertMessage, GETDATE());
    END;
END;
GO

-- Schedule this procedure to run every 15 minutes via SQL Server Agent
```

## Troubleshooting

### Issue: Power BI Refresh Timeout

**Symptoms:** Dataset refresh fails with "timeout expired" error

**Solutions:**
1. Enable incremental refresh for large fact tables
2. Create indexed views for complex cross-fact queries
3. Partition fact tables by month/quarter
4. Use Direct Query mode for real-time dashboards instead of Import

```sql
-- Create partitioning scheme for FactWarehouseOperations
CREATE PARTITION FUNCTION pf_MonthlyOperations (DATETIME)
AS RANGE RIGHT FOR VALUES (
    '2024-01-01', '2024-02-01', '2024-03-01', '2024-04-01',
    '2024-05-01', '2024-06-01', '2024-07-01', '2024-08-01',
    '2024-09-01', '2024-10-01', '2024-11-01', '2024-12-01',
    '2025-01-01', '2025-02-01', '2025-03-01', '2025-04-01',
    '2025-05-01', '2025-06-01', '2025-07-01', '2025-08-01',
    '2025-09-01', '2025-10-01', '2025-11-11', '2025-12-01',
    '2026-01-01', '2026-02-01', '2026-03-01', '2026-04-01',
    '2026-05-01', '2026-06-01', '2026-07-01', '2026-08-01'
);
```

### Issue: Slow Cross-Fact Queries

**Symptoms:** Queries joining FactWarehouseOperations and FactFleetTrips take > 30 seconds

**Solutions:**
1. Ensure indexes on bridge table (BridgeTripOperations)
2. Create filtered indexes for common query patterns
3. Use columnstore indexes for analytical queries

```sql
-- Create columnstore index for analytical workloads
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactWarehouse_Columnstore
ON FactWarehouseOperations (
    TimeKey, GeographyKey, ProductKey, OperationType,
    Quantity, DurationMinutes, DwellTimeHours
);

-- Create filtered index for high-dwell queries
CREATE INDEX IX_FactWarehouse_HighDwell
ON FactWarehouseOperations (DwellTimeHours, ProductKey, TimeKey)
WHERE DwellTimeHours > 48;
```

### Issue: Gravity Zone Calculations Not Updating

**Symptoms:** Products remain in old gravity zones despite changed pick frequency

**Solutions:**
1. Recalculate gravity scores via stored procedure
2. Update ValidFrom/ValidTo dates for slowly changing dimension

```sql
-- Stored procedure to recalculate gravity scores
CREATE PROCEDURE sp_RecalculateGravityZones
AS
BEGIN
    -- Calculate pick frequency from last 30 days
    UPDATE p
    SET 
        p.AvgPickFrequency = picks.AvgFrequency,
        p.ValidFrom = GETDATE()
    FROM DimProduct p
    INNER JOIN (
        SELECT 
            ProductKey,
            AVG(DailyPicks) AS AvgFrequency
        FROM (
            SELECT 
                ProductKey,
                CAST(TimeKey AS DATE) AS PickDate,
                COUNT(*) AS DailyPicks
            FROM FactWarehouseOperations
            WHERE OperationType = 'Picking'
                AND CreatedAt >= DATEADD(DAY, -30, GETDATE())
            GROUP BY ProductKey, CAST(TimeKey AS DATE)
        ) daily
        GROUP BY ProductKey
    ) picks ON p.ProductKey = picks.ProductKey;
    
    PRINT 'Gravity zones recalculated for ' + CAST(@@ROWCOUNT AS VARCHAR) + ' products';
END;
GO
```

### Issue: Missing Data in Power BI Dashboards

**Symptoms:** Recent operations not showing in dashboards

**Solutions:**
1. Verify incremental load procedure ran successfully
2. Check DimTime table has entries for current time buckets
3. Validate foreign key relationships in fact tables

```sql
-- Check for orphaned fact records (missing dimension keys)
SELECT 'Missing Time Keys' AS Issue, COUNT(*) AS Count
FROM FactWarehouseOperations wo
WHERE NOT EXISTS (SELECT 1 FROM DimTime WHERE TimeKey = wo.TimeKey)
UNION ALL
SELECT 'Missing Geography Keys', COUNT(*)
FROM FactWarehouseOperations wo
WHERE NOT EXISTS (SELECT 1 FROM DimGeography WHERE GeographyKey
