---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server and Power BI data warehousing engine for logistics intelligence with multi-fact star schema for fleet and warehouse analytics
triggers:
  - set up logifleet pulse supply chain analytics
  - configure logistics data warehouse with power bi
  - implement multi-fact star schema for fleet operations
  - create warehouse gravity zone analytics
  - build real-time logistics intelligence dashboard
  - deploy logicore supply chain pulse database
  - integrate fleet telemetry with warehouse operations
  - design cross-modal logistics kpi dashboard
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a multi-modal logistics intelligence engine that combines warehouse operations, fleet telemetry, inventory management, and external data sources into a unified semantic layer. Built on MS SQL Server with Power BI visualization, it provides:

- **Multi-fact star schema** connecting warehouse operations, fleet trips, and cross-dock activities
- **Time-phased dimensions** with 15-minute granularity for real-time analysis
- **Warehouse Gravity Zones™** — spatial optimization based on pick frequency, value, and fragility
- **Cross-fact KPI harmonization** linking inventory metrics with fleet performance
- **Predictive bottleneck detection** using temporal elasticity modeling
- **Role-based dashboards** with row-level security

The system answers complex cross-domain questions like: "Show shipments delayed by weather that originated from cold-storage zones with >72hr dwell time."

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- Network access to WMS/TMS data sources or staging area
- SQL Server Management Studio (SSMS) recommended

### Database Setup

1. **Clone the repository:**

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**

```sql
-- Connect to your SQL Server instance via SSMS
-- Create the database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Execute the schema deployment script
-- (Assumed to be in sql/schema_deployment.sql)
:r sql/schema_deployment.sql
GO

-- Create service account for Power BI
CREATE LOGIN pbi_service WITH PASSWORD = '$(PBI_SERVICE_PASSWORD)'
CREATE USER pbi_service FOR LOGIN pbi_service
EXEC sp_addrolemember 'db_datareader', 'pbi_service'
GO
```

3. **Configure data source connections:**

```json
// config.json (do not commit with real credentials)
{
  "connections": {
    "wms_api": {
      "type": "rest",
      "endpoint": "${WMS_API_ENDPOINT}",
      "auth_token": "${WMS_API_TOKEN}"
    },
    "fleet_telemetry": {
      "type": "sql",
      "server": "${FLEET_DB_SERVER}",
      "database": "${FLEET_DB_NAME}",
      "user": "${FLEET_DB_USER}",
      "password": "${FLEET_DB_PASSWORD}"
    },
    "erp_system": {
      "type": "odbc",
      "dsn": "${ERP_DSN}",
      "credentials": "${ERP_CREDENTIALS}"
    }
  },
  "refresh_interval_minutes": 15,
  "alert_email_smtp": "${SMTP_SERVER}",
  "alert_recipients": "${ALERT_EMAIL_LIST}"
}
```

### Power BI Template Setup

1. **Open the template:**

```bash
# Open the Power BI template file
start LogiFleet_Pulse_Master.pbit
```

2. **Configure connection in Power BI:**

- When prompted, enter your SQL Server instance name
- Use the `pbi_service` credentials or Windows authentication
- Click "Load" to import the data model

3. **Publish to Power BI Service (optional):**

- File → Publish → Select workspace
- Configure scheduled refresh with gateway if needed

## Key Database Objects

### Core Fact Tables

#### FactWarehouseOperations

```sql
-- Main warehouse activity fact table
CREATE TABLE dbo.FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ZoneKey INT NOT NULL,
    OperationType VARCHAR(20), -- 'Putaway', 'Pick', 'Pack', 'Ship'
    DwellTimeMinutes DECIMAL(10,2),
    PickRate DECIMAL(8,2), -- Items per hour
    PackingTimeSeconds INT,
    OperatorKey INT,
    BatchID VARCHAR(50),
    CreatedTimestamp DATETIME2 DEFAULT GETUTCDATE(),
    CONSTRAINT FK_Warehouse_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Warehouse_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
)

-- Create columnstore index for analytics
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_WarehouseOps 
ON dbo.FactWarehouseOperations (TimeKey, ProductKey, WarehouseKey, DwellTimeMinutes, PickRate)

-- Partitioning by month for performance
CREATE PARTITION FUNCTION PF_MonthlyWarehouse (INT)
AS RANGE RIGHT FOR VALUES (20260101, 20260201, 20260301, ...)
```

#### FactFleetTrips

```sql
-- Fleet and route performance fact table
CREATE TABLE dbo.FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TripStartTimeKey INT NOT NULL,
    TripEndTimeKey INT,
    VehicleKey INT NOT NULL,
    DriverKey INT NOT NULL,
    RouteKey INT NOT NULL,
    OriginGeographyKey INT,
    DestinationGeographyKey INT,
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(8,2),
    IdleTimeMinutes DECIMAL(8,2),
    LoadWeightKG DECIMAL(10,2),
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    DelayMinutes INT,
    DelayReason VARCHAR(100),
    AvgSpeedKPH DECIMAL(6,2),
    TripStatus VARCHAR(20), -- 'Completed', 'InProgress', 'Delayed', 'Cancelled'
    CONSTRAINT FK_Fleet_StartTime FOREIGN KEY (TripStartTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey)
)

-- Index for real-time monitoring
CREATE NONCLUSTERED INDEX IX_FleetTrips_Status 
ON dbo.FactFleetTrips (TripStatus, TripStartTimeKey) 
INCLUDE (VehicleKey, DelayMinutes, IdleTimeMinutes)
```

#### FactCrossDock

```sql
-- Cross-dock operations (direct transfers)
CREATE TABLE dbo.FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    InboundTimeKey INT NOT NULL,
    OutboundTimeKey INT,
    ProductKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    QuantityUnits INT,
    DwellMinutes DECIMAL(8,2),
    CrossDockZoneKey INT,
    CONSTRAINT FK_CrossDock_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_CrossDock_Inbound FOREIGN KEY (InboundTripKey) REFERENCES FactFleetTrips(TripKey)
)
```

### Key Dimension Tables

#### DimTime (Time-phased dimension)

```sql
-- 15-minute granularity time dimension
CREATE TABLE dbo.DimTime (
    TimeKey INT PRIMARY KEY,
    FullDatetime DATETIME2 NOT NULL,
    Date DATE NOT NULL,
    Year INT,
    Quarter INT,
    Month INT,
    Week INT,
    DayOfYear INT,
    DayOfMonth INT,
    DayOfWeek INT,
    DayName VARCHAR(10),
    Hour INT,
    Minute INT,
    TimeSlot15Min VARCHAR(5), -- '00:00', '00:15', '00:30', '00:45'
    IsWeekend BIT,
    IsHoliday BIT,
    FiscalYear INT,
    FiscalQuarter INT,
    FiscalPeriod INT
)

-- Stored procedure to populate time dimension
CREATE PROCEDURE dbo.PopulateTimeDimension
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    DECLARE @CurrentDateTime DATETIME2 = @StartDate
    DECLARE @TimeKey INT
    
    WHILE @CurrentDateTime <= @EndDate
    BEGIN
        -- Round to 15-minute intervals
        SET @CurrentDateTime = DATEADD(MINUTE, 
            (DATEDIFF(MINUTE, 0, @CurrentDateTime) / 15) * 15, 0)
        
        SET @TimeKey = CAST(FORMAT(@CurrentDateTime, 'yyyyMMddHHmm') AS INT)
        
        INSERT INTO dbo.DimTime (TimeKey, FullDatetime, Date, Year, Quarter, Month, 
            Week, DayOfYear, DayOfMonth, DayOfWeek, DayName, Hour, Minute, 
            TimeSlot15Min, IsWeekend, FiscalYear, FiscalQuarter)
        VALUES (
            @TimeKey,
            @CurrentDateTime,
            CAST(@CurrentDateTime AS DATE),
            YEAR(@CurrentDateTime),
            DATEPART(QUARTER, @CurrentDateTime),
            MONTH(@CurrentDateTime),
            DATEPART(WEEK, @CurrentDateTime),
            DATEPART(DAYOFYEAR, @CurrentDateTime),
            DAY(@CurrentDateTime),
            DATEPART(WEEKDAY, @CurrentDateTime),
            DATENAME(WEEKDAY, @CurrentDateTime),
            DATEPART(HOUR, @CurrentDateTime),
            DATEPART(MINUTE, @CurrentDateTime),
            FORMAT(@CurrentDateTime, 'HH:mm'),
            CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1,7) THEN 1 ELSE 0 END,
            -- Fiscal year logic (example: starts April 1)
            CASE WHEN MONTH(@CurrentDateTime) >= 4 
                THEN YEAR(@CurrentDateTime) 
                ELSE YEAR(@CurrentDateTime) - 1 END,
            CASE WHEN MONTH(@CurrentDateTime) >= 4 
                THEN ((MONTH(@CurrentDateTime) - 4) / 3) + 1
                ELSE ((MONTH(@CurrentDateTime) + 8) / 3) + 1 END
        )
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime)
    END
END
GO

-- Execute to populate 2 years of data
EXEC dbo.PopulateTimeDimension '2026-01-01', '2027-12-31'
```

#### DimProductGravity (Warehouse Gravity Zones)

```sql
-- Product dimension with gravity score
CREATE TABLE dbo.DimProduct (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    ProductID VARCHAR(50) UNIQUE NOT NULL,
    ProductName VARCHAR(200),
    CategoryL1 VARCHAR(100),
    CategoryL2 VARCHAR(100),
    CategoryL3 VARCHAR(100),
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    FragilityScore DECIMAL(3,2), -- 0.00 to 1.00
    ValuePerUnit DECIMAL(12,2),
    TargetInventoryDays INT,
    SupplierKey INT,
    IsActive BIT DEFAULT 1,
    EffectiveDate DATE,
    ExpirationDate DATE
)

CREATE TABLE dbo.DimProductGravity (
    ProductGravityKey INT IDENTITY(1,1) PRIMARY KEY,
    ProductKey INT NOT NULL,
    SnapshotDate DATE NOT NULL,
    PickFrequencyScore DECIMAL(5,2), -- Calculated from historical data
    ValueScore DECIMAL(5,2),
    FragilityScore DECIMAL(5,2),
    ReplenishmentLeadTimeDays INT,
    GravityIndex AS (PickFrequencyScore * 0.4 + ValueScore * 0.3 + 
        FragilityScore * 0.2 + (100.0 / NULLIF(ReplenishmentLeadTimeDays, 0)) * 0.1),
    RecommendedZoneType VARCHAR(20), -- 'HighGravity', 'MediumGravity', 'LowGravity'
    CONSTRAINT FK_ProductGravity_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
)

-- Stored procedure to calculate gravity scores
CREATE PROCEDURE dbo.CalculateProductGravity
    @AnalysisPeriodDays INT = 90
AS
BEGIN
    INSERT INTO dbo.DimProductGravity (ProductKey, SnapshotDate, PickFrequencyScore, 
        ValueScore, FragilityScore, ReplenishmentLeadTimeDays, RecommendedZoneType)
    SELECT 
        p.ProductKey,
        CAST(GETDATE() AS DATE) AS SnapshotDate,
        -- Pick frequency normalized to 0-100 scale
        CASE 
            WHEN picks.TotalPicks IS NULL THEN 0
            ELSE (picks.TotalPicks * 100.0 / NULLIF(picks.MaxPicks, 0))
        END AS PickFrequencyScore,
        -- Value score normalized
        (p.ValuePerUnit * 100.0 / NULLIF(val.MaxValue, 0)) AS ValueScore,
        -- Fragility already 0-100
        p.FragilityScore * 100 AS FragilityScore,
        ISNULL(s.AvgLeadTimeDays, 30) AS ReplenishmentLeadTimeDays,
        -- Zone assignment logic
        CASE 
            WHEN (picks.TotalPicks * 100.0 / NULLIF(picks.MaxPicks, 0)) > 70 THEN 'HighGravity'
            WHEN (picks.TotalPicks * 100.0 / NULLIF(picks.MaxPicks, 0)) > 30 THEN 'MediumGravity'
            ELSE 'LowGravity'
        END AS RecommendedZoneType
    FROM dbo.DimProduct p
    OUTER APPLY (
        SELECT 
            COUNT(*) AS TotalPicks,
            MAX(COUNT(*)) OVER () AS MaxPicks
        FROM dbo.FactWarehouseOperations wo
        WHERE wo.ProductKey = p.ProductKey
            AND wo.OperationType = 'Pick'
            AND wo.CreatedTimestamp >= DATEADD(DAY, -@AnalysisPeriodDays, GETDATE())
    ) picks
    CROSS APPLY (
        SELECT MAX(ValuePerUnit) AS MaxValue FROM dbo.DimProduct
    ) val
    LEFT JOIN dbo.DimSupplier s ON p.SupplierKey = s.SupplierKey
    WHERE p.IsActive = 1
END
GO
```

#### DimGeography (Hierarchical location)

```sql
-- Geographic hierarchy
CREATE TABLE dbo.DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationID VARCHAR(50) UNIQUE NOT NULL,
    LocationName VARCHAR(200),
    LocationType VARCHAR(50), -- 'Warehouse', 'CrossDock', 'CustomerSite', 'RouteNode'
    Continent VARCHAR(50),
    Country VARCHAR(100),
    Region VARCHAR(100),
    City VARCHAR(100),
    PostalCode VARCHAR(20),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    TimeZone VARCHAR(50),
    ParentGeographyKey INT,
    CONSTRAINT FK_Geography_Parent FOREIGN KEY (ParentGeographyKey) REFERENCES DimGeography(GeographyKey)
)

-- Spatial index for distance calculations
CREATE SPATIAL INDEX IX_Geography_Spatial 
ON dbo.DimGeography (Geography)
```

## Data Loading Patterns

### Incremental ETL for Warehouse Operations

```sql
-- Stored procedure for incremental load from WMS staging
CREATE PROCEDURE dbo.LoadWarehouseOperations
    @LastLoadTimestamp DATETIME2
AS
BEGIN
    SET NOCOUNT ON
    
    -- Insert new records from staging
    INSERT INTO dbo.FactWarehouseOperations (
        TimeKey, ProductKey, WarehouseKey, ZoneKey, OperationType,
        DwellTimeMinutes, PickRate, PackingTimeSeconds, OperatorKey, BatchID
    )
    SELECT 
        -- Convert timestamp to TimeKey (15-min grain)
        CAST(FORMAT(
            DATEADD(MINUTE, (DATEDIFF(MINUTE, 0, s.OperationTimestamp) / 15) * 15, 0),
            'yyyyMMddHHmm'
        ) AS INT) AS TimeKey,
        p.ProductKey,
        w.WarehouseKey,
        z.ZoneKey,
        s.OperationType,
        DATEDIFF(MINUTE, s.StartTime, s.EndTime) AS DwellTimeMinutes,
        s.UnitsProcessed * 60.0 / NULLIF(DATEDIFF(SECOND, s.StartTime, s.EndTime), 0) AS PickRate,
        CASE WHEN s.OperationType = 'Pack' 
            THEN DATEDIFF(SECOND, s.StartTime, s.EndTime) 
            ELSE NULL END AS PackingTimeSeconds,
        o.OperatorKey,
        s.BatchID
    FROM staging.WarehouseOperations s
    INNER JOIN dbo.DimProduct p ON s.ProductID = p.ProductID AND p.IsActive = 1
    INNER JOIN dbo.DimWarehouse w ON s.WarehouseID = w.WarehouseID
    INNER JOIN dbo.DimZone z ON s.ZoneID = z.ZoneID
    LEFT JOIN dbo.DimOperator o ON s.OperatorID = o.OperatorID
    WHERE s.LoadedTimestamp > @LastLoadTimestamp
        AND NOT EXISTS (
            SELECT 1 FROM dbo.FactWarehouseOperations f
            WHERE f.BatchID = s.BatchID 
                AND f.OperationType = s.OperationType
        )
    
    -- Update control table
    UPDATE dbo.ETLControl
    SET LastLoadTimestamp = GETUTCDATE()
    WHERE TableName = 'FactWarehouseOperations'
END
GO
```

### Fleet Telemetry Integration

```sql
-- Load fleet trip data from telemetry system
CREATE PROCEDURE dbo.LoadFleetTrips
AS
BEGIN
    -- Using external table for real-time telemetry stream
    INSERT INTO dbo.FactFleetTrips (
        TripStartTimeKey, TripEndTimeKey, VehicleKey, DriverKey, RouteKey,
        OriginGeographyKey, DestinationGeographyKey, DistanceKM, FuelConsumedLiters,
        IdleTimeMinutes, LoadWeightKG, LoadingTimeMinutes, UnloadingTimeMinutes,
        DelayMinutes, DelayReason, AvgSpeedKPH, TripStatus
    )
    SELECT 
        CAST(FORMAT(
            DATEADD(MINUTE, (DATEDIFF(MINUTE, 0, t.TripStartTime) / 15) * 15, 0),
            'yyyyMMddHHmm'
        ) AS INT) AS TripStartTimeKey,
        CASE WHEN t.TripEndTime IS NOT NULL THEN
            CAST(FORMAT(
                DATEADD(MINUTE, (DATEDIFF(MINUTE, 0, t.TripEndTime) / 15) * 15, 0),
                'yyyyMMddHHmm'
            ) AS INT)
        ELSE NULL END AS TripEndTimeKey,
        v.VehicleKey,
        d.DriverKey,
        r.RouteKey,
        go.GeographyKey AS OriginKey,
        gd.GeographyKey AS DestKey,
        t.TotalDistanceKM,
        t.FuelUsedLiters,
        t.IdleMinutes,
        t.CargoWeightKG,
        t.LoadDurationMinutes,
        t.UnloadDurationMinutes,
        CASE WHEN t.ScheduledArrival IS NOT NULL AND t.ActualArrival IS NOT NULL
            THEN DATEDIFF(MINUTE, t.ScheduledArrival, t.ActualArrival)
            ELSE NULL END AS DelayMinutes,
        t.DelayCategory,
        t.AverageSpeedKPH,
        CASE 
            WHEN t.TripEndTime IS NULL THEN 'InProgress'
            WHEN DATEDIFF(MINUTE, t.ScheduledArrival, t.ActualArrival) > 30 THEN 'Delayed'
            ELSE 'Completed'
        END AS TripStatus
    FROM external_telemetry.TripData t
    INNER JOIN dbo.DimVehicle v ON t.VehicleID = v.VehicleID
    INNER JOIN dbo.DimDriver d ON t.DriverID = d.DriverID
    INNER JOIN dbo.DimRoute r ON t.RouteID = r.RouteID
    LEFT JOIN dbo.DimGeography go ON t.OriginLocationID = go.LocationID
    LEFT JOIN dbo.DimGeography gd ON t.DestinationLocationID = gd.LocationID
    WHERE t.ProcessedFlag = 0
END
GO
```

## Cross-Fact Analytics Queries

### Fleet Utilization vs. Warehouse Dwell Time

```sql
-- Analyze correlation between warehouse dwell and fleet delays
SELECT 
    t.Date,
    t.DayName,
    w.WarehouseName,
    -- Warehouse metrics
    AVG(wo.DwellTimeMinutes) AS AvgDwellMinutes,
    SUM(CASE WHEN wo.DwellTimeMinutes > 72 THEN 1 ELSE 0 END) AS ItemsOver72hrs,
    -- Fleet metrics for trips originating from this warehouse
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdleMinutes,
    AVG(ft.DelayMinutes) AS AvgDelayMinutes,
    SUM(ft.FuelConsumedLiters) AS TotalFuelLiters,
    -- Cross-fact KPI
    (SUM(ft.IdleTimeMinutes) * 100.0 / NULLIF(SUM(DATEDIFF(MINUTE, ts.FullDatetime, te.FullDatetime)), 0)) AS IdlePercentage
FROM dbo.DimTime t
INNER JOIN dbo.FactWarehouseOperations wo ON t.TimeKey = wo.TimeKey
INNER JOIN dbo.DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
LEFT JOIN dbo.FactFleetTrips ft ON ft.OriginGeographyKey = w.GeographyKey
    AND ft.TripStartTimeKey / 10000 = t.TimeKey / 10000 -- Same day
LEFT JOIN dbo.DimTime ts ON ft.TripStartTimeKey = ts.TimeKey
LEFT JOIN dbo.DimTime te ON ft.TripEndTimeKey = te.TimeKey
WHERE t.Date >= DATEADD(MONTH, -3, GETDATE())
    AND t.Date <= GETDATE()
GROUP BY t.Date, t.DayName, w.WarehouseName
HAVING AVG(wo.DwellTimeMinutes) > 48
ORDER BY t.Date DESC, AvgDwellMinutes DESC
```

### Gravity Zone Performance Analysis

```sql
-- Compare pick rates across gravity zones
WITH GravityPerformance AS (
    SELECT 
        pg.RecommendedZoneType,
        p.CategoryL1,
        COUNT(DISTINCT wo.OperationKey) AS TotalPicks,
        AVG(wo.PickRate) AS AvgPickRate,
        AVG(wo.DwellTimeMinutes) AS AvgDwellMinutes,
        SUM(p.ValuePerUnit * wo.PickRate) AS TotalValueThroughput
    FROM dbo.FactWarehouseOperations wo
    INNER JOIN dbo.DimProduct p ON wo.ProductKey = p.ProductKey
    INNER JOIN dbo.DimProductGravity pg ON p.ProductKey = pg.ProductKey
        AND pg.SnapshotDate = (SELECT MAX(SnapshotDate) FROM dbo.DimProductGravity)
    WHERE wo.OperationType = 'Pick'
        AND wo.TimeKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm') AS INT)
    GROUP BY pg.RecommendedZoneType, p.CategoryL1
)
SELECT 
    RecommendedZoneType,
    CategoryL1,
    TotalPicks,
    AvgPickRate,
    AvgDwellMinutes,
    TotalValueThroughput,
    -- Performance index
    (AvgPickRate * TotalValueThroughput / NULLIF(AvgDwellMinutes, 0)) AS PerformanceIndex,
    -- Rank within zone type
    RANK() OVER (PARTITION BY RecommendedZoneType ORDER BY AvgPickRate DESC) AS RankInZone
FROM GravityPerformance
ORDER BY RecommendedZoneType, PerformanceIndex DESC
```

### Predictive Bottleneck Detection

```sql
-- Identify potential bottlenecks using time-series patterns
CREATE PROCEDURE dbo.DetectBottlenecks
    @ForecastHours INT = 24
AS
BEGIN
    -- Calculate moving average and standard deviation
    WITH HistoricalPatterns AS (
        SELECT 
            t.Hour,
            t.DayOfWeek,
            wo.WarehouseKey,
            wo.ZoneKey,
            AVG(wo.DwellTimeMinutes) AS AvgDwell,
            STDEV(wo.DwellTimeMinutes) AS StdDevDwell,
            COUNT(*) AS OperationCount
        FROM dbo.FactWarehouseOperations wo
        INNER JOIN dbo.DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.FullDatetime >= DATEADD(DAY, -90, GETDATE())
        GROUP BY t.Hour, t.DayOfWeek, wo.WarehouseKey, wo.ZoneKey
    ),
    CurrentLoad AS (
        SELECT 
            t.Hour,
            t.DayOfWeek,
            wo.WarehouseKey,
            wo.ZoneKey,
            AVG(wo.DwellTimeMinutes) AS CurrentAvgDwell,
            COUNT(*) AS CurrentOps
        FROM dbo.FactWarehouseOperations wo
        INNER JOIN dbo.DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.FullDatetime >= DATEADD(HOUR, -4, GETDATE())
        GROUP BY t.Hour, t.DayOfWeek, wo.WarehouseKey, wo.ZoneKey
    )
    SELECT 
        w.WarehouseName,
        z.ZoneName,
        c.Hour,
        c.CurrentAvgDwell,
        h.AvgDwell AS HistoricalAvg,
        h.StdDevDwell,
        -- Anomaly score (how many std devs from mean)
        (c.CurrentAvgDwell - h.AvgDwell) / NULLIF(h.StdDevDwell, 0) AS AnomalyScore,
        -- Bottleneck probability
        CASE 
            WHEN (c.CurrentAvgDwell - h.AvgDwell) / NULLIF(h.StdDevDwell, 0) > 2 THEN 'High'
            WHEN (c.CurrentAvgDwell - h.AvgDwell) / NULLIF(h.StdDevDwell, 0) > 1 THEN 'Medium'
            ELSE 'Low'
        END AS BottleneckRisk,
        -- Recommended action
        CASE 
            WHEN (c.CurrentAvgDwell - h.AvgDwell) / NULLIF(h.StdDevDwell, 0) > 2 
                THEN 'Immediate staff reassignment needed'
            WHEN (c.CurrentAvgDwell - h.AvgDwell) / NULLIF(h.StdDevDwell, 0) > 1 
                THEN 'Monitor closely - may need intervention'
            ELSE 'Normal operations'
        END AS RecommendedAction
    FROM CurrentLoad c
    INNER JOIN HistoricalPatterns h ON c.Hour = h.Hour 
        AND c.DayOfWeek = h.DayOfWeek
        AND c.WarehouseKey = h.WarehouseKey
        AND c.ZoneKey = h.ZoneKey
    INNER JOIN dbo.DimWarehouse w ON c.WarehouseKey = w.WarehouseKey
    INNER JOIN dbo.DimZone z ON c.ZoneKey = z.ZoneKey
    WHERE (c.CurrentAvgDwell - h.AvgDwell) / NULLIF(h.StdDevDwell, 0) > 1
    ORDER BY AnomalyScore DESC
END
GO
```

## Automated Alerting System

```sql
-- Alert configuration table
CREATE TABLE dbo.AlertRules (
    AlertRuleID INT IDENTITY(1,1) PRIMARY KEY,
    RuleName VARCHAR(100) NOT NULL,
    MetricName VARCHAR(100),
    ThresholdValue DECIMAL(18,2),
    ComparisonOperator VARCHAR(10), -- '>', '<', '>=', '<=', '='
    AlertSeverity VARCHAR(20), -- 'Critical', 'Warning', 'Info'
    EmailRecipients VARCHAR(500),
    SMSRecipients VARCHAR(500),
    IsActive BIT DEFAULT 1
)

-- Stored procedure to evaluate and send alerts
CREATE PROCEDURE dbo.EvaluateAlerts
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX)
    
    -- Fleet idle time threshold
    IF EXISTS (
        SELECT 1 FROM dbo.FactFleetTrips ft
        INNER JOIN dbo.DimTime t ON ft.TripStartTimeKey = t.TimeKey
        WHERE t.Date = CAST(GETDATE() AS DATE)
        GROUP BY ft.VehicleKey
        HAVING (SUM(ft.IdleTimeMinutes) * 100.0 / SUM(DATEDIFF(MINUTE, 
            t.FullDatetime, 
            DATEADD(MINUTE, DATEDIFF(MINUTE
