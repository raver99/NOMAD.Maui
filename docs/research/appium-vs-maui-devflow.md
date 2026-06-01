# Appium vs .NET MAUI DevFlow — Comparative Benchmark

**Last updated:** June 2026  
**Platform:** iOS Simulator, iPhone 17, iOS 26.1  
**App:** BenchmarkApp (this repo) — .NET MAUI 10.0, Debug build  

---

## Background

When an AI agent automates a .NET MAUI app, it reaches the app through a tool stack. Two stacks exist today:

**Appium (established)**
```
AI agent → MCP wrapper → WebDriver (HTTP) → Appium server → XCUITest / UiAutomator2 → OS accessibility tree → app
```

**DevFlow (experimental, Microsoft)**
```
AI agent → MCP (stdio) → maui devflow CLI → in-process HTTP agent → MAUI visual tree
```

The question this benchmark answers: **does it matter which stack you use, and by how much?**

---

## Why the stacks perform differently

The MCP layer is present in both stacks and adds negligible overhead (~1–5 ms per call). The real cost difference lives below MCP:

| Layer | Appium | DevFlow |
|---|---|---|
| Protocol | WebDriver (HTTP, session-based, chatty) | Direct HTTP to localhost agent |
| Element lookup | OS accessibility tree traversal (slow, cross-process) | MAUI visual tree walk (in-process) |
| Text input | XCUITest synthesises keystrokes via accessibility API | Writes directly to MAUI Entry element |
| Process boundary | 3 separate processes: app, Appium server, driver | 1 process: app contains the agent |
| Session concept | Requires explicit session create/destroy per workflow | No session — agent is always running |

---

## What we built

### BenchmarkApp

A purpose-built 3-screen .NET MAUI app with stable `AutomationId` on every interactive element:

| Screen | Key elements |
|---|---|
| **Login** | `UsernameEntry`, `PasswordEntry`, `RememberMeCheckbox`, `LoginButton`, `ValidationMessage` |
| **Items** | `ItemList` (CollectionView, 30 items), each row `Item_N`, `Item_N_Title`, `Item_N_Category` |
| **Settings** | `NotificationsSwitch`, `DarkModeSwitch`, `VolumeSlider`, `StatusLabel` (live-updating) |

**Planted bug:** `LoginViewModel.ExecuteLogin()` has an inverted condition — valid credentials (non-empty username, password ≥ 4 chars) trigger `"Invalid credentials"`, while empty/short credentials show no error. Used as the target for both the scripted harness and agent sessions.

**DevFlow agent** is registered in `MauiProgram.cs` behind `#if DEBUG`. The same binary is used for both stacks — build config is not a variable.

### Two automation harnesses

`Harness/appium/harness_appium.py` — Appium Python client, XCUITest driver  
`Harness/devflow/harness_devflow.py` — `maui devflow ui` CLI subprocess calls  
`Harness/analyze.py` — reads both result JSONs, produces per-step timing table and analysis

Both harnesses run the same logical steps against the same pre-started app process. The app is launched once before any timed run to eliminate cold-start and JIT overhead symmetrically. Between runs, each harness resets UI state (navigate to Login, clear fields) without relaunching the app.

---

## Test 1 — Scripted timing benchmark

### Methodology

- App pre-started once on simulator; neither harness relaunches it
- 6 runs per stack (first run discarded as cold; 5 warm runs averaged)
- Steps 1–5 measured on Appium (steps 6–9 blocked by an iOS 26.1 / XCUITest keyboard-visibility issue unrelated to Appium itself)
- All 9 steps measured on DevFlow
- MCP layer excluded — harnesses call each stack's native API directly, isolating stack overhead

**Appium note:** Each run opens a new XCUITest session (attach to running app via `bundleId` + `noReset: true`). Session startup time is reported separately from scenario steps.

### Results (5 warm runs, steps 1–5)

| Step | Appium mean | DevFlow mean | Speedup |
|---|---|---|---|
| 1. Inspect visual tree | 0.645s ±0.017 | 0.143s ±0.005 | **4.5×** |
| 2. Fill username | 1.321s ±0.008 | 0.131s ±0.027 | **10.1×** |
| 3. Fill password | 1.361s ±0.007 | 0.119s ±0.004 | **11.5×** |
| 4. Tap Login button | 1.115s ±0.004 | 0.417s ±0.003 | **2.7×** |
| 5. Assert + screenshot | 0.231s ±0.004 | 0.210s ±0.008 | 1.1× |
| **Scenario total (1–5)** | **4.673s ±0.027** | **1.019s ±0.039** | **4.58×** |
| XCUITest session startup | 1.009s ±0.029 | — (not applicable) | — |

**Key observations:**

- **Text input is the largest gap (10–11×).** Each `sendKeys` in Appium traverses: WebDriver HTTP → Appium server → XCUITest → OS accessibility layer → app. DevFlow writes directly to the MAUI `Entry` binding.
- **Tree inspection is 4.5× slower** because Appium serialises the full OS accessibility tree; DevFlow walks the MAUI visual tree in-process.
- **Screenshot is essentially equal (1.1×)** — both call the simulator's screenshot API. This is the step where neither approach has a structural advantage.
- **XCUITest session startup (~1s) is a fixed per-workflow cost** Appium always pays. In an AI agentic loop that starts a new session per task, this compounds.
- **DevFlow run-to-run variance is tight (±2.5% total across two independent benchmark runs)**, confirming the numbers are stable.

---

## Test 2 — AI agent sessions

### Methodology

Two completely fresh `claude` subprocess sessions were run — one with only the Appium MCP server configured, one with only the DevFlow MCP server. Neither session had any knowledge of the source code, the planted bug location, or prior benchmark results. The only inputs were:

1. A shared task prompt (identical for both): explore the Login screen, fill in credentials, tap Login, observe the result, report any bug found
2. A stack-specific `CLAUDE.md` describing the available tool vocabulary (no hints about what to look for)

Sessions ran sequentially on the same machine. Timing is wall-clock from first event to last.

### Results

| | DevFlow | Appium |
|---|---|---|
| **Time to find bug** | **34 seconds** | **246 seconds** |
| Total tool calls | 12 | 36 |
| Screenshots taken | 3 | 1 |
| Transcript size | 176 KB | 1,822 KB |
| MCP tools used | `fill`, `query`, `screenshot`, `tap` | `create_session`, `screenshot` |
| Fallback to non-MCP tools | No | Yes (Bash, Glob, Read) |
| Bug correctly identified | ✓ | ✓ |

**DevFlow agent path (12 calls):** Navigate to login → screenshot → fill username → fill password → screenshot → tap button → screenshot → query `ValidationMessage` text → conclude. Clean, linear, four focused MCP tools.

**Appium agent path (36 calls):** Create session → screenshot → hit friction with session/element model → fall back to Bash, Glob, and Read to inspect its environment → eventually interact with the app → screenshot → conclude. Found the bug but via a much noisier path.

**Root cause diagnoses:**
- DevFlow agent: hypothesised a hardcoded credential comparison (incorrect mechanism, correct symptom)
- Appium agent: identified the inverted `if`/`else` branches specifically — a more precise code-level hypothesis

Both diagnoses would lead a developer to the same fix.

---

## Conclusions

### The hypothesis was confirmed

DevFlow is meaningfully faster than Appium for AI-driven MAUI app automation. The gap is not marginal:

- **4.58× faster** on individual action latency (scripted benchmark)
- **7.2× faster** end-to-end for an AI agent completing the same task (agent session benchmark)

### Where the gap comes from

The speed difference is structural, not incidental:

1. **In-process vs cross-process.** DevFlow's agent runs inside the app. Every query is a direct method call on the MAUI visual tree. Appium's driver is a separate process communicating via HTTP with another separate process (Appium server) that then calls XCUITest.

2. **Framework-native element model.** DevFlow selects elements using CSS selectors against MAUI's own tree — the same tree the app uses to render. Appium queries the OS accessibility tree, which is a lossy projection of the native view hierarchy, serialised over XCUITest on each request.

3. **No session lifecycle overhead.** DevFlow's agent is always running inside the app; there is no session to create or destroy. Appium requires session initialisation (~1s) before any automation can begin.

### Tool ergonomics affect AI agent behaviour

The agent session results reveal something the latency numbers alone don't capture: **hard-to-use tools cause AI agents to work less efficiently.** The Appium agent used 3× as many tool calls, took fewer screenshots (less visual evidence), and fell back to filesystem tools when the MCP tools felt insufficient. The DevFlow agent stayed on task with a small, coherent set of purpose-built operations.

This matters in practice: an agent's ability to reason correctly depends on the quality of evidence it gathers. DevFlow's tools produced 3 screenshots with clear before/after state; Appium's produced 1. More screenshots = more evidence = higher confidence diagnosis.

### Caveats

- The Appium steps 6–9 (tab navigation) failed due to an iOS 26.1 / XCUITest interaction where the keyboard marks tab bar buttons as not-visible. This is a known friction point with XCUITest on iOS 26.x, not a fundamental Appium limitation. The login-screen steps (1–5) are sufficient to validate the hypothesis.
- DevFlow is experimental and versions change between releases. Pin package and CLI versions before depending on specific behaviour.
- Both stacks were tested on iOS Simulator only. Performance characteristics on a physical device may differ.

---

## Environment

| Component | Version |
|---|---|
| macOS | Darwin 25.5.0 (Apple Silicon) |
| .NET SDK | 10.0.100 |
| .NET MAUI workload | 10.0.0 (maui-ios) |
| Microsoft.Maui.DevFlow.Agent | 0.1.0-preview.10.26274.3 |
| maui CLI | 0.1.0-preview.10.26274.3 |
| Appium | 2.19.0 |
| appium-xcuitest-driver | 9.9.1 |
| appium-mcp | 1.9.2 |
| Appium Python client | 5.3.1 |
| Xcode | 26.1 (required by .NET iOS workload) |
| Simulator | iPhone 17, iOS 26.1 (UDID: 58157DBC-54EA-431E-8416-475C2B9CAB75) |
| Claude model | claude-sonnet-4-6 |

---

## Repository structure

```
BenchmarkApp/               .NET MAUI app (iOS target, Debug build)
  MauiProgram.cs            DevFlow agent registered here (#if DEBUG)
  Pages/                    LoginPage, ItemListPage, ItemDetailPage, SettingsPage
  ViewModels/               LoginViewModel (contains planted bug), others
  Models/Item.cs

Harness/
  appium/harness_appium.py  Scripted Appium timing harness (steps 1–5)
  devflow/harness_devflow.py Scripted DevFlow timing harness (all 9 steps)
  analyze.py                Produces comparison table from result JSONs

Sessions/
  BugHuntTask.md            Shared task prompt for agent sessions
  appium/CLAUDE.md          Appium session context (no code hints)
  devflow/CLAUDE.md         DevFlow session context (no code hints)
  run_agent_sessions.sh     Launches fresh claude subprocesses per stack

Results/
  results_appium.json       Scripted harness output (Appium)
  results_devflow.json      Scripted harness output (DevFlow, run 1)
  results_devflow_run2.json Scripted harness output (DevFlow, run 2 — reproducibility check)
  benchmark_summary.json    Per-step stats summary
  sessions/                 Agent session transcripts (.jsonl) and timings

mcp_appium.json             MCP server config for Appium (for qualitative agent runs)
mcp_devflow.json            MCP server config for DevFlow
run_benchmark.sh            Builds app and runs both scripted harnesses
```

---

## Running the benchmark yourself

```bash
# 1. Build the app
DEVELOPER_DIR="/Applications/Xcode 26.1.app/Contents/Developer" \
  dotnet build BenchmarkApp -f net10.0-ios -c Debug \
  /p:RuntimeIdentifier=iossimulator-arm64

# 2. Boot simulator and install
xcrun simctl boot 58157DBC-54EA-431E-8416-475C2B9CAB75
xcrun simctl install 58157DBC-54EA-431E-8416-475C2B9CAB75 \
  BenchmarkApp/bin/Debug/net10.0-ios/iossimulator-arm64/BenchmarkApp.app

# 3. Launch app (keeps it running for all runs)
xcrun simctl launch 58157DBC-54EA-431E-8416-475C2B9CAB75 \
  com.companyname.benchmarkapp

# 4. Start Appium
appium server --port 4723

# 5. Run scripted benchmark (6 runs each)
./run_benchmark.sh

# 6. Run agent sessions
./Sessions/run_agent_sessions.sh both
```
