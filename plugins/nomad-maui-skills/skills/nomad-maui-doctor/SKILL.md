---
name: nomad-maui-doctor
description: Check .NET MAUI project health — environment tools, workloads, project config, and architecture best practices
---

Run a comprehensive health check on the .NET MAUI project. Produce a formatted report with PASS / WARN / FAIL / INFO for each item.

**Argument:** $ARGUMENTS — optional filter: `env` (environment/tools only), `project` (project config + architecture only), or omit for all checks.

---

## How to run

Work through each section below in order. For every check, run the indicated bash command, evaluate the result against the criteria, and record the status. After all checks, print the full report.

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

### Xcode
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

### iOS Simulators
```bash
xcrun simctl list devices available 2>/dev/null | grep -c "iPhone\|iPad" || echo "0"
```
- PASS if count ≥ 1
- WARN if 0 — no simulators available; create one via `xcrun simctl create`

### Android SDK
```bash
adb --version 2>/dev/null || echo "NOT FOUND"
```
- PASS if found
- FAIL if not found and `net*-android` is present in any .csproj TargetFrameworks
- INFO if not found and Android is not targeted

```bash
echo "ANDROID_HOME=${ANDROID_HOME:-NOT SET}" && echo "ANDROID_SDK_ROOT=${ANDROID_SDK_ROOT:-NOT SET}"
```
- WARN if both `ANDROID_HOME` and `ANDROID_SDK_ROOT` are unset

### Java / JDK
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

### MAUI DevFlow CLI  *(optional)*
```bash
maui --version 2>/dev/null || echo "NOT FOUND"
```
- PASS if found (any version)
- INFO if not found — DevFlow is optional; it enables 4–7× faster AI-driven UI automation vs Appium (see `docs/appium-vs-maui-devflow.md`). Install via: `dotnet tool install -g Microsoft.Maui.DevFlow`

### Appium  *(optional)*
```bash
appium --version 2>/dev/null || echo "NOT FOUND"
```
- PASS if version 2.x
- WARN if version 1.x (deprecated)
- INFO if not found — optional, only needed for XCUITest/UiAutomator2 automation

### Node.js
```bash
node --version 2>/dev/null || echo "NOT FOUND"
```
- PASS if v18+
- WARN if found but < v18
- INFO if not found — only required if using Appium

### MAUI Sherpa  *(optional)*
GUI desktop app — has no CLI, so detect the installed app bundle or Homebrew cask:
```bash
ls -d "/Applications/MAUI Sherpa.app" "$HOME/Applications/MAUI Sherpa.app" 2>/dev/null | head -1 || { brew list --cask maui-sherpa >/dev/null 2>&1 && echo "installed (brew cask)"; } || echo "NOT FOUND"
```
- PASS if an app bundle or brew cask is found
- INFO if not found — optional dev-machine tool for managing Android SDKs, Apple certs/profiles, emulators, keystores & device inspectors (see `docs/maui-sherpa.md`). Install via: `brew install --cask redth/tap/maui-sherpa` (macOS) or download from the GitHub releases page.

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

### TargetFrameworks — both platforms
- PASS if contains `net9.0-ios` or `net10.0-ios` (or newer)
- PASS if contains `net9.0-android` or `net10.0-android` (or newer)
- WARN for each platform that is missing

### SupportedOSPlatformVersion
- WARN if iOS version is missing (recommended: ≥ 14.2)
- WARN if Android version is missing (recommended: ≥ 21.0)
- WARN if iOS < 14.2 or Android < 21

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

### DevFlow Agent registration  *(optional)*
Read `MauiProgram.cs` and check for a `#if DEBUG` block that registers a DevFlow or automation agent.
- PASS if found
- INFO if not found — optional; recommended for AI-driven automation workflows (see `docs/appium-vs-maui-devflow.md`)

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
ℹ️  Appium               not installed  (optional)
ℹ️  MAUI Sherpa          not installed  (optional — see docs/maui-sherpa.md)

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

Adapt the table rows to only show items that actually apply — skip sections with no findings. Always close with the actions list.
