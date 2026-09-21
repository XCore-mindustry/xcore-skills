# DevHarness Reference Template

This file provides a complete, copy-pasteable reference template for implementing `DevHarness` in any Mindustry v160 client mod.

```java
package admintools.dev;

import arc.Core;
import arc.Events;
import arc.files.Fi;
import arc.input.KeyCode;
import arc.scene.event.InputEvent;
import arc.scene.event.InputListener;
import arc.scene.ui.Dialog;
import arc.util.Log;
import arc.util.ScreenUtils;
import arc.util.Time;
import mindustry.Vars;
import mindustry.game.EventType;
import mindustry.gen.Icon;

/**
 * Universal developer and automated test harness for Mindustry client mods.
 */
public class DevHarness {

    private static boolean initialized = false;

    public static void init(Object controller) {
        if (initialized) return;
        initialized = true;

        Log.info("[DevHarness] Initializing Mindustry mod developer harness...");

        configureDisplay();
        setupKeyListeners();

        String screen = System.getProperty("admintools.screen");
        String screenshotPath = System.getProperty("admintools.screenshot");

        if (screen != null && !screen.isEmpty()) {
            Time.runTask(20f, () -> {
                applyScreen(screen);

                if (screenshotPath != null && !screenshotPath.isEmpty()) {
                    int delay = Integer.getInteger("admintools.delay", 30);
                    boolean autoExit = !"false".equalsIgnoreCase(System.getProperty("admintools.exit", "true"));
                    scheduleScreenshot(screenshotPath, delay, autoExit);
                }
            });
        } else if (screenshotPath != null && !screenshotPath.isEmpty()) {
            int delay = Integer.getInteger("admintools.delay", 30);
            boolean autoExit = !"false".equalsIgnoreCase(System.getProperty("admintools.exit", "true"));
            scheduleScreenshot(screenshotPath, delay, autoExit);
        }
    }

    private static void configureDisplay() {
        String mobileProp = System.getProperty("admintools.mobile");
        if (mobileProp != null) {
            boolean isLandscape = "landscape".equalsIgnoreCase(mobileProp) || Boolean.getBoolean("admintools.mobile.landscape");
            boolean isMobile = "true".equalsIgnoreCase(mobileProp) || "portrait".equalsIgnoreCase(mobileProp) || isLandscape;

            if (isMobile) {
                Vars.mobile = true;
                Core.app.post(() -> {
                    if (isLandscape) {
                        Core.graphics.setWindowSize(800, 420);
                    } else {
                        Core.graphics.setWindowSize(480, 800);
                    }
                });
            }
        }

        int customWidth = Integer.getInteger("admintools.width", 0);
        int customHeight = Integer.getInteger("admintools.height", 0);
        if (customWidth > 0 && customHeight > 0) {
            Core.app.post(() -> Core.graphics.setWindowSize(customWidth, customHeight));
        }
    }

    private static void setupKeyListeners() {
        Core.scene.addListener(new InputListener() {
            @Override
            public boolean keyDown(InputEvent event, KeyCode keycode) {
                if (keycode == KeyCode.f8) {
                    showDevMenu();
                    return true;
                }
                return false;
            }
        });
    }

    public static void applyScreen(String screen) {
        Log.info("[DevHarness] Applying mock screen: @", screen);
        // Custom screen mocking logic here
    }

    public static void scheduleScreenshot(String outputPath, int delayTicks, boolean autoExit) {
        Log.info("[DevHarness] Scheduling automated screenshot in @ ticks to: @", delayTicks, outputPath);

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
                    Log.info("[DevHarness] Auto-exit requested. Terminating game client.");
                    Time.runTask(5f, () -> Core.app.exit());
                }
            }
        });

        Time.runTask(delayTicks, () -> ready[0] = true);
    }

    public static void showDevMenu() {
        Dialog dialog = new Dialog("Mod Dev Menu");
        dialog.cont.table(t -> {
            t.defaults().size(220f, 40f).pad(4f);
            t.button("Capture Screenshot", Icon.save, () -> {
                String path = "/tmp/mod_dev_" + System.currentTimeMillis() + ".png";
                scheduleScreenshot(path, 2, false);
            }).row();
        });
        dialog.addCloseButton();
        dialog.show();
    }
}
```
