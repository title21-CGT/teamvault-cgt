---
metadata:
  confidence: 0.8
  created: '2026-08-04T06:59:27.858145+00:00'
  source: /teamvault-publish
  tags:
  - dnd-kit
  - form-builder
  - react
  - pattern
  - bugfix
---

---
decision_type: pattern
kingdom: title21-CGT
palace: cgt
wing: form-builder
hall: architecture
room: _
links:
  - https://app.clickup.com/t/86eyfxwky
tunnels:
  - clickup-ticket-grounding
---

# @dnd-kit grouped sortables: own the move in `onDragOver`, or it will move the DOM for you

## Context

Ticket 86eyfxwky: dragging a field from one Form Builder **section** into another crashed
the app with `NotFoundError: Failed to execute 'removeChild' on 'Node': The node to be
removed is not a child of this node.` Reordering *within* a section was always fine. The
distinction is the whole diagnosis.

We are on `@dnd-kit/react` / `@dnd-kit/dom` **0.4.0** — the rewritten dnd-kit, not the
classic `@dnd-kit/core` + `@dnd-kit/sortable`. Its `useSortable` behaves differently in a
way that is not obvious from the API surface.

## Root cause

`useSortable` ships `OptimisticSortingPlugin` **by default**. On every `dragover` where
source and target are both sortables, it mutates the DOM itself:

```js
// @dnd-kit/dom/sortable.js
function reorder(sourceElement, sourceIndex, targetElement, targetIndex) {
  const position = targetIndex < sourceIndex ? 'afterend' : 'beforebegin'
  targetElement.insertAdjacentElement(position, sourceElement)  // physically re-parents
}
```

- **Same container** (within-section reorder): a sibling shuffle. React reconciles a keyed
  list with `insertBefore` and never calls `removeChild`, so the vDOM and DOM stay
  compatible. No crash — which is why this looked like it worked.
- **Different container** (cross-section): the node is re-parented into the *destination*
  section while React's fiber tree still believes it lives in the origin. The next commit
  calls `originContainer.removeChild(node)` and throws.

The plugin is a **fallback for apps that don't manage the move themselves**, not a
convenience layered on top of your state. Its `dragover` handler re-reads every sortable's
`group`/`index` *after* React has rendered and returns early if they no longer match what
it captured — and `@dnd-kit/helpers`' `move` calls `event.preventDefault()` on no-op
drags, which also makes it bail. Both escape hatches exist so that **application state
wins**.

## Decision

For any **grouped** (multi-container) sortable, apply the move to React state in
`onDragOver` using `move` from `@dnd-kit/helpers`, keyed by the same group id the
sortables use:

```ts
// src/lib/formBuilderDnd.ts
export const sectionGroupId = (sectionId: string) => `sec:${sectionId}`

export function applyFieldMove(sections, event) {
  const groups = {}
  for (const s of sections) groups[sectionGroupId(s.id)] = s.fields
  const next = move(groups, event)
  if (next === groups) return sections          // no-op → same ref, React bails out
  return sections.map(s => {
    const fields = next[sectionGroupId(s.id)]
    return !fields || fields === s.fields ? s : { ...s, fields }
  })
}
```

The plugin then never touches the DOM, so the crash is structurally impossible rather than
patched around, and the live shuffle preview is preserved.

## Rules that fell out of this

1. **The group id must be a single exported helper.** The sortable `group`, the container's
   `useDroppable` id, and the `move` record key must be the same string. Three
   re-derivations of `` `sec:${id}` `` is a silent-drift bug waiting to happen.
2. **Do not run destructive normalization in `onDragOver`.** It fires continuously. Our
   `pruneUnlockRefs` / `pruneVisibilityRefs` passes drop rules that point at fields no
   longer earlier in the order — running them per-dragover strips a field's conditions as
   it merely passes *over* an intervening section, and nothing restores them. Prune once,
   on `dragend`.
3. **Moving in `onDragOver` means `onDragEnd` must handle cancel by rewinding.** State has
   already changed by drop time, so an Esc needs a pre-drag snapshot taken in
   `onDragStart` — `if (event.canceled) return` is no longer sufficient for that branch.
   Restoring the snapshot also makes the plugin's own cancel-restore bail out, consistently.
4. **`move` only understands targets that are a sortable item or a group key.** A drop on a
   *section header* (`section:<id>`, a sortable in a different group) is neither, so keep a
   target-resolving fallback on `dragend` for it.
5. **Empty containers were never the bug.** An empty section's only drop target is a plain
   `useDroppable`, and the plugin requires `isSortable(target)` before it does anything. A
   first, discarded attempt at this ticket hardened the empty-state rendering — the wrong
   target entirely. Read the plugin's guards before theorizing about React.

## Testing note

The repo's vitest run is node-only (`src/**/*.test.ts`), so the placement logic lives in a
pure lib module and is tested with a hand-built event stub — `move` only needs
`operation.{source,target,canceled}`, `source.manager.dragOperation.position`, and
`target.shape.center`. Pointer and keyboard sensors both resolve to the same drag operation
before this code runs, so one suite covers both; there is no separate keyboard path. Real
pointer-level coverage would need a browser harness we don't have.

## Open

Two other surfaces use dnd-kit and should be checked against rule 1–3 if they ever gain
cross-container drags: `src/components/scheduling/ResourceCapacityGrid.tsx` and
`src/components/equipment/schedule/TeamScheduleTab.tsx`. Both are currently
single-container as far as this ticket looked.

## Related

- 2026-07-30 field-library-export-fieldid-rewrite-pattern — same Form Builder area, unrelated mechanism
- 2026-07-29 formula-engine-nary-function-pattern — same area, unrelated mechanism