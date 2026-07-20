---
name: maui-devflow
description: Set up and drive MAUI DevFlow in-process automation — install the CLI, register the agent, build/run a MAUI app, and inspect or interact with its live visual tree. Use when asked to automate, inspect, screenshot, or UI-test a .NET MAUI app with DevFlow.
---

DevFlow is Microsoft's experimental in-process automation toolkit for .NET MAUI: an HTTP agent runs **inside** the app and the `maui devflow` CLI drives it (dump the visual tree, tap/fill/scroll, screenshot). This skill is the action-oriented runbook. For the full command catalog, CSS selectors, MCP server, and benchmarks, see the [official DevFlow docs](https://learn.microsoft.com/en-us/dotnet/maui/developer-tools/devflow/?view=net-maui-10.0) (and, in the NOMAD.Maui repo, `docs/maui-devflow.md`).

---

## 0. Hard requirements (check these first — they cause most failures)

- **.NET 10 or later.** The DevFlow agent (preview) needs `net10.0-*` at minimum — a `net9.0` app cannot reference it. It also builds and runs fine on **`net11.0-ios`** (verified), so retargeting to a .NET 11 preview to get `dotnet watch` hot reload does not cost you DevFlow.
- **`Microsoft.Maui.Controls ≥ 10.0.20` — on net10 only.** The agent depends on it, and without an explicit pin you get `NU1605` downgrade errors. On **net11 the reverse applies**: remove explicit `Microsoft.Maui.Controls` / `Microsoft.Maui.Essentials` pins, or they trigger `NU1605` against the workload's implicit 11.x references.
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

UI commands live under `ui`, and the platform goes on the `devflow` verb (`-p ios|android|maccatalyst|windows`). There is **no `MAUI` or `agent interact` subcommand** — `agent` only discovers/inspects connected agents.

```bash
export PATH="$PATH:$HOME/.dotnet/tools"
maui devflow -p ios ui tree                                       # dump the live visual tree (JSON)
maui devflow -p ios ui query --selector "#LoginButton"            # find one element (most reliable form)
maui devflow -p ios ui screenshot --output screen.png             # capture
maui devflow -p ios ui tap  --automationId "LoginButton"
maui devflow -p ios ui fill UsernameEntry "a@b.com"                # elementId AND text are positional
maui devflow -p ios ui scroll --element "ItemList" --dy -400      # negative dy scrolls down
maui devflow -p ios ui navigate "//MainPage/Details"              # route is positional
```
Run `maui devflow ui <cmd> --help` to confirm flags — this is experimental and they move between previews.

The agent needs a moment to start its HTTP server after launch — retry `ui tree` a couple of times if the first call doesn't connect.

## 5. Keep the loop fast

Action latency is rarely the bottleneck; round trips are. Prefer these over the naive act-then-look pattern:

```bash
maui devflow -p ios ui tap --automationId "LoginButton" --and-screenshot after.png  # act + observe, one call
maui devflow -p ios ui query --selector "#List" --wait-until exists --timeout 10    # wait, don't screenshot-poll
maui devflow -p ios ui tree --format compact --depth 4                             # small dumps, not full trees
maui devflow -p ios ui scroll --element "List" --item-index 137                     # jump to a row directly
maui devflow batch --continue-on-error --delay 0 < setup.txt        # replay setup; --delay 0 matters
```

When the screen under test is several navigations deep, script the navigation with `batch` rather than re-deriving it visually each run — the setup steps aren't what's being tested. `batch` reads **plain CLI lines** on stdin (one command per line, no `maui devflow` prefix) and returns one JSON result per line.

Scope queries instead of dumping the tree — attribute-prefix selectors work and are dramatically cheaper:

```bash
maui devflow -p ios ui query --selector "Grid[automationId^=Revision_]" --format compact
```

Counting rows this way instead of grepping a full `ui tree` cut one measured step from **188 KB to 3 KB at identical speed**.

### Wait on *data*, not on the container

This is the most expensive mistake available here, and it fails silently.

`--wait-until exists` waits for the **element**, not its content. A `CollectionView`, a page root, or a `Label` bound to a not-yet-loaded property all exist *immediately* — so the wait returns at once and the next command acts on an empty screen. There is no `--wait-until non-empty`.

```bash
# WRONG — the list exists before its items load; the scroll then hits an empty list
maui devflow -p ios ui query --selector "#ProjectList" --wait-until exists --timeout 10
maui devflow -p ios ui scroll --element ProjectList --item-index 136

# RIGHT — wait for a row, which cannot exist before the data arrives
maui devflow -p ios ui query --selector "#Project_1" --wait-until exists --timeout 10
maui devflow -p ios ui scroll --element ProjectList --item-index 136
```

Same trap when reading a computed label: wait for a data-dependent child (`#Revision_1`), then read the label. Waiting on the label itself returns `"text": ""`.

Symptom when you get it wrong: every later `--wait-until` burns its **full timeout** and the run takes ~10s per step instead of ~0.15s.

### After launching or reinstalling, let the Shell settle

`ui query` reports tabs before Shell will accept taps on them. Tapping too early **silently no-ops** and every downstream wait then times out. Wait for a tab to exist, then pause ~1.5s before the first tap.

### Hot reload — **.NET 11 only**, and poll instead of sleeping

> **Applies to `net11.0-ios` with .NET 11 Preview 4+ only.** On **.NET 10 there is no hot
> reload from the CLI at all** — `dotnet watch` prints "🔥 Hot reload enabled", then never
> watches and ignores every edit. On net10, budget a rebuild (~9s) per change.

Setup (all four matter):

```xml
<TargetFrameworks>net11.0-ios</TargetFrameworks>
<MtouchLink>None</MtouchLink>          <!-- required; watch does nothing without it -->
```
- Pick the preview whose iOS SDK matches your installed Xcode (P4→Xcode 26.4, P6→26.6).
- **Remove** explicit `Microsoft.Maui.Controls` / `Microsoft.Maui.Essentials` pins (net11
  inverts the net10 rule — pins now cause `NU1605`).
- Pass **`--device <udid>` directly to `dotnet watch`**. After `--`, or as
  `--property:_DeviceName=`, it is silently ignored; the picker also renders an empty device
  list when stdin is not a TTY, so non-interactive runs must pass it.

```bash
dotnet watch --framework net11.0-ios \
  --property:RuntimeIdentifier=iossimulator-arm64 \
  --device <udid> > /tmp/watch.log 2>&1 &
```

The DevFlow agent runs happily inside the watched app, so you reload and drive in one session.

**Don't sleep — poll the log.** Measured latencies from the file write:

| write → watch detects | write → applied | write → observable via DevFlow |
|---|---|---|
| **0.07s** | **0.17s** (~100ms reported) | **0.67s** |

A defensive `sleep 8` costs more than the entire loop. Wait on the actual signal:

```bash
before=$(wc -c < /tmp/watch.log)
# ...make the edit...
until tail -c +$((before+1)) /tmp/watch.log | grep -aq "changes applied"; do sleep 0.1; done
```

**Use `grep -a`.** The watch log contains emoji (🔥 ⌚), so `grep` treats it as binary and
prints *nothing* while silently matching — it looks like the reload never happened.

**XAML does not hot reload.** Editing XAML logs `C# and Razor changes applied` and the UI
does **not** update. That message is about C#/Razor; treat XAML edits as needing a rebuild.

## 6. Gotchas & tips

- **Shell `Tab`s don't resolve via `--automationId`** (measured on CLI `0.1.0-preview.10.26274.3`). `ui query --selector "#SettingsTab"` finds the tab; `--automationId "SettingsTab"` returns nothing and `ui tap --automationId` errors with a misleading *"Check automationId spelling"*. Tap tabs with `--text "Settings" --type Tab`. Regular controls resolve fine either way.
- **`fill` and `set-property` take positional arguments**, not flags: `ui fill <Id> <text>`, `ui set-property <Id> <Property> <Value>`. Using `--automationId` with `fill` fails ("required argument missing") because the text binds to the elementId slot.
- **`network` (bare) is a streaming watch that never returns** — the one-shot is `network list`. It also does **not** capture native `HttpClient` traffic; only logs will show it.
- **The .NET 10 "Hot reload enabled" banner is a lie** — `dotnet watch` prints it on net10 iOS, then never watches. The `Socket error while connecting to IDE on 127.0.0.1:10000` line alongside it is the legacy Xamarin debugger channel, not hot reload — benign noise. See *Hot reload* in section 5; .NET 11 GA is targeted 2026-11-10.
- **Discover the real command surface** with `maui devflow commands` — machine-readable JSON of every command, with a `mutating` flag. Better than guessing from docs, including these.
- **`isVisible: true` does not mean on-screen.** An element clipped outside its parent still reports visible; compare `bounds` against the parent/screen to detect layout defects. MAUI emits no overflow warning of its own.
- **Target elements by `AutomationId`, not element id.** DevFlow falls back to opaque generated ids (e.g. `6a52a05152fd`) when a control lacks an `AutomationId` — brittle and unreadable. Ensure every interactive control has one (this project mandates it).
- **Android is in progress.** Needs `adb reverse tcp:<port> tcp:<port>` to reach the in-app agent; prefer iOS simulator or Mac Catalyst for now.
- **Release safety.** Keep the package reference and registration both behind Debug — never ship the agent.
- **Version skew.** Keep the `Microsoft.Maui.Cli` tool version and the `Microsoft.Maui.DevFlow.Agent` package version aligned (both `0.1.0-preview.10.*` here); it's experimental and APIs move between previews.
