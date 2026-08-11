---
metadata:
  confidence: 0.8
  created: '2026-08-11T15:35:18.277655+00:00'
  source: /teamvault-publish
  tags:
  - kitting
  - inventory
  - cgt
  - pattern
  - process-steps
  - ux-copy
---

---
decision_type: pattern
kingdom: title21-CGT
palace: cgt
wing: inventory
hall: architecture
room: _
links:
  - https://app.clickup.com/t/z8nrz7bur0
tunnels:
  - cgt-kitting-and-pick-list-foundations
---

# Kit assembly and process-step Start are two writers on one record

## Context

Amends `2026-08-10-cgt-kitting-and-pick-list-foundations`, whose "Open" section already
said:

> Kit assembly does not decrement stock yet. Scans persist into
> `ProcessStepExecution.inventorySelections`, which the step's own Start/End settlement
> also consumes — **decide which owns the decrement before adding one.**

A decrement was added (`KitBuildService.consumeComponents`) **without making that
decision**. This entry records what that cost, so the warning is not ignored twice.

## The defect

`KitBuildService.scanLot` and `ProcessStepExecutionsService.start` both persist to
`ProcessStepExecution.inventorySelections` for the same `(submissionId, fieldId)` — the kit
`fieldId` is `processStepKey(field)`, the same library-field key the renderer resolves. Three
consequences, all reachable:

1. **Start wipes scans.** `start()` replaces the whole array and an empty allocation still
   sends `[]`. A kit run's execution never leaves `ASSIGNED`, so the renderer keeps
   offering *Initiate* on the very execution the kit screen is writing into.
2. **Double decrement.** Both write `CONSUMED`, deduped on **different keys** —
   `requirementId:lotId:qty` in `settleInventoryFor`, `requirementId:lotId` in
   `consumeComponents`. Different quantities → no match → the lot decrements twice.
3. **The kit path is the weaker control.** `start()` requires an assignee, requires the
   caller *be* the assignee, and verifies the Cognito password. Kit scan/sign/complete
   check none of that, and the "signature" is a typed name with no audit row — on the
   regulated action.

## The rule

**One lifecycle per record.** If a feature needs a status, a signature and a completion
gate over work that a process step already models, it is a *step*, not a parallel object.
Assembly is `start` → `updateSelections` → `end` → `complete`; the dual e-signature is
`end` + `complete`, which are already two password-verified signatures by two actors.
`KitBuild` survives as the **record** (supply id, frozen expiry + components, produced
stock) and stops being a lifecycle.

## The dedupe-key trap, generalised

`requirementId:lotId:qty` is not a sound identity for "has this reservation been settled".
Churn a lot 5 → 3 → 5 and the log holds `R5, REL5, R3, REL3, R5`; the settled-key set is
`{…:5, …:3}`, so **all three** reservations are skipped and nothing is consumed.

Fix: `InventoryMovement.resolvesId` with `@@unique`, linking each `CONSUMED`/`RELEASED` to
the `RESERVED` row it settles. Settlement becomes "every RESERVED with nothing pointing at
it", and a second settlement is a `P2002` at the database rather than a logic bug. Prefer a
real link over a composite string key whenever "已 processed?" must be exact.

## Error copy: short, and one fact per message

No prior convention existed, so: **one sentence, no instructions the reader did not ask
for, no restating what they just did.** Before → after, from this area:

| Before | After |
|---|---|
| `This kit has no components configured. Add inventory to the kit form's process step before completing it.` | `This kit has no components. Add inventory to its process step.` |
| `Lot X does not belong to MAT-CD3. Scan a lot of the correct material.` | `Lot X is not a lot of MAT-CD3.` |
| `The verifying signature must be from a different person than the one who kitted it.` | `The verifier must be a different person from the assembler.` |
| `Completion blocked. 1 required line(s) not fulfilled: …` | `1 line(s) still need a lot: …` |
| `This kit has no started process step yet — start the kit form before scanning.` | `Initiate this run before scanning.` |

Two costs the long form actually carried here: the "start the kit form" message pointed at
the patient procedure page, **which a patient-less kit can never open**, and the
"no components" message named a process step the reader had no way to identify when two
library fields shared the label "Process". Long copy hides wrong copy.

**Test coupling:** four specs asserted on message substrings and broke on the rewrite. That
is the assertions being over-specific, not the rewrite being wrong — match the shortest
stable fragment, never the sentence.

## Client model (11 Aug 2026)

> New KIT does not require an assign to resource — it requires selecting a Procedure that =
> KIT, then you create the KIT Procedure record. You can then assign.

So creation takes the **template only**. Assignment happens afterwards, through the normal
procedure assignee. Since `start()` refuses an unassigned step, the Kits list's Assigned To
column is load-bearing, not decorative.

## Related

- `2026-08-10-cgt-kitting-and-pick-list-foundations` — the entry this amends; its rules 1–3
  (kit = FormCategory, expiry derived-then-frozen, config keyed on the library field) still hold
- `z8nrz7bygp` — kit assembly stock movement