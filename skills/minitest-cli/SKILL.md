---
name: minitest-cli
description: >
  Use the minitest CLI to manage user stories, upload builds, execute test runs
  on virtual devices (simulators/emulators), and analyse results. Use when the user asks to test
  their mobile app, create test scenarios, run tests, check test results, or
  manage builds via the command line. Also use after any code change that
  affects UI, navigation, or user journeys to check if existing tests need
  to be updated.
---

# Minitest CLI

`minitest` is a command-line tool for automated mobile app testing on virtual
devices (simulators & emulators). An AI agent analyses the app screen and verifies acceptance criteria
you define. You manage everything through the CLI: user stories, builds, runs, batches, and
results.

## Prerequisites

- Install: `curl -fsSL https://raw.githubusercontent.com/minitap-ai/minitest-cli/main/install.sh | bash`
- Authenticate: `minitest auth login` (opens browser for OAuth)
- Set target app: `export MINITEST_APP_ID=<uuid>` or pass `--app <uuid>` before
  any subcommand

## Global Flags

| Flag         | Effect                                                                      |
| ------------ | --------------------------------------------------------------------------- |
| `--json`     | camelCase JSON to stdout, diagnostics to stderr — ideal for piping          |
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
minitest --json apps list         # JSON array of {id, name, tenantId}
```

#### Creating apps

If the user does not yet have an app for the project, create one. The app
lives under a tenant; when the authenticated user belongs to a single
tenant the CLI auto-resolves it, otherwise pass `--tenant <id>` explicitly
(`apps list` exposes existing tenant IDs in JSON mode).

```bash
# Auto-resolve tenant (single-tenant users)
minitest apps create --name "My App"

# Explicit tenant; print just the new app id on stdout
minitest apps create --tenant <tenant_id> --name "My App"

# Full record as JSON, suitable for piping
minitest --json apps create --tenant <tenant_id> --name "My App" \
  --description "Mobile companion" --slug "my-app" --icon ./icon.png
```

In a multi-tenant non-interactive context (CI, piped invocation), `--tenant`
is required: the command exits 1 with a clear error otherwise.

### 2. Create user stories

A **user story** describes a user journey to test. It has a name, a type, an
optional description, and a list of **acceptance criteria** — plain-text
assertions the AI agent will verify visually on the device screen.

```bash
minitest --app <app_id> user-story create \
  --name "User Login" \
  --type login \
  --description "Email/password login from welcome screen" \
  --criteria "The login screen shows email and password fields" \
  --criteria "After submitting valid credentials, a loading indicator appears" \
  --criteria "The home screen is displayed after successful login"
```

**User story types:** `login`, `registration`, `onboarding`, `search`,
`settings`, `navigation`, `form`, `profile`, `other`, `custom`.

> **Restricted:** Do not create `checkout`, billing, or payment user stories —
> these involve real transactions and are not yet supported. Skip them during
> codebase analysis and inform the user.

**Test account requirement:** Before creating user stories that require login
or account-specific state, ensure the user provides test credentials via the
Minitest web app's test configuration. User stories should only cover journeys
the test account can actually perform.

**Acceptance criteria rules:**

- Must be **visually verifiable** (the agent only sees the screen)
- Must be **specific** and **unambiguous**
- One assertion per criterion
- Order them chronologically as they appear in the journey

Other user story commands:

```bash
minitest --app <app_id> user-story list
minitest --app <app_id> user-story get <user_story_id>
minitest --app <app_id> user-story update <user_story_id> --name "New Name"
minitest --app <app_id> user-story update <user_story_id> --add-criteria "New check"
minitest --app <app_id> user-story update <user_story_id> \
  --criteria "First check" --criteria "Second check"   # full replace
minitest --app <app_id> user-story delete <user_story_id> --force
```

> **Acceptance criteria are versioned.** `--criteria` fully replaces the set:
> unchanged content preserves identity (stable `criterionId`), modified content
> creates a new version on the same criterion, and removed items are
> soft-deleted. `--add-criteria` only appends.

### 3. Upload builds

Upload your `.apk` (Android) or `.ipa` (iOS) build artifacts. The platform is
auto-detected from the file extension. Only `.apk` and `.ipa` files are supported.

> **Important — virtual-device builds required:**
>
> Tests run on simulators/emulators, not physical devices. You must upload
> builds that are compatible with virtual devices:
>
> - **iOS:** a **Simulator build** (`.ipa` built for the iOS Simulator
>   destination, not a physical device). In Xcode: build for
>   *"Any iOS Simulator Device"* or a specific Simulator target.
> - **Android:** an **x86_64-compatible `.apk`**. Ensure your Gradle build
>   includes the `x86_64` ABI.
>
> Uploading a device-only build (e.g. an arm64 iOS archive or an arm-only
> Android APK) will cause test runs to fail.

```bash
minitest --app <app_id> build upload ./app-release.apk
minitest --app <app_id> build upload ./MyApp.ipa
minitest --app <app_id> build list
```

### 4. Run tests

Execute a user story on virtual devices. Provide at least one of
`--ios-build` or `--android-build`; single-platform apps may omit the other.

```bash
# Run a single user story (by name or UUID) and wait for results
minitest --app <app_id> run start "User Login" \
  --ios-build <ios_build_id> \
  --android-build <android_build_id>

# Android-only app
minitest --app <app_id> run start "User Login" \
  --android-build <android_build_id>

# Fire-and-forget (returns runId immediately — useful in CI)
minitest --app <app_id> --json run start "User Login" \
  --ios-build <ios_build_id> \
  --android-build <android_build_id> \
  --no-watch

# Run ALL user stories at once (creates one batch, fire-and-forget)
minitest --app <app_id> run all \
  --ios-build <ios_build_id> \
  --android-build <android_build_id>

# Cancel a running or pending run
minitest --app <app_id> run cancel <run_id>
```

Under the hood, `run start` and `run all` create a **batch**. A single run is
just a batch with one user story.

### 5. Check results

```bash
# Check a specific run
minitest --app <app_id> run status <run_id>

# Poll until completion
minitest --app <app_id> run status <run_id> --watch

# List all runs for a user story
minitest --app <app_id> run list "User Login"
minitest --app <app_id> run list "User Login" --status failed
minitest --app <app_id> run list "User Login" --all
```

**Run statuses:** `pending` → `running` → `completed` | `failed` | `cancelled`

A completed run includes per-platform results: pass/fail for each acceptance
criterion, fail reasons, and recording URLs.

### 6. Work with batches

A batch groups runs triggered together (by `run all`, CI, or a single
`run start`). Use the `batch` group to inspect or cancel them.

```bash
minitest --app <app_id> batch list                      # recent batches
minitest --app <app_id> batch list --status running
minitest --app <app_id> batch list --commit-sha abc1234
minitest --app <app_id> batch list --user-story <id>
minitest --app <app_id> batch get <batch_id>            # batch + its runs
minitest --app <app_id> batch cancel <batch_id>         # cancels all pending/running runs
```

**Batch statuses:** `pending` | `awaiting_build` | `running` | `completed` | `failed` | `cancelled`

### 7. Verify and acknowledge test maintenance

After making code changes, **always** check whether the changes affect existing
user stories before opening or updating a pull request. Follow this process:

1. **Review the impact** — look at the code changes and determine if they affect
   any screens, navigation, or user journeys covered by existing user stories.
   Use `minitest --app <app_id> user-story list` to see current user stories
   and their acceptance criteria.

2. **Propose changes to the user and wait for confirmation** — if your code
   changes modify UI, navigation, or behavior covered by existing user stories,
   do NOT silently update them. Present a clear summary of every proposed
   change and **wait for the user to explicitly approve** before running any
   `user-story create`, `user-story update`, or `user-story delete` commands:
   - New acceptance criteria for new functionality
   - Updated criteria for changed behavior
   - New user stories for entirely new user journeys
   - User stories to delete for removed features

   Never proceed without explicit user approval — the user must have the
   final say on what gets tested.

3. **Acknowledge** — once user stories are aligned with the code changes (or
   the user confirms no update is needed), stamp the HEAD commit:

```bash
minitest --app <app_id> maintenance-check "$(git rev-parse HEAD)"
```

If the app has maintenance checks enabled, a GitHub Check Run "Minitest
Maintenance" will appear on the PR. It fails until the HEAD commit is
acknowledged. Running `maintenance-check` flips it to success.

Possible error outcomes:

- _"Maintenance check is not enabled"_ — suggest the user enable automatic
  test maintenance checks at
  `https://app.minitap.ai/apps/<app_id>/test/settings`.
- _"App ... has no GitHub repository connected"_ — the CLI returns a link to
  `https://app.minitap.ai/settings/integrations` where the user can connect
  their GitHub repo.

**When to run:** after every commit that changes application code, before
opening or pushing to a PR. Do not acknowledge without first verifying that
user stories are still aligned with the code.

## CI / Automation Pattern

```bash
# Upload builds, run all user stories, collect results
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

Every command supports `--json`. JSON goes to stdout (camelCase keys, matching
the backend API), diagnostics go to stderr. Safe to pipe:

```bash
minitest --json user-story list | jq '.items[].name'
minitest --json run status <run_id> | jq '.status'
minitest --json batch list | jq '.items[] | {id, status, storyRuns: (.storyRuns | length)}'
```

## Quick Reference

| Task                | Command                                                                           |
| ------------------- | --------------------------------------------------------------------------------- |
| List apps           | `minitest apps list`                                                              |
| Create app          | `minitest apps create --name "My App" [--tenant ID] [--description ...] [--slug ...] [--icon ./icon.png]` |
| Create user story   | `minitest --app ID user-story create --name "..." --type login --criteria "..."` |
| List user stories   | `minitest --app ID user-story list`                                               |
| Update user story   | `minitest --app ID user-story update <id> --add-criteria "..."`                   |
| Upload build        | `minitest --app ID build upload ./app.apk`                                        |
| List builds         | `minitest --app ID build list`                                                    |
| Run one user story  | `minitest --app ID run start "Story Name" --ios-build X --android-build Y`        |
| Run all user stories| `minitest --app ID run all --ios-build X --android-build Y`                       |
| Cancel a run        | `minitest --app ID run cancel <run_id>`                                           |
| Check run           | `minitest --app ID run status <run_id>`                                           |
| List runs for story | `minitest --app ID run list "Story Name"`                                         |
| List batches        | `minitest --app ID batch list`                                                    |
| Get batch + runs    | `minitest --app ID batch get <batch_id>`                                          |
| Cancel batch        | `minitest --app ID batch cancel <batch_id>`                                       |
| Ack tests           | `minitest --app ID maintenance-check $(git rev-parse HEAD)`                       |
| Auth                | `minitest auth login`                                                             |
