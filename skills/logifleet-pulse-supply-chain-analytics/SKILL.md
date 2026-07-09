---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehousing platform for supply chain analytics with multi-fact star schema
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "deploy logistics data warehouse with Power BI"
  - "configure multi-fact star schema for fleet management"
  - "implement warehouse gravity zones analytics"
  - "build cross-modal supply chain dashboard"
  - "create logistics KPI harmonization model"
  - "set up real-time fleet telemetry dashboard"
  - "deploy adaptive supply chain intelligence platform"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive logistics intelligence platform that combines:
- **Multi-fact star schema** data warehouse for warehouse operations, fleet telemetry, and cross-dock activities
- **MS SQL Server backend** with time-phased dimensions and bridge tables
- **Power BI dashboards** for real-time supply chain visualization
- **Predictive analytics** for bottleneck detection and fleet optimization
- **Warehouse Gravity Zones™** for spatial optimization based on pick frequency and value

The platform unifies disparate logistics data (WMS, telematics, supplier portals, external APIs) into a single semantic layer for cross-fact KPI analysis.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to your warehouse management system (WMS) and fleet telematics APIs

### Step 1: Clone the Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### Step 2: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance and create database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute the schema deployment script
-- This creates all dimension and fact tables
:r schema/01_create_dimensions.sql
:r schema/02_create_facts.sql
:r schema/03_create_views.sql
:r schema/04_create_procedures.sql
:r schema/05_create_indexes.sql
```

### Step 3: Configure Data Sources

Update `config.json` with your connection details:

```json
{
  "sql_server": {
    "server": "localhost",
    "database": "LogiFleetPulse",
    "trusted_connection": true
  },
  "data_sources": {
    "wms_api": "${WMS_API_ENDPOINT}",
    "telematics_api": "${FLEET_TELEMETRY_API}",
    "weather_api": "${WEATHER_API_ENDPOINT}"
  },
  "refresh_interval_minutes": 15,
  "alert_thresholds": {
    "fleet_idle_percentage": 15,
    "dwell_time_hours": 72,
    "temperature_deviation_celsius": 2
  }
}
```

### Step 4: Import Power BI Template

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. Enter your SQL Server connection string when prompted
3. Configure row-level security based on user roles
4. Publish to Power BI Service for team access

## Core Data Model

### Dimension Tables

```sql
-- DimTime: 15-minute granularity time dimension
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    TimeStamp DATETIME2 NOT NULL,
    Hour INT,
    DayOfWeek VARCHAR(10),
    FiscalPeriod VARCHAR(20),
    IsBusinessHour BIT
);

-- DimGeography: Hierarchical location dimension
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    Continent VARCHAR(50),
    Country VARCHAR(50),
    Region VARCHAR(100),
    WarehouseCode VARCHAR(20),
    RouteNodeID VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6)
);

-- DimProductGravity: Product classification with gravity score
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    GravityScore DECIMAL(5,2), -- Composite: velocity + value + fragility
    IsPerishable BIT,
    DefaultStorageZone VARCHAR(20)
);

-- DimSupplierReliability: Supplier performance metrics
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierCode VARCHAR(50) NOT NULL UNIQUE,
    SupplierName VARCHAR(200),
    LeadTimeVarianceDays DECIMAL(5,2),
    DefectPercentage DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2)
);
```

### Fact Tables

```sql
-- FactWarehouseOperations: Core warehouse activity tracking
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    OperationType VARCHAR(20), -- 'RECEIVING', 'PUTAWAY', 'PICKING', 'PACKING', 'SHIPPING'
    Quantity INT,
    DurationMinutes INT,
    DwellTimeHours DECIMAL(10,2),
    StorageZone VARCHAR(20),
    OperatorID VARCHAR(50)
);

-- FactFleetTrips: Fleet telemetry and route performance
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleID VARCHAR(50),
    RouteID VARCHAR(50),
    OriginGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadWeightKG DECIMAL(10,2),
    TirePressuePSI DECIMAL(5,2),
    EngineHealthScore DECIMAL(5,2)
);

-- FactCrossDock: Transfer operations without long-term storage
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    TransferDurationMinutes INT,
    Quantity INT
);
```

## Key SQL Procedures

### Incremental Data Loading

```sql
-- Stored procedure for incremental warehouse operations load
CREATE PROCEDURE usp_LoadWarehouseOperations
    @StartTime DATETIME2,
    @EndTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Merge from external WMS source
    MERGE FactWarehouseOperations AS target
    USING (
        SELECT 
            t.TimeKey,
            p.ProductKey,
            g.GeographyKey,
            ext.OperationType,
            ext.Quantity,
            ext.DurationMinutes,
            ext.DwellTimeHours,
            ext.StorageZone,
            ext.OperatorID
        FROM ExternalWMSData ext
        INNER JOIN DimTime t ON ext.OperationTimestamp = t.TimeStamp
        INNER JOIN DimProductGravity p ON ext.SKU = p.SKU
        INNER JOIN DimGeography g ON ext.WarehouseCode = g.WarehouseCode
        WHERE ext.OperationTimestamp BETWEEN @StartTime AND @EndTime
    ) AS source
    ON target.TimeKey = source.TimeKey 
        AND target.ProductKey = source.ProductKey
        AND target.OperationType = source.OperationType
    WHEN NOT MATCHED THEN
        INSERT (TimeKey, ProductKey, GeographyKey, OperationType, 
                Quantity, DurationMinutes, DwellTimeHours, StorageZone, OperatorID)
        VALUES (source.TimeKey, source.ProductKey, source.GeographyKey, 
                source.OperationType, source.Quantity, source.DurationMinutes,
                source.DwellTimeHours, source.StorageZone, source.OperatorID);
END;
GO
```

### Cross-Fact KPI Query

```sql
-- View: Unified logistics efficiency score
CREATE VIEW vw_LogisticsEfficiencyScore
AS
SELECT 
    t.FiscalPeriod,
    g.Region,
    p.Category,
    -- Warehouse metrics
    AVG(wo.DwellTimeHours) AS AvgDwellTimeHours,
    SUM(wo.Quantity) AS TotalUnitsProcessed,
    -- Fleet metrics
    AVG(ft.IdleTimeMinutes * 1.0 / NULLIF(ft.DistanceKM, 0)) AS IdlePerKM,
    AVG(ft.FuelConsumedLiters * 1.0 / NULLIF(ft.DistanceKM, 0)) AS FuelPerKM,
    -- Composite efficiency score (lower is better)
    (AVG(wo.DwellTimeHours) * 0.3 + 
     AVG(ft.IdleTimeMinutes) * 0.4 + 
     AVG(ft.FuelConsumedLiters) * 0.3) AS CompositeEfficiencyScore
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
LEFT JOIN FactFleetTrips ft ON ft.TimeKey = t.TimeKey AND ft.OriginGeographyKey = g.GeographyKey
GROUP BY t.FiscalPeriod, g.Region, p.Category;
GO
```

### Warehouse Gravity Zone Optimization

```sql
-- Procedure to calculate and update gravity scores
CREATE PROCEDURE usp_RecalculateGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    WITH ProductMetrics AS (
        SELECT 
            p.ProductKey,
            p.SKU,
            -- Velocity score (picks per day)
            COUNT(CASE WHEN wo.OperationType = 'PICKING' THEN 1 END) * 1.0 / 
                NULLIF(DATEDIFF(DAY, MIN(t.TimeStamp), MAX(t.TimeStamp)), 0) AS VelocityScore,
            -- Value concentration (assume higher quantity = lower unit value for gravity)
            1.0 / NULLIF(AVG(wo.Quantity), 0) AS ValueScore,
            -- Fragility (perishables get higher score)
            CASE WHEN p.IsPerishable = 1 THEN 2.0 ELSE 1.0 END AS FragilityScore
        FROM DimProductGravity p
        LEFT JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
        LEFT JOIN DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.TimeStamp >= DATEADD(MONTH, -3, GETDATE())
        GROUP BY p.ProductKey, p.SKU, p.IsPerishable
    )
    UPDATE p
    SET GravityScore = (
        ISNULL(pm.VelocityScore, 0) * 0.5 + 
        ISNULL(pm.ValueScore, 0) * 0.3 + 
        ISNULL(pm.FragilityScore, 1) * 0.2
    ) * 100 -- Scale to 0-100
    FROM DimProductGravity p
    INNER JOIN ProductMetrics pm ON p.ProductKey = pm.ProductKey;
END;
GO
```

### Automated Alerting

```sql
-- Procedure to detect and log anomalies
CREATE PROCEDURE usp_DetectAnomalies
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Create alerts table if not exists
    IF OBJECT_ID('AlertLog', 'U') IS NULL
    BEGIN
        CREATE TABLE AlertLog (
            AlertID BIGINT PRIMARY KEY IDENTITY(1,1),
            AlertTimestamp DATETIME2 DEFAULT GETDATE(),
            AlertType VARCHAR(50),
            Severity VARCHAR(20),
            Description NVARCHAR(500),
            AffectedEntity VARCHAR(100),
            IsResolved BIT DEFAULT 0
        );
    END
    
    -- Fleet idle time alerts
    INSERT INTO AlertLog (AlertType, Severity, Description, AffectedEntity)
    SELECT 
        'FLEET_IDLE_EXCESSIVE',
        'HIGH',
        'Vehicle ' + VehicleID + ' idle time is ' + 
            CAST(IdleTimeMinutes * 100.0 / NULLIF((IdleTimeMinutes + DATEDIFF(MINUTE, 0, CAST(DistanceKM / 60.0 AS TIME))), 0) AS VARCHAR(10)) + 
            '% of trip duration',
        VehicleID
    FROM FactFleetTrips ft
    INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
    WHERE t.TimeStamp >= DATEADD(HOUR, -1, GETDATE())
        AND IdleTimeMinutes * 100.0 / NULLIF((IdleTimeMinutes + DATEDIFF(MINUTE, 0, CAST(DistanceKM / 60.0 AS TIME))), 0) > 15;
    
    -- Warehouse dwell time alerts
    INSERT INTO AlertLog (AlertType, Severity, Description, AffectedEntity)
    SELECT 
        'WAREHOUSE_DWELL_EXCESSIVE',
        'MEDIUM',
        'SKU ' + p.SKU + ' has dwell time of ' + CAST(wo.DwellTimeHours AS VARCHAR(10)) + ' hours',
        p.SKU
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.TimeStamp >= DATEADD(HOUR, -1, GETDATE())
        AND wo.DwellTimeHours > 72;
END;
GO
```

## Power BI DAX Measures

### Cross-Fact KPIs

```dax
// Measure: Total Logistics Cost Index
Total Logistics Cost Index = 
VAR WarehouseCost = 
    SUMX(
        FactWarehouseOperations,
        FactWarehouseOperations[DurationMinutes] * 0.50 // $0.50 per labor minute
    )
VAR FleetCost = 
    SUMX(
        FactFleetTrips,
        FactFleetTrips[FuelConsumedLiters] * 1.20 + // $1.20 per liter
        FactFleetTrips[IdleTimeMinutes] * 0.30 // $0.30 per idle minute
    )
RETURN WarehouseCost + FleetCost

// Measure: Warehouse Efficiency Ratio
Warehouse Efficiency Ratio = 
DIVIDE(
    SUM(FactWarehouseOperations[Quantity]),
    SUM(FactWarehouseOperations[DurationMinutes]),
    0
) * 60 // Units per hour

// Measure: Fleet Utilization Score
Fleet Utilization Score = 
VAR TotalTripTime = SUM(FactFleetTrips[DistanceKM]) / 60 // Assume 60 km/h avg
VAR ActiveTime = TotalTripTime - SUM(FactFleetTrips[IdleTimeMinutes])
RETURN DIVIDE(ActiveTime, TotalTripTime, 0) * 100

// Measure: Predictive Bottleneck Score
Predictive Bottleneck Score = 
VAR HighDwellCount = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[DwellTimeHours] > 48
    )
VAR HighIdleCount = 
    CALCULATE(
        COUNTROWS(FactFleetTrips),
        FactFleetTrips[IdleTimeMinutes] > 30
    )
VAR TotalOperations = COUNTROWS(FactWarehouseOperations) + COUNTROWS(FactFleetTrips)
RETURN (HighDwellCount + HighIdleCount) * 100.0 / NULLIF(TotalOperations, 0)
```

### Time Intelligence

```dax
// Measure: Dwell Time Moving Average (7-day)
Dwell Time MA7 = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeHours]),
    DATESINPERIOD(
        DimTime[TimeStamp],
        LASTDATE(DimTime[TimeStamp]),
        -7,
        DAY
    )
)

// Measure: Fleet Fuel Efficiency Trend
Fuel Efficiency Trend = 
VAR CurrentPeriod = 
    DIVIDE(
        SUM(FactFleetTrips[DistanceKM]),
        SUM(FactFleetTrips[FuelConsumedLiters])
    )
VAR PreviousPeriod = 
    CALCULATE(
        DIVIDE(
            SUM(FactFleetTrips[DistanceKM]),
            SUM(FactFleetTrips[FuelConsumedLiters])
        ),
        DATEADD(DimTime[TimeStamp], -1, MONTH)
    )
RETURN DIVIDE(CurrentPeriod - PreviousPeriod, PreviousPeriod, 0) * 100
```

## Common Patterns

### Pattern 1: Daily Incremental Load

```sql
-- Schedule this as SQL Server Agent job to run every 15 minutes
DECLARE @LastLoadTime DATETIME2;
DECLARE @CurrentLoadTime DATETIME2 = GETDATE();

SELECT @LastLoadTime = MAX(TimeStamp) 
FROM DimTime 
WHERE TimeKey IN (SELECT TimeKey FROM FactWarehouseOperations);

EXEC usp_LoadWarehouseOperations @LastLoadTime, @CurrentLoadTime;
EXEC usp_LoadFleetTrips @LastLoadTime, @CurrentLoadTime;
EXEC usp_DetectAnomalies;
```

### Pattern 2: Custom Gravity Zone Assignment

```sql
-- Assign products to optimal storage zones based on gravity score
UPDATE p
SET DefaultStorageZone = 
    CASE 
        WHEN GravityScore >= 80 THEN 'ZONE_A_HIGH_GRAVITY'
        WHEN GravityScore >= 50 THEN 'ZONE_B_MEDIUM_GRAVITY'
        WHEN GravityScore >= 20 THEN 'ZONE_C_LOW_GRAVITY'
        ELSE 'ZONE_D_BULK_STORAGE'
    END
FROM DimProductGravity p
WHERE GravityScore IS NOT NULL;
```

### Pattern 3: Route Optimization Query

```sql
-- Find optimal routes based on historical performance
SELECT TOP 10
    RouteID,
    AVG(DistanceKM) AS AvgDistanceKM,
    AVG(FuelConsumedLiters * 1.0 / NULLIF(DistanceKM, 0)) AS AvgFuelEfficiency,
    AVG(IdleTimeMinutes) AS AvgIdleMinutes,
    COUNT(*) AS TripCount
FROM FactFleetTrips
WHERE TimeKey IN (
    SELECT TimeKey FROM DimTime WHERE TimeStamp >= DATEADD(MONTH, -1, GETDATE())
)
GROUP BY RouteID
ORDER BY AvgFuelEfficiency ASC, AvgIdleMinutes ASC;
```

## Configuration

### Row-Level Security (Power BI)

```dax
// Role: Regional Manager - sees only their region
[Region] = USERNAME()

// Role: Warehouse Supervisor - sees only their warehouse
[WarehouseCode] IN LOOKUPVALUE(
    UserWarehouseMapping[WarehouseCode],
    UserWarehouseMapping[Email],
    USERNAME()
)
```

### External Data Source Connection (Polybase)

```sql
-- Create external data source for WMS API
CREATE EXTERNAL DATA SOURCE WMS_API
WITH (
    TYPE = BLOB_STORAGE,
    LOCATION = '${WMS_API_ENDPOINT}',
    CREDENTIAL = WMS_Credential
);

-- Create external table for live WMS data
CREATE EXTERNAL TABLE ExternalWMSData (
    OperationTimestamp DATETIME2,
    SKU VARCHAR(50),
    WarehouseCode VARCHAR(20),
    OperationType VARCHAR(20),
    Quantity INT,
    DurationMinutes INT,
    DwellTimeHours DECIMAL(10,2),
    StorageZone VARCHAR(20),
    OperatorID VARCHAR(50)
)
WITH (
    LOCATION = '/wms-exports/',
    DATA_SOURCE = WMS_API,
    FILE_FORMAT = CSV_Format
);
```

## Troubleshooting

### Issue: Power BI Refresh Timeout

**Symptom**: Dashboard refresh fails with timeout error

**Solution**: Implement incremental refresh in Power BI
```m
// Power Query M - Incremental refresh parameter
let
    Source = Sql.Database(
        server, 
        database, 
        [Query="
            SELECT * FROM FactWarehouseOperations 
            WHERE TimeKey >= (SELECT TimeKey FROM DimTime WHERE TimeStamp >= DATEADD(DAY, -7, GETDATE()))
        "]
    )
in
    Source
```

### Issue: Slow Cross-Fact Queries

**Symptom**: Queries joining FactWarehouseOperations and FactFleetTrips are slow

**Solution**: Create indexed views or materialized aggregations
```sql
CREATE VIEW vw_DailyLogisticsSummary
WITH SCHEMABINDING
AS
SELECT 
    t.FiscalPeriod,
    g.Region,
    COUNT_BIG(*) AS RecordCount,
    SUM(wo.Quantity) AS TotalQuantity,
    AVG(ISNULL(ft.IdleTimeMinutes, 0)) AS AvgIdleTime
FROM dbo.FactWarehouseOperations wo
INNER JOIN dbo.DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN dbo.DimGeography g ON wo.GeographyKey = g.GeographyKey
LEFT JOIN dbo.FactFleetTrips ft ON ft.TimeKey = t.TimeKey
GROUP BY t.FiscalPeriod, g.Region;

CREATE UNIQUE CLUSTERED INDEX IX_DailyLogistics ON vw_DailyLogisticsSummary(FiscalPeriod, Region);
```

### Issue: Missing Gravity Scores

**Symptom**: New products have NULL gravity scores

**Solution**: Run recalculation procedure and set defaults
```sql
-- Set default gravity score for new products
UPDATE DimProductGravity
SET GravityScore = 50.0 -- Neutral score
WHERE GravityScore IS NULL;

-- Schedule regular recalculation
EXEC usp_RecalculateGravityScores;
```

### Issue: Alert Flooding

**Symptom**: Too many alerts generated

**Solution**: Implement alert suppression logic
```sql
ALTER PROCEDURE usp_DetectAnomalies
AS
BEGIN
    -- Only alert if same condition hasn't been alerted in last 4 hours
    INSERT INTO AlertLog (AlertType, Severity, Description, AffectedEntity)
    SELECT 'FLEET_IDLE_EXCESSIVE', 'HIGH', description, VehicleID
    FROM (
        -- detection logic here
    ) AS new_alerts
    WHERE NOT EXISTS (
        SELECT 1 FROM AlertLog
        WHERE AlertType = 'FLEET_IDLE_EXCESSIVE'
            AND AffectedEntity = new_alerts.VehicleID
            AND AlertTimestamp >= DATEADD(HOUR, -4, GETDATE())
            AND IsResolved = 0
    );
END;
```

## Best Practices

1. **Partition large fact tables** by time dimension for performance
2. **Use columnstore indexes** on fact tables for analytical queries
3. **Implement incremental refresh** in Power BI for datasets > 1GB
4. **Schedule gravity score recalculation** weekly during off-peak hours
5. **Monitor query performance** using SQL Server Query Store
6. **Validate external data sources** daily with data quality checks
7. **Document custom DAX measures** with comments explaining business logic
8. **Set up automated backups** for both SQL database and Power BI workspace
