# 07 · 웹 UI와 REST/WS API 계약

> 확정 스택: 백엔드 **FastAPI**, 프론트 **React(Vite + TypeScript) SPA**. 타입은 백엔드 OpenAPI에서 생성([[01-architecture]] §5, [[00-overview]] D3·D4).
> 본 문서는 **계획**이다 — 화면 흐름·API 계약·상태 설계까지만 정의하고 구현 코드는 작성하지 않는다.
> 최상위 불변식: **제어 평면(읽기·계획, 실행 권한 0) / 실행 평면(유일 실행 권한) 분리**를 화면 흐름과 버튼 활성 조건으로 그대로 반영한다([[01-architecture]] §1).

---

## 1. 단계 ↔ 평면 ↔ 라우트 매핑

화면 6단계는 [[01-architecture]] §3 요청 생명주기를 1:1로 옮긴 것이다. ①~④는 **제어 평면(실행 권한 없음)**, ⑤만 **실행 평면**, ⑥은 **읽기 전용**이다.

| # | 화면(page) | 라우트 | 주 엔드포인트 | 평면 | 실행 권한 |
|---|---|---|---|---|---|
| ① | 프롬프트 입력 | `/` | `POST /intake` | 제어 | 없음 |
| ② | 명확화·슬롯 채움 | `/clarify/:intentId` | `POST /match`, `POST /slots` | 제어 | 없음 |
| ③ | 계획 검토(읽기) | `/plan/:planId` | `POST /plan`, `GET /plan/:planId` | 제어 | 없음 |
| ④ | 승인 | `/plan/:planId/approve` | `POST /approve` | 제어 | 없음(승인은 권한 부여일 뿐 실행 아님) |
| ⑤ | 실행 진행 | `/run/:runId` | `POST /run`, `WS /run/:runId/stream` | **실행** | **있음(승인 바인딩 후)** |
| ⑥ | 리포트 | `/report/:runId` | `GET /report/:runId`, `GET /runs` | 읽기 | 없음 |
| — | 템플릿 카탈로그 | `/templates` | `GET /templates` | 읽기 | 없음 |
| — | 실행 이력·baseline | `/runs` | `GET /runs?baseline=` | 읽기 | 없음 |

**불변식의 UI 표현**: ①~④ 어느 화면에도 "실행" 버튼이 없다. 실행 버튼은 ⑤ 화면에만 존재하며, **유효한 `approval_id + plan_hash` 바인딩이 없으면 비활성**이다(§5).

---

## 2. REST/WS API 계약 표

스키마명은 [[02-data-model]]의 도메인 모델(`UserIntent`/`TemplateMatch`/`TestPlanSpec`/`ToolAction`/`ApprovalRequest`/`RunRecord`/`FinalReport`/`ScenarioTemplate`)을 그대로 사용한다. 요청/응답은 이들을 감싸는 얇은 wrapper 스키마(`*Request`/`*Response`)로 노출되어 OpenAPI 타입 생성에 그대로 쓰인다.

### 2.1 REST

| Method · Path | 요청 스키마 | 응답 스키마 | 권한 | 평면 | 멱등/비고 |
|---|---|---|---|---|---|
| `POST /intake` | `IntakeRequest{ prompt, locale?, session_id? }` | `IntakeResponse{ intent_id, UserIntent, safety_verdict }` | planner | 제어 | 모호 필드 추정 금지. `safety_verdict ∈ {allow, deny}`(=`IntakeSafetyVerdict`, [[02-data-model]] §6) — [[04-safety-approval-audit]] §2.1 3분류의 intake 시점 사영: DENY 술어 매치 → `deny` 즉시 단락, 그 외 `allow`(승인 필요 여부는 plan 단계 `classify`가 판정) |
| `POST /match` | `MatchRequest{ intent_id, k?=3 }` | `MatchResponse{ decision, candidates: TemplateMatch[], classification }` | planner | 제어 | `decision ∈ {auto-proceed, clarify, reject, route-to-expert}`(=`MatchDecision`, [[02-data-model]] §6) (§6.2) |
| `POST /slots` | `SlotsRequest{ intent_id, template_choice?, slot_values?{} }` | `SlotsResponse{ status, filled_slots{}, missing_slots[], clamp_notes[], validation_errors[] }` | planner | 제어 | `status ∈ {needs_input, ready}`. 명확화 답변(`template_choice`)·슬롯 채움 모두 이 엔드포인트 |
| `POST /plan` | `PlanRequest{ intent_id, template_id, slot_values{} }` | `PlanResponse` = **Plan Review Bundle**`{ plan_id, plan_hash, UserIntent, TemplateMatch, TestPlanSpec, ToolAction[], safety_judgement, blast_radius, cost_estimate, ApprovalRequest(pending) }` | planner | 제어 | `plan_hash = sha256(canonical(bundle))`. 동일 입력 → 동일 `plan_hash`(결정성) |
| `GET /plan/{plan_id}` | — | `PlanResponse` | viewer | 제어(읽기) | 검토 화면 재진입·새로고침용. 읽기 전용 |
| `POST /approve` | `ApproveRequest{ plan_id, plan_hash, decision, reason? }` | `ApproveResponse{ approval_id, status, expires_at }` | approver | 제어 | `decision ∈ {approve, reject}`. `plan_hash` 불일치 시 409(§6.1). 거부 사유 audit |
| `POST /run` | `RunRequest{ approval_id, plan_hash }` | `RunResponse{ run_id, RunRecord(state=Queued) }` | operator | **실행** | **유일 실행 트리거**. 승인된 `ToolAction[]`만 실행. `Idempotency-Key` 헤더 필수(중복 실행 차단) |
| `GET /runs/{run_id}` | — | `RunRecord` | viewer | 읽기 | WS 단절 시 폴링 폴백·현재 상태 조회 |
| `WS /run/{run_id}/stream` | (§2.2) | (§2.2) | operator(kill)/viewer(구독) | **실행** | 실시간 진행률·canary 판정·kill. `?from_seq=N` 재연결 |
| `GET /report/{run_id}` | — | `ReportResponse{ data_source, report: FinalReport{ executive_summary, technical_summary, slo_results[], risk_assessment, recommendations[], artifacts[], limitations[], template_id, match_confidence } }` | viewer | 읽기 | 실행 미완료 시 404/425(too early). `data_source`는 `ObservationBundle.source`에서 온다([[02-data-model]]·[[06-reporter-observability]]) |
| `GET /templates` | `?q=&intent=` | `TemplateListResponse{ items: ScenarioTemplate[] }` | viewer | 읽기 | 카탈로그. safe_limits·target_whitelist 노출(읽기) |
| `GET /runs` | `?baseline=<run_id>&limit=&intent=` | `RunListResponse{ items: RunRecord[], baseline?: RunRecord, delta?{} }` | viewer | 읽기 | `baseline` 지정 시 p95/error_rate 등 delta 동봉([[06-reporter-observability]]) |

> 권한 모델: 로컬 단일 사용자 MVP에서는 4개 역할(`viewer`/`planner`/`approver`/`operator`)이 한 사용자에 합쳐질 수 있으나, **계약은 분리를 보존**한다(클라우드/사내 단계에서 분리 적용, [[00-overview]] D2·D11). 실행 평면(`/run`, WS kill)은 항상 `operator` 권한을 별도로 확인한다.
>
> 인증 경계(로컬 MVP): 별도 인증 백엔드가 없는 동안 app은 **`127.0.0.1` 전용 바인딩을 강제**하고(Docker 모드 포함 — 포트 게시를 loopback으로 제한, [[05-runners-and-mock-target]] §7.2), 이를 명시적 신뢰 경계로 둔다. `requester`/`approver` 신원은 로컬 세션 상수에서 온다(`docs/report.md` §7-3). **인증되지 않은 자기신고 신원 위에서는 self-approval 차단([[04-safety-approval-audit]] §4.4)이 실수 방지 수준**임을 한계로 명시한다 — 네트워크 노출·멀티유저 사용은 토큰/세션 인증 도입([[10-open-questions-reverify]] §C-4) 전까지 금지.

### 2.2 WebSocket `/run/{run_id}/stream`

연결 즉시 서버가 현재 상태 스냅샷(`state` 이벤트)을 보내고, 이후 증분 이벤트를 push 한다. 모든 이벤트는 `{ seq, ts, type, payload }` 봉투를 가지며 `seq`는 단조 증가(재연결·중복 제거 기준).

**서버 → 클라이언트 이벤트**

| `type` | payload | 의미 |
|---|---|---|
| `state` | `{ state, since, reason? }` | 상태머신 전이(§6.4). 연결 직후 1회 + 전이 시마다 |
| `progress` | `{ phase, percent, elapsed_s, vus, rps }` | 진행률(phase ∈ canary/full) |
| `metric` | `{ p50, p95, p99, error_rate, throughput }` | 표본 메트릭(라이브 그래프용, read-only) |
| `canary` | `{ verdict, checked_thresholds[], evidence_ref }` | canary 게이트 판정(`pass`면 full 진행, `fail`이면 abort) |
| `log` | `{ level, message }` | 사람 친화 로그(원시 도구 stdout 아님) |
| `error` | `{ code, message, retriable }` | 실행 오류 |
| `done` | `{ final_state, report_ready }` | 종료. `report_ready=true`면 ⑥로 유도 |

**클라이언트 → 서버 명령**

| `type` | payload | 권한 | 효과 |
|---|---|---|---|
| `kill` | `{ reason }` | operator | kill switch 트리거 → `Aborting`. 소유권은 Execution Controller/Runner([[01-architecture]] §1) |
| `ping` | `{}` | any | heartbeat(서버 `pong`) |

**재연결 계약**: 클라이언트는 마지막 수신 `seq`를 보관하고 `?from_seq=N`으로 재연결한다. 서버는 버퍼된 이벤트를 replay 한다(버퍼 윈도 초과 시 `state` 스냅샷으로 리셋). kill은 서버가 **멱등 처리**(이미 Aborting/종료면 무시) — 단절 중 중복 전송이 안전하다.

### 2.3 공통 오류 봉투

모든 4xx/5xx는 단일 형태로 응답해 계약 테스트(§9.3)에서 검증한다.

```
ApiError { code: string, message: string, detail?: object, retriable: bool }
```

| code | HTTP | 트리거 | UI 처리 |
|---|---|---|---|
| `PLAN_HASH_MISMATCH` | 409 | 승인/실행 시 `plan_hash` 불일치(번들 변조·재생성) | ③로 되돌리고 재검토 요구 |
| `APPROVAL_EXPIRED` | 409 | `expires_at` 경과 후 `/run` | ④ 재승인 유도(§8) |
| `APPROVAL_REQUIRED` | 403 | 승인 없이 `/run` 시도 | 실행 버튼이 애초에 비활성이어야 함(방어선) |
| `SAFETY_BLOCKED` | 422 | 위험 요청 차단(prod/고부하/결제 등) | 차단 사유 카드 표시, 진행 경로 없음 |
| `SLOT_VALIDATION` | 422 | 슬롯 타입/범위 위반 | ② 인라인 검증 메시지 |
| `RUN_CONFLICT` | 409 | 동일 `Idempotency-Key` 재실행 | 기존 `run_id`로 안내 |
| `REPORT_NOT_READY` | 425 | 실행 전 리포트 요청 | ⑤로 유도 |

---

## 3. 화면 흐름 (pages) — ASCII 와이어프레임

### ① 프롬프트 입력 (`/`)

```
┌────────────────────────────────────────────────────────────┐
│  부하 테스트 — 무엇을 확인하고 싶으세요?                       │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐│
│  │ 예: "신규 가입 이벤트 오픈 때 서버가 버티는지 보고 싶어요"  ││
│  │                                                          ││
│  └────────────────────────────────────────────────────────┘│
│  [ 분석 시작 ]                                                │
│                                                              │
│  ─ 예시 프롬프트 ─────────────────────────────────────────── │
│  · "평소 트래픽의 5배가 몰리면 응답이 느려지나요?"             │
│  · "회원가입→로그인 흐름을 30분간 꾸준히 돌려보고 싶어요"      │
│                                                              │
│  ⓘ 이 도구는 사전 검증된 시나리오 안에서만 동작합니다.         │
│    실서비스(prod)·결제 같은 위험 대상은 자동 차단됩니다.       │
└────────────────────────────────────────────────────────────┘
```
`POST /intake` → `intent_id` 확보. `safety_verdict="deny"`(정책 차단)면 ②로 가지 않고 차단 카드(§3 하단 공통) 표시.

### ② 명확화·슬롯 채움 (`/clarify/:intentId`)

분기 A — **top-k 후보 선택(저신뢰, τ_low~τ_high)**. `POST /match → decision=clarify`.

```
┌────────────────────────────────────────────────────────────┐
│  몇 가지만 확인할게요. 어떤 상황에 가장 가까운가요?            │
│                                                              │
│  ◉ 이벤트 오픈 순간 폭주 대비   (스파이크)   유사도 0.78  📦템플릿│
│  ○ 평소의 N배 지속 부하        (부하)       유사도 0.61      │
│  ○ 장시간 안정성(누수) 확인     (soak)       유사도 0.55      │
│                                                              │
│  [ 이걸로 진행 ]   [ 다시 입력 ]                              │
│  ⓘ 자동 선택이 아니라 사람이 고르는 단계입니다(저신뢰 매칭).   │
└────────────────────────────────────────────────────────────┘
```

분기 B — **필수 슬롯 채움**. `POST /slots → status=needs_input, missing_slots[]`.

```
┌────────────────────────────────────────────────────────────┐
│  선택: [스파이크 - 이벤트 오픈 대비]                          │
│                                                              │
│  대상 (target)        [ 번들 목 서버 (mock-target) ▼ ]  필수 │
│      └ 실서비스 주소는 이 단계에서 선택할 수 없습니다.        │
│  예상 동접 (peak VUs)  [   500   ]  필수                      │
│      └ 안전 상한 2,000으로 clamp됨 (템플릿 내장 한계)         │
│  지속 시간 (duration)  [   3분   ]  필수                      │
│  목표 응답(p95)        [  800ms  ]  선택                      │
│                                                              │
│  ⚠ 입력하신 5,000은 템플릿 안전 한계 2,000으로 조정됩니다.    │
│  [ 계획 만들기 → ]   (필수 슬롯이 모두 채워져야 활성)         │
└────────────────────────────────────────────────────────────┘
```
필수 슬롯이 다 차면 `status=ready` → `POST /plan` → ③. `clamp_notes`는 인라인 경고로 노출(우회 불가).

### ③ 계획 검토 — 읽기 전용 (`/plan/:planId`)

```
┌────────────────────────────────────────────────────────────┐
│  계획 검토   plan_hash: 3f9a…c21   [읽기 전용]               │
│ ───────────────────────────────────────────────────────────│
│  TestPlanSpec                                                │
│   유형: 스파이크 | 대상: mock-target | VUs 500 | 3분         │
│   상승 곡선: 0→500 (10s) 유지 → 감소                          │
│   실행 토폴로지: L0 local-workstation → local-container → local│
│   network_path: same-host                                    │
│                                                              │
│  ToolAction (실행될 작업)            도구: k6                 │
│   ▸ dry-run 정적 검증  →  승인 후 canary(5% · 30s) → full run │
│   ※ 자유 명령이 아닌 정해진 작업만 실행됩니다(typed).         │
│                                                              │
│  안전 판정 (safety_judgement)                                │
│   ✔ 대상 화이트리스트 통과   ✔ RPS 상한 내   ✔ 결제/외부 없음 │
│  blast_radius:  로컬 목 서버 1대 (외부 영향 없음)            │
│  cost_estimate: 0원 (로컬 실행)                              │
│ ───────────────────────────────────────────────────────────│
│  [ 승인 단계로 → ]      [ 수정(②로) ]                        │
│  ⛔ 이 화면에는 실행 버튼이 없습니다.                         │
└────────────────────────────────────────────────────────────┘
```
모든 필드는 `GET /plan/{plan_id}` 응답을 그대로 렌더. **편집 불가**(편집하려면 ②로 돌아가 `plan_hash` 재계산). 계획 검토 화면은 `ProvisionSpec.control_plane_location`/`runner_location`/`runner_provider`/`target_provider`/`network_path`를 표시해 L0/L1/C0/C1/X0 토폴로지를 사람이 확인할 수 있게 한다([[12-execution-topology-matrix]]).

### ④ 승인 (`/plan/:planId/approve`)

```
┌────────────────────────────────────────────────────────────┐
│  승인                       바인딩 plan_hash: 3f9a…c21       │
│                                                              │
│  위 계획(plan_hash 3f9a…c21)을 실행하도록 승인합니다.        │
│  · 승인은 "실행 권한 부여"이며, 승인 즉시 실행되지 않습니다. │
│  · 승인 유효기간: 15분 (이후 재승인 필요)                    │
│                                                              │
│  [ 승인 ]            [ 거부 ]                                 │
│                                                              │
│  거부 사유(거부 시 필수): [______________________________]   │
└────────────────────────────────────────────────────────────┘
```
`POST /approve{ plan_id, plan_hash, decision }` → `approval_id` + `expires_at`. 승인 성공 시 ⑤로 이동(아직 실행 전). 거부는 사유와 함께 audit, 흐름 종료.

### ⑤ 실행 진행 — 실행 평면 (`/run/:runId`)

```
┌────────────────────────────────────────────────────────────┐
│  실행            run_id: r-204   상태: ● Running             │
│  데이터 출처: 🟢 실측 (mock-target 대상 k6 실제 실행)        │
│ ───────────────────────────────────────────────────────────│
│  Queued ─▶ Canary ✔ ─▶ Running ━━━━━━━━░░░░ 64%             │
│                                                              │
│  canary(30s): 통과 ✔  (오류율 0.2% < 1% 기준)               │
│  VUs 500 | RPS 1,180 | p95 612ms | 오류율 0.3%               │
│                                                              │
│  [라이브 그래프: p95 / error_rate]                            │
│   ░▁▂▃▄▅▆▅▄▃  (read-only 관측)                              │
│ ───────────────────────────────────────────────────────────│
│  [ 즉시 중단 (Kill) ]   ← operator 전용, 누르면 Aborting     │
└────────────────────────────────────────────────────────────┘
```
진입 직전 `POST /run{ approval_id, plan_hash }`(+`Idempotency-Key`)로 트리거, 즉시 `WS /run/:runId/stream` 구독. canary 실패 시 자동 `Failed`로 단락(full run 미진입, [[00-overview]] §6).

### ⑥ 리포트 (`/report/:runId`)

```
┌────────────────────────────────────────────────────────────┐
│  리포트  run_id: r-204   [경영 요약]  [기술 상세]            │
│  데이터 출처: 🟢 실측  |  template_id: spike_event_open      │
│                            match_confidence: 0.78           │
│ ── 경영 요약 ───────────────────────────────────────────────│
│  결론: 동접 500까지는 목표(p95<800ms) 충족, 그 이상은 미검증 │
│  · p95 612ms (목표 800ms 이내)                               │
│  · 오류 예산 소진 12% (기준 내)                              │
│  한계(limitations):                                          │
│   - 대상은 번들 목 서버이며 실서비스가 아님                  │
│   - 500 VUs 단일 시나리오, 복합 트래픽 미반영                │
│  근거(evidence_refs): [k6 summary ▸] [canary 판정 ▸] [샘플▸]│
│                                                              │
│  [ baseline과 비교 ]   [ 내보내기 (JSON / PDF) ]            │
└────────────────────────────────────────────────────────────┘
```
`GET /report/{run_id}`. 모든 주장에 `evidence_refs` 링크. `data_source`(=`ObservationBundle.source`, [[06-reporter-observability]])는 M3/M4에서 **`frozen(고정)`** 또는 **`measured(실측)`** 배지로 항상 표시(§7). 후속 cloud는 `measured-remote`.

### 공통 — 차단/거부 카드 (위험·저신뢰)

```
┌────────────────────────────────────────────────────────────┐
│  ⛔ 이 요청은 진행할 수 없습니다                              │
│  이유: 실서비스(prod) 주소를 대상으로 지정했습니다.          │
│  안전 정책상 이 도구는 prod 직접 부하를 허용하지 않습니다.   │
│  → 대신 할 수 있는 것: 목 서버 대상으로 동일 시나리오 실행   │
│  [ 목 서버로 바꿔 진행 ]   [ 처음으로 ]                      │
└────────────────────────────────────────────────────────────┘
```

---

## 4. 제어/실행 평면의 UI 반영 (실행 버튼 활성 규칙)

| 화면 | 사용자 액션 | 평면 | 실행 버튼 존재 | 활성 조건 |
|---|---|---|---|---|
| ①~③ | 입력·매칭·슬롯·검토 | 제어 | **없음** | — |
| ④ 승인 | approve/reject | 제어 | 없음(승인 ≠ 실행) | `plan_hash` 바인딩 일치 시 승인 가능 |
| ⑤ 실행 | Run 트리거 | **실행** | **있음** | `approval.status=approved` AND `expires_at` 미경과 AND `plan_hash` 일치 |
| ⑤ 실행 | Kill | 실행 | 있음 | 상태 ∈ {Queued, Canary, Running} AND `operator` 권한 |

규칙(프론트 가드 + 백엔드 이중 방어):
- **승인 전 실행 버튼 비활성**이며, ⑤ 화면 진입 자체가 유효 `approval_id` 없이는 불가(라우트 가드).
- 프론트의 비활성은 UX일 뿐, 백엔드가 `APPROVAL_REQUIRED`/`PLAN_HASH_MISMATCH`로 **최종 강제**([[04-safety-approval-audit]]).
- ToolAction은 ③에서 **읽기만** 가능하고 편집 불가. 편집은 ②로 회귀 → `plan_hash` 재계산 → 기존 승인 무효화.

---

## 5. 상태 관리

### 5.1 프론트 상태 구획 (3계층)

| 계층 | 책임 | 도구(계획) |
|---|---|---|
| 서버 상태 | REST 응답 캐시·무효화 | TanStack Query (엔드포인트별 query key) |
| 마법사(wizard) 상태 | ①~⑥ 단계 전이·가드 | 명시적 유한 상태기(reducer 기반), URL = source of truth |
| 실시간 실행 상태 | WS 이벤트 → 실행 상태머신 | 전용 WS 스토어(§5.4) |

`intent_id → plan_id → plan_hash → approval_id → run_id` 식별자 사슬을 URL에 싣고, 새로고침/딥링크에도 `GET`으로 복원 가능하게 한다(③·⑤·⑥은 서버 재조회로 무상태 복원).

### 5.2 plan_hash ↔ 승인 바인딩

```
③ POST /plan        → { plan_id, plan_hash=H1, ApprovalRequest(pending) }
④ POST /approve     → 요청 본문에 (plan_id, plan_hash=H1) 동봉
                       서버: 저장된 번들 해시 == H1 ?  → approval_id 발급(상태 approved, expires_at)
                                                    아니면 → 409 PLAN_HASH_MISMATCH
⑤ POST /run         → (approval_id, plan_hash=H1)
                       서버: approval.plan_hash == H1 == 현재 번들 해시 ?
                             AND not expired AND status=approved → run_id (Queued)
                             아니면 → 409 (MISMATCH / EXPIRED) 또는 403 (REQUIRED)
```
- 승인은 **특정 `plan_hash`에 고정**된다. 계획이 한 글자라도 바뀌면 `plan_hash`가 달라져 이전 승인은 자동 무효.
- 프론트는 ⑤ 진입 시 `plan_hash`를 URL/스토어로 끌고 와 `POST /run` 본문에 그대로 싣는다 — 사용자가 임의로 바꿀 표면이 없다.

### 5.3 매칭 분기 표시 (저신뢰 거부 / 명확화 / 위험 차단)

`POST /match` 응답의 `decision`에 따라 ② 화면이 분기한다.

| `decision` (`MatchDecision`, [[02-data-model]] §6) | 의미 | UI |
|---|---|---|
| `auto-proceed` | ≥ τ_high 단일 우세 | 슬롯 채움(분기 B)으로 바로 진입 |
| `clarify` | τ_low ~ τ_high | top-k 후보 선택(분기 A) → 사람이 선택 |
| `reject` | < τ_low | 거부 카드: "확신할 수 있는 시나리오를 찾지 못했습니다." **자유 생성 폴백 없음**, 다시 입력/전문가 경로 안내 |
| `route-to-expert` | 전문가(`requester_expertise=="expert"`)의 저신뢰 | 전문가 자유생성+승인 경로 안내([[03-engine-template-matching]]) |

> **매칭 `decision`과 안전 차단은 별개 축이다.** 위험 게이트(prod/고부하/결제/외부)는 `decision`이 아니라 `/intake`의 `safety_verdict="deny"`(정책 차단, [[04-safety-approval-audit]] §2)로 단락된다. `safety_verdict="deny"`(차단)와 `decision ∈ {reject, route-to-expert}`(거부/전문가) 모두 **실행 경로로 새지 않는 것**이 성공 기준이다([[00-overview]] §7-1,2). 프론트는 이 분기들에서 ③로 가는 어떤 버튼도 렌더하지 않는다.

### 5.4 실행 WS 상태머신

```
        POST /run
           │
           ▼
       ┌────────┐  start    ┌────────┐  pass    ┌─────────┐  finish   ┌───────────┐
       │ Queued │ ───────▶ │ Canary │ ──────▶ │ Running │ ───────▶ │ Completed │ (terminal)
       └────────┘           └────────┘          └─────────┘           └───────────┘
           │ kill            │      │ fail        │ error    │ kill / ceiling
           │            kill │      ▼             ▼          │
           │                 │   ┌─────────────────────┐     │
           │                 │   │       Failed        │     │
           │                 │   │     (terminal)      │     │
           │                 │   └─────────────────────┘     │
           ▼                 ▼                               ▼
       ┌─────────────────────────────────────────────────────────┐
       │                        Aborting                         │
       └────────────────────────────┬────────────────────────────┘
                                    │ drained
                                    ▼
                               ┌─────────┐
                               │ Aborted │ (terminal)
                               └─────────┘
```

| 상태 | 진입 트리거 | UI | 허용 명령 |
|---|---|---|---|
| `Queued` | `POST /run` 성공 | "대기 중" 스피너 | kill |
| `Canary` | 컨트롤러 canary 시작 | canary 진행률·임계값 | kill |
| `Running` | `canary.verdict=pass` | 라이브 그래프·진행 바 | kill |
| `Completed` | full run 정상 종료 | "완료 → 리포트 보기" | — |
| `Aborting` | kill 수신 / ceiling 초과 | "중단 중…"(드레이닝) | — |
| `Aborted` | Aborting 드레인 완료 | "사용자 중단됨" + 부분 리포트 | — |
| `Failed` | canary fail / 실행 오류 | 실패 사유 + 부분 근거 | — |

- 위 상태명은 **표시용(대문자)** 이며, 영속 `RunStatus` 값은 소문자 `queued/canary/running/aborting/aborted/completed/failed`다([[02-data-model]] §6). WS `state` 이벤트는 이 값을 그대로 싣는다.
- 상태는 **서버가 유일 권위**다. WS `state` 이벤트로만 전이하고, 프론트는 미러링만 한다(낙관적 전이 금지).
- `canary.verdict=fail` → 서버가 `Failed`로 단락(full run 미진입). 프론트는 이를 그대로 표시.
- `done` 이벤트의 `report_ready`로 ⑥ 이동 버튼을 켠다.

---

## 6. 비전문가 UX 원칙

1. **부하/인프라 용어 최소화·번역**: 화면 주 라벨은 평이어("예상 동접", "지속 시간", "즉시 중단"), 정밀 용어(VUs·RPS·p95)는 보조 표기로만. 템플릿 카탈로그는 의도("이벤트 오픈 대비")로 검색하게 한다.
2. **안전 한계를 우회 불가·설명 가능하게**: clamp가 일어나면 "5,000 → 2,000으로 조정됨(템플릿 안전 한계)"처럼 **무엇이·왜** 바뀌었는지 인라인 표기. 입력칸에서 상한을 넘길 수 없다([[04-safety-approval-audit]]).
3. **왜 거부/차단됐는지 설명**: `reject`(저신뢰)/`deny`(안전 차단)는 코드가 아니라 평이한 사유 + **할 수 있는 대안**을 함께 제시(§3 차단 카드).
4. **"가짜/실측" 라벨 명확화**: 모든 리포트·실행 화면에 `data_source` 배지. Template/Sandbox 산출은 **`frozen(고정)`**(회색, 실행 0의 고정 fixture), Local Runner 실측은 **`measured(실측)`**(녹색, 목 서버 대상 실제 실행)으로 시각 구분([[00-overview]] D6·D8, [[06-reporter-observability]]). 목 서버 대상이어도 실제 실행 결과는 `measured`다(혼동 주의). 두 라벨을 섞어 보여주지 않는다.
5. **승인 = 실행 아님**을 문구로 못박기: ④에서 "승인은 권한 부여이며 즉시 실행되지 않습니다"를 명시, 실행은 ⑤의 별도 행위.
6. **읽기 전용 강조**: ③에 "[읽기 전용]"·"이 화면에는 실행 버튼이 없습니다" 배지로 제어/실행 분리를 사용자에게 가시화.

---

## 7. 에러·엣지 케이스

| 케이스 | 트리거 | 동작 | 회복 경로 |
|---|---|---|---|
| 슬롯 누락 반복 | `POST /slots` → `needs_input` 재발 | 남은 필수 슬롯만 강조, 진행 버튼 비활성 유지 | 모두 채우면 `ready` → ③ |
| 슬롯 검증 실패 | 타입/범위 위반(`SLOT_VALIDATION`) | 해당 입력칸 인라인 에러, clamp 가능 시 자동 제안 | 수정 후 재제출 |
| 승인 만료 | `expires_at` 경과 후 `/run`(`APPROVAL_EXPIRED`) | ⑤ 실행 버튼 비활성 + "승인이 만료됨" 배너 | ④ 재승인(동일 `plan_hash`면 재발급) |
| plan_hash 불일치 | 번들 재생성/변조(`PLAN_HASH_MISMATCH`) | 실행 차단 | ③로 회귀해 재검토·재승인 |
| 실행 중 kill | ⑤ Kill 버튼 / ceiling 초과 | `Aborting`→`Aborted`, 부분 리포트 보존 | ⑥에서 부분 근거 확인, 필요 시 재계획 |
| 네트워크 단절 | WS 끊김 | "재연결 중" 토스트, `?from_seq=N` 자동 재연결(지수 백오프) | 실패 지속 시 `GET /runs/{id}` 폴링 폴백 |
| WS 버퍼 초과 | 장시간 단절 | 서버가 `state` 스냅샷으로 리셋 | 스냅샷 기준 재동기화 |
| 중복 실행 | `Idempotency-Key` 충돌(`RUN_CONFLICT`) | 새 run 생성 안 함 | 기존 `run_id`로 이동 |
| 리포트 조기 요청 | 실행 전 `GET /report`(`REPORT_NOT_READY`) | ⑤로 유도 | 완료 후 재요청 |

---

## 8. 테스트 가능성 ([[09-testing-strategy]] 연동)

프론트는 **목 백엔드로 결정적 테스트**한다(LLM·러너 비결정성을 프론트 경계 밖으로 격리).

### 8.1 컴포넌트 테스트 (목 API)
- Vitest + React Testing Library, **MSW(Mock Service Worker)** 로 REST/WS 응답 고정.
- 고정 fixture: 각 `decision`(`auto-proceed`/`clarify`/`reject`/`route-to-expert`) + `safety_verdict`(`allow`/`deny`), `needs_input`↔`ready`, `PLAN_HASH_MISMATCH`/`APPROVAL_EXPIRED` 등 §7 케이스별 시나리오.
- 검증 대상: **실행 버튼 활성 규칙**(§4), clamp 인라인 표기, 차단/거부 카드, `data_source` 배지 분기.

### 8.2 E2E (Playwright)
- ①→⑥ 전체 흐름을 목 백엔드(시드된 FastAPI 테스트 서버 + 목 LLM·목 러너, [[01-architecture]] §4)로 결정적 재현.
- 핵심 시나리오: (a) 정상 happy path, (b) 명확화 후 진행, (c) 위험 차단으로 단락, (d) 승인 만료 → 재승인, (e) 실행 중 kill, (f) WS 단절 → 재연결.
- WS 상태머신(§5.4) 전이를 이벤트 주입으로 검증.

### 8.3 계약 테스트 (OpenAPI 일치)
- 백엔드 OpenAPI에서 프론트 타입 생성(빌드 단계) — **드리프트 시 타입 컴파일 실패**로 조기 검출.
- 백엔드: schemathesis 등으로 스키마·상태코드 적합성 검사.
- 프론트: 모든 MSW fixture를 OpenAPI 스키마로 검증해 목과 실계약 불일치 차단.
- WS는 별도 메시지 스키마(JSON Schema)로 이벤트 봉투·`type`별 payload 계약 고정.

---

## 9. 형제 문서 연결

- 스키마 정의: [[02-data-model]] (UserIntent/TemplateMatch/TestPlanSpec/ToolAction/ApprovalRequest/RunRecord/FinalReport/ScenarioTemplate)
- 평면 분리·요청 생명주기·리포 구조: [[01-architecture]]
- 매칭 분기·임계값(τ): [[03-engine-template-matching]]
- 승인·plan_hash·kill switch·clamp: [[04-safety-approval-audit]]
- 러너·canary·목 서버: [[05-runners-and-mock-target]]
- 리포트·evidence·baseline·data_source: [[06-reporter-observability]]
- 테스트 철학·계층: [[09-testing-strategy]]
- 남은 결정·재검증: [[10-open-questions-reverify]]
