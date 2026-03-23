# Minitap Testing Flows

Create and manage Minitap testing flow templates from a mobile app codebase.

## Overview

This skill helps AI agents create and maintain **flow templates** — automated test scenarios that will be executed on real mobile devices. The agent analyzes your codebase to identify testable user journeys and creates flow templates with visually verifiable acceptance criteria.

## Use Cases

- "Create testing flows for my app"
- "Generate test scenarios for my mobile app"
- "Reconcile testing flows with my codebase changes"
- "Update my Minitap tests"

## What It Does

1. **Discovers your app** via the `testing-service` MCP
2. **Analyzes your codebase** to identify user journeys (login, registration, checkout, etc.)
3. **Creates flow templates** with acceptance criteria that an AI agent can verify visually on a phone screen
4. **Reconciles changes** when your codebase evolves

## Flow Template Structure

Each flow template includes:
- **Name**: Human-readable title (e.g., "User Login")
- **Type**: Category (`login`, `registration`, `checkout`, `onboarding`, `search`, `settings`, `navigation`, `form`, `profile`, `other`)
- **Description**: Optional context
- **Acceptance Criteria**: Ordered list of visual assertions

## Requirements

- Access to the `testing-service` MCP with flow template tools
- A mobile app registered in Minitap

## Installation

```bash
npx skills add minitap-ai/agent-skills --skill minitap-testing-flows
```
