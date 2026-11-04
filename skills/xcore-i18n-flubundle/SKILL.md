---
name: xcore-i18n-flubundle
description: >
  Standards and usage guide for FluBundle 2.x (Project Fluent localization) in XCore Mindustry plugins on Java 25.
  Covers .ftl file naming and syntax, the shared Bundle, Messenger/Audience delivery, Args and Text, player and
  pre-join locale resolution, escaping player text, DURATION/COLOR/STRIP/ESCAPE, vanilla content names, missing keys,
  and the build-time checks. Use whenever adding, editing, or formatting translated messages, multi-language UI,
  or localization bundles in XCore, HexedCore, TileLogger or XCore Sentinel.
---

# FluBundle: Fluent localization for XCore plugins

`FluBundle` (`com.ospx:flubundle`, source in `xcore/libs/flubundle`) renders [Project Fluent](https://projectfluent.org/)
(`.ftl`) messages for Mindustry servers: plurals, selectors, terms, locale fallback, and delivery to players in
the language each of them selected.

## 1. Files

```text
src/main/resources/bundles/
├── bundle_en.ftl      # base locale: every key must exist here
├── bundle_ru.ftl
├── bundle_uk_UA.ftl   # XCore aliases "uk" to uk_UA
└── bundle_pt_BR.ftl
```

- The locale comes from the file name suffix (`bundle_en`, `bundle_uk_UA`). **A file without one (`bundle.ftl`)
  is skipped** with a warning; there is no suffix-less default file.
- Files of one locale (`bundle_en.ftl`, `menus_en.ftl`) load into the same bundle and see each other's ids.
- All plugins share one bundle at runtime, so prefix keys per plugin (`hexed_...`, `tilelogger-...`). Loading
  a key another plugin already defined overrides it and logs the override.
- Fallback: exact locale → language (`pt_BR` → `pt`) → default locale (`en`) → missing-key policy.

## 2. Fluent syntax

```ftl
-server = XCore

welcome = Welcome to { -server }, { $player }!
blocks-destroyed = { $count ->
    [one] destroyed { $count } block
   *[other] destroyed { $count } blocks
}
ban-notice = Banned for { DURATION($seconds, style: "full", maxUnits: 2) }: { ESCAPE($reason) }
report = [scarlet]{ $reporter }[] reported: { ESCAPE($message) }
```

- Every select expression needs a `*` default variant; plural selects need `*[other]`.
- Mindustry color tags (`[scarlet]`, `[]`) are plain text to Fluent.
- Functions: `DURATION` (CLDR unit names; `style: "compact" | "full" | "timer"`, `unit`, `colored`, `maxUnits`),
  `COLOR($x, color: "accent")`, `STRIP($x)` (removes color tags), `ESCAPE($x)` (shows `[` literally), plus
  Fluent's `NUMBER` and `DATETIME`. Narrow and no-break spaces are replaced with spaces the client font can draw.

## 3. The bundle in XCore

XCore owns the bundle: `BundleFactory` provides `Bundle.INSTANCE` and a `Messenger` bean, and
`SessionLocaleResolver` installs the player's selected language on it. Dependent plugins only add their files:

```java
// HexedCore, TileLogger, Sentinel: in the plugin's init, with the shared scope
scope.get(Bundle.class).addSource(getClass());   // loads bundles/*.ftl from this plugin's jar
```

Never create a `new Bundle()` in a plugin that runs next to XCore, and never parse `player.locale` yourself:
both bypass the language the player chose in settings.

## 4. Sending messages

Prefer `Messenger`; it resolves each player's locale and renders a message once per locale:

```java
Messenger messenger = Messenger.of(bundle);                 // or inject XCore's bean
messenger.to(player).send("welcome", Args.of("player", player.coloredName()));
messenger.all().announce("server-restart-countdown", Args.of("seconds", 10));
messenger.team(team).toast(Iconc.warning, "core-under-attack");
messenger.filter(p -> p.admin).popup(text, Popup.at(Align.top).duration(5f));
messenger.to(players).kick("subnet-denied");
messenger.to(list).deliver(text, (player, message) -> customTransport(player, message));
```

Delivery methods: `send` (chat), `announce`, `infoMessage`, `hud`, `toast`, `label`, `popup`, `kick`, `deliver`.

- `Args.of("k1", v1, "k2", v2)` (up to six pairs) or `Args.builder()`; `Bundle.args(...)` still works.
- `Text.of(key, args)` is a message rendered later: return it from services and let the caller pick recipients.
- Formatting a string: `bundle.format(bundle.locale(player), key, args)`; in commands `sender.send(key, args)` /
  `sender.format(key, args)` (XCore's `XCoreSender`), which use the session language.
- A value that depends on the recipient's language: `Localized.by(locale -> teamsLine(alive, locale))`; a `Text`
  argument (`Args.of("reason", Text.of("ban-reason-griefing"))`) is rendered in the same locale. Both still render
  once per locale in a broadcast. (Monorepo build; dependent plugins with the next release.)
- `Team` and `UnlockableContent` arguments (`Team.crux`, `UnitTypes.dagger`, `Blocks.router`) render as the
  vanilla name in the recipient's language (`ContentNames.vanilla()`). A `Player` argument renders as `player.name`.

## 5. Before the player joins

Connection checks run before a `Player` exists; the resolver looks the language up by the packet's uuid:

```java
String reason = messenger.connecting(con, packet).format("whitelist-denied");
messenger.connecting(con, packet).kick("ban-notice", Args.of("seconds", ban.remainingSeconds(), "reason", ban.reason));
Locale locale = bundle.locale(packet);
```

## 6. Text from players

Chat, Discord messages, sign text and reasons typed by players can contain `[]` or color tags that recolor or
close the rest of the line. Escape them: `ESCAPE($message)` in the FTL file, or `Markup.escape(text)` in Java.
Do not escape values meant to be colored, such as `player.coloredName()`.

## 7. Missing keys and errors

- XCore sets `MissingKeyPolicy.logOnce()`: a missing key is logged once and rendered as the key itself.
  Other policies: `returnKey()`, `bracketed()` (`⟦key⟧`), `throwing()` (tests).
- Render errors (a missing `$variable`, a failing function) are logged once per key and locale;
  `setFormatErrorHandler(...)` replaces that.
- Introspection: `bundle.has(key)`, `bundle.has(locale, key)`, `bundle.variables(key)`, `bundle.keys(locale)`.

## 8. Build-time checks

Every plugin keeps a test like this (see `FtlBundleCompilationTest` in each repository):

```java
// monorepo build (XCore, TileLogger); dependent plugins with the next release
BundleChecks.of(Path.of("src/main/resources/bundles"))
        .requireKeys(List.of("en", "ru"), keysTheseLocalesMustTranslate)   // optional
        .assertValid();

// released 2.1.3
CompilationResult compiled = FtlCompiler.compile(Path.of("src/main/resources/bundles"));
assertFalse(compiled.hasErrors(), compiled.formatReport());
CompilationResult consistent = LocaleConsistencyChecker.check(Path.of("src/main/resources/bundles"), "en");
assertFalse(consistent.hasErrors(), consistent.formatReport());
```

- `FtlCompiler`: syntax errors, unknown functions and options, select expressions without a default. In the
  monorepo build it also rejects references to messages, terms or attributes the locale does not define and warns
  about ids defined twice; allow ids from another plugin's bundle with
  `FtlCompiler.builder().externalReferences("-xcore").build()`. Plugins on the released 2.1.3 get this with the
  next release.
- `LocaleConsistencyChecker`: a translation using a `$variable` the English message does not is an error;
  dropped variables, orphan keys and untranslated keys are warnings.
- `KeyConstantsGenerator` generates a class with a constant per key and a typed method per message
  (`BundleKeys.hexedCountdownTick(Object seconds)` returns a `Text`), so a renamed key or a missing argument
  breaks compilation (see the FluBundle README; typed methods in the monorepo build).

## 9. Gradle

```kotlin
repositories {
    maven("https://maven.x-core.org/releases")
}

dependencies {
    // XCore ships FluBundle; dependent plugins compile against it
    compileOnly("com.ospx:flubundle:2.1.3")
    testImplementation("com.ospx:flubundle:2.1.3")
}
```

Inside the XCore monorepo use `project(":libs:flubundle")`. Upgrading from 1.x: `libs/flubundle/docs/migrating-to-2.0.md`.
