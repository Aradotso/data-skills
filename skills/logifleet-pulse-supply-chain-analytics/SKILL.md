---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehouse for fleet, warehouse, and supply chain KPI modeling
triggers:
  - "set up logifleet pulse supply chain analytics"
  - "deploy logistics data warehouse with power bi"
  - "create multi-fact star schema for fleet management"
  - "build warehouse operations dashboard"
  - "implement cross-modal supply chain analytics"
  - "configure logicore fleet intelligence"
  - "connect power bi to logistics sql database"
  - "model warehouse gravity zones and fleet telemetry"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a multi-modal logistics intelligence engine built on MS SQL Server and Power BI. It provides:

- **Multi-fact star schema** for warehouse operations, fleet trips, and cross-dock activities
- **Unified semantic layer** connecting inventory, fleet telemetry, and external data sources
- **Real-time Power BI dashboards** with 15-minute refresh cycles
- **Warehouse Gravity Zones™** — spatial optimization based on pick frequency and item value
- **Predictive bottleneck detection** using cross-fact KPI harmonization
- **Role-based access control** with row-level security

The system ingests data from WMS, telematics, GPS feeds, supplier portals, and external APIs to provide holistic supply chain visibility.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Power BI Desktop (latest version)
- Access to WMS, TMS, or ERP data sources

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTimeValue DATETIME2 NOT NULL,
    HourOfDay INT,
    DayOfWeek INT,
    WeekOfYear INT,
    MonthNumber INT,
    QuarterNumber INT,
    FiscalYear INT,
    IsWeekend BIT,
    TimeSlot15Min INT -- 0-95 for 15-minute buckets
);
GO

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    NodeID VARCHAR(50) UNIQUE NOT NULL,
    NodeName NVARCHAR(100),
    NodeType VARCHAR(20), -- 'Warehouse', 'RouteNode', 'Customer'
    AddressLine1 NVARCHAR(200),
    City NVARCHAR(100),
    Region NVARCHAR(100),
    Country NVARCHAR(100),
    Continent NVARCHAR(50),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    PostalCode VARCHAR(20)
);
GO

CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName NVARCHAR(200),
    ProductCategory NVARCHAR(100),
    GravityScore DECIMAL(5,2), -- 0-100 scale
    PickFrequencyTier VARCHAR(20), -- 'Fast', 'Medium', 'Slow'
    UnitValue DECIMAL(10,2),
    FragilityIndex DECIMAL(3,2), -- 0-1 scale
    WeightKG DECIMAL(8,2),
    VolumeM3 DECIMAL(8,4),
    RequiresColdStorage BIT,
    ReplenishmentLeadTimeDays INT
);
GO

CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) UNIQUE NOT NULL,
    SupplierName NVARCHAR(200),
    LeadTimeAvgDays DECIMAL(5,1),
    LeadTimeVarianceDays DECIMAL(5,1),
    DefectPercentage DECIMAL(5,2),
    OnTimeDeliveryRate DECIMAL(5,2), -- 0-100
    ComplianceScore DECIMAL(3,1), -- 0-10 scale
    LastAuditDate DATE
);
GO

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    OperationType VARCHAR(20), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    Quantity INT,
    DwellTimeMinutes INT,
    StorageZone VARCHAR(50),
    OperatorID VARCHAR(50),
    CycleTimeSeconds INT,
    ErrorFlag BIT,
    BatchID VARCHAR(50)
);
GO

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID VARCHAR(50),
    DriverID VARCHAR(50),
    RouteID VARCHAR(50),
    TripDistanceKM DECIMAL(8,2),
    TripDurationMinutes INT,
    FuelConsumedLiters DECIMAL(6,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    LoadWeightKG DECIMAL(10,2),
    WeatherCondition VARCHAR(50),
    DelayReasonCode VARCHAR(20),
    OnTimeFlag BIT
);
GO

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    Quantity INT,
    TransferDurationMinutes INT,
    TemporaryStorageFlag BIT
);
GO

-- Create indexes for performance
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time 
ON FactWarehouseOperations(TimeKey) INCLUDE (Quantity, DwellTimeMinutes);

CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product 
ON FactWarehouseOperations(ProductKey, TimeKey);

CREATE NONCLUSTERED INDEX IX_FactFleet_Time 
ON FactFleetTrips(TimeKey) INCLUDE (FuelConsumedLiters, IdleTimeMinutes);

CREATE NONCLUSTERED INDEX IX_FactFleet_Route 
ON FactFleetTrips(RouteID, TimeKey);
```

### Step 2: Create ETL Stored Procedures

```sql
-- Incremental load procedure for warehouse operations
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadTime DATETIME2,
    @CurrentLoadTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Example: Load from staging table or external WMS
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, SupplierKey,
        OperationType, Quantity, DwellTimeMinutes, StorageZone,
        OperatorID, CycleTimeSeconds, ErrorFlag, BatchID
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.SupplierKey,
        stg.OperationType,
        stg.Quantity,
        stg.DwellTimeMinutes,
        stg.StorageZone,
        stg.OperatorID,
        stg.CycleTimeSeconds,
        stg.ErrorFlag,
        stg.BatchID
    FROM StagingWarehouseOps stg
    INNER JOIN DimTime t ON CAST(stg.OperationDateTime AS DATE) = CAST(t.DateTimeValue AS DATE)
        AND DATEPART(HOUR, stg.OperationDateTime) = t.HourOfDay
        AND (DATEPART(MINUTE, stg.OperationDateTime) / 15) = (t.TimeSlot15Min / 4)
    INNER JOIN DimGeography g ON stg.WarehouseID = g.NodeID
    INNER JOIN DimProductGravity p ON stg.SKU = p.SKU
    LEFT JOIN DimSupplierReliability s ON stg.SupplierID = s.SupplierID
    WHERE stg.OperationDateTime >= @LastLoadTime
        AND stg.OperationDateTime < @CurrentLoadTime;
        
    RETURN @@ROWCOUNT;
END;
GO

-- Update product gravity scores based on recent activity
CREATE PROCEDURE sp_UpdateProductGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    UPDATE p
    SET GravityScore = (
        -- Weighted calculation: 40% frequency, 30% value, 30% replenishment urgency
        (ISNULL(freq.PicksPerDay, 0) * 0.4) +
        ((p.UnitValue / NULLIF(maxval.MaxValue, 0)) * 100 * 0.3) +
        ((1.0 / NULLIF(p.ReplenishmentLeadTimeDays, 0)) * 100 * 0.3)
    )
    FROM DimProductGravity p
    CROSS JOIN (
        SELECT MAX(UnitValue) AS MaxValue FROM DimProductGravity
    ) maxval
    LEFT JOIN (
        SELECT 
            ProductKey,
            COUNT(*) * 1.0 / NULLIF(DATEDIFF(DAY, MIN(t.DateTimeValue), MAX(t.DateTimeValue)), 0) AS PicksPerDay
        FROM FactWarehouseOperations f
        INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
        WHERE f.OperationType = 'Picking'
            AND t.DateTimeValue >= DATEADD(DAY, -30, GETDATE())
        GROUP BY ProductKey
    ) freq ON p.ProductKey = freq.ProductKey;
    
    -- Update tier classifications
    UPDATE DimProductGravity
    SET PickFrequencyTier = CASE
        WHEN GravityScore >= 70 THEN 'Fast'
        WHEN GravityScore >= 40 THEN 'Medium'
        ELSE 'Slow'
    END;
END;
GO
```

### Step 3: Configure Data Sources

Create a configuration file for connection strings (not committed to version control):

```json
{
  "sqlServer": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "ActiveDirectoryIntegrated"
  },
  "wmsApi": {
    "endpoint": "${WMS_API_ENDPOINT}",
    "apiKey": "${WMS_API_KEY}"
  },
  "telematicsApi": {
    "endpoint": "${TELEMETRICS_API_ENDPOINT}",
    "apiKey": "${TELEMETRICS_API_KEY}"
  },
  "weatherApi": {
    "endpoint": "${WEATHER_API_ENDPOINT}",
    "apiKey": "${WEATHER_API_KEY}"
  },
  "refreshSchedule": {
    "intervalMinutes": 15,
    "fullRefreshHour": 2
  }
}
```

### Step 4: Connect Power BI

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Enter server name: `${SQL_SERVER_HOST}`
4. Database: `LogiFleetPulse`
5. Select tables: `FactWarehouseOperations`, `FactFleetTrips`, `FactCrossDock`, and all `Dim*` tables
6. Load data and verify relationships

## Key Power BI DAX Measures

### Cross-Fact KPI: Dwell Time Impact on Fleet Idle

```dax
DwellVsFleetIdle = 
VAR AvgDwellTime = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        FactWarehouseOperations[OperationType] = "Picking"
    )
VAR AvgIdleTime = 
    CALCULATE(
        AVERAGE(FactFleetTrips[IdleTimeMinutes])
    )
RETURN
    DIVIDE(AvgIdleTime, AvgDwellTime, 0) * 100
```

### Warehouse Gravity Zone Optimization

```dax
GravityZoneMismatch = 
CALCULATE(
    COUNTROWS(FactWarehouseOperations),
    FILTER(
        FactWarehouseOperations,
        (DimProductGravity[PickFrequencyTier] = "Fast" && 
         FactWarehouseOperations[StorageZone] NOT IN {"A", "B"}) ||
        (DimProductGravity[PickFrequencyTier] = "Slow" && 
         FactWarehouseOperations[StorageZone] IN {"A", "B"})
    )
)
```

### Fleet Utilization with Weighted Priority

```dax
FleetUtilizationScore = 
VAR TotalTime = SUM(FactFleetTrips[TripDurationMinutes])
VAR ProductiveTime = 
    SUM(FactFleetTrips[TripDurationMinutes]) - 
    SUM(FactFleetTrips[IdleTimeMinutes])
VAR HighPriorityLoads = 
    CALCULATE(
        COUNTROWS(FactFleetTrips),
        RELATEDTABLE(FactWarehouseOperations),
        DimProductGravity[GravityScore] > 70
    )
VAR TotalLoads = COUNTROWS(FactFleetTrips)
RETURN
    (DIVIDE(ProductiveTime, TotalTime) * 0.6) + 
    (DIVIDE(HighPriorityLoads, TotalLoads) * 0.4)
```

### Predictive Bottleneck Index

```dax
BottleneckRiskScore = 
VAR DwellTrend = 
    CALCULATE(
        [AvgDwellTime],
        DATESINPERIOD(DimTime[DateTimeValue], MAX(DimTime[DateTimeValue]), -7, DAY)
    ) / 
    CALCULATE(
        [AvgDwellTime],
        DATESINPERIOD(DimTime[DateTimeValue], MAX(DimTime[DateTimeValue]), -30, DAY)
    )
VAR CapacityUtilization = 
    DIVIDE(
        CALCULATE(COUNTROWS(FactWarehouseOperations), 
                  FactWarehouseOperations[OperationType] = "Putaway"),
        [WarehouseCapacity]
    )
VAR DelayRate = 
    DIVIDE(
        CALCULATE(COUNTROWS(FactFleetTrips), FactFleetTrips[OnTimeFlag] = 0),
        COUNTROWS(FactFleetTrips)
    )
RETURN
    (DwellTrend * 0.4) + (CapacityUtilization * 0.35) + (DelayRate * 0.25)
```

## Common Analysis Patterns

### Pattern 1: Cross-Fact Shipment Lifecycle

```sql
-- SQL query to track end-to-end shipment with warehouse and fleet operations
SELECT 
    wo.BatchID,
    p.SKU,
    p.ProductName,
    g_warehouse.NodeName AS WarehouseName,
    wo.DwellTimeMinutes AS WarehouseDwellTime,
    ft.VehicleID,
    ft.TripDistanceKM,
    ft.IdleTimeMinutes AS FleetIdleTime,
    g_dest.NodeName AS DestinationName,
    ft.OnTimeFlag,
    CASE 
        WHEN ft.DelayReasonCode = 'WEATHER' AND wo.DwellTimeMinutes > 72*60 THEN 'High Risk'
        WHEN ft.DelayReasonCode = 'TRAFFIC' AND p.RequiresColdStorage = 1 THEN 'Medium Risk'
        ELSE 'Normal'
    END AS RiskCategory
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
INNER JOIN DimGeography g_warehouse ON wo.GeographyKey = g_warehouse.GeographyKey
LEFT JOIN FactFleetTrips ft ON wo.BatchID = ft.RouteID -- Simplified join for example
LEFT JOIN DimGeography g_dest ON ft.DestinationGeographyKey = g_dest.GeographyKey
WHERE wo.OperationType = 'Shipping'
    AND wo.TimeKey >= (SELECT TimeKey FROM DimTime WHERE DateTimeValue >= DATEADD(QUARTER, -1, GETDATE()))
ORDER BY wo.BatchID;
```

### Pattern 2: Warehouse Zone Reassignment Recommendations

```sql
-- Identify products that should be moved to different gravity zones
WITH ProductActivity AS (
    SELECT 
        p.ProductKey,
        p.SKU,
        p.ProductName,
        p.GravityScore,
        p.PickFrequencyTier,
        wo.StorageZone,
        COUNT(*) AS PickCount,
        AVG(wo.CycleTimeSeconds) AS AvgCycleTime
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE wo.OperationType = 'Picking'
        AND t.DateTimeValue >= DATEADD(DAY, -30, GETDATE())
    GROUP BY p.ProductKey, p.SKU, p.ProductName, p.GravityScore, 
             p.PickFrequencyTier, wo.StorageZone
)
SELECT 
    SKU,
    ProductName,
    GravityScore,
    PickFrequencyTier,
    StorageZone AS CurrentZone,
    CASE 
        WHEN PickFrequencyTier = 'Fast' THEN 'A'
        WHEN PickFrequencyTier = 'Medium' THEN 'C'
        ELSE 'E'
    END AS RecommendedZone,
    PickCount,
    AvgCycleTime,
    (AvgCycleTime * PickCount) AS TotalLostSeconds
FROM ProductActivity
WHERE (PickFrequencyTier = 'Fast' AND StorageZone NOT IN ('A', 'B'))
   OR (PickFrequencyTier = 'Slow' AND StorageZone IN ('A', 'B'))
ORDER BY TotalLostSeconds DESC;
```

### Pattern 3: Fleet Route Optimization by Product Priority

```sql
-- Analyze fleet efficiency based on cargo gravity scores
SELECT 
    ft.RouteID,
    ft.VehicleID,
    AVG(p.GravityScore) AS AvgCargoGravity,
    SUM(ft.TripDistanceKM) AS TotalDistanceKM,
    SUM(ft.IdleTimeMinutes) AS TotalIdleTime,
    SUM(ft.FuelConsumedLiters) AS TotalFuel,
    CASE 
        WHEN AVG(p.GravityScore) > 70 AND SUM(ft.IdleTimeMinutes) > 120 THEN 'Priority_Inefficient'
        WHEN AVG(p.GravityScore) > 70 THEN 'Priority_Efficient'
        WHEN SUM(ft.IdleTimeMinutes) > 180 THEN 'Standard_Inefficient'
        ELSE 'Standard_Normal'
    END AS RouteCategory
FROM FactFleetTrips ft
LEFT JOIN FactWarehouseOperations wo ON ft.RouteID = wo.BatchID
LEFT JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
WHERE t.DateTimeValue >= DATEADD(MONTH, -1, GETDATE())
GROUP BY ft.RouteID, ft.VehicleID
HAVING AVG(p.GravityScore) > 50 OR SUM(ft.IdleTimeMinutes) > 120
ORDER BY AvgCargoGravity DESC, TotalIdleTime DESC;
```

## Configuration & Security

### Row-Level Security (RLS) Setup

```sql
-- Create roles for different user types
CREATE ROLE WarehouseOperator;
CREATE ROLE FleetManager;
CREATE ROLE Executive;

-- Create RLS filter function
CREATE FUNCTION fn_SecurityFilter(@UserRole AS VARCHAR(50))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN
    SELECT 1 AS AccessFlag
    WHERE 
        (@UserRole = 'Executive') OR
        (@UserRole = 'WarehouseOperator' AND OBJECT_NAME(@@PROCID) LIKE '%Warehouse%') OR
        (@UserRole = 'FleetManager' AND OBJECT_NAME(@@PROCID) LIKE '%Fleet%');
GO

-- Apply security policy
CREATE SECURITY POLICY WarehouseFleetSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityFilter(USER_NAME())
ON dbo.FactWarehouseOperations,
ADD FILTER PREDICATE dbo.fn_SecurityFilter(USER_NAME())
ON dbo.FactFleetTrips
WITH (STATE = ON);
```

### Automated Alerting Configuration

```sql
-- Create alert threshold table
CREATE TABLE AlertThresholds (
    AlertID INT PRIMARY KEY IDENTITY(1,1),
    AlertName NVARCHAR(100),
    MetricName VARCHAR(50),
    ThresholdValue DECIMAL(10,2),
    ComparisonOperator VARCHAR(2), -- '>', '<', '=', '!='
    AlertRecipients NVARCHAR(500), -- Email addresses
    IsActive BIT DEFAULT 1
);

-- Insert sample thresholds
INSERT INTO AlertThresholds (AlertName, MetricName, ThresholdValue, ComparisonOperator, AlertRecipients)
VALUES 
    ('High Fleet Idle Time', 'IdleTimePercentage', 15.0, '>', '${FLEET_MANAGER_EMAIL}'),
    ('Warehouse Dwell Spike', 'AvgDwellTimeMinutes', 180.0, '>', '${WAREHOUSE_MANAGER_EMAIL}'),
    ('Cold Storage Temp Drift', 'TemperatureCelsius', -18.0, '>', '${OPERATIONS_DIRECTOR_EMAIL}'),
    ('Low On-Time Delivery', 'OnTimePercentage', 85.0, '<', '${LOGISTICS_DIRECTOR_EMAIL}');

-- Alerting stored procedure
CREATE PROCEDURE sp_CheckAlerts
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    -- Fleet idle time check
    IF EXISTS (
        SELECT 1 
        FROM FactFleetTrips ft
        INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
        WHERE t.DateTimeValue >= DATEADD(HOUR, -1, GETDATE())
        GROUP BY ft.VehicleID
        HAVING (SUM(ft.IdleTimeMinutes) * 100.0 / SUM(ft.TripDurationMinutes)) > 
               (SELECT ThresholdValue FROM AlertThresholds WHERE MetricName = 'IdleTimePercentage')
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Fleet idle time exceeded 15% in the last hour.';
        -- Send notification via SQL Server Database Mail
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = (SELECT AlertRecipients FROM AlertThresholds WHERE MetricName = 'IdleTimePercentage'),
            @subject = 'LogiFleet Pulse - Fleet Idle Time Alert',
            @body = @AlertMessage;
    END
    
    -- Additional alert checks...
END;
GO
```

## Troubleshooting

### Issue: Power BI Refresh Takes Too Long

**Solution**: Implement incremental refresh and partitioning

```sql
-- Create partitioned views for large fact tables
CREATE VIEW vw_FactFleetTrips_Recent
WITH SCHEMABINDING
AS
SELECT *
FROM dbo.FactFleetTrips ft
INNER JOIN dbo.DimTime t ON ft.TimeKey = t.TimeKey
WHERE t.DateTimeValue >= DATEADD(MONTH, -3, GETDATE());
GO

-- In Power BI: Use this view instead of the full table for real-time dashboards
```

### Issue: Cross-Fact Queries Timeout

**Solution**: Create indexed materialized views

```sql
-- Materialized view for common cross-fact query
CREATE VIEW vw_ShipmentLifecycle
WITH SCHEMABINDING
AS
SELECT 
    wo.BatchID,
    wo.ProductKey,
    wo.GeographyKey,
    wo.DwellTimeMinutes,
    ft.TripKey,
    ft.IdleTimeMinutes,
    ft.OnTimeFlag,
    t.DateTimeValue
FROM dbo.FactWarehouseOperations wo
INNER JOIN dbo.FactFleetTrips ft ON wo.BatchID = ft.RouteID
INNER JOIN dbo.DimTime t ON wo.TimeKey = t.TimeKey;
GO

-- Create clustered index on materialized view
CREATE UNIQUE CLUSTERED INDEX IX_ShipmentLifecycle 
ON vw_ShipmentLifecycle (BatchID, DateTimeValue);
```

### Issue: Gravity Score Calculation Produces NULL Values

**Solution**: Handle division by zero and missing data

```sql
-- Updated gravity score calculation with safeguards
UPDATE p
SET GravityScore = COALESCE(
    (
        (ISNULL(freq.PicksPerDay, 0) * 0.4) +
        ((p.UnitValue / NULLIF(maxval.MaxValue, 1)) * 100 * 0.3) +
        ((1.0 / NULLIF(GREATEST(p.ReplenishmentLeadTimeDays, 1), 0)) * 100 * 0.3)
    ),
    50.0 -- Default score for new products
)
FROM DimProductGravity p
-- ... rest of query
```

### Issue: Duplicate Records After ETL

**Solution**: Implement MERGE instead of INSERT

```sql
MERGE INTO FactWarehouseOperations AS target
USING StagingWarehouseOps AS source
ON target.BatchID = source.BatchID 
   AND target.OperationType = source.OperationType
   AND target.TimeKey = (SELECT TimeKey FROM DimTime WHERE ...)
WHEN NOT MATCHED BY TARGET THEN
    INSERT (TimeKey, GeographyKey, ProductKey, ...)
    VALUES (source.TimeKey, source.GeographyKey, source.ProductKey, ...);
```

## Advanced Patterns

### Real-Time Streaming via PolyBase

```sql
-- Create external data source for real-time telemetry
CREATE EXTERNAL DATA SOURCE TelematicsStream
WITH (
    TYPE = BLOB_STORAGE,
    LOCATION = '${TELEMETRICS_BLOB_STORAGE}',
    CREDENTIAL = TelematicsCredential
);

-- Create external table for streaming data
CREATE EXTERNAL TABLE ext_FleetTelemetryStream (
    VehicleID VARCHAR(50),
    Timestamp DATETIME2,
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    SpeedKmh DECIMAL(5,2),
    FuelLevelPercent DECIMAL(5,2),
    EngineTemp DECIMAL(5,2)
)
WITH (
    LOCATION = '/telemetry/',
    DATA_SOURCE = TelematicsStream,
    FILE_FORMAT = JSONFormat
);
```

### Machine Learning Integration for Demand Forecasting

```sql
-- Store prediction results
CREATE TABLE PredictedDemand (
    PredictionID INT PRIMARY KEY IDENTITY(1,1),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    ForecastDate DATE,
    PredictedQuantity INT,
    ConfidenceInterval DECIMAL(5,2),
    ModelVersion VARCHAR(20),
    GeneratedAt DATETIME2 DEFAULT GETDATE()
);

-- Join predictions with actual operations
SELECT 
    p.SKU,
    pd.ForecastDate,
    pd.PredictedQuantity,
    ISNULL(SUM(wo.Quantity), 0) AS ActualQuantity,
    ABS(pd.PredictedQuantity - ISNULL(SUM(wo.Quantity), 0)) AS Variance,
    pd.ConfidenceInterval
FROM PredictedDemand pd
INNER JOIN DimProductGravity p ON pd.ProductKey = p.ProductKey
LEFT JOIN FactWarehouseOperations wo ON pd.ProductKey = wo.ProductKey
    AND CAST(pd.ForecastDate AS DATE) = CAST((SELECT DateTimeValue FROM DimTime WHERE TimeKey = wo.TimeKey) AS DATE)
    AND wo.OperationType = 'Picking'
GROUP BY p.SKU, pd.ForecastDate, pd.PredictedQuantity, pd.ConfidenceInterval;
```

## Environment Variables Reference

Required environment variables for deployment:

```bash
# SQL Server
SQL_SERVER_HOST=your-server.database.windows.net
SQL_DB_NAME=LogiFleetPulse
SQL_USERNAME=${SQL_USERNAME}
SQL_PASSWORD=${SQL_PASSWORD}

# External APIs
WMS_API_ENDPOINT=https://your-wms.com/api/v1
WMS_API_KEY=${WMS_API_KEY}
TELEMETRICS_API_ENDPOINT=https://telemetry-provider.com/api
TELEMETRICS_API_KEY=${TELEMETRICS_API_KEY}
WEATHER_API_ENDPOINT=https://api.weatherapi.com/v1
WEATHER_API_KEY=${WEATHER_API_KEY}

# Email Alerts
FLEET_MANAGER_EMAIL=fleet@yourcompany.com
WAREHOUSE_MANAGER_EMAIL=warehouse@yourcompany.com
OPERATIONS_DIRECTOR_EMAIL=operations@yourcompany.com
LOGISTICS_DIRECTOR_EMAIL=logistics@yourcompany.com

# Azure Storage (optional)
TELEMETRICS_BLOB_STORAGE=https://youraccount.blob.core.windows.net/telemetry
AZURE_STORAGE_CONNECTION_STRING=${AZURE_STORAGE_CONNECTION_STRING}
```
