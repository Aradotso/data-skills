---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehouse for multi-modal logistics intelligence with warehouse operations, fleet tracking, and predictive analytics
triggers:
  - set up logistics analytics dashboard
  - create supply chain data warehouse
  - build fleet management Power BI reports
  - implement warehouse operations analytics
  - configure cross-fact logistics KPIs
  - deploy logifleet pulse analytics
  - design star schema for logistics data
  - integrate warehouse and fleet telemetry
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a **multi-modal logistics intelligence engine** that combines:
- Warehouse management operations (receiving, putaway, picking, packing, shipping)
- Fleet telemetry and GPS tracking (routes, fuel consumption, idle time)
- Inventory management and aging analysis
- Supplier reliability metrics
- External data integration (weather, traffic APIs)

Built on **MS SQL Server** with **Power BI** dashboards, it provides a unified semantic layer using a multi-fact star schema architecture with time-phased dimensions for cross-functional logistics analytics.

## Installation & Setup

### Prerequisites

- **MS SQL Server 2019+** (Express, Standard, or Enterprise)
- **Power BI Desktop** (latest version)
- **SQL Server Management Studio (SSMS)** or Azure Data Studio
- Network access to data sources (WMS, TMS, telemetry APIs)

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Create new database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create dimension tables first
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    DateTime DATETIME NOT NULL,
    DateKey INT NOT NULL,
    TimeSlot VARCHAR(20), -- e.g., "09:00-09:15"
    HourOfDay INT,
    DayOfWeek VARCHAR(10),
    DayOfMonth INT,
    WeekOfYear INT,
    MonthName VARCHAR(20),
    Quarter INT,
    FiscalYear INT,
    IsWeekend BIT,
    IsHoliday BIT
);
GO

CREATE UNIQUE INDEX IX_DimTime_DateTime ON DimTime(DateTime);
GO

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationCode VARCHAR(50) NOT NULL,
    LocationName VARCHAR(200),
    LocationType VARCHAR(50), -- 'Warehouse', 'Route Node', 'Supplier'
    Address VARCHAR(500),
    City VARCHAR(100),
    Region VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    TimeZone VARCHAR(50),
    IsActive BIT DEFAULT 1
);
GO

CREATE TABLE DimProduct (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) NOT NULL UNIQUE,
    ProductName VARCHAR(500),
    Category VARCHAR(100),
    Subcategory VARCHAR(100),
    UnitOfMeasure VARCHAR(20),
    Weight DECIMAL(10,2),
    Volume DECIMAL(10,2),
    IsFragile BIT,
    IsPerishable BIT,
    TemperatureControl VARCHAR(50), -- 'Ambient', 'Refrigerated', 'Frozen'
    UnitCost DECIMAL(18,2),
    RetailPrice DECIMAL(18,2),
    -- Warehouse Gravity Zone attributes
    PickFrequencyScore INT, -- 1-100
    ValueScore INT, -- 1-100
    FragilityScore INT, -- 1-100
    GravityZone VARCHAR(20), -- 'High', 'Medium', 'Low'
    OptimalStorageZone VARCHAR(50)
);
GO

CREATE TABLE DimSupplier (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierCode VARCHAR(50) NOT NULL UNIQUE,
    SupplierName VARCHAR(200),
    SupplierType VARCHAR(50),
    LeadTimeDays INT,
    LeadTimeVariance DECIMAL(5,2),
    DefectPercentage DECIMAL(5,2),
    OnTimeDeliveryPercentage DECIMAL(5,2),
    ComplianceScore INT, -- 1-100
    ReliabilityTier VARCHAR(20) -- 'Tier 1', 'Tier 2', 'Tier 3'
);
GO

CREATE TABLE DimFleetVehicle (
    VehicleKey INT PRIMARY KEY IDENTITY(1,1),
    VehicleID VARCHAR(50) NOT NULL UNIQUE,
    VehicleType VARCHAR(50), -- 'Box Truck', 'Semi', 'Van'
    Make VARCHAR(50),
    Model VARCHAR(50),
    Year INT,
    CapacityWeight DECIMAL(10,2),
    CapacityVolume DECIMAL(10,2),
    FuelType VARCHAR(20),
    AvgFuelConsumption DECIMAL(5,2),
    LastMaintenanceDate DATE,
    NextMaintenanceDue DATE,
    IsActive BIT DEFAULT 1
);
GO

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    WarehouseKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProduct(ProductKey),
    SupplierKey INT FOREIGN KEY REFERENCES DimSupplier(SupplierKey),
    
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OrderID VARCHAR(100),
    BatchNumber VARCHAR(100),
    
    -- Metrics
    QuantityHandled INT,
    DwellTimeMinutes INT,
    CycleTimeMinutes INT,
    StorageZoneAssigned VARCHAR(50),
    PickPathDistance DECIMAL(10,2),
    LaborHours DECIMAL(6,2),
    ErrorCount INT,
    
    -- Financial
    OperationCost DECIMAL(18,2),
    
    -- Timestamps
    StartDateTime DATETIME,
    EndDateTime DATETIME,
    
    -- Flags
    IsExpedited BIT,
    HasQualityIssue BIT
);
GO

CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey);
GO

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleKey INT NOT NULL FOREIGN KEY REFERENCES DimFleetVehicle(VehicleKey),
    OriginKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    
    TripID VARCHAR(100),
    DriverID VARCHAR(50),
    
    -- Metrics
    DistanceMiles DECIMAL(10,2),
    TripDurationMinutes INT,
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    FuelConsumed DECIMAL(10,2),
    AvgSpeed DECIMAL(5,2),
    MaxSpeed DECIMAL(5,2),
    
    -- Load characteristics
    LoadWeight DECIMAL(10,2),
    LoadVolume DECIMAL(10,2),
    StopCount INT,
    
    -- Financial
    FuelCost DECIMAL(18,2),
    TripRevenue DECIMAL(18,2),
    
    -- Timestamps
    DepartureDateTime DATETIME,
    ArrivalDateTime DATETIME,
    
    -- Flags
    IsDelayed BIT,
    HasMaintenceAlert BIT,
    WeatherImpact BIT,
    TrafficImpact BIT
);
GO

CREATE NONCLUSTERED INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleKey);
GO

-- Bridge table for many-to-many relationships
CREATE TABLE BridgeRouteWarehouse (
    RouteWarehouseKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OperationKey BIGINT FOREIGN KEY REFERENCES FactWarehouseOperations(OperationKey),
    SequenceNumber INT,
    TransferType VARCHAR(50) -- 'Pickup', 'Delivery', 'Cross-Dock'
);
GO
```

### Step 2: Create Helper Stored Procedures

```sql
-- Stored procedure for time dimension population
CREATE PROCEDURE PopulateTimeDimension
    @StartDate DATETIME,
    @EndDate DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentDateTime DATETIME = @StartDate;
    
    WHILE @CurrentDateTime < @EndDate
    BEGIN
        IF NOT EXISTS (SELECT 1 FROM DimTime WHERE DateTime = @CurrentDateTime)
        BEGIN
            INSERT INTO DimTime (
                DateTime, DateKey, TimeSlot, HourOfDay, DayOfWeek,
                DayOfMonth, WeekOfYear, MonthName, Quarter, FiscalYear,
                IsWeekend, IsHoliday
            )
            VALUES (
                @CurrentDateTime,
                CONVERT(INT, FORMAT(@CurrentDateTime, 'yyyyMMdd')),
                FORMAT(@CurrentDateTime, 'HH:mm') + '-' + 
                    FORMAT(DATEADD(MINUTE, 15, @CurrentDateTime), 'HH:mm'),
                DATEPART(HOUR, @CurrentDateTime),
                DATENAME(WEEKDAY, @CurrentDateTime),
                DATEPART(DAY, @CurrentDateTime),
                DATEPART(WEEK, @CurrentDateTime),
                DATENAME(MONTH, @CurrentDateTime),
                DATEPART(QUARTER, @CurrentDateTime),
                YEAR(@CurrentDateTime),
                CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1,7) THEN 1 ELSE 0 END,
                0 -- Update separately with holiday calendar
            );
        END
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END
GO

-- Populate time dimension for 2 years
EXEC PopulateTimeDimension '2026-01-01', '2028-01-01';
GO

-- Stored procedure for automated alerts
CREATE PROCEDURE CheckLogisticsAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Alert: High fleet idling percentage
    SELECT 
        v.VehicleID,
        AVG(f.IdleTimeMinutes * 100.0 / NULLIF(f.TripDurationMinutes, 0)) AS IdlePercentage,
        COUNT(*) AS TripCount
    FROM FactFleetTrips f
    INNER JOIN DimFleetVehicle v ON f.VehicleKey = v.VehicleKey
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.DateTime >= DATEADD(DAY, -7, GETDATE())
    GROUP BY v.VehicleID
    HAVING AVG(f.IdleTimeMinutes * 100.0 / NULLIF(f.TripDurationMinutes, 0)) > 15;
    
    -- Alert: Excessive warehouse dwell time
    SELECT 
        p.SKU,
        p.ProductName,
        g.LocationName AS Warehouse,
        AVG(w.DwellTimeMinutes / 60.0) AS AvgDwellHours,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations w
    INNER JOIN DimProduct p ON w.ProductKey = p.ProductKey
    INNER JOIN DimGeography g ON w.WarehouseKey = g.GeographyKey
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.DateTime >= DATEADD(DAY, -30, GETDATE())
        AND w.OperationType = 'Putaway'
    GROUP BY p.SKU, p.ProductName, g.LocationName
    HAVING AVG(w.DwellTimeMinutes / 60.0) > 72;
END
GO
```

### Step 3: Configure Data Ingestion

```sql
-- Example: External table for warehouse management system integration
-- (Requires SQL Server PolyBase or linked servers)

-- For CSV/flat file ingestion
CREATE PROCEDURE ImportWarehouseOperations
    @FilePath VARCHAR(500)
AS
BEGIN
    -- Staging table
    CREATE TABLE #StagingWarehouse (
        OperationDateTime DATETIME,
        WarehouseCode VARCHAR(50),
        SKU VARCHAR(100),
        OperationType VARCHAR(50),
        Quantity INT,
        DwellMinutes INT,
        CycleMinutes INT
    );
    
    -- BULK INSERT from file
    DECLARE @SQL NVARCHAR(MAX) = 
        'BULK INSERT #StagingWarehouse FROM ''' + @FilePath + '''
        WITH (FIRSTROW = 2, FIELDTERMINATOR = '','', ROWTERMINATOR = ''\n'')';
    
    EXEC sp_executesql @SQL;
    
    -- Transform and load into fact table
    INSERT INTO FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductKey, OperationType,
        QuantityHandled, DwellTimeMinutes, CycleTimeMinutes,
        StartDateTime
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.OperationType,
        s.Quantity,
        s.DwellMinutes,
        s.CycleMinutes,
        s.OperationDateTime
    FROM #StagingWarehouse s
    INNER JOIN DimTime t ON CONVERT(DATETIME, 
        DATEADD(MINUTE, (DATEPART(MINUTE, s.OperationDateTime) / 15) * 15, 
        DATEADD(HOUR, DATEDIFF(HOUR, 0, s.OperationDateTime), 0))) = t.DateTime
    INNER JOIN DimGeography g ON s.WarehouseCode = g.LocationCode
    INNER JOIN DimProduct p ON s.SKU = p.SKU
    WHERE NOT EXISTS (
        SELECT 1 FROM FactWarehouseOperations w2
        WHERE w2.StartDateTime = s.OperationDateTime
            AND w2.ProductKey = p.ProductKey
    );
    
    DROP TABLE #StagingWarehouse;
END
GO
```

### Step 4: Configure Power BI Connection

1. Open **Power BI Desktop**
2. **Get Data** → **SQL Server**
3. Enter server name and database: `LogiFleetPulse`
4. Select tables: All `Fact*` and `Dim*` tables
5. Power BI will auto-detect relationships based on foreign keys

## Power BI Report Configuration

### Creating Cross-Fact KPI Measures

```dax
-- Measure: Average Warehouse Dwell Time
AvgDwellTime = 
AVERAGE(FactWarehouseOperations[DwellTimeMinutes])

-- Measure: Fleet Idle Percentage
FleetIdlePercentage = 
DIVIDE(
    SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[TripDurationMinutes])
) * 100

-- Measure: Cross-Fact Cost Per Unit Delivered
CostPerUnitDelivered = 
DIVIDE(
    SUM(FactFleetTrips[FuelCost]) + SUM(FactWarehouseOperations[OperationCost]),
    SUM(FactWarehouseOperations[QuantityHandled])
)

-- Measure: Warehouse-Fleet Efficiency Index
EfficiencyIndex = 
VAR AvgCycleTime = AVERAGE(FactWarehouseOperations[CycleTimeMinutes])
VAR AvgTripTime = AVERAGE(FactFleetTrips[TripDurationMinutes])
VAR IdlePct = [FleetIdlePercentage]
RETURN
    (100 - IdlePct) * (1 / (AvgCycleTime + AvgTripTime)) * 1000

-- Measure: Gravity Zone Compliance
GravityZoneCompliance = 
VAR HighGravityProducts = 
    FILTER(
        DimProduct,
        DimProduct[GravityZone] = "High"
    )
VAR CorrectlyPlaced = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[StorageZoneAssigned] = DimProduct[OptimalStorageZone]
    )
VAR Total = COUNTROWS(FactWarehouseOperations)
RETURN
    DIVIDE(CorrectlyPlaced, Total) * 100

-- Time Intelligence: Month-over-Month Change
MoM_DwellTime = 
VAR CurrentMonth = [AvgDwellTime]
VAR PreviousMonth = 
    CALCULATE(
        [AvgDwellTime],
        DATEADD(DimTime[DateTime], -1, MONTH)
    )
RETURN
    DIVIDE(CurrentMonth - PreviousMonth, PreviousMonth) * 100
```

### Dashboard Layout Configuration

**Page 1: Executive Overview**
- KPI Cards: Total Operations, Fleet Utilization, Avg Dwell Time, Cost per Unit
- Line Chart: Daily operations volume (Warehouse + Fleet)
- Donut Chart: Operations by Type
- Map: Geographic distribution of warehouses and routes

**Page 2: Warehouse Operations**
- Matrix: Operations by Product Category × Time
- Bar Chart: Top 10 products by dwell time
- Scatter Plot: Cycle Time vs Quantity Handled (colored by gravity zone)
- Table: Recent quality issues with drill-through to details

**Page 3: Fleet Performance**
- Gauge: Fleet idle percentage vs target (15%)
- Column Chart: Fuel consumption by vehicle
- Line + Clustered Column: Distance vs Duration by route
- Heatmap: Delays by day of week × hour of day

**Page 4: Cross-Modal Analytics**
- Scatter Plot: Warehouse dwell time (X) vs Fleet idle time (Y), sized by total cost
- Waterfall Chart: Cost breakdown (warehouse labor + fleet fuel + overhead)
- Custom Visual: Sankey diagram showing product flow from supplier → warehouse → route → customer

### Row-Level Security Setup

```dax
-- Create role: Regional Manager
-- Filter: DimGeography[Region] = USERNAME()

-- Create role: Warehouse Supervisor
-- Filter: DimGeography[LocationCode] = LOOKUPVALUE(
    UserWarehouseMapping[WarehouseCode],
    UserWarehouseMapping[Username],
    USERNAME()
)
```

## Common Usage Patterns

### Pattern 1: Identify Products for Gravity Zone Reassignment

```sql
-- Find products currently in wrong gravity zones
WITH ProductPerformance AS (
    SELECT 
        p.ProductKey,
        p.SKU,
        p.GravityZone AS CurrentZone,
        COUNT(*) AS PickCount,
        AVG(w.CycleTimeMinutes) AS AvgCycleTime,
        AVG(p.UnitCost * w.QuantityHandled) AS AvgValueHandled
    FROM FactWarehouseOperations w
    INNER JOIN DimProduct p ON w.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.DateTime >= DATEADD(MONTH, -3, GETDATE())
        AND w.OperationType = 'Picking'
    GROUP BY p.ProductKey, p.SKU, p.GravityZone
)
SELECT 
    SKU,
    CurrentZone,
    PickCount,
    AvgCycleTime,
    AvgValueHandled,
    CASE 
        WHEN PickCount > 100 AND AvgValueHandled > 500 THEN 'High'
        WHEN PickCount > 50 OR AvgValueHandled > 200 THEN 'Medium'
        ELSE 'Low'
    END AS RecommendedZone
FROM ProductPerformance
WHERE CurrentZone <> CASE 
    WHEN PickCount > 100 AND AvgValueHandled > 500 THEN 'High'
    WHEN PickCount > 50 OR AvgValueHandled > 200 THEN 'Medium'
    ELSE 'Low'
END
ORDER BY PickCount DESC;
```

### Pattern 2: Correlate Weather Impact with Fleet Performance

```sql
-- Requires external weather data in staging table
SELECT 
    f.TripID,
    v.VehicleID,
    t.DateTime,
    f.DistanceMiles,
    f.TripDurationMinutes,
    f.DistanceMiles / NULLIF(f.TripDurationMinutes / 60.0, 0) AS AvgSpeedMPH,
    CASE WHEN f.WeatherImpact = 1 THEN 'Weather' ELSE 'Normal' END AS Condition
FROM FactFleetTrips f
INNER JOIN DimFleetVehicle v ON f.VehicleKey = v.VehicleKey
INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
WHERE t.DateTime >= DATEADD(MONTH, -1, GETDATE())
ORDER BY t.DateTime DESC;

-- Statistical comparison
SELECT 
    CASE WHEN WeatherImpact = 1 THEN 'Weather Impacted' ELSE 'Normal' END AS TripType,
    AVG(TripDurationMinutes) AS AvgDuration,
    AVG(IdleTimeMinutes) AS AvgIdleTime,
    AVG(FuelConsumed) AS AvgFuel,
    COUNT(*) AS TripCount
FROM FactFleetTrips
WHERE DepartureDateTime >= DATEADD(MONTH, -3, GETDATE())
GROUP BY WeatherImpact;
```

### Pattern 3: Proactive Maintenance Queue

```sql
-- Fleet triage based on maintenance alerts and load value
WITH VehicleRisk AS (
    SELECT 
        v.VehicleKey,
        v.VehicleID,
        v.NextMaintenanceDue,
        DATEDIFF(DAY, GETDATE(), v.NextMaintenanceDue) AS DaysUntilMaintenance,
        SUM(f.LoadWeight * p.UnitCost) AS TotalValueHauled,
        COUNT(CASE WHEN f.HasMaintenceAlert = 1 THEN 1 END) AS AlertCount,
        AVG(f.FuelConsumed / NULLIF(f.DistanceMiles, 0)) AS FuelEfficiency
    FROM DimFleetVehicle v
    LEFT JOIN FactFleetTrips f ON v.VehicleKey = f.VehicleKey
    LEFT JOIN BridgeRouteWarehouse b ON f.TripKey = b.TripKey
    LEFT JOIN FactWarehouseOperations w ON b.OperationKey = w.OperationKey
    LEFT JOIN DimProduct p ON w.ProductKey = p.ProductKey
    WHERE f.DepartureDateTime >= DATEADD(MONTH, -1, GETDATE())
    GROUP BY v.VehicleKey, v.VehicleID, v.NextMaintenanceDue
)
SELECT 
    VehicleID,
    DaysUntilMaintenance,
    AlertCount,
    TotalValueHauled,
    FuelEfficiency,
    -- Risk score: alerts + proximity to maintenance + value exposure
    (AlertCount * 10) + 
    (CASE WHEN DaysUntilMaintenance < 7 THEN 20 ELSE 0 END) +
    (TotalValueHauled / 10000) AS RiskScore
FROM VehicleRisk
WHERE AlertCount > 0 OR DaysUntilMaintenance < 14
ORDER BY RiskScore DESC;
```

### Pattern 4: Cross-Dock Efficiency Analysis

```sql
-- Analyze shipments that went directly from inbound to outbound
SELECT 
    g.LocationName AS Warehouse,
    p.Category,
    COUNT(DISTINCT b.RouteWarehouseKey) AS CrossDockCount,
    AVG(DATEDIFF(MINUTE, win.StartDateTime, wout.StartDateTime)) AS AvgTransferMinutes,
    SUM(win.QuantityHandled) AS TotalUnitsTransferred
FROM BridgeRouteWarehouse b
INNER JOIN FactWarehouseOperations win ON b.OperationKey = win.OperationKey
    AND win.OperationType = 'Receiving'
INNER JOIN FactFleetTrips f ON b.TripKey = f.TripKey
INNER JOIN BridgeRouteWarehouse b2 ON f.TripKey = b2.TripKey
    AND b2.RouteWarehouseKey <> b.RouteWarehouseKey
INNER JOIN FactWarehouseOperations wout ON b2.OperationKey = wout.OperationKey
    AND wout.OperationType = 'Shipping'
INNER JOIN DimProduct p ON win.ProductKey = p.ProductKey
INNER JOIN DimGeography g ON win.WarehouseKey = g.GeographyKey
WHERE DATEDIFF(HOUR, win.StartDateTime, wout.StartDateTime) < 24
GROUP BY g.LocationName, p.Category
ORDER BY CrossDockCount DESC;
```

## Environment Configuration

Create a configuration file `config.json` (add to `.gitignore`):

```json
{
  "database": {
    "server": "${SQL_SERVER_NAME}",
    "database": "LogiFleetPulse",
    "username": "${SQL_USERNAME}",
    "password": "${SQL_PASSWORD}",
    "connection_timeout": 30
  },
  "data_sources": {
    "wms_api": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "api_key": "${WMS_API_KEY}",
      "refresh_interval_minutes": 15
    },
    "fleet_telemetry": {
      "endpoint": "${TELEMETRY_API_ENDPOINT}",
      "api_key": "${TELEMETRY_API_KEY}",
      "refresh_interval_minutes": 5
    },
    "weather_api": {
      "endpoint": "https://api.weatherapi.com/v1",
      "api_key": "${WEATHER_API_KEY}"
    }
  },
  "alerts": {
    "email_recipients": ["${ALERT_EMAIL_1}", "${ALERT_EMAIL_2}"],
    "smtp_server": "${SMTP_SERVER}",
    "smtp_port": 587
  },
  "power_bi": {
    "workspace_id": "${POWERBI_WORKSPACE_ID}",
    "refresh_schedule": "0 */15 * * *"
  }
}
```

## Troubleshooting

### Issue: Power BI Relationship Detection Fails

**Symptom:** Relationships between fact and dimension tables not auto-detected

**Solution:**
```sql
-- Verify foreign key constraints exist
SELECT 
    fk.name AS ForeignKeyName,
    OBJECT_NAME(fk.parent_object_id) AS TableName,
    COL_NAME(fkc.parent_object_id, fkc.parent_column_id) AS ColumnName,
    OBJECT_NAME(fk.referenced_object_id) AS ReferencedTable
FROM sys.foreign_keys fk
INNER JOIN sys.foreign_key_columns fkc ON fk.object_id = fkc.constraint_object_id
WHERE OBJECT_NAME(fk.parent_object_id) LIKE 'Fact%';

-- If missing, recreate constraints
ALTER TABLE FactWarehouseOperations
ADD CONSTRAINT FK_Warehouse_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey);
```

In Power BI: **Manage Relationships** → **Autodetect** or manually create with:
- `FactWarehouseOperations[TimeKey]` → `DimTime[TimeKey]` (Many-to-One)

### Issue: Slow Query Performance

**Symptom:** Dashboards take >30 seconds to refresh

**Solution:**
```sql
-- Create columnstore index on large fact tables
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactWarehouse_Columnstore
ON FactWarehouseOperations (TimeKey, ProductKey, WarehouseKey, QuantityHandled, DwellTimeMinutes);

-- Partition fact tables by month
CREATE PARTITION FUNCTION PF_MonthlyPartition (INT)
AS RANGE RIGHT FOR VALUES (20260101, 20260201, 20260301); -- Add all months

CREATE PARTITION SCHEME PS_MonthlyPartition
AS PARTITION PF_MonthlyPartition ALL TO ([PRIMARY]);

-- Rebuild table on partition scheme (requires table recreation)
```

Use **Power BI Query Diagnostics** to identify slow queries.

### Issue: Time Dimension Granularity Mismatch

**Symptom:** Transactions don't align with 15-minute time slots

**Solution:**
```sql
-- Create helper function to round timestamps
CREATE FUNCTION dbo.RoundTo15Minutes (@DateTime DATETIME)
RETURNS DATETIME
AS
BEGIN
    RETURN DATEADD(MINUTE, 
        (DATEDIFF(MINUTE, 0, @DateTime) / 15) * 15, 
        CAST(CAST(@DateTime AS DATE) AS DATETIME));
END
GO

-- Use in ETL
SELECT dbo.RoundTo15Minutes(OperationDateTime) AS RoundedTime;
```

### Issue: Row-Level Security Not Filtering

**Symptom:** Users see data outside their region

**Solution:**
```dax
-- Verify role DAX filter is correct
-- In Power BI Desktop → Modeling → Manage Roles → View as Role

-- Common fix: Ensure user mapping table exists
-- Create table in SQL:
CREATE TABLE UserWarehouseMapping (
    Username VARCHAR(200),
    WarehouseCode VARCHAR(50),
    Region VARCHAR(100)
);

-- Populate with organizational data
INSERT INTO UserWarehouseMapping VALUES 
('user@company.com', 'WH-WEST-01', 'West'),
('manager@company.com', 'WH-EAST-01', 'East');
```

### Issue: External API Data Not Refreshing

**Symptom:** Weather/traffic data remains stale

**Solution:**
```sql
-- Create scheduled job for API ingestion
USE msdb;
GO

EXEC sp_add_job @job_name = 'RefreshExternalData';

EXEC sp_add_jobstep 
    @job_name = 'RefreshExternalData',
    @step_name = 'Call Weather API',
    @subsystem = 'TSQL',
    @command = 'EXEC dbo.ImportWeatherData',
    @database_name = 'LogiFleetPulse';

EXEC sp_add_schedule 
    @schedule_name = 'Every15Minutes',
    @freq_
