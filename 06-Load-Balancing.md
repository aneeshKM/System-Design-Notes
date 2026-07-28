# Load Balancing

## Overview

A load balancer distributes incoming client traffic across multiple backend
servers. It improves scalability and availability by allowing servers to
process requests in parallel and by avoiding unhealthy instances.

```text
                         ┌─→ Server A
Client → Load balancer ──┼─→ Server B
                         └─→ Server C
```

A load balancer commonly:

- Prevents one server from receiving all traffic
- Selects a backend using a configured routing algorithm
- Checks backend health and stops using failed instances
- Restores an instance to the pool after it becomes healthy
- Operates at either the transport or application layer

For example, an e-commerce application can place three application servers
behind a load balancer. Requests are distributed among all three while they
are healthy. If one fails, new requests are sent only to the other two.

## Layer 4 load balancing

A Layer 4 (L4) load balancer operates at the transport layer. It makes routing
decisions using connection information such as source and destination IP
addresses, protocol, and port.

- Commonly handles TCP or UDP traffic.
- Does not inspect URLs, cookies, or HTTP headers.
- Can balance protocols other than HTTP, including database and game traffic.
- Has less application context and typically less processing overhead than an
  L7 load balancer.

For example, an L4 load balancer can accept a TCP connection on port 443 and
forward the entire connection to Server A. It can send the next connection to
Server B without knowing which URLs the clients request.

## Layer 7 load balancing

A Layer 7 (L7) load balancer operates at the application layer. It understands
an application protocol, such as HTTP, and can inspect request information
including the path, method, headers, host, and cookies.

- Supports path-, host-, and header-based routing.
- Can direct application features to different backend pools.
- Can apply HTTP-aware behavior such as redirects or request rewriting.
- Provides more routing flexibility but requires more processing than L4
  balancing.

For example, an e-commerce application might route:

- `/images` to media servers
- `/login` to authentication servers
- `/payments` to payment servers

### Layer 4 versus Layer 7

| Property | Layer 4 | Layer 7 |
| --- | --- | --- |
| Routing input | IP address, protocol, and port | URL, host, headers, cookies, method, and other request content |
| Application awareness | No | Yes |
| Typical unit routed | Connection or transport flow | Application request |
| Strength | Fast, general-purpose transport routing | Flexible, application-aware routing |
| Example | Distribute TCP connections among servers | Route `/login` and `/images` to different services |

## Load-balancing algorithms

A load-balancing algorithm determines which healthy backend receives new
traffic. The right choice depends on server capacity, request duration,
connection behavior, and the state visible to the load balancer.

### Round-robin

Round-robin selects servers sequentially in a repeating order:

```text
Request 1 → A
Request 2 → B
Request 3 → C
Request 4 → A
```

It is simple and works well when servers have similar capacity and requests
require roughly similar amounts of work. It can create uneven load when some
requests take much longer than others.

### Least connections

Least connections sends new traffic to the server with the fewest active
connections. If Server A has 30 active connections and Server B has 25, the
next connection goes to Server B.

This approach adapts better than basic round-robin when connection durations
vary, although connection count is only an approximation of actual server
load.

### Weighted load balancing

Weighted load balancing assigns servers different shares of traffic according
to their capacity. It can be combined with round-robin or least-connections.

If Server A has weight 2 and Server B has weight 1:

- Server A receives approximately two out of every three requests.
- Server B receives approximately one out of every three requests.

Weights are useful during mixed-hardware deployments or when gradually
introducing a new server pool.

### Geographic routing

Geographic routing directs users to an appropriate region, often considering
proximity, latency, capacity, policy, and regional health. For example,
European users can be routed to European infrastructure while North American
users are routed to North American infrastructure.

Geography alone is not sufficient: the closest region may be overloaded or
unavailable, so production routing also needs health and capacity signals.

## Global and regional load balancing

Large systems often use two levels of traffic distribution:

```text
                             ┌─→ North America load balancer → Local servers
User → Global routing layer ─┼─→ Europe load balancer        → Local servers
                             └─→ Asia load balancer          → Local servers
```

The global layer selects a region. A regional load balancer then selects a
healthy server within that region. This avoids routing every request through
one distant, centralized load balancer.

The levels can use different algorithms. A global layer might consider
latency, capacity, and region health, while a regional layer uses least
connections. Regions can also use different local policies: one might use
least connections while another uses round-robin for equal-capacity servers.

## Health checks and failover

Health checks determine whether a backend is ready to receive traffic.
Failover redirects traffic after a component or region becomes unavailable.

A typical backend flow is:

1. The load balancer periodically checks each server.
2. After a configured failure threshold, it removes an unhealthy server from
   rotation.
3. New traffic is distributed among the remaining healthy servers.
4. The server continues to be checked while out of rotation.
5. After a recovery threshold, it is returned to the pool.

Checks can test basic network reachability or call an application endpoint
that verifies critical dependencies. A shallow check can mistakenly send
traffic to an application that is running but unable to serve requests. A
check that tests every dependency can instead remove too much capacity during
a downstream failure, so the check should reflect whether the instance can
usefully handle traffic.

Failover can occur at several levels:

- A failed backend is replaced by another server in its pool.
- A failed load-balancer instance is replaced by a peer.
- A failed or unhealthy region is bypassed by the global routing layer.

Health thresholds and timeouts balance detection speed against false
positives. Draining existing connections before removing a server also helps
avoid interrupting in-flight work during planned deployments.

## Load-balancer replication

A single load balancer would itself be a single point of failure. Production
systems therefore run multiple load-balancer instances.

### Active-active

Multiple instances accept traffic simultaneously. If one fails, the remaining
instances continue serving requests, provided they have enough spare capacity.
This design uses all instances during normal operation but requires traffic
distribution and configuration consistency across them.

### Active-passive

One instance accepts traffic while a standby waits to take over. This can make
normal routing simpler, but failover must reliably detect failure and transfer
the active role. The passive instance also reserves capacity that normally
does not serve traffic.

In either model, configuration should be synchronized and the failover path
should be tested. Replicating the load balancer removes one failure point but
does not by itself protect against failures in DNS, the network, a whole
region, or shared configuration.

## Design considerations

- Decide whether routing happens per connection or per request.
- Choose an algorithm that reflects the best available approximation of
  backend load.
- Account for long-lived connections, retries, and uneven request cost.
- Use health checks that represent whether an instance can serve useful work.
- Provide enough spare capacity to absorb instance or regional failures.
- Avoid storing essential session state only on one backend; use shared state
  or carefully designed session affinity when necessary.
- Monitor request rate, active connections, backend latency, error rate,
  rejected traffic, health-check status, and load-balancer saturation.
- Test backend, load-balancer, and regional failover rather than assuming each
  path works.
