<!-- rendered by Minitap — do not edit -->
# The Minitest Executor (Mini)

Every scenario you design is ultimately run by **Mini**, the Minitest tester
agent, on a real device or browser. Design within what Mini can physically do and
observe — a scenario that asks for something Mini cannot perform fails as
unprocessable, not because the app is broken. This file is the envelope; consult
it whenever a step or acceptance criterion depends on a device capability,
connectivity, camera, multi-device simultaneity, or an identity/OTP.

## What Mini can and cannot do

Mini is the Minitap testing agent. It drives real Android, iOS and web sessions and executes acceptance criteria. A criterion Mini cannot physically perform or observe will fail as unprocessable, not because the app is broken.

**What Mini can do:**

- **Gestures:** tap (by label, coordinates or screen percentage), long-press, swipe, drag (including pick-up drag with a hold before the move), pinch and zoom. *(Android, iOS, web)*
- **Curved gestures:** an arbitrary path in a single press-move-release — circles, arcs, loops, figure-eights. Rotary dials, knobs, circular unlocks, pattern locks, signature pads and arc sliders are all testable. *(Android, iOS, web)* **Android:** Paths anchor to an element's bounds. **iOS:** Paths anchor to an element's bounds.
- **Text:** type into the focused field or a targeted element, erase, press enter. *(Android, iOS, web)* **Android:** Can also navigate back and home.
- **Observation:** screenshots (standard and high-res), a compact or full UI hierarchy, finding elements by text or label, and visual questions. *(Android, iOS, web)*
- **Retrospective transient observation:** inspect recent screen activity after it disappears to identify brief loading, transition, or intermediate UI states. This does not provide precise motion, easing, or sub-100ms continuous-video analysis. *(Android, iOS)*
- **App lifecycle:** launch, stop, open a URL or deep link, install a provided build. *(Android, iOS)*
- **Media & audio:** speak text aloud or play an audio file into the browser microphone, and transcribe what the browser's speakers output. *(web)*
- **Camera:** the browser camera is simulated — a per-story image or MP4 is fed as the live webcam feed, so QR scanning and document presentation are testable. *(web)*
- **Files:** a story can have files bound to it, and the harness places them on the device or machine before the run starts. Placing files is never Mini's job — bound files are listed in the goal under "Test Files" with a `device_path`, and Mini just uses them. A bound file that is missing or that the app cannot see is a Minitap harness failure to escalate to Minitap engineering — never a Mini limitation, and never something to ask the customer for. *(Android, iOS, web)*
- **Connectivity:** change the network state mid-run. *(Android, web)* **Android:** Wifi on/off, airplane mode, and reading the connectivity state. **web:** Take the page offline and back online.
- **Screen orientation:** rotate to landscape or portrait. *(Android, iOS)*
- **Device state:** grant or revoke permissions without the system prompt, and switch between light and dark appearance. *(Android, iOS, web)* **iOS:** Can also freeze the status bar to a fixed time and battery for stable screenshots.
- **Push notifications:** deliver an arbitrary payload to the app under test. The payload JSON is bound to the story as a test file and seeded before the run. *(iOS)*
- **Geolocation** (cloud devices): mock GPS, simulate movement along a route at a given speed, and restore the real location. *(Android, iOS)* **Android:** Play Services `GeofencingClient` transition callbacks may not fire (cloud devices run microG, not real GMS); apps reading location directly work fine.
- **Identity & email:** read any `<prefix>@qa.minitap.ai` inbox for OTP codes, verification links and magic links, and sign in with Google through the shared Minitap account pool (leased at runtime, 2FA handled). *(Android, iOS, web)*
- **Phone-OTP login** — *only when the persona carries both a phone number and a static OTP code.* Minitap owns no phone number and receives no real SMS: the customer whitelists a number and a fixed code in their own staging backend and stores both on the persona. Mini then enters the number and, on the app's OTP screen, the configured code. A persona missing either field cannot pass a phone-OTP screen at all. Configuration guide: https://www.minitap.ai/docs/suite/phone-otp *(Android, iOS, web)*
- **Replay:** re-run a previously captured trajectory to reach a known state faster. *(Android, iOS)*
- **Multi-device:** up to `min(3, tenant device quota)` devices at once, set by the scenario's device-count setting. Auto resolves to one device per bound persona (minimum one), so a sequential multi-persona scenario pins an explicit count of 1. The count is decoupled from personas — two devices on one persona for session-conflict tests, or one device and two personas by signing in and out. This is what makes real-time cross-account behaviour assertable. Every device runs the same OS and the same app. *(Android, iOS)*

**What Mini cannot do — never write criteria that require:**

- Server-side, database, log or analytics verification — only what is visible on screen counts. *(Android, iOS, web)*
- Read email outside `@qa.minitap.ai` inboxes. *(Android, iOS, web)*
- Receive a real SMS or phone call — Minitap owns no phone number, so any OTP that can only arrive by SMS is untestable unless the persona carries a static code (see the phone-OTP capability above). *(Android, iOS, web)*
- Biometric auth (fingerprint, Face ID), hardware buttons beyond back and home, NFC, or Bluetooth pairing. *(Android, iOS)*
- Use camera input. *(Android, iOS)*
- Feed audio into the microphone or hear what the app plays — the cloud device provider exposes no audio path, so voice-driven flows are web-only. *(Android, iOS)*
- Toggle connectivity or airplane mode. *(iOS)*
- Rotate the viewport or mock a location — the stealth browser pins the window to its fingerprint and blocks the geolocation override, so pick the right viewport preset up front. *(web)*
- Pair with a smartwatch, wearable or other external hardware (a second phone or tablet IS supported). *(Android, iOS, web)*
- Enter a real payment card or make a real-money purchase. Sandbox and test cards, in-app purchases and subscriptions through the RevenueCat Test Store or an Apple/Google platform sandbox, and an app's own custom payment flow are all testable, so this limits the money-moving step alone and not checkout as a feature. Write these criteria only against the test payment method the app actually provides — naming which one, since a Test Store build mocks billing while a platform sandbox drives the real store flow — and stop before the charge when no supported test payment method is available. *(Android, iOS, web)*
- Control or guarantee precise timing or gesture velocity ("tap within 200ms", "flick fast enough to fling the list") — action timing and gesture pacing are approximate, even when Mini can retrospectively observe the resulting transient state. *(Android, iOS, web)* **Android:** Pacing is noticeably slower than requested.

## Device count

A scenario's **device count** decides how many devices one run drives at once. A scenario stored without an explicit count uses **auto**: one device per bound persona, minimum one — a story binding zero or one persona runs on a single device, a story binding several personas gets one device per persona at run time. An explicit integer overrides auto in either direction, capped at `min(3, the tenant's device quota)`. Omitting the field when *creating* a story leaves it on auto; on *updates*, omission leaves the current value unchanged — how to set or reset it is documented on each create/update call. The count is decoupled from the personas the story binds — see the persona-vs-device rules in "Test Profiles".

**Extra devices are only for simultaneity.** Go above one device **only** for flows that verify **real-time cross-account or cross-device behavior**: live chat or a call between two users, a push or notification one account triggers for another, presence/typing indicators, or cross-account visibility (A blocks B, and B can no longer see A). A journey where one identity acts and another checks the result **sequentially** does not need extra devices — the tester signs in and out on one. Because auto gives a multi-persona story one device per persona, **explicitly set the device count to 1 on every sequential multi-persona story**; never rely on auto to "cover more personas". Most multi-persona suites still run one device per story.

Multi-device is the deliberate exception, not a default to reach for. For an ordinary app with no simultaneous-identity flow the correct counts are: **unset on single-persona stories** (auto already means one device) and **an explicit 1 on multi-persona ones** — anything higher changes cost and behavior for nothing.


## Offline and connectivity

**Cover offline and connectivity scenarios when the app warrants it.** If the app shows offline support, cached content, or sync mechanisms, create dedicated stories to test those flows. Example criterion: "Disable wifi, confirm the app shows cached content, then re-enable and verify data syncs."

- On Android, use the wording **"Offline (wifi off)"** — the tester toggles wifi, never "airplane mode".
- On web, the tester takes the page offline and back online, so word it as **"Offline"** rather than "wifi off".
- **iOS** runs cannot change connectivity — skip network-toggle criteria there.


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

