# roledef Schema — v0.3.0

This document defines the structure and validation rules for **roledefs**: portable, machine-readable specifications of AI roles, expressed as catdef-compliant `.openthing` files.

roledef defines two artifact tiers:

- **Roles** (`roledef:Role`) — abstract role specifications, broadly applicable, published in canonical libraries
- **Jobs** (`roledef:Job`) — concrete instantiations of a role at a specific organization, carrying org-specific specializations

Together they describe a role completely enough that:

- A fresh AI runtime loaded with the appropriate role-or-job artifact reliably **becomes** that role-holder
- The behavior is reproducible across instantiations
- The behavior is portable across catdef-compliant runtimes (Claude, Grok, GPT, Gemini, etc.)
- The artifact can be forked, customized, versioned, and published independently

This schema is the **load-bearing artifact** for roledef. Everything else (the README, the seed library, the validator, the Claude Code skill) is built around it.

---

## Conformance language

This document uses the following terms with the meanings from RFC 2119:

- **MUST** / **MUST NOT** — required for conformance; absence or violation makes the roledef invalid
- **SHOULD** / **SHOULD NOT** — recommended; well-designed roledefs follow these but they are not required for validity
- **MAY** — permitted; an author's choice with no recommendation either way

The schema also uses catdef's `x.` extension namespace pattern. Fields prefixed with `x.<domain>.<identifier>` are extensions — defined by adopters, not by the roledef spec, and ignored by runtimes that don't recognize them.

---

## Substrate: catdef

A roledef is a catdef `.openthing` document. catdef provides the structural format; roledef defines the semantic shape inside that format. Every roledef artifact MUST:

- Be a valid catdef document (parseable as JSON conforming to catdef structural rules)
- Declare the catdef version it stamps under (`catdef` field, per CA-002 writer-strict stamping)
- Declare the roledef schema version it conforms to (`roledef` field)
- Have its `type` set to either `roledef:Role` (abstract role) or `roledef:Job` (concrete instantiation)

Beyond these constraints, roledef inherits catdef's full semantic toolkit:

- Field types (String, RichText, Enumerated, etc.)
- Polymorphic translatable fields (i18n via `.en` / `.fr` / `.context` / `.machine-translate`)
- Subcat enrichment for structured value namespaces
- Extension namespace (`x.<domain>.<identifier>`)
- Forward-compatibility rules (writer-strict, reader-lenient)
- Policy compliance (value #9: declared policies are conformance requirements)

A roledef is therefore a specialization of an `.openthing` for the role-description content type.

---

## Roles vs Jobs

roledef defines **two distinct artifact tiers** that share substantial structure but serve different purposes.

### Role (`roledef:Role`)

An **abstract role specification** — broadly applicable, content-domain-specific but org-neutral. Examples:

- `senior-open-standards-strategist` — applicable to any open-standards stewardship context
- `blackhat-tester` — applicable to any security-testing context
- `roledef-validator` — applicable to any roledef ecosystem context

Roles live in canonical libraries (the reference library at `github.com/roledef-spec/roledef`, mirror at `roledef.org`, or any third-party library using the same schema). One role can be instantiated by many jobs across many orgs.

### Job (`roledef:Job`)

A **concrete instantiation of a role at a specific organization**, carrying org-specific specializations. Examples:

- `catdef-spec/jobs/strategist.openthing` — the catdef-strategist job at catdef-spec, derived from `senior-open-standards-strategist`
- `acme/jobs/payments-junior-dev.openthing` — Acme's Junior Developer job for the payments team, derived from a generic `senior-software-developer` role

Jobs live in **the working repo of the org that defines them**, at `<project>/<org-name>-org/jobs/<job-id>.openthing` (or `<project>/jobs/<job-id>.openthing` for orgs without a nested-org-folder convention). Public projects publish jobs in public repos; private orgs keep them in private repos. Authorization to read and modify a job artifact is implied by repo access.

### The three-tier graph

A position in an org chart references a job; the job references a role:

```
orgdef:Position
  └─ position.job_definition → roledef:Job
                                 └─ metadata.role_definition → roledef:Role
```

`orgdef:Position` is defined by the [orgdef](https://github.com/orgdef-spec/orgdef) sibling spec. Positions are the staffable slots in an org chart. One job can be referenced by multiple positions (e.g., three "Junior Dev Team A" positions all filling the same `acme/jobs/junior-dev-team-a.openthing` job; team-specific specializations live at the Job tier; the Position tier handles per-seat staffing).

For backward compatibility with v0.1, `orgdef:Position` MAY also use `position.role_definition` pointing directly at a Role, interpreted as "this position uses the abstract role with no org-specific specializations." Most positions in real use will have `position.job_definition`.

### When to use Role vs Job

- **Abstract methodology, broadly applicable** → Role. Lives in canonical or third-party library.
- **Org-specific specialization with charter and placement** → Job. Lives in owning org's working repo.
- **Same role, multiple jobs in the same org** (e.g., three junior-dev jobs across three teams) → one Role, multiple Jobs.
- **Same job, multiple positions** (e.g., three identical seats filling that job) → one Job, multiple Positions.

### Schema differences between Role and Job

Most fields are common to both tiers. The differences are concentrated in metadata and a small number of Job-specific fields:

| Field | Role | Job |
|-------|------|-----|
| `type` | `"roledef:Role"` | `"roledef:Job"` |
| `id` | role id (e.g., `senior-open-standards-strategist`) | job id (e.g., `catdef-strategist`) |
| `charter` (top-level) | not used | RECOMMENDED — single statement of the stable slot of responsibility this Job fills |
| `metadata.role_definition` | not used | REQUIRED — pointer to the abstract Role this Job instantiates |
| `metadata.org_definition` | not used | REQUIRED — pointer to the orgdef this Job exists in |
| `metadata.placement` | not used | OPTIONAL/RECOMMENDED — denormalized snapshot of org-chart placement |
| `metadata.derived_from` | OPTIONAL — Role-to-Role lineage | OPTIONAL — Job-to-Job lineage (rare; for jobs that derive from other jobs) |

All other fields (`identity`, `voice`, `output_contract`, `guardrails`, `description`, `conversation_rules`, `workflow`, `reaction_style`, `design_constraints`, `recommended_capabilities`, `examples`) apply uniformly to both tiers.

One field has tier-specific *composition* semantics worth noting here: `recommended_capabilities` (v0.3.0) is declared on a Role and MAY be extended by a derived Job — the Job's entries append to the Role's, and on a shared capability `id` the Job's entry wins. See [`recommended_capabilities`](#recommended_capabilities-array-of-objects) below.

### Authorization-implies-access

A Job artifact's repo location is its access-control mechanism. Public-repo jobs are readable by anyone; private-repo jobs are readable only by entities with repo access. There is no separate `x.job.commercial_sensitivity` flag — repo visibility IS the flag. Validators MAY skip reference-resolution checks for Job artifacts whose URL pattern indicates a private repo (with WARN, not FAIL).

---

## sample-jobs/ directory

The canonical roledef library at `github.com/roledef-spec/roledef` includes a `sample-jobs/` directory hosting **sample / canonical Job artifacts for demonstration purposes**. These are not operational jobs (no live org-holder occupies them); they document patterns and serve as adopter starting-points.

```
roledef-spec/roledef/
├── roledefs/         ← abstract Roles (canonical library, primary content)
├── sample-jobs/      ← canonical sample Jobs (demonstration, not operational)
└── ...
```

The bootstrap-era artifacts that are conceptually Job-shaped (catdef-strategist, roledef-strategist, sncro-blackhat-tester, senior-jaded-vc-associate) are mirrored as samples in `sample-jobs/` while operational copies live in their respective owning project repos.

Sample jobs MAY omit fields that operational jobs SHOULD include (e.g., `charter`, `metadata.placement`, `metadata.org_definition` may point at illustrative-only orgdefs).

---

## File extension

A roledef is stored as a single `.openthing` file:

```
senior-slightly-jaded-vc.openthing
catdef-strategist.openthing
roledef-contributor.openthing
```

Multiple roledefs can be aggregated in a `catalog.opencatalog` index, but each individual roledef is a standalone `.openthing` file. This allows independent forking, citation, and distribution.

---

## Top-level structure

```json
{
  "catdef": "1.4",
  "roledef": "0.1.0",
  "type": "roledef:Role",
  "id": "<short-identifier>",
  "name": "<human-readable name>",
  "version": "<semver>",
  
  "identity": "...",
  "voice": "...",
  "output_contract": [...],
  "guardrails": [...],
  
  "description": "...",
  "conversation_rules": [...],
  "workflow": {...},
  "reaction_style": {...},
  "design_constraints": [...],
  "recommended_capabilities": [...],
  "examples": [...],
  "metadata": {...},
  
  "x.<domain>.<identifier>": ...
}
```

---

## Required fields (MUST)

Every roledef MUST have all of the following fields, with non-empty values.

### `catdef` (string, semver)

The catdef version the roledef stamps under. Per CA-002, the writer MUST declare the minimum catdef version that defines every feature used.

```json
"catdef": "1.4"
```

### `roledef` (string, semver)

The roledef schema version this roledef conforms to. Used by validators to apply the correct schema rules.

```json
"roledef": "0.2.0"
```

### `type` (string, fixed value)

MUST be exactly one of:

- `"roledef:Role"` for an abstract role specification
- `"roledef:Job"` for a concrete org-specific instantiation

Distinguishes roledef `.openthing` files from other `.openthing` types and selects which tier-specific validation rules apply.

```json
"type": "roledef:Role"
```

```json
"type": "roledef:Job"
```

### `id` (string)

Short, kebab-case identifier for the roledef. Used in filenames, URLs, and cross-references. MUST be unique within its publishing namespace (e.g., within `github.com/roledef/roledefs/`).

```json
"id": "senior-slightly-jaded-vc"
```

### `name` (string or polymorphic)

Human-readable name of the role. May be a plain string or a polymorphic translatable field (per catdef i18n).

```json
"name": "Senior, Slightly Jaded Silicon-Valley Associate VC"
```

### `version` (string, semver)

The roledef's own version (separate from the schema version). Authors version their roledefs as they refine them.

```json
"version": "1.0.0"
```

### `identity` (RichText or polymorphic)

Who this role IS. A paragraph (or several) that captures the persona: background, perspective, character. This is what makes the role recognizable across instantiations. Without identity, you have a set of rules, not a role.

```json
"identity": "You are DangerStorm — a confident, direct, product-savvy AI that helps people turn product ideas into professional pitch deck prompts in under 90 seconds. You talk like a senior product manager who's evaluated a thousand ideas and knows instantly what makes one work."
```

### `voice` (RichText or polymorphic)

How this role talks. Tone, register, characteristic phrasing, signature moves. The voice MUST be specific enough that a reader (or fresh AI loaded with the roledef) can answer "would the role say it this way?" reliably.

```json
"voice": "Confident, direct, excited when you see a great angle, and willing to push back when something is vague. Short conversational responses (1-3 sentences before asking the next question). Paraphrase to confirm. Show enthusiasm at the right moments. Push back without softening when the user is being vague."
```

### `output_contract` (array of objects)

What the role produces. Every roledef MUST declare at least one output. Each output specifies:

- `name` — short identifier for this output
- `description` — what the output is, in plain language
- `format` — structural format (markdown, json, plain text, structured-with-markers)
- `schema` — content schema or template (free-form description, or formal schema reference)
- `length_guidance` (optional) — soft constraints on length
- `examples` (optional) — sample outputs

```json
"output_contract": [
  {
    "name": "deck_prompt",
    "description": "A complete prompt that another AI can execute to produce an 8-slide pitch deck.",
    "format": "markdown",
    "schema": "Slide 1: TITLE — product name, domain, tagline, attribution. Slide 2: PROBLEM — pain, who feels it, why unsolved. Slide 3: SOLUTION — what it does, the KEY INSIGHT. Slide 4: HOW IT WORKS — 3-4 user-experience steps. Slide 5: WHO BUYS IT — primary and secondary audiences. Slide 6: REVENUE MODEL — pricing, unit economics. Slide 7: STATUS & PROOF — validation, what's needed next. Slide 8: CLOSING — product, pitch, contact.",
    "length_guidance": "One prompt block, structured so a downstream AI can produce all 8 slides in one response."
  }
]
```

### `guardrails` (array of strings or objects)

Explicit MUST-NOT behaviors. Every roledef MUST declare at least one guardrail. Guardrails are the role's hard limits — what it will refuse to do regardless of input.

```json
"guardrails": [
  "Never break character.",
  "Never show the question sequence as a list to the user.",
  "Never bundle multiple questions in one response.",
  "Keep conversational responses to 1-3 sentences before the next question.",
  "Never truncate or skip outputs when generating the final deliverables."
]
```

Guardrails MAY be expressed as objects with additional metadata (severity, rationale, related-tests) for richer validation:

```json
"guardrails": [
  {
    "rule": "Never break character.",
    "severity": "critical",
    "rationale": "The role's value depends on consistent persona across the entire interaction."
  }
]
```

---

## Job-specific required fields (MUST for `type: "roledef:Job"`)

Artifacts with `type: "roledef:Job"` MUST include the following metadata fields **in addition to** all the common required fields above.

### `metadata.role_definition` (object, MUST)

A pointer to the abstract `roledef:Role` this Job instantiates. Object with `id`, `version`, and `url`:

```json
"metadata": {
  "role_definition": {
    "id": "senior-open-standards-strategist",
    "version": "1.0.0",
    "url": "https://roledef.org/roledefs/senior-open-standards-strategist.openthing"
  }
}
```

The `id` and `version` together pin the Job's role-side derivation to a specific snapshot of the parent Role. The `url` provides a fetchable reference. A Job's specializations are additive over the pinned Role per the conventions for safe derivation (see `metadata.derived_from`); the additive-only invariant is verifiable against the pinned Role.

### `metadata.org_definition` (object, MUST)

A pointer to the `orgdef:Organization` this Job exists in. Object with `id`, `version`, and `url`:

```json
"metadata": {
  "org_definition": {
    "id": "orgdef-spec-org",
    "version": "0.1.0",
    "url": "https://github.com/orgdef-spec/orgdef/blob/main/orgdef-spec-org/orgdef.openthing"
  }
}
```

The `org_definition` URL MAY point at a private-repo orgdef. Validators MAY skip resolution checks for private orgdefs (WARN, not FAIL).

A Job that does not yet have a concrete org context (e.g., a sample job in the canonical library) MAY use a placeholder org_definition pointing at an illustrative-only orgdef.

---

## Recommended fields (SHOULD)

Well-designed roledefs SHOULD include the following fields. They are not required for validity but are strongly recommended for Turing-test fidelity.

### `description` (string)

One-sentence summary of the role. Used in catalog indexes, search results, and quick-reference contexts.

```json
"description": "Confident, direct, product-savvy interviewer who turns rough product ideas into professional pitch deck prompts in under 90 seconds."
```

### `charter` (RichText or polymorphic) — Job-only, RECOMMENDED for operational Jobs

A single statement of the **stable slot of responsibility** this Job fills. Distinguishes one Job from another when both derive from the same parent Role.

```json
"charter": "Holds delegated authority for orgdef-spec design decisions: library curation, scope narrowing, pattern promotion, MUST/SHOULD calls, and architectural deliberation. The stable slot of responsibility is 'orgdef as a credible open standard within the OAGP family.'"
```

Format guidance: prose, 1–4 sentences. Polymorphic translatable form per catdef. Should articulate *what stable purpose this Job serves*, not what tasks the role-holder performs (those belong in the parent Role's `output_contract` and `workflow`).

`charter` is OPTIONAL for `type: "roledef:Role"` artifacts (Roles are abstract; they don't have a charter — their derivations do). Sample Jobs in the canonical library MAY omit `charter` if they exist purely as illustrative templates.

### `conversation_rules` (array of strings or objects)

How the role interacts. Distinct from `guardrails` (hard MUST-NOTs) — these are interaction patterns the role follows.

```json
"conversation_rules": [
  "Always ask exactly one question per response.",
  "Be aggressive about extracting info from previous answers — skip questions whose answers are already implied.",
  "Maximum 5 total exchanges (including opener); often 3-4 is enough.",
  "After each user reply: acknowledge/react conversationally, then either ask one next question OR generate all outputs if you have enough."
]
```

### `workflow` (object)

The steps the role takes, with sequencing logic. Captures the role's procedural behavior — what it does in what order.

```json
"workflow": {
  "type": "sequence",
  "steps": [
    {
      "step": "opener",
      "action": "Ask for elevator pitch + domain together.",
      "skip_conditions": []
    },
    {
      "step": "primary_user",
      "action": "Ask who buys it / who uses it.",
      "skip_conditions": ["clear from pitch"]
    },
    {
      "step": "differentiator",
      "action": "Ask for the one key insight that makes it different.",
      "skip_conditions": [],
      "notes": "Almost always ask — this is the heart of the deliverable."
    }
  ],
  "termination": "When you have enough info (typically 3-5 exchanges), generate ALL outputs in one response."
}
```

Workflow is the most expressive recommended field. roledefs with rich workflows are more reliably reproducible tha roledefs without.

### `reaction_style` (object)

How the role reacts to inputs, with examples. Few-shot patterns that anchor the role's tone in concrete moves.

```json
"reaction_style": {
  "patterns": [
    "Paraphrase to confirm understanding before moving on.",
    "Push back if vague — don't accept generic answers.",
    "Show genuine excitement when the user lands on a sharp insight.",
    "If they give a URL, identify the comparison and the differentiator quickly."
  ],
  "examples": [
    {
      "context": "Confirming understanding",
      "example": "OK, so [domain] — [what it does]. I like the angle."
    },
    {
      "context": "Pushing back on vague answer",
      "example": "That's still pretty broad. Nail the one thing it does better than anything else. What's that?"
    },
    {
      "context": "Reacting to a sharp insight",
      "example": "Boom — that's the insight nobody else has. That's your Slide 3 money."
    }
  ]
}
```

The `examples` array is **the most important Turing-test signal**. Concrete examples let validators check whether a roledef-instantiated runtime produces equivalent reactions to the original.

### `design_constraints` (array of strings or objects)

Quality standards the role's outputs MUST meet. Distinct from `guardrails` (behavioral MUST-NOTs) — these are output-quality constraints.

```json
"design_constraints": [
  "Output 1 (deck prompt) must specify a bold color palette appropriate to the product category, not generic blue.",
  "Output 1 must specify dark title and closing slides, light content slides (sandwich structure).",
  "Output 1 must specify clean typography (Trebuchet MS or Georgia for headers, Calibri for body).",
  "Output 1 must specify that each slide has a visual element — no bullet-points-on-white-background slides.",
  "Output 1 must specify 16:9 format."
]
```

### `recommended_capabilities` (array of objects)

*Introduced in v0.3.0.* The role's **capability surface** — the tools, MCP servers, and runtime skills the role brings to a seat. A runtime that instantiates a position bound to this role MAY use this field to pre-configure the seat-occupant's environment *from the artifact*, rather than relying on out-of-band setup. Applies to both `roledef:Role` and `roledef:Job`.

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

Each entry has the following sub-fields:

| Sub-field | Conformance | Meaning |
|---|---|---|
| `id` | MUST | Short kebab-case identifier for the capability, unique within the array. |
| `kind` | MUST | One of the well-known kinds (below) or `"other"`. |
| `reference` | MUST | How a runtime locates the capability: a URL, an MCP server reference, or a skill name. |
| `version` | SHOULD | Semver **range** (e.g. `"^1.0.0"`), not a strict pin. |
| `purpose` | SHOULD | One line: why this role needs the capability. |
| `required` | OPTIONAL | Boolean; **defaults to `false`**. |

#### `kind` enumeration

The spec enumerates well-known kinds and provides an `"other"` escape hatch (informative enumeration, not closed-for-validity):

- `mcp-server` — an MCP server the seat connects to.
- `claude-code-skill` — a Claude Code skill resolved from the runtime's skill registry.
- `url-resolver` — a fetchable URL the role expects to dereference.
- `other` — anything not covered above. When `kind: "other"`, the entry SHOULD carry enough self-description (per the [extension self-description SHOULD](#extension-self-description-should)) that an unfamiliar runtime can act intelligently.

An unrecognized `kind` is a SHOULD-warning, **not** a hard validation failure — preserving cross-runtime portability.

#### `required` semantics (portability guard)

`required: false` (the default) means degraded-mode operation is acceptable when the runtime cannot resolve the capability. `required: true` is a strong claim that the seat genuinely cannot function without the capability — and it **constrains cross-runtime portability**, because a runtime lacking it cannot host the role. Roles SHOULD default to graceful degradation; `required: true` SHOULD be reserved for genuine hard dependencies and SHOULD carry a `purpose` explaining the dependency.

#### Role/Job composition

When a `roledef:Job` derives from a `roledef:Role` and both declare `recommended_capabilities[]`:

- The Job's array **appends to** the Role's (it does not replace it).
- If the same capability `id` appears in both, the **Job's entry wins** (the Job specialization overrides the Role's declaration for that id).

This matches the additive-only derivation discipline established for `guardrails` and `output_contract`.

#### Resolution (spec declares; runtime resolves)

The spec defines the field's shape and semantics; **runtimes resolve** the references. A Claude Code runtime resolves `kind: claude-code-skill` against its skill registry; an MCP-capable runtime resolves `kind: mcp-server` to MCP server configuration. A runtime that does not recognize a `kind`, or cannot resolve a non-`required` capability, **SHOULD log a warning and continue — never fail** (reader-lenient, per CA-003).

#### Versioning of capability references

Capability `version` references SHOULD use semver **ranges** (`"^1.0.0"`) rather than strict pins (`"1.0.0"`). Strict pinning couples the role's version to the capability's patch cadence and is brittle; a range lets a capability patch-update without forcing a roledef bump.

`recommended_capabilities` is OPTIONAL on both tiers. A roledef that uses it MUST stamp `"roledef": "0.3.0"` or higher (writer-strict per CA-002).

### `examples` (array of objects)

Sample interactions or sample outputs. Useful for Turing-test fixtures and for human implementers learning the role's texture.

```json
"examples": [
  {
    "type": "sample_interaction",
    "user_input": "I have an idea for an AI tool that helps photographers date old photos.",
    "role_response": "OK, so consumer photo tools — AI photo dating for scanned family archives. I like the angle. Who's the primary user — collectors? archivists? family historians?"
  }
]
```

### `metadata` (object)

Authorship, licensing, attribution, version history, related roledefs, and lineage.

```json
"metadata": {
  "authors": ["catdef-strategist@catdef.org", "scottconfusedgorilla@github"],
  "license": "MIT",
  "extracted_from": "DangerStorm v1.0 (theborrowedsoul.com)",
  "created": "2026-04-25",
  "extends": null,
  "derived_from": {
    "id": "blackhat-tester",
    "version": "1.0.0",
    "url": "https://roledef.org/roledefs/blackhat-tester.openthing"
  },
  "related": ["roledef-contributor", "patient-senior-editor"],
  "homepage": "https://roledef.org/roledefs/senior-slightly-jaded-vc",
  "repository": "https://github.com/roledef-spec/roledef/blob/main/roledefs/senior-slightly-jaded-vc.openthing"
}
```

#### `derived_from` (SHOULD when applicable)

When a roledef is a specialization of another roledef (the "abstract → derived" pattern), it SHOULD declare its lineage via the `derived_from` field. The field is **purely declarative** — runtimes do NOT resolve it as inheritance. It exists for:

- **Lineage transparency**: readers of a derived roledef can navigate to the parent and assess what the specialization changed
- **Validation chain**: validators can follow `derived_from` to verify the parent exists and check its conformance status
- **Library curation**: maintainers can identify abstract roles that have many specializations vs. one-off custom roles

The recommended form is an object with `id`, `version`, and `url`:

```json
"derived_from": {
  "id": "blackhat-tester",
  "version": "1.0.0",
  "url": "https://roledef.org/roledefs/blackhat-tester.openthing"
}
```

The `id` and `version` together pin the derivation to a specific snapshot of the parent. The `url` provides a fetchable reference. Multiple derived_from entries (an array of objects) are permitted for roles that compose from multiple parents, though composition semantics are not yet formalized — for v0.1, prefer single inheritance.

**Inheritance is git-fork-based, not runtime-merged.** A derived roledef is a complete, self-contained roledef — not a delta against the parent. The author copies the parent file (typically by git fork, but any copy works) and modifies as needed. The runtime treats the derived roledef as a standalone artifact; no parent fetch happens at instantiation time. This keeps the runtime simple and avoids merge-semantics complexity.

**Conventions for safe derivation** (additive-only principles, intended to make `derived_from` declarations meaningful for trust-chain navigation):

- **Guardrails:** derived SHOULD be additive only. Do not remove or relax parent guardrails. Adding new guardrails is fine.
- **Voice and identity:** derived MAY refine (more domain-specific, more specialized register) but SHOULD NOT contradict parent essence. A derived role of "slightly-jaded-VC" should not become "warmly-enthusiastic-VC."
- **Output contract:** derived MAY add entries; MAY tighten schema constraints on existing entries; SHOULD NOT remove or loosen existing entries.
- **Workflow:** derived MAY replace parent workflow entirely (different domain, different sequence); when this happens, the `derived_from` declaration is more about lineage of methodology than literal procedural inheritance.
- **Reaction style examples:** derived SHOULD preserve parent examples and add specialization examples; do not remove parent examples (they remain valid behavioral anchors).

These conventions are SHOULDs, not MUSTs — domain-specific specializations may have legitimate reasons to deviate. But authors who follow them produce derivations that downstream readers can trust to inherit the parent's validation properties.

#### `metadata.placement` (object, Job-only, OPTIONAL/RECOMMENDED)

A **denormalized snapshot** of where this Job sits in its org chart. Used by an AI session bootstrapping into the Job to know its structural context (who to escalate to, who to coordinate with, what to direct) without graph-walking the orgdef.

```json
"metadata": {
  "placement": {
    "reports_to":       "<job-id or position-id, optional>",
    "directs":          ["<job-id>", "..."],
    "coordinates_with": ["<job-id>", "..."]
  }
}
```

| Sub-field | Cardinality | Purpose |
|---|---|---|
| `reports_to` | zero-or-one | The Job/Position this Job escalates to |
| `directs` | zero-or-many | Jobs/Positions this Job has authority over |
| `coordinates_with` | zero-or-many | Peer Jobs (often in sibling orgs) this Job collaborates with |

All sub-fields are individually optional. `metadata.placement` itself is OPTIONAL but RECOMMENDED for any operational Job (any Job an AI session will actually be instantiated into).

**SSoT rule.** When `metadata.placement` exists alongside `orgdef.relationships` (the org-side org-chart expression), **`orgdef.relationships` is authoritative**. `metadata.placement` is a denormalized snapshot maintained at Job-authoring time and kept in sync going forward.

Validators SHOULD verify consistency between `metadata.placement` and the referenced `orgdef.relationships` when both are accessible (public-public case). Validators MAY skip the consistency check with WARN (not FAIL) when one or both reside in private repos that the validator cannot resolve.

References use the same shape as `metadata.role_definition` (id + version + url) where pinning matters; bare ids are acceptable for intra-org references.

`metadata.placement` is OPTIONAL on `type: "roledef:Role"` artifacts (Roles are abstract; placement applies to derivations). Sample Jobs MAY include placement as exemplars.

#### `metadata.opd_frequency_days` (integer, OPTIONAL)

The role's Ongoing Professional Development (OPD) frequency in days. Default: 7. Roles MAY tighten (e.g., 1 day for fast-moving roles) or loosen (e.g., 30 days for stable roles). Derivations MAY override the parent's frequency. See [Ongoing Professional Development (OPD)](#ongoing-professional-development-opd) below for the full pattern.

```json
"metadata": {
  "opd_frequency_days": 7
}
```

This field applies to roledefs with `workflow.type: "engagement_loop"` that include the OPD pattern. Sequence-typed roledefs MAY include the field but it is informational only (sequence-typed roles are typically transactional and do not run OPD; see OPD section).

---

## Ongoing Professional Development (OPD)

**Status:** SHOULD-pattern, primarily for `workflow.type: "engagement_loop"` roles.

Roles with `metadata.derived_from` set face a drift problem: the derivation is forked at a pinned parent version, and the parent evolves over time. Without periodic re-fork, the derivation falls behind. Multi-tier chains compound this (Role → derived Role → Job → ... — each layer accumulates staleness).

Ongoing Professional Development (OPD) solves the drift problem **behaviorally** — by making "stay current with your parent role" part of every role's own job description. The role-holder periodically (default at most every 7 days) checks `metadata.derived_from`, fetches the parent at its current version, diffs against the version this role was derived from, analyzes whether changes are material, and prompts the overseer with a memo recommending whether to incorporate updates.

This is the AI equivalent of continuing professional education. The mechanism reuses existing fetch capability and the catdef-org memo convention; no new infrastructure required.

### Scope: engagement_loop roles primarily

OPD's mechanism is workflow-phase-shaped — designed to plug into `workflow.type: "engagement_loop"` roles where the role-holder runs an ongoing engagement (catdef-strategist, roledef-strategist, blackhat-tester-derivations, etc.).

`workflow.type: "sequence"` roles are transactional/one-shot — a session runs the sequence to completion and exits. There is no continuing role-holder relationship to maintain currency for. **Sequence-typed roles SHOULD skip OPD.** Validators MUST NOT warn about missing OPD on sequence-typed roles. If a sequence-typed role's parent evolves and the derivation needs to keep pace, the appropriate mechanism is a manual fork-update by the maintainer.

### When `metadata.derived_from` is null

OPD is a no-op for roles with `metadata.derived_from: null` — no parent means nothing to track. Abstract roles MAY still include OPD in their workflow / conversation_rules / output_contract; future derivations of the abstract role will inherit OPD via the additive-only invariant.

### The four components

#### 1. Workflow phase

Add a recognized phase to the role's `workflow` as a flat sibling key alongside the existing engagement_loop phases (`engagement_setup`, `submission_decision`, etc.):

```json
"workflow": {
  "type": "engagement_loop",
  "engagement_setup": [...],
  "submission_decision": [...],
  "atomic_promotion": [...],
  "pattern_promotion": [...],
  "ongoing_professional_development": [
    "Check metadata.derived_from. If null, skip OPD entirely.",
    "Read the OPD state file at <working-repo>/opd-state/<role-id>.json (create on first run with last_check timestamp = now).",
    "If less than the role's metadata.opd_frequency_days days have elapsed since last_check, skip OPD until next session.",
    "Otherwise: fetch the current version of metadata.derived_from.url. Compare against the version recorded in metadata.derived_from.version (the fork-time version).",
    "Diff the parent at fork-time-version vs current-version. Identify changes affecting any field this role inherits or could be impacted by.",
    "Categorize changes: (a) material (would change this role's behavior or output if adopted), (b) cosmetic (no behavioral impact), (c) inapplicable (parent change targets a domain this role doesn't operate in).",
    "If material changes exist: draft an opd_review_memo to the overseer with parent identity, fork version, current version, categorized diff, and recommendation (incorporate / partially incorporate / decline / defer).",
    "Update the OPD state file: last_check = now, last_known_parent_version = parent's current version.",
    "If no material changes: still update last_check timestamp; no memo needed (or send an informational 'no material drift' memo at the overseer's preference).",
    "Continue with the engagement after dispatching the OPD memo; do not block on overseer response."
  ],
  "engagement_close": [...]
}
```

The `ongoing_professional_development` key is a flat sibling — engagement_loop workflows do NOT use a `phases` wrapper. This matches the existing convention established by `senior-open-standards-strategist` v1.0.0's workflow shape.

#### 2. Conversation rule

Add to the role's `conversation_rules` array:

> "At session start, after reading the operating manual and memory file, run the OPD check (per workflow.ongoing_professional_development). The check is fast when no review is due (state-file timestamp comparison only). When a review IS due, run it before substantive work; the overseer's response is asynchronous."

#### 3. Output contract entry

Add to the role's `output_contract` array:

```json
{
  "name": "opd_review_memo",
  "description": "A memo to the role's overseer (per the org's reports_to relationship from the orgdef artifact) summarizing parent-role changes since this role was derived, categorizing changes as material/cosmetic/inapplicable, and recommending whether to incorporate updates. Issued at most every metadata.opd_frequency_days days when material changes exist.",
  "format": "memodef:Memo (or legacy x.memo.* form on a catdef:Thing)",
  "schema": "Required fields: from (this role's id), to (overseer's position id from the orgdef), subject ('OPD review: <parent-id> v<fork-version> -> v<current-version>'), sent (ISO 8601), body (markdown). Body sections: (1) Parent identity + URLs; (2) Versions (fork-time and current); (3) Material changes (per-field diff with intent); (4) Cosmetic changes (brief mention; no action recommended); (5) Inapplicable changes (changes that touch domains this role doesn't operate in); (6) Recommendation (incorporate all / incorporate selected / decline / defer with reason); (7) If recommending adoption, the proposed new derivation version (semver bump rationale).",
  "destination": "Overseer's working repo at <overseer.x.position.working_directory>/memos/ per the catdef-org memo convention. Filename: YYYY-MM-DD-HHMM--<this-role>--<overseer>--opd-review-<parent-id>.openthing"
}
```

#### 4. Per-role-instance state file

Each role-instance carries a small JSON state file at `<working-repo>/opd-state/<role-id>.json`:

```json
{
  "role_id": "catdef-strategist",
  "derived_from": {
    "id": "senior-open-standards-strategist",
    "url": "https://roledef.org/roledefs/senior-open-standards-strategist.openthing"
  },
  "fork_version": "1.0.0",
  "last_check": "2026-04-27T08:00:00Z",
  "last_known_parent_version": "1.0.0",
  "opd_frequency_days": 7
}
```

The state file lives in the role-holder's working repo (per authorization-implies-write-access — the role-holder writes the state file, just like memos). Created lazily on first OPD check; updated on every check. Committed to the working repo for cross-machine portability.

### OPD failure handling

When the parent URL fails to resolve:

- **Transient failures** (network blip, momentary 503): silently skip OPD this cycle and try next time
- **Persistent failures** (3+ consecutive failed checks): send an informational memo to the overseer noting the resolution failure and asking for guidance (URL changed? Repo moved? Credentials needed?)
- **MUST NOT FAIL** the session — OPD is a SHOULD-pattern; resolution failure is informational, not blocking

### Decision authority

The agent's OPD output is a **recommendation** to the overseer. The overseer makes the incorporation decision. The agent does NOT auto-adopt parent changes. This preserves the bounded-authority discipline: the agent reviews and recommends; the overseer (typically the human steward, but possibly a more-senior AI role per the org's `reports_to` chain) decides.

### Self-bootstrap (prospective only)

Once a parent role gets OPD, **future derivations forked from that point onward inherit OPD automatically** via the additive-only invariant. They will run OPD on their next session start.

**Existing pre-OPD derivations do NOT self-bootstrap.** A derivation forked from a pre-OPD version of the parent has no OPD code paths and will never run an OPD check to discover the parent's new OPD obligation. Migration of existing derivations to OPD requires a coordinated v-bump as part of the parent's OPD-adoption rollout — performed by the maintainer, not by the derivation's own self-update.

---

## Extension fields (x. namespace)

roledefs MAY include any number of extension fields under the `x.<domain>.<identifier>` namespace, per catdef's extension convention. Extensions are domain-specific and not governed by the roledef spec.

### Examples of extensions

Different domains may need fields the roledef spec doesn't anticipate. Some plausible extensions:

```json
"x.intimate.consent_protocols": [...],
"x.intimate.boundary_phrases": [...],

"x.legal.disclosure_requirements": [...],
"x.legal.bar_admission_required": ["NY", "CA"],

"x.medical.diagnostic_authority_bounds": "advisory only — never offers diagnoses",

"x.security.escalation_path": "...",
"x.security.severity_thresholds": {...},

"x.financial.trading_authority_bounds": {"max_position_usd": 10000, "asset_classes": ["equities"]},

"x.creative.style_references": [...],
"x.creative.brand_guidelines_url": "..."
```

Extensions allow domain communities to enrich roledefs without coordinating with roledef governance. The roledef spec is intentionally minimal; domains extend as needed.

### Extension self-description (SHOULD)

Each extension SHOULD carry sufficient self-description that a naive agent — one that has never encountered this specific extension before — can handle it intelligently. The recommended pattern is to express the extension as an object with at least a `description` field:

```json
"x.intimate.consent_protocols": {
  "description": "Explicit consent verification phrases the role uses before specific topics. Naive agents: surface to user, do not act on autonomously.",
  "value": ["..."]
}
```

When the extension's name is fully self-evident and the value is structurally obvious, a bare value is acceptable:

```json
"x.security.severity_thresholds": {"critical": 9, "high": 7}
```

Richer extensions MAY include additional metadata to help unfamiliar agents act correctly:

```json
"x.legal.bar_admission_required": {
  "description": "Jurisdictions where this role can offer practice-specific advice.",
  "value": ["NY", "CA"],
  "schema": "https://roledef.org/extensions/legal/bar-admission",
  "agent_handling_advice": "If the user's location is outside this list, surface a warning rather than offering jurisdiction-specific advice."
}
```

**Why this matters:** the OAGP architecture assumes catdef artifacts move between runtimes that may not share prior context. A roledef authored by Acme Corp's domain experts may travel to a runtime that's never seen Acme's extensions. If the extensions are opaque, the receiving runtime either ignores them (losing information) or refuses to process the roledef (losing the role). If the extensions are self-describing, the receiving runtime can act intelligently — surface to user, apply the advice, defer to the roledef's own guidance.

This is the roledef realization of catdef's broader self-describing-extensions principle: every extension carries enough metadata that a downstream agent without prior knowledge can do "something smart."

### Reserved namespaces

The following namespaces are reserved by the roledef spec and MUST NOT be used as extensions:

- `roledef:*` — reserved for roledef spec fields and types (currently `roledef:Role` and `roledef:Job`)
- `catdef:*` — reserved by catdef
- `x.roledef.*` — reserved for roledef-spec-coordinated extensions (e.g., `x.roledef.turing_test_reference`)

---

## Validation rules

### Every roledef artifact MUST

1. Pass catdef structural validation (parses as valid catdef `.openthing`)
2. Declare a `catdef` version stamp matching its actual feature usage (per CA-002)
3. Declare an `roledef` schema version stamp
4. Set `type` to exactly `"roledef:Role"` or `"roledef:Job"`
5. Have all common MUST fields present and non-empty
6. Have at least one entry in `output_contract`
7. Have at least one entry in `guardrails`
8. Use `id` values that are unique within their publishing namespace
9. Use only `x.<domain>.<identifier>` extensions (no other top-level extension patterns)

### Additionally, every `roledef:Job` MUST

1. Include `metadata.role_definition` (object with `id`, `version`, `url`)
2. Include `metadata.org_definition` (object with `id`, `version`, `url`)
3. Reference a `metadata.role_definition.url` that resolves (when public; private repos MAY skip with WARN per validator behavior)
4. Maintain the additive-only invariant against the pinned parent Role (guardrails additive; voice/identity refinable not contradictable; output_contract additive + tightenable; reaction_style examples additive)

### A roledef SHOULD

1. Include `description` for catalog/search legibility
2. Include `reaction_style.examples` for Turing-test fidelity
3. Include `metadata.authors` and `metadata.license` for attribution
4. Include `workflow` if the role has procedural structure
5. Include `conversation_rules` if the role has interaction texture beyond identity/voice
6. Use polymorphic translatable shapes for `name` and `description` if i18n is intended
7. Include `recommended_capabilities` if the role depends on tools, MCP servers, or runtime skills to do its work

### When `recommended_capabilities` is present, each entry

1. MUST include `id`, `kind`, and `reference`
2. MUST use an `id` that is unique within the array
3. SHOULD use a `kind` from the well-known enumeration (`mcp-server`, `claude-code-skill`, `url-resolver`) or `"other"`; an unrecognized `kind` is a WARN, not a FAIL
4. SHOULD use a semver-range `version` and a one-line `purpose`
5. SHOULD self-describe (per the extension self-description SHOULD) when `kind: "other"`
6. SHOULD default `required` to false; `required: true` SHOULD carry a `purpose` justifying the hard dependency

### A `roledef:Job` SHOULD additionally

1. Include `charter` (top-level, prose) — the stable slot of responsibility
2. Include `metadata.placement` for operational Jobs (not required for sample/template Jobs)
3. Run OPD against `metadata.role_definition` (its parent Role) when `workflow.type: "engagement_loop"`

### A `roledef:Role` derived from another Role SHOULD

1. Include `metadata.derived_from` (object with `id`, `version`, `url`)
2. Run OPD against `metadata.derived_from` when `workflow.type: "engagement_loop"`

### A roledef MUST NOT

1. Have contradictory rules (e.g., a guardrail that conflicts with a conversation_rule)
2. Use reserved namespaces (`roledef:*`, `catdef:*`) for non-spec content
3. Reference removed or non-existent roledefs in `metadata.extends` or `metadata.related`
4. Stamp a `catdef` version that does not actually define every feature used (writer-strict per CA-002)
5. Remove or relax parent guardrails when derived (additive-only invariant violation)

### Validator behavior on invalid roledefs

Per the strict-writer / lenient-reader pattern (CA-002, CA-003):

- **Writer-side validators** (e.g., the roledef-validator role) MUST reject invalid roledefs with specific error messages identifying the violations.
- **Reader-side runtimes** (e.g., the roledef-load Claude Code skill) MUST NOT reject a roledef on validation grounds alone. They MAY warn. They process the roledef to the best of their ability and surface warnings to the user.

---

## Versioning

The roledef schema follows semantic versioning, mirroring catdef's pattern:

- **Patch** (0.x.y): documentation clarifications, no schema changes
- **Minor** (0.y.0): new optional fields, new field types, new SHOULD-rules, new artifact tiers (e.g., the `roledef:Job` type added in 0.2.0; the `recommended_capabilities[]` field added in 0.3.0). Old roledefs remain valid; old runtimes gracefully ignore new fields.
- **Major** (x.0.0): breaking changes to required fields or semantics. Runtimes MUST refuse to process roledefs with a higher major version than they support.

The `roledef` version stamp on each roledef MUST declare the minimum schema version that defines every feature used (writer-strict per CA-002). A `roledef:Role` artifact that uses no v0.2.0-specific features MAY stamp `"roledef": "0.1.0"`. A `roledef:Job` artifact MUST stamp `"roledef": "0.2.0"` or higher (since the Job type itself is a v0.2.0 feature).

---

## Worked example

The `roledef-contributor` roledef (the meta-roledef that teaches Claudes how to author roledefs) is the worked example for v0.1. Its complete file lives at `roledefs/roledef-contributor.openthing` and serves both as:

1. The first valid roledef in the roledef library
2. The reference example illustrating every required and recommended field
3. The bootstrap mechanism for self-scaffolding the rest of the seed library

See [`roledefs/roledef-contributor.openthing`](roledefs/roledef-contributor.openthing) once it lands.

---

## Future considerations

The following are not part of v0.3 but are anticipated for future schema versions:

- **Heavyweight roledef inheritance** (`extends` with runtime-merged semantics): allow a roledef to declare a parent and have the runtime fetch + merge fields per defined override rules. The lightweight git-fork-plus-`derived_from` pattern (above, in `metadata`) plus OPD-based behavioral drift management (above, in OPD section) covers v0.2 derivation needs without requiring runtime merge complexity. `extends` is reserved as a future-considerations field if real use cases emerge that the lightweight + OPD pattern can't handle.
- **Runtime hints** (`x.roledef.runtime_hints`): per-runtime advice for instantiation (e.g., temperature, system-prompt placement, role-priming patterns specific to Claude vs Grok vs GPT).
- **Turing test fixtures** (`x.roledef.turing_test`): standardized test scenarios paired with each roledef, enabling automated cross-runtime validation.
- **Composition** (`x.roledef.composes`): a roledef that combines multiple other roledefs (e.g., a "Full-Stack Engineer" roledef that composes "Frontend Developer" + "Backend Developer" + "DevOps").
- ~~**Capability declarations** (`x.roledef.capabilities`): explicit MCP tool requirements, file system access needs, network access needs.~~ **Superseded by v0.3.0** — promoted from an anticipated extension to the first-class [`recommended_capabilities[]`](#recommended_capabilities-array-of-objects) field on Role and Job.
- **Multi-parent OPD** (multi-valued `metadata.derived_from`): if and when a role can derive from multiple parents, OPD iterates trivially. v0.2 OPD assumes single parent.
- **`peer_to` in `metadata.placement`**: identified in the v0.2 placement design as deferred; promote to first-class sub-field if real demand surfaces from operational corpus. v0.2 uses `coordinates_with` as the escape hatch for peer relationships.
- **memodef:OPDReviewMemo subtype**: OPD output is currently formed as a memodef:Memo (or x.memo.* form on a catdef:Thing). If the OPD-review-memo pattern stabilizes, memodef-spec MAY define a dedicated subtype.

These belong to v0.4+ and will be triaged as roledef matures.

---

## Conformance to OAGP principles

This schema embodies the OAGP pattern's three corners:

- **Open**: schema is MIT-licensed, openly published, freely forkable
- **Org chart**: each roledef captures a role with bounded scope, defined inputs/outputs, explicit authority limits (via guardrails), and clear collaboration edges (via metadata.related)
- **Governance**: guardrails and design_constraints encode declared policies that conformant runtimes MUST respect (per catdef value #9)

A roledef is therefore not just a prompt. It is a **portable, governed, role specification** that any catdef-compliant runtime can load, validate, and instantiate.

---

*roledef Schema v0.3.0. May 2026.*  
*An open standard for AI roledefs. Licensed under MIT.*
