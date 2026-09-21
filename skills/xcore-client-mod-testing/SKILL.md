---
name: xcore-client-mod-testing
description: >
  Standards and automated workflows for developing, debugging, visually verifying, and testing Mindustry v160 client mods
  (e.g., AdminTools). Covers the DevHarness pattern, zero-overhead release stripping, automated in-engine framebuffer screenshots
  via ScreenUtils, responsive device emulation (mobile portrait, mobile landscape, desktop), and Gradle property forwarding.
  Use whenever modifying client mod UI/UX, verifying visual layouts, adding mock data for testing, configuring headless or automated
  screenshots, or stripping debug code for release builds.
---

# Mindustry Client Mod Testing & DevHarness Architecture

This skill defines the standard engineering workflow for testing and visually verifying Mindustry v160 client mods (such as `AdminTools`). It eliminates manual sleep-and-kill scripts, compositor dependencies, and lingering processes by providing an automated, in-engine testing harness.

---

## 1. Core Principles

1. **Zero Runtime Overhead for Players**:
   - Production code MUST NOT unconditionally execute or instantiate developer tools.
   - The developer harness MUST be loaded via dynamic reflection (`Class.forName(...)`), so the class is not even touched by the JVM classloader during normal player gameplay.
2. **Physical Release Stripping**:
   - Release builds (`./gradlew build -Prelease`) MUST exclude all dev packages (`**/dev/**`) from the output JAR.
   - Because the main mod class uses dynamic reflection, missing dev classes in release builds fail silently without breaking the mod.
3. **Automated In-Engine Visual Verification**:
   - Never rely on external window capture tools (`niri`, `wayshot`, `import`) or arbitrary `sleep` intervals.
   - Use Arc's native `ScreenUtils.saveScreenshot(Fi)` attached to `EventType.Trigger.uiDrawEnd` to capture the OpenGL framebuffer directly once layout and animations settle.
   - The test run MUST terminate cleanly (`Core.app.exit()`) with exit code `0`.

---

## 2. Quick Execution Guide (For Coding Agents & Developers)

All testing and visual captures are run using single-line Gradle commands from the mod root:

### Capture Specific Screen in Mobile Portrait (480x800)

```bash
DISPLAY=":1" WAYLAND_DISPLAY="wayland-1" XDG_RUNTIME_DIR="/run/user/1000" NIRI_SOCKET="/run/user/1000/niri.wayland-1.1064.sock" \
    ./gradlew runMindustryDesktop -Pscreen=karma -Pmobile=portrait -Pscreenshot=/tmp/karma_mobile.png
```

### Capture Specific Screen in Mobile Landscape (800x420)

```bash
DISPLAY=":1" WAYLAND_DISPLAY="wayland-1" XDG_RUNTIME_DIR="/run/user/1000" NIRI_SOCKET="/run/user/1000/niri.wayland-1.1064.sock" \
    ./gradlew runMindustryDesktop -Pscreen=history -Pmobile=landscape -Pscreenshot=/tmp/history_landscape.png
```

### Capture Specific Screen in Desktop Resolution

```bash
DISPLAY=":1" WAYLAND_DISPLAY="wayland-1" XDG_RUNTIME_DIR="/run/user/1000" NIRI_SOCKET="/run/user/1000/niri.wayland-1.1064.sock" \
    ./gradlew runMindustryDesktop -Pscreen=portal -Pmobile=false -Pscreenshot=/tmp/portal_desktop.png
```

### Supported Screen Keys (`-Pscreen`)

- `karma`: Injects mock griefers, builders, and suspicious players, then opens the Karma Detector window.
- `history`: Injects realistic tile action entries (break, build, rotate, config) and opens Tile History.
- `portal`: Opens the server portal window with mock/live server entries.
- `ban`: Opens the modal BanDialog with target player JSON.
- `auth`: Opens the AuthDialog.
- `dock_collapsed`: Toggles and displays QuickDock in its collapsed state (`[ ✥ | > ]`).
- `dock_expanded`: Displays QuickDock in its fully expanded state (`[ ✥ | buttons | < ]`).
- `freecam`: Toggles and displays the FreeCam floating status HUD.
- `all`: Opens all primary floating windows positioned across the canvas.

### Supported Parameters

| Property | Gradle Flag | Description | Default |
| --- | --- | --- | --- |
| Screen | `-Pscreen=<name>` | Component/window to mock and display | `""` |
| Device Mode | `-Pmobile=portrait\|landscape\|false` | Sets `Vars.mobile` and resizes window | `portrait` |
| Custom Size | `-Pwidth=W -Pheight=H` | Overrides window dimensions | `0` |
| Screenshot Path | `-Pscreenshot=/path/to.png` | Framebuffer target path | `""` |
| Capture Delay | `-Pdelay=<ticks>` | Ticks to wait after applying screen for animations/layout | `30` |
| Auto-Exit | `-Pexit=true\|false` | Closes game client after capture | `true` |
| Interactive Dev | `-Pdev=true` | Registers in-game `F8` dev control dialog | `false` |

---

## 3. Architecture & Implementation Pattern

### A. Dynamic Hook in Main Mod Class (`AdminTools.java`)

Place this hook inside `ClientLoadEvent` handler. It ensures zero overhead when running without dev flags:

```java
Events.on(EventType.ClientLoadEvent.class, e -> {
    ui = new UIController();

    // Dev harness: dynamically loaded via reflection only when explicit dev flags are present
    if (System.getProperty("admintools.screen") != null
        || System.getProperty("admintools.dev") != null
        || System.getProperty("admintools.screenshot") != null) {
        try {
            Class.forName("admintools.dev.DevHarness")
                .getMethod("init", UIController.class)
                .invoke(null, ui);
        } catch (Throwable ignored) {
            // Silently ignored if dev classes were stripped in release JAR
        }
    }
    
    // ... remaining mod initialization ...
});
```

### B. Framebuffer Screenshot Capture in DevHarness

Always attach screenshot capture to `EventType.Trigger.uiDrawEnd` after a stabilization delay:

```java
public static void scheduleScreenshot(String outputPath, int delayTicks, boolean autoExit) {
    boolean[] captured = {false};
    boolean[] ready = {false};

    Events.run(EventType.Trigger.uiDrawEnd, () -> {
        if (!ready[0] || captured[0]) return;
        captured[0] = true;

        try {
            Fi file = Fi.get(outputPath);
            file.parent().mkdirs();
            ScreenUtils.saveScreenshot(file);
            Log.info("[DevHarness] Screenshot successfully captured to: @", file.absolutePath());
        } catch (Throwable t) {
            Log.err("[DevHarness] Failed to capture screenshot", t);
        } finally {
            if (autoExit) {
                Time.runTask(5f, () -> Core.app.exit());
            }
        }
    });

    Time.runTask(delayTicks, () -> ready[0] = true);
}
```

### C. Gradle Build Configuration (`build.gradle.kts`)

Configure `Jar` task to exclude dev classes when `-Prelease` is passed, and forward CLI properties to `runMindustryDesktop`:

```kotlin
val isRelease = project.hasProperty("release")

tasks {
    withType<Jar> {
        duplicatesStrategy = DuplicatesStrategy.EXCLUDE
        from(rootDir) {
            include("mod.json")
            include("bundles/**")
        }
        from({
            configurations.runtimeClasspath.get().map { if (it.isDirectory) it else zipTree(it) }
        })
        
        // Strip developer harness and mock files for release builds
        if (isRelease) {
            exclude("admintools/dev/**")
        }
    }

    named<JavaExec>("runMindustryDesktop") {
        listOf("screen", "mobile", "mobile.landscape", "screenshot", "width", "height", "delay", "exit", "dev").forEach { key ->
            val prop = "admintools.$key"
            if (System.getProperty(prop) != null) {
                systemProperty(prop, System.getProperty(prop))
            } else if (project.hasProperty(key)) {
                systemProperty(prop, project.property(key).toString())
            }
        }
    }
}
```

---

## 4. Building Production Releases

To create an Android- and Desktop-compatible release JAR with all developer tools cleanly stripped:

```bash
./gradlew build -Prelease
```

Verify stripping by inspecting the generated JAR:

```bash
python3 -c '
import zipfile
with zipfile.ZipFile("build/libs/AdminTools.jar") as z:
    devs = [f for f in z.namelist() if "dev" in f]
    assert len(devs) == 0, f"Found dev files in release: {devs}"
print("✅ Verified: release JAR is completely clean of dev tools.")
'
```

---

## 5. In-Game Developer Control Panel (`F8`)

When running locally with `-Pdev=true`, the harness hooks the `F8` keyboard event. Pressing `F8` opens a floating dialog allowing the developer to:

- Instantly switch or trigger any modal dialog or floating window.
- Manually trigger a framebuffer capture saved to `/tmp/admintools_dev_<timestamp>.png`.
- Toggle dock collapse and FreeCam states live during play.
