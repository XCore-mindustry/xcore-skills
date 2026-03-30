# Testing with Avaje Inject

## Recommended approach

For most service tests, prefer a **small local scope** over bootstrapping the whole production graph.

```java
scope = BeanScope.builder()
  .modules(new UserServiceModule())
  .forTesting()
  .mock(UserRepository.class)
  .build();
```

This keeps tests:

- deterministic
- fast
- independent of environment/config/global runtime state

## `forTesting()`

Use `forTesting()` when building a test-specific scope. Pair it with explicit modules and mocked collaborators.

## Mocking

```java
scope = BeanScope.builder()
  .modules(new MyModule())
  .forTesting()
  .mock(SessionService.class)
  .mock(NetworkService.class)
  .build();
```

Then retrieve mocks from the scope:

```java
SessionService sessionService = scope.get(SessionService.class);
```

## Minimal manual modules

When a test should only cover one or two beans, write a tiny `AvajeModule` instead of loading the whole app.

```java
private static final class FindServiceModule implements AvajeModule {
  @Override
  public Class<?>[] classes() {
    return new Class<?>[]{FindService.class, TranslatorLanguagesProvider.class};
  }

  @Override
  public void build(Builder builder) {
    if (builder.isBeanAbsent(TranslatorLanguagesProvider.class)) {
      builder.register(new TranslatorLanguagesProvider());
    }
    if (builder.isBeanAbsent(FindService.class)) {
      builder.register(new FindService(
        builder.get(SessionService.class),
        builder.get(TranslatorLanguagesProvider.class)
      ));
    }
  }
}
```

## Why this pattern matters in XCore

The XCore project explicitly avoids global/full graph bootstrap in some tests because production wiring may touch:

- `Vars.dataDirectory`
- Mindustry runtime globals
- configuration files
- network/backend setup

That means a local test scope is usually the right approach.

## Real XCore test patterns

### Pattern 1: local service + mocks

Seen in:

- `ModerationServiceAvajeTest`
- `SecurityServiceTest`
- `PrivateMessageServiceTest`
- `FindServiceTest`

### Pattern 2: local module + no mocks except framework globals

Seen in:

- `IngressServiceAvajeTest`
- `VoteServiceAvajeTest`

### Pattern 3: stub global runtime first, then build scope

If the tested bean touches Mindustry globals (`Vars.netServer`, `Core.app`, etc.), stub those before container creation.

## `@InjectTest`, `@Mock`, `@Spy`

Official docs mention component testing support through `avaje-inject-test`, including `@InjectTest`, `@Mock`, and `@Spy`.

However, if the project already uses explicit local scopes and custom modules, prefer that existing style unless there is a clear reason to switch.
