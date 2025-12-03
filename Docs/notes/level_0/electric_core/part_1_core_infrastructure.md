# Electric.Core - Level_0 Notes - PART 1

## 1. APACHE_PULSAR
### Purpose
Wrapper over DotPulsar library providing simplified access to Apache Pulsar with producer/consumer management, compression, and JSON/MongoDB serialization.

### 1.1 SmartpulsePulsarClient

- **Purpose**: Main client for Apache Pulsar communication, managing producers (cached) and consumers (IAsyncEnumerable).
- **Patterns**: Repository pattern (cached producers), Factory pattern, Dispose pattern (IAsyncDisposable)
- **Internal Dependencies**: None
- **Threading**: Thread-safe via ConcurrentDictionary for producers, async/await for consumers
- **Public Properties**:
  - `Client: IPulsarClient` - Access to underlying DotPulsar client (read-only)
- **Public Methods**:
  - `SmartpulsePulsarClient(configuration: IConfiguration)` - Constructor with IConfiguration (reads "Pulsar:PULSAR_CONNSTR")
  - `SmartpulsePulsarClient(pulsarConnStr: string?)` - Constructor with connection string
  - `CreateTopicConsumerAsync<T>(topic: string, subscriptionName: string, subscriptionType: SubscriptionType = Exclusive, subscriptionInitialPosition: SubscriptionInitialPosition = Latest, messagePrefetchCount: uint = 1000, stateChangeHandler: Func<ConsumerStateChanged, CancellationToken, ValueTask>? = null, cancellationToken: CancellationToken = default) -> IAsyncEnumerable<(T, IMessage<ReadOnlySequence<byte>>)>` - Consumer with JSON deserialization to type T
  - `CreateTopicConsumerAsync(topic: string, subscriptionName: string, ...) -> IAsyncEnumerable<(string data, IMessage<ReadOnlySequence<byte>> rawData)>` - Consumer returning UTF-8 string
  - `CreateTopicConsumerRawAsync(topic: string, subscriptionName: string, ...) -> IAsyncEnumerable<IMessage<ReadOnlySequence<byte>>>` - Consumer for raw binary data
  - `CreateTopicToProduce(topic: string, attachTraceInfoMessages: bool = false, maxPendingMessages: uint = 500, compressionType: CompressionType = None, stateChangeHandler: Func<ProducerStateChanged, CancellationToken, ValueTask>? = null) -> void` - Producer registration for topic (cached in ConcurrentDictionary)
  - `WriteObj(topic: string, obj: object) -> ValueTask<MessageId?>` - Send object as JSON
  - `WriteObj<T>(topic: string, obj: T) -> ValueTask<MessageId?>` - Send generic object as JSON
  - `WriteText(topic: string, text: string) -> ValueTask<MessageId?>` - Send UTF-8 text
  - `WriteBytes(topic: string, data: byte[]) -> ValueTask<MessageId?>` - Send raw bytes (requires prior producer registration)
  - `DisposeTopic(topic: string) -> ValueTask` - Remove and dispose producer for specific topic
  - `DisposeAsync() -> ValueTask` - Dispose all producers and client
- **Notes**:
  - **Performance**: Producers are cached (ConcurrentDictionary), eliminating creation overhead
  - **Compression**: Supported via CompressionType (None/LZ4/Zlib/Zstd/Snappy)
  - **Safety**: Thread-safe for multiple producers/consumers; write errors logged to Console, returned as null
  - **Consumer pattern**: IAsyncEnumerable with [EnumeratorCancellation] for efficient cancellation handling
  - **Prefetch**: Default prefetch 1000 messages for throughput optimization

### 1.2 IServiceCollectionExtension

- **Purpose**: Extension method for registering SmartpulsePulsarClient in DI container as Singleton.
- **Patterns**: Extension Method pattern, Dependency Injection
- **Internal Dependencies**: SmartpulsePulsarClient
- **Threading**: N/A (DI registration only)
- **Public Methods**:
  - `AddApachePulsarClient(this IServiceCollection services, pulsarConnectionStringFactory: Func<string>? = null) -> IServiceCollection` - Registration as Singleton; if factory = null, uses IConfiguration, otherwise factory
- **Notes**: Singleton lifetime - one client per application (preferred pattern for Pulsar client pooling)

### 1.3 DateTimeToObjectConverter

- **Purpose**: JsonConverter for DateTime to MongoDB BSON format ($data with $numberLong).
- **Patterns**: Adapter pattern (System.Text.Json.Serialization.JsonConverter)
- **Internal Dependencies**: None
- **External Dependencies**: MongoDB.Bson (BsonUtils)
- **Threading**: Stateless, thread-safe
- **Public Methods**:
  - `Read(ref Utf8JsonReader reader, Type typeToConvert, JsonSerializerOptions options) -> DateTime` - Returns DateTime.MinValue (not implemented)
  - `Write(Utf8JsonWriter writer, DateTime value, JsonSerializerOptions options) -> void` - Writes DateTime as {"$data":{"$numberLong":"ticks"}}
- **Notes**:
  - **Read not implemented** - returns DateTime.MinValue (possible bug or intentional for write-only use case)
  - **MongoDB-compatible serialization**: ticks = milliseconds since Unix epoch (BsonUtils.ToMillisecondsSinceEpoch)
  - **WriteRawValue**: Uses raw JSON for performance (skipInputValidation: true)

### 1.4 IdToObjectConverter

- **Purpose**: JsonConverter for MongoDB ObjectId to format MongoDB JSON ({$oid:"..."}).
- **Patterns**: Adapter pattern (System.Text.Json.Serialization.JsonConverter)
- **Internal Dependencies**: None
- **External Dependencies**: MongoDB.Bson (ObjectId)
- **Threading**: Stateless, thread-safe
- **Public Methods**:
  - `Read(ref Utf8JsonReader reader, Type typeToConvert, JsonSerializerOptions options) -> ObjectId` - Returns new ObjectId() (not implemented)
  - `Write(Utf8JsonWriter writer, ObjectId value, JsonSerializerOptions options) -> void` - Writes ObjectId as {$oid:"..."}
- **Notes**:
  - **Read not implemented** - returns empty ObjectId (possible bug or intentional for write-only use case)
  - **Serialization MongoDB-compatible**: Format Extended JSON v2 for ObjectId
  - **WriteRawValue**: Raw JSON for performance

---

## 2. CACHING

### Purpose
Caching strategies in-memory (System.Runtime.Caching.MemoryCache) with thread-safety, SemaphoreSlim locking i object pooling.

### 2.1 MemoryCacheHelper

- **Purpose**: Wrapper over MemoryCache with automatic cache-aside pattern, locking for cache stampede, i versioned cache clearing.
- **Patterns**: Cache-Aside pattern, Double-checked locking, Versioned cache swap (zero-downtime clear)
- **Internal Dependencies**: None
- **Threading**: Thread-safe; SemaphoreSlim per cache key (cached as "_semaphore" suffix) for cache stampede prevention
- **Public Properties**:
  - `Cache: MemoryCache` - Access to base cache (read-only)
- **Public Methods**:
  - `MemoryCacheHelper(configuration: MemoryCacheHelperConfiguration? = null)` - Constructor with optional configuration (default: 1h TTL, "TheCache" name)
  - `Get<T>(key: string, whenNotHitAction: Func<string, Task<T>>, cacheTime: DateTimeOffset? = default, withLock: bool = false) -> ValueTask<T>` - Cache-aside: returns from cache or calls factory and caches; withLock = double-checked locking
  - `Add(key: string, cacheObj: object?, cacheTime: DateTimeOffset? = null) -> void` - Direct addition to cache
  - `Remove(key: string) -> object` - Removal with cache
  - `ClearCache() -> Task` - Zero-downtime cache clear via versioned swap (_v1 toggle) and dispose old cache after 1s delay
- **Patterns locking**:
  - **GetAddSemaphore**: Creates SemaphoreSlim per key, caches it with the same TTL as data
  - **Double-checked locking**: Po WaitAsync checks cache again (lines 43-48)
- **Notes**:
  - **Thread-safety**: SemaphoreSlim(1,1) per key prevents cache stampede (N threads calling simultaneously whenNotHitAction)
  - **Versioned swap**: ClearCache() does not block; creates new cache, exchanges atomically (Interlocked.Exchange), removes old after 1s
  - **Performance**: withLock = false for hot paths (no locking overhead); true for expensive operations (DB queries)
  - **Configuration**: MemoryCacheHelperConfiguration allows customization (SemaphoreSuffix, DefaultCacheHours, CacheName)

### 2.2 MemoryCacheHelperConfiguration

- **Purpose**: POCO configuration for MemoryCacheHelper.
- **Public Properties**:
  - `SemaphoreSuffix: string` - Suffix for semaphore keys (default: "_semaphore")
  - `DefaultCacheHours: int` - Default TTL in hours (default: 1)
  - `CacheName: string` - Name cache (default: "TheCache")

### 2.3 ObjectPool<T> where T : new()

- **Purpose**: Generic object pool with auto-creation (new T()) for reuse heavy objects (e.g. List<T>).
- **Patterns**: Object Pool pattern
- **Internal Dependencies**: None
- **Threading**: Thread-safe (ConcurrentQueue)
- **Public Methods**:
  - `ObjectPool(maxPoolCount: short)` - Constructor with max pool size
  - `GetOne() -> T` - Retrieves from pool or creates new (new T())
  - `GiveBackOne(item: T) -> void` - Returns to pool (if max not exceeded)
- **Notes**:
  - **Auto-creation**: Always returns object (new T() fallback)
  - **Bounded pool**: maxPoolCount prevents unbounded growth
  - **Use case**: AutoConcurrentPartitionedQueue uses ObjectPool<List<T>> (line 193)

### 2.4 ObjectPoolNoCreate<T>

- **Purpose**: Object pool WITHOUT auto-creation (returns null if empty).
- **Patterns**: Object Pool pattern
- **Internal Dependencies**: None
- **Threading**: Thread-safe (ConcurrentQueue)
- **Public Methods**:
  - `ObjectPoolNoCreate(maxPoolCount: short)` - Constructor with max pool size
  - `GetOne() -> T?` - Retrieves from pool or returns default (null)
  - `GiveBackOne(item: T) -> void` - Returns to pool (if max not exceeded)
- **Notes**:
  - **No auto-creation**: Caller must handle null
  - **Use case**: PartitionedConcurrentConsumer uses ObjectPoolNoCreate<Channel<TPayload>> (lines 293-294)

### 2.5 MemoryCacheExtension

- **Purpose**: Extension method for registering MemoryCacheHelper in DI container as Singleton.
- **Patterns**: Extension Method pattern, Dependency Injection
- **Internal Dependencies**: MemoryCacheHelper
- **Threading**: N/A (DI registration only)
- **Public Methods**:
  - `AddSmartpulseMemoryCache(this IServiceCollection services, memoryCacheConfiguration: MemoryCacheHelperConfiguration? = null) -> IServiceCollection` - Registration as Singleton
- **Notes**: Singleton lifetime - one cache helper per application

---

## 3. COLLECTIONS

### Purpose
Non-concurrent observable collections (Dictionary) with INotifyCollectionChanged for UI binding i reactive patterns.

### 3.1 ObservableDictionary<TKey, TValue>

- **Purpose**: Dictionary with INotifyCollectionChanged events for Add/Remove/Replace.
- **Patterns**: Observer pattern (INotifyCollectionChanged), Decorator pattern (wraps Dictionary)
- **Internal Dependencies**: None
- **Threading**: **NOT thread-safe** (based on Dictionary<TKey, TValue>)
- **Public properties**:
  - `this[TKey key]: TValue` - Indexer with event notification (Add or Replace)
- **Public Methods**:
  - `Add(key: TKey, value: TValue) -> void` - Adds i emits CollectionChanged (Add action)
  - `Remove(key: TKey) -> bool` - Removes i emits CollectionChanged (Remove action)
  - `Remove(key: TKey, out val: TValue) -> bool` - Removes with output value i emits event
- **Events**:
  - `CollectionChanged: NotifyCollectionChangedEventHandler?` - Emitted on Add/Replace/Remove
- **Notes**:
  - **NOT thread-safe**: Usage in multi-threaded environment inymaga external locking
  - **Use case**: WPF/MAUI UI binding, reactive streams (Rx)
  - **Replace detection**: Indexer checks existsBefore and emits Replace event if key existed

---

## 4. COLLECTIONS.CONCURRENT

### Purpose
Thread-safe observable collections with advanced patterns: partitioned queues (auto-scaling, backpressure), priority queues, rate limiting i IAsyncEnumerable reactive streams.

### 4.1 AutoConcurrentPartitionedQueue<T>

- **Purpose**: Auto-scaling partitioned queue with bulk processing, dynamic throttling i background cleanup for completed partitions. Main pattern for high-throughput partitioned workloads.
- **Patterns**: Partitioned queue pattern, Producer-Consumer pattern, Bulk processing, Lazy cleanup, Object pooling
- **Internal Dependencies**: ObjectPool<List<T>> (Caching/ObjectPool.cs)
- **Threading**:
  - **Thread-safe**: ConcurrentDictionary for partitions, Channel<T> per partition (SingleReader/SingleWriter)
  - **Long-running task**: mainQueueProcessTask (TaskCreationOptions.LongRunning)
  - **Consumer throttling**: ConsumerThrottleCount prevents multiple consumer tasks per partition (max 2 concurrent)
- **Public Properties**:
  - `PartitionKeyCount: int` - Number of active partitions (read-only)
  - `PartitionItemCount: int` - Sum items in all partitions (read-only)
  - `MainQueueCount: int` - Number of items in main queue (read-only)
  - `ClearMaxCounter: short` - Frequency cleanup (default: 1000 iterations) (settable)
  - `LogClearJob: bool` - Enables logging for cleanup jobs (settable)
- **Public Methods**:
  - `AutoConcurrentPartitionedQueue(onProcessCallback: Func<string, List<T>, Task>)` - Constructor; callback called per partition with batched items
  - `Enqueue(key: string, item: T, maxCount: int? = null, mode: BoundedChannelFullMode = Wait) -> void` - Adds item to main queue; auto-starts processing; maxCount per partition (bounded channel)
  - `WaitForAllFinish() -> Task` - Async await aż all partitions are empty i completed
- **Architektura**:
  - **Main queue**: Channel<ChannelItem<T>> (unbounded) - central queue for all partitions
  - **Partition queues**: ConcurrentDictionary<string, PartitionQueueItem<T>> - per-partition Channel<T>
  - **Processing flow**: mainQueue → batch grouping → SendToTaskChainForProducer → partition queues → CreateConsumerTask → onProcessCallback
- **Bulk processing**:
  - **Dynamic bulk size**: DecideMaxBulkSize(queueCount) = queueCount < 1000 ? 800 : queueCount/0.8 (80% batch)
  - **ProcessMainQueue**: Reads bulk from mainQueue, groups by key, distributes to partition queues
- **Throttling**:
  - **Consumer throttle**: ConsumerThrottleCount > 2 prevents spawning new consumer tasks (lines 169-173)
  - **Consumer chaining**: CreateConsumerTask awaits previous task (line 182)
- **Cleanup**:
  - **Lazy cleanup**: TryClearCompletedPartitionKeys() runs every ClearMaxCounter iterations OR when PartitionKeyCount > allBulk.Count * 20
  - **Zero overhead**: Cleanup runs asynchronously (Task.Run) without blocking
- **Notes**:
  - **Performance**: Bulk processing (List<T>) + object pooling (allItemListPool) for zero-allocation
  - **Backpressure**: Bounded channels per partition (BoundedChannelFullMode: Wait/DropWrite/DropOldest)
  - **Scalability**: Auto-scaling partitions (lazy creation) + auto-cleanup (memory efficiency)
  - **Use case**: Kafka-style partitioned message processing, batch database writes per entity

### 4.2 AutoConcurrentPartitionedQueueV2<T>

- **Purpose**: Simplified version V1 - eliminates main queue, direct enqueue to partition channels with auto-timeout cleanup.
- **Patterns**: Partitioned queue pattern, Producer-Consumer pattern, Bulk processing, Object pooling, Timeout-based cleanup
- **Internal Dependencies**: ObjectPool<List<T>> (Caching/ObjectPool.cs)
- **Threading**:
  - **Thread-safe**: ConcurrentDictionary for partitions, Channel<T> per partition (SingleReader)
  - **Per-partition task**: Task.Run per partition (lines 46)
  - **No throttling**: Each partition has own consumer task (simpler than V1)
- **Public Properties**:
  - `PartitionKeyCount: int` - Number of active partitions (read-only)
  - `MaxWaitTimeWhenNoItemInQueue: int` - Max ms waiting na now items before partition completion (default: 5000ms) (settable)
- **Public Methods**:
  - `AutoConcurrentPartitionedQueueV2(onProcessCallback: Func<string, List<T>, Task>)` - Constructor
  - `Enqueue(key: string, item: T, maxCount: int? = null, mode: BoundedChannelFullMode = Wait) -> void` - Direct enqueue to partition channel; spin-loop if channel complete (lines 26-51)
  - `WaitForAllFinish() -> Task` - Async await aż all partitions are removed
- **Differences vs V1**:
  - **No main queue**: Direct enqueue to partition channels (simpler, fewer allocations)
  - **Timeout-based cleanup**: WaitToReadAsync with CancellationTokenSource(MaxWaitTimeWhenNoItemInQueue) (lines 89-97)
  - **Auto-completion**: TryComplete() channel writer after timeout (line 101)
  - **Simpler**: None consumer throttling, brak dynamic bulk sizing, brak ClearMaxCounter
- **Notes**:
  - **Performance**: Better for low-latency (no main queue overhead)
  - **Tradeoff**: Timeout-based cleanup can prematurely close partition (MaxWaitTimeWhenNoItemInQueue)
  - **Use case**: High-throughput with short bursts per partition

### 4.3 ConcurrentObservableDictionary<TKey, TValue>

- **Purpose**: Thread-safe observable dictionary with INotifyCollectionChanged events i LastModifiedDateTime tracking.
- **Patterns**: Observer pattern, Decorator pattern (wraps ConcurrentDictionary), GetKeysEnumeratorAsync (IAsyncEnumerable reactive pattern)
- **Internal Dependencies**: NotifyCollectionChangedEventReceiver (4.7)
- **Threading**: **Thread-safe** (ConcurrentDictionary base class)
- **Public Properties**:
  - `LastModifiedDateTime: DateTime?` - Timestamp last modification (Add/Update/Remove/Clear) (read-only)
  - `this[TKey key]: TValue` - Indexer with event notification (uses AddOrUpdate internally)
- **Public Methods**:
  - All method ConcurrentDictionary with overridami for event emission:
  - `AddOrUpdate<TArg>(key: TKey, addValueFactory: Func<TKey, TArg, TValue>, updateValueFactory: Func<TKey, TValue, TArg, TValue>, factoryArgument: TArg, out isAdded: bool) -> TValue` - Main method; emits Add or Replace event; returns whether tfromano (isAdded)
  - `AddOrUpdate(...)` - 3 overloady for different factory signatures
  - `GetOrAdd<TArg>(key: TKey, valueFactory: Func<TKey, TArg, TValue>, factoryArgument: TArg, out isAdded: bool) -> TValue` - Z isAdded output
  - `GetOrAdd(...)` - 3 overloady
  - `TryAdd(key: TKey, value: TValue) -> bool` - Emituje Add event
  - `TryRemove(key: TKey, out value: TValue) -> bool` - Emituje Remove event
  - `TryRemove(item: KeyValuePair<TKey, TValue>) -> bool` - Emituje Remove event
  - `TryUpdate(key: TKey, newValue: TValue, comparisonValue: TValue) -> bool` - Emituje Replace event
  - `Clear() -> void` - Emituje Reset event
  - `GetKeysEnumeratorAsync(stoppingToken: CancellationToken = default) -> IAsyncEnumerable<TKey>` - Reactive stream all keyy (snapshot + live events)
- **Events**:
  - `CollectionChanged: NotifyCollectionChangedEventHandler?` - Emitted on Add/Replace/Remove/Reset (try-catch ignored)
- **GetKeysEnumeratorAsync flow**:
  1. Returns snapshot Keys (lines 277-287)
  2. Returns events during snapshot load (lines 290-308) - deduplikacja
  3. Returns live events (lines 312-326)
  4. Unsubscribe in finally (line 330)
- **Notes**:
  - **Thread-safe**: All operations thread-safe (ConcurrentDictionary base)
  - **Event safety**: try-catch in CollectionChanged?.Invoke (ignores exceptions in subscribers)
  - **Reactive pattern**: GetKeysEnumeratorAsync + NotifyCollectionChangedEventReceiver for reactive streams
  - **Use case**: Shared cache with observability, real-time UI updates

### 4.4 ConcurrentObservablePartitionedQueueWithKeyContext<TKey, TValue, TContext>

- **Purpose**: Observable partitioned queue with per-key context, auto-processing (await foreach) i background expiration cleanup (24h + optional KeyExpireTime).
- **Patterns**: Partitioned queue pattern, Observer pattern, Background service pattern, Context per partition pattern
- **Internal Dependencies**: ConcurrentObservableDictionary<TKey, ...> (4.3), ConcurrentObservableQueue<T> (4.5), BackgroundCheckClearKeysService (inner class)
- **Threading**:
  - **Thread-safe**: ConcurrentObservableDictionary base
  - **Per-partition task**: StartToListenToProcessPartitionKey (await foreach) per partition
  - **Background cleanup**: BackgroundCheckClearKeysService (BackgroundService) every 10min
- **Public Properties**:
  - `KeyCount: int` - Number of active partitions (read-only)
  - `Count: int` - Sum items in all partitions (read-only)
  - `ProcessAction: Func<TKey, TValue, ConcurrentQueue<TValue>, TContext?, Task>?` - Callback per item (settable, default: Task.CompletedTask)
  - `KeyContextCreateAction: Func<TContext?>?` - Factory for per-key context (settable, default: null)
- **Public Methods**:
  - `ConcurrentObservablePartitionedQueueWithKeyContext(keyContextCreateAction: Func<TContext?>? = default, processsAction: Func<TKey, TValue, ConcurrentQueue<TValue>, TContext?, Task>? = default, cancelToken: CancellationToken = default)` - Constructor; startuje background cleanup
  - `Enqueue(partitionedKey: TKey, item: TValue, keyExpireTime: DateTime? = null) -> void` - Adds item; creates partition if not exists; calls KeyContext.ItemEnqueueing
  - `TryRemovePartitionKey(partitionedKey: TKey, out queueItem: ConcurrentObservablePartitionedQueueItem<TValue, TContext?>) -> bool` - Removes partition (cancels listener task)
- **Context pattern**:
  - **TContext constraint**: where TContext : IPartitionedQueueContextItem
  - **ItemEnqueueing hook**: KeyContext?.ItemEnqueueing(partitionedKey, item) called before Enqueue (line 94)
- **Listener flow**:
  - KeyCollectionChanged event → Add action → StartToListenToProcessPartitionKey (line 58)
  - await foreach queueItem.Queue.DequeueAsync (line 77) → processsAction per item (line 81)
  - Remove action → CancelToken.Cancel() (line 68)
- **Cleanup flow** (BackgroundCheckClearKeysService):
  - Runs every 10min (line 130-133)
  - Removes keys where: LastUpdateTime < now-24h OR KeyExpireTime < now (lines 137-143)
- **Notes**:
  - **Context use case**: Per-key aggregation state (counters, rate limits, session data)
  - **Auto-cleanup**: 24h idle timeout + optional KeyExpireTime (manual expiration)
  - **Backpressure**: ConcurrentObservableQueue (unbounded) - no backpressure control
  - **Cancellation**: Per-partition CancellationTokenSource for graceful shutdown

### 4.5 ConcurrentObservablePartitionedQueue<TKey, TValue>

- **Purpose**: Simplified wrapper over ConcurrentObservablePartitionedQueueWithKeyContext without context (uses DefaultPartitionedQueueContextItem).
- **Patterns**: Facade pattern, Template pattern
- **Internal Dependencies**: ConcurrentObservablePartitionedQueueWithKeyContext (4.4)
- **Threading**: Inherits with base class
- **Public Methods**:
  - `ConcurrentObservablePartitionedQueue(processsAction: Func<TKey, TValue, ConcurrentQueue<TValue>, Task>? = default, cancelToken: CancellationToken = default)` - Constructor; adapter for processsAction (removes TContext pairmeter)
- **Notes**: Preferred for simple use cases without per-key context

### 4.6 ConcurrentObservableQueue<T>

- **Purpose**: Thread-safe observable queue with INotifyCollectionChanged events, LastEnqueue/LastDequeue timestamps i DequeueAsync (IAsyncEnumerable reactive pattern).
- **Patterns**: Observer pattern, Decorator pattern (wraps ConcurrentQueue), Reactive pattern (DequeueAsync)
- **Internal Dependencies**: NotifyCollectionChangedEventReceiver (4.7)
- **Threading**: **Thread-safe** (ConcurrentQueue base class)
- **Public Properties**:
  - `LastEnqueueDateTime: DateTime?` - Timestamp last Enqueue (read-only)
  - `LastDequeueDateTime: DateTime?` - Timestamp last Dequeue (read-only)
- **Public Methods**:
  - `Enqueue(item: T) -> void` - Enqueue with event emission (Add action)
  - `TryDequeue(out result: T) -> bool` - Dequeue with event emission (Remove action)
  - `Clear() -> void` - Clear with event emission (Reset action)
  - `DequeueAsync(stoppingToken: CancellationToken = default) -> IAsyncEnumerable<T>` - Reactive stream: snapshot + live dequeue events
- **Events**:
  - `CollectionChanged: NotifyCollectionChangedEventHandler?` - Emitted on Enqueue/Dequeue/Clear (try-catch ignored)
- **DequeueAsync flow**:
  1. Dequeue snapshot (currentCount) (lines 88-96)
  2. await foreach live events (lines 98-110) → TryDequeue per event
  3. Unsubscribe in finally (line 114)
- **Notes**:
  - **Thread-safe**: All operations thread-safe (ConcurrentQueue base)
  - **Reactive pattern**: DequeueAsync for consumer patterns (does not block, IAsyncEnumerable)
  - **Use case**: Partitioned queue item storage (ConcurrentObservablePartitionedQueueItem.Queue, line 159)

### 4.7 ConcurrentPriorityQueue<T>

- **Purpose**: Thread-safe priority queue (array of ConcurrentQueue[priority]) for fixed priority levels (max 100).
- **Patterns**: Priority queue pattern, Bucket pattern (priority buckets)
- **Internal Dependencies**: None
- **Threading**: **Thread-safe** (ConcurrentQueue[] base)
- **Public Properties**:
  - `IsEmpty: bool` - Whether all priority buckets are empty (read-only)
  - `Count: int` - Sum items in all buckets (read-only)
  - `IsSynchronized: bool` - false (read-only)
- **Public Methods**:
  - `ConcurrentPriorityQueue(maxPriority: short)` - Constructor; maxPriority must be <= 100
  - `Enqueue(item: T, priority: short) -> void` - Enqueue to bucket[priority]; throws if priority >= maxPriority
  - `TryDequeue(out item: T, out priority: short) -> bool` - Dequeue with highest priority (lowest index first)
  - `TryPeek(out item: T?) -> bool` - Peek with highest priority (lowest index first)
  - `ToArray() -> T[]` - Returns all items in priority order (slow path)
  - `TryAdd(item: T) -> bool` - IProducerConsumerCollection interface (enqueues with priority=0)
  - `TryTake(out item: T) -> bool` - IProducerConsumerCollection interface (dequeues)
- **Notes**:
  - **Priority order**: 0 = highest priority, maxPriority-1 = lowest priority
  - **Fixed buckets**: Array size fixed in constructor (not grows)
  - **Performance**: O(maxPriority) for Dequeue/Peek (linear scan); O(1) for Enqueue
  - **Use case**: Task scheduling with priority levels (PartitionedConcurrentConsumer, line 77)

### 4.8 NotifyCollectionChangedEventReceiver

- **Purpose**: Helper to conversion INotifyCollectionChanged events na async IAsyncEnumerable stream (Channel-based).
- **Patterns**: Adapter pattern, Pub-Sub pattern, Reactive pattern
- **Internal Dependencies**: None
- **Threading**: Thread-safe (Channel<NotifyCollectionChangedEventArgs> unbounded)
- **Public Properties**:
  - `Count: int` - Number of pending events in channel (read-only)
  - `Unsubscribed: bool` - Whether unsubscribed (read-only)
- **Public Methods static**:
  - `Subscribe(notifyCollectionChanged: INotifyCollectionChanged, filterAction: Func<object?, NotifyCollectionChangedEventArgs, bool>? = null) -> NotifyCollectionChangedEventReceiver` - Factory method
- **Public Methods**:
  - `GetNextEventAsync(stoppingToken: CancellationToken = default) -> ValueTask<NotifyCollectionChangedEventArgs?>` - Async read next event (returns null on cancellation)
  - `Unsubscribe() -> void` - Unsubscribe from events i Complete channel writer (idempotent)
- **Flow**:
  - Subscribe → CollectionChanged event handler → filter → Channel.Writer.WriteAsync (line 30-31)
  - Consumer calls GetNextEventAsync → Channel.Reader.ReadAsync (line 38)
- **Notes**:
  - **Filter pattern**: filterAction for selective event processing (e.g. only Add events)
  - **Backpressure**: Unbounded channel (can grow indefinitely if consumer slow)
  - **Use case**: ConcurrentObservableDictionary.GetKeysEnumeratorAsync (line 273), ConcurrentObservableQueue.DequeueAsync (line 83)

### 4.9 RateLimitWithPriority

- **Purpose**: Rate limiter with priority queue (PriorityQueue<TaskCompletionSource, int>) i MinWaitTime enforcement.
- **Patterns**: Rate limiting pattern, Priority queue pattern, TaskCompletionSource async pattern
- **Internal Dependencies**: None
- **Threading**: Thread-safe (SemaphoreSlim protection for queue access)
- **Public Properties**:
  - `MinWaitTime: TimeSpan` - Minimum time waiting between dequeue (default: 10s) (settable)
- **Public Methods**:
  - `WaitWithLowToHighPriority(priority: int, cancellationToken: CancellationToken) -> Task` - Async await; lower priority value = higher precedence
  - `WaitWithHighToLowPriority(priority: int, cancellationToken: CancellationToken) -> Task` - Async await; higher priority value = higher precedence (negates priority)
- **Flow**:
  1. Enqueue(TaskCompletionSource, priority) to PriorityQueue (lines 52-54)
  2. Process() task: await 900ms → SemaphoreSlim lock → Dequeue → TrySetResult (lines 13-40)
  3. WaitAsync TaskCompletionSource.Task with timeout (lines 65)
  4. Post-delay: MinWaitTime - passedTime (lines 68-70)
- **Notes**:
  - **Rate limiting**: MinWaitTime enforcement (lines 68-70) + 900ms delay (line 18)
  - **Priority**: Lower int = higher priority (standard convention)
  - **Timeout**: Max await = max(60s, MinWaitTime) (line 63)
  - **Use case**: API rate limiting with priority (premium users = lower priority value)

### 4.10 ConcurrentQueueExtension

- **Purpose**: Extension method for bulk pairllel dequeue with ConcurrentQueue.
- **Patterns**: Extension Method pattern, Parallel pattern
- **Internal Dependencies**: None
- **Threading**: Parallel.ForAll for dequeue (PLINQ)
- **Public Methods**:
  - `GetDequeueAllItemsAndReturn<T>(this ConcurrentQueue<T> concurrentQueue) -> List<T>` - Parallel dequeue all items (AsParallel)
- **Notes**:
  - **Performance**: AsParallel can być faster for large queue (trade-off: overhead pairllel scheduling)
  - **Race condition safety**: maxCount snapshot (line 9); can miss items enqueued during dequeue
  - **Allocation**: Creates intermediate ConcurrentQueue (line 10) + List conversion (line 21)

---

## 5. COLLECTIONS.CONSUMERS

### Purpose
Advanced partitioned consumer with priority, bounded channels, batch processing i pluggable task execution strategies.

### 5.1 PartitionedConcurrentConsumer<TKey, TPayload>

- **Purpose**: Enterprise-grade partitioned consumer with priority queue, bounded channels, batch processing, object pooling i pluggable task execution strategy (Limited/Custom).
- **Patterns**: Partitioned queue pattern, Strategy pattern (BaseTaskExecutionStrategy), Observer pattern (event-based callbacks), Object pooling, Versioned CAS (Compare-And-Swap) for partition updates
- **Internal Dependencies**: ObjectPoolNoCreate<Channel<TPayload>> (Caching/ObjectPool.cs), BaseTaskExecutionStrategy (Threading/TaskStrategies/)
- **Threading**:
  - **Thread-safe**: ConcurrentDictionary for partitions, Channel<TPayload> per partition
  - **Task strategy**: Pluggable (LimitedTaskExecutionStrategy default) - controls concurrency
  - **Per-partition processing**: ProcessOnePartitionAsync task per partition
- **Public Properties**:
  - `InternalPoolMaxSize: short` - Max channel pool size (default: 1000) (settable, lazy initialization)
- **Public events**:
  - `ConsumeAsync: AsyncConsumePayloadsEventHandler?` - Async callback per batch (preferred)
  - `Consume: EventHandler<ConsumePayloadsEventArgs<TKey, TPayload>>?` - Sync callback per batch
- **Public Methods**:
  - `PartitionedConcurrentConsumer(taskStrategy: BaseTaskExecutionStrategy? = null)` - Constructor; default strategy = LimitedTaskExecutionStrategy (shared static)
  - `Enqueue(enqueueItem: EnqueueItem<TKey, TPayload>) -> void` - Enqueue with partition key, priority, bounded channel config
  - `CancelPartitionKey(partitionKey: TKey) -> void` - Removes partition (cancels processing)
  - `Stop() -> void` - Graceful shutdown (cancels all tasks, disposes strategy)
  - `WaitForAllFinish() -> Task` - Async await aż all partitions are empty
- **EnqueueItem<TKey, TPayload> structure**:
  - `PartitionKey: TKey` - Partition key
  - `Data: TPayload` - Payload item
  - `BoundedMaxCount: int?` - Optional bounded channel capacity (null = unbounded)
  - `ModeWhenMaxCountReached: BoundedChannelFullMode` - Backpressure strategy (default: DropWrite)
  - `PayloadMaxBatchSize: short` - Batch size for callback (default: 1000)
  - `IsCallbackSyncMethod: bool` - Whether use Consume (sync) vs ConsumeAsync (async)
  - `Priority: short` - Task priority for execution strategy
- **Enqueue flow** (versioned CAS):
  1. TryGetValue(key) → not found: TryAdd (lines 51-80)
  2. Found: VersionId+1 → TryUpdate (lines 84-99) - spin-loop to success
  3. Enqueue to Channel.Writer.TryWrite (lines 64, 98)
- **Processing flow**:
  - ProcessOnePartitionAsync (strategy callback):
    1. GetAndSendOnePayload → GetOnePayload (batch up to PayloadMaxBatchSize) (lines 174-200, 236-252)
    2. PublishOnePayload → Consume/ConsumeAsync event (lines 202-234)
    3. TryClearPartitionKey if payloads finished (lines 127, 131-172)
  - Return false if more items → strategy re-enqueues task (line 196)
- **Cleanup flow**:
  - TryClearPartitionKey: TryRemove with key-value match (line 143) for safety
  - Channel pooling: GiveBack to pool after remove (line 168)
- **Notes**:
  - **Strategy pattern**: BaseTaskExecutionStrategy abstracts task scheduling (Limited/Unlimited/Custom concurrency)
  - **Versioned updates**: VersionId prevents ABA problem (lines 61, 88)
  - **Bounded channels**: Per-partition backpressure (BoundedMaxCount + DropWrite/DropOldest/Wait)
  - **Priority**: Task priority passed to strategy (line 77)
  - **Object pooling**: Channel pooling (ChannelCreator) for zero-allocation (lines 291-340)
  - **Use case**: Kafka-style message processing with backpressure, priority i batch optimization

---

## 6. COLLECTIONS.WORKERS

### Purpose
Single-partition workers with auto-processing: AutoWorker (per-item) i AutoBatchWorker (batch processing).

### 6.1 AutoWorker<TInput, TOutput>

- **Purpose**: Single-partition auto-processing worker (per-item callback) with async result (TaskCompletionSource) i optional fire-and-forget mode.
- **Patterns**: Producer-Consumer pattern, Task completion pattern, Single-threaded processing (Interlocked flag)
- **Internal Dependencies**: AtuoWorkerResult<TOutput> (6.3)
- **Threading**:
  - **Thread-safe**: ConcurrentQueue for enqueue
  - **Single processsor**: Interlocked flag (_isProcessing) ensures only 1 ProcessQueueAsync task
- **Public Methods**:
  - `AutoWorker(processsItemAsync: Func<TInput, Task<TOutput>>)` - Constructor with async callback
  - `Enqueue(item: TInput, checkExpiredKeys: Action? = null) -> void` - Fire-and-forget enqueue
  - `EnqueueAsync(item: TInput, checkExpiredKeys: Action? = null) -> Task<AtuoWorkerResult<TOutput>>` - Async enqueue with result
- **Processing flow**:
  - TryStartProcessing → Interlocked.CompareExchange (line 55) → Task.Run ProcessQueueAsync (line 56)
  - ProcessQueueAsync: while TryDequeue → processsItemAsync → TrySetResult (lines 30-51)
  - After queue empty: checkExpiredKeys() callback (line 50)
- **Notes**:
  - **Single-threaded**: Only 1 ProcessQueueAsync task per worker (Interlocked flag)
  - **Fire-and-forget vs Async**: Enqueue (void) vs EnqueueAsync (Task<Result>)
  - **checkExpiredKeys**: Optional callback after processing all items (use case: cleanup)
  - **Use case**: Sequential processing per entity (e.g. per-user commands)

### 6.2 AutoBatchWorker<TInput>

- **Purpose**: Single-partition batch worker (batch callback) with configurable batch size i async result.
- **Patterns**: Producer-Consumer pattern, Batch processing pattern, Task completion pattern
- **Internal Dependencies**: None
- **Threading**:
  - **Thread-safe**: ConcurrentQueue for enqueue
  - **Single processsor**: Interlocked flag (_isProcessing)
- **Public Methods**:
  - `AutoBatchWorker(processsItemAsync: Func<List<TInput>, Task>, batchSize: int = 100)` - Constructor with batch callback i size
  - `Enqueue(item: TInput, checkExpiredKeys: Action? = null) -> void` - Fire-and-forget enqueue
  - `EnqueueAsync(item: TInput, checkExpiredKeys: Action? = null) -> Task<bool>` - Async enqueue with bool result (success/failure)
- **Processing flow**:
  - ProcessQueueAsync: to-while loop:
    1. TryDequeue up to batchSize (lines 39-46)
    2. processsItemAsync(batchData) (line 50)
    3. TrySetResult(true) for all TaskCompletionSource in batch (lines 52-53)
  - On exception: TrySetResult(false) (lines 57-59)
- **Notes**:
  - **Batch optimization**: Reduces overhead callbacks (bulk DB writes, bulk API calls)
  - **Partial batch**: Last batch can być < batchSize (line 62 - queue empty)
  - **Error handling**: Entire batch fails if exception (lines 56-59)
  - **Use case**: Bulk database inserts, bulk log writes

### 6.3 AtuoWorkerResult<T>

- **Purpose**: Result type for AutoWorker - Either<T, Exception> pattern.
- **Patterns**: Result pattern, Discriminated union (IsSuccess flag)
- **Public Properties**:
  - `Result: T?` - Success result (null if error)
  - `Exception: Exception?` - Error exception (null if success)
  - `IsSuccess: bool` - true if Exception == null (computed property)
- **Public Methods static** (internal):
  - `Error(ex: Exception) -> AtuoWorkerResult<T>` - Factory for error result
  - `Success(result: T) -> AtuoWorkerResult<T>` - Factory for success result
- **Notes**: Lightweight Either moover - allows na functional error handling

---

## NOTES & CROSS-REFERENCES

### Internal Relations

1. **Object Pooling hierarchy**:
   - `ObjectPool<T>` (Caching) ← used through `AutoConcurrentPartitionedQueue<T>` (line 193)
   - `ObjectPoolNoCreate<T>` (Caching) ← used through `PartitionedConcurrentConsumer<TKey, TPayload>` (lines 293-294)

2. **Observable patterns**:
   - `NotifyCollectionChangedEventReceiver` (Concurrent) ← used through:
     - `ConcurrentObservableDictionary.GetKeysEnumeratorAsync` (line 273)
     - `ConcurrentObservableQueue.DequeueAsync` (line 83)
   - `ConcurrentObservableQueue<T>` ← used through `ConcurrentObservablePartitionedQueueItem<T, TContext>` (line 159)

3. **Partitioned queue evolution**:
   - **V1**: `AutoConcurrentPartitionedQueue<T>` - main queue + partition queues + throttling + cleanup
   - **V2**: `AutoConcurrentPartitionedQueueV2<T>` - direct partition queues + timeout cleanup (simpler)
   - **Observable**: `ConcurrentObservablePartitionedQueue<TKey, TValue>` - event-based + per-partition tasks + 24h expiration
   - **Enterprise**: `PartitionedConcurrentConsumer<TKey, TPayload>` - priority + bounded channels + strategy pattern + pooling

4. **Apache Pulsar integration**:
   - `SmartpulsePulsarClient` uses:
     - JSON serialization per message (System.Text.Json)
     - Compression types: None/LZ4/Zlib/Zstd/Snappy
     - `DateTimeToObjectConverter` i `IdToObjectConverter` for MongoDB-compatible JSON

5. **Caching patterns**:
   - `MemoryCacheHelper`: Cache-aside + double-checked locking + versioned swap
   - Used through application via DI (`MemoryCacheExtension`)

### Design Patterns Used Together

1. **Producer-Consumer + Partitioning**:
   - `AutoConcurrentPartitionedQueue` + `AutoConcurrentPartitionedQueueV2` + `ConcurrentObservablePartitionedQueue` + `PartitionedConcurrentConsumer`
   - Use case: High-throughput event processing per entity (Kafka-style)

2. **Observable + Reactive**:
   - `ConcurrentObservableDictionary.GetKeysEnumeratorAsync` + `NotifyCollectionChangedEventReceiver`
   - Use case: Real-time cache observability, UI updates

3. **Object Pooling + Batch Processing**:
   - `ObjectPool<List<T>>` + `AutoConcurrentPartitionedQueue.ProcessAllItemsInQueue`
   - Benefit: Zero-allocation bulk processing

4. **Priority + Rate Limiting**:
   - `ConcurrentPriorityQueue` + `RateLimitWithPriority`
   - Use case: API throttling with premium users

### Threading Considerations

1. **Thread-safe collections**:
   - All Collections.Concurrent classes są thread-safe (ConcurrentDictionary/ConcurrentQueue base)
   - Collections (non-concurrent) are NOT thread-safe (ObservableDictionary)

2. **Concurrency patterns**:
   - **Single-threaded processing**: AutoWorker, AutoBatchWorker (Interlocked flag)
   - **Multi-threaded with throttling**: AutoConcurrentPartitionedQueue (ConsumerThrottleCount)
   - **Per-partition pairllelism**: PartitionedConcurrentConsumer (strategy-based)
   - **Long-running tasks**: AutoConcurrentPartitionedQueue.mainQueueProcessTask (TaskCreationOptions.LongRunning)

3. **Backpressure mechanisms**:
   - **Bounded channels**: PartitionedConcurrentConsumer (BoundedChannelFullMode: Wait/DropWrite/DropOldest)
   - **Priority**: RateLimitWithPriority, ConcurrentPriorityQueue
   - **Throttling**: AutoConcurrentPartitionedQueue (ConsumerThrottleCount)

### Performance Notes

1. **Bulk processing**:
   - AutoConcurrentPartitionedQueue: Dynamic bulk sizing (80% of queue count)
   - AutoBatchWorker: Fixed batch size (configurable)
   - PartitionedConcurrentConsumer: PayloadMaxBatchSize (default 1000)

2. **Zero-allocation patterns**:
   - Object pooling (List<T>, Channel<T>)
   - ArrayPool/MemoryPool not used (potential optimization)

3. **Cleanup strategies**:
   - **Lazy**: AutoConcurrentPartitionedQueue (ClearMaxCounter iterations)
   - **Timeout-based**: AutoConcurrentPartitionedQueueV2 (MaxWaitTimeWhenNoItemInQueue)
   - **Background service**: ConcurrentObservablePartitionedQueue (every 10min + 24h expiration)

### Recommended Use Cases

1. **Apache Pulsar**: Message bus integration with compression, JSON/MongoDB serialization
2. **Caching**: MemoryCacheHelper for cache-aside with stampede prevention
3. **Partitioned processing**:
   - High-throughput: AutoConcurrentPartitionedQueue/V2
   - Enterprise: PartitionedConcurrentConsumer (priority, backpressure, strategy)
   - Observable: ConcurrentObservablePartitionedQueue (event-driven)
4. **Workers**:
   - Sequential per-entity: AutoWorker
   - Bulk operations: AutoBatchWorker
5. **Priority/Rate limiting**: ConcurrentPriorityQueue + RateLimitWithPriority for API throttling