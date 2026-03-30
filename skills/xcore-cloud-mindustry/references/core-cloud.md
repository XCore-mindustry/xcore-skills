# Incendo Cloud Core

## Command model

In Cloud, a command is a chain of components.

Components include:

- literals
- required variables
- optional variables
- flags

### Unambiguity rules

- no required component after an optional one
- no conflicting variables at the same level
- literals win over a single variable on the same level

## Builder API

Prefer `CommandManager#commandBuilder(...)` over manual builder construction, because it is tied to the parser registry.

Common builder operations:

- `literal(...)`
- `required(...)`
- `optional(...)`
- `flag(...)`
- `handler(...)`
- `futureHandler(...)`

### Important note

Builders are immutable. Each call returns a new builder.

## Registration

Built commands do nothing until they are registered.

Common registration methods:

- `manager.command(Command)`
- `manager.command(Command.Builder)`
- `manager.command(CommandFactory)`

## Sender type

Cloud is generic over sender type `<C>`.

That matters because platform bridges decide what sender is available to handlers. In XCore, that ultimately becomes `XCoreSender`.

## Permissions

Commands can carry permissions. Core helpers include:

- `Permission.anyOf(...)`
- `Permission.allOf(...)`
- `PredicatePermission.of(...)`

In practice, platform integrations decide how permission strings are checked.

## ExecutionCoordinator

Useful choices:

- `ExecutionCoordinator.simpleCoordinator()`
- `ExecutionCoordinator.asyncCoordinator()`
- `ExecutionCoordinator.builder()`
- `ExecutionCoordinator.coordinatorFor(executor)`

In XCore’s current pattern, the manager factory uses `simpleCoordinator()`.

## Optional vs core features

Core concepts worth centering in the skill:

- manager
- builder API
- parser registry
- permissions
- exception controller
- help handler model

Features to mention more carefully:

- annotation layer
- platform-specific rendering
- rich optional modules outside the XCore usage pattern
