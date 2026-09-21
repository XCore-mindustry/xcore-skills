---
name: xcore-i18n-flubundle
description: >
  Standards and usage guide for FluBundle (Project Fluent localization) in Mindustry plugins and mods on Java 25.
  Covers .ftl file organization, Fluent syntax (parameters, plurals, select, term references), Bundle initialization,
  Localizer, BundleContext, locale normalization, player locale resolution, and Avaje Inject integration.
  Use whenever adding, editing, or formatting translated messages, multi-language UI, or localization bundles in XCore.
---

# FluBundle: Fluent Localization Guide for Mindustry

`FluBundle` (`com.ospx:flubundle`) is the standard localization engine for Mindustry plugins and mods in XCore. It replaces legacy `.properties` files with modern [Project Fluent](https://projectfluent.org/) (`.ftl`) syntax, supporting asymmetric plurals, parameters, gender, terms, and locale fallbacks.

---

## 1. When to Use

Use this skill when:

- Creating or editing localization messages in `bundles/*.ftl`.
- Formatting localized messages for players (`bundle.format(...)`, `Localizer`).
- Handling player language detection and per-player message delivery.
- Setting up DI bindings for `Bundle` in Avaje Inject modules.

---

## 2. Directory Structure & File Naming

Localization files live inside `bundles/` in the mod or plugin resources:

```text
src/main/resources/bundles/
├── bundle.ftl       # Default fallback (English)
├── bundle_en.ftl    # English
├── bundle_uk.ftl    # Ukrainian
├── bundle_ru.ftl    # Russian
└── bundle_zh.ftl    # Chinese
```

FluBundle normalizes locale tags automatically: `en-US`, `en_US`, and `EN_us` all resolve properly.

---

## 3. Fluent (.ftl) Syntax Guidelines

### Simple Messages & Terms

Terms start with `-` and cannot be queried directly from Java; they are referenced by other messages:

```ftl
-server-name = XCore Network

welcome = Welcome to { -server-name }!
login-required = Please log in using /login <password>.
```

### Variables & Parameters

Use `{ $varName }` for dynamic parameters:

```ftl
player-joined = [green]+[] { $playerName } joined the game.
balance-info = Your balance: [gold]{ $amount }[] coins.
```

### Plurals & Select Expressions

Fluent handles language-specific plural categories (`one`, `few`, `many`, `other`):

```ftl
ban-time-remaining =
    { $count ->
        [one] { $count } hour remaining
       *[other] { $count } hours remaining
    }

blocks-destroyed =
    { $count ->
        [one] destroyed { $count } block
        [few] destroyed { $count } blocks
       *[other] destroyed { $count } blocks
    }
```

---

## 4. Java API & Usage Patterns

### A. Initialization & Loading

```java
// Create bundle instance (or use Bundle.INSTANCE)
Bundle bundle = new Bundle();

// Load all .ftl bundles from the plugin/mod classpath
bundle.addSource(MyPlugin.class);

// Configure locale aliases if needed
bundle.addLocaleAlias("ua", "uk");
```

### B. Formatting Messages

```java
// 1. Direct formatting with Map arguments
String text = bundle.format(Locale.forLanguageTag("uk"), "player-joined", Map.of("playerName", "Vova"));

// 2. Localizer helper (bound to a specific locale)
Localizer localizer = bundle.localizer(Locale.forLanguageTag("uk"));
String welcome = localizer.format("welcome");

// 3. Fallback behavior:
// Exact locale -> Language locale -> Default bundle.ftl -> Missing key fallback
```

### C. Player Locale Resolution in Mindustry

Mindustry players report their client locale in `player.locale` (e.g. `uk_UA`):

```java
public void notifyPlayer(Player player, String messageKey, Map<String, Object> args) {
    Locale locale = Locale.forLanguageTag(player.locale.replace('_', '-'));
    String formatted = bundle.format(locale, messageKey, args);
    player.sendMessage(formatted);
}
```

### D. Avaje Inject Integration (`@Factory`)

Expose `Bundle` as a singleton bean in your DI configuration:

```java
@Factory
public class LocalizationFactory {

    @Singleton
    public Bundle bundle() {
        Bundle bundle = new Bundle();
        bundle.addSource(XcorePlugin.class);
        return bundle;
    }
}
```

---

## 5. Gradle Dependency

```kotlin
repositories {
    maven("https://maven.x-core.org/releases")
    maven("https://maven.x-core.org/snapshots")
}

dependencies {
    implementation("com.ospx:flubundle:1.3")
}
```
