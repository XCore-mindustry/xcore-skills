# XCore Patterns for Avaje Inject

This file maps official Avaje features to the patterns actually used in XCore.

## 1. Composition root in `XcorePlugin`

`XcorePlugin` is the startup boundary and builds the main container:

```java
container = BeanScope.builder()
  .classLoader(getClass().getClassLoader())
  .build();
```

Then it retrieves startup services directly:

```java
var migrationService = container.get(MigrationService.class);
var mapSelector = container.get(SmartMapSelector.class);
var netEvents = container.get(NetEventService.class);
```

### Why this matters

Keep container assembly in one place. Don’t scatter `BeanScope.builder()` calls through ordinary services.

## 2. Constructor injection only

XCore prefers constructor injection and explicitly discourages field injection.

That means when writing new beans for this ecosystem:

- put required dependencies in the constructor
- use `Provider<T>` only when a real lazy/cycle need exists
- avoid field injection unless there is a compelling technical reason

## 3. Factory beans own external/configured resources

XCore uses `@Factory` heavily for:

- config file handles (`Fi`)
- loaded config objects (`Config`, `GlobalConfig`)
- serializer variants (`Gson`)
- transport helpers (`RedisStreamRouter`)
- localization bundle setup (`Bundle`)

This is a strong pattern to copy.

## 4. `@Named` is the main qualifier strategy

XCore uses string-based qualifiers consistently for common shared types:

- `pretty`
- `raw`
- `redis`
- `xcConfigFile`

When adding another shared type variant, follow the same pattern unless a custom qualifier would make the API significantly clearer.

## 5. `Provider<T>` is used deliberately

XCore uses `Provider<T>` in places where direct constructor injection would create awkward coupling, especially for:

- menu graphs
- command/configurer dependencies
- services that need session access but shouldn’t force eager creation order

This is a good example of using provider injection as a boundary-management tool rather than as a default dependency style.

## 6. Lifecycle hooks for startup/shutdown boundaries

Examples:

- `NetworkService` → `@PostConstruct` connect, `@PreDestroy` disconnect
- `CloudService` → `@PostConstruct` command infrastructure init
- `SessionService` → `@PostConstruct` startup log/init hook

Use lifecycle hooks when initialization depends on all collaborators being wired.

## 7. Downstream plugin integration uses parent scopes

Other XCore plugins build child scopes from `XcorePlugin.container`:

```java
scope = BeanScope.builder()
  .classLoader(getClass().getClassLoader())
  .parent(XcorePlugin.container)
  .modules(new EntropyPluginModule())
  .build();
```

This lets downstream plugins reuse shared infrastructure beans while adding their own feature-local beans.

## 8. Testing avoids the full production graph

XCore’s Avaje tests intentionally keep the graph small and local.

Reason: the full graph may depend on runtime state like `Vars.dataDirectory`, networking, or plugin boot order.

### Preferred XCore test style

```java
scope = BeanScope.builder()
  .modules(new LocalTestModule())
  .forTesting()
  .mock(DependencyA.class)
  .mock(DependencyB.class)
  .build();
```

If a bean touches Mindustry globals, stub those first.

## 9. What not to do in this ecosystem

- Don’t make random services build their own scopes
- Don’t switch to field injection casually
- Don’t bootstrap the whole graph in unit tests when a local module is enough
- Don’t add shared-type beans without qualifiers if another bean of that type already exists
- Don’t hide important startup work in constructors when `@PostConstruct` is the right boundary
