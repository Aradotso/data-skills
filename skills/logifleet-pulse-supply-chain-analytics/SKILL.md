---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics analytics platform for warehouse operations, fleet management, and supply chain intelligence
triggers:
  - "set up LogiFleet Pulse analytics"
  - "configure supply chain dashboard with Power BI"
  - "implement warehouse gravity zones"
  - "create multi-fact star schema for logistics"
  - "build fleet telemetry dashboard"
  - "design logistics data warehouse"
  - "integrate warehouse and fleet KPIs"
  - "deploy real-time supply chain analytics"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is an advanced MS SQL Server and Power BI data warehousing platform for logistics and supply chain analytics. It combines warehouse operations, fleet telemetry, inventory management, and external data sources into a unified semantic layer using a multi-fact star schema architecture.

**Key Capabilities:**
- Multi-fact star schema with time-phased dimensions
- Warehouse gravity zone optimization
- Cross-fact KPI harmonization (warehouse + fleet metrics)
- Real-time dashboarding with 15-minute refresh cycles
- Predictive bottleneck detection
- Fleet maintenance triage scoring
- Role-based access control with row-level security

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio

### Setup Steps

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy SQL Schema:**
```sql
-- Connect to your SQL Server instance via SSMS
-- Open and execute: schema/01_create_database.sql
-- Then execute in order:
-- schema/02_create_dimensions.sql
-- schema/03_create_facts.sql
-- schema/04_create_views.sql
-- schema/05_create_stored_procedures.sql
```

3. **Configure Data Sources:**
```json
// Copy config_sample.json to config.json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_USERNAME}",
    "password": "${SQL_PASSWORD}"
  },
  "data_sources": {
    "wms_connection": "${WMS_CONNECTION_STRING}",
    "telematics_api": "${TELEMATICS_API_URL}",
    "weather_api_key": "${WEATHER_API_KEY}"
  }
}
```

4. **Import Power BI Template:**
- Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
- Enter SQL Server connection details when prompted
- Refresh data model to validate connections

## Core Database Schema

### Fact Tables

**FactWarehouseOperations** - Tracks all warehouse activities:
```sql
CREATE TABLE FactWarehouseOperations (
    OperationID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    DateKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(20), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    QuantityHandled INT,
    CycleTimeMinutes DECIMAL(10,2),
    DwellTimeHours DECIMAL(10,2),
    OperatorID INT,
    ZoneID INT,
    CONSTRAINT FK_WHOps_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WHOps_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey),
    CONSTRAINT FK_WHOps_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);

-- Create clustered columnstore index for fast aggregation
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactWarehouseOps 
ON FactWarehouseOperations;
```

**FactFleetTrips** - Captures fleet telemetry and route data:
```sql
CREATE TABLE FactFleetTrips (
    TripID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    DateKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT NOT NULL,
    OriginGeographyKey INT,
    DestinationGeographyKey INT,
    TripDistanceMiles DECIMAL(10,2),
    TripDurationMinutes DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    FuelConsumedGallons DECIMAL(8,2),
    LoadWeightLbs INT,
    OnTimeDeliveryFlag BIT,
    DelayReasonCode VARCHAR(20),
    CONSTRAINT FK_FleetTrips_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FleetTrips_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_FleetTrips_Route FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey)
);
```

### Dimension Tables

**DimTime** - Time-phased dimension with 15-minute granularity:
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    TimeOfDay TIME NOT NULL,
    Hour TINYINT,
    Minute TINYINT,
    QuarterHour TINYINT, -- 0, 15, 30, 45
    IsBusinessHours BIT,
    ShiftCode VARCHAR(10), -- 'Day', 'Night', 'Swing'
    DateKey INT
);
```

**DimProductGravity** - Warehouse gravity zone assignment:
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL,
    ProductName VARCHAR(200),
    Category VARCHAR(50),
    VelocityScore DECIMAL(5,2), -- Pick frequency normalized 0-100
    ValueScore DECIMAL(5,2), -- Unit value normalized 0-100
    FragilityScore DECIMAL(5,2), -- 0 = bulk, 100 = fragile
    GravityZone VARCHAR(20), -- 'High', 'Medium', 'Low'
    OptimalStorageZoneID INT,
    LastRecalcDate DATE
);
```

## Key Stored Procedures

### Incremental Data Load

```sql
-- Load warehouse operations incrementally
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactWarehouseOperations (
        TimeKey, DateKey, WarehouseKey, ProductKey, 
        OperationType, QuantityHandled, CycleTimeMinutes, DwellTimeHours
    )
    SELECT 
        t.TimeKey,
        d.DateKey,
        w.WarehouseKey,
        p.ProductKey,
        src.OperationType,
        src.QuantityHandled,
        DATEDIFF(MINUTE, src.StartTime, src.EndTime) AS CycleTimeMinutes,
        DATEDIFF(HOUR, src.StartTime, GETDATE()) AS DwellTimeHours
    FROM ExternalWMSData.dbo.Operations src
    INNER JOIN DimTime t ON CAST(src.OperationDateTime AS TIME) = t.TimeOfDay
    INNER JOIN DimDate d ON CAST(src.OperationDateTime AS DATE) = d.FullDate
    INNER JOIN DimWarehouse w ON src.WarehouseCode = w.WarehouseCode
    INNER JOIN DimProduct p ON src.SKU = p.SKU
    WHERE src.OperationDateTime > @LastLoadDateTime;
    
    RETURN @@ROWCOUNT;
END;
```

### Calculate Warehouse Gravity Zones

```sql
CREATE PROCEDURE usp_RecalculateGravityZones
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Calculate velocity score (last 90 days pick frequency)
    WITH VelocityCalc AS (
        SELECT 
            ProductKey,
            COUNT(*) AS PickCount,
            PERCENT_RANK() OVER (ORDER BY COUNT(*)) * 100 AS VelocityScore
        FROM FactWarehouseOperations
        WHERE OperationType = 'Picking'
            AND DateKey >= CAST(CONVERT(VARCHAR(8), DATEADD(DAY, -90, GETDATE()), 112) AS INT)
        GROUP BY ProductKey
    ),
    -- Calculate value score
    ValueCalc AS (
        SELECT 
            p.ProductKey,
            PERCENT_RANK() OVER (ORDER BY p.UnitValue) * 100 AS ValueScore
        FROM DimProduct p
    )
    UPDATE dpg
    SET 
        VelocityScore = ISNULL(v.VelocityScore, 0),
        ValueScore = ISNULL(val.ValueScore, 0),
        GravityZone = CASE 
            WHEN (ISNULL(v.VelocityScore, 0) + ISNULL(val.ValueScore, 0)) / 2 >= 70 THEN 'High'
            WHEN (ISNULL(v.VelocityScore, 0) + ISNULL(val.ValueScore, 0)) / 2 >= 40 THEN 'Medium'
            ELSE 'Low'
        END,
        LastRecalcDate = CAST(GETDATE() AS DATE)
    FROM DimProductGravity dpg
    LEFT JOIN VelocityCalc v ON dpg.ProductKey = v.ProductKey
    LEFT JOIN ValueCalc val ON dpg.ProductKey = val.ProductKey;
END;
```

### Fleet Maintenance Triage Scoring

```sql
CREATE PROCEDURE usp_CalculateFleetTriageScore
AS
BEGIN
    -- Score fleet maintenance needs by revenue impact
    SELECT 
        v.VehicleID,
        v.VehicleType,
        v.MaintenanceAlertCode,
        SUM(ft.LoadWeightLbs * p.AvgValuePerLb) AS PotentialRevenueAtRisk,
        AVG(ft.OnTimeDeliveryFlag) AS OnTimeRate,
        COUNT(DISTINCT ft.RouteKey) AS ActiveRoutes,
        -- Composite triage score: revenue risk + OTD impact
        (SUM(ft.LoadWeightLbs * p.AvgValuePerLb) / 10000) 
        + ((1 - AVG(CAST(ft.OnTimeDeliveryFlag AS FLOAT))) * 100) AS TriageScore
    FROM DimVehicle v
    INNER JOIN FactFleetTrips ft ON v.VehicleKey = ft.VehicleKey
    CROSS APPLY (
        SELECT AVG(UnitValue / UnitWeight) AS AvgValuePerLb
        FROM DimProduct
    ) p
    WHERE v.MaintenanceAlertFlag = 1
        AND ft.DateKey >= CAST(CONVERT(VARCHAR(8), DATEADD(DAY, -7, GETDATE()), 112) AS INT)
    GROUP BY v.VehicleID, v.VehicleType, v.MaintenanceAlertCode
    ORDER BY TriageScore DESC;
END;
```

## Power BI Dashboard Patterns

### Cross-Fact KPI Measures

**DAX measures for unified metrics:**

```dax
// Total Warehouse-to-Fleet Cost Per Unit
Unified_Cost_Per_Unit = 
VAR WarehouseCost = 
    SUMX(
        FactWarehouseOperations,
        FactWarehouseOperations[CycleTimeMinutes] * 0.5 // $0.50/min labor
    )
VAR FleetCost = 
    SUMX(
        FactFleetTrips,
        FactFleetTrips[FuelConsumedGallons] * 3.5 // $3.50/gallon
    )
VAR TotalUnits = SUM(FactWarehouseOperations[QuantityHandled])
RETURN 
    DIVIDE(WarehouseCost + FleetCost, TotalUnits, 0)
```

```dax
// Dwell Time Impact on Fleet Idling
Dwell_To_Idle_Correlation = 
VAR HighDwellSKUs = 
    CALCULATETABLE(
        VALUES(DimProduct[SKU]),
        FactWarehouseOperations[DwellTimeHours] > 72
    )
VAR AvgIdleTime = 
    CALCULATE(
        AVERAGE(FactFleetTrips[IdleTimeMinutes]),
        FactFleetTrips[RouteKey] IN 
            CALCULATETABLE(
                VALUES(FactFleetTrips[RouteKey]),
                TREATAS(HighDwellSKUs, DimProduct[SKU])
            )
    )
RETURN AvgIdleTime
```

```dax
// Predictive Bottleneck Index
Bottleneck_Risk_Score = 
VAR CurrentCapacity = 
    CALCULATE(
        SUM(FactWarehouseOperations[QuantityHandled]),
        DimTime[IsBusinessHours] = TRUE
    )
VAR HistoricalAvg = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[QuantityHandled]),
        DATESINPERIOD(DimDate[FullDate], MAX(DimDate[FullDate]), -90, DAY)
    )
VAR FleetUtilization = 
    DIVIDE(
        CALCULATE(
            SUM(FactFleetTrips[TripDurationMinutes]),
            FactFleetTrips[OnTimeDeliveryFlag] = 1
        ),
        CALCULATE(SUM(FactFleetTrips[TripDurationMinutes]))
    )
RETURN 
    (CurrentCapacity / HistoricalAvg) * 50 + (1 - FleetUtilization) * 50
```

### Time Intelligence Patterns

```dax
// Same Period Last Year Comparison
WH_Operations_SPLY = 
CALCULATE(
    SUM(FactWarehouseOperations[QuantityHandled]),
    SAMEPERIODLASTYEAR(DimDate[FullDate])
)

// Moving Average for Smoothing
Fleet_Fuel_7Day_MA = 
AVERAGEX(
    DATESINPERIOD(
        DimDate[FullDate],
        LASTDATE(DimDate[FullDate]),
        -7,
        DAY
    ),
    [Total Fuel Consumed]
)
```

## Configuration

### Row-Level Security

Implement department-based data access:

```sql
-- Create security table
CREATE TABLE SecurityRoles (
    UserEmail VARCHAR(100) PRIMARY KEY,
    Role VARCHAR(50),
    AllowedWarehouseKeys VARCHAR(500), -- Comma-separated
    AllowedRouteKeys VARCHAR(500)
);

-- Power BI RLS DAX filter
[RLS_Warehouse_Filter] = 
VAR CurrentUser = USERPRINCIPALNAME()
VAR AllowedKeys = 
    LOOKUPVALUE(
        SecurityRoles[AllowedWarehouseKeys],
        SecurityRoles[UserEmail], CurrentUser
    )
RETURN 
    PATHCONTAINS(AllowedKeys, DimWarehouse[WarehouseKey])
```

### Automated Alert Configuration

```sql
-- Alert threshold table
CREATE TABLE AlertThresholds (
    AlertID INT PRIMARY KEY IDENTITY(1,1),
    MetricName VARCHAR(100),
    ThresholdValue DECIMAL(10,2),
    ComparisonOperator VARCHAR(10), -- '>', '<', '>=', '<=', '='
    AlertSeverity VARCHAR(20), -- 'Critical', 'Warning', 'Info'
    NotificationEmails VARCHAR(500),
    IsActive BIT DEFAULT 1
);

-- Alert monitoring procedure
CREATE PROCEDURE usp_CheckAlertThresholds
AS
BEGIN
    -- Fleet idle time alert
    INSERT INTO AlertLog (MetricName, CurrentValue, ThresholdValue, Severity)
    SELECT 
        'Fleet Idle Time Percentage',
        AVG(IdleTimeMinutes) * 100.0 / AVG(TripDurationMinutes) AS CurrentValue,
        at.ThresholdValue,
        at.AlertSeverity
    FROM FactFleetTrips ft
    CROSS JOIN AlertThresholds at
    WHERE at.MetricName = 'Fleet Idle Time Percentage'
        AND ft.DateKey = CAST(CONVERT(VARCHAR(8), GETDATE(), 112) AS INT)
    HAVING AVG(IdleTimeMinutes) * 100.0 / AVG(TripDurationMinutes) > at.ThresholdValue;
    
    -- Send alerts via SQL Server Database Mail
    IF @@ROWCOUNT > 0
    BEGIN
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = 'logistics-team@company.com',
            @subject = 'LogiFleet Pulse Alert: Threshold Exceeded',
            @body = 'Fleet idle time has exceeded configured threshold. Check dashboard for details.';
    END;
END;
```

## Common Query Patterns

### Cross-Fact Analysis

**Find products with high dwell time that caused fleet delays:**

```sql
SELECT 
    p.SKU,
    p.ProductName,
    pg.GravityZone,
    AVG(wh.DwellTimeHours) AS AvgDwellHours,
    COUNT(DISTINCT ft.TripID) AS AffectedTrips,
    AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
    SUM(CASE WHEN ft.OnTimeDeliveryFlag = 0 THEN 1 ELSE 0 END) AS DelayedDeliveries
FROM FactWarehouseOperations wh
INNER JOIN DimProduct p ON wh.ProductKey = p.ProductKey
INNER JOIN DimProductGravity pg ON p.ProductKey = pg.ProductKey
INNER JOIN FactFleetTrips ft ON wh.DateKey = ft.DateKey
WHERE wh.DwellTimeHours > 72
    AND ft.DelayReasonCode IN ('WAIT_LOAD', 'DOCK_CONGESTION')
    AND wh.DateKey >= CAST(CONVERT(VARCHAR(8), DATEADD(MONTH, -1, GETDATE()), 112) AS INT)
GROUP BY p.SKU, p.ProductName, pg.GravityZone
HAVING AVG(wh.DwellTimeHours) > 96
ORDER BY DelayedDeliveries DESC;
```

### Temporal Elasticity Query

**What-if scenario: warehouse capacity increase impact:**

```sql
WITH BaselineMetrics AS (
    SELECT 
        AVG(QuantityHandled) AS AvgThroughput,
        AVG(CycleTimeMinutes) AS AvgCycleTime
    FROM FactWarehouseOperations
    WHERE DateKey >= CAST(CONVERT(VARCHAR(8), DATEADD(MONTH, -3, GETDATE()), 112) AS INT)
),
SimulatedCapacity AS (
    SELECT 
        QuantityHandled * 1.15 AS ProjectedQuantity, -- 15% increase
        CycleTimeMinutes * 1.08 AS ProjectedCycleTime -- 8% degradation assumption
    FROM FactWarehouseOperations
    WHERE DateKey >= CAST(CONVERT(VARCHAR(8), DATEADD(MONTH, -3, GETDATE()), 112) AS INT)
)
SELECT 
    b.AvgThroughput AS CurrentThroughput,
    AVG(s.ProjectedQuantity) AS ProjectedThroughput,
    b.AvgCycleTime AS CurrentCycleTime,
    AVG(s.ProjectedCycleTime) AS ProjectedCycleTime,
    (AVG(s.ProjectedQuantity) - b.AvgThroughput) / b.AvgThroughput * 100 AS ThroughputIncreasePercent,
    (AVG(s.ProjectedCycleTime) - b.AvgCycleTime) / b.AvgCycleTime * 100 AS CycleTimeDegradationPercent
FROM BaselineMetrics b
CROSS JOIN SimulatedCapacity s
GROUP BY b.AvgThroughput, b.AvgCycleTime;
```

### Route Optimization Candidate Identification

```sql
SELECT 
    r.RouteCode,
    r.OriginCity,
    r.DestinationCity,
    COUNT(DISTINCT ft.TripID) AS TotalTrips,
    AVG(ft.TripDurationMinutes) AS AvgDuration,
    AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
    AVG(ft.FuelConsumedGallons) AS AvgFuelConsumption,
    AVG(ft.IdleTimeMinutes) / NULLIF(AVG(ft.TripDurationMinutes), 0) * 100 AS IdlePercentage,
    -- Score: higher idle % + higher fuel = higher optimization potential
    (AVG(ft.IdleTimeMinutes) / NULLIF(AVG(ft.TripDurationMinutes), 0) * 100) 
    + (AVG(ft.FuelConsumedGallons) * 10) AS OptimizationScore
FROM FactFleetTrips ft
INNER JOIN DimRoute r ON ft.RouteKey = r.RouteKey
WHERE ft.DateKey >= CAST(CONVERT(VARCHAR(8), DATEADD(MONTH, -6, GETDATE()), 112) AS INT)
GROUP BY r.RouteCode, r.OriginCity, r.DestinationCity
HAVING COUNT(DISTINCT ft.TripID) >= 10
ORDER BY OptimizationScore DESC;
```

## Troubleshooting

### Performance Issues

**Slow dashboard refresh:**

1. Check columnstore index fragmentation:
```sql
SELECT 
    OBJECT_NAME(object_id) AS TableName,
    partition_number,
    state_desc,
    total_rows,
    deleted_rows
FROM sys.dm_db_column_store_row_group_physical_stats
WHERE object_id = OBJECT_ID('FactWarehouseOperations')
    AND state_desc <> 'COMPRESSED';

-- Rebuild if needed
ALTER INDEX CCI_FactWarehouseOps ON FactWarehouseOperations REORGANIZE;
```

2. Verify partition pruning is working:
```sql
-- Enable actual execution plan (Ctrl+M in SSMS)
SELECT 
    DateKey,
    SUM(QuantityHandled)
FROM FactWarehouseOperations
WHERE DateKey >= 20260601 -- Last month
GROUP BY DateKey;

-- Check for partition elimination in plan
```

**Power BI timeout errors:**

```sql
-- Increase query timeout
ALTER DATABASE LogiFleetPulse 
SET QUERY_GOVERNOR_COST_LIMIT = 0; -- Unlimited

-- Create indexed views for common aggregations
CREATE VIEW vw_DailyWarehouseMetrics
WITH SCHEMABINDING
AS
SELECT 
    DateKey,
    WarehouseKey,
    COUNT_BIG(*) AS OperationCount,
    SUM(QuantityHandled) AS TotalQuantity,
    AVG(CycleTimeMinutes) AS AvgCycleTime
FROM dbo.FactWarehouseOperations
GROUP BY DateKey, WarehouseKey;

CREATE UNIQUE CLUSTERED INDEX IX_DailyWHMetrics 
ON vw_DailyWarehouseMetrics(DateKey, WarehouseKey);
```

### Data Quality Issues

**Missing dimension lookups:**

```sql
-- Find orphaned fact records
SELECT 
    'WarehouseOperations' AS FactTable,
    COUNT(*) AS OrphanedRecords
FROM FactWarehouseOperations fo
WHERE NOT EXISTS (
    SELECT 1 FROM DimProduct p WHERE p.ProductKey = fo.ProductKey
)
UNION ALL
SELECT 
    'FleetTrips',
    COUNT(*)
FROM FactFleetTrips ft
WHERE NOT EXISTS (
    SELECT 1 FROM DimVehicle v WHERE v.VehicleKey = ft.VehicleKey
);

-- Add referential integrity checks
ALTER TABLE FactWarehouseOperations WITH CHECK
ADD CONSTRAINT CHK_ValidProduct
CHECK (ProductKey IN (SELECT ProductKey FROM DimProduct));
```

**Duplicate records in incremental load:**

```sql
-- Add unique constraint
CREATE UNIQUE NONCLUSTERED INDEX UX_WHOps_Dedup
ON FactWarehouseOperations(WarehouseKey, ProductKey, TimeKey, OperationType)
WHERE DwellTimeHours IS NOT NULL;

-- Modify load procedure to handle duplicates
MERGE FactWarehouseOperations AS target
USING StagingWarehouseOps AS source
ON target.WarehouseKey = source.WarehouseKey
    AND target.ProductKey = source.ProductKey
    AND target.TimeKey = source.TimeKey
WHEN NOT MATCHED THEN
    INSERT (TimeKey, DateKey, WarehouseKey, ProductKey, OperationType, QuantityHandled)
    VALUES (source.TimeKey, source.DateKey, source.WarehouseKey, source.ProductKey, 
            source.OperationType, source.QuantityHandled);
```

### Power BI Connection Issues

**Parameter-based connection for multi-environment support:**

```powerquery
let
    ServerName = #"SQL Server Host",
    DatabaseName = "LogiFleetPulse",
    Source = Sql.Database(ServerName, DatabaseName, [
        Query = "SELECT * FROM FactWarehouseOperations WHERE DateKey >= " & 
                Text.From(Date.AddMonths(Date.From(DateTime.LocalNow()), -6))
    ])
in
    Source
```

**Row-level security not applying:**

1. Verify user email matches Azure AD UPN:
```sql
-- Test RLS filter
EXECUTE AS USER = 'testuser@company.com';
SELECT COUNT(*) FROM FactWarehouseOperations;
REVERT;
```

2. Check Power BI service dataset permissions:
- Dataset Settings → Security → Add user email
- Ensure role matches SQL security table

## Advanced Patterns

### Streaming Data Integration

```sql
-- Create external table for real-time telemetry
CREATE EXTERNAL DATA SOURCE TelematicsStream
WITH (
    TYPE = BLOB_STORAGE,
    LOCATION = '${AZURE_BLOB_URL}',
    CREDENTIAL = TelematicsCredential
);

CREATE EXTERNAL FILE FORMAT TelematicsJSON
WITH (
    FORMAT_TYPE = JSON
);

CREATE EXTERNAL TABLE ext_RealtimeTelemetry (
    VehicleID VARCHAR(50),
    Timestamp DATETIME2,
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    Speed DECIMAL(5,2),
    FuelLevel DECIMAL(5,2)
)
WITH (
    LOCATION = '/telemetry/',
    DATA_SOURCE = TelematicsStream,
    FILE_FORMAT = TelematicsJSON
);
```

### Machine Learning Integration

```python
# Python script for predictive bottleneck modeling (executed via SQL Server ML Services)
import pandas as pd
from sklearn.ensemble import RandomForestClassifier
import pyodbc

# Connect to SQL Server
conn = pyodbc.connect(
    f"DRIVER={{ODBC Driver 17 for SQL Server}};"
    f"SERVER={os.environ['SQL_SERVER_HOST']};"
    f"DATABASE=LogiFleetPulse;"
    f"UID={os.environ['SQL_USERNAME']};"
    f"PWD={os.environ['SQL_PASSWORD']}"
)

# Load training data
query = """
    SELECT 
        DwellTimeHours,
        CycleTimeMinutes,
        QuantityHandled,
        VelocityScore,
        CASE WHEN DwellTimeHours > 96 THEN 1 ELSE 0 END AS IsBottleneck
    FROM FactWarehouseOperations fo
    INNER JOIN DimProductGravity pg ON fo.ProductKey = pg.ProductKey
    WHERE DateKey >= 20260301
"""
df = pd.read_sql(query, conn)

# Train model
X = df[['DwellTimeHours', 'CycleTimeMinutes', 'QuantityHandled', 'VelocityScore']]
y = df['IsBottleneck']
model = RandomForestClassifier(n_estimators=100)
model.fit(X, y)

# Score current operations
current_ops = pd.read_sql("SELECT * FROM vw_CurrentOperations", conn)
predictions = model.predict_proba(current_ops[X.columns])[:, 1]

# Write predictions back to SQL
current_ops['BottleneckProbability'] = predictions
current_ops[['OperationID', 'BottleneckProbability']].to_sql(
    'PredictedBottlenecks', 
    conn, 
    if_exists='replace', 
    index=False
)
```

## Maintenance Tasks

### Weekly Maintenance Checklist

```sql
-- Schedule these via SQL Server Agent jobs

-- 1. Update statistics (every Sunday 2 AM)
EXEC sp_updatestats;

-- 2. Rebuild fragmented indexes (>30% fragmentation)
DECLARE @TableName NVARCHAR(255);
DECLARE @IndexName NVARCHAR(255);
DECLARE @SQL NVARCHAR(MAX);

DECLARE index_cursor CURSOR FOR
SELECT 
    OBJECT_NAME(ips.object_id) AS TableName,
    i.name AS IndexName
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') ips
INNER JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.avg_fragmentation_in_percent > 30
    AND i.name IS NOT NULL;

OPEN index_cursor;
FETCH NEXT FROM index_cursor INTO @TableName, @IndexName;

WHILE @@FETCH_STATUS = 0
BEGIN
    SET @SQL = 'ALTER INDEX ' + @IndexName + ' ON ' + @TableName + ' REBUILD';
    EXEC sp_executesql @SQL;
    FETCH NEXT FROM index_cursor INTO @TableName, @IndexName;
END;

CLOSE index_cursor;
DEALLOCATE index_cursor;

-- 3. Archive old data (older than 2 years)
DELETE FROM FactWarehouseOperations
WHERE DateKey < CAST(CONVERT(VARCHAR(8), DATEADD(YEAR, -2, GETDATE()), 112) AS INT);

-- 4. Recalculate gravity zones
EXEC usp_RecalculateGravityZones;

-- 5. Check alert thresholds
EXEC usp_CheckAlertThresholds;
```

This skill provides comprehensive coverage of LogiFleet Pulse's data warehousing architecture, Power BI integration, and operational patterns for real-world supply chain analytics implementation.
