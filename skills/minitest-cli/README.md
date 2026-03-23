# Minitest CLI

Use the minitest CLI to manage testing flows, upload builds, execute test runs on real mobile devices, and analyse results.

## Overview

This skill teaches AI agents how to use the `minitest` command-line tool. Where the [minitest](../minitest/) skill uses MCP tools to manage flow templates directly, this skill drives the same workflows through the CLI — useful when MCP is not available or when working in CI/automation contexts.

## Use Cases

- "Run my mobile app tests"
- "Upload a build and start a test run"
- "Create testing flows via the CLI"
- "Check test results for my app"
- "Set up minitest in CI"

## What It Does

1. **Guides CLI installation and authentication** (`pip install minitest-cli`, `minitest auth login`)
2. **Manages testing flows** — create, list, update, and delete flows with acceptance criteria
3. **Handles build uploads** — upload `.apk` / `.app` artifacts
4. **Runs tests on real devices** — start runs, watch progress, collect results
5. **Supports CI/automation** — JSON output mode, exit codes, scripting patterns

## Requirements

- Python 3.10+
- `minitest-cli` installed (`pip install minitest-cli`)
- A Minitap account and registered app

## Installation

```
npx skills add minitap-ai/agent-skills --skill minitest-cli
```
