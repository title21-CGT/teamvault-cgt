---
metadata:
  confidence: 0.8
  created: '2026-08-20T13:59:52.520816+00:00'
  source: /teamvault-publish
  tags:
  - auth
  - e-signature
  - cognito
  - cgt
  - pattern
  - audit-log
---

---
decision_type: decision
kingdom: title21-CGT
palace: cgt
wing: auth
hall: architecture
room: _
links:
  - https://app.clickup.com/t/z8nrz7c9cu
  - https://github.com/title21-CGT/cgt-backend/pull/181
  - https://github.com/title21-CGT/cgt-frontend/pull/308
tunnels:
  - cgt-kit-lifecycle-two-writers-and-error-copy
---

# Signing is a password now: the token design, and the identity that took three tries

## Context

A signature used to be the operator typing their own name, matched against their account.
That is an *assertion* by the signer, not an *authentication* of them — the audit trail could
only show that someone who knew the name was at the keyboard. `2026-08-11-cgt-kit-lifecycle-two-writers-and-error-copy`
already flagged this as the weaker control on the regulated action. `z8nrz7c9cu` reversed it.

## The design

The signer re-enters their password and exchanges it for a **single-use, 60-second signing
token**, which the signing endpoint itself spends as the change is applied.

- `POST /auth/signing-token` verifies against the **authenticated principal**. No email in
  the body, so the route cannot be used as a password oracle.
- A `SignatureGuard` + `@RequiresSignature()` enforce it on all 12 signed routes **before the
  handler runs**, so a refused signature cannot half-write a record.
- Redemption is **one conditional update** on `usedAt IS NULL` + unexpired + same principal.
  Two concurrent requests cannot both spend one token.
- The token binds to **`sub`, not `userId`**. `sub` is the one identifier every principal has,
  including a token-only user with no local row, so the check stays total. The kit's
  "different, qualified" dual-signature rule depends on a token minted by one person never
  signing as another.

Two things that do not fit the guard:

- **Signature *fields*** redeem inside `FormSubmissionsService` instead. Which fields are
  signatures is only knowable from the form definition, so a route-level guard cannot see it.
- **Refusals are 400, not 401.** The SPA treats every 401 as a dead session and redirects to
  `/login` — a mistyped signing password would have thrown the operator out mid-step. This is
  a general rule for this codebase, not a signing quirk: *never answer a recoverable
  in-session failure with 401.*

`signedBy` is gone from every DTO; the signer is resolved from the request principal, so a
crafted request can no longer put another name on an audit row. `/auth/verify-password` — an
unauthenticated route that reported whether a password was correct, with zero callers — was
deleted rather than left as a target.

## The equipment carve-out was not needed

The ticket flagged equipment signing as blocked: `StartEventDto` said the module had no
authenticated principal, so a password could not be verified for it. **The comment was
stale.** Those routes carry `@Permissions('equipment.execute')`, so `request.user` was always
populated. Equipment moved with everything else, and Undo-End — which had no signature at all
before — gained one.

**Rule:** a comment claiming a module lacks auth is a claim about the past. Check the route
decorators before scoping work around it.

## The identity bug, which took three tries

Signing refused a correct password. The fix went wrong twice, and the second wrong turn is
the one worth remembering.

1. **Verified against `User.email`** (the local column). That column and the Cognito identity
   drift — this instance has a user whose local email column and whose Cognito username are
   two entirely different addresses. Cognito does not know the first, so it answered
   `NotAuthorized` no matter what was typed.
2. **Resolved the signer through `AdminGetUser`**, copying `resolveDbUser` in the auth guard,
   which does exactly this for exactly this reason. But `AdminGetUser` **needs AWS
   credentials**, and this environment has none — `.env` names `AWS_PROFILE=cgt` and there is
   no `~/.aws` at all. The lookup threw, the catch fell back to `User.email`, and that is the
   broken path it was meant to avoid.
3. **Read the access token's own `username` claim** (`RequestUser.cognitoUsername`). That IS
   the pool's Username — the email in a legacy pool, the UUID in an email-alias pool — so
   `InitiateAuth` accepts it either way. It cannot drift from the account, and reading it
   costs no AWS call.

**The generalisable trap:** `InitiateAuth` is an **unauthenticated** Cognito API. So an
environment can sign users in perfectly well while every credentialed AWS call fails. An
identity lookup that quietly needs credentials therefore fails *only at signing time* — which
is precisely how this hid through sign-in, through tests, and into manual use.

`RequestUser.username` could **not** be reused for this: it prefers the local `User.username`
column and so masks the claim.

Also split one refusal message in two. A refused password and a spent token are different
problems for the signer, and sharing a message is what made a wrong-password bug
indistinguishable from a token bug on the only screen anyone would look at. The *redemption*
message stays vague about which redemption failure it was — that one would tell a guesser
whether a token exists.

## The cross-repo drift, and the test that now spans it

The backend stopped accepting `signedBy`; five frontend clients kept sending it. The global
`ValidationPipe` runs with `forbidNonWhitelisted`, so the field is not ignored — **the whole
request is rejected** with `property signedBy should not exist`. The signature minted fine and
the action behind it failed anyway.

Nothing failed until a human clicked Save & Sign. The gap is closed by
`src/api/signedRequests.test.ts`, which asserts each signed body carries the token and *not*
the signer; putting `signedBy` back into one client turns its test red.

**Rule:** when a DTO drops a field, the contract test belongs on the **client** side. The
server's own specs cannot fail for a field the server no longer knows about.

## Frontend: one mechanism, one UI per surface class

- `SignatureDialog` (modal) — signature fields, form completion, every version-control event.
  It mints the token itself, so its five call sites do not each reimplement minting.
- `SignatureConfirmPanel` (inline) — process-step transitions **and** the three equipment
  modals, which each carried a bespoke signature bar.
- `SignaturePasswordField` (mobile) — replaces three copy-pasted password inputs in
  `ScanFlowScreen`; mobile's dark shell cannot use the shadcn panel.

Form completion used to sign **silently from the session**, with no dialog at all. It now asks.

Password fields are deliberately hostile to password managers (`autoComplete='off'`,
`data-1p-ignore`): a credential replayed from a browser store was not executed by the signer
at the moment of signing, which is the entire point of re-entry.

`lib/signature.ts` and its tests are deleted — nothing imported them.

## Open

- The dual-signature e2e happy path (`kit assembly › a second signer can verify and complete
  the kit`) still needs a **second** fixture account (`TEST_VERIFIER_EMAIL` /
  `TEST_VERIFIER_PASSWORD`), because the service refuses one principal for both roles.
  Password signing makes that account mandatory rather than nice-to-have. See
  https://app.clickup.com/t/z8nrz7bygx
- Expired `SigningToken` rows need a sweep. The columns and indexes are there; the job is not.

## Related

- `2026-08-11-cgt-kit-lifecycle-two-writers-and-error-copy` — called the typed-name signature
  the weaker control on the regulated action; this entry is the reversal it asked for
- `2026-08-10-cgt-kitting-and-pick-list-foundations` — the kit dual-signature rule the
  principal-binding protects