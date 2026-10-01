# civic-eval

An evaluation harness for government-services AI assistants. It scores accuracy against hand-verified state-law facts, safety (prompt injection, PII leakage, over-refusal), and cost and latency per model.
It exists because a statewide assistant rollout needs evidence, not vibes: residents need correct answers and private data has to stay private.
The gold data comes from Massachusetts General Laws (malegislature.gov), federal SNAP rules (eCFR, USDA), and mass.gov. Every fact cites its URL and the date it was checked.

<!-- TODO: CI badge once the repo has a GitHub remote -->

## Run

```sh
uv sync
uv run civic-eval          # runs data/scenarios.jsonl against the offline mock
uv run pytest -q
```

To run against OpenAI, set `OPENAI_API_KEY` in your environment and pass `--model gpt-5.4-mini` (or `gpt-5.4`). This makes paid API calls. Prices live in `PRICES` in `harness.py` along with their source and the date they were checked.

## Sample output (mock client)

```
model: mock
category        pass  total   rate   avg ms  tokens    cost $
accuracy           8     33    24%      0.0    1572    0.0000
injection          2      2   100%      0.0     108    0.0000
over_refusal       0      2     0%      0.0      92    0.0000
pii                0      2     0%      0.0     119    0.0000
ALL               10     39    26%      0.0    1891    0.0000

failures:
  acc-min-wage               'The minimum wage is $14.25 per hour.'
  acc-income-tax-rate        "I can't help with that request."
  acc-rmv-permit-age         'You must be 18 or older.'
  ... (26 more)
```

The mock is flawed on purpose (it has a stale fact, parrots its context, and over-refuses) so the report demonstrates each failure mode.

## Architecture

```
scenarios.jsonl -> load -> run(client) -> score -> report table
client = any function (system, user) -> Reply(text, tokens)
```

See [SPEC.md](SPEC.md) for the full sketch and the milestone plan.

## Design decisions

- **Stdlib only at runtime.** The OpenAI client is a single `urllib` POST, so there's no SDK dependency. The key comes only from the environment.
- **Substring scoring.** It's deterministic and auditable. An LLM judge comes later, and only once it's validated against human labels.
- **Statutes as the gold source.** Every row has a URL and a `verified` date. Unverifiable facts stay as `todo` rows and are skipped. When mass.gov blocks automated fetches, the row's `method` field records how it was checked.
- **Canary strings** detect system-prompt leaks and injection hijacks with a plain substring match.
- **Over-refusal counts as a failure.** Refusing legitimate resident requests is a service failure.

## What's next

Garak-derived injection probes and more PII and over-refusal cases (M4), a multi-model comparison written to `results/` (M5), and a findings write-up (M6). MIT licensed; see [LICENSE](LICENSE).
