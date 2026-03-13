# Agent Skills

A collection of skills for AI coding agents. Skills are packaged instructions and scripts that extend agent capabilities.

Skills follow the [Agent Skills](https://agentskills.io/) format.

## Available Skills

### minitap-testing-flows

Create and manage Minitap testing flow templates from a mobile app codebase. Use when building automated test scenarios for mobile apps.

Use when:
- "Create testing flows for my app"
- "Generate test scenarios"
- "Reconcile testing flows"
- "Update my Minitap tests"

## Installation

```bash
npx skills add minitap-ai/agent-skills
```

To install a specific skill:

```bash
npx skills add minitap-ai/agent-skills --skill minitap-testing-flows
```

## Usage

Skills are automatically available once installed. The agent will use them when relevant tasks are detected.

## Skill Structure

Each skill contains:
- `SKILL.md` - Instructions for the agent
- `metadata.json` - Skill metadata (name, description, version)
- `references/` - Supporting documentation (optional)

## License

MIT
