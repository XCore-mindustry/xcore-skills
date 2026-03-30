---
name: mindustry-api
description: >
  Mindustry server-side plugin API reference covering Arc framework (Seq, ObjectMap, Events, Log, Timer, Fi, Http,
  CommandHandler, Color, Strings) and Mindustry core (Call, Vars, Groups, Player, Menus, netServer, Administration,
  GameState, Rules, World/Tile, Content). Use this skill whenever writing or reviewing Mindustry plugin code, working
  with Call.*, Vars.*, Groups.*, Events.on/fire, netServer, Menus.register*, Arc collections, or any server-side
  Mindustry Java development. Also use when someone mentions Mindustry modding, plugin lifecycle, player events,
  chat filters, action filters, or custom packets — even if they don't name the specific API class.
---

# Mindustry Server-Side Plugin API

This skill is a quick-reference for the Mindustry + Arc API surface relevant to **server-side plugin development**. It covers the patterns, classes, and methods you actually reach for when building a dedicated-server plugin — not client rendering, shaders, or UI scenes.

## When to read which reference

| You need to...                                          | Read                          |
|---------------------------------------------------------|-------------------------------|
| Send messages, announcements, menus, effects to players | `references/call.md`          |
| Access global state, game rules, map, world tiles       | `references/vars-state.md`    |
| React to player join/leave, waves, builds, chat         | `references/events.md`        |
| Manage connections, bans, filters, admin commands        | `references/networking.md`    |
| Work with Player objects, entity groups, units           | `references/player.md`        |
| Show interactive menus or text inputs                    | `references/menus.md`         |
| Use Seq, ObjectMap, ObjectSet, IntMap, Queue             | `references/arc-core.md`      |
| Log, schedule tasks, read files, make HTTP calls         | `references/arc-util.md`      |

Read only the reference file you need — don't load them all at once.

## Architecture at a glance

```
Arc framework (arc-core)
├── Collections: Seq, ObjectMap, ObjectSet, IntMap, IntSet, Queue, OrderedMap ...
├── Events: global event bus (class-based + enum triggers)
├── Utilities: Log, Timer, Time, Interval, Fi, Http, Strings, Structs, Threads
├── Functional: Cons, Func, Prov, Boolf (Arc's own functional interfaces)
└── Math/Geom: Vec2, Point2, Rect, Geometry, QuadTree, Color

Mindustry core
├── Vars            — static global state (netServer, state, world, content, maps, mods ...)
├── Call (generated) — server→client RPC (messages, UI, world mutations, effects)
├── Groups (generated) — live entity collections (player, unit, build, bullet ...)
├── Player (generated) — player entity with name, team, unit, connection, admin
├── Menus           — interactive menu/text-input registration
├── NetServer       — server networking, commands, admin, chat/action filters
├── Administration  — bans, kicks, whitelist, PlayerInfo, Config
├── EventType       — all event classes + Trigger enum
├── GameState       — wave, rules, map, paused/playing
├── World / Tile    — tile grid access and mutation
└── Content         — blocks, items, liquids, units, status effects (Blocks, Items, UnitTypes ...)
```

## Plugin lifecycle

A Mindustry server plugin is a class extending `mindustry.mod.Plugin` (which extends `Mod`), packaged as a JAR with a `plugin.json` / `plugin.hjson` metadata file in the root.

```java
public class MyPlugin extends Plugin {

    // 1. Constructor — called when the plugin is first instantiated
    public MyPlugin() { }

    // 2. Register console commands (server terminal, "" prefix)
    @Override
    public void registerServerCommands(CommandHandler handler) {
        handler.register("mycmd", "Description", args -> { ... });
    }

    // 3. Register player commands (in-game chat, "/" prefix)
    @Override
    public void registerClientCommands(CommandHandler handler) {
        handler.<Player>register("mycmd", "[arg]", "Description", (args, player) -> {
            player.sendMessage("Hello!");
        });
    }

    // 4. Init — called after ALL plugins are loaded and commands registered
    @Override
    public void init() {
        Events.on(PlayerJoin.class, e -> { ... });
        Events.on(ServerLoadEvent.class, e -> Log.info("Plugin ready"));
    }
}
```

Order: constructor → `registerServerCommands` → `registerClientCommands` → `init`.

Plugins compile against Mindustry as `compileOnly` — never package Mindustry/Arc classes in the plugin JAR.

## Key patterns and conventions

### Everything is global statics

Mindustry uses a static-global architecture. All runtime state lives on `Vars.*`, all entity groups on `Groups.*`, all RPC on `Call.*`. This is by design — embrace it rather than wrapping it.

```java
import static mindustry.Vars.*;

// access state
state.rules.waves = false;
world.tile(10, 10).setBlock(Blocks.router, Team.sharded, 0);
Groups.player.each(p -> p.sendMessage("Hi"));
```

### Thread safety: main thread only

Almost all Mindustry/Arc APIs are **not thread-safe**. Game logic, events, entity access, and `Call.*` must run on the main game thread. If you're on another thread (Redis listener, HTTP callback, database result), dispatch back:

```java
Core.app.post(() -> {
    // safe to touch game state here
    Groups.player.each(p -> p.sendMessage(result));
});
```

### Events are the backbone

The event system is the primary integration point. Listen for events in `init()`:

```java
Events.on(PlayerJoin.class, e -> { ... });
Events.on(PlayerChatEvent.class, e -> { ... });
Events.run(Trigger.update, () -> { /* every tick */ });
```

Some events **reuse their instance** across firings (e.g., `TileChangeEvent`, `UnitDamageEvent`). Never store a reference to the event object — extract what you need immediately.

To remove a listener, you must hold the exact `Cons` reference used at registration time. Listeners registered via `Events.run()` (Runnable wrapper) effectively cannot be removed.

### Call: one-target vs broadcast

`Call` methods come in two flavors:
- **Broadcast**: `Call.sendMessage("text")` — sends to all connected clients
- **Targeted**: `Call.sendMessage(player.con, "text", ...)` — sends to one player's connection

The `NetConnection` parameter is always the first argument when targeting a specific client.

### Arc collections, not Java collections

Mindustry uses Arc's collection types everywhere: `Seq<T>` instead of `ArrayList`, `ObjectMap<K,V>` instead of `HashMap`, `ObjectSet<T>` instead of `HashSet`. These are GC-friendly, use `@`-based iteration, and have different APIs from Java stdlib.

Key gotcha: `ObjectMap` does **not** allow null keys. `Seq` iterator instances are **reused** — breaking out of a for-each early can cause issues; prefer `.each()` or indexed loops.

### Log format: `@` not `{}`

Arc's logging uses `@` as the placeholder character:

```java
Log.info("Player @ joined from @", player.name, player.ip());
Log.err("Failed to load config: @", exception);
```

### CommandHandler parameter syntax

- `<param>` — mandatory (angle brackets)
- `[param]` — optional (square brackets)  
- `<param...>` / `[param...]` — variadic, captures all remaining text, must be last

### Point2 packing

Tile coordinates are often packed into a single `int` using `Point2.pack(x, y)` and unpacked with `Point2.x(packed)`, `Point2.y(packed)`. This is used extensively in network packets and tile references.

## Common gotchas

1. **Reused event instances** — `TileChangeEvent`, `TilePreChangeEvent`, `UnitDamageEvent` etc. reuse the same object. Extract data immediately, never cache the event.

2. **Iterator reuse in Seq/ObjectMap** — Arc reuses iterator objects. Nested for-each over the same collection works (2 cached iterators), but 3+ levels deep will allocate. Breaking out early can clog the iterator. Prefer `.each()`.

3. **Null keys in ObjectMap** — throws, not silently ignored. `ObjectSet.add(null)` is silently ignored.

4. **`Events.fire(subclassInstance)`** — uses `getClass()` for lookup, so it fires on the subclass key, not the parent. Use `Events.fire(ParentClass.class, instance)` to fire on a specific type.

5. **Timer vs Time** — `Timer.schedule()` uses real-world seconds and runs via `Core.app.post()`. `Time.run()` uses game ticks (60/sec) and is cleared on game reset. Use Timer for persistent scheduling, Time for gameplay-tied delays.

6. **HTTP callbacks run on HTTP thread** — always `Core.app.post()` results back to main thread before touching game state.

7. **`player.con` can be null** — for the local player on a non-headless instance. On dedicated servers with real players it's always non-null, but guard against it in shared code.

8. **Menu/text-input IDs are global** — `Menus.registerMenu()` returns an `int` ID. Register once (typically in `init()`), reuse the ID. Don't register per-player or per-invocation.

9. **`Vars.player` is null on headless servers** — it represents the local player, which doesn't exist on dedicated servers.

10. **Generated classes** (`Call`, `Groups`, `Player`, `Unit`, `Building`) live in `mindustry.gen` — they're annotation-processed at build time, not in the source tree.
