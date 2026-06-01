# Exploration Radar

Candidates worth a look — libraries, tools, themes, techniques — that have been **noted but not evaluated**. 

> ⚠️ **These are not recommendations.** Nothing here is endorsed for use. An item earns a place in the rest of the knowledge base (a `docs/` guide, a toolchain entry) only after it's been explored and judged worth adopting. Until then it sits here so it isn't forgotten.

## How to use this list

- **Note something:** append a row to the table below — link, a one-line "what it is," what you'd want to evaluate, the date, and status `Noted`.
- **Status lifecycle:** `Noted` → `Exploring` → `Adopted` *or* `Parked`.
  - **Adopted** — write it up properly (a `docs/` page and, if it's a tool/CLI/framework, run `/nomad-add-tool` to register it), then set status to **Adopted** with a link to the writeup.
  - **Parked** — not pursuing (for now). Keep the row with a one-line reason so we don't re-evaluate it from scratch.

## Radar

| Item | What it is | Why noted / what to evaluate | Added | Status |
|---|---|---|---|---|
| [FlagstoneUI](https://github.com/matt-goldman/flagstone-ui) | .NET MAUI UI library — "enhanced, neutral controls designed for full visual control from shared code"; core controls (Button, Entry, Card, Editor), a Material theme, CommunityToolkit integrations. MIT. | Consistent control surface/behaviour across platforms with no platform code — a possible alternative to per-platform styling. Evaluate maturity (early: ~19★, 6 releases), control coverage, and fit with our MVVM/AutomationId conventions. | 2026-06-02 | Noted |
| [MauiBootstrapTheme](https://github.com/davidortinau/MauiBootstrapTheme) | Applies Bootstrap / Bootswatch CSS themes to **standard** MAUI controls via an MSBuild task that converts CSS into native `ResourceDictionary`s at compile time. MIT. | Theme stock controls without adopting a custom control library; the build-time CSS→ResourceDictionary approach is interesting. Evaluate maturity (early: ~14★, preview v0.1.0 Feb 2026), theming coverage, and runtime cost. | 2026-06-02 | Noted |
