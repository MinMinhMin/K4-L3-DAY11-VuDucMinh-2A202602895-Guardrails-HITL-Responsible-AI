# Guardrails HITL Lab Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the Day 11 guardrails, defense-in-depth pipeline, generated artifacts, and five red-team prompts so the lab's mandatory and bonus paths are runnable and verifiable.

**Architecture:** Keep the starter's ADK callback interfaces and implement deterministic offline defenses around them. Build the assignment suite as a direct callback harness that exports `results.json`, audit, and metrics artifacts; leave live LLM calls isolated to Checkpoint 4.

**Tech Stack:** Python 3.10+, Google ADK / `google.genai.types`, OpenAI-compatible runtime, pytest, jsonschema, standard-library regex/URL parsing/JSON.

**Spec:** `docs/superpowers/specs/2026-09-26-guardrails-hitl-lab-design.md`

## Global Constraints

- Keep the demo values in `data/protected/vinbank_secrets.json` unchanged.
- Do not commit `.env`, real API keys, or hand-written placeholder artifacts.
- Blue remains locked to OpenRouter `liquid/lfm-2.5-2.6b`; do not modify the agent secret prompts.
- Offline defense work must not require an LLM API call.
- Production plugin order is `RateLimitPlugin → InputGuardrailPlugin → OutputGuardrailPlugin`.
- Egress decisions are deterministic and must not ask an LLM.
- If a Red/Red Advance provider call fails because credentials are invalid, expired, unauthorized, forbidden, or otherwise rejected by the provider, stop and report the `.env` problem rather than fabricating success.
- Generate output JSON through the suite/runner and `scripts/grade.py`, not by manually editing JSON.

## Review Focus

- Zero-width Unicode inside an injection phrase must still block; covered by `test_detect_injection_strips_zero_width_characters` in Task 1.
- A benign external transfer-document summary must remain allowed; covered by `test_benign_external_document_is_allowed` in Task 1.
- A password-like token without a literal `password:` label must not bypass output redaction when it is a protected API key/DB host; covered by `test_content_filter_redacts_all_protected_output_classes` in Task 2.
- A user at the rate-limit boundary must be isolated from another user and allowed after expiry; covered by `test_rate_limit_is_per_user_and_expires` in Task 3.
- A lookalike hostname, non-HTTPS URL, or URL user-info trick must not pass egress; covered by `test_egress_requires_exact_https_vinbank_host` in Task 4.

### Task 1: Input Guardrails

**Files:**
- Modify: `src/guardrails/input_guardrails.py`
- Test: `tests/unit/test_input_guardrails.py`

**Interfaces:**
- Consumes: `ALLOWED_TOPICS` and `BLOCKED_TOPICS` from `core.config`.
- Produces: `detect_injection(text) -> Literal["ALLOW", "BLOCK"]`, `topic_filter(text) -> Literal["ALLOW", "BLOCK"]`, and `InputGuardrailPlugin.on_user_message_callback(...) -> types.Content | None`.

- [ ] **Step 1: Write the failing tests**

  Add tests for the required injection families, zero-width Unicode normalization, benign external-document text, blocked/off-topic topics, a valid banking topic, and plugin refusal/pass-through with counters.

- [ ] **Step 2: Run the focused tests to verify RED**

  Run: `pytest tests/unit/test_input_guardrails.py -q`

  Expected: FAIL because the starter patterns and callback/topic implementation are incomplete.

- [ ] **Step 3: Implement the minimal input defenses**

  Normalize Unicode format/zero-width characters and whitespace before applying at least five case-insensitive regex signals. Check blocked topics before requiring an allowed topic. Return clear refusal `types.Content` from the plugin and `None` only when both statuses are `ALLOW`.

- [ ] **Step 4: Run focused and existing guardrail tests**

  Run: `pytest tests/unit/test_input_guardrails.py tests/public/test_lab_contracts.py -q`

  Expected: all selected tests PASS.

- [ ] **Step 5: Commit**

  ```bash
  git add tests/unit/test_input_guardrails.py src/guardrails/input_guardrails.py
  git commit -m "feat: implement input guardrails"
  ```

### Task 2: Output Guardrails

**Files:**
- Modify: `src/guardrails/output_guardrails.py`
- Test: `tests/unit/test_output_guardrails.py`

**Interfaces:**
- Consumes: `types.Content`-like model responses and the existing optional judge hook.
- Produces: `content_filter(response) -> dict` with `safe`, `issues`, `redacted`; `OutputGuardrailPlugin.after_model_callback(...)` mutating or replacing the response safely.

- [ ] **Step 1: Write the failing tests**

  Add tests that assert clean banking text remains safe, phone/email/CCCD/API-key/password/DB-host categories are detected and replaced with `[REDACTED]`, the returned keys are always present, and the plugin redacts deterministic findings while preserving clean responses.

- [ ] **Step 2: Run the focused tests to verify RED**

  Run: `pytest tests/unit/test_output_guardrails.py -q`

  Expected: FAIL because the pattern table and plugin callback are still TODOs.

- [ ] **Step 3: Implement deterministic filtering and callback behavior**

  Use named regexes and count each category from the original response, then apply replacements to the response. In the callback, replace content with the redacted text for deterministic issues; only use the optional judge when explicitly enabled and replace unsafe judged output with a safe refusal.

- [ ] **Step 4: Run focused and public tests**

  Run: `pytest tests/unit/test_output_guardrails.py tests/public/test_lab_contracts.py -q`

  Expected: all selected tests PASS.

- [ ] **Step 5: Commit**

  ```bash
  git add tests/unit/test_output_guardrails.py src/guardrails/output_guardrails.py
  git commit -m "feat: implement output guardrails"
  ```

### Task 3: Rate Limiting, Audit, and Monitoring

**Files:**
- Modify: `src/assignment/rate_limiter.py`
- Modify: `src/assignment/audit_log.py`
- Modify: `src/assignment/monitoring.py`
- Test: `tests/unit/test_assignment_observability.py`

**Interfaces:**
- Consumes: ADK callback context/user message objects, request IDs, and the existing dataclass thresholds.
- Produces: working `RateLimitPlugin.on_user_message_callback`, `AuditLogPlugin.record_input/record_output/export_json`, and `MonitoringAlert.check_metrics/export_json/snapshot`.

- [ ] **Step 1: Write the failing tests**

  Add tests for sliding-window blocking and retry text, per-user isolation and expiry, audit latency/request correlation plus JSON export, zero-safe metrics, threshold alerts, and metrics export.

- [ ] **Step 2: Run the focused tests to verify RED**

  Run: `pytest tests/unit/test_assignment_observability.py -q`

  Expected: FAIL on the starter `NotImplementedError`/missing behavior.

- [ ] **Step 3: Implement the three deterministic components**

  Prune expired timestamps and append only allowed requests; store open timestamps keyed by request ID/user; serialize UTF-8 JSON with parent-directory creation; compute rates and append structured `Alert` objects when thresholds are exceeded without duplicating alerts on repeated checks.

- [ ] **Step 4: Run focused and smoke tests**

  Run: `pytest tests/unit/test_assignment_observability.py tests/smoke -q`

  Expected: all selected tests PASS.

- [ ] **Step 5: Commit**

  ```bash
  git add tests/unit/test_assignment_observability.py src/assignment/rate_limiter.py src/assignment/audit_log.py src/assignment/monitoring.py
  git commit -m "feat: add rate limit and observability layers"
  ```

### Task 4: Pipeline Assembly and Egress Policy

**Files:**
- Modify: `src/assignment/pipeline.py`
- Test: `tests/unit/test_pipeline.py`

**Interfaces:**
- Consumes: Task 1/2 guardrail plugins and Task 3 observer classes.
- Produces: `is_egress_allowed(destination, payload) -> bool`, `build_production_plugins(...) -> list`, and `build_observability() -> tuple[AuditLogPlugin, MonitoringAlert]`.

- [ ] **Step 1: Write the failing tests**

  Add tests for exact HTTPS VinBank allowlisting, rejection of unknown/non-HTTPS/user-info/lookalike hosts, sensitive payload rejection, exact plugin order and configuration, and observer construction.

- [ ] **Step 2: Run the focused tests to verify RED**

  Run: `pytest tests/unit/test_pipeline.py -q`

  Expected: FAIL because pipeline functions are `NotImplementedError`.

- [ ] **Step 3: Implement pipeline assembly and deterministic egress**

  Import and instantiate the three plugins in the required order. Parse URLs with `urllib.parse`, require HTTPS and exact approved hostnames, and scan payloads for demo secrets/DB host/phone/email patterns without using an LLM.

- [ ] **Step 4: Run focused and public tests**

  Run: `pytest tests/unit/test_pipeline.py tests/public/test_lab_contracts.py -q`

  Expected: all selected tests PASS.

- [ ] **Step 5: Commit**

  ```bash
  git add tests/unit/test_pipeline.py src/assignment/pipeline.py
  git commit -m "feat: assemble defense pipeline and egress policy"
  ```

### Task 5: Offline Assignment Suite and Defense Artifacts

**Files:**
- Modify: `src/assignment/pipeline.py`
- Modify: `src/main.py` (avoid API-key setup for offline parts)
- Test: `tests/unit/test_assignment_suite.py`

**Interfaces:**
- Consumes: plugin/observer factories from Task 4 and callback behavior from Tasks 1–3.
- Produces: `async run_assignment_suite(pipeline) -> dict` and generated repo-root `outputs/results.json`, `outputs/audit_log.json`, `outputs/metrics.json` matching the schema contract.

- [ ] **Step 1: Write the failing tests**

  Add an async test that runs the suite without credentials and asserts framework, safe/attack/edge minimum counts, all safe queries unblocked, at least five attacks blocked, rate-limit arithmetic, schema validation, and generated artifact existence.

- [ ] **Step 2: Run the focused test to verify RED**

  Run: `pytest tests/unit/test_assignment_suite.py -q`

  Expected: FAIL because `run_assignment_suite` is not implemented.

- [ ] **Step 3: Implement the offline callback harness and exports**

  Run safe, attack, rate-limit, and edge cases through the plugin list, record request/output events, update monitoring counters, use simulated response text for output redaction, produce rows with `input`, `blocked`, `layer`, and `response_preview`, and write artifacts under the repository `outputs/` directory. Keep `--part 2` and `--part 3` free of `setup_api_key()` calls.

- [ ] **Step 4: Run the suite and contract checks**

  Run: `python src/main.py --part 3`, then `pytest tests/unit/test_assignment_suite.py tests/public -q`.

  Expected: `results.json`, `audit_log.json`, and `metrics.json` exist; schema/public tests PASS with safe blocked count 0, attack blocked count at least 5, and rate-limit blocked count at least 1.

- [ ] **Step 5: Commit**

  ```bash
  git add tests/unit/test_assignment_suite.py src/assignment/pipeline.py src/main.py outputs/results.json outputs/audit_log.json outputs/metrics.json
  git commit -m "feat: generate defense assignment artifacts"
  ```

### Task 6: Red-Team Prompts and Credential-Failure Handling

**Files:**
- Modify: `src/attacks/attacks.py`
- Modify: `src/main.py` if needed for clear fail-fast reporting
- Test: `tests/unit/test_attacks.py`

**Interfaces:**
- Consumes: existing Red/Red Advance factories and `adversarial_prompts`/`run_attacks` interfaces.
- Produces: five non-placeholder adversarial prompt records with distinct techniques, plus live API failures surfaced clearly instead of being recorded as successful attack results.

- [ ] **Step 1: Write the failing tests**

  Add tests that assert five prompts, distinct required categories, substantial prompt text, no `TODO`, and that a simulated authentication/expired-key error from `chat_with_agent` is raised as a credential failure rather than converted into a normal result row.

- [ ] **Step 2: Run the focused tests to verify RED**

  Run: `pytest tests/unit/test_attacks.py -q`

  Expected: the prompt-quality assertion fails on the starter TODOs; the error-propagation assertion fails because `run_attacks` currently catches every exception.

- [ ] **Step 3: Implement prompt records and fail-fast provider handling**

  Write detailed completion, translation/reformatting, hypothetical/creative, confirmation/side-channel, and multi-step prompts. Detect provider authentication/authorization/expired-key/quota failures from the exception type/message, raise a clear `RuntimeError` with `.env` repair guidance, and preserve existing handling for non-credential per-prompt errors.

- [ ] **Step 4: Run focused tests and offline regression**

  Run: `pytest tests/unit/test_attacks.py tests/smoke tests/public -q`.

  Expected: all tests PASS; no external API call is made by this command.

- [ ] **Step 5: Commit**

  ```bash
  git add tests/unit/test_attacks.py src/attacks/attacks.py src/main.py
  git commit -m "feat: add adversarial prompts and credential fail-fast"
  ```

### Task 7: Final Verification and Live Red Run

**Files:**
- Generated by commands: `outputs/attack_results.json`, `outputs/unsafe_attack_result.json`, `outputs/guards_attack_result.json`, `outputs/grade_report.json`, `outputs/lab_report.md`

- [ ] **Step 1: Run complete offline verification**

  Run: `pytest tests/smoke -q && pytest tests/public -q && python scripts/grade.py --submission-dir . --out outputs/grade_report.json`.

  Expected: smoke/public tests PASS, schema/package checks are valid, and grader report is generated without technical failure.

- [ ] **Step 2: Inspect tracked-secret safety and artifact shape**

  Run: `git status --short --branch` and a targeted check that `.env` is absent from tracked files; inspect artifact keys/counts without printing API-key values.

  Expected: `.env` is not tracked; results/attack artifacts have the required top-level keys.

- [ ] **Step 3: Check configured provider presence without printing secrets**

  Read only whether the required `.env` variable names are present/nonempty. If a key is absent, stop and report that the user must configure `.env` before Checkpoint 4.

- [ ] **Step 4: Run live Red/Red Advance only when credentials are present**

  Run: `python src/main.py --part 4`.

  Expected: both per-target files and combined `outputs/attack_results.json` are generated. If the provider rejects the key or reports expiry/authorization/quota failure, stop immediately and report the error without claiming a leak or bonus.

- [ ] **Step 5: Re-run grader after live artifacts**

  Run: `python scripts/grade.py --submission-dir . --out outputs/grade_report.json`.

  Expected: packaging/schema/public checks remain valid; live Red/Red Advance leak counts are reported as observed, not inferred.

