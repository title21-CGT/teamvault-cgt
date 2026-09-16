---
metadata:
  confidence: 0.8
  created: '2026-09-16T09:58:33.993988+00:00'
  source: /teamvault-publish
  tags:
  - dnd-kit
  - form-builder
  - drag-and-drop
  - bugfix
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

# Two `@dnd-kit` 0.4 traps: `canceled` means Escape only, and `move` reorders by a stale index on an off-list target

## Context

Found reviewing `z8nrz7djxb` (reorder a select field's inline options). Both bugs were
written, passed unit tests and type-checks, and would have shipped silently changing saved
form definitions. Both are properties of `@dnd-kit` 0.4, so they apply to every drag
surface in this repo — the Form Builder canvas, scheduling, the topology diagram.

## Trap 1 — `event.canceled` does NOT mean "the drop was rejected"

It means **Escape was pressed**. Verified in `@dnd-kit/dom@0.4.0/index.js`:

```js
// handlePointerUp — the ordinary mouse-release path
const canceled = !status.initialized;
this.manager.actions.stop({ event, canceled });

// handleCancel — the Escape path
if (dragOperation.status.initialized) {
  this.manager.actions.stop({ event, canceled: true });
}
```

`status.initialized` is true for any drag past initialization, so **every real pointerup
reports `canceled === false`** — including a release over empty space, over a different
container, or anywhere off the drop target.

**Consequence for any handler that applies the move during `dragover`** (which is the
house pattern here — see `dnd-kit-grouped-sortables-own-the-move-in-dragover`): testing
`event.canceled` alone leaves the previewed move committed on a drop-outside. There is no
error and the UI looks deliberate.

**Rule:** a drag applied live during dragover must decide "commit or restore" from the
**drop target**, not from `canceled`:

```ts
const committed = !event.canceled && <target is a valid member of this list>
if (!committed) restoreFromSnapshot()
```

`canceled` still covers Escape; the target check covers everything else.

## Trap 2 — `move` reorders by a stale index when the target is off-list

`@dnd-kit/helpers`' `move(items, event)` looks up both source and target in `items`. When
either misses it does **not** bail:

```js
// @dnd-kit/helpers/dist/index.js
if (sourceIndex2 === -1 || targetIndex2 === -1) {
  if (hasSortableIndices(source)) {
    const from = source.initialIndex;   // PRE-drag index
    const to = source.index;            // current projected index
    ...
    return mutation(items, from, to);
  }
  return items;
}
```

`initialIndex` is the index before the drag started, but `items` has already been rewritten
by earlier dragovers. So an off-list target produces a reorder of **an element the user
never grabbed**.

Worked example — options `A,B,C,D`, drag `A` onto `C`: dragover commits `B,C,A,D` and
`source.index` becomes 2 while `initialIndex` stays 0. Slide the pointer onto a canvas
field: that dragover runs `arrayMove(['opt:b','opt:c','opt:a','opt:d'], 0, 2)` → `C,A,B,D`,
moving `B`. Release there and it saves.

**Rule:** screen the target before calling `move`. Never rely on `move` to no-op.

## Why an off-list target is reachable at all

A sortable `group` only drives index math — **it does not gate collision detection**. No
`useSortable` site in the builder sets `type`/`accept` (`CanvasField.tsx`,
`FormCanvas.tsx`), and `Droppable.accepts` returns `true` when `accept` is unset. So
nesting two drag contexts inside one `DragDropProvider` — which the options list does,
since the Properties panel renders inside the canvas provider — means each context can win
the other's collisions.

Setting `type` + `accept` on **both** sides is the structural fix. Setting it on one side
only buys you half: an option row with `accept: ['option']` refuses a canvas field, but a
canvas field with no `accept` still accepts an option drag. Until both sides are
constrained, **the real guard belongs in your own handler** as an explicit
"is this target a member of my list?" predicate.

## Testing note

Both bugs passed a green unit suite. Two reasons, both worth copying as habits:

1. **Stub the ids a row actually registers with.** The rows register as
   `optionSortableId(o.id)`, not the bare id. Stubs using bare ids passed against an
   implementation that no-ops in the real panel.
2. **Assert on the persisted shape, not the in-memory array.** Asserting on
   `toSectionDefinitions(...)` output is what exposed trap 1's sibling bug. The builder
   array and the saved `string[]` are different shapes and only one is the contract.

A drop-outside case cannot be covered by feeding `canceled: true` — that is the Escape
path. Cover it with `canceled: false` plus an off-list target id.

## Related

- `2026-08-04-dnd-kit-grouped-sortables-own-the-move-in-dragover` — establishes the
  apply-during-dragover pattern; these two traps are the bill that pattern comes with
- `select-option-identity-is-the-label-not-an-id` — why a spurious option reorder is a
  data change and not a cosmetic one
- `z8nrz7djxb` — https://app.clickup.com/t/z8nrz7djxb