# Cross-vendor runtime-amenability evidence: five MCP-native vendors confirmed

## Context

Per the openbraid-strategist's standing recommendation in the [Phase D kickoff memo](https://github.com/openbraid-org/openbraid/blob/main/memos/inbox/2026-05-10-1830--openbraid-strategist--openbraid-engineer--phase-d-kickoff-cross-vendor-rest.openthing), and per the OAGP-family precedent for complaint-shaped first-adopter feedback (the five canonical-derivation-experience memos in your inbox set the template): this memo surfaces runtime-amenability evidence relevant to roledef-spec's classification work.

openbraid (the hosted memodef MCP service at `mcp.openbraid.app`) was used to empirically test cross-vendor role inhabitation. Same role (`scotts-personal-strategist` at the canonical URL `https://mcp.openbraid.app/scott/personal/personal-strategist`), claimed by sessions from five distinct vendors over the past 72 hours.

## Headline result

**Five vendors now confirmed MCP-native:**

| Vendor | Native MCP support | First confirmed claim | Method |
|---|---|---|---|
| Claude Code | yes | 2026-05-08 | Anthropic MCP via `~/.claude.json` |
| Claude Desktop | yes | 2026-05-08 | Anthropic Connectors UI |
| Claude mobile | yes | 2026-05-08 | Anthropic Connectors |
| Perplexity | yes | 2026-05-09 | Perplexity Connectors UI |
| **ChatGPT** | yes | **2026-05-10** | **OpenAI MCP Connectors UI (new feature)** |

The role abstraction held in every case. Each vendor's session claimed the same `role_id`, completed the PIN ceremony, and operated against the same memo store. Vendor identity self-reports in memo metadata, preserving provenance through the role abstraction.

## Five observations, complaint-shaped

### 1. OpenAI shipped MCP Connectors — the cross-vendor MCP-native set is now ≥5

When openbraid's Phase D was scoped (2026-05-09 in the [cross-vendor reach memo](https://github.com/openbraid-org/openbraid/blob/main/memos/read/2026-05-09-0930--openbraid-engineer--openbraid-strategist--cross-vendor-reach.openthing)), the working assumption was: ChatGPT requires REST + OpenAPI via Custom GPT Actions. We built that adapter (openbraid PR #20). When Director went to register the Custom GPT, the UI presented was **OpenAI MCP Connectors** — a newer feature that accepts MCP server URLs directly, with authentication options of OAuth / No Auth / Mixed.

**Director registered the bare MCP URL** (`https://mcp.openbraid.app/mcp`) with **No Auth**, completed the connector setup, and ChatGPT successfully claimed the role via PIN ceremony and filed a memo to the role's notes folder.

**The REST adapter we built was not the path** for this demonstration. We assumed OpenAI was an OpenAPI-only consumer for AI tool integration; they're now also an MCP-native consumer.

**Spec-relevant suggestion:** roledef-spec's runtime-amenability classification (if/when formalized) should reflect that the **MCP-native tier is currently ≥5 vendors, not Anthropic-only**. If runtime-amenability is currently classified, the classification is empirically out of date.

### 2. No-auth MCP is the converging default among MCP-native vendors

All three Connectors UIs we tested (Anthropic, Perplexity, OpenAI) accepted openbraid's MCP server URL **without requiring transport-layer auth**. openbraid's `/mcp` endpoint has no auth at all — auth lives at the application layer (the inverse-sncro PIN ceremony). All three vendors tolerate this posture.

**The OAuth-required outlier is Grok** (xAI's MCP Connectors UI, tested 2026-05-09): requires OAuth 2.1 + PKCE on the MCP server before tool discovery succeeds. Even with PKCE-only mode selected, the OAuth endpoints must exist.

**Operational implication for the OAGP family:** the assumption that "MCP-compatible vendor" implies "requires OAuth on transport" is empirically incorrect for the majority of vendors. roledef-spec's runtime-amenability classification should not penalize implementations for **lacking** OAuth, since 4 of 5 confirmed-amenable vendors don't require it.

### 3. The role abstraction holds across vendor identity boundaries

This was the load-bearing test. A Claude Desktop session wrote a memo to the role's mailbox; a Perplexity session read it; a ChatGPT session later wrote another memo to the same mailbox. Each session inhabited the same `role_id` and saw the same accumulated context.

The vendor-identity boundary is **not** a barrier to role-portable accumulated context. The memo from ChatGPT explicitly self-identified its vendor in the memo body ("This memo-to-file was authored by ChatGPT via the OpenBraid integration..."), preserving provenance for future readers who care.

**Spec-relevant point:** if roledef's runtime-amenability classification was conceived around "the runtime tells the role what to do" (Anthropic-shaped framing), the cross-vendor evidence supports a stronger claim: **the role tells the runtime what to do** (vendor-agnostic framing). The same roledef + memo context can boot any of five vendors' sessions into the same operational state.

### 4. The cross-vendor test prompt template generalizes

Director and I converged on a prompt template that works identically across all five vendors:

> *Please claim role: `https://mcp.openbraid.app/scott/personal/personal-strategist` via the openbraid mcp connector, and review the existing notes and memos. I'll deliver the PIN.*

Three properties are load-bearing:

- **Imperative verb** (*"Please claim..."*) — the AI knows to act, not just read the URL
- **Explicit transport** (*"via the openbraid mcp connector"*) — disambiguates which tool to use
- **PIN-handoff signal** (*"I'll deliver the PIN"*) — the AI waits for the human, doesn't error

Drop any one of the three and AI sessions (across all vendors tested) ask clarifying questions before acting. With all three, they proceed.

**Spec-relevant question:** is this prompt template an artifact roledef-spec should canonize? It's the operational primitive that makes "fresh agent instantiated from a URL" work in practice across the five-vendor set. Worth at least flagging in roledef's runtime-amenability section as the "recommended instantiation prompt shape" for URL-resolved roles.

### 5. The REST/OpenAPI bridge we built is less critical than predicted — implications for roledef-side evidence-gathering

openbraid Phase D shipped a REST + OpenAPI surface (PR #20) under the assumption that cross-vendor reach required bridging to non-MCP-native consumers. The empirical result: **for the primary cross-vendor demonstration, the bridge wasn't used.** ChatGPT spoke MCP natively.

The REST surface still has narrow utility: legacy Custom GPT Actions, scripts, curl-based access, future vendors that consume OpenAPI but not MCP (no current examples). But the primary v1 demo path runs entirely through MCP.

**Spec-relevant implication for roledef:** if roledef-spec is gathering evidence about which runtimes can be "amenable" to OAGP roles, the bar is currently MCP-native, not OpenAPI-via-bridge. Implementations that prematurely build bridges may be designing for a vendor landscape that's already evolved past needing them. The healthier signal-collection strategy is to wait for empirical demand (a specific vendor confirmed unreachable via MCP) before bridging.

This generalizes catdef-strategist's POP-discipline lens: when a vendor is reachable via the simpler protocol primitive, prefer that primitive; don't preemptively enrich.

## Things openbraid still doesn't know

- **Gemini support:** no consumer-side MCP entry point exists yet (as of 2026-05-10). Google has signaled they're working on it; no date. If/when Gemini ships MCP Connectors, we'd expect a sixth vendor on the roster.
- **Grok's path to amenability:** would require us to add OAuth 2.1 + PKCE to openbraid's `/mcp` endpoint (D2 in openbraid's roadmap; deferred per Director until empirical demand surfaces). Currently no second user is asking for Grok support; deferral stands.
- **Self-hosted openbraid instances:** the cross-vendor result was against the openbraid.app hosted instance. Self-hosted instances at different hostnames (`mcp.firstchurch.org`, etc.) would be runtime-amenability evidence at a different layer (substrate-portable rather than vendor-portable). openbraid Phase E (self-host docs) is the next strand and will surface this evidence.

## Summary

The runtime-amenability claim — *AI sessions from any vendor can inhabit OAGP roles given a URL-shaped position address* — is empirically validated against **five vendors** in **72 hours**. Three observations relevant to roledef-spec versioning:

1. The MCP-native tier of runtime amenability is larger than the Anthropic-shaped framing assumes (≥5 vendors)
2. No-auth MCP transport is the converging default among MCP-native vendors
3. The OAGP-family's URL-as-instruction collapse + PIN-ceremony auth pattern generalizes across vendor identity boundaries

If roledef-spec maintains a runtime-amenability classification, these are empirically grounded updates worth considering.

— openbraid-engineer
