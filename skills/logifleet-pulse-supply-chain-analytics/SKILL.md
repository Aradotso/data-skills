---
name: logifleet-pulse-supply-chain-analytics
description: Multi-modal logistics intelligence engine combining MS SQL Server data warehousing with Power BI dashboards for warehouse, fleet, and supply chain analytics
triggers:
  - "set up logistics analytics dashboard"
  - "create supply chain data warehouse"
  - "implement warehouse and fleet tracking system"
  - "build Power BI logistics dashboard"
  - "design multi-fact star schema for supply chain"
  - "configure real-time fleet monitoring"
  - "optimize warehouse gravity zones"
  - "implement cross-modal supply chain analytics"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive logistics intelligence platform that unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a single semantic layer. Built on MS SQL Server with Power BI visualization, it provides:

- **Multi-fact star schema** with time-phased dimensions
- **Cross-fact KPI harmonization** linking warehouse and fleet metrics
- **Real-time dashboards** with 15-minute refresh cycles
- **Predictive bottleneck detection** using historical patterns
- **Warehouse Gravity Zones™** for optimal spatial organization
- **Adaptive fleet triage** based on weighted telemetry scoring

The platform is designed for 3PL operators, retail chains, food distributors, and any organization managing complex supply chain operations.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- Access to data sources: WMS, TMS, GPS/telematics, ERP systems
- Optional: Azure Synapse Analytics for big data enrichment

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Create the LogiFleet database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Deploy the core dimension tables
-- DimTime: 15-minute granularity time dimension
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    [Date] DATE NOT NULL,
    [Year] INT NOT NULL,
    Quarter INT NOT NULL,
    [Month] INT NOT NULL,
    [Week] INT NOT NULL,
    [Day] INT NOT NULL,
    [Hour] INT NOT NULL,
    [Minute] INT NOT NULL,
    DayOfWeek INT NOT NULL,
    DayName VARCHAR(10) NOT NULL,
    IsWeekend BIT NOT NULL,
    FiscalYear INT NOT NULL,
    FiscalQuarter INT NOT NULL,
    FiscalPeriod INT NOT NULL
);
GO

-- Create clustered columnstore index for optimal performance
CREATE CLUSTERED COLUMNSTORE INDEX IX_DimTime_CCS ON DimTime;
GO

-- DimGeography: Hierarchical geographic dimension
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    NodeID VARCHAR(50) UNIQUE NOT NULL,
    NodeType VARCHAR(20) NOT NULL, -- 'Warehouse', 'RouteNode', 'CustomerSite'
    NodeName VARCHAR(200) NOT NULL,
    Address VARCHAR(500),
    City VARCHAR(100),
    Region VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone VARCHAR(50),
    IsActive BIT DEFAULT 1
);
GO

CREATE NONCLUSTERED INDEX IX_Geography_Type ON DimGeography(NodeType) INCLUDE (Country, Region);
GO

-- DimProduct: Product hierarchy with gravity scoring
CREATE TABLE DimProduct (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName VARCHAR(200) NOT NULL,
    Category VARCHAR(100),
    Subcategory VARCHAR(100),
    Brand VARCHAR(100),
    UnitWeight DECIMAL(10,3), -- kg
    UnitVolume DECIMAL(10,3), -- cubic meters
    IsFragile BIT DEFAULT 0,
    IsPerishable BIT DEFAULT 0,
    RequiresRefrigeration BIT DEFAULT 0,
    UnitValue DECIMAL(10,2), -- USD
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value / (fragility_factor)
    OptimalStorageZone VARCHAR(50),
    LastGravityUpdate DATETIME
);
GO

CREATE NONCLUSTERED INDEX IX_Product_Gravity ON DimProduct(GravityScore DESC, IsPerishable, IsFragile);
GO

-- DimSupplier: Supplier reliability tracking
CREATE TABLE DimSupplier (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) UNIQUE NOT NULL,
    SupplierName VARCHAR(200) NOT NULL,
    Country VARCHAR(100),
    AvgLeadTimeDays DECIMAL(5,1),
    LeadTimeVariance DECIMAL(5,2), -- Standard deviation
    DefectRate DECIMAL(5,4), -- Percentage
    OnTimeDeliveryRate DECIMAL(5,4), -- Percentage
    ComplianceScore DECIMAL(5,2), -- 0-100
    ReliabilityTier VARCHAR(10), -- 'A', 'B', 'C', 'D'
    LastScoreUpdate DATETIME
);
GO

-- DimVehicle: Fleet vehicle master
CREATE TABLE DimVehicle (
    VehicleKey INT PRIMARY KEY IDENTITY(1,1),
    VehicleID VARCHAR(50) UNIQUE NOT NULL,
    LicensePlate VARCHAR(20),
    VehicleType VARCHAR(50), -- 'Truck', 'Van', 'Refrigerated', 'Flatbed'
    Make VARCHAR(50),
    Model VARCHAR(50),
    [Year] INT,
    MaxCapacityKg DECIMAL(10,2),
    MaxCapacityCubicMeters DECIMAL(10,2),
    FuelType VARCHAR(20), -- 'Diesel', 'Electric', 'Hybrid'
    AvgFuelConsumption DECIMAL(5,2), -- L/100km or kWh/100km
    LastMaintenanceDate DATE,
    NextMaintenanceDue DATE,
    IsActive BIT DEFAULT 1
);
GO

-- FactWarehouseOperations: Core warehouse activity fact
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(20) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OperationID VARCHAR(50) NOT NULL,
    Quantity INT NOT NULL,
    UnitWeight DECIMAL(10,3),
    StorageZone VARCHAR(50),
    DwellTimeMinutes INT, -- Time in current zone
    OperatorID VARCHAR(50),
    EquipmentID VARCHAR(50), -- Forklift, pallet jack, etc.
    IsComplete BIT DEFAULT 0,
    QualityFlag VARCHAR(20), -- 'OK', 'Damaged', 'Expired', 'Quarantine'
    
    CONSTRAINT FK_WarehouseOps_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WarehouseOps_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_WarehouseOps_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);
GO

CREATE NONCLUSTERED INDEX IX_WarehouseOps_Time ON FactWarehouseOperations(TimeKey, OperationType);
CREATE NONCLUSTERED INDEX IX_WarehouseOps_Product ON FactWarehouseOperations(ProductKey, TimeKey);
CREATE NONCLUSTERED INDEX IX_WarehouseOps_Dwell ON FactWarehouseOperations(DwellTimeMinutes DESC) WHERE DwellTimeMinutes > 60;
GO

-- FactFleetTrips: Fleet movement and telemetry fact
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    TripID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50),
    DistanceKm DECIMAL(10,2),
    DurationMinutes INT,
    IdleTimeMinutes INT, -- Engine running, not moving
    FuelConsumed DECIMAL(10,2), -- Liters or kWh
    AvgSpeed DECIMAL(5,1), -- km/h
    MaxSpeed DECIMAL(5,1),
    LoadWeightKg DECIMAL(10,2),
    LoadVolumePercent DECIMAL(5,2), -- Percentage of capacity
    NumStops INT,
    TotalLoadingTimeMinutes INT,
    TotalUnloadingTimeMinutes INT,
    WeatherCondition VARCHAR(50), -- 'Clear', 'Rain', 'Snow', 'Fog'
    TrafficDelayMinutes INT,
    TripStatus VARCHAR(20), -- 'Completed', 'InProgress', 'Delayed', 'Cancelled'
    
    CONSTRAINT FK_FleetTrips_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FleetTrips_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_FleetTrips_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FleetTrips_Dest FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
);
GO

CREATE NONCLUSTERED INDEX IX_FleetTrips_Time ON FactFleetTrips(TimeKey, TripStatus);
CREATE NONCLUSTERED INDEX IX_FleetTrips_Vehicle ON FactFleetTrips(VehicleKey, TimeKey);
CREATE NONCLUSTERED INDEX IX_FleetTrips_Idle ON FactFleetTrips(IdleTimeMinutes DESC) WHERE IdleTimeMinutes > 30;
GO

-- FactCrossDock: Cross-dock transfer operations
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    SupplierKey INT NOT NULL,
    InboundTripID VARCHAR(50),
    OutboundTripID VARCHAR(50),
    Quantity INT NOT NULL,
    DockDwellMinutes INT, -- Time between inbound and outbound
    IsDirectTransfer BIT DEFAULT 0, -- True if no floor storage
    
    CONSTRAINT FK_CrossDock_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_CrossDock_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_CrossDock_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_CrossDock_Supplier FOREIGN KEY (SupplierKey) REFERENCES DimSupplier(SupplierKey)
);
GO

CREATE NONCLUSTERED INDEX IX_CrossDock_Time ON FactCrossDock(TimeKey);
CREATE NONCLUSTERED INDEX IX_CrossDock_Dwell ON FactCrossDock(DockDwellMinutes) WHERE IsDirectTransfer = 0;
GO
```

### Step 2: Create Key Views and Stored Procedures

```sql
-- View: Unified logistics performance (cross-fact KPI)
CREATE VIEW vw_UnifiedLogisticsPerformance AS
SELECT 
    t.[Date],
    t.[Year],
    t.[Month],
    g.Country,
    g.Region,
    p.Category AS ProductCategory,
    
    -- Warehouse metrics
    SUM(CASE WHEN wo.OperationType = 'Picking' THEN wo.Quantity ELSE 0 END) AS TotalPickedUnits,
    AVG(CASE WHEN wo.OperationType = 'Picking' THEN wo.DwellTimeMinutes ELSE NULL END) AS AvgPickDwellMinutes,
    
    -- Fleet metrics
    SUM(ft.DistanceKm) AS TotalDistanceKm,
    SUM(ft.FuelConsumed) AS TotalFuelConsumed,
    SUM(ft.IdleTimeMinutes) AS TotalIdleMinutes,
    AVG(ft.LoadVolumePercent) AS AvgLoadUtilization,
    
    -- Cross-dock metrics
    AVG(cd.DockDwellMinutes) AS AvgCrossDockDwellMinutes,
    SUM(CASE WHEN cd.IsDirectTransfer = 1 THEN cd.Quantity ELSE 0 END) AS DirectTransferUnits,
    
    -- Composite KPI: Efficiency score
    (SUM(ft.DistanceKm) / NULLIF(SUM(ft.FuelConsumed), 0)) * 
    (100 - AVG(ft.IdleTimeMinutes * 100.0 / NULLIF(ft.DurationMinutes, 0))) / 100.0 AS FleetEfficiencyIndex
    
FROM DimTime t
LEFT JOIN FactWarehouseOperations wo ON t.TimeKey = wo.TimeKey
LEFT JOIN FactFleetTrips ft ON t.TimeKey = ft.TimeKey
LEFT JOIN FactCrossDock cd ON t.TimeKey = cd.TimeKey
LEFT JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey OR ft.OriginGeographyKey = g.GeographyKey
LEFT JOIN DimProduct p ON wo.ProductKey = p.ProductKey OR cd.ProductKey = p.ProductKey

WHERE t.[Date] >= DATEADD(MONTH, -12, GETDATE())

GROUP BY 
    t.[Date], t.[Year], t.[Month],
    g.Country, g.Region, p.Category;
GO

-- Stored Procedure: Calculate and update product gravity scores
CREATE PROCEDURE usp_UpdateProductGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    WITH ProductVelocity AS (
        SELECT 
            ProductKey,
            COUNT(*) AS PickFrequency,
            SUM(Quantity) AS TotalUnitsLast30Days
        FROM FactWarehouseOperations
        WHERE OperationType = 'Picking'
          AND TimeKey >= (SELECT MIN(TimeKey) FROM DimTime WHERE [Date] >= DATEADD(DAY, -30, GETDATE()))
        GROUP BY ProductKey
    ),
    GravityCalculation AS (
        SELECT 
            p.ProductKey,
            p.SKU,
            COALESCE(pv.PickFrequency, 0) AS PickFrequency,
            p.UnitValue,
            CASE WHEN p.IsFragile = 1 THEN 1.5 ELSE 1.0 END AS FragilityFactor,
            CASE WHEN p.IsPerishable = 1 THEN 1.3 ELSE 1.0 END AS PerishabilityFactor,
            
            -- Gravity Score = (Frequency * Value) / (Fragility * Perishability)
            (COALESCE(pv.PickFrequency, 0) * p.UnitValue) / 
            (CASE WHEN p.IsFragile = 1 THEN 1.5 ELSE 1.0 END * 
             CASE WHEN p.IsPerishable = 1 THEN 1.3 ELSE 1.0 END) AS CalculatedGravityScore,
            
            -- Optimal storage zone based on gravity tiers
            CASE 
                WHEN (COALESCE(pv.PickFrequency, 0) * p.UnitValue) / 
                     (CASE WHEN p.IsFragile = 1 THEN 1.5 ELSE 1.0 END * 
                      CASE WHEN p.IsPerishable = 1 THEN 1.3 ELSE 1.0 END) > 1000 THEN 'HotZone-A'
                WHEN (COALESCE(pv.PickFrequency, 0) * p.UnitValue) / 
                     (CASE WHEN p.IsFragile = 1 THEN 1.5 ELSE 1.0 END * 
                      CASE WHEN p.IsPerishable = 1 THEN 1.3 ELSE 1.0 END) > 500 THEN 'WarmZone-B'
                WHEN (COALESCE(pv.PickFrequency, 0) * p.UnitValue) / 
                     (CASE WHEN p.IsFragile = 1 THEN 1.5 ELSE 1.0 END * 
                      CASE WHEN p.IsPerishable = 1 THEN 1.3 ELSE 1.0 END) > 100 THEN 'CoolZone-C'
                ELSE 'ColdZone-D'
            END AS OptimalZone
        FROM DimProduct p
        LEFT JOIN ProductVelocity pv ON p.ProductKey = pv.ProductKey
    )
    UPDATE p
    SET 
        GravityScore = gc.CalculatedGravityScore,
        OptimalStorageZone = gc.OptimalZone,
        LastGravityUpdate = GETDATE()
    FROM DimProduct p
    INNER JOIN GravityCalculation gc ON p.ProductKey = gc.ProductKey;
    
    PRINT 'Product gravity scores updated successfully.';
END;
GO

-- Stored Procedure: Fleet triage scoring (proactive maintenance)
CREATE PROCEDURE usp_GenerateFleetTriageQueue
    @MinPriorityScore DECIMAL(5,2) = 50.0
AS
BEGIN
    SET NOCOUNT ON;
    
    WITH RecentTripMetrics AS (
        SELECT 
            VehicleKey,
            AVG(IdleTimeMinutes * 100.0 / NULLIF(DurationMinutes, 0)) AS AvgIdlePercent,
            AVG(FuelConsumed / NULLIF(DistanceKm, 0)) AS AvgFuelPerKm,
            SUM(CASE WHEN LoadVolumePercent > 0 THEN 1 ELSE 0 END) AS LoadedTrips,
            AVG(LoadWeightKg) AS AvgLoadWeight
        FROM FactFleetTrips
        WHERE TimeKey >= (SELECT MIN(TimeKey) FROM DimTime WHERE [Date] >= DATEADD(DAY, -14, GETDATE()))
        GROUP BY VehicleKey
    ),
    HighValueCargo AS (
        SELECT 
            ft.VehicleKey,
            SUM(p.UnitValue * wo.Quantity) AS TotalCargoValue
        FROM FactFleetTrips ft
        INNER JOIN FactWarehouseOperations wo ON ft.TripID = wo.OperationID
        INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
        WHERE ft.TimeKey >= (SELECT MIN(TimeKey) FROM DimTime WHERE [Date] >= DATEADD(DAY, -7, GETDATE()))
        GROUP BY ft.VehicleKey
    ),
    TriageScoring AS (
        SELECT 
            v.VehicleID,
            v.VehicleType,
            v.LicensePlate,
            DATEDIFF(DAY, v.LastMaintenanceDate, GETDATE()) AS DaysSinceLastMaintenance,
            DATEDIFF(DAY, GETDATE(), v.NextMaintenanceDue) AS DaysUntilNextMaintenance,
            rtm.AvgIdlePercent,
            rtm.AvgFuelPerKm,
            COALESCE(hvc.TotalCargoValue, 0) AS RecentCargoValue,
            
            -- Priority Score = (Maintenance Urgency) + (Operational Impact) + (Cargo Risk)
            (CASE 
                WHEN DATEDIFF(DAY, GETDATE(), v.NextMaintenanceDue) < 0 THEN 40 -- Overdue
                WHEN DATEDIFF(DAY, GETDATE(), v.NextMaintenanceDue) <= 7 THEN 30 -- Due soon
                WHEN DATEDIFF(DAY, GETDATE(), v.NextMaintenanceDue) <= 14 THEN 20
                ELSE 10
            END) +
            (CASE 
                WHEN rtm.AvgIdlePercent > 25 THEN 25 -- High idle time
                WHEN rtm.AvgIdlePercent > 15 THEN 15
                ELSE 5
            END) +
            (CASE 
                WHEN COALESCE(hvc.TotalCargoValue, 0) > 100000 THEN 35 -- High-value cargo
                WHEN COALESCE(hvc.TotalCargoValue, 0) > 50000 THEN 20
                WHEN COALESCE(hvc.TotalCargoValue, 0) > 10000 THEN 10
                ELSE 0
            END) AS TriagePriorityScore
            
        FROM DimVehicle v
        LEFT JOIN RecentTripMetrics rtm ON v.VehicleKey = rtm.VehicleKey
        LEFT JOIN HighValueCargo hvc ON v.VehicleKey = hvc.VehicleKey
        WHERE v.IsActive = 1
    )
    SELECT 
        VehicleID,
        VehicleType,
        LicensePlate,
        DaysSinceLastMaintenance,
        DaysUntilNextMaintenance,
        CAST(AvgIdlePercent AS DECIMAL(5,2)) AS AvgIdlePercent,
        CAST(AvgFuelPerKm AS DECIMAL(5,3)) AS AvgFuelPerKm,
        RecentCargoValue,
        TriagePriorityScore,
        CASE 
            WHEN TriagePriorityScore >= 80 THEN 'CRITICAL'
            WHEN TriagePriorityScore >= 60 THEN 'HIGH'
            WHEN TriagePriorityScore >= 40 THEN 'MEDIUM'
            ELSE 'LOW'
        END AS PriorityLevel
    FROM TriageScoring
    WHERE TriagePriorityScore >= @MinPriorityScore
    ORDER BY TriagePriorityScore DESC;
END;
GO
```

### Step 3: Configure Data Ingestion

```sql
-- Example: Create external table for real-time WMS feed (requires PolyBase)
-- First, create master key and database scoped credential
CREATE MASTER KEY ENCRYPTION BY PASSWORD = '${SQL_MASTER_KEY_PASSWORD}';
GO

CREATE DATABASE SCOPED CREDENTIAL WMS_API_Credential
WITH IDENTITY = 'API_USER', 
     SECRET = '${WMS_API_SECRET}';
GO

-- Create external data source (example: REST API via PolyBase)
CREATE EXTERNAL DATA SOURCE WMS_LiveFeed
WITH (
    TYPE = HADOOP, -- Use appropriate type for your source
    LOCATION = '${WMS_API_ENDPOINT}',
    CREDENTIAL = WMS_API_Credential
);
GO

-- For file-based ingestion (CSV from S3, Azure Blob, etc.)
-- Bulk insert example
BULK INSERT FactWarehouseOperations
FROM '${WAREHOUSE_DATA_PATH}/daily_operations.csv'
WITH (
    FIELDTERMINATOR = ',',
    ROWTERMINATOR = '\n',
    FIRSTROW = 2,
    ERRORFILE = '${ERROR_LOG_PATH}/warehouse_errors.txt',
    TABLOCK
);
GO
```

### Step 4: Set Up Automated Alerts

```sql
-- Create alert threshold table
CREATE TABLE AlertThresholds (
    AlertID INT PRIMARY KEY IDENTITY(1,1),
    MetricName VARCHAR(100) NOT NULL,
    ThresholdValue DECIMAL(10,2),
    ThresholdOperator VARCHAR(10), -- '>', '<', '=', '>='
    AlertLevel VARCHAR(20), -- 'INFO', 'WARNING', 'CRITICAL'
    IsActive BIT DEFAULT 1
);
GO

INSERT INTO AlertThresholds (MetricName, ThresholdValue, ThresholdOperator, AlertLevel)
VALUES 
    ('FleetIdleTimePercent', 15.0, '>', 'WARNING'),
    ('WarehouseDwellTimeHours', 72.0, '>', 'CRITICAL'),
    ('FuelConsumptionVariance', 20.0, '>', 'WARNING'),
    ('CrossDockDwellMinutes', 120.0, '>', 'WARNING');
GO

-- Stored procedure to check thresholds and log alerts
CREATE PROCEDURE usp_CheckAlertThresholds
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Create alert log table if not exists
    IF NOT EXISTS (SELECT * FROM sys.tables WHERE name = 'AlertLog')
    BEGIN
        CREATE TABLE AlertLog (
            LogID BIGINT PRIMARY KEY IDENTITY(1,1),
            AlertTime DATETIME DEFAULT GETDATE(),
            MetricName VARCHAR(100),
            ActualValue DECIMAL(10,2),
            ThresholdValue DECIMAL(10,2),
            AlertLevel VARCHAR(20),
            AlertMessage NVARCHAR(500)
        );
    END
    
    -- Check fleet idle time
    INSERT INTO AlertLog (MetricName, ActualValue, ThresholdValue, AlertLevel, AlertMessage)
    SELECT 
        'FleetIdleTimePercent',
        AVG(IdleTimeMinutes * 100.0 / NULLIF(DurationMinutes, 0)) AS ActualIdlePercent,
        at.ThresholdValue,
        at.AlertLevel,
        'Fleet average idle time exceeded threshold: ' + 
        CAST(AVG(IdleTimeMinutes * 100.0 / NULLIF(DurationMinutes, 0)) AS VARCHAR(10)) + '%'
    FROM FactFleetTrips ft
    CROSS JOIN AlertThresholds at
    WHERE at.MetricName = 'FleetIdleTimePercent'
      AND at.IsActive = 1
      AND ft.TimeKey >= (SELECT MIN(TimeKey) FROM DimTime WHERE [Date] >= CAST(GETDATE() AS DATE))
    HAVING AVG(IdleTimeMinutes * 100.0 / NULLIF(DurationMinutes, 0)) > at.ThresholdValue;
    
    -- Check warehouse dwell time
    INSERT INTO AlertLog (MetricName, ActualValue, ThresholdValue, AlertLevel, AlertMessage)
    SELECT 
        'WarehouseDwellTimeHours',
        AVG(DwellTimeMinutes / 60.0) AS ActualDwellHours,
        at.ThresholdValue,
        at.AlertLevel,
        'Warehouse dwell time exceeded threshold: ' + 
        CAST(AVG(DwellTimeMinutes / 60.0) AS VARCHAR(10)) + ' hours'
    FROM FactWarehouseOperations wo
    CROSS JOIN AlertThresholds at
    WHERE at.MetricName = 'WarehouseDwellTimeHours'
      AND at.IsActive = 1
      AND wo.TimeKey >= (SELECT MIN(TimeKey) FROM DimTime WHERE [Date] >= CAST(GETDATE() AS DATE))
      AND wo.IsComplete = 0
    HAVING AVG(DwellTimeMinutes / 60.0) > at.ThresholdValue;
    
    -- Return today's alerts
    SELECT * FROM AlertLog 
    WHERE AlertTime >= CAST(GETDATE() AS DATE)
    ORDER BY AlertTime DESC;
END;
GO

-- Schedule this procedure via SQL Server Agent job (run every 15 minutes)
```

## Power BI Configuration

### Step 1: Import Power BI Template

1. Open Power BI Desktop
2. File → Import → Power BI Template (`.pbit`)
3. Open `LogiFleet_Pulse_Master.pbit` from the repository
4. Enter connection parameters:
   - **Server**: Your SQL Server instance
   - **Database**: LogiFleetPulse
   - **Authentication**: Windows or SQL Server authentication

### Step 2: Key DAX Measures

```dax
// Measure: Fleet Efficiency Index
Fleet Efficiency Index = 
VAR TotalDistance = SUM(FactFleetTrips[DistanceKm])
VAR TotalFuel = SUM(FactFleetTrips[FuelConsumed])
VAR AvgIdlePercent = 
    AVERAGEX(
        FactFleetTrips,
        FactFleetTrips[IdleTimeMinutes] * 100 / FactFleetTrips[DurationMinutes]
    )
VAR FuelEfficiency = DIVIDE(TotalDistance, TotalFuel, 0)
VAR UtilizationFactor = (100 - AvgIdlePercent) / 100
RETURN FuelEfficiency * UtilizationFactor

// Measure: Warehouse Gravity Compliance Rate
Gravity Compliance Rate = 
VAR CurrentPlacements = 
    COUNTROWS(
        FILTER(
            FactWarehouseOperations,
            FactWarehouseOperations[StorageZone] = RELATED(DimProduct[OptimalStorageZone])
        )
    )
VAR TotalPlacements = COUNTROWS(FactWarehouseOperations)
RETURN DIVIDE(CurrentPlacements, TotalPlacements, 0) * 100

// Measure: Cross-Dock Velocity
Cross-Dock Velocity = 
AVERAGE(FactCrossDock[DockDwellMinutes])

// Measure: High-Priority Dwell Items (> 72 hours)
Critical Dwell Items = 
CALCULATE(
    DISTINCTCOUNT(FactWarehouseOperations[ProductKey]),
    FactWarehouseOperations[DwellTimeMinutes] > 4320, // 72 hours
    FactWarehouseOperations[IsComplete] = FALSE()
)

// Measure: Fleet Triage Score (rolling 7 days)
Fleet Triage Priority = 
VAR DaysOverdue = 
    DATEDIFF(
        MAX(DimVehicle[NextMaintenanceDue]),
        TODAY(),
        DAY
    )
VAR MaintenanceUrgency = 
    SWITCH(
        TRUE(),
        DaysOverdue < 0, 40,
        DaysOverdue <= 7, 30,
        DaysOverdue <= 14, 20,
        10
    )
VAR AvgIdlePercent = [Fleet Efficiency Index] // Reuse measure
VAR OperationalImpact = 
    SWITCH(
        TRUE(),
        AvgIdlePercent > 25, 25,
        AvgIdlePercent > 15, 15,
        5
    )
RETURN MaintenanceUrgency + OperationalImpact

// Calculated Column: Product Velocity Tier (in DimProduct)
Velocity Tier = 
VAR PicksLast30 = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        
