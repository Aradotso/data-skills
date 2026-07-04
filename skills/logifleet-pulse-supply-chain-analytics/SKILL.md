---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI multi-modal logistics intelligence engine with warehouse operations, fleet telemetry, and cross-fact KPI harmonization
triggers:
  - set up logifleet pulse analytics platform
  - deploy supply chain data warehouse schema
  - configure power bi logistics dashboard
  - implement warehouse gravity zones
  - create fleet optimization queries
  - integrate cross-modal supply chain analytics
  - build logistics star schema model
  - setup real-time fleet telemetry dashboard
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is an advanced logistics intelligence platform that unifies warehouse operations, fleet telemetry, inventory management, and external data signals into a single semantic layer. Built on MS SQL Server with Power BI visualization, it uses a multi-fact star schema to enable cross-functional KPI analysis like correlating warehouse dwell time with fleet fuel efficiency.

**Core capabilities:**
- Multi-fact star schema with time-phased dimensions
- Warehouse gravity zone optimization based on pick frequency and value
- Fleet telemetry integration (GPS, fuel, maintenance)
- Cross-fact KPI harmonization (e.g., dwell time vs. route efficiency)
- Predictive bottleneck detection
- Role-based security and multilingual dashboards

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Power BI Desktop (latest version)
- Access to data sources: WMS, TMS, telematics APIs

### Database Setup

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**
```sql
-- Run the main schema creation script
-- Execute in SSMS or via sqlcmd
sqlcmd -S YOUR_SERVER_NAME -d LogiFleetDB -i schema/01_create_database.sql
sqlcmd -S YOUR_SERVER_NAME -d LogiFleetDB -i schema/02_create_dimensions.sql
sqlcmd -S YOUR_SERVER_NAME -d LogiFleetDB -i schema/03_create_facts.sql
sqlcmd -S YOUR_SERVER_NAME -d LogiFleetDB -i schema/04_create_views.sql
sqlcmd -S YOUR_SERVER_NAME -d LogiFleetDB -i schema/05_create_stored_procedures.sql
```

3. **Configure connection strings:**
```json
{
  "sql_server": {
    "server": "YOUR_SQL_SERVER",
    "database": "LogiFleetDB",
    "authentication": "SQL",
    "username": "logifleet_user",
    "password": "${SQL_PASSWORD}"
  },
  "data_sources": {
    "wms_api": "${WMS_API_ENDPOINT}",
    "telematics_api": "${TELEMATICS_API_KEY}",
    "weather_api": "${WEATHER_API_KEY}"
  }
}
```

4. **Open Power BI template:**
```
Open LogiFleet_Pulse_Master.pbit in Power BI Desktop
Enter SQL Server connection details when prompted
```

## Core Schema Structure

### Key Fact Tables

**FactWarehouseOperations** - Records all warehouse activities
```sql
CREATE TABLE FactWarehouseOperations (
    OperationID INT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    WarehouseKey INT FOREIGN KEY REFERENCES DimWarehouse(WarehouseKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProduct(ProductKey),
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    DwellTimeMinutes INT,
    PickRateUnitsPerHour DECIMAL(10,2),
    GravityZoneID INT,
    QuantityHandled INT,
    OperationStartTime DATETIME,
    OperationEndTime DATETIME
);

-- Index for time-based queries
CREATE NONCLUSTERED INDEX IX_FactWO_Time_Product 
ON FactWarehouseOperations(TimeKey, ProductKey) 
INCLUDE (DwellTimeMinutes, QuantityHandled);
```

**FactFleetTrips** - Fleet telemetry and route data
```sql
CREATE TABLE FactFleetTrips (
    TripID INT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleKey INT FOREIGN KEY REFERENCES DimVehicle(VehicleKey),
    RouteKey INT FOREIGN KEY REFERENCES DimRoute(RouteKey),
    DriverKey INT FOREIGN KEY REFERENCES DimDriver(DriverKey),
    TripStartTime DATETIME,
    TripEndTime DATETIME,
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadWeightKG DECIMAL(10,2),
    OnTimeDelivery BIT,
    WeatherCondition VARCHAR(50)
);

-- Composite index for fleet efficiency queries
CREATE NONCLUSTERED INDEX IX_FactFT_Vehicle_Time 
ON FactFleetTrips(VehicleKey, TimeKey) 
INCLUDE (FuelConsumedLiters, IdleTimeMinutes, OnTimeDelivery);
```

### Key Dimension Tables

**DimTime** - Time intelligence with 15-minute granularity
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME,
    DateValue DATE,
    TimeValue TIME,
    QuarterHour INT, -- 1-4 for each hour
    HourOfDay INT,
    DayOfWeek VARCHAR(20),
    DayOfMonth INT,
    WeekOfYear INT,
    MonthName VARCHAR(20),
    Quarter INT,
    FiscalYear INT,
    IsWeekend BIT,
    IsHoliday BIT
);
```

**DimProductGravity** - Warehouse gravity zone classification
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY FOREIGN KEY REFERENCES DimProduct(ProductKey),
    GravityScore DECIMAL(5,2), -- 0-100
    PickFrequencyRank INT,
    ValuePerUnit DECIMAL(10,2),
    FragilityIndex INT, -- 1-5
    ReplenishmentLeadTimeDays INT,
    OptimalZoneID INT,
    LastRecalculatedDate DATETIME
);
```

## Common SQL Queries

### Cross-Fact KPI: Dwell Time vs Fleet Idle Time

```sql
-- Correlate warehouse dwell time with fleet idle time by product category
WITH WarehouseDwell AS (
    SELECT 
        p.CategoryName,
        AVG(wo.DwellTimeMinutes) AS AvgDwellTimeMinutes,
        SUM(wo.QuantityHandled) AS TotalUnitsHandled
    FROM FactWarehouseOperations wo
    INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.DateValue >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY p.CategoryName
),
FleetIdle AS (
    SELECT 
        p.CategoryName,
        AVG(ft.IdleTimeMinutes) AS AvgIdleTimeMinutes,
        AVG(ft.FuelConsumedLiters / NULLIF(ft.DistanceKM, 0)) AS AvgFuelEfficiency
    FROM FactFleetTrips ft
    INNER JOIN FactShipmentDetails sd ON ft.TripID = sd.TripID
    INNER JOIN DimProduct p ON sd.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
    WHERE t.DateValue >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY p.CategoryName
)
SELECT 
    wd.CategoryName,
    wd.AvgDwellTimeMinutes,
    wd.TotalUnitsHandled,
    fi.AvgIdleTimeMinutes,
    fi.AvgFuelEfficiency,
    -- Correlation indicator: high dwell + high idle = bottleneck
    CASE 
        WHEN wd.AvgDwellTimeMinutes > 72 AND fi.AvgIdleTimeMinutes > 30 
        THEN 'Critical Bottleneck'
        WHEN wd.AvgDwellTimeMinutes > 48 AND fi.AvgIdleTimeMinutes > 20 
        THEN 'Warning'
        ELSE 'Normal'
    END AS BottleneckStatus
FROM WarehouseDwell wd
INNER JOIN FleetIdle fi ON wd.CategoryName = fi.CategoryName
ORDER BY wd.AvgDwellTimeMinutes DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify products that should be relocated to higher-gravity zones
SELECT 
    p.ProductID,
    p.ProductName,
    pg.GravityScore,
    pg.PickFrequencyRank,
    pg.OptimalZoneID,
    wz.CurrentZoneID,
    wz.ZoneName AS CurrentZoneName,
    opt.ZoneName AS OptimalZoneName,
    CASE 
        WHEN pg.OptimalZoneID != wz.CurrentZoneID 
        THEN 'Relocation Recommended'
        ELSE 'Optimal'
    END AS RecommendationStatus
FROM DimProduct p
INNER JOIN DimProductGravity pg ON p.ProductKey = pg.ProductKey
INNER JOIN FactWarehouseInventory wi ON p.ProductKey = wi.ProductKey
INNER JOIN DimWarehouseZone wz ON wi.ZoneKey = wz.ZoneKey
LEFT JOIN DimWarehouseZone opt ON pg.OptimalZoneID = opt.ZoneKey
WHERE pg.PickFrequencyRank <= 20 -- Top 20 most-picked items
  AND pg.OptimalZoneID != wz.CurrentZoneID
ORDER BY pg.GravityScore DESC;
```

### Predictive Fleet Maintenance Queue

```sql
-- Generate prioritized maintenance queue based on telemetry and revenue impact
WITH VehicleHealth AS (
    SELECT 
        v.VehicleKey,
        v.VehicleID,
        v.Model,
        AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
        MAX(vd.EngineTemperatureCelsius) AS MaxEngineTemp,
        MIN(vd.TirePressurePSI) AS MinTirePressure,
        COUNT(CASE WHEN ft.OnTimeDelivery = 0 THEN 1 END) AS LateDeliveries
    FROM DimVehicle v
    INNER JOIN FactFleetTrips ft ON v.VehicleKey = ft.VehicleKey
    INNER JOIN FactVehicleDiagnostics vd ON v.VehicleKey = vd.VehicleKey
    WHERE ft.TripStartTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY v.VehicleKey, v.VehicleID, v.Model
),
RevenueImpact AS (
    SELECT 
        ft.VehicleKey,
        SUM(sd.OrderValue) AS TotalRevenueHandled
    FROM FactFleetTrips ft
    INNER JOIN FactShipmentDetails sd ON ft.TripID = sd.TripID
    WHERE ft.TripStartTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY ft.VehicleKey
)
SELECT 
    vh.VehicleID,
    vh.Model,
    vh.AvgIdleTime,
    vh.MaxEngineTemp,
    vh.MinTirePressure,
    vh.LateDeliveries,
    ri.TotalRevenueHandled,
    -- Weighted priority score
    (vh.AvgIdleTime * 0.2 + 
     (vh.MaxEngineTemp - 90) * 0.3 + 
     (35 - vh.MinTirePressure) * 0.2 + 
     vh.LateDeliveries * 10 + 
     (ri.TotalRevenueHandled / 10000) * 0.3) AS PriorityScore,
    CASE 
        WHEN vh.MaxEngineTemp > 105 OR vh.MinTirePressure < 28 
        THEN 'Immediate'
        WHEN vh.AvgIdleTime > 40 OR vh.LateDeliveries > 3 
        THEN 'Urgent'
        ELSE 'Scheduled'
    END AS MaintenancePriority
FROM VehicleHealth vh
LEFT JOIN RevenueImpact ri ON vh.VehicleKey = ri.VehicleKey
ORDER BY PriorityScore DESC;
```

## Stored Procedures

### Incremental Data Load

```sql
CREATE PROCEDURE sp_IncrementalLoadWarehouseOps
    @LastLoadTimestamp DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Merge new operations from staging table
    MERGE INTO FactWarehouseOperations AS target
    USING (
        SELECT 
            t.TimeKey,
            w.WarehouseKey,
            p.ProductKey,
            s.OperationType,
            s.DwellTimeMinutes,
            s.PickRateUnitsPerHour,
            s.GravityZoneID,
            s.QuantityHandled,
            s.OperationStartTime,
            s.OperationEndTime
        FROM StagingWarehouseOps s
        INNER JOIN DimTime t ON s.OperationStartTime >= t.FullDateTime 
            AND s.OperationStartTime < DATEADD(MINUTE, 15, t.FullDateTime)
        INNER JOIN DimWarehouse w ON s.WarehouseID = w.WarehouseID
        INNER JOIN DimProduct p ON s.ProductSKU = p.SKU
        WHERE s.LoadedAt > @LastLoadTimestamp
    ) AS source
    ON target.OperationID = source.OperationID
    WHEN NOT MATCHED THEN
        INSERT (TimeKey, WarehouseKey, ProductKey, OperationType, 
                DwellTimeMinutes, PickRateUnitsPerHour, GravityZoneID,
                QuantityHandled, OperationStartTime, OperationEndTime)
        VALUES (source.TimeKey, source.WarehouseKey, source.ProductKey,
                source.OperationType, source.DwellTimeMinutes,
                source.PickRateUnitsPerHour, source.GravityZoneID,
                source.QuantityHandled, source.OperationStartTime,
                source.OperationEndTime);
    
    -- Update last load timestamp
    UPDATE ETLControl 
    SET LastLoadTimestamp = GETDATE() 
    WHERE TableName = 'FactWarehouseOperations';
END;
```

### Gravity Zone Recalculation

```sql
CREATE PROCEDURE sp_RecalculateProductGravity
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Recalculate gravity scores based on 90-day rolling window
    UPDATE pg
    SET 
        pg.GravityScore = (
            (ISNULL(stats.PickFrequency, 0) * 0.4) +
            (ISNULL(stats.ValueScore, 0) * 0.3) +
            ((6 - ISNULL(p.FragilityIndex, 3)) * 10 * 0.2) +
            ((30 - ISNULL(s.LeadTimeDays, 15)) * 2 * 0.1)
        ),
        pg.PickFrequencyRank = stats.FreqRank,
        pg.OptimalZoneID = CASE 
            WHEN stats.PickFrequency > 50 THEN 1 -- High-gravity zone
            WHEN stats.PickFrequency > 20 THEN 2 -- Medium-gravity zone
            ELSE 3 -- Low-gravity zone
        END,
        pg.LastRecalculatedDate = GETDATE()
    FROM DimProductGravity pg
    INNER JOIN DimProduct p ON pg.ProductKey = p.ProductKey
    LEFT JOIN DimSupplier s ON p.SupplierKey = s.SupplierKey
    LEFT JOIN (
        SELECT 
            ProductKey,
            COUNT(*) AS PickFrequency,
            AVG(wo.QuantityHandled * p.UnitPrice) AS ValueScore,
            RANK() OVER (ORDER BY COUNT(*) DESC) AS FreqRank
        FROM FactWarehouseOperations wo
        INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
        WHERE wo.OperationType = 'Picking'
          AND wo.OperationStartTime >= DATEADD(DAY, -90, GETDATE())
        GROUP BY ProductKey
    ) stats ON pg.ProductKey = stats.ProductKey;
END;
```

## Power BI Integration

### DAX Measures

**Cross-Fact KPI: Logistics Efficiency Index**
```dax
Logistics Efficiency Index = 
VAR AvgDwellHours = 
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]) / 60
VAR AvgIdlePercent = 
    DIVIDE(
        AVERAGE(FactFleetTrips[IdleTimeMinutes]),
        AVERAGE(FactFleetTrips[TripDurationMinutes]),
        0
    ) * 100
VAR OnTimeRate = 
    DIVIDE(
        CALCULATE(COUNT(FactFleetTrips[TripID]), FactFleetTrips[OnTimeDelivery] = TRUE()),
        COUNT(FactFleetTrips[TripID]),
        0
    ) * 100
RETURN
    -- Weighted composite score (0-100, higher is better)
    (100 - AvgDwellHours * 2) * 0.4 +
    (100 - AvgIdlePercent) * 0.3 +
    OnTimeRate * 0.3
```

**Predictive Bottleneck Score**
```dax
Bottleneck Risk Score = 
VAR CurrentDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR HistoricalAvgDwell = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        DATESINPERIOD(DimTime[DateValue], MAX(DimTime[DateValue]), -90, DAY)
    )
VAR CurrentIdleTime = AVERAGE(FactFleetTrips[IdleTimeMinutes])
VAR HistoricalAvgIdle = 
    CALCULATE(
        AVERAGE(FactFleetTrips[IdleTimeMinutes]),
        DATESINPERIOD(DimTime[DateValue], MAX(DimTime[DateValue]), -90, DAY)
    )
VAR DwellDeviation = DIVIDE(CurrentDwell - HistoricalAvgDwell, HistoricalAvgDwell, 0)
VAR IdleDeviation = DIVIDE(CurrentIdleTime - HistoricalAvgIdle, HistoricalAvgIdle, 0)
RETURN
    IF(
        DwellDeviation > 0.2 && IdleDeviation > 0.15,
        100, -- Critical
        IF(
            DwellDeviation > 0.1 || IdleDeviation > 0.1,
            60, -- Warning
            20 -- Normal
        )
    )
```

### Power BI Template Configuration

1. **Data source setup in Power BI:**
```
Home > Transform Data > Data source settings
Server: YOUR_SQL_SERVER
Database: LogiFleetDB
Data Connectivity mode: DirectQuery (for real-time) or Import (for scheduled refresh)
```

2. **Row-level security setup:**
```dax
-- In Power BI Desktop: Modeling > Manage Roles
[WarehouseManager] = 
    DimWarehouse[RegionID] = LOOKUPVALUE(
        DimUser[RegionID],
        DimUser[UserEmail],
        USERPRINCIPALNAME()
    )

[FleetSupervisor] = 
    DimRoute[RegionID] = LOOKUPVALUE(
        DimUser[RegionID],
        DimUser[UserEmail],
        USERPRINCIPALNAME()
    )
```

3. **Schedule refresh:**
```
File > Options and settings > Options > Data load
Set refresh interval: Every 15 minutes (requires Power BI Pro/Premium)
```

## Configuration

### Environment Variables

```bash
# SQL Server connection
export SQL_SERVER="your-server.database.windows.net"
export SQL_DATABASE="LogiFleetDB"
export SQL_USERNAME="logifleet_app"
export SQL_PASSWORD="your-secure-password"

# Data source APIs
export WMS_API_ENDPOINT="https://wms.yourcompany.com/api/v1"
export WMS_API_KEY="your-wms-api-key"
export TELEMATICS_API_ENDPOINT="https://fleet.provider.com/api"
export TELEMATICS_API_KEY="your-telematics-key"
export WEATHER_API_KEY="your-weather-api-key"

# Alert configuration
export ALERT_EMAIL_SMTP="smtp.office365.com"
export ALERT_EMAIL_FROM="logifleet@yourcompany.com"
export ALERT_EMAIL_TO="operations@yourcompany.com"
```

### Alert Configuration SQL

```sql
-- Configure automated alerting thresholds
INSERT INTO AlertConfiguration (AlertName, Metric, ThresholdValue, ComparisonOperator, AlertSeverity)
VALUES 
    ('High Fleet Idle Time', 'AvgIdleTimePercent', 15, '>', 'Warning'),
    ('Critical Dwell Time', 'AvgDwellTimeHours', 72, '>', 'Critical'),
    ('Low On-Time Delivery', 'OnTimeDeliveryRate', 85, '<', 'Critical'),
    ('Fuel Efficiency Drop', 'AvgFuelEfficiencyChange', -10, '<', 'Warning'),
    ('Gravity Zone Drift', 'MisplacedHighGravityItems', 5, '>', 'Info');

-- Stored procedure to check and send alerts
CREATE PROCEDURE sp_CheckAndSendAlerts
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    -- Check each configured alert
    DECLARE alert_cursor CURSOR FOR
    SELECT AlertName, Metric, ThresholdValue, ComparisonOperator, AlertSeverity
    FROM AlertConfiguration WHERE IsActive = 1;
    
    -- Process alerts and log to AlertHistory table
    -- Integration with SQL Server Agent for scheduled execution
END;
```

## Common Patterns

### Pattern 1: Time-Phased Analysis

```sql
-- Analyze KPI trends across different time granularities
SELECT 
    t.FiscalYear,
    t.Quarter,
    t.MonthName,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
    AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
    AVG(CAST(ft.OnTimeDelivery AS DECIMAL)) * 100 AS OnTimePercent
FROM DimTime t
LEFT JOIN FactWarehouseOperations wo ON t.TimeKey = wo.TimeKey
LEFT JOIN FactFleetTrips ft ON t.TimeKey = ft.TimeKey
WHERE t.DateValue >= DATEADD(YEAR, -2, GETDATE())
GROUP BY t.FiscalYear, t.Quarter, t.MonthName
ORDER BY t.FiscalYear, t.Quarter;
```

### Pattern 2: Bridge Table for Many-to-Many

```sql
-- Routes can serve multiple warehouses, warehouses can use multiple routes
-- Bridge table pattern for complex relationships
SELECT 
    w.WarehouseName,
    r.RouteName,
    COUNT(ft.TripID) AS TripCount,
    AVG(ft.DistanceKM) AS AvgDistance,
    SUM(ft.FuelConsumedLiters) AS TotalFuelConsumed
FROM BridgeWarehouseRoute bwr
INNER JOIN DimWarehouse w ON bwr.WarehouseKey = w.WarehouseKey
INNER JOIN DimRoute r ON bwr.RouteKey = r.RouteKey
LEFT JOIN FactFleetTrips ft ON r.RouteKey = ft.RouteKey
WHERE bwr.IsActive = 1
GROUP BY w.WarehouseName, r.RouteName
ORDER BY TripCount DESC;
```

### Pattern 3: Slowly Changing Dimension (Type 2)

```sql
-- Track historical changes in product gravity scores
SELECT 
    p.ProductName,
    pg.GravityScore,
    pg.OptimalZoneID,
    pg.ValidFrom,
    pg.ValidTo,
    pg.IsCurrent
FROM DimProductGravity_History pg
INNER JOIN DimProduct p ON pg.ProductKey = p.ProductKey
WHERE p.ProductID = 'SKU-12345'
ORDER BY pg.ValidFrom DESC;
```

## Troubleshooting

### Performance Issues

**Problem:** Queries are slow on large fact tables
```sql
-- Solution: Partition fact tables by date
ALTER PARTITION FUNCTION PF_FactByMonth (datetime)
SPLIT RANGE ('2026-08-01');

-- Verify partition efficiency
SELECT 
    p.partition_number,
    p.rows,
    rv.value AS boundary_value
FROM sys.partitions p
INNER JOIN sys.partition_functions pf ON p.partition_id = pf.function_id
INNER JOIN sys.partition_range_values rv ON pf.function_id = rv.function_id
WHERE p.object_id = OBJECT_ID('FactWarehouseOperations')
ORDER BY p.partition_number;
```

**Problem:** Power BI dashboard refresh timeout
```
Solution: Switch from Import to DirectQuery mode for large datasets
Or implement incremental refresh:
1. Power BI Desktop > Transform Data
2. Right-click table > Incremental refresh
3. Set RangeStart and RangeEnd parameters
4. Configure policy: Archive last 2 years, refresh last 7 days
```

### Data Quality Issues

**Problem:** Missing dimension keys causing fact record failures
```sql
-- Implement default/unknown dimension records
INSERT INTO DimProduct (ProductKey, ProductID, ProductName, CategoryName)
VALUES (-1, 'UNKNOWN', 'Unknown Product', 'Unknown');

-- Use COALESCE in ETL to handle missing lookups
UPDATE StagingWarehouseOps
SET ProductKey = COALESCE(
    (SELECT ProductKey FROM DimProduct WHERE SKU = StagingWarehouseOps.ProductSKU),
    -1 -- Default to unknown
);
```

**Problem:** Duplicate records in fact tables
```sql
-- Create unique constraint on natural key
ALTER TABLE FactFleetTrips
ADD CONSTRAINT UQ_FleetTrips_NaturalKey 
UNIQUE (VehicleKey, TripStartTime, TripEndTime);

-- Clean existing duplicates
WITH DuplicateTrips AS (
    SELECT *, ROW_NUMBER() OVER (
        PARTITION BY VehicleKey, TripStartTime, TripEndTime 
        ORDER BY TripID
    ) AS RowNum
    FROM FactFleetTrips
)
DELETE FROM DuplicateTrips WHERE RowNum > 1;
```

### Integration Issues

**Problem:** Telematics API rate limiting
```sql
-- Implement exponential backoff in ETL stored procedure
CREATE PROCEDURE sp_LoadTelematicsData
AS
BEGIN
    DECLARE @RetryCount INT = 0;
    DECLARE @MaxRetries INT = 5;
    DECLARE @WaitSeconds INT = 2;
    
    WHILE @RetryCount < @MaxRetries
    BEGIN
        BEGIN TRY
            -- External API call via CLR or linked server
            EXEC sp_ExternalAPICall @Endpoint = '${TELEMATICS_API_ENDPOINT}';
            BREAK; -- Success
        END TRY
        BEGIN CATCH
            SET @RetryCount = @RetryCount + 1;
            WAITFOR DELAY @WaitSeconds;
            SET @WaitSeconds = @WaitSeconds * 2; -- Exponential backoff
        END CATCH
    END
END;
```

## Best Practices

1. **Indexing Strategy:**
   - Create covering indexes on frequently joined columns
   - Use filtered indexes for common WHERE clauses
   - Monitor index fragmentation weekly

2. **Data Refresh:**
   - Implement incremental loads for fact tables
   - Full refresh dimensions nightly
   - Partition large fact tables by month

3. **Security:**
   - Use row-level security in Power BI for multi-tenant scenarios
   - Encrypt connection strings using SQL Server Credential Manager
   - Audit all data access via SQL Server Audit

4. **Documentation:**
   - Maintain data dictionary in extended properties
   - Document all calculated measures in Power BI
   - Version control SQL scripts and Power BI templates in Git
