---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing platform for logistics intelligence with multi-fact star schema and real-time fleet optimization
triggers:
  - set up supply chain analytics dashboard
  - create logistics data warehouse
  - implement fleet tracking with Power BI
  - build warehouse operations reporting
  - design multi-fact star schema for logistics
  - configure supply chain KPI dashboard
  - integrate fleet telemetry with warehouse data
  - deploy logistics intelligence platform
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

LogiFleet Pulse is an advanced logistics intelligence platform that unifies warehouse operations, fleet telemetry, and supply chain data into a single analytical layer. Built on MS SQL Server with Power BI visualizations, it provides real-time insights through a multi-fact star schema architecture.

## Project Overview

This is a **data modeling and visualization template** for logistics operations that includes:
- Multi-fact star schema SQL database design
- Power BI dashboard templates (.pbit files)
- Stored procedures for incremental data loading
- Cross-fact KPI calculations
- Time-phased dimensional modeling
- Role-based security implementation

**Primary Language**: SQL (MS SQL Server)
**Visualization**: Power BI
**Architecture**: Star schema data warehouse with time-phased dimensions

## Installation & Setup

### Prerequisites

- MS SQL Server 2019 or later
- Power BI Desktop (latest version)
- SSMS (SQL Server Management Studio)
- Access to source systems (WMS, TMS, telematics feeds)

### Database Deployment

1. **Clone the repository**:
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the schema**:
```sql
-- Connect to your SQL Server instance via SSMS
-- Create the database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Execute the schema script (typically in /sql/schema.sql)
-- This creates dimension and fact tables
```

3. **Configure connection strings**:
```json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_SERVER_USER}",
    "password": "${SQL_SERVER_PASSWORD}",
    "trusted_connection": false
  },
  "data_sources": {
    "wms_api": "${WMS_API_ENDPOINT}",
    "telematics_api": "${FLEET_TELEMATICS_ENDPOINT}",
    "weather_api": "${WEATHER_API_ENDPOINT}"
  }
}
```

## Core Schema Architecture

### Dimension Tables

**DimTime** - Time dimension at 15-minute granularity:
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    DateTime DATETIME NOT NULL,
    TimeSlot15Min TIME,
    HourOfDay INT,
    DayOfWeek INT,
    WeekOfYear INT,
    MonthNumber INT,
    QuarterNumber INT,
    FiscalYear INT,
    IsWeekend BIT,
    IsHoliday BIT
)

-- Create indexes for common query patterns
CREATE INDEX IX_DimTime_DateTime ON DimTime(DateTime)
CREATE INDEX IX_DimTime_FiscalYear_Quarter ON DimTime(FiscalYear, QuarterNumber)
```

**DimGeography** - Hierarchical location dimension:
```sql
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) UNIQUE NOT NULL,
    LocationName VARCHAR(200),
    LocationType VARCHAR(50), -- 'Warehouse', 'RouteNode', 'Customer', 'Supplier'
    Address VARCHAR(500),
    City VARCHAR(100),
    Region VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    EffectiveDate DATE,
    ExpirationDate DATE,
    IsCurrent BIT
)

CREATE INDEX IX_DimGeography_LocationID ON DimGeography(LocationID)
CREATE INDEX IX_DimGeography_Current ON DimGeography(IsCurrent, LocationType)
```

**DimProductGravity** - Product classification with velocity scoring:
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) UNIQUE NOT NULL,
    ProductName VARCHAR(500),
    Category VARCHAR(100),
    Subcategory VARCHAR(100),
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value / fragility
    VelocityClass VARCHAR(20), -- 'Fast', 'Medium', 'Slow'
    ValueClass VARCHAR(20), -- 'High', 'Medium', 'Low'
    FragilityIndex DECIMAL(3,2),
    Weight DECIMAL(10,2),
    Volume DECIMAL(10,2),
    RequiresColdStorage BIT,
    EffectiveDate DATE,
    ExpirationDate DATE,
    IsCurrent BIT
)

CREATE INDEX IX_DimProductGravity_SKU ON DimProductGravity(SKU)
CREATE INDEX IX_DimProductGravity_Gravity ON DimProductGravity(GravityScore DESC, IsCurrent)
```

### Fact Tables

**FactWarehouseOperations** - Core warehouse activity tracking:
```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OperationStartTime DATETIME,
    OperationEndTime DATETIME,
    QuantityUnits INT,
    DwellTimeHours DECIMAL(10,2),
    OperatorID VARCHAR(50),
    StorageZone VARCHAR(50),
    BinLocation VARCHAR(50),
    OrderID VARCHAR(100),
    BatchNumber VARCHAR(100),
    DefectFlag BIT,
    CostPerUnit DECIMAL(10,4)
)

CREATE INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey, OperationType)
CREATE INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey, GeographyKey)
CREATE INDEX IX_FactWarehouse_Order ON FactWarehouseOperations(OrderID)
```

**FactFleetTrips** - Fleet movement and telemetry:
```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKeyStart INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    TimeKeyEnd INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID VARCHAR(50),
    DriverID VARCHAR(50),
    RouteID VARCHAR(100),
    TripStartTime DATETIME,
    TripEndTime DATETIME,
    DistanceKm DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    LoadWeightKg DECIMAL(10,2),
    StopCount INT,
    OnTimeDeliveryFlag BIT,
    DelayMinutes INT,
    WeatherCondition VARCHAR(50),
    TrafficDelayMinutes INT,
    MaintenanceFlag BIT,
    CostTotal DECIMAL(10,2)
)

CREATE INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKeyStart, TimeKeyEnd)
CREATE INDEX IX_FactFleet_Route ON FactFleetTrips(RouteID, VehicleID)
CREATE INDEX IX_FactFleet_Geography ON FactFleetTrips(OriginGeographyKey, DestinationGeographyKey)
```

## Key Stored Procedures

### Incremental Data Loading

```sql
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LoadDate DATE = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Default to yesterday if no date provided
    IF @LoadDate IS NULL
        SET @LoadDate = CAST(GETDATE() - 1 AS DATE)
    
    -- Stage data from source system
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, GeographyKey, OperationType,
        OperationStartTime, OperationEndTime, QuantityUnits,
        DwellTimeHours, OperatorID, StorageZone, BinLocation,
        OrderID, BatchNumber, DefectFlag, CostPerUnit
    )
    SELECT 
        t.TimeKey,
        p.ProductKey,
        g.GeographyKey,
        src.OperationType,
        src.StartTime,
        src.EndTime,
        src.Quantity,
        DATEDIFF(MINUTE, src.StartTime, src.EndTime) / 60.0 AS DwellTimeHours,
        src.OperatorID,
        src.ZoneCode,
        src.BinLocation,
        src.OrderID,
        src.BatchNumber,
        src.DefectFlag,
        src.CostPerUnit
    FROM StagingWarehouseOps src
    INNER JOIN DimTime t ON CAST(src.StartTime AS DATETIME) = t.DateTime
    INNER JOIN DimProductGravity p ON src.SKU = p.SKU AND p.IsCurrent = 1
    INNER JOIN DimGeography g ON src.WarehouseCode = g.LocationID AND g.IsCurrent = 1
    WHERE CAST(src.StartTime AS DATE) = @LoadDate
        AND NOT EXISTS (
            SELECT 1 FROM FactWarehouseOperations f
            WHERE f.OrderID = src.OrderID AND f.OperationStartTime = src.StartTime
        )
    
    -- Update gravity scores based on new activity
    EXEC usp_UpdateProductGravityScores @LoadDate
END
GO
```

### Cross-Fact KPI Calculation

```sql
CREATE PROCEDURE usp_CalculateCrossFactKPIs
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    -- Calculate dwell time vs fleet idle correlation
    SELECT 
        p.Category,
        g.Region,
        AVG(w.DwellTimeHours) AS AvgDwellTimeHours,
        AVG(f.IdleTimeMinutes) AS AvgFleetIdleMinutes,
        SUM(w.QuantityUnits) AS TotalUnitsProcessed,
        SUM(f.DistanceKm) AS TotalDistanceKm,
        SUM(f.FuelConsumedLiters) AS TotalFuelLiters,
        -- Composite efficiency score
        (SUM(w.QuantityUnits) / NULLIF(AVG(w.DwellTimeHours), 0)) * 
        (SUM(f.DistanceKm) / NULLIF(SUM(f.FuelConsumedLiters), 0)) AS EfficiencyScore
    FROM FactWarehouseOperations w
    INNER JOIN DimTime tw ON w.TimeKey = tw.TimeKey
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    INNER JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
    LEFT JOIN FactFleetTrips f ON w.OrderID = f.RouteID 
        AND CAST(w.OperationEndTime AS DATE) = CAST(f.TripStartTime AS DATE)
    WHERE CAST(tw.DateTime AS DATE) BETWEEN @StartDate AND @EndDate
    GROUP BY p.Category, g.Region
    ORDER BY EfficiencyScore DESC
END
GO
```

### Predictive Bottleneck Detection

```sql
CREATE PROCEDURE usp_DetectBottlenecks
    @ForecastHours INT = 24
AS
BEGIN
    -- Identify zones with increasing dwell time trend
    WITH DwellTrends AS (
        SELECT 
            StorageZone,
            DATEPART(HOUR, OperationStartTime) AS HourOfDay,
            AVG(DwellTimeHours) AS AvgDwell,
            STDEV(DwellTimeHours) AS StdDevDwell,
            COUNT(*) AS OperationCount
        FROM FactWarehouseOperations
        WHERE OperationStartTime >= DATEADD(DAY, -7, GETDATE())
        GROUP BY StorageZone, DATEPART(HOUR, OperationStartTime)
    ),
    BottleneckScore AS (
        SELECT 
            StorageZone,
            HourOfDay,
            AvgDwell,
            -- Score: weighted by variance and volume
            (AvgDwell * StdDevDwell * OperationCount) / 100.0 AS BottleneckIndex,
            RANK() OVER (ORDER BY (AvgDwell * StdDevDwell * OperationCount) DESC) AS RiskRank
        FROM DwellTrends
        WHERE AvgDwell > 2.0 -- Threshold: more than 2 hours
    )
    SELECT 
        StorageZone,
        HourOfDay,
        AvgDwell,
        BottleneckIndex,
        RiskRank,
        CASE 
            WHEN RiskRank <= 5 THEN 'CRITICAL'
            WHEN RiskRank <= 15 THEN 'HIGH'
            WHEN RiskRank <= 30 THEN 'MEDIUM'
            ELSE 'LOW'
        END AS AlertLevel
    FROM BottleneckScore
    WHERE RiskRank <= 30
    ORDER BY RiskRank
END
GO
```

## Power BI Integration

### Connection Setup

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Server: `${SQL_SERVER_HOST}`
4. Database: `LogiFleetPulse`
5. Data Connectivity mode: **DirectQuery** (for real-time) or **Import** (for performance)

### DAX Measures

**Total Dwell Time**:
```dax
TotalDwellTime = 
SUM(FactWarehouseOperations[DwellTimeHours])
```

**Average Fleet Efficiency**:
```dax
AvgFleetEfficiency = 
DIVIDE(
    SUM(FactFleetTrips[DistanceKm]),
    SUM(FactFleetTrips[FuelConsumedLiters]),
    0
)
```

**Cross-Fact Correlation Score**:
```dax
CorrelationScore = 
VAR WarehouseEfficiency = 
    DIVIDE(
        SUM(FactWarehouseOperations[QuantityUnits]),
        SUM(FactWarehouseOperations[DwellTimeHours]),
        0
    )
VAR FleetEfficiency = [AvgFleetEfficiency]
RETURN
    WarehouseEfficiency * FleetEfficiency * 10
```

**Dynamic Gravity Zone Ranking**:
```dax
GravityZoneRank = 
RANKX(
    ALL(DimProductGravity[Category]),
    CALCULATE(SUM(DimProductGravity[GravityScore])),
    ,
    DESC,
    DENSE
)
```

### Row-Level Security

```dax
-- RLS rule for regional managers
[Region] = USERNAME()

-- RLS rule for warehouse-specific access
[GeographyKey] IN 
    LOOKUPVALUE(
        UserAccess[GeographyKey],
        UserAccess[UserEmail],
        USERNAME()
    )
```

## Common Usage Patterns

### Daily Operations Dashboard Query

```sql
-- Dashboard summary for current day
DECLARE @Today DATE = CAST(GETDATE() AS DATE)

SELECT 
    'Warehouse' AS Metric,
    COUNT(DISTINCT OperationKey) AS Operations,
    SUM(QuantityUnits) AS TotalUnits,
    AVG(DwellTimeHours) AS AvgDwellHours,
    SUM(CostPerUnit * QuantityUnits) AS TotalCost
FROM FactWarehouseOperations w
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
WHERE CAST(t.DateTime AS DATE) = @Today

UNION ALL

SELECT 
    'Fleet' AS Metric,
    COUNT(DISTINCT TripKey) AS Operations,
    SUM(DistanceKm) AS TotalUnits,
    AVG(IdleTimeMinutes / 60.0) AS AvgDwellHours,
    SUM(CostTotal) AS TotalCost
FROM FactFleetTrips f
INNER JOIN DimTime t ON f.TimeKeyStart = t.TimeKey
WHERE CAST(t.DateTime AS DATE) = @Today
```

### Automated Alert Configuration

```sql
-- Setup threshold-based alerts
CREATE TABLE AlertThresholds (
    AlertID INT PRIMARY KEY IDENTITY(1,1),
    MetricName VARCHAR(100),
    ThresholdValue DECIMAL(10,2),
    ComparisonOperator VARCHAR(10), -- '>', '<', '>=', '<=', '='
    AlertLevel VARCHAR(20), -- 'INFO', 'WARNING', 'CRITICAL'
    NotificationEmail VARCHAR(500),
    IsActive BIT DEFAULT 1
)

-- Insert sample thresholds
INSERT INTO AlertThresholds (MetricName, ThresholdValue, ComparisonOperator, AlertLevel, NotificationEmail)
VALUES 
    ('DwellTimeHours', 3.0, '>', 'WARNING', '${ALERT_EMAIL}'),
    ('FleetIdleTimePercent', 15.0, '>', 'CRITICAL', '${ALERT_EMAIL}'),
    ('OnTimeDeliveryRate', 95.0, '<', 'WARNING', '${ALERT_EMAIL}')

-- Evaluation procedure
CREATE PROCEDURE usp_EvaluateAlerts
AS
BEGIN
    -- Check dwell time threshold
    IF EXISTS (
        SELECT 1 FROM FactWarehouseOperations
        WHERE OperationStartTime >= DATEADD(HOUR, -1, GETDATE())
            AND DwellTimeHours > (SELECT ThresholdValue FROM AlertThresholds WHERE MetricName = 'DwellTimeHours')
    )
    BEGIN
        -- Trigger notification (integrate with email service)
        PRINT 'ALERT: Dwell time threshold exceeded'
    END
END
GO
```

## Configuration Files

### SQL Server Connection

Create `config/database.json`:
```json
{
  "server": "${SQL_SERVER_HOST}",
  "database": "LogiFleetPulse",
  "authentication": {
    "type": "sql",
    "username": "${SQL_SERVER_USER}",
    "password": "${SQL_SERVER_PASSWORD}"
  },
  "connection_pooling": {
    "min_pool_size": 5,
    "max_pool_size": 100,
    "connection_timeout": 30
  },
  "advanced": {
    "multi_subnet_failover": true,
    "encrypt_connection": true,
    "trust_server_certificate": false
  }
}
```

### Data Source Connections

Create `config/data_sources.json`:
```json
{
  "wms": {
    "type": "rest_api",
    "endpoint": "${WMS_API_ENDPOINT}",
    "auth": {
      "type": "bearer",
      "token": "${WMS_API_TOKEN}"
    },
    "refresh_interval_minutes": 15
  },
  "telematics": {
    "type": "mqtt",
    "broker": "${MQTT_BROKER_HOST}",
    "port": 8883,
    "topics": ["fleet/*/telemetry", "fleet/*/location"],
    "auth": {
      "username": "${MQTT_USER}",
      "password": "${MQTT_PASSWORD}"
    }
  },
  "weather": {
    "type": "rest_api",
    "endpoint": "${WEATHER_API_ENDPOINT}",
    "auth": {
      "type": "query_param",
      "param_name": "apikey",
      "param_value": "${WEATHER_API_KEY}"
    },
    "refresh_interval_minutes": 60
  }
}
```

## Troubleshooting

### Performance Issues

**Slow queries on large fact tables**:
```sql
-- Add columnstore indexes for analytical workloads
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouseOps
ON FactWarehouseOperations (TimeKey, ProductKey, GeographyKey, QuantityUnits, DwellTimeHours)

CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactFleetTrips
ON FactFleetTrips (TimeKeyStart, OriginGeographyKey, DestinationGeographyKey, DistanceKm, FuelConsumedLiters)

-- Partition by date for time-series queries
ALTER TABLE FactWarehouseOperations
SWITCH PARTITION ... -- Implement monthly partitioning scheme
```

**Power BI refresh timeout**:
```sql
-- Use indexed views for complex aggregations
CREATE VIEW vw_DailyWarehouseSummary
WITH SCHEMABINDING
AS
SELECT 
    CAST(t.DateTime AS DATE) AS OperationDate,
    w.GeographyKey,
    w.ProductKey,
    COUNT_BIG(*) AS OperationCount,
    SUM(w.QuantityUnits) AS TotalUnits,
    AVG(w.DwellTimeHours) AS AvgDwellTime
FROM dbo.FactWarehouseOperations w
INNER JOIN dbo.DimTime t ON w.TimeKey = t.TimeKey
GROUP BY CAST(t.DateTime AS DATE), w.GeographyKey, w.ProductKey
GO

CREATE UNIQUE CLUSTERED INDEX IX_DailyWarehouseSummary
ON vw_DailyWarehouseSummary(OperationDate, GeographyKey, ProductKey)
```

### Data Quality Issues

**Duplicate detection**:
```sql
-- Identify duplicate operations
WITH Duplicates AS (
    SELECT 
        OrderID, 
        OperationStartTime,
        COUNT(*) AS DuplicateCount
    FROM FactWarehouseOperations
    GROUP BY OrderID, OperationStartTime
    HAVING COUNT(*) > 1
)
SELECT 
    f.*,
    d.DuplicateCount
FROM FactWarehouseOperations f
INNER JOIN Duplicates d ON f.OrderID = d.OrderID 
    AND f.OperationStartTime = d.OperationStartTime
ORDER BY d.DuplicateCount DESC, f.OrderID

-- Remove duplicates (keep latest entry)
;WITH CTE AS (
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY OrderID, OperationStartTime 
            ORDER BY OperationKey DESC
        ) AS rn
    FROM FactWarehouseOperations
)
DELETE FROM CTE WHERE rn > 1
```

**Missing dimension references**:
```sql
-- Find orphaned fact records
SELECT 
    'Missing Product' AS Issue,
    COUNT(*) AS RecordCount
FROM FactWarehouseOperations w
LEFT JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
WHERE p.ProductKey IS NULL

UNION ALL

SELECT 
    'Missing Geography' AS Issue,
    COUNT(*) AS RecordCount
FROM FactWarehouseOperations w
LEFT JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
WHERE g.GeographyKey IS NULL
```

### Power BI Connection Errors

**DirectQuery timeout**:
- Increase timeout in Power BI: File → Options → Data Load → Query timeout
- Or switch to Import mode for better performance

**Authentication failures**:
```sql
-- Verify SQL Server authentication mode
SELECT 
    SERVERPROPERTY('IsIntegratedSecurityOnly') AS IsWindowsAuth
-- If 1, enable mixed mode authentication in SQL Server Configuration
```

**Row-level security not applying**:
```dax
// Debug RLS in Power BI
// Create a measure to show current user context
CurrentUser = USERNAME()

// Verify filter context
FilteredRegions = 
CONCATENATEX(
    VALUES(DimGeography[Region]),
    DimGeography[Region],
    ", "
)
```

## Maintenance Scripts

### Daily Maintenance Job

```sql
CREATE PROCEDURE usp_DailyMaintenance
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Update statistics on fact tables
    UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN
    UPDATE STATISTICS FactFleetTrips WITH FULLSCAN
    
    -- Rebuild fragmented indexes (if fragmentation > 30%)
    EXEC sp_MSforeachtable @command1="
        DECLARE @frag FLOAT
        SELECT @frag = avg_fragmentation_in_percent 
        FROM sys.dm_db_index_physical_stats(DB_ID(), OBJECT_ID('?'), NULL, NULL, 'LIMITED')
        WHERE index_id > 0
        IF @frag > 30
            ALTER INDEX ALL ON ? REBUILD WITH (ONLINE = ON)
    "
    
    -- Archive old data (older than 2 years)
    DELETE FROM FactWarehouseOperations
    WHERE OperationStartTime < DATEADD(YEAR, -2, GETDATE())
    
    DELETE FROM FactFleetTrips
    WHERE TripStartTime < DATEADD(YEAR, -2, GETDATE())
    
    -- Shrink log file if necessary
    DBCC SHRINKFILE (LogiFleetPulse_log, 1024)
END
GO

-- Schedule as SQL Agent job to run daily at 2 AM
```

This skill provides the complete foundation for deploying and operating the LogiFleet Pulse logistics analytics platform using MS SQL Server and Power BI.
