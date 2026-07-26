<!-- rendered by Minitap — do not edit -->
# The Minitest Executor (Mini)

Every scenario you design is ultimately run by **Mini**, the Minitest tester
agent, on a real device or browser. Design within what Mini can physically do and
observe — a scenario that asks for something Mini cannot perform fails as
unprocessable, not because the app is broken. This file is the envelope; consult
it whenever a step or acceptance criterion depends on a device capability,
connectivity, camera, multi-device simultaneity, or an identity/OTP.

## What Mini can and cannot do

Mini is the Minitap testing agent. It drives real Android/iOS devices through the
mobile-use CLI and executes acceptance criteria. Author scenarios ONLY within these
capabilities — a criterion Mini cannot physically perform or observe will fail as
unprocessable, not because the app is broken.

**What Mini can do on the device:**

- Gestures: tap (by label, coordinates, or percent), long-press, swipe, drag
  (including pick-up drag with hold-before-move), pinch/zoom (scale or explicit
  two-finger points).
- Text: type into the focused field or a targeted element, erase, press enter,
  navigate back/home (Android).
- Observation: screenshots (standard and high-res), compact or full UI hierarchy,
  find elements by text/label, visual questions answered from the screen.
- App lifecycle: launch, stop, open a URL/deep link, install a provided build.
- Media & audio: play an audio file through the device microphone path and speak
  text aloud (for voice-input flows). Web runs can also inject microphone audio
  and transcribe browser speaker output to verify audio responses.
- Camera (**web only**): the browser camera is simulated — a per-story image or
  MP4 video is fed into it as the live webcam feed, so camera-based flows (e.g.
  scanning a QR code, presenting a document) ARE testable in the web lane.
- Files: push a file to the device (for upload/attachment flows, seeded via
  story-bound test files).
- Connectivity (**Android only**): toggle wifi on/off and read connectivity state.
  iOS and web runs cannot change connectivity.
- Geolocation (**Android and iOS** cloud devices): mock the GPS position to any
  coordinates, simulate movement along a route at a given speed, and restore the
  real location. Web runs have no device location. Caveat: apps that rely on
  Play Services `GeofencingClient` transition callbacks may not react on Android
  (Limrun runs microG, not real GMS) — apps reading location directly work fine.
- Identity & email: read any `<prefix>@qa.minitap.ai` inbox at runtime (OTP codes,
  verification links, magic links). Sign in with Google via the shared Minitap
  account pool (leased at runtime, 2FA handled).
- Replay: re-run previously captured trajectories to reach a known state faster.
- Multi-device: a single scenario can drive **several devices at once** — up to
  `min(3, the tenant's device quota)` — set by its device-count setting. Auto
  (unset) resolves to one device per bound persona, minimum one — so a
  sequential multi-persona scenario pins an explicit count of 1. The count is
  decoupled from personas:
  two devices for one persona (a session-conflict test — the same account signed
  in twice) and one device for two personas (identities reused by signing in and
  out) are both valid. With several devices live at once Mini can assert
  **real-time cross-account behavior** — a message sent from account A on device 1
  appears for account B on device 2 — which a single device cannot observe. Same
  OS and same app on every device.

**What Mini cannot do — never write criteria that require:**

- Server-side, database, log, or analytics verification — only what is visible on
  the device screen counts.
- Receiving SMS or phone calls, or reading email outside `@qa.minitap.ai` inboxes.
- Biometric authentication (fingerprint/Face ID), hardware buttons beyond
  back/home, NFC, or Bluetooth pairing.
- Camera input on **mobile** runs (no real-world scene capture there). The web
  lane does NOT have this limit — see the simulated web camera above.
- Toggling connectivity on iOS or web, or using airplane mode anywhere (wifi
  toggle only, Android only).
- Pairing or interacting with a smartwatch, wearable, or other external hardware
  (a second phone or tablet IS supported — see multi-device above).
- Real payment card entry or purchases with real money (sandbox/test payment
  flows only, when the app provides them).
- Precise timing guarantees (e.g. "responds within 200ms") — Mini can observe
  order and outcomes, not millisecond latency.


## Device count

A scenario's **device count** decides how many devices one run drives at once. A scenario stored without an explicit count uses **auto**: one device per bound persona, minimum one — a story binding zero or one persona runs on a single device, a story binding several personas gets one device per persona at run time. An explicit integer overrides auto in either direction, capped at `min(3, the tenant's device quota)`. Omitting the field when *creating* a story leaves it on auto; on *updates*, omission leaves the current value unchanged — how to set or reset it is documented on each create/update call. The count is decoupled from the personas the story binds — see the persona-vs-device rules in "Test Profiles".

**Extra devices are only for simultaneity.** Go above one device **only** for flows that verify **real-time cross-account or cross-device behavior**: live chat or a call between two users, a push or notification one account triggers for another, presence/typing indicators, or cross-account visibility (A blocks B, and B can no longer see A). A journey where one identity acts and another checks the result **sequentially** does not need extra devices — the tester signs in and out on one. Because auto gives a multi-persona story one device per persona, **explicitly set the device count to 1 on every sequential multi-persona story**; never rely on auto to "cover more personas". Most multi-persona suites still run one device per story.

Multi-device is the deliberate exception, not a default to reach for. For an ordinary app with no simultaneous-identity flow the correct counts are: **unset on single-persona stories** (auto already means one device) and **an explicit 1 on multi-persona ones** — anything higher changes cost and behavior for nothing.


## Offline and connectivity

**Cover offline and connectivity scenarios when the app warrants it.** If the app shows offline support, cached content, or sync mechanisms, create dedicated stories to test those flows. Example criterion: "Disable wifi, confirm the app shows cached content, then re-enable and verify data syncs."

- Use the wording **"Offline (wifi off)"** — the tester toggles wifi, never "airplane mode".
- Connectivity control is **Android-only**; iOS and web runs skip network-toggle criteria.


## Story types

Every story carries a **type** used by the UI to group stories. Picking the right one is not optional.

1. **Try the built-in types first.** If the flow's primary purpose matches a built-in (`login`, `registration`, `checkout`, `onboarding`, `search`, `settings`, `navigation`, `form`, `profile`), use that exact type with no custom-type id.
2. **If no built-in fits, use `type: "custom"` with a real custom-type id.** Query the live list of custom types from the API first — never hardcode or invent type values. Reuse an existing custom type, or promote a new one (e.g. `Reservation`, `Loyalty`, `Order Tracking`, `Wishlist`, `Messaging`, `Notifications`, `Appointment Booking`, `Tipping`, `Ratings`, `Returns`, `Playback`, `Sharing` — pick a generic, reusable PascalCase name for the category). Then set BOTH `type: "custom"` AND the custom-type id on the payload.
3. **`type: "other"` is BANNED for any flow with a recognisable category.** Almost every real user flow has one. Before falling back to `other`, ask whether this flow maps to any named user-facing capability (reservations, messaging, tracking, subscriptions, payments, social features, etc.). If yes, promote a custom type. Expect to use `other` zero times in a typical generation.
4. **NEVER emit `type: "custom"` without a custom-type id** — the backend rejects it. **NEVER emit a built-in type with a non-null custom-type id** — the backend silently zeros it.


## Personas and the `@qa.minitap.ai` test-profile mechanics

The executor signs in and reads codes through Minitap's test-profile machinery.
The full persona doctrine below governs how you name profiles and, critically, how
the `@qa.minitap.ai` inbox, OTP, passwordless vs. provisioned accounts, and the
shared Google pool actually behave at runtime — design credentials and account
state to match.

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

