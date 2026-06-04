# Reference Implementation Appendix — Negotiations Agent (Sandbox Build)

> **Status:** working, structurally tested sandbox code. This is the **canonical
> behavioral reference** for the Kiro build. It is NOT a benchmarked or externally
> validated design — Category 1 and the healthcare layer originate from the user’s
> prior conversations, not from any benchmarked source.
> 
> **How Kiro must use this appendix:**
> 
> - Treat this code as the source of truth for **what each tool does, its inputs/
>   outputs, the result schema, the payer system prompt, the compliance gating, and
>   the hybrid LLM/deterministic split.**
> - Do NOT copy this file/module structure verbatim. **Structure (state schema,
>   tool registration, retriever, LLM client, persistence, tests) must follow the
>   existing repo patterns discovered in Step 0.** Where this code’s structure and
>   the repo’s conventions differ, the repo wins; preserve the *behavior* shown here.
> - The mock retriever and hardcoded reference data here are stand-ins. Per Step 4,
>   replace them with the production retriever and externalized, freshness-checked,
>   validated data sources. The hardcoded numbers in `reference_data.py` are sample
>   values only and MUST NOT ship as code literals.
> - **Required NEW work not present in this sandbox reference** (build per Step 4 +
>   acceptance tests): (1) data freshness fail-closed (AT-17–AT-21); (2) source
>   validation — sanity bounds, cross-source agreement, schema checks (AT-22–AT-25);
>   (3) the grounding gate — pre-LLM retrieval check, citation-required output, and
>   “no data” vs “silent on X” disambiguation (AT-26–AT-29). The sandbox mock
>   retriever always returns chunks, so it does NOT exercise the empty/irrelevant-store
>   path; that gate must be added and tested against the production retriever.
> - Live-LLM calls in this sandbox were verified against a mocked SDK response only
>   (no API key in the sandbox); run the live tests in Claude Code / CI with a key.

-----

## `negotiations_agent/agent.py`

```python
"""
Negotiations Agent — LangGraph graph.
Payer-perspective contract negotiation assistant.
Structured as: route -> run_tools -> synthesize, with a checkpointer for state.
(Mirrors a production agent skeleton; deterministic tools keep the test key-free.)
"""
from typing import TypedDict, List, Dict, Any
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import InMemorySaver
from .tools.payer_healthcare_hybrid import PAYER_TOOLS, TOOL_MODES
from .reference_data import RETRIEVER


class NegotiationState(TypedDict):
    query: str
    contract_text: str
    state_code: str
    target_clause: str
    selected_tools: List[str]
    tool_results: Dict[str, Any]
    answer: str


# Simple intent router: maps query keywords to payer tools.
ROUTING = [
    (("risk", "riskiest", "exposure", "worst clause", "score"), "clause_risk_scorer"),
    (("redline", "rewrite", "alternative", "draft", "push back on"), "redline_generator"),
    (("rate", "reimburse", "310", "percent of medicare", "overpay"), "rate_benchmarker"),
    (("escalator", "basis", "compounding", "annual increase"), "rate_basis_escalator"),
    (("prior auth", "authorization", "utilization"), "prior_auth_analyzer"),
    (("parity", "behavioral", "mental health", "mhpaea"), "parity_checker"),
    (("surprise", "balance billing", "nsa"), "nsa_scanner"),
    (("clean claim", "recoup", "audit", "medical necessity", "language"), "language_optimizer"),
    (("value based", "shared savings", "vbc", "risk corridor"), "vbc_modeler"),
    (("terminat", "renew", "network adequacy", "disruption"), "termination_risk"),
    (("state", "texas", "regulatory", "prompt pay"), "state_overlay"),
]


def route(state: NegotiationState) -> NegotiationState:
    q = state["query"].lower()
    selected = [tool for keys, tool in ROUTING if any(k in q for k in keys)]
    if not selected:  # full review fallback
        selected = list(PAYER_TOOLS.keys())
    return {**state, "selected_tools": selected}


def run_tools(state: NegotiationState) -> NegotiationState:
    ctext = state["contract_text"]
    results = {}
    for name in state["selected_tools"]:
        fn = PAYER_TOOLS[name]
        if name == "rate_benchmarker":
            results[name] = fn("inpatient", 310, "high")
        elif name == "vbc_modeler":
            results[name] = fn(50, "readmission<10%", 12000)
        elif name == "termination_risk":
            results[name] = fn(ctext, hospital_is_sole_specialist=False, member_share=0.18)
        elif name == "state_overlay":
            results[name] = fn(state.get("state_code", "TX"))
        elif name == "redline_generator":
            results[name] = fn(ctext, state.get("target_clause", "Section 3.3 automatic 5% escalator"))
        elif name == "clause_risk_scorer":
            results[name] = fn(ctext, state.get("target_clause", ""))
        else:  # prior_auth_analyzer, parity_checker, nsa_scanner, language_optimizer, rate_basis_escalator
            results[name] = fn(ctext)
    return {**state, "tool_results": results}


def synthesize(state: NegotiationState) -> NegotiationState:
    lines = ["NEGOTIATIONS AGENT — PAYER-SIDE ANALYSIS", "=" * 44]
    for name, r in state["tool_results"].items():
        mode = TOOL_MODES.get(name, "?")
        lines.append(f"\n[{name}]  ({mode})  risk: {r.get('payer_risk')}")
        lines.append(f"  {r.get('finding')}")
        for a in r.get("leverage_or_action", []):
            lines.append(f"   - {a}")
        if r.get("compliance_gated"):
            lines.append("   (compliance-gated: recommendations move toward legal compliance only)")
    return {**state, "answer": "\n".join(lines)}


def build_agent():
    g = StateGraph(NegotiationState)
    g.add_node("route", route)
    g.add_node("run_tools", run_tools)
    g.add_node("synthesize", synthesize)
    g.add_edge(START, "route")
    g.add_edge("route", "run_tools")
    g.add_edge("run_tools", "synthesize")
    g.add_edge("synthesize", END)
    return g.compile(checkpointer=InMemorySaver())


def ask(agent, query, state_code="TX", target_clause="", thread="t1"):
    ctext = open("/home/claude/negotiations_agent/data/sample_contract.txt").read()
    out = agent.invoke(
        {"query": query, "contract_text": ctext, "state_code": state_code,
         "target_clause": target_clause, "selected_tools": [], "tool_results": {}, "answer": ""},
        config={"configurable": {"thread_id": thread}},
    )
    return out["answer"]
```

-----

## `negotiations_agent/llm_client.py`

```python
"""
LLM client for the judgment-heavy payer tools.
Uses the Anthropic SDK. Reads ANTHROPIC_API_KEY from the environment
(set automatically in Claude Code). No key is ever hardcoded.

Returns structured JSON via tool-use forcing so downstream code parses reliably.
"""
import os
import json

try:
    from anthropic import Anthropic
except ImportError:  # surfaced clearly rather than failing cryptically
    Anthropic = None

MODEL = os.environ.get("NEGOTIATIONS_MODEL", "claude-sonnet-4-6")

# Shared schema every judgment tool returns. Keeps output uniform with the
# deterministic tools: {finding, payer_risk, leverage_or_action[]}.
RESULT_SCHEMA = {
    "type": "object",
    "properties": {
        "finding": {"type": "string", "description": "One-line summary of what was analyzed."},
        "payer_risk": {"type": "string", "description": "Risk level/label from the PAYER's perspective."},
        "leverage_or_action": {
            "type": "array", "items": {"type": "string"},
            "description": "Concrete payer-side leverage points or recommended actions.",
        },
    },
    "required": ["finding", "payer_risk", "leverage_or_action"],
}

# Hard constraint applied to every judgment tool. This is the payer-perspective
# guardrail enforced at the prompt layer; tests assert it holds in output.
PAYER_SYSTEM = (
    "You are a contract-negotiation analyst working SOLELY for the health "
    "insurance company (the PAYER) negotiating with a hospital. Every analysis "
    "is from the payer's side: protect the payer's cost position, surface the "
    "payer's leverage, and flag the payer's own regulatory liability. NEVER "
    "argue for higher hospital reimbursement or recommend concessions that only "
    "benefit the hospital. Ground every claim in the provided contract text; do "
    "not invent figures, statutes, or benchmark numbers. If a number is needed "
    "and not provided, say so rather than guessing."
)


class LLMUnavailable(RuntimeError):
    pass


def llm_analyze(task_instructions: str, contract_excerpt: str, extra_context: str = "") -> dict:
    """Run one judgment tool. Forces structured JSON output via tool-use."""
    if Anthropic is None:
        raise LLMUnavailable("anthropic SDK not installed (pip install anthropic).")
    if not os.environ.get("ANTHROPIC_API_KEY"):
        raise LLMUnavailable("ANTHROPIC_API_KEY not set in environment.")

    client = Anthropic()
    user_content = (
        f"{task_instructions}\n\n"
        f"--- CONTRACT TEXT ---\n{contract_excerpt}\n"
        + (f"\n--- ADDITIONAL CONTEXT ---\n{extra_context}\n" if extra_context else "")
    )
    resp = client.messages.create(
        model=MODEL,
        max_tokens=1024,
        system=PAYER_SYSTEM,
        tools=[{
            "name": "report",
            "description": "Report the payer-side analysis in structured form.",
            "input_schema": RESULT_SCHEMA,
        }],
        tool_choice={"type": "tool", "name": "report"},
        messages=[{"role": "user", "content": user_content}],
    )
    for block in resp.content:
        if block.type == "tool_use" and block.name == "report":
            return dict(block.input)
    raise LLMUnavailable("Model did not return structured report output.")
```

-----

## `negotiations_agent/tools/payer_healthcare_hybrid.py`

```python
"""
Hybrid payer-perspective tool layer.

MODE per tool:
  LLM           -> judgment/legal-language reasoning (clause analysis, redlines)
  DETERMINISTIC -> math & reference lookups (rates, escalators, VBC, state rules)

Rationale: numeric tools must NOT be LLM-generated — hallucinated benchmark
percentages are exactly the failure a payer negotiation can't tolerate. The LLM
reasons over language and cites the contract; the math stays deterministic.

Each LLM tool degrades gracefully: if no key/SDK, it returns a clear stub telling
the caller to run in Claude Code, so the graph never crashes during offline dev.
"""
import re
from ..reference_data import MARKET_BENCHMARKS, TRANSPARENCY_FILE, STATE_RULES
from ..llm_client import llm_analyze, LLMUnavailable


def _num(pattern, text, default=None):
    m = re.search(pattern, text, re.I | re.S)
    return float(m.group(1)) if m else default


def _llm_or_stub(instructions, contract_text, extra=""):
    try:
        return llm_analyze(instructions, contract_text, extra)
    except LLMUnavailable as e:
        return {
            "finding": "[LLM unavailable — run in Claude Code with ANTHROPIC_API_KEY]",
            "payer_risk": "unknown (offline)",
            "leverage_or_action": [f"Could not run live analysis: {e}"],
            "_stub": True,
        }


# ───────────────────────── LLM-BACKED (judgment) ─────────────────────────

def clause_risk_scorer(contract_text, clause_hint=""):  # MODE: LLM
    return _llm_or_stub(
        "Identify the highest-risk clauses for the PAYER in this contract. For each, "
        "give a risk label and explain the payer's exposure. Focus on clauses that shift "
        "cost or legal risk onto the payer." + (f" Focus area: {clause_hint}." if clause_hint else ""),
        contract_text)


def redline_generator(contract_text, target_clause):  # MODE: LLM
    return _llm_or_stub(
        f"The payer wants to push back on this clause: '{target_clause}'. Draft a "
        "payer-protective alternative and explain why it improves the payer's position. "
        "Put the proposed redline language in leverage_or_action.",
        contract_text)


def prior_auth_analyzer(contract_text):  # MODE: LLM
    return _llm_or_stub(
        "Analyze the prior-authorization section for whether it preserves the PAYER's "
        "utilization-management authority. Flag auto-approval triggers, aggressive turnaround "
        "requirements, and blanket retroactive-denial bans. Suggest payer-protective language, "
        "bounded by lawful prior-auth reform limits.",
        contract_text)


def parity_checker(contract_text):  # MODE: LLM (compliance-gated)
    r = _llm_or_stub(
        "MHPAEA parity review. The PAYER is the regulated party and bears liability. "
        "Compare behavioral-health terms to comparable medical/surgical terms (reimbursement "
        "rates AND non-quantitative treatment limits like prior-auth). Flag every disparity as "
        "a payer compliance liability. COMPLIANCE-GATED: only recommend changes that move TOWARD "
        "legal parity; never frame a disparity as payer leverage to keep.",
        contract_text)
    r["compliance_gated"] = True
    return r


def nsa_scanner(contract_text):  # MODE: LLM (compliance-gated)
    r = _llm_or_stub(
        "No Surprises Act compliance scan. Flag missing balance-billing/hold-harmless protection, "
        "missing good-faith-estimate obligations, and OON emergency gaps that create PAYER liability. "
        "COMPLIANCE-GATED: recommend only toward legal compliance.",
        contract_text)
    r["compliance_gated"] = True
    return r


def language_optimizer(contract_text):  # MODE: LLM
    return _llm_or_stub(
        "Review operational clauses that govern effective cost regardless of headline rate: "
        "clean-claim definition, downcoding/bundling rights, medical-necessity discretion, "
        "recoupment/audit windows, appeal timelines. Flag hospital-favorable versions and draft "
        "payer-protective alternatives.",
        contract_text)


# ───────────────────────── DETERMINISTIC (math/lookup) ─────────────────────────

def rate_benchmarker(service, proposed_pct_medicare, payer_volume="high"):  # MODE: DETERMINISTIC
    b = MARKET_BENCHMARKS.get(service, {})
    avg = b.get("medicare_pct_avg")
    trans = TRANSPARENCY_FILE.get(service, {})
    comp = [v for k, v in trans.items() if isinstance(v, (int, float))]
    comp_avg = round(sum(comp) / len(comp), 1) if comp else None
    over_market = proposed_pct_medicare - avg if avg else None
    leverage = []
    if over_market and over_market > 0:
        leverage.append(f"{over_market:.0f} pts above national avg ({avg}%); overpayment signal.")
    if comp_avg and proposed_pct_medicare > comp_avg:
        leverage.append(f"Other payers pay this hospital ~{comp_avg}% (transparency file); "
                        f"payer asked for {proposed_pct_medicare - comp_avg:.0f} pts more than peers.")
    if payer_volume == "high":
        leverage.append("High regional membership = strong buyer leverage toward peer level.")
    return {"finding": f"Proposed {service} rate = {proposed_pct_medicare}% of Medicare.",
            "payer_risk": "HIGH overpayment" if (over_market or 0) > 30 else "moderate",
            "leverage_or_action": leverage or ["Rate at/below market; limited room to push."]}


def rate_basis_escalator(contract_text):  # MODE: DETERMINISTIC
    flags = []
    if re.search(r"current[- ]year", contract_text, re.I):
        flags.append("Rate pegged to CURRENT-year Medicare — favorable to payer; keep.")
    if re.search(r"fixed[- ]?year|specific year", contract_text, re.I):
        flags.append("Fixed-year Medicare basis favors hospital; resist.")
    esc = _num(r"fixed\s+\w+\s*\(?(\d+)\s*%?\)?\s*(?:percent)?\s*each contract year", contract_text) \
        or _num(r"increase by[^0-9]*(\d+)\s*%", contract_text)
    if esc:
        flags.append(f"Automatic {esc:.0f}% compounding escalator on top of Medicare updates — "
                     f"double-counts inflation; strike or cap.")
    return {"finding": "Rate-basis and escalator review.",
            "payer_risk": "HIGH multi-year cost growth" if esc and esc >= 4 else "moderate",
            "leverage_or_action": flags or ["No problematic escalator/basis found."]}


def vbc_modeler(shared_savings_pct, quality_threshold, panel_size, baseline_pmpy=9000):  # MODE: DETERMINISTIC
    baseline = panel_size * baseline_pmpy
    gross = baseline * 0.04
    keep = gross * (1 - shared_savings_pct / 100)
    return {"finding": f"VBC model: panel {panel_size}, baseline ${baseline:,.0f}.",
            "payer_risk": "downside exposure if savings unrealized; start upside-only",
            "leverage_or_action": [
                f"Projected gross savings ~${gross:,.0f} (4% assumption).",
                f"Payer retains ~${keep:,.0f}; hospital share ~${gross - keep:,.0f} at {shared_savings_pct}%.",
                f"Tie hospital share to quality threshold {quality_threshold}."]}


def termination_risk(contract_text, hospital_is_sole_specialist=False, member_share=0.18):  # MODE: DETERMINISTIC
    flags = []
    if re.search(r"automatically renew", contract_text, re.I):
        flags.append("Auto-renewal locks payer in; add active-renewal trigger.")
    if re.search(r"Hospital may terminate immediately", contract_text, re.I):
        flags.append("Hospital can terminate IMMEDIATELY on dispute — asymmetric; require equal notice.")
    notice = _num(r"(\d+)\s*days'? written notice", contract_text)
    if notice:
        flags.append(f"{notice:.0f}-day notice; standard 90-120 days — make mutual.")
    disruption = "HIGH" if hospital_is_sole_specialist else ("MODERATE" if member_share > 0.15 else "LOW")
    return {"finding": "Termination / network-adequacy disruption assessment (payer view).",
            "payer_risk": f"member disruption: {disruption} ({member_share:.0%} of regional members)",
            "leverage_or_action": flags + [
                "If hospital is sole regional specialist, protect continuity-of-care.",
                "Account for hospital-initiated MA termination trend over payment disputes."]}


def state_overlay(state):  # MODE: DETERMINISTIC
    r = STATE_RULES.get(state.upper())
    if not r:
        return {"finding": f"No rules loaded for {state}.", "payer_risk": "unknown", "leverage_or_action": []}
    return {"finding": f"{state} regulatory overlay.",
            "payer_risk": f"prompt-pay obligation: {r['prompt_pay_days']} days (payer exposure).",
            "leverage_or_action": [
                f"Any-willing-provider: {'YES — limits payer exclusion ability' if r['any_willing_provider'] else 'NO — payer may use selective contracting as leverage'}.",
                f"Prior-auth reform: {r['prior_auth_reform']}"]}


TOOL_MODES = {
    "clause_risk_scorer": "LLM", "redline_generator": "LLM", "prior_auth_analyzer": "LLM",
    "parity_checker": "LLM", "nsa_scanner": "LLM", "language_optimizer": "LLM",
    "rate_benchmarker": "DETERMINISTIC", "rate_basis_escalator": "DETERMINISTIC",
    "vbc_modeler": "DETERMINISTIC", "termination_risk": "DETERMINISTIC", "state_overlay": "DETERMINISTIC",
}

PAYER_TOOLS = {
    "clause_risk_scorer": clause_risk_scorer, "redline_generator": redline_generator,
    "prior_auth_analyzer": prior_auth_analyzer, "parity_checker": parity_checker,
    "nsa_scanner": nsa_scanner, "language_optimizer": language_optimizer,
    "rate_benchmarker": rate_benchmarker, "rate_basis_escalator": rate_basis_escalator,
    "vbc_modeler": vbc_modeler, "termination_risk": termination_risk, "state_overlay": state_overlay,
}
```

-----

## `negotiations_agent/reference_data.py`

```python
"""
Reference data + a mock retriever that stands in for the existing RDS vector store.
In the real repo, RETRIEVER would be the production retriever discovered in Step 0.
Benchmark figures are grounded in real 2024-2025 public sources (Milliman, RAND, CBO).
"""
import os, re

# --- Real-world commercial-rate-as-%-of-Medicare benchmarks (citable) ---
# Milliman 2025: IP avg 209%, OP avg 263%, Prof 148% of Medicare FFS (nationwide).
# RAND Round 5 (2022 data): IP facility 254%, OP facility 279% of Medicare.
MARKET_BENCHMARKS = {
    "inpatient":  {"medicare_pct_avg": 209, "medicare_pct_high": 254, "source": "Milliman 2025 / RAND R5"},
    "outpatient": {"medicare_pct_avg": 263, "medicare_pct_high": 279, "source": "Milliman 2025 / RAND R5"},
    "professional": {"medicare_pct_avg": 148, "medicare_pct_high": 184, "source": "Milliman 2025 / RAND R5"},
}

# Mock "price transparency file" — what other payers reportedly pay THIS hospital.
# Illustrates real finding: large payer-to-payer spread at the same hospital.
TRANSPARENCY_FILE = {
    "inpatient":  {"UHC": 245, "Aetna": 230, "BCBS": 265, "unit": "% of Medicare"},
    "outpatient": {"UHC": 260, "Aetna": 250, "BCBS": 285, "unit": "% of Medicare"},
}

# State regulatory facts (mock subset).
STATE_RULES = {
    "TX": {
        "prompt_pay_days": 30,
        "any_willing_provider": False,
        "prior_auth_reform": "TX 'gold card' law exempts high-approval providers from PA.",
    }
}


class MockRetriever:
    """Stands in for the production RDS vector retriever (similarity_search-like API)."""
    def __init__(self, path):
        with open(path) as f:
            self.text = f.read()
        # crude section chunking; real store has embedded chunks already
        self.chunks = re.split(r"\n(?=SECTION )", self.text)

    def get_relevant_chunks(self, query, k=3):
        q = set(re.findall(r"[a-z]+", query.lower()))
        scored = []
        for c in self.chunks:
            words = set(re.findall(r"[a-z]+", c.lower()))
            scored.append((len(q & words), c))
        scored.sort(key=lambda x: -x[0])
        return [c for _, c in scored[:k]]


RETRIEVER = MockRetriever(os.path.join(os.path.dirname(__file__), "data", "sample_contract.txt"))
```

-----

## `negotiations_agent/tests/test_payer_tools.py`

```python
"""Tests: each tool must catch planted payer-unfavorable terms; perspective must be payer."""
import sys
from negotiations_agent.agent import build_agent, ask
from negotiations_agent.tools.payer_healthcare_hybrid import PAYER_TOOLS
from negotiations_agent.reference_data import RETRIEVER

CONTRACT = open("/home/claude/negotiations_agent/data/sample_contract.txt").read()
results, failures = [], []

def check(name, cond, detail=""):
    results.append((name, cond))
    if not cond:
        failures.append(f"{name}: {detail}")

# 1 rate benchmarker flags 310% as high overpayment vs 209% avg
r = PAYER_TOOLS["rate_benchmarker"]("inpatient", 310, "high")
check("rate: high overpayment", r["payer_risk"] == "HIGH overpayment", r["payer_risk"])
check("rate: cites peer transparency", any("transparency" in x for x in r["leverage_or_action"]))

# 2 escalator caught
r = PAYER_TOOLS["rate_basis_escalator"](CONTRACT)
check("escalator: 5% compounding flagged", any("compounding escalator" in x for x in r["leverage_or_action"]))
check("escalator: current-year kept as payer-favorable", any("favorable to payer" in x for x in r["leverage_or_action"]))

# 3 prior auth: deemed approval + fraud ban
r = PAYER_TOOLS["prior_auth_analyzer"](CONTRACT)
check("PA: auto-approval flagged", any("Auto-approval" in x for x in r["leverage_or_action"]))
check("PA: fraud carveout flagged", any("fraud" in x.lower() for x in r["leverage_or_action"]))

# 4 parity: must catch BOTH 90% rate AND PA-after-3-visits
r = PAYER_TOOLS["parity_checker"](CONTRACT)
check("parity: BH rate 90% disparity caught", any("90% of medical" in x for x in r["leverage_or_action"]))
check("parity: NQTL PA disparity caught", any("NQTL" in x for x in r["leverage_or_action"]))
check("parity: gated", r.get("compliance_gated") is True)

# 5 NSA
r = PAYER_TOOLS["nsa_scanner"](CONTRACT)
check("nsa: balance billing gap", any("balance-billing" in x for x in r["leverage_or_action"]))
check("nsa: gated", r.get("compliance_gated") is True)

# 6 language optimizer: must catch ALL THREE planted clauses
r = PAYER_TOOLS["language_optimizer"](CONTRACT)
check("lang: clean claim caught", any("clean claim" in x.lower() for x in r["leverage_or_action"]))
check("lang: recoup window caught", any("recoupment" in x.lower() or "30-day" in x for x in r["leverage_or_action"]))
check("lang: med necessity caught", any("medical-necessity" in x.lower() for x in r["leverage_or_action"]))

# 8 termination
r = PAYER_TOOLS["termination_risk"](CONTRACT, False, 0.18)
check("term: asymmetric immediate termination", any("IMMEDIATELY" in x for x in r["leverage_or_action"]))
check("term: auto-renewal", any("Auto-renewal" in x for x in r["leverage_or_action"]))

# perspective guard: no tool should ever recommend RAISING the hospital rate
agent = build_agent()
full = ask(agent, "review full contract from payer side")
check("perspective: never argues for higher hospital rate",
      "higher reimbursement" not in full.lower() and "raise the rate" not in full.lower())

print(f"\nPASSED {sum(1 for _,c in results if c)}/{len(results)}")
for name, c in results:
    print(("  ok  " if c else " FAIL ") + name)
if failures:
    print("\nFAILURES:")
    for f in failures: print("  -", f)
    sys.exit(1)
```

-----

## `negotiations_agent/tests/test_live_llm.py`

```python
"""
LIVE LLM TEST — run this inside Claude Code (where ANTHROPIC_API_KEY is set).

    python -m negotiations_agent.tests.test_live_llm

It calls the real model for the six judgment tools and asserts:
  1. structured output parses (no stubs),
  2. payer perspective holds (never argues for higher hospital pay),
  3. compliance-gated tools (parity, NSA) only recommend toward compliance,
  4. the LLM actually catches the planted issues a payer cares about.

Deterministic tools are checked separately in test_payer_tools.py and need no key.
"""
import os
import sys

CONTRACT = open(os.path.join(os.path.dirname(__file__), "..", "data", "sample_contract.txt")).read()


def main():
    if not os.environ.get("ANTHROPIC_API_KEY"):
        print("SKIP: ANTHROPIC_API_KEY not set. Run this in Claude Code.")
        sys.exit(2)

    from negotiations_agent.tools.payer_healthcare_hybrid import (
        clause_risk_scorer, prior_auth_analyzer, parity_checker, nsa_scanner,
        language_optimizer, redline_generator,
    )

    results, failures = [], []

    def check(name, cond, detail=""):
        results.append((name, cond))
        if not cond:
            failures.append(f"{name}: {detail}")

    def text_of(r):
        return (r.get("finding", "") + " " + " ".join(r.get("leverage_or_action", []))).lower()

    FORBIDDEN = ["higher reimbursement", "raise the rate", "increase the hospital", "in the hospital's favor"]

    # clause risk
    r = clause_risk_scorer(CONTRACT)
    check("clause: live (not stub)", not r.get("_stub"), str(r))
    check("clause: payer perspective", not any(f in text_of(r) for f in FORBIDDEN), text_of(r)[:200])

    # prior auth — should catch deemed-approval and/or fraud carveout
    r = prior_auth_analyzer(CONTRACT)
    check("PA: live", not r.get("_stub"))
    check("PA: catches auto-approval or fraud", any(k in text_of(r) for k in ["approval", "fraud", "retro"]), text_of(r)[:200])

    # parity — must catch BH disparity AND be gated AND not suggest keeping disparity as leverage
    r = parity_checker(CONTRACT)
    check("parity: live", not r.get("_stub"))
    check("parity: gated flag", r.get("compliance_gated") is True)
    check("parity: catches behavioral disparity", "behavioral" in text_of(r) or "parity" in text_of(r), text_of(r)[:200])
    check("parity: no 'keep disparity as leverage'", "leverage" not in text_of(r) or "compliance" in text_of(r), text_of(r)[:200])

    # NSA gated
    r = nsa_scanner(CONTRACT)
    check("nsa: live", not r.get("_stub"))
    check("nsa: gated flag", r.get("compliance_gated") is True)

    # language optimizer — payer cost-control clauses
    r = language_optimizer(CONTRACT)
    check("lang: live", not r.get("_stub"))
    check("lang: payer perspective", not any(f in text_of(r) for f in FORBIDDEN))

    # redline — must produce proposed language, payer-protective
    r = redline_generator(CONTRACT, "Section 3.3 automatic 5% compounding escalator")
    check("redline: live", not r.get("_stub"))
    check("redline: payer perspective", not any(f in text_of(r) for f in FORBIDDEN), text_of(r)[:200])

    print(f"\nLIVE LLM: PASSED {sum(1 for _, c in results if c)}/{len(results)}")
    for name, c in results:
        print(("  ok  " if c else " FAIL ") + name)
    if failures:
        print("\nFAILURES:")
        for f in failures:
            print("  -", f)
        sys.exit(1)


if __name__ == "__main__":
    main()
```

-----

## `negotiations_agent/data/sample_contract.txt`

```text
MANAGED CARE PARTICIPATION AGREEMENT

This Agreement is entered into between Meridian Health Plan, Inc. ("Plan") and
Riverside Regional Medical Center ("Hospital"), effective January 1, 2026.

SECTION 3. REIMBURSEMENT.
3.1 Inpatient Services. Plan shall reimburse Hospital for covered inpatient
services at three hundred ten percent (310%) of the then-current Medicare
fee-for-service rate (current-year basis), updated automatically each January 1
as CMS publishes revised rates.
3.2 Outpatient Services. Plan shall reimburse Hospital at two hundred ninety
percent (290%) of the current-year Medicare OPPS rate.
3.3 Annual Escalator. In addition to the Medicare-based updates in 3.1 and 3.2,
all rates shall increase by a fixed five percent (5%) each contract year,
compounding, for the term of this Agreement.

SECTION 4. CLAIMS.
4.1 Clean Claim. A "clean claim" means any claim submitted by Hospital. Plan
shall pay all submitted claims within sixty (60) days.
4.2 Retroactive Adjustment. Plan may not adjust, recoup, or audit any paid claim
more than thirty (30) days after payment.
4.3 Medical Necessity. Determinations of medical necessity shall be made solely
by the treating physician at Hospital, and such determination shall be binding
on Plan.

SECTION 5. PRIOR AUTHORIZATION.
5.1 Plan shall render any prior authorization determination within two (2)
business days of request. Failure to respond within two business days shall be
deemed an approval.
5.2 Plan shall not retroactively deny any service for which prior authorization
was granted, under any circumstances, including fraud.

SECTION 6. BEHAVIORAL HEALTH.
6.1 Behavioral health and substance use disorder services shall be reimbursed at
ninety percent (90%) of the corresponding medical/surgical rate. Prior
authorization for behavioral health services shall be required for all visits
beyond the third visit, whereas medical/surgical services require authorization
only for inpatient admission.

SECTION 7. TERM AND TERMINATION.
7.1 This Agreement shall have an initial term of three (3) years and shall
automatically renew for successive three-year terms.
7.2 Either party may terminate without cause upon ninety (90) days' written
notice. Plan's right to terminate is subject to Hospital's confirmation that
network adequacy can be maintained.
7.3 Hospital may terminate immediately for any payment dispute.

SECTION 8. GENERAL.
8.1 This Agreement shall be governed by the laws of the State of Texas.
8.2 No limitation of liability or indemnification provision is included.
```

-----