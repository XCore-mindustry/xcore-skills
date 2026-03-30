# Scope, Lifecycle, and Conditional Wiring

## Lifecycle hooks

### `@PostConstruct`

Runs after beans are created and wired.

Use it for:

- startup work that depends on fully initialized collaborators
- subscriptions, registrations, connection startup
- initialization that should not run in the constructor

```java
@Singleton
public class NetworkService {

  @PostConstruct
  public void init() {
    safeConnect();
  }
}
```

### `@PreDestroy`

Runs when the scope is closing.

Use it for:

- disconnecting clients
- releasing resources
- closing executors or transports

```java
@PreDestroy
public void disconnect() {
  backend.disconnect();
}
```

Docs note `priority` is supported and defaults to `1000`.

### `AutoCloseable` / `Closeable`

Beans implementing either are treated like they have a pre-destroy close hook.

### Shutdown hook

```java
BeanScope.builder().shutdownHook(true).build();
```

Useful for standalone apps, less important when a plugin/container lifecycle is already owned externally.

## Scopes and lifetimes

### Default scope

`@Singleton` beans live in the default scope.

### `@Prototype`

Creates a new instance each time the bean is requested/wired.

Use sparingly, typically when the object is inherently per-use and stateful.

### `@Lazy`

Defers initialization until needed.

This can be helpful for expensive beans, but many projects get enough control with `Provider<T>` and explicit boundaries.

## Priority selection

When multiple beans match a type:

- `@Primary` → highest priority
- `@Priority(n)` → custom priority
- `@Secondary` → fallback / lowest priority

These are especially useful in modular applications where an override bean should win over a default implementation.

## Conditional wiring

### `@RequiresBean`

Conditionally registers a bean only if another bean is present or absent.

### `@RequiresProperty`

Conditionally registers a bean based on properties/environment values.

### `@Profile`

Conditionally registers a bean based on active profiles (`avaje.profiles`).

### Config property plugin

Docs note condition evaluation can integrate with a `ConfigPropertyPlugin`, otherwise it falls back to system properties and env vars.

## AOP (briefly)

Avaje Inject also supports AOP-style features:

- `@Aspect`
- `AspectProvider`
- `@AOPFallback`

This skill focuses mostly on DI usage, but if the user is implementing interceptors or resilience-like wrappers, mention these and recommend reading current docs directly.
