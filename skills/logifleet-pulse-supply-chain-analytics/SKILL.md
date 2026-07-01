---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics analytics platform with multi-fact star schema for warehouse, fleet, and supply chain intelligence
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure Power BI logistics dashboard"
  - "deploy warehouse and fleet data model"
  - "create supply chain star schema in SQL Server"
  - "implement cross-fact KPI harmonization"
  - "build real-time logistics intelligence platform"
  - "integrate warehouse and fleet telemetry data"
  - "troubleshoot Power BI logistics template"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

LogiFleet Pulse is a comprehensive logistics intelligence platform that unifies warehouse operations, fleet telemetry, inventory management, and supply chain data into a single analytical layer. Built on MS SQL Server with Power BI visualization, it provides:

- **Multi-fact star schema** linking warehouse, fleet, and cross-dock operations
- **Time-phased dimensions** for temporal analysis (15-minute granularity)
- **Cross-modal KPI harmonization** (e.g., dwell time vs. fleet idle cost)
- **Predictive bottleneck detection** using composite metrics
- **Warehouse Gravity Zones** for optimal storage placement
- **Real-time dashboards** with role-based access control

The platform integrates data from WMS, TMS, telematics, supplier portals, and external APIs (weather, traffic) to provide unified supply chain visibility.

## Installation & Prerequisites

### Requirements

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) 18+
- Access to source systems: WMS, TMS, telemetry feeds

### Database Setup

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**
```sql
-- Connect to your SQL Server instance
-- Execute the main schema script
:r ./sql/01_CreateDatabase.sql
:r ./sql/02_CreateDimensions.sql
:r ./sql/03_CreateFacts.sql
:r ./sql/04_CreateViews.sql
:r ./sql/05_CreateStoredProcedures.sql
:r ./sql/06_CreateIndexes.sql
```

3. **Configure connection strings:**
```json
// config.json (create from config_sample.json)
{
  "sqlServer": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_SERVER_USER}",
    "password": "${SQL_SERVER_PASSWORD}",
    "port": 1433
  },
  "dataSources": {
    "wms": {
      "type": "REST_API",
      "endpoint": "${WMS_API_ENDPOINT}",
      "apiKey": "${WMS_API_KEY}"
    },
    "telematics": {
      "type": "ODBC",
      "connectionString": "${TELEMATICS_CONNECTION_STRING}"
    }
  },
  "refreshInterval": 900
}
```

## Key Database Objects

### Core Fact Tables

#### FactWarehouseOperations
```sql
-- Primary warehouse activity fact table
CREATE TABLE dbo.FactWarehouseOperations (
    OperationID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    OperationType VARCHAR(20) NOT NULL, -- 'RECEIVING','PUTAWAY','PICKING','PACKING','SHIPPING'
    DurationMinutes DECIMAL(10,2),
    DwellTimeHours DECIMAL(10,2),
    UnitsProcessed INT,
    OperatorID INT,
    ZoneGravityScore DECIMAL(5,2),
    BatchNumber VARCHAR(50),
    RecordedTimestamp DATETIME2 DEFAULT GETDATE()
);

-- Example query: Average dwell time by product category
SELECT 
    dp.ProductCategory,
    AVG(fwo.DwellTimeHours) AS AvgDwellTime,
    COUNT(*) AS OperationCount
FROM dbo.FactWarehouseOperations fwo
INNER JOIN dbo.DimProduct dp ON fwo.ProductKey = dp.ProductKey
WHERE fwo.OperationType = 'PUTAWAY'
    AND fwo.TimeKey >= CONVERT(INT, FORMAT(DATEADD(day, -30, GETDATE()), 'yyyyMMdd'))
GROUP BY dp.ProductCategory
ORDER BY AvgDwellTime DESC;
```

#### FactFleetTrips
```sql
-- Fleet telemetry and trip data
CREATE TABLE dbo.FactFleetTrips (
    TripID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    TripStartTime DATETIME2,
    TripEndTime DATETIME2,
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    LoadingTimeMinutes DECIMAL(10,2),
    UnloadingTimeMinutes DECIMAL(10,2),
    DriverID INT,
    WeatherCondition VARCHAR(50),
    DelayMinutes DECIMAL(10,2)
);

-- Example query: Fleet utilization by route
SELECT 
    dr.RouteName,
    COUNT(DISTINCT fft.VehicleKey) AS VehicleCount,
    SUM(fft.DistanceKM) AS TotalDistanceKM,
    AVG(fft.IdleTimeMinutes) AS AvgIdleTime,
    SUM(fft.FuelConsumedLiters) AS TotalFuelLiters
FROM dbo.FactFleetTrips fft
INNER JOIN dbo.DimRoute dr ON fft.RouteKey = dr.RouteKey
WHERE fft.TripStartTime >= DATEADD(day, -7, GETDATE())
GROUP BY dr.RouteName
ORDER BY TotalDistanceKM DESC;
```

### Key Dimension Tables

#### DimTime (Time-phased dimension)
```sql
CREATE TABLE dbo.DimTime (
    TimeKey INT PRIMARY KEY, -- Format: YYYYMMDDHHmm (e.g., 202607011430)
    FullDate DATE,
    Year INT,
    Quarter INT,
    Month INT,
    Week INT,
    DayOfMonth INT,
    DayOfWeek INT,
    Hour INT,
    Minute15Bucket INT, -- 0, 15, 30, 45
    IsWeekend BIT,
    FiscalYear INT,
    FiscalQuarter INT
);

-- Populate time dimension (15-minute granularity)
WITH TimeSeries AS (
    SELECT CAST('2025-01-01 00:00:00' AS DATETIME2) AS TimeValue
    UNION ALL
    SELECT DATEADD(MINUTE, 15, TimeValue)
    FROM TimeSeries
    WHERE TimeValue < '2027-12-31 23:45:00'
)
INSERT INTO dbo.DimTime (TimeKey, FullDate, Year, Quarter, Month, Week, DayOfMonth, DayOfWeek, Hour, Minute15Bucket, IsWeekend, FiscalYear, FiscalQuarter)
SELECT 
    CONVERT(INT, FORMAT(TimeValue, 'yyyyMMddHHmm')) AS TimeKey,
    CAST(TimeValue AS DATE) AS FullDate,
    YEAR(TimeValue) AS Year,
    DATEPART(QUARTER, TimeValue) AS Quarter,
    MONTH(TimeValue) AS Month,
    DATEPART(WEEK, TimeValue) AS Week,
    DAY(TimeValue) AS DayOfMonth,
    DATEPART(WEEKDAY, TimeValue) AS DayOfWeek,
    DATEPART(HOUR, TimeValue) AS Hour,
    (DATEPART(MINUTE, TimeValue) / 15) * 15 AS Minute15Bucket,
    CASE WHEN DATEPART(WEEKDAY, TimeValue) IN (1, 7) THEN 1 ELSE 0 END AS IsWeekend,
    CASE WHEN MONTH(TimeValue) >= 7 THEN YEAR(TimeValue) + 1 ELSE YEAR(TimeValue) END AS FiscalYear,
    CASE WHEN MONTH(TimeValue) BETWEEN 7 AND 9 THEN 1
         WHEN MONTH(TimeValue) BETWEEN 10 AND 12 THEN 2
         WHEN MONTH(TimeValue) BETWEEN 1 AND 3 THEN 3
         ELSE 4 END AS FiscalQuarter
FROM TimeSeries
OPTION (MAXRECURSION 0);
```

#### DimProductGravity (Warehouse Gravity Zones)
```sql
CREATE TABLE dbo.DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL,
    ProductName VARCHAR(200),
    ProductCategory VARCHAR(100),
    VelocityScore DECIMAL(5,2), -- Pick frequency normalized
    ValueScore DECIMAL(5,2), -- Unit price normalized
    FragilityScore DECIMAL(5,2), -- Handling complexity
    GravityScore AS (VelocityScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) PERSISTED,
    RecommendedZone VARCHAR(50), -- 'HIGH_GRAVITY', 'MEDIUM_GRAVITY', 'LOW_GRAVITY'
    LastUpdated DATETIME2 DEFAULT GETDATE()
);

-- Calculate and update gravity scores
CREATE PROCEDURE dbo.sp_UpdateProductGravity
AS
BEGIN
    UPDATE pg
    SET 
        VelocityScore = CAST(pick_freq.NormalizedFrequency * 100 AS DECIMAL(5,2)),
        ValueScore = CAST(price.NormalizedPrice * 100 AS DECIMAL(5,2)),
        RecommendedZone = CASE 
            WHEN (pick_freq.NormalizedFrequency * 0.5 + price.NormalizedPrice * 0.3) > 0.7 THEN 'HIGH_GRAVITY'
            WHEN (pick_freq.NormalizedFrequency * 0.5 + price.NormalizedPrice * 0.3) > 0.4 THEN 'MEDIUM_GRAVITY'
            ELSE 'LOW_GRAVITY'
        END,
        LastUpdated = GETDATE()
    FROM dbo.DimProductGravity pg
    CROSS APPLY (
        SELECT 
            (COUNT(*) - MIN(COUNT(*)) OVER()) * 1.0 / NULLIF(MAX(COUNT(*)) OVER() - MIN(COUNT(*)) OVER(), 0) AS NormalizedFrequency
        FROM dbo.FactWarehouseOperations
        WHERE ProductKey = pg.ProductKey
            AND OperationType = 'PICKING'
            AND TimeKey >= CONVERT(INT, FORMAT(DATEADD(day, -30, GETDATE()), 'yyyyMMdd'))
        GROUP BY ProductKey
    ) pick_freq
    CROSS APPLY (
        SELECT 
            (UnitPrice - MIN(UnitPrice) OVER()) * 1.0 / NULLIF(MAX(UnitPrice) OVER() - MIN(UnitPrice) OVER(), 0) AS NormalizedPrice
        FROM dbo.DimProduct
        WHERE ProductKey = pg.ProductKey
    ) price;
END;
```

## Data Loading Patterns

### Incremental ETL Stored Procedure
```sql
-- Incremental load from WMS staging table
CREATE PROCEDURE dbo.sp_LoadWarehouseOperations
    @LastLoadTimestamp DATETIME2 = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Default to last 24 hours if not specified
    IF @LastLoadTimestamp IS NULL
        SET @LastLoadTimestamp = DATEADD(hour, -24, GETDATE());
    
    -- Insert new operations
    INSERT INTO dbo.FactWarehouseOperations (
        TimeKey, ProductKey, WarehouseKey, OperationType,
        DurationMinutes, DwellTimeHours, UnitsProcessed,
        OperatorID, ZoneGravityScore, BatchNumber
    )
    SELECT 
        CONVERT(INT, FORMAT(stg.OperationTimestamp, 'yyyyMMddHHmm')) AS TimeKey,
        dp.ProductKey,
        dw.WarehouseKey,
        stg.OperationType,
        DATEDIFF(MINUTE, stg.StartTime, stg.EndTime) AS DurationMinutes,
        CASE WHEN stg.OperationType = 'PUTAWAY' 
             THEN DATEDIFF(MINUTE, stg.EndTime, stg.PickTime) / 60.0 
             ELSE NULL END AS DwellTimeHours,
        stg.UnitsProcessed,
        stg.OperatorID,
        dpg.GravityScore,
        stg.BatchNumber
    FROM dbo.StagingWarehouseOperations stg
    INNER JOIN dbo.DimProduct dp ON stg.SKU = dp.SKU
    INNER JOIN dbo.DimWarehouse dw ON stg.WarehouseCode = dw.WarehouseCode
    LEFT JOIN dbo.DimProductGravity dpg ON dp.ProductKey = dpg.ProductKey
    WHERE stg.OperationTimestamp > @LastLoadTimestamp
        AND NOT EXISTS (
            SELECT 1 FROM dbo.FactWarehouseOperations fwo
            WHERE fwo.BatchNumber = stg.BatchNumber
                AND fwo.ProductKey = dp.ProductKey
        );
    
    -- Update metadata
    INSERT INTO dbo.ETLLog (ProcedureName, RecordsProcessed, ExecutionTime)
    VALUES ('sp_LoadWarehouseOperations', @@ROWCOUNT, GETDATE());
END;
```

## Cross-Fact KPI Queries

### Dwell Time vs. Fleet Idle Cost
```sql
-- Correlate warehouse dwell time with downstream fleet idle costs
WITH WarehouseDwell AS (
    SELECT 
        fwo.ProductKey,
        dp.ProductCategory,
        AVG(fwo.DwellTimeHours) AS AvgDwellTime,
        COUNT(*) AS OperationCount
    FROM dbo.FactWarehouseOperations fwo
    INNER JOIN dbo.DimProduct dp ON fwo.ProductKey = dp.ProductKey
    WHERE fwo.TimeKey >= CONVERT(INT, FORMAT(DATEADD(day, -30, GETDATE()), 'yyyyMMdd'))
        AND fwo.OperationType IN ('PUTAWAY', 'PICKING')
    GROUP BY fwo.ProductKey, dp.ProductCategory
),
FleetIdle AS (
    SELECT 
        fft.OriginGeographyKey,
        dg.WarehouseCode,
        AVG(fft.IdleTimeMinutes) AS AvgIdleTime,
        SUM(fft.FuelConsumedLiters * 1.5) AS EstimatedIdleCost -- $1.50/liter fuel cost
    FROM dbo.FactFleetTrips fft
    INNER JOIN dbo.DimGeography dg ON fft.OriginGeographyKey = dg.GeographyKey
    WHERE fft.TripStartTime >= DATEADD(day, -30, GETDATE())
    GROUP BY fft.OriginGeographyKey, dg.WarehouseCode
)
SELECT 
    wd.ProductCategory,
    wd.AvgDwellTime,
    fi.AvgIdleTime,
    fi.EstimatedIdleCost,
    (wd.AvgDwellTime * fi.EstimatedIdleCost / NULLIF(fi.AvgIdleTime, 0)) AS DwellIdleCostCorrelation
FROM WarehouseDwell wd
CROSS JOIN FleetIdle fi
WHERE wd.OperationCount > 10
ORDER BY DwellIdleCostCorrelation DESC;
```

### Predictive Bottleneck Index
```sql
-- Calculate bottleneck probability score
CREATE VIEW dbo.vw_BottleneckIndex AS
WITH CapacityMetrics AS (
    SELECT 
        WarehouseKey,
        TimeKey,
        SUM(UnitsProcessed) AS TotalUnits,
        AVG(DurationMinutes) AS AvgDuration,
        COUNT(*) AS OperationCount
    FROM dbo.FactWarehouseOperations
    WHERE TimeKey >= CONVERT(INT, FORMAT(DATEADD(day, -7, GETDATE()), 'yyyyMMdd'))
    GROUP BY WarehouseKey, TimeKey
),
FleetLoad AS (
    SELECT 
        OriginGeographyKey,
        TimeKey,
        COUNT(*) AS TripCount,
        AVG(LoadingTimeMinutes) AS AvgLoadingTime
    FROM dbo.FactFleetTrips
    WHERE TripStartTime >= DATEADD(day, -7, GETDATE())
    GROUP BY OriginGeographyKey, TimeKey
)
SELECT 
    dw.WarehouseName,
    dt.FullDate,
    dt.Hour,
    cm.TotalUnits,
    cm.AvgDuration,
    fl.TripCount,
    fl.AvgLoadingTime,
    -- Bottleneck score: higher = more congestion risk
    (
        (cm.OperationCount * 1.0 / NULLIF(dw.MaxCapacity, 0)) * 0.4 +
        (cm.AvgDuration / NULLIF(dw.StandardDurationMinutes, 0)) * 0.3 +
        (fl.AvgLoadingTime / 30.0) * 0.3
    ) * 100 AS BottleneckScore
FROM CapacityMetrics cm
INNER JOIN dbo.DimWarehouse dw ON cm.WarehouseKey = dw.WarehouseKey
INNER JOIN dbo.DimTime dt ON cm.TimeKey = dt.TimeKey
LEFT JOIN FleetLoad fl ON dw.GeographyKey = fl.OriginGeographyKey AND cm.TimeKey = fl.TimeKey
WHERE cm.OperationCount > 0;
```

## Power BI Integration

### Connect Power BI to SQL Server
```powerquery
// M Query - Connect to LogiFleet database
let
    Source = Sql.Databases(#"${SQL_SERVER_HOST}"),
    LogiFleetDB = Source{[Name="LogiFleetPulse"]}[Data],
    
    // Load fact tables
    FactWarehouse = LogiFleetDB{[Schema="dbo",Item="FactWarehouseOperations"]}[Data],
    FactFleet = LogiFleetDB{[Schema="dbo",Item="FactFleetTrips"]}[Data],
    
    // Load dimensions
    DimTime = LogiFleetDB{[Schema="dbo",Item="DimTime"]}[Data],
    DimProduct = LogiFleetDB{[Schema="dbo",Item="DimProductGravity"]}[Data],
    DimGeography = LogiFleetDB{[Schema="dbo",Item="DimGeography"]}[Data]
in
    {FactWarehouse, FactFleet, DimTime, DimProduct, DimGeography}
```

### DAX Measures for Cross-Fact KPIs
```dax
// Total Dwell Time (Warehouse)
TotalDwellTime = 
SUM(FactWarehouseOperations[DwellTimeHours])

// Fleet Utilization Rate
FleetUtilizationRate = 
DIVIDE(
    SUM(FactFleetTrips[DistanceKM]) - SUM(FactFleetTrips[IdleTimeMinutes]) / 60,
    SUM(FactFleetTrips[DistanceKM]),
    0
) * 100

// Cross-Fact: Dwell Impact on Fleet Delay
DwellFleetImpactScore = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR AvgDelay = AVERAGE(FactFleetTrips[DelayMinutes])
RETURN
    IF(
        AvgDwell > 48,
        AvgDelay * 1.2,
        AvgDelay
    )

// Warehouse Gravity Zone Performance
GravityZoneEfficiency = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DurationMinutes]),
    FILTER(
        DimProductGravity,
        DimProductGravity[RecommendedZone] = "HIGH_GRAVITY"
    )
)
```

### Power BI Template Configuration

1. **Open the template:**
```bash
# Extract and open Power BI template
powershell -Command "Expand-Archive -Path './powerbi/LogiFleet_Pulse_Master.pbit' -DestinationPath './powerbi/template'"
```

2. **Update parameters:**
   - File → Options → Data source settings
   - Update SQL Server connection string
   - Configure refresh schedule (15-minute intervals recommended)

3. **Apply row-level security:**
```dax
// RLS Rule: Users see only their assigned warehouses
[WarehouseCode] IN (
    LOOKUPVALUE(
        UserWarehouseAccess[WarehouseCode],
        UserWarehouseAccess[UserEmail],
        USERNAME()
    )
)
```

## Alerting Configuration

### SQL Server Agent Job for Threshold Alerts
```sql
-- Create alert stored procedure
CREATE PROCEDURE dbo.sp_CheckBottleneckAlerts
AS
BEGIN
    DECLARE @AlertRecipients VARCHAR(500) = '${ALERT_EMAIL_RECIPIENTS}';
    
    -- Check for high bottleneck scores
    IF EXISTS (
        SELECT 1 FROM dbo.vw_BottleneckIndex
        WHERE BottleneckScore > 75
            AND FullDate >= CAST(GETDATE() AS DATE)
    )
    BEGIN
        DECLARE @AlertBody NVARCHAR(MAX);
        
        SELECT @AlertBody = STRING_AGG(
            CONCAT(
                'Warehouse: ', WarehouseName, 
                ' | Hour: ', Hour, 
                ' | Score: ', CAST(BottleneckScore AS VARCHAR(10))
            ), 
            CHAR(13) + CHAR(10)
        )
        FROM dbo.vw_BottleneckIndex
        WHERE BottleneckScore > 75
            AND FullDate >= CAST(GETDATE() AS DATE);
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = @AlertRecipients,
            @subject = 'ALERT: High Bottleneck Risk Detected',
            @body = @AlertBody;
    END;
END;

-- Schedule job (run every 15 minutes)
EXEC msdb.dbo.sp_add_job @job_name = 'LogiFleet_BottleneckMonitor';
EXEC msdb.dbo.sp_add_jobstep 
    @job_name = 'LogiFleet_BottleneckMonitor',
    @step_name = 'Check Alerts',
    @command = 'EXEC dbo.sp_CheckBottleneckAlerts',
    @database_name = 'LogiFleetPulse';
EXEC msdb.dbo.sp_add_schedule
    @schedule_name = 'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;
EXEC msdb.dbo.sp_attach_schedule 
    @job_name = 'LogiFleet_BottleneckMonitor',
    @schedule_name = 'Every15Minutes';
```

## External Data Integration

### Weather API Integration
```sql
-- Staging table for weather data
CREATE TABLE dbo.StagingWeatherData (
    GeographyKey INT,
    RecordedTimestamp DATETIME2,
    Temperature DECIMAL(5,2),
    Condition VARCHAR(50),
    WindSpeedKPH DECIMAL(5,2),
    Visibility VARCHAR(20)
);

-- Python script to fetch weather data
/*
import requests
import pyodbc
import os
from datetime import datetime

conn = pyodbc.connect(
    f"DRIVER={{ODBC Driver 17 for SQL Server}};"
    f"SERVER={os.getenv('SQL_SERVER_HOST')};"
    f"DATABASE=LogiFleetPulse;"
    f"UID={os.getenv('SQL_SERVER_USER')};"
    f"PWD={os.getenv('SQL_SERVER_PASSWORD')}"
)

weather_api_key = os.getenv('WEATHER_API_KEY')

# Fetch active geography locations
cursor = conn.cursor()
cursor.execute("SELECT GeographyKey, Latitude, Longitude FROM dbo.DimGeography WHERE IsActive = 1")

for row in cursor.fetchall():
    geo_key, lat, lon = row
    
    response = requests.get(
        f"https://api.weatherapi.com/v1/current.json",
        params={
            'key': weather_api_key,
            'q': f"{lat},{lon}"
        }
    )
    
    if response.status_code == 200:
        data = response.json()
        cursor.execute(
            "INSERT INTO dbo.StagingWeatherData (GeographyKey, RecordedTimestamp, Temperature, Condition, WindSpeedKPH, Visibility) VALUES (?, ?, ?, ?, ?, ?)",
            geo_key,
            datetime.now(),
            data['current']['temp_c'],
            data['current']['condition']['text'],
            data['current']['wind_kph'],
            data['current']['vis_km']
        )

conn.commit()
conn.close()
*/

-- Link weather data to fleet trips
ALTER TABLE dbo.FactFleetTrips ADD WeatherCondition VARCHAR(50);

UPDATE fft
SET WeatherCondition = swd.Condition
FROM dbo.FactFleetTrips fft
INNER JOIN dbo.StagingWeatherData swd 
    ON fft.OriginGeographyKey = swd.GeographyKey
    AND ABS(DATEDIFF(MINUTE, fft.TripStartTime, swd.RecordedTimestamp)) <= 30;
```

## Troubleshooting

### Performance Issues

**Problem:** Queries on FactWarehouseOperations taking >5 seconds

**Solution:** Add covering indexes
```sql
-- Index for time-based queries
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time_Product
ON dbo.FactWarehouseOperations (TimeKey, ProductKey)
INCLUDE (DwellTimeHours, UnitsProcessed, OperationType);

-- Index for product-based analysis
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product_Type
ON dbo.FactWarehouseOperations (ProductKey, OperationType)
INCLUDE (DurationMinutes, ZoneGravityScore);

-- Partition by month for large tables (>10M rows)
CREATE PARTITION FUNCTION PF_TimeKey_Monthly (INT)
AS RANGE RIGHT FOR VALUES (202601, 202602, 202603 /* ... */);

CREATE PARTITION SCHEME PS_TimeKey_Monthly
AS PARTITION PF_TimeKey_Monthly ALL TO ([PRIMARY]);

-- Recreate table on partition scheme
```

### Power BI Refresh Failures

**Problem:** Power BI refresh timing out

**Solution:** Implement incremental refresh
```powerquery
// Add RangeStart and RangeEnd parameters in Power BI
let
    Source = Sql.Database(
        #"${SQL_SERVER_HOST}", 
        "LogiFleetPulse",
        [
            Query="SELECT * FROM dbo.FactWarehouseOperations 
                   WHERE RecordedTimestamp >= @RangeStart 
                   AND RecordedTimestamp < @RangeEnd"
        ]
    )
in
    Source
```

### Missing Cross-Fact Relationships

**Problem:** DAX measures returning blank for cross-fact calculations

**Solution:** Verify dimension relationships
```sql
-- Check for orphaned records
SELECT 'FactWarehouseOperations' AS TableName, COUNT(*) AS OrphanCount
FROM dbo.FactWarehouseOperations fwo
LEFT JOIN dbo.DimTime dt ON fwo.TimeKey = dt.TimeKey
WHERE dt.TimeKey IS NULL

UNION ALL

SELECT 'FactFleetTrips', COUNT(*)
FROM dbo.FactFleetTrips fft
LEFT JOIN dbo.DimTime dt ON fft.TimeKey = dt.TimeKey
WHERE dt.TimeKey IS NULL;

-- Fix: Add missing dimension records or clean up facts
DELETE FROM dbo.FactWarehouseOperations
WHERE TimeKey NOT IN (SELECT TimeKey FROM dbo.DimTime);
```

### Data Freshness Issues

**Problem:** Dashboards showing stale data despite scheduled refresh

**Solution:** Verify ETL job execution
```sql
-- Check last successful load
SELECT 
    ProcedureName,
    MAX(ExecutionTime) AS LastRun,
    SUM(RecordsProcessed) AS TotalRecords
FROM dbo.ETLLog
WHERE ExecutionTime >= DATEADD(hour, -24, GETDATE())
GROUP BY ProcedureName
ORDER BY LastRun DESC;

-- Enable SQL Server Agent if jobs not running
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'Agent XPs', 1;
RECONFIGURE;
```

## Best Practices

1. **Partition large fact tables** by month when row count exceeds 10 million
2. **Use columnstore indexes** for analytical queries on fact tables with >50M rows
3. **Implement incremental refresh** in Power BI for tables updated frequently
4. **Schedule gravity score recalculation** weekly (computationally expensive)
5. **Archive historical data** older than 2 years to separate database
6. **Monitor SQL Server wait stats** to identify bottlenecks (PAGEIOLATCH, LCK_M_*)
7. **Use DirectQuery** for real-time dashboards, Import mode for historical analysis

## Common Usage Patterns

### Weekly Supply Chain Executive Report
```sql
EXEC dbo.sp_GenerateExecutiveReport 
    @StartDate = '2026-06-24',
    @EndDate = '2026-07-01',
    @OutputFormat = 'EMAIL';
```

### Real-Time Warehouse Heatmap Query
```sql
SELECT 
    dw.WarehouseName,
    dt.Hour,
    COUNT(*) AS ActiveOperations,
    AVG(fwo.DurationMinutes) AS AvgDuration,
    CASE 
        WHEN COUNT(*) > dw.MaxCapacity * 0.8 THEN 'RED'
        WHEN COUNT(*) > dw.MaxCapacity * 0.6 THEN 'YELLOW'
        ELSE 'GREEN'
    END AS StatusColor
FROM dbo.FactWarehouse
