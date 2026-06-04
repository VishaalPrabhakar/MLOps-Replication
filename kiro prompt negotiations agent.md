# Kiro Build Prompt — “Negotiations Agent” (LangGraph, Production-Grade, Hybrid LLM)

> Paste this entire file into Kiro as the build specification. Work in reviewable
> increments and **pause for review after Step 0 and after Step 1.**

-----

## Context & Objective

You are working inside the existing application project **`Contracts Agent Initiative`**. This codebase already contains one or more **production-grade LangGraph agents** (e.g., the existing Contracts Q&A agent). Add a new agent: **`Negotiations Agent`**.

The agent serves the **health insurance company (the payer)** negotiating a contract **with a hospital (the provider)**. It upgrades passive contract Q&A into an active, payer-side negotiation assistant exposing a **Category 1 universal contract toolset** plus a **payer-perspective healthcare domain layer**.

**Critical constraint — reuse, do not reinvent.** The application already has established, production-tested patterns (state schema, node/edge wiring, tool registration, RDS vector retrieval, memory/checkpointing, config/secrets, logging/tracing, error handling, auth, tests). **Discover and conform to those patterns.** Do not introduce new frameworks, directory conventions, state approaches, or tool-registration mechanisms. When the clean-from-scratch way conflicts with the repo’s existing way, **choose the repo’s way.**

**Architecture is hybrid by design** (see Step 2/3): judgment/legal-language tools are LLM-backed; numeric and reference-lookup tools are deterministic. Do not make the numeric tools LLM-generated — hallucinated benchmark figures are an unacceptable failure in rate negotiation.

**Canonical behavioral reference.** A companion file, `reference_implementation_appendix.md`, contains working sandbox code for the agent, the hybrid tool layer, the LLM client (forced-schema + payer system prompt), the deterministic tools, the result schema, the seeded test contract, and the test harnesses. **It is the source of truth for tool behavior, inputs/outputs, the result schema, compliance gating, and the LLM/deterministic split.** Use it as the behavioral spec for Steps 2–3. Do **not** copy its file/module structure verbatim — structure must follow the repo patterns from Step 0 (the repo wins on any structural conflict). Its mock retriever and hardcoded reference data are stand-ins to be replaced per Step 4; the freshness/validation fail-closed behavior is NOT in the sandbox code and is required new work.

-----

## Step 0 — Discovery (do first; produce a written summary; then PAUSE for review)

Do not write code yet. Inspect the repo and report:

1. Every existing LangGraph agent and its file path.
1. For the most representative production agent, document its anatomy:
- graph **state** definition (schema/typed dict/pydantic);
- **node/edge** declaration and graph compilation;
- **tool** definition + registration mechanism;
- **RDS vector retrieval** (retriever/client, connection handling, chunk query API);
- **memory / checkpointing / conversation persistence**;
- **config, secrets, logging, tracing, error handling**;
- **LLM client** usage (how the repo calls the model, model-ID config, structured-output/tool-use pattern);
- **test** structure for an agent.
1. A **“Patterns I will follow”** summary naming the exact base classes, helpers, and conventions to be reused.

**PAUSE. Present this summary for review before Step 1.**

-----

## Step 1 — Scaffold the Negotiations Agent (then PAUSE for review)

Using the discovered patterns:

1. Create `Negotiations Agent` in the same location/structure existing agents use.
1. Define its graph state by **reusing the existing state pattern**, extended only with fields this agent needs: active contract reference, contract version set, target clause, jurisdiction/state, payer-context inputs (regional membership/volume, member share for the hospital), and detected domain = healthcare.
1. Wire it to the **existing RDS vector retriever** to read contract-document chunks already stored there. Do not write a new retriever.
1. Register the agent so it is discoverable/deployable exactly as existing agents are.

**PAUSE. Present the scaffold for review before Step 2.**

-----

## Step 2 — Category 1 Universal Toolset

Implement as a **single cohesive backend tool module** (eight capabilities as sub-functions), registered via the existing tool mechanism. Tools that answer from contract content must ground their answers in retrieved chunks via the existing retriever and are subject to the Step 4 grounding gate (no answer without sufficient retrieved context). **Mode** indicates LLM-backed vs deterministic. For exact behavior, I/O, and the result schema, follow `reference_implementation_appendix.md` (the LLM-backed tools mirror the judgment tools shown there; the deterministic tools mirror the deterministic ones).

1. **Clause Risk Scorer** *(LLM)* — risk rating + exposure explanation + safer alternative; categorizes provisions (indemnification, liability caps, termination, dispute resolution, etc.).
1. **Redline Suggester / Alternative Clause Generator** *(LLM)* — given a clause + intent, drafts a narrower/compliant alternative. Highest-leverage capability.
1. **Obligations & Deadlines Extractor** *(LLM extraction, deterministic table assembly)* — structured table: obligation, owner, due date; surfaces buried auto-renewal/notice deadlines.
1. **Missing Clause Detector** *(LLM)* — compares against a standard template for the contract type; flags absent clauses.
1. **Negotiation Position Advisor** *(LLM)* — proposes ask / reasonable concession / fallback, drawing on contract terms + playbook/precedent docs via the retriever.
1. **Multi-Version Diff Tool** *(deterministic diff + LLM explanation)* — compares counterparty redline vs prior version; explains changed/accepted/rejected/new language; preserves history.
1. **Plain-Language Summary Generator** *(LLM)* — plain-English summary of the whole contract or a section.
1. **Contract Expiry & Renewal Tracker** *(deterministic + existing persistence)* — extracts renewal/expiry dates; answers “what’s renewing in 90 days?” via the repo’s persistence pattern.

-----

## Step 3 — Payer-Perspective Healthcare Domain Layer (strict; nine tools)

**Perspective — non-negotiable.** Every tool analyzes, scores, and drafts **from the payer’s side only.** There is no perspective parameter. Default to the payer’s cost exposure, leverage, and the payer’s own regulatory liability. **Compliance tools (4, 5) are liability-prevention, not leverage**: they must be structurally prevented from recommending anything that reduces the payer’s legal compliance, and from framing a disparity/gap as leverage to retain. Enforce the payer stance and the “do not invent figures/statutes” rule at the LLM system-prompt layer for every LLM-backed tool. For exact behavior, I/O, result schema, the payer system prompt, and the compliance-gating flag, follow `reference_implementation_appendix.md` — the nine tools below correspond directly to the tools in that file.

1. **Reimbursement Rate Benchmarker** *(DETERMINISTIC)* — CPT/service line + proposed rate; benchmarks vs Medicare, regional commercial ranges, MGMA; **ingests price-transparency files** (this hospital + competitors) to show what other payers pay; quantifies annual overpayment; accepts payer volume as a leverage input. Prioritize transparency-file ingestion; stub external sources behind clean interfaces.
1. **Rate-Basis & Escalator Analyzer** *(DETERMINISTIC)* — identifies fee-schedule basis (current-year vs fixed-year Medicare peg), flags automatic/compounding escalators, recommends payer-favorable bases/caps; models multi-year cost delta.
1. **Prior Authorization Clause Analyzer** *(LLM)* — evaluates whether PA terms preserve payer UM authority; flags auto-approval triggers, aggressive turnaround, blanket retroactive-denial bans; suggests payer-protective language bounded by lawful PA-reform limits.
1. **Parity Compliance Checker** *(LLM, COMPLIANCE-GATED)* — payer bears MHPAEA obligation; compares behavioral-health vs comparable med/surg terms incl. **non-quantitative treatment limits** (e.g., differential prior-auth); flags every disparity as payer liability; recommends only toward parity.
1. **No Surprises Act & Balance-Billing Scanner** *(LLM, COMPLIANCE-GATED)* — flags missing balance-billing/hold-harmless protection, missing good-faith-estimate obligations, OON emergency gaps creating payer liability; recommends only toward compliance.
1. **Payer-Favorable Contract-Language Optimizer** *(LLM)* — strengthens operational clauses governing effective cost regardless of headline rate: clean-claim definition, downcoding/bundling/edit rights, medical-necessity discretion, recoupment/audit windows, appeal timelines. (MFN language only where legal — defer to item 9.)
1. **Value-Based Care Scenario Modeler** *(DETERMINISTIC)* — models shared-savings/bundled/risk-corridor arrangements from payer total-cost-of-care and downside exposure; inputs: shared-savings %, quality thresholds, attributed panel size.
1. **Network Adequacy & Termination/Disruption Risk Assessor** *(DETERMINISTIC + LLM clause reading)* — assesses payer exposure if the hospital terminates: hospital essentiality to regional network adequacy, member-disruption/continuity-of-care exposure, regulatory adequacy risk; flags notice-period terms (standard 90–120 days) and asymmetric termination rights; accounts for current hospital-initiated MA-termination dynamics.
1. **State-Specific Regulatory Overlay** *(DETERMINISTIC lookup)* — accepts state + contract type; retrieves applicable rules (prompt-payment timelines, any-willing-provider, PA-reform statutes; ERISA self-funded carve-outs); annotates clauses with both payer obligations and constraints on the payer. Governs MFN legality for item 6.

-----

## Step 4 — Production-Grade Requirements (match existing agents)

Conform to existing patterns; do not invent new ones.

- **LLM client:** use the repo’s existing model-calling pattern. Force **structured output** (tool-use/JSON-schema) so every tool returns a uniform, parseable result. Make the **model ID configurable** (do not hardcode); default to a current production model ID verified against Anthropic’s model docs. Read credentials from the environment/secrets manager the repo already uses — **never hardcode an API key**.
- **Payer guardrail (system-prompt layer):** every LLM tool runs under a system instruction that (a) fixes the payer perspective, (b) forbids inventing figures/statutes/benchmarks, (c) requires grounding claims in provided contract text.
- **Grounding gate — no answer without retrieved context (fail-closed, code-level):** The agent must NEVER fabricate answers for questions that depend on contract data that is absent from the vector store (empty store, document not yet ingested, or retrieval returns nothing relevant). This is distinct from freshness (data exists but is old) and validation (data exists but is wrong); this guards data that is ABSENT. Apply in this order:
  - **Pre-generation retrieval gate (primary, deterministic).** For every retrieval-dependent tool, run retrieval BEFORE calling the LLM and inspect the result. If the store is empty, or no retrieved chunk clears a configurable relevance/similarity threshold, the tool **short-circuits and returns a fixed “insufficient grounding data” result — the LLM is never invoked for that query.** Because the guard is a code branch, not a prompt instruction, the model cannot hallucinate: it is not called. This is the primary protection.
  - **Grounding contract (secondary).** For queries that pass the gate but have thin context, the system prompt requires the model to answer ONLY from provided context and to state explicitly that it cannot answer when the context does not contain the answer. This is a backstop to the code gate, not a substitute for it.
  - **Citation-required output (verification).** Every grounded tool returns the chunk IDs / source spans it used. The synthesis step drops or flags any claim that carries no citation to a retrieved chunk. An uncited claim is treated as ungrounded.
  - **“No data” ≠ “no answer” ≠ “silent on X”.** The agent must distinguish, in its output, three different states: (a) the data needed has not been loaded/retrieved (“I don’t have the contract data to answer this”); (b) the contract was retrieved and is genuinely silent on the point (“the contract does not address X”); (c) a substantive grounded answer. It must NEVER let an empty/absent store masquerade as “the contract is silent on X,” and never return a confident answer in state (a).
  - **Scope:** the grounding gate applies to ALL retrieval-dependent tools with no exemption. Tools that do not depend on retrieved contract data for a given call are unaffected. No tool may fall back to model training knowledge to fill an absent-data gap.
- **Compliance gating (structural):** tools 4 and 5 carry an explicit gated flag and a post-generation check that rejects/strips any output framing a disparity or compliance gap as leverage to retain.
- **PII/PHI guardrail:** reuse the existing redaction/guardrail pattern; ensure no PHI reaches the LLM improperly. If none exists, add a pre-LLM PHI scan behind a clean interface and flag it.
- **Hybrid integrity:** numeric/lookup tools (Rate Benchmarker, Escalator, VBC, Termination scoring, State Overlay, Expiry Tracker) must compute deterministically; the LLM may explain but must not originate the numbers.
- **Data freshness & sourcing (fail-closed):** The agent must NEVER use stale reference data. Apply these rules:
  - **No hardcoded real-world facts.** Medicare/MGMA rates, price-transparency figures, state regulatory rules (prompt-pay timelines, any-willing-provider, PA-reform statutes), and any other volatile fact must live in a **versioned external source behind the existing data-access pattern** — never as constants in tool code. The VBC savings assumption and any benchmark percentages are configuration/data, not code literals.
  - **Every reference dataset carries provenance metadata:** a source identifier, a publication/effective date, and an `as_of` retrieval timestamp. Tool output that relies on it must surface that provenance (source + as-of date) so any figure is auditable and defensible.
  - **Per-source, scoped fail-closed.** Each volatile dataset has a configurable maximum age (freshness SLA). If a dataset is older than its SLA, missing, or its provenance can’t be verified, **every tool that depends on that dataset must refuse to produce a figure-bearing recommendation** and return a clear “data stale/unavailable — refusing to answer from stale data” result naming the source and its age. Refusal is scoped to the dependent tools only: tools with no external-data dependency (Multi-Version Diff, Plain-Language Summary, clause-language reasoning) continue to operate. One stale table must not brick the whole agent, and no tool may silently fall back to a cached/last-known or model-recalled figure.
  - **Freshness is checked at use time, not just load time**, and “today”-relative logic (Expiry Tracker) must read the current date dynamically.
- **Source validation (separate from freshness; fail-closed per source):** Fresh data can still be wrong. Before any reference dataset is used in a figure-bearing recommendation, validate it independently of its age:
  - **Sanity bounds (per field).** Each volatile field carries configurable plausibility bounds (e.g., commercial rate as % of Medicare within a sane floor/ceiling; prompt-pay days within a legal range). A value outside bounds is treated as invalid data, not a finding — the dependent tool fails closed and flags the source for review, rather than emitting an implausible recommendation.
  - **Cross-source agreement.** Where the same fact is available from more than one source (e.g., a hospital’s rate from its transparency file vs. a market benchmark, or a state rule from two references), compare them. Agreement within a configurable tolerance → proceed and record which sources concurred. Material disagreement beyond tolerance → do NOT silently pick one; surface the conflict (both values + sources + as-of dates) and either fail closed for that figure or return it explicitly flagged as unverified-conflicting, per the source’s configured policy.
  - **Schema/integrity checks.** Validate structure, units, and completeness on load (expected fields present, units as declared, no nulls in required fields). Reject and fail closed on malformed data rather than coercing it.
  - **Scoping and no-silent-fallback (same rule as freshness).** Validation failure is scoped to the tools depending on that source; unrelated tools continue. No tool may substitute a cached, defaulted, or model-recalled value to get around a validation failure.
  - **Auditability.** Validation outcome (bounds pass/fail, sources compared, agreement/conflict, as-of dates) is logged and, where it affects a figure, surfaced in the tool’s output.
- **Observability:** reuse existing logging/tracing; every tool call, retrieval, and model call is auditable, including the provenance/as-of date and validation outcome of any reference data used.
- **Resilience:** no infinite ReAct loops (explicit termination conditions, tool-call caps); LLM tools degrade gracefully (clear error result, not a crash) on API/credential failure.
- **Config & secrets:** reuse existing handling for RDS, model credentials, and reference-data connections.

-----

## Step 5 — Deliverables & Acceptance Criteria

### 5A. Deliverables (in order)

1. Step 0 discovery summary (already reviewed).
1. New agent files following existing structure.
1. Category 1 module (8 capabilities) + payer healthcare layer (9 tools), registered via the existing mechanism, each tagged with its mode (LLM / DETERMINISTIC).
1. Test suite following the existing agent test structure (see 5C).
1. README/CHANGELOG entry explicitly listing which existing patterns were reused (state, retrieval, tooling, LLM client, memory, guardrails, tests) and which external data sources remain stubbed.

### 5B. Structural self-review checklist (must all be true)

- [ ] No new framework or architectural pattern introduced; mirrors existing agents.
- [ ] Existing RDS retriever reused, not rewritten.
- [ ] All 8 Category 1 capabilities and all 9 payer healthcare tools implemented, registered, and mode-tagged.
- [ ] Hybrid integrity honored: numeric tools deterministic; LLM does not originate figures.
- [ ] Every healthcare tool is payer-perspective; no perspective parameter exists anywhere.
- [ ] Compliance tools (4, 5) are structurally gated and cannot emit non-compliant or leverage-framed suggestions.
- [ ] PHI guardrail present, using/extending the existing pattern.
- [ ] Model ID is configurable; no API key or secret is hardcoded.
- [ ] No unbounded agent loops; LLM tools fail gracefully.
- [ ] Stubbed external data sources (transparency files, claims, MGMA/Medicare reference) clearly flagged.
- [ ] No volatile real-world fact (rates, state rules, benchmarks, VBC assumptions) is hardcoded in tool code; all live in versioned external data with source + effective date + as-of timestamp.
- [ ] Every figure-bearing tool surfaces the provenance and as-of date of the data it used.
- [ ] Per-source freshness SLA enforced at use time; dependent tools fail closed (refuse, naming source + age) when data is stale/missing/unverifiable; no silent cached or model-recalled fallback.
- [ ] Source validation enforced before use: per-field sanity bounds, cross-source agreement check where multiple sources exist, and schema/units/completeness checks; invalid data fails closed (scoped to dependent tools) with no silent fallback.
- [ ] Cross-source conflicts beyond tolerance are surfaced (both values + sources + as-of dates), never silently resolved; validation outcome is logged and surfaced where it affects a figure.
- [ ] Grounding gate enforced: retrieval-dependent tools run retrieval before the LLM and short-circuit (LLM never called) when the store is empty or no chunk clears the relevance threshold; no tool fills an absent-data gap from model training knowledge.
- [ ] Grounded outputs carry citations to retrieved chunks; uncited claims are dropped/flagged in synthesis.
- [ ] Output distinguishes “data not loaded/retrieved” from “contract is silent on X” from a substantive grounded answer; an absent store is never presented as “silent on X” or as a confident answer.

### 5C. FORMAL ACCEPTANCE TESTS (must pass before the agent is considered done)

Implement these as automated tests in the repo’s existing test framework. **Deterministic tests run with no API key. Live-LLM tests require a real model call and must be run where credentials are configured (e.g., Claude Code / CI with the secret set); they must SKIP cleanly — not fail — when no key is present.** Use a seeded test contract containing known payer-unfavorable terms (above-market rate, compounding escalator, deemed-PA-approval, retroactive-denial ban including fraud, behavioral-health rate + prior-auth disparity, short recoupment window, physician-binding medical-necessity, asymmetric immediate termination, auto-renewal, missing balance-billing/good-faith-estimate language).

**Group 1 — Deterministic tool correctness (no key required):**

- [ ] AT-1 Rate Benchmarker labels the above-market rate as high overpayment and cites the peer-payer transparency delta.
- [ ] AT-2 Escalator Analyzer flags the compounding escalator AND preserves a current-year Medicare basis as payer-favorable.
- [ ] AT-3 Termination Assessor flags asymmetric immediate termination and auto-renewal, and scores member-disruption from member share.
- [ ] AT-4 State Overlay returns the correct payer obligation/constraint pair for the test state.
- [ ] AT-5 VBC Modeler returns internally consistent arithmetic (payer-keep + hospital-share == projected gross savings).

**Group 2 — LLM tool behavior (live model; SKIP without key):**

- [ ] AT-6 Each LLM tool returns valid structured output that parses (no stub/error result) when a key is present.
- [ ] AT-7 **Payer perspective holds:** no LLM tool output recommends higher hospital reimbursement or concessions that only benefit the hospital (assert against a forbidden-phrase set AND a semantic check).
- [ ] AT-8 **Parity (gated):** the Parity Checker catches BOTH the behavioral-health reimbursement disparity AND the non-quantitative prior-auth disparity, carries the gated flag, and does not frame either as leverage to retain.
- [ ] AT-9 **NSA (gated):** the NSA Scanner flags the missing balance-billing/good-faith-estimate protections, carries the gated flag, and recommends only toward compliance.
- [ ] AT-10 **Prior-Auth:** the analyzer flags the deemed-approval trigger and/or the fraud-inclusive retroactive-denial ban, and proposes payer-protective language.
- [ ] AT-11 **Redline:** the generator returns concrete proposed clause language that is payer-protective for the targeted clause.
- [ ] AT-12 **No invented numbers:** LLM tool outputs contain no benchmark percentages or dollar figures not present in the provided contract/context (assert numeric tokens in LLM output trace to inputs).

**Group 3 — Graph & resilience (no key required):**

- [ ] AT-13 The compiled graph runs end-to-end on a full-review query and returns a synthesized payer-side analysis.
- [ ] AT-14 Query routing dispatches a targeted query to the correct single tool.
- [ ] AT-15 With no key, LLM tools return a graceful, clearly-labeled unavailable result and the graph still completes (no crash, no infinite loop).
- [ ] AT-16 PHI guardrail: a contract containing PHI does not transmit raw PHI to the model (verify via the guardrail/redaction path).

**Group 4 — Data freshness & sourcing (no key required):**

- [ ] AT-17 No volatile real-world fact is a code literal: a scan/test confirms rates, state rules, benchmarks, and the VBC assumption are loaded from external data, not hardcoded.
- [ ] AT-18 Provenance present: every figure-bearing tool result includes the source identifier and the data’s effective/as-of date.
- [ ] AT-19 **Fail-closed on stale data:** with a reference dataset artificially aged past its freshness SLA, the dependent tool refuses to produce a figure-bearing recommendation and returns a clear stale-data result naming the source and its age — and does NOT fall back to a cached or model-recalled number.
- [ ] AT-20 **Scoped refusal:** under the same stale-data condition, tools with no dependency on that dataset (e.g., Multi-Version Diff, Plain-Language Summary) still complete normally, and the graph does not crash.
- [ ] AT-21 Use-time check: freshness is evaluated at invocation (not only at load); a dataset that passes at load but is expired at call time is rejected. Expiry Tracker computes against the current date dynamically.

**Group 5 — Source validation (no key required):**

- [ ] AT-22 **Sanity bounds:** a reference value injected outside its configured plausibility bounds causes the dependent tool to fail closed and flag the source — it does not emit the implausible figure as a finding.
- [ ] AT-23 **Cross-source agreement:** when two sources for the same fact agree within tolerance, the tool proceeds and records the concurring sources; when they disagree beyond tolerance, the tool surfaces both values with sources and as-of dates and does not silently pick one.
- [ ] AT-24 **Schema/integrity:** malformed reference data (missing required field, wrong unit, null where disallowed) is rejected and the dependent tool fails closed rather than coercing it.
- [ ] AT-25 **Scoped + no fallback:** under a validation failure, only the dependent tools refuse; unrelated tools complete; and no tool substitutes a cached, default, or model-recalled value to bypass the failure.

**Group 6 — Grounding gate / no-fabrication on absent data (mostly no key required):**

- [ ] AT-26 **Empty store fails closed before the LLM:** with an empty vector store, every retrieval-dependent tool returns the “insufficient grounding data” result and the LLM is **never invoked** — assert via a mock/spy LLM client that records zero calls for those tools.
- [ ] AT-27 **Irrelevant store refuses:** with a store populated only with chunks irrelevant to the query (none clearing the relevance threshold), the tool refuses rather than answering.
- [ ] AT-28 **Citations required:** with a populated store, every grounded claim in tool output carries a citation to a retrieved chunk; an injected uncited claim is dropped or flagged by synthesis. (Live-LLM portion SKIPs without a key; the citation-enforcement logic is tested deterministically.)
- [ ] AT-29 **State disambiguation:** the “data not loaded/retrieved” message is distinct in the output schema from a substantive “the contract is silent on X” answer; a test confirms an absent store yields state (a), not state (b) or a confident answer.

**Definition of done:** every item in 5B checked; Groups 1, 3, 4, 5, and 6 tests green in CI without a key (the live-LLM portions of Groups 2 and 6 SKIP cleanly without a key and must pass with credentials configured). Any failing Group 2 assertion is resolved by prompt/guardrail tuning, not by weakening the assertion. The following are **hard gates that must never be relaxed** to let a bad answer through: fail-closed freshness (AT-19), validation fail-closed (AT-22–AT-25), and the grounding gate (AT-26–AT-27, AT-29).

-----

## Working agreement

- Pause for review after Step 0 and Step 1.
- Prefer the repo’s existing way over any cleaner-from-scratch approach.
- Verify model IDs and the structured-output API against current Anthropic docs before finalizing the LLM client; do not rely on memory.