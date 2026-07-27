# DNS and Content Delivery Networks

## Domain Name System (DNS)

## What DNS does

The Domain Name System is a distributed lookup system that maps human-readable
domain names such as `example.com` to records used to locate services, most
notably IP addresses.

DNS is like a distributed phone book:

- The browser needs an IP address before it can connect to a web server.
- DNS resolves the domain name to that address.
- After resolution, the browser connects to the destination server.
- The web server—not DNS—returns the page or API content.

DNS is therefore part of connection setup, not page rendering.

## Lookup flow

When an answer is not already cached, resolution follows a hierarchy:

```text
Browser / application cache
        ↓
Operating-system cache and stub resolver
        ↓
Recursive DNS resolver
        ↓
Root name server
        ↓
Top-level-domain (TLD) name server
        ↓
Authoritative name server
        ↓
DNS answer returned and cached
```

For `www.example.com`, the process is:

1. The browser and operating system check their caches.
2. The device asks its configured recursive resolver, often through a router.
3. If the resolver has no cached answer, it asks a root server where to find
   `.com`.
4. The root server refers it to a `.com` TLD server.
5. The TLD server refers it to the authoritative name server for
   `example.com`.
6. The authoritative server returns the requested record for
   `www.example.com`.
7. The resolver caches the answer and returns it to the device.
8. The browser connects to the returned address.

A root server usually returns a referral, not the destination IP. A TLD server
usually identifies the domain's authoritative name server. The authoritative
server provides the final record.

## Components involved

### Client device

The user's computer or phone starts the lookup. The browser and operating
system may each have a DNS cache.

### Router

A home or office router typically forwards DNS traffic and may cache answers.
It is not the complete DNS system.

### Recursive resolver

The recursive resolver performs lookup work on the client's behalf and caches
answers for later clients. It may be operated by an ISP, public DNS provider,
company, or local network.

### Root name server

A root server directs the resolver toward the appropriate TLD name servers,
such as those for `.com` or `.org`.

### TLD name server

The TLD server directs the resolver toward the domain's authoritative name
servers.

### Authoritative name server

The authoritative server is the source of truth for the domain's DNS zone and
returns its configured records.

## Example

When a laptop on home Wi-Fi opens `amazon.com`:

1. The browser checks its DNS cache.
2. The operating system checks its cache and configured DNS settings.
3. If needed, the request passes through the Wi-Fi router to a recursive
   resolver.
4. The resolver finds the address, using its cache or the DNS hierarchy.
5. The answer returns to the laptop and is cached according to its TTL.
6. The browser connects to Amazon's service using the returned address.

## Design implications

- Caching reduces lookup latency and traffic to authoritative servers.
- DNS TTL controls how long an answer can remain cached, trading faster changes
  for fewer lookups.
- DNS can route users toward a nearby CDN edge or regional endpoint.
- DNS changes are not instantaneous because caches may retain the older answer
  until its TTL expires.
- DNS resolution can return multiple addresses for load distribution and
  resilience, but DNS alone does not verify that an application is healthy
  unless it is integrated with health-aware routing.

---

## Content Delivery Network (CDN)

A Content Delivery Network is a distributed network of edge servers that
serves content closer to users instead of sending every request to the origin.

- The **origin** is the backend or storage system containing the original
  content.
- **Edge servers** are geographically distributed points of presence close to
  users.
- A **cache hit** occurs when the edge already has a valid copy.
- A **cache miss** occurs when the edge must fetch the content from the origin
  or an upstream cache.

CDNs reduce user latency, lower origin load, absorb traffic spikes, and can
improve content availability.

### Basic request flow

```text
User
  ↓
DNS directs request to a suitable CDN edge
  ↓
Edge cache ── hit ───────────────→ Return content
  │
  └── miss → Fetch from origin → Cache → Return content
```

DNS and CDN work together: DNS directs the user to a suitable edge endpoint,
and that edge serves or retrieves the requested content.

For example, a viewer in New York can receive video from a nearby edge rather
than a distant origin, reducing network delay and buffering.

### Push CDN

With a push strategy, the origin system or an administrator proactively
uploads content to the CDN before users request it.

**Advantages**

- The first user request can be served from the edge.
- Placement is predictable for planned, popular content.
- The origin avoids a burst of first-time cache misses.

**Trade-offs**

- The system must decide what to distribute and where.
- Unrequested content consumes edge storage and transfer capacity.
- Updating and invalidating stale copies requires careful management.

Push delivery fits predictable, widely accessed content such as software
updates, movie releases, or assets for a major product launch.

### Pull CDN

With a pull strategy, the CDN fetches content from the origin when an edge
receives the first request, then caches it for later requests.

**Advantages**

- Only requested objects occupy the cache.
- It works well when demand is difficult to predict.
- Content placement adapts naturally to regional popularity.

**Trade-offs**

- The first request at an edge has extra origin-fetch latency.
- A sudden request spike for many uncached objects can load the origin.
- Cache eviction may cause later misses.

Pull delivery fits user-generated and long-tail content, such as social-media
images whose popularity is unpredictable.

| Property | Push CDN | Pull CDN |
| --- | --- | --- |
| Placement trigger | Origin or administrator | First user request |
| First request | Usually an edge hit | May be an edge miss |
| Storage use | Content is pre-positioned | Only requested content is cached |
| Best fit | Predictable popular content | Unpredictable or long-tail content |

### Time To Live (TTL)

TTL is the time for which cached content is considered valid before the edge
must revalidate or fetch it again.

- A **long TTL** improves cache-hit rate and reduces origin load, but increases
  the risk of stale content.
- A **short TTL** improves freshness, but increases revalidation, origin
  traffic, and latency.
- After expiry, the CDN may validate the cached object with the origin or
  fetch a fresh copy.

A rarely changing versioned logo can use a long TTL. A frequently changing
news page generally needs a short TTL or an explicit invalidation strategy.
Versioned asset names, such as `app.a1b2c3.js`, allow long TTLs because a
content change creates a new URL.

### Live streaming

Live streaming continuously distributes newly produced media segments rather
than caching one complete static file.

Typical flow:

1. The platform ingests the live feed.
2. It encodes the feed, often at several quality levels.
3. It divides the stream into small segments and updates a manifest.
4. CDN edges distribute recent segments to nearby viewers.

The goals are low startup time, limited buffering, high availability, and
scalable fan-out. Shorter segments can reduce end-to-end delay but increase
request overhead and sensitivity to network variation.

For a widely watched live event, CDN edges allow millions of viewers to share
cached or relayed segments instead of every viewer connecting to the origin.

### CDN versus data-center replication

CDN caching and backend replication solve different problems.

| Concern | CDN | Data-center replication |
| --- | --- | --- |
| Primary goal | Fast, scalable content delivery | Backend data durability and availability |
| Typical data | Images, video, CSS, JavaScript, downloads | Users, orders, payments, application state |
| Location | Edge points of presence | Application or database regions |
| Freshness | Controlled by TTL and invalidation | Controlled by replication and consistency model |
| Source of truth | Usually the origin | Replicated backend data store |

A CDN copy can be discarded and refetched; it is normally not the system of
record. Replicated backend data often is part of the durable system of record.

Real systems use both. A streaming service might replicate account and billing
data across regions while using CDN edges to deliver video. The backend data
prioritizes correctness and durability, while media delivery prioritizes
latency and scale.

### CDN design considerations

- Choose cache keys carefully so one user's private response is not served to
  another user.
- Define invalidation or versioning before caching mutable content.
- Protect the origin against cache-miss storms.
- Decide whether stale content may be served when the origin is unavailable.
- Measure cache-hit ratio, origin traffic, edge latency, error rate, and
  eviction behavior.
- Use access controls and signed URLs or cookies for protected content.
