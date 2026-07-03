---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing for logistics, fleet management, and supply chain intelligence with multi-fact star schema
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "create logistics data warehouse with Power BI"
  - "implement fleet and warehouse KPI dashboard"
  - "configure supply chain star schema in SQL Server"
  - "build cross-modal logistics intelligence system"
  - "integrate warehouse and fleet telemetry data"
  - "deploy multi-fact logistics data model"
  - "troubleshoot LogiFleet Pulse dashboard connections"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI agents to help developers implement LogiFleet Pulse, an advanced logistics intelligence platform combining MS SQL Server data warehousing with Power BI dashboards. It unifies warehouse operations, fleet telemetry, and supply chain metrics through a multi-fact star schema architecture.

## What LogiFleet Pulse Does

LogiFleet Pulse is a **logistics data warehouse and analytics platform** that:

- **Unifies disparate data sources**: WMS, telematics, GPS, supplier portals, external APIs (weather, traffic)
- **Multi-fact star schema**: Links warehouse operations, fleet trips, and cross-dock activities through shared dimensions
- **Real-time dashboards**: Power BI reports refreshed every 15 minutes for operational awareness
- **Predictive analytics**: Bottleneck detection, fleet maintenance triage, warehouse gravity zone optimization
- **Cross-KPI analysis**: Correlates metrics across domains (e.g., dwell time vs. fuel costs)

Core components:
- SQL Server database schema (fact tables, dimensions, bridges)
- Power BI template (`.pbit`) with pre-built semantic layer
- Stored procedures for incremental data loading
- Alert triggers and role-based security

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- Access to source systems: WMS, TMS, telematics API, ERP

### Step 1: Clone Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### Step 2: Deploy SQL Schema

Connect to your SQL Server instance and execute the schema creation script:

```sql
-- Run the main schema creation script
-- This creates all fact tables, dimensions, views, and stored procedures
:r schema/01_create_database.sql
:r schema/02_create_dimensions.sql
:r schema/03_create_facts.sql
:r schema/04_create_bridges.sql
:r schema/05_create_views.sql
:r schema/06_create_stored_procedures.sql
```

Or use `sqlcmd` from command line:

```bash
sqlcmd -S localhost -U sa -P $SQL_SERVER_PASSWORD -i schema/01_create_database.sql
sqlcmd -S localhost -U sa -P $SQL_SERVER_PASSWORD -i schema/02_create_dimensions.sql
sqlcmd -S localhost -U sa -P $SQL_SERVER_PASSWORD -i schema/03_create_facts.sql
sqlcmd -S localhost -U sa -P $SQL_SERVER_PASSWORD -i schema/04_create_bridges.sql
sqlcmd -S localhost -U sa -P $SQL_SERVER_PASSWORD -i schema/05_create_views.sql
sqlcmd -S localhost -U sa -P $SQL_SERVER_PASSWORD -i schema/06_create_stored_procedures.sql
```

### Step 3: Configure Data Sources

Copy and customize the configuration template:

```bash
cp config_sample.json config.json
```

Edit `config.json` with your connection strings:

```json
{
  "sql_server": {
    "server": "localhost",
    "database": "LogiFleetPulse",
    "username": "${SQL_USERNAME}",
    "password": "${SQL_PASSWORD}"
  },
  "data_sources": {
    "wms_api": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "api_key": "${WMS_API_KEY}"
    },
    "telematics_api": {
      "endpoint": "${TELEMATICS_API_ENDPOINT}",
      "api_key": "${TELEMATICS_API_KEY}"
    },
    "weather_api": {
      "endpoint": "https://api.weather.com/v3",
      "api_key": "${WEATHER_API_KEY}"
    }
  },
  "refresh_interval_minutes": 15
}
```

### Step 4: Import Power BI Template

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. When prompted, enter SQL Server connection details
3. Configure data refresh schedule (15-minute intervals recommended)
4. Publish to Power BI Service workspace

## Key Database Schema Components

### Core Fact Tables

**FactWarehouseOperations** - Warehouse activity events:

```sql
CREATE TABLE FactWarehouseOperations (
    OperationID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType NVARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    Quantity DECIMAL(18,2),
    DwellTimeMinutes INT,
    PickRateUnitsPerHour DECIMAL(10,2),
    StorageZoneKey INT,
    EmployeeKey INT,
    CONSTRAINT FK_WH_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WH_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_WH_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
);

-- Indexing strategy for performance
CREATE NONCLUSTERED INDEX IX_WH_Time_Warehouse ON FactWarehouseOperations(TimeKey, WarehouseKey);
CREATE NONCLUSTERED INDEX IX_WH_Product_DwellTime ON FactWarehouseOperations(ProductKey, DwellTimeMinutes);
```

**FactFleetTrips** - Vehicle route and telemetry data:

```sql
CREATE TABLE FactFleetTrips (
    TripID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT NOT NULL,
    OriginGeographyKey INT,
    DestinationGeographyKey INT,
    DistanceKM DECIMAL(10,2),
    TripDurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(10,2),
    LoadWeightKG DECIMAL(12,2),
    OnTimeDeliveryFlag BIT,
    DelayReasonKey INT,
    CONSTRAINT FK_FT_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FT_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey)
);

CREATE NONCLUSTERED INDEX IX_FT_Time_Route ON FactFleetTrips(TimeKey, RouteKey);
CREATE NONCLUSTERED INDEX IX_FT_OnTime_Delay ON FactFleetTrips(OnTimeDeliveryFlag, DelayReasonKey);
```

**FactCrossDock** - Cross-dock transfer operations:

```sql
CREATE TABLE FactCrossDock (
    CrossDockID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    ProductKey INT NOT NULL,
    Quantity DECIMAL(18,2),
    TransferDurationMinutes INT,
    CONSTRAINT FK_CD_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_CD_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
);
```

### Dimension Tables

**DimTime** - Time-phased dimension with 15-minute granularity:

```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY, -- Format: YYYYMMDDHHMM
    FullDate DATETIME NOT NULL,
    Year INT,
    Quarter INT,
    Month INT,
    Day INT,
    Hour INT,
    Minute15Bucket INT, -- 0, 15, 30, 45
    DayOfWeek NVARCHAR(10),
    IsWeekend BIT,
    FiscalPeriod NVARCHAR(20)
);

-- Populate time dimension
EXEC sp_PopulateTimeDimension @StartDate = '2025-01-01', @EndDate = '2027-12-31';
```

**DimProductGravity** - Product hierarchy with gravity scoring:

```sql
CREATE TABLE DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU NVARCHAR(50) UNIQUE NOT NULL,
    ProductName NVARCHAR(200),
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value * (1/fragility)
    VelocityClass NVARCHAR(20), -- 'Fast', 'Medium', 'Slow'
    ValueClass NVARCHAR(20), -- 'High', 'Medium', 'Low'
    FragilityIndex DECIMAL(3,2), -- 0.0 to 1.0
    OptimalStorageZone NVARCHAR(50),
    RecommendedProximityToDock INT -- meters
);

-- Calculate gravity score
UPDATE DimProductGravity
SET GravityScore = (
    CASE VelocityClass
        WHEN 'Fast' THEN 3
        WHEN 'Medium' THEN 2
        ELSE 1
    END *
    CASE ValueClass
        WHEN 'High' THEN 3
        WHEN 'Medium' THEN 2
        ELSE 1
    END *
    (1.0 / NULLIF(FragilityIndex, 0))
);
```

**DimGeography** - Hierarchical location dimension:

```sql
CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationID NVARCHAR(50) UNIQUE,
    LocationName NVARCHAR(200),
    LocationType NVARCHAR(50), -- 'Warehouse', 'Distribution Center', 'Route Node'
    Continent NVARCHAR(50),
    Country NVARCHAR(100),
    Region NVARCHAR(100),
    City NVARCHAR(100),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    CapacityM3 DECIMAL(12,2),
    CurrentUtilizationPercent DECIMAL(5,2)
);
```

## Stored Procedures for Data Loading

### Incremental Warehouse Operations Load

```sql
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadTimestamp DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Default to last successful load if not provided
    IF @LastLoadTimestamp IS NULL
        SELECT @LastLoadTimestamp = MAX(LoadTimestamp) FROM ETL_LoadLog WHERE TableName = 'FactWarehouseOperations';
    
    -- Insert new records from staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductKey, OperationType, 
        Quantity, DwellTimeMinutes, PickRateUnitsPerHour, StorageZoneKey
    )
    SELECT 
        CONVERT(INT, FORMAT(s.OperationTimestamp, 'yyyyMMddHHmm')) AS TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.OperationType,
        s.Quantity,
        DATEDIFF(MINUTE, s.StartTime, s.EndTime) AS DwellTimeMinutes,
        CASE WHEN s.OperationType = 'Picking' 
             THEN s.Quantity / NULLIF(DATEDIFF(MINUTE, s.StartTime, s.EndTime) / 60.0, 0)
             ELSE NULL 
        END AS PickRateUnitsPerHour,
        sz.StorageZoneKey
    FROM Stage_WarehouseOperations s
    INNER JOIN DimGeography g ON s.WarehouseID = g.LocationID
    INNER JOIN DimProductGravity p ON s.SKU = p.SKU
    LEFT JOIN DimStorageZone sz ON s.ZoneCode = sz.ZoneCode
    WHERE s.OperationTimestamp > @LastLoadTimestamp;
    
    -- Log successful load
    INSERT INTO ETL_LoadLog (TableName, LoadTimestamp, RowsInserted)
    VALUES ('FactWarehouseOperations', GETDATE(), @@ROWCOUNT);
END;
```

### Fleet Telemetry Data Load

```sql
CREATE PROCEDURE sp_LoadFleetTrips
    @LastLoadTimestamp DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    IF @LastLoadTimestamp IS NULL
        SELECT @LastLoadTimestamp = MAX(LoadTimestamp) FROM ETL_LoadLog WHERE TableName = 'FactFleetTrips';
    
    INSERT INTO FactFleetTrips (
        TimeKey, VehicleKey, RouteKey, DriverKey,
        OriginGeographyKey, DestinationGeographyKey,
        DistanceKM, TripDurationMinutes, IdleTimeMinutes,
        FuelConsumedLiters, LoadWeightKG, OnTimeDeliveryFlag
    )
    SELECT 
        CONVERT(INT, FORMAT(t.TripStartTime, 'yyyyMMddHHmm')) AS TimeKey,
        v.VehicleKey,
        r.RouteKey,
        d.DriverKey,
        g1.GeographyKey AS OriginGeographyKey,
        g2.GeographyKey AS DestinationGeographyKey,
        t.DistanceKM,
        DATEDIFF(MINUTE, t.TripStartTime, t.TripEndTime) AS TripDurationMinutes,
        t.IdleTimeMinutes,
        t.FuelConsumedLiters,
        t.LoadWeightKG,
        CASE WHEN t.ActualArrivalTime <= t.ScheduledArrivalTime THEN 1 ELSE 0 END AS OnTimeDeliveryFlag
    FROM Stage_FleetTelemetry t
    INNER JOIN DimVehicle v ON t.VehicleID = v.VehicleID
    INNER JOIN DimRoute r ON t.RouteID = r.RouteID
    INNER JOIN DimDriver d ON t.DriverID = d.DriverID
    INNER JOIN DimGeography g1 ON t.OriginLocationID = g1.LocationID
    INNER JOIN DimGeography g2 ON t.DestinationLocationID = g2.LocationID
    WHERE t.TripStartTime > @LastLoadTimestamp;
    
    INSERT INTO ETL_LoadLog (TableName, LoadTimestamp, RowsInserted)
    VALUES ('FactFleetTrips', GETDATE(), @@ROWCOUNT);
END;
```

## Key Analytical Queries

### Cross-Fact KPI: Dwell Time vs Fleet Idling Cost

```sql
-- Find correlation between warehouse dwell time and fleet idling
WITH WarehouseDwell AS (
    SELECT 
        t.FullDate,
        g.Region,
        AVG(w.DwellTimeMinutes) AS AvgDwellTime,
        SUM(w.Quantity) AS TotalVolume
    FROM FactWarehouseOperations w
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    INNER JOIN DimGeography g ON w.WarehouseKey = g.GeographyKey
    WHERE t.FullDate >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY t.FullDate, g.Region
),
FleetIdling AS (
    SELECT 
        t.FullDate,
        g2.Region,
        AVG(f.IdleTimeMinutes) AS AvgIdleTime,
        AVG(f.FuelConsumedLiters * 1.50) AS AvgFuelCost -- $1.50/liter
    FROM FactFleetTrips f
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    INNER JOIN DimGeography g2 ON f.DestinationGeographyKey = g2.GeographyKey
    WHERE t.FullDate >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY t.FullDate, g2.Region
)
SELECT 
    wd.FullDate,
    wd.Region,
    wd.AvgDwellTime,
    fi.AvgIdleTime,
    fi.AvgFuelCost,
    -- Correlation indicator
    CASE 
        WHEN wd.AvgDwellTime > 120 AND fi.AvgIdleTime > 30 THEN 'High Friction'
        WHEN wd.AvgDwellTime > 60 OR fi.AvgIdleTime > 15 THEN 'Moderate Friction'
        ELSE 'Normal Operations'
    END AS FrictionLevel
FROM WarehouseDwell wd
INNER JOIN FleetIdling fi ON wd.FullDate = fi.FullDate AND wd.Region = fi.Region
ORDER BY wd.FullDate DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify products that should be reassigned to different storage zones
SELECT 
    p.SKU,
    p.ProductName,
    p.CurrentStorageZone,
    p.GravityScore,
    sz.ZoneName AS CurrentZone,
    sz.ProximityToDockMeters AS CurrentProximity,
    rec_sz.ZoneName AS RecommendedZone,
    rec_sz.ProximityToDockMeters AS RecommendedProximity,
    AVG(w.DwellTimeMinutes) AS AvgDwellTime,
    COUNT(*) AS PickEvents,
    SUM(w.Quantity) AS TotalVolumeMovedLast30Days
FROM DimProductGravity p
LEFT JOIN DimStorageZone sz ON p.CurrentStorageZone = sz.ZoneCode
LEFT JOIN DimStorageZone rec_sz ON p.OptimalStorageZone = rec_sz.ZoneCode
INNER JOIN FactWarehouseOperations w ON p.ProductKey = w.ProductKey
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
WHERE 
    t.FullDate >= DATEADD(DAY, -30, GETDATE())
    AND w.OperationType = 'Picking'
    AND p.CurrentStorageZone <> p.OptimalStorageZone
GROUP BY 
    p.SKU, p.ProductName, p.CurrentStorageZone, p.GravityScore,
    sz.ZoneName, sz.ProximityToDockMeters,
    rec_sz.ZoneName, rec_sz.ProximityToDockMeters
HAVING COUNT(*) > 50 -- Only high-frequency items worth moving
ORDER BY p.GravityScore DESC, PickEvents DESC;
```

### Predictive Bottleneck Detection

```sql
-- Detect emerging bottlenecks before they become critical
WITH RecentTrends AS (
    SELECT 
        w.WarehouseKey,
        w.StorageZoneKey,
        t.Hour,
        AVG(w.DwellTimeMinutes) AS AvgDwellThisWeek,
        LAG(AVG(w.DwellTimeMinutes), 1) OVER (
            PARTITION BY w.WarehouseKey, w.StorageZoneKey, t.Hour 
            ORDER BY t.Year, t.Month, t.Day
        ) AS AvgDwellPrevWeek
    FROM FactWarehouseOperations w
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.FullDate >= DATEADD(DAY, -14, GETDATE())
    GROUP BY w.WarehouseKey, w.StorageZoneKey, t.Hour, t.Year, t.Month, t.Day
)
SELECT 
    g.LocationName AS Warehouse,
    sz.ZoneName AS Zone,
    rt.Hour,
    rt.AvgDwellThisWeek,
    rt.AvgDwellPrevWeek,
    ((rt.AvgDwellThisWeek - rt.AvgDwellPrevWeek) / NULLIF(rt.AvgDwellPrevWeek, 0)) * 100 AS PercentChange,
    CASE 
        WHEN rt.AvgDwellThisWeek > rt.AvgDwellPrevWeek * 1.5 THEN 'CRITICAL'
        WHEN rt.AvgDwellThisWeek > rt.AvgDwellPrevWeek * 1.2 THEN 'WARNING'
        ELSE 'NORMAL'
    END AS AlertLevel
FROM RecentTrends rt
INNER JOIN DimGeography g ON rt.WarehouseKey = g.GeographyKey
INNER JOIN DimStorageZone sz ON rt.StorageZoneKey = sz.StorageZoneKey
WHERE rt.AvgDwellThisWeek > rt.AvgDwellPrevWeek * 1.2
ORDER BY PercentChange DESC;
```

## Power BI Dashboard Configuration

### Connect to SQL Server

In Power BI Desktop, configure the SQL Server connection:

1. Get Data → SQL Server
2. Server: `localhost` (or your SQL Server instance)
3. Database: `LogiFleetPulse`
4. Data Connectivity mode: **DirectQuery** (for real-time) or **Import** (for performance)
5. Advanced options → SQL statement (optional custom query)

### Key DAX Measures

**Fleet Efficiency Score**:

```dax
Fleet Efficiency = 
VAR TotalDistance = SUM(FactFleetTrips[DistanceKM])
VAR TotalIdleTime = SUM(FactFleetTrips[IdleTimeMinutes])
VAR TotalTripTime = SUM(FactFleetTrips[TripDurationMinutes])
VAR IdlePercentage = DIVIDE(TotalIdleTime, TotalTripTime, 0)
RETURN
    (1 - IdlePercentage) * 100
```

**Warehouse Throughput Rate**:

```dax
Throughput Rate = 
VAR TotalUnits = SUM(FactWarehouseOperations[Quantity])
VAR TotalHours = 
    SUMX(
        FactWarehouseOperations,
        FactWarehouseOperations[DwellTimeMinutes] / 60
    )
RETURN
    DIVIDE(TotalUnits, TotalHours, 0)
```

**Cross-Dock Velocity**:

```dax
Cross-Dock Velocity = 
AVERAGE(FactCrossDock[TransferDurationMinutes])
```

**Gravity Zone Compliance**:

```dax
Gravity Zone Compliance % = 
VAR TotalProducts = COUNTROWS(DimProductGravity)
VAR CompliantProducts = 
    CALCULATE(
        COUNTROWS(DimProductGravity),
        DimProductGravity[CurrentStorageZone] = DimProductGravity[OptimalStorageZone]
    )
RETURN
    DIVIDE(CompliantProducts, TotalProducts, 0) * 100
```

### Time Intelligence for Trend Analysis

```dax
Dwell Time MoM Growth = 
VAR CurrentMonth = CALCULATE(AVERAGE(FactWarehouseOperations[DwellTimeMinutes]))
VAR PreviousMonth = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        DATEADD(DimTime[FullDate], -1, MONTH)
    )
RETURN
    DIVIDE(CurrentMonth - PreviousMonth, PreviousMonth, 0) * 100
```

## Automated Alerting Setup

### Create Alert Trigger

```sql
CREATE PROCEDURE sp_CheckKPIThresholds
AS
BEGIN
    DECLARE @AlertThresholds TABLE (
        KPI NVARCHAR(100),
        ThresholdValue DECIMAL(10,2),
        CurrentValue DECIMAL(10,2),
        AlertLevel NVARCHAR(20)
    );
    
    -- Check Fleet Idle Time
    INSERT INTO @AlertThresholds
    SELECT 
        'Fleet Idle Time %' AS KPI,
        15.0 AS ThresholdValue,
        AVG(CAST(IdleTimeMinutes AS DECIMAL) / TripDurationMinutes * 100) AS CurrentValue,
        CASE 
            WHEN AVG(CAST(IdleTimeMinutes AS DECIMAL) / TripDurationMinutes * 100) > 20 THEN 'CRITICAL'
            WHEN AVG(CAST(IdleTimeMinutes AS DECIMAL) / TripDurationMinutes * 100) > 15 THEN 'WARNING'
            ELSE 'NORMAL'
        END AS AlertLevel
    FROM FactFleetTrips f
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.FullDate >= DATEADD(HOUR, -24, GETDATE());
    
    -- Check Warehouse Dwell Time
    INSERT INTO @AlertThresholds
    SELECT 
        'Warehouse Avg Dwell (minutes)' AS KPI,
        90.0 AS ThresholdValue,
        AVG(CAST(DwellTimeMinutes AS DECIMAL)) AS CurrentValue,
        CASE 
            WHEN AVG(CAST(DwellTimeMinutes AS DECIMAL)) > 120 THEN 'CRITICAL'
            WHEN AVG(CAST(DwellTimeMinutes AS DECIMAL)) > 90 THEN 'WARNING'
            ELSE 'NORMAL'
        END AS AlertLevel
    FROM FactWarehouseOperations w
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.FullDate >= DATEADD(HOUR, -24, GETDATE());
    
    -- Send alerts for breached thresholds
    INSERT INTO AlertLog (KPI, CurrentValue, ThresholdValue, AlertLevel, AlertTimestamp)
    SELECT KPI, CurrentValue, ThresholdValue, AlertLevel, GETDATE()
    FROM @AlertThresholds
    WHERE CurrentValue > ThresholdValue;
    
    -- Return breached KPIs
    SELECT * FROM @AlertThresholds WHERE CurrentValue > ThresholdValue;
END;
```

### Schedule Alert Job (SQL Server Agent)

```sql
-- Create SQL Server Agent job to run every 15 minutes
USE msdb;
GO

EXEC sp_add_job 
    @job_name = N'LogiFleet_KPI_Alert_Monitor',
    @enabled = 1;

EXEC sp_add_jobstep
    @job_name = N'LogiFleet_KPI_Alert_Monitor',
    @step_name = N'Check Thresholds',
    @subsystem = N'TSQL',
    @command = N'EXEC sp_CheckKPIThresholds;',
    @database_name = N'LogiFleetPulse';

EXEC sp_add_schedule
    @schedule_name = N'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;

EXEC sp_attach_schedule
    @job_name = N'LogiFleet_KPI_Alert_Monitor',
    @schedule_name = N'Every15Minutes';

EXEC sp_add_jobserver
    @job_name = N'LogiFleet_KPI_Alert_Monitor';
```

## Common Patterns & Best Practices

### Pattern 1: Incremental Data Loading

Always use watermark-based incremental loading to avoid full table scans:

```sql
-- Track last successful load
CREATE TABLE ETL_LoadLog (
    TableName NVARCHAR(100),
    LoadTimestamp DATETIME,
    RowsInserted INT,
    LoadStatus NVARCHAR(50) DEFAULT 'SUCCESS'
);

-- Use in load procedures
DECLARE @LastLoad DATETIME;
SELECT @LastLoad = MAX(LoadTimestamp) 
FROM ETL_LoadLog 
WHERE TableName = 'FactWarehouseOperations' AND LoadStatus = 'SUCCESS';
```

### Pattern 2: Role-Based Security

Implement row-level security for multi-tenant scenarios:

```sql
CREATE FUNCTION fn_SecurityPredicate(@Region NVARCHAR(100))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN 
    SELECT 1 AS AccessGranted
    WHERE @Region IN (
        SELECT Region FROM UserRegionAccess 
        WHERE Username = USER_NAME()
    );

CREATE SECURITY POLICY RegionSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(Region)
ON dbo.DimGeography
WITH (STATE = ON);
```

### Pattern 3: Handling Many-to-Many Relationships

Use bridge tables for complex relationships (e.g., routes servicing multiple warehouses):

```sql
CREATE TABLE Bridge_Route_Warehouse (
    RouteKey INT,
    WarehouseKey INT,
    ServiceOrder INT,
    AvgStopDurationMinutes INT,
    CONSTRAINT PK_Bridge_Route_WH PRIMARY KEY (RouteKey, WarehouseKey)
);

-- Query through bridge
SELECT 
    r.RouteName,
    g.LocationName AS Warehouse,
    brw.ServiceOrder,
    SUM(f.Quantity) AS TotalDelivered
FROM FactWarehouseOperations f
INNER JOIN Bridge_Route_Warehouse brw ON f.WarehouseKey = brw.WarehouseKey
INNER JOIN DimRoute r ON brw.RouteKey = r.RouteKey
INNER JOIN DimGeography g ON f.WarehouseKey = g.GeographyKey
GROUP BY r.RouteName, g.LocationName, brw.ServiceOrder;
```

## Troubleshooting

### Issue: Power BI Dashboard Shows No Data

**Symptoms**: Connected successfully but visuals are empty

**Solutions**:
1. Verify data has been loaded into fact tables:
   ```sql
   SELECT COUNT(*) FROM FactWarehouseOperations;
   SELECT COUNT(*) FROM FactFleetTrips;
   ```

2. Check time dimension includes current dates:
   ```sql
   SELECT MIN(FullDate), MAX(FullDate) FROM DimTime;
   -- If missing, run: EXEC sp_PopulateTimeDimension @StartDate = '2025-01-01', @EndDate = '2027-12-31';
   ```

3. Ensure relationships are active in Power BI model (Model View → check relationship arrows)

### Issue: Slow Query Performance

**Symptoms**: Dashboards take >30 seconds to load
