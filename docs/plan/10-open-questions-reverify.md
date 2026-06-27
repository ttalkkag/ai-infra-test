# 10 · 남은 결정·가정과 구현 전 재검증 항목

> 본 문서는 (A) 구현 직전 다시 확인할 외부 사실, (B) 계획 수립 중 채택한 가정, (C) 후속 단계에서 정할 열린 결정을 모은다. 확정 결정은 [[00-overview]] §2.

---

## A. 구현 전 재검증 항목 (외부 사실)

### A-1. 검증에서 확정·정정된 사실 (반영 완료)

> 2026-06-26 2차 웹 재검증으로, 1차 검토의 일부 "정정"이 부정확함이 드러나 리서치 본문을 직접 정정했다(상세 [[00-overview]] §5).

| 심각도 | 항목 | 확정·정정 결과 |
|---|---|---|
| med | **Azure dataplane REST API `2026-04-01`** | **GA 확정**(`azure-rest-api-specs` `stable/`·default moniker `rest-loadtesting-dataplane-2026-04-01`). 상태 페이지의 `2022-11-01` 표기는 갱신 지연. 1차의 "GA 미확인"은 정정. M5 재검증은 **지원 엔진/region**으로 한정(GA 여부 아님). 로컬 범위(M1~M4) 영향 없음 |
| med | **AWS DLT MCP 도입 버전** | **v4.1.0(2026-05-13) 도입**(공식 CHANGELOG `[4.1.0]`). 1차의 "v4.0.0(2025-11-19) 도입" 표기는 **오류로 정정**. MCP=read-only 7 tools, 최신 v4.1.4(2026-06-25). [[05-runners-and-mock-target]] 클라우드 확장 주석 정정 완료 |
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

> 로컬 범위(M1~M4)는 위 항목 대부분이 **불필요**하다(목 서버·로컬 도구만). 클라우드/BaaS 실연동(M5~M6) 직전에 체크리스트로 재확인한다.

---

## B. 계획 수립 중 채택한 가정 (이의 없으면 그대로 진행)

| # | 가정 | 근거 | 번복 시 영향 |
|---|---|---|---|
| B1 | 첫 테스트 대상 = **번들 로컬 목 서버**(실서비스/공개 엔드포인트 아님) | 사용자 "스스로 판단 + 코드 대부분 테스트 가능" → 결정성·안전 | 실대상이면 안전 게이트·allowlist를 M4에 앞당겨야 함 |
| B2 | 소형 템플릿(5종)에는 **LLM 분류기 + 임베딩 보조(hybrid)** 우선 | top-k가 유한·소규모 | 순수 임베딩만이면 벡터스토어 도입을 앞당김 |
| B3 | 로컬 영속화 = **SQLite/SQLAlchemy** | 로컬 우선·결정성 | 다중 사용자/동시성 요구 시 Postgres 조기 도입 |
| B4 | 신규 앱은 **별도 루트**(`docs/`와 분리) | 리서치 산출물과 코드 분리 | 위치는 M0에서 사용자와 최종 확정 |
| B5 | 클라우드 단계(M5+)는 **이번 계획 범위 밖**(설계만) | 사용자 "로컬 우선, 추후 배포" | 클라우드를 앞당기면 비용·권한·안전 작업이 선행 |
| B6 | k6·Artillery **바이너리는 로컬 설치** 전제 | Local Runner MVP | 컨테이너화 선호 시 M0에서 도커 하니스 추가 |

---

## C. 후속 단계에서 정할 열린 결정 (지금 확정 불필요)

1. **호스티드 LLM 기본 제공자**: Claude vs OpenAI 중 dev 기본값(추상화 계층이라 교체 가능). 임베딩이 필요해지면 OpenAI 임베딩 vs 로컬 임베딩(sentence-transformers) vs Voyage.
2. **클라우드 러너 1순위**: 로컬 검증 후 Grafana Cloud k6 / AWS DLT / Azure LT 중 조직 환경에 맞춰 선택(M5).
3. **인증/멀티테넌시**: 사내 배포(M6) 시 SSO·역할(기획/QE/SRE)·승인자 지정 방식.
4. **관측 백엔드**: 로컬은 k6 summary로 충분, 클라우드는 Grafana/Prometheus 연동 범위.
5. **템플릿 큐레이션 운영**: 누가 draft→active를 승인하는지, 리뷰 SLA.
6. **실시간(WS/gRPC) 부하 재현 깊이**: Firebase 실시간 경로를 와이어 수준 재현 vs 브라우저 SDK 소수 VU 검증(PoC 필요, [[03-engine-template-matching]] firebase_realtime_sync_probe).
7. **리포트 export 포맷**: 웹 뷰 외 PDF/markdown 중 우선순위.

---

## D. 다음 행동 제안

1. 본 계획서(00~10) 검토 → B 가정 확인/수정.
2. 이의 없으면 **M0 부트스트랩**부터 구현 착수(코드 단계 진입). 착수 전 신규 앱 리포 위치(B4) 확정.
3. M4 완료 시 [[00-overview]] §7 성공 기준으로 게이트 → 통과 시 M5(클라우드) 설계 + A 재검증 체크리스트 수행.
