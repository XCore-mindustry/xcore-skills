---
name: xcore-cloud-mindustry
description: >
  XCore-aware Incendo Cloud v2 and cloud-mindustry reference for Mindustry command infrastructure. Use this skill whenever
  the user is building or reviewing commands with MindustryCommandManager, CloudService, CloudCommandRegistrar,
  command controllers, parser registry, caption providers, exception handlers, permission checks, or annotation-based
  command registration in XCore-style plugins. Also use it when someone mentions cloud-mindustry, Incendo Cloud,
  command guards/parsers/captions, or asks where command logic should live in XCore.
---

# XCore Cloud / cloud-mindustry

This skill explains how **Incendo Cloud v2** and **`cloud-mindustry`** are used in XCore-style Mindustry plugins.

It is not meant to be a full reference for every `cloud-*` module. It focuses on the parts that matter most in this ecosystem:

- command manager mental model
- builder vs annotation registration
- parser registry and custom parsers
- permission checks, captions, and exception handling
- `MindustryCommandManager`, `MindustrySender`, `SenderMapper`, `ConflictStrategy`
- how XCore wraps all of that with `CloudService`, configurers, and thin command controllers

## When to read which reference

| You need to... | Read |
|---|---|
| Understand Cloud core concepts, command chains, builder API, registration | `references/core-cloud.md` |
| Understand `cloud-mindustry` API and Mindustry sender bridging | `references/cloud-mindustry.md` |
| Work on parsers, suggestions, captions, flags, or exception handlers | `references/parsers-captions-exceptions.md` |
| Use or evaluate annotations with XCore command registration | `references/annotations.md` |
| Follow XCore architectural boundaries for commands | `references/xcore-command-patterns.md` |
| Copy practical snippets and common recipes | `references/examples.md` |

Read only the relevant reference file.

## Mental model

There are three layers here:

1. **Incendo Cloud core** — generic command model, parsers, permissions, exceptions, help, annotations
2. **`cloud-mindustry`** — bridge from Cloud to Mindustry `CommandHandler`
3. **XCore command infrastructure** — wraps the bridge with XCore sender mapping, parser config, guard config, captions, and exception conventions

That means a command change should be designed by asking:

- is this generic Cloud behavior?
- is this bridge behavior (`cloud-mindustry`)?
- or is this XCore policy/parsing/business behavior?

## The most important XCore rules

### Keep responsibilities separate

- `cloud/` owns command infrastructure: parsers, guards, captions, exception behavior, permission policy
- `command/controller/` owns command endpoints only
- `service/` owns business logic and orchestration

If a controller is doing parsing policy, permission logic, or multi-step orchestration, it is in the wrong layer.

### Use `CloudService` as the entrypoint

In XCore, `CloudService` creates and configures the client/server command managers and owns annotation parsing.

Use it as the integration boundary rather than wiring ad-hoc command managers inside random features.

### Keep controllers declarative

Controllers should mostly:

- define command shape
- call services
- return/send results

They should not become business-logic hubs.

## Core patterns

### Builder API vs annotations

Cloud supports both builder-style commands and annotation-based registration.

In XCore, annotation-based registration is the main ergonomic layer, but it still relies on the same manager, parser registry, captions, exception controller, and command metadata.

Use builder mental models even when reading annotation-based code.

### Parser registry is the reusable extension point

If multiple commands need the same argument type, register a parser once instead of rebuilding command-local parsing every time.

This is how XCore handles `Team`, `Duration`, player lookup, maps, and translator languages.

### Captions and exceptions should stay centralized

Do not hardcode user-facing parse/permission/internal error messages in every controller.

Prefer:

- caption providers for translatable Cloud caption keys
- centralized exception handlers in `CloudExceptionConfigurer`
- domain exceptions like `XCoreCommandException` for business-facing command errors

### Permissions are policy, not controller trivia

Permission logic belongs in manager configuration or dedicated command policy/guard infrastructure — not scattered across controllers.

### `XCoreSender` is the command-facing sender abstraction

`cloud-mindustry` exposes `MindustrySender`, but XCore wraps it into `XCoreSender` to add:

- localization-aware output
- session-aware locale resolution
- a richer interface for the rest of the command layer

## Important gotchas

1. **`cloud-mindustry` is intentionally small** — don’t assume it includes every parser/helper/help renderer you might expect.
2. **Annotation support comes mostly from Cloud, not from custom bridge magic** — document it carefully.
3. **Parser registry matters for annotations** — annotated commands still depend on registered parsers.
4. **`cloud-mindustry` internal command registration uses reflection** — that’s an implementation detail, not the API you should build around.
5. **Don’t duplicate caption/exception logic in controllers** — XCore already centralizes this.
6. **Don’t put business logic into guards/parsers/configurers** — keep those focused on command infrastructure concerns.
7. **Builders are immutable** — each builder step returns a new builder.

## Recommended response style when using this skill

When helping the user:

1. identify which layer the problem belongs to: Cloud core, bridge, or XCore policy
2. recommend the narrowest correct extension point
3. preserve XCore boundaries (`cloud/` vs controller vs service)
4. prefer centralized parsers/captions/exceptions over one-off per-command fixes
5. if the user is writing a new command, explain both the command shape and where the non-command logic should live

## Quick reminders

### Typical XCore command path

```text
Mindustry CommandHandler
  -> MindustryCommandManager<XCoreSender>
  -> parser pipeline
  -> pre/post processors / guards
  -> exception controller
  -> controller method
  -> service
```

### Good rule of thumb

- syntax/parsing issue -> `cloud/` parser/configurer
- permission/guard/policy issue -> `cloud/` configurer/policy
- localized command errors -> caption/exception configurer
- business behavior -> service
- command endpoint shape -> controller
