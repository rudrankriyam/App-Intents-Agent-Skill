# App Intents Expert Skill

Building intents, entities, queries, Siri integration, Shortcuts, Spotlight, Apple Intelligence, and interactive snippets for iOS 26+. This skill distills App Intents information into actionable, concise references for agents, code generation, and code review workflows.

## Who this is for

- Developers integrating their apps with Siri, Shortcuts, Spotlight, and Apple Intelligence

- Teams adopting App Intents who want correct patterns and modern APIs from day one

- Anyone building interactive snippets, entities, queries, or App Shortcuts for iOS 26+

## See also my other skills:

- [App Store Connect CLI Skills](https://github.com/rudrankriyam/app-store-connect-cli-skills)

## How to Use This Skill

### Option A: Using skills.sh (recommended)

Install this skill with a single command:

```shell
npx skills add https://github.com/rudrankriyam/App-Intents-Agent-Skill --skill app-intents-expert-skill
```

Then use the skill in your AI agent, for example:

> Use the app intents expert skill and help me add Siri support to my app

### Option B: Claude Code Plugin

#### Personal Usage

To install this Skill for your personal use in Claude Code:

- Add the marketplace:

```shell
/plugin marketplace add rudrankriyam/App-Intents-Agent-Skill
```

- Install the Skill:

```shell
/plugin install app-intents-expert@app-intents-expert-skill
```

#### Project Configuration

To automatically provide this Skill to everyone working in a repository, configure the repository's `.claude/settings.json`:

```json
{
  "enabledPlugins": {
    "app-intents-expert@app-intents-expert-skill": true
  },
  "extraKnownMarketplaces": {
    "app-intents-expert-skill": {
      "source": {
        "source": "github",
        "repo": "rudrankriyam/App-Intents-Agent-Skill"
      }
    }
  }
}
```

When team members open the project, Claude Code will prompt them to install the Skill.

### Option C: Manual install

- Clone this repository.

- Install or symlink the `app-intents-expert-skill/` folder following your tool's official skills installation docs (see links below).

- Use your AI tool as usual and ask it to use the "app-intents-expert" skill for App Intents tasks.

#### Where to Save Skills

Follow your tool's official documentation, here are a few popular ones:

- Codex: [Where to save skills](https://developers.openai.com/codex/skills/#where-to-save-skills)

- Claude: [Using Skills](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#using-skills)

- Cursor: [Enabling Skills](https://cursor.com/docs/context/skills#enabling-skills)

How to verify:

Your agent should reference the workflow/checklists in `app-intents-expert-skill/SKILL.md` and jump into the relevant reference file for your task.

## What This Skill Offers

This skill gives your AI coding tool practical App Intents expertise. It can:

### Guide Your App Intents Architecture

- Choose the right protocol for your use case (`AppIntent`, `OpenIntent`, `SnippetIntent`, `UndoableIntent`)

- Design entities and queries that work across Siri, Spotlight, Shortcuts, and Apple Intelligence

- Structure intent-driven code that shares logic between your UI and system integrations

- Advise on iOS 26+ interactive snippets, Visual Intelligence, and the new `@AppIntent` macro

### Write Better App Intents Code

- Build correct `AppEntity` types with `@ComputedProperty` and `@DeferredProperty`

- Create `AppShortcutsProvider` with proper phrases, parameters, and localization

- Implement `EntityQuery` subtypes (`EntityStringQuery`, `EntityPropertyQuery`) for rich entity resolution

- Use `TargetContentProvidingIntent` and `onAppIntentExecution` for clean SwiftUI navigation

### Integrate with the Full System

- Make intents discoverable through Siri, Spotlight, Shortcuts, Action Button, and Apple Pencil

- Build interactive snippets with buttons, toggles, and confirmation flows

- Support Visual Intelligence with image search and `SemanticContentDescriptor`

- Index entities with `IndexedEntity` and donate to Spotlight with `@Property(indexingKey:)`

- Integrate with Apple Intelligence through assistant schemas and the `@AppIntent` macro

## Contributing

Contributions are welcome! This repository follows the [Agent Skills open format](https://agentskills.io/home), which has specific structural requirements.

Please read [CONTRIBUTING.md](/CONTRIBUTING.md) for:

- How to contribute improvements to `SKILL.md` and the reference files

- Format requirements and quality standards

- Pull request process

## Resources

- [Get to know App Intents — WWDC 2025](https://developer.apple.com/videos/play/wwdc2025/244/)
- [Explore new advances in App Intents — WWDC 2025](https://developer.apple.com/videos/play/wwdc2025/275/)
- [Design interactive snippets — WWDC 2025](https://developer.apple.com/videos/play/wwdc2025/281/)
- [Develop for Shortcuts and Spotlight with App Intents — WWDC 2025](https://developer.apple.com/videos/play/wwdc2025/260/)
- [App Intents Documentation](https://developer.apple.com/documentation/AppIntents)

## License

This skill is open-source and available under the MIT License. See [LICENSE](/LICENSE) for details.
