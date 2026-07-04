---
name: logifleet-pulse-supply-chain-analytics
description: Advanced MS SQL Server & Power BI supply chain analytics platform with multi-fact star schema, warehouse optimization, and fleet telemetry integration
triggers:
  - "set up LogiFleet Pulse analytics warehouse"
  - "deploy supply chain analytics SQL schema"
  - "configure Power BI logistics dashboard"
  - "implement warehouse gravity zones"
  - "create fleet telemetry tracking"
  - "build cross-fact supply chain KPIs"
  - "optimize logistics data model"
  - "integrate warehouse management system analytics"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a multi-modal logistics intelligence engine that combines:
- **MS SQL Server data warehouse** with multi-fact star schema
- **Power BI dashboards** for real-time supply chain visualization
- **Warehouse gravity zone optimization** (spatial analysis of storage efficiency)
- **Fleet telemetry integration** (GPS, fuel consumption, idle time tracking)
- **Cross-fact KPI harmonization** (unified metrics across warehouse, fleet, and supplier data)
- **Predictive bottleneck detection** using temporal elasticity modeling

The platform connects disparate logistics data sources (WMS, telematics, ERP, external APIs) into a unified semantic layer for decision support.

## Installation & Setup

### Prerequisites

```bash
# Required software
- MS SQL Server 2019+ (Standard or Enterprise edition)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) 18+
```

### Database Deployment

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy SQL schema:**
```sql
-- Connect to your SQL Server instance via SSMS
-- Run the master schema script
:r .\sql\schema\01_create_database.sql
:r .\sql\schema\02_dimension_tables.sql
:r .\sql\schema\03_fact_tables.sql
:r .\sql\schema\04_bridge_tables.sql
:r .\sql\schema\05_indexes_constraints.sql
:r .\sql\schema\06_stored_procedures.sql
:r .\sql\schema\07_views.sql
```

3. **Configure data sources:**
```json
// Update config.json with your connection strings
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_SERVER_USER}",
    "password": "${SQL_SERVER_PASSWORD}"
  },
  "data_sources": {
    "wms_api": "${WMS_API_ENDPOINT}",
    "telematics_api": "${FLEET_TELEMETRY_ENDPOINT}",
    "weather_api": "${WEATHER_API_KEY}"
  }
}
```

4. **Import Power BI template:**
- Open `LogiFleet_Pulse_Master.pbit`
- Connect to your SQL Server instance
- Configure refresh schedule

## Key SQL Schema Components

### Dimension Tables

```sql
-- DimTime: 15-minute granularity time dimension
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    Year SMALLINT,
    Quarter TINYINT,
    Month TINYINT,
    Day TINYINT,
    Hour TINYINT,
    Minute TINYINT,
    DayOfWeek TINYINT,
    FiscalPeriod SMALLINT,
    IsWeekend BIT,
    IsHoliday BIT
);

-- DimProductGravity: Products with calculated gravity score
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU NVARCHAR(50) NOT NULL,
    ProductName NVARCHAR(200),
    Category NVARCHAR(100),
    GravityScore DECIMAL(5,2), -- Calculated from velocity, value, fragility
    PickFrequency DECIMAL(10,2),
    UnitValue MONEY,
    FragilityIndex TINYINT,
    OptimalZone NVARCHAR(20)
);

-- DimGeography: Hierarchical location dimension
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY,
    LocationID NVARCHAR(50),
    LocationName NVARCHAR(200),
    LocationType NVARCHAR(50), -- Warehouse, Route Node, Distribution Center
    ParentGeographyKey INT,
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    Region NVARCHAR(100),
    Country NVARCHAR(100)
);
```

### Fact Tables

```sql
-- FactWarehouseOperations: Granular warehouse events
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    OperationType NVARCHAR(50), -- Receiving, Putaway, Picking, Packing, Shipping
    Quantity INT,
    DwellTimeMinutes INT,
    PickRateUnitsPerHour DECIMAL(10,2),
    ZoneTemperature DECIMAL(5,2),
    HandlerID NVARCHAR(50)
);

-- FactFleetTrips: Fleet telemetry and route data
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY,
    StartTimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    EndTimeKey INT FOREIGN KEY REFERENCES DimTime(EndTimeKey),
    OriginGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID NVARCHAR(50),
    DriverID NVARCHAR(50),
    DistanceKM DECIMAL(10,2),
    FuelLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadWeightKG DECIMAL(10,2),
    AverageSpeedKPH DECIMAL(5,2),
    DelayMinutes INT,
    DelayReason NVARCHAR(200)
);

-- FactCrossDock: Cross-dock transfer events
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    TransferTimeMinutes INT,
    Quantity INT,
    TemperatureBreachFlag BIT
);
```

## Core Stored Procedures

### Incremental Data Loading

```sql
-- Incremental load for warehouse operations
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LoadDate DATE = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    IF @LoadDate IS NULL
        SET @LoadDate = CAST(GETDATE() AS DATE);
    
    -- ETL logic with surrogate key lookups
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, GeographyKey,
        OperationType, Quantity, DwellTimeMinutes,
        PickRateUnitsPerHour, ZoneTemperature, HandlerID
    )
    SELECT 
        dt.TimeKey,
        dp.ProductKey,
        dg.GeographyKey,
        src.OperationType,
        src.Quantity,
        DATEDIFF(MINUTE, src.StartTime, src.EndTime) AS DwellTimeMinutes,
        CASE WHEN DATEDIFF(MINUTE, src.StartTime, src.EndTime) > 0 
             THEN src.Quantity * 60.0 / DATEDIFF(MINUTE, src.StartTime, src.EndTime)
             ELSE 0 END AS PickRateUnitsPerHour,
        src.ZoneTemperature,
        src.HandlerID
    FROM staging.WarehouseOperations_Raw src
    INNER JOIN DimTime dt 
        ON dt.FullDateTime = DATEADD(MINUTE, (DATEDIFF(MINUTE, 0, src.StartTime) / 15) * 15, 0)
    INNER JOIN DimProductGravity dp 
        ON dp.SKU = src.SKU
    INNER JOIN DimGeography dg 
        ON dg.LocationID = src.WarehouseZone
    WHERE CAST(src.StartTime AS DATE) = @LoadDate
        AND NOT EXISTS (
            SELECT 1 FROM FactWarehouseOperations f
            WHERE f.TimeKey = dt.TimeKey
                AND f.ProductKey = dp.ProductKey
                AND f.GeographyKey = dg.GeographyKey
        );
END;
```

### Gravity Zone Calculation

```sql
-- Calculate and update product gravity scores
CREATE PROCEDURE usp_CalculateProductGravity
    @AnalysisPeriodDays INT = 90
AS
BEGIN
    -- Weighted scoring: 50% velocity, 30% value, 20% fragility
    UPDATE DimProductGravity
    SET GravityScore = (
        (PickFrequency / NULLIF(MaxPick, 0) * 50) +
        (UnitValue / NULLIF(MaxValue, 0) * 30) +
        (FragilityIndex / 10.0 * 20)
    ),
    OptimalZone = CASE 
        WHEN GravityScore >= 80 THEN 'HIGH-GRAVITY'
        WHEN GravityScore >= 50 THEN 'MEDIUM-GRAVITY'
        ELSE 'LOW-GRAVITY'
    END
    FROM (
        SELECT 
            ProductKey,
            SUM(Quantity) * 1.0 / @AnalysisPeriodDays AS PickFrequency,
            MAX(SUM(Quantity)) OVER () AS MaxPick,
            MAX(UnitValue) OVER () AS MaxValue
        FROM FactWarehouseOperations
        WHERE TimeKey >= (SELECT MIN(TimeKey) FROM DimTime WHERE FullDateTime >= DATEADD(DAY, -@AnalysisPeriodDays, GETDATE()))
        GROUP BY ProductKey
    ) freq
    WHERE DimProductGravity.ProductKey = freq.ProductKey;
END;
```

### Cross-Fact KPI Queries

```sql
-- Fleet idle cost correlated with warehouse dwell time
CREATE VIEW vw_DwellTimeVsFleetIdleCost AS
SELECT 
    dt.Year,
    dt.Month,
    dp.Category,
    AVG(w.DwellTimeMinutes) AS AvgWarehouseDwellMinutes,
    AVG(f.IdleTimeMinutes) AS AvgFleetIdleMinutes,
    AVG(f.FuelLiters * 1.5) AS AvgIdleFuelCostUSD, -- Assuming $1.50/liter
    CORR(w.DwellTimeMinutes, f.IdleTimeMinutes) AS CorrelationCoefficient
FROM FactWarehouseOperations w
INNER JOIN DimTime dt ON w.TimeKey = dt.TimeKey
INNER JOIN DimProductGravity dp ON w.ProductKey = dp.ProductKey
LEFT JOIN FactFleetTrips f 
    ON f.StartTimeKey = w.TimeKey
    AND f.OriginGeographyKey = w.GeographyKey
GROUP BY dt.Year, dt.Month, dp.Category;
```

## Power BI Integration Patterns

### DAX Measures for Cross-Fact Analysis

```dax
// Total Dwell Time (Warehouse)
Total Dwell Time = 
SUMX(
    FactWarehouseOperations,
    FactWarehouseOperations[DwellTimeMinutes]
)

// Fleet Utilization Rate
Fleet Utilization % = 
DIVIDE(
    SUM(FactFleetTrips[DistanceKM]),
    SUM(FactFleetTrips[DistanceKM]) + (SUM(FactFleetTrips[IdleTimeMinutes]) / 60 * 50), 
    // Assuming 50 KPH average speed during idle conversion
    0
) * 100

// Gravity Zone Compliance
Gravity Zone Compliance % = 
VAR CurrentPlacements = 
    SUMMARIZE(
        FactWarehouseOperations,
        DimProductGravity[SKU],
        DimGeography[LocationName],
        "ActualZone", FIRSTNONBLANK(DimGeography[LocationType], 1)
    )
RETURN
    DIVIDE(
        COUNTROWS(
            FILTER(
                CurrentPlacements,
                DimProductGravity[OptimalZone] = [ActualZone]
            )
        ),
        COUNTROWS(CurrentPlacements),
        0
    ) * 100

// Predictive Bottleneck Score
Bottleneck Risk Score = 
VAR HighDwellThreshold = 120 // minutes
VAR HighIdleThreshold = 20 // percentage
VAR CurrentDwell = [Total Dwell Time] / COUNTROWS(FactWarehouseOperations)
VAR CurrentIdle = [Fleet Utilization %]
RETURN
    (CurrentDwell / HighDwellThreshold * 50) +
    ((100 - CurrentIdle) / HighIdleThreshold * 50)
```

### Report Configuration Example

```json
// Power BI refresh configuration (via Power BI Service API)
{
  "refreshSchedule": {
    "frequency": "15min",
    "enabled": true,
    "localTimeZoneId": "UTC"
  },
  "gatewayConnection": {
    "gatewayObjectId": "${GATEWAY_ID}",
    "datasourceObjectId": "${DATASOURCE_ID}"
  },
  "rowLevelSecurity": {
    "roles": [
      {
        "name": "WarehouseManager",
        "filterExpression": "[DimGeography].[Region] = USERNAME()"
      },
      {
        "name": "FleetSupervisor",
        "filterExpression": "[FactFleetTrips].[DriverID] IN (SELECT DriverID FROM SecurityMapping WHERE Manager = USERNAME())"
      }
    ]
  }
}
```

## Common Configuration Patterns

### Alert Configuration

```sql
-- Set up proactive alerts for threshold breaches
CREATE PROCEDURE usp_CheckAndAlertThresholds
AS
BEGIN
    -- Fleet idle time alert
    IF EXISTS (
        SELECT 1 
        FROM FactFleetTrips
        WHERE EndTimeKey >= (SELECT TOP 1 TimeKey FROM DimTime ORDER BY TimeKey DESC)
            AND (IdleTimeMinutes * 100.0 / NULLIF(DATEDIFF(MINUTE, 
                (SELECT FullDateTime FROM DimTime WHERE TimeKey = StartTimeKey),
                (SELECT FullDateTime FROM DimTime WHERE TimeKey = EndTimeKey)), 0)) > 15
    )
    BEGIN
        -- Send alert (integrate with SQL Server Database Mail or external API)
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL}',
            @subject = 'Fleet Idle Time Threshold Exceeded',
            @body = 'One or more vehicles exceeded 15% idle time threshold.';
    END
    
    -- Warehouse dwell time alert
    INSERT INTO AlertLog (AlertType, Severity, Message, CreatedAt)
    SELECT 
        'HIGH_DWELL_TIME',
        'WARNING',
        'SKU ' + dp.SKU + ' has exceeded 72 hours dwell time',
        GETDATE()
    FROM FactWarehouseOperations w
    INNER JOIN DimProductGravity dp ON w.ProductKey = dp.ProductKey
    WHERE w.DwellTimeMinutes > 4320 -- 72 hours
        AND w.TimeKey >= (SELECT TOP 1 TimeKey FROM DimTime ORDER BY TimeKey DESC);
END;
```

### External API Integration (Python Example)

```python
import pyodbc
import requests
import os
from datetime import datetime

# Database connection
conn_str = (
    f"DRIVER={{ODBC Driver 17 for SQL Server}};"
    f"SERVER={os.getenv('SQL_SERVER_HOST')};"
    f"DATABASE=LogiFleetPulse;"
    f"UID={os.getenv('SQL_SERVER_USER')};"
    f"PWD={os.getenv('SQL_SERVER_PASSWORD')}"
)
conn = pyodbc.connect(conn_str)
cursor = conn.cursor()

# Fetch pending fleet trips
cursor.execute("""
    SELECT TripKey, OriginGeographyKey, DestinationGeographyKey, 
           VehicleID, DriverID
    FROM FactFleetTrips
    WHERE EndTimeKey IS NULL
""")

trips = cursor.fetchall()

# Enrich with real-time traffic data
for trip in trips:
    origin_lat, origin_lon = get_coordinates(trip.OriginGeographyKey)
    dest_lat, dest_lon = get_coordinates(trip.DestinationGeographyKey)
    
    # Call traffic API
    traffic_response = requests.get(
        f"{os.getenv('TRAFFIC_API_ENDPOINT')}/route",
        params={
            'origin': f"{origin_lat},{origin_lon}",
            'destination': f"{dest_lat},{dest_lon}",
            'key': os.getenv('TRAFFIC_API_KEY')
        }
    )
    
    if traffic_response.status_code == 200:
        data = traffic_response.json()
        estimated_delay = data.get('duration_in_traffic', 0) - data.get('duration', 0)
        
        # Update trip with predicted delay
        cursor.execute("""
            UPDATE FactFleetTrips
            SET DelayMinutes = ?,
                DelayReason = 'TRAFFIC_CONGESTION'
            WHERE TripKey = ?
        """, estimated_delay / 60, trip.TripKey)

conn.commit()
conn.close()

def get_coordinates(geography_key):
    cursor.execute("""
        SELECT Latitude, Longitude
        FROM DimGeography
        WHERE GeographyKey = ?
    """, geography_key)
    return cursor.fetchone()
```

## Troubleshooting

### Performance Optimization

```sql
-- Add columnstore index for fact tables (analytics workload)
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouseOperations
ON FactWarehouseOperations (
    TimeKey, ProductKey, GeographyKey, Quantity, DwellTimeMinutes
);

-- Partition large fact tables by time
CREATE PARTITION FUNCTION PF_TimeKey (INT)
AS RANGE RIGHT FOR VALUES (
    20260101, 20260201, 20260301, 20260401, 20260501, 20260601
);

CREATE PARTITION SCHEME PS_TimeKey
AS PARTITION PF_TimeKey
ALL TO ([PRIMARY]);

-- Rebuild table on partition scheme
CREATE TABLE FactWarehouseOperations_Partitioned (
    /* same schema */
) ON PS_TimeKey(TimeKey);
```

### Data Quality Checks

```sql
-- Identify orphaned records (missing dimension keys)
SELECT 'FactWarehouseOperations' AS TableName, COUNT(*) AS OrphanCount
FROM FactWarehouseOperations w
WHERE NOT EXISTS (SELECT 1 FROM DimProductGravity WHERE ProductKey = w.ProductKey)

UNION ALL

SELECT 'FactFleetTrips', COUNT(*)
FROM FactFleetTrips f
WHERE NOT EXISTS (SELECT 1 FROM DimGeography WHERE GeographyKey = f.OriginGeographyKey);

-- Detect anomalous dwell times
SELECT ProductKey, AVG(DwellTimeMinutes) AS AvgDwell,
       STDEV(DwellTimeMinutes) AS StdDev,
       MAX(DwellTimeMinutes) AS MaxDwell
FROM FactWarehouseOperations
GROUP BY ProductKey
HAVING MAX(DwellTimeMinutes) > AVG(DwellTimeMinutes) + (3 * STDEV(DwellTimeMinutes));
```

### Power BI Refresh Failures

```powershell
# Test database connectivity from Power BI gateway
Test-NetConnection -ComputerName $env:SQL_SERVER_HOST -Port 1433

# Check gateway logs
Get-Content "C:\Program Files\On-premises data gateway\GatewayLogs.log" -Tail 50

# Verify service account permissions
Invoke-Sqlcmd -ServerInstance $env:SQL_SERVER_HOST -Database LogiFleetPulse -Query "
    SELECT HAS_PERMS_BY_NAME(NULL, NULL, 'VIEW SERVER STATE') AS ViewServerState,
           HAS_PERMS_BY_NAME('LogiFleetPulse', 'DATABASE', 'SELECT') AS DatabaseSelect
"
```

## Real-World Usage Example

```sql
-- Complete workflow: Daily gravity zone optimization
-- Step 1: Load yesterday's warehouse operations
EXEC usp_LoadWarehouseOperations @LoadDate = CAST(GETDATE() - 1 AS DATE);

-- Step 2: Recalculate product gravity scores
EXEC usp_CalculateProductGravity @AnalysisPeriodDays = 90;

-- Step 3: Generate zone reassignment recommendations
SELECT 
    dp.SKU,
    dp.ProductName,
    dp.GravityScore,
    dp.OptimalZone AS RecommendedZone,
    dg.LocationName AS CurrentZone,
    COUNT(*) AS RecentPicks,
    AVG(w.DwellTimeMinutes) AS AvgDwellTime
FROM FactWarehouseOperations w
INNER JOIN DimProductGravity dp ON w.ProductKey = dp.ProductKey
INNER JOIN DimGeography dg ON w.GeographyKey = dg.GeographyKey
WHERE w.TimeKey >= (SELECT MIN(TimeKey) FROM DimTime WHERE FullDateTime >= DATEADD(DAY, -7, GETDATE()))
    AND dg.LocationType <> dp.OptimalZone
GROUP BY dp.SKU, dp.ProductName, dp.GravityScore, dp.OptimalZone, dg.LocationName
HAVING COUNT(*) > 10
ORDER BY dp.GravityScore DESC;

-- Step 4: Check for alert conditions
EXEC usp_CheckAndAlertThresholds;

-- Step 5: Refresh Power BI dataset (via API or scheduled refresh)
```

This skill enables AI agents to deploy, configure, and optimize the LogiFleet Pulse supply chain analytics platform with proper multi-fact data modeling and cross-modal KPI analysis.
