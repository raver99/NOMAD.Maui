# NOMAD.Maui

The .NET MAUI implementation of the [NOMAD Framework](https://github.com/raver99/NOMAD-Framework) — opinionated best practices, tooling, and AI-assisted workflows for building cross-platform mobile apps with .NET MAUI (iOS & Android).

> NOMAD (Nextgen Optimized Mobile Application Development) is a platform-agnostic framework. NOMAD.Maui brings its principles to the .NET MAUI stack.

A reusable knowledge base bundling **tools**, **guides**, and a **sample app**. This README is the index — start here.

---

## Tools — Claude Code plugin

Skills for AI-assisted .NET MAUI development. Install in any project:

```
/plugin marketplace add raver99/NOMAD.Maui
/plugin install nomad-maui-skills@nomad-maui-skills
```

- **`/nomad-maui-doctor`** — profile-driven health check for your environment and project conventions (first run stores a profile in `.nomad/doctor.json`). [Skill](plugins/nomad-maui-skills/skills/nomad-maui-doctor/SKILL.md)
- **`/maui-devflow`** — runbook for DevFlow in-process UI automation: install, register the agent, build/run, and drive a live app. [Skill](plugins/nomad-maui-skills/skills/maui-devflow/SKILL.md)

## Knowledge base — [`docs/`](docs/README.md)

- **[Conventions](docs/maui-conventions.md)** — MVVM, DI, mandatory `AutomationId`, DevFlow agent, screen boilerplate, app identity.
- **Tool guides** — [DevFlow](docs/maui-devflow.md) · [MAUI Sherpa](docs/maui-sherpa.md) · [MAUI Skills](docs/maui-skills.md) · [Syncfusion Skills](docs/syncfusion-maui-skills.md) · [UI Components](docs/ui-components.md)
- **[Exploration Radar](docs/radar.md)** — candidates noted for future exploration (libraries, tools, themes). **Not recommendations.**
- **Research** — [Appium vs MAUI DevFlow benchmark](docs/research/appium-vs-maui-devflow.md)

## Sample app — [`src/NOMAD.Maui/`](src/NOMAD.Maui/)

A net10 MAUI app demonstrating the conventions above.

## Working in this repo — [`CLAUDE.md`](CLAUDE.md)

What the repo is, how to work with it, and the maintenance rules (keep the doctor in sync, bump the plugin version).
