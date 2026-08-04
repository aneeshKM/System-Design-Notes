# Cache Strategies

A cache is a temporary, high-speed storage layer that keeps reusable data
closer to the requester. Serving a cached copy is usually faster and cheaper
than repeatedly reading from a database, API, file system, or origin server.

The underlying database or origin normally remains the source of truth. A
cache improves latency and reduces backend load, but it introduces another
copy of the data whose freshness and failure behavior must be designed
explicitly.

```text
Requester → Cache ── hit ───────────────→ Return cached value
               │
               └── miss → Source of truth → Cache → Return value
```

Caching is most useful when data is expensive to obtain, requested repeatedly,
and allowed to be at least briefly stale. It is less useful for one-time reads
or data whose correctness requires every read to observe the latest committed
write.

## Cache terminology

- A **cache hit** occurs when the requested entry exists and is valid.
- A **cache miss** occurs when the cache cannot serve the request.
- **Hit rate** is the proportion of lookups served from the cache.
- **Stale data** is a cached value that no longer matches the source of truth.
- **Invalidation** marks or removes an entry when its source data changes.
- **Eviction** removes an entry, often to reclaim space; an evicted entry is not
  necessarily expired or stale.

A high hit rate alone does not prove that a cache is effective. Useful
measurements also include hit and miss latency, source-load reduction, data
freshness, eviction rate, error rate, and the latency of requests that must
refill the cache.

## Time to Live (TTL)

Time to Live is the duration for which a cached entry is considered valid. A
cache can remove an expired entry immediately, discover it during the next
lookup, or clean it up later in the background. Logical expiration and
physical deletion therefore do not always happen at the same moment.

- A short TTL improves freshness but creates more cache misses, revalidations,
  and source traffic.
- A long TTL improves hit rate and reduces source load but increases the chance
  and duration of stale results.
- An explicit invalidation event can remove an entry before its TTL expires.
- A stale-while-revalidate policy may deliberately serve an expired value for
  a bounded period while a fresh value is loaded in the background.

For example, a weather API response might be cached for five minutes. Requests
during that interval reuse the cached response. After expiration, the next
request either fetches updated data or receives a bounded stale response while
another process refreshes the entry, depending on policy.

TTL should follow the data's acceptable staleness rather than use one default
for every key. Adding small random variation, or **jitter**, to expiration
times prevents many related entries from expiring simultaneously and sending
a sudden traffic spike to the source.

## Refresh-ahead caching

Refresh-ahead caching proactively reloads selected entries shortly before
their TTL expires. It is usually applied to frequently accessed or especially
important data rather than every cached value.

```text
Cached entry approaches expiration
              ↓
Background worker fetches a fresh value
              ↓
Cache entry and expiration are replaced
```

Users can continue reading from the cache while the refresh is in progress.
This reduces cache-miss latency and avoids a burst of source requests when a
popular entry expires. However, refreshing data that is never requested again
wastes network, compute, and storage resources. The cached copy can also
remain stale between refreshes.

A news application might refresh its cached homepage headlines every few
minutes before expiration. Requests continue using the cached homepage while
the background refresh obtains the next version.

Refresh-ahead needs a failure policy. If a refresh fails, the system must
decide whether to retry, serve stale data for a bounded interval, fall back to
the source on the next request, or return an error.

## Write-behind caching

Write-behind, also called write-back, updates the cache or an associated write
buffer first and persists the change to the primary database asynchronously.

```text
Client → Cache / durable buffer → Acknowledge
                 │
                 └── asynchronous worker → Database
```

Because the client does not wait for database persistence, write latency can
be low and many updates can be batched. Reads from the cache can immediately
observe the new value.

The trade-off is that the cache and database are temporarily inconsistent. If
the cache or worker fails before persistence, an acknowledged update can be
lost. A production design therefore needs to address:

- Durable buffering or a persistent write-ahead log.
- Replication of pending writes.
- Retry behavior and idempotent database operations.
- Ordering between multiple updates to the same key.
- Duplicate delivery and failure recovery.
- Backpressure when the database cannot keep up.

An analytics system might increment view counts in Redis and periodically
write aggregated counts to a database. This supports a high write rate, but
recent increments are at risk if the pending updates are not stored durably.

Write-behind is a poor fit when the system must guarantee that every
acknowledged write has already reached the primary database.

## Write-through caching

Write-through synchronously persists a write to the primary database as part
of the cache write path. The operation is not acknowledged as successful until
the required database update succeeds.

```text
Client → Cache layer → Database
             │            │
             └── update cached value after successful persistence
                          ↓
                    Acknowledge write
```

The exact internal order can vary by implementation. The important guarantee
is that success is not returned before the database write succeeds and the
cache is left with the intended value. This lowers the risk of losing an
acknowledged update and makes future reads likely to hit a fresh cached value.

Write-through adds database latency to every write, even when the newly cached
data is never read. The cache layer also becomes part of the write path and
must define what happens when either the cache update or database update
fails.

A banking application could update account preferences through a
write-through cache. It reports success only after the preferences have been
persisted, then later reads can use the cached copy.

Write-through describes a **write path**. It does not imply read-through
behavior. In read-through caching, the cache provider itself loads a missing
value from the source; in cache-aside, the application performs that work.

## Cache-aside caching

Cache-aside, or lazy loading, makes the application responsible for cache
reads, miss handling, population, and invalidation.

### Read flow

1. The application looks up the key in the cache.
2. On a hit, it returns the cached value.
3. On a miss, it queries the database.
4. It stores the result in the cache with an appropriate TTL.
5. It returns the result.

```text
Application → Cache
     │          │
     │          ├── hit → Return value
     │          │
     │          └── miss
     ↓
Database → Populate cache → Return value
```

### Write flow

A common write flow is:

1. Update the database.
2. Delete or invalidate the corresponding cache entry.
3. Let the next read load the new database value into the cache.

Deleting the old entry is often safer than trying to update both copies
because the database remains authoritative. The order matters: invalidating
first and then failing to update the database creates an unnecessary miss,
while updating the database first avoids exposing an uncommitted value through
the cache.

The pattern still has concurrency races. For example, a reader can fetch an
old database value, a writer can then commit and invalidate the cache, and the
reader can finally repopulate the cache with the old value. Versioned values,
ordered change events, conditional cache writes, and bounded TTLs are possible
mitigations; the right choice depends on the required consistency.

### Benefits and limitations

Cache-aside only stores data that applications actually request, works with
ordinary cache and database products, and lets the application control error
handling. Its costs include:

- Higher latency on the first request for an uncached or expired key.
- Application code for population, expiration, and invalidation.
- Stale data when invalidation is delayed, lost, or races with a refill.
- A possible cache stampede when many requests miss the same popular key.

A product service can check Redis for product information, read a missing
product from PostgreSQL, cache it, and return it. When the product changes, the
service commits the PostgreSQL update and removes the old Redis entry.

## Strategy comparison

| Strategy | Read-miss owner | Write acknowledgement | Main benefit | Main risk or cost |
| --- | --- | --- | --- | --- |
| Cache-aside | Application | Normally follows the database write | Simple, demand-driven population | Miss latency and invalidation races |
| Read-through | Cache provider | Not defined by this read strategy | Centralized miss handling | Provider complexity and source coupling |
| Write-through | Cache layer and database synchronously | After required persistence succeeds | Fresh cached writes and lower acknowledged-data-loss risk | Higher write latency |
| Write-behind | Background worker | Before database persistence | Low write latency and batching | Temporary inconsistency and possible data loss |
| Refresh-ahead | Background refresher | Not a write-path strategy | Fewer misses for hot entries | Unnecessary refresh work |

Read and write strategies can be combined. For example, a cache can use
read-through on misses and write-through on updates. Cache-aside can use TTL,
explicit invalidation, and refresh-ahead for different classes of data.

## Caching locations and use cases

The best cache location depends on where requests originate and where reusable
work occurs. One request may pass through several independent caches.

```text
Browser → CDN → Web / reverse-proxy cache → Application cache → Database
                                                               internal cache
```

### Client-side cache

A browser, mobile device, or desktop application can store images, JavaScript,
CSS, API responses, and application data. This eliminates network requests and
often provides the lowest user-perceived latency, but the server has limited
control over already-cached copies and must avoid storing sensitive shared data
incorrectly.

### CDN cache

A CDN stores content on geographically distributed edge servers and serves it
near users. It commonly caches images, video segments, scripts, stylesheets,
downloads, and cacheable HTTP responses. CDN behavior, push and pull delivery,
and HTTP-oriented TTL trade-offs are covered in
[DNS and Content Delivery Networks](05-DNS-and-CDN.md#content-delivery-network-cdn).

### Web-server or reverse-proxy cache

A web server or reverse proxy such as Nginx or Varnish can cache complete
pages, rendered fragments, static files, and API responses. A shared response
can serve many users, so cache keys must include every request property that
materially changes the response, such as tenant, language, or authorization
scope.

### Application cache

An application can use in-process memory or a shared system such as Redis or
Memcached for sessions, product data, permissions, configuration, rate-limit
counters, or expensive computation results.

An in-process cache has very low latency but each application instance can
hold a different value. A shared distributed cache gives instances a common
view but adds a network hop and another service to operate.

### Database cache

Databases internally cache data pages, index pages, query plans, and other
structures in memory to reduce repeated disk access. The exact contents and
replacement behavior are database-specific. An external Redis or Memcached
deployment in front of a database is normally an application or distributed
cache, not the database's internal cache.

For one e-commerce product-page request, the browser might reuse cached
images, a CDN might serve static files, a reverse proxy might return cached
HTML, the application might retrieve product details from Redis, and the
database might use its internal buffer cache for any remaining query.

## Common failure modes

- A **cache stampede** occurs when many requests miss the same popular key and
  all query the source. Per-key request coalescing, refresh-ahead, stale serving,
  and TTL jitter can reduce it.
- A **hot key** sends disproportionate traffic to one cache entry or shard.
  Replication, local caching, or request coalescing may distribute the work.
- **Cache penetration** repeatedly requests values that do not exist. Briefly
  caching negative results or using a membership filter can protect the
  database, provided authorization and freshness remain correct.
- A **cache outage** can redirect the cache's full traffic to the source. The
  source needs protection through timeouts, concurrency limits, load shedding,
  or controlled bypass behavior.
- **Unbounded keys** or oversized values can exhaust memory and cause heavy
  eviction. Capacity limits, admission rules, and size monitoring are needed.

## Design checklist

- Identify the expensive read, computation, or write path that caching is
  intended to improve.
- Define the source of truth and acceptable staleness for each data class.
- Choose a cache key that includes every dimension that changes the result and
  never leaks data across users or tenants.
- Select TTL, invalidation, and refresh behavior from freshness requirements.
- Decide how misses, partial failures, retries, and cache outages behave.
- Prevent concurrent misses from overwhelming the source.
- For write-behind, make pending acknowledged writes durable and idempotent.
- Measure hit rate, latency, freshness, evictions, and source-load reduction.
- Verify that the system remains safe when the cache is empty or unavailable.
