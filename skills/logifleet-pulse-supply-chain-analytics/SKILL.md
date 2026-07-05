---
name: logifleet-pulse-supply-chain-analytics
description: Advanced MS SQL Server & Power BI logistics data warehouse for fleet, warehouse, and cross-modal supply chain intelligence
triggers:
  - set up logifleet pulse supply chain analytics
  - deploy logicore warehouse and fleet database schema
  - configure power bi logistics dashboard
  - build multi-fact star schema for supply chain
  - create warehouse gravity zones analysis
  - implement fleet telemetry tracking dashboard
  - query cross-dock operations with sql
  - set up real-time logistics kpi alerts
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a **multi-modal logistics intelligence platform** built on MS SQL Server and Power BI. It implements a custom multi-fact star schema that unifies:

- **Warehouse operations** (putaway, picking, packing, dwell time)
- **Fleet telemetry** (routes, fuel consumption, idle time, maintenance)
- **Inventory management** (aging curves, gravity zones, turnover rates)
- **External signals** (weather, traffic, supplier reliability)

The platform provides cross-fact KPI harmonization, predictive bottleneck detection, and adaptive spatial optimization through "Warehouse Gravity Zones™" — a weighted scoring system that maps storage areas by pick frequency, item value, and fragility.

## Installation & Deployment

### Prerequisites

- **MS SQL Server 2019+** (Standard or Enterprise edition recommended)
- **Power BI Desktop** (latest version)
- **SQL Server Management Studio (SSMS)** or Azure Data Studio
- Access to warehouse management system (WMS) and fleet telematics APIs
- .NET Framework 4.7.2+ (for automated alerting stored procedures)

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Create the LogiFleet database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    DateTime DATETIME2(0) NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME(0) NOT NULL,
    HourOfDay TINYINT NOT NULL,
    DayOfWeek TINYINT NOT NULL,
    WeekOfYear TINYINT NOT NULL,
    MonthOfYear TINYINT NOT NULL,
    Quarter TINYINT NOT NULL,
    FiscalYear SMALLINT NOT NULL,
    FiscalPeriod TINYINT NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT NOT NULL
)
GO

CREATE CLUSTERED INDEX IX_DimTime_DateTime ON DimTime(DateTime)
CREATE NONCLUSTERED INDEX IX_DimTime_DateKey ON DimTime(DateKey)
GO

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) NOT NULL,
    LocationName NVARCHAR(200) NOT NULL,
    LocationType VARCHAR(50) NOT NULL, -- Warehouse, RoutingNode, Distribution Center
    Continent NVARCHAR(50),
    Country NVARCHAR(100),
    Region NVARCHAR(100),
    City NVARCHAR(100),
    PostalCode VARCHAR(20),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    IsActive BIT NOT NULL DEFAULT 1,
    ValidFrom DATETIME2(0) NOT NULL,
    ValidTo DATETIME2(0) NULL
)
GO

CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) NOT NULL,
    ProductName NVARCHAR(300) NOT NULL,
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    UnitWeight DECIMAL(10,2),
    UnitVolume DECIMAL(10,2),
    IsFragile BIT NOT NULL DEFAULT 0,
    IsPerishable BIT NOT NULL DEFAULT 0,
    ShelfLifeDays INT,
    GravityScore DECIMAL(5,2), -- Weighted score: velocity + value + fragility
    VelocityTier VARCHAR(20), -- Fast, Medium, Slow
    ValueTier VARCHAR(20), -- High, Medium, Low
    OptimalZone VARCHAR(50) -- Recommended storage zone
)
GO

CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) NOT NULL,
    SupplierName NVARCHAR(200) NOT NULL,
    LeadTimeDaysAvg DECIMAL(6,2),
    LeadTimeDaysStdDev DECIMAL(6,2),
    DefectRatePct DECIMAL(5,2),
    OnTimeDeliveryPct DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2), -- 0-100 scale
    ReliabilityTier VARCHAR(20) -- Platinum, Gold, Silver, Bronze
)
GO

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    SupplierKey INT NULL,
    OperationType VARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    OperationID VARCHAR(100) NOT NULL,
    OrderID VARCHAR(100),
    BatchID VARCHAR(100),
    QuantityUnits INT NOT NULL,
    DwellTimeMinutes INT,
    ProcessingTimeSeconds INT,
    WorkerID VARCHAR(50),
    EquipmentID VARCHAR(50),
    StorageZone VARCHAR(50),
    TemperatureC DECIMAL(5,2),
    HumidityPct DECIMAL(5,2),
    QualityCheckPassed BIT,
    CONSTRAINT FK_WarehouseOps_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WarehouseOps_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_WarehouseOps_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    CONSTRAINT FK_WarehouseOps_Supplier FOREIGN KEY (SupplierKey) REFERENCES DimSupplierReliability(SupplierKey)
)
GO

CREATE CLUSTERED INDEX IX_FactWarehouseOps_Time ON FactWarehouseOperations(TimeKey)
CREATE NONCLUSTERED INDEX IX_FactWarehouseOps_Product ON FactWarehouseOperations(ProductKey)
CREATE NONCLUSTERED INDEX IX_FactWarehouseOps_Geography ON FactWarehouseOperations(GeographyKey)
GO

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50),
    TripID VARCHAR(100) NOT NULL,
    RouteSegment NVARCHAR(500),
    DistanceKm DECIMAL(10,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(8,2),
    AvgSpeedKmh DECIMAL(6,2),
    MaxSpeedKmh DECIMAL(6,2),
    LoadWeightKg DECIMAL(10,2),
    LoadVolumeCubicM DECIMAL(10,2),
    StopCount INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    WeatherCondition VARCHAR(50),
    TrafficDelay BIT,
    MaintenanceAlerts INT,
    FuelEfficiencyKmL DECIMAL(6,2),
    CONSTRAINT FK_FleetTrips_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FleetTrips_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FleetTrips_Destination FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
)
GO

CREATE CLUSTERED INDEX IX_FactFleetTrips_Time ON FactFleetTrips(TimeKey)
CREATE NONCLUSTERED INDEX IX_FactFleetTrips_Vehicle ON FactFleetTrips(VehicleID)
GO

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    TransferID VARCHAR(100) NOT NULL,
    QuantityUnits INT NOT NULL,
    DockTimeMinutes INT,
    QualityCheckPassed BIT,
    TemperatureBreachFlag BIT,
    CONSTRAINT FK_CrossDock_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_CrossDock_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_CrossDock_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
)
GO
```

### Step 2: Populate Time Dimension

```sql
-- Generate 15-minute granularity time dimension for 3 years
DECLARE @StartDate DATETIME2(0) = '2025-01-01 00:00:00'
DECLARE @EndDate DATETIME2(0) = '2027-12-31 23:45:00'
DECLARE @CurrentDate DATETIME2(0) = @StartDate

WHILE @CurrentDate <= @EndDate
BEGIN
    INSERT INTO DimTime (
        DateTime, DateKey, TimeOfDay, HourOfDay, DayOfWeek, 
        WeekOfYear, MonthOfYear, Quarter, FiscalYear, FiscalPeriod,
        IsWeekend, IsHoliday
    )
    VALUES (
        @CurrentDate,
        CAST(FORMAT(@CurrentDate, 'yyyyMMdd') AS INT),
        CAST(@CurrentDate AS TIME(0)),
        DATEPART(HOUR, @CurrentDate),
        DATEPART(WEEKDAY, @CurrentDate),
        DATEPART(WEEK, @CurrentDate),
        DATEPART(MONTH, @CurrentDate),
        DATEPART(QUARTER, @CurrentDate),
        CASE WHEN DATEPART(MONTH, @CurrentDate) >= 7 
             THEN YEAR(@CurrentDate) + 1 
             ELSE YEAR(@CurrentDate) END,
        CASE WHEN DATEPART(MONTH, @CurrentDate) >= 7 
             THEN DATEPART(MONTH, @CurrentDate) - 6 
             ELSE DATEPART(MONTH, @CurrentDate) + 6 END,
        CASE WHEN DATEPART(WEEKDAY, @CurrentDate) IN (1, 7) 
             THEN 1 ELSE 0 END,
        0 -- Update with holiday logic as needed
    )
    
    SET @CurrentDate = DATEADD(MINUTE, 15, @CurrentDate)
END
GO
```

### Step 3: Create Gravity Score Calculation Procedure

```sql
CREATE PROCEDURE [dbo].[usp_UpdateProductGravityScores]
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Calculate velocity tier based on last 90 days picking frequency
    WITH ProductVelocity AS (
        SELECT 
            p.ProductKey,
            COUNT(DISTINCT w.OperationKey) AS PickCount,
            NTILE(3) OVER (ORDER BY COUNT(DISTINCT w.OperationKey) DESC) AS VelocityBucket
        FROM DimProductGravity p
        LEFT JOIN FactWarehouseOperations w ON p.ProductKey = w.ProductKey
            AND w.OperationType = 'Picking'
            AND w.TimeKey >= (SELECT MIN(TimeKey) FROM DimTime 
                              WHERE DateTime >= DATEADD(DAY, -90, GETDATE()))
        GROUP BY p.ProductKey
    ),
    GravityCalculation AS (
        SELECT 
            p.ProductKey,
            CASE pv.VelocityBucket
                WHEN 1 THEN 'Fast'
                WHEN 2 THEN 'Medium'
                ELSE 'Slow'
            END AS VelocityTier,
            CASE 
                WHEN p.UnitWeight * pv.PickCount > 10000 THEN 'High'
                WHEN p.UnitWeight * pv.PickCount > 2000 THEN 'Medium'
                ELSE 'Low'
            END AS ValueTier,
            -- Gravity Score: 50% velocity + 30% value + 20% fragility
            (
                (CASE pv.VelocityBucket WHEN 1 THEN 50 WHEN 2 THEN 30 ELSE 10 END) +
                (CASE WHEN p.UnitWeight * pv.PickCount > 10000 THEN 30
                      WHEN p.UnitWeight * pv.PickCount > 2000 THEN 20
                      ELSE 5 END) +
                (CASE WHEN p.IsFragile = 1 THEN 20 ELSE 0 END)
            ) / 100.0 AS GravityScore
        FROM DimProductGravity p
        INNER JOIN ProductVelocity pv ON p.ProductKey = pv.ProductKey
    )
    UPDATE p
    SET 
        p.GravityScore = gc.GravityScore,
        p.VelocityTier = gc.VelocityTier,
        p.ValueTier = gc.ValueTier,
        p.OptimalZone = CASE 
            WHEN gc.GravityScore >= 0.70 THEN 'Zone-A-Express'
            WHEN gc.GravityScore >= 0.45 THEN 'Zone-B-Standard'
            WHEN gc.GravityScore >= 0.25 THEN 'Zone-C-Reserve'
            ELSE 'Zone-D-LongTail'
        END
    FROM DimProductGravity p
    INNER JOIN GravityCalculation gc ON p.ProductKey = gc.ProductKey
END
GO
```

### Step 4: Configure Data Ingestion

```sql
-- Example: Load warehouse operations from WMS export
CREATE PROCEDURE [dbo].[usp_LoadWarehouseOperations]
    @FilePath NVARCHAR(500)
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Create staging table
    IF OBJECT_ID('tempdb..#StagingWarehouseOps') IS NOT NULL
        DROP TABLE #StagingWarehouseOps
    
    CREATE TABLE #StagingWarehouseOps (
        OperationDateTime DATETIME2(0),
        LocationID VARCHAR(50),
        SKU VARCHAR(100),
        OperationType VARCHAR(50),
        OperationID VARCHAR(100),
        QuantityUnits INT,
        ProcessingTimeSeconds INT,
        StorageZone VARCHAR(50)
    )
    
    -- Load from file (adjust based on your WMS export format)
    DECLARE @SQL NVARCHAR(MAX) = N'
    BULK INSERT #StagingWarehouseOps
    FROM ''' + @FilePath + '''
    WITH (
        FIRSTROW = 2,
        FIELDTERMINATOR = '','',
        ROWTERMINATOR = ''\n'',
        TABLOCK
    )'
    
    EXEC sp_executesql @SQL
    
    -- Insert into fact table with dimension lookups
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType, 
        OperationID, QuantityUnits, ProcessingTimeSeconds, StorageZone
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.OperationType,
        s.OperationID,
        s.QuantityUnits,
        s.ProcessingTimeSeconds,
        s.StorageZone
    FROM #StagingWarehouseOps s
    INNER JOIN DimTime t ON s.OperationDateTime = t.DateTime
    INNER JOIN DimGeography g ON s.LocationID = g.LocationID
    INNER JOIN DimProductGravity p ON s.SKU = p.SKU
    WHERE NOT EXISTS (
        SELECT 1 FROM FactWarehouseOperations f 
        WHERE f.OperationID = s.OperationID
    )
END
GO
```

## Power BI Configuration

### Step 1: Connect Power BI to SQL Server

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Server: `your-sql-server.database.windows.net` or local instance
4. Database: `LogiFleetPulse`
5. Data Connectivity mode: **DirectQuery** (for real-time) or **Import** (for scheduled refresh)

### Step 2: Build Relationships in Power BI Model

```dax
// Power BI will auto-detect relationships, verify:
// FactWarehouseOperations[TimeKey] → DimTime[TimeKey]
// FactWarehouseOperations[GeographyKey] → DimGeography[GeographyKey]
// FactWarehouseOperations[ProductKey] → DimProductGravity[ProductKey]
// FactFleetTrips[OriginGeographyKey] → DimGeography[GeographyKey]
// FactFleetTrips[DestinationGeographyKey] → DimGeography[GeographyKey] (set to inactive, use USERELATIONSHIP)
```

### Step 3: Create Key Measures (DAX)

```dax
// Total Warehouse Throughput
TotalThroughput = 
SUM(FactWarehouseOperations[QuantityUnits])

// Average Dwell Time (in hours)
AvgDwellTimeHours = 
AVERAGE(FactWarehouseOperations[DwellTimeMinutes]) / 60

// Fleet Fuel Efficiency
FleetFuelEfficiency = 
DIVIDE(
    SUM(FactFleetTrips[DistanceKm]),
    SUM(FactFleetTrips[FuelConsumedLiters]),
    0
)

// Fleet Idle Time Percentage
FleetIdlePct = 
DIVIDE(
    SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[DurationMinutes]),
    0
)

// Cross-Fact: Dwell Time per Fleet Idle Cost
// Assumption: $50/hour idle cost
DwellCostImpact = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR TotalIdle = SUM(FactFleetTrips[IdleTimeMinutes])
VAR IdleCost = (TotalIdle / 60) * 50
RETURN
    IdleCost * (AvgDwell / 60)

// Predictive Bottleneck Index (simplified)
// Based on: high dwell + high idle + low gravity score products
BottleneckIndex = 
VAR HighDwell = CALCULATE(
    COUNTROWS(FactWarehouseOperations),
    FactWarehouseOperations[DwellTimeMinutes] > 120
)
VAR HighIdle = CALCULATE(
    COUNTROWS(FactFleetTrips),
    FactFleetTrips[IdleTimeMinutes] > 30
)
VAR LowGravity = CALCULATE(
    COUNTROWS(DimProductGravity),
    DimProductGravity[GravityScore] < 0.3
)
RETURN
    (HighDwell * 0.5) + (HighIdle * 0.3) + (LowGravity * 0.2)

// Warehouse Gravity Zone Utilization
GravityZoneUtilization = 
DIVIDE(
    CALCULATE(
        SUM(FactWarehouseOperations[QuantityUnits]),
        FactWarehouseOperations[StorageZone] = DimProductGravity[OptimalZone]
    ),
    SUM(FactWarehouseOperations[QuantityUnits]),
    0
)

// Supplier Reliability Impact
SupplierDelayImpact = 
VAR AvgLeadTime = AVERAGE(DimSupplierReliability[LeadTimeDaysAvg])
VAR OnTimePct = AVERAGE(DimSupplierReliability[OnTimeDeliveryPct])
RETURN
    IF(OnTimePct < 0.85, AvgLeadTime * (1 - OnTimePct), 0)
```

### Step 4: Create Visualizations

**Warehouse Heatmap Dashboard:**
```dax
// Matrix visual: StorageZone (rows) × VelocityTier (columns)
// Values: Sum of DwellTimeMinutes
// Conditional formatting: Red (>120 min), Yellow (60-120), Green (<60)

// Add slicer: DimTime[FiscalPeriod]
// Add slicer: DimGeography[Region]
```

**Fleet Performance Dashboard:**
```dax
// Line chart: DimTime[DateTime] (X-axis), FleetFuelEfficiency (Y-axis)
// Add secondary axis: FleetIdlePct
// Filter: Last 30 days
```

**Cross-Modal KPI Card:**
```dax
// Card visual showing DwellCostImpact
// Tooltip: "Estimated cost impact of warehouse dwell on fleet idle time"
```

## Key SQL Queries for Analysis

### Query 1: Identify Products in Wrong Gravity Zones

```sql
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.OptimalZone,
    w.StorageZone AS CurrentZone,
    COUNT(*) AS MisplacedOperations,
    AVG(w.DwellTimeMinutes) AS AvgDwellTime
FROM FactWarehouseOperations w
INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
WHERE w.StorageZone <> p.OptimalZone
  AND w.OperationType = 'Picking'
  AND w.TimeKey >= (SELECT MIN(TimeKey) FROM DimTime 
                    WHERE DateTime >= DATEADD(DAY, -30, GETDATE()))
GROUP BY p.SKU, p.ProductName, p.GravityScore, p.OptimalZone, w.StorageZone
HAVING AVG(w.DwellTimeMinutes) > 90
ORDER BY AvgDwellTime DESC
```

### Query 2: Fleet Routes with Excessive Idle Time

```sql
SELECT 
    f.VehicleID,
    g1.LocationName AS Origin,
    g2.LocationName AS Destination,
    t.DateTime AS TripStartTime,
    f.DurationMinutes,
    f.IdleTimeMinutes,
    (CAST(f.IdleTimeMinutes AS FLOAT) / f.DurationMinutes) * 100 AS IdlePct,
    f.WeatherCondition,
    f.TrafficDelay
FROM FactFleetTrips f
INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
INNER JOIN DimGeography g1 ON f.OriginGeographyKey = g1.GeographyKey
INNER JOIN DimGeography g2 ON f.DestinationGeographyKey = g2.GeographyKey
WHERE (CAST(f.IdleTimeMinutes AS FLOAT) / f.DurationMinutes) > 0.25
  AND t.DateTime >= DATEADD(DAY, -7, GETDATE())
ORDER BY IdlePct DESC
```

### Query 3: Cross-Dock Bottleneck Analysis

```sql
SELECT 
    cd.TransferID,
    t.DateTime,
    g.LocationName AS CrossDockFacility,
    p.SKU,
    p.OptimalZone,
    cd.DockTimeMinutes,
    cd.TemperatureBreachFlag,
    CASE 
        WHEN cd.DockTimeMinutes > 45 THEN 'Critical'
        WHEN cd.DockTimeMinutes > 30 THEN 'Warning'
        ELSE 'Normal'
    END AS BottleneckStatus
FROM FactCrossDock cd
INNER JOIN DimTime t ON cd.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON cd.GeographyKey = g.GeographyKey
INNER JOIN DimProductGravity p ON cd.ProductKey = p.ProductKey
WHERE cd.DockTimeMinutes > 30
  AND t.DateTime >= DATEADD(DAY, -14, GETDATE())
ORDER BY cd.DockTimeMinutes DESC
```

### Query 4: Supplier Impact on Warehouse Dwell

```sql
SELECT 
    s.SupplierName,
    s.LeadTimeDaysAvg,
    s.OnTimeDeliveryPct,
    s.ReliabilityTier,
    COUNT(DISTINCT w.OperationID) AS TotalReceivingOps,
    AVG(w.DwellTimeMinutes) AS AvgDwellTime,
    STDEV(w.DwellTimeMinutes) AS DwellTimeStdDev
FROM FactWarehouseOperations w
INNER JOIN DimSupplierReliability s ON w.SupplierKey = s.SupplierKey
WHERE w.OperationType = 'Receiving'
  AND w.TimeKey >= (SELECT MIN(TimeKey) FROM DimTime 
                    WHERE DateTime >= DATEADD(DAY, -60, GETDATE()))
GROUP BY s.SupplierName, s.LeadTimeDaysAvg, s.OnTimeDeliveryPct, s.ReliabilityTier
HAVING COUNT(DISTINCT w.OperationID) > 10
ORDER BY AvgDwellTime DESC
```

## Automated Alerting Configuration

### Create Alert Stored Procedure

```sql
CREATE PROCEDURE [dbo].[usp_CheckLogisticsAlerts]
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertLog TABLE (
        AlertType VARCHAR(100),
        Severity VARCHAR(20),
        Message NVARCHAR(500),
        MetricValue DECIMAL(10,2),
        Threshold DECIMAL(10,2)
    )
    
    -- Alert 1: Fleet idle time > 15% of trip duration
    INSERT INTO @AlertLog
    SELECT 
        'Fleet Idle Time Exceeded' AS AlertType,
        'High' AS Severity,
        'Vehicle ' + VehicleID + ' idle time: ' + 
            CAST((CAST(IdleTimeMinutes AS FLOAT) / DurationMinutes) * 100 AS VARCHAR(10)) + '%' AS Message,
        (CAST(IdleTimeMinutes AS FLOAT) / DurationMinutes) * 100 AS MetricValue,
        15.0 AS Threshold
    FROM FactFleetTrips f
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.DateTime >= DATEADD(HOUR, -4, GETDATE())
      AND (CAST(IdleTimeMinutes AS FLOAT) / DurationMinutes) > 0.15
    
    -- Alert 2: High-value products in wrong gravity zone
    INSERT INTO @AlertLog
    SELECT 
        'Product Misplacement' AS AlertType,
        'Medium' AS Severity,
        'SKU ' + p.SKU + ' (Gravity: ' + CAST(p.GravityScore AS VARCHAR(10)) + 
            ') in zone ' + w.StorageZone + ' instead of ' + p.OptimalZone AS Message,
        p.GravityScore AS MetricValue,
        0.60 AS Threshold
    FROM FactWarehouseOperations w
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.DateTime >= DATEADD(HOUR, -2, GETDATE())
      AND w.StorageZone <> p.OptimalZone
      AND p.GravityScore >= 0.60
    
    -- Alert 3: Cross-dock temperature breach
    INSERT INTO @AlertLog
    SELECT 
        'Temperature Breach' AS AlertType,
        'Critical' AS Severity,
        'Transfer ' + TransferID + ' at ' + g.LocationName + 
            ' - Temperature breach detected' AS Message,
        1.0 AS MetricValue,
        1.0 AS Threshold
    FROM FactCrossDock cd
    INNER JOIN DimGeography g ON cd.GeographyKey = g.GeographyKey
    INNER JOIN DimTime t ON cd.TimeKey = t.TimeKey
    WHERE t.DateTime >= DATEADD(HOUR, -1, GETDATE())
      AND cd.TemperatureBreachFlag = 1
    
    -- Send alerts (integrate with email/SMS/Teams)
    IF EXISTS (SELECT 1 FROM @AlertLog)
    BEGIN
        SELECT * FROM @AlertLog ORDER BY 
            CASE Severity 
                WHEN 'Critical' THEN 1 
                WHEN 'High' THEN 2 
                WHEN 'Medium' THEN 3 
                ELSE 4 
            END
        
        -- Email notification logic here
        -- EXEC msdb.dbo.sp_send_dbmail ...
    END
END
GO

-- Schedule alert checks every 15 minutes
-- Use SQL Server Agent or Azure Logic Apps
```

## Common Integration Patterns

### Pattern 1: WMS API Integration (Python Example)

```python
import pyodbc
import requests
import os
from datetime import datetime

# SQL Server connection
conn_string = (
    f"DRIVER={{ODBC Driver 17 for SQL Server}};"
    f"SERVER={os.getenv('SQL_SERVER')};"
    f"DATABASE=LogiFleetPulse;"
    f"UID={os.getenv('SQL_USER')};"
    f"PWD={os.getenv('SQL_PASSWORD')}"
)

# Fetch warehouse operations from WMS API
wms_api_url = os.getenv('WMS_API_URL')
api_key = os.getenv('WMS_API_KEY')

response = requests.get(
    f"{wms_api_url}/operations/recent",
    headers={"Authorization": f"Bearer {api_key}"}
)

operations = response.json()

# Insert into SQL Server
conn = pyodbc.connect(conn_string)
cursor = conn.cursor()

for op in operations:
    cursor.execute("""
        EXEC dbo.usp_LoadWarehouseOperations_SingleRecord
            @OperationDateTime = ?,
            @LocationID = ?,
            @SKU = ?,
            @OperationType = ?,
            @OperationID = ?,
            @QuantityUnits = ?,
            @ProcessingTimeSeconds = ?
    """, 
    op['timestamp'],
    op['location_id'],
    op['sku'],
    op['operation_type'],
    op['operation_id
