# xcore-skills

Reusable AI skills for the XCore Mindustry ecosystem.

## Install

You can install these skills in two ways.

### Option 1: install from GitHub

```bash
npx skills add https://github.com/XCore-mindustry/xcore-skills
```

### Option 2: install manually

Clone the repository:

```bash
git clone https://github.com/XCore-mindustry/xcore-skills.git
```

Then copy the skill folders you want from `skills/` into your local skills directory, for example:

```bash
cp -R xcore-skills/skills/mindustry-api ~/.agents/skills/
cp -R xcore-skills/skills/avaje-inject ~/.agents/skills/
cp -R xcore-skills/skills/xcore-cloud-mindustry ~/.agents/skills/
```

If your setup uses a different skills directory, copy them there instead.

## Scope

This repository is intended to hold reusable skills for:

- Mindustry server/plugin development
- XCore infrastructure and plugin conventions
- shared Java tooling used across the XCore ecosystem

## Planned structure

```text
xcore-skills/
├── README.md
└── skills/
    ├── mindustry-api/
    ├── avaje-inject/
    ├── xcore-cloud-mindustry/
    ├── xcore-client-mod-testing/
    ├── xcore-gradle-toxopid/
    ├── xcore-mindustry-concurrency/
    ├── xcore-protocol-evolution/
    ├── xcore-python-services/
    ├── xcore-ui-architecture/
    ├── xcore-mindustry-testkit/
    └── xcore-i18n-flubundle/
```

## Skills Overview

### Mindustry & Arc Framework

- `mindustry-api` — Mindustry + Arc server-side plugin API reference (Call, Vars, Groups, events, networking)
- `xcore-mindustry-concurrency` — Rules for thread safety, non-blocking main loop, async I/O, and Redis Streams

### Tooling, Testing & Build System

- `xcore-gradle-toxopid` — Gradle & Toxopid guidelines for Java 25 server plugins and Java 17/Android client mods
- `xcore-client-mod-testing` — Client mod automated visual testing, DevHarness framebuffer capture, and release stripping
- `xcore-mindustry-testkit` — Deterministic headless testing for Mindustry plugins, virtual UI event loop, and client simulation

### Dependency Injection, Commands & Localization

- `avaje-inject` — Avaje Inject compile-time DI reference with XCore patterns and BeanScope
- `xcore-cloud-mindustry` — Incendo Cloud v2 + cloud-mindustry command infrastructure and controller patterns
- `xcore-i18n-flubundle` — Project Fluent (`.ftl`) localization engine for Mindustry, bundle contexts, and player delivery

### UI, Protocol & Services Ecosystem

- `xcore-ui-architecture` — Reactive server-driven UI architecture (Elm/MVI flow, record lenses, slots)
- `xcore-protocol-evolution` — Cross-service protocol evolution workflow (`xcore-protocol` schemas, fixtures, codegen)
- `xcore-python-services` — Standards and testing procedures for Python services (`uv`, `pytest`, `ruff`, Prometheus)
