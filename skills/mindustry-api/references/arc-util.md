# Arc Utilities

Logging, scheduling, file I/O, HTTP, string handling, commands, color, threading, and geometry utilities from the Arc framework.

## Table of Contents

- [Log](#log)
- [Timer & Time & Interval](#timer--time--interval)
- [Fi — File I/O](#fi--file-io)
- [Http](#http)
- [Strings](#strings)
- [CommandHandler](#commandhandler)
- [Color](#color)
- [Core globals](#core-globals)
- [Threads](#threads)
- [Structs](#structs)
- [Geometry: Point2, Vec2, Rect](#geometry)
- [Pooling](#pooling)

---

## Log

**Import**: `import arc.util.Log;`

Uses `@` as the placeholder character (not `{}` or `%s`).

```java
Log.debug("debug message with @ and @", arg1, arg2);
Log.info("Player @ joined", player.name);
Log.info(anyObject);                    // calls String.valueOf()
Log.infoTag("TAG", "message");          // "[TAG] message"
Log.warn("warning: @", detail);
Log.err("error: @", detail);
Log.err(throwable);                     // prints stack trace
Log.err("context", throwable);          // "context: <stacktrace>"
```

### Log levels

`LogLevel.debug` < `info` < `warn` < `err` < `none`

```java
Log.level = LogLevel.debug;             // show all
Log.useColors = false;                  // disable ANSI codes
```

### ANSI color codes in log messages

Prefix with `&`: `&r` red, `&g` green, `&b` blue, `&y` yellow, `&p` purple, `&c` cyan, `&w` white, `&fr` reset, `&fb` bold. Light variants: `&lr`, `&lg`, `&ly`, `&lb`, `&lc`, `&lm`.

---

## Timer & Time & Interval

### Timer — Real-world scheduling

**Import**: `import arc.util.Timer;`

Tasks execute on the **main game thread** (via `Core.app.post()`). Uses a background daemon thread for timing.

```java
// One-shot after 5 seconds
Timer.schedule(() -> { ... }, 5f);

// Repeating: 1s initial delay, then every 0.5s
Timer.schedule(() -> { ... }, 1f, 0.5f);

// Repeating with limited count (10 repeats after initial)
Timer.schedule(() -> { ... }, 1f, 0.5f, 10);

// Cancel a scheduled task
Timer.Task task = Timer.schedule(() -> { ... }, 5f);
task.cancel();
task.isScheduled();
```

Timer is thread-safe. A `Timer.Task` cannot be scheduled twice simultaneously.

### Time — Game-tick scheduling

**Import**: `import arc.util.Time;`

Based on game ticks (~60/sec). Gets cleared on game reset.

```java
Time.delta          // delta time in ticks (usually ~1.0)
Time.time           // accumulated game time in ticks

// Schedule in game ticks (cleared by Time.clear())
Time.run(300f, () -> { /* after 5 seconds of game time */ });

// Schedule via Timer (survives Time.clear())
Time.runTask(300f, () -> { ... });

// Real clock
Time.millis()
Time.nanos()
Time.timeSinceMillis(prev)

// Constants
Time.toSeconds = 60f     // divide ticks by this → seconds
Time.toMinutes = 3600f   // divide ticks by this → minutes
```

### Interval — Rate limiting

**Import**: `import arc.util.Interval;`

Tick-based rate limiter. Returns true once per interval.

```java
Interval interval = new Interval(3);      // 3 independent timers

// In a per-tick loop:
if (interval.get(0, 300f)) {              // timer 0: every 5 seconds
    // do periodic work
}
if (interval.get(1, 60f)) {               // timer 1: every 1 second
    // different periodic work
}
```

---

## Fi — File I/O

**Import**: `import arc.files.Fi;`

Arc's cross-platform file handle.

### Construction

```java
Fi file = new Fi("/path/to/file");
Fi file = new Fi(javaFile);
Fi file = Fi.get("/path/to/file");
Fi temp = Fi.tempFile("prefix");
Fi tempDir = Fi.tempDirectory("prefix");
```

### Path operations

```java
file.path()                   // forward-slashed path
file.absolutePath()           // absolute path
file.name()                   // filename only
file.nameWithoutExtension()
file.extension()              // without dot
file.parent()                 // parent Fi
file.child("subfile")         // child Fi
file.sibling("other")        // sibling in same dir
```

### Reading

```java
file.readString()             // entire file as UTF-8 String
file.readBytes()              // entire file as byte[]
file.read()                   // raw InputStream
file.reader()                 // BufferedReader (UTF-8)
```

### Writing

```java
file.writeString("text")              // overwrite
file.writeString("text", true)        // append
file.writeBytes(bytes)
file.write()                          // OutputStream (auto-creates parent dirs)
file.writer(false)                    // Writer (append=false)
```

### File system

```java
file.exists()
file.isDirectory()
file.length()
file.lastModified()
file.mkdirs()                 // create parent directories
file.delete()
file.deleteDirectory()        // recursive delete
file.copyTo(destFi)
file.moveTo(destFi)
file.list()                   // children as Fi[]
file.seq()                    // children as Seq<Fi>
file.walk(Cons<Fi>)           // recursive visitor (files only)
```

---

## Http

**Import**: `import arc.util.Http;`

Async HTTP client.

```java
// GET
Http.get("https://api.example.com/data", response -> {
    String body = response.getResultAsString();
    // ⚠️ runs on HTTP thread, NOT main thread
    Core.app.post(() -> processOnMainThread(body));
}, error -> Log.err("Request failed", error));

// POST with builder
Http.post("https://api.example.com/data", "{\"key\":\"value\"}")
    .header("Content-Type", "application/json")
    .timeout(5000)
    .error(Log::err)
    .submit(response -> {
        // handle response on HTTP thread
    });

// Synchronous (blocking)
Http.get("https://example.com")
    .block(response -> {
        // runs on calling thread
    });
```

### HttpResponse

```java
response.getResultAsString()    // body as String (call once)
response.getResult()            // body as byte[] (call once)
response.getStatus()            // HttpStatus enum
response.getHeader("name")      // header value
response.getContentLength()     // content length
```

**Critical**: `submit()` callbacks run on the HTTP thread. Always `Core.app.post()` before touching game state.

---

## Strings

**Import**: `import arc.util.Strings;`

### Formatting

```java
Strings.format("Hello @ from @", name, place)  // @ placeholder
Strings.join(", ", strings)
```

### Number parsing (no exceptions)

```java
Strings.parseInt("42")                 // 42 or Integer.MIN_VALUE on failure
Strings.parseInt("ff", 16, -1)         // parse hex, -1 on failure
Strings.parseFloat("3.14", 0f)         // parse or default
Strings.canParseInt("42")              // true
Strings.canParseFloat("3.14")          // true
```

### Number formatting

```java
Strings.fixed(3.14159f, 2)            // "3.14" (no allocation, shared buffer)
Strings.autoFixed(3.10f, 2)           // "3.1" (auto-trim trailing zeros)
Strings.formatMillis(125000L)          // "0:02:05"
```

### String manipulation

```java
Strings.truncate("long text", 8)              // "long tex"
Strings.truncate("long text", 8, "...")       // "long..."
Strings.capitalize("snake_case")              // "Snake Case"
Strings.stripColors("[accent]Hello[]")        // "Hello" (remove color markup)
Strings.stripGlyphs(text)                    // remove PUA characters
Strings.sanitizeFilename("bad/name")          // "bad_name"
Strings.encode(text)                          // URL encode
```

### String distance

```java
Strings.levenshtein("hello", "helo")          // 1
Strings.biasedLevenshtein("query", "name")    // case-insensitive, prefix-biased
```

### Error formatting

```java
Strings.getStackTrace(throwable)
Strings.neatError(throwable)
Strings.getFinalCause(throwable)
Strings.getFinalMessage(throwable)
```

---

## CommandHandler

**Import**: `import arc.util.CommandHandler;`

Mindustry's command registration and parsing system. Used for both server console and player chat commands.

```java
CommandHandler handler = new CommandHandler("/");  // prefix

// Server command (no player param)
handler.register("status", "Show status", args -> {
    Log.info("Players: @", Groups.player.size());
});

// Command with params
handler.register("kick", "<name> [reason...]", "Kick player", args -> {
    String name = args[0];      // mandatory
    String reason = args.length > 1 ? args[1] : "No reason";
});

// Player command (generic parameter)
handler.<Player>register("tp", "<x> <y>", "Teleport", (args, player) -> {
    float x = Strings.parseFloat(args[0], -1);
    float y = Strings.parseFloat(args[1], -1);
    player.unit().set(x, y);
});

// Remove a command
handler.removeCommand("help");

// Get all commands
Seq<CommandHandler.Command> commands = handler.getCommandList();
```

### Parameter syntax

- `<param>` — mandatory
- `[param]` — optional
- `<param...>` / `[param...]` — variadic (captures remaining text, must be last)
- All mandatory params must come before optional ones

### Response types

When using `handleMessage()` directly:
- `noCommand` — doesn't start with prefix
- `unknownCommand` — prefix present, command not found
- `fewArguments` — not enough mandatory args
- `manyArguments` — too many args
- `valid` — command matched and executed

---

## Color

**Import**: `import arc.graphics.Color;`

Mutable color with `float r, g, b, a` components in [0, 1].

```java
new Color(1f, 0f, 0f, 1f)       // red
Color.valueOf("ff0000")          // hex parsing (RRGGBB or RRGGBBAA)
Color.valueOf("#ff0000")         // # prefix optional
Color.rgb(255, 0, 0)            // from 0-255 ints

// Named constants
Color.white, Color.black, Color.red, Color.green, Color.blue,
Color.yellow, Color.cyan, Color.scarlet, Color.royal, Color.sky,
Color.gold, Color.orange, Color.coral, Color.forest, Color.gray

// Methods
color.cpy()                      // create copy (Color is mutable!)
color.toString()                 // "rrggbbaa" hex
color.rgba()                     // pack to RGBA8888 int
color.set(r, g, b, a)           // mutate
color.lerp(other, t)             // interpolate (mutating)
```

### Mindustry color markup (in chat/menus)

```
[accent]colored text[]       — reset color
[scarlet]red text[]
[green]green text[]
[#ff8800]custom hex[]
[[literal bracket]           — escape bracket with [[
```

---

## Core globals

**Import**: `import arc.Core;`

```java
Core.app.post(() -> { ... })     // dispatch to main thread (thread-safe)
Core.app.exit()                  // shutdown
Core.app.isHeadless()            // true on server
Core.app.getJavaHeap()           // memory usage

Core.settings.getString("key")   // persistent settings
Core.settings.put("key", value)  // save setting
Core.settings.has("key")         // check existence
Core.settings.getDataDirectory() // data folder Fi

Core.files                       // platform file access
```

---

## Threads

**Import**: `import arc.util.Threads;`

```java
Threads.thread("name", () -> { ... })       // non-daemon thread
Threads.daemon("name", () -> { ... })       // daemon thread
Threads.sleep(millis)                        // non-throwing sleep
Threads.executor("name", count)              // fixed thread pool
Threads.unboundedExecutor("name", min)       // auto-scaling pool
Threads.await(executorService)               // shutdown and wait
Threads.throwAppException(throwable)         // post exception to main thread
```

---

## Structs

**Import**: `import arc.util.Structs;`

Static utility methods for arrays:

```java
Structs.arr(a, b, c)                   // create typed array
Structs.contains(arr, item)            // search
Structs.contains(arr, Boolf)           // predicate search
Structs.find(arr, Boolf)               // first match or null
Structs.indexOf(arr, Boolf)            // index or -1
Structs.random(arr)                    // random element
Structs.each(Cons, items...)           // iterate varargs

// Comparator factories
Structs.comparing(Func)                // key extractor → Comparator
Structs.comparingFloat(Floatf)
Structs.comparingInt(Intf)
Structs.comps(comp1, comp2)            // chain comparators

// Min/max finders
Structs.findMin(arr, Comparator)
Structs.findMin(iterable, Floatf)

// Bounds
Structs.inBounds(x, y, width, height)
Structs.eq(a, b)                       // null-safe equals
```

---

## Geometry

### Point2 — Integer grid coordinates

**Import**: `import arc.math.geom.Point2;`

```java
Point2.pack(x, y)         // pack two shorts into one int
Point2.x(packed)           // extract x from packed int
Point2.y(packed)           // extract y from packed int
Point2.unpack(packed)      // → new Point2

new Point2(x, y)
point.x, point.y           // int fields
```

### Vec2 — Float 2D vector

**Import**: `import arc.math.geom.Vec2;`

```java
new Vec2(x, y)
vec.add(other), vec.sub(other), vec.scl(factor)
vec.len(), vec.nor()       // length, normalize
vec.angle(), vec.rotate(degrees)
vec.dst(other)             // distance
vec.lerp(target, alpha)    // interpolate
vec.set(x, y)
vec.trns(angle, length)    // set from polar
```

### Other

- `Rect` — axis-aligned rectangle (`contains`, `overlaps`, `merge`)
- `Geometry.d4x/d4y` — directional arrays (right, up, left, down)
- `Mathf` — math utilities (`clamp`, `lerp`, `random`, `dst`, `within`, `ceil`, `floor`, `mod`, etc.)

---

## Pooling

**Import**: `import arc.util.pooling.Pools;`

```java
// Obtain from pool (or create new)
MyObj obj = Pools.obtain(MyObj.class, MyObj::new);

// Return to pool
Pools.free(obj);  // calls reset() if Poolable

// Poolable interface
class MyObj implements Pool.Poolable {
    @Override
    public void reset() { /* clear state */ }
}
```
