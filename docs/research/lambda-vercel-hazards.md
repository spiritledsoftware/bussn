# AWS Lambda and Vercel portability hazards

**Question (issue 5):** Which documented lifecycle, storage, connection, concurrency, and invocation constraints in AWS Lambda and Vercel could invalidate assumptions made for Node/Linux or Cloudflare Workers?

**Research date:** 2026-08-18

**Method:** This report uses AWS and Vercel documentation only. It does not select a storage or relay adapter. The implications below are design constraints inferred from the documented platform behavior.

## Executive conclusion

AWS Lambda and Vercel Functions are invocation-oriented compute, not an always-running broker process. A portable broker therefore cannot make correctness depend on a function instance staying alive, retaining memory, retaining a local file, owning a listening socket, or receiving every relay notification.

The core should instead make the durable store authoritative for message state, claims, lease expiry, and acknowledgements. A relay—including a WebSocket relay—can be a best-effort wake-up path. Consumers must reconnect and recover by querying the store; missed, duplicated, delayed, and reordered wake-ups must not change correctness.

Vercel's current Fluid Compute model adds an important difference from traditional single-request serverless assumptions: multiple invocations can run concurrently in one function instance. Code that uses process globals, connection pools, or in-memory subscription state must therefore be safe for concurrent invocations and still treat that state as disposable.

## AWS Lambda

### Lifecycle and invocation duration

- AWS describes standard Lambda as short-lived compute that should not retain or rely on state between invocations. An execution environment can be reused, but Lambda freezes it after work completes and terminates environments every few hours, even when a function is invoked continuously. An invocation failure can reset the environment as well. [AWS, “Understanding the Lambda execution environment lifecycle”](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtime-environment.html) and [AWS, “Lambda quotas”](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html)
- Standard Lambda's maximum function timeout is **900 seconds (15 minutes)**. The timeout covers the invocation phase, including the runtime and extensions. [AWS, “Lambda quotas”](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html) and [AWS, “Understanding the Lambda execution environment lifecycle”](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtime-environment.html)
- Lambda explicitly says there is no independent post-invoke phase. Work that has not completed when the invocation ends cannot be treated as a reliable background worker. [AWS, “Understanding the Lambda execution environment lifecycle”](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtime-environment.html)

**Portability consequence:** A Lambda deployment cannot be the sole resident relay or subscription consumer. It needs an external trigger or polling strategy, and any work in progress must be represented durably before the invocation ends. Claims should have expiry and recovery behavior that does not depend on a shutdown callback.

### Local storage

- `/tmp` is temporary and unique to each Lambda execution environment. Lambda allows 512 MB to 10,240 MB, and the contents may remain when the same environment is frozen and reused. [AWS, “Configure ephemeral storage for Lambda functions”](https://docs.aws.amazon.com/lambda/latest/dg/configuration-ephemeral-storage.html)
- The lifecycle documentation also notes that `/tmp` is not cleared by an environment reset, but the environment itself is not guaranteed to persist. [AWS, “Understanding the Lambda execution environment lifecycle”](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtime-environment.html)

**Portability consequence:** `/tmp` can be an optimization (for example, a local cache), never the broker's log, subscription position, lease, or acknowledgement authority. A store adapter must use a service or resource whose durability and sharing semantics are explicit. The report does not choose that service.

### Concurrency and isolation

- For standard Lambda, each concurrent request gets a separate execution-environment instance. Lambda may reuse an available environment for a later request, but a busy environment does not process another request concurrently. [AWS, “Understanding Lambda function scaling”](https://docs.aws.amazon.com/lambda/latest/dg/lambda-concurrency.html)
- The default account concurrency quota is 1,000 concurrent executions per AWS Region; functions can be throttled when available concurrency is exhausted. Lambda also documents a per-function scaling limit of 1,000 new execution environments every 10 seconds. [AWS, “Understanding Lambda function scaling”](https://docs.aws.amazon.com/lambda/latest/dg/lambda-concurrency.html) and [AWS, “Lambda quotas”](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html)
- Lambda's documented invocation quotas include a synchronous invocation limit of up to 10 requests per second per execution environment (10 times the concurrency limit for a function), while asynchronous invocation has different scaling behavior. [AWS, “Lambda quotas”](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html)

**Portability consequence:** A Lambda consumer cannot assume one process represents one broker domain, one subscription, or one ordered stream. Multiple environments may claim concurrently, and throttling or burst limits may delay claims. Atomic claim/lease operations must live in the store, not in process memory or a local lock.

### Connections and invocation modes

- Lambda may reuse objects initialized outside the handler, and AWS gives database connections as an example of a resource that can be reused across invocations. AWS also recommends checking that a connection exists before creating one. This is an optimization, not a lifetime guarantee: the environment can freeze, reset, or terminate. [AWS, “Understanding the Lambda execution environment lifecycle”](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtime-environment.html)
- Lambda supports synchronous invocation, where the caller waits for a response, and asynchronous invocation, where Lambda queues the event and returns immediately. AWS handles retries for asynchronous invocation. [AWS, “Understanding Lambda function invocation methods”](https://docs.aws.amazon.com/lambda/latest/dg/lambda-invocation.html)
- Standard synchronous request and response payloads are limited to 6 MB each; asynchronous invocation payloads are limited to 1 MB. [AWS, “Lambda quotas”](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html)

**Portability consequence:** A relay connection held by a Lambda invocation must be considered disposable and bounded by invocation lifetime. Publish/claim/ack APIs must tolerate retries and duplicate delivery. If the broker exposes a transport over Lambda invocation, payload limits and the distinction between synchronous and asynchronous invocation become part of that adapter's boundary—not the portable message semantics.

## Vercel Functions

### Lifecycle, regions, and filesystem

- Vercel describes each incoming request as a new function invocation. It may reuse a function instance for a later request, but scales functions down to zero when there are no incoming requests. [Vercel, “Vercel Functions”](https://vercel.com/docs/functions)
- Vercel's runtime documentation says Functions run in a single region by default (currently `iad1`), can be configured for multiple regions on eligible plans, and may fail over to configured regions. Multiple regions therefore require data that is replicated or otherwise valid across those regions. [Vercel, “Runtimes”](https://vercel.com/docs/functions/runtimes)
- Vercel documents a read-only filesystem with writable `/tmp` scratch space up to 500 MB. It also documents archiving functions that are not invoked, with an extra cold-start cost when they are unarchived. [Vercel, “Runtimes”](https://vercel.com/docs/functions/runtimes)

**Portability consequence:** A Vercel function's process memory and `/tmp` are caches only. They cannot be the source of truth for a broker domain, and a deployment that uses multiple regions cannot silently assume one local store is visible everywhere. Region placement and the store's consistency/replication scope must be explicit in a future profile.

### Concurrency and resource sharing

- Fluid Compute is enabled by default for new projects as of April 23, 2025, and allows multiple invocations to share one function instance. Vercel states that optimized concurrency is available for Node.js and Python and that a single instance can process multiple invocations concurrently. [Vercel, “Fluid compute”](https://vercel.com/docs/functions/fluid-compute)
- Vercel's documented Fluid Compute limits include automatic concurrency scaling up to 30,000 for Hobby and Pro or 100,000+ for Enterprise. File descriptors are limited to 1,024 and shared across concurrent executions, including descriptors used by the runtime itself. [Vercel, “Limits”](https://vercel.com/docs/functions/limitations)

**Portability consequence:** Unlike standard Lambda, Vercel Fluid Compute must be treated as concurrent within one process. A broker integration cannot use unsynchronized process globals, assume one active invocation per instance, or open an unbounded number of relay/database sockets. Pools and caches need bounded resource use and cleanup, while correctness still belongs to the external store.

### Duration, WebSockets, and request size

- Vercel documents a maximum invocation duration of 300 seconds on Hobby. Pro and Enterprise have a 300-second default and an 800-second maximum; an 1,800-second (30-minute) extended maximum is documented as beta for supported Node.js and Python versions. A function that exceeds its duration is terminated and returns a function-invocation-timeout error. [Vercel, “Limits”](https://vercel.com/docs/functions/limitations)
- Vercel WebSockets are currently documented as beta. A connection is pinned to one Function instance for its lifetime, but it closes when that function reaches its maximum duration. Reconnected clients are not guaranteed to reach the same instance, and Vercel explicitly recommends an external data store for durable state, presence, counters, rooms, and pub/sub coordination. WebSockets require Fluid Compute. [Vercel, “WebSockets”](https://vercel.com/docs/functions/websockets)
- Vercel limits a Function request body and response body to 4.5 MB. [Vercel, “Limits”](https://vercel.com/docs/functions/limitations)

**Portability consequence:** Vercel WebSockets can be considered a relay/wake-up transport, not a durable subscription or broker authority. The relay must reconnect, re-subscribe, and tolerate instance replacement and deployment rollover. A message API should not require an unbounded streaming connection or an invocation longer than the configured maximum; larger messages need an external payload strategy or explicit rejection.

## Requirements to carry into the portable design

1. **External durable authority:** Store message state, subscription positions, claims/leases, and acknowledgements outside function memory and scratch files.
2. **Best-effort relay:** Treat relay notifications as hints. A missed notification must be recoverable by polling or another store-backed scan; duplicate and reordered notifications must be harmless.
3. **Lease-based recovery:** Claims must expire (or be renewed explicitly) so an invocation killed by timeout, scale-down, reset, or deployment cannot strand a message forever.
4. **Invocation-bounded workers:** A consumer should checkpoint/ack before returning and stop starting work when its remaining invocation budget is too small. Durable work must not depend on post-response execution.
5. **Idempotent retry handling:** Lambda asynchronous retries and platform termination can repeat work. The core contract should define redelivery and let higher-level libraries provide idempotency/result handling.
6. **Concurrent-process safety:** The same API must work when multiple Lambda environments claim concurrently and when Vercel Fluid Compute runs multiple invocations in one process. Atomicity belongs to the store.
7. **Bounded resources:** Adapters must account for platform-specific request/response limits, file-descriptor limits, connection limits, memory, and maximum duration without weakening message correctness.
8. **Explicit region scope:** A broker domain's store and relay must declare their region/replication scope. Vercel's optional multi-region execution and AWS's regional concurrency quotas are not evidence of a shared message domain.
9. **No premature adapter assumptions:** These constraints identify the seams and failure cases. They do not establish that DynamoDB, a Vercel storage product, a particular queue, or a particular relay is the right adapter.

## What remains open

- Which AWS storage and invocation/notification services can implement atomic claims, lease expiry, and durable subscriptions with the required semantics.
- Which Vercel-supported external store and relay combination provides the desired latency, region behavior, and connection economics.
- Whether Lambda's event-source mappings or Vercel's managed Queues/Workflows should be optional higher-level integrations rather than part of the portable core.
- The exact polling, wake-up, backoff, and shutdown APIs after the core delivery contract is decided.

## Sources

All sources are first-party documentation:

- [AWS — Understanding the Lambda execution environment lifecycle](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtime-environment.html)
- [AWS — Understanding Lambda function scaling](https://docs.aws.amazon.com/lambda/latest/dg/lambda-concurrency.html)
- [AWS — Lambda quotas](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html)
- [AWS — Configure ephemeral storage for Lambda functions](https://docs.aws.amazon.com/lambda/latest/dg/configuration-ephemeral-storage.html)
- [AWS — Understanding Lambda function invocation methods](https://docs.aws.amazon.com/lambda/latest/dg/lambda-invocation.html)
- [Vercel — Vercel Functions](https://vercel.com/docs/functions)
- [Vercel — Fluid compute](https://vercel.com/docs/functions/fluid-compute)
- [Vercel — Functions Limits](https://vercel.com/docs/functions/limitations)
- [Vercel — Runtimes](https://vercel.com/docs/functions/runtimes)
- [Vercel — WebSockets](https://vercel.com/docs/functions/websockets)
