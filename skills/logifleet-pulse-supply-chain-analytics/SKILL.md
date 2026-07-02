---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehouse with multi-fact star schema for fleet, warehouse, and supply chain intelligence
triggers:
  - set up logifleet pulse analytics
  - configure supply chain data warehouse
  - implement logistics intelligence dashboard
  - query fleet and warehouse operations
  - build power bi logistics reports
  - deploy logicore analytics schema
  - create multi-fact star schema for logistics
  - analyze cross-modal supply chain data
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

LogiFleet Pulse is a comprehensive logistics intelligence platform built on MS SQL Server and Power BI. It implements a multi-fact star schema that unifies warehouse operations, fleet telemetry, inventory management, and external data feeds into a single semantic layer for real-time supply chain analytics.

## What It Does

- **Unified Data Warehouse**: Multi-fact star schema linking warehouse, fleet, and cross-dock operations
- **Time-Phased Dimensions**: 15-minute granularity time buckets with fiscal period support
- **Cross-Fact KPI Harmonization**: Query across warehouse and fleet metrics using shared dimensions
- **Predictive Analytics**: Bottleneck detection, maintenance prioritization, demand elasticity modeling
- **Warehouse Gravity Zones™**: Spatial optimization based on pick frequency, value, and fragility
- **Power BI Dashboards**: Pre-built templates with role-based access and natural language queries

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Appropriate permissions to create databases and tables

### Deploy the Database Schema

1. **Clone or download the repository**:
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Create the database**:
```sql
-- Execute in SSMS or Azure Data Studio
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO
```

3. **Deploy the schema** (execute the provided SQL scripts in order):
```sql
-- 01_CreateDimensions.sql
-- 02_CreateFacts.sql
-- 03_CreateViews.sql
-- 04_CreateStoredProcedures.sql
-- 05_CreateIndexes.sql
```

### Configure Data Sources

Create a `config.json` file with your data source connections:

```json
{
  "database": {
    "server": "${SQL_SERVER}",
    "database": "LogiFleetPulse",
    "username": "${SQL_USERNAME}",
    "password": "${SQL_PASSWORD}"
  },
  "dataSources": {
    "wms": {
      "type": "rest_api",
      "endpoint": "${WMS_API_ENDPOINT}",
      "apiKey": "${WMS_API_KEY}"
    },
    "telemetry": {
      "type": "mqtt",
      "broker": "${TELEMETRY_BROKER}",
      "topic": "fleet/+/telemetry"
    },
    "erp": {
      "type": "sql_server",
      "connectionString": "${ERP_CONNECTION_STRING}"
    }
  },
  "refresh": {
    "intervalMinutes": 15
  }
}
```

## Core Schema Structure

### Dimension Tables

**DimTime** - Time dimension with 15-minute granularity:
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    FullDateTime DATETIME NOT NULL,
    [Date] DATE NOT NULL,
    [Year] INT NOT NULL,
    [Quarter] INT NOT NULL,
    [Month] INT NOT NULL,
    [Day] INT NOT NULL,
    [Hour] INT NOT NULL,
    [Minute] INT NOT NULL,
    DayOfWeek INT NOT NULL,
    DayName NVARCHAR(20),
    IsWeekend BIT,
    FiscalYear INT,
    FiscalPeriod INT
);

CREATE UNIQUE INDEX UQ_DimTime_FullDateTime ON DimTime(FullDateTime);
```

**DimGeography** - Hierarchical location dimension:
```sql
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID NVARCHAR(50) NOT NULL,
    LocationName NVARCHAR(200) NOT NULL,
    LocationType NVARCHAR(50), -- 'Warehouse', 'Route Node', 'Distribution Center'
    Continent NVARCHAR(50),
    Country NVARCHAR(100),
    Region NVARCHAR(100),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    ValidFrom DATETIME NOT NULL,
    ValidTo DATETIME,
    IsCurrent BIT DEFAULT 1
);
```

**DimProduct** - Product hierarchy with gravity scoring:
```sql
CREATE TABLE DimProduct (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    ProductID NVARCHAR(50) NOT NULL,
    ProductName NVARCHAR(200) NOT NULL,
    SKU NVARCHAR(100),
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    ProductLine NVARCHAR(100),
    GravityScore DECIMAL(5,2), -- Composite score: velocity + value + fragility
    IsFragile BIT,
    RequiresRefrigeration BIT,
    StandardWeight DECIMAL(10,2),
    StandardVolume DECIMAL(10,2),
    ValidFrom DATETIME NOT NULL,
    ValidTo DATETIME,
    IsCurrent BIT DEFAULT 1
);
```

### Fact Tables

**FactWarehouseOperations** - Warehouse activity tracking:
```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProduct(ProductKey),
    OperationType NVARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OperationID NVARCHAR(100) UNIQUE NOT NULL,
    QuantityHandled INT,
    DwellTimeMinutes INT,
    StorageZone NVARCHAR(50),
    GravityZone NVARCHAR(50), -- 'High', 'Medium', 'Low'
    EmployeeID NVARCHAR(50),
    EquipmentID NVARCHAR(50),
    CycleTimeMinutes DECIMAL(10,2),
    ErrorCount INT DEFAULT 0,
    CreatedDateTime DATETIME DEFAULT GETDATE()
);

CREATE INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey);
CREATE INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey);
CREATE INDEX IX_FactWarehouse_Geography ON FactWarehouseOperations(GeographyKey);
```

**FactFleetTrips** - Fleet telemetry and route data:
```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    StartTimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    EndTimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    TripID NVARCHAR(100) UNIQUE NOT NULL,
    VehicleID NVARCHAR(50) NOT NULL,
    DriverID NVARCHAR(50),
    DistanceKm DECIMAL(10,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(10,2),
    AverageSpeed DECIMAL(5,2),
    MaxSpeed DECIMAL(5,2),
    LoadWeightKg DECIMAL(10,2),
    StopCount INT,
    DelayMinutes INT,
    DelayReason NVARCHAR(200),
    MaintenanceAlertFlag BIT DEFAULT 0,
    CreatedDateTime DATETIME DEFAULT GETDATE()
);

CREATE INDEX IX_FactFleet_StartTime ON FactFleetTrips(StartTimeKey);
CREATE INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleID);
```

**FactCrossDock** - Cross-docking operations:
```sql
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    InboundTimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    OutboundTimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProduct(ProductKey),
    CrossDockID NVARCHAR(100) UNIQUE NOT NULL,
    InboundTripID NVARCHAR(100),
    OutboundTripID NVARCHAR(100),
    QuantityTransferred INT,
    DwellMinutes INT,
    DirectTransfer BIT, -- No staging area
    CreatedDateTime DATETIME DEFAULT GETDATE()
);
```

## Key Stored Procedures

### Incremental Data Loading

**usp_LoadWarehouseOperations** - ETL for warehouse data:
```sql
CREATE PROCEDURE usp_LoadWarehouseOperations
    @StartDateTime DATETIME,
    @EndDateTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new operations from staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        OperationID, QuantityHandled, DwellTimeMinutes,
        StorageZone, GravityZone, EmployeeID, CycleTimeMinutes
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.OperationType,
        s.OperationID,
        s.Quantity,
        DATEDIFF(MINUTE, s.StartTime, s.EndTime) AS DwellTimeMinutes,
        s.Zone,
        CASE 
            WHEN p.GravityScore >= 80 THEN 'High'
            WHEN p.GravityScore >= 50 THEN 'Medium'
            ELSE 'Low'
        END AS GravityZone,
        s.EmployeeID,
        DATEDIFF(MINUTE, s.StartTime, s.EndTime) AS CycleTimeMinutes
    FROM StagingWarehouseOps s
    INNER JOIN DimTime t ON CAST(s.StartTime AS DATETIME) = t.FullDateTime
    INNER JOIN DimGeography g ON s.LocationID = g.LocationID AND g.IsCurrent = 1
    INNER JOIN DimProduct p ON s.SKU = p.SKU AND p.IsCurrent = 1
    WHERE s.StartTime >= @StartDateTime 
        AND s.StartTime < @EndDateTime
        AND NOT EXISTS (
            SELECT 1 FROM FactWarehouseOperations f 
            WHERE f.OperationID = s.OperationID
        );
    
    SELECT @@ROWCOUNT AS RowsInserted;
END;
```

### Cross-Fact Analysis Views

**vw_WarehouseFleetCrossAnalysis** - Unified view linking warehouse and fleet:
```sql
CREATE VIEW vw_WarehouseFleetCrossAnalysis AS
SELECT 
    t.[Date],
    t.[Year],
    t.[Month],
    g.LocationName,
    g.Region,
    p.Category,
    p.ProductLine,
    p.GravityScore,
    -- Warehouse metrics
    SUM(CASE WHEN w.OperationType = 'Picking' THEN w.QuantityHandled ELSE 0 END) AS TotalPicked,
    AVG(CASE WHEN w.OperationType = 'Picking' THEN w.CycleTimeMinutes ELSE NULL END) AS AvgPickTime,
    AVG(w.DwellTimeMinutes) AS AvgDwellTime,
    -- Fleet metrics
    COUNT(DISTINCT f.TripID) AS TripCount,
    SUM(f.DistanceKm) AS TotalDistanceKm,
    SUM(f.FuelConsumedLiters) AS TotalFuelLiters,
    AVG(f.IdleTimeMinutes) AS AvgIdleMinutes,
    SUM(f.DelayMinutes) AS TotalDelayMinutes,
    -- Cross-fact KPIs
    CASE 
        WHEN SUM(f.DistanceKm) > 0 
        THEN SUM(f.FuelConsumedLiters) / SUM(f.DistanceKm) 
        ELSE NULL 
    END AS FuelEfficiencyLitersPerKm,
    CASE 
        WHEN COUNT(DISTINCT f.TripID) > 0 
        THEN CAST(SUM(CASE WHEN w.OperationType = 'Shipping' THEN w.QuantityHandled ELSE 0 END) AS FLOAT) / COUNT(DISTINCT f.TripID)
        ELSE NULL
    END AS AvgUnitsPerTrip
FROM FactWarehouseOperations w
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
INNER JOIN DimProduct p ON w.ProductKey = p.ProductKey
LEFT JOIN FactFleetTrips f ON f.OriginGeographyKey = g.GeographyKey 
    AND f.StartTimeKey = t.TimeKey
GROUP BY 
    t.[Date], t.[Year], t.[Month],
    g.LocationName, g.Region,
    p.Category, p.ProductLine, p.GravityScore;
```

## Common Query Patterns

### Calculate Warehouse Gravity Zone Performance

```sql
-- Compare cycle times across gravity zones
SELECT 
    w.GravityZone,
    COUNT(*) AS OperationCount,
    AVG(w.CycleTimeMinutes) AS AvgCycleTime,
    AVG(w.DwellTimeMinutes) AS AvgDwellTime,
    SUM(w.ErrorCount) AS TotalErrors,
    AVG(p.GravityScore) AS AvgGravityScore
FROM FactWarehouseOperations w
INNER JOIN DimProduct p ON w.ProductKey = p.ProductKey
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
WHERE t.[Date] >= DATEADD(MONTH, -1, GETDATE())
    AND w.OperationType = 'Picking'
GROUP BY w.GravityZone
ORDER BY AvgGravityScore DESC;
```

### Fleet Idle Time Analysis by Route

```sql
-- Identify routes with excessive idle time
SELECT 
    og.LocationName AS Origin,
    dg.LocationName AS Destination,
    COUNT(*) AS TripCount,
    AVG(f.DurationMinutes) AS AvgTripDuration,
    AVG(f.IdleTimeMinutes) AS AvgIdleTime,
    AVG(f.IdleTimeMinutes * 100.0 / NULLIF(f.DurationMinutes, 0)) AS IdleTimePercentage,
    AVG(f.FuelConsumedLiters) AS AvgFuelConsumption
FROM FactFleetTrips f
INNER JOIN DimGeography og ON f.OriginGeographyKey = og.GeographyKey
INNER JOIN DimGeography dg ON f.DestinationGeographyKey = dg.GeographyKey
INNER JOIN DimTime t ON f.StartTimeKey = t.TimeKey
WHERE t.[Date] >= DATEADD(DAY, -30, GETDATE())
GROUP BY og.LocationName, dg.LocationName
HAVING AVG(f.IdleTimeMinutes * 100.0 / NULLIF(f.DurationMinutes, 0)) > 15
ORDER BY IdleTimePercentage DESC;
```

### Cross-Dock Efficiency Tracking

```sql
-- Analyze cross-dock dwell times by product category
SELECT 
    p.Category,
    p.SubCategory,
    COUNT(*) AS CrossDockOperations,
    AVG(cd.DwellMinutes) AS AvgDwellMinutes,
    SUM(CASE WHEN cd.DirectTransfer = 1 THEN 1 ELSE 0 END) AS DirectTransferCount,
    AVG(CASE WHEN cd.DirectTransfer = 1 THEN cd.DwellMinutes ELSE NULL END) AS AvgDirectDwell,
    AVG(CASE WHEN cd.DirectTransfer = 0 THEN cd.DwellMinutes ELSE NULL END) AS AvgStagedDwell
FROM FactCrossDock cd
INNER JOIN DimProduct p ON cd.ProductKey = p.ProductKey
INNER JOIN DimTime t ON cd.InboundTimeKey = t.TimeKey
WHERE t.[Date] >= DATEADD(WEEK, -1, GETDATE())
GROUP BY p.Category, p.SubCategory
ORDER BY AvgDwellMinutes DESC;
```

### Predictive Bottleneck Detection

```sql
-- Identify products with increasing dwell times (early warning)
WITH DwellTrends AS (
    SELECT 
        p.ProductID,
        p.ProductName,
        p.GravityZone,
        t.[Year],
        t.[Month],
        AVG(w.DwellTimeMinutes) AS AvgDwellTime
    FROM FactWarehouseOperations w
    INNER JOIN DimProduct p ON w.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.[Date] >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY p.ProductID, p.ProductName, p.GravityZone, t.[Year], t.[Month]
),
DwellComparison AS (
    SELECT 
        ProductID,
        ProductName,
        GravityZone,
        LAG(AvgDwellTime, 1) OVER (PARTITION BY ProductID ORDER BY [Year], [Month]) AS PriorMonthDwell,
        AvgDwellTime AS CurrentMonthDwell
    FROM DwellTrends
)
SELECT 
    ProductID,
    ProductName,
    GravityZone,
    PriorMonthDwell,
    CurrentMonthDwell,
    ((CurrentMonthDwell - PriorMonthDwell) * 100.0 / NULLIF(PriorMonthDwell, 0)) AS DwellIncreasePercentage
FROM DwellComparison
WHERE PriorMonthDwell IS NOT NULL
    AND CurrentMonthDwell > PriorMonthDwell * 1.2 -- 20% increase threshold
ORDER BY DwellIncreasePercentage DESC;
```

## Power BI Integration

### Connect Power BI to SQL Server

1. Open `LogiFleet_Pulse_Master.pbit` template
2. Enter your SQL Server connection details:
   - Server: `${SQL_SERVER}`
   - Database: `LogiFleetPulse`
   - Authentication: Windows or SQL Server

### Key DAX Measures

**Warehouse Velocity Score**:
```dax
Warehouse Velocity = 
VAR TotalOperations = COUNTROWS(FactWarehouseOperations)
VAR AvgCycleTime = AVERAGE(FactWarehouseOperations[CycleTimeMinutes])
VAR ErrorRate = DIVIDE(SUM(FactWarehouseOperations[ErrorCount]), TotalOperations, 0)
RETURN
    DIVIDE(TotalOperations, AvgCycleTime, 0) * (1 - ErrorRate)
```

**Fleet Efficiency Index**:
```dax
Fleet Efficiency Index = 
VAR TotalDistance = SUM(FactFleetTrips[DistanceKm])
VAR TotalFuel = SUM(FactFleetTrips[FuelConsumedLiters])
VAR TotalIdle = SUM(FactFleetTrips[IdleTimeMinutes])
VAR TotalDuration = SUM(FactFleetTrips[DurationMinutes])
VAR IdlePercentage = DIVIDE(TotalIdle, TotalDuration, 0)
RETURN
    (DIVIDE(TotalDistance, TotalFuel, 0)) * (1 - IdlePercentage) * 100
```

**Cross-Fact Sync Score** (measures warehouse-fleet alignment):
```dax
Sync Score = 
VAR ShippedUnits = 
    CALCULATE(
        SUM(FactWarehouseOperations[QuantityHandled]),
        FactWarehouseOperations[OperationType] = "Shipping"
    )
VAR TripCapacity = 
    SUMX(
        FactFleetTrips,
        FactFleetTrips[LoadWeightKg]
    )
VAR UtilizationRate = DIVIDE(ShippedUnits, TripCapacity, 0)
RETURN
    MIN(UtilizationRate * 100, 100)
```

### Row-Level Security (RLS)

Create roles for different user levels:

```dax
-- Regional Manager Role
[Region] = USERPRINCIPALNAME()

-- Warehouse Supervisor Role
[LocationName] IN { 
    LOOKUPVALUE(
        UserWarehouseMapping[WarehouseName],
        UserWarehouseMapping[UserEmail], 
        USERPRINCIPALNAME()
    )
}

-- Executive Role (no filters - see all data)
1 = 1
```

## Automated Alerting

**usp_CheckKPIThresholds** - Trigger alerts on threshold breaches:
```sql
CREATE PROCEDURE usp_CheckKPIThresholds
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertTable TABLE (
        AlertType NVARCHAR(100),
        Location NVARCHAR(200),
        MetricValue DECIMAL(10,2),
        ThresholdValue DECIMAL(10,2),
        Message NVARCHAR(500)
    );
    
    -- Check for excessive fleet idle time
    INSERT INTO @AlertTable
    SELECT 
        'Fleet Idle Time Breach',
        g.LocationName,
        AVG(f.IdleTimeMinutes * 100.0 / NULLIF(f.DurationMinutes, 0)) AS IdlePercentage,
        15.0 AS Threshold,
        'Fleet idle time exceeds 15% threshold for route: ' + g.LocationName
    FROM FactFleetTrips f
    INNER JOIN DimGeography g ON f.OriginGeographyKey = g.GeographyKey
    INNER JOIN DimTime t ON f.StartTimeKey = t.TimeKey
    WHERE t.[Date] = CAST(GETDATE() AS DATE)
    GROUP BY g.LocationName
    HAVING AVG(f.IdleTimeMinutes * 100.0 / NULLIF(f.DurationMinutes, 0)) > 15;
    
    -- Check for warehouse dwell time increases
    INSERT INTO @AlertTable
    SELECT 
        'Warehouse Dwell Time Spike',
        g.LocationName,
        AVG(w.DwellTimeMinutes) AS CurrentDwell,
        72.0 AS Threshold,
        'Warehouse dwell time exceeds 72 hours at: ' + g.LocationName
    FROM FactWarehouseOperations w
    INNER JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.[Date] >= DATEADD(DAY, -1, GETDATE())
    GROUP BY g.LocationName
    HAVING AVG(w.DwellTimeMinutes) > 4320; -- 72 hours in minutes
    
    -- Check for maintenance alerts
    INSERT INTO @AlertTable
    SELECT 
        'Fleet Maintenance Required',
        f.VehicleID,
        1.0,
        1.0,
        'Vehicle ' + f.VehicleID + ' has triggered maintenance alert'
    FROM FactFleetTrips f
    INNER JOIN DimTime t ON f.StartTimeKey = t.TimeKey
    WHERE t.[Date] = CAST(GETDATE() AS DATE)
        AND f.MaintenanceAlertFlag = 1;
    
    -- Return all alerts
    SELECT * FROM @AlertTable;
    
    -- Log alerts to audit table
    INSERT INTO AlertLog (AlertDateTime, AlertType, Location, MetricValue, ThresholdValue, Message)
    SELECT GETDATE(), AlertType, Location, MetricValue, ThresholdValue, Message
    FROM @AlertTable;
END;
```

Schedule this procedure via SQL Server Agent:
```sql
-- Run every 15 minutes during business hours
EXEC msdb.dbo.sp_add_job @job_name = 'LogiFleet_KPI_Monitoring';

EXEC msdb.dbo.sp_add_jobstep 
    @job_name = 'LogiFleet_KPI_Monitoring',
    @step_name = 'Check Thresholds',
    @command = 'EXEC usp_CheckKPIThresholds',
    @database_name = 'LogiFleetPulse';

EXEC msdb.dbo.sp_add_schedule 
    @schedule_name = 'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15,
    @active_start_time = 060000, -- 6 AM
    @active_end_time = 220000; -- 10 PM
```

## Troubleshooting

### Issue: Slow Cross-Fact Queries

**Solution**: Ensure columnstore indexes on fact tables:
```sql
-- Create columnstore indexes for analytical queries
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouse
ON FactWarehouseOperations (
    TimeKey, GeographyKey, ProductKey, OperationType,
    QuantityHandled, DwellTimeMinutes, CycleTimeMinutes
);

CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactFleet
ON FactFleetTrips (
    StartTimeKey, OriginGeographyKey, DestinationGeographyKey,
    VehicleID, DistanceKm, FuelConsumedLiters, IdleTimeMinutes
);
```

### Issue: Power BI Refresh Failures

**Solution**: Check DirectQuery vs Import mode settings. For large datasets:
```sql
-- Create aggregated views for Power BI import mode
CREATE VIEW vw_WarehouseDailySummary AS
SELECT 
    t.[Date],
    g.GeographyKey,
    g.LocationName,
    p.Category,
    w.OperationType,
    COUNT(*) AS OperationCount,
    SUM(w.QuantityHandled) AS TotalQuantity,
    AVG(w.CycleTimeMinutes) AS AvgCycleTime,
    AVG(w.DwellTimeMinutes) AS AvgDwellTime
FROM FactWarehouseOperations w
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
INNER JOIN DimProduct p ON w.ProductKey = p.ProductKey
GROUP BY t.[Date], g.GeographyKey, g.LocationName, p.Category, w.OperationType;
```

### Issue: Data Quality - Missing Dimension Lookups

**Solution**: Implement default dimension handling:
```sql
-- Insert default "Unknown" records in dimension tables
INSERT INTO DimGeography (LocationID, LocationName, LocationType, Continent, Country, Region, ValidFrom, IsCurrent)
VALUES ('UNKNOWN', 'Unknown Location', 'Unknown', 'Unknown', 'Unknown', 'Unknown', '1900-01-01', 1);

INSERT INTO DimProduct (ProductID, ProductName, SKU, Category, GravityScore, ValidFrom, IsCurrent)
VALUES ('UNKNOWN', 'Unknown Product', 'UNK-000', 'Unknown', 0, '1900-01-01', 1);

-- Use ISNULL in ETL procedures
SELECT 
    ISNULL(g.GeographyKey, (SELECT GeographyKey FROM DimGeography WHERE LocationID = 'UNKNOWN'))
FROM StagingData s
LEFT JOIN DimGeography g ON s.LocationID = g.LocationID AND g.IsCurrent = 1;
```

### Issue: Time Dimension Missing Records

**Solution**: Populate time dimension programmatically:
```sql
-- Generate time dimension for next 5 years at 15-minute intervals
DECLARE @StartDate DATETIME = '2025-01-01 00:00:00';
DECLARE @EndDate DATETIME = '2030-12-31 23:45:00';
DECLARE @CurrentDate DATETIME = @StartDate;

WHILE @CurrentDate <= @EndDate
BEGIN
    INSERT INTO DimTime (
        FullDateTime, [Date], [Year], [Quarter], [Month], [Day],
        [Hour], [Minute], DayOfWeek, DayName, IsWeekend,
        FiscalYear, FiscalPeriod
    )
    VALUES (
        @CurrentDate,
        CAST(@CurrentDate AS DATE),
        YEAR(@CurrentDate),
        DATEPART(QUARTER, @CurrentDate),
        MONTH(@CurrentDate),
        DAY(@CurrentDate),
        DATEPART(HOUR, @CurrentDate),
        DATEPART(MINUTE, @CurrentDate),
        DATEPART(WEEKDAY, @CurrentDate),
        DATENAME(WEEKDAY, @CurrentDate),
        CASE WHEN DATEPART(WEEKDAY, @CurrentDate) IN (1, 7) THEN 1 ELSE 0 END,
        CASE WHEN MONTH(@CurrentDate) >= 7 THEN YEAR(@CurrentDate) + 1 ELSE YEAR(@CurrentDate) END,
        CASE WHEN MONTH(@CurrentDate) >= 7 THEN MONTH(@CurrentDate) - 6 ELSE MONTH(@CurrentDate) + 6 END
    );
    
    SET @CurrentDate = DATEADD(MINUTE, 15, @CurrentDate);
END;
```

## Environment Variables Reference

Required environment variables for deployment:

- `SQL_SERVER`: SQL Server instance name or IP
- `SQL_USERNAME`: Database login username
- `SQL_PASSWORD`: Database login password
- `WMS_API_ENDPOINT`: Warehouse Management System API URL
- `WMS_API_KEY`: API key for WMS authentication
- `TELEMETRY_BROKER`: MQTT broker for fleet telemetry
- `ERP_CONNECTION_STRING`: Connection string to ERP system
- `POWERBI_WORKSPACE_ID`: Power BI workspace for report deployment
- `ALERT_EMAIL_SMTP`: SMTP server for alert emails
- `ALERT_EMAIL_FROM`: Sender email for alerts

## Best Practices

1. **Partition large fact tables** by date for better query performance
