# output-governor design

Author: Joseph Farzinzad

## 1. Goal

Control LLM output cost without turning complete answers into incomplete ones.

The governor sits around an LLM call. It classifies the task, selects an answer contract, assigns an output budget, translates that budget into provider controls, observes the response, stops supported streams only at a safe point, validates the result, and writes a ledger event.

## 2. Core truth

There are three different kinds of output reduction:

- **Generation control** prevents tokens from being generated. It can reduce the current provider bill.
- **Post-processing** removes text after generation. It can reduce downstream context and storage, but cannot refund the current call.
- **Shadow analysis** estimates what could have been saved. It proves nothing about realized savings until enforced.

The ledger must never mix them.

## 3. Non-goals

The MVP will not:

- Promise a universal output/input price ratio
- Cut arbitrary streams at a raw token count
- Use an LLM judge on every response
- Claim savings without a named baseline
- Replace provider gateways, caches, or observability systems
- Rewrite exact code, legal text, citations, or structured data to make it shorter
- Control hidden reasoning a provider does not expose or support

## 4. Architecture

```text
Request
  -> task classifier
  -> answer-contract resolver
  -> budget allocator
  -> provider capability negotiation
  -> request adapter
  -> provider stream
  -> normalized stream events
  -> repetition and filler detector
  -> structural parser
  -> completeness tracker
  -> safe-stop decision
  -> final validator
  -> bounded repair or fail open
  -> application response
  -> savings ledger and optional raw store
```

The core uses interfaces so it can run as a library, proxy, framework callback, or gateway plugin.

## 5. Data contracts

```python
GenerationRequest(
    task_id: str,
    call_id: str,
    provider: str,
    model: str,
    messages: list[Message],
    tools: list[ToolSchema],
    requested_contract: str | None,
    requested_budget: OutputBudget | None,
    metadata: dict[str, Any],
)
```

```python
OutputBudget(
    target_tokens: int,
    hard_tokens: int,
    repair_reserve_tokens: int,
    dollar_limit: Decimal | None,
    reasoning_limit: int | None,
)
```

```python
AnswerContract(
    name: str,
    version: str,
    output_format: str,
    required_fields: list[Requirement],
    optional_fields: list[Requirement],
    structure_rules: StructureRules,
    style_rules: StyleRules,
    default_budget: OutputBudget,
    risk: RiskLevel,
    validator_names: list[str],
    incomplete_action: str,
)
```

```python
GovernedResponse(
    content: Any,
    usage: NormalizedUsage,
    contract: ContractResult,
    stop: StopDecision,
    savings: SavingsResult,
    warnings: list[str],
    raw_response_ref: str | None,
)
```

## 6. Task classifier

Classification priority:

1. Explicit contract from the application
2. Tool or route configuration
3. Structured response schema
4. Deterministic request-shape rules
5. Optional learned classifier later
6. Conservative generic fallback

MVP classes:

- classification
- extraction
- direct_answer
- summary
- code
- plan
- tool_decision
- long_form
- exact_text

The classifier returns class, confidence, reasons, and risk signals. Low confidence means a larger budget and no semantic early stop.

## 7. Answer contracts

A contract defines the smallest usable answer, not merely a style preference.

### Required elements

- Format
- Required fields or sections
- Target and hard budgets
- Structural rules
- Semantic completion rules
- Risk level
- Repair policy
- Stop policy

### Example

```yaml
name: deployment_plan
version: 1
format: markdown
required:
  - prerequisites
  - ordered_steps
  - rollback
  - verification
budget:
  target_tokens: 450
  hard_tokens: 700
  repair_reserve_tokens: 120
structure:
  max_sections: 6
  max_bullets_per_section: 8
style:
  no_preamble: true
  no_repeated_summary: true
risk: high
stop:
  semantic_early_stop: false
on_incomplete: fail_open
```

Contracts are versioned. Every ledger event stores the exact version.

## 8. Budget allocator

Inputs:

- Contract defaults
- Application hard limit
- Provider model limits
- Current task spend
- Dollar limit
- Reasoning settings
- Repair reserve
- Risk

Rules:

1. A hard provider limit cannot exceed the model's supported maximum.
2. A dollar limit is translated with the current configured price.
3. Repair reserve is protected from the first attempt.
4. High-risk contracts use conservative caps.
5. Missing pricing disables dollar enforcement, not token enforcement.
6. Hidden reasoning is tracked separately when usage exposes it.

```text
first_attempt_limit = hard_tokens - repair_reserve_tokens
```

If that value is below the contract minimum, fail before calling the provider.

## 9. Provider capability negotiation

Each adapter declares capabilities:

```python
ProviderCapabilities(
    max_output_tokens: bool,
    native_verbosity: bool,
    reasoning_budget: bool,
    structured_output: bool,
    stop_sequences: bool,
    stream_cancel: bool,
    exact_usage: bool,
    separate_reasoning_usage: bool,
)
```

Negotiation order:

1. Structured schema
2. Provider maximum-output field
3. Native verbosity or reasoning control
4. Safe stop sequences
5. Prompt-side contract instruction
6. Stream observer and cancellation

The adapter records requested, accepted, unavailable, ignored, and inferred controls. Tests must use provider fixtures rather than assumptions.

Initial adapters:

- OpenAI Responses and Chat Completions shapes
- Anthropic Messages shape
- Gemini generate-content shape
- Generic OpenAI-compatible shape

## 10. Streaming event model

All provider streams normalize to:

```python
StreamEvent(
    kind: str,
    text_delta: str | None,
    structured_delta: Any | None,
    usage_delta: NormalizedUsage | None,
    finish_reason: str | None,
    provider_event: str,
    timestamp: datetime,
)
```

The normalizer preserves provider events for audit but exposes one control interface.

## 11. Structural tracker

The structural tracker incrementally checks:

- JSON braces, arrays, strings, and escapes
- XML tags
- Markdown code fences
- Markdown lists and headings
- Parentheses and brackets for code
- Function and class boundaries where parsers support them
- Citation markers
- Tool-call schemas

A safe stop requires a valid boundary for the contract format.

## 12. Completeness tracker

Deterministic validators first:

- Required JSON keys and types
- Allowed classification labels
- Required headings
- Requested item count
- Code entry point, closed blocks, and syntax parse
- Tool name and required arguments
- Explicit user questions answered
- Required identifiers and citations retained

Semantic completion uses a rule score in the MVP. An optional local or model evaluator comes later.

```python
CompletenessResult(
    score: float,
    required_passed: int,
    required_total: int,
    missing: list[str],
    confidence: float,
)
```

A high score without high confidence does not authorize early stop.

## 13. Filler and repetition detector

Deterministic signals:

- Repeated sentence or paragraph
- Restatement of the question
- Duplicate conclusion
- Empty transition phrases
- Repeated disclaimer not required by policy
- Excess headings for a short contract
- Long preamble before the first required field
- Repeated code or JSON block

The detector is evidence for stopping or post-processing. It never deletes facts on its own.

Project policy can protect required legal, safety, medical, or financial language.

## 14. Safe-stop controller

State:

```python
StopState(
    generated_tokens: int,
    structural_valid: bool,
    structural_boundary: bool,
    completeness: CompletenessResult,
    filler_score: float,
    repetition_score: float,
    contract_risk: RiskLevel,
    provider_cancel_supported: bool,
)
```

Early stop requires all configured gates:

- Minimum useful length reached
- Structural boundary is valid
- Required contract checks pass
- Completeness confidence meets threshold
- No protected section is open
- Provider supports cancellation with known behavior
- Contract risk allows semantic stop

Hard limits remain the last boundary. The controller never claims semantic completeness from token count alone.

## 15. Final validation and repair

After normal completion or cancellation:

1. Parse structure.
2. Run contract validators.
3. Check exact-value invariants.
4. Confirm stop and usage data.
5. Return if valid.
6. If invalid and policy allows, use one bounded repair pass from the reserve.
7. If repair fails, return the original response with warnings or block only in strict mode.

Repair instructions include missing requirements and the remaining budget. They do not ask for a full rewrite when a small patch works.

## 16. Modes

### Observe

Record actual output, contract fit, and usage. Do not intervene.

### Shadow

Calculate controls and potential stop points, but send the original request and return the original response. Report potential savings only.

### Enforce

Apply provider controls and safe stopping. Report realized savings only against a named baseline.

### Strict

Block outputs that fail contract validation or ledger recording. Intended for typed automation, not casual chat.

## 17. Savings ledger

```python
LedgerEvent(
    timestamp: datetime,
    task_id: str,
    call_id: str,
    provider: str,
    model: str,
    contract_name: str,
    contract_version: str,
    mode: str,
    controls_requested: dict,
    controls_applied: dict,
    baseline_type: str | None,
    baseline_tokens: int | None,
    actual_visible_tokens: int | None,
    actual_reasoning_tokens: int | None,
    realized_tokens_avoided: int | None,
    downstream_tokens_removed: int,
    potential_tokens_avoidable: int | None,
    input_price_per_million: Decimal | None,
    output_price_per_million: Decimal | None,
    estimated_realized_savings: Decimal | None,
    completeness_score: float | None,
    completeness_confidence: float | None,
    structure_valid: bool,
    repair_calls: int,
    finish_reason: str | None,
    stop_reason: str,
    latency_ms: float,
    raw_response_ref: str | None,
    warnings: list[str],
)
```

Baseline types:

- randomized_control
- paired_replay
- historical_contract_median
- shadow_observation
- none

Only randomized control and carefully matched replay are strong evidence. Historical and shadow baselines are estimates.

## 18. Cost accounting

```text
actual_cost = reported_output_tokens * configured_output_price
estimated_realized_savings = realized_tokens_avoided * configured_output_price
```

Do not subtract post-processed tokens from the current-call cost.

If a provider reports the actual charge, store it separately and prefer it for spend reporting. Price-based calculations stay labeled estimates.

## 19. Composition with token-governor

Shared envelope:

```python
TaskEconomics(
    task_id: str,
    total_dollar_budget: Decimal | None,
    total_token_budget: int | None,
    input_tokens_saved: int,
    output_tokens_avoided: int,
    actual_cost: Decimal | None,
)
```

Order:

```text
application
  -> token-governor for tool and context input
  -> output-governor preflight
  -> model
  -> output-governor stream and validation
  -> shared ledger
```

The packages can run independently. Neither imports private internals from the other. A small shared protocol package may come later.

## 20. Python package layout

```text
src/output_governor/
  __init__.py
  governor.py
  config.py
  models.py
  classify.py
  contracts.py
  budget.py
  pricing.py
  capabilities.py
  events.py
  structure.py
  completeness.py
  repetition.py
  stop.py
  repair.py
  cli.py
  providers/
    base.py
    openai.py
    anthropic.py
    gemini.py
    openai_compatible.py
  validators/
    base.py
    classification.py
    extraction.py
    direct_answer.py
    summary.py
    code.py
    plan.py
    tool_call.py
  ledger/
    base.py
    jsonl.py
    sqlite.py
    reports.py
  raw_store/
    base.py
    filesystem.py
  proxy/
    app.py
    auth.py
  integrations/
    simple_loop.py
    token_governor.py
    hermes.py
```

## 21. Configuration

```yaml
mode: shadow
pricing:
  source: static
ledger:
  adapter: sqlite
  path: .output-governor/ledger.db
raw_store:
  enabled: false
contracts:
  default: direct_answer
budget:
  task_output_tokens: 5000
  repair_reserve_percent: 15
stop:
  enabled: true
  minimum_completeness: 1.0
  minimum_confidence: 0.95
providers:
  openai:
    enabled: true
  anthropic:
    enabled: true
  gemini:
    enabled: true
fallback: fail_open
```

Tool output and model output can never change configuration.

## 22. Evaluation

Fixture categories:

- One-label classification
- Typed extraction
- Direct factual answer
- Summary with required topics
- Python and TypeScript code
- Deployment plan
- Tool call
- Long-form document
- JSON streaming
- Markdown with code fences
- Repetitive and padded responses
- Answers with important late details

Metrics:

- Actual output tokens
- Realized, downstream, and potential savings
- Contract pass rate
- Exact-field retention
- Syntax and schema validity
- Task success
- Repair rate and cost
- Incorrect early-stop rate
- Latency and time to first token

Release gates:

- Zero invalid JSON or tool-call stops in the fixture suite
- Zero changed exact values in protected fixtures
- Incorrect early-stop rate below the configured threshold
- Ledger totals reproduce from raw events
- Enforced tasks do not lose success against control beyond the stated tolerance

## 23. Competitive design response

The reviewed tools establish a high baseline:

- LiteLLM already tracks cost, routes models, enforces spend budgets, and supports output-token throughput limits.
- Helicone already acts as a gateway and observes model calls.
- GPTCache already avoids repeated calls.
- PromptLayer and LangSmith already provide strong traces and evaluation workflows.
- Martian already focuses on cost-aware model routing.
- TokenCut and Edgee already discuss output compression.
- Budget Guidance already studies length-conditioned reasoning.

Therefore output-governor should not compete as another dashboard, cache, router, or text shortener. Its center is the answer contract plus safe generation control and honest savings attribution.

## 24. MVP cut

Build now:

- Core types and YAML configuration
- Contract registry
- Deterministic classifier
- Budget and static price registry
- OpenAI, Anthropic, Gemini, and generic adapters
- Stream event normalizer
- Structural tracker
- Deterministic validators
- Safe-stop controller
- One repair pass
- Observe, shadow, and enforce modes
- SQLite and JSONL ledger
- OpenAI-compatible proxy
- Simple agent-loop example
- Replay benchmark and CI
- Token-governor composition example

Build later:

- Learned task classifier
- LLM or local semantic judge
- Production gateway plugins
- Hosted dashboard
- Team policy service
- Automatic price downloads
- Full Hermes integration
- Multi-turn budget optimizer

## 25. Security

- Treat model output as untrusted data.
- Do not execute generated code during validation outside a sandbox.
- Protect provider keys in environment or secret storage.
- Authenticate any non-local proxy.
- Keep full prompts and responses out of normal metrics.
- Make raw storage optional with retention controls.
- Guard against model output attempting to alter budgets or contracts.

## 26. Release definition

Version 0.1.0 is ready when:

- Three provider adapters pass recorded fixtures.
- Library and proxy examples work.
- All contract families have pass, fail, and over-short fixtures.
- Stream stopping preserves structure.
- Repair cannot exceed its reserve.
- Shadow mode changes no returned output.
- The ledger separates all three savings categories.
- Benchmarks include quality and cost.
- README claims match measured results.

## Author

Joseph Farzinzad
