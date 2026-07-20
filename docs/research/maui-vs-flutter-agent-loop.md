# MAUI + DevFlow vs Flutter + Marionette — the AI agent feedback loop

**Last updated:** July 2026
**Status:** In progress — apps built and verified, scenario measurements not yet run.
**Workspace:** `~/Projects/Sample/MauiVsFlutterAgentLoop/`

---

## The question

DevFlow is the eval feedback loop for AI-assisted MAUI development: the agent implements,
then checks the running app against the requirement. If that loop is slow, the whole
development process is slow — which makes it a legitimate framework-selection criterion,
not a tooling detail.

So: **is the MAUI agent loop actually slower than Flutter's, and if so, where exactly?**

Flutter's closest equivalent to DevFlow is [Marionette MCP](https://pub.dev/packages/marionette_mcp)
(LeanCode) — architecturally the same shape: an in-app package (`marionette_flutter`)
plus an MCP server, initialised behind a debug-mode guard, driving the framework's own
widget tree. This is *not* the in-process-vs-cross-process comparison that the
[Appium benchmark](appium-vs-maui-devflow.md) made. Both stacks here are in-process.

---

## What the existing benchmark does not measure

Every number in [`appium-vs-maui-devflow.md`](appium-vs-maui-devflow.md) is taken against a
**pre-started app** — deliberately, to isolate stack overhead. That makes those numbers
valid for what they claim and silent about the loop developers actually feel:

```
edit code → build → install → launch → navigate to the screen → assert
└──────────────────── the part that hurts ───────────────────┘   └─ ~1s ─┘
```

Optimising DevFlow's tool calls optimises the last second. The three costs that dominate
the rest are build/deploy, navigation-to-state, and how the *agent* chooses to work.

---

## Three costs, only one of which is about tooling

| Cost | Whose problem |
|---|---|
| **Build + deploy** | Framework/toolchain |
| **Navigate back to the screen under test** | Usage pattern — fixable on both sides |
| **Agent re-deriving that path visually each run** | Usage pattern — fixable on both sides |

The third is the one most likely to be misread as "the tooling is slow". An agent that
screenshots after every step — including setup steps that are not under test — pays for
observation it does not need. That cost is identical on Flutter, so it is not a framework
difference at all. Measuring it separately is the point of the scenario design below.

---

## Finding 1 — cold build: MAUI is not the slow one

Both apps built clean and launched on the iOS Simulator:

| | MAUI + DevFlow | Flutter + Marionette |
|---|---|---|
| Cold build | 36s | — |
| Install + launch | 5s | — |
| **Total, clean → running** | **41s** | **68s** |
| Automation agent reachable | first attempt, 0 retries | VM service up |

MAUI's cold path was **faster** than Flutter's. This cuts against the "MAUI is just slow"
intuition. The cost only becomes punishing because MAUI pays it *per iteration* where
Flutter pays it once and then hot-reloads.

---

## Finding 2 — hot reload: the capability exists, the trigger does not

The decisive difference is not whether MAUI can hot reload. It can. It is whether an
**agent** can invoke it mid-session.

| | MAUI | Flutter |
|---|---|---|
| IDE + human driving | XAML *and* C# reload, state + nav preserved | equivalent |
| CLI, no IDE | **not on net10 GA** — IDE/debugger only | `flutter run` + `r`, for years |
| **Agent tool call mid-session** | **nothing exists** | `hot_reload` via VM Service RPC |

- MAUI's C# hot reload depends on the Mono interpreter on iOS, which is **already enabled
  by default in Debug builds** — not a blocker.
  ([interpreter docs](https://learn.microsoft.com/en-us/dotnet/maui/macios/interpreter?view=net-maui-9.0))
- MAUI's reload is wired to the Visual Studio debugger's Edit-and-Continue pipeline.
  Flutter's is wired to `reloadSources`, an RPC on the Dart VM Service — externally
  callable over a WebSocket by anything speaking the protocol. That is what Marionette's
  `hot_reload` tool wraps. ([Flutter hot reload](https://docs.flutter.dev/tools/hot-reload))
- **DevFlow exposes no reload capability of any kind.** Its command surface is
  `agent, batch, broker, commands, device, diagnose, extensions, init, list, logs, mcp,
  network, recording, skills, storage, theme, ui, version` — verified against
  `maui devflow --help`, and matching the
  [MCP tool categories](https://learn.microsoft.com/en-us/dotnet/maui/developer-tools/devflow/mcp-server?view=net-maui-10.0).
- **.NET 11 Preview 4 (May 2026)** added `dotnet watch` for MAUI mobile including iOS
  Simulator — file-save triggered, still no force-reload call, and **not GA**. Sourced from
  a community blog rather than Microsoft docs; treat the specifics as softer than the rest.

Flutter's hot reload is not magic either — `main()` edits, static initialisers, enum↔class
changes and any native code change all force a full restart.

**Verdict:** a missing bridge, not a missing capability. `dotnet watch` doing the reload
plus a DevFlow-side tool exposing it would close most of the gap. That is a concrete,
buildable contribution.

---

## Finding 3 — DevFlow has an efficiency surface nobody documented

Verified against `maui devflow ui <cmd> --help`. None of this appeared in our own docs,
and all of it targets exactly the costs above:

| Capability | What it removes |
|---|---|
| `--and-screenshot`, `--and-tree` on `tap`/`fill` | Action + observation in **one** call instead of two round trips |
| `query --wait-until exists --timeout N` | Waiting for async loads **without** screenshot-polling |
| `tree --format compact`, `--fields`, `--depth` | Tree dumps small enough not to flood agent context |
| `scroll --item-index N` | Jump straight to item N in a CollectionView — no repeated scroll-and-look |
| `maui devflow batch` (JSONL over stdin) | A whole setup sequence as **one** call, no per-step model round trip |

`batch` is the direct answer to "bring the app to the screen with a hardcoded script
instead of having the model re-derive it". Marionette has no documented equivalent, so
this may be a place MAUI can genuinely win.

**Caveat:** these are documented flags, not yet measured. Finding 4 is why that distinction
matters.

---

## Finding 4 — our own docs were wrong, and that is a plausible speed bug

Every DevFlow command in `docs/maui-devflow.md` and the `maui-devflow` skill used a
non-existent `MAUI` subcommand:

```bash
# documented (does not exist)        # actual
maui devflow MAUI tree               maui devflow -p ios ui tree
maui devflow agent interact tap …    maui devflow ui tap --automationId "LoginButton"
```

`agent` exists but only discovers and inspects connected agents; there is no `interact`
subcommand. So every documented tap/fill/scroll/navigate call would have failed.

This is not just a docs defect — **it is a candidate root cause for the observed slowness.**
An agent following the skill burns turns on failing commands and error recovery before
doing any real work, on every session. That is indistinguishable from "DevFlow is slow".

Fixed in the same change as this document.

---

## Finding 5 — a misleading error that costs turns

Shell `Tab` elements do not resolve via `--automationId`, though the CSS selector form finds
them and regular controls resolve fine either way:

```bash
maui devflow -p ios ui query --automationId "SettingsTab"   # → []
maui devflow -p ios ui query --selector "#SettingsTab"      # → found
maui devflow -p ios ui tap --automationId "SettingsTab"     # → error: "Check automationId spelling"
maui devflow -p ios ui tap --text "Settings" --type Tab     # → success
```

The cost isn't the failed call — it's the **suggestion**. The error tells the agent to check
spelling when the spelling is correct, so the agent re-dumps the tree and re-reads the XAML
before finding the real cause. Several wasted turns, reproducibly, every session that touches
a tab.

This is the shape of the problem worth generalising: **agent loop speed is dominated by
recoverable-error paths, not by action latency.** A tool call that fails informatively costs
one turn; one that fails misleadingly costs five. No latency benchmark captures that
difference, which is part of why the original benchmark showed DevFlow winning while the
lived experience was of slowness.

---

## Measured results (scripted, no AI agent)

Deterministic scripts, 6 runs each, first discarded as cold, 5 warm runs averaged. Both
stacks drive the same task on mirrored apps: reach the deep bug (Project 137 → first
document) and read the revision count. **Wall-clock**, so blocking waits and polling sleeps
both count. Every run's assertion was verified correct (`"Total revisions: 4"` against 5
actual rows).

### Navigate → assert (warm app, optimized both sides)

| | MAUI + DevFlow | Flutter + Marionette |
|---|---|---|
| Wall clock | **2.69s** ±0.11 | 7.90s ±0.01 |
| Tool calls | **12** | 66 |
| Payload returned | **3 KB** | 561 KB |

MAUI wins this decisively, for two structural reasons:

1. **Virtualized-list addressing.** `ui scroll --item-index 136` jumps straight to row 137 in
   one call. Marionette's `scroll_to` only resolves widgets the `ListView.builder` has
   already built — roughly 9 at a time — and errors with `Extension marionette.scrollTo
   failed` for anything further. Reaching row 137 requires ~15 swipe-and-look iterations.
2. **Scoped queries.** DevFlow's CSS selectors allow `ui query --selector
   "Grid[automationId^=Revision_]"` — 3 KB. Marionette's `get_interactive_elements` takes
   **no parameters**, so every observation returns the whole screen. Switching MAUI's row
   count from a full tree dump to a scoped selector cut payload from 188 KB to 3 KB at
   identical speed — a 60× context reduction with no downside.

### Edit → verify (the loop that matters)

| | MAUI + DevFlow | Flutter + Marionette |
|---|---|---|
| Apply change | build **4.9s** + deploy **4.4s** | `hot_reload` **0.26s** |
| Navigate back + assert | 4.1s | 8.2s |
| **Total** | **13.4s** | **8.5s** |

**The reload step is ~35× faster on Flutter** (0.26s vs 9.3s). That is the structural gap,
and it is real.

But the end-to-end loop is only **1.6× faster**, because Flutter gives back most of the
advantage in navigation. Two findings that each looked decisive in isolation substantially
cancel.

Two things that surprised me, both cutting against my own predictions:

- **MAUI's incremental build is ~4.9s, not 36s.** The 36s figure is a *cold* build. I had
  predicted ~41s per iteration; the real per-edit cost is a quarter of that. Anyone reasoning
  about MAUI's loop from cold-build numbers is overestimating the pain by roughly 4×.
- **Hot reload does not remove the navigation cost.** It preserves app state, but the agent
  still has to get back to the screen and re-observe it. When navigation is the expensive
  part — as it is with Marionette on a long list — a near-instant reload buys less than
  expected.

### Cold agent sessions (fresh `claude` subprocess + tuned per-stack guide)

Each session is a fresh subprocess with no knowledge of the app, the bug, or the element
names — but with a stack-specific `CLAUDE.md` encoding the knowledge a team would have in
practice. Both guides were written to the same structure and depth. Sessions ran with an
explicit tool allowlist, no permission bypass. Task: confirm the deep bug and hypothesise a
cause, without reading source first.

| | MAUI + DevFlow | Flutter + Marionette |
|---|---|---|
| Wall clock | 88s, 122s (**~105s**) | 167s, 117s (~142s) |
| Tool calls | 13, 21 (**17**) | 50, 45 (47) |
| Bug found correctly | 2/2 | 2/2 |

Both stacks found the bug every valid run, and both produced good root-cause hypotheses —
several sessions independently guessed `length - 1` and ran their own controls (checking a
second document, ruling out list clipping and virtualization).

**The tool-call gap is the solid result** — 13–21 vs 45–50, with no overlap, and it traces
directly to the two scripted findings: Marionette burns 20+ swipes walking to row 137 and
cannot scope a query. The wall-clock gap is **weak evidence**: n=2 per stack, ranges of
88–122s and 117–167s that nearly touch, against a stochastic model. Directionally it favours
MAUI; it should not be quoted as a ratio.

**Three runs were discarded, and why it matters.** The first three Flutter sessions finished
in 46–59s with as few as 15 calls — an apparent 2–3× win. They were invalid. The Flutter app
keeps per-tab `Navigator` stacks *and* list scroll offsets alive in an `IndexedStack`, so
previous runs left the app parked four screens deep with the list already scrolled to row
137. Sessions started mid-chain and skipped the navigation being measured; one session's own
report gave it away by mentioning it had "backed out to the document list". Resetting
required entering the tab, popping it via the AppBar back button (`press_back_button` is
Android-only and silently does nothing on iOS), *and* swiping the list back to the top.
Once fixed, the same benchmark inverted from Flutter winning to MAUI winning.

The general lesson is worth more than the numbers: **a benchmark that does not verify its
own starting state will measure leftover state instead of the thing under test**, and it
fails in the flattering direction — the stack whose app was left closest to the target looks
fastest.

## Extended scenarios (7 usage shapes)

The first round measured one edit shape and one read-only inspection. These seven cover the
rest of what an agent actually does while building and reviewing a feature.

### 1. Rude-edit ladder — does hot reload survive structural change?

Each rung verifies the change **took effect**, not merely that reload reported success —
Flutter returns "Hot reload completed successfully" for edits that never apply.

| Edit shape | Flutter hot reload | MAUI rebuild+deploy |
|---|---|---|
| Expression inside build | 0.22s ✓ | ~9s |
| Add a widget | 0.22s ✓ | ~9s |
| Add a field to a State class | 0.22s ✓ | ~9s |
| Add a top-level const | 0.22s ✓ | ~9s |
| Add an enum | 0.22s ✓ | ~9s |
| **Add a whole new file + import** | **0.22s ✓** | ~9s |
| Convert enum → class | ✗ restart required | ~9s |

**Prediction falsified.** I expected Flutter's advantage to collapse on structural edits. It
does not: reload survives everything a normal feature iteration does, including adding a new
source file, and only breaks on the exotic enum↔class conversion. MAUI is flat at ~9s
regardless of edit shape — it has no cliff to fall off because it rebuilds every time.

**The ~40× reload advantage is real and general.** The case for a hot-reload bridge is
stronger than the first round suggested, not weaker.

### 2. Diagnostics — logs and network

Planted a log line, a caught exception, and an HTTP call returning 503.

| | MAUI + DevFlow | Flutter + Marionette |
|---|---|---|
| Logs, zero config | ✅ `Console.WriteLine` captured | ❌ `get_logs` errors until a `LogCollector` is wired |
| Logs after setup | 13.3 KB / 300 entries (raw stream) | 142 bytes / 3 entries (curated) |
| What gets captured | everything on stdout | **only explicit `addLog` calls** — `print` and `developer.log` are not |
| Native HTTP traffic | ❌ **`network list` empty** | ❌ no network tool at all |

Two findings worth separating:

- **DevFlow's network monitoring did not capture native `HttpClient` traffic.** The app
  logged `DIAG_HTTP_STATUS 503`, proving the request happened; `network list` returned
  nothing. The documented "Network — HTTP request/response monitoring" capability appears to
  apply to Blazor/WebView (CDP) traffic, not native .NET HTTP. Neither stack gives an agent
  visibility into native network calls.
- **Marionette's failure message is a model of what DevFlow's are not.** Unconfigured
  `get_logs` returns a message naming three concrete fixes with package names and code.
  Compare DevFlow's `tap --automationId` on a tab, which says *"Check automationId spelling"*
  when the spelling is correct. Same class of error; opposite cost to an agent.

Also: `maui devflow network` (bare) is a streaming watch that never returns — the one-shot is
`network list`. A blocking command with no output looks exactly like a hang.

### 3–5. Form input, state mutation, regression sweep

| | MAUI + DevFlow | Flutter + Marionette |
|---|---|---|
| **Form** (2 fields, checkbox, submit, assert) | 1.02s, 8 calls, 1 KB | 0.82s, 6 calls, 8 KB |
| **State** (2 toggles + slider + computed label) | 1.01s, 8 calls, 1 KB | 0.73s, 5 calls, 12 KB |
| **Sweep** (18 assertions across 4 screens) | 3.64s, 26 calls, 6 KB | 2.19s, 8 calls, 42 KB |
| Sweep, one-dump-per-screen strategy | **1.74s, 12 calls, 475 KB** | (only strategy available) |

Flutter is faster on all three in wall-clock. But the sweep is the interesting one:

- Marionette needs one `get_interactive_elements` per screen to answer many assertions, so
  its call count is low by construction. DevFlow's per-selector queries cost more calls but
  return 7× less data.
- Giving DevFlow the same strategy (one compact tree per screen) makes it the fastest option
  measured — 1.74s — but at **475 KB**, because MAUI's tree dumps the whole visual tree while
  Marionette reports only *interactive* elements.
- So DevFlow uniquely offers a **speed-versus-context dial**; Marionette has one setting, and
  its default payload is far leaner than MAUI's full tree.

**Capability gap:** DevFlow's `ui set-property` can set a slider's `Value` directly (verified:
status label went to "Volume: 80%"). Marionette has no property-mutation tool — a slider can
only be driven by gesture.

### 6. Visual / layout review

Planted an overflowing row in both apps.

| | MAUI + DevFlow | Flutter + Marionette |
|---|---|---|
| Framework detects it | ❌ silent, 0 log mentions | ✅ "A RenderFlex overflowed by 371 pixels on the right" |
| …reachable via the agent's tools | — | ❌ console only; **not** in `get_logs` |
| Overflowing element in the dump | ✅ present, bounds `x:380 w:315` vs parent right edge 378 | ❌ absent entirely |
| Detectable by bounds arithmetic | ✅ | ❌ |
| Screenshot | 0.67s, 24 KB **to disk** | 0.20s, 160 KB **inline into context** |

Complementary and both imperfect: Flutter *knows* precisely what's wrong but won't tell the
agent through its own tooling; MAUI says nothing but exposes enough geometry to derive it.
Note MAUI reported `isVisible: true` for an element clipped off-screen — a misleading signal.

The screenshot difference compounds over a session: DevFlow writes a file the agent may
choose to read; Marionette's image lands in context whether needed or not.

### 7. Repeatable artifact

| | MAUI + DevFlow | Flutter + Marionette |
|---|---|---|
| Batch/script runner | ✅ `batch` | ❌ none |
| Recording | ✅ `recording start/stop` | ❌ none |
| Machine-readable command schema | ✅ `commands` (incl. a `mutating` flag) | MCP `tools/list` only |

`maui devflow batch` takes **plain CLI lines on stdin** (not JSONL input, despite emitting
JSONL results) and returns `{command, exit_code, output}` per line. A 12-command deep-bug
regression ran in one invocation and asserted correctly.

**`--delay 0` matters:** the same script took 4.60s at the default inter-command delay versus
**2.07s** at `--delay 0` — and 2.07s beats the 2.69s of running those 12 commands as separate
CLI processes. Left at the default, batch looks slower than not batching.

### What the seven change

The first round read as "Flutter wins the reload, MAUI wins the querying." The extended set
sharpens both halves:

- Flutter's reload advantage is **more general** than assumed (survives new files).
- MAUI's tooling advantage is **narrower** than assumed: it wins decisively on virtualized-list
  navigation, property mutation, batching, and context economy — but Marionette is faster and
  fewer-calls on ordinary screen-level work, and its curated element dumps beat MAUI's full
  tree for broad assertions.
- Both stacks have a **blind spot in the same place**: native network traffic, and
  agent-reachable layout diagnostics.

### Not measured / unreliable

- **Naive baseline.** Intended as the "drift cost" number. The scripted naive path
  (full tree + screenshot after every step, fixed sleeps, blind `--dy` scrolling) reached
  47.4s / 193 calls / **35.6 MB** before failing to complete its assertion — `ui scroll --dy`
  did not advance the `CollectionView` and the full `ui tree` format did not contain the
  target row that `--format compact` did. The magnitude is indicative and the direction is
  clear, but **the run never completed the task, so treat these as a floor, not a result.**
- **Hot-reload state preservation.** The harness resets to the root before each run, so the
  post-reload state check only ever observed the root screen. Flutter documents that hot
  reload preserves state; this benchmark did not verify it.

---

## Benchmark design

### The apps

Two behaviourally identical apps built from a shared spec
(`MauiVsFlutterAgentLoop/SPEC.md`), with every MAUI `AutomationId` mirrored as a Flutter
`ValueKey` of the same name. The MAUI app is a copy of the Appium benchmark app, so that
benchmark's published results stay valid.

Both carry **two planted bugs at different depths**, which is what lets us measure how
agent cost scales with navigation depth:

- **Shallow** — inverted login validation. 0 navigations.
- **Deep** — `RevisionCountLabel` off-by-one, disagreeing with the visible list.
  4 navigations, 3 async waits, and a scroll into a 200-row list.

The deep chain is `ProjectList (200) → ProjectDetail → DocumentList → DocumentDetail`,
each screen loading asynchronously with a 400 ms delay behind a `LoadingIndicator`.
The latency is deliberate: a fast tool that the agent polls three times while a spinner
spins is not fast in practice, and an instant-load app cannot expose that.

### The three scenarios

Run each on both stacks, against both the shallow and the deep bug:

| | What it isolates |
|---|---|
| **A. Scripted** | Pure tooling speed, agent removed |
| **B. Agent, naive** | The observed behaviour — model re-derives the path each run |
| **C. Agent + scripted setup** | `batch` / equivalent brings the app to the screen |

- **B − C** on each stack = what better *usage* buys, independent of framework.
- **C(MAUI) − C(Flutter)** = what is genuinely *framework*.

Conflating those two is why "MAUI feels slow" has been hard to act on.

---

## Status

- [x] Both apps built, running, verified drivable by their respective agents
- [x] Deep navigation chain + second planted bug
- [x] Scripted navigate → assert, both stacks
- [x] Scripted edit → verify, both stacks
- [ ] Naive baseline — attempted, did not complete (see above)
- [x] Cold agent sessions with tuned per-stack guide (n=2 valid per stack — thin)
- [ ] Hot-reload bridge spike (`dotnet watch` + DevFlow tool)

---

## Environment

| Component | Version |
|---|---|
| macOS | Darwin 25.5.0 (Apple Silicon) |
| .NET SDK | 10.0.100 |
| Microsoft.Maui.Cli | 0.1.0-preview.10.26274.3 |
| Flutter | 3.38.5 (Dart 3.10.4) |
| marionette_flutter / marionette_mcp | 0.5.0 (pinned to match) |
| Simulators | iPhone 17 Pro (MAUI), iPhone 16 Pro (Flutter), iOS 26.x |
