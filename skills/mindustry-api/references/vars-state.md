# Vars, GameState, Rules, Content, Maps, World & Tiles

## Vars — Global State Container

`mindustry.Vars` holds all runtime state as `public static` fields. Most plugin code imports it statically:

```java
import static mindustry.Vars.*;
```

### Key fields

| Field | Type | Description |
|-------|------|-------------|
| `headless` | `boolean` | `true` on dedicated servers |
| `net` | `Net` | Network manager |
| `netServer` | `NetServer` | Server-side networking (see `networking.md`) |
| `state` | `GameState` | Current game state |
| `world` | `World` | Tile grid |
| `content` | `ContentLoader` | All loaded content |
| `mods` | `Mods` | Mod/plugin manager |
| `maps` | `Maps` | Map management |
| `player` | `Player` | Local player (**null on headless**) |
| `logic` | `Logic` | Game logic handler |
| `spawner` | `WaveSpawner` | Wave spawning |
| `indexer` | `BlockIndexer` | Spatial block indexing |
| `pathfinder` | `Pathfinder` | Unit pathfinding |
| `dataDirectory` | `Fi` | Root data directory |
| `modDirectory` | `Fi` | Mods directory |
| `saveDirectory` | `Fi` | Saves directory |
| `customMapDirectory` | `Fi` | Custom maps directory |

### Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `port` | 6567 | Default server port |
| `maxTextLength` | 150 | Max chat message length |
| `tilesize` | 8 | Tile size in world units |

---

## GameState (`Vars.state`)

```java
state.wave               // current wave number (int)
state.wavetime           // time until next wave (float, ticks)
state.gameOver           // is game over (boolean)
state.rules              // Rules object
state.map                // current Map
state.teams              // Teams data

state.isPaused()         // is paused
state.isPlaying()        // is actively playing
state.isGame()           // not in menu
state.isMenu()           // in menu state
state.set(State.playing) // change state
state.isCampaign()       // is campaign mode
```

### State enum

`State.playing`, `State.paused`, `State.menu`

---

## Rules (`state.rules`)

Rules control game behavior. Dozens of boolean toggles and float multipliers:

### Key booleans

| Rule | Default | Description |
|------|---------|-------------|
| `waves` | true | Enable waves |
| `pvp` | false | PvP mode |
| `attackMode` | false | Attack mode |
| `infiniteResources` | false | Unlimited resources |
| `canGameOver` | true | Allow game over |
| `coreCapture` | false | Cores can be captured |
| `ghostBlocks` | true | Show ghost blocks while building |
| `fire` | true | Enable fire spread |
| `unitAmmo` | false | Units consume ammo |
| `schematicsAllowed` | true | Allow schematics |
| `damageExplosions` | true | Damage causes explosions |
| `reactorExplosions` | true | Reactors can explode |
| `logicUnitBuild` | true | Logic processors can build units |

### Key multipliers (float)

| Rule | Default | Description |
|------|---------|-------------|
| `unitDamageMultiplier` | 1.0 | Unit damage scale |
| `unitHealthMultiplier` | 1.0 | Unit health scale |
| `buildSpeedMultiplier` | 1.0 | Build speed scale |
| `blockDamageMultiplier` | 1.0 | Block damage scale |
| `blockHealthMultiplier` | 1.0 | Block health scale |
| `solarMultiplier` | 1.0 | Solar panel output scale |

### Team-specific rules

```java
// Access per-team rules
Rules.TeamRule teamRule = state.rules.teams.get(Team.sharded);
teamRule.buildSpeedMultiplier = 2f;
teamRule.unitDamageMultiplier = 1.5f;
teamRule.infiniteResources = true;
```

### Pushing rules to clients

After modifying rules server-side, push them:
```java
Call.setRules(state.rules);
```

---

## Team

```java
// Built-in teams
Team.sharded    // default player team (orange, id=1)
Team.derelict   // neutral/abandoned (id=0)
Team.crux       // enemy team (red, id=2)
Team.malis      // (id=3)
Team.green      // (id=4)
Team.blue       // (id=5)

Team.all        // Team[256] — all possible teams
Team.get(id)    // get team by ID (0-255)

// Team instance methods
team.cores()         // Seq<CoreBuild> — all cores
team.core()          // first core (nullable)
team.data()          // TeamData
team.items()         // core item inventory
team.active()        // has buildings
team.isAI()          // AI-controlled
team.color           // Color
team.id              // int (0-255)
```

---

## Content (`Vars.content`)

Access all loaded game content:

```java
content.blocks()          // Seq<Block>
content.items()           // Seq<Item>
content.liquids()         // Seq<Liquid>
content.units()           // Seq<UnitType>
content.statusEffects()   // Seq<StatusEffect>
content.block(id)         // Block by numeric ID
content.item(id)          // Item by numeric ID
```

### Static content references

```java
// Blocks
Blocks.copperWall, Blocks.router, Blocks.coreShard, Blocks.coreNucleus ...

// Items
Items.copper, Items.lead, Items.silicon, Items.titanium, Items.thorium ...

// Liquids
Liquids.water, Liquids.slag, Liquids.oil, Liquids.cryofluid ...

// Unit types
UnitTypes.dagger, UnitTypes.flare, UnitTypes.mono, UnitTypes.poly ...

// Status effects
StatusEffects.burning, StatusEffects.wet, StatusEffects.freezing ...
```

### ContentType enum

`item`, `block`, `bullet`, `liquid`, `status`, `unit`, `weather`, `sector`, `planet`, `team`, `unitCommand`, `unitStance`

---

## Maps (`Vars.maps`)

```java
maps.all()                      // Seq<Map> — all available maps
maps.customMaps()               // custom maps only
maps.getNextMap(mode, prev)     // next map for rotation
maps.setNextMapOverride(map)    // force next map
maps.setShuffleMode(mode)       // set shuffle mode
maps.setMapProvider(provider)   // custom map selection logic
```

---

## World (`Vars.world`)

```java
world.width()              // map width in tiles (int)
world.height()             // map height in tiles (int)
world.tile(x, y)           // get Tile (nullable if out of bounds)
world.tileWorld(wx, wy)    // get Tile at world coordinates (wx/tilesize)
world.tiles                // Tiles grid (width * height)
world.solid(x, y)          // is tile solid
world.passable(x, y)       // is tile passable
```

---

## Tile

```java
tile.x, tile.y             // tile coordinates (short)
tile.pos()                 // packed position via Point2.pack(x, y)
tile.block()               // the Block on this tile
tile.floor()               // the Floor
tile.overlay()             // the overlay (ore, etc.)
tile.build                 // the Building entity (nullable)
tile.solid()               // is solid
tile.passable()            // is passable

// Mutation
tile.setBlock(Block, Team, int rotation)
tile.setFloor(Floor)
tile.setOverlay(Block)

// Coordinate conversion
tile.getX()                // world X = tile.x * tilesize
tile.getY()                // world Y = tile.y * tilesize

// Neighbors
tile.nearby(0..3)          // adjacent tiles (4 directions)
```

### World ↔ Tile coordinate conversion

```java
// tile coords → world coords
float worldX = tileX * tilesize;
float worldY = tileY * tilesize;

// world coords → tile coords
int tileX = World.toTile(worldX);  // or (int)(worldX / tilesize)
int tileY = World.toTile(worldY);
```
