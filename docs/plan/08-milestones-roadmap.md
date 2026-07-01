# 08 · 마일스톤 로드맵

> 확정 범위: 엔진 코어 + Template/Sandbox + Local Runner + Web UI 전체(L0 로컬). L1/C0/C1/X0 토폴로지는 후속. ([[00-overview]] D6/D11/D12, [[12-execution-topology-matrix]])
> 각 마일스톤은 **목표 / 산출물 / 검증 / 제외 / 위험·완화**로 정의한다. 검증 기준은 [[09-testing-strategy]]의 테스트로 자동화한다.

리서치의 4단계(Research Prototype → Template/Sandbox → Local Runner → Cloud)를 본 프로젝트의 마일스톤으로 재배치한다. **Research Prototype은 이미 완료**(리서치 문서 + 1단계 검증)이므로 M1부터 코드화한다.

```
M0 부트스트랩 ─ M1 데이터·안전 코어 ─ M2 엔진/매칭 ─ M3 Sandbox MVP ─ M4 Local Runner MVP ─ M4.5 실행평면 리허설 ─┊─ M5 Cloud 설계 ─ M6 배포
                                                                  (L0 범위 끝)       (선택)                         (후속)
```

의존: M1→M2→M3→M4 순차(각 단계가 앞 산출물에 의존). M0는 전 단계 토대. M4.5는 선택 리허설이며, M5/M6은 M4 검증 후.

---

## M0 · 부트스트랩

- **목표**: 코드 한 줄을 더하기 전에 **테스트 가능한 골격**을 세운다. (D9 선반영)
- **산출물**:
  - 리포 구조([[01-architecture]] §4), `pyproject.toml`(FastAPI/SQLAlchemy/pydantic/pytest), `frontend/`(Vite+TS+vitest)
  - CI 파이프라인(lint·type·test), pre-commit
  - 테스트 하니스: 목 LLM 어댑터(`llm/`의 fake), pytest fixtures, OpenAPI→프론트 타입 생성 파이프라인
  - **번들 목 서버 골격**(`mock-target/`): 헬스 엔드포인트 + 지연/에러율 주입 파라미터 stub ([[05-runners-and-mock-target]])
  - Docker 재현성 하니스 초안: `app`/`mock-target`/`runner-tools` 서비스 경계, `tools/versions.json` 버전 매니페스트
  - `/healthz` 라우트로 백·프론트 왕복 1개 통과
- **검증**: CI green, 빈 엔드포인트 계약 테스트 통과, 목 LLM으로 결정적 실행 확인, Docker smoke는 선택 게이트로 1회 실행 가능
- **제외**: 실제 엔진 로직, 실제 LLM 호출, 실제 부하 도구
- **위험·완화**: 과설계 → 모듈 스텁만, 스키마 10여 종 이하 유지([[00-overview]] §3)

## M1 · 데이터 모델 + 안전 코어 (실행 0)

- **목표**: 스키마·영속·정책·승인·감사를 먼저 굳혀, 이후 모든 기능이 안전 불변식 위에 올라가게 한다.
- **산출물** ([[02-data-model]], [[04-safety-approval-audit]]):
  - Pydantic 도메인 + SQLAlchemy(SQLite) 12 스키마, `models/store.py` 라운드트립
  - `safety/policy.py`(3분류 규칙), `safety/clamp.py`(safe_limits·allowlist), `safety/audit.py`(append-only)
  - `plan_hash` 계산, 승인 단위 `(ApprovalRequest, plan_hash, ToolAction[])`
  - `safety/sanitize.py`(외부 콘텐츠=데이터) 스텁
- **검증**:
  - **불변식 테스트**: 승인 없는 `ToolAction`은 실행 진입 불가(계약상 거부)
  - 위험 요청 분류(prod/고부하/결제) → 기본 금지 단위 테스트 표
  - clamp 상한 초과 → 거부/승인 경로 테스트, plan_hash 결정성 테스트
- **제외**: 매칭·LLM·실행·UI
- **위험·완화**: 정책 규칙 누락 → 표 기반 테스트로 모든 분류 케이스 망라; enum 불일치 → [[02-data-model]]에서 통일(match_method 등)

## M2 · 엔진 / 템플릿 매칭 + LLM 추상화

- **목표**: 자연어 → top-k 매칭 → 슬롯 채움 → clamp 가 제어 평면에서 동작.
- **산출물** ([[03-engine-template-matching]]):
  - `engine/orchestrator.py`·`matcher.py`·`strategy.py`
  - `llm/base.py` + `hosted.py` + `local.py` + `registry.py` (둘 다 초기 지원, 목 어댑터)
  - `templates/` 5종 YAML + 레지스트리 로더, 큐레이션 상태(draft→reviewed→active)
  - `eval/` 발화 30+ 평가셋, τ_low/τ_high 오프라인 보정 노트
- **검증**:
  - 평가셋에서 오매칭·저신뢰가 실행 경로로 새지 않음(거부/명확화로 닫힘)
  - 필수 슬롯 누락 → 추정 없이 질문, 위험 요청 → 매칭과 무관하게 차단
  - 목 LLM 기반 결정적 회귀 테스트
- **제외**: 실제 부하 실행, UI(다음 단계), 클라우드
- **위험·완화**: structured output 신뢰성 편차 → 스키마 강제 + 검증·재질의; 임계 과민/둔감 → 평가셋 기반 튜닝, 거짓매칭 vs 불필요 명확화 트레이드오프 기록

## M3 · Template/Sandbox MVP (가짜/고정 메트릭 + Web UI 1차)

- **목표**: 비전문가가 웹에서 프롬프트 → (명확화) → 계획 검토 → 승인 → **가짜/고정 메트릭 리포트**까지 한 흐름을 경험. 실제 부하 0.
- **산출물** ([[07-web-ui-api]], [[06-reporter-observability]]):
  - REST/WS API(intake·match·slots·plan·approve·report) 구현
  - React 화면: 프롬프트 → 슬롯/명확화 → 계획 검토 → 승인 → 리포트
  - 가짜/고정 메트릭 리포트(경영/기술 분리, evidence_refs, template_id·match_confidence·limitations, "가짜" 라벨)
- **검증**:
  - E2E(Playwright): 정상 흐름 + 저신뢰 거부 + 위험 차단 + 슬롯 누락 재질의
  - 모든 리포트가 limitations·template_id·match_confidence 포함, "AI가 실제 실행" 과장 표현 없음
- **제외**: 실제 부하 도구 호출, cloud 자격증명, Terraform
- **위험·완화**: 기획 과장 → report-only/dry-run 라벨 고정; UX 복잡도 → 비전문가 용어 최소화·차단 사유 설명

## M4 · Local Runner MVP (실제 로컬 실행)

- **목표**: 승인된 계획을 **k6/Artillery로 번들 목 서버 대상 실제 실행**하고 실측 리포트를 만든다.
- **산출물** ([[05-runners-and-mock-target]], [[06-reporter-observability]]):
  - `runners/base.py`·`k6_adapter.py`·`artillery_adapter.py`·`compose.py`·`controller.py`
  - 번들 목 서버 완성(지연·에러율·동시성·포화 주입, REST + WS)
  - host-native 실행과 Docker smoke 실행 모두 지원(같은 profile·같은 버전 매니페스트)
  - dry-run/정적 검증 → 승인 → canary → full run → 실측 수집 → 해석 → 리포트
  - 생성기 병목 감지(dropped_iterations 등), baseline 비교
- **검증**:
  - k6·Artillery **둘 다** 목 서버 상대 실제 실행, percentile/error budget 리포트 생성
  - Docker smoke에서 k6 1개·Artillery 1개 최소 시나리오가 `mock-target`에 대해 완료
  - 목 서버에 알려진 지연/에러 주입 → 해석기가 기대 판정 산출(골든 테스트)
  - canary 실패 시 full run 미승격, kill switch 동작, 승인 없이는 실행 불가(재확인)
- **제외**: 클라우드 러너 실제 실행, 실서비스 대상, chaos, payment/email/SMS
- **위험·완화**: 도구 미설치/버전 차 → 버전 매니페스트·설치 점검·Docker smoke·CI 캐시; 생성기 자체 병목 오판 → 생성기 신호 분리 테스트; 로컬 자원 한계 → 목 서버·부하 상한을 보수적으로

> **여기까지가 이번 로컬 범위의 완성선.** [[00-overview]] §7 성공 기준을 모두 만족해야 M5로 진행.

---

## M4.5 · (선택) 실행평면 구조 리허설 (L0 kind/k3d + L1 Remote Self-hosted)

- **목표**: M5 cloud action 전에 실행평면 구조를 두 단계로 검증한다. (a) L0 로컬 kind/k3d Job으로 K8s Job·timeout·TTL·kill switch 구조를 비용 0으로 확인하고, (b) 필요 시 L1 토폴로지([[12-execution-topology-matrix]])를 remote self-hosted container로 리허설한다. L1은 앱/러너를 클라우드 VM 또는 사내 서버에 올리되, 부하 생성은 관리형 cloud runner가 아니라 해당 host의 Docker/k3d/kind 컨테이너 자원으로 수행한다.
- **산출물**:
  - local kind/k3d Job runbook(Job completions/parallelism, `activeDeadlineSeconds`, TTL cleanup, artifact collection)
  - remote compose/k3d/kind runbook(서버 bootstrap, image pull, profile mount, artifact collection)
  - L1 선택 시 source IP 고정·allowlist·egress cost 체크리스트
  - self-hosted container runner adapter 계약(`runner_location=remote-container`)
- **검증**:
  - 로컬 kind/k3d Job이 `mock-target` 대상에서 L0 host/Docker 실행과 동일한 결과·abort semantics를 보이는지 확인
  - remote 대상이 `mock-target`이면 L0와 동일한 결과가 나오는지 확인
  - 대상이 dev/staging이면 승인·시간창·allowlist·kill switch 없이는 실행 불가
- **제외**: 관리형 cloud load test, production, cross-cloud full run
- **위험·완화**: "컨테이너니까 로컬"이라는 오해 → target이 cloud/dev/staging이면 승인 필수. 리포트에 self-hosted runner와 cloud-managed runner를 구분 표기.

---

## M5 · (후속) Cloud MVP 설계·부분 구현

- **목표**: C0 토폴로지(local control → Cloud Provider)를 기준으로 Terraform `plan→정책/비용→승인→apply` 게이트와 ephemeral runner·teardown을 설계하고 일부 구현. 실제 고부하·실서비스 대상은 여전히 금지.
- **산출물**: Terraform module skeleton 계획, plan JSON policy checklist, approval workflow(plan_hash↔approval), canary→full 승격 규칙, teardown/TTL 회수, OIDC 단명 토큰·비용 3중 백스톱 설계, `runner_provider`/`target_provider`/`network_path`별 approval bundle
- **검증**: `plan -out` artifact hash ↔ approval id 연결, cost/quota cap, canary 없이는 full run 불가, destroy/TTL 회수 확인
- **제외**: production 대상, chaos, 외부 API
- **위험·완화**: cleanup 실패 → TTL sweeper+cost alarm; 비용 폭증 → max cost budget+auto abort; **Azure 경로 시 지원 엔진/region 재확인(`2026-04-01` dataplane API는 Microsoft Learn reference로 확인), AWS DLT artifact/version pin**([[10-open-questions-reverify]]). X0(cross-cloud)는 양 Provider 정책·egress·allowlist 검토 없이는 실행 금지([[12-execution-topology-matrix]] §5)

## M6 · (후속) 사내/클라우드 배포·실대상 연동

- **목표**: L1/C1/X0 중 조직 배포 모델을 선택해 사내 서버 또는 클라우드에 배포하고, 실제 dev/staging 대상 연동(승인·env allowlist·kill switch 선적용)을 수행한다.
- **산출물**: 배포 IaC, 인증/권한, 관측 연동(Grafana/Prometheus), 운영 런북
- **검증**: 실대상은 read-only smoke부터, on-call 승인·시간창·공지·rollback 없으면 실행 불가. X0는 target Provider와 runner Provider 양쪽 승인·비용·네트워크 증적 필요
- **제외**: production 직접 고부하(영구 기본 금지)
- **위험·완화**: 영속화 Postgres 이전, BaaS는 **테스트 전용 프로젝트 격리 + 쿼터/예산 cap**(예산 알림은 사용을 cap하지 않음 전제)

---

## 산출물 ↔ 마일스톤 추적표

| 컴포넌트 문서 | 주 구현 마일스톤 |
|---|---|
| [[02-data-model]] | M1 |
| [[04-safety-approval-audit]] | M1 (+ M4 kill switch, M5 cloud 정책) |
| [[03-engine-template-matching]] | M2 |
| [[07-web-ui-api]] | M3 (+ M4 실행 화면) |
| [[06-reporter-observability]] | M3(가짜) → M4(실측) |
| [[05-runners-and-mock-target]] | M4 (+ M5 클라우드 러너 확장) |
