---
name: xcore-mindustry-testkit
description: >
  Deterministic, headless testing toolkit and client simulator for Mindustry v160 plugins on Java 25.
  Covers DeterministicQueue, HeadlessMenuClient, DeterministicUiLoop, UiSnapshot wire verification, and
  testing reactive server-driven UI, async events, and menus without graphics contexts, Xvfb, or thread sleeps.
  Use whenever writing unit tests for Mindustry plugins, simulating player UI interactions, testing network
  roundtrips, or testing xcore-ui components headlessly.
---

# Mindustry Testkit (`mindustry-testkit`)

`mindustry-testkit` provides test infrastructure for testing reactive, server-driven UI, async events, and network lifecycles in Mindustry v160 server plugins on Java 25 without real players, graphics contexts, Xvfb, or thread sleeps.

---

## 1. When to Use

Use `mindustry-testkit` when you need to:

- Test server-driven menus (`xcore-ui`, `Menus.registerMenu`, `Call.menuBuilder*`) headlessly in pure JUnit 5.
- Simulate player actions: opening windows, clicking buttons, closing/dismissing dialogs.
- Deterministically orchestrate asynchronous tasks, turns, and network propagation without race conditions or `Thread.sleep()`.
- Assert exact byte-for-byte binary wire snapshots (`UiSnapshot`) sent to clients.

---

## 2. Core Modules & Architecture

### A. `org.xcore.testkit:core`

Zero dependencies on Mindustry or Arc.

- **`DeterministicQueue`**: Single-threaded, explicitly stepped task queue adhering to Arc's snapshot-drain turn model (`runTurn()`).
  - `post(Runnable)`: Adds a task to the queue.
  - `runNext()`: Executes exactly one pending task.
  - `runTurn()`: Takes a snapshot of all currently pending tasks, clears the queue, and executes the snapshot. Any tasks posted *during* execution wait deterministically for the next turn.

### B. `org.xcore.testkit:fixtures`

Headless World and Entity simulation fixtures for server-side plugin tests (commands, anti-grief action filters, spatial queries, team cores, and packet verification) without launching live server sockets or graphics contexts:

- **`HeadlessWorld`**: Managed, `AutoCloseable` environment providing an initialized `Vars.world` grid, `GameState`, `NetServer`, and `Groups`. Supports spatial queries, floor and block mutation, team cores with multi-tile footprint linking, command routing, and action filtering.
- **`MockPlayer`**: Simulated player entity wrapping `mindustry.gen.Player` backed by an in-memory `MockNetConnection`. Features builders, coordinate/tile positioning, chat, and command dispatch.
- **`MockNetConnection`**: Headless `NetConnection` recording all outgoing `Call.*` RPC packets (`sendMessage`, `announce`, `infoMessage`, every `infoPopup` variant, `warningToast`), player kicks (`kick(String)` / `kick(KickReason)`), and stream chunks without socket I/O.
- **`MockNet`**: Headless `Net` reporting `server()`/`active()`, holding a connection registry so broadcast `Call.*` overloads reach every registered `MockPlayer`.
- **`HeadlessContent`**: Thread-safe base content loader caching vanilla blocks, units, and items once per JVM to guarantee sub-millisecond test fixture setup.
- **`HeadlessWorldExtension` & `@WithHeadlessWorld`**: JUnit 5 Jupiter extension for automated fixture lifecycle management and parameter resolution.

**World generation mode**: `Tile.setFloor` / `Tile.setBlock` fire a `TileFloorChangeEvent` / `TileChangeEvent` and update the pathfinder unless `World.isGenerating()`. Bulk map authoring (map generators, terrain painters, layout planners) should use `HeadlessWorld.builder().generating(true)` so hundreds of writes do not flood the event bus or wake the pathfinder. A single write in a non-generating `HeadlessWorld` is already safe — `Vars.headless` is true so the `Vars.renderer` NPE branch is skipped and `Vars.pathfinder` is null. When the world comes from `@WithHeadlessWorld` there is no builder, so call `world.world().setGenerating(true)` in `@BeforeEach`.

**Popup transcripts**: `Call.infoPopup(con, ...)` sends a different packet than `Call.infoMessage(con, ...)`. `MockNetConnection` records all five variants into one ordered transcript, so assertions never need to reflect into private packet fields. `player.infoPopups()` returns every variant in wire order, `player.lastInfoPopupPacket()` returns an `InfoPopup` record (`message`, `duration`, `align`, `id`, `variant`), and `player.infoMessages()` stays restricted to `InfoMessageCallPacket`. `MockNetConnection.PopupVariant` distinguishes `MESSAGE`, `POPUP`, `RELIABLE`, `KEYED`, and `KEYED_RELIABLE`. Empty-message popups are recorded too, which is how services signal "clear this popup".

### C. `org.xcore.testkit:ui`

Client simulation and wire verification (declares Mindustry/Arc as `compileOnly`).

- **`HeadlessMenuClient`**: Semantic stand-in for the Mindustry client menu registry (`MenuDialog` + `Menus`).
  - `show(menuId, token, hidePrevious, snapshot)`: Simulates server displaying a window.
  - `update(menuId, targetId, snapshot)`: Simulates partial VSlot DOM updates.
  - `click(menuId, action)`: Simulates player clicking a button.
  - `dismiss(menuId)`: Simulates player pressing Escape / Back.
  - `outbox()`: Queue of client-to-server responses (`MenuChoose(menuId, token, action)`).
- **`DeterministicUiLoop`**: Orchestrates two isolated FIFO transport queues (`serverToClient` and `clientToServer`) plus a `serverPost` queue.
  - `stepServerToClient()`: Delivers one pending packet to the headless client.
  - `flushClientOutbox()`: Moves client clicks into the client-to-server transport queue.
  - `stepClientToServer()`: Delivers one client click/cancel packet to the server listener.
  - `runTurn()`: Flushes all queues in deterministic order.
- **`UiSnapshot`**: Immutable wire copy of `NodeBuilder<?>` trees captured at send time using Mindustry's native binary codec (`builder.write(Writes)`). Eliminates mutable reference leaks between test and production code.
- **`UiTranscript`**: Append-only log recording every transport message (`Show`, `Update`, `Hide`, `Choose`) for exact assertions.

---

## 3. Standard Test Pattern

```java
import org.junit.jupiter.api.Test;
import org.xcore.testkit.ui.DeterministicUiLoop;
import org.xcore.testkit.ui.UiWireMessage;
import static org.assertj.core.api.Assertions.assertThat;

class ProfileMenuTest {

    @Test
    void shouldHandleButtonClickAndPartialUpdate() {
        DeterministicUiLoop loop = new DeterministicUiLoop();
        
        // 1. Trigger server opening a menu for player
        var menu = new ProfileMenu();
        menu.open(loop); // sends UiWireMessage.Show to loop.sendServerToClient(...)

        // 2. Deliver packet to client
        loop.stepServerToClient();
        assertThat(loop.client().isVisible(1)).isTrue();

        // 3. Simulate player clicking a button
        loop.client().click(1, "toggle_notifications");
        loop.flushClientOutbox();

        // 4. Deliver response to server and process server-side update
        loop.stepClientToServer();
        loop.serverPost().runTurn(); // drain server-side state transitions

        // 5. Verify server emitted a partial DOM update
        loop.stepServerToClient();
        assertThat(loop.transcript().updates()).hasSize(1);
    }
}
```

---

## 4. Gradle Configuration

```kotlin
repositories {
    maven("https://maven.x-core.org/releases")
    maven("https://maven.x-core.org/snapshots")
}

dependencies {
    testImplementation("org.xcore.testkit:fixtures:0.1.0-SNAPSHOT")
    testImplementation("org.xcore.testkit:ui:0.1.0-SNAPSHOT")
}
```

`fixtures` pulls in `core` transitively and declares Mindustry/Arc as `compileOnly`, so it never embeds game engine classes.

For local work without publishing, add a property-guarded `includeBuild` to the consumer's `settings.gradle.kts` so CI keeps resolving published artifacts by default:

```kotlin
if (providers.gradleProperty("xcoreTestkitFromSource").orNull.toBoolean()) {
    includeBuild(file("../mindustry-testkit"))
}
```

```bash
./gradlew test -PxcoreTestkitFromSource=true
```
