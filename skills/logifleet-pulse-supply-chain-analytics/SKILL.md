---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server star schema and Power BI dashboard system for logistics, warehouse, and fleet management analytics
triggers:
  - "set up LogiFleet Pulse analytics"
  - "create supply chain data warehouse"
  - "build logistics dashboard with Power BI"
  - "implement warehouse gravity zone analytics"
  - "configure fleet telemetry tracking"
  - "deploy multi-fact star schema for logistics"
  - "integrate warehouse and fleet KPIs"
  - "setup cross-modal supply chain reporting"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a multi-modal logistics intelligence engine combining MS SQL Server data warehousing with Power BI visualization. It provides a unified semantic layer across warehouse operations, fleet telemetry, inventory management, and external data sources through a custom multi-fact star schema architecture.

**Key capabilities:**
- Cross-fact KPI harmonization (warehouse + fleet + inventory)
- Time-phased dimension tables (15-minute granularity)
- Warehouse Gravity Zones™ spatial optimization
- Predictive bottleneck detection
- Real-time dashboard updates
- Role-based access control

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Network access to warehouse/fleet data sources

### Database Setup

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**
```sql
-- Connect to your SQL Server instance via SSMS
-- Create database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Run the schema creation script
-- Execute schema/01_create_tables.sql
-- Execute schema/02_create_relationships.sql
-- Execute schema/03_create_indexes.sql
-- Execute schema/04_create_stored_procedures.sql
```

3. **Configure data sources:**
```bash
# Copy sample configuration
cp config_sample.json config.json

# Edit with your connection strings
# Use environment variables for sensitive data
```

### Power BI Setup

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. Enter SQL Server connection details when prompted
3. Configure refresh schedules in Power BI Service (if publishing)

## Core Schema Architecture

### Fact Tables

**FactWarehouseOperations** - Warehouse activity grain at operation level:
```sql
CREATE TABLE FactWarehouseOperations (
    OperationID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseZoneKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'PUTAWAY', 'PICK', 'PACK', 'SHIP'
    DwellTimeMinutes INT,
    QuantityHandled DECIMAL(18,2),
    OperatorID INT,
    OperationDurationSeconds INT,
    RecordedTimestamp DATETIME2,
    CONSTRAINT FK_WarehouseTime FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_Zone FOREIGN KEY (WarehouseZoneKey) REFERENCES DimWarehouseZone(ZoneKey)
);

-- Columnstore index for analytical queries
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactWarehouse_Analytics
ON FactWarehouseOperations (TimeKey, ProductKey, WarehouseZoneKey, QuantityHandled, DwellTimeMinutes);
```

**FactFleetTrips** - Fleet telemetry at trip segment level:
```sql
CREATE TABLE FactFleetTrips (
    TripSegmentID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT NOT NULL,
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    AverageSpeedKPH DECIMAL(5,2),
    LoadWeightKG DECIMAL(10,2),
    SegmentStartTimestamp DATETIME2,
    SegmentEndTimestamp DATETIME2,
    WeatherCondition VARCHAR(50),
    CONSTRAINT FK_FleetTime FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_Route FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey)
);

CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactFleet_Analytics
ON FactFleetTrips (TimeKey, VehicleKey, RouteKey, FuelConsumedLiters, IdleTimeMinutes);
```

**FactCrossDock** - Cross-dock transfers:
```sql
CREATE TABLE FactCrossDock (
    CrossDockID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundShipmentKey INT,
    OutboundShipmentKey INT,
    QuantityTransferred DECIMAL(18,2),
    DwellTimeMinutes INT, -- Time between inbound receipt and outbound dispatch
    TransferTimestamp DATETIME2,
    CONSTRAINT FK_CrossDockTime FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey)
);
```

### Dimension Tables

**DimTime** - Time-phased at 15-minute granularity:
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2,
    Year INT,
    Quarter INT,
    Month INT,
    Week INT,
    Day INT,
    Hour INT,
    Minute15Bucket INT, -- 0, 15, 30, 45
    DayOfWeek VARCHAR(10),
    IsWeekend BIT,
    FiscalPeriod INT,
    ShiftType VARCHAR(20) -- 'MORNING', 'AFTERNOON', 'NIGHT'
);

-- Populate with 15-minute intervals
CREATE PROCEDURE sp_PopulateTimeDimension
    @StartDate DATETIME2,
    @EndDate DATETIME2
AS
BEGIN
    DECLARE @CurrentDateTime DATETIME2 = @StartDate;
    
    WHILE @CurrentDateTime <= @EndDate
    BEGIN
        INSERT INTO DimTime (TimeKey, FullDateTime, Year, Quarter, Month, Week, Day, Hour, Minute15Bucket, DayOfWeek, IsWeekend)
        VALUES (
            CONVERT(INT, FORMAT(@CurrentDateTime, 'yyyyMMddHHmm')),
            @CurrentDateTime,
            YEAR(@CurrentDateTime),
            DATEPART(QUARTER, @CurrentDateTime),
            MONTH(@CurrentDateTime),
            DATEPART(WEEK, @CurrentDateTime),
            DAY(@CurrentDateTime),
            DATEPART(HOUR, @CurrentDateTime),
            (DATEPART(MINUTE, @CurrentDateTime) / 15) * 15,
            DATENAME(WEEKDAY, @CurrentDateTime),
            CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1, 7) THEN 1 ELSE 0 END
        );
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END;
```

**DimProductGravity** - Products with calculated gravity scores:
```sql
CREATE TABLE DimProduct (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    ProductSKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName VARCHAR(255),
    CategoryL1 VARCHAR(100),
    CategoryL2 VARCHAR(100),
    UnitWeight DECIMAL(10,2),
    IsFragile BIT,
    -- Gravity calculation fields
    AvgDailyVelocity DECIMAL(10,2), -- Updated by nightly ETL
    UnitValue DECIMAL(10,2),
    GravityScore AS (
        (AvgDailyVelocity * 0.5) + 
        (UnitValue * 0.3) + 
        (CASE WHEN IsFragile = 1 THEN 20 ELSE 0 END)
    ) PERSISTED,
    GravityZone AS (
        CASE 
            WHEN (AvgDailyVelocity * 0.5 + UnitValue * 0.3 + CASE WHEN IsFragile = 1 THEN 20 ELSE 0 END) > 80 THEN 'HIGH'
            WHEN (AvgDailyVelocity * 0.5 + UnitValue * 0.3 + CASE WHEN IsFragile = 1 THEN 20 ELSE 0 END) > 40 THEN 'MEDIUM'
            ELSE 'LOW'
        END
    ) PERSISTED
);
```

**DimWarehouseZone** - Spatial warehouse organization:
```sql
CREATE TABLE DimWarehouseZone (
    ZoneKey INT PRIMARY KEY IDENTITY(1,1),
    ZoneID VARCHAR(50) UNIQUE NOT NULL,
    ZoneName VARCHAR(100),
    WarehouseID VARCHAR(50),
    ZoneType VARCHAR(50), -- 'RECEIVING', 'STORAGE', 'PICKING', 'SHIPPING', 'CROSS_DOCK'
    OptimalGravityZone VARCHAR(20), -- 'HIGH', 'MEDIUM', 'LOW'
    DistanceFromShippingMeters DECIMAL(10,2),
    CapacityPallets INT,
    TemperatureControlled BIT
);
```

## Key Stored Procedures

### Incremental Data Loading

```sql
CREATE PROCEDURE sp_IncrementalLoadWarehouseOps
    @LastLoadTimestamp DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new warehouse operations since last load
    INSERT INTO FactWarehouseOperations (
        TimeKey, 
        ProductKey, 
        WarehouseZoneKey, 
        OperationType, 
        DwellTimeMinutes,
        QuantityHandled,
        OperatorID,
        OperationDurationSeconds,
        RecordedTimestamp
    )
    SELECT 
        CONVERT(INT, FORMAT(wms.OperationTimestamp, 'yyyyMMddHHmm')) / 15 * 15 AS TimeKey, -- Round to 15-min bucket
        p.ProductKey,
        z.ZoneKey,
        wms.OperationType,
        wms.DwellTimeMinutes,
        wms.Quantity,
        wms.OperatorID,
        wms.DurationSeconds,
        wms.OperationTimestamp
    FROM EXTERNAL_WMS.Operations wms
    INNER JOIN DimProduct p ON wms.SKU = p.ProductSKU
    INNER JOIN DimWarehouseZone z ON wms.ZoneID = z.ZoneID
    WHERE wms.OperationTimestamp > @LastLoadTimestamp
        AND wms.OperationTimestamp <= GETDATE();
    
    -- Update last load timestamp
    UPDATE ETL_Metadata 
    SET LastLoadTimestamp = GETDATE() 
    WHERE TableName = 'FactWarehouseOperations';
END;
```

### Cross-Fact KPI Calculation

```sql
CREATE PROCEDURE sp_CalculateCrossDockEfficiency
    @StartDate DATETIME2,
    @EndDate DATETIME2
AS
BEGIN
    -- Returns cross-dock efficiency with integrated fleet impact
    SELECT 
        cd.ProductKey,
        p.ProductSKU,
        p.ProductName,
        p.GravityZone,
        COUNT(DISTINCT cd.CrossDockID) AS TotalTransfers,
        AVG(cd.DwellTimeMinutes) AS AvgCrossDockDwellMinutes,
        -- Join with fleet data to see impact
        AVG(ft.IdleTimeMinutes) AS AvgFleetIdleAtPickup,
        -- Efficiency score
        CASE 
            WHEN AVG(cd.DwellTimeMinutes) < 30 THEN 'EXCELLENT'
            WHEN AVG(cd.DwellTimeMinutes) < 60 THEN 'GOOD'
            WHEN AVG(cd.DwellTimeMinutes) < 120 THEN 'FAIR'
            ELSE 'POOR'
        END AS EfficiencyRating
    FROM FactCrossDock cd
    INNER JOIN DimProduct p ON cd.ProductKey = p.ProductKey
    LEFT JOIN FactFleetTrips ft 
        ON ft.TimeKey >= cd.TimeKey 
        AND ft.TimeKey <= cd.TimeKey + 100 -- Within ~1.5 hours
    WHERE cd.TransferTimestamp BETWEEN @StartDate AND @EndDate
    GROUP BY cd.ProductKey, p.ProductSKU, p.ProductName, p.GravityZone
    ORDER BY AvgCrossDockDwellMinutes DESC;
END;
```

### Automated Alerting

```sql
CREATE PROCEDURE sp_CheckFleetIdleAlerts
AS
BEGIN
    -- Check for excessive fleet idle time and send alerts
    DECLARE @ThresholdPercent DECIMAL(5,2) = 15.0;
    
    SELECT 
        v.VehicleID,
        v.VehiclePlateNumber,
        d.DriverName,
        SUM(ft.IdleTimeMinutes) AS TotalIdleMinutes,
        SUM(DATEDIFF(MINUTE, ft.SegmentStartTimestamp, ft.SegmentEndTimestamp)) AS TotalTripMinutes,
        (SUM(ft.IdleTimeMinutes) * 100.0 / NULLIF(SUM(DATEDIFF(MINUTE, ft.SegmentStartTimestamp, ft.SegmentEndTimestamp)), 0)) AS IdlePercent
    FROM FactFleetTrips ft
    INNER JOIN DimVehicle v ON ft.VehicleKey = v.VehicleKey
    INNER JOIN DimDriver d ON ft.DriverKey = d.DriverKey
    WHERE ft.SegmentStartTimestamp >= DATEADD(DAY, -1, GETDATE())
    GROUP BY v.VehicleID, v.VehiclePlateNumber, d.DriverName
    HAVING (SUM(ft.IdleTimeMinutes) * 100.0 / NULLIF(SUM(DATEDIFF(MINUTE, ft.SegmentStartTimestamp, ft.SegmentEndTimestamp)), 0)) > @ThresholdPercent;
    
    -- Insert alert records (integrate with email/Teams via external service)
    INSERT INTO AlertLog (AlertType, VehicleID, Message, Timestamp)
    SELECT 
        'EXCESSIVE_IDLE',
        VehicleID,
        'Vehicle ' + VehiclePlateNumber + ' exceeded idle threshold: ' + CAST(IdlePercent AS VARCHAR) + '%',
        GETDATE()
    FROM (
        -- Repeat query logic here or use temp table
    ) AS Alerts;
END;
```

## Power BI Integration

### DAX Measures for Cross-Fact Analysis

**Warehouse Gravity Alignment Score:**
```dax
Gravity_Alignment_Score = 
VAR CurrentZoneGravity = SELECTEDVALUE(DimWarehouseZone[OptimalGravityZone])
VAR ProductGravity = SELECTEDVALUE(DimProduct[GravityZone])
RETURN
IF(
    CurrentZoneGravity = ProductGravity,
    100,
    IF(
        (CurrentZoneGravity = "HIGH" && ProductGravity = "MEDIUM") || 
        (CurrentZoneGravity = "MEDIUM" && ProductGravity = "HIGH"),
        60,
        20
    )
)
```

**Fleet Efficiency Composite:**
```dax
Fleet_Efficiency_Index = 
VAR AvgIdlePercent = 
    DIVIDE(
        SUM(FactFleetTrips[IdleTimeMinutes]),
        SUM(FactFleetTrips[IdleTimeMinutes]) + SUM(FactFleetTrips[DistanceKM]) / AVERAGE(FactFleetTrips[AverageSpeedKPH]) * 60
    ) * 100

VAR FuelEfficiency = 
    DIVIDE(
        SUM(FactFleetTrips[DistanceKM]),
        SUM(FactFleetTrips[FuelConsumedLiters])
    )

VAR NormalizedIdle = (15 - AvgIdlePercent) / 15 * 50 // 50% weight, ideal < 15%
VAR NormalizedFuel = (FuelEfficiency / 10) * 50 // 50% weight, target 10 km/L

RETURN NormalizedIdle + NormalizedFuel
```

**Cross-Dock to Fleet Impact:**
```dax
CrossDock_Fleet_Impact = 
CALCULATE(
    AVERAGE(FactFleetTrips[IdleTimeMinutes]),
    FILTER(
        FactFleetTrips,
        FactFleetTrips[TimeKey] >= EARLIER(FactCrossDock[TimeKey]) &&
        FactFleetTrips[TimeKey] <= EARLIER(FactCrossDock[TimeKey]) + 100
    )
)
```

### Report Structure

**Page 1: Executive Dashboard**
- KPI cards: Total shipments, avg delivery time, fleet utilization, warehouse capacity
- Time series: Daily throughput trends
- Map visual: Route density with idle time heat overlay

**Page 2: Warehouse Gravity Analysis**
- Matrix: Products by current zone vs. optimal gravity zone
- Scatter plot: Gravity score vs. dwell time
- Recommendations table: Zone reassignments

**Page 3: Fleet Optimization**
- Gantt chart: Vehicle schedules with idle time highlighted
- Fuel consumption by route
- Maintenance priority queue

**Page 4: Cross-Modal Insights**
- Combined warehouse dwell time + fleet delay correlation
- SKU-level analysis: warehouse velocity → delivery performance
- Weather/traffic impact overlay

## Configuration

### Connection String Setup

**config.json structure:**
```json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "Active Directory Integrated"
  },
  "data_sources": {
    "wms_api": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "api_key": "${WMS_API_KEY}"
    },
    "telematics": {
      "endpoint": "${TELEMATICS_ENDPOINT}",
      "api_key": "${TELEMATICS_API_KEY}"
    },
    "weather_api": {
      "endpoint": "https://api.weatherapi.com/v1",
      "api_key": "${WEATHER_API_KEY}"
    }
  },
  "refresh_schedule": {
    "warehouse_ops": "*/15 * * * *",
    "fleet_trips": "*/15 * * * *",
    "dimensions": "0 2 * * *"
  }
}
```

### Role-Based Security

**SQL Server row-level security:**
```sql
CREATE SCHEMA Security;
GO

CREATE FUNCTION Security.fn_WarehouseSecurityPredicate(@WarehouseID VARCHAR(50))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN 
    SELECT 1 AS result
    WHERE @WarehouseID IN (
        SELECT WarehouseID 
        FROM dbo.UserWarehouseAccess 
        WHERE UserName = USER_NAME()
    );
GO

CREATE SECURITY POLICY WarehouseSecurityPolicy
ADD FILTER PREDICATE Security.fn_WarehouseSecurityPredicate(WarehouseID)
ON dbo.DimWarehouseZone
WITH (STATE = ON);
```

**Power BI role definition:**
```dax
// Role: Regional Manager - West
[WarehouseID] IN {"WH-WEST-01", "WH-WEST-02", "WH-WEST-03"}
```

## Common Patterns

### Pattern 1: Warehouse Rebalancing Recommendation

```sql
-- Identify products in wrong gravity zones
WITH ProductGravityMismatch AS (
    SELECT 
        p.ProductKey,
        p.ProductSKU,
        p.ProductName,
        p.GravityZone AS OptimalGravity,
        z.ZoneID,
        z.ZoneName,
        z.OptimalGravityZone AS CurrentZoneGravity,
        AVG(wo.DwellTimeMinutes) AS AvgDwellMinutes,
        SUM(wo.QuantityHandled) AS TotalVolume
    FROM FactWarehouseOperations wo
    INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
    INNER JOIN DimWarehouseZone z ON wo.WarehouseZoneKey = z.ZoneKey
    WHERE wo.RecordedTimestamp >= DATEADD(DAY, -30, GETDATE())
    GROUP BY p.ProductKey, p.ProductSKU, p.ProductName, p.GravityZone, z.ZoneID, z.ZoneName, z.OptimalGravityZone
    HAVING p.GravityZone <> z.OptimalGravityZone
)
SELECT 
    ProductSKU,
    ProductName,
    CurrentZoneGravity,
    OptimalGravity,
    AvgDwellMinutes,
    TotalVolume,
    -- Suggest target zone
    (SELECT TOP 1 ZoneID 
     FROM DimWarehouseZone 
     WHERE OptimalGravityZone = pgm.OptimalGravity 
     ORDER BY DistanceFromShippingMeters ASC) AS RecommendedZoneID
FROM ProductGravityMismatch pgm
WHERE TotalVolume > 100 -- Only high-volume SKUs
ORDER BY AvgDwellMinutes DESC;
```

### Pattern 2: Predictive Maintenance Scoring

```sql
-- Fleet vehicles requiring proactive maintenance
WITH FleetRiskScoring AS (
    SELECT 
        v.VehicleKey,
        v.VehicleID,
        v.MaintenanceIntervalKM,
        v.CurrentOdometerKM,
        -- Calculate risk factors
        (v.CurrentOdometerKM * 1.0 / v.MaintenanceIntervalKM) AS MaintenanceProximityRatio,
        AVG(ft.IdleTimeMinutes) AS AvgIdleMinutes,
        AVG(ft.LoadWeightKG) AS AvgLoadWeight,
        COUNT(DISTINCT ft.RouteKey) AS UniqueRoutes,
        -- Revenue impact: vehicles carrying high-value goods
        AVG(p.UnitValue * ft.LoadWeightKG) AS AvgCargoValue
    FROM DimVehicle v
    INNER JOIN FactFleetTrips ft ON v.VehicleKey = ft.VehicleKey
    INNER JOIN (
        -- Link trips to products via shipments (requires bridge table in full schema)
        SELECT ft2.TripSegmentID, p2.UnitValue
        FROM FactFleetTrips ft2
        -- Simplified: assume product key linkage
        CROSS APPLY (SELECT TOP 1 UnitValue FROM DimProduct) p2
    ) p ON ft.TripSegmentID = p.TripSegmentID
    WHERE ft.SegmentStartTimestamp >= DATEADD(DAY, -60, GETDATE())
    GROUP BY v.VehicleKey, v.VehicleID, v.MaintenanceIntervalKM, v.CurrentOdometerKM
)
SELECT 
    VehicleID,
    MaintenanceProximityRatio,
    AvgIdleMinutes,
    AvgCargoValue,
    -- Composite risk score
    (MaintenanceProximityRatio * 40) + 
    ((AvgIdleMinutes / 60.0) * 30) + 
    ((AvgCargoValue / 100000.0) * 30) AS RiskScore,
    CASE 
        WHEN ((MaintenanceProximityRatio * 40) + ((AvgIdleMinutes / 60.0) * 30) + ((AvgCargoValue / 100000.0) * 30)) > 75 THEN 'URGENT'
        WHEN ((MaintenanceProximityRatio * 40) + ((AvgIdleMinutes / 60.0) * 30) + ((AvgCargoValue / 100000.0) * 30)) > 50 THEN 'HIGH'
        ELSE 'NORMAL'
    END AS MaintenancePriority
FROM FleetRiskScoring
ORDER BY RiskScore DESC;
```

### Pattern 3: Time-Phased Demand Forecasting

```sql
-- Predict warehouse throughput for next week based on historical patterns
WITH HistoricalPatterns AS (
    SELECT 
        t.DayOfWeek,
        t.Hour,
        AVG(wo.QuantityHandled) AS AvgQuantity,
        STDEV(wo.QuantityHandled) AS StdDevQuantity
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE wo.RecordedTimestamp >= DATEADD(DAY, -90, GETDATE())
        AND wo.RecordedTimestamp < DATEADD(DAY, -7, GETDATE()) -- Exclude last week
    GROUP BY t.DayOfWeek, t.Hour
),
NextWeekForecast AS (
    SELECT 
        DATEADD(DAY, n, DATEADD(DAY, DATEDIFF(DAY, 0, GETDATE()), 0)) AS ForecastDate,
        DATENAME(WEEKDAY, DATEADD(DAY, n, GETDATE())) AS DayOfWeek,
        h AS Hour
    FROM (VALUES (0),(1),(2),(3),(4),(5),(6)) AS Days(n)
    CROSS JOIN (SELECT TOP 24 ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1 AS h FROM sys.objects) AS Hours
)
SELECT 
    nwf.ForecastDate,
    nwf.DayOfWeek,
    nwf.Hour,
    ISNULL(hp.AvgQuantity, 0) AS PredictedQuantity,
    ISNULL(hp.StdDevQuantity, 0) AS Uncertainty,
    -- Confidence interval
    ISNULL(hp.AvgQuantity - (1.96 * hp.StdDevQuantity), 0) AS LowerBound95,
    ISNULL(hp.AvgQuantity + (1.96 * hp.StdDevQuantity), 0) AS UpperBound95
FROM NextWeekForecast nwf
LEFT JOIN HistoricalPatterns hp 
    ON nwf.DayOfWeek = hp.DayOfWeek 
    AND nwf.Hour = hp.Hour
ORDER BY nwf.ForecastDate, nwf.Hour;
```

## Troubleshooting

### Issue: Slow Cross-Fact Queries

**Symptom:** Queries joining warehouse and fleet facts take > 30 seconds

**Solution:**
```sql
-- Ensure columnstore indexes exist
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactWarehouse_Analytics
ON FactWarehouseOperations (TimeKey, ProductKey, WarehouseZoneKey);

CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactFleet_Analytics
ON FactFleetTrips (TimeKey, VehicleKey, RouteKey);

-- Partition large fact tables by time
ALTER PARTITION SCHEME PS_LogiFleetByMonth NEXT USED [PRIMARY];
ALTER PARTITION FUNCTION PF_LogiFleetByMonth() SPLIT RANGE ('2026-08-01');

-- Update statistics
UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS FactFleetTrips WITH FULLSCAN;
```

### Issue: Gravity Scores Not Updating

**Symptom:** `DimProduct.GravityScore` shows stale data

**Solution:**
Gravity scores are computed columns relying on `AvgDailyVelocity`. Ensure nightly ETL runs:

```sql
CREATE PROCEDURE sp_UpdateProductVelocity
AS
BEGIN
    UPDATE p
    SET AvgDailyVelocity = v.DailyAvg
    FROM DimProduct p
    INNER JOIN (
        SELECT 
            ProductKey,
            AVG(DailyQty) AS DailyAvg
        FROM (
            SELECT 
                ProductKey,
                CAST(RecordedTimestamp AS DATE) AS OperationDate,
                SUM(QuantityHandled) AS DailyQty
            FROM FactWarehouseOperations
            WHERE RecordedTimestamp >= DATEADD(DAY, -30, GETDATE())
            GROUP BY ProductKey, CAST(RecordedTimestamp AS DATE)
        ) AS DailyAgg
        GROUP BY ProductKey
    ) v ON p.ProductKey = v.ProductKey;
END;

-- Schedule via SQL Agent job daily at 2 AM
```

### Issue: Power BI Dashboard Not Refreshing

**Symptom:** Data in Power BI is hours old despite 15-minute refresh schedule

**Solution:**
1. Check Power BI Service gateway status
2. Verify SQL Server network connectivity
3. Review refresh history in Power BI Service:
   - Settings → Datasets → LogiFleet Pulse → Refresh history
4. Ensure service account has db_datareader permissions:
```sql
CREATE USER [PowerBIServiceAccount] FROM LOGIN [PowerBIServiceAccount];
ALTER ROLE db_datareader ADD MEMBER [PowerBIServiceAccount];
```

### Issue: Role-Based Security Not Filtering

**Symptom:** Users see data from warehouses they shouldn't access

**Solution:**
1. Verify security policy is enabled:
```sql
SELECT * FROM sys.security_policies WHERE name = 'WarehouseSecurityPolicy';
-- Ensure is_enabled = 1
```

2. Check UserWarehouseAccess table is populated:
```sql
SELECT * FROM dbo.UserWarehouseAccess WHERE UserName = 'domain\username';
```

3. Test with specific user context:
```sql
EXECUTE AS USER = 'domain\username';
SELECT * FROM DimWarehouseZone; -- Should only show authorized warehouses
REVERT;
```

### Issue: External Data Sources Not Loading

**Symptom:** `sp_IncrementalLoadWarehouseOps` fails with "Invalid object name 'EXTERNAL_WMS.Operations'"

**Solution:**
Configure external table using PolyBase:

```sql
-- Create master key (one time)
CREATE MASTER KEY ENCRYPTION BY PASSWORD = '${MASTER_KEY_PASSWORD}';

-- Create database scoped credential
CREATE DATABASE SCOPED CREDENTIAL WMSCredential
WITH IDENTITY = '${WMS_DB_USERNAME}',
SECRET = '${WMS_DB_PASSWORD}';

-- Create external data source
CREATE EXTERNAL DATA SOURCE WMS_DataSource
WITH (
    TYPE = RDBMS,
    LOCATION = '${WMS_SQL_SERVER}',
    
