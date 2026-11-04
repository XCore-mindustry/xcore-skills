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

`create(handler)` coordinates the whole pipeline on the game thread. The three-argument constructor keeps the
coordinator you pass (XCore wraps `simpleCoordinator()` for telemetry); commands issued from another thread through
`commandExecutor()` are still queued onto the game thread first.

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
  base -> new XCoreSender(base, bundle, sessionService, remoteConsole.consoleActor()),
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

## Target selectors

`Player`, `SinglePlayerSelector`, `MultiplePlayerSelector`, `SingleUnitSelector` and `MultipleUnitSelector`
arguments accept `@s`, `@p`, `@r`, `@a`, `@e[...]`, player names and `#id`s. Filters: `team`, `type`,
`distance`/`r`, `x`/`y` (also `~` relative), `admin`, `name`, `limit`/`c`, `sort`
(`nearest`, `furthest`, `random`, `health`, ...).

- Player names with spaces work as typed when an online player has that name, or in quotes (`"Big Bob"`);
  `PlayerNameInput.read(input, names)` does the same in custom parsers (XCore's `PlayerParser` uses it).
- Resolve a selector with `selector.resolve(sender.getHandle())` (the `MindustrySender`, not `XCoreSender`).
- For unit arguments, `@e` selects any unit; `@s`, `@p`, `@r`, `@a` and names select the units of those players
  (players without a unit are skipped).
- A sorted selector considers every candidate before `limit`; results are capped at 100.
- Restrict with `@DenySelectors(reason = ...)` / `@AllowedSelectors(SelectorKind...)`, on the parameter (checked
  while parsing) or on the method (checked for every selector argument after parsing). Literal names and `#id`s
  are never restricted.
- In XCore the `Player` parser is XCore's `PlayerParser`, which delegates `@...` input to
  `PlayerSelectorAdapter` and honours parameter-level restrictions.

## Player-only commands

`@PlayerOnly` (method or class) or `.meta(MindustryCommandManager.PLAYER_ONLY, true)` refuses non-players with the
`mindustry.sender.player_required` caption. It needs `registerMindustryAnnotations(parser)`, which XCore calls.
Client controllers in XCore are registered on the client handler only, so the console never reaches them anyway;
`@PlayerOnly` matters for commands registered on both managers.

## Threads

Game state belongs to the simulation thread (`mgr.simulationExecutor()`). Selector resolution and the
`PlayerInfo` parser throw `IllegalStateException` off that thread instead of blocking. After async work, hop
back with `thenAcceptAsync(..., mgr.simulationExecutor())` before touching players or units.

## Other parsers

- `Team` (base teams; `MindustryCommandManager.ALL_TEAMS`, in XCore `@AllTeams`, for all 256),
  `UnitType`, `Block`, `Item`, `Liquid`, `StatusEffect` by internal name.
- `Administration.PlayerInfo`: an offline record by UUID, any used name, `#id` if online, or IP from the console
  only. Ambiguous names are an error; names with spaces work as typed or quoted.

## Default error handling

The manager installs Cloud's default handlers plus unwrapping of `CommandExecutionException` /
`PipelineException`, and captioned messages for selector, parser and player-only failures. Unexpected failures
are logged and shown as `exception.unexpected`, never with the exception text. XCore replaces several of these
with `CloudExceptionConfigurer` (see `parsers-captions-exceptions.md`).

## Help

`MindustryHelp` renders Cloud help with permission and player-only filtering, pagination and `PREFIX` names
(`mindustry.help.*` captions). XCore has its own help menu and does not use it.

## Internal implementation detail to avoid centering

`ArcCommandRegistrationHandler` uses reflection to hook into Mindustry/Arc command registration internals.

That is useful context, but it is not the main API users should build around.
