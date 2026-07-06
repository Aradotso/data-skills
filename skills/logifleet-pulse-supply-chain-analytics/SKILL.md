---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing engine for logistics, fleet management, and supply chain analytics with multi-fact star schema
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure Power BI logistics dashboard"
  - "deploy SQL Server warehouse schema for fleet management"
  - "create supply chain KPI dashboard"
  - "implement multi-fact star schema for logistics"
  - "build real-time fleet tracking analytics"
  - "configure warehouse gravity zone modeling"
  - "set up cross-modal logistics intelligence"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

LogiFleet Pulse is a comprehensive data warehousing and business intelligence solution for logistics and supply chain management. It combines MS SQL Server for data storage with Power BI for visualization, implementing a multi-fact star schema that unifies:

- Warehouse operations (receiving, putaway, picking, packing, shipping)
- Fleet telemetry (GPS, fuel consumption, vehicle diagnostics)
- Inventory management (aging curves, dwell time, turnover)
- Supplier reliability metrics
- External data (weather, traffic) correlated with logistics events

**Key Features:**
- Multi-fact star schema with time-phased dimensions
- Cross-fact KPI harmonization (e.g., inventory vs. fleet metrics)
- Warehouse Gravity Zones™ spatial optimization
- Predictive bottleneck detection
- Real-time dashboards with 15-minute refresh cycles
- Role-based access control

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to source systems (WMS, TMS, telemetry APIs)

### Step 1: Clone Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### Step 2: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance via SSMS
-- Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute the schema deployment script
-- Located in: /sql/schema/01_create_dimensions.sql
-- Then: /sql/schema/02_create_facts.sql
-- Then: /sql/schema/03_create_views.sql
-- Then: /sql/schema/04_create_procedures.sql
```

### Step 3: Configure Data Sources

Create a configuration file for connection strings:

```json
{
  "connections": {
    "sql_server": {
      "server": "${SQL_SERVER_HOST}",
      "database": "LogiFleetPulse",
      "username": "${SQL_USER}",
      "password": "${SQL_PASSWORD}",
      "encrypt": true
    },
    "wms_api": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "api_key": "${WMS_API_KEY}"
    },
    "fleet_telemetry": {
      "endpoint": "${FLEET_API_ENDPOINT}",
      "api_key": "${FLEET_API_KEY}"
    }
  },
  "refresh_intervals": {
    "warehouse_ops": 15,
    "fleet_trips": 15,
    "supplier_data": 60
  }
}
```

### Step 4: Load Power BI Template

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. Enter SQL Server connection details when prompted
3. Configure data refresh schedule in Power BI Service

## Core SQL Schema Structure

### Dimension Tables

```sql
-- DimTime: 15-minute granularity time dimension
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    TimeSlot15Min TIME,
    HourOfDay INT,
    DayOfWeek INT,
    WeekOfYear INT,
    MonthNumber INT,
    QuarterNumber INT,
    FiscalYear INT,
    IsWeekend BIT,
    IsHoliday BIT
);

-- DimGeography: Hierarchical location dimension
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationCode NVARCHAR(50) UNIQUE NOT NULL,
    LocationName NVARCHAR(255),
    LocationType NVARCHAR(50), -- 'Warehouse', 'Distribution Center', 'Route Node'
    AddressLine1 NVARCHAR(255),
    City NVARCHAR(100),
    StateProvince NVARCHAR(100),
    Country NVARCHAR(100),
    PostalCode NVARCHAR(20),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    RegionKey INT,
    IsActive BIT DEFAULT 1
);

-- DimProductGravity: Product classification with gravity scoring
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU NVARCHAR(50) UNIQUE NOT NULL,
    ProductName NVARCHAR(255),
    ProductCategory NVARCHAR(100),
    ProductSubcategory NVARCHAR(100),
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value * fragility
    VelocityClass NVARCHAR(20), -- 'Fast', 'Medium', 'Slow'
    ValueClass NVARCHAR(20), -- 'High', 'Medium', 'Low'
    FragilityScore INT, -- 1-10 scale
    IsPerishable BIT,
    OptimalStorageZone NVARCHAR(50)
);
```

### Fact Tables

```sql
-- FactWarehouseOperations: Core warehouse activity tracking
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType NVARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OperationStartTime DATETIME,
    OperationEndTime DATETIME,
    DurationMinutes DECIMAL(10,2),
    QuantityHandled INT,
    StorageZone NVARCHAR(50),
    DwellTimeHours DECIMAL(10,2), -- Time in storage before next operation
    OperatorID NVARCHAR(50),
    EquipmentID NVARCHAR(50),
    BatchNumber NVARCHAR(50)
);

-- FactFleetTrips: Vehicle movement and telemetry
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleKey INT FOREIGN KEY REFERENCES DimVehicle(VehicleKey),
    OriginGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    TripStartTime DATETIME,
    TripEndTime DATETIME,
    DistanceKM DECIMAL(10,2),
    DurationMinutes DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    AverageSpeed DECIMAL(6,2),
    LoadWeightKG DECIMAL(10,2),
    DriverID NVARCHAR(50),
    WeatherCondition NVARCHAR(50),
    TrafficDelayMinutes DECIMAL(10,2)
);
```

## Key Stored Procedures

### Incremental Data Loading

```sql
-- Load warehouse operations incrementally
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadTime DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Default to last 24 hours if not specified
    IF @LastLoadTime IS NULL
        SET @LastLoadTime = DATEADD(HOUR, -24, GETDATE());
    
    -- Insert new operations from staging/source
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        OperationStartTime, OperationEndTime, DurationMinutes,
        QuantityHandled, StorageZone, DwellTimeHours,
        OperatorID, EquipmentID, BatchNumber
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.OperationType,
        s.StartTime,
        s.EndTime,
        DATEDIFF(MINUTE, s.StartTime, s.EndTime) AS DurationMinutes,
        s.Quantity,
        s.Zone,
        s.DwellHours,
        s.OperatorID,
        s.EquipmentID,
        s.BatchNumber
    FROM StagingWarehouseOps s
    INNER JOIN DimTime t ON CAST(s.StartTime AS DATE) = CAST(t.FullDateTime AS DATE)
        AND DATEPART(HOUR, s.StartTime) = t.HourOfDay
        AND (DATEPART(MINUTE, s.StartTime) / 15) * 15 = DATEPART(MINUTE, t.TimeSlot15Min)
    INNER JOIN DimGeography g ON s.LocationCode = g.LocationCode
    INNER JOIN DimProductGravity p ON s.SKU = p.SKU
    WHERE s.StartTime > @LastLoadTime
        AND s.IsProcessed = 0;
    
    -- Mark staging records as processed
    UPDATE StagingWarehouseOps
    SET IsProcessed = 1, ProcessedDate = GETDATE()
    WHERE StartTime > @LastLoadTime AND IsProcessed = 0;
END;
GO
```

### Calculate Gravity Scores

```sql
-- Recalculate product gravity scores based on recent activity
CREATE PROCEDURE usp_UpdateProductGravityScores
    @AnalysisDays INT = 30
AS
BEGIN
    SET NOCOUNT ON;
    
    WITH ProductMetrics AS (
        SELECT 
            p.ProductKey,
            COUNT(DISTINCT wo.OperationKey) AS PickFrequency,
            AVG(CAST(wo.DwellTimeHours AS DECIMAL(10,2))) AS AvgDwellTime,
            SUM(wo.QuantityHandled) AS TotalVolume
        FROM FactWarehouseOperations wo
        INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
        WHERE wo.OperationStartTime >= DATEADD(DAY, -@AnalysisDays, GETDATE())
            AND wo.OperationType IN ('Picking', 'Packing')
        GROUP BY p.ProductKey
    ),
    Scoring AS (
        SELECT 
            pm.ProductKey,
            CASE 
                WHEN pm.PickFrequency >= PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY pm.PickFrequency) OVER () THEN 'Fast'
                WHEN pm.PickFrequency >= PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY pm.PickFrequency) OVER () THEN 'Medium'
                ELSE 'Slow'
            END AS VelocityClass,
            -- Gravity = (velocity factor * value factor * fragility factor)
            (pm.PickFrequency * 1.0 / NULLIF(pm.AvgDwellTime, 0)) * p.FragilityScore AS GravityScore
        FROM ProductMetrics pm
        INNER JOIN DimProductGravity p ON pm.ProductKey = p.ProductKey
    )
    UPDATE p
    SET 
        p.GravityScore = s.GravityScore,
        p.VelocityClass = s.VelocityClass,
        p.OptimalStorageZone = CASE 
            WHEN s.GravityScore > 50 THEN 'Zone A - High Gravity'
            WHEN s.GravityScore > 20 THEN 'Zone B - Medium Gravity'
            ELSE 'Zone C - Low Gravity'
        END
    FROM DimProductGravity p
    INNER JOIN Scoring s ON p.ProductKey = s.ProductKey;
END;
GO
```

## Power BI DAX Measures

### Cross-Fact KPI: Fleet Cost per SKU Moved

```dax
Fleet Cost per SKU = 
DIVIDE(
    CALCULATE(
        SUM(FactFleetTrips[FuelConsumedLiters]) * 1.5, -- $1.50 per liter
        ALL(FactFleetTrips)
    ),
    CALCULATE(
        SUM(FactWarehouseOperations[QuantityHandled]),
        FactWarehouseOperations[OperationType] = "Shipping"
    ),
    0
)
```

### Predictive Bottleneck Index

```dax
Bottleneck Risk Score = 
VAR CurrentDwellTime = AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR HistoricalAvgDwell = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeHours]),
        DATEADD(DimTime[FullDateTime], -30, DAY)
    )
VAR FleetUtilization = 
    DIVIDE(
        CALCULATE(SUM(FactFleetTrips[DurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes])),
        SUM(FactFleetTrips[DurationMinutes]),
        0
    )
RETURN
    (CurrentDwellTime / HistoricalAvgDwell - 1) * 50 + (1 - FleetUtilization) * 50
```

### Temporal Elasticity Simulation

```dax
Simulated Capacity Impact = 
VAR BaseCapacity = 0.80 -- Current 80% capacity
VAR SimulatedCapacity = 0.95 -- Simulate 95%
VAR CapacityRatio = SimulatedCapacity / BaseCapacity
VAR HistoricalThroughput = 
    CALCULATE(
        SUM(FactWarehouseOperations[QuantityHandled]),
        FactWarehouseOperations[OperationType] = "Shipping"
    )
RETURN
    HistoricalThroughput * CapacityRatio * 0.92 -- 8% efficiency loss at high capacity
```

## Common Patterns

### Pattern 1: Daily Warehouse Performance Report

```sql
-- Generate daily summary with gravity zone efficiency
SELECT 
    CAST(wo.OperationStartTime AS DATE) AS OperationDate,
    p.OptimalStorageZone,
    COUNT(DISTINCT wo.OperationKey) AS TotalOperations,
    AVG(wo.DurationMinutes) AS AvgDurationMinutes,
    SUM(wo.QuantityHandled) AS TotalQuantity,
    AVG(wo.DwellTimeHours) AS AvgDwellHours,
    COUNT(DISTINCT wo.ProductKey) AS UniqueSKUs
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
WHERE wo.OperationStartTime >= DATEADD(DAY, -7, GETDATE())
GROUP BY CAST(wo.OperationStartTime AS DATE), p.OptimalStorageZone
ORDER BY OperationDate DESC, p.OptimalStorageZone;
```

### Pattern 2: Fleet Route Optimization Analysis

```sql
-- Identify routes with high idle time vs. distance
WITH RouteMetrics AS (
    SELECT 
        og.LocationName AS Origin,
        dg.LocationName AS Destination,
        COUNT(*) AS TripCount,
        AVG(ft.DistanceKM) AS AvgDistance,
        AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
        AVG(ft.FuelConsumedLiters) AS AvgFuelConsumed,
        AVG(ft.TrafficDelayMinutes) AS AvgTrafficDelay
    FROM FactFleetTrips ft
    INNER JOIN DimGeography og ON ft.OriginGeographyKey = og.GeographyKey
    INNER JOIN DimGeography dg ON ft.DestinationGeographyKey = dg.GeographyKey
    WHERE ft.TripStartTime >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY og.LocationName, dg.LocationName
)
SELECT 
    *,
    (AvgIdleTime / NULLIF(AvgDistance, 0)) AS IdlePerKM,
    (AvgFuelConsumed / NULLIF(AvgDistance, 0)) AS FuelEfficiency
FROM RouteMetrics
WHERE TripCount > 5 -- Only routes with sufficient data
ORDER BY IdlePerKM DESC;
```

### Pattern 3: Cross-Fact Correlation Query

```sql
-- Correlate warehouse dwell time with fleet delays
SELECT 
    p.ProductCategory,
    AVG(wo.DwellTimeHours) AS AvgWarehouseDwell,
    AVG(ft.TrafficDelayMinutes) AS AvgFleetDelay,
    COUNT(DISTINCT wo.OperationKey) AS WarehouseOps,
    COUNT(DISTINCT ft.TripKey) AS FleetTrips
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
INNER JOIN FactFleetTrips ft ON 
    CAST(wo.OperationEndTime AS DATE) = CAST(ft.TripStartTime AS DATE)
    AND wo.GeographyKey = ft.OriginGeographyKey
WHERE wo.OperationType = 'Shipping'
    AND wo.OperationStartTime >= DATEADD(MONTH, -1, GETDATE())
GROUP BY p.ProductCategory
HAVING COUNT(DISTINCT wo.OperationKey) > 10
ORDER BY AvgWarehouseDwell DESC;
```

## Configuration Best Practices

### Indexing Strategy

```sql
-- Composite index for time-based queries
CREATE NONCLUSTERED INDEX IX_WarehouseOps_TimeGeo
ON FactWarehouseOperations (TimeKey, GeographyKey)
INCLUDE (ProductKey, OperationType, QuantityHandled);

-- Filtered index for active shipments
CREATE NONCLUSTERED INDEX IX_WarehouseOps_ActiveShipments
ON FactWarehouseOperations (OperationStartTime, GeographyKey)
WHERE OperationType = 'Shipping' AND OperationEndTime IS NOT NULL;

-- Columnstore index for analytical queries
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactFleetTrips
ON FactFleetTrips (TripStartTime, DistanceKM, FuelConsumedLiters, IdleTimeMinutes);
```

### Row-Level Security Setup

```sql
-- Create security table
CREATE TABLE DimUserSecurity (
    UserID NVARCHAR(100) PRIMARY KEY,
    AllowedGeographyKeys NVARCHAR(MAX), -- Comma-separated list
    AllowedCategories NVARCHAR(MAX),
    SecurityRole NVARCHAR(50) -- 'User', 'Supervisor', 'Executive'
);

-- Create security function
CREATE FUNCTION fn_SecurityPredicate(@GeographyKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN
    SELECT 1 AS AccessGranted
    FROM DimUserSecurity
    WHERE UserID = USER_NAME()
        AND (
            SecurityRole = 'Executive' 
            OR @GeographyKey IN (
                SELECT value FROM STRING_SPLIT(AllowedGeographyKeys, ',')
            )
        );
GO

-- Apply security policy
CREATE SECURITY POLICY WarehouseSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(GeographyKey)
ON FactWarehouseOperations
WITH (STATE = ON);
```

### Automated Alerting

```sql
-- Create alert procedure for critical KPI breaches
CREATE PROCEDURE usp_CheckKPIAlerts
AS
BEGIN
    DECLARE @AlertThresholdIdleTime DECIMAL(5,2) = 15.0; -- 15% idle time threshold
    DECLARE @AlertThresholdDwell DECIMAL(10,2) = 72.0; -- 72 hours dwell threshold
    
    -- Check fleet idle time
    INSERT INTO AlertLog (AlertType, Severity, Message, CreatedDate)
    SELECT 
        'Fleet Idle Time',
        'High',
        'Vehicle ' + CAST(VehicleKey AS NVARCHAR) + ' exceeded ' + 
        CAST(@AlertThresholdIdleTime AS NVARCHAR) + '% idle time',
        GETDATE()
    FROM FactFleetTrips
    WHERE TripStartTime >= DATEADD(HOUR, -24, GETDATE())
        AND (IdleTimeMinutes / NULLIF(DurationMinutes, 0)) * 100 > @AlertThresholdIdleTime;
    
    -- Check warehouse dwell time
    INSERT INTO AlertLog (AlertType, Severity, Message, CreatedDate)
    SELECT 
        'Warehouse Dwell',
        'Critical',
        'SKU ' + p.SKU + ' in zone ' + wo.StorageZone + ' dwell time: ' + 
        CAST(wo.DwellTimeHours AS NVARCHAR) + ' hours',
        GETDATE()
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    WHERE wo.DwellTimeHours > @AlertThresholdDwell
        AND wo.OperationType IN ('Putaway', 'Picking')
        AND wo.OperationStartTime >= DATEADD(HOUR, -6, GETDATE());
END;
GO

-- Schedule via SQL Server Agent job (run every 15 minutes)
```

## Troubleshooting

### Issue: Power BI Dashboard Not Refreshing

**Symptoms:** Data appears stale despite scheduled refresh.

**Solutions:**
1. Verify SQL Server connection in Power BI Service
2. Check firewall rules allow Power BI gateway access
3. Validate stored procedures execute without errors:
   ```sql
   EXEC usp_LoadWarehouseOperations @LastLoadTime = '2026-01-01';
   ```
4. Review Power BI refresh history for error messages

### Issue: Slow Query Performance on Fact Tables

**Symptoms:** Dashboards take >30 seconds to load.

**Solutions:**
1. Verify indexes are in place:
   ```sql
   SELECT * FROM sys.indexes WHERE object_id = OBJECT_ID('FactWarehouseOperations');
   ```
2. Update statistics:
   ```sql
   UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
   UPDATE STATISTICS FactFleetTrips WITH FULLSCAN;
   ```
3. Implement table partitioning for large fact tables (>50M rows):
   ```sql
   -- Partition by month
   CREATE PARTITION FUNCTION PF_ByMonth (DATETIME)
   AS RANGE RIGHT FOR VALUES (
       '2026-01-01', '2026-02-01', '2026-03-01', '2026-04-01'
   );
   ```

### Issue: Gravity Score Calculation Returns NULL

**Symptoms:** Products show NULL for GravityScore after running update.

**Solutions:**
1. Check for zero-division issues in stored procedure
2. Ensure sufficient historical data (minimum 7 days)
3. Verify DwellTimeHours is populated:
   ```sql
   SELECT ProductKey, COUNT(*), AVG(DwellTimeHours)
   FROM FactWarehouseOperations
   WHERE DwellTimeHours IS NOT NULL
   GROUP BY ProductKey
   HAVING COUNT(*) < 5;
   ```

### Issue: Cross-Fact DAX Measures Return Incorrect Values

**Symptoms:** KPIs show unexpected results when filtering.

**Solutions:**
1. Verify relationship directions in Power BI model (should be one-to-many)
2. Use `CALCULATE` with explicit filter context:
   ```dax
   Corrected Measure = 
   CALCULATE(
       [Base Measure],
       REMOVEFILTERS(FactFleetTrips),
       VALUES(DimTime[FullDateTime])
   )
   ```
3. Check for circular dependencies in relationships

## Integration Examples

### Python Data Ingestion Script

```python
import pyodbc
import requests
import os
from datetime import datetime

# Connect to SQL Server
conn = pyodbc.connect(
    f"DRIVER={{ODBC Driver 17 for SQL Server}};"
    f"SERVER={os.getenv('SQL_SERVER_HOST')};"
    f"DATABASE=LogiFleetPulse;"
    f"UID={os.getenv('SQL_USER')};"
    f"PWD={os.getenv('SQL_PASSWORD')}"
)
cursor = conn.cursor()

# Fetch fleet telemetry from API
response = requests.get(
    f"{os.getenv('FLEET_API_ENDPOINT')}/trips/recent",
    headers={"Authorization": f"Bearer {os.getenv('FLEET_API_KEY')}"}
)
trips = response.json()

# Insert into staging table
for trip in trips:
    cursor.execute("""
        INSERT INTO StagingFleetTrips (
            VehicleID, OriginCode, DestinationCode,
            StartTime, EndTime, DistanceKM, FuelLiters, IdleMinutes
        ) VALUES (?, ?, ?, ?, ?, ?, ?, ?)
    """, (
        trip['vehicle_id'],
        trip['origin'],
        trip['destination'],
        datetime.fromisoformat(trip['start_time']),
        datetime.fromisoformat(trip['end_time']),
        trip['distance'],
        trip['fuel_consumed'],
        trip['idle_time']
    ))

conn.commit()
cursor.close()
conn.close()
```

### Power Automate Alert Workflow

Create a flow triggered by SQL Server Agent job completion:

1. Trigger: HTTP request from SQL job
2. Action: Query AlertLog table for new alerts
3. Condition: If Severity = 'Critical'
4. Action: Send Teams notification with alert details

```sql
-- Call from SQL Agent job
DECLARE @WebhookURL NVARCHAR(500) = '${POWER_AUTOMATE_WEBHOOK_URL}';
EXEC msdb.dbo.sp_send_dbmail
    @profile_name = 'LogiFleetAlerts',
    @recipients = 'logistics@company.com',
    @subject = 'Critical Alert: High Dwell Time Detected',
    @body = 'Check AlertLog table for details.';
```

This skill provides comprehensive coverage for deploying and utilizing LogiFleet Pulse for advanced supply chain analytics.
