---
name: minitest-cli
description: >
  Use the minitest CLI to manage user stories, upload native builds, execute
  mobile and web app test runs, and analyse results. Use when the user asks to test
  their mobile or web app, create test scenarios, run tests, check test results, or
  manage native builds via the command line. Also use after any code change that
  affects UI, navigation, or user journeys to check if existing tests need
  to be updated.
---

# Minitest CLI

`minitest` is a command-line tool for automated mobile and web app testing. For
native apps it runs on virtual devices (simulators & emulators); for web apps it
runs browser-based targets configured on the app. An AI agent analyses the app
screen and verifies acceptance criteria you define. You manage everything through
the CLI: user stories, native builds, web lanes, runs, batches, and results.

## Prerequisites

- Install: `curl -fsSL https://raw.githubusercontent.com/minitap-ai/minitest-cli/main/install.sh | bash`
- Authenticate: `minitest auth login` (opens browser for OAuth)
- Set target app: `export MINITEST_APP_ID=<uuid>` or pass `--app <uuid>` before
  any subcommand

## Onboarding a new app (`minitest init`)

If you are onboarding an app from scratch, run `minitest init` first. It prints
an end-to-end onboarding playbook — the ordered steps to take this app from
nothing to a wired test suite: authenticate, find/create the app, define personas
(test profiles), map user journeys, and create scenarios with wired dependencies.
The current playbook then hands off to the Minitest web app to collect any needed
native build and launch the suite. Follow the printed steps in order, in the
app's repository.

```bash
minitest init            # prints the onboarding playbook (raw markdown when piped/non-interactive)
minitest init --agent    # force raw markdown output regardless of context
```

Conventions the playbook relies on:

- **Map all the main paths**: enumerate ALL the key user journeys the app
  genuinely warrants, not just a sample. Cover the happy paths AND, especially,
  the paths that can break: failure states, validation errors, permission/auth
  denials, empty states, and edge cases — these are what real testing must catch.
  Write goal-oriented acceptance criteria (each criterion is a job to be done,
  not a micro-step).
- **Offline criteria**: word them as "Offline (wifi off)" — mobile cloud test
  devices have no airplane mode, so never write "airplane mode".
- **File seeding**: if a scenario needs a file available in the test environment,
  `minitest test-file upload` it and bind it with
  `minitest user-story-binding set-files` before runs.

The sections below document each command the playbook refers to.

## Maintaining existing tests (`minitest maintenance`)

Use `minitest maintenance` when the app UI/code changed and the customer wants
their Minitest user stories updated without sharing GitHub/code with Minitap.
Run it from the app repository. The customer's own coding agent reads local code;
the CLI sends only proposed test-flow edits and an opaque local HEAD SHA.

```bash
# Fetch the server-composed maintenance brain (same rules as cloud maintenance)
minitest maintenance --agent

# First agent step: opens a maintenance run and returns context as JSON
minitest --json maintenance context
```

The `context` output includes:

- `mode`: `audit` on the first run, `incremental` after a watermark exists
- `fromSha`: null in audit mode; diff baseline in incremental mode
- `headSha`: local HEAD SHA reported by the CLI
- `context`: current stories, stable criterion ids, dependencies, app memory
- `guardrail`: pending Release Queue count; warn the user before continuing when non-zero

In incremental mode, inspect changes with `git diff <fromSha>..HEAD` locally.
In audit mode, inspect the full current codebase against every active story.

Post findings back through these mechanics-only commands:

```bash
minitest maintenance affected --file affected.json
minitest maintenance change --file change.json
minitest maintenance divergence --file divergence.json
minitest maintenance status --phase triage --message "Mapped affected stories" --progress 40
minitest maintenance complete --changed      # or --no-changed when nothing changed
```

The exact JSON fields for `affected.json`, `change.json`, and `divergence.json`
are specified by the maintenance brain fetched via `minitest maintenance --agent`;
follow that contract rather than inventing field names. `complete --changed` is
what advances the watermark to the current HEAD, so the next run resolves to
`incremental` mode; use `--no-changed` when nothing was edited.

At the end, always show the user the Release Queue review link so they can inspect
the proposed edits in the webapp, and offer to apply them right away — it is one
flow, not two modes:

```bash
minitest maintenance apply --review # prints the Release Queue link (no changes made)
minitest maintenance apply          # applies all pending edits now
```

Present it as a single choice: surface the review link, then ask whether the user
wants you to apply the edits now or review them manually through the link first.
Auto-applying is simply the option they can pick, not a separate mode.

Never ask the user to paste code or diffs into Minitap. The privacy contract is:
local code stays local; proposed criteria/dependency edits are the only payload.

## Authentication

Three credential sources, in priority order:

1. `MINITEST_TOKEN` — raw bearer override (legacy; usually unset).
2. `MINITEST_API_KEY` — tenant-scoped `mtk_…` key, recommended for CI/scripts.
3. `minitest auth login` — interactive OAuth.

If both `MINITEST_TOKEN` and `MINITEST_API_KEY` are set, `MINITEST_TOKEN` wins and a stderr warning is emitted once per process.

### Key rotation

`mtk_` keys are mintable and revocable but do not expire. To rotate: mint a new key, update the secret in your CI/orchestrator, then revoke the old key:

```
minitest auth api-key mint --tenant <tenant-id> --name new-ci
minitest auth api-key list --tenant <tenant-id>
minitest auth api-key revoke --tenant <tenant-id> --key <old-key-id>
```

### CI usage

```yaml
env:
  MINITEST_API_KEY: ${{ secrets.MINITEST_API_KEY }}
steps:
  - run: minitest apps list
```

Treat `MINITEST_API_KEY` as a credential. Never commit it; rotate on suspected leak.

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

#### Dependency graph

Visualise the user-story dependency DAG as a Mermaid flowchart — useful for
understanding the execution order before creating or modifying stories:

```bash
# Mermaid flowchart to stdout (LLM-friendly)
minitest apps dependencies <app_id>

# Raw graph JSON (nodes + edges)
minitest --json apps dependencies <app_id>

# Using global --app flag or MINITEST_APP_ID
minitest --app <app_id> apps dependencies
```

The output is a `flowchart TD` with each node labelled `"Story Name\n(type)"`
and directed edges showing dependency relationships (parent → child).

#### Creating apps

If the user does not yet have an app for the project, create one. The app
lives under a tenant; when the authenticated user belongs to a single
tenant the CLI auto-resolves it, otherwise pass `--tenant <id>` explicitly
(`apps list` exposes existing tenant IDs in JSON mode).

```bash
# Auto-resolve tenant (single-tenant users), native mobile app
minitest apps create --name "My Mobile App" --platform ios --platform android

# Explicit tenant; print just the new app id on stdout
minitest apps create --tenant <tenant_id> --name "My Web App" \
  --platform web --web-url https://example.com

# Full record as JSON, suitable for piping
minitest --json apps create --tenant <tenant_id> --name "My App" \
  --platform web --web-url https://example.com \
  --description "Customer portal" --slug "my-app" --icon ./icon.png
```

In a multi-tenant non-interactive context (CI, piped invocation), `--tenant`
is required: the command exits 1 with a clear error otherwise.

### 2. Create user stories

A **user story** describes a user journey to test. It has a name, a type, an
optional description, and a list of **acceptance criteria** — plain-text
assertions the AI agent will verify visually on the target screen (mobile device
or browser).

`--profile <profile_id>` is optional. If omitted, Minitest auto-assigns the app's default profile when one is configured.

```bash
minitest --app <app_id> user-story create \
  --name "User Login" \
  --type login \
  --profile <profile_id> \
  --description "Email/password login from welcome screen" \
  --criteria "The login screen shows email and password fields" \
  --criteria "After submitting valid credentials, a loading indicator appears" \
  --criteria "The home screen is displayed after successful login"
```

Use `--depends-on` to declare that this story must be run after another story
completes successfully (repeatable for multiple parents):

```bash
minitest --app <app_id> user-story create \
  --name "View Order History" \
  --type navigation \
  --depends-on <login_story_id> \
  --criteria "The order history screen is displayed"
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

### Test Profiles

When generating user stories, create a test profile for every distinct role or subscription tier the app requires. Each profile represents a unique starting state the agent needs to run a story (e.g. "Free User", "Pro User", "Admin", "Driver").

**Default to email-OTP personas.** Give each profile a `<prefix>@qa.minitap.ai` username and NO password. Every `@qa.minitap.ai` address delivers into a shared inbox the tester agent reads at run time, so it can sign in (or sign up) by pulling the login/verification code itself — no real credentials to manage:

```bash
minitest --app $APP test-profile create \
  --name "Pro User" --username "pro@qa.minitap.ai" \
  --about "Pro subscription active, has saved items, payment method on file"
```

A non-`@qa.minitap.ai` username with no password is rejected — keep the domain (or leave the username blank to auto-generate one).

**Bring-your-own account.** Only when the user supplies a real account they own (and the app needs a password) pass both, via stdin:

```bash
printf "%s" "$PASSWORD" | minitest --app $APP test-profile create \
  --name "Pro User" --username "real-user@example.com" --password-stdin --about "..."
```

**Specific account state (e.g. premium).** To exercise a flow that needs a particular state, proactively create a `<something>@qa.minitap.ai` persona WITH an explicit password, then ask the user to link that exact email+password combo to a pro/specific-state account in their backend. The `@qa.minitap.ai` address keeps the inbox readable for any OTP while the password lets them pre-provision the account state.

**No persona bound.** If a story has no profile, the agent defaults to anonymous (skip login). If a flow forces authentication, it self-generates a `<random>@qa.minitap.ai` with a runtime password, signs up, and reads the inbox for the confirmation/OTP code — so unbound scenarios still work without you provisioning anything.

Fill the `about` field with what makes each profile distinct (e.g. "Pro subscription active, has saved items, payment method on file"). This context is injected into the tester agent's prompt at run time.

If the app uses a third-party auth provider (e.g. Google OAuth), a shared Minitap account covers that flow — bind it to the relevant story instead of creating a new profile. Those shared pool addresses are also `@qa.minitap.ai`, so their inboxes are readable the same way.

Bind every story that requires authentication to its profile at creation time:
- Use `user-story create --profile <profile_id>` when you need a specific profile.
- If you omit `--profile`, ensure the app default profile is already configured so story creation auto-binds it.
- If needed, use `user-story-binding set-profile` immediately after creation.

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

#### Story dependencies

Use `--depends-on` / `--remove-dependency` on `user-story update` to manage
which stories gate this one:

```bash
# Replace the full dependency set (all parents at once)
minitest --app <app_id> user-story update <story_id> \
  --depends-on <parent_id_1> --depends-on <parent_id_2>

# Remove a single dependency without touching the others
minitest --app <app_id> user-story update <story_id> \
  --remove-dependency <parent_id>

# Clear all dependencies (pass empty --depends-on list)
minitest --app <app_id> user-story update <story_id> --depends-on ""
```

> `--depends-on` is a **full replace**: omitting a previously set parent removes
> it. Use `--remove-dependency` for a surgical delta when you only want to drop
> one parent. The two flags are mutually exclusive on the same invocation —
> `--remove-dependency` is ignored when `--depends-on` is also provided.

#### Device count

A story's **device count** is how many virtual devices a single run provisions.
By default it is **auto**: one device per bound persona (minimum 1). Set an
explicit override with `--device-count` on `user-story create` / `update`. The
value is capped server-side at `min(3, tenant device quota)`.

```bash
# Create a story that always runs on 2 devices
minitest --app <app_id> user-story create \
  --name "Two-player match" --type other --device-count 2 \
  --criteria "Both players see the shared game state"

# Override an existing story to 3 devices
minitest --app <app_id> user-story update <story_id> --device-count 3

# Reset back to auto (one device per bound persona)
minitest --app <app_id> user-story update <story_id> --device-count auto
```

Use more than one device only for flows that genuinely need concurrent devices:

- **Real-time cross-account flows** — two accounts interacting live (chat,
  multiplayer, sharing, calls) where one device acts and the other must observe.
- **Session-conflict same-account tests** — the same account signed in on two
  devices at once to exercise session invalidation, presence, or sync conflicts.

Single-user journeys (login, checkout, navigation) should stay on auto. `list`
and `get` show the effective device count when it is greater than 1; omitting
`--device-count` on create leaves the story on auto.

### 3. Reading flow types and app knowledge

When generating user stories programmatically (e.g. from an exploration
subagent), validate every `--type` value against the live list of flow types
before calling `user-story create` — invalid values exit non-zero.

```bash
# List valid flow (user-story) type values, one per line
minitest flow-types list

# Same data as a JSON array, easy to pipe into jq
minitest --json flow-types list
```

`flow-types list` wraps `GET /api/v1/user-story-types` on testing-service.
There is no public write endpoint at the time of writing — adding new types
requires a backend change.

For app-level prompt context (the markdown blob that conditions the AI agent
during runs), use `app-knowledge`:

```bash
# Read the current AppKnowledge content (markdown to stdout)
minitest app-knowledge get --app <app_id>

# Same plus version metadata as JSON
minitest --json app-knowledge get --app <app_id>

# Push a new version inline
minitest app-knowledge update --app <app_id> --content "# App Knowledge\n..."

# Or load it from a file (preferred for non-trivial markdown)
minitest app-knowledge update --app <app_id> --content-file ./app-knowledge.md
```

`app-knowledge update` calls `PUT /api/v1/apps/{app_id}/app-knowledge` and
prints the new `versionNumber` to stdout (full record with `--json`). Each
update creates a new prompt version — there is no rollback shortcut.

### 4. Upload native builds

For iOS/Android apps, upload your `.apk` (Android) or `.ipa` (iOS) build
artifacts. The platform is auto-detected from the file extension. Only `.apk`
and `.ipa` files are supported. Web apps do **not** upload builds; create the app
with `--platform web --web-url ...` and run the web lane with `--web`.

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

### 5. Run tests

Execute a user story on either native lanes or the web lane. For native runs,
provide at least one of `--ios-build` or `--android-build`; single-platform apps
may omit the other. For web runs, pass `--web` by itself — do **not** combine it
with native build flags. Web runs use the app's configured web URL and default web
targets; there are no per-run `--web-url`, `--browser`, or `--viewport` overrides
in the CLI.

```bash
# Run a single user story (by name or UUID) and wait for results
minitest --app <app_id> run start "User Login" \
  --ios-build <ios_build_id> \
  --android-build <android_build_id>

# Android-only app
minitest --app <app_id> run start "User Login" \
  --android-build <android_build_id>

# Web app (no build upload needed)
minitest --app <app_id> run start "User Login" --web

# Fire-and-forget (returns runId immediately — useful in CI)
minitest --app <app_id> --json run start "User Login" \
  --ios-build <ios_build_id> \
  --android-build <android_build_id> \
  --no-watch

# Run ALL user stories at once (creates one batch, fire-and-forget)
minitest --app <app_id> run all \
  --ios-build <ios_build_id> \
  --android-build <android_build_id>

# Run ALL user stories on web targets only
minitest --app <app_id> run all --web

# Cancel a running or pending run
minitest --app <app_id> run cancel <run_id>
```

Under the hood, `run start` and `run all` create a **batch**. A single run is
just a batch with one user story.

### 6. Check results

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

A completed run includes per-target results: pass/fail for each acceptance
criterion, fail reasons, and recording URLs.

### 7. Work with batches

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

## CI / Automation Pattern

```bash
# Native CI: upload builds, run all user stories, collect results
export MINITEST_APP_ID="<app_id>"

minitest --json build upload ./app.apk
minitest --json build upload ./MyApp.ipa

IOS_BUILD=$(minitest --json build list --platform ios --page-size 1 | jq -r '.[0].id')
ANDROID_BUILD=$(minitest --json build list --platform android --page-size 1 | jq -r '.[0].id')

minitest --json run all \
  --ios-build "$IOS_BUILD" \
  --android-build "$ANDROID_BUILD"

# Web CI: no build upload; use configured web targets
minitest --json run all --web
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
| Onboard a new app   | `minitest init` (prints the end-to-end onboarding playbook)                       |
| Maintain tests locally | `minitest maintenance --agent` (prints the CLI maintenance playbook)           |
| Open maintenance run | `minitest --json maintenance context`                                            |
| Post maintenance change | `minitest maintenance change --file change.json`                              |
| Complete maintenance run | `minitest maintenance complete --changed`                                    |
| Apply maintenance edits | `minitest maintenance apply` or `minitest maintenance apply --review`          |
| List apps           | `minitest apps list`                                                              |
| App dependency graph| `minitest apps dependencies <app_id>` (Mermaid flowchart to stdout)               |
| Create native app   | `minitest apps create --name "My App" --platform ios --platform android [--tenant ID] [--description ...] [--slug ...] [--icon ./icon.png]` |
| Create web app      | `minitest apps create --name "My Web App" --platform web --web-url https://example.com [--tenant ID]` |
| Create user story   | `minitest --app ID user-story create --name "..." --type login --criteria "..."` |
| Create user story with profile | `minitest --app ID user-story create --name "..." --type login --profile <profile_id> --criteria "..."` |
| List user stories   | `minitest --app ID user-story list`                                               |
| Update user story   | `minitest --app ID user-story update <id> --add-criteria "..."`                   |
| Set story dependencies | `minitest --app ID user-story update <id> --depends-on <parent_id> [--depends-on <parent_id2>]` |
| Remove a dependency | `minitest --app ID user-story update <id> --remove-dependency <parent_id>`        |
| Set story device count | `minitest --app ID user-story update <id> --device-count 2` (or `auto` to reset) |
| List flow types     | `minitest flow-types list`                                                        |
| Read app knowledge  | `minitest app-knowledge get --app ID`                                             |
| Update app knowledge| `minitest app-knowledge update --app ID --content-file ./knowledge.md`            |
| List env vars       | `minitest --app ID env list` (values masked; `--show` reveals)                    |
| Reveal one env var  | `minitest --app ID env get <KEY>` (prints the value verbatim to stdout)           |
| Set an env var      | `minitest --app ID env set <KEY> <VALUE> --yes [--dry-run]`                       |
| Remove an env var   | `minitest --app ID env unset <KEY> --yes [--dry-run]`                             |
| Clear all env vars  | `minitest --app ID env clear --yes [--dry-run]`                                   |
| Upload native build | `minitest --app ID build upload ./app.apk`                                        |
| List builds         | `minitest --app ID build list`                                                    |
| Run one native story| `minitest --app ID run start "Story Name" --ios-build X --android-build Y`        |
| Run one web story   | `minitest --app ID run start "Story Name" --web`                                 |
| Run all native stories | `minitest --app ID run all --ios-build X --android-build Y`                    |
| Run all web stories | `minitest --app ID run all --web`                                                 |
| Cancel a run        | `minitest --app ID run cancel <run_id>`                                           |
| Check run           | `minitest --app ID run status <run_id>`                                           |
| List runs for story | `minitest --app ID run list "Story Name"`                                         |
| List batches        | `minitest --app ID batch list`                                                    |
| Get batch + runs    | `minitest --app ID batch get <batch_id>`                                          |
| Cancel batch        | `minitest --app ID batch cancel <batch_id>`                                       |
| Auth                | `minitest auth login`                                                             |
| Mint API key        | `minitest auth api-key mint --tenant <id> --name <label>` (OAuth only)            |
| List API keys       | `minitest auth api-key list --tenant <id>`                                        |
| Revoke API key      | `minitest auth api-key revoke --tenant <id> --key <id>`                           |
| List test profiles  | `minitest --app ID test-profile list`                                             |
| List shared profiles| `minitest test-profile list-shared` (Minitap-provided pool; currently Google account only) |
| Create test profile | `minitest --app ID test-profile create --name "..." --username "..." --password-stdin` |
| Set default profile | `minitest --app ID test-profile set-default <profile_id>` |
| Clear default profile | `minitest --app ID test-profile clear-default` |
| Update test profile | `minitest --app ID test-profile update <id> [--name ...] [--clear-password]`      |
| Delete test profile | `minitest --app ID test-profile delete <id> --force`                              |
| List test files     | `minitest --app ID test-file list [--kind image\|document\|video\|audio\|other]`  |
| Upload test file    | `minitest --app ID test-file upload ./local/file.pdf --note "..."`                |
| Get test file       | `minitest --app ID test-file get <id>` (returns short-lived download URL)         |
| Update test file    | `minitest --app ID test-file update <id> [--name ...] [--clear-note]`             |
| Delete test file    | `minitest --app ID test-file delete <id> --force`                                 |
| Bind profile to story | `minitest --app ID user-story-binding set-profile <story_id> --profile <id>`    |
| Clear story profile | `minitest --app ID user-story-binding set-profile <story_id> --clear`             |
| Bind files to story | `minitest --app ID user-story-binding set-files <story_id> --file <id> --file <id>` |
| List story files    | `minitest --app ID user-story-binding list-files <story_id>`                      |

## Test profiles, test files, and story bindings

Test profiles let you store credentials (username/password/about) that the agent
will use when running a user story. They are app-scoped by default. Shared
profiles are Minitap-provided accounts available to all test-enabled tenants and
surface via `list-shared` (currently only a Google account).

Test files are arbitrary blobs (max 25 MB; image/video/audio/document/other) that
get pushed into the test environment before the agent runs the story. Use them
for things like profile photos, sample PDFs, or recordings the story under test
depends on.

Bindings link profiles or files to a specific user story:

- Profile binding: at most one profile per story. `set-profile --clear` unbinds.
- File binding: many files per story. `set-files` is **atomic replace** — pass
  every file id you want bound; omitted ids are unbound. `--clear` unbinds all.

## Environment variables

Apps can carry a set of environment variables (`KEY=value` pairs) that the test
runtime injects. They are app-scoped and stored encrypted server-side. Manage
them with the `env` group; every command needs an app (`--app ID` or
`MINITEST_APP_ID`).

```bash
minitest --app $APP env list            # values masked as ******** by default
minitest --app $APP env list --show     # reveal every value
minitest --app $APP env get API_TOKEN   # print ONE value verbatim to stdout
```

`env list` masks values so a screen-share or log never leaks a secret. `env get`
is the deliberate single-value reveal — it prints the raw value (and nothing
else) to stdout, so `VALUE=$(minitest --app $APP env get API_TOKEN)` is safe to
script.

Writes are **read-merge-write**: `set`/`unset` fetch the current set, apply your
change, and send the full map back, so they never clobber the other keys. Every
mutating command (`set`, `unset`, `clear`) requires explicit `--yes`/`-y` — it
never prompts. Without `--yes` it refuses and exits non-zero, which keeps an
agent or CI job from mutating secrets by accident.

```bash
minitest --app $APP env set API_TOKEN abc123 --yes
minitest --app $APP env set API_TOKEN abc123 --dry-run   # show the diff, change nothing
minitest --app $APP env unset OLD_KEY --yes
minitest --app $APP env clear --yes                      # deletes ALL env vars for the app
```

Use `--dry-run` first when unsure: it prints the added/changed/removed keys
(`+`/`~`/`-`) without touching the backend. `unset` a missing key and `clear`
with nothing configured both exit `4` (not found).

### Passwords on the CLI

`--password` accepts an inline value but is logged by your shell history. Prefer
`--password-stdin` and pipe the secret in:

```bash
printf "%s" "$MY_PASSWORD" | minitest --app $APP test-profile create \
  --name "Customer A" --username alice --password-stdin
```

The two flags are mutually exclusive. To wipe an existing password, use
`update --clear-password`.
