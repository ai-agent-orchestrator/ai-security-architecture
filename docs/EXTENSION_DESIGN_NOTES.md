# Extension Design Notes: Python Actions, an Agent Runtime and Multi-Agent Operation

**Status: planned design, not implemented.** The layers described in the main [README](../README.md) (JWT, Fixed Guard, Dynamic Metric Guard, Cost Control, Privacy Output Guard, NeMo Colang fixed flows, reasonCode, Prometheus/Grafana) exist. Everything in this document extends them. No part of the extension has been integrated or tested yet, and statements about third-party tools come from their public documentation, not from running them.

## 1. Goal

Run several agents (personally, and later as a commercial service) behind the same control layers that already protect a single LLM call, without trusting the agent runtime to police itself.

Three additions:

1. A **decision contract**: one narrow JSON interface that any agent runtime calls before a model call or tool call.
2. **Python actions**: small, testable checks (masking, tool-argument validation, budget lookup) attached to NeMo and to the decision service.
3. An **agent runtime** (OpenClaw as the first candidate) connected through its plugin hooks, kept replaceable.

## 2. Design rules

| Rule | Why |
| --- | --- |
| The guard runs **outside** the agent runtime. | An agent that can edit its own guard has no guard. |
| The decision contract is **runtime-agnostic**. | The runtime can be replaced (NemoClaw/OpenShell, another harness) without touching the guard. |
| **Fail closed.** If the decision service is unreachable, the tool call is denied. | A missing check must never mean "allow". |
| **Deny by default.** Each agent has an explicit tool allowlist. | New tools stay off until reviewed. |
| Anything an agent reads (web pages, files, tool output, other agents' messages) is **data, not instructions**. | Injection arrives through content, not only through the user prompt. |
| Irreversible actions need **human approval**. | Delete, send and pay are never automatic. |
| Every decision carries a **reasonCode** and is logged. | Explainable control is the core value of this repository. |

## 3. Decision contract (draft)

Request (from the runtime plugin to the decision service):

```json
{
  "tenantId": "string",
  "agentId": "string",
  "sessionId": "string",
  "stage": "agent_input | tool_call | tool_result | agent_output",
  "tool": "string or null",
  "argsSummary": "bounded, masked summary of the arguments",
  "estimatedCost": { "tokens": 0, "currency": "optional" }
}
```

Response:

```json
{
  "decision": "allow | deny | ask",
  "reasonCode": "string",
  "redactions": ["field names changed or removed"],
  "budget": { "remaining": 0 }
}
```

New reasonCodes to add next to the existing ones: `TOOL_NOT_ALLOWED`, `ARG_INVALID`, `APPROVAL_REQUIRED`, `LOOP_DETECTED`, `DELEGATION_DEPTH_EXCEEDED`, `AGENT_BUDGET_EXCEEDED`, `TOOL_RESULT_INJECTION`, `KILL_SWITCH_ACTIVE`.

## 4. Python actions

| Action | Job | Notes |
| --- | --- | --- |
| Masking | Mask personal information in text before any model or tool sees it, and in output before it leaves | Reuses the Privacy Output Guard rules; mapping tables stay server-side |
| Tool-argument validation | Check types, lengths, allowed paths and hosts for each tool | Allowlist per tool, not a generic filter |
| Budget lookup | Read the tenant, agent and run budget from the cost ledger | Reserve before the call, settle after (same model as Cost Control) |
| Tool-result inspection | Run the Fixed Guard injection rules on tool output, not only on user input | Closes the gap between user-input checks and agent-read content |

Python actions live behind the decision service (Spring RiskEngine calls them over the narrow JSON contract) and, where useful, inside NeMo flows. Each action returns allow, deny or ask plus a reasonCode, and each has its own tests.

## 5. Agent runtime integration (OpenClaw as first candidate)

The OpenClaw plugin documentation describes typed plugin hooks, including `before_agent_run` (gate model input and return a block with a reason) and `before_tool_call` (block, rewrite parameters or require approval). The plan is a thin plugin that calls the decision contract at those two points and returns the decision unchanged.

Operating rules for the runtime:

- Run it in a container or separate environment, with test data first.
- Do not expose its gateway to the internet; keep credentials out of its configuration files.
- Treat skills and plugins as untrusted code: read, pin and approve each one; no automatic installation.
- Keep the runtime replaceable. Nothing in the guard depends on it.

## 6. Multi-agent operation

| Control | Description |
| --- | --- |
| Per-agent identity | JWT or service-token scopes: tenant, agent, allowed tools, budget |
| Per-agent and per-run budgets | Cost ledger entries per agent and per run; hard caps stop work instead of retrying |
| Loop and delegation limits | Dynamic Guard extended to repeated tool calls, cost growth rate and delegation depth |
| Kill switch | One switch that denies every decision for an agent, a tenant or everything |
| Capability separation | No single agent combines private data access, untrusted content and external sending |

## 7. Where to go deepest

### Personal operation of a multi-agent setup

| Priority | Area | Direction |
| --- | --- | --- |
| 1 | Least privilege | Injection cannot be fully blocked, so limit what a compromised agent can do (capability separation) rather than only improving detection |
| 2 | Evaluation and regression tests | Automated attack and quality scenarios so models, prompts and runtimes can change safely |
| 3 | Operating safeguards | Per-agent budgets, loop detection, kill switch, human approval for irreversible actions |
| 4 | Memory hygiene | Validate before storing; keep source links; support deletion; never let stored text act as instructions |
| 5 | Tracing and audit | Per-run record of what was read, what was called and why it was blocked |

### Spring and Python hybrid structure in general

| Area | Direction |
| --- | --- |
| Authorization propagation | Spring validates tenant and case scope; Python receives only a narrow internal token and never widens it |
| Contracts and idempotency | Versioned JSON contracts; idempotency keys so retries never run or bill twice |
| Cancel-safe billing | Reserve before the call; release on failure or cancellation; unresolved usage is never assumed free |
| Data lifecycle | Where masking happens, where mappings live, PII in logs, encryption, retention and deletion |
| Reliability | Job state machine, bounded retries, circuit breaker, load measurement |
| Deployment and secrets | Key rotation, backup and restore, environment separation |

## 8. Commercialization checklist (design only)

Tenant isolation and per-tenant budgets with usage metering, SSO/OIDC, key management and rotation, retention and deletion policy, tamper-evident audit log, penetration testing, review against applicable privacy law, and a documented incident-response path.

## 9. Verification plan

1. **Shadow mode first.** Log decisions without enforcing them, then tune thresholds.
2. **Proof of concept.** One harmless tool, one plugin hook, fake data, separate environment.
3. **Regression suite.** Extend the Postman scenarios to automated tests: injection through tool results, repeated calls, budget overrun, approval bypass attempts.
4. **Status wording.** A control is described as working only after a test or measurement shows it; until then it is labeled planned or design.

## 10. Open questions

- Which actions run inside NeMo and which only in the decision service.
- How approval requests are delivered to a person (web, phone or tablet) and how long they stay valid.
- Whether the runtime plugin interface is stable enough to depend on; version pinning and an upgrade review step are needed either way.

---

## 한국어 핵심 요약

- 가드는 **에이전트 런타임 밖**에 두고, 어떤 런타임이든 부를 수 있는 좁은 JSON **판단 계약**(allow / deny / ask + reasonCode)으로 연결합니다. 장애 시에는 거부합니다.
- **Python action**: 마스킹, 도구 인자 검증, 예산 조회, 도구 결과 인젝션 검사. 각각 테스트를 둡니다.
- **OpenClaw**는 첫 후보 런타임이며 플러그인 훅(`before_agent_run`, `before_tool_call`)으로 연결하고 언제든 교체할 수 있게 둡니다. 스킬은 읽고 고정한 것만 씁니다.
- 멀티 에이전트: 에이전트별 신원·예산, 루프·위임 깊이 제한, 즉시 중단 스위치, 한 에이전트가 사적 데이터·외부 콘텐츠·외부 전송을 모두 갖지 않게 분리합니다.
- 가장 깊게 팔 곳: 권한 최소화 → 평가·회귀 테스트 → 운영 안전장치 → 메모리 위생 → 추적·감사. 하이브리드에서는 권한 전파와 데이터 수명 주기부터.
- 검증 순서: 섀도 모드 → 작은 시험판(가짜 데이터, 분리 환경) → 회귀 테스트. 시험으로 확인되기 전에는 동작한다고 쓰지 않습니다.
