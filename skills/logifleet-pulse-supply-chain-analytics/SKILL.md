---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehouse with multi-fact star schema for warehouse, fleet, and supply chain analytics
triggers:
  - set up logifleet pulse supply chain analytics
  - configure logistics data warehouse with sql server
  - implement power bi fleet and warehouse dashboards
  - create multi-fact star schema for supply chain
  - integrate warehouse and fleet telemetry data
  - build cross-modal logistics intelligence platform
  - deploy logicore analytics supply chain engine
  - analyze warehouse gravity zones and fleet optimization
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive logistics intelligence platform that unifies warehouse operations, fleet telemetry, and supply chain data into a single MS SQL Server data warehouse with Power BI visualization. It implements a multi-fact star schema architecture that enables cross-domain KPI analysis (e.g., correlating warehouse dwell time with fleet idle time).

**Key Capabilities:**
- Multi-fact star schema with time-phased dimensions
- Real-time warehouse operations tracking (putaway, picking, packing, shipping)
- Fleet telemetry integration (GPS, fuel consumption, driver behavior)
- Cross-fact KPI harmonization via composite dimensions
- Predictive bottleneck detection using temporal modeling
- Warehouse "gravity zone" optimization based on pick frequency and item value
- Role-based access control with row-level security

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- Git for cloning the repository

### Clone the Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### Deploy SQL Schema

1. **Connect to your SQL Server instance:**

```sql
-- Run this in SQL Server Management Studio (SSMS)
-- First create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO
```

2. **Execute the schema deployment script:**

```bash
# From the repository root
sqlcmd -S your-server-name -d LogiFleetPulse -i ./sql/01_create_schema.sql
sqlcmd -S your-server-name -d LogiFleetPulse -i ./sql/02_create_dimensions.sql
sqlcmd -S your-server-name -d LogiFleetPulse -i ./sql/03_create_facts.sql
sqlcmd -S your-server-name -d LogiFleetPulse -i ./sql/04_create_views.sql
sqlcmd -S your-server-name -d LogiFleetPulse -i ./sql/05_create_stored_procedures.sql
```

### Configure Data Sources

Create a configuration file for your data sources:

```json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "SQL",
    "username": "${SQL_SERVER_USER}",
    "password": "${SQL_SERVER_PASSWORD}"
  },
  "data_sources": {
    "wms_api": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "api_key": "${WMS_API_KEY}"
    },
    "fleet_telemetry": {
      "endpoint": "${FLEET_TELEMETRY_ENDPOINT}",
      "api_key": "${FLEET_API_KEY}"
    },
    "weather_api": {
      "endpoint": "https://api.openweathermap.org/data/2.5",
      "api_key": "${WEATHER_API_KEY}"
    }
  },
  "refresh_intervals": {
    "warehouse_operations": "00:15:00",
    "fleet_trips": "00:15:00",
    "external_factors": "01:00:00"
  }
}
```

## Core SQL Schema Structure

### Dimension Tables

**DimTime** - Time dimension with 15-minute granularity:

```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    TimeStamp DATETIME2 NOT NULL,
    Hour INT NOT NULL,
    Minute INT NOT NULL,
    DayOfWeek INT NOT NULL,
    DayOfMonth INT NOT NULL,
    Month INT NOT NULL,
    Quarter INT NOT NULL,
    FiscalYear INT NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT NOT NULL,
    CONSTRAINT UQ_DimTime_TimeStamp UNIQUE (TimeStamp)
);

-- Create clustered columnstore index for better compression
CREATE CLUSTERED COLUMNSTORE INDEX CCI_DimTime ON DimTime;
```

**DimProductGravity** - Products with warehouse gravity scoring:

```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName NVARCHAR(200) NOT NULL,
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    UnitValue DECIMAL(10,2),
    FragilityScore DECIMAL(3,2), -- 0.0 to 1.0
    VelocityScore DECIMAL(5,2), -- Picks per day average
    GravityScore AS (UnitValue * 0.4 + VelocityScore * 0.4 + FragilityScore * 100 * 0.2) PERSISTED,
    GravityZone AS (
        CASE 
            WHEN (UnitValue * 0.4 + VelocityScore * 0.4 + FragilityScore * 100 * 0.2) > 80 THEN 'HIGH'
            WHEN (UnitValue * 0.4 + VelocityScore * 0.4 + FragilityScore * 100 * 0.2) > 40 THEN 'MEDIUM'
            ELSE 'LOW'
        END
    ) PERSISTED,
    EffectiveDate DATE NOT NULL,
    ExpiryDate DATE NULL
);

CREATE INDEX IX_DimProductGravity_GravityZone ON DimProductGravity(GravityZone);
```

**DimGeography** - Hierarchical location dimension:

```sql
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    NodeID VARCHAR(50) NOT NULL UNIQUE,
    NodeType VARCHAR(20) NOT NULL, -- 'WAREHOUSE', 'ROUTE_POINT', 'CUSTOMER'
    NodeName NVARCHAR(200),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    Address NVARCHAR(500),
    City NVARCHAR(100),
    StateProvince NVARCHAR(100),
    Country NVARCHAR(100),
    Continent NVARCHAR(50),
    PostalCode VARCHAR(20),
    TimeZone VARCHAR(50)
);

CREATE INDEX IX_DimGeography_NodeType ON DimGeography(NodeType);
CREATE INDEX IX_DimGeography_Country ON DimGeography(Country);
```

### Fact Tables

**FactWarehouseOperations** - Core warehouse activity fact:

```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    OperationType VARCHAR(20) NOT NULL, -- 'RECEIVING', 'PUTAWAY', 'PICKING', 'PACKING', 'SHIPPING'
    OperationID VARCHAR(50) NOT NULL,
    Quantity INT NOT NULL,
    DurationMinutes DECIMAL(8,2),
    DwellTimeHours DECIMAL(10,2),
    StorageZone VARCHAR(50),
    PickPathDistance DECIMAL(8,2), -- Meters
    WorkerID VARCHAR(50),
    BatchID VARCHAR(50),
    Cost DECIMAL(10,2),
    CONSTRAINT FK_FactWarehouseOps_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactWarehouseOps_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    CONSTRAINT FK_FactWarehouseOps_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimGeography(GeographyKey)
);

-- Partitioning by TimeKey for better performance
CREATE PARTITION FUNCTION PF_TimeKey (INT)
AS RANGE RIGHT FOR VALUES (/* Add time key boundaries */);

CREATE PARTITION SCHEME PS_FactWarehouseOps
AS PARTITION PF_TimeKey ALL TO ([PRIMARY]);

-- Create clustered index on partitioning key
CREATE CLUSTERED INDEX CIX_FactWarehouseOps_TimeKey 
ON FactWarehouseOperations(TimeKey) ON PS_FactWarehouseOps(TimeKey);
```

**FactFleetTrips** - Fleet telemetry and trip data:

```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    StartTimeKey INT NOT NULL,
    EndTimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    DriverKey INT NOT NULL,
    OriginKey INT NOT NULL,
    DestinationKey INT NOT NULL,
    RouteID VARCHAR(50) NOT NULL,
    TripID VARCHAR(50) NOT NULL UNIQUE,
    DistanceKM DECIMAL(10,2),
    DurationMinutes DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(8,2),
    IdleTimeMinutes DECIMAL(8,2),
    LoadingTimeMinutes DECIMAL(8,2),
    UnloadingTimeMinutes DECIMAL(8,2),
    AverageSpeedKPH DECIMAL(6,2),
    MaxSpeedKPH DECIMAL(6,2),
    LoadWeight DECIMAL(10,2),
    WeatherCondition VARCHAR(50),
    TrafficDelay BIT,
    Cost DECIMAL(10,2),
    CONSTRAINT FK_FactFleetTrips_StartTime FOREIGN KEY (StartTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactFleetTrips_EndTime FOREIGN KEY (EndTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactFleetTrips_Origin FOREIGN KEY (OriginKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FactFleetTrips_Destination FOREIGN KEY (DestinationKey) REFERENCES DimGeography(GeographyKey)
);

CREATE INDEX IX_FactFleetTrips_StartTime ON FactFleetTrips(StartTimeKey);
CREATE INDEX IX_FactFleetTrips_Route ON FactFleetTrips(RouteID);
```

## Key Stored Procedures

### Cross-Fact KPI Query

```sql
CREATE PROCEDURE usp_GetCrossDomainKPIs
    @StartDate DATE,
    @EndDate DATE,
    @WarehouseID INT = NULL
AS
BEGIN
    SET NOCOUNT ON;

    -- Cross-fact query linking warehouse dwell time with fleet idle time
    SELECT 
        dt.FiscalYear,
        dt.Month,
        dg.NodeName AS Warehouse,
        dp.Category AS ProductCategory,
        
        -- Warehouse KPIs
        AVG(fwo.DwellTimeHours) AS AvgDwellTimeHours,
        SUM(fwo.Quantity) AS TotalUnitsProcessed,
        AVG(fwo.DurationMinutes) AS AvgOperationDuration,
        
        -- Fleet KPIs (for shipments from this warehouse)
        AVG(fft.IdleTimeMinutes) AS AvgFleetIdleTime,
        AVG(fft.FuelConsumedLiters / NULLIF(fft.DistanceKM, 0)) AS AvgFuelEfficiency,
        SUM(fft.Cost) AS TotalFleetCost,
        
        -- Cross-domain metrics
        AVG(fwo.DwellTimeHours) * AVG(fft.IdleTimeMinutes) AS FrictionIndex,
        
        -- Gravity zone distribution
        dp.GravityZone,
        COUNT(DISTINCT fwo.OperationID) AS OperationCount
        
    FROM FactWarehouseOperations fwo
    INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
    INNER JOIN DimGeography dg ON fwo.WarehouseKey = dg.GeographyKey
    LEFT JOIN FactFleetTrips fft ON fft.OriginKey = fwo.WarehouseKey 
        AND fft.StartTimeKey BETWEEN dt.TimeKey - 96 AND dt.TimeKey + 96 -- +/- 24 hours
    
    WHERE dt.TimeStamp BETWEEN @StartDate AND @EndDate
        AND (@WarehouseID IS NULL OR fwo.WarehouseKey = @WarehouseID)
        
    GROUP BY 
        dt.FiscalYear,
        dt.Month,
        dg.NodeName,
        dp.Category,
        dp.GravityZone
        
    ORDER BY 
        dt.FiscalYear DESC,
        dt.Month DESC,
        FrictionIndex DESC;
END;
GO
```

### Warehouse Gravity Zone Optimization

```sql
CREATE PROCEDURE usp_RecommendGravityZoneReassignments
    @WarehouseKey INT,
    @AnalysisPeriodDays INT = 30
AS
BEGIN
    SET NOCOUNT ON;

    -- Analyze product movement patterns and recommend zone reassignments
    WITH ProductVelocity AS (
        SELECT 
            fwo.ProductKey,
            dp.SKU,
            dp.ProductName,
            dp.GravityZone AS CurrentZone,
            COUNT(*) AS PickCount,
            AVG(fwo.PickPathDistance) AS AvgPickDistance,
            AVG(fwo.DurationMinutes) AS AvgPickDuration,
            SUM(fwo.Quantity) AS TotalQuantity,
            -- Calculate new gravity score based on actual velocity
            (dp.UnitValue * 0.4 + (COUNT(*) * 1.0 / @AnalysisPeriodDays) * 0.4 + dp.FragilityScore * 100 * 0.2) AS RecalculatedGravity
        FROM FactWarehouseOperations fwo
        INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
        INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
        WHERE fwo.WarehouseKey = @WarehouseKey
            AND dt.TimeStamp >= DATEADD(DAY, -@AnalysisPeriodDays, GETDATE())
            AND fwo.OperationType = 'PICKING'
        GROUP BY fwo.ProductKey, dp.SKU, dp.ProductName, dp.GravityZone, dp.UnitValue, dp.FragilityScore
    )
    SELECT 
        SKU,
        ProductName,
        CurrentZone,
        CASE 
            WHEN RecalculatedGravity > 80 THEN 'HIGH'
            WHEN RecalculatedGravity > 40 THEN 'MEDIUM'
            ELSE 'LOW'
        END AS RecommendedZone,
        RecalculatedGravity,
        PickCount,
        AvgPickDistance,
        AvgPickDuration,
        TotalQuantity,
        CASE 
            WHEN CurrentZone != CASE 
                WHEN RecalculatedGravity > 80 THEN 'HIGH'
                WHEN RecalculatedGravity > 40 THEN 'MEDIUM'
                ELSE 'LOW'
            END THEN 'REASSIGNMENT RECOMMENDED'
            ELSE 'OPTIMAL'
        END AS Status,
        -- Estimate time savings if moved to correct zone
        CASE 
            WHEN CurrentZone = 'LOW' AND RecalculatedGravity > 80 
                THEN AvgPickDistance * 0.4 * PickCount -- 40% distance reduction
            WHEN CurrentZone = 'HIGH' AND RecalculatedGravity < 40 
                THEN -AvgPickDistance * 0.3 * PickCount -- Negative = freed high-value space
            ELSE 0
        END AS EstimatedTimeSavingsMinutes
    FROM ProductVelocity
    WHERE CurrentZone != CASE 
            WHEN RecalculatedGravity > 80 THEN 'HIGH'
            WHEN RecalculatedGravity > 40 THEN 'MEDIUM'
            ELSE 'LOW'
        END
    ORDER BY ABS(EstimatedTimeSavingsMinutes) DESC;
END;
GO
```

### Predictive Bottleneck Detection

```sql
CREATE PROCEDURE usp_PredictBottlenecks
    @ForecastHorizonHours INT = 24,
    @ThresholdPercentile DECIMAL(3,2) = 0.90
AS
BEGIN
    SET NOCOUNT ON;

    -- Predict bottlenecks using historical patterns
    WITH HistoricalPatterns AS (
        SELECT 
            dt.Hour,
            dt.DayOfWeek,
            fwo.WarehouseKey,
            fwo.StorageZone,
            AVG(fwo.DurationMinutes) AS AvgDuration,
            STDEV(fwo.DurationMinutes) AS StdDevDuration,
            COUNT(*) AS OperationCount,
            PERCENTILE_CONT(@ThresholdPercentile) 
                WITHIN GROUP (ORDER BY fwo.DurationMinutes) 
                OVER (PARTITION BY dt.Hour, dt.DayOfWeek, fwo.WarehouseKey, fwo.StorageZone) AS Percentile90Duration
        FROM FactWarehouseOperations fwo
        INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
        WHERE dt.TimeStamp >= DATEADD(DAY, -90, GETDATE()) -- 90 day history
        GROUP BY dt.Hour, dt.DayOfWeek, fwo.WarehouseKey, fwo.StorageZone
    ),
    FutureTimeWindows AS (
        SELECT 
            DATEADD(HOUR, n, GETDATE()) AS ForecastTime,
            DATEPART(HOUR, DATEADD(HOUR, n, GETDATE())) AS Hour,
            DATEPART(WEEKDAY, DATEADD(HOUR, n, GETDATE())) AS DayOfWeek
        FROM (SELECT TOP (@ForecastHorizonHours) ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS n FROM sys.objects) nums
    )
    SELECT 
        ftw.ForecastTime,
        dg.NodeName AS Warehouse,
        hp.StorageZone,
        hp.AvgDuration,
        hp.Percentile90Duration,
        hp.OperationCount AS HistoricalOpsAtThisTime,
        CASE 
            WHEN hp.Percentile90Duration > hp.AvgDuration * 1.5 THEN 'HIGH RISK'
            WHEN hp.Percentile90Duration > hp.AvgDuration * 1.2 THEN 'MODERATE RISK'
            ELSE 'LOW RISK'
        END AS BottleneckRisk,
        -- Confidence based on data availability
        CASE 
            WHEN hp.OperationCount > 100 THEN 'HIGH'
            WHEN hp.OperationCount > 30 THEN 'MEDIUM'
            ELSE 'LOW'
        END AS PredictionConfidence
    FROM FutureTimeWindows ftw
    INNER JOIN HistoricalPatterns hp ON ftw.Hour = hp.Hour AND ftw.DayOfWeek = hp.DayOfWeek
    INNER JOIN DimGeography dg ON hp.WarehouseKey = dg.GeographyKey
    WHERE hp.Percentile90Duration > hp.AvgDuration * 1.2 -- Only show potential bottlenecks
    ORDER BY ftw.ForecastTime, BottleneckRisk DESC, hp.Percentile90Duration DESC;
END;
GO
```

## Power BI Configuration

### Connect to SQL Server

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Enter connection details:
   - Server: `${SQL_SERVER_HOST}`
   - Database: `LogiFleetPulse`
   - Data Connectivity mode: DirectQuery (for real-time) or Import (for better performance)

### Import the Template

```bash
# The repository includes a pre-configured template
# Open LogiFleet_Pulse_Master.pbit and enter your connection details
```

### Key DAX Measures

**Cross-Domain Friction Index:**

```dax
Friction Index = 
VAR AvgDwellTime = AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR AvgIdleTime = AVERAGE(FactFleetTrips[IdleTimeMinutes])
RETURN
    AvgDwellTime * AvgIdleTime / 60 -- Normalize to consistent units
```

**Gravity Zone Efficiency:**

```dax
Gravity Zone Efficiency % = 
VAR HighGravityInHighZone = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        DimProductGravity[GravityZone] = "HIGH",
        FactWarehouseOperations[StorageZone] IN {"A1", "A2", "B1"} -- Define high-priority zones
    )
VAR TotalHighGravityOps = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        DimProductGravity[GravityZone] = "HIGH"
    )
RETURN
    DIVIDE(HighGravityInHighZone, TotalHighGravityOps, 0)
```

**Fleet Utilization Rate:**

```dax
Fleet Utilization % = 
VAR TotalTripTime = SUM(FactFleetTrips[DurationMinutes])
VAR TotalIdleTime = SUM(FactFleetTrips[IdleTimeMinutes])
RETURN
    DIVIDE(TotalTripTime - TotalIdleTime, TotalTripTime, 0)
```

**Temporal Elasticity Scenario:**

```dax
Scenario: 95% Capacity Impact = 
VAR CurrentCapacity = 0.80 -- Current warehouse fill rate
VAR TargetCapacity = 0.95
VAR CapacityMultiplier = TargetCapacity / CurrentCapacity
VAR BaseOpsCount = COUNTROWS(FactWarehouseOperations)
VAR ProjectedOpsCount = BaseOpsCount * CapacityMultiplier
VAR AvgDuration = AVERAGE(FactWarehouseOperations[DurationMinutes])
VAR ConvestionFactor = 1 + ((TargetCapacity - 0.85) * 2) -- Exponential congestion above 85%
RETURN
    ProjectedOpsCount * AvgDuration * ConvestionFactor
```

## Data Loading Patterns

### Incremental Load Stored Procedure

```sql
CREATE PROCEDURE usp_IncrementalLoad_WarehouseOperations
    @LastLoadTimestamp DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    BEGIN TRANSACTION;
    
    BEGIN TRY
        -- Insert new warehouse operations since last load
        INSERT INTO FactWarehouseOperations (
            TimeKey, ProductKey, WarehouseKey, OperationType, 
            OperationID, Quantity, DurationMinutes, DwellTimeHours,
            StorageZone, PickPathDistance, WorkerID, BatchID, Cost
        )
        SELECT 
            dt.TimeKey,
            dp.ProductKey,
            dg.GeographyKey,
            src.OperationType,
            src.OperationID,
            src.Quantity,
            src.DurationMinutes,
            src.DwellTimeHours,
            src.StorageZone,
            src.PickPathDistance,
            src.WorkerID,
            src.BatchID,
            src.Cost
        FROM ExternalDataSource.WarehouseOps src
        INNER JOIN DimTime dt ON DATEADD(MINUTE, (DATEPART(MINUTE, src.OperationTimestamp) / 15) * 15, 
            DATEADD(HOUR, DATEDIFF(HOUR, 0, src.OperationTimestamp), 0)) = dt.TimeStamp
        INNER JOIN DimProductGravity dp ON src.SKU = dp.SKU 
            AND src.OperationTimestamp BETWEEN dp.EffectiveDate AND ISNULL(dp.ExpiryDate, '9999-12-31')
        INNER JOIN DimGeography dg ON src.WarehouseID = dg.NodeID
        WHERE src.OperationTimestamp > @LastLoadTimestamp
            AND NOT EXISTS (
                SELECT 1 FROM FactWarehouseOperations fwo 
                WHERE fwo.OperationID = src.OperationID
            );
        
        -- Update last load timestamp
        UPDATE ETL_LoadControl 
        SET LastLoadTimestamp = GETDATE() 
        WHERE TableName = 'FactWarehouseOperations';
        
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        THROW;
    END CATCH
END;
GO
```

### External Data Source Configuration

```sql
-- Configure external data source for streaming data
CREATE MASTER KEY ENCRYPTION BY PASSWORD = '${MASTER_KEY_PASSWORD}';

CREATE DATABASE SCOPED CREDENTIAL WMS_Credential
WITH IDENTITY = 'WMS_API_USER',
     SECRET = '${WMS_API_KEY}';

CREATE EXTERNAL DATA SOURCE WMS_API
WITH (
    TYPE = RDBMS,
    LOCATION = '${WMS_DATABASE_SERVER}',
    DATABASE_NAME = 'WMS_Production',
    CREDENTIAL = WMS_Credential
);

CREATE EXTERNAL TABLE ExternalDataSource.WarehouseOps (
    OperationID VARCHAR(50),
    OperationType VARCHAR(20),
    OperationTimestamp DATETIME2,
    SKU VARCHAR(50),
    WarehouseID VARCHAR(50),
    Quantity INT,
    DurationMinutes DECIMAL(8,2),
    DwellTimeHours DECIMAL(10,2),
    StorageZone VARCHAR(50),
    PickPathDistance DECIMAL(8,2),
    WorkerID VARCHAR(50),
    BatchID VARCHAR(50),
    Cost DECIMAL(10,2)
)
WITH (
    DATA_SOURCE = WMS_API,
    SCHEMA_NAME = 'dbo',
    OBJECT_NAME = 'Operations'
);
```

## Automated Alerting

### Create Alert Stored Procedure

```sql
CREATE PROCEDURE usp_CheckKPIThresholds
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertRecipients VARCHAR(MAX) = '${ALERT_EMAIL_RECIPIENTS}';
    DECLARE @AlertBody NVARCHAR(MAX) = '';
    DECLARE @AlertSubject NVARCHAR(200) = '';
    DECLARE @ThresholdBreached BIT = 0;
    
    -- Check fleet idle time threshold (> 15% of trip duration)
    IF EXISTS (
        SELECT 1 
        FROM FactFleetTrips 
        WHERE StartTimeKey >= (SELECT TimeKey FROM DimTime WHERE TimeStamp >= DATEADD(HOUR, -4, GETDATE()))
            AND (IdleTimeMinutes / NULLIF(DurationMinutes, 0)) > 0.15
    )
    BEGIN
        SET @ThresholdBreached = 1;
        SET @AlertBody = @AlertBody + CHAR(13) + CHAR(10) + 
            'ALERT: Fleet idle time exceeds 15% threshold in the last 4 hours.' + CHAR(13) + CHAR(10);
    END
    
    -- Check warehouse dwell time (> 72 hours for high-gravity items)
    IF EXISTS (
        SELECT 1
        FROM FactWarehouseOperations fwo
        INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
        WHERE fwo.TimeKey >= (SELECT TimeKey FROM DimTime WHERE TimeStamp >= DATEADD(HOUR, -4, GETDATE()))
            AND dp.GravityZone = 'HIGH'
            AND fwo.DwellTimeHours > 72
    )
    BEGIN
        SET @ThresholdBreached = 1;
        SET @AlertBody = @AlertBody + CHAR(13) + CHAR(10) + 
            'ALERT: High-gravity items with dwell time > 72 hours detected.' + CHAR(13) + CHAR(10);
    END
    
    -- Check gravity zone misalignment (> 20% of operations)
    DECLARE @MisalignmentPct DECIMAL(5,2);
    SELECT @MisalignmentPct = 
        CAST(SUM(CASE 
            WHEN (dp.GravityZone = 'HIGH' AND fwo.StorageZone NOT IN ('A1', 'A2', 'B1'))
                OR (dp.GravityZone = 'LOW' AND fwo.StorageZone IN ('A1', 'A2', 'B1'))
            THEN 1 ELSE 0 END) AS DECIMAL(10,2)) / COUNT(*) * 100
    FROM FactWarehouseOperations fwo
    INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
    WHERE fwo.TimeKey >= (SELECT TimeKey FROM DimTime WHERE TimeStamp >= DATEADD(DAY, -1, GETDATE()));
    
    IF @MisalignmentPct > 20
    BEGIN
        SET @ThresholdBreached = 1;
        SET @AlertBody = @AlertBody + CHAR(13) + CHAR(10) + 
            'ALERT: Gravity zone misalignment at ' + CAST(@MisalignmentPct AS VARCHAR(10)) + '%.' + CHAR(13) + CHAR(10);
    END
    
    -- Send alert if any threshold breached
    IF @ThresholdBreached = 1
    BEGIN
        SET @AlertSubject = 'LogiFleet Pulse - KPI Threshold Alert';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = @AlertRecipients,
            @subject = @AlertSubject,
            @body = @AlertBody,
            @body_format = 'TEXT';
    END
END;
GO

-- Schedule this to run every 15 minutes via SQL Server Agent
```

## Row-Level Security Implementation

```sql
-- Create security predicate function
CREATE FUNCTION dbo.fn_SecurityPredicate(@WarehouseKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN (
    SELECT 1 AS Result
    WHERE 
        -- Admins see all
        IS_MEMBER('LogiFleet_Admin') = 1
        
