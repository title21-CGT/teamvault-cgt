---
metadata:
  confidence: 0.8
  created: '2026-09-16T09:50:56.155904+00:00'
  source: /teamvault-publish
  tags:
  - form-builder
  - dnd-kit
  - select-options
  - data-model
  - pattern
---

---
decision_type: pattern
kingdom: title21-CGT
palace: cgt
wing: form-builder
hall: architecture
room: _
links:
  - https://app.clickup.com/t/z8nrz7djxb
tunnels:
  - clickup-ticket-grounding
---

# A select option's identity is its LABEL, and `SelectOption.id` never leaves the browser

## Context

Ticket `z8nrz7djxb` (reorder a select field's inline options) opened with a blocking
question: does reordering options rewrite historical clinical answers? It framed the
answer as two branches — options carry a stable id (safe) or submissions store an array
index (catastrophe: `No` becomes `Yes` on filed records).

Neither is what this codebase does, and the third answer is not visible from any single
file. Anyone touching option ordering, renaming, dedup or migration needs it first.

## The actual model

Four layers, three shapes:

| Layer | Shape | Where |
|---|---|---|
| Builder, in memory | `SelectOption[]` = `{id, label}` | `form-builder/types.ts` `SelectOption` |
| Saved definition | `options?: string[]` — **labels only** | `types/cgt.ts` `FieldDefinition.options` |
| Builder → save | `f.options.map(o => o.label)` — **id dropped** | `FormDefinitionPanel.tsx` `toSectionDefinitions` |
| Load → builder | `{id: crypto.randomUUID(), label: o}` — **fresh id every load** | `FormEditorPage.tsx`, `field-config.ts` |
| Submission value | `values: Record<string, unknown>`, field id → **label string** | `types/cgt.ts` `FormSubmission.values`; `FormRenderer` renders `<SelectItem value={label}>` |

**`SelectOption.id` is ephemeral and session-local.** It is minted on load, used as a React
key and a sortable id, and thrown away on save. It is never persisted and never the same
twice. Nothing durable may key on it.

**Option order is the only thing about an option that round-trips**, other than the label
text itself.

## What follows

- **Reordering is safe, for free.** Presentation order lives in the array; stored answers
  key on the label. No migration, no data-model change. `z8nrz7djxb` stayed a UI ticket.
- **Renaming a label is the dangerous operation**, and it is already permitted by the
  existing Add/Remove/edit UI with no guard. Renaming `No` → `Not known` silently orphans
  every filed submission holding `No`, on every version that submission belongs to. That is
  a real, unticketed exposure — not caused by reorder, but revealed while scoping it.
- **Duplicate labels are indistinguishable once saved.** Two options labelled `Yes` persist
  as `['Yes','Yes']`; a submission storing `Yes` cannot say which. In the builder they are
  distinct (ephemeral ids), so a reorder can target the grabbed row — but the distinction
  dies at save. `FormRenderer` also emits duplicate React keys for them.
- **Empty-label options are dropped in two existing paths**, both silently: the table-cell
  renderer (`(resolved.options ?? []).filter(Boolean)`) and Excel import
  (`.filter(Boolean)`). Excel import additionally `.trim()`s every label and splits on `;`,
  so a label containing `;` or meaningful whitespace does not survive a round-trip.
- **Zero-option select fields are unreachable through the UI** (Remove is disabled at
  `length <= 1`) but reachable via import and legacy data. Handle them; don't assume them
  away.

## The dnd-kit trap this produced

`@dnd-kit/helpers`' `move(items, event)` matches the event's source/target against the ids
of **the array you hand it**. Option rows register with dnd-kit as
`optionSortableId(o.id)` (`opt:<id>`) so an option can never be confused with a canvas
field, while the options array carries the bare `id`. So `move(options, event)` matches
nothing and returns the list **unchanged** — a no-op drag that looks exactly like a working
one, with no error.

Worse, it tests green if the test stubs use bare ids. That happened here: the first unit
suite passed against a `applyOptionMove` that did nothing in the real panel. What caught it
was asserting on the **saved definition** (`toSectionDefinitions`) rather than on the
in-memory array.

**Rules:**

1. Hand `move` the same id space the sortables registered with — map to sortable ids, move
   those, map back.
2. Stub drag events with the **prefixed** ids a row actually registers, never the bare
   domain id.
3. For anything option-ordering-related, assert on the saved `string[]`, not the builder
   array. The two are different shapes and only one is the contract.

## Related

- `2026-08-04-dnd-kit-grouped-sortables-own-the-move-in-dragover` — same drag stack; its
  cross-container `removeChild` crash does NOT apply to the options list (flat, single
  container), but its "application state owns the move" rule and its node-only testing
  shape both carried over
- `z8nrz7djxb` inline option reorder — https://app.clickup.com/t/z8nrz7djxb
- `86ey3urkw` Option Sets — the `optionSetId` path, which bypasses inline options entirely