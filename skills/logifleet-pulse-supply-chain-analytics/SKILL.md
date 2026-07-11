---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehouse for logistics analytics with multi-fact star schema, fleet telemetry, and warehouse optimization
triggers:
  - set up supply chain analytics dashboard
  - configure logistics data warehouse
  - implement fleet tracking analytics
  - create warehouse operations reporting
  - deploy logifleet pulse database
  - build supply chain intelligence platform
  - setup multi-fact star schema for logistics
  - configure power bi logistics dashboards
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

LogiFleet Pulse is a comprehensive logistics intelligence platform built on MS SQL Server and Power BI. It implements a multi-fact star schema that unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a single semantic layer for real-time supply chain decision-making.

## What It Does

- **Multi-Modal Data Integration**: Combines warehouse management, fleet GPS/telemetry, supplier data, and external APIs (weather, traffic)
- **Advanced Star Schema**: Multi-fact architecture with time-phased dimensions and bridge tables for many-to-many relationships
- **Real-Time Dashboards**: Power BI dashboards with 15-minute refresh intervals
- **Predictive Analytics**: Bottleneck detection, fleet maintenance prioritization, and temporal elasticity modeling
- **Warehouse Gravity Zones**: Spatial optimization based on pick frequency, item value, and fragility
- **Cross-Fact KPI Harmonization**: Link inventory turnover with fleet fuel consumption, dwell time with route efficiency

## Installation

### Prerequisites

- MS SQL Server 2019 or later
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) recommended
- Admin access to deploy database schema

### Database Setup

1. **Clone the repository**:
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy SQL schema**:
```sql
-- Connect to your SQL Server instance via SSMS
-- Run the schema deployment script
:r schema_deployment.sql

-- Verify tables were created
SELECT TABLE_NAME 
FROM INFORMATION_SCHEMA.TABLES 
WHERE TABLE_SCHEMA = 'dbo' 
AND TABLE_NAME LIKE 'Fact%' OR TABLE_NAME LIKE 'Dim%';
```

3. **Configure data sources**:
```json
// config_sample.json - copy to config.json and update
{
  "sql_server": {
    "server": "localhost\\SQLEXPRESS",
    "database": "LogiFleetPulse",
    "authentication": "windows"
  },
  "data_sources": {
    "wms_api": "${WMS_API_ENDPOINT}",
    "fleet_telemetry": "${FLEET_API_ENDPOINT}",
    "weather_api_key": "${WEATHER_API_KEY}"
  },
  "refresh_interval_minutes": 15
}
```

4. **Initialize dimension tables**:
```sql
-- Create time dimension (15-minute granularity)
EXEC sp_PopulateDimTime @StartDate = '2024-01-01', @EndDate = '2027-12-31';

-- Initialize geography hierarchy
EXEC sp_PopulateDimGeography;

-- Setup product gravity scoring
EXEC sp_CalculateProductGravityScores;
```

### Power BI Template Setup

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. Enter your SQL Server connection details when prompted
3. Configure row-level security for user roles
4. Publish to Power BI Service for scheduled refresh

## Key SQL Objects

### Fact Tables

**FactWarehouseOperations** - Core warehouse activity tracking:
```sql
-- Sample query: Daily putaway efficiency by zone
SELECT 
    d.CalendarDate,
    g.WarehouseZone,
    COUNT(f.OperationID) AS TotalOperations,
    AVG(f.DwellTimeMinutes) AS AvgDwellTime,
    SUM(f.UnitsProcessed) AS TotalUnits,
    AVG(f.PickRatePerHour) AS AvgPickRate
FROM FactWarehouseOperations f
INNER JOIN DimTime d ON f.TimeKey = d.TimeKey
INNER JOIN DimGeography g ON f.GeographyKey = g.GeographyKey
WHERE d.CalendarDate >= DATEADD(day, -30, GETDATE())
GROUP BY d.CalendarDate, g.WarehouseZone
ORDER BY d.CalendarDate DESC;
```

**FactFleetTrips** - Fleet telemetry and route data:
```sql
-- Sample query: Fleet idle time vs fuel consumption correlation
SELECT 
    v.VehicleID,
    v.VehicleType,
    COUNT(f.TripID) AS TotalTrips,
    SUM(f.IdleTimeMinutes) AS TotalIdleTime,
    SUM(f.FuelConsumedLiters) AS TotalFuel,
    ROUND(SUM(f.IdleTimeMinutes) * 100.0 / SUM(f.TripDurationMinutes), 2) AS IdlePercentage,
    ROUND(SUM(f.FuelConsumedLiters) / SUM(f.DistanceKM), 2) AS FuelEfficiency
FROM FactFleetTrips f
INNER JOIN DimVehicle v ON f.VehicleKey = v.VehicleKey
INNER JOIN DimTime d ON f.DeparturTimeKey = d.TimeKey
WHERE d.CalendarDate >= DATEADD(day, -7, GETDATE())
GROUP BY v.VehicleID, v.VehicleType
HAVING SUM(f.IdleTimeMinutes) > 0
ORDER BY IdlePercentage DESC;
```

**FactCrossDock** - Cross-dock transfer operations:
```sql
-- Sample query: Cross-dock throughput by product category
SELECT 
    p.ProductCategory,
    p.ProductSubCategory,
    COUNT(f.CrossDockID) AS Transfers,
    AVG(f.TransferTimeMinutes) AS AvgTransferTime,
    SUM(f.UnitsTransferred) AS TotalUnits,
    SUM(CASE WHEN f.TransferTimeMinutes > 120 THEN 1 ELSE 0 END) AS DelayedTransfers
FROM FactCrossDock f
INNER JOIN DimProduct p ON f.ProductKey = p.ProductKey
INNER JOIN DimTime d ON f.InboundTimeKey = d.TimeKey
WHERE d.CalendarDate >= DATEADD(day, -14, GETDATE())
GROUP BY p.ProductCategory, p.ProductSubCategory
ORDER BY TotalUnits DESC;
```

### Dimension Tables

**DimTime** - Time-phased dimension with 15-minute buckets:
```sql
-- Structure
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    CalendarDate DATE NOT NULL,
    TimeOfDay TIME NOT NULL,
    HourOfDay TINYINT NOT NULL,
    QuarterHour TINYINT NOT NULL,
    DayOfWeek TINYINT NOT NULL,
    DayName VARCHAR(10) NOT NULL,
    IsWeekend BIT NOT NULL,
    FiscalPeriod VARCHAR(10) NOT NULL,
    FiscalQuarter TINYINT NOT NULL,
    FiscalYear SMALLINT NOT NULL
);

-- Usage: Find peak warehouse activity hours
SELECT 
    t.HourOfDay,
    t.DayName,
    COUNT(f.OperationID) AS Operations,
    AVG(f.PickRatePerHour) AS AvgPickRate
FROM FactWarehouseOperations f
INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
WHERE t.CalendarDate >= DATEADD(day, -90, GETDATE())
GROUP BY t.HourOfDay, t.DayName
ORDER BY Operations DESC;
```

**DimProductGravity** - Product classification by movement velocity:
```sql
-- Calculate gravity score for warehouse placement
SELECT 
    p.ProductID,
    p.ProductName,
    p.GravityScore,
    p.PickFrequencyRank,
    p.UnitValue,
    p.FragilityIndex,
    CASE 
        WHEN p.GravityScore >= 80 THEN 'High Gravity - Near Dock'
        WHEN p.GravityScore >= 50 THEN 'Medium Gravity - Mid Zone'
        ELSE 'Low Gravity - Deep Storage'
    END AS RecommendedZone
FROM DimProductGravity p
ORDER BY p.GravityScore DESC;
```

**DimSupplierReliability** - Supplier performance metrics:
```sql
-- Identify unreliable suppliers affecting inventory
SELECT 
    s.SupplierName,
    s.LeadTimeVarianceDays,
    s.DefectPercentage,
    s.OnTimeDeliveryRate,
    s.ComplianceScore,
    COUNT(f.OperationID) AS InboundOperations
FROM DimSupplierReliability s
LEFT JOIN FactWarehouseOperations f ON s.SupplierKey = f.SupplierKey
WHERE s.OnTimeDeliveryRate < 85.0 OR s.DefectPercentage > 2.0
GROUP BY s.SupplierName, s.LeadTimeVarianceDays, s.DefectPercentage, 
         s.OnTimeDeliveryRate, s.ComplianceScore
ORDER BY s.ComplianceScore ASC;
```

## Common Analytics Patterns

### Cross-Fact KPI Queries

**Correlate warehouse dwell time with fleet delivery delays**:
```sql
WITH WarehouseDwell AS (
    SELECT 
        p.ProductID,
        d.CalendarDate,
        AVG(w.DwellTimeMinutes) AS AvgDwellTime
    FROM FactWarehouseOperations w
    INNER JOIN DimProduct p ON w.ProductKey = p.ProductKey
    INNER JOIN DimTime d ON w.TimeKey = d.TimeKey
    WHERE d.CalendarDate >= DATEADD(day, -30, GETDATE())
    GROUP BY p.ProductID, d.CalendarDate
),
FleetDelays AS (
    SELECT 
        p.ProductID,
        d.CalendarDate,
        AVG(f.DelayMinutes) AS AvgDelayMinutes
    FROM FactFleetTrips f
    INNER JOIN BridgeProductRoute br ON f.TripID = br.TripID
    INNER JOIN DimProduct p ON br.ProductKey = p.ProductKey
    INNER JOIN DimTime d ON f.DeparturTimeKey = d.TimeKey
    WHERE d.CalendarDate >= DATEADD(day, -30, GETDATE())
    GROUP BY p.ProductID, d.CalendarDate
)
SELECT 
    wd.ProductID,
    AVG(wd.AvgDwellTime) AS AvgWarehouseDwell,
    AVG(fd.AvgDelayMinutes) AS AvgFleetDelay,
    CORR(wd.AvgDwellTime, fd.AvgDelayMinutes) AS CorrelationCoefficient
FROM WarehouseDwell wd
INNER JOIN FleetDelays fd ON wd.ProductID = fd.ProductID AND wd.CalendarDate = fd.CalendarDate
GROUP BY wd.ProductID
HAVING COUNT(*) >= 10
ORDER BY CorrelationCoefficient DESC;
```

### Predictive Bottleneck Detection

**Identify emerging warehouse congestion**:
```sql
-- Stored procedure for bottleneck prediction
CREATE PROCEDURE sp_PredictBottlenecks
    @HorizonHours INT = 4
AS
BEGIN
    WITH CurrentLoad AS (
        SELECT 
            g.WarehouseZone,
            COUNT(w.OperationID) AS ActiveOperations,
            AVG(w.DwellTimeMinutes) AS CurrentDwellTime,
            MAX(w.DwellTimeMinutes) AS MaxDwellTime
        FROM FactWarehouseOperations w
        INNER JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
        INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
        WHERE t.FullDateTime >= DATEADD(hour, -1, GETDATE())
        GROUP BY g.WarehouseZone
    ),
    HistoricalAverage AS (
        SELECT 
            g.WarehouseZone,
            t.HourOfDay,
            AVG(w.DwellTimeMinutes) AS HistoricalDwellTime,
            STDEV(w.DwellTimeMinutes) AS DwellTimeStdDev
        FROM FactWarehouseOperations w
        INNER JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
        INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
        WHERE t.CalendarDate >= DATEADD(day, -30, GETDATE())
        GROUP BY g.WarehouseZone, t.HourOfDay
    )
    SELECT 
        cl.WarehouseZone,
        cl.ActiveOperations,
        cl.CurrentDwellTime,
        ha.HistoricalDwellTime,
        ha.DwellTimeStdDev,
        ROUND((cl.CurrentDwellTime - ha.HistoricalDwellTime) / ha.DwellTimeStdDev, 2) AS ZScore,
        CASE 
            WHEN (cl.CurrentDwellTime - ha.HistoricalDwellTime) / ha.DwellTimeStdDev > 2 THEN 'CRITICAL'
            WHEN (cl.CurrentDwellTime - ha.HistoricalDwellTime) / ha.DwellTimeStdDev > 1 THEN 'WARNING'
            ELSE 'NORMAL'
        END AS BottleneckRisk
    FROM CurrentLoad cl
    INNER JOIN HistoricalAverage ha ON cl.WarehouseZone = ha.WarehouseZone
    WHERE ha.HourOfDay = DATEPART(hour, GETDATE())
    ORDER BY ZScore DESC;
END;
```

### Fleet Maintenance Priority Scoring

**Rank fleet maintenance needs by revenue impact**:
```sql
SELECT 
    v.VehicleID,
    v.VehicleType,
    v.LastMaintenanceDate,
    DATEDIFF(day, v.LastMaintenanceDate, GETDATE()) AS DaysSinceMaintenance,
    AVG(f.LoadValue) AS AvgCargoValue,
    SUM(CASE WHEN f.IsHighPriority = 1 THEN 1 ELSE 0 END) AS HighPriorityTrips,
    MAX(v.EngineDiagnosticScore) AS DiagnosticScore,
    MAX(v.TirePressureAlertLevel) AS TireAlertLevel,
    -- Weighted scoring formula
    (DATEDIFF(day, v.LastMaintenanceDate, GETDATE()) * 0.3 +
     AVG(f.LoadValue) / 1000 * 0.4 +
     MAX(v.EngineDiagnosticScore) * 0.2 +
     MAX(v.TirePressureAlertLevel) * 0.1) AS MaintenancePriorityScore
FROM DimVehicle v
LEFT JOIN FactFleetTrips f ON v.VehicleKey = f.VehicleKey
WHERE v.IsActive = 1
GROUP BY v.VehicleID, v.VehicleType, v.LastMaintenanceDate
ORDER BY MaintenancePriorityScore DESC;
```

## Configuration

### Row-Level Security Setup

Implement role-based access control in SQL:

```sql
-- Create security roles
CREATE ROLE WarehouseOperator;
CREATE ROLE FleetManager;
CREATE ROLE ExecutiveDashboard;

-- Create security filter function
CREATE FUNCTION fn_SecurityFilter(@UserRole VARCHAR(50))
RETURNS TABLE
AS
RETURN
(
    SELECT 
        u.UserID,
        u.UserRole,
        u.AllowedWarehouses,
        u.AllowedRegions
    FROM DimUserSecurity u
    WHERE u.UserRole = @UserRole
);

-- Apply to fact table
CREATE SECURITY POLICY WarehouseSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityFilter(USER_NAME()) ON FactWarehouseOperations,
ADD FILTER PREDICATE dbo.fn_SecurityFilter(USER_NAME()) ON FactFleetTrips
WITH (STATE = ON);
```

### Automated Alert Configuration

**Setup threshold-based alerts**:

```sql
CREATE PROCEDURE sp_CheckKPIAlerts
AS
BEGIN
    -- Fleet idle time threshold
    INSERT INTO AlertLog (AlertType, Severity, Message, CreatedAt)
    SELECT 
        'FLEET_IDLE_EXCESSIVE',
        'HIGH',
        'Vehicle ' + v.VehicleID + ' idle time: ' + 
        CAST(SUM(f.IdleTimeMinutes) AS VARCHAR) + ' minutes (threshold: 180)',
        GETDATE()
    FROM FactFleetTrips f
    INNER JOIN DimVehicle v ON f.VehicleKey = v.VehicleKey
    INNER JOIN DimTime d ON f.DeparturTimeKey = d.TimeKey
    WHERE d.CalendarDate = CAST(GETDATE() AS DATE)
    GROUP BY v.VehicleID
    HAVING SUM(f.IdleTimeMinutes) > 180;
    
    -- Warehouse dwell time threshold
    INSERT INTO AlertLog (AlertType, Severity, Message, CreatedAt)
    SELECT 
        'WAREHOUSE_DWELL_CRITICAL',
        'CRITICAL',
        'Zone ' + g.WarehouseZone + ' avg dwell: ' + 
        CAST(AVG(w.DwellTimeMinutes) AS VARCHAR) + ' minutes (threshold: 240)',
        GETDATE()
    FROM FactWarehouseOperations w
    INNER JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
    INNER JOIN DimTime d ON w.TimeKey = d.TimeKey
    WHERE d.FullDateTime >= DATEADD(hour, -4, GETDATE())
    GROUP BY g.WarehouseZone
    HAVING AVG(w.DwellTimeMinutes) > 240;
END;

-- Schedule via SQL Agent job (runs every 15 minutes)
EXEC sp_CheckKPIAlerts;
```

### Power BI DAX Measures

**Key calculated measures for dashboards**:

```dax
// Total Warehouse Operations
TotalOperations = COUNTROWS(FactWarehouseOperations)

// Average Dwell Time with time intelligence
AvgDwellTime = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
AvgDwellTimePriorPeriod = CALCULATE([AvgDwellTime], DATEADD(DimTime[CalendarDate], -1, MONTH))
DwellTimeChange% = DIVIDE([AvgDwellTime] - [AvgDwellTimePriorPeriod], [AvgDwellTimePriorPeriod], 0)

// Fleet Efficiency Score
FleetEfficiencyScore = 
    DIVIDE(
        SUM(FactFleetTrips[DistanceKM]),
        SUM(FactFleetTrips[FuelConsumedLiters]) * 
        (1 + SUM(FactFleetTrips[IdleTimeMinutes]) / SUM(FactFleetTrips[TripDurationMinutes]))
    , 0) * 100

// Cross-Fact KPI: Delivery Performance Index
DeliveryPerformanceIndex = 
    VAR WarehouseEfficiency = DIVIDE([TotalOperations], [AvgDwellTime], 0)
    VAR FleetOnTimeRate = DIVIDE(
        COUNTROWS(FILTER(FactFleetTrips, FactFleetTrips[DelayMinutes] <= 15)),
        COUNTROWS(FactFleetTrips), 0)
    RETURN (WarehouseEfficiency * 0.4 + FleetOnTimeRate * 0.6) * 100

// Product Gravity Zone Compliance
GravityZoneCompliance% = 
    DIVIDE(
        COUNTROWS(FILTER(FactWarehouseOperations, 
            FactWarehouseOperations[ActualZone] = FactWarehouseOperations[RecommendedZone])),
        COUNTROWS(FactWarehouseOperations), 0)
```

## Data Loading Patterns

### Incremental ETL Process

**Load warehouse operations incrementally**:

```sql
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Staging table from WMS source
    TRUNCATE TABLE StagingWarehouseOperations;
    
    INSERT INTO StagingWarehouseOperations
    SELECT * FROM OPENQUERY(WMS_LINKEDSERVER, 
        'SELECT operation_id, warehouse_id, product_sku, operation_type, 
                timestamp, dwell_minutes, units_processed, pick_rate
         FROM operations_log 
         WHERE last_modified > ''' + CONVERT(VARCHAR, @LastLoadDateTime, 120) + '''');
    
    -- Merge into fact table
    MERGE FactWarehouseOperations AS target
    USING (
        SELECT 
            s.operation_id,
            t.TimeKey,
            g.GeographyKey,
            p.ProductKey,
            s.operation_type,
            s.dwell_minutes,
            s.units_processed,
            s.pick_rate
        FROM StagingWarehouseOperations s
        LEFT JOIN DimTime t ON CAST(s.timestamp AS DATETIME) = t.FullDateTime
        LEFT JOIN DimGeography g ON s.warehouse_id = g.WarehouseID
        LEFT JOIN DimProduct p ON s.product_sku = p.SKU
    ) AS source
    ON target.OperationID = source.operation_id
    WHEN MATCHED THEN
        UPDATE SET 
            target.DwellTimeMinutes = source.dwell_minutes,
            target.UnitsProcessed = source.units_processed
    WHEN NOT MATCHED THEN
        INSERT (OperationID, TimeKey, GeographyKey, ProductKey, OperationType, 
                DwellTimeMinutes, UnitsProcessed, PickRatePerHour)
        VALUES (source.operation_id, source.TimeKey, source.GeographyKey, 
                source.ProductKey, source.operation_type, source.dwell_minutes,
                source.units_processed, source.pick_rate);
    
    -- Update control table
    UPDATE ETLControl 
    SET LastLoadDateTime = GETDATE(), 
        RowsProcessed = @@ROWCOUNT
    WHERE TableName = 'FactWarehouseOperations';
END;
```

### External API Integration

**Weather data enrichment for fleet analytics**:

```sql
-- SQL CLR function or external table approach
CREATE EXTERNAL DATA SOURCE WeatherAPI
WITH (
    TYPE = BLOB_STORAGE,
    LOCATION = '${WEATHER_API_ENDPOINT}',
    CREDENTIAL = WeatherAPICredential
);

-- Load weather conditions for route correlation
INSERT INTO DimWeatherConditions (RouteID, Timestamp, Temperature, Precipitation, WindSpeed)
SELECT 
    r.RouteID,
    w.timestamp,
    w.temperature_celsius,
    w.precipitation_mm,
    w.wind_speed_kmh
FROM ExternalWeatherData w
CROSS APPLY (
    SELECT RouteID FROM DimRoute 
    WHERE Latitude BETWEEN w.lat - 0.5 AND w.lat + 0.5
    AND Longitude BETWEEN w.lon - 0.5 AND w.lon + 0.5
) r
WHERE w.timestamp >= DATEADD(day, -7, GETDATE());
```

## Troubleshooting

### Common Issues

**Slow query performance on cross-fact joins**:
```sql
-- Check missing indexes
SELECT 
    t.name AS TableName,
    i.name AS IndexName,
    dm_ius.user_seeks,
    dm_ius.user_scans,
    dm_ius.last_user_seek
FROM sys.dm_db_index_usage_stats dm_ius
INNER JOIN sys.tables t ON dm_ius.object_id = t.object_id
INNER JOIN sys.indexes i ON dm_ius.index_id = i.index_id AND dm_ius.object_id = i.object_id
WHERE database_id = DB_ID('LogiFleetPulse')
ORDER BY dm_ius.user_seeks DESC;

-- Add recommended indexes
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time_Geography
ON FactWarehouseOperations (TimeKey, GeographyKey)
INCLUDE (DwellTimeMinutes, UnitsProcessed);

CREATE NONCLUSTERED INDEX IX_FactFleet_Time_Vehicle
ON FactFleetTrips (DeparturTimeKey, VehicleKey)
INCLUDE (FuelConsumedLiters, IdleTimeMinutes);
```

**Power BI refresh failures**:
```powershell
# Check Power BI gateway logs
Get-Content "C:\Program Files\On-premises data gateway\GatewayLogs.log" -Tail 50

# Test SQL connection from gateway machine
Test-NetConnection -ComputerName "sql-server.domain.com" -Port 1433

# Verify service account permissions
sqlcmd -S sql-server -Q "EXEC sp_helprotect NULL, '${POWERBI_SERVICE_ACCOUNT}'"
```

**Data freshness issues**:
```sql
-- Monitor ETL execution history
SELECT 
    TableName,
    LastLoadDateTime,
    RowsProcessed,
    DATEDIFF(minute, LastLoadDateTime, GETDATE()) AS MinutesSinceLoad
FROM ETLControl
WHERE MinutesSinceLoad > 30
ORDER BY MinutesSinceLoad DESC;

-- Force immediate refresh
EXEC sp_LoadWarehouseOperations @LastLoadDateTime = (
    SELECT LastLoadDateTime FROM ETLControl WHERE TableName = 'FactWarehouseOperations'
);
EXEC sp_LoadFleetTrips @LastLoadDateTime = (
    SELECT LastLoadDateTime FROM ETLControl WHERE TableName = 'FactFleetTrips'
);
```

**Gravity score calculation errors**:
```sql
-- Validate product gravity calculations
SELECT 
    ProductID,
    PickFrequencyRank,
    UnitValue,
    FragilityIndex,
    GravityScore,
    -- Recalculate to verify
    (PickFrequencyRank * 0.5 + 
     CASE WHEN UnitValue > 100 THEN 30 ELSE 10 END +
     FragilityIndex * 0.2) AS RecalculatedScore
FROM DimProductGravity
WHERE ABS(GravityScore - 
    (PickFrequencyRank * 0.5 + 
     CASE WHEN UnitValue > 100 THEN 30 ELSE 10 END +
     FragilityIndex * 0.2)) > 5;

-- Refresh gravity scores
EXEC sp_CalculateProductGravityScores;
```

## Best Practices

1. **Partition large fact tables** by date for better query performance:
```sql
ALTER DATABASE LogiFleetPulse ADD FILEGROUP FG_2024;
ALTER DATABASE LogiFleetPulse ADD FILEGROUP FG_2025;
-- Create partition function and scheme accordingly
```

2. **Use columnstore indexes** for analytical queries:
```sql
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouseOps
ON FactWarehouseOperations (TimeKey, GeographyKey, ProductKey, DwellTimeMinutes, UnitsProcessed);
```

3. **Schedule statistics updates** during off-peak hours:
```sql
UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS FactFleetTrips WITH FULLSCAN;
```

4. **Implement archival strategy** for historical data:
```sql
-- Move data older than 2 years to archive table
INSERT INTO FactWarehouseOperations_Archive
SELECT * FROM FactWarehouseOperations
WHERE TimeKey IN (SELECT TimeKey FROM DimTime WHERE CalendarDate < DATEADD(year, -2, GETDATE()));
```

5. **Monitor Power BI dataset refresh** via Power BI REST API:
```powershell
$headers = @{Authorization = "Bearer ${POWERBI_ACCESS_TOKEN}"}
Invoke-RestMethod -Uri "https://api.powerbi.com/v1.0/myorg/groups/${WORKSPACE_ID}/datasets/${DATASET_ID}/refreshes" -Headers $headers
```
