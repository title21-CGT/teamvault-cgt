---
metadata:
  confidence: 0.8
  created: '2026-07-30T07:46:10.689145+00:00'
  source: /teamvault-publish
  tags:
  - field-library
  - form-builder
  - pattern
  - bugfix
---

---
decision_type: pattern
kingdom: title21-CGT
palace: cgt
wing: field-library
hall: architecture
room: _
links:
  - https://github.com/title21-CGT/cgt-frontend/pull/188
tunnels:
  - field-library-export-fieldid-rewrite-pattern
---

# Fuzzy label-matching: always check exact match before substring match

## Context

Addendum to `field-library-export-fieldid-rewrite-pattern` (86eyf0nhd). CodeRabbit caught a real bug in that ticket's `guessFieldForRef` (`formExcel.ts`) during PR review (#188): a single `.find()` predicate that checked exact-match-OR-substring-match in one pass returns whichever field comes **first in array order**, not whichever match is **best**.

Concretely: fields `["Total Weight", "Weight"]`, ref `weight` → `normalizeForMatch("Total Weight")` = `"totalweight"`, which `.includes("weight")` → the substring check on "Total Weight" wins and short-circuits `.find()` before ever reaching "Weight", the actual exact match. This silently rewrites a calculated field's operand to the wrong field. Very plausible in this domain: Weight/Total Weight, Dose/Total Dose, Volume/Total Volume, etc.

## Decision

Always resolve exact matches in a separate pass **before** falling back to substring matching — never combine them into one `.find()` predicate:

```ts
const exact = fields.find(f => normalizeForMatch(f.label || '') === r)
if (exact) return exact
return fields.find(f => /* substring fallback only */)
```

Order-independent, and exact match — the strongest possible signal — always wins regardless of where it sits in the array.

## Known unfixed instance elsewhere

`guessOperandMapping` in `cgt-frontend/src/pages/FormEditorPage.tsx` (~line 1772, used to re-map Field Library placeholders onto real form fields at insertion time) has the **identical** bug — it was the precedent `guessFieldForRef` was deliberately modeled after, bug included, since nobody had noticed until this review. Not fixed as part of 86eyf0nhd (out of scope — a different feature/ticket). Worth a small standalone fix ticket if picked up: same one-line restructure, same regression-test shape (two fields with a substring collision, assert the exact match wins).

## Related

- field-library-export-fieldid-rewrite-pattern — the original pattern entry this amends
- https://github.com/title21-CGT/cgt-frontend/pull/188 — PR where CodeRabbit caught this