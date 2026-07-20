# .NET MAUI Knowledge Base

Best practices, tools, and research for .NET MAUI development.

## Exploration Radar

- [Exploration Radar](radar.md)  
  Candidates noted for future exploration (libraries, tools, themes) — explicitly **not recommendations**. Items graduate to a proper guide once evaluated, or get parked with a reason.

## Conventions

- [.NET MAUI Conventions & Best Practices](maui-conventions.md)  
  The conventions the NOMAD sample app follows — MVVM, DI, mandatory AutomationId, DevFlow agent registration, screen boilerplate, and app-identity rules. Enforced by `/nomad-maui-doctor`.

## UI Components

- [UI Components Library — Recommended Controls](ui-components.md)  
  Placeholder/living catalog of the recommended MAUI control per UI need, organized by type (input, selection, lists, navigation, feedback, etc.). One canonical answer to "which control do I use for X?"

## Environment & Tooling

- [MAUI Sherpa — How to Use & Best Practices](maui-sherpa.md)  
  Cross-platform desktop app that manages Android SDKs, Apple certificates/profiles, emulators/simulators, keystores, device inspectors, and a DevFlow app inspector. Optional dev-machine convenience tool; complements `/nomad-maui-doctor`.

## AI Agent Skills

- [MAUI Skills — Reference Catalog](maui-skills.md)  
  Reference for `davidortinau/maui-skills`: what the ~37 on-demand .NET MAUI / Xamarin-migration skills are, the full skill catalog, how they map to NOMAD.Maui conventions, and how they relate to the official `dotnet/skills` repo.

- [MAUI Skills — How to Install & Use](maui-skills-usage.md)  
  Install/usage guide for the skills plugin in Claude Code and GitHub Copilot CLI — marketplace commands, the on-demand invocation model, prompting tips, updating/removing, and when to use them vs. project docs.

- [Syncfusion MAUI Skills — How to Install & Use](syncfusion-maui-skills.md)  
  Per-control AI-agent skills for Syncfusion MAUI components via the `npx skills` CLI. Relevant only when the project references `Syncfusion.Maui.*` packages — `/nomad-maui-doctor` auto-detects this and expects the skills installed.

## UI Automation & Testing

- [MAUI DevFlow — How to Use & Best Practices](maui-devflow.md)  
  Setup, CLI commands, MCP server config, CSS selector reference, AutomationId guidance, and best practices drawn from official docs and hands-on benchmarking.

- [Appium vs MAUI DevFlow — Comparative Benchmark](research/appium-vs-maui-devflow.md)
- [MAUI + DevFlow vs Flutter + Marionette — the AI agent feedback loop](research/maui-vs-flutter-agent-loop.md) — where the loop actually costs time; hot-reload gap  
  Scripted timing benchmark + AI agent sessions comparing Appium (XCUITest) against Microsoft's experimental DevFlow stack. DevFlow is 4.6× faster on action latency, 7.2× faster end-to-end for an AI agent task.
