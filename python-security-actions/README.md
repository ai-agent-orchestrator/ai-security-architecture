# Python Security Actions (9)

The nine security actions implemented in Python for the legal intake service, brought here so this architecture repository holds working code next to its design documents.

**Source:** [ai-agent-orchestrator/legal-intake-fastapi-lab](https://github.com/ai-agent-orchestrator/legal-intake-fastapi-lab) at commit `4fc98d5` (2026-11-10). `app/security/*.py` and the test files listed below are copied byte for byte; the import path `app.security` is kept so they stay comparable with the source. The source repository remains the canonical copy until a sync rule is decided (see "Open questions").

## The nine actions

| # | Action | File | Job | Main codes |
| --- | --- | --- | --- | --- |
| 1 | `validate_request_context` | `session_actions.py` | Check that request, snapshot and response belong together | raises `SecurityError("REQUEST_CONTEXT_MISMATCH", 500)` |
| 2 | `check_session_risk` | `session_actions.py` | Cumulative risk and already-blocked sessions | `DYNAMIC_SESSION_BLOCKED` |
| 3 | `check_input_policy` | `policy_actions.py` → `policy_core.py` | Legal abuse, jailbreak, prompt injection, secret and privacy requests | `BLOCKED_LEGAL_HARM`, `BLOCKED_JAILBREAK`, `BLOCKED_PROMPT_INJECTION`, `BLOCKED_SECRET_REQUEST`, `BLOCKED_AUTH_BYPASS`, `BLOCKED_PRIVACY_RISK` |
| 4 | `mask_personal_data` | `policy_actions.py` → `policy_core.py` | Replace defined personal data before storage, external sending or return | `PERSONAL_DATA_MASKED` (decision `TRANSFORM`) |
| 5 | `check_generation_admission` | `resource_actions.py` → `admission.py` | Concurrency limit, bounded queue, wait limit | raises `SecurityError("GENERATION_CAPACITY_EXCEEDED")` |
| 6 | `check_request_budget` | `resource_actions.py` → `budget_store.py` | Atomic budget check and reservation | `COST_LIMIT_BLOCKED` |
| 7 | `check_output_policy` | `policy_actions.py` → `policy_core.py` | Empty, length and policy check on the whole generation | `OUTPUT_POLICY_BLOCKED` (detailCode `EMPTY_OUTPUT`, `OUTPUT_TOO_LONG` or the matched policy) |
| 8 | `check_marketing_response` | `marketing_actions.py` → `marketing_core.py` | Mark guarantee and exaggeration wording for human review | `MARKETING_REVIEW_REQUIRED` (decision `REVIEW`) |
| 9 | `record_risk_event` | `session_actions.py` → `risk_store.py` | Record confirmed input-policy blocks once, without duplicates; output and retrieval events go to the audit log only | detailCode `RECORDED`, `DUPLICATE_OR_BLOCKED`, `NON_POLICY_EVENT` |

Every action returns an `ActionDecision` (`decisions.py`): `ALLOW / BLOCK / TRANSFORM / REVIEW`, plus `reasonCode` and an optional `detailCode`. When an action cannot decide or a precondition fails, it raises `SecurityError` (for example `GUARDRAIL_UNAVAILABLE`, `REQUEST_CONTEXT_MISMATCH`), and the caller denies the request (fail closed).

Supporting files:

| File | Role |
| --- | --- |
| `context.py` | Immutable `RequestContext` and `Envelope` that bind text to one request |
| `nemo_adapter.py` | Runs the actions as NeMo Guardrails actions (`nemo` backend) or locally (`local` backend) with the same decision rules |
| `session_guard.py` | Connects session risk checks to the NeMo adapter |
| `policy_core.py`, `marketing_core.py` | Standard-library-only rule cores; each file can run on its own (checked by `test_policy_core_structure.py`) |

## Where they sit in this architecture

| Layer in the main [README](../README.md) | Actions |
| --- | --- |
| Request access control | 1 `validate_request_context` |
| Fixed Guard | 3 `check_input_policy` |
| Dynamic Metric Guard | 2 `check_session_risk`, 9 `record_risk_event` |
| Cost Control | 5 `check_generation_admission`, 6 `check_request_budget` |
| NeMo Output Guardrail | 7 `check_output_policy`, 8 `check_marketing_response` |
| Privacy Output Guard | 4 `mask_personal_data` |

These are the Python actions planned in [EXTENSION_DESIGN_NOTES.md](../docs/EXTENSION_DESIGN_NOTES.md) section 4. They are also the guard functions the red-team runner in [REDTEAM_REGRESSION_DESIGN.md](../docs/REDTEAM_REGRESSION_DESIGN.md) would call directly. The OpenClaw masking hooks in [OPENCLAW_STUDY_NOTES.md](../docs/OPENCLAW_STUDY_NOTES.md) would call action 4.

The source service runs them in this order: capacity → budget reservation → call → settlement → slot release, with cleanup on timeout, cancellation and failure.

## Run the tests

```bash
cd python-security-actions
uv run --group dev pytest -m "not nemo"        # local backend, no extra packages
# real NeMo rails (Python 3.12/3.13):
uv run --group dev --extra nemo pytest
```

| Included test | What it covers |
| --- | --- |
| `test_policy_actions.py` | Actions 3, 4, 7; registry; import isolation |
| `test_marketing_actions.py` | Action 8, including negation, quotation and mixed-sentence bypass cases |
| `test_l02_input_only_risk.py` | Actions 2 and 9 on input-only risk |
| `test_nemo_integration.py` | NeMo adapter consumes every decision in both rail directions |
| `test_policy_core_structure.py` | Rule cores import only the standard library and run as single files |

**Verified when copying:** `pytest -m "not nemo"` on Python 3.13 → **121 passed, 22 deselected** (the deselected ones are the real NeMo tests, not run here). The source repository reports its own wider results (214 local, 285 with real NeMo); those include tests that need the full FastAPI app and are not repeated here.

**Not included, because they need the full FastAPI app:** `test_session_security.py`, `test_security_pipeline.py`, `test_paid_call_service.py`, `test_usage_accounting.py` (session guard wiring, end-to-end pipeline, capacity/budget/settlement service, usage accounting). Actions 1, 5 and 6 and `session_guard.py` are therefore covered here only indirectly; their full tests stay in the source repository.

## Limits (carried over from the source)

- Rule-based checks. Masking does not guarantee complete anonymization, and the marketing check does not guarantee legal compliance of wording.
- Real paid-provider calls are not verified.
- The risk store and budget store are in-memory; restarting resets them.

## Open questions

- Which copy is canonical going forward: keep `legal-intake-fastapi-lab` canonical and add a byte-equality check (as its `intake_guard` package mirror already does), or move the canonical copy here.
- Whether to publish `policy_core` and `marketing_core` as an installable package from this repository, or keep using `intake_guard` from the source repository.

---

## 한국어 요약

- `legal-intake-fastapi-lab` 커밋 `4fc98d5`의 보안 action 9개(`app/security/` 14개 파일)와, 앱 전체 없이 실행되는 시험 5개를 그대로 복사했습니다. import 경로 `app.security`도 그대로 둬서 원본과 비교할 수 있습니다.
- 9개 action: 요청 문맥 확인, 세션 누적 위험, 입력 정책, 개인정보 마스킹, 동시 실행 제한, 예산 예약, 출력 정책, 마케팅 문구 검토, 위험 이벤트 기록. 결과는 `ALLOW/BLOCK/TRANSFORM/REVIEW`와 reasonCode·detailCode로 반환하고, 판단할 수 없거나 전제 조건이 틀리면(문맥 불일치, 용량 초과 등) `SecurityError`를 던져 거부합니다.
- 이 저장소의 계층(Fixed Guard, Dynamic Metric Guard, Cost Control, Privacy Output Guard, NeMo 출력)에 각각 대응시켰습니다. 레드팀 실행기와 OpenClaw 마스킹 훅이 호출할 대상입니다.
- 복사 후 확인: Python 3.13에서 `pytest -m "not nemo"` **121개 통과**. 실제 NeMo 시험 22개는 여기서 실행하지 않았습니다.
- FastAPI 앱 전체가 필요한 시험(세션 연결, 전체 파이프라인, 용량·예산·정산 서비스, 사용량 정산)은 원본 저장소에 남겨 두었습니다.
