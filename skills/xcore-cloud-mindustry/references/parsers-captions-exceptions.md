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

In XCore, `CloudCaptionConfigurer` maps Cloud caption keys to FluBundle keys and resolves placeholders through `BundlePlaceholderRegistry`.

Important pattern:

- convert `.` keys to `-`
- resolve placeholders centrally
- use session locale when player session exists
- fall back to bundle locale/default locale when needed

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

- `XCoreCommandException`
- invalid syntax
- no permission
- parser/caption-aware parse failures
- generic fallback exceptions

### Important design rule

If a command error is meant to be user-facing and localized, prefer a centralized exception path over ad-hoc controller-specific formatting.

## Silent command exceptions

XCore uses a domain exception type that can be silent. This is useful for guard-like flows where no user-facing message should be emitted from the general exception layer.
