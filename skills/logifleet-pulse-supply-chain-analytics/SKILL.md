---
name: logifleet-pulse-supply-chain-analytics
description: Advanced MS SQL Server & Power BI data warehousing and fleet logistics intelligence platform for supply chain analytics
triggers:
  - set up LogiFleet Pulse analytics
  - deploy supply chain analytics dashboard
  - configure Power BI logistics warehouse
  - implement multi-fact star schema for logistics
  - create fleet and warehouse KPI dashboard
  - build real-time supply chain intelligence
  - integrate warehouse and fleet telemetry data
  - design logistics data warehouse architecture
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a multi-modal logistics intelligence engine that combines warehouse operations, fleet telemetry, inventory management, and external data sources into a unified semantic layer. Built on MS SQL Server with Power BI visualization, it provides real-time cross-fact KPI analysis for supply chain decision-making.

**Core capabilities:**
- Multi-fact star schema with time-phased dimensions
- Warehouse gravity zone optimization
- Fleet triage engine with predictive maintenance
- Cross-functional KPI harmonization
- Real-time bottleneck detection
- Temporal elasticity modeling

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- Access to data sources: WMS, TMS, telematics APIs
- Optional: Azure Synapse Analytics for big data enrichment

### Database Deployment

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy SQL schema:**
```sql
-- Connect to your SQL Server instance
-- Run the main schema script
:r .\sql\schema\01_create_database.sql
:r .\sql\schema\02_create_dimensions.sql
:r .\sql\schema\03_create_facts.sql
:r .\sql\schema\04_create_views.sql
:r .\sql\schema\05_create_stored_procedures.sql
```

3. **Configure data sources:**
```json
{
  "dataSources": {
    "wms": {
      "connectionString": "${WMS_CONNECTION_STRING}",
      "refreshInterval": "15m"
    },
    "telematics": {
      "apiEndpoint": "${FLEET_API_ENDPOINT}",
      "apiKey": "${FLEET_API_KEY}",
      "refreshInterval": "5m"
    },
    "weather": {
      "apiEndpoint": "${WEATHER_API_ENDPOINT}",
      "apiKey": "${WEATHER_API_KEY}"
    }
  }
}
```

4. **Import Power BI template:**
- Open `LogiFleet_Pulse_Master.pbit`
- Enter SQL Server connection details
- Configure row-level security roles

## Core Data Model

### Fact Tables

**FactWarehouseOperations** - Warehouse activity tracking
```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    DwellTimeMinutes INT,
    PickRate DECIMAL(10,2),
    PackingTimeMinutes INT,
    GravityZone VARCHAR(10),
    CONSTRAINT FK_WH_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WH_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_WH_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey)
);

-- Optimized indexing for cross-fact queries
CREATE NONCLUSTERED INDEX IX_WH_Time_Product 
ON FactWarehouseOperations(TimeKey, ProductKey) 
INCLUDE (DwellTimeMinutes, GravityZone);
```

**FactFleetTrips** - Fleet telemetry and route data
```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT NOT NULL,
    TripDistanceKm DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    DelayMinutes INT,
    DelayReason VARCHAR(100),
    CONSTRAINT FK_Fleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_Fleet_Route FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey)
);

CREATE NONCLUSTERED INDEX IX_Fleet_Time_Vehicle 
ON FactFleetTrips(TimeKey, VehicleKey) 
INCLUDE (IdleTimeMinutes, DelayMinutes);
```

**FactCrossDock** - Cross-docking operations
```sql
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    TransferTimeMinutes INT,
    TemperatureCompliance BIT,
    CONSTRAINT FK_CD_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_CD_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);
```

### Dimension Tables

**DimTime** - Time-phased dimension (15-minute granularity)
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTimeValue DATETIME NOT NULL,
    Hour INT,
    Minute INT,
    DayOfWeek VARCHAR(10),
    WeekOfYear INT,
    MonthName VARCHAR(10),
    Quarter INT,
    FiscalYear INT,
    IsBusinessHours BIT,
    IsWeekend BIT
);
```

**DimProductGravity** - Product classification with gravity scoring
```sql
CREATE TABLE DimProduct (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    Subcategory VARCHAR(100),
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value / fragility
    VelocityClass VARCHAR(10), -- 'Fast', 'Medium', 'Slow'
    FragilityIndex DECIMAL(3,2),
    UnitValue DECIMAL(10,2),
    OptimalZone VARCHAR(10)
);
```

**DimGeography** - Hierarchical location dimension
```sql
CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationCode VARCHAR(50),
    LocationName VARCHAR(200),
    LocationType VARCHAR(50), -- 'Warehouse', 'Distribution Center', 'Route Node'
    City VARCHAR(100),
    Region VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(10,6),
    Longitude DECIMAL(10,6)
);
```

## Key Stored Procedures

### Incremental Data Loading

```sql
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new warehouse operations
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, WarehouseKey, OperationType,
        DwellTimeMinutes, PickRate, PackingTimeMinutes, GravityZone
    )
    SELECT 
        t.TimeKey,
        p.ProductKey,
        w.WarehouseKey,
        staging.OperationType,
        DATEDIFF(MINUTE, staging.StartTime, staging.EndTime) AS DwellTimeMinutes,
        staging.UnitsProcessed / NULLIF(DATEDIFF(MINUTE, staging.StartTime, staging.EndTime), 0) AS PickRate,
        staging.PackingMinutes,
        p.OptimalZone
    FROM StagingWarehouseOperations staging
    INNER JOIN DimTime t ON staging.OperationDateTime = t.DateTimeValue
    INNER JOIN DimProduct p ON staging.SKU = p.SKU
    INNER JOIN DimWarehouse w ON staging.WarehouseCode = w.WarehouseCode
    WHERE staging.LoadedDateTime > @LastLoadTime;
    
    -- Update gravity zones based on new velocity patterns
    EXEC sp_UpdateGravityZones;
END;
GO
```

### Cross-Fact KPI Calculation

```sql
CREATE PROCEDURE sp_CalculateCrossFactKPIs
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    -- Calculate inventory turnover impact on fleet utilization
    SELECT 
        p.Category,
        p.VelocityClass,
        AVG(wh.DwellTimeMinutes) AS AvgDwellTime,
        AVG(f.IdleTimeMinutes) AS AvgFleetIdleTime,
        CORR(wh.DwellTimeMinutes, f.IdleTimeMinutes) AS Correlation,
        COUNT(DISTINCT f.TripKey) AS TripCount
    FROM FactWarehouseOperations wh
    INNER JOIN DimProduct p ON wh.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON wh.TimeKey = t.TimeKey
    INNER JOIN FactFleetTrips f ON t.TimeKey = f.TimeKey
    WHERE t.DateTimeValue BETWEEN @StartDate AND @EndDate
    GROUP BY p.Category, p.VelocityClass
    ORDER BY Correlation DESC;
END;
GO
```

### Predictive Bottleneck Detection

```sql
CREATE PROCEDURE sp_DetectBottlenecks
    @ThresholdPercentile DECIMAL(3,2) = 0.85
AS
BEGIN
    -- Identify potential bottlenecks using statistical thresholds
    WITH HistoricalBaseline AS (
        SELECT 
            WarehouseKey,
            PERCENTILE_CONT(@ThresholdPercentile) WITHIN GROUP (ORDER BY DwellTimeMinutes) AS Threshold
        FROM FactWarehouseOperations
        WHERE TimeKey >= (SELECT MAX(TimeKey) - 672 FROM DimTime) -- Last 7 days
        GROUP BY WarehouseKey
    )
    SELECT 
        w.WarehouseName,
        p.Category,
        wh.OperationType,
        AVG(wh.DwellTimeMinutes) AS CurrentAvgDwell,
        hb.Threshold AS HistoricalThreshold,
        (AVG(wh.DwellTimeMinutes) - hb.Threshold) / hb.Threshold * 100 AS PercentageOverThreshold,
        COUNT(*) AS AffectedOperations
    FROM FactWarehouseOperations wh
    INNER JOIN DimWarehouse w ON wh.WarehouseKey = w.WarehouseKey
    INNER JOIN DimProduct p ON wh.ProductKey = p.ProductKey
    INNER JOIN HistoricalBaseline hb ON wh.WarehouseKey = hb.WarehouseKey
    WHERE wh.TimeKey >= (SELECT MAX(TimeKey) - 96 FROM DimTime) -- Last 24 hours
    GROUP BY w.WarehouseName, p.Category, wh.OperationType, hb.Threshold
    HAVING AVG(wh.DwellTimeMinutes) > hb.Threshold
    ORDER BY PercentageOverThreshold DESC;
END;
GO
```

### Fleet Triage Scoring

```sql
CREATE PROCEDURE sp_GenerateFleetTriageQueue
AS
BEGIN
    -- Weighted scoring: (delay_impact * 0.4) + (maintenance_urgency * 0.3) + (revenue_risk * 0.3)
    SELECT 
        v.VehicleID,
        v.VehicleMake,
        v.VehicleModel,
        d.DriverName,
        f.DelayMinutes,
        v.MaintenanceScore,
        SUM(p.UnitValue * cd.Quantity) AS CargoValue,
        (
            (f.DelayMinutes / NULLIF(AVG(f.DelayMinutes) OVER(), 0)) * 0.4 +
            (v.MaintenanceScore / 100.0) * 0.3 +
            (SUM(p.UnitValue * cd.Quantity) / NULLIF(MAX(SUM(p.UnitValue * cd.Quantity)) OVER(), 0)) * 0.3
        ) * 100 AS TriagePriority
    FROM FactFleetTrips f
    INNER JOIN DimVehicle v ON f.VehicleKey = v.VehicleKey
    INNER JOIN DimDriver d ON f.DriverKey = d.DriverKey
    INNER JOIN FactCrossDock cd ON f.TripKey = cd.OutboundTripKey
    INNER JOIN DimProduct p ON cd.ProductKey = p.ProductKey
    WHERE f.TimeKey >= (SELECT MAX(TimeKey) - 96 FROM DimTime)
    GROUP BY v.VehicleID, v.VehicleMake, v.VehicleModel, d.DriverName, f.DelayMinutes, v.MaintenanceScore
    ORDER BY TriagePriority DESC;
END;
GO
```

## Power BI Configuration

### DAX Measures for Cross-Fact Analysis

**Dwell Time vs Fleet Efficiency:**
```dax
DwellImpactOnFleet = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR AvgIdle = AVERAGE(FactFleetTrips[IdleTimeMinutes])
RETURN
    DIVIDE(AvgIdle, AvgDwell, 0) * 100
```

**Warehouse Gravity Zone Performance:**
```dax
GravityZoneEfficiency = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[PickRate]),
    FactWarehouseOperations[GravityZone] = "High"
) / 
CALCULATE(
    AVERAGE(FactWarehouseOperations[PickRate]),
    FactWarehouseOperations[GravityZone] = "Low"
)
```

**Real-Time Bottleneck Alert:**
```dax
BottleneckFlag = 
VAR CurrentDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR HistoricalAvg = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        DATESINPERIOD(DimTime[DateTimeValue], MAX(DimTime[DateTimeValue]), -7, DAY)
    )
RETURN
    IF(CurrentDwell > HistoricalAvg * 1.2, "⚠️ Alert", "✓ Normal")
```

### Row-Level Security (RLS)

```dax
-- Create role: RegionalManager
[GeographyRegion] = USERNAME()

-- Create role: WarehouseOperator
[WarehouseCode] IN LOOKUPVALUE(
    UserWarehouseAccess[WarehouseCode],
    UserWarehouseAccess[UserEmail],
    USERPRINCIPALNAME()
)
```

## Data Source Integration

### Polybase External Table for Streaming Telemetry

```sql
-- Create external data source
CREATE EXTERNAL DATA SOURCE FleetTelemetryAPI
WITH (
    TYPE = HADOOP,
    LOCATION = '${FLEET_API_ENDPOINT}',
    CREDENTIAL = FleetAPICredential
);

-- Create external file format
CREATE EXTERNAL FILE FORMAT JSONFormat
WITH (FORMAT_TYPE = JSON);

-- Create external table
CREATE EXTERNAL TABLE ext_FleetTelemetry (
    VehicleID VARCHAR(50),
    Timestamp DATETIME2,
    Latitude DECIMAL(10,6),
    Longitude DECIMAL(10,6),
    Speed DECIMAL(5,2),
    FuelLevel DECIMAL(5,2),
    EngineTemp DECIMAL(5,2)
)
WITH (
    LOCATION = '/telemetry/',
    DATA_SOURCE = FleetTelemetryAPI,
    FILE_FORMAT = JSONFormat
);
```

### Scheduled ETL Job

```sql
CREATE PROCEDURE sp_ETL_Master
AS
BEGIN
    DECLARE @LastLoadTime DATETIME;
    SELECT @LastLoadTime = MAX(LoadedDateTime) FROM ETLAudit;
    
    BEGIN TRANSACTION;
    BEGIN TRY
        -- Load warehouse operations
        EXEC sp_LoadWarehouseOperations @LastLoadTime;
        
        -- Load fleet trips
        EXEC sp_LoadFleetTrips @LastLoadTime;
        
        -- Load cross-dock data
        EXEC sp_LoadCrossDock @LastLoadTime;
        
        -- Update gravity zones
        EXEC sp_UpdateGravityZones;
        
        -- Calculate predictive scores
        EXEC sp_CalculatePredictiveScores;
        
        -- Log successful run
        INSERT INTO ETLAudit (LoadedDateTime, Status) 
        VALUES (GETDATE(), 'Success');
        
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        
        INSERT INTO ETLAudit (LoadedDateTime, Status, ErrorMessage)
        VALUES (GETDATE(), 'Failed', ERROR_MESSAGE());
        
        THROW;
    END CATCH;
END;
GO
```

## Common Usage Patterns

### Query: Identify Slow-Moving SKUs in Wrong Gravity Zones

```sql
SELECT 
    p.SKU,
    p.ProductName,
    p.VelocityClass,
    wh.GravityZone AS CurrentZone,
    p.OptimalZone AS RecommendedZone,
    AVG(wh.DwellTimeMinutes) AS AvgDwellTime,
    COUNT(*) AS OperationCount
FROM FactWarehouseOperations wh
INNER JOIN DimProduct p ON wh.ProductKey = p.ProductKey
WHERE wh.GravityZone <> p.OptimalZone
    AND p.VelocityClass = 'Slow'
GROUP BY p.SKU, p.ProductName, p.VelocityClass, wh.GravityZone, p.OptimalZone
HAVING AVG(wh.DwellTimeMinutes) > 72 * 60 -- 72 hours
ORDER BY AvgDwellTime DESC;
```

### Query: Fleet Fuel Efficiency by Route with Weather Correlation

```sql
SELECT 
    r.RouteName,
    AVG(f.FuelConsumedLiters / NULLIF(f.TripDistanceKm, 0)) AS AvgFuelPerKm,
    w.WeatherCondition,
    AVG(f.DelayMinutes) AS AvgDelay,
    COUNT(*) AS TripCount
FROM FactFleetTrips f
INNER JOIN DimRoute r ON f.RouteKey = r.RouteKey
INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
LEFT JOIN ExternalWeather w ON t.DateTimeValue = w.WeatherDateTime AND r.RouteKey = w.RouteKey
WHERE t.DateTimeValue >= DATEADD(DAY, -30, GETDATE())
GROUP BY r.RouteName, w.WeatherCondition
ORDER BY AvgFuelPerKm DESC;
```

### Query: Real-Time Perishable Goods Alert

```sql
SELECT 
    cd.CrossDockKey,
    p.SKU,
    p.ProductName,
    p.Category,
    cd.TransferTimeMinutes,
    cd.TemperatureCompliance,
    f.VehicleID,
    f.EstimatedArrival,
    CASE 
        WHEN cd.TemperatureCompliance = 0 THEN 'CRITICAL'
        WHEN cd.TransferTimeMinutes > 20 THEN 'WARNING'
        ELSE 'OK'
    END AS AlertLevel
FROM FactCrossDock cd
INNER JOIN DimProduct p ON cd.ProductKey = p.ProductKey
INNER JOIN FactFleetTrips f ON cd.OutboundTripKey = f.TripKey
WHERE p.Category = 'Perishable'
    AND cd.TimeKey >= (SELECT MAX(TimeKey) - 4 FROM DimTime) -- Last hour
    AND (cd.TemperatureCompliance = 0 OR cd.TransferTimeMinutes > 20)
ORDER BY AlertLevel DESC, cd.TransferTimeMinutes DESC;
```

## Automated Alerts Configuration

```sql
CREATE PROCEDURE sp_GenerateAlerts
AS
BEGIN
    -- Alert 1: Fleet idling exceeds 15%
    INSERT INTO AlertQueue (AlertType, Severity, Message, TriggeredAt)
    SELECT 
        'FleetIdling',
        'High',
        'Vehicle ' + v.VehicleID + ' idle time: ' + CAST(f.IdleTimeMinutes AS VARCHAR) + ' min (' + 
        CAST((f.IdleTimeMinutes * 100.0 / NULLIF(DATEDIFF(MINUTE, f.StartTime, f.EndTime), 0)) AS VARCHAR(10)) + '%)',
        GETDATE()
    FROM FactFleetTrips f
    INNER JOIN DimVehicle v ON f.VehicleKey = v.VehicleKey
    WHERE f.IdleTimeMinutes * 100.0 / NULLIF(DATEDIFF(MINUTE, f.StartTime, f.EndTime), 0) > 15
        AND f.TimeKey >= (SELECT MAX(TimeKey) - 4 FROM DimTime);
    
    -- Alert 2: Warehouse capacity exceeds 85%
    INSERT INTO AlertQueue (AlertType, Severity, Message, TriggeredAt)
    SELECT 
        'WarehouseCapacity',
        'Medium',
        'Warehouse ' + w.WarehouseName + ' at ' + CAST(uc.UtilizationPercent AS VARCHAR(10)) + '% capacity',
        GETDATE()
    FROM (
        SELECT 
            WarehouseKey,
            COUNT(*) * 100.0 / (SELECT COUNT(*) FROM DimWarehouseLocation WHERE Active = 1) AS UtilizationPercent
        FROM FactWarehouseOperations
        WHERE TimeKey >= (SELECT MAX(TimeKey) FROM DimTime)
        GROUP BY WarehouseKey
    ) uc
    INNER JOIN DimWarehouse w ON uc.WarehouseKey = w.WarehouseKey
    WHERE uc.UtilizationPercent > 85;
END;
GO

-- Schedule via SQL Agent Job (runs every 15 minutes)
```

## Troubleshooting

### Issue: Slow Cross-Fact Queries

**Symptoms:** Queries joining FactWarehouseOperations and FactFleetTrips take >10 seconds

**Solution:**
```sql
-- Verify indexes exist
EXEC sp_helpindex 'FactWarehouseOperations';
EXEC sp_helpindex 'FactFleetTrips';

-- Create covering indexes if missing
CREATE NONCLUSTERED INDEX IX_WH_Time_Product_INCLUDE 
ON FactWarehouseOperations(TimeKey, ProductKey) 
INCLUDE (DwellTimeMinutes, GravityZone, OperationType);

-- Update statistics
UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS FactFleetTrips WITH FULLSCAN;
```

### Issue: Power BI Refresh Failures

**Symptoms:** Scheduled refresh fails with timeout error

**Solution:**
```sql
-- Partition large fact tables by date
CREATE PARTITION FUNCTION PF_DateRange (DATE)
AS RANGE RIGHT FOR VALUES 
('2026-01-01', '2026-02-01', '2026-03-01', '2026-04-01', '2026-05-01', '2026-06-01');

CREATE PARTITION SCHEME PS_DateRange
AS PARTITION PF_DateRange ALL TO ([PRIMARY]);

-- Rebuild table on partition scheme
CREATE TABLE FactWarehouseOperations_Partitioned (
    -- Same schema as original
) ON PS_DateRange(OperationDate);

-- In Power BI, use incremental refresh with date filter
```

### Issue: Gravity Zone Scores Not Updating

**Symptoms:** Product OptimalZone values remain static despite velocity changes

**Solution:**
```sql
-- Manually trigger gravity recalculation
EXEC sp_UpdateGravityZones;

-- Verify calculation logic
SELECT 
    SKU,
    VelocityClass,
    UnitValue,
    FragilityIndex,
    (VelocityScore * UnitValue / NULLIF(FragilityIndex, 0)) AS RecalculatedGravity,
    GravityScore AS CurrentGravity
FROM DimProduct
WHERE ABS((VelocityScore * UnitValue / NULLIF(FragilityIndex, 0)) - GravityScore) > 5;

-- Schedule as nightly job if needed
```

### Issue: External API Connection Failures

**Symptoms:** Telemetry data not loading, external table queries fail

**Solution:**
```sql
-- Test external data source connectivity
SELECT * FROM ext_FleetTelemetry WHERE Timestamp >= DATEADD(HOUR, -1, GETDATE());

-- Check credential expiration
SELECT * FROM sys.database_scoped_credentials;

-- Refresh credential if expired
ALTER DATABASE SCOPED CREDENTIAL FleetAPICredential
WITH IDENTITY = 'API_USER', SECRET = '${FLEET_API_KEY}';
```

## Performance Optimization

### Columnstore Indexes for Aggregations

```sql
-- Convert historical fact tables to clustered columnstore
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactWarehouseOperations
ON FactWarehouseOperations_Archive
WITH (DROP_EXISTING = OFF, COMPRESSION_DELAY = 0);

-- Use for aggregation queries
SELECT 
    p.Category,
    t.MonthName,
    SUM(wh.DwellTimeMinutes) AS TotalDwell,
    AVG(wh.PickRate) AS AvgPickRate
FROM FactWarehouseOperations_Archive wh
INNER JOIN DimProduct p ON wh.ProductKey = p.ProductKey
INNER JOIN DimTime t ON wh.TimeKey = t.TimeKey
WHERE t.FiscalYear = 2025
GROUP BY p.Category, t.MonthName;
```

### Materialized Views for Complex KPIs

```sql
CREATE VIEW vw_CrossFactKPISnapshot
WITH SCHEMABINDING
AS
SELECT 
    t.DateTimeValue,
    w.WarehouseName,
    p.Category,
    AVG(wh.DwellTimeMinutes) AS AvgDwell,
    AVG(f.IdleTimeMinutes) AS AvgIdle,
    COUNT_BIG(*) AS RecordCount
FROM dbo.FactWarehouseOperations wh
INNER JOIN dbo.DimTime t ON wh.TimeKey = t.TimeKey
INNER JOIN dbo.DimWarehouse w ON wh.WarehouseKey = w.WarehouseKey
INNER JOIN dbo.DimProduct p ON wh.ProductKey = p.ProductKey
INNER JOIN dbo.FactFleetTrips f ON t.TimeKey = f.TimeKey
GROUP BY t.DateTimeValue, w.WarehouseName, p.Category;

-- Create clustered index to materialize
CREATE UNIQUE CLUSTERED INDEX IX_KPISnapshot 
ON vw_CrossFactKPISnapshot(DateTimeValue, WarehouseName, Category);
```

## Best Practices

1. **Incremental Loading:** Always use time-based filtering for ETL to avoid full table scans
2. **Partition Management:** Archive data older than 18 months to separate partition
3. **Security:** Implement row-level security in Power BI, never expose raw connection strings
4. **Documentation:** Annotate all custom DAX measures with business context
5. **Testing:** Validate cross-fact queries against known manual calculations before deploying
6. **Monitoring:** Set up SQL Agent alerts for failed ETL jobs and database space utilization

## Environment Variables Reference

```bash
# SQL Server Connection
SQL_SERVER_HOST=your-server.database.windows.net
SQL_SERVER_DATABASE=LogiFleetPulse
SQL_SERVER_USER=${SQL_SERVER_USER}
SQL_SERVER_PASSWORD=${SQL_SERVER_PASSWORD}

# External APIs
FLEET_API_ENDPOINT=https://telemetry.fleet-provider.com/api/v2
FLEET_API_KEY=${FLEET_API_KEY}
WEATHER_API_ENDPOINT=https://api.weather.com/v3
WEATHER_API_KEY=${WEATHER_API_KEY}
WMS_CONNECTION_STRING=${WMS_CONNECTION_STRING}

# Power BI Service
POWERBI_WORKSPACE_ID=${POWERBI_WORKSPACE_ID}
POWERBI_REFRESH_TOKEN=${POWERBI_REFRESH_TOKEN}
```
