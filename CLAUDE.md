# NOMAD.Maui — Project Instructions

## Project Overview

.NET MAUI mobile application targeting iOS (`net9.0-ios`) and Android (`net9.0-android`).
The repo doubles as a research platform for MAUI automation tooling — see `docs/` for benchmarks and best-practice writeups.

## Project Structure

```
src/NOMAD.Maui/          Main application
  MauiProgram.cs         App bootstrap, DI registration
  Pages/                 XAML pages (one file per screen)
  ViewModels/            ViewModels (one per page, MVVM)
  Services/              Business logic, data access
  Models/                Plain data models
  Resources/             Fonts, images, app icon, splash

docs/                    Knowledge base: benchmarks, best practices
```

## Required Toolchain

Run `/nomad-maui-doctor` to verify your environment. Required tools:

| Tool | Minimum | Purpose |
|---|---|---|
| .NET SDK | 9.0 | Build and run |
| `maui-ios` workload | latest | iOS builds |
| `maui-android` workload | latest | Android builds |
| Xcode | 16.0 | iOS simulator and device builds |
| Java JDK | 17 | Android toolchain |
| Android SDK / adb | any | Android device/emulator |

Optional tools (for automation and testing):

| Tool | Purpose |
|---|---|
| `maui` DevFlow CLI | In-process UI automation — 4–7× faster than Appium for AI agents (see `docs/appium-vs-maui-devflow.md`) |
| Appium 2.x | XCUITest/UiAutomator2-based UI automation |
| Node.js 18+ | Required by Appium |
| MAUI Sherpa | Desktop GUI to manage Android SDKs, Apple certificates/profiles, emulators, keystores & device inspectors (see `docs/maui-sherpa.md`) |
| MAUI Skills (Claude Code / Copilot plugin) | ~37 on-demand .NET MAUI & Xamarin-migration skills for AI coding agents — reference catalog in `docs/maui-skills.md`, install/usage in `docs/maui-skills-usage.md` |
| Syncfusion MAUI Skills | Per-control AI-agent skills for Syncfusion MAUI controls — **expected only when the project references `Syncfusion.Maui.*` packages** (auto-detected by the doctor). Install: `npx skills add syncfusion/maui-ui-components-skills` (see `docs/syncfusion-maui-skills.md`) |

## Architecture Rules

### MVVM — mandatory
- Every screen has a ViewModel in `ViewModels/` that extends `ObservableObject` (CommunityToolkit.Mvvm)
- Pages contain no business logic — only UI wiring
- Commands use `[RelayCommand]` attribute

### Dependency Injection — mandatory
- All ViewModels and services registered in `MauiProgram.cs` via `builder.Services`
- Pages and ViewModels receive dependencies via constructor injection
- No service locators or static singletons

### AutomationId — mandatory on every interactive element
- Every `Button`, `Entry`, `Switch`, `Slider`, `CheckBox`, `Picker`, `SearchBar`, `Editor`, `CollectionView` must have `AutomationId`
- Format: `PascalCase` matching element purpose — e.g. `LoginButton`, `UsernameEntry`, `NotificationsSwitch`
- Required for both UI testing and AI-driven automation workflows

### DevFlow Agent — mandatory in Debug builds
Register the MAUI DevFlow agent in `MauiProgram.cs` behind `#if DEBUG`:
```csharp
#if DEBUG
builder.UseMauiDevFlowAgent(); // enables AI-driven in-process automation
#endif
```
This enables the faster DevFlow automation path described in `docs/appium-vs-maui-devflow.md`.

## Boilerplate Every Screen Must Include

1. ViewModel backing with `[ObservableProperty]` and `[RelayCommand]`
2. `IsBusy` / loading indicator while async operations run
3. Error state with user-visible message
4. `AutomationId` on every interactive control
5. Keyboard avoidance (`ScrollView` or `SafeAreaView` wrapping form content)

## App Identity — must not be defaults

- `ApplicationId`: must not contain `com.companyname` — use a real reverse-domain ID
- `ApplicationIdGuid`: must be a unique GUID (not removed or duplicated from another project)
- App icon: must not use default MAUI purple `#512BD4`
- Splash screen: must not use default MAUI purple `#512BD4`

---

## Doctor Skill Integration

**Every time you add something to this repo, evaluate whether `/nomad-maui-doctor` needs a new check.**

| What you add | Evaluate for `/nomad-maui-doctor` |
|---|---|
| New required system tool or CLI | Add tool-presence and version check to Section 1 |
| New NuGet package that wraps a native SDK | Add SDK/runtime check to Section 1 |
| New platform target (e.g. Windows, macOS Catalyst) | Add workload check and platform-specific SDK check |
| New architecture convention (pattern, folder, naming) | Add structure/content check to Section 3 |
| New boilerplate requirement for screens | Add detection check to Section 3 |
| New automation framework or test runner | Add install check to Section 1, config check to Section 3 |
| New required app-config property | Add .csproj config check to Section 2 |

When a new check is warranted, update **`plugins/nomad-maui-skills/skills/nomad-maui-doctor/SKILL.md`** — add it to the appropriate section with clear PASS/WARN/FAIL criteria.

### Project profile (`.nomad/doctor.json`)

The doctor is **profile-driven**. On first run (or via `/nomad-maui-doctor init`) it asks the user about the project — target platforms, UI automation stack, optional tools — and stores the answers in `.nomad/doctor.json` (committed to git). Checks tagged **[profile]** in `SKILL.md` use it to decide whether a tool is **required** (FAIL if missing), expected (INFO), or **skipped** entirely.

When you add an **optional or conditional** tool (one that only some configurations use — an automation framework, a platform-specific SDK, a convenience tool), don't just add an always-on check. Instead:
1. Add a profile dimension/answer for it in **Step 0 — Initialization Mode**, and extend the `.nomad/doctor.json` schema.
2. Tag the new check **[profile]** and define its skip / escalation behavior against that profile value.

The goal: running `/nomad-maui-skills:nomad-maui-doctor` on a fresh clone should surface every missing setup step and every deviation from project conventions — tailored to how *this* project is configured — without requiring prior knowledge of this repo.

---

## Plugin Versioning

This repo ships a Claude Code plugin. Its payload is everything under **`plugins/nomad-maui-skills/`** (skills, scripts), and its version lives in **`.claude-plugin/marketplace.json`**.

**Whenever you change anything under `plugins/nomad-maui-skills/`, bump the plugin `version` in `.claude-plugin/marketplace.json` in the same change.** Claude Code uses that version to detect updates — without a bump, installed users won't be offered your changes.

Use semver:
- **patch** (`x.y.Z`) — fixes/wording in a skill, no behavior change for users
- **minor** (`x.Y.0`) — new skill, new check, new feature (backward-compatible)
- **major** (`X.0.0`) — breaking change (renamed/removed skill, changed invocation or arguments)

Changes outside `plugins/` (e.g. `docs/`, this `CLAUDE.md`) are **not** part of the plugin payload and do **not** require a version bump.
