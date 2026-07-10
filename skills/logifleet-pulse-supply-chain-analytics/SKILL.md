---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing for logistics, fleet management, and supply chain intelligence with multi-fact star schema
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "deploy SQL schema for warehouse and fleet tracking"
  - "configure Power BI logistics dashboard"
  - "implement multi-fact star schema for supply chain"
  - "create warehouse gravity zones in LogiFleet"
  - "build fleet optimization data model"
  - "integrate telemetry data with logistics warehouse"
  - "query cross-modal supply chain KPIs"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive data warehousing and analytics platform for logistics and supply chain management. It combines warehouse operations, fleet telemetry, inventory management, and external data sources into a unified semantic layer using MS SQL Server and Power BI.

The platform uses a multi-fact star schema architecture with time-phased dimensions to enable cross-functional KPI analysis, predictive bottleneck detection, and real-time operational intelligence.

**Core capabilities:**
- Multi-fact data warehouse (warehouse operations, fleet trips, cross-dock)
- Time-phased dimension modeling (15-minute granularity)
- Warehouse gravity zone optimization
- Fleet maintenance triage engine
- Power BI visualization templates
- Role-based access control
- Predictive analytics for bottleneck detection

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to data sources: WMS, TMS, telemetry APIs, ERP systems

### Step 1: Deploy SQL Schema

1. Clone the repository:
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. Connect to your SQL Server instance and create the database:
```sql
-- Create the LogiFleet database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO
```

3. Execute the schema deployment scripts in order:
```sql
-- 1. Create dimension tables
:r schema/01_dimensions.sql

-- 2. Create fact tables
:r schema/02_facts.sql

-- 3. Create views and stored procedures
:r schema/03_views_procedures.sql

-- 4. Create indexes and partitions
:r schema/04_indexes.sql

-- 5. Set up security
:r schema/05_security.sql
```

### Step 2: Configure Data Sources

Update connection strings in `config.json`:
```json
{
  "connections": {
    "sqlServer": "Server=${SQL_SERVER};Database=LogiFleetPulse;Trusted_Connection=True;",
    "wmsApi": "${WMS_API_ENDPOINT}",
    "telemetryApi": "${TELEMETRY_API_ENDPOINT}",
    "erpConnection": "${ERP_CONNECTION_STRING}"
  },
  "refreshInterval": 900,
  "timeZone": "UTC"
}
```

### Step 3: Load Power BI Template

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. Enter SQL Server connection parameters when prompted
3. Configure data refresh schedule in Power BI Service (if publishing)

## Database Schema Structure

### Dimension Tables

**DimTime** - Time dimension with 15-minute granularity:
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2(0) NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME(0) NOT NULL,
    HourOfDay TINYINT NOT NULL,
    MinuteOfHour TINYINT NOT NULL,
    DayOfWeek TINYINT NOT NULL,
    DayName NVARCHAR(10) NOT NULL,
    IsWeekend BIT NOT NULL,
    WeekOfYear TINYINT NOT NULL,
    MonthOfYear TINYINT NOT NULL,
    QuarterOfYear TINYINT NOT NULL,
    FiscalYear INT NOT NULL,
    FiscalQuarter INT NOT NULL
);

-- Query example: Get hourly distribution
SELECT 
    HourOfDay,
    COUNT(*) as EventCount
FROM FactWarehouseOperations f
INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
WHERE t.DateKey = CONVERT(INT, CONVERT(VARCHAR(8), GETDATE(), 112))
GROUP BY HourOfDay
ORDER BY HourOfDay;
```

**DimProductGravity** - Product classification with gravity scoring:
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU NVARCHAR(50) NOT NULL,
    ProductName NVARCHAR(255) NOT NULL,
    Category NVARCHAR(100),
    GravityScore DECIMAL(5,2), -- Composite score
    VelocityClass NVARCHAR(20), -- Fast/Medium/Slow
    ValueClass NVARCHAR(20), -- High/Medium/Low
    FragilityScore TINYINT, -- 1-10
    TemperatureZone NVARCHAR(20), -- Ambient/Cold/Frozen
    OptimalStorageZone NVARCHAR(50),
    LastUpdated DATETIME2(0)
);

-- Calculate gravity score (run periodically)
UPDATE DimProductGravity
SET GravityScore = (
    (VelocityScore * 0.5) + 
    (ValueScore * 0.3) + 
    (10 - FragilityScore) * 0.2
);
```

**DimGeography** - Hierarchical location dimension:
```sql
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY,
    LocationID NVARCHAR(50) NOT NULL,
    LocationName NVARCHAR(255) NOT NULL,
    LocationType NVARCHAR(50), -- Warehouse/DistributionCenter/Route/Customer
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    AddressLine1 NVARCHAR(255),
    City NVARCHAR(100),
    StateProvince NVARCHAR(100),
    Country NVARCHAR(100),
    PostalCode NVARCHAR(20),
    Region NVARCHAR(100),
    Continent NVARCHAR(50),
    TimeZone NVARCHAR(50)
);
```

### Fact Tables

**FactWarehouseOperations** - Warehouse activity tracking:
```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    OperationType NVARCHAR(50) NOT NULL, -- Receiving/Putaway/Picking/Packing/Shipping
    OperationStartTime DATETIME2(0),
    OperationEndTime DATETIME2(0),
    DurationMinutes AS DATEDIFF(MINUTE, OperationStartTime, OperationEndTime),
    DwellTimeHours DECIMAL(10,2), -- Time item sat before next operation
    QuantityHandled INT,
    StorageZone NVARCHAR(50),
    OperatorID NVARCHAR(50),
    EquipmentID NVARCHAR(50),
    BatchNumber NVARCHAR(50),
    OrderID NVARCHAR(50),
    CONSTRAINT FK_WarehouseOps_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WarehouseOps_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    CONSTRAINT FK_WarehouseOps_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey)
);

-- Create indexes for common queries
CREATE INDEX IX_WarehouseOps_Time ON FactWarehouseOperations(TimeKey);
CREATE INDEX IX_WarehouseOps_Product ON FactWarehouseOperations(ProductKey);
CREATE INDEX IX_WarehouseOps_Type ON FactWarehouseOperations(OperationType);
```

**FactFleetTrips** - Fleet and vehicle telemetry:
```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    VehicleKey INT NOT NULL,
    DriverKey INT NOT NULL,
    StartTimeKey INT NOT NULL,
    EndTimeKey INT,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    TripStartTime DATETIME2(0),
    TripEndTime DATETIME2(0),
    TripDurationMinutes AS DATEDIFF(MINUTE, TripStartTime, TripEndTime),
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    AverageSpeedKPH DECIMAL(5,2),
    MaxSpeedKPH DECIMAL(5,2),
    HarshBrakingCount INT,
    HarshAccelerationCount INT,
    LoadWeightKG DECIMAL(10,2),
    DeliveryStatus NVARCHAR(50), -- OnTime/Delayed/InProgress
    DelayReasonCode NVARCHAR(50), -- Traffic/Weather/Mechanical/Loading
    CONSTRAINT FK_FleetTrips_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_FleetTrips_StartTime FOREIGN KEY (StartTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FleetTrips_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FleetTrips_Destination FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
);
```

## Data Loading Patterns

### Incremental ETL Stored Procedure

```sql
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME2(0)
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Stage data from source
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, GeographyKey, OperationType,
        OperationStartTime, OperationEndTime, DwellTimeHours,
        QuantityHandled, StorageZone, OperatorID, EquipmentID, 
        BatchNumber, OrderID
    )
    SELECT 
        t.TimeKey,
        p.ProductKey,
        g.GeographyKey,
        src.OperationType,
        src.OperationStartTime,
        src.OperationEndTime,
        DATEDIFF(HOUR, src.PreviousOperationTime, src.OperationStartTime) as DwellTimeHours,
        src.Quantity,
        src.StorageZone,
        src.OperatorID,
        src.EquipmentID,
        src.BatchNumber,
        src.OrderID
    FROM StagingWarehouseOps src
    INNER JOIN DimTime t ON DATEADD(MINUTE, (DATEPART(MINUTE, src.OperationStartTime) / 15) * 15, 
                                     CAST(CAST(src.OperationStartTime AS DATE) AS DATETIME) + 
                                     CAST(CAST(DATEPART(HOUR, src.OperationStartTime) AS VARCHAR) + ':00' AS TIME)) = t.FullDateTime
    INNER JOIN DimProductGravity p ON src.SKU = p.SKU
    INNER JOIN DimGeography g ON src.WarehouseID = g.LocationID
    WHERE src.OperationStartTime > @LastLoadDateTime
        AND src.IsProcessed = 0;
    
    -- Mark as processed
    UPDATE StagingWarehouseOps
    SET IsProcessed = 1, ProcessedDate = GETDATE()
    WHERE OperationStartTime > @LastLoadDateTime
        AND IsProcessed = 0;
END;
GO

-- Execute incremental load
DECLARE @LastLoad DATETIME2(0);
SELECT @LastLoad = MAX(OperationStartTime) FROM FactWarehouseOperations;
EXEC usp_LoadWarehouseOperations @LastLoadDateTime = @LastLoad;
```

### Real-Time Data Stream Integration

```sql
-- Create external table for streaming telemetry data
CREATE EXTERNAL DATA SOURCE TelemetryStream
WITH (
    TYPE = BLOB_STORAGE,
    LOCATION = '${AZURE_BLOB_ENDPOINT}',
    CREDENTIAL = TelemetryCredential
);

CREATE EXTERNAL TABLE ExternalFleetTelemetry (
    VehicleID NVARCHAR(50),
    Timestamp DATETIME2(0),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    Speed DECIMAL(5,2),
    FuelLevel DECIMAL(5,2),
    EngineTemp DECIMAL(5,2),
    TirePressureFrontLeft DECIMAL(5,2),
    TirePressureFrontRight DECIMAL(5,2)
)
WITH (
    LOCATION = '/telemetry/',
    DATA_SOURCE = TelemetryStream,
    FILE_FORMAT = JSONFormat
);

-- Process streaming data
CREATE PROCEDURE usp_ProcessTelemetryStream
AS
BEGIN
    -- Aggregate to trip segments
    INSERT INTO FactFleetTrips (
        VehicleKey, DriverKey, StartTimeKey, TripStartTime,
        AverageSpeedKPH, MaxSpeedKPH
    )
    SELECT 
        v.VehicleKey,
        d.DriverKey,
        t.TimeKey,
        MIN(tel.Timestamp) as TripStart,
        AVG(tel.Speed) as AvgSpeed,
        MAX(tel.Speed) as MaxSpeed
    FROM ExternalFleetTelemetry tel
    INNER JOIN DimVehicle v ON tel.VehicleID = v.VehicleID
    INNER JOIN DimDriver d ON v.CurrentDriverID = d.DriverID
    INNER JOIN DimTime t ON DATEADD(MINUTE, (DATEPART(MINUTE, tel.Timestamp) / 15) * 15, 
                                     CAST(CAST(tel.Timestamp AS DATE) AS DATETIME)) = t.FullDateTime
    GROUP BY v.VehicleKey, d.DriverKey, t.TimeKey;
END;
```

## Key Analytics Queries

### Cross-Fact KPI: Dwell Time vs Fleet Idle Time

```sql
-- Analyze correlation between warehouse dwell and fleet idle
WITH WarehouseDwell AS (
    SELECT 
        wo.OrderID,
        p.SKU,
        AVG(wo.DwellTimeHours) as AvgDwellHours
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    WHERE wo.OperationType = 'Picking'
        AND wo.OperationStartTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY wo.OrderID, p.SKU
),
FleetIdle AS (
    SELECT 
        ft.TripKey,
        ft.OrderID,
        ft.IdleTimeMinutes,
        ft.TripDurationMinutes
    FROM FactFleetTrips ft
    WHERE ft.TripStartTime >= DATEADD(DAY, -30, GETDATE())
)
SELECT 
    wd.SKU,
    AVG(wd.AvgDwellHours) as AvgWarehouseDwell,
    AVG(fi.IdleTimeMinutes) as AvgFleetIdle,
    AVG(CAST(fi.IdleTimeMinutes AS FLOAT) / NULLIF(fi.TripDurationMinutes, 0)) * 100 as IdlePercentage,
    COUNT(DISTINCT fi.TripKey) as TripCount
FROM WarehouseDwell wd
INNER JOIN FleetIdle fi ON wd.OrderID = fi.OrderID
GROUP BY wd.SKU
HAVING AVG(wd.AvgDwellHours) > 24
ORDER BY AvgFleetIdle DESC;
```

### Warehouse Gravity Zone Analysis

```sql
-- Identify products that should be reassigned to different storage zones
SELECT 
    p.SKU,
    p.ProductName,
    p.CurrentStorageZone,
    p.GravityScore,
    COUNT(wo.OperationKey) as PickCount,
    AVG(wo.DurationMinutes) as AvgPickTime,
    AVG(wo.DwellTimeHours) as AvgDwellTime,
    CASE 
        WHEN p.GravityScore > 7.5 THEN 'Zone-A-FastPick'
        WHEN p.GravityScore > 5.0 THEN 'Zone-B-Standard'
        ELSE 'Zone-C-SlowMove'
    END as RecommendedZone,
    CASE 
        WHEN p.CurrentStorageZone != (
            CASE 
                WHEN p.GravityScore > 7.5 THEN 'Zone-A-FastPick'
                WHEN p.GravityScore > 5.0 THEN 'Zone-B-Standard'
                ELSE 'Zone-C-SlowMove'
            END
        ) THEN 'REASSIGN'
        ELSE 'OK'
    END as Action
FROM DimProductGravity p
INNER JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
WHERE wo.OperationType = 'Picking'
    AND wo.OperationStartTime >= DATEADD(DAY, -90, GETDATE())
GROUP BY 
    p.SKU, p.ProductName, p.CurrentStorageZone, p.GravityScore
HAVING COUNT(wo.OperationKey) >= 10
ORDER BY p.GravityScore DESC;
```

### Predictive Bottleneck Detection

```sql
-- Identify potential bottlenecks using moving averages
WITH HourlyMetrics AS (
    SELECT 
        t.DateKey,
        t.HourOfDay,
        wo.StorageZone,
        COUNT(*) as OperationCount,
        AVG(wo.DurationMinutes) as AvgDuration,
        STDEV(wo.DurationMinutes) as StdDevDuration
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE wo.OperationStartTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY t.DateKey, t.HourOfDay, wo.StorageZone
),
MovingAvg AS (
    SELECT 
        DateKey,
        HourOfDay,
        StorageZone,
        OperationCount,
        AvgDuration,
        AVG(AvgDuration) OVER (
            PARTITION BY StorageZone, HourOfDay 
            ORDER BY DateKey 
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) as MovingAvgDuration
    FROM HourlyMetrics
)
SELECT 
    StorageZone,
    HourOfDay,
    AvgDuration,
    MovingAvgDuration,
    (AvgDuration - MovingAvgDuration) / NULLIF(MovingAvgDuration, 0) * 100 as PercentDeviation,
    CASE 
        WHEN (AvgDuration - MovingAvgDuration) / NULLIF(MovingAvgDuration, 0) > 0.25 THEN 'HIGH RISK'
        WHEN (AvgDuration - MovingAvgDuration) / NULLIF(MovingAvgDuration, 0) > 0.15 THEN 'MEDIUM RISK'
        ELSE 'LOW RISK'
    END as BottleneckRisk
FROM MovingAvg
WHERE DateKey = CONVERT(INT, CONVERT(VARCHAR(8), GETDATE(), 112))
ORDER BY PercentDeviation DESC;
```

### Fleet Maintenance Triage

```sql
-- Prioritize maintenance based on revenue impact
WITH VehicleHealth AS (
    SELECT 
        v.VehicleKey,
        v.VehicleID,
        v.Make,
        v.Model,
        DATEDIFF(DAY, v.LastMaintenanceDate, GETDATE()) as DaysSinceLastMaintenance,
        AVG(ft.HarshBrakingCount + ft.HarshAccelerationCount) as AvgHarshEvents,
        AVG(ft.FuelConsumedLiters / NULLIF(ft.DistanceKM, 0)) as AvgFuelEfficiency,
        SUM(CASE WHEN ft.DeliveryStatus = 'Delayed' THEN 1 ELSE 0 END) as DelayedTrips,
        COUNT(ft.TripKey) as TotalTrips
    FROM DimVehicle v
    LEFT JOIN FactFleetTrips ft ON v.VehicleKey = ft.VehicleKey
    WHERE ft.TripStartTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY v.VehicleKey, v.VehicleID, v.Make, v.Model, v.LastMaintenanceDate
),
RevenueImpact AS (
    SELECT 
        ft.VehicleKey,
        SUM(o.OrderValue) as TotalRevenue,
        AVG(o.OrderValue) as AvgOrderValue
    FROM FactFleetTrips ft
    INNER JOIN Orders o ON ft.OrderID = o.OrderID
    WHERE ft.TripStartTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY ft.VehicleKey
)
SELECT 
    vh.VehicleID,
    vh.Make + ' ' + vh.Model as Vehicle,
    vh.DaysSinceLastMaintenance,
    vh.AvgHarshEvents,
    vh.AvgFuelEfficiency,
    vh.DelayedTrips,
    ri.TotalRevenue,
    ri.AvgOrderValue,
    -- Priority score: older maintenance + harsh driving + delays + revenue impact
    (
        (vh.DaysSinceLastMaintenance / 30.0 * 0.3) +
        (vh.AvgHarshEvents / 10.0 * 0.2) +
        (CAST(vh.DelayedTrips AS FLOAT) / NULLIF(vh.TotalTrips, 0) * 0.3) +
        (ri.TotalRevenue / 100000.0 * 0.2)
    ) as MaintenancePriorityScore,
    CASE 
        WHEN vh.DaysSinceLastMaintenance > 60 THEN 'URGENT'
        WHEN vh.DaysSinceLastMaintenance > 45 THEN 'HIGH'
        ELSE 'NORMAL'
    END as MaintenanceUrgency
FROM VehicleHealth vh
LEFT JOIN RevenueImpact ri ON vh.VehicleKey = ri.VehicleKey
ORDER BY MaintenancePriorityScore DESC;
```

## Alerting Configuration

### Create Alert Stored Procedure

```sql
CREATE PROCEDURE usp_CheckAlertsAndNotify
AS
BEGIN
    DECLARE @AlertTable TABLE (
        AlertType NVARCHAR(50),
        Severity NVARCHAR(20),
        Message NVARCHAR(500),
        Metric DECIMAL(10,2),
        Threshold DECIMAL(10,2)
    );
    
    -- Alert 1: Fleet idle time > 15%
    INSERT INTO @AlertTable
    SELECT 
        'Fleet Idle Time' as AlertType,
        'HIGH' as Severity,
        'Vehicle ' + v.VehicleID + ' has idle time of ' + 
            CAST(AVG(CAST(ft.IdleTimeMinutes AS FLOAT) / NULLIF(ft.TripDurationMinutes, 0)) * 100 AS VARCHAR(10)) + 
            '% over last 24 hours' as Message,
        AVG(CAST(ft.IdleTimeMinutes AS FLOAT) / NULLIF(ft.TripDurationMinutes, 0)) * 100 as Metric,
        15.0 as Threshold
    FROM FactFleetTrips ft
    INNER JOIN DimVehicle v ON ft.VehicleKey = v.VehicleKey
    WHERE ft.TripStartTime >= DATEADD(HOUR, -24, GETDATE())
    GROUP BY v.VehicleKey, v.VehicleID
    HAVING AVG(CAST(ft.IdleTimeMinutes AS FLOAT) / NULLIF(ft.TripDurationMinutes, 0)) > 0.15;
    
    -- Alert 2: Warehouse dwell time > 72 hours
    INSERT INTO @AlertTable
    SELECT 
        'Warehouse Dwell Time' as AlertType,
        'MEDIUM' as Severity,
        'SKU ' + p.SKU + ' in zone ' + wo.StorageZone + 
            ' has dwell time of ' + CAST(wo.DwellTimeHours AS VARCHAR(10)) + ' hours' as Message,
        wo.DwellTimeHours as Metric,
        72.0 as Threshold
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    WHERE wo.DwellTimeHours > 72
        AND wo.OperationStartTime >= DATEADD(HOUR, -24, GETDATE());
    
    -- Log alerts
    INSERT INTO AlertLog (AlertType, Severity, Message, Metric, Threshold, AlertTime)
    SELECT AlertType, Severity, Message, Metric, Threshold, GETDATE()
    FROM @AlertTable;
    
    -- Return active alerts
    SELECT * FROM @AlertTable;
END;
GO

-- Schedule to run every 15 minutes
-- (Configure via SQL Server Agent or external scheduler)
```

## Power BI Configuration

### Key DAX Measures

Create these measures in Power BI:

```dax
// Total Operations
Total Operations = COUNTROWS(FactWarehouseOperations)

// Average Dwell Time
Avg Dwell Time = 
AVERAGE(FactWarehouseOperations[DwellTimeHours])

// Fleet Utilization %
Fleet Utilization = 
DIVIDE(
    SUM(FactFleetTrips[TripDurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[TripDurationMinutes]),
    0
) * 100

// On-Time Delivery Rate
On-Time Delivery % = 
DIVIDE(
    CALCULATE(
        COUNTROWS(FactFleetTrips),
        FactFleetTrips[DeliveryStatus] = "OnTime"
    ),
    COUNTROWS(FactFleetTrips),
    0
) * 100

// Fuel Efficiency
Fuel Efficiency KM/L = 
DIVIDE(
    SUM(FactFleetTrips[DistanceKM]),
    SUM(FactFleetTrips[FuelConsumedLiters]),
    0
)

// Warehouse Throughput (operations per hour)
Warehouse Throughput = 
DIVIDE(
    COUNTROWS(FactWarehouseOperations),
    DISTINCTCOUNT(DimTime[HourOfDay]),
    0
)

// Cross-Fact: Orders with High Dwell + Delayed Delivery
Problem Orders = 
CALCULATE(
    DISTINCTCOUNT(FactWarehouseOperations[OrderID]),
    FactWarehouseOperations[DwellTimeHours] > 48,
    FILTER(
        FactFleetTrips,
        FactFleetTrips[DeliveryStatus] = "Delayed"
    )
)
```

### Row-Level Security (RLS)

```dax
// Create role: RegionalManager
[Region] = USERNAME()

// Create role: WarehouseOperator  
[GeographyKey] IN (
    LOOKUPVALUE(
        UserWarehouseAccess[GeographyKey],
        UserWarehouseAccess[UserEmail], USERNAME()
    )
)
```

## Troubleshooting

### Performance Issues

**Slow queries on fact tables:**
```sql
-- Check index usage
SELECT 
    OBJECT_NAME(s.object_id) as TableName,
    i.name as IndexName,
    s.user_seeks,
    s.user_scans,
    s.user_lookups,
    s.user_updates
FROM sys.dm_db_index_usage_stats s
INNER JOIN sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id
WHERE OBJECT_NAME(s.object_id) LIKE 'Fact%'
ORDER BY s.user_seeks + s.user_scans DESC;

-- Add missing indexes if needed
CREATE NONCLUSTERED INDEX IX_WarehouseOps_TimeProduct 
ON FactWarehouseOperations(TimeKey, ProductKey)
INCLUDE (DwellTimeHours, QuantityHandled);
```

**Power BI refresh timeout:**
- Enable query folding in Power Query
- Use incremental refresh for large fact tables
- Partition fact tables by month/quarter

### Data Quality Issues

**Missing dimension keys:**
```sql
-- Find orphaned fact records
SELECT 
    'FactWarehouseOperations' as TableName,
    COUNT(*) as OrphanedRecords
FROM FactWarehouseOperations wo
WHERE NOT EXISTS (
    SELECT 1 FROM DimProductGravity p WHERE p.ProductKey = wo.ProductKey
)

UNION ALL

SELECT 
    'FactFleetTrips',
    COUNT(*)
FROM FactFleetTrips ft
WHERE NOT EXISTS (
    SELECT 1 FROM DimVehicle v WHERE v.VehicleKey = ft.VehicleKey
);

-- Clean up or assign to "Unknown" dimension member
UPDATE FactWarehouseOperations
SET ProductKey = -1  -- Unknown product key
WHERE NOT EXISTS (
    SELECT 1 FROM DimProductGravity p WHERE p.ProductKey = ProductKey
);
```

### Connection Issues

**External table not accessible:**
```sql
-- Test external data source
SELECT TOP 10 * FROM ExternalFleetTelemetry;

-- If fails, verify credential
SELECT * FROM sys.external_data_sources WHERE name = 'TelemetryStream';
SELECT * FROM sys.database_scoped_credentials WHERE name = 'TelemetryCredential';

-- Recreate credential if expired
DROP DATABASE SCOPED CREDENTIAL TelemetryCredential;
CREATE DATABASE SCOPED CREDENTIAL TelemetryCredential
WITH IDENTITY = 'SHARED ACCESS SIGNATURE',
SECRET = '${AZURE_SAS_TOKEN}';
```

## Best Practices

1. **Time dimension**: Always round timestamps to 15-minute intervals for consistent joins
2. **Partitioning**: Partition fact tables by month for queries that filter by date
3. **Indexes**: Create covering indexes for common filter + include columns
4. **Data refresh**: Use stored procedures for incremental loads, never full truncate/reload
5. **Security**: Implement RLS
