---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI supply chain analytics platform with multi-fact star schema for warehouse, fleet, and logistics intelligence
triggers:
  - set up LogiFleet Pulse analytics platform
  - deploy supply chain data warehouse schema
  - configure Power BI logistics dashboard
  - implement warehouse and fleet KPI tracking
  - create multi-fact star schema for logistics
  - build real-time supply chain analytics
  - integrate warehouse and fleet telemetry data
  - set up cross-modal logistics intelligence
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is an advanced supply chain analytics platform that unifies warehouse operations, fleet telemetry, inventory management, and external data signals into a single semantic data layer. Built on MS SQL Server with Power BI visualization, it provides:

- **Multi-fact star schema** linking warehouse operations, fleet trips, and cross-dock activities
- **Time-phased dimensions** with 15-minute granularity for real-time insights
- **Cross-fact KPI harmonization** (e.g., inventory dwell time vs. fleet idling costs)
- **Predictive bottleneck detection** using historical pattern analysis
- **Warehouse gravity zones** optimizing storage based on velocity, value, and fragility
- **Adaptive fleet triage** prioritizing maintenance by revenue impact

The platform is designed for 3PLs, retail chains, distributors, and any organization managing complex supply chain operations.

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- Data sources: WMS, TMS, telematics APIs, or sample CSV files

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Run the schema deployment script

USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME2 NOT NULL,
    Date DATE NOT NULL,
    Year INT NOT NULL,
    Quarter INT NOT NULL,
    Month INT NOT NULL,
    Week INT NOT NULL,
    DayOfWeek INT NOT NULL,
    DayName VARCHAR(10) NOT NULL,
    Hour INT NOT NULL,
    HourBucket INT NOT NULL, -- 15-minute intervals
    FiscalYear INT NOT NULL,
    FiscalQuarter INT NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT DEFAULT 0
);

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    NodeID VARCHAR(50) UNIQUE NOT NULL,
    NodeType VARCHAR(20) NOT NULL, -- 'Warehouse', 'RoutePoint', 'CrossDock'
    NodeName VARCHAR(200) NOT NULL,
    Address VARCHAR(500),
    City VARCHAR(100),
    Region VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone VARCHAR(50)
);

CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) UNIQUE NOT NULL,
    ProductName VARCHAR(500),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    UnitCost DECIMAL(18,2),
    UnitPrice DECIMAL(18,2),
    Weight DECIMAL(10,2),
    Volume DECIMAL(10,2),
    FragilityScore INT CHECK (FragilityScore BETWEEN 1 AND 10),
    VelocityScore INT CHECK (VelocityScore BETWEEN 1 AND 10),
    GravityZone VARCHAR(20), -- 'High', 'Medium', 'Low'
    OptimalStorageTemp DECIMAL(5,2),
    ShelfLifeDays INT
);

CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) UNIQUE NOT NULL,
    SupplierName VARCHAR(200),
    LeadTimeAvgDays INT,
    LeadTimeVarianceDays DECIMAL(10,2),
    DefectPercentage DECIMAL(5,2),
    ComplianceScore INT CHECK (ComplianceScore BETWEEN 0 AND 100),
    OnTimeDeliveryPercentage DECIMAL(5,2)
);

CREATE TABLE DimVehicle (
    VehicleKey INT PRIMARY KEY IDENTITY(1,1),
    VehicleID VARCHAR(50) UNIQUE NOT NULL,
    VehicleType VARCHAR(50), -- 'Box Truck', 'Semi', 'Van', etc.
    Capacity DECIMAL(10,2),
    FuelType VARCHAR(20),
    YearManufactured INT,
    LastMaintenanceDate DATE,
    MaintenanceScore INT CHECK (MaintenanceScore BETWEEN 0 AND 100)
);

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    OperationType VARCHAR(20) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    Quantity INT NOT NULL,
    DurationMinutes DECIMAL(10,2),
    DwellTimeHours DECIMAL(10,2),
    ZoneID VARCHAR(50),
    EmployeeID VARCHAR(50),
    BatchNumber VARCHAR(100),
    DefectCount INT DEFAULT 0,
    CostUSD DECIMAL(18,2)
);

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleKey INT NOT NULL FOREIGN KEY REFERENCES DimVehicle(VehicleKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DriverID VARCHAR(50),
    RouteID VARCHAR(100),
    PlannedDurationMinutes INT,
    ActualDurationMinutes INT,
    IdleTimeMinutes INT,
    DistanceMiles DECIMAL(10,2),
    FuelConsumedGallons DECIMAL(10,2),
    LoadWeight DECIMAL(10,2),
    StopCount INT,
    DelayMinutes INT,
    DelayReason VARCHAR(200),
    CostUSD DECIMAL(18,2)
);

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    Quantity INT NOT NULL,
    DwellTimeMinutes DECIMAL(10,2),
    TransferType VARCHAR(50)
);

-- Create indexes for performance
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Geography ON FactWarehouseOperations(GeographyKey);
CREATE NONCLUSTERED INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleKey);
CREATE NONCLUSTERED INDEX IX_FactFleet_Origin ON FactFleetTrips(OriginGeographyKey);
```

### Step 2: Populate Time Dimension

```sql
-- Stored procedure to populate DimTime with 15-minute intervals
CREATE PROCEDURE PopulateTimeDimension
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentDateTime DATETIME2 = @StartDate;
    DECLARE @TimeKey INT = 1;
    
    WHILE @CurrentDateTime <= @EndDate
    BEGIN
        INSERT INTO DimTime (
            TimeKey, DateTime, Date, Year, Quarter, Month, Week, 
            DayOfWeek, DayName, Hour, HourBucket, 
            FiscalYear, FiscalQuarter, IsWeekend
        )
        VALUES (
            @TimeKey,
            @CurrentDateTime,
            CAST(@CurrentDateTime AS DATE),
            YEAR(@CurrentDateTime),
            DATEPART(QUARTER, @CurrentDateTime),
            MONTH(@CurrentDateTime),
            DATEPART(WEEK, @CurrentDateTime),
            DATEPART(WEEKDAY, @CurrentDateTime),
            DATENAME(WEEKDAY, @CurrentDateTime),
            DATEPART(HOUR, @CurrentDateTime),
            (DATEPART(HOUR, @CurrentDateTime) * 4) + (DATEPART(MINUTE, @CurrentDateTime) / 15),
            CASE WHEN MONTH(@CurrentDateTime) >= 7 THEN YEAR(@CurrentDateTime) + 1 ELSE YEAR(@CurrentDateTime) END,
            CASE WHEN MONTH(@CurrentDateTime) >= 7 THEN DATEPART(QUARTER, @CurrentDateTime) - 2 ELSE DATEPART(QUARTER, @CurrentDateTime) + 2 END,
            CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1, 7) THEN 1 ELSE 0 END
        );
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
        SET @TimeKey = @TimeKey + 1;
    END
END;
GO

-- Execute to populate 2 years of data
EXEC PopulateTimeDimension '2025-01-01', '2026-12-31';
```

### Step 3: Create Data Loading Procedures

```sql
-- Incremental loading for warehouse operations
CREATE PROCEDURE LoadWarehouseOperations
    @SourceConnectionString NVARCHAR(500)
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Load from external source (WMS database, API, CSV)
    -- Example using OPENROWSET for CSV
    
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, SupplierKey,
        OperationType, Quantity, DurationMinutes, DwellTimeHours,
        ZoneID, EmployeeID, BatchNumber, DefectCount, CostUSD
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.SupplierKey,
        src.OperationType,
        src.Quantity,
        src.DurationMinutes,
        src.DwellTimeHours,
        src.ZoneID,
        src.EmployeeID,
        src.BatchNumber,
        src.DefectCount,
        src.CostUSD
    FROM OPENROWSET(
        BULK 'C:\data\warehouse_ops.csv',
        FORMATFILE = 'C:\data\warehouse_format.xml'
    ) AS src
    INNER JOIN DimTime t ON t.DateTime = src.OperationDateTime
    INNER JOIN DimGeography g ON g.NodeID = src.WarehouseID
    INNER JOIN DimProductGravity p ON p.SKU = src.SKU
    LEFT JOIN DimSupplierReliability s ON s.SupplierID = src.SupplierID
    WHERE NOT EXISTS (
        SELECT 1 FROM FactWarehouseOperations f
        WHERE f.TimeKey = t.TimeKey 
        AND f.ProductKey = p.ProductKey 
        AND f.BatchNumber = src.BatchNumber
    );
END;
GO
```

### Step 4: Configure Power BI Connection

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Server: `your-server-name`
4. Database: `LogiFleetPulse`
5. Data Connectivity mode: Import (for smaller datasets) or DirectQuery (for real-time)
6. Load all dimension and fact tables

## Key SQL Views for Power BI

```sql
-- Unified KPI view combining warehouse and fleet metrics
CREATE VIEW vw_UnifiedLogisticsKPI AS
SELECT 
    t.Date,
    t.Year,
    t.Month,
    g.Country,
    g.Region,
    g.NodeName,
    
    -- Warehouse metrics
    SUM(CASE WHEN wo.OperationType = 'Picking' THEN wo.Quantity ELSE 0 END) AS TotalPickedUnits,
    AVG(CASE WHEN wo.OperationType = 'Picking' THEN wo.DurationMinutes ELSE NULL END) AS AvgPickTimeMinutes,
    SUM(wo.DwellTimeHours) AS TotalDwellTimeHours,
    SUM(wo.DefectCount) AS TotalDefects,
    
    -- Fleet metrics
    COUNT(DISTINCT ft.TripKey) AS TotalTrips,
    SUM(ft.IdleTimeMinutes) AS TotalIdleTimeMinutes,
    SUM(ft.FuelConsumedGallons) AS TotalFuelGallons,
    AVG(ft.ActualDurationMinutes - ft.PlannedDurationMinutes) AS AvgDelayMinutes,
    
    -- Cross-fact KPIs
    SUM(wo.CostUSD) + SUM(ft.CostUSD) AS TotalOperatingCostUSD,
    CASE 
        WHEN SUM(wo.Quantity) > 0 
        THEN (SUM(wo.CostUSD) + SUM(ft.CostUSD)) / SUM(wo.Quantity)
        ELSE 0 
    END AS CostPerUnit
    
FROM DimTime t
LEFT JOIN FactWarehouseOperations wo ON wo.TimeKey = t.TimeKey
LEFT JOIN DimGeography g ON g.GeographyKey = wo.GeographyKey
LEFT JOIN FactFleetTrips ft ON ft.TimeKey = t.TimeKey AND ft.OriginGeographyKey = g.GeographyKey
GROUP BY 
    t.Date, t.Year, t.Month, g.Country, g.Region, g.NodeName;
GO

-- Product gravity analysis
CREATE VIEW vw_ProductGravityAnalysis AS
SELECT 
    p.SKU,
    p.ProductName,
    p.Category,
    p.GravityZone,
    p.VelocityScore,
    p.FragilityScore,
    
    COUNT(wo.OperationKey) AS OperationCount,
    AVG(wo.DwellTimeHours) AS AvgDwellHours,
    SUM(wo.Quantity) AS TotalUnitsHandled,
    AVG(wo.DurationMinutes) AS AvgHandlingMinutes,
    
    -- Recommend zone reassignment
    CASE 
        WHEN AVG(wo.DwellTimeHours) < 24 AND p.GravityZone != 'High' THEN 'Consider High Gravity'
        WHEN AVG(wo.DwellTimeHours) > 168 AND p.GravityZone != 'Low' THEN 'Consider Low Gravity'
        ELSE 'Optimal'
    END AS ZoneRecommendation
    
FROM DimProductGravity p
INNER JOIN FactWarehouseOperations wo ON wo.ProductKey = p.ProductKey
WHERE wo.TimeKey IN (SELECT TimeKey FROM DimTime WHERE Date >= DATEADD(MONTH, -3, GETDATE()))
GROUP BY 
    p.SKU, p.ProductName, p.Category, p.GravityZone, p.VelocityScore, p.FragilityScore;
GO

-- Fleet maintenance prioritization
CREATE VIEW vw_FleetMaintenancePriority AS
SELECT 
    v.VehicleID,
    v.VehicleType,
    v.LastMaintenanceDate,
    DATEDIFF(DAY, v.LastMaintenanceDate, GETDATE()) AS DaysSinceLastMaintenance,
    
    AVG(ft.IdleTimeMinutes) AS AvgIdleTimeMinutes,
    SUM(ft.DistanceMiles) AS TotalMileage,
    SUM(ft.DelayMinutes) AS TotalDelayMinutes,
    
    -- Calculate priority score
    (
        (DATEDIFF(DAY, v.LastMaintenanceDate, GETDATE()) / 30.0) * 0.3 +
        (AVG(ft.IdleTimeMinutes) / 60.0) * 0.3 +
        ((100 - v.MaintenanceScore) / 10.0) * 0.4
    ) AS PriorityScore,
    
    CASE 
        WHEN (DATEDIFF(DAY, v.LastMaintenanceDate, GETDATE()) / 30.0) * 0.3 +
             (AVG(ft.IdleTimeMinutes) / 60.0) * 0.3 +
             ((100 - v.MaintenanceScore) / 10.0) * 0.4 > 7 THEN 'Critical'
        WHEN (DATEDIFF(DAY, v.LastMaintenanceDate, GETDATE()) / 30.0) * 0.3 +
             (AVG(ft.IdleTimeMinutes) / 60.0) * 0.3 +
             ((100 - v.MaintenanceScore) / 10.0) * 0.4 > 5 THEN 'High'
        ELSE 'Normal'
    END AS MaintenancePriority
    
FROM DimVehicle v
INNER JOIN FactFleetTrips ft ON ft.VehicleKey = v.VehicleKey
WHERE ft.TimeKey IN (SELECT TimeKey FROM DimTime WHERE Date >= DATEADD(MONTH, -1, GETDATE()))
GROUP BY 
    v.VehicleID, v.VehicleType, v.LastMaintenanceDate, v.MaintenanceScore;
GO
```

## Power BI DAX Measures

```dax
// Key DAX measures for the Power BI model

// Warehouse velocity score
WarehouseVelocityScore = 
VAR TotalOps = COUNTROWS(FactWarehouseOperations)
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR PickEfficiency = 
    DIVIDE(
        CALCULATE(SUM(FactWarehouseOperations[Quantity]), 
                  FactWarehouseOperations[OperationType] = "Picking"),
        CALCULATE(SUM(FactWarehouseOperations[DurationMinutes]),
                  FactWarehouseOperations[OperationType] = "Picking")
    )
RETURN
    (TotalOps * 0.3) + ((168 - AvgDwell) * 0.4) + (PickEfficiency * 100 * 0.3)

// Fleet efficiency ratio
FleetEfficiencyRatio = 
DIVIDE(
    SUM(FactFleetTrips[ActualDurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[ActualDurationMinutes]),
    0
)

// Cross-fact bottleneck index
BottleneckIndex = 
VAR HighDwellProducts = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[DwellTimeHours] > 72
    )
VAR DelayedTrips = 
    CALCULATE(
        COUNTROWS(FactFleetTrips),
        FactFleetTrips[DelayMinutes] > 30
    )
VAR TotalOperations = COUNTROWS(FactWarehouseOperations) + COUNTROWS(FactFleetTrips)
RETURN
    DIVIDE(HighDwellProducts + DelayedTrips, TotalOperations, 0) * 100

// Cost per mile with fuel efficiency
CostPerMileOptimized = 
DIVIDE(
    SUM(FactFleetTrips[CostUSD]),
    SUM(FactFleetTrips[DistanceMiles])
) - 
(0.1 * AVERAGE(DimVehicle[MaintenanceScore]))

// Dynamic gravity zone recommendation
GravityZoneOptimal = 
SWITCH(
    TRUE(),
    AVERAGE(FactWarehouseOperations[DwellTimeHours]) < 24, "High",
    AVERAGE(FactWarehouseOperations[DwellTimeHours]) < 72, "Medium",
    "Low"
)
```

## Configuration

### Environment Variables

```bash
# SQL Server connection
export SQL_SERVER="your-server.database.windows.net"
export SQL_DATABASE="LogiFleetPulse"
export SQL_USERNAME="logifleet_admin"
export SQL_PASSWORD="${SQL_ADMIN_PASSWORD}"

# External data sources
export WMS_API_ENDPOINT="https://wms.yourcompany.com/api/v1"
export WMS_API_KEY="${WMS_API_SECRET}"
export TELEMETRY_API_ENDPOINT="https://fleet.yourcompany.com/telemetry"
export TELEMETRY_API_KEY="${TELEMETRY_API_SECRET}"

# Alert configuration
export ALERT_EMAIL_SMTP="smtp.office365.com"
export ALERT_EMAIL_FROM="logifleet@yourcompany.com"
export ALERT_EMAIL_TO="operations@yourcompany.com"

# Power BI service (for automated refresh)
export POWERBI_WORKSPACE_ID="${POWERBI_WORKSPACE_ID}"
export POWERBI_DATASET_ID="${POWERBI_DATASET_ID}"
export POWERBI_CLIENT_ID="${AZURE_CLIENT_ID}"
export POWERBI_CLIENT_SECRET="${AZURE_CLIENT_SECRET}"
```

### Alert Configuration

```sql
-- Create alerting procedure
CREATE PROCEDURE CheckThresholdsAndAlert
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX) = '';
    
    -- Check for high dwell time
    IF EXISTS (
        SELECT 1 FROM FactWarehouseOperations wo
        INNER JOIN DimTime t ON t.TimeKey = wo.TimeKey
        WHERE t.Date = CAST(GETDATE() AS DATE)
        AND wo.DwellTimeHours > 72
        GROUP BY wo.ProductKey
        HAVING COUNT(*) > 5
    )
    BEGIN
        SET @AlertMessage += 'ALERT: Multiple high-gravity products experiencing extended dwell time (>72 hours). ';
    END
    
    -- Check for fleet idling
    IF EXISTS (
        SELECT 1 FROM FactFleetTrips ft
        INNER JOIN DimTime t ON t.TimeKey = ft.TimeKey
        WHERE t.Date = CAST(GETDATE() AS DATE)
        AND (CAST(ft.IdleTimeMinutes AS FLOAT) / ft.ActualDurationMinutes) > 0.15
        HAVING COUNT(*) > 10
    )
    BEGIN
        SET @AlertMessage += 'ALERT: Excessive fleet idling detected (>15% of trip time across 10+ trips). ';
    END
    
    -- Check for maintenance priority
    IF EXISTS (
        SELECT 1 FROM vw_FleetMaintenancePriority
        WHERE MaintenancePriority = 'Critical'
    )
    BEGIN
        SET @AlertMessage += 'ALERT: Critical fleet maintenance required. Review vw_FleetMaintenancePriority. ';
    END
    
    -- Send alert if any conditions met
    IF @AlertMessage != ''
    BEGIN
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = 'operations@yourcompany.com',
            @subject = 'LogiFleet Pulse: Threshold Alert',
            @body = @AlertMessage;
    END
END;
GO

-- Schedule via SQL Agent (run every 15 minutes)
```

## Common Patterns

### Pattern 1: Linking Warehouse Dwell to Fleet Delays

```sql
-- Identify products with high dwell time that also correlate with delayed shipments
SELECT 
    p.SKU,
    p.ProductName,
    AVG(wo.DwellTimeHours) AS AvgWarehouseDwellHours,
    AVG(ft.DelayMinutes) AS AvgFleetDelayMinutes,
    COUNT(DISTINCT ft.TripKey) AS AffectedTrips,
    SUM(wo.Quantity) AS TotalUnits
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON p.ProductKey = wo.ProductKey
INNER JOIN FactCrossDock cd ON cd.ProductKey = p.ProductKey
INNER JOIN FactFleetTrips ft ON ft.TripKey = cd.OutboundTripKey
WHERE wo.DwellTimeHours > 48
GROUP BY p.SKU, p.ProductName
HAVING AVG(ft.DelayMinutes) > 20
ORDER BY AvgFleetDelayMinutes DESC;
```

### Pattern 2: Seasonal Demand Forecasting

```sql
-- Analyze seasonal patterns for inventory planning
WITH MonthlyTrends AS (
    SELECT 
        p.Category,
        t.Month,
        t.Year,
        SUM(wo.Quantity) AS TotalUnits,
        AVG(wo.DwellTimeHours) AS AvgDwellHours
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON p.ProductKey = wo.ProductKey
    INNER JOIN DimTime t ON t.TimeKey = wo.TimeKey
    WHERE wo.OperationType = 'Shipping'
    GROUP BY p.Category, t.Month, t.Year
)
SELECT 
    Category,
    Month,
    AVG(TotalUnits) AS AvgMonthlyUnits,
    STDEV(TotalUnits) AS UnitVariance,
    CASE 
        WHEN STDEV(TotalUnits) > AVG(TotalUnits) * 0.5 THEN 'High Volatility'
        ELSE 'Stable'
    END AS DemandPattern
FROM MonthlyTrends
GROUP BY Category, Month
ORDER BY Category, Month;
```

### Pattern 3: Route Optimization by Geography

```sql
-- Identify optimal routes based on cost, delay, and fuel efficiency
SELECT 
    origin.NodeName AS OriginNode,
    dest.NodeName AS DestinationNode,
    COUNT(ft.TripKey) AS TripCount,
    AVG(ft.DistanceMiles) AS AvgDistance,
    AVG(ft.FuelConsumedGallons) AS AvgFuelGallons,
    AVG(ft.ActualDurationMinutes) AS AvgDurationMinutes,
    AVG(ft.DelayMinutes) AS AvgDelayMinutes,
    AVG(ft.CostUSD) AS AvgCostUSD,
    
    -- Calculate efficiency score
    (
        (1 - (AVG(ft.DelayMinutes) / NULLIF(AVG(ft.ActualDurationMinutes), 0))) * 0.4 +
        (1 - (AVG(ft.FuelConsumedGallons) / NULLIF(AVG(ft.DistanceMiles), 0) / 0.08)) * 0.3 +
        (1 - (AVG(ft.IdleTimeMinutes) / NULLIF(AVG(ft.ActualDurationMinutes), 0))) * 0.3
    ) * 100 AS RouteEfficiencyScore
    
FROM FactFleetTrips ft
INNER JOIN DimGeography origin ON origin.GeographyKey = ft.OriginGeographyKey
INNER JOIN DimGeography dest ON dest.GeographyKey = ft.DestinationGeographyKey
GROUP BY origin.NodeName, dest.NodeName
HAVING COUNT(ft.TripKey) >= 5
ORDER BY RouteEfficiencyScore DESC;
```

## Troubleshooting

### Issue: Power BI Refresh Timeout

**Symptom**: Dataset refresh fails with timeout error

**Solution**: Switch to incremental refresh

```sql
-- Create partitioned views for large fact tables
CREATE VIEW FactWarehouseOperations_Current AS
SELECT * FROM FactWarehouseOperations
WHERE TimeKey IN (
    SELECT TimeKey FROM DimTime 
    WHERE Date >= DATEADD(MONTH, -3, GETDATE())
);

CREATE VIEW FactWarehouseOperations_Historical AS
SELECT * FROM FactWarehouseOperations
WHERE TimeKey IN (
    SELECT TimeKey FROM DimTime 
    WHERE Date < DATEADD(MONTH, -3, GETDATE())
);
```

In Power BI:
1. Load `_Current` view with Import mode
2. Load `_Historical` view with DirectQuery mode
3. Combine in composite model

### Issue: Slow Query Performance

**Symptom**: Dashboard takes >10 seconds to load

**Solution**: Add columnstore indexes

```sql
-- Create columnstore index on fact tables
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouse
ON FactWarehouseOperations (
    TimeKey, GeographyKey, ProductKey, OperationType, 
    Quantity, DurationMinutes, DwellTimeHours
);

CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactFleet
ON FactFleetTrips (
    TimeKey, VehicleKey, OriginGeographyKey, DestinationGeographyKey,
    ActualDurationMinutes, IdleTimeMinutes, FuelConsumedGallons
);

-- Update statistics
UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS FactFleetTrips WITH FULLSCAN;
```

### Issue: Duplicate Records in Fact Tables

**Symptom**: KPIs showing inflated numbers

**Solution**: Implement deduplication logic

```sql
-- Remove duplicates based on natural key
WITH Duplicates AS (
    SELECT 
        OperationKey,
        ROW_NUMBER() OVER (
            PARTITION BY TimeKey, ProductKey, BatchNumber, OperationType
            ORDER BY OperationKey
        ) AS RowNum
    FROM FactWarehouseOperations
)
DELETE FROM Duplicates WHERE RowNum > 1;

-- Add unique constraint to prevent future duplicates
ALTER TABLE FactWarehouseOperations
ADD CONSTRAINT UQ_WarehouseOperation 
UNIQUE (TimeKey, ProductKey, BatchNumber, OperationType);
```

### Issue: External Data Source Connectivity

**Symptom**: ETL job fails to pull from WMS API

**Solution**: Implement retry logic with logging

```sql
CREATE PROCEDURE LoadWarehouseOperationsWithRetry
    @MaxRetries INT = 3
