# NOMAD.Maui

The .NET MAUI implementation of the [NOMAD Framework](https://github.com/raver99/NOMAD-Framework) — opinionated best practices, tooling, and AI-assisted workflows for building cross-platform mobile apps with .NET MAUI (iOS & Android).

> NOMAD (Nextgen Optimized Mobile Application Development) is a platform-agnostic framework. NOMAD.Maui brings its principles to the .NET MAUI stack.

---

## Claude Code Plugin

This repo ships a Claude Code plugin with skills for AI-assisted .NET MAUI development. Install it in any project:

```
/plugin marketplace add raver99/NOMAD.Maui
```

### `/nomad-maui-doctor`

Health check for your .NET MAUI environment and project setup. Verifies:

- .NET SDK and MAUI workloads (`maui-ios`, `maui-android`)
- Xcode, iOS simulators, Android SDK, Java
- Optional tools: DevFlow CLI, Appium, MAUI Sherpa
- Project config: `ApplicationId`, target frameworks, OS version minimums
- Architecture conventions: MVVM structure, DI setup, `AutomationId` coverage

Run it after cloning or setting up a new machine.

---

## Knowledge Base

Guides, best practices, and research for .NET MAUI development — see [`docs/`](docs/).

---

## Project Conventions

Architecture rules, required toolchain, and boilerplate requirements — see [`CLAUDE.md`](CLAUDE.md).
