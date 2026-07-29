# Microservice Architecture

## Monolithic architecture

A monolithic architecture packages an application's controllers, business
logic, and data-access components into one deployable application. Its
components usually run in the same process or JVM and are commonly packaged as
one JAR, WAR, or container.

The monolith is developed, deployed, and scaled as a unit. This is
operationally simpler than managing several distributed services, but it also
means:

- A change normally requires deploying the complete application.
- Increased load in one feature may require scaling the entire application.
- A process-level failure can affect the complete application.
- Components generally communicate through in-process method calls.

For example, an e-commerce Spring Boot application might contain user,
product, order, payment, and notification components in one project. If only
product traffic increases, another instance of the entire application must
still be deployed.

```text
Client → Monolithic application
         ├── User
         ├── Product
         ├── Order
         ├── Payment
         └── Notification
```

## Microservices architecture

A microservices architecture divides a system into small, independently
deployable services. Each service runs as a separate process or container and
is responsible for a specific business capability.

Services communicate over a network using mechanisms such as HTTP, gRPC, or
asynchronous messages. Each service can be deployed and scaled independently,
although this flexibility introduces network and operational complexity.

```text
                         ┌──→ Order Service
Client → API Gateway ────┼──→ Payment Service
                         ├──→ Inventory Service
                         └──→ Notification Service
```

In an e-commerce system, Order, Payment, Inventory, and Notification can be
separate microservices. During a sale, only the Order and Inventory services
may need additional instances.

### Monolith versus microservices

| Property | Monolith | Microservices |
| --- | --- | --- |
| Deployment | Entire application together | Each service independently |
| Scaling | Entire application | Individual service |
| Communication | Direct method calls | Network calls or messages |
| Failure isolation | Lower | Higher when boundaries and dependencies are designed well |
| Operational complexity | Lower | Higher |

Microservices do not automatically provide isolation. A service can still
cause cascading failures through synchronous dependencies, shared resources,
or uncontrolled retries. The resilience patterns later in this chapter help
preserve the intended isolation.

### Spring Boot service classes are not microservices

A Spring Boot service class is an internal Java component containing business
logic. A microservice is a complete, separately running and deployable
application.

Multiple `@Service` classes inside one Spring Boot application are therefore
not multiple microservices:

- They share the same JVM and deployment.
- They generally call one another through normal Java method calls.
- They cannot normally be deployed or scaled independently.
- Spring Boot can be used to build either one monolith or several independent
  microservice applications.

For example, a project containing `UserService`, `EventService`, and
`PaymentService` with one `main()` method is one monolithic application. Each
capability would need to become a separate Spring Boot application to be an
independently deployable microservice.

## API gateway pattern

An API gateway provides a single entry point through which clients access
multiple backend microservices.

```text
                         /users    → User Service
Client → API Gateway ─── /orders   → Order Service
                         /payments → Payment Service
```

The gateway can:

- Hide internal service addresses from clients.
- Route each request to the correct service.
- Apply authentication, rate limiting, and logging.
- Combine responses from multiple services.

Because client traffic converges at this layer, the gateway can become a
bottleneck or single point of failure if it is not replicated and scaled
properly.

## Database per service pattern

With Database per Service, each microservice owns its data store. Other
services use the owner's API or consume its events instead of directly
modifying its data.

```text
Order Service   → Order database
Payment Service → Payment database
```

This pattern:

- Reduces database-level coupling between services.
- Allows each service to choose a suitable database technology.
- Prevents one service from depending directly on another service's schema.
- Makes cross-service joins and transactions more difficult.

For example, the Order Service stores order data while the Payment Service
stores payment data separately. The Order Service requests payment information
through the Payment Service API rather than querying the payment database.

## Saga pattern

A Saga manages one business transaction across multiple microservices as a
sequence of local transactions. Each service updates only its own database,
then triggers the next step. If a later step fails, compensating transactions
semantically undo the steps that already completed.

Compensation is a business-level undo operation, not a database rollback.

For an online order:

1. The Order Service creates the order.
2. The Payment Service charges the customer.
3. The Inventory Service reserves the item.
4. The Shipping Service creates the shipment.

If inventory reservation fails after payment succeeds, the Payment Service
refunds the customer and the Order Service cancels the order.

```text
Create order → Charge payment → Reserve inventory → Create shipment
                                     │ failure
                                     ↓
                              Refund payment
                                     ↓
                               Cancel order
```

### Choreography versus orchestration

| Property | Choreography | Orchestration |
| --- | --- | --- |
| Control | Distributed among services | A central orchestrator controls the workflow |
| Communication | Events | Commands and responses |
| Advantage | Loose coupling | Workflow is easier to understand |
| Limitation | Complex flows can be difficult to trace | The orchestrator can become complex |

With choreography, each service reacts to events and publishes the event that
drives the next step. With orchestration, a coordinator explicitly tells each
participant what to do.

## Service discovery pattern

Service discovery lets services locate available instances of another service
without storing fixed IP addresses. This is necessary because instances can be
created, removed, or assigned new addresses.

A registry or orchestration platform tracks healthy instances. Client-side or
server-side load balancing then selects one:

```text
Order Service → payment-service → Service discovery → Payment instance
```

For example, the Order Service calls `payment-service` rather than a fixed
server IP. Discovery resolves that logical name to an available Payment
Service instance. Kubernetes commonly provides this behavior through Services
and internal DNS.

## Event-driven communication pattern

Event-driven communication allows a service to publish an event that one or
more interested services consume asynchronously. Publishers announce what
happened without directly calling every consumer.

```text
                                  ┌──→ Notification Service
Order Service → OrderCompleted ───┼──→ Analytics Service
                                  └──→ Loyalty Service
```

This communication style:

- Loosely couples producers and consumers.
- Allows several services to react independently to the same event.
- Can use a message broker such as Kafka to store and distribute events.
- Requires consumers to handle duplicate, delayed, or out-of-order events.

For example, after completing an order, the Order Service publishes an
`OrderCompleted` event. Notification, Analytics, and Loyalty consume it
independently.

## Resilience patterns

Network calls can be slow, fail partially, or reach an unavailable service.
Timeouts, retries, circuit breakers, and bulkheads limit how those failures
affect callers and unrelated work.

### Timeouts and retries

A timeout limits how long a service waits for a response. A retry repeats a
failed operation when the failure may be temporary.

- Every network request should have a reasonable timeout.
- Retries should be limited and use increasing delays.
- Excessive retries can amplify load during an outage.
- Retried operations should be idempotent whenever possible.

For example, the Order Service may wait two seconds for the Payment Service.
If a temporary failure occurs, it retries twice with increasing delays before
returning an error.

### Circuit breaker

A circuit breaker temporarily stops requests to an unhealthy service after
repeated failures. It fails fast instead of tying up resources on calls that
are likely to fail.

The circuit moves through three states:

1. **Closed:** Requests are allowed normally. Repeated failures can open the
   circuit.
2. **Open:** Requests fail immediately without calling the dependency.
3. **Half-open:** A limited test request checks whether the dependency has
   recovered. Success can close the circuit; failure opens it again.

Circuit breakers are commonly combined with timeouts, limited retries, and
fallback responses. If the Payment Service repeatedly times out, for example,
the Order Service can open its circuit and temporarily reject payment requests
instead of waiting for more network timeouts.

### Bulkhead

The Bulkhead pattern isolates resources used by different operations so that
failure or overload in one area cannot exhaust all resources required by the
application.

Bulkheads can separate:

- Thread pools
- Connection pools
- Concurrency limits

For example, an application can reserve separate thread pools for Payment and
Product requests. If Payment requests become slow, Product requests can still
be processed. The resource limits should be configured according to expected
traffic so that isolation does not unnecessarily restrict healthy work.

## CQRS pattern

Command Query Responsibility Segregation (CQRS) separates operations that
modify data from operations that read data:

- **Commands** create, update, or delete data.
- **Queries** retrieve data without modifying it.

Separate paths or models allow read and write workloads to be optimized and
scaled independently. When the read model is updated asynchronously, it may
temporarily lag behind the write model.

For example, an order system can send order creation and cancellation commands
to a write database while serving order-history requests from a separate,
read-optimized database.

## Strangler Fig pattern

The Strangler Fig pattern gradually replaces parts of a monolithic application
with new services rather than rewriting the entire system at once.

```text
                    ┌── extracted Payment Service
Requests → Router ──┤
                    └── remaining monolith
```

- New requests are gradually routed to extracted services.
- Unchanged functionality continues to run in the monolith.
- Incremental extraction reduces the risk of a complete rewrite.
- The transition requires temporary integration between the monolith and the
  new services.

For example, a company might first extract payment processing from its
e-commerce monolith. It can later extract Inventory and Order while the
remaining features continue running in the monolith. Once no traffic depends
on an old implementation, that part of the monolith can be removed.
