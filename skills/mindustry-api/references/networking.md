# NetServer, Net, NetConnection, Administration

## NetServer (`Vars.netServer`)

The server-side networking brain. Handles connections, commands, chat, admin actions.

**Import**: `import mindustry.core.NetServer;`

### Key fields

```java
netServer.admins          // Administration — bans, kicks, filters, config
netServer.clientCommands   // CommandHandler — registered /commands (prefix "/")
netServer.assigner         // TeamAssigner — replaceable team assignment logic
netServer.chatFormatter    // ChatFormatter — replaceable chat formatting
netServer.invalidHandler   // InvalidCommandHandler — custom invalid command response
netServer.currentlyKicking // VoteSession — active votekick (nullable)
```

### Custom packet handlers

```java
// Handle string packets from client mods
netServer.addPacketHandler("myType", (player, contents) -> {
    Log.info("Got packet from @: @", player.name, contents);
});

// Handle binary packets
netServer.addBinaryPacketHandler("myType", (player, bytes) -> {
    // process binary data
});

// Handle logic data channel messages
netServer.addLogicDataHandler("channel", (player, value) -> {
    // value from setprop in logic processors
});
```

### Chat formatting

```java
// Replace the default chat format
netServer.chatFormatter = (player, message) -> {
    if (player == null) return message; // system message
    return "[accent][" + player.coloredName() + "[]]: " + message;
};
```

### Team assignment

```java
netServer.assigner = (player, players) -> {
    // return the Team this player should join
    return Team.sharded;
};
```

---

## Net (`Vars.net`)

Lower-level network manager.

```java
net.server()              // is this instance a server? (boolean)
net.client()              // is this instance a client?
net.active()              // is networking active?
net.host(port)            // start hosting
net.closeServer()         // stop hosting
net.getConnections()      // Iterable<NetConnection> — all client connections
net.send(object, reliable)      // send packet to all
net.sendExcept(con, object, reliable)  // send to all except one

// Register a server-side packet handler
net.handleServer(MyPacket.class, (con, packet) -> { ... });

// IP-level connection filter (before handshake)
net.setConnectFilter((address) -> {
    // return true to allow, false to reject
    return !isBanned(address);
});
```

---

## NetConnection

Represents a single client connection.

```java
con.address          // IP address (String)
con.uuid             // player UUID (String)
con.usid             // player USID (String)
con.player           // associated Player (nullable until fully connected)
con.mobile           // is mobile client (boolean)
con.modclient        // is modded client (boolean)
con.kicked           // was kicked (boolean)
con.hasConnected     // completed full connection (boolean)

con.kick(KickReason.kick)       // kick with standard reason
con.kick("Custom message")      // kick with custom message
con.kick("message", 60_000L)    // kick with temporary ban duration (ms)
con.send(object, reliable)      // send packet
con.isConnected()               // still connected
```

### KickReason enum

`kick`, `clientOutdated`, `serverOutdated`, `banned`, `gameover`, `recentKick`, `nameInUse`, `nameEmpty`, `customClient`, `serverClose`, `vote`, `typeMismatch`, `whitelist`, `playerLimit`, `serverRestarting`

---

## Administration (`Vars.netServer.admins`)

Player management, bans, filters, server configuration.

### Banning

```java
admins.banPlayerID(uuid)       // ban by UUID
admins.banPlayerIP(ip)         // ban by IP
admins.unbanPlayerID(uuid)     // unban by UUID
admins.unbanPlayerIP(ip)       // unban by IP
admins.isIDBanned(uuid)        // check UUID ban
admins.isIPBanned(ip)          // check IP ban
admins.isSubnetBanned(ip)      // check subnet ban
admins.getBanned()             // Seq<PlayerInfo> — all banned players
```

### Admin management

```java
admins.adminPlayer(uuid, usid)   // make admin
admins.unAdminPlayer(uuid)       // remove admin
admins.isAdmin(uuid, usid)       // check admin status
admins.getAdmins()               // Seq<PlayerInfo> — all admins
```

### Player lookup

```java
admins.getInfo(uuid)           // PlayerInfo by UUID
admins.findByName(name)        // Seq<PlayerInfo> — search by name
admins.getInfoOptional(uuid)   // PlayerInfo or null (no create)
```

### Kick cooldowns

```java
admins.handleKicked(uuid, ip, durationMs)  // set kick cooldown
admins.getKickTime(uuid, ip)               // remaining kick time (ms)
```

### Chat filters

Filters transform or block chat messages before they're broadcast:

```java
admins.addChatFilter((player, message) -> {
    if (message.contains("spam")) return null;  // block message
    return message.replace("badword", "***");    // modify message
});

// Apply all registered filters
String filtered = admins.filterMessage(player, rawMessage);
// null = message was blocked
```

### Action filters

Action filters prevent player actions (building, breaking, configuring, etc.):

```java
admins.addActionFilter(action -> {
    // action.player — who
    // action.type   — ActionType enum
    // action.tile   — target tile
    // action.block  — block involved (nullable)
    // action.item   — item involved (nullable)

    if (action.type == ActionType.breakBlock && !action.player.admin) {
        return false; // deny
    }
    return true; // allow
});
```

### ActionType enum

`breakBlock`, `placeBlock`, `rotate`, `configure`, `withdrawItem`, `depositItem`, `control`, `buildSelect`, `command`, `removePlanned`, `commandUnits`, `commandBuilding`, `respawn`, `pickupBlock`, `dropPayload`, `pingLocation`

### PlayerInfo

Persisted player data (survives restarts):

```java
info.id              // UUID (String)
info.lastName        // last known name
info.lastIP          // last IP address
info.names           // Seq<String> — all known names
info.ips             // Seq<String> — all known IPs
info.banned          // ban status (boolean)
info.admin           // admin status (boolean)
info.timesJoined     // join count (int)
info.timesKicked     // kick count (int)
info.lastKicked      // kick expiry timestamp (long)
```

### Server Config (`Administration.Config`)

Built-in server settings (accessible from console and code):

```java
Config.serverName.string()      // server name
Config.motd.string()            // message of the day
Config.desc.string()            // server description
Config.port.num()               // server port
Config.enableVotekick.bool()    // votekick enabled
Config.antiSpam.bool()          // anti-spam
Config.strict.bool()            // strict mode
Config.whitelist.bool()         // whitelist enabled
Config.autoPause.bool()         // auto-pause when empty

// Create custom plugin config (shows in console `config` command)
new Config("myPluginSetting", "Description of this setting", defaultValue);
```

## Patterns

### Connection validation pipeline

```java
// Layer 1: IP filter (earliest, before handshake)
net.setConnectFilter(address -> !isBlacklisted(address));

// Layer 2: Connect packet handler (after handshake, before world data)
Events.on(ConnectPacketEvent.class, e -> {
    if (!validateClient(e.packet)) {
        e.connection.kick("Invalid client");
    }
});

// Layer 3: Chat/action filters (during gameplay)
admins.addChatFilter((player, msg) -> sanitize(msg));
admins.addActionFilter(action -> isAllowed(action));
```
