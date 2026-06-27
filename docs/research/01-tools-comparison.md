# 부하 테스트 도구 상세 비교 (AI + Terraform 기반 프롬프트 주도 부하 테스트 자동화)

## 1. 한 줄 요약

- 기준일: **2026-06-26 (Asia/Seoul)**. 모든 `fetched_at`은 2026-06-26.
- 결론 한 줄: **AI 생성 친화성 + scripts-as-code + 명시적 SLO(threshold) + 실재하는 Terraform 리소스 + 관측성**을 한 번에 만족하는 기본 후보는 **Grafana k6 / Grafana Cloud k6**이다. 클라우드 종속 시나리오는 AWS DLT(CloudFormation/CDK-first) 또는 Azure Load Testing(`2026-04-01` GA REST API, JMeter/Locust)이 현실적이고, self-managed MVP는 Artillery/Locust가 빠르다. Gatling/BlazeMeter/NeoLoad는 엔터프라이즈 폭은 넓지만 상용 종속이 크다.
- 이 문서는 **리서치 단계 산출물**이다. 코드, Terraform apply, 클라우드 리소스 생성, 실제 부하 실행을 포함하지 않는다.

### 이전 리서치 대비 핵심 정정 (이번에 공식 문서로 재확인)

| # | 항목 | 이전 서술 | 이번 확인 결과 |
|---|---|---|---|
| 1 | Grafana Terraform provider k6 리소스 | `grafana_k6_load_test` resource "존재 추정" | **6종 실재 확정**: `grafana_k6_installation`, `grafana_k6_project`, `grafana_k6_load_test`, `grafana_k6_project_limits`, `grafana_k6_project_allowed_load_zones`, `grafana_k6_schedule` (beta/experimental 라벨 없는 정식 리소스) |
| 2 | k6 MCP server 상태 | "지원" | **MCP 서버(`k6 x mcp`)는 preview**, `k6 x agent`는 2.0 정식 릴리스에 포함(공식 stability 배지 없음). 둘 다 k6 2.0(2026-05-12)에서 도입 |
| 3 | Azure Load Testing 지원 엔진 | "JMeter/Locust/**Playwright**" | **JMeter + Locust만 부하 엔진(GA)**. Playwright는 부하 엔진 아님 → Playwright Workspaces(기능/E2E)로 별개. 부하 엔진은 "JMeter/Locust 외 미지원" 공식 명시 |
| 4 | Gatling MCP server 범위 | "조회 외 deploy/start도 가능" | **MCP 프로토콜 표면은 read-only(4개 `list_*` 도구)**. deploy/start는 **별도 "Claude Skills"(`gatling-build-tools`)**가 수행 — MCP 자체가 실행하는 것이 아님 |
| 5 | AWS DLT 버전/MCP | "선택적 MCP integration" | **v4.1.4(2026-06-25)**. MCP/CLI는 **v4.1.0(2026-05-13)**에서 도입. MCP는 **read-only 7개 도구**(`list_scenarios` 등), 시작/중지/수정 불가 |
| 6 | Gatling Terraform | (미상) | **공식 Terraform _모듈_(provider 아님)** 존재: `gatling/control-plane/{aws,azure,gcp}` (Control Plane 인프라 부트스트랩용, 테스트 리소스 아님) |
| 7 | Azure dataplane API `2026-04-01` | "GA 추정" | **GA 확정**(reference defaultMoniker, `-preview` 없음). 단 api-lifecycle 상태 페이지는 아직 `2022-11-01`을 baseline으로 표기(문서 갱신 지연) |

### 이번 보강(2026-06-26): 게임사·Firebase·실시간·템플릿 관점 추가

- **새 맥락**: 게임 회사(20여 개 타이틀), 백엔드는 **Firebase**(클라이언트 개발자가 서버 로직을 직접 구현, 인프라 전문성 낮음). 목표는 QE/기술지원팀의 **수동 부하테스트 병목 해소** — 비전문가가 **자연어 프롬프트 → 가장 유사한 사전 검증 템플릿(시나리오) 실행**. 환경은 **범용 유지**(AWS/GCP/Azure/Firebase 전부, Firebase로 좁히지 않음). 게임 시나리오 예: 로그인 → 컨텐츠 진행 → 게임 종료.
- **이 보강이 더하는 4개 관점**(기존 출처 레지스트리·도구 카드·점수 매트릭스는 **보존**, 신규는 §2.1·§7~§10에 추가): §7 Firebase/BaaS를 "부하 대상"으로 분석, §8 실시간/프로토콜 부하 도구 커버리지, §9 멀티 대상(AWS/Azure/GCP/Firebase) 도구 적합성, §10 템플릿(시나리오) 라이브러리화 적합성. 신규 불확실성은 §11에 추가.
- **보강 결론 한 줄**: Firebase처럼 **클라이언트 SDK·실시간 스트리밍(Firestore 실시간 Listen=gRPC/WebChannel, RTDB=WebSocket) 의존이 큰 대상은 일반 HTTP 부하도구로 그대로 재현 불가**. 재현 경로는 (a) WS/gRPC 모듈로 와이어 프로토콜 직접 구사(SDK 프레이밍/인증 재구현 부담) 또는 (b) 실브라우저로 실제 SDK 구동(자원 비용 큼)뿐이다. 단일 도구로 프로토콜 폭이 가장 넓은 것은 **k6**(WS+gRPC+browser), 실제 SDK 재현 편의·비전문가 YAML 템플릿화는 **Artillery**(socketio/playwright, YAML+CSV)가 강점. 도구 선택은 결국 **대상이 REST냐·실시간이냐·관리형 BaaS냐**로 갈린다.

---

## 2. 출처 레지스트리

`trust_level`: 공식=벤더 1차 문서, 공식(repo)=프로젝트 공식 저장소. `freshness_risk`: 가격/preview·GA/API version 등 변동성. 다수 docs 페이지는 게시/수정일 비노출 → `미표시`.

| 주제 | source_url | vendor | published/updated | fetched_at | trust_level | freshness_risk | 핵심 근거 |
|---|---|---|---|---|---|---|---|
| k6 executors | https://grafana.com/docs/k6/latest/using-k6/scenarios/executors/ | Grafana | 미표시 | 2026-06-26 | 공식 | 낮음 | `constant-vus`/`ramping-vus`/`constant-arrival-rate`/`ramping-arrival-rate`/`shared-iterations`/`per-vu-iterations`/`externally-controlled` |
| k6 thresholds | https://grafana.com/docs/k6/latest/using-k6/thresholds/ | Grafana | 미표시 | 2026-06-26 | 공식 | 낮음 | pass/fail 기준, breach 시 non-zero exit code → CI 게이트 |
| k6 2.0 릴리스 | https://grafana.com/blog/k6-2-0-release/ | Grafana | 2026-05-12 | 2026-06-26 | 공식 | 낮음 | `k6 x agent`/`k6 x mcp` 도입 시점 |
| k6 AI assistant | https://grafana.com/docs/k6/latest/set-up/configure-ai-assistant/ | Grafana | 미표시 | 2026-06-26 | 공식 | 중간 | MCP 서버 "currently in preview" |
| k6 MCP tools | https://grafana.com/docs/k6/latest/set-up/configure-ai-assistant/tools-prompts-resources/ | Grafana | 미표시 | 2026-06-26 | 공식 | 중간 | tools 6 / prompts 2 / resources(docs·types) |
| k6 x agent | https://grafana.com/docs/k6/latest/set-up/configure-ai-assistant/bootstrap-with-k6-x-agent/ | Grafana | 미표시 | 2026-06-26 | 공식 | 낮음 | SKILL.md + MCP 등록 스캐폴딩(stable) |
| Grafana TF provider k6 | https://registry.terraform.io/providers/grafana/grafana/latest/docs/resources/k6_load_test | Grafana | 미표시 | 2026-06-26 | 공식 | 중간 | `grafana_k6_*` 6 리소스 실재(주석 1번 참조) |
| Grafana TF provider repo | https://github.com/grafana/terraform-provider-grafana/tree/main/docs/resources | Grafana | 미표시 | 2026-06-26 | 공식(repo) | 낮음 | `k6_*.md` 6개로 리소스명 직접 확인 |
| k6 Private Load Zone | https://grafana.com/docs/grafana-cloud/testing/k6/author-run/private-load-zone/ | Grafana | 미표시 | 2026-06-26 | 공식 | 낮음 | 고객 k8s 내부 부하 생성기, 메트릭 Grafana Cloud push |
| k6 public load zones | https://grafana.com/docs/grafana-cloud/testing/k6/author-run/use-load-zones/ | Grafana | 미표시 | 2026-06-26 | 공식 | 낮음 | 21개 멀티리전, `options.cloud.distribution` |
| k6 real-time outputs | https://grafana.com/docs/k6/latest/results-output/real-time/ | Grafana | 미표시 | 2026-06-26 | 공식 | 낮음 | Prometheus RW/OTel/Datadog/CloudWatch/InfluxDB 등 |
| k6 Prometheus RW | https://grafana.com/docs/k6/latest/results-output/real-time/prometheus-remote-write/ | Grafana | 미표시 | 2026-06-26 | 공식 | 중간 | 플래그 `--out experimental-prometheus-rw`(experimental 라벨 유지) |
| k6 OTel output | https://grafana.com/docs/k6/latest/results-output/real-time/opentelemetry/ | Grafana | 미표시 | 2026-06-26 | 공식 | 중간 | `--out opentelemetry` |
| k6 Cloud REST API | https://grafana.com/docs/grafana-cloud/testing/k6/reference/cloud-rest-api/ | Grafana | 미표시 | 2026-06-26 | 공식 | 중간 | 현재 **v6**(`api.k6.io/cloud/v6`), Bearer+`X-Stack-Id` |
| k6 legacy API deprecation | https://grafana.com/docs/grafana-cloud/testing/k6/reference/cloud-rest-api/deprecated-rest-api/ | Grafana | 미표시 | 2026-06-26 | 공식 | 중간 | 레거시 API deprecated, 제거 예정 |
| k6 v2 마이그레이션 | https://grafana.com/docs/k6/latest/get-started/migrating-to-v2/ | Grafana | 미표시 | 2026-06-26 | 공식 | 낮음 | `options.ext.loadimpact` 제거 → `options.cloud` |
| k6 가격/무료티어 | https://grafana.com/pricing/ | Grafana | 미표시 | 2026-06-26 | 공식 | 높음 | 과금 단위 **VUh**, Free에 월 **500 VUh** |
| AWS DLT 아키텍처 | https://docs.aws.amazon.com/solutions/latest/distributed-load-testing-on-aws/architecture-overview.html | AWS | 미표시 | 2026-06-26 | 공식 | 낮음 | Taurus on ECS/Fargate, CFN from CDK, Cognito가 REST/CLI/MCP 접근 관리 |
| AWS DLT features | https://docs.aws.amazon.com/solutions/latest/distributed-load-testing-on-aws/features.html | AWS | 미표시 | 2026-06-26 | 공식 | 낮음 | JMeter/K6/Locust/단순 HTTP, 멀티리전, 스케줄 |
| AWS DLT 동작 | https://docs.aws.amazon.com/solutions/latest/distributed-load-testing-on-aws/how-distributed-load-testing-on-aws-works.html | AWS | 미표시 | 2026-06-26 | 공식 | 낮음 | Step Functions가 리전별 ECS task, S3/CloudWatch/DynamoDB |
| AWS DLT MCP tools | https://docs.aws.amazon.com/solutions/latest/distributed-load-testing-on-aws/mcp-tools-specification.html | AWS | 미표시 | 2026-06-26 | 공식 | 중간 | **read-only 7 tools**, 시나리오/설정 수정 불가 |
| AWS DLT CHANGELOG | https://github.com/aws-solutions/distributed-load-testing-on-aws/blob/main/CHANGELOG.md | AWS | 2026-06-25 | 2026-06-26 | 공식(repo) | 낮음 | v4.1.4(2026-06-25), MCP/CLI는 v4.1.0(2026-05-13) |
| Azure ALT 개요 | https://learn.microsoft.com/en-us/azure/app-testing/load-testing/overview-what-is-azure-load-testing | Microsoft | 2026-06-11 갱신 | 2026-06-26 | 공식 | 낮음 | "JMeter 또는 Locust만 지원", 엔진 인스턴스/멀티리전/Azure Monitor |
| Azure ALT REST 참조 | https://learn.microsoft.com/en-us/rest/api/loadtesting/dataplane/ | Microsoft | 2026-05-05 갱신 | 2026-06-26 | 공식 | 중간 | dataplane defaultMoniker = **`2026-04-01`(GA)** |
| Azure ALT API status | https://learn.microsoft.com/en-us/rest/api/apptesting/loadtest/load-testing-api-status | Microsoft | 2026-06-11 갱신 | 2026-06-26 | 공식 | 중간 | 상태 페이지는 아직 `2022-11-01` baseline 표기(지연) |
| Azure ALT Locust | https://learn.microsoft.com/en-us/azure/app-testing/load-testing/quickstart-create-run-load-test-with-locust | Microsoft | 미표시 | 2026-06-26 | 공식 | 낮음 | Locust 2.33.2 엔진 지원 |
| Azure ALT JMeter | https://learn.microsoft.com/en-us/azure/app-testing/load-testing/how-to-create-and-run-load-test-with-jmeter-script | Microsoft | 2026-06-11 갱신 | 2026-06-26 | 공식 | 낮음 | 기존 `.jmx` 업로드, JMeter 5.6.3 |
| Azure ALT 멀티리전 | https://learn.microsoft.com/en-us/azure/app-testing/load-testing/how-to-generate-load-from-multiple-regions | Microsoft | 미표시 | 2026-06-26 | 공식 | 낮음 | 최대 5개 리전, 공개 endpoint 한정 |
| Azure TF 리소스 | https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/load_test | HashiCorp | 미표시 | 2026-06-26 | 공식 | 중간 | `azurerm_load_test`(LoadTest 리소스만, 테스트 정의 아님) |
| Azure az CLI | https://learn.microsoft.com/en-us/cli/azure/load | Microsoft | 미표시 | 2026-06-26 | 공식 | 낮음 | `az load` 확장(자동 설치, az ≥2.66) |
| Azure 가격 | https://azure.microsoft.com/en-us/pricing/details/app-testing/ | Microsoft | 미표시 | 2026-06-26 | 공식 | 높음 | 과금 단위 VUH(=VU×시간×엔진 수) |
| Gatling MCP server | https://docs.gatling.io/ai-for-scripting/mcp-server/ | Gatling | 미표시 | 2026-06-26 | 공식 | 중간 | **read-only 4 `list_*` tools**, deploy는 Skills |
| Gatling AI assistant | https://docs.gatling.io/ai-for-scripting/assistant/vscode/overview/ | Gatling | 미표시 | 2026-06-26 | 공식 | 중간 | Java/Kotlin/Scala/JS/TS, 사용자 LLM 키 필요 |
| Gatling Skills | https://docs.gatling.io/ai-for-scripting/skills/ | Gatling | 2026-04-17 | 2026-06-26 | 공식 | 중간 | `gatling-build-tools`(deploy/start), JMeter/LoadRunner 변환 |
| Gatling API | https://docs.gatling.io/reference/api/ | Gatling | 미표시 | 2026-06-26 | 공식 | 중간 | public API로 run 트리거/결과·메트릭 조회 |
| Gatling Private Locations | https://docs.gatling.io/reference/deploy/private-locations/introduction/ | Gatling | 미표시 | 2026-06-26 | 공식 | 낮음 | Control Plane(아웃바운드 poll), AWS/Azure/GCP/K8s/on-prem |
| Gatling Terraform 모듈 | https://docs.gatling.io/reference/deploy/private-locations/whats-new/terraform-official-registry/ | Gatling | 미표시 | 2026-06-26 | 공식 | 중간 | Registry 모듈(provider 아님), Control Plane 부트스트랩 |
| Gatling TF 모듈(레지스트리) | https://registry.terraform.io/modules/gatling/control-plane/aws | Gatling | 2026-06-18 | 2026-06-26 | 공식 | 중간 | aws v0.2.0(`verified:false`), azure/gcp도 존재 |
| Gatling 관측성 | https://docs.gatling.io/integrations/observability-tools/ | Gatling | 미표시 | 2026-06-26 | 공식 | 낮음 | Datadog/Dynatrace/InfluxDB/NewRelic/OTel(Prometheus는 OTel 경유) |
| Gatling OSS vs Enterprise | https://gatling.io/community-vs-enterprise | Gatling | 미표시 | 2026-06-26 | 공식 | 중간 | 분산/Private Location/대시보드/API/MCP=Enterprise 전용 |
| BlazeMeter Taurus 실행기 | https://gettaurus.org/docs/ExecutionSettings/ | BlazeMeter/Perforce | 미표시 | 2026-06-26 | 공식 | 낮음 | JMeter/Gatling/k6/Locust/Selenium/Playwright 등 실행기 |
| BlazeMeter MCP | https://help.blazemeter.com/docs/guide/integrations-blazemeter-mcp-server.html | BlazeMeter/Perforce | 미표시 | 2026-06-26 | 공식 | 중간 | MCP로 create/run/report/workspace 관리 |
| BlazeMeter AI | https://www.blazemeter.com/solutions/ai-software-testing | BlazeMeter/Perforce | 미표시 | 2026-06-26 | 공식 | 중간 | AI Script Assistant/이상탐지, "AI 성능테스트(coming soon)" |
| BlazeMeter 과금 | https://help.blazemeter.com/docs/guide/usage-billing-blazemeter-credit-types-and-how-they-are-charged.html | BlazeMeter/Perforce | 미표시 | 2026-06-26 | 공식 | 높음 | 크레딧(VU/VUH/Tests), GUI/브라우저 100×VUH |
| Artillery test script | https://www.artillery.io/docs/reference/test-script | Artillery | 미표시 | 2026-06-26 | 공식 | 낮음 | YAML config+scenarios, phases(arrivalRate/rampTo) |
| Artillery Pro deprecation | https://www.artillery.io/docs/artillery-pro/installing-artillery-pro | Artillery | 미표시 | 2026-06-26 | 공식 | 낮음 | "Pro deprecated June 2024", 분산 기능 main으로 이동 |
| Artillery scale | https://www.artillery.io/docs/load-testing-at-scale | Artillery | 미표시 | 2026-06-26 | 공식 | 낮음 | Fargate/ACI/Lambda, CLI가 리소스 생성·삭제 |
| Artillery Playwright | https://www.artillery.io/docs/reference/engines/playwright | Artillery | 미표시 | 2026-06-26 | 공식 | 낮음 | 브라우저 레벨 엔진, Web Vitals 캡처 |
| Artillery OTel | https://www.artillery.io/docs/reference/extensions/publish-metrics/opentelemetry | Artillery | 미표시 | 2026-06-26 | 공식 | 낮음 | metrics+traces export |
| Locust 작성 | https://docs.locust.io/en/stable/writing-a-locustfile.html | Locust | 2.44.x | 2026-06-26 | 공식 | 낮음 | `HttpUser`/`@task`/`FastHttpUser` |
| Locust 분산 | https://docs.locust.io/en/stable/running-distributed.html | Locust | 2.44.4 | 2026-06-26 | 공식 | 낮음 | master/worker, GIL→core당 worker 1개 |
| Locust telemetry | https://docs.locust.io/en/stable/telemetry.html | Locust | 2.43.x | 2026-06-26 | 공식 | 중간 | OTel **traces+metrics**(logs 아님), `requests`만 auto-instrument |
| GCP Cloud Run 부하 | https://docs.cloud.google.com/run/docs/about-load-testing | Google | 2026-06-18 갱신 | 2026-06-26 | 공식 | 낮음 | 관리형 부하 제품 없음, JMeter/Selenium 패턴 안내 |
| GCP GKE 분산 부하 | https://docs.cloud.google.com/architecture/distributed-load-testing-using-gke | Google | 미표시 | 2026-06-26 | 공식 | 중간 | GKE에 분산 Locust master/worker 배포 패턴 |
| JMeter 시작 | https://jmeter.apache.org/usermanual/get-started.html | Apache | 미표시 | 2026-06-26 | 공식 | 낮음 | GUI는 작성만, 부하는 CLI(`-n`), `.jmx` XML |
| JMeter 원격 | https://jmeter.apache.org/usermanual/remote-test.html | Apache | 미표시 | 2026-06-26 | 공식 | 중간 | RMI controller/server, 버전·Java·SSL keystore 주의 |
| JMeter 실시간 결과 | https://jmeter.apache.org/usermanual/realtime-results.html | Apache | 미표시 | 2026-06-26 | 공식 | 중간 | InfluxDB Backend Listener→Grafana |

> 주: 위 표의 일부 registry.terraform.io 페이지는 JS 렌더링이라 본문 fetch 시 GitHub raw / Registry API로 교차 확인했다(리소스명·모듈 버전은 확정, "GA 배지" 명시는 부재 → "비-베타 정식 리소스"로 판정).

### 2.1 보강 출처 레지스트리 (Firebase/BaaS 부하 대상 · 실시간·프로토콜 도구 · 템플릿화)

> 2026-06-26 신규 검증분. 컬럼 정의는 §2와 동일. Firebase 문서는 대부분 "Last updated 2026-06-22 UTC", GCP 문서는 2026-06-18/19 표기. `cloud.google.com/*/docs/*`는 현재 `docs.cloud.google.com/*/docs/*`로 301 이전(동일 공식 콘텐츠, 호스트만 변경).

| 주제 | source_url | vendor | published/updated | fetched_at | trust_level | freshness_risk | 핵심 근거 |
|---|---|---|---|---|---|---|---|
| Firestore best practices | https://firebase.google.com/docs/firestore/best-practices | Google | 2026-06-22 | 2026-06-26 | 공식 | 낮음 | **"500/50/5" 권고**(신규 컬렉션 500 ops/s에서 시작→5분마다 50%↑, 강제 아닌 권고), "부하 테스트가 성능 특성화 최선" |
| Firestore real-time at scale | https://firebase.google.com/docs/firestore/real-time_queries_at_scale | Google | 2026-06-22 | 2026-06-26 | 공식 | 낮음 | reverse query matcher·changelog 서버 샤딩, "앱을 부하 테스트하라"(동시 리스너 **수치 캡 없음**) |
| Firestore RPC ref(Listen) | https://docs.cloud.google.com/firestore/docs/reference/rpc/google.firestore.v1 | Google | 미표시 | 2026-06-26 | 공식 | 낮음 | **"Listen … only available via gRPC or WebChannel (not REST)"**; 스트리밍 Write 동일 |
| Firestore REST ref | https://docs.cloud.google.com/firestore/docs/reference/rest | Google | 2026-03-20 | 2026-06-26 | 공식 | 낮음 | `firestore.googleapis.com` v1, get/list/runQuery/batchGet 등(실시간 Listen은 제외) |
| Firestore pricing | https://firebase.google.com/docs/firestore/pricing | Google | 2026-06-22 | 2026-06-26 | 공식 | 높음 | read/write/delete 건수+인덱스+저장+egress, "월 10GB egress 무료"(CPU-시간 과금 없음) |
| Firestore quotas | https://firebase.google.com/docs/firestore/quotas | Google | 2026-06-22 | 2026-06-26 | 공식 | 중간 | 표준 한도表에 **동시 연결·스냅샷 리스너 수치 캡 부재**("1M" 미확인) |
| Firestore rules conditions | https://firebase.google.com/docs/firestore/security/rules-conditions | Google | 2026-06-22 | 2026-06-26 | 공식 | 중간 | 접근 호출 한도 **10(단일·쿼리)/20(다중·트랜잭션·배치)**, `get()/exists()`는 거부돼도 read 과금 |
| RTDB limits | https://firebase.google.com/docs/database/usage/limits | Google | 2026-06-22 | 2026-06-26 | 공식 | 중간 | **동시 연결 200,000(Blaze)/쓰기 1,000·s/총 쓰기 64MB·분/동시 응답 ~100,000·s** |
| RTDB monitor usage | https://firebase.google.com/docs/database/usage/monitor-usage | Google | 2026-06-22 | 2026-06-26 | 공식 | 낮음 | 실시간 연결=**WebSocket/long polling/SSE**, "RESTful 요청 미포함" |
| RTDB profile | https://firebase.google.com/docs/database/usage/profile | Google | 2026-06-22 | 2026-06-26 | 공식 | 낮음 | "**WebSocket overhead**" 명시(실시간 프로토콜 오버헤드) |
| RTDB billing | https://firebase.google.com/docs/database/usage/billing | Google | 2026-06-22 | 2026-06-26 | 공식 | 높음 | 저장 **$5/GB·월**, 무료 1GB 저장+10GB·월 다운로드, "보안규칙 거부 트래픽도 과금" |
| RTDB sharding | https://firebase.google.com/docs/database/usage/sharding | Google | 2026-06-22 | 2026-06-26 | 공식 | 낮음 | 200k 연결/1,000 쓰기 초과 시 샤딩, 프로젝트당 **최대 1,000 인스턴스** |
| Auth limits | https://firebase.google.com/docs/auth/limits | Google | 2026-06-22 | 2026-06-26 | 공식 | 중간 | **IP당 신규계정 100/시간**, 프로젝트 **1,000 req/s·1,000만/일**, "경고 없이 anti-abuse 차단 가능" |
| Identity Platform quotas | https://docs.cloud.google.com/identity-platform/quotas | Google | 2026-06-18 | 2026-06-26 | 공식 | 중간 | Auth 쿼터 교차검증(GetAccountInfo 50만/분, 커스텀 토큰 사인인 4.5만/분, 서비스계정 500 req/s) |
| Identity Platform pricing | https://cloud.google.com/identity-platform/pricing | Google | 미표시 | 2026-06-26 | 공식 | 높음 | **MAU 과금, 5만 무료**, Spark는 DAU 3,000 상한($/MAU 단가는 미확보) |
| Functions version comparison | https://firebase.google.com/docs/functions/version-comparison | Google | 2026-06-22 | 2026-06-26 | 공식 | 낮음 | "**2nd gen은 Cloud Run + Eventarc 기반**, Cloud Run 서비스로 배포" |
| Functions manage | https://firebase.google.com/docs/functions/manage-functions | Google | 미표시 | 2026-06-26 | 공식 | 중간 | 동시성 **기본 80(1~1,000 설정)**, 요청 기반 오토스케일(0까지), min/max instances |
| Cloud Run load testing | https://docs.cloud.google.com/run/docs/about-load-testing | Google | 2026-06-18 | 2026-06-26 | 공식 | 낮음 | "**JMeter 같은 test harness**로 부하 생성", max instances로 수동 스케일 근사, **선형 확장 확인** 권장 |
| Cloud Run concurrency | https://docs.cloud.google.com/run/docs/about-concurrency | Google | 2026-06-18 | 2026-06-26 | 공식 | 낮음 | 기본 **80×vCPU**(콘솔 기본 80), 최대 1,000 |
| Cloud Run autoscaling | https://docs.cloud.google.com/run/docs/about-instance-autoscaling | Google | 2026-06-18 | 2026-06-26 | 공식 | 낮음 | 유휴 인스턴스 최대 15분 유지, max 초과 요청 대기 후 **429** |
| App Check overview | https://firebase.google.com/docs/app-check | Google | 2026-06-22 | 2026-06-26 | 공식 | 중간 | "**유효한 App Check 토큰 동반 요청만 수락**", 미인증·미승인 플랫폼 거부 |
| App Check enforcement | https://firebase.google.com/docs/app-check/enable-enforcement | Google | 2026-06-22 | 2026-06-26 | 공식 | 중간 | 제품별 적용(Firestore/RTDB/Storage/Auth 등), 적용 후 미검증 거부, **~15분 전파** |
| App Check debug provider | https://firebase.google.com/docs/app-check/web/debug-provider | Google | 2026-06-22 | 2026-06-26 | 공식 | 중간 | **CI/자동 테스트는 디버그 토큰**으로 미인증 환경 허용(비밀 유지 필수) |
| Firebase emulator suite | https://firebase.google.com/docs/emulator-suite | Google | 2026-06-22 | 2026-06-26 | 공식 | 낮음 | "정확성용이지 성능/보안용 아님" → **부하 테스트 부적합** |
| k6 websockets module | https://grafana.com/docs/k6/latest/javascript-api/k6-websockets/ | Grafana | 미표시 | 2026-06-26 | 공식 | 낮음 | `k6/websockets` **stable/core**, "신규 테스트는 k6/websockets 권장" |
| k6 experimental websockets | https://grafana.com/docs/k6/latest/javascript-api/k6-experimental/websockets/ | Grafana | 미표시 | 2026-06-26 | 공식 | 중간 | `k6/experimental/websockets` **deprecated**(→`k6/websockets`), `k6/ws`는 레거시 콜백 API |
| k6 gRPC | https://grafana.com/docs/k6/latest/using-k6/protocols/grpc/ | Grafana | 미표시 | 2026-06-26 | 공식 | 낮음 | `k6/net/grpc` **unary+streaming**(v0.49+), HTTP/2 |
| k6 1.0 release | https://grafana.com/blog/grafana-k6-1-0-release/ | Grafana | 2025-05-08 | 2026-06-26 | 공식 | 낮음 | "**k6/browser, k6/net/grpc, k6/crypto now stable**" |
| k6 browser graduate(v0.52) | https://grafana.com/docs/k6/latest/using-k6-browser/migrating-to-k6-v0-52/ | Grafana | 미표시 | 2026-06-26 | 공식 | 낮음 | experimental 졸업 → `k6/browser`(CDP로 **실 Chromium** 구동) |
| k6 load testing websites | https://grafana.com/docs/k6/latest/testing-guides/load-testing-websites/ | Grafana | 미표시 | 2026-06-26 | 공식 | 낮음 | 브라우저 자동화 "**자원 집약 → 고부하 부적합**", 프로토콜 다수+브라우저 소수 권장 |
| k6 extensions explore | https://grafana.com/docs/k6/latest/extensions/explore/ | Grafana | 미표시 | 2026-06-26 | 공식 | 중간 | **SSE는 1차 모듈 없음**, 커뮤니티 확장 `xk6-sse` |
| k6 environment variables | https://grafana.com/docs/k6/latest/using-k6/environment-variables/ | Grafana | 미표시 | 2026-06-26 | 공식 | 낮음 | `__ENV`+`-e/--env`, "여러 스크립트 대신 환경변수로 스크립트 일부를 가변화" |
| Artillery websocket engine | https://www.artillery.io/docs/reference/engines/websocket | Artillery | 미표시 | 2026-06-26 | 공식 | 낮음 | `engine: ws` **내장**(connect/send/think/loop; connect는 v2) |
| Artillery socketio engine | https://www.artillery.io/docs/reference/engines/socketio | Artillery | 미표시 | 2026-06-26 | 공식 | 낮음 | `engine: socketio` **내장**(실제 Socket.IO 클라이언트 구동) |
| Artillery engines index | https://www.artillery.io/docs/reference/engines | Artillery | 미표시 | 2026-06-26 | 공식 | 낮음 | HTTP/WebSocket/Socket.io **out of the box**, HLS/Kinesis/Kafka는 플러그인 |
| Gatling websocket | https://docs.gatling.io/reference/script/websocket/ | Gatling | 미표시 | 2026-06-26 | 공식 | 낮음 | `ws`는 HTTP SDK 확장(**OSS 코어**, Enterprise 게이팅 없음) |
| Gatling sse | https://docs.gatling.io/reference/script/sse/ | Gatling | 미표시 | 2026-06-26 | 공식 | 낮음 | `sse`는 HTTP SDK 확장(**OSS 코어**), `Accept: text/event-stream` 자동 |
| Gatling gRPC | https://docs.gatling.io/reference/script/grpc/setup/ | Gatling | v3.15.1 | 2026-06-26 | 공식 | 중간 | gRPC **플러그인**, OSS는 **5 VU·5분 제한**/Enterprise 무제한, server/client/bidi streaming |
| Locust other systems | https://docs.locust.io/en/stable/testing-other-systems.html | Locust | v2.44.0 | 2026-06-26 | 공식 | 중간 | "**HTTP/HTTPS만 내장**", WS/gRPC는 라이브러리 래핑+request 이벤트 발생(custom client) |
| Fortio | https://fortio.org/ | Fortio | v1.75.2 | 2026-06-26 | 공식 | 낮음 | HTTP/1.1·2(h2/h2c)·**gRPC**·TCP/UDP, Istio 출신 |
| ghz | https://ghz.sh/docs/intro | ghz | 미표시 | 2026-06-26 | 공식 | 중간 | **gRPC 전용** CLI, proto/protoset/reflection로 스텁 불필요 |
| Thor | https://github.com/observing/thor | observing | 미표시 | 2026-06-26 | 공식(repo) | 중간 | **WebSocket 부하 생성기**(Node, raw 프레임; 최근 활동 낮음) |

> 신뢰성 주의(이번 보강): **(1)** Firestore "동시 연결 1,000,000"은 공식 쿼터 페이지에서 확인 불가 → **인용 금지**(200,000은 RTDB 수치). **(2)** "GCP는 1차 관리형 부하 제품 없음"은 명시 인용이 아니라 Cloud Run 가이드가 외부 하니스(JMeter 등)를 권장하는 데서 **추론**(기존 §3.9와 일관). **(3)** Firestore 보안규칙 "쿼리=20 호출"은 **오류** — 쿼리는 단일 문서와 함께 **10 버킷**, 20은 다중 문서·트랜잭션·배치. **(4)** RTDB 다운로드 $/GB, Identity Platform $/MAU 단가는 본 보강에서 verbatim 미확보 → 가격 페이지 재확인(§11). 템플릿 파라미터화의 Artillery 근거는 §2의 "Artillery test script" 행(variables/payload/phases/environments)을 재사용.

---

## 3. 도구별 상세 카드

### 3.1 Grafana k6 / Grafana Cloud k6

- **개요**: Go 엔진 + JavaScript/TypeScript 테스트(scripts-as-code). OSS CLI(`k6`)는 무료 로컬 실행, Grafana Cloud k6는 분산 실행·결과 저장·대시보드·협업·스케줄 제공. k6 2.0이 2026-05-12 릴리스되며 AI 워크플로(`k6 x agent`/`k6 x mcp`)가 들어왔다.
- **테스트 작성 모델**: `scenarios` 안에서 `executors`(`constant-vus`, `ramping-vus`, `constant-arrival-rate`, `ramping-arrival-rate`, `shared-iterations`, `per-vu-iterations`, `externally-controlled`)로 부하 모델 구성. arrival-rate executor는 open-model이라 RPS 기준 SLO에 적합. **`thresholds`로 pass/fail을 코드화**하고, breach 시 **k6가 non-zero exit code로 종료** → CI/CD 게이트로 직접 사용 가능. (threshold 실패 종료코드는 **99**(`ThresholdsHaveFailed`, k6 소스 `errext/exitcodes/codes.go`에 공식 정의); 사용자 문서에는 "non-zero"로만 표기.)
- **AI 친화성**: JS/TS + 공식 TypeScript 타입 + MCP의 `validate_script`/`run_script`/`get_documentation`로 **LLM 생성→자가검증 루프**가 자연스럽다. `k6 x agent`는 **k6 2.0 정식 릴리스에 포함**(별도 "stable" 배지는 문서에 없음): Claude Code/Cursor/Copilot/Codex CLI/OpenCode/Cline에 SKILL.md+MCP를 스캐폴딩. **MCP 서버(`k6 x mcp`)는 공식 문구상 "currently in preview"** — production 단정 금지. MCP tools 6개(`info`, `validate_script`, `run_script`(VU≤50·duration≤5m 제한), `list_sections`, `get_documentation`, `search_terraform`), prompts 2개(`generate_script`, `convert_playwright_script`), resources(best practices, TS 타입).
- **IaC 지원(★)**: 공식 `grafana/grafana` provider에 k6 리소스 **6종 실재** — `grafana_k6_installation`, `grafana_k6_project`, `grafana_k6_load_test`, `grafana_k6_project_limits`, `grafana_k6_project_allowed_load_zones`, `grafana_k6_schedule`. beta/experimental 라벨 없는 정식 문서화 리소스(단 명시적 "GA 배지"는 없음 → "비-베타 정식 리소스"). 테스트 정의(`grafana_k6_load_test`)와 스케줄(cron)까지 Terraform으로 선언 가능.
- **자동화 API**: 현재 Grafana Cloud k6 REST API는 **v6**(`api.k6.io/cloud/v6`, Bearer + `X-Stack-Id`). **레거시 API는 deprecated, 제거 예정**. 스크립트의 `options.ext.loadimpact`는 k6 v2에서 제거 → `options.cloud` 사용. CLI: `k6 run`(로컬), `k6 cloud run`(클라우드).
- **분산·클라우드**: Public load zone **21개 멀티리전**(Seoul/Tokyo/Singapore/Frankfurt 등), `options.cloud.distribution`로 리전별 비율 분배. **Private Load Zone**: 고객 k8s 클러스터 내부에 부하 생성기를 띄우고 메트릭만 Grafana Cloud로 push(내부망/비공개 서비스 테스트). 프로젝트별 PLZ 권한 분리 지원.
- **관측성**: 실시간 output 다수 — Prometheus remote write(플래그 `--out experimental-prometheus-rw`, **experimental 라벨 유지**), OpenTelemetry(`--out opentelemetry`), InfluxDB/Datadog/CloudWatch/New Relic/Dynatrace/Elasticsearch/Kafka/StatsD 등. 공식 Grafana 대시보드 "k6 Prometheus"(ID 19665).
- **비용·제한**: 과금 단위 **VUh(Virtual User Hours)**. Free forever에 **월 500 VUh**. (단가 $/VUh는 변동성 커서 인용 비권장.) OSS는 전면 무료(로컬). MCP `run_script`는 VU 50·5분 상한.
- **장점**: scripts-as-code + 명시적 SLO(threshold) + 실재 Terraform 리소스 + 풍부한 관측성 + Private Load Zone + AI 워크플로가 한 그림으로 맞물림. 개인 MVP~실무 설득력 균형이 가장 좋음.
- **단점·리스크**: MCP는 preview(권한/기능 변동 가능). Prometheus RW 플래그가 여전히 experimental 명칭(graduation 시 플래그명 변경 가능). Cloud 사용 시 토큰/RBAC/스택 ID 취급 필요. threshold 실패 종료코드=99(소스 정의). VUh 단가는 볼륨 의존.
- **출처**: 위 레지스트리 k6 항목 일체.

### 3.2 Gatling Enterprise / Gatling Cloud

- **개요**: Open-Source(Community) Edition(코어 SDK + 로컬 단일 머신 실행 + 정적 HTML 리포트, 무료) vs **Enterprise Edition**(분산/멀티클라우드, Private Location, 대시보드/트렌드/비교, RBAC/SSO, API, MCP, AI). 스크립트는 동일 SDK라 이식성 있음.
- **테스트 작성 모델**: DSL 언어 **Java/Kotlin/Scala/JavaScript/TypeScript**. 단 **JS/TS SDK는 현재 HTTP 프로토콜만** 커버(Java/Kotlin/Scala가 풀 기능). Simulation/Scenario/Injection profile/Assertions/Checks/Feeders가 1급 개념.
- **AI 친화성**: (1) **Gatling AI Assistant** IDE 확장(VS Code/Cursor/Windsurf/Antigravity) — 프롬프트로 시뮬레이션 생성/설명/리팩토링, **사용자 본인 LLM API 키 필요**(OpenAI 권장, Anthropic/Azure OpenAI 가능; 키는 로컬 저장). (2) **Claude Skills**: `gatling-bootstrap-project`, `gatling-build-tools`, `gatling-configuration-as-code`, 변환 스킬 `gatling-convert-from-jmeter`/`gatling-convert-from-loadrunner`. (3) AI for analysis(Run Summary/Comparison/Trends). 명시적 GA/preview 라벨은 없고 "AI-Generated Code Notice(항상 검증)" 경고만.
- **MCP server(★)**: **실재하고 read-only**. 노출 도구는 **4개 `list_*`**(`list_gatling_enterprise_teams`/`packages`/`tests`/`locations`)뿐 — **MCP 자체는 deploy/start/stop을 하지 않는다**. deploy/run은 **별도 Skills(`gatling-build-tools`)**가 수행. 마케팅의 "coding agent에서 배포"는 MCP+Skills 번들을 의미. NPM `@gatling.io/gatling-mcp-server` / Docker 배포, stdio 로컬 실행, Enterprise 계정+토큰 필요. (런칭 ~2026-03, 마케팅 기반이라 날짜는 중간 신뢰.)
- **IaC 지원**: 공식 **Terraform _모듈_(provider 아님)** — `gatling/control-plane/aws`(v0.2.0, 2026-06-18), `azure`(v0.0.2), `gcp`(v0.0.1), 모두 `verified:false`. **Control Plane 인프라 부트스트랩용**이지 테스트 리소스 선언용이 아님. Helm 차트도 존재. `gatling` Terraform provider는 없음(404).
- **자동화 API**: public REST API로 run 트리거 + 결과/메트릭 조회(Authorization 토큰). 공식 페이지는 stub 수준이라 엔드포인트 전수는 미문서화(재확인 필요).
- **분산·클라우드**: **Private Locations** — Control Plane 에이전트가 아웃바운드 poll만(인바운드 연결/자격증명 공유 없음). AWS/Azure/GCP/Kubernetes(+OpenShift)/Dedicated. **Enterprise 전용**.
- **관측성**: Datadog/Dynatrace/InfluxDB/New Relic/OpenTelemetry. **Prometheus는 OTel 경유**, **Grafana는 1급 통합 아님**(InfluxDB/Prometheus-over-OTel 간접).
- **비용·제한**: 분산/Private Location/대시보드/API/MCP/AI/IaC 모두 Enterprise 전용. 공개 가격 미게시(달러 인용 금지).
- **장점**: AI 작성(5개 언어)+JMeter/LoadRunner 변환 스킬+Private Location(아웃바운드 only)이 엔터프라이즈에 강함. high-throughput에 강한 엔진.
- **단점·리스크**: 핵심 운영/리포트/API/AI/MCP가 상용 종속. JS/TS는 HTTP 한정. MCP는 read-only(실행은 Skills 의존). 개인 MVP엔 과함. Grafana 직접 통합 부재.
- **출처**: 레지스트리 Gatling 항목.

### 3.3 AWS Distributed Load Testing on AWS (DLT)

- **개요**: AWS 공식 "솔루션". 현재 **v4.1.4(CHANGELOG 2026-06-25)**. 웹 콘솔에서 시나리오를 만들면 컨테이너 부하 생성기가 여러 리전에서 실행. **MCP/CLI는 v4.1.0(2026-05-13)**에서 도입.
- **테스트 작성 모델**: 부하 컨테이너가 **Taurus** 프레임워크로 **JMeter/K6/Locust/단순 HTTP**를 실행. 시나리오는 콘솔/REST/CLI로 정의. 단순 HTTP는 스크립트 없이 JMeter plan으로 변환.
- **AI 친화성**: **MCP read-only 7 tools**(`list_scenarios`/`get_scenario_details`/`list_test_runs`/`get_test_run`/`get_latest_test_run`/`get_baseline_test_run`/`get_test_run_artifacts`). Bedrock AgentCore Gateway + Cognito(OAuth2) + Lambda가 기존 DLT API 호출. **"모든 MCP 도구는 read-only, 시나리오/설정 수정 불가"** 공식 명시 → AI는 과거 실행 분석엔 적합하나 MCP만으로 실행/변경 불가.
- **IaC 지원**: **CloudFormation으로 배포, 소스는 CDK(v2) 기반**. **공식 Terraform 모듈/경로 없음**. Terraform-first로 보이려면 계정/네트워크/보조 리소스 또는 wrapper를 별도 설계해야 함.
- **자동화 API**: REST API + **TypeScript CLI**(Cognito 인증→REST 직접 호출, CI/CD 스크립팅). headless 콘솔 호스팅 옵션도 존재.
- **분산·클라우드**: **Amazon ECS on Fargate** 태스크가 선택 리전마다 실행(멀티리전). EventBridge Scheduler로 즉시/예약/cron 반복. 동시 테스트(테스트 ID별 동시성 가드) 지원.
- **관측성**: 결과는 S3, 로그·대시보드는 CloudWatch, 시나리오/집계는 DynamoDB. 선택적 라이브 스트림(CloudWatch→Lambda→IoT Core→콘솔).
- **비용·제한**: AWS 리소스 비용(ECS/Fargate, S3, DynamoDB, 데이터 전송)에 의존. Gatling/Playwright 엔진 미지원.
- **장점**: AWS-native, JMeter/K6/Locust 동시 수용, 멀티리전/스케줄/동시 실행, read-only MCP로 분석 AI 연계.
- **단점·리스크**: **CloudFormation/CDK-first**라 Terraform 통합은 wrapper 필요. MCP는 실행 불가(분석 전용). 번들된 third-party 엔진 버전은 릴리스 시점 implementation guide에서 재확인 필요.
- **출처**: 레지스트리 AWS DLT 항목.

### 3.4 Azure Load Testing (Azure App Testing hub)

- **개요**: **Azure App Testing** 우산 아래의 성능 테스트 서비스(다른 한 축은 Playwright Workspaces=기능/E2E). 관리형 분산 실행 + Azure Monitor 통합.
- **테스트 작성 모델**: **JMeter(JMX, 5.6.3) 또는 Locust(2.33.2)만 지원** — 공식 문구상 "JMeter/Locust 외 프레임워크 미지원". 기존 `.jmx` 재사용 강점. `2026-04-01` 스키마에 `BROWSER_RECORDING`/AI `TEST_PLAN_RECOMMENDATIONS` 파일 타입 존재(브라우저 레코딩→JMeter/Locust plan 제안)이나, 이는 **Playwright 부하 엔진이 아님**.
- **AI 친화성**: 직접적 AI 스크립트 어시스턴트는 약함(브라우저 레코딩 기반 plan 추천 정도). 자연어→JMeter 생성은 외부 LLM에 의존.
- **IaC 지원**: AzureRM provider에 **`azurerm_load_test` 리소스 실재**(`Microsoft.LoadTestService/loadTests`). 단 **LoadTest 리소스 자체만 프로비저닝**, 개별 테스트/JMX 정의는 별도(REST/CLI). 동명 data source도 있음.
- **자동화 API**: **dataplane REST API `2026-04-01`(GA 확정, `-preview` 없음)**. 단 api-lifecycle 상태 페이지는 갱신 지연으로 아직 `2022-11-01`을 baseline 표기 → "latest"는 reference defaultMoniker 기준. **`az load` CLI 확장**(자동 설치, az ≥2.66; `az load`/`az load test`/`az load test-run`). Test Profile 계열은 별도 `-preview` 트랙(2024-07~2025-11).
- **분산·클라우드**: 엔진 인스턴스 수로 스케일아웃, **최대 5개 리전**(단 멀티리전은 공개 endpoint 한정).
- **관측성**: 클라이언트 메트릭 + **Azure Monitor/Application Insights/Container insights** 서버사이드 메트릭, 실행 이력 시각 비교(회귀 탐지), fail criteria/auto-stop.
- **비용·제한**: 과금 단위 **VUH**(VU×시간×엔진 수), pay-as-you-go, 월 VUH 한도 설정 가능. (브라우저 분(分)은 Playwright/Workspaces 쪽.)
- **장점**: Azure-native, 기존 JMeter 자산 재사용, Azure Monitor 일체화, REST/CLI/Terraform(리소스)·회귀 비교가 균형.
- **단점·리스크**: **부하 엔진이 JMeter/Locust로 한정**(이전 리서치의 Playwright 부하 주장은 오류). Terraform은 리소스만, 테스트 정의는 비-IaC 경로. AI 작성 직접성 약함. 멀티리전은 공개 endpoint 한정.
- **출처**: 레지스트리 Azure 항목.

### 3.5 Artillery

- **개요**: OSS(코어). YAML + JS/TS. **분산 실행이 메인 배포에 내장**(별도 상용 Pro 불필요).
- **테스트 작성 모델**: top-level `config` + `scenarios`, load `phases`(`duration`/`arrivalRate`/`rampTo`), JS/TS processors, 엔진 `http`/`websocket`/**`playwright`(브라우저 레벨)**.
- **AI 친화성**: YAML+JS/TS는 LLM 생성에 매우 친화적(단 이는 해석적 평가이며 공식 문구는 아님). 공식 AI 어시스턴트/MCP는 확인되지 않음.
- **IaC 지원**: **공식 네이티브 Terraform provider 없음**. 분산 실행 시 **CLI가 클라우드 리소스를 즉석 생성·삭제**(`run-fargate`/`run-aci`/`run-lambda`). → Terraform과 권한 경계 설계 필요.
- **자동화 API**: CLI 중심(`artillery run`, `run-fargate`, `run-aci`, `run-lambda`). 리포트/플러그인 구성.
- **분산·클라우드**: **AWS Fargate / Azure ACI / AWS Lambda**가 메인 배포에 포함. "장기 인프라 0, CLI가 ECS 클러스터/S3/SQS/Lambda를 즉석 생성하고 종료 시 제거" 공식 명시.
- **관측성**: `publish-metrics` 플러그인 — **OpenTelemetry(metrics+traces)**, Datadog, CloudWatch(metrics+traces), Prometheus(Pushgateway).
- **비용·제한**: OSS라 도구 비용 0(클라우드 실행 시 해당 클라우드 비용). 라이선스는 docs 사이트에 미명시(저장소 LICENSE 기준 MPL-2.0 — 공식 docs 도메인 밖이라 중간 신뢰).
- **장점**: 작성 쉬움(YAML/JS), 분산 실행 내장, Pro 종속 제거됨, OTel/멀티 백엔드 관측성, 브라우저 엔진(Playwright)까지. 개인 MVP에 강함.
- **단점·리스크**: CLI가 클라우드 리소스를 직접 만들고 지움 → 승인 게이트/권한 경계 필수. 네이티브 IaC 없음. 엔터프라이즈 거버넌스(대시보드/RBAC)는 상대적으로 약함.
- **출처**: 레지스트리 Artillery 항목.

### 3.6 Locust

- **개요**: Python 기반 OSS(MIT). 코드로 사용자 행동을 표현. 2.44.x stable.
- **테스트 작성 모델**: `User` 상속, HTTP는 `HttpUser`(`client`=`requests.Session` 래퍼), 고성능은 `FastHttpUser`(geventhttpclient, drop-in). 태스크 `@task`, 커스텀 부하 곡선 `LoadTestShape`.
- **AI 친화성**: Python scripts-as-code라 LLM 생성에 우호적(해석적). 공식 AI/MCP는 없음.
- **IaC 지원**: 네이티브 Terraform 없음 → 컨테이너/Kubernetes wrapping(예: GKE 분산 패턴)으로 IaC화.
- **자동화 API**: CLI(`--processes`로 master+worker), Web UI, CSV/stats export, headless 모드.
- **분산·클라우드**: master/worker. **GIL 때문에 core당 worker 1개** 권장(`fork()` 사용, Windows 제외).
- **관측성**: **OpenTelemetry로 traces+metrics export(logs 아님)**, `--otel` 플래그, **`requests` 라이브러리만 auto-instrument**(다른 라이브러리는 수동). Prometheus는 커뮤니티 `locust-exporter` 또는 OTel 경유(코어 1급 기능 아님).
- **비용·제한**: OSS 무료. 단일 프로세스 CPU 한계 → core당 worker 분산 필요.
- **장점**: Python 로직 자유도, 분산, OTel, 낮은 lock-in. 복잡한 사용자 여정/커스텀 로직에 강함.
- **단점·리스크**: 분산/관측/권한/cleanup을 직접 설계. GIL로 인한 프로세스 분산 운영 부담. Prometheus 1급 미지원.
- **출처**: 레지스트리 Locust 항목.

### 3.7 Apache JMeter

- **개요**: 100% Java(Java 8+) 성숙 도구. GUI는 작성용, 부하는 CLI(non-GUI) 필수. 테스트 plan은 `.jmx`(XML).
- **테스트 작성 모델**: GUI 드래그+`.jmx` XML. "scripts-as-code 친화성 낮음"은 해석적 평가(장황한 XML 근거).
- **AI 친화성**: 네이티브 AI 없음. XML이라 LLM 직접 생성 난이도 높음(해석적).
- **IaC 지원**: 네이티브 Terraform 없음.
- **자동화 API**: CLI non-GUI(`jmeter -n -t plan.jmx -l log.jtl`), HTML 리포트 자동 생성. Taurus(third-party, BlazeMeter/Perforce)로 래핑 가능 — 단 Taurus는 공식 JMeter 구성요소 아님.
- **분산·클라우드**: **RMI** controller/server(slave) 분산. 주의: 정확히 동일 버전 일치, 동일 Java, **JMeter 4.0+는 RMI 기본 SSL → keystore 필요**(또는 SSL 비활성), 서브넷 횡단엔 프록시.
- **관측성**: `InfluxDBBackendListenerClient`(3.2+) → InfluxDB(UDP/HTTP) → **Grafana 대시보드**. InfluxDB v2(5.2+), Graphite도.
- **비용·제한**: OSS 무료. 운영 난이도(버전/Java/SSL/RMI)가 높음.
- **장점**: 성숙·광범위 프로토콜·풍부한 플러그인. **Azure Load Testing / AWS DLT / BlazeMeter의 내장 엔진**으로 재사용됨(자산 이식성).
- **단점·리스크**: XML/RMI/keystore 운영 난이도, AI 직접 생성 비친화, IaC 비네이티브 → AI-friendly 직접 대상보다 Azure/AWS/BlazeMeter/Taurus 래퍼가 현실적.
- **출처**: 레지스트리 JMeter 항목.

### 3.8 BlazeMeter (Perforce)

- **개요**: 상용 SaaS. OSS 래퍼 **Taurus**로 다중 도구 실행. AWS Marketplace 제공.
- **테스트 작성 모델**: Taurus 실행기로 **JMeter(기본)/Gatling/k6/Locust/Selenium/Playwright/Newman 등** 다수 수용. 스크립트 이식성 높음.
- **AI 친화성**: **BlazeMeter MCP Server**(create/configure/run/report/workspace; Release 1.4(2026-03-01)에 API Test MCP 추가), **AI Script Assistant**(NL→API 테스트 스크립트), AI 이상탐지/근본원인/Correlation. **"AI 성능 테스트"는 "coming soon"** 표기. 무료 플랜 ~5 AI/일.
- **IaC 지원**: **공식 Terraform provider 없음**(Registry 조회 total-count 0). 자동화는 REST API + CI/CD 플러그인.
- **자동화 API**: REST v4(`a.blazemeter.com/api/v4`), 테스트 생성/시작/정지/리포트/스크립트 업로드, Swagger 제공.
- **분산·클라우드**: **Private Location**(구 harbor) + **Agent**(구 ship), Kubernetes/Radar 에이전트, 아웃바운드 poll. ~1,000 VUs/엔진(HTTP/S).
- **관측성**: AppDynamics/DX APM/CloudWatch/New Relic/Dynatrace/Datadog 미드테스트 메트릭.
- **비용·제한**: 크레딧(VU/VUH/Tests), GUI/브라우저 100×VUH, Test Data ~+50%. 공개 정가 미게시.
- **장점**: 멀티 도구 수용, 폭넓은 GenAI 스위트(MCP 포함), Private Location, APM 통합. 엔터프라이즈 readiness 높음.
- **단점·리스크**: 상용 SaaS 종속(실행/리포트/크레딧/AI). Terraform 비지원. 크레딧 모델 복잡(브라우저 100×).
- **출처**: 레지스트리 BlazeMeter 항목.

### 3.9 Google Cloud 부하 테스트 패턴

- **개요**: **Azure Load Testing/AWS DLT에 대응하는 1차 관리형 부하 테스트 제품이 없다.** 대신 OSS 도구를 GCP compute에서 돌리는 **패턴/아키텍처 가이드**를 제공.
- **패턴**: "Distributed load testing using GKE"(GKE에 분산 Locust master/worker 배포, master Pod=Web UI, worker Pod=트래픽). Cloud Run "Load testing best practices"(JMeter/Selenium 명시, 관리형 부하 제품 없음 명시, 2026-06-18 갱신). Cloud Load Balancing 부하 가이드.
- **관측성**: Cloud Monitoring/Cloud Trace로 SUT 관측, 로그를 BigQuery로 export해 분석.
- **위치**: GCP는 **실행/대상 계층(GKE/Cloud Run/Compute Engine)** + 관측 계층. 부하 SaaS가 아님.
- **장점**: GKE/Cloud Run으로 Locust/JMeter/k6를 유연하게 자가 운영, 관측성 강함.
- **단점·리스크**: 관리형 제품 부재 → 분산/스케줄/리포트/cleanup을 직접 구성. Terraform으로 인프라는 다루기 쉬우나 "부하 테스트 리소스" 추상화는 없음.
- **출처**: 레지스트리 GCP 항목.

---

## 4. 대체제 (명시 도구 외)

각 행: 무엇 / 모델 / 분산 / 유지보수·상태 / 한 줄 평가(언제 더 나음/부족). 출처와 날짜 포함.

| 도구 | 무엇 · 모델 | 프로토콜 | 분산/클라우드 | 상태·최신 | 출처 |
|---|---|---|---|---|---|
| **NeoLoad (Tricentis)** | 엔터프라이즈 플랫폼. 비주얼 + as-code(YAML) | HTTP/WebSocket/REST/**SAP/Citrix**/Oracle/Flex/모바일/RealBrowser | 온프렘+클라우드 생성기(NeoLoad Web SaaS) | v2026.1, AI Chat·**MCP server**·Agentic Perf Testing(Q1 2026) 발표 | tricentis.com/products/performance-testing-neoload, documentation.tricentis.com/neoload (미표시) |
| **Tsung** | Erlang 분산, 단일 XML 선언 | HTTP/WebDAV/SOAP/**PostgreSQL·MySQL**/LDAP/MQTT/AMQP/XMPP | 네이티브 master/worker(다수 노드) | **stale**: v1.8.0(2023-03), develop 커밋은 2026-03까지 | github.com/processone/tsung |
| **Vegeta** | Go, **constant-rate**(open model), targets 파일 | HTTP/1.1·2·h2c, mTLS | 수동(N 인스턴스 분할+`report` 집계) | 활성, CLI+Go 라이브러리, HDR 히스토그램 | github.com/tsenart/vegeta (미표시) |
| **wrk** | C, 단일 박스 최대 throughput, 선택 LuaJIT | HTTP/HTTPS | 없음(단일 머신) | 활성, README에 CO 언급 없음 | github.com/wg/wrk (미표시) |
| **wrk2** | wrk 포크, **constant throughput + CO 보정(HdrHistogram)** | HTTP/HTTPS, Lua | 없음 | **stale**: master 마지막 커밋 2019-09 | github.com/giltene/wrk2 (미표시) |
| **Fortio (Istio)** | Go, constant **QPS** + 지연 히스토그램, server/REST/UI | HTTP/1.1·**HTTP/2**·**gRPC**·TCP/UDP | 단일 인스턴스(REST로 오케스트레이션, 멀티노드 집계 없음) | 활성: v1.75.2(2026-06-11) | github.com/fortio/fortio |
| **ddosify → Anteon** | Go 엔진(노코드 시나리오·Postman import) + eBPF K8s 모니터링 | HTTP/HTTPS(HTTP/2 미확인) | **Anteon Cloud/Self-Hosted/AWS MP**(OSS 엔진은 단일 노드) | AGPL-3.0, 플랫폼 피벗(엔진 cadence 불명확), selfhosted-2.6.0(2024-08) | github.com/ddosify/ddosify, getanteon.com |
| **oha** | Rust, 실시간 **TUI**, CLI 플래그 | HTTP/1·2·(실험적 3) | 없음 | 활성: v1.14.0(2026-02-28) | github.com/hatoo/oha |
| **Bombardier** | Go, 정적 요청 CLI(스크립팅 없음) | HTTP/1.x·2(fasthttp/net-http) | 없음 | 활성, 단일 바이너리 | github.com/codesenberg/bombardier (미표시) |
| **Playwright 브라우저 부하** | 실브라우저 VU(Web Vitals 측정) | 브라우저 레벨(LCP/FCP/TTFB/CLS/INP) | Artillery 엔진→Fargate/ACI | Artillery 내장(`@artillery/engine-playwright` 패키지명 미확인). **Azure는 부하 아님(Workspaces=E2E, 2026-03-08 구 제품 은퇴)** | artillery.io/docs/reference/engines/playwright |
| **Grafana Cloud Synthetic Monitoring** | k6 기반 합성 모니터링(체크/프로브) | ping/HTTP/DNS/TCP/traceroute + k6 스크립트·브라우저 체크 | 글로벌 프로브 위치(Prometheus+Loki 저장) | **모니터링이지 부하 아님**: "1 iteration", vus/duration/stages 무시 | grafana.com/docs/grafana-cloud/testing/synthetic-monitoring/ |

### 한 줄 평가

- **NeoLoad**: SAP/Citrix/레거시 프로토콜 폭 + 벤더 지원 + 관리형 분석이 필요하면 k6/Locust/Artillery보다 우위. 단 상용·종속이 크고 비용 민감/OSS-first엔 부적합. (OSS 도구 중 유일하게 AI/MCP/Agentic을 광고하는 상용 대안.)
- **Tsung**: 저비용 대규모 분산 + DB/XMPP/AMQP 프로토콜이 필요하고 Erlang+XML을 수용하면 강점. 단 릴리스가 ~3년 정체 → DX/생태계는 현대 도구에 열세.
- **Vegeta**: 빠르고 정확한 **constant-rate HTTP** 벤치 + Go 라이브러리 임베딩에 k6/Gatling/JMeter보다 가볍고 정확. 비-HTTP/상태있는 시나리오/내장 분산 제어판은 부족.
- **wrk**: 초경량 단일 박스 최대 throughput 스모크에 최적. tail latency 정확도/멀티프로토콜/분산/대시보드는 부족.
- **wrk2**: 고정 부하에서 tail latency 정확도(CO 보정)의 정석. 단 **미유지(2019)** + 단일 노드라 프로덕션 채택은 신중.
- **Fortio**: **gRPC/HTTP2 마이크로서비스·메시** 테스트와 Go 서비스/CI 내 임베딩에 k6/JMeter보다 적합. 풍부한 시나리오 스크립팅·대규모 분산은 약함.
- **ddosify/Anteon**: 노코드 시나리오 + 관리형 클라우드 + K8s 관측을 한 플랫폼으로 원하면 자가운영 k6/JMeter보다 편함. 코드-as-test 세밀도, AGPL, 플랫폼 피벗은 채택 고려사항.
- **oha**: 라이브 TUI + 모던 HTTP/2·3로 빠른 단발 벤치에 친화적. 스크립트 시나리오/gRPC/WebSocket/분산은 부족.
- **Bombardier**: 단일 endpoint throughput/latency 스모크에 최단. 다단계 흐름/어서션/분산/대시보드 필요 시 부적합.
- **Playwright 브라우저 부하**: 실브라우저 Web Vitals 측정·E2E 재사용엔 프로토콜 도구보다 우월. 단 **머신당 VU가 수십~수백 배 적고 비용 큼**(메모리 병목) → 고볼륨 백엔드 용량 테스트엔 부적합.
- **Grafana Cloud Synthetic Monitoring**: 글로벌 가용성/SLO/업타임 상시 프로빙·조기 경보엔 최적. **부하/스트레스/용량 테스트엔 틀린 도구**(설계상 1 iteration) → 부하는 Grafana Cloud k6.

---

## 5. 점수 비교 매트릭스

점수 5=높음, 1=낮음. 방향 주의:
- **테스트 작성 난이도** 컬럼은 "작성 용이성"(5=작성 쉬움, 1=어려움)으로 읽는다.
- **lock-in** 컬럼은 5=lock-in 높음(주의), 1=낮음. (다른 컬럼과 달리 높을수록 불리.)
- 점수는 본 리서치의 정성 평가이며 절대값이 아니다.

| 도구 | 작성 용이성 | AI 생성 친화 | Terraform/IaC | CLI/API 자동화 | cloud/distributed | private endpoint | observability | 비용 통제 | enterprise readiness | lock-in | 개인 MVP |
|---|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|
| Grafana k6 / Cloud k6 | 4 | 5 | **5** | 4 | 5 | 5 | 5 | 4 | 4 | 3 | 5 |
| Gatling Enterprise | 3 | 4 | 3 | 4 | 5 | 5 | 4 | 3 | 5 | 4 | 2 |
| AWS DLT | 3 | 3 | 2 | 4 | 5 | 4 | 4 | 4 | 4 | 4 | 3 |
| Azure Load Testing | 3 | 3 | 3 | 5 | 4 | 3 | 5 | 4 | 4 | 4 | 3 |
| Artillery | 5 | 5 | 3 | 5 | 5 | 4 | 4 | 4 | 3 | 2 | 5 |
| Locust | 4 | 4 | 3 | 4 | 4 | 4 | 4 | 4 | 3 | 2 | 4 |
| Apache JMeter | 2 | 2 | 2 | 4 | 4 | 4 | 3 | 3 | 5 | 2 | 2 |
| BlazeMeter | 3 | 4 | 1 | 5 | 5 | 5 | 4 | 3 | 5 | 5 | 2 |
| Google Cloud(직접 구성) | 3 | 3 | 4 | 3 | 4 | 4 | 5 | 4 | 3 | 3 | 3 |

매트릭스 해설(이전 리서치 대비 변경):
- k6 **Terraform/IaC 4→5**: `grafana_k6_*` 리소스 6종(테스트 정의/스케줄 포함)이 실재 확정되어 상향.
- k6 **lock-in 중→3**: OSS 로컬 무료 + 스크립트 이식성은 높으나 Cloud 기능(PLZ/대시보드/스케줄)은 Grafana Cloud 종속 → 중간.
- AWS DLT **Terraform 3→2, AI 4→3**: CloudFormation/CDK-first(네이티브 Terraform 없음), MCP가 read-only(실행 불가)임을 반영.
- Azure **private endpoint 4→3**: 멀티리전이 공개 endpoint 한정(프라이빗 endpoint 부하 제약) 반영.
- BlazeMeter **Terraform 2→1, lock-in 높음→5**: 공식 provider 부재 + SaaS 종속 강함 반영.

---

## 6. 도구별 추천 상황

| 상황 | 1순위 | 근거 | 보조/대안 |
|---|---|---|---|
| **개인/포트폴리오 MVP** | **k6(OSS) 또는 Artillery** | 로컬 무료, scripts-as-code, threshold=SLO 게이트(k6), YAML/JS 작성 쉬움(Artillery). k6 Cloud는 월 500 VUh 무료로 분산/대시보드 데모 | Locust(Python 로직), oha/Vegeta(빠른 단발 벤치) |
| **AWS 조직/포트폴리오** | **AWS DLT** | AWS-native, JMeter/K6/Locust 수용, 멀티리전/스케줄, read-only MCP 분석 | Artillery `run-fargate`(경량), k6 + Terraform(보조 인프라) |
| **Azure 조직/포트폴리오** | **Azure Load Testing** | `2026-04-01` GA REST API, JMeter 자산 재사용, Azure Monitor 통합, `azurerm_load_test` 리소스 | Artillery `run-aci` |
| **엔터프라이즈(거버넌스/폭)** | **Gatling Enterprise / BlazeMeter / NeoLoad** | Private Location, RBAC/SSO, 대시보드, API/MCP, (NeoLoad는 SAP/Citrix 폭) | k6 Cloud(관측성 중심 시) |
| **observability 중심 데모** | **k6 + Grafana(+Prometheus/OTel)** | threshold→non-zero exit CI 게이트 + Prometheus/OTel output → Grafana 대시보드 + Terraform IaC를 한 스토리로 | Azure Load Testing(Azure Monitor), JMeter→InfluxDB→Grafana |
| **gRPC/HTTP2 마이크로서비스** | **Fortio** | gRPC/HTTP2 1급, Go 임베딩, 활성 유지 | k6(gRPC 모듈) |
| **실브라우저 Web Vitals 부하** | **Artillery + Playwright** | 브라우저 레벨, Web Vitals 캡처, Fargate/ACI 분산 | (Azure Playwright Workspaces는 부하 아님 — E2E 전용) |
| **상시 가용성/SLO 모니터링** | **Grafana Cloud Synthetic Monitoring** | k6 기반 글로벌 프로브, 부하가 아닌 모니터링 | (부하 필요 시 Grafana Cloud k6로 전환) |
| **게임 백엔드(Firebase) — REST 부분(로그인/세이브)** | **k6 또는 Artillery** | Auth/Functions/Firestore CRUD는 REST → HTTP 도구 적합, threshold·YAML 파라미터로 템플릿화 | 디버그 토큰으로 App Check 우회, 테스트 프로젝트·예산 격리 |
| **게임 백엔드(Firebase) — 실시간 부분(랭킹/매칭/RTDB 동기화)** | **k6(`k6/websockets`/`k6/net/grpc`) 또는 Artillery(`ws`/`socketio`)** | 실시간 Listen은 REST 불가(gRPC/WebChannel), RTDB는 WebSocket → WS/gRPC 모듈 필수 | 실제 SDK 그대로면 브라우저 엔진(k6 browser/Artillery playwright, 소수 VU) |
| **비전문가 템플릿 라이브러리(20+ 타이틀 재사용)** | **Artillery(YAML/CSV) 또는 k6(scenarios/`__ENV`)** | 선언적 파라미터화로 "값만 채우는" 재사용, 사전 검증 템플릿 라이브러리화 | NL→유사 템플릿 매칭·실행은 외부 제어면(§10) |

설계 권고(프롬프트 주도 자동화 관점):
- **기본 후보 = k6/Grafana Cloud k6.** AI 생성(`k6 x agent`/MCP preview) + threshold 기반 SLO 코드화 + `grafana_k6_*` Terraform 리소스 + 관측성이 제어 평면 설계와 가장 잘 맞물린다. 단 MCP가 preview임을 명시하고 실행은 좁은 권한의 Execution Controller가 맡는다.
- **AWS/Azure는 wrapper 전제.** AWS DLT는 CloudFormation/CDK이므로 Terraform은 계정/네트워크/보조 리소스에, DLT는 CFN로 둔다. Azure는 `azurerm_load_test`로 리소스만 IaC화하고 테스트 정의는 REST/CLI로 둔다.
- **MCP는 "조회/분석"으로 제한.** AWS DLT MCP(read-only 7), Gatling MCP(read-only 4)는 분석 보조이며, 실행/배포는 별도 승인된 액션(Gatling은 Skills, AWS는 REST/CLI)으로만.

---

## 7. Firebase / BaaS를 "부하 대상(SUT)"으로 본 분석

게임사 백엔드가 Firebase일 때는 부하 "도구"보다 부하 **대상**의 특성이 도구 선택을 좌우한다. Firebase는 단일 서버가 아니라 매니지드 서비스 묶음이며, 서비스마다 트래픽 모델·한도·과금이 다르다. 핵심 난점 둘: (1) **실시간 리스너·클라이언트 SDK 트래픽을 일반 HTTP 부하도구로 재현하기 어렵다**, (2) **App Check·보안규칙·사용량 과금이 부하 테스트 자체를 차단·왜곡·과금**한다. (출처는 §2.1 Firebase/BaaS 행.)

### 7.1 서비스별 부하 특성

| 서비스 | 트래픽·전송 모델 | 부하 핵심 한도·동작 | 과금 단위 | 일반 HTTP 도구 재현 난이도 |
|---|---|---|---|---|
| **Cloud Firestore** | REST(point ops·`runQuery`·`batchGet`) + **gRPC/WebChannel 스트리밍**(실시간 `Listen`) | **"500/50/5" 권고**(신규 컬렉션 500 ops/s→5분마다 50%↑, 강제 아님); 실시간 `Listen`은 **REST 불가**; 동시 리스너 수치 캡 공식 미기재 | 문서 read/write/delete 건수+인덱스+저장+egress(월 10GB 무료) | CRUD는 REST로 쉬움 / **실시간 리스너는 HTTP로 재현 불가** |
| **Realtime Database(RTDB)** | **WebSocket 영구 연결**(+long polling/SSE), REST도 가능 | **동시 연결 200,000(Blaze)/쓰기 1,000·s/총 64MB·분**; 초과 시 **샤딩**(프로젝트당 최대 1,000 인스턴스) | 저장 $5/GB·월 + 다운로드 GB(**보안규칙 거부 트래픽도 과금**) | WebSocket 모듈 필요(영구 연결·연결 유지 부하) |
| **Authentication** | REST(Identity Toolkit) + SDK | 신규 계정 **IP당 100/시간**, 프로젝트 **1,000 req/s·1,000만/일**, GetAccountInfo 50만/분, 커스텀 토큰 4.5만/분; **의심 트래픽엔 경고 없이 anti-abuse 차단** | MAU(월 5만 무료; Spark는 DAU 3,000 상한) | REST로 재현 가능하나 **부하가 anti-abuse·쿼터에 막힘** |
| **Cloud Functions(2nd gen)** | HTTP 트리거 = **Cloud Run 서비스**(2nd gen은 Cloud Run+Eventarc 기반) | **인스턴스당 동시 80(1~1,000 설정)**, 요청 기반 오토스케일(0까지 축소), min instances로 콜드스타트 완화 | 호출수·vCPU·메모리·네트워크 | 일반 HTTP 도구로 재현 쉬움 |
| **Cloud Run** | HTTP/gRPC | 동시성 기본 **80×vCPU**(콘솔 80, 최대 1,000), max instances로 수동 스케일 근사, 초과 요청 대기 후 **429**, 유휴 15분 유지 | vCPU·메모리·요청수 | 일반 HTTP 도구로 재현 쉬움; 공식 가이드가 **JMeter 등 외부 하니스 권장** |

### 7.2 실시간 리스너·SDK 트래픽 재현 난점 (핵심)

- Firestore 실시간 `Listen`은 공식 RPC 레퍼런스상 **"gRPC 또는 WebChannel로만 가능(REST 불가)"**. 스트리밍 `Write`도 동일. 즉 일반 HTTP 요청/응답 부하도구로는 **실시간 동기화 부하를 만들 수 없다**.
- RTDB 실시간 트래픽은 공식 모니터링 문서상 **WebSocket·long polling·SSE를 포함하며 "RESTful 요청은 포함하지 않는다"**. 영구 연결·연결 유지 비용·동시 응답이 부하의 본질.
- 재현 경로는 사실상 둘뿐: **(a) 와이어 프로토콜 직접 구사** — WebSocket/gRPC 모듈로 SDK의 프레이밍·인증·재연결을 손수 재구현(정확하지만 깨지기 쉬움); **(b) 실브라우저로 실제 SDK 구동** — Firebase JS SDK를 변형 없이 돌리는 유일한 길이나 **자원 비용이 커 고볼륨 부적합**(브라우저 VU). 도구별 매핑은 §8.

### 7.3 App Check·보안규칙·과금이 부하 테스트에 주는 영향

- **App Check**: enforcement 시 **유효한 attestation 토큰 없는 요청은 거부** → 합성/비실기기 부하 트래픽이 막힌다. CI/자동 테스트는 **디버그 토큰**으로 우회(비밀 유지 필수). 적용은 제품별, 전파 ~15분. 따라서 **부하 전 디버그 토큰 등록 또는 enforcement 해제**가 선행돼야 함.
- **보안규칙**: Firestore 규칙은 **규칙 평가당 문서 접근 호출 한도(단일·쿼리 10 / 다중·트랜잭션·배치 20)**, 초과 시 permission denied. 규칙 내 `get()/exists()`는 **거부되더라도 read로 과금**. → 부하 시 규칙 평가가 지연·비용·실패율에 직접 영향. (흔한 "쿼리=20" 서술은 오류 — 쿼리는 10 버킷.)
- **사용량 과금**: Firestore=작업 건수, RTDB=저장+다운로드(거부 트래픽 포함), Auth=MAU, Functions/Run=vCPU·요청. **부하 테스트가 곧 청구액** → 예산 한도·쿼터·테스트 프로젝트 격리가 필수. 공식적으로 **에뮬레이터는 성능/부하용이 아님**.

### 7.4 함의

- Firebase는 "단일 REST 대상"이 아니라 **REST 가능 부분(Auth/Functions/Run/Firestore CRUD)과 실시간 전용 부분(Firestore `Listen`, RTDB)이 섞인 혼합 대상**. 템플릿 라이브러리도 이 둘을 분리 설계해야 한다. 게임 시나리오 "로그인 → 컨텐츠 진행 → 종료"에서 **로그인·세이브는 REST**, **실시간 랭킹·매칭·동기화는 WS/gRPC**로 나뉜다.
- 부하도구 선정은 §8(실시간 커버리지)·§9(멀티 대상 적합성)로 이어진다.

---

## 8. 실시간/프로토콜 부하 도구 커버리지 (WebSocket·gRPC·브라우저)

Firebase처럼 SDK·실시간 의존이 큰 대상엔 도구의 **프로토콜 모듈**이 결정적이다. 아래는 공식 문서 기준 커버리지(출처 §2.1 도구 행).

### 8.1 도구별 모듈·상태

| 도구 | WebSocket | gRPC | 브라우저(실 SDK) | SSE | 상태 요약 |
|---|---|---|---|---|---|
| **k6** | `k6/websockets`(stable/core; `k6/experimental/websockets` deprecated, `k6/ws` 레거시) | `k6/net/grpc`(stable, k6 1.0서 GA; unary+streaming) | `k6/browser`(GA, v0.52 졸업; CDP 실 Chromium, Web Vitals) | 1차 모듈 없음(커뮤니티 `xk6-sse`) | **실시간 3종(WS/gRPC/브라우저) 모두 커버 — 가장 폭넓음** |
| **Artillery** | `ws` 엔진(내장) | 플러그인(서드파티) | `playwright` 엔진(내장, GA) | — | **`socketio` 엔진(내장)으로 실제 Socket.IO 클라이언트 SDK 구동** |
| **Gatling** | `ws`(OSS 코어) | gRPC 플러그인(OSS 5VU·5분 제한, Enterprise 무제한; streaming) | — | `sse`(OSS 코어) | WS·SSE를 OSS 1급 지원(SSE 지원이 드문 강점) |
| **Locust** | 내장 없음(custom client 래핑) | 문서화된 예제(custom client) | — | 없음 | **HTTP만 내장**; WS/gRPC는 직접 구현(Python 라이브러리 래핑) |
| **Fortio** | — | HTTP/2·gRPC 1급 | — | — | gRPC/HTTP2 마이크로서비스 부하의 정석(Istio 출신) |
| **ghz** | — | gRPC 전용(proto/reflection로 스텁 불필요) | — | — | gRPC 벤치 특화 CLI |
| **Thor** | WebSocket 전용(raw 프레임) | — | — | — | WS 부하 특화(Node, 활동성 낮음) |

> 주: JMeter도 WebSocket은 서드파티 플러그인, gRPC도 별도 플러그인 — 1급 내장 아님. k6 브라우저/Artillery Playwright는 공통적으로 **자원 집약 → 고부하 부적합**, "프로토콜 부하 다수 + 브라우저 소수" 혼합이 공식 권장.

### 8.2 Firebase 대상에 현실적인 도구

- **실시간(Firestore `Listen`/RTDB) — 와이어 직접 구사**: **k6(`k6/websockets`/`k6/net/grpc`)·Artillery `ws`·Gatling `ws`·Thor**. 단 SDK의 프레이밍·인증·재연결을 손수 재현해야 함. RTDB는 WebSocket이라 WS 모듈로 접근 가능하나 Firebase 내부 프로토콜(인증 토큰·경로 규약)을 직접 다뤄야 함.
- **실제 SDK 그대로**: **Artillery `socketio`**(범용 Socket.IO 앱) 또는 **브라우저 엔진(k6 `k6/browser`/Artillery `playwright`)으로 Firebase JS SDK 구동**. 정확하지만 브라우저 VU는 자원 비용이 커 **고볼륨엔 소수만** 배치.
- **REST 가능 부분(Auth/Functions/Run/Firestore CRUD)**: 일반 HTTP 부하도구(k6/Artillery/Locust/JMeter) 전부 적합.
- **결론**: Firebase 한 대상 안에서도 **REST 부분은 HTTP 도구, 실시간 부분은 WS/gRPC 또는 브라우저 도구**로 갈린다. 단일 도구로 폭이 가장 넓은 것은 **k6**(WS+gRPC+browser), 실제 SDK 재현 편의는 **Artillery**(socketio/playwright)가 강점.

---

## 9. 멀티 대상(AWS/Azure/GCP/Firebase) 도구 적합성

환경을 범용 유지하므로, 대상이 **REST API냐·실시간이냐·관리형 BaaS냐**에 따라 권장 도구가 달라진다. 한 줄 정리.

| 대상 유형(예) | 1차 권장 | 한 줄 근거 |
|---|---|---|
| **REST/HTTP API** (AWS API GW·ALB, Azure App Service, GCP Cloud Run/Functions, Firebase Functions/Auth) | **k6 / Artillery** | scripts/YAML-as-code + threshold, 모든 HTTP 도구 적합; 클라우드 관리형(AWS DLT/Azure ALT)도 가능 |
| **실시간 WebSocket** (게임 채팅·매칭·랭킹, RTDB) | **k6(`k6/websockets`)·Artillery(`ws`/`socketio`)·Gatling(`ws`)** | 영구 연결·프레임 부하는 WS 모듈 필수, 일반 HTTP 도구 부적합 |
| **gRPC 마이크로서비스** (GKE/Cloud Run gRPC, 게임 서버) | **Fortio·ghz·k6(`k6/net/grpc`)·Gatling(gRPC)** | 와이어 직접 구사, HTTP/2 스트리밍 |
| **관리형 BaaS — Firebase Firestore 실시간/RTDB** | (와이어) **k6/Artillery/Gatling WS·gRPC** + (실 SDK) **브라우저 엔진** | 실시간 `Listen`은 REST 불가(gRPC/WebChannel), SDK는 브라우저로만 변형 없이 구동 |
| **AWS 조직 전반** | **AWS DLT**(+ Artillery `run-fargate`) | AWS-native, JMeter/k6/Locust 수용, 멀티리전 |
| **Azure 조직 전반** | **Azure Load Testing**(+ Artillery `run-aci`) | JMeter/Locust 엔진, Azure Monitor, GA REST |
| **GCP 조직 전반** | **k6/Locust on GKE·Cloud Run**(자가 구성) | GCP는 1차 관리형 부하 제품 없음 → 가이드가 외부 하니스(JMeter 등) 권장 |
| **브라우저 Web Vitals** (게임 웹 포털·런처) | **Artillery Playwright·k6 browser** | 실브라우저, 단 소수 VU |

**핵심**:
- "환경 범용"이라면 **공통 분모는 k6/Artillery**(클라우드 비종속, scripts/YAML-as-code, REST+WS+gRPC+browser 폭). 클라우드 관리형(AWS DLT/Azure ALT)은 해당 클라우드엔 강하나 **Firebase·실시간엔 직접 해법이 아니다**.
- **Firebase 실시간**은 어느 클라우드 관리형으로도 "그대로" 커버되지 않음 → **와이어(WS/gRPC) 또는 브라우저 SDK 경로가 공통 필수**.

---

## 10. 템플릿(시나리오) 라이브러리화 적합성 — "자연어 → 유사 템플릿 실행"

비전문가(QE/기술지원)가 스크립트를 직접 짜지 않고 **파라미터만 채워 20+ 타이틀에 재사용**하는 것이 목표다. 즉 "자연어 프롬프트 → 가장 유사한 사전 검증 템플릿 선택 → 파라미터 주입 실행". 도구는 **파라미터화·재사용·환경 분리**가 쉬워야 한다.

### 10.1 도구별 파라미터화 수단

| 도구 | 파라미터화·재사용 수단 | 비전문가 친화 | 한 줄 평가 |
|---|---|---|---|
| **k6** | `__ENV`(`-e KEY=VALUE` CLI), `scenarios`(executor별 부하 모델), 시나리오별 `env`, options/threshold 코드화. 공식: "여러 스크립트 대신 환경변수로 스크립트 일부를 가변화" | 템플릿 1개 + env/scenario 파라미터로 타이틀 교체 | 파라미터화 + **SLO 코드화**가 강함; JS 본문은 엔지니어가 1회 작성 |
| **Artillery** | `config.variables`(다값), `config.payload`(**CSV→변수 매핑**, 계정/아이템 데이터), `config.phases`(부하 곡선), `config.environments`(`-e staging`), `$env`(시크릿) | **YAML이라 비전문가가 값만 채우기 가장 쉬움** | 선언적 YAML+CSV가 "값만 채우는" 템플릿 모델에 최적 |
| **Gatling** | feeders(CSV/JSON 데이터 주입), Java/Kotlin/Scala/JS DSL | 코드 DSL이라 비전문가 직접 작성 난이도↑ | 강력하나 템플릿 작성·파라미터는 엔지니어 영역 |
| **Locust** | Python 코드 + `--host`·커스텀 인자·환경변수 | Python 코드 의존 | 로직 자유도 높으나 비전문가 친화 낮음 |
| **JMeter** | `.jmx` + `__P()`/`-J` 프로퍼티, CSV Data Set | XML/GUI | 자산 재사용 가능하나 NL→템플릿 자동화엔 무거움 |
| **관리형(AWS DLT/Azure ALT)** | 콘솔·REST로 시나리오 저장·재실행, 파라미터는 환경변수·JMX 파라미터 | 콘솔 폼 기반 | 저장 시나리오 재실행엔 좋으나 NL 매칭·교차클라우드는 별도 제어면 필요 |

### 10.2 핵심

- **선언적 파라미터화 = 비전문가 템플릿화 우위**: **Artillery(YAML+CSV+environments)**가 "값만 채우는" 모델에 가장 직관적. **k6**는 `__ENV`+`scenarios`+threshold로 **파라미터화 + SLO 게이트**를 함께 코드화 → 자동 제어면(자연어→템플릿 선택→실행)과 가장 잘 맞물림(이미 §3.1에서 기본 후보).
- **권장 구조**: **타이틀-불변 템플릿(시나리오 로직)** + **타이틀-가변 파라미터(host·계정·부하량·SLO)** 분리. k6는 `scenarios`+`__ENV`, Artillery는 `environments`+`variables`+`payload(CSV)`로 구현 가능. 게임 시나리오(로그인 → 컨텐츠 진행 → 종료)는 단계별 파라미터(엔드포인트, 토큰 발급, 진행 호출수, 종료 저장)로 노출.
- **실시간 템플릿 주의**: WS/gRPC·브라우저 SDK 템플릿은 파라미터화는 되나 **본문 작성이 어려워**, 사전 검증된 소수 템플릿을 엔지니어가 작성·검증해 라이브러리화하는 편이 현실적(자연어→유사 템플릿 매칭 모델과 부합).

---

## 11. 남은 불확실성 + 구현 전 재검증 항목

가격·preview/GA·API version·provider 스키마는 변동성이 크다. 구현 직전 아래를 재확인한다.

| # | 항목 | 현재 상태(2026-06-26) | 재검증 포인트 |
|---|---|---|---|
| 1 | k6 MCP 서버 성숙도 | preview | GA 승격 여부, tools/권한 변경, `run_script` 상한(VU50/5분) 변동 |
| 2 | k6 Prometheus RW 플래그 | `--out experimental-prometheus-rw`(experimental) | 정식 graduation 시 플래그명 변경 가능 |
| 3 | k6 threshold exit code | 실패 시 **99**(`ThresholdsHaveFailed`, k6 소스 `errext/exitcodes/codes.go`에 정의). 사용자 문서엔 "non-zero"로만 표기 | 구현 시 99 사용·0 여부로 게이트 |
| 4 | `grafana_k6_*` 리소스 스키마 | 6종 실재, 비-베타 | provider 버전별 인자/상태, "GA 배지" 명시 여부 |
| 5 | Grafana Cloud k6 REST API | v6, 레거시 deprecated | 레거시 제거 시점, v6 엔드포인트 세부 |
| 6 | AWS DLT 버전/엔진 | v4.1.4, JMeter/K6/Locust | 번들 엔진 버전, MCP/CLI 변경, region availability |
| 7 | AWS DLT Terraform 경로 | 없음(CFN/CDK) | 공식 Terraform 경로 신설 여부(현재 없음) |
| 8 | Azure dataplane API | `2026-04-01` GA(상태 페이지는 지연) | api-lifecycle 페이지 갱신, Test Profile preview 트랙(2025-11 등) |
| 9 | Azure 부하 엔진 | JMeter+Locust(GA), Playwright는 부하 아님 | 신규 엔진 추가 여부(현재 미지원 명시) |
| 10 | `azurerm_load_test` | 리소스 실재(리소스만) | 테스트/JMX 정의 IaC화 가능 여부(현재 비-IaC 경로) |
| 11 | Gatling MCP 범위 | read-only 4 tools, 배포는 Skills | MCP에 실행 도구 추가 여부, 성숙도 라벨 신설 |
| 12 | Gatling Terraform 모듈 | `gatling/control-plane/*`(verified:false) | 모듈 버전/verified 상태, 테스트 리소스 모듈 신설 여부 |
| 13 | Artillery 라이선스 | MPL-2.0(저장소 기준, docs엔 미명시) | 공식 docs 명시 여부 |
| 14 | Locust OTel | traces+metrics(logs 아님), `requests`만 auto | 추가 라이브러리 auto-instrument 확대 여부 |
| 15 | NeoLoad AI/MCP | AI Chat·MCP·APT "발표"(마케팅) | 실제 GA/문서화 범위, 권한 모델 |
| 16 | 대체제 유지보수 | wrk2(2019)/Tsung(2023) stale, Fortio/oha 활성 | 채택 전 최신 릴리스/보안 패치 |
| 17 | 가격/쿼터 | VUh(k6/Azure/BlazeMeter), AWS는 리소스 과금 | 공식 가격/쿼터 페이지에서 단가·한도 별도 확인 |
| 18 | Firebase 실시간 Listen 재현 | gRPC/WebChannel 전용(REST 불가), SDK 내부 프로토콜 | WebChannel 프레이밍·토큰 규약, k6/Artillery로 실제 재현 PoC 필요 |
| 19 | Firestore 동시 리스너·연결 한도 | 공식 쿼터 페이지에 동시 연결·리스너 수치 캡 미기재 | 공식 한도 재확인, **"1,000,000" 수치 인용 금지**(미검증) |
| 20 | RTDB 다운로드 단가 | 저장 $5/GB·월만 verbatim 확정, 다운로드 $/GB 미확정 | firebase.google.com/pricing에서 다운로드 단가 확인 |
| 21 | App Check enforcement | 토큰 없는 트래픽 거부, 디버그 토큰 우회, 전파 ~15분 | 부하 전 디버그 토큰 등록/enforcement 토글 절차, 제품별 적용 범위 |
| 22 | Firestore 보안규칙 접근 한도 | 단일·쿼리 10 / 다중·트랜잭션·배치 20, `get()/exists()` 과금 | 부하 시 규칙 평가 비용·실패율, 일부 호출 캐시 동작 |
| 23 | Functions 2nd gen 동시성 | 기본 80(1~1,000), Cloud Run 기반 | "동시성엔 ≥1 vCPU 필요" 류 제약은 공식 재확인(이번 보강서 미확정) |
| 24 | 실시간 도구 버전·자원 | k6/browser GA·`k6/websockets` stable, Artillery `socketio` 내장 | Socket.IO 버전 호환(서드파티 v3 엔진 주장은 비공식), 브라우저 VU 자원 한도 |
| 25 | Identity Platform MAU 단가 | 월 5만 무료 확정, $/MAU 미확정 | 공식 가격 페이지에서 단가 확인 |

검증 체크리스트:
- [x] 9개 명시 도구 + 11개 대체제를 2025~2026 공식 문서로 확인
- [x] 핵심 8개 재검증 주장(k6 threshold/MCP/Terraform/PLZ, AWS DLT, Azure API, Artillery Pro, Gatling MCP) 모두 공식 출처로 판정
- [x] 이전 리서치 대비 7개 정정 사항 명시(특히 Azure Playwright 부하 오류, Gatling MCP read-only)
- [x] 장점과 단점/리스크/lock-in 병기, 가격은 단위만(단가 인용 회피)
- [x] 코드/Terraform apply/실행 없음(리서치 단계 한정)
- [x] (보강) Firebase/BaaS 5개 서비스(Firestore/RTDB/Auth/Functions/Cloud Run)를 부하 대상으로 2026 공식 문서 검증
- [x] (보강) 실시간/프로토콜 커버리지(k6 WS/gRPC/browser, Artillery ws/socketio, Gatling ws/sse/grpc, Locust, Fortio/ghz/Thor) 공식 모듈 상태 확인
- [x] (보강) 멀티 대상 적합성·템플릿 라이브러리화·App Check/보안규칙/사용량 과금 영향 추가
- [x] (보강) 미검증 수치 인용 회피 명시(Firestore "1M 동시연결" 금지, 보안규칙 "쿼리=20" 오류 정정, GCP "관리형 부하 제품 없음"은 추론)
