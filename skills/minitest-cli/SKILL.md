---
name: minitest-cli
description: >
  Use the minitest CLI to manage testing flows, upload builds, execute test runs
  on real mobile devices, and analyse results. Use when the user asks to test
  their mobile app, create test scenarios, run tests, check test results, or
  manage builds via the command line. Also use after any code change that
  affects UI, navigation, or user journeys to check if existing tests need
  to be updated.
---

# Minitest CLI

`minitest` is a command-line tool for automated mobile app testing on real
devices. An AI agent analyses the app screen and verifies acceptance criteria
you define. You manage everything through the CLI: flows, builds, runs, and
results.

## Prerequisites

- Install: `curl -fsSL https://raw.githubusercontent.com/minitap-ai/minitest-cli/main/install.sh | bash`
- Authenticate: `minitest auth login` (opens browser for OAuth)
- Set target app: `export MINITEST_APP_ID=<uuid>` or pass `--app <uuid>` before
  any subcommand

## Global Flags

| Flag         | Effect                                                                      |
| ------------ | --------------------------------------------------------------------------- |
| `--json`     | JSON to stdout, diagnostics to stderr — ideal for piping and parsing        |
| `--app <id>` | Target app (overrides `MINITEST_APP_ID`). Must appear before the subcommand |

## Exit Codes

| Code | Meaning                 |
| ---- | ----------------------- |
| 0    | Success                 |
| 1    | General error           |
| 2    | Authentication required |
| 3    | Network / API error     |
| 4    | Resource not found      |

## Core Workflow

### 1. Identify the app

```bash
minitest apps list                # find your app ID
minitest --json apps list         # JSON array of {id, name, tenant_id}
```

### 2. Create testing flows

A **flow** describes a user journey to test. It has a name, a type, an optional
description, and a list of **acceptance criteria** — plain-text assertions the
AI agent will verify visually on the device screen.

```bash
minitest --app <app_id> flow create \
  --name "User Login" \
  --type login \
  --description "Email/password login from welcome screen" \
  --criteria "The login screen shows email and password fields" \
  --criteria "After submitting valid credentials, a loading indicator appears" \
  --criteria "The home screen is displayed after successful login"
```

**Flow types:** `login`, `registration`, `onboarding`, `search`,
`settings`, `navigation`, `form`, `profile`, `other`.

> **Restricted:** Do not create `checkout`, billing, or payment flows — these
> involve real transactions and are not yet supported. Skip them during codebase
> analysis and inform the user.

**Test account requirement:** Before creating flows that require login or
account-specific state, ensure the user provides test credentials via
`minitest --app <app_id> config set`. Flows should only cover journeys the
test account can actually perform.

**Acceptance criteria rules:**

- Must be **visually verifiable** (the agent only sees the screen)
- Must be **specific** and **unambiguous**
- One assertion per criterion
- Order them chronologically as they appear in the journey

Other flow commands:

```bash
minitest --app <app_id> flow list
minitest --app <app_id> flow get <flow_id>
minitest --app <app_id> flow update <flow_id> --name "New Name" --add-criteria "New check"
minitest --app <app_id> flow delete <flow_id> --force
```

### 3. Upload builds

Upload your `.apk` (Android) or `.ipa` (iOS) build artifacts. The platform is
auto-detected from the file extension. Only `.apk` and `.ipa` files are supported.

```bash
minitest --app <app_id> build upload ./app-release.apk
minitest --app <app_id> build upload ./MyApp.ipa
minitest --app <app_id> build list
```

### 4. Run tests

Execute a flow on real devices. You must provide both an iOS and an Android
build ID.

```bash
# Run a single flow (by name or UUID) and wait for results
minitest --app <app_id> run start "User Login" \
  --ios-build <ios_build_id> \
  --android-build <android_build_id>

# Fire-and-forget (returns run ID immediately — useful in CI)
minitest --app <app_id> --json run start "User Login" \
  --ios-build <ios_build_id> \
  --android-build <android_build_id> \
  --no-watch

# Run ALL flows at once (always fire-and-forget)
minitest --app <app_id> run all \
  --ios-build <ios_build_id> \
  --android-build <android_build_id>
```

### 5. Check results

```bash
# Check a specific run
minitest --app <app_id> run status <run_id>

# Poll until completion
minitest --app <app_id> run status <run_id> --watch

# List all runs for a flow
minitest --app <app_id> run list "User Login"
minitest --app <app_id> run list "User Login" --status failed
minitest --app <app_id> run list "User Login" --all
```

**Run statuses:** `pending` -> `running` -> `completed` | `failed`

A completed run includes per-platform results: pass/fail for each acceptance
criterion, fail reasons, and recording URLs.

### 6. Verify and acknowledge test maintenance

After making code changes, **always** check whether the changes affect existing
test flows before opening or updating a pull request. Follow this process:

1. **Review the impact** - look at the code changes and determine if they affect
   any screens, navigation, or user journeys covered by existing test flows.
   Use `minitest --app <app_id> flow list` to see current flows and their
   acceptance criteria.

2. **Propose changes to the user** - if your code changes modify UI,
   navigation, or behavior covered by existing flows, do NOT silently update
   them. Present a summary of proposed changes and ask the user to confirm:
   - New acceptance criteria for new functionality
   - Updated criteria for changed behavior
   - New flows for entirely new user journeys
   - Flows to delete for removed features

   Only apply the updates after the user confirms.

3. **Acknowledge** - once tests are aligned with the code changes (or the user
   confirms no update is needed), stamp the HEAD commit:

```bash
minitest --app <app_id> maintenance-check "$(git rev-parse HEAD)"
```

If the app has maintenance checks enabled, a GitHub Check Run "Minitest
Maintenance" will appear on the PR. It fails until the HEAD commit is
acknowledged. Running `maintenance-check` flips it to success.

If the command returns "Maintenance check is not enabled", suggest the user
to enable automatic test maintenance checks in their app's test configuration
settings on the Minitest web app (`https://app.minitap.ai/apps/<app_id>/test/settings`).

**When to run:** after every commit that changes application code, before
opening or pushing to a PR. Do not acknowledge without first verifying that
tests are still aligned with the code.

## CI / Automation Pattern

```bash
# Upload builds, run all flows, collect results
export MINITEST_APP_ID="<app_id>"

minitest --json build upload ./app.apk
minitest --json build upload ./MyApp.ipa

IOS_BUILD=$(minitest --json build list --platform ios --page-size 1 | jq -r '.[0].id')
ANDROID_BUILD=$(minitest --json build list --platform android --page-size 1 | jq -r '.[0].id')

minitest --json run all \
  --ios-build "$IOS_BUILD" \
  --android-build "$ANDROID_BUILD"
```

## JSON Output

Every command supports `--json`. JSON goes to stdout, diagnostics to stderr.
This makes it safe to pipe:

```bash
minitest --json flow list | jq '.[].name'
minitest --json run status <run_id> | jq '.status'
```

## Quick Reference

| Task          | Command                                                                    |
| ------------- | -------------------------------------------------------------------------- |
| List apps     | `minitest apps list`                                                       |
| Create flow   | `minitest --app ID flow create --name "..." --type login --criteria "..."` |
| List flows    | `minitest --app ID flow list`                                              |
| Upload build  | `minitest --app ID build upload ./app.apk`                                 |
| List builds   | `minitest --app ID build list`                                             |
| Run one flow  | `minitest --app ID run start "Flow Name" --ios-build X --android-build Y`  |
| Run all flows | `minitest --app ID run all --ios-build X --android-build Y`                |
| Check run     | `minitest --app ID run status RUN_ID`                                      |
| List runs     | `minitest --app ID run list "Flow Name"`                                   |
| Ack tests     | `minitest --app ID maintenance-check $(git rev-parse HEAD)`                |
| Auth          | `minitest auth login`                                                      |
