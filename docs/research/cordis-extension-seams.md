# Cordis extension seams for the broker

**Question:** Which Cordis patterns for context, service injection, plugin lifecycle, scopes, and disposal should this broker enable, and which plugin-system responsibilities must remain outside the broker?

## Sources and method

This report uses the checked-out Cordis source at commit [`8cc9e33fab69e2d0476d126baaf2acb24e6a6ab4`](https://github.com/cordiverse/cordis/tree/8cc9e33fab69e2d0476d126baaf2acb24e6a6ab4). The implementation and executable tests are the primary sources; Cordis's README describes the project as a meta-framework and links to its documentation ([README](https://github.com/cordiverse/cordis/blob/8cc9e33fab69e2d0476d126baaf2acb24e6a6ab4/README.md)).

## What Cordis actually provides

### 1. A context is a capability scope, not merely a bag of values

`Context` creates and owns core services (`events`, `logger`, `reflect`, and `registry`) and a root `Fiber`. `extend()` creates a derived context through the prototype chain. `isolate(name, label)` creates a context whose service visibility is separated by an isolation label; `intercept(name, config)` adds inherited configuration overrides ([context.ts](https://github.com/cordiverse/cordis/blob/8cc9e33fab69e2d0476d126baaf2acb24e6a6ab4/packages/core/src/context.ts)). The isolation tests show that two derived contexts can host the same plugin while seeing different providers, and that a shared label intentionally shares one provider ([isolate.spec.ts](https://github.com/cordiverse/cordis/blob/8cc9e33fab69e2d0476d126baaf2acb24e6a6ab4/packages/core/tests/isolate.spec.ts)).

**Broker implication:** enable a host to provide broker capabilities through an explicit scope/context (for example, a broker service plus store and relay services), and make identity/namespace boundaries explicit. The broker should not copy Cordis's JavaScript `Proxy`, prototype-shadowing, or dynamic property interception machinery. Those are host-framework techniques, not message-delivery semantics.

### 2. Named services and dependency injection are lifecycle-aware

A `Service` registers a named implementation through `ctx.reflect.provide()` during construction. A plugin may declare dependencies with `inject`; a dependent Fiber remains pending until its dependencies are available, then loads, and unloads/reloads when dependencies disappear or change ([service.ts](https://github.com/cordiverse/cordis/blob/8cc9e33fab69e2d0476d126baaf2acb24e6a6ab4/packages/core/src/service.ts), [registry.ts](https://github.com/cordiverse/cordis/blob/8cc9e33fab69e2d0476d126baaf2acb24e6a6ab4/packages/core/src/registry.ts), [fiber.ts](https://github.com/cordiverse/cordis/blob/8cc9e33fab69e2d0476d126baaf2acb24e6a6ab4/packages/core/src/fiber.ts)). The service tests cover pending activation and multiple dependency declarations ([service.spec.ts](https://github.com/cordiverse/cordis/blob/8cc9e33fab69e2d0476d126baaf2acb24e6a6ab4/packages/core/tests/service.spec.ts)); the Fiber tests cover provider removal and reactivation ([fiber.spec.ts](https://github.com/cordiverse/cordis/blob/8cc9e33fab69e2d0476d126baaf2acb24e6a6ab4/packages/core/tests/fiber.spec.ts)).

**Broker implication:** publish stable, named Effect services/layers for the broker, store, relay, clock, and codecs so a plugin host can inject or replace them. The broker should expose explicit readiness/close effects and typed failures so a host can gate plugin activation on broker availability. It should not implement a general dependency graph, service registry, or automatic plugin reloader; those belong to Cordis-like hosts (or Effect `Layer` composition).

### 3. Plugins are scoped computations with normalized identity

`RegistryService.plugin()` accepts a function, constructor, or object with `apply()`. It normalizes these to a callback identity, creates a Fiber per invocation, tracks active Fibers, validates configuration, and supports nested plugins. `inject()` is implemented as a plugin with dependencies ([registry.ts](https://github.com/cordiverse/cordis/blob/8cc9e33fab69e2d0476d126baaf2acb24e6a6ab4/packages/core/src/registry.ts)). Tests verify all plugin forms, nested plugin disposal, duplicate runtime cleanup, and configuration updates ([plugin.spec.ts](https://github.com/cordiverse/cordis/blob/8cc9e33fab69e2d0476d126baaf2acb24e6a6ab4/packages/core/tests/plugin.spec.ts), [fiber.spec.ts](https://github.com/cordiverse/cordis/blob/8cc9e33fab69e2d0476d126baaf2acb24e6a6ab4/packages/core/tests/fiber.spec.ts)).

**Broker implication:** subscriptions and consumers need stable, explicit identities (destination plus subscription/consumer identity) and must be usable from a host-managed scope. A plugin should be able to create a subscription using the injected broker and have its resources stop when its scope closes. The broker should not decide what a plugin is, discover plugins, normalize plugin forms, validate plugin configuration, order activation, or manage plugin dependency graphs.

### 4. Effects provide hierarchical, idempotent cleanup

`Fiber.effect()` records synchronous, asynchronous, iterable, and async-iterable effects. Each effect returns an idempotent disposer; nested effects are tracked, and disposal runs registered cleanup in reverse order. Plugin unload clears its effects and waits for asynchronous cleanup ([fiber.ts](https://github.com/cordiverse/cordis/blob/8cc9e33fab69e2d0476d126baaf2acb24e6a6ab4/packages/core/src/fiber.ts), [utils.ts](https://github.com/cordiverse/cordis/blob/8cc9e33fab69e2d0476d126baaf2acb24e6a6ab4/packages/core/src/utils.ts)). The disposal tests verify reverse ordering, idempotence, asynchronous cleanup, and cancellation of an in-progress async effect ([dispose.spec.ts](https://github.com/cordiverse/cordis/blob/8cc9e33fab69e2d0476d126baaf2acb24e6a6ab4/packages/core/tests/dispose.spec.ts)). `ctx.on()` itself registers its listener through `ctx.fiber.effect()`, making listener lifetime scope-bound ([events.ts](https://github.com/cordiverse/cordis/blob/8cc9e33fab69e2d0476d126baaf2acb24e6a6ab4/packages/core/src/events.ts)).

**Broker implication:** make subscription/consumer creation return an Effect resource (or accept an Effect `Scope`) with idempotent async close. Closing it must stop relay activity and local reads; in-flight messages must follow the broker's explicit claim/lease rules rather than being silently acknowledged. This is the most important seam for Cordis-style plugins: resource lifetime follows plugin lifetime.

### 5. Cordis events are local dispatch, not durable messaging

`EventsService` stores in-memory hooks and offers `emit`, `parallel`, `serial`, `bail`, `waterfall`, `on`, and `once`. Hook registration is local to a Context/Fiber and is disposed through effects ([events.ts](https://github.com/cordiverse/cordis/blob/8cc9e33fab69e2d0476d126baaf2acb24e6a6ab4/packages/core/src/events.ts)). Tests show synchronous dispatch, parallel error aggregation, serial/bail short-circuiting, and waterfall chaining ([events.spec.ts](https://github.com/cordiverse/cordis/blob/8cc9e33fab69e2d0476d126baaf2acb24e6a6ab4/packages/core/tests/events.spec.ts)).

**Broker implication:** do not reproduce these dispatch modes in the broker core. The broker's durable message stream, claim, lease expiry, acknowledgement, and redelivery semantics are a different layer. A Cordis adapter or higher-level library may translate broker messages into local events and choose serial/parallel behavior there.

## Recommended seam boundary

### The broker should enable

1. **Effect-native capability injection:** store, relay, broker, clock, and codec services can be supplied as Layers/services without the broker owning plugin discovery.
2. **Explicit broker and subscription scopes:** every subscription/consumer has an idempotent asynchronous close and is attachable to a host-managed scope. Scope closure stops local relay/read loops and leaves delivery recovery to lease expiry/redelivery rules.
3. **Stable identities:** broker-domain identity, destination name, subscription identity, and consumer identity are explicit and portable. They must not depend on object identity or process-local callbacks.
4. **Lifecycle observability:** typed status/failure signals for opening, active, closing, unavailable dependencies, and failed reads so a plugin host can decide whether to activate/restart a plugin.
5. **Namespace/isolation hooks:** callers can explicitly choose a broker domain or destination namespace. This preserves the useful boundary represented by Cordis `isolate()` without importing dynamic proxy behavior.
6. **Adapter-independent behavior:** the above lifecycle and resource rules hold whether the store/relay is local or remote; a relay notification is only a wake-up hint, never the source of correctness.

### The broker must not absorb

- Plugin discovery, loading, duplicate plugin normalization, configuration schemas, activation ordering, or dependency graphs.
- A general-purpose service registry or dynamic property injection system.
- Cordis's context proxy/shadow/trace machinery.
- In-process event dispatch policies (`serial`, `parallel`, `bail`, `waterfall`) or event listener APIs.
- Plugin restart/HMR policy. The broker can expose close/reopen and failures; the host decides whether and when to restart.
- Plugin-specific isolation policy beyond explicit broker-domain, namespace, subscription, and consumer identities.

## Resolution

Cordis suggests a narrow contract: the broker should be an Effect-composable capability that offers durable, scope-bound subscriptions with stable identities and clear lifecycle signals. It should integrate cleanly with a Cordis-like host through injected services and effect disposal, while leaving context proxies, service/plugin registries, dependency management, local event dispatch, and restart policy to the host. The broker is infrastructure that a plugin system consumes—not another plugin system.
