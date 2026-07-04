---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehouse for logistics intelligence with multi-fact star schema, fleet optimization, and warehouse analytics
triggers:
  - "set up logifleet pulse supply chain analytics"
  - "configure logistics data warehouse with power bi"
  - "implement warehouse gravity zones and fleet optimization"
  - "create multi-fact star schema for logistics"
  - "build supply chain intelligence dashboard"
  - "integrate fleet telemetry with warehouse operations"
  - "deploy logicore analytics supply chain"
  - "optimize logistics kpis with temporal modeling"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive logistics intelligence platform that unifies warehouse operations, fleet management, and supply chain analytics into a single semantic data model. It combines:

- **Multi-fact star schema** data warehouse on MS SQL Server
- **Power BI dashboards** for real-time logistics visualization
- **Cross-modal KPI harmonization** (warehouse + fleet + supplier metrics)
- **Predictive analytics** for bottleneck detection and maintenance triage
- **Warehouse Gravity Zones** spatial optimization
- **Temporal elasticity modeling** for scenario planning

The platform ingests data from WMS, telematics, GPS, supplier portals, and external APIs to provide unified logistics intelligence.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to source systems (WMS, TMS, telematics APIs)

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance and create database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    DateTime DATETIME NOT NULL,
    Hour TINYINT NOT NULL,
    DayOfWeek TINYINT NOT NULL,
    DayOfMonth TINYINT NOT NULL,
    WeekOfYear TINYINT NOT NULL,
    Month TINYINT NOT NULL,
    Quarter TINYINT NOT NULL,
    Year SMALLINT NOT NULL,
    FiscalPeriod VARCHAR(10),
    IsWeekend BIT NOT NULL,
    INDEX IX_DateTime NONCLUSTERED (DateTime)
);

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) NOT NULL UNIQUE,
    LocationName NVARCHAR(100) NOT NULL,
    LocationType VARCHAR(20) NOT NULL, -- 'Warehouse', 'RouteNode', 'CustomerSite'
    Address NVARCHAR(200),
    City NVARCHAR(100),
    StateProvince NVARCHAR(100),
    Country NVARCHAR(100),
    Continent NVARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone VARCHAR(50),
    INDEX IX_LocationID NONCLUSTERED (LocationID)
);

CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName NVARCHAR(200) NOT NULL,
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    UnitWeight DECIMAL(10,2), -- kg
    UnitVolume DECIMAL(10,3), -- cubic meters
    IsFragile BIT DEFAULT 0,
    IsPerishable BIT DEFAULT 0,
    ShelfLifeDays INT,
    UnitValue DECIMAL(12,2), -- currency
    GravityScore DECIMAL(5,2), -- calculated: velocity * value * fragility
    OptimalZoneType VARCHAR(20), -- 'FastMoving', 'SlowMoving', 'ColdStorage'
    INDEX IX_SKU NONCLUSTERED (SKU),
    INDEX IX_GravityScore NONCLUSTERED (GravityScore DESC)
);

CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) NOT NULL UNIQUE,
    SupplierName NVARCHAR(200) NOT NULL,
    Country NVARCHAR(100),
    AvgLeadTimeDays DECIMAL(5,1),
    LeadTimeVariance DECIMAL(5,2),
    DefectRate DECIMAL(5,4), -- percentage as decimal
    OnTimeDeliveryRate DECIMAL(5,4),
    ComplianceScore DECIMAL(5,2), -- 0-100
    INDEX IX_SupplierID NONCLUSTERED (SupplierID)
);

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    OperationType VARCHAR(20) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OperationTimestamp DATETIME NOT NULL,
    QuantityUnits INT NOT NULL,
    DwellTimeMinutes INT, -- time in storage or queue
    ProcessingTimeMinutes DECIMAL(8,2),
    StorageZoneID VARCHAR(50),
    EmployeeID VARCHAR(50),
    OrderID VARCHAR(50),
    INDEX IX_TimeGeo NONCLUSTERED (TimeKey, GeographyKey),
    INDEX IX_Product NONCLUSTERED (ProductKey),
    INDEX IX_OperationType NONCLUSTERED (OperationType, OperationTimestamp)
);

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50),
    TripStartTimestamp DATETIME NOT NULL,
    TripEndTimestamp DATETIME,
    DistanceKM DECIMAL(10,2),
    DurationMinutes DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(8,2),
    FuelConsumedLiters DECIMAL(8,2),
    LoadWeightKG DECIMAL(10,2),
    LoadVolumeCubicM DECIMAL(10,3),
    AvgSpeedKMH DECIMAL(5,2),
    MaxSpeedKMH DECIMAL(5,2),
    HarshBrakingCount INT DEFAULT 0,
    RapidAccelerationCount INT DEFAULT 0,
    WeatherCondition VARCHAR(50), -- from external API
    TrafficDelayMinutes INT DEFAULT 0,
    OrderIDs VARCHAR(MAX), -- comma-separated for linkage
    INDEX IX_TimeOrigin NONCLUSTERED (TimeKey, OriginGeographyKey),
    INDEX IX_Vehicle NONCLUSTERED (VehicleID, TripStartTimestamp),
    INDEX IX_Destination NONCLUSTERED (DestinationGeographyKey)
);

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    ArrivalTimestamp DATETIME NOT NULL,
    DepartureTimestamp DATETIME,
    DwellMinutes AS DATEDIFF(MINUTE, ArrivalTimestamp, DepartureTimestamp),
    QuantityUnits INT NOT NULL,
    INDEX IX_TimeGeo NONCLUSTERED (TimeKey, GeographyKey),
    INDEX IX_Trips NONCLUSTERED (InboundTripKey, OutboundTripKey)
);
```

### Step 2: Populate Time Dimension

```sql
-- Stored procedure to populate DimTime for 5 years
CREATE PROCEDURE PopulateDimTime
    @StartDate DATETIME = '2024-01-01',
    @EndDate DATETIME = '2029-12-31'
AS
BEGIN
    DECLARE @CurrentDateTime DATETIME = @StartDate;
    
    WHILE @CurrentDateTime <= @EndDate
    BEGIN
        INSERT INTO DimTime (
            DateTime, Hour, DayOfWeek, DayOfMonth, WeekOfYear,
            Month, Quarter, Year, FiscalPeriod, IsWeekend
        )
        VALUES (
            @CurrentDateTime,
            DATEPART(HOUR, @CurrentDateTime),
            DATEPART(WEEKDAY, @CurrentDateTime),
            DATEPART(DAY, @CurrentDateTime),
            DATEPART(WEEK, @CurrentDateTime),
            DATEPART(MONTH, @CurrentDateTime),
            DATEPART(QUARTER, @CurrentDateTime),
            DATEPART(YEAR, @CurrentDateTime),
            'FY' + CAST(DATEPART(YEAR, @CurrentDateTime) AS VARCHAR(4)) + 'Q' + CAST(DATEPART(QUARTER, @CurrentDateTime) AS VARCHAR(1)),
            CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1, 7) THEN 1 ELSE 0 END
        );
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END;
GO

EXEC PopulateDimTime;
```

### Step 3: Create Data Loading Views

```sql
-- View for cross-fact KPI analysis
CREATE VIEW vw_FleetWarehouseIntegration AS
SELECT 
    t.DateTime,
    t.Year, t.Quarter, t.Month,
    g.LocationName AS Warehouse,
    p.SKU, p.ProductName, p.GravityScore,
    wo.OperationType,
    wo.DwellTimeMinutes AS WarehouseDwellMin,
    wo.QuantityUnits AS WarehouseQty,
    ft.VehicleID,
    ft.IdleTimeMinutes AS FleetIdleMin,
    ft.FuelConsumedLiters,
    ft.DistanceKM,
    ft.LoadWeightKG
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
LEFT JOIN FactFleetTrips ft ON 
    ft.OriginGeographyKey = wo.GeographyKey 
    AND CHARINDEX(wo.OrderID, ft.OrderIDs) > 0
WHERE wo.OperationType = 'Shipping';
GO

-- View for gravity zone optimization
CREATE VIEW vw_WarehouseGravityAnalysis AS
SELECT 
    p.SKU, p.ProductName, p.Category,
    p.GravityScore,
    p.OptimalZoneType AS RecommendedZone,
    wo.StorageZoneID AS CurrentZone,
    AVG(wo.DwellTimeMinutes) AS AvgDwellMin,
    AVG(wo.ProcessingTimeMinutes) AS AvgProcessingMin,
    COUNT(*) AS OperationCount,
    SUM(wo.QuantityUnits) AS TotalUnits
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
WHERE wo.OperationType IN ('Picking', 'Putaway')
GROUP BY p.SKU, p.ProductName, p.Category, p.GravityScore, 
         p.OptimalZoneType, wo.StorageZoneID;
GO
```

### Step 4: Configure Data Sources

Create a configuration file `config.json`:

```json
{
  "sql_server": {
    "server": "SQL_SERVER_HOST",
    "database": "LogiFleetPulse",
    "authentication": "windows",
    "connection_string": "Server=${SQL_SERVER_HOST};Database=LogiFleetPulse;Trusted_Connection=True;"
  },
  "data_sources": {
    "wms_api": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "api_key": "${WMS_API_KEY}",
      "refresh_interval_minutes": 15
    },
    "telematics_api": {
      "endpoint": "${TELEMATICS_API_ENDPOINT}",
      "api_key": "${TELEMATICS_API_KEY}",
      "refresh_interval_minutes": 5
    },
    "weather_api": {
      "endpoint": "https://api.weatherapi.com/v1",
      "api_key": "${WEATHER_API_KEY}"
    }
  },
  "alerts": {
    "smtp_server": "${SMTP_SERVER}",
    "smtp_port": 587,
    "from_email": "${ALERT_FROM_EMAIL}",
    "recipients": ["${ALERT_RECIPIENT_EMAIL}"]
  }
}
```

### Step 5: Create ETL Stored Procedures

```sql
-- Incremental load for warehouse operations
CREATE PROCEDURE LoadWarehouseOperations
    @LastLoadTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new operations from staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, SupplierKey,
        OperationType, OperationTimestamp, QuantityUnits,
        DwellTimeMinutes, ProcessingTimeMinutes, StorageZoneID,
        EmployeeID, OrderID
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.SupplierKey,
        stg.OperationType,
        stg.OperationTimestamp,
        stg.QuantityUnits,
        stg.DwellTimeMinutes,
        stg.ProcessingTimeMinutes,
        stg.StorageZoneID,
        stg.EmployeeID,
        stg.OrderID
    FROM StagingWarehouseOperations stg
    INNER JOIN DimTime t ON 
        DATEADD(MINUTE, DATEDIFF(MINUTE, 0, stg.OperationTimestamp) / 15 * 15, 0) = t.DateTime
    INNER JOIN DimGeography g ON stg.LocationID = g.LocationID
    INNER JOIN DimProductGravity p ON stg.SKU = p.SKU
    LEFT JOIN DimSupplierReliability s ON stg.SupplierID = s.SupplierID
    WHERE stg.OperationTimestamp > @LastLoadTime
    AND NOT EXISTS (
        SELECT 1 FROM FactWarehouseOperations f
        WHERE f.OrderID = stg.OrderID 
        AND f.OperationType = stg.OperationType
        AND f.OperationTimestamp = stg.OperationTimestamp
    );
    
    SELECT @@ROWCOUNT AS RowsInserted;
END;
GO

-- Calculate gravity scores (run nightly)
CREATE PROCEDURE UpdateProductGravityScores
AS
BEGIN
    UPDATE p
    SET p.GravityScore = 
        (velocity.PickFrequency * 0.4) + 
        (p.UnitValue / 1000.0 * 0.3) + 
        (CASE WHEN p.IsFragile = 1 THEN 20 ELSE 0 END * 0.2) +
        (CASE WHEN p.IsPerishable = 1 THEN 15 ELSE 0 END * 0.1),
        p.OptimalZoneType = 
            CASE 
                WHEN (velocity.PickFrequency * 0.4 + p.UnitValue / 1000.0 * 0.3) > 50 THEN 'FastMoving'
                WHEN p.IsPerishable = 1 THEN 'ColdStorage'
                ELSE 'SlowMoving'
            END
    FROM DimProductGravity p
    INNER JOIN (
        SELECT 
            ProductKey,
            COUNT(*) * 100.0 / NULLIF(DATEDIFF(DAY, MIN(OperationTimestamp), MAX(OperationTimestamp)), 0) AS PickFrequency
        FROM FactWarehouseOperations
        WHERE OperationType = 'Picking'
        AND OperationTimestamp >= DATEADD(DAY, -90, GETDATE())
        GROUP BY ProductKey
    ) velocity ON p.ProductKey = velocity.ProductKey;
END;
GO
```

## Power BI Configuration

### Step 1: Connect to SQL Server

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Server: `${SQL_SERVER_HOST}`
4. Database: `LogiFleetPulse`
5. Import tables:
   - DimTime
   - DimGeography
   - DimProductGravity
   - DimSupplierReliability
   - FactWarehouseOperations
   - FactFleetTrips
   - FactCrossDock
   - vw_FleetWarehouseIntegration
   - vw_WarehouseGravityAnalysis

### Step 2: Configure Relationships

Power BI should auto-detect, but verify:

```
DimTime[TimeKey] → FactWarehouseOperations[TimeKey] (Many-to-One)
DimGeography[GeographyKey] → FactWarehouseOperations[GeographyKey] (Many-to-One)
DimProductGravity[ProductKey] → FactWarehouseOperations[ProductKey] (Many-to-One)
DimSupplierReliability[SupplierKey] → FactWarehouseOperations[SupplierKey] (Many-to-One)

DimTime[TimeKey] → FactFleetTrips[TimeKey] (Many-to-One)
DimGeography[GeographyKey] → FactFleetTrips[OriginGeographyKey] (Many-to-One)
DimGeography[GeographyKey] → FactFleetTrips[DestinationGeographyKey] (Many-to-One, Inactive - use for specific calculations)
```

### Step 3: Create DAX Measures

```dax
-- Fleet Utilization Rate
FleetUtilizationRate = 
DIVIDE(
    SUM(FactFleetTrips[DurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[DurationMinutes]),
    0
) * 100

-- Average Dwell Time by Gravity Zone
AvgDwellByGravity = 
AVERAGEX(
    FILTER(
        FactWarehouseOperations,
        FactWarehouseOperations[DwellTimeMinutes] > 0
    ),
    FactWarehouseOperations[DwellTimeMinutes]
)

-- Fuel Efficiency (KM per Liter)
FuelEfficiency = 
DIVIDE(
    SUM(FactFleetTrips[DistanceKM]),
    SUM(FactFleetTrips[FuelConsumedLiters]),
    0
)

-- Warehouse Throughput (units per hour)
WarehouseThroughput = 
DIVIDE(
    SUM(FactWarehouseOperations[QuantityUnits]),
    SUM(FactWarehouseOperations[ProcessingTimeMinutes]) / 60,
    0
)

-- Cross-Dock Efficiency Score (target < 120 minutes)
CrossDockEfficiency = 
VAR AvgDwell = AVERAGE(FactCrossDock[DwellMinutes])
RETURN
IF(
    AvgDwell <= 120,
    100,
    100 - ((AvgDwell - 120) / 120 * 100)
)

-- Predictive Bottleneck Index
BottleneckIndex = 
VAR HighDwell = 
    CALCULATE(
        COUNT(FactWarehouseOperations[OperationKey]),
        FactWarehouseOperations[DwellTimeMinutes] > 240
    )
VAR HighIdle = 
    CALCULATE(
        COUNT(FactFleetTrips[TripKey]),
        FactFleetTrips[IdleTimeMinutes] > 60
    )
VAR TotalOps = COUNT(FactWarehouseOperations[OperationKey]) + COUNT(FactFleetTrips[TripKey])
RETURN
(HighDwell + HighIdle) / NULLIF(TotalOps, 0) * 100

-- Supplier Impact Score
SupplierImpactScore = 
AVERAGEX(
    DimSupplierReliability,
    (DimSupplierReliability[OnTimeDeliveryRate] * 0.4) + 
    ((1 - DimSupplierReliability[DefectRate]) * 0.3) +
    (DimSupplierReliability[ComplianceScore] / 100 * 0.3)
) * 100
```

### Step 4: Create Key Visualizations

**Fleet Dashboard Page:**
- Map visual: Fleet real-time locations (OriginGeography)
- Line chart: Fuel efficiency over time
- KPI cards: Fleet utilization, average idle time, total distance
- Table: Top 10 vehicles by harsh braking events

**Warehouse Dashboard Page:**
- Heatmap: Dwell time by storage zone and product category
- Gauge: Warehouse throughput vs. target
- Scatter chart: Gravity score vs. actual pick frequency
- Slicer: Operation type, time period

**Executive Dashboard Page:**
- KPI cards: Bottleneck Index, Cross-Dock Efficiency, Supplier Impact Score
- Combo chart: Warehouse ops + fleet trips over time
- Matrix: Top products by gravity score with recommended vs. actual zone

## Common Patterns & Use Cases

### Pattern 1: Identify Misplaced Inventory

```sql
-- Find products in wrong gravity zones
SELECT 
    SKU, ProductName, Category,
    GravityScore,
    RecommendedZone,
    CurrentZone,
    AvgDwellMin,
    TotalUnits
FROM vw_WarehouseGravityAnalysis
WHERE RecommendedZone <> CurrentZone
AND OperationCount > 10 -- minimum significance
ORDER BY GravityScore DESC, AvgDwellMin DESC;
```

### Pattern 2: Fleet Maintenance Priority

```sql
-- Calculate proactive maintenance queue
WITH FleetMetrics AS (
    SELECT 
        VehicleID,
        COUNT(*) AS TripCount,
        AVG(IdleTimeMinutes) AS AvgIdle,
        SUM(HarshBrakingCount + RapidAccelerationCount) AS SafetyEvents,
        AVG(LoadWeightKG) AS AvgLoad,
        SUM(CASE WHEN p.UnitValue > 1000 THEN 1 ELSE 0 END) AS HighValueTrips
    FROM FactFleetTrips ft
    LEFT JOIN FactWarehouseOperations wo ON CHARINDEX(wo.OrderID, ft.OrderIDs) > 0
    LEFT JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    WHERE ft.TripStartTimestamp >= DATEADD(DAY, -30, GETDATE())
    GROUP BY VehicleID
)
SELECT 
    VehicleID,
    TripCount,
    AvgIdle,
    SafetyEvents,
    (SafetyEvents * 0.4) + 
    (AvgIdle / 10 * 0.3) + 
    (HighValueTrips * 0.3) AS MaintenancePriority
FROM FleetMetrics
ORDER BY MaintenancePriority DESC;
```

### Pattern 3: Temporal Scenario Analysis

```dax
// DAX measure for "What If" capacity analysis
ScenarioThroughput = 
VAR CurrentCapacity = 0.80 -- 80% current utilization
VAR TargetCapacity = 0.95 -- 95% target scenario
VAR ScalingFactor = TargetCapacity / CurrentCapacity
VAR BaselineThroughput = [WarehouseThroughput]
RETURN
BaselineThroughput * ScalingFactor * 
(1 - ((TargetCapacity - 0.85) * 0.15)) -- efficiency penalty above 85%
```

### Pattern 4: Anomaly Detection with User Feedback

```sql
-- Create anomaly tracking table
CREATE TABLE AnomalyLog (
    AnomalyID INT PRIMARY KEY IDENTITY(1,1),
    DetectedTimestamp DATETIME DEFAULT GETDATE(),
    AnomalyType VARCHAR(50), -- 'DwellSpike', 'IdleAnomaly', 'FuelSpike'
    EntityType VARCHAR(20), -- 'Warehouse', 'Fleet', 'Product'
    EntityID VARCHAR(50),
    Severity VARCHAR(10), -- 'Low', 'Medium', 'High'
    Description NVARCHAR(500),
    UserVerified BIT DEFAULT 0,
    VerifiedBy VARCHAR(100),
    RootCause NVARCHAR(500),
    INDEX IX_Detected NONCLUSTERED (DetectedTimestamp DESC)
);

-- Stored procedure to detect dwell time anomalies
CREATE PROCEDURE DetectDwellAnomalies
AS
BEGIN
    INSERT INTO AnomalyLog (AnomalyType, EntityType, EntityID, Severity, Description)
    SELECT 
        'DwellSpike' AS AnomalyType,
        'Product' AS EntityType,
        p.SKU AS EntityID,
        CASE 
            WHEN current.AvgDwell > baseline.AvgDwell * 2.5 THEN 'High'
            WHEN current.AvgDwell > baseline.AvgDwell * 1.8 THEN 'Medium'
            ELSE 'Low'
        END AS Severity,
        'Dwell time spiked to ' + CAST(current.AvgDwell AS VARCHAR(10)) + 
        ' minutes (baseline: ' + CAST(baseline.AvgDwell AS VARCHAR(10)) + ')' AS Description
    FROM (
        SELECT ProductKey, AVG(DwellTimeMinutes) AS AvgDwell
        FROM FactWarehouseOperations
        WHERE OperationTimestamp >= DATEADD(DAY, -1, GETDATE())
        GROUP BY ProductKey
    ) current
    INNER JOIN (
        SELECT ProductKey, AVG(DwellTimeMinutes) AS AvgDwell
        FROM FactWarehouseOperations
        WHERE OperationTimestamp BETWEEN DATEADD(DAY, -30, GETDATE()) AND DATEADD(DAY, -7, GETDATE())
        GROUP BY ProductKey
    ) baseline ON current.ProductKey = baseline.ProductKey
    INNER JOIN DimProductGravity p ON current.ProductKey = p.ProductKey
    WHERE current.AvgDwell > baseline.AvgDwell * 1.5
    AND NOT EXISTS (
        SELECT 1 FROM AnomalyLog a
        WHERE a.EntityID = p.SKU 
        AND a.DetectedTimestamp > DATEADD(HOUR, -24, GETDATE())
    );
END;
GO
```

## Automated Alerts Configuration

```sql
-- Stored procedure for threshold-based alerts
CREATE PROCEDURE CheckKPIThresholds
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX) = '';
    
    -- Check fleet idle time
    IF EXISTS (
        SELECT 1 FROM FactFleetTrips
        WHERE TripStartTimestamp >= DATEADD(HOUR, -2, GETDATE())
        AND IdleTimeMinutes > 60
    )
    BEGIN
        SET @AlertMessage += 'ALERT: Fleet vehicles with >60 min idle time detected. ';
    END
    
    -- Check warehouse dwell time
    IF EXISTS (
        SELECT 1 FROM FactWarehouseOperations
        WHERE OperationTimestamp >= DATEADD(HOUR, -4, GETDATE())
        AND DwellTimeMinutes > 240
        AND OperationType = 'Shipping'
    )
    BEGIN
        SET @AlertMessage += 'ALERT: Shipping orders with >4 hour dwell time. ';
    END
    
    -- Send email if alerts exist (requires Database Mail)
    IF LEN(@AlertMessage) > 0
    BEGIN
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_RECIPIENT_EMAIL}',
            @subject = 'LogiFleet Pulse KPI Threshold Breach',
            @body = @AlertMessage;
    END
END;
GO

-- Schedule via SQL Agent (run every 15 minutes)
-- EXEC msdb.dbo.sp_add_job @job_name = N'LogiFleet_KPI_Monitor';
```

## Troubleshooting

### Issue: Power BI Refresh Takes Too Long

**Solution:** Implement incremental refresh
```
1. In Power BI Desktop → Transform Data
2. Create parameters: RangeStart (DateTime), RangeEnd (DateTime)
3. Filter each fact table:
   - OperationTimestamp >= RangeStart AND OperationTimestamp < RangeEnd
4. Save and publish to Power BI Service
5. Configure incremental refresh policy:
   - Archive data older than 2 years
   - Refresh data in the last 30 days
```

### Issue: Relationships Not Working Correctly

**Solution:** Check cardinality and filter direction
```dax
// Test relationship with explicit USERELATIONSHIP
TestMeasure = 
CALCULATE(
    COUNT(FactFleetTrips[TripKey]),
    USERELATIONSHIP(DimGeography[GeographyKey], FactFleetTrips[DestinationGeographyKey])
)
```

### Issue: Gravity Score Calculations Seem Off

**Solution:** Verify staging data quality
```sql
-- Check for NULL or zero values
SELECT 
    'Missing Weight' AS Issue,
    COUNT(*) AS RecordCount
FROM DimProductGravity
WHERE UnitWeight IS NULL OR UnitWeight = 0
UNION ALL
SELECT 
    'Missing Value' AS Issue,
    COUNT(*)
FROM DimProductGravity
WHERE UnitValue IS NULL OR UnitValue = 0;

-- Recalculate with defaults
UPDATE DimProductGravity
SET UnitWeight = 1.0
WHERE UnitWeight IS NULL OR UnitWeight = 0;
```

### Issue: Slow Query Performance

**Solution:** Add columnstore indexes for fact tables
```sql
-- Create column
