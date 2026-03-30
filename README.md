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
    └── xcore-cloud-mindustry/
```

## Initial skills

- `mindustry-api` — Mindustry + Arc server-side plugin API reference
- `avaje-inject` — Avaje Inject compile-time DI reference with XCore patterns
- `xcore-cloud-mindustry` — Incendo Cloud + cloud-mindustry + XCore command infrastructure guidance
