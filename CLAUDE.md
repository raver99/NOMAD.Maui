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

The goal: running `/nomad-maui-skills:nomad-maui-doctor` on a fresh clone should surface every missing setup step and every deviation from project conventions, without requiring prior knowledge of this repo.
