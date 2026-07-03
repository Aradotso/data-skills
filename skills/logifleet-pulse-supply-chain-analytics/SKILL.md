---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server data warehouse and Power BI dashboards for multi-modal logistics intelligence, fleet optimization, and warehouse analytics
triggers:
  - "set up LogiFleet Pulse logistics analytics"
  - "deploy supply chain data warehouse schema"
  - "configure Power BI logistics dashboards"
  - "integrate warehouse and fleet data sources"
  - "query cross-fact logistics KPIs"
  - "implement warehouse gravity zones"
  - "create fleet optimization reports"
  - "troubleshoot Power BI logistics model"
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

LogiFleet Pulse is an advanced MS SQL Server data warehouse and Power BI visualization platform for supply chain analytics. It unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a multi-fact star schema with time-phased dimensions. The platform enables cross-modal KPI analysis, predictive bottleneck detection, and warehouse optimization through "gravity zones."

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Network access to your WMS, TMS, or ERP systems

### Deploy the SQL Schema

1. **Clone or download the repository:**

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Execute the main schema deployment script:**

```sql
-- Connect to your SQL Server instance via SSMS
-- Run the schema creation script
:r schema/01_create_database.sql
:r schema/02_create_dimensions.sql
:r schema/03_create_facts.sql
:r schema/04_create_views.sql
:r schema/05_create_stored_procedures.sql
```

3. **Configure data source connections:**

```json
// config.json
{
  "data_sources": {
    "wms_connection": "$(WMS_CONNECTION_STRING)",
    "fleet_telemetry_api": "$(FLEET_API_ENDPOINT)",
    "erp_database": "$(ERP_DB_CONNECTION)",
    "weather_api_key": "$(WEATHER_API_KEY)"
  },
  "refresh_interval_minutes": 15,
  "alert_email": "$(ALERT_EMAIL)",
  "smtp_server": "$(SMTP_SERVER)"
}
```

### Import Power BI Template

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. Enter your SQL Server connection details when prompted
3. Configure row-level security roles in Power BI
4. Publish to Power BI Service for sharing and scheduled refresh

## Core Data Model

### Fact Tables

**FactWarehouseOperations** - Warehouse micro-operations tracking:

```sql
CREATE TABLE FactWarehouseOperations (
    OperationID INT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    WarehouseKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    DwellTimeMinutes INT,
    PickRate DECIMAL(10,2), -- Items per hour
    PackingTimeSeconds INT,
    ZoneID VARCHAR(20),
    StaffID INT,
    CreatedDate DATETIME2 DEFAULT GETDATE()
);

CREATE INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey);
CREATE INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey);
```

**FactFleetTrips** - Fleet telemetry and trip data:

```sql
CREATE TABLE FactFleetTrips (
    TripID INT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleKey INT FOREIGN KEY REFERENCES DimVehicle(VehicleKey),
    RouteKey INT FOREIGN KEY REFERENCES DimRoute(RouteKey),
    OriginGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    TotalDistanceKM DECIMAL(10,2),
    AverageSpeedKPH DECIMAL(5,2),
    DelayMinutes INT,
    DelayReason VARCHAR(100),
    TripStartDateTime DATETIME2,
    TripEndDateTime DATETIME2
);

CREATE INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey);
CREATE INDEX IX_FactFleet_Route ON FactFleetTrips(RouteKey);
```

### Dimension Tables

**DimTime** - Time dimension with 15-minute granularity:

```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2,
    DateOnly DATE,
    TimeOnly TIME,
    Year INT,
    Quarter INT,
    Month INT,
    Week INT,
    DayOfMonth INT,
    DayOfWeek INT,
    DayName VARCHAR(20),
    Hour INT,
    Minute15Bucket INT, -- 0, 15, 30, 45
    IsWeekend BIT,
    IsFiscalYearEnd BIT,
    FiscalPeriod VARCHAR(10)
);

-- Populate time dimension
DECLARE @StartDate DATETIME2 = '2020-01-01';
DECLARE @EndDate DATETIME2 = '2030-12-31';
DECLARE @CurrentTime DATETIME2 = @StartDate;

WHILE @CurrentTime <= @EndDate
BEGIN
    INSERT INTO DimTime (TimeKey, FullDateTime, DateOnly, TimeOnly, Year, Quarter, Month, Week, DayOfMonth, DayOfWeek, DayName, Hour, Minute15Bucket, IsWeekend)
    VALUES (
        CAST(FORMAT(@CurrentTime, 'yyyyMMddHHmm') AS INT),
        @CurrentTime,
        CAST(@CurrentTime AS DATE),
        CAST(@CurrentTime AS TIME),
        YEAR(@CurrentTime),
        DATEPART(QUARTER, @CurrentTime),
        MONTH(@CurrentTime),
        DATEPART(WEEK, @CurrentTime),
        DAY(@CurrentTime),
        DATEPART(WEEKDAY, @CurrentTime),
        DATENAME(WEEKDAY, @CurrentTime),
        DATEPART(HOUR, @CurrentTime),
        (DATEPART(MINUTE, @CurrentTime) / 15) * 15,
        CASE WHEN DATEPART(WEEKDAY, @CurrentTime) IN (1, 7) THEN 1 ELSE 0 END
    );
    
    SET @CurrentTime = DATEADD(MINUTE, 15, @CurrentTime);
END;
```

**DimProductGravity** - Product dimension with gravity scoring:

```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    VelocityScore DECIMAL(5,2), -- Picks per day
    ValueScore DECIMAL(10,2), -- Unit price
    FragilityScore INT, -- 1-10 scale
    GravityScore AS (VelocityScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) PERSISTED,
    OptimalZoneType VARCHAR(50), -- 'High-Gravity', 'Medium-Gravity', 'Low-Gravity'
    ReplenishmentLeadTimeDays INT,
    StorageRequirements VARCHAR(100), -- 'Ambient', 'Refrigerated', 'Frozen', 'Hazardous'
    LastUpdated DATETIME2 DEFAULT GETDATE()
);

CREATE INDEX IX_Product_Gravity ON DimProductGravity(GravityScore DESC);
CREATE INDEX IX_Product_SKU ON DimProductGravity(SKU);
```

## Key SQL Queries and Patterns

### Cross-Fact KPI Analysis

**Query: Correlate dwell time with fleet idle costs**

```sql
-- Find SKUs with high dwell time and their impact on fleet efficiency
WITH WarehouseDwell AS (
    SELECT 
        p.SKU,
        p.ProductName,
        p.GravityScore,
        AVG(w.DwellTimeMinutes) AS AvgDwellTime,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations w
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.DateOnly >= DATEADD(DAY, -30, GETDATE())
    GROUP BY p.SKU, p.ProductName, p.GravityScore
),
FleetImpact AS (
    SELECT 
        p.SKU,
        AVG(f.IdleTimeMinutes) AS AvgIdleTime,
        AVG(f.LoadingTimeMinutes + f.UnloadingTimeMinutes) AS AvgHandlingTime,
        SUM(f.FuelConsumedLiters * 1.5) AS EstimatedIdleCost -- $1.5 per liter
    FROM FactFleetTrips f
    INNER JOIN FactWarehouseOperations w ON f.OriginGeographyKey = w.WarehouseKey
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.DateOnly >= DATEADD(DAY, -30, GETDATE())
    GROUP BY p.SKU
)
SELECT 
    wd.SKU,
    wd.ProductName,
    wd.GravityScore,
    wd.AvgDwellTime,
    fi.AvgIdleTime,
    fi.AvgHandlingTime,
    fi.EstimatedIdleCost,
    (wd.AvgDwellTime * fi.AvgIdleTime) / NULLIF(wd.GravityScore, 0) AS ImpactRatio
FROM WarehouseDwell wd
LEFT JOIN FleetImpact fi ON wd.SKU = fi.SKU
WHERE wd.AvgDwellTime > 72 -- More than 72 hours
ORDER BY ImpactRatio DESC;
```

### Warehouse Gravity Zone Optimization

**Stored Procedure: Recommend zone rearrangements**

```sql
CREATE PROCEDURE sp_RecommendZoneRearrangements
AS
BEGIN
    -- Identify products in wrong gravity zones
    WITH CurrentPlacements AS (
        SELECT 
            p.ProductKey,
            p.SKU,
            p.ProductName,
            p.GravityScore,
            p.OptimalZoneType,
            w.ZoneID AS CurrentZone,
            AVG(w.DwellTimeMinutes) AS AvgDwellTime,
            COUNT(*) AS PickFrequency
        FROM FactWarehouseOperations w
        INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
        INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
        WHERE t.DateOnly >= DATEADD(DAY, -90, GETDATE())
            AND w.OperationType IN ('Picking', 'Packing')
        GROUP BY p.ProductKey, p.SKU, p.ProductName, p.GravityScore, p.OptimalZoneType, w.ZoneID
    ),
    ZoneAnalysis AS (
        SELECT 
            cp.*,
            CASE 
                WHEN cp.GravityScore > 7.5 AND cp.CurrentZone NOT LIKE 'HG%' THEN 'Move to High-Gravity'
                WHEN cp.GravityScore BETWEEN 4.0 AND 7.5 AND cp.CurrentZone NOT LIKE 'MG%' THEN 'Move to Medium-Gravity'
                WHEN cp.GravityScore < 4.0 AND cp.CurrentZone NOT LIKE 'LG%' THEN 'Move to Low-Gravity'
                ELSE 'Correctly Placed'
            END AS Recommendation,
            (cp.PickFrequency * cp.AvgDwellTime) AS WasteCost
        FROM CurrentPlacements cp
    )
    SELECT 
        SKU,
        ProductName,
        GravityScore,
        CurrentZone,
        Recommendation,
        AvgDwellTime,
        PickFrequency,
        WasteCost
    FROM ZoneAnalysis
    WHERE Recommendation <> 'Correctly Placed'
    ORDER BY WasteCost DESC;
END;
```

### Predictive Bottleneck Detection

**Query: Identify probable congestion points**

```sql
-- Detect warehouse zones with rising congestion trends
WITH WeeklyMetrics AS (
    SELECT 
        w.ZoneID,
        DATEPART(WEEK, t.DateOnly) AS WeekNumber,
        AVG(w.DwellTimeMinutes) AS AvgDwellTime,
        AVG(w.PickRate) AS AvgPickRate,
        COUNT(*) AS OperationVolume
    FROM FactWarehouseOperations w
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.DateOnly >= DATEADD(DAY, -60, GETDATE())
    GROUP BY w.ZoneID, DATEPART(WEEK, t.DateOnly)
),
TrendAnalysis AS (
    SELECT 
        ZoneID,
        WeekNumber,
        AvgDwellTime,
        AvgPickRate,
        OperationVolume,
        LAG(AvgDwellTime, 1) OVER (PARTITION BY ZoneID ORDER BY WeekNumber) AS PrevWeekDwell,
        LAG(OperationVolume, 1) OVER (PARTITION BY ZoneID ORDER BY WeekNumber) AS PrevWeekVolume
    FROM WeeklyMetrics
)
SELECT 
    ZoneID,
    WeekNumber,
    AvgDwellTime,
    PrevWeekDwell,
    ((AvgDwellTime - PrevWeekDwell) / NULLIF(PrevWeekDwell, 0)) * 100 AS DwellTimeChangePercent,
    OperationVolume,
    PrevWeekVolume,
    ((OperationVolume - PrevWeekVolume) / NULLIF(PrevWeekVolume, 0)) * 100 AS VolumeChangePercent,
    CASE 
        WHEN ((AvgDwellTime - PrevWeekDwell) / NULLIF(PrevWeekDwell, 0)) > 0.15 
            AND ((OperationVolume - PrevWeekVolume) / NULLIF(PrevWeekVolume, 0)) > 0.10 
        THEN 'High Risk'
        WHEN ((AvgDwellTime - PrevWeekDwell) / NULLIF(PrevWeekDwell, 0)) > 0.10 
        THEN 'Medium Risk'
        ELSE 'Low Risk'
    END AS BottleneckRisk
FROM TrendAnalysis
WHERE PrevWeekDwell IS NOT NULL
    AND ((AvgDwellTime - PrevWeekDwell) / NULLIF(PrevWeekDwell, 0)) > 0.05
ORDER BY BottleneckRisk, DwellTimeChangePercent DESC;
```

### Fleet Maintenance Priority Queue

**Query: Rank maintenance needs by revenue impact**

```sql
-- Generate proactive maintenance queue with revenue impact scoring
WITH VehicleHealth AS (
    SELECT 
        v.VehicleKey,
        v.VehicleID,
        v.TirePressurePSI,
        v.EngineDiagnosticCode,
        v.LastMaintenanceDate,
        DATEDIFF(DAY, v.LastMaintenanceDate, GETDATE()) AS DaysSinceService
    FROM DimVehicle v
),
RevenueImpact AS (
    SELECT 
        f.VehicleKey,
        COUNT(*) AS TripsLast30Days,
        SUM(f.TotalDistanceKM) AS TotalKM,
        AVG(f.IdleTimeMinutes) AS AvgIdleTime,
        SUM(CASE WHEN f.DelayMinutes > 30 THEN 1 ELSE 0 END) AS DelayedTrips,
        -- Assume $50 revenue per successful on-time trip
        COUNT(*) * 50 AS PotentialRevenue
    FROM FactFleetTrips f
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.DateOnly >= DATEADD(DAY, -30, GETDATE())
    GROUP BY f.VehicleKey
)
SELECT 
    vh.VehicleID,
    vh.DaysSinceService,
    vh.TirePressurePSI,
    vh.EngineDiagnosticCode,
    ri.TripsLast30Days,
    ri.TotalKM,
    ri.AvgIdleTime,
    ri.DelayedTrips,
    ri.PotentialRevenue,
    CASE 
        WHEN vh.TirePressurePSI < 30 OR vh.EngineDiagnosticCode IS NOT NULL THEN 'Critical'
        WHEN vh.DaysSinceService > 180 THEN 'High'
        WHEN vh.DaysSinceService > 90 THEN 'Medium'
        ELSE 'Low'
    END AS MaintenancePriority,
    (ri.PotentialRevenue * 0.001 * vh.DaysSinceService) + 
        (ri.DelayedTrips * 100) AS RiskScore
FROM VehicleHealth vh
INNER JOIN RevenueImpact ri ON vh.VehicleKey = ri.VehicleKey
ORDER BY RiskScore DESC, MaintenancePriority;
```

## Power BI Configuration

### Row-Level Security Setup

```dax
-- Create RLS role for regional managers (only see their warehouse)
[GeographyRegion] = USERNAME()

-- Create RLS role for product category managers
[ProductCategory] IN USERPRINCIPALNAME()
```

### Key DAX Measures

**Cross-Fact Composite KPI:**

```dax
Fleet Efficiency Index = 
VAR TotalTrips = COUNTROWS(FactFleetTrips)
VAR AvgIdleTime = AVERAGE(FactFleetTrips[IdleTimeMinutes])
VAR AvgDwellTime = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR IdleRatio = DIVIDE(AvgIdleTime, 60, 0) -- Convert to hours
VAR DwellRatio = DIVIDE(AvgDwellTime, 1440, 0) -- Convert to days
RETURN
    100 - ((IdleRatio * 30) + (DwellRatio * 70)) -- Weighted efficiency score
```

**Temporal Elasticity Simulation:**

```dax
Projected Capacity Impact = 
VAR CurrentCapacity = SUM(FactWarehouseOperations[OperationVolume])
VAR TargetCapacity = CurrentCapacity * 1.15 -- 15% increase
VAR HistoricalElasticity = 
    CALCULATE(
        AVERAGE(FactFleetTrips[IdleTimeMinutes]),
        FILTER(
            ALL(DimTime),
            DimTime[DateOnly] >= DATE(2025, 1, 1) &&
            DimTime[DateOnly] <= DATE(2025, 12, 31)
        )
    )
RETURN
    TargetCapacity * (HistoricalElasticity / 100)
```

**Warehouse Gravity Zone Score:**

```dax
Gravity Efficiency = 
VAR HighGravityPicks = 
    CALCULATE(
        SUM(FactWarehouseOperations[OperationCount]),
        DimProductGravity[GravityScore] > 7.5,
        FactWarehouseOperations[ZoneID] IN {"HG-A", "HG-B", "HG-C"}
    )
VAR TotalPicks = SUM(FactWarehouseOperations[OperationCount])
RETURN
    DIVIDE(HighGravityPicks, TotalPicks, 0) * 100
```

## Automated Alerting

### Configure Email Alerts for KPI Breaches

```sql
CREATE PROCEDURE sp_MonitorKPIThresholds
AS
BEGIN
    DECLARE @AlertBody NVARCHAR(MAX);
    DECLARE @Subject NVARCHAR(200);
    
    -- Check for excessive fleet idle time (> 15% of trip duration)
    IF EXISTS (
        SELECT 1 
        FROM FactFleetTrips f
        INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
        WHERE t.DateOnly = CAST(GETDATE() AS DATE)
        GROUP BY f.VehicleKey
        HAVING AVG(CAST(f.IdleTimeMinutes AS FLOAT) / NULLIF(DATEDIFF(MINUTE, f.TripStartDateTime, f.TripEndDateTime), 0)) > 0.15
    )
    BEGIN
        SET @Subject = 'ALERT: Fleet Idle Time Exceeded Threshold';
        SET @AlertBody = 'One or more vehicles exceeded 15% idle time ratio today. Review fleet performance dashboard.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '$(ALERT_EMAIL)',
            @subject = @Subject,
            @body = @AlertBody,
            @body_format = 'HTML';
    END
    
    -- Check for warehouse dwell time anomalies
    IF EXISTS (
        SELECT 1
        FROM FactWarehouseOperations w
        INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
        WHERE t.DateOnly >= DATEADD(DAY, -1, GETDATE())
        GROUP BY w.ZoneID
        HAVING AVG(w.DwellTimeMinutes) > 120 -- 2+ hours avg dwell
    )
    BEGIN
        SET @Subject = 'ALERT: Warehouse Dwell Time Spike Detected';
        SET @AlertBody = 'Multiple zones showing elevated dwell times. Investigate staffing and process bottlenecks.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '$(ALERT_EMAIL)',
            @subject = @Subject,
            @body = @AlertBody,
            @body_format = 'HTML';
    END
END;

-- Schedule to run every 15 minutes
EXEC sp_add_schedule
    @schedule_name = 'KPI_Monitoring_15min',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;
```

## Data Integration Patterns

### Incremental Loading from WMS

```sql
CREATE PROCEDURE sp_IncrementalLoadWarehouseOps
    @LastLoadDateTime DATETIME2
AS
BEGIN
    -- Upsert pattern for incremental warehouse operations
    MERGE INTO FactWarehouseOperations AS Target
    USING (
        SELECT 
            t.TimeKey,
            g.GeographyKey,
            p.ProductKey,
            ext.OperationType,
            ext.DwellTimeMinutes,
            ext.PickRate,
            ext.PackingTimeSeconds,
            ext.ZoneID,
            ext.StaffID,
            ext.CreatedDate
        FROM ExternalWMSData ext -- Linked server or staging table
        INNER JOIN DimTime t ON CAST(ext.OperationDateTime AS DATETIME2) = t.FullDateTime
        INNER JOIN DimGeography g ON ext.WarehouseCode = g.WarehouseCode
        INNER JOIN DimProductGravity p ON ext.SKU = p.SKU
        WHERE ext.CreatedDate > @LastLoadDateTime
    ) AS Source
    ON Target.OperationID = Source.OperationID
    WHEN MATCHED THEN
        UPDATE SET 
            Target.DwellTimeMinutes = Source.DwellTimeMinutes,
            Target.PickRate = Source.PickRate,
            Target.PackingTimeSeconds = Source.PackingTimeSeconds
    WHEN NOT MATCHED BY TARGET THEN
        INSERT (TimeKey, WarehouseKey, ProductKey, OperationType, DwellTimeMinutes, PickRate, PackingTimeSeconds, ZoneID, StaffID, CreatedDate)
        VALUES (Source.TimeKey, Source.GeographyKey, Source.ProductKey, Source.OperationType, Source.DwellTimeMinutes, Source.PickRate, Source.PackingTimeSeconds, Source.ZoneID, Source.StaffID, Source.CreatedDate);
        
    -- Update last load timestamp
    UPDATE ETL_Config SET LastLoadDateTime = GETDATE() WHERE TableName = 'FactWarehouseOperations';
END;
```

### Real-Time Fleet Telemetry Ingestion

```sql
-- Create external table for streaming telemetry (requires PolyBase or Azure Synapse Link)
CREATE EXTERNAL TABLE FleetTelemetryStream (
    VehicleID VARCHAR(50),
    TelemetryDateTime DATETIME2,
    Latitude DECIMAL(10, 6),
    Longitude DECIMAL(10, 6),
    SpeedKPH DECIMAL(5, 2),
    FuelLevelPercent DECIMAL(5, 2),
    EngineRPM INT,
    TirePressurePSI DECIMAL(5, 2)
)
WITH (
    LOCATION = '$(FLEET_API_ENDPOINT)',
    DATA_SOURCE = FleetTelemetryAPI,
    FILE_FORMAT = JSONFormat
);

-- Transform and load into fact table
INSERT INTO FactFleetTrips (TimeKey, VehicleKey, RouteKey, FuelConsumedLiters, TotalDistanceKM, AverageSpeedKPH, TripStartDateTime, TripEndDateTime)
SELECT 
    t.TimeKey,
    v.VehicleKey,
    r.RouteKey,
    SUM((100 - fts.FuelLevelPercent) * v.TankCapacityLiters / 100) AS FuelConsumed,
    SUM(fts.SpeedKPH * (1.0 / 60)) AS TotalKM, -- Simplified distance calc
    AVG(fts.SpeedKPH) AS AvgSpeed,
    MIN(fts.TelemetryDateTime) AS TripStart,
    MAX(fts.TelemetryDateTime) AS TripEnd
FROM FleetTelemetryStream fts
INNER JOIN DimVehicle v ON fts.VehicleID = v.VehicleID
INNER JOIN DimTime t ON CAST(fts.TelemetryDateTime AS DATETIME2) = t.FullDateTime
LEFT JOIN DimRoute r ON ... -- Route matching logic
WHERE fts.TelemetryDateTime >= DATEADD(HOUR, -1, GETDATE())
GROUP BY t.TimeKey, v.VehicleKey, r.RouteKey;
```

## Troubleshooting

### Performance Optimization

**Issue: Slow cross-fact queries**

```sql
-- Add covering indexes for common join patterns
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Composite
ON FactWarehouseOperations (TimeKey, WarehouseKey, ProductKey)
INCLUDE (DwellTimeMinutes, PickRate, OperationType);

CREATE NONCLUSTERED INDEX IX_FactFleet_Composite
ON FactFleetTrips (TimeKey, VehicleKey, RouteKey)
INCLUDE (IdleTimeMinutes, FuelConsumedLiters, TotalDistanceKM);

-- Enable columnstore index for analytical queries
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactWarehouse_CS
ON FactWarehouseOperations (TimeKey, ProductKey, DwellTimeMinutes, PickRate);
```

**Issue: Power BI refresh timeouts**

```sql
-- Partition large fact tables by year/month
ALTER DATABASE LogiFleetPulse SET COMPATIBILITY_LEVEL = 150;
GO

CREATE PARTITION FUNCTION pf_YearMonth (INT)
AS RANGE RIGHT FOR VALUES (202001, 202002, 202003, ...);

CREATE PARTITION SCHEME ps_YearMonth
AS PARTITION pf_YearMonth ALL TO ([PRIMARY]);

-- Recreate fact table on partition scheme
CREATE TABLE FactWarehouseOperations_Partitioned (
    ...
) ON ps_YearMonth(TimeKey);
```

### Data Quality Issues

**Issue: Missing gravity scores for new products**

```sql
-- Auto-calculate gravity score for products missing values
UPDATE DimProductGravity
SET 
    VelocityScore = ISNULL(VelocityScore, (
        SELECT AVG(CAST(COUNT(*) AS DECIMAL(10,2)))
        FROM FactWarehouseOperations w
        WHERE w.ProductKey = DimProductGravity.ProductKey
            AND w.OperationType IN ('Picking', 'Packing')
        GROUP BY CAST(w.CreatedDate AS DATE)
    )),
    ValueScore = ISNULL(ValueScore, 5.0), -- Default mid-range
    FragilityScore = ISNULL(FragilityScore, 5) -- Default mid-range
WHERE GravityScore IS NULL;
```

**Issue: Duplicate fleet trips from telemetry**

```sql
-- Deduplicate using ROW_NUMBER window function
WITH DuplicateTrips AS (
    SELECT 
        TripID,
        ROW_NUMBER() OVER (
            PARTITION BY VehicleKey, TripStartDateTime, TripEndDateTime 
            ORDER BY TripID
        ) AS RowNum
    FROM FactFleetTrips
)
DELETE FROM DuplicateTrips WHERE RowNum > 1;

-- Add unique constraint to prevent future duplicates
ALTER TABLE FactFleetTrips
ADD CONSTRAINT UQ_VehicleTrip UNIQUE (VehicleKey, TripStartDateTime);
```

### Power BI Connectivity Issues

**Issue: Row-level security not applying**

1. Verify RLS roles are published to Power BI Service
2. Check user email matches USERNAME() or USERPRINCIPALNAME()
3. Test RLS in Power BI Desktop using "View as Role"

```dax
-- Debug RLS filter context
RLS Debug = 
VAR CurrentUser = USERNAME()
VAR AllowedRegions = 
    CALCULATETABLE(
        VALUES(DimGeography[Region]),
        FILTER(
            DimGeography,
            DimGeography[Region] = CurrentUser
        )
    )
RETURN
    CONCATENATEX(Allow
