---
name: logifleet-pulse-supply-chain-analytics
description: Advanced MS SQL Server & Power BI data warehousing engine for fleet logistics and supply chain analytics with multi-fact star schema modeling
triggers:
  - set up logifleet pulse supply chain analytics
  - configure power bi logistics dashboard
  - implement multi-fact star schema for warehouse operations
  - deploy sql server supply chain data warehouse
  - create fleet telemetry analytics pipeline
  - build cross-modal logistics intelligence dashboard
  - optimize warehouse gravity zones with sql
  - integrate logistics data sources with power bi
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a multi-modal logistics intelligence engine that combines:
- **MS SQL Server data warehouse** with custom multi-fact star schema
- **Power BI dashboards** for real-time logistics visualization
- **Cross-fact KPI harmonization** linking warehouse, fleet, and supplier metrics
- **Predictive analytics** for bottleneck detection and fleet optimization
- **Warehouse Gravity Zones** for spatial optimization based on pick frequency and value

It ingests data from warehouse management systems, fleet telemetry, supplier portals, and external APIs to provide unified supply chain visibility.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise)
- Power BI Desktop (latest version)
- Network access to WMS, TMS, or telemetry data sources

### Step 1: Clone Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### Step 2: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Run the main schema deployment script
USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute schema scripts in order:
-- 1. Create dimension tables
-- 2. Create fact tables
-- 3. Create views and stored procedures
-- 4. Set up indexing strategy
```

### Step 3: Configure Data Sources

Update connection settings (use environment variables for sensitive data):

```json
{
  "sql_server": {
    "host": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_SERVER_USER}",
    "password": "${SQL_SERVER_PASSWORD}"
  },
  "data_sources": {
    "wms_api": "${WMS_API_ENDPOINT}",
    "fleet_telemetry": "${FLEET_TELEMETRY_ENDPOINT}",
    "weather_api": "${WEATHER_API_ENDPOINT}"
  }
}
```

### Step 4: Import Power BI Template

```powershell
# Open Power BI Desktop
# File → Import → Power BI Template
# Select LogiFleet_Pulse_Master.pbit
# Enter SQL Server connection string when prompted
```

## Core Data Model Architecture

### Dimension Tables

```sql
-- DimTime: 15-minute granularity time dimension
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateValue DATE NOT NULL,
    TimeSlot TIME NOT NULL,
    HourOfDay TINYINT,
    DayOfWeek VARCHAR(10),
    FiscalPeriod VARCHAR(6),
    IsBusinessDay BIT,
    INDEX IX_DimTime_Date (DateValue)
);

-- DimGeography: Hierarchical location dimension
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationCode VARCHAR(20) UNIQUE NOT NULL,
    LocationName VARCHAR(100),
    LocationType VARCHAR(20), -- 'Warehouse', 'RouteNode', 'CustomerSite'
    Continent VARCHAR(50),
    Country VARCHAR(50),
    Region VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    INDEX IX_DimGeography_Type (LocationType)
);

-- DimProductGravity: Product classification with gravity scoring
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    VelocityScore DECIMAL(5,2), -- Pick frequency normalized
    ValueScore DECIMAL(5,2), -- Unit value normalized
    FragilityScore DECIMAL(5,2), -- 0-100 scale
    GravityIndex AS (VelocityScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) PERSISTED,
    RecommendedZone VARCHAR(20), -- 'High', 'Medium', 'Low' gravity
    INDEX IX_DimProduct_Gravity (GravityIndex DESC)
);

-- DimSupplierReliability: Supplier performance tracking
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierCode VARCHAR(20) UNIQUE NOT NULL,
    SupplierName VARCHAR(200),
    LeadTimeVariance DECIMAL(5,2), -- Days std dev
    DefectRate DECIMAL(5,4), -- Percentage
    OnTimeDeliveryRate DECIMAL(5,4), -- Percentage
    ComplianceScore DECIMAL(5,2), -- 0-100
    INDEX IX_DimSupplier_Reliability (ComplianceScore DESC)
);
```

### Fact Tables

```sql
-- FactWarehouseOperations: Core warehouse activity tracking
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    OperationType VARCHAR(20), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    QuantityHandled INT,
    DwellTimeMinutes INT,
    CycleTimeSeconds INT,
    LaborCost DECIMAL(10,2),
    EquipmentUsedCode VARCHAR(20),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    INDEX IX_FactWarehouse_Time (TimeKey),
    INDEX IX_FactWarehouse_Product (ProductKey),
    INDEX IX_FactWarehouse_OpType (OperationType)
);

-- FactFleetTrips: Fleet telemetry and route performance
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    VehicleID VARCHAR(20) NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    DistanceKM DECIMAL(8,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(8,2),
    LoadWeightKG DECIMAL(10,2),
    DriverID VARCHAR(20),
    RouteEfficiencyScore DECIMAL(5,2), -- 0-100
    MaintenanceFlag BIT,
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey),
    INDEX IX_FactFleet_Time (TimeKey),
    INDEX IX_FactFleet_Vehicle (VehicleID)
);

-- FactCrossDock: Cross-dock transfers without long-term storage
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    SupplierKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    QuantityTransferred INT,
    TransferTimeMinutes INT,
    QualityCheckPassed BIT,
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    FOREIGN KEY (SupplierKey) REFERENCES DimSupplierReliability(SupplierKey),
    INDEX IX_FactCrossDock_Time (TimeKey)
);
```

## Key SQL Queries & Stored Procedures

### Cross-Fact KPI: Dwell Time vs Fleet Idling Cost

```sql
-- Calculate correlation between warehouse dwell time and fleet idle costs
CREATE PROCEDURE sp_AnalyzeDwellVsIdleCost
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    WITH WarehouseDwell AS (
        SELECT 
            wo.ProductKey,
            AVG(wo.DwellTimeMinutes) AS AvgDwellMinutes,
            dt.DateValue
        FROM FactWarehouseOperations wo
        INNER JOIN DimTime dt ON wo.TimeKey = dt.TimeKey
        WHERE dt.DateValue BETWEEN @StartDate AND @EndDate
            AND wo.OperationType = 'Putaway'
        GROUP BY wo.ProductKey, dt.DateValue
    ),
    FleetIdle AS (
        SELECT 
            dt.DateValue,
            AVG(ft.IdleTimeMinutes) AS AvgIdleMinutes,
            SUM(ft.FuelConsumedLiters * 1.5) AS IdleCostUSD -- $1.5 per liter estimate
        FROM FactFleetTrips ft
        INNER JOIN DimTime dt ON ft.TimeKey = dt.TimeKey
        WHERE dt.DateValue BETWEEN @StartDate AND @EndDate
        GROUP BY dt.DateValue
    )
    SELECT 
        wd.DateValue,
        wd.ProductKey,
        pg.SKU,
        pg.ProductName,
        wd.AvgDwellMinutes,
        fi.AvgIdleMinutes,
        fi.IdleCostUSD,
        pg.GravityIndex,
        CASE 
            WHEN wd.AvgDwellMinutes > 72 AND pg.GravityIndex < 30 
            THEN 'Recommend Zone Reassignment'
            ELSE 'Optimal'
        END AS Recommendation
    FROM WarehouseDwell wd
    INNER JOIN DimProductGravity pg ON wd.ProductKey = pg.ProductKey
    CROSS JOIN FleetIdle fi
    WHERE wd.DateValue = fi.DateValue
    ORDER BY wd.AvgDwellMinutes DESC, fi.IdleCostUSD DESC;
END;
GO

-- Execute the stored procedure
EXEC sp_AnalyzeDwellVsIdleCost 
    @StartDate = '2026-06-01', 
    @EndDate = '2026-06-30';
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify products in suboptimal zones based on gravity index
CREATE VIEW vw_GravityZoneOptimization AS
SELECT 
    pg.ProductKey,
    pg.SKU,
    pg.ProductName,
    pg.GravityIndex,
    pg.RecommendedZone,
    dg.LocationName AS CurrentZone,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
    COUNT(wo.OperationKey) AS PickFrequency,
    CASE 
        WHEN pg.GravityIndex >= 70 AND dg.LocationName NOT LIKE '%Zone-A%' 
        THEN 'Move to High-Gravity Zone'
        WHEN pg.GravityIndex BETWEEN 40 AND 69 AND dg.LocationName NOT LIKE '%Zone-B%'
        THEN 'Move to Medium-Gravity Zone'
        WHEN pg.GravityIndex < 40 AND dg.LocationName NOT LIKE '%Zone-C%'
        THEN 'Move to Low-Gravity Zone'
        ELSE 'Optimally Placed'
    END AS OptimizationAction
FROM DimProductGravity pg
INNER JOIN FactWarehouseOperations wo ON pg.ProductKey = wo.ProductKey
INNER JOIN DimGeography dg ON wo.GeographyKey = dg.GeographyKey
WHERE wo.OperationType IN ('Picking', 'Putaway')
GROUP BY pg.ProductKey, pg.SKU, pg.ProductName, pg.GravityIndex, 
         pg.RecommendedZone, dg.LocationName
HAVING AVG(wo.DwellTimeMinutes) > 48; -- Focus on items with >48hr dwell
GO

-- Query the optimization view
SELECT * 
FROM vw_GravityZoneOptimization
WHERE OptimizationAction <> 'Optimally Placed'
ORDER BY GravityIndex DESC, AvgDwellTime DESC;
```

### Predictive Bottleneck Detection

```sql
-- Detect potential bottlenecks using time-series patterns
CREATE PROCEDURE sp_PredictBottlenecks
    @LookbackDays INT = 30,
    @ThresholdPercentile DECIMAL(3,2) = 0.90
AS
BEGIN
    WITH TimeSeriesMetrics AS (
        SELECT 
            dt.DateValue,
            dt.HourOfDay,
            dg.LocationName,
            AVG(wo.CycleTimeSeconds) AS AvgCycleTime,
            STDEV(wo.CycleTimeSeconds) AS StdDevCycleTime,
            COUNT(wo.OperationKey) AS OperationVolume
        FROM FactWarehouseOperations wo
        INNER JOIN DimTime dt ON wo.TimeKey = dt.TimeKey
        INNER JOIN DimGeography dg ON wo.GeographyKey = dg.GeographyKey
        WHERE dt.DateValue >= DATEADD(DAY, -@LookbackDays, GETDATE())
        GROUP BY dt.DateValue, dt.HourOfDay, dg.LocationName
    ),
    Percentiles AS (
        SELECT 
            LocationName,
            PERCENTILE_CONT(@ThresholdPercentile) 
                WITHIN GROUP (ORDER BY AvgCycleTime) 
                OVER (PARTITION BY LocationName) AS P90CycleTime
        FROM TimeSeriesMetrics
    )
    SELECT DISTINCT
        tsm.LocationName,
        tsm.DateValue,
        tsm.HourOfDay,
        tsm.AvgCycleTime,
        tsm.StdDevCycleTime,
        tsm.OperationVolume,
        p.P90CycleTime,
        CASE 
            WHEN tsm.AvgCycleTime > p.P90CycleTime 
            THEN 'HIGH RISK'
            WHEN tsm.AvgCycleTime > p.P90CycleTime * 0.85
            THEN 'MODERATE RISK'
            ELSE 'LOW RISK'
        END AS BottleneckRisk
    FROM TimeSeriesMetrics tsm
    INNER JOIN Percentiles p ON tsm.LocationName = p.LocationName
    WHERE tsm.AvgCycleTime > p.P90CycleTime * 0.8 -- Focus on near-threshold cases
    ORDER BY tsm.DateValue DESC, tsm.AvgCycleTime DESC;
END;
GO

-- Execute bottleneck prediction
EXEC sp_PredictBottlenecks 
    @LookbackDays = 30, 
    @ThresholdPercentile = 0.90;
```

## Data Loading Patterns

### Incremental ETL with Polybase

```sql
-- Create external data source for WMS flat files
CREATE EXTERNAL DATA SOURCE WMS_FileShare
WITH (
    TYPE = HADOOP,
    LOCATION = 'wasbs://wms-exports@${STORAGE_ACCOUNT}.blob.core.windows.net'
);

-- Create external file format
CREATE EXTERNAL FILE FORMAT CSV_Format
WITH (
    FORMAT_TYPE = DELIMITEDTEXT,
    FORMAT_OPTIONS (
        FIELD_TERMINATOR = ',',
        STRING_DELIMITER = '"',
        FIRST_ROW = 2,
        USE_TYPE_DEFAULT = FALSE
    )
);

-- Create external table for staging
CREATE EXTERNAL TABLE ext_WarehouseOperations_Staging (
    OperationTimestamp DATETIME2,
    SKU VARCHAR(50),
    LocationCode VARCHAR(20),
    OperationType VARCHAR(20),
    QuantityHandled INT,
    CycleTimeSeconds INT
)
WITH (
    LOCATION = '/warehouse-ops/',
    DATA_SOURCE = WMS_FileShare,
    FILE_FORMAT = CSV_Format
);

-- Incremental merge procedure
CREATE PROCEDURE sp_LoadWarehouseOperations
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Upsert from staging to dimension tables first
    MERGE DimProductGravity AS target
    USING (SELECT DISTINCT SKU FROM ext_WarehouseOperations_Staging) AS source
    ON target.SKU = source.SKU
    WHEN NOT MATCHED THEN
        INSERT (SKU, ProductName, VelocityScore, ValueScore, FragilityScore)
        VALUES (source.SKU, 'Pending Classification', 50, 50, 50);
    
    -- Insert fact records with proper foreign keys
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, GeographyKey, OperationType,
        QuantityHandled, CycleTimeSeconds, DwellTimeMinutes
    )
    SELECT 
        dt.TimeKey,
        pg.ProductKey,
        dg.GeographyKey,
        ext.OperationType,
        ext.QuantityHandled,
        ext.CycleTimeSeconds,
        0 -- Dwell time calculated in separate process
    FROM ext_WarehouseOperations_Staging ext
    INNER JOIN DimTime dt ON 
        CAST(ext.OperationTimestamp AS DATE) = dt.DateValue
        AND DATEPART(HOUR, ext.OperationTimestamp) = dt.HourOfDay
    INNER JOIN DimProductGravity pg ON ext.SKU = pg.SKU
    INNER JOIN DimGeography dg ON ext.LocationCode = dg.LocationCode
    WHERE NOT EXISTS (
        SELECT 1 FROM FactWarehouseOperations wo
        WHERE wo.TimeKey = dt.TimeKey
          AND wo.ProductKey = pg.ProductKey
          AND wo.CycleTimeSeconds = ext.CycleTimeSeconds
    );
    
    PRINT CONCAT('Loaded ', @@ROWCOUNT, ' warehouse operation records');
END;
GO

-- Schedule this procedure to run every 15 minutes
```

## Power BI Integration

### DAX Measures for Cross-Fact Analysis

```dax
// Composite Fleet Efficiency Score
Fleet Efficiency Score = 
VAR AvgRouteEfficiency = AVERAGE(FactFleetTrips[RouteEfficiencyScore])
VAR IdleTimeRatio = 
    DIVIDE(
        SUM(FactFleetTrips[IdleTimeMinutes]),
        SUM(FactFleetTrips[DurationMinutes]),
        0
    )
VAR FuelEfficiency = 
    DIVIDE(
        SUM(FactFleetTrips[DistanceKM]),
        SUM(FactFleetTrips[FuelConsumedLiters]),
        0
    )
RETURN
    (AvgRouteEfficiency * 0.4) + 
    ((1 - IdleTimeRatio) * 100 * 0.3) + 
    (FuelEfficiency * 10 * 0.3) // Normalize to 0-100 scale

// Warehouse Gravity Zone Compliance
Gravity Zone Compliance % = 
VAR CorrectlyPlaced = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FILTER(
            DimProductGravity,
            DimProductGravity[RecommendedZone] = 
                SWITCH(
                    TRUE(),
                    RELATED(DimGeography[LocationName]) LIKE "%Zone-A%", "High",
                    RELATED(DimGeography[LocationName]) LIKE "%Zone-B%", "Medium",
                    RELATED(DimGeography[LocationName]) LIKE "%Zone-C%", "Low",
                    "Unknown"
                )
        )
    )
VAR TotalOperations = COUNTROWS(FactWarehouseOperations)
RETURN
    DIVIDE(CorrectlyPlaced, TotalOperations, 0) * 100

// Predictive Bottleneck Index
Bottleneck Risk Index = 
VAR CurrentCycleTime = AVERAGE(FactWarehouseOperations[CycleTimeSeconds])
VAR HistoricalP90 = 
    CALCULATE(
        PERCENTILE.INC(FactWarehouseOperations[CycleTimeSeconds], 0.90),
        DATESINPERIOD(DimTime[DateValue], MAX(DimTime[DateValue]), -30, DAY)
    )
VAR RiskScore = DIVIDE(CurrentCycleTime, HistoricalP90, 1) * 100
RETURN
    MIN(RiskScore, 100) // Cap at 100
```

### Power BI Report Filters (Row-Level Security)

```dax
// Create RLS role for warehouse managers (only see their location)
[LocationCode] = USERNAME()

// Create RLS role for regional directors (see entire region)
LOOKUPVALUE(
    DimGeography[Region],
    DimGeography[GeographyKey],
    [GeographyKey]
) = LOOKUPVALUE(
    UserRegionMapping[Region],
    UserRegionMapping[UserEmail],
    USERNAME()
)
```

## Automated Alerting System

### SQL Server Agent Job for KPI Breach Detection

```sql
-- Create alert monitoring stored procedure
CREATE PROCEDURE sp_MonitorKPIBreaches
AS
BEGIN
    DECLARE @AlertBody NVARCHAR(MAX) = '';
    
    -- Check for excessive idle time
    IF EXISTS (
        SELECT 1 
        FROM FactFleetTrips
        WHERE TimeKey IN (
            SELECT TimeKey FROM DimTime 
            WHERE DateValue = CAST(GETDATE() AS DATE)
        )
        GROUP BY VehicleID
        HAVING AVG(CAST(IdleTimeMinutes AS FLOAT) / DurationMinutes) > 0.15
    )
    BEGIN
        SET @AlertBody += 'ALERT: Fleet idle time exceeds 15% threshold. ';
    END
    
    -- Check for gravity zone non-compliance
    IF EXISTS (
        SELECT 1
        FROM vw_GravityZoneOptimization
        WHERE OptimizationAction <> 'Optimally Placed'
        HAVING COUNT(*) > 50
    )
    BEGIN
        SET @AlertBody += 'ALERT: 50+ products in suboptimal gravity zones. ';
    END
    
    -- Send notification if alerts exist
    IF LEN(@AlertBody) > 0
    BEGIN
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetPulse_Alerts',
            @recipients = '${ALERT_EMAIL_RECIPIENTS}',
            @subject = 'LogiFleet Pulse KPI Breach Alert',
            @body = @AlertBody;
    END
END;
GO

-- Schedule via SQL Server Agent (every 15 minutes)
```

## Common Patterns & Troubleshooting

### Pattern: Time-Phased Dimension Queries

```sql
-- Always join to DimTime for temporal analysis
SELECT 
    dt.DateValue,
    dt.DayOfWeek,
    SUM(wo.QuantityHandled) AS TotalVolume
FROM FactWarehouseOperations wo
INNER JOIN DimTime dt ON wo.TimeKey = dt.TimeKey
WHERE dt.DateValue >= DATEADD(DAY, -7, GETDATE())
  AND dt.IsBusinessDay = 1
GROUP BY dt.DateValue, dt.DayOfWeek
ORDER BY dt.DateValue;
```

### Pattern: Handling Many-to-Many Relationships

```sql
-- Use bridge tables for route-to-warehouse relationships
CREATE TABLE Bridge_RouteWarehouse (
    RouteID VARCHAR(20),
    GeographyKey INT,
    SequenceOrder INT,
    PRIMARY KEY (RouteID, GeographyKey)
);

-- Query with bridge table
SELECT 
    ft.VehicleID,
    dg_orig.LocationName AS Origin,
    dg_dest.LocationName AS Destination,
    brw.SequenceOrder
FROM FactFleetTrips ft
INNER JOIN Bridge_RouteWarehouse brw ON ft.VehicleID = brw.RouteID
INNER JOIN DimGeography dg_orig ON ft.OriginGeographyKey = dg_orig.GeographyKey
INNER JOIN DimGeography dg_dest ON brw.GeographyKey = dg_dest.GeographyKey;
```

### Troubleshooting: Power BI Refresh Failures

**Issue**: Timeout during scheduled refresh  
**Solution**: Implement incremental refresh on large fact tables

```powerquery
// Power Query M code for incremental refresh
let
    Source = Sql.Database(
        Environment.GetEnvironmentVariable("SQL_SERVER_HOST"), 
        "LogiFleetPulse"
    ),
    FactTable = Source{[Schema="dbo",Item="FactWarehouseOperations"]}[Data],
    FilteredRows = Table.SelectRows(
        FactTable, 
        each [TimeKey] >= RangeStart and [TimeKey] < RangeEnd
    )
in
    FilteredRows
```

**Issue**: Row-level security not filtering correctly  
**Solution**: Verify RLS roles match email format

```sql
-- Check current user mapping
SELECT 
    USERNAME() AS CurrentUser,
    ur.Region,
    dg.LocationName
FROM UserRegionMapping ur
INNER JOIN DimGeography dg ON ur.Region = dg.Region
WHERE ur.UserEmail = USERNAME();
```

### Troubleshooting: Performance Optimization

```sql
-- Add columnstore index for large fact tables (10M+ rows)
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouse
ON FactWarehouseOperations (
    TimeKey, ProductKey, GeographyKey, 
    QuantityHandled, DwellTimeMinutes, CycleTimeSeconds
);

-- Partition fact tables by date for faster queries
CREATE PARTITION FUNCTION PF_FactByMonth (INT)
AS RANGE RIGHT FOR VALUES (
    20260101, 20260201, 20260301, 20260401, 20260501, 20260601
);

CREATE PARTITION SCHEME PS_FactByMonth
AS PARTITION PF_FactByMonth ALL TO ([PRIMARY]);

-- Recreate fact table on partition scheme
```

## Advanced Use Cases

### Temporal Elasticity Simulation

```sql
-- Simulate "what-if" scenarios with different capacity levels
CREATE PROCEDURE sp_SimulateCapacityImpact
    @CapacityChangePercent DECIMAL(5,2)
AS
BEGIN
    WITH BaselineMetrics AS (
        SELECT 
            AVG(IdleTimeMinutes) AS AvgIdle,
            AVG(DurationMinutes) AS AvgDuration
        FROM FactFleetTrips
        WHERE TimeKey IN (
            SELECT TimeKey FROM DimTime 
            WHERE DateValue >= DATEADD(DAY, -30, GETDATE())
        )
    )
    SELECT 
        'Baseline' AS Scenario,
        AvgIdle,
        AvgDuration,
        (AvgIdle / AvgDuration) * 100 AS IdlePercentage
    FROM BaselineMetrics
    
    UNION ALL
    
    SELECT 
        CONCAT(@CapacityChangePercent, '% Capacity Change') AS Scenario,
        AvgIdle * (1 + (@CapacityChangePercent / 100)) AS ProjectedIdle,
        AvgDuration * (1 - (@CapacityChangePercent / 200)) AS ProjectedDuration, -- Assumes efficiency gain
        (AvgIdle * (1 + (@CapacityChangePercent / 100))) / 
        (AvgDuration * (1 - (@CapacityChangePercent / 200))) * 100 AS ProjectedIdlePercentage
    FROM BaselineMetrics;
END;
GO

-- Run simulation: what if we increase capacity by 15%?
EXEC sp_SimulateCapacityImpact @CapacityChangePercent = 15;
```

## Environment Variables Reference

Required environment variables:

```bash
# SQL Server Connection
SQL_SERVER_HOST=your-server.database.windows.net
SQL_SERVER_USER=logifleet_admin
SQL_SERVER_PASSWORD=your_secure_password

# Data Source Endpoints
WMS_API_ENDPOINT=https://wms.yourcompany.com/api/v1
FLEET_TELEMETRY_ENDPOINT=https://telemetry.fleet.com/api
WEATHER_API_ENDPOINT=https://api.weather.com/v3

# Azure Storage (for Polybase)
STORAGE_ACCOUNT=yourlogifleetstorage

# Alerting
ALERT_EMAIL_RECIPIENTS=logistics-team@yourcompany.com
```

## Best Practices

1. **Always use parameterized queries** to prevent SQL injection
2. **Index foreign keys** in fact tables for join performance
3. **Partition large fact tables** by date (monthly or quarterly)
4. **Use columnstore indexes** for analytical queries on tables >10M rows
5. **Implement incremental refresh** in Power BI for fact tables
6. **Test RLS policies** in Power BI Desktop before deploying to service
7. **Schedule ETL jobs** during off-peak hours (avoid business hours)
8. **Monitor query execution plans** for expensive scans
9. **Use environment variables** for all connection strings and credentials
10. **Version control your DAX measures** and SQL scripts in Git
