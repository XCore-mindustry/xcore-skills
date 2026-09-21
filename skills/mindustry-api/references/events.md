# Events System

The event system (`arc.Events`) is the primary integration mechanism for Mindustry plugins. It's a global static event bus that supports class-based events (carrying data) and enum triggers (no data).

**Import**: `import arc.Events;`  
**Event types**: `import mindustry.game.EventType;` and `import mindustry.game.EventType.*;`

## Core API

```java
// Listen for a class-based event
Events.on(PlayerJoin.class, event -> {
    Player player = event.player;
});

// Listen for an enum trigger (no data)
Events.run(Trigger.update, () -> {
    // runs every game tick
});

// Fire a class-based event
Events.fire(new PlayerJoin(player));

// Fire with explicit class key (important for subclasses)
Events.fire(PlayerJoin.class, event);

// Remove a listener (requires exact reference)
Cons<PlayerJoin> listener = e -> { ... };
Events.on(PlayerJoin.class, listener);
Events.remove(PlayerJoin.class, listener);  // must be same reference
```

### Gotcha: Removing Class Listeners vs Trigger Runnables

- `Events.remove(Class<T>, Cons<T>)` works **only** for class-based listeners registered via `Events.on(Class<T>, Cons<T>)`.
- `Events.run(Object trigger, Runnable)` registers a runnable for an enum trigger (e.g. `Trigger.update`, `Trigger.uiDrawEnd`). **Arc does NOT provide a method to remove individual Trigger runnables!**
- **Solution for one-time or cancelable triggers**: Use a boolean guard flag or schedule with `Time.runTask`:

  ```java
  boolean[] done = {false};
  Events.run(Trigger.uiDrawEnd, () -> {
      if (done[0]) return;
      done[0] = true;
      // one-time execution logic
  });
  ```

## Important behavior

- **Not thread-safe** — register and fire only on the main game thread. Use `Core.app.post()` from other threads.
- **Registration order** — listeners fire in the order they were registered.
- **Listeners added during firing** — will NOT be called in that firing cycle (size is captured before iteration).
- **`fire(event)` uses `event.getClass()`** — if the event is a subclass, listeners on the parent class won't fire. Use `Events.fire(ParentClass.class, event)` to target a specific class.
- **`Events.run()` wraps Runnable in Cons** — the wrapper reference is not accessible, so listeners registered via `run()` effectively cannot be removed.
- **Some events reuse instances** — see warnings below. Never cache the event object.

## Player Events

```java
// Player connecting (not yet fully in game)
Events.on(PlayerConnect.class, e -> {
    // e.player — the connecting player
});

// Player received world data, confirmed connection
Events.on(PlayerConnectionConfirmed.class, e -> {
    // e.player
});

// Player fully joined (first join only per connection)
Events.on(PlayerJoin.class, e -> {
    // e.player
});

// Player leaving
Events.on(PlayerLeave.class, e -> {
    // e.player
});

// Player sent a chat message
Events.on(PlayerChatEvent.class, e -> {
    // e.player, e.message
});

// Admin action requested
Events.on(AdminRequestEvent.class, e -> {
    // e.player (admin), e.other (target), e.action (AdminAction enum)
});
```

## Connection Events

```java
// Raw TCP connection established
Events.on(ConnectionEvent.class, e -> {
    // e.connection — NetConnection
});

// Connection handshake packet received
Events.on(ConnectPacketEvent.class, e -> {
    // e.connection, e.packet (ConnectPacket)
});
```

## Ban/Kick Events

```java
Events.on(PlayerBanEvent.class, e -> { /* e.player, e.uuid */ });
Events.on(PlayerUnbanEvent.class, e -> { /* e.player, e.uuid */ });
Events.on(PlayerIpBanEvent.class, e -> { /* e.ip */ });
Events.on(PlayerIpUnbanEvent.class, e -> { /* e.ip */ });
```

## Server Lifecycle Events

```java
// Server fully loaded and ready
Events.on(ServerLoadEvent.class, e -> {
    Log.info("Server ready");
});

// A game has started (map loaded, playing)
Events.on(PlayEvent.class, e -> { });

// Game state reset
Events.on(ResetEvent.class, e -> { });

// Game over
Events.on(GameOverEvent.class, e -> {
    // e.winner — winning Team
});

// A wave has started
Events.on(WaveEvent.class, e -> { });

// Server started hosting
Events.on(HostEvent.class, e -> { });

// Server shutting down
Events.on(DisposeEvent.class, e -> { });

// World tiles loaded
Events.on(WorldLoadEvent.class, e -> { });

// Save written to disk
Events.on(SaveWriteEvent.class, e -> { });

// Game state changed
Events.on(StateChangeEvent.class, e -> {
    // e.from, e.to — State enum values
});
```

## Building Events

```java
// Construction started
Events.on(BlockBuildBeginEvent.class, e -> {
    // e.tile, e.team, e.unit, e.breaking (boolean)
});

// Construction finished
Events.on(BlockBuildEndEvent.class, e -> {
    // e.tile, e.unit, e.team, e.breaking, e.config
});

// Block destroyed
Events.on(BlockDestroyEvent.class, e -> {
    // e.tile
});

// Block configuration changed
Events.on(ConfigEvent.class, e -> {
    // e.tile (Building), e.player, e.value
});

// ⚠️ REUSED INSTANCE — tile block changed
Events.on(TileChangeEvent.class, e -> {
    Tile tile = e.tile;  // extract immediately, do NOT cache 'e'
});

// ⚠️ REUSED INSTANCE — before tile change
Events.on(TilePreChangeEvent.class, e -> {
    // extract data immediately
});
```

## Unit Events

```java
// Unit produced (factory, reconstructor, etc.)
Events.on(UnitCreateEvent.class, e -> {
    // e.unit, e.spawnerBuilding, e.spawnerUnit
});

// Unit spawned by wave
Events.on(UnitSpawnEvent.class, e -> {
    // e.unit
});

// Unit destroyed
Events.on(UnitDestroyEvent.class, e -> {
    // e.unit
});

// Player changed their controlling unit
Events.on(UnitChangeEvent.class, e -> {
    // e.player, e.unit
});

// Player took manual control of a unit
Events.on(UnitControlEvent.class, e -> {
    // e.player, e.unit
});

// ⚠️ REUSED INSTANCE
Events.on(UnitDamageEvent.class, e -> {
    // e.unit, e.bullet — extract immediately
});
```

## Gameplay / Interaction Events

```java
// Items withdrawn from a building
Events.on(WithdrawEvent.class, e -> {
    // e.tile (Building), e.player, e.item, e.amount
});

// Items deposited into a building
Events.on(DepositEvent.class, e -> {
    // e.tile (Building), e.player, e.item, e.amount
});

// Player tapped a tile
Events.on(TapEvent.class, e -> {
    // e.player, e.tile
});

// Player chose a menu option (from Call.menu)
Events.on(MenuOptionChooseEvent.class, e -> {
    // e.player, e.menuId, e.option
});

// Player submitted text input (from Call.textInput)
Events.on(TextInputEvent.class, e -> {
    // e.player, e.textInputId, e.text
});
```

## Enum Triggers (`Trigger`)

Triggers carry no data — they're fire-and-forget signals. Register with `Events.run()`.

```java
Events.run(Trigger.update, () -> {
    // every game tick (~60/sec when not lagging)
});
```

### Key triggers

| Trigger | When |
| --------- | ------ |
| `Trigger.update` | Every game tick |
| `Trigger.beforeGameUpdate` | Before game logic update |
| `Trigger.afterGameUpdate` | After game logic update |
| `Trigger.newGame` | New game started |
| `Trigger.draw` | During render (client only) |
| `Trigger.preDraw` | Before render (client only) |
| `Trigger.postDraw` | After render (client only) |

For server plugins, `Trigger.update` is the most commonly used — it's the per-tick hook for continuous logic.

## Patterns

### One-time event

```java
Cons<ServerLoadEvent> listener = new Cons<>() {
    @Override
    public void get(ServerLoadEvent e) {
        // do something once
        Events.remove(ServerLoadEvent.class, this);
    }
};
Events.on(ServerLoadEvent.class, listener);
```

### Rate-limited tick handler

```java
Interval interval = new Interval();
Events.run(Trigger.update, () -> {
    if (interval.get(300f)) { // every 5 seconds (300 ticks)
        // periodic work
    }
});
```
