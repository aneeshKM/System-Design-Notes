# CAP Theorem

The CAP theorem describes the trade-off a distributed system must make when a
network partition prevents nodes from communicating.

## CAP properties

### Consistency

CAP consistency means clients observe a single, up-to-date view of the data.
An operation appears to take effect atomically, and a read does not return an
older value after a newer write has completed.

See [Consistency and Availability](02-Consistency-and-Availability.md) for
weak, eventual, and strong consistency models.

### Availability

CAP availability means every request received by a non-failing node eventually
gets a response, even if the response cannot contain the latest data.

This is a request-level guarantee. It is different from operational
availability, which is measured as an uptime percentage.

### Partition tolerance

A network partition occurs when nodes or data centers cannot communicate
reliably. A partition-tolerant system continues operating according to its
chosen guarantees despite that communication failure.

This meaning of *partition* is unrelated to Kafka topic partitions, database
partitioning, or sharding.

## The trade-off

When a network partition exists, a distributed system cannot simultaneously
guarantee both CAP consistency and CAP availability. It must choose how to
behave:

- **CP (Consistency + Partition tolerance):** preserve a single current view,
  but reject, block, or delay operations that cannot be safely coordinated.
- **AP (Availability + Partition tolerance):** continue responding on both
  sides, but permit temporarily stale or conflicting data.

CAP is not a general instruction to "choose any two" during normal operation.
When there is no partition, a system may provide both consistency and
availability. The unavoidable trade-off appears while nodes cannot
communicate.

## Example

Suppose the East Coast and West Coast data centers lose communication, and a
user deletes a social-media post through the West Coast:

- In an **AP** design, the East Coast keeps serving requests and may briefly
  show the old post.
- In a **CP** design, the East Coast blocks or rejects the affected request
  until it can confirm the latest state.

The correct choice is made per operation and business requirement. A product
may favor AP behavior for a social feed while favoring CP behavior for balance
updates or uniqueness constraints.

## Revision summary

- A partition is a communication failure between nodes.
- During a partition, CP sacrifices some responses to preserve consistency.
- During a partition, AP sacrifices immediate consistency to keep responding.
- Without a partition, consistency and availability can both be provided.
- CAP availability and uptime-based operational availability are different
  concepts.
