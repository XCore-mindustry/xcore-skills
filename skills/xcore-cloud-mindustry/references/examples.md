# Practical Examples

## Minimal `cloud-mindustry` setup

```java
var mgr = MindustryCommandManager.create(Vars.netServer.clientCommands);

mgr.setConflictStrategy(ConflictStrategy.OVERRIDE);
mgr.setPermissionChecker((sender, perm) -> {
  if (sender.isPlayer() && sender.player() != null) {
    return sender.player().admin;
  }
  return true;
});
```

## Builder-style command

```java
mgr.command(mgr.commandBuilder("broadcast", "bc")
    .permission("myplugin.broadcast")
    .required("message", greedyStringParser())
    .handler(ctx -> {
      String msg = ctx.get("message");
      Call.sendMessage("[Gold][Broadcast] " + msg);
    })
);
```

## Integer component reuse

```java
var a = CommandComponent.<MindustrySender, Integer>builder("a", IntegerParser.integerParser()).build();
var b = CommandComponent.<MindustrySender, Integer>builder("b", IntegerParser.integerParser()).build();
```

## XCore manager creation

```java
SenderMapper<MindustrySender, XCoreSender> mapper = SenderMapper.create(
    base -> new XCoreSender(base, bundle, sessionService),
    XCoreSender::getHandle
);

MindustryCommandManager<XCoreSender> manager = new MindustryCommandManager<>(
    handler,
    ExecutionCoordinator.simpleCoordinator(),
    mapper
);

manager.setConflictStrategy(ConflictStrategy.OVERRIDE);
manager.setPermissionChecker(cloudPermissionPolicy::hasPermission);
cloudCaptionConfigurer.configure(manager);
```

## Shared parser registration in XCore

```java
manager.parserRegistry().registerParserSupplier(
    TypeToken.get(Duration.class),
    params -> new SmartDurationParser(
        timeService,
        params.get(DefaultUnit.PARAM, TimeUnit.DAYS),
        params.get(AllowNegativeDuration.PARAM, false)
    )
);
```

## Metadata-driven annotation guard pattern

```java
parser.registerBuilderModifier(RequiresMuteCheck.class,
    (_, builder) -> builder.meta(REQUIRES_MUTE_CHECK_META, true));

manager.registerCommandPostProcessor(context -> {
  if (context.command().commandMeta().getOrDefault(REQUIRES_MUTE_CHECK_META, false)) {
    ...
  }
});
```

## Caption localization pattern

As in XCore's `CloudCaptionConfigurer`:

```java
manager.captionRegistry().registerProvider((caption, recipient) -> {
  String key = CloudCaptionConfigurer.bundleKey(caption);  // "." and "_" become "-"
  if (!bundle.has(key)) {
    return null;                                           // keep the English default, not the raw key
  }
  // $variables become <variable> placeholders that Cloud fills with the caption variables
  var placeholders = new LinkedHashMap<String, Object>();
  for (String name : bundle.variables(key)) {
    placeholders.put(name, "<" + name + ">");
  }
  return bundle.format(recipient.locale(), key, placeholders);
});
```

## Centralized exception handler pattern

```java
manager.exceptionController().registerHandler(InvalidSyntaxException.class, ctx -> {
  String syntax = ctx.exception().correctSyntax();
  XCoreSender sender = ctx.context().sender();
  ...
});
```
