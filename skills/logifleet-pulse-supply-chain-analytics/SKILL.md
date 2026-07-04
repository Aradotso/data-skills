---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehouse for logistics and supply chain analytics with multi-fact star schema
triggers:
  - set up logifleet pulse supply chain analytics
  - deploy logistics data warehouse with power bi
  - configure warehouse and fleet tracking analytics
  - implement multi-fact star schema for logistics
  - build supply chain intelligence dashboard
  - create logistics kpi dashboard with sql server
  - integrate warehouse and fleet telematics data
  - set up cross-modal supply chain analytics
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

LogiFleet Pulse is a multi-modal logistics intelligence engine that combines warehouse operations, fleet telemetry, inventory management, and external data signals into a unified MS SQL Server data warehouse with Power BI visualization layer. It uses a custom multi-fact star schema with time-phased dimensions to enable cross-fact KPI analysis.

## What It Does

- **Unified data model** linking warehouse operations, fleet trips, cross-dock activities
- **Multi-fact star schema** with shared dimensions (time, geography, product, supplier)
- **Real-time dashboards** in Power BI with 15-minute refresh cycles
- **Predictive analytics** for bottleneck detection and maintenance prioritization
- **Warehouse gravity zones** that optimize storage based on velocity, value, and fragility
- **Cross-fact queries** that answer questions spanning multiple operational domains

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- Access to data sources: WMS, TMS, telematics APIs, ERP systems

### Deploy the SQL Schema

1. Clone the repository:
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. Connect to your SQL Server instance and execute the schema scripts:
```sql
-- Create database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute schema creation scripts in order
:r DimensionTables.sql
:r FactTables.sql
:r BridgeTables.sql
:r Views.sql
:r StoredProcedures.sql
:r Indexes.sql
```

3. Configure data source connections in `config_sample.json`:
```json
{
  "sqlServer": {
    "server": "localhost",
    "database": "LogiFleetPulse",
    "authentication": "SqlServer",
    "username": "${SQL_USER}",
    "password": "${SQL_PASSWORD}"
  },
  "dataSources": {
    "wms": {
      "apiUrl": "${WMS_API_URL}",
      "apiKey": "${WMS_API_KEY}"
    },
    "telematics": {
      "apiUrl": "${TELEMATICS_API_URL}",
      "apiKey": "${TELEMATICS_API_KEY}"
    }
  }
}
```

### Import Power BI Template

1. Open Power BI Desktop
2. File → Import → Power BI Template
3. Select `LogiFleet_Pulse_Master.pbit`
4. Enter SQL Server connection details when prompted
5. Configure row-level security based on user roles

## Core Data Model

### Dimension Tables

```sql
-- DimTime: 15-minute granularity with fiscal calendar
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME NOT NULL,
    FifteenMinuteBucket SMALLINT,
    HourOfDay TINYINT,
    DayOfWeek TINYINT,
    IsWeekend BIT,
    FiscalPeriod VARCHAR(10),
    FiscalYear SMALLINT
);

-- DimGeography: Hierarchical location model
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY,
    LocationID VARCHAR(50) NOT NULL,
    LocationType VARCHAR(20), -- Warehouse, Route Node, Customer
    LocationName NVARCHAR(100),
    Region VARCHAR(50),
    Country VARCHAR(50),
    Continent VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6)
);

-- DimProductGravity: Velocity-based classification
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL,
    ProductName NVARCHAR(200),
    Category NVARCHAR(100),
    GravityScore DECIMAL(5,2), -- Calculated from velocity, value, fragility
    GravityZone VARCHAR(20), -- High, Medium, Low
    UnitValue DECIMAL(10,2),
    FragilityIndex DECIMAL(3,2),
    WeightKg DECIMAL(8,2)
);

-- DimSupplierReliability: Performance metrics
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY,
    SupplierID VARCHAR(50) NOT NULL,
    SupplierName NVARCHAR(200),
    LeadTimeDays INT,
    LeadTimeVariance DECIMAL(5,2),
    DefectRate DECIMAL(5,4),
    ComplianceScore DECIMAL(3,2)
);
```

### Fact Tables

```sql
-- FactWarehouseOperations: All warehouse activities
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    OperationType VARCHAR(20), -- Receiving, Putaway, Picking, Packing, Shipping
    Quantity INT,
    DwellTimeMinutes INT,
    PickTimeSeconds INT,
    StorageZone VARCHAR(20),
    OperatorID VARCHAR(50)
);

-- FactFleetTrips: Route and vehicle telemetry
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleID VARCHAR(50),
    OriginGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DistanceKm DECIMAL(8,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(8,2),
    LoadWeightKg DECIMAL(10,2),
    DriverID VARCHAR(50)
);

-- FactCrossDock: Direct transfer operations
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    DwellTimeMinutes INT,
    Quantity INT
);
```

## Key Stored Procedures

### Incremental Data Loading

```sql
-- Incremental load for warehouse operations
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME
AS
BEGIN
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, GeographyKey, OperationType,
        Quantity, DwellTimeMinutes, PickTimeSeconds, StorageZone, OperatorID
    )
    SELECT 
        t.TimeKey,
        p.ProductKey,
        g.GeographyKey,
        wms.OperationType,
        wms.Quantity,
        DATEDIFF(MINUTE, wms.StartTime, wms.EndTime) AS DwellTimeMinutes,
        wms.PickTimeSeconds,
        wms.StorageZone,
        wms.OperatorID
    FROM ExternalWMS.dbo.Operations wms
    INNER JOIN DimTime t ON CAST(wms.OperationDateTime AS DATE) = CAST(t.DateTime AS DATE)
        AND DATEPART(HOUR, wms.OperationDateTime) = t.HourOfDay
        AND (DATEPART(MINUTE, wms.OperationDateTime) / 15) = t.FifteenMinuteBucket
    INNER JOIN DimProductGravity p ON wms.SKU = p.SKU
    INNER JOIN DimGeography g ON wms.WarehouseID = g.LocationID
    WHERE wms.OperationDateTime > @LastLoadDateTime;
END;
```

### Cross-Fact Analysis Views

```sql
-- Cross-fact KPI: Dwell time vs fleet idle time
CREATE VIEW vw_DwellTimeFleetImpact AS
SELECT 
    p.Category,
    g.Region,
    AVG(wo.DwellTimeMinutes) AS AvgWarehouseDwellMinutes,
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdleMinutes,
    COUNT(DISTINCT ft.TripKey) AS TripCount,
    SUM(ft.FuelConsumedLiters) AS TotalFuelLiters
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
INNER JOIN FactFleetTrips ft ON wo.GeographyKey = ft.OriginGeographyKey
    AND wo.TimeKey = ft.TimeKey
WHERE wo.OperationType = 'Shipping'
GROUP BY p.Category, g.Region;
```

### Gravity Zone Recommendation

```sql
-- Calculate product gravity scores and recommend zone reassignments
CREATE PROCEDURE usp_UpdateGravityScores
AS
BEGIN
    WITH ProductMetrics AS (
        SELECT 
            ProductKey,
            COUNT(*) AS PickFrequency,
            AVG(PickTimeSeconds) AS AvgPickTime,
            SUM(Quantity) AS TotalVolume
        FROM FactWarehouseOperations
        WHERE OperationType = 'Picking'
            AND TimeKey >= (SELECT MAX(TimeKey) - 2880 FROM DimTime) -- Last 30 days
        GROUP BY ProductKey
    )
    UPDATE p
    SET GravityScore = (
        (pm.PickFrequency * 0.4) + 
        ((1.0 / NULLIF(pm.AvgPickTime, 0)) * 0.3) +
        (p.UnitValue * 0.2) +
        (p.FragilityIndex * 0.1)
    ),
    GravityZone = CASE 
        WHEN GravityScore >= 75 THEN 'High'
        WHEN GravityScore >= 40 THEN 'Medium'
        ELSE 'Low'
    END
    FROM DimProductGravity p
    INNER JOIN ProductMetrics pm ON p.ProductKey = pm.ProductKey;
END;
```

## Power BI DAX Measures

### Cross-Fact KPIs

```dax
// Composite KPI: Fleet efficiency per warehouse gravity zone
Fleet Efficiency by Gravity = 
VAR TotalDistance = SUM(FactFleetTrips[DistanceKm])
VAR TotalIdleTime = SUM(FactFleetTrips[IdleTimeMinutes])
VAR HighGravityShipments = 
    CALCULATE(
        COUNT(FactWarehouseOperations[OperationKey]),
        FactWarehouseOperations[OperationType] = "Shipping",
        DimProductGravity[GravityZone] = "High"
    )
RETURN
DIVIDE(TotalDistance, TotalIdleTime + 1, 0) * HighGravityShipments

// Predictive bottleneck index
Bottleneck Risk Score = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR CurrentDwell = SUM(FactWarehouseOperations[DwellTimeMinutes])
VAR FleetUtilization = DIVIDE(
    SUM(FactFleetTrips[DurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[DurationMinutes]),
    0
)
RETURN
IF(
    CurrentDwell > AvgDwell * 1.5 && FleetUtilization > 0.85,
    1, // High risk
    0  // Normal
)
```

### Temporal Elasticity

```dax
// Simulate capacity change impact
Simulated Fleet Impact = 
VAR CapacityChange = [Capacity Slider Value] // Parameter from slicer
VAR BaselineTrips = COUNTROWS(FactFleetTrips)
VAR SimulatedTrips = BaselineTrips * (1 + CapacityChange)
VAR AvgFuelPerTrip = AVERAGE(FactFleetTrips[FuelConsumedLiters])
RETURN
SimulatedTrips * AvgFuelPerTrip
```

## Configuration and Setup

### Row-Level Security

```sql
-- Create role for regional managers
CREATE ROLE RegionalManager;

-- Apply filter
ALTER ROLE RegionalManager ADD MEMBER [DOMAIN\RegionalUser];

-- Define RLS in Power BI model
[Region Security] = 
IF(
    USERNAME() = "global@company.com",
    TRUE(), // Global admin sees all
    DimGeography[Region] = LOOKUPVALUE(
        UserRegions[Region],
        UserRegions[Email],
        USERNAME()
    )
)
```

### Automated Alerts

```sql
-- Alert stored procedure for KPI breaches
CREATE PROCEDURE usp_CheckKPIAlerts
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    -- Check fleet idle time threshold
    IF EXISTS (
        SELECT 1 
        FROM FactFleetTrips ft
        INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
        WHERE t.DateTime >= DATEADD(HOUR, -1, GETDATE())
            AND (CAST(ft.IdleTimeMinutes AS FLOAT) / ft.DurationMinutes) > 0.15
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Fleet idle time exceeded 15% threshold in last hour';
        
        -- Send notification (integrate with email/Teams)
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = '${ALERT_EMAIL}',
            @subject = 'LogiFleet Alert',
            @body = @AlertMessage;
    END;
END;

-- Schedule via SQL Agent job (every 15 minutes)
```

## Common Patterns

### Query Cross-Fact Relationships

```sql
-- Find shipments delayed by weather affecting specific products
SELECT 
    p.SKU,
    p.ProductName,
    g.LocationName AS Warehouse,
    wo.DwellTimeMinutes,
    ft.IdleTimeMinutes,
    ext.WeatherCondition,
    ext.TrafficDelay
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
INNER JOIN FactFleetTrips ft ON wo.GeographyKey = ft.OriginGeographyKey
    AND wo.TimeKey = ft.TimeKey
LEFT JOIN ExternalWeatherAPI ext ON g.Latitude = ext.Lat 
    AND g.Longitude = ext.Lon
    AND wo.TimeKey = ext.TimeKey
WHERE wo.DwellTimeMinutes > 72 * 60 -- More than 72 hours
    AND ext.TrafficDelay > 30 -- 30+ minutes delay
    AND p.GravityZone = 'High';
```

### Batch Data Refresh

```sql
-- Full refresh orchestration
CREATE PROCEDURE usp_RefreshAllFacts
AS
BEGIN
    DECLARE @LastLoad DATETIME;
    
    -- Get last successful load timestamp
    SELECT @LastLoad = MAX(LoadDateTime) FROM ETL_AuditLog;
    
    -- Load each fact table
    EXEC usp_LoadWarehouseOperations @LastLoad;
    EXEC usp_LoadFleetTrips @LastLoad;
    EXEC usp_LoadCrossDock @LastLoad;
    
    -- Update gravity scores
    EXEC usp_UpdateGravityScores;
    
    -- Log completion
    INSERT INTO ETL_AuditLog (LoadDateTime, Status)
    VALUES (GETDATE(), 'Success');
END;
```

## Troubleshooting

### Performance Issues

```sql
-- Check for missing indexes
SELECT 
    t.name AS TableName,
    i.name AS IndexName,
    dm_mid.user_seeks,
    dm_mid.user_scans,
    dm_migs.avg_total_user_cost,
    dm_migs.avg_user_impact
FROM sys.dm_db_missing_index_details dm_mid
INNER JOIN sys.dm_db_missing_index_groups dm_mig 
    ON dm_mid.index_handle = dm_mig.index_handle
INNER JOIN sys.dm_db_missing_index_group_stats dm_migs 
    ON dm_mig.index_group_handle = dm_migs.group_handle
INNER JOIN sys.tables t ON dm_mid.object_id = t.object_id
LEFT JOIN sys.indexes i ON t.object_id = i.object_id
WHERE dm_migs.avg_user_impact > 50
ORDER BY dm_migs.avg_user_impact DESC;
```

### Data Quality Validation

```sql
-- Detect orphaned fact records
SELECT 'Orphaned Products' AS Issue, COUNT(*) AS RecordCount
FROM FactWarehouseOperations wo
LEFT JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
WHERE p.ProductKey IS NULL

UNION ALL

SELECT 'Missing Geography' AS Issue, COUNT(*) AS RecordCount
FROM FactFleetTrips ft
LEFT JOIN DimGeography g ON ft.OriginGeographyKey = g.GeographyKey
WHERE g.GeographyKey IS NULL;
```

### Power BI Refresh Failures

Check connection strings and credentials:
```powerquery
// In Power Query Editor
let
    Source = Sql.Database(
        #"SQL_SERVER" meta [CredentialEncryption = CredentialEncryption.Encrypted], 
        "LogiFleetPulse"
    ),
    Navigation = Source{[Schema="dbo"]}[Data]
in
    Navigation
```

## Best Practices

1. **Partition large fact tables** by TimeKey for query performance
2. **Use columnstore indexes** on fact tables for analytical queries
3. **Refresh Power BI datasets** every 15 minutes during business hours
4. **Implement incremental refresh** in Power BI for fact tables over 1M rows
5. **Archive historical data** older than 2 years to separate database
6. **Monitor ETL job duration** and adjust batch sizes if loads exceed 10 minutes
7. **Use parameters** in Power BI for dynamic filtering (regions, time ranges)
8. **Document calculation logic** for all custom DAX measures
