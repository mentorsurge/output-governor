# Using output-governor

Author: Joseph Farzinzad

This guide describes the target interface after the MVP is built.

## Install

```bash
git clone https://github.com/mentorsurge/output-governor.git
cd output-governor
python -m venv .venv
source .venv/bin/activate
python -m pip install -e ".[dev]"
ruff check .
mypy src
pytest
```

Published package:

```bash
python -m pip install output-governor
```

## Start in shadow mode

Do not enforce short answers first. Run shadow mode, collect real response shapes, then set contracts.

```yaml
mode: shadow
ledger:
  adapter: sqlite
  path: .output-governor/ledger.db
raw_store:
  enabled: true
  path: .output-governor/raw
  retention_days: 7
fallback: fail_open
```

Shadow mode returns the original response. It records potential controls and potential savings only.

## Configuration

```yaml
mode: enforce
budget:
  task_output_tokens: 5000
  task_dollar_limit: null
  repair_reserve_percent: 15
contracts:
  default: direct_answer
  directory: ./contracts
stop:
  enabled: true
  minimum_completeness: 1.0
  minimum_confidence: 0.95
pricing:
  source: static
  file: ./model-prices.yaml
ledger:
  adapter: sqlite
  path: .output-governor/ledger.db
raw_store:
  enabled: false
providers:
  openai:
    enabled: true
  anthropic:
    enabled: true
  gemini:
    enabled: true
fallback: fail_open
```

Environment overrides:

```bash
export OUTPUT_GOVERNOR_CONFIG=./output-governor.yaml
export OUTPUT_GOVERNOR_MODE=shadow
export OUTPUT_GOVERNOR_TASK_OUTPUT_TOKENS=5000
export OUTPUT_GOVERNOR_REPAIR_RESERVE_PERCENT=15
export OUTPUT_GOVERNOR_LEDGER_PATH=.output-governor/ledger.db
export OUTPUT_GOVERNOR_RAW_STORE_ENABLED=false
```

Configuration order:

1. Built-in defaults
2. YAML file
3. Environment variables
4. Explicit application settings allowed by policy

## Library integration

```python
from output_governor import Governor, GovernorConfig

config = GovernorConfig.from_yaml("output-governor.yaml")
governor = Governor(config)

response = governor.generate(
    provider="anthropic",
    model="claude-sonnet",
    messages=messages,
    task_id="ticket-42",
    call_id="answer-1",
    contract="direct_answer",
    budget={
        "target_tokens": 140,
        "hard_tokens": 220,
        "repair_reserve_tokens": 40,
    },
)

print(response.content)
```

Use stable task and call IDs. A task ID groups the full economics of one outcome.

## Proxy integration

```bash
output-governor proxy \
  --config output-governor.yaml \
  --listen 127.0.0.1:8788
```

Point an OpenAI-compatible client at the proxy:

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://127.0.0.1:8788/v1",
    api_key="local-proxy-key",
)

response = client.chat.completions.create(
    model="gpt-5",
    messages=[{"role": "user", "content": "Classify this ticket"}],
    extra_headers={
        "X-Output-Governor-Task": "ticket-42",
        "X-Output-Governor-Contract": "classification",
    },
)
```

Bind to localhost by default. Add authentication and TLS before exposing the proxy.

## Define a contract

```yaml
name: support_reply
version: 1
format: markdown
required:
  - answer
  - next_step
budget:
  target_tokens: 120
  hard_tokens: 180
  repair_reserve_tokens: 30
structure:
  max_paragraphs: 3
style:
  no_preamble: true
  no_repeated_summary: true
risk: low
stop:
  semantic_early_stop: true
on_incomplete: repair_with_remaining_budget
```

A contract should define what complete means. Do not start with a token number and hope the answer fits.

## Choose budgets

Start from real shadow data:

1. Group responses by task type.
2. Find the shortest answers that still pass review.
3. Set the target near that range.
4. Set the hard limit high enough for unusual valid cases.
5. Reserve 10 to 20 percent for one repair pass.
6. Test late-detail and edge-case fixtures.

Raise the budget when required fields are frequently missing or repair use is high. Lower it when answers pass consistently with repeated filler or unused headroom.

Use level 0, observe-only, or a conservative contract for exact legal text, code migration, security analysis, medical or financial explanations, and other high-risk work.

## Read the ledger

```bash
output-governor report --task ticket-42
output-governor report --by contract
output-governor report --by provider
output-governor report --by stop-reason
output-governor report --format csv > output-cost.csv
```

Key columns:

| Column | Meaning |
|---|---|
| `mode` | observe, shadow, enforce, or strict |
| `contract_name` | Answer contract used |
| `controls_requested` | Limits and controls the governor asked for |
| `controls_applied` | Controls the provider adapter could apply |
| `baseline_type` | Source of the counterfactual output length |
| `baseline_tokens` | Counterfactual output tokens |
| `actual_visible_tokens` | Visible answer tokens reported or counted |
| `actual_reasoning_tokens` | Hidden reasoning tokens when reported separately |
| `realized_tokens_avoided` | Tokens not generated under enforcement, against the named baseline |
| `downstream_tokens_removed` | Tokens removed after generation |
| `potential_tokens_avoidable` | Shadow or post-processing estimate |
| `estimated_realized_savings` | Realized tokens avoided times configured output price |
| `completeness_score` | Required contract coverage |
| `completeness_confidence` | Confidence in the score |
| `structure_valid` | Whether JSON, code, Markdown, or tool shape is valid |
| `repair_calls` | Number of bounded repair calls |
| `stop_reason` | Provider finish, hard cap, safe stop, fallback, or error |

No baseline means no avoided-token claim. Post-processing belongs under downstream savings.

## Roll out enforcement

Recommended order:

1. Observe mode to verify provider adapters and usage.
2. Shadow mode on one task family.
3. Review false stop points and missing requirements.
4. Add fixtures and tighten the contract.
5. Enforce for a small allowlist.
6. Compare task success, tokens, repair rate, and latency against control.
7. Expand only when quality holds.

## Compose with token-governor

```python
raw_tool_result = tool.execute(call)
input_result = token_governor.process(
    task_id=task_id,
    call_id=call.id,
    result=raw_tool_result,
    request_hint=call.purpose,
)

output = output_governor.generate(
    task_id=task_id,
    call_id="model-2",
    messages=build_messages(input_result.content),
    contract="direct_answer",
)
```

Use the same task ID. Keep input tokens saved and output tokens avoided in separate fields, then sum cost at the task report layer.

## Troubleshooting

### Answers stop mid-sentence

Disable semantic early stop for that contract. Check sentence-boundary and structure trackers. Add the failure as a fixture.

### JSON or tool calls are invalid

Require the provider's structured-output feature when available. Never cancel until the incremental parser reports a closed valid object. Use fail-open or one bounded repair.

### Required details are missing

Raise the hard budget, improve the contract requirements, lower the stop threshold, or disable early stop for the task type. Check whether the missing detail usually appears late.

### Repairs erase savings

Increase the first-attempt budget or fix the contract. Frequent repair means the initial budget is wrong. Report net cost after repair, not first-attempt savings.

### Shadow savings look large but enforcement saves little

Shadow results are potential savings. The provider may ignore controls, cancellation may arrive late, or the baseline may be weak. Check `controls_applied`, usage, stop time, and baseline type.

### The provider reports no exact usage

Use a provider tokenizer or a labeled estimate. Do not turn missing usage into zero. Dollar totals remain estimated.

### Hidden reasoning dominates cost

Use provider-native reasoning limits when supported. Visible-text trimming will not solve hidden reasoning spend. Record visible and reasoning tokens separately.

### Latency increases

Disable expensive validators for low-value calls. Use deterministic checks first. A judge that costs more than the saved output is a loss.

### Filler detection removes useful safety text

Protect required policy language in the contract. The detector can recommend a stop but cannot delete protected content.

### Raw response recovery

Enable local raw storage during development:

```yaml
raw_store:
  enabled: true
  path: .output-governor/raw
  retention_days: 7
```

Retrieve by ledger reference:

```bash
output-governor raw get sha256:abc123 --output /tmp/original-response.json
```

Raw responses can contain private data. Keep them out of normal reports and use short retention.

## Hermes integration

**Later phase. Not part of the MVP.**

1. Find the real Hermes model-call and stream hooks.
2. Run observe mode without changing responses.
3. Add contract metadata at the task boundary.
4. Run shadow mode and collect potential stop points.
5. Add a feature flag and task allowlist.
6. Enforce on fixed replayable tasks.
7. Keep immediate rollback.

Do not invent Hermes APIs. If streaming cancellation is unavailable, use preflight limits and validation only.

## Production checklist

- Provider controls verified with fixtures
- Contracts versioned
- Baselines named
- Structure and late-detail tests passing
- Repair reserve bounded
- Proxy authenticated
- Raw retention approved
- Prices current and dated
- Fail-open behavior tested
- Task success compared against control
- No downstream or potential savings labeled realized

## Author

Joseph Farzinzad
