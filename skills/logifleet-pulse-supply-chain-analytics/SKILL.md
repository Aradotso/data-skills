---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing solution for logistics, fleet management, and supply chain analytics with multi-fact star schema
triggers:
  - set up logistics analytics dashboard
  - configure supply chain data warehouse
  - implement fleet management KPI tracking
  - create warehouse operations analytics
  - build multi-fact star schema for logistics
  - deploy Power BI logistics intelligence platform
  - analyze cross-dock and fleet telemetry data
  - set up real-time supply chain monitoring
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive data warehousing and business intelligence solution for logistics operations. It provides:

- **Multi-fact star schema** for warehouse operations, fleet trips, and cross-dock activities
- **Power BI dashboards** for real-time logistics monitoring
- **Cross-fact KPI harmonization** linking inventory, fleet, and operational metrics
- **Predictive analytics** for bottleneck detection and maintenance prioritization
- **Time-phased dimensions** with 15-minute granularity for operational awareness
- **Role-based access control** with row-level security

The platform integrates data from warehouse management systems (WMS), fleet telematics, supplier portals, and external APIs (weather, traffic) into a unified semantic layer.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition)
- Power BI Desktop (latest version)
- Access to source systems: WMS, TMS, telematics APIs
- SQL Server Management Studio (SSMS) recommended

### Step 1: Clone the Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### Step 2: Deploy SQL Schema

```bash
# Connect to your SQL Server instance
sqlcmd -S your-server-name -d LogiFleetDB -i schema/00_CreateDatabase.sql
sqlcmd -S your-server-name -d LogiFleetDB -i schema/01_DimensionTables.sql
sqlcmd -S your-server-name -d LogiFleetDB -i schema/02_FactTables.sql
sqlcmd -S your-server-name -d LogiFleetDB -i schema/03_Views.sql
sqlcmd -S your-server-name -d LogiFleetDB -i schema/04_StoredProcedures.sql
sqlcmd -S your-server-name -d LogiFleetDB -i schema/05_Indexes.sql
```

### Step 3: Configure Data Sources

Copy and customize the configuration file:

```bash
cp config_sample.json config.json
```

Edit `config.json` with your connection details:

```json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetDB",
    "username": "${SQL_SERVER_USER}",
    "password": "${SQL_SERVER_PASSWORD}"
  },
  "data_sources": {
    "wms_api": "${WMS_API_ENDPOINT}",
    "telematics_api": "${TELEMATICS_API_ENDPOINT}",
    "weather_api": "${WEATHER_API_ENDPOINT}"
  },
  "refresh_interval_minutes": 15
}
```

### Step 4: Import Power BI Template

1. Open Power BI Desktop
2. File → Import → Power BI Template
3. Select `LogiFleet_Pulse_Master.pbit`
4. Enter your SQL Server connection string when prompted
5. Configure credentials for the data source

## Core Data Model

### Dimension Tables

```sql
-- Time dimension with 15-minute granularity
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    TimeSlot15Min VARCHAR(5),
    HourOfDay INT,
    DayOfWeek INT,
    FiscalPeriod VARCHAR(10),
    IsWeekend BIT
);

-- Geography hierarchy
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY,
    LocationID VARCHAR(50) NOT NULL,
    LocationName VARCHAR(200),
    LocationType VARCHAR(50), -- 'Warehouse', 'Distribution Center', 'Route Node'
    City VARCHAR(100),
    Region VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6)
);

-- Product dimension with gravity scoring
CREATE TABLE DimProduct (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(100) NOT NULL,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    Subcategory VARCHAR(100),
    GravityScore DECIMAL(5,2), -- Velocity + value + fragility composite
    IsPerishable BIT,
    WeightKg DECIMAL(10,2),
    VolumeM3 DECIMAL(10,4)
);

-- Supplier reliability tracking
CREATE TABLE DimSupplier (
    SupplierKey INT PRIMARY KEY,
    SupplierID VARCHAR(50) NOT NULL,
    SupplierName VARCHAR(200),
    LeadTimeDays INT,
    LeadTimeVariance DECIMAL(5,2),
    DefectRate DECIMAL(5,4),
    ComplianceScore DECIMAL(5,2),
    IsActive BIT
);
```

### Fact Tables

```sql
-- Warehouse operations fact
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProduct(ProductKey),
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    QuantityHandled INT,
    DwellTimeMinutes INT,
    ProcessingTimeMinutes DECIMAL(10,2),
    StorageZone VARCHAR(50),
    BatchNumber VARCHAR(100),
    OperatorID VARCHAR(50)
);

-- Fleet trips fact
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID VARCHAR(50),
    DriverID VARCHAR(50),
    DistanceKm DECIMAL(10,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelLiters DECIMAL(10,2),
    LoadWeightKg DECIMAL(10,2),
    OnTimeStatus VARCHAR(20), -- 'OnTime', 'Delayed', 'Early'
    DelayReasonCode VARCHAR(50)
);

-- Cross-dock operations fact
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProduct(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    QuantityTransferred INT,
    TransferTimeMinutes DECIMAL(10,2),
    TemperatureCompliant BIT
);
```

## Key Stored Procedures

### Incremental Data Loading

```sql
-- Load warehouse operations incrementally
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, 
        OperationType, QuantityHandled, DwellTimeMinutes,
        ProcessingTimeMinutes, StorageZone, BatchNumber, OperatorID
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        wo.operation_type,
        wo.quantity_handled,
        DATEDIFF(MINUTE, wo.start_time, wo.end_time) AS DwellTimeMinutes,
        wo.processing_time_minutes,
        wo.storage_zone,
        wo.batch_number,
        wo.operator_id
    FROM external_wms_operations wo
    INNER JOIN DimTime t ON CAST(wo.operation_datetime AS DATETIME) = t.FullDateTime
    INNER JOIN DimGeography g ON wo.warehouse_id = g.LocationID
    INNER JOIN DimProduct p ON wo.sku = p.SKU
    WHERE wo.operation_datetime > @LastLoadDateTime
        AND wo.is_processed = 0;
    
    -- Mark as processed
    UPDATE external_wms_operations
    SET is_processed = 1
    WHERE operation_datetime > @LastLoadDateTime;
END;
```

### Cross-Fact KPI Calculation

```sql
-- Calculate composite logistics efficiency score
CREATE PROCEDURE usp_CalculateLogisticsEfficiency
    @StartDate DATE,
    @EndDate DATE,
    @GeographyKey INT = NULL
AS
BEGIN
    WITH WarehouseMetrics AS (
        SELECT 
            GeographyKey,
            AVG(DwellTimeMinutes) AS AvgDwellTime,
            AVG(ProcessingTimeMinutes) AS AvgProcessingTime,
            COUNT(*) AS OperationCount
        FROM FactWarehouseOperations wo
        INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
        WHERE CAST(t.FullDateTime AS DATE) BETWEEN @StartDate AND @EndDate
            AND (@GeographyKey IS NULL OR wo.GeographyKey = @GeographyKey)
        GROUP BY GeographyKey
    ),
    FleetMetrics AS (
        SELECT 
            OriginGeographyKey AS GeographyKey,
            AVG(CAST(IdleTimeMinutes AS FLOAT) / NULLIF(DurationMinutes, 0)) AS IdleTimeRatio,
            AVG(FuelLiters / NULLIF(DistanceKm, 0)) AS FuelEfficiency,
            SUM(CASE WHEN OnTimeStatus = 'OnTime' THEN 1 ELSE 0 END) * 100.0 / COUNT(*) AS OnTimePercentage
        FROM FactFleetTrips ft
        INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
        WHERE CAST(t.FullDateTime AS DATE) BETWEEN @StartDate AND @EndDate
            AND (@GeographyKey IS NULL OR ft.OriginGeographyKey = @GeographyKey)
        GROUP BY OriginGeographyKey
    )
    SELECT 
        g.LocationName,
        wm.AvgDwellTime,
        wm.AvgProcessingTime,
        fm.IdleTimeRatio,
        fm.FuelEfficiency,
        fm.OnTimePercentage,
        -- Composite efficiency score (lower is better)
        (wm.AvgDwellTime * 0.3 + 
         wm.AvgProcessingTime * 0.2 + 
         fm.IdleTimeRatio * 100 * 0.3 + 
         (100 - fm.OnTimePercentage) * 0.2) AS CompositeEfficiencyScore
    FROM WarehouseMetrics wm
    INNER JOIN FleetMetrics fm ON wm.GeographyKey = fm.GeographyKey
    INNER JOIN DimGeography g ON wm.GeographyKey = g.GeographyKey
    ORDER BY CompositeEfficiencyScore;
END;
```

### Automated Alerting

```sql
-- Configure threshold-based alerts
CREATE PROCEDURE usp_CheckLogisticsAlerts
AS
BEGIN
    DECLARE @AlertThresholds TABLE (
        MetricName VARCHAR(100),
        ThresholdValue DECIMAL(10,2),
        AlertLevel VARCHAR(20)
    );
    
    INSERT INTO @AlertThresholds VALUES
        ('IdleTimeRatio', 0.15, 'Warning'),
        ('IdleTimeRatio', 0.25, 'Critical'),
        ('DwellTimeHours', 72, 'Warning'),
        ('OnTimePercentage', 85, 'Warning');
    
    -- Check recent fleet idle time
    WITH RecentFleet AS (
        SELECT 
            VehicleID,
            AVG(CAST(IdleTimeMinutes AS FLOAT) / NULLIF(DurationMinutes, 0)) AS IdleRatio
        FROM FactFleetTrips ft
        INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
        WHERE t.FullDateTime >= DATEADD(HOUR, -4, GETDATE())
        GROUP BY VehicleID
    )
    INSERT INTO AlertLog (AlertTime, AlertLevel, MetricName, EntityID, CurrentValue, ThresholdValue)
    SELECT 
        GETDATE(),
        at.AlertLevel,
        'IdleTimeRatio',
        rf.VehicleID,
        rf.IdleRatio,
        at.ThresholdValue
    FROM RecentFleet rf
    CROSS APPLY (
        SELECT TOP 1 AlertLevel, ThresholdValue 
        FROM @AlertThresholds 
        WHERE MetricName = 'IdleTimeRatio' 
            AND rf.IdleRatio > ThresholdValue
        ORDER BY ThresholdValue DESC
    ) at;
    
    -- Check warehouse dwell time
    WITH RecentDwell AS (
        SELECT 
            p.SKU,
            g.LocationName,
            MAX(DwellTimeMinutes) / 60.0 AS MaxDwellHours
        FROM FactWarehouseOperations wo
        INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
        INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
        INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
        WHERE t.FullDateTime >= DATEADD(HOUR, -24, GETDATE())
            AND wo.OperationType = 'Putaway'
        GROUP BY p.SKU, g.LocationName
    )
    INSERT INTO AlertLog (AlertTime, AlertLevel, MetricName, EntityID, CurrentValue, ThresholdValue)
    SELECT 
        GETDATE(),
        at.AlertLevel,
        'DwellTimeHours',
        rd.SKU + ' @ ' + rd.LocationName,
        rd.MaxDwellHours,
        at.ThresholdValue
    FROM RecentDwell rd
    CROSS APPLY (
        SELECT TOP 1 AlertLevel, ThresholdValue 
        FROM @AlertThresholds 
        WHERE MetricName = 'DwellTimeHours' 
            AND rd.MaxDwellHours > ThresholdValue
        ORDER BY ThresholdValue DESC
    ) at;
END;
```

## Power BI Configuration

### DAX Measures for Cross-Fact Analysis

```dax
// Warehouse velocity score
WarehouseVelocity = 
VAR TotalOperations = COUNTROWS(FactWarehouseOperations)
VAR AvgProcessingTime = AVERAGE(FactWarehouseOperations[ProcessingTimeMinutes])
RETURN
    DIVIDE(TotalOperations, AvgProcessingTime, 0) * 100

// Fleet efficiency composite
FleetEfficiency = 
VAR OnTimeTrips = CALCULATE(
    COUNTROWS(FactFleetTrips),
    FactFleetTrips[OnTimeStatus] = "OnTime"
)
VAR TotalTrips = COUNTROWS(FactFleetTrips)
VAR AvgFuelEfficiency = AVERAGE(FactFleetTrips[FuelLiters]) / AVERAGE(FactFleetTrips[DistanceKm])
VAR AvgIdleRatio = DIVIDE(
    SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[DurationMinutes]),
    0
)
RETURN
    (DIVIDE(OnTimeTrips, TotalTrips, 0) * 0.5 + 
     (1 - AvgIdleRatio) * 0.3 + 
     (1 - MIN(AvgFuelEfficiency, 0.15) / 0.15) * 0.2) * 100

// Cross-fact correlation: Dwell time vs delivery delay
DwellDelayCorrelation = 
VAR DwellData = 
    ADDCOLUMNS(
        SUMMARIZE(
            FactWarehouseOperations,
            FactWarehouseOperations[ProductKey],
            "AvgDwell", AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
        ),
        "ProductKey", [ProductKey]
    )
VAR DelayData = 
    ADDCOLUMNS(
        SUMMARIZE(
            FactFleetTrips,
            FactFleetTrips[ProductKey],
            "DelayRate", DIVIDE(
                CALCULATE(COUNTROWS(FactFleetTrips), FactFleetTrips[OnTimeStatus] = "Delayed"),
                COUNTROWS(FactFleetTrips),
                0
            )
        ),
        "ProductKey", [ProductKey]
    )
RETURN
    // Simplified correlation calculation
    AVERAGEX(
        NATURALINNERJOIN(DwellData, DelayData),
        [AvgDwell] * [DelayRate]
    )
```

### Row-Level Security

```dax
// Role: Regional Manager (filter by geography)
[GeographyKey] IN {1, 2, 5, 8}

// Role: Fleet Supervisor (filter by vehicle assignments)
[VehicleID] IN (
    SELECTCOLUMNS(
        FILTER(
            UserVehicleAssignments,
            UserVehicleAssignments[UserEmail] = USERPRINCIPALNAME()
        ),
        "VehicleID", UserVehicleAssignments[VehicleID]
    )
)
```

## Common Usage Patterns

### Pattern 1: Daily Operations Dashboard Refresh

```sql
-- Schedule this as a SQL Agent job
DECLARE @LastLoad DATETIME;

SELECT @LastLoad = MAX(LoadDateTime) 
FROM DataLoadLog 
WHERE LoadType = 'WarehouseOperations';

EXEC usp_LoadWarehouseOperations @LastLoadDateTime = @LastLoad;
EXEC usp_LoadFleetTrips @LastLoadDateTime = @LastLoad;
EXEC usp_LoadCrossDock @LastLoadDateTime = @LastLoad;

INSERT INTO DataLoadLog (LoadDateTime, LoadType, RecordsProcessed)
VALUES (GETDATE(), 'DailyRefresh', @@ROWCOUNT);
```

### Pattern 2: Gravity Zone Optimization Analysis

```sql
-- Identify products for gravity zone reassignment
WITH ProductVelocity AS (
    SELECT 
        p.ProductKey,
        p.SKU,
        p.ProductName,
        p.GravityScore AS CurrentGravity,
        COUNT(*) AS PickFrequency,
        AVG(wo.ProcessingTimeMinutes) AS AvgPickTime,
        SUM(wo.QuantityHandled * p.WeightKg) AS TotalWeightHandled
    FROM FactWarehouseOperations wo
    INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE wo.OperationType = 'Picking'
        AND t.FullDateTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY p.ProductKey, p.SKU, p.ProductName, p.GravityScore
)
SELECT 
    SKU,
    ProductName,
    CurrentGravity,
    PickFrequency,
    -- Calculate optimal gravity score
    (PickFrequency * 0.4 + 
     (1.0 / NULLIF(AvgPickTime, 0)) * 100 * 0.4 + 
     TotalWeightHandled / 1000 * 0.2) AS OptimalGravity,
    CASE 
        WHEN ABS(CurrentGravity - (PickFrequency * 0.4 + (1.0 / NULLIF(AvgPickTime, 0)) * 100 * 0.4 + TotalWeightHandled / 1000 * 0.2)) > 20
        THEN 'Reassign Recommended'
        ELSE 'Current Zone OK'
    END AS Recommendation
FROM ProductVelocity
ORDER BY PickFrequency DESC;
```

### Pattern 3: Predictive Maintenance Queue

```sql
-- Generate fleet maintenance priority queue
WITH VehicleHealth AS (
    SELECT 
        ft.VehicleID,
        AVG(ft.FuelLiters / NULLIF(ft.DistanceKm, 0)) AS AvgFuelConsumption,
        AVG(CAST(ft.IdleTimeMinutes AS FLOAT) / NULLIF(ft.DurationMinutes, 0)) AS AvgIdleRatio,
        COUNT(*) AS TripCount,
        SUM(ft.DistanceKm) AS TotalDistanceKm
    FROM FactFleetTrips ft
    INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(DAY, -90, GETDATE())
    GROUP BY ft.VehicleID
),
VehicleRevenue AS (
    SELECT 
        ft.VehicleID,
        SUM(cd.QuantityTransferred * p.GravityScore) AS RevenueImpactScore
    FROM FactFleetTrips ft
    INNER JOIN FactCrossDock cd ON ft.TripKey = cd.OutboundTripKey
    INNER JOIN DimProduct p ON cd.ProductKey = p.ProductKey
    GROUP BY ft.VehicleID
)
SELECT 
    vh.VehicleID,
    vh.AvgFuelConsumption,
    vh.AvgIdleRatio,
    vh.TotalDistanceKm,
    ISNULL(vr.RevenueImpactScore, 0) AS RevenueImpact,
    -- Priority score: higher fuel consumption + higher idle + higher revenue impact
    (vh.AvgFuelConsumption * 100 * 0.3 + 
     vh.AvgIdleRatio * 100 * 0.3 + 
     ISNULL(vr.RevenueImpactScore, 0) / 100 * 0.4) AS MaintenancePriorityScore
FROM VehicleHealth vh
LEFT JOIN VehicleRevenue vr ON vh.VehicleID = vr.VehicleID
WHERE vh.TotalDistanceKm > 10000 -- Only vehicles with significant usage
ORDER BY MaintenancePriorityScore DESC;
```

## Troubleshooting

### Issue: Power BI Refresh Fails

**Symptoms**: "Unable to connect to the data source" error

**Solutions**:
1. Verify SQL Server allows remote connections:
   ```sql
   EXEC sp_configure 'remote access', 1;
   RECONFIGURE;
   ```

2. Check firewall rules for port 1433

3. Update credentials in Power BI:
   - File → Options → Data source settings
   - Select SQL Server connection
   - Edit Permissions → Credentials → Edit
   - Use Windows or Database authentication with `${SQL_SERVER_USER}` and `${SQL_SERVER_PASSWORD}`

### Issue: Slow Cross-Fact Queries

**Symptoms**: Queries joining FactWarehouseOperations and FactFleetTrips take >30 seconds

**Solutions**:
1. Ensure indexes exist on foreign keys:
   ```sql
   CREATE NONCLUSTERED INDEX IX_FactWarehouseOps_Time 
   ON FactWarehouseOperations(TimeKey) INCLUDE (GeographyKey, ProductKey);
   
   CREATE NONCLUSTERED INDEX IX_FactFleetTrips_Time 
   ON FactFleetTrips(TimeKey) INCLUDE (OriginGeographyKey, DestinationGeographyKey);
   ```

2. Implement table partitioning by time:
   ```sql
   CREATE PARTITION FUNCTION PF_TimeKey (INT)
   AS RANGE RIGHT FOR VALUES (20260101, 20260201, 20260301, 20260401);
   
   CREATE PARTITION SCHEME PS_TimeKey
   AS PARTITION PF_TimeKey ALL TO ([PRIMARY]);
   
   ALTER TABLE FactWarehouseOperations 
   DROP CONSTRAINT PK_FactWarehouseOperations;
   
   ALTER TABLE FactWarehouseOperations 
   ADD CONSTRAINT PK_FactWarehouseOperations PRIMARY KEY (OperationKey, TimeKey)
   ON PS_TimeKey(TimeKey);
   ```

3. Use indexed views for common aggregations:
   ```sql
   CREATE VIEW vw_DailyWarehouseMetrics
   WITH SCHEMABINDING
   AS
   SELECT 
       wo.GeographyKey,
       CAST(t.FullDateTime AS DATE) AS OperationDate,
       COUNT_BIG(*) AS OperationCount,
       SUM(wo.QuantityHandled) AS TotalQuantity,
       AVG(wo.DwellTimeMinutes) AS AvgDwellTime
   FROM dbo.FactWarehouseOperations wo
   INNER JOIN dbo.DimTime t ON wo.TimeKey = t.TimeKey
   GROUP BY wo.GeographyKey, CAST(t.FullDateTime AS DATE);
   
   CREATE UNIQUE CLUSTERED INDEX IX_vw_DailyWarehouse
   ON vw_DailyWarehouseMetrics(GeographyKey, OperationDate);
   ```

### Issue: Alert Procedure Not Triggering

**Symptoms**: `usp_CheckLogisticsAlerts` runs but no alerts generated

**Solutions**:
1. Verify AlertLog table exists:
   ```sql
   IF NOT EXISTS (SELECT * FROM sys.tables WHERE name = 'AlertLog')
   CREATE TABLE AlertLog (
       AlertID INT PRIMARY KEY IDENTITY,
       AlertTime DATETIME NOT NULL,
       AlertLevel VARCHAR(20),
       MetricName VARCHAR(100),
       EntityID VARCHAR(100),
       CurrentValue DECIMAL(10,2),
       ThresholdValue DECIMAL(10,2),
       IsAcknowledged BIT DEFAULT 0
   );
   ```

2. Test alert logic manually:
   ```sql
   SELECT 
       VehicleID,
       AVG(CAST(IdleTimeMinutes AS FLOAT) / NULLIF(DurationMinutes, 0)) AS IdleRatio
   FROM FactFleetTrips ft
   INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
   WHERE t.FullDateTime >= DATEADD(HOUR, -4, GETDATE())
   GROUP BY VehicleID
   HAVING AVG(CAST(IdleTimeMinutes AS FLOAT) / NULLIF(DurationMinutes, 0)) > 0.15;
   ```

3. Enable SQL Agent job logging:
   - SQL Server Agent → Jobs → Properties
   - Steps → Edit → Advanced
   - Check "Log to table" and "Include step output in history"

### Issue: Missing External Data (Weather/Traffic)

**Symptoms**: External API data not appearing in dashboard

**Solutions**:
1. Verify external table configuration:
   ```sql
   -- Check if external data source is configured
   SELECT * FROM sys.external_data_sources;
   
   -- Test external table query
   SELECT TOP 10 * FROM external_weather_data;
   ```

2. Implement fallback logic for missing data:
   ```sql
   -- In your ETL procedure
   IF NOT EXISTS (SELECT 1 FROM external_weather_data WHERE event_date = CAST(GETDATE() AS DATE))
   BEGIN
       -- Use default values or skip weather correlation
       INSERT INTO FactFleetTrips (..., WeatherImpact)
       SELECT ..., 'Unknown' AS WeatherImpact
       FROM staging_fleet_data;
   END
   ```

3. Log API errors to troubleshooting table:
   ```sql
   CREATE TABLE APIErrorLog (
       ErrorID INT PRIMARY KEY IDENTITY,
       ErrorTime DATETIME DEFAULT GETDATE(),
       APIName VARCHAR(100),
       ErrorMessage NVARCHAR(MAX),
       RequestPayload NVARCHAR(MAX)
   );
   ```

## Environment Variables Reference

Configure these in your environment before deployment:

- `SQL_SERVER_HOST` - SQL Server hostname or IP
- `SQL_SERVER_USER` - Database user with read/write permissions
- `SQL_SERVER_PASSWORD` - Database password
- `WMS_API_ENDPOINT` - Warehouse management system API URL
- `TELEMATICS_API_ENDPOINT` - Fleet telematics API URL
- `WEATHER_API_ENDPOINT` - Weather data API URL (optional)
- `POWERBI_WORKSPACE_ID` - Power BI workspace for publishing dashboards
- `ALERT_EMAIL_SMTP` - SMTP server for email alerts
- `ALERT_EMAIL_FROM` - Sender email address for alerts

## Best Practices

1. **Schedule incremental loads every 15 minutes** to maintain real-time dashboards
2. **Partition fact tables by time** for queries spanning long date ranges
3. **Use columnstore indexes** on fact tables with millions of rows
4. **Implement data retention policies** (e.g., aggregate daily after 90 days, archive after 2 years)
5. **Test row-level security** with multiple user accounts before production deployment
6. **Monitor query performance** using SQL Server Query Store
7. **Version control Power BI templates** (.pbit files) in git repository
8. **Document custom DAX measures** with comments explaining business logic
