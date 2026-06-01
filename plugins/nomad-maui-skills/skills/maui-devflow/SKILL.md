---
name: maui-devflow
description: Set up and drive MAUI DevFlow in-process automation — install the CLI, register the agent, build/run a MAUI app, and inspect or interact with its live visual tree. Use when asked to automate, inspect, screenshot, or UI-test a .NET MAUI app with DevFlow.
---

DevFlow is Microsoft's experimental in-process automation toolkit for .NET MAUI: an HTTP agent runs **inside** the app and the `maui devflow` CLI drives it (dump the visual tree, tap/fill/scroll, screenshot). This skill is the action-oriented runbook. For the full command catalog, CSS selectors, MCP server, and benchmarks, see the [official DevFlow docs](https://learn.microsoft.com/en-us/dotnet/maui/developer-tools/devflow/?view=net-maui-10.0) (and, in the NOMAD.Maui repo, `docs/maui-devflow.md`).

---

## 0. Hard requirements (check these first — they cause most failures)

- **.NET 10.** The DevFlow agent (preview) ships **net10-only** assemblies. A `net9.0` app cannot reference it — the app must target `net10.0-*`.
- **`Microsoft.Maui.Controls ≥ 10.0.20`.** The agent depends on it; without an explicit pin you get `NU1605` package-downgrade errors.
- **net10 MAUI workloads** installed (`dotnet workload list` shows `maui-ios` / `maui-android` at 10.x).
- **Apple toolchain (iOS/MacCatalyst):** the installed `Microsoft.iOS.Sdk` pack pins an **exact Xcode version**. If the active Xcode differs, builds fail with "requires Xcode X". Don't change the global selection — point one build at the right Xcode with `DEVELOPER_DIR` (see step 3).

## 1. Install the CLI

```bash
dotnet tool install -g Microsoft.Maui.Cli --prerelease
```
Detect it PATH-independently (global tools live in `~/.dotnet/tools`, not on PATH in non-interactive shells):
```bash
dotnet tool list -g | grep -iE '^microsoft\.maui\.cli' && PATH="$PATH:$HOME/.dotnet/tools" maui devflow --version
```

## 2. Wire the agent into the app

`.csproj` — reference the agent **Debug-only** (it must never ship in release) and pin Controls:
```xml
<ItemGroup>
  <PackageReference Include="Microsoft.Maui.Controls" Version="10.0.20" />
</ItemGroup>
<ItemGroup Condition="'$(Configuration)' == 'Debug'">
  <PackageReference Include="Microsoft.Maui.DevFlow.Agent" Version="0.1.0-preview.10.26274.3" />
</ItemGroup>
```
`MauiProgram.cs` — register under `#if DEBUG`:
```csharp
#if DEBUG
using Microsoft.Maui.DevFlow.Agent;
#endif
// ...
var builder = MauiApp.CreateBuilder();
builder.UseMauiApp<App>();
#if DEBUG
builder.AddMauiDevFlowAgent();   // NOT "UseMauiDevFlowAgent"
#endif
```

## 3. Build & run on a simulator (iOS — best-supported path)

iOS simulator shares the host network, so no port forwarding is needed.
```bash
# Select the Xcode the iOS SDK pack expects (adjust path/version to the error message):
export DEVELOPER_DIR="/Applications/Xcode 26.1.app/Contents/Developer"

dotnet build path/to/App.csproj -f net10.0-ios -c Debug -p:RuntimeIdentifier=iossimulator-arm64

app="path/to/bin/Debug/net10.0-ios/iossimulator-arm64/App.app"
udid=$(xcrun simctl list devices available | grep -oE '[0-9A-F-]{36}' | head -1)
xcrun simctl boot "$udid"; open -a Simulator; xcrun simctl bootstatus "$udid" -b
xcrun simctl install "$udid" "$app"
xcrun simctl launch "$udid" <ApplicationId>
```

## 4. Drive it

```bash
export PATH="$PATH:$HOME/.dotnet/tools"
maui devflow MAUI tree                                   # dump the live visual tree (JSON)
maui devflow MAUI tree --query "#LoginButton"            # filter by AutomationId
maui devflow MAUI screenshot -o screen.png               # capture
maui devflow agent interact tap   --automationid "LoginButton"
maui devflow agent interact fill  --automationid "UsernameEntry" --text "a@b.com"
maui devflow agent interact scroll --automationid "ItemList" --direction down
```
The agent needs a moment to start its HTTP server after launch — retry `tree` a couple of times if the first call doesn't connect.

## 5. Gotchas & tips

- **Target elements by `AutomationId`, not element id.** DevFlow falls back to opaque generated ids (e.g. `6a52a05152fd`) when a control lacks an `AutomationId` — brittle and unreadable. Ensure every interactive control has one (this project mandates it).
- **Android is in progress.** Needs `adb reverse tcp:<port> tcp:<port>` to reach the in-app agent; prefer iOS simulator or Mac Catalyst for now.
- **Release safety.** Keep the package reference and registration both behind Debug — never ship the agent.
- **Version skew.** Keep the `Microsoft.Maui.Cli` tool version and the `Microsoft.Maui.DevFlow.Agent` package version aligned (both `0.1.0-preview.10.*` here); it's experimental and APIs move between previews.
