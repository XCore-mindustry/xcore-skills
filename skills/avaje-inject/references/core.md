# Core Avaje Concepts

## What Avaje Inject is

Avaje Inject is a compile-time dependency injection library for Java and Kotlin. It generates source code for wiring rather than discovering everything through runtime reflection.

### Why that matters

- faster startup
- explicit generated wiring
- fewer runtime surprises
- easier to debug than opaque reflection-based DI

## Setup

### Gradle

```kotlin
implementation("io.avaje:avaje-inject:${injectVersion}")
annotationProcessor("io.avaje:avaje-inject-generator:${injectVersion}")

testImplementation("io.avaje:avaje-inject-test:${injectVersion}")
testAnnotationProcessor("io.avaje:avaje-inject-generator:${injectVersion}")
```

In XCore this appears as:

```kotlin
implementation(libs.avaje.inject)
annotationProcessor(libs.avaje.inject.generator)
testImplementation(libs.avaje.inject.test)
testAnnotationProcessor(libs.avaje.inject.generator)
```

## Generated code

The processor generates module and bean wiring classes, often with names like:

- `Example$DI`
- `ExampleModule`

Generated modules typically implement `AvajeModule` and register beans into the builder.

## Key annotations

### `@Singleton`

Marks a bean as DI-managed and part of the default scope.

```java
@Singleton
public class UserService { ... }
```

### `@Component`

Avaje-specific alternative to `@Singleton`. Useful when you need Avaje-managed beans without leaning on `@Singleton`, or when mixing ecosystems.

### `@Inject`

Used for constructor, field, or method injection.

#### Constructor injection

- preferred style
- `@Inject` can be omitted on the single constructor

#### Field injection

- supported
- field cannot be `private` or `final`
- mainly useful when breaking circular dependencies

#### Method injection

- supported for setter-like injection patterns

### Circular dependencies

Pure constructor cycles cannot be resolved. If a cycle truly exists, at least one side must use field injection, method injection, or `Provider<T>`.

## Optional and external dependencies

### `Optional<T>`

Supported when a dependency may or may not be present.

### `@Nullable`

Also supported for absent dependencies.

### `@External`

Use when a dependency is expected to be supplied programmatically at runtime via the builder rather than discovered/generated from annotated beans.

## Collection injection

Avaje supports injecting:

- `List<T>`
- `Set<T>`

This is useful when multiple beans implement the same interface and should all be supplied together.

## Exposed types

By default, a bean may be injectable by:

- its concrete type
- implemented interfaces
- superclasses

Use `@BeanTypes` when you want to narrow what types are exposed for injection.
