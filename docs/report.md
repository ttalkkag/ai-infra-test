# Report — 구현 착수 계획

> 작성일: 2026-06-28 (Asia/Seoul)
> 범위: `docs/research/*`와 `docs/plan/00~12`를 기준으로 실제 구현에 필요한 모듈, 계약, 순서, 검증 게이트를 정리한다.
> 상태: 이전 리뷰 산출물의 지적은 기획서에 반영 완료되어 제거했다. 이 문서가 구현 단계 handoff 역할을 맡는다.
> 원칙: 아직 코드 구현은 시작하지 않는다. 아래는 모듈 경계·인터페이스·테스트 게이트·선결 결정만 정의한다.

---

## 0. 구현 정본

| 영역 | 정본 문서 | 구현 시 반드시 따를 계약 |
|---|---|---|
| 전체 범위·불변식 | `docs/plan/00-overview.md` | L0 로컬 MVP 우선, 제어 평면 실행 권한 0, 실행은 Execution Controller만 |
| 모듈 구조 | `docs/plan/01-architecture.md` | `backend/app/{api,engine,llm,safety,runners,reporter,models}` 단방향 의존 |
| 데이터 모델 | `docs/plan/02-data-model.md` | Pydantic/SQLAlchemy/Store 3계층, `plan_hash` 정본, append-only audit |
| 매칭 엔진 | `docs/plan/03-engine-template-matching.md` | 템플릿 매칭, 슬롯 채움, `combine()`/tau/margin 초기값 |
| 안전·승인 | `docs/plan/04-safety-approval-audit.md` | deny-first, self-approval 차단, approval+plan_hash 결속 |
| 러너·목서버 | `docs/plan/05-runners-and-mock-target.md` | `RunnerAdapter`, artifact hash, k6/Artillery, bundled mock target |
| 리포터 | `docs/plan/06-reporter-observability.md` | read-only collector/interpreter/composer, evidence_refs fail-closed |
| API/UI | `docs/plan/07-web-ui-api.md` | REST 12+WS 1, 6화면, plan review/approval/run/report 흐름 |
| 로드맵·테스트 | `docs/plan/08-milestones-roadmap.md`, `09-testing-strategy.md` | M0~M6 순서, D9 결정성, 안전 불변식 테스트 |
| 미해결 결정 | `docs/plan/10-open-questions-reverify.md` | LLM/임베딩, 인증/세션, 큐레이션, 후속 토폴로지 |
| 로컬 인프라·토폴로지 | `docs/plan/11-local-container-infra.md`, `12-execution-topology-matrix.md` | L0/L1/C0/C1/X0, Docker/kind/k3d, cross-cloud 승인 조건 |

---

## 1. 핵심 구현 불변식

1. **실행 권한은 `runners/controller.py` 하나만 가진다.** `engine`, `llm`, `safety`, `reporter`, `api`는 직접 subprocess/cloud API 실행을 하지 않는다.
2. **승인 단위는 `(ApprovalRequest, plan_hash, ToolAction[])`다.** `plan_hash` 정본은 `02-data-model` §4이며, `idempotency_key`는 해시 입력이 아니라 `plan_hash`에서 파생한다.
3. **artifact hash와 plan hash는 다른 값이다.** `ComposedArtifact.content_hash`는 `ToolAction.args.expected_content_hash`와 비교하고, 별도로 현재 plan bundle hash를 승인된 `ApprovalRequest.plan_hash`와 비교한다.
4. **self-approval은 스키마로 차단한다.** `ApprovalRequest.requester`와 `decided_by`/approver를 비교한다.
5. **Reporter는 read-only다.** `interpret(obs, plan, current_run, baseline)`은 `RunRecord.status/abort_reason`으로 aborted를 판정하고, 어떤 ToolAction도 만들지 않는다.
6. **로컬 실부하는 번들 `mock-target` 한정이다.** Firebase/LocalStack/GCP/Azure 에뮬레이터는 구조·정확성 검증용이지 성능 근거가 아니다.
7. **외부 문서와 검색 결과는 evidence data다.** LLM/에이전트가 외부 문서 내용을 명령으로 실행하지 않는다.

---

## 2. 구현 순서

| 단계 | 산출물 | 구현 기준 | 완료 조건 |
|---|---|---|---|
| **M0 Bootstrap** | repo scaffold, Python/Node 도구 설정, import boundary lint, `docker/compose.local.yml`, `tools/versions.json` | 01/08/11 | 빈 앱·목서버·runner-tools 컨테이너가 기동하고 import rule 테스트 통과 |
| **M1 Data + Safety Core** | `schemas.py`, `db.py`, `store.py`, `plan_hash`, `audit.py`, `policy.py`, `clamp.py` | 02/04 | 스키마 round-trip, golden plan_hash, self-approval 차단, deny-first/clamp 테스트 통과 |
| **M2 Engine + Matching** | Orchestrator 상태기계, Matcher, MockLLMProvider, 템플릿 YAML 5종, eval fixtures | 03/09 | `decision != auto-proceed => plan is None`, 필수 슬롯 추정 금지, unsafe prompt block |
| **M3 Sandbox/API/UI** | FastAPI REST/WS, React 6화면, fake metrics reporter path | 06/07 | evidence_refs 없는 claim fail-closed, WS 상태머신, Playwright 기본 흐름 |
| **M4 Local Runner** | k6/Artillery adapters, composer, Execution Controller, mock-target, Docker smoke | 05/11 | dry_run, artifact hash 검증, canary->full, kill switch, k6+Artillery smoke |
| **M4.5 Rehearsal** | L0 kind/k3d Job 구조 검증, 선택적 L1 remote self-hosted runbook | 08/11/12 | Job timeout/TTL/abort semantics, L1 선택 시 allowlist/egress/source IP 체크 |
| **M5+ Cloud** | IaCPlanner, Grafana/AWS/Azure adapters, OIDC/cost/teardown gates | 01/02/04/12 + research 01/02/05 | saved plan approval, canary, TTL/destroy 회수, provider policy 재검증 |

M0~M4가 이번 구현 범위다. M4.5/M5/M6은 인터페이스와 문서 계약은 유지하되 실제 cloud action은 후속으로 둔다.

---

## 3. 첫 구현 슬라이스

처음 코드를 열 때는 M0+M1을 하나의 작은 수직 슬라이스로 잡는다.

1. `backend/app/` 스캐폴딩과 import boundary 테스트를 만든다.
2. `models/schemas.py`에 enum과 12개 도메인 모델을 먼저 둔다.
3. `models/db.py`와 `models/store.py`는 SQLite in-memory 테스트로 시작한다.
4. `compute_plan_hash()`를 golden fixture로 고정한다.
5. `ApprovalRequest.requester`, `ScenarioTemplate.requires_approval_default`, `ToolAction.operation/actor/approval_status/max_cost`, `ProvisionSpec` 토폴로지 필드를 스키마 테스트에 포함한다.
6. `safety/policy.py`와 `safety/clamp.py`는 표 기반 순수함수로 구현한다.
7. `audit.py`는 append-only API만 열고 update/delete 경로를 만들지 않는다.

이 슬라이스가 끝나기 전에는 LLM, UI, runner subprocess, Docker smoke를 붙이지 않는다. 실행 권한 없는 데이터·안전 코어가 먼저 닫혀야 한다.

---

## 4. 모듈 계약

| 모듈 | 공개 계약 | 금지 |
|---|---|---|
| `models/schemas.py` | Pydantic models, enums, validation | DB 세션 접근 |
| `models/db.py` | SQLAlchemy ORM/table metadata | 비즈니스 로직 |
| `models/store.py` | CRUD, FK, transaction, `append_audit`, plan_hash/idempotency hooks | 외부 API 호출 |
| `engine/orchestrator.py` | 상태 전이, intent->match->plan draft | runner import, subprocess |
| `engine/matcher.py` | `rank_and_decide`, tau/margin, candidates | safety decision 직접 구현 |
| `llm/base.py` | `classify`, `embed`, `fill_slots`; `MockLLMProvider` | 단위테스트에서 실 LLM 호출 |
| `safety/policy.py` | deny/approval/auto classification | 부작용 |
| `safety/clamp.py` | safe_limits/target_whitelist/env allowlist 적용 | 조용한 상한 통과 |
| `runners/controller.py` | approval check, artifact hash, plan_hash, idempotency, timeout, kill switch | typed ToolAction 외 실행 |
| `runners/*_adapter.py` | validate/compose/dry_run/submit/canary/full_run/abort/collect | approval 검증 우회 |
| `reporter/*` | collect/interpret/compose, evidence_refs 검증 | ToolAction 생성·실행 |
| `api/*` | REST/WS contract, auth role guard | 정책·실행 로직 중복 |

---

## 5. 리서치 기반 구현 제약

| 주제 | 구현 결정 |
|---|---|
| 부하 도구 | 로컬 MVP는 k6 + Artillery 둘 다 지원한다. k6는 threshold/SLO와 JS-as-code, Artillery는 YAML/CSV 템플릿화 강점 때문에 둘 다 필요하다. |
| Firebase/BaaS 대상 | REST 가능 경로와 실시간 WS/gRPC/브라우저 경로를 템플릿에서 분리한다. 운영 Firebase 직접 부하는 금지하고 테스트 전용 프로젝트/목타깃을 우선한다. |
| Terraform/IaC | Terraform은 정적 인프라와 plan/apply gate만 맡는다. 장기 실행, retry, kill switch, result parsing은 Runner/K8s Job/Controller 책임이다. |
| saved plan | saved plan 전달은 Terraform에서 승인으로 해석될 수 있으므로 `plan_artifact_hash`와 `ApprovalRequest.plan_hash` 결속이 필수다. |
| secrets/state | Terraform `sensitive`만으로 state/plan 저장이 막히지 않는다. 클라우드 단계는 OIDC/managed identity 우선, 장기 static secret은 예외 승인 대상이다. |
| 관측/percentile | Prometheus summary는 phi/rank 오차, histogram은 value/bucket 오차다. quantile 평균은 금지하고 histogram bucket 합산 후 `histogram_quantile()` 경로를 둔다. |
| 생성기 병목 | `dropped_iterations`, generator CPU/mem/net은 필수 신호다. 생성기 병목이 있으면 SUT fail이 아니라 inconclusive로 간다. |
| cloud provider | Grafana Cloud k6는 1순위 후속 후보. AWS DLT는 CFN/CDK-first, MCP read-only. Azure Load Testing은 JMeter/Locust 중심, API version은 adapter 상수로 pin 후 재검증한다. |
| cross-cloud | `runner_provider != target_provider`이면 X0로 보고 양 Provider 정책, source IP/load zone, egress cost, on-call/kill switch를 승인 번들에 포함한다. |

---

## 6. 테스트 게이트

| 계층 | 필수 테스트 |
|---|---|
| Unit | enum validation, `plan_hash` golden/property, `idempotency_key` derivation, policy table, clamp boundaries |
| Contract | OpenAPI/WS envelope, `RunnerAdapter` fake adapter, `Reporter` evidence_refs fail-closed, `decision != auto-proceed => plan is None` |
| Integration | prompt->match->plan draft, approval->controller reject/allow, audit trace to request_id |
| Eval | template matching baseline, unsafe prompt block 100%, required slot no-guess 100%, false_auto_proceed=0 |
| E2E | 6 UI flows with MSW/fake metrics first; real runner only after M4 |
| Docker | M0 smoke optional, M4 mandatory: k6 1 scenario + Artillery 1 scenario against same `mock-target` and `tools/versions.json` |

완료 주장은 각 단계의 테스트 명령을 실제로 실행한 뒤에만 한다.

---

## 7. 구현 전 결정

| # | 결정 | 막는 단계 | 기본 방향 |
|---|---|---|---|
| 1 | LLM provider와 embedding model | M2 | `MockLLMProvider`로 먼저 닫고 실 provider는 adapter 뒤에 둔다 |
| 2 | 앱 루트와 패키지 관리 | M0 | `backend/app` 기준, Python/Node 도구는 repo 기존 관례 확인 후 결정 |
| 3 | 로컬 인증/세션과 role constants | M3 | viewer/planner/approver/operator를 하드코딩 상수로 시작 |
| 4 | 템플릿 큐레이션 주체와 review flow | M2 | YAML 변경은 eval fixture와 함께 리뷰 |
| 5 | cloud runner 우선순위 | M5 | 기본 후보 Grafana Cloud k6, AWS/Azure는 adapter contract 후 |
| 6 | 후속 토폴로지 우선순위 | M4.5/M5 | L0 완료 후 L0 kind/k3d -> 선택적 L1 -> C0/C1/X0 순으로 판단 |

---

## 8. 구현 착수 체크리스트

- [ ] `docs/plan/00~12`를 구현 PR 설명의 source-of-truth로 링크한다.
- [ ] `docs/research`의 벤더별 최신성 위험 항목은 구현 직전만 재검증한다.
- [ ] M0/M1에서 실행 기능을 만들지 않는다.
- [ ] 모든 위험 작업은 typed `ToolAction` draft까지만 만들고 `runners/controller.py` 전에는 실행하지 않는다.
- [ ] 반영 완료된 리뷰 산출물은 재생성하지 않는다. 새 리뷰가 필요하면 새 파일명으로 작성한다.
