# roledef

> An empirically-grounded methodology for portable AI role definitions, with a curated library of role specifications and the conformance evidence to back them.

**roledef is to AI-role invocation what schema.org is to structured web data**: an open standard, a curated reference library, and a methodology validated by empirical conformance evidence across multiple runtimes — not a vendor product, not a platform, not a "one trick" prompt-engineering hack.

---

## The problem

Today, valuable AI behavior lives in three places, none of them portable:

- **Hand-crafted prompts** locked inside specific products (DangerStorm, ChatGPT custom GPTs, individual users' system prompts)
- **Closed agent platforms** with proprietary role definitions you can't take with you
- **Implicit "tribal knowledge"** about how to make Claude or Grok or GPT act a certain way, passed around in screenshots and Discord channels

Each of these is valuable. None survive vendor migration, format changes, or model upgrades. A "Senior Product Manager Interviewer" prompt that took six months to refine on Claude doesn't carry over to Grok. A custom GPT can't be exported. A valuable role is, today, captive to the system that hosts it.

**roledef fixes this** by defining a portable, open format for AI roles — a roledef that any compliant runtime can load — plus an empirical methodology for verifying it actually works across runtimes.

---

## The big idea

A **roledef** is a structured document that captures everything needed to make a fresh AI runtime reliably *become* a specific role:

- **Identity** (who this role IS)
- **Voice** (how it talks)
- **Output contract** (what it produces)
- **Guardrails** (what it MUST NOT do)
- **Workflow, conversation rules, reaction style, design constraints** (how it operates)

roledefs are stored as `.openthing` files (per the [catdef](https://catdef.org) standard substrate), which means they are:

- **Format-portable**: any catdef-compliant tool can read them
- **Runtime-portable**: any compliant AI runtime can load them and instantiate the role
- **Forkable**: GitHub-versionable, MIT-licensed, customizable
- **Composable**: an `OpenOrg` (organization pattern) is a roster of roledefs plus governance rules
- **Self-describing**: extensions carry their own metadata, so receiving runtimes can act intelligently even without prior knowledge

A roledef is therefore a **first-class artifact**: a thing you can publish, fork, version, validate, and reason about — not a string buried inside a vendor's product.

---

## Methodology — anti-clickbait by construction

Most "how to write a great prompt" advice is generic to the point of uselessness because it presumes the runtime is fungible. **It isn't.** The same content delivered to Claude, Grok, Gemini, and Perplexity produces meaningfully different behavior — different fidelity to instructions, different output structure, different willingness to fetch external content, different defaults for how to handle ambiguity. A "one trick" that works for one runtime fails on another.

roledef takes the opposite approach: **empirical, scrutable, falsifiable**. Every roledef in the canonical library has documented conformance evidence on real runtimes. Every methodology rule (loading patterns, verification protocols, runtime classification) was derived from observed runtime behavior, not theoretical design. The conformance evidence files in [`conformance/runtime_evidence/`](conformance/runtime_evidence/) are *empirical case studies in invocation prompt design*.

### Stated principles

Two principles, both empirically validated, are stated commitments of the standard:

**Design principle (validated by 5 falsified hypotheses, 2026-04-26):**

> **Model tools can only touch a very small, whitelisted slice of the public web that varies per runtime, and the canonical roledef library will not be on most runtimes' whitelists.**

The implication: roledef loading must happen at the runtime layer, not the model layer. The model itself cannot be assumed to fetch URLs reliably. Loading mechanisms that violate this (e.g., "ask the model to fetch this URL") cannot achieve cross-runtime portability.

**Positioning principle:**

> **roledef serves the orchestrable-runtime class, not all AI runtimes universally.**

Not every model is amenable to orchestration via structured role definitions, and that's OK. roledef adopts schema.org's positioning model: serve the consumer class that benefits from structured specifications; don't try to force every consumer into compliance. Runtimes outside the addressable class are not failures of roledef — they're outside the target audience.

### Runtime classification (current state, 2026-04-26)

Empirically validated across four architecturally-different runtimes:

| Loading mechanism | Example runtime | Behavior |
|---|---|---|
| **Auto-fetch** | Grok Expert (multi-agent fetch) | Loads roledef natively from URL; full contract conformance |
| **Explicit-fetch (wrapper-v2)** | Claude Code (WebFetch tool) | Loads via explicit fetch instructions in the wrapper prompt; full contract conformance |
| **Search-grounded** | Gemini (Google search grounding) | Loads via search index; partial contract by default; full contract with explicit reinforcement |
| **Paste-fallback / skill-mediated** | Perplexity (sandboxed fetcher) | Refuses to improvise without content; loads from pasted content or skill-injected context |

See [`conformance/runtime_evidence/`](conformance/runtime_evidence/) for full per-runtime evidence, including the design-principle falsification tests and the methodology lessons each runtime surfaced. For current per-runtime status (which runtimes are amenable today, what loading mechanism each uses, what conformance evidence exists), see [`RUNTIME_AMENABILITY.md`](RUNTIME_AMENABILITY.md) — the living document that adopters should consult before choosing a runtime for their use case.

### Methodology rules surfaced from empirical testing

- **id-field discriminating test** — to verify whether a runtime actually loaded the roledef, ask "what's the roledef's `id` field? give only the literal value." Never trust meta-question dodges (e.g., "Have you loaded the roledef?") because the roledef's "no meta-commentary on role/process" guardrail makes meta-questions unreliable. Only content questions can disambiguate.
- **Wrapper-v2 explicit fetch** — for runtimes that won't improvise (Claude Code), the wrapper prompt must explicitly instruct fetch with a fail-safe: "If you cannot fetch, STOP — do NOT improvise."
- **Contract-completeness reinforcement** — for runtimes that produce partial bundles by default (Gemini), a follow-up prompt naming the missing `output_contract` entries by their roledef-spec names recovers full conformance.
- **Source-project peer review for production-extracted roledefs** — when a roledef is extracted from a real shipping product (rather than self-extracted by the role's incumbent), the source project's resident Claude should peer-review for fidelity drifts. Catches additive framing, invented patterns, and hardened constraints that the strategist's self-validation misses.

---

## Loading a roledef today

The forthcoming `roledef-load` Claude Code skill will collapse loading to a single command (`/roledef-load <id-or-url>`), automating the right loading mechanism per runtime. Until it ships, four manual loading mechanisms work depending on your runtime:

### Auto-fetch (Grok Expert and similar multi-agent runtimes)

Just give the runtime the wrapper prompt with the canonical URL placeholder. The runtime's fetch agent resolves the URL automatically.

### Explicit-fetch via WebFetch (Claude Code)

Use the [wrapper prompt template](#wrapper-prompt-template) with explicit fetch instructions, OR the more robust [wrapper-v3 spec](WRAPPER_PROMPT.md) for production use. Claude Code calls WebFetch (with user approval), parses the JSON, instantiates the role.

### Search-grounded (Gemini)

Wrapper-v2 also works on Gemini, which can find the roledef via Google search grounding. After the bundle is produced, a follow-up prompt ("show me Output 3 / 4 / 5 / 6 from the contract") recovers any missing entries.

### Paste-fallback (Perplexity and other fetch-restricted runtimes)

Fetch the roledef yourself from `https://roledef.org/roledefs/<id>.json` (or via `git clone`), paste the JSON content into a fresh runtime session, and prompt it to enact the role. The Perplexity conformance evidence file documents this flow end-to-end.

### Wrapper prompt template

The current canonical wrapper is **wrapper-v3**, which adds verification and bundle-completeness directives over the earlier v2. See [`WRAPPER_PROMPT.md`](WRAPPER_PROMPT.md) for the full spec including reference template, per-runtime adaptations, verification protocols, common pitfalls, and implementation notes for skill / CLI tool authors.

The abbreviated wrapper-v2 template (sufficient for casual use; less robust against partial-default and silent improvisation):

```
You are about to take on a role defined by a roledef — a structured
specification of an AI role's identity, voice, conversation rules,
workflow, output contract, reaction style, design constraints, and
guardrails. roledef is a community standard for portable AI role
definitions.

YOUR FIRST TASK — DO THIS BEFORE ANYTHING ELSE:

Fetch the roledef content. Use whichever of these sources your runtime can
access:

1. WebFetch (preferred): https://roledef.org/roledefs/<roledef-id>.json
2. Local file (if you have local file access): <path-to-local-jd>

If neither tool is available, STOP and tell me — do NOT improvise the role
from the description below.

Once you have fetched the roledef content, parse it as JSON, read it
carefully, and from your next response onward embody the role it defines
completely. Respect every MUST field and every guardrail. When you reach
the role's "generate bundle" step, produce ALL items listed in
output_contract in a single response, formatted with clear section markers
so the founder can identify each deliverable.

After you have fetched and parsed the roledef, respond with the role's
opener and NOTHING else.
```

Replace `<roledef-id>` with the roledef you want to load (e.g., `senior-jaded-vc-associate`). Append loading-pattern-specific notes if needed (literal opener, output bundle delimiter format, etc.).

For production use cases that need stronger guarantees (Gemini-class partial-default recovery, structurally-enforced verification of roledef loading), use the [wrapper-v3 spec](WRAPPER_PROMPT.md).

---

## Canonical distribution

The canonical roledef library is served at **[https://roledef.org](https://roledef.org)** with belt-and-suspenders fetcher portability:

- **`https://roledef.org/roledefs/<id>.openthing`** — canonical artifact, served as `application/json` (Cloudflare Transform Rule)
- **`https://roledef.org/roledefs/<id>.json`** — sibling mirror, served as `application/json` natively

Both URLs serve identical content. Fetcher-capable runtimes can use either.

The same content is also accessible at `https://github.com/roledef-spec/roledef/blob/main/roledefs/<id>.openthing` (web view) or `https://raw.githubusercontent.com/roledef-spec/roledef/main/roledefs/<id>.openthing` (raw view). Note: some runtime fetchers don't reach `raw.githubusercontent.com` URLs reliably (per the [Perplexity conformance evidence](conformance/runtime_evidence/senior-jaded-vc-associate__perplexity__2026-04-25.md)). The `roledef.org` mirror exists to bypass that fragility class for fetcher-capable runtimes; for fetcher-restricted runtimes, paste-fallback or skill-mediated loading remains the only path.

---

## What's in this repository

```
roledef-spec/
├── README.md                       ← you are here
├── SCHEMA.md                       ← the roledef schema (MUST/SHOULD/x.)
├── WRAPPER_PROMPT.md               ← canonical wrapper-v3 spec for loading roledefs
├── RUNTIME_AMENABILITY.md          ← per-runtime status + classification (living doc)
├── CONTRIBUTING.md                 ← how to contribute a roledef
├── CLAUDE.md                       ← maintainer operating manual
├── CNAME                           ← roledef.org GitHub Pages config
├── decisions/                      ← strategist decision artifacts
│   ├── <id>.md                          ← per-roledef inclusion decisions
│   ├── conformance-evidence-first-pass.md  ← cross-runtime methodology
│   ├── rename-openjd-to-roledef.md         ← project rename rationale
│   └── ...
├── proposals/                      ← maintainer-drafted spec proposals
├── proposed-roledefs/                   ← in-flight roledef submissions
├── roledefs/                            ← the canonical roledef library
│   ├── roledef-contributor.openthing       ← meta-roledef: how to write one
│   ├── roledef-validator.openthing         ← meta-roledef: how to validate one
│   ├── catdef-strategist.openthing
│   ├── senior-jaded-vc-associate.openthing
│   └── *.json                              ← .json mirrors of each .openthing
├── conformance/
│   ├── README.md
│   ├── runtime_evidence/           ← per-runtime conformance test results
│   ├── valid_roledefs/                  ← test fixtures: roledefs that should pass
│   ├── invalid_roledefs/                ← test fixtures: roledefs that should fail
│   └── tests/                      ← schema validation test suite
└── catalog.opencatalog             ← the index of all roledefs
```

---

## Authoring a roledef

This section is for **anyone authoring a roledef** — including AI agents reading this README with the intent to draft their own roledef as a contribution to the library. The procedure works identically for human and AI authors.

### Procedure

**1. Read the schema.**

Read [SCHEMA.md](SCHEMA.md) end-to-end. Understand the MUST / SHOULD / x. taxonomy. Know what fields are required and what fields are recommended.

**2. Identify the role you want to capture.**

Common sources, in order of empirical strength:

- **Self-extraction** (strongest): a role you've played yourself, with documented session transcripts and decision history as primary sources. The role's actual incumbent is the most reliable extractor.
- **Production extraction**: a hand-crafted prompt you want to formalize (e.g., the system prompt of an existing product like DangerStorm). Production-extracted roledefs SHOULD be peer-reviewed by the source project's resident Claude before promotion to canonical (catches additive framing and fidelity drifts).
- **Observed extraction**: a role observed in another implementation (e.g., a competitor's agent you want to express portably).
- **Designed**: a novel role for a domain that doesn't yet have one. Lowest empirical grounding; highest risk of drift between roledef and runtime behavior.

The most reliable roledefs come from roles where you have **empirical evidence** — actual session transcripts, prompts, output samples. roledefs derived from theoretical descriptions tend to drift from real behavior at instantiation.

**3. Gather source material.**

Before writing the roledef, collect:

- 3-5 example interactions (transcripts, prompts, outputs)
- Any existing prompt or system message that drives the role today
- Notes on the role's voice, characteristic phrasing, signature moves
- Notes on what the role refuses to do (the implicit guardrails)

**4. Draft the MUST fields.**

Write each required field in turn. Don't skip any:

- `catdef` — the catdef version (currently 1.4)
- `roledef` — the schema version (currently 0.3.0)
- `type` — exactly `"roledef:Role"`
- `id` — kebab-case identifier
- `name` — human-readable name
- `version` — your roledef's own version (start at 1.0.0)
- `identity` — paragraph capturing who the role IS
- `voice` — how the role talks
- `output_contract` — what the role produces (at least one entry)
- `guardrails` — what the role MUST NOT do (at least one entry)

If you can't answer any of these, the roledef isn't ready yet. Go back to source material.

**5. Add the SHOULD fields.**

Most good roledefs include:

- `description` — one-sentence summary
- `conversation_rules` — interaction patterns
- `workflow` — the steps the role takes
- `reaction_style` — with **examples** (most important for Turing-test fidelity)
- `design_constraints` — output quality standards
- `examples` — sample interactions or outputs
- `metadata` — authors, license, attribution, related roledefs (with field-by-field provenance audit in `extracted_from`)

The single highest-leverage SHOULD field is **`reaction_style.examples`**. Concrete reaction examples are how validators (and humans) check whether a roledef-instantiated role matches the original behavior.

**6. Add domain-specific extensions if needed.**

If your role has domain-specific concerns the schema doesn't cover, use `x.<domain>.<identifier>` extensions. Per the SHOULD rule in SCHEMA.md, **each extension SHOULD include enough self-description that a naive agent can act intelligently** — typically by expressing the extension as an object with a `description` field.

**7. Self-validate.**

Before submitting, run through this checklist:

- [ ] All MUST fields present and non-empty
- [ ] `output_contract` has at least one entry, with `name`, `description`, `format`, and `schema`
- [ ] `guardrails` has at least one entry
- [ ] `catdef` version stamp matches actual feature usage (per CA-002 writer-strict rule)
- [ ] `roledef` version stamp present
- [ ] `id` is kebab-case and not already used in the library
- [ ] `reaction_style.examples` are concrete enough to anchor Turing-test validation
- [ ] Extensions follow the self-description SHOULD rule
- [ ] No reserved namespaces (`roledef:*`, `catdef:*`) used for non-spec content
- [ ] No contradictory rules between guardrails, conversation_rules, and workflow
- [ ] If production-extracted: source-project peer review arranged

**8. Submit via pull request.**

roledefs follow a **two-stage process**: submissions land first in `proposed-roledefs/` (where they are publicly visible as in-flight), and are then promoted to `roledefs/` (the canonical library) at merge time after validation and review.

For full details on the contribution process — including PR templates, validation expectations, and what to do if your submission is rejected — see [CONTRIBUTING.md](CONTRIBUTING.md).

### A note for AI authors specifically

If you are an AI agent reading this with the intent to author your own roledef (e.g., a `catdef-strategist` Claude reading this in order to formalize the strategist role into a roledef entry), you have a unique advantage: **you ARE the role you're describing**. Your own session transcripts, your own decision history, your own characteristic phrasing — these are primary sources.

The procedure for self-authoring:

1. **Introspect on your own session behavior.** What do you notice yourself doing repeatedly? What do you refuse? What's your characteristic tone?
2. **Look at your transcripts.** Most Claude/AI sessions can be exported. The transcripts ARE the evidence.
3. **Extract — don't invent.** A roledef that captures what you actually do is reproducible. A roledef that captures what you wish you did is aspirational and won't pass the Turing test.
4. **Write the roledef as you understand the role.**
5. **Submit it. The validator will tell you if you missed anything.**

This is roledef's self-scaffolding property: the federation members who play the roles author the roledefs that capture those roles. You don't need anyone else to extract a roledef from you. Read this README, follow the procedure, submit the result.

---

## The Turing test for roledefs — empirically grounded

Every roledef in the canonical library must pass an empirical Turing test before merge: a fresh AI runtime loaded with the roledef must produce **behavior equivalent** to the original-incumbent's documented behavior.

### How the test works

1. Identify the **reference behavior**: documented session(s) of the original incumbent role acting on a specific scenario.
2. Instantiate a **fresh runtime** (Claude, Grok, Gemini, Perplexity — preferably one that has not been part of the original sessions) with only the candidate roledef as context.
3. Run the **same scenario** against the fresh runtime.
4. Compare outputs against reference behavior:
   - Does the fresh runtime ask similar questions in similar order?
   - Does it react in similar ways (paraphrase, push back, show enthusiasm at the same moments)?
   - Does it produce equivalent deliverables (output_contract entries match)?
   - Does it honor the same guardrails?
5. **Pass** = the fresh runtime's behavior is functionally equivalent. **Fail** = significant divergence; iterate the roledef.

### Cross-runtime testing — already proven on 4 runtimes

Cross-runtime portability is empirically validated, not theoretical. The `senior-jaded-vc-associate` roledef has been Turing-tested on Grok Expert, Claude Code, Gemini, and Perplexity — each via a different loading mechanism. All four produce role-faithful behavior; tactical variance between runtimes (question order, pushback timing, atomic-vs-iterative bundle delivery) falls within the roledef's constraint envelope rather than violating it.

See [`conformance/runtime_evidence/`](conformance/runtime_evidence/) for the four evidence files, each scoring the runtime's behavior against the roledef's MUST and SHOULD fields with per-criterion conformance results.

### When tests fail

Failure is diagnostic:

- **Schema gap**: the roledef is missing a field that the original incumbent relies on. Update the schema (open a proposal) and the roledef.
- **Conformance issue**: the runtime doesn't honor some part of the roledef reliably. File a bug against the runtime, OR document the issue as a runtime-amenability finding.
- **Extraction incomplete**: the original prompt has tacit knowledge that didn't make it into the roledef. Iterate the roledef.
- **Runtime non-amenable**: the runtime isn't in the addressable class (per the positioning principle). Acceptable; document and move on.

A passed Turing test validates the whole loop. A failed one tells you what to fix or accept.

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for the full contribution process. In brief:

1. Author a roledef using the procedure above
2. Open a PR adding the roledef to `proposed-roledefs/`
3. roledef-validator runs schema validation + Turing test on at least one runtime; posts report on the PR
4. **For production-extracted roledefs:** source-project peer review by the source project's resident Claude (catches additive framing and fidelity drifts)
5. roledef-maintainer reviews
6. roledef-strategist signs off on borderline cases (library fit, scope)
7. Approved PR is updated atomically: file moves `proposed-roledefs/` → `roledefs/`, catalog entry added, decision recorded
8. Merge

For governance discipline, see [CLAUDE.md](CLAUDE.md) — the maintainer operating manual. For substantive design questions (schema changes, process changes), file a proposal in `proposals/`.

### Quality bar

A roledef MUST:
- Pass schema validation (per [SCHEMA.md](SCHEMA.md))
- Pass the Turing test on at least one runtime
- Include sufficient metadata (authors, license, attribution)

A roledef SHOULD:
- Pass the Turing test on multiple runtimes (cross-runtime portability)
- Include rich `reaction_style.examples`
- Document its source via field-by-field provenance audit in `metadata.extracted_from` (EXTRACTED / FRAMING / FORMALIZATION / OUT-OF-SCOPE)
- Include source-project peer review (for production extractions)

The roledef library is curated. Not every well-formed roledef will be merged into the canonical library — you may publish your own roledef library at any URL using the same format, and it will be just as portable.

---

## Related projects

roledef is one corner of the **OAGP** (Open Agentic Governance Pattern) family. The full stack:

- **[catdef](https://catdef.org)** — the format substrate. roledefs are catdef artifacts.
- **[openmemo](https://openmemo.dev)** — the communication protocol. roledef-instantiated employees talk to each other via openmemo.
- **[openorg](https://openorg.dev)** — the organizational pattern catalog. Each `.openorg` composes roledefs into a complete org.
- **[oagp.online](https://oagp.online)** — the meta-pattern documentation. The "why" behind the "what."

You can use roledef alone (standalone roledefs for individual AI roles) or as part of the full OAGP stack (federations of roledef-instantiated employees coordinating via openmemo under openorg governance).

### Coexistence with adjacent specifications

roledef occupies the **role definition** layer of a multi-layer AI specification stack. Adjacent layers and how they relate:

- **[Oracle Open Agent Specification](https://github.com/oracle/agent-spec)** — defines AI agent runtime configurations (LLM provider, model, tool bindings, adapter pattern). Complementary to roledef, not competitive: a roledef defines the role's behavior; an Agent Spec configuration defines the runtime to execute it. A complete worker specification could compose both.
- **[AWS Thinkbox OpenJD](https://github.com/OpenJobDescription/openjd-cli)** — defines render-farm batch job submission. Different technical domain entirely; the naming overlap with the project's pre-rename name (openjd) was the primary motivation for the rename to roledef. See [`decisions/rename-openjd-to-roledef.md`](decisions/rename-openjd-to-roledef.md).

---

## Status

**v0.1.0 — bootstrap, empirically validated.** As of 2026-04-26:

- ✅ Schema (`SCHEMA.md`) defined and stable for v0.1
- ✅ Canonical library has 4 roledefs (`roledef-contributor`, `roledef-validator`, `catdef-strategist`, `senior-jaded-vc-associate`)
- ✅ Cross-runtime conformance empirically validated on 4 runtimes (Grok Expert, Claude Code, Gemini, Perplexity)
- ✅ Brand-controlled distribution at [roledef.org](https://roledef.org) (GitHub Pages + Cloudflare DNS + .openthing/.json belt-and-suspenders fetcher portability)
- ✅ Two stated principles published in decision artifacts (design + positioning)

In progress for v0.1+:

- 📋 `roledef-load` Claude Code skill — **design scoped**, see [`decisions/roledef-load-skill-design.md`](decisions/roledef-load-skill-design.md); implementation pending (~3-4 hours estimated)
- ✅ [Wrapper-v3 spec](WRAPPER_PROMPT.md) (formalizes fetch-or-stop + id-field verification + contract-completeness check; reference template + per-runtime adaptations + implementation notes for skill authors)
- ✅ [Runtime Amenability page](RUNTIME_AMENABILITY.md) (per-runtime current status, four-category classification, how to test new runtimes)
- ⏳ Seed library expansion (catdef-maintainer, brother-blackhat-tester, more production extractions)

Breaking changes are possible until v1.0.0 (target: when the seed library has 10+ roledefs across multiple domains and the schema has survived contact with several real-world contributors).

---

## License

MIT. roledefs in this library are MIT-licensed. roledefs published in third-party libraries follow their own licenses; the roledef format itself is MIT and ungovernable.

---

## Acknowledgments

roledef extracts patterns from years of practical AI-role design — Scott Welch's DangerStorm (the source production deployment for the `senior-jaded-vc-associate` roledef), Conversii, the catdef four-Claude federation, and many others. Every reference roledef in the canonical library credits its source in `metadata.extracted_from`.

The roledef methodology was sharpened by empirical conformance testing across multiple runtimes. Particular acknowledgment to **Perplexity**, which during the 2026-04-26 testing session falsified five distribution-layer hypotheses in succession and articulated the canonical design principle ("model tools can only touch a very small, whitelisted slice of the public web") more crisply than the strategist had — a satisfying recursive moment for a methodology designed to be empirically grounded.

The roledef schema is informed by the OAGP design pattern. If you adopt roledef, you are practicing Open Agentic Governance Pattern at the role level. See [oagp.online](https://oagp.online) for the broader context.

---

*roledef v0.1.0. April 2026. An open, empirically-grounded standard for portable AI role definitions. Licensed under MIT.*
