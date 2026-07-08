---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing template for logistics, fleet, and supply chain analytics with multi-fact star schema
triggers:
  - "set up logistics analytics dashboard"
  - "implement supply chain data warehouse"
  - "create fleet management Power BI reports"
  - "deploy warehouse operations analytics"
  - "build multi-fact star schema for logistics"
  - "configure cross-modal supply chain tracking"
  - "implement KPI dashboard for warehouse and fleet"
  - "set up real-time logistics intelligence platform"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a **comprehensive data warehousing and analytics template** for logistics operations. It provides:

- **Multi-fact star schema** for warehouse operations, fleet trips, and cross-dock activities
- **Power BI dashboard templates** for real-time logistics intelligence
- **SQL Server database schema** with time-phased dimensions and bridge tables
- **Cross-KPI harmonization** linking warehouse metrics with fleet performance
- **Predictive analytics** for bottleneck detection and maintenance scheduling

The project is designed for 3PL operators, retail chains, food distributors, and any organization managing complex supply chain operations.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version recommended)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to source systems (WMS, TMS, telemetry feeds)

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Create a new database for LogiFleet Pulse
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Deploy dimension tables first
-- DimTime: 15-minute granularity time dimension
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME NOT NULL,
    HourOfDay TINYINT NOT NULL,
    MinuteBucket TINYINT NOT NULL, -- 0, 15, 30, 45
    DayOfWeek TINYINT NOT NULL,
    DayName VARCHAR(20) NOT NULL,
    WeekOfYear TINYINT NOT NULL,
    MonthNumber TINYINT NOT NULL,
    MonthName VARCHAR(20) NOT NULL,
    Quarter TINYINT NOT NULL,
    FiscalYear INT NOT NULL,
    IsWeekday BIT NOT NULL,
    IsHoliday BIT NOT NULL
);

-- Create clustered index on FullDateTime for range queries
CREATE CLUSTERED INDEX IX_DimTime_FullDateTime ON DimTime(FullDateTime);
GO

-- DimGeography: Hierarchical location dimension
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) NOT NULL,
    LocationName VARCHAR(200) NOT NULL,
    LocationType VARCHAR(50) NOT NULL, -- Warehouse, Route Node, Cross-Dock
    AddressLine1 VARCHAR(200),
    City VARCHAR(100),
    StateProvince VARCHAR(100),
    PostalCode VARCHAR(20),
    Country VARCHAR(100),
    Region VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone VARCHAR(50),
    IsActive BIT NOT NULL DEFAULT 1,
    ValidFrom DATETIME2 NOT NULL DEFAULT GETDATE(),
    ValidTo DATETIME2
);

CREATE UNIQUE INDEX IX_DimGeography_LocationID ON DimGeography(LocationID);
GO

-- DimProductGravity: Product classification with gravity scoring
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) NOT NULL,
    ProductName VARCHAR(200) NOT NULL,
    ProductCategory VARCHAR(100),
    ProductSubcategory VARCHAR(100),
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    UnitValue DECIMAL(12,2),
    Fragility TINYINT, -- 1-10 scale
    PickFrequency INT, -- Monthly picks
    GravityScore DECIMAL(5,2), -- Calculated: (PickFrequency * UnitValue) / (UnitWeight * Fragility)
    StorageZoneRecommendation VARCHAR(50),
    IsPerishable BIT NOT NULL DEFAULT 0,
    ShelfLifeDays INT,
    ValidFrom DATETIME2 NOT NULL DEFAULT GETDATE(),
    ValidTo DATETIME2
);

CREATE UNIQUE INDEX IX_DimProductGravity_SKU ON DimProductGravity(SKU);
CREATE INDEX IX_DimProductGravity_GravityScore ON DimProductGravity(GravityScore DESC);
GO

-- DimSupplierReliability: Supplier performance metrics
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) NOT NULL,
    SupplierName VARCHAR(200) NOT NULL,
    SupplierCountry VARCHAR(100),
    LeadTimeMeanDays DECIMAL(5,2),
    LeadTimeStdDev DECIMAL(5,2),
    DefectRate DECIMAL(5,4), -- Percentage
    OnTimeDeliveryRate DECIMAL(5,4), -- Percentage
    ComplianceScore TINYINT, -- 1-100
    ReliabilityRank INT,
    ValidFrom DATETIME2 NOT NULL DEFAULT GETDATE(),
    ValidTo DATETIME2
);

CREATE UNIQUE INDEX IX_DimSupplierReliability_SupplierID ON DimSupplierReliability(SupplierID);
GO
```

### Step 2: Create Fact Tables

```sql
-- FactWarehouseOperations: Core warehouse activities
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    
    OperationType VARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    OrderID VARCHAR(100),
    BatchID VARCHAR(100),
    
    QuantityHandled INT NOT NULL,
    DwellTimeMinutes INT, -- Time spent in warehouse
    ProcessingTimeMinutes INT, -- Active handling time
    StorageZone VARCHAR(50),
    
    OperatorID VARCHAR(50),
    EquipmentID VARCHAR(50),
    
    QualityCheckPassed BIT,
    DefectCount INT DEFAULT 0,
    
    CostAmount DECIMAL(12,2),
    
    CreatedAt DATETIME2 NOT NULL DEFAULT GETDATE()
);

-- Partitioning strategy: by month for efficient archival
CREATE INDEX IX_FactWarehouseOperations_Time ON FactWarehouseOperations(TimeKey);
CREATE INDEX IX_FactWarehouseOperations_Product ON FactWarehouseOperations(ProductKey);
CREATE INDEX IX_FactWarehouseOperations_Geography ON FactWarehouseOperations(GeographyKey);
GO

-- FactFleetTrips: Fleet telemetry and route data
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    StartTimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    EndTimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50),
    RouteID VARCHAR(100),
    
    DistanceKm DECIMAL(10,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    
    FuelConsumedLiters DECIMAL(8,2),
    AverageMPG DECIMAL(5,2),
    
    LoadWeight DECIMAL(10,2),
    LoadVolume DECIMAL(10,2),
    
    SpeedViolations INT DEFAULT 0,
    HarshBrakingEvents INT DEFAULT 0,
    
    WeatherCondition VARCHAR(50),
    TrafficDelay BIT DEFAULT 0,
    
    MaintenanceAlertTriggered BIT DEFAULT 0,
    TirePressureWarning BIT DEFAULT 0,
    
    TripCost DECIMAL(12,2),
    
    CreatedAt DATETIME2 NOT NULL DEFAULT GETDATE()
);

CREATE INDEX IX_FactFleetTrips_StartTime ON FactFleetTrips(StartTimeKey);
CREATE INDEX IX_FactFleetTrips_Vehicle ON FactFleetTrips(VehicleID);
CREATE INDEX IX_FactFleetTrips_Route ON FactFleetTrips(RouteID);
GO

-- FactCrossDock: Transfer activities bypassing long-term storage
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    InboundTimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    OutboundTimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    CrossDockGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    
    QuantityTransferred INT NOT NULL,
    DockDwellMinutes INT,
    
    OriginSupplierKey INT FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    DestinationCustomerID VARCHAR(50),
    
    CreatedAt DATETIME2 NOT NULL DEFAULT GETDATE()
);

CREATE INDEX IX_FactCrossDock_InboundTime ON FactCrossDock(InboundTimeKey);
CREATE INDEX IX_FactCrossDock_Product ON FactCrossDock(ProductKey);
GO
```

### Step 3: Create Bridge Table for Many-to-Many Relationships

```sql
-- BridgeRouteStorageZone: Links fleet routes to warehouse zones
CREATE TABLE BridgeRouteStorageZone (
    BridgeKey INT PRIMARY KEY IDENTITY(1,1),
    RouteID VARCHAR(100) NOT NULL,
    StorageZone VARCHAR(50) NOT NULL,
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    RelationshipWeight DECIMAL(3,2) NOT NULL DEFAULT 1.0, -- For weighted aggregations
    ValidFrom DATETIME2 NOT NULL DEFAULT GETDATE(),
    ValidTo DATETIME2
);

CREATE INDEX IX_BridgeRouteStorageZone_Route ON BridgeRouteStorageZone(RouteID);
CREATE INDEX IX_BridgeRouteStorageZone_Zone ON BridgeRouteStorageZone(StorageZone);
GO
```

### Step 4: Create Stored Procedures for Data Loading

```sql
-- Procedure: Load DimTime dimension (populate for multiple years)
CREATE PROCEDURE usp_PopulateDimTime
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentDateTime DATETIME2 = @StartDate;
    DECLARE @TimeKey INT;
    DECLARE @MinuteBucket TINYINT;
    
    WHILE @CurrentDateTime <= @EndDate
    BEGIN
        SET @MinuteBucket = (DATEPART(MINUTE, @CurrentDateTime) / 15) * 15;
        SET @CurrentDateTime = DATEADD(MINUTE, @MinuteBucket - DATEPART(MINUTE, @CurrentDateTime), @CurrentDateTime);
        
        SET @TimeKey = CAST(FORMAT(@CurrentDateTime, 'yyyyMMddHHmm') AS INT);
        
        IF NOT EXISTS (SELECT 1 FROM DimTime WHERE TimeKey = @TimeKey)
        BEGIN
            INSERT INTO DimTime (
                TimeKey, FullDateTime, DateKey, TimeOfDay, HourOfDay, MinuteBucket,
                DayOfWeek, DayName, WeekOfYear, MonthNumber, MonthName,
                Quarter, FiscalYear, IsWeekday, IsHoliday
            )
            VALUES (
                @TimeKey,
                @CurrentDateTime,
                CAST(FORMAT(@CurrentDateTime, 'yyyyMMdd') AS INT),
                CAST(@CurrentDateTime AS TIME),
                DATEPART(HOUR, @CurrentDateTime),
                @MinuteBucket,
                DATEPART(WEEKDAY, @CurrentDateTime),
                DATENAME(WEEKDAY, @CurrentDateTime),
                DATEPART(WEEK, @CurrentDateTime),
                DATEPART(MONTH, @CurrentDateTime),
                DATENAME(MONTH, @CurrentDateTime),
                DATEPART(QUARTER, @CurrentDateTime),
                YEAR(@CurrentDateTime),
                CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1, 7) THEN 0 ELSE 1 END,
                0 -- Update holiday logic separately
            );
        END;
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END;
END;
GO

-- Execute to populate 2 years of time dimension
EXEC usp_PopulateDimTime @StartDate = '2025-01-01', @EndDate = '2026-12-31';
GO
```

### Step 5: Create Views for Common Analytics Queries

```sql
-- View: Cross-fact KPI combining warehouse dwell with fleet idle time
CREATE VIEW vw_DwellTimeToFleetIdle AS
SELECT 
    dt.FiscalYear,
    dt.Quarter,
    dt.MonthName,
    dg.LocationName AS Warehouse,
    dp.ProductCategory,
    dp.GravityScore,
    
    -- Warehouse metrics
    AVG(fwo.DwellTimeMinutes) AS AvgWarehouseDwellMinutes,
    SUM(fwo.QuantityHandled) AS TotalQuantityHandled,
    
    -- Fleet metrics (linked via bridge)
    AVG(fft.IdleTimeMinutes) AS AvgFleetIdleMinutes,
    AVG(fft.FuelConsumedLiters) AS AvgFuelConsumedPerTrip,
    
    -- Cross-KPI calculation
    AVG(fwo.DwellTimeMinutes) * AVG(fft.IdleTimeMinutes) AS DwellIdleProduct
    
FROM FactWarehouseOperations fwo
INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
INNER JOIN DimGeography dg ON fwo.GeographyKey = dg.GeographyKey
INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
INNER JOIN BridgeRouteStorageZone brs ON dg.GeographyKey = brs.GeographyKey AND fwo.StorageZone = brs.StorageZone
INNER JOIN FactFleetTrips fft ON brs.RouteID = fft.RouteID
    AND fft.StartTimeKey BETWEEN dt.TimeKey - 100 AND dt.TimeKey + 100 -- Within 1-2 hours

GROUP BY 
    dt.FiscalYear, dt.Quarter, dt.MonthName, dg.LocationName, 
    dp.ProductCategory, dp.GravityScore;
GO

-- View: Predictive bottleneck index
CREATE VIEW vw_BottleneckIndex AS
SELECT 
    dg.LocationName,
    fwo.StorageZone,
    dt.DayOfWeek,
    dt.HourOfDay,
    
    -- Bottleneck indicators
    AVG(fwo.ProcessingTimeMinutes) AS AvgProcessingTime,
    STDEV(fwo.ProcessingTimeMinutes) AS ProcessingTimeVariance,
    SUM(CASE WHEN fwo.DwellTimeMinutes > 180 THEN 1 ELSE 0 END) AS HighDwellCount,
    AVG(fft.TrafficDelay) AS TrafficDelayRate,
    
    -- Bottleneck score: higher = more risk
    (AVG(fwo.ProcessingTimeMinutes) * STDEV(fwo.ProcessingTimeMinutes)) / NULLIF(AVG(fwo.QuantityHandled), 0) AS BottleneckScore
    
FROM FactWarehouseOperations fwo
INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
INNER JOIN DimGeography dg ON fwo.GeographyKey = dg.GeographyKey
LEFT JOIN BridgeRouteStorageZone brs ON dg.GeographyKey = brs.GeographyKey AND fwo.StorageZone = brs.StorageZone
LEFT JOIN FactFleetTrips fft ON brs.RouteID = fft.RouteID

GROUP BY 
    dg.LocationName, fwo.StorageZone, dt.DayOfWeek, dt.HourOfDay;
GO
```

### Step 6: Configure Data Sources

Create a configuration file `config.json` (not included in repo):

```json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_USERNAME}",
    "password": "${SQL_PASSWORD}",
    "trusted_connection": false
  },
  "data_sources": {
    "wms_api": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "api_key": "${WMS_API_KEY}",
      "poll_interval_minutes": 15
    },
    "telematics_feed": {
      "endpoint": "${TELEMATICS_FEED_URL}",
      "auth_token": "${TELEMATICS_AUTH_TOKEN}",
      "poll_interval_minutes": 5
    },
    "weather_api": {
      "endpoint": "https://api.weatherapi.com/v1",
      "api_key": "${WEATHER_API_KEY}"
    }
  },
  "powerbi": {
    "workspace_id": "${POWERBI_WORKSPACE_ID}",
    "dataset_id": "${POWERBI_DATASET_ID}",
    "refresh_schedule": "0 */15 * * *"
  }
}
```

### Step 7: Import Power BI Template

1. Open Power BI Desktop
2. File → Import → Power BI Template
3. Select `LogiFleet_Pulse_Master.pbit` from the repository
4. Enter connection parameters:
   - SQL Server: `${SQL_SERVER_HOST}`
   - Database: `LogiFleetPulse`
   - Authentication: Use your credentials or service principal

The template automatically detects fact tables and builds relationships.

## Key SQL Queries & Patterns

### Calculate Gravity Score for Products

```sql
-- Update gravity scores based on recent activity
UPDATE dp
SET 
    GravityScore = (
        (ISNULL(pick_freq.MonthlyPicks, 0) * dp.UnitValue) / 
        NULLIF((dp.UnitWeight * ISNULL(dp.Fragility, 5)), 0)
    ),
    StorageZoneRecommendation = CASE 
        WHEN (ISNULL(pick_freq.MonthlyPicks, 0) * dp.UnitValue) / NULLIF((dp.UnitWeight * ISNULL(dp.Fragility, 5)), 0) > 100 THEN 'High-Gravity'
        WHEN (ISNULL(pick_freq.MonthlyPicks, 0) * dp.UnitValue) / NULLIF((dp.UnitWeight * ISNULL(dp.Fragility, 5)), 0) > 50 THEN 'Medium-Gravity'
        ELSE 'Low-Gravity'
    END
FROM DimProductGravity dp
LEFT JOIN (
    SELECT 
        ProductKey,
        COUNT(*) AS MonthlyPicks
    FROM FactWarehouseOperations
    WHERE 
        OperationType = 'Picking'
        AND CreatedAt >= DATEADD(MONTH, -1, GETDATE())
    GROUP BY ProductKey
) pick_freq ON dp.ProductKey = pick_freq.ProductKey;
```

### Identify Fleet Maintenance Priority

```sql
-- Adaptive fleet triage: maintenance priority by revenue impact
SELECT TOP 20
    fft.VehicleID,
    COUNT(*) AS TotalTrips,
    SUM(fft.DistanceKm) AS TotalDistanceKm,
    AVG(fft.FuelConsumedLiters) AS AvgFuelConsumption,
    SUM(fft.SpeedViolations + fft.HarshBrakingEvents) AS SafetyIncidents,
    SUM(fft.TripCost) AS TotalRevenue,
    
    -- Maintenance urgency score
    (
        (SUM(fft.SpeedViolations + fft.HarshBrakingEvents) * 10) +
        (AVG(fft.FuelConsumedLiters) * 5) +
        (SUM(CASE WHEN fft.TirePressureWarning = 1 THEN 1 ELSE 0 END) * 20)
    ) * (SUM(fft.TripCost) / NULLIF(COUNT(*), 0)) AS MaintenancePriorityScore
    
FROM FactFleetTrips fft
WHERE 
    fft.CreatedAt >= DATEADD(MONTH, -3, GETDATE())
    AND (fft.MaintenanceAlertTriggered = 1 OR fft.TirePressureWarning = 1 OR fft.SpeedViolations > 0)
    
GROUP BY fft.VehicleID
ORDER BY MaintenancePriorityScore DESC;
```

### Cross-Fact KPI: Dwell Time Impact on Fleet Costs

```sql
-- Correlate warehouse dwell time with downstream fleet fuel consumption
SELECT 
    dp.ProductCategory,
    fwo.StorageZone,
    
    AVG(fwo.DwellTimeMinutes) AS AvgWarehouseDwellMinutes,
    
    -- Fleet metrics for shipments of these products
    AVG(fft.FuelConsumedLiters) AS AvgFuelPerTrip,
    AVG(fft.IdleTimeMinutes) AS AvgIdleTime,
    
    -- Correlation indicator
    CASE 
        WHEN AVG(fwo.DwellTimeMinutes) > 240 AND AVG(fft.IdleTimeMinutes) > 45 THEN 'High Correlation'
        WHEN AVG(fwo.DwellTimeMinutes) > 180 AND AVG(fft.IdleTimeMinutes) > 30 THEN 'Medium Correlation'
        ELSE 'Low Correlation'
    END AS DwellToIdleCorrelation

FROM FactWarehouseOperations fwo
INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
INNER JOIN FactCrossDock fcd ON fwo.ProductKey = fcd.ProductKey
INNER JOIN FactFleetTrips fft ON fcd.OutboundTripKey = fft.TripKey

WHERE fwo.OperationType = 'Shipping'
    AND fwo.CreatedAt >= DATEADD(MONTH, -1, GETDATE())

GROUP BY dp.ProductCategory, fwo.StorageZone;
```

### Temporal Elasticity Simulation

```sql
-- Scenario: What if warehouse capacity increases from 80% to 95%?
WITH CapacityScenario AS (
    SELECT 
        dg.LocationName,
        COUNT(DISTINCT fwo.OrderID) AS CurrentOrders,
        AVG(fwo.DwellTimeMinutes) AS CurrentAvgDwell,
        
        -- Simulate 19% increase in orders (95/80 = 1.1875)
        COUNT(DISTINCT fwo.OrderID) * 1.1875 AS ProjectedOrders,
        
        -- Estimate dwell time increase (empirical: +15% per 10% capacity increase)
        AVG(fwo.DwellTimeMinutes) * (1 + (0.15 * 1.5)) AS ProjectedAvgDwell
        
    FROM FactWarehouseOperations fwo
    INNER JOIN DimGeography dg ON fwo.GeographyKey = dg.GeographyKey
    WHERE fwo.CreatedAt >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY dg.LocationName
)
SELECT 
    LocationName,
    CurrentOrders,
    ProjectedOrders,
    CurrentAvgDwell,
    ProjectedAvgDwell,
    (ProjectedAvgDwell - CurrentAvgDwell) AS DwellTimeIncrease,
    
    -- Impact on fleet utilization (linked via cross-dock)
    ProjectedOrders * (ProjectedAvgDwell / 60.0) / 24.0 AS EstimatedFleetDaysImpact
    
FROM CapacityScenario;
```

## Power BI Configuration

### Connect to SQL Server

In Power BI Desktop:

1. Get Data → SQL Server
2. Server: `${SQL_SERVER_HOST}`
3. Database: `LogiFleetPulse`
4. Data Connectivity mode: **Import** (for smaller datasets) or **DirectQuery** (for real-time)
5. Advanced Options:
   - Command timeout: 30 minutes
   - Enable query folding: Yes

### Define Relationships (Auto-Detected)

The star schema relationships are automatically created:

- `FactWarehouseOperations[TimeKey]` → `DimTime[TimeKey]`
- `FactWarehouseOperations[GeographyKey]` → `DimGeography[GeographyKey]`
- `FactWarehouseOperations[ProductKey]` → `DimProductGravity[ProductKey]`
- `FactFleetTrips[StartTimeKey]` → `DimTime[TimeKey]` (role-playing dimension)
- `FactFleetTrips[OriginGeographyKey]` → `DimGeography[GeographyKey]`

### Create DAX Measures

```dax
// Total Dwell Time (Hours)
TotalDwellHours = 
SUMX(
    FactWarehouseOperations,
    FactWarehouseOperations[DwellTimeMinutes] / 60
)

// Fleet Utilization Rate
FleetUtilizationRate = 
DIVIDE(
    SUM(FactFleetTrips[DurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[DurationMinutes]),
    0
)

// Gravity-Weighted Throughput
GravityWeightedThroughput = 
SUMX(
    FactWarehouseOperations,
    FactWarehouseOperations[QuantityHandled] * RELATED(DimProductGravity[GravityScore])
)

// Bottleneck Risk Index (0-100)
BottleneckRiskIndex = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR StdDevDwell = STDEV.P(FactWarehouseOperations[DwellTimeMinutes])
VAR HighDwellCount = COUNTROWS(FILTER(FactWarehouseOperations, FactWarehouseOperations[DwellTimeMinutes] > 180))
RETURN
    MIN(
        ((AvgDwell * StdDevDwell) / 1000) + (HighDwellCount / 10),
        100
    )

// Cross-Modal Efficiency Score
CrossModalEfficiency = 
VAR WarehouseEfficiency = DIVIDE([TotalQuantityHandled], [TotalDwellHours], 0)
VAR FleetEfficiency = [FleetUtilizationRate] * 100
RETURN
    (WarehouseEfficiency + FleetEfficiency) / 2
```

### Configure Row-Level Security

```dax
// Create role: WarehouseManager
// Filter: DimGeography[LocationName] = USERNAME()

// Create role: FleetSupervisor
// Filter: FactFleetTrips[DriverID] IN LOOKUPVALUE(UserRoles[DriverIDs], UserRoles[Username], USERNAME())

// Assign users to roles in Power BI Service
```

## Automated Alerts

### SQL Agent Job for KPI Breach Alerts

```sql
-- Create stored procedure for alert monitoring
CREATE PROCEDURE usp_MonitorKPIBreaches
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Check for high dwell time (> 240 minutes)
    DECLARE @HighDwellCount INT;
    SELECT @HighDwellCount = COUNT(*)
    FROM FactWarehouseOperations
    WHERE DwellTimeMinutes > 240
        AND CreatedAt >= DATEADD(HOUR, -1, GETDATE());
    
    IF @HighDwellCount > 10
    BEGIN
        -- Send alert via email (requires Database Mail configured)
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetPulse',
            @recipients = '${ALERT_EMAIL}',
            @subject = 'LogiFleet Alert: High Dwell Time Detected',
            @body = 'More than 10 items have exceeded 240 minutes dwell time in the last hour. Review warehouse operations immediately.',
            @importance = 'High';
    END;
    
    -- Check for fleet maintenance alerts
    DECLARE @MaintenanceAlerts INT;
    SELECT @MaintenanceAlerts = COUNT(*)
    FROM FactFleetTrips
    WHERE MaintenanceAlertTriggered = 1
        AND CreatedAt >= DATEADD(HOUR, -2, GETDATE());
    
    IF @MaintenanceAlerts > 5
    BEGIN
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetPulse',
            @recipients = '${FLEET_ALERT_EMAIL}',
            @subject = 'LogiFleet Alert: Multiple Maintenance Alerts',
            @body = 'More than 5 vehicles have triggered maintenance alerts in the last 2 hours. Review fleet status.',
            @importance = 'High';
    END;
END;
GO

-- Schedule SQL Agent job to run every 15 minutes
-- USE msdb; EXEC sp_add_job... (see SQL Server Agent documentation)
```

## Troubleshooting

### Issue: Power BI Refresh Fails with Timeout

**Solution**: Use incremental refresh for large fact tables.

```powerquery
// In Power Query Editor (M language)
let
    Source = Sql.Database("${SQL_SERVER_HOST}", "LogiFl
