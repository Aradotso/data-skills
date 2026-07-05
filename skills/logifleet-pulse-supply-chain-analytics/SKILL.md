---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehouse for logistics, fleet, and supply chain intelligence with multi-fact star schema
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure Power BI logistics dashboard"
  - "implement warehouse and fleet data model"
  - "create multi-fact star schema for logistics"
  - "build supply chain intelligence warehouse"
  - "deploy LogiCore analytics platform"
  - "integrate fleet telemetry with warehouse data"
  - "optimize warehouse gravity zones"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive logistics intelligence platform built on MS SQL Server and Power BI that unifies:

- **Warehouse operations** (receiving, putaway, picking, packing, shipping)
- **Fleet telemetry** (GPS, fuel consumption, driver behavior, maintenance)
- **Inventory management** (aging curves, SKU velocity, gravity zones)
- **External data** (weather, traffic, supplier reliability)

It uses a **multi-fact star schema** with time-phased dimensions to enable cross-fact KPI queries like "show dwell time per SKU vs. fleet idling cost per route."

Key capabilities:
- Cross-fact KPI harmonization across warehouse and fleet
- Warehouse Gravity Zones™ for optimal storage placement
- Predictive bottleneck detection using ML
- Temporal elasticity modeling for scenario analysis
- 15-minute granularity real-time dashboards

## Installation & Deployment

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS)
- Access to data sources (WMS, TMS, telemetry APIs)

### Step 1: Deploy SQL Schema

```sql
-- Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create time dimension with 15-minute granularity
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    TimeOf15Min TIME,
    HourOfDay INT,
    DayOfWeek VARCHAR(10),
    WeekOfYear INT,
    MonthName VARCHAR(10),
    Quarter INT,
    FiscalYear INT,
    IsWeekend BIT,
    IsHoliday BIT
);

-- Create partitioning function for time-based tables
CREATE PARTITION FUNCTION PF_LogiFleet_Time (DATETIME)
AS RANGE RIGHT FOR VALUES (
    '2025-01-01', '2025-04-01', '2025-07-01', '2025-10-01',
    '2026-01-01', '2026-04-01', '2026-07-01', '2026-10-01'
);

CREATE PARTITION SCHEME PS_LogiFleet_Time
AS PARTITION PF_LogiFleet_Time ALL TO ([PRIMARY]);
```

### Step 2: Create Core Dimension Tables

```sql
-- Geography dimension (hierarchical)
CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    GeographyID VARCHAR(50) UNIQUE NOT NULL,
    LocationName VARCHAR(100),
    LocationType VARCHAR(20), -- 'Warehouse', 'Route Node', 'Hub'
    Region VARCHAR(50),
    Country VARCHAR(50),
    Continent VARCHAR(50),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    TimeZone VARCHAR(50),
    ValidFrom DATETIME DEFAULT GETDATE(),
    ValidTo DATETIME DEFAULT '9999-12-31',
    IsCurrent BIT DEFAULT 1
);

-- Product dimension with gravity score
CREATE TABLE DimProduct (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName VARCHAR(200),
    Category VARCHAR(50),
    Subcategory VARCHAR(50),
    IsPerishable BIT,
    RequiresColdStorage BIT,
    Weight DECIMAL(10,2),
    Volume DECIMAL(10,2),
    UnitCost DECIMAL(10,2),
    GravityScore DECIMAL(5,2), -- Composite score: velocity + value + fragility
    GravityZone VARCHAR(10), -- 'High', 'Medium', 'Low'
    LastRecalculatedDate DATETIME
);

-- Supplier reliability dimension
CREATE TABLE DimSupplier (
    SupplierKey INT IDENTITY(1,1) PRIMARY KEY,
    SupplierID VARCHAR(50) UNIQUE NOT NULL,
    SupplierName VARCHAR(200),
    Country VARCHAR(50),
    LeadTimeAvg INT, -- Days
    LeadTimeVariance DECIMAL(5,2),
    DefectRate DECIMAL(5,4),
    OnTimeDeliveryRate DECIMAL(5,4),
    ComplianceScore DECIMAL(5,2),
    ReliabilityTier VARCHAR(20) -- 'Platinum', 'Gold', 'Silver', 'Bronze'
);

-- Fleet vehicle dimension
CREATE TABLE DimVehicle (
    VehicleKey INT IDENTITY(1,1) PRIMARY KEY,
    VehicleID VARCHAR(50) UNIQUE NOT NULL,
    VehicleType VARCHAR(50), -- 'Box Truck', 'Refrigerated', 'Flatbed'
    Capacity DECIMAL(10,2),
    FuelType VARCHAR(20),
    YearManufactured INT,
    LastMaintenanceDate DATETIME,
    NextScheduledMaintenance DATETIME,
    TelemetryEnabled BIT
);
```

### Step 3: Create Fact Tables

```sql
-- Fact: Warehouse Operations
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(20), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    Quantity INT,
    DwellTimeMinutes INT,
    ProcessingTimeMinutes INT,
    OperatorID VARCHAR(50),
    ZoneID VARCHAR(20),
    BatchNumber VARCHAR(50),
    CONSTRAINT FK_WarehouseOp_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WarehouseOp_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_WarehouseOp_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
) ON PS_LogiFleet_Time(TimeKey);

-- Create columnstore index for analytics
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactWarehouseOp_CS
ON FactWarehouseOperations (TimeKey, GeographyKey, ProductKey, Quantity, DwellTimeMinutes, ProcessingTimeMinutes);

-- Fact: Fleet Trips
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    StartTimeKey INT NOT NULL,
    EndTimeKey INT,
    VehicleKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT,
    DriverID VARCHAR(50),
    DistanceKm DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    AverageSpeed DECIMAL(5,2),
    MaxSpeed DECIMAL(5,2),
    HarshBrakingEvents INT,
    WeatherCondition VARCHAR(50),
    TrafficDelayMinutes INT,
    CONSTRAINT FK_FleetTrip_StartTime FOREIGN KEY (StartTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FleetTrip_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_FleetTrip_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey)
) ON PS_LogiFleet_Time(StartTimeKey);

CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactFleetTrips_CS
ON FactFleetTrips (StartTimeKey, VehicleKey, DistanceKm, FuelConsumedLiters, IdleTimeMinutes);

-- Fact: Cross-Dock Operations
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    Quantity INT,
    TransferTimeMinutes INT,
    CONSTRAINT FK_CrossDock_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_CrossDock_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_CrossDock_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);
```

### Step 4: Create Bridge Tables for Many-to-Many Relationships

```sql
-- Bridge: Trip to Products (one trip can carry multiple products)
CREATE TABLE BridgeTripProduct (
    TripKey BIGINT NOT NULL,
    ProductKey INT NOT NULL,
    Quantity INT,
    LoadSequence INT,
    PRIMARY KEY (TripKey, ProductKey),
    CONSTRAINT FK_Bridge_Trip FOREIGN KEY (TripKey) REFERENCES FactFleetTrips(TripKey),
    CONSTRAINT FK_Bridge_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);

-- Bridge: Warehouse Zone to Products (dynamic gravity assignments)
CREATE TABLE BridgeZoneProduct (
    ZoneID VARCHAR(20) NOT NULL,
    ProductKey INT NOT NULL,
    AssignedDate DATETIME DEFAULT GETDATE(),
    PickFrequency INT,
    PRIMARY KEY (ZoneID, ProductKey),
    CONSTRAINT FK_BridgeZone_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);
```

## Configuration

### Environment Variables

Create a configuration file or use environment variables:

```bash
# Database connection
export LOGIFLEET_DB_SERVER="your-sql-server.database.windows.net"
export LOGIFLEET_DB_NAME="LogiFleetPulse"
export LOGIFLEET_DB_USER="logifleet_app"
export LOGIFLEET_DB_PASSWORD="${DB_PASSWORD}"

# External API endpoints
export LOGIFLEET_WEATHER_API_URL="https://api.weather.service/v1"
export LOGIFLEET_WEATHER_API_KEY="${WEATHER_API_KEY}"
export LOGIFLEET_TRAFFIC_API_URL="https://api.traffic.service/v2"
export LOGIFLEET_TRAFFIC_API_KEY="${TRAFFIC_API_KEY}"

# Data refresh intervals (minutes)
export LOGIFLEET_WAREHOUSE_REFRESH=15
export LOGIFLEET_FLEET_REFRESH=15
export LOGIFLEET_EXTERNAL_REFRESH=60
```

### Power BI Connection String

In Power BI Desktop, use this connection pattern:

```powerquery
// M Query for SQL Server connection
let
    Server = Environment.GetEnvironmentVariable("LOGIFLEET_DB_SERVER"),
    Database = Environment.GetEnvironmentVariable("LOGIFLEET_DB_NAME"),
    Source = Sql.Database(Server, Database, [
        Query = "
            SELECT * FROM FactWarehouseOperations 
            WHERE TimeKey >= (SELECT MAX(TimeKey) - 672 FROM DimTime)
        "
    ])
in
    Source
```

## Key Stored Procedures

### ETL: Populate Time Dimension

```sql
CREATE PROCEDURE sp_PopulateDimTime
    @StartDate DATETIME,
    @EndDate DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentDateTime DATETIME = @StartDate;
    DECLARE @TimeKey INT;
    
    WHILE @CurrentDateTime <= @EndDate
    BEGIN
        SET @TimeKey = CAST(CONVERT(VARCHAR(8), @CurrentDateTime, 112) AS INT) * 10000 +
                       DATEPART(HOUR, @CurrentDateTime) * 100 +
                       (DATEPART(MINUTE, @CurrentDateTime) / 15) * 15;
        
        INSERT INTO DimTime (TimeKey, FullDateTime, TimeOf15Min, HourOfDay, DayOfWeek, 
                             WeekOfYear, MonthName, Quarter, FiscalYear, IsWeekend, IsHoliday)
        VALUES (
            @TimeKey,
            @CurrentDateTime,
            CAST(@CurrentDateTime AS TIME),
            DATEPART(HOUR, @CurrentDateTime),
            DATENAME(WEEKDAY, @CurrentDateTime),
            DATEPART(WEEK, @CurrentDateTime),
            DATENAME(MONTH, @CurrentDateTime),
            DATEPART(QUARTER, @CurrentDateTime),
            CASE 
                WHEN MONTH(@CurrentDateTime) >= 4 THEN YEAR(@CurrentDateTime)
                ELSE YEAR(@CurrentDateTime) - 1
            END,
            CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1,7) THEN 1 ELSE 0 END,
            0 -- Holiday detection logic goes here
        );
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END;
GO

-- Execute to populate 2 years
EXEC sp_PopulateDimTime '2025-01-01', '2026-12-31';
```

### Calculate Product Gravity Scores

```sql
CREATE PROCEDURE sp_RecalculateProductGravity
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Calculate gravity score based on velocity, value, and fragility
    WITH ProductMetrics AS (
        SELECT 
            p.ProductKey,
            p.SKU,
            -- Velocity score (0-100): picks per day normalized
            (COUNT(wo.OperationKey) * 1.0 / NULLIF(DATEDIFF(DAY, MIN(t.FullDateTime), MAX(t.FullDateTime)), 0)) * 10 AS VelocityScore,
            -- Value score (0-100): unit cost normalized
            (p.UnitCost / (SELECT MAX(UnitCost) FROM DimProduct)) * 100 AS ValueScore,
            -- Fragility score (0-50): perishable + cold storage
            (CASE WHEN p.IsPerishable = 1 THEN 25 ELSE 0 END + 
             CASE WHEN p.RequiresColdStorage = 1 THEN 25 ELSE 0 END) AS FragilityScore
        FROM DimProduct p
        LEFT JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
        LEFT JOIN DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.FullDateTime >= DATEADD(DAY, -90, GETDATE())
          AND wo.OperationType = 'Picking'
        GROUP BY p.ProductKey, p.SKU, p.UnitCost, p.IsPerishable, p.RequiresColdStorage
    )
    UPDATE p
    SET 
        p.GravityScore = pm.VelocityScore + pm.ValueScore + pm.FragilityScore,
        p.GravityZone = CASE 
            WHEN (pm.VelocityScore + pm.ValueScore + pm.FragilityScore) >= 150 THEN 'High'
            WHEN (pm.VelocityScore + pm.ValueScore + pm.FragilityScore) >= 75 THEN 'Medium'
            ELSE 'Low'
        END,
        p.LastRecalculatedDate = GETDATE()
    FROM DimProduct p
    INNER JOIN ProductMetrics pm ON p.ProductKey = pm.ProductKey;
    
    PRINT 'Product gravity scores recalculated: ' + CAST(@@ROWCOUNT AS VARCHAR(10));
END;
GO
```

### Cross-Fact KPI Query: Dwell Time vs. Fleet Idling

```sql
CREATE PROCEDURE sp_GetDwellTimeVsFleetIdling
    @StartDate DATETIME,
    @EndDate DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Join warehouse dwell time with fleet idling for same products/time periods
    SELECT 
        t.MonthName,
        t.FiscalYear,
        p.Category,
        g.Region,
        -- Warehouse metrics
        AVG(wo.DwellTimeMinutes) AS AvgDwellTimeMinutes,
        SUM(wo.Quantity) AS TotalWarehouseVolume,
        -- Fleet metrics
        AVG(ft.IdleTimeMinutes) AS AvgFleetIdleTimeMinutes,
        SUM(ft.FuelConsumedLiters) AS TotalFuelConsumed,
        -- Cross-fact KPI: Cost per unit
        (SUM(ft.FuelConsumedLiters) * 1.5 / NULLIF(SUM(wo.Quantity), 0)) AS FuelCostPerUnit,
        -- Efficiency index
        (AVG(wo.DwellTimeMinutes) * AVG(ft.IdleTimeMinutes)) AS EfficiencyPenaltyIndex
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
    INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
    INNER JOIN BridgeTripProduct btp ON p.ProductKey = btp.ProductKey
    INNER JOIN FactFleetTrips ft ON btp.TripKey = ft.TripKey
    WHERE t.FullDateTime BETWEEN @StartDate AND @EndDate
    GROUP BY t.MonthName, t.FiscalYear, p.Category, g.Region
    ORDER BY EfficiencyPenaltyIndex DESC;
END;
GO
```

## Power BI Integration

### Create the Semantic Model

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Import the core tables:

```powerquery
// Import Fact Warehouse Operations
let
    Source = Sql.Database(ServerName, DatabaseName),
    dbo_FactWarehouseOperations = Source{[Schema="dbo",Item="FactWarehouseOperations"]}[Data]
in
    dbo_FactWarehouseOperations

// Import Fact Fleet Trips
let
    Source = Sql.Database(ServerName, DatabaseName),
    dbo_FactFleetTrips = Source{[Schema="dbo",Item="FactFleetTrips"]}[Data]
in
    dbo_FactFleetTrips
```

### Create DAX Measures

```dax
-- Total Dwell Time
TotalDwellTime = SUM(FactWarehouseOperations[DwellTimeMinutes])

-- Average Fleet Idle Percentage
AvgFleetIdlePct = 
DIVIDE(
    AVERAGE(FactFleetTrips[IdleTimeMinutes]),
    AVERAGE(FactFleetTrips[IdleTimeMinutes]) + 
    AVERAGE(FactFleetTrips[LoadingTimeMinutes]) + 
    AVERAGE(FactFleetTrips[UnloadingTimeMinutes]) +
    (FactFleetTrips[DistanceKm] / FactFleetTrips[AverageSpeed] * 60),
    0
) * 100

-- Gravity Zone Compliance
GravityZoneCompliance = 
VAR HighGravityProducts = 
    CALCULATE(
        COUNTROWS(DimProduct),
        DimProduct[GravityZone] = "High"
    )
VAR HighGravityInOptimalZones = 
    CALCULATE(
        COUNTROWS(BridgeZoneProduct),
        DimProduct[GravityZone] = "High",
        LEFT(BridgeZoneProduct[ZoneID], 1) = "A"  -- Assuming A zones are near shipping
    )
RETURN
    DIVIDE(HighGravityInOptimalZones, HighGravityProducts, 0) * 100

-- Predictive Bottleneck Index (simplified)
BottleneckIndex = 
VAR DwellThreshold = 120  -- minutes
VAR IdleThreshold = 20    -- percent
VAR HighDwellCount = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[DwellTimeMinutes] > DwellThreshold
    )
VAR HighIdleCount = 
    CALCULATE(
        COUNTROWS(FactFleetTrips),
        [AvgFleetIdlePct] > IdleThreshold
    )
RETURN
    HighDwellCount + HighIdleCount

-- Fuel Efficiency per Product Category
FuelPerCategory = 
CALCULATE(
    DIVIDE(
        SUM(FactFleetTrips[FuelConsumedLiters]),
        SUM(BridgeTripProduct[Quantity])
    ),
    USERELATIONSHIP(BridgeTripProduct[ProductKey], DimProduct[ProductKey])
)
```

### Create Relationships in Model View

```
DimTime (TimeKey) 1:* FactWarehouseOperations (TimeKey)
DimTime (TimeKey) 1:* FactFleetTrips (StartTimeKey)
DimProduct (ProductKey) 1:* FactWarehouseOperations (ProductKey)
DimProduct (ProductKey) 1:* BridgeTripProduct (ProductKey)
DimGeography (GeographyKey) 1:* FactWarehouseOperations (GeographyKey)
DimGeography (GeographyKey) 1:* FactFleetTrips (OriginGeographyKey)
DimVehicle (VehicleKey) 1:* FactFleetTrips (VehicleKey)
FactFleetTrips (TripKey) 1:* BridgeTripProduct (TripKey)
```

Set cross-filter direction to **Both** for bridge tables.

## Common Patterns & Use Cases

### Pattern 1: Identify Slow-Moving Inventory in High-Gravity Zones

```sql
-- Find products miscategorized in high-gravity zones
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityZone AS AssignedZone,
    bzp.ZoneID AS ActualZone,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
    COUNT(wo.OperationKey) AS PicksLast30Days
FROM DimProduct p
INNER JOIN BridgeZoneProduct bzp ON p.ProductKey = bzp.ProductKey
LEFT JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
    AND wo.OperationType = 'Picking'
    AND wo.TimeKey >= (SELECT MAX(TimeKey) - 288 FROM DimTime)  -- Last 30 days at 15min intervals
WHERE p.GravityZone = 'High' 
  AND LEFT(bzp.ZoneID, 1) = 'A'
GROUP BY p.SKU, p.ProductName, p.GravityZone, bzp.ZoneID
HAVING COUNT(wo.OperationKey) < 10  -- Less than 10 picks in 30 days
ORDER BY AvgDwellTime DESC;
```

### Pattern 2: Route Optimization Based on Load Composition

```sql
-- Analyze fuel efficiency by product mix on routes
WITH RouteMix AS (
    SELECT 
        ft.TripKey,
        ft.DistanceKm,
        ft.FuelConsumedLiters,
        SUM(CASE WHEN p.RequiresColdStorage = 1 THEN btp.Quantity ELSE 0 END) AS ColdStorageUnits,
        SUM(CASE WHEN p.IsPerishable = 1 THEN btp.Quantity ELSE 0 END) AS PerishableUnits,
        SUM(btp.Quantity) AS TotalUnits
    FROM FactFleetTrips ft
    INNER JOIN BridgeTripProduct btp ON ft.TripKey = btp.TripKey
    INNER JOIN DimProduct p ON btp.ProductKey = p.ProductKey
    GROUP BY ft.TripKey, ft.DistanceKm, ft.FuelConsumedLiters
)
SELECT 
    CASE 
        WHEN ColdStorageUnits * 1.0 / TotalUnits > 0.5 THEN 'Majority Cold'
        WHEN PerishableUnits * 1.0 / TotalUnits > 0.5 THEN 'Majority Perishable'
        ELSE 'Mixed'
    END AS LoadType,
    AVG(FuelConsumedLiters / NULLIF(DistanceKm, 0)) AS AvgFuelPerKm,
    AVG(DistanceKm / NULLIF(TotalUnits, 0)) AS AvgKmPerUnit
FROM RouteMix
GROUP BY 
    CASE 
        WHEN ColdStorageUnits * 1.0 / TotalUnits > 0.5 THEN 'Majority Cold'
        WHEN PerishableUnits * 1.0 / TotalUnits > 0.5 THEN 'Majority Perishable'
        ELSE 'Mixed'
    END;
```

### Pattern 3: Temporal Elasticity Simulation

```sql
-- Simulate impact of 95% warehouse utilization
CREATE PROCEDURE sp_SimulateWarehouseCapacity
    @TargetUtilization DECIMAL(5,2)
AS
BEGIN
    -- Baseline: Current average dwell time
    DECLARE @CurrentAvgDwell DECIMAL(10,2);
    SELECT @CurrentAvgDwell = AVG(DwellTimeMinutes)
    FROM FactWarehouseOperations
    WHERE TimeKey >= (SELECT MAX(TimeKey) - 672 FROM DimTime);
    
    -- Projected dwell time increase (non-linear model)
    DECLARE @ProjectedAvgDwell DECIMAL(10,2) = 
        @CurrentAvgDwell * POWER((@TargetUtilization / 80.0), 2);  -- Assumes current is 80%
    
    -- Impact on fleet (longer loading times)
    DECLARE @ProjectedLoadingTime DECIMAL(10,2);
    SELECT @ProjectedLoadingTime = AVG(LoadingTimeMinutes) * 
        (@ProjectedAvgDwell / @CurrentAvgDwell)
    FROM FactFleetTrips
    WHERE StartTimeKey >= (SELECT MAX(TimeKey) - 672 FROM DimTime);
    
    SELECT 
        'Current' AS Scenario,
        @CurrentAvgDwell AS AvgDwellTime,
        AVG(LoadingTimeMinutes) AS AvgLoadingTime,
        AVG(IdleTimeMinutes) AS AvgIdleTime
    FROM FactFleetTrips
    WHERE StartTimeKey >= (SELECT MAX(TimeKey) - 672 FROM DimTime)
    
    UNION ALL
    
    SELECT 
        'Projected (' + CAST(@TargetUtilization AS VARCHAR(5)) + '% utilization)' AS Scenario,
        @ProjectedAvgDwell AS AvgDwellTime,
        @ProjectedLoadingTime AS AvgLoadingTime,
        AVG(IdleTimeMinutes) * 1.15 AS AvgIdleTime  -- 15% increase assumption
    FROM FactFleetTrips
    WHERE StartTimeKey >= (SELECT MAX(TimeKey) - 672 FROM DimTime);
END;
GO
```

## Automated Alerting

### Create Alert Stored Procedure

```sql
CREATE PROCEDURE sp_CheckThresholdsAndAlert
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertMessages TABLE (
        AlertType VARCHAR(50),
        Severity VARCHAR(20),
        Message VARCHAR(500)
    );
    
    -- Alert 1: High fleet idle time
    INSERT INTO @AlertMessages
    SELECT 
        'Fleet Idle Time' AS AlertType,
        'HIGH' AS Severity,
        'Vehicle ' + v.VehicleID + ' idle time: ' + 
        CAST(AVG(ft.IdleTimeMinutes) AS VARCHAR(10)) + ' min (threshold: 20)'
    FROM FactFleetTrips ft
    INNER JOIN DimVehicle v ON ft.VehicleKey = v.VehicleKey
    WHERE ft.StartTimeKey >= (SELECT MAX(TimeKey) - 96 FROM DimTime)  -- Last 24 hours
    GROUP BY v.VehicleID
    HAVING AVG(ft.IdleTimeMinutes) > 20;
    
    -- Alert 2: Excessive warehouse dwell time
    INSERT INTO @AlertMessages
    SELECT 
        'Warehouse Dwell' AS AlertType,
        'MEDIUM' AS Severity,
        'SKU ' + p.SKU + ' dwell time: ' + 
        CAST(AVG(wo.DwellTimeMinutes) AS VARCHAR(10)) + ' min (threshold: 120)'
    FROM FactWarehouseOperations wo
    INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
    WHERE wo.TimeKey >= (SELECT MAX(TimeKey) - 96 FROM DimTime)
    GROUP BY p.SKU
    HAVING AVG(wo.DwellTimeMinutes) > 120;
    
    -- Alert 3: Products in wrong gravity zone
    INSERT INTO @AlertMessages
    SELECT 
        'Zone Mismatch' AS AlertType,
        'LOW' AS Severity,
        'High-gravity SKU ' + p.SKU + ' found in low-priority zone ' + bzp.ZoneID
    FROM DimProduct p
    INNER JOIN BridgeZoneProduct bzp ON p.ProductKey = bzp.ProductKey
    WHERE p.GravityZone = 'High' 
      AND LEFT(bzp.ZoneID, 1) NOT IN ('A', 'B');
    
    -- Return alerts
    SELECT * FROM @AlertMessages ORDER BY 
        CASE Severity 
            WHEN 'HIGH' THEN 1
            WHEN 'MEDIUM' THEN 2
            ELSE 3
        END;
END;
GO

-- Schedule via SQL Server Agent
-- EXEC msdb.dbo.sp_add_job @job_name = 'LogiFleet_HourlyAlerts';
-- Configure job step to execute sp_CheckThresholdsAndAlert and email results
```

## Troubleshooting

### Issue: Slow Cross-Fact Queries

**Symptom**: Queries joining FactWarehouseOperations and FactFleetTrips take >10 seconds.

**Solution**:
```sql
-- Ensure columnstore indexes are present
IF NOT EXISTS (
    SELECT * FROM sys.indexes 
    WHERE name = 'IX_FactWarehouseOp_CS' 
    AND object_id = OBJECT_ID('FactWarehouseOperations')
)
BEGIN
    CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactWarehouseOp_CS
    ON FactWarehouseOperations (TimeKey, GeographyKey, ProductKey, Quantity, DwellTimeMinutes);
END;
