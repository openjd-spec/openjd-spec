# Proposal Decision: `recommended_capabilities[]` field on Role and Job

**Disposition:** Accepted
**Origin:** [proposals/recommended-capabilities-field.md](../proposals/recommended-capabilities-field.md), from memo [`2026-05-17-1500--orgdef-strategist--roledef-strategist--recommended-capabilities-field-proposal.openthing`](../memos/2026-05-17-1500--orgdef-strategist--roledef-strategist--recommended-capabilities-field-proposal.openthing)
**Decided:** 2026-05-29 by roledef-strategist, on explicit Director direction ("We should definitely add the recommended_capabilities[]")
**Target version:** roledef SCHEMA v0.3.0

## Disposition

Accept the addition of one optional, additive **SHOULD** field — `recommended_capabilities[]` — to `roledef:Role` and `roledef:Job`, shipping as schema minor bump **v0.2.0 → v0.3.0**.

## Rationale

1. **Real use case in flight.** The Director's planned autonomous content-retrieval org needs its developer position's capability surface (browser automation, mailbox-reader, markdown-writer) declared on the artifact so the seat boots ready to work. The field is validated by a concrete deployment, not theory — consistent with the org's *empirical grounding* value.
2. **Correct altitude.** Capabilities are a role concern, not a slot concern. Declaring them on `roledef:Role`/`roledef:Job` and letting `orgdef:Position` inherit transitively keeps the single-source-of-truth on the role and requires **no orgdef schema change**.
3. **Additive and portable.** Optional field, reader-lenient resolution, `required` defaults false, unrecognized `kind` warns-not-fails — the design respects the *cross-runtime portability* value and the strict-writer/lenient-reader pattern (CA-002/CA-003).
4. **Promotes an already-anticipated item.** SCHEMA.md v0.2 "Future considerations" already listed *Capability declarations (`x.roledef.capabilities`)*. This decision promotes that from an envisioned extension to a first-class recommended field and marks the future-considerations item superseded.

## Modifications from the proposal-as-sketched

The originating memo left five shape decisions to strategist judgment. Resolved as follows (all matching the proposer's stated leans):

1. **`kind` enumeration** — enumerate well-known kinds (`mcp-server`, `claude-code-skill`, `url-resolver`) + `"other"` escape hatch. Informative, not closed-for-validity.
2. **`required`** — boolean, **defaults false**; `true` reserved for genuine hard dependencies and flagged as a portability constraint.
3. **Role/Job composition** — Job **appends to** Role; on shared `id`, **Job wins**. Matches additive-only derivation discipline.
4. **Capability `version`** — SHOULD use semver **ranges** (`"^1.0.0"`), not strict pins.
5. **Resolution** — spec declares, runtime resolves; unknown `kind` or unresolved non-`required` capability → WARN, continue.

One correction to the memo: the suggested schema number **"v1.2.0" was wrong** — it conflated the org charter's retrospective "v1 maturity" framing with the schema semver (actually v0.2.0). Per Director direction 2026-05-29, this lands as the ordinary additive minor bump **v0.3.0**. The schema's own v1.0.0 milestone remains a separate, future decision.

## Build directive (for the maintainer)

Accepted; the following implementation work is authorized (drafted in the same PR as this decision per the atomic-promotion discipline; final merge + version bump is the Director's ratification):

1. **SCHEMA.md** — bump title and footer v0.2.0 → v0.3.0; add `recommended_capabilities` to the top-level structure example; add a "Recommended fields (SHOULD)" subsection defining the field, sub-fields, `kind` enumeration, `required` semantics, Role/Job composition, resolution, and versioning; add SHOULD validation rules for Role and Job; add the field to the Role/Job differences note; add a v0.3.0 entry to the Versioning section's minor-bump line; mark the "Future considerations" *Capability declarations* item **superseded by v0.3.0**.
2. **README.md** — update the stale "schema version (currently 0.1.0)" reference to 0.3.0.
3. **Conformance fixtures** — specified in the proposal's *Conformance tests* section; authored as part of the broader conformance-suite buildout (evidence-first-pass), not blocking this decision.

## Cross-spec coordination

- **orgdef** — unaffected. orgdef:Position inherits capabilities transitively through role/job references; the independently-filed orgdef v1.1.0 proposal does not and need not include this field.
- **catdef / memodef** — substrate unchanged.
- **openbraid runtime** — a future capability-resolver subsystem is an engineer-track concern, no blocker.

## Notes

- Acknowledgment memo filed back to orgdef-strategist closing the loop: [`2026-05-29-1704--roledef-strategist--orgdef-strategist--recommended-capabilities-accepted.openthing`](../memos/2026-05-29-1704--roledef-strategist--orgdef-strategist--recommended-capabilities-accepted.openthing).
- The incoming proposal memo (2026-05-17) is now processed; `action_required` satisfied.
