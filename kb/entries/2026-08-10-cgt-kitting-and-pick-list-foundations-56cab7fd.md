---
metadata:
  confidence: 0.8
  created: '2026-08-10T06:50:10.595577+00:00'
  source: /teamvault-publish
  tags:
  - kitting
  - pick-list
  - inventory
  - fefo
  - pattern
  - cgt
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
  - https://app.clickup.com/t/z8nrz7buqk
tunnels:
  - clickup-ticket-grounding
---

# Kitting and pick lists: the four decisions that cost rework

## Context

Built across the Kitting / Cleaning (`z8nrz7bur0`) and Pick List (`z8nrz7buqk`) epics. Four
things in this area were each got wrong once before being got right; all four are
non-obvious from the code and easy to repeat.

## 1. A "kit" is a FORM category, not a ProcedureTemplate type

The spec says *"Kit will be FORM with new Type = KIT"* and *"show ONLY Transplant with
their FORM Type = KIT"*. The schema also has `ProcedureTemplate.type` with a `KitCreation`
member — a **different axis**, added by the Transplant→Procedure rename. Gating kit
behaviour on `ProcedureTemplate.type` looks right and is wrong.

`FormCategory` originally had 8 members and no `kit`; adding it (migration
`20260809200000_add_kit_form_category`) was the keystone the rest depends on. Postgres
`ALTER TYPE ... ADD VALUE` cannot share a transaction with other DDL, so that migration
contains nothing else.

**Rule:** kit-ness is `FormDefinition.category === 'kit'`. A procedure whose forms are
*all* kit forms is a kit build and is hidden from the patient/procedure surfaces; one that
merely *contains* a kit form stays visible (spec p5). "Every form" is the test, not "any
form".

## 2. Kit expiry is derived, not stored — until it is frozen

Kit expiry is the **earliest component expiry**. Derived on read from the lots actually
recorded, so it can never contradict the component list, then **frozen at completion** so
a later lot edit cannot change what a released kit claims. Two follow-on rules:

- Undated lots are **ignored**, not treated as never-expiring — letting one win would push
  a kit's expiry past a dated component.
- A kit with **no components** must not be completable. With zero lines the "no blocking
  lines" check passes vacuously, and an empty kit would be released as stock with a null
  expiry. This shipped broken and was caught in manual testing, not by tests.

## 3. Process-step config keys on the LIBRARY field, not the placed field

The Form Builder's Process tab lists **`LibraryField` rows** (`useFieldLibraryList({
fieldType: 'process' })`), so a `ProcessStepConnection.fieldId` is a *library* field id.
A process field placed in a form gets a fresh instance id and keeps `libraryFieldId`.

The renderer already resolves `field.libraryFieldId ?? field.id`, so it works — but the
consequence surprises people: **every form using the same process library field shares one
bill of materials.** Distinct BOMs need distinct process library fields.

Corollary: `PUT /process-steps/:fieldId` deletes and recreates all child rows, minting new
`InventoryRequirement` ids on every save. Nothing durable may key on a requirement id —
pick list lines key on `materialId` and resolve live ids at write time.

## 4. Renaming a constraint whose target name is still occupied

The Transplant→Procedure rename renamed **tables** but left **constraints** behind:
`Procedure` carried `Transplant_pkey` while `ProcedureTemplate` held `Procedure_pkey`.
Prisma auto-generated a drift fix that failed with `42P07: relation "Procedure_pkey"
already exists`, because it tried to claim the name before its occupier released it.

**Rule:** when a batch of renames swaps names between objects, topologically sort them so
every target name is free when claimed — rename the occupier first. Fixed in
`20260809153246_align_renamed_constraint_names` (38 renames, 17 reordered). Until it was
applied, every `prisma migrate dev` regenerated the same broken migration.

## Also worth knowing

- **One definition of "available".** `src/common/inventory-availability.ts` was extracted
  because two existed and disagreed — the write path computed `held - settled` unfloored,
  so a lot whose CONSUMED+RELEASED exceeded RESERVED reported *more* available than the
  shelf held. FEFO cannot be correct against two definitions.
- **`FormDefinition.fields` has three shapes**: `SectionDefinition[]`, a legacy flat
  `FieldDefinition[]`, and arrays mixing both. `sections.flatMap(s => s.fields)` silently
  returns `[]` for every legacy form. Use `flattenDefinitionFields` /
  `processFieldIds` in `src/form-engine/definition-fields.ts`.
- **Kit assembly does not decrement stock yet** (`z8nrz7bygp`). Scans persist into
  `ProcessStepExecution.inventorySelections`, which the step's own Start/End settlement
  also consumes — decide which owns the decrement before adding one.

## Open

- Lot-level `FOR UPDATE` does not exist anywhere. The only row-lock precedent is equipment
  (`process-step-reservations.service.ts:211`). Needed before kit completion decrements
  shared lots.
- `InventoryLot` has no unique on `(tenantId, materialId, lotNumber)`, so duplicate lot
  numbers are possible and `findFirst` picks arbitrarily. Pick lists emit a
  `duplicate_lot_number` alert and always persist `pickedLotId` alongside the number;
  adding the unique needs a dupe-detection pass first.

## Related

- `z8nrz7bur0` Kitting / Cleaning epic — https://app.clickup.com/t/z8nrz7bur0
- `z8nrz7buqk` Pick List epic — https://app.clickup.com/t/z8nrz7buqk
- `z8nrz7buhh` Transplant→Procedure rename — the source of the constraint-rename trap