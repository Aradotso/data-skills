---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehouse with multi-fact star schema for warehouse, fleet, and supply chain analytics
triggers:
  - set up logifleet pulse analytics warehouse
  - deploy supply chain data model with power bi
  - create logistics star schema in sql server
  - configure warehouse and fleet analytics dashboard
  - implement cross-fact kpi reporting for logistics
  - build real-time supply chain intelligence system
  - integrate warehouse operations with fleet telemetry data
  - design multi-modal logistics data warehouse
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive data warehousing and business intelligence solution for logistics and supply chain management. It provides:

- **Multi-fact star schema** linking warehouse operations, fleet trips, and cross-dock activities
- **Time-phased dimensions** for 15-minute granularity operational analytics
- **Power BI dashboards** for real-time logistics KPI visualization
- **Cross-fact harmonization** to correlate warehouse metrics with fleet performance
- **Predictive analytics** for bottleneck detection and maintenance prioritization
- **Role-based access control** with row-level security

The platform integrates data from WMS, TMS, GPS/telematics, supplier portals, and external APIs into a unified semantic layer.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Developer, Standard, or Enterprise edition)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to source systems (WMS, fleet telemetry, ERP)

### Database Schema Deployment

1. **Clone the repository:**

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**

```sql
-- Connect to your SQL Server instance and create the database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Execute the schema creation script
-- Assumes script is in /sql/schema/create_tables.sql
:r .\sql\schema\create_tables.sql
GO

-- Create indexes and partitions
:r .\sql\schema\create_indexes.sql
GO

-- Deploy stored procedures for ETL
:r .\sql\etl\stored_procedures.sql
GO
```

3. **Configure data connections:**

Update the configuration file with your connection strings:

```json
{
  "connections": {
    "wms": {
      "server": "${WMS_SQL_SERVER}",
      "database": "${WMS_DATABASE}",
      "authentication": "integrated"
    },
    "telematics": {
      "api_endpoint": "${FLEET_API_URL}",
      "api_key": "${FLEET_API_KEY}"
    },
    "erp": {
      "server": "${ERP_SQL_SERVER}",
      "database": "${ERP_DATABASE}",
      "username": "${ERP_USERNAME}",
      "password": "${ERP_PASSWORD}"
    }
  },
  "refresh_interval_minutes": 15
}
```

## Core Data Model

### Fact Tables

```sql
-- FactWarehouseOperations: Core warehouse activity metrics
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    OperationTypeKey INT NOT NULL,
    EmployeeKey INT,
    
    -- Measures
    QuantityHandled INT,
    DwellTimeMinutes DECIMAL(10,2),
    PickingTimeSeconds INT,
    PackingTimeSeconds INT,
    TravelDistanceMeters DECIMAL(10,2),
    
    -- Foreign Keys
    CONSTRAINT FK_WO_Time FOREIGN KEY (TimeKey) 
        REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WO_Product FOREIGN KEY (ProductKey) 
        REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_WO_Warehouse FOREIGN KEY (WarehouseKey) 
        REFERENCES DimWarehouse(WarehouseKey)
)
GO

-- Create partitioning by month for performance
CREATE PARTITION FUNCTION PF_WarehouseOps_Month (INT)
AS RANGE RIGHT FOR VALUES (20260101, 20260201, 20260301, 20260401)
GO

-- FactFleetTrips: Vehicle and route performance
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    StartTimeKey INT NOT NULL,
    EndTimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT,
    
    -- Measures
    TotalDistanceKm DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    TotalStops INT,
    OnTimeDeliveryFlag BIT,
    
    CONSTRAINT FK_FT_Vehicle FOREIGN KEY (VehicleKey) 
        REFERENCES DimVehicle(VehicleKey)
)
GO

-- FactCrossDock: Fast-track transfers
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    ArrivalTimeKey INT NOT NULL,
    DepartureTimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OriginWarehouseKey INT,
    DestinationWarehouseKey INT,
    
    -- Measures
    QuantityTransferred INT,
    TransferTimeMinutes INT,
    TemperatureCompliantFlag BIT
)
GO
```

### Dimension Tables

```sql
-- DimTime: 15-minute granularity time dimension
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    
    -- Date attributes
    DateKey INT,
    CalendarYear INT,
    CalendarQuarter INT,
    CalendarMonth INT,
    CalendarDay INT,
    FiscalYear INT,
    FiscalQuarter INT,
    DayOfWeek INT,
    DayName VARCHAR(20),
    IsWeekend BIT,
    IsHoliday BIT,
    
    -- Time attributes
    Hour INT,
    Minute INT,
    QuarterHour INT, -- 1-4 within the hour
    TimeSlot VARCHAR(20), -- e.g., "Morning", "Afternoon"
    ShiftNumber INT -- 1=Morning, 2=Afternoon, 3=Night
)
GO

-- DimProductGravity: Product classification with velocity scoring
CREATE TABLE DimProduct (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    ProductSKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName VARCHAR(200),
    
    -- Hierarchy
    CategoryLevel1 VARCHAR(100),
    CategoryLevel2 VARCHAR(100),
    CategoryLevel3 VARCHAR(100),
    
    -- Attributes
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    IsFragile BIT,
    RequiresRefrigeration BIT,
    
    -- Gravity Zone attributes
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value / volume
    OptimalZoneType VARCHAR(50),
    AveragePickFrequency DECIMAL(10,2),
    
    -- SCD Type 2 fields
    EffectiveDate DATE,
    ExpirationDate DATE,
    IsCurrent BIT
)
GO

-- DimWarehouse: Warehouse and zone configuration
CREATE TABLE DimWarehouse (
    WarehouseKey INT IDENTITY(1,1) PRIMARY KEY,
    WarehouseCode VARCHAR(20) UNIQUE NOT NULL,
    WarehouseName VARCHAR(100),
    
    -- Location hierarchy
    GeographyKey INT,
    Address VARCHAR(200),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    
    -- Capacity
    TotalCapacityM3 DECIMAL(12,2),
    CurrentUtilizationPct DECIMAL(5,2),
    
    -- Zone information
    ZoneType VARCHAR(50), -- 'Receiving', 'Storage', 'Picking', 'Shipping'
    GravityZoneLevel INT, -- 1=Highest priority, 5=Lowest
    
    CONSTRAINT FK_Warehouse_Geo FOREIGN KEY (GeographyKey) 
        REFERENCES DimGeography(GeographyKey)
)
GO
```

## ETL Stored Procedures

### Incremental Load Pattern

```sql
-- Stored procedure for incremental warehouse operations load
CREATE PROCEDURE usp_LoadFactWarehouseOperations
    @LoadDate DATE = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @LastLoadDate DATETIME2;
    
    -- Get last successful load timestamp
    SELECT @LastLoadDate = MAX(LoadTimestamp)
    FROM ETL.LoadLog
    WHERE TableName = 'FactWarehouseOperations'
      AND Status = 'Success';
    
    -- Default to yesterday if no history
    IF @LastLoadDate IS NULL
        SET @LastLoadDate = DATEADD(DAY, -1, GETDATE());
    
    BEGIN TRY
        BEGIN TRANSACTION;
        
        -- Insert new records from staging
        INSERT INTO FactWarehouseOperations (
            TimeKey, ProductKey, WarehouseKey, OperationTypeKey,
            QuantityHandled, DwellTimeMinutes, PickingTimeSeconds
        )
        SELECT 
            dt.TimeKey,
            dp.ProductKey,
            dw.WarehouseKey,
            dot.OperationTypeKey,
            stg.Quantity,
            DATEDIFF(MINUTE, stg.StartTime, stg.EndTime) AS DwellTime,
            stg.ProcessingSeconds
        FROM Staging.WarehouseOperations stg
        INNER JOIN DimTime dt 
            ON dt.FullDateTime = DATEADD(MINUTE, 
                (DATEPART(MINUTE, stg.StartTime) / 15) * 15,
                DATEADD(HOUR, DATEPART(HOUR, stg.StartTime), 
                    CAST(CAST(stg.StartTime AS DATE) AS DATETIME2)))
        INNER JOIN DimProduct dp 
            ON dp.ProductSKU = stg.SKU 
            AND dp.IsCurrent = 1
        INNER JOIN DimWarehouse dw 
            ON dw.WarehouseCode = stg.WarehouseCode
        INNER JOIN DimOperationType dot 
            ON dot.OperationTypeName = stg.OperationType
        WHERE stg.StartTime > @LastLoadDate
          AND stg.LoadedFlag = 0;
        
        -- Mark staging records as loaded
        UPDATE Staging.WarehouseOperations
        SET LoadedFlag = 1
        WHERE StartTime > @LastLoadDate;
        
        -- Log success
        INSERT INTO ETL.LoadLog (TableName, LoadTimestamp, RowsLoaded, Status)
        VALUES ('FactWarehouseOperations', GETDATE(), @@ROWCOUNT, 'Success');
        
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        
        -- Log error
        INSERT INTO ETL.LoadLog (TableName, LoadTimestamp, Status, ErrorMessage)
        VALUES ('FactWarehouseOperations', GETDATE(), 'Failed', ERROR_MESSAGE());
        
        THROW;
    END CATCH
END
GO
```

### Cross-Fact KPI Calculation

```sql
-- Calculate composite KPI: Fleet efficiency relative to warehouse dwell time
CREATE PROCEDURE usp_CalculateCrossDockEfficiency
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    WITH WarehouseDwell AS (
        SELECT 
            dw.WarehouseKey,
            dw.WarehouseName,
            dt.CalendarMonth,
            AVG(fwo.DwellTimeMinutes) AS AvgDwellTime,
            SUM(fwo.QuantityHandled) AS TotalQuantity
        FROM FactWarehouseOperations fwo
        INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
        INNER JOIN DimWarehouse dw ON fwo.WarehouseKey = dw.WarehouseKey
        WHERE dt.DateKey BETWEEN CONVERT(INT, FORMAT(@StartDate, 'yyyyMMdd'))
                            AND CONVERT(INT, FORMAT(@EndDate, 'yyyyMMdd'))
        GROUP BY dw.WarehouseKey, dw.WarehouseName, dt.CalendarMonth
    ),
    FleetPerformance AS (
        SELECT 
            dw.WarehouseKey,
            dt.CalendarMonth,
            AVG(fft.IdleTimeMinutes) AS AvgIdleTime,
            SUM(fft.TotalDistanceKm) AS TotalDistance,
            SUM(fft.FuelConsumedLiters) AS TotalFuel
        FROM FactFleetTrips fft
        INNER JOIN DimTime dt ON fft.StartTimeKey = dt.TimeKey
        INNER JOIN DimRoute dr ON fft.RouteKey = dr.RouteKey
        INNER JOIN DimWarehouse dw ON dr.OriginWarehouseKey = dw.WarehouseKey
        WHERE dt.DateKey BETWEEN CONVERT(INT, FORMAT(@StartDate, 'yyyyMMdd'))
                            AND CONVERT(INT, FORMAT(@EndDate, 'yyyyMMdd'))
        GROUP BY dw.WarehouseKey, dt.CalendarMonth
    )
    SELECT 
        wd.WarehouseName,
        wd.CalendarMonth,
        wd.AvgDwellTime,
        wd.TotalQuantity,
        fp.AvgIdleTime,
        fp.TotalDistance,
        fp.TotalFuel,
        -- Composite efficiency score
        (wd.TotalQuantity / NULLIF(wd.AvgDwellTime, 0)) 
            * (fp.TotalDistance / NULLIF(fp.AvgIdleTime, 0)) 
            AS EfficiencyScore
    FROM WarehouseDwell wd
    INNER JOIN FleetPerformance fp 
        ON wd.WarehouseKey = fp.WarehouseKey
        AND wd.CalendarMonth = fp.CalendarMonth
    ORDER BY EfficiencyScore DESC;
END
GO
```

## Power BI Integration

### Connect to SQL Server

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Connection settings:

```
Server: ${SQL_SERVER_NAME}
Database: LogiFleetPulse
Data Connectivity mode: DirectQuery (for real-time) or Import (for better performance)
Advanced options:
  - Command timeout: 0
  - SQL statement: (leave blank to import all tables)
```

### DAX Measures for Cross-Fact Analytics

```dax
// Warehouse Gravity Score (calculated measure)
Warehouse_GravityScore = 
CALCULATE(
    AVERAGE(DimProduct[GravityScore]),
    FILTER(
        FactWarehouseOperations,
        FactWarehouseOperations[QuantityHandled] > 0
    )
)

// Fleet Utilization Rate
Fleet_UtilizationRate = 
DIVIDE(
    SUM(FactFleetTrips[TotalDistanceKm]) - SUM(FactFleetTrips[IdleTimeMinutes]) / 60,
    SUM(FactFleetTrips[TotalDistanceKm]),
    0
)

// Cross-Dock Transfer Efficiency
CrossDock_Efficiency = 
VAR TotalTransfers = COUNTROWS(FactCrossDock)
VAR FastTransfers = 
    CALCULATE(
        COUNTROWS(FactCrossDock),
        FactCrossDock[TransferTimeMinutes] <= 30
    )
RETURN
    DIVIDE(FastTransfers, TotalTransfers, 0)

// Time-Phased Bottleneck Index (predictive)
Bottleneck_Index = 
VAR CurrentDwell = [Warehouse_AvgDwellTime]
VAR HistoricalAvg = 
    CALCULATE(
        [Warehouse_AvgDwellTime],
        DATEADD(DimTime[FullDateTime], -30, DAY)
    )
VAR FleetStrain = 
    DIVIDE(
        [Fleet_AvgIdleTime],
        [Fleet_TotalTripTime],
        0
    )
RETURN
    IF(
        CurrentDwell > HistoricalAvg * 1.2 && FleetStrain > 0.15,
        1, // High bottleneck risk
        0  // Normal operations
    )

// Supplier Reliability Impact on Operations
Supplier_ImpactScore = 
SUMX(
    FILTER(
        FactWarehouseOperations,
        RELATED(DimSupplier[DefectRate]) > 0.05
    ),
    FactWarehouseOperations[DwellTimeMinutes] * RELATED(DimSupplier[DefectRate])
)
```

### Row-Level Security Configuration

```dax
// Create role in Power BI: "Regional Manager"
// Table: DimGeography
// Filter: [Region] = USERPRINCIPALNAME()

// Or for more complex logic:
[Region] IN (
    LOOKUPVALUE(
        UserRegionMapping[Region],
        UserRegionMapping[UserEmail],
        USERPRINCIPALNAME()
    )
)
```

## Common Usage Patterns

### Pattern 1: Daily KPI Refresh

```sql
-- SQL Server Agent job or Azure Data Factory pipeline
EXEC usp_LoadFactWarehouseOperations;
EXEC usp_LoadFactFleetTrips;
EXEC usp_UpdateDimensionSCD;
EXEC usp_RefreshAggregatedMetrics;

-- Trigger Power BI dataset refresh via API
-- (Called from orchestration tool)
```

### Pattern 2: Real-Time Alerting

```sql
-- Stored procedure to detect anomalies
CREATE PROCEDURE usp_DetectOperationalAnomalies
AS
BEGIN
    -- Check for excessive dwell time
    INSERT INTO Alerts.OperationalAlerts (AlertType, Severity, Message)
    SELECT 
        'HighDwellTime',
        'Critical',
        'SKU ' + dp.ProductSKU + ' in ' + dw.WarehouseName + 
        ' has dwell time of ' + CAST(AVG(fwo.DwellTimeMinutes) AS VARCHAR) + ' minutes'
    FROM FactWarehouseOperations fwo
    INNER JOIN DimProduct dp ON fwo.ProductKey = dp.ProductKey
    INNER JOIN DimWarehouse dw ON fwo.WarehouseKey = dw.WarehouseKey
    INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    WHERE dt.FullDateTime >= DATEADD(HOUR, -2, GETDATE())
    GROUP BY dp.ProductSKU, dw.WarehouseName
    HAVING AVG(fwo.DwellTimeMinutes) > (
        SELECT AVG(DwellTimeMinutes) * 1.5 
        FROM FactWarehouseOperations
        WHERE TimeKey IN (
            SELECT TimeKey 
            FROM DimTime 
            WHERE FullDateTime >= DATEADD(DAY, -7, GETDATE())
        )
    );
    
    -- Check for high fleet idle time
    INSERT INTO Alerts.OperationalAlerts (AlertType, Severity, Message)
    SELECT 
        'HighIdleTime',
        'Warning',
        'Vehicle ' + dv.VehicleNumber + ' on route ' + dr.RouteName + 
        ' has ' + CAST(SUM(fft.IdleTimeMinutes) AS VARCHAR) + ' minutes idle'
    FROM FactFleetTrips fft
    INNER JOIN DimVehicle dv ON fft.VehicleKey = dv.VehicleKey
    INNER JOIN DimRoute dr ON fft.RouteKey = dr.RouteKey
    INNER JOIN DimTime dt ON fft.StartTimeKey = dt.TimeKey
    WHERE dt.FullDateTime >= DATEADD(HOUR, -4, GETDATE())
    GROUP BY dv.VehicleNumber, dr.RouteName
    HAVING SUM(fft.IdleTimeMinutes) > 60;
END
GO
```

### Pattern 3: Gravity Zone Optimization

```sql
-- Recalculate product gravity scores and recommend zone reassignment
CREATE PROCEDURE usp_OptimizeGravityZones
AS
BEGIN
    -- Calculate velocity-based gravity
    WITH ProductVelocity AS (
        SELECT 
            dp.ProductKey,
            dp.ProductSKU,
            COUNT(*) AS PickFrequency,
            AVG(fwo.PickingTimeSeconds) AS AvgPickTime,
            SUM(fwo.QuantityHandled) AS TotalVolume
        FROM FactWarehouseOperations fwo
        INNER JOIN DimProduct dp ON fwo.ProductKey = dp.ProductKey
        INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
        WHERE dt.FullDateTime >= DATEADD(DAY, -30, GETDATE())
        GROUP BY dp.ProductKey, dp.ProductSKU
    )
    UPDATE dp
    SET 
        dp.GravityScore = (pv.PickFrequency * 100.0) / NULLIF(pv.AvgPickTime, 0),
        dp.OptimalZoneType = CASE
            WHEN (pv.PickFrequency * 100.0) / NULLIF(pv.AvgPickTime, 0) > 80 THEN 'High-Velocity'
            WHEN (pv.PickFrequency * 100.0) / NULLIF(pv.AvgPickTime, 0) > 40 THEN 'Medium-Velocity'
            ELSE 'Low-Velocity'
        END
    FROM DimProduct dp
    INNER JOIN ProductVelocity pv ON dp.ProductKey = pv.ProductKey
    WHERE dp.IsCurrent = 1;
    
    -- Generate recommendations
    SELECT 
        dp.ProductSKU,
        dp.ProductName,
        dw.WarehouseName,
        dw.ZoneType AS CurrentZone,
        dp.OptimalZoneType AS RecommendedZone,
        dp.GravityScore,
        'Consider relocating to ' + dp.OptimalZoneType + ' zone' AS Action
    FROM DimProduct dp
    CROSS JOIN DimWarehouse dw
    WHERE dp.OptimalZoneType != dw.ZoneType
      AND dp.IsCurrent = 1
    ORDER BY dp.GravityScore DESC;
END
GO
```

## Configuration & Customization

### Time Dimension Granularity

Adjust the time grain in `DimTime` population script:

```sql
-- Generate 15-minute intervals for 2 years
DECLARE @StartDate DATETIME2 = '2026-01-01';
DECLARE @EndDate DATETIME2 = '2027-12-31';
DECLARE @Interval INT = 15; -- minutes

WITH TimeGenerator AS (
    SELECT @StartDate AS TimeValue
    UNION ALL
    SELECT DATEADD(MINUTE, @Interval, TimeValue)
    FROM TimeGenerator
    WHERE TimeValue < @EndDate
)
INSERT INTO DimTime (TimeKey, FullDateTime, DateKey, Hour, Minute, QuarterHour)
SELECT 
    CONVERT(INT, FORMAT(TimeValue, 'yyyyMMddHHmm')),
    TimeValue,
    CONVERT(INT, FORMAT(TimeValue, 'yyyyMMdd')),
    DATEPART(HOUR, TimeValue),
    DATEPART(MINUTE, TimeValue),
    (DATEPART(MINUTE, TimeValue) / 15) + 1
FROM TimeGenerator
OPTION (MAXRECURSION 0);
```

### External Data Integration

Connect weather API for delay correlation:

```sql
-- Table to store external weather data
CREATE TABLE Staging.WeatherData (
    WeatherKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    GeographyKey INT,
    ObservationTime DATETIME2,
    TemperatureC DECIMAL(5,2),
    PrecipitationMM DECIMAL(6,2),
    WindSpeedKmh INT,
    VisibilityKm INT,
    IsSevereWeather BIT
);

-- Link to fleet trips
ALTER TABLE FactFleetTrips
ADD WeatherKey BIGINT,
    CONSTRAINT FK_FT_Weather FOREIGN KEY (WeatherKey) 
        REFERENCES Staging.WeatherData(WeatherKey);

-- Populate via external API (Python/PowerShell script scheduled task)
```

### Performance Tuning

```sql
-- Create columnstore index for analytics queries
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouseOps
ON FactWarehouseOperations (TimeKey, ProductKey, WarehouseKey, QuantityHandled, DwellTimeMinutes)
WITH (DROP_EXISTING = OFF, COMPRESSION_DELAY = 0);

-- Partition large fact tables by month
CREATE PARTITION SCHEME PS_FactFleetTrips
AS PARTITION PF_FleetTrips_Month
TO ([PRIMARY], [FG_2026Q1], [FG_2026Q2], [FG_2026Q3], [FG_2026Q4]);

-- Statistics for query optimization
CREATE STATISTICS STAT_WO_TimeProduct 
ON FactWarehouseOperations (TimeKey, ProductKey)
WITH FULLSCAN;
```

## Troubleshooting

### Issue: Power BI refresh fails with timeout

**Solution:** Switch from DirectQuery to Import mode or increase command timeout:

```sql
-- In Power BI Advanced Options when connecting:
Command timeout in minutes: 0 (unlimited)

-- Or optimize the query with indexed views
CREATE VIEW vw_WarehouseKPIs_Aggregated
WITH SCHEMABINDING
AS
SELECT 
    TimeKey,
    WarehouseKey,
    COUNT_BIG(*) AS TotalOps,
    SUM(QuantityHandled) AS TotalQty,
    AVG(DwellTimeMinutes) AS AvgDwell
FROM dbo.FactWarehouseOperations
GROUP BY TimeKey, WarehouseKey;

CREATE UNIQUE CLUSTERED INDEX UCI_WarehouseKPIs 
ON vw_WarehouseKPIs_Aggregated (TimeKey, WarehouseKey);
```

### Issue: SCD Type 2 dimension not updating correctly

**Solution:** Verify the effective/expiration date logic:

```sql
-- Correct SCD Type 2 update pattern
UPDATE DimProduct
SET ExpirationDate = GETDATE(), IsCurrent = 0
WHERE ProductSKU = @SKU AND IsCurrent = 1;

INSERT INTO DimProduct (ProductSKU, ProductName, GravityScore, EffectiveDate, IsCurrent)
VALUES (@SKU, @Name, @GravityScore, GETDATE(), 1);
```

### Issue: Cross-fact queries return incorrect results

**Solution:** Ensure dimension roles are properly defined:

```sql
-- Use role-playing dimensions for start/end time
SELECT 
    dt_start.FullDateTime AS TripStart,
    dt_end.FullDateTime AS TripEnd,
    DATEDIFF(MINUTE, dt_start.FullDateTime, dt_end.FullDateTime) AS TripDuration
FROM FactFleetTrips fft
INNER JOIN DimTime dt_start ON fft.StartTimeKey = dt_start.TimeKey
INNER JOIN DimTime dt_end ON fft.EndTimeKey = dt_end.TimeKey;
```

### Issue: Row-level security not applying in embedded reports

**Solution:** Ensure roles are published and users are assigned:

```powershell
# PowerShell to manage Power BI roles via API
$headers = @{
    "Authorization" = "Bearer ${env:POWERBI_ACCESS_TOKEN}"
}

$body = @{
    "datasetUserAccessRight" = "ReadWrite"
    "identifier" = "user@domain.com"
} | ConvertTo-Json

Invoke-RestMethod -Uri "https://api.powerbi.com/v1.0/myorg/groups/${WORKSPACE_ID}/datasets/${DATASET_ID}/users" `
    -Method Post -Headers $headers -Body $body -ContentType "application/json"
```

## Advanced Patterns

### Temporal Elasticity Simulation

```sql
-- Simulate capacity changes and predict bottlenecks
CREATE PROCEDURE usp_SimulateCapacityImpact
    @CapacityIncreasePct DECIMAL(5,2),
    @SimulationDays INT = 30
AS
BEGIN
    WITH SimulatedCapacity AS (
        SELECT 
            WarehouseKey,
            TotalCapacityM3 * (1 + @CapacityIncreasePct / 100.0) AS NewCapacity
        FROM DimWarehouse
    ),
    ProjectedLoad AS (
        SELECT 
            fwo.WarehouseKey,
            AVG(fwo.QuantityHandled) * @SimulationDays AS ProjectedVolume
        FROM FactWarehouseOperations fwo
        INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
        WHERE dt.FullDateTime >= DATEADD(DAY, -30, GETDATE())
        GROUP BY fwo.WarehouseKey
    )
    SELECT 
        dw.WarehouseName,
        sc.NewCapacity,
        pl.ProjectedVolume,
        (pl.ProjectedVolume / sc.NewCapacity * 100) AS ProjectedUtilization,
        CASE 
            WHEN (pl.ProjectedVolume / sc.NewCapacity) > 0.95 THEN 'Critical'
            WHEN (pl.ProjectedVolume / sc.NewCapacity) > 0.80 THEN 'Warning'
            ELSE 'Normal'
        END AS RiskLevel
    FROM DimWarehouse dw
    INNER JOIN SimulatedCapacity sc ON dw.WarehouseKey = sc.WarehouseKey
    INNER JOIN ProjectedLoad pl ON dw.WarehouseKey = pl.WarehouseKey;
END
GO
```

### Machine Learning Integration (via Azure ML)

```python
# Python script to train bottleneck prediction model
import pyodbc
import pandas as pd
from sklearn.ensemble import RandomForestClassifier
import joblib

# Connect to SQL Server
conn = pyodbc.connect(
    f"DRIVER={{ODBC Driver 17 for SQL Server}};"
    f"SERVER={os.environ['SQL_SERVER']};"
    f"DATABASE=LogiFleetPulse;"
    f"Trusted_Connection=yes;"
)

# Extract training data
query = """
SELECT 
    fwo.DwellTimeMinutes,
    dw.CurrentUtilizationPct,
    fft.IdleTimeMinutes,
    dp.GravityScore,
    CASE WHEN fwo.DwellTimeMinutes > 120 THEN 1 ELSE 0 END AS IsBottleneck
FROM FactWarehouseOperations fwo
