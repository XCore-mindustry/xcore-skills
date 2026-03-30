# Call — Server-to-Client RPC

`Call` is an auto-generated static class (`mindustry.gen.Call`) that provides server→client remote procedure calls. Every method annotated with `@Remote` in the Mindustry source generates a corresponding static method on `Call`.

**Import**: `import mindustry.gen.Call;`

## Targeting

- **Broadcast** (no `NetConnection` param): sends to ALL connected clients
- **Targeted** (`NetConnection` as first param): sends to ONE specific client

```java
Call.sendMessage("Hello everyone");              // all players
Call.sendMessage(player.con, "Hello you", null, null); // one player
```

## Table of Contents

- [Messaging](#messaging)
- [Announcements & HUD](#announcements--hud)
- [Menus & Text Input](#menus--text-input)
- [World Labels](#world-labels)
- [Effects](#effects)
- [World Mutation](#world-mutation)
- [Units & Players](#units--players)
- [Connection Management](#connection-management)
- [Custom Packets](#custom-packets)
- [Rules & State](#rules--state)

---

## Messaging

```java
// Broadcast chat message
Call.sendMessage(String message)

// Send to one player (fromPlayer can be null for system messages)
Call.sendMessage(NetConnection con, String message, String unformatted, Player fromPlayer)
```

## Announcements & HUD

```java
// Large center-screen announcement
Call.announce(String message)
Call.announce(NetConnection con, String message)

// Modal info dialog (player must dismiss)
Call.infoMessage(String message)
Call.infoMessage(NetConnection con, String message)

// HUD popup (non-blocking, timed)
Call.infoPopup(String message, float duration, int align, int top, int left, int bottom, int right)
Call.infoPopupReliable(...)  // TCP version

// Toast notification
Call.infoToast(String message, float duration)
Call.infoToast(NetConnection con, String message, float duration)

// Warning toast with icon
Call.warningToast(int iconID, String message)
Call.warningToast(NetConnection con, int iconID, String message)

// Persistent HUD text (stays until hidden)
Call.setHudText(String message)
Call.setHudTextReliable(String message)
Call.hideHudText()

// Open URL in player's browser
Call.openURI(String uri)
Call.openURI(NetConnection con, String uri)
```

## Menus & Text Input

See `references/menus.md` for the full pattern. These are the Call-level methods:

```java
// Show a selectable menu
Call.menu(NetConnection con, int menuId, String title, String message, String[][] options)

// Update an existing menu
Call.followUpMenu(NetConnection con, int menuId, String title, String message, String[][] options)

// Hide a follow-up menu
Call.hideFollowUpMenu(NetConnection con, int menuId)

// Show a text input dialog
Call.textInput(NetConnection con, int textInputId, String title, String message,
               int maxLength, String defaultText, boolean numeric)
```

## World Labels

```java
// World-space text label (visible to all, UDP)
Call.label(String text, float duration, float worldX, float worldY)

// TCP version (reliable delivery)
Call.labelReliable(String text, float duration, float worldX, float worldY)

// Targeted variants
Call.label(NetConnection con, String text, float duration, float worldX, float worldY)
Call.labelReliable(NetConnection con, String text, float duration, float worldX, float worldY)
```

## Effects

```java
// Spawn a visual effect at a position
Call.effect(Effect effect, float x, float y, float rotation, Color color)
Call.effect(NetConnection con, Effect effect, float x, float y, float rotation, Color color)
```

## World Mutation

```java
// Set a tile's block
Call.setTile(Tile tile, Block block, Team team, int rotation)

// Set floor/overlay
Call.setFloor(Tile tile, Block floor, Block overlay)

// Change a building's team
Call.setTeam(Building building, Team team)
```

## Units & Players

```java
// Unit lifecycle
Call.unitSpawn(Tile tile, UnitType type, int count, Team team, float x, float y)
Call.unitDeath(int unitId)
Call.unitDestroy(int unitId)

// Kick a player
Call.kick(NetConnection con, KickReason reason)
Call.kick(NetConnection con, String message)

// Redirect player to another server
Call.connect(NetConnection con, String ip, int port)
```

## Connection Management

```java
// Ping-related
Call.pingResponse(NetConnection con, long time)

// World sync
Call.worldDataBegin(NetConnection con)
```

## Custom Packets

For plugin-to-plugin or plugin-to-client-mod communication:

```java
// String packets
Call.clientPacketReliable(String type, String contents)   // client → server
Call.serverPacketReliable(String type, String contents)   // server → client (broadcast)
Call.serverPacketReliable(NetConnection con, String type, String contents) // targeted

// Binary packets
Call.clientBinaryPacketReliable(String type, byte[] data)
Call.serverBinaryPacketReliable(String type, byte[] data)
```

Handle incoming custom packets via `netServer.addPacketHandler()` — see `references/networking.md`.

## Rules & State

```java
// Push game rules to all clients
Call.setRules(Rules rules)
Call.setRules(NetConnection con, Rules rules)
```

## Reliability: TCP vs UDP

- Methods without `Reliable` in the name use **UDP** (fast, may drop)
- Methods with `Reliable` or that are critical (menus, kicks, messages) use **TCP**
- For labels/effects that should always arrive, use the `Reliable` variant
