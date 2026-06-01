# .NET MAUI Conventions & Best Practices

The conventions the NOMAD sample app follows and that new MAUI work should adopt. Verify a project against these with `/nomad-maui-doctor`.

> **Toolchain:** this project targets **net10** (`net10.0-ios` / `net10.0-android`). Don't hard-code tool versions here — `/nomad-maui-doctor` is the source of truth for the required SDK / workloads / Xcode and adapts to the project profile (`.nomad/doctor.json`).

---

## MVVM — mandatory
- Every screen has a ViewModel in `ViewModels/` that extends `ObservableObject` (CommunityToolkit.Mvvm).
- Pages contain no business logic — only UI wiring.
- Commands use the `[RelayCommand]` attribute.

## Dependency Injection — mandatory
- All ViewModels and services are registered in `MauiProgram.cs` via `builder.Services`.
- Pages and ViewModels receive dependencies via constructor injection.
- No service locators or static singletons.

## AutomationId — mandatory on every interactive element
- Every `Button`, `Entry`, `Switch`, `Slider`, `CheckBox`, `Picker`, `SearchBar`, `Editor`, `CollectionView` must have an `AutomationId`.
- Format: `PascalCase` matching the element's purpose — e.g. `LoginButton`, `UsernameEntry`, `NotificationsSwitch`.
- Required for both UI testing and AI-driven automation (DevFlow targets elements by `AutomationId`; opaque generated ids are brittle — see [`maui-devflow.md`](maui-devflow.md)).

## DevFlow agent — mandatory in Debug builds
Register the MAUI DevFlow agent in `MauiProgram.cs` behind `#if DEBUG`:
```csharp
#if DEBUG
using Microsoft.Maui.DevFlow.Agent;
// ...
builder.AddMauiDevFlowAgent();
#endif
```
Setup, version requirements, and usage are in [`maui-devflow.md`](maui-devflow.md) and the `/maui-devflow` skill.

## Boilerplate every screen must include
1. ViewModel backing with `[ObservableProperty]` and `[RelayCommand]`.
2. `IsBusy` / loading indicator while async operations run.
3. Error state with a user-visible message.
4. `AutomationId` on every interactive control.
5. Keyboard avoidance (`ScrollView` or `SafeAreaView` wrapping form content).

## App identity — must not be scaffold defaults
- `ApplicationId`: must not contain `com.companyname` — use a real reverse-domain ID.
- `ApplicationIdGuid`: must be a unique GUID (not removed or duplicated from another project).
- App icon: must not use the default MAUI purple `#512BD4`.
- Splash screen: must not use the default MAUI purple `#512BD4`.
