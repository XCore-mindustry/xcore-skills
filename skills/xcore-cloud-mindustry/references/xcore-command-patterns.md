# XCore Command Patterns

This file maps the XCore command stack into clear ownership boundaries.

## 1. `CloudService` is the infrastructure entrypoint

`CloudService`:

- creates client and server `MindustryCommandManager<XCoreSender>`
- configures parsers, guards, and exceptions
- creates annotation parsers
- exposes registration methods for controllers
- exposes help handlers

That makes it the primary integration point for command infrastructure.

## 2. `CloudManagerFactory` owns sender mapping and baseline manager policy

Factory responsibilities:

- create `SenderMapper<MindustrySender, XCoreSender>`
- choose execution coordinator
- set conflict strategy
- install permission checker
- install caption configuration

This is the right place for manager-wide policy, not controllers.

## 3. Configurers own narrow infrastructure concerns

### `CloudParserConfigurer`

Owns shared parsers and annotation parameter mapping.

### `CloudGuardConfigurer`

Owns pre/post-processors and metadata-driven command restrictions:

- disabled command checks
- mute checks
- playtime restrictions

### `CloudCaptionConfigurer`

Owns caption localization.

### `CloudExceptionConfigurer`

Owns centralized exception-to-user-message behavior.

## 4. `CloudCommandRegistrar` wires controllers into the managers

The registrar:

- receives `List<CloudClientController>` and `List<CloudServerController>`
- registers them at startup
- conditionally skips some client controllers based on config/gamemode

This is a good example of registration orchestration staying out of the controllers themselves.

## 5. Controllers are endpoint adapters

Controllers should:

- define command interface
- call services
- keep command-specific formatting small

Controllers should not:

- own persistence
- embed command infrastructure policy
- orchestrate large business workflows

## 6. `XCoreSender` wraps bridge sender behavior

`XCoreSender` adds:

- session-aware locale resolution
- Fluent bundle formatting
- sender API that is friendlier for XCore command code

This is a useful pattern for larger projects: wrap the bridge/platform sender in a domain-aware command sender.

## 7. Where changes should go

### Add a new command

- usually `command/controller/...`
- service call in `service/...`
- add bundle keys if player-facing text is introduced

### Add new syntax / argument type

- `cloud/parser/...`
- register in `CloudParserConfigurer`

### Add a new command-wide policy

- `cloud/config/...`
- use pre/post processors or metadata-based enforcement

### Add localized infrastructure errors

- caption provider / exception configurer / bundle keys

### Add a manager-wide sender/permission behavior change

- `CloudManagerFactory`

## 8. What to avoid

- custom ad-hoc managers in individual features
- duplicated permission logic across controllers
- parser registration inside one controller for a shared type
- localized error text hardcoded in every command handler
- business orchestration dumped into `cloud/` configurers
