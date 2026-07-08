---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server and Power BI data warehousing template for logistics, fleet management, and supply chain analytics with multi-fact star schema
triggers:
  - set up logistics analytics dashboard
  - implement supply chain data warehouse
  - create fleet management Power BI reports
  - build warehouse operations analytics
  - deploy logistics intelligence system
  - configure multi-fact star schema for supply chain
  - integrate fleet telemetry with warehouse data
  - design logistics KPI dashboard
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive data warehousing and analytics template for logistics and supply chain operations. It combines MS SQL Server multi-fact star schema with Power BI dashboards to provide unified visibility across warehouse operations, fleet management, inventory tracking, and cross-dock operations.

**Key Capabilities:**
- Multi-fact star schema with time-phased dimensions
- Cross-fact KPI harmonization (warehouse + fleet + inventory)
- Real-time operational dashboards (15-minute refresh cycles)
- Predictive bottleneck detection
- Warehouse gravity zone optimization
- Fleet triage and maintenance prioritization
- Role-based access control with row-level security

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to data sources (WMS, TMS, telemetry APIs, ERP)

### Deployment Steps

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy SQL schema:**
```sql
-- Connect to your SQL Server instance via SSMS
-- Execute schema creation script
:r schema\01_create_database.sql
:r schema\02_create_dimensions.sql
:r schema\03_create_facts.sql
:r schema\04_create_views.sql
:r schema\05_create_stored_procedures.sql
```

3. **Configure data sources:**
```json
// Update config_sample.json with your connection details
{
  "sqlServer": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_SERVER_USER}",
    "password": "${SQL_SERVER_PASSWORD}"
  },
  "dataSources": {
    "wmsApi": "${WMS_API_ENDPOINT}",
    "telemetryApi": "${FLEET_TELEMETRY_ENDPOINT}",
    "erpConnection": "${ERP_CONNECTION_STRING}"
  }
}
```

4. **Import Power BI template:**
- Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
- Enter SQL Server connection details when prompted
- Configure refresh schedule (recommended: every 15 minutes)

## Core Database Schema

### Fact Tables

**FactWarehouseOperations** - Core warehouse activity tracking
```sql
CREATE TABLE FactWarehouseOperations (
    OperationID INT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    WarehouseKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    DwellTimeMinutes INT,
    PickingTimeSeconds INT,
    PackingTimeSeconds INT,
    UnitsHandled INT,
    OperationCost DECIMAL(10,2),
    StaffKey INT FOREIGN KEY REFERENCES DimStaff(StaffKey)
);

-- Index for performance
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time_Product 
ON FactWarehouseOperations(TimeKey, ProductKey) 
INCLUDE (DwellTimeMinutes, UnitsHandled);
```

**FactFleetTrips** - Fleet telemetry and trip data
```sql
CREATE TABLE FactFleetTrips (
    TripID INT PRIMARY KEY IDENTITY(1,1),
    StartTimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    EndTimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleKey INT FOREIGN KEY REFERENCES DimVehicle(VehicleKey),
    RouteKey INT FOREIGN KEY REFERENCES DimRoute(RouteKey),
    OriginGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DistanceMiles DECIMAL(8,2),
    FuelConsumedGallons DECIMAL(6,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    TripCost DECIMAL(10,2),
    WeatherImpactFlag BIT
);
```

**FactCrossDock** - Cross-docking operations
```sql
CREATE TABLE FactCrossDock (
    CrossDockID INT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    InboundTripKey INT FOREIGN KEY REFERENCES FactFleetTrips(TripID),
    OutboundTripKey INT FOREIGN KEY REFERENCES FactFleetTrips(TripID),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    CrossDockTimeMinutes INT,
    UnitsTransferred INT,
    DockBayKey INT FOREIGN KEY REFERENCES DimDockBay(DockBayKey)
);
```

### Dimension Tables

**DimTime** - Time dimension with 15-minute granularity
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    Year INT,
    Quarter INT,
    Month INT,
    Week INT,
    DayOfMonth INT,
    DayOfWeek INT,
    DayName VARCHAR(10),
    Hour INT,
    MinuteBucket INT, -- 0, 15, 30, 45
    IsWeekend BIT,
    FiscalPeriod VARCHAR(10)
);

-- Populate time dimension
CREATE PROCEDURE PopulateTimeDimension
    @StartDate DATETIME,
    @EndDate DATETIME
AS
BEGIN
    DECLARE @CurrentDate DATETIME = @StartDate;
    WHILE @CurrentDate <= @EndDate
    BEGIN
        INSERT INTO DimTime (TimeKey, FullDateTime, Year, Quarter, Month, Week, 
                            DayOfMonth, DayOfWeek, DayName, Hour, MinuteBucket, IsWeekend)
        SELECT 
            CAST(FORMAT(@CurrentDate, 'yyyyMMddHHmm') AS INT),
            @CurrentDate,
            YEAR(@CurrentDate),
            DATEPART(QUARTER, @CurrentDate),
            MONTH(@CurrentDate),
            DATEPART(WEEK, @CurrentDate),
            DAY(@CurrentDate),
            DATEPART(WEEKDAY, @CurrentDate),
            DATENAME(WEEKDAY, @CurrentDate),
            DATEPART(HOUR, @CurrentDate),
            (DATEPART(MINUTE, @CurrentDate) / 15) * 15,
            CASE WHEN DATEPART(WEEKDAY, @CurrentDate) IN (1,7) THEN 1 ELSE 0 END;
        
        SET @CurrentDate = DATEADD(MINUTE, 15, @CurrentDate);
    END
END;
```

**DimProductGravity** - Product with calculated gravity score
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    ProductSKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    VelocityScore INT, -- Pick frequency ranking
    ValueScore INT, -- Margin contribution
    FragilityScore INT, -- Handling care required
    GravityScore AS (VelocityScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2),
    RecommendedZoneType VARCHAR(50), -- 'High-Gravity', 'Medium-Gravity', 'Low-Gravity'
    CurrentWarehouseZone VARCHAR(50),
    LeadTimeDays INT
);
```

**DimGeography** - Hierarchical location dimension
```sql
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    GeographyType VARCHAR(50), -- 'Warehouse', 'Route Node', 'Customer Location'
    LocationName VARCHAR(200),
    Address VARCHAR(500),
    City VARCHAR(100),
    State VARCHAR(50),
    Country VARCHAR(50),
    Region VARCHAR(50),
    Continent VARCHAR(50),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    TimeZone VARCHAR(50)
);
```

## Key Stored Procedures

### Incremental Data Loading

```sql
CREATE PROCEDURE LoadWarehouseOperations
    @LastLoadDateTime DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Get last load time if not provided
    IF @LastLoadDateTime IS NULL
        SELECT @LastLoadDateTime = ISNULL(MAX(LoadDateTime), '2000-01-01')
        FROM LoadControl WHERE TableName = 'FactWarehouseOperations';
    
    -- Load new operations from staging
    INSERT INTO FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductKey, OperationType,
        DwellTimeMinutes, PickingTimeSeconds, PackingTimeSeconds,
        UnitsHandled, OperationCost, StaffKey
    )
    SELECT 
        CAST(FORMAT(s.OperationDateTime, 'yyyyMMddHHmm') AS INT),
        g.GeographyKey,
        p.ProductKey,
        s.OperationType,
        DATEDIFF(MINUTE, s.StartTime, s.EndTime),
        s.PickingSeconds,
        s.PackingSeconds,
        s.Units,
        s.Cost,
        st.StaffKey
    FROM StagingWarehouseOperations s
    INNER JOIN DimGeography g ON s.WarehouseCode = g.LocationName
    INNER JOIN DimProductGravity p ON s.SKU = p.ProductSKU
    LEFT JOIN DimStaff st ON s.EmployeeID = st.EmployeeID
    WHERE s.OperationDateTime > @LastLoadDateTime
        AND s.ProcessedFlag = 0;
    
    -- Update load control
    UPDATE LoadControl 
    SET LoadDateTime = GETDATE(), 
        RowsLoaded = @@ROWCOUNT
    WHERE TableName = 'FactWarehouseOperations';
    
    -- Mark staging as processed
    UPDATE StagingWarehouseOperations 
    SET ProcessedFlag = 1 
    WHERE OperationDateTime > @LastLoadDateTime;
END;
```

### Cross-Fact KPI Calculation

```sql
CREATE PROCEDURE CalculateCrossFactKPIs
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    -- Calculate warehouse efficiency with fleet impact
    SELECT 
        t.Year,
        t.Month,
        g.Region,
        p.Category,
        -- Warehouse metrics
        AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
        SUM(wo.UnitsHandled) AS TotalUnitsHandled,
        AVG(wo.PickingTimeSeconds) AS AvgPickingTime,
        -- Fleet metrics
        AVG(ft.IdleTimeMinutes) AS AvgFleetIdleTime,
        AVG(ft.FuelConsumedGallons / NULLIF(ft.DistanceMiles, 0)) AS AvgFuelEfficiency,
        -- Cross-fact KPI: Warehouse to Fleet correlation
        CASE 
            WHEN AVG(wo.DwellTimeMinutes) > 240 
                AND AVG(ft.IdleTimeMinutes) > 30 
            THEN 'High Risk'
            WHEN AVG(wo.DwellTimeMinutes) > 120 
                OR AVG(ft.IdleTimeMinutes) > 15 
            THEN 'Medium Risk'
            ELSE 'Normal'
        END AS BottleneckRiskLevel
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    INNER JOIN DimGeography g ON wo.WarehouseKey = g.GeographyKey
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    LEFT JOIN FactFleetTrips ft 
        ON ft.OriginGeographyKey = wo.WarehouseKey
        AND ft.StartTimeKey BETWEEN t.TimeKey - 100 AND t.TimeKey + 100
    WHERE CAST(t.FullDateTime AS DATE) BETWEEN @StartDate AND @EndDate
    GROUP BY t.Year, t.Month, g.Region, p.Category
    ORDER BY BottleneckRiskLevel DESC, AvgDwellTime DESC;
END;
```

### Predictive Bottleneck Detection

```sql
CREATE PROCEDURE DetectPredictiveBottlenecks
AS
BEGIN
    -- Identify potential bottlenecks based on trending patterns
    WITH TrendingMetrics AS (
        SELECT 
            wo.WarehouseKey,
            p.ProductKey,
            DATEPART(HOUR, t.FullDateTime) AS HourOfDay,
            AVG(wo.DwellTimeMinutes) AS AvgDwell,
            STDEV(wo.DwellTimeMinutes) AS StdDevDwell,
            COUNT(*) AS SampleSize
        FROM FactWarehouseOperations wo
        INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
        INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
        WHERE t.FullDateTime >= DATEADD(DAY, -30, GETDATE())
        GROUP BY wo.WarehouseKey, p.ProductKey, DATEPART(HOUR, t.FullDateTime)
        HAVING COUNT(*) > 10
    )
    SELECT 
        g.LocationName AS Warehouse,
        p.ProductName,
        tm.HourOfDay,
        tm.AvgDwell,
        tm.StdDevDwell,
        p.GravityScore,
        CASE 
            WHEN tm.AvgDwell > 180 AND p.GravityScore > 70 THEN 'Critical'
            WHEN tm.AvgDwell > 120 AND p.GravityScore > 50 THEN 'High'
            WHEN tm.AvgDwell > 60 THEN 'Medium'
            ELSE 'Low'
        END AS BottleneckSeverity,
        CASE 
            WHEN p.CurrentWarehouseZone <> p.RecommendedZoneType 
            THEN 'Rezone to ' + p.RecommendedZoneType
            ELSE 'Increase staffing during hour ' + CAST(tm.HourOfDay AS VARCHAR)
        END AS RecommendedAction
    FROM TrendingMetrics tm
    INNER JOIN DimGeography g ON tm.WarehouseKey = g.GeographyKey
    INNER JOIN DimProductGravity p ON tm.ProductKey = p.ProductKey
    WHERE tm.AvgDwell > 60
    ORDER BY 
        CASE 
            WHEN tm.AvgDwell > 180 AND p.GravityScore > 70 THEN 1
            WHEN tm.AvgDwell > 120 AND p.GravityScore > 50 THEN 2
            WHEN tm.AvgDwell > 60 THEN 3
            ELSE 4
        END,
        tm.AvgDwell DESC;
END;
```

## Power BI Integration

### DAX Measures for Cross-Fact Analysis

```dax
// Composite efficiency score combining warehouse and fleet
Composite Efficiency Score = 
VAR WarehouseEfficiency = 
    1 - (
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]) / 
        MAX(FactWarehouseOperations[DwellTimeMinutes])
    )
VAR FleetEfficiency = 
    1 - (
        AVERAGE(FactFleetTrips[IdleTimeMinutes]) / 
        MAX(FactFleetTrips[IdleTimeMinutes])
    )
RETURN
    (WarehouseEfficiency * 0.6 + FleetEfficiency * 0.4) * 100

// Gravity zone optimization flag
Gravity Zone Misalignment = 
CALCULATE(
    COUNTROWS(DimProductGravity),
    DimProductGravity[CurrentWarehouseZone] <> DimProductGravity[RecommendedZoneType]
)

// Fleet idle cost by warehouse dwell correlation
Idle Cost per Dwell Hour = 
VAR TotalIdleCost = 
    SUMX(
        FactFleetTrips,
        FactFleetTrips[IdleTimeMinutes] / 60 * 45  // $45/hour idle cost
    )
VAR TotalDwellHours = 
    SUM(FactWarehouseOperations[DwellTimeMinutes]) / 60
RETURN
    DIVIDE(TotalIdleCost, TotalDwellHours, 0)

// Predictive bottleneck index
Bottleneck Risk Index = 
VAR CurrentDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR HistoricalAvg = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        DATESINPERIOD(DimTime[FullDateTime], LASTDATE(DimTime[FullDateTime]), -30, DAY)
    )
VAR DwellTrend = DIVIDE(CurrentDwell - HistoricalAvg, HistoricalAvg, 0)
VAR FleetIdle = AVERAGE(FactFleetTrips[IdleTimeMinutes])
RETURN
    (DwellTrend * 0.6 + (FleetIdle / 60) * 0.4) * 100
```

### Power BI Dashboard Configuration

```javascript
// Example: Connecting to SQL Server from Power BI
// File > Options > Data source settings
{
  "dataSource": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "windows",  // or "sql" with username/password
    "connectionTimeout": 30,
    "commandTimeout": 120
  },
  "refreshSchedule": {
    "frequency": "minutes",
    "interval": 15,
    "enabled": true
  }
}
```

## Common Patterns

### Pattern 1: Real-Time Alert Configuration

```sql
-- Create alert monitoring procedure
CREATE PROCEDURE MonitorCriticalKPIs
AS
BEGIN
    DECLARE @AlertThreshold INT = 15; -- Fleet idle time threshold in minutes
    DECLARE @DwellThreshold INT = 240; -- Warehouse dwell threshold in minutes
    
    -- Check for critical fleet idle time
    INSERT INTO AlertLog (AlertType, Severity, Message, EntityKey, DetectedAt)
    SELECT 
        'Fleet Idle Exceeded',
        'High',
        'Vehicle ' + v.VehicleName + ' idle for ' + CAST(ft.IdleTimeMinutes AS VARCHAR) + ' minutes',
        ft.VehicleKey,
        GETDATE()
    FROM FactFleetTrips ft
    INNER JOIN DimVehicle v ON ft.VehicleKey = v.VehicleKey
    WHERE ft.EndTimeKey >= CAST(FORMAT(DATEADD(HOUR, -1, GETDATE()), 'yyyyMMddHHmm') AS INT)
        AND ft.IdleTimeMinutes > @AlertThreshold;
    
    -- Check for excessive warehouse dwell
    INSERT INTO AlertLog (AlertType, Severity, Message, EntityKey, DetectedAt)
    SELECT 
        'Dwell Time Exceeded',
        CASE 
            WHEN wo.DwellTimeMinutes > @DwellThreshold * 2 THEN 'Critical'
            ELSE 'Medium'
        END,
        'Product ' + p.ProductSKU + ' at ' + g.LocationName + ' dwelling for ' + 
            CAST(wo.DwellTimeMinutes AS VARCHAR) + ' minutes',
        wo.ProductKey,
        GETDATE()
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    INNER JOIN DimGeography g ON wo.WarehouseKey = g.GeographyKey
    WHERE wo.TimeKey >= CAST(FORMAT(DATEADD(HOUR, -2, GETDATE()), 'yyyyMMddHHmm') AS INT)
        AND wo.DwellTimeMinutes > @DwellThreshold
        AND p.GravityScore > 60;
END;

-- Schedule this to run every 15 minutes via SQL Agent
```

### Pattern 2: Warehouse Gravity Zone Recalculation

```sql
-- Recalculate product gravity scores and recommend zone changes
CREATE PROCEDURE RecalculateProductGravity
AS
BEGIN
    -- Update velocity scores based on last 30 days
    WITH ProductVelocity AS (
        SELECT 
            ProductKey,
            COUNT(*) AS PickCount,
            NTILE(100) OVER (ORDER BY COUNT(*) DESC) AS VelocityPercentile
        FROM FactWarehouseOperations
        WHERE TimeKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm') AS INT)
            AND OperationType = 'Picking'
        GROUP BY ProductKey
    )
    UPDATE p
    SET 
        p.VelocityScore = pv.VelocityPercentile,
        p.RecommendedZoneType = 
            CASE 
                WHEN pv.VelocityPercentile >= 80 AND p.ValueScore >= 70 THEN 'High-Gravity'
                WHEN pv.VelocityPercentile >= 50 OR p.ValueScore >= 50 THEN 'Medium-Gravity'
                ELSE 'Low-Gravity'
            END
    FROM DimProductGravity p
    INNER JOIN ProductVelocity pv ON p.ProductKey = pv.ProductKey;
    
    -- Generate rezone recommendations
    SELECT 
        p.ProductSKU,
        p.ProductName,
        p.CurrentWarehouseZone,
        p.RecommendedZoneType,
        p.GravityScore,
        COUNT(wo.OperationID) AS Last30DayPicks,
        AVG(wo.PickingTimeSeconds) AS AvgPickTime
    FROM DimProductGravity p
    LEFT JOIN FactWarehouseOperations wo 
        ON p.ProductKey = wo.ProductKey
        AND wo.TimeKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm') AS INT)
        AND wo.OperationType = 'Picking'
    WHERE p.CurrentWarehouseZone <> p.RecommendedZoneType
    GROUP BY p.ProductSKU, p.ProductName, p.CurrentWarehouseZone, 
             p.RecommendedZoneType, p.GravityScore
    ORDER BY p.GravityScore DESC, Last30DayPicks DESC;
END;
```

### Pattern 3: External Data Integration (Weather/Traffic)

```sql
-- Create external table for weather data (requires PolyBase)
CREATE EXTERNAL DATA SOURCE WeatherAPISource
WITH (
    TYPE = EXTERNAL_GENERIC,
    LOCATION = '${WEATHER_API_ENDPOINT}'
);

CREATE EXTERNAL TABLE ExternalWeather (
    WeatherDateTime DATETIME,
    LocationKey INT,
    Temperature DECIMAL(5,2),
    Precipitation DECIMAL(5,2),
    WindSpeed DECIMAL(5,2),
    VisibilityMiles DECIMAL(5,2),
    WeatherCondition VARCHAR(100)
)
WITH (
    LOCATION = '/weather/current',
    DATA_SOURCE = WeatherAPISource,
    FILE_FORMAT = JSONFormat
);

-- Correlate weather with fleet delays
CREATE PROCEDURE AnalyzeWeatherImpact
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    SELECT 
        ft.TripID,
        v.VehicleName,
        r.RouteName,
        t.FullDateTime AS TripStart,
        ft.DistanceMiles,
        ft.IdleTimeMinutes,
        ew.WeatherCondition,
        ew.Precipitation,
        CASE 
            WHEN ew.Precipitation > 0.5 OR ew.VisibilityMiles < 5 
            THEN CAST(ft.IdleTimeMinutes * 0.3 AS INT)  -- Estimate weather-caused delay
            ELSE 0
        END AS EstimatedWeatherDelayMinutes
    FROM FactFleetTrips ft
    INNER JOIN DimTime t ON ft.StartTimeKey = t.TimeKey
    INNER JOIN DimVehicle v ON ft.VehicleKey = v.VehicleKey
    INNER JOIN DimRoute r ON ft.RouteKey = r.RouteKey
    LEFT JOIN ExternalWeather ew 
        ON CAST(t.FullDateTime AS DATE) = CAST(ew.WeatherDateTime AS DATE)
        AND ft.OriginGeographyKey = ew.LocationKey
    WHERE CAST(t.FullDateTime AS DATE) BETWEEN @StartDate AND @EndDate
        AND ft.WeatherImpactFlag = 1
    ORDER BY EstimatedWeatherDelayMinutes DESC;
END;
```

## Configuration

### Row-Level Security Setup

```sql
-- Create security roles
CREATE ROLE WarehouseManager;
CREATE ROLE FleetManager;
CREATE ROLE ExecutiveViewer;

-- Create security predicates
CREATE FUNCTION SecurityPredicateWarehouse(@WarehouseKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN 
    SELECT 1 AS AccessPermitted
    WHERE 
        @WarehouseKey IN (
            SELECT WarehouseKey 
            FROM dbo.UserWarehouseAccess 
            WHERE UserName = USER_NAME()
        )
        OR IS_MEMBER('ExecutiveViewer') = 1;

-- Apply security policy
CREATE SECURITY POLICY WarehouseSecurityPolicy
ADD FILTER PREDICATE dbo.SecurityPredicateWarehouse(WarehouseKey)
ON dbo.FactWarehouseOperations
WITH (STATE = ON);

-- Grant permissions
GRANT SELECT ON DimProductGravity TO WarehouseManager;
GRANT SELECT ON FactWarehouseOperations TO WarehouseManager;
GRANT SELECT ON FactFleetTrips TO FleetManager;
GRANT SELECT ON SCHEMA::dbo TO ExecutiveViewer;
```

### Environment Variables Configuration

```bash
# .env file for configuration
SQL_SERVER_HOST=your-sql-server.database.windows.net
SQL_SERVER_DATABASE=LogiFleetPulse
SQL_SERVER_USER=logifleet_app
SQL_SERVER_PASSWORD=${SQL_SERVER_PASSWORD}

WMS_API_ENDPOINT=https://your-wms-api.com/v1
WMS_API_KEY=${WMS_API_KEY}

FLEET_TELEMETRY_ENDPOINT=https://your-fleet-api.com/telemetry
FLEET_API_KEY=${FLEET_API_KEY}

ERP_CONNECTION_STRING=Server=${ERP_SERVER};Database=${ERP_DB};User Id=${ERP_USER};Password=${ERP_PASSWORD};

WEATHER_API_ENDPOINT=https://api.weather.com/v3
WEATHER_API_KEY=${WEATHER_API_KEY}

POWERBI_WORKSPACE_ID=${POWERBI_WORKSPACE_ID}
POWERBI_REPORT_ID=${POWERBI_REPORT_ID}
```

## Troubleshooting

### Issue: Slow Query Performance on Cross-Fact Queries

**Symptom:** Queries joining FactWarehouseOperations and FactFleetTrips take >30 seconds

**Solution:**
```sql
-- Create indexed views for common cross-fact queries
CREATE VIEW vw_WarehouseFleetCorrelation
WITH SCHEMABINDING
AS
SELECT 
    wo.WarehouseKey,
    wo.TimeKey,
    wo.ProductKey,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
    COUNT_BIG(*) AS OperationCount,
    SUM(wo.UnitsHandled) AS TotalUnits
FROM dbo.FactWarehouseOperations wo
GROUP BY wo.WarehouseKey, wo.TimeKey, wo.ProductKey;

CREATE UNIQUE CLUSTERED INDEX IX_WarehouseFleet 
ON vw_WarehouseFleetCorrelation(WarehouseKey, TimeKey, ProductKey);

-- Use the indexed view in queries
SELECT 
    vwf.*,
    AVG(ft.IdleTimeMinutes) AS AvgIdleTime
FROM vw_WarehouseFleetCorrelation vwf
LEFT JOIN FactFleetTrips ft 
    ON ft.OriginGeographyKey = vwf.WarehouseKey
    AND ft.StartTimeKey BETWEEN vwf.TimeKey - 100 AND vwf.TimeKey + 100
GROUP BY vwf.WarehouseKey, vwf.TimeKey, vwf.ProductKey, 
         vwf.AvgDwellTime, vwf.OperationCount, vwf.TotalUnits;
```

### Issue: Power BI Gateway Timeout on Refresh

**Symptom:** Scheduled refresh fails with "Query timeout expired"

**Solution:**
```sql
-- Partition large fact tables by date
CREATE PARTITION FUNCTION pfTimeKey (INT)
AS RANGE RIGHT FOR VALUES 
    (202601010000, 202602010000, 202603010000); -- Monthly partitions

CREATE PARTITION SCHEME psTimeKey
AS PARTITION pfTimeKey
ALL TO ([PRIMARY]);

-- Recreate fact table with partitioning
CREATE TABLE FactWarehouseOperations_Partitioned (
    OperationID INT IDENTITY(1,1),
    TimeKey INT,
    -- other columns...
    CONSTRAINT PK_FactWarehouse_Partitioned PRIMARY KEY (OperationID, TimeKey)
) ON psTimeKey(TimeKey);

-- In Power BI, use incremental refresh settings:
-- Range: Last 90 days
-- Archive: Store data older than 90 days
```

### Issue: Gravity Score Not Updating

**Symptom:** DimProductGravity.GravityScore remains static despite operational changes

**Solution:**
```sql
-- GravityScore is a computed column, update the source columns
UPDATE DimProductGravity
SET 
    VelocityScore = (
        SELECT NTILE(100) OVER (ORDER BY PickCount DESC)
        FROM (
            SELECT ProductKey, COUNT(*) AS PickCount
            FROM FactWarehouseOperations
            WHERE OperationType = 'Picking'
