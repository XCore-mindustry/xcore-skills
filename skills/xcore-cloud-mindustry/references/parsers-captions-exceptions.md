# Parsers, Suggestions, Captions, Flags, and Exceptions

## Parser registry

Parser registry is the reusable parser extension point.

Use it when:

- multiple commands share the same argument type
- annotation-based commands need to resolve custom types
- parser behavior depends on annotation-supplied parameters

### Useful upstream concepts

- `registerParserSupplier(...)`
- named parsers
- parser parameter mapping
- parser registry participation for annotation layer

## Standard parser types worth knowing

- strings: single, quoted, greedy
- numeric parsers
- boolean parsers
- enum parsers
- UUID parsers

## XCore parser pattern

XCore configures the manager once in `CloudParserConfigurer` and registers shared parsers there.

Examples:

- `TeamParser`
- `SmartDurationParser`
- `LanguageParser`
- `PlayerParser`
- `MapParser`

### Annotation parameter mapping pattern

```java
manager.parserRegistry().registerAnnotationMapper(
  AllTeams.class,
  (_, _) -> ParserParameters.single(AllTeams.PARAM, true)
);
```

This lets annotations drive parser behavior without pushing parsing policy into controllers.

## Suggestions

Parsers can supply suggestions, and components usually use parser suggestions by default.

Useful interfaces from Cloud:

- `SuggestionProvider`
- `BlockingSuggestionProvider`

## Flags

Flags are always optional and are parsed at the tail of the command chain.

Kinds:

- presence flags
- value flags

Examples:

- `--name`
- `-a`
- chained short aliases like `-abc`

## Captions

Cloud captions are the main hook for localized infrastructure messages.

In XCore, `CloudCaptionConfigurer` registers a caption provider that maps a Cloud caption key to a FluBundle key
(`CloudCaptionConfigurer.bundleKey`: `.` and `_` become `-`, so `argument.parse.failure.player_info` is
`argument-parse-failure-player-info`). It renders the bundle message with every `$variable` replaced by a
`<variable>` placeholder, which Cloud then fills with the caption variables.

Important pattern:

- return `null` from the provider when the bundle has no such key, so the English default of Cloud or
  cloud-mindustry applies instead of a raw key
- the message is rendered in the recipient's locale (`XCoreSender.locale()`: session language, else client locale,
  else the default locale for the console)
- because the values arrive as text placeholders, plural selectors and `NUMBER()` on caption variables do not work
  on this path; keep caption messages free of them
- cloud-mindustry's own captions are listed in `MindustryCaptionKeys` and `SelectorCaptionKeys`;
  XCore's `FtlBundleCompilationTest` checks that en/ru/uk_UA translate all of them except `mindustry.help.*`

## Exception handling

Cloud uses `ExceptionController`.

Common exception types:

- `NoPermissionException`
- `InvalidSyntaxException`
- `ArgumentParseException`
- `CommandExecutionException`

### XCore pattern

XCore centralizes exception handling in `CloudExceptionConfigurer`.

It handles:

- `XCoreCommandException` (its key and args; silent ones send nothing)
- invalid syntax → `error-invalid-syntax`
- no permission → `error-access-denied`
- parse failures: a `ParserException` cause shows its caption through the bundle when XCore translates the key,
  otherwise the caption's default text; any other cause shows `error-argument-parse-generic`
- everything else → `error-internal`, with the stack trace logged

Players and the console get the same localized messages (the console in the default locale). The text of an
exception is never sent to the sender, because it can carry internal details; it goes to the log.

### Important design rule

If a command error is meant to be user-facing and localized, prefer a centralized exception path over ad-hoc controller-specific formatting.

## Silent command exceptions

XCore uses a domain exception type that can be silent. This is useful for guard-like flows where no user-facing message should be emitted from the general exception layer.
