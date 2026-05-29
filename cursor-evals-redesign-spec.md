# Cursor Implementation Spec — Rebuild the Evals Section (Agent Builder Wizard)

> **Operating instructions for the implementing LLM (any model):**
> This is a build-ready spec, not a brief. It has two tiers, and you must treat them differently:
> 
> **TIER A — FROZEN CONTRACT (non-negotiable).** The eval *functionality* (domain model, evaluator types, granularities, every field and its MANDATORY/OPTIONAL status, metrics, run semantics, results) and the *UI/UX* (information architecture, component behaviors, mandatory/optional rendering, conditional-required logic, info/help, empty/loading/error states, accessibility) are FIXED. Implement them **exactly**. You may not drop, rename, merge, or reinterpret a Tier-A requirement. If a Tier-A requirement appears impossible given the codebase, **stop and report the conflict** — do not silently substitute.
> 
> **TIER B — DERIVE FROM THE CODEBASE (study, don’t guess).** Everything stack/repo-specific — exact file paths, the key/value wrapper’s real method names, the agent-invocation entrypoint, the async/background-task mechanism, the nav/tab registration point, auth/permission hooks, the HTTP client, test setup, design tokens/Tailwind config — must be **deduced by thoroughly studying the existing codebase** in P1 and recorded in writing before you build. The names and paths in this doc are defaults/placeholders: replace them with what the repo actually uses and state every substitution. Never invent an API that the repo doesn’t have; find the real one.
> 
> Rule of thumb: **what** the evals do and **how they look/behave** = Tier A, copy exactly. **Where** code lives and **which repo primitives** it calls = Tier B, derive and report. Work phase by phase (P0→P9); after each phase output files changed, commands run, and test results. No new libraries beyond the declared stack without flagging.

-----

## Known Stack (confirm by inspection; do not re-derive the language/framework)

**Frontend:** React 18 + TypeScript, Vite, React Router v6, Tailwind CSS (+ `tailwindcss-animate`), Zustand + React hooks, `lucide-react`, Vitest + Testing Library, pnpm.
**Backend:** Python 3.12, FastAPI (uvicorn), Pydantic v2, OpenAI + LangChain/LangGraph + httpx.
**Persistence:** DynamoDB (optional DAX) accessed **only** through the app’s existing **custom key/value persistence layer** — there is no SQLAlchemy, no ORM, no Alembic, and no relational joins. All reads/writes go through that wrapper.

-----

## P0.5 — MANDATORY codebase study (Tier B inputs — complete before any code)

Study the repo and produce a written **“Codebase Contract” report** that resolves every Tier-B unknown. The build may not begin until this is filled in with real values from the code (not guesses). At minimum, find and record:

|# |What to find                                                                                                                                                                                         |Why it’s needed                                                 |
|--|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------|
|1 |Custom key/value wrapper: module path + exact public methods/signatures (get/put/update/delete/query-by-prefix/batch/transaction), PK/SK key scheme, any GSIs, (de)serialization approach, DAX toggle|All eval persistence is written against this verbatim           |
|2 |Agent invocation entrypoint (the LangGraph/agent call) and exactly what it returns: final output, per-step trajectory, tool calls + args, token usage, latency, errors                               |The runner reuses this; it must not build a parallel one        |
|3 |Async / background-task mechanism used in the app (FastAPI BackgroundTasks? a queue? a worker?)                                                                                                      |Long eval runs use the existing mechanism                       |
|4 |Where Zustand stores live + the store pattern/conventions                                                                                                                                            |New `evalsStore` must match                                     |
|5 |The HTTP/API client pattern on the frontend + error handling convention                                                                                                                              |`evalsApi` must match                                           |
|6 |React Router v6 setup + the wizard’s nav/tab registration point                                                                                                                                      |New Evals section is wired in the same way                      |
|7 |Auth/permissions hooks on routes and endpoints                                                                                                                                                       |Eval routes/endpoints must respect them                         |
|8 |Tailwind config / design tokens / existing shared UI components (buttons, inputs, modals, drawers, tables)                                                                                           |UI/UX must reuse the design system, not reinvent it             |
|9 |Test setup for both Vitest and pytest (fixtures, mocks, how DynamoDB is faked)                                                                                                                       |New tests plug into existing harness                            |
|10|Existing eval inventory + all inbound references (see P1)                                                                                                                                            |Clean removal                                                   |
|11|How model spend/token cost is tracked or priced in the app (pricing table? usage from the provider response?)                                                                                        |Cost ceiling + run cost estimate need a real source, not a guess|
|12|Feature-flag mechanism (if any) and telemetry/logging/metrics utilities                                                                                                                              |Safe rollout + observability (P4.5)                             |
|13|Large-value strategy for the kv wrapper (how oversized payloads are stored vs DynamoDB’s item-size limit)                                                                                            |Persisting big inputs/outputs/traces safely                     |

For each row: give the real path/name, a one-line note, and — if not found — say so explicitly and propose the safest option rather than assuming. **Stop and report this report before P1 continues into code.**

-----

## Domain Model — TIER A, FROZEN (the single source of truth)

The mental model is the industry standard **Dataset → Run → Score**, with evaluation at three granularities (final-response, trajectory, single-step) and operating-envelope metrics tracked alongside quality.

Entities:

- **Dataset** has many **DatasetItem** (golden examples).
- **Evaluator** = one scoring unit (type + granularity + config).
- **EvalRun** = execute a set of Evaluators over a Dataset against one Agent version → produces **CaseResult** per (DatasetItem × trial) and **EvaluatorScore** per (CaseResult × Evaluator), plus run-level aggregates.

-----

## P0 — Branch & guardrails

1. Create branch `feat/evals-rebuild`.
1. Confirm app builds: `pnpm install && pnpm build` (frontend), backend boots via uvicorn.
1. Complete the **P0.5 Codebase Contract report** above. **Stop and report.**

-----

## P1 — Discovery & inventory (write no feature code)

The P0.5 Codebase Contract report already captured the wrapper, agent-invocation, async, store, client, nav, auth, design-system, and test-setup facts. P1 adds the **teardown inventory**:

1. Full inventory of the **existing** evals: every frontend file, route, component, Zustand slice, API call, backend route, Pydantic model, persisted key/entity, and test that belongs to the old evals section.
1. All inbound references to old evals (imports, nav entries, router `<Route>`s, feature flags, analytics).
1. Confirm (from P0.5) the exact nav/tab component where the new top-level section will register.
   **Stop and report.** Do not continue until the inventory is complete.

-----

## P2 — Remove old evals (clean teardown)

1. Delete every file/route/component/store/test/model from the P1 inventory.
1. Old eval data is worthless and probably absent. There are no relational tables to drop — instead, delete old eval **records/key-prefixes** from DynamoDB through the persistence layer (write a one-off idempotent cleanup script that lists and deletes the old eval key namespace via the wrapper’s query/delete methods). **Guard:** if the old eval key prefix returns a non-trivial number of items, or any item is referenced by a non-eval feature, pause and report instead of deleting. If the old code persisted nothing, note that and skip.
1. Remove dead imports, nav entries, routes, flags, analytics events.
1. Acceptance: `pnpm build`, `pnpm lint`, `pnpm typecheck` (or repo equivalents) pass; backend imports clean; no reference to deleted symbols remains (grep to confirm). Commit: `chore(evals): remove legacy evals section`.
   **Stop and report.**

-----

## P3 — Backend data model (Pydantic v2 + DynamoDB single-table via the key/value layer)

Create `backend/app/evals/` with: `schemas.py` (Pydantic v2 domain + request/response models), `repository.py` (all persistence, built on the custom key/value wrapper from P1 — no boto3/DynamoDB calls outside this file), `router.py`, `service.py`, `evaluators/` (runner logic), `__init__.py`. Adjust the package root to match the repo.

### 3.1 Enums (`schemas.py`)

> **TIER A — FROZEN.** Sections 3.1, 3.2, 3.4 (enums, every field + its MANDATORY/OPTIONAL status + conditional-required rules, and the metric catalog) are the eval functionality contract. Reproduce them exactly. Tier-B applies only to *where* this code lives and *how* it persists (3.3, via the wrapper from P0.5).

```python
from enum import Enum

class EvaluatorType(str, Enum):
    LLM_JUDGE = "llm_judge"
    CODE = "code"
    HUMAN = "human"
    PAIRWISE = "pairwise"          # optional 4th mode

class Granularity(str, Enum):
    FINAL_RESPONSE = "final_response"   # what failed
    TRAJECTORY = "trajectory"           # where it failed
    SINGLE_STEP = "single_step"         # why it failed

class ScoringScale(str, Enum):
    BINARY = "binary"               # pass/fail -> 1.0 / 0.0
    LIKERT_1_5 = "likert_1_5"       # 1..5, normalized to (n-1)/4 for aggregation
    NUMERIC_0_1 = "numeric_0_1"     # already normalized
    CATEGORICAL = "categorical"     # labeled buckets w/ explicit score map (e.g. correct/partial/incorrect)

class ScoreDirection(str, Enum):
    HIGHER_IS_BETTER = "higher_is_better"   # quality metrics
    LOWER_IS_BETTER = "lower_is_better"     # latency, cost, error_rate, redundant_step

class CaseOutcome(str, Enum):
    SCORED = "scored"               # produced a normalized score
    SKIPPED = "skipped"             # metric not applicable to this case/agent kind
    ABORTED = "aborted"             # agent invocation errored before producing output
    INDETERMINATE = "indeterminate" # judge could not decide / parse failure

class CodeAssertion(str, Enum):
    EXACT_MATCH = "exact_match"
    REGEX = "regex"
    CONTAINS = "contains"
    NOT_CONTAINS = "not_contains"
    JSON_SCHEMA_VALID = "json_schema_valid"
    NUMERIC_THRESHOLD = "numeric_threshold"
    TOOL_CALLED = "tool_called"
    TOOL_ARGS_MATCH = "tool_args_match"        # tool called WITH expected arguments
    TRAJECTORY_MATCH = "trajectory_match"      # see TrajectoryMatchMode
    NO_PII = "no_pii"                          # output contains no PII (safety)
    LATENCY_BUDGET_MS = "latency_budget_ms"
    COST_BUDGET_USD = "cost_budget_usd"
    STEP_BUDGET = "step_budget"                # <= N steps

class TrajectoryMatchMode(str, Enum):
    EXACT_ORDER = "exact_order"     # exact sequence + args
    IN_ORDER_SUBSEQUENCE = "in_order_subsequence"  # expected tools appear in order, extras allowed
    ANY_ORDER_SET = "any_order_set" # expected tool set present, order ignored
    # NOTE: agents often have multiple valid paths; prefer set/subsequence over exact unless order is semantically required.

class JudgeReferenceMode(str, Enum):
    REFERENCE_FREE = "reference_free"     # judge on rubric alone (no golden answer)
    REFERENCE_BASED = "reference_based"   # judge output against expected_output
    GROUNDING = "grounding"               # judge output against ACTUAL retrieved_context (RAG faithfulness)

class AgentKind(str, Enum):
    CONVERSATIONAL = "conversational"
    TOOL_CALLING = "tool_calling"
    RAG = "rag"

class RunStatus(str, Enum):
    QUEUED = "queued"; RUNNING = "running"; COMPLETED = "completed"
    FAILED = "failed"; CANCELLED = "cancelled"
    ABORTED_COST = "aborted_cost"   # hit cost_ceiling_usd
    # A RUNNING run whose heartbeat is stale (worker died) is reconciled to FAILED on next read/sweep.

class DatasetItemSource(str, Enum):
    MANUAL = "manual"; PRODUCTION = "production"; SYNTHETIC = "synthetic"; IMPORT = "import"
```

### 3.2 Pydantic schemas — fields MANDATORY/OPTIONAL exactly as marked

Use `Field(...)` for mandatory, `Field(default=...)` / `Optional[...]` for optional. The frontend mirrors these.

**DatasetCreate**

- `name: str` — **MANDATORY**
- `description: Optional[str]` — OPTIONAL
- `agent_kind: AgentKind` — **MANDATORY**

**DatasetItemCreate**

- `input: str | list[ChatMessage]` — **MANDATORY** (single query or multi-turn conversation)
- `expected_output: Optional[str]` — OPTIONAL (the expected *final* answer; required when an evaluator uses `REFERENCE_BASED` mode or a reference-based code assertion)
- `expected_trajectory: Optional[list[ExpectedToolCall]]` — OPTIONAL (required for trajectory-granularity evaluators)
- `context_documents: Optional[list[str]]` — OPTIONAL (RAG: the *golden* context, for context-recall metrics — distinct from what the agent actually retrieves at run time)
- `persona: Optional[str]` — OPTIONAL
- `difficulty: Optional[str]` — OPTIONAL
- `intent: Optional[str]` — OPTIONAL
- `tags: list[str] = []` — OPTIONAL
- `source: DatasetItemSource = MANUAL` — OPTIONAL
- `split: str = "test"` — OPTIONAL (e.g. `dev`/`test`/`regression`; lets users hold out a regression set)
- `is_adversarial: bool = False` — OPTIONAL (safety/red-team case)
- `dedup_hash` — server-computed from normalized `input`; reject or flag exact duplicates on add

`ExpectedToolCall = {tool_name: str (MANDATORY), arguments: Optional[dict]}`
`ChatMessage = {role: Literal["user","assistant","system","tool"], content: str}`

**EvaluatorCreate**

- `name: str` — **MANDATORY**
- `type: EvaluatorType` — **MANDATORY**
- `granularity: Granularity = FINAL_RESPONSE` — **MANDATORY** (defaulted)
- `metric_key: str` — **MANDATORY** (from the metric catalog in §3.4)
- `direction: ScoreDirection` — **MANDATORY** (defaulted from the metric catalog; e.g. latency → lower_is_better). Determines pass/fail comparison and regression sign.
- `weight: float = 1.0` — OPTIONAL (used only in the weighted overall; ignored for envelope metrics)
- `pass_threshold: Optional[float]` — OPTIONAL (on the **normalized 0–1 scale**; required if used as a CI gate). Pass = normalized_score ≥ threshold when higher_is_better, ≤ threshold when lower_is_better.
- `llm_judge: Optional[LLMJudgeConfig]` — required iff `type==LLM_JUDGE`
- `code: Optional[CodeConfig]` — required iff `type==CODE`
- `human: Optional[HumanConfig]` — required iff `type==HUMAN`

Enforce the conditional requirements with a Pydantic `model_validator(mode="after")` that raises if the matching config block is missing.

**LLMJudgeConfig**

- `judge_model: str = "gpt-4.1"` — OPTIONAL (defaulted; should differ from the agent’s own model where possible to reduce self-preference bias)
- `reference_mode: JudgeReferenceMode = REFERENCE_FREE` — **MANDATORY** (REFERENCE_BASED requires `expected_output`; GROUNDING judges against the **actual** `retrieved_context` captured at run time, not the golden context)
- `rubric: str` — **MANDATORY** (explicit, criterion-by-criterion; vague rubrics are the #1 cause of unreliable judges)
- `scale: ScoringScale = BINARY` — **MANDATORY**
- `category_score_map: Optional[dict[str,float]]` — required iff `scale==CATEGORICAL` (maps each label to a 0–1 score, e.g. `{"correct":1.0,"partial":0.5,"incorrect":0.0}`)
- `prompt_template: str` — **MANDATORY**; must contain `{input}` and `{output}`; `{reference}` required if REFERENCE_BASED, `{context}` required if GROUNDING. Validate placeholders against `reference_mode` on save.
- `require_reasoning_before_score: bool = True` — OPTIONAL (judge must emit rationale *before* the verdict — improves reliability; store the rationale)
- `few_shot_examples: Optional[list[dict]]` — OPTIONAL (labeled examples to anchor the judge)

> **Judge reliability is part of the contract:** judge output must be parsed into the declared scale with a strict parser; on parse failure the case outcome is `INDETERMINATE` (never silently 0). For PAIRWISE judging, run each comparison in both orders (A/B and B/A) and only count a win if consistent, to cancel position bias. Known biases (length, style, self-preference, position) must be noted in code comments and in the UI help.

**CodeConfig**

- `assertion: CodeAssertion` — **MANDATORY**
- `params: dict` — **MANDATORY** (shape depends on assertion; validate per-assertion: REGEX needs `pattern`; NUMERIC_THRESHOLD needs `op`+`value`; JSON_SCHEMA_VALID needs `schema`; TRAJECTORY_MATCH needs `mode: TrajectoryMatchMode`; TOOL_ARGS_MATCH needs `tool_name`+`expected_args` (+optional `match: exact|subset`); STEP_BUDGET needs `max_steps`; LATENCY/COST budgets need the numeric limit).
- Code assertions return a normalized 0–1 score (pass=1.0/fail=0.0 for boolean assertions; budgets return 1.0 within budget else 0.0, with the raw measured value also stored).

**HumanConfig**

- `rubric: str` — **MANDATORY** (same rubric structure as the judge it calibrates, so labels are comparable)
- `scale: ScoringScale = BINARY` — **MANDATORY** (must match the judge’s scale when used for calibration)
- `reviewer_emails: list[str] = []` — OPTIONAL
- `sampling_rate: float = 1.0` (0–1) — OPTIONAL (fraction of cases routed to humans)
- `reviewers_per_case: int = 1` — OPTIONAL (≥2 enables inter-rater agreement)
- `calibrates_evaluator_id: Optional[str]` — OPTIONAL (links this human eval to an LLM-judge; the run then reports judge-vs-human agreement, e.g. Cohen’s κ / % agreement)

**EvalRunCreate**

- `agent_id: str` — **MANDATORY**
- `agent_version: str` — **MANDATORY**
- `dataset_id: str` — **MANDATORY**
- `evaluator_ids: list[str]` — **MANDATORY**, min length 1
- `splits: Optional[list[str]]` — OPTIONAL (run only on selected dataset splits, e.g. just `regression`)
- `sample_size: Optional[int]` — OPTIONAL (run on a random N-item sample for a quick smoke check)
- `trials_per_case: int = 1` (≥1; recommend ≥3 for stochastic agents) — OPTIONAL
- `seed: Optional[int]` — OPTIONAL (for reproducibility where the agent/model honors it)
- `max_concurrency: int = 5` — OPTIONAL (parallelism for invocation + judging; respect provider rate limits)
- `idempotency_key: Optional[str]` — OPTIONAL (dedupe accidental double-submits; same key returns the existing run, never starts a second)
- `cost_ceiling_usd: Optional[float]` — OPTIONAL (hard stop: abort the run if cumulative model spend would exceed this; mark remaining cases `SKIPPED` with reason)
- `per_call_timeout_s: int = 60` — OPTIONAL (timeout for each agent invocation and each judge call)
- `max_retries: int = 2` — OPTIONAL (retries on transient/5xx/429 with exponential backoff + jitter; exhausted retries → `ABORTED`/`INDETERMINATE`, never a hang)
- `baseline_run_id: Optional[str]` — OPTIONAL
- `ci_gate: Optional[CIGateConfig]` — OPTIONAL
- `schedule: Optional[ScheduleConfig]` — OPTIONAL (one_off | on_deploy | recurring+cron)
- `track_envelope: EnvelopeToggles` — OPTIONAL, default `{latency:true, cost:true, errors:true, steps:false}`

Server captures `env_snapshot: dict` (agent + judge model config, seed, `dataset_version`, timestamps, eval-section version) automatically — read-only, so any run is reproducible/auditable. Each CaseResult stores the captured `retrieved_context` (when present) so grounding scores and the trace drawer can show what the agent actually saw.

### 3.3 Persistence design (DynamoDB single-table, via the key/value layer — no ORM)

All persistence lives in `repository.py` and is expressed **only** through the custom wrapper’s methods discovered in P1. Use a single-table, hierarchical-key design so related items are retrievable by partition + sort-prefix queries (no joins). If the wrapper imposes its own key convention, conform to it and map the scheme below onto it.

**Key scheme (adapt to the wrapper’s PK/SK naming):**

- Dataset:        `PK = DATASET#{dataset_id}`,  `SK = META`
- Dataset item:   `PK = DATASET#{dataset_id}`,  `SK = ITEM#{item_id}`  → all items fetched by partition query on the dataset PK.
- Evaluator:      `PK = EVALUATOR#{evaluator_id}`, `SK = META`
- Eval run:       `PK = RUN#{run_id}`,          `SK = META`
- Case result:    `PK = RUN#{run_id}`,          `SK = CASE#{item_id}#{trial}`
- Evaluator score:`PK = RUN#{run_id}`,          `SK = CASE#{item_id}#{trial}#SCORE#{evaluator_id}`  → a run’s full results fetched by partition query on the run PK, then grouped in memory.
- Listing collections (datasets/evaluators/runs): maintain index items, e.g. `PK = INDEX#DATASETS`, `SK = {created_at}#{dataset_id}`, written alongside the entity in the same logical operation, so list endpoints are a single prefix query (no table scan). Use the same pattern for `INDEX#EVALUATORS` and `INDEX#RUNS`. If the wrapper exposes GSIs, prefer a GSI over manual index items and say so.
- Human review queue: `PK = RUN#{run_id}`, `SK = REVIEW#{case_id}` for sampled cases needing human scores.

**Repository responsibilities:**

- One function per access pattern (e.g. `put_dataset`, `get_dataset`, `list_datasets`, `put_dataset_item`, `query_dataset_items`, `put_run`, `query_run_results`, `put_evaluator_score`, `list_review_queue`, etc.).
- (De)serialize Pydantic v2 models to/from the wrapper’s value format (`model_dump(mode="json")` / `model_validate`).
- Generate UUIDs for ids; stamp `created_at`/`updated_at`; honor soft-delete if the app already uses it (else hard delete via the wrapper).
- Writes that span multiple items (entity + its index item) should use the wrapper’s batch/transaction method if one exists; if not, write the entity first, then the index item, and document the non-atomicity.
- DAX: rely on the wrapper’s existing DAX toggle; do not add caching logic here.

**Dataset versioning:** keep an immutable `version: int` on the dataset META item, bumped on any item change, and snapshot the dataset’s item set into the run record at run time (`dataset_version` on the run) so results are reproducible. Surface version read-only in the UI. There is no migration tool — no Alembic step exists or is needed.

### 3.4 Metric catalog (`evaluators/metrics.py`)

Expose `GET /evals/metrics` returning grouped presets so the UI never shows a blank form:

- **final_response (outcome):** `task_success, correctness, relevance, answer_completeness, hallucination, tone`
- **rag / grounding:** `faithfulness` (output grounded in *retrieved* context), `answer_relevance`, `context_precision`, `context_recall` (retrieved vs golden `context_documents`)
- **safety:** `toxicity, pii_leakage, jailbreak_resistance, refusal_appropriateness` (pair with `is_adversarial` items)
- **trajectory:** `traj_match` (with `TrajectoryMatchMode`), `traj_precision, traj_recall, tool_selection_correct, parameter_accuracy, redundant_step`
- **single_step:** `step_reasoning_quality, step_tool_correct`
- **envelope:** `latency_ms, cost_usd, step_count, error_rate` (all lower_is_better)

Each catalog entry: `{key, label, granularity, default_evaluator_type, default_direction, default_reference_mode, applies_to_agent_kinds, requires_reference: bool, requires_trajectory: bool, requires_retrieved_context: bool, one_liner}`. The `requires_*` flags and `default_*` values drive the frontend conditional-required logic and sensible defaults. `applies_to_agent_kinds` lets the UI hide irrelevant metrics (e.g. trajectory metrics for a pure conversational agent) and marks them `SKIPPED` if applied anyway.

### 3.5 Normalization & aggregation contract (TIER A — must be exact)

Mixing scales is undefined unless normalized. Enforce:

1. **Every score is stored twice:** `raw_value` (e.g. Likert 4, latency 820ms, label “partial”) and `normalized_score` ∈ [0,1]. Normalization: BINARY→{0,1}; LIKERT_1_5→(n−1)/4; NUMERIC_0_1→as-is; CATEGORICAL→`category_score_map`; budgets/envelope→1.0 if within budget else 0.0 (raw value always retained).
1. **Outcome gating:** only cases with `outcome==SCORED` enter quality aggregates. `SKIPPED`/`ABORTED`/`INDETERMINATE` are counted and reported separately (e.g. “12% indeterminate”) and **never** silently treated as 0 — that distinction is the difference between “agent is wrong” and “we couldn’t measure.”
1. **Per-evaluator aggregate:** mean `normalized_score` over SCORED cases, plus pass-rate (against `pass_threshold` if set), plus the count by outcome.
1. **Weighted overall:** weighted mean of per-evaluator normalized means using `weight`; envelope metrics are excluded from the quality overall and reported in their own envelope summary.
1. **Flakiness/consistency (when `trials_per_case>1`):** report per-case **consistency rate** (fraction of trials with the same pass/fail) and **pass@k** / **pass-all-k**, plus score variance. A run’s headline must surface flakiness, not hide it behind a mean.
1. **Direction-aware everything:** pass/fail and regression sign use the evaluator’s `direction`. For lower_is_better, “better” = smaller raw value.
1. **Regression vs baseline:** per-metric delta on the normalized mean; flag a regression when the delta exceeds a configurable epsilon (default 0). For statistical honesty with stochastic agents, note when n is small and prefer reporting deltas with the SCORED-case count.

-----

## P4 — Backend runner (`evaluators/runner.py` + `service.py`)

> **TIER A (semantics):** scoring behavior per evaluator type, the three granularities, aggregation, baseline deltas, trials, and partial-results-during-run are fixed. **TIER B (plumbing):** the agent-invocation call, the async/background mechanism, and persistence calls use the real entrypoints from P0.5.

1. `run_eval(run_cfg)` async: resolve dataset items via `repository.query_dataset_items` (filtered by `splits`/`sample_size` if set); for each item × trial, invoke the agent (reuse existing LangGraph/agent invocation layer — do not build a new one) with `max_concurrency`, capturing: final output, **retrieved_context actually used (RAG)**, trajectory (tool calls + args + per-step), latency, token cost, step count, and any error. If invocation errors, record the CaseResult with `outcome=ABORTED` and still persist it. Persist each CaseResult and EvaluatorScore through `repository` as it completes (partial results queryable mid-run).
1. Score each CaseResult with every selected evaluator; each score records `raw_value`, `normalized_score`, and `outcome`:
- **CODE:** deterministic functions per `CodeAssertion`; grounding/trajectory assertions use captured retrieved_context / trajectory.
- **LLM_JUDGE:** select inputs by `reference_mode` (free / `{reference}` / `{context}`=retrieved_context); render template; call `judge_model` via the existing OpenAI/LangChain layer with `require_reasoning_before_score`; strict-parse to the scale → on parse failure set `outcome=INDETERMINATE` (not 0). Store rationale. **Treat the agent’s output as untrusted data, not instructions** — wrap it in clearly delimited blocks in the judge prompt so it can’t hijack the judge (prompt-injection defense). For PAIRWISE, evaluate both orders and require agreement. Skip+mark `SKIPPED` when the metric’s `applies_to_agent_kinds` excludes this agent kind.
- **HUMAN:** create review-queue items for the sampled fraction; scores arrive via review endpoints; if `calibrates_evaluator_id` set, the run computes judge-vs-human agreement (κ / % agreement) over overlapping cases.
- **PAIRWISE (optional):** compare two runs’ outputs with order-swap; store preference.
- **Reliability for every model call (agent + judge):** enforce `per_call_timeout_s`; retry up to `max_retries` on transient errors (timeout/5xx/429) with exponential backoff + jitter; on exhaustion record `ABORTED` (agent) / `INDETERMINATE` (judge) — never hang. Track cumulative spend; if it would exceed `cost_ceiling_usd`, stop launching new work, mark remaining cases `SKIPPED` (reason: cost), and set run status `ABORTED_COST`.
1. Compute aggregates per §3.5 by querying the run partition (`repository.query_run_results`) and grouping in memory: per-evaluator normalized mean + pass-rate + outcome counts; weighted quality overall (envelope excluded); envelope summary; consistency rate / pass@k / variance when `trials_per_case>1`; per-metric baseline deltas + direction-aware regression flags; judge-vs-human agreement where applicable. Persist the aggregate onto the run META item.
1. **Run lifecycle & crash safety:** on create, dedupe via `idempotency_key`; set `QUEUED`→`RUNNING`. Persist `status`, a monotonically updated `heartbeat_at`, and progress (cases done / total) on the run META item. Run async via the existing background mechanism (P0.5). Cancel stops launching new invocations and keeps partial results. A run stuck in `RUNNING` with a stale `heartbeat_at` (worker died) is reconciled to `FAILED` by a lightweight sweep/on-read check, so no run is orphaned forever. Endpoint polls status + partial results.

### API (`router.py`, prefix `/api/evals`)

```
POST   /datasets                 GET /datasets            GET /datasets/{id}
PUT    /datasets/{id}            DELETE /datasets/{id}
POST   /datasets/{id}/items      PUT /items/{id}          DELETE /items/{id}
POST   /datasets/{id}/import     (CSV/JSON; OPTIONAL)
POST   /datasets/{id}/from-traces (production traces; OPTIONAL)
POST   /evaluators               GET /evaluators          GET/PUT/DELETE /evaluators/{id}
GET    /metrics
POST   /runs                     GET /runs                GET /runs/{id}
POST   /runs/estimate            (returns estimated cost + call count for an EvalRunCreate)
GET    /runs/{id}/results        POST /runs/{id}/cancel
GET    /runs/{id}/compare?baseline={id}
GET    /runs/{id}/review-queue   POST /review/{case_id}   (human scores)
```

- All list/results endpoints are **paginated** (cursor/limit) — never return an unbounded collection; `GET /runs/{id}/results` pages over case results.
- `POST /runs` accepts `idempotency_key` and is safe to retry.
- Every endpoint enforces auth + **resource ownership/tenant scoping** (see P4.5); a caller can only read/run/mutate datasets, evaluators, and runs they own or are shared with.
- Standard error envelope: validation → 422 with field-level messages; not-found → 404; forbidden → 403; conflicts (e.g. idempotency) → 409. Pydantic request/response models for all. Backend tests in §P9.

-----

## P4.5 — Production hardening (TIER A — required for “done”)

This feature touches user data, external paid models, and multi-tenant access; the following are not optional.

1. **AuthZ & tenancy:** reuse the app’s existing auth (from P0.5). Persist an owner/tenant id on every Dataset, Evaluator, and Run; scope all reads/queries by it; deny cross-tenant access. Add tests that prove isolation.
1. **Secrets:** judge-model and provider keys come from the app’s existing secret management/config — never hardcoded, never persisted into `env_snapshot` or returned by the API.
1. **PII & trace privacy:** `from-traces` import must run optional PII redaction and record data provenance; document a retention expectation for imported traces; never echo secrets/PII into judge prompts or logs beyond what the case requires.
1. **Limits & validation:** cap dataset item count per request, max input/output size persisted (respect the DynamoDB item-size limit — store oversized payloads via the wrapper’s large-value strategy or truncate-with-flag), max evaluators per run, max `trials_per_case`, max `sample_size`. Reject over-limit with 422.
1. **Cost & rate safety:** `cost_ceiling_usd` enforced (above); `max_concurrency` respects provider rate limits; surface estimated run cost in the UI before launch.
1. **Observability:** structured logs for run start/finish/cancel/abort with run id, counts, total cost, and duration; emit metrics (runs started/failed, mean latency, spend) via the app’s existing telemetry; the run META item is an audit record (who ran what, against which agent/dataset version, when).
1. **Idempotency & crash recovery:** as specified in P4 (idempotency_key, heartbeat, stale-run reconciliation).
1. **Safe rollout:** ship the new section behind the app’s existing feature-flag mechanism if one exists; the teardown (P2) and the new section land in separate commits so either can be reverted independently.
1. **Graceful degradation:** if the agent-invocation layer or judge provider is down, runs fail cleanly with a clear status and message; the UI never spins forever.

-----

## P5 — Frontend structure

> **TIER A — FROZEN (UI/UX).** P5–P7 define the information architecture, component behaviors, mandatory/optional rendering, conditional-required logic, help system, and empty/loading/error/accessibility states. Implement all of it exactly. **TIER B:** the concrete file paths below, the Zustand store wiring, the HTTP client, the nav registration point, and which shared UI primitives (button/input/modal/drawer/table) you compose from — all come from the P0.5 study; reuse the repo’s design system and conventions rather than inventing components, and state substitutions.

Create `src/features/evals/`:

```
evals/
  routes/            EvalsLayout.tsx, DatasetsPage.tsx, DatasetDetailPage.tsx,
                     EvaluatorsPage.tsx, RunsPage.tsx, RunDetailPage.tsx, SettingsPage.tsx
  components/        DatasetForm.tsx, DatasetItemForm.tsx, EvaluatorForm.tsx,
                     EvaluatorTypePicker.tsx, MetricPicker.tsx, RunForm.tsx,
                     ResultsTable.tsx, RunComparison.tsx, CaseTraceDrawer.tsx,
                     ReviewQueue.tsx, FieldLabel.tsx, InfoButton.tsx, HelpDrawer.tsx,
                     EmptyState.tsx
  store/             evalsStore.ts      (Zustand)
  api/               evalsApi.ts        (typed client, mirrors §P4)
  types/             evals.types.ts     (TS mirror of §3.1–3.2 enums/schemas)
  help/              helpContent.ts     (plain-language copy, §P7)
  __tests__/
```

Register routes under React Router v6 nested in `EvalsLayout` (tabs: Datasets, Evaluators, Runs, Results, Settings). Add one nav entry to the wizard’s existing nav.

### Zustand store shape (`evalsStore.ts`)

```ts
interface EvalsState {
  datasets: Dataset[]; evaluators: Evaluator[]; runs: EvalRun[];
  metrics: MetricCatalogEntry[];
  selectedRun?: RunDetail;
  loading: { datasets: boolean; evaluators: boolean; runs: boolean; run: boolean };
  error?: string;
  // actions
  fetchDatasets(cursor?: string): Promise<void>;
  fetchEvaluators(): Promise<void>;
  fetchMetrics(): Promise<void>;
  fetchRuns(cursor?: string): Promise<void>;
  fetchRun(id: string): Promise<void>;          // includes paginated results
  pollRun(id: string): Promise<void>;            // poll status+progress while RUNNING, then stop
  estimateRunCost(r: EvalRunCreate): Promise<{ estimatedUsd: number; calls: number }>;
  createDataset(d: DatasetCreate): Promise<Dataset>;
  createEvaluator(e: EvaluatorCreate): Promise<Evaluator>;
  startRun(r: EvalRunCreate): Promise<EvalRun>;  // pass idempotency_key
  cancelRun(id: string): Promise<void>;
}
```

All async actions set `loading`/`error` and use `evalsApi`. No business logic in components.

### Mandatory/optional in the UI

- `FieldLabel.tsx` renders `*` (red) for mandatory and a muted “(optional)” suffix for optional. Every form field uses it.
- Conditional-required logic (driven by metric catalog `requires_reference` / `requires_trajectory` / `requires_retrieved_context` and the judge’s `reference_mode`): REFERENCE_BASED makes “Expected output” mandatory live; trajectory granularity makes “Expected trajectory” mandatory; GROUNDING mode requires the agent to be a RAG kind that emits retrieved context (warn if not). CATEGORICAL scale requires a category→score map. Show inline validation, block submit, never silently fail.

-----

## P6 — Component behavior specifics

- **EvaluatorTypePicker:** cards for LLM-judge / Code / Human / Pairwise, each with a one-line “use when…”. Selecting one reveals only that type’s config block.
- **MetricPicker:** grouped by granularity using `/metrics`; preselect a sensible default; show the 3–5-metric mix recommendation as a hint.
- **RunForm:** agent+version (mandatory), dataset (mandatory), evaluators multi-select (≥1), optional splits/sample-size, trials (default 1, hint “≥3 reduces flakiness”), optional baseline, optional CI gate, optional schedule, envelope toggles (latency/cost/errors on by default), advanced (collapsed): concurrency, timeout, retries, cost ceiling. **Show an estimated cost + call count before the user launches** (via `estimateRunCost`), and show a live progress bar + Cancel while running.
- **ResultsTable:** per-case rows, per-evaluator columns showing normalized score + raw value, pass/fail chips, **distinct visual states for SKIPPED / ABORTED / INDETERMINATE** (never shown as 0); filter by tag/persona/difficulty/split/pass-fail/outcome; when trials>1 show per-case consistency. Row → `CaseTraceDrawer`.
- **CaseTraceDrawer:** full trace — input, **retrieved context the agent actually used (RAG)**, each step + tool call + args, final output, each evaluator’s raw+normalized score, outcome, and LLM-judge rationale. Organize so final-response, trajectory, and single-step views are distinct (what / where / why). For multi-trial cases, show each trial.
- **RunComparison:** side-by-side vs baseline, direction-aware per-metric deltas, regressions highlighted; show outcome-mix and consistency deltas, not just means.
- **ReviewQueue:** sampled human-review cases with rubric + scoring controls → `POST /review/{case_id}`.
- All list pages use `EmptyState` (explains the concept + “create your first…” CTA). All async UI has loading/error/success; running runs show progress + Cancel.

-----

## P7 — Info button & help (MANDATORY)

- `InfoButton.tsx` (lucide `Info` icon) visible in `EvalsLayout` header → opens `HelpDrawer.tsx`.
- `helpContent.ts` plain-language copy covering: the 5-step loop (build dataset → pick evaluators → run → read results → fix & re-run); each evaluator type with a “use this when…” example; mandatory vs optional fields and why; a “first eval in 2 minutes” quickstart.
- Inline `?` tooltips on complex fields (rubric, granularity, trajectory, trials, CI gate, threshold). Help text must avoid jargon or define it on first use.

-----

## P8 — Multi-perspective review (MANDATORY gate before “done”)

Produce `EVALS_REVIEW.md` with a section per viewpoint listing issues found and how each was resolved (or logged as a tracked follow-up):

1. **Staff Software Architect** — single-table key design & access patterns, hot-partition risk, query cost vs scans, eventual-consistency/non-atomic multi-item writes, run-storage scalability, API design, extensibility for new evaluator types, large-dataset performance.
1. **UI/UX Expert** — clarity of Dataset→Run→Score flow, cognitive load, progressive disclosure, consistency, error prevention.
1. **Naive End User** — can a first-timer create & run one eval in minutes using only the Info button + inline help? Where would jargon trip them?
1. **Power User / ML Engineer** — trajectory metrics, multi-trial, CI gating, baselines, trace drill-down all reachable; parity with Braintrust/Langfuse/LangSmith.
1. **Accessibility & QA** — keyboard nav, ARIA, contrast, empty/error/validation states.
1. **Security/Privacy** — production-trace handling, PII in datasets, judge-model data exposure, permissions.
   Resolve blocking issues before marking complete.

-----

## P9 — Tests & delivery (acceptance criteria)

**Backend (pytest):** schema validation incl. conditional-required rules; score normalization for every scale + direction; outcome handling (SKIPPED/ABORTED/INDETERMINATE never counted as 0); each `CodeAssertion` incl. trajectory match modes; LLM-judge parsing per scale + parse-failure→INDETERMINATE (judge call mocked); prompt-injection isolation (malicious agent output cannot change the judge verdict in a test fixture); retry/timeout/cost-ceiling behavior (mocked transient failures, forced ceiling); idempotency_key dedupe; stale-run reconciliation; runner aggregation + consistency/pass@k + baseline deltas; **tenant isolation (caller A cannot read/run/mutate B’s resources)**; pagination; every endpoint happy + error path (422/403/404/409); **repository tests against a mocked/local DynamoDB (moto or a fake wrapper) covering each access pattern — put/get/list/query-by-prefix and the entity+index write path, including the non-atomic-write reconciliation.**
**Frontend (Vitest + Testing Library):** each form’s mandatory/optional + conditional-required behavior (incl. reference_mode/grounding/categorical rules); store actions (api mocked); ResultsTable filtering + distinct rendering of SKIPPED/ABORTED/INDETERMINATE; CaseTraceDrawer renders trace incl. retrieved context + per-trial; RunComparison deltas; HelpDrawer opens with required content; empty/loading/error states; estimated-cost shown before run launch.
**Gates:** `pnpm build`, `pnpm lint`, `pnpm typecheck`, `pnpm test` all green; backend tests green; no dead references to old evals; nav shows new Evals; a manual smoke run (create dataset → 1 item → 1 code evaluator + 1 LLM-judge → run trials=1 → results render → drill into trace) succeeds.
**Deliver:** PR description summarizing removal, new architecture, data model, the three evaluator types, and the three granularity levels.

-----

## P10 — Production-readiness gate (must ALL be true before claiming done)

Do not report the work complete until every item below is satisfied and demonstrated; if any cannot be met, stop and report it rather than declaring success.

- [ ] The core loop works end to end on a real agent of each kind (conversational, tool-calling, RAG): build dataset → pick evaluators (code + judge + a human-sampled one) → run → read results → drill into a trace → re-run and see a baseline comparison.
- [ ] All three evaluator types and all three granularities produce correctly normalized, direction-aware, outcome-tagged scores.
- [ ] No run can hang, orphan, or exceed its cost ceiling; timeouts, retries, cancel, and crash-recovery all verified.
- [ ] Tenant isolation verified; secrets never leak to API/logs/snapshots; trace import redaction in place.
- [ ] List/results endpoints paginate; oversized payloads handled within DynamoDB limits.
- [ ] Observability (structured logs + metrics + audit record) emits for run start/finish/cancel/abort.
- [ ] Info button + inline help let a first-timer run their first eval unaided (verified in the P8 naive-user review).
- [ ] All P9 gates green; the six-viewpoint `EVALS_REVIEW.md` has no unresolved blocking issues.
- [ ] New section is behind a feature flag (if the app has one) and teardown/new-section are separately revertible.

-----

## Global rules

- **Tier A (eval functionality + UI/UX) is frozen** — implement exactly; never drop, rename, merge, or reinterpret. If it seems impossible in this codebase, stop and report the conflict instead of substituting.
- **Tier B (paths, repo primitives, plumbing) is derived** — from the P0.5 Codebase Contract report; use the repo’s real APIs/conventions and state every substitution. Deviating on Tier B to match the repo is expected; deviating on Tier A is not allowed.
- Never invent a persistence, agent-invocation, async, auth, or UI primitive the repo doesn’t have — find and reuse the real one.
- Mandatory/optional labels from §3.2 must be visible in the UI verbatim in intent (`*` for required, “(optional)” for optional).
- No new dependencies beyond the declared stack without flagging and justifying.
- Stop and report after every phase (P0.5, P0, P1, P2, P3, P4, P4.5, P5–P10).