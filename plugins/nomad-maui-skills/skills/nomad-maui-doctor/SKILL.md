---
name: nomad-maui-doctor
description: Check .NET MAUI project health — environment tools, workloads, project config, and architecture best practices
---

Run a comprehensive health check on the .NET MAUI project. Produce a formatted report with PASS / WARN / FAIL / INFO for each item.

**Argument:** $ARGUMENTS — optional:
- `init` — force re-initialization (re-ask the project profile questions, even if a profile already exists).
- `env` — environment/tools checks only.
- `project` — project config + architecture checks only.
- omit — run initialization if needed, then all checks.

---

## How to run

1. **Run Step 0 first** to load (or create) the project profile.
2. Then work through each section in order. For every check, run the indicated bash command, evaluate the result against the criteria, and record the status. Checks tagged **[profile]** change behavior or are skipped based on the profile.
3. After all checks, print the full report.

---

## Step 0 — Project profile (run first)

The doctor adapts to how *this* project is actually set up, using a profile stored at `.nomad/doctor.json` (committed to git, so the whole team shares one check policy).

Load it and decide whether to initialize:
```bash
cat .nomad/doctor.json 2>/dev/null || echo "NO PROFILE"
```

Enter **Initialization Mode** if the file is missing/unparseable **OR** `$ARGUMENTS` contains `init`. Otherwise, parse the JSON and continue to Section 1.

### Initialization Mode

Ask the user the following with the **AskUserQuestion** tool. When re-initializing (`init`), pre-fill the current profile values as the defaults so the user only changes what's needed.

1. **Target platforms** *(multi-select)* — `iOS`, `Android`. Gates the iOS toolchain checks (Xcode, simulators) and the Android toolchain checks (Android SDK, Java).
2. **UI automation stack** *(single-select)* — `MAUI DevFlow`, `Appium`, `Both`, `None`. Gates the DevFlow, Appium, and Node.js checks and the DevFlow-agent registration check.
3. **Optional dev tools in use** *(multi-select)* — `MAUI Sherpa`, `MAUI Skills plugin`. A tool selected here is expected on the machine; one not selected is skipped entirely.

Then write the profile (create the folder first):
```bash
mkdir -p .nomad
```
Write `.nomad/doctor.json` in this shape (use today's date; map the answers to the enums shown):
```json
{
  "version": 1,
  "initializedAt": "YYYY-MM-DD",
  "platforms": ["ios", "android"],
  "uiAutomation": "devflow",
  "optionalTools": ["maui-skills"]
}
```
- `platforms`: any subset of `["ios", "android"]`
- `uiAutomation`: one of `"devflow"`, `"appium"`, `"both"`, `"none"`
- `optionalTools`: any subset of `["maui-sherpa", "maui-skills"]`

Confirm the file was written, tell the user it's committed to git and can be changed anytime via `/nomad-maui-doctor init`, then continue to the checks.

### How the profile gates checks

A check tagged **[profile]** below reads the loaded profile:
- A tool/platform the project **declares it uses** but that is **missing** → escalate severity (build-critical tools → **FAIL**; convenience tools → **INFO**, as each check notes).
- A tool/platform the project **does not use** → **skip** the check entirely (do not run the command, do not print a report row).

If the profile somehow failed to load, fall back to running every check at its default (non-escalated) severity.

---

## Section 1 — Environment & Tools

### .NET SDK
```bash
dotnet --version 2>/dev/null || echo "NOT FOUND"
```
- PASS if version ≥ 9.0
- WARN if version is 8.x (outdated but functional)
- FAIL if not found or < 8.0

### .NET MAUI Workloads
```bash
dotnet workload list 2>/dev/null
```
Look for `maui-ios` and `maui-android` in the output.
- PASS if both are installed
- WARN if only one is installed (note which is missing)
- FAIL if neither is installed
- WARN additionally if any workload shows "update available" — note the update command: `dotnet workload update`

### Xcode  **[profile]**
Only run if `ios` is in `platforms`; otherwise skip (don't print a row).
```bash
xcodebuild -version 2>/dev/null || echo "NOT FOUND"
```
- PASS if Xcode 16.x or newer
- WARN if Xcode 15.x (may work but not recommended)
- FAIL if not found
Also run:
```bash
xcode-select -p 2>/dev/null || echo "NOT SET"
```
- WARN if output is not pointing to a real Xcode.app (e.g. points to CLT-only path)

### iOS Simulators  **[profile]**
Only run if `ios` is in `platforms`; otherwise skip.
```bash
xcrun simctl list devices available 2>/dev/null | grep -c "iPhone\|iPad" || echo "0"
```
- PASS if count ≥ 1
- WARN if 0 — no simulators available; create one via `xcrun simctl create`

### Android SDK  **[profile]**
Only run if `android` is in `platforms`; otherwise skip.
```bash
adb --version 2>/dev/null || echo "NOT FOUND"
```
- PASS if found
- FAIL if not found (Android is a declared target platform)

```bash
echo "ANDROID_HOME=${ANDROID_HOME:-NOT SET}" && echo "ANDROID_SDK_ROOT=${ANDROID_SDK_ROOT:-NOT SET}"
```
- WARN if both `ANDROID_HOME` and `ANDROID_SDK_ROOT` are unset

### Java / JDK  **[profile]**
Only run if `android` is in `platforms` (the JDK is needed for the Android toolchain); otherwise skip.
```bash
java --version 2>/dev/null || echo "NOT FOUND"
```
- PASS if Java 17+
- WARN if Java 11–16 (minimum for Android builds is 17)
- FAIL if not found

### Git
```bash
git --version 2>/dev/null || echo "NOT FOUND"
```
- PASS if found (any version)
- FAIL if not found

### MAUI DevFlow CLI  **[profile]**
Run only if `uiAutomation` is `devflow` or `both`; otherwise skip.
```bash
maui --version 2>/dev/null || echo "NOT FOUND"
```
- PASS if found (any version)
- FAIL if not found — the project's automation stack relies on DevFlow. Install via: `dotnet tool install -g Microsoft.Maui.DevFlow` (see `docs/appium-vs-maui-devflow.md`)

### Appium  **[profile]**
Run only if `uiAutomation` is `appium` or `both`; otherwise skip.
```bash
appium --version 2>/dev/null || echo "NOT FOUND"
```
- PASS if version 2.x
- WARN if version 1.x (deprecated)
- FAIL if not found — the project's automation stack relies on Appium. Install via: `npm install -g appium`

### Node.js  **[profile]**
Run only if `uiAutomation` is `appium` or `both` (Appium requires Node); otherwise skip.
```bash
node --version 2>/dev/null || echo "NOT FOUND"
```
- PASS if v18+
- WARN if found but < v18
- FAIL if not found — required by Appium

### MAUI Sherpa  **[profile]**
Run only if `maui-sherpa` is in `optionalTools`; otherwise skip.
GUI desktop app — has no CLI, so detect the installed app bundle or Homebrew cask:
```bash
ls -d "/Applications/MAUI Sherpa.app" "$HOME/Applications/MAUI Sherpa.app" 2>/dev/null | head -1 || { brew list --cask maui-sherpa >/dev/null 2>&1 && echo "installed (brew cask)"; } || echo "NOT FOUND"
```
- PASS if an app bundle or brew cask is found
- INFO if not found — optional dev-machine tool for managing Android SDKs, Apple certs/profiles, emulators, keystores & device inspectors (see `docs/maui-sherpa.md`). Install via: `brew install --cask redth/tap/maui-sherpa` (macOS) or download from the GitHub releases page.

### MAUI Skills plugin  **[profile]**
Run only if `maui-skills` is in `optionalTools`; otherwise skip.
AI-agent skill pack — not a CLI on PATH; it installs as a Claude Code / Copilot CLI plugin. Detect via the Claude Code plugin registry or cached marketplace clone:
```bash
grep -l "maui-skills" ~/.claude/plugins/installed_plugins.json ~/.claude/plugins/known_marketplaces.json 2>/dev/null || find ~/.claude/plugins/cache -maxdepth 2 -name maui-skills 2>/dev/null | grep . || echo "NOT FOUND"
```
- PASS if the plugin or its marketplace is found
- INFO if not found — optional reference resource: ~37 on-demand .NET MAUI / Xamarin-migration skills for AI coding agents (see `docs/maui-skills.md`). Install via: `/plugin marketplace add davidortinau/maui-skills` then `/plugin install maui-skills@maui-skills` (full steps in `docs/maui-skills-usage.md`). Detection only covers Claude Code; on Copilot CLI verify with `/skills`.

### Syncfusion MAUI Skills  **[auto-detected]**
Not profile-gated — this check keys off whether the project actually uses Syncfusion controls. First detect Syncfusion usage from any `.csproj`:
```bash
grep -rl "Syncfusion\.Maui" --include="*.csproj" . 2>/dev/null || echo "NOT USED"
```
- If the project does **not** reference `Syncfusion.Maui.*` → **skip** this check (don't print a row).
- If it **does**, check whether the Syncfusion skills are installed, project-level or globally (best-effort across common Skills-CLI locations):
```bash
{ find . -path ./.git -prune -o -type d -name 'syncfusion-maui-*' -print 2>/dev/null; find ~/.claude/skills ~/.config/skills "$HOME/.skills" -type d -name 'syncfusion-maui-*' 2>/dev/null; } | head -1 | grep . || echo "NOT INSTALLED"
```
  - PASS if at least one `syncfusion-maui-*` skill directory is found (project or global).
  - WARN if none found — the project uses Syncfusion controls but the matching agent skills aren't installed. Install via: `npx skills add syncfusion/maui-ui-components-skills` (requires Node ≥ 16; choose project-level to commit them or global to share across projects — see `docs/syncfusion-maui-skills.md`).

---

## Section 2 — Project Configuration

Find all .csproj files:
```bash
find . -name "*.csproj" -not -path "*/.git/*"
```
Read each .csproj found for the checks below.

### ApplicationId — not default
- FAIL if `ApplicationId` value contains `com.companyname` (still the scaffolded default)
- PASS otherwise
- Fix: change to a real reverse-domain ID like `com.yourcompany.nomad`

### ApplicationIdGuid — present
- FAIL if `ApplicationIdGuid` element is missing
- PASS if a GUID is set

### TargetFrameworks — match the profile  **[profile]**
Compare the `.csproj` TargetFrameworks against the profile's `platforms`:
- For each platform in `platforms` (`ios` → `net*-ios`, `android` → `net*-android`): PASS if a matching target (e.g. `net9.0-ios`/`net10.0-ios` or newer) is present; **FAIL** if the declared platform has no matching target.
- If a target exists for a platform **not** in `platforms` (e.g. an `-ios` target but iOS wasn't selected) → **WARN** (profile/project mismatch — re-run `init` or fix the csproj).

### SupportedOSPlatformVersion  **[profile]**
Only check the versions for platforms in the profile:
- iOS (if `ios` in `platforms`): WARN if missing or < 14.2 (recommended: ≥ 14.2)
- Android (if `android` in `platforms`): WARN if missing or < 21 (recommended: ≥ 21.0)

### Application Version numbers
- INFO if both `ApplicationDisplayVersion` = `1.0` and `ApplicationVersion` = `1` — likely never updated from scaffold defaults

### App Branding
Check `<MauiIcon>` and `<MauiSplashScreen>` elements:
- WARN if `Color="#512BD4"` on the icon — still using default MAUI purple
- WARN if `Color="#512BD4"` on the splash screen — still using default MAUI purple

---

## Section 3 — Architecture & Best Practices

### Solution file
```bash
find . -name "*.sln" -not -path "*/.git/*"
```
- PASS if found
- WARN if not found

### .gitignore
```bash
test -f .gitignore && echo "FOUND" || echo "NOT FOUND"
```
- PASS if found
- WARN if not found

### MVVM — ViewModels directory
```bash
find . -type d -name "ViewModels" -not -path "*/.git/*"
```
- PASS if found
- WARN if not found — MVVM pattern requires a ViewModels directory; see CLAUDE.md for architecture guidelines

### Pages directory
```bash
find . -type d -name "Pages" -not -path "*/.git/*"
```
- PASS if found
- WARN if not found — pages should live in a dedicated `Pages/` directory

### Services directory
```bash
find . -type d -name "Services" -not -path "*/.git/*"
```
- PASS if found
- INFO if not found — optional until service layer is needed

### Dependency Injection in MauiProgram.cs
Read `MauiProgram.cs` (search for it if not at root):
```bash
find . -name "MauiProgram.cs" -not -path "*/.git/*"
```
Then check whether `builder.Services` appears in the file.
- PASS if `builder.Services` is used (DI is configured)
- WARN if not found — all ViewModels and services should be registered via DI

### AutomationId coverage
```bash
grep -r "AutomationId" --include="*.xaml" . 2>/dev/null | wc -l
```
Also count total interactive XAML elements for context:
```bash
grep -rEc "(Button|Entry|Switch|Slider|CheckBox|Picker|DatePicker|TimePicker|SearchBar|Editor|CollectionView|ListView)" --include="*.xaml" . 2>/dev/null || echo "0"
```
- PASS if AutomationId count > 0 and covers most interactive elements
- WARN if AutomationId count = 0 — required for UI automation; every interactive element needs one
- WARN if AutomationId count << interactive element count — some elements are missing IDs

### DevFlow Agent registration  **[profile]**
Run only if `uiAutomation` is `devflow` or `both`; otherwise skip.
Read `MauiProgram.cs` and check for a `#if DEBUG` block that registers a DevFlow or automation agent.
- PASS if found
- WARN if not found — the project uses DevFlow, so the agent should be registered under `#if DEBUG` (see `docs/appium-vs-maui-devflow.md`)

### CommunityToolkit.Mvvm  *(recommended)*
```bash
grep -r "CommunityToolkit.Mvvm" --include="*.csproj" . 2>/dev/null | head -3
```
- PASS if found
- INFO if not found — CommunityToolkit.Mvvm is recommended for MVVM boilerplate (ObservableObject, RelayCommand, etc.)

---

## Section 4 — Report

Print a formatted report like this:

```
╔══════════════════════════════════════════════════╗
║          NOMAD.Maui — Doctor Report              ║
║          [today's date]                          ║
╚══════════════════════════════════════════════════╝

PROFILE  (.nomad/doctor.json — re-run with `init` to change)
──────────────────────────────────────────────────
Platforms        iOS, Android
UI automation    devflow
Optional tools   maui-skills

ENVIRONMENT & TOOLS
──────────────────────────────────────────────────
✅ .NET SDK              [version]
✅ MAUI workloads        maui-ios ✓  maui-android ✓
✅ Xcode                 [version]
✅ iOS Simulators        [N] available
❌ Android SDK           adb not found
✅ Java                  [version]
✅ Git                   [version]
ℹ️  MAUI DevFlow CLI      not installed  (optional — see docs/appium-vs-maui-devflow.md)
ℹ️  MAUI Sherpa          not installed  (optional — see docs/maui-sherpa.md)
ℹ️  MAUI Skills plugin    not installed  (optional — see docs/maui-skills.md)
⚠️  Syncfusion skills     project uses Syncfusion — skills not installed  (npx skills add syncfusion/maui-ui-components-skills)

PROJECT CONFIGURATION
──────────────────────────────────────────────────
❌ ApplicationId         com.companyname.nomad.maui  ← still default
✅ ApplicationIdGuid     [guid]
✅ TargetFrameworks      net9.0-ios  net9.0-android
✅ SupportedOSPlatform   iOS 14.2  Android 21
ℹ️  App version          1.0 / 1  (still scaffold defaults)
⚠️  Branding (icon)      still default purple #512BD4
⚠️  Branding (splash)    still default purple #512BD4

ARCHITECTURE & BEST PRACTICES
──────────────────────────────────────────────────
✅ Solution file         [name].sln
✅ .gitignore            present
⚠️  ViewModels dir       not found
⚠️  Pages dir            not found
ℹ️  Services dir         not found  (optional)
⚠️  DI in MauiProgram    builder.Services not configured
⚠️  AutomationIds        0 found in XAML (N interactive elements without IDs)
ℹ️  DevFlow agent        not registered  (optional)
ℹ️  CommunityToolkit     not referenced  (optional but recommended)

──────────────────────────────────────────────────
SUMMARY   ✅ [N] pass   ⚠️ [N] warn   ❌ [N] fail   ℹ️ [N] info
──────────────────────────────────────────────────

ACTIONS REQUIRED (FAIL)
  • Change ApplicationId from default — edit NOMAD.Maui.csproj

SHOULD FIX (WARN)
  • Create ViewModels/ directory and move logic out of code-behind
  • Create Pages/ directory and move page XAML files there
  • Add builder.Services registrations in MauiProgram.cs
  • Add AutomationId to every Button, Entry, Switch, etc. in XAML
  • Customise app icon and splash screen colours

OPTIONAL (INFO)
  • Install MAUI DevFlow CLI for AI-driven automation
  • Register DevFlow agent in MauiProgram.cs (#if DEBUG)
  • Add CommunityToolkit.Mvvm NuGet package
```

Adapt the table rows to only show items that actually apply — skip sections with no findings, and **omit any check skipped by the profile** (e.g. no iOS rows when iOS isn't a target platform; no Appium row unless the automation stack uses it). The PROFILE banner reflects the values loaded from `.nomad/doctor.json`. Always close with the actions list.
