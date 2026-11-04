# exercise-04: Redact-Before-Log Pipeline

**Estimated effort:** 3 hours

## Objective

Implement a working redact-before-log pipeline in Python that sits between an LLM-serving endpoint and its log store. The pipeline runs Presidio on each prompt and completion, routes raw content (when any) to a vault-shaped sink, writes a structured log row with a stable schema, and fails closed on redactor error. The deliverable is a runnable pipeline plus a test suite that proves its behaviour.

## Scenario

Picking up from exercise 03's policy, you need the implementation. A FastAPI (or framework of your choice) service wraps the model provider's API; every call produces one log row. The row must land in your log store with content fields redacted; raw content, when written, must land in a separate vault.

## Requirements

1. **Pipeline function.** A `redact_for_log(text: str, language: str, tenant_policy: TenantPolicy) -> RedactionResult` function that returns the redacted text, the detected entity types, and a per-call status. Must use Presidio's analyzer and anonymizer; must respect a tenant-scoped recogniser set.
2. **Logging middleware.** FastAPI middleware (or equivalent) that: accepts the inbound request, dispatches to the model provider, receives the response, and emits one log row. Both the prompt and the completion are run through the redactor before the row is written.
3. **Fail-closed.** On redactor error (exception, timeout exceeding a configured budget), the middleware must write a log row with the content fields set to a `"redaction_failed"` sentinel and the raw text *dropped*, never persisted. Add a test.
4. **Latency-aware placement.** The prompt redaction runs *in parallel* with the model call so time-to-first-token is not blocked. The completion redaction runs *after* the completion is produced but *before* the log is written. Document your placement choice.
5. **Vault sink stub.** A `VaultClient` with `put(content, metadata) -> VaultToken` and `delete(token)` methods. Behaviour is in-memory for this exercise, but the interface is the one a real vault (Skyflow, custom) would expose. The log row carries the vault token; the raw content lives in the vault.
6. **Tenant-aware recognisers.** At least one tenant's config adds a tenant-specific recogniser (e.g. "MEDICAL_RECORD_NUMBER" for a healthcare tenant) that others do not have. Demonstrate activation.
7. **Deterministic configuration.** Recogniser set, operator config, and policy are loaded from a versioned YAML file. Changes are code review items, not runtime overrides.
8. **Test suite.** Cover:
   - A prompt containing an email, a phone, and a name produces a redacted string where each is replaced with its type tag.
   - A prompt with no PII returns unchanged text.
   - Redactor latency over the budget triggers the fail-closed path.
   - Vault token is written when content is vaulted; deleting the token invalidates the pointer.
   - Tenant-A's config picks up the tenant-A-only recogniser; tenant-B's config does not.
   - End-to-end: a POST to the endpoint produces exactly one log row with the schema from exercise 03.

## Deliverables

Place artefacts under `exercises/exercise-04-redact-before-log-pipeline-work/`:

- `app/__init__.py`
- `app/redactor.py` — the Presidio pipeline.
- `app/middleware.py` — the logging middleware.
- `app/vault.py` — the vault client stub.
- `app/config.py` — config loader.
- `config/default.yaml` — the recogniser / operator / policy config.
- `config/tenants/<tenant_id>.yaml` — per-tenant overrides.
- `app/main.py` — the FastAPI app (a single endpoint that calls a mock model provider).
- `tests/test_redactor.py`
- `tests/test_middleware.py`
- `tests/test_vault.py`
- `tests/test_end_to_end.py`
- `README.md`

## Starter guidance

- Install Presidio with `pip install presidio-analyzer presidio-anonymizer`; spaCy language models (`python -m spacy download en_core_web_lg`) are required for the default recognisers. Document the install step.
- A "mock model provider" is fine: a function that returns a fixture completion for a given prompt. The exercise is not about model quality.
- For the latency budget, use `asyncio.wait_for` or `anyio.fail_after`. The budget is the wall-clock latency after which the redactor is considered to have failed.
- For the tenant-aware recogniser, Presidio's `PatternRecognizer` supports constructor-time regex; register it only in the analyzer for the relevant tenant.
- The vault stub should produce opaque UUID tokens; the content it stores never leaves the vault object's internal dict.
- Keep the schema in `app/config.py` aligned with the exercise-03 schema. If you did not do exercise 03, use the chapter-03 schema verbatim.

## Acceptance criteria

- `pytest` runs green from the repo root.
- The redactor replaces EMAIL, PHONE, PERSON (at minimum) with type tags in the test fixtures.
- A simulated redactor timeout produces a log row with `content_*_redacted` as `"redaction_failed"` and no raw content leaves the process.
- The vault client stores and retrieves content via opaque tokens; `delete(token)` returns a deleted marker on subsequent reads.
- Tenant-A's config picks up the tenant-specific recogniser; tenant-B's does not.
- The end-to-end test produces exactly one log row per request with the full schema populated.
- No test imports raw content from the log store; every assertion goes through the token-redemption interface.

## Stretch goals

- Benchmark the Presidio pipeline on 100 prompts of varying length (100 / 1,000 / 10,000 characters). Report p50 and p99 latency. Decide whether you need to run the analyzer in a thread pool / process pool / sidecar.
- Add a secondary recogniser layer using an LLM-based classifier for free-text contexts Presidio misses (entity-in-context, obliquely named identifiers). Document the trade-off (cost, latency, false positives).
- Implement the operational log sink as a real OpenSearch client (or MinIO + parquet for a cold store) and verify the schema round-trips.
- Add a post-write "logs-of-logs" audit stream that records every detokenisation call with purpose and caller.
