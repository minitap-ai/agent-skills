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

Use placeholder credentials so the user can configure them on their backend:
- Username: `minitest-<role>@<appname>.app` (e.g. `minitest-pro@acme.app`)
- Password: `MiniTest-<Role>-2024!`

After creating profiles, clearly surface the credentials to the user in plain text so they can provision matching accounts in their system. Example:

```text
Created 2 test profiles:
  - Free User: minitest-free@acme.app / MiniTest-Free-2024!
  - Pro User:  minitest-pro@acme.app  / MiniTest-Pro-2024!
Set these credentials in your backend so the agent can sign in during test runs.
```

Fill the `about` field with what makes each profile distinct (e.g. "Pro subscription active, has saved items, payment method on file"). This context is injected into the tester agent's prompt at run time.

If the app uses a third-party auth provider (e.g. Google OAuth) and a shared Minitap account covers that flow, bind it to the relevant story instead of creating a new profile.

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

### 4. Upload builds

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

### 5. Run tests

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

A completed run includes per-platform results: pass/fail for each acceptance
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
| App dependency graph| `minitest apps dependencies <app_id>` (Mermaid flowchart to stdout)               |
| Create app          | `minitest apps create --name "My App" [--tenant ID] [--description ...] [--slug ...] [--icon ./icon.png]` |
| Create user story   | `minitest --app ID user-story create --name "..." --type login --criteria "..."` |
| Create user story with profile | `minitest --app ID user-story create --name "..." --type login --profile <profile_id> --criteria "..."` |
| List user stories   | `minitest --app ID user-story list`                                               |
| Update user story   | `minitest --app ID user-story update <id> --add-criteria "..."`                   |
| Set story dependencies | `minitest --app ID user-story update <id> --depends-on <parent_id> [--depends-on <parent_id2>]` |
| Remove a dependency | `minitest --app ID user-story update <id> --remove-dependency <parent_id>`        |
| List flow types     | `minitest flow-types list`                                                        |
| Read app knowledge  | `minitest app-knowledge get --app ID`                                             |
| Update app knowledge| `minitest app-knowledge update --app ID --content-file ./knowledge.md`            |
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
| Auth                | `minitest auth login`                                                             |
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
get pushed to the device before the agent runs the story. Use them for things
like profile photos, sample PDFs, or recordings the story under test depends on.

Bindings link profiles or files to a specific user story:

- Profile binding: at most one profile per story. `set-profile --clear` unbinds.
- File binding: many files per story. `set-files` is **atomic replace** — pass
  every file id you want bound; omitted ids are unbound. `--clear` unbinds all.

### Passwords on the CLI

`--password` accepts an inline value but is logged by your shell history. Prefer
`--password-stdin` and pipe the secret in:

```bash
printf "%s" "$MY_PASSWORD" | minitest --app $APP test-profile create \
  --name "Customer A" --username alice --password-stdin
```

The two flags are mutually exclusive. To wipe an existing password, use
`update --clear-password`.
