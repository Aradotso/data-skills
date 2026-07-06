---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing platform for logistics, fleet management, and supply chain analytics with multi-fact star schema
triggers:
  - set up supply chain analytics dashboard
  - configure logifleet pulse warehouse tracking
  - implement fleet logistics data model
  - create power bi supply chain reports
  - deploy ms sql logistics schema
  - build warehouse and fleet analytics
  - integrate logistics intelligence platform
  - configure cross-modal supply chain analytics
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive data warehousing and business intelligence platform for logistics and supply chain management. It combines:

- **Multi-fact star schema** for warehouse operations, fleet trips, and cross-dock activities
- **MS SQL Server backend** with time-phased dimensions and bridge tables
- **Power BI dashboards** for real-time visualization and KPI tracking
- **Predictive analytics** for bottleneck detection and fleet optimization
- **Cross-fact KPI harmonization** linking inventory, fleet, and supplier metrics

The platform ingests data from WMS, telematics, supplier portals, and external APIs to provide unified logistics intelligence.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019 or later
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Network access to your WMS, TMS, and telemetry data sources

### Step 1: Deploy SQL Schema

1. Clone or download the repository
2. Connect to your SQL Server instance via SSMS
3. Execute the schema deployment script:

```sql
-- Run the main schema creation script
-- Adjust database name as needed
USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute the provided schema scripts in order:
-- 1. Create dimension tables
-- 2. Create fact tables
-- 3. Create bridge tables
-- 4. Create views and stored procedures
-- 5. Create indexes and constraints
```

### Step 2: Configure Data Sources

Create a configuration file for connection strings:

```json
{
  "dataSources": {
    "wmsConnection": "${WMS_CONNECTION_STRING}",
    "telematicsApi": "${TELEMETRY_API_ENDPOINT}",
    "telematicsApiKey": "${TELEMETRY_API_KEY}",
    "erpConnection": "${ERP_CONNECTION_STRING}",
    "weatherApi": "${WEATHER_API_ENDPOINT}",
    "weatherApiKey": "${WEATHER_API_KEY}"
  },
  "refreshSchedule": {
    "warehouseOperations": "*/15 * * * *",
    "fleetTrips": "*/15 * * * *",
    "crossDock": "*/30 * * * *",
    "dimensions": "0 2 * * *"
  }
}
```

### Step 3: Import Power BI Template

1. Open Power BI Desktop
2. File → Import → Power BI Template
3. Select `LogiFleet_Pulse_Master.pbit`
4. Enter your SQL Server connection parameters
5. Configure row-level security roles as needed

## Core Data Model Architecture

### Dimension Tables

```sql
-- DimTime: 15-minute granularity time dimension
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME2 NOT NULL,
    Date DATE NOT NULL,
    TimeOfDay TIME NOT NULL,
    Hour TINYINT NOT NULL,
    QuarterHour TINYINT NOT NULL,
    DayOfWeek TINYINT NOT NULL,
    WeekOfYear TINYINT NOT NULL,
    Month TINYINT NOT NULL,
    Quarter TINYINT NOT NULL,
    FiscalYear INT NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT NOT NULL
);
CREATE CLUSTERED INDEX IX_DimTime_DateTime ON DimTime(DateTime);

-- DimGeography: Hierarchical location dimension
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID NVARCHAR(50) NOT NULL,
    LocationName NVARCHAR(200) NOT NULL,
    LocationType NVARCHAR(50) NOT NULL, -- Warehouse, Route Node, Distribution Center
    StreetAddress NVARCHAR(300),
    City NVARCHAR(100),
    Region NVARCHAR(100),
    Country NVARCHAR(100),
    Continent NVARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone NVARCHAR(50),
    IsActive BIT DEFAULT 1
);
CREATE INDEX IX_DimGeography_LocationID ON DimGeography(LocationID);

-- DimProductGravity: Product classification with gravity scoring
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU NVARCHAR(50) NOT NULL UNIQUE,
    ProductName NVARCHAR(200) NOT NULL,
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    UnitWeight DECIMAL(10,2),
    UnitVolume DECIMAL(10,2),
    IsFragile BIT DEFAULT 0,
    IsPerishable BIT DEFAULT 0,
    TemperatureMin DECIMAL(5,2),
    TemperatureMax DECIMAL(5,2),
    VelocityScore DECIMAL(5,2), -- Calculated from historical movement
    ValueScore DECIMAL(5,2), -- Based on unit price
    GravityScore AS (VelocityScore * 0.5 + ValueScore * 0.3 + CASE WHEN IsFragile = 1 THEN 20 ELSE 0 END),
    RecommendedZone NVARCHAR(50)
);
CREATE INDEX IX_DimProductGravity_SKU ON DimProductGravity(SKU);

-- DimSupplierReliability: Supplier performance metrics
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID NVARCHAR(50) NOT NULL UNIQUE,
    SupplierName NVARCHAR(200) NOT NULL,
    LeadTimeMean DECIMAL(5,2),
    LeadTimeStdDev DECIMAL(5,2),
    DefectRate DECIMAL(5,4),
    ComplianceScore DECIMAL(5,2),
    ReliabilityRating AS (
        100 - (LeadTimeStdDev * 10) - (DefectRate * 1000) + ComplianceScore
    )
);
```

### Fact Tables

```sql
-- FactWarehouseOperations: Core warehouse activity fact
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType NVARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    OrderID NVARCHAR(50),
    BatchID NVARCHAR(50),
    Quantity INT NOT NULL,
    DwellTimeMinutes INT,
    ProcessingTimeMinutes INT,
    StorageZone NVARCHAR(50),
    EmployeeID NVARCHAR(50),
    IsCompleted BIT DEFAULT 0,
    HasException BIT DEFAULT 0,
    ExceptionReason NVARCHAR(500)
);
CREATE INDEX IX_FactWarehouseOps_Time ON FactWarehouseOperations(TimeKey);
CREATE INDEX IX_FactWarehouseOps_Product ON FactWarehouseOperations(ProductKey);

-- FactFleetTrips: Fleet and route performance fact
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleID NVARCHAR(50) NOT NULL,
    DriverID NVARCHAR(50),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DistanceKm DECIMAL(10,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(10,2),
    LoadWeightKg DECIMAL(10,2),
    DelayMinutes INT,
    DelayReason NVARCHAR(200),
    WeatherCondition NVARCHAR(100),
    TrafficCondition NVARCHAR(100),
    MaintenanceAlerts INT DEFAULT 0,
    TirePressureStatus NVARCHAR(50),
    EngineStatus NVARCHAR(50)
);
CREATE INDEX IX_FactFleetTrips_Time ON FactFleetTrips(TimeKey);
CREATE INDEX IX_FactFleetTrips_Vehicle ON FactFleetTrips(VehicleID);

-- FactCrossDock: Cross-docking operations
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    Quantity INT NOT NULL,
    TransferTimeMinutes INT,
    IsDirectTransfer BIT DEFAULT 0
);
```

## Data Loading Patterns

### Incremental Loading Stored Procedure

```sql
CREATE PROCEDURE usp_LoadWarehouseOperations
    @StartTime DATETIME2,
    @EndTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Load from external WMS source
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        OrderID, BatchID, Quantity, DwellTimeMinutes,
        ProcessingTimeMinutes, StorageZone, EmployeeID,
        IsCompleted, HasException, ExceptionReason
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        wms.operation_type,
        wms.order_id,
        wms.batch_id,
        wms.quantity,
        DATEDIFF(MINUTE, wms.start_time, wms.end_time) AS DwellTimeMinutes,
        wms.processing_minutes,
        wms.zone_code,
        wms.employee_id,
        CASE WHEN wms.status = 'Completed' THEN 1 ELSE 0 END,
        CASE WHEN wms.exception_flag = 'Y' THEN 1 ELSE 0 END,
        wms.exception_notes
    FROM OPENROWSET(
        'SQLNCLI',
        '${WMS_CONNECTION_STRING}',
        'SELECT * FROM wms_operations WHERE timestamp BETWEEN ? AND ?',
        @StartTime, @EndTime
    ) AS wms
    INNER JOIN DimTime t ON CAST(wms.timestamp AS DATETIME2) = t.DateTime
    INNER JOIN DimGeography g ON wms.warehouse_id = g.LocationID
    INNER JOIN DimProductGravity p ON wms.sku = p.SKU;
    
    PRINT 'Loaded ' + CAST(@@ROWCOUNT AS NVARCHAR(20)) + ' warehouse operations';
END;
GO

-- Schedule execution
EXEC usp_LoadWarehouseOperations 
    @StartTime = '2026-07-06 00:00:00',
    @EndTime = '2026-07-06 23:59:59';
```

### Fleet Telemetry Integration

```sql
CREATE PROCEDURE usp_LoadFleetTripsFromAPI
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @ApiEndpoint NVARCHAR(500) = '${TELEMETRY_API_ENDPOINT}';
    DECLARE @ApiKey NVARCHAR(200) = '${TELEMETRY_API_KEY}';
    
    -- Using JSON parsing for API response
    DECLARE @JsonResponse NVARCHAR(MAX);
    
    -- Execute HTTP request (requires CLR integration or external tool)
    -- This is a simplified example; actual implementation varies
    
    INSERT INTO FactFleetTrips (
        TimeKey, VehicleID, DriverID, OriginGeographyKey,
        DestinationGeographyKey, DistanceKm, DurationMinutes,
        IdleTimeMinutes, FuelConsumedLiters, LoadWeightKg,
        DelayMinutes, DelayReason, MaintenanceAlerts
    )
    SELECT 
        t.TimeKey,
        json.vehicle_id,
        json.driver_id,
        orig.GeographyKey,
        dest.GeographyKey,
        json.distance_km,
        json.duration_minutes,
        json.idle_time_minutes,
        json.fuel_liters,
        json.load_weight_kg,
        json.delay_minutes,
        json.delay_reason,
        json.maintenance_alert_count
    FROM OPENJSON(@JsonResponse, '$.trips') WITH (
        trip_id NVARCHAR(50),
        vehicle_id NVARCHAR(50),
        driver_id NVARCHAR(50),
        origin_location_id NVARCHAR(50),
        destination_location_id NVARCHAR(50),
        start_time DATETIME2,
        distance_km DECIMAL(10,2),
        duration_minutes INT,
        idle_time_minutes INT,
        fuel_liters DECIMAL(10,2),
        load_weight_kg DECIMAL(10,2),
        delay_minutes INT,
        delay_reason NVARCHAR(200),
        maintenance_alert_count INT
    ) AS json
    INNER JOIN DimTime t ON json.start_time = t.DateTime
    INNER JOIN DimGeography orig ON json.origin_location_id = orig.LocationID
    INNER JOIN DimGeography dest ON json.destination_location_id = dest.LocationID;
END;
GO
```

## Key Analytical Queries

### Cross-Fact KPI: Dwell Time vs Fleet Idle Cost

```sql
-- Correlate warehouse dwell time with fleet idle time by product
WITH WarehouseDwell AS (
    SELECT 
        p.SKU,
        p.ProductName,
        AVG(wo.DwellTimeMinutes) AS AvgDwellMinutes,
        SUM(wo.Quantity) AS TotalQuantity
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    WHERE wo.OperationType = 'Putaway'
        AND wo.TimeKey >= (SELECT TimeKey FROM DimTime WHERE Date = DATEADD(DAY, -30, CAST(GETDATE() AS DATE)))
    GROUP BY p.SKU, p.ProductName
),
FleetIdle AS (
    SELECT 
        p.SKU,
        AVG(ft.IdleTimeMinutes) AS AvgIdleMinutes,
        AVG(ft.FuelConsumedLiters * 1.5) AS AvgIdleCostUSD -- Assuming $1.50/liter
    FROM FactFleetTrips ft
    INNER JOIN FactCrossDock cd ON ft.TripKey = cd.OutboundTripKey
    INNER JOIN DimProductGravity p ON cd.ProductKey = p.ProductKey
    WHERE ft.TimeKey >= (SELECT TimeKey FROM DimTime WHERE Date = DATEADD(DAY, -30, CAST(GETDATE() AS DATE)))
    GROUP BY p.SKU
)
SELECT 
    wd.SKU,
    wd.ProductName,
    wd.AvgDwellMinutes,
    wd.TotalQuantity,
    fi.AvgIdleMinutes,
    fi.AvgIdleCostUSD,
    (wd.AvgDwellMinutes * fi.AvgIdleCostUSD / 60) AS EstimatedCostImpact
FROM WarehouseDwell wd
LEFT JOIN FleetIdle fi ON wd.SKU = fi.SKU
WHERE wd.AvgDwellMinutes > 120 -- Products with >2 hour dwell
ORDER BY EstimatedCostImpact DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify products that should be reassigned to different zones
WITH ProductMovement AS (
    SELECT 
        p.ProductKey,
        p.SKU,
        p.ProductName,
        p.GravityScore,
        p.RecommendedZone,
        wo.StorageZone AS CurrentZone,
        COUNT(*) AS PickCount,
        AVG(wo.ProcessingTimeMinutes) AS AvgPickTime
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    WHERE wo.OperationType = 'Picking'
        AND wo.TimeKey >= (SELECT TimeKey FROM DimTime WHERE Date = DATEADD(DAY, -14, CAST(GETDATE() AS DATE)))
    GROUP BY p.ProductKey, p.SKU, p.ProductName, p.GravityScore, p.RecommendedZone, wo.StorageZone
)
SELECT 
    SKU,
    ProductName,
    GravityScore,
    CurrentZone,
    RecommendedZone,
    PickCount,
    AvgPickTime,
    CASE 
        WHEN GravityScore > 70 AND CurrentZone NOT LIKE 'A%' THEN 'Move to Zone A (High Priority)'
        WHEN GravityScore BETWEEN 40 AND 70 AND CurrentZone NOT LIKE 'B%' THEN 'Move to Zone B (Medium Priority)'
        WHEN GravityScore < 40 AND CurrentZone NOT LIKE 'C%' THEN 'Move to Zone C (Low Priority)'
        ELSE 'Current zone optimal'
    END AS RecommendedAction
FROM ProductMovement
WHERE CurrentZone <> RecommendedZone
ORDER BY GravityScore DESC;
```

### Predictive Bottleneck Detection

```sql
-- Identify potential bottlenecks using time-series analysis
WITH HourlyMetrics AS (
    SELECT 
        t.Date,
        t.Hour,
        g.LocationName,
        COUNT(*) AS OperationCount,
        AVG(wo.ProcessingTimeMinutes) AS AvgProcessTime,
        STDEV(wo.ProcessingTimeMinutes) AS StdDevProcessTime
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
    WHERE t.Date >= DATEADD(DAY, -7, CAST(GETDATE() AS DATE))
    GROUP BY t.Date, t.Hour, g.LocationName
),
Baseline AS (
    SELECT 
        Hour,
        LocationName,
        AVG(AvgProcessTime) AS BaselineProcessTime,
        AVG(StdDevProcessTime) AS BaselineStdDev
    FROM HourlyMetrics
    GROUP BY Hour, LocationName
)
SELECT 
    hm.Date,
    hm.Hour,
    hm.LocationName,
    hm.AvgProcessTime,
    b.BaselineProcessTime,
    (hm.AvgProcessTime - b.BaselineProcessTime) AS Deviation,
    CASE 
        WHEN hm.AvgProcessTime > (b.BaselineProcessTime + 2 * b.BaselineStdDev) THEN 'Critical'
        WHEN hm.AvgProcessTime > (b.BaselineProcessTime + b.BaselineStdDev) THEN 'Warning'
        ELSE 'Normal'
    END AS BottleneckStatus
FROM HourlyMetrics hm
INNER JOIN Baseline b ON hm.Hour = b.Hour AND hm.LocationName = b.LocationName
WHERE hm.Date = CAST(GETDATE() AS DATE)
    AND hm.AvgProcessTime > (b.BaselineProcessTime + b.BaselineStdDev)
ORDER BY Deviation DESC;
```

## Power BI Configuration

### DAX Measures for Cross-Fact Analysis

```dax
// Warehouse Velocity Score
Warehouse Velocity = 
VAR TotalPicks = COUNTROWS(FactWarehouseOperations)
VAR AvgPickTime = AVERAGE(FactWarehouseOperations[ProcessingTimeMinutes])
RETURN
    DIVIDE(TotalPicks, AvgPickTime, 0) * 100

// Fleet Efficiency Ratio
Fleet Efficiency = 
VAR ActualTime = SUM(FactFleetTrips[DurationMinutes])
VAR IdleTime = SUM(FactFleetTrips[IdleTimeMinutes])
RETURN
    DIVIDE(ActualTime - IdleTime, ActualTime, 0)

// Composite Supply Chain Score
Supply Chain Health = 
VAR WarehouseScore = [Warehouse Velocity]
VAR FleetScore = [Fleet Efficiency] * 100
VAR SupplierScore = AVERAGE(DimSupplierReliability[ReliabilityRating])
RETURN
    (WarehouseScore * 0.4 + FleetScore * 0.4 + SupplierScore * 0.2)

// Time-Phased Dwell Analysis
Dwell Time Trend = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    DATESINPERIOD(
        DimTime[Date],
        MAX(DimTime[Date]),
        -30,
        DAY
    )
)
```

### Row-Level Security Configuration

```dax
// Role: Regional Manager (sees only their region)
[Region] = USERNAME()

// Role: Warehouse Supervisor (sees only their warehouse)
DimGeography[LocationID] = LOOKUPVALUE(
    EmployeeWarehouse[WarehouseID],
    EmployeeWarehouse[EmployeeEmail],
    USERNAME()
)
```

## Automated Alerting System

```sql
-- Create alert monitoring procedure
CREATE PROCEDURE usp_MonitorKPIThresholds
AS
BEGIN
    DECLARE @AlertTable TABLE (
        AlertType NVARCHAR(100),
        Severity NVARCHAR(20),
        Message NVARCHAR(500),
        Value DECIMAL(10,2),
        Threshold DECIMAL(10,2)
    );
    
    -- Check fleet idle time threshold (>15%)
    INSERT INTO @AlertTable
    SELECT 
        'Fleet Idle Time Excessive',
        'Warning',
        'Vehicle ' + VehicleID + ' exceeded idle time threshold',
        CAST(IdleTimeMinutes AS DECIMAL(10,2)) / NULLIF(DurationMinutes, 0) * 100,
        15.0
    FROM FactFleetTrips
    WHERE TimeKey >= (SELECT TimeKey FROM DimTime WHERE DateTime >= DATEADD(HOUR, -1, GETDATE()))
        AND CAST(IdleTimeMinutes AS DECIMAL(10,2)) / NULLIF(DurationMinutes, 0) > 0.15;
    
    -- Check warehouse dwell time threshold (>72 hours)
    INSERT INTO @AlertTable
    SELECT 
        'Warehouse Dwell Time Critical',
        'Critical',
        'SKU ' + p.SKU + ' in zone ' + wo.StorageZone + ' exceeds dwell threshold',
        wo.DwellTimeMinutes,
        4320.0 -- 72 hours in minutes
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    WHERE wo.DwellTimeMinutes > 4320
        AND wo.IsCompleted = 0;
    
    -- Send alerts (integration with email/Teams required)
    IF EXISTS (SELECT 1 FROM @AlertTable)
    BEGIN
        SELECT * FROM @AlertTable ORDER BY Severity DESC;
        -- Execute notification logic here
    END
END;
GO

-- Schedule via SQL Agent
-- Job step: EXEC usp_MonitorKPIThresholds
-- Schedule: Every 15 minutes
```

## Troubleshooting Common Issues

### Issue: Power BI Refresh Fails

**Symptom**: "Unable to connect to data source" error

**Solution**:
1. Verify SQL Server connection string in Power BI
2. Check firewall rules allow Power BI service IP ranges
3. Ensure SQL login has db_datareader permissions
4. Test connection using SQL Server Management Studio first

```sql
-- Grant necessary permissions
USE LogiFleetPulse;
GO
CREATE USER [powerbi_service] FOR LOGIN [powerbi_service];
ALTER ROLE db_datareader ADD MEMBER [powerbi_service];
GRANT EXECUTE ON SCHEMA::dbo TO [powerbi_service];
```

### Issue: Slow Query Performance

**Symptom**: Dashboards take >30 seconds to load

**Solution**: Add covering indexes for common query patterns

```sql
-- Index for time-based warehouse queries
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time_Product_INCLUDE
ON FactWarehouseOperations(TimeKey, ProductKey)
INCLUDE (Quantity, DwellTimeMinutes, ProcessingTimeMinutes);

-- Index for fleet analysis by vehicle
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle_Time_INCLUDE
ON FactFleetTrips(VehicleID, TimeKey)
INCLUDE (DistanceKm, DurationMinutes, IdleTimeMinutes, FuelConsumedLiters);

-- Update statistics
UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS FactFleetTrips WITH FULLSCAN;
```

### Issue: Data Duplication in Incremental Loads

**Symptom**: Same records appear multiple times

**Solution**: Implement merge pattern with staging table

```sql
CREATE PROCEDURE usp_MergeWarehouseOperations
AS
BEGIN
    -- Use staging table to prevent duplicates
    MERGE FactWarehouseOperations AS target
    USING StagingWarehouseOperations AS source
    ON target.OrderID = source.OrderID 
        AND target.OperationType = source.OperationType
        AND target.TimeKey = source.TimeKey
    WHEN NOT MATCHED THEN
        INSERT (TimeKey, GeographyKey, ProductKey, OperationType, OrderID, Quantity, DwellTimeMinutes)
        VALUES (source.TimeKey, source.GeographyKey, source.ProductKey, source.OperationType, source.OrderID, source.Quantity, source.DwellTimeMinutes)
    WHEN MATCHED AND source.IsCompleted = 1 THEN
        UPDATE SET 
            target.IsCompleted = source.IsCompleted,
            target.DwellTimeMinutes = source.DwellTimeMinutes;
    
    TRUNCATE TABLE StagingWarehouseOperations;
END;
```

## Best Practices

1. **Partition large fact tables** by date for better query performance:
```sql
CREATE PARTITION FUNCTION PF_Date (DATE)
AS RANGE RIGHT FOR VALUES 
('2026-01-01', '2026-02-01', '2026-03-01', /* ... */);

CREATE PARTITION SCHEME PS_Date
AS PARTITION PF_Date ALL TO ([PRIMARY]);

ALTER TABLE FactWarehouseOperations
DROP CONSTRAINT PK_FactWarehouseOperations;

ALTER TABLE FactWarehouseOperations
ADD CONSTRAINT PK_FactWarehouseOperations 
PRIMARY KEY (OperationKey, TimeKey)
ON PS_Date(TimeKey);
```

2. **Use incremental refresh** in Power BI for large datasets

3. **Archive historical data** older than 2 years to separate tables

4. **Monitor SQL Server performance** using DMVs:
```sql
-- Find expensive queries
SELECT TOP 10
    SUBSTRING(qt.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset
            WHEN -1 THEN DATALENGTH(qt.text)
            ELSE qs.statement_end_offset
        END - qs.statement_start_offset)/2)+1) AS query_text,
    qs.execution_count,
    qs.total_elapsed_time / 1000000 AS total_elapsed_time_sec,
    qs.total_worker_time / 1000000 AS total_worker_time_sec
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
WHERE qt.text LIKE '%FactWarehouseOperations%'
ORDER BY qs.total_elapsed_time DESC;
```

5. **Validate data quality** with regular checks:
```sql
-- Check for orphaned records
SELECT 'Orphaned Product Keys' AS Issue, COUNT(*) AS Count
FROM FactWarehouseOperations wo
LEFT JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
WHERE p.ProductKey IS NULL

UNION ALL

SELECT 'Negative Dwell Times', COUNT(*)
FROM FactWarehouseOperations
WHERE DwellTimeMinutes < 0;
```
