# Player, Groups, EntityGroup

## Player (Generated)

`Player` is a generated entity class (`mindustry.gen.Player`) created from `PlayerComp.java` by the annotation processor. It represents a connected player.

**Import**: `import mindustry.gen.Player;`

### Identity & metadata

```java
player.name              // display name (String, may contain color codes)
player.admin             // is admin (boolean)
player.locale            // client locale string (e.g. "en", "uk")
player.color             // player Color
player.team()            // current Team
player.team(Team.crux)   // set team
```

### Connection

```java
player.con               // NetConnection (null for local player on non-headless)
player.uuid()            // UUID string (from con.uuid)
player.usid()            // USID string (from con.usid)
player.ip()              // IP address string
player.getInfo()         // Administration.PlayerInfo (persisted data)
player.isLocal()         // is the local player (false on headless servers)
player.isAdded()         // in the entity system
player.id()              // entity ID (int)
```

### Position & state

```java
player.x, player.y             // world position (float)
player.mouseX, player.mouseY   // mouse position
player.shooting                // is shooting (boolean)
player.boosting                // is boosting (boolean)
player.typing                  // is typing in chat (boolean)
```

### Unit control

```java
player.unit()            // controlled Unit (may return a "null" unit, check .isNull())
player.dead()            // has no valid unit
player.isBuilder()       // can build
```

### Cores

```java
player.core()            // player's team core (nullable)
player.closestCore()     // closest core
player.bestCore()        // best/largest core
```

### Location

```java
player.tileOn()          // Tile at player position (nullable)
player.buildOn()         // Building at player position (nullable)
```

### Communication

```java
player.sendMessage(text)                       // send chat message
player.sendMessage(text, from)                 // with sender Player
player.sendMessage(text, from, unformatted)    // formatted + raw text
```

### Name utilities

```java
player.coloredName()     // "[#RRGGBB]name" with player color
player.plainName()       // name stripped of color codes
```

### Kick

```java
player.kick(KickReason.kick)       // kick with enum reason
player.kick("Custom reason")       // kick with message
player.kick("reason", 60_000L)     // kick with temp ban duration (ms)
```

---

## Groups (Generated)

`Groups` (`mindustry.gen.Groups`) provides typed **entity groups** — live collections of all entities of each type. Generated from `GroupDefs.java`.

**Import**: `import mindustry.gen.Groups;`

### Available groups

| Group | Element Type | Spatial | Mapped | Description |
|-------|-------------|---------|--------|-------------|
| `Groups.all` | `Entityc` | No | No | All entities |
| `Groups.player` | `Player` | No | Yes (ID) | All connected players |
| `Groups.unit` | `Unit` | Yes (QuadTree) | Yes | All units |
| `Groups.build` | `Building` | No | No | All buildings |
| `Groups.bullet` | `Bullet` | Yes (QuadTree) | No | All bullets |
| `Groups.sync` | `Syncc` | No | Yes | All synced entities |
| `Groups.draw` | `Drawc` | No | No | All drawable entities |
| `Groups.fire` | `Fire` | No | No | All fires |
| `Groups.puddle` | `Puddle` | No | No | All puddles |
| `Groups.weather` | `WeatherState` | No | No | Weather states |
| `Groups.label` | `WorldLabel` | No | Yes | World labels |

### EntityGroup API

```java
// Count
Groups.player.size()

// Iterate all
Groups.player.each(p -> {
    p.sendMessage("Hello!");
});

// Iterate with filter
Groups.player.each(
    p -> p.team() == Team.sharded,  // filter
    p -> p.sendMessage("Team msg")   // action
);

// Find first matching
Player found = Groups.player.find(p -> p.name.equals("target"));

// Check existence
boolean exists = Groups.player.contains(p -> p.admin);

// Count matching
int count = Groups.player.count(p -> p.team() == Team.sharded);

// Snapshot to Seq (allocates a copy)
Seq<Player> snapshot = Groups.player.copy();

// Lookup by entity ID (mapped groups only)
Player p = Groups.player.getByID(entityId);

// Spatial query (spatial groups only)
Groups.unit.intersect(x, y, width, height, unit -> {
    // all units in this rectangle
});

// Check empty
boolean empty = Groups.player.isEmpty();
```

## Common patterns

### Broadcast to all players

```java
Groups.player.each(p -> p.sendMessage("Server restarting in 5 minutes"));
```

### Find player by name (fuzzy)

```java
Player target = Groups.player.find(p ->
    Strings.stripColors(p.name).equalsIgnoreCase(targetName)
);
```

### Count players per team

```java
int shardedCount = Groups.player.count(p -> p.team() == Team.sharded);
int cruxCount = Groups.player.count(p -> p.team() == Team.crux);
```

### Iterate units in an area

```java
float x = player.x, y = player.y;
float radius = 100f;
Groups.unit.intersect(x - radius, y - radius, radius * 2, radius * 2, unit -> {
    if (unit.within(player, radius)) {
        // unit is within circular radius
    }
});
```

### Get all buildings of a type

```java
Seq<Building> turrets = new Seq<>();
Groups.build.each(b -> b.block == Blocks.cyclone, turrets::add);
```

## Unit (Generated)

`Unit` (`mindustry.gen.Unit`) is the generated entity for all units.

```java
unit.type              // UnitType
unit.team              // Team
unit.x, unit.y         // position
unit.health            // current health
unit.maxHealth         // max health
unit.dead()            // is dead
unit.kill()            // destroy this unit
unit.heal()            // heal to full
unit.heal(amount)      // heal by amount
unit.damage(amount)    // deal damage
unit.isPlayer()        // is controlled by a player
unit.getPlayer()       // controlling Player (nullable)
unit.isNull()          // is the "null" placeholder unit
unit.within(x, y, r)  // is within radius of point
unit.within(pos, r)    // is within radius of Position
unit.dst(x, y)         // distance to point
unit.dst(pos)          // distance to Position
```

## Building (Generated)

`Building` (`mindustry.gen.Building`) is the generated entity for placed blocks.

```java
building.block         // Block type
building.team          // Team
building.tile          // the Tile this building is on
building.x, building.y // world position
building.health        // current health
building.maxHealth     // max health
building.items         // ItemModule (nullable, for blocks with inventory)
building.liquids       // LiquidModule (nullable)
building.power         // PowerModule (nullable)
building.kill()        // destroy
building.damage(amt)   // deal damage
building.heal()        // heal to full
building.configure(v)  // set configuration value
```
