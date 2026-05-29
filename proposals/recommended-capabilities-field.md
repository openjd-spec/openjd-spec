# Proposal: `recommended_capabilities[]` field on Role and Job

**Status:** Accepted (2026-05-29 by Director direction; decision at [decisions/proposal-recommended-capabilities.md](../decisions/proposal-recommended-capabilities.md))
**Target version:** roledef SCHEMA v0.3.0
**Author:** roledef-strategist (provisional bot identity), refining a direction proposed by orgdef-strategist
**Created:** 2026-05-29
**Origin:** Memo [`2026-05-17-1500--orgdef-strategist--roledef-strategist--recommended-capabilities-field-proposal.openthing`](../memos/2026-05-17-1500--orgdef-strategist--roledef-strategist--recommended-capabilities-field-proposal.openthing), carrying a Director-flagged strategic direction from a 2026-05-16 brainstorm on autonomous-org runtime architecture.

## Summary

Add one optional, additive field — `recommended_capabilities[]` — to `roledef:Role` and `roledef:Job`. It lets a role declare its **capability surface**: the tools, MCP servers, and runtime skills the role brings to a seat, so that a runtime instantiating a position bound to the role can pre-configure the seat-occupant's environment *from the artifact* rather than from out-of-band setup.

Director's framing example: "a marketing role could come with a PowerPoint skill." Generalizes broadly — a developer role with a Playwright skill, a QA role with a regression-checklist skill, an analyst role with a spreadsheet skill.

The field is optional and backward-compatible, so it ships as a minor version bump: **roledef SCHEMA v0.2.0 → v0.3.0**. (The originating memo suggested "v1.2.0"; that number conflated the org charter's "v1 maturity recognized retrospectively" framing with the *schema* semver, which is at v0.2.0. Per Director direction 2026-05-29, this lands as the ordinary additive minor bump v0.3.0; the schema reaches v1.0.0 on its own separate milestone.)

This proposal also **promotes and supersedes** the v0.2 "Future considerations" item *Capability declarations (`x.roledef.capabilities`)* (SCHEMA.md). What was anticipated as an extension is hereby elevated to a first-class recommended field.

## Motivation

### The concrete use case

The Director intends to charter an autonomous content-retrieval org (three positions: strategist, developer, QA) as a test-bed for OAGP autonomous-org primitives. The developer role's capability surface — Playwright, a mailbox-reader, a markdown-writer — is exactly the kind of declared surface this field enables. With v0.3.0 shipped, that position's opencatalog references its capabilities declaratively, and the developer seat **boots ready to work** instead of relying on the Director to hand-configure the dev environment.

This is not blocking the experiment (the Director can configure the first run manually), but it is a clean opportunity to land the field shape while a real use case validates it.

### Why this lives on roledef, not orgdef

Capabilities are a **role** concern (what this role has at its disposal), not a **slot** concern (where the role sits in an org chart). A marketing role has PowerPoint regardless of which org instantiates it. A developer role has Playwright whether it works at thingalog or YesterYacht.

`orgdef:Position`'s capability surface is therefore properly **derived** from its `role_definition` / `job_definition` references, not separately declared. **No orgdef schema change is required** — positions inherit capabilities transitively. If a specific org's specific position needs an extra capability beyond what the canonical role declares, the per-org `roledef:Job` specialization (which already overrides charter/identity/voice for that org) is the right place to add it.

## Proposed change

A new **SHOULD** field, valid on both `roledef:Role` and `roledef:Job`.

### Field shape

```json
"recommended_capabilities": [
  {
    "id": "playwright",
    "kind": "claude-code-skill",
    "reference": "playwright",
    "version": "^1.0.0",
    "purpose": "Drive a headless browser to retrieve and screenshot target pages.",
    "required": false
  },
  {
    "id": "openbraid",
    "kind": "mcp-server",
    "reference": "https://mcp.openbraid.app/<account>/<org>/<position>",
    "version": "^0.2.0",
    "purpose": "Read and send inter-position memos for the seat.",
    "required": false
  }
]
```

### Sub-field semantics

| Sub-field | Required? | Meaning |
|---|---|---|
| `id` | MUST | Short kebab-case identifier for the capability, unique within the array. |
| `kind` | MUST | One of the well-known kinds (below) or `"other"`. |
| `reference` | MUST | How a runtime locates the capability: a URL, an MCP server reference, or a skill name. |
| `version` | SHOULD | Semver **range** (e.g. `"^1.0.0"`), not a strict pin. |
| `purpose` | SHOULD | One line: why this role needs the capability. |
| `required` | OPTIONAL | Boolean; **defaults to `false`**. |

### `kind` enumeration (informative, with escape hatch)

The spec enumerates the well-known kinds and provides `"other"` for runtime-specific cases, mirroring the openbraid relationship-types appendix pattern (informative enumeration + `other` reads well across runtimes):

- `mcp-server` — an MCP server the seat connects to.
- `claude-code-skill` — a Claude Code skill resolved from the runtime's skill registry.
- `url-resolver` — a fetchable URL the role expects to dereference.
- `other` — anything not covered above. When `kind: "other"`, the entry SHOULD carry enough self-description (per the extension self-description SHOULD) that an unfamiliar runtime can act intelligently.

The enumeration is informative, not closed-for-validity: an unrecognized `kind` is a SHOULD-warning, not a hard validation failure, preserving cross-runtime portability.

### `required` defaults to false (portability guard)

`required: false` means degraded-mode operation is acceptable when the runtime cannot resolve the capability. `required: true` is a strong claim that the seat genuinely cannot function without it — and it **constrains cross-runtime portability**, because a runtime lacking the capability cannot host the role. Roles SHOULD default to graceful degradation; `required: true` SHOULD be reserved for genuine hard dependencies and SHOULD carry a `purpose` explaining the dependency.

### Role/Job composition

When a `roledef:Job` derives from a `roledef:Role` and both declare `recommended_capabilities[]`:

- The Job's array **appends to** the Role's (it does not replace it).
- If the same capability `id` appears in both, the **Job's entry wins** (the Job specialization overrides the Role's declaration for that id).

This matches the additive-only derivation discipline already established for guardrails and output_contract.

### Resolution (spec declares; runtime resolves)

The spec defines the field's shape and semantics; **runtimes resolve** the references:

- A Claude Code runtime resolves `kind: claude-code-skill` against its skill registry.
- An MCP-capable runtime resolves `kind: mcp-server` to MCP server configuration.
- A runtime that does not recognize a `kind`, or cannot resolve a non-`required` capability, **SHOULD log a warning and continue**, never fail. (Per the reader-lenient rule, CA-003.)

### Versioning of capability references

Capability `version` references SHOULD use semver **ranges** (`"^1.0.0"`) rather than strict pins (`"1.0.0"`). Strict pinning couples the role's version to the capability's patch cadence unnecessarily and is brittle; a range lets a capability patch-update without forcing a roledef bump.

## Backward compatibility

Fully additive. The field is OPTIONAL on both tiers:

- **Existing roledefs in the canonical library** — unaffected; none declare the field, all remain valid.
- **Existing roledefs in third-party libraries** — unaffected.
- **Old runtimes** — gracefully ignore the unknown field (reader-lenient).
- **Writer-strict stamping (CA-002)** — a roledef that uses `recommended_capabilities[]` MUST stamp `"roledef": "0.3.0"` or higher; roledefs that don't use it MAY keep their existing stamp.

No migration is required for any existing artifact.

## Conformance tests

The conformance suite is currently evidence-first-pass (empty `valid_roledefs/` / `invalid_roledefs/` scaffolding per [decisions/conformance-evidence-first-pass.md](../decisions/conformance-evidence-first-pass.md)). When the suite is populated, this field needs:

**`valid_roledefs/` fixtures:**
1. A Role declaring two capabilities of different `kind`s, `required: false`, with semver-range `version`.
2. A Role declaring a `kind: "other"` capability with self-describing metadata.
3. A Job that appends a capability to its parent Role and overrides one shared `id`.

**`invalid_roledefs/` fixtures (each one violation, with paired `.md`):**
1. A capability entry missing the MUST sub-field `id`.
2. A capability entry missing the MUST sub-field `kind`.
3. A capability entry missing the MUST sub-field `reference`.
4. Two capability entries sharing the same `id` within one array (id not unique).

**Behavioral (reader-lenient) checks:**
5. Unrecognized `kind` produces WARN, not FAIL.
6. Unresolved non-`required` capability produces WARN and the runtime continues.

These are specified in the decision's build directive for the maintainer; fixture authoring rides the broader conformance-suite buildout, not this PR.

## Alternatives considered

- **Keep it as an `x.roledef.capabilities` extension** (the v0.2 future-considerations shape). Rejected: the capability surface is broadly useful and cross-runtime — exactly the kind of thing the core spec should make first-class so receiving runtimes can act on it without prior knowledge of an adopter's extension. Promoting it to a recommended field is the right altitude.
- **Put the field on `orgdef:Position`.** Rejected: capabilities are a role concern, not a slot concern (see Motivation). Positions derive the surface transitively; declaring it on the position would denormalize and invite drift against the role.
- **`required` defaulting to true.** Rejected: breaks portability by construction. Default false; reserve true for genuine hard dependencies.
- **Strict version pinning.** Rejected: couples roledef versioning to capability patch cadence.
- **Job replaces (not appends to) Role's capabilities.** Rejected: inconsistent with the additive-only derivation discipline used elsewhere; append-with-id-override is the established shape.

## Open questions

None blocking. Two for future cycles:

- **`memodef:` / `transcriptdef:` capability kinds.** As the OAGP family's runtime story matures, well-known kinds may want first-class entries beyond the current three. Left to additive future minor bumps; `"other"` covers them in the interim.
- **Capability-resolver subsystem in openbraid.** A runtime-side concern (openbraid-engineer's track), not a spec concern. No blocker.
