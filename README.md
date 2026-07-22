# Xcode Skills

An unofficial export of Apple agent skills from **Xcode 27 developer beta 4**, repackaged as a cross-platform plugin marketplace for Claude Code, Codex, Grok CLI, and other compatible coding agents.

**These skills were not created by this repository.** They are Apple-authored agent skills bundled with the latest Xcode 27 dev beta, exported with:

```bash
xcrun agent skills export
```

This repo only packages and distributes those exported files for easier installation outside Xcode.

## Skills Included

| Skill | Description |
|-------|-------------|
| `swiftui-specialist` | SwiftUI best practices, data flow, modifiers, localization, and performance patterns |
| `swiftui-whats-new-27` | New SwiftUI APIs and SDK 27 migrations (`@State` macro, reorderable, swipe actions, toolbars, and more) |
| `uikit-app-modernization` | Modernize UIKit apps for multi-window environments (safe area, orientation, scene lifecycle) |
| `audit-xcode-security-settings` | Audit and enable security-oriented Xcode build settings and Enhanced Security features |
| `adopt-c-bounds-safety` | Guide for adopting the C `-fbounds-safety` language extension |
| `modernize-tests` | Migrate XCTest suites to Swift Testing and adopt modern testing features |
| `device-interaction` | Verify app behavior on device or simulator via screenshots, UI hierarchy, and touch |

The plugin package name remains `xcode-skills` across Claude Code, Codex, Grok CLI, and compatible agents. Xcode 27 beta 2 renamed `test-modernizer` to `modernize-tests`; beta 4 renamed `c-bounds-safety` to `adopt-c-bounds-safety`.

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

- **Skill content:** Apple Inc., exported from Xcode 27 developer beta 4 via `xcrun agent skills export`
- **This repository:** packaging and marketplace manifests only
