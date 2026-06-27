# 01 · 아키텍처와 리포지토리 구조

> 확정 결정은 [[00-overview]] §2 참조. 본 문서는 시스템을 모듈 경계로 분해하고, 의존 방향과 요청 생명주기를 정의한다.

---

## 1. 제어 평면 / 실행 평면 분리

이 분리는 본 시스템의 최상위 불변식이다. (근거: OpenAI Sandbox Agents가 control plane / execution plane 분리를 권고 — beta)

```
┌──────────────────────── 제어 평면 (실행 권한 0) ────────────────────────┐
│  Prompt Intake → Orchestrator → Template Matcher → Test Strategy        │
│       → Tool Selection → (Terraform Planner*) → Safety/Policy Judge      │
│   산출물: UserIntent · TemplateMatch · TestPlanSpec · ToolAction(draft)  │
│           · ApprovalRequest  →  "Plan Review Bundle"                     │
└─────────────────────────────────┬───────────────────────────────────────┘
                                   │  plan_hash
                          ┌────────▼─────────┐
                          │  인간 승인 게이트  │  approve(approval_id + plan_hash)
                          └────────┬─────────┘
┌──────────────────────── 실행 평면 (유일 실행 권한) ─────────────────────┐
│  Execution Controller  →  Runner(k6/Artillery, 로컬)  →  목 서버         │
│     canary → (통과 시) full run → Observation Collector(read-only)       │
│  kill switch 소유 = Execution Controller/Runner                          │
└─────────────────────────────────┬───────────────────────────────────────┘
                                   ▼
        Observer → Interpreter → Report Composer → FinalReport + Audit Log
```
\* Terraform Planner는 로컬 범위에서 **인터페이스/계약만**(plan/show-json까지 설계), 실제 apply는 후속 ([[00-overview]] D11).

**불변식**: 제어 평면의 어떤 모듈도 실행 권한이 없다 → 위험 작업은 승인 게이트를 통과해야 한다 → 승인 후에도 Execution Controller만 allowlist된 ToolAction으로 실행 → 외부 문서는 절대 지시문으로 취급하지 않는다.

---

## 2. 컴포넌트 3중 매핑 (기획 ↔ 리서치 8-에이전트 ↔ FastAPI 모듈)

기획안의 3요소(Test Generator / Load Generator / Reporter)와 리서치의 8-에이전트, 그리고 실제 코드 모듈의 대응이다. **Template Matcher는 새 에이전트가 아니라 Orchestrator/Test Strategy 책임군의 retrieval 하위기능**이며 실행 권한은 없다.

| 기획 컴포넌트 | 리서치 에이전트(권한) | FastAPI 모듈 | 평면 |
|---|---|---|---|
| **Test Generator** | Orchestrator(조정), Retriever/Freshness(읽기), Test Strategy(계획), Template Matcher | `engine/orchestrator.py`, `engine/matcher.py`, `engine/strategy.py`, `engine/retriever.py` | 제어(read-only) |
| (계획 보강) | Tool Selection, Terraform/IaC Planner(plan까지) | `engine/tool_select.py`, `engine/iac_planner.py` | 제어 |
| (안전) | Safety/Policy Judge(판정) | `safety/judge.py`, `safety/clamp.py`, `safety/policy.py` | 제어 |
| **Load Generator** | **Execution Controller(유일 실행권)**, Runner | `runners/controller.py`, `runners/k6_adapter.py`, `runners/artillery_adapter.py` | 제어→**실행** |
| **Reporter** | Observer, Interpreter, Report Composer(읽기) | `reporter/collector.py`, `reporter/interpreter.py`, `reporter/composer.py` | 제어(읽기) |

상세 에이전트별 입력/출력/실패모드/검증은 `docs/research/03-ai-agent-orchestration.md` §4와 [[03-engine-template-matching]] / [[04-safety-approval-audit]] / [[05-runners-and-mock-target]] / [[06-reporter-observability]]에 분배한다.

---

## 3. 요청 생명주기 (해피 패스 + 분기)

```
1. POST /intake        자연어 프롬프트 → UserIntent (모호 필드는 추정 금지)
2. 의도 분류           load/stress/spike/soak/breakpoint/report-only
3. 템플릿 매칭          프롬프트 ⇄ ScenarioTemplate 의미 검색, top-k, match_confidence
   ├─ ≥ τ_high·단일우세 → 4
   ├─ τ_low ~ τ_high    → 명확화 질문(top-k 후보 제시) → 사람 선택 → 4
   └─ < τ_low           → 실행 거부(자유생성 폴백 금지; 전문가+승인 경로만)
4. 슬롯 채움            structured output(typed), 필수 슬롯 누락 → 질문
5. 안전 clamp/검증      safe_limits·target_whitelist·env allowlist 적용
6. 계획 번들 생성        UserIntent+TemplateMatch+TestPlanSpec+ToolAction(draft)+ApprovalRequest, plan_hash 계산
7. POST /approve       인간 승인(approval_id + plan_hash 결합) — 거부 시 사유 audit
8. 실행                 Execution Controller → canary → (통과) full run (로컬 목 서버)
9. 관측                 Observation Collector(read-only) → ObservationBundle
10. 해석·리포트         InterpretationSpec → FinalReport(경영/기술) + evidence_refs
모든 단계: AuditLog append-only 기록
```

위험 요청(prod/고부하/결제 등)은 3~5 어느 단계에서든 **기본 금지**로 단락(short-circuit)된다 ([[04-safety-approval-audit]] §결정 흐름).

---

## 4. 리포지토리 구조 (확정)

```
repo/  (docs/ 와 별개의 신규 앱 루트 — 위치는 M0에서 확정)
  backend/
    app/
      main.py            # FastAPI 앱 부트스트랩, 라우터 등록
      api/               # 라우트(얇게): intake, match, slots, approve, run, report, templates
        deps.py          # 의존성 주입(세션·현재 사용자·정책 로더)
      engine/            # 제어 평면: 의도→매칭→슬롯→계획 (실행 권한 없음)
        orchestrator.py  matcher.py  strategy.py  retriever.py
        tool_select.py   iac_planner.py
      llm/               # provider 추상화
        base.py          # LLMProvider 인터페이스(complete/structured/embed/classify)
        hosted.py        # 호스티드 어댑터(Claude/OpenAI 등)
        local.py         # 로컬 어댑터(Ollama 등)
        registry.py      # 설정 기반 어댑터 선택
      safety/            # 게이트·clamp·정책·인젝션 방어·audit
        judge.py  clamp.py  policy.py  audit.py  sanitize.py
      runners/           # 실행 평면: 유일 실행 권한
        controller.py    # Execution Controller(ToolAction 실행, kill switch)
        base.py          # RunnerAdapter 인터페이스
        k6_adapter.py  artillery_adapter.py
        compose.py       # Template Composer(슬롯→도구별 스크립트/구성)
      reporter/          # 관측·해석·리포트
        collector.py  interpreter.py  composer.py
      models/            # Pydantic(도메인) + SQLAlchemy(영속) + 스키마 변환
        schemas.py  db.py  store.py
      config.py          # 설정(env, 안전 상한 기본값, 임계값)
    tests/               # 단위/통합/계약/E2E (목 LLM·목 서버 활용)
    pyproject.toml
  frontend/              # React (Vite + TypeScript)
    src/
      pages/             # 프롬프트 → 슬롯/명확화 → 계획검토 → 승인 → 실행 → 리포트
      api/               # 백엔드 클라이언트(타입은 OpenAPI에서 생성)
      components/
    package.json
  templates/             # ScenarioTemplate 레지스트리(YAML, 버전관리 대상)
    event_open_readiness.yaml  common_journey_regression.yaml
    firebase_rest_crud_capacity.yaml  firebase_realtime_sync_probe.yaml
    baseline_compare_report.yaml
    _schema.md           # 템플릿 스키마 설명
  mock-target/           # 번들 로컬 목 서버(지연·에러율·동접 주입)
  eval/                  # 매칭 평가셋(발화 30+)·기대 라벨
  docs/                  # 운영/개발 README (리서치 docs와 구분)
```

### 의존 방향 규칙 (단방향)
```
api → engine → (llm, models, safety)        # api는 얇게, 로직은 engine
api → runners → (models, safety, llm)        # 실행은 controller 통해서만
reporter → (models)                          # 읽기 전용
engine ↛ runners                              # 제어 평면은 실행기를 직접 호출하지 않음
safety는 engine·runners 양쪽에서 호출되는 공통 게이트
```
- **engine은 runners를 import하지 않는다**(평면 분리). 실행은 항상 `ToolAction` → `runners/controller.py`로 넘어간다.
- `llm`은 어디서도 비결정적 부작용을 내장하지 않는다(테스트 시 목 주입).

---

## 5. 기술 선택 근거 (요약)

| 선택 | 근거 |
|---|---|
| FastAPI | typed(Pydantic) 모델이 스키마 12종과 1:1, 자동 OpenAPI로 프론트 타입 생성, async·WebSocket(실시간 진행률) 용이, 작은 의존성 |
| React/Vite SPA | 슬롯 채움·명확화·계획 검토·승인·리포트의 다단계 인터랙션에 적합, 제어/실행 분리를 화면 흐름으로 반영 |
| SQLite/SQLAlchemy | 로컬 우선·결정성, 클라우드 단계에서 엔진 교체만으로 Postgres 이전 |
| LLM provider 추상화 | 교체 잦음(D5). 인터페이스 뒤에서 호스티드/로컬을 스왑, 테스트는 목 어댑터 |
| k6 + Artillery 어댑터 | k6=thresholds로 SLO 코드화·관측 강점, Artillery=YAML "값만 채우는" 템플릿 친화. Composer가 추상화(D7) |
| 번들 목 서버 | 실대상 없이 결정적·재현 가능한 부하 실행/검증, 안전·비용0 (D8) |

> 설계 자체는 도구·클라우드 **벤더 비종속**을 유지한다. k6/Artillery는 어댑터 뒤에 있고, 클라우드 러너(Grafana Cloud k6/AWS DLT/Azure LT)는 동일 `RunnerAdapter` 인터페이스로 후속 확장한다.
