# cloud-mindustry API

`cloud-mindustry` is a small adapter that plugs Incendo Cloud v2 into Mindustry `CommandHandler`.

## Main public API

### `MindustryCommandManager<C>`

This is the primary entrypoint.

Key uses:

- create/register commands
- configure permissions
- configure conflict behavior
- attach caption providers
- use inherited Cloud manager APIs

### Manager creation

Simple mode:

```java
var mgr = MindustryCommandManager.create(Vars.netServer.clientCommands);
```

Custom sender mode:

```java
new MindustryCommandManager<>(handler, ExecutionCoordinator.simpleCoordinator(), mapper)
```

## `MindustrySender`

Built-in sender abstraction with two variants:

- player sender
- console sender

Important methods:

- `sendMessage(String)`
- `name()`
- `isPlayer()`
- `player()`

## `SenderMapper`

One of the most important design points.

`cloud-mindustry` receives raw Mindustry-facing senders, then maps them into your command sender type.

In XCore:

```java
SenderMapper<MindustrySender, XCoreSender> mapper = SenderMapper.create(
  base -> new XCoreSender(base, bundle, sessionService),
  XCoreSender::getHandle
);
```

That reverse mapping matters because exception delivery flows back through the original handle.

## `ConflictStrategy`

The bridge defines explicit conflict behavior:

- `FAIL`
- `OVERRIDE`
- `SKIP`
- `PREFIX`

### Practical meaning

- `FAIL` -> throw when a command name collides
- `OVERRIDE` -> replace existing command
- `SKIP` -> do not register the conflicting command
- `PREFIX` -> register with a prefix like `plugin:command` on collision

In XCore, the manager factory currently chooses `OVERRIDE`.

## Permission hook

`MindustryCommandManager` exposes a simple permission-checking hook:

```java
manager.setPermissionChecker((sender, permission) -> ...);
```

If the permission string is empty, it effectively passes.

## Captions

The bridge exposes caption customization:

```java
manager.captionRegistry().registerProvider((caption, sender) -> ...);
```

This is the intended localization extension point.

## Internal implementation detail to avoid centering

`ArcCommandRegistrationHandler` uses reflection to hook into Mindustry/Arc command registration internals.

That is useful context, but it is not the main API users should build around.
