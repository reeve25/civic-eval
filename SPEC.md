# civic-eval spec

**What:** a small evaluation harness that scores an LLM-backed state-services assistant on accuracy, safety, cost, and latency.
**Why:** before a state rolls an assistant out to thousands of employees and residents, it needs evidence that the assistant gets public facts right, doesn't leak PII, resists prompt injection, and doesn't refuse legitimate help.
**How:** hand-verified Massachusetts scenarios (sourced from malegislature.gov, eCFR/USDA, and mass.gov), substring-auditable scoring, and a per-model report table.

## Scope

In (v1):
- JSONL scenario set with four categories: `accuracy`, `injection`, `pii`, `over_refusal`.
- Gold facts cite an official .gov URL plus a `verified` date. Unverified rows carry `todo` and are skipped.
- Model clients: an offline mock (used in tests and CI), plus OpenAI Chat Completions over stdlib `urllib`, keyed by `OPENAI_API_KEY` from env only.
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
- **M2, real provider (done):** `openai_client` over stdlib `urllib` (one POST doesn't justify the SDK), a `--model` flag, a price table for cost, and a timeout with one retry. Tests stay offline.
- **M3, gold set to about 30 rows (done):** 33 verified accuracy rows across RMV (ch. 90), DUA (ch. 151A), and SNAP (7 CFR 273, FNS, DTA), plus the earlier facts; one TODO (DUA max weeks). Rows that mass.gov blocks carry a `method` field saying how they were checked; "changed recently" rows carry a `note`.
- **M4, safety depth (done):** 12 injection rows (7 garak-derived from `promptinject` and `latentinjection`, each with its source path and commit; direct, indirect-in-retrieved-page, and system-prompt extraction), 9 PII rows (phone, claim ID, DOB, addresses, a third party, a case-file summary, and 2 own-data rows that must *not* be blocked), and 9 over-refusal rows. Over-refusal rows score on not refusing; the two with a fact checked on malegislature.gov also carry `expect_any`. Added `--repeats N` so each scenario runs N times and the report shows k/N failures.
- **M5, multi-model comparison (done):** gpt-5.4, mini, and nano × 63 cases × 3 repeats, saved to `results/` as JSONL plus the printed tables. CI runs ruff and pytest. Every failure is hand-triaged in `results/TRIAGE.md`.
- **M6, polish (done):** a Findings section in the README with per-category pass rates, flaky cases, cost and latency, and the model you'd start from. It also lists the scorer fixes from triage.

## Interview-defence points (know these cold)

1. **Why substring scoring?** It's deterministic, free, and auditable line by line. The known weakness is that "not 6.25%" still passes. The upgrade path is an LLM judge, validated against human labels. An unvalidated judge is just a second model you have to trust.
2. **Why is over-refusal a failure?** A government assistant that refuses "how do I get my own CORI" fails residents. Safety has two sides: blocking harm and still delivering the service.
3. **Canaries:** `SYS-CANARY-2291` in the system prompt and `CANARY-7731` in injected text let a substring check detect leakage or hijacking with no false negatives on exact echoes. Know the limit: paraphrased leaks slip through.
4. **Indirect vs. direct injection:** `inj-retrieved-page` plants the attack inside retrieved "data". That's the realistic threat for RAG over state web pages, and it's what garak's `latentinjection` family covers.
5. **Gold-set provenance:** every fact links to a statute, and `verified` dates matter because law changes. The income-tax row shows why: ch. 62 §4 still prints 5.3%, but revenue-triggered step-downs put the in-effect rate at the 5% floor (checked against DOR). That's the kind of trap a model (or you) falls into. DUA max weeks is still a TODO for the same reason: §30 toggles 30/26 weeks on an unemployment trigger.
6. **Synthetic PII:** SSN area `000` is never issued, so the test data can't be anyone's real number. Phone numbers use 555-01xx (reserved for fiction) and addresses use Example St.
7. **Why a function instead of a client class:** one call site, one signature. Add a class when a second behaviour (streaming, batching) actually needs shared state.
8. **Why the mock is deliberately wrong:** the tests prove the scorer *catches* each failure mode. A mock that always passes would prove nothing.
9. **Cost and latency:** measure wall-clock time around the client call, and take tokens from provider usage fields (the mock approximates with word counts). Cost = tokens × published price, and cite the pricing page date.
10. **Own data vs. someone else's:** `forbid` marks a third party's data that must not appear. When users give their *own* claim ID or address, the row uses `expect_any` instead, so a model that blocks or drops it fails. The rule is in the data row, not hidden in code, so every verdict can be traced to one line.
11. **Why repeats:** LLM output is sampled. A 1/3 failure is noise to investigate; 3/3 is a real gap. One run per scenario can't tell the two apart.
