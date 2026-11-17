# OpenClaw Study Notes and Additional Controls for Legal and Medical AI

**Status: study notes and planned design, not implemented.** Statements about OpenClaw come from reading its documentation in [ai-agent-orchestrator/openclaw](https://github.com/ai-agent-orchestrator/openclaw) at commit `6747b1e` (2026-11-17), not from running it. OpenClaw changes quickly; re-check the linked pages before relying on a detail. This document extends [EXTENSION_DESIGN_NOTES.md](EXTENSION_DESIGN_NOTES.md), where OpenClaw is the first candidate agent runtime.

## 1. What to read, and in what order

The repository has more than 65,000 files. For this architecture (guard outside the runtime, connected through `before_agent_run` and `before_tool_call`), about eight documents are enough. Source code can wait until a specific question needs it.

| # | File in the OpenClaw repo | Why it matters here |
| --- | --- | --- |
| 1 | `docs/gateway/security/trust-model.md` | **Most important.** One gateway is one trust boundary; it is not a hostile multi-tenant boundary (section 2). |
| 2 | `docs/plugins/hooks/tool-policy.md` | Where the guard plugs in. `before_tool_call` can return `block`, rewritten `params` or `requireApproval`. Missing `ctx.requester` fields are "unproven" and policy should deny. Also documents `tool_result_persist` and `before_message_write`. |
| 3 | `docs/plugins/hooks/prompt-and-session.md` | `before_agent_run`, `before_prompt_build` and `ctx.inputProvenance` (host-classified origin of input). |
| 4 | `docs/gateway/security/prompt-injection.md` | Untrusted-content wrapping, bypass flags to keep off, and the **reader agent** pattern. |
| 5 | `docs/gateway/security/tool-permissions.md` | Per-agent access profiles, sub-agent delegation guardrail, network-read taint on a turn. |
| 6 | `docs/gateway/sandbox-vs-tool-policy-vs-elevated.md` | Sandbox (where tools run) vs tool policy (which tools exist) vs elevated (exec on host). Short, and the most common source of confusion. |
| 7 | `docs/gateway/security/hardened-baseline.md` | Copy/paste locked-down configuration. |
| 8 | `docs/gateway/security/secrets-and-storage.md` → "Logs and transcripts" | The biggest risk for legal and medical data (section 3.1). |

Optional next: `docs/security/THREAT-MODEL-ATLAS.md` sections 4 (ClawHub skill supply chain) and 5.2 (critical attack chains), `docs/gateway/security/running-the-audit.md` (`openclaw security audit`), and `docs/gateway/prometheus.md`.

## 2. Key finding: OpenClaw is not a tenant-isolation layer

The trust-model page states that one gateway serves one operator or a mutually trusting team, and that mutually untrusted users sharing one gateway or agent is not supported. The defaults follow that assumption:

| Default | Effect |
| --- | --- |
| `tools.sessions.visibility: "all"`, `tools.agentToAgent.enabled: true` | An unsandboxed tool-enabled agent can list, read and message every agent's sessions, including other users' transcripts. |
| Host exec `security="full"`, `ask="off"` for the trusted single operator | Commands run on the host without approval prompts. |
| `sessionKey` | A routing selector, not an authorization token. |
| Operator roles | Collaboration guardrails, not tenant isolation. |

**Design consequence:** clients and patients must not message OpenClaw directly. The Spring service stays the client-facing layer with JWT, tenant and case scope. OpenClaw runs as an internal operator-side runtime. If per-client isolation is ever needed inside OpenClaw, it means one gateway per tenant, ideally on separate OS users or hosts.

## 3. Additional controls for legal and medical AI

### 3.1 Personal information at rest (highest priority)

- Session transcripts are stored in a per-agent SQLite database, and tool arguments and tool results keep their original text.
- `logging.redactPatterns` masks logs and diagnostics only. It does not change transcripts or model context, so the existing Privacy Output Guard does not cover stored data.
- Planned controls:
  - Call the masking Python action from `before_message_write` and `tool_result_persist` (both synchronous hooks; handlers must not be `async`).
  - Turn off memory indexing for agents that handle case or patient data.
  - Apply a retention policy through session maintenance. Do not delete cold archive files separately from their database records.
  - Encrypt the disk holding `~/.openclaw` and keep its permissions owner-only.
- Health information is sensitive information under the Korean Personal Information Protection Act, and medical records may fall under separate rules. Retention and processing need legal review before real data is used.

### 3.2 A separate reader agent for uploaded documents

- Evidence files and medical-record PDFs are untrusted content, even when the client is trusted.
- A reader agent with no tools (or read-only tools and a read-only workspace) summarizes them.
- The main agent receives only that summary, through send-only delegation (`tools.agentToAgent.allow` plus a narrow `tools.sessions.visibility`). The summary is still data, not instructions.
- This turns the existing rule "no single agent combines private data, untrusted content and external sending" into concrete configuration.

### 3.3 Human approval for irreversible actions

- Sending a message to a client, submitting a filing, or sending any medical guidance goes through `before_tool_call` → `requireApproval` with `severity: "critical"` and `allowedDecisions: ["allow-once", "deny"]`.
- Per the hook documentation, the first `requireApproval` freezes the parameter snapshot, so later handlers can block but cannot change the approved parameters.
- Tools that would finalize a diagnosis, a prescription or a legal judgment are not on any allowlist.

### 3.4 Hardened baseline by default, audit in CI

Start every agent from the hardened baseline and open tools one agent at a time:

```json5
{
  gateway: { mode: "local", bind: "loopback", auth: { mode: "token", token: "<long random>" } },
  session: { dmScope: "per-channel-peer" },
  tools: {
    profile: "messaging",
    deny: ["group:automation", "group:runtime", "group:fs", "sessions_spawn", "sessions_send"],
    fs: { workspaceOnly: true },
    exec: { security: "deny", ask: "always" },
    elevated: { enabled: false },
    sessions: { visibility: "agent" },
    agentToAgent: { enabled: false },
  },
}
```

- If interpreters are ever allowlisted, enable `tools.exec.strictInlineEval`.
- If delegation is needed, restrict `subagents.allowAgents` and call `sessions_spawn` with `sandbox: "require"`.
- Add `openclaw security audit` to the CI gate described in [REDTEAM_REGRESSION_DESIGN.md](REDTEAM_REGRESSION_DESIGN.md).
- Treat skills and plugins as untrusted code: read, pin and approve each one (already a rule in the extension notes).

### 3.5 Map hook decisions to reasonCodes

| OpenClaw hook outcome | reasonCode |
| --- | --- |
| `before_tool_call` → `block` (tool not on allowlist) | `TOOL_NOT_ALLOWED` |
| `before_tool_call` → `block` (argument check failed) | `ARG_INVALID` |
| `before_tool_call` → `requireApproval` | `APPROVAL_REQUIRED` |
| Injection found in a tool result | `TOOL_RESULT_INJECTION` |
| Decision service unreachable | deny (fail closed), `GUARD_UNAVAILABLE` (new) |

The same reasonCode appears in the hook's `blockReason`, the audit log and the Prometheus label, so OpenClaw decisions show up next to the existing Spring guard metrics in Grafana.

### 3.6 OpenClaw cases for the red-team dataset

Add to the dataset planned in [REDTEAM_REGRESSION_DESIGN.md](REDTEAM_REGRESSION_DESIGN.md):

- Instructions hidden inside an uploaded evidence file or medical record
- Injection through a tool result (web fetch, document parse)
- An agent trying to list or read another agent's sessions
- Sub-agent delegation beyond the allowed depth or to an unlisted agent
- Tool calls with a missing or unproven requester

## 4. Open questions

- Whether the reader agent and the main agent run on the same gateway (simpler) or separate gateways (stronger isolation).
- How approval requests reach a lawyer or clinician, and how long they stay valid.
- Whether Codex-native tool records, which the persistence hooks do not rewrite, will be used at all; if so, masking needs another path.

---

## 한국어 핵심 요약

- **읽을 순서:** trust-model → hooks/tool-policy → hooks/prompt-and-session → prompt-injection → tool-permissions → sandbox-vs-tool-policy-vs-elevated → hardened-baseline → secrets-and-storage의 "Logs and transcripts". 소스 코드는 필요할 때 봅니다. (기준: openclaw 커밋 `6747b1e`, 문서만 읽었고 실행은 안 해 봤습니다.)
- **핵심 사실:** OpenClaw는 게이트웨이 하나를 신뢰 경계 하나로 보며, 서로 신뢰하지 않는 사용자를 한 게이트웨이에서 격리하지 않습니다. 기본값에서는 다른 에이전트의 대화 기록을 읽을 수 있고, 호스트 명령도 승인 없이 실행됩니다. 따라서 의뢰인·환자는 Spring 서버만 상대하고, OpenClaw는 내부 운영용으로만 씁니다.
- **법률·의료 추가 통제:**
  1. 저장 데이터 개인정보: 대화 기록과 도구 결과가 원문으로 저장되므로 `before_message_write`·`tool_result_persist` 훅에서 마스킹하고, 메모리 인덱싱을 끄고, 보관 기간을 정하고, 디스크를 암호화합니다. 실제 데이터 사용 전에 법률 검토가 필요합니다.
  2. 업로드 문서는 도구 권한이 없는 읽기 전용 에이전트가 요약하고, 주 에이전트에는 요약만 send-only로 넘깁니다.
  3. 의뢰인 발송, 서면 제출, 의료 안내 발송은 `requireApproval`로 사람이 승인합니다. 진단·처방·법률 판단을 확정하는 도구는 허용하지 않습니다.
  4. 강화 기본 설정에서 시작하고 `openclaw security audit`를 CI에 넣습니다.
  5. 훅 판단 결과를 기존 reasonCode와 Grafana에 연결합니다.
  6. 레드팀 데이터셋에 OpenClaw 전용 케이스(문서 속 숨은 지시, 도구 결과 인젝션, 다른 세션 조회, 위임 초과)를 추가합니다.
