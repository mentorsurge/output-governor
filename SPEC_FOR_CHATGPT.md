# Build specification for ChatGPT

Author: Joseph Farzinzad

> ChatGPT: build this repository. Do not rewrite this specification, return another plan, or substitute a different architecture. Create the files, implement the package, run the tests, fix failures, and leave a working repository. Follow `DESIGN.md` as the source of truth. Use complete files with no placeholders. Do not publish performance or savings claims until the benchmark measures them.

## 1. Create the repository

Create:

```text
output-governor/
├── README.md
├── DESIGN.md
├── SPEC_FOR_CHATGPT.md
├── USAGE.md
├── LICENSE
├── CONTRIBUTING.md
├── SECURITY.md
├── pyproject.toml
├── output-governor.example.yaml
├── model-prices.example.yaml
├── src/output_governor/
│   ├── __init__.py
│   ├── governor.py
│   ├── config.py
│   ├── models.py
│   ├── classify.py
│   ├── contracts.py
│   ├── budget.py
│   ├── pricing.py
│   ├── capabilities.py
│   ├── events.py
│   ├── structure.py
│   ├── completeness.py
│   ├── repetition.py
│   ├── stop.py
│   ├── repair.py
│   ├── cli.py
│   ├── providers/
│   ├── validators/
│   ├── ledger/
│   ├── raw_store/
│   ├── proxy/
│   └── integrations/
├── contracts/
├── examples/
├── tests/fixtures/
└── .github/workflows/
```

Use Python 3.11+, a `src` layout, dataclasses or Pydantic consistently, pytest, Ruff, and mypy.

Acceptance:

```bash
python -m pip install -e ".[dev]"
ruff check .
mypy src
pytest
```

## 2. Implement core models

Implement and serialize:

- `GenerationRequest`
- `OutputBudget`
- `AnswerContract`
- `ProviderCapabilities`
- `StreamEvent`
- `CompletenessResult`
- `StopDecision`
- `NormalizedUsage`
- `SavingsResult`
- `LedgerEvent`
- `GovernedResponse`

Use `Decimal` for money. Missing token counts must remain missing, not zero.

## 3. Implement configuration

Support YAML, environment overrides, and allowed request overrides. Validate budgets, repair reserve, modes, contract names, and provider settings.

Model output must never change configuration.

Add example files and precedence tests.

## 4. Implement the contract registry

Load versioned built-in and project contracts.

Build contracts for:

- classification
- extraction
- direct answer
- summary
- code
- plan
- tool decision
- long form
- exact text

Validate format, required fields, budget, risk, stop policy, and incomplete action.

## 5. Implement task classification

Priority:

1. Explicit request contract
2. Route or tool configuration
3. Structured response schema
4. Deterministic request-shape rules
5. Conservative generic fallback

Return class, confidence, reasons, and risk signals. Low confidence selects a conservative contract.

## 6. Implement budgets and pricing

Calculate target, hard, repair, reasoning, and dollar limits.

Requirements:

- Provider maximum validation
- Static dated price registry
- Separate input, visible output, and reasoning rates where available
- No dollar enforcement without a price
- Protected repair reserve
- Deterministic allocation
- No universal price-ratio assumption

Test boundaries and missing prices.

## 7. Implement provider capability negotiation

Define a provider adapter protocol. Each adapter declares support for maximum output, native verbosity, reasoning budget, structured output, stop sequences, stream cancellation, exact usage, and separate reasoning usage.

Record requested, applied, unavailable, ignored, and inferred controls.

## 8. Implement provider adapters

Build recorded-fixture adapters for:

- OpenAI Responses and Chat Completions shapes
- Anthropic Messages
- Gemini generate content
- Generic OpenAI-compatible APIs

Do not call paid APIs in tests. Use captured event fixtures and fake transports.

## 9. Implement the stream event normalizer

Normalize provider events into text deltas, structured deltas, usage, finish reason, and raw event type.

Preserve ordering and provider event metadata. Test split Unicode, empty deltas, late usage, cancellation, errors, and tool-call streams.

## 10. Implement structural tracking

Incrementally track:

- JSON
- XML
- Markdown code fences
- Lists and headings
- Python and TypeScript code boundaries where parsers allow
- Citations
- Tool-call schemas

Expose current validity, safe boundary, open structures, and parse warnings.

Never authorize a stop inside an open string, code fence, JSON object, or required structured field.

## 11. Implement contract validators

Create deterministic validators for every MVP contract.

Test:

- Valid minimum answer
- Missing required element
- Extra but valid detail
- Over-short answer
- Malformed structure
- Exact numbers and identifiers
- Important detail placed late

Validators return score, confidence, passed requirements, and missing requirements.

## 12. Implement repetition and filler detection

Detect exact and near duplicate sentences, repeated conclusions, repeated question text, long preambles, empty transitions, and duplicate code or JSON.

Detection is advisory. It cannot remove facts or protected text by itself.

Add protected-content configuration and tests.

## 13. Implement the safe-stop controller

Use the gates in `DESIGN.md`.

An early stop requires:

- Minimum useful length
- Safe structural boundary
- Required checks passed
- Completeness and confidence thresholds
- No protected section open
- Contract risk allows stopping
- Provider cancellation is supported and tested

Return a typed decision with every reason.

Test every gate alone and in combination.

## 14. Implement modes

- Observe: measure only
- Shadow: calculate potential interventions without changing output
- Enforce: apply controls and safe stopping
- Strict: enforce validation and ledger requirements

Prove in tests that observe and shadow return byte-identical provider content.

## 15. Implement final validation and repair

Validate structure, contract, exact values, stop state, and usage.

Allow one repair pass within the protected reserve. Prefer a patch request naming missing requirements. Do not silently run an unbounded rewrite.

Report gross and net tokens including repair cost.

## 16. Implement honest savings accounting

Keep separate:

- Realized generation savings
- Downstream post-processing savings
- Potential shadow savings

Require a baseline type before calculating avoided tokens.

Support:

- randomized control
- paired replay
- historical contract median
- shadow observation
- none

Label baseline strength. Never count post-processing as current-call savings.

## 17. Implement ledgers and reports

Build JSONL and SQLite adapters.

Reports:

```bash
output-governor report --task ID
output-governor report --by contract
output-governor report --by provider
output-governor report --by stop-reason
output-governor report --format json|csv|markdown
```

Totals must reproduce exactly from fixture events.

## 18. Implement the raw store

Build a content-addressed local filesystem adapter with SHA-256 IDs, atomic writes, metadata, retention, and disabled behavior.

Normal ledger reports must not include full raw prompts or responses.

## 19. Implement the governor

Orchestrate:

1. Resolve contract
2. Classify if needed
3. Allocate budget
4. Negotiate capabilities
5. Adapt request
6. Normalize stream
7. Track structure and completeness
8. Decide safe stop
9. Validate final output
10. Repair if allowed
11. Calculate savings
12. Write ledger event
13. Return response

Use dependency injection so tests need no network.

## 20. Build the OpenAI-compatible proxy

Support Chat Completions and Responses request shapes needed by fixtures.

Requirements:

- Localhost default
- API-key authentication option
- Provider credential forwarding without logging secrets
- Streaming and non-streaming paths
- Governor headers and request metadata
- Timeouts and cancellation
- Health endpoint
- No unsafe shell execution

## 21. Build the CLI

```bash
output-governor inspect REQUEST.json
output-governor proxy --config FILE
output-governor replay FIXTURE_DIRECTORY
output-governor report
output-governor contracts list
output-governor contracts validate FILE
output-governor raw get REF --output FILE
```

Label measured, estimated, downstream, and potential figures.

## 22. Build a simple agent-loop integration

Create a local example using fake providers and fixture tools.

Prove:

- Stable task IDs
- Explicit answer contract
- Tool result passes through token-governor-compatible envelope
- Output budget is applied
- Stream events are normalized
- Ledger event is written
- Shadow mode changes nothing
- Enforce mode stops only at a safe boundary

## 23. Build token-governor composition

Create a public protocol and example, not a private cross-import.

Report one task with:

- Input tokens saved
- Output tokens avoided
- Actual cost
- Separate baseline and estimate notes

The packages must remain independently installable.

## 24. Build the benchmark harness

Fixture corpus must cover all contracts, providers, structure types, repetition cases, late details, malformed streams, and cancellation timing.

Report:

- Actual output tokens
- Realized, downstream, and potential savings
- Contract pass rate
- Exact value retention
- Structure validity
- Incorrect early-stop rate
- Repair rate and net cost
- Task success
- Latency

Output JSON and Markdown.

## 25. Add competitive tests

Encode the product claims as behavior tests:

- A global cap cannot replace a contract.
- Post-processing never increments realized savings.
- Missing baseline never produces avoided tokens.
- Safe stop cannot break structure.
- Low confidence fails open.
- Repair cost reduces net savings.
- Provider controls are recorded as applied or unavailable.
- Shadow mode reports potential only.

## 26. Add CI and security checks

Run lint, types, tests, package build, and dependency audit. Add secret scanning and fixture checks that reject real credentials.

Document private security reporting.

## 27. Prepare later integrations

After MVP tests pass, add protocols and checklists for:

- LiteLLM
- LangChain and LangGraph
- Hermes
- OpenTelemetry

Hermes must start in observe and shadow mode. Use the real hook. Do not invent an API.

## 28. Finish documentation

Update README and USAGE with real install commands and actual benchmark results. Remove any target API that changed during implementation.

Keep source links for competitive statements. Do not claim the category is empty.

## 29. Definition of done

The MVP is done when:

- Clean install succeeds.
- Lint, typing, and tests pass.
- Three provider adapters pass recorded fixtures.
- Observe and shadow are byte-identical to the original response.
- Enforce mode never breaks fixture structure.
- Every contract catches its over-short case.
- Repair stays within reserve.
- Ledger savings classes never mix.
- Replay benchmark produces JSON and Markdown.
- Simple loop and proxy examples work locally.
- README claims match measured evidence.

Return:

1. Final file tree
2. Commands run
3. Test results
4. Benchmark results
5. Known limits
6. Next smallest Hermes task

Do not return another design. Build and verify the repository.
