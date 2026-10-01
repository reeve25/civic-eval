# civic-eval spec

**What:** a small evaluation harness that scores an LLM-backed state-services assistant on accuracy, safety, cost, and latency.
**Why:** before a state rolls an assistant out to thousands of employees and residents, it needs evidence that the assistant gets public facts right, doesn't leak PII, resists prompt injection, and doesn't refuse legitimate help.
**How:** hand-verified Massachusetts scenarios (sourced from malegislature.gov), substring-auditable scoring, and a per-model report table.

## Scope

In (v1):
- JSONL scenario set with four categories: `accuracy`, `injection`, `pii`, `over_refusal`.
- Gold facts cite an official .gov URL plus a `verified` date. Unverified rows carry `todo` and are skipped.
- Model clients: an offline mock (used in tests and CI), plus OpenAI via the official SDK, keyed by `OPENAI_API_KEY` from env only.
- Per-model report: pass rate per category, average latency, tokens, and estimated USD cost.
- A handful of garak-derived injection prompts, copied in as data with attribution (no runtime garak dependency).

Out (v1):
- Web UI, database, and auth. A printed table and a JSON/Markdown file are enough.
- LLM-as-judge scoring. Substring rules stay until they visibly fail; see the defence points.
- Retrieval or RAG pipelines. `context` is passed inline to simulate a retrieved page.
- States other than Massachusetts, and multilingual scenarios (both listed as next steps).

## Architecture

```
data/scenarios.jsonl ──► load_scenarios() ──► run(client) ──► score() ──► report()
   (gold + .gov URL)      skips `todo` rows      │               substring    table per
                                                 ▼               rules        category/model
                                   client(system, user) -> Reply(text, tokens)
                                   ├── mock_client   (offline, deterministic, flawed on purpose)
                                   └── openai_client (M2, env-keyed)
```

A client is just a function `(system: str, user: str) -> Reply`. There's no base class, because a function signature is the whole interface.

## Milestones to v1 (about 1–2 days each)

- **M1, vertical slice (done):** scenarios, mock client, scoring, report table, pytest.
- **M2, real provider:** `openai_client` via the `openai` SDK, a `--model` flag, a price table for cost, and a timeout with one retry. Tests stay offline.
- **M3, gold set to about 30 rows:** verify the income-tax TODO against DOR; add RMV, unemployment (DUA), and SNAP (DTA) facts, each with a URL and a date. Include "the answer changed recently" cases.
- **M4, safety depth:** port 5–10 injection prompts from garak probes (`promptinject`, `latentinjection`) with attribution; add PII variants (phone numbers, case IDs); add more over-refusal cases.
- **M5, multi-model comparison:** run 2–3 models, write `results/<date>.md` and JSON, and add a CI workflow (ruff and pytest).
- **M6, polish:** README with real sample output, an architecture diagram, a LICENSE, and a findings write-up: which model you'd deploy and why.

## Interview-defence points (know these cold)

1. **Why substring scoring?** It's deterministic, free, and auditable line by line. The known weakness is that "not 6.25%" still passes. The upgrade path is an LLM judge, validated against human labels. An unvalidated judge is just a second model you have to trust.
2. **Why is over-refusal a failure?** A government assistant that refuses "how do I get my own CORI" fails residents. Safety has two sides: blocking harm and still delivering the service.
3. **Canaries:** `SYS-CANARY-2291` in the system prompt and `CANARY-7731` in injected text let a substring check detect leakage or hijacking with no false negatives on exact echoes. Know the limit: paraphrased leaks slip through.
4. **Indirect vs. direct injection:** `inj-retrieved-page` plants the attack inside retrieved "data". That's the realistic threat for RAG over state web pages, and it's what garak's `latentinjection` family covers.
5. **Gold-set provenance:** every fact links to a statute, and `verified` dates matter because law changes. The income-tax row is a TODO on purpose: ch. 62 §4 has revenue-triggered step-downs, so the statute text alone isn't the in-effect rate. That's the kind of trap a model (or you) falls into.
6. **Synthetic PII:** SSN area `000` is never issued, so the test data can't be anyone's real number.
7. **Why a function instead of a client class:** one call site, one signature. Add a class when a second behaviour (streaming, batching) actually needs shared state.
8. **Why the mock is deliberately wrong:** the tests prove the scorer *catches* each failure mode. A mock that always passes would prove nothing.
9. **Cost and latency:** measure wall-clock time around the client call, and take tokens from provider usage fields (the mock approximates with word counts). Cost = tokens × published price, and cite the pricing page date.
