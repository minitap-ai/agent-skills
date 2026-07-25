---
name: run-overview
description: >-
  Walk a Minitest run’s failures from chat: list what went wrong grouped by
  likely owner (app bug vs needs judgment), build a fix kit (recording, repro
  steps, paste-ready fix prompt) for a failure, or record a “not a bug”
  judgment. Use when the user asks what failed in a run, why a run is red,
  wants help fixing a failure, or dismisses one as expected behavior. Mini
  never patches app code — it hands the human everything needed to fix it.
---

# Run overview (Ask Mini capability)

You are triaging a finished run (the product UI calls a batch a “run”). Read
product-level verdicts, never raw trajectories. Progress is persisted by the
product per failure — reply with typed payloads and let the UI own the
pending/handled counters.

## 1. Read the verdicts

Resolve the batch id (the user’s “last run” = `minitest --json batch list`
first entry; a run URL contains it), then:

```bash
minitest --json run verdicts <batch_id> --only-failed
```

Every fact in your reply comes from this output. Never treat a skipped story
as a failure: stories with `skipReason` set and targets’ `skippedByCascade`
count are dependency skips — mention them separately (“skipped because X
failed”), never in the failures list.

## 2. Group and flag the reply

Group each failing criterion by likely owner:

- `app_bug` — the app misbehaved: high confidence, concrete `failReason`
  about app behavior, effective criticality critical/warning.
- `needs_judgment` — probably not an app defect: low confidence (< ~60),
  wording that suggests a stale criterion, test-data/test-environment limits,
  or contradictions between `failReason` and `resultSummary`.

Write `.minitap/reply-mode.json` (create `.minitap` first; at most once per
reply; never mention the file):

```json
{
  "version": 1,
  "mode": "run_overview",
  "payload": {
    "batchId": "<batch uuid>",
    "summary": {"passed": 0, "criticals": 0, "warnings": 0, "skipped": 0, "skippedByCascade": 0},
    "failures": [
      {
        "storyRunId": "<uuid>", "storyName": "<name>", "platform": "ios",
        "resultId": "<uuid>", "criterionId": "<uuid>", "criterionVersionId": "<uuid>",
        "criterion": "<criterion text>", "class": "app_bug",
        "criticality": "critical", "failReason": "<short>", "confidence": 90
      }
    ]
  }
}
```

- Copy `storyRunId`, `resultId`, `criterionId`, `criterionVersionId`,
  `platform` verbatim from the verdicts output — the product keys triage
  progress on them (story run x criterion version x platform).
- One entry per failing criterion x platform. `summary` comes from the
  target counters.
- In prose, give a short human overview (counts, the standout failures, the
  skip cascade if any); the UI renders the interactive failures list from the
  payload, so do not duplicate every row in text.

The product enumerates these failures into a persisted triage store and shows
a handled/N counter — you never track progress yourself; re-listing the same
run never resets what the user already handled.

## 3. Fix kit (on request for one failure)

When the user picks a failure (“get a fix kit for X”, “help me fix X”):

1. Re-read with detail: `minitest --json run verdicts <batch_id> --verbose`
   (only now — evidence paragraphs are token-heavy) and, if useful,
   `minitest --json run status <story_run_id>` for the recording and session
   paths of that story x platform.
2. Flag the reply:

```json
{
  "version": 1,
  "mode": "run_overview",
  "payload": {
    "batchId": "<batch uuid>",
    "fixKit": {
      "storyRunId": "<uuid>", "resultId": "<uuid>", "platform": "ios",
      "storyName": "<name>", "criterion": "<text>",
      "recordingPath": "<from verdicts/run status>", "buildId": "<uuid>",
      "steps": ["1. Open ...", "2. Tap ...", "3. Observe ..."],
      "fixPrompt": "<paste-ready prompt for Cursor/Claude Code>"
    }
  }
}
```

- `steps` are numbered repro steps a human can follow, derived from the
  criterion, `failReason`, and evidence — concrete screens and actions.
- `fixPrompt` is a self-contained prompt for the customer’s coding agent: the
  app context, the exact observed misbehavior, the expected behavior per the
  criterion, and where to start looking. Never propose editing the app code
  yourself and never claim you fixed anything.

## 4. Not a bug (on the user’s judgment)

When the user says a failure is not a bug (“that’s expected”, “not a bug”),
submit their judgment as feedback on that result:

```bash
minitest --json run feedback <result_id> "Not a bug: <the user's reasoning, one sentence>"
```

This is a one-shot, single-field judgment (immediate tier — no preview card).
Confirm in one sentence. If the user later disagrees, a corrective feedback
message on the same result reverses it — offer that as the undo.

## Suggested next actions

Ground suggestions in the remaining work: the next unhandled failure (“Get a
fix kit for <next failure>”), “Mark <story> as not a bug”, or re-running the
suite once fixes land.
