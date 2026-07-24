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

Mini drives real Android and iOS devices and browser sessions. A criterion Mini
cannot physically perform or observe will be unprocessable rather than proving
the app is broken.

Mini can:

- tap by label, coordinates, or percentage; long-press; swipe; drag; and
  pinch or zoom;
- type, erase, press enter, and navigate back or home on Android;
- inspect screenshots and UI hierarchies, find visible elements, and answer
  visual questions;
- launch and stop apps, open URLs or deep links, and install provided builds;
- play audio through the device microphone path and speak text for voice-input
  flows;
- inject microphone audio and transcribe browser speaker output on web runs;
- simulate a browser camera with a scenario-bound image or MP4 on web runs;
- push scenario-bound files to the device for upload and attachment flows;
- toggle wifi and read connectivity state on Android;
- read any `<prefix>@qa.minitap.ai` inbox for OTPs, verification links, and
  magic links;
- sign in with Google through Minitap's shared account pool, including 2FA;
- replay captured trajectories to reach known states faster; and
- drive several devices at once, up to the lower of three and the tenant's
  device quota, for genuinely simultaneous behavior.

Mini cannot verify or perform:

- server-side, database, log, analytics, or other invisible state;
- SMS, phone calls, or email outside `@qa.minitap.ai` inboxes;
- biometrics, NFC, Bluetooth pairing, wearables, or unsupported hardware;
- camera input on mobile runs;
- connectivity changes on iOS or web, or airplane mode on any platform;
- real-money payments without an app-provided sandbox flow; or
- precise millisecond timing guarantees.

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
sync. Say **Offline (wifi off)** rather than airplane mode. Wifi control is
Android-only; iOS and web cannot execute network-toggle criteria.

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
