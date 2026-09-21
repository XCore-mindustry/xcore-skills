---
name: xcore-mindustry-concurrency
description: Rules for thread safety, asynchronous I/O, and non-blocking operations in Mindustry plugins (XCore-plugin, Sentinel, Gamemodes). Use whenever writing database, network, Redis, or heavy computational logic.
---

# Mindustry Main Thread Safety & Async Architecture

## Core Invariant: NEVER BLOCK THE GAME TICK
Mindustry executes the entire game simulation loop, packet processing, and event dispatch on a single primary thread ("main thread", 60 TPS). Any pause stalls the game for all players.

### Strict Prohibitions on the Main Thread:
1. **NEVER** call `Thread.sleep()` or blocking synchronization waits.
2. **NEVER** call `.get()` or `.join()` on `CompletableFuture`, `RedisFuture`, or reactive Mono/Flux.
3. **NEVER** perform synchronous network calls (HTTP requests, Redis `connection.sync()`, synchronous MongoDB queries).
4. **NEVER** perform synchronous file disk I/O during game ticks.

---

## Architectural Patterns

### 1. Write-Behind (Fire-and-Forget Mutations — 90% of operations)
When updating ratings, persisting session state, recording audit logs, or updating Redis caches:
- Update authoritative in-memory state (`PlayerData`, `Session`) immediately on the game tick.
- Dispatch persistence asynchronously via Lettuce async or Mongo Reactive:

```java
// Redis: Lettuce Async with timeout guard
redisCommands.hset("player:" + id, "rating", String.valueOf(rating))
    .orTimeout(500, TimeUnit.MILLISECONDS)
    .exceptionally(ex -> {
        Log.err("Failed to persist rating for @: @", id, ex.getMessage());
        return null;
    });

// MongoDB: Reactive Streams
mongoCollection.updateOne(Filters.eq("uuid", uuid), Updates.set("pvpRating", rating))
    .subscribe(new LoggingSubscriber<>("Update PvP rating"));
```

### 2. Read-Forward with UI Dispatch (Async Helper — 10% of operations)
When data is genuinely required from storage to render a UI or answer a command:
```java
// Dispatch off-thread, return to main thread safely:
Async.forPlayer(player, () -> statsRepository.findProfile(player.uuid()))
    .thenMain((p, profile) -> {
        // Automatically checks if player is still connected before invoking!
        uiService.showProfileMenu(p, profile);
    })
    .exceptionally(err -> {
        player.sendMessage("[scarlet]Failed to load profile: " + err.getMessage());
    });
```

### 3. Ingress & Security Handshakes
Security checks in `IngressCheck` (e.g. in `xcore-sentinel`) must evaluate in ≤ 50 nanoseconds in memory using local Trie/CIDR tables (`SubnetTable`). Never perform remote database lookups inside the connection handshake hook.
