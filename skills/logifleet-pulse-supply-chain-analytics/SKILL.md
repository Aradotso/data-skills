---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing template for logistics, fleet management, and supply chain intelligence
triggers:
  - "set up LogiFleet Pulse analytics platform"
  - "deploy supply chain data warehouse schema"
  - "configure Power BI logistics dashboard"
  - "implement warehouse gravity zone optimization"
  - "create fleet telemetry tracking system"
  - "build cross-modal supply chain KPI reporting"
  - "integrate warehouse and fleet analytics"
  - "setup multi-fact star schema for logistics"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive data warehousing and analytics template for logistics operations. It provides:

- **Multi-fact star schema** linking warehouse operations, fleet telemetry, and inventory data
- **Power BI dashboard templates** for real-time logistics visualization
- **MS SQL Server scripts** for dimensional modeling and ETL processes
- **Predictive analytics** for bottleneck detection and fleet optimization
- **Cross-functional KPIs** harmonizing warehouse velocity with fleet performance

The platform unifies disparate logistics data sources (WMS, TMS, telematics, supplier portals) into a single semantic layer for decision support.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to source systems (WMS, fleet telemetry, ERP)

### Database Deployment

1. **Clone the repository:**

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Create the database:**

```sql
-- Create the logistics data warehouse
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Set recovery model for production
ALTER DATABASE LogiFleetPulse SET RECOVERY FULL
GO
```

3. **Deploy dimension tables:**

```sql
-- DimTime: 15-minute granularity time dimension
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    TimeValue DATETIME NOT NULL,
    Hour TINYINT NOT NULL,
    QuarterHour TINYINT NOT NULL,
    DayOfWeek TINYINT NOT NULL,
    DayName VARCHAR(10) NOT NULL,
    IsWeekend BIT NOT NULL,
    FiscalPeriod VARCHAR(7) NOT NULL,
    INDEX IX_TimeValue (TimeValue)
)
GO

-- DimGeography: Hierarchical location dimension
CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationCode VARCHAR(20) NOT NULL,
    LocationName VARCHAR(100) NOT NULL,
    LocationType VARCHAR(20) NOT NULL, -- 'Warehouse', 'RouteNode', 'Hub'
    ParentGeographyKey INT NULL,
    Region VARCHAR(50) NOT NULL,
    Country VARCHAR(50) NOT NULL,
    Latitude DECIMAL(9,6) NULL,
    Longitude DECIMAL(9,6) NULL,
    IsActive BIT NOT NULL DEFAULT 1,
    FOREIGN KEY (ParentGeographyKey) REFERENCES DimGeography(GeographyKey)
)
GO

-- DimProductGravity: Product classification with velocity scoring
CREATE TABLE DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName VARCHAR(200) NOT NULL,
    Category VARCHAR(50) NOT NULL,
    SubCategory VARCHAR(50) NULL,
    GravityScore DECIMAL(5,2) NOT NULL DEFAULT 0, -- Higher = faster moving
    ValuePerUnit DECIMAL(10,2) NOT NULL,
    FragilityIndex TINYINT NOT NULL DEFAULT 1, -- 1-10 scale
    RequiresColdStorage BIT NOT NULL DEFAULT 0,
    AverageLeadTimeDays INT NULL,
    IsActive BIT NOT NULL DEFAULT 1,
    INDEX IX_GravityScore (GravityScore DESC),
    INDEX IX_SKU (SKU)
)
GO

-- DimSupplierReliability: Supplier performance tracking
CREATE TABLE DimSupplierReliability (
    SupplierKey INT IDENTITY(1,1) PRIMARY KEY,
    SupplierCode VARCHAR(20) NOT NULL UNIQUE,
    SupplierName VARCHAR(100) NOT NULL,
    LeadTimeVariance DECIMAL(5,2) NULL, -- Standard deviation in days
    DefectRate DECIMAL(5,4) NULL, -- Percentage
    ComplianceScore TINYINT NULL, -- 0-100
    LastAuditDate DATE NULL,
    IsActive BIT NOT NULL DEFAULT 1
)
GO

-- DimFleetVehicle: Fleet asset dimension
CREATE TABLE DimFleetVehicle (
    VehicleKey INT IDENTITY(1,1) PRIMARY KEY,
    VehicleID VARCHAR(20) NOT NULL UNIQUE,
    VehicleType VARCHAR(50) NOT NULL, -- 'Box Truck', 'Refrigerated', 'Van'
    Capacity DECIMAL(10,2) NOT NULL, -- Cubic feet or weight
    FuelType VARCHAR(20) NOT NULL,
    YearManufactured INT NOT NULL,
    LastMaintenanceDate DATE NULL,
    MaintenanceStatus VARCHAR(20) NOT NULL DEFAULT 'Operational',
    IsActive BIT NOT NULL DEFAULT 1
)
GO
```

4. **Deploy fact tables:**

```sql
-- FactWarehouseOperations: Core warehouse activity tracking
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(20) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    Quantity INT NOT NULL,
    DwellTimeMinutes INT NULL,
    ProcessingTimeMinutes INT NOT NULL,
    AssignedZone VARCHAR(20) NULL,
    OperatorID VARCHAR(20) NULL,
    BatchID VARCHAR(50) NULL,
    CostPerUnit DECIMAL(10,4) NULL,
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
)
GO

CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey)
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey)
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Geography ON FactWarehouseOperations(GeographyKey)
GO

-- FactFleetTrips: Fleet telemetry and route performance
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    StartTimeKey INT NOT NULL,
    EndTimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    DistanceMiles DECIMAL(8,2) NOT NULL,
    DurationMinutes INT NOT NULL,
    IdleTimeMinutes INT NOT NULL,
    FuelConsumedGallons DECIMAL(8,3) NOT NULL,
    LoadWeight DECIMAL(10,2) NULL,
    AverageSpeed DECIMAL(5,2) NULL,
    DelayMinutes INT NOT NULL DEFAULT 0,
    DelayReason VARCHAR(100) NULL,
    DriverID VARCHAR(20) NULL,
    TripCost DECIMAL(10,2) NULL,
    FOREIGN KEY (StartTimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (EndTimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (VehicleKey) REFERENCES DimFleetVehicle(VehicleKey),
    FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
)
GO

CREATE NONCLUSTERED INDEX IX_FactFleet_StartTime ON FactFleetTrips(StartTimeKey)
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleKey)
GO

-- FactCrossDock: Cross-docking operations tracking
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    InboundTimeKey INT NOT NULL,
    OutboundTimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundGeographyKey INT NOT NULL,
    OutboundGeographyKey INT NOT NULL,
    Quantity INT NOT NULL,
    DwellTimeMinutes INT NOT NULL,
    InboundTripKey BIGINT NULL,
    OutboundTripKey BIGINT NULL,
    FOREIGN KEY (InboundTimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (OutboundTimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    FOREIGN KEY (InboundGeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (OutboundGeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (InboundTripKey) REFERENCES FactFleetTrips(TripKey),
    FOREIGN KEY (OutboundTripKey) REFERENCES FactFleetTrips(TripKey)
)
GO
```

### Data Integration

Configure data sources in your ETL process:

```sql
-- Stored procedure for incremental warehouse operations load
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Example: Load from external WMS staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        Quantity, DwellTimeMinutes, ProcessingTimeMinutes,
        AssignedZone, OperatorID, BatchID, CostPerUnit
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        wms.OperationType,
        wms.Quantity,
        wms.DwellTimeMinutes,
        wms.ProcessingTimeMinutes,
        wms.AssignedZone,
        wms.OperatorID,
        wms.BatchID,
        wms.CostPerUnit
    FROM WMS_StagingOperations wms
    INNER JOIN DimTime t ON CAST(wms.OperationTimestamp AS DATE) = CAST(t.TimeValue AS DATE)
        AND DATEPART(HOUR, wms.OperationTimestamp) = t.Hour
        AND DATEPART(MINUTE, wms.OperationTimestamp) / 15 = t.QuarterHour
    INNER JOIN DimGeography g ON wms.WarehouseCode = g.LocationCode
    INNER JOIN DimProductGravity p ON wms.SKU = p.SKU
    WHERE wms.LoadedToWarehouse = 0
        AND wms.OperationTimestamp > @LastLoadTime
    
    -- Mark records as loaded
    UPDATE WMS_StagingOperations
    SET LoadedToWarehouse = 1
    WHERE OperationTimestamp > @LastLoadTime
END
GO

-- Stored procedure for fleet telemetry load
CREATE PROCEDURE usp_LoadFleetTrips
    @LastLoadTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactFleetTrips (
        StartTimeKey, EndTimeKey, VehicleKey,
        OriginGeographyKey, DestinationGeographyKey,
        DistanceMiles, DurationMinutes, IdleTimeMinutes,
        FuelConsumedGallons, LoadWeight, AverageSpeed,
        DelayMinutes, DelayReason, DriverID, TripCost
    )
    SELECT 
        ts.TimeKey AS StartTimeKey,
        te.TimeKey AS EndTimeKey,
        v.VehicleKey,
        go.GeographyKey AS OriginGeographyKey,
        gd.GeographyKey AS DestinationGeographyKey,
        telem.DistanceMiles,
        telem.DurationMinutes,
        telem.IdleTimeMinutes,
        telem.FuelConsumedGallons,
        telem.LoadWeight,
        telem.AverageSpeed,
        telem.DelayMinutes,
        telem.DelayReason,
        telem.DriverID,
        telem.TripCost
    FROM Fleet_StagingTelemetry telem
    INNER JOIN DimTime ts ON CAST(telem.TripStartTime AS DATE) = CAST(ts.TimeValue AS DATE)
        AND DATEPART(HOUR, telem.TripStartTime) = ts.Hour
    INNER JOIN DimTime te ON CAST(telem.TripEndTime AS DATE) = CAST(te.TimeValue AS DATE)
        AND DATEPART(HOUR, telem.TripEndTime) = te.Hour
    INNER JOIN DimFleetVehicle v ON telem.VehicleID = v.VehicleID
    INNER JOIN DimGeography go ON telem.OriginCode = go.LocationCode
    INNER JOIN DimGeography gd ON telem.DestinationCode = gd.LocationCode
    WHERE telem.LoadedToWarehouse = 0
        AND telem.TripStartTime > @LastLoadTime
    
    UPDATE Fleet_StagingTelemetry
    SET LoadedToWarehouse = 1
    WHERE TripStartTime > @LastLoadTime
END
GO
```

### Power BI Configuration

1. **Open the template file** `LogiFleet_Pulse_Master.pbit`

2. **Configure data source connection:**
   - Server: `${SQL_SERVER_HOST}`
   - Database: `LogiFleetPulse`
   - Authentication: Use environment variable `${SQL_SERVER_USERNAME}` and `${SQL_SERVER_PASSWORD}`

3. **Key measures are pre-configured** in the template:

```dax
-- Average Dwell Time
AvgDwellTime = 
AVERAGE(FactWarehouseOperations[DwellTimeMinutes])

-- Fleet Utilization Rate
FleetUtilization = 
DIVIDE(
    SUM(FactFleetTrips[DurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[DurationMinutes]),
    0
)

-- Warehouse Throughput per Hour
ThroughputPerHour = 
DIVIDE(
    SUM(FactWarehouseOperations[Quantity]),
    DISTINCTCOUNT(DimTime[Hour]),
    0
)

-- Cross-Fact KPI: Dwell Cost per Mile
DwellCostPerMile = 
VAR TotalDwellCost = 
    SUMX(
        FactWarehouseOperations,
        [DwellTimeMinutes] * [CostPerUnit]
    )
VAR TotalMiles = SUM(FactFleetTrips[DistanceMiles])
RETURN
    DIVIDE(TotalDwellCost, TotalMiles, 0)

-- Gravity Zone Performance Score
GravityZoneScore = 
AVERAGEX(
    VALUES(FactWarehouseOperations[AssignedZone]),
    CALCULATE(
        DIVIDE(
            SUM(FactWarehouseOperations[Quantity]),
            SUM(FactWarehouseOperations[ProcessingTimeMinutes]),
            0
        ) * AVERAGE(DimProductGravity[GravityScore])
    )
)
```

## Key Features & Usage Patterns

### 1. Warehouse Gravity Zone Optimization

Calculate and assign gravity scores based on product velocity:

```sql
-- Update gravity scores based on 90-day moving average
UPDATE DimProductGravity
SET GravityScore = subq.CalculatedScore
FROM (
    SELECT 
        p.ProductKey,
        (
            SUM(f.Quantity) / NULLIF(SUM(f.ProcessingTimeMinutes), 0) * 100 +
            p.ValuePerUnit / 10 +
            (10 - p.FragilityIndex)
        ) AS CalculatedScore
    FROM DimProductGravity p
    LEFT JOIN FactWarehouseOperations f ON p.ProductKey = f.ProductKey
    WHERE f.TimeKey >= (SELECT MAX(TimeKey) - 8640 FROM DimTime) -- Last 90 days
    GROUP BY p.ProductKey, p.ValuePerUnit, p.FragilityIndex
) subq
WHERE DimProductGravity.ProductKey = subq.ProductKey
GO

-- Recommend zone reassignments
SELECT 
    p.SKU,
    p.ProductName,
    f.AssignedZone AS CurrentZone,
    CASE 
        WHEN p.GravityScore > 75 THEN 'A' -- High velocity, near shipping
        WHEN p.GravityScore > 50 THEN 'B' -- Medium velocity
        WHEN p.GravityScore > 25 THEN 'C' -- Low velocity
        ELSE 'D' -- Very low velocity, deep storage
    END AS RecommendedZone,
    p.GravityScore,
    AVG(f.DwellTimeMinutes) AS AvgDwellTime
FROM DimProductGravity p
INNER JOIN FactWarehouseOperations f ON p.ProductKey = f.ProductKey
WHERE f.TimeKey >= (SELECT MAX(TimeKey) - 2880 FROM DimTime) -- Last 30 days
GROUP BY p.SKU, p.ProductName, f.AssignedZone, p.GravityScore
HAVING f.AssignedZone <> CASE 
    WHEN p.GravityScore > 75 THEN 'A'
    WHEN p.GravityScore > 50 THEN 'B'
    WHEN p.GravityScore > 25 THEN 'C'
    ELSE 'D'
END
ORDER BY p.GravityScore DESC
GO
```

### 2. Fleet Idle Time Analysis

Identify routes with excessive idle time:

```sql
-- Fleet idle time hotspots
SELECT 
    v.VehicleID,
    v.VehicleType,
    go.LocationName AS Origin,
    gd.LocationName AS Destination,
    COUNT(*) AS TripCount,
    AVG(f.IdleTimeMinutes) AS AvgIdleTime,
    AVG(f.DurationMinutes) AS AvgTripDuration,
    CAST(AVG(f.IdleTimeMinutes) * 100.0 / NULLIF(AVG(f.DurationMinutes), 0) AS DECIMAL(5,2)) AS IdlePercentage,
    SUM(f.IdleTimeMinutes * 0.35) AS EstimatedIdleCost -- $0.35 per minute assumption
FROM FactFleetTrips f
INNER JOIN DimFleetVehicle v ON f.VehicleKey = v.VehicleKey
INNER JOIN DimGeography go ON f.OriginGeographyKey = go.GeographyKey
INNER JOIN DimGeography gd ON f.DestinationGeographyKey = gd.GeographyKey
WHERE f.StartTimeKey >= (SELECT MAX(TimeKey) - 2880 FROM DimTime) -- Last 30 days
GROUP BY v.VehicleID, v.VehicleType, go.LocationName, gd.LocationName
HAVING AVG(f.IdleTimeMinutes) * 100.0 / NULLIF(AVG(f.DurationMinutes), 0) > 15
ORDER BY IdlePercentage DESC
GO
```

### 3. Cross-Modal KPI Reporting

Link warehouse dwell time to fleet delays:

```sql
-- Products with high dwell causing fleet delays
WITH HighDwellProducts AS (
    SELECT 
        ProductKey,
        AVG(DwellTimeMinutes) AS AvgDwell
    FROM FactWarehouseOperations
    WHERE TimeKey >= (SELECT MAX(TimeKey) - 2880 FROM DimTime)
        AND OperationType = 'Shipping'
    GROUP BY ProductKey
    HAVING AVG(DwellTimeMinutes) > 120 -- Over 2 hours
)
SELECT 
    p.SKU,
    p.ProductName,
    hdp.AvgDwell,
    COUNT(DISTINCT f.TripKey) AS AffectedTrips,
    AVG(f.DelayMinutes) AS AvgTripDelay,
    SUM(f.DelayMinutes) AS TotalDelayMinutes
FROM HighDwellProducts hdp
INNER JOIN DimProductGravity p ON hdp.ProductKey = p.ProductKey
INNER JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
INNER JOIN FactFleetTrips f ON wo.GeographyKey = f.OriginGeographyKey
    AND wo.TimeKey BETWEEN f.StartTimeKey - 96 AND f.StartTimeKey -- 24-hour window before trip
WHERE f.DelayMinutes > 30
GROUP BY p.SKU, p.ProductName, hdp.AvgDwell
ORDER BY TotalDelayMinutes DESC
GO
```

### 4. Predictive Maintenance Alerts

Generate fleet maintenance queue based on weighted criteria:

```sql
-- Adaptive fleet triage scoring
CREATE VIEW vw_FleetMaintenancePriority
AS
SELECT 
    v.VehicleID,
    v.VehicleType,
    v.LastMaintenanceDate,
    DATEDIFF(DAY, v.LastMaintenanceDate, GETDATE()) AS DaysSinceService,
    COUNT(f.TripKey) AS RecentTripCount,
    AVG(f.DistanceMiles) AS AvgTripDistance,
    SUM(f.FuelConsumedGallons) AS TotalFuelConsumed,
    AVG(f.IdleTimeMinutes) AS AvgIdleTime,
    -- Weighted priority score
    (
        DATEDIFF(DAY, v.LastMaintenanceDate, GETDATE()) * 0.5 + -- Age factor
        COUNT(f.TripKey) * 0.3 + -- Usage frequency
        AVG(f.IdleTimeMinutes) * 0.2 -- Stress indicator
    ) AS MaintenancePriorityScore,
    CASE 
        WHEN DATEDIFF(DAY, v.LastMaintenanceDate, GETDATE()) > 180 THEN 'Critical'
        WHEN DATEDIFF(DAY, v.LastMaintenanceDate, GETDATE()) > 120 THEN 'High'
        WHEN DATEDIFF(DAY, v.LastMaintenanceDate, GETDATE()) > 60 THEN 'Medium'
        ELSE 'Low'
    END AS MaintenanceUrgency
FROM DimFleetVehicle v
LEFT JOIN FactFleetTrips f ON v.VehicleKey = f.VehicleKey
    AND f.StartTimeKey >= (SELECT MAX(TimeKey) - 2880 FROM DimTime) -- Last 30 days
WHERE v.IsActive = 1
GROUP BY v.VehicleID, v.VehicleType, v.LastMaintenanceDate
GO

-- Use the view
SELECT TOP 10 *
FROM vw_FleetMaintenancePriority
ORDER BY MaintenancePriorityScore DESC
GO
```

### 5. Temporal Elasticity Simulation

Run what-if scenarios for capacity changes:

```sql
-- Simulate 95% warehouse capacity impact on throughput
DECLARE @TargetCapacityPercent DECIMAL(5,2) = 95.0
DECLARE @CurrentAvgThroughput DECIMAL(10,2)
DECLARE @ProjectedThroughput DECIMAL(10,2)

-- Calculate current baseline
SELECT @CurrentAvgThroughput = AVG(HourlyThroughput)
FROM (
    SELECT 
        TimeKey,
        SUM(Quantity) AS HourlyThroughput
    FROM FactWarehouseOperations
    WHERE TimeKey >= (SELECT MAX(TimeKey) - 2880 FROM DimTime)
    GROUP BY TimeKey
) subq

-- Project throughput with capacity constraint
-- Assumes exponential decay in efficiency beyond 85% capacity
SELECT @ProjectedThroughput = @CurrentAvgThroughput * 
    CASE 
        WHEN @TargetCapacityPercent <= 85 THEN 1.0
        ELSE 1.0 - ((@TargetCapacityPercent - 85) / 100.0) * 0.4 -- 40% efficiency loss at 100%
    END

SELECT 
    'Current Avg Throughput' AS Metric,
    @CurrentAvgThroughput AS Value
UNION ALL
SELECT 
    'Projected Throughput at ' + CAST(@TargetCapacityPercent AS VARCHAR) + '% Capacity',
    @ProjectedThroughput
UNION ALL
SELECT 
    'Estimated Throughput Change',
    @ProjectedThroughput - @CurrentAvgThroughput
GO
```

## Configuration & Environment Variables

Set these environment variables for automated ETL and reporting:

```bash
# Database connection
export SQL_SERVER_HOST="your-server.database.windows.net"
export SQL_SERVER_DATABASE="LogiFleetPulse"
export SQL_SERVER_USERNAME="logifleet_etl_user"
export SQL_SERVER_PASSWORD="${SECURE_PASSWORD}"

# External data sources
export WMS_API_ENDPOINT="https://wms.yourcompany.com/api/v1"
export WMS_API_KEY="${WMS_API_KEY}"
export FLEET_TELEMETRY_ENDPOINT="https://telematics.provider.com/api"
export FLEET_API_KEY="${FLEET_API_KEY}"

# Power BI service (for scheduled refresh)
export POWERBI_WORKSPACE_ID="${WORKSPACE_ID}"
export POWERBI_DATASET_ID="${DATASET_ID}"
export POWERBI_CLIENT_ID="${AZURE_AD_CLIENT_ID}"
export POWERBI_CLIENT_SECRET="${AZURE_AD_CLIENT_SECRET}"

# Alert thresholds
export ALERT_DWELL_TIME_THRESHOLD=120  # minutes
export ALERT_IDLE_TIME_THRESHOLD=15    # percent of trip duration
export ALERT_EMAIL_RECIPIENTS="logistics@yourcompany.com,ops@yourcompany.com"
```

## Automated Alerting

Set up threshold-based notifications:

```sql
-- Stored procedure for automated alerting
CREATE PROCEDURE usp_GenerateLogisticsAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    -- High dwell time alerts
    INSERT INTO AlertQueue (AlertType, Severity, Message, GeneratedAt)
    SELECT 
        'HighDwellTime',
        'Warning',
        'SKU ' + p.SKU + ' (' + p.ProductName + ') has avg dwell time of ' + 
        CAST(AVG(f.DwellTimeMinutes) AS VARCHAR) + ' minutes in last 24 hours',
        GETDATE()
    FROM FactWarehouseOperations f
    INNER JOIN DimProductGravity p ON f.ProductKey = p.ProductKey
    WHERE f.TimeKey >= (SELECT MAX(TimeKey) - 96 FROM DimTime) -- Last 24 hours
        AND f.DwellTimeMinutes IS NOT NULL
    GROUP BY p.SKU, p.ProductName, p.ProductKey
    HAVING AVG(f.DwellTimeMinutes) > CAST('${ALERT_DWELL_TIME_THRESHOLD}' AS INT)
    
    -- Excessive fleet idle time alerts
    INSERT INTO AlertQueue (AlertType, Severity, Message, GeneratedAt)
    SELECT 
        'ExcessiveIdleTime',
        'Critical',
        'Vehicle ' + v.VehicleID + ' idle time is ' + 
        CAST(AVG(f.IdleTimeMinutes * 100.0 / f.DurationMinutes) AS DECIMAL(5,2)) + 
        '% of trip duration',
        GETDATE()
    FROM FactFleetTrips f
    INNER JOIN DimFleetVehicle v ON f.VehicleKey = v.VehicleKey
    WHERE f.StartTimeKey >= (SELECT MAX(TimeKey) - 96 FROM DimTime)
    GROUP BY v.VehicleID, v.VehicleKey
    HAVING AVG(f.IdleTimeMinutes * 100.0 / f.DurationMinutes) > 
        CAST('${ALERT_IDLE_TIME_THRESHOLD}' AS DECIMAL)
    
    -- Cross-dock bottleneck alerts
    INSERT INTO AlertQueue (AlertType, Severity, Message, GeneratedAt)
    SELECT 
        'CrossDockBottleneck',
        'High',
        'Cross-dock location ' + g.LocationName + ' has avg dwell of ' + 
        CAST(AVG(cd.DwellTimeMinutes) AS VARCHAR) + ' minutes',
        GETDATE()
    FROM FactCrossDock cd
    INNER JOIN DimGeography g ON cd.InboundGeographyKey = g.GeographyKey
    WHERE cd.InboundTimeKey >= (SELECT MAX(TimeKey) - 96 FROM DimTime)
    GROUP BY g.LocationName, g.GeographyKey
    HAVING AVG(cd.DwellTimeMinutes) > 45
END
GO

-- Schedule via SQL Server Agent Job (execute every 15 minutes)
```

## Troubleshooting

### Common Issues

**1. Power BI gateway timeout errors:**

```sql
-- Create indexed views for complex cross-fact queries
CREATE VIEW vw_WarehouseFleetSummary
WITH SCHEMABINDING
AS
SELECT 
    t.TimeKey,
    g.GeographyKey,
    COUNT_BIG(*) AS RecordCount,
    SUM(ISNULL(wo.Quantity, 0)) AS TotalWarehouseQty,
    SUM(ISNULL(wo.DwellTimeMinutes, 0)) AS TotalDwellMinutes,
    COUNT(DISTINCT f.TripKey) AS TripCount
FROM dbo.DimTime t
LEFT JOIN dbo.FactWarehouseOperations wo ON t.TimeKey = wo.TimeKey
LEFT JOIN dbo.DimGeography g ON wo.GeographyKey = g.GeographyKey
LEFT JOIN dbo.FactFleetTrips f ON g.GeographyKey = f.OriginGeographyKey
    AND t.TimeKey = f.StartTimeKey
GROUP BY t.TimeKey, g.GeographyKey
GO

CREATE UNIQUE CLUSTERED INDEX IX_Summary ON vw_WarehouseFleetSummary(TimeKey, GeographyKey)
GO
```

**2. Slow dimension table lookups:**

```sql
-- Ensure all foreign keys are indexed
IF NOT EXISTS (SELECT 1 FROM sys.indexes WHERE name = 'IX_FactWarehouse_ProductKey')
    CREATE NONCLUSTERED INDEX IX_FactWarehouse_ProductKey 
    ON FactWarehouseOperations(Product
