---
name: run-overview
description: >-
  Turn a finished Minitest run into a persistent session issue board: separate
  customer app work from flaky acceptance criteria and Minitest product
  limitations, propose one test-suite remedy where appropriate, or build a fix
  kit for an explicitly selected app failure. Use when the user asks what failed
  in a run, why a run is red, or wants help fixing a surfaced issue.
---

# Run overview (Ask Mini capability)

You are triaging a finished run (the product UI calls a batch a "run"). Read
product-level verdicts, identify the mechanism behind each actionable issue,
aggregate occurrences of the same semantic issue across iOS, Android, and web,
and emit a v2 issue board. The product persists board state for the session and
keys it by `issueKey`; never track pending, handled, or dismissed state yourself.

## 1. Read the verdicts

Resolve the batch id (the user's "last run" is the first entry from
`minitest --json batch list`; a run URL contains the id), then run:

```bash
minitest --json run verdicts <batch_id> --actionable
```

`--actionable` is the triage shape: it already keeps only criteria that failed
or were unprocessable *and* are `critical` or `warning`, and it already drops
replay-only fields (`confidence`, `buildId`, `recordingPath`, `sessionPaths`).
Every row it returns is a candidate issue — do not re-derive that filter, and do
not use `confidence` or infrastructure signals as a classification proxy. Each
story carries `userStoryId` (stable scenario identity, used for
`archive_scenario` and for grouping across runs) and `userStoryName`; use them
as given rather than joining ids to names yourself.

Use only facts from this output. If the failure mechanism is not clear enough
to classify, re-read the batch with `--verbose`; do not guess.

Classify setup and build outcomes by ownership and remediation, not pipeline
stage. A signing, packaging, configuration, artifact, or build-prerequisite
failure is customer action when the verdict provides a customer-owned error
class and concrete user-action remediation. Silently omit only system-owned
infrastructure or build-execution defects, such as provisioning, device,
session, network, or service failures the customer cannot remediate. Do not
include omitted failures in the payload, prose, summary, or counts. A story with
`skipReason` or a target counted in `skippedByCascade` is a dependency skip, not
an issue; omit it as well.

## 2. Classify by mechanism

Surface each actionable issue in exactly one bucket:

- `customer_action`: the criterion is executable and observable, and the app,
  its data, or its customer-owned configuration did not meet it; or the verdict
  identifies a customer-owned signing, configuration, artifact, or build
  prerequisite error with concrete remediation. This bucket never has a
  proposal.
- `flaky_ac`: the acceptance criterion itself makes the verdict unreliable,
  for example because it is ambiguous, non-deterministic, state-dependent,
  combines assertions, or conflicts with the intended journey.
- `product_limitation`: Mini cannot reliably perform or observe the mechanism
  the criterion requires. Name the concrete execution or observability boundary,
  not merely "a product limitation."

Do not use confidence, likelihood thresholds, severity, or infrastructure as a
proxy for classification. Write a short noun-phrase `title` and a one-sentence
`reason` that states the causal mechanism and observed consequence.

Preserve effective `criticality` as exactly `critical` or `warning`; never infer
or downgrade it. When grouped occurrences differ, the issue is `critical` if any
occurrence is critical; otherwise it is `warning`. Criticality controls
presentation order, not bucket classification.

Every `flaky_ac` and `product_limitation` issue gets exactly one narrowest viable
proposal. Choose among:

- `edit_criterion` when a stable, supported assertion can preserve the intent.
- `remove_criteria` when the listed criteria should go but the scenario remains
  useful.
- `archive_scenario` when the scenario's core journey cannot produce a reliable
  result. This is a scenario-level remedy, so it belongs only to a
  `product_limitation` issue keyed by `userStoryId`; a `flaky_ac` issue is
  criterion-scoped and takes `edit_criterion` or `remove_criteria`.

In particular, do not list alternative remedies for a `product_limitation`.
Choose one.

## Output quality

Use concise active voice. Name the concrete mechanism and consequence. Never
hedge (`may`, `might`, `likely`) or substitute generic labels for an explanation.

**`summary`**

Exactly one sentence, at most 280 characters.

- Good: "The board separates an app behavior failure from an unstable criterion."
- Good: "The run surfaced customer configuration work and an unobservable scenario."
- Bad: "There may be several issues that need attention."
- Bad: "Run analysis complete."

**`title`**

Use 8-80 characters.

- Good: "Save button ignores valid input"
- Good: "Criterion depends on transient timing"
- Bad: "Possible issue with save"
- Bad: "Product limitation"

**Issue `reason`**

Exactly one sentence, at most 280 characters.

- Good: "The app keeps the form open after valid submission, so the user cannot reach confirmation."
- Good: "The criterion requires an exact animation duration that Mini cannot observe, so repeated runs can disagree."
- Bad: "This might be flaky because confidence is low."
- Bad: "The criterion failed due to an issue."

## 3. Emit the persistent issue board

Write `.minitap/reply-mode.json` (create `.minitap` first; at most once per
reply; never mention the file):

```json
{
  "version": 2,
  "mode": "run_overview",
  "payload": {
    "batchId": "<batch uuid>",
    "summary": "<one concise board-level sentence>",
    "issues": [
      {
        "issueKey": "<stable key per the rules below>",
        "bucket": "customer_action|flaky_ac|product_limitation",
        "batchId": "<batch uuid>",
        "userStoryId": "<user story uuid>",
        "storyName": "<exact user story name>",
        "criterionId": "<criterion uuid>",
        "criterion": "<exact current criterion text>",
        "occurrences": [
          {
            "platform": "ios|android|web",
            "resultId": "<criterion result uuid, when present>",
            "appFailureId": "<app failure uuid, when present>",
            "storyRunId": "<story run uuid, when present>",
            "criterionVersionId": "<current criterion version uuid, when present>"
          }
        ],
        "criticality": "critical|warning",
        "title": "<short issue title>",
        "reason": "<one-sentence causal explanation>",
        "proposal": {
          "type": "edit_criterion",
          "before": "<exact current criterion text>",
          "after": "<complete replacement criterion text>",
          "reason": "<why this edit removes the failure mechanism>"
        }
      }
    ]
  }
}
```

Aggregate all verdict evidence before emitting any issue. Identity is semantic,
never based on text similarity:

- A criterion-backed issue is identified by `criterionId` + `bucket`; set
  `issueKey` to `<criterionId>:<bucket>`.
- A scenario-level `product_limitation` is identified by `userStoryId` +
  `bucket`; set `issueKey` to `<userStoryId>:<bucket>`. Use this identity when
  the limitation applies to the scenario as a whole, including an
  `archive_scenario` remedy, so multiple criteria do not duplicate it.

Never include `platform`, `appFailureId`, `resultId`, or `batchId` in an
`issueKey`, and never derive identity from mutable `title`, `reason`, criterion
text, or other prose. Emit exactly one issue per identity and never emit
duplicate issues for the same identity.

Copy shared `batchId`, `userStoryId`, and `storyName` verbatim from the verdicts.
For a criterion-backed issue, also copy one shared `criterionId` and `criterion`
verbatim. A scenario-level issue omits criterion-specific issue fields unless
they are genuinely shared by every occurrence. Keep one shared `title`,
`reason`, and `criticality` per grouped issue. Set it to `critical` if any
occurrence is critical; otherwise set it to `warning`.

Put each underlying verdict occurrence in `occurrences`. Copy its `platform`
verbatim and include its `resultId`, `appFailureId`, `storyRunId`, and
`criterionVersionId` when supplied; omit only the unavailable fields. Every
occurrence must include at least one of `appFailureId` or `resultId` so Conductor
can resolve a Testing Service target; both may be present, but never emit an
occurrence with neither. Do not keep scalar `platform`, `resultId`,
`appFailureId`, `storyRunId`, or `criterionVersionId` fields on the issue. Do not
emit infrastructure or confidence fields anywhere in the payload.
Re-enumerating a run is an upsert into the session board and must not reset
existing issue state.

For `customer_action`, omit `proposal`. For the other buckets, attach exactly
one proposal to the grouped issue, regardless of its number of occurrences, and
use exactly one of these proposal shapes:

```json
{"type":"edit_criterion","before":"<exact current text>","after":"<replacement text>","reason":"<mechanism-based reason>"}
```

```json
{"type":"remove_criteria","criteria":[{"criterionId":"<uuid>","expectedCriterionVersionId":"<current version uuid>","content":"<exact current criterion text>"}],"reason":"<mechanism-based reason>"}
```

```json
{"type":"archive_scenario","userStoryId":"<uuid>","reason":"<mechanism-based reason>"}
```

For `edit_criterion`, `before` must match the current version exactly and `after`
must be paste-ready. For `remove_criteria`, include every criterion id to remove
with its current expected version and `content` copied exactly from the current
criterion. Before proposing `edit_criterion` or `remove_criteria` for a
criterion-backed grouped issue, verify that all occurrences carry the same
current `criterionVersionId`. If versions disagree, re-read current state instead
of emitting an actionable grouped issue; emit it only after resolving one shared
current version. Do not add proposal kinds or fields outside these contracts.

Keep `summary` to one sentence about the board as a whole. In prose, provide at
most one short framing sentence; the UI renders the issue rows and remedies, so
never restate rows or offer suggested actions that duplicate the board.

## 4. Fix kit (explicit follow-up only)

Build a fix kit only after the user explicitly selects a `customer_action` issue
and asks for help fixing it. A fix kit targets one occurrence of that grouped
issue; use the occurrence the user selected, or the clearest occurrence while
preferring a critical one if no platform was specified. Never emit a fix kit
during board enumeration.

Re-read the evidence with:

```bash
minitest --json run verdicts <batch_id> --verbose
minitest --json run status <story_run_id>
```

The second command is optional when the verdict already contains the recording
and session paths. Emit a separate v2 reply with `fixKit` and no `issues`:

```json
{
  "version": 2,
  "mode": "run_overview",
  "payload": {
    "batchId": "<batch uuid>",
    "fixKit": {
      "issueKey": "<the selected issue key>",
      "userStoryId": "<uuid>",
      "storyRunId": "<uuid>",
      "resultId": "<uuid>",
      "criterionId": "<uuid>",
      "criterionVersionId": "<current version uuid>",
      "platform": "ios|android|web",
      "recordingPath": "<path from verdicts or run status>",
      "buildId": "<uuid when present>",
      "steps": ["1. Open ...", "2. Tap ...", "3. Observe ..."],
      "fixPrompt": "<paste-ready prompt for the customer's coding agent>"
    }
  }
}
```

Derive concrete repro steps from the criterion, failure reason, and evidence.
The concise, self-contained `fixPrompt` must identify the app context, observed
behavior, expected behavior, and the likely code mechanism to inspect. Never
patch app code yourself or claim it is fixed.
