---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics intelligence platform with multi-fact star schema for warehouse, fleet, and supply chain analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "deploy the logistics data warehouse schema"
  - "configure Power BI for fleet and warehouse dashboards"
  - "implement cross-fact KPI harmonization for logistics"
  - "create warehouse gravity zones in SQL"
  - "build real-time fleet telemetry dashboard"
  - "set up adaptive supply chain analytics"
  - "integrate warehouse and fleet data models"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a **multi-modal logistics intelligence engine** that combines warehouse operations, fleet telemetry, inventory management, and external data sources into a unified semantic layer. Built on MS SQL Server and Power BI, it provides:

- **Multi-fact star schema** linking warehouse operations, fleet trips, and cross-dock activities
- **Time-phased dimensions** with 15-minute granularity for real-time analysis
- **Warehouse Gravity Zones™** for spatial optimization based on pick frequency and item value
- **Cross-fact KPI harmonization** connecting inventory metrics with fleet performance
- **Predictive bottleneck detection** using historical pattern analysis
- **Role-based Power BI dashboards** with row-level security

The platform is designed for 3PL operators, retail chains, food distributors, and any organization managing complex supply chain operations.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to data sources: WMS, TMS, telematics APIs, ERP systems

### Step 1: Deploy SQL Schema

```sql
-- Run the complete schema deployment script
-- This creates all fact tables, dimensions, views, and stored procedures

USE master;
GO

-- Create database
CREATE DATABASE LogiFleetPulse
ON PRIMARY 
( NAME = N'LogiFleetPulse_Data', 
    FILENAME = N'C:\SQLData\LogiFleetPulse_Data.mdf' , 
    SIZE = 1024MB , 
    FILEGROWTH = 512MB )
LOG ON 
( NAME = N'LogiFleetPulse_Log', 
    FILENAME = N'C:\SQLData\LogiFleetPulse_Log.ldf' , 
    SIZE = 512MB , 
    FILEGROWTH = 256MB );
GO

USE LogiFleetPulse;
GO

-- Create schemas for organization
CREATE SCHEMA DWH AUTHORIZATION dbo;
CREATE SCHEMA ETL AUTHORIZATION dbo;
CREATE SCHEMA SEC AUTHORIZATION dbo;
GO
```

### Step 2: Create Dimension Tables

```sql
-- DimTime: 15-minute granularity time dimension
CREATE TABLE DWH.DimTime (
    TimeKey INT IDENTITY(1,1) PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME NOT NULL,
    Hour TINYINT NOT NULL,
    Minute TINYINT NOT NULL,
    QuarterHour TINYINT NOT NULL, -- 0, 1, 2, 3 (15-min buckets)
    DayOfWeek TINYINT NOT NULL,
    DayName VARCHAR(10) NOT NULL,
    IsWeekend BIT NOT NULL,
    FiscalPeriod VARCHAR(10) NOT NULL,
    FiscalQuarter TINYINT NOT NULL,
    FiscalYear SMALLINT NOT NULL
);
CREATE UNIQUE INDEX UX_DimTime_DateTime ON DWH.DimTime(FullDateTime);
GO

-- DimGeography: Hierarchical location dimension
CREATE TABLE DWH.DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationID VARCHAR(50) NOT NULL UNIQUE,
    LocationName NVARCHAR(200) NOT NULL,
    LocationType VARCHAR(20) NOT NULL, -- Warehouse, RoutingNode, CustomerSite
    Address NVARCHAR(500),
    City NVARCHAR(100),
    Region NVARCHAR(100),
    Country NVARCHAR(100),
    Continent NVARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone VARCHAR(50),
    IsActive BIT NOT NULL DEFAULT 1,
    EffectiveDate DATE NOT NULL,
    ExpirationDate DATE NULL
);
CREATE INDEX IX_DimGeography_Type ON DWH.DimGeography(LocationType) WHERE IsActive = 1;
GO

-- DimProductGravity: Product classification with gravity scoring
CREATE TABLE DWH.DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU VARCHAR(100) NOT NULL UNIQUE,
    ProductName NVARCHAR(300) NOT NULL,
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    UnitValue DECIMAL(12,2),
    FragilityScore DECIMAL(3,2), -- 0.00 to 1.00
    VelocityClass VARCHAR(20), -- Fast, Medium, Slow
    GravityScore DECIMAL(5,2), -- Computed: velocity * value * (1/fragility)
    PreferredZoneType VARCHAR(50), -- HighGravity, MediumGravity, LowGravity, Bulk
    IsPerishable BIT NOT NULL DEFAULT 0,
    ShelfLifeDays SMALLINT,
    LastUpdated DATETIME NOT NULL DEFAULT GETDATE()
);
CREATE INDEX IX_DimProduct_Gravity ON DWH.DimProductGravity(GravityScore DESC);
CREATE INDEX IX_DimProduct_Velocity ON DWH.DimProductGravity(VelocityClass);
GO

-- DimSupplierReliability: Supplier performance tracking
CREATE TABLE DWH.DimSupplierReliability (
    SupplierKey INT IDENTITY(1,1) PRIMARY KEY,
    SupplierID VARCHAR(50) NOT NULL UNIQUE,
    SupplierName NVARCHAR(200) NOT NULL,
    LeadTimeMean DECIMAL(8,2), -- Average days
    LeadTimeStdDev DECIMAL(8,2), -- Variance indicator
    DefectRate DECIMAL(5,4), -- Percentage as decimal
    ComplianceScore DECIMAL(5,2), -- 0-100
    ReliabilityTier VARCHAR(20), -- Platinum, Gold, Silver, Bronze
    LastEvaluationDate DATE
);
GO

-- DimWarehouseZone: Spatial zones with gravity attributes
CREATE TABLE DWH.DimWarehouseZone (
    ZoneKey INT IDENTITY(1,1) PRIMARY KEY,
    WarehouseGeographyKey INT FOREIGN KEY REFERENCES DWH.DimGeography(GeographyKey),
    ZoneID VARCHAR(50) NOT NULL,
    ZoneName NVARCHAR(100) NOT NULL,
    ZoneType VARCHAR(50), -- HighGravity, Receiving, Packing, Shipping, Bulk
    DistanceToShippingDock DECIMAL(8,2), -- Meters
    PickPathOptimizationScore DECIMAL(5,2),
    CapacityPallets INT,
    CapacityUtilization DECIMAL(5,2), -- Percentage
    CONSTRAINT UX_Zone UNIQUE (WarehouseGeographyKey, ZoneID)
);
GO
```

### Step 3: Create Fact Tables

```sql
-- FactWarehouseOperations: Granular warehouse activity tracking
CREATE TABLE DWH.FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DWH.DimTime(TimeKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DWH.DimProductGravity(ProductKey),
    ZoneKey INT NOT NULL FOREIGN KEY REFERENCES DWH.DimWarehouseZone(ZoneKey),
    SupplierKey INT FOREIGN KEY REFERENCES DWH.DimSupplierReliability(SupplierKey),
    
    OperationType VARCHAR(20) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    OrderID VARCHAR(50),
    BatchNumber VARCHAR(50),
    
    -- Metrics
    QuantityUnits INT NOT NULL,
    DwellTimeMinutes DECIMAL(10,2),
    CycleTimeMinutes DECIMAL(10,2),
    DistanceTraveled DECIMAL(10,2), -- Meters
    LaborHours DECIMAL(8,2),
    
    -- Flags
    IsDelayed BIT NOT NULL DEFAULT 0,
    DelayReasonCode VARCHAR(50),
    IsDamaged BIT NOT NULL DEFAULT 0,
    DamageValue DECIMAL(12,2),
    
    RecordInsertedUTC DATETIME NOT NULL DEFAULT GETUTCDATE()
);

-- Columnstore index for fast aggregation
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouse 
ON DWH.FactWarehouseOperations (TimeKey, ProductKey, ZoneKey, OperationType, QuantityUnits, DwellTimeMinutes);

-- Filtered index for recent operations
CREATE INDEX IX_FactWarehouse_Recent 
ON DWH.FactWarehouseOperations(TimeKey, OperationType) 
WHERE TimeKey >= DATEPART(YEAR, GETDATE()) * 10000;
GO

-- FactFleetTrips: Fleet and route performance
CREATE TABLE DWH.FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    StartTimeKey INT NOT NULL FOREIGN KEY REFERENCES DWH.DimTime(TimeKey),
    EndTimeKey INT NOT NULL FOREIGN KEY REFERENCES DWH.DimTime(TimeKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DWH.DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DWH.DimGeography(GeographyKey),
    
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50),
    TripID VARCHAR(100) NOT NULL,
    
    -- Route Metrics
    PlannedDistanceKM DECIMAL(10,2),
    ActualDistanceKM DECIMAL(10,2),
    PlannedDurationMinutes INT,
    ActualDurationMinutes INT,
    IdleTimeMinutes INT,
    LoadingUnloadingMinutes INT,
    
    -- Performance Metrics
    FuelConsumedLiters DECIMAL(10,2),
    FuelEfficiencyKMPerLiter DECIMAL(8,2),
    AverageSpeedKMH DECIMAL(6,2),
    HarshBrakingEvents INT,
    HarshAccelerationEvents INT,
    
    -- Cost & Revenue
    FuelCost DECIMAL(12,2),
    LaborCost DECIMAL(12,2),
    RevenuePotential DECIMAL(12,2),
    
    -- Status Flags
    IsDelayed BIT NOT NULL DEFAULT 0,
    DelayReasonCode VARCHAR(50),
    WeatherImpact BIT DEFAULT 0,
    TrafficImpact BIT DEFAULT 0,
    
    RecordInsertedUTC DATETIME NOT NULL DEFAULT GETUTCDATE()
);

CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactFleet 
ON DWH.FactFleetTrips (StartTimeKey, VehicleID, ActualDurationMinutes, FuelConsumedLiters, IdleTimeMinutes);
GO

-- FactCrossDock: Cross-dock transfer operations
CREATE TABLE DWH.FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    InboundTimeKey INT NOT NULL FOREIGN KEY REFERENCES DWH.DimTime(TimeKey),
    OutboundTimeKey INT NOT NULL FOREIGN KEY REFERENCES DWH.DimTime(TimeKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DWH.DimProductGravity(ProductKey),
    WarehouseGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DWH.DimGeography(GeographyKey),
    
    QuantityUnits INT NOT NULL,
    DwellMinutes DECIMAL(10,2),
    TransferDistance DECIMAL(8,2),
    
    IsPurePlay BIT NOT NULL DEFAULT 1, -- True if no storage, direct transfer
    IsDelayed BIT NOT NULL DEFAULT 0,
    
    RecordInsertedUTC DATETIME NOT NULL DEFAULT GETUTCDATE()
);
GO
```

### Step 4: Create Bridge Tables for Many-to-Many Relationships

```sql
-- Bridge table linking trips to multiple products/orders
CREATE TABLE DWH.BridgeTripProduct (
    TripKey BIGINT NOT NULL FOREIGN KEY REFERENCES DWH.FactFleetTrips(TripKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DWH.DimProductGravity(ProductKey),
    QuantityLoaded INT NOT NULL,
    LoadSequence TINYINT,
    CONSTRAINT PK_BridgeTripProduct PRIMARY KEY (TripKey, ProductKey)
);
GO

-- Bridge table linking zones to trips (multi-stop routes)
CREATE TABLE DWH.BridgeTripZone (
    TripKey BIGINT NOT NULL FOREIGN KEY REFERENCES DWH.FactFleetTrips(TripKey),
    ZoneKey INT NOT NULL FOREIGN KEY REFERENCES DWH.DimWarehouseZone(ZoneKey),
    StopSequence TINYINT NOT NULL,
    TimeAtZoneMinutes INT,
    CONSTRAINT PK_BridgeTripZone PRIMARY KEY (TripKey, ZoneKey, StopSequence)
);
GO
```

### Step 5: Create Key Views for Cross-Fact Analysis

```sql
-- Unified view linking warehouse dwell time with fleet performance
CREATE VIEW DWH.vw_WarehouseFleetImpact AS
SELECT 
    t.FullDateTime AS TripStartTime,
    g_origin.LocationName AS OriginWarehouse,
    g_dest.LocationName AS DestinationSite,
    p.SKU,
    p.ProductName,
    p.GravityScore,
    
    -- Warehouse metrics
    AVG(wo.DwellTimeMinutes) AS AvgWarehouseDwellMinutes,
    SUM(wo.QuantityUnits) AS TotalUnitsProcessed,
    
    -- Fleet metrics
    ft.ActualDurationMinutes,
    ft.IdleTimeMinutes,
    ft.FuelConsumedLiters,
    ft.IsDelayed AS FleetDelayed,
    
    -- Cross-fact KPI
    CASE 
        WHEN AVG(wo.DwellTimeMinutes) > 240 AND ft.IsDelayed = 1 
        THEN 1 ELSE 0 
    END AS HighDwellCausedDelay
    
FROM DWH.FactFleetTrips ft
INNER JOIN DWH.DimTime t ON ft.StartTimeKey = t.TimeKey
INNER JOIN DWH.DimGeography g_origin ON ft.OriginGeographyKey = g_origin.GeographyKey
INNER JOIN DWH.DimGeography g_dest ON ft.DestinationGeographyKey = g_dest.GeographyKey
INNER JOIN DWH.BridgeTripProduct btp ON ft.TripKey = btp.TripKey
INNER JOIN DWH.DimProductGravity p ON btp.ProductKey = p.ProductKey
LEFT JOIN DWH.FactWarehouseOperations wo 
    ON wo.ProductKey = p.ProductKey 
    AND wo.TimeKey BETWEEN DATEADD(HOUR, -24, t.TimeKey) AND t.TimeKey
WHERE g_origin.LocationType = 'Warehouse'
GROUP BY 
    t.FullDateTime, g_origin.LocationName, g_dest.LocationName,
    p.SKU, p.ProductName, p.GravityScore,
    ft.ActualDurationMinutes, ft.IdleTimeMinutes, 
    ft.FuelConsumedLiters, ft.IsDelayed;
GO

-- Gravity zone optimization view
CREATE VIEW DWH.vw_GravityZonePerformance AS
SELECT 
    wz.ZoneName,
    wz.ZoneType,
    wz.DistanceToShippingDock,
    AVG(p.GravityScore) AS AvgProductGravity,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
    AVG(wo.CycleTimeMinutes) AS AvgCycleTime,
    COUNT(DISTINCT wo.ProductKey) AS UniqueProducts,
    SUM(wo.QuantityUnits) AS TotalUnitsProcessed,
    
    -- Optimization flag
    CASE 
        WHEN AVG(p.GravityScore) > 50 AND wz.DistanceToShippingDock > 100 
        THEN 'RELOCATE_CLOSER'
        WHEN AVG(p.GravityScore) < 20 AND wz.DistanceToShippingDock < 50 
        THEN 'RELOCATE_FURTHER'
        ELSE 'OPTIMAL'
    END AS OptimizationRecommendation
    
FROM DWH.DimWarehouseZone wz
INNER JOIN DWH.FactWarehouseOperations wo ON wz.ZoneKey = wo.ZoneKey
INNER JOIN DWH.DimProductGravity p ON wo.ProductKey = p.ProductKey
WHERE wo.TimeKey >= DATEADD(DAY, -30, GETDATE())
GROUP BY wz.ZoneName, wz.ZoneType, wz.DistanceToShippingDock;
GO
```

### Step 6: Create Stored Procedures for ETL and Alerts

```sql
-- Incremental load procedure for warehouse operations
CREATE PROCEDURE ETL.usp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- If no checkpoint, load last 24 hours
    IF @LastLoadDateTime IS NULL
        SET @LastLoadDateTime = DATEADD(HOUR, -24, GETUTCDATE());
    
    -- Insert from staging (assuming external table or linked server)
    INSERT INTO DWH.FactWarehouseOperations (
        TimeKey, ProductKey, ZoneKey, SupplierKey,
        OperationType, OrderID, BatchNumber,
        QuantityUnits, DwellTimeMinutes, CycleTimeMinutes,
        DistanceTraveled, LaborHours, IsDelayed, DelayReasonCode
    )
    SELECT 
        dt.TimeKey,
        dp.ProductKey,
        dz.ZoneKey,
        ds.SupplierKey,
        stg.OperationType,
        stg.OrderID,
        stg.BatchNumber,
        stg.Quantity,
        DATEDIFF(MINUTE, stg.StartTime, stg.EndTime) AS DwellTimeMinutes,
        stg.CycleTime,
        stg.Distance,
        stg.LaborHours,
        CASE WHEN stg.CycleTime > stg.StandardCycleTime * 1.2 THEN 1 ELSE 0 END,
        stg.DelayReason
    FROM StagingSchema.WarehouseOperationsStaging stg
    INNER JOIN DWH.DimTime dt ON stg.OperationDateTime = dt.FullDateTime
    INNER JOIN DWH.DimProductGravity dp ON stg.SKU = dp.SKU
    INNER JOIN DWH.DimWarehouseZone dz ON stg.ZoneID = dz.ZoneID
    LEFT JOIN DWH.DimSupplierReliability ds ON stg.SupplierID = ds.SupplierID
    WHERE stg.OperationDateTime > @LastLoadDateTime
        AND NOT EXISTS (
            SELECT 1 FROM DWH.FactWarehouseOperations wo
            WHERE wo.OrderID = stg.OrderID AND wo.OperationType = stg.OperationType
        );
    
    -- Update checkpoint
    UPDATE ETL.LoadCheckpoints 
    SET LastLoadDateTime = GETUTCDATE() 
    WHERE TableName = 'FactWarehouseOperations';
END;
GO

-- Alert stored procedure for proactive notifications
CREATE PROCEDURE ETL.usp_CheckLogisticsAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertMessages TABLE (
        AlertType VARCHAR(50),
        AlertMessage NVARCHAR(500),
        Severity VARCHAR(20),
        TriggeredAt DATETIME
    );
    
    -- Alert 1: High dwell time in high-gravity zones
    INSERT INTO @AlertMessages
    SELECT 
        'HIGH_DWELL_GRAVITY_ZONE',
        CONCAT('Zone ', wz.ZoneName, ' has avg dwell time of ', 
               CAST(AVG(wo.DwellTimeMinutes) AS INT), ' min for high-gravity products'),
        'WARNING',
        GETUTCDATE()
    FROM DWH.FactWarehouseOperations wo
    INNER JOIN DWH.DimWarehouseZone wz ON wo.ZoneKey = wz.ZoneKey
    INNER JOIN DWH.DimProductGravity p ON wo.ProductKey = p.ProductKey
    WHERE wo.TimeKey >= DATEADD(HOUR, -4, GETUTCDATE())
        AND p.GravityScore > 50
        AND wz.ZoneType = 'HighGravity'
    GROUP BY wz.ZoneName
    HAVING AVG(wo.DwellTimeMinutes) > 180;
    
    -- Alert 2: Fleet idling exceeds 15% of trip time
    INSERT INTO @AlertMessages
    SELECT 
        'EXCESSIVE_FLEET_IDLE',
        CONCAT('Vehicle ', ft.VehicleID, ' had ', 
               CAST((ft.IdleTimeMinutes * 100.0 / ft.ActualDurationMinutes) AS INT), 
               '% idle time on trip ', ft.TripID),
        'CRITICAL',
        GETUTCDATE()
    FROM DWH.FactFleetTrips ft
    WHERE ft.StartTimeKey >= DATEADD(HOUR, -12, GETUTCDATE())
        AND (ft.IdleTimeMinutes * 100.0 / NULLIF(ft.ActualDurationMinutes, 0)) > 15;
    
    -- Output alerts (integrate with email/Teams via SQL Agent or external service)
    SELECT * FROM @AlertMessages
    ORDER BY 
        CASE Severity 
            WHEN 'CRITICAL' THEN 1 
            WHEN 'WARNING' THEN 2 
            ELSE 3 
        END,
        TriggeredAt DESC;
END;
GO
```

## Power BI Configuration

### Connect to SQL Server

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Connection settings:
   - Server: `${SQL_SERVER_HOSTNAME}`
   - Database: `LogiFleetPulse`
   - Data Connectivity mode: **DirectQuery** (for real-time) or **Import** (for better performance)

### Import the Data Model

```powerquery
// Power Query M code for custom fact aggregation
let
    Source = Sql.Database("${SQL_SERVER_HOSTNAME}", "LogiFleetPulse"),
    
    // Load fact tables
    FactWarehouse = Source{[Schema="DWH",Item="FactWarehouseOperations"]}[Data],
    FactFleet = Source{[Schema="DWH",Item="FactFleetTrips"]}[Data],
    
    // Load dimensions
    DimTime = Source{[Schema="DWH",Item="DimTime"]}[Data],
    DimProduct = Source{[Schema="DWH",Item="DimProductGravity"]}[Data],
    DimGeo = Source{[Schema="DWH",Item="DimGeography"]}[Data],
    
    // Load views
    CrossFactView = Source{[Schema="DWH",Item="vw_WarehouseFleetImpact"]}[Data]
in
    CrossFactView
```

### Create DAX Measures for Cross-Fact KPIs

```dax
// Total Dwell Time Hours
Total Dwell Hours = 
SUMX(
    FactWarehouseOperations,
    FactWarehouseOperations[DwellTimeMinutes] / 60
)

// Fleet Efficiency Score (composite metric)
Fleet Efficiency Score = 
VAR AvgIdlePercent = 
    AVERAGEX(
        FactFleetTrips,
        DIVIDE(
            FactFleetTrips[IdleTimeMinutes],
            FactFleetTrips[ActualDurationMinutes],
            0
        )
    ) * 100
VAR AvgFuelEfficiency = AVERAGE(FactFleetTrips[FuelEfficiencyKMPerLiter])
VAR NormalizedEfficiency = DIVIDE(AvgFuelEfficiency, 10, 0) * 100 // Assume 10 km/L is baseline
RETURN
    (100 - AvgIdlePercent) * 0.6 + NormalizedEfficiency * 0.4

// Gravity Zone Alignment Score
Gravity Alignment % = 
VAR HighGravityInOptimalZones = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        DimProductGravity[GravityScore] > 50,
        DimWarehouseZone[ZoneType] = "HighGravity"
    )
VAR TotalHighGravityOps = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        DimProductGravity[GravityScore] > 50
    )
RETURN
    DIVIDE(HighGravityInOptimalZones, TotalHighGravityOps, 0) * 100

// Cross-Fact: Dwell Impact on Fleet Delays
Dwell-Driven Delay Rate = 
CALCULATE(
    DIVIDE(
        COUNTROWS(vw_WarehouseFleetImpact),
        COUNTROWS(FactFleetTrips),
        0
    ) * 100,
    vw_WarehouseFleetImpact[HighDwellCausedDelay] = 1
)
```

### Configure Row-Level Security

```dax
// Create role "Regional Managers" - only see their assigned regions
[Region] IN VALUES(UserRegions[Region])

// Where UserRegions is a table mapping UserPrincipalName to allowed regions
```

In Power BI Desktop:
1. Modeling → Manage Roles
2. Create role "RegionalManager"
3. Add DAX filter to DimGeography: `[Region] = USERNAME()`
4. Publish to Power BI Service
5. Assign Azure AD users to roles

## Common Patterns and Workflows

### Pattern 1: Real-Time Dashboard Refresh

```sql
-- Schedule this via SQL Server Agent every 15 minutes
EXEC ETL.usp_LoadWarehouseOperations;
EXEC ETL.usp_LoadFleetTrips; -- Similar procedure for fleet data
EXEC ETL.usp_CheckLogisticsAlerts;
```

In Power BI Service:
- Configure dataset refresh: **Every 15 minutes** (requires Power BI Premium)
- Set up alerts on key measures (e.g., Fleet Efficiency Score < 70)

### Pattern 2: Gravity Score Recalculation

```sql
-- Quarterly job to recalculate product gravity based on recent velocity
UPDATE DWH.DimProductGravity
SET 
    GravityScore = (
        (VelocityRank * 0.4) +                    -- 40% weight on velocity
        (UnitValue / 100 * 0.3) +                 -- 30% weight on value
        ((1.0 / NULLIF(FragilityScore, 0)) * 0.3) -- 30% inverse fragility
    ),
    LastUpdated = GETUTCDATE()
FROM DWH.DimProductGravity p
CROSS APPLY (
    SELECT 
        NTILE(100) OVER (ORDER BY COUNT(*) DESC) AS VelocityRank
    FROM DWH.FactWarehouseOperations wo
    WHERE wo.ProductKey = p.ProductKey
        AND wo.TimeKey >= DATEADD(MONTH, -3, GETUTCDATE())
) v;
```

### Pattern 3: External Data Integration (Weather API)

```sql
-- Create external table for weather delays (using Polybase or linked server)
CREATE EXTERNAL TABLE ExtData.WeatherEvents (
    EventDateTime DATETIME,
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    SeverityCode VARCHAR(20),
    EventType VARCHAR(50)
)
WITH (
    DATA_SOURCE = WeatherAPIDataSource,
    LOCATION = '/api/v1/events',
    FILE_FORMAT = JSONFormat
);

-- Correlate with fleet delays
SELECT 
    ft.TripID,
    ft.VehicleID,
    g.LocationName,
    ft.IsDelayed,
    w.SeverityCode,
    w.EventType
FROM DWH.FactFleetTrips ft
INNER JOIN DWH.DimGeography g ON ft.OriginGeographyKey = g.GeographyKey
CROSS APPLY (
    SELECT TOP 1 *
    FROM ExtData.WeatherEvents w
    WHERE w.EventDateTime BETWEEN DATEADD(HOUR, -2, ft.StartTime) AND ft.StartTime
        AND geography::Point(g.Latitude, g.Longitude, 4326)
            .STDistance(geography::Point(w.Latitude, w.Longitude, 4326)) < 50000 -- Within 50km
    ORDER BY w.EventDateTime DESC
) w
WHERE ft.WeatherImpact = 1;
```

## Troubleshooting

### Issue: Slow Cross-Fact Queries

**Symptoms:** Queries joining FactWarehouseOperations and FactFleetTrips timeout

**Solutions:**

```sql
-- 1. Verify columnstore indexes exist
SELECT 
    OBJECT_NAME(object_id) AS TableName,
    name AS IndexName,
    type_desc
FROM sys.indexes
WHERE type_desc LIKE '%COLUMNSTORE%'
    AND OBJECT_NAME(object_id) LIKE 'Fact%';

-- 2. Update statistics
UPDATE STATISTICS DWH.FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS DWH.FactFleetTrips WITH FULLSCAN;

-- 3. Consider partitioning by TimeKey
CREATE PARTITION FUNCTION PF_TimeKey (INT)
AS RANGE RIGHT FOR VALUES (
    20260101, 20260201, 20260301, 20260401, 20260501, 20260601
);

CREATE PARTITION SCHEME PS_TimeKey
AS PARTITION PF_TimeKey ALL TO ([PRIMARY]);

-- Rebuild fact tables on partition scheme
```

### Issue: Power
