# Xcode Skills

An unofficial export of Apple agent skills from **Xcode 27.3 beta 2 (27C5035i)**, repackaged for Claude Code, Codex, Grok CLI, Google Antigravity, and other compatible coding agents.

**These skills were not created by this repository.** They are Apple-authored agent skills bundled with Xcode 27.3 beta 2 (27C5035i), exported with:

```bash
xcrun agent skills export
```

This repo only packages and distributes those exported files for easier installation outside Xcode.

## Skills Included

| Skill | Description |
|-------|-------------|
| `swiftui-specialist` | SwiftUI best practices, data flow, modifiers, localization, and performance patterns |
| `swiftui-whats-new-27` | New SwiftUI APIs and SDK 27 migrations (`@State` macro, reorderable, swipe actions, toolbars, and more) |
| `building-document-based-swiftui-applications` | Build and migrate document-based SwiftUI apps with the `Document` protocol (`DocumentGroup`, readers/writers, UTTypes) |
| `app-intents-specialist` | App Intents best practices: entities, queries, parameters, donations, shortcuts phrases, and localization |
| `app-intents-whats-new-27` | New App Intents APIs for iOS 26/27 (execution modes, snippets, Visual Intelligence, Spotlight, testing) |
| `app-resizability` | Make UIKit apps adapt to resizable windows, split-screen, Stage Manager, and the foldable iPhone Duo |
| `audit-xcode-security-settings` | Audit and enable security-oriented Xcode build settings, Enhanced Security, and checked pointer arithmetic |
| `adopt-c-bounds-safety` | Guide for adopting the C `-fbounds-safety` language extension |
| `modernize-tests` | Migrate XCTest and XCUIAutomation suites to Swift Testing and adopt modern testing patterns |
| `device-interaction` | Verify app behavior on device or simulator via screenshots, UI hierarchy, touch, tvOS Siri Remote focus, and foldable hinge angle |
| `xctrace` | Capture and read Instruments traces (`.trace`, `.atrc`) and triage app hangs with `xctrace triage` |
| `macos-touch-gestures` | Add direct touch support (e.g. iPad via Sidecar) to macOS apps with gesture recognizers across AppKit, SwiftUI, and Mac Catalyst |
| `siri-app-integration` | Make an app work with Siri's model: `@AppIntent(.tool)`, `@AppEntity(.generable)`, Skills bundles, and the Siri Metadata Analyzer |
| `siri-evaluations` | Write, run, and analyze SiriEvaluations scenarios that grade how Siri uses an app's App Intents |
| `foundation-models-specialist` | Use Foundation Models in Swift with `LanguageModelSession`, streaming, `@Generable` structured output, tool calling, and on-device, Private Cloud Compute, or custom models |

The plugin package name remains `xcode-skills` across Claude Code, Codex, Grok CLI, and compatible agents. Xcode 27 beta 2 renamed `test-modernizer` to `modernize-tests`; beta 4 renamed `c-bounds-safety` to `adopt-c-bounds-safety`; beta 5 adds App Intents skills and splits document-based SwiftUI guidance into `building-document-based-swiftui-applications`; beta 6 adds tvOS Siri Remote focus commands to `device-interaction`; Xcode 27 adds checked pointer arithmetic (`arm64e.x1`) to `audit-xcode-security-settings`; Xcode 27.1 beta 1 replaces `uikit-app-modernization` with `app-resizability` (resizable windows, iPhone Duo, `userInterfaceIdiom`); Xcode 27.2 beta 3 adds `xctrace` for Instruments traces and hang triage, and records that lazy `@State` initialization requires a private getter; Xcode 27.2 RC and final (27B52) skill guidance is unchanged from beta 3; Xcode 27.3 beta 1 adds `macos-touch-gestures`, `siri-app-integration`, and `siri-evaluations`, adds a foldable `hinge` command to `device-interaction`, rewrites `modernize-tests` to cover migrating XCUIAutomation UI tests to Swift Testing, and shortens the App Intents skill descriptions.

Xcode 27.3 beta 2 adds `foundation-models-specialist`, expands `siri-evaluations` guidance on UI test targets and expectations, documents `parameterSummary` for Siri tools with parameters, and adds Markdown output guidance for `xctrace`. It also shortens several skill descriptions.

## Installation

### Claude Code

Register the marketplace and install:

```bash
/plugin marketplace add mariusfanu/xcode-skills
/plugin install xcode-skills@xcode-skills
```

Or install from a local clone:

```bash
git clone https://github.com/mariusfanu/xcode-skills.git
/plugin marketplace add ./xcode-skills
/plugin install xcode-skills@xcode-skills
```

To pick up new skill versions later:

```bash
/plugin marketplace update xcode-skills
/plugin update xcode-skills@xcode-skills
```

The same commands also work from the terminal via `claude plugin ...` (e.g. `claude plugin marketplace add mariusfanu/xcode-skills`).

### Codex CLI / Codex App

**CLI** — add the GitHub marketplace and install:

```bash
codex plugin marketplace add mariusfanu/xcode-skills
codex plugin add xcode-skills@xcode-skills
```

**App** — open Plugins in the sidebar, add `mariusfanu/xcode-skills` as a marketplace source, then install `xcode-skills`.

For direct CLI install from a specific branch or tag:

```bash
codex plugin marketplace add mariusfanu/xcode-skills --ref main
codex plugin add xcode-skills@xcode-skills
```

### Grok CLI

Add this repo as a marketplace source, then install the plugin:

```bash
grok plugin marketplace add mariusfanu/xcode-skills
grok plugin install mariusfanu/xcode-skills --trust
```

Or install directly without adding the marketplace first:

```bash
grok plugin install mariusfanu/xcode-skills --trust
```

Verify:

```bash
grok plugin list
grok plugin details xcode-skills
```

In the TUI, `/marketplace` lists available plugins; `/skills` shows loaded skills.

### Google Antigravity App

To install the skills for all workspaces, clone this repository and copy the skill folders into Antigravity's global skills directory.

```bash
git clone https://github.com/mariusfanu/xcode-skills.git
cd xcode-skills
mkdir -p "$HOME/.gemini/config/skills"
cp -Rf plugins/xcode-skills/skills/* "$HOME/.gemini/config/skills/"
```

To install for a single workspace, run these commands from that workspace's root. Replace `/path/to/xcode-skills` with the path to your clone.

```bash
mkdir -p .agents/skills
cp -Rf "/path/to/xcode-skills"/plugins/xcode-skills/skills/* .agents/skills/
```

Start a new conversation in Antigravity and type `/foundation-models-specialist` to invoke the skill. See [Google's Antigravity App skill documentation](https://antigravity.google/docs/skills?app=antigravity#antigravity-app-skill-locations) for details.

### Manual / Project-Scoped Install

Copy or symlink the `skills/` directory into your project:

```bash
# Claude Code
mkdir -p .claude/skills
cp -R plugins/xcode-skills/skills/* .claude/skills/

# Grok CLI
mkdir -p .grok/skills
cp -R plugins/xcode-skills/skills/* .grok/skills/
```

## Attribution

- **Skill content:** Apple Inc., exported from Xcode 27.3 beta 2 (27C5035i) via `xcrun agent skills export`
- **This repository:** packaging and marketplace manifests only
