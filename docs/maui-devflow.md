# MAUI DevFlow — How to Use & Best Practices

**Source:** [Official Microsoft Docs](https://learn.microsoft.com/en-us/dotnet/maui/developer-tools/devflow/?view=net-maui-10.0)  
**Status:** Experimental — API changes between releases. Pin package and CLI versions.

---

## What is DevFlow

DevFlow is an in-process testing, automation, and debugging toolkit for .NET MAUI apps. Unlike Appium (which drives apps through the OS accessibility tree via a separate process), DevFlow runs an HTTP agent **inside** the app. This gives it direct access to the MAUI visual tree — no cross-process serialisation, no accessibility layer lossy translation.

See [Appium vs MAUI DevFlow Benchmark](research/appium-vs-maui-devflow.md) for measured performance comparisons (4.6× faster actions, 7.2× faster end-to-end AI agent task).

### Three components

| Component | Role |
|---|---|
| **Agent** | In-app HTTP server. Exposes visual tree, interactions, screenshots, profiling. Runs inside the app process. |
| **CLI** (`maui devflow`) | Command-line interface with 50+ commands. Talks to the agent. |
| **Broker** | Background daemon for port assignment when multiple apps run simultaneously. |

---

## Setup

### 1. Add NuGet packages

```xml
<PackageReference Include="Microsoft.Maui.DevFlow.Agent" />

<!-- If using Blazor Hybrid: -->
<PackageReference Include="Microsoft.Maui.DevFlow.Blazor" />
```

### 2. Register in MauiProgram.cs

```csharp
using Microsoft.Maui.DevFlow.Agent;

public static MauiApp CreateMauiApp()
{
    var builder = MauiApp.CreateBuilder();
    builder.UseMauiApp<App>();

#if DEBUG
    builder.AddMauiDevFlowAgent();
#endif

    return builder.Build();
}
```

The `#if DEBUG` guard is essential — the agent must never ship in release builds.

### 3. Install the CLI

```bash
dotnet tool install -g Microsoft.Maui.Cli --prerelease
```

Verify with:

```bash
maui devflow --version
```

---

## Platform support

| Platform | Status | Notes |
|---|---|---|
| iOS Simulator | ✅ | Shares host network — no extra setup |
| Mac Catalyst | ✅ | Direct localhost |
| Linux/GTK | ✅ | Direct localhost |
| Android | 🔄 In progress | Requires `adb reverse` for port forwarding |
| Windows | 🔄 In progress | — |

---

## Core CLI commands

### Inspect the visual tree

```bash
maui devflow MAUI tree
```

Dumps all elements with: control type, `AutomationId`, bounding rectangles, and property values. Use this first to orient an automation session or debug layout issues.

Filter with a CSS selector:

```bash
maui devflow MAUI tree --query "Button"
maui devflow MAUI tree --query "#LoginButton"
```

### Take a screenshot

```bash
maui devflow MAUI screenshot -o screenshot.png
```

Works on all supported platforms. Both the CLI and the AI agent can capture screenshots this way.

### Tap an element

```bash
maui devflow agent interact tap --automationid "LoginButton"
```

Simulates a real user tap, triggering event handlers and commands.

### Fill text

```bash
maui devflow agent interact fill --automationid "UsernameEntry" --text "testuser@example.com"
```

Clears existing text and writes directly to the MAUI `Entry` binding — not via simulated keystrokes. This is why fill is 10–11× faster than Appium's `sendKeys`.

### Scroll

```bash
maui devflow agent interact scroll --automationid "ItemList" --direction down
```

Supported directions: `up`, `down`, `left`, `right`. Works on `ScrollView` and `CollectionView`.

### Navigate

```bash
# Shell-based apps
maui devflow agent interact navigate --route "//MainPage/Details"

# NavigationPage-based apps — specify target page type
```

### Mutate a property (runtime debugging)

```bash
maui devflow agent interact mutate --automationid "StatusLabel" --property "Text" --value "Overridden"
```

Applied immediately to the running app. Not persisted — resets on restart. Useful for testing visual states without rebuilding.

---

## CSS selector reference

DevFlow uses a CSS selector engine to query the MAUI visual tree.

| Selector | Matches |
|---|---|
| `Button` | All `Button` elements |
| `#MyButton` | Element with `AutomationId="MyButton"` |
| `.MyClass` | Elements with `StyleClass="MyClass"` |
| `StackLayout > Button` | Direct child `Button`s of a `StackLayout` |

Selectors work in the CLI (`--query`), in MCP tool calls, and in the HTTP API.

---

## MCP server (AI agent integration)

Start the MCP server:

```bash
maui devflow mcp
```

This launches a stdio-based MCP server. Add it to your AI tool's config:

```json
{
  "mcpServers": {
    "maui-devflow": {
      "command": "maui",
      "args": ["devflow", "mcp"]
    }
  }
}
```

### Tool categories

| Category | What it does |
|---|---|
| **Tree** | Visual tree inspection |
| **Query** | CSS selector-based element queries |
| **Screenshot** | Capture PNG screenshots |
| **Interaction** | Tap, fill, scroll UI elements |
| **Navigation** | Shell and page navigation |
| **Property** | Inspect and mutate element properties |
| **Assert** | UI assertions for testing |
| **Logs** | Read app logs |
| **Network** | HTTP request/response monitoring |
| **Storage** | Sandboxed file access (list, download, upload, delete) |
| **Preferences** | App preferences read/write |
| **Platform** | Device and platform info |
| **Sensor** | Device sensor data |
| **CDP** | Chrome DevTools Protocol for Blazor WebViews |
| **Recording** | Interaction recording |
| **Agent** | Agent status and connected agent listing |

### Storage tools quick reference

| Tool | Purpose |
|---|---|
| `maui_storage_roots` | List available file storage roots |
| `maui_files_list` | List files under a storage root |
| `maui_files_download` | Download a file (returns base64) |
| `maui_files_upload` | Upload a file (base64 content) |
| `maui_files_delete` | Delete a file |

Always call `maui_storage_roots` before specifying a non-default root.

---

## Best practices

### AutomationId on every interactive element

DevFlow finds elements by `AutomationId`. Without it, you're relying on fragile CSS type selectors that break when you refactor. Set `AutomationId` on every element that automation or an AI agent will need to interact with:

```xml
<Entry AutomationId="UsernameEntry" />
<Entry AutomationId="PasswordEntry" />
<Button AutomationId="LoginButton" />
<Label AutomationId="ValidationMessage" />
```

Use stable, descriptive IDs — treat them like a public API for your UI.

### Guard the agent behind `#if DEBUG`

The agent exposes a local HTTP server. It must never ship in release builds. The `#if DEBUG` guard in `MauiProgram.cs` is the right pattern. Confirm your CI release pipeline doesn't build with `DEBUG` defined.

### Pin package and CLI versions

DevFlow is experimental. A version bump can change tool names, parameters, or behaviour. Pin both:

```xml
<PackageReference Include="Microsoft.Maui.DevFlow.Agent" Version="0.1.0-preview.10.26274.3" />
```

```bash
dotnet tool install -g Microsoft.Maui.Cli --version 0.1.0-preview.10.26274.3
```

Document which version your automation harness was written against.

### Pre-launch the app; don't relaunch per test

The agent is always running once the app is started. There is no session lifecycle. The cheapest pattern is:

1. Launch the app once before any test run.
2. Between tests, reset UI state via navigation commands — don't relaunch.
3. Only relaunch if your test genuinely corrupts app state that navigation can't undo.

This avoids cold-start JIT cost and any reattachment overhead.

### Use screenshots as checkpoints

DevFlow's screenshot call is fast and equal in cost to Appium's (both hit the simulator screenshot API). Screenshots are your evidence — take one before and after key state changes. AI agents that take more screenshots make better diagnoses (see benchmark: DevFlow agent took 3 vs Appium agent's 1, leading to higher-confidence bug identification).

### Keep your MCP tool vocabulary small and purposeful

From the agent benchmark sessions: when the tool surface is clean and well-scoped, AI agents stay on task. When tools are awkward or require boilerplate (like Appium's session create/destroy), agents burn extra tool calls managing infrastructure rather than testing the app. DevFlow's four core MCP tools (`fill`, `query`, `screenshot`, `tap`) were enough for a complete bug-hunt session. Only expose what you need.

### Avoid iOS Simulator keyboard visibility issues on iOS 26.x

On iOS 26.1 + XCUITest, the system keyboard can mark tab bar buttons as not-visible. This affects Appium but is irrelevant to DevFlow (it doesn't use XCUITest). However, be aware that DevFlow's `interact scroll` and `interact tap` assume the target element is in the visible viewport — scroll to bring it into view before tapping if needed.

---

## Available packages

| Package | Purpose |
|---|---|
| `Microsoft.Maui.DevFlow.Agent` | In-app agent (iOS, Android, Mac, GTK) |
| `Microsoft.Maui.DevFlow.Agent.Core` | Platform-agnostic HTTP server and CSS engine |
| `Microsoft.Maui.DevFlow.Agent.Gtk` | GTK/Linux agent |
| `Microsoft.Maui.DevFlow.Blazor` | CDP bridge for Blazor Hybrid |
| `Microsoft.Maui.DevFlow.Blazor.Gtk` | CDP bridge for WebKitGTK |
| `Microsoft.Maui.DevFlow.Driver` | Platform-aware app driver (for test frameworks) |
| `Microsoft.Maui.DevFlow.Logging` | Buffered rotating JSONL logger (no MAUI dependency) |

---

## Caveats

- **Experimental.** Microsoft explicitly states the API will change between releases. Do not depend on specific tool names or parameters without pinning.
- **iOS Simulator and Mac Catalyst only** (currently). Android and Windows are in progress.
- **Debug builds only.** Release builds must not include the agent — verify this in your CI pipeline.
- **No remote device support yet.** The agent runs on localhost; physical devices require additional network routing.
