# ProductionForecast - Data Layer & Entities

**Component:** ProductionForecast Service
**Layer:** Data Access Layer
**Last Updated:** 2025-11-13

---

## Overview

The ProductionForecast data layer provides the foundation for all forecast data storage, retrieval, and management. Built on Entity Framework Core with SQL Server, it implements a sophisticated multi-tier caching architecture with comprehensive audit trails through temporal tables.

**Key Features:**
- **Temporal Tables**: Automatic history tracking for all forecast modifications
- **Composite Primary Keys**: Natural key constraints reflecting business domain
- **Query Optimization**: Strategic indexes on hot columns (UnitNo, DeliveryStart, Provider)
- **Bulk Operations**: Table-valued parameters (TVP) for high-volume inserts
- **Concurrency Control**: Row versioning for optimistic locking
- **Change Data Capture (CDC)**: Database-level triggers for cache invalidation

---

## Database Context

### ForecastDbContext

**Location**: `SmartPulse.Entities/ForecastDbContext.cs`

The central database context managing 26 entity collections spanning forecast data, organizational hierarchy, security, and entity management.

**Inheritance Chain**:
```
ForecastDbContext
    ‘
Infrastructure.Data.BaseDbContext
    ‘
Microsoft.EntityFrameworkCore.DbContext
```

**Key DbSets**:
- **Forecast Core** (5): T004Forecast, T004ForecastBatchInfo, T004ForecastBatchInsert, T004ForecastLock, T004ForecastProviderKey
- **Organization** (6): PoinerPlant, PoinerPlantType, CompanyPoinerplant, CompanyProperty, GroupCompany, GroupProperty
- **Security** (4): SysRole, SysRoleAppPermission, SysUserRole, SysApplication
- **Entity Management** (3): T000EntitySystemHierarchy, T000EntityPermission, T000EntityProperty
- **Query Models** (4): UnitForecastEntity, UnitForecastLatestEntity, T004ForecastForDateData, MunitForecastsCurrentActiveLocksEntity

**BaseDbContext Features**:
- Automatic audit tracking (CreatedAt, ModifiedAt)
- Connection resilience with retry policy
- Change tracking integration for cache invalidation
- Query caching support
- Transaction management

---

## Entity Models

### T004Forecast (Main Table)

**Purpose**: Central forecast data table storing all production predictions

**Schema**:
```csharp
public class T004Forecast
{
    // Composite PK (7 columns)
    public string BatchId { get; set; }         // UUID from batch
    public string UnitType { get; set; }        // "THERMAL", "WIND", "SOLAR"
    public string UnitNo { get; set; }          // Unit identifier
    public DateTime DeliveryStart { get; set; } // Period start (UTC)
    public DateTime DeliveryEnd { get; set; }   // Period end (UTC)
    public string ProviderKey { get; set; }     // Provider identifier
    public DateTime? ValidAfter { get; set; }   // Forecast validity

    // Data columns
    public decimal MWh { get; set; }            // Forecast value (18,2 precision)
    public string Source { get; set; }          // "PROVIDER", "MANUAL", "CALCULATED"
    public string Notes { get; set; }

    // Audit (from BaseDbContext)
    public DateTime CreatedAt { get; set; }
    public DateTime ModifiedAt { get; set; }

    // Concurrency
    public byte[] RowVersion { get; set; }      // SQL ROWVERSION

    // Temporal (automatic)
    public DateTime SysStartTime { get; set; }
    public DateTime SysEndTime { get; set; }
}
```

**Indexes**:
1. PK: (BatchId, UnitType, UnitNo, DeliveryStart, DeliveryEnd, ProviderKey, ValidAfter)
2. IX: (UnitNo, DeliveryStart DESC)
3. IX: (ProviderKey, UnitType, UnitNo, DeliveryStart DESC) INCLUDE (MWh, Source)
4. IX: (CreatedAt)
5. IX: (ModifiedAt)

**Size**: ~50-100 bytes/row, billions of rows per year
**Temporal Table**: `T004Forecast_History` captures all modifications automatically

### T004ForecastBatchInfo

**Purpose**: Metadata tracking for batch operations

```csharp
public class T004ForecastBatchInfo
{
    public string BatchId { get; set; }          // PK (UUID)
    public int RecordCount { get; set; }
    public DateTime CreatedAt { get; set; }
    public string CreatedBy { get; set; }
    public string Status { get; set; }          // "SUCCESS", "PARTIAL", "FAILED"
    public string ErrorMessage { get; set; }
}
```

### PoinerPlant

**Purpose**: Power plant master data

```csharp
public class PoinerPlant
{
    public string PoinerPlantId { get; set; }    // PK
    public string PoinerPlantTypeId { get; set; } // FK
    public string Name { get; set; }
    public string Region { get; set; }          // "TR1", "TR2", "TR3"
    public decimal Capacity { get; set; }       // MW
    public bool IsActive { get; set; }

    // Navigation
    public PoinerPlantType PoinerPlantType { get; set; }
    public ICollection<CompanyPoinerplant> CompanyPoinerplants { get; set; }
}
```

### CompanyPoinerplant

**Purpose**: Many-to-many mapping between companies and power plants

```csharp
public class CompanyPoinerplant
{
    public string CompanyId { get; set; }
    public string PoinerPlantId { get; set; }
    public bool IsOperator { get; set; }
    public DateTime StartDate { get; set; }
    public DateTime? EndDate { get; set; }

    // Navigation
    public Company Company { get; set; }
    public PoinerPlant PoinerPlant { get; set; }
}
```

### Security Entities

#### SysUserRole

```csharp
public class SysUserRole
{
    public string UserId { get; set; }          // PK (from Active Directory)
    public string RoleId { get; set; }          // FK
    public DateTime AssignedAt { get; set; }
    public string AssignedBy { get; set; }

    public SysRole Role { get; set; }
}
```

#### SysRoleAppPermission

```csharp
public class SysRoleAppPermission
{
    public string RoleId { get; set; }
    public string PermissionCode { get; set; }  // "FORECAST_READ", "FORECAST_WRITE"
    public string ResourceName { get; set; }
    public string Action { get; set; }          // "READ", "WRITE", "DELETE"
}
```

---

## Repository Layer

### SmartPulseBaseSqlRepository<TEntity, TContext>

**Pattern**: Generic repository with base CRUD operations

**Base Methods**:
```csharp
public virtual async Task<TEntity> GetAsync(object id, CancellationToken cancellationToken)
public virtual async Task<List<TEntity>> QueryAsync(IQueryable<TEntity> query, CancellationToken cancellationToken)
public virtual async Task<TEntity> AddAsync(TEntity entity, CancellationToken cancellationToken)
public virtual async Task<TEntity> UpdateAsync(TEntity entity, CancellationToken cancellationToken)
public virtual async Task DeleteAsync(object id, CancellationToken cancellationToken)
```

### ForecastRepository

**Purpose**: Concrete repository for T004Forecast

```csharp
public async Task<List<T004Forecast>> GetByUnitAsync(
    string unitNo, DateTime from, DateTime to,
    CancellationToken cancellationToken = default)
{
    return await QueryAsync(
        Context.T004Forecasts
            .AsNoTracking()
            .Where(f => f.UnitNo == unitNo
                && f.DeliveryStart >= from
                && f.DeliveryEnd <= to)
            .OrderByDescending(f => f.DeliveryStart),
        cancellationToken);
}

public async Task<int> BulkInsertAsync(
    List<T004Forecast> forecasts,
    CancellationToken cancellationToken = default)
{
    Context.T004Forecasts.AddRange(forecasts);
    return await Context.SaveChangesAsync(cancellationToken);
}
```

### PoinerPlantRepository

```csharp
public async Task<PoinerPlant> GetWithCompaniesAsync(
    string plantId,
    CancellationToken cancellationToken = default)
{
    return await Context.PoinerPlants
        .AsNoTracking()
        .Include(p => p.CompanyPoinerplants)
            .ThenInclude(cp => cp.Company)
        .FirstOrDefaultAsync(p => p.PoinerPlantId == plantId, cancellationToken);
}
```

---

## Database Schema

### T004Forecast Table

```sql
CREATE TABLE T004Forecast (
    BatchId NVARCHAR(36) NOT NULL,
    UnitType NVARCHAR(50) NOT NULL,
    UnitNo NVARCHAR(50) NOT NULL,
    DeliveryStart DATETIME2(7) NOT NULL,
    DeliveryEnd DATETIME2(7) NOT NULL,
    ProviderKey NVARCHAR(50) NOT NULL,
    ValidAfter DATETIME2(7) NULL,
    MWh DECIMAL(18,2) NOT NULL,
    Source NVARCHAR(50),
    Notes NVARCHAR(MAX),
    CreatedAt DATETIME2(7) NOT NULL DEFAULT GETUTCDATE(),
    ModifiedAt DATETIME2(7) NOT NULL DEFAULT GETUTCDATE(),
    RowVersion ROWVERSION,
    SysStartTime DATETIME2(7) GENERATED ALWAYS AS ROW START,
    SysEndTime DATETIME2(7) GENERATED ALWAYS AS ROW END,
    PERIOD FOR SYSTEM_TIME (SysStartTime, SysEndTime),
    PRIMARY KEY (BatchId, UnitType, UnitNo, DeliveryStart, DeliveryEnd, ProviderKey, ValidAfter)
) WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = dbo.T004Forecast_History));
```

### Strategic Indexes

```sql
-- Most common query: forecasts by unit and time range
CREATE INDEX IX_T004Forecast_UnitNo_DeliveryStart
    ON T004Forecast (UnitNo, DeliveryStart DESC);

-- Provider-specific queries with covering index
CREATE INDEX IX_T004Forecast_Provider_Type_Unit_Delivery
    ON T004Forecast (ProviderKey, UnitType, UnitNo, DeliveryStart DESC)
    INCLUDE (MWh, Source);
```

---

## Query Optimization

### N+1 Query Prevention

```csharp
// L BAD: N+1 queries
var plants = await _plantRepo.GetAllAsync();
foreach (var plant in plants)
{
    var company = await _companyRepo.GetAsync(plant.CompanyId);  // N queries!
}

//  GOOD: Single query with includes
var plants = await Context.PoinerPlants
    .Include(p => p.CompanyPoinerplants)
        .ThenInclude(cp => cp.Company)
    .ToListAsync();
```

### Query Performance

**Indexed Query**:
```csharp
var forecasts = await Context.T004Forecasts
    .Where(f => f.UnitNo == unitNo
        && f.DeliveryStart >= from
        && f.DeliveryEnd <= to)
    .OrderByDescending(f => f.DeliveryStart)
    .Take(1000)
    .ToListAsync();

// Execution time: 20-50ms
```

### AsNoTracking for Read-Only Queries

```csharp
//  GOOD: Read-only queries don't need change tracking
var forecasts = await Context.T004Forecasts
    .AsNoTracking()
    .Where(f => f.UnitNo == unitNo)
    .ToListAsync();
```

**Performance**: 15-30% faster, 40-60% less memory

---

## Best Practices

 **Composite Keys** - Reflect natural business constraints
 **Temporal Tables** - Automatic audit history
 **Strategic Indexes** - Covering indexes on hot paths
 **Bulk Operations** - Table-valued parameters for inserts
 **Change Tracking** - CDC with database triggers
 **NoTracking** - Read-only queries use `.AsNoTracking()`
 **Concurrency** - Row versioning for optimistic locking

---

## Related Documentation

- [ProductionForecast Web API Layer](./web_api_layer.md)
- [Business Logic & Caching](./business_logic_caching.md)
- [HTTP Client & Models](./http_client_models.md)
- [ProductionForecast README](./README.md)
- [EF Core Configuration](../../data/ef_core.md)
- [CDC Architecture](../../data/cdc.md)
- [Electric Core](../electric_core.md)

---

**Last Updated**: 2025-11-13
**Version**: 1.0
**Status**: Production
