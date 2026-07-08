---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing platform for logistics intelligence with multi-fact star schema and real-time fleet/warehouse analytics
triggers:
  - set up LogiFleet Pulse supply chain analytics
  - configure Power BI logistics dashboard
  - create multi-fact star schema for warehouse data
  - implement fleet telemetry tracking in SQL Server
  - build cross-modal supply chain KPI dashboard
  - deploy LogiFleet warehouse gravity zones
  - integrate real-time logistics data warehouse
  - design predictive bottleneck detection system
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is an advanced data warehousing and analytics platform that unifies warehouse operations, fleet telemetry, and supply chain metrics into a single semantic layer. Built on MS SQL Server with Power BI visualization, it implements a multi-fact star schema with time-phased dimensions for cross-modal logistics intelligence.

**Core capabilities:**
- Multi-fact data warehouse (warehouse operations, fleet trips, cross-dock)
- Real-time KPI harmonization across logistics domains
- Predictive bottleneck detection and fleet triage
- Warehouse gravity zone optimization
- Temporal elasticity modeling for scenario analysis

## Installation & Setup

### Prerequisites

```bash
# Required components
- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Power BI Desktop (latest version)
- .NET Framework 4.7.2+ (for stored procedures)
```

### Database Schema Deployment

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**
```sql
-- Connect to your SQL Server instance via SSMS
-- Execute the schema creation script
:r schema/01_create_database.sql
:r schema/02_create_dimensions.sql
:r schema/03_create_facts.sql
:r schema/04_create_views.sql
:r schema/05_create_procedures.sql
:r schema/06_create_indexes.sql
```

3. **Configure data source connections:**
```json
// config/data_sources.json
{
  "wms_connection": "Server=${WMS_SERVER};Database=${WMS_DB};Trusted_Connection=True;",
  "telemetry_api": "${TELEMETRY_API_ENDPOINT}",
  "erp_connection": "Server=${ERP_SERVER};Database=${ERP_DB};User Id=${ERP_USER};Password=${ERP_PASSWORD};",
  "refresh_interval_minutes": 15,
  "alert_recipients": ["${ALERT_EMAIL}"]
}
```

## Core Data Model

### Dimension Tables

```sql
-- DimTime: 15-minute granularity time dimension
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    Date DATE NOT NULL,
    TimeOfDay TIME NOT NULL,
    Hour TINYINT NOT NULL,
    QuarterHour TINYINT NOT NULL, -- 0, 15, 30, 45
    DayOfWeek TINYINT NOT NULL,
    DayName VARCHAR(20),
    IsWeekend BIT NOT NULL,
    FiscalPeriod VARCHAR(10),
    INDEX IX_DimTime_Date (Date),
    INDEX IX_DimTime_Hour (Hour)
);

-- DimProductGravity: Product classification with gravity scoring
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName NVARCHAR(200),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    GravityScore DECIMAL(5,2), -- Composite score: velocity + value + fragility
    PickFrequency DECIMAL(8,2), -- Picks per day
    UnitValue DECIMAL(12,2),
    FragilityIndex DECIMAL(3,2), -- 0.0 to 1.0
    OptimalZone VARCHAR(20), -- High/Medium/Low gravity zone
    LastRecalculated DATETIME2,
    INDEX IX_ProductGravity_Score (GravityScore DESC)
);

-- DimGeography: Hierarchical location dimension
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationCode VARCHAR(50) NOT NULL UNIQUE,
    LocationName NVARCHAR(200),
    LocationType VARCHAR(50), -- Warehouse, Route Node, Customer Site
    Address NVARCHAR(500),
    City VARCHAR(100),
    Region VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(10,8),
    Longitude DECIMAL(11,8),
    ParentLocationKey INT,
    FOREIGN KEY (ParentLocationKey) REFERENCES DimGeography(GeographyKey)
);

-- DimSupplierReliability: Supplier performance metrics
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierCode VARCHAR(50) NOT NULL UNIQUE,
    SupplierName NVARCHAR(200),
    LeadTimeMean DECIMAL(6,2), -- Days
    LeadTimeStdDev DECIMAL(6,2),
    DefectRate DECIMAL(5,4), -- Percentage
    ComplianceScore DECIMAL(5,2), -- 0-100
    ReliabilityGrade CHAR(1), -- A, B, C, D, F
    LastUpdated DATETIME2
);
```

### Fact Tables

```sql
-- FactWarehouseOperations: Core warehouse activity tracking
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    OperationType VARCHAR(50), -- Receiving, Putaway, Picking, Packing, Shipping
    Quantity INT NOT NULL,
    DwellTimeMinutes INT,
    ZoneLocation VARCHAR(50),
    OperatorID VARCHAR(50),
    DurationMinutes DECIMAL(8,2),
    CostEstimate DECIMAL(12,2),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    FOREIGN KEY (WarehouseKey) REFERENCES DimGeography(GeographyKey)
);

CREATE COLUMNSTORE INDEX IX_FactWarehouse_CS 
ON FactWarehouseOperations (TimeKey, ProductKey, WarehouseKey, OperationType);

-- FactFleetTrips: Fleet telemetry and route tracking
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    StartTimeKey INT NOT NULL,
    EndTimeKey INT,
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50),
    OriginKey INT NOT NULL,
    DestinationKey INT NOT NULL,
    RouteCode VARCHAR(50),
    DistanceKm DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    DelayMinutes INT,
    DelayReason VARCHAR(200),
    WeatherCondition VARCHAR(50),
    LoadWeight DECIMAL(10,2),
    TripCost DECIMAL(12,2),
    FOREIGN KEY (StartTimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (EndTimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (OriginKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (DestinationKey) REFERENCES DimGeography(GeographyKey)
);

CREATE COLUMNSTORE INDEX IX_FactFleet_CS 
ON FactFleetTrips (StartTimeKey, VehicleID, OriginKey, DestinationKey);

-- FactCrossDock: Cross-docking operations
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    InboundTimeKey INT NOT NULL,
    OutboundTimeKey INT,
    ProductKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    Quantity INT NOT NULL,
    CrossDockDurationMinutes INT,
    CrossDockZone VARCHAR(50),
    FOREIGN KEY (InboundTimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (OutboundTimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    FOREIGN KEY (InboundTripKey) REFERENCES FactFleetTrips(TripKey),
    FOREIGN KEY (OutboundTripKey) REFERENCES FactFleetTrips(TripKey)
);
```

## Key Stored Procedures

### Calculate Product Gravity Scores

```sql
CREATE PROCEDURE sp_RecalculateProductGravity
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Update gravity scores based on pick frequency, value, and fragility
    UPDATE DimProductGravity
    SET 
        GravityScore = (
            (PickFrequency / NULLIF((SELECT MAX(PickFrequency) FROM DimProductGravity), 0) * 40) +
            (UnitValue / NULLIF((SELECT MAX(UnitValue) FROM DimProductGravity), 0) * 40) +
            (FragilityIndex * 20)
        ),
        OptimalZone = CASE
            WHEN (PickFrequency / NULLIF((SELECT MAX(PickFrequency) FROM DimProductGravity), 0) * 40) +
                 (UnitValue / NULLIF((SELECT MAX(UnitValue) FROM DimProductGravity), 0) * 40) +
                 (FragilityIndex * 20) >= 70 THEN 'High'
            WHEN (PickFrequency / NULLIF((SELECT MAX(PickFrequency) FROM DimProductGravity), 0) * 40) +
                 (UnitValue / NULLIF((SELECT MAX(UnitValue) FROM DimProductGravity), 0) * 40) +
                 (FragilityIndex * 20) >= 40 THEN 'Medium'
            ELSE 'Low'
        END,
        LastRecalculated = GETDATE();
        
    -- Update pick frequency from recent operations
    UPDATE p
    SET p.PickFrequency = picks.AvgDailyPicks
    FROM DimProductGravity p
    INNER JOIN (
        SELECT 
            ProductKey,
            AVG(DailyPicks) as AvgDailyPicks
        FROM (
            SELECT 
                ProductKey,
                CAST(w.TimeKey / 100 AS DATE) as Date,
                COUNT(*) as DailyPicks
            FROM FactWarehouseOperations w
            INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
            WHERE w.OperationType = 'Picking'
              AND t.Date >= DATEADD(day, -30, GETDATE())
            GROUP BY ProductKey, CAST(w.TimeKey / 100 AS DATE)
        ) daily
        GROUP BY ProductKey
    ) picks ON p.ProductKey = picks.ProductKey;
END;
```

### Incremental Data Load

```sql
CREATE PROCEDURE sp_IncrementalLoadWarehouseOps
    @LastLoadTime DATETIME2 = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Default to last 15 minutes if no time specified
    IF @LastLoadTime IS NULL
        SET @LastLoadTime = DATEADD(minute, -15, GETDATE());
    
    -- Insert new warehouse operations from staging
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, WarehouseKey, OperationType,
        Quantity, DwellTimeMinutes, ZoneLocation, OperatorID,
        DurationMinutes, CostEstimate
    )
    SELECT 
        t.TimeKey,
        p.ProductKey,
        g.GeographyKey,
        s.OperationType,
        s.Quantity,
        s.DwellTimeMinutes,
        s.ZoneLocation,
        s.OperatorID,
        s.DurationMinutes,
        s.DurationMinutes * 0.75 as CostEstimate -- $0.75/minute labor estimate
    FROM staging.WarehouseOperations s
    INNER JOIN DimTime t ON DATEADD(minute, 
        (t.QuarterHour - DATEPART(minute, s.OperationTimestamp) % 15), 
        CAST(CAST(s.OperationTimestamp AS DATE) AS DATETIME) + 
        CAST(CAST(DATEPART(hour, s.OperationTimestamp) AS VARCHAR) + ':' + 
        CAST((DATEPART(minute, s.OperationTimestamp) / 15) * 15 AS VARCHAR) AS TIME)
    ) = t.FullDateTime
    INNER JOIN DimProductGravity p ON s.SKU = p.SKU
    INNER JOIN DimGeography g ON s.WarehouseCode = g.LocationCode
    WHERE s.OperationTimestamp >= @LastLoadTime
      AND NOT EXISTS (
          SELECT 1 FROM FactWarehouseOperations f
          WHERE f.TimeKey = t.TimeKey
            AND f.ProductKey = p.ProductKey
            AND f.OperatorID = s.OperatorID
            AND f.ZoneLocation = s.ZoneLocation
      );
END;
```

### Predictive Bottleneck Detection

```sql
CREATE PROCEDURE sp_DetectBottlenecks
    @HoursAhead INT = 4,
    @ThresholdScore DECIMAL(5,2) = 0.75
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Identify predicted bottlenecks based on historical patterns
    WITH HistoricalLoad AS (
        SELECT 
            t.Hour,
            t.DayOfWeek,
            w.WarehouseKey,
            w.ZoneLocation,
            AVG(CAST(w.DurationMinutes AS FLOAT)) as AvgDuration,
            STDEV(w.DurationMinutes) as StdDevDuration,
            COUNT(*) as OperationCount
        FROM FactWarehouseOperations w
        INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
        WHERE t.Date >= DATEADD(day, -90, GETDATE())
        GROUP BY t.Hour, t.DayOfWeek, w.WarehouseKey, w.ZoneLocation
        HAVING COUNT(*) > 10
    ),
    CurrentLoad AS (
        SELECT 
            w.WarehouseKey,
            w.ZoneLocation,
            COUNT(*) as CurrentOps,
            AVG(CAST(w.DurationMinutes AS FLOAT)) as CurrentAvgDuration
        FROM FactWarehouseOperations w
        INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
        WHERE t.FullDateTime >= DATEADD(hour, -1, GETDATE())
        GROUP BY w.WarehouseKey, w.ZoneLocation
    )
    SELECT 
        g.LocationName as Warehouse,
        h.ZoneLocation,
        h.AvgDuration as HistoricalAvgDuration,
        c.CurrentAvgDuration,
        h.OperationCount as HistoricalVolume,
        c.CurrentOps as CurrentVolume,
        -- Bottleneck score: combination of duration increase and volume
        (
            (c.CurrentAvgDuration - h.AvgDuration) / NULLIF(h.StdDevDuration, 0) * 0.6 +
            (CAST(c.CurrentOps AS FLOAT) / NULLIF(h.OperationCount, 0)) * 0.4
        ) as BottleneckScore,
        DATEADD(hour, @HoursAhead, GETDATE()) as PredictedTime
    FROM HistoricalLoad h
    INNER JOIN CurrentLoad c ON h.WarehouseKey = c.WarehouseKey 
        AND h.ZoneLocation = c.ZoneLocation
    INNER JOIN DimGeography g ON h.WarehouseKey = g.GeographyKey
    WHERE DATEPART(hour, GETDATE()) = h.Hour
      AND DATEPART(dw, GETDATE()) = h.DayOfWeek
      AND (
        (c.CurrentAvgDuration - h.AvgDuration) / NULLIF(h.StdDevDuration, 0) * 0.6 +
        (CAST(c.CurrentOps AS FLOAT) / NULLIF(h.OperationCount, 0)) * 0.4
      ) >= @ThresholdScore
    ORDER BY BottleneckScore DESC;
END;
```

## Analytical Views

### Cross-Fact KPI View

```sql
CREATE VIEW vw_CrossFactKPIs AS
SELECT 
    t.Date,
    t.Hour,
    g.LocationName as Warehouse,
    p.Category as ProductCategory,
    
    -- Warehouse metrics
    COUNT(DISTINCT wo.OperationKey) as TotalOperations,
    AVG(wo.DwellTimeMinutes) as AvgDwellTime,
    SUM(CASE WHEN wo.OperationType = 'Picking' THEN wo.Quantity ELSE 0 END) as TotalPicks,
    
    -- Fleet metrics
    COUNT(DISTINCT ft.TripKey) as TotalTrips,
    AVG(ft.IdleTimeMinutes) as AvgIdleTime,
    SUM(ft.FuelConsumedLiters) as TotalFuel,
    
    -- Cross-modal efficiency
    AVG(wo.DwellTimeMinutes) * AVG(ft.IdleTimeMinutes) / 100.0 as WastageIndex,
    
    -- Cost estimates
    SUM(wo.CostEstimate) + SUM(ft.TripCost) as TotalOperatingCost
    
FROM DimTime t
LEFT JOIN FactWarehouseOperations wo ON t.TimeKey = wo.TimeKey
LEFT JOIN FactFleetTrips ft ON t.TimeKey = ft.StartTimeKey
LEFT JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
LEFT JOIN DimGeography g ON wo.WarehouseKey = g.GeographyKey
WHERE t.Date >= DATEADD(day, -30, GETDATE())
GROUP BY t.Date, t.Hour, g.LocationName, p.Category;
```

### Fleet Triage Priority View

```sql
CREATE VIEW vw_FleetTriagePriority AS
WITH RecentIssues AS (
    SELECT 
        VehicleID,
        COUNT(*) as IssueCount,
        SUM(DelayMinutes) as TotalDelayMinutes,
        AVG(IdleTimeMinutes) as AvgIdleTime
    FROM FactFleetTrips
    WHERE StartTimeKey >= (
        SELECT TimeKey FROM DimTime 
        WHERE FullDateTime >= DATEADD(day, -7, GETDATE())
        ORDER BY TimeKey ASC
        OFFSET 0 ROWS FETCH NEXT 1 ROWS ONLY
    )
    GROUP BY VehicleID
),
ProductValue AS (
    SELECT 
        ft.VehicleID,
        AVG(p.UnitValue * wo.Quantity) as AvgCargoValue
    FROM FactFleetTrips ft
    INNER JOIN FactWarehouseOperations wo ON EXISTS (
        SELECT 1 FROM DimTime t1, DimTime t2
        WHERE t1.TimeKey = ft.StartTimeKey
          AND t2.TimeKey = wo.TimeKey
          AND ABS(DATEDIFF(hour, t1.FullDateTime, t2.FullDateTime)) <= 2
    )
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    GROUP BY ft.VehicleID
)
SELECT 
    ri.VehicleID,
    ri.IssueCount,
    ri.TotalDelayMinutes,
    ri.AvgIdleTime,
    ISNULL(pv.AvgCargoValue, 0) as AvgCargoValue,
    -- Priority score: (issue frequency × delay impact × cargo value)
    (ri.IssueCount * 0.3 + 
     ri.TotalDelayMinutes / 60.0 * 0.4 + 
     ISNULL(pv.AvgCargoValue, 0) / 10000.0 * 0.3) as TriagePriority
FROM RecentIssues ri
LEFT JOIN ProductValue pv ON ri.VehicleID = pv.VehicleID
WHERE ri.IssueCount > 0 OR ri.TotalDelayMinutes > 30;
```

## Power BI Integration

### Connecting Power BI to SQL Server

1. **Open Power BI Desktop**
2. **Get Data → SQL Server**
```
Server: ${SQL_SERVER_HOSTNAME}
Database: LogiFleetPulse
Data Connectivity mode: DirectQuery (for real-time) or Import (for performance)
```

3. **Import key tables and views:**
```
- FactWarehouseOperations
- FactFleetTrips
- FactCrossDock
- DimTime
- DimProductGravity
- DimGeography
- DimSupplierReliability
- vw_CrossFactKPIs
- vw_FleetTriagePriority
```

### DAX Measures for Cross-Fact Analysis

```dax
// Warehouse Efficiency Score
WarehouseEfficiency = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR TargetDwell = 120 -- 2 hours target
VAR DwellScore = MAX(0, 1 - (AvgDwell - TargetDwell) / TargetDwell)

VAR AvgDuration = AVERAGE(FactWarehouseOperations[DurationMinutes])
VAR TargetDuration = 15 -- 15 minutes per operation
VAR DurationScore = MAX(0, 1 - (AvgDuration - TargetDuration) / TargetDuration)

RETURN (DwellScore * 0.6 + DurationScore * 0.4) * 100

// Fleet Utilization Rate
FleetUtilization = 
VAR TotalTripTime = SUMX(
    FactFleetTrips,
    DATEDIFF(
        RELATED(DimTime[FullDateTime]),
        LOOKUPVALUE(DimTime[FullDateTime], DimTime[TimeKey], FactFleetTrips[EndTimeKey]),
        MINUTE
    )
)
VAR TotalIdleTime = SUM(FactFleetTrips[IdleTimeMinutes])
RETURN 
IF(
    TotalTripTime > 0,
    (TotalTripTime - TotalIdleTime) / TotalTripTime,
    BLANK()
)

// Cross-Modal Delay Correlation
DelayCorrelation = 
VAR WarehouseDelays = 
    CALCULATETABLE(
        ADDCOLUMNS(
            FactWarehouseOperations,
            "IsDelayed", [DwellTimeMinutes] > 120
        ),
        ALL(FactWarehouseOperations)
    )
VAR FleetDelays = 
    CALCULATETABLE(
        ADDCOLUMNS(
            FactFleetTrips,
            "IsDelayed", [DelayMinutes] > 30
        ),
        ALL(FactFleetTrips)
    )
RETURN
-- Simplified correlation: percentage of overlapping delays
DIVIDE(
    COUNTROWS(FILTER(WarehouseDelays, [IsDelayed])) +
    COUNTROWS(FILTER(FleetDelays, [IsDelayed])),
    COUNTROWS(WarehouseDelays) + COUNTROWS(FleetDelays)
)

// Dynamic Gravity Zone Recommendation
RecommendedZone = 
VAR CurrentProduct = SELECTEDVALUE(DimProductGravity[ProductKey])
VAR GravityScore = SELECTEDVALUE(DimProductGravity[GravityScore])
RETURN
SWITCH(
    TRUE(),
    GravityScore >= 70, "Zone A (High Gravity - Near Dock)",
    GravityScore >= 40, "Zone B (Medium Gravity - Mid Floor)",
    "Zone C (Low Gravity - Back Storage)"
)
```

### Sample Power BI Refresh Query

```powerquery
let
    Source = Sql.Database("${SQL_SERVER}", "LogiFleetPulse"),
    
    // Warehouse Operations with incremental refresh
    WarehouseOps = Table.SelectRows(
        Source{[Schema="dbo",Item="FactWarehouseOperations"]}[Data],
        each [TimeKey] >= Number.From(RangeStart) and [TimeKey] < Number.From(RangeEnd)
    ),
    
    // Join with dimensions
    WithProduct = Table.NestedJoin(
        WarehouseOps, {"ProductKey"},
        Source{[Schema="dbo",Item="DimProductGravity"]}[Data], {"ProductKey"},
        "Product", JoinKind.Inner
    ),
    
    ExpandedProduct = Table.ExpandTableColumn(
        WithProduct, "Product",
        {"SKU", "ProductName", "Category", "GravityScore", "OptimalZone"},
        {"SKU", "ProductName", "Category", "GravityScore", "OptimalZone"}
    )
in
    ExpandedProduct
```

## Alert Configuration

### SQL Server Agent Job for Bottleneck Alerts

```sql
-- Create alert job
USE msdb;
GO

EXEC dbo.sp_add_job
    @job_name = N'LogiFleet_BottleneckMonitor';

EXEC sp_add_jobstep
    @job_name = N'LogiFleet_BottleneckMonitor',
    @step_name = N'Check and Alert',
    @subsystem = N'TSQL',
    @command = N'
        DECLARE @Bottlenecks TABLE (
            Warehouse NVARCHAR(200),
            Zone VARCHAR(50),
            Score DECIMAL(5,2)
        );
        
        INSERT INTO @Bottlenecks
        EXEC sp_DetectBottlenecks @HoursAhead = 2, @ThresholdScore = 0.8;
        
        IF EXISTS (SELECT 1 FROM @Bottlenecks)
        BEGIN
            DECLARE @EmailBody NVARCHAR(MAX);
            
            SET @EmailBody = N''<h2>Predicted Bottlenecks Alert</h2><table border="1">'';
            SET @EmailBody += N''<tr><th>Warehouse</th><th>Zone</th><th>Score</th></tr>'';
            
            SELECT @EmailBody += 
                N''<tr><td>'' + Warehouse + N''</td><td>'' + Zone + 
                N''</td><td>'' + CAST(Score AS NVARCHAR) + N''</td></tr>''
            FROM @Bottlenecks;
            
            SET @EmailBody += N''</table>'';
            
            EXEC msdb.dbo.sp_send_dbmail
                @recipients = ''${ALERT_EMAIL}'',
                @subject = ''LogiFleet Bottleneck Alert'',
                @body = @EmailBody,
                @body_format = ''HTML'';
        END
    ',
    @database_name = N'LogiFleetPulse';

EXEC sp_add_jobschedule
    @job_name = N'LogiFleet_BottleneckMonitor',
    @name = N'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;

EXEC sp_add_jobserver
    @job_name = N'LogiFleet_BottleneckMonitor';
GO
```

## Common Patterns

### Pattern 1: Daily ETL Pipeline

```sql
-- Master ETL procedure
CREATE PROCEDURE sp_DailyETLPipeline
AS
BEGIN
    SET NOCOUNT ON;
    BEGIN TRY
        BEGIN TRANSACTION;
        
        -- 1. Load warehouse operations
        DECLARE @LastWarehouseLoad DATETIME2;
        SELECT @LastWarehouseLoad = MAX(LoadTimestamp) 
        FROM etl.LoadLog WHERE TableName = 'FactWarehouseOperations';
        
        EXEC sp_IncrementalLoadWarehouseOps @LastLoadTime = @LastWarehouseLoad;
        
        -- 2. Load fleet trips
        EXEC sp_IncrementalLoadFleetTrips @LastLoadTime = @LastWarehouseLoad;
        
        -- 3. Recalculate product gravity scores
        EXEC sp_RecalculateProductGravity;
        
        -- 4. Update supplier reliability metrics
        EXEC sp_UpdateSupplierReliability;
        
        -- 5. Log completion
        INSERT INTO etl.LoadLog (TableName, LoadTimestamp, RowsAffected)
        VALUES ('DailyETL', GETDATE(), @@ROWCOUNT);
        
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        
        -- Log error
        INSERT INTO etl.ErrorLog (ErrorMessage, ErrorTimestamp)
        VALUES (ERROR_MESSAGE(), GETDATE());
        
        THROW;
    END CATCH
END;
```

### Pattern 2: Real-Time Streaming Integration

```sql
-- External table for streaming telemetry data
CREATE EXTERNAL DATA SOURCE TelemetryStream
WITH (
    TYPE = BLOB_STORAGE,
    LOCATION = '${AZURE_BLOB_ENDPOINT}',
    CREDENTIAL = TelemetryCredential
);

CREATE EXTERNAL FILE FORMAT TelemetryJSON
WITH (FORMAT_TYPE = JSON);

CREATE EXTERNAL TABLE staging.StreamingTelemetry (
    VehicleID VARCHAR(50),
    Timestamp DATETIME2,
    Latitude DECIMAL(10,8),
    Longitude DECIMAL(11,8),
    Speed DECIMAL(5,2),
    FuelLevel DECIMAL(5,2),
    EngineTemp DECIMAL(5,2)
)
WITH (
    LOCATION = '/telemetry/',
    DATA_SOURCE = TelemetryStream,
    FILE_FORMAT = TelemetryJSON
);

-- Continuous insert from stream
CREATE PROCEDURE sp_ProcessStreamingTelemetry
AS
BEGIN
    INSERT INTO FactFleetTrips (
        StartTimeKey, VehicleID, OriginKey, DestinationKey,
        DistanceKm, FuelConsumedLiters
    )
    SELECT 
        t.TimeKey,
        s.VehicleID,
        geo_start.GeographyKey,
        geo_end.GeographyKey,
        geography::Point(s_start.Latitude, s_start.Longitude, 4326)
            .STDistance(geography
