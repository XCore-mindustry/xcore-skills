# Annotations in Cloud / XCore

## Baseline

Cloud supports an annotation-based command layer through `AnnotationParser<C>`.

Basic shape:

```java
AnnotationParser<C> parser = new AnnotationParser<>(commandManager, SenderType.class);
```

## Important caution

`cloud-mindustry` itself does not provide a large custom annotation framework. The annotation layer comes primarily from upstream Cloud.

So when documenting or using annotations here:

- treat them as Cloud features running on top of `MindustryCommandManager`
- avoid claiming special bridge-specific annotation magic unless the repo actually implements it

## Why parser registry still matters

Annotations still resolve arguments through parser registration.

That means a missing parser is still a command infrastructure problem, even if the command endpoint itself is annotation-based.

## XCore annotation pattern

XCore creates separate annotation parsers for client and server managers inside `CloudService`:

```java
this.clientAnnotationParser = new AnnotationParser<>(clientManager, XCoreSender.class);
this.serverAnnotationParser = new AnnotationParser<>(serverManager, XCoreSender.class);
```

It also uses annotation-to-meta bridging for custom guard semantics:

```java
parser.registerBuilderModifier(RequiresMuteCheck.class,
    (_, builder) -> builder.meta(REQUIRES_MUTE_CHECK_META, true));
```

and:

```java
parser.registerBuilderModifier(RequiresPlayTime.class,
    (annotation, builder) -> builder.meta(REQUIRES_PLAY_TIME_META, annotation.value()));
```

This is an important XCore pattern:

- annotations define declarative command intent
- metadata stores the command requirement
- post-processors / guards enforce that requirement centrally

## Good use of annotations

Use annotations when they improve readability and keep controllers concise.

Avoid them if:

- the command shape is highly dynamic
- the command needs unusual builder-only constructs
- it would hide too much infrastructure behavior from maintainers
