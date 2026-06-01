---
name: nomad-add-tool
description: Register a new tool into the NOMAD.Maui knowledge base — add it to the CLAUDE.md toolchain table, optionally scaffold a docs/ writeup and index it, and add a matching /nomad-doctor check. Use when the user wants to add, document, or onboard a new CLI / SDK / framework / test runner / automation tool for this project.
---

Register a new tool into this project's knowledge base in **one consistent pass**, so a fresh clone discovers it through CLAUDE.md, the `docs/` knowledge base, and `/nomad-doctor` without prior knowledge.

This skill operationalizes the **"Doctor Skill Integration"** rule in `CLAUDE.md`: every time a tool is added, the toolchain tables, the docs index, and the doctor checks must all stay in sync.

**Argument:** `$ARGUMENTS` — optional tool name (and any details the user already gave, e.g. `appium`, `firebase-cli`, `fastlane`). If omitted, gather the details in Step 1.

---

## Step 1 — Gather the tool's facts

Collect the following. Ask the user for anything not already provided in `$ARGUMENTS` or obvious from context. **Ask only for what's missing — don't re-ask what you already know.**

| Field | What it is | Example |
|---|---|---|
| **Name** | Display name as it appears to a developer | `Fastlane` |
| **Purpose** | One line: what it does for this project | `Automates iOS/Android build signing and store uploads` |
| **Requirement** | `required` or `optional` | `optional` |
| **Minimum version** | Lowest supported version, or `any` / `latest` | `2.x` |
| **Detection command** | A shell command that proves it's installed and prints a version | `fastlane --version` |
| **Install command** | How to install it | `gem install fastlane` |
| **Platform relevance** | iOS, Android, both, or build-host only — and whether a `.csproj` target makes it required | `both` |
| **Wraps a native SDK?** | If it wraps/needs a native SDK or runtime, name it (drives an extra doctor check) | `no` |
| **Needs a writeup?** | Does this deserve a `docs/` knowledge-base page (setup, commands, best practices, benchmarks)? Default: yes for frameworks/automation tools, no for trivial CLIs | `yes` |

Before writing anything, **read the current state** so you append in the right place and match the house style:
- `CLAUDE.md` — the "Required Toolchain" and "Optional tools" tables under *Required Toolchain*.
- `docs/README.md` — the knowledge-base index and its section headings.
- `plugins/nomad-maui-skills/skills/nomad-maui-doctor/SKILL.md` — Section 1 (env/tools), and the report layout in Section 4.

---

## Step 2 — Update the CLAUDE.md toolchain table

Open `CLAUDE.md` and add a row:

- If **required** → add to the **Required Toolchain** table with columns `| Tool | Minimum | Purpose |`.
- If **optional** → add to the **Optional tools** table with columns `| Tool | Purpose |`.

Keep rows aligned with the existing formatting. Use backticks for CLI names exactly as the existing rows do (e.g. `maui-ios`, `adb`). If the tool relates to a doc page you'll create in Step 3, you may reference it in the Purpose cell (see how the existing DevFlow row points at `docs/`).

---

## Step 3 — Scaffold a knowledge-base writeup (if warranted)

Only if **Needs a writeup? = yes**.

1. Decide placement:
   - Practical "how to use / best practices" guide → `docs/<tool-slug>.md`
   - Comparative benchmark or experiment → `docs/research/<tool-slug>.md`
   - Use a lower-case kebab-case slug derived from the name (e.g. `Fastlane` → `fastlane.md`).

2. Create the file following the house style seen in `docs/maui-devflow.md` — start with a title, a **Source:** line linking official docs, a **Status:** line if experimental, then sections: *What is it*, *Setup* (NuGet/install + `MauiProgram.cs` registration if applicable), *Commands / usage*, *Best practices*. Pull real content from the tool's official docs (use `WebFetch`; if it 403s, fall back to Chrome MCP per the global instructions). Do not invent benchmark numbers — only include measurements you or the user actually have.

3. Add an entry to `docs/README.md` under the most fitting section heading (create a new `##` section if none fits), matching the existing two-line format: a linked title followed by an indented one-line description.

---

## Step 4 — Add a /nomad-doctor check

Edit `plugins/nomad-maui-skills/skills/nomad-maui-doctor/SKILL.md`.

1. **Add a detection check** in **Section 1 — Environment & Tools** (or Section 2/3 if it's a `.csproj` config or architecture convention rather than a system tool). Match the existing check format exactly:

   ```
   ### <Tool Name>  *(optional)*   ← include "*(optional)*" only if optional
   ```bash
   <detection command> 2>/dev/null || echo "NOT FOUND"
   ```
   - PASS if <version condition>
   - WARN if <degraded-but-works condition>
   - FAIL / INFO if not found  ← FAIL if required; INFO if optional (mention install command)
   ```

   Apply the same PASS/WARN/FAIL/INFO conventions used by sibling checks:
   - **Required** tool missing → **FAIL** (or FAIL only when a matching `.csproj` target is present, like the Android SDK check does).
   - **Optional** tool missing → **INFO**, and name the install command.
   - Outdated-but-functional version → **WARN**.

2. If the tool **wraps a native SDK/runtime**, add a second check verifying that SDK/runtime (mirror how the Android SDK check pairs `adb` with `ANDROID_HOME`/`ANDROID_SDK_ROOT`).

3. **Add a report row** in **Section 4** under the matching report block (ENVIRONMENT & TOOLS, PROJECT CONFIGURATION, or ARCHITECTURE & BEST PRACTICES) so the printed report includes it, using the same emoji/column style.

---

## Step 5 — Summarize

Report back a concise changelog of exactly what changed and where:
- `CLAUDE.md` — row added to the *Required*/*Optional* table.
- `docs/<path>` — new writeup (or "skipped — no writeup warranted").
- `docs/README.md` — index entry added (if a writeup was made).
- `nomad-doctor/SKILL.md` — check + report row added.

Then **stop and let the user review** — do not commit. Per the project's git policy, committing requires explicit per-commit approval.
