---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing platform for logistics intelligence with multi-fact star schema and fleet optimization
triggers:
  - set up logifleet pulse supply chain analytics
  - configure warehouse and fleet data model
  - deploy logistics intelligence dashboard
  - implement multi-fact star schema for supply chain
  - create power bi logistics visualization
  - build warehouse gravity zone optimization
  - integrate fleet telemetry with warehouse operations
  - troubleshoot logifleet pulse data model
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

LogiFleet Pulse is a multi-modal logistics intelligence platform that combines MS SQL Server data warehousing with Power BI visualization. It unifies warehouse operations, fleet telemetry, inventory management, and external data sources through a custom multi-fact star schema with time-phased dimensions. The platform enables cross-fact KPI harmonization, predictive bottleneck detection, and adaptive fleet optimization.

## Installation

### Prerequisites

- MS SQL Server 2019 or later
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio

### Clone Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### Deploy SQL Schema

1. **Connect to SQL Server** using SSMS or Azure Data Studio
2. **Create database**:

```sql
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO
```

3. **Execute schema script** (typically `schema.sql` or similar):

```sql
-- Execute the main schema deployment script
-- This creates all dimension and fact tables
```

### Configure Data Sources

Create a configuration file for connection strings:

```json
{
  "sqlServer": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_USERNAME}",
    "password": "${SQL_PASSWORD}"
  },
  "dataSources": {
    "wmsApi": "${WMS_API_ENDPOINT}",
    "telematicsApi": "${TELEMATICS_API_ENDPOINT}",
    "weatherApi": "${WEATHER_API_ENDPOINT}"
  }
}
```

## Core Data Model Structure

### Key Fact Tables

**FactWarehouseOperations** - Warehouse activity tracking:

```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    OperationType VARCHAR(50) NOT NULL, -- 'RECEIVING', 'PUTAWAY', 'PICKING', 'PACKING', 'SHIPPING'
    QuantityHandled DECIMAL(18,2),
    DwellTimeMinutes INT,
    PickRatePerHour DECIMAL(10,2),
    OperationStartTime DATETIME2,
    OperationEndTime DATETIME2,
    CONSTRAINT FK_WO_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WO_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_WO_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey)
);

CREATE NONCLUSTERED INDEX IX_FactWO_Time ON FactWarehouseOperations(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactWO_Product ON FactWarehouseOperations(ProductKey);
```

**FactFleetTrips** - Fleet and telemetry data:

```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    DriverKey INT NOT NULL,
    RouteKey INT NOT NULL,
    TripStartTime DATETIME2,
    TripEndTime DATETIME2,
    DistanceMiles DECIMAL(10,2),
    FuelConsumedGallons DECIMAL(10,2),
    IdleTimeMinutes INT,
    AverageSpeedMPH DECIMAL(5,2),
    LoadWeightLbs DECIMAL(10,2),
    OnTimeDelivery BIT,
    CONSTRAINT FK_FT_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FT_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_FT_Route FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey)
);

CREATE NONCLUSTERED INDEX IX_FactFT_Time ON FactFleetTrips(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactFT_Vehicle ON FactFleetTrips(VehicleKey);
```

### Key Dimension Tables

**DimTime** - Time-phased dimension with 15-minute granularity:

```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2,
    Date DATE,
    Year INT,
    Quarter INT,
    Month INT,
    Week INT,
    Day INT,
    Hour INT,
    Minute15Bucket INT, -- 0, 15, 30, 45
    DayOfWeek VARCHAR(20),
    IsWeekend BIT,
    FiscalYear INT,
    FiscalQuarter INT
);

CREATE NONCLUSTERED INDEX IX_DimTime_Date ON DimTime(Date);
```

**DimProductGravity** - Warehouse gravity zone assignment:

```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50) UNIQUE,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    GravityScore DECIMAL(5,2), -- 0-100, higher = faster moving
    PickFrequencyPerDay DECIMAL(10,2),
    ValuePerUnit DECIMAL(18,2),
    FragilityIndex DECIMAL(3,2), -- 0-1
    OptimalZone VARCHAR(20), -- 'HIGH_GRAVITY', 'MID_GRAVITY', 'LOW_GRAVITY'
    LastRecalculated DATETIME2
);
```

## Core SQL Queries and Patterns

### Cross-Fact KPI Harmonization

Query linking warehouse dwell time with fleet idle time:

```sql
-- Identify products with high dwell time causing fleet delays
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    AVG(wo.DwellTimeMinutes) AS AvgWarehouseDwellMinutes,
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdleMinutes,
    COUNT(DISTINCT ft.TripKey) AS AffectedTrips,
    SUM(ft.FuelConsumedGallons * 3.50) AS EstimatedIdleFuelCost -- $3.50/gallon
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
LEFT JOIN FactFleetTrips ft ON t.Date = (SELECT Date FROM DimTime WHERE TimeKey = ft.TimeKey)
WHERE 
    wo.OperationType = 'PICKING'
    AND t.Date >= DATEADD(DAY, -30, GETDATE())
    AND wo.DwellTimeMinutes > 72 -- More than 3 days
GROUP BY 
    p.SKU,
    p.ProductName,
    p.GravityScore
HAVING 
    AVG(ft.IdleTimeMinutes) > 20
ORDER BY 
    EstimatedIdleFuelCost DESC;
```

### Warehouse Gravity Zone Optimization

Recalculate and recommend gravity zone reassignments:

```sql
-- Calculate gravity scores and recommend zone changes
WITH ProductVelocity AS (
    SELECT 
        ProductKey,
        COUNT(*) AS PickFrequency,
        AVG(QuantityHandled) AS AvgQuantity,
        AVG(DwellTimeMinutes) AS AvgDwell
    FROM FactWarehouseOperations
    WHERE 
        OperationType = 'PICKING'
        AND OperationStartTime >= DATEADD(DAY, -90, GETDATE())
    GROUP BY ProductKey
),
GravityCalc AS (
    SELECT 
        p.ProductKey,
        p.SKU,
        p.ProductName,
        p.ValuePerUnit,
        p.FragilityIndex,
        pv.PickFrequency,
        pv.AvgDwell,
        -- Gravity formula: (pick frequency * value) / (dwell time * fragility penalty)
        (pv.PickFrequency * p.ValuePerUnit) / 
        (NULLIF(pv.AvgDwell, 0) * (1 + p.FragilityIndex)) AS NewGravityScore
    FROM DimProductGravity p
    INNER JOIN ProductVelocity pv ON p.ProductKey = pv.ProductKey
)
SELECT 
    ProductKey,
    SKU,
    ProductName,
    ROUND(NewGravityScore, 2) AS NewGravityScore,
    CASE 
        WHEN NewGravityScore >= 75 THEN 'HIGH_GRAVITY'
        WHEN NewGravityScore >= 40 THEN 'MID_GRAVITY'
        ELSE 'LOW_GRAVITY'
    END AS RecommendedZone,
    p.OptimalZone AS CurrentZone
FROM GravityCalc
INNER JOIN DimProductGravity p ON GravityCalc.ProductKey = p.ProductKey
WHERE 
    CASE 
        WHEN NewGravityScore >= 75 THEN 'HIGH_GRAVITY'
        WHEN NewGravityScore >= 40 THEN 'MID_GRAVITY'
        ELSE 'LOW_GRAVITY'
    END <> p.OptimalZone
ORDER BY NewGravityScore DESC;
```

### Predictive Bottleneck Detection

Identify potential operational bottlenecks:

```sql
-- Detect warehouse operations at risk of creating fleet delays
WITH HourlyMetrics AS (
    SELECT 
        t.Date,
        t.Hour,
        w.WarehouseName,
        COUNT(DISTINCT wo.OperationKey) AS OperationCount,
        AVG(wo.DwellTimeMinutes) AS AvgDwell,
        SUM(CASE WHEN wo.DwellTimeMinutes > 120 THEN 1 ELSE 0 END) AS HighDwellCount
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    INNER JOIN DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
    WHERE 
        t.Date >= DATEADD(DAY, -7, GETDATE())
    GROUP BY 
        t.Date,
        t.Hour,
        w.WarehouseName
),
BottleneckScore AS (
    SELECT 
        Date,
        Hour,
        WarehouseName,
        OperationCount,
        AvgDwell,
        HighDwellCount,
        -- Bottleneck score: high operations + high dwell + high dwell count
        (OperationCount * 0.3 + AvgDwell * 0.5 + HighDwellCount * 1.5) AS BottleneckRisk
    FROM HourlyMetrics
)
SELECT TOP 20
    Date,
    Hour,
    WarehouseName,
    OperationCount,
    ROUND(AvgDwell, 2) AS AvgDwellMinutes,
    HighDwellCount,
    ROUND(BottleneckRisk, 2) AS BottleneckRiskScore,
    CASE 
        WHEN BottleneckRisk >= 100 THEN 'CRITICAL'
        WHEN BottleneckRisk >= 60 THEN 'HIGH'
        WHEN BottleneckRisk >= 30 THEN 'MEDIUM'
        ELSE 'LOW'
    END AS RiskLevel
FROM BottleneckScore
WHERE Date >= CAST(GETDATE() AS DATE)
ORDER BY BottleneckRisk DESC;
```

### Adaptive Fleet Triage

Prioritize vehicle maintenance based on revenue impact:

```sql
-- Fleet maintenance priority queue based on load value and diagnostics
WITH FleetDiagnostics AS (
    SELECT 
        v.VehicleKey,
        v.VehicleID,
        v.VehicleType,
        v.TirePressurePSI,
        v.EngineDiagnosticCode,
        v.LastMaintenanceDate,
        AVG(ft.LoadWeightLbs) AS AvgLoadWeight,
        SUM(ft.DistanceMiles) AS TotalMilesLastWeek,
        COUNT(*) AS TripCount
    FROM DimVehicle v
    INNER JOIN FactFleetTrips ft ON v.VehicleKey = ft.VehicleKey
    INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
    WHERE 
        t.Date >= DATEADD(DAY, -7, GETDATE())
    GROUP BY 
        v.VehicleKey,
        v.VehicleID,
        v.VehicleType,
        v.TirePressurePSI,
        v.EngineDiagnosticCode,
        v.LastMaintenanceDate
),
MaintenanceScore AS (
    SELECT 
        VehicleKey,
        VehicleID,
        VehicleType,
        TirePressurePSI,
        EngineDiagnosticCode,
        AvgLoadWeight,
        TotalMilesLastWeek,
        DATEDIFF(DAY, LastMaintenanceDate, GETDATE()) AS DaysSinceLastMaintenance,
        -- Score: tire pressure penalty + engine issues + high utilization + overdue maintenance
        CASE WHEN TirePressurePSI < 30 THEN 50 ELSE 0 END +
        CASE WHEN EngineDiagnosticCode IS NOT NULL THEN 75 ELSE 0 END +
        CASE WHEN TotalMilesLastWeek > 1000 THEN 30 ELSE 0 END +
        CASE WHEN DATEDIFF(DAY, LastMaintenanceDate, GETDATE()) > 90 THEN 40 ELSE 0 END AS PriorityScore
    FROM FleetDiagnostics
)
SELECT 
    VehicleID,
    VehicleType,
    TirePressurePSI,
    EngineDiagnosticCode,
    ROUND(AvgLoadWeight, 2) AS AvgLoadWeightLbs,
    TotalMilesLastWeek,
    DaysSinceLastMaintenance,
    PriorityScore,
    CASE 
        WHEN PriorityScore >= 100 THEN 'IMMEDIATE'
        WHEN PriorityScore >= 60 THEN 'URGENT'
        WHEN PriorityScore >= 30 THEN 'SCHEDULED'
        ELSE 'ROUTINE'
    END AS MaintenancePriority
FROM MaintenanceScore
WHERE PriorityScore > 0
ORDER BY PriorityScore DESC;
```

## Power BI Integration

### Connecting to SQL Server

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Enter server details:

```
Server: ${SQL_SERVER_HOST}
Database: LogiFleetPulse
```

4. Use DirectQuery for real-time data or Import for faster performance

### DAX Measures for Cross-Fact Analysis

**Total Logistics Cost** (combines warehouse and fleet):

```dax
Total Logistics Cost = 
VAR WarehouseCost = 
    SUMX(
        FactWarehouseOperations,
        FactWarehouseOperations[DwellTimeMinutes] * 0.25  // $0.25 per minute storage
    )
VAR FleetCost = 
    SUMX(
        FactFleetTrips,
        FactFleetTrips[FuelConsumedGallons] * 3.50 +  // Fuel cost
        FactFleetTrips[IdleTimeMinutes] * 0.50  // Idle penalty
    )
RETURN WarehouseCost + FleetCost
```

**Fleet Efficiency Ratio**:

```dax
Fleet Efficiency % = 
DIVIDE(
    SUMX(FactFleetTrips, FactFleetTrips[DistanceMiles]),
    SUMX(
        FactFleetTrips,
        FactFleetTrips[DistanceMiles] + 
        (FactFleetTrips[IdleTimeMinutes] / 60 * FactFleetTrips[AverageSpeedMPH])
    ),
    0
) * 100
```

**Warehouse Gravity Compliance**:

```dax
Gravity Zone Compliance % = 
VAR TotalOperations = COUNTROWS(FactWarehouseOperations)
VAR CompliantOperations = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FILTER(
            FactWarehouseOperations,
            RELATED(DimProductGravity[OptimalZone]) = FactWarehouseOperations[AssignedZone]
        )
    )
RETURN DIVIDE(CompliantOperations, TotalOperations, 0) * 100
```

## Stored Procedures for Automation

### Incremental Data Load

```sql
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LoadDate DATE = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    IF @LoadDate IS NULL
        SET @LoadDate = CAST(GETDATE() AS DATE);
    
    BEGIN TRANSACTION;
    
    BEGIN TRY
        -- Load from staging table (populated from WMS API)
        INSERT INTO FactWarehouseOperations (
            TimeKey,
            ProductKey,
            WarehouseKey,
            OperationType,
            QuantityHandled,
            DwellTimeMinutes,
            PickRatePerHour,
            OperationStartTime,
            OperationEndTime
        )
        SELECT 
            t.TimeKey,
            p.ProductKey,
            w.WarehouseKey,
            stg.OperationType,
            stg.QuantityHandled,
            DATEDIFF(MINUTE, stg.OperationStartTime, stg.OperationEndTime) AS DwellTimeMinutes,
            CASE 
                WHEN stg.OperationType = 'PICKING' THEN
                    stg.QuantityHandled / NULLIF(DATEDIFF(MINUTE, stg.OperationStartTime, stg.OperationEndTime) / 60.0, 0)
                ELSE NULL
            END AS PickRatePerHour,
            stg.OperationStartTime,
            stg.OperationEndTime
        FROM StagingWarehouseOperations stg
        INNER JOIN DimTime t ON 
            CAST(stg.OperationStartTime AS DATE) = t.Date AND
            DATEPART(HOUR, stg.OperationStartTime) = t.Hour AND
            (DATEPART(MINUTE, stg.OperationStartTime) / 15) * 15 = t.Minute15Bucket
        INNER JOIN DimProductGravity p ON stg.SKU = p.SKU
        INNER JOIN DimWarehouse w ON stg.WarehouseID = w.WarehouseID
        WHERE 
            CAST(stg.OperationStartTime AS DATE) = @LoadDate
            AND NOT EXISTS (
                SELECT 1 
                FROM FactWarehouseOperations wo
                WHERE wo.OperationStartTime = stg.OperationStartTime
                AND wo.ProductKey = p.ProductKey
            );
        
        -- Delete loaded staging records
        DELETE FROM StagingWarehouseOperations
        WHERE CAST(OperationStartTime AS DATE) = @LoadDate;
        
        COMMIT TRANSACTION;
        
        PRINT 'Successfully loaded warehouse operations for ' + CAST(@LoadDate AS VARCHAR(10));
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        
        DECLARE @ErrorMessage NVARCHAR(4000) = ERROR_MESSAGE();
        RAISERROR(@ErrorMessage, 16, 1);
    END CATCH
END;
GO
```

### Automated Alert Generation

```sql
CREATE PROCEDURE usp_GenerateLogisticsAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Alert 1: High dwell time products
    INSERT INTO AlertQueue (AlertType, Severity, Message, CreatedAt)
    SELECT 
        'HIGH_DWELL_TIME',
        'WARNING',
        'Product ' + p.SKU + ' has average dwell time of ' + 
        CAST(AVG(wo.DwellTimeMinutes) AS VARCHAR(10)) + ' minutes (threshold: 72)',
        GETDATE()
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE 
        t.Date >= DATEADD(DAY, -1, GETDATE())
    GROUP BY p.SKU
    HAVING AVG(wo.DwellTimeMinutes) > 72;
    
    -- Alert 2: Fleet idle time exceeding threshold
    INSERT INTO AlertQueue (AlertType, Severity, Message, CreatedAt)
    SELECT 
        'FLEET_IDLE_TIME',
        'CRITICAL',
        'Vehicle ' + v.VehicleID + ' idle time: ' + 
        CAST(SUM(ft.IdleTimeMinutes) AS VARCHAR(10)) + ' minutes (threshold: 15% of trip)',
        GETDATE()
    FROM FactFleetTrips ft
    INNER JOIN DimVehicle v ON ft.VehicleKey = v.VehicleKey
    INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
    WHERE 
        t.Date >= DATEADD(DAY, -1, GETDATE())
    GROUP BY v.VehicleID
    HAVING 
        SUM(ft.IdleTimeMinutes) > (SUM(DATEDIFF(MINUTE, ft.TripStartTime, ft.TripEndTime)) * 0.15);
    
    -- Alert 3: Vehicle maintenance overdue
    INSERT INTO AlertQueue (AlertType, Severity, Message, CreatedAt)
    SELECT 
        'MAINTENANCE_OVERDUE',
        'URGENT',
        'Vehicle ' + VehicleID + ' maintenance overdue by ' + 
        CAST(DATEDIFF(DAY, LastMaintenanceDate, GETDATE()) - 90 AS VARCHAR(10)) + ' days',
        GETDATE()
    FROM DimVehicle
    WHERE 
        DATEDIFF(DAY, LastMaintenanceDate, GETDATE()) > 90
        AND IsActive = 1;
END;
GO
```

### Gravity Score Recalculation

```sql
CREATE PROCEDURE usp_RecalculateGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRANSACTION;
    
    BEGIN TRY
        -- Update gravity scores based on last 90 days
        UPDATE p
        SET 
            PickFrequencyPerDay = pv.PickFrequency / 90.0,
            GravityScore = (pv.PickFrequency * p.ValuePerUnit) / 
                           (NULLIF(pv.AvgDwell, 0) * (1 + p.FragilityIndex)),
            OptimalZone = CASE 
                WHEN (pv.PickFrequency * p.ValuePerUnit) / 
                     (NULLIF(pv.AvgDwell, 0) * (1 + p.FragilityIndex)) >= 75 
                    THEN 'HIGH_GRAVITY'
                WHEN (pv.PickFrequency * p.ValuePerUnit) / 
                     (NULLIF(pv.AvgDwell, 0) * (1 + p.FragilityIndex)) >= 40 
                    THEN 'MID_GRAVITY'
                ELSE 'LOW_GRAVITY'
            END,
            LastRecalculated = GETDATE()
        FROM DimProductGravity p
        INNER JOIN (
            SELECT 
                ProductKey,
                COUNT(*) AS PickFrequency,
                AVG(DwellTimeMinutes) AS AvgDwell
            FROM FactWarehouseOperations
            WHERE 
                OperationType = 'PICKING'
                AND OperationStartTime >= DATEADD(DAY, -90, GETDATE())
            GROUP BY ProductKey
        ) pv ON p.ProductKey = pv.ProductKey;
        
        COMMIT TRANSACTION;
        
        PRINT 'Gravity scores recalculated successfully';
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        
        DECLARE @ErrorMessage NVARCHAR(4000) = ERROR_MESSAGE();
        RAISERROR(@ErrorMessage, 16, 1);
    END CATCH
END;
GO
```

## Configuration and Environment Variables

### SQL Server Connection

```bash
# Environment variables for connection
export SQL_SERVER_HOST="your-server.database.windows.net"
export SQL_USERNAME="logifleet_user"
export SQL_PASSWORD="your-secure-password"
export SQL_DATABASE="LogiFleetPulse"
```

### External API Integration

```bash
# WMS API endpoint
export WMS_API_ENDPOINT="https://wms.yourcompany.com/api/v1"
export WMS_API_KEY="your-wms-api-key"

# Fleet telemetry endpoint
export TELEMATICS_API_ENDPOINT="https://telematics.provider.com/api"
export TELEMATICS_API_KEY="your-telematics-key"

# Weather API (for delay correlation)
export WEATHER_API_ENDPOINT="https://api.weatherapi.com/v1"
export WEATHER_API_KEY="your-weather-api-key"
```

### Power BI Gateway Configuration

For scheduled refresh, configure the Power BI On-Premises Data Gateway:

```json
{
  "dataSource": {
    "connectionDetails": {
      "server": "${SQL_SERVER_HOST}",
      "database": "LogiFleetPulse"
    },
    "credentials": {
      "credentialType": "Windows",
      "username": "${SQL_USERNAME}",
      "password": "${SQL_PASSWORD}"
    }
  },
  "refreshSchedule": {
    "frequency": "Every15Minutes",
    "enabled": true
  }
}
```

## Common Patterns and Best Practices

### Pattern 1: Time-Phased Dimension Queries

Always join fact tables through the time dimension for consistency:

```sql
-- Good: Using time dimension as bridge
SELECT 
    wo.OperationType,
    ft.VehicleID,
    COUNT(*) AS Occurrences
FROM FactWarehouseOperations wo
INNER JOIN DimTime t1 ON wo.TimeKey = t1.TimeKey
INNER JOIN FactFleetTrips ft ON 
    ft.TimeKey IN (
        SELECT TimeKey 
        FROM DimTime 
        WHERE Date = t1.Date AND Hour = t1.Hour
    )
GROUP BY wo.OperationType, ft.VehicleID;

-- Bad: Direct join without time dimension
-- This can create incorrect aggregations
```

### Pattern 2: Incremental Loading Strategy

Use watermark columns for efficient incremental loads:

```sql
-- Create watermark tracking table
CREATE TABLE ETL_Watermarks (
    TableName VARCHAR(100) PRIMARY KEY,
    LastLoadedTimestamp DATETIME2,
    LastUpdated DATETIME2 DEFAULT GETDATE()
);

-- Incremental load example
DECLARE @LastLoaded DATETIME2;
SELECT @LastLoaded = LastLoadedTimestamp 
FROM ETL_Watermarks 
WHERE TableName = 'FactWarehouseOperations';

-- Load only new/updated records
INSERT INTO FactWarehouseOperations (...)
SELECT ...
FROM StagingWarehouseOperations
WHERE ModifiedDate > @LastLoaded;

-- Update watermark
UPDATE ETL_Watermarks
SET LastLoadedTimestamp = GETDATE(),
    LastUpdated = GETDATE()
WHERE TableName = 'FactWarehouseOperations';
```

### Pattern 3: Role-Based Security

Implement row-level security for multi-tenant access:

```sql
-- Create security predicate function
CREATE FUNCTION dbo.fn_SecurityPredicate(@WarehouseKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN (
    SELECT 1 AS AccessGranted
    FROM dbo.UserWarehouseAccess
    WHERE 
        UserName = USER_NAME()
        AND WarehouseKey = @WarehouseKey
);
GO

-- Apply security policy
CREATE SECURITY POLICY WarehouseSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(WarehouseKey)
ON dbo.FactWarehouseOperations,
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(WarehouseKey)
ON dbo.DimWarehouse
WITH (STATE = ON);
GO
```

## Troubleshooting

### Issue: Slow Cross-Fact Queries

**Symptom**: Queries joining FactWarehouseOperations and FactFleetTrips take 30+ seconds

**Solution**: Ensure proper indexing and use time dimension as bridge:

```sql
-- Add covering indexes
CREATE NONCLUSTERED INDEX IX_FactWO_Time_Product_Includes
ON FactWarehouseOperations(TimeKey, ProductKey)
INCLUDE (OperationType, DwellTimeMinutes, QuantityHandled);

CREATE NONCLUSTERED INDEX IX_FactFT_Time_Vehicle_Includes
ON FactFleetTrips(TimeKey, VehicleKey)
INCLUDE (IdleTimeMinutes, FuelConsumedGallons, DistanceMiles);

-- Update statistics
UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS FactFleetTrips WITH FULLSCAN;
```

### Issue: Power BI Gateway Timeout

**Symptom**: Scheduled refresh fails with "Query timeout expired"

**Solution**: Switch to DirectQuery or optimize DAX measures:

```dax
-- Instead of this (slow):
Total Cost = SUMX(
    ALL(FactWarehouseOperations),
    [Complex Calculation]
)

-- Use this (fast):
Total Cost = 
VAR PreAgg = SUMMARIZE(
    FactWarehouseOperations,
    FactWarehouseOperations[TimeKey],
    "Cost", [Complex Calculation]
)
RETURN SUMX(PreAgg, [Cost])
```

### Issue: Gravity Score Calculation Producing Nulls

**Symptom**: DimProductGravity.GravityScore has NULL values

**Solution**: Handle division by zero and ensure data exists:

```sql
-- Fix null gravity scores
UPDATE DimProductGravity
SET GravityScore = 0
WHERE Grav
