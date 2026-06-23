# Capstone Project 3: Distributed Cache Service — Client LLD (Senior Level)

## Project Overview

Design the in-process Java client library for a distributed cache. This is NOT the server — it is the client-side cache abstraction that application code uses. Focus is on: clean interface design, multiple eviction strategies, read-through/write-through, metrics, and configurability.

**Learning Objectives:**
- Apply Strategy pattern for pluggable eviction policies
- Apply Decorator pattern to layer behaviors (metrics, read-through, write-through)
- Apply Template Method for consistent cross-cutting concerns
- Apply Builder for complex configuration
- Understand consistent hashing for partitioning

**Difficulty:** Senior Engineer Level  
**Estimated Time:** 4-6 hours  
**Topics Covered:** Strategy, Decorator, Template Method, Builder, Factory, Consistent Hashing

---

## Requirements

### Functional Requirements
1. Cache interface: get, put, delete, getOrLoad (cache-aside helper)
2. Eviction strategies: LRU (Least Recently Used), LFU (Least Frequently Used), TTL (Time To Live), FIFO
3. Cache partitioning: consistent hashing for key-to-partition mapping (client-side only)
4. Read-through: on cache miss, auto-load from a provided CacheLoader
5. Write-through: on put, auto-write to backing store via CacheWriter
6. Cache warming: pre-populate cache from a data source on startup
7. Batch operations: getAll, putAll, deleteAll
8. Metrics: hit count, miss count, eviction count, hit rate
9. Configuration: max size, eviction policy, TTL default, partition count

### Non-Functional Requirements
1. O(1) get and put for LRU and FIFO
2. O(log n) for LFU using min-heap
3. Thread-safe option via ReadWriteLock
4. Minimal object allocation in hot paths

---

## Class Diagram

```
+------------------+       +-------------------+       +--------------------+
|   Cache<K,V>     |<------|MetricsCollecting   |       |  CacheConfig       |
|  (interface)     |       |Cache<K,V>          |       |  maxSize           |
|  get(K)          |       |(decorator)         |       |  evictionPolicy    |
|  put(K,V)        |       +-------------------+|       |  defaultTtlSecs    |
|  putWithTtl(K,V) |                            |       |  partitionCount    |
|  delete(K)       |<------|ReadThroughCache<K,V>       +--------------------+
|  getOrLoad(K,    |       |(decorator)         |
|   CacheLoader)   |<------|WriteThroughCache<K,V>
|  getAll(keys)    |       |(decorator)         |
|  putAll(map)     |       +--------------------+
|  deleteAll(keys) |
|  getStats()      |<------|AbstractCache<K,V>  |------>|EvictionPolicy<K,V>|
|  clear()         |       |(abstract)          |       |(interface)         |
|  size()          |       +--------------------+       |selectEviction      |
+------------------+              ^                     |Candidate()         |
                                  |                     |onAccess(K)         |
                         +--------+--------+            |onPut(K,V)          |
                         |                |             +--------------------+
               +---------+------+ +-------+--------+           ^
               |InMemoryCache   | |PartitionedCache |           |
               |<K,V>           | |<K,V>            |    +------+------+------+------+
               |(HashMap +      | |(consistent hash |    |LRU   |LFU   |TTL   |FIFO  |
               | EvictionPolicy)| | to N shards)    |    |Policy|Policy|Policy|Policy|
               +----------------+ +-----------------+    +------+------+------+------+
```

---

## Design Patterns Applied

| Pattern | Where Applied | Benefit |
|---|---|---|
| Strategy | EvictionPolicy implementations | Swap eviction algorithm without changing cache code |
| Decorator | MetricsCollectingCache, ReadThroughCache, WriteThroughCache | Layer behaviors without inheritance explosion |
| Builder | CacheConfig.Builder | Safe construction of complex config with defaults |
| Template Method | AbstractCache handles stats/logging, subclasses handle storage | Eliminate duplicate cross-cutting code |
| Factory | CacheFactory.create(CacheConfig) | Centralize wiring of decorators |

---

## Complete Java Implementation

### Core Types

```java
// CacheKey.java
package com.example.cache;

import java.util.Objects;

/**
 * Type-safe wrapper for cache keys.
 * Ensures consistent hashCode/equals contract regardless of K's implementation.
 */
public final class CacheKey<K> {
    private final K key;

    public CacheKey(K key) {
        if (key == null) throw new IllegalArgumentException("Cache key cannot be null");
        this.key = key;
    }

    public K getKey() {
        return key;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof CacheKey)) return false;
        CacheKey<?> cacheKey = (CacheKey<?>) o;
        return Objects.equals(key, cacheKey.key);
    }

    @Override
    public int hashCode() {
        return Objects.hash(key);
    }

    @Override
    public String toString() {
        return "CacheKey{" + key + "}";
    }
}
```

```java
// CacheEntry.java
package com.example.cache;

import java.time.Instant;

/**
 * Container for a cached value along with metadata needed by eviction policies.
 * Immutable value, mutable metadata fields accessed/updated by eviction policies.
 */
public class CacheEntry<V> {
    private final V value;
    private final Instant createdAt;
    private volatile Instant lastAccessedAt;
    private volatile long accessCount;
    private final Instant expiresAt; // null means no TTL

    public CacheEntry(V value, Instant createdAt, Instant expiresAt) {
        this.value = value;
        this.createdAt = createdAt;
        this.lastAccessedAt = createdAt;
        this.accessCount = 1L;
        this.expiresAt = expiresAt;
    }

    public V getValue() {
        return value;
    }

    public Instant getCreatedAt() {
        return createdAt;
    }

    public Instant getLastAccessedAt() {
        return lastAccessedAt;
    }

    public long getAccessCount() {
        return accessCount;
    }

    public Instant getExpiresAt() {
        return expiresAt;
    }

    public boolean isExpired() {
        return expiresAt != null && Instant.now().isAfter(expiresAt);
    }

    public void recordAccess() {
        this.lastAccessedAt = Instant.now();
        this.accessCount++;
    }

    @Override
    public String toString() {
        return "CacheEntry{value=" + value
                + ", createdAt=" + createdAt
                + ", lastAccessedAt=" + lastAccessedAt
                + ", accessCount=" + accessCount
                + ", expiresAt=" + expiresAt + "}";
    }
}
```

```java
// EvictionPolicy.java
package com.example.cache;

/**
 * Enum representing the supported eviction policies.
 */
public enum EvictionPolicy {
    LRU,
    LFU,
    TTL,
    FIFO
}
```

```java
// CacheConfig.java
package com.example.cache;

/**
 * Immutable configuration for a cache instance.
 * Use the inner Builder for fluent construction.
 */
public final class CacheConfig {
    private final int maxSize;
    private final EvictionPolicy evictionPolicy;
    private final long defaultTtlSeconds;  // 0 means no TTL
    private final int partitionCount;
    private final boolean threadSafe;

    private CacheConfig(Builder builder) {
        this.maxSize = builder.maxSize;
        this.evictionPolicy = builder.evictionPolicy;
        this.defaultTtlSeconds = builder.defaultTtlSeconds;
        this.partitionCount = builder.partitionCount;
        this.threadSafe = builder.threadSafe;
    }

    public int getMaxSize() { return maxSize; }
    public EvictionPolicy getEvictionPolicy() { return evictionPolicy; }
    public long getDefaultTtlSeconds() { return defaultTtlSeconds; }
    public int getPartitionCount() { return partitionCount; }
    public boolean isThreadSafe() { return threadSafe; }

    @Override
    public String toString() {
        return "CacheConfig{maxSize=" + maxSize
                + ", evictionPolicy=" + evictionPolicy
                + ", defaultTtlSeconds=" + defaultTtlSeconds
                + ", partitionCount=" + partitionCount
                + ", threadSafe=" + threadSafe + "}";
    }

    public static Builder builder() {
        return new Builder();
    }

    public static final class Builder {
        private int maxSize = 1000;
        private EvictionPolicy evictionPolicy = EvictionPolicy.LRU;
        private long defaultTtlSeconds = 0;
        private int partitionCount = 1;
        private boolean threadSafe = false;

        public Builder maxSize(int maxSize) {
            if (maxSize <= 0) throw new IllegalArgumentException("maxSize must be positive");
            this.maxSize = maxSize;
            return this;
        }

        public Builder evictionPolicy(EvictionPolicy evictionPolicy) {
            if (evictionPolicy == null) throw new IllegalArgumentException("evictionPolicy cannot be null");
            this.evictionPolicy = evictionPolicy;
            return this;
        }

        public Builder defaultTtlSeconds(long defaultTtlSeconds) {
            if (defaultTtlSeconds < 0) throw new IllegalArgumentException("defaultTtlSeconds cannot be negative");
            this.defaultTtlSeconds = defaultTtlSeconds;
            return this;
        }

        public Builder partitionCount(int partitionCount) {
            if (partitionCount <= 0) throw new IllegalArgumentException("partitionCount must be positive");
            this.partitionCount = partitionCount;
            return this;
        }

        public Builder threadSafe(boolean threadSafe) {
            this.threadSafe = threadSafe;
            return this;
        }

        public CacheConfig build() {
            return new CacheConfig(this);
        }
    }
}
```

```java
// CacheStats.java
package com.example.cache;

import java.util.concurrent.atomic.AtomicLong;

/**
 * Thread-safe metrics container for a cache instance.
 * hitRate is computed on demand from hitCount and missCount.
 */
public class CacheStats {
    private final AtomicLong hitCount = new AtomicLong(0);
    private final AtomicLong missCount = new AtomicLong(0);
    private final AtomicLong evictionCount = new AtomicLong(0);
    private final AtomicLong putCount = new AtomicLong(0);

    public void recordHit() {
        hitCount.incrementAndGet();
    }

    public void recordMiss() {
        missCount.incrementAndGet();
    }

    public void recordEviction() {
        evictionCount.incrementAndGet();
    }

    public void recordPut() {
        putCount.incrementAndGet();
    }

    public long getHitCount() {
        return hitCount.get();
    }

    public long getMissCount() {
        return missCount.get();
    }

    public long getEvictionCount() {
        return evictionCount.get();
    }

    public long getPutCount() {
        return putCount.get();
    }

    public double getHitRate() {
        long hits = hitCount.get();
        long total = hits + missCount.get();
        return total == 0 ? 0.0 : (double) hits / total;
    }

    public void reset() {
        hitCount.set(0);
        missCount.set(0);
        evictionCount.set(0);
        putCount.set(0);
    }

    @Override
    public String toString() {
        return "CacheStats{hitCount=" + getHitCount()
                + ", missCount=" + getMissCount()
                + ", evictionCount=" + getEvictionCount()
                + ", putCount=" + getPutCount()
                + ", hitRate=" + String.format("%.2f%%", getHitRate() * 100) + "}";
    }
}
```

### Core Interface

```java
// Cache.java
package com.example.cache;

import java.util.Collection;
import java.util.Map;
import java.util.Optional;
import java.util.function.Function;

/**
 * Core cache interface.
 * All implementations and decorators implement this contract.
 *
 * @param <K> Key type
 * @param <V> Value type
 */
public interface Cache<K, V> {

    /**
     * Retrieve a value by key. Returns empty Optional on cache miss.
     */
    Optional<V> get(K key);

    /**
     * Store a key-value pair. Uses the cache's default TTL if configured.
     */
    void put(K key, V value);

    /**
     * Store a key-value pair with an explicit TTL in seconds.
     */
    void putWithTtl(K key, V value, long ttlSeconds);

    /**
     * Remove a key from the cache.
     */
    void delete(K key);

    /**
     * Cache-aside helper: return the cached value if present, otherwise invoke
     * loader, store the result, and return it.
     */
    V getOrLoad(K key, Function<K, V> loader);

    /**
     * Batch get. Returns a map of found key-value pairs; absent keys are omitted.
     */
    Map<K, V> getAll(Collection<K> keys);

    /**
     * Batch put using the default TTL.
     */
    void putAll(Map<K, V> entries);

    /**
     * Batch delete.
     */
    void deleteAll(Collection<K> keys);

    /**
     * Return a snapshot of cache statistics.
     */
    CacheStats getStats();

    /**
     * Remove all entries from the cache.
     */
    void clear();

    /**
     * Return the number of entries currently in the cache.
     */
    int size();
}
```

### Eviction Policy Interfaces and Implementations

```java
// EvictionPolicyStrategy.java
package com.example.cache.eviction;

import com.example.cache.CacheEntry;

/**
 * Strategy interface for eviction algorithms.
 * Each implementation maintains its own bookkeeping data structures.
 *
 * @param <K> Key type
 * @param <V> Value type
 */
public interface EvictionPolicyStrategy<K, V> {

    /**
     * Called when a key is accessed (get hit). Update bookkeeping.
     */
    void onAccess(K key, CacheEntry<V> entry);

    /**
     * Called when a new key-value pair is put into the cache.
     */
    void onPut(K key, CacheEntry<V> entry);

    /**
     * Called when a key is removed explicitly (delete or clear).
     */
    void onRemove(K key);

    /**
     * Select the key that should be evicted next.
     * The caller is responsible for actually removing it from the cache map.
     */
    K selectEvictionCandidate();

    /**
     * Clear all internal bookkeeping state.
     */
    void clear();
}
```

```java
// LruEvictionPolicy.java
package com.example.cache.eviction;

import com.example.cache.CacheEntry;

import java.util.LinkedHashMap;
import java.util.Map;

/**
 * LRU eviction using a LinkedHashMap in access-order mode.
 *
 * LinkedHashMap with accessOrder=true maintains insertion/access order in O(1).
 * The least-recently-used entry is always at the head of the iteration order.
 *
 * Time complexity:
 *   onAccess: O(1) — LinkedHashMap moves accessed entry to tail
 *   onPut:    O(1)
 *   selectEvictionCandidate: O(1) — peek at head
 */
public class LruEvictionPolicy<K, V> implements EvictionPolicyStrategy<K, V> {

    // accessOrder=true: iteration order reflects access time (LRU at head)
    private final LinkedHashMap<K, Boolean> accessOrderMap;

    public LruEvictionPolicy(int maxSize) {
        this.accessOrderMap = new LinkedHashMap<K, Boolean>(maxSize, 0.75f, true) {
            // Not using removeEldestEntry here; the cache itself controls eviction
        };
    }

    @Override
    public void onAccess(K key, CacheEntry<V> entry) {
        // Accessing the key causes LinkedHashMap to move it to the tail (most recently used)
        accessOrderMap.get(key);
    }

    @Override
    public void onPut(K key, CacheEntry<V> entry) {
        accessOrderMap.put(key, Boolean.TRUE);
    }

    @Override
    public void onRemove(K key) {
        accessOrderMap.remove(key);
    }

    @Override
    public K selectEvictionCandidate() {
        if (accessOrderMap.isEmpty()) {
            throw new IllegalStateException("No entries available for eviction");
        }
        // Head of access-order map is the least recently used
        return accessOrderMap.keySet().iterator().next();
    }

    @Override
    public void clear() {
        accessOrderMap.clear();
    }
}
```

```java
// LfuEvictionPolicy.java
package com.example.cache.eviction;

import com.example.cache.CacheEntry;

import java.util.Comparator;
import java.util.HashMap;
import java.util.Map;
import java.util.PriorityQueue;

/**
 * LFU eviction using a frequency map and a min-heap.
 *
 * The min-heap always keeps the minimum-frequency key at the top.
 * On ties, the key with the earlier insertion time is evicted (approximation).
 *
 * Time complexity:
 *   onAccess: O(log n) — heap restructure after frequency update
 *   onPut:    O(log n)
 *   selectEvictionCandidate: O(1) — peek at heap root
 */
public class LfuEvictionPolicy<K, V> implements EvictionPolicyStrategy<K, V> {

    private final Map<K, Long> frequencyMap = new HashMap<>();
    private final Map<K, Long> insertionOrderMap = new HashMap<>();
    private final PriorityQueue<K> minHeap;
    private long insertionCounter = 0;

    public LfuEvictionPolicy() {
        this.minHeap = new PriorityQueue<>(Comparator
                .comparingLong((K k) -> frequencyMap.getOrDefault(k, 0L))
                .thenComparingLong(k -> insertionOrderMap.getOrDefault(k, 0L)));
    }

    @Override
    public void onAccess(K key, CacheEntry<V> entry) {
        if (!frequencyMap.containsKey(key)) return;
        // Remove, increment frequency, re-insert to trigger heap re-order
        minHeap.remove(key);
        frequencyMap.merge(key, 1L, Long::sum);
        minHeap.offer(key);
    }

    @Override
    public void onPut(K key, CacheEntry<V> entry) {
        frequencyMap.put(key, 1L);
        insertionOrderMap.put(key, insertionCounter++);
        minHeap.offer(key);
    }

    @Override
    public void onRemove(K key) {
        minHeap.remove(key);
        frequencyMap.remove(key);
        insertionOrderMap.remove(key);
    }

    @Override
    public K selectEvictionCandidate() {
        if (minHeap.isEmpty()) {
            throw new IllegalStateException("No entries available for eviction");
        }
        return minHeap.peek();
    }

    @Override
    public void clear() {
        frequencyMap.clear();
        insertionOrderMap.clear();
        minHeap.clear();
        insertionCounter = 0;
    }
}
```

```java
// TtlEvictionPolicy.java
package com.example.cache.eviction;

import com.example.cache.CacheEntry;

import java.time.Instant;
import java.util.Comparator;
import java.util.HashMap;
import java.util.Map;
import java.util.PriorityQueue;

/**
 * TTL eviction: evicts the entry whose expiry time is earliest.
 *
 * Uses a min-heap ordered by expiresAt. Entries without an expiresAt
 * are placed at the end (treated as never-expiring; evicted last).
 *
 * Time complexity:
 *   onPut: O(log n)
 *   selectEvictionCandidate: O(1)
 */
public class TtlEvictionPolicy<K, V> implements EvictionPolicyStrategy<K, V> {

    private static final Instant FAR_FUTURE = Instant.MAX;

    private final Map<K, Instant> expiryMap = new HashMap<>();
    private final PriorityQueue<K> expiryHeap;

    public TtlEvictionPolicy() {
        this.expiryHeap = new PriorityQueue<>(
                Comparator.comparing(k -> expiryMap.getOrDefault(k, FAR_FUTURE))
        );
    }

    @Override
    public void onAccess(K key, CacheEntry<V> entry) {
        // TTL is fixed at insertion; access does not change expiry
    }

    @Override
    public void onPut(K key, CacheEntry<V> entry) {
        Instant expiry = entry.getExpiresAt() != null ? entry.getExpiresAt() : FAR_FUTURE;
        expiryMap.put(key, expiry);
        expiryHeap.offer(key);
    }

    @Override
    public void onRemove(K key) {
        expiryHeap.remove(key);
        expiryMap.remove(key);
    }

    @Override
    public K selectEvictionCandidate() {
        if (expiryHeap.isEmpty()) {
            throw new IllegalStateException("No entries available for eviction");
        }
        return expiryHeap.peek();
    }

    @Override
    public void clear() {
        expiryHeap.clear();
        expiryMap.clear();
    }
}
```

```java
// FifoEvictionPolicy.java
package com.example.cache.eviction;

import com.example.cache.CacheEntry;

import java.util.ArrayDeque;
import java.util.Deque;
import java.util.HashSet;
import java.util.Set;

/**
 * FIFO eviction: the first key inserted is the first evicted.
 *
 * Uses an ArrayDeque for O(1) insertion at tail and removal from head.
 * A HashSet tracks current keys to handle the case where onRemove is called
 * for a key that was explicitly deleted before eviction.
 *
 * Time complexity:
 *   onPut: O(1)
 *   selectEvictionCandidate: O(1) amortized (skip removed keys)
 */
public class FifoEvictionPolicy<K, V> implements EvictionPolicyStrategy<K, V> {

    private final Deque<K> insertionQueue = new ArrayDeque<>();
    private final Set<K> liveKeys = new HashSet<>();

    @Override
    public void onAccess(K key, CacheEntry<V> entry) {
        // FIFO does not change order on access
    }

    @Override
    public void onPut(K key, CacheEntry<V> entry) {
        if (!liveKeys.contains(key)) {
            insertionQueue.addLast(key);
            liveKeys.add(key);
        }
    }

    @Override
    public void onRemove(K key) {
        // Mark as removed; the queue will skip this key lazily
        liveKeys.remove(key);
    }

    @Override
    public K selectEvictionCandidate() {
        // Skip keys that were explicitly removed (deleted) before we could evict them
        while (!insertionQueue.isEmpty() && !liveKeys.contains(insertionQueue.peekFirst())) {
            insertionQueue.pollFirst();
        }
        if (insertionQueue.isEmpty()) {
            throw new IllegalStateException("No entries available for eviction");
        }
        return insertionQueue.peekFirst();
    }

    @Override
    public void clear() {
        insertionQueue.clear();
        liveKeys.clear();
    }
}
```

```java
// EvictionPolicyFactory.java
package com.example.cache.eviction;

import com.example.cache.EvictionPolicy;

/**
 * Factory that creates the correct EvictionPolicyStrategy given an EvictionPolicy enum value.
 */
public class EvictionPolicyFactory {

    private EvictionPolicyFactory() {}

    public static <K, V> EvictionPolicyStrategy<K, V> create(EvictionPolicy policy, int maxSize) {
        switch (policy) {
            case LRU:
                return new LruEvictionPolicy<>(maxSize);
            case LFU:
                return new LfuEvictionPolicy<>();
            case TTL:
                return new TtlEvictionPolicy<>();
            case FIFO:
                return new FifoEvictionPolicy<>();
            default:
                throw new IllegalArgumentException("Unknown eviction policy: " + policy);
        }
    }
}
```

### Cache Implementations

```java
// AbstractCache.java
package com.example.cache.impl;

import com.example.cache.Cache;
import com.example.cache.CacheStats;

import java.util.Collection;
import java.util.Map;
import java.util.Optional;
import java.util.function.Function;

/**
 * Template Method base class.
 * Handles CacheStats tracking uniformly; delegates storage operations to subclasses.
 * Subclasses implement doGet, doPut, doPutWithTtl, doDelete, doClear, doSize.
 */
public abstract class AbstractCache<K, V> implements Cache<K, V> {

    protected final CacheStats stats = new CacheStats();

    @Override
    public final Optional<V> get(K key) {
        Optional<V> result = doGet(key);
        if (result.isPresent()) {
            stats.recordHit();
        } else {
            stats.recordMiss();
        }
        return result;
    }

    @Override
    public final void put(K key, V value) {
        doPut(key, value);
        stats.recordPut();
    }

    @Override
    public final void putWithTtl(K key, V value, long ttlSeconds) {
        doPutWithTtl(key, value, ttlSeconds);
        stats.recordPut();
    }

    @Override
    public final void delete(K key) {
        doDelete(key);
    }

    @Override
    public final V getOrLoad(K key, Function<K, V> loader) {
        Optional<V> cached = get(key);
        if (cached.isPresent()) {
            return cached.get();
        }
        V loaded = loader.apply(key);
        if (loaded != null) {
            put(key, loaded);
        }
        return loaded;
    }

    @Override
    public final Map<K, V> getAll(Collection<K> keys) {
        return doGetAll(keys);
    }

    @Override
    public final void putAll(Map<K, V> entries) {
        doPutAll(entries);
    }

    @Override
    public final void deleteAll(Collection<K> keys) {
        doDeleteAll(keys);
    }

    @Override
    public final CacheStats getStats() {
        return stats;
    }

    @Override
    public final void clear() {
        doClear();
    }

    @Override
    public final int size() {
        return doSize();
    }

    // --- Abstract hooks for subclasses ---

    protected abstract Optional<V> doGet(K key);

    protected abstract void doPut(K key, V value);

    protected abstract void doPutWithTtl(K key, V value, long ttlSeconds);

    protected abstract void doDelete(K key);

    protected abstract Map<K, V> doGetAll(Collection<K> keys);

    protected abstract void doPutAll(Map<K, V> entries);

    protected abstract void doDeleteAll(Collection<K> keys);

    protected abstract void doClear();

    protected abstract int doSize();
}
```

```java
// InMemoryCache.java
package com.example.cache.impl;

import com.example.cache.CacheEntry;
import com.example.cache.CacheConfig;
import com.example.cache.eviction.EvictionPolicyFactory;
import com.example.cache.eviction.EvictionPolicyStrategy;

import java.time.Instant;
import java.util.Collection;
import java.util.HashMap;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.Optional;

/**
 * Single-partition in-memory cache backed by a HashMap.
 * Eviction is delegated to the plugged-in EvictionPolicyStrategy.
 *
 * When the cache reaches maxSize, selectEvictionCandidate() is called
 * before each new insertion to free one slot.
 */
public class InMemoryCache<K, V> extends AbstractCache<K, V> {

    private final Map<K, CacheEntry<V>> store;
    private final EvictionPolicyStrategy<K, V> evictionPolicy;
    private final int maxSize;
    private final long defaultTtlSeconds;

    public InMemoryCache(CacheConfig config) {
        this.maxSize = config.getMaxSize();
        this.defaultTtlSeconds = config.getDefaultTtlSeconds();
        this.store = new LinkedHashMap<>();
        this.evictionPolicy = EvictionPolicyFactory.create(config.getEvictionPolicy(), maxSize);
    }

    @Override
    protected Optional<V> doGet(K key) {
        CacheEntry<V> entry = store.get(key);
        if (entry == null) {
            return Optional.empty();
        }
        if (entry.isExpired()) {
            store.remove(key);
            evictionPolicy.onRemove(key);
            stats.recordEviction();
            return Optional.empty();
        }
        entry.recordAccess();
        evictionPolicy.onAccess(key, entry);
        return Optional.of(entry.getValue());
    }

    @Override
    protected void doPut(K key, V value) {
        Instant expiresAt = defaultTtlSeconds > 0
                ? Instant.now().plusSeconds(defaultTtlSeconds)
                : null;
        doPutInternal(key, value, expiresAt);
    }

    @Override
    protected void doPutWithTtl(K key, V value, long ttlSeconds) {
        Instant expiresAt = ttlSeconds > 0
                ? Instant.now().plusSeconds(ttlSeconds)
                : null;
        doPutInternal(key, value, expiresAt);
    }

    private void doPutInternal(K key, V value, Instant expiresAt) {
        // If key already exists, update in-place
        if (store.containsKey(key)) {
            CacheEntry<V> entry = new CacheEntry<>(value, Instant.now(), expiresAt);
            store.put(key, entry);
            evictionPolicy.onPut(key, entry);
            return;
        }
        // Evict if at capacity
        if (store.size() >= maxSize) {
            K victim = evictionPolicy.selectEvictionCandidate();
            store.remove(victim);
            evictionPolicy.onRemove(victim);
            stats.recordEviction();
        }
        CacheEntry<V> entry = new CacheEntry<>(value, Instant.now(), expiresAt);
        store.put(key, entry);
        evictionPolicy.onPut(key, entry);
    }

    @Override
    protected void doDelete(K key) {
        CacheEntry<V> removed = store.remove(key);
        if (removed != null) {
            evictionPolicy.onRemove(key);
        }
    }

    @Override
    protected Map<K, V> doGetAll(Collection<K> keys) {
        Map<K, V> result = new HashMap<>();
        for (K key : keys) {
            Optional<V> value = doGet(key);
            if (value.isPresent()) {
                stats.recordHit();
                result.put(key, value.get());
            } else {
                stats.recordMiss();
            }
        }
        return result;
    }

    @Override
    protected void doPutAll(Map<K, V> entries) {
        for (Map.Entry<K, V> e : entries.entrySet()) {
            doPut(e.getKey(), e.getValue());
        }
    }

    @Override
    protected void doDeleteAll(Collection<K> keys) {
        for (K key : keys) {
            doDelete(key);
        }
    }

    @Override
    protected void doClear() {
        store.clear();
        evictionPolicy.clear();
    }

    @Override
    protected int doSize() {
        return store.size();
    }
}
```

```java
// ConsistentHashRing.java
package com.example.cache.impl;

import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.SortedMap;
import java.util.TreeMap;

/**
 * Consistent hash ring used by PartitionedCache to map keys to shard indices.
 *
 * Each shard is represented by multiple virtual nodes (replicas) to achieve
 * a more even key distribution. This avoids hotspots when shards are added
 * or removed.
 */
public class ConsistentHashRing {

    private static final int VIRTUAL_NODES_PER_SHARD = 150;
    private final SortedMap<Long, Integer> ring = new TreeMap<>();
    private final int shardCount;

    public ConsistentHashRing(int shardCount) {
        this.shardCount = shardCount;
        buildRing();
    }

    private void buildRing() {
        for (int shardIndex = 0; shardIndex < shardCount; shardIndex++) {
            for (int replica = 0; replica < VIRTUAL_NODES_PER_SHARD; replica++) {
                String virtualNodeKey = "shard-" + shardIndex + "-replica-" + replica;
                long hash = hash(virtualNodeKey);
                ring.put(hash, shardIndex);
            }
        }
    }

    /**
     * Return the shard index for the given string key.
     */
    public int getShardIndex(String key) {
        if (ring.isEmpty()) throw new IllegalStateException("Hash ring is empty");
        long hash = hash(key);
        SortedMap<Long, Integer> tailMap = ring.tailMap(hash);
        Long nodeHash = tailMap.isEmpty() ? ring.firstKey() : tailMap.firstKey();
        return ring.get(nodeHash);
    }

    private long hash(String key) {
        try {
            MessageDigest md = MessageDigest.getInstance("MD5");
            byte[] digest = md.digest(key.getBytes(StandardCharsets.UTF_8));
            long h = 0;
            for (int i = 0; i < 4; i++) {
                h = (h << 8) | (digest[i] & 0xFFL);
            }
            return h;
        } catch (NoSuchAlgorithmException e) {
            throw new RuntimeException("MD5 not available", e);
        }
    }
}
```

```java
// PartitionedCache.java
package com.example.cache.impl;

import com.example.cache.Cache;
import com.example.cache.CacheConfig;
import com.example.cache.CacheStats;

import java.util.Collection;
import java.util.HashMap;
import java.util.Map;
import java.util.Optional;
import java.util.function.Function;

/**
 * Partitioned cache that uses consistent hashing to spread keys across
 * N InMemoryCache shards.
 *
 * Each shard has maxSize / partitionCount capacity so the total capacity
 * equals the configured maxSize.
 *
 * CacheStats is the sum of all shard stats.
 */
public class PartitionedCache<K, V> implements Cache<K, V> {

    private final InMemoryCache<K, V>[] shards;
    private final ConsistentHashRing hashRing;
    private final int partitionCount;

    @SuppressWarnings("unchecked")
    public PartitionedCache(CacheConfig config) {
        this.partitionCount = config.getPartitionCount();
        this.hashRing = new ConsistentHashRing(partitionCount);

        int shardMaxSize = Math.max(1, config.getMaxSize() / partitionCount);
        this.shards = new InMemoryCache[partitionCount];

        CacheConfig shardConfig = CacheConfig.builder()
                .maxSize(shardMaxSize)
                .evictionPolicy(config.getEvictionPolicy())
                .defaultTtlSeconds(config.getDefaultTtlSeconds())
                .partitionCount(1)
                .threadSafe(config.isThreadSafe())
                .build();

        for (int i = 0; i < partitionCount; i++) {
            shards[i] = new InMemoryCache<>(shardConfig);
        }
    }

    private InMemoryCache<K, V> shardFor(K key) {
        int index = hashRing.getShardIndex(key.toString());
        return shards[index];
    }

    @Override
    public Optional<V> get(K key) {
        return shardFor(key).get(key);
    }

    @Override
    public void put(K key, V value) {
        shardFor(key).put(key, value);
    }

    @Override
    public void putWithTtl(K key, V value, long ttlSeconds) {
        shardFor(key).putWithTtl(key, value, ttlSeconds);
    }

    @Override
    public void delete(K key) {
        shardFor(key).delete(key);
    }

    @Override
    public V getOrLoad(K key, Function<K, V> loader) {
        return shardFor(key).getOrLoad(key, loader);
    }

    @Override
    public Map<K, V> getAll(Collection<K> keys) {
        Map<K, V> result = new HashMap<>();
        for (K key : keys) {
            Optional<V> value = get(key);
            value.ifPresent(v -> result.put(key, v));
        }
        return result;
    }

    @Override
    public void putAll(Map<K, V> entries) {
        for (Map.Entry<K, V> entry : entries.entrySet()) {
            put(entry.getKey(), entry.getValue());
        }
    }

    @Override
    public void deleteAll(Collection<K> keys) {
        for (K key : keys) {
            delete(key);
        }
    }

    @Override
    public CacheStats getStats() {
        CacheStats combined = new CacheStats();
        for (InMemoryCache<K, V> shard : shards) {
            CacheStats shardStats = shard.getStats();
            for (long i = 0; i < shardStats.getHitCount(); i++) combined.recordHit();
            for (long i = 0; i < shardStats.getMissCount(); i++) combined.recordMiss();
            for (long i = 0; i < shardStats.getEvictionCount(); i++) combined.recordEviction();
            for (long i = 0; i < shardStats.getPutCount(); i++) combined.recordPut();
        }
        return combined;
    }

    @Override
    public void clear() {
        for (InMemoryCache<K, V> shard : shards) {
            shard.clear();
        }
    }

    @Override
    public int size() {
        int total = 0;
        for (InMemoryCache<K, V> shard : shards) {
            total += shard.size();
        }
        return total;
    }
}
```

### Read-Through and Write-Through

```java
// CacheLoader.java
package com.example.cache;

/**
 * Loads a value from the backing data source when a cache miss occurs.
 * Implement this to integrate with a database, HTTP service, or file system.
 */
@FunctionalInterface
public interface CacheLoader<K, V> {
    /**
     * Load the value for the given key from the backing data source.
     * Return null if the value does not exist in the source.
     */
    V load(K key);
}
```

```java
// CacheWriter.java
package com.example.cache;

/**
 * Writes changes back to the backing data source as part of write-through caching.
 */
public interface CacheWriter<K, V> {
    /**
     * Persist or update the key-value pair in the backing store.
     */
    void write(K key, V value);

    /**
     * Remove the key from the backing store.
     */
    void delete(K key);
}
```

```java
// ReadThroughCache.java
package com.example.cache.decorator;

import com.example.cache.Cache;
import com.example.cache.CacheLoader;
import com.example.cache.CacheStats;

import java.util.Collection;
import java.util.HashMap;
import java.util.Map;
import java.util.Optional;
import java.util.function.Function;

/**
 * Decorator that enables read-through: on a cache miss, automatically loads
 * the value from CacheLoader, stores it, and returns it.
 *
 * Transparent to callers — they see no difference between a cache hit and
 * a transparently loaded value.
 */
public class ReadThroughCache<K, V> implements Cache<K, V> {

    private final Cache<K, V> delegate;
    private final CacheLoader<K, V> loader;

    public ReadThroughCache(Cache<K, V> delegate, CacheLoader<K, V> loader) {
        if (delegate == null) throw new IllegalArgumentException("delegate cannot be null");
        if (loader == null) throw new IllegalArgumentException("loader cannot be null");
        this.delegate = delegate;
        this.loader = loader;
    }

    @Override
    public Optional<V> get(K key) {
        Optional<V> cached = delegate.get(key);
        if (cached.isPresent()) {
            return cached;
        }
        // Cache miss: load from backing store
        V loaded = loader.load(key);
        if (loaded != null) {
            delegate.put(key, loaded);
            return Optional.of(loaded);
        }
        return Optional.empty();
    }

    @Override
    public void put(K key, V value) {
        delegate.put(key, value);
    }

    @Override
    public void putWithTtl(K key, V value, long ttlSeconds) {
        delegate.putWithTtl(key, value, ttlSeconds);
    }

    @Override
    public void delete(K key) {
        delegate.delete(key);
    }

    @Override
    public V getOrLoad(K key, Function<K, V> loaderFn) {
        // Use this cache's read-through loader, ignoring the passed function
        return get(key).orElse(null);
    }

    @Override
    public Map<K, V> getAll(Collection<K> keys) {
        Map<K, V> result = new HashMap<>();
        for (K key : keys) {
            Optional<V> value = get(key);
            value.ifPresent(v -> result.put(key, v));
        }
        return result;
    }

    @Override
    public void putAll(Map<K, V> entries) {
        delegate.putAll(entries);
    }

    @Override
    public void deleteAll(Collection<K> keys) {
        delegate.deleteAll(keys);
    }

    @Override
    public CacheStats getStats() {
        return delegate.getStats();
    }

    @Override
    public void clear() {
        delegate.clear();
    }

    @Override
    public int size() {
        return delegate.size();
    }
}
```

```java
// WriteThroughCache.java
package com.example.cache.decorator;

import com.example.cache.Cache;
import com.example.cache.CacheStats;
import com.example.cache.CacheWriter;

import java.util.Collection;
import java.util.Map;
import java.util.Optional;
import java.util.function.Function;

/**
 * Decorator that enables write-through: on every put or delete, immediately
 * write the change to the backing store via CacheWriter before (or after)
 * updating the cache.
 *
 * Strategy: write to backing store first, then update cache (ensures backing
 * store is always consistent even if the process crashes after the write).
 */
public class WriteThroughCache<K, V> implements Cache<K, V> {

    private final Cache<K, V> delegate;
    private final CacheWriter<K, V> writer;

    public WriteThroughCache(Cache<K, V> delegate, CacheWriter<K, V> writer) {
        if (delegate == null) throw new IllegalArgumentException("delegate cannot be null");
        if (writer == null) throw new IllegalArgumentException("writer cannot be null");
        this.delegate = delegate;
        this.writer = writer;
    }

    @Override
    public Optional<V> get(K key) {
        return delegate.get(key);
    }

    @Override
    public void put(K key, V value) {
        writer.write(key, value);    // Write to backing store first
        delegate.put(key, value);   // Then update cache
    }

    @Override
    public void putWithTtl(K key, V value, long ttlSeconds) {
        writer.write(key, value);
        delegate.putWithTtl(key, value, ttlSeconds);
    }

    @Override
    public void delete(K key) {
        writer.delete(key);         // Delete from backing store first
        delegate.delete(key);       // Then remove from cache
    }

    @Override
    public V getOrLoad(K key, Function<K, V> loader) {
        return delegate.getOrLoad(key, loader);
    }

    @Override
    public Map<K, V> getAll(Collection<K> keys) {
        return delegate.getAll(keys);
    }

    @Override
    public void putAll(Map<K, V> entries) {
        for (Map.Entry<K, V> entry : entries.entrySet()) {
            put(entry.getKey(), entry.getValue());
        }
    }

    @Override
    public void deleteAll(Collection<K> keys) {
        for (K key : keys) {
            delete(key);
        }
    }

    @Override
    public CacheStats getStats() {
        return delegate.getStats();
    }

    @Override
    public void clear() {
        delegate.clear();
    }

    @Override
    public int size() {
        return delegate.size();
    }
}
```

### Metrics Decorator

```java
// MetricsCollectingCache.java
package com.example.cache.decorator;

import com.example.cache.Cache;
import com.example.cache.CacheStats;

import java.util.Collection;
import java.util.Map;
import java.util.Optional;
import java.util.function.Function;

/**
 * Decorator that intercepts get and put calls to maintain an independent
 * CacheStats counter, separate from the delegate's own stats.
 *
 * This is useful when you want metrics at the decorator layer (e.g., to measure
 * read-through effectiveness without conflating with inner cache stats).
 */
public class MetricsCollectingCache<K, V> implements Cache<K, V> {

    private final Cache<K, V> delegate;
    private final CacheStats metrics = new CacheStats();

    public MetricsCollectingCache(Cache<K, V> delegate) {
        if (delegate == null) throw new IllegalArgumentException("delegate cannot be null");
        this.delegate = delegate;
    }

    @Override
    public Optional<V> get(K key) {
        Optional<V> result = delegate.get(key);
        if (result.isPresent()) {
            metrics.recordHit();
        } else {
            metrics.recordMiss();
        }
        return result;
    }

    @Override
    public void put(K key, V value) {
        delegate.put(key, value);
        metrics.recordPut();
    }

    @Override
    public void putWithTtl(K key, V value, long ttlSeconds) {
        delegate.putWithTtl(key, value, ttlSeconds);
        metrics.recordPut();
    }

    @Override
    public void delete(K key) {
        delegate.delete(key);
    }

    @Override
    public V getOrLoad(K key, Function<K, V> loader) {
        V result = delegate.getOrLoad(key, loader);
        // getOrLoad is a combined operation; count it as one get
        if (result != null) {
            metrics.recordHit();
        } else {
            metrics.recordMiss();
        }
        return result;
    }

    @Override
    public Map<K, V> getAll(Collection<K> keys) {
        Map<K, V> result = delegate.getAll(keys);
        int hits = result.size();
        int misses = keys.size() - hits;
        for (int i = 0; i < hits; i++) metrics.recordHit();
        for (int i = 0; i < misses; i++) metrics.recordMiss();
        return result;
    }

    @Override
    public void putAll(Map<K, V> entries) {
        delegate.putAll(entries);
        for (int i = 0; i < entries.size(); i++) metrics.recordPut();
    }

    @Override
    public void deleteAll(Collection<K> keys) {
        delegate.deleteAll(keys);
    }

    @Override
    public CacheStats getStats() {
        // Return the outermost metrics (this decorator's view)
        return metrics;
    }

    public CacheStats getDelegateStats() {
        return delegate.getStats();
    }

    @Override
    public void clear() {
        delegate.clear();
    }

    @Override
    public int size() {
        return delegate.size();
    }
}
```

### Cache Warming

```java
// CacheWarmer.java
package com.example.cache.warming;

import com.example.cache.Cache;

/**
 * Strategy interface for pre-populating a cache on startup.
 */
public interface CacheWarmer<K, V> {
    /**
     * Populate the given cache. Called during application startup.
     */
    void warm(Cache<K, V> cache);
}
```

```java
// DataSourceCacheWarmer.java
package com.example.cache.warming;

import com.example.cache.Cache;
import com.example.cache.CacheLoader;

import java.util.List;
import java.util.Map;

/**
 * Warms a cache from either a static Map or a CacheLoader + key list.
 *
 * Use the Map constructor for small datasets already in memory.
 * Use the CacheLoader + keys constructor to load from a database or file.
 */
public class DataSourceCacheWarmer<K, V> implements CacheWarmer<K, V> {

    private final Map<K, V> staticData;
    private final CacheLoader<K, V> loader;
    private final List<K> keysToLoad;

    /**
     * Warm from a static map.
     */
    public DataSourceCacheWarmer(Map<K, V> staticData) {
        if (staticData == null) throw new IllegalArgumentException("staticData cannot be null");
        this.staticData = staticData;
        this.loader = null;
        this.keysToLoad = null;
    }

    /**
     * Warm by loading each key in keysToLoad via the CacheLoader.
     */
    public DataSourceCacheWarmer(CacheLoader<K, V> loader, List<K> keysToLoad) {
        if (loader == null) throw new IllegalArgumentException("loader cannot be null");
        if (keysToLoad == null) throw new IllegalArgumentException("keysToLoad cannot be null");
        this.loader = loader;
        this.keysToLoad = keysToLoad;
        this.staticData = null;
    }

    @Override
    public void warm(Cache<K, V> cache) {
        if (staticData != null) {
            System.out.println("[CacheWarmer] Warming cache with " + staticData.size() + " static entries.");
            cache.putAll(staticData);
        } else if (loader != null && keysToLoad != null) {
            System.out.println("[CacheWarmer] Warming cache with " + keysToLoad.size() + " keys via loader.");
            for (K key : keysToLoad) {
                V value = loader.load(key);
                if (value != null) {
                    cache.put(key, value);
                }
            }
        }
    }
}
```

### Factory

```java
// CacheFactory.java
package com.example.cache;

import com.example.cache.decorator.MetricsCollectingCache;
import com.example.cache.impl.InMemoryCache;
import com.example.cache.impl.PartitionedCache;

/**
 * Factory that creates a fully decorated Cache from a CacheConfig.
 *
 * Wiring order (inner to outer):
 *   InMemoryCache (or PartitionedCache)
 *   -> MetricsCollectingCache
 *
 * Read-through and write-through decorators are added separately by the caller
 * because they require CacheLoader/CacheWriter instances that come from the
 * application domain.
 */
public class CacheFactory {

    private CacheFactory() {}

    /**
     * Create a cache based on config. If partitionCount > 1, creates a PartitionedCache;
     * otherwise creates a single InMemoryCache. Always wraps with MetricsCollectingCache.
     */
    public static <K, V> MetricsCollectingCache<K, V> create(CacheConfig config) {
        Cache<K, V> base;
        if (config.getPartitionCount() > 1) {
            base = new PartitionedCache<>(config);
        } else {
            base = new InMemoryCache<>(config);
        }
        return new MetricsCollectingCache<>(base);
    }

    /**
     * Convenience method that also wraps with ReadThroughCache.
     */
    public static <K, V> MetricsCollectingCache<K, V> createReadThrough(
            CacheConfig config, CacheLoader<K, V> loader) {
        Cache<K, V> base;
        if (config.getPartitionCount() > 1) {
            base = new PartitionedCache<>(config);
        } else {
            base = new InMemoryCache<>(config);
        }
        com.example.cache.decorator.ReadThroughCache<K, V> readThrough =
                new com.example.cache.decorator.ReadThroughCache<>(base, loader);
        return new MetricsCollectingCache<>(readThrough);
    }

    /**
     * Convenience method that wraps with both ReadThroughCache and WriteThroughCache.
     */
    public static <K, V> MetricsCollectingCache<K, V> createReadWriteThrough(
            CacheConfig config, CacheLoader<K, V> loader, CacheWriter<K, V> writer) {
        Cache<K, V> base;
        if (config.getPartitionCount() > 1) {
            base = new PartitionedCache<>(config);
        } else {
            base = new InMemoryCache<>(config);
        }
        com.example.cache.decorator.WriteThroughCache<K, V> writeThrough =
                new com.example.cache.decorator.WriteThroughCache<>(base, writer);
        com.example.cache.decorator.ReadThroughCache<K, V> readThrough =
                new com.example.cache.decorator.ReadThroughCache<>(writeThrough, loader);
        return new MetricsCollectingCache<>(readThrough);
    }
}
```

### Demo

```java
// CacheDemo.java
package com.example.cache.demo;

import com.example.cache.CacheConfig;
import com.example.cache.CacheFactory;
import com.example.cache.CacheLoader;
import com.example.cache.CacheStats;
import com.example.cache.EvictionPolicy;
import com.example.cache.decorator.MetricsCollectingCache;
import com.example.cache.decorator.ReadThroughCache;
import com.example.cache.warming.DataSourceCacheWarmer;

import java.util.Arrays;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.concurrent.atomic.AtomicInteger;

public class CacheDemo {

    public static void main(String[] args) throws InterruptedException {
        System.out.println("=== Distributed Cache Service — Client LLD Demo ===\n");

        demonstrateLruEviction();
        System.out.println();

        demonstrateTtlEviction();
        System.out.println();

        demonstrateMetrics();
        System.out.println();

        demonstrateReadThrough();
        System.out.println();

        demonstrateCacheWarming();
        System.out.println();

        demonstratePartitionedCache();
    }

    // ------------------------------------------------------------------
    // 1. LRU Eviction
    // ------------------------------------------------------------------
    private static void demonstrateLruEviction() {
        System.out.println("--- 1. LRU Eviction (maxSize=3) ---");

        CacheConfig config = CacheConfig.builder()
                .maxSize(3)
                .evictionPolicy(EvictionPolicy.LRU)
                .build();

        MetricsCollectingCache<String, String> cache = CacheFactory.create(config);

        cache.put("A", "Apple");
        cache.put("B", "Banana");
        cache.put("C", "Cherry");
        System.out.println("After putting A, B, C -> size=" + cache.size()); // 3

        // Access A to make it recently used; B becomes LRU
        cache.get("A");
        cache.put("D", "Durian"); // Should evict B (LRU)

        System.out.println("After accessing A and inserting D:");
        System.out.println("  A present: " + cache.get("A").isPresent()); // true
        System.out.println("  B present: " + cache.get("B").isPresent()); // false (evicted)
        System.out.println("  C present: " + cache.get("C").isPresent()); // true
        System.out.println("  D present: " + cache.get("D").isPresent()); // true
        System.out.println("  Eviction count: " + cache.getStats().getEvictionCount());
    }

    // ------------------------------------------------------------------
    // 2. TTL Eviction
    // ------------------------------------------------------------------
    private static void demonstrateTtlEviction() throws InterruptedException {
        System.out.println("--- 2. TTL Eviction ---");

        CacheConfig config = CacheConfig.builder()
                .maxSize(100)
                .evictionPolicy(EvictionPolicy.LRU)
                .defaultTtlSeconds(1) // 1 second TTL
                .build();

        MetricsCollectingCache<String, String> cache = CacheFactory.create(config);

        cache.put("short", "expires soon");
        cache.putWithTtl("long", "expires later", 60);

        System.out.println("Immediately after put:");
        System.out.println("  short present: " + cache.get("short").isPresent()); // true

        Thread.sleep(1200); // Wait for TTL to expire

        System.out.println("After 1.2 seconds:");
        System.out.println("  short present: " + cache.get("short").isPresent()); // false (expired)
        System.out.println("  long present:  " + cache.get("long").isPresent());  // true
    }

    // ------------------------------------------------------------------
    // 3. Metrics
    // ------------------------------------------------------------------
    private static void demonstrateMetrics() {
        System.out.println("--- 3. Metrics ---");

        CacheConfig config = CacheConfig.builder()
                .maxSize(5)
                .evictionPolicy(EvictionPolicy.LFU)
                .build();

        MetricsCollectingCache<String, Integer> cache = CacheFactory.create(config);

        cache.put("x", 1);
        cache.put("y", 2);
        cache.put("z", 3);

        cache.get("x"); // hit
        cache.get("x"); // hit
        cache.get("y"); // hit
        cache.get("w"); // miss (not in cache)
        cache.get("v"); // miss

        CacheStats stats = cache.getStats();
        System.out.println("Stats: " + stats);
        System.out.println("  Hit rate: " + String.format("%.1f%%", stats.getHitRate() * 100));
    }

    // ------------------------------------------------------------------
    // 4. Read-Through
    // ------------------------------------------------------------------
    private static void demonstrateReadThrough() {
        System.out.println("--- 4. Read-Through Cache ---");

        // Simulate a database
        Map<String, String> fakeDatabase = new HashMap<>();
        fakeDatabase.put("user:1", "Alice");
        fakeDatabase.put("user:2", "Bob");
        fakeDatabase.put("user:3", "Charlie");

        AtomicInteger dbCallCount = new AtomicInteger(0);
        CacheLoader<String, String> dbLoader = key -> {
            dbCallCount.incrementAndGet();
            System.out.println("  [DB] Loading key: " + key);
            return fakeDatabase.get(key);
        };

        CacheConfig config = CacheConfig.builder()
                .maxSize(100)
                .evictionPolicy(EvictionPolicy.LRU)
                .build();

        MetricsCollectingCache<String, String> cache = CacheFactory.createReadThrough(config, dbLoader);

        System.out.println("First access (cache miss, loads from DB):");
        Optional<String> user1 = cache.get("user:1");
        System.out.println("  user:1 = " + user1.orElse("null"));

        System.out.println("Second access (cache hit, no DB call):");
        Optional<String> user1Again = cache.get("user:1");
        System.out.println("  user:1 = " + user1Again.orElse("null"));

        System.out.println("Non-existent key:");
        Optional<String> missing = cache.get("user:99");
        System.out.println("  user:99 = " + missing.orElse("(not found)"));

        System.out.println("Total DB calls: " + dbCallCount.get()); // Should be 2 (user:1, user:99)
        System.out.println("Cache stats: " + cache.getStats());
    }

    // ------------------------------------------------------------------
    // 5. Cache Warming
    // ------------------------------------------------------------------
    private static void demonstrateCacheWarming() {
        System.out.println("--- 5. Cache Warming ---");

        CacheConfig config = CacheConfig.builder()
                .maxSize(100)
                .evictionPolicy(EvictionPolicy.LRU)
                .build();

        MetricsCollectingCache<String, String> cache = CacheFactory.create(config);

        Map<String, String> seedData = new HashMap<>();
        seedData.put("config:timeout", "30");
        seedData.put("config:maxRetry", "3");
        seedData.put("config:region", "us-west-2");

        DataSourceCacheWarmer<String, String> warmer = new DataSourceCacheWarmer<>(seedData);
        warmer.warm(cache);

        System.out.println("After warming, size=" + cache.size());
        System.out.println("  config:timeout = " + cache.get("config:timeout").orElse("null"));
        System.out.println("  config:region  = " + cache.get("config:region").orElse("null"));

        // Warm with loader + keys
        CacheLoader<String, String> loader = key -> "value-for-" + key;
        List<String> hotKeys = Arrays.asList("feature:darkMode", "feature:betaUI");
        DataSourceCacheWarmer<String, String> loaderWarmer =
                new DataSourceCacheWarmer<>(loader, hotKeys);
        loaderWarmer.warm(cache);

        System.out.println("  feature:darkMode = " + cache.get("feature:darkMode").orElse("null"));
    }

    // ------------------------------------------------------------------
    // 6. Partitioned Cache
    // ------------------------------------------------------------------
    private static void demonstratePartitionedCache() {
        System.out.println("--- 6. Partitioned Cache (4 partitions) ---");

        CacheConfig config = CacheConfig.builder()
                .maxSize(100)
                .evictionPolicy(EvictionPolicy.LRU)
                .partitionCount(4)
                .build();

        MetricsCollectingCache<String, String> cache = CacheFactory.create(config);

        // Put 20 entries; consistent hashing distributes across 4 shards
        for (int i = 0; i < 20; i++) {
            cache.put("key" + i, "value" + i);
        }

        System.out.println("Total entries in partitioned cache: " + cache.size());

        // Verify all 20 are retrievable
        long found = 0;
        for (int i = 0; i < 20; i++) {
            if (cache.get("key" + i).isPresent()) found++;
        }
        System.out.println("Entries retrievable after distribution: " + found + "/20");

        cache.delete("key5");
        System.out.println("After deleting key5: present=" + cache.get("key5").isPresent());
    }
}
```

---

## Unit Tests Design

### LruEvictionPolicyTest

```java
// LruEvictionPolicyTest.java (test method signatures and assertions)
package com.example.cache.eviction;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class LruEvictionPolicyTest {

    @Test
    void givenSingleEntry_whenSelectCandidate_thenReturnsThatEntry() {
        LruEvictionPolicy<String, String> policy = new LruEvictionPolicy<>(5);
        policy.onPut("a", null);
        assertEquals("a", policy.selectEvictionCandidate());
    }

    @Test
    void givenMultipleEntries_whenNoAccess_thenEvictsFirstInserted() {
        LruEvictionPolicy<String, String> policy = new LruEvictionPolicy<>(5);
        policy.onPut("first", null);
        policy.onPut("second", null);
        policy.onPut("third", null);
        // "first" should be the LRU
        assertEquals("first", policy.selectEvictionCandidate());
    }

    @Test
    void givenAccessedFirstKey_whenSelectCandidate_thenEvictsSecondKey() {
        LruEvictionPolicy<String, String> policy = new LruEvictionPolicy<>(5);
        policy.onPut("a", null);
        policy.onPut("b", null);
        policy.onAccess("a", null); // "a" becomes most recently used
        // "b" is now LRU
        assertEquals("b", policy.selectEvictionCandidate());
    }

    @Test
    void givenRemovedKey_whenSelectCandidate_thenSkipsRemovedKey() {
        LruEvictionPolicy<String, String> policy = new LruEvictionPolicy<>(5);
        policy.onPut("a", null);
        policy.onPut("b", null);
        policy.onRemove("a");
        // "b" should now be the only candidate
        assertEquals("b", policy.selectEvictionCandidate());
    }

    @Test
    void givenEmptyPolicy_whenSelectCandidate_thenThrowsException() {
        LruEvictionPolicy<String, String> policy = new LruEvictionPolicy<>(5);
        assertThrows(IllegalStateException.class, policy::selectEvictionCandidate);
    }
}
```

### LfuEvictionPolicyTest

```java
// LfuEvictionPolicyTest.java
package com.example.cache.eviction;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class LfuEvictionPolicyTest {

    @Test
    void givenEqualFrequency_whenSelectCandidate_thenEvictsEarliestInserted() {
        LfuEvictionPolicy<String, String> policy = new LfuEvictionPolicy<>();
        policy.onPut("first", null);
        policy.onPut("second", null);
        // Both have frequency 1; "first" was inserted earlier
        assertEquals("first", policy.selectEvictionCandidate());
    }

    @Test
    void givenDifferentFrequencies_whenSelectCandidate_thenEvictsLowestFrequency() {
        LfuEvictionPolicy<String, String> policy = new LfuEvictionPolicy<>();
        policy.onPut("hot", null);
        policy.onPut("cold", null);
        policy.onAccess("hot", null);
        policy.onAccess("hot", null);
        policy.onAccess("hot", null);
        // "cold" has frequency 1; "hot" has frequency 4
        assertEquals("cold", policy.selectEvictionCandidate());
    }

    @Test
    void givenSingleEntry_whenSelectCandidate_thenReturnsThatEntry() {
        LfuEvictionPolicy<String, String> policy = new LfuEvictionPolicy<>();
        policy.onPut("only", null);
        assertEquals("only", policy.selectEvictionCandidate());
    }

    @Test
    void givenRemovedKey_whenSelectCandidate_thenSkipsRemovedKey() {
        LfuEvictionPolicy<String, String> policy = new LfuEvictionPolicy<>();
        policy.onPut("a", null);
        policy.onPut("b", null);
        policy.onRemove("a");
        assertEquals("b", policy.selectEvictionCandidate());
    }

    @Test
    void givenClearedPolicy_whenPutNew_thenFrequencyResetsToOne() {
        LfuEvictionPolicy<String, String> policy = new LfuEvictionPolicy<>();
        policy.onPut("a", null);
        policy.onAccess("a", null);
        policy.onAccess("a", null);
        policy.clear();
        policy.onPut("a", null);
        policy.onPut("b", null);
        // After clear, "a" starts at 1 again; both have equal frequency
        assertEquals("a", policy.selectEvictionCandidate());
    }
}
```

### TtlEvictionPolicyTest

```java
// TtlEvictionPolicyTest.java
package com.example.cache.eviction;

import com.example.cache.CacheEntry;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import static org.junit.jupiter.api.Assertions.*;

class TtlEvictionPolicyTest {

    @Test
    void givenEntryWithEarlierExpiry_whenSelectCandidate_thenEvictsEarlierExpiry() {
        TtlEvictionPolicy<String, String> policy = new TtlEvictionPolicy<>();
        CacheEntry<String> earlyEntry = new CacheEntry<>("v1", Instant.now(),
                Instant.now().plusSeconds(10));
        CacheEntry<String> lateEntry = new CacheEntry<>("v2", Instant.now(),
                Instant.now().plusSeconds(60));
        policy.onPut("early", earlyEntry);
        policy.onPut("late", lateEntry);
        assertEquals("early", policy.selectEvictionCandidate());
    }

    @Test
    void givenEntryWithNoTtl_whenAnotherEntryHasTtl_thenEvictsTtlEntryLast() {
        TtlEvictionPolicy<String, String> policy = new TtlEvictionPolicy<>();
        CacheEntry<String> noTtlEntry = new CacheEntry<>("v1", Instant.now(), null);
        CacheEntry<String> ttlEntry = new CacheEntry<>("v2", Instant.now(),
                Instant.now().plusSeconds(5));
        policy.onPut("noTtl", noTtlEntry);
        policy.onPut("hasTtl", ttlEntry);
        assertEquals("hasTtl", policy.selectEvictionCandidate());
    }

    @Test
    void givenRemovedEntry_whenSelectCandidate_thenSkipsIt() {
        TtlEvictionPolicy<String, String> policy = new TtlEvictionPolicy<>();
        CacheEntry<String> e1 = new CacheEntry<>("v1", Instant.now(), Instant.now().plusSeconds(1));
        CacheEntry<String> e2 = new CacheEntry<>("v2", Instant.now(), Instant.now().plusSeconds(2));
        policy.onPut("a", e1);
        policy.onPut("b", e2);
        policy.onRemove("a");
        assertEquals("b", policy.selectEvictionCandidate());
    }

    @Test
    void givenEmptyPolicy_whenSelectCandidate_thenThrowsException() {
        TtlEvictionPolicy<String, String> policy = new TtlEvictionPolicy<>();
        assertThrows(IllegalStateException.class, policy::selectEvictionCandidate);
    }
}
```

### InMemoryCacheTest

```java
// InMemoryCacheTest.java
package com.example.cache.impl;

import com.example.cache.CacheConfig;
import com.example.cache.EvictionPolicy;
import org.junit.jupiter.api.Test;
import java.util.*;
import static org.junit.jupiter.api.Assertions.*;

class InMemoryCacheTest {

    private InMemoryCache<String, String> newCache(int maxSize, EvictionPolicy policy) {
        return new InMemoryCache<>(CacheConfig.builder()
                .maxSize(maxSize)
                .evictionPolicy(policy)
                .build());
    }

    @Test
    void givenPutEntry_whenGetSameKey_thenReturnsValue() {
        InMemoryCache<String, String> cache = newCache(10, EvictionPolicy.LRU);
        cache.put("key", "value");
        assertEquals("value", cache.get("key").orElse(null));
    }

    @Test
    void givenMissingKey_whenGet_thenReturnsEmpty() {
        InMemoryCache<String, String> cache = newCache(10, EvictionPolicy.LRU);
        assertFalse(cache.get("nonexistent").isPresent());
    }

    @Test
    void givenFullCache_whenNewKeyPut_thenSizeDoesNotExceedMax() {
        InMemoryCache<String, String> cache = newCache(3, EvictionPolicy.LRU);
        cache.put("a", "1");
        cache.put("b", "2");
        cache.put("c", "3");
        cache.put("d", "4"); // triggers eviction
        assertEquals(3, cache.size());
    }

    @Test
    void givenDeletedKey_whenGet_thenReturnsEmpty() {
        InMemoryCache<String, String> cache = newCache(10, EvictionPolicy.LRU);
        cache.put("k", "v");
        cache.delete("k");
        assertFalse(cache.get("k").isPresent());
    }

    @Test
    void givenPutAllMap_whenGetAll_thenAllPresentInResult() {
        InMemoryCache<String, String> cache = newCache(100, EvictionPolicy.LRU);
        Map<String, String> entries = new HashMap<>();
        entries.put("x", "1"); entries.put("y", "2"); entries.put("z", "3");
        cache.putAll(entries);
        Map<String, String> result = cache.getAll(Arrays.asList("x", "y", "z", "missing"));
        assertEquals(3, result.size());
        assertEquals("1", result.get("x"));
        assertNull(result.get("missing"));
    }

    @Test
    void givenExpiredEntry_whenGet_thenReturnsEmptyAndIncrementsEviction() throws InterruptedException {
        CacheConfig config = CacheConfig.builder()
                .maxSize(10)
                .evictionPolicy(EvictionPolicy.LRU)
                .defaultTtlSeconds(1)
                .build();
        InMemoryCache<String, String> cache = new InMemoryCache<>(config);
        cache.put("ttl", "temporary");
        Thread.sleep(1200);
        assertFalse(cache.get("ttl").isPresent());
        assertTrue(cache.getStats().getEvictionCount() > 0);
    }
}
```

### ReadThroughCacheTest

```java
// ReadThroughCacheTest.java
package com.example.cache.decorator;

import com.example.cache.CacheConfig;
import com.example.cache.EvictionPolicy;
import com.example.cache.impl.InMemoryCache;
import org.junit.jupiter.api.Test;
import java.util.concurrent.atomic.AtomicInteger;
import static org.junit.jupiter.api.Assertions.*;

class ReadThroughCacheTest {

    private ReadThroughCache<String, String> newReadThrough(AtomicInteger callCount) {
        InMemoryCache<String, String> inner = new InMemoryCache<>(
                CacheConfig.builder().maxSize(10).evictionPolicy(EvictionPolicy.LRU).build());
        return new ReadThroughCache<>(inner, key -> {
            callCount.incrementAndGet();
            return "loaded:" + key;
        });
    }

    @Test
    void givenCacheMiss_whenGet_thenLoaderIsCalledAndValueCached() {
        AtomicInteger callCount = new AtomicInteger(0);
        ReadThroughCache<String, String> cache = newReadThrough(callCount);
        String result = cache.get("key1").orElse(null);
        assertEquals("loaded:key1", result);
        assertEquals(1, callCount.get());
    }

    @Test
    void givenSubsequentGet_whenValueCached_thenLoaderNotCalledAgain() {
        AtomicInteger callCount = new AtomicInteger(0);
        ReadThroughCache<String, String> cache = newReadThrough(callCount);
        cache.get("key1"); // first get: loads and caches
        cache.get("key1"); // second get: served from cache
        assertEquals(1, callCount.get()); // loader called only once
    }

    @Test
    void givenLoaderReturnsNull_whenGet_thenReturnsEmpty() {
        InMemoryCache<String, String> inner = new InMemoryCache<>(
                CacheConfig.builder().maxSize(10).evictionPolicy(EvictionPolicy.LRU).build());
        ReadThroughCache<String, String> cache = new ReadThroughCache<>(inner, key -> null);
        assertFalse(cache.get("missing").isPresent());
    }

    @Test
    void givenPutThenGet_whenValuePresentInCache_thenLoaderNotCalled() {
        AtomicInteger callCount = new AtomicInteger(0);
        ReadThroughCache<String, String> cache = newReadThrough(callCount);
        cache.put("preloaded", "manual");
        cache.get("preloaded");
        assertEquals(0, callCount.get()); // never hit loader
    }
}
```

### MetricsCollectingCacheTest

```java
// MetricsCollectingCacheTest.java
package com.example.cache.decorator;

import com.example.cache.CacheConfig;
import com.example.cache.CacheStats;
import com.example.cache.EvictionPolicy;
import com.example.cache.impl.InMemoryCache;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class MetricsCollectingCacheTest {

    private MetricsCollectingCache<String, String> newMetricsCache() {
        InMemoryCache<String, String> inner = new InMemoryCache<>(
                CacheConfig.builder().maxSize(10).evictionPolicy(EvictionPolicy.LRU).build());
        return new MetricsCollectingCache<>(inner);
    }

    @Test
    void givenHit_whenGetStats_thenHitCountIncremented() {
        MetricsCollectingCache<String, String> cache = newMetricsCache();
        cache.put("k", "v");
        cache.get("k");
        assertEquals(1, cache.getStats().getHitCount());
        assertEquals(0, cache.getStats().getMissCount());
    }

    @Test
    void givenMiss_whenGetStats_thenMissCountIncremented() {
        MetricsCollectingCache<String, String> cache = newMetricsCache();
        cache.get("nonexistent");
        assertEquals(0, cache.getStats().getHitCount());
        assertEquals(1, cache.getStats().getMissCount());
    }

    @Test
    void givenMixedAccesses_whenComputeHitRate_thenCorrectRatio() {
        MetricsCollectingCache<String, String> cache = newMetricsCache();
        cache.put("a", "1");
        cache.put("b", "2");
        cache.get("a"); // hit
        cache.get("b"); // hit
        cache.get("c"); // miss
        cache.get("d"); // miss
        CacheStats stats = cache.getStats();
        assertEquals(2, stats.getHitCount());
        assertEquals(2, stats.getMissCount());
        assertEquals(0.5, stats.getHitRate(), 0.001);
    }

    @Test
    void givenNullDelegate_whenConstruct_thenThrowsException() {
        assertThrows(IllegalArgumentException.class,
                () -> new MetricsCollectingCache<String, String>(null));
    }
}
```

### PartitionedCacheTest

```java
// PartitionedCacheTest.java
package com.example.cache.impl;

import com.example.cache.CacheConfig;
import com.example.cache.EvictionPolicy;
import org.junit.jupiter.api.Test;
import java.util.*;
import static org.junit.jupiter.api.Assertions.*;

class PartitionedCacheTest {

    private PartitionedCache<String, String> newPartitionedCache(int maxSize, int partitions) {
        return new PartitionedCache<>(CacheConfig.builder()
                .maxSize(maxSize)
                .evictionPolicy(EvictionPolicy.LRU)
                .partitionCount(partitions)
                .build());
    }

    @Test
    void givenMultipleKeys_whenPutAndGet_thenAllRetrievable() {
        PartitionedCache<String, String> cache = newPartitionedCache(100, 4);
        for (int i = 0; i < 20; i++) cache.put("key" + i, "value" + i);
        for (int i = 0; i < 20; i++) {
            assertEquals("value" + i, cache.get("key" + i).orElse(null));
        }
    }

    @Test
    void givenDeletedKey_whenGet_thenReturnsEmpty() {
        PartitionedCache<String, String> cache = newPartitionedCache(100, 4);
        cache.put("deleteme", "v");
        cache.delete("deleteme");
        assertFalse(cache.get("deleteme").isPresent());
    }

    @Test
    void givenClear_whenSize_thenReturnsZero() {
        PartitionedCache<String, String> cache = newPartitionedCache(100, 4);
        cache.put("a", "1");
        cache.put("b", "2");
        cache.clear();
        assertEquals(0, cache.size());
    }

    @Test
    void givenGetAll_whenSomeKeysPresent_thenOnlyFoundKeysReturned() {
        PartitionedCache<String, String> cache = newPartitionedCache(100, 4);
        cache.put("found1", "v1");
        cache.put("found2", "v2");
        Map<String, String> result = cache.getAll(
                Arrays.asList("found1", "found2", "missing"));
        assertEquals(2, result.size());
        assertTrue(result.containsKey("found1"));
        assertTrue(result.containsKey("found2"));
        assertFalse(result.containsKey("missing"));
    }
}
```

---

## Performance Considerations

### Why LinkedHashMap for LRU — O(1) Access + Order

A standard HashMap gives O(1) get/put but has no concept of access order. A doubly linked list gives O(1) head/tail operations but O(n) lookup. The `LinkedHashMap` combines both: it is internally a HashMap (O(1) lookup by hash) with each bucket node also participating in a doubly linked list that tracks either insertion order or access order.

When constructed with `accessOrder=true`, every call to `get()` moves the accessed node from its current position to the tail of the linked list. The head is always the least recently used entry. This makes both the access update and the eviction candidate selection O(1) without any extra bookkeeping.

```
LinkedHashMap internal structure (access-order):
 HashMap array     Doubly-linked list (head = LRU)
 [bucket 0] ─── [A] ⟷ [C] ⟷ [B] ── head=A (LRU), tail=B (MRU)
                        after get(A):
                        [C] ⟷ [B] ⟷ [A] ── head=C (LRU), tail=A (MRU)
```

### Why Min-Heap + Frequency Map for LFU

LFU needs to find the minimum-frequency key efficiently. Options:

| Data Structure | selectEviction | onAccess | Space |
|---|---|---|---|
| Linear scan of HashMap | O(n) | O(1) | O(n) |
| Sorted TreeMap | O(1) | O(log n) | O(n) |
| Min-heap + frequency map | O(1) peek | O(log n) re-heap | O(n) |
| Doubly-linked frequency buckets | O(1) | O(1) | O(n) |

The min-heap approach is easy to implement correctly and gives O(log n) for updates with O(1) eviction candidate selection. The optimal O(1) LFU uses a doubly linked list of frequency buckets (O'Neil's algorithm), but introduces significant implementation complexity.

### Thread Safety Options

| Option | Throughput | Fairness | When to Choose |
|---|---|---|---|
| `synchronized` method/block | Low (full mutex) | JVM decides | Single-threaded or very low contention |
| `ConcurrentHashMap` | High (segment locks) | N/A | Read-heavy, many concurrent threads |
| `ReadWriteLock` | Medium-high | Configurable | Many reads, infrequent writes |
| No synchronization | Highest | N/A | Single-threaded, or immutable after init |

For a cache where reads outnumber writes 10:1, `ReadWriteLock` is the preferred default: multiple threads can read concurrently (shared lock), while a put/delete takes an exclusive write lock. `ConcurrentHashMap` is preferred when writes are also frequent because it allows concurrent writes to different segments.

The `InMemoryCache` in this implementation is intentionally not thread-safe by default (controlled by `CacheConfig.threadSafe`). For production use, wrap with `ReadWriteLock` in a `SynchronizedCache` decorator.

### Memory Overhead of CacheEntry Fields

Each `CacheEntry<V>` adds the following overhead beyond the raw value:

| Field | Size (64-bit JVM) |
|---|---|
| Object header | 16 bytes |
| `V value` (reference) | 8 bytes |
| `Instant createdAt` | 8 bytes ref + 24 bytes Instant object = 32 bytes |
| `Instant lastAccessedAt` | 32 bytes |
| `long accessCount` | 8 bytes |
| `Instant expiresAt` (nullable) | 8 bytes ref + 24 bytes = 32 bytes |
| Total per entry | ~106 bytes overhead |

For 1 million entries this is ~101 MB of metadata alone. If TTL is not used, `expiresAt` can be eliminated. If access counting is not needed, `accessCount` and `lastAccessedAt` can be removed, saving ~40 bytes per entry.

---

## Interview Questions

1. **Why is a decorator chain (MetricsCollectingCache -> ReadThroughCache -> InMemoryCache) better than a single class that does all three?**  
   Separation of concerns: each decorator is testable in isolation. You can add/remove behaviors (e.g., disable metrics in tests) without touching unrelated code. This follows the Open/Closed Principle.

2. **What happens to CacheStats when you use PartitionedCache?**  
   Each shard has its own CacheStats. The partitioned cache aggregates them by iterating shards. This means hit rate is the global ratio across all shards, which is the correct view for callers.

3. **Why use consistent hashing instead of `key.hashCode() % partitionCount`?**  
   Simple modulo hashing means that when you change the partition count, nearly every key maps to a different partition, invalidating the entire cache. Consistent hashing minimizes the number of keys that need to be remapped when the ring changes — typically only `totalKeys / newPartitionCount` keys are affected.

4. **How would you add write-behind (async write) as an alternative to write-through?**  
   Create a `WriteBehindCache` decorator that accepts a `ScheduledExecutorService` and a `CacheWriter`. On put, update the cache immediately and enqueue the write. The executor flushes the queue periodically. Callers see fast puts; the backing store is eventually consistent.
