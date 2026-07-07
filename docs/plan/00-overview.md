# 00 · 구현 계획 개요와 확정 결정

> 본 문서군(`docs/plan/*`)은 `docs/research/01~05`와 `docs/research/workflow.md`의 리서치·설계를 **실제 구현 가능한 작업 계획**으로 옮긴 것이다. (상위 입력이던 요구사항 원문·통합 리서치 초안은 별도 파일로 포함하지 않고 그 내용을 `docs/research/*`에 정리·반영했다.)
> 기준일: 2026-06-26 (Asia/Seoul). **이 단계는 계획 수립이며, 아직 코드를 작성하지 않는다.**

---

## 1. 한 줄 정의

비전문가(기획·QE·클라 개발자)가 **자연어 프롬프트**를 입력하면, 시스템이 의도를 구조화하고 **사전 검증된 `ScenarioTemplate`을 매칭**해 슬롯을 채우고, **안전 게이트와 인간 승인**을 거쳐, **좁은 권한의 실행기(k6/Artillery)**로 부하 테스트를 수행하고, **SLO·error budget 기준 리포트**를 만드는 **로컬 우선 웹 서비스**.

핵심 설계 불변식: **자유 생성이 아니라 유한 템플릿 선택 + 슬롯 채움**, **제어 평면/실행 평면 분리**, **실행 권한은 Execution Controller에만**.

---

## 2. 확정된 결정 (Decision Log)

| # | 결정 항목 | 확정 내용 | 비고 |
|---|---|---|---|
| D1 | 산출물 형태 | **웹 서비스** (전용 데스크톱 프로그램 아님) | 사용자 결정 |
| D2 | 배포 단계 | **L0 로컬(맥북/CI 컨테이너) 우선 → 검증 후 L1/C0/C1/X0 토폴로지 선택** | 로컬/클라우드의 정확한 조합은 [[12-execution-topology-matrix]]가 정본 |
| D3 | 백엔드 스택 | **Python + FastAPI** | 사용자 결정 |
| D4 | 프론트엔드 | **React (Vite) SPA** | 제어/실행 평면 분리·인터랙티브 슬롯/승인/리포트 UI |
| D5 | LLM 연동 | **provider 추상화 인터페이스 + 호스티드·로컬 어댑터 둘 다 초기 지원** | 교체 잦음 전제 |
| D6 | 첫 MVP 범위(로컬) | **엔진 코어 + Template/Sandbox(가짜 리포트) + Local Runner(실제 로컬 실행) + Web UI 전체** | 사용자 결정 |
| D7 | 부하 도구 | **k6 + Artillery 어댑터 둘 다** (Template Composer가 도구별 추상화) | 사용자 결정 |
| D8 | 첫 테스트 대상 | **번들 로컬 목(mock) 서버** (지연·에러율·동접 주입 가능) | 결정성·재현·비용0, 위임 판단 |
| D9 | 테스트 가능성 | **"구현 코드 대부분이 자동 테스트 가능"을 1급 설계 원칙으로 채택** | 사용자 강조 ([[09-testing-strategy]]) |
| D10 | 영속화 | 로컬은 **SQLite + SQLAlchemy**, 클라우드 단계에서 Postgres 검토 | 단순·로컬 우선 |
| D11 | 클라우드 실행(Terraform apply, ephemeral runner) | **설계만, 로컬 범위에서 제외**. 별도 후속 단계 | [[08-milestones-roadmap]] M5+ |
| D12 | 로컬/클라우드 토폴로지 | `local/cloud`를 단일 플래그로 쓰지 않고 **제어 평면 위치·러너 위치·러너 Provider·대상 Provider·네트워크 경로**로 분리 | AWS 대상 + GCP 러너 같은 cross-cloud도 표현 |

---

## 3. 범위 / 비범위

### 이번(로컬) 범위에 포함
- 자연어 intake → `UserIntent` 구조화
- 템플릿 레지스트리(YAML) + 의도 분류 + top-k 매칭 + 슬롯 채움 + 안전 clamp
- 안전/승인 게이트, audit log, kill switch(로컬 실행기 수준)
- k6/Artillery 어댑터: 템플릿 채움 → dry-run/정적 검증 → **번들 로컬 목 서버 대상 실제 실행**
- Docker `runner-tools`/`mock-target` 기반 L0 재현성 하니스. 클라우드 에뮬레이터는 구조 검증용이며 성능 근거로 쓰지 않음([[11-local-container-infra]])
- 관측 수집(로컬) → 해석 → 경영/기술 리포트 (가짜/고정 메트릭 → 실제 로컬 메트릭)
- React 웹 UI: 프롬프트 → (명확화 질문) → 계획 검토 → 승인 → 실행 → 리포트
- 전 구간 자동 테스트(단위/통합/계약/E2E)

### 이번 범위에서 명시적 제외
- `terraform apply` / `terraform destroy`, cloud resource 실제 생성
- 클라우드 distributed runner(Grafana Cloud k6 / AWS DLT / Azure LT) **실제 실행**
- L1(remote self-hosted local), C0(local control→cloud), C1(cloud control→cloud), X0(cross-cloud) 실제 실행
- production·staging 등 실서비스 endpoint 대상 부하
- payment/email/SMS/destructive endpoint, 외부 파트너 API 부하
- chaos/fault injection 실행
- 실제 secrets를 사용하는 경로 (로컬은 더미/단명 토큰만)

> 제외 항목은 **설계·인터페이스(ToolAction 계약, ProvisionSpec 등)** 수준에서는 다루되, 실행 코드는 클라우드 단계로 미룬다.

---

## 4. 문서 목록

| 파일 | 내용 |
|---|---|
| [[00-overview]] | 개요·확정 결정·검증 요약·성공 기준 (본 문서) |
| [[01-architecture]] | 제어/실행 평면, 컴포넌트 맵, 리포 구조, 요청 생명주기 |
| [[02-data-model]] | 스키마 12종 → Pydantic/SQLAlchemy 매핑, 영속화 |
| [[03-engine-template-matching]] | 템플릿 레지스트리, 매칭 파이프라인, LLM 추상화, 슬롯·임계값 |
| [[04-safety-approval-audit]] | 3분류 게이트, clamp, 승인(plan_hash), kill switch, 인젝션 방어, audit |
| [[05-runners-and-mock-target]] | k6/Artillery 어댑터, dry-run, 목 서버, Execution Controller |
| [[06-reporter-observability]] | 관측 수집·해석·리포트(가짜→실제), baseline 비교 |
| [[07-web-ui-api]] | 화면 흐름, REST/WS API 계약, 상태 관리 |
| [[08-milestones-roadmap]] | M0~M6 마일스톤 작업 분해(목표/산출물/검증/제외/위험) |
| [[09-testing-strategy]] | 테스트 철학·계층·결정성·평가셋 (D9) |
| [[10-open-questions-reverify]] | 남은 결정·가정, 구현 전 재검증 항목 |
| [[11-local-container-infra]] | 로컬 컨테이너 하니스, 로컬 K8s, 클라우드 IaC 구조 사전 검증 |
| [[12-execution-topology-matrix]] | L0/L1/C0/C1/X0 실행 토폴로지, cross-cloud 조합, ProvisionSpec 매핑 |

---

## 5. 리서치 문서 검증 결과 (1단계 산출)

`docs/research/01~05`를 공식 문서로 **2차에 걸쳐** 교차검증했다. 1차 검토에서 기록한 일부 "정정"이 **재검증(2026-06-26, 공식 1차 문서 웹 확인) 결과 오히려 부정확**해(아래 표의 ⚠ 표시), 본 개정에서 해당 리서치 본문을 **직접 정정**했다. 설계 결론을 흔드는 오류는 없다. 상세·재검증 항목은 [[10-open-questions-reverify]].

| 항목 | 검증·정정 결과(2026-06-26 재검증) |
|---|---|
| AWS DLT MCP 도입 버전 | **v4.1.0(2026-05-13) 도입** — 공식 CHANGELOG `[4.1.0]`이 "introduces MCP server (Bedrock AgentCore) + TypeScript CLI"로 명시. MCP는 read-only 7 tools, 최신 v4.1.4(2026-06-25). ⚠ 1차 검토의 "v4.0.0(2025-11-19) 도입" 표기는 **오류로 정정**(`docs/research/01` 원문 v4.1.0이 정확) |
| Azure dataplane REST API `2026-04-01` | **GA 확정** — `azure-rest-api-specs` `stable/`(비-preview), default moniker `rest-loadtesting-dataplane-2026-04-01`. api-lifecycle 상태 페이지의 `2022-11-01` 표기는 갱신 지연. ⚠ 1차 검토의 "독립 확인 실패"는 **정정**; M5 재검증은 GA 여부가 아니라 지원 엔진/region으로 한정 |
| AWS 네트워크 스트레스 정책 | `aws.amazon.com/ec2/testing/` 원문: 트래픽 서지가 **25Gbps 또는 100Gbps(Gbps)** 초과 시 AWS가 traffic shaping 적용 가능, **volumetric DDoS 시뮬은 명시적 금지**. ⚠ 이는 *승인 임계*가 아니라 셰이핑·금지 정책 — `docs/research/05 §8.2`를 이 내용으로 정정(대량 스트레스는 AWS 네트워크 스트레스 테스트 신청으로 사전 조율) |
| SRE Workbook burn-rate 윈도 | **30일** SLO 윈도(공식 확인: 5%/1h→36x, 2%/1h→14.4x). ✅ 1차 정정(28일→30일)이 옳음 — `docs/research/04 §9` 본문 반영 |
| k6 threshold 실패 종료코드 | **99**(`ThresholdsHaveFailed`, k6 소스 `errext/exitcodes/codes.go` 정의). 사용자 문서엔 "non-zero"로만 표기 — `docs/research/01`·[[05-runners-and-mock-target]] 정정 |
| `TemplateMatch.match_method` enum | **`embedding-cosine`/`llm-classifier`/`hybrid`/`keyword`로 통일** ([[02-data-model]]) |
| k6-learn 직접 인용 문구 | 개념은 정확하나 verbatim 아님 → **의역 표기**로 |

> 정확성 확인된 핵심 사실: Terraform MCP 1.0 GA(2026-06-11)·saved plan=승인·write-only(v1.11+)·HCP cost는 Sentinel만 접근, Grafana k6 Terraform 리소스 6종 실재, Gatling/AWS DLT MCP=read-only, Azure LT=JMeter/Locust만, Firebase 예산 알림은 사용을 cap하지 않음, Firestore 500/50/5·RTDB 200k(하드 한계)·Cloud Run은 revision **기본** max instances 100(설정·quota로 변경 가능 — 하드 한계 아님), OpenAI Sandbox=beta·Structured Outputs/Embeddings 동작, Anthropic routing·인젝션 방어 가이드 — 모두 공식 문서와 일치.

---

## 6. 핵심 안전 불변식 (구현 시 항상 참; 상세 [[04-safety-approval-audit]])

- [ ] 계획 단계 컴포넌트는 실행 권한 0. 실행은 **Execution Controller만**.
- [ ] 모든 실행은 typed `ToolAction`으로만 — **자유 텍스트 shell 생성 금지**.
- [ ] 승인 단위 = `(ApprovalRequest, plan_hash, ToolAction[])`, tamper-proof 저장.
- [ ] 외부 문서/웹/tool 결과는 **evidence data** — 지시문으로 따르지 않음(tool_result에만 JSON 인코딩).
- [ ] 비전문가 기본 모드 = 템플릿 매칭. **저신뢰(τ 미만) 매칭은 실행 거부 + 사람 확인**.
- [ ] 템플릿에 안전 한계(max RPS/duration/cost·대상 env 화이트리스트) 내장, 비전문가가 우회 불가.
- [ ] production 직접 고부하·destructive·외부 파트너 부하는 기본 금지(로컬 범위에선 목 서버만).
- [ ] kill switch는 Orchestrator/Runner 소유, ceiling 초과 시 즉시 abort.
- [ ] canary 통과 없이는 full run 금지. 멱등 키·timeout·audit 항상 기록.

---

## 7. 성공 기준 (이번 로컬 범위)

1. 30개 이상 비전문가 발화 평가셋에서 **오매칭·저신뢰가 실행 경로로 새지 않음**(거부/명확화로 닫힘).
2. 위험 요청(prod/10만 RPS/결제)은 템플릿 매칭과 무관하게 **기본 거부**된다.
3. 승인 없이는 어떤 `ToolAction`도 실행되지 않는다 (계약·테스트로 강제).
4. 번들 목 서버 대상으로 k6·Artillery **둘 다 실제 실행**되어 percentile/error budget 기반 리포트를 만든다. host-native와 Docker smoke가 같은 `tools/versions.json` 기준을 따른다.
5. 모든 리포트 주장에 `evidence_refs`가 있고, `template_id`·`match_confidence`·`limitations`를 포함한다.
6. **테스트 커버리지: 엔진·안전·어댑터·API 핵심 경로가 자동 테스트로 검증된다** (D9).

---

## 8. 작업 순서 한눈 (상세 [[08-milestones-roadmap]])

```
M0 부트스트랩(리포·CI·테스트 하니스·목 서버 골격)
M1 데이터 모델 + 안전 게이트 코어(스키마·clamp·audit, 실행 0)
M2 엔진/템플릿 매칭(의도→top-k→슬롯→clamp) + LLM 추상화 2어댑터
M3 Template/Sandbox MVP(가짜/고정 메트릭 리포트 + Web UI 1차)
M4 Local Runner MVP(k6/Artillery 어댑터, dry-run→목 서버 실제 실행, 실측 리포트)
M4.5 (선택) 실행평면 구조 리허설(L0 kind/k3d Job + L1 remote self-hosted container)
M5 (후속) Cloud MVP 설계(C0: local control→cloud provider, Terraform plan/apply 승인·ephemeral runner·teardown)
M6 (후속) 사내/클라우드 배포(L1/C1/X0 선택, 승인·allowlist 선적용)
```
