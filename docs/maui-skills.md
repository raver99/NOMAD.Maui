# MAUI Skills — Reference Catalog

**Source:** [GitHub (davidortinau/maui-skills)](https://github.com/davidortinau/maui-skills)
**Status:** Community resource maintained by David Ortinau (Principal PM, .NET MAUI @ Microsoft). Not an official Microsoft repository — see note below. Loaded as a plugin into Claude Code or GitHub Copilot CLI.

> **How to install and use these →** [`maui-skills-usage.md`](maui-skills-usage.md)

---

## What it is

`maui-skills` is a collection of **~37 on-demand "skills"** for AI coding agents working on .NET MAUI apps and Xamarin → .NET migrations. Each skill is a `SKILL.md` file (with optional helper scripts) that the agent loads *automatically* when your prompt matches the skill's topic — injecting focused, expert-level guidance, code examples, and platform-specific notes into the model's context.

In other words: instead of the agent guessing at, say, the correct `CollectionView` grouping pattern or APNS push setup, the matching skill supplies the canonical approach on demand.

These skills are a **reference / knowledge-augmentation layer for AI agents** — they do not ship in the app, add NuGet dependencies, or change the build. They complement this repo's own conventions in [`CLAUDE.md`](../CLAUDE.md) and the [DevFlow automation](maui-devflow.md) tooling.

### Relationship to the official `dotnet/skills` repo

Microsoft maintains an official AI-agent skills repo at [`dotnet/skills`](https://github.com/dotnet/skills) that includes a `dotnet-maui` plugin. As of this writing that official MAUI plugin is **narrower** (environment setup, diagnostics, troubleshooting) and is **not** a migration of David Ortinau's collection — the two coexist. `davidortinau/maui-skills` remains the broader, dev-focused set and shows no archive/move notice.

---

## Skill catalog (37)

### .NET MAUI development

| Skill | Covers |
|---|---|
| `maui-accessibility` | Screen readers, `SemanticProperties`, focus management, TalkBack/VoiceOver |
| `maui-animations` | View animations, easing, rotation, scale, translation, fade |
| `maui-app-icons-splash` | Icon config, splash screens, SVG conversion, adaptive icons |
| `maui-app-lifecycle` | App states, Window lifecycle, backgrounding, state preservation |
| `maui-aspire` | .NET Aspire backend integration, service discovery, MSAL.NET auth |
| `maui-authentication` | OAuth 2.0, MSAL.NET, Entra ID, token caching |
| `maui-collectionview` | Data display, layouts, selection, grouping, pull-to-refresh |
| `maui-current-apis` | Deprecated-API detection, framework version reasoning |
| `maui-custom-handlers` | Custom handlers, property mappers, native view implementation |
| `maui-data-binding` | XAML bindings, compiled bindings, value converters, MVVM patterns |
| `maui-deep-linking` | Android App Links, iOS Universal Links, domain verification |
| `maui-dependency-injection` | Service registration, lifetimes, constructor injection |
| `maui-file-handling` | File picker, assets, bundled content, app-data storage |
| `maui-geolocation` | GPS, location sensors, permissions, mock-location detection |
| `maui-gestures` | Tap, swipe, pan, pinch, drag-and-drop recognizers |
| `maui-graphics-drawing` | `GraphicsView`, canvas operations, shapes, text rendering |
| `maui-hot-reload-diagnostics` | Troubleshoot C#/XAML/Blazor hot reload issues |
| `maui-hybridwebview` | Web embedding, JavaScript–C# interop, trimming |
| `maui-local-notifications` | Scheduled reminders, notification channels, foreground handling |
| `maui-localization` | Multi-language `.resx`, RTL layout, culture switching |
| `maui-maps` | Map controls, pins, polygons, geocoding, Google Maps setup |
| `maui-media-picker` | Photo/video selection, camera capture, multi-select |
| `maui-performance` | Profiling, compiled bindings, layout efficiency, trimming |
| `maui-permissions` | Runtime permissions, `PermissionStatus`, custom permissions |
| `maui-platform-invoke` | Native API calls, partial classes, multi-targeting |
| `maui-push-notifications` | Firebase, APNS, Azure Notification Hubs, backend integration |
| `maui-rest-api` | `HttpClient` setup, CRUD, `System.Text.Json` |
| `maui-safe-area` | `SafeAreaEdges`, keyboard avoidance, edge-to-edge layout |
| `maui-secure-storage` | `SecureStorage`, Keychain / Credential Manager |
| `maui-shell-navigation` | Shell hierarchy, tabs, flyout menus, URI-based routing |
| `maui-speech-to-text` | Voice input, speech recognition, microphone permissions |
| `maui-sqlite-database` | SQLite via `sqlite-net-pcl`, CRUD, WAL mode |
| `maui-theming` | Light/dark mode, `AppThemeBinding`, dynamic resources |
| `maui-unit-testing` | xUnit, ViewModel testing, mocking MAUI services |

### Xamarin → .NET migration

| Skill | Covers |
|---|---|
| `xamarin-android-migration` | SDK-style conversion, `AndroidManifest` updates |
| `xamarin-forms-migration` | Namespace renames, layout changes, handler migration |
| `xamarin-ios-migration` | Xamarin.iOS/Mac/tvOS → .NET conversion, code signing |

---

## How this maps to NOMAD.Maui conventions

Several skills line up directly with mandatory rules in [`CLAUDE.md`](../CLAUDE.md) — useful when prompting an agent to scaffold a screen:

| NOMAD.Maui rule | Relevant skill(s) |
|---|---|
| MVVM with `ObservableObject` / `[RelayCommand]` | `maui-data-binding` |
| Dependency Injection in `MauiProgram.cs` | `maui-dependency-injection` |
| Keyboard avoidance / safe-area wrapping | `maui-safe-area` |
| App icon & splash must not be default purple | `maui-app-icons-splash` |
| ViewModel unit tests | `maui-unit-testing` |

> Note: these skills give general MAUI best practice. Where they differ from this repo's rules (e.g. `AutomationId` on **every** interactive control, DevFlow agent registration), **`CLAUDE.md` wins.**

---

## Repository structure

```
maui-skills/
├── plugins/maui-skills/skills/
│   ├── maui-<topic>/
│   │   ├── SKILL.md          # YAML frontmatter: name, description
│   │   └── scripts/          # optional helper scripts
│   └── xamarin-<topic>/
│       └── SKILL.md
├── .github/plugin/marketplace.json
└── LICENSE
```
