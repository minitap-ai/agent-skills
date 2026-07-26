<!-- rendered by Minitap — do not edit -->
---
name: test-suite-design
description: >
  Design a coherent, dependency-aware Minitest suite (scenarios with
  natural-language acceptance criteria, personas, and dependency edges) from a
  frontend or mobile app codebase, then apply it through the minitest CLI. Use
  when asked to "design a test suite", "generate test scenarios from this
  codebase", "map features to account tiers for testing", or "produce
  persona-based coverage for this app".
---

# Test Suite Design

Turn an app codebase into a **Minitest suite**: scenarios with visually
verifiable acceptance criteria, a **persona** (the account expected connected at
scenario start), and **dependency edges**. You reach a reviewed `suite.yaml` in a
temp working dir through a disciplined multi-wave analysis, then apply it with the
existing `minitest` CLI.

Read [suite-schemas.md](suite-schemas.md) for the local `suite.yaml` schema and
step-writing rules, and [minitest-target.md](minitest-target.md) for what the
Minitest executor (Mini) can and cannot run. Read both now.

## Iron rules (read first)

1. **You (the main agent) never open a source file.** All investigation is
   delegated to subagents (see "Subagents" below). If you feel the urge to read
   code, a wave's dispatch prompt is missing a question — fix it and re-dispatch.
   Your context holds only the artifacts.
2. **Each wave produces exactly one artifact.** Subagents return facts with
   `file:line` evidence; you merge and design. Subagents never design scenarios.
3. **Waves run in order.** Within a wave, dispatch subagents in parallel.
4. Journeys are derived from **what a user can reach and tap**, never from code
   module structure.
5. **Checkpoints are questions to the user.** Where a wave says "ask the user",
   stop and ask — don't assume.

## Subagents

Every wave except Wave 4 delegates read-only investigation to subagents. Use
whichever mechanism your host exposes, in this order of preference:

- **opencode**: dispatch the `explore` subagent, all subagents of a wave in one
  message so they run in parallel.
- **Claude Code**: dispatch via the `Task` tool (one call per subagent, batched
  in a single message for parallelism).
- **No subagent mechanism**: fall back to running each dispatch **sequentially
  yourself** as a scoped investigation, writing each result into the wave's
  artifact before starting the next. The iron rule still holds in spirit — treat
  each investigation as isolated and record only its facts.

Every dispatch carries four parts: **scope** (where to look / not look),
**questions** (closed-ended, exhaustive), **output schema** (fixed, bounded,
`file:line` on every claim), and **the don't** (no scenario design, facts only).
Embed the relevant recon pointers in each later dispatch so subagents never
rediscover the layout.

## Working directory

Create a **temp working directory** for artifacts — e.g. `mktemp -d` or an OS
temp path — NOT a `.test-suite-design/` folder inside the customer's repo. The
repo stays untouched; the suite is applied through the CLI, not committed as
files. Set `WORK=<temp dir>` and `REPO=<app path>`.

| File | Artifact | Produced by |
|---|---|---|
| `00-recon.md` | Pointer map (stack, routing, auth, flags, strings, analytics) | Wave 0 |
| `01-app-map.md` | Navigation graph + candidate journeys | Wave 1 |
| `02-capability-matrix.md` | Feature × account-variant availability | Wave 2 |
| `03-personas.yaml` | Minimal persona catalog | Wave 2 (derived by you) |
| `04-state-model.md` | Per-journey consumes/produces/destroys + auth trajectory | Wave 3 |
| `suite.yaml` | The reviewed design, applied in Wave 6 | Wave 4, patched by Wave 5 |

## Pre-wave — establish context (ask the user)

Before Wave 0, ask the user three things and record the answers:

1. **Context.** What is this app, who uses it, and what matters most to test?
   Anything a codebase read won't reveal (backend states you can provision, real
   accounts you own, known-flaky areas).
2. **Sources of truth.** Where should ground truth come from beyond the code — a
   design doc, an analytics event catalog, an existing test plan, a product
   spec? Point subagents at these as authoritative.
3. **Scope.** Design for the **whole app**, or a **sub-part** (one tab, one
   flow, one feature area)? Scope narrows every later wave — confirm the boundary
   before spending analysis on surfaces the user doesn't care about.

## Wave 0 — Recon

One subagent, breadth only. It locates *where things live*: framework, routing,
auth implementation, feature-flag/remote-config surfaces, entitlement/role
checks, i18n/strings, analytics events, build variants. It answers "where to
dig", not "how it works". Merge into `00-recon.md`; every later dispatch embeds
the relevant pointers.

## Wave 1 — Surface mapping

Shard the app by top-level navigation area (one subagent per tab / drawer section
/ auth-gated zone, enumerated from the recon pointer map, bounded by the chosen
scope). Each returns: screens, entry paths (tab, button, deep link, push,
onboarding), and **candidate journeys** at UX level.

Ground truth to prioritize over code structure: the wired route/navigation graph
(excludes dead code), analytics events, onboarding/deep-link/push entry points,
i18n/strings, plus any sources of truth the user named.

Merge into `01-app-map.md`, dedup journeys across shards. Keep the ~15–40
journeys a real user plausibly performs; drop admin/debug surfaces unless asked.

## Wave 2 — Gating hunt (the trap wave)

A dedicated wave — do **not** treat flags/tiers as incidental Wave 1 findings.
Parallel subagents hunt what differentiates one account from another: feature
flags (static, build variants, and **remote config** — whose runtime value is
unknowable from code); entitlements/tiers (paywalls, plan checks, role guards,
server-driven `capabilities`); other differentiators (locale/region, verification
state, empty-vs-populated data, A/B assignments).

Merge into `02-capability-matrix.md`: each Wave 1 journey annotated
`visible / hidden / paywalled / degraded / unknown(remote)` per account variant,
with evidence. Then **you** derive `03-personas.yaml`, applying the account-gating
and persona doctrine below — the minimal basis set, never the cross-product.

**What separates one account from another is the highest-value signal in the app — and the easiest to miss.** Two users on the same screen can see different things: a Pro badge, an extra tab, a paywall, a region-only feature, an experiment variant. Hunt these differentiators on purpose while you explore; don't treat them as incidental. Each one you find is a persona worth testing and a flow worth covering; each one you miss is a whole surface that never gets tested.

Look for every axis that gates what a user can reach or do:

- **Tiers and entitlements** — free vs paid plans, subscription levels, `capabilities`/`permissions` on the user, paywalls (hard = blocks the feature, soft = dismissible upsell).
- **Roles and permission guards** — route or screen guards keyed on a role (Driver vs Diner, Admin vs Member), features hidden behind a permission.
- **Feature flags and remote config** — static flags in the build and server-driven ones (LaunchDarkly, Firebase Remote Config, a homegrown `/config` call). A remote flag's real value is unknowable from the code alone: you can see the flag exists, not what it's set to. Model the variant most users see, record the assumption in app knowledge, and never write a criterion that silently assumes one side of a flag you can't read.
- **Locale, region, and verification state** — country/language gates, KYC/age/verification walls, features that only appear once an account is verified.
- **Data state** — empty vs populated (a fresh account with no orders vs a returning one with history) often renders entirely different screens.

**Cover each differentiated capability once, with the cheapest persona that unlocks it — not the cross-product of every feature by every persona.** If a feature behaves the same for Free and Pro, test it once as the cheaper persona; spend a Pro-only scenario on what Pro actually changes. The goal is every capability covered, not every persona re-walking every flow.


A **test profile** (persona) represents one specific user identity and state. Stories bind profiles via a many-to-many relation: several profiles may be bound to one story, none is "primary", and the tester agent picks whichever fits each step of the journey.

**Every story is bound to at least one persona.** There is no such thing as an unbound story: if you create or update a story without binding a profile, the system automatically binds the app's **"New user" persona** — a system-managed, immutable profile that represents a genuine brand-new user. At runtime it gets a unique disposable `<random>@qa.minitap.ai` inbox with no pre-existing account or state; the tester registers fresh via OTP if the flow needs an identity, and proceeds anonymously where the app allows it. Use the "New user" persona deliberately for first-launch, guest-browsing, registration, and anonymous flows — do not create your own "anonymous" or "guest" profile, and do not treat sign-in as mandatory for it (many apps are usable anonymously).

**Proactively create every profile the app needs for thorough testing.** During discovery, identify all distinct user roles, subscription tiers, and permission levels. Each one that affects what the user sees or can do on screen deserves its own test profile. Common examples:

- Free vs. Premium/Pro users (different features, paywalls, limits)
- Different roles (Driver vs. Diner, Patient vs. Doctor, Admin vs. Member)
- Returning user with populated data (vs. the empty-state case, which is the system "New user" persona)

**Freemium and paywalls:** when the app has tiers, create one persona per tier and spell out the entitlements in each persona's `about` field (what this tier can and cannot do). Classify paywalls explicitly: a **hard paywall** blocks the feature entirely for a lower tier; a **soft paywall** shows an upsell that can be dismissed or bypassed. Record which is which (and how a free user gets past a soft wall) in the app knowledge, so the tester never guesses.

**Naming:** name profiles after the app's real personas — not generic labels. A food delivery app has "Driver" and "Diner", not "Standard user". A clinic app with a freemium model has "Patient", "Doctor", "Free User", "Pro User".

**Credentials:** generate a passwordless `@qa.minitap.ai` persona for each profile and leave the password blank. Use a descriptive prefix derived from the role plus the app name, e.g. for an app called "FoodDash": `minitest-free@qa.minitap.ai`, `minitest-driver@qa.minitap.ai`. Leaving the password blank makes the persona OTP-first: at runtime the tester registers/signs in with that `@qa.minitap.ai` address and reads the confirmation or one-time code straight from its inbox — no real backend account needed. **Never invent a password or use a non-`@qa.minitap.ai` domain for a generated persona** — a passwordless non-`qa` address is rejected at creation, and a non-readable inbox defeats OTP. Only set a username + password when the customer has a real account they own (any domain is then allowed). **For a persona that needs a specific account state (e.g. premium/pro), generate a `<something>@qa.minitap.ai` persona *with* an explicit password and ask the user to add that email + password combo to their backend** so it is linked to a user in that state — the `qa` address keeps the inbox readable for OTP while the password lets them pre-provision the account. The `about` field must explain who this persona is and what state they need to be in (e.g. "Pro subscriber with an active monthly plan. Has completed onboarding and has at least one saved item.").

**For Google sign-in, link the existing shared Minitap Google account pool (the "Shared by Minitap" group) — do not invent your own profile for it and do not bind credentials.** At runtime the tester leases one account from the pool and fetches its credentials and 2FA code on demand — these are never injected into the prompt, so static credentials would be wrong and unusable. A leased account may already carry data from a previous run, so the tester resets it to a clean starting state before the scenario.

**State must match the scenario.** A story must link a profile whose state actually matches its scenario. Do not bind a generic profile whose state contradicts the story (e.g. an active subscriber on a "lapsed subscriber" story). When no existing profile matches the required state, create a dedicated one and describe that state plainly in its `about` field.

**Personas vs. devices.** A persona is an *identity*; a device is a *surface*, and the two are decoupled — a scenario's device count is set independently of how many personas it binds (see "Device count" below). Bind the personas the journey needs, then decide the device count from *how* those identities are used:

- **Simultaneous identities → multiple devices.** When two identities must be live *at the same time* — real-time chat or a call between two users, a background push one account triggers for another, presence/typing indicators, or cross-account visibility (A blocks B, and B can no longer see A) — each concurrent identity needs its own device so the tester can observe both screens at once.
- **Sequential identities → one device.** When the journey uses identities one after another — post as A, then sign out and check as B — a single device is enough; the tester signs in and out. Binding two personas does **not** by itself require two devices — but because the auto default resolves to one device per bound persona, pin an explicit device count of 1 on these stories.
- **Same account twice → two devices, one persona.** A session-conflict test (the same account signed in on two devices) binds a single persona but runs on two devices.

**Coverage check:** before story creation, verify you have a profile for every distinct persona visible in the app. Missing a profile means missing entire feature surfaces.

**Default profile rule:** set a default profile only when a single persona is clearly the one most newly created stories should start with (typically the main signed-in account). If multiple personas are equally primary, leave default unset.


**Checkpoint (ask the user):** confirm the persona catalog and whether each
account can actually be provisioned, before Wave 3. This is the one checkpoint
worth a human answer.

## Wave 3 — State & auth-transition analysis

For each journey (batch ~5–8 per subagent), answer three questions: **Consumes**
(what must exist before — account-level or device-level?), **Produces/destroys**
(what persists after — reversible in-app or not?), and **Auth trajectory** (does
it sign out, switch, or create an account? default `output_account =
input_account`). Merge into `04-state-model.md`. Dependencies and account wiring
are now **derived facts**, not guesses.

## Wave 4 — Suite design (you alone, zero delegation)

Compose `suite.yaml` in `$WORK` (schema + step-writing rules in suite-schemas.md),
governed by the doctrine below.

**A user story = a full session, and it is self-contained.** Each story runs in a fresh session, from scratch, and owns its own path from app launch to the exact state it tests — signing in, onboarding, navigating, and setting up whatever cheap in-app state it needs. It never assumes a previous story left the device in a particular state; even a story with dependencies must be runnable on its own. Put that setup path in the **description** (as straightforwardly as possible, so the agent reaches the testable state fast); the acceptance criteria stay focused on WHAT to verify once there. Spinning up a clean environment, loading the app, signing in, and reaching the feature is the bulk of the cost — once the agent is there, it should exercise the feature exhaustively before the session is torn down.

The only state a story does **not** recreate itself is state that is expensive or irreversible to produce (an upgraded account, a placed order, moderated content). That comes from a real dependency (see the dependencies guidance). Identity is likewise not recreated: a story inherits the exact credentials of its chain. Everything cheap and reversible, the story does itself.

**Default to ONE story per feature area.** Pack everything reachable in a single uninterrupted session into the same story's acceptance criteria. The criteria list IS the test script; long criteria lists are GOOD, not a smell.

**Only create a separate story when starting from scratch is actually required.** Split a flow into multiple stories ONLY if one of these is true:

1. **App state must change between scenarios** in a way the agent cannot undo in the same session (e.g. "first-launch onboarding" vs. "returning user home screen" — the first needs a clean environment).
2. **A different profile/persona is needed** (e.g. "Driver accepts delivery" vs. "Diner places order" — different accounts, different permissions). **Exception — collaborative journeys:** when a single flow genuinely spans two accounts (sign in as A and send a message, sign out, sign in as B and confirm B received it), keep it as ONE story and bind BOTH personas to it. The tester drives both accounts sequentially on the one device, switching accounts as its own setup steps. Split only when the two roles run genuinely separate journeys, not when one journey needs both.
3. **A different entry path is required** that cannot be reached by in-app navigation (e.g. "deep link opens product page from external app").
4. **The feature is genuinely independent** and isolating it gives clearer failure signals (e.g. "Login" stays its own story even though every other story logs in — because login failure should not cascade into ten unrelated reds).

**If none of those apply, fold the scenarios into ONE story.** Examples of what to collapse:

- Five separate stories about viewing, updating, and managing a single resource (e.g. orders, bookings, listings) → ONE story covering the full lifecycle in a single session.
- "Add to Cart" + "View Cart" + "Update Quantity" + "Remove from Cart" → ONE story "Cart Management" with the four flows as sequential criteria.
- "Open Settings" + "Toggle Notifications" + "Change Language" + "View About" → ONE story "Settings Screen" walking through each toggle and screen.

**Test before splitting:** ask yourself "could a tester, already logged in and standing on the home screen, walk through both of these without resetting the environment, switching accounts, or restarting the app?" If yes — same story.

**Coverage:** cover every major user-facing flow. Login, registration, onboarding, the app's core value prop, search, profile, settings — if the app has it, there must be a story. Cover the happy path end-to-end AND the paths that can break: failure states, validation errors, permission/auth denials, empty states, and edge cases. Prefer fewer, well-defined stories over many vague ones.

**Feature flags:** when a flow's visible version depends on a rollout or feature flag, model the most common variant (the one most users see). If a major variant is gated to a small rollout, note it in the story description or app knowledge rather than authoring ambiguous criteria.


A dependency is **not** a way to put the device into a prior state. Every story is **self-contained**: it carries the device from a fresh launch to the exact state it tests, using its own setup path, then verifies its criteria. Any story — even a leaf deep in a chain — can run on its own without replaying its ancestors. So a dependency exists for exactly two reasons:

1. **Fail-fast.** When a parent story ends in a critical failure (or fails on infra), the batch engine **cascades a SKIPPED state** to every transitive dependent on the same platform. The user sees one root-cause red instead of a wall of twenty, and no device time is burned on runs that were doomed the moment the foundation died.
2. **Device reuse.** When the engine can run a chain on one already-warm device, it saves the cost and time of re-establishing shared state. This is an optimization the engine applies — never a correctness crutch the story leans on.

Declare these relationships up front — the cascade engine cannot infer them.

### Rule of thumb — when to declare a dependency

Two questions, in order:

> **1. "Would I learn anything useful from running B if its parent A just crashed?"**

- **No → the fail-fast case.** Running it is pure waste; declare the dependency so it cascade-SKIPs.
- **Yes → leave it independent.** B reaches its own state anyway, so it can genuinely test itself even if A failed.

> **2. "To run B, some state must already exist. Can B create that state itself — cheaply and safely — with its own setup (setup ≠ acceptance criteria)?"**

- **Yes → reducible.** Inline the setup into B's own description; declare **no** dependency for it.
- **No → irreducible.** The state is expensive or irreversible to produce (account upgraded, order placed, content moderated). Give it a dedicated story and declare a real dependency on that story.

The bar is **deliberately high**. Cheap in-app state (an item in the cart, a filter applied, a screen navigated to) is always reducible — the story does it itself. The feature's value comes from the few foundational flows (login, onboarding, subscription upgrade) and from genuinely irreversible produced state, not from modeling the whole app graph.

**Every non-gate edge names its state.** A dependency that exists for device-reuse (not just fail-fast) has to justify itself: name the exact state the parent *produces* and the child *consumes*, and say why that state is expensive or irreversible to re-create — that's what keeps it above the reducible bar. ✅ "Checkout depends on Place Order: Place Order leaves the account with a placed, non-cancellable order that Checkout's refund flow reads." ❌ "Checkout depends on Add to Cart" — a cart item is cheap in-app state the child sets up itself. **No such state, no edge:** if you can't name it, the dependency is reducible and shouldn't exist. Gate (sign-in/onboarding) edges are exempt — they exist purely for fail-fast, so "requires the persona's session" is reason enough.

**Keep chains shallow.** The dependency graph is a shallow DAG. Counting only non-gate edges (gate roots don't add depth), no chain should run more than **three** levels deep. A deeper chain is a smell: a story buried under that many prerequisites is almost always one that should have inlined its own cheap state instead.

### Isolate write scenarios

A scenario that **produces irreversible state** (places an order, upgrades an account, posts content, cancels a subscription) is a **write** scenario. Keep write scenarios apart:

- **One writer per irreversible resource.** Never let two scenarios mutate the same irreversible state on the same account in ways that can run concurrently — the winner is nondeterministic and both readings become flaky. If two flows both need to place an order, give each its own persona (its own chain) so they can never collide.
- **A write scenario is a chain leaf for anything that reads what it wrote.** If B verifies the result of A's write, B depends on A (irreducible state) and A's write is not re-run underneath B. Do not fan out multiple concurrent writers under one shared setup root.
- **Reads are free to parallelize; writes are not.** Read-only journeys on the same persona can run independently; write journeys on the same persona must be serialized by the dependency chain that owns that state.

### Concrete examples

- **`Login` is a parent of most other flows** — almost nothing else can reach the home screen if login crashes.
- **`Onboarding` is a parent of every flow that needs a fully-onboarded account.**
- **`Add to Cart` is a parent of `Checkout`** — there's nothing to check out if you couldn't add an item.
- **`Edit Profile` is *not* a dependency of `View Profile`** — viewing works whether or not editing did.
- **`Search` is *not* a dependency of `Settings`** — independent journeys that share an app shell.

### One persona per chain (hard rule)

Every story in a chain shares the SAME persona, and — this is the invariant — the SAME identity. Identity (which account is logged in) is **injected context, never re-created**: the first story in a chain to establish a persona pins its credentials, and every dependent is handed those exact credentials, on any device, warm or cold. A dependent must **never** register or mint a fresh identity for a persona a prior story in its chain already established. This matters most for the system "New user" persona, whose disposable `<random>@qa.minitap.ai` identity is minted at runtime: the chain persists that exact minted identity and reuses it, so a dependent running on a fresh parallel device signs in as the *same* human — not a new one — and irreversible state produced by the parent is really there.

Give each persona one **"Sign in as `<Persona>`" root story** bound to that persona's profile, then chain that persona's stories off it — every node, root included, carries that profile. Two personas must be two separate chains, each rooted in its own sign-in story. The backend rejects a cross-persona edge with `422 persona_mismatch`, so a chain wired wrong will fail to save. **Never change a story's persona just to make an edge legal** — the fix is always to point the story at its own persona's sign-in root.

- ✅ `Sign in as Diner (Diner) → Browse Restaurants (Diner) → Checkout (Diner)` — one persona, one sign-in root, same profile on every node.
- ✅ `Sign in as Driver (Driver) → Accept Delivery (Driver)` and a separate `Sign in as Diner (Diner) → Place Order (Diner)` — two personas, two independent chains.
- ❌ `Sign in as Driver (Driver) → Place Order (Diner)` — two accounts in one chain; Place Order belongs under the Diner sign-in root.
- ❌ Re-binding a child to a different persona purely to satisfy the edge — keep its identity, rewire the edge.

### Constraints (server-validated)

- **Same-app only.** A story can't depend on a story from another app.
- **No cycles.** Direct (A→A), transitive (A→B→A), or longer chains are all rejected.
- **References must exist.** Don't invent IDs; pull them from the story list.
- **Replace-all semantics.** `depends_on=[]` clears the set; `depends_on=null` (omit the field) leaves it untouched. Don't conflate.
- **Per-platform isolation.** Dependencies are resolved separately per platform — a failure on one target never skips another target's story. You don't declare per-platform; the engine handles it.


Every story has three parts: **title**, **description**, and **acceptance criteria**. All three are mandatory.

**Title** — starts with a verb. Action-oriented and user-facing (e.g. "Add Item to Cart", "Comment on Own Game", "Discover Restaurants and Manage Cards"). Never noun phrases like "Restaurant Discovery and Card Management".

**Description** — one or two sentences explaining what the user does and any important scope constraints. This tells the tester agent the intent of the story without reading all criteria. Example: "Logged-in user opens the comments drawer on one of their OWN published games and posts a comment. Do not comment on other users' games." When reaching the tested state is not obvious from the criteria alone, the description is also where you spell out that setup path (how to get there) — the criteria then stay focused on what to verify once there. Keep it a path, not choreography: name the state to reach, not every tap.

**Acceptance criteria** — each criterion is a **job to be done**, not a micro-step or a single interaction. Mini acts like a user, so it can figure out HOW to accomplish a goal up to a point — as long as it's given enough context grounded in how the app actually works. The criteria say WHAT to accomplish and WHAT the expected outcome looks like; the description carries the grounding — the feature/product-area anchor so Mini can orient itself, and the big UX path (a path, not choreography). No Given/When/Then. No exact interaction sequences. No embedded reasoning. Aim for 5-15 criteria per story.

Each criterion **must** be:

- **Visually verifiable** — the AI agent only sees the screen. Write assertions about what is visible: text, UI elements, navigation state.
  - Good: "A success toast or confirmation message is displayed"
  - Bad: "The backend creates a new user record in the database"
- **Specific and unambiguous** — avoid vague assertions.
  - Good: "The cart badge shows the updated item count"
  - Bad: "The cart works correctly"
- **Goal-scoped** — each criterion bundles a small goal and its verification. Do not split a single action and its result into two criteria.
  - Good: "Add an item to the cart and confirm the cart badge increments"
  - Bad (too granular): criterion 1 "Tap the Add to Cart button", criterion 2 "The cart badge shows 1"
- **Ordered** — list criteria in the order they should be observed during the user journey execution.

Each criterion **must NOT**:

- Prescribe exact interaction sequences, stepper increments, or input choreography. State the intent: "Add two items to the list" not "Open + Add, pick Item A, confirm, open + Add, pick Item B, confirm..."
- Include internal route paths (e.g. `/(auth)/sign-in`), code identifiers, component names, or implementation details. Reference only user-visible labels, screen names, and UI landmarks (e.g. "the home screen", "the settings page", "the checkout button").
- Hardcode verbatim marketing copy or long strings from the app. Reference content by purpose ("the onboarding slide explains the app's core feature") not by literal text.
- Require anything an agent on a real device or browser cannot check. No server-side checks, no DB inspection.

**Never inline credentials or profile-specific data into criteria.** A story's login/account details come from its linked test profile, which the customer can edit at any time. Hardcoding the email, password, or other profile values into a criterion duplicates that data and goes stale the moment the profile changes. Refer to the persona abstractly instead, e.g. "Sign in with the profile's credentials and confirm the home screen loads". The backend rejects any create request that contains a credential value (such as a password) in a user-facing field.

**Examples of bad vs. good criteria:**

Bad (micro-steps):

1. Tap "Next" — slide 2 appears with its title
2. The body text reads "Discover amazing features..."
3. The "Skip" button is still visible in the header
4. Tap "Next" — slide 3 appears with its title

Good (jobs to be done):

1. Advance through all onboarding slides and confirm the final slide leads to the sign-in screen

Bad (over-prescribed):

1. Tap the "+" button — the item picker modal opens
2. Type "blue" in the search field — only matching items appear
3. Clear the search field and tap the "Clothing" category filter
4. Tap "Blue Jacket" — the modal closes and the item appears in the list

Good (goal-oriented):

1. Add two items using the picker, verifying search and category filters work
2. Confirm both items appear in the list with their default details


Design rules on top of that doctrine:

- **Gates first.** For every persona that must be established (onboarding for
  fresh-install personas, sign-in for pre-existing accounts), write one
  `kind: gate` scenario from a fresh app launch that ends established as that
  persona. Every feature scenario depends on its persona's gate — a broken gate
  cascade-skips its dependents.
- **Coverage goal.** Every capability-matrix row covered at least once by the
  cheapest persona that unlocks it, plus every auth-transition journey — NOT the
  feature × persona cross-product.
- **Remote-flagged features.** Template the most common variant and record the
  flag in the scenario's `assumptions`.

## Wave 5 — Adversarial verification

Dispatch each scenario (batch a few per subagent) to a **fresh** subagent that
has NOT seen your design rationale, asking only: *"Given the code, can persona P
actually execute these steps and observe these criteria?"* It checks
reachability for that tier/flag set, criteria observability, and gating
mismatches. In parallel, run your own mechanical invariant pass over `suite.yaml`
(full list in suite-schemas.md), then the suite self-check below.

**Before the suite is final, audit it as if someone else built it.** You wrote these stories knowing what you intended, which is exactly why you'll miss where they don't hold up. Take one cold pass over the finished set and fix what it surfaces — a few quiet corrections here save the user from a suite that looks complete but doesn't run.

For each story, ask the question a stranger would: **"Given how this app actually works, can this persona reach these screens and observe every criterion?"** If a criterion checks something this tier can't see, a screen this role can't open, or a state the story never set up, correct the story — retarget the persona, inline the missing setup, or fix the criterion.

Then check the structure mechanically:

- Every chain is rooted in its persona's sign-in story, and every story in a chain shares that one persona (no cross-persona edges).
- No dependency cycles, and chains stay shallow — a story buried under many prerequisites usually just needs to set up its own cheap state instead.
- Every persona you created is used by a story, and every persona a story needs exists.
- Every distinct capability and account-differentiator you found is covered by at least one story.
- Every acceptance criterion is checkable on-screen — no backend, database, or network assertions.

Fix issues in place; don't redesign a working suite around one bad edge.


Apply findings as **patches** to `suite.yaml`, not redesigns. Loop once more only
if patches were substantial.

## Wave 6 — Apply through the minitest CLI

There is no `minitest apply`. You replay the reviewed `suite.yaml` by running the
existing CLI commands, in dependency order. Read `minitest <group> --help` when
unsure of a flag.

1. **Distill app knowledge.** Fold the recon + capability findings the tester
   needs at runtime (paywall hard/soft classification, remote-flag assumptions,
   provisioning notes) into an app-knowledge update:
   `minitest app-knowledge update <app id> --content "<distilled knowledge>"`.
   App-level assumptions live here; scenario-local assumptions go in that
   story's description.
2. **Create personas.** One `minitest test-profile create` per persona in
   `suite.yaml` (see minitest-target.md for the `@qa.minitap.ai` mechanics).
   Record each returned id.
3. **Create scenarios in dependency order** — parents before children. For each,
   fold its `steps` into the story **description** as the big UX path plus the
   feature/product-area anchor (minitest stories persist no steps field), pass
   the acceptance criteria, bind the persona, and wire prerequisites:
   `minitest user-story create --name "<name>" --type "<flow type>" --description "<UX path + anchor>" --criteria "<criterion>" [--criteria ...] --profile "<persona id>" [--depends-on "<parent story id>" ...]`
   Set `--device-count` only when minitest-target.md's device rules call for it.
4. **Verify the wiring.** `minitest apps dependencies` to review the graph; fix
   with `minitest user-story update` / `minitest user-story-binding set-profile`.

**Checkpoint (ask the user):** present the persona table, the scenario table
(id, name, persona, input→output account, depends_on), the dependency DAG, and
the coverage summary, and get approval **before** creating anything. After
applying, offer to replay the suite: hand off to the web app's "Run tests" flow,
or trigger a run through the CLI if the user wants one now.
