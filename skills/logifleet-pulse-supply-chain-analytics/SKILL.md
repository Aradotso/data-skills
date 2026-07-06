---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server and Power BI data warehouse for logistics, fleet management, and supply chain analytics with multi-fact star schema
triggers:
  - "set up logistics analytics dashboard"
  - "create supply chain data warehouse"
  - "build fleet management reporting system"
  - "implement warehouse operations KPI tracking"
  - "configure Power BI for logistics intelligence"
  - "design multi-fact star schema for supply chain"
  - "set up real-time fleet telemetry analytics"
  - "create cross-modal logistics reporting"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI agents to help developers implement LogiFleet Pulse, an advanced MS SQL Server and Power BI-based data warehouse for logistics, fleet management, and supply chain analytics. The system uses a multi-fact star schema to harmonize warehouse operations, fleet telemetry, inventory management, and external data sources into unified analytics.

## What LogiFleet Pulse Does

LogiFleet Pulse is a data warehousing and analytics platform that:

- **Unifies logistics data** from warehouse management systems, fleet telemetry, supplier portals, and external APIs
- **Implements multi-fact star schema** with time-phased dimensions for cross-domain KPI analysis
- **Provides Power BI dashboards** for real-time operational visibility (15-minute refresh cycles)
- **Enables predictive analytics** for bottleneck detection, fleet maintenance triage, and warehouse optimization
- **Supports cross-fact queries** linking warehouse operations with fleet performance metrics
- **Includes role-based security** for multi-department access control

## Installation and Setup

### Prerequisites

- MS SQL Server 2019+ (Enterprise or Standard edition)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to source systems: WMS, TMS, telematics APIs

### Step 1: Deploy SQL Schema

```sql
-- Create the LogiFleet database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Create time dimension (15-minute granularity)
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    DateKey INT NOT NULL,
    TimeSlot TIME NOT NULL,
    HourOfDay TINYINT NOT NULL,
    DayOfWeek TINYINT NOT NULL,
    WeekOfYear TINYINT NOT NULL,
    MonthNumber TINYINT NOT NULL,
    QuarterNumber TINYINT NOT NULL,
    FiscalYear INT NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT NOT NULL
)
GO

CREATE CLUSTERED INDEX IX_DimTime_DateTime ON DimTime(FullDateTime)
CREATE NONCLUSTERED INDEX IX_DimTime_DateKey ON DimTime(DateKey)
GO

-- Create geography dimension (hierarchical)
CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationID NVARCHAR(50) NOT NULL,
    LocationName NVARCHAR(200) NOT NULL,
    LocationType NVARCHAR(50) NOT NULL, -- 'Warehouse', 'RouteNode', 'CustomerSite'
    AddressLine1 NVARCHAR(200),
    City NVARCHAR(100),
    StateProvince NVARCHAR(100),
    Country NVARCHAR(100),
    PostalCode NVARCHAR(20),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    RegionCode NVARCHAR(50),
    WarehouseCapacityM3 DECIMAL(12,2),
    IsActive BIT NOT NULL DEFAULT 1,
    ValidFrom DATETIME NOT NULL,
    ValidTo DATETIME NULL
)
GO

CREATE UNIQUE INDEX UIX_DimGeography_LocationID ON DimGeography(LocationID, ValidFrom)
GO

-- Create product dimension with gravity scoring
CREATE TABLE DimProduct (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU NVARCHAR(50) NOT NULL,
    ProductName NVARCHAR(200) NOT NULL,
    ProductCategory NVARCHAR(100),
    ProductSubcategory NVARCHAR(100),
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    UnitValue DECIMAL(12,2),
    IsFragile BIT NOT NULL DEFAULT 0,
    IsPerishable BIT NOT NULL DEFAULT 0,
    ShelfLifeDays INT,
    GravityScore DECIMAL(5,2), -- Computed: velocity * value / fragility
    IsActive BIT NOT NULL DEFAULT 1,
    ValidFrom DATETIME NOT NULL,
    ValidTo DATETIME NULL
)
GO

CREATE UNIQUE INDEX UIX_DimProduct_SKU ON DimProduct(SKU, ValidFrom)
CREATE INDEX IX_DimProduct_GravityScore ON DimProduct(GravityScore DESC)
GO

-- Create warehouse operations fact table
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType NVARCHAR(50) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OperationID NVARCHAR(50) NOT NULL,
    QuantityHandled INT NOT NULL,
    DurationMinutes DECIMAL(10,2),
    DwellTimeHours DECIMAL(10,2),
    StorageZone NVARCHAR(50),
    OperatorID NVARCHAR(50),
    EquipmentID NVARCHAR(50),
    QualityFlag BIT NOT NULL DEFAULT 1,
    CostAmount DECIMAL(12,2),
    CONSTRAINT FK_FactWH_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactWH_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FactWH_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
)
GO

CREATE COLUMNSTORE INDEX CCIX_FactWarehouseOperations ON FactWarehouseOperations
    (TimeKey, GeographyKey, ProductKey, OperationType, QuantityHandled, DurationMinutes, DwellTimeHours)
GO

-- Create fleet trips fact table
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKeyStart INT NOT NULL,
    TimeKeyEnd INT NOT NULL,
    VehicleID NVARCHAR(50) NOT NULL,
    DriverID NVARCHAR(50) NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    RouteID NVARCHAR(50),
    PlannedDistanceKm DECIMAL(10,2),
    ActualDistanceKm DECIMAL(10,2),
    PlannedDurationMinutes INT,
    ActualDurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(10,2),
    LoadWeightKg DECIMAL(12,2),
    LoadVolumeM3 DECIMAL(10,2),
    OnTimeFlag BIT NOT NULL,
    WeatherDelayFlag BIT NOT NULL DEFAULT 0,
    TrafficDelayFlag BIT NOT NULL DEFAULT 0,
    CostAmount DECIMAL(12,2),
    CONSTRAINT FK_FactFleet_TimeStart FOREIGN KEY (TimeKeyStart) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactFleet_TimeEnd FOREIGN KEY (TimeKeyEnd) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactFleet_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FactFleet_Dest FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
)
GO

CREATE COLUMNSTORE INDEX CCIX_FactFleetTrips ON FactFleetTrips
    (TimeKeyStart, VehicleID, RouteID, ActualDistanceKm, ActualDurationMinutes, FuelConsumedLiters)
GO

-- Create bridge table for many-to-many relationships
CREATE TABLE BridgeWarehouseFleet (
    BridgeKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    OperationKey BIGINT NOT NULL,
    TripKey BIGINT NOT NULL,
    ShipmentID NVARCHAR(50) NOT NULL,
    CONSTRAINT FK_Bridge_Operation FOREIGN KEY (OperationKey) REFERENCES FactWarehouseOperations(OperationKey),
    CONSTRAINT FK_Bridge_Trip FOREIGN KEY (TripKey) REFERENCES FactFleetTrips(TripKey)
)
GO

CREATE INDEX IX_Bridge_Shipment ON BridgeWarehouseFleet(ShipmentID)
GO
```

### Step 2: Create ETL Stored Procedures

```sql
-- Incremental load procedure for warehouse operations
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Load from staging table (populated by external ETL tool)
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType, OperationID,
        QuantityHandled, DurationMinutes, DwellTimeHours, StorageZone,
        OperatorID, EquipmentID, QualityFlag, CostAmount
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.OperationType,
        s.OperationID,
        s.QuantityHandled,
        s.DurationMinutes,
        s.DwellTimeHours,
        s.StorageZone,
        s.OperatorID,
        s.EquipmentID,
        s.QualityFlag,
        s.CostAmount
    FROM StagingWarehouseOperations s
    INNER JOIN DimTime t ON CAST(s.OperationDateTime AS DATETIME) = t.FullDateTime
    INNER JOIN DimGeography g ON s.LocationID = g.LocationID AND s.OperationDateTime BETWEEN g.ValidFrom AND COALESCE(g.ValidTo, '9999-12-31')
    INNER JOIN DimProduct p ON s.SKU = p.SKU AND s.OperationDateTime BETWEEN p.ValidFrom AND COALESCE(p.ValidTo, '9999-12-31')
    WHERE s.OperationDateTime > @LastLoadDateTime
        AND s.IsProcessed = 0;
    
    -- Mark staging records as processed
    UPDATE StagingWarehouseOperations
    SET IsProcessed = 1, ProcessedDateTime = GETDATE()
    WHERE OperationDateTime > @LastLoadDateTime
        AND IsProcessed = 0;
END
GO

-- Procedure to calculate product gravity scores
CREATE PROCEDURE usp_CalculateProductGravity
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Calculate gravity based on last 90 days of operations
    WITH ProductVelocity AS (
        SELECT 
            p.ProductKey,
            p.SKU,
            p.UnitValue,
            p.IsFragile,
            COUNT(DISTINCT f.OperationID) AS OperationCount,
            SUM(f.QuantityHandled) AS TotalQuantity,
            AVG(f.DwellTimeHours) AS AvgDwellTime
        FROM DimProduct p
        INNER JOIN FactWarehouseOperations f ON p.ProductKey = f.ProductKey
        INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
        WHERE t.FullDateTime >= DATEADD(DAY, -90, GETDATE())
            AND f.OperationType IN ('Picking', 'Shipping')
        GROUP BY p.ProductKey, p.SKU, p.UnitValue, p.IsFragile
    )
    UPDATE p
    SET GravityScore = 
        CASE 
            WHEN pv.AvgDwellTime > 0 THEN
                (pv.TotalQuantity / NULLIF(pv.AvgDwellTime, 0)) * 
                (p.UnitValue / 100.0) * 
                (CASE WHEN p.IsFragile = 1 THEN 1.5 ELSE 1.0 END)
            ELSE 0
        END
    FROM DimProduct p
    INNER JOIN ProductVelocity pv ON p.ProductKey = pv.ProductKey
    WHERE p.IsActive = 1;
END
GO
```

### Step 3: Configure Data Sources

Create a configuration file to connect external systems:

```json
{
  "dataSources": {
    "wms": {
      "type": "ODBC",
      "connectionString": "DSN=${WMS_DSN};UID=${WMS_USER};PWD=${WMS_PASSWORD}",
      "tables": ["Receipts", "Putaways", "Picks", "Shipments"]
    },
    "fleet": {
      "type": "REST_API",
      "baseUrl": "https://api.fleettelemetry.example.com",
      "authToken": "${FLEET_API_KEY}",
      "endpoints": {
        "trips": "/v2/trips",
        "telemetry": "/v2/telemetry/realtime"
      }
    },
    "weather": {
      "type": "REST_API",
      "baseUrl": "https://api.weatherservice.example.com",
      "authToken": "${WEATHER_API_KEY}"
    }
  },
  "refreshSchedule": {
    "warehouse": "*/15 * * * *",
    "fleet": "*/15 * * * *",
    "dimensions": "0 2 * * *"
  }
}
```

## Key SQL Queries and Patterns

### Cross-Fact KPI Query: Warehouse Dwell Time vs Fleet Idle Time

```sql
-- Find correlation between warehouse dwell time and fleet idle time
-- for shipments in the last 30 days
SELECT 
    p.ProductCategory,
    AVG(wh.DwellTimeHours) AS AvgWarehouseDwellHours,
    AVG(fl.IdleTimeMinutes) AS AvgFleetIdleMinutes,
    AVG(fl.ActualDurationMinutes - fl.PlannedDurationMinutes) AS AvgDeliveryDelayMinutes,
    COUNT(DISTINCT b.ShipmentID) AS ShipmentCount,
    SUM(wh.CostAmount + fl.CostAmount) AS TotalCost
FROM BridgeWarehouseFleet b
INNER JOIN FactWarehouseOperations wh ON b.OperationKey = wh.OperationKey
INNER JOIN FactFleetTrips fl ON b.TripKey = fl.TripKey
INNER JOIN DimProduct p ON wh.ProductKey = p.ProductKey
INNER JOIN DimTime t ON wh.TimeKey = t.TimeKey
WHERE t.FullDateTime >= DATEADD(DAY, -30, GETDATE())
    AND wh.OperationType IN ('Picking', 'Packing')
GROUP BY p.ProductCategory
ORDER BY TotalCost DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify products in wrong gravity zones
-- (high-gravity products far from shipping docks)
WITH ProductGravityRank AS (
    SELECT 
        p.ProductKey,
        p.SKU,
        p.ProductName,
        p.GravityScore,
        wh.StorageZone,
        COUNT(*) AS PickCount,
        AVG(wh.DurationMinutes) AS AvgPickDuration,
        NTILE(4) OVER (ORDER BY p.GravityScore DESC) AS GravityQuartile
    FROM DimProduct p
    INNER JOIN FactWarehouseOperations wh ON p.ProductKey = wh.ProductKey
    INNER JOIN DimTime t ON wh.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(DAY, -90, GETDATE())
        AND wh.OperationType = 'Picking'
        AND p.IsActive = 1
    GROUP BY p.ProductKey, p.SKU, p.ProductName, p.GravityScore, wh.StorageZone
)
SELECT 
    SKU,
    ProductName,
    StorageZone,
    GravityScore,
    GravityQuartile,
    PickCount,
    AvgPickDuration,
    CASE 
        WHEN GravityQuartile = 1 AND StorageZone LIKE '%BACK%' THEN 'Move to Front'
        WHEN GravityQuartile = 4 AND StorageZone LIKE '%FRONT%' THEN 'Move to Back'
        ELSE 'OK'
    END AS RecommendedAction
FROM ProductGravityRank
WHERE GravityQuartile IN (1, 4)
ORDER BY GravityScore DESC;
```

### Fleet Predictive Maintenance Triage

```sql
-- Generate maintenance priority queue based on
-- recent trip patterns and telemetry
WITH VehicleMetrics AS (
    SELECT 
        VehicleID,
        COUNT(*) AS TripCount,
        SUM(ActualDistanceKm) AS TotalDistanceKm,
        AVG(FuelConsumedLiters / NULLIF(ActualDistanceKm, 0)) AS AvgFuelEfficiency,
        SUM(CASE WHEN OnTimeFlag = 0 THEN 1 ELSE 0 END) AS DelayCount,
        AVG(IdleTimeMinutes) AS AvgIdleMinutes,
        SUM(LoadWeightKg) AS TotalLoadKg
    FROM FactFleetTrips
    WHERE TimeKeyStart IN (
        SELECT TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(DAY, -30, GETDATE())
    )
    GROUP BY VehicleID
)
SELECT 
    vm.VehicleID,
    vm.TripCount,
    vm.TotalDistanceKm,
    vm.AvgFuelEfficiency,
    vm.DelayCount,
    vm.AvgIdleMinutes,
    -- Weighted risk score (higher = more urgent)
    (vm.DelayCount * 10.0) + 
    (vm.AvgIdleMinutes * 0.5) + 
    (CASE WHEN vm.AvgFuelEfficiency > 8.5 THEN 50 ELSE 0 END) +
    (vm.TotalDistanceKm / 1000.0) AS MaintenancePriorityScore
FROM VehicleMetrics vm
ORDER BY MaintenancePriorityScore DESC;
```

### Time-Phased Analysis: Hourly Operation Patterns

```sql
-- Identify peak warehouse operation hours
-- to optimize staffing
SELECT 
    t.HourOfDay,
    t.DayOfWeek,
    COUNT(*) AS OperationCount,
    AVG(f.DurationMinutes) AS AvgDurationMinutes,
    SUM(f.QuantityHandled) AS TotalQuantity,
    STRING_AGG(f.OperationType, ',') WITHIN GROUP (ORDER BY COUNT(*) DESC) AS TopOperationTypes
FROM FactWarehouseOperations f
INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
WHERE t.FullDateTime >= DATEADD(DAY, -90, GETDATE())
GROUP BY t.HourOfDay, t.DayOfWeek
ORDER BY OperationCount DESC;
```

## Power BI Configuration

### Connect Power BI to SQL Server

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Server: `${SQL_SERVER_HOST}`
4. Database: `LogiFleetPulse`
5. Data Connectivity mode: Import (for historical) or DirectQuery (for real-time)

### Create Relationships in Power BI Model

Power BI should auto-detect relationships based on foreign keys. Verify:

```
DimTime (TimeKey) ←→ FactWarehouseOperations (TimeKey)
DimTime (TimeKey) ←→ FactFleetTrips (TimeKeyStart)
DimGeography (GeographyKey) ←→ FactWarehouseOperations (GeographyKey)
DimGeography (GeographyKey) ←→ FactFleetTrips (OriginGeographyKey)
DimProduct (ProductKey) ←→ FactWarehouseOperations (ProductKey)
```

### Key DAX Measures

```dax
// Total Dwell Time (weighted average)
Total Dwell Time Hours = 
SUMX(
    FactWarehouseOperations,
    FactWarehouseOperations[DwellTimeHours] * FactWarehouseOperations[QuantityHandled]
) / SUM(FactWarehouseOperations[QuantityHandled])

// Fleet On-Time Percentage
Fleet On-Time % = 
DIVIDE(
    CALCULATE(COUNT(FactFleetTrips[TripKey]), FactFleetTrips[OnTimeFlag] = TRUE()),
    COUNT(FactFleetTrips[TripKey]),
    0
) * 100

// Cross-Fact: Cost Per Unit Delivered
Cost Per Unit Delivered = 
VAR WHCost = SUM(FactWarehouseOperations[CostAmount])
VAR FleetCost = SUM(FactFleetTrips[CostAmount])
VAR TotalUnits = SUM(FactWarehouseOperations[QuantityHandled])
RETURN
DIVIDE(WHCost + FleetCost, TotalUnits, 0)

// Predictive Bottleneck Index (requires historical data)
Bottleneck Risk Index = 
VAR CurrentCapacity = SUM(DimGeography[WarehouseCapacityM3])
VAR CurrentUtilization = SUM(FactWarehouseOperations[QuantityHandled]) * AVERAGE(DimProduct[UnitVolume])
VAR UtilizationRate = DIVIDE(CurrentUtilization, CurrentCapacity, 0)
VAR AvgDwellTime = [Total Dwell Time Hours]
RETURN
IF(
    UtilizationRate > 0.85 && AvgDwellTime > 48,
    "High Risk",
    IF(UtilizationRate > 0.70 || AvgDwellTime > 36, "Medium Risk", "Low Risk")
)
```

### Row-Level Security (RLS)

```dax
// Create role "Regional Manager" with filter:
[DimGeography[RegionCode]] = USERNAME()

// Create role "Warehouse Supervisor" with filter:
[DimGeography[LocationID]] IN (
    LOOKUPVALUE(
        UserWarehouseAccess[LocationID],
        UserWarehouseAccess[Username],
        USERNAME()
    )
)
```

## Common Patterns and Workflows

### Pattern 1: Daily ETL Refresh

```sql
-- Scheduled job (SQL Server Agent)
DECLARE @LastLoad DATETIME;

SELECT @LastLoad = MAX(ProcessedDateTime)
FROM ETLControlLog
WHERE TableName = 'FactWarehouseOperations';

EXEC usp_LoadWarehouseOperations @LastLoadDateTime = @LastLoad;
EXEC usp_LoadFleetTrips @LastLoadDateTime = @LastLoad;
EXEC usp_CalculateProductGravity;

INSERT INTO ETLControlLog (TableName, ProcessedDateTime, RowsProcessed)
VALUES ('FactWarehouseOperations', GETDATE(), @@ROWCOUNT);
```

### Pattern 2: Real-Time Alert Trigger

```sql
-- Create alert for high dwell time
CREATE PROCEDURE usp_AlertHighDwellTime
AS
BEGIN
    DECLARE @AlertThreshold INT = 72; -- hours
    
    SELECT 
        p.SKU,
        p.ProductName,
        wh.DwellTimeHours,
        g.LocationName,
        wh.OperationID
    INTO #AlertCandidates
    FROM FactWarehouseOperations wh
    INNER JOIN DimProduct p ON wh.ProductKey = p.ProductKey
    INNER JOIN DimGeography g ON wh.GeographyKey = g.GeographyKey
    INNER JOIN DimTime t ON wh.TimeKey = t.TimeKey
    WHERE wh.DwellTimeHours > @AlertThreshold
        AND t.FullDateTime >= DATEADD(HOUR, -1, GETDATE())
        AND p.IsPerishable = 1;
    
    IF EXISTS (SELECT 1 FROM #AlertCandidates)
    BEGIN
        -- Send email via Database Mail
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = 'operations@company.example.com',
            @subject = 'ALERT: High Dwell Time for Perishable Items',
            @body = 'See attached report',
            @query = 'SELECT * FROM #AlertCandidates',
            @attach_query_result_as_file = 1;
    END
END
GO
```

### Pattern 3: Incremental Dimension Updates (SCD Type 2)

```sql
-- Update DimProduct with new attributes while preserving history
MERGE INTO DimProduct AS Target
USING StagingProduct AS Source
ON Target.SKU = Source.SKU AND Target.ValidTo IS NULL
WHEN MATCHED AND (
    Target.ProductName <> Source.ProductName OR
    Target.UnitWeight <> Source.UnitWeight OR
    Target.UnitValue <> Source.UnitValue
)
THEN UPDATE SET 
    Target.ValidTo = GETDATE()
WHEN NOT MATCHED BY TARGET
THEN INSERT (
    SKU, ProductName, ProductCategory, UnitWeight, UnitVolume,
    UnitValue, IsFragile, IsPerishable, ShelfLifeDays, ValidFrom
)
VALUES (
    Source.SKU, Source.ProductName, Source.ProductCategory,
    Source.UnitWeight, Source.UnitVolume, Source.UnitValue,
    Source.IsFragile, Source.IsPerishable, Source.ShelfLifeDays,
    GETDATE()
);

-- Insert new versions for changed records
INSERT INTO DimProduct (
    SKU, ProductName, ProductCategory, UnitWeight, UnitVolume,
    UnitValue, IsFragile, IsPerishable, ShelfLifeDays, ValidFrom
)
SELECT 
    s.SKU, s.ProductName, s.ProductCategory, s.UnitWeight,
    s.UnitVolume, s.UnitValue, s.IsFragile, s.IsPerishable,
    s.ShelfLifeDays, GETDATE()
FROM StagingProduct s
INNER JOIN DimProduct p ON s.SKU = p.SKU
WHERE p.ValidTo IS NOT NULL
    AND p.ValidTo = CAST(GETDATE() AS DATE);
```

## Troubleshooting

### Issue: Power BI Refresh Timeout

**Symptom**: Scheduled refresh fails with "Gateway timeout" error

**Solution**: 
- Switch from Import to DirectQuery mode for large fact tables
- Add partitioning to fact tables by month:

```sql
-- Create partition function
CREATE PARTITION FUNCTION PF_MonthlyPartition (DATETIME)
AS RANGE RIGHT FOR VALUES (
    '2025-01-01', '2025-02-01', '2025-03-01', 
    '2025-04-01', '2025-05-01', '2025-06-01'
);

-- Create partition scheme
CREATE PARTITION SCHEME PS_MonthlyPartition
AS PARTITION PF_MonthlyPartition
ALL TO ([PRIMARY]);

-- Recreate fact table on partition scheme
CREATE TABLE FactWarehouseOperations_Partitioned (
    -- same columns as before
) ON PS_MonthlyPartition(OperationDateTime);
```

### Issue: Slow Cross-Fact Queries

**Symptom**: Queries joining warehouse and fleet facts take >30 seconds

**Solution**: Ensure bridge table has proper indexes and statistics updated:

```sql
-- Update statistics with full scan
UPDATE STATISTICS BridgeWarehouseFleet WITH FULLSCAN;
UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS FactFleetTrips WITH FULLSCAN;

-- Add covering index if missing
CREATE NONCLUSTERED INDEX IX_Bridge_Covering
ON BridgeWarehouseFleet (ShipmentID)
INCLUDE (OperationKey, TripKey);
```

### Issue: Gravity Score Not Updating

**Symptom**: `DimProduct.GravityScore` remains NULL or outdated

**Solution**: Ensure calculation procedure runs and has sufficient data:

```sql
-- Check if procedure completes
EXEC usp_CalculateProductGravity;

-- Verify data exists for calculation
SELECT 
    p.SKU,
    COUNT(f.OperationKey) AS OpCount,
    MIN(t.FullDateTime) AS EarliestOp
FROM DimProduct p
LEFT JOIN FactWarehouseOperations f ON p.ProductKey = f.ProductKey
LEFT JOIN DimTime t ON f.TimeKey = t.TimeKey
WHERE t.FullDateTime >= DATEADD(DAY, -90, GETDATE())
GROUP BY p.SKU
HAVING COUNT(f.OperationKey) = 0;
```

### Issue: Role-Level Security Not Applied

**Symptom**: Users see data from other regions despite RLS configured

**Solution**: Verify roles are assigned and usernames match:

```sql
-- Check effective username in Power BI
-- (run in DAX query window)
EVALUATE { USERNAME() }

-- Verify role assignment in SQL
SELECT 
    rp.name AS RoleName,
    dp.name AS UserName
FROM sys.database_role_members drm
INNER JOIN sys.database_principals rp ON drm.role_principal_id = rp.principal_id
INNER JOIN sys.database_principals dp ON drm.member_principal_id = dp.principal_id;
```

## Advanced: External Data Integration

### Integrate Weather API for Delay Attribution

```sql
-- Create external table for weather data (PolyBase)
CREATE EXTERNAL DATA SOURCE WeatherAPI
WITH (
    TYPE = GENERIC,
    LOCATION = 'https://api.weatherservice.example.com'
);

-- Create staging table for weather delays
CREATE TABLE StagingWeatherDelays (
    EventDateTime DATETIME,
    LocationID NVARCHAR(50),
    WeatherCode NVARCHAR(20),
    SeverityLevel INT,
    ImpactRadiusKm DECIMAL(10,2)
);

-- Correlate weather events with fleet delays
UPDATE ft
SET WeatherDelayFlag = 1
FROM FactFleetTrips ft
INNER JOIN DimTime ts ON ft.TimeKeyStart = ts.TimeKey
INNER JOIN DimGeography g ON ft.OriginGeographyKey = g.GeographyKey
INNER JOIN StagingWeatherDelays w ON 
    w.LocationID = g.LocationID
    AND w.EventDateTime BETWEEN ts.FullDateTime AND DATEADD(HOUR, 6, ts.FullDateTime)
WHERE w.SeverityLevel >= 3
    AND ft.OnTimeFlag = 0;
```

## Performance Optimization Checklist

- [ ] Columnstore indexes on all fact tables
