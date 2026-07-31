# Database Scaling and Performance

Database scalability is not one technique. Replication creates additional
copies, sharding divides a dataset, and federation separates business domains.
Denormalization and indexes instead change how efficiently particular queries
can retrieve data. Each technique solves a different bottleneck and introduces
its own correctness and operational costs.

## Database replication

Replication maintains copies of the same records on multiple database nodes.
It can improve fault tolerance, geographic access, and read capacity.

In primary-replica replication:

1. Clients send writes to the primary.
2. The primary records and propagates changes to one or more replicas.
3. Replicas may serve reads, subject to the required freshness guarantee.
4. If the primary fails, an eligible replica can be promoted and traffic must
   be redirected to it.

```text
                         ┌──→ Replica A → reads
Writes → Primary ────────┤
                         └──→ Replica B → reads / failover candidate
```

Asynchronous replicas may return stale data. Promotion can also lose recently
acknowledged writes if they had not reached the chosen replica. Applications
that require read-after-write behavior may route those reads to the primary or
use a database-specific consistency mechanism.

In multi-primary replication, several nodes accept writes. This can improve
write locality or availability, but concurrent changes may conflict and the
system needs explicit conflict prevention or resolution.

### Synchronous versus asynchronous replication

| Property | Synchronous | Asynchronous |
| --- | --- | --- |
| Write acknowledgement | Waits for the configured replica acknowledgement | Can return before replicas receive the change |
| Write latency | Higher because replication is on the write path | Usually lower |
| Replica lag | Reduced for participating replicas | Expected |
| Failure concern | An unavailable replica can delay or block writes, depending on policy | Recent writes may be absent from a failover replica |

"Synchronous" does not necessarily mean every replica acknowledges every
write. The database may wait for one standby, a quorum, or another configured
threshold. The exact acknowledgement rule determines the latency and data-loss
trade-off.

Replication is not a backup. Accidental deletes, corrupt updates, and some
operator errors can be copied to every replica. Backups provide an independent
recovery history. Replication and failover are discussed further in
[Consistency and Availability](02-Consistency-and-Availability.md#replication).

## Database sharding

Sharding horizontally partitions one logical dataset and stores different
subsets on different database servers. It scales storage and can distribute
read and write traffic beyond one node's capacity.

A **shard key** determines the shard that owns a record. For hash-based
sharding, routing can be illustrated as:

```text
shard_number = hash(customer_id) mod number_of_shards
```

An order system using this rule can route a request containing `customer_id`
directly to the owning shard. A query without the shard key may require a
scatter-gather operation across all shards.

Good shard keys should:

- Have enough distinct values to distribute data and traffic.
- Avoid monotonically increasing or otherwise hot values when those values
  concentrate new writes.
- Appear in the application's important queries.
- Keep data that is frequently accessed together on the same shard where
  practical.
- Remain stable, because moving a record between shards is costly.

Hash sharding usually distributes load well but makes range routing difficult.
Range sharding makes adjacent values easy to scan but can create hotspots.
Directory-based sharding stores an explicit key-to-shard mapping, which offers
control at the cost of another routing component.

Sharding introduces operational complexity:

- Cross-shard joins, aggregations, and transactions require coordination.
- Global unique constraints and secondary indexes are harder to maintain.
- Adding shards requires data rebalancing and careful routing changes.
- Uneven data or traffic can create hot shards.
- Every shard still needs replication, backup, monitoring, and failover.

## Database federation

Database federation, or functional partitioning, separates data stores by
business domain. Unlike sharding, it does not divide rows from the same logical
dataset according to a shard key.

```text
Customer database → customers, addresses, preferences
Order database    → orders, order_items, shipments
Payment database  → payments, refunds, payment_attempts
```

Federation can give domains independent schemas, scaling policies, deployment
cycles, and even database technologies. Failures and heavy workloads can also
be isolated by function.

The cost is that cross-database joins, referential integrity, reporting, and
atomic transactions become more difficult. The split should follow cohesive
business ownership rather than creating a separate database for every table.
In a microservices system, federation often appears as the
[Database per Service pattern](07-Microservice-Architecture.md#database-per-service-pattern).

### Replication, sharding, and federation

| Technique | Data placement | Primary goal | Main complication |
| --- | --- | --- | --- |
| Replication | Copies of the same data | Availability and read capacity | Lag, promotion, and write conflicts |
| Sharding | Different rows or partitions of one dataset | Storage and throughput | Routing, rebalancing, and cross-shard operations |
| Federation | Different business domains | Ownership and independent scaling | Cross-domain queries and transactions |

These techniques can be combined. A federated Order database can be sharded by
customer, and each shard can have replicas for availability.

## Denormalization

Denormalization intentionally duplicates, embeds, or precomputes selected data
to reduce joins and improve read performance. It exchanges storage and update
complexity for a read path tailored to a known workload.

A normalized order query might join `orders`, `customers`, `order_items`, and
`products`. A denormalized design can preserve names and prices as they were at
purchase time:

```text
orders(
  order_id,
  customer_id,
  customer_name_at_purchase,
  total
)

order_items(
  order_id,
  product_id,
  product_name_at_purchase,
  unit_price_at_purchase
)
```

Here the copied fields are also a historical snapshot: a later customer rename
or price change should not rewrite the original invoice. Other duplicated
fields may need synchronous updates, asynchronous events, or periodic repair
jobs to remain consistent.

A document database can store a similar read model in one order document:

```json
{
  "_id": 5001,
  "customerSnapshot": {
    "customerId": 101,
    "name": "Alice"
  },
  "items": [
    {
      "productId": 901,
      "productName": "Laptop",
      "unitPrice": 1000
    }
  ]
}
```

Denormalize only for measured access patterns. Before duplicating data, define
which copy is authoritative, how derived copies are updated, how stale they may
be, and how inconsistencies will be detected and repaired.

## Database indexing

An index is an additional data structure that helps the database locate rows
or documents without scanning the complete dataset. Indexes are useful for
fields frequently used in filters, joins, sorting, and uniqueness constraints.

They are not free:

- Each index consumes storage and memory.
- Inserts, deletes, and indexed-field updates must also update the index.
- Low-selectivity indexes may not reduce enough work to be useful.
- Unused or redundant indexes add write cost without improving reads.

### SQL indexes

```sql
CREATE UNIQUE INDEX idx_customers_email
ON customers(email);

CREATE INDEX idx_orders_customer_created
ON orders(customer_id, created_at DESC);
```

The first index both accelerates email lookup and enforces uniqueness. The
second supports finding a customer's orders and returning the newest ones
first.

Column order matters in a composite index. The `(customer_id, created_at)`
index naturally supports a filter on `customer_id` and an ordered scan over
that customer's `created_at` values. It is generally not equivalent to an
index with the columns reversed.

A primary-key constraint normally creates a unique index automatically. An
ordinary index improves an access path but does not necessarily enforce
uniqueness.

### MongoDB indexes

```javascript
db.customers.createIndex(
  { email: 1 },
  { unique: true }
)

db.orders.createIndex({
  customerId: 1,
  createdAt: -1
})
```

The same high-level trade-off applies: the index can accelerate the matching
query shape, but every relevant write must maintain it.

## SQL query tuning

Query tuning starts with measurement. Inspect the actual execution plan and
runtime behavior before rewriting a query or adding an index.

A practical process is:

1. Identify an important slow query using latency, throughput, and database
   workload measurements.
2. Run `EXPLAIN`; where safe and supported, run `EXPLAIN ANALYZE` to obtain
   actual row counts and timings.
3. Compare estimated and actual rows, scans, join algorithms, sorts, temporary
   data, and I/O.
4. Improve the query, index, schema, or access pattern that causes the measured
   cost.
5. Rerun the plan and workload to verify the improvement and check write-side
   impact.

Common improvements include:

- Indexing important filters, joins, and orderings.
- Retrieving required columns instead of using `SELECT *`.
- Filtering rows before expensive downstream operations.
- Removing unnecessary joins and repeated calculations.
- Using cursor-based pagination for large, frequently changing result sets
  when offset pagination becomes expensive.
- Introducing caches, summary tables, or materialized views for suitable
  repeated reads.
- Keeping optimizer statistics current so row-count estimates remain useful.

For example:

```sql
EXPLAIN ANALYZE
SELECT
  o.order_id,
  o.status,
  o.total,
  o.created_at
FROM customers AS c
JOIN orders AS o
  ON o.customer_id = c.customer_id
WHERE c.email = 'alice@example.com'
ORDER BY o.created_at DESC
LIMIT 20;
```

The indexes on `customers(email)` and
`orders(customer_id, created_at DESC)` shown earlier match the lookup, join,
and ordering pattern. Whether the optimizer uses them still depends on table
size, selectivity, statistics, and database-specific behavior.

## Join order and early filtering

Joins can become expensive when the database processes large intermediate
results. A selective predicate that reduces the input early can lower CPU,
memory, and I/O work.

Consider a query for one customer's purchased products:

```sql
SELECT
  o.order_id,
  p.product_name,
  oi.quantity
FROM order_items AS oi
JOIN products AS p
  ON p.product_id = oi.product_id
JOIN orders AS o
  ON o.order_id = oi.order_id
WHERE o.customer_id = 101;
```

The desired plan finds the small set of matching orders before reading their
order items. A filter-first formulation makes that intent explicit:

```sql
SELECT
  filtered_orders.order_id,
  p.product_name,
  oi.quantity
FROM (
  SELECT order_id
  FROM orders
  WHERE customer_id = 101
) AS filtered_orders
JOIN order_items AS oi
  ON oi.order_id = filtered_orders.order_id
JOIN products AS p
  ON p.product_id = oi.product_id;
```

Useful supporting indexes are:

```sql
CREATE INDEX idx_orders_customer
ON orders(customer_id, order_id);

CREATE INDEX idx_order_items_order
ON order_items(order_id);
```

Modern optimizers often reorder inner joins, push predicates down, and flatten
the derived table, so these two written queries may produce the same plan. The
rewrite by itself is not proof of an optimization. `EXPLAIN ANALYZE` must show
that the database starts from the selective customer orders and avoids an
unnecessary large intermediate result.

## Design checklist

- Identify whether the current limit is read traffic, write traffic, storage,
  query shape, or availability before selecting a technique.
- Replicate for copies, shard for partitions, and federate for domain
  boundaries.
- Choose shard and index keys from real access patterns and measured data
  distribution.
- Preserve a clear source of truth when creating denormalized read models.
- Treat failover, rebalancing, backup restoration, and consistency repair as
  workflows that need testing.
- Validate performance changes with execution plans and representative load,
  including their effect on writes and operations.
