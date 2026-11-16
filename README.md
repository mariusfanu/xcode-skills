# Xcode Skills

Give your coding agent Apple’s guidance for building and testing apps with Xcode. This unofficial package contains **15 Apple-authored skills** covering SwiftUI, App Intents, Foundation Models, testing, and more.

**Plugin version `1.10.0`** · Exported from **Xcode 27.3 beta 2 (`27C5035i`)**

## Installation

Choose your coding agent. Each method installs the same skills.

[Ask your agent to install](#ask-your-agent-to-install) · [Claude Code](#claude-code) · [Codex](#codex) · [GitHub Copilot](#github-copilot) · [Grok CLI](#grok-cli) · [Google Antigravity](#google-antigravity-app) · [Manual install](#manual-install)

### Ask your agent to install

Copy this prompt into your coding agent.

```text
Install or update Xcode Skills using the section for my coding agent at
https://github.com/mariusfanu/xcode-skills#installation. If my agent or install
scope is unclear, ask me before making changes.

After installing, check that the agent can discover the skills listed in the README.
Tell me what you verified. If you can't access the intended machine or a step must
be done in the app, give me the exact step that remains.
```

### Claude Code

Run these commands inside Claude Code.

```text
/plugin marketplace add mariusfanu/xcode-skills
/plugin install xcode-skills@xcode-skills
```

From a terminal, use `claude plugin marketplace add mariusfanu/xcode-skills` and `claude plugin install xcode-skills@xcode-skills`. For a local clone, pass its path to `marketplace add` instead of the GitHub repository name.

To update an existing installation, run `/plugin marketplace update xcode-skills`, then `/plugin update xcode-skills@xcode-skills` inside Claude Code.

### Codex

Run these commands in a terminal. To use the desktop app, register the marketplace with the first command, then open **Plugins** and install `xcode-skills` from the `xcode-skills` marketplace.

```bash
codex plugin marketplace add mariusfanu/xcode-skills
codex plugin add xcode-skills@xcode-skills
```

To update an existing installation, run `codex plugin marketplace upgrade xcode-skills` followed by `codex plugin add xcode-skills@xcode-skills`. Check the installed version with `codex plugin list --marketplace xcode-skills`.

To install from a specific branch or tag, add `--ref <branch-or-tag>` to the marketplace command.

### GitHub Copilot

Install the plugin with GitHub Copilot CLI. [Copilot CLI documentation](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/plugins-finding-installing)

```bash
copilot plugin marketplace add mariusfanu/xcode-skills
copilot plugin install xcode-skills@xcode-skills
copilot skill list
```

The Copilot desktop app also picks up skills configured for Copilot CLI. Open **Customize → Skills** to see them. [Copilot desktop documentation](https://docs.github.com/en/copilot/how-tos/github-copilot-app/customize-github-copilot-app#adding-agent-skills)

For VS Code or a project you want to share with teammates, use the [manual install](#manual-install). Open a new Copilot chat and type `/foundation-models-specialist` to invoke the skill. In an open Copilot CLI session, run `/skills reload` after copying the files. [VS Code documentation](https://code.visualstudio.com/docs/agent-customization/agent-skills)

### Grok CLI

Add the marketplace and install the plugin.

```bash
grok plugin marketplace add mariusfanu/xcode-skills
grok plugin install xcode-skills --trust
```

For a direct install, run `grok plugin install 'mariusfanu/xcode-skills#plugins/xcode-skills' --trust`. Use `grok plugin details xcode-skills` to check the install or `/skills` inside Grok to see the available skills.

### Google Antigravity App

Use the [manual install](#manual-install) to add the skills to one workspace or make them available across all workspaces. Start a new conversation and type `/foundation-models-specialist` to invoke the skill. [Antigravity documentation](https://antigravity.google/docs/skills?app=antigravity#antigravity-app-skill-locations)

### Manual install

Clone this repository and keep the following commands in the same terminal so `skills_source` points to the clone.

```bash
git clone https://github.com/mariusfanu/xcode-skills.git
skills_source="$PWD/xcode-skills/plugins/xcode-skills/skills"
```

If you already have a clone, set `skills_source` to the full path to its `plugins/xcode-skills/skills` folder.

Choose a destination. For a project install, switch to that project's root before copying the skills.

| Coding agent | Scope | Destination |
| --- | --- | --- |
| GitHub Copilot | All local projects | `$HOME/.copilot/skills` |
| GitHub Copilot | One project | `.github/skills` |
| Google Antigravity App | All workspaces | `$HOME/.gemini/config/skills` |
| Google Antigravity App | One workspace | `.agents/skills` |
| Claude Code | One project | `.claude/skills` |
| Grok CLI | One project | `.grok/skills` |

Set `skills_destination` to your chosen path. This example installs for all local Copilot projects.

```bash
skills_destination="$HOME/.copilot/skills"
mkdir -p "$skills_destination"
cp -Rf "$skills_source"/* "$skills_destination/"
```

To update a manual install, pull the latest changes in your clone and run the copy step again. It replaces existing files with the same names.

## Use the skills

Start a new chat in your coding agent and ask it to use the skill that matches your task. For example, “Use `swiftui-specialist` to review this view for unnecessary updates.”

### Skills included

| Skill | What it helps with |
| --- | --- |
| [`swiftui-specialist`](plugins/xcode-skills/skills/swiftui-specialist/SKILL.md) | SwiftUI best practices, data flow, modifiers, localization, and performance |
| [`swiftui-whats-new-27`](plugins/xcode-skills/skills/swiftui-whats-new-27/SKILL.md) | SwiftUI APIs and SDK 27 migrations, including `@State`, reordering, swipe actions, and toolbars |
| [`building-document-based-swiftui-applications`](plugins/xcode-skills/skills/building-document-based-swiftui-applications/SKILL.md) | Build or migrate document apps with `Document`, `DocumentGroup`, readers, writers, and UTTypes |
| [`app-intents-specialist`](plugins/xcode-skills/skills/app-intents-specialist/SKILL.md) | App Intents best practices for entities, queries, parameters, Shortcuts, and localization |
| [`app-intents-whats-new-27`](plugins/xcode-skills/skills/app-intents-whats-new-27/SKILL.md) | App Intents APIs for iOS 26 and 27, including execution modes, snippets, Visual Intelligence, and Spotlight |
| [`siri-app-integration`](plugins/xcode-skills/skills/siri-app-integration/SKILL.md) | Expose tools and generable entities to Siri, write App Skills, and use the Siri Metadata Analyzer |
| [`siri-evaluations`](plugins/xcode-skills/skills/siri-evaluations/SKILL.md) | Write, run, and analyze tests that check how Siri uses an app’s App Intents |
| [`foundation-models-specialist`](plugins/xcode-skills/skills/foundation-models-specialist/SKILL.md) | Use Foundation Models with `LanguageModelSession`, `@Generable`, streaming, and tool calling |
| [`app-resizability`](plugins/xcode-skills/skills/app-resizability/SKILL.md) | Adapt UIKit apps to resizable windows, split screen, Stage Manager, and iPhone Duo |
| [`macos-touch-gestures`](plugins/xcode-skills/skills/macos-touch-gestures/SKILL.md) | Add direct touch support to macOS apps, including through Sidecar |
| [`audit-xcode-security-settings`](plugins/xcode-skills/skills/audit-xcode-security-settings/SKILL.md) | Audit compiler and analyzer settings, Enhanced Security, and checked pointer arithmetic |
| [`adopt-c-bounds-safety`](plugins/xcode-skills/skills/adopt-c-bounds-safety/SKILL.md) | Adopt the C `-fbounds-safety` language extension |
| [`modernize-tests`](plugins/xcode-skills/skills/modernize-tests/SKILL.md) | Migrate XCTest and XCUIAutomation suites to Swift Testing |
| [`device-interaction`](plugins/xcode-skills/skills/device-interaction/SKILL.md) | Inspect and interact with devices and simulators, including tvOS focus and foldable hinge angle |
| [`xctrace`](plugins/xcode-skills/skills/xctrace/SKILL.md) | Capture Instruments traces, read `.trace` and `.atrc` files, and triage hangs |

## What’s new in Xcode 27.3 beta 2

- Adds `foundation-models-specialist`.
- Expands Siri evaluation guidance on UI test targets and expectations and documents `parameterSummary` for Siri tools with parameters.
- Adds Markdown output guidance for `xctrace` and shortens several skill descriptions.

<details>
<summary>Earlier Xcode releases</summary>

- **27.3 beta 1** adds `macos-touch-gestures`, `siri-app-integration`, and `siri-evaluations`. It adds the `hinge` command to `device-interaction`, expands `modernize-tests` to cover XCUIAutomation migrations, and shortens the App Intents skill descriptions.
- **27.2 beta 3** adds `xctrace` for Instruments and hang triage and records that lazy `@State` initialization requires a private getter. The RC and final release (`27B52`) keep the same guidance.
- **27.1 beta 1** replaces `uikit-app-modernization` with `app-resizability` and covers resizable windows, iPhone Duo, and `userInterfaceIdiom`.
- **27 final** adds checked pointer arithmetic (`arm64e.x1`) to `audit-xcode-security-settings`.
- **27 beta 6** adds tvOS Siri Remote focus commands to `device-interaction`.
- **27 beta 5** adds App Intents skills and separates the guidance for document apps into `building-document-based-swiftui-applications`.
- **27 beta 4** renames `c-bounds-safety` to `adopt-c-bounds-safety`.
- **27 beta 2** renames `test-modernizer` to `modernize-tests`.

</details>

## Attribution

Apple Inc. authored the skills and bundles them with Xcode. This repository packages the files produced by `xcrun agent skills export` and provides the marketplace manifests.
