# Effect patterns for a minimal broker API

**Question:** Which established Effect APIs and patterns should shape the broker’s public service, lifecycle, streams, schemas, typed failures, and backpressure while avoiding custom abstractions that Effect already provides?

## Answer

Use Effect’s existing `Effect`, `Context.Service`, `Layer`, `Scope`, `Stream`, `Schema`, `Data.TaggedError`, `Queue`, `PubSub`, and `Schedule` primitives. The broker still needs domain interfaces for durable storage and wake-up relays: Effect does not provide a durable, cross-process claim/ack protocol. Those interfaces should be small services expressed as ordinary Effect values, not a second effect system or callback API.

The recommended split is:

- **Broker service:** the small application-facing facade for publishing and opening/operating subscriptions.
- **Store service:** the correctness authority for durable messages, subscriptions, timed claims, acknowledgements, and claim expiry.
- **Relay service:** a best-effort wake-up mechanism. A missed, duplicated, delayed, or disconnected wake-up must not lose a message or change its state; consumers must be able to poll the store again.

Higher-level event, command, queue, job, RPC, and plugin libraries should compose these operations. They should not require separate broker message types.

## Existing primitives to use

### `Effect<A, E, R>` for every operation

Effect’s core type already models a success value, typed expected errors, and required services (`Effect<A, E, R>`). Service methods should return these effects rather than promises, callbacks, or custom task types. This keeps adapter errors visible in the type and lets callers compose publishing, claiming, handling, and acknowledging with normal Effect combinators.

Source: [`Effect.ts` (Effect model)](https://github.com/Effect-TS/effect/blob/196ddab00844ef673adc1489cf55d769725b6fc7/packages/effect/src/Effect.ts#L117-L145)

### `Context.Service` for injectable service contracts

Define service keys with `Context.Service`. The key can be yielded from a generator and its `use` helper retrieves the implementation from the current context. This is the smallest existing dependency-injection seam and naturally supports replacing the store and relay in tests or deployment layers. The class-style form is useful if the public API wants a named service value with a stable identity; the function-style form is sufficient for a plain interface.

Source: [`Context.ts` (service keys and `Service.use`)](https://github.com/Effect-TS/effect/blob/196ddab00844ef673adc1489cf55d769725b6fc7/packages/effect/src/Context.ts#L70-L220)

Suggested shape (illustrative, not a final contract):

```ts
class Broker extends Context.Service<Broker, Broker.Shape>()("BussnBroker") {}

interface Store {
  readonly publish: (message: EncodedMessage) => Effect.Effect<MessageId, StoreError>
  readonly claim: (request: ClaimRequest) => Effect.Effect<Claim | undefined, StoreError>
  readonly acknowledge: (claim: Claim) => Effect.Effect<void, StoreError>
  readonly release: (claim: Claim) => Effect.Effect<void, StoreError>
}
```

The exact operations and types belong to the domain and delivery-contract tickets; the important decision here is that they are normal Effect-returning service methods.

### `Layer.effect` and `Layer.effectContext` for adapter wiring

`Layer<ROut, E, RIn>` already describes provided services, construction errors, and required dependencies. `Layer.effect` constructs one service from an Effect and runs that acquisition in the layer scope; `Layer.effectContext` handles several services. Use these to provide `Store`, `Relay`, and `Broker` implementations. Compose the SQLite/Unix-socket and Durable-Object/WebSocket implementations as layers rather than adding a broker-specific provider mechanism.

Sources: [`Layer.ts` (Layer model)](https://github.com/Effect-TS/effect/blob/196ddab00844ef673adc1489cf55d769725b6fc7/packages/effect/src/Layer.ts#L30-L60), [`Layer.ts` (`Layer.effect`)](https://github.com/Effect-TS/effect/blob/196ddab00844ef673adc1489cf55d769725b6fc7/packages/effect/src/Layer.ts#L960-L1035)

### `Scope`, `Effect.acquireRelease`, and `Effect.forkScoped` for lifecycle

A store connection, relay listener, and subscription are scoped resources. `Effect.acquireRelease` registers cleanup in the current scope; `Effect.scoped` closes the scope on success, failure, or interruption. Long-running consumer loops should use `Effect.forkScoped`, so closing the application or subscription scope interrupts the loop and releases its listener/claim resources. Do not expose a custom `close()` convention as the primary lifecycle mechanism.

Sources: [`Effect.ts` (`scoped` and `acquireRelease`)](https://github.com/Effect-TS/effect/blob/196ddab00844ef673adc1489cf55d769725b6fc7/packages/effect/src/Effect.ts#L6460-L6595), [`Effect.ts` (`forkScoped`)](https://github.com/Effect-TS/effect/blob/196ddab00844ef673adc1489cf55d769725b6fc7/packages/effect/src/Effect.ts#L8600-L8670), [`Scope.ts`](https://github.com/Effect-TS/effect/blob/196ddab00844ef673adc1489cf55d769725b6fc7/packages/effect/src/Scope.ts#L40-L110)

### `Stream` for receiving, not for durable state

A subscription’s receive side should be exposed as `Stream<Delivery, BrokerError, R>` (or an Effect that opens one) because a stream is already a pull-based, effectful sequence. `Stream.fromPull` is the right escape hatch for a durable claim loop: each pull can claim from the store, wait for a relay signal when no claim is available, and retry the store query. This keeps the relay an optimization rather than the source of truth.

Use `Stream.mapEffect` to run handlers and configure concurrency explicitly. Use `Stream.runForEach`/`runDrain` to execute a consumer. Stream scope and interruption then compose with the subscription scope.

Sources: [`Stream.ts` (`fromPull`)](https://github.com/Effect-TS/effect/blob/196ddab00844ef673adc1489cf55d769725b6fc7/packages/effect/src/Stream.ts#L540-L565), [`Stream.ts` (`mapEffect` and concurrency)](https://github.com/Effect-TS/effect/blob/196ddab00844ef673adc1489cf55d769725b6fc7/packages/effect/src/Stream.ts#L1790-L1855), [`Stream.ts` (`runForEach`)](https://github.com/Effect-TS/effect/blob/196ddab00844ef673adc1489cf55d769725b6fc7/packages/effect/src/Stream.ts#L10620-L10685)

Do not use stream completion as acknowledgement. A delivery is acknowledged only by an explicit store operation after its handler succeeds. If a handler fails or its fiber is interrupted, leave the claim to expire or explicitly release it according to the eventual delivery contract.

### `Schema` at untrusted/encoded boundaries

Effect’s `Schema.decodeUnknownEffect` decodes unknown input to a typed value and fails with `SchemaError`; `Schema.decodeEffect` handles an already typed encoded value. Use caller-supplied schemas for message payloads and envelope decoding at adapter boundaries. This avoids manual parsing and keeps serialization/validation errors in the Effect error channel. The broker should not invent a parallel validator or make every payload `unknown` internally.

Source: [`Schema.ts` (`decodeUnknownEffect` and `decodeEffect`)](https://github.com/Effect-TS/effect/blob/196ddab00844ef673adc1489cf55d769725b6fc7/packages/effect/src/Schema.ts#L1500-L1565)

The final contract still needs to decide whether the broker stores encoded bytes plus schema metadata, or stores already-encoded adapter values. This report only recommends using `Schema` for the boundary and allowing application libraries to own payload schemas.

### `Data.TaggedError` for expected broker failures

Represent expected failures as tagged error classes (`StoreUnavailable`, `ClaimExpired`, `InvalidMessage`, and similar only when the domain confirms them). Tagged errors support `Effect.catchTag` and preserve discriminated handling without stringly typed errors. Defects remain defects; do not convert programming bugs into broker errors.

Source: [`Data.ts` (`TaggedError`)](https://github.com/Effect-TS/effect/blob/196ddab00844ef673adc1489cf55d769725b6fc7/packages/effect/src/Data.ts#L720-L765)

### `Queue` and `PubSub` only for local ephemeral coordination

Effect’s `Queue` and `PubSub` are useful inside an adapter or test layer:

- `Queue.bounded(n)` suspends offers when full, providing backpressure.
- `Queue.dropping` and `Queue.sliding` deliberately lose messages and therefore must not be used for durable broker delivery unless a caller explicitly asks for lossy behavior.
- `PubSub` broadcasts to active subscribers and can use bounded backpressure or replay, but its subscriptions are in-memory and scoped. It is a good local implementation of relay notifications, not a replacement for durable store state.

Sources: [`Queue.ts` (queue strategies)](https://github.com/Effect-TS/effect/blob/196ddab00844ef673adc1489cf55d769725b6fc7/packages/effect/src/Queue.ts#L280-L620), [`Queue.ts` (`offer`)](https://github.com/Effect-TS/effect/blob/196ddab00844ef673adc1489cf55d769725b6fc7/packages/effect/src/Queue.ts#L630-L700), [`PubSub.ts` (model and strategies)](https://github.com/Effect-TS/effect/blob/196ddab00844ef673adc1489cf55d769725b6fc7/packages/effect/src/PubSub.ts#L40-L120), [`PubSub.ts` (`publish` and scoped `subscribe`)](https://github.com/Effect-TS/effect/blob/196ddab00844ef673adc1489cf55d769725b6fc7/packages/effect/src/PubSub.ts#L900-L1105)

`Stream.fromQueue` and `Stream.fromPubSub` are convenient for adapter internals and tests, but a durable subscription should be backed by a store pull/claim operation and a relay wake-up path, not by an Effect queue.

### `Schedule` and `Effect.retry` for transient adapter calls

Use `Effect.retry` with a caller- or adapter-selected `Schedule` for transient infrastructure failures where retrying the same store/relay call is safe. `Schedule.exponential` supplies backoff. Lease expiry and redelivery remain broker/store semantics; do not turn them into an implicit `Effect.retry` loop that can duplicate side effects.

Sources: [`Effect.ts` (`retry`)](https://github.com/Effect-TS/effect/blob/196ddab00844ef673adc1489cf55d769725b6fc7/packages/effect/src/Effect.ts#L4040-L4135), [`Schedule.ts` (`exponential`)](https://github.com/Effect-TS/effect/blob/196ddab00844ef673adc1489cf55d769725b6fc7/packages/effect/src/Schedule.ts#L830-L900)

## Backpressure recommendation

There are two different kinds of pressure:

1. **Ephemeral in-process pressure:** use `Queue.bounded` or `Stream.buffer({ capacity, strategy: "suspend" })` for relay notifications and local processing. This controls memory and naturally slows producers. The `Stream.buffer` documentation explicitly distinguishes suspension from dropping/sliding strategies.
2. **Durable broker pressure:** the store contract must define what a publish does when storage or a subscription’s pending work is at a limit. It cannot be delegated to an in-memory queue, because a process can disappear and a relay notification can be missed.

Default to suspension/failure rather than silently dropping messages. Do not expose `dropping` or `sliding` as the broker’s default. If a future lossy telemetry library needs those semantics, it can opt into them above the core.

Source: [`Stream.ts` (`buffer` strategies)](https://github.com/Effect-TS/effect/blob/196ddab00844ef673adc1489cf55d769725b6fc7/packages/effect/src/Stream.ts#L4560-L4625)

## Minimal API direction

The smallest Effect-shaped public surface should have:

- one `Broker` service key for application-facing operations;
- one `Store` service key containing only durable operations required by the delivery contract;
- one `Relay` service key containing only wake-up/send and receive/wait operations;
- Effect-returning methods with typed errors and no promises/callbacks;
- a scoped subscription/consumer handle whose receive side is a `Stream` or pull effect;
- explicit `acknowledge` and `release`/expiry operations, never implicit acknowledgement from stream consumption;
- Layers for each store/relay combination.

The broker should not add custom `Task`, `Resource`, `Observable`, `EventEmitter`, retry, dependency-injection, schema, or backpressure abstractions. The domain-specific Store/Relay interfaces are necessary because Effect has no durable cross-runtime broker protocol; everything around them should use the existing primitives above.

## Decisions this research does not make

The following remain for the domain and delivery-contract tickets: message envelope and identity, destination addressing, subscription persistence, claim ownership and lease renewal, ordering, redelivery rules, retention/cleanup, durable backpressure limits, and whether a subscription API returns a stream directly or an explicit scoped handle containing one. This report intentionally recommends primitives and seams without pre-solving those domain decisions.
