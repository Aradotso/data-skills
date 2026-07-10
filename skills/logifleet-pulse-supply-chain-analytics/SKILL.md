---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing template for logistics, fleet management, and supply chain analytics with multi-fact star schema
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "deploy the logistics data warehouse schema"
  - "configure Power BI logistics dashboard"
  - "create warehouse operations fact tables"
  - "implement fleet analytics star schema"
  - "build cross-modal supply chain KPI reports"
  - "integrate logistics telemetry with SQL Server"
  - "optimize warehouse gravity zone queries"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive data warehousing and business intelligence template for logistics and supply chain operations. It provides:

- **Multi-fact star schema** for warehouse operations, fleet trips, and cross-dock activities
- **Power BI dashboard templates** with real-time KPI tracking
- **MS SQL Server database scripts** with optimized indexing and partitioning
- **Unified semantic layer** linking warehouse, fleet, and supplier data
- **Predictive analytics** for bottleneck detection and fleet optimization

The project bridges warehouse management systems (WMS), telematics feeds, and procurement data into a single analytical platform.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQLCMD or Azure Data Studio for script execution
- Access to source systems (WMS, TMS, ERP) via SQL, API, or flat files

### Database Deployment

1. **Clone the repository**:
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the schema**:
```bash
sqlcmd -S your-server-name -d LogiFleetPulse -U your-username -P $SQL_PASSWORD -i schema/01_create_dimensions.sql
sqlcmd -S your-server-name -d LogiFleetPulse -U your-username -P $SQL_PASSWORD -i schema/02_create_facts.sql
sqlcmd -S your-server-name -d LogiFleetPulse -U your-username -P $SQL_PASSWORD -i schema/03_create_bridges.sql
sqlcmd -S your-server-name -d LogiFleetPulse -U your-username -P $SQL_PASSWORD -i schema/04_create_views.sql
sqlcmd -S your-server-name -d LogiFleetPulse -U your-username -P $SQL_PASSWORD -i schema/05_create_stored_procedures.sql
```

3. **Configure data sources** in `config.json`:
```json
{
  "sql_server": {
    "server": "your-server.database.windows.net",
    "database": "LogiFleetPulse",
    "username": "sqladmin",
    "password": "${SQL_PASSWORD}"
  },
  "wms_api": {
    "endpoint": "https://wms.example.com/api/v1",
    "api_key": "${WMS_API_KEY}"
  },
  "telematics_feed": {
    "endpoint": "https://fleet.example.com/telemetry",
    "auth_token": "${FLEET_TOKEN}"
  }
}
```

### Power BI Setup

1. **Open the template**:
```bash
# Open LogiFleet_Pulse_Master.pbit in Power BI Desktop
```

2. **Update data source connections** in Power Query:
```m
let
    Source = Sql.Database(
        "your-server-name", 
        "LogiFleetPulse",
        [Query="SELECT * FROM vw_WarehouseOperationsDashboard"]
    )
in
    Source
```

## Key Schema Components

### Dimension Tables

**DimTime** - 15-minute granularity time dimension:
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    [Date] DATE NOT NULL,
    [Year] INT NOT NULL,
    Quarter INT NOT NULL,
    Month INT NOT NULL,
    [DayOfWeek] INT NOT NULL,
    [Hour] INT NOT NULL,
    MinuteBucket INT NOT NULL, -- 0, 15, 30, 45
    FiscalYear INT NOT NULL,
    FiscalQuarter INT NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT NOT NULL
);

-- Create indexes
CREATE NONCLUSTERED INDEX IX_DimTime_Date ON DimTime([Date]);
CREATE NONCLUSTERED INDEX IX_DimTime_Hour ON DimTime([Hour]);
```

**DimProductGravity** - Product classification with velocity scoring:
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU NVARCHAR(50) NOT NULL UNIQUE,
    ProductName NVARCHAR(200) NOT NULL,
    Category NVARCHAR(100),
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value / fragility
    VelocityClass NVARCHAR(20), -- Fast, Medium, Slow
    ValueTier NVARCHAR(20), -- High, Medium, Low
    FragilityIndex DECIMAL(3,2),
    OptimalZone NVARCHAR(50), -- Recommended warehouse zone
    LastRecalculated DATETIME DEFAULT GETDATE()
);

CREATE NONCLUSTERED INDEX IX_DimProductGravity_SKU ON DimProductGravity(SKU);
CREATE NONCLUSTERED INDEX IX_DimProductGravity_GravityScore ON DimProductGravity(GravityScore DESC);
```

**DimGeography** - Hierarchical location dimension:
```sql
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID NVARCHAR(50) NOT NULL UNIQUE,
    LocationName NVARCHAR(200) NOT NULL,
    LocationType NVARCHAR(50), -- Warehouse, Hub, Route Node
    Continent NVARCHAR(50),
    Country NVARCHAR(100),
    Region NVARCHAR(100),
    City NVARCHAR(100),
    PostalCode NVARCHAR(20),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    ParentLocationKey INT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey)
);

CREATE NONCLUSTERED INDEX IX_DimGeography_LocationType ON DimGeography(LocationType);
CREATE NONCLUSTERED INDEX IX_DimGeography_Country_Region ON DimGeography(Country, Region);
```

### Fact Tables

**FactWarehouseOperations** - Warehouse activity tracking:
```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    OperationID NVARCHAR(50) NOT NULL UNIQUE,
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    OperationType NVARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    Quantity INT NOT NULL,
    DurationMinutes INT,
    DwellTimeHours DECIMAL(10,2),
    StorageZone NVARCHAR(50),
    OperatorID NVARCHAR(50),
    EquipmentID NVARCHAR(50),
    ErrorFlag BIT DEFAULT 0,
    ErrorType NVARCHAR(100),
    ProcessedDate DATETIME DEFAULT GETDATE()
);

-- Columnstore index for analytical queries
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactWarehouseOperations_Analytics 
ON FactWarehouseOperations (TimeKey, ProductKey, GeographyKey, Quantity, DurationMinutes, DwellTimeHours);

-- Partition by month for performance
CREATE PARTITION FUNCTION PF_WarehouseOps_Month (INT)
AS RANGE RIGHT FOR VALUES (202601, 202602, 202603, 202604, 202605, 202606, 202607, 202608, 202609, 202610, 202611, 202612);
```

**FactFleetTrips** - Fleet telemetry and route data:
```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TripID NVARCHAR(50) NOT NULL UNIQUE,
    TimeKeyStart INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    TimeKeyEnd INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKeyOrigin INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    GeographyKeyDestination INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID NVARCHAR(50) NOT NULL,
    DriverID NVARCHAR(50),
    DistanceKM DECIMAL(10,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelLiters DECIMAL(10,2),
    LoadWeightKG DECIMAL(10,2),
    AverageSpeedKPH DECIMAL(5,2),
    DelayMinutes INT DEFAULT 0,
    DelayReason NVARCHAR(200),
    WeatherCondition NVARCHAR(50),
    TrafficIndex DECIMAL(3,2), -- 1.0 = normal, >1.0 = congestion
    ProcessedDate DATETIME DEFAULT GETDATE()
);

CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactFleetTrips_Analytics 
ON FactFleetTrips (TimeKeyStart, GeographyKeyOrigin, GeographyKeyDestination, DistanceKM, DurationMinutes, FuelLiters);

CREATE NONCLUSTERED INDEX IX_FactFleetTrips_VehicleID ON FactFleetTrips(VehicleID);
```

**FactCrossDock** - Cross-docking operations:
```sql
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    CrossDockID NVARCHAR(50) NOT NULL UNIQUE,
    TimeKeyInbound INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    TimeKeyOutbound INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    Quantity INT NOT NULL,
    DwellTimeMinutes INT,
    QualityCheckPassed BIT DEFAULT 1,
    ProcessedDate DATETIME DEFAULT GETDATE()
);

CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactCrossDock_Analytics 
ON FactCrossDock (TimeKeyInbound, TimeKeyOutbound, ProductKey, Quantity, DwellTimeMinutes);
```

## Key Stored Procedures

### Data Loading Procedure

```sql
CREATE PROCEDURE sp_LoadWarehouseOperations
    @SourceTable NVARCHAR(200),
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRY
        BEGIN TRANSACTION;
        
        -- Merge operation data
        MERGE FactWarehouseOperations AS target
        USING (
            SELECT 
                o.OperationID,
                t.TimeKey,
                p.ProductKey,
                g.GeographyKey,
                o.OperationType,
                o.Quantity,
                DATEDIFF(MINUTE, o.StartTime, o.EndTime) AS DurationMinutes,
                DATEDIFF(HOUR, o.StartTime, COALESCE(o.ShippedTime, GETDATE())) AS DwellTimeHours,
                o.StorageZone,
                o.OperatorID,
                o.EquipmentID,
                o.ErrorFlag,
                o.ErrorType
            FROM @SourceTable o
            INNER JOIN DimTime t ON CAST(o.StartTime AS DATE) = t.[Date] 
                AND DATEPART(HOUR, o.StartTime) = t.[Hour]
                AND (DATEPART(MINUTE, o.StartTime) / 15) * 15 = t.MinuteBucket
            INNER JOIN DimProductGravity p ON o.SKU = p.SKU
            INNER JOIN DimGeography g ON o.WarehouseCode = g.LocationID
            WHERE CAST(o.StartTime AS DATE) BETWEEN @StartDate AND @EndDate
        ) AS source
        ON target.OperationID = source.OperationID
        WHEN MATCHED THEN
            UPDATE SET 
                target.DwellTimeHours = source.DwellTimeHours,
                target.ErrorFlag = source.ErrorFlag,
                target.ErrorType = source.ErrorType
        WHEN NOT MATCHED THEN
            INSERT (OperationID, TimeKey, ProductKey, GeographyKey, OperationType, 
                    Quantity, DurationMinutes, DwellTimeHours, StorageZone, 
                    OperatorID, EquipmentID, ErrorFlag, ErrorType)
            VALUES (source.OperationID, source.TimeKey, source.ProductKey, source.GeographyKey, 
                    source.OperationType, source.Quantity, source.DurationMinutes, 
                    source.DwellTimeHours, source.StorageZone, source.OperatorID, 
                    source.EquipmentID, source.ErrorFlag, source.ErrorType);
        
        COMMIT TRANSACTION;
        
        PRINT 'Successfully loaded warehouse operations from ' + CAST(@StartDate AS NVARCHAR) + ' to ' + CAST(@EndDate AS NVARCHAR);
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        
        DECLARE @ErrorMessage NVARCHAR(4000) = ERROR_MESSAGE();
        RAISERROR(@ErrorMessage, 16, 1);
    END CATCH
END;
GO
```

### KPI Calculation Procedure

```sql
CREATE PROCEDURE sp_CalculateFleetEfficiency
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    WITH FleetMetrics AS (
        SELECT
            f.VehicleID,
            g_origin.Country AS OriginCountry,
            g_dest.Country AS DestinationCountry,
            COUNT(*) AS TotalTrips,
            SUM(f.DistanceKM) AS TotalDistanceKM,
            SUM(f.DurationMinutes) AS TotalDurationMinutes,
            SUM(f.IdleTimeMinutes) AS TotalIdleTimeMinutes,
            SUM(f.FuelLiters) AS TotalFuelLiters,
            AVG(f.AverageSpeedKPH) AS AvgSpeedKPH,
            SUM(CASE WHEN f.DelayMinutes > 0 THEN 1 ELSE 0 END) AS DelayedTrips
        FROM FactFleetTrips f
        INNER JOIN DimTime t ON f.TimeKeyStart = t.TimeKey
        INNER JOIN DimGeography g_origin ON f.GeographyKeyOrigin = g_origin.GeographyKey
        INNER JOIN DimGeography g_dest ON f.GeographyKeyDestination = g_dest.GeographyKey
        WHERE t.[Date] BETWEEN @StartDate AND @EndDate
        GROUP BY f.VehicleID, g_origin.Country, g_dest.Country
    )
    SELECT
        VehicleID,
        OriginCountry,
        DestinationCountry,
        TotalTrips,
        TotalDistanceKM,
        TotalDurationMinutes,
        TotalIdleTimeMinutes,
        TotalFuelLiters,
        -- Efficiency metrics
        CAST(TotalIdleTimeMinutes AS DECIMAL(10,2)) / NULLIF(TotalDurationMinutes, 0) * 100 AS IdleTimePercentage,
        CAST(TotalFuelLiters AS DECIMAL(10,2)) / NULLIF(TotalDistanceKM, 0) AS FuelEfficiencyLitersPer100KM,
        CAST(DelayedTrips AS DECIMAL(10,2)) / NULLIF(TotalTrips, 0) * 100 AS DelayRate,
        AvgSpeedKPH
    FROM FleetMetrics
    ORDER BY TotalDistanceKM DESC;
END;
GO
```

### Alert Trigger Procedure

```sql
CREATE PROCEDURE sp_CheckAnomalies
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Check for high dwell time
    INSERT INTO AlertLog (AlertType, AlertLevel, Message, EntityID, EntityType, DetectedAt)
    SELECT 
        'HighDwellTime' AS AlertType,
        'Warning' AS AlertLevel,
        'Product ' + p.SKU + ' has dwell time of ' + CAST(w.DwellTimeHours AS NVARCHAR) + ' hours at ' + g.LocationName AS Message,
        w.OperationID AS EntityID,
        'WarehouseOperation' AS EntityType,
        GETDATE() AS DetectedAt
    FROM FactWarehouseOperations w
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    INNER JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
    WHERE w.DwellTimeHours > 72
        AND w.OperationType = 'Receiving'
        AND CAST(w.ProcessedDate AS DATE) = CAST(GETDATE() AS DATE);
    
    -- Check for excessive fleet idling
    INSERT INTO AlertLog (AlertType, AlertLevel, Message, EntityID, EntityType, DetectedAt)
    SELECT 
        'ExcessiveIdling' AS AlertType,
        'Critical' AS AlertLevel,
        'Vehicle ' + f.VehicleID + ' has idle time of ' + CAST(f.IdleTimeMinutes AS NVARCHAR) + ' minutes (' + 
        CAST(CAST(f.IdleTimeMinutes AS DECIMAL(10,2)) / NULLIF(f.DurationMinutes, 0) * 100 AS NVARCHAR) + '%)',
        f.TripID AS EntityID,
        'FleetTrip' AS EntityType,
        GETDATE() AS DetectedAt
    FROM FactFleetTrips f
    WHERE CAST(f.IdleTimeMinutes AS DECIMAL(10,2)) / NULLIF(f.DurationMinutes, 0) > 0.15
        AND CAST(f.ProcessedDate AS DATE) = CAST(GETDATE() AS DATE);
    
    PRINT 'Anomaly check completed. ' + CAST(@@ROWCOUNT AS NVARCHAR) + ' alerts generated.';
END;
GO
```

## Power BI DAX Measures

### Cross-Fact KPI Harmonization

```dax
// Warehouse-Fleet Efficiency Score
WarehouseFleetEfficiency = 
VAR AvgDwellTime = AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR AvgIdlePercentage = DIVIDE(SUM(FactFleetTrips[IdleTimeMinutes]), SUM(FactFleetTrips[DurationMinutes]))
VAR DwellNormalized = 1 - (AvgDwellTime / 168) // 168 hours = 1 week
VAR IdleNormalized = 1 - AvgIdlePercentage
VAR Score = (DwellNormalized * 0.6) + (IdleNormalized * 0.4)
RETURN Score
```

### Predictive Bottleneck Index

```dax
// Bottleneck Risk Score
BottleneckRisk = 
VAR CurrentCapacity = SUM(FactWarehouseOperations[Quantity])
VAR MaxCapacity = MAX(DimGeography[StorageCapacity])
VAR CapacityUtilization = DIVIDE(CurrentCapacity, MaxCapacity)
VAR HighGravityItems = CALCULATE(
    COUNTROWS(FactWarehouseOperations),
    DimProductGravity[VelocityClass] = "Fast"
)
VAR TotalItems = COUNTROWS(FactWarehouseOperations)
VAR HighGravityPercentage = DIVIDE(HighGravityItems, TotalItems)
VAR RiskScore = (CapacityUtilization * 0.7) + (HighGravityPercentage * 0.3)
RETURN 
    SWITCH(
        TRUE(),
        RiskScore > 0.8, "Critical",
        RiskScore > 0.6, "High",
        RiskScore > 0.4, "Moderate",
        "Low"
    )
```

### Temporal Elasticity Model

```dax
// Projected Fleet Utilization (Next 30 Days)
ProjectedFleetUtilization = 
VAR HistoricalAvg = 
    CALCULATE(
        DIVIDE(SUM(FactFleetTrips[DurationMinutes]), 
               DISTINCTCOUNT(FactFleetTrips[VehicleID]) * 60 * 24),
        DATESINPERIOD(DimTime[Date], TODAY(), -90, DAY)
    )
VAR SeasonalFactor = 
    CALCULATE(
        DIVIDE(
            CALCULATE(SUM(FactFleetTrips[DurationMinutes]), DimTime[Month] = MONTH(TODAY())),
            CALCULATE(SUM(FactFleetTrips[DurationMinutes]))
        ),
        DATESINPERIOD(DimTime[Date], TODAY(), -365, DAY)
    )
VAR Projection = HistoricalAvg * SeasonalFactor * 1.05 // 5% growth assumption
RETURN Projection
```

### Gravity Zone Optimization

```dax
// Optimal Zone Reassignment Candidates
ZoneOptimizationCandidates = 
CALCULATE(
    COUNTROWS(FactWarehouseOperations),
    FILTER(
        DimProductGravity,
        DimProductGravity[VelocityClass] <> 
        SWITCH(
            TRUE(),
            DimProductGravity[GravityScore] > 8, "Fast",
            DimProductGravity[GravityScore] > 5, "Medium",
            "Slow"
        )
    )
)
```

## Common Query Patterns

### Cross-Fact Analysis: Dwell Time vs. Fuel Efficiency

```sql
SELECT 
    p.Category,
    AVG(w.DwellTimeHours) AS AvgDwellTimeHours,
    AVG(CAST(f.FuelLiters AS DECIMAL(10,2)) / NULLIF(f.DistanceKM, 0) * 100) AS AvgFuelPer100KM,
    COUNT(DISTINCT w.OperationID) AS TotalWarehouseOps,
    COUNT(DISTINCT f.TripID) AS TotalTrips
FROM FactWarehouseOperations w
INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
INNER JOIN FactFleetTrips f ON 
    CAST(w.TimeKey AS DATE) = CAST(f.TimeKeyStart AS DATE)
    AND w.GeographyKey = f.GeographyKeyOrigin
WHERE w.OperationType = 'Shipping'
    AND f.DelayMinutes = 0
GROUP BY p.Category
ORDER BY AvgDwellTimeHours DESC;
```

### Warehouse Gravity Zone Performance

```sql
WITH ZoneMetrics AS (
    SELECT 
        w.StorageZone,
        p.VelocityClass,
        COUNT(*) AS OperationCount,
        AVG(w.DurationMinutes) AS AvgPickTime,
        AVG(w.DwellTimeHours) AS AvgDwellTime
    FROM FactWarehouseOperations w
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    WHERE w.OperationType = 'Picking'
        AND w.ErrorFlag = 0
    GROUP BY w.StorageZone, p.VelocityClass
)
SELECT 
    StorageZone,
    VelocityClass,
    OperationCount,
    AvgPickTime,
    AvgDwellTime,
    CASE 
        WHEN VelocityClass = 'Fast' AND AvgPickTime > 10 THEN 'Needs Optimization'
        WHEN VelocityClass = 'Fast' AND AvgPickTime <= 10 THEN 'Optimal'
        WHEN VelocityClass = 'Medium' AND AvgPickTime > 15 THEN 'Needs Optimization'
        WHEN VelocityClass = 'Medium' AND AvgPickTime <= 15 THEN 'Optimal'
        ELSE 'Acceptable'
    END AS ZoneEfficiency
FROM ZoneMetrics
ORDER BY VelocityClass, AvgPickTime;
```

### Route Delay Root Cause Analysis

```sql
SELECT 
    f.DelayReason,
    f.WeatherCondition,
    g_origin.Region AS OriginRegion,
    g_dest.Region AS DestinationRegion,
    COUNT(*) AS DelayCount,
    AVG(f.DelayMinutes) AS AvgDelayMinutes,
    SUM(f.DelayMinutes) AS TotalDelayMinutes
FROM FactFleetTrips f
INNER JOIN DimGeography g_origin ON f.GeographyKeyOrigin = g_origin.GeographyKey
INNER JOIN DimGeography g_dest ON f.GeographyKeyDestination = g_dest.GeographyKey
WHERE f.DelayMinutes > 0
    AND f.ProcessedDate >= DATEADD(MONTH, -1, GETDATE())
GROUP BY f.DelayReason, f.WeatherCondition, g_origin.Region, g_dest.Region
HAVING COUNT(*) > 5
ORDER BY TotalDelayMinutes DESC;
```

## Configuration Best Practices

### Connection String Configuration

Store connection details in environment variables:

```bash
# .env file (never commit this)
export SQL_SERVER="your-server.database.windows.net"
export SQL_DATABASE="LogiFleetPulse"
export SQL_USERNAME="sqladmin"
export SQL_PASSWORD="your-secure-password"
export WMS_API_KEY="your-wms-api-key"
export FLEET_API_TOKEN="your-fleet-token"
```

### Incremental Data Loading

Schedule data loads using SQL Server Agent:

```sql
-- Create job for hourly data refresh
EXEC sp_add_job 
    @job_name = 'LogiFleet_Hourly_Refresh',
    @enabled = 1,
    @description = 'Incremental load of warehouse and fleet data';

EXEC sp_add_jobstep
    @job_name = 'LogiFleet_Hourly_Refresh',
    @step_name = 'Load Warehouse Operations',
    @subsystem = 'TSQL',
    @command = 'EXEC sp_LoadWarehouseOperations @StartDate = CAST(DATEADD(HOUR, -1, GETDATE()) AS DATE), @EndDate = CAST(GETDATE() AS DATE);';

EXEC sp_add_schedule
    @schedule_name = 'Hourly',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 8, -- Hours
    @freq_subday_interval = 1;

EXEC sp_attach_schedule
    @job_name = 'LogiFleet_Hourly_Refresh',
    @schedule_name = 'Hourly';
```

### Row-Level Security Setup

```sql
-- Create security predicate for role-based access
CREATE FUNCTION dbo.fn_SecurityPredicate(@GeographyKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS result
WHERE @GeographyKey IN (
    SELECT GeographyKey 
    FROM dbo.UserGeographyAccess 
    WHERE UserName = USER_NAME()
);
GO

CREATE SECURITY POLICY WarehouseSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(GeographyKey)
ON dbo.FactWarehouseOperations,
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(GeographyKeyOrigin)
ON dbo.FactFleetTrips
WITH (STATE = ON);
GO
```

## Troubleshooting

### Power BI Refresh Failures

**Issue**: "Data refresh failed - timeout error"

**Solution**: Implement query folding and partitioning:

```m
// Power Query - Partitioned Load
let
    StartDate = Date.AddDays(DateTime.Date(DateTime.LocalNow()), -30),
    EndDate = DateTime.Date(DateTime.LocalNow()),
    Source = Sql.Database("your-server", "LogiFleetPulse"),
    FilteredRows = Table.SelectRows(
        Source{[Schema="dbo",Item="FactWarehouseOperations"]}[Data],
        each [ProcessedDate] >= StartDate and [ProcessedDate] <= EndDate
    )
in
    FilteredRows
```

### Slow Cross-Fact Queries

**Issue**: Queries joining multiple fact tables are slow

**Solution**: Create indexed views or aggregate tables:

```sql
CREATE VIEW vw_WarehouseFleetSummary
WITH SCHEMABINDING
AS
SELECT 
    t.[Date],
    g.Country,
    p.Category,
    COUNT_BIG(*) AS OperationCount,
    SUM(w.Quantity) AS TotalQuantity,
    AVG(w.DwellTimeHours) AS AvgDwellTime,
    SUM(f.FuelLiters) AS TotalFuelLiters
FROM dbo.FactWarehouseOperations w
INNER JOIN dbo.DimTime t ON w.TimeKey = t.TimeKey
INNER JOIN dbo.DimGeography g ON w.GeographyKey = g.GeographyKey
INNER JOIN dbo.DimProductGravity p ON w.ProductKey = p.ProductKey
LEFT JOIN dbo.FactFleetTrips f ON 
    t.[Date] = (SELECT [Date] FROM dbo.DimTime WHERE TimeKey = f.TimeKeyStart)
    AND w.GeographyKey = f.GeographyKeyOrigin
GROUP BY t.[Date], g.Country, p.Category;
GO

CREATE UNIQUE CLUSTERED INDEX IX_WarehouseFleetSummary 
ON vw_WarehouseFleetSummary([Date], Country, Category);
```

### Missing Dimension Values

**Issue**: Fact records with NULL dimension keys

**Solution**: Create "Unknown" dimension records:

```sql
--
