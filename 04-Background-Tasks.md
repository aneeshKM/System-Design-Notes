# Background Tasks

## Overview

A background task runs outside the main user-facing request-response path. It
is useful when work is slow, delayed, or does not need to finish before the
user receives a response.

Common examples include:

- Sending email or notifications
- Processing uploaded files
- Generating reports or media
- Running data pipelines
- Performing cleanup and backups

Background work usually runs in a separate worker process or service. Moving
work off the request path improves perceived latency, but it also makes the
workflow asynchronous: the system must track state, retries, failures, and
eventual completion.

**Example:** A meeting-summary application accepts and stores a transcript,
returns promptly to the user, and has workers generate the summary afterward.

## Event-driven tasks

An event-driven task starts because an event occurs, such as a user action,
database change, message arrival, file upload, or API call.

Typical flow:

```text
Producer → Queue / broker / pub-sub → Worker → Result
```

1. A producer emits an event such as `OrderPlaced` or `FileUploaded`.
2. A queue, message broker, or pub-sub system stores and delivers it.
3. One or more workers consume and process it.

Event-driven tasks react quickly to new work and decouple producers from
consumers. Production designs commonly require:

- Retries with backoff
- Dead-letter queues for work that repeatedly fails
- Idempotent handlers so repeated delivery does not repeat the side effect
- Durable event storage and acknowledgement
- Monitoring for queue depth, task age, failures, and processing time

**Example:** When a food-delivery order is placed, an `OrderPlaced` event
triggers payment processing, restaurant notification, and driver assignment.

## Scheduled tasks

A scheduled task runs at a predefined time or fixed interval. A clock, cron
job, or scheduling service triggers it whether or not new work exists.

Good uses include:

- Nightly backups
- Periodic cleanup
- Billing and settlement reports
- Daily summaries
- Routine maintenance

Scheduled work is simple and predictable, but a run may do nothing when there
is no eligible work. The scheduler also needs a policy for missed runs,
overlapping runs, time zones, and daylight-saving changes.

**Example:** A food-delivery platform calculates restaurant earnings and
generates settlement reports every night at 2:00 AM.

## Polling-based tasks

A polling worker repeatedly checks a database, API, queue, or system condition
and processes work only when it finds eligible items.

```text
Check for work → Work exists? ── yes → Process it
       ↑              │
       └── wait ←──── no
```

Polling is useful when the monitored system cannot publish events.

- A short polling interval reduces detection delay but increases queries and
  resource usage.
- A long interval reduces resource usage but increases detection delay.
- Multiple pollers need a claiming or locking strategy to avoid processing the
  same item concurrently.

**Example:** A meeting-summary worker periodically searches for uploaded
transcripts without summary artifacts and creates the next workflow tasks.

Polling is not the same as ordinary scheduled work. A scheduled job performs a
known task because its configured time arrived. A polling worker wakes
repeatedly to discover whether a condition has become true.

## Choosing a trigger

| Trigger | Starts when | Strengths | Trade-offs | Typical use |
| --- | --- | --- | --- | --- |
| Event-driven | An event arrives | Responsive; naturally decoupled | Broker and delivery semantics add complexity | Order processing |
| Scheduled | A configured time arrives | Predictable; simple | May run without useful work | Nightly settlement |
| Polling | A repeated check finds work | Works without event support | Detection delay and repeated reads | Detecting stale records |

Real systems often combine all three. A food-delivery system might use events
for new orders, a schedule for nightly settlement, and polling to detect
deliveries stuck in an old state.

## Reliability checklist

Regardless of the trigger, consider:

- **Idempotency:** Can the task safely run more than once?
- **Delivery guarantee:** Can work be lost or delivered more than once?
- **Retries:** Which errors are retryable, and with what backoff?
- **Poison work:** Where do repeatedly failing tasks go?
- **Ordering:** Must tasks for the same entity run in order?
- **Concurrency:** Can multiple workers modify the same item safely?
- **Visibility:** Can users and operators see pending, running, failed, and
  completed states?
- **Backpressure:** What happens when producers create work faster than
  workers complete it?
