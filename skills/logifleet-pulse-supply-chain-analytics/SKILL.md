---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing engine for logistics, fleet management, and supply chain KPI visualization with multi-fact star schema.
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure Power BI logistics dashboard"
  - "create warehouse operations fact table"
  - "build fleet telemetry data model"
  - "implement supply chain star schema"
  - "connect logistics data warehouse to Power BI"
  - "query cross-modal supply chain metrics"
  - "optimize warehouse gravity zones"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

LogiFleet Pulse is a comprehensive data warehousing and analytics platform for logistics and supply chain management. It combines MS SQL Server for data storage with Power BI for visualization, implementing a multi-fact star schema that integrates:

- Warehouse operations (receiving, putaway, picking, packing, shipping)
- Fleet telemetry and GPS data
- Supplier reliability metrics
- Cross-dock operations
- Time-phased dimensions for temporal analysis

The platform provides unified KPI tracking across warehouse and fleet operations, predictive bottleneck detection, and adaptive visual dashboards.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Developer, Standard, or Enterprise edition)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio

### Database Deployment

1. **Clone the repository**:
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema**:
```sql
-- Execute in SSMS against your target database
-- Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Run the schema creation script
-- (Assuming schema.sql is in the repository)
:r schema.sql
GO
```

3. **Configure data source connections**:
```json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "auth": "integrated"
  },
  "wms_api": {
    "endpoint": "${WMS_API_ENDPOINT}",
    "api_key": "${WMS_API_KEY}"
  },
  "telemetry_feed": {
    "endpoint": "${FLEET_TELEMETRY_URL}",
    "api_key": "${FLEET_API_KEY}"
  }
}
```

## Core Data Model Architecture

### Fact Tables

#### FactWarehouseOperations
Primary fact table for warehouse-level metrics:

```sql
CREATE TABLE FactWarehouseOperations (
    OperationID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    SupplierKey INT,
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    Quantity DECIMAL(18,2),
    DwellTimeMinutes INT,
    CycleTimeSeconds INT,
    ErrorCount INT DEFAULT 0,
    OperationCost DECIMAL(18,2),
    CONSTRAINT FK_Warehouse_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Warehouse_Location FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey),
    CONSTRAINT FK_Warehouse_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);

-- Critical index for time-based aggregations
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time_Type 
ON FactWarehouseOperations(TimeKey, OperationType) 
INCLUDE (Quantity, DwellTimeMinutes, OperationCost);
```

#### FactFleetTrips
Captures fleet movement and performance:

```sql
CREATE TABLE FactFleetTrips (
    TripID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    DriverKey INT NOT NULL,
    RouteKey INT NOT NULL,
    OriginWarehouseKey INT,
    DestinationKey INT,
    TripStartTime DATETIME2,
    TripEndTime DATETIME2,
    DistanceMiles DECIMAL(10,2),
    FuelConsumedGallons DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadWeight DECIMAL(10,2),
    DeliveryStatus VARCHAR(50), -- 'OnTime', 'Delayed', 'Failed'
    DelayReasonKey INT,
    TripCost DECIMAL(18,2),
    CONSTRAINT FK_Fleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_Fleet_Route FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey)
);

CREATE NONCLUSTERED INDEX IX_FactFleet_Time_Status 
ON FactFleetTrips(TimeKey, DeliveryStatus) 
INCLUDE (DistanceMiles, FuelConsumedGallons, IdleTimeMinutes);
```

### Dimension Tables

#### DimTime (Time-Phased Dimension)
15-minute granularity time dimension:

```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDate DATE NOT NULL,
    TimeOfDay TIME NOT NULL,
    HourOfDay TINYINT,
    DayOfWeek TINYINT,
    DayName VARCHAR(10),
    WeekOfYear TINYINT,
    MonthNumber TINYINT,
    MonthName VARCHAR(10),
    Quarter TINYINT,
    FiscalYear INT,
    IsWeekend BIT,
    IsHoliday BIT,
    HolidayName VARCHAR(100)
);

-- Populate time dimension (example for one day)
DECLARE @StartDate DATE = '2026-01-01';
DECLARE @EndDate DATE = '2026-12-31';
DECLARE @CurrentDateTime DATETIME = @StartDate;

WHILE @CurrentDateTime <= @EndDate
BEGIN
    INSERT INTO DimTime (
        TimeKey, FullDate, TimeOfDay, HourOfDay, 
        DayOfWeek, DayName, MonthNumber, Quarter, FiscalYear, IsWeekend
    )
    VALUES (
        CONVERT(INT, FORMAT(@CurrentDateTime, 'yyyyMMddHHmm')),
        CAST(@CurrentDateTime AS DATE),
        CAST(@CurrentDateTime AS TIME),
        DATEPART(HOUR, @CurrentDateTime),
        DATEPART(WEEKDAY, @CurrentDateTime),
        DATENAME(WEEKDAY, @CurrentDateTime),
        MONTH(@CurrentDateTime),
        DATEPART(QUARTER, @CurrentDateTime),
        YEAR(@CurrentDateTime),
        CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1,7) THEN 1 ELSE 0 END
    );
    
    SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
END;
```

#### DimProduct (with Gravity Score)
```sql
CREATE TABLE DimProduct (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName VARCHAR(200),
    CategoryKey INT,
    Weight DECIMAL(10,2),
    Volume DECIMAL(10,2),
    IsFragile BIT DEFAULT 0,
    IsPerishable BIT DEFAULT 0,
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value / fragility
    OptimalStorageZone VARCHAR(50),
    LeadTimeDays INT,
    CONSTRAINT FK_Product_Category FOREIGN KEY (CategoryKey) REFERENCES DimCategory(CategoryKey)
);

-- Calculate and update gravity scores
UPDATE DimProduct
SET GravityScore = (
    SELECT (COUNT(fo.OperationID) * AVG(fo.OperationCost)) / 
           CASE WHEN p.IsFragile = 1 THEN 2.0 ELSE 1.0 END
    FROM FactWarehouseOperations fo
    WHERE fo.ProductKey = DimProduct.ProductKey
      AND fo.TimeKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -90, GETDATE()), 'yyyyMMdd0000'))
);
```

## Key Queries & Analytics Patterns

### Cross-Fact KPI: Warehouse Dwell Time vs Fleet Idle Time
```sql
-- Correlate warehouse dwell with fleet idle time by product category
WITH WarehouseDwell AS (
    SELECT 
        p.CategoryKey,
        c.CategoryName,
        AVG(fo.DwellTimeMinutes) AS AvgDwellMinutes,
        dt.FullDate
    FROM FactWarehouseOperations fo
    JOIN DimProduct p ON fo.ProductKey = p.ProductKey
    JOIN DimCategory c ON p.CategoryKey = c.CategoryKey
    JOIN DimTime dt ON fo.TimeKey = dt.TimeKey
    WHERE dt.FullDate >= DATEADD(MONTH, -1, GETDATE())
    GROUP BY p.CategoryKey, c.CategoryName, dt.FullDate
),
FleetIdle AS (
    SELECT 
        p.CategoryKey,
        AVG(ft.IdleTimeMinutes) AS AvgIdleMinutes,
        dt.FullDate
    FROM FactFleetTrips ft
    JOIN DimWarehouse w ON ft.OriginWarehouseKey = w.WarehouseKey
    JOIN FactWarehouseOperations fo ON fo.WarehouseKey = w.WarehouseKey
    JOIN DimProduct p ON fo.ProductKey = p.ProductKey
    JOIN DimTime dt ON ft.TimeKey = dt.TimeKey
    WHERE dt.FullDate >= DATEADD(MONTH, -1, GETDATE())
    GROUP BY p.CategoryKey, dt.FullDate
)
SELECT 
    wd.CategoryName,
    AVG(wd.AvgDwellMinutes) AS AvgWarehouseDwell,
    AVG(fi.AvgIdleMinutes) AS AvgFleetIdle,
    CORR(wd.AvgDwellMinutes, fi.AvgIdleMinutes) AS Correlation
FROM WarehouseDwell wd
JOIN FleetIdle fi ON wd.CategoryKey = fi.CategoryKey AND wd.FullDate = fi.FullDate
GROUP BY wd.CategoryName
ORDER BY Correlation DESC;
```

### Predictive Bottleneck Detection
```sql
-- Identify potential bottlenecks using rolling averages
WITH OperationMetrics AS (
    SELECT 
        fo.WarehouseKey,
        w.WarehouseName,
        dt.FullDate,
        dt.HourOfDay,
        COUNT(*) AS OperationCount,
        AVG(fo.CycleTimeSeconds) AS AvgCycleTime,
        AVG(fo.ErrorCount) AS AvgErrors
    FROM FactWarehouseOperations fo
    JOIN DimWarehouse w ON fo.WarehouseKey = w.WarehouseKey
    JOIN DimTime dt ON fo.TimeKey = dt.TimeKey
    WHERE dt.FullDate >= DATEADD(DAY, -7, GETDATE())
    GROUP BY fo.WarehouseKey, w.WarehouseName, dt.FullDate, dt.HourOfDay
),
RollingAvg AS (
    SELECT 
        WarehouseKey,
        WarehouseName,
        FullDate,
        HourOfDay,
        AvgCycleTime,
        AVG(AvgCycleTime) OVER (
            PARTITION BY WarehouseKey, HourOfDay 
            ORDER BY FullDate 
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) AS RollingAvgCycleTime,
        STDEV(AvgCycleTime) OVER (
            PARTITION BY WarehouseKey, HourOfDay 
            ORDER BY FullDate 
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) AS StdDevCycleTime
    FROM OperationMetrics
)
SELECT 
    WarehouseName,
    FullDate,
    HourOfDay,
    AvgCycleTime,
    RollingAvgCycleTime,
    CASE 
        WHEN AvgCycleTime > RollingAvgCycleTime + (2 * StdDevCycleTime) 
        THEN 'HIGH_RISK'
        WHEN AvgCycleTime > RollingAvgCycleTime + StdDevCycleTime 
        THEN 'MODERATE_RISK'
        ELSE 'NORMAL'
    END AS BottleneckRisk
FROM RollingAvg
WHERE FullDate = CAST(GETDATE() AS DATE)
ORDER BY BottleneckRisk DESC, AvgCycleTime DESC;
```

### Fleet Fuel Efficiency Analysis
```sql
-- Calculate fuel efficiency by route and vehicle type
SELECT 
    r.RouteCode,
    r.RouteName,
    v.VehicleType,
    COUNT(ft.TripID) AS TotalTrips,
    SUM(ft.DistanceMiles) AS TotalMiles,
    SUM(ft.FuelConsumedGallons) AS TotalFuel,
    AVG(ft.DistanceMiles / NULLIF(ft.FuelConsumedGallons, 0)) AS AvgMPG,
    AVG(ft.IdleTimeMinutes * 100.0 / NULLIF(DATEDIFF(MINUTE, ft.TripStartTime, ft.TripEndTime), 0)) AS IdlePercentage,
    SUM(ft.TripCost) AS TotalCost
FROM FactFleetTrips ft
JOIN DimRoute r ON ft.RouteKey = r.RouteKey
JOIN DimVehicle v ON ft.VehicleKey = v.VehicleKey
JOIN DimTime dt ON ft.TimeKey = dt.TimeKey
WHERE dt.FullDate >= DATEADD(MONTH, -3, GETDATE())
  AND ft.DeliveryStatus = 'OnTime'
GROUP BY r.RouteCode, r.RouteName, v.VehicleType
ORDER BY AvgMPG DESC;
```

## Stored Procedures for Data Loading

### Incremental Warehouse Operations Load
```sql
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadTime DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Default to last 15 minutes if not specified
    IF @LastLoadTime IS NULL
        SET @LastLoadTime = DATEADD(MINUTE, -15, GETDATE());
    
    INSERT INTO FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductKey, SupplierKey,
        OperationType, Quantity, DwellTimeMinutes, 
        CycleTimeSeconds, ErrorCount, OperationCost
    )
    SELECT 
        CONVERT(INT, FORMAT(wms.OperationDateTime, 'yyyyMMddHHmm')),
        w.WarehouseKey,
        p.ProductKey,
        s.SupplierKey,
        wms.OperationType,
        wms.Quantity,
        DATEDIFF(MINUTE, wms.StartTime, wms.EndTime),
        DATEDIFF(SECOND, wms.StartTime, wms.EndTime),
        wms.ErrorCount,
        wms.LaborCost + wms.EquipmentCost
    FROM ExternalWarehouseData.dbo.WMS_Operations wms
    JOIN DimWarehouse w ON wms.WarehouseCode = w.WarehouseCode
    JOIN DimProduct p ON wms.SKU = p.SKU
    LEFT JOIN DimSupplier s ON wms.SupplierCode = s.SupplierCode
    WHERE wms.OperationDateTime > @LastLoadTime
      AND wms.IsProcessed = 0;
    
    -- Mark as processed
    UPDATE ExternalWarehouseData.dbo.WMS_Operations
    SET IsProcessed = 1
    WHERE OperationDateTime > @LastLoadTime;
END;
GO
```

### Fleet Telemetry Ingestion
```sql
CREATE PROCEDURE sp_LoadFleetTrips
    @LastLoadTime DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    IF @LastLoadTime IS NULL
        SET @LastLoadTime = DATEADD(MINUTE, -15, GETDATE());
    
    INSERT INTO FactFleetTrips (
        TimeKey, VehicleKey, DriverKey, RouteKey,
        OriginWarehouseKey, DestinationKey,
        TripStartTime, TripEndTime, DistanceMiles,
        FuelConsumedGallons, IdleTimeMinutes, LoadWeight,
        DeliveryStatus, TripCost
    )
    SELECT 
        CONVERT(INT, FORMAT(tel.TripStartTime, 'yyyyMMddHHmm')),
        v.VehicleKey,
        d.DriverKey,
        r.RouteKey,
        wo.WarehouseKey AS OriginKey,
        wd.WarehouseKey AS DestKey,
        tel.TripStartTime,
        tel.TripEndTime,
        tel.TotalMiles,
        tel.FuelUsed,
        tel.IdleMinutes,
        tel.CargoWeight,
        CASE 
            WHEN tel.ActualArrival <= tel.ScheduledArrival THEN 'OnTime'
            WHEN tel.ActualArrival > tel.ScheduledArrival THEN 'Delayed'
            ELSE 'Failed'
        END,
        tel.FuelCost + tel.TollCost + tel.DriverWage
    FROM ExternalFleetData.dbo.Telemetry tel
    JOIN DimVehicle v ON tel.VehicleVIN = v.VIN
    JOIN DimDriver d ON tel.DriverID = d.DriverExternalID
    JOIN DimRoute r ON tel.RouteCode = r.RouteCode
    LEFT JOIN DimWarehouse wo ON tel.OriginCode = wo.WarehouseCode
    LEFT JOIN DimWarehouse wd ON tel.DestinationCode = wd.WarehouseCode
    WHERE tel.TripStartTime > @LastLoadTime
      AND tel.IsLoaded = 0;
    
    UPDATE ExternalFleetData.dbo.Telemetry
    SET IsLoaded = 1
    WHERE TripStartTime > @LastLoadTime;
END;
GO
```

## Power BI Integration

### Connecting to SQL Server
1. Open Power BI Desktop
2. Get Data → SQL Server
3. Server: `${SQL_SERVER_HOST}`
4. Database: `LogiFleetPulse`
5. Data Connectivity mode: **DirectQuery** for real-time or **Import** for better performance

### DAX Measures for KPIs

#### Warehouse Efficiency Score
```dax
Warehouse Efficiency Score = 
VAR TotalOperations = COUNTROWS(FactWarehouseOperations)
VAR ErrorRate = DIVIDE(SUM(FactWarehouseOperations[ErrorCount]), TotalOperations, 0)
VAR AvgCycleTime = AVERAGE(FactWarehouseOperations[CycleTimeSeconds])
VAR BenchmarkCycleTime = 180 -- 3 minutes benchmark
VAR CycleEfficiency = DIVIDE(BenchmarkCycleTime, AvgCycleTime, 0)
VAR ErrorPenalty = 1 - ErrorRate
RETURN
    ROUND(CycleEfficiency * ErrorPenalty * 100, 2)
```

#### Fleet Utilization Rate
```dax
Fleet Utilization % = 
VAR TotalTripTime = 
    SUMX(
        FactFleetTrips,
        DATEDIFF(FactFleetTrips[TripStartTime], FactFleetTrips[TripEndTime], MINUTE)
    )
VAR TotalIdleTime = SUM(FactFleetTrips[IdleTimeMinutes])
VAR ActiveTime = TotalTripTime - TotalIdleTime
RETURN
    DIVIDE(ActiveTime, TotalTripTime, 0) * 100
```

#### Cross-Fact: Delivery Cost per Warehouse Operation
```dax
Cost Per Operation = 
VAR WarehouseCost = SUM(FactWarehouseOperations[OperationCost])
VAR FleetCost = 
    CALCULATE(
        SUM(FactFleetTrips[TripCost]),
        USERELATIONSHIP(FactFleetTrips[OriginWarehouseKey], DimWarehouse[WarehouseKey])
    )
VAR TotalOperations = COUNTROWS(FactWarehouseOperations)
RETURN
    DIVIDE(WarehouseCost + FleetCost, TotalOperations, 0)
```

### Dynamic Time Intelligence
```dax
YoY Dwell Time Change % = 
VAR CurrentPeriod = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR PriorYear = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        SAMEPERIODLASTYEAR(DimTime[FullDate])
    )
RETURN
    DIVIDE(CurrentPeriod - PriorYear, PriorYear, 0) * 100
```

## Automated Alerting System

### Configure Threshold-Based Alerts
```sql
CREATE TABLE AlertConfiguration (
    AlertID INT PRIMARY KEY IDENTITY(1,1),
    AlertName VARCHAR(200),
    MetricType VARCHAR(100), -- 'DwellTime', 'IdleTime', 'ErrorRate', etc.
    ThresholdValue DECIMAL(18,2),
    ComparisonOperator VARCHAR(10), -- '>', '<', '>=', '<=', '='
    RecipientEmails VARCHAR(MAX),
    IsActive BIT DEFAULT 1
);

-- Insert sample alert configuration
INSERT INTO AlertConfiguration (AlertName, MetricType, ThresholdValue, ComparisonOperator, RecipientEmails)
VALUES 
    ('High Dwell Time Alert', 'DwellTime', 120, '>', 'warehouse-ops@company.com'),
    ('Excessive Fleet Idle', 'IdleTime', 30, '>', 'fleet-manager@company.com'),
    ('Low Efficiency Score', 'WarehouseEfficiency', 75, '<', 'ops-director@company.com');
```

### Alert Execution Procedure
```sql
CREATE PROCEDURE sp_CheckAndSendAlerts
AS
BEGIN
    DECLARE @AlertID INT, @MetricType VARCHAR(100), @Threshold DECIMAL(18,2), 
            @Operator VARCHAR(10), @Recipients VARCHAR(MAX);
    
    DECLARE alert_cursor CURSOR FOR
    SELECT AlertID, MetricType, ThresholdValue, ComparisonOperator, RecipientEmails
    FROM AlertConfiguration
    WHERE IsActive = 1;
    
    OPEN alert_cursor;
    FETCH NEXT FROM alert_cursor INTO @AlertID, @MetricType, @Threshold, @Operator, @Recipients;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        IF @MetricType = 'DwellTime'
        BEGIN
            DECLARE @CurrentDwell DECIMAL(18,2) = (
                SELECT AVG(DwellTimeMinutes)
                FROM FactWarehouseOperations
                WHERE TimeKey >= CONVERT(INT, FORMAT(DATEADD(HOUR, -1, GETDATE()), 'yyyyMMddHHmm'))
            );
            
            IF (@Operator = '>' AND @CurrentDwell > @Threshold)
            BEGIN
                EXEC msdb.dbo.sp_send_dbmail
                    @recipients = @Recipients,
                    @subject = 'Alert: High Warehouse Dwell Time',
                    @body = CONCAT('Average dwell time in last hour: ', @CurrentDwell, ' minutes (Threshold: ', @Threshold, ')');
            END
        END
        
        -- Add similar logic for other metric types
        
        FETCH NEXT FROM alert_cursor INTO @AlertID, @MetricType, @Threshold, @Operator, @Recipients;
    END
    
    CLOSE alert_cursor;
    DEALLOCATE alert_cursor;
END;
GO

-- Schedule via SQL Agent Job to run every 15 minutes
```

## Configuration Management

### Environment-Specific Settings
```sql
CREATE TABLE SystemConfiguration (
    ConfigKey VARCHAR(100) PRIMARY KEY,
    ConfigValue VARCHAR(500),
    ConfigType VARCHAR(50), -- 'Connection', 'API', 'Alert', 'Calculation'
    Description VARCHAR(500),
    LastUpdated DATETIME DEFAULT GETDATE()
);

INSERT INTO SystemConfiguration (ConfigKey, ConfigValue, ConfigType, Description)
VALUES 
    ('DataRefreshInterval', '15', 'System', 'Minutes between incremental data loads'),
    ('AlertCheckInterval', '15', 'Alert', 'Minutes between alert checks'),
    ('BottleneckThresholdStdDev', '2', 'Calculation', 'Standard deviations for bottleneck detection'),
    ('GravityScoreLookbackDays', '90', 'Calculation', 'Days of history for gravity score calculation'),
    ('WMS_API_Endpoint', '${WMS_API_ENDPOINT}', 'Connection', 'Warehouse Management System API URL'),
    ('Fleet_API_Endpoint', '${FLEET_TELEMETRY_URL}', 'Connection', 'Fleet telemetry feed URL');
```

## Troubleshooting

### Performance Optimization

**Slow queries on time-based aggregations:**
```sql
-- Ensure time dimension is properly indexed
CREATE NONCLUSTERED INDEX IX_DimTime_Date_Hour 
ON DimTime(FullDate, HourOfDay) INCLUDE (DayOfWeek, IsWeekend);

-- Partition fact tables by month
ALTER PARTITION SCHEME ps_Monthly_Facts 
NEXT USED [PRIMARY];

ALTER TABLE FactWarehouseOperations 
SWITCH PARTITION $PARTITION.pf_Monthly(20260101) 
TO FactWarehouseOperations_Archive PARTITION $PARTITION.pf_Monthly(20260101);
```

**Power BI report refresh timeout:**
```dax
-- Use incremental refresh configuration
-- In Power BI Desktop: Table → Incremental refresh settings
-- RangeStart and RangeEnd parameters
Table.SelectRows(
    FactWarehouseOperations, 
    each [OperationDateTime] >= RangeStart and [OperationDateTime] < RangeEnd
)
```

### Data Quality Issues

**Missing dimension keys:**
```sql
-- Find orphaned fact records
SELECT 'FactWarehouseOperations' AS TableName, COUNT(*) AS OrphanCount
FROM FactWarehouseOperations fo
LEFT JOIN DimProduct p ON fo.ProductKey = p.ProductKey
WHERE p.ProductKey IS NULL

UNION ALL

SELECT 'FactFleetTrips', COUNT(*)
FROM FactFleetTrips ft
LEFT JOIN DimVehicle v ON ft.VehicleKey = v.VehicleKey
WHERE v.VehicleKey IS NULL;

-- Create default "Unknown" dimension records
INSERT INTO DimProduct (SKU, ProductName, GravityScore)
VALUES ('UNKNOWN', 'Unknown Product', 0);

INSERT INTO DimVehicle (VIN, VehicleType)
VALUES ('UNKNOWN', 'Unknown Vehicle');
```

**Duplicate records in fact tables:**
```sql
-- Identify duplicates
WITH DuplicateOps AS (
    SELECT 
        WarehouseKey, ProductKey, TimeKey, OperationType,
        ROW_NUMBER() OVER (
            PARTITION BY WarehouseKey, ProductKey, TimeKey, OperationType 
            ORDER BY OperationID DESC
        ) AS RowNum
    FROM FactWarehouseOperations
)
DELETE FROM DuplicateOps WHERE RowNum > 1;

-- Add unique constraint to prevent future duplicates
CREATE UNIQUE NONCLUSTERED INDEX UX_FactWarehouse_Business_Key
ON FactWarehouseOperations(WarehouseKey, ProductKey, TimeKey, OperationType);
```

### Common Errors

**"Cannot resolve the collation conflict" error:**
```sql
-- Standardize collation across all text columns
ALTER TABLE DimProduct ALTER COLUMN SKU VARCHAR(50) COLLATE SQL_Latin1_General_CP1_CI_AS;
ALTER TABLE DimWarehouse ALTER COLUMN WarehouseCode VARCHAR(50) COLLATE SQL_Latin1_General_CP1_CI_AS;
```

**Power BI gateway connection failures:**
- Ensure the gateway service account has db_datareader role
- Check firewall rules allow port 1433 (default SQL Server)
- Verify connection string matches gateway configuration:
```
Server=${SQL_SERVER_HOST};Database=LogiFleetPulse;Trusted_Connection=True;
```

**Time dimension not aligning with fact records:**
```sql
-- Verify time key format consistency
SELECT TOP 10
    TimeKey,
    FORMAT(CAST(CAST(TimeKey AS VARCHAR(12)) AS DATETIME), 'yyyyMMddHHmm') AS FormattedKey
FROM DimTime
WHERE TimeKey NOT IN (SELECT DISTINCT TimeKey FROM FactWarehouseOperations);
```

## Advanced Patterns

### Many-to-Many Bridge Table for Routes and Storage Zones
```sql
CREATE TABLE BridgeRouteStorageZone (
    RouteKey INT NOT NULL,
    StorageZoneKey INT NOT NULL,
    AllocationPercentage DECIMAL(5,2),
    PRIMARY KEY (RouteKey, StorageZoneKey),
    CONSTRAINT FK_Bridge_Route FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey),
    CONSTRAINT FK_Bridge_Zone FOREIGN KEY (StorageZoneKey) REFERENCES DimStorageZone(StorageZoneKey)
);

-- Query using bridge table
SELECT 
    r.RouteName,
    sz.ZoneName,
    AVG(fo.DwellTimeMinutes) AS AvgDwell,
    AVG(ft.IdleTimeMinutes) AS AvgIdle,
    br.AllocationPercentage
FROM BridgeRouteStorageZone br
JOIN DimRoute r ON br.RouteKey = r.RouteKey
JOIN DimStorageZone sz ON br.StorageZoneKey = sz.StorageZoneKey
JOIN FactWarehouseOperations fo ON fo.StorageZoneKey = sz.StorageZoneKey
JOIN FactFleetTrips ft ON ft.RouteKey = r.RouteKey
GROUP BY r.RouteName, sz.ZoneName, br.AllocationPercentage;
```

### Snapshot Fact Table for Inventory Levels
```sql
CREATE TABLE FactInventorySnapshot (
    SnapshotID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    StorageZoneKey INT,
    QuantityOnHand DECIMAL(18,2),
    QuantityAllocated DECIMAL(18
