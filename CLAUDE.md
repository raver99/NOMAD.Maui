# NOMAD.Maui

The **.NET MAUI–specific** companion to the conceptual **NOMAD** project: a **reusable knowledge base** that bundles the tools, guides, and sample code for building and automating .NET MAUI apps the NOMAD way. It is not primarily an app — the app is a sample. Three things live here:

- **Tools** — the `nomad-maui-skills` Claude Code plugin (`plugins/`): `/nomad-maui-doctor` (project & environment health check) and `/maui-devflow` (DevFlow automation runbook).
- **Guides** — the `docs/` knowledge base: conventions, tool how-tos, and benchmarks.
- **Sample app** — `src/NOMAD.Maui/`, a net10 MAUI app demonstrating the conventions.

## How to work with this repo

- **Check a project's health:** run `/nomad-maui-doctor` (first run asks a few questions and stores a profile in `.nomad/doctor.json`). It's the source of truth for required toolchain and convention compliance — don't duplicate version/tool specifics elsewhere.
- **Follow the conventions:** the MAUI conventions (MVVM, DI, AutomationId, DevFlow agent, screen boilerplate, app identity) are in [`docs/maui-conventions.md`](docs/maui-conventions.md).
- **Use a tool or guide:** browse [`docs/README.md`](docs/README.md) — each tool (DevFlow, Sherpa, MAUI Skills, Syncfusion) has its own page.
- **Install these skills elsewhere:** `/plugin marketplace add raver99/NOMAD.Maui` then `/plugin install nomad-maui-skills@nomad-maui-skills`.

## Project structure

```
src/NOMAD.Maui/          Sample MAUI app (net10-ios / net10-android)
  MauiProgram.cs         App bootstrap, DI registration
  Pages/ ViewModels/     XAML pages + MVVM ViewModels (one per screen)
  Services/ Models/       Business logic / data access, plain data models
  Resources/             Fonts, images, app icon, splash

plugins/nomad-maui-skills/   The Claude Code plugin (skills payload)
.claude-plugin/              Plugin marketplace manifest (version lives here)
docs/                        Knowledge base: conventions, tool guides, benchmarks
.nomad/                      Per-project doctor profile (doctor.json)
```

---

## Maintaining this repo

### Keep the doctor in sync
When you add a tool, convention, platform target, or screen requirement to this repo, evaluate whether `/nomad-maui-doctor` needs a new check, and update `plugins/nomad-maui-skills/skills/nomad-maui-doctor/SKILL.md` accordingly. For **optional or conditional** tools, add a profile dimension (Step 0 / `.nomad/doctor.json`) and tag the check `[profile]` rather than making it always-on. The doctor's own *Authoring notes* cover how to write robust detection.

### Bump the plugin version
The plugin payload is everything under `plugins/nomad-maui-skills/`; its version lives in `.claude-plugin/marketplace.json`. **Whenever you change anything under `plugins/nomad-maui-skills/`, bump that `version` in the same change** — Claude Code uses it to detect updates. Semver: **patch** = skill fix/wording, **minor** = new skill/check/feature, **major** = breaking change (renamed/removed skill or changed invocation). Changes outside `plugins/` (e.g. `docs/`, this file) don't need a bump.
