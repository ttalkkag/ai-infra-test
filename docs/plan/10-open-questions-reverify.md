# 10 · 남은 결정·가정과 구현 전 재검증 항목

> 본 문서는 (A) 구현 직전 다시 확인할 외부 사실, (B) 계획 수립 중 채택한 가정, (C) 후속 단계에서 정할 열린 결정을 모은다. 확정 결정은 [[00-overview]] §2.

---

## A. 구현 전 재검증 항목 (외부 사실)

### A-1. 검증에서 확정·정정된 사실 (반영 완료)

> 2026-06-26 2차 웹 재검증으로, 1차 검토의 일부 "정정"이 부정확함이 드러나 리서치 본문을 직접 정정했다(상세 [[00-overview]] §5).

| 심각도 | 항목 | 확정·정정 결과 |
|---|---|---|
| med | **Azure dataplane REST API `2026-04-01`** | **Microsoft Learn dataplane reference로 확인**(`rest-loadtesting-dataplane-2026-04-01`, 개별 operation의 `api-version=2026-04-01`). API lifecycle의 `2022-11-01`은 지원 baseline으로 해석. M5 재검증은 **지원 엔진/region과 adapter 상수 pin**으로 한정. 로컬 범위(M1~M4) 영향 없음 |
| med | **AWS DLT MCP/CLI 및 최신 버전** | MCP=read-only 7 tools. GitHub Releases는 v4.1.4를 최신으로 보여 주지만 AWS Solutions 랜딩 페이지 표기가 뒤따라올 수 있으므로, M5에서는 실제 배포 artifact URL·CloudFormation 템플릿 버전·CLI 호환 버전을 pin한다. [[05-runners-and-mock-target]] 클라우드 확장 주석 정정 완료 |
| med | **AWS 네트워크 스트레스 정책** | `/ec2/testing/` 원문: 서지 **25Gbps/100Gbps(Gbps)** 초과 시 traffic shaping, **volumetric DDoS 시뮬 명시적 금지**(승인 임계 아님). 대량 스트레스는 AWS 네트워크 스트레스 테스트 신청으로 사전 조율. [[04-safety-approval-audit]]·`docs/research/05 §8.2` 정정 완료 |
| low | **SRE Workbook burn-rate 윈도** | **30일** SLO 윈도 확정(5%/1h→36x). 1차 정정(28일→30일)이 옳음. [[06-reporter-observability]]·`docs/research/04 §9` 반영 완료 |
| low | **k6 threshold 실패 종료코드** | **99**(`ThresholdsHaveFailed`, k6 소스 정의). 사용자 문서엔 "non-zero"로만 표기. `docs/research/01`·[[05-runners-and-mock-target]] 정정 완료 |
| low | **`match_method` enum 불일치** | `embedding-cosine`/`llm-classifier`/`hybrid`/`keyword`로 **통일**([[02-data-model]]) |
| low | k6-learn 직접 인용 | 개념 정확하나 verbatim 아님 → 의역 표기 |

### A-2. 리서치 문서가 스스로 지목한 재검증(구현 직전) — 요약

- 도구: k6 2.0 `k6 x mcp`=preview·`k6 x agent`=stable 상태, Grafana k6 Terraform provider 리소스 스키마, AWS DLT 최신 번들 엔진 버전, Azure 지원 엔진/region.
- Terraform: MCP 1.0.x read/write 경계·`ENABLE_TF_OPERATIONS`·보안 기본값, HCP 정책/비용 단계 정밀 순서, write-only provider 지원, OpenTofu 암호화.
- 오케스트레이션: OpenAI Sandbox=beta(GA 재확인), LangGraph 노드 재실행 멱등성, **τ_high/τ_low 임계 오프라인 보정**.
- SLO: k6 버전 고정 후 옵션 문법, 클라이언트(k6) vs 서버(OTel/Prom) percentile 정합, Firestore/RTDB/Cloud Run 한도 수치, 게임 tick rate 메트릭 실측.
- 안전: GitHub required reviewers 플랜 제약(private repo는 Enterprise), 클라우드 사업자 부하/DoS 정책(AWS Simulated Events·Azure ROE·GCP AUP), BaaS 테스트 전용 프로젝트 격리·App Check 영향.
- BaaS: Firestore 실시간=gRPC/WebChannel·RTDB=WebSocket 재현 경로, k6 ws/grpc/browser·Artillery 엔진 버전, Firebase는 `google-beta` provider·일부 리소스(RTDB 규칙/MFA/기본 Storage 버킷) TF 미지원.
- 실행 토폴로지: L1(remote self-hosted local), C0/C1(cloud managed), X0(cross-cloud) 채택 시 runner Provider와 target Provider 각각의 정책·egress·source IP/load zone·관측 지연을 재확인([[12-execution-topology-matrix]]).

> 로컬 범위(M1~M4)는 위 항목 대부분이 **불필요**하다(목 서버·로컬 도구만). 클라우드/BaaS 실연동(M5~M6) 직전에 체크리스트로 재확인한다.

---

## B. 계획 수립 중 채택한 가정 (이의 없으면 그대로 진행)

| # | 가정 | 근거 | 번복 시 영향 |
|---|---|---|---|
| B1 | 첫 테스트 대상 = **번들 로컬 목 서버**(실서비스/공개 엔드포인트 아님) | 사용자 "스스로 판단 + 코드 대부분 테스트 가능" → 결정성·안전 | 실대상이면 안전 게이트·allowlist를 M4에 앞당겨야 함 |
| B2 | 소형 템플릿(5종)에는 **LLM 분류기 + 임베딩 보조(hybrid)** 우선 | top-k가 유한·소규모 | 순수 임베딩만이면 벡터스토어 도입을 앞당김 |
| B3 | 로컬 영속화 = **SQLite/SQLAlchemy** | 로컬 우선·결정성 | 다중 사용자/동시성 요구 시 Postgres 조기 도입 |
| B4 | 신규 앱은 **별도 루트**(`docs/`와 분리) | 리서치 산출물과 코드 분리 | 위치는 M0에서 사용자와 최종 확정 |
| B5 | L1/C0/C1/X0 후속 토폴로지는 **이번 L0 구현 범위 밖**(설계만) | 사용자 "로컬 우선, 추후 배포" | 후속 토폴로지를 앞당기면 비용·권한·안전 작업이 선행 |
| B6 | 로컬 실행은 **host-native 빠른 루프 + Docker 재현성 하니스**를 모두 둔다 | Local Runner MVP, CI 재현성 | Docker를 제외하면 CI/온보딩 재현성이 약해짐. host-native를 제외하면 개발 반복이 느려짐 |
| B7 | "로컬/클라우드"는 단일 플래그가 아니라 L0/L1/C0/C1/X0 토폴로지로 기록한다 | 배포·러너·대상 Provider가 독립적으로 바뀜 | 단일 `local/cloud` 플래그로는 AWS 대상+GCP 러너, cloud VM의 self-hosted container 같은 조합을 표현하지 못함 |

---

## C. 후속 단계에서 정할 열린 결정 (지금 확정 불필요)

1. **호스티드 LLM 기본 제공자**: Claude vs OpenAI 중 dev 기본값(추상화 계층이라 교체 가능). 임베딩이 필요해지면 OpenAI 임베딩 vs 로컬 임베딩(sentence-transformers) vs Voyage.
2. **클라우드 러너 1순위**: 로컬 검증 후 Grafana Cloud k6 / AWS DLT / Azure LT 중 조직 환경에 맞춰 선택(M5).
3. **후속 배포 토폴로지**: L1(remote self-hosted local), C0(local control→cloud), C1(cloud control→cloud), X0(cross-cloud) 중 어떤 모델을 먼저 제품화할지. AWS 대상+GCP/Grafana Cloud 러너처럼 Provider가 갈라지는 경우 별도 승인 체계 필요.
4. **인증/멀티테넌시**: 사내 배포(M6) 시 SSO·역할(기획/QE/SRE)·승인자 지정 방식.
5. **관측 백엔드**: 로컬은 k6 summary로 충분, 클라우드는 Grafana/Prometheus 연동 범위.
6. **템플릿 큐레이션 운영**: 누가 draft→reviewed→active를 승인하는지, 리뷰 SLA.
7. **실시간(WS/gRPC) 부하 재현 깊이**: Firebase 실시간 경로를 와이어 수준 재현 vs 브라우저 SDK 소수 VU 검증(PoC 필요, [[03-engine-template-matching]] firebase_realtime_sync_probe).
8. **리포트 export 포맷**: 웹 뷰 외 PDF/markdown 중 우선순위.

---

## D. 다음 행동 제안

1. 본 계획서(00~12) 검토 → B 가정 확인/수정.
2. 이의 없으면 **M0 부트스트랩**부터 구현 착수(코드 단계 진입). 착수 전 신규 앱 리포 위치(B4) 확정.
3. M4 완료 시 [[00-overview]] §7 성공 기준으로 게이트 → 통과 시 M4.5의 L0 kind/k3d 구조 리허설과 L1 remote self-hosted 리허설 여부를 나눠 결정하고, M5(C0/C1/X0) 우선순위와 A 재검증 체크리스트를 수행.
