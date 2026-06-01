# UI Components Library — Recommended Controls

> **Status:** Placeholder / living document. Fill in the recommended control for each UI need as the app's design system settles. The goal: one canonical answer to "which control do I use for X?" so screens stay consistent and automation-friendly.

This catalogs the **recommended control per UI task**, organized by type. Every interactive control listed here must carry an `AutomationId` (see [CLAUDE.md → AutomationId rule](../CLAUDE.md)) and, where applicable, be bound to a ViewModel property/command per the MVVM convention.

Columns:
- **Use case** — what the developer is trying to build.
- **Recommended control** — the control to reach for first.
- **Source** — `MAUI` (built-in), `CommunityToolkit.Maui`, or a named NuGet package.
- **Notes** — binding, accessibility, AutomationId, or gotcha guidance.

---

## Text & Input

| Use case | Recommended control | Source | Notes |
|---|---|---|---|
| Single-line text input | `Entry` | MAUI | _TBD_ |
| Multi-line text input | `Editor` | MAUI | _TBD_ |
| Search box | `SearchBar` | MAUI | _TBD_ |
| Masked / numeric input | _TBD_ | _TBD_ | _TBD_ |

## Buttons & Actions

| Use case | Recommended control | Source | Notes |
|---|---|---|---|
| Primary action | `Button` | MAUI | _TBD_ |
| Icon-only / tappable image | _TBD_ | _TBD_ | _TBD_ |
| Floating action button | _TBD_ | _TBD_ | _TBD_ |

## Selection & Toggles

| Use case | Recommended control | Source | Notes |
|---|---|---|---|
| On/off toggle | `Switch` | MAUI | _TBD_ |
| Single choice from list | `Picker` | MAUI | _TBD_ |
| Multiple choice | `CheckBox` | MAUI | _TBD_ |
| Continuous value | `Slider` | MAUI | _TBD_ |
| Radio-style single select | `RadioButton` | MAUI | _TBD_ |

## Lists & Collections

| Use case | Recommended control | Source | Notes |
|---|---|---|---|
| Scrollable item list | `CollectionView` | MAUI | Prefer over `ListView`. _TBD_ |
| Grouped / sectioned list | _TBD_ | _TBD_ | _TBD_ |
| Horizontal carousel | `CarouselView` | MAUI | _TBD_ |
| Pull-to-refresh | `RefreshView` | MAUI | _TBD_ |

## Navigation & Structure

| Use case | Recommended control | Source | Notes |
|---|---|---|---|
| App-level navigation | `Shell` | MAUI | _TBD_ |
| Tabbed sections | _TBD_ | _TBD_ | _TBD_ |
| Modal / bottom sheet | _TBD_ | _TBD_ | _TBD_ |

## Date & Time

| Use case | Recommended control | Source | Notes |
|---|---|---|---|
| Date selection | `DatePicker` | MAUI | _TBD_ |
| Time selection | `TimePicker` | MAUI | _TBD_ |

## Feedback & Status

| Use case | Recommended control | Source | Notes |
|---|---|---|---|
| Indeterminate loading | `ActivityIndicator` | MAUI | Wire to ViewModel `IsBusy`. _TBD_ |
| Determinate progress | `ProgressBar` | MAUI | _TBD_ |
| Transient message / toast | _TBD_ | `CommunityToolkit.Maui` | _TBD_ |
| Inline error / validation | _TBD_ | _TBD_ | _TBD_ |

## Media & Imagery

| Use case | Recommended control | Source | Notes |
|---|---|---|---|
| Static image | `Image` | MAUI | _TBD_ |
| Cached remote image | _TBD_ | _TBD_ | _TBD_ |

---

## How to extend this document

1. Add a row (or a new `##` section by control type) with the recommended control.
2. Fill in **Source** and **Notes** — especially binding pattern, `AutomationId` naming, and accessibility traits.
3. If the recommendation introduces a new NuGet package or tool, register it via the **`/nomad-add-tool`** skill so CLAUDE.md, this knowledge base, and `/nomad-doctor` stay in sync.
