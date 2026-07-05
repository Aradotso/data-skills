---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehouse with multi-fact star schema for supply chain, fleet, and warehouse intelligence
triggers:
  - set up logifleet pulse supply chain analytics
  - deploy logistics data warehouse with power bi
  - configure fleet and warehouse intelligence dashboards
  - implement multi-fact star schema for logistics
  - create supply chain analytics with sql server
  - build logistics kpi dashboard in power bi
  - integrate warehouse and fleet telemetry data
  - design cross-modal supply chain data model
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive logistics intelligence platform built on MS SQL Server and Power BI. It provides a multi-fact star schema data warehouse that integrates:

- **Warehouse operations** (receiving, putaway, picking, packing, shipping)
- **Fleet telemetry** (GPS, fuel consumption, driver behavior, vehicle diagnostics)
- **Inventory management** (dwell time, turnover rates, aging curves)
- **Supplier data** (lead times, defect rates, reliability scoring)
- **External signals** (weather, traffic, demand seasonality)

The platform uses time-phased dimensions, bridge tables for many-to-many relationships, and composite KPIs to provide unified cross-modal analytics.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to source systems (WMS, TMS, telematics APIs)

### Database Deployment

1. **Clone the repository:**

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**

```sql
-- Connect to your SQL Server instance via SSMS
-- Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Run the schema creation script
-- (Assuming schema.sql is provided in the repository)
-- Execute the complete schema deployment
```

3. **Configure data sources:**

Create a configuration file based on `config_sample.json`:

```json
{
  "connections": {
    "wms_database": {
      "server": "${WMS_SQL_SERVER}",
      "database": "${WMS_DATABASE}",
      "username": "${WMS_SQL_USER}",
      "password": "${WMS_SQL_PASSWORD}"
    },
    "fleet_api": {
      "endpoint": "${FLEET_TELEMETRY_API_URL}",
      "api_key": "${FLEET_API_KEY}"
    },
    "weather_api": {
      "endpoint": "https://api.weather.service/v1",
      "api_key": "${WEATHER_API_KEY}"
    }
  },
  "refresh_intervals": {
    "warehouse_operations": "15min",
    "fleet_trips": "15min",
    "supplier_data": "daily"
  }
}
```

## Core Data Model

### Fact Tables

**FactWarehouseOperations** - Tracks warehouse micro-operations:

```sql
CREATE TABLE FactWarehouseOperations (
    OperationID INT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    DateKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    QuantityHandled INT,
    CycleTimeMinutes DECIMAL(10,2),
    DwellTimeHours DECIMAL(10,2),
    OperatorID INT,
    ZoneKey INT,
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (DateKey) REFERENCES DimDate(DateKey),
    FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);

-- Create clustered columnstore index for analytics performance
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactWarehouseOperations 
ON FactWarehouseOperations;
```

**FactFleetTrips** - Tracks vehicle journey segments:

```sql
CREATE TABLE FactFleetTrips (
    TripID INT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    DateKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT,
    OriginGeographyKey INT,
    DestinationGeographyKey INT,
    DistanceKM DECIMAL(10,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(10,2),
    LoadWeight DECIMAL(10,2),
    DelayMinutes INT,
    DelayReason VARCHAR(100),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey)
);

CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactFleetTrips 
ON FactFleetTrips;
```

**FactCrossDock** - Transfers without long-term storage:

```sql
CREATE TABLE FactCrossDock (
    CrossDockID INT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    DateKey INT NOT NULL,
    InboundTripKey INT,
    OutboundTripKey INT,
    ProductKey INT,
    QuantityTransferred INT,
    TransferTimeMinutes INT,
    DockBayID INT,
    FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);
```

### Dimension Tables

**DimTime** - 15-minute granularity time slices:

```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    TimeValue TIME,
    Hour INT,
    Minute INT,
    QuarterHour INT, -- 1-4 for 15-min buckets
    ShiftID INT, -- 1=Morning, 2=Afternoon, 3=Night
    IsBusinessHours BIT
);

-- Populate 15-minute buckets for 24 hours
INSERT INTO DimTime (TimeKey, TimeValue, Hour, Minute, QuarterHour, ShiftID, IsBusinessHours)
SELECT 
    DATEPART(HOUR, t) * 100 + DATEPART(MINUTE, t) AS TimeKey,
    t AS TimeValue,
    DATEPART(HOUR, t) AS Hour,
    DATEPART(MINUTE, t) AS Minute,
    CEILING(DATEPART(MINUTE, t) / 15.0) AS QuarterHour,
    CASE 
        WHEN DATEPART(HOUR, t) BETWEEN 6 AND 13 THEN 1
        WHEN DATEPART(HOUR, t) BETWEEN 14 AND 21 THEN 2
        ELSE 3
    END AS ShiftID,
    CASE 
        WHEN DATEPART(HOUR, t) BETWEEN 8 AND 17 THEN 1 
        ELSE 0 
    END AS IsBusinessHours
FROM (
    SELECT CAST('00:00' AS TIME) AS t
    UNION ALL SELECT CAST('00:15' AS TIME)
    UNION ALL SELECT CAST('00:30' AS TIME)
    -- ... continue for all 96 15-minute intervals
) AS TimeBuckets;
```

**DimProductGravity** - Products with velocity-based scoring:

```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50) UNIQUE,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    Subcategory VARCHAR(100),
    IsPerishable BIT,
    FragilityScore INT, -- 1-10
    UnitValue DECIMAL(10,2),
    VelocityScore DECIMAL(5,2), -- Calculated weekly from FactWarehouseOperations
    GravityScore DECIMAL(5,2), -- Composite: Velocity * Value * Fragility
    RecommendedZoneType VARCHAR(50) -- 'HighGravity', 'MediumGravity', 'LowGravity'
);

-- Stored procedure to recalculate gravity scores
CREATE PROCEDURE sp_RecalculateProductGravity
AS
BEGIN
    UPDATE DimProductGravity
    SET 
        VelocityScore = v.PickFrequency,
        GravityScore = (v.PickFrequency * UnitValue * FragilityScore) / 100
    FROM DimProductGravity p
    INNER JOIN (
        SELECT 
            ProductKey,
            COUNT(*) / 7.0 AS PickFrequency -- Picks per day over last week
        FROM FactWarehouseOperations
        WHERE OperationType = 'Picking'
        AND DateKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -7, GETDATE()), 'yyyyMMdd'))
        GROUP BY ProductKey
    ) v ON p.ProductKey = v.ProductKey;
    
    -- Assign recommended zones based on gravity score distribution
    UPDATE DimProductGravity
    SET RecommendedZoneType = CASE
        WHEN GravityScore >= (SELECT PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY GravityScore) OVER() FROM DimProductGravity) THEN 'HighGravity'
        WHEN GravityScore >= (SELECT PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY GravityScore) OVER() FROM DimProductGravity) THEN 'MediumGravity'
        ELSE 'LowGravity'
    END;
END;
```

**DimGeography** - Hierarchical location data:

```sql
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY,
    LocationID VARCHAR(50),
    LocationName VARCHAR(200),
    LocationType VARCHAR(50), -- 'Warehouse', 'RouteNode', 'CustomerSite'
    AddressLine1 VARCHAR(200),
    City VARCHAR(100),
    StateProvince VARCHAR(100),
    Country VARCHAR(100),
    PostalCode VARCHAR(20),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    RegionKey INT,
    TimeZone VARCHAR(50)
);
```

## Data Integration Patterns

### Incremental Load from WMS

```sql
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadTimestamp DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Stage data from external WMS database
    INSERT INTO FactWarehouseOperations (
        TimeKey, DateKey, WarehouseKey, ProductKey, 
        OperationType, QuantityHandled, CycleTimeMinutes, DwellTimeHours
    )
    SELECT 
        CAST(FORMAT(w.OperationTimestamp, 'HHmm') AS INT) / 15 * 15 AS TimeKey,
        CAST(FORMAT(w.OperationTimestamp, 'yyyyMMdd') AS INT) AS DateKey,
        dw.WarehouseKey,
        dp.ProductKey,
        w.OperationType,
        w.Quantity,
        DATEDIFF(MINUTE, w.StartTime, w.EndTime) AS CycleTimeMinutes,
        CASE 
            WHEN w.OperationType = 'Receiving' 
            THEN DATEDIFF(HOUR, w.EndTime, COALESCE(next_op.StartTime, GETDATE()))
            ELSE NULL 
        END AS DwellTimeHours
    FROM OPENQUERY([WMS_LINKED_SERVER], 
        'SELECT * FROM WarehouseOperations WHERE LastModified > ''' + CONVERT(VARCHAR, @LastLoadTimestamp, 120) + '''') w
    LEFT JOIN DimWarehouse dw ON w.WarehouseCode = dw.WarehouseCode
    LEFT JOIN DimProduct dp ON w.SKU = dp.SKU
    LEFT JOIN OPENQUERY([WMS_LINKED_SERVER], 
        'SELECT * FROM WarehouseOperations') next_op 
        ON w.BatchID = next_op.BatchID AND next_op.SequenceNum = w.SequenceNum + 1
    WHERE w.OperationTimestamp > @LastLoadTimestamp;
    
    -- Update last load timestamp
    UPDATE ETL_Config SET LastLoadTimestamp = GETDATE() WHERE TableName = 'FactWarehouseOperations';
END;
```

### Fleet Telemetry API Integration

```sql
CREATE PROCEDURE sp_LoadFleetTelemetry
AS
BEGIN
    -- Create staging table for API response
    IF OBJECT_ID('tempdb..#FleetStaging') IS NOT NULL DROP TABLE #FleetStaging;
    
    CREATE TABLE #FleetStaging (
        VehicleID VARCHAR(50),
        TripStartTime DATETIME,
        TripEndTime DATETIME,
        OriginCode VARCHAR(50),
        DestinationCode VARCHAR(50),
        Distance DECIMAL(10,2),
        FuelConsumed DECIMAL(10,2),
        IdleMinutes INT,
        DelayMinutes INT,
        DelayReason VARCHAR(100)
    );
    
    -- Call API via CLR stored procedure or external data source
    -- (Assuming sp_InvokeFleetAPI is a CLR procedure that calls REST API)
    DECLARE @ApiResponse NVARCHAR(MAX);
    EXEC sp_InvokeFleetAPI 
        @Endpoint = '${FLEET_TELEMETRY_API_URL}/trips/recent',
        @ApiKey = '${FLEET_API_KEY}',
        @Response = @ApiResponse OUTPUT;
    
    -- Parse JSON response
    INSERT INTO #FleetStaging
    SELECT 
        vehicleId,
        tripStartTime,
        tripEndTime,
        originCode,
        destinationCode,
        distanceKm,
        fuelLiters,
        idleMinutes,
        delayMinutes,
        delayReason
    FROM OPENJSON(@ApiResponse)
    WITH (
        vehicleId VARCHAR(50) '$.vehicleId',
        tripStartTime DATETIME '$.tripStartTime',
        tripEndTime DATETIME '$.tripEndTime',
        originCode VARCHAR(50) '$.origin.code',
        destinationCode VARCHAR(50) '$.destination.code',
        distanceKm DECIMAL(10,2) '$.metrics.distanceKm',
        fuelLiters DECIMAL(10,2) '$.metrics.fuelLiters',
        idleMinutes INT '$.metrics.idleMinutes',
        delayMinutes INT '$.metrics.delayMinutes',
        delayReason VARCHAR(100) '$.metrics.delayReason'
    );
    
    -- Insert into fact table
    INSERT INTO FactFleetTrips (
        TimeKey, DateKey, VehicleKey, OriginGeographyKey, DestinationGeographyKey,
        DistanceKM, DurationMinutes, IdleTimeMinutes, FuelConsumedLiters, DelayMinutes, DelayReason
    )
    SELECT 
        CAST(FORMAT(s.TripStartTime, 'HHmm') AS INT) / 15 * 15 AS TimeKey,
        CAST(FORMAT(s.TripStartTime, 'yyyyMMdd') AS INT) AS DateKey,
        dv.VehicleKey,
        og.GeographyKey AS OriginGeographyKey,
        dg.GeographyKey AS DestinationGeographyKey,
        s.Distance,
        DATEDIFF(MINUTE, s.TripStartTime, s.TripEndTime) AS DurationMinutes,
        s.IdleMinutes,
        s.FuelConsumed,
        s.DelayMinutes,
        s.DelayReason
    FROM #FleetStaging s
    INNER JOIN DimVehicle dv ON s.VehicleID = dv.VehicleID
    INNER JOIN DimGeography og ON s.OriginCode = og.LocationID
    INNER JOIN DimGeography dg ON s.DestinationCode = dg.LocationID;
END;
```

## Power BI Configuration

### Connecting to SQL Server

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. When prompted, enter your SQL Server connection details:
   - **Server:** Your SQL Server instance name
   - **Database:** LogiFleetPulse
   - **Data Connectivity mode:** DirectQuery (for real-time) or Import (for performance)

### Key DAX Measures

**Fleet Idle Percentage:**

```dax
Fleet Idle % = 
DIVIDE(
    SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[DurationMinutes]),
    0
) * 100
```

**Cross-Fact KPI: Dwell Time vs Fleet Delay Correlation:**

```dax
Dwell-Delay Correlation = 
VAR DwellTable = 
    SUMMARIZE(
        FactWarehouseOperations,
        DimProduct[ProductKey],
        "AvgDwell", AVERAGE(FactWarehouseOperations[DwellTimeHours])
    )
VAR DelayTable = 
    SUMMARIZE(
        FactFleetTrips,
        DimProduct[ProductKey],
        "AvgDelay", AVERAGE(FactFleetTrips[DelayMinutes])
    )
VAR JoinedTable = 
    NATURALINNERJOIN(DwellTable, DelayTable)
RETURN
    CORRELATIONX(
        JoinedTable,
        [AvgDwell],
        [AvgDelay]
    )
```

**Warehouse Gravity Zone Efficiency:**

```dax
Gravity Zone Match % = 
VAR TotalOps = COUNTROWS(FactWarehouseOperations)
VAR MatchedOps = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        DimProduct[RecommendedZoneType] = DimWarehouse[ActualZoneType]
    )
RETURN
    DIVIDE(MatchedOps, TotalOps, 0) * 100
```

**Predictive Bottleneck Index:**

```dax
Bottleneck Risk Score = 
VAR CurrentUtilization = 
    DIVIDE(
        CALCULATE(SUM(FactWarehouseOperations[QuantityHandled]), DimTime[IsBusinessHours] = TRUE),
        MAX(DimWarehouse[MaxDailyCapacity]),
        0
    )
VAR FleetUtilization = 
    DIVIDE(
        CALCULATE(COUNTROWS(FactFleetTrips), FactFleetTrips[DelayMinutes] > 0),
        COUNTROWS(FactFleetTrips),
        0
    )
VAR SupplierRisk = AVERAGE(DimSupplier[ReliabilityScore]) / 100
RETURN
    (CurrentUtilization * 0.5 + FleetUtilization * 0.3 + (1 - SupplierRisk) * 0.2) * 100
```

### Row-Level Security (RLS)

```dax
-- Create role: Regional Manager
[GeographyRegion] = USERNAME()

-- Create role: Warehouse Supervisor
[WarehouseCode] = LOOKUPVALUE(
    DimUser[AssignedWarehouse],
    DimUser[Email], USERNAME()
)
```

Apply in Power BI Desktop:
1. Modeling tab → Manage Roles
2. Create roles and define DAX filters on dimension tables
3. Test using "View as Role"

## Common Analytical Patterns

### Query: Top 10 High-Dwell Products with Fleet Impact

```sql
SELECT TOP 10
    p.SKU,
    p.ProductName,
    p.GravityScore,
    AVG(wo.DwellTimeHours) AS AvgDwellHours,
    COUNT(DISTINCT ft.TripID) AS AffectedTrips,
    AVG(ft.DelayMinutes) AS AvgTripDelayMinutes
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
LEFT JOIN FactFleetTrips ft ON wo.ProductKey = ft.ProductKey  -- Assuming bridge table exists
WHERE wo.OperationType = 'Receiving'
AND wo.DateKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMdd'))
GROUP BY p.SKU, p.ProductName, p.GravityScore
HAVING AVG(wo.DwellTimeHours) > 48
ORDER BY AvgDwellHours DESC, AffectedTrips DESC;
```

### Query: Route Optimization Candidates (High Idle Time Routes)

```sql
WITH RouteMetrics AS (
    SELECT 
        r.RouteID,
        r.RouteName,
        COUNT(*) AS TripCount,
        AVG(ft.IdleTimeMinutes) AS AvgIdleMinutes,
        AVG(ft.FuelConsumedLiters) AS AvgFuelLiters,
        SUM(ft.DelayMinutes) AS TotalDelayMinutes
    FROM FactFleetTrips ft
    INNER JOIN DimRoute r ON ft.RouteKey = r.RouteKey
    WHERE ft.DateKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -90, GETDATE()), 'yyyyMMdd'))
    GROUP BY r.RouteID, r.RouteName
)
SELECT 
    RouteID,
    RouteName,
    TripCount,
    AvgIdleMinutes,
    AvgFuelLiters,
    TotalDelayMinutes,
    RANK() OVER (ORDER BY AvgIdleMinutes DESC) AS IdleRank,
    CASE 
        WHEN AvgIdleMinutes > 30 AND TotalDelayMinutes > 500 THEN 'High Priority'
        WHEN AvgIdleMinutes > 20 THEN 'Medium Priority'
        ELSE 'Monitor'
    END AS OptimizationPriority
FROM RouteMetrics
WHERE TripCount >= 10
ORDER BY AvgIdleMinutes DESC;
```

### Query: Seasonal Demand Pattern Detection

```sql
SELECT 
    d.MonthName,
    d.WeekOfMonth,
    p.Category,
    SUM(wo.QuantityHandled) AS TotalQuantity,
    AVG(wo.CycleTimeMinutes) AS AvgCycleTime,
    COUNT(DISTINCT wo.DateKey) AS ActiveDays
FROM FactWarehouseOperations wo
INNER JOIN DimDate d ON wo.DateKey = d.DateKey
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
WHERE wo.OperationType = 'Picking'
AND d.Year = YEAR(GETDATE()) - 1  -- Last year's pattern
GROUP BY d.MonthName, d.WeekOfMonth, p.Category
ORDER BY d.MonthNumber, d.WeekOfMonth, TotalQuantity DESC;
```

## Alerting & Automation

### SQL Agent Job for KPI Breach Alerts

```sql
CREATE PROCEDURE sp_CheckKPIBreaches
AS
BEGIN
    DECLARE @AlertMessages TABLE (
        AlertType VARCHAR(100),
        Message VARCHAR(500),
        Severity INT
    );
    
    -- Check 1: Warehouse capacity breach
    INSERT INTO @AlertMessages
    SELECT 
        'Warehouse Capacity',
        'Warehouse ' + w.WarehouseName + ' is at ' + 
        CAST(ROUND((CurrentLoad * 100.0 / MaxCapacity), 2) AS VARCHAR) + '% capacity',
        CASE WHEN CurrentLoad > MaxCapacity * 0.95 THEN 3 ELSE 2 END
    FROM (
        SELECT 
            w.WarehouseKey,
            w.WarehouseName,
            w.MaxDailyCapacity AS MaxCapacity,
            COUNT(*) AS CurrentLoad
        FROM FactWarehouseOperations wo
        INNER JOIN DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
        WHERE wo.DateKey = CONVERT(INT, FORMAT(GETDATE(), 'yyyyMMdd'))
        AND wo.OperationType = 'Picking'
        GROUP BY w.WarehouseKey, w.WarehouseName, w.MaxDailyCapacity
    ) w
    WHERE CurrentLoad > MaxCapacity * 0.85;
    
    -- Check 2: Fleet idle time threshold
    INSERT INTO @AlertMessages
    SELECT 
        'Fleet Idle',
        'Route ' + r.RouteName + ' has ' + 
        CAST(ROUND(AVG(IdleTimeMinutes), 1) AS VARCHAR) + ' min avg idle time today',
        2
    FROM FactFleetTrips ft
    INNER JOIN DimRoute r ON ft.RouteKey = r.RouteKey
    WHERE ft.DateKey = CONVERT(INT, FORMAT(GETDATE(), 'yyyyMMdd'))
    GROUP BY r.RouteKey, r.RouteName
    HAVING AVG(IdleTimeMinutes) > 25;
    
    -- Check 3: High-gravity product in wrong zone
    INSERT INTO @AlertMessages
    SELECT 
        'Gravity Mismatch',
        'Product ' + p.SKU + ' (Gravity: ' + CAST(p.GravityScore AS VARCHAR) + 
        ') is in ' + z.ZoneType + ' zone',
        2
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    INNER JOIN DimZone z ON wo.ZoneKey = z.ZoneKey
    WHERE p.RecommendedZoneType = 'HighGravity'
    AND z.ZoneType <> 'HighGravity'
    AND wo.DateKey = CONVERT(INT, FORMAT(GETDATE(), 'yyyyMMdd'));
    
    -- Send alerts via Database Mail
    IF EXISTS (SELECT 1 FROM @AlertMessages)
    BEGIN
        DECLARE @EmailBody NVARCHAR(MAX);
        SET @EmailBody = 
            '<h2>LogiFleet Pulse Daily Alerts</h2>' +
            '<table border="1"><tr><th>Type</th><th>Message</th><th>Severity</th></tr>';
        
        SELECT @EmailBody = @EmailBody + 
            '<tr><td>' + AlertType + '</td><td>' + Message + '</td><td>' + 
            CASE Severity 
                WHEN 3 THEN '<b>CRITICAL</b>'
                WHEN 2 THEN 'WARNING'
                ELSE 'INFO'
            END + '</td></tr>'
        FROM @AlertMessages;
        
        SET @EmailBody = @EmailBody + '</table>';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL_RECIPIENTS}',
            @subject = 'LogiFleet Pulse: Daily KPI Alerts',
            @body = @EmailBody,
            @body_format = 'HTML';
    END
END;
```

Schedule via SQL Server Agent:
```sql
EXEC msdb.dbo.sp_add_job @job_name = 'LogiFleet_KPI_Alerts';
EXEC msdb.dbo.sp_add_jobstep 
    @job_name = 'LogiFleet_KPI_Alerts',
    @step_name = 'Check_KPIs',
    @command = 'EXEC sp_CheckKPIBreaches';
EXEC msdb.dbo.sp_add_schedule 
    @schedule_name = 'Daily_6PM',
    @freq_type = 4, -- Daily
    @active_start_time = 180000; -- 6:00 PM
EXEC msdb.dbo.sp_attach_schedule 
    @job_name = 'LogiFleet_KPI_Alerts',
    @schedule_name = 'Daily_6PM';
```

## Troubleshooting

### Issue: Power BI Refresh Timeout

**Symptom:** DirectQuery times out or Import mode fails on large fact tables.

**Solution:**
1. Create indexed views for common aggregations:

```sql
CREATE VIEW vw_DailyWarehouseMetrics
WITH SCHEMABINDING
AS
SELECT 
    wo.DateKey,
    wo.WarehouseKey,
    wo.ProductKey,
    SUM(wo.QuantityHandled) AS TotalQuantity,
    AVG(wo.CycleTimeMinutes) AS AvgCycleTime,
    COUNT_BIG(*) AS OperationCount
FROM dbo.FactWarehouseOperations wo
GROUP BY wo.DateKey, wo.WarehouseKey, wo.ProductKey;

CREATE UNIQUE CLUSTERED INDEX IX_DailyWarehouseMetrics 
ON vw_DailyWarehouseMetrics(DateKey, WarehouseKey, ProductKey);
```

2. Use incremental refresh in Power BI:
   - Add `RangeStart` and `RangeEnd` parameters
   - Filter fact tables: `WHERE DateKey >= RangeStart AND DateKey < RangeEnd`
   - Configure incremental refresh policy (e.g., refresh last 7 days, archive 2 years)

### Issue: Cross-Fact Relationships Not Working

**Symptom:** DAX measures return BLANK when joining warehouse and fleet data.

**Solution:** Create a bridge table for many-to-many relationships:

```sql
CREATE TABLE BridgeProductTrip (
    ProductKey INT,
    TripID INT,
    QuantityShipped INT,
    PRIMARY KEY (ProductKey, TripID)
);

-- Populate from shipping manifests
INSERT INTO BridgeProductTrip (ProductKey, TripID, QuantityShipped)
SELECT 
    wo.ProductKey,
    ft.TripID,
    wo.QuantityHandled
FROM FactWarehouseOperations wo
INNER JOIN DimShipment s ON wo.ShipmentID = s.ShipmentID
INNER JOIN FactFleetTrips ft ON s.TripID = ft.TripID
WHERE wo.OperationType = 'Shipping';
```

In Power BI, set cardinality to Many-to-Many and cross-filter direction to Both.

### Issue: Gravity Score Not Updating

**Symptom:** `DimProductGravity.GravityScore` remains static despite new warehouse operations.

**Solution:** Schedule the recalculation stored procedure:

```sql
EXEC msdb.dbo.sp_add_job @job_name = 'Recalc_Gravity_Scores';
EXEC msdb.dbo.sp_add_jobstep 
    @job_name = 'Recalc_Gravity_Scores',
    @step_name = 'Recalculate',
    @command = 'EXEC sp_RecalculateProductGravity';
EXEC msdb.dbo.sp_add_schedule 
