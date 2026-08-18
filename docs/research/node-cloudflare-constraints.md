# Node/Linux and Cloudflare Workers runtime constraints

Research for [Establish Node and Cloudflare runtime constraints](https://github.com/spiritledsoftware/bussn/issues/4).

**Scope.** This report records constraints that a portable broker must accommodate. It does not choose the SQLite schema, Durable Object layout, relay protocol, or other adapter design. “Correctness constraint” means an assumption that could lose, duplicate, corrupt, or strand broker work. “Performance choice” means a viable implementation trade-off whose selection does not change the portable contract.

## Findings at a glance

| Area | Node/Linux with SQLite and Unix sockets | Cloudflare Workers with Durable Objects and WebSockets |
| --- | --- | --- |
| Durable authority | SQLite transactions and file locks provide persistence and atomic commit. SQLite permits many simultaneous readers but only one simultaneous writer. | Each Durable Object has private, transactional, strongly consistent storage; the SQLite-backed API exposes SQL and KV storage. |
| Coordination | A Unix-domain socket is local IPC. Its pathname is a filesystem entry and can remain after a crash until removed. | A Durable Object is a single point of coordination for requests; its execution is single-threaded and cooperatively multitasked. |
| Process/object lifetime | A Node process exits once its event loop has no more work; normal signal handlers must arrange shutdown. | Objects can hibernate, be evicted, or be restarted. In-memory state is lost; WebSocket hibernation can keep clients connected while the object is re-created. |
| Wakeups | A socket connection is a live process-local channel, not durable storage. The documented socket behavior requires handling stale pathname state after crashes. | Alarms provide a durable wakeup with guaranteed at-least-once execution, while WebSockets provide a live connection and may disconnect on shutdown. |
| Concurrency | SQLite's write serialization and possible `SQLITE_BUSY` outcomes are part of the store's observable operating environment. | Requests can arrive while asynchronous work is awaiting; important state must be in Durable Object storage, not JavaScript memory. |

## Correctness constraints

### 1. Durable storage, not the relay or memory, must be authoritative

**SQLite.** SQLite's transaction documentation says that SQLite supports multiple simultaneous read transactions but only one simultaneous write transaction. A read transaction sees a snapshot, and upgrading a read transaction to a write transaction can fail with `SQLITE_BUSY` when another connection is writing. The locking documentation describes SQLite's locking and journaling as the mechanism that provides atomic commits and protects database integrity ([SQLite, “Transaction”](https://www.sqlite.org/lang_transaction.html), sections 2 and 2.1; [SQLite, “File Locking And Concurrency In SQLite Version 3”](https://www.sqlite.org/lockingv3.html), sections 1 and 2).

The broker therefore cannot treat “publish,” “claim,” “acknowledge,” or lease expiry as an in-memory operation guarded only by a JavaScript mutex. The store adapter has to make the state transition durable and has to expose or handle contention according to the portable contract. A process crash after a relay notification but before a durable commit must not make a message disappear.

**Durable Objects.** Cloudflare documents each Durable Object as having storage that is private to its unique instance, transactional, strongly consistent, and persisted across requests. The SQLite-backed storage page says each storage method is implicitly wrapped in a transaction and its results are atomic and isolated from other storage operations. Cloudflare also documents the Durable Object as a single-threaded, cooperatively multitasked actor ([What are Durable Objects?](https://developers.cloudflare.com/durable-objects/concepts/what-are-durable-objects/); [SQLite-backed Durable Object Storage](https://developers.cloudflare.com/durable-objects/api/storage-api/)).

This is a correctness boundary, not permission to rely on object memory. Durable Object storage is private to one object: a logical broker domain that needs multiple objects must define how messages and claims are partitioned and coordinated; the platform does not make storage from two different objects one transaction.

### 2. Node/Linux Unix sockets are local, addressable files with crash cleanup implications

Node's `node:net` documentation states that Unix-domain sockets are supported on Unix, that `server.listen(path)` and `socket.connect(path)` use a filesystem pathname, and that Linux pathname length is typically 107 bytes (macOS typically 103). If Node creates the socket, `server.close()` unlinks it; if the process crashes, the pathname can remain and must be removed. Linux abstract socket names (a path beginning with `\\0`) are not visible in the filesystem and disappear when all references close ([Node.js `node:net`, “IPC support” and “Identifying paths for IPC connections”](https://nodejs.org/api/net.html#ipc-support)).

Correctness consequences:

- A Unix socket is a local wake/transport path, not a durable message log and not cross-host coordination.
- Startup must distinguish a live listener from a stale pathname; blindly treating `EADDRINUSE` as a live broker can strand clients, while blindly unlinking can remove another live broker's endpoint.
- Crash recovery must not depend on a `close` callback running.
- Endpoint names, filesystem permissions, and path-length limits are deployment inputs.
- Any relay disconnect or process restart must be recoverable by reading durable store state rather than waiting for a notification that may already be gone.

The last two bullets are design implications of the documented Unix-socket lifecycle; they are not additional Node guarantees.

### 3. SQLite's locking environment constrains a Linux store

SQLite's WAL documentation reports that WAL permits readers and a writer to proceed concurrently, but also explicitly says WAL does not work over a network filesystem because it requires shared memory. The same page notes that WAL introduces `-wal` and `-shm` files and can return `SQLITE_BUSY` in some WAL-mode scenarios ([SQLite, “Write-Ahead Logging”](https://www.sqlite.org/wal.html), sections 2, 2.2, 3, and 9).

Correctness consequences:

- A SQLite file used by a local broker must live on a filesystem with the locking and shared-memory behavior SQLite expects; a network filesystem cannot be assumed equivalent.
- Multiple broker processes can contend for the single writer slot. The adapter must make claim/ack/publish transitions safe under `SQLITE_BUSY` rather than assuming an uncontended local mutex.
- WAL sidecar files are part of the database's crash/recovery footprint and must remain with the database under the deployment's filesystem policy.
- A long-running read transaction can hold an old snapshot; claim selection and cleanup logic must use bounded transactions and explicit retry/timeout behavior.

Choosing WAL rather than rollback journaling is a performance/concurrency choice, subject to the filesystem constraint. The one-writer fact and transactional atomicity are correctness constraints.

### 4. Node processes do not provide an always-running lifecycle

Node documents that the `beforeExit` event fires when the event loop has no additional work and that the process normally exits when no work is scheduled. The `exit` event is emitted when the event loop has no more work or when termination is explicit; once at the `exit` stage there is no way to prevent termination. Node also documents signal events such as `SIGINT` and `SIGTERM` ([Node.js `process`, “Event: 'beforeExit'”, “Event: 'exit'”, and “Signal events”](https://nodejs.org/api/process.html#event-beforeexit), https://nodejs.org/api/process.html#event-exit, https://nodejs.org/api/process.html#signal-events).

A broker adapter may run as a persistent server, but the portable semantics cannot require the process to stay alive. Shutdown can interrupt a relay connection or an in-flight handler. Durable claims therefore need expiry/recovery semantics, and the store must remain correct if a process disappears without graceful cleanup. Graceful signal handling is an operational aid, not a substitute for crash recovery.

### 5. Cloudflare Workers request routing and global memory are not stable coordination

Cloudflare's “How Workers works” documentation says an isolate may be spun down or evicted, that a single Workers instance may handle concurrent requests on a single-threaded event loop while asynchronous work is pending, and that there is no guarantee two requests are routed to the same or different Worker instance. It recommends not using or mutating global state ([Cloudflare, “How Workers works”](https://developers.cloudflare.com/workers/reference/how-workers-works/)).

A regular Worker global variable cannot be the broker's durable queue, claim table, lock, or subscriber registry. Concurrent requests must be safe when execution interleaves at `await`, and any process-local optimization must tolerate disappearing or changing isolates. Durable Object storage (not Worker globals) is the relevant authority when a Cloudflare profile needs durable coordination.

### 6. Durable Object memory is disposable, including during WebSocket hibernation

Cloudflare documents that a Durable Object can be hibernated after inactivity, that hibernation removes it from memory, and that its constructor runs again when a later request/event wakes it. The lifecycle documentation further states that objects can be evicted, restarted, or shut down; in-flight requests that access object storage can be stopped with an error during shutdown, and WebSocket requests are terminated during shutdown. Cloudflare explicitly recommends persisting important progress to storage rather than relying on shutdown hooks ([Lifecycle of a Durable Object](https://developers.cloudflare.com/durable-objects/concepts/durable-object-lifecycle/)).

For hibernating WebSockets, Cloudflare says clients remain connected to the Cloudflare network, the object is re-initialized on an event, and `serializeAttachment`/`deserializeAttachment` can restore per-connection state. Attachments are not a general broker log: the documented maximum serialized attachment size is 16,384 bytes ([Use WebSockets](https://developers.cloudflare.com/durable-objects/best-practices/websockets/), “How hibernation works” and `serializeAttachment`).

Correctness consequences:

- Rebuild subscriptions, indexes, and connection metadata from durable state after object construction.
- Treat a WebSocket connection as a notification/stream endpoint whose process can be restarted, not as the message source of truth.
- Handle shutdown-induced disconnects and storage errors as ordinary recovery paths.
- Do not depend on a shutdown callback to release claims or persist progress.

### 7. Durable Object alarms are durable at-least-once wakeups, not exact timers

Cloudflare's Alarms API documents that each Durable Object can schedule one alarm, that alarms have guaranteed at-least-once execution, and that the handler is retried when it throws. It also documents that only one `alarm()` invocation runs at a time for a given object, with exponential-backoff retries (up to six automatic retries for an uncaught failure). A later `setAlarm` replaces an existing scheduled alarm ([Cloudflare, “Alarms”](https://developers.cloudflare.com/durable-objects/api/alarms/)).

An alarm can therefore wake a broker to inspect due work, but its invocation must be idempotent and safe to repeat. The alarm is not a per-message exactly-once timer, and one alarm per object means a schedule would need durable multiplexing if the broker exposes many deadlines. Timing precision and batching are performance/product choices; at-least-once behavior and replacement semantics are correctness constraints.

### 8. WebSocket delivery is live and bounded, not durable publication

Cloudflare documents WebSockets as long-lived bidirectional connections. It recommends the Durable Object WebSocket API when multiple connections need a single point of coordination, and says the Hibernation API allows the object to sleep while clients remain connected. The Workers WebSocket API documents a 32 MiB maximum received message size; an oversized message is closed with code 1009 ([Cloudflare, “Use WebSockets”](https://developers.cloudflare.com/durable-objects/best-practices/websockets/); [Cloudflare, “WebSockets”](https://developers.cloudflare.com/workers/runtime-apis/websockets/)).

Correctness consequences:

- A relay message may be delayed, duplicated, or absent due to connection loss or object shutdown; consumers must be able to catch up from the durable store.
- Broker payload/framing limits must account for the Workers WebSocket maximum or reject/segment larger payloads before sending.
- Connection establishment, closure, and hibernation are lifecycle events, not acknowledgements of durable message state.

The documented ability to connect “thousands” of clients per Durable Object, hibernation billing behavior, batching, and attachment use are capacity/cost/latency choices, not portable delivery guarantees.

## Performance choices (not yet selected)

The sources leave several valid choices open. They should not be smuggled into the portable correctness contract:

- **SQLite journaling:** WAL improves reader/writer overlap and often throughput, while rollback journaling may suit other filesystem or transaction profiles. Either way, SQLite's transaction and locking rules still apply.
- **Relay wake strategy:** Node Unix sockets and Cloudflare WebSockets can reduce polling latency, but a periodic or alarm-driven scan is needed for recovery from missed notifications. Wake frequency, batching, and reconnect backoff affect cost and latency.
- **Cloudflare WebSocket API:** Standard WebSockets keep the object active; the Hibernation API trades reinitialization work for lower idle duration charges. This changes resource/cost behavior, not the need for durable state.
- **In-memory indexes and batching:** Both Node and Durable Objects can cache indexes or batch writes for speed, but caches must be reconstructible and batches must be committed transactionally before being treated as durable.
- **Alarm scheduling:** One durable alarm per Durable Object can drive a scan; how many due messages to process per invocation and how aggressively to reschedule are throughput and latency choices.
- **Claim retry policy:** Backoff and busy-timeout values can tune contention. They cannot turn an uncertain write into an assumed successful write.

## Required accommodations for the eventual portable contract

Without choosing an adapter design, the documented constraints imply that the broker specification must:

1. Define the durable store as the authority for publication, subscription position, claims, lease expiry, and acknowledgements.
2. Define relay notifications as hints/wakeups; correctness must survive missed, duplicated, delayed, and disconnected notifications.
3. Make claim, lease expiry, redelivery, and acknowledgement recovery-safe across process/object restart.
4. Require idempotent or safely repeatable recovery work for Durable Object alarms and reconnecting consumers.
5. Avoid correctness dependence on JavaScript globals, process memory, Durable Object memory, WebSocket attachments, or graceful shutdown.
6. Account for SQLite's single-writer contention, `SQLITE_BUSY`, WAL filesystem requirements, and Unix socket pathname/crash behavior in the Node profile.
7. Account for Durable Object storage privacy/partitioning, single-threaded interleaving, hibernation/restart, one-alarm-per-object, at-least-once alarms, WebSocket disconnects, and WebSocket message-size limits in the Cloudflare profile.
8. State payload and framing limits explicitly; a relay or runtime limit must not silently truncate or acknowledge an oversized message.

These are accommodations, not a recommendation that Node and Cloudflare use identical internal mechanisms.

## Primary sources

- Node.js, [`node:net` IPC support](https://nodejs.org/api/net.html#ipc-support), especially “Identifying paths for IPC connections”.
- Node.js, [`process` lifecycle](https://nodejs.org/api/process.html#process-events), especially [`beforeExit`](https://nodejs.org/api/process.html#event-beforeexit), [`exit`](https://nodejs.org/api/process.html#event-exit), and [signal events](https://nodejs.org/api/process.html#signal-events).
- SQLite, [Transaction](https://www.sqlite.org/lang_transaction.html).
- SQLite, [File Locking And Concurrency In SQLite Version 3](https://www.sqlite.org/lockingv3.html).
- SQLite, [Write-Ahead Logging](https://www.sqlite.org/wal.html).
- Cloudflare, [What are Durable Objects?](https://developers.cloudflare.com/durable-objects/concepts/what-are-durable-objects/).
- Cloudflare, [Lifecycle of a Durable Object](https://developers.cloudflare.com/durable-objects/concepts/durable-object-lifecycle/).
- Cloudflare, [SQLite-backed Durable Object Storage](https://developers.cloudflare.com/durable-objects/api/storage-api/).
- Cloudflare, [Alarms](https://developers.cloudflare.com/durable-objects/api/alarms/).
- Cloudflare, [Use WebSockets](https://developers.cloudflare.com/durable-objects/best-practices/websockets/).
- Cloudflare, [WebSockets](https://developers.cloudflare.com/workers/runtime-apis/websockets/).
- Cloudflare, [How Workers works](https://developers.cloudflare.com/workers/reference/how-workers-works/).
