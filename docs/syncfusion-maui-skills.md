# Syncfusion MAUI Skills — How to Install & Use

**Source:** [Syncfusion .NET MAUI agent skills](https://help.syncfusion.com/maui/skills/controls-user-skills) · repo [`syncfusion/maui-ui-components-skills`](https://github.com/syncfusion/maui-ui-components-skills)
**Status:** Relevant **only if this project uses Syncfusion MAUI controls.** If it doesn't, ignore this page — `/nomad-maui-doctor` will skip the check automatically.

---

## What it is

A set of per-control AI-agent skills for **Syncfusion® .NET MAUI** components. Each skill teaches the agent the correct XAML/C# usage, properties, and patterns for one control (e.g. `syncfusion-maui-autocomplete`, `syncfusion-maui-charts`, `syncfusion-maui-datagrid`), so generated code matches Syncfusion's APIs instead of being guessed.

Unlike `davidortinau/maui-skills` (a Claude Code / Copilot **plugin** — see [`maui-skills.md`](maui-skills.md)), these are distributed through the agent-agnostic **Skills CLI** (`npx skills …`), which works across Claude Code, Copilot, VS Code, Cursor, and others.

## When this applies to NOMAD.Maui

These skills only matter if the project actually depends on Syncfusion. The signal is a `Syncfusion.Maui.*` NuGet reference in a `.csproj` (e.g. `Syncfusion.Maui.Core`, `Syncfusion.Maui.Charts`, `Syncfusion.Maui.Inputs`), usually paired with `builder.ConfigureSyncfusionCore()` in `MauiProgram.cs` and `clr-namespace:Syncfusion.Maui.*` XAML namespaces.

`/nomad-maui-doctor` auto-detects this — there is **no profile question** for it. If Syncfusion packages are present, the doctor expects the skills to be installed and WARNs if they're missing; if no Syncfusion packages are present, the check is skipped entirely.

## Prerequisites

- **Node.js ≥ 16** (the Skills CLI runs via `npx`).
- An existing or new MAUI app already using (or about to use) Syncfusion controls.

## Install

The same command works for all supported agents. Run it from the repo root:

```bash
# Install all Syncfusion MAUI control skills (non-interactive)
npx skills add syncfusion/maui-ui-components-skills -y

# Or pick specific controls interactively
npx skills add syncfusion/maui-ui-components-skills
```

During setup you choose the **scope**:

- **Project-level** — skills are written into the repo and committed, so every clone and teammate gets them. Recommended when the whole team works on Syncfusion screens.
- **Global** — installed once on your machine and available to every project. Convenient for an individual who touches many Syncfusion repos.

Either scope satisfies the doctor check.

## Use

Once installed, the agent loads the matching control skill on demand when your prompt references that control — e.g. *"add a Syncfusion `SfAutoComplete` with filtered suggestions"* pulls in `syncfusion-maui-autocomplete`. No manual activation.

## Best practices

- **Commit project-level skills** if Syncfusion usage is a project decision — it keeps the doctor green for every clone and documents the dependency.
- **Keep them updated** as you add Syncfusion packages — re-run `npx skills add …` to pull skills for newly used controls.
- These skills cover Syncfusion API usage; project conventions in [`CLAUDE.md`](../CLAUDE.md) (MVVM, DI, mandatory `AutomationId`) still win on conflict.
