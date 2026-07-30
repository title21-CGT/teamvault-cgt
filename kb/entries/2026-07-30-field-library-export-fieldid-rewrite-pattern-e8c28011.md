---
metadata:
  confidence: 0.8
  created: '2026-07-30T07:06:02.811133+00:00'
  source: /teamvault-publish
  tags:
  - field-library
  - excel
  - round-trip
  - pattern
  - form-builder
---

---
decision_type: pattern
kingdom: title21-CGT
palace: cgt
wing: field-library
hall: architecture
room: _
links:
  - https://app.clickup.com/t/86eyf0nhd
tunnels:
  - formula-engine-nary-function-pattern
---

# Field Library Excel export: reconstructing an unpersisted Field ID at round-trip time

## Context

Ticket 86eyf0nhd added "Export fields (.xlsx)" to the Field Library page — a sibling to the existing "Download template" blank-template flow, snapshotting real `LibraryField` data into the same sheet shape so it round-trips (export → import produces no errors, no unintended diffs).

## The gap

The import sheet's **Field ID** column (e.g. `weight` for a row labeled "Patient Weight") declares the `{placeholder}` name other rows in the *same file* can reference in a calculated field's formula. Traced through both repos before implementing: **this value is never persisted.** `LibraryField` (`prisma/schema.prisma`) stores `id` (UUID), `label`, `formula` (verbatim), `formulaRefs` (just the extracted `{token}` strings, unresolved) — no column remembers "this field declared itself as `weight`." The backend's own import (`field-library.service.ts`) doesn't even validate ref cross-references — only the frontend's `parseLibraryFieldWorkbook` does, and only within a single uploaded file. Field Library formula placeholders are free text, decoupled from any real field record, resolved only much later at "insert onto a form" re-map time.

**Consequence:** exporting a calculated field's `formula` verbatim with a blank Field ID column breaks re-import — `parseLibraryFieldWorkbook` rejects any `{ref}` that no row's Field ID column declares.

## Decision

Since there is no ground truth to reconstruct, export **invents** a deterministic Field ID and **rewrites** formula refs to match:

1. **Field ID** = slug of the field's own label (`slugifyLabel`: lowercase, non-alnum runs → `_`, trimmed), deduped on collision (`_2`, `_3`, …).
2. **Formula ref rewrite** (`rewriteFormulaForExport`): for each `{ref}` token in a calculated field's formula, fuzzy-match it against every field's label using the *same* normalize-and-substring heuristic `FormEditorPage.tsx`'s `guessOperandMapping` already uses to re-map library placeholders onto real form fields (lowercase, strip non-alnum, then equality or substring-containment either direction). If a confident match is found, rewrite the ref to that field's new slug. If not, **leave it untouched** — re-import will then correctly reject it with a clear "no row declares Field ID X" error, rather than silently dropping the reference.

This resolves the common case (placeholders that were originally named after their field, e.g. `{weight}` for a field labeled "Patient Weight") and fails loudly — not silently — on the residual case where a formula ref genuinely doesn't correspond to anything in the current library.

**Known, disclosed side-limitation, not solved by this ticket:** a field can belong to multiple categories (`categoryIds: string[]`) but the import/export sheet's Category column is single-valued — export takes only the first. This is a pre-existing constraint of the sheet format (the ticket's own AC required matching the import template's columns exactly), not something introduced here.

## Why

- Reusing `guessOperandMapping`'s exact heuristic (re-derived locally in `formExcel.ts` rather than imported, since it's a page-local, non-exported helper) keeps the "guess which field a placeholder means" logic consistent across the codebase's two separate places that need it (form re-map, export rewrite) instead of inventing a second guessing algorithm with different edge-case behavior.
- Failing loudly on unmatchable refs (rather than e.g. silently stripping them or emitting an internal UUID nobody can hand-edit) matches the existing philosophy of `parseLibraryFieldWorkbook`'s validation — errors are always row-referenced and actionable.

## Open

If this gap becomes a recurring pain point (e.g. large libraries with many free-form/non-label-matching placeholders), the real fix is a schema change: persist a Field ID / placeholder-name column on `LibraryField` itself so it's no longer a file-scoped, re-derived value. Not attempted here — out of scope for an export-only ticket.

## Related

- 2026-07-30 ticket 86eyf0nhd (Field Library Excel export) — https://app.clickup.com/t/86eyf0nhd
- formula-engine-nary-function-pattern (SUM/MAX) — same "extend the shared formula engine + mirror the UI builder pattern" area, unrelated mechanism