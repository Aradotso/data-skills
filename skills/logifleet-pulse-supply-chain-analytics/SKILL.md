---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server + Power BI logistics intelligence platform with multi-fact star schema for warehouse, fleet, and supply chain analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure warehouse and fleet data model"
  - "create Power BI logistics dashboard"
  - "implement multi-fact star schema for supply chain"
  - "build logistics KPI tracking system"
  - "deploy SQL Server warehouse analytics"
  - "integrate fleet telemetry with warehouse operations"
  - "optimize supply chain data warehousing"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive logistics intelligence platform that unifies warehouse operations, fleet management, and supply chain analytics into a single semantic data layer. Built on MS SQL Server with Power BI visualization, it uses a multi-fact star schema architecture to correlate cross-functional KPIs like warehouse dwell time, fleet fuel efficiency, and inventory turnover.

**Key Capabilities:**
- Multi-fact data warehousing with time-phased dimensions
- Real-time logistics dashboards (15-minute refresh)
- Cross-modal KPI harmonization (warehouse + fleet + external data)
- Predictive bottleneck detection
- Warehouse gravity zone optimization
- Fleet maintenance triage engine

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- Access to data sources: WMS, TMS, GPS/telematics feeds
- (Optional) Azure Synapse Analytics for external data enrichment

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Run the main schema deployment script

USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    Date DATE NOT NULL,
    FiscalYear INT,
    FiscalQuarter INT,
    FiscalMonth INT,
    DayOfWeek TINYINT,
    HourOfDay TINYINT,
    MinuteBucket TINYINT, -- 0, 15, 30, 45
    IsWeekend BIT,
    IsHoliday BIT
);

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) UNIQUE NOT NULL,
    LocationName NVARCHAR(200),
    LocationType VARCHAR(20), -- 'Warehouse', 'Route Node', 'Customer Site'
    Address NVARCHAR(500),
    City NVARCHAR(100),
    Region NVARCHAR(100),
    Country NVARCHAR(100),
    Continent NVARCHAR(50),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    TimeZone VARCHAR(50)
);

CREATE TABLE DimProduct (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName NVARCHAR(200),
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    UnitWeight DECIMAL(10,2), -- kg
    UnitVolume DECIMAL(10,3), -- cubic meters
    IsFragile BIT,
    RequiresRefrigeration BIT,
    StandardCost MONEY,
    ListPrice MONEY,
    GravityScore DECIMAL(5,2) -- Calculated: velocity * value * fragility
);

CREATE TABLE DimSupplier (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) UNIQUE NOT NULL,
    SupplierName NVARCHAR(200),
    Country NVARCHAR(100),
    LeadTimeDays INT,
    LeadTimeVariance DECIMAL(5,2),
    DefectRate DECIMAL(5,4),
    ComplianceScore DECIMAL(3,2), -- 0-1 scale
    ReliabilityTier VARCHAR(10) -- 'A', 'B', 'C'
);

CREATE TABLE DimFleet (
    FleetKey INT PRIMARY KEY IDENTITY(1,1),
    VehicleID VARCHAR(50) UNIQUE NOT NULL,
    VehicleType VARCHAR(50), -- 'Truck', 'Van', 'Refrigerated'
    Capacity DECIMAL(10,2), -- cubic meters
    MaxWeight DECIMAL(10,2), -- kg
    FuelType VARCHAR(20),
    YearManufactured INT,
    LastMaintenanceDate DATE,
    NextMaintenanceDue DATE,
    IsActive BIT
);

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProduct(ProductKey),
    SupplierKey INT FOREIGN KEY REFERENCES DimSupplier(SupplierKey),
    OperationType VARCHAR(20), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OperationStartTime DATETIME2,
    OperationEndTime DATETIME2,
    OperationDurationMinutes AS DATEDIFF(MINUTE, OperationStartTime, OperationEndTime),
    QuantityHandled INT,
    StorageZone VARCHAR(20), -- 'High-Gravity', 'Medium-Gravity', 'Low-Gravity'
    DwellTimeHours DECIMAL(10,2),
    WorkerID VARCHAR(50),
    EquipmentID VARCHAR(50)
);

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    FleetKey INT NOT NULL FOREIGN KEY REFERENCES DimFleet(FleetKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    TripStartTime DATETIME2,
    TripEndTime DATETIME2,
    TripDurationMinutes AS DATEDIFF(MINUTE, TripStartTime, TripEndTime),
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadWeightKG DECIMAL(10,2),
    LoadVolumeM3 DECIMAL(10,3),
    DriverID VARCHAR(50),
    WeatherCondition VARCHAR(50),
    TrafficDelayMinutes INT,
    OnTimeDelivery BIT
);

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProduct(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    ReceiptTime DATETIME2,
    ShipmentTime DATETIME2,
    DockDwellMinutes AS DATEDIFF(MINUTE, ReceiptTime, ShipmentTime),
    Quantity INT
);

-- Create indexes for performance
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Geography ON FactWarehouseOperations(GeographyKey);
CREATE NONCLUSTERED INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle ON FactFleetTrips(FleetKey);
CREATE NONCLUSTERED INDEX IX_FactFleet_Route ON FactFleetTrips(OriginGeographyKey, DestinationGeographyKey);
```

### Step 2: Create Data Loading Stored Procedures

```sql
-- Incremental load for warehouse operations
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Example: Load from external WMS system
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, SupplierKey,
        OperationType, OperationStartTime, OperationEndTime,
        QuantityHandled, StorageZone, DwellTimeHours, WorkerID, EquipmentID
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.SupplierKey,
        wms.OperationType,
        wms.StartTime,
        wms.EndTime,
        wms.Quantity,
        wms.Zone,
        wms.DwellHours,
        wms.WorkerID,
        wms.EquipmentID
    FROM WMS_ExternalTable wms
    INNER JOIN DimTime t ON CAST(wms.StartTime AS DATE) = t.Date 
        AND DATEPART(HOUR, wms.StartTime) = t.HourOfDay
        AND (DATEPART(MINUTE, wms.StartTime) / 15) * 15 = t.MinuteBucket
    INNER JOIN DimGeography g ON wms.WarehouseID = g.LocationID
    INNER JOIN DimProduct p ON wms.SKU = p.SKU
    LEFT JOIN DimSupplier s ON wms.SupplierID = s.SupplierID
    WHERE wms.StartTime > @LastLoadDateTime;
END;
GO

-- Fleet trips incremental load
CREATE PROCEDURE usp_LoadFleetTrips
    @LastLoadDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactFleetTrips (
        TimeKey, FleetKey, OriginGeographyKey, DestinationGeographyKey,
        TripStartTime, TripEndTime, DistanceKM, FuelConsumedLiters,
        IdleTimeMinutes, LoadWeightKG, LoadVolumeM3, DriverID,
        WeatherCondition, TrafficDelayMinutes, OnTimeDelivery
    )
    SELECT 
        t.TimeKey,
        f.FleetKey,
        orig.GeographyKey,
        dest.GeographyKey,
        tms.TripStart,
        tms.TripEnd,
        tms.Distance,
        tms.FuelUsed,
        tms.IdleMinutes,
        tms.LoadWeight,
        tms.LoadVolume,
        tms.DriverID,
        weather.Condition,
        traffic.DelayMinutes,
        CASE WHEN tms.ActualArrival <= tms.PromisedArrival THEN 1 ELSE 0 END
    FROM TMS_ExternalTable tms
    INNER JOIN DimTime t ON CAST(tms.TripStart AS DATE) = t.Date
        AND DATEPART(HOUR, tms.TripStart) = t.HourOfDay
        AND (DATEPART(MINUTE, tms.TripStart) / 15) * 15 = t.MinuteBucket
    INNER JOIN DimFleet f ON tms.VehicleID = f.VehicleID
    INNER JOIN DimGeography orig ON tms.OriginID = orig.LocationID
    INNER JOIN DimGeography dest ON tms.DestinationID = dest.LocationID
    LEFT JOIN Weather_API weather ON tms.TripStart = weather.Timestamp AND dest.LocationID = weather.LocationID
    LEFT JOIN Traffic_API traffic ON tms.TripStart = traffic.Timestamp AND tms.RouteID = traffic.RouteID
    WHERE tms.TripStart > @LastLoadDateTime;
END;
GO
```

### Step 3: Configure Power BI Connection

Create a `config.json` file (do not commit to version control):

```json
{
  "sql_server": {
    "server": "YOUR_SQL_SERVER_HOSTNAME",
    "database": "LogiFleetPulse",
    "authentication": "Windows",
    "connection_timeout": 30
  },
  "refresh_schedule": {
    "interval_minutes": 15,
    "off_peak_hours": [22, 23, 0, 1, 2, 3, 4, 5]
  },
  "alert_thresholds": {
    "dwell_time_hours": 72,
    "idle_time_percentage": 15,
    "fuel_efficiency_variance": 0.2,
    "on_time_delivery_percentage": 95
  }
}
```

### Step 4: Import Power BI Template

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. Enter your SQL Server connection details
3. Click "Load" to import the data model
4. Verify relationships are auto-detected:
   - FactWarehouseOperations → DimTime (TimeKey)
   - FactWarehouseOperations → DimProduct (ProductKey)
   - FactFleetTrips → DimFleet (FleetKey)
   - FactFleetTrips → DimGeography (OriginGeographyKey, DestinationGeographyKey)

## Key Analytics Patterns

### Cross-Fact KPI: Warehouse Dwell vs Fleet Idle

```sql
-- DAX Measure in Power BI
WarehouseDwellVsFleetIdle = 
VAR AvgDwell = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeHours]),
        FactWarehouseOperations[OperationType] = "Picking"
    )
VAR AvgIdlePercent = 
    CALCULATE(
        AVERAGE(FactFleetTrips[IdleTimeMinutes]) / AVERAGE(FactFleetTrips[TripDurationMinutes])
    )
RETURN
    AvgDwell * AvgIdlePercent -- Correlation metric
```

### Gravity Zone Optimization Query

```sql
-- SQL View for identifying misallocated products
CREATE VIEW vw_GravityZoneMismatches AS
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    wo.StorageZone,
    AVG(wo.OperationDurationMinutes) AS AvgPickTimeMinutes,
    COUNT(*) AS TotalOperations,
    CASE 
        WHEN p.GravityScore > 70 AND wo.StorageZone <> 'High-Gravity' THEN 'Move to High-Gravity'
        WHEN p.GravityScore BETWEEN 30 AND 70 AND wo.StorageZone NOT IN ('Medium-Gravity') THEN 'Move to Medium-Gravity'
        WHEN p.GravityScore < 30 AND wo.StorageZone <> 'Low-Gravity' THEN 'Move to Low-Gravity'
        ELSE 'Correctly Allocated'
    END AS Recommendation
FROM FactWarehouseOperations wo
INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
WHERE wo.OperationType = 'Picking'
    AND wo.OperationStartTime >= DATEADD(DAY, -30, GETDATE())
GROUP BY p.SKU, p.ProductName, p.GravityScore, wo.StorageZone
HAVING AVG(wo.OperationDurationMinutes) > 5; -- Only show if pick time is significant
```

### Fleet Maintenance Triage

```sql
-- Stored procedure to rank vehicles by maintenance urgency
CREATE PROCEDURE usp_FleetMaintenanceTriage
AS
BEGIN
    SELECT 
        f.VehicleID,
        f.VehicleType,
        f.NextMaintenanceDue,
        AVG(ft.FuelConsumedLiters / ft.DistanceKM) AS AvgFuelEfficiency,
        AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
        SUM(ft.LoadWeightKG * ft.DistanceKM) AS TotalRevenueKM, -- Proxy for revenue
        DATEDIFF(DAY, GETDATE(), f.NextMaintenanceDue) AS DaysUntilMaintenance,
        CASE 
            WHEN DATEDIFF(DAY, GETDATE(), f.NextMaintenanceDue) < 7 
                AND SUM(ft.LoadWeightKG * ft.DistanceKM) > 50000 THEN 'Critical'
            WHEN DATEDIFF(DAY, GETDATE(), f.NextMaintenanceDue) < 14 
                AND SUM(ft.LoadWeightKG * ft.DistanceKM) > 30000 THEN 'High'
            WHEN DATEDIFF(DAY, GETDATE(), f.NextMaintenanceDue) < 30 THEN 'Medium'
            ELSE 'Low'
        END AS UrgencyLevel
    FROM DimFleet f
    INNER JOIN FactFleetTrips ft ON f.FleetKey = ft.FleetKey
    WHERE f.IsActive = 1
        AND ft.TripStartTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY f.VehicleID, f.VehicleType, f.NextMaintenanceDue
    ORDER BY 
        CASE 
            WHEN DATEDIFF(DAY, GETDATE(), f.NextMaintenanceDue) < 7 
                AND SUM(ft.LoadWeightKG * ft.DistanceKM) > 50000 THEN 1
            WHEN DATEDIFF(DAY, GETDATE(), f.NextMaintenanceDue) < 14 
                AND SUM(ft.LoadWeightKG * ft.DistanceKM) > 30000 THEN 2
            WHEN DATEDIFF(DAY, GETDATE(), f.NextMaintenanceDue) < 30 THEN 3
            ELSE 4
        END;
END;
GO
```

### Predictive Bottleneck Detection

```sql
-- Identify warehouse zones approaching capacity threshold
CREATE VIEW vw_WarehouseBottleneckForecast AS
WITH ZoneActivity AS (
    SELECT 
        wo.GeographyKey,
        g.LocationName,
        wo.StorageZone,
        COUNT(*) AS OperationsLast7Days,
        AVG(wo.OperationDurationMinutes) AS AvgDuration,
        MAX(wo.OperationDurationMinutes) AS MaxDuration
    FROM FactWarehouseOperations wo
    INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
    WHERE wo.OperationStartTime >= DATEADD(DAY, -7, GETDATE())
    GROUP BY wo.GeographyKey, g.LocationName, wo.StorageZone
),
HistoricalBaseline AS (
    SELECT 
        wo.GeographyKey,
        wo.StorageZone,
        AVG(COUNT(*)) OVER (PARTITION BY wo.GeographyKey, wo.StorageZone) AS AvgOperationsPerWeek
    FROM FactWarehouseOperations wo
    WHERE wo.OperationStartTime BETWEEN DATEADD(DAY, -90, GETDATE()) AND DATEADD(DAY, -7, GETDATE())
    GROUP BY wo.GeographyKey, wo.StorageZone, DATEPART(WEEK, wo.OperationStartTime)
)
SELECT 
    za.LocationName,
    za.StorageZone,
    za.OperationsLast7Days,
    hb.AvgOperationsPerWeek,
    (za.OperationsLast7Days - hb.AvgOperationsPerWeek) / hb.AvgOperationsPerWeek * 100 AS PercentChange,
    za.AvgDuration,
    za.MaxDuration,
    CASE 
        WHEN (za.OperationsLast7Days - hb.AvgOperationsPerWeek) / hb.AvgOperationsPerWeek > 0.3 THEN 'High Risk'
        WHEN (za.OperationsLast7Days - hb.AvgOperationsPerWeek) / hb.AvgOperationsPerWeek > 0.15 THEN 'Moderate Risk'
        ELSE 'Normal'
    END AS BottleneckRisk
FROM ZoneActivity za
INNER JOIN HistoricalBaseline hb ON za.GeographyKey = hb.GeographyKey AND za.StorageZone = hb.StorageZone
WHERE (za.OperationsLast7Days - hb.AvgOperationsPerWeek) / hb.AvgOperationsPerWeek > 0.15;
```

## Power BI Dashboard Configuration

### Key DAX Measures

```dax
// Total Operations
TotalOperations = COUNTROWS(FactWarehouseOperations)

// Average Dwell Time
AvgDwellTime = AVERAGE(FactWarehouseOperations[DwellTimeHours])

// Fleet Utilization Rate
FleetUtilization = 
DIVIDE(
    SUMX(FactFleetTrips, FactFleetTrips[TripDurationMinutes] - FactFleetTrips[IdleTimeMinutes]),
    SUMX(FactFleetTrips, FactFleetTrips[TripDurationMinutes]),
    0
) * 100

// On-Time Delivery %
OnTimeDeliveryPct = 
DIVIDE(
    COUNTROWS(FILTER(FactFleetTrips, FactFleetTrips[OnTimeDelivery] = TRUE())),
    COUNTROWS(FactFleetTrips),
    0
) * 100

// Fuel Efficiency (KM per Liter)
FuelEfficiency = 
DIVIDE(
    SUM(FactFleetTrips[DistanceKM]),
    SUM(FactFleetTrips[FuelConsumedLiters]),
    0
)

// Cross-Fact: Dwell Cost Impact ($/hour)
DwellCostImpact = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR AvgProductValue = AVERAGE(DimProduct[StandardCost])
RETURN AvgDwell * AvgProductValue * 0.02 // 2% holding cost per hour assumption
```

### Row-Level Security

```dax
// Create role: Regional Manager (can only see their region)
[DimGeography[Region]] = USERNAME()

// Create role: Warehouse Supervisor (can only see their warehouse)
[DimGeography[LocationID]] = LOOKUPVALUE(
    Users[WarehouseID],
    Users[Email],
    USERNAME()
)

// Create role: Executive (sees all data)
1=1
```

## Automated Alerts Setup

```sql
-- Create alert monitoring procedure
CREATE PROCEDURE usp_GenerateAlerts
AS
BEGIN
    DECLARE @AlertThresholdDwell INT = 72; -- hours
    DECLARE @AlertThresholdIdle DECIMAL(5,2) = 0.15; -- 15%
    
    -- Alert: High dwell time
    INSERT INTO AlertLog (AlertType, Severity, Description, GeneratedAt)
    SELECT 
        'High Dwell Time',
        'Warning',
        CONCAT('SKU ', p.SKU, ' in ', g.LocationName, ' has dwell time of ', 
               CAST(AVG(wo.DwellTimeHours) AS VARCHAR), ' hours (threshold: ', 
               @AlertThresholdDwell, ')'),
        GETDATE()
    FROM FactWarehouseOperations wo
    INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
    INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
    WHERE wo.OperationStartTime >= DATEADD(DAY, -1, GETDATE())
    GROUP BY p.SKU, g.LocationName
    HAVING AVG(wo.DwellTimeHours) > @AlertThresholdDwell;
    
    -- Alert: High fleet idle time
    INSERT INTO AlertLog (AlertType, Severity, Description, GeneratedAt)
    SELECT 
        'High Fleet Idle Time',
        'Critical',
        CONCAT('Vehicle ', f.VehicleID, ' idle time is ', 
               CAST(AVG(CAST(ft.IdleTimeMinutes AS FLOAT) / ft.TripDurationMinutes) * 100 AS VARCHAR(5)), 
               '% (threshold: ', @AlertThresholdIdle * 100, '%)'),
        GETDATE()
    FROM FactFleetTrips ft
    INNER JOIN DimFleet f ON ft.FleetKey = f.FleetKey
    WHERE ft.TripStartTime >= DATEADD(DAY, -1, GETDATE())
    GROUP BY f.VehicleID
    HAVING AVG(CAST(ft.IdleTimeMinutes AS FLOAT) / ft.TripDurationMinutes) > @AlertThresholdIdle;
END;
GO

-- Schedule alert job (SQL Server Agent)
EXEC msdb.dbo.sp_add_job
    @job_name = N'LogiFleet_Alert_Monitor',
    @enabled = 1;

EXEC msdb.dbo.sp_add_jobstep
    @job_name = N'LogiFleet_Alert_Monitor',
    @step_name = N'Generate Alerts',
    @command = N'EXEC usp_GenerateAlerts',
    @database_name = N'LogiFleetPulse';

EXEC msdb.dbo.sp_add_schedule
    @schedule_name = N'Every_15_Minutes',
    @freq_type = 4,
    @freq_interval = 1,
    @freq_subday_type = 4,
    @freq_subday_interval = 15;

EXEC msdb.dbo.sp_attach_schedule
    @job_name = N'LogiFleet_Alert_Monitor',
    @schedule_name = N'Every_15_Minutes';

EXEC msdb.dbo.sp_add_jobserver
    @job_name = N'LogiFleet_Alert_Monitor';
```

## External Data Integration

### Weather API Integration (Azure Function Example)

```sql
-- Create external table for weather data
CREATE EXTERNAL TABLE Weather_API (
    Timestamp DATETIME2,
    LocationID VARCHAR(50),
    Condition VARCHAR(50),
    Temperature DECIMAL(5,2),
    Precipitation DECIMAL(5,2),
    Visibility DECIMAL(5,2)
)
WITH (
    DATA_SOURCE = WeatherDataSource,
    LOCATION = '/weather/',
    FILE_FORMAT = JSONFormat
);

-- Use in fleet trip analysis
SELECT 
    f.VehicleID,
    AVG(CASE WHEN w.Condition LIKE '%Rain%' THEN ft.TripDurationMinutes ELSE NULL END) AS AvgTripDurationRain,
    AVG(CASE WHEN w.Condition LIKE '%Clear%' THEN ft.TripDurationMinutes ELSE NULL END) AS AvgTripDurationClear
FROM FactFleetTrips ft
INNER JOIN DimFleet f ON ft.FleetKey = f.FleetKey
LEFT JOIN Weather_API w ON ft.TripStartTime = w.Timestamp AND ft.DestinationGeographyKey = w.LocationID
GROUP BY f.VehicleID;
```

## Common Troubleshooting

### Issue: Slow Dashboard Refresh

**Solution:** Implement aggregated tables

```sql
-- Create aggregated fact table for faster queries
CREATE TABLE FactWarehouseOperations_Daily (
    DateKey INT,
    GeographyKey INT,
    ProductKey INT,
    TotalOperations INT,
    AvgDwellTimeHours DECIMAL(10,2),
    TotalQuantityHandled INT,
    PRIMARY KEY (DateKey, GeographyKey, ProductKey)
);

-- Populate with nightly job
INSERT INTO FactWarehouseOperations_Daily
SELECT 
    t.Date AS DateKey,
    wo.GeographyKey,
    wo.ProductKey,
    COUNT(*) AS TotalOperations,
    AVG(wo.DwellTimeHours) AS AvgDwellTimeHours,
    SUM(wo.QuantityHandled) AS TotalQuantityHandled
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE CAST(wo.OperationStartTime AS DATE) = CAST(DATEADD(DAY, -1, GETDATE()) AS DATE)
GROUP BY t.Date, wo.GeographyKey, wo.ProductKey;
```

### Issue: Missing Relationships in Power BI

**Solution:** Verify foreign key constraints

```sql
-- Check for orphaned records
SELECT wo.OperationKey
FROM FactWarehouseOperations wo
LEFT JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE t.TimeKey IS NULL;

-- Fix by populating missing dimension records
INSERT INTO DimTime (TimeKey, FullDateTime, Date, FiscalYear, FiscalQuarter, FiscalMonth, DayOfWeek, HourOfDay, MinuteBucket)
SELECT DISTINCT
    CONVERT(INT, FORMAT(wo.OperationStartTime, 'yyyyMMddHHmm')),
    wo.OperationStartTime,
    CAST(wo.OperationStartTime AS DATE),
    YEAR(wo.OperationStartTime),
    DATEPART(QUARTER, wo.OperationStartTime),
    MONTH(wo.OperationStartTime),
    DATEPART(WEEKDAY, wo.OperationStartTime),
    DATEPART(HOUR, wo.OperationStartTime),
    (DATEPART(MINUTE, wo.OperationStartTime) / 15) * 15
FROM FactWarehouseOperations wo
LEFT JOIN DimTime t ON CONVERT(INT, FORMAT(wo.OperationStartTime, 'yyyyMMddHHmm')) = t.TimeKey
WHERE t.TimeKey IS NULL;
```

### Issue: Incorrect Gravity Score Calculation

**Solution:** Recalculate based on recent activity

```sql
-- Update product gravity scores quarterly
UPDATE p
SET GravityScore = calc.NewGravityScore
FROM DimProduct p
INNER JOIN (
    SELECT 
        wo.ProductKey,
        (COUNT(*) / 90.0) * 10 + -- Velocity component (picks per day * 10)
        (AVG(p.StandardCost) / 100) * 5 + -- Value component
        (CASE WHEN p.IsFragile = 1 THEN 20 ELSE 0 END) AS NewGravityScore -- Fragility component
    FROM FactWarehouseOperations wo
    INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
    WHERE wo.Oper
