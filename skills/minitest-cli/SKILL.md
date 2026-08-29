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
<!-- skill-references-hash: 4863f6d2a0c168ba2caa42be6b9f5262082a6e80c7be489f5dfcf2ec22ccc51d -->

# Minitest CLI

`minitest` is a command-line tool for automated mobile and web app testing. For
native apps it runs on virtual devices (simulators & emulators); for web apps it
runs browser-based targets configured on the app. An AI agent analyses the app
screen and verifies acceptance criteria you define. You manage everything through
the CLI: user stories, native builds, web lanes, runs, batches, and results.

## Command shape

Every invocation follows one shape. The global options come **before** the
subcommand, because they are parsed by the root callback:

```
minitest [--json] [--app <APP_ID>] <subcommand> [args…] [subcommand flags…]
```

```bash
minitest --json --app 713c6550-7d36-41aa-b1cd-3240b3a0dda1 user-story list
minitest --json --app $APP run verdicts <batch_id> --actionable
```

**Agents must always pass `--json`.** Without it the CLI prints Rich tables and
panels meant for humans — box-drawing characters, truncated columns, colour
escapes — which are lossy and unparseable. `--json` writes camelCase JSON to
stdout and keeps every diagnostic on stderr, so `| jq` is always safe.

`--app` may be replaced by `export MINITEST_APP_ID=<uuid>`; the flag wins when
both are set. Commands that are tenant-scoped rather than app-scoped (`apps`,
`auth`, `flow-types`) do not require it, though `flow-types` needs it as a tenant
hint when your account spans several tenants.

### Three exceptions to the shape

| Command group | Deviation |
| --- | --- |
| `app-knowledge` | Declares its **own** `--app`, so it goes **after** the subcommand: `minitest --json app-knowledge get --app <id>`. The global pre-subcommand `--app` exits 2. |
| `auth api-key` | Declares its **own** `--json`, so it goes **after** the subcommand: `minitest auth api-key list --tenant <id> --json`. The global `--json` is silently ignored and a table is printed. |
| `flow-types` | `--app` must stay in the global position; `flow-types list --app <id>` exits 2. |

### Commands that do not emit JSON

`--json` is still harmless on these, but their payload is markdown or a raw
value by design:

- `minitest init --agent`, `minitest maintenance --agent`, `minitest skill` —
  print playbook markdown for the agent to read.
- `minitest --app $APP env get <KEY>` — prints the raw value verbatim **only
  without** `--json`; with `--json` it returns `{"KEY": "value"}`. Use the
  non-JSON form when capturing into a shell variable.

## Prerequisites

- Install: `curl -fsSL https://raw.githubusercontent.com/minitap-ai/minitest-cli/main/install.sh | bash`
- Authenticate: `minitest auth login` (opens browser for OAuth), check with
  `minitest --json auth status`
- Set target app: `export MINITEST_APP_ID=<uuid>` or pass `--app <uuid>` before
  any subcommand
- Upgrade: `minitest upgrade` (check the installed version with `minitest --version`)

## Onboarding a new app (`minitest init`)

If you are onboarding an app from scratch, run `minitest init` first, in the
app's repository. It prints a short playbook covering only what is specific to
the CLI entry point — authenticating, resolving the app that was already created
for you, and where onboarding stops (the Minitest web app collects any needed
native build and launches the suite). For the suite itself the playbook sends
you to **[Designing a full test suite](#designing-a-full-test-suite)** below;
follow that methodology rather than improvising one.

```bash
minitest init            # prints the onboarding playbook (raw markdown when piped/non-interactive)
minitest init --agent    # force raw markdown output regardless of context
```

The playbook is served by Minitap so it stays in step with the methodology; the
CLI falls back to an embedded copy when it cannot reach the API (which is the
normal case before `minitest auth login`).

## Designing a full test suite

When the user wants to design a **complete** test suite from an app codebase —
not just add a story or two — follow the disciplined multi-wave workflow in
[`reference/test-suite-design.md`](reference/test-suite-design.md). It takes an
app repository to a reviewed suite through ordered waves of read-only codebase
analysis: recon, surface mapping, gating/persona discovery, state modelling,
suite design, and adversarial verification.

Two companion references support that workflow:

- [`reference/suite-schemas.md`](reference/suite-schemas.md) — the local
  `suite.yaml` schema and the step-writing rules the workflow's artifacts follow.
- [`reference/minitest-target.md`](reference/minitest-target.md) — the executor
  envelope: what the Minitest tester agent (Mini) can and cannot run or observe,
  so you never design a scenario it can't execute.

The workflow ends by **applying the suite through the ordinary `minitest` CLI
commands already documented in this SKILL.md** — `test-profile create`,
`user-story create` (with `--criteria`, `--profile`, `--depends-on`),
`app-knowledge update`, and so on. There is no special apply command: the
reviewed suite is replayed as regular CLI calls in dependency order.

If a scenario needs a file available in the test environment, `minitest
test-file upload` it and bind it with `minitest user-story-binding set-files`
while applying the suite.

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
minitest --json maintenance affected --file affected.json
minitest --json maintenance change --file change.json
minitest --json maintenance divergence --file divergence.json
minitest --json maintenance status --phase triage --message "Mapped affected stories" --progress 40
minitest --json maintenance complete --changed   # or --no-changed when nothing changed
```

> `maintenance status`, `complete`, and `apply` print nothing on stdout under
> `--json` — judge them by their exit code (0 = accepted), or drop `--json` to
> read the human confirmation.

The exact JSON fields for `affected.json`, `change.json`, and `divergence.json`
are specified by the maintenance brain fetched via `minitest maintenance --agent`;
follow that contract rather than inventing field names — the endpoint currently
returns a 500 for some tenants, and without it the payload shapes cannot be guessed
(the API rejects invented fields). `complete --changed` is
what advances the watermark to the current HEAD, so the next run resolves to
`incremental` mode; use `--no-changed` when nothing was edited.

At the end, always show the user the Release Queue review link so they can inspect
the proposed edits in the webapp, and offer to apply them right away — it is one
flow, not two modes:

```bash
minitest maintenance apply --review # prints the Release Queue link (no changes made)
minitest maintenance apply          # applies all pending edits now
```

Leave `--json` off these two: they only ever print human text, and the review
link is the whole point of `--review`.

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

```bash
minitest auth api-key mint --tenant <tenant-id> --name new-ci --json
minitest auth api-key list --tenant <tenant-id> --json
minitest auth api-key revoke --tenant <tenant-id> --key <old-key-id> --json
```

Note the trailing `--json`: this group owns its own flag, and the global
pre-subcommand `--json` is ignored here.

### CI usage

```yaml
env:
  MINITEST_API_KEY: ${{ secrets.MINITEST_API_KEY }}
steps:
  - run: minitest --json apps list
```

Treat `MINITEST_API_KEY` as a credential. Never commit it; rotate on suspected leak.

## Global Flags

| Flag / environment | Effect                                                                |
| ------------------ | --------------------------------------------------------------------- |
| `--json`     | camelCase JSON to stdout, diagnostics to stderr. Must appear before the subcommand. **Always pass it when driving the CLI from an agent** |
| `--app <id>` | Target app (overrides `MINITEST_APP_ID`). Must appear before the subcommand |
| `MINITEST_CHANNEL` | Overrides the `X-Minitest-Channel` header value (default `cli`) for provenance tagging of automated editors |

## Exit Codes

| Code | Meaning                                 |
| ---- | --------------------------------------- |
| 0    | Success                                 |
| 1    | General error                           |
| 2    | Authentication required (`auth` commands) |
| 3    | Network / API error                     |
| 4    | Resource not found                      |
| 5    | Build rejected as invalid               |
| 6    | Conflict — re-read, rebuild, retry once |

Credentials rejected on a normal command exit 1, not 2 — only the `auth`
commands themselves use 2.

Only 3 is worth a blind retry. **6 is not a transport failure**: something you
based the call on moved (a stale `expectedMainRev`, an `expectedVersion` that no
longer matches, a branch mid-rebase). Re-read the resource, rebuild the request
against what it now returns, and try **once** more — retrying the same body
loops forever.

## Core Workflow

### 1. Identify the app

```bash
minitest --json apps list         # bare JSON array of {id, name, tenantId, …}
```

#### Dependency graph

Visualise the user-story dependency DAG as a Mermaid flowchart — useful for
understanding the execution order before creating or modifying stories:

```bash
# Raw graph JSON (nodes + edges)
minitest --json apps dependencies <app_id>

# Same, taking the app from the global flag or MINITEST_APP_ID
minitest --json --app <app_id> apps dependencies

# Without --json: a Mermaid `flowchart TD` for a human to paste into a doc
minitest apps dependencies <app_id>
```

The Mermaid form labels each node `"Story Name\n(type)"` with directed edges
showing dependency relationships (parent → child).

Before changing dependencies, agents must simulate the delta through the CLI
instead of computing graph effects themselves:

```bash
minitest --json --app <app_id> apps dependencies <app_id> --simulate \
  --add <story_id>:<depends_on_id> \
  --remove <story_id>:<depends_on_id>
```

Simulation is read-only and returns `valid`, `cycle` (including the cycle path
when invalid), `addedEdges`, `removedEdges`, `affectedStories`, `runOrder`
(parallel waves), and the resulting edges. An edge `A:B` means “A depends on
B”, so B runs first. `--add` and `--remove` require `--simulate`, and
`--simulate` requires at least one of them.

#### Creating apps

If the user does not yet have an app for the project, create one. The app
lives under a tenant; when the authenticated user belongs to a single
tenant the CLI auto-resolves it, otherwise pass `--tenant <id>` explicitly
(`apps list` exposes existing tenant IDs in JSON mode).

```bash
# Auto-resolve tenant (single-tenant users), native mobile app
minitest --json apps create --name "My Mobile App" --platform ios --platform android

# Full record as JSON, suitable for piping
minitest --json apps create --tenant <tenant_id> --name "My App" \
  --platform web --web-url https://example.com \
  --description "Customer portal" --slug "my-app" --icon ./icon.png
```

In a multi-tenant non-interactive context (CI, piped invocation), `--tenant`
is required: the command exits 1 with a clear error otherwise.

> An app created with `--platform web` still has **no web execution target**
> until one is configured in the Minitest web app, and `run start --web` refuses
> to launch until then. Creating a web app end-to-end is not a CLI-only flow.
> There is also no `apps delete`, so avoid creating throwaway apps.

### 2. Create user stories

A **user story** describes a user journey to test. It has a name, a type, an
optional description, and a list of **acceptance criteria** — plain-text
assertions the AI agent will verify visually on the target screen (mobile device
or browser).

`--profile <profile_id>` is optional. If omitted, Minitest auto-assigns the app's default profile when one is configured.

```bash
minitest --json --app <app_id> user-story create \
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
minitest --json --app <app_id> user-story create \
  --name "View Order History" \
  --type navigation \
  --depends-on <login_story_id> \
  --criteria "The order history screen is displayed"
```

**User story types:** `login`, `registration`, `checkout`, `onboarding`,
`search`, `settings`, `navigation`, `form`, `profile`, `other`, `custom`.

> **Ask first:** Do not create `checkout`, billing, or payment user stories
> until you know how this app expects payment to be exercised. The hard limit is
> narrow — the tester never enters a real card and never makes a real-money
> purchase — but everything short of that is testable: sandbox and test cards,
> in-app purchases and subscriptions through the RevenueCat Test Store or an
> Apple/Google platform sandbox, and whatever custom payment flow the app uses.
> What breaks a run is guessing, not the payment step itself.
>
> So when you find a paid flow during codebase analysis, ask the user which
> applies: a test card and its number, a build wired to the RevenueCat Test Store
> with its Test Store API key, an Apple or Google platform sandbox account, a
> bypass or promo code, a staging payment provider, or none of these — in which
> case the story stops before the charge and says so in its last criterion. The
> two RevenueCat environments are not interchangeable, so record which one this
> build uses: the Test Store mocks billing entirely and needs no store setup,
> while a platform sandbox drives the real App Store or Play purchase flow.
>
> Record the answer where the tester will read it at run time: the flow type's
> `--usage-prompt`
> (e.g. `"Paid plans are sandboxed: use card 4242 4242 4242 4242."`), the app
> knowledge, or the persona's `--about`.
>
> Once you have that answer, author these stories like any other. Do not refuse a
> whole feature over the payment step it ends on, and never invent a restriction
> of your own or bake one into a flow type.

**Test account requirement:** Before creating user stories that require login
or account-specific state, ensure the user provides test credentials via the
Minitest web app's test configuration. User stories should only cover journeys
the test account can actually perform.

### Test Profiles

When generating user stories, create a test profile for every distinct role or subscription tier the app requires. Each profile represents a unique starting state the agent needs to run a story (e.g. "Free User", "Pro User", "Admin", "Driver").

**Default to email-OTP personas.** Give each profile a `<prefix>@qa.minitap.ai` username and NO password. Every `@qa.minitap.ai` address delivers into a shared inbox the tester agent reads at run time, so it can sign in (or sign up) by pulling the login/verification code itself — no real credentials to manage:

```bash
minitest --json --app $APP test-profile create \
  --name "Pro User" --username "pro@qa.minitap.ai" \
  --about "Pro subscription active, has saved items, payment method on file"
```

A non-`@qa.minitap.ai` username with no password is rejected — keep the domain (or leave the username blank to auto-generate one).

**Bring-your-own account.** Only when the user supplies a real account they own (and the app needs a password) pass both, via stdin:

```bash
printf "%s" "$PASSWORD" | `minitest --json --app $APP test-profile create \
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
minitest --json --app <app_id> user-story list
minitest --json --app <app_id> user-story get <user_story_id>
minitest --json --app <app_id> user-story update <user_story_id> --name "New Name"
minitest --json --app <app_id> user-story update <user_story_id> --add-criteria "New check"
minitest --json --app <app_id> user-story update <user_story_id> \
  --set-criterion <criterion-id-or-index>="New text"
minitest --json --app <app_id> user-story update <user_story_id> \
  --revert-criterion <criterion-id-or-index>=<version_id>
minitest --json --app <app_id> user-story update <user_story_id> \
  --criteria "First check" --criteria "Second check"   # full replace
minitest --json --app <app_id> user-story delete <user_story_id> --force
```

> **Acceptance criteria are versioned.** For rewording, always prefer repeatable
> `--set-criterion <criterion-id-or-index>="new text"`: it changes one criterion
> while preserving its identity and version history.
> `--revert-criterion <criterion-id-or-index>=<version_id>` restores that version's exact text and
> records the change as a revert. Both flags conflict with `--criteria` and
> `--add-criteria`. `--criteria` fully replaces the set and severs history for
> reworded criteria; `--add-criteria` only appends.

> **Use `criterionId`, not `id`.** Each entry of `acceptanceCriteria[]` in
> `user-story get --json` carries both: `id` is the *version* id and `criterionId`
> is the stable identity. `--set-criterion` accepts `criterionId` or a 1-based
> index; passing `id` fails with "No criterion matches id".
> `--revert-criterion <criterionId>=<id-of-the-target-version>` takes one of each.

#### Story dependencies

Use `--depends-on` / `--remove-dependency` on `user-story update` to manage
which stories gate this one:

```bash
# Replace the full dependency set (all parents at once)
minitest --json --app <app_id> user-story update <story_id> \
  --depends-on <parent_id_1> --depends-on <parent_id_2>

# Remove a single dependency without touching the others
minitest --json --app <app_id> user-story update <story_id> \
  --remove-dependency <parent_id>

# Clear all dependencies (pass empty --depends-on list)
minitest --json --app <app_id> user-story update <story_id> --depends-on ""
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
minitest --json --app <app_id> user-story create \
  --name "Two-player match" --type other --device-count 2 \
  --criteria "Both players see the shared game state"

# Override an existing story to 3 devices
minitest --json --app <app_id> user-story update <story_id> --device-count 3

# Reset back to auto (one device per bound persona)
minitest --json --app <app_id> user-story update <story_id> --device-count auto
```

Use more than one device only for flows that genuinely need concurrent devices:

- **Real-time cross-account flows** — two accounts interacting live (chat,
  multiplayer, sharing, calls) where one device acts and the other must observe.
- **Session-conflict same-account tests** — the same account signed in on two
  devices at once to exercise session invalidation, presence, or sync conflicts.

Single-user journeys (login, checkout, navigation) should stay on auto. `list`
and `get` show the effective device count when it is greater than 1; omitting
`--device-count` on create leaves the story on auto.

#### Camera media (web runs)

Use `--camera-media` on `user-story create` or `user-story update` to feed a
specific image or video into the virtual webcam during web runs (e.g. a story
that uploads an ID photo or scans a QR code). The value is either a **local
file path** to upload or an **existing test-file ID** to reuse:

```bash
# Upload a local image/video and attach it to a new story
minitest --app <app_id> user-story create \
  --name "Scan QR to check in" --type navigation \
  --camera-media ./fixtures/checkin-qr.png

# Reuse an already-uploaded test file by its ID
minitest --app <app_id> user-story update <story_id> \
  --camera-media <test_file_id>

# Reset the story back to the built-in default camera feed
minitest --app <app_id> user-story update <story_id> --clear-camera-media
```

> A **UUID** value is treated as an existing test-file ID and reused as-is; any
> other value is a **local path** that gets uploaded as a test file first. The
> file must be an image or video within the size caps — **video ≤ 50 MB, image
> ≤ 25 MB**. `--camera-media` and `--clear-camera-media` are mutually exclusive.

### 3. Reading and managing flow types, and app knowledge

> Needs **minitest-cli ≥ 0.22.0**. On an older CLI the `create` / `update` /
> `delete` subcommands below do not exist, and `--json flow-types list` still
> returns a flat array of names. Check with `minitest --version` and run
> `minitest upgrade` before assuming a command is broken.

When generating user stories programmatically (e.g. from an exploration
subagent), validate every `--type` value against the live list of flow types
before calling `user-story create` — invalid values exit non-zero.

```bash
# Bare JSON array, with each custom type's id and presentation fields
minitest --json flow-types list
# [{"name": "login",    "custom": false, "id": null,    "icon": null,          …},
#  {"name": "Billing",  "custom": true,  "id": "3fa85…", "icon": "credit-card", …}]

# Just the names, for a membership check
minitest --json flow-types list | jq -r '.[].name'
```

The list is the built-in types plus your tenant's custom ones. Flow types are
**tenant-scoped**, so `--app` is only a tenant hint. It is nonetheless
**required whenever your account spans more than one tenant** — otherwise the
command exits non-zero asking which tenant to act on. Keep it in the global
position (`minitest --json --app <id> flow-types list`); a trailing `--app`
exits 2.

When no built-in type fits a journey, create your own instead of forcing it into
`other`:

```bash
minitest --json flow-types create --name "Subscription" \
  --usage-prompt "Paid plans are sandboxed: use card 4242 4242 4242 4242."

# Optional presentation flags, defaults are tag/gray
minitest --json flow-types create --name "Subscription" --icon credit-card --color green
```

`--usage-prompt` is extra context handed to the testing agent whenever it runs a
story of that type — use it for domain rules that apply to the whole family of
flows. Creating a name that already exists (or that collides with a built-in)
fails with the server's conflict message.

Rename or restyle an existing custom type with `update`, addressing it by name
or by id. The type keeps its identity, so stories already on it stay on it:

```bash
minitest --json flow-types update "Subscription" --name "Billing"
minitest --json flow-types update "Billing" --usage-prompt "Card 4242… declines above 500."
```

`delete` removes a custom type and **resets every user story on it to `other`**,
so it refuses to run without `--yes`:

```bash
minitest --json flow-types delete "Billing" --yes
```

Pass the type name to any user-story command; the CLI resolves it for you
(matching is case-insensitive):

```bash
minitest --json --app <app_id> user-story create --name "Upgrade to Pro" --type "Billing" \
  --criteria "The plan badge reads Pro after checkout"

minitest --json --app <app_id> user-story list --type "Billing"
```

`flow-types list` wraps `GET /api/v1/user-story-types` (built-ins) plus
`GET /api/v1/apps/<app_id>/custom-user-story-types`; `create`, `update` and
`delete` wrap `POST` / `PATCH` / `DELETE` on the latter. The app in that path is
only addressing: the CLI picks one of your apps automatically when you do not
pass `--app` and your account has a single tenant.

For app-level prompt context (the markdown blob that conditions the AI agent
during runs), use `app-knowledge`:

Note the `--app` position: this group declares its own, so it goes **after** the
subcommand and the global pre-subcommand form exits 2.

```bash
# {"appId": …, "content": "<markdown>"} — no version metadata on read,
# and exit 4 when the app has no knowledge set yet
minitest --json app-knowledge get --app <app_id>

# Push a new version inline
minitest --json app-knowledge update --app <app_id> --content "# App Knowledge\n..."

# Or load it from a file (preferred for non-trivial markdown)
minitest --json app-knowledge update --app <app_id> --content-file ./app-knowledge.md
```

`app-knowledge update` calls `PUT /api/v1/apps/{app_id}/app-knowledge` and
prints the new `versionNumber` to stdout (full record with `--json`). Each
update creates a new prompt version — there is no rollback shortcut.

### 3b. See what exploration actually mapped (`minitest screens`)

Before you invent scenarios for an app, read the screen map: it is the list of
screens the exploration crawl genuinely stood on, so it tells you what the app
really contains instead of what its name or flow types imply.

```bash
# Every mapped screen, shallowest first
minitest --app <app_id> screens list

# One platform only
minitest --app <app_id> screens list --platform ios

# The navigation shape — the fastest way to see where a crawl stopped
minitest --app <app_id> screens list --tree

# Only the screens the crawl could not get past
minitest --app <app_id> screens list --blocked

# One screen: how to reach it, and where it leads
minitest --app <app_id> screens get "Onboarding age"
```

`screens list` wraps `GET /api/v1/apps/{app_id}/screens`, which returns the
whole map in one call (screenshots are signed in a single batch). Filtering by
`--platform` is server-side; `--area` and `--blocked` are applied client-side
over that one response.

With `--json` the command returns the map itself, and `screenCount` always
matches the `screens` array beside it (so a filtered call reports the filtered
count, not the server total). `--tree` is a human rendering only — under
`--json` it is ignored and you get the same map object, whose `outgoing` edges
carry the graph. Note the casing seam inherited from the API: the envelope is
camelCase (`screenKey`, `displayName`) while `outgoing` and `context` are
snake_case (`to_screen_key`, `parked_reason`, `requires_auth`).

**Read the frontier line**, printed under the table. It reports three things
that each mean something different:

| Signal | Meaning |
| ------ | ------- |
| *parked edge* | The crawl saw a way onward and deliberately did not follow it (a duplicate branch, a login wall, a non-navigating toggle). Parked edges are the unexplored frontier. |
| *blocked screen* | The crawl reached the screen but could not get past it. `screens get` shows the reason, and the ask gating it. |
| *edge leading to a screen with no row* | The crawl recorded a step to a destination that was never written as a screen — usually the destination was named slightly differently than the screen later called itself. The map understates what was reached. |

Two shapes worth recognising in `--tree` output:

- **A long unbranching chain** — exploration never escaped a funnel (typically
  onboarding). Scenarios written from this map will all be onboarding scenarios.
- **A wide, shallow tree** — exploration never got past the lobby, usually
  because of auth. Check `screens get` for `Requires auth`.

An empty result is not an error. It means no crawl has run against a build for
this app yet — the map is written by exploration as it walks, so it stays empty
until then.

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
minitest --json --app <app_id> build upload ./app-release.apk
minitest --json --app <app_id> build upload ./MyApp.ipa
minitest --json --app <app_id> build list --platform ios --page-size 1
```

Pass `--platform ios|android` to `build upload` when the filename does not end
in `.ipa` / `.apk` and auto-detection cannot fire.

For a repo-connected app, queue a build directly from GitHub without starting
tests:

```bash
minitest --json --app <app_id> build from-commit [<full_sha>] \
  [--platform ios|android|web] [--platform ...] [--force-full]
```

The SHA is optional. When omitted, the platform resolves the default branch's
HEAD. When supplied, it must be the full 40-character lowercase hexadecimal
SHA. `--force-full` bypasses incremental build caches. Inspect failures with
`build list --status failed`; use `--kind web` for web builds because web rows
may have no `platform`, and therefore are not selected by `--platform web`.

`build list` returns completed builds only unless you pass `--status`, so always
use `--status failed` to see failures at all. `--status` is repeatable.

Every item in the `build list` JSON carries a `guidance` object next to the raw
envelope fields. It is `null` when the build did not fail, so the key is always
present:

```json
{ "source": "fix_prompt", "text": "..." }
```

`source` tells you how much to trust `text`. Read it before acting:

| `source` | Meaning |
|---|---|
| `fix_prompt` | Machine-authored and actionable. Act on it directly. |
| `remediation` | Human-readable next step. Reliable, less specific. |
| `summary` | One-line classification only. Investigate before changing code. |
| `raw` | Unstructured builder stderr, no envelope was recorded. May be a stale heartbeat or a timeout rather than a real defect. Treat as a lead, not a diagnosis. |
| `withheld` | An internal failure we deliberately suppressed. Nothing for you to fix: retry the build, escalate if it persists. |
| `none` | No failure details were recorded at all. |

Guidance falls back down that ladder, so `raw` only appears when no fix prompt,
remediation, or summary exists. When `source` is `withheld` the CLI also blanks
`errorSummary` and nulls `errorRemediation`, `errorFixPrompt`, and `errorRaw` on
that item, so there is nothing to dig into. Otherwise the envelope fields are
left untouched alongside `guidance`.

### 5. Run tests

Execute a user story on either native lanes or the web lane. For native runs,
provide at least one of `--ios-build` or `--android-build`; single-platform apps
may omit the other. For web runs, pass `--web` by itself — do **not** combine it
with native build flags. Web runs use the app's configured web URL and default web
targets; there are no per-run `--web-url`, `--browser`, or `--viewport` overrides
in the CLI.

```bash
# Run a single user story (by name or UUID) and wait for results
minitest --json --app <app_id> run start "User Login" \
  --ios-build <ios_build_id> \
  --android-build <android_build_id>

# Android-only app
minitest --json --app <app_id> run start "User Login" \
  --android-build <android_build_id>

# Web app (no build upload needed)
minitest --json --app <app_id> run start "User Login" --web

# Fire-and-forget (returns runId immediately — useful in CI)
minitest --json --app <app_id> run start "User Login" \
  --ios-build <ios_build_id> \
  --android-build <android_build_id> \
  --no-watch

# Run ALL user stories at once (creates one batch, fire-and-forget)
minitest --json --app <app_id> run all \
  --ios-build <ios_build_id> \
  --android-build <android_build_id>

# Run ALL user stories on web targets only
minitest --json --app <app_id> run all --web

# Build a required commit SHA and run its suite, polling by default
minitest --json --app <app_id> run from-commit <full_sha> \
  [--platform ios|android|web] [--platform ...] \
  [--user-story <id-or-name>] [--no-watch] [--timeout <seconds>]

# Cancel a running or pending run
minitest --json --app <app_id> run cancel <run_id>
```

Under the hood, `run start` and `run all` create a **batch**. A single run is
just a batch with one user story. With `--json --no-watch`, `run start` emits
only `{"runId": …, "status": …}`.

`--web` requires a web execution target configured on the app in the Minitest
web app; without one the run is refused even though the app was created with
`--platform web --web-url …`.

`run from-commit` requires a full SHA and polls the batch until a verdict unless
`--no-watch` is passed. Its initial response can legitimately contain no story
runs while the commit build is being prepared. For a web-only app, explicitly
pass `--platform web`: omitting platforms defaults server-side to iOS and
Android. Failed web builds currently may have no error envelope or fix prompt.

### 6. Check results

```bash
# Check a specific run
minitest --json --app <app_id> run status <run_id>

# Poll until completion
minitest --json --app <app_id> run status <run_id> --watch

# List all runs for a user story
minitest --json --app <app_id> run list "User Login"
minitest --json --app <app_id> run list "User Login" --status failed
minitest --json --app <app_id> run list "User Login" --all
```

**Run statuses:** `pending` → `running` → `completed` | `failed` | `cancelled`

A completed run includes per-target results: pass/fail for each acceptance
criterion, fail reasons, recording URLs, and stable `resultId`, `criterionId`,
and `criterionVersionId` identifiers for each criterion result.

### 7. Work with batches

A batch groups runs triggered together (by `run all`, CI, or a single
`run start`). Use the `batch` group to inspect or cancel them.

```bash
minitest --json --app <app_id> batch list                      # recent batches
minitest --json --app <app_id> batch list --status running
minitest --json --app <app_id> batch list --commit-sha abc1234
minitest --json --app <app_id> batch list --user-story <id>
minitest --json --app <app_id> batch get <batch_id>            # batch + its runs
minitest --json --app <app_id> batch cancel <batch_id>         # cancels all pending/running runs
```

**Batch statuses:** `pending` | `awaiting_build` | `running` | `completed` | `failed` | `cancelled`

### 8. Read a whole batch’s verdicts in one call

`run verdicts` projects a batch into a compact, product-level pass/fail
structure across all three grains — per-platform target roll-up, story ×
platform outcome (with `skipReason`), and per-criterion results (`status`,
`criticality`, `failReason`, `resultSummary`). Each story carries both
`userStoryId` (stable scenario identity, safe to key issues on) and
`userStoryName`. Use it instead of calling `batch get` + `run status` per story.

```bash
minitest --json --app <app_id> run verdicts <batch_id> [--platform ios|android|web] [--only-failed] [--actionable] [--verbose]
```

By default only failing criteria are listed per story, evidence is omitted, and
passing criteria appear only in the counters. Skipped stories carry `skipReason`
and targets carry `skippedByCascade`, so a dependency-skipped story is not a
failure.

**`--actionable` is the flag to use for triage.** `--only-failed` filters on
status alone, so cascade skips and passing-but-unprocessable leaves ride along —
often nearly half the rows, which you then have to read and discard.
`--actionable` keeps only criteria whose `status` is `failed` or
`unprocessable` **and** whose `criticality` is `critical` or `warning`, and
drops stories left with nothing.

**`--verbose` is the replay/debug mode.** It adds evidence, passing criteria,
and the artefacts needed to reconstruct a run — `buildId`, `recordingPath`,
`sessionPaths`, and per-criterion `confidence`. Those are omitted by default:
they are not triage signal, and `confidence` in particular must not be used as
a proxy for classifying a failure. Reach for `--verbose` only once you have
picked a specific failure to investigate.

Submit feedback on a criterion result by its `resultId`. This records a verdict
judgment such as "not a bug" or expected behavior. It does not mark an app issue
as fixed:

```bash
minitest --json --app <app_id> run feedback <result_id> "Not a bug: expected behavior for this account"
```

### 9. Read open issues and fix prompts

`issues list` returns the findings to fix as JSON, including each issue's
`fixPrompt` and exact webapp `deeplink`. It always emits JSON, regardless of the
global `--json` flag.

```bash
minitest --app <app_id> issues list
minitest --app <app_id> issues list --issue <failure_id>
minitest --app <app_id> issues list --run <story_run_id>
minitest --app <app_id> issues list --batch <batch_id>
minitest --app <app_id> issues list [--platform ios|android|web] [--criticality critical|warning] [--include-resolved]
```

Choose at most one of `--issue`, `--run`, and `--batch`. With no scope flag,
the command uses the latest batch. Its `scope` block then reports
`otherBatchesWithOpenIssues` and `openIssuesInOtherBatches`, so an agent can
offer to inspect older batches without prompting interactively.

The CLI never widens scope or asks an interactive question. If either widen
counter is non-zero, report the current result and the option to inspect older
batches without blocking for input. Use `batch list` to obtain batch IDs, then
call `issues list --batch <batch_id>` only when wider scope is requested.

The response has three blocks:

- `scope`: selected scope and filters, plus the non-interactive widen counters.
- `build`: attached provenance and failed-build details.
- `issues`: findings with per-issue `fixPrompt` and exact `deeplink`.

There is no top-level count field. Count the findings with `jq '.issues | length'`.
An empty `issues` array with exit code 0 means nothing to fix; it is not an error.

Build provenance uses only attached metadata: commit SHA first, then app version
and build number, then `no build info attached`. It does not inspect repository
history or compute commits ahead. Failed code builds may carry a build-level fix
prompt. Fix prompts from infrastructure failures are withheld.

Mark one or more app findings as fixed with their failure IDs:

```bash
minitest --app <app_id> issues fix <failure_id> [<failure_id> ...]
```

`issues fix` always returns JSON with one result per ID and continues after an
individual failure. Use it only after the app defect has been fixed. This is
different from `run feedback <result_id> "Not a bug: ..."`, which records that a
criterion result was expected behavior and does not close a fixed app defect.

The payload is `{"results": [...], "fixed": N, "failed": N}`. `results` carries
one entry per ID in the order given: `{"issueId": "...", "status": "fixed"}` when
the finding was closed, `{"issueId": "...", "status": "failed", "error": "..."}`
when it was not. The JSON is always printed, including on a non-zero exit, so
parse it rather than reading the exit code alone.

Exit codes:

| Situation | Exit |
|-----------|------|
| Every ID closed | 0 |
| At least one closed and at least one failed | 1 |
| Every ID failed, all not found | 4 |
| Every ID failed, all unauthorized | 2 |
| Every ID failed, at least one network error | 3 |
| Every ID failed, any other mix (malformed ID, feedback still processing, server error) | 1 |

A partial failure always exits 1, even when every failed ID was a 404, so a
stale ID mixed into a batch of good ones turns the whole call non-zero. Check
`failed` and the per-ID `error` strings to tell a stale ID apart from a real
problem, and retry only the IDs whose `status` is `failed`.

Webapp links are input hints for the agent, not CLI arguments. Extract IDs from
the known routes, set `--app` from `app_id`, and pass only IDs to the CLI:

```text
/t/{tenant}/apps/{app_id}/test/runs/{batch_id}?flow={user_story_id}
/t/{tenant}/apps/{app_id}/test/issues?issueId={failure_id}
```

Despite the path segment, `/test/runs/{batch_id}` carries a **batch id**, not a
story-run id. Route it to `--batch` directly; passing it to `issues list --run`
fails with a plain "StoryRun not found" that gives no hint of the mistake.

From a run URL, use `batch_id` with `run verdicts` or `issues list --batch`, and
use the optional `flow` value as the user story ID when needed. From an issue
URL, use `issueId` with `issues list --issue` or `issues fix`. The CLI itself
never parses webapp URLs.

## CI / Automation Pattern

```bash
# Native CI: upload builds, run all user stories, collect results
export MINITEST_APP_ID="<app_id>"

minitest --json build upload ./app.apk
minitest --json build upload ./MyApp.ipa

IOS_BUILD=$(minitest --json build list --platform ios --page-size 1 | jq -r '.items[0].id')
ANDROID_BUILD=$(minitest --json build list --platform android --page-size 1 | jq -r '.items[0].id')

minitest --json run all \
  --ios-build "$IOS_BUILD" \
  --android-build "$ANDROID_BUILD"

# Web CI: no build upload; use configured web targets
minitest --json run all --web
```

## JSON Output

JSON goes to stdout (camelCase keys, matching the backend API), diagnostics go
to stderr. Safe to pipe:

```bash
minitest --json --app $APP user-story list | jq '.items[].name'
minitest --json --app $APP run status <run_id> | jq '.status'
minitest --json --app $APP batch list | jq '.items[] | {id, status}'
```

Two shapes to expect: **list endpoints are paginated envelopes**
(`{items, page, pageSize, total}`) — `user-story`, `build`, `run`, `batch`,
`test-profile`, `test-file` — while `apps list` and `flow-types list` return
**bare arrays**. Reach for `.items[]` first and fall back to `.[]`.

`batch list` returns `storyRuns: []` for every batch; only `batch get` populates
the runs. Use `run verdicts <batch_id>` when you actually want the outcomes.

`auth status` is the one command that answers in snake_case
(`token_configured`, `method`, `user_id`).

## Quick Reference

| Task                | Command                                                                           |
| ------------------- | --------------------------------------------------------------------------------- |
| Onboard a new app   | `minitest init` (prints the end-to-end onboarding playbook)                       |
| Maintain tests locally | `minitest maintenance --agent` (prints the CLI maintenance playbook)           |
| Open maintenance run | `minitest --json maintenance context`                                            |
| Post maintenance change | `minitest maintenance change --file change.json`                              |
| Complete maintenance run | `minitest maintenance complete --changed`                                    |
| Apply maintenance edits | `minitest maintenance apply` or `minitest maintenance apply --review`          |
| List apps           | `minitest --json apps list`                                                              |
| App dependency graph| `minitest apps dependencies <app_id>` (Mermaid flowchart to stdout)               |
| Simulate dependency change | `minitest --json --app ID apps dependencies <id> --simulate --add <story>:<parent>` |
| Create native app   | `minitest --json apps create --name "My App" --platform ios --platform android [--tenant ID] [--description ...] [--slug ...] [--icon ./icon.png]` |
| Create web app      | `minitest --json apps create --name "My Web App" --platform web --web-url https://example.com [--tenant ID]` |
| Create user story   | `minitest --json --app ID user-story create --name "..." --type login --criteria "..."` |
| Create user story with profile | `minitest --json --app ID user-story create --name "..." --type login --profile <profile_id> --criteria "..."` |
| List user stories   | `minitest --json --app ID user-story list`                                               |
| Update user story   | `minitest --json --app ID user-story update <id> --add-criteria "..."`                   |
| Reword one criterion (keep history) | `minitest --json --app ID user-story update <id> --set-criterion <crit_id>="text"` |
| Revert criterion to a version | `minitest --json --app ID user-story update <id> --revert-criterion <crit_id>=<version_id>` |
| Set story dependencies | `minitest --json --app ID user-story update <id> --depends-on <parent_id> [--depends-on <parent_id2>]` |
| Remove a dependency | `minitest --json --app ID user-story update <id> --remove-dependency <parent_id>`        |
| Set story device count | `minitest --json --app ID user-story update <id> --device-count 2` (or `auto` to reset) |
| Set story camera media | `minitest --json --app ID user-story update <id> --camera-media <path-or-file-id>` (video ≤ 50 MB / image ≤ 25 MB) |
| Clear story camera media | `minitest --json --app ID user-story update <id> --clear-camera-media` (back to default feed) |
| List flow types     | `minitest --json flow-types list` (built-ins + your tenant's custom types)                |
| Create custom flow type | `minitest --json flow-types create --name "Subscription" [--usage-prompt "..."] [--icon tag] [--color gray]` |
| Rename custom flow type | `minitest --json flow-types update "Subscription" --name "Billing"`                  |
| Delete custom flow type | `minitest --json flow-types delete "Billing" --yes` (its stories fall back to `other`) |
| List mapped screens | `minitest --json --app ID screens list [--platform ios]`                                 |
| See the crawl's shape | `minitest --app ID screens list --tree`                                                |
| Screens the crawl was blocked on | `minitest --json --app ID screens list --blocked`                           |
| Inspect one screen  | `minitest --json --app ID screens get "Onboarding age"`                                  |
| Read app knowledge  | `minitest --json app-knowledge get --app ID`                                             |
| Update app knowledge| `minitest --json app-knowledge update --app ID --content-file ./knowledge.md`            |
| List env vars       | `minitest --json --app ID env list` (values masked; `--show` reveals)                    |
| Reveal one env var  | `minitest --app ID env get <KEY>` (prints the value verbatim to stdout)           |
| Set an env var      | `minitest --json --app ID env set <KEY> <VALUE> --yes [--dry-run]`                       |
| Remove an env var   | `minitest --json --app ID env unset <KEY> --yes [--dry-run]`                             |
| Clear all env vars  | `minitest --json --app ID env clear --yes [--dry-run]`                                   |
| Upload native build | `minitest --json --app ID build upload ./app.apk`                                        |
| List builds         | `minitest --json --app ID build list [--platform P] [--status S] [--kind K]`            |
| Build from GitHub commit | `minitest --json --app ID build from-commit [SHA] [--platform P] [--force-full]`   |
| Run one native story| `minitest --json --app ID run start "Story Name" --ios-build X --android-build Y`        |
| Run one web story   | `minitest --json --app ID run start "Story Name" --web`                                 |
| Run all native stories | `minitest --json --app ID run all --ios-build X --android-build Y`                    |
| Run all web stories | `minitest --json --app ID run all --web`                                                 |
| Build and run commit | `minitest --json --app ID run from-commit SHA [--platform P] [--user-story ID] [--no-watch] [--timeout N]` |
| Cancel a run        | `minitest --json --app ID run cancel <run_id>`                                           |
| Check run           | `minitest --json --app ID run status <run_id>`                                           |
| List runs for story | `minitest --json --app ID run list "Story Name"`                                         |
| List batches        | `minitest --json --app ID batch list`                                                    |
| Get batch + runs    | `minitest --json --app ID batch get <batch_id>`                                          |
| Batch verdicts (one call) | `minitest --json --app ID run verdicts <batch_id> [--platform P] [--only-failed] [--actionable] [--verbose]` |
| List issues and fix prompts | `minitest --app ID issues list [--issue ID \| --run ID \| --batch ID] [--platform P] [--criticality C] [--include-resolved]` |
| Mark app issues fixed | `minitest --app ID issues fix <failure_id> [<failure_id> ...]`                          |
| Submit result feedback | `minitest --json --app ID run feedback <result_id> "text"`                         |
| Cancel batch        | `minitest --json --app ID batch cancel <batch_id>`                                       |
| Auth                | `minitest auth login`, `minitest --json auth status`, `minitest auth logout`      |
| Refresh this skill  | `minitest skill` (prints the latest skill markdown)                               |
| Upgrade the CLI     | `minitest upgrade`                                                                |
| Mint API key        | `minitest auth api-key mint --tenant <id> --name <label> --json` (OAuth only)            |
| List API keys       | `minitest auth api-key list --tenant <id> --json`                                        |
| Revoke API key      | `minitest auth api-key revoke --tenant <id> --key <id> --json`                           |
| List test profiles  | `minitest --json --app ID test-profile list`                                             |
| Get test profile    | `minitest --json --app ID test-profile get <profile_id>`                                 |
| List shared profiles| `minitest --json test-profile list-shared` (Minitap-provided pool; currently Google account only) |
| Create test profile | `minitest --json --app ID test-profile create --name "..." --username "..." --password-stdin` |
| Set default profile | `minitest --json --app ID test-profile set-default <profile_id>` |
| Clear default profile | `minitest --json --app ID test-profile clear-default` |
| Update test profile | `minitest --json --app ID test-profile update <id> [--name ...] [--clear-password]`      |
| Delete test profile | `minitest --json --app ID test-profile delete <id> --force`                              |
| List test files     | `minitest --json --app ID test-file list [--kind image\|document\|video\|audio\|other]`  |
| Upload test file    | `minitest --json --app ID test-file upload ./local/file.pdf --note "..."`                |
| Get test file       | `minitest --json --app ID test-file get <id>` (returns short-lived download URL)         |
| Update test file    | `minitest --json --app ID test-file update <id> [--name ...] [--clear-note]`             |
| Delete test file    | `minitest --json --app ID test-file delete <id> --force`                                 |
| Bind profile to story | `minitest --json --app ID user-story-binding set-profile <story_id> --profile <id>`    |
| Clear story profile | `minitest --json --app ID user-story-binding set-profile <story_id> --clear`             |
| Bind files to story | `minitest --json --app ID user-story-binding set-files <story_id> --file <id> --file <id>` |
| List story files    | `minitest --json --app ID user-story-binding list-files <story_id>`                      |
| List draft features | `minitest --json --app ID df list [--status open] [--status merged]`                     |
| Open a draft feature | `minitest --json --app ID df create --title "..." [--description "..."]`                |
| See what a branch changes | `minitest --json --app ID df show <id> --view diff`                                |
| See the suite a branch would run | `minitest --json --app ID df show <id> --view effective`                    |
| Write to a branch   | `minitest --json --app ID df apply <id> --changeset ./changeset.json`                    |
| Abandon a branch    | `minitest --json --app ID df delete <id> --force`                                        |

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

## Draft features (`minitest df`)

A draft feature is a **branch of the app's test suite**: the tests a product
change will need, written before that change ships. It holds a *delta* against
main — new stories, edited ones, deletions, dependency changes — never a copy of
main. Everything you do to a branch goes through one atomic call, so a branch is
never half-written.

```bash
minitest --json --app $APP df list --status open
minitest --json --app $APP df create --title "Guest checkout" \
  --description "Behind the CHECKOUT_GUEST flag; needs a fresh cart per run."
minitest --json --app $APP df show $DF --view diff        # what it changes
minitest --json --app $APP df show $DF --view effective   # the suite it would run
minitest --json --app $APP df apply $DF --changeset ./changeset.json
minitest --json --app $APP df delete $DF --force          # abandon, keeps history
```

`show --view diff` is the branch as a list of operations, in the same vocabulary
`apply` accepts — read a branch, reason about it, write the correction back
without translating. `--view effective` is the resolved suite: main with the
branch folded in, which is what a run against this branch would execute.

### Writing to a branch

`apply` takes a JSON file containing the whole request body:

```json
{
  "expectedMainRev": 7,
  "idempotencyKey": "add-guest-checkout-1",
  "ops": [
    { "op": "story.create", "tmpId": "guest", "fields": { "name": "Guest checkout", "type": "checkout" }, "criteria": ["Order completes without an account"] },
    { "op": "criterion.edit", "criterionId": "…", "expectedVersion": "…", "content": "…" },
    { "op": "dep.add", "childRef": "guest", "parentRef": "…" }
  ]
}
```

The eight operations are `story.create` / `story.edit` / `story.delete`,
`criterion.create` / `criterion.edit` / `criterion.delete`, and `dep.add` /
`dep.remove`. Three things to know:

- **Name main, not the branch.** Ops refer to the ids you see on the main suite.
  The server copies a story into the branch the first time you touch it; you
  never handle the copy's id. `tmpId` binds ops to each other inside one batch
  (a `dep.add` pointing at a story the same batch creates), and it is also how
  `show --view diff` names every story the branch invented — which is what makes
  that output replayable onto another branch as-is.
- **Unknown keys are rejected.** A misspelt field is a 422, not a silent drop, so
  a changeset you hand-edited fails loudly instead of half-applying.
- **Send the tokens you read.** `expectedMainRev` comes from `df show --view
  diff`; if main moved since that read, the apply is refused with 409 instead of
  landing on a suite you did not look at. Per-entity `expectedVersion` works the
  same way. Both are optional, and omitting them means writing blind.
- **Retries are free.** Reusing an `idempotencyKey` returns the first response
  rather than applying twice.

Semantics worth stating: `story.edit` fields you omit are left alone, whereas an
explicit `null` clears them. A batch is all-or-nothing, and a batch whose
dependency edges would close a cycle is rejected whole.

### Branch state

Two independent axes. `status` is `open` → `merging` → `merged`, or `abandoned`.
`rebaseState` is `in_sync` / `pending` / `rebasing` / `conflicts` and says
whether the branch has caught up with main. A branch is runnable only when it is
`open` **and** `in_sync`. `apply` is refused while `rebasing` (the branch is
being recomputed) and accepted while `conflicts` — resolving a conflict *is* an
apply. `df delete` abandons a branch without erasing it: the rows stay for audit
and drop out of `df list` unless you ask for them with `--status abandoned`.

## Environment variables

Apps can carry a set of environment variables (`KEY=value` pairs) that the test
runtime injects. They are app-scoped and stored encrypted server-side. Manage
them with the `env` group; every command needs an app (`--app ID` or
`MINITEST_APP_ID`).

```bash
minitest --json --app $APP env list            # values masked as ******** by default
minitest --json --app $APP env list --show     # reveal every value
minitest --app $APP env get API_TOKEN   # print ONE value verbatim to stdout
```

`env list` masks values so a screen-share or log never leaks a secret. `env get`
is the deliberate single-value reveal — it prints the raw value (and nothing
else) to stdout, so `VALUE=$(minitest --app $APP env get API_TOKEN)` is safe to
script. This is the one place to drop `--json`: with it you get
`{"API_TOKEN": "…"}` and need `| jq -r '.API_TOKEN'`.

Writes are **read-merge-write**: `set`/`unset` fetch the current set, apply your
change, and send the full map back, so they never clobber the other keys. Every
mutating command (`set`, `unset`, `clear`) requires explicit `--yes`/`-y` — it
never prompts. Without `--yes` it refuses and exits non-zero, which keeps an
agent or CI job from mutating secrets by accident.

```bash
minitest --json --app $APP env set API_TOKEN abc123 --yes
minitest --json --app $APP env set API_TOKEN abc123 --dry-run   # show the diff, change nothing
minitest --json --app $APP env unset OLD_KEY --yes
minitest --json --app $APP env clear --yes                      # deletes ALL env vars for the app
```

Use `--dry-run` first when unsure: it prints the added/changed/removed keys
(`+`/`~`/`-`) without touching the backend. `unset` a missing key and `clear`
with nothing configured both exit `4` (not found).

### Passwords on the CLI

`--password` accepts an inline value but is logged by your shell history. Prefer
`--password-stdin` and pipe the secret in:

```bash
printf "%s" "$MY_PASSWORD" | `minitest --json --app $APP test-profile create \
  --name "Customer A" --username alice --password-stdin
```

The two flags are mutually exclusive. To wipe an existing password, use
`update --clear-password`.
