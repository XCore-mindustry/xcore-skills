# Beans, Factories, Qualifiers, and Providers

## `@Factory` and `@Bean`

Use `@Factory` when a class exists to produce beans programmatically.

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

## When to use a factory

Prefer `@Factory` for:

- third-party library objects
- configuration-backed objects
- multiple variants of the same type
- objects that need ad-hoc construction logic

## `@Bean` options from docs

`@Bean` supports options like:

- `autocloseable = true`
- `initMethod = "..."`
- `destroyMethod = "..."`

It can also return `Optional<T>` for optional beans.

## Qualifiers

### `@Named`

Use `@Named` to disambiguate multiple beans of the same type.

```java
public MaintainController(@Named("xcConfigFile") Fi configFile,
                          @Named("pretty") Gson prettyGson) {
  ...
}
```

Docs note names are case-insensitive.

### `@Qualifier`

Use a custom qualifier annotation when you want stronger typing than raw string names.

### `@QualifiedMap`

Inject `Map<String, T>` keyed by qualifier name when you need all qualified variants together.

## Provider injection

### `Provider<T>`

Use `Provider<T>` when the dependency should be resolved lazily or repeatedly requested.

```java
@Singleton
public class SecurityService {
  private final Provider<SessionService> sessionService;

  public SecurityService(MuteDataRepository muteDataRepository,
                         Provider<SessionService> sessionService) {
    ...
  }
}
```

### Why use it

- break dependency cycles cleanly
- defer heavy bean access until actually needed
- avoid eager initialization order issues

### When not to use it

If plain constructor injection is enough, prefer plain constructor injection.

## XCore examples

### Config and file factories

- `ConfigFactory`
- `SerializationFactory`
- `RedisFactory`
- `BundleFactory`

### Named variants in XCore

- `@Named("pretty")` → pretty-printing `Gson`
- `@Named("raw")` → raw `Gson`
- `@Named("redis")` → transport `Gson`
- `@Named("xcConfigFile")` → config file `Fi`

### Provider usage in XCore

Provider is used heavily around:

- `SessionService`
- `CloudService`
- menu-to-menu dependencies like `MapMenu`, `EventMenu`, `HelpMenu`, `PlayerMenu`

These are good examples of Avaje being used to soften startup coupling without abandoning explicit wiring.
