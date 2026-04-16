# Agent Skills

A collection of skills for AI coding agents. Skills are packaged instructions and scripts that extend agent capabilities.

Skills follow the [Agent Skills](https://agentskills.io/) format.

## Available Skills

### minitest-cli

Manage Minitest testing flows, upload builds, execute test runs, and acknowledge test maintenance via the `minitest` CLI.

Use when:
- "Create testing flows for my app"
- "Run mobile tests"
- "Upload build"
- After code changes that affect UI or user journeys

## Installation

```bash
npx skills add minitap-ai/agent-skills
```

To install a specific skill:

```bash
npx skills add minitap-ai/agent-skills --skill minitest-cli
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
