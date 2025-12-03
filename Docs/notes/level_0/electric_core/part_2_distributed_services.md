# Electric.Core - Level_0 Notes - PART 2

## 7. DISTRIBUTEDDATA

### Purpose
Redis-based distributed cache system with change data capture (CDC), version tracking, JSON patch-based synchronization, and automatic conflict resolution. Core pattern for distributed state management across multiple services with optimistic locking and event-driven updates.

### 7.1 DistributedDataManager<T> where T : IDistributedData

- **Purpose**: Abstract base class for distributed cache managers with Redis backend, version-based optimistic locking, JSON patch delta tracking, change buffer for late-subscriber synchronization, and background sync service integration.
- **Patterns**: Repository pattern, Cache-Aside pattern, Optimistic locking (VersionId), JSON Patch (RFC 6902), Change Data Capture (CDC), Event-Driven Architecture, Double-checked locking
- **Internal Dependencies**:
  - DistributedDataConnection (7.5)
  - ConcurrentObservableDictionary<TKey, TValue> (Collections.Concurrent 4.3)
  - DataWithLock<T> (7.2)
- **External Dependencies**:
  - Newtonsoft.Json (JObject/JToken for patch operations)
  - StackExchange.Redis (IConnectionMultiplexer)
  - Electric.Core.Text.Json (DecimalFormatConverter, FloatFormatConverter)
- **Threading**:
  - **Thread-safe**: ConcurrentObservableDictionary for data cache (line 33)
  - **Per-key locking**: SemaphoreSlim(1,1) per DataKey in DataWithLock<T> (line 765)
  - **Retry logic**: 100 iterations with 50ms delay for Redis operations (lines 259-272, 300-318, 482-494, 501-514, 744-757)
- **Public Properties**:
  - `DataManagerId: Guid` - Unique manager instance identifier (read-only, line 27)
  - `PartitionKey: string` - Abstract partition key (must override, line 29)
  - `Section: string` - Redis hash section name (read-only, line 31)
  - `WriteConnectionRestoredLog: bool` - Enable logging on Redis reconnect (settable, default: false, line 51)
- **Public Events**:
  - `DistributedDataChanged: EventHandler<DistributedDataChangedInfo>?` - Fired after data changes applied (line 35)
  - `SyncErrorTaken: EventHandler<DistributedDataChangedInfo>?` - Fired on version mismatch/sync errors (line 36)
- **Protected Abstract Methods**:
  - `CreateEmpty(dataKey: string) -> T` - Factory for empty data objects (line 220)
- **Public Methods**:
  - `DistributedDataManager(distributedDataConnection: DistributedDataConnection, section: string)` - Constructor; registers JSON converters (lines 38-49)
  - `GetAsync(dataKey: string) -> ValueTask<T?>` - Get from cache or Redis; null if not exists (line 222-225)
  - `GetOrDefaultAsync(dataKey: string) -> ValueTask<T>` - Get or CreateEmpty fallback (lines 227-230)
  - `SetAsync(data: T, updateAction: Func<T, Task>, keyExpireTime: TimeSpan? = null, publish: bool = true, isTest: bool = false, copyToKey: string? = null) -> Task<bool>` - Update with optimistic locking (lines 232-235)
  - `SetAsync(dataKey: string, updateAction: Func<T, Task>, keyExpireTime: TimeSpan? = null, publish: bool = true, isTest: bool = false, copyToKey: string? = null) -> Task<bool>` - Core update method (lines 237-339)
  - `DoDummyConnectionToSubscribe() -> void` - Initialize Redis pub/sub channel (line 470)
  - `GetDistributedDataChangeEnumerationAsync(dataKey: string = "*", stoppingToken: CancellationToken = default, maxChangeBufferCount: int = 50) -> IAsyncEnumerable<DistributedDataChangedInfo>` - Subscribe to CDC stream (lines 472-478)
  - `PublishAsync(distributedDataChangedInfo: DistributedDataChangedInfo) -> Task` - Manual publish with retry (lines 480-497)
  - `KeyExpireAsync(distributedDataChangedInfo: DistributedDataChangedInfo, expiry: TimeSpan, publish: bool, expireWhen: ExpireWhen = HasNoExpiry) -> Task` - Set key TTL (lines 499-517)
  - `ApplyDataChangesAsync(distributedDataChangedInfo: DistributedDataChangedInfo) -> Task` - Apply remote changes (public interface, lines 524-527)
  - `FlushChangeBufferAsync(fieldMaxAge: TimeSpan) -> Task` - Cleanup old buffered changes (lines 661-677)
  - `FlushDataAsync(maxAge: TimeSpan) -> Task` - Evict stale cache entries (lines 692-708)
  - `RemoveFromCache(dataKey: string) -> void` - Manual cache eviction (lines 687-690)
  - `CheckSyncVersionErrorsAsync() -> Task` - Validate VersionId consistency with Redis (lines 710-734)
  - `GetChangesBuffer(dataKey: string) -> DistributedField[]?` - Inspect change buffer (lines 653-659)
- **Protected Methods**:
  - `GetOrCreateDataWithLock(dataKey: string) -> DataWithLock<T>` - Lazy initialization of per-key cache/lock (lines 61-64)
  - `ApplyDeltaChanges3(sourceJToken: JToken?, source: T?, target: T, dataKey: string) -> (List<PatchItem>, List<PatchItem>, List<PatchItem>)` - Virtual hook for delta computation (lines 359-364)
  - `CreateTargetObjectClone(dataKey: string, oldItem: DataWithLock<T>) -> Task<(T, JToken?)>` - Virtual clone factory for update (lines 450-458)
  - `ApplyDataChangesInMemoryAsync(distributedDataChangedInfo: DistributedDataChangedInfo, newData: T) -> Task` - Virtual post-save hook (lines 519-522)
  - `DataIsChanged(distributedDataChangedInfo: DistributedDataChangedInfo) -> Task` - Fire DistributedDataChanged event (lines 583-592)
  - `SyncErrorHappenedAsync(dataKey: string) -> Task` - Fire SyncErrorTaken event (lines 736-740)
- **Public Static Methods**:
  - `CloseAllConnections(clearFromCache: bool = false) -> Task` - Global Redis connection disposal (line 685)
- **Architecture**:
  - **Cache structure**: ConcurrentObservableDictionary<string, DataWithLock<T>> - per dataKey (line 33)
  - **Optimistic locking**: VersionId incremented atomically in Redis (lines 263-275)
  - **Change buffer**: Late subscribers receive missed changes (lines 539, 606-650)
  - **JSON Patch flow**: source → target → delta (add/remove/replace) → Redis HSET/HDEL (lines 366-438)
  - **Sync flow**: Redis pub/sub → ApplyDataChangesAsyncCore → change buffer or direct apply (lines 529-581)
- **Get Flow** (GetAsync):
  1. Check cache (line 82-83)
  2. SemaphoreSlim.WaitAsync (line 86)
  3. Double-checked cache lookup (lines 94-96)
  4. GetInitialFieldValuesFromDB (Redis HGETALL) with stability retry (lines 100, 156-188)
  5. SetDbValuesToNewCreatedObject → JObject (lines 104, 144-154)
  6. ApplyChangeBufferChangesToObject (lines 106, 131-142)
  7. Deserialize JObject → T (lines 110-111)
  8. Store in cache (line 113-114)
  9. Release semaphore (line 127)
- **Set Flow** (SetAsync):
  1. SemaphoreSlim.WaitAsync (line 246)
  2. CreateTargetObjectClone (clone from cache/Redis) (line 249)
  3. Execute updateAction(target) (line 251)
  4. ApplyDeltaChanges3 → compute JSON patches (line 253)
  5. IncrementVersionIdAsync (Redis HINCRBY) with retry (lines 263-275)
  6. ApplyDataChangesInMemoryAsync → update cache (line 289)
  7. SaveDistributedDataChangesAsync → Redis HSET/HDEL + pub/sub (lines 306)
  8. Release semaphore (line 335)
- **Sync Flow** (ApplyDataChangesAsyncCore):
  1. Check IsInitializingFromDB flag (lines 533-537)
  2. If initializing: buffer changes (lines 539, 606-650)
  3. If cached: apply buffered changes + new changes (lines 569-578)
  4. Version gap > 1: evict cache, fire SyncErrorTaken (lines 560-566)
  5. Fire DistributedDataChanged event (line 580)
- **Stability Retry Pattern** (GetInitialFieldValuesFromDB):
  - Problem: Redis HSET operations may not be atomic across multiple fields
  - Solution: Retry up to 10 times with 50ms delay, compare HLEN with received field count (lines 163-180)
  - Stable when: HLEN == received count (line 173-174)
- **Notes**:
  - **Performance**: Change buffer prevents full reload for late subscribers (lines 539, 606-650)
  - **Conflict resolution**: Last-write-wins (LWW) via VersionId; version gaps trigger cache eviction (lines 560-566)
  - **Pub/Sub pattern**: DistributedDataConnection publishes to "{PartitionKey}:{Section}" Redis channel (line 486)
  - **JSON converters**: DecimalFormatConverter, FloatFormatConverter (lines 45-48) for precision control
  - **Change buffer cleanup**: FlushChangeBufferAsync removes fields older than fieldMaxAge (lines 661-677)
  - **Cache eviction**: FlushDataAsync removes entries with ChangeTime > maxAge (lines 692-708)
  - **Connection resilience**: OnConnectionRestored event clears all caches (lines 41, 53-59)
  - **Test mode**: isTest=true skips Redis write (line 304-307)
  - **Copy-on-write**: copyToKey parameter copies data to another key (line 306)
  - **Retry strategy**: 100 iterations × 50ms = 5s max retry for Redis operations
  - **Use case**: Distributed cache for intraday market data (trades, orders, depths) with real-time synchronization across services

### 7.2 DataWithLock<T>

- **Purpose**: Per-key cache entry with locking, change buffer, and JToken source tracking for delta computation.
- **Patterns**: Per-key locking, Change buffer pattern
- **Internal Dependencies**: ConcurrentObservableDictionary (4.3)
- **Threading**: SemaphoreSlim(1,1) per instance (line 765)
- **Public Properties**:
  - `SemaphoreSlim: SemaphoreSlim` - Per-key lock (1,1) (read-only, line 765)
  - `Changes: ConcurrentObservableDictionary<string, DistributedField>` - Buffered field changes (read-only, line 767)
  - `SourceJToken: JToken?` - Cached JSON for delta computation (settable, line 769)
  - `Data: T?` - Cached data object (settable, line 771)
  - `IsInitializingFromDB: bool` - Flag to defer sync until initialization complete (settable, default: false, line 773)
- **Notes**:
  - **Change buffer**: Stores DistributedField (path, value, versionId) for late subscribers (line 767)
  - **JToken cache**: Avoids re-serialization for delta computation (line 769)
  - **Initialization flag**: Defers ApplyDataChangesAsync until GetAsync completes (line 773)

### 7.3 IDistributedData

- **Purpose**: Base interface for distributed data objects with partition/key/version tracking.
- **Public Properties**:
  - `PartitionKey: string` - First-level partition (e.g. "Intraday:EPIAS")
  - `DataKey: string` - Second-level key (e.g. "MCP_20231115")
  - `Section: string` - Third-level section (e.g. "Trades")
  - `VersionId: long` - Optimistic lock version (settable)
  - `ChangeTime: DateTimeOffset` - Last modification timestamp (read-only)
- **Notes**:
  - **Redis key format**: `{PartitionKey}:{Section}:{DataKey}` (e.g. "Intraday:EPIAS:Trades:MCP_20231115")
  - **VersionId semantics**: Incremented atomically via HINCRBY; gaps indicate sync errors

### 7.4 IDistributedDataManager

- **Purpose**: Public interface for DistributedDataManager<T> (non-generic methods).
- **Public Properties**:
  - `PartitionKey: string` - Partition key (read-only)
  - `Section: string` - Section name (read-only)
  - `DataManagerId: Guid` - Manager instance ID (read-only)
- **Public Methods**:
  - `DoDummyConnectionToSubscribe() -> void`
  - `GetDistributedDataChangeEnumerationAsync(dataKey: string = "*", stoppingToken: CancellationToken = default, maxChangeBufferCount: int = 50) -> IAsyncEnumerable<DistributedDataChangedInfo>`
  - `ApplyDataChangesAsync(distributedDataChangedInfo: DistributedDataChangedInfo) -> Task`
  - `FlushChangeBufferAsync(fieldMaxAge: TimeSpan) -> Task`
  - `GetChangesBuffer(dataKey: string) -> DistributedField[]?`
  - `FlushDataAsync(maxAge: TimeSpan) -> Task`
  - `RemoveFromCache(dataKey: string) -> void`
  - `CheckSyncVersionErrorsAsync() -> Task`
  - `PublishAsync(distributedDataChangedInfo: DistributedDataChangedInfo) -> Task`
  - `KeyExpireAsync(distributedDataChangedInfo: DistributedDataChangedInfo, expiry: TimeSpan, publish: bool, expireWhen: ExpireWhen = HasNoExpiry) -> Task`
- **Notes**: Used by DistributedDataSyncService (7.6) for polymorphic background sync

### 7.5 DistributedDataConnection

- **Purpose**: Abstract base class for Redis connection management with pub/sub, field CRUD, version increment, and CDC stream.
- **Patterns**: Abstract Factory pattern, Repository pattern
- **Threading**: Implementations must be thread-safe
- **Public Events**:
  - `OnConnectionRestored: EventHandler<ConnectionFailedEventArgs>?` - Fired on Redis reconnect (line 8)
- **Public Abstract Methods**:
  - `GetFieldValuesAsync(partitionKey: string, section: string, dataKey: string, version: string = "latest") -> IAsyncEnumerable<DistributedField>` - HGETALL wrapper (line 10)
  - `GetGivenFieldValuesAsync(partitionKey: string, section: string, dataKey: string, hashFields: IEnumerable<string>, version: string = "latest") -> IAsyncEnumerable<DistributedField>` - HMGET wrapper (line 12)
  - `IncrementVersionIdAsync(partitionKey: string, section: string, dataKey: string, increment: int = 1) -> Task<long>` - HINCRBY wrapper (line 14)
  - `SaveDistributedDataChangesAllPatchAsync(distributedDataChangedInfo: DistributedDataChangedInfo, publish: bool, keyExpireTime: TimeSpan, copyToKey: string) -> Task` - Full patch (line 16)
  - `SaveDistributedDataChangesAsync(distributedDataChangedInfo: DistributedDataChangedInfo, removedFields: List<PatchItem>, addedOrReplacedFields: List<PatchItem>, publish: bool, keyExpireTime: TimeSpan?, copyToKey: string?) -> Task` - Delta patch (line 18)
  - `GetDistributedDataChangeEnumerationAsync(partitionKey: string, section: string, dataKey: string = "*", maxChangeBufferCount: int = 50, stoppingToken: CancellationToken = default) -> IAsyncEnumerable<DistributedDataChangedInfo>` - Pub/sub stream (line 20)
  - `KeyExpireAsync(distributedDataChangedInfo: DistributedDataChangedInfo, expiry: TimeSpan, publish: bool, expireWhen: ExpireWhen = HasNoExpiry) -> Task` - EXPIRE wrapper (line 22)
  - `PublishAsync(distributedDataChangedInfo: DistributedDataChangedInfo) -> Task` - PUBLISH wrapper (line 24)
  - `GetKeyCountAsync(partitionKey: string, section: string, dataKey: string) -> Task<long>` - HLEN wrapper (line 26)
  - `DoDummyConnectionToSubscribe(partitionKey: string) -> void` - Initialize pub/sub (line 28)
- **Notes**:
  - **Implementation**: RedisDistributedDataConnection (DistributedData/Redis/RedisDistributedDataConnection.cs)
  - **Pub/sub channel format**: `{PartitionKey}:{Section}` (e.g. "Intraday:EPIAS:Trades")

### 7.6 DistributedDataSyncService

- **Purpose**: Background service (HostedService) for automatic synchronization of all registered IDistributedDataManager instances with Redis pub/sub CDC, change buffer flush, and version error detection.
- **Patterns**: Background Service pattern, Fan-out pattern (per-manager tasks), AutoConcurrentPartitionedQueue for per-dataKey ordering
- **Internal Dependencies**:
  - IDistributedDataManager (7.4)
  - DistributedDataSyncServiceInfo (7.7)
  - AutoConcurrentPartitionedQueue<T> (4.1)
- **Threading**:
  - **Per-manager tasks**: 2 tasks per manager (GetAndApplyDistributedDataManagerChangesAsync + FlushDistributedDataManagerLocalCacheAsync) (line 19-21)
  - **Partitioned queue**: AutoConcurrentPartitionedQueue per manager ensures per-dataKey ordering (line 32-43)
- **Constructor**:
  - `DistributedDataSyncService(distributedDataManagers: IEnumerable<IDistributedDataManager>, info: DistributedDataSyncServiceInfo? = default)` - Injected via DI (lines 11-15)
- **Protected Methods**:
  - `ExecuteAsync(stoppingToken: CancellationToken) -> Task` - Entry point; spawns tasks per manager (lines 17-22)
- **Private Methods**:
  - `GetAndApplyDistributedDataManagerChangesAsync(distributedDataManager: IDistributedDataManager, stoppingToken: CancellationToken) -> Task` - CDC subscription loop (lines 24-61)
  - `FlushDistributedDataManagerLocalCacheAsync(distributedDataManager: IDistributedDataManager, stoppingToken: CancellationToken) -> Task` - Background cleanup loop (lines 63-92)
- **CDC Sync Flow** (GetAndApplyDistributedDataManagerChangesAsync):
  1. Filter by SyncSections whitelist (lines 28-30)
  2. Create AutoConcurrentPartitionedQueue<DistributedDataChangedInfo> (line 32)
  3. await foreach GetDistributedDataChangeEnumerationAsync (line 49)
  4. Enqueue to partitioned queue (key = DataKey) (line 51)
  5. Callback: ApplyDataChangesAsync per item (line 37)
  6. Retry loop: 500ms delay on exception (line 59)
- **Cleanup Flow** (FlushDistributedDataManagerLocalCacheAsync):
  1. CheckSyncVersionErrorsAsync every SyncCheckVersionErrorInterval (lines 72-75)
  2. FlushChangeBufferAsync every GeneralWaitInterval (line 77)
  3. FlushDataAsync every DataFlushInterval (lines 79-82)
  4. Delay GeneralWaitInterval (line 90)
- **Notes**:
  - **Per-dataKey ordering**: AutoConcurrentPartitionedQueue ensures changes for same DataKey are applied sequentially (line 32)
  - **Section filtering**: SyncSections allows selective sync (e.g. only "Trades" section) (lines 28-30)
  - **Error handling**: Console.WriteLine for all exceptions (lines 40-42, 54-57, 84-86)
  - **Graceful shutdown**: CancellationToken propagation (line 17)
  - **Use case**: Automatically syncs all managers (e.g. OrderDataManager, TradesDataManager) in background

### 7.7 DistributedDataSyncServiceInfo

- **Purpose**: POCO configuration for DistributedDataSyncService intervals and section filtering.
- **Public Properties**:
  - `SyncSections: Type[]?` - Whitelist of manager types to sync (default: null = all)
  - `MaxChangeBufferCount: int` - Max buffered changes per dataKey (default: 50)
  - `ChangeBufferFieldMaxAge: TimeSpan` - Max age for buffered fields (default: varies)
  - `DataFlushMaxAge: TimeSpan` - Max age for cached data (default: varies)
  - `DataFlushInterval: TimeSpan` - Frequency of FlushDataAsync (default: varies)
  - `SyncCheckVersionErrorInterval: TimeSpan` - Frequency of CheckSyncVersionErrorsAsync (default: varies)
  - `GeneralWaitInterval: TimeSpan` - Delay between cleanup iterations (default: varies)
- **Notes**: Defaults defined in constructor (not shown in snippet)

### 7.8 DistributedDataChangedInfo

- **Purpose**: Immutable struct for change event payload with VersionId, PatchItems, and metadata.
- **Patterns**: Value Object pattern, Event Sourcing pattern
- **Public Properties**:
  - `DataManagerId: Guid` - Source manager ID (init-only)
  - `PartitionKey: string` - Partition key (init-only)
  - `DataKey: string` - Data key (init-only)
  - `Section: string` - Section name (init-only)
  - `VersionId: long` - Version after change (settable)
  - `ChangeTime: DateTimeOffset` - UTC timestamp (init-only, line 22)
  - `PatchItems: List<PatchItem>?` - JSON patch operations (init-only, line 23)
- **Constructor**:
  - `DistributedDataChangedInfo(dataManagerId: Guid, partitionKey: string, dataKey: string, section: string, versionId: long, patchItems: List<PatchItem>?)` - All fields (lines 15-24)
- **Notes**:
  - **Immutable**: All fields init-only except VersionId (settable for rare cases)
  - **PatchItems format**: List of {Op: add/remove/replace, Path: "fieldName", Value: "serializedValue"}
  - **Event payload**: Published to Redis pub/sub "{PartitionKey}:{Section}" channel

---

## 8. ELECTRICITY

### Purpose
Domain models, enums, extensions, and specialized DistributedDataManager implementations for electricity market data (intraday trading, orders, positions, depths, contracts). Core business logic for EPIAS (Turkey), NORDPOOL (Nordic), OMIE (Spain) market integration.

### 8.1 Market Enum

- **Purpose**: Market type discriminator.
- **Values**:
  - `None = 0`
  - `Intraday = 1`
- **Notes**: Extensible for DayAhead, Balancing, etc.

### 8.2 TradeDirection Enum

- **Purpose**: Order side discriminator.
- **Values**: (not shown in snippet, likely Buy/Sell/None)

### 8.3 UnitTypes Enum

- **Purpose**: Energy/power unit discriminator (MWh, MW, etc.)
- **Values**: (not shown in snippet)

### 8.4 PlatformType Enum

- **Purpose**: Market platform discriminator (EPIAS, NORDPOOL, OMIE)
- **Values**: (not shown in snippet)

### 8.5 CurrencyCodes Enum

- **Purpose**: Currency discriminator (TRY, EUR, NOK, etc.)
- **Values**: (not shown in snippet)

### 8.6 Operator Enum

- **Purpose**: Comparison operator for filters (Equals, GreaterThan, etc.)
- **Values**: (not shown in snippet)

### 8.7 IMarketIdentifier Interface

- **Purpose**: Base interface for market-aware entities.
- **Public Properties**:
  - `Market: Market` - Market type
  - `PlatformType: PlatformType` - Platform identifier
- **Notes**: Used by IntradayOrder, IntradayTrade, etc.

### 8.8 IMarketIdentifierExtension

- **Purpose**: Extension methods for IMarketIdentifier.
- **Public Methods**: (not shown in snippet, likely market-specific helpers)

### 8.9 OrderIdentity

- **Purpose**: Composite key for order identification.
- **Public Properties**: (not shown in snippet, likely OrderId + ContractId + PlatformType)

### 8.10 RegionDataHelper

- **Purpose**: Regional configuration helper (time zones, holidays, market hours).
- **Public Methods**: (not shown in snippet)

### 8.11 Electricity Domain Models (Intraday)

All models inherit from IDistributedData (7.3) and are managed by specialized DistributedDataManager<T> (7.1) implementations.

#### 8.11.1 IntradayTrade

- **Purpose**: Single trade execution record.
- **Public Properties**: (typical)
  - `TradeId: string` - Unique trade ID
  - `ContractId: string` - Contract identifier
  - `Price: decimal` - Execution price
  - `Quantity: decimal` - Execution quantity
  - `TradeTime: DateTimeOffset` - Execution timestamp
  - `BuyOrderId: string` - Buy side order ID
  - `SellOrderId: string` - Sell side order ID
  - `PlatformType: PlatformType` - Market platform
  - Plus: PartitionKey, DataKey, Section, VersionId, ChangeTime (from IDistributedData)
- **Manager**: TradesDataManager (8.12.1)

#### 8.11.2 IntradayTrades

- **Purpose**: Collection of trades (aggregate).
- **Public Properties**: `Trades: List<IntradayTrade>`
- **Manager**: TradesDataManager (8.12.1)

#### 8.11.3 IntradayOrder

- **Purpose**: Order book entry.
- **Public Properties**: (typical)
  - `OrderId: string` - Unique order ID
  - `ContractId: string` - Contract identifier
  - `Price: decimal` - Order price
  - `Quantity: decimal` - Order quantity
  - `Direction: TradeDirection` - Buy/Sell
  - `OrderTime: DateTimeOffset` - Order timestamp
  - `OrderStatus: string` - Status (Active, Filled, Cancelled)
  - `PlatformType: PlatformType` - Market platform
  - Plus: IDistributedData fields
- **Manager**: OrderDataManager (8.12.2)

#### 8.11.4 IntradayOrderLimit

- **Purpose**: Order limits per participant (max quantity, max price, etc.)
- **Manager**: OrderLimitDataManager (8.12.3)

#### 8.11.5 IntradayPosition

- **Purpose**: Net position per participant/contract.
- **Public Properties**: (typical)
  - `CompanyId: string` - Participant ID
  - `ContractId: string` - Contract identifier
  - `NetPosition: decimal` - Net quantity (positive = long, negative = short)
  - `AveragePrice: decimal` - Weighted average price
  - Plus: IDistributedData fields
- **Manager**: PositionDataManager (8.12.4)

#### 8.11.6 IntradayDepth / IntradayDepths

- **Purpose**: Order book depth (bid/ask levels).
- **Public Properties**: (typical for IntradayDepth)
  - `ContractId: string` - Contract identifier
  - `BidLevels: List<PriceLevel>` - Bid side (buy orders)
  - `AskLevels: List<PriceLevel>` - Ask side (sell orders)
  - `PriceLevel: {Price: decimal, Quantity: decimal, OrderCount: int}`
- **Manager**: DepthsDataManager (8.12.5)

#### 8.11.7 IntradayLimits

- **Purpose**: Trading limits (price limits, quantity limits, circuit breakers).
- **Manager**: LimitsDataManager (8.12.6)

#### 8.11.8 IntradayMarketValue

- **Purpose**: Market statistics (last price, volume, VWAP, high/low).
- **Manager**: MarketValueDataManager (8.12.7)

#### 8.11.9 IntradayContracts

- **Purpose**: Active contracts metadata (delivery periods, units, status).
- **Manager**: ContractsDataManager (8.12.8)

#### 8.11.10 IntradaySystemStatus

- **Purpose**: Market system status (open/closed, auction phase, emergency).
- **Manager**: SystemStatusManager (8.12.9)

#### 8.11.11 IntradayServerInfo

- **Purpose**: Server metadata (version, uptime, health).
- **Manager**: ServerInfoManager (8.12.10)

#### 8.11.12 IntradayPreOrderRequest

- **Purpose**: Pre-order validation request/response.
- **Manager**: PreOrderRequestDataManager (8.12.11)

#### 8.11.13 CompanyPositionDetails

- **Purpose**: Detailed position breakdown per company.
- **Manager**: CompanyPositionDetailsDataManager (8.12.12)

#### 8.11.14 AuctionCollectorInfo

- **Purpose**: Auction metadata (auction ID, phase, clearing price).
- **Manager**: AuctionCollectorDataManager (8.12.13)

### 8.12 Electricity DistributedDataManager Implementations

All managers inherit from DistributedDataManager<T> (7.1) and provide market-specific logic.

#### 8.12.1 TradesDataManager

- **Purpose**: Manages IntradayTrades with Redis backend.
- **Base Class**: DistributedDataManager<IntradayTrades>
- **Override**: CreateEmpty, PartitionKey, Section
- **Notes**: Used by trading services to cache/sync trades

#### 8.12.2 OrderDataManager

- **Purpose**: Manages IntradayOrder with Redis backend.
- **Base Class**: DistributedDataManager<IntradayOrder>
- **Override**: CreateEmpty, PartitionKey, Section
- **Notes**: Used by order matching services

#### 8.12.3 OrderLimitDataManager

- **Purpose**: Manages IntradayOrderLimit.
- **Base Class**: DistributedDataManager<IntradayOrderLimit>

#### 8.12.4 PositionDataManager

- **Purpose**: Manages IntradayPosition.
- **Base Class**: DistributedDataManager<IntradayPosition>

#### 8.12.5 DepthsDataManager

- **Purpose**: Manages IntradayDepths.
- **Base Class**: DistributedDataManager<IntradayDepths>

#### 8.12.6 LimitsDataManager

- **Purpose**: Manages IntradayLimits.
- **Base Class**: DistributedDataManager<IntradayLimits>

#### 8.12.7 MarketValueDataManager

- **Purpose**: Manages IntradayMarketValue.
- **Base Class**: DistributedDataManager<IntradayMarketValue>

#### 8.12.8 ContractsDataManager

- **Purpose**: Manages IntradayContracts.
- **Base Class**: DistributedDataManager<IntradayContracts>

#### 8.12.9 SystemStatusManager

- **Purpose**: Manages IntradaySystemStatus.
- **Base Class**: DistributedDataManager<IntradaySystemStatus>

#### 8.12.10 ServerInfoManager

- **Purpose**: Manages IntradayServerInfo.
- **Base Class**: DistributedDataManager<IntradayServerInfo>

#### 8.12.11 PreOrderRequestDataManager

- **Purpose**: Manages IntradayPreOrderRequest.
- **Base Class**: DistributedDataManager<IntradayPreOrderRequest>

#### 8.12.12 CompanyPositionDetailsDataManager

- **Purpose**: Manages CompanyPositionDetails.
- **Base Class**: DistributedDataManager<CompanyPositionDetails>

#### 8.12.13 AuctionCollectorDataManager

- **Purpose**: Manages AuctionCollectorInfo.
- **Base Class**: DistributedDataManager<AuctionCollectorInfo>

#### 8.12.14 MarketDataManager

- **Purpose**: Manages MarketData (composite).
- **Base Class**: DistributedDataManager<MarketData>

### 8.13 IDataIdentifier Interface

- **Purpose**: Base interface for data identifiers (likely ContractId, CompanyId, etc.)
- **Public Properties**: (not shown in snippet)

### 8.14 MarketData

- **Purpose**: Composite data model (aggregates trades, orders, depths, etc.)
- **Public Properties**: (not shown in snippet)

### 8.15 GroupCompany.GroupCompanyCache

- **Purpose**: Cache for group company relationships (parent-child companies).
- **Public Methods**: (not shown in snippet)

### 8.16 Electricity Extensions

#### 8.16.1 RegionExtension

- **Purpose**: Extension methods for regional calculations (time zone conversion, market hours).
- **Public Methods**: (not shown in snippet)

#### 8.16.2 IntradayDepthExtension

- **Purpose**: Extension methods for depth calculations (best bid/ask, spread, etc.)
- **Public Methods**: (not shown in snippet)

#### 8.16.3 MarketDataExtensions

- **Purpose**: Extension methods for MarketData aggregations.
- **Public Methods**: (not shown in snippet)

#### 8.16.4 RedisDistributedDataConnectionExtensions

- **Purpose**: Extension methods for Redis connection setup (EPIAS, NORDPOOL, OMIE).
- **Public Methods**: (likely AddEpiasDistributedData, AddNordpoolDistributedData, AddOmieDistributedData)
- **Notes**: Registers managers in DI container

### 8.17 Market-Specific DI Extensions

#### 8.17.1 EPIAS.IServiceCollectionExtension

- **Purpose**: DI registration for EPIAS (Turkey) market services.
- **Public Methods**: (likely AddEpiasServices)
- **Notes**: Registers managers: OrderDataManager, TradesDataManager, DepthsDataManager, etc.

#### 8.17.2 NORDPOOL.IServiceCollectionExtension

- **Purpose**: DI registration for NORDPOOL (Nordic) market services.
- **Public Methods**: (likely AddNordpoolServices)

#### 8.17.3 OMIE.IServiceCollectionExtension

- **Purpose**: DI registration for OMIE (Spain) market services.
- **Public Methods**: (likely AddOmieServices)

---

## 9. TRACKCHANGES

### Purpose
SQL Server Change Data Capture (CDC) integration using CHANGETABLE for real-time database change tracking. Core pattern for syncing database changes to distributed cache (DistributedDataManager) or message bus (Apache Pulsar).

### 9.1 ChangeTracker : IChangeTracker

- **Purpose**: Core CDC tracker using SQL Server CHANGETABLE with version-based polling, exponential backoff, and IAsyncEnumerable streaming.
- **Patterns**: Change Data Capture (CDC) pattern, Polling pattern, Iterator pattern (IAsyncEnumerable), Exponential backoff
- **Internal Dependencies**: IServiceProvider (DI), DbContext (EF Core)
- **Threading**: Thread-safe (scoped DbContext per iteration)
- **Constructor**:
  - `ChangeTracker(serviceProvider: IServiceProvider)` - DI injection (lines 14-17)
- **Public Methods**:
  - `TrackChangesAsync(tableName: string, selectColumns: string? = null, extraFilter: string? = null, cancellationToken: CancellationToken = default, expectedColumnCount: int = 8, waitTimeBetweenQueriesAction: Func<int, TimeSpan>? = null, changeVersionIdSelect: string? = null, changeOperationSelect: string? = null, sqlBeforeSelect: string? = null) -> IAsyncEnumerable<List<ChangeItem>>` - Main CDC stream (lines 19-100)
- **Private Methods**:
  - `EnableTrackChangingAsync(tableName: string, cancellationToken: CancellationToken) -> Task` - Auto-enable CDC on table (lines 133-144)
  - `GetCurrentTrackingVersionAsync() -> Task<long>` - Get current CHANGE_TRACKING_CURRENT_VERSION() (lines 120-126)
  - `GetCurrentTrackingVersionCoreAsync(dbContext: DbContext) -> Task<long>` - Core version query (lines 128-131)
  - `GetChangesAsync(dbContext: DbContext, sqlToGetChanges: string, expectedColumnCount: int, parameters: Dictionary<string, object>) -> Task<List<ChangeItem>?>` - Execute CHANGETABLE query (lines 107-118)
  - `AddChangeTableColumns(changeVersionIdSelect: string?, changeOperationSelect: string?) -> string` - Append SYS_CHANGE_VERSION/SYS_CHANGE_OPERATION columns (lines 102-105)
- **CDC Flow**:
  1. EnableTrackChangingAsync (lines 21) - ALTER TABLE {tableName} ENABLE CHANGE_TRACKING (line 143)
  2. GetCurrentTrackingVersionAsync (line 23) - Get baseline version
  3. Build CHANGETABLE query (lines 25-31):
     ```sql
     SELECT {selectColumns}, [change_table].SYS_CHANGE_VERSION, [change_table].SYS_CHANGE_OPERATION
     FROM CHANGETABLE(CHANGES {tableName}, @version_id) AS [change_table]
     {extraFilter}
     ```
  4. Polling loop (lines 38-99):
     - CreateAsyncScope → GetRequiredService<DbContext> (lines 40-41)
     - Inner loop (lines 45-96):
       - Execute CHANGETABLE query (line 58)
       - If empty: update version to current, apply backoff (lines 66-84)
       - If non-empty: extract max version, yield results (lines 89-95)
     - Recreate DbContext every 1s (line 98)
- **Exponential Backoff**:
  - emptyCounter increments on empty results (line 75)
  - Clamped to 1,000,000 (line 76)
  - waitTimeBetweenQueriesAction(emptyCounter) calculates delay (line 78)
  - Typical pattern: `counter => TimeSpan.FromMilliseconds(Math.Min(counter * 10, 5000))` (10ms → 5s)
- **Version Management**:
  - Baseline: CHANGE_TRACKING_CURRENT_VERSION() (line 130)
  - Per-query: @version_id parameter (line 35)
  - Update: max(SYS_CHANGE_VERSION) from results (line 89)
  - Periodic refresh: Every 10 empty iterations (lines 49-56)
- **Change Detection**:
  - SYS_CHANGE_VERSION: Incremental version per change
  - SYS_CHANGE_OPERATION: 'I' (Insert), 'U' (Update), 'D' (Delete)
- **Notes**:
  - **Performance**: expectedColumnCount hint for DynamicListFromSqlAsync (line 111)
  - **Safety**: Auto-enables CDC if not enabled (lines 138-143)
  - **Error handling**: Break inner loop to recreate DbContext (line 63)
  - **Custom columns**: changeVersionIdSelect/changeOperationSelect for joined queries (lines 102-105)
  - **SQL injection**: sqlBeforeSelect for WITH clauses, extraFilter for WHERE (lines 27-31)
  - **Use case**: Sync SQL Server → Redis/Pulsar for orders, trades, positions

### 9.2 IChangeTracker Interface

- **Purpose**: Public interface for ChangeTracker.
- **Public Methods**:
  - `TrackChangesAsync(tableName: string, selectColumns: string?, extraFilter: string?, cancellationToken: CancellationToken, expectedColumnCount: int, waitTimeBetweenQueriesAction: Func<int, TimeSpan>?, changeVersionIdSelect: string?, changeOperationSelect: string?, sqlBeforeSelect: string?) -> IAsyncEnumerable<List<ChangeItem>>`
- **Notes**: Allows mocking for tests

### 9.3 ChangeItem

- **Purpose**: Single change record with metadata.
- **Public Properties**: (typical)
  - `VersionId: long` - SYS_CHANGE_VERSION
  - `Operation: string` - SYS_CHANGE_OPERATION ('I'/'U'/'D')
  - `Data: IDictionary<string, object?>` - Column values
- **Notes**: Deserialized from dynamic query result via ConvertDynamicDataToChangeItem() (line 114)

### 9.4 TableChangeTrackerBase

- **Purpose**: Abstract base class for table-specific CDC trackers with listener pattern (Channel<List<ChangeItem>>) and background service integration.
- **Patterns**: Template Method pattern, Pub-Sub pattern (Channel-based), Background Service pattern
- **Internal Dependencies**: IChangeTracker (9.2)
- **Threading**: Thread-safe (ConcurrentDictionary for listeners, Channel per listener)
- **Protected Abstract Properties**:
  - `TableName: string` - SQL table name (line 12)
  - `ExtraSqlFilter: string` - WHERE clause (line 13)
  - `SelectColumns: string` - Column list (line 14)
- **Protected Virtual Properties**:
  - `ChangeVersionIdSelect: string?` - Custom version column (default: null, line 16)
  - `ChangeOperationSelect: string?` - Custom operation column (default: null, line 17)
  - `SqlBeforeSelect: string?` - SQL prefix (WITH clauses) (default: null, line 18)
  - `ExpectedColumnCount: int` - Column count hint (default: 8, line 20)
  - `WaitTimeBetweenQueriesAction: Func<int, TimeSpan>?` - Backoff function (default: null, line 21)
- **Protected Fields**:
  - `_changeTracker: IChangeTracker` - Injected tracker (line 10)
  - `listeners: ConcurrentDictionary<string, Channel<List<ChangeItem>>>` - Per-session channels (line 23)
  - `_uniqueTableNames: static ConcurrentDictionary<string, Type>` - Table name uniqueness check (line 25)
- **Constructor**:
  - `TableChangeTrackerBase(serviceProvider: IServiceProvider)` - DI injection + uniqueness check (lines 27-33)
- **Public Virtual Methods**:
  - `InitializeChangeTrackingAsync(cancellationToken: CancellationToken) -> Task` - Background CDC loop (lines 45-75)
- **Public Methods**:
  - `TrackChangesAsync(cancellationToken: CancellationToken) -> IAsyncEnumerable<List<ChangeItem>>` - Subscribe to changes (lines 77-95)
- **Protected Abstract Methods**:
  - `OnChange(changes: List<ChangeItem>) -> Task` - Template method for subclass logic (line 97)
- **Flow**:
  1. InitializeChangeTrackingAsync runs in background (lines 45-75):
     - await foreach _changeTracker.TrackChangesAsync (line 51)
     - Call OnChange(item) (line 55)
     - Broadcast to all listeners (lines 62-63)
     - Retry loop with 1s delay on exception (lines 65-71)
  2. TrackChangesAsync creates listener (lines 77-95):
     - Generate sessionId (line 79)
     - Create unbounded channel (line 81)
     - Register in listeners dictionary (line 82)
     - await foreach channel.Reader.ReadAllAsync (line 86)
     - Unregister in finally (line 93)
- **Notes**:
  - **Uniqueness check**: Ensures one tracker per table name (lines 35-43)
  - **Listener pattern**: Multiple consumers can subscribe via TrackChangesAsync (lines 77-95)
  - **OnChange hook**: Subclasses override to apply changes (e.g. update DistributedDataManager) (line 97)
  - **Error isolation**: OnChange exceptions logged, do not stop CDC loop (lines 57-60)
  - **Use case**: Base class for OrderChangeTracker, TradeChangeTracker, etc.

### 9.5 TableChangeTrackerWithMemoryCacheBase

- **Purpose**: Extension of TableChangeTrackerBase with MemoryCacheHelper integration for caching unchanged data.
- **Patterns**: Template Method pattern, Cache-Aside pattern
- **Internal Dependencies**: MemoryCacheHelper (2.1)
- **Notes**: (not shown in snippet, likely adds GetOrCache methods)

### 9.6 ObjectExtensions

- **Purpose**: Extension methods for ChangeItem deserialization.
- **Public Methods**: (likely)
  - `ConvertDynamicDataToChangeItem(this IDictionary<string, object?> dynamicData) -> ChangeItem` - Convert dynamic result to ChangeItem (line 114 reference)
- **Constants**:
  - `COLUMN_NAME_SYS_CHANGE_VERSION: string` - "SYS_CHANGE_VERSION" (line 103)
  - `COLUMN_NAME_SYS_CHANGE_OPERATION: string` - "SYS_CHANGE_OPERATION" (line 103)

### 9.7 IServiceCollectionExtension

- **Purpose**: DI registration for IChangeTracker as Singleton.
- **Public Methods**: (likely)
  - `AddChangeTracker(this IServiceCollection services) -> IServiceCollection` - Register ChangeTracker
- **Notes**: Singleton lifetime (one tracker per application)

---

## 10. PIPELINE

### Purpose
Wrapper over TPL Dataflow (System.Threading.Tasks.Dataflow) for building modular, composable, backpressure-aware data processing pipelines with broadcast, batch, buffer, transform, and action blocks.

### 10.1 TplPipeline<T>

- **Purpose**: Fluent builder for TPL Dataflow pipelines with block chaining, branching (LinkTo), and completion propagation.
- **Patterns**: Fluent Interface pattern, Builder pattern, Pipeline pattern, Dataflow pattern
- **Internal Dependencies**: IPipelineStep interfaces (10.2-10.5), TplPipelineExtensions (10.6)
- **External Dependencies**: System.Threading.Tasks.Dataflow (BroadcastBlock, BatchBlock, BufferBlock, ActionBlock, TransformBlock)
- **Threading**: Thread-safe (underlying Dataflow blocks are thread-safe)
- **Private Fields**:
  - `firstStep: ITargetBlock<T>?` - Pipeline entry point (line 13)
  - `_steps: List<IDataflowBlock>` - All blocks in pipeline (line 14)
- **Public Static Methods**:
  - `CreatePipeline() -> TplPipeline<T>` - Factory method (lines 19-22)
- **Public Methods (Builder)**:
  - `AddBroadcastBlock(stepImplementation: IBroadcastStep<T>? = default, linkToStepIndex: int? = null) -> TplPipeline<T>` - Add broadcast block (fan-out) (lines 24-38)
  - `AddBatchActionBlock(stepImplementation: IBatchActionPipelineStep<T>, linkToStepIndex: int? = null) -> TplPipeline<T>` - Add batch + action block (lines 40-70)
  - `AddBufferActionBlock(stepImplementation: IBufferActionPipelineStep<T>, linkToStepIndex: int? = null) -> TplPipeline<T>` - Add buffer + action block (lines 72-102)
  - `AddTransformBlock<TOut>(stepImplementation: ITransformStep<T, TOut>?, linkToStepIndex: int? = null) -> TplPipeline<T>` - Add transform block (lines 104-125)
  - `AddStep<TLocalIn>(newStep: IDataflowBlock, linkOptions: DataflowLinkOptions? = default, predicate: Predicate<TLocalIn>? = default, linkToStepIndex: int? = null) -> TplPipeline<T>` - Add custom block (lines 127-132)
- **Public Methods (Execution)**:
  - `SendAsync(input: T) -> Task<bool>` - Async post with backpressure (lines 204-214)
  - `Post(input: T) -> bool` - Sync post (lines 232-235)
  - `Complete() -> Task` - Signal completion + await all blocks (lines 241-256)
- **Private Methods**:
  - `AddOrLinkToStep<TLocalIn>(newStep: IDataflowBlock, linkOptions: DataflowLinkOptions?, predicate: Predicate<TLocalIn>?, linkToStepIndex: int?) -> void` - Router for add vs link (lines 137-143)
  - `AddStepCore<TLocalIn>(newStep: IDataflowBlock, linkOptions: DataflowLinkOptions?, predicate: Predicate<TLocalIn>?) -> void` - Append to pipeline (lines 145-159)
  - `LinkToGivenStepCore<TLocalIn>(newStep: IDataflowBlock, linkToStepIndex: int, linkOptions: DataflowLinkOptions?, predicate: Predicate<TLocalIn>?) -> void` - Branch to existing step (lines 161-167)
  - `LinkToCore<TLocalIn>(newStep: IDataflowBlock, linkToStepIndex: int, linkOptions: DataflowLinkOptions?, predicate: Predicate<TLocalIn>?) -> void` - Core linking logic (lines 169-183)
- **Block Configurations**:
  - **BroadcastBlock**: Unbounded, not ordered, unbounded messages per task (lines 27-32)
  - **BatchBlock**: Unbounded, not ordered, unbounded messages/groups (lines 45-51)
  - **BufferBlock**: Bounded by stepImplementation.BufferCapacity, unbounded messages per task (lines 77-81)
  - **ActionBlock (Batch)**: SingleProducer, unbounded parallelism, unbounded capacity/messages (lines 55-62)
  - **ActionBlock (Buffer)**: SingleProducer, parallelism=1, bounded capacity=1 (lines 85-92)
  - **TransformBlock**: Default options (line 108-120)
- **Pipeline Flow**:
  1. CreatePipeline() → empty pipeline (line 21)
  2. AddXxxBlock(...) → append/branch blocks (lines 24-132)
  3. Post/SendAsync(input) → firstStep.Post/SendAsync (lines 232-235, 204-214)
  4. Data flows through linked blocks (automatic by TPL Dataflow)
  5. Complete() → signal all blocks, await completion (lines 241-256)
- **Linking**:
  - **Sequential**: linkToStepIndex = null → append to last block (line 139)
  - **Branch**: linkToStepIndex = N → link to step[N] (line 141-142)
  - **Predicate**: Optional filter per link (line 179-182)
  - **PropagateCompletion**: Enabled for all links (lines 65-67, 95)
- **Notes**:
  - **Backpressure**: SendAsync respects BoundedCapacity (returns false if full) (line 204)
  - **Completion**: Complete() signals all blocks and awaits (lines 241-256)
  - **Error handling**: try-catch in Complete() (lines 247-253)
  - **Branching**: Multiple blocks can link to same source (fan-out) (line 161-167)
  - **Use case**: Data ingestion pipelines (Pulsar → Transform → Batch → Redis), ETL workflows

### 10.2 IBroadcastStep<T>

- **Purpose**: Interface for broadcast block (fan-out with cloning).
- **Public Properties**:
  - `CloningFunction: Func<T, T>?` - Clone function for broadcast (line 26)
- **Notes**: null = no cloning (reference shared)

### 10.3 IBatchActionPipelineStep<T>

- **Purpose**: Interface for batch action step.
- **Public Properties**:
  - `BatchSize: int` - Batch size (line 44)
  - `DoAction: Action<T[]>` - Batch action (line 54)
- **Notes**: Batch accumulates BatchSize items before calling DoAction

### 10.4 IBufferActionPipelineStep<T>

- **Purpose**: Interface for buffer action step (read all buffered items on trigger).
- **Public Properties**:
  - `BufferCapacity: int` - Max buffer size (line 79)
- **Public Methods**:
  - `CreateFilterPredicate() -> Predicate<T>?` - Optional filter (line 97)
  - `SetBufferBlock(bufferBlock: BufferBlock<T>) -> void` - Inject buffer for manual inspection (line 99)
- **Notes**: ActionBlock reads ALL buffered items via ReadAllMessagesAndDoAction extension (line 84)

### 10.5 ITransformStep<TIn, TOut>

- **Purpose**: Interface for transform block.
- **Public Methods**:
  - `DoAction(item: TIn) -> Task<TOut?>` - Async transform (line 112)
- **Notes**: null output items are propagated (line 119)

### 10.6 TplPipelineExtensions

- **Purpose**: Extension methods for BufferBlock (ReadAllMessagesAndDoAction).
- **Public Methods**: (likely)
  - `ReadAllMessagesAndDoAction<T>(this BufferBlock<T> bufferBlock, item: T, stepImplementation: IBufferActionPipelineStep<T>) -> Task` - Read all + execute (line 84 reference)
- **Notes**: Drains buffer and calls stepImplementation.DoAction with all items

### 10.7 IPipelineStep

- **Purpose**: Base marker interface for all pipeline steps.
- **Notes**: (not shown in snippet)

---

## 11. EXTENSIONS

### Purpose
Utility extensions for DI registration, SQL dynamic queries, date/time conversions, service naming, and custom exceptions.

### 11.1 IServiceCollectionExtensions

- **Purpose**: DI registration helpers for hosted services and graceful shutdown.
- **Public Methods**:
  - `AddSmartpulseEndPointFinalizer(this IServiceCollection services, endServicesFunc: Func<IServiceProvider, IEnumerable<IEndService>>?) -> IServiceCollection` - Register shutdown finalizer (lines 9-17)
- **Flow**:
  1. Register endServicesFunc as Transient (lines 11-12)
  2. Register SmartpulseStopHostedService as HostedService (line 14)
  3. SmartpulseStopHostedService calls IEndService.Stop() on all registered services during shutdown
- **Notes**:
  - **IEndService**: Interface with Stop() method for graceful shutdown (line 9)
  - **SmartpulseStopHostedService**: BackgroundService that invokes IEndService.Stop() on app shutdown (line 14)
  - **Use case**: Close Apache Pulsar producers, flush Redis connections, etc.

### 11.2 DynamicListSqlExtensions

- **Purpose**: Extension methods for dynamic SQL queries with IAsyncEnumerable results.
- **Public Methods**: (likely)
  - `DynamicListFromSqlAsync(this DbContext dbContext, sql: string, expectedColumnCount: int, parameters: Dictionary<string, object>) -> IAsyncEnumerable<IDictionary<string, object?>>` - Execute SQL and stream results (line 111 reference)
- **Notes**: Used by ChangeTracker for CHANGETABLE queries (line 111)

### 11.3 NodaTimeExtensions

- **Purpose**: Extension methods for NodaTime (Instant, LocalDateTime, ZonedDateTime) ↔ DateTimeOffset conversions.
- **Public Methods**: (likely)
  - `ToInstant(this DateTimeOffset dateTimeOffset) -> Instant`
  - `ToLocalDateTime(this DateTimeOffset dateTimeOffset, DateTimeZone zone) -> LocalDateTime`
  - `ToZonedDateTime(this DateTimeOffset dateTimeOffset, DateTimeZone zone) -> ZonedDateTime`
  - `ToDateTimeOffset(this Instant instant) -> DateTimeOffset`
- **Notes**: Used for time zone-aware calculations (market hours, delivery periods)

### 11.4 ServiceNameConvertExtensions

- **Purpose**: Extension methods for service name conversions (PascalCase ↔ kebab-case, etc.)
- **Public Methods**: (likely)
  - `ToPascalCase(this string serviceName) -> string`
  - `ToKebabCase(this string serviceName) -> string`
- **Notes**: Used for service discovery, logging, etc.

### 11.5 TicketException

- **Purpose**: Custom exception with ticket ID for tracking/logging.
- **Public Properties**: (likely)
  - `TicketId: string` - Unique ticket ID (Guid or timestamp-based)
  - `Message: string` - Error message
- **Notes**: Used for user-facing errors with support ticket reference

---

## NOTES & CROSS-REFERENCES

### Internal Relations

1. **DistributedData → Collections.Concurrent**:
   - DistributedDataManager<T> (7.1) uses ConcurrentObservableDictionary<TKey, TValue> (4.3) for data cache (line 33)
   - DataWithLock<T> (7.2) uses ConcurrentObservableDictionary for change buffer (line 767)
   - DistributedDataSyncService (7.6) uses AutoConcurrentPartitionedQueue<T> (4.1) for per-dataKey ordering (line 32)

2. **DistributedData → Apache Pulsar**:
   - DistributedDataManager<T> can publish changes to Apache Pulsar for cross-region sync
   - Change stream (GetDistributedDataChangeEnumerationAsync) can feed SmartpulsePulsarClient (1.1)

3. **TrackChanges → DistributedData**:
   - TableChangeTrackerBase (9.4) → OnChange → DistributedDataManager<T>.SetAsync (7.1)
   - Flow: SQL Server CDC → ChangeItem → JSON Patch → Redis HSET → Pub/Sub → Remote services

4. **Electricity → DistributedData**:
   - All Electricity managers (8.12.x) inherit DistributedDataManager<T> (7.1)
   - Intraday models (8.11.x) implement IDistributedData (7.3)

5. **Pipeline → Apache Pulsar**:
   - TplPipeline<T> (10.1) consumes SmartpulsePulsarClient.CreateTopicConsumerAsync (1.1)
   - Flow: Pulsar → TplPipeline → Transform → Batch → DistributedDataManager

6. **Pipeline → DistributedData**:
   - Pipeline can batch updates to DistributedDataManager<T>.SetAsync (7.1)
   - Flow: CDC stream → TplPipeline → Batch → Redis bulk write

### Design Patterns Used Together

1. **Cache-Aside + Optimistic Locking**:
   - DistributedDataManager<T> (7.1): GetAsync (cache-aside) + SetAsync (optimistic locking via VersionId)
   - Conflict resolution: Version gap > 1 → cache eviction

2. **Event Sourcing + JSON Patch**:
   - DistributedDataChangedInfo (7.8): PatchItems (add/remove/replace operations)
   - Change stream as event log for replay/debugging

3. **CDC + Distributed Cache**:
   - ChangeTracker (9.1) → TableChangeTrackerBase (9.4) → DistributedDataManager (7.1)
   - Real-time sync: SQL Server → Redis → Remote services

4. **Pipeline + Batch Processing**:
   - TplPipeline<T> (10.1) + IBatchActionPipelineStep<T> (10.3)
   - Bulk operations: Pulsar stream → Batch → Redis HMSET (multiple fields)

5. **Pub-Sub + Partitioned Queue**:
   - DistributedDataSyncService (7.6) + AutoConcurrentPartitionedQueue<T> (4.1)
   - Per-dataKey ordering: Redis pub/sub → partitioned queue → sequential apply

### Threading Considerations

1. **Per-key locking**:
   - DistributedDataManager<T> (7.1): SemaphoreSlim per DataKey (line 765)
   - Prevents concurrent updates to same key, allows parallel updates to different keys

2. **Double-checked locking**:
   - DistributedDataManager<T>.GetAsync: Check cache → lock → check again → load from Redis (lines 82-96)
   - Prevents cache stampede

3. **Retry with exponential backoff**:
   - ChangeTracker (9.1): waitTimeBetweenQueriesAction (emptyCounter) (line 78)
   - Typical: 10ms → 100ms → 1s → 5s (capped)

4. **Partitioned concurrency**:
   - DistributedDataSyncService (7.6): AutoConcurrentPartitionedQueue per manager (line 32)
   - Parallelism: Multiple dataKeys processed concurrently, same dataKey sequentially

5. **Pipeline parallelism**:
   - TplPipeline<T> (10.1): MaxDegreeOfParallelism = Unbounded for BatchActionBlock (line 58)
   - Batch processing: Multiple batches processed in parallel

### Performance Notes

1. **Change buffer optimization**:
   - DistributedDataManager<T> (7.1): Change buffer (line 767) prevents full reload for late subscribers
   - Trade-off: Memory vs. network (buffer size controlled by maxChangeBufferCount)

2. **JSON Patch efficiency**:
   - Only changed fields transmitted (add/remove/replace operations)
   - Large objects: 1% change → 1% network overhead (vs. 100% for full object)

3. **Bulk operations**:
   - TplPipeline<T> + IBatchActionPipelineStep<T>: Batch Redis HMSET (multiple fields in one call)
   - Latency: N individual HSET = N × RTT; 1 HMSET = 1 × RTT

4. **Cache eviction strategies**:
   - FlushDataAsync (7.1): Evict by ChangeTime > maxAge (LRU-like)
   - FlushChangeBufferAsync (7.1): Evict buffered changes > fieldMaxAge

5. **Redis connection pooling**:
   - StackExchange.Redis connection multiplexing (single connection per server)
   - Retry logic: 100 iterations × 50ms = 5s max (lines 259-272)

### Recommended Use Cases

1. **Distributed Cache with CDC**:
   - SQL Server (source of truth) → ChangeTracker (9.1) → DistributedDataManager (7.1) → Redis (cache)
   - Remote services subscribe via GetDistributedDataChangeEnumerationAsync (7.1)

2. **Cross-Region Sync**:
   - Region A: DistributedDataManager → PublishAsync → Apache Pulsar (1.1)
   - Region B: Pulsar consumer → TplPipeline (10.1) → DistributedDataManager.SetAsync

3. **Real-Time Market Data**:
   - Exchange API → IntradayOrder (8.11.3) → OrderDataManager (8.12.2) → Redis
   - Trading services subscribe: GetDistributedDataChangeEnumerationAsync → Order book updates

4. **ETL Pipelines**:
   - CDC stream → TplPipeline (10.1) → Transform → Batch → DistributedDataManager → Redis
   - Flow: SQL Server → ChangeTracker → TransformBlock → BatchActionBlock → SetAsync

5. **Graceful Shutdown**:
   - IServiceCollectionExtensions (11.1) → SmartpulseEndPointFinalizer
   - Ensures: Flush Redis, close Pulsar producers, complete pipelines

### Scalability Patterns

1. **Horizontal Scaling**:
   - Multiple DistributedDataManager instances (different partitions/sections)
   - Redis cluster: Partition by {PartitionKey}:{Section} (hash slot distribution)

2. **Vertical Scaling**:
   - TplPipeline MaxDegreeOfParallelism controls CPU utilization
   - AutoConcurrentPartitionedQueue auto-scales partitions

3. **Backpressure**:
   - TplPipeline SendAsync: Respects BoundedCapacity (returns false if full)
   - BufferBlock: Bounded capacity prevents memory overflow (line 79)

4. **Lazy Cleanup**:
   - AutoConcurrentPartitionedQueue (4.1): TryClearCompletedPartitionKeys
   - DistributedDataManager (7.1): FlushDataAsync, FlushChangeBufferAsync

### Monitoring & Observability

1. **Version Tracking**:
   - DistributedDataChangedInfo.VersionId: Monotonic counter for ordering/debugging
   - CheckSyncVersionErrorsAsync (7.1): Detect version mismatches (lines 710-734)

2. **Change Stream Inspection**:
   - GetChangesBuffer (7.1): Inspect buffered changes for debugging (lines 653-659)
   - PatchItems: Full audit trail (add/remove/replace operations)

3. **Pipeline Completion**:
   - TplPipeline.Complete (10.1): Await all blocks → completion status (lines 241-256)

4. **CDC Metrics**:
   - ChangeTracker emptyCounter: Tracks polling frequency (line 75)
   - Console.WriteLine: Version updates, data counts (lines 33, 93)

### Error Handling Strategies

1. **Retry with Backoff**:
   - Redis operations: 100 iterations × 50ms (lines 259-272, 300-318, 482-494)
   - CDC polling: Exponential backoff (line 78)

2. **Graceful Degradation**:
   - DistributedDataManager.GetAsync: Returns null if Redis unavailable (line 77)
   - DistributedDataManager.SetAsync: Returns false if no changes (line 338)

3. **Cache Invalidation**:
   - OnConnectionRestored: Clear all caches on Redis reconnect (lines 53-59)
   - Version mismatch: Evict cache, fire SyncErrorTaken (lines 560-566)

4. **Pipeline Error Isolation**:
   - TransformBlock: try-catch, return default on exception (lines 114-117)
   - TableChangeTrackerBase.OnChange: Exception logged, CDC continues (lines 57-60)

5. **Stability Retry**:
   - GetInitialFieldValuesFromDB: Retry up to 10 times if HLEN unstable (lines 163-180)