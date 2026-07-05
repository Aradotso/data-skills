---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server and Power BI data warehousing template for logistics, fleet management, and supply chain analytics with multi-fact star schema
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure Power BI logistics dashboard"
  - "deploy SQL server warehouse schema for fleet management"
  - "implement multi-fact star schema for logistics"
  - "create supply chain KPI dashboard"
  - "integrate warehouse and fleet telemetry data"
  - "build logistics intelligence platform"
  - "query cross-modal supply chain analytics"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a data warehousing and analytics template that unifies logistics operations data across warehousing, fleet management, and supply chain processes. It provides:

- **Multi-fact star schema** for MS SQL Server with time-phased dimensions
- **Power BI dashboard templates** for real-time logistics visualization
- **Cross-fact KPI harmonization** linking warehouse operations to fleet metrics
- **Predictive analytics** for bottleneck detection and maintenance prioritization
- **Spatial modeling** with "Warehouse Gravity Zones" for optimization

The platform integrates data from WMS systems, telematics feeds, supplier portals, and external APIs into a unified semantic layer.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) recommended
- Access to source systems (WMS, TMS, telematics APIs)

### 1. Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Create a new database for LogiFleet Pulse
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute the schema deployment script
-- (Downloaded from the repository)
-- This creates all dimension and fact tables

-- Example dimension table: DimTime
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    Hour INT NOT NULL,
    Minute INT NOT NULL,
    DayOfWeek NVARCHAR(10),
    FiscalPeriod NVARCHAR(10),
    FifteenMinuteBucket INT
);

-- Example dimension: DimGeography
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    Continent NVARCHAR(50),
    Country NVARCHAR(50),
    Region NVARCHAR(50),
    WarehouseNode NVARCHAR(100),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6)
);

-- Example dimension: DimProductGravity
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU NVARCHAR(50) NOT NULL,
    ProductName NVARCHAR(200),
    Category NVARCHAR(100),
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value * fragility weight
    IsPerishable BIT,
    StorageZoneRecommendation NVARCHAR(50)
);

-- Fact table: FactWarehouseOperations
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType NVARCHAR(20), -- 'Putaway', 'Picking', 'Packing', 'Shipping'
    DwellTimeMinutes INT,
    PickRateUnitsPerHour DECIMAL(10,2),
    PackingTimeMinutes INT,
    WorkerID NVARCHAR(50)
);

-- Fact table: FactFleetTrips
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID NVARCHAR(50),
    RouteSegment NVARCHAR(100),
    FuelConsumptionLiters DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadWeightKg DECIMAL(10,2),
    DistanceKm DECIMAL(10,2),
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT
);

-- Bridge table for many-to-many relationships
CREATE TABLE BridgeRouteStorageZone (
    RouteID NVARCHAR(50),
    StorageZoneID NVARCHAR(50),
    RelationshipWeight DECIMAL(3,2) -- Proportion of route sourced from this zone
);

-- Create indexes for performance
CREATE NONCLUSTERED INDEX IX_WarehouseOps_Time ON FactWarehouseOperations(TimeKey);
CREATE NONCLUSTERED INDEX IX_WarehouseOps_Product ON FactWarehouseOperations(ProductKey);
CREATE NONCLUSTERED INDEX IX_FleetTrips_Time ON FactFleetTrips(TimeKey);
CREATE NONCLUSTERED INDEX IX_FleetTrips_Vehicle ON FactFleetTrips(VehicleID);
```

### 2. Configure Data Sources

Create a configuration file for ETL connections:

```json
{
  "connections": {
    "wms_api": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "auth_token": "${WMS_API_TOKEN}"
    },
    "telematics_feed": {
      "endpoint": "${TELEMATICS_ENDPOINT}",
      "api_key": "${TELEMATICS_API_KEY}"
    },
    "weather_api": {
      "endpoint": "https://api.weatherapi.com/v1",
      "api_key": "${WEATHER_API_KEY}"
    }
  },
  "refresh_interval_minutes": 15,
  "data_retention_days": 730
}
```

### 3. Import Power BI Template

1. Download `LogiFleet_Pulse_Master.pbit` from the repository
2. Open in Power BI Desktop
3. Enter SQL Server connection details when prompted:
   - Server: `${SQL_SERVER_HOST}`
   - Database: `LogiFleetPulse`
   - Authentication: Windows or SQL Server

```powerquery
// Power Query M code for connecting to SQL Server
let
    Source = Sql.Database(
        "${SQL_SERVER_HOST}", 
        "LogiFleetPulse",
        [Query="SELECT * FROM FactWarehouseOperations"]
    )
in
    Source
```

## Key SQL Queries & Patterns

### Cross-Fact KPI: Dwell Time vs. Fleet Idling

```sql
-- Correlate warehouse dwell time with fleet idle time for same products
WITH WarehouseDwell AS (
    SELECT 
        p.SKU,
        p.ProductName,
        AVG(wo.DwellTimeMinutes) AS AvgDwellMinutes,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(day, -30, GETDATE())
        AND wo.OperationType = 'Putaway'
    GROUP BY p.SKU, p.ProductName
),
FleetIdle AS (
    SELECT 
        p.SKU,
        AVG(ft.IdleTimeMinutes) AS AvgIdleMinutes,
        COUNT(*) AS TripCount
    FROM FactFleetTrips ft
    INNER JOIN BridgeRouteStorageZone br ON ft.RouteSegment = br.RouteID
    -- Link products to routes via storage zones (simplified)
    INNER JOIN FactWarehouseOperations wo ON wo.GeographyKey = ft.GeographyKey
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(day, -30, GETDATE())
    GROUP BY p.SKU
)
SELECT 
    wd.SKU,
    wd.ProductName,
    wd.AvgDwellMinutes,
    fi.AvgIdleMinutes,
    (wd.AvgDwellMinutes + fi.AvgIdleMinutes) AS TotalDelayMinutes
FROM WarehouseDwell wd
LEFT JOIN FleetIdle fi ON wd.SKU = fi.SKU
ORDER BY TotalDelayMinutes DESC;
```

### Warehouse Gravity Zone Analysis

```sql
-- Identify products that should be moved to higher-gravity zones
SELECT 
    p.SKU,
    p.ProductName,
    p.StorageZoneRecommendation AS CurrentZone,
    p.GravityScore,
    AVG(wo.PickRateUnitsPerHour) AS AvgPickRate,
    COUNT(*) AS PickFrequency,
    CASE 
        WHEN p.GravityScore > 7.5 AND p.StorageZoneRecommendation <> 'HighGravity'
            THEN 'RELOCATE to HighGravity'
        WHEN p.GravityScore < 3.0 AND p.StorageZoneRecommendation = 'HighGravity'
            THEN 'RELOCATE to LowGravity'
        ELSE 'OK'
    END AS RecommendedAction
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE t.FullDateTime >= DATEADD(day, -90, GETDATE())
    AND wo.OperationType = 'Picking'
GROUP BY p.SKU, p.ProductName, p.StorageZoneRecommendation, p.GravityScore
HAVING AVG(wo.PickRateUnitsPerHour) > 0
ORDER BY p.GravityScore DESC;
```

### Predictive Bottleneck Detection

```sql
-- Stored procedure for calculating bottleneck risk index
CREATE PROCEDURE sp_CalculateBottleneckIndex
    @LookbackDays INT = 7,
    @ThresholdPercentile DECIMAL(3,2) = 0.90
AS
BEGIN
    WITH TimeSegments AS (
        SELECT 
            t.TimeKey,
            t.FullDateTime,
            t.Hour,
            g.WarehouseNode,
            COUNT(wo.OperationKey) AS OperationVolume,
            AVG(wo.DwellTimeMinutes) AS AvgDwell,
            PERCENTILE_CONT(@ThresholdPercentile) 
                WITHIN GROUP (ORDER BY wo.DwellTimeMinutes) 
                OVER (PARTITION BY g.WarehouseNode) AS DwellThreshold
        FROM FactWarehouseOperations wo
        INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
        INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
        WHERE t.FullDateTime >= DATEADD(day, -@LookbackDays, GETDATE())
        GROUP BY t.TimeKey, t.FullDateTime, t.Hour, g.WarehouseNode
    )
    SELECT 
        WarehouseNode,
        Hour,
        AVG(OperationVolume) AS AvgHourlyVolume,
        AVG(AvgDwell) AS AvgDwellTime,
        MAX(DwellThreshold) AS BottleneckThreshold,
        CASE 
            WHEN AVG(AvgDwell) > MAX(DwellThreshold) 
                THEN 'HIGH RISK'
            WHEN AVG(AvgDwell) > MAX(DwellThreshold) * 0.80 
                THEN 'MODERATE RISK'
            ELSE 'LOW RISK'
        END AS BottleneckRisk
    FROM TimeSegments
    GROUP BY WarehouseNode, Hour
    ORDER BY WarehouseNode, Hour;
END;
GO

-- Execute the procedure
EXEC sp_CalculateBottleneckIndex @LookbackDays = 14, @ThresholdPercentile = 0.85;
```

### Fleet Triage Scoring

```sql
-- Calculate maintenance priority based on revenue impact
WITH FleetHealth AS (
    SELECT 
        ft.VehicleID,
        AVG(ft.FuelConsumptionLiters / NULLIF(ft.DistanceKm, 0)) AS FuelEfficiency,
        AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
        SUM(ft.DistanceKm) AS TotalDistance,
        COUNT(*) AS TripCount
    FROM FactFleetTrips ft
    INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(day, -30, GETDATE())
    GROUP BY ft.VehicleID
),
ProductValue AS (
    SELECT 
        ft.VehicleID,
        SUM(p.GravityScore * ft.LoadWeightKg) AS RevenueRiskScore
    FROM FactFleetTrips ft
    INNER JOIN FactWarehouseOperations wo ON ft.GeographyKey = wo.GeographyKey
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    GROUP BY ft.VehicleID
)
SELECT 
    fh.VehicleID,
    fh.FuelEfficiency,
    fh.AvgIdleTime,
    fh.TotalDistance,
    pv.RevenueRiskScore,
    (
        (1.0 / NULLIF(fh.FuelEfficiency, 0)) * 0.3 +
        (fh.AvgIdleTime / 60.0) * 0.2 +
        (pv.RevenueRiskScore / 1000.0) * 0.5
    ) AS MaintenancePriorityScore
FROM FleetHealth fh
LEFT JOIN ProductValue pv ON fh.VehicleID = pv.VehicleID
ORDER BY MaintenancePriorityScore DESC;
```

## Power BI DAX Measures

### Cross-Fact Composite KPI

```dax
// Total Logistics Friction Index
LogisticsFrictionIndex = 
VAR WarehouseFriction = 
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]) / 60
VAR FleetFriction = 
    AVERAGE(FactFleetTrips[IdleTimeMinutes]) / 60
RETURN
    (WarehouseFriction * 0.4) + (FleetFriction * 0.6)
```

### Time Intelligence for Trends

```dax
// Dwell Time vs. Previous Period
DwellTimeMoM = 
VAR CurrentPeriod = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR PreviousPeriod = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        DATEADD(DimTime[FullDateTime], -1, MONTH)
    )
RETURN
    DIVIDE(CurrentPeriod - PreviousPeriod, PreviousPeriod, 0)
```

### Dynamic Gravity Score Calculation

```dax
// Recalculate Gravity Score dynamically
DynamicGravityScore = 
VAR Velocity = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[OperationType] = "Picking"
    )
VAR Value = AVERAGE(DimProductGravity[GravityScore])
VAR Fragility = IF(DimProductGravity[IsPerishable] = TRUE(), 1.5, 1.0)
RETURN
    (Velocity / 10) * Value * Fragility
```

## ETL & Data Loading Patterns

### Incremental Load with Stored Procedure

```sql
-- Stored procedure for incremental warehouse operations load
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME
AS
BEGIN
    -- Insert new records from staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType, 
        DwellTimeMinutes, PickRateUnitsPerHour, PackingTimeMinutes, WorkerID
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.OperationType,
        s.DwellTimeMinutes,
        s.PickRateUnitsPerHour,
        s.PackingTimeMinutes,
        s.WorkerID
    FROM StagingWarehouseOperations s
    INNER JOIN DimTime t ON DATEADD(MINUTE, 
        (DATEDIFF(MINUTE, '2000-01-01', s.OperationDateTime) / 15) * 15, 
        '2000-01-01') = t.FullDateTime
    INNER JOIN DimGeography g ON s.WarehouseCode = g.WarehouseNode
    INNER JOIN DimProductGravity p ON s.SKU = p.SKU
    WHERE s.OperationDateTime > @LastLoadDateTime;
    
    -- Update last load timestamp
    UPDATE ETLControl 
    SET LastLoadDateTime = GETDATE()
    WHERE TableName = 'FactWarehouseOperations';
END;
GO
```

### External API Integration (Python)

```python
import pyodbc
import requests
import os
from datetime import datetime, timedelta

# Environment variables for credentials
SQL_SERVER = os.getenv('SQL_SERVER_HOST')
SQL_DB = 'LogiFleetPulse'
SQL_USER = os.getenv('SQL_SERVER_USER')
SQL_PASSWORD = os.getenv('SQL_SERVER_PASSWORD')
TELEMATICS_API = os.getenv('TELEMATICS_ENDPOINT')
TELEMATICS_KEY = os.getenv('TELEMATICS_API_KEY')

def extract_telematics_data(hours_back=1):
    """Extract fleet telemetry from external API"""
    since = (datetime.now() - timedelta(hours=hours_back)).isoformat()
    
    response = requests.get(
        f"{TELEMATICS_API}/fleet/trips",
        headers={'Authorization': f'Bearer {TELEMATICS_KEY}'},
        params={'since': since}
    )
    
    return response.json()

def load_fleet_trips(trips_data):
    """Load trip data into SQL Server"""
    conn = pyodbc.connect(
        f'DRIVER={{ODBC Driver 17 for SQL Server}};'
        f'SERVER={SQL_SERVER};DATABASE={SQL_DB};'
        f'UID={SQL_USER};PWD={SQL_PASSWORD}'
    )
    cursor = conn.cursor()
    
    for trip in trips_data:
        cursor.execute("""
            INSERT INTO FactFleetTrips (
                TimeKey, GeographyKey, VehicleID, RouteSegment,
                FuelConsumptionLiters, IdleTimeMinutes, LoadWeightKg,
                DistanceKm, LoadingTimeMinutes, UnloadingTimeMinutes
            )
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
        """, (
            get_time_key(trip['timestamp']),
            get_geography_key(trip['start_location']),
            trip['vehicle_id'],
            trip['route_segment'],
            trip['fuel_liters'],
            trip['idle_minutes'],
            trip['load_kg'],
            trip['distance_km'],
            trip['loading_minutes'],
            trip['unloading_minutes']
        ))
    
    conn.commit()
    cursor.close()
    conn.close()

def get_time_key(timestamp_str):
    """Convert timestamp to TimeKey (rounded to 15-min bucket)"""
    dt = datetime.fromisoformat(timestamp_str)
    rounded = dt.replace(minute=(dt.minute // 15) * 15, second=0, microsecond=0)
    # TimeKey format: YYYYMMDDHHMM
    return int(rounded.strftime('%Y%m%d%H%M'))

def get_geography_key(location_code):
    """Look up GeographyKey from location code"""
    # Simplified - in production, query DimGeography
    return 1  # Placeholder

# Run ETL
if __name__ == '__main__':
    trips = extract_telematics_data(hours_back=1)
    load_fleet_trips(trips)
    print(f"Loaded {len(trips)} fleet trips")
```

## Alerting Configuration

### SQL Server Automated Alerts

```sql
-- Create alert threshold table
CREATE TABLE AlertThresholds (
    AlertID INT PRIMARY KEY IDENTITY(1,1),
    MetricName NVARCHAR(100),
    ThresholdValue DECIMAL(10,2),
    ComparisonOperator NVARCHAR(10), -- '>', '<', '='
    AlertRecipients NVARCHAR(500), -- Email addresses
    IsActive BIT DEFAULT 1
);

-- Insert sample thresholds
INSERT INTO AlertThresholds (MetricName, ThresholdValue, ComparisonOperator, AlertRecipients)
VALUES 
    ('FleetIdleTimePercent', 15.0, '>', 'fleet-ops@company.com'),
    ('WarehouseDwellTimeHours', 72.0, '>', 'warehouse-mgr@company.com'),
    ('FuelEfficiency', 8.0, '<', 'fleet-ops@company.com');

-- Stored procedure to check thresholds and send alerts
CREATE PROCEDURE sp_CheckAlertsAndNotify
AS
BEGIN
    DECLARE @AlertsTriggered TABLE (
        MetricName NVARCHAR(100),
        CurrentValue DECIMAL(10,2),
        ThresholdValue DECIMAL(10,2),
        Recipients NVARCHAR(500)
    );
    
    -- Check fleet idle time
    INSERT INTO @AlertsTriggered
    SELECT 
        'FleetIdleTimePercent',
        AVG(ft.IdleTimeMinutes) * 100.0 / NULLIF(AVG(ft.IdleTimeMinutes + ft.LoadingTimeMinutes + ft.UnloadingTimeMinutes), 0),
        at.ThresholdValue,
        at.AlertRecipients
    FROM FactFleetTrips ft
    CROSS JOIN AlertThresholds at
    INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
    WHERE at.MetricName = 'FleetIdleTimePercent'
        AND at.IsActive = 1
        AND t.FullDateTime >= DATEADD(hour, -24, GETDATE())
    GROUP BY at.ThresholdValue, at.AlertRecipients
    HAVING AVG(ft.IdleTimeMinutes) * 100.0 / NULLIF(AVG(ft.IdleTimeMinutes + ft.LoadingTimeMinutes + ft.UnloadingTimeMinutes), 0) > at.ThresholdValue;
    
    -- Send alerts (integrate with SMTP or external service)
    IF EXISTS (SELECT 1 FROM @AlertsTriggered)
    BEGIN
        -- Log to alert history
        INSERT INTO AlertHistory (AlertDateTime, MetricName, CurrentValue, Recipients)
        SELECT GETDATE(), MetricName, CurrentValue, Recipients
        FROM @AlertsTriggered;
        
        -- In production, call sp_send_dbmail or external API
        SELECT * FROM @AlertsTriggered;
    END
END;
GO

-- Schedule this procedure to run every 15 minutes via SQL Server Agent
```

## Troubleshooting

### Issue: Slow Cross-Fact Queries

**Solution**: Ensure composite indexes exist on join keys

```sql
-- Create covering index for common cross-fact query pattern
CREATE NONCLUSTERED INDEX IX_WarehouseOps_Coverage
ON FactWarehouseOperations (TimeKey, GeographyKey, ProductKey)
INCLUDE (DwellTimeMinutes, OperationType);

CREATE NONCLUSTERED INDEX IX_FleetTrips_Coverage
ON FactFleetTrips (TimeKey, GeographyKey, VehicleID)
INCLUDE (IdleTimeMinutes, FuelConsumptionLiters);
```

### Issue: Power BI Refresh Failures

**Solution**: Check row-level security and query folding

```powerquery
// Ensure query folding by avoiding premature transformations
let
    Source = Sql.Database("${SQL_SERVER_HOST}", "LogiFleetPulse"),
    FactTable = Source{[Schema="dbo",Item="FactWarehouseOperations"]}[Data],
    // Filter pushed to SQL Server (query folding occurs)
    FilteredRows = Table.SelectRows(FactTable, each [DwellTimeMinutes] > 0)
in
    FilteredRows
```

### Issue: Missing Time Dimension Records

**Solution**: Populate DimTime with all 15-minute intervals

```sql
-- Populate DimTime for 2 years (105,120 rows)
DECLARE @StartDate DATETIME = '2025-01-01 00:00:00';
DECLARE @EndDate DATETIME = '2027-01-01 00:00:00';
DECLARE @Interval INT = 15; -- minutes

WHILE @StartDate < @EndDate
BEGIN
    INSERT INTO DimTime (TimeKey, FullDateTime, Hour, Minute, DayOfWeek, FiscalPeriod, FifteenMinuteBucket)
    VALUES (
        CAST(FORMAT(@StartDate, 'yyyyMMddHHmm') AS INT),
        @StartDate,
        DATEPART(HOUR, @StartDate),
        DATEPART(MINUTE, @StartDate),
        DATENAME(WEEKDAY, @StartDate),
        CONCAT('FY', YEAR(@StartDate), 'Q', DATEPART(QUARTER, @StartDate)),
        (DATEPART(HOUR, @StartDate) * 4) + (DATEPART(MINUTE, @StartDate) / 15)
    );
    
    SET @StartDate = DATEADD(MINUTE, @Interval, @StartDate);
END;
```

### Issue: Gravity Score Calculation Inaccuracies

**Solution**: Recalculate based on rolling 90-day window

```sql
-- Update gravity scores based on recent activity
UPDATE DimProductGravity
SET GravityScore = sub.NewScore
FROM DimProductGravity p
INNER JOIN (
    SELECT 
        p.ProductKey,
        (
            (COUNT(*) / 1000.0) * -- Velocity factor
            (AVG(wo.PickRateUnitsPerHour) / 10.0) * -- Efficiency factor
            CASE WHEN p.IsPerishable = 1 THEN 1.5 ELSE 1.0 END -- Fragility weight
        ) AS NewScore
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(day, -90, GETDATE())
        AND wo.OperationType = 'Picking'
    GROUP BY p.ProductKey, p.IsPerishable
) sub ON p.ProductKey = sub.ProductKey;
```

## Best Practices

1. **Partition Large Fact Tables**: Use SQL Server partitioning on TimeKey for tables exceeding 10M rows
2. **Incremental Refresh in Power BI**: Configure RangeStart/RangeEnd parameters for efficient data refresh
3. **Row-Level Security**: Implement in both SQL Server and Power BI for consistent access control
4. **Monitor Query Performance**: Use `sys.dm_exec_query_stats` to identify slow queries
5. **Version Control DAX Measures**: Store measures in external files and deploy via Tabular Editor
6. **Validate Data Lineage**: Maintain ETLControl table with load timestamps and row counts

## Common Patterns

### Pattern: Snapshot Fact Table for Slowly Changing Metrics

```sql
-- Create snapshot table for end-of-day inventory positions
CREATE TABLE FactInventorySnapshot (
    SnapshotKey BIGINT PRIMARY KEY IDENTITY(1,1),
    SnapshotDate DATE NOT NULL,
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    UnitsOnHand INT,
    UnitsInTransit INT,
    DaysOfSupply DECIMAL(5,2)
);

-- Daily snapshot job
INSERT INTO FactInventorySnapshot (SnapshotDate, ProductKey, GeographyKey, UnitsOnHand, UnitsInTransit, DaysOfSupply)
SELECT 
    CAST(GETDATE() AS DATE),
    ProductKey,
    GeographyKey,
    SUM(CASE WHEN Status = 'InStock' THEN Quantity ELSE 0 END),
    SUM(CASE WHEN Status = 'InTransit' THEN Quantity ELSE 0 END),
    SUM(CASE WHEN Status = 'InStock' THEN Quantity ELSE 0 END) / NULLIF(AVG(DailyUsage), 0)
FROM InventoryLedger
GROUP BY ProductKey, GeographyKey;
```

### Pattern: Role-Playing Dimension (Multiple Time References)

```sql
-- Use same DimTime for different contexts
SELECT 
    ft.VehicleID,
    t_departure.FullDateTime AS DepartureTime,
    t_arrival.FullDateTime AS ArrivalTime,
    ft.DistanceKm
FROM FactFleetTrips ft
INNER JOIN DimTime t_departure ON ft.DepartureTimeKey = t_departure.TimeKey
INNER JOIN DimTime t_arrival ON ft.ArrivalTimeKey = t_arrival.TimeKey;
```
