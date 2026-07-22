# 20 Distributed Systems Scenarios — Architect-Level Answers

A running theme across almost all of these: **synchronous coupling, missing idempotency, shared-resource contention, and observability gaps**. If you internalize those four failure families, you can reason your way through most of them live.

---

### 1. One service suddenly responds in 10s and drags down the whole system. How do you isolate it?

This is a cascading failure through synchronous coupling — one slow dependency saturates the thread pools and connection pools of everything calling it (head-of-line blocking), and the slowness propagates upstream.

Isolate by walking the latency per hop with distributed tracing or per-service RED metrics (rate, errors, duration) to find where the time is actually spent — the symptom appears everywhere, but the source is one node. Check thread/connection-pool saturation, GC pauses, a slow downstream (DB, external API), and any recent deploy on the suspect service.

Fix with bulkheads (isolated thread pools per dependency), aggressive timeouts, and circuit breakers so one slow service can't hold callers hostage.

### 2. Kafka consumers are alive but lag keeps climbing every minute. What do you inspect?

Consumers are up but consuming slower than producers are producing — throughput is below ingress rate.

Inspect per-message processing time, partition count vs consumer count (your parallelism ceiling is bounded by partitions), rebalancing storms, a slow downstream call inside the handler (usually a synchronous DB write or external API), and key distribution — a hot partition caused by skewed keys will lag while others sit idle.

Fix by adding partitions and consumers, batching, moving slow I/O off the hot path, and rebalancing keys. If it's one hot partition, more consumers won't help until you fix the key.

### 3. APIs randomly fail only behind the API Gateway. What could cause this?

The gateway is the differentiator, so look at what it adds: timeout config shorter than backend p99 latency, connection-pool / keep-alive limits at the gateway, retry amplification, rate limiting, or the gateway's load balancer routing to one unhealthy instance.

Inspect gateway access logs and metrics, correlate failures with specific backend pods (often a single bad instance still in rotation), and diff the timeout settings on both sides.

Fix by aligning timeouts end to end, ejecting unhealthy instances via active health checks, and tuning connection pools.

### 4. A retry mechanism starts generating duplicate orders during failures. How do you handle it?

Classic non-idempotent retry: the write succeeded but the response was lost (timeout/network blip), the client retries, and a second order is created.

Fix with idempotency keys — the client generates a unique key per logical operation, and the server deduplicates on it so a retry is a no-op that returns the original result. Only auto-retry idempotent operations, and keep a dedup store (with TTL) for write paths.

### 5. Autoscaling adds more pods but latency keeps rising. Why?

The bottleneck isn't the tier you're scaling. Adding stateless pods does nothing if the real constraint is downstream and shared — DB connection limits, a distributed lock, a shared cache, or an external rate limit. Worse, more pods pile more pressure onto that shared resource.

Inspect DB connection saturation, pool exhaustion, lock contention, and downstream quotas. Fix the actual bottleneck: connection pooling (e.g. pgbouncer), caching, read replicas, or removing the contended lock.

### 6. One microservice deployment causes sudden 4xx spikes across APIs. What do you verify?

4xx means requests are being *rejected as malformed/unauthorized*, not that the server is erroring (that'd be 5xx). After a deploy, that points to a breaking contract change: a schema change, a newly required field, tightened validation, an auth/token format change, or a routing change.

Verify the API contract diff, request/response schemas, auth expectations, and the deploy's changelog against consumer compatibility. Fix by rolling back, making changes backward-compatible, and adding contract testing plus versioning.

### 7. Distributed tracing disappears exactly during high-traffic incidents. What do you improve?

The worst possible time to lose visibility, and it's usually sampling. Under load, head-based sampling drops traces, or the collector/agent gets overwhelmed and drops spans, or buffers fill and shed.

Improve by moving to tail-based sampling (decide after seeing the trace, so you keep the slow and errored ones), scaling the collector, adding backpressure handling, and never sampling out error traces. Also verify context propagation survives async/batching boundaries.

### 8. A cache shared between services returns stale data after scaling. How do you fix it?

More writers and readers after scaling expose invalidation races. Typical causes: no invalidation on write, TTL too long, or a cache-aside race where a read-miss repopulates with a value that a concurrent write already superseded.

Fix with a disciplined invalidation strategy — write-through or explicit invalidate-on-write, versioning / compare-and-set to avoid the stale-repopulate race, and shorter TTLs. Structurally, a *shared* cache across services is a coupling smell; per-service caches remove the whole class of problem.

### 9. Database connections get exhausted across multiple service instances. What do you check first?

Do the math first: instances × pool size per instance versus the DB's max connections. It's easy to exceed the DB limit just by scaling out, since each instance holds its own pool.

Then check for connection leaks (connections not returned to the pool), long-running transactions holding connections open, and slow queries. Fix with an external connection pooler (pgbouncer) so the DB sees a bounded number of connections regardless of instance count, right-sized pools, and query/transaction timeouts.

### 10. Inter-service calls time out intermittently with no infra alerts. How do you debug it?

No CPU/memory/network alerts but timeouts means it's above the infra layer: thread-pool exhaustion, GC pauses, connection-pool waits, DNS resolution latency, or a dependency degrading just below alert thresholds. Ephemeral port exhaustion and TCP retransmits also fit.

Debug with distributed tracing to see *where* the time goes, look at p99/p99.9 rather than averages (intermittent means the tail), capture thread dumps and GC logs during an incident, and watch connection-pool wait metrics.

### 11. Scheduled jobs begin running multiple times after horizontal scaling. How do you prevent it?

Each instance runs its own in-process scheduler, so N instances means N executions of the same job.

Prevent it with leader election or a distributed lock so only one instance fires the job, a clustered scheduler (e.g. Quartz in clustered mode), or by externalizing scheduling entirely — a Kubernetes CronJob or a single-consumer queue that triggers the work exactly once.

### 12. JVM memory slowly increases across pods after every release. How do you investigate?

Steady growth is a leak, and "after every release" specifically hints at a classloader/metaspace leak from redeploys, or an unbounded cache/collection, thread-local leak, or unclosed resources accumulating.

Investigate by taking heap dumps and diffing them over time to find what's retained, watching GC logs for old-gen (or metaspace) that never recovers after full GC, and profiling. Correlate the onset with what the release changed. If metaspace grows on each redeploy, suspect classloader retention.

### 13. One slow downstream API blocks all incoming requests. What pattern would you use?

The combination of **timeout + circuit breaker + bulkhead**, with a fallback for graceful degradation. Timeout bounds how long you wait, the bulkhead isolates the thread pool so the slow dependency can't consume all your workers, and the circuit breaker fails fast once it trips instead of piling requests onto a dead dependency. Where feasible, decouple the call asynchronously via a queue.

### 14. Logs exist everywhere but cross-service debugging is still hard. What's missing?

Correlation. You have logs but no way to stitch a single request across service boundaries. What's missing is a **correlation/trace ID** propagated through every hop (ideally full distributed tracing), plus structured logging and centralized aggregation that can join on that ID. Without it, you're grepping timestamps across systems and guessing.

### 15. APIs work perfectly in staging but fail under production traffic. Why?

Staging almost never replicates production's concurrency, data volume, or traffic distribution. Problems that only appear at scale: connection/pool limits, N+1 queries that are fine on small datasets, cold vs warm caches, rate limits, and race conditions that only surface under real concurrency.

Address it by load testing with production-like traffic shapes, seeding staging with production-scale data volumes, and running soak/chaos tests. Shadowing real production traffic to staging is the strongest signal.

### 16. Message ordering breaks between async services during retries. How do you solve it?

A retried message re-enters the stream after later messages, reordering it — and multiple partitions/consumers break global ordering by design.

Solve it by scoping ordering: partition by the entity key so all messages for one entity land on one partition and one consumer (Kafka's model), and use sequence numbers with a reordering buffer where needed. Better still, make handlers idempotent and commutative so order matters less, or carry a version so consumers can discard stale updates. Strict global ordering means a single partition — expensive, so scope ordering to the entity, not the whole topic.

### 17. Circuit breakers are configured but cascading failures still happen. What could be wrong?

The breaker is likely misconfigured or bypassed: thresholds set too high or timeouts too long so it trips *after* the thread pool is already exhausted; no bulkhead, so workers are consumed before the breaker reacts; retries firing around the breaker and amplifying load; a per-instance breaker that never sees the aggregate; or a fallback that itself calls the failing dependency.

Fix by tuning thresholds and timeouts so the breaker trips before resource exhaustion, adding bulkheads, ensuring per-call timeouts are shorter than the breaker's evaluation window, capping retries with backoff, and adding load-shedding.

### 18. A deployment succeeds but traffic starts timing out after ~20 minutes. What do you inspect?

The delayed onset is the clue — this is resource accumulation, not a bad build. Something slowly fills up: a connection leak exhausting the pool, a memory leak driving GC pressure, threads slowly saturating, file-descriptor leaks, or an unbounded internal queue growing until it stalls.

Inspect the *trends* over that 20-minute window rather than a point-in-time snapshot: connection count, heap and GC frequency, thread count, FD count, and queue depths. Whichever climbs monotonically until the cliff is your culprit.

### 19. A production issue can't be reproduced locally or in staging. What's your approach?

Accept that it's environment-specific — production data, concurrency, config, real dependencies, or timing that lower environments don't reproduce — and bring the investigation to production instead of trying to drag production down to your laptop.

Increase targeted observability in prod (tracing, scoped debug logging, capturing the exact failing request), use canary/feature flags to test hypotheses safely on a slice of traffic, shadow production traffic to a test environment, and methodically diff the environments across data volume, config, scale, and dependency versions.

### 20. One service failure slowly increases latency across the whole platform. Why can this happen?

Same cascade mechanism as #1 and #20's cousin #13: a failing service makes its callers wait on long timeouts, retries stack up, thread pools fill, and backpressure propagates upstream through synchronous dependency chains — often amplified by retry storms.

It happens because of synchronous coupling, generous timeouts, and aggressive retries with no isolation. The fixes are the standard resilience kit: fail fast, bulkheads, circuit breakers, bounded retries with backoff and jitter, backpressure, and asynchronous decoupling wherever a call doesn't need to be synchronous.

---

## The patterns behind the questions

If you group these, they collapse into a handful of principles worth stating explicitly in an interview:

- **Synchronous coupling causes cascades** (1, 5, 13, 17, 20) — isolate with timeouts, bulkheads, circuit breakers, and async decoupling.
- **Retries need idempotency and ordering discipline** (4, 16) — idempotency keys, dedup, partition-by-key.
- **Shared resources are the real bottleneck** (5, 8, 9) — connection poolers, per-service caches, scale the constraint not the tier.
- **Observability must survive the incident** (7, 10, 14, 19) — correlation IDs, tracing, tail-based sampling, trend-based dashboards.
- **Scale and concurrency expose what staging hides** (11, 12, 15, 18) — leader election, leak hunting via trends, production-like load testing.
