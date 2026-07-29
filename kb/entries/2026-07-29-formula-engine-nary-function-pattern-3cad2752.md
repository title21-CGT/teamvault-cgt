---
metadata:
  confidence: 0.8
  created: '2026-07-29T18:53:50.463436+00:00'
  source: /teamvault-publish
  tags:
  - formula-engine
  - calculated-fields
  - pattern
  - form-builder
---

---
decision_type: pattern
kingdom: title21-CGT
palace: cgt
wing: form-engine
hall: architecture
room: _
links:
  - https://app.clickup.com/t/86eyey1y3
tunnels:
  - clickup-ticket-grounding
---

# Formula engine: adding a new variadic (n-ary) function

## Context

The calculated-field formula language (`cgt-frontend/src/lib/formula.ts` mirrored byte-for-byte in `cgt-backend/src/form-engine/evaluate-formula.ts`) already had two variadic logical functions, `ANY(c1, c2, …)` / `ALL(c1, c2, …)`. Ticket 86eyey1y3 added the first **numeric aggregate** variadic functions, `SUM(e1, e2, …)` and `MAX(e1, e2, …)`. The pattern that emerged generalizes to any future n-ary function (e.g. a deferred `COUNT`).

## Decision

**Engine (both repos, kept identical):**
- All variadic functions share one parser branch (`parsePrimary`'s `ANY | ALL | SUM | MAX` check) — same `FUNC(`, comma-separated `parseExpr()` args, `)` grammar. Adding a function to the family is a matter of adding its keyword to `TOKEN_RE`, `classify()`, the `Expr` union's `op` literal, the parser's function-name check, and a branch in `evalNode`'s `case 'nary'`.
- Evaluation semantics differ per function but reuse `evalNode`/`toNum`: SUM/MAX skip blank (`null`) args, treat a non-numeric arg as blanking the *whole* result (same rule as `-`, `*`, `/`), and return blank only when *every* arg is blank. This is the "spreadsheet skip-blanks" convention — use it for any future numeric aggregate rather than inventing new blank-handling rules per function.
- **No special-casing needed for cross-form references** (`{handle.fieldId}` tokens, from the separate cross-form-refs ticket 86eyej9a4): the tokenizer's `REF_RE` (`\{([^}]+)\}`) already matches any content inside braces, and cross-form refs resolve to a flat value map *before* the formula evaluates — so any existing or future function that takes `{ref}` args gets cross-form support for free.

**UI (Form Builder + Field Library):**
- The raw formula field is **display-only** in the builder — there's no free-text formula editor. Every function needs a structured "build it, then insert the whole expression in one shot" widget (see `ConditionBuilder.tsx` for IF, `AggregateBuilder.tsx` for SUM/MAX), never a token-by-token scaffold — that's what made the old raw `IF ( , , )` UX unusable before `ConditionBuilder` existed.
- `ConditionBuilder.tsx` exports its operand-picker primitives (`OperandInput`, `OperandState`, `serializeOperand`, `emptyOperand`) specifically so new builders (like `AggregateBuilder`) can reuse the same field/number/text/expression picker instead of re-implementing it. Reuse these for any future structured builder rather than writing a new operand picker.
- Both `FormulaBuilder.tsx` (form-scoped fields) and `LibraryFormulaBuilder.tsx` (named placeholders) need the same button + builder wiring — they're parallel surfaces, not shared components, so a new function's UI must be added to both.

## Why

- One parse path for all n-ary functions keeps the grammar small and avoids drift between logical and numeric aggregates.
- The "non-numeric arg blanks the whole result" rule mirrors the existing `-`, `*`, `/` behavior so admins don't have to learn a second blank-handling convention.
- Reusing `OperandInput` across builders means a fix or new operand kind (e.g. a future cross-form picker) lands in one place, not N.

## Open

`COUNT` (and filtered aggregates) were explicitly deferred on 86eyey1y3 pending a real use case — when it lands, it's another branch in the same `case 'nary'` and another button next to SUM/MAX, per this pattern.

## Related

- 2026-07-29 ticket 86eyey1y3 (SUM/MAX) — https://app.clickup.com/t/86eyey1y3
- 2026-07-28 ticket 86eyej9a4 (cross-form field references) — in progress as of 2026-07-29