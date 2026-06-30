---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehouse with multi-fact star schema for warehouse operations, fleet tracking, and supply chain KPI harmonization
triggers:
  - set up logifleet pulse supply chain analytics
  - deploy logistics data warehouse with power bi
  - configure multi-fact star schema for warehouse and fleet
  - integrate warehouse management and fleet telemetry data
  - build supply chain analytics dashboard
  - implement cross-modal logistics intelligence
  - create warehouse gravity zone optimization
  - set up predictive bottleneck detection for logistics
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive logistics intelligence platform that combines:

- **Multi-fact star schema** data warehouse on MS SQL Server
- **Power BI dashboards** for real-time visualization
- **Cross-fact KPI harmonization** linking warehouse operations with fleet performance
- **Warehouse Gravity Zones** for optimal storage placement
- **Predictive bottleneck detection** using temporal elasticity modeling
- **Adaptive fleet triage** based on telemetry and revenue impact

The system integrates data from warehouse management systems (WMS), fleet telematics, supplier portals, weather APIs, and customer orders into a unified semantic layer that enables cross-domain analytics (e.g., "how does warehouse dwell time impact fleet fuel efficiency?").

## Installation & Deployment

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to source systems: WMS, fleet telematics, ERP

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance and create the database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    FullDateTime DATETIME2 NOT NULL,
    [Date] DATE NOT NULL,
    [Hour] TINYINT NOT NULL,
    [Minute] TINYINT NOT NULL,
    QuarterHour TINYINT NOT NULL, -- 15-minute bucket (0-3)
    DayOfWeek TINYINT NOT NULL,
    WeekOfYear TINYINT NOT NULL,
    [Month] TINYINT NOT NULL,
    [Quarter] TINYINT NOT NULL,
    FiscalYear INT NOT NULL,
    IsBusinessHour BIT NOT NULL,
    CONSTRAINT UQ_DimTime_DateTime UNIQUE (FullDateTime)
)
GO

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) NOT NULL,
    LocationName NVARCHAR(200) NOT NULL,
    LocationType VARCHAR(50) NOT NULL, -- 'Warehouse', 'RouteNode', 'Supplier', 'Customer'
    StreetAddress NVARCHAR(300),
    City NVARCHAR(100),
    StateProvince NVARCHAR(100),
    Country NVARCHAR(100),
    PostalCode VARCHAR(20),
    Region NVARCHAR(100),
    Continent NVARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone VARCHAR(50),
    IsActive BIT DEFAULT 1,
    CONSTRAINT UQ_DimGeography_LocationID UNIQUE (LocationID)
)
GO

CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) NOT NULL,
    ProductName NVARCHAR(300) NOT NULL,
    ProductCategory NVARCHAR(100),
    ProductSubcategory NVARCHAR(100),
    UnitCost DECIMAL(18,4),
    UnitPrice DECIMAL(18,4),
    UnitWeight DECIMAL(10,3), -- kg
    IsFragile BIT DEFAULT 0,
    IsPerishable BIT DEFAULT 0,
    RequiresRefrigeration BIT DEFAULT 0,
    -- Gravity zone calculation inputs
    AvgDailyPickVolume INT DEFAULT 0,
    AvgReplenishmentLeadDays DECIMAL(5,2) DEFAULT 0,
    GravityScore DECIMAL(8,4) DEFAULT 0, -- Higher = closer to shipping dock
    RecommendedZone VARCHAR(50),
    LastGravityRecalc DATETIME2,
    IsActive BIT DEFAULT 1,
    CONSTRAINT UQ_DimProductGravity_SKU UNIQUE (SKU)
)
GO

CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) NOT NULL,
    SupplierName NVARCHAR(200) NOT NULL,
    SupplierCategory VARCHAR(100),
    AvgLeadTimeDays DECIMAL(5,2),
    LeadTimeVariancePercent DECIMAL(5,2), -- Standard deviation as %
    DefectRatePercent DECIMAL(5,4),
    OnTimeDeliveryPercent DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2), -- 0-100
    ReliabilityTier VARCHAR(20), -- 'Platinum', 'Gold', 'Silver', 'Bronze', 'Watch'
    LastScoringDate DATETIME2,
    IsActive BIT DEFAULT 1,
    CONSTRAINT UQ_DimSupplierReliability_SupplierID UNIQUE (SupplierID)
)
GO

CREATE TABLE DimFleetVehicle (
    VehicleKey INT PRIMARY KEY IDENTITY(1,1),
    VehicleID VARCHAR(50) NOT NULL,
    VehicleType VARCHAR(50) NOT NULL, -- 'Box Truck', 'Refrigerated', 'Flatbed', etc.
    Make VARCHAR(50),
    Model VARCHAR(50),
    [Year] INT,
    LicensePlate VARCHAR(20),
    MaxLoadCapacityKg DECIMAL(10,2),
    MaxVolumeM3 DECIMAL(8,2),
    FuelType VARCHAR(30),
    HasTelematics BIT DEFAULT 0,
    HasRefrigeration BIT DEFAULT 0,
    LastMaintenanceDate DATE,
    NextScheduledMaintenanceDate DATE,
    MaintenancePriorityScore DECIMAL(5,2) DEFAULT 0,
    IsActive BIT DEFAULT 1,
    CONSTRAINT UQ_DimFleetVehicle_VehicleID UNIQUE (VehicleID)
)
GO

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT NULL FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    OperationType VARCHAR(50) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OrderID VARCHAR(100),
    QuantityUnits INT NOT NULL,
    QuantityWeight DECIMAL(10,3), -- kg
    DwellTimeMinutes DECIMAL(10,2), -- Time from arrival to next stage
    ProcessingTimeMinutes DECIMAL(10,2), -- Actual handling time
    AssignedZone VARCHAR(50),
    OperatorID VARCHAR(50),
    EquipmentID VARCHAR(50), -- Forklift, pallet jack, etc.
    ExceptionFlag BIT DEFAULT 0,
    ExceptionReason NVARCHAR(500),
    TimestampUTC DATETIME2 NOT NULL,
    INDEX IX_FactWarehouseOps_Time (TimeKey),
    INDEX IX_FactWarehouseOps_Product (ProductKey),
    INDEX IX_FactWarehouseOps_Geography (GeographyKey),
    INDEX IX_FactWarehouseOps_OpType (OperationType)
)
GO

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleKey INT NOT NULL FOREIGN KEY REFERENCES DimFleetVehicle(VehicleKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    RouteID VARCHAR(100),
    TripDate DATE NOT NULL,
    DepartureTime DATETIME2 NOT NULL,
    ArrivalTime DATETIME2,
    PlannedDistanceKm DECIMAL(10,2),
    ActualDistanceKm DECIMAL(10,2),
    PlannedDurationMinutes DECIMAL(10,2),
    ActualDurationMinutes DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2) DEFAULT 0,
    FuelConsumedLiters DECIMAL(10,3),
    LoadWeightKg DECIMAL(10,2),
    LoadVolumeM3 DECIMAL(8,2),
    DriverID VARCHAR(50),
    WeatherCondition VARCHAR(50), -- From external API
    TrafficDelayMinutes DECIMAL(8,2) DEFAULT 0,
    MaintenanceEventFlag BIT DEFAULT 0,
    MaintenanceNotes NVARCHAR(1000),
    TimestampUTC DATETIME2 NOT NULL,
    INDEX IX_FactFleetTrips_Time (TimeKey),
    INDEX IX_FactFleetTrips_Vehicle (VehicleKey),
    INDEX IX_FactFleetTrips_Route (RouteID),
    INDEX IX_FactFleetTrips_TripDate (TripDate)
)
GO

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundVehicleKey INT NOT NULL FOREIGN KEY REFERENCES DimFleetVehicle(VehicleKey),
    OutboundVehicleKey INT NOT NULL FOREIGN KEY REFERENCES DimFleetVehicle(VehicleKey),
    InboundArrivalTime DATETIME2 NOT NULL,
    OutboundDepartureTime DATETIME2 NOT NULL,
    CrossDockDurationMinutes DECIMAL(10,2),
    QuantityUnits INT NOT NULL,
    QuantityWeight DECIMAL(10,3),
    TransferZone VARCHAR(50),
    HandlingEquipmentID VARCHAR(50),
    ExceptionFlag BIT DEFAULT 0,
    ExceptionReason NVARCHAR(500),
    TimestampUTC DATETIME2 NOT NULL,
    INDEX IX_FactCrossDock_Time (TimeKey),
    INDEX IX_FactCrossDock_Product (ProductKey),
    INDEX IX_FactCrossDock_Geography (GeographyKey)
)
GO
```

### Step 2: Populate Time Dimension

```sql
-- Stored procedure to populate DimTime with 15-minute granularity
CREATE PROCEDURE sp_PopulateDimTime
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentDateTime DATETIME2 = @StartDate;
    DECLARE @EndDateTime DATETIME2 = DATEADD(DAY, 1, @EndDate);
    
    WHILE @CurrentDateTime < @EndDateTime
    BEGIN
        INSERT INTO DimTime (
            FullDateTime, [Date], [Hour], [Minute], QuarterHour,
            DayOfWeek, WeekOfYear, [Month], [Quarter], FiscalYear, IsBusinessHour
        )
        VALUES (
            @CurrentDateTime,
            CAST(@CurrentDateTime AS DATE),
            DATEPART(HOUR, @CurrentDateTime),
            DATEPART(MINUTE, @CurrentDateTime),
            DATEPART(MINUTE, @CurrentDateTime) / 15,
            DATEPART(WEEKDAY, @CurrentDateTime),
            DATEPART(WEEK, @CurrentDateTime),
            DATEPART(MONTH, @CurrentDateTime),
            DATEPART(QUARTER, @CurrentDateTime),
            CASE 
                WHEN DATEPART(MONTH, @CurrentDateTime) >= 7 
                THEN YEAR(@CurrentDateTime) + 1
                ELSE YEAR(@CurrentDateTime)
            END,
            CASE 
                WHEN DATEPART(WEEKDAY, @CurrentDateTime) BETWEEN 2 AND 6 -- Mon-Fri
                     AND DATEPART(HOUR, @CurrentDateTime) BETWEEN 8 AND 17 -- 8am-5pm
                THEN 1
                ELSE 0
            END
        );
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END
GO

-- Execute to populate 2 years of time dimension
EXEC sp_PopulateDimTime '2025-01-01', '2026-12-31';
GO
```

### Step 3: Calculate Product Gravity Scores

```sql
-- Stored procedure to calculate warehouse gravity zones
CREATE PROCEDURE sp_CalculateProductGravity
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Update gravity scores based on velocity, value, and replenishment lead time
    UPDATE DimProductGravity
    SET 
        GravityScore = (
            (AvgDailyPickVolume * 0.5) + -- Pick velocity weight
            ((UnitPrice / NULLIF(UnitCost, 0)) * 10 * 0.3) + -- Margin contribution
            ((10 / NULLIF(AvgReplenishmentLeadDays, 0)) * 0.2) -- Replenishment urgency
        ),
        RecommendedZone = CASE
            WHEN (AvgDailyPickVolume * 0.5 + ((UnitPrice / NULLIF(UnitCost, 0)) * 10 * 0.3) + ((10 / NULLIF(AvgReplenishmentLeadDays, 0)) * 0.2)) > 50
                THEN 'High-Gravity-A'
            WHEN (AvgDailyPickVolume * 0.5 + ((UnitPrice / NULLIF(UnitCost, 0)) * 10 * 0.3) + ((10 / NULLIF(AvgReplenishmentLeadDays, 0)) * 0.2)) > 25
                THEN 'Medium-Gravity-B'
            ELSE 'Low-Gravity-C'
        END,
        LastGravityRecalc = GETUTCDATE()
    WHERE IsActive = 1;
    
    -- Log recommendation
    PRINT 'Gravity scores updated for ' + CAST(@@ROWCOUNT AS VARCHAR) + ' active products';
END
GO
```

### Step 4: Configure Data Sources

Create a configuration file `config.json` (not included in repo - user must create):

```json
{
  "connections": {
    "wms_api": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "auth_token": "${WMS_API_TOKEN}"
    },
    "fleet_telematics": {
      "endpoint": "${TELEMATICS_API_ENDPOINT}",
      "api_key": "${TELEMATICS_API_KEY}"
    },
    "weather_api": {
      "endpoint": "https://api.weatherapi.com/v1",
      "api_key": "${WEATHER_API_KEY}"
    },
    "erp_connection": {
      "server": "${ERP_SQL_SERVER}",
      "database": "${ERP_DATABASE}",
      "username": "${ERP_USERNAME}",
      "password": "${ERP_PASSWORD}"
    }
  },
  "refresh_intervals": {
    "warehouse_operations_minutes": 15,
    "fleet_trips_minutes": 15,
    "gravity_recalc_hours": 24
  },
  "alert_thresholds": {
    "fleet_idle_percent": 15,
    "dwell_time_hours": 72,
    "cross_dock_minutes": 120
  }
}
```

### Step 5: Deploy Power BI Template

1. Open Power BI Desktop
2. File → Import → Power BI Template
3. Select `LogiFleet_Pulse_Master.pbit` from the repository
4. Enter connection parameters:
   - SQL Server instance name
   - Database name: `LogiFleetPulse`
   - Authentication mode (Windows or SQL Server)

The template includes:

- **Executive Overview Dashboard**: High-level KPIs across all domains
- **Warehouse Operations View**: Dwell time, zone utilization, gravity zone recommendations
- **Fleet Performance View**: Fuel efficiency, idle time, maintenance priority queue
- **Cross-Fact Analysis**: Combined warehouse + fleet metrics
- **Predictive Bottleneck Heatmap**: Temporal elasticity modeling results

## Key SQL Patterns & Queries

### Cross-Fact KPI: Dwell Time vs. Fleet Idle Time

```sql
-- Find correlation between warehouse dwell time and downstream fleet delays
WITH WarehouseDwell AS (
    SELECT 
        p.SKU,
        p.ProductName,
        AVG(wo.DwellTimeMinutes) AS AvgDwellMinutes,
        COUNT(*) AS TotalOperations
    FROM FactWarehouseOperations wo
    JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.[Date] >= DATEADD(DAY, -30, GETDATE())
        AND wo.OperationType = 'Picking'
    GROUP BY p.SKU, p.ProductName
),
FleetIdle AS (
    SELECT 
        p.SKU,
        AVG(ft.IdleTimeMinutes) AS AvgIdleMinutes,
        COUNT(*) AS TotalTrips
    FROM FactFleetTrips ft
    JOIN FactWarehouseOperations wo ON ft.RouteID = wo.OrderID
    JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    JOIN DimTime t ON ft.TimeKey = t.TimeKey
    WHERE t.[Date] >= DATEADD(DAY, -30, GETDATE())
    GROUP BY p.SKU
)
SELECT 
    wd.SKU,
    wd.ProductName,
    wd.AvgDwellMinutes,
    fi.AvgIdleMinutes,
    (fi.AvgIdleMinutes / NULLIF(wd.AvgDwellMinutes, 0)) AS DwellToIdleRatio,
    wd.TotalOperations,
    fi.TotalTrips
FROM WarehouseDwell wd
LEFT JOIN FleetIdle fi ON wd.SKU = fi.SKU
WHERE fi.AvgIdleMinutes > 0
ORDER BY DwellToIdleRatio DESC;
```

### Adaptive Fleet Triage Engine

```sql
-- Generate maintenance priority queue based on telemetry + revenue impact
WITH VehicleIssues AS (
    SELECT 
        v.VehicleID,
        v.VehicleType,
        DATEDIFF(DAY, v.LastMaintenanceDate, GETDATE()) AS DaysSinceLastMaintenance,
        DATEDIFF(DAY, GETDATE(), v.NextScheduledMaintenanceDate) AS DaysUntilScheduled,
        AVG(ft.LoadWeightKg * p.UnitPrice) AS AvgRevenuePerTrip,
        SUM(CASE WHEN ft.MaintenanceEventFlag = 1 THEN 1 ELSE 0 END) AS RecentIssueCount
    FROM DimFleetVehicle v
    JOIN FactFleetTrips ft ON v.VehicleKey = ft.VehicleKey
    JOIN FactWarehouseOperations wo ON ft.RouteID = wo.OrderID
    JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    JOIN DimTime t ON ft.TimeKey = t.TimeKey
    WHERE t.[Date] >= DATEADD(DAY, -90, GETDATE())
    GROUP BY v.VehicleID, v.VehicleType, v.LastMaintenanceDate, v.NextScheduledMaintenanceDate
)
SELECT 
    VehicleID,
    VehicleType,
    DaysSinceLastMaintenance,
    DaysUntilScheduled,
    AvgRevenuePerTrip,
    RecentIssueCount,
    -- Weighted priority score
    (
        (DaysSinceLastMaintenance * 0.3) + 
        (CASE WHEN DaysUntilScheduled < 0 THEN ABS(DaysUntilScheduled) * 2 ELSE 0 END * 0.4) +
        (AvgRevenuePerTrip / 1000 * 0.2) +
        (RecentIssueCount * 5 * 0.1)
    ) AS PriorityScore
FROM VehicleIssues
ORDER BY PriorityScore DESC;
```

### Warehouse Gravity Zone Optimization Report

```sql
-- Identify products that should be moved to different zones
SELECT 
    p.SKU,
    p.ProductName,
    p.RecommendedZone,
    wo.AssignedZone AS CurrentZone,
    p.GravityScore,
    AVG(wo.DwellTimeMinutes) AS AvgDwellMinutes,
    AVG(wo.ProcessingTimeMinutes) AS AvgProcessingMinutes,
    COUNT(*) AS PicksLast30Days,
    CASE 
        WHEN p.RecommendedZone != wo.AssignedZone THEN 'REALLOCATE'
        ELSE 'OK'
    END AS Action
FROM DimProductGravity p
JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE t.[Date] >= DATEADD(DAY, -30, GETDATE())
    AND wo.OperationType = 'Picking'
    AND p.IsActive = 1
GROUP BY p.SKU, p.ProductName, p.RecommendedZone, wo.AssignedZone, p.GravityScore
HAVING COUNT(*) > 10 -- Only consider products with sufficient pick volume
ORDER BY 
    CASE WHEN p.RecommendedZone != wo.AssignedZone THEN 0 ELSE 1 END,
    p.GravityScore DESC;
```

### Predictive Bottleneck Detection

```sql
-- Temporal elasticity: predict congestion based on historical patterns
WITH HourlyPatterns AS (
    SELECT 
        t.[Hour],
        t.DayOfWeek,
        COUNT(*) AS TotalOperations,
        AVG(wo.ProcessingTimeMinutes) AS AvgProcessingMinutes,
        STDEV(wo.ProcessingTimeMinutes) AS StdDevProcessing
    FROM FactWarehouseOperations wo
    JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.[Date] >= DATEADD(DAY, -90, GETDATE())
        AND wo.OperationType IN ('Picking', 'Packing')
    GROUP BY t.[Hour], t.DayOfWeek
),
CurrentCapacity AS (
    SELECT 
        COUNT(DISTINCT wo.OperatorID) AS ActiveOperators,
        COUNT(DISTINCT wo.EquipmentID) AS ActiveEquipment
    FROM FactWarehouseOperations wo
    JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(HOUR, -1, GETUTCDATE())
)
SELECT 
    hp.[Hour],
    CASE hp.DayOfWeek
        WHEN 1 THEN 'Sunday'
        WHEN 2 THEN 'Monday'
        WHEN 3 THEN 'Tuesday'
        WHEN 4 THEN 'Wednesday'
        WHEN 5 THEN 'Thursday'
        WHEN 6 THEN 'Friday'
        WHEN 7 THEN 'Saturday'
    END AS DayName,
    hp.TotalOperations,
    hp.AvgProcessingMinutes,
    hp.StdDevProcessing,
    cc.ActiveOperators,
    cc.ActiveEquipment,
    -- Bottleneck risk score
    CASE 
        WHEN hp.TotalOperations > 200 AND hp.AvgProcessingMinutes > 8 THEN 'HIGH RISK'
        WHEN hp.TotalOperations > 150 AND hp.AvgProcessingMinutes > 6 THEN 'MEDIUM RISK'
        ELSE 'LOW RISK'
    END AS BottleneckRisk
FROM HourlyPatterns hp
CROSS JOIN CurrentCapacity cc
ORDER BY hp.TotalOperations DESC;
```

## Incremental Data Loading

### ETL Stored Procedure for Warehouse Operations

```sql
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadTimestamp DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new records from staging table (populated via SSIS, Polybase, or bulk insert)
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, SupplierKey,
        OperationType, OrderID, QuantityUnits, QuantityWeight,
        DwellTimeMinutes, ProcessingTimeMinutes, AssignedZone,
        OperatorID, EquipmentID, ExceptionFlag, ExceptionReason, TimestampUTC
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.SupplierKey,
        stg.OperationType,
        stg.OrderID,
        stg.QuantityUnits,
        stg.QuantityWeight,
        stg.DwellTimeMinutes,
        stg.ProcessingTimeMinutes,
        stg.AssignedZone,
        stg.OperatorID,
        stg.EquipmentID,
        stg.ExceptionFlag,
        stg.ExceptionReason,
        stg.TimestampUTC
    FROM StagingWarehouseOperations stg
    JOIN DimTime t ON stg.OperationDateTime = t.FullDateTime
    JOIN DimGeography g ON stg.LocationID = g.LocationID
    JOIN DimProductGravity p ON stg.SKU = p.SKU
    LEFT JOIN DimSupplierReliability s ON stg.SupplierID = s.SupplierID
    WHERE stg.TimestampUTC > @LastLoadTimestamp;
    
    PRINT 'Loaded ' + CAST(@@ROWCOUNT AS VARCHAR) + ' new warehouse operations';
END
GO
```

## Power BI DAX Measures

### Key Performance Indicators

```dax
-- Average Warehouse Dwell Time (Hours)
AvgDwellTimeHours = 
AVERAGEX(
    FactWarehouseOperations,
    FactWarehouseOperations[DwellTimeMinutes] / 60
)

-- Fleet Utilization Percentage
FleetUtilization = 
DIVIDE(
    SUM(FactFleetTrips[ActualDurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[ActualDurationMinutes]),
    0
) * 100

-- Fuel Efficiency (km per liter)
FuelEfficiency = 
DIVIDE(
    SUM(FactFleetTrips[ActualDistanceKm]),
    SUM(FactFleetTrips[FuelConsumedLiters]),
    0
)

-- Cross-Fact: Dwell Impact on Delivery Delay
DwellImpactOnDelivery = 
VAR AvgDwell = [AvgDwellTimeHours]
VAR AvgDelay = 
    AVERAGEX(
        FactFleetTrips,
        FactFleetTrips[ActualDurationMinutes] - FactFleetTrips[PlannedDurationMinutes]
    )
RETURN
    IF(AvgDwell > 72, AvgDelay * 1.2, AvgDelay) -- Penalize long dwell times

-- Gravity Zone Compliance Rate
GravityZoneCompliance = 
VAR Compliant = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[AssignedZone] = RELATED(DimProductGravity[RecommendedZone])
    )
VAR Total = COUNTROWS(FactWarehouseOperations)
RETURN
    DIVIDE(Compliant, Total, 0) * 100
```

## Automated Alerting

### SQL Agent Job for Threshold Monitoring

```sql
CREATE PROCEDURE sp_CheckAlertThresholds
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    -- Check fleet idle time threshold
    IF EXISTS (
        SELECT 1
        FROM FactFleetTrips ft
        JOIN DimTime t ON ft.TimeKey = t.TimeKey
        WHERE t.[Date] = CAST(GETDATE() AS DATE)
        GROUP BY ft.VehicleKey
        HAVING (SUM(ft.IdleTimeMinutes) / NULLIF(SUM(ft.ActualDurationMinutes), 0)) * 100 > 15
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Fleet idle time exceeded 15% threshold for one or more vehicles today.';
        -- Send email via sp_send_dbmail or log to alerting table
        PRINT @AlertMessage;
    END
    
    -- Check warehouse dwell time threshold
    IF EXISTS (
        SELECT 1
        FROM FactWarehouseOperations wo
        JOIN DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.[Date] >= DATEADD(DAY, -7, GETDATE())
            AND wo.DwellTimeMinutes > (72 * 60)
    )
    BEGIN
        SET @AlertMessage = 'ALERT: One or more products exceeded 72-hour dwell time in the past week.';
        PRINT @AlertMessage;
    END
    
    -- Check cross-dock duration threshold
    IF EXISTS (
        SELECT 1
        FROM FactCrossDock cd
        JOIN DimTime t ON cd.TimeKey = t.TimeKey
        WHERE t.[Date] = CAST(GETDATE() AS DATE)
            AND cd.CrossDockDurationMinutes > 120
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Cross-dock duration exceeded 120 minutes threshold today.';
        PRINT @AlertMessage;
    END
END
GO

-- Schedule this procedure via SQL Server Agent (runs every 15 minutes)
```

## Configuration Best Practices

### Indexing Strategy

```sql
-- Add covering indexes for common queries
CREATE NONCLUSTERED INDEX IX_FactWarehouseOps_Tim
