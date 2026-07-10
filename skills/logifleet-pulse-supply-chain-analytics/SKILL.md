---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server data warehouse and Power BI analytics platform for logistics, fleet management, and supply chain intelligence
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure Power BI logistics dashboard"
  - "create multi-fact star schema for warehouse data"
  - "implement fleet telemetry data warehouse"
  - "build supply chain KPI dashboard"
  - "deploy logistics analytics SQL schema"
  - "integrate warehouse and fleet data models"
  - "analyze cross-modal supply chain metrics"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive data warehousing and analytics platform built on MS SQL Server and Power BI for supply chain, logistics, and fleet management. It implements a multi-fact star schema that integrates:

- Warehouse operations (receiving, putaway, picking, packing, shipping)
- Fleet telemetry and GPS data
- Supplier reliability metrics
- External signals (weather, traffic)
- Customer order patterns

The platform provides unified cross-fact KPI analysis, predictive bottleneck detection, and real-time operational dashboards.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Power BI Desktop (latest version)
- .NET Framework 4.7.2+ (for data loading scripts)

### Initial Deployment

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy SQL schema:**
```sql
-- Run in SSMS connected to your target database
-- Creates all tables, views, stored procedures
:r schema/00_create_database.sql
:r schema/01_dimension_tables.sql
:r schema/02_fact_tables.sql
:r schema/03_bridge_tables.sql
:r schema/04_views.sql
:r schema/05_stored_procedures.sql
:r schema/06_indexes.sql
```

3. **Configure data source connections:**
```json
// config.json (copy from config_sample.json)
{
  "sqlServer": {
    "server": "${SQL_SERVER}",
    "database": "${SQL_DATABASE}",
    "authentication": "integrated"
  },
  "dataSources": {
    "wmsApiEndpoint": "${WMS_API_URL}",
    "fleetTelemetryEndpoint": "${FLEET_API_URL}",
    "weatherApiKey": "${WEATHER_API_KEY}"
  },
  "refreshInterval": 900
}
```

4. **Import Power BI template:**
```powershell
# Open Power BI Desktop
# File → Import → Power BI Template (.pbit)
# Select LogiFleet_Pulse_Master.pbit
# Enter SQL Server connection details when prompted
```

## Core Data Model

### Dimension Tables

```sql
-- DimTime: 15-minute granularity time dimension
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME NOT NULL,
    Hour TINYINT,
    DayOfWeek VARCHAR(10),
    IsWeekend BIT,
    FiscalQuarter TINYINT,
    FiscalYear SMALLINT
);

-- DimGeography: Hierarchical location data
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY,
    LocationID VARCHAR(50) NOT NULL,
    LocationName VARCHAR(100),
    LocationType VARCHAR(20), -- Warehouse, RouteNode, Facility
    RegionID VARCHAR(20),
    CountryCode CHAR(2),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6)
);

-- DimProductGravity: Product classification with velocity scoring
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL,
    ProductName VARCHAR(200),
    Category VARCHAR(50),
    GravityScore DECIMAL(5,2), -- Higher = faster moving
    IsFragile BIT,
    RequiresColdStorage BIT,
    UnitValue DECIMAL(10,2)
);

-- DimSupplierReliability: Supplier performance metrics
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY,
    SupplierID VARCHAR(50) NOT NULL,
    SupplierName VARCHAR(200),
    LeadTimeDays INT,
    LeadTimeVariance DECIMAL(5,2),
    DefectRate DECIMAL(5,4),
    ComplianceScore DECIMAL(3,2)
);
```

### Fact Tables

```sql
-- FactWarehouseOperations: Core warehouse activity
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    OperationType VARCHAR(20), -- Receiving, Putaway, Picking, Packing, Shipping
    DwellTimeMinutes INT,
    UnitsProcessed INT,
    LaborHours DECIMAL(6,2),
    ZoneID VARCHAR(20),
    BatchID VARCHAR(50)
);

-- FactFleetTrips: Vehicle telemetry and route data
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleID VARCHAR(50) NOT NULL,
    OriginGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DistanceMiles DECIMAL(8,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelGallons DECIMAL(6,2),
    LoadWeightLbs INT,
    DriverID VARCHAR(50)
);

-- FactCrossDock: Transfer operations
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    TransferTimeMinutes INT,
    UnitsTransferred INT
);
```

## Data Loading Patterns

### Incremental ETL Stored Procedure

```sql
CREATE PROCEDURE sp_LoadWarehouseOperations
    @StartDateTime DATETIME,
    @EndDateTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Load from external WMS system
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, GeographyKey,
        OperationType, DwellTimeMinutes, UnitsProcessed,
        LaborHours, ZoneID, BatchID
    )
    SELECT 
        t.TimeKey,
        p.ProductKey,
        g.GeographyKey,
        wms.OperationType,
        DATEDIFF(MINUTE, wms.StartTime, wms.EndTime) AS DwellTimeMinutes,
        wms.UnitsProcessed,
        wms.LaborHours,
        wms.ZoneID,
        wms.BatchID
    FROM OPENQUERY(WMS_LinkedServer, 
        'SELECT * FROM Operations 
         WHERE OperationDate >= ''' + CAST(@StartDateTime AS VARCHAR) + '''
         AND OperationDate < ''' + CAST(@EndDateTime AS VARCHAR) + '''
    ') wms
    INNER JOIN DimTime t ON DATEADD(MINUTE, 
        (DATEDIFF(MINUTE, 0, wms.OperationDate) / 15) * 15, 0) = t.DateTime
    INNER JOIN DimProductGravity p ON wms.SKU = p.SKU
    INNER JOIN DimGeography g ON wms.WarehouseID = g.LocationID
    WHERE NOT EXISTS (
        SELECT 1 FROM FactWarehouseOperations f
        WHERE f.BatchID = wms.BatchID
    );
    
    -- Update gravity scores based on velocity
    UPDATE p
    SET GravityScore = CASE 
        WHEN velocity.AvgDailyUnits > 1000 THEN 95.0
        WHEN velocity.AvgDailyUnits > 500 THEN 80.0
        WHEN velocity.AvgDailyUnits > 100 THEN 60.0
        ELSE 30.0
    END
    FROM DimProductGravity p
    INNER JOIN (
        SELECT 
            ProductKey,
            AVG(UnitsProcessed) AS AvgDailyUnits
        FROM FactWarehouseOperations
        WHERE TimeKey >= (SELECT MAX(TimeKey) - 672 FROM DimTime) -- Last 7 days
        GROUP BY ProductKey
    ) velocity ON p.ProductKey = velocity.ProductKey;
END;
```

### Automated Refresh Job

```sql
-- SQL Server Agent Job for scheduled execution
EXEC msdb.dbo.sp_add_job
    @job_name = N'LogiFleet_Incremental_Load',
    @enabled = 1;

EXEC msdb.dbo.sp_add_jobstep
    @job_name = N'LogiFleet_Incremental_Load',
    @step_name = N'Load_Warehouse_Operations',
    @subsystem = N'TSQL',
    @command = N'
        DECLARE @EndTime DATETIME = GETDATE();
        DECLARE @StartTime DATETIME = DATEADD(MINUTE, -15, @EndTime);
        EXEC sp_LoadWarehouseOperations @StartTime, @EndTime;
    ';

EXEC msdb.dbo.sp_add_schedule
    @schedule_name = N'Every_15_Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;

EXEC msdb.dbo.sp_attach_schedule
    @job_name = N'LogiFleet_Incremental_Load',
    @schedule_name = N'Every_15_Minutes';
```

## Key Analytical Queries

### Cross-Fact KPI: Dwell Time vs Fleet Idle Time

```sql
-- Correlate warehouse dwell with fleet idle time
WITH WarehouseDwell AS (
    SELECT 
        t.DateTime,
        g.LocationName AS Warehouse,
        AVG(w.DwellTimeMinutes) AS AvgDwellTime
    FROM FactWarehouseOperations w
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    INNER JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
    WHERE t.DateTime >= DATEADD(DAY, -7, GETDATE())
    GROUP BY t.DateTime, g.LocationName
),
FleetIdle AS (
    SELECT 
        t.DateTime,
        og.LocationName AS OriginWarehouse,
        AVG(f.IdleTimeMinutes) AS AvgIdleTime
    FROM FactFleetTrips f
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    INNER JOIN DimGeography og ON f.OriginGeographyKey = og.GeographyKey
    WHERE t.DateTime >= DATEADD(DAY, -7, GETDATE())
    GROUP BY t.DateTime, og.LocationName
)
SELECT 
    wd.DateTime,
    wd.Warehouse,
    wd.AvgDwellTime,
    fi.AvgIdleTime,
    (wd.AvgDwellTime * fi.AvgIdleTime) / 60.0 AS CombinedDelayHours
FROM WarehouseDwell wd
INNER JOIN FleetIdle fi 
    ON wd.DateTime = fi.DateTime 
    AND wd.Warehouse = fi.OriginWarehouse
WHERE wd.AvgDwellTime > 30 OR fi.AvgIdleTime > 20
ORDER BY CombinedDelayHours DESC;
```

### Predictive Bottleneck Detection

```sql
-- Identify zones approaching capacity with trend analysis
CREATE VIEW vw_BottleneckRisk AS
SELECT 
    g.LocationName,
    g.LocationID,
    COUNT(DISTINCT w.OperationKey) AS ActiveOperations,
    AVG(w.DwellTimeMinutes) AS AvgDwellTime,
    STDEV(w.DwellTimeMinutes) AS StdDevDwellTime,
    CASE 
        WHEN AVG(w.DwellTimeMinutes) > 120 AND COUNT(*) > 50 THEN 'CRITICAL'
        WHEN AVG(w.DwellTimeMinutes) > 80 OR COUNT(*) > 100 THEN 'WARNING'
        ELSE 'NORMAL'
    END AS RiskLevel,
    -- Trend: Compare last hour to previous 6 hours
    (SELECT AVG(DwellTimeMinutes) 
     FROM FactWarehouseOperations w2 
     WHERE w2.GeographyKey = w.GeographyKey 
     AND w2.TimeKey >= (SELECT MAX(TimeKey) - 4 FROM DimTime)) /
    (SELECT AVG(DwellTimeMinutes) 
     FROM FactWarehouseOperations w3 
     WHERE w3.GeographyKey = w.GeographyKey 
     AND w3.TimeKey BETWEEN (SELECT MAX(TimeKey) - 28 FROM DimTime) 
                        AND (SELECT MAX(TimeKey) - 4 FROM DimTime)) AS TrendRatio
FROM FactWarehouseOperations w
INNER JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
WHERE t.DateTime >= DATEADD(HOUR, -1, GETDATE())
GROUP BY g.LocationName, g.LocationID, w.GeographyKey;
```

### Fleet Efficiency by Route Segment

```sql
-- Calculate fuel efficiency and identify optimization opportunities
SELECT 
    og.LocationName AS Origin,
    dg.LocationName AS Destination,
    COUNT(*) AS TripCount,
    AVG(f.DistanceMiles) AS AvgDistance,
    AVG(f.FuelGallons) AS AvgFuel,
    AVG(f.DistanceMiles / NULLIF(f.FuelGallons, 0)) AS AvgMPG,
    AVG(CAST(f.IdleTimeMinutes AS FLOAT) / NULLIF(f.DurationMinutes, 0)) AS IdleTimeRatio,
    SUM(f.FuelGallons * 3.50) AS EstimatedFuelCost -- $3.50/gallon
FROM FactFleetTrips f
INNER JOIN DimGeography og ON f.OriginGeographyKey = og.GeographyKey
INNER JOIN DimGeography dg ON f.DestinationGeographyKey = dg.GeographyKey
INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
WHERE t.DateTime >= DATEADD(DAY, -30, GETDATE())
GROUP BY og.LocationName, dg.LocationName
HAVING COUNT(*) >= 10
ORDER BY IdleTimeRatio DESC;
```

## Power BI Integration

### DAX Measures for Cross-Fact Analysis

```dax
// Composite KPI: Warehouse Velocity Score
Warehouse Velocity Score = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR AvgUnits = AVERAGE(FactWarehouseOperations[UnitsProcessed])
VAR AvgLabor = AVERAGE(FactWarehouseOperations[LaborHours])
RETURN
    (AvgUnits / NULLIF(AvgLabor, 0)) * (1 / NULLIF(AvgDwell, 0)) * 1000

// Fleet Utilization Percentage
Fleet Utilization % = 
DIVIDE(
    SUM(FactFleetTrips[DurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[DurationMinutes]),
    0
) * 100

// Cross-Fact: Cost Per Unit Shipped
Cost Per Unit = 
VAR TotalFuelCost = SUMX(FactFleetTrips, [FuelGallons] * 3.50)
VAR TotalLaborCost = SUMX(FactWarehouseOperations, [LaborHours] * 25)
VAR TotalUnits = SUM(FactWarehouseOperations[UnitsProcessed])
RETURN
    DIVIDE(TotalFuelCost + TotalLaborCost, TotalUnits, 0)

// Time Intelligence: Week-over-Week Growth
WoW Dwell Time Change = 
VAR CurrentWeek = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR PreviousWeek = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        DATEADD(DimTime[DateTime], -7, DAY)
    )
RETURN
    DIVIDE(CurrentWeek - PreviousWeek, PreviousWeek, 0)
```

### Power BI Template Connection

```powerquery
// M Query for SQL Server connection
let
    Source = Sql.Database(
        Environment.GetEnvironmentVariable("SQL_SERVER"),
        Environment.GetEnvironmentVariable("SQL_DATABASE"),
        [
            Query = "SELECT * FROM vw_OperationalSummary WHERE DateTime >= DATEADD(DAY, -90, GETDATE())"
        ]
    ),
    ChangeTypes = Table.TransformColumnTypes(Source, {
        {"DateTime", type datetime},
        {"DwellTimeMinutes", type number},
        {"UnitsProcessed", Int64.Type}
    })
in
    ChangeTypes
```

## Configuration Management

### Row-Level Security Setup

```sql
-- Create security roles for department-based access
CREATE ROLE WarehouseManager;
CREATE ROLE FleetManager;
CREATE ROLE Executive;

-- Security predicate function
CREATE FUNCTION fn_SecurityPredicate(@LocationID VARCHAR(50))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN (
    SELECT 1 AS AccessGranted
    WHERE 
        IS_MEMBER('Executive') = 1
        OR (IS_MEMBER('WarehouseManager') = 1 
            AND @LocationID IN (SELECT LocationID FROM dbo.UserWarehouseAccess WHERE UserName = USER_NAME()))
        OR (IS_MEMBER('FleetManager') = 1 
            AND @LocationID IN (SELECT LocationID FROM dbo.UserFleetAccess WHERE UserName = USER_NAME()))
);

-- Apply security policy
CREATE SECURITY POLICY DepartmentSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(LocationID)
ON dbo.FactWarehouseOperations,
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(OriginLocationID)
ON dbo.FactFleetTrips
WITH (STATE = ON);
```

### Alert Configuration

```sql
-- Automated alert for critical bottlenecks
CREATE PROCEDURE sp_CheckAndAlert_Bottlenecks
AS
BEGIN
    DECLARE @AlertThreshold INT = 120; -- Minutes
    DECLARE @EmailBody NVARCHAR(MAX);
    
    SELECT 
        @EmailBody = STRING_AGG(
            CONCAT(
                'Location: ', LocationName, 
                ' | Avg Dwell: ', CAST(AvgDwellTime AS VARCHAR), ' min',
                ' | Risk: ', RiskLevel
            ), 
            CHAR(13) + CHAR(10)
        )
    FROM vw_BottleneckRisk
    WHERE RiskLevel IN ('CRITICAL', 'WARNING');
    
    IF @EmailBody IS NOT NULL
    BEGIN
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleet_Alerts',
            @recipients = '${ALERT_EMAIL_RECIPIENTS}',
            @subject = 'LogiFleet Pulse: Bottleneck Alert',
            @body = @EmailBody;
    END
END;
```

## Common Patterns & Troubleshooting

### Pattern: Slowly Changing Dimensions (Type 2)

```sql
-- Handle supplier reliability changes with history tracking
CREATE PROCEDURE sp_UpdateSupplierReliability
    @SupplierID VARCHAR(50),
    @NewLeadTimeDays INT,
    @NewDefectRate DECIMAL(5,4)
AS
BEGIN
    -- Expire current record
    UPDATE DimSupplierReliability
    SET ValidTo = GETDATE(), IsCurrent = 0
    WHERE SupplierID = @SupplierID AND IsCurrent = 1;
    
    -- Insert new record
    INSERT INTO DimSupplierReliability (
        SupplierID, SupplierName, LeadTimeDays, DefectRate, 
        ValidFrom, ValidTo, IsCurrent
    )
    SELECT 
        SupplierID, SupplierName, @NewLeadTimeDays, @NewDefectRate,
        GETDATE(), '9999-12-31', 1
    FROM DimSupplierReliability
    WHERE SupplierID = @SupplierID AND IsCurrent = 0
    AND ValidTo = (SELECT MAX(ValidTo) FROM DimSupplierReliability WHERE SupplierID = @SupplierID);
END;
```

### Troubleshooting: Performance Optimization

```sql
-- Create covering indexes for common query patterns
CREATE NONCLUSTERED INDEX IX_FactWarehouse_TimeProduct
ON FactWarehouseOperations (TimeKey, ProductKey)
INCLUDE (DwellTimeMinutes, UnitsProcessed, LaborHours);

CREATE NONCLUSTERED INDEX IX_FactFleet_TimeRoute
ON FactFleetTrips (TimeKey, OriginGeographyKey, DestinationGeographyKey)
INCLUDE (DistanceMiles, FuelGallons, IdleTimeMinutes);

-- Partitioning strategy for large fact tables
CREATE PARTITION FUNCTION pf_Monthly (DATETIME)
AS RANGE RIGHT FOR VALUES (
    '2026-01-01', '2026-02-01', '2026-03-01', '2026-04-01',
    '2026-05-01', '2026-06-01', '2026-07-01', '2026-08-01'
);

CREATE PARTITION SCHEME ps_Monthly
AS PARTITION pf_Monthly
ALL TO ([PRIMARY]);

-- Rebuild table with partitioning
CREATE TABLE FactWarehouseOperations_Partitioned (
    -- ... same columns ...
) ON ps_Monthly(OperationDate);
```

### Troubleshooting: Data Quality Checks

```sql
-- Identify orphaned records and data anomalies
SELECT 
    'Orphaned Products' AS Issue,
    COUNT(*) AS RecordCount
FROM FactWarehouseOperations w
LEFT JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
WHERE p.ProductKey IS NULL

UNION ALL

SELECT 
    'Negative Dwell Time' AS Issue,
    COUNT(*)
FROM FactWarehouseOperations
WHERE DwellTimeMinutes < 0

UNION ALL

SELECT 
    'Impossible Fuel Efficiency' AS Issue,
    COUNT(*)
FROM FactFleetTrips
WHERE (DistanceMiles / NULLIF(FuelGallons, 0)) > 20 OR (DistanceMiles / NULLIF(FuelGallons, 0)) < 2;
```

## API Integration Examples

### Fleet Telemetry Ingestion

```csharp
// C# example for streaming fleet data into SQL
using System.Data.SqlClient;
using Newtonsoft.Json;

public class FleetTelemetryLoader
{
    private string _connectionString = Environment.GetEnvironmentVariable("SQL_CONNECTION_STRING");
    
    public async Task LoadTelemetryBatch(List<FleetTelemetry> batch)
    {
        using (var connection = new SqlConnection(_connectionString))
        {
            await connection.OpenAsync();
            
            using (var bulkCopy = new SqlBulkCopy(connection))
            {
                bulkCopy.DestinationTableName = "Staging_FleetTelemetry";
                bulkCopy.BatchSize = 1000;
                
                var dataTable = ConvertToDataTable(batch);
                await bulkCopy.WriteToServerAsync(dataTable);
            }
            
            // Call stored procedure to process staging data
            using (var command = new SqlCommand("sp_ProcessFleetTelemetry", connection))
            {
                command.CommandType = CommandType.StoredProcedure;
                await command.ExecuteNonQueryAsync();
            }
        }
    }
}
```

### Weather API Integration

```python
# Python script for external weather data enrichment
import requests
import pyodbc
import os
from datetime import datetime

def enrich_with_weather():
    conn_str = os.environ['SQL_CONNECTION_STRING']
    weather_api_key = os.environ['WEATHER_API_KEY']
    
    conn = pyodbc.connect(conn_str)
    cursor = conn.cursor()
    
    # Get trips from last hour without weather data
    cursor.execute("""
        SELECT TripKey, OriginGeographyKey, DateTime, Latitude, Longitude
        FROM vw_TripsNeedingWeather
    """)
    
    for row in cursor.fetchall():
        weather_data = requests.get(
            f"https://api.weather.com/historical",
            params={
                'lat': row.Latitude,
                'lon': row.Longitude,
                'dt': row.DateTime.isoformat(),
                'apikey': weather_api_key
            }
        ).json()
        
        cursor.execute("""
            UPDATE FactFleetTrips
            SET WeatherCondition = ?, Temperature = ?, Precipitation = ?
            WHERE TripKey = ?
        """, weather_data['condition'], weather_data['temp'], 
             weather_data['precip'], row.TripKey)
    
    conn.commit()
    conn.close()
```

## Best Practices

1. **Incremental Loading**: Always use watermark columns (TimeKey, LastModified) to avoid full table scans
2. **Indexing Strategy**: Create covering indexes for fact table queries joining to 2+ dimensions
3. **Partitioning**: Partition fact tables by month for tables exceeding 50M rows
4. **Security**: Always use row-level security for multi-tenant deployments
5. **Monitoring**: Set up SQL Server Extended Events to track long-running queries
6. **Backup Strategy**: Daily full backups of dimension tables, hourly transaction log backups for fact tables

This skill provides the foundation for implementing and extending LogiFleet Pulse for real-world supply chain analytics scenarios.
