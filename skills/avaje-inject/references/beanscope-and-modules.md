# BeanScope and Modules

## `BeanScope`

`BeanScope` is the container/runtime view of the Avaje DI graph.

### Basic creation

```java
BeanScope scope = BeanScope.builder().build();
```

### Common builder options

```java
BeanScope.builder()
  .classLoader(getClass().getClassLoader())
  .parent(parentScope)
  .modules(new CustomModule())
  .shutdownHook(true)
  .build();
```

## Scope retrieval

```java
MyService service = scope.get(MyService.class);
```

## Parent-child scopes

Parent-child scopes are useful in plugin ecosystems.

### Pattern

```java
BeanScope child = BeanScope.builder()
  .parent(parent)
  .modules(new ChildModule())
  .build();
```

Use this when:

- a shared root app exposes infrastructure beans
- plugins/features add their own beans without rebuilding the whole graph

### XCore example

`XcorePlugin` exposes a shared parent scope:

```java
container = BeanScope.builder()
  .classLoader(getClass().getClassLoader())
  .build();
```

Downstream plugins then do:

```java
scope = BeanScope.builder()
  .classLoader(getClass().getClassLoader())
  .parent(XcorePlugin.container)
  .modules(new EntropyPluginModule())
  .build();
```

## Modules

Modules are the generated or manual wiring units.

### `@InjectModule`

Avaje uses generated modules for bean registration. A package-level `@InjectModule` can shape the generated module.

Example from XCore:

```java
@InjectModule(name = "xcore")
package org.xcore.plugin;
```

### Manual `AvajeModule`

In tests or specialized assembly, you can write your own module.

```java
public final class MyModule implements AvajeModule {
  @Override
  public Class<?>[] classes() {
    return new Class<?>[]{MyService.class};
  }

  @Override
  public void build(Builder builder) {
    if (builder.isBeanAbsent(MyService.class)) {
      builder.register(new MyService(builder.get(Dep.class)));
    }
  }
}
```

### Why manual modules are useful

- keep tests minimal
- register hand-built objects
- avoid loading the full production graph
- control exactly what the scope contains

## Builder utilities seen in practice

### `builder.isBeanAbsent(...)`

Used in manual modules to avoid duplicate registrations.

### `builder.register(...)`

Registers a concrete instance.

### `builder.get(...)`

Resolves a dependency while constructing a manual bean.

## JPMS / module-info

When using Java modules, Avaje docs note:

- `requires io.avaje.inject`
- `provides io.avaje.inject.spi.InjectSPI ...`

This matters more in modular libraries than in typical plugin projects.
