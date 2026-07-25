---
name: dependency-change
description: >-
  Change the dependency graph between a Minitest app’s scenarios from chat:
  add, remove, or replace which scenarios run after which. Use when the user
  asks to make one scenario depend on another, run after another, remove a
  dependency, or reorder the run flow. Always previews the blast radius and
  applies only after an explicit confirm turn.
---

# Dependency change (Ask Mini capability)

You are changing which scenarios depend on which — a blast-radius edit that
alters the run order and what gets skipped on failure. It is ALWAYS
preview-tier: never write before a `[Structured apply turn]` arrives.

Vocabulary: the product UI says “scenario” for user story and “run” for batch.
A depends on B means B runs first and A is skipped if B fails.

## 1. Read, then simulate — never do graph math yourself

Read the current graph:

```bash
minitest --json apps dependencies
```

Then compute the outcome of the requested edit with the deterministic
simulator. Edge syntax is `<story_id>:<depends_on_id>` (“story depends on”):

```bash
minitest --json apps dependencies --simulate \
  --add <story_id>:<new_parent_id> \
  --remove <story_id>:<old_parent_id>
```

Repeat `--add`/`--remove` for multi-edge changes (a multi-parent replace is
one `--remove` per old parent plus one `--add` per new parent). The simulation
returns `valid`, `cycle`, `addedEdges`, `removedEdges`, `affectedStories`,
`runOrder` (waves that run in parallel), and the resulting edges. Every
slice/order/cycle fact in your preview MUST come from this output — never from
your own reasoning over the edge list.

## 2. Flag the reply (preview card)

Write `.minitap/reply-mode.json` (create `.minitap` first; at most once per
reply; never mention the file):

```json
{
  "version": 1,
  "mode": "dependency_change",
  "payload": {
    "proposal": {
      "changes": [
        {
          "storyId": "<id>", "storyName": "<name>",
          "addDependsOn": [{"id": "<id>", "name": "<name>"}],
          "removeDependsOn": []
        }
      ],
      "proposedDependencySet": [
        {"storyId": "<id>", "dependsOn": ["<parent_id>", "..."]}
      ]
    },
    "graphSlice": {
      "nodes": [{"id": "<id>", "name": "<name>", "type": "<type>", "affected": true}],
      "edges": [{"source": "<parent>", "target": "<child>", "state": "existing|added|removed"}]
    },
    "impact": ["<one short bullet per consequence>"],
    "runOrder": [[{"id": "<id>", "name": "<name>"}]],
    "simulation": {"valid": true, "cycle": null}
  }
}
```

- `proposedDependencySet` lists the FULL new parent set for every story whose
  parents change — it is what the apply turn executes.
- `graphSlice` covers only the affected slice plus direct neighbours, taken
  from the simulation; mark changed edges `added`/`removed`.
- `impact` is 2–4 bullets: serialization cost, newly skip-on-fail chains, wave
  changes. `runOrder` is the simulator’s waves verbatim.
- Do NOT invent a `proposalId` — the product pins one server-side.
- In prose, summarise the change and ask the user to Apply or Cancel; the UI
  renders the card and buttons from the payload, so do not repeat the graph in
  text.

If the simulation says `valid: false`, refuse: explain the cycle path in prose
and flag the reply with `simulation` and NO `proposal` (nothing to apply). Do
the same when a referenced scenario does not exist.

## 3. Apply only on the structured turn

When a `[Structured apply turn]` arrives with the confirmed proposal:

1. Re-read `minitest --json apps dependencies` and re-run the same
   `--simulate` — state may have changed since the preview. If the simulation
   is no longer valid or the affected scenarios changed, report the divergence
   and stop; do not improvise.
2. Apply the full new parent set per changed story:

   ```bash
   minitest user-story update <story_id> --depends-on <parent_id> [--depends-on <parent_id2> ...]
   minitest user-story update <story_id> --remove-dependency <old_parent_id>
   ```

   `--depends-on` REPLACES the whole parent set (pass every parent, including
   kept ones); use `--remove-dependency` alone for pure removals. The server
   re-validates same-app, no-cycle, no-self-loop — if it rejects, report its
   error verbatim.
3. Verify with a final `minitest --json apps dependencies` read and confirm
   the result in one sentence, product language.

Never apply from a plain user message (“yes” arrives as a structured turn —
if it does not, ask them to use the Apply button or re-state the request).

## Suggested next actions

Ground suggestions in the new graph: the natural follow-up edit (“Also make
Payment depend on Login”), a run of the affected slice, or reviewing a story
the change newly serializes.
