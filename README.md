# Xcode Skills

An unofficial export of Apple agent skills from **Xcode 27.2 beta 3**, repackaged as a cross-platform plugin marketplace for Claude Code, Codex, Grok CLI, and other compatible coding agents.

**These skills were not created by this repository.** They are Apple-authored agent skills bundled with Xcode 27.2 beta 3, exported with:

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
| `modernize-tests` | Migrate XCTest suites to Swift Testing and adopt modern testing features |
| `device-interaction` | Verify app behavior on device or simulator via screenshots, UI hierarchy, touch, and tvOS Siri Remote focus |
| `xctrace` | Capture and read Instruments traces (`.trace`, `.atrc`) and triage app hangs with `xctrace triage` |

The plugin package name remains `xcode-skills` across Claude Code, Codex, Grok CLI, and compatible agents. Xcode 27 beta 2 renamed `test-modernizer` to `modernize-tests`; beta 4 renamed `c-bounds-safety` to `adopt-c-bounds-safety`; beta 5 adds App Intents skills and splits document-based SwiftUI guidance into `building-document-based-swiftui-applications`; beta 6 adds tvOS Siri Remote focus commands to `device-interaction`; Xcode 27 adds checked pointer arithmetic (`arm64e.x1`) to `audit-xcode-security-settings`; Xcode 27.1 beta 1 replaces `uikit-app-modernization` with `app-resizability` (resizable windows, iPhone Duo, `userInterfaceIdiom`); Xcode 27.2 beta 3 adds `xctrace` for Instruments traces and hang triage, and records that lazy `@State` initialization requires a private getter.

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

- **Skill content:** Apple Inc., exported from Xcode 27.2 beta 3 via `xcrun agent skills export`
- **This repository:** packaging and marketplace manifests only
