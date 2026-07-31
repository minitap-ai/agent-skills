---
name: minitest-capabilities
description: >-
  Canonical Minitest product capabilities and scenario-authoring rules. Use
  when creating, reviewing, or changing apps, scenarios, acceptance criteria,
  personas, dependencies, device counts, connectivity coverage, or app
  knowledge.
---

# Minitest Capabilities

Use these rules whenever reasoning about a Minitest suite. In the product UI,
a user story is called a **scenario** and a batch is called a **run**.

## What Mini can test

This envelope is the rendered `mini-capabilities` fragment from Minitap's
capability source of truth. Do not edit it by hand — refresh it with
`minitest capabilities` (add `--platform <ios|android|web>` to see only what a
given platform supports).

Mini is the Minitap testing agent. It drives real Android, iOS and web sessions and executes acceptance criteria. A criterion Mini cannot physically perform or observe will fail as unprocessable, not because the app is broken.

**What Mini can do:**

- **Gestures:** tap (by label, coordinates or screen percentage), long-press, swipe, drag (including pick-up drag with a hold before the move), pinch and zoom. *(Android, iOS, web)*
- **Curved gestures:** an arbitrary path in a single press-move-release — circles, arcs, loops, figure-eights. Rotary dials, knobs, circular unlocks, pattern locks, signature pads and arc sliders are all testable. *(Android, iOS, web)* **Android:** Paths anchor to an element's bounds. **iOS:** Paths anchor to an element's bounds.
- **Text:** type into the focused field or a targeted element, erase, press enter. *(Android, iOS, web)* **Android:** Can also navigate back and home.
- **Observation:** screenshots (standard and high-res), a compact or full UI hierarchy, finding elements by text or label, and visual questions. *(Android, iOS, web)*
- **App lifecycle:** launch, stop, open a URL or deep link, install a provided build. *(Android, iOS)*
- **Media & audio:** speak text aloud or play an audio file into the browser microphone, and transcribe what the browser's speakers output. *(web)*
- **Camera:** the browser camera is simulated — a per-story image or MP4 is fed as the live webcam feed, so QR scanning and document presentation are testable. *(web)*
- **Files:** push a file to the device for upload and attachment flows, seeded from story-bound test files. *(Android, iOS, web)*
- **Connectivity:** change the network state mid-run. *(Android, web)* **Android:** Wifi on/off, airplane mode, and reading the connectivity state. **web:** Take the page offline and back online.
- **Screen orientation:** rotate to landscape or portrait. *(Android, iOS)*
- **Device state:** grant or revoke permissions without the system prompt, and switch between light and dark appearance. *(Android, iOS, web)* **iOS:** Can also freeze the status bar to a fixed time and battery for stable screenshots.
- **Push notifications:** deliver an arbitrary payload to the app under test. *(iOS)*
- **Geolocation** (cloud devices): mock GPS, simulate movement along a route at a given speed, and restore the real location. *(Android, iOS)* **Android:** Play Services `GeofencingClient` transition callbacks may not fire (cloud devices run microG, not real GMS); apps reading location directly work fine.
- **Identity & email:** read any `<prefix>@qa.minitap.ai` inbox for OTP codes, verification links and magic links, and sign in with Google through the shared Minitap account pool (leased at runtime, 2FA handled). *(Android, iOS, web)*
- **Replay:** re-run a previously captured trajectory to reach a known state faster. *(Android, iOS)*
- **Multi-device:** up to `min(3, tenant device quota)` devices at once, set by the scenario's device-count setting. Auto resolves to one device per bound persona (minimum one), so a sequential multi-persona scenario pins an explicit count of 1. The count is decoupled from personas — two devices on one persona for session-conflict tests, or one device and two personas by signing in and out. This is what makes real-time cross-account behaviour assertable. Every device runs the same OS and the same app. *(Android, iOS)*

**What Mini cannot do — never write criteria that require:**

- Server-side, database, log or analytics verification — only what is visible on screen counts. *(Android, iOS, web)*
- Receive SMS or phone calls, or read email outside `@qa.minitap.ai` inboxes. *(Android, iOS, web)*
- Biometric auth (fingerprint, Face ID), hardware buttons beyond back and home, NFC, or Bluetooth pairing. *(Android, iOS)*
- Use camera input. *(Android, iOS)*
- Feed audio into the microphone or hear what the app plays — the cloud device provider exposes no audio path, so voice-driven flows are web-only. *(Android, iOS)*
- Toggle connectivity or airplane mode. *(iOS)*
- Rotate the viewport or mock a location — the stealth browser pins the window to its fingerprint and blocks the geolocation override, so pick the right viewport preset up front. *(web)*
- Pair with a smartwatch, wearable or other external hardware (a second phone or tablet IS supported). *(Android, iOS, web)*
- Enter a real payment card or make a real-money purchase (sandbox flows only). *(Android, iOS, web)*
- Guarantee precise timing or gesture velocity ("responds within 200ms", "flick fast enough to fling the list") — Mini observes order and outcomes, not millisecond latency, and gesture pacing is approximate. *(Android, iOS, web)* **Android:** Pacing is noticeably slower than requested.

Every criterion must stay within these observable capabilities.

## Scenario design

A scenario is one fresh, self-contained test session. It owns the path from app
launch to the state it tests, including sign-in, onboarding, navigation, and
cheap reversible setup. Put that setup path in the description and keep the
acceptance criteria focused on what to verify once there.

Default to one scenario per feature area. Once Mini has paid the cost to reach a
feature, cover the reachable behavior thoroughly in that session. Split only
when at least one of these is true:

1. The app needs a reset or irreversible state change between journeys.
2. A different persona runs a genuinely separate journey.
3. The journey requires an entry path unavailable through in-app navigation.
4. Isolation gives a meaningfully clearer failure signal for an independent
   capability.

Keep collaborative journeys that span accounts in one scenario when the
identities act within one flow. Bind all needed personas, then choose devices
based on whether those identities must be simultaneous.

Cover every major user-facing flow, including happy paths, validation and
failure states, permission or authentication denials, empty states, and
meaningful edge cases. Prefer fewer complete scenarios over many vague ones.
Model the product variant most users see when a feature flag creates multiple
visible variants, and record important rollout caveats in the scenario
description or app knowledge.

## Titles, descriptions, and acceptance criteria

Every scenario requires all three fields:

- **Title:** begin with a verb and describe a user-facing action.
- **Description:** use one or two sentences for intent, scope, and the concise
  setup path needed to reach the tested state.
- **Acceptance criteria:** ordered, visually verifiable jobs to be done. Aim for
  5–15 criteria per scenario.

Each criterion must be specific, unambiguous, goal-scoped, and observable on
the screen. Bundle an action with its expected visible result. State what to
accomplish and observe, not the exact tap sequence Mini should follow.

Do not use Given/When/Then, micro-steps, internal routes, code identifiers,
component names, implementation details, long hardcoded marketing copy, or
invisible backend assertions. Never put credentials or profile-specific values
in user-facing fields; refer to the bound persona's credentials abstractly.

## Scenario types

Every scenario has a type:

1. Prefer a built-in type when the primary purpose matches `login`,
   `registration`, `checkout`, `onboarding`, `search`, `settings`, `navigation`,
   `form`, or `profile`.
2. Otherwise query current custom types, reuse or create a generic reusable
   type, and set both `type: "custom"` and its real custom-type ID.
3. Do not use `other` for a recognizable product capability.
4. Never invent IDs, use custom without a custom-type ID, or attach a custom ID
   to a built-in type.

## Personas

A test profile, called a persona in the UI, represents one user identity and
state. Every scenario must bind at least one persona. If none is supplied, the
system binds the immutable **New user** persona.

Use New user for genuine first-launch, guest, registration, and anonymous
journeys. It receives a unique disposable `@qa.minitap.ai` inbox and may proceed
anonymously where the app allows it. Do not create duplicate guest personas or
assume it must sign in.

Create a distinct persona for every role, tier, permission level, or returning
state that changes visible behavior. Name personas after the app's real roles.
Describe required state and entitlements in the `about` field. For tiered apps,
identify hard paywalls that fully block a feature and soft paywalls that can be
dismissed or bypassed.

For generated identities, use a descriptive passwordless `@qa.minitap.ai`
address so Mini can complete OTP flows. Do not invent passwords or use an
unreadable domain. When a persona needs a pre-provisioned state such as a paid
subscription, use an explicit `@qa.minitap.ai` email and password and ask the
user to create that account in the required state. Use customer-owned real
credentials only when they provide them.

For Google sign-in, link the existing **Shared by Minitap** account pool. Do not
invent static Google credentials; Mini leases an account and obtains its 2FA
code at runtime.

Bind personas whose state matches the scenario. Create a dedicated persona if
no existing one does. Set a default persona only when one identity is clearly
the normal starting point for most new scenarios.

## Personas and device count

A persona is an identity; a device is a testing surface. They are independent.
An unset device count uses auto: one device per bound persona, with a minimum of
one. An explicit count overrides auto up to the lower of three and the tenant's
quota.

Use multiple devices only when identities must be live simultaneously, such as
real-time messaging, calls, presence, typing indicators, cross-account
visibility, or a notification one account triggers for another. Sequential
identity changes use one device and sign in and out; explicitly set the count to
one on multi-persona sequential scenarios because auto would allocate one per
persona. A same-account session-conflict scenario may bind one persona and use
two devices.

For ordinary single-persona scenarios, leave device count unset. Additional
devices change both cost and behavior and must never be used as speculative
coverage.

## Dependencies

Dependencies never replace self-contained setup. They exist only for:

1. **Fail-fast:** skip dependents when a foundational scenario critically fails
   and no useful signal could come from running them.
2. **Device reuse:** let the engine reuse a warm device as an optimization.

Before adding a dependency, ask:

1. Would running the child still produce useful information if the parent
   crashed? If yes, leave it independent.
2. Can the child cheaply and safely create its required state in its own setup?
   If yes, describe that setup and leave it independent. Use a dependency only
   for expensive or irreversible state.

Keep irreversible write scenarios isolated. Use one writer per irreversible
resource and serialize reads of produced state behind that writer. Independent
read-only journeys may run in parallel.

Every scenario in a dependency chain must use the same persona and exact
identity. Give each persona its own sign-in root and chain only that persona's
scenarios beneath it. Never change a persona just to make an edge legal.

Dependencies must stay within one app, contain no cycles, and reference real
scenario IDs. An empty dependency list clears all dependencies while an omitted
field leaves them unchanged. The engine resolves dependencies independently per
platform.

## Connectivity

Add offline coverage only when the app exposes offline behavior, caching, or
sync. On Android say **Offline (wifi off)** rather than airplane mode, since
that is the wording the tester recognizes. On web, word it simply as
**Offline**. iOS runs cannot change connectivity — skip network-toggle criteria
there.

## App knowledge

App knowledge is concise orientation injected into every test run. Keep it
under 1000 characters and include:

- the app's purpose, main screens, and navigation structure;
- key roles or personas;
- non-obvious interactions such as swipe-to-delete or long-press-to-edit;
- hard and soft paywalls and how a free user passes a soft wall; and
- major feature-flag or rollout caveats relevant to the modeled scenarios.

App knowledge is not a full specification. Include only information that helps
Mini navigate and test accurately.
