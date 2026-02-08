# Contributing to App Intents Expert Skill

Thank you for your interest in improving this skill! This repository follows the [Agent Skills open format](https://agentskills.io/specification), so contributions must adhere to that specification.

## How to Contribute

### Improving Existing Content

1. Fork the repository
2. Make changes to `SKILL.md` or files in `references/`
3. Validate your changes (see below)
4. Submit a pull request with a clear description of what you changed and why

### Adding New Reference Files

1. Create a new `.md` file in `app-intents-expert-skill/references/`
2. Keep it focused on a single topic
3. Include practical code examples with correct Swift syntax
4. Update `SKILL.md` to reference the new file where appropriate
5. Update `README.md` skill structure if adding a new file

### Reporting Issues

- Use GitHub Issues for incorrect information, missing coverage, or suggestions
- Include the specific file and section when reporting inaccuracies
- Reference Apple documentation or WWDC sessions when possible

## Quality Standards

### Content Requirements

- **Accuracy**: All code examples must compile and follow Apple's current APIs. Reference specific iOS/macOS versions for availability.
- **Conciseness**: Agents load reference files on demand. Keep files focused and under 300 lines where possible.
- **Practicality**: Provide complete, copy-paste ready code examples. Avoid pseudo-code.
- **Currency**: Prefer modern APIs (iOS 26+). When covering older APIs, clearly mark them with availability annotations.

### Format Requirements

- `SKILL.md` must stay under 500 lines (per Agent Skills spec recommendation)
- Frontmatter `name` must remain `app-intents-expert-skill` and match the directory name
- Frontmatter `description` must describe both what the skill does and when to use it
- Reference files should use relative paths from the skill root
- Keep file references one level deep from `SKILL.md`

### Code Examples

- Use Swift syntax with proper formatting
- Include `import` statements when they clarify which framework is needed
- Add brief comments for non-obvious patterns
- Show complete, minimal examples (not snippets that require imagination to complete)

## Validation

Before submitting, verify your changes:

```bash
# If you have skills-ref installed
skills-ref validate ./app-intents-expert-skill
```

Or manually check:
- [ ] `SKILL.md` frontmatter has valid `name` and `description`
- [ ] `name` matches directory name (`app-intents-expert-skill`)
- [ ] No broken relative links in `SKILL.md`
- [ ] Code examples use correct Swift syntax
- [ ] New files are referenced from `SKILL.md` where appropriate

## Pull Request Process

1. Ensure your branch is up to date with `main`
2. Include a description of changes and the reasoning
3. Reference any Apple documentation or WWDC sessions that support your changes
4. One topic per PR when possible

## Code of Conduct

Be respectful and constructive. We're all here to help developers build better App Intents integrations.
