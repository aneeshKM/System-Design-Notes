# Consistency and Availability

## Consistency

Consistency describes the rules governing which version of replicated data a
read may observe. In the strongest model discussed here, every read after a
successful write returns that write or a newer value, so clients do not
observe stale data.

For example, a bank balance changes from $2,000 to $1,500 after a $500
transfer. If every ATM, mobile app, and bank service subsequently shows
$1,500, clients see a consistent view. An ATM that still shows $2,000 is
temporarily stale.

The word *consistency* has several precise meanings in distributed systems.
Always name the model being promised rather than saying only "the system is
consistent."

## Consistency models

### Weak consistency

Weak consistency does not guarantee that a read immediately following a write
returns the newest value.

- A write may not be immediately visible to all clients.
- Different replicas may temporarily return different values.
- Reads may return stale data.
- Weak consistency by itself does not promise when replicas will converge.

**Example:** An e-commerce platform changes a product price from $100 to $120.
One user may see $120 while another user reading from a different replica
still sees $100.

### Eventual consistency

Eventual consistency is a weak-consistency model in which replicas may
temporarily disagree but will converge if no new writes occur.

- Updates are commonly propagated asynchronously.
- Reads may temporarily return stale values.
- Replicas eventually converge when updates stop.
- Avoiding coordination can improve availability and write latency, at the
  cost of temporarily stale reads.

**Example:** After a user changes a profile picture, some replicas may continue
serving the old image. Once replication completes, every replica serves the
new image.

> Weak consistency does not necessarily promise convergence. Eventual
> consistency does.

### Strong consistency

In these notes, strong consistency means that after a write completes
successfully, subsequent reads observe that completed write or a newer one.
The system appears to clients as though it has one up-to-date copy of the
data.

- Acknowledged writes cannot be followed by stale reads.
- Writes may require coordination with a leader or replica quorum.
- Coordination can increase latency.
- During a partition, preserving this guarantee may require rejecting or
  delaying some operations.

**Example:** After a successful $200 withdrawal from a $1,000 bank balance,
subsequent balance reads must return $800. Returning $1,000 would violate the
guarantee.

Strong consistency does not require every physical replica to update at the
same instant. It requires the system to prevent clients from observing an
invalid stale view after acknowledging the write.

| Model | Can a read be stale? | Convergence guaranteed? | Typical trade-off |
| --- | --- | --- | --- |
| Weak | Yes | Not necessarily | Less coordination |
| Eventual | Temporarily | Yes, if writes stop | High availability and lower write latency |
| Strong | Not after an acknowledged write | Clients see one current view | More coordination |

## Operational availability

Availability is the proportion of time a system is operational and accessible
during a specified period.

```text
Availability (%) = Uptime / (Uptime + Downtime) × 100

Allowed downtime = Total time × (1 - Availability)
```

Replication, redundancy, health checks, and failover can improve availability.
They do not remove the need to define what counts as "available"—for example,
whether a degraded or stale response satisfies the service objective.

> Operational availability is an observed or targeted uptime ratio. It is
> different from the request-level definition of availability used by the
> [CAP theorem](03-CAP-Theorem.md#availability).

## Availability nines

Availability targets are often described by the number of consecutive nines.
More nines permit less downtime but usually require more redundancy,
automation, operational maturity, and cost.

| Availability | Name | Approximate annual downtime |
| ---: | --- | ---: |
| 99% | Two nines | 3.65 days |
| 99.9% | Three nines | 8.76 hours |
| 99.99% | Four nines | 52.56 minutes |
| 99.999% | Five nines | 5.26 minutes |

### Worked example

For 99.99% annual availability:

```text
Minutes/year = 365 × 24 × 60 = 525,600
Downtime = 525,600 × (1 - 0.9999)
         = 52.56 minutes/year
```

## Consistency versus availability

Consistency describes which data version a read may observe. Operational
availability describes how often the service is usable. They are different
properties and must be measured separately.

A system can be highly available while occasionally serving stale data, or it
can preserve strong consistency while becoming unavailable during failures.
The precise trade-off during a network partition is covered in
[CAP Theorem](03-CAP-Theorem.md).

## Replication

Replication maintains copies of the same data on multiple servers to improve
availability and fault tolerance.

- Replicas reduce dependence on a single database server.
- Replication may be synchronous or asynchronous.
- Synchronous replication generally offers fresher failover data but adds
  write latency and coordination.
- Asynchronous replication generally lowers write latency but introduces
  replica lag and possible data loss during failover.
- Replication alone does not perform recovery; health detection, leader
  election or promotion, and traffic redirection are also needed.

### Primary-primary replication

Multiple primary nodes accept reads and writes and replicate changes to one
another.

**Advantages**

- Reads and writes can remain available when a node fails.
- Nodes in multiple regions can serve nearby users.
- All primary nodes can contribute capacity.

**Disadvantages**

- Concurrent updates can conflict.
- Conflict detection and deterministic resolution are required.
- Preventing split-brain and operating the system are more complex.

**Example:** US and European nodes both accept local writes. If both update the
same record, the system must reconcile those changes.

### Primary-replica replication

One primary accepts writes, while replicas copy its changes and typically
serve reads. A replica may be promoted if the primary fails.

**Advantages**

- A single writer simplifies ordering and conflict management.
- Read traffic can be distributed across replicas.
- It is generally simpler to operate than primary-primary replication.

**Disadvantages**

- The primary may become the write bottleneck.
- Replica lag may cause stale reads.
- Promotion and traffic redirection can cause temporary downtime.

**Example:** Product updates go to an e-commerce database's primary, while
search and product-detail requests use read replicas.

| Property | Primary-primary | Primary-replica |
| --- | --- | --- |
| Write nodes | Multiple | One at a time |
| Read scaling | Yes | Yes |
| Write scaling | Potentially | Limited by primary |
| Conflict risk | Higher | Lower |
| Common concern | Conflicts and split-brain | Replica lag and promotion |

## Failover

Failover transfers traffic or responsibility from a failed component to a
healthy replacement.

A complete failover path normally includes:

1. Health checks or heartbeats detect the failure.
2. The system selects or promotes a healthy replacement.
3. Routing or discovery sends traffic to the replacement.
4. The replacement has sufficiently current data and configuration.

Failover may be automatic or manual. Its duration contributes directly to
downtime and therefore affects recovery-time objectives.

### Active-active

Multiple healthy components serve traffic simultaneously. If one fails, the
others absorb its traffic.

**Advantages**

- Good resource utilization
- Minimal interruption after a failure
- Natural fit for horizontal scaling

**Disadvantages**

- Remaining nodes need enough spare capacity for failover traffic.
- Synchronizing state can be difficult.
- Stateful systems can experience conflicts or split-brain.

Stateless services behind a load balancer are a common active-active design.

### Active-passive

One component serves traffic while another remains on standby and takes over
after a failure.

**Advantages**

- Simpler state and consistency management
- Lower risk of concurrent conflicting writes
- Often easier for stateful databases

**Disadvantages**

- Standby capacity is underused during normal operation.
- Promotion introduces some interruption.
- Asynchronous standby data may lag behind the active node.

A primary database with a continuously updated standby is a common
active-passive design.

## Availability of dependencies and replicas

The following formulas assume component failures are independent. Correlated
failures—such as a shared network, region, power source, or bad deployment—can
make real availability substantially lower.

### Sequential dependencies

Components are sequential when every component must work for the operation to
succeed:

```text
A_system = A₁ × A₂ × ... × Aₙ
```

Adding mandatory dependencies lowers total availability.

**Example:** Checkout requires the order service, payment service, inventory
service, and database. If any required dependency fails, checkout fails.

For two independent components that are each 99.99% available:

```text
A_system = 0.9999 × 0.9999
         = 0.99980001
         = 99.980001% ≈ 99.98%
```

### Parallel redundancy

Components are parallel when any one of the redundant components can provide
the operation:

```text
Two components: A_system = 1 - (1 - A₁)(1 - A₂)

n identical components: A_system = 1 - (1 - A)ⁿ
```

The system fails only if every redundant component fails.

For two independent servers that are each 99.99% available:

```text
A_system = 1 - (1 - 0.9999)(1 - 0.9999)
         = 1 - (0.0001 × 0.0001)
         = 0.99999999
         = 99.999999%
```

This result assumes both servers can actually serve the operation and that
load balancing or failover logic is itself available.

## Choosing an availability pattern

Choose an availability pattern based on state, consistency requirements,
traffic, recovery objectives, failure domains, and cost.

- Stateless application servers commonly use active-active deployment.
- Stateful databases often use active-passive or primary-replica deployment.
- Primary-primary replication fits cases that require writes in multiple
  locations and can tolerate or resolve conflicts.
- Primary-replica replication fits cases that prioritize simpler write
  ordering and read scaling.
- Capacity planning must account for traffic after a component fails, not only
  normal traffic.

A typical e-commerce design may combine active-active web servers behind a
load balancer with an active-passive primary-replica database. The web tier
scales and redistributes traffic quickly, while the database avoids concurrent
writer conflicts.

## Revision summary

- Weak consistency permits stale reads without a convergence deadline.
- Eventual consistency permits stale reads but promises convergence if writes
  stop.
- Strong consistency prevents stale reads after acknowledged writes.
- Availability measures how often the defined service remains usable.
- Replication provides redundant data copies; failover moves work to a healthy
  copy.
- Sequential dependencies reduce total availability, while independent
  parallel redundancy increases it.
