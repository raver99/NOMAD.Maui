# MAUI Sherpa — How to Use & Best Practices

**Source:** [Official site](https://redth.github.io/MAUI.Sherpa/index.html) · [GitHub (Redth/MAUI.Sherpa)](https://github.com/Redth/MAUI.Sherpa)
**Status:** Pre-1.0 (latest v0.10.0, released 2026-05-26) — features and UI change between releases. MIT-licensed, open source.

---

## What is MAUI Sherpa

MAUI Sherpa is a **cross-platform desktop application** (macOS, Windows, Linux) that consolidates the chores of running a .NET MAUI dev environment into one GUI — *"Your guide to .NET MAUI development."* It is **not** a CLI, NuGet package, or in-app agent: it's a developer-machine tool you launch alongside your IDE.

It complements, rather than replaces, the project's own [`/nomad-maui-doctor`](../CLAUDE.md) skill and [MAUI DevFlow](maui-devflow.md) automation — Sherpa even bundles its own visual "MAUI Doctor" and a DevFlow app inspector.

### What it manages

| Feature | Role |
|---|---|
| **MAUI Doctor** | Environment diagnostics with AI-powered fix suggestions |
| **Android SDK Manager** | Visual browsing/installation of SDK packages |
| **Emulators & Simulators** | Create and launch Android emulators / iOS simulators |
| **Android Keystores** | Create and manage signing keystores |
| **Apple Certificates & Profiles** | Manage iOS signing certificates and provisioning profiles |
| **Device Inspectors** | Logcat viewers, file browsers, on-device terminals |
| **DevFlow App Inspector** | Remote MAUI app inspection with visual-tree browsing |
| **GitHub Copilot Integration** | Built-in AI chat assistance |

---

## Setup

MAUI Sherpa is a GUI desktop app — there is nothing to add to the project (no NuGet package, no `MauiProgram.cs` registration).

### Install (macOS — recommended)

```
brew install --cask redth/tap/maui-sherpa
```

### Manual download

Grab the latest build from the [Releases](https://github.com/Redth/MAUI.Sherpa/releases) page:

- **macOS** — `MAUI-Sherpa.macos.zip` → extract and move the app into `/Applications`.
- **Windows** — `MAUI-Sherpa.windows-x64.zip` or `MAUI-Sherpa.windows-arm64.zip` → extract and run `MauiSherpa.exe`.
- **Linux** — AppImage, `.deb`, or Flatpak.

### Related: `sherpa-inspector`

A separate `sherpa-inspector` dotnet tool exists for embedding Sherpa's app-inspection capability into external tooling. Not required for normal Sherpa use; out of scope for this project today.

---

## Usage

Sherpa is GUI-driven; typical flows for this project:

- **Onboarding a new machine** — run Sherpa's *MAUI Doctor* to install missing Android SDK packages and Java, then cross-check against `/nomad-maui-doctor` for project-specific conventions.
- **iOS signing** — manage Apple certificates and provisioning profiles before a device build.
- **Android signing** — create/manage the release keystore.
- **Emulator/simulator management** — spin up devices for manual testing or UI automation runs.
- **Inspecting a running app** — use the DevFlow App Inspector to browse the live visual tree (complements the [DevFlow CLI/MCP automation](maui-devflow.md)).

---

## Best practices

- **Sherpa is a convenience, not a source of truth.** Project requirements live in `CLAUDE.md` and are enforced by `/nomad-maui-doctor`. Use Sherpa to *fix* environment gaps; use `/nomad-maui-doctor` to *verify* the project's specific conventions (target frameworks, AutomationIds, MVVM layout, app identity).
- **Pre-1.0 — expect change.** Don't script against its behavior; treat it as an interactive tool.
- **Keep signing assets out of git.** Keystores, certificates, and provisioning profiles managed via Sherpa must never be committed.
- **Two doctors, two jobs.** Sherpa's *MAUI Doctor* checks the general MAUI/SDK environment; the project's `/nomad-maui-doctor` checks *this repo's* setup and architecture. Run both.
