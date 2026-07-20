# Minitest CLI

Use the minitest CLI to manage testing flows, upload native builds, execute mobile and web app test runs, and analyse results.

## Overview

This skill teaches AI agents how to use the `minitest` command-line tool. Where the [minitest](../minitest/) skill uses MCP tools to manage flow templates directly, this skill drives the same workflows through the CLI — useful when MCP is not available or when working in CI/automation contexts.

## Use Cases

- "Run my mobile or web app tests"
- "Upload a native build and start a test run"
- "Run my web app tests from the CLI"
- "Create testing flows via the CLI"
- "Check test results for my app"
- "Set up minitest in CI"

## What It Does

1. **Guides CLI installation and authentication** (`curl -fsSL .../install.sh | bash`, `minitest auth login`)
2. **Manages testing flows** — create, list, update, and delete flows with acceptance criteria
3. **Handles native build uploads** — upload `.apk` / `.ipa` artifacts for Android/iOS apps
4. **Runs tests on mobile and web targets** — start runs, watch progress, collect results
5. **Supports CI/automation** — JSON output mode, exit codes, scripting patterns

## Requirements

- Python 3.10+
- `minitest-cli` installed (`curl -fsSL https://raw.githubusercontent.com/minitap-ai/minitest-cli/main/install.sh | bash`)
- A Minitap account and registered app

## Installation

```
npx skills add minitap-ai/agent-skills --skill minitest-cli
```
