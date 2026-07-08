---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehouse with multi-fact star schema for real-time fleet, warehouse, and supply chain analytics
triggers:
  - set up logistics analytics dashboard with power bi
  - create supply chain data warehouse schema
  - implement fleet and warehouse kpi tracking
  - build multi-fact star schema for logistics
  - deploy logifleet pulse analytics platform
  - configure real-time supply chain intelligence
  - integrate warehouse and fleet data in sql server
  - analyze cross-modal logistics with power bi
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What It Does

LogiFleet Pulse is an advanced MS SQL Server data warehouse and Power BI analytics platform for logistics and supply chain management. It provides:

- **Multi-fact star schema** linking warehouse operations, fleet telemetry, and supplier data
- **Real-time dashboards** with 15-minute refresh granularity
- **Cross-fact KPI harmonization** (e.g., inventory dwell time vs. fleet idle costs)
- **Predictive analytics** for bottleneck detection and maintenance triage
- **Warehouse gravity zone optimization** based on pick frequency and item characteristics
- **Role-based access control** with row-level security

The platform ingests data from WMS, TMS, telematics systems, and external APIs to create a unified semantic layer for logistics intelligence.

## Installation

### Prerequisites

- MS SQL Server 2019 or later
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio

### Step 1: Clone the Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### Step 2: Deploy SQL Schema

```bash
# Connect to your SQL Server instance
sqlcmd -S YOUR_SERVER_NAME -d YOUR_DATABASE_NAME -E -i schema/01_create_database.sql
sqlcmd -S YOUR_SERVER_NAME -d YOUR_DATABASE_NAME -E -i schema/02_create_dimensions.sql
sqlcmd -S YOUR_SERVER_NAME -d YOUR_DATABASE_NAME -E -i schema/03_create_facts.sql
sqlcmd -S YOUR_SERVER_NAME -d YOUR_DATABASE_NAME -E -i schema/04_create_views.sql
sqlcmd -S YOUR_SERVER_NAME -d YOUR_DATABASE_NAME -E -i schema/05_create_stored_procedures.sql
```

### Step 3: Configure Data Sources

```bash
# Copy and edit configuration template
cp config_sample.json config.json
# Edit config.json with your connection strings
```

Configuration structure:

```json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "${SQL_DATABASE_NAME}",
    "authentication": "Windows|SQLAuth",
    "username": "${SQL_USERNAME}",
    "password": "${SQL_PASSWORD}"
  },
  "data_sources": {
    "wms_api": "${WMS_API_ENDPOINT}",
    "telematics_api": "${TELEMATICS_API_ENDPOINT}",
    "erp_connection": "${ERP_CONNECTION_STRING}"
  },
  "refresh_interval_minutes": 15,
  "alert_thresholds": {
    "fleet_idle_percentage": 15,
    "dwell_time_hours": 72,
    "temperature_variance_celsius": 2
  }
}
```

## Core Schema Architecture

### Key Fact Tables

**FactWarehouseOperations** - Warehouse activity metrics

```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    DwellTimeMinutes INT,
    PickRate DECIMAL(10,2),
    PackingTimeMinutes INT,
    QuantityHandled INT,
    GravityZoneID INT,
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (WarehouseKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
);

CREATE NONCLUSTERED INDEX IX_WarehouseOps_Time ON FactWarehouseOperations(TimeKey);
CREATE NONCLUSTERED INDEX IX_WarehouseOps_Product ON FactWarehouseOperations(ProductKey);
```

**FactFleetTrips** - Fleet telemetry and route data

```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    DriverKey INT NOT NULL,
    RouteKey INT NOT NULL,
    OriginGeographyKey INT,
    DestinationGeographyKey INT,
    TripDurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(10,2),
    DistanceKilometers DECIMAL(10,2),
    LoadWeightKg DECIMAL(10,2),
    OnTimeDelivery BIT,
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
);

CREATE NONCLUSTERED INDEX IX_FleetTrips_Time ON FactFleetTrips(TimeKey);
CREATE NONCLUSTERED INDEX IX_FleetTrips_Route ON FactFleetTrips(RouteKey);
```

### Key Dimension Tables

**DimTime** - 15-minute granularity time dimension

```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTimeValue DATETIME NOT NULL,
    DateKey INT,
    TimeOfDay TIME,
    HourOfDay INT,
    DayOfWeek INT,
    DayName VARCHAR(20),
    IsWeekend BIT,
    FiscalQuarter INT,
    FiscalYear INT
);

-- Generate time dimension data
CREATE PROCEDURE spPopulateTimeDimension
    @StartDate DATETIME,
    @EndDate DATETIME
AS
BEGIN
    DECLARE @CurrentDate DATETIME = @StartDate;
    WHILE @CurrentDate <= @EndDate
    BEGIN
        INSERT INTO DimTime (TimeKey, DateTimeValue, DateKey, TimeOfDay, HourOfDay, DayOfWeek, DayName, IsWeekend)
        VALUES (
            CONVERT(INT, FORMAT(@CurrentDate, 'yyyyMMddHHmm')),
            @CurrentDate,
            CONVERT(INT, FORMAT(@CurrentDate, 'yyyyMMdd')),
            CONVERT(TIME, @CurrentDate),
            DATEPART(HOUR, @CurrentDate),
            DATEPART(WEEKDAY, @CurrentDate),
            DATENAME(WEEKDAY, @CurrentDate),
            CASE WHEN DATEPART(WEEKDAY, @CurrentDate) IN (1, 7) THEN 1 ELSE 0 END
        );
        SET @CurrentDate = DATEADD(MINUTE, 15, @CurrentDate);
    END
END;
```

**DimProductGravity** - Product hierarchy with gravity scoring

```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    GravityScore DECIMAL(5,2), -- Calculated: velocity × value × fragility
    PickFrequency INT,
    UnitValue DECIMAL(10,2),
    FragilityIndex DECIMAL(3,2),
    RecommendedZone INT,
    IsCurrent BIT DEFAULT 1
);

-- Calculate gravity score
CREATE PROCEDURE spCalculateProductGravity
AS
BEGIN
    UPDATE DimProductGravity
    SET GravityScore = (
        (PickFrequency / NULLIF((SELECT MAX(PickFrequency) FROM DimProductGravity), 0)) * 0.5 +
        (UnitValue / NULLIF((SELECT MAX(UnitValue) FROM DimProductGravity), 0)) * 0.3 +
        (FragilityIndex) * 0.2
    ) * 100;
END;
```

## Common Analytical Queries

### Cross-Fact KPI: Dwell Time vs Fleet Idle Cost

```sql
-- Find SKUs with high dwell time that correlate with fleet idle time
WITH WarehouseDwell AS (
    SELECT 
        p.SKU,
        p.Category,
        AVG(w.DwellTimeMinutes) AS AvgDwellTime,
        w.WarehouseKey
    FROM FactWarehouseOperations w
    JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.DateTimeValue >= DATEADD(DAY, -30, GETDATE())
    GROUP BY p.SKU, p.Category, w.WarehouseKey
    HAVING AVG(w.DwellTimeMinutes) > 4320 -- 72 hours
),
FleetIdle AS (
    SELECT 
        f.OriginGeographyKey,
        AVG(f.IdleTimeMinutes) AS AvgIdleTime,
        AVG(f.FuelConsumedLiters * 1.5) AS AvgIdleCostUSD -- Assuming $1.50/liter
    FROM FactFleetTrips f
    JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.DateTimeValue >= DATEADD(DAY, -30, GETDATE())
    GROUP BY f.OriginGeographyKey
)
SELECT 
    wd.SKU,
    wd.Category,
    wd.AvgDwellTime,
    fi.AvgIdleTime,
    fi.AvgIdleCostUSD,
    (wd.AvgDwellTime / 60.0) * (fi.AvgIdleCostUSD / fi.AvgIdleTime) AS EstimatedCostImpact
FROM WarehouseDwell wd
JOIN FleetIdle fi ON wd.WarehouseKey = fi.OriginGeographyKey
ORDER BY EstimatedCostImpact DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify products in wrong gravity zones
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.RecommendedZone AS CurrentZone,
    CASE 
        WHEN p.GravityScore > 75 THEN 1
        WHEN p.GravityScore > 50 THEN 2
        WHEN p.GravityScore > 25 THEN 3
        ELSE 4
    END AS OptimalZone,
    AVG(w.DwellTimeMinutes) AS AvgDwellTime,
    COUNT(*) AS OperationCount
FROM DimProductGravity p
JOIN FactWarehouseOperations w ON p.ProductKey = w.ProductKey
JOIN DimTime t ON w.TimeKey = t.TimeKey
WHERE t.DateTimeValue >= DATEADD(DAY, -7, GETDATE())
    AND p.IsCurrent = 1
GROUP BY p.SKU, p.ProductName, p.GravityScore, p.RecommendedZone
HAVING p.RecommendedZone <> CASE 
    WHEN p.GravityScore > 75 THEN 1
    WHEN p.GravityScore > 50 THEN 2
    WHEN p.GravityScore > 25 THEN 3
    ELSE 4
END
ORDER BY OperationCount DESC;
```

### Predictive Bottleneck Detection

```sql
-- Identify probable congestion points based on historical patterns
CREATE VIEW vwBottleneckPrediction AS
SELECT 
    g.WarehouseName,
    t.HourOfDay,
    t.DayOfWeek,
    COUNT(*) AS OperationVolume,
    AVG(w.DwellTimeMinutes) AS AvgDwellTime,
    STDEV(w.DwellTimeMinutes) AS DwellTimeVariance,
    CASE 
        WHEN AVG(w.DwellTimeMinutes) > 120 AND STDEV(w.DwellTimeMinutes) > 60 THEN 'HIGH'
        WHEN AVG(w.DwellTimeMinutes) > 60 AND STDEV(w.DwellTimeMinutes) > 30 THEN 'MEDIUM'
        ELSE 'LOW'
    END AS BottleneckRisk
FROM FactWarehouseOperations w
JOIN DimTime t ON w.TimeKey = t.TimeKey
JOIN DimGeography g ON w.WarehouseKey = g.GeographyKey
WHERE t.DateTimeValue >= DATEADD(DAY, -90, GETDATE())
GROUP BY g.WarehouseName, t.HourOfDay, t.DayOfWeek;
```

## Data Loading Patterns

### Incremental ETL Stored Procedure

```sql
CREATE PROCEDURE spLoadWarehouseOperationsIncremental
    @LastLoadTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Load from staging table (populated by external ETL)
    INSERT INTO FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductKey, OperationType, 
        DwellTimeMinutes, PickRate, PackingTimeMinutes, 
        QuantityHandled, GravityZoneID
    )
    SELECT 
        CONVERT(INT, FORMAT(s.OperationTimestamp, 'yyyyMMddHHmm')) AS TimeKey,
        g.GeographyKey AS WarehouseKey,
        p.ProductKey,
        s.OperationType,
        DATEDIFF(MINUTE, s.StartTime, s.EndTime) AS DwellTimeMinutes,
        s.PickRate,
        DATEDIFF(MINUTE, s.PackStartTime, s.PackEndTime) AS PackingTimeMinutes,
        s.Quantity,
        p.RecommendedZone
    FROM StagingWarehouseOperations s
    JOIN DimGeography g ON s.WarehouseCode = g.WarehouseCode
    JOIN DimProductGravity p ON s.SKU = p.SKU
    WHERE s.LoadedTimestamp > @LastLoadTime
        AND s.IsProcessed = 0;
    
    -- Mark staging records as processed
    UPDATE StagingWarehouseOperations
    SET IsProcessed = 1
    WHERE LoadedTimestamp > @LastLoadTime;
    
    -- Update last load timestamp
    UPDATE ETLControl
    SET LastLoadTime = GETDATE()
    WHERE TableName = 'FactWarehouseOperations';
END;
```

### Fleet Telemetry Integration

```sql
-- Load fleet trips from external API (via staging)
CREATE PROCEDURE spLoadFleetTripsFromAPI
AS
BEGIN
    -- Assumes external process populates StagingFleetTelemetry
    INSERT INTO FactFleetTrips (
        TimeKey, VehicleKey, DriverKey, RouteKey,
        OriginGeographyKey, DestinationGeographyKey,
        TripDurationMinutes, IdleTimeMinutes, FuelConsumedLiters,
        DistanceKilometers, LoadWeightKg, OnTimeDelivery
    )
    SELECT 
        CONVERT(INT, FORMAT(s.TripStartTime, 'yyyyMMddHHmm')),
        v.VehicleKey,
        d.DriverKey,
        r.RouteKey,
        og.GeographyKey,
        dg.GeographyKey,
        DATEDIFF(MINUTE, s.TripStartTime, s.TripEndTime),
        s.IdleMinutes,
        s.FuelUsed,
        s.DistanceTraveled,
        s.LoadWeight,
        CASE WHEN s.ActualArrival <= s.ScheduledArrival THEN 1 ELSE 0 END
    FROM StagingFleetTelemetry s
    JOIN DimVehicle v ON s.VehicleID = v.VehicleID
    JOIN DimDriver d ON s.DriverID = d.DriverID
    JOIN DimRoute r ON s.RouteID = r.RouteID
    JOIN DimGeography og ON s.OriginCode = og.LocationCode
    JOIN DimGeography dg ON s.DestinationCode = dg.LocationCode
    WHERE s.IsProcessed = 0;
    
    UPDATE StagingFleetTelemetry
    SET IsProcessed = 1
    WHERE IsProcessed = 0;
END;
```

## Power BI Integration

### Connect Power BI to SQL Server

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. Enter connection parameters when prompted:
   - Server: `${SQL_SERVER_HOST}`
   - Database: `${SQL_DATABASE_NAME}`
3. Choose authentication method (Windows or SQL Server)

### Key DAX Measures

**Fleet Idle Percentage**

```dax
Fleet Idle % = 
DIVIDE(
    SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[TripDurationMinutes]),
    0
) * 100
```

**Warehouse Gravity Score Trend**

```dax
Avg Gravity Score = 
CALCULATE(
    AVERAGE(DimProductGravity[GravityScore]),
    DimProductGravity[IsCurrent] = TRUE
)
```

**Cross-Fact Efficiency Index**

```dax
Efficiency Index = 
VAR WarehouseEff = DIVIDE(SUM(FactWarehouseOperations[QuantityHandled]), SUM(FactWarehouseOperations[DwellTimeMinutes]))
VAR FleetEff = DIVIDE(SUM(FactFleetTrips[DistanceKilometers]), SUM(FactFleetTrips[FuelConsumedLiters]))
RETURN
    (WarehouseEff * 0.6) + (FleetEff * 0.4)
```

### Configure Automatic Refresh

```json
// Dataset settings in Power BI Service
{
  "refreshSchedule": {
    "frequency": "every15Minutes",
    "enabled": true
  },
  "directQuery": false,
  "incrementalRefresh": {
    "enabled": true,
    "rollingWindow": {
      "count": 3,
      "granularity": "Month"
    }
  }
}
```

## Alerting System

### Create Alert Stored Procedure

```sql
CREATE PROCEDURE spCheckAlertThresholds
AS
BEGIN
    -- Fleet idle time alerts
    INSERT INTO AlertQueue (AlertType, Severity, Message, CreatedAt)
    SELECT 
        'Fleet Idle',
        'HIGH',
        'Vehicle ' + v.VehicleID + ' exceeded idle threshold: ' + 
        CAST(AVG(f.IdleTimeMinutes) * 100.0 / AVG(f.TripDurationMinutes) AS VARCHAR) + '%',
        GETDATE()
    FROM FactFleetTrips f
    JOIN DimVehicle v ON f.VehicleKey = v.VehicleKey
    JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.DateTimeValue >= DATEADD(HOUR, -4, GETDATE())
    GROUP BY v.VehicleID
    HAVING AVG(f.IdleTimeMinutes) * 100.0 / NULLIF(AVG(f.TripDurationMinutes), 0) > 15;
    
    -- Warehouse dwell time alerts
    INSERT INTO AlertQueue (AlertType, Severity, Message, CreatedAt)
    SELECT 
        'Warehouse Dwell',
        'MEDIUM',
        'SKU ' + p.SKU + ' in ' + g.WarehouseName + ' exceeded 72-hour dwell: ' +
        CAST(AVG(w.DwellTimeMinutes) / 60.0 AS VARCHAR) + ' hours',
        GETDATE()
    FROM FactWarehouseOperations w
    JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    JOIN DimGeography g ON w.WarehouseKey = g.GeographyKey
    JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.DateTimeValue >= DATEADD(DAY, -1, GETDATE())
    GROUP BY p.SKU, g.WarehouseName
    HAVING AVG(w.DwellTimeMinutes) > 4320;
END;

-- Schedule with SQL Server Agent
-- Job runs every 15 minutes
```

## Row-Level Security Configuration

```sql
-- Create security roles
CREATE ROLE WarehouseManager;
CREATE ROLE FleetSupervisor;
CREATE ROLE Executive;

-- Create security function
CREATE FUNCTION fnSecurityPredicate(@GeographyKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN (
    SELECT 1 AS AccessGranted
    FROM dbo.UserGeographyAccess
    WHERE UserName = USER_NAME()
        AND GeographyKey = @GeographyKey
);

-- Apply security policy
CREATE SECURITY POLICY WarehouseSecurityPolicy
ADD FILTER PREDICATE dbo.fnSecurityPredicate(WarehouseKey)
ON dbo.FactWarehouseOperations,
ADD FILTER PREDICATE dbo.fnSecurityPredicate(OriginGeographyKey)
ON dbo.FactFleetTrips
WITH (STATE = ON);
```

## Troubleshooting

### Performance Issues

**Problem:** Slow cross-fact queries
**Solution:** Ensure columnstore indexes on fact tables

```sql
-- Add columnstore index for analytical queries
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_WarehouseOps_Columnstore
ON FactWarehouseOperations (
    TimeKey, WarehouseKey, ProductKey, DwellTimeMinutes, PickRate
);

CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FleetTrips_Columnstore
ON FactFleetTrips (
    TimeKey, VehicleKey, RouteKey, IdleTimeMinutes, FuelConsumedLiters
);
```

**Problem:** Power BI refresh timeouts
**Solution:** Implement incremental refresh with partitioning

```sql
-- Partition fact table by date
ALTER PARTITION FUNCTION pfDateRange()
SPLIT RANGE ('2026-08-01');

-- Move old data to archive
INSERT INTO FactWarehouseOperations_Archive
SELECT * FROM FactWarehouseOperations
WHERE TimeKey < 202607010000;

DELETE FROM FactWarehouseOperations
WHERE TimeKey < 202607010000;
```

### Data Quality Issues

**Problem:** Missing time dimension keys
**Solution:** Add default handling in ETL

```sql
-- Handle missing time keys
MERGE INTO FactWarehouseOperations AS target
USING StagingWarehouseOperations AS source
ON 1=0
WHEN NOT MATCHED THEN
    INSERT (TimeKey, WarehouseKey, ProductKey, ...)
    VALUES (
        COALESCE(
            (SELECT TimeKey FROM DimTime WHERE DateTimeValue = source.OperationTimestamp),
            999999 -- Unknown time key
        ),
        ...
    );
```

**Problem:** Gravity scores not updating
**Solution:** Schedule gravity recalculation

```sql
-- SQL Server Agent job: Daily at 2 AM
EXEC spCalculateProductGravity;

-- Verify calculation
SELECT 
    SKU, 
    GravityScore, 
    PickFrequency, 
    UnitValue, 
    FragilityIndex
FROM DimProductGravity
WHERE GravityScore IS NULL OR GravityScore = 0;
```

### Connection Issues

**Problem:** Power BI can't connect to SQL Server
**Solution:** Verify firewall and gateway settings

```powershell
# Test SQL Server connectivity
Test-NetConnection -ComputerName YOUR_SERVER -Port 1433

# Check SQL Server authentication
sqlcmd -S YOUR_SERVER -U YOUR_USERNAME -P $env:SQL_PASSWORD -Q "SELECT @@VERSION"
```

**Problem:** External API data not loading
**Solution:** Verify external table configuration

```sql
-- Check external data source
SELECT * FROM sys.external_data_sources;

-- Test external table
SELECT TOP 10 * FROM StagingFleetTelemetry;

-- Recreate if needed
DROP EXTERNAL TABLE StagingFleetTelemetry;
CREATE EXTERNAL TABLE StagingFleetTelemetry (...)
WITH (
    DATA_SOURCE = TelematicsAPI,
    LOCATION = '/api/v1/telemetry',
    FILE_FORMAT = JSONFormat
);
```

## Best Practices

1. **Always use environment variables** for connection strings and API keys
2. **Schedule gravity recalculation** at least daily to keep warehouse zones optimal
3. **Monitor fact table growth** and implement partitioning when tables exceed 50M rows
4. **Use incremental refresh** in Power BI for datasets larger than 1GB
5. **Test alert thresholds** in development before deploying to production
6. **Document custom DAX measures** for team knowledge sharing
7. **Regular index maintenance** on high-transaction fact tables

## Additional Resources

- SQL Server indexing strategies for star schemas
- Power BI incremental refresh documentation
- DAX patterns for time intelligence in logistics
- Best practices for row-level security in multi-tenant BI environments
