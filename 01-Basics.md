# System Design Basics

## Distributed systems

A distributed system is a collection of independent components running on
different machines. The components communicate by exchanging messages and
coordinate to achieve a common goal.

Distribution makes it possible to scale and survive individual machine
failures, but it also introduces partial failures, network delays,
coordination, and consistency challenges.

## Performance

Performance describes how a system behaves under a **specific workload**. It
is not a single number; it is a collection of measurements such as:

- Latency
- Throughput and goodput
- Error rate
- CPU, memory, network, and storage utilization
- Queue length
- Uptime and recovery time

For example, suppose a payment system receives 1,000 requests per second.
Under that load it completes 950 requests per second, has an average latency
of 300 ms, a 2% error rate, and 75% CPU utilization. Together, these values
describe its performance. A performance claim without its workload and
measurement conditions is incomplete.

### Common formulas

```text
Error rate (%) = Failed requests / Total requests × 100

Uptime (%) = (Total time - Downtime) / Total time × 100

MTBF = Total operating time / Number of failures

MTTR = Total recovery time / Number of failures
```

- **MTBF (Mean Time Between Failures)** indicates how frequently a repairable
  system fails.
- **MTTR (Mean Time To Repair/Recover)** indicates how quickly service is
  restored after failure.

### Worked example

A system processes 3,600 requests in 12 hours. During that period, 72 requests
fail and the system has three outages of 15 minutes each.

```text
Throughput = 3,600 / 12 = 300 requests/hour

Error rate = 72 / 3,600 × 100 = 2%

MTBF = 12 / 3 = 4 hours

MTTR = 15 minutes

Total downtime = 3 × 15 = 45 minutes
Total time = 12 × 60 = 720 minutes
Uptime = (720 - 45) / 720 × 100 = 93.75%
```

## Scalability

Scalability is a system's ability to handle increasing demand by adding
resources while maintaining acceptable latency, error rate, and other service
objectives.

### Vertical and horizontal scaling

- **Vertical scaling (scale up):** make one machine more powerful by adding
  CPU, memory, or faster storage. It is operationally simple but has a hardware
  ceiling and may preserve a single point of failure.
- **Horizontal scaling (scale out):** add more machines or service instances.
  It offers a higher potential ceiling and better fault isolation, but requires
  load distribution, coordination, and data partitioning or replication.

### Capacity, elasticity, and cost efficiency

**Capacity** is the maximum workload a system can sustain while meeting its
latency and error-rate requirements.

```text
Resource multiplier = New resources / Original resources

Capacity multiplier = New capacity / Original capacity

Capacity increase (%) =
    (New capacity - Original capacity) / Original capacity × 100

Scaling efficiency (%) =
    Capacity multiplier / Resource multiplier × 100

Capacity per dollar = System capacity / Hourly cost
```

**Elasticity** is how quickly a system adds or removes resources as demand
changes. Scalability answers whether capacity can grow; elasticity answers how
quickly capacity adapts.

#### Scaling-efficiency example

A system grows from four servers supporting 1,200 requests per second to eight
servers supporting 2,040 requests per second.

```text
Resource multiplier = 8 / 4 = 2
Capacity multiplier = 2,040 / 1,200 = 1.7
Capacity increase = (2,040 - 1,200) / 1,200 × 100 = 70%
Scaling efficiency = 1.7 / 2 × 100 = 85%
```

The system scales horizontally, but not perfectly: doubling its resources
produces only 1.7 times the capacity. Coordination, contention, or another
bottleneck may explain the loss.

#### Elasticity example

A streaming system has five servers, each able to support 200 streams.

```text
Original capacity = 5 × 200 = 1,000 streams
Demand = 1,600 streams
Required servers = 1,600 / 200 = 8
Additional servers = 8 - 5 = 3
Capacity shortfall before scaling = 1,600 - 1,000 = 600 streams
```

If autoscaling takes five minutes, its scale-up response time is five minutes.
The system needs a strategy for the temporary 600-stream shortfall, such as
headroom, admission control, or graceful degradation.

## Latency

Latency is the elapsed time between sending a request and receiving its
response.

```text
Latency = Response time - Request start time
```

For a payment, latency begins when the user submits the payment and ends when
the user receives confirmation.

### Components of latency

```text
Total latency =
    Network delay
  + Queueing delay
  + Processing delay
  + Database delay
  + External-service delay
```

- **Network delay:** time for data to travel between components.
- **Queueing delay:** time waiting for processing capacity.
- **Processing delay:** time spent executing application logic.
- **Database delay:** time spent reading or writing data.
- **External-service delay:** time waiting on another system.

Average latency can hide a poor user experience for the slowest requests.
Real systems commonly track percentile latency such as p50, p95, and p99
alongside the average.

### Percentile latency: p50, p95, and p99

Percentile latency shows the latency distribution across many requests.

- **p50 latency:** the median latency; 50% of requests finish within this
  time.
- **p95 latency:** 95% of requests finish within this time; the slowest 5%
  take longer.
- **p99 latency:** 99% of requests finish within this time; the slowest 1%
  take longer.

> p50 shows typical latency, while p95 and p99 expose the slow-user
> experience.

#### Example

Suppose ten payment requests have these sorted latencies:

```text
100, 120, 150, 180, 200, 250, 300, 400, 800, 1,200 ms
```

Using the nearest-rank method, the percentile position is:

```text
Position = ⌈percentile × number of requests⌉
```

Therefore:

```text
p50 position = ⌈0.50 × 10⌉ = 5
p50 latency = 200 ms

p95 position = ⌈0.95 × 10⌉ = 10
p95 latency = 1,200 ms

p99 position = ⌈0.99 × 10⌉ = 10
p99 latency = 1,200 ms
```

Most requests are fast, but a few are very slow. The average latency is 370
ms, which does not clearly communicate that some users wait 800–1,200 ms.
The p95 and p99 values make that tail latency visible.

> Average latency can hide outliers; percentile latency exposes them.

### Queueing example

A server processes 100 requests per second and 700 requests arrive
simultaneously. If work proceeds in batches of 100, the groups finish at
approximately one-second intervals:

| Group | Requests | Approximate latency |
| ---: | ---: | ---: |
| 1 | 1–100 | 1 second |
| 2 | 101–200 | 2 seconds |
| 3 | 201–300 | 3 seconds |
| 4 | 301–400 | 4 seconds |
| 5 | 401–500 | 5 seconds |
| 6 | 501–600 | 6 seconds |
| 7 | 601–700 | 7 seconds |

The 700th request takes only one second of processing, but has approximately
seven seconds of total latency because it waits in the queue.

If the timeout is five seconds:

```text
Completed before timeout = 100 × 5 = 500 requests
Timed out = 700 - 500 = 200 requests
Timeout error rate = 200 / 700 × 100 = 28.57%
```

## Throughput

Throughput is the amount of work a system completes in a given period.

```text
Throughput = Completed work / Time
```

Common units include requests per second (RPS), transactions per second
(TPS), and messages per second.

### Incoming rate, completed throughput, and goodput

```text
Incoming rate = Requests received / Time

Completed throughput = Completed requests / Time

Goodput = Successful, useful requests / Time
```

- **Incoming rate** counts requests entering the system.
- **Completed throughput** counts all requests that finish, including errors.
- **Goodput** counts only successful, useful results.

Suppose 100 requests arrive in one second: 95 succeed, three finish with
errors, and two remain queued.

```text
Incoming rate = 100 requests/second
Completed throughput = 95 + 3 = 98 requests/second
Goodput = 95 requests/second
```

Queued work is not completed throughput yet.

### Worked example

A system receives 700 requests and completes 500 within five seconds. Of those
completed requests, 480 succeed and 20 fail. The remaining 200 time out.

```text
Completed throughput = 500 / 5 = 100 requests/second
Goodput = 480 / 5 = 96 requests/second
Overall unsuccessful requests = 20 + 200 = 220
Overall unsuccessful rate = 220 / 700 × 100 = 31.43%
```

## How the basics relate

- A scalable system can add resources as its incoming rate grows.
- Its usable capacity is limited by the first saturated bottleneck.
- When arrivals exceed service capacity, queues grow.
- Growing queues increase latency and may cause timeouts.
- Timeouts and failed requests reduce goodput even if completed throughput
  remains high.
- Performance must therefore be evaluated as a combination of workload,
  latency, throughput, errors, resource use, and availability.
