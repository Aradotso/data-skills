---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing template for logistics, fleet management, and supply chain analytics with multi-fact star schema
triggers:
  - "set up logistics analytics dashboard"
  - "implement supply chain data warehouse"
  - "create fleet management Power BI reports"
  - "configure warehouse operations analytics"
  - "deploy logifleet pulse schema"
  - "build logistics intelligence platform"
  - "integrate fleet telemetry with warehouse data"
  - "design multi-fact star schema for supply chain"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive data warehousing and analytics template for logistics operations. It provides:

- **Multi-fact star schema** for warehouse operations, fleet trips, and cross-dock activities
- **Power BI dashboards** for real-time logistics intelligence
- **MS SQL Server scripts** for dimensional modeling with time-phased dimensions
- **Integrated KPI framework** linking warehouse velocity with fleet performance
- **Predictive analytics** for bottleneck detection and resource optimization

The platform harmonizes data from WMS, telematics, GPS feeds, supplier portals, and external APIs into a unified semantic layer.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS)
- Access to data sources (WMS, fleet telemetry, ERP systems)

### Initial Deployment

1. **Clone the repository**:
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy SQL schema**:
```sql
-- Connect to your SQL Server instance in SSMS
-- Execute the schema creation script
USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Run the schema deployment script
:r .\sql\01_create_schema.sql
:r .\sql\02_create_dimensions.sql
:r .\sql\03_create_facts.sql
:r .\sql\04_create_views.sql
:r .\sql\05_create_stored_procedures.sql
```

3. **Configure data sources** in `config.json`:
```json
{
  "sqlServer": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "SqlLogin",
    "username": "${SQL_USERNAME}",
    "password": "${SQL_PASSWORD}"
  },
  "dataSources": {
    "wms": {
      "connectionString": "${WMS_CONNECTION_STRING}",
      "refreshInterval": 900
    },
    "telematics": {
      "apiEndpoint": "${FLEET_API_ENDPOINT}",
      "apiKey": "${FLEET_API_KEY}"
    }
  }
}
```

## Core Data Model

### Dimension Tables

```sql
-- DimTime: 15-minute granularity time dimension
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    TimeInterval VARCHAR(10),
    HourOfDay TINYINT,
    DayOfWeek VARCHAR(10),
    FiscalPeriod VARCHAR(10),
    IsBusinessHour BIT
);

-- DimGeography: Hierarchical location dimension
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY,
    LocationID VARCHAR(50) NOT NULL,
    LocationName VARCHAR(200),
    LocationType VARCHAR(50), -- Warehouse, DistributionCenter, RouteNode
    Region VARCHAR(100),
    Country VARCHAR(100),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6)
);

-- DimProductGravity: Product categorization with velocity scoring
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(100) NOT NULL,
    ProductName VARCHAR(500),
    Category VARCHAR(100),
    GravityScore DECIMAL(5,2), -- Computed: velocity * value * fragility
    VelocityClass VARCHAR(20), -- Fast, Medium, Slow
    ValueClass VARCHAR(20), -- High, Medium, Low
    FragilityScore TINYINT
);

-- DimSupplierReliability: Supplier performance metrics
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY,
    SupplierID VARCHAR(50) NOT NULL,
    SupplierName VARCHAR(200),
    LeadTimeVariance DECIMAL(5,2),
    DefectPercentage DECIMAL(5,4),
    ComplianceScore DECIMAL(5,2),
    ReliabilityTier VARCHAR(20) -- Platinum, Gold, Silver, Bronze
);
```

### Fact Tables

```sql
-- FactWarehouseOperations: Core warehouse activity metrics
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType VARCHAR(50), -- Receiving, Putaway, Picking, Packing, Shipping
    QuantityHandled INT,
    DwellTimeMinutes INT,
    CycleTimeSeconds INT,
    LabourMinutes DECIMAL(10,2),
    StorageZone VARCHAR(50),
    OperationCost DECIMAL(12,2)
);

-- FactFleetTrips: Fleet and route performance
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50),
    DistanceKM DECIMAL(10,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLitres DECIMAL(10,3),
    LoadWeight DECIMAL(10,2),
    TripCost DECIMAL(12,2),
    DelayMinutes INT,
    DelayReason VARCHAR(100)
);

-- FactCrossDock: Cross-docking activities
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    TransferTimeMinutes INT,
    QuantityTransferred INT,
    TemperatureCompliance BIT
);
```

## Key Stored Procedures

### Data Loading Procedures

```sql
-- Incremental load for warehouse operations
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME,
    @CurrentLoadDateTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        QuantityHandled, DwellTimeMinutes, CycleTimeSeconds,
        LabourMinutes, StorageZone, OperationCost
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        src.OperationType,
        src.Quantity,
        DATEDIFF(MINUTE, src.StartTime, src.EndTime) AS DwellTimeMinutes,
        DATEDIFF(SECOND, src.StartTime, src.EndTime) AS CycleTimeSeconds,
        src.LabourMinutes,
        src.StorageZone,
        src.OperationCost
    FROM ExternalWMS.dbo.Operations src
    INNER JOIN DimTime t ON DATEPART(HOUR, src.StartTime) = t.HourOfDay 
        AND DATEPART(MINUTE, src.StartTime) / 15 * 15 = t.TimeInterval
    INNER JOIN DimGeography g ON src.WarehouseID = g.LocationID
    INNER JOIN DimProductGravity p ON src.SKU = p.SKU
    WHERE src.LoadDateTime > @LastLoadDateTime
        AND src.LoadDateTime <= @CurrentLoadDateTime;
        
    -- Update last load timestamp
    UPDATE ETL.LoadControl
    SET LastLoadDateTime = @CurrentLoadDateTime
    WHERE TableName = 'FactWarehouseOperations';
END;
GO

-- Load fleet trips with telemetry data
CREATE PROCEDURE usp_LoadFleetTrips
    @LoadDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactFleetTrips (
        TimeKey, OriginGeographyKey, DestinationGeographyKey,
        VehicleID, DriverID, DistanceKM, DurationMinutes,
        IdleTimeMinutes, FuelConsumedLitres, LoadWeight,
        TripCost, DelayMinutes, DelayReason
    )
    SELECT 
        t.TimeKey,
        og.GeographyKey AS OriginGeographyKey,
        dg.GeographyKey AS DestinationGeographyKey,
        tel.VehicleID,
        tel.DriverID,
        tel.TotalDistanceKM,
        tel.TotalDurationMinutes,
        tel.IdleTimeMinutes,
        tel.FuelConsumed,
        tel.LoadWeight,
        (tel.FuelConsumed * @FuelPricePerLitre + tel.TotalDurationMinutes * @DriverHourlyCost / 60) AS TripCost,
        CASE WHEN tel.ActualArrival > tel.PlannedArrival 
            THEN DATEDIFF(MINUTE, tel.PlannedArrival, tel.ActualArrival)
            ELSE 0 END AS DelayMinutes,
        tel.DelayReason
    FROM ExternalTelematics.dbo.TripData tel
    INNER JOIN DimTime t ON CAST(tel.DepartureTime AS DATE) = @LoadDate
    INNER JOIN DimGeography og ON tel.OriginLocationID = og.LocationID
    INNER JOIN DimGeography dg ON tel.DestinationLocationID = dg.LocationID
    WHERE CAST(tel.DepartureTime AS DATE) = @LoadDate;
END;
GO
```

### Analytics Procedures

```sql
-- Calculate warehouse gravity zone recommendations
CREATE PROCEDURE usp_CalculateGravityZoneRecommendations
AS
BEGIN
    WITH ProductVelocity AS (
        SELECT 
            ProductKey,
            COUNT(*) AS PickFrequency,
            AVG(CycleTimeSeconds) AS AvgCycleTime,
            SUM(QuantityHandled) AS TotalVolume
        FROM FactWarehouseOperations
        WHERE OperationType = 'Picking'
            AND TimeKey >= (SELECT TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(DAY, -30, GETDATE()))
        GROUP BY ProductKey
    )
    SELECT 
        p.SKU,
        p.ProductName,
        p.GravityScore AS CurrentGravityScore,
        (pv.PickFrequency * 0.4 + 
         (1000.0 / NULLIF(pv.AvgCycleTime, 0)) * 0.3 + 
         (pv.TotalVolume / 1000.0) * 0.3) AS RecommendedGravityScore,
        CASE 
            WHEN (pv.PickFrequency * 0.4 + 
                  (1000.0 / NULLIF(pv.AvgCycleTime, 0)) * 0.3 + 
                  (pv.TotalVolume / 1000.0) * 0.3) > 75 THEN 'High-Gravity'
            WHEN (pv.PickFrequency * 0.4 + 
                  (1000.0 / NULLIF(pv.AvgCycleTime, 0)) * 0.3 + 
                  (pv.TotalVolume / 1000.0) * 0.3) > 40 THEN 'Medium-Gravity'
            ELSE 'Low-Gravity'
        END AS RecommendedZone
    FROM DimProductGravity p
    INNER JOIN ProductVelocity pv ON p.ProductKey = pv.ProductKey
    ORDER BY RecommendedGravityScore DESC;
END;
GO

-- Predictive bottleneck detection
CREATE PROCEDURE usp_DetectBottlenecks
    @ThresholdPercentile DECIMAL(3,2) = 0.90
AS
BEGIN
    WITH OperationMetrics AS (
        SELECT 
            g.LocationName,
            wo.StorageZone,
            wo.OperationType,
            AVG(wo.CycleTimeSeconds) AS AvgCycleTime,
            PERCENTILE_CONT(@ThresholdPercentile) 
                WITHIN GROUP (ORDER BY wo.CycleTimeSeconds) 
                OVER (PARTITION BY g.LocationName, wo.StorageZone, wo.OperationType) AS P90CycleTime
        FROM FactWarehouseOperations wo
        INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
        WHERE wo.TimeKey >= (SELECT TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(HOUR, -24, GETDATE()))
        GROUP BY g.LocationName, wo.StorageZone, wo.OperationType
    )
    SELECT 
        LocationName,
        StorageZone,
        OperationType,
        AvgCycleTime,
        P90CycleTime,
        (P90CycleTime - AvgCycleTime) / NULLIF(AvgCycleTime, 0) * 100 AS VariabilityPercent,
        CASE 
            WHEN (P90CycleTime - AvgCycleTime) / NULLIF(AvgCycleTime, 0) > 0.5 THEN 'Critical'
            WHEN (P90CycleTime - AvgCycleTime) / NULLIF(AvgCycleTime, 0) > 0.3 THEN 'Warning'
            ELSE 'Normal'
        END AS BottleneckStatus
    FROM OperationMetrics
    WHERE (P90CycleTime - AvgCycleTime) / NULLIF(AvgCycleTime, 0) > 0.2
    ORDER BY VariabilityPercent DESC;
END;
GO
```

## Power BI Integration

### DAX Measures

Create these measures in Power BI after connecting to the SQL Server database:

```dax
// Cross-fact KPI: Fleet efficiency per warehouse throughput
Fleet Efficiency Index = 
VAR TotalTripCost = SUM(FactFleetTrips[TripCost])
VAR TotalWarehouseOps = CALCULATE(
    COUNT(FactWarehouseOperations[OperationKey]),
    FactWarehouseOperations[OperationType] = "Shipping"
)
RETURN
DIVIDE(TotalWarehouseOps, TotalTripCost, 0) * 100

// Dwell time analysis with time intelligence
Avg Dwell Time (30D) = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    DATESINPERIOD(
        DimTime[FullDateTime],
        LASTDATE(DimTime[FullDateTime]),
        -30,
        DAY
    )
)

// Gravity zone compliance percentage
Gravity Zone Compliance = 
VAR HighGravityProducts = 
    CALCULATETABLE(
        VALUES(DimProductGravity[ProductKey]),
        DimProductGravity[GravityScore] > 75
    )
VAR HighGravityInCorrectZone = 
    CALCULATE(
        COUNT(FactWarehouseOperations[OperationKey]),
        FactWarehouseOperations[StorageZone] IN {"A", "B"}, // Near shipping dock
        DimProductGravity[ProductKey] IN HighGravityProducts
    )
VAR TotalHighGravityOps = 
    CALCULATE(
        COUNT(FactWarehouseOperations[OperationKey]),
        DimProductGravity[ProductKey] IN HighGravityProducts
    )
RETURN
DIVIDE(HighGravityInCorrectZone, TotalHighGravityOps, 0)

// Fleet idle time percentage with threshold alerting
Fleet Idle % = 
VAR IdleTime = SUM(FactFleetTrips[IdleTimeMinutes])
VAR TotalTime = SUM(FactFleetTrips[DurationMinutes])
VAR IdlePercentage = DIVIDE(IdleTime, TotalTime, 0)
RETURN
IF(IdlePercentage > 0.15, IdlePercentage & " ⚠️", IdlePercentage)

// Supplier reliability impact on dwell time
Supplier Impact Score = 
VAR AvgDwellReliableSuppliers = 
    CALCULATE(
        AVERAGE(FactCrossDock[TransferTimeMinutes]),
        DimSupplierReliability[ComplianceScore] > 90
    )
VAR AvgDwellAllSuppliers = AVERAGE(FactCrossDock[TransferTimeMinutes])
RETURN
DIVIDE(AvgDwellAllSuppliers - AvgDwellReliableSuppliers, AvgDwellAllSuppliers, 0) * 100
```

### Power BI Template Configuration

1. **Open the template**:
```powershell
# From repository root
Start-Process ".\powerbi\LogiFleet_Pulse_Master.pbit"
```

2. **Update data source connections**:
   - Click Transform Data → Data Source Settings
   - Update SQL Server connection string
   - Enter credentials (use Windows Authentication or SQL Login)

3. **Refresh data model**:
```dax
// In Power Query, verify these source queries are configured:
Source_FactWarehouseOperations = Sql.Database("${SQL_SERVER_HOST}", "LogiFleetPulse", [Query="SELECT * FROM FactWarehouseOperations"])
Source_FactFleetTrips = Sql.Database("${SQL_SERVER_HOST}", "LogiFleetPulse", [Query="SELECT * FROM FactFleetTrips"])
```

## Common Usage Patterns

### Pattern 1: Daily Warehouse Performance Report

```sql
-- Executive summary view
CREATE VIEW vw_DailyWarehouseSummary AS
SELECT 
    CAST(t.FullDateTime AS DATE) AS OperationDate,
    g.LocationName,
    COUNT(DISTINCT wo.OperationKey) AS TotalOperations,
    SUM(wo.QuantityHandled) AS TotalUnitsProcessed,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
    AVG(wo.CycleTimeSeconds) AS AvgCycleTime,
    SUM(wo.OperationCost) AS TotalOperationCost,
    SUM(wo.LabourMinutes) / 60.0 AS TotalLabourHours
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
GROUP BY CAST(t.FullDateTime AS DATE), g.LocationName;
GO

-- Query for today's performance vs last 7 days
SELECT 
    LocationName,
    TotalUnitsProcessed AS Today_Units,
    LAG(TotalUnitsProcessed, 7) OVER (PARTITION BY LocationName ORDER BY OperationDate) AS LastWeek_Units,
    (TotalUnitsProcessed - LAG(TotalUnitsProcessed, 7) OVER (PARTITION BY LocationName ORDER BY OperationDate)) / 
        NULLIF(LAG(TotalUnitsProcessed, 7) OVER (PARTITION BY LocationName ORDER BY OperationDate), 0) * 100 AS PercentChange
FROM vw_DailyWarehouseSummary
WHERE OperationDate = CAST(GETDATE() AS DATE)
ORDER BY PercentChange DESC;
```

### Pattern 2: Cross-Modal Cost Analysis

```sql
-- Analyze total supply chain cost per SKU
WITH WarehouseCosts AS (
    SELECT 
        ProductKey,
        SUM(OperationCost) AS TotalWarehouseCost,
        AVG(DwellTimeMinutes) AS AvgDwellTime
    FROM FactWarehouseOperations
    WHERE TimeKey >= (SELECT TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(MONTH, -1, GETDATE()))
    GROUP BY ProductKey
),
FleetCosts AS (
    SELECT 
        cd.ProductKey,
        SUM(ft.TripCost) AS TotalFleetCost
    FROM FactCrossDock cd
    INNER JOIN FactFleetTrips ft ON cd.OutboundTripKey = ft.TripKey
    WHERE cd.TimeKey >= (SELECT TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(MONTH, -1, GETDATE()))
    GROUP BY cd.ProductKey
)
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    ISNULL(wc.TotalWarehouseCost, 0) AS WarehouseCost,
    ISNULL(fc.TotalFleetCost, 0) AS FleetCost,
    ISNULL(wc.TotalWarehouseCost, 0) + ISNULL(fc.TotalFleetCost, 0) AS TotalSupplyChainCost,
    wc.AvgDwellTime
FROM DimProductGravity p
LEFT JOIN WarehouseCosts wc ON p.ProductKey = wc.ProductKey
LEFT JOIN FleetCosts fc ON p.ProductKey = fc.ProductKey
WHERE ISNULL(wc.TotalWarehouseCost, 0) + ISNULL(fc.TotalFleetCost, 0) > 0
ORDER BY TotalSupplyChainCost DESC;
```

### Pattern 3: Real-Time Alert Configuration

```sql
-- Create alert for excessive idle time
CREATE PROCEDURE usp_AlertExcessiveFleetIdle
    @ThresholdPercent DECIMAL(5,2) = 15.0
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    SELECT 
        @AlertMessage = STRING_AGG(
            'Vehicle ' + VehicleID + ' idle ' + 
            CAST(IdlePercentage AS VARCHAR) + '% on trip ' + 
            CAST(TripKey AS VARCHAR),
            '; '
        )
    FROM (
        SELECT 
            TripKey,
            VehicleID,
            CAST((IdleTimeMinutes * 100.0 / NULLIF(DurationMinutes, 0)) AS DECIMAL(5,2)) AS IdlePercentage
        FROM FactFleetTrips
        WHERE TimeKey >= (SELECT TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(HOUR, -1, GETDATE()))
            AND (IdleTimeMinutes * 100.0 / NULLIF(DurationMinutes, 0)) > @ThresholdPercent
    ) AlertData;
    
    IF @AlertMessage IS NOT NULL
    BEGIN
        -- Send email via Database Mail
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL_RECIPIENTS}',
            @subject = 'Fleet Idle Time Alert',
            @body = @AlertMessage,
            @importance = 'High';
    END
END;
GO

-- Schedule as SQL Agent job to run every 15 minutes
```

## Configuration Files

### ETL Configuration Template

Create `etl_config.json`:

```json
{
  "dataRefresh": {
    "warehouseOps": {
      "enabled": true,
      "intervalMinutes": 15,
      "sourceProcedure": "usp_LoadWarehouseOperations"
    },
    "fleetTrips": {
      "enabled": true,
      "intervalMinutes": 30,
      "sourceProcedure": "usp_LoadFleetTrips"
    },
    "crossDock": {
      "enabled": true,
      "intervalMinutes": 15,
      "sourceProcedure": "usp_LoadCrossDock"
    }
  },
  "analytics": {
    "gravityZoneUpdate": {
      "enabled": true,
      "scheduleDaily": "02:00",
      "procedure": "usp_CalculateGravityZoneRecommendations"
    },
    "bottleneckDetection": {
      "enabled": true,
      "intervalMinutes": 60,
      "procedure": "usp_DetectBottlenecks",
      "parameters": {
        "thresholdPercentile": 0.90
      }
    }
  },
  "alerts": {
    "fleetIdleThreshold": 15.0,
    "dwellTimeThresholdMinutes": 120,
    "temperatureComplianceRequired": true,
    "notificationChannels": ["email", "teams"]
  }
}
```

### Row-Level Security Setup

```sql
-- Create security roles
CREATE ROLE WarehouseManager;
CREATE ROLE FleetManager;
CREATE ROLE ExecutiveView;

-- Create RLS function for warehouse data
CREATE FUNCTION dbo.fn_WarehouseSecurityPredicate(@LocationName VARCHAR(200))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS result
WHERE @LocationName IN (
    SELECT LocationName 
    FROM dbo.UserLocationAccess 
    WHERE Username = USER_NAME()
)
OR IS_MEMBER('ExecutiveView') = 1;
GO

-- Apply security policy
CREATE SECURITY POLICY WarehouseSecurityPolicy
ADD FILTER PREDICATE dbo.fn_WarehouseSecurityPredicate(LocationName)
ON dbo.vw_DailyWarehouseSummary
WITH (STATE = ON);
GO
```

## Troubleshooting

### Issue: Slow Query Performance

**Symptom**: Cross-fact queries taking >5 seconds

**Solution**: Create covering indexes and partition large fact tables

```sql
-- Create columnstore index for analytical queries
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactWarehouseOps_Analytics
ON FactWarehouseOperations (
    TimeKey, GeographyKey, ProductKey, OperationType,
    QuantityHandled, DwellTimeMinutes, CycleTimeSeconds
);

-- Partition fact table by date range
CREATE PARTITION FUNCTION pf_TimeRange (INT)
AS RANGE RIGHT FOR VALUES (
    20260101, 20260201, 20260301, 20260401, 
    20260501, 20260601, 20260701, 20260801,
    20260901, 20261001, 20261101, 20261201
);
GO

CREATE PARTITION SCHEME ps_TimeRange
AS PARTITION pf_TimeRange
ALL TO ([PRIMARY]);
GO

-- Recreate fact table on partition scheme (example for new table)
CREATE TABLE FactWarehouseOperations_Partitioned (
    OperationKey BIGINT IDENTITY(1,1),
    TimeKey INT,
    -- ... other columns
    CONSTRAINT PK_FactWarehouseOps_Part PRIMARY KEY (TimeKey, OperationKey)
) ON ps_TimeRange(TimeKey);
```

### Issue: Power BI Refresh Failures

**Symptom**: "Data source connection error" in Power BI Service

**Solution**: Configure gateway and credentials

1. Install Power BI Gateway on server with SQL access
2. Configure data source in gateway:
```powershell
# PowerShell commands for gateway configuration
Set-DataGatewayCluster -GatewayClusterId $gatewayId
Add-PowerBIDataSource -GatewayClusterId $gatewayId `
    -DataSourceType "SQL" `
    -ConnectionDetails @{
        server = $env:SQL_SERVER_HOST
        database = "LogiFleetPulse"
    } `
    -CredentialType "Windows"
```

3. Update dataset credentials in Power BI Service

### Issue: Missing External Data

**Symptom**: NULL values in telematics or weather data columns

**Solution**: Implement fallback logic in stored procedures

```sql
-- Enhanced load procedure with fallback
ALTER PROCEDURE usp_LoadFleetTrips
    @LoadDate DATE
AS
BEGIN
    -- Check if external source is available
    DECLARE @SourceAvailable BIT = 0;
    
    BEGIN TRY
        SELECT TOP 1 @SourceAvailable = 1
        FROM ExternalTelematics.dbo.TripData;
    END TRY
    BEGIN CATCH
        SET @SourceAvailable = 0;
        
        -- Log error to monitoring table
        INSERT INTO ETL.ErrorLog (ProcedureName, ErrorMessage, ErrorTime)
        VALUES ('usp_LoadFleetTrips', ERROR_MESSAGE(), GETDATE());
    END CATCH;
    
    IF @SourceAvailable = 0
    BEGIN
        -- Use cached/historical data or skip load
        RAISERROR('External telematics source unavailable', 16, 1);
        RETURN;
    END;
    
    -- Proceed with normal load
    -- ... existing insert logic
END;
```

### Issue: Gravity Score Calculation Errors

**Symptom**: Products with incorrect gravity zone assignments

**Solution**: Recalculate with updated business rules

```sql
-- Diagnostic query to identify misclassified products
SELECT 
    p.SKU,
    p.GravityScore AS CurrentScore,
    COUNT(wo.OperationKey) AS RecentPickCount,
    AVG(wo.CycleTimeSeconds) AS AvgCycleTime,
    CASE 
        WHEN COUNT(wo.OperationKey) > 100 AND AVG(wo.CycleTimeSeconds) < 30 THEN 'Should be High-Gravity'
        WHEN COUNT(wo.OperationKey) < 10 AND AVG(wo.CycleTimeSeconds) > 120 THEN 'Should be Low-Gravity'
        ELSE 'Correctly Classified'
    END AS RecommendedChange
FROM DimProductGravity p
LEFT JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
    AND wo.OperationType = 'Picking'
    AND wo.TimeKey >= (SELECT TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(DAY, -30, GETDATE()))
