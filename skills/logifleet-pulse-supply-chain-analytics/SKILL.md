---
name: logifleet-pulse-supply-chain-analytics
description: Multi-modal logistics intelligence platform using MS SQL Server star schema and Power BI for warehouse, fleet, and supply chain analytics
triggers:
  - "set up LogiFleet Pulse logistics analytics"
  - "deploy supply chain data warehouse schema"
  - "create Power BI logistics dashboard"
  - "configure warehouse and fleet analytics"
  - "implement multi-fact star schema for logistics"
  - "build real-time supply chain KPI tracking"
  - "integrate warehouse management system data"
  - "set up fleet telemetry analytics"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is an advanced logistics intelligence platform that unifies warehouse operations, fleet telemetry, inventory management, and supply chain data into a multi-fact star schema data warehouse. Built on MS SQL Server with Power BI visualization, it provides:

- **Unified semantic layer** linking warehouse, fleet, and external data sources
- **Multi-fact star schema** with time-phased dimensions for cross-functional KPIs
- **Real-time dashboards** with 15-minute refresh intervals
- **Predictive analytics** for bottleneck detection and fleet optimization
- **Warehouse Gravity Zones™** for spatial optimization based on pick frequency and item value
- **Cross-fact KPI harmonization** (e.g., inventory turnover vs. fuel consumption)

The platform ingests data from WMS, TMS, telematics feeds, supplier portals, and external APIs to create a single source of truth for logistics operations.

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- Database admin credentials
- Source system connection strings (WMS, TMS, telematics APIs)

### Database Schema Deployment

1. **Clone the repository:**

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**

```sql
-- Connect to your SQL Server instance
-- Run the schema creation script
:r schema/01_create_database.sql
:r schema/02_create_dimensions.sql
:r schema/03_create_facts.sql
:r schema/04_create_views.sql
:r schema/05_create_stored_procedures.sql
:r schema/06_create_indexes.sql
```

3. **Configure data sources:**

```json
// config.json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_SERVER_USER}",
    "password": "${SQL_SERVER_PASSWORD}",
    "port": 1433
  },
  "data_sources": {
    "wms_api": "${WMS_API_ENDPOINT}",
    "telematics_api": "${TELEMATICS_API_ENDPOINT}",
    "weather_api": "${WEATHER_API_KEY}"
  },
  "refresh_interval_minutes": 15
}
```

4. **Import Power BI template:**

```bash
# Open Power BI Desktop
# File > Import > Power BI Template
# Select LogiFleet_Pulse_Master.pbit
# Enter SQL Server connection details when prompted
```

## Core Data Model

### Key Fact Tables

**FactWarehouseOperations** - Warehouse activity metrics:

```sql
CREATE TABLE FactWarehouseOperations (
    OperationID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'Putaway', 'Pick', 'Pack', 'Ship'
    Quantity INT,
    DurationMinutes DECIMAL(10,2),
    DwellTimeHours DECIMAL(10,2),
    OperationCost DECIMAL(12,2),
    OperatorID INT,
    CONSTRAINT FK_WO_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WO_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey),
    CONSTRAINT FK_WO_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);

-- Index for performance
CREATE NONCLUSTERED INDEX IX_WO_Time_Warehouse 
ON FactWarehouseOperations(TimeKey, WarehouseKey) 
INCLUDE (ProductKey, DwellTimeHours);
```

**FactFleetTrips** - Fleet telemetry and trip data:

```sql
CREATE TABLE FactFleetTrips (
    TripID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT NOT NULL,
    TripStartTime DATETIME2,
    TripEndTime DATETIME2,
    DistanceKM DECIMAL(10,2),
    FuelLiters DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    LoadWeight DECIMAL(10,2),
    DeliveryCount INT,
    OnTimeFlag BIT,
    CONSTRAINT FK_FT_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FT_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_FT_Route FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey)
);
```

### Key Dimension Tables

**DimTime** - Time dimension with 15-minute granularity:

```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2,
    Date DATE,
    Year INT,
    Quarter INT,
    Month INT,
    Week INT,
    DayOfWeek INT,
    DayName VARCHAR(20),
    Hour INT,
    Minute15Bucket INT, -- 0, 15, 30, 45
    IsWeekend BIT,
    FiscalPeriod INT
);
```

**DimProductGravity** - Product classification with gravity score:

```sql
CREATE TABLE DimProduct (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU VARCHAR(50) UNIQUE,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    UnitWeight DECIMAL(10,2),
    UnitVolume DECIMAL(10,2),
    IsFragile BIT,
    GravityScore DECIMAL(5,2), -- Higher = higher priority placement
    VelocityClass VARCHAR(20), -- 'Fast', 'Medium', 'Slow'
    ValueTier VARCHAR(20) -- 'High', 'Medium', 'Low'
);
```

## Common SQL Queries

### Cross-Fact KPI: Dwell Time vs Fleet Idle Time

```sql
-- Correlate warehouse dwell time with fleet idle time by product category
WITH WarehouseDwell AS (
    SELECT 
        p.Category,
        AVG(wo.DwellTimeHours) AS AvgDwellHours,
        SUM(wo.Quantity) AS TotalUnits
    FROM FactWarehouseOperations wo
    JOIN DimProduct p ON wo.ProductKey = p.ProductKey
    JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.Date >= DATEADD(DAY, -30, GETDATE())
    GROUP BY p.Category
),
FleetIdle AS (
    SELECT 
        p.Category,
        AVG(ft.IdleTimeMinutes) AS AvgIdleMinutes,
        COUNT(DISTINCT ft.TripID) AS TripCount
    FROM FactFleetTrips ft
    JOIN BridgeTripProduct btp ON ft.TripID = btp.TripID
    JOIN DimProduct p ON btp.ProductKey = p.ProductKey
    JOIN DimTime t ON ft.TimeKey = t.TimeKey
    WHERE t.Date >= DATEADD(DAY, -30, GETDATE())
    GROUP BY p.Category
)
SELECT 
    wd.Category,
    wd.AvgDwellHours,
    fi.AvgIdleMinutes,
    wd.TotalUnits,
    fi.TripCount,
    -- Correlation indicator
    CASE 
        WHEN wd.AvgDwellHours > 72 AND fi.AvgIdleMinutes > 30 THEN 'High Risk'
        WHEN wd.AvgDwellHours > 48 OR fi.AvgIdleMinutes > 20 THEN 'Medium Risk'
        ELSE 'Low Risk'
    END AS RiskLevel
FROM WarehouseDwell wd
JOIN FleetIdle fi ON wd.Category = fi.Category
ORDER BY wd.AvgDwellHours DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify products that should be moved to higher gravity zones
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore AS CurrentGravity,
    COUNT(wo.OperationID) AS PickFrequency,
    AVG(wo.DurationMinutes) AS AvgPickTime,
    p.ValueTier,
    -- Calculate recommended gravity score
    (COUNT(wo.OperationID) * 0.5 + 
     CASE p.ValueTier 
         WHEN 'High' THEN 30 
         WHEN 'Medium' THEN 15 
         ELSE 5 
     END) AS RecommendedGravity,
    CASE 
        WHEN (COUNT(wo.OperationID) * 0.5 + 
              CASE p.ValueTier WHEN 'High' THEN 30 WHEN 'Medium' THEN 15 ELSE 5 END) 
             > p.GravityScore + 10 
        THEN 'RELOCATE TO HIGHER ZONE'
        ELSE 'CURRENT PLACEMENT OK'
    END AS Recommendation
FROM DimProduct p
JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE t.Date >= DATEADD(DAY, -60, GETDATE())
    AND wo.OperationType = 'Pick'
GROUP BY p.SKU, p.ProductName, p.GravityScore, p.ValueTier
HAVING COUNT(wo.OperationID) > 10
ORDER BY RecommendedGravity DESC;
```

### Predictive Bottleneck Detection

```sql
-- Stored procedure for bottleneck detection
CREATE PROCEDURE sp_DetectBottlenecks
    @ForecastHours INT = 24
AS
BEGIN
    -- Analyze patterns and predict congestion
    WITH CurrentLoad AS (
        SELECT 
            w.WarehouseName,
            w.MaxCapacity,
            SUM(wo.Quantity) AS CurrentInventory,
            COUNT(DISTINCT wo.OperationID) AS ActiveOperations,
            AVG(wo.DurationMinutes) AS AvgOperationTime
        FROM FactWarehouseOperations wo
        JOIN DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
        JOIN DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.FullDateTime >= DATEADD(HOUR, -4, GETDATE())
        GROUP BY w.WarehouseName, w.MaxCapacity
    ),
    HistoricalPattern AS (
        SELECT 
            w.WarehouseName,
            t.Hour,
            t.DayOfWeek,
            AVG(wo.Quantity) AS AvgHistoricalLoad
        FROM FactWarehouseOperations wo
        JOIN DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
        JOIN DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.Date >= DATEADD(DAY, -90, GETDATE())
            AND t.Date < DATEADD(DAY, -1, GETDATE())
        GROUP BY w.WarehouseName, t.Hour, t.DayOfWeek
    )
    SELECT 
        cl.WarehouseName,
        cl.CurrentInventory,
        cl.MaxCapacity,
        (cl.CurrentInventory * 100.0 / cl.MaxCapacity) AS CapacityUtilization,
        hp.AvgHistoricalLoad AS ExpectedLoadNextPeriod,
        cl.AvgOperationTime,
        CASE 
            WHEN (cl.CurrentInventory * 100.0 / cl.MaxCapacity) > 90 
                THEN 'CRITICAL - Immediate Action'
            WHEN (cl.CurrentInventory * 100.0 / cl.MaxCapacity) > 75 
                THEN 'WARNING - Monitor Closely'
            ELSE 'NORMAL'
        END AS AlertLevel,
        -- Predicted time to capacity
        CASE 
            WHEN hp.AvgHistoricalLoad > cl.CurrentInventory 
            THEN ((cl.MaxCapacity - cl.CurrentInventory) / 
                  (hp.AvgHistoricalLoad - cl.CurrentInventory)) * 60
            ELSE NULL
        END AS MinutesToCapacity
    FROM CurrentLoad cl
    JOIN HistoricalPattern hp ON cl.WarehouseName = hp.WarehouseName
    WHERE hp.Hour = DATEPART(HOUR, GETDATE())
        AND hp.DayOfWeek = DATEPART(WEEKDAY, GETDATE())
    ORDER BY CapacityUtilization DESC;
END;
```

## Power BI Configuration

### Connection String Setup

```powerquery
// M Query for SQL Server connection
let
    Source = Sql.Database(
        Environment.GetEnvironmentVariable("SQL_SERVER_HOST"),
        "LogiFleetPulse",
        [
            Query = "SELECT * FROM vw_WarehouseKPIDashboard"
        ]
    )
in
    Source
```

### Key DAX Measures

**Fleet Efficiency Score:**

```dax
Fleet_Efficiency_Score = 
VAR TotalDistance = SUM(FactFleetTrips[DistanceKM])
VAR TotalFuel = SUM(FactFleetTrips[FuelLiters])
VAR TotalIdleTime = SUM(FactFleetTrips[IdleTimeMinutes])
VAR TotalTripTime = 
    SUMX(
        FactFleetTrips,
        DATEDIFF(FactFleetTrips[TripStartTime], FactFleetTrips[TripEndTime], MINUTE)
    )
VAR FuelEfficiency = DIVIDE(TotalDistance, TotalFuel, 0)
VAR IdleRatio = DIVIDE(TotalIdleTime, TotalTripTime, 0)
RETURN
    (FuelEfficiency * (1 - IdleRatio)) * 100
```

**Warehouse Gravity Score:**

```dax
Warehouse_Gravity_Score = 
CALCULATE(
    AVERAGE(DimProduct[GravityScore]),
    FILTER(
        FactWarehouseOperations,
        FactWarehouseOperations[OperationType] = "Pick"
    )
)
```

**Cross-Fact KPI - Cost Per Delivery:**

```dax
Cost_Per_Delivery = 
VAR WarehouseCosts = SUM(FactWarehouseOperations[OperationCost])
VAR FleetCosts = 
    SUMX(
        FactFleetTrips,
        FactFleetTrips[FuelLiters] * 1.5 + 
        FactFleetTrips[IdleTimeMinutes] * 0.8
    )
VAR TotalDeliveries = SUM(FactFleetTrips[DeliveryCount])
RETURN
    DIVIDE(WarehouseCosts + FleetCosts, TotalDeliveries, 0)
```

## Data Loading Patterns

### Incremental ETL with Stored Procedure

```sql
-- Incremental load for warehouse operations
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Load from staging table (populated by external tool/API)
    INSERT INTO FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductKey, OperationType,
        Quantity, DurationMinutes, DwellTimeHours, OperationCost, OperatorID
    )
    SELECT 
        t.TimeKey,
        w.WarehouseKey,
        p.ProductKey,
        stg.OperationType,
        stg.Quantity,
        stg.DurationMinutes,
        DATEDIFF(HOUR, stg.OperationStart, stg.OperationEnd) AS DwellTimeHours,
        stg.OperationCost,
        stg.OperatorID
    FROM StagingWarehouseOps stg
    JOIN DimTime t ON CAST(stg.OperationStart AS DATE) = t.Date 
        AND DATEPART(HOUR, stg.OperationStart) = t.Hour
        AND (DATEPART(MINUTE, stg.OperationStart) / 15) * 15 = t.Minute15Bucket
    JOIN DimWarehouse w ON stg.WarehouseCode = w.WarehouseCode
    JOIN DimProduct p ON stg.SKU = p.SKU
    WHERE stg.OperationStart > @LastLoadDateTime
        AND stg.IsProcessed = 0;
    
    -- Mark staging records as processed
    UPDATE StagingWarehouseOps
    SET IsProcessed = 1, ProcessedDateTime = GETDATE()
    WHERE OperationStart > @LastLoadDateTime;
    
    -- Update product gravity scores based on new data
    EXEC sp_RecalculateGravityScores;
END;
```

### Real-Time Alert Configuration

```sql
-- Alert trigger for critical bottlenecks
CREATE PROCEDURE sp_CheckAndAlert
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    -- Check for critical capacity issues
    IF EXISTS (
        SELECT 1 
        FROM vw_CurrentCapacityStatus
        WHERE CapacityUtilization > 90
    )
    BEGIN
        SET @AlertMessage = (
            SELECT STRING_AGG(
                'CRITICAL: ' + WarehouseName + 
                ' at ' + CAST(CapacityUtilization AS VARCHAR(10)) + 
                '% capacity', '; '
            )
            FROM vw_CurrentCapacityStatus
            WHERE CapacityUtilization > 90
        );
        
        -- Insert into alert table (processed by external notification service)
        INSERT INTO AlertLog (AlertType, AlertMessage, AlertTime, Severity)
        VALUES ('CAPACITY_CRITICAL', @AlertMessage, GETDATE(), 'HIGH');
    END;
    
    -- Check for fleet maintenance needs
    IF EXISTS (
        SELECT 1 
        FROM FactFleetTrips ft
        JOIN DimVehicle v ON ft.VehicleKey = v.VehicleKey
        WHERE ft.TripEndTime >= DATEADD(HOUR, -1, GETDATE())
            AND (
                ft.IdleTimeMinutes > 45 OR
                (ft.FuelLiters / ft.DistanceKM) > v.MaxFuelConsumptionRate * 1.2
            )
    )
    BEGIN
        SET @AlertMessage = 'Fleet performance anomaly detected - check maintenance queue';
        INSERT INTO AlertLog (AlertType, AlertMessage, AlertTime, Severity)
        VALUES ('FLEET_ANOMALY', @AlertMessage, GETDATE(), 'MEDIUM');
    END;
END;

-- Schedule via SQL Server Agent to run every 15 minutes
```

## Common Patterns

### Pattern 1: Time-Phased Analysis

```sql
-- Compare current period vs same period last year
WITH CurrentPeriod AS (
    SELECT 
        w.WarehouseName,
        COUNT(wo.OperationID) AS Operations,
        AVG(wo.DurationMinutes) AS AvgDuration
    FROM FactWarehouseOperations wo
    JOIN DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
    JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.Date >= DATEADD(DAY, -30, GETDATE())
    GROUP BY w.WarehouseName
),
LastYearPeriod AS (
    SELECT 
        w.WarehouseName,
        COUNT(wo.OperationID) AS Operations,
        AVG(wo.DurationMinutes) AS AvgDuration
    FROM FactWarehouseOperations wo
    JOIN DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
    JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.Date >= DATEADD(DAY, -30, DATEADD(YEAR, -1, GETDATE()))
        AND t.Date < DATEADD(YEAR, -1, GETDATE())
    GROUP BY w.WarehouseName
)
SELECT 
    cp.WarehouseName,
    cp.Operations AS Current_Operations,
    ly.Operations AS LastYear_Operations,
    ((cp.Operations - ly.Operations) * 100.0 / ly.Operations) AS PercentChange_Operations,
    cp.AvgDuration AS Current_AvgDuration,
    ly.AvgDuration AS LastYear_AvgDuration,
    (cp.AvgDuration - ly.AvgDuration) AS Duration_Variance
FROM CurrentPeriod cp
JOIN LastYearPeriod ly ON cp.WarehouseName = ly.WarehouseName;
```

### Pattern 2: Bridge Table for Many-to-Many

```sql
-- Create bridge table for trip-to-product relationship
CREATE TABLE BridgeTripProduct (
    BridgeKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TripID BIGINT NOT NULL,
    ProductKey INT NOT NULL,
    Quantity INT,
    CONSTRAINT FK_BTP_Trip FOREIGN KEY (TripID) REFERENCES FactFleetTrips(TripID),
    CONSTRAINT FK_BTP_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);

-- Query using bridge table
SELECT 
    ft.TripID,
    r.RouteName,
    p.Category,
    SUM(btp.Quantity) AS TotalQuantity,
    ft.FuelLiters,
    -- Calculate fuel per unit
    ft.FuelLiters / NULLIF(SUM(btp.Quantity), 0) AS FuelPerUnit
FROM FactFleetTrips ft
JOIN BridgeTripProduct btp ON ft.TripID = btp.TripID
JOIN DimProduct p ON btp.ProductKey = p.ProductKey
JOIN DimRoute r ON ft.RouteKey = r.RouteKey
GROUP BY ft.TripID, r.RouteName, p.Category, ft.FuelLiters;
```

### Pattern 3: Role-Playing Dimensions

```sql
-- Time dimension used multiple times in same fact table
SELECT 
    wo.OperationID,
    ts.Date AS StartDate,
    ts.Hour AS StartHour,
    te.Date AS EndDate,
    te.Hour AS EndHour,
    wo.DurationMinutes
FROM FactWarehouseOperations wo
JOIN DimTime ts ON wo.StartTimeKey = ts.TimeKey
JOIN DimTime te ON wo.EndTimeKey = te.TimeKey
WHERE ts.Date = CAST(GETDATE() AS DATE);
```

## Troubleshooting

### Issue: Slow Query Performance

**Symptoms:** Queries taking >5 seconds on moderate data volumes

**Solution:**

```sql
-- Check missing indexes
SELECT 
    DatabaseName = DB_NAME(database_id),
    [schema] = OBJECT_SCHEMA_NAME(object_id, database_id),
    [table] = OBJECT_NAME(object_id, database_id),
    equality_columns,
    inequality_columns,
    included_columns
FROM sys.dm_db_missing_index_details
WHERE database_id = DB_ID('LogiFleetPulse')
ORDER BY OBJECT_NAME(object_id, database_id);

-- Add recommended indexes
CREATE NONCLUSTERED INDEX IX_WO_TimeProduct 
ON FactWarehouseOperations(TimeKey, ProductKey) 
INCLUDE (Quantity, DurationMinutes, DwellTimeHours);

-- Update statistics
UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS FactFleetTrips WITH FULLSCAN;
```

### Issue: Power BI Refresh Failures

**Symptoms:** "Data source error" or timeout messages

**Solution:**

```powerquery
// Increase query timeout in M
let
    Source = Sql.Database(
        Environment.GetEnvironmentVariable("SQL_SERVER_HOST"),
        "LogiFleetPulse",
        [
            CommandTimeout = #duration(0, 1, 0, 0), // 1 hour
            ConnectionTimeout = #duration(0, 0, 5, 0) // 5 minutes
        ]
    )
in
    Source
```

### Issue: Gravity Score Calculations Incorrect

**Symptoms:** Products with high pick frequency have low gravity scores

**Solution:**

```sql
-- Recalculate gravity scores with correct weights
UPDATE p
SET GravityScore = (
    -- Pick frequency weight (50%)
    (ISNULL(pf.PickFrequency, 0) / 100.0) * 50 +
    -- Value tier weight (30%)
    CASE p.ValueTier 
        WHEN 'High' THEN 30 
        WHEN 'Medium' THEN 15 
        ELSE 5 
    END +
    -- Fragility weight (20%)
    CASE WHEN p.IsFragile = 1 THEN 20 ELSE 0 END
)
FROM DimProduct p
LEFT JOIN (
    SELECT 
        ProductKey,
        COUNT(*) AS PickFrequency
    FROM FactWarehouseOperations
    WHERE OperationType = 'Pick'
        AND TimeKey >= (SELECT TimeKey FROM DimTime WHERE Date = DATEADD(DAY, -90, GETDATE()))
    GROUP BY ProductKey
) pf ON p.ProductKey = pf.ProductKey;
```

### Issue: Duplicate Records in Fact Tables

**Symptoms:** KPIs showing inflated values

**Solution:**

```sql
-- Find duplicates
WITH Duplicates AS (
    SELECT 
        TimeKey, WarehouseKey, ProductKey, OperationType,
        COUNT(*) AS DuplicateCount
    FROM FactWarehouseOperations
    GROUP BY TimeKey, WarehouseKey, ProductKey, OperationType
    HAVING COUNT(*) > 1
)
SELECT * FROM Duplicates;

-- Remove duplicates (keep most recent)
WITH CTE AS (
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY TimeKey, WarehouseKey, ProductKey, OperationType 
            ORDER BY OperationID DESC
        ) AS rn
    FROM FactWarehouseOperations
)
DELETE FROM CTE WHERE rn > 1;

-- Prevent future duplicates with unique constraint
CREATE UNIQUE NONCLUSTERED INDEX UX_WO_NoDuplicates
ON FactWarehouseOperations(TimeKey, WarehouseKey, ProductKey, OperationType, OperatorID);
```

## Integration Examples

### Connecting to External WMS via API

```sql
-- External table for API integration (requires PolyBase)
CREATE EXTERNAL DATA SOURCE WMS_API
WITH (
    TYPE = REST,
    LOCATION = '${WMS_API_ENDPOINT}'
);

-- Stored procedure to pull and stage data
CREATE PROCEDURE sp_ImportFromWMS
AS
BEGIN
    -- Call external API and insert into staging
    EXEC sp_invoke_external_rest_endpoint
        @url = '${WMS_API_ENDPOINT}/operations',
        @method = 'GET',
        @headers = '{"Authorization": "Bearer ${WMS_API_TOKEN}"}',
        @payload = NULL,
        @timeout = 300;
    
    -- Process JSON response into staging table
    -- (Implementation depends on WMS API structure)
END;
```

### Power Automate Integration

```json
{
  "trigger": {
    "type": "Recurrence",
    "recurrence": {
      "frequency": "Minute",
      "interval": 15
    }
  },
  "actions": {
    "Execute_SQL_Procedure": {
      "type": "SqlServer",
      "inputs": {
        "server": "${SQL_SERVER_HOST}",
        "database": "LogiFleetPulse",
        "procedure": "sp_CheckAndAlert",
        "authentication": {
          "type": "SQL",
          "username": "${SQL_SERVER_USER}",
          "password": "${SQL_SERVER_PASSWORD}"
        }
      }
    },
    "Send_Teams_Notification": {
      "type": "MicrosoftTeams",
      "inputs": {
        "message": "@{outputs('Execute_SQL_Procedure')['AlertMessage']}"
      },
      "runAfter": {
        "Execute_SQL_Procedure": ["Succeeded"]
      }
    }
  }
}
```

## Best Practices

1. **Partition large fact tables** by date for improved query performance
2. **Use incremental loads** with watermark columns instead of full refreshes
3. **Implement row-level security** in Power BI for multi-tenant scenarios
4. **Create indexed views** for frequently accessed aggregations
5. **Schedule maintenance windows** for index rebuilds (weekly recommended)
6. **Monitor query execution plans** and optimize high-cost operations
7. **Use columnstore indexes** for analytical queries on fact tables >10M rows
8. **Document custom DAX measures** with comments explaining business logic
