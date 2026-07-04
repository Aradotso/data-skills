---
name: logifleet-pulse-supply-chain-analytics
description: Multi-modal logistics intelligence engine with MS SQL Server data warehousing and Power BI dashboards for real-time supply chain analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure warehouse and fleet logistics dashboard"
  - "create multi-fact star schema for logistics"
  - "implement Power BI supply chain visualization"
  - "build warehouse gravity zone analytics"
  - "deploy SQL Server logistics data warehouse"
  - "integrate fleet telemetry with warehouse operations"
  - "create cross-modal supply chain KPIs"
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

LogiFleet Pulse is a multi-modal logistics intelligence engine that combines warehouse operations, fleet telemetry, inventory management, and external data sources into a unified semantic layer. Built on MS SQL Server with Power BI visualization, it provides real-time cross-fact KPI harmonization for supply chain decision-making.

## What It Does

- **Unified Data Warehouse**: Multi-fact star schema linking warehouse operations, fleet trips, cross-dock transfers, and supplier data
- **Real-Time Dashboards**: Power BI templates refreshed every 15 minutes for operational awareness
- **Cross-Fact Analytics**: Query relationships between warehouse dwell time, fleet fuel consumption, and delivery performance
- **Predictive Insights**: Bottleneck detection, maintenance prioritization, and warehouse layout optimization
- **Temporal Modeling**: Time-phased simulations for capacity planning and route optimization

## Installation

### Prerequisites

- MS SQL Server 2019+ (Developer, Standard, or Enterprise edition)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to source systems (WMS, TMS, telemetry APIs)

### Database Setup

1. **Clone the repository**:
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema**:
```sql
-- Connect to your SQL Server instance
-- Create the database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Run the schema creation script
-- Execute scripts/01_CreateSchema.sql
-- Execute scripts/02_CreateDimensions.sql
-- Execute scripts/03_CreateFacts.sql
-- Execute scripts/04_CreateViews.sql
-- Execute scripts/05_CreateStoredProcedures.sql
```

3. **Configure data connections**:
```json
{
  "connections": {
    "wms_api": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "api_key": "${WMS_API_KEY}"
    },
    "fleet_telemetry": {
      "endpoint": "${FLEET_API_ENDPOINT}",
      "api_key": "${FLEET_API_KEY}"
    },
    "weather_api": {
      "endpoint": "https://api.weatherapi.com/v1",
      "api_key": "${WEATHER_API_KEY}"
    }
  }
}
```

## Core Database Schema

### Dimension Tables

```sql
-- Time Dimension (15-minute granularity)
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    [Date] DATE NOT NULL,
    [Year] INT NOT NULL,
    [Quarter] INT NOT NULL,
    [Month] INT NOT NULL,
    [Week] INT NOT NULL,
    [DayOfYear] INT NOT NULL,
    [DayOfMonth] INT NOT NULL,
    [DayOfWeek] INT NOT NULL,
    [Hour] INT NOT NULL,
    [Minute] INT NOT NULL,
    FiscalYear INT NOT NULL,
    FiscalQuarter INT NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT NOT NULL
)

CREATE CLUSTERED INDEX IX_DimTime_Date ON DimTime([Date], [Hour], [Minute])
```

```sql
-- Product with Gravity Score
CREATE TABLE DimProduct (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    ProductID NVARCHAR(50) NOT NULL UNIQUE,
    SKU NVARCHAR(50) NOT NULL,
    ProductName NVARCHAR(200) NOT NULL,
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    UnitWeight DECIMAL(10,2),
    UnitVolume DECIMAL(10,2),
    IsFragile BIT DEFAULT 0,
    IsPerishable BIT DEFAULT 0,
    ShelfLife INT, -- days
    GravityScore DECIMAL(5,2), -- calculated: velocity + value + fragility
    ValidFrom DATETIME2 DEFAULT GETDATE(),
    ValidTo DATETIME2 DEFAULT '9999-12-31',
    IsCurrent BIT DEFAULT 1
)

CREATE INDEX IX_DimProduct_GravityScore ON DimProduct(GravityScore DESC)
```

```sql
-- Geography Hierarchy
CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationID NVARCHAR(50) NOT NULL UNIQUE,
    LocationName NVARCHAR(200) NOT NULL,
    LocationType NVARCHAR(50), -- Warehouse, Route Node, Distribution Center
    AddressLine1 NVARCHAR(200),
    City NVARCHAR(100),
    StateProvince NVARCHAR(100),
    PostalCode NVARCHAR(20),
    Country NVARCHAR(100),
    Region NVARCHAR(100),
    Continent NVARCHAR(50),
    Latitude DECIMAL(10,6),
    Longitude DECIMAL(10,6),
    TimeZone NVARCHAR(50)
)

CREATE INDEX IX_DimGeography_Type ON DimGeography(LocationType)
```

```sql
-- Supplier Reliability
CREATE TABLE DimSupplier (
    SupplierKey INT IDENTITY(1,1) PRIMARY KEY,
    SupplierID NVARCHAR(50) NOT NULL UNIQUE,
    SupplierName NVARCHAR(200) NOT NULL,
    LeadTimeDaysAvg DECIMAL(5,1),
    LeadTimeDaysStdDev DECIMAL(5,1),
    DefectRatePercent DECIMAL(5,2),
    OnTimeDeliveryPercent DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2), -- 0-100
    ReliabilityScore AS (
        (OnTimeDeliveryPercent * 0.4) + 
        ((100 - DefectRatePercent) * 0.3) + 
        (ComplianceScore * 0.3)
    ) PERSISTED
)
```

### Fact Tables

```sql
-- Warehouse Operations
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProduct(ProductKey),
    WarehouseKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    OperationType NVARCHAR(50), -- Receiving, Putaway, Picking, Packing, Shipping
    Quantity INT NOT NULL,
    DwellTimeMinutes INT,
    ProcessingTimeMinutes DECIMAL(10,2),
    ZoneID NVARCHAR(50),
    GravityZone NVARCHAR(50), -- High, Medium, Low
    StaffMembersAssigned INT,
    EquipmentUsed NVARCHAR(100),
    BatchNumber NVARCHAR(50),
    CreatedAt DATETIME2 DEFAULT GETDATE()
)

CREATE CLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey, ProductKey)
CREATE INDEX IX_FactWarehouse_GravityZone ON FactWarehouseOperations(GravityZone, DwellTimeMinutes)
```

```sql
-- Fleet Trips
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleKey INT NOT NULL FOREIGN KEY REFERENCES DimVehicle(VehicleKey),
    DriverKey INT NOT NULL FOREIGN KEY REFERENCES DimDriver(DriverKey),
    OriginKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DistanceKm DECIMAL(10,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(10,2),
    AverageSpeedKmh DECIMAL(5,2),
    LoadWeightKg DECIMAL(10,2),
    WeatherCondition NVARCHAR(50),
    TrafficDelayMinutes INT,
    MaintenanceIssues NVARCHAR(500),
    MaintenancePriorityScore DECIMAL(5,2),
    TripStatus NVARCHAR(50), -- Completed, InProgress, Delayed, Cancelled
    CreatedAt DATETIME2 DEFAULT GETDATE()
)

CREATE CLUSTERED INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey, VehicleKey)
CREATE INDEX IX_FactFleet_Status ON FactFleetTrips(TripStatus, TimeKey)
```

```sql
-- Cross-Dock Operations (Bridge Table)
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    InboundOperationKey BIGINT FOREIGN KEY REFERENCES FactWarehouseOperations(OperationKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProduct(ProductKey),
    Quantity INT NOT NULL,
    DockDwellMinutes INT,
    DirectTransfer BIT DEFAULT 1, -- True if no intermediate storage
    CreatedAt DATETIME2 DEFAULT GETDATE()
)

CREATE INDEX IX_FactCrossDock_Dwell ON FactCrossDock(DockDwellMinutes DESC)
```

## Key Stored Procedures

### Incremental Data Loading

```sql
CREATE PROCEDURE usp_LoadWarehouseOperations
    @StartTime DATETIME2,
    @EndTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, WarehouseKey, OperationType,
        Quantity, DwellTimeMinutes, ProcessingTimeMinutes,
        ZoneID, GravityZone, StaffMembersAssigned
    )
    SELECT 
        dt.TimeKey,
        dp.ProductKey,
        dg.GeographyKey,
        src.OperationType,
        src.Quantity,
        DATEDIFF(MINUTE, src.StartTime, src.EndTime) AS DwellTimeMinutes,
        src.ProcessingTimeMinutes,
        src.ZoneID,
        CASE 
            WHEN dp.GravityScore >= 80 THEN 'High'
            WHEN dp.GravityScore >= 40 THEN 'Medium'
            ELSE 'Low'
        END AS GravityZone,
        src.StaffCount
    FROM ${WMS_STAGING_TABLE} src
    INNER JOIN DimTime dt ON CAST(src.StartTime AS DATETIME2) = dt.FullDateTime
    INNER JOIN DimProduct dp ON src.SKU = dp.SKU AND dp.IsCurrent = 1
    INNER JOIN DimGeography dg ON src.WarehouseID = dg.LocationID
    WHERE src.StartTime >= @StartTime
      AND src.StartTime < @EndTime
      AND NOT EXISTS (
          SELECT 1 FROM FactWarehouseOperations f
          WHERE f.TimeKey = dt.TimeKey
            AND f.ProductKey = dp.ProductKey
            AND f.OperationType = src.OperationType
      )
    
    RETURN @@ROWCOUNT
END
```

### Gravity Score Calculation

```sql
CREATE PROCEDURE usp_UpdateProductGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Update gravity scores based on 90-day rolling metrics
    UPDATE dp
    SET GravityScore = (
        -- Velocity component (40%)
        (ISNULL(metrics.PickFrequency, 0) * 0.4) +
        -- Value component (30%)
        (ISNULL(metrics.AvgOrderValue, 0) / 1000 * 0.3) +
        -- Fragility/Priority component (30%)
        ((CASE WHEN dp.IsFragile = 1 THEN 25 ELSE 0 END +
          CASE WHEN dp.IsPerishable = 1 THEN 25 ELSE 0 END) * 0.3)
    )
    FROM DimProduct dp
    OUTER APPLY (
        SELECT 
            COUNT(*) * 100.0 / NULLIF(DATEDIFF(DAY, MIN(dt.Date), MAX(dt.Date)), 0) AS PickFrequency,
            AVG(f.Quantity * dp.UnitWeight) AS AvgOrderValue
        FROM FactWarehouseOperations f
        INNER JOIN DimTime dt ON f.TimeKey = dt.TimeKey
        WHERE f.ProductKey = dp.ProductKey
          AND f.OperationType = 'Picking'
          AND dt.Date >= DATEADD(DAY, -90, GETDATE())
    ) metrics
    WHERE dp.IsCurrent = 1
END
```

### Cross-Fact KPI Query

```sql
CREATE PROCEDURE usp_GetDwellTimeVsFuelCost
    @StartDate DATE,
    @EndDate DATE,
    @MinGravityScore DECIMAL(5,2) = 0
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Correlate warehouse dwell time with fleet fuel consumption
    SELECT 
        dp.Category,
        dp.GravityScore,
        AVG(fwo.DwellTimeMinutes) AS AvgWarehouseDwellMinutes,
        AVG(fft.FuelConsumedLiters) AS AvgFuelPerTrip,
        AVG(fft.IdleTimeMinutes) AS AvgFleetIdleMinutes,
        COUNT(DISTINCT fwo.OperationKey) AS TotalWarehouseOps,
        COUNT(DISTINCT fft.TripKey) AS TotalFleetTrips,
        -- Correlation metric: high dwell + high idle = inefficiency
        (AVG(fwo.DwellTimeMinutes) * AVG(fft.IdleTimeMinutes)) / 100.0 AS InefficencyIndex
    FROM FactWarehouseOperations fwo
    INNER JOIN DimProduct dp ON fwo.ProductKey = dp.ProductKey
    INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    INNER JOIN FactCrossDock fcd ON fwo.OperationKey = fcd.InboundOperationKey
    INNER JOIN FactFleetTrips fft ON fcd.OutboundTripKey = fft.TripKey
    WHERE dt.Date BETWEEN @StartDate AND @EndDate
      AND dp.GravityScore >= @MinGravityScore
    GROUP BY dp.Category, dp.GravityScore
    ORDER BY InefficencyIndex DESC
END
```

## Power BI Integration

### Connecting to SQL Server

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. When prompted, enter connection parameters:

```
Server: ${SQL_SERVER_NAME}
Database: LogiFleetPulse
Authentication: Windows / SQL Server
```

3. Power BI automatically detects relationships based on foreign keys

### Key DAX Measures

```dax
// Fleet Utilization Rate
Fleet Utilization % = 
VAR TotalTripMinutes = SUM(FactFleetTrips[DurationMinutes])
VAR TotalIdleMinutes = SUM(FactFleetTrips[IdleTimeMinutes])
RETURN
DIVIDE(
    TotalTripMinutes - TotalIdleMinutes,
    TotalTripMinutes,
    0
) * 100

// Warehouse Gravity Efficiency
Gravity Zone Efficiency = 
VAR HighGravityAvgDwell = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        FactWarehouseOperations[GravityZone] = "High"
    )
VAR LowGravityAvgDwell = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        FactWarehouseOperations[GravityZone] = "Low"
    )
RETURN
IF(
    HighGravityAvgDwell < LowGravityAvgDwell,
    100, -- Efficient: high gravity items move faster
    (LowGravityAvgDwell / HighGravityAvgDwell) * 100
)

// Predictive Bottleneck Index
Bottleneck Risk Score = 
VAR CurrentDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR HistoricalP95 = 
    CALCULATE(
        PERCENTILE.INC(FactWarehouseOperations[DwellTimeMinutes], 0.95),
        DATESINPERIOD(DimTime[Date], MAX(DimTime[Date]), -90, DAY)
    )
VAR FleetCapacity = COUNTROWS(DimVehicle)
VAR TripsInProgress = 
    CALCULATE(
        COUNTROWS(FactFleetTrips),
        FactFleetTrips[TripStatus] = "InProgress"
    )
RETURN
(CurrentDwell / HistoricalP95) * (TripsInProgress / FleetCapacity) * 100
```

### Time Intelligence Patterns

```dax
// Rolling 7-Day Average Dwell Time
Dwell Time 7D MA = 
AVERAGEX(
    DATESINPERIOD(DimTime[Date], LASTDATE(DimTime[Date]), -7, DAY),
    CALCULATE(AVERAGE(FactWarehouseOperations[DwellTimeMinutes]))
)

// Year-over-Year Fleet Fuel Comparison
Fuel YoY Change % = 
VAR CurrentYearFuel = SUM(FactFleetTrips[FuelConsumedLiters])
VAR PreviousYearFuel = 
    CALCULATE(
        SUM(FactFleetTrips[FuelConsumedLiters]),
        SAMEPERIODLASTYEAR(DimTime[Date])
    )
RETURN
DIVIDE(
    CurrentYearFuel - PreviousYearFuel,
    PreviousYearFuel,
    0
) * 100
```

## Common Workflows

### Adding New Data Source

```sql
-- 1. Create staging table for external feed
CREATE TABLE Staging_FleetTelemetry (
    VehicleID NVARCHAR(50),
    Timestamp DATETIME2,
    Latitude DECIMAL(10,6),
    Longitude DECIMAL(10,6),
    Speed DECIMAL(5,2),
    FuelLevel DECIMAL(5,2),
    EngineTemp DECIMAL(5,2),
    LoadedAt DATETIME2 DEFAULT GETDATE()
)

-- 2. Create SSIS package or stored procedure to transform and load
CREATE PROCEDURE usp_LoadFleetTelemetry
AS
BEGIN
    INSERT INTO FactFleetTrips (TimeKey, VehicleKey, ...)
    SELECT 
        dt.TimeKey,
        dv.VehicleKey,
        ...
    FROM Staging_FleetTelemetry src
    INNER JOIN DimTime dt ON 
        DATEPART(YEAR, src.Timestamp) = dt.Year AND
        DATEPART(MONTH, src.Timestamp) = dt.Month AND
        DATEPART(DAY, src.Timestamp) = dt.DayOfMonth AND
        DATEPART(HOUR, src.Timestamp) = dt.Hour AND
        (DATEPART(MINUTE, src.Timestamp) / 15) * 15 = dt.Minute
    INNER JOIN DimVehicle dv ON src.VehicleID = dv.VehicleID
END

-- 3. Schedule via SQL Agent Job
EXEC msdb.dbo.sp_add_job @job_name = 'Load Fleet Telemetry'
EXEC msdb.dbo.sp_add_jobstep 
    @job_name = 'Load Fleet Telemetry',
    @step_name = 'Execute Load Procedure',
    @command = 'EXEC usp_LoadFleetTelemetry'
EXEC msdb.dbo.sp_add_schedule 
    @schedule_name = 'Every 15 Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15
```

### Implementing Row-Level Security

```sql
-- 1. Create security dimension
CREATE TABLE DimUserSecurity (
    UserKey INT IDENTITY(1,1) PRIMARY KEY,
    Username NVARCHAR(100) NOT NULL,
    EmailAddress NVARCHAR(200),
    Role NVARCHAR(50), -- Executive, Manager, Supervisor, Analyst
    AllowedRegions NVARCHAR(MAX), -- JSON array
    AllowedWarehouses NVARCHAR(MAX) -- JSON array
)

-- 2. Create security function
CREATE FUNCTION fn_SecurityFilter(@Username NVARCHAR(100))
RETURNS TABLE
AS
RETURN
(
    SELECT GeographyKey
    FROM DimGeography dg
    INNER JOIN DimUserSecurity dus ON 
        dus.Username = @Username AND
        (
            dus.Role = 'Executive' OR
            dg.Region IN (SELECT value FROM OPENJSON(dus.AllowedRegions)) OR
            dg.LocationID IN (SELECT value FROM OPENJSON(dus.AllowedWarehouses))
        )
)

-- 3. Apply to Power BI model (DAX)
-- In Power BI Desktop > Manage Roles > Create Role:
[GeographyKey] IN VALUES(fn_SecurityFilter(USERNAME()))
```

### Creating Alerting Logic

```sql
CREATE PROCEDURE usp_CheckKPIThresholds
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @Alerts TABLE (
        AlertType NVARCHAR(100),
        Severity NVARCHAR(20),
        Message NVARCHAR(500),
        AffectedEntity NVARCHAR(200),
        CurrentValue DECIMAL(10,2),
        ThresholdValue DECIMAL(10,2)
    )
    
    -- Check fleet idle time
    INSERT INTO @Alerts
    SELECT 
        'High Fleet Idle Time',
        'Warning',
        'Vehicle ' + dv.VehicleName + ' exceeded 15% idle time on route ' + 
            dgo.LocationName + ' to ' + dgd.LocationName,
        dv.VehicleName,
        (fft.IdleTimeMinutes * 100.0 / fft.DurationMinutes),
        15.0
    FROM FactFleetTrips fft
    INNER JOIN DimVehicle dv ON fft.VehicleKey = dv.VehicleKey
    INNER JOIN DimGeography dgo ON fft.OriginKey = dgo.GeographyKey
    INNER JOIN DimGeography dgd ON fft.DestinationKey = dgd.GeographyKey
    INNER JOIN DimTime dt ON fft.TimeKey = dt.TimeKey
    WHERE dt.Date = CAST(GETDATE() AS DATE)
      AND (fft.IdleTimeMinutes * 100.0 / fft.DurationMinutes) > 15
    
    -- Check warehouse dwell time for high gravity items
    INSERT INTO @Alerts
    SELECT 
        'Excessive Dwell Time - High Value',
        'Critical',
        'SKU ' + dp.SKU + ' (' + dp.ProductName + ') has dwell time of ' +
            CAST(AVG(fwo.DwellTimeMinutes) AS NVARCHAR(10)) + ' minutes',
        dp.SKU,
        AVG(fwo.DwellTimeMinutes),
        120.0
    FROM FactWarehouseOperations fwo
    INNER JOIN DimProduct dp ON fwo.ProductKey = dp.ProductKey
    INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    WHERE dt.Date >= CAST(DATEADD(DAY, -1, GETDATE()) AS DATE)
      AND dp.GravityScore >= 80
    GROUP BY dp.SKU, dp.ProductName
    HAVING AVG(fwo.DwellTimeMinutes) > 120
    
    -- Send alerts (integrate with email or Teams)
    IF EXISTS (SELECT 1 FROM @Alerts)
    BEGIN
        -- Log to alert table
        INSERT INTO AlertLog (AlertTime, AlertType, Severity, Message, AffectedEntity)
        SELECT GETDATE(), AlertType, Severity, Message, AffectedEntity
        FROM @Alerts
        
        -- Call external notification service
        -- EXEC msdb.dbo.sp_send_dbmail ...
    END
    
    SELECT * FROM @Alerts
END
```

## Troubleshooting

### Query Performance Issues

**Problem**: Cross-fact queries taking too long

**Solution**: Add covering indexes and partitioning

```sql
-- Partition fact tables by date
ALTER TABLE FactWarehouseOperations 
DROP CONSTRAINT PK_FactWarehouseOperations

CREATE PARTITION FUNCTION PF_TimeKey (INT)
AS RANGE RIGHT FOR VALUES (20260101, 20260201, 20260301, ...) -- Monthly partitions

CREATE PARTITION SCHEME PS_TimeKey
AS PARTITION PF_TimeKey ALL TO ([PRIMARY])

ALTER TABLE FactWarehouseOperations
ADD CONSTRAINT PK_FactWarehouseOperations 
PRIMARY KEY CLUSTERED (OperationKey, TimeKey)
ON PS_TimeKey(TimeKey)

-- Create columnstore index for analytics
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouse_Analytics
ON FactWarehouseOperations (
    TimeKey, ProductKey, WarehouseKey, GravityZone, 
    DwellTimeMinutes, Quantity
)
```

### Power BI Refresh Failures

**Problem**: Scheduled refresh fails with timeout

**Solution**: Use incremental refresh and query folding

```powerquery
// In Power BI Power Query Editor
let
    Source = Sql.Database("${SQL_SERVER_NAME}", "LogiFleetPulse"),
    // Use RangeStart and RangeEnd parameters for incremental refresh
    FilteredRows = Table.SelectRows(
        Source{[Schema="dbo",Item="FactWarehouseOperations"]}[Data],
        each [CreatedAt] >= RangeStart and [CreatedAt] < RangeEnd
    )
in
    FilteredRows
```

Configure incremental refresh in Power BI:
- Archive data older than 2 years
- Refresh last 90 days
- Detect data changes based on `CreatedAt`

### Gravity Score Calculation Anomalies

**Problem**: Products showing incorrect gravity zones

**Solution**: Validate data quality and recalculate

```sql
-- Identify products with missing metrics
SELECT 
    dp.SKU,
    dp.ProductName,
    dp.GravityScore,
    COUNT(fwo.OperationKey) AS RecentOperations,
    AVG(fwo.DwellTimeMinutes) AS AvgDwell
FROM DimProduct dp
LEFT JOIN FactWarehouseOperations fwo ON 
    dp.ProductKey = fwo.ProductKey AND
    fwo.TimeKey >= (SELECT MIN(TimeKey) FROM DimTime WHERE Date >= DATEADD(DAY, -90, GETDATE()))
WHERE dp.IsCurrent = 1
GROUP BY dp.SKU, dp.ProductName, dp.GravityScore
HAVING COUNT(fwo.OperationKey) = 0 OR dp.GravityScore IS NULL

-- Recalculate with defaults for new products
UPDATE dp
SET GravityScore = 50 -- Medium default
FROM DimProduct dp
WHERE dp.IsCurrent = 1 AND dp.GravityScore IS NULL
```

### Data Integration Errors

**Problem**: External API data not loading

**Solution**: Implement retry logic and error handling

```sql
CREATE PROCEDURE usp_LoadExternalWeatherData
    @RetryCount INT = 3
AS
BEGIN
    DECLARE @Attempt INT = 0
    DECLARE @Success BIT = 0
    
    WHILE @Attempt < @RetryCount AND @Success = 0
    BEGIN TRY
        -- Call external API via CLR or Python integration
        INSERT INTO Staging_WeatherData
        EXEC sp_invoke_external_rest_endpoint 
            @url = N'${WEATHER_API_ENDPOINT}',
            @method = N'GET',
            @headers = N'{"X-API-Key":"${WEATHER_API_KEY}"}'
        
        SET @Success = 1
    END TRY
    BEGIN CATCH
        SET @Attempt = @Attempt + 1
        WAITFOR DELAY '00:00:05' -- Wait 5 seconds before retry
        
        IF @Attempt >= @RetryCount
        BEGIN
            INSERT INTO ErrorLog (ErrorTime, ProcedureName, ErrorMessage)
            VALUES (GETDATE(), 'usp_LoadExternalWeatherData', ERROR_MESSAGE())
        END
    END CATCH
END
```

## Configuration Best Practices

### Environment Variables

Store connection strings and API keys in environment variables or Azure Key Vault:

```bash
# .env file (local development)
SQL_SERVER_NAME=localhost
SQL_DATABASE=LogiFleetPulse
SQL_USERNAME=logifleet_user
SQL_PASSWORD=${SQL_PASSWORD}
WMS_API_ENDPOINT=https://api.wms-system.com/v1
WMS_API_KEY=${WMS_API_KEY}
FLEET_API_ENDPOINT=https://api.fleet-telemetry.com/v2
FLEET_API_KEY=${FLEET_API_KEY}
WEATHER_API_KEY=${WEATHER_API_KEY}
```

### Maintenance Schedule

```sql
-- Weekly index maintenance
CREATE PROCEDURE usp_WeeklyMaintenance
AS
BEGIN
    -- Rebuild fragmented indexes
    DECLARE @SQL NVARCHAR(MAX)
    SELECT @SQL = STRING_AGG(
        'ALTER INDEX ' + i.name + ' ON ' + OBJECT_NAME(i.object_id) + ' REBUILD',
        '; '
    )
    FROM sys.indexes i
    INNER JOIN sys.dm_db_index_physical_stats(
        DB_ID(), NULL, NULL, NULL, 'LIMITED'
    ) s ON i.object_id = s.object_id AND i.index_id = s.index_id
    WHERE s.avg_fragmentation_in_percent > 30
    
    EXEC sp_executesql @SQL
    
    -- Update statistics
    EXEC sp_updatestats
    
    -- Archive old data (older than 2 years)
    DELETE FROM FactWarehouseOperations
    WHERE TimeKey IN (
        SELECT TimeKey FROM DimTime 
        WHERE Date 
