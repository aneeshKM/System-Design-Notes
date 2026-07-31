# Database Models and Properties

## Relational and NoSQL databases

Relational databases store structured data in tables whose columns,
relationships, and constraints are described by a schema. SQL databases are a
strong fit when the system needs joins, referential integrity, flexible
queries, and transactions spanning related records.

NoSQL is an umbrella term for non-relational models such as key-value,
document, wide-column, and graph databases. These models are commonly chosen
for flexible records, specialized access patterns, high write volume, or
horizontal distribution. NoSQL does not mean that SQL-like queries,
transactions, or schemas are always unavailable; the exact guarantees depend
on the database.

| Property | Relational | NoSQL |
| --- | --- | --- |
| Primary model | Tables, rows, and columns | Key-value, document, wide-column, or graph |
| Schema | Usually declared and enforced by the database | Often flexible, although applications still need data-shape rules |
| Relationships | Foreign keys and joins | Embedding, application-side references, or graph edges |
| Typical strength | Constraints, transactions, and ad hoc queries | Model-specific access patterns and distributed scale |
| Examples | PostgreSQL, MySQL, Oracle, SQL Server | Redis, DynamoDB, MongoDB, Cassandra, Neo4j |

The choice is not simply SQL *or* NoSQL for an entire system. An e-commerce
platform might use PostgreSQL for orders and payments, MongoDB for a varied
product catalog, Redis for short-lived cart data, Cassandra for activity
events, and a graph database for relationship-heavy recommendations. This use
of different data stores for different workloads is often called **polyglot
persistence**.

## ACID properties

ACID describes properties that make database transactions reliable during
failures and concurrent operations:

- **Atomicity:** Every operation in a transaction commits, or none of them
  does. The database does not expose a partially completed transaction.
- **Consistency:** A transaction takes the database from one valid state to
  another while preserving declared constraints and application invariants.
- **Isolation:** Concurrent transactions interact according to a defined
  isolation level, which controls anomalies such as dirty or non-repeatable
  reads.
- **Durability:** Once a commit succeeds, its effects survive a process or
  machine crash within the database's stated failure model.

For a bank transfer, debiting one account and crediting another belong in one
transaction. If the credit cannot be performed, the debit must not remain
committed.

> ACID consistency means preserving rules and invariants inside a transaction.
> CAP consistency concerns what clients can observe across distributed copies
> of data. The meanings are related to correctness but are not interchangeable;
> see [CAP Theorem](03-CAP-Theorem.md).

ACID is not exclusive to relational systems, and selecting a relational
database does not eliminate the need to choose appropriate constraints,
transaction boundaries, and isolation levels.

## Database normalization

Normalization organizes relational data so that each fact has an appropriate
home. It reduces unnecessary duplication and helps prevent:

- **Update anomalies:** The same fact must be changed in several rows.
- **Insertion anomalies:** A fact cannot be recorded without an unrelated
  record.
- **Deletion anomalies:** Deleting one record accidentally removes the only
  copy of another fact.

The first three normal forms are the most commonly discussed:

1. **First Normal Form (1NF):** Each column contains one atomic value and there
   are no repeating column groups.
2. **Second Normal Form (2NF):** The table is in 1NF and every non-key attribute
   depends on the whole candidate key, not only part of a composite key.
3. **Third Normal Form (3NF):** The table is in 2NF and non-key attributes do
   not depend transitively on a key through another non-key attribute.

BCNF, 4NF, and 5NF handle more specialized dependency cases.

Suppose every student row repeats the name and phone number of the student's
department. Updating a department phone number could then require changing
many rows. A normalized design stores the department fact once:

```text
Student(student_id, name, department_id)
Department(department_id, department_name, phone)
```

`Student.department_id` references the corresponding department. This avoids
duplicating department details while still allowing them to be retrieved with
a join. Normalization is a logical starting point, not a rule that every
production read must perform many joins; deliberate denormalization is covered
in [Database Scaling and Performance](09-Database-Scaling-and-Performance.md#denormalization).

## Key-value databases

A key-value database stores a value under a unique key and retrieves it
primarily through direct key lookup. It behaves like a distributed dictionary.

- Keys determine how values are addressed and often how they are partitioned.
- Values may be strings, numbers, collections, or serialized objects.
- The database may not be able to filter efficiently by fields hidden inside
  an opaque value.
- Key design affects locality, distribution, and the operations the system can
  perform efficiently.
- Redis and Amazon DynamoDB are common examples, although their capabilities
  differ substantially.

An application can store a shopping cart by user identifier:

```text
Key:   cart:user:101
Value: {items: [901, 902], total_items: 2}
```

This is efficient when the main operation is "get the cart for user 101." It
is a poor fit if the application frequently needs arbitrary searches across
fields inside every cart.

## Document databases

A document database stores each record as a self-contained JSON-like document.
Documents are grouped into collections and can contain nested objects and
arrays.

- Documents in one collection may have different fields.
- Fields inside a document can be filtered, projected, sorted, aggregated, and
  indexed.
- Related data that is read and changed together can often be embedded in one
  document.
- Duplicated embedded data and unbounded arrays still require careful schema
  design.
- MongoDB is a document database, not merely a key-value store.

A product collection can represent category-specific attributes without
adding every possible attribute to one fixed table:

```json
{
  "_id": 901,
  "name": "Laptop",
  "specifications": {
    "ram": "16 GB",
    "storage": "1 TB"
  },
  "tags": ["electronics", "computer"]
}
```

MongoDB can query and project fields inside the document:

```javascript
db.products.find(
  { "specifications.ram": "16 GB" },
  { name: 1, specifications: 1, _id: 0 }
)
```

## Wide-column databases

A wide-column database organizes rows around partition keys and flexible
groups of columns. Its schema and storage layout are normally designed around
known access patterns instead of ad hoc joins.

- A partition key decides where related rows are stored.
- Rows may have sparse or varying columns, depending on the database.
- Sequential clustering fields can make range scans within a partition
  efficient.
- Poor partition-key choices can create oversized partitions or concentrate
  traffic on a few nodes.
- Cassandra and HBase are common examples for large event, logging, and
  time-series workloads.

A sensor table might group readings by device and order them by timestamp:

```text
device_id | timestamp | temperature | battery
----------|-----------|-------------|--------
A101      | 10:00     | 24.5        | 91
A101      | 10:05     | 25.0        | 90
```

This layout efficiently answers a planned query such as "retrieve device
A101's readings for a time range." A query that omits the partition key may be
far more expensive.

## Graph databases

A graph database represents entities as **nodes** and relationships as
**edges**. Nodes and edges can both have properties.

- Nodes can represent users, products, accounts, or companies.
- Edges can represent relationships such as `FRIEND_OF`, `BOUGHT`, or
  `WORKS_AT`.
- Traversals follow relationships without repeatedly reconstructing them
  through relational joins.
- Common use cases include fraud detection, social networks, knowledge graphs,
  and recommendation systems.
- Neo4j is a common property-graph database.

For example:

```text
(Alice)-[:BOUGHT]->(Laptop)
(Bob)-[:BOUGHT]->(Laptop)
(Bob)-[:BOUGHT]->(Mouse)
```

A recommendation query can traverse from Alice to the laptop, to other buyers
of that laptop, and then to their other purchases. Graph databases are most
valuable when relationship traversal is central to the workload; they are not
automatically faster for ordinary table-shaped queries.

## Choosing a database model

Start with the operations and guarantees the application needs, not with a
database brand.

| Workload characteristic | Model commonly considered |
| --- | --- |
| Multi-record transactions, constraints, and varied queries | Relational |
| Direct lookup by a stable identifier | Key-value |
| Aggregate-shaped records with nested or varying fields | Document |
| High-volume writes queried by known partition and range keys | Wide-column |
| Multi-hop relationship traversal | Graph |

The model is only one part of the decision. Also evaluate consistency and
transaction guarantees, expected data and traffic growth, query flexibility,
operational experience, backup and recovery support, deployment options, and
total cost. A specialized database can simplify its target access pattern but
adds another system that must be operated correctly.
