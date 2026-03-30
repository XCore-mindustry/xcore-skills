---
name: avaje-inject
description: >
  Avaje Inject compile-time dependency injection reference for Java projects using BeanScope, @Singleton, @Factory,
  @Bean, @Named, Provider<T>, @PostConstruct, @PreDestroy, @InjectModule, parent-child scopes, and forTesting()
  test wiring. Use this skill whenever the user mentions Avaje Inject, BeanScope, avaje-inject, compile-time DI,
  constructor injection with Avaje, factory beans, qualifiers, Provider<T>, @InjectModule, or asks how to wire,
  test, or structure Avaje-based applications — especially in plugin/server applications or XCore-style projects.
---

# Avaje Inject

This skill is a practical reference for **Avaje Inject** in real Java applications. It focuses on the parts people actually need while implementing or reviewing code:

- bean registration with `@Singleton`, `@Component`, `@Factory`, `@Bean`
- container setup with `BeanScope.builder()`
- module generation with `@InjectModule`
- constructor injection, `Provider<T>`, `Optional<T>`, collection injection
- lifecycle hooks like `@PostConstruct` and `@PreDestroy`
- qualifiers like `@Named`
- parent-child scopes for plugin-style architectures
- minimal Avaje test scopes using `forTesting()` and `.mock(...)`

## When to read which reference

| You need to... | Read |
|---|---|
| Understand Avaje mental model, generated wiring, and annotations | `references/core.md` |
| Build a container, use modules, parent scopes, or custom modules | `references/beanscope-and-modules.md` |
| Register beans with factories, qualifiers, or providers | `references/beans-and-factories.md` |
| Handle startup/shutdown, lazy/prototype/conditional wiring | `references/scope-and-lifecycle.md` |
| Write tests with `forTesting()`, mocks, or minimal module graphs | `references/testing.md` |
| Follow XCore-style patterns and avoid repo-specific mistakes | `references/xcore-patterns.md` |

Read only the reference file that matches the task.

## Mental model

Avaje Inject is **compile-time DI**. It does not rely on runtime classpath scanning or reflection-heavy bean discovery. Instead, annotation processing generates wiring code and module classes.

That means:

- startup is usually fast
- generated code is inspectable and debuggable
- your DI graph is largely validated at build time
- module boundaries matter more than in reflection-heavy DI frameworks

Think of it like this:

1. annotate beans or factories
2. annotation processor generates module/wiring code
3. `BeanScope.builder().build()` assembles those generated modules
4. beans are created, wired, post-constructed, and later destroyed on scope close

## Core patterns

### Prefer constructor injection

Constructor injection is the default and preferred style. If a class has a single constructor, `@Inject` is usually optional.

```java
@Singleton
public class UserService {

  private final UserRepository repository;
  private final Clock clock;

  public UserService(UserRepository repository, Clock clock) {
    this.repository = repository;
    this.clock = clock;
  }
}
```

Prefer this because it keeps dependencies explicit, keeps objects valid after construction, and makes tests easier to wire.

### Use `@Factory` + `@Bean` for external or configured objects

Use factories when you need to construct objects that:

- come from third-party libraries
- need runtime configuration
- need multiple qualified variants
- should not themselves be DI-managed classes

```java
@Factory
public class SerializationFactory {

  @Bean
  @Named("pretty")
  Gson prettyGson() { ... }

  @Bean
  @Named("raw")
  Gson rawGson() { ... }
}
```

### Use `Provider<T>` for lazy access or cycle-breaking

Use `Provider<T>` when eager constructor injection would create an awkward cycle or when the dependency is expensive / late-initialized.

```java
@Singleton
public class HelpMenu {
  private final Provider<CloudService> cloud;

  public HelpMenu(Provider<CloudService> cloud) {
    this.cloud = cloud;
  }
}
```

Do not use `Provider<T>` as a default habit. Use it when it solves a real initialization or dependency problem.

### Use `@Named` when multiple beans share a type

When there are several beans of the same type, qualify them.

```java
public ConfigFactory(@Named("pretty") Gson gson) {
  ...
}
```

### Treat `BeanScope` as the composition root boundary

One place should assemble the scope for the application or plugin. In plugin-style apps, downstream modules can create child scopes from the parent.

```java
BeanScope root = BeanScope.builder()
  .classLoader(getClass().getClassLoader())
  .build();

BeanScope child = BeanScope.builder()
  .parent(root)
  .modules(new MyPluginModule())
  .build();
```

## XCore-specific guidance

In XCore-style projects:

- keep `XcorePlugin` as the composition root
- use constructor injection, not field injection
- use `@Factory` for config objects, `Gson`, `Fi`, routers, and similar external resources
- use `Provider<T>` selectively for menu/controller/service relationships that would otherwise create tight startup coupling
- use minimal local `BeanScope.builder().modules(...).forTesting().mock(...)` test scopes instead of bootstrapping the full graph

If you're working in XCore or a similar plugin ecosystem, read `references/xcore-patterns.md`.

## Common gotchas

1. **Annotation processor missing** — Avaje wiring depends on generated code. If the generator is not configured, the app compiles partially or fails at runtime/container creation.
2. **Using full graph in tests** — this often drags in config, globals, or environment state. Prefer local minimal modules.
3. **Overusing `Provider<T>`** — useful for cycles/lazy access, but it can also hide poor boundaries.
4. **Missing qualifiers** — if you have multiple `Gson`, `Clock`, `DataSource`, or serializer beans, use `@Named` or a custom qualifier.
5. **Putting startup logic in constructors** — prefer `@PostConstruct` for work that should happen after full wiring.
6. **Forgetting shutdown behavior** — close scopes, use `@PreDestroy`, and remember `AutoCloseable` support.
7. **Assuming Spring semantics** — Avaje is smaller and more explicit; many things should be modeled directly with factories/modules instead of broad framework magic.

## Recommended response style when using this skill

When helping the user:

1. identify whether the task is about bean registration, scope assembly, testing, or lifecycle
2. explain the smallest Avaje mechanism that solves it
3. prefer examples that are explicit and compile-time friendly
4. if relevant, explain how the generated module/container wiring behaves
5. if multiple approaches work, recommend the simplest one that preserves explicit structure

## Quick examples

### Basic app scope

```java
BeanScope scope = BeanScope.builder().build();
MyService service = scope.get(MyService.class);
```

### Minimal testing scope

```java
scope = BeanScope.builder()
  .modules(new UserServiceModule())
  .forTesting()
  .mock(UserRepository.class)
  .build();
```

### Parent-child plugin scope

```java
scope = BeanScope.builder()
  .parent(XcorePlugin.container)
  .modules(new EntropyPluginModule())
  .build();
```
