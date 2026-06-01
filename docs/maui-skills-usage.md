# MAUI Skills — How to Install & Use

**Source:** [GitHub (davidortinau/maui-skills)](https://github.com/davidortinau/maui-skills)

> **What these skills are and the full catalog →** [`maui-skills.md`](maui-skills.md)

---

## What you get

A plugin that adds ~37 MAUI / Xamarin-migration skills to your AI coding agent. The agent loads the matching skill **automatically** when your prompt is on-topic — there is no manual "activate skill X" step. You install once; the skills then surface themselves on demand.

Works with two agents:

- **Claude Code**
- **GitHub Copilot CLI**

---

## Install

Both agents install via the same plugin-marketplace flow.

### Claude Code

```bash
/plugin marketplace add davidortinau/maui-skills
/plugin install maui-skills@maui-skills
```

### GitHub Copilot CLI

```bash
/plugin marketplace add davidortinau/maui-skills
/plugin install maui-skills@maui-skills
/skills reload
```

`/plugin marketplace add` registers the repo as a source; `/plugin install <plugin>@<marketplace>` installs the plugin from it. On Copilot CLI, `/skills reload` makes the newly installed skills available in the current session.

> The skills live in the plugin once installed — you do **not** clone this into the NOMAD.Maui repo or add any NuGet package. Nothing about the app build changes.

---

## Use

1. **Install once** (above).
2. **Prompt normally.** Describe the MAUI task in plain language. When your prompt matches a skill's topic, the agent pulls that skill's guidance into context automatically.
3. **Be topic-explicit for best matching.** Naming the area helps the right skill load:
   - "Add **pull-to-refresh** to the contacts `CollectionView`" → `maui-collectionview`
   - "Set up **APNS push notifications** with Azure Notification Hubs" → `maui-push-notifications`
   - "**Migrate** this Xamarin.Forms page to MAUI" → `xamarin-forms-migration`
4. **Stack with repo conventions.** These skills give general MAUI best practice; this project layers stricter rules on top (mandatory `AutomationId`, DevFlow agent registration, MVVM + DI). When guidance conflicts, **[`CLAUDE.md`](../CLAUDE.md) wins** — tell the agent so if needed.

### Verifying it's active

In Claude Code, list installed plugins with `/plugin` (or check that a MAUI prompt pulls in skill guidance). In Copilot CLI, `/skills` lists available skills after a `reload`.

---

## Updating & removing

- **Update:** re-run `/plugin install maui-skills@maui-skills` (and `/skills reload` on Copilot CLI) to pull the latest skills as the upstream repo grows.
- **Remove:** uninstall the plugin via your agent's plugin manager (`/plugin` in Claude Code).

---

## When to reach for these vs. project docs

| Situation | Use |
|---|---|
| Implementing a standard MAUI feature (notifications, maps, secure storage…) | MAUI Skills — let the matching skill guide the agent |
| Migrating Xamarin code | MAUI Skills (`xamarin-*`) |
| Project-specific rules (folder layout, `AutomationId`, DevFlow, app identity) | [`CLAUDE.md`](../CLAUDE.md) |
| UI automation / testing setup | [`maui-devflow.md`](maui-devflow.md), [`research/appium-vs-maui-devflow.md`](research/appium-vs-maui-devflow.md) |
| Dev-machine environment management | [`maui-sherpa.md`](maui-sherpa.md), `/nomad-doctor` |
