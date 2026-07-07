# 02 · 데이터 모델 (Pydantic 도메인 ↔ SQLAlchemy 영속 ↔ OpenAPI)

> 확정 결정은 [[00-overview]] §2, 모듈 경계는 [[01-architecture]] §4 참조. 스키마 정본은 `docs/research/03-ai-agent-orchestration.md` §7(기본 10종)·§9.5(템플릿 매칭 2종). 본 문서는 **계획**이며 의사코드·스키마 초안·인터페이스 스케치까지만 다룬다(완성 코드 금지).
> 로컬 영속화는 **SQLite + SQLAlchemy** ([[00-overview]] D10), 도메인/직렬화는 **Pydantic**. 모듈은 `models/schemas.py`·`models/db.py`·`models/store.py`.

---

## 1. 스키마 12종 매핑 표

`docs/research/03` §7·§9.5의 12 스키마(기본 10종 + 템플릿 매칭 2종)를 영속 대상/transient로 분류하고 1차 키·외래 연결을 정한다. 중첩 객체(`workload_goal`, `slo_targets`, `safe_limits`, `param_slots` 등)는 **JSON 컬럼**으로 저장하고 Pydantic이 read/write 경계에서 검증한다.

| # | 스키마 | 영속/transient | 1차 키(PK) | 외래 연결(FK) | 비고 |
|---|---|---|---|---|---|
| 1 | **UserIntent** | 영속 | `request_id` | `template_match_ref → TemplateMatch.match_id`(역참조, nullable) | 추적 체인의 루트. `raw_prompt`는 감사 원본 |
| 2 | **ScenarioTemplate** | 영속(읽기전용 미러), 원본=`templates/*.yaml` | `template_id` | — | 원본은 버전관리 YAML([[01-architecture]] §4). 부팅 시 DB로 동기화, FK 무결성·감사 스냅샷용 |
| 3 | **TemplateMatch** | 영속 | `match_id` | `intent_ref → UserIntent.request_id`, `matched_template_id → ScenarioTemplate.template_id`(nullable) | `candidates`/`param_slots_filled`/`clarification_questions`는 JSON |
| 4 | **KnowledgePack** | 영속(경량, 참조 시 고정) | `pack_id` | `plan_ref → TestPlanSpec.plan_id`(nullable) | freshness/근거 감사용. `sources`·`tool_versions` JSON |
| 5 | **TestPlanSpec** | 영속 | `plan_id` | `match_ref → TemplateMatch.match_id`✚, `template_id → ScenarioTemplate.template_id`, `baseline_ref → RunRecord.run_id`(nullable) | `match_ref`는 체인 보강 컬럼(§2·assumptions) |
| 6 | **ProvisionSpec** | 영속(로컬은 거의 미사용, `topology=local`) | `provision_id` | — | 클라우드 단계 대비 인터페이스 보존([[00-overview]] D11) |
| 7 | **ToolAction** | 영속(번들 생성 시 고정; 그 전 draft는 in-memory) | `action_id` | `plan_ref → TestPlanSpec.plan_id`✚, `approval_id → ApprovalRequest.approval_id`(nullable), `template_id → ScenarioTemplate.template_id` | `args` JSON. `idempotency_key`는 §8 정책 |
| 8 | **ApprovalRequest** | 영속(현재 상태 row + 감사 결속) | `approval_id` | `plan_ref → TestPlanSpec.plan_id`✚ | **`plan_hash` 컬럼 보유**✚(§4). `requested_actions`=ToolAction 1:N |
| 9 | **RunRecord** | 영속 | `run_id` | `plan_id → TestPlanSpec.plan_id`, `approval_ref → ApprovalRequest.approval_id`✚, `template_id`, `provision_id`(nullable) | `artifacts` JSON(참조만) |
| 10 | **ObservationBundle** | 영속 | `observation_id` | `run_id → RunRecord.run_id` | `source` + `collected_as_of` + `metrics`/`logs`/`traces`/`costs` JSON 또는 artifact 참조 |
| 11 | **InterpretationSpec** | 영속 | `interpretation_id` | `run_id → RunRecord.run_id`, `observation_ref → ObservationBundle.observation_id`✚ | `findings`/`evidence_refs` JSON |
| 12 | **FinalReport** | 영속 | `report_id` | `interpretation_ref → InterpretationSpec.interpretation_id`✚, `run_ref → RunRecord.run_id`✚, `template_id` | 모든 주장에 `evidence_refs`([[00-overview]] §7-5) |
| + | **AuditLog** | 영속, **append-only** (12 스키마 외) | `audit_id`(+ `seq`) | 전 스테이지 약결합 참조 | §5 |

✚ = `docs/research/03` §7·§9.5 스키마에 없으나 추적 체인·승인 단위 구현을 위해 영속 계층에서 추가하는 링크/필드. siblings 정합 필요(문서 끝 assumptions).

**transient(미영속) 취급**: ToolAction의 승인 전 draft, 명확화 루프 중간 후보 상태, LLM 임베딩/분류 중간값. 이들은 번들 확정(생명주기 6단계, [[01-architecture]] §3) 시점에 영속 엔티티로 고정된다.

---

## 2. 엔티티 관계 — 추적 체인과 ER 다이어그램

요청 1건은 다음 단방향 추적 체인으로 이어지며, 각 단계는 직전 단계의 PK를 FK로 보유한다.

```
request_id → match_id → plan_id → approval_id → run_id → observation_id → interpretation_id → report_id
(UserIntent)(Template (TestPlan (Approval  (Run     (Observation  (Interpretation (FinalReport)
            Match)    Spec)     Request)  Record)  Bundle)       Spec)
```

ASCII ER (1:N은 `──<`, 1:1/0..1은 `──`):

```
                         templates/*.yaml
                                │ (부팅 동기화)
                                ▼
                       ┌─────────────────┐
                       │ ScenarioTemplate│ PK template_id
                       └───┬────┬────┬───┘
            matched_template│    │    │ template_id (run/report 스냅샷)
                            │    │    └────────────────────────┐
  ┌────────────┐  intent   ▼    │                              │
  │ UserIntent │──<──┌──────────────┐  match_ref  ┌──────────────┐
  │ request_id │     │ TemplateMatch│──────────<──│ TestPlanSpec │
  └────────────┘     │   match_id   │             │   plan_id    │
        ▲            └──────────────┘             └──┬────────┬──┘
        │ template_match_ref(역참조)                 │        │ plan_ref
        └────────────────────────────────────────────┘        ▼
                                              ┌─────────────────────┐
                            plan_ref          │     ToolAction      │
              ┌──────────────────────────────<│     action_id       │
              │                               └──────────┬──────────┘
              │                                approval_id│ (N:1)
        ┌─────▼──────────┐  plan_ref(0..1) ┌──────────────▼──────────┐
        │ KnowledgePack  │                 │    ApprovalRequest      │
        │    pack_id     │                 │  approval_id, plan_hash │
        └────────────────┘                 └──────────┬──────────────┘
                                            approval_ref│
                       ┌──────────────┐  provision_id   ▼
                       │ ProvisionSpec│──────────<┌──────────────┐ baseline_ref
                       │ provision_id │           │  RunRecord   │<───────(self)
                       └──────────────┘           │    run_id    │
                                                  └──────┬───────┘
                                              run_id     │ (1:N)
                                          ┌──────────────▼──────────┐
                                          │   ObservationBundle     │
                                          │     observation_id      │
                                          └──────────┬──────────────┘
                                      observation_ref │
                                          ┌───────────▼─────────────┐
                                          │   InterpretationSpec    │
                                          │   interpretation_id     │
                                          └───────────┬─────────────┘
                                  interpretation_ref   │
                                          ┌────────────▼────────────┐
                                          │      FinalReport        │
                                          │       report_id         │
                                          └─────────────────────────┘

  AuditLog(append-only): 위 모든 PK/plan_hash/approval_id를 약결합 참조로 보존 (§5)
```

체인의 모든 간선은 **FK 제약 + 인덱스**로 강제되어, `report_id` 하나에서 `request_id`까지 역추적이 단일 조인 경로로 가능하다.

---

## 3. SQLite 테이블 스케치

SQLite + SQLAlchemy. 규약: PK/FK·식별자=`TEXT`, datetime=ISO8601 `TEXT`, 불리언=`INTEGER(0/1)`, 수치=`INTEGER`/`REAL`, 중첩객체/배열=`TEXT`(JSON1, `json_valid` CHECK). enum은 앱(Pydantic)에서 강제하되 핵심은 `CHECK(col IN (...))` 보강. `PRAGMA foreign_keys=ON` 필수.

```sql
-- 의사 DDL (스케치). 실제 컬럼은 schemas.py와 1:1로 생성

user_intent(
  request_id TEXT PRIMARY KEY,
  raw_prompt TEXT NOT NULL,
  input_mode TEXT NOT NULL CHECK(input_mode IN ('template','generative','report-only')),
  requester_expertise TEXT NOT NULL DEFAULT 'nonexpert'
    CHECK(requester_expertise IN ('nonexpert','expert')),
  template_match_ref TEXT REFERENCES template_match(match_id),
  target_service TEXT NOT NULL,
  target_env TEXT NOT NULL CHECK(target_env IN
    ('local','dev','staging','pre-prod','prod-like','prod')),
  test_type TEXT NOT NULL CHECK(test_type IN
    ('load','stress','spike','soak','breakpoint','report-only')),
  endpoints TEXT NOT NULL,            -- JSON[]
  workload_goal TEXT NOT NULL,        -- JSON
  slo_targets TEXT NOT NULL,          -- JSON
  safety_limits TEXT NOT NULL,        -- JSON
  destructive_allowed INTEGER NOT NULL DEFAULT 0,
  needs_clarification INTEGER NOT NULL DEFAULT 0,
  created_at TEXT NOT NULL
);

scenario_template(                    -- YAML 미러(읽기전용)
  template_id TEXT PRIMARY KEY,
  name TEXT NOT NULL, description TEXT NOT NULL,
  example_utterances TEXT NOT NULL,   -- JSON[]
  test_type TEXT NOT NULL CHECK(test_type IN
    ('load','stress','spike','soak','breakpoint','report-only')),
  tool_family TEXT NOT NULL, target_pattern TEXT NOT NULL,
  param_slots TEXT NOT NULL,          -- JSON[]
  safe_limits TEXT NOT NULL,          -- JSON
  target_whitelist TEXT, default_slo TEXT,  -- JSON
  requires_approval_default TEXT NOT NULL CHECK(requires_approval_default IN
    ('always','by-env','never')),
  curated_by TEXT NOT NULL, reviewed_at TEXT NOT NULL,
  status TEXT NOT NULL CHECK(status IN ('draft','reviewed','active','deprecated')),
  source_path TEXT, source_sha TEXT   -- YAML 출처/체크섬
);

template_match(
  match_id TEXT PRIMARY KEY,
  intent_ref TEXT NOT NULL REFERENCES user_intent(request_id),
  matched_template_id TEXT REFERENCES scenario_template(template_id),
  match_method TEXT NOT NULL CHECK(match_method IN
    ('embedding-cosine','llm-classifier','hybrid','keyword')),  -- §6 통일
  match_confidence REAL NOT NULL,
  candidates TEXT NOT NULL,           -- JSON[] {template_id, score}
  threshold_high REAL NOT NULL, threshold_low REAL NOT NULL,
  decision TEXT NOT NULL CHECK(decision IN
    ('auto-proceed','clarify','reject','route-to-expert')),
  param_slots_filled TEXT, unfilled_required_slots TEXT, clarification_questions TEXT,
  created_at TEXT NOT NULL
);
-- INDEX(intent_ref), INDEX(matched_template_id)

test_plan_spec(
  plan_id TEXT PRIMARY KEY,
  match_ref TEXT REFERENCES template_match(match_id),     -- ✚ 체인 보강
  template_id TEXT REFERENCES scenario_template(template_id),
  match_confidence REAL,
  param_slots TEXT, hypothesis TEXT NOT NULL,
  test_type TEXT NOT NULL CHECK(test_type IN
    ('load','stress','spike','soak','breakpoint','report-only')),
  workload_phases TEXT NOT NULL,      -- JSON[]
  canary_phase TEXT NOT NULL,         -- JSON: canary VU/rate/duration, pass gates
  generator_capacity_check TEXT NOT NULL,  -- JSON: generator health preflight/limits
  max_blast_radius TEXT NOT NULL CHECK(max_blast_radius IN ('low','medium','high')),
  success_criteria TEXT NOT NULL, abort_thresholds TEXT NOT NULL,
  data_policy TEXT NOT NULL, observability_queries TEXT NOT NULL,
  baseline_ref TEXT REFERENCES run_record(run_id),
  created_at TEXT NOT NULL
);
-- INDEX(match_ref), INDEX(template_id)

tool_action(
  action_id TEXT PRIMARY KEY,
  plan_ref TEXT NOT NULL REFERENCES test_plan_spec(plan_id),   -- ✚
  template_id TEXT REFERENCES scenario_template(template_id),
  tool TEXT NOT NULL,                 -- Tool enum (§6)
  operation TEXT NOT NULL CHECK(operation IN
    ('plan','apply','submit','canary','run','abort','query')),
  actor TEXT NOT NULL DEFAULT 'execution-controller'
    CHECK(actor = 'execution-controller'),
  args TEXT NOT NULL,                 -- JSON
  requires_approval INTEGER NOT NULL, fallback_to_human INTEGER NOT NULL,
  approval_status TEXT NOT NULL CHECK(approval_status IN
    ('not_required','pending','approved','rejected')),
  max_cost REAL,
  approval_id TEXT REFERENCES approval_request(approval_id),
  idempotency_key TEXT NOT NULL,
  timeout_sec INTEGER NOT NULL, dry_run INTEGER NOT NULL,
  UNIQUE(idempotency_key)             -- §8 멱등
);
-- INDEX(plan_ref), INDEX(approval_id)

approval_request(
  approval_id TEXT PRIMARY KEY,
  plan_ref TEXT NOT NULL REFERENCES test_plan_spec(plan_id),   -- ✚
  plan_hash TEXT NOT NULL,            -- ✚ 승인 단위(§4)
  requester TEXT NOT NULL,            -- self-approval 차단 비교 대상
  blast_radius TEXT NOT NULL CHECK(blast_radius IN ('low','medium','high')),
  cost_estimate TEXT NOT NULL,
  environment TEXT NOT NULL CHECK(environment IN
    ('local','dev','staging','pre-prod','prod-like','prod')),
  risk_summary TEXT NOT NULL, expires_at TEXT NOT NULL,
  status TEXT NOT NULL CHECK(status IN ('pending','approved','rejected','expired')),
  decided_by TEXT, decided_at TEXT, created_at TEXT NOT NULL
);
-- INDEX(plan_ref), INDEX(plan_hash)

run_record(
  run_id TEXT PRIMARY KEY,
  plan_id TEXT NOT NULL REFERENCES test_plan_spec(plan_id),
  approval_ref TEXT REFERENCES approval_request(approval_id),  -- ✚
  template_id TEXT, provision_id TEXT REFERENCES provision_spec(provision_id),
  status TEXT NOT NULL CHECK(status IN
    ('queued','canary','running','aborting','aborted','completed','failed')),
  started_at TEXT, finished_at TEXT, abort_reason TEXT,
  artifacts TEXT NOT NULL DEFAULT '[]'
);
-- INDEX(plan_id), INDEX(approval_ref), INDEX(status)

observation_bundle(observation_id TEXT PRIMARY KEY,
  run_id TEXT NOT NULL REFERENCES run_record(run_id),
  source TEXT NOT NULL CHECK(source IN ('frozen','measured','measured-remote')),
  collected_as_of TEXT NOT NULL,      -- 관측 시각(관측 지연 추적, [[06-reporter-observability]] §2·§7.2)
  metrics TEXT NOT NULL, logs TEXT NOT NULL, traces TEXT, costs TEXT NOT NULL, anomalies TEXT);
-- INDEX(run_id)

interpretation_spec(interpretation_id TEXT PRIMARY KEY,
  run_id TEXT NOT NULL REFERENCES run_record(run_id),
  observation_ref TEXT REFERENCES observation_bundle(observation_id),  -- ✚
  outcome TEXT NOT NULL CHECK(outcome IN ('pass','fail','inconclusive','aborted')),
  findings TEXT NOT NULL, bottlenecks TEXT, regressions TEXT,
  confidence REAL NOT NULL, evidence_refs TEXT NOT NULL);
-- INDEX(run_id)

final_report(report_id TEXT PRIMARY KEY,
  interpretation_ref TEXT REFERENCES interpretation_spec(interpretation_id),  -- ✚
  run_ref TEXT REFERENCES run_record(run_id), template_id TEXT, match_confidence REAL,
  executive_summary TEXT NOT NULL, technical_summary TEXT NOT NULL,
  slo_results TEXT NOT NULL, risk_assessment TEXT NOT NULL,
  recommendations TEXT NOT NULL, artifacts TEXT NOT NULL, limitations TEXT NOT NULL);

knowledge_pack(pack_id TEXT PRIMARY KEY,
  plan_ref TEXT REFERENCES test_plan_spec(plan_id),
  sources TEXT NOT NULL, tool_versions TEXT NOT NULL, deprecations TEXT,
  freshness_risk TEXT NOT NULL CHECK(freshness_risk IN ('low','medium','high')),
  fetched_at TEXT NOT NULL, stale_after TEXT NOT NULL);

provision_spec(provision_id TEXT PRIMARY KEY,
  topology TEXT NOT NULL CHECK(topology IN ('local','container','k8s','managed-cloud')),
  control_plane_location TEXT NOT NULL DEFAULT 'local-workstation' CHECK(control_plane_location IN
    ('local-workstation','self-hosted-server','cloud-service')),
  runner_location TEXT NOT NULL DEFAULT 'local-container' CHECK(runner_location IN
    ('local-host','local-container','remote-container','cloud-managed','cloud-k8s-job')),
  runner_provider TEXT NOT NULL DEFAULT 'local' CHECK(runner_provider IN
    ('local','aws','gcp','azure','grafana-cloud','on-prem','other')),
  target_provider TEXT NOT NULL DEFAULT 'local' CHECK(target_provider IN
    ('local','aws','gcp','azure','firebase','on-prem','other')),
  network_path TEXT NOT NULL DEFAULT 'same-host' CHECK(network_path IN
    ('same-host','same-vpc','public-internet','private-link','cross-cloud-public','cross-cloud-private')),
  iac_tool TEXT NOT NULL DEFAULT 'managed' CHECK(iac_tool IN
    ('terraform','opentofu','pulumi','cdk','k8s','managed')),
  region TEXT,
  identity TEXT NOT NULL,             -- JSON {mode: oidc|managed-identity|static|none, scope}
  terraform_workspace TEXT, resources TEXT NOT NULL,
  plan_artifact_hash TEXT,            -- 승인 결속용, saved plan/hash
  teardown_strategy TEXT NOT NULL DEFAULT 'ttl-sweeper'
    CHECK(teardown_strategy IN ('destroy-plan','ttl-sweeper','hold-for-debug')),
  state_backend TEXT,                 -- JSON {type, encrypted, locking}
  ttl_minutes INTEGER NOT NULL, cost_estimate TEXT NOT NULL, destroy_plan_required INTEGER NOT NULL);
```

`audit_log` 테이블은 §5에 별도 정의.

---

## 4. `plan_hash` 정의

`plan_hash`는 **승인 단위**의 결정적 지문이다. 승인은 `(ApprovalRequest.approval_id, plan_hash, ToolAction[])` 삼자에 묶이고([[00-overview]] §6, [[01-architecture]] §1), Execution Controller는 실행 직전 번들에서 `plan_hash`를 재계산해 `approval_request.plan_hash`와 **불일치 시 abort**한다(변조 탐지). 상세 승인·게이트 흐름은 [[04-safety-approval-audit]].

**해시 입력(정규화 후 직렬화 대상)**

```
plan_hash = "sha256:v1:" + SHA256(
  canonical_json({
    "plan":   normalize(TestPlanSpec_core),   # 의미 필드만
    "actions": [ normalize(ToolAction_core) for a in actions ordered ],
    "limits": normalize(safe_limits),         # clamp 후 최종 상한
    "provision": normalize(ProvisionSpec_core)
  })
)
```

- **TestPlanSpec_core**: `test_type`, `hypothesis`, `param_slots`, `workload_phases`, `success_criteria`, `abort_thresholds`, `data_policy`, `template_id`, 대상(`target_service`/`target_env`/`endpoints`). 제외: 서버 발급 `plan_id`, `match_confidence`, 타임스탬프.
- **ToolAction_core**: `tool`, `operation`, `args`, `timeout_sec`, `dry_run`, `requires_approval`, `max_cost`. 제외: `action_id`, `approval_id`, `approval_status`, `idempotency_key`(아래 순환 회피).
- **`dry_run` 불변 규칙(플립 금지)**: 승인 번들에 담기는 ToolAction은 생성 시점에 실행 모드가 고정된다. 승인 전 정적 검증은 별도의 `dry_run=true` draft 액션([[04-safety-approval-audit]] A5, 자동 허용)으로 수행하고, **승인 대상 번들에는 처음부터 `dry_run=false`인 실행 액션**을 담는다. 승인 후 `dry_run` 값을 뒤집는 경로는 존재하지 않으므로, 실행 시점 재계산 해시는 `dry_run`을 포함한 채 항상 승인값과 일치한다([[05-runners-and-mock-target]] §7.3).
- **ProvisionSpec_core**: `topology`, `control_plane_location`, `runner_location`, `runner_provider`, `target_provider`, `network_path`, `iac_tool`, `region`, `identity`(**`mode`+`scope` 전체** — scope만 바꾸는 권한 상승 탐지), `resources`, `plan_artifact_hash`, `ttl_minutes`, `destroy_plan_required`, `teardown_strategy`. 승인 후 이들 변경은 hash 불일치로 탐지된다(승인 후 스왑 방지). 대상 Provider와 러너 Provider가 달라지는 cross-cloud 실행은 이 필드들이 승인 번들에 반드시 포함된다([[12-execution-topology-matrix]]).
- **safe_limits**: clamp([[04-safety-approval-audit]]) 적용 후 `max_rps`/`max_vu`/`max_duration`/`max_cost`/`allowed_envs`.

**정규화(normalize) 규칙** — 같은 의미면 같은 해시:
- 객체 키 사전식 정렬, JSON 분리자 고정(`(",",":")`), 비ASCII 보존(`ensure_ascii=false`).
- 수치 표준화(단위 통일: duration→초, rps→정수), 부동소수 캐노니컬 표기.
- `null`/기본값 일관 처리(누락=기본값으로 채운 뒤 직렬화), 배열은 **의미 순서 보존**(`workload_phases`는 순서 유의, `candidates`는 입력 아님).

**`idempotency_key`와의 관계**: 무작위 nonce가 아니라 `plan_hash`에서 **결정적으로 파생**한다(`idem = sha256(plan_hash + ":" + action_ordinal)`). 따라서 해시 입력에서 제외해 순환을 피하고, 동일 승인 재실행 시 동일 키로 멱등 보장(§8). → [[05-runners-and-mock-target]] Execution Controller가 키 충돌을 dedup.

---

## 5. `AuditLog` — append-only 설계

`docs/research/05-safety-governance.md` §11 audit 필드 요구사항을 단일 테이블로 구현한다. **insert-only**: store 계층에 update/delete 메서드를 두지 않고, 선택적으로 tamper-evident 해시 체인을 둔다.

도메인 테이블(`ApprovalRequest.status/decided_by/decided_at`, `RunRecord.status/abort_reason`, `FinalReport` 생성 등)은 **현재 상태 row**로 제한적 갱신을 허용한다. 단, 모든 상태 전이는 같은 트랜잭션에서 `AuditLog` 이벤트를 append해야 한다. `UPDATE`/`DELETE` 금지는 `audit_log`에 적용하며, M1은 도메인 상태를 event-sourcing으로 구현하지 않는다.

```sql
audit_log(
  audit_id TEXT PRIMARY KEY,          -- ULID/UUID
  seq INTEGER NOT NULL UNIQUE,        -- 단조 증가(append 순서)
  ts TEXT NOT NULL,                   -- ISO8601 UTC
  stage TEXT NOT NULL,                -- intake|match|plan|approve|run|observe|interpret|report
  event_type TEXT NOT NULL,           -- created|clamped|approved|rejected|started|aborted|...
  actor TEXT,                         -- 사용자/시스템 행위자
  -- intake/normalize
  request_id TEXT, raw_prompt TEXT, normalized_intent TEXT,  -- JSON
  source_metadata TEXT,               -- KnowledgePack.sources 스냅샷
  -- tool/version
  tool TEXT, tool_version TEXT, api_version TEXT, provider TEXT,
  -- approval binding
  plan_hash TEXT, approval_id TEXT,
  -- action
  action_id TEXT, tool_action_args TEXT, idempotency_key TEXT, timeout_sec INTEGER,
  -- run lifecycle
  run_id TEXT, run_event TEXT,        -- start|stop|abort
  abort_reason TEXT,
  -- artifacts & model
  artifact_refs TEXT,                 -- metrics/logs/traces/artifact 참조 JSON
  model_version TEXT, prompt_version TEXT, confidence REAL,
  -- 무결성(선택)
  prev_hash TEXT, row_hash TEXT       -- row_hash = sha256(prev_hash + canonical(row))
);
-- INDEX(request_id), INDEX(run_id), INDEX(approval_id), INDEX(stage, seq)
```

**append-only 강제**:
- store에 `append_audit(event)`만 노출, `update`/`delete` 없음.
- `seq`는 단조 증가, `row_hash` 체인으로 중간 삭제/변조 탐지(WORM 근사).
- 모든 생명주기 단계([[01-architecture]] §3 "모든 단계: AuditLog append-only 기록")가 최소 1행을 남긴다: raw prompt·normalized intent·source metadata·tool/version·plan_hash·approval_id·ToolAction args·idempotency key·timeout·run start/stop/abort·artifact refs·model/prompt version·confidence.

---

## 6. enum 통일 표

검증([[00-overview]] §5)에서 나온 `TemplateMatch.match_method` 불일치를 **`embedding-cosine`/`llm-classifier`/`hybrid`/`keyword`**로 통일한다. `schemas.py`에 `str, Enum` 단일 정의를 두고 모든 스키마/테이블이 동일 enum을 참조한다.

| enum 이름 | 값 | 사용 스키마 | 비고 |
|---|---|---|---|
| `MatchMethod` | `embedding-cosine` / `llm-classifier` / `hybrid` / `keyword` | TemplateMatch | **정정 통일**(문서 내 불일치 해소) |
| `MatchDecision` | `auto-proceed` / `clarify` / `reject` / `route-to-expert` | TemplateMatch | τ 기반 분기([[01-architecture]] §3) |
| `InputMode` | `template` / `generative` / `report-only` | UserIntent | 비전문가 기본 `template` |
| `RequesterExpertise` | `nonexpert` / `expert` | UserIntent.requester_expertise | 기본 `nonexpert`; `expert`만 저신뢰 자유생성/큐레이션 경로 가능. **신원/역할 속성이지 자기신고 값이 아님** — 프롬프트·요청 바디에서 유래 금지, 세션 신원(로컬은 세션 상수, 사내 단계는 인증 역할)에서만 결정([[07-web-ui-api]] §2.1 인증 경계) |
| `TestType` | `load` / `stress` / `spike` / `soak` / `breakpoint` / `report-only` | UserIntent·ScenarioTemplate·TestPlanSpec | **통일**: 원안의 `chaos-adjacent`는 로컬 범위 제외([[00-overview]] §3)→ 인식 라벨이지만 실행 불가, `reject`/`route-to-expert`로 라우팅. `report-only`는 전 스키마 공통 |
| `Env` | `local` / `dev` / `staging` / `pre-prod` / `prod-like` / `prod` | UserIntent.target_env · ApprovalRequest.environment | 동일 enum 공유 |
| `ToolFamily` | `k6` / `artillery` / `locust` / `jmeter` / `gatling` / `managed` / `report-only` | ScenarioTemplate | 카탈로그 폭 |
| `Tool` | `k6` / `artillery` / `mock` (로컬 활성) ⊂ ToolFamily allowlist | ToolAction | 로컬 실행은 k6/artillery만([[00-overview]] D6/D7) |
| `ToolOperation` | `plan` / `apply` / `submit` / `canary` / `run` / `abort` / `query` | ToolAction.operation | 실행 단계 분기. 로컬 M4는 `submit`/`canary`/`run`/`abort`/`query` 중심 |
| `ToolApprovalStatus` | `not_required` / `pending` / `approved` / `rejected` | ToolAction.approval_status | ApprovalRequest와 별개로 action별 실행 가능 상태를 고정 |
| `TargetPattern` | `REST` / `WebSocket` / `gRPC` / `browser` / `BaaS-mixed` | ScenarioTemplate | |
| `TemplateStatus` | `draft` / `reviewed` / `active` / `deprecated` | ScenarioTemplate | `reviewed`는 engineer review 통과 후 policy review 전 상태. 검색 인덱스에는 `active`만 포함 |
| `ObservationSource` | `frozen` / `measured` / `measured-remote` | ObservationBundle.source / ReportResponse.data_source | M3 fixture는 `frozen`, M4 목 서버 실측은 `measured`, cloud 후속은 `measured-remote` |
| `RequiresApprovalDefault` | `always` / `by-env` / `never` | ScenarioTemplate.requires_approval_default | 템플릿 기본 승인 정책. 실제 실행 전 [[04-safety-approval-audit]]가 env·risk로 최종 판정 |
| `IntakeSafetyVerdict` | `allow` / `deny` | IntakeResponse.safety_verdict | intake 시점 deny-first 사전 차단 결과([[07-web-ui-api]] §2.1). [[04-safety-approval-audit]] §2.1 3분류의 사영 — `deny`=DENY 술어 매치 즉시 단락, `allow`=나머지(REQUIRE_APPROVAL/AUTO_ALLOW 구분은 plan 단계 `classify`가 판정) |
| `ApprovalStatus` | `pending` / `approved` / `rejected` / `expired` | ApprovalRequest | |
| `RunStatus` | `queued` / `canary` / `running` / `aborting` / `aborted` / `completed` / `failed` | RunRecord | 생명주기 단일화(연구 초안 5값을 canary·aborted 포함 7값으로 확장). UI 표시는 대문자이나 영속값은 소문자([[07-web-ui-api]] §5.4) |
| `Outcome` | `pass` / `fail` / `inconclusive` / `aborted` | InterpretationSpec | |
| `BlastRadius` | `low` / `medium` / `high` | ApprovalRequest | |
| `FreshnessRisk` | `low` / `medium` / `high` | KnowledgePack | |
| `Topology` | `local` / `container` / `k8s` / `managed-cloud` | ProvisionSpec | 로컬은 `local` |
| `ControlPlaneLocation` | `local-workstation` / `self-hosted-server` / `cloud-service` | ProvisionSpec.control_plane_location | 계획·승인·audit 앱의 위치. [[12-execution-topology-matrix]] L0/L1/C0/C1/X0 판정 축 |
| `RunnerLocation` | `local-host` / `local-container` / `remote-container` / `cloud-managed` / `cloud-k8s-job` | ProvisionSpec.runner_location | 부하 생성기가 실제로 도는 substrate |
| `ProviderRef` | runner: `local` / `aws` / `gcp` / `azure` / `grafana-cloud` / `on-prem` / `other`; target: `local` / `aws` / `gcp` / `azure` / `firebase` / `on-prem` / `other` | ProvisionSpec.runner_provider / target_provider | 러너 Provider와 대상 Provider를 분리. AWS 대상 + GCP 러너 같은 cross-cloud를 표현 |
| `NetworkPath` | `same-host` / `same-vpc` / `public-internet` / `private-link` / `cross-cloud-public` / `cross-cloud-private` | ProvisionSpec.network_path | egress 비용·allowlist·보안 정책 판단 |
| `IaCTool` | `terraform` / `opentofu` / `pulumi` / `cdk` / `k8s` / `managed` | ProvisionSpec.iac_tool | 로컬/관리형은 `managed`, M5+에서 실제 IaC 선택 |
| `IdentityMode` | `oidc` / `managed-identity` / `static` / `none` | ProvisionSpec.identity.mode | 로컬은 `none`; 클라우드는 OIDC/관리형 ID 우선, 장기 static은 예외 |
| `TeardownStrategy` | `destroy-plan` / `ttl-sweeper` / `hold-for-debug` | ProvisionSpec.teardown_strategy | 기본 `ttl-sweeper`; 클라우드 apply 전 destroy/TTL 경로 명시 |

---

## 7. 도메인 ↔ DB ↔ API 변환 경계와 책임

세 모듈이 단방향 책임을 가진다([[01-architecture]] §4: `models/schemas.py`·`db.py`·`store.py`).

| 모듈 | 역할 | 책임 | import 경계 |
|---|---|---|---|
| `models/schemas.py` | **Pydantic 도메인 + API 모델** | 필드 타입·enum·검증 규칙의 단일 진실원. FastAPI가 그대로 사용 → **OpenAPI 자동 생성**(프론트 타입 소스, [[07-web-ui-api]]). 중첩객체 모델 정의 | DB/ORM 모름 |
| `models/db.py` | **SQLAlchemy ORM** | 테이블·컬럼·인덱스·FK/CHECK 제약, JSON 컬럼 매핑(§3). 영속 표현만 | schemas 모름 |
| `models/store.py` | **매퍼/리포지토리(유일 교차점)** | Pydantic⇄ORM 변환(`to_orm`/`from_orm`), 트랜잭션, FK 무결성, **append-only audit 쓰기**, `plan_hash` 계산 훅, 멱등 강제. 라운드트립 보장 | schemas·db 둘 다 import |

경계 규칙:
- **API는 ORM을 노출하지 않는다.** 라우트는 Pydantic in/out만, 영속은 store 경유.
- 검증은 두 지점: API 엣지(Pydantic) + 영속 직전(store의 무결성/멱등/clamp 사전조건).
- 초기엔 도메인=API Pydantic을 **단일 모델로 공용**(MVP 단순), 표현 분리가 필요해지면 `schemas.py` 내에서 도메인/응답 모델만 파생(추가 모듈 없이).

```python
# store.py 인터페이스 스케치 (계획, 구현 아님)
class Store(Protocol):
    def save_intent(self, x: UserIntent) -> None: ...
    def get_plan(self, plan_id: str) -> TestPlanSpec | None: ...
    def create_approval(self, plan: TestPlanSpec, actions: list[ToolAction]) -> ApprovalRequest: ...
        # 내부에서 plan_hash 계산·고정, idempotency_key 파생
    def append_audit(self, event: AuditEvent) -> None: ...   # insert-only
    def trace(self, report_id: str) -> ChainView: ...        # report→request 역추적
```

---

## 8. 데이터 격리·테스트 데이터·멱등 정책 요약

[[05-runners-and-mock-target]]와 연결. 로컬 범위는 **번들 목 서버**만 대상([[00-overview]] D8, §3).

- **데이터 격리(`TestPlanSpec.data_policy`)**: 합성·시딩 데이터만, 실데이터·실 secret 금지(로컬은 더미/단명 토큰만, [[00-overview]] §3). 격리 단위는 `run_id` — 산출물 디렉터리·로그·메트릭이 `run_id` 네임스페이스로 분리되어 교차 오염 없음.
- **`destructive_allowed`**: 로컬 기본 `false`. 상태 변경 endpoint는 목 서버 한정, 화이트리스트(`target_whitelist`)·env allowlist 통과 시에만.
- **멱등(`ToolAction.idempotency_key`)**: §4대로 `plan_hash`에서 결정적 파생 → `tool_action.idempotency_key`에 `UNIQUE` 제약. 같은 승인 재실행 시 store가 기존 `run_id`를 반환(중복 실행 차단). Execution Controller가 실행 직전 키 충돌을 dedup.
- **timeout/abort**: `ToolAction.timeout_sec`·`RunRecord.abort_reason`·kill switch 이벤트가 audit에 기록(§5). canary 통과 없이는 full run 금지([[00-overview]] §6).
- **재현성**: 목 서버 결정성 + 시딩 고정으로 동일 plan→동일 관측 분포, 테스트 회귀 비교(`baseline_ref`) 가능.

---

## 9. 테스트 가능성 ([[09-testing-strategy]])

D9(테스트 가능성 1급 원칙)에 따라 데이터 계층은 in-memory SQLite(`sqlite:///:memory:`)로 네트워크 없이 결정적으로 검증한다.

| 대상 | 단위 테스트 방법 |
|---|---|
| **모델 검증** | Pydantic 필수 필드 누락→`ValidationError`, 잘못된 enum값 거부, `match_method`가 통일 4값만 허용, 경계값(confidence 0~1, 음수 RPS 거부) |
| **직렬화** | JSON 컬럼 round-trip: `model_dump()→DB write→read→model_validate()` 동치, 비ASCII 보존, 중첩객체 무손실 |
| **`plan_hash` 결정성** | 키 순서/공백/단위 표기만 다른 의미동치 plan → **동일 해시**(property test). 실행 관련 필드 1개 변경 → **다른 해시**. golden-hash 픽스처 고정. `idempotency_key`가 해시에서 결정적으로 재현되는지 |
| **store 라운드트립** | 도메인 insert→fetch 동치, FK 무결성(고아 참조 거부), 추적 체인 `trace(report_id)`가 `request_id`까지 완주, append-only(audit update/delete 부재·`seq` 단조·`row_hash` 체인 검증), 멱등(`UNIQUE` 충돌 시 dedup 동작) |

---

## 부록: 본 문서가 도입한 체인 보강 컬럼(✚) 요약

`docs/research/03` §7·§9.5 스키마에 없으나 추적 체인·승인 단위를 닫기 위해 영속 계층에서 추가: `TestPlanSpec.match_ref`·`canary_phase`·`generator_capacity_check`·`max_blast_radius`, `ToolAction.plan_ref`·`operation`·`actor`·`approval_status`·`max_cost`, `ApprovalRequest.plan_ref`·**`plan_hash`**·`requester`, `RunRecord.approval_ref`, `InterpretationSpec.observation_ref`, `FinalReport.interpretation_ref`·`run_ref`, `ProvisionSpec.control_plane_location`·`runner_location`·`runner_provider`·`target_provider`·`network_path`·`iac_tool`·`region`·`identity`·`plan_artifact_hash`·`teardown_strategy`·`state_backend`. 도메인 의미 변경 없이 승인·실행·클라우드 확장 계약을 보강한다.
