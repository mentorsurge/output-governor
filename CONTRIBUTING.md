# Contributing

Issues and pull requests are welcome.

A change to a budget, detector, or validator must include:

- A fixture showing the problem
- The shortest acceptable answer
- An over-short failure case
- A structure or completeness invariant
- Token and latency measurements
- No savings claim without a named baseline

Do not add a percent to the README or the repo description unless a ledger run with realized or downstream tokens is in the same change.

Planned checks once code exists:

```
ruff check .
mypy src
pytest
```
