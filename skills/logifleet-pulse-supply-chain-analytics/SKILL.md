---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehouse with multi-fact star schema for warehouse, fleet, and supply chain analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "deploy the logistics data warehouse schema"
  - "configure Power BI for fleet and warehouse KPIs"
  - "create supply chain analytics dashboard"
  - "implement multi-fact star schema for logistics"
  - "connect warehouse and fleet data sources"
  - "optimize logistics data model in SQL Server"
  - "build cross-modal supply chain reports"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive data warehousing solution for supply chain and logistics analytics, combining MS SQL Server backend with Power BI visualization. It implements a multi-fact star schema that unifies:

- Warehouse operations (receiving, putaway, picking, packing, shipping)
- Fleet telemetry (GPS, fuel consumption, idle time, maintenance)
- Inventory management (aging, gravity zones, turnover)
- Supplier reliability metrics
- External factors (weather, traffic delays)

The platform enables cross-fact KPI analysis (e.g., correlating dwell time with fleet costs) and predictive bottleneck detection.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio

### Database Deployment

1. **Clone the repository:**

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**

```sql
-- Connect to your SQL Server instance in SSMS
-- Execute the schema creation script
:r schema/01_create_database.sql
:r schema/02_create_dimensions.sql
:r schema/03_create_facts.sql
:r schema/04_create_views.sql
:r schema/05_create_procedures.sql
```

3. **Configure data sources:**

Edit `config_sample.json` and save as `config.json`:

```json
{
  "sql_server": {
    "server": "localhost\\SQLEXPRESS",
    "database": "LogiFleetPulse",
    "authentication": "windows",
    "connection_timeout": 30
  },
  "data_sources": {
    "wms_api": "${WMS_API_ENDPOINT}",
    "wms_api_key": "${WMS_API_KEY}",
    "telematics_feed": "${TELEMATICS_FEED_URL}",
    "telematics_token": "${TELEMATICS_AUTH_TOKEN}",
    "weather_api": "${WEATHER_API_URL}",
    "weather_key": "${WEATHER_API_KEY}"
  },
  "refresh_intervals": {
    "warehouse_operations": "5min",
    "fleet_telemetry": "1min",
    "inventory_snapshot": "15min"
  }
}
```

### Power BI Setup

1. **Open the template:**

```bash
# Open Power BI Desktop and load the template
LogiFleet_Pulse_Master.pbit
```

2. **Configure data source connection:**

- File → Options and settings → Data source settings
- Update SQL Server connection string
- Enter credentials (Windows Auth or SQL Auth)
- Apply changes and refresh

## Data Model Architecture

### Core Fact Tables

#### FactWarehouseOperations

Tracks all warehouse activities with time-phased granularity:

```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    OperationType VARCHAR(20) NOT NULL, -- 'RECEIVE', 'PUTAWAY', 'PICK', 'PACK', 'SHIP'
    QuantityHandled INT NOT NULL,
    DwellTimeMinutes INT NULL,
    EmployeeKey INT NULL,
    StorageZoneKey INT NULL,
    ProcessingTimeSeconds INT NOT NULL,
    ErrorFlag BIT DEFAULT 0,
    RecordedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    CONSTRAINT FK_WO_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WO_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_WO_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey)
);

-- Recommended indexes
CREATE NONCLUSTERED INDEX IX_WO_Time ON FactWarehouseOperations(TimeKey) INCLUDE (ProductKey, QuantityHandled);
CREATE NONCLUSTERED INDEX IX_WO_Product ON FactWarehouseOperations(ProductKey, TimeKey);
CREATE NONCLUSTERED INDEX IX_WO_Zone ON FactWarehouseOperations(StorageZoneKey, TimeKey);
```

#### FactFleetTrips

Captures fleet movement and telemetry:

```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    VehicleKey INT NOT NULL,
    DriverKey INT NULL,
    DepartureTimeKey INT NOT NULL,
    ArrivalTimeKey INT NULL,
    RouteKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    DistanceMiles DECIMAL(10,2) NOT NULL,
    FuelConsumedGallons DECIMAL(8,3) NOT NULL,
    IdleTimeMinutes INT DEFAULT 0,
    LoadingTimeMinutes INT DEFAULT 0,
    UnloadingTimeMinutes INT DEFAULT 0,
    AverageSpeedMPH DECIMAL(5,2) NULL,
    WeatherConditionKey INT NULL,
    DelayMinutes INT DEFAULT 0,
    DelayReasonKey INT NULL,
    TripCost DECIMAL(10,2) NULL,
    RecordedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    CONSTRAINT FK_FT_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_FT_Route FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey)
);

CREATE NONCLUSTERED INDEX IX_FT_Departure ON FactFleetTrips(DepartureTimeKey, VehicleKey);
CREATE NONCLUSTERED INDEX IX_FT_Route ON FactFleetTrips(RouteKey, DepartureTimeKey);
```

### Key Dimension Tables

#### DimTime (15-minute grain)

```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    Year INT NOT NULL,
    Quarter INT NOT NULL,
    Month INT NOT NULL,
    DayOfMonth INT NOT NULL,
    DayOfWeek INT NOT NULL,
    DayName VARCHAR(10) NOT NULL,
    WeekOfYear INT NOT NULL,
    Hour INT NOT NULL,
    Minute INT NOT NULL,
    TimeSlot15Min INT NOT NULL, -- 0-95 (96 slots per day)
    IsWeekend BIT NOT NULL,
    IsHoliday BIT DEFAULT 0,
    FiscalYear INT NULL,
    FiscalQuarter INT NULL,
    ShiftName VARCHAR(20) NULL -- 'DAY', 'EVENING', 'NIGHT'
);

-- Populate time dimension
CREATE PROCEDURE PopulateDimTime
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    DECLARE @CurrentDateTime DATETIME2 = @StartDate;
    
    WHILE @CurrentDateTime < @EndDate
    BEGIN
        INSERT INTO DimTime (TimeKey, FullDateTime, Year, Quarter, Month, DayOfMonth, DayOfWeek, DayName, WeekOfYear, Hour, Minute, TimeSlot15Min, IsWeekend)
        VALUES (
            CAST(FORMAT(@CurrentDateTime, 'yyyyMMddHHmm') AS INT),
            @CurrentDateTime,
            YEAR(@CurrentDateTime),
            DATEPART(QUARTER, @CurrentDateTime),
            MONTH(@CurrentDateTime),
            DAY(@CurrentDateTime),
            DATEPART(WEEKDAY, @CurrentDateTime),
            DATENAME(WEEKDAY, @CurrentDateTime),
            DATEPART(WEEK, @CurrentDateTime),
            DATEPART(HOUR, @CurrentDateTime),
            DATEPART(MINUTE, @CurrentDateTime),
            (DATEPART(HOUR, @CurrentDateTime) * 4) + (DATEPART(MINUTE, @CurrentDateTime) / 15),
            CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1,7) THEN 1 ELSE 0 END
        );
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END;
```

#### DimProductGravity (Warehouse optimization)

```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName VARCHAR(200) NOT NULL,
    CategoryKey INT NOT NULL,
    GravityScore DECIMAL(5,2) NOT NULL, -- Calculated: velocity * value / fragility
    VelocityClass VARCHAR(20) NOT NULL, -- 'FAST', 'MEDIUM', 'SLOW'
    ValueClass VARCHAR(20) NOT NULL, -- 'HIGH', 'MEDIUM', 'LOW'
    FragilityScore INT NOT NULL, -- 1-10
    OptimalZoneType VARCHAR(30) NULL, -- 'NEAR_DOCK', 'MID_ZONE', 'DEEP_STORAGE'
    WeightLbs DECIMAL(8,2) NULL,
    VolumeCubicFt DECIMAL(8,2) NULL,
    RequiresRefrigeration BIT DEFAULT 0,
    LastGravityUpdate DATETIME2 DEFAULT GETUTCDATE()
);

-- Calculate gravity score
CREATE PROCEDURE RecalculateProductGravity
    @LookbackDays INT = 90
AS
BEGIN
    UPDATE p
    SET 
        GravityScore = (v.PicksPerDay * v.AvgValue) / NULLIF(p.FragilityScore, 0),
        VelocityClass = CASE 
            WHEN v.PicksPerDay >= 20 THEN 'FAST'
            WHEN v.PicksPerDay >= 5 THEN 'MEDIUM'
            ELSE 'SLOW'
        END,
        OptimalZoneType = CASE 
            WHEN (v.PicksPerDay * v.AvgValue) / NULLIF(p.FragilityScore, 0) >= 100 THEN 'NEAR_DOCK'
            WHEN (v.PicksPerDay * v.AvgValue) / NULLIF(p.FragilityScore, 0) >= 50 THEN 'MID_ZONE'
            ELSE 'DEEP_STORAGE'
        END,
        LastGravityUpdate = GETUTCDATE()
    FROM DimProductGravity p
    INNER JOIN (
        SELECT 
            ProductKey,
            COUNT(*) * 1.0 / @LookbackDays AS PicksPerDay,
            AVG(o.UnitPrice * o.QuantityHandled) AS AvgValue
        FROM FactWarehouseOperations wo
        INNER JOIN Orders o ON wo.OrderKey = o.OrderKey
        WHERE wo.OperationType = 'PICK'
            AND wo.RecordedAt >= DATEADD(DAY, -@LookbackDays, GETUTCDATE())
        GROUP BY ProductKey
    ) v ON p.ProductKey = v.ProductKey;
END;
```

## Key Analytical Queries

### Cross-Fact KPI: Dwell Time vs Fleet Idle Cost

```sql
-- Correlation between warehouse dwell time and fleet idle costs
WITH WarehouseDwell AS (
    SELECT 
        t.Year,
        t.Month,
        w.WarehouseName,
        AVG(wo.DwellTimeMinutes) AS AvgDwellMinutes,
        SUM(wo.QuantityHandled) AS TotalVolume
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    INNER JOIN DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
    WHERE wo.OperationType = 'PICK'
    GROUP BY t.Year, t.Month, w.WarehouseName
),
FleetIdle AS (
    SELECT 
        t.Year,
        t.Month,
        g.Region,
        SUM(ft.IdleTimeMinutes) AS TotalIdleMinutes,
        SUM(ft.IdleTimeMinutes * v.HourlyCost / 60.0) AS TotalIdleCost
    FROM FactFleetTrips ft
    INNER JOIN DimTime t ON ft.DepartureTimeKey = t.TimeKey
    INNER JOIN DimGeography g ON ft.OriginGeographyKey = g.GeographyKey
    INNER JOIN DimVehicle v ON ft.VehicleKey = v.VehicleKey
    GROUP BY t.Year, t.Month, g.Region
)
SELECT 
    wd.Year,
    wd.Month,
    wd.WarehouseName,
    wd.AvgDwellMinutes,
    fi.TotalIdleMinutes,
    fi.TotalIdleCost,
    CASE 
        WHEN wd.AvgDwellMinutes > 120 AND fi.TotalIdleCost > 5000 THEN 'HIGH_RISK'
        WHEN wd.AvgDwellMinutes > 60 OR fi.TotalIdleCost > 2000 THEN 'MEDIUM_RISK'
        ELSE 'LOW_RISK'
    END AS RiskLevel
FROM WarehouseDwell wd
LEFT JOIN FleetIdle fi ON wd.Year = fi.Year AND wd.Month = fi.Month
ORDER BY wd.Year DESC, wd.Month DESC, fi.TotalIdleCost DESC;
```

### Predictive Bottleneck Detection

```sql
-- Identify warehouses and time slots approaching capacity bottlenecks
CREATE PROCEDURE PredictBottlenecks
    @ForecastDays INT = 7,
    @CapacityThresholdPct DECIMAL(5,2) = 85.0
AS
BEGIN
    WITH HistoricalVolume AS (
        SELECT 
            wo.WarehouseKey,
            t.DayOfWeek,
            t.Hour,
            AVG(wo.QuantityHandled) AS AvgVolume,
            STDEV(wo.QuantityHandled) AS StdDevVolume
        FROM FactWarehouseOperations wo
        INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.FullDateTime >= DATEADD(DAY, -90, GETUTCDATE())
        GROUP BY wo.WarehouseKey, t.DayOfWeek, t.Hour
    ),
    PredictedLoad AS (
        SELECT 
            hv.WarehouseKey,
            hv.DayOfWeek,
            hv.Hour,
            hv.AvgVolume + (1.96 * hv.StdDevVolume) AS PredictedMaxVolume, -- 95% confidence
            w.HourlyCapacity
        FROM HistoricalVolume hv
        INNER JOIN DimWarehouse w ON hv.WarehouseKey = w.WarehouseKey
    )
    SELECT 
        w.WarehouseName,
        pl.DayOfWeek,
        pl.Hour,
        pl.PredictedMaxVolume,
        pl.HourlyCapacity,
        (pl.PredictedMaxVolume / pl.HourlyCapacity) * 100 AS CapacityUtilizationPct,
        CASE 
            WHEN (pl.PredictedMaxVolume / pl.HourlyCapacity) * 100 >= 95 THEN 'CRITICAL'
            WHEN (pl.PredictedMaxVolume / pl.HourlyCapacity) * 100 >= @CapacityThresholdPct THEN 'WARNING'
            ELSE 'NORMAL'
        END AS AlertLevel
    FROM PredictedLoad pl
    INNER JOIN DimWarehouse w ON pl.WarehouseKey = w.WarehouseKey
    WHERE (pl.PredictedMaxVolume / pl.HourlyCapacity) * 100 >= @CapacityThresholdPct
    ORDER BY CapacityUtilizationPct DESC;
END;
```

### Fleet Maintenance Priority Queue

```sql
-- Adaptive fleet triage based on telemetry and cargo value
SELECT TOP 20
    v.VehicleID,
    v.VehicleName,
    v.CurrentOdometer,
    v.LastMaintenanceDate,
    DATEDIFF(DAY, v.LastMaintenanceDate, GETUTCDATE()) AS DaysSinceMaintenance,
    t.DiagnosticCode,
    t.Severity,
    AVG(o.OrderValue) AS AvgCargoValue,
    COUNT(DISTINCT ft.TripKey) AS RecentTrips,
    -- Priority score: severity * days overdue * cargo value factor
    (t.Severity * 
     CASE WHEN DATEDIFF(DAY, v.LastMaintenanceDate, GETUTCDATE()) > v.MaintenanceIntervalDays 
          THEN DATEDIFF(DAY, v.LastMaintenanceDate, GETUTCDATE()) - v.MaintenanceIntervalDays 
          ELSE 1 END * 
     LOG(1 + AVG(o.OrderValue))) AS PriorityScore
FROM DimVehicle v
INNER JOIN VehicleTelemetry t ON v.VehicleKey = t.VehicleKey
LEFT JOIN FactFleetTrips ft ON v.VehicleKey = ft.VehicleKey AND ft.DepartureTimeKey >= DATEADD(DAY, -7, GETUTCDATE())
LEFT JOIN TripOrders tor ON ft.TripKey = tor.TripKey
LEFT JOIN Orders o ON tor.OrderKey = o.OrderKey
WHERE t.RequiresAttention = 1
    AND t.RecordedAt >= DATEADD(DAY, -3, GETUTCDATE())
GROUP BY v.VehicleID, v.VehicleName, v.CurrentOdometer, v.LastMaintenanceDate, v.MaintenanceIntervalDays, t.DiagnosticCode, t.Severity
ORDER BY PriorityScore DESC;
```

## Power BI Configuration

### DAX Measures for Cross-Fact Analysis

```dax
// Total Dwell Time Hours
Total Dwell Hours = 
SUMX(
    FactWarehouseOperations,
    FactWarehouseOperations[DwellTimeMinutes] / 60
)

// Fleet Utilization Rate
Fleet Utilization % = 
DIVIDE(
    SUM(FactFleetTrips[DistanceMiles]),
    SUM(DimVehicle[MaxDailyMiles]) * DISTINCTCOUNT(DimTime[Date]),
    0
) * 100

// Cross-Fact: Cost per Unit Shipped
Cost per Unit Shipped = 
DIVIDE(
    SUM(FactFleetTrips[TripCost]) + SUM(FactWarehouseOperations[ProcessingCost]),
    SUMX(
        FILTER(FactWarehouseOperations, FactWarehouseOperations[OperationType] = "SHIP"),
        FactWarehouseOperations[QuantityHandled]
    ),
    0
)

// Warehouse Gravity Score Compliance
Gravity Compliance % = 
VAR OptimalPlacements = 
    COUNTROWS(
        FILTER(
            FactWarehouseOperations,
            RELATED(DimProductGravity[OptimalZoneType]) = RELATED(DimStorageZone[ZoneType])
        )
    )
VAR TotalPlacements = COUNTROWS(FactWarehouseOperations)
RETURN
    DIVIDE(OptimalPlacements, TotalPlacements, 0) * 100

// Predictive Delay Risk Index
Delay Risk Index = 
VAR AvgDelay = AVERAGE(FactFleetTrips[DelayMinutes])
VAR StdDevDelay = STDEV.P(FactFleetTrips[DelayMinutes])
VAR CurrentDelay = SUM(FactFleetTrips[DelayMinutes])
RETURN
    IF(
        StdDevDelay > 0,
        (CurrentDelay - AvgDelay) / StdDevDelay,
        0
    )
```

### Row-Level Security Setup

```dax
// Create role: Regional Manager
[Region] = USERPRINCIPALNAME()

// Create role: Warehouse Supervisor
[WarehouseCode] IN (
    SELECTCOLUMNS(
        FILTER(UserWarehouseAccess, UserWarehouseAccess[UserEmail] = USERPRINCIPALNAME()),
        "Code", UserWarehouseAccess[WarehouseCode]
    )
)
```

## Data Ingestion Patterns

### Incremental Load Stored Procedure

```sql
CREATE PROCEDURE IncrementalLoadWarehouseOperations
    @LastLoadDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRY
        BEGIN TRANSACTION;
        
        -- Stage new records
        INSERT INTO FactWarehouseOperations (
            TimeKey, ProductKey, WarehouseKey, OperationType, 
            QuantityHandled, DwellTimeMinutes, EmployeeKey, 
            StorageZoneKey, ProcessingTimeSeconds, ErrorFlag, RecordedAt
        )
        SELECT 
            CAST(FORMAT(src.OperationDateTime, 'yyyyMMddHHmm') AS INT) AS TimeKey,
            p.ProductKey,
            w.WarehouseKey,
            src.OperationType,
            src.Quantity,
            DATEDIFF(MINUTE, src.ArrivalDateTime, src.OperationDateTime) AS DwellTimeMinutes,
            e.EmployeeKey,
            z.StorageZoneKey,
            src.ProcessingSeconds,
            CASE WHEN src.ErrorCode IS NOT NULL THEN 1 ELSE 0 END,
            src.OperationDateTime
        FROM StagingWarehouseOperations src
        INNER JOIN DimProduct p ON src.SKU = p.SKU
        INNER JOIN DimWarehouse w ON src.WarehouseCode = w.WarehouseCode
        LEFT JOIN DimEmployee e ON src.EmployeeID = e.EmployeeID
        LEFT JOIN DimStorageZone z ON src.ZoneCode = z.ZoneCode AND z.WarehouseKey = w.WarehouseKey
        WHERE src.OperationDateTime > @LastLoadDateTime
            AND src.IsProcessed = 0;
        
        -- Mark staged records as processed
        UPDATE StagingWarehouseOperations
        SET IsProcessed = 1, ProcessedAt = GETUTCDATE()
        WHERE OperationDateTime > @LastLoadDateTime;
        
        COMMIT TRANSACTION;
        
        -- Log success
        INSERT INTO ETLLog (ProcedureName, Status, RecordsProcessed, ExecutedAt)
        VALUES ('IncrementalLoadWarehouseOperations', 'SUCCESS', @@ROWCOUNT, GETUTCDATE());
        
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
        
        -- Log error
        INSERT INTO ETLLog (ProcedureName, Status, ErrorMessage, ExecutedAt)
        VALUES ('IncrementalLoadWarehouseOperations', 'ERROR', ERROR_MESSAGE(), GETUTCDATE());
        
        THROW;
    END CATCH
END;
```

### API Integration Example (Python)

```python
import pyodbc
import requests
import os
from datetime import datetime, timedelta

# Database connection
conn_str = (
    f"DRIVER={{ODBC Driver 17 for SQL Server}};"
    f"SERVER={os.getenv('SQL_SERVER')};"
    f"DATABASE={os.getenv('SQL_DATABASE')};"
    f"Trusted_Connection=yes;"
)

def ingest_telematics_data():
    """Pull fleet telemetry from API and insert into staging table"""
    
    # Get last ingestion timestamp
    with pyodbc.connect(conn_str) as conn:
        cursor = conn.cursor()
        cursor.execute("SELECT MAX(RecordedAt) FROM StagingFleetTelemetry")
        last_sync = cursor.fetchone()[0] or (datetime.utcnow() - timedelta(days=1))
    
    # Fetch from telematics API
    headers = {"Authorization": f"Bearer {os.getenv('TELEMATICS_AUTH_TOKEN')}"}
    params = {"since": last_sync.isoformat(), "limit": 1000}
    
    response = requests.get(
        os.getenv('TELEMATICS_FEED_URL'),
        headers=headers,
        params=params,
        timeout=30
    )
    response.raise_for_status()
    
    telemetry_data = response.json()
    
    # Insert into staging
    with pyodbc.connect(conn_str) as conn:
        cursor = conn.cursor()
        
        for record in telemetry_data.get('vehicles', []):
            cursor.execute("""
                INSERT INTO StagingFleetTelemetry (
                    VehicleID, Latitude, Longitude, SpeedMPH, 
                    FuelLevelPct, EngineTempF, TirePressurePSI, 
                    DiagnosticCodes, RecordedAt, IsProcessed
                ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, 0)
            """, 
                record['vehicle_id'],
                record['gps']['lat'],
                record['gps']['lon'],
                record['speed'],
                record['fuel_level'],
                record['engine_temp'],
                record.get('tire_pressure'),
                ','.join(record.get('diagnostic_codes', [])),
                datetime.fromisoformat(record['timestamp'])
            )
        
        conn.commit()
        print(f"Ingested {len(telemetry_data.get('vehicles', []))} telemetry records")
    
    # Trigger processing stored procedure
    with pyodbc.connect(conn_str) as conn:
        cursor = conn.cursor()
        cursor.execute("EXEC ProcessStagedTelemetry")
        conn.commit()

if __name__ == "__main__":
    ingest_telematics_data()
```

## Alerting Configuration

### SQL Server Agent Job for Threshold Alerts

```sql
-- Stored procedure for automated alerting
CREATE PROCEDURE CheckKPIThresholdsAndAlert
AS
BEGIN
    DECLARE @AlertRecipients VARCHAR(500) = 'ops-team@company.com';
    DECLARE @AlertSubject VARCHAR(200);
    DECLARE @AlertBody NVARCHAR(MAX);
    
    -- Check 1: Warehouse capacity exceeding 85%
    IF EXISTS (
        SELECT 1 FROM FactWarehouseOperations wo
        INNER JOIN DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
        WHERE wo.TimeKey >= CAST(FORMAT(DATEADD(HOUR, -1, GETUTCDATE()), 'yyyyMMddHHmm') AS INT)
        GROUP BY w.WarehouseName, w.HourlyCapacity
        HAVING SUM(wo.QuantityHandled) > (w.HourlyCapacity * 0.85)
    )
    BEGIN
        SET @AlertSubject = 'ALERT: Warehouse Capacity Threshold Exceeded';
        SET @AlertBody = N'One or more warehouses exceeded 85% capacity in the last hour. Check dashboard for details.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = @AlertRecipients,
            @subject = @AlertSubject,
            @body = @AlertBody;
    END
    
    -- Check 2: Fleet idle time exceeding 15%
    IF EXISTS (
        SELECT 1 FROM FactFleetTrips ft
        WHERE ft.DepartureTimeKey >= CAST(FORMAT(DATEADD(HOUR, -4, GETUTCDATE()), 'yyyyMMddHHmm') AS INT)
        GROUP BY ft.VehicleKey
        HAVING (SUM(ft.IdleTimeMinutes) * 1.0 / SUM(ft.IdleTimeMinutes + (ft.DistanceMiles / NULLIF(ft.AverageSpeedMPH, 0) * 60))) > 0.15
    )
    BEGIN
        SET @AlertSubject = 'ALERT: Fleet Idle Time Exceeds 15%';
        SET @AlertBody = N'One or more vehicles have excessive idle time. Review fleet utilization dashboard.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = @AlertRecipients,
            @subject = @AlertSubject,
            @body = @AlertBody;
    END
    
    -- Check 3: Product gravity zone misalignment
    DECLARE @MisalignedCount INT;
    SELECT @MisalignedCount = COUNT(*)
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity pg ON wo.ProductKey = pg.ProductKey
    INNER JOIN DimStorageZone sz ON wo.StorageZoneKey = sz.StorageZoneKey
    WHERE pg.OptimalZoneType <> sz.ZoneType
        AND wo.TimeKey >= CAST(FORMAT(DATEADD(DAY, -1, GETUTCDATE()), 'yyyyMMddHHmm') AS INT);
    
    IF @MisalignedCount > 100
    BEGIN
        SET @AlertSubject = 'ALERT: High Product Gravity Misalignment';
        SET @AlertBody = N'Over 100 products are stored in non-optimal gravity zones. Consider running zone rebalancing.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = @AlertRecipients,
            @subject = @AlertSubject,
            @body = @AlertBody;
    END
END;
```

## Troubleshooting

### Common Issues

**Power BI refresh fails with "Unable to connect":**
- Verify SQL Server allows remote connections
- Check firewall rules (port 1433)
- Ensure service account has db_datareader permissions
- Test connection string in SSMS first

**Slow dashboard performance:**
```sql
-- Check for missing indexes
SELECT 
    OBJECT_NAME(s.object_id) AS TableName,
    i.name AS IndexName,
    s.user_seeks,
    s.user_scans,
    s.last_user_seek,
    s.last_user_scan
FROM sys.dm_db_index_usage_stats s
INNER JOIN sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id
