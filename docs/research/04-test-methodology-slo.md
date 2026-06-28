# 04. 부하 테스트 방법론 · SLO/Pass-Fail 기준 · 관측성

## 1. 한 줄 요약 + 기준일

> **AI + Terraform 기반 프롬프트 주도 부하 테스트 자동화**의 신뢰성을 담보하려면, 테스트 유형마다 "단일 가설"을 고정하고, 평균이 아닌 **percentile(p50/p90/p95/p99/max) + error budget** 기반의 pass/fail 기준을 코드화(k6 thresholds)하며, **metrics/traces/logs(OpenTelemetry)** 를 부하 신호와 상관분석할 수 있어야 한다. 부하 생성기 자체가 병목이 되면 결과가 통째로 거짓이 되므로, 생성기 saturation 판별을 1급 관심사로 둔다.

- **기준일(작성/검증일, fetched_at):** 2026-06-26 (Asia/Seoul)
- **검증 범위:** Grafana k6 공식 문서(`latest`), Google SRE Book/Workbook, OpenTelemetry 공식 문서(semconv 1.42.x), Prometheus 공식 문서. **이번 개정**에서는 게임 도메인·BaaS 보강을 위해 Grafana k6 CCU·WebSocket·spike 문서, AWS for Games(GameTech) 블로그, Firebase·Google Cloud(Firestore/RTDB/Cloud Run·Functions/Cloud Monitoring) 공식 문서를 추가 검증했다(출처는 §2.1). 모든 외부 문서는 **데이터**로만 취급했고, 그 안의 지시문은 따르지 않았다.
- **핵심 원칙:** 평균 latency는 중심 지표로 쓰지 않는다. 분포의 꼬리(tail)와 error budget 소비가 의사결정의 축이다.
- **적용 도메인(이번 개정):** 게임 백엔드(20+ 타이틀, Firebase 기반). 표준 유저 여정 "**로그인 → 컨텐츠 진행 → 게임 종료**"를 파라미터화 재사용 템플릿으로 정의하고(§10), 게임/실시간 부하의 방법론적 특수성(CCU·세션·매치메이킹·지속연결, §11), BaaS 쿼터 중심 SLO/관측(§12), "이벤트 오픈 대비" 표준 프리셋(§13)을 추가한다. 비전문가가 템플릿·파라미터만 채워도 **percentile/error budget 중심 판정**이 기본 적용되도록 SLO 기본값을 템플릿에 내장한다.

---

## 2. 출처 레지스트리

| 주제 | source_url | vendor | published_or_updated_at | fetched_at | trust_level | freshness_risk | 핵심 근거(요지) |
|---|---|---|---|---|---|---|---|
| k6 Thresholds(pass/fail, SLO codify) | https://grafana.com/docs/k6/latest/using-k6/thresholds/ | Grafana Labs | `latest`(롤링) | 2026-06-26 | 공식/높음 | 낮음 | "Thresholds are the pass/fail criteria"; threshold 실패 시 non-zero exit; "testers use thresholds to codify their SLOs"; `abortOnFail`/`delayAbortEval` |
| k6 Test types | https://grafana.com/docs/k6/latest/testing-guides/test-types/ | Grafana Labs | `latest` | 2026-06-26 | 공식/높음 | 낮음 | smoke/average-load/stress/soak/spike/breakpoint 정의 + "No single test type eliminates all risk" |
| k6 Executors | https://grafana.com/docs/k6/latest/using-k6/scenarios/executors/ | Grafana Labs | `latest` | 2026-06-26 | 공식/높음 | 낮음 | 6개 executor(constant/ramping-vus, constant/ramping-arrival-rate, per-vu/shared-iterations) |
| k6 Open vs closed model | https://grafana.com/docs/k6/latest/using-k6/scenarios/concepts/open-vs-closed/ | Grafana Labs | `latest` | 2026-06-26 | 공식/높음 | 낮음 | closed=직전 iteration 종료 후 시작(coordinated omission), open=도착률이 응답시간과 무관 |
| k6 Running large tests(생성기 병목) | https://grafana.com/docs/k6/latest/testing-guides/running-large-tests/ | Grafana Labs | `latest` | 2026-06-26 | 공식/높음 | 낮음 | CPU 80% 상한, 100% 시 결과 response time 왜곡(inflation), 네트워크 1Gbit/s 한계(`iftop`) |
| k6 Built-in metrics | https://grafana.com/docs/k6/latest/using-k6/metrics/reference/ | Grafana Labs | `latest` | 2026-06-26 | 공식/높음 | 낮음 | `http_req_duration`(p95/p99), `http_req_failed`(rate), `dropped_iterations`, `vus` 등 |
| SRE Book — SLO(SLI/SLO/SLA, 분포/percentile) | https://sre.google/sre-book/service-level-objectives/ | Google | 2016 출판(온라인 상시 공개) | 2026-06-26 | 공식/높음 | 중간(2016 기반, 단 표준) | "averaging ... obscures ... a long tail"; 99th=worst-case, 50th=typical |
| SRE Book — Monitoring(4 Golden Signals) | https://sre.google/sre-book/monitoring-distributed-systems/ | Google | 2016 | 2026-06-26 | 공식/높음 | 중간 | latency/traffic/errors/saturation; "latency increases are often a leading indicator of saturation" |
| SRE Workbook — Alerting on SLOs(burn rate) | https://sre.google/workbook/alerting-on-slos/ | Google | 2018 | 2026-06-26 | 공식/높음 | 중간 | "Burn rate is how fast, relative to the SLO, the service consumes the error budget"; multiwindow multi-burn-rate |
| OpenTelemetry — Signals | https://opentelemetry.io/docs/concepts/signals/ | OpenTelemetry(CNCF) | 롤링 | 2026-06-26 | 공식/높음 | 낮음 | traces/metrics/logs/baggage 정의, context 전파로 신호 상관 |
| OTel — HTTP metrics semconv | https://opentelemetry.io/docs/specs/semconv/http/http-metrics/ | OpenTelemetry | semconv 1.42.x(stable) | 2026-06-26 | 공식/높음 | 낮음 | `http.server.request.duration` = Histogram, 단위 `s`, 권장 bucket 경계 |
| Prometheus — Histograms & summaries | https://prometheus.io/docs/practices/histograms/ | Prometheus | 롤링 | 2026-06-26 | 공식/높음 | 낮음 | summary는 φ(rank) 차원 오차, histogram은 관측값(value/버킷) 차원 오차; quantile 평균은 "BAD!"; histogram=서버측 계산, summary=클라이언트측 |
| Chaos Engineering 원칙(steady-state/blast radius) | https://cloud.google.com/blog/products/devops-sre/getting-started-with-chaos-engineering | Google Cloud | 블로그(상시) | 2026-06-26 | 벤더/중상 | 중간 | steady-state hypothesis, 최소 blast radius부터 확장, 실제 장애 모델링 |
| 대체 스택 비교(LGTM/VictoriaMetrics) | https://victoriametrics.com/blog/prometheus-monitoring-metrics-counters-gauges-histogram-summaries/ · https://onidel.com/blog/prometheus-storage-comparison-2025 | VictoriaMetrics / 커뮤니티 | 2025 | 2026-06-26 | 2차/중 | 중상 | metrics 전용 TSDB 비교(보조 근거) |

> freshness 메모: Grafana k6 `latest`는 롤링 URL이라 시점 표기가 없다 → **구현 직전 재확인 필요**(특히 executor/threshold 옵션명). Google SRE Book은 2016/2018 텍스트지만 SLI/SLO/error budget의 사실상 표준 정의이므로 freshness보다 "정의 안정성"이 높다. OTel semconv는 **버전 고정(1.42.x)** 으로 인용했다.

### 2.1 게임 도메인·BaaS 보강 출처 (이번 개정 추가분)

| 주제 | source_url | vendor | published_or_updated_at | fetched_at | trust_level | freshness_risk | 핵심 근거(요지) |
|---|---|---|---|---|---|---|---|
| k6 동시접속(CCU) 계산 | https://grafana.com/docs/k6/latest/testing-guides/calculate-concurrent-users/ | Grafana Labs | `latest`(롤링) | 2026-06-26 | 공식/높음 | 낮음 | "Concurrent users = Hourly sessions × Average session duration / 3600"; "Daily averages can mask ... variations" → 피크 기준 |
| k6 Spike testing | https://grafana.com/docs/k6/latest/testing-guides/test-types/spike-testing/ | Grafana Labs | `latest` | 2026-06-26 | 공식/높음 | 낮음 | "sudden and massive rushes"; 예시 product launch(PS5)·seasonal sales |
| k6 WebSockets(지속 연결) | https://grafana.com/docs/k6/latest/using-k6/protocols/websockets/ | Grafana Labs | `latest` | 2026-06-26 | 공식/높음 | 낮음 | VU가 연결 수명 동안 "asynchronous event loop" → 부하 단위=연결 |
| k6-learn think time | https://github.com/grafana/k6-learn/blob/main/Modules/II-k6-Foundations/05-Adding-think-time-using-sleep.md | Grafana(k6-learn) | repo archived 2026-06-05 | 2026-06-26 | 공식(교육)/중상 | 중간(archived) | think time 정의; 없으면 "stress the load generator itself rather than the application" |
| AWS GameTech — 1M CCU 부하테스트 | https://aws.amazon.com/blogs/gametech/load-testing-the-pragma-backend-game-engine-to-1-million-concurrent-users-on-aws/ | AWS for Games | 2024-01-16 | 2026-06-26 | 벤더 블로그/중상 | 중간 | CCU를 백엔드 용량 1차 척도로 1k→1M 스케일 |
| AWS GameLift — 100M CCU/런치 스케일 | https://aws.amazon.com/blogs/gametech/amazon-gamelift-achieves-100-million-concurrently-connected-users-per-game/ | AWS for Games | 2025-02-06 | 2026-06-26 | 벤더 블로그/중상 | 중간 | "session creation rate·VM deployment speed"가 런치 핵심 인자(CCU만으론 부족) |
| AWS GameLift FlexMatch — 매치메이킹 메트릭 | https://aws.amazon.com/blogs/gametech/matchmaking-your-way-amazon-gamelift-flexmatch-and-game-session-queues/ | AWS for Games | 2017-08-16 | 2026-06-26 | 벤더 블로그/중상 | 중상(2017) | Time to Ticket Success·Player Demand·Match Success/Failure Rates |
| Firestore best practices(500/50/5·hotspot) | https://cloud.google.com/firestore/docs/best-practices | Google Cloud | 2026-06-19 | 2026-06-26 | 공식/높음 | 낮음 | "500 ops/sec ... +50% every 5 min"; hotspotting; 단일문서 쓰기율 "depends on workload" |
| Firestore quotas(하드 한계·무료등급) | https://firebase.google.com/docs/firestore/quotas | Firebase | 2026-06-22 | 2026-06-26 | 공식/높음 | 낮음 | "hard limits unless otherwise noted"; 읽기 50k·쓰기 20k·삭제 20k per day |
| Firestore error codes(RESOURCE_EXHAUSTED) | https://cloud.google.com/firestore/native/docs/understand-error-codes | Google Cloud | 2026-06-19 | 2026-06-26 | 공식/높음 | 낮음 | 쿼터/ramp 초과 → RESOURCE_EXHAUSTED, "retry with exponential backoff" |
| RTDB limits(연결·쓰기) | https://firebase.google.com/docs/database/usage/limits | Firebase | 2026-06-22 | 2026-06-26 | 공식/높음 | 낮음 | 동시연결 200,000(Spark 100)·쓰기 1,000/sec(소프트)·응답 ~100,000/sec |
| RTDB sharding | https://firebase.google.com/docs/database/usage/sharding | Firebase | 2026-06-22 | 2026-06-26 | 공식/높음 | 낮음 | 한계 초과 시 샤딩; Blaze 프로젝트당 최대 1,000 인스턴스 |
| Cloud Run/Functions cold start·best practices | https://cloud.google.com/run/docs/tips/functions-best-practices | Google Cloud | 2026-06-18 | 2026-06-26 | 공식/높음 | 낮음 | "initialized from scratch ... cold start"; min instances 권장 |
| Cloud Run min instances | https://cloud.google.com/run/docs/configuring/min-instances | Google Cloud | 2026-06-18 | 2026-06-26 | 공식/높음 | 낮음 | warm 인스턴스 최소 유지로 cold start 완화 |
| Cloud Run instance autoscaling | https://cloud.google.com/run/docs/about-instance-autoscaling | Google Cloud | 2026-06-18 | 2026-06-26 | 공식/높음 | 낮음 | scale-to-zero 기본·idle 최대 15분 |
| Cloud Run concurrency | https://cloud.google.com/run/docs/about-concurrency | Google Cloud | 2026-06-18 | 2026-06-26 | 공식/높음 | 낮음 | 기본 80×vCPU·최대 1000·concurrency=1은 스케일 악화 |
| Cloud Run service configuration / max instances | https://docs.cloud.google.com/run/docs/configuring · https://docs.cloud.google.com/run/docs/configuring/max-instances-limits | Google Cloud | 2026-06-18 | 2026-06-26 | 공식/높음 | 낮음 | revision은 기본 최대 100 인스턴스까지 scale out. 서비스 최대치는 CPU/메모리/GPU regional quota 중 최솟값 영향 |
| Cloud Functions(Firebase) quotas | https://firebase.google.com/docs/functions/quotas | Firebase | 2026-06-22 | 2026-06-26 | 공식/높음 | 낮음 | (1세대) 동시호출 3,000·이벤트율 1,000/sec·동시 이벤트 데이터 10MB |
| Firestore monitor usage(Cloud Monitoring) | https://firebase.google.com/docs/firestore/monitor-usage | Firebase | 2026-06-22 | 2026-06-26 | 공식/높음 | 낮음 | Document Reads/Writes/Deletes·Active Connections·Snapshot Listeners; 1분 샘플·~4분 지연 |
| Cloud Functions monitoring metrics | https://cloud.google.com/functions/docs/monitoring/metrics · https://cloud.google.com/monitoring/api/metrics_gcp_c | Google Cloud | 2026-06-18 / 2026-06-24 | 2026-06-26 | 공식/높음 | 낮음 | execution_count(상태별)·execution_times(분포)·active_instances·user_memory_bytes |

> **미확정·인용 금지 항목(정직성):** ① Firestore "단일 문서 1 write/sec" 하드 숫자(현재 공식은 "워크로드 의존") ② Cloud Functions "기본 최대 인스턴스" 단일 수치 ③ 게임 런치 "10x/100x in minutes"류 수치 — 모두 1차 페이지에서 확정 못 했으므로 인용하지 않는다(§9). Cloud Run revision의 기본 최대 인스턴스 100은 2026-06-26 공식 문서로 확인했다.

---

## 3. 테스트 유형 표

각 테스트는 **한 번에 하나의 주요 가설만** 검증하도록 설계한다(k6: "No single test type eliminates all risk").

| 유형 | 목적 | 트래픽 프로파일(executor 권장) | 단일 가설 | 성공 기준(pass/fail) | 흔한 함정 |
|---|---|---|---|---|---|
| **Load(Average-load)** | 예상 정상 피크에서 SLO 충족 확인 | 정상 피크 도착률 유지, ramp-up→sustain→ramp-down. **open model**(`constant-arrival-rate` / `ramping-arrival-rate`) | "정상 피크 RPS에서 p95/p99 latency·error율이 SLO 이내로 유지된다" | `p(95)<목표`, `p(99)<목표`, `http_req_failed: rate<목표`, error budget 소비 ≤ 허용치 | think time 없음 / cache warm-only / 데이터셋이 작아 비현실적 캐시 적중 |
| **Stress** | 정상 범위 초과 시 성능 저하 양상·복구 패턴 관찰 | 평균 초과(고VU/고도착률), 단계적 상향. `ramping-arrival-rate` | "정상 초과 부하에서 시스템이 graceful degradation 후 부하 제거 시 회복한다" | 저하가 graceful(타임아웃·5xx 급증 없이 점진), ramp-down 후 recovery time ≤ 목표 | closed model 사용으로 부하가 응답시간에 끌려가 실제보다 약하게 가해짐(coordinated omission) |
| **Spike** | 짧은 급증/급감 대응(autoscaling/queue/backpressure) | 매우 짧고 큰 급증→급감. `ramping-arrival-rate`(가파른 stage) | "초단기 트래픽 급증 시 backpressure/큐/오토스케일이 동작하고 사용자 영향이 한정된다" | 급증 구간 error율·timeout 상한 이내, autoscaling delay·queue depth 회복, 급감 후 정상화 | autoscaling warm-up 지연 무시 / 급증 직후 cold cache 효과 미반영 |
| **Soak(Endurance)** | 장시간 실행으로 memory/connection leak·성능 드리프트 탐지 | 평균 부하를 수 시간 유지. `constant-arrival-rate` | "장시간 동일 부하에서 메모리/커넥션/latency가 시간에 따라 드리프트하지 않는다" | 시간축 회귀 기울기 ≈ 0(메모리·p99·GC·커넥션 수), OOM/재시작 0 | soak 자체를 생략 / 관측 윈도가 짧아 누수 미검출 / 생성기도 장시간 누수 |
| **Breakpoint** | 최초 SLO 위반점·포화점·실패점 탐색 | 한계까지 점진 상향(중단 없이 계속 증가). `ramping-arrival-rate` + `abortOnFail` | "부하를 점증할 때 최초로 SLO를 위반하는 도착률(임계 용량)은 X다" | 산출물은 pass/fail보다 **임계 용량값**(최초 SLO 위반 RPS, 포화 시작점, 실패점) | 자주 돌려 환경을 흔듦 / 생성기 병목을 시스템 한계로 오판 |
| **Chaos-adjacent** | 제한적 장애(네트워크 지연·의존성 오류율↑·느린 downstream)에서 회복성 | 정상~약상 부하 + 결함 주입(최소 blast radius부터). open model 부하 위에 fault inject | "의존성 지연/오류 주입 시에도 steady-state SLI가 유지(또는 한정 열화 후 회복)된다" | steady-state hypothesis 충족: 결함 중/후 SLI가 정의된 한계 이내, 회복 시간 ≤ 목표 | 부하·결함을 동시에 크게 줘 원인 분리 불가 / blast radius 미통제 / 비현실적 결함 |

근거: 유형 정의는 [k6 Test types](https://grafana.com/docs/k6/latest/testing-guides/test-types/), open/closed·coordinated omission은 [k6 open vs closed](https://grafana.com/docs/k6/latest/using-k6/scenarios/concepts/open-vs-closed/), chaos 원칙은 [Google Cloud — Chaos Engineering](https://cloud.google.com/blog/products/devops-sre/getting-started-with-chaos-engineering).

---

## 4. 필수 입력값 체크리스트 (프롬프트가 모호해도 시스템이 구조화해야 할 항목)

> 프롬프트 주도 자동화의 1차 책임은 "모호한 자연어"를 아래 구조화 슬롯으로 **강제 정규화**하고, 빠진 값은 안전 기본값·명시적 질문으로 메우는 것이다.

| 분류 | 슬롯 | 비고/안전장치 |
|---|---|---|
| 대상 정의 | 서비스/엔드포인트, **사용자 여정(journey)**, 읽기/쓰기 비율, **destructive 여부** | 쓰기·destructive면 격리/롤백 필수 |
| 환경 | local / dev / staging / **prod** 여부 | prod 부하는 별도 승인 게이트 |
| 테스트 의도 | 테스트 유형(Load/Stress/Spike/Soak/Breakpoint/Chaos), **단일 가설** | 한 실행=한 가설 강제 |
| 부하량 | 목표 RPS, concurrency, VU, **arrival rate**, duration, ramp(상/하강 stage) | open model 우선(아래 §3·§8) |
| 분포/환경 | 지역 분포, 네트워크 조건, 시간대(피크) | 생성기 지역·대역폭 영향 |
| 인증 | 인증/세션/토큰 발급·갱신 | 토큰 만료로 5xx/401 오탐 방지 |
| Synthetic user | 사용자 유형·비율, **think time** | think time 0 = 비현실적 부하(§8) |
| 데이터 | 시딩/격리/**idempotency**, 데이터셋 크기·카디널리티 | 작은 데이터셋 = 캐시 과적중(§8) |
| 외부 의존성 | partner API / email / SMS / payment 등 | 실호출 금지 대상·모킹·쿼터 |
| 기준 | **SLO**(아래 §5 지표 일체) | pass/fail로 코드화(threshold) |
| 관측 | 관측 도구/대시보드/상관 키(trace id) | metrics/traces/logs 연결(§6) |
| 안전 | 안전 상한(max VU/RPS, kill switch), 비용 상한 | `abortOnFail`로 자동 중단 |
| 거버넌스 | 승인자, 사전 공지(on-call/관련팀) | prod·고부하 시 필수 |

---

## 5. SLO / Pass-Fail 지표 표

> 모든 latency 지표는 **평균이 아니라 분포(percentile)** 로 본다. Google SRE: 평균은 "a long tail of requests"를 가린다; 99th=plausible worst-case, 50th=typical. (출처: [SRE Book — SLO](https://sre.google/sre-book/service-level-objectives/))

| 지표 | 정의 | 왜 필요한가(SRE 근거) | 권장 기준 예시(threshold 표현) | 주의점 |
|---|---|---|---|---|
| **p50 latency** | 응답시간 중앙값 | "typical case"를 대표. 평균보다 분포 왜곡에 강함 | `http_req_duration: p(50)<목표` | 정상으로 보여도 꼬리가 나쁠 수 있음 → 단독 판단 금지 |
| **p90 / p95 latency** | 90/95 percentile | 대다수 사용자 체감 상한. UX SLO의 주 기준 | `p(95)<200` (예: 95%가 200ms 미만) | bucket/표본 부족 시 부정확 |
| **p99 latency** | 99 percentile | 꼬리 지연·saturation **선행 신호**("99th over a small window = early signal of saturation") | `p(99)<목표` | **누락 금지**(§8). 표본 적으면 변동 큼 |
| **max latency** | 최악값 | 타임아웃·극단 outlier 탐지 | `max<타임아웃` | 단일 outlier에 민감 → 보조 지표 |
| **request success ratio** | good/total 이벤트 비율 | SLI의 정석: "ratio of good events / total events" | `checks: rate>0.99` | "good" 정의(상태코드+의미적 정확성) 명시 |
| **4xx/5xx/error/timeout/retry rate** | 실패·재시도 비율 | error budget 직접 소비. 4xx(클라)·5xx(서버) 분리 | `http_req_failed: rate<0.01` | 재시도가 실패를 가릴 수 있음(retry storm) |
| **RPS/throughput** | 초당 요청·처리량 | Golden Signal "traffic". 용량/추세 분석 | 목표 RPS 달성 여부 + `dropped_iterations` 0 | 달성 못하면 SUT 한계인지 **생성기 한계**인지 구분(§8) |
| **concurrency / 활성 VU** | 동시 처리 수 | 부하 수준·포화 해석의 분모 | `vus`, `vus_max` 모니터 | open model에선 VU가 자동 증감 |
| **session completion rate** | 여정 완주 비율 | 부분 실패(중간 단계 깨짐) 포착 | 그룹/check 통과율 | 단건 요청만 보면 여정 단절 미검출 |
| **CPU / memory / GC / network / disk I/O** | 리소스 사용률 | Golden Signal "saturation". 한계·누수 근거 | soak에서 기울기≈0, CPU 포화 임계 | **SUT의 리소스**와 **생성기 리소스**를 분리 측정 |
| **DB connections / slow query / lock wait / replication lag** | DB 포화·경합·지연 | 흔한 1차 병목·꼬리 지연 원인 | 커넥션 풀 고갈 0, lock wait 상한 | 풀 고갈이 app 5xx로만 보일 수 있음 |
| **queue depth / lag / worker saturation** | 큐 적체·소비 지연 | 비동기/backpressure 동작 검증(spike) | 적체 회복, lag 상한 | 큐가 latency를 "흡수"해 app latency가 좋아 보임 |
| **autoscaling delay / pod restart / OOM / eviction** | 오토스케일 지연·재시작 | spike/stress의 핵심 회복 메커니즘 | scale-out 지연 ≤ 목표, OOM·eviction 0 | warm-up 지연이 spike 초반 SLO 위반 유발 |
| **recovery time(ramp-down 후)** | 부하 제거 후 정상화 시간 | 회복성(stress/spike/chaos)의 정량 지표 | 정상 SLI 복귀 ≤ 목표 시간 | 회복 미측정 시 "버텼다"만 알고 회복 모름 |
| **error budget 소비량** | (1 − SLO) 대비 소비 비율·**burn rate** | "Burn rate is how fast, relative to the SLO, the service consumes the error budget" | 테스트 구간 burn rate ≤ 임계, multiwindow multi-burn-rate | 짧은 테스트는 burn rate 변동이 큼 → 윈도 길이 주의 |

**threshold = SLO 코드화:** k6 threshold가 깨지면 **non-zero exit**로 종료되어 CI/CD가 자동 실패한다. 장기 위반이 누적되면 `abortOnFail`로 즉시 중단 가능. (출처: [k6 Thresholds](https://grafana.com/docs/k6/latest/using-k6/thresholds/))

**burn rate 운영:** SRE Workbook은 고정 오류율 알림 대신 **multiwindow multi-burn-rate**(짧은 윈도로 "여전히 소비 중"인지 확인; 짧은 윈도≈긴 윈도의 1/12)를 권장한다. fast burn=즉시 page, slow burn=티켓. (출처: [SRE Workbook — Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/))

---

## 6. 관측성 시그널 매핑 (metrics/logs/traces가 각 테스트 유형에서 답하는 질문)

OpenTelemetry 신호: **traces**(요청의 경로), **metrics**(런타임 측정값), **logs**(이벤트 기록), **baggage**(신호 간 전파 컨텍스트). 셋을 같은 trace context로 묶어야 상관분석이 가능하다. (출처: [OTel Signals](https://opentelemetry.io/docs/concepts/signals/))

| 테스트 유형 | metrics가 답하는 질문 | traces가 답하는 질문 | logs가 답하는 질문 |
|---|---|---|---|
| **Load** | "정상 피크에서 p95/p99·error율이 SLO 이내인가? throughput 목표 도달?" | "느린 요청의 시간이 어느 span(DB/외부호출)에 쌓이나?" | "임계 부근 경고/에러 메시지가 나오는가?" |
| **Stress** | "포화 시작 시 saturation 지표(CPU/큐/커넥션)가 어떻게 오르나?" | "저하 시 어떤 의존성이 임계 경로(critical path)인가?" | "타임아웃·서킷브레이커·풀 고갈 로그 시점은?" |
| **Spike** | "급증 시 autoscaling delay·queue depth·error 스파이크 크기는?" | "급증 초반 cold start/커넥션 셋업이 latency를 만드나?" | "scale-out/throttle/429 backpressure 이벤트 기록" |
| **Soak** | "시간축으로 memory/p99/커넥션이 드리프트하나?(기울기)" | "오래 살아남은 span/누적 리소스 보유 추적" | "주기적 누수·재연결·GC 압박 로그 추세" |
| **Breakpoint** | "최초 SLO 위반·dropped_iterations 발생 도착률은?" | "포화 직전 임계 경로가 어디로 이동하나?" | "한계 직전 최초 에러 유형/엔드포인트" |
| **Chaos-adjacent** | "결함 주입 중 steady-state SLI 유지/회복 시간?" | "지연·오류가 주입된 의존성과 사용자 영향의 인과 경로" | "재시도·폴백·서킷 오픈/클로즈 이벤트" |

- **표준 메트릭 권장:** HTTP 서버 지연은 `http.server.request.duration`(Histogram, 단위 `s`)로 수집하면 percentile 계산에 적합. 트레이스 span과 duration 정합을 맞춘다. (출처: [OTel HTTP metrics semconv](https://opentelemetry.io/docs/specs/semconv/http/http-metrics/))
- **상관분석 키:** 부하 생성기(k6) 쪽 지표(클라이언트 측정 latency)와 SUT 측 OTel 메트릭/트레이스를 **trace id/시각**으로 정렬해, "느린 건 네트워크인가 서버인가 생성기인가"를 분리한다.
- **Golden Signals와의 매핑:** latency(p50~p99)·traffic(RPS)·errors(4xx/5xx)·saturation(CPU/큐/커넥션). "latency 증가는 saturation의 선행 신호." (출처: [SRE Book — Monitoring](https://sre.google/sre-book/monitoring-distributed-systems/))

---

## 7. 대체 관측 스택 비교 (부하 결과 상관분석 관점 한 줄 평가)

| 스택/제품 | 신호 범위 | 부하 결과 상관분석 관점 한 줄 평가 |
|---|---|---|
| **Grafana Tempo**(traces) | traces | k6/Grafana와 동일 생태계라 trace↔메트릭 연결(exemplar)·드릴다운이 자연스러움 |
| **Grafana Loki**(logs) | logs | 라벨 기반 로그; 부하 구간 에러 로그를 메트릭과 시간축 정렬하기 좋음(단 고카디널리티 라벨 주의) |
| **Grafana Mimir**(metrics 장기저장) | metrics(장기) | Prometheus 호환·멀티테넌시; soak 등 장기 추세·회귀 분석에 유리(운영 복잡도↑) |
| **Prometheus vs VictoriaMetrics** | metrics | Prometheus=표준·생태계; VictoriaMetrics=Prometheus 호환 + 압축/리소스 효율로 대규모·장기 보존에 유리(단 metrics 전용) |
| **Datadog**(상용 APM) | metrics+traces+logs | 통합·즉시성 강점, 부하 시 카디널리티/이벤트 폭증 = 비용 급증 리스크 |
| **New Relic**(상용 APM) | metrics+traces+logs | 통합 APM·트랜잭션 분해 용이; 부하 테스트 트래픽도 과금 대상이라 비용 통제 필요 |
| **Honeycomb**(상용) | traces 중심(high-cardinality) | 고카디널리티 이벤트 탐색이 강점 → 꼬리 지연 원인을 차원별로 쪼개 분석하기 좋음 |
| **Elastic APM** | metrics+traces+logs | ELK 보유 시 로그-트레이스 통합 용이; 대규모 부하 시 인덱싱 비용·클러스터 튜닝 부담 |
| **AWS CloudWatch** | metrics+logs(+X-Ray traces) | AWS 인프라 지표(ALB/RDS/ASG)와 부하 결과를 같은 콘솔에서 상관; 세밀 percentile은 설정 필요 |
| **Azure Monitor / App Insights** | metrics+traces+logs | Azure 리소스·App 지표 통합, 분산 추적 제공; 쿼리(KQL) 학습 곡선 |
| **Google Cloud Monitoring** | metrics+logs(+Cloud Trace) | GCP 인프라·SLO 기능 내장, 부하 결과를 SLO/burn rate와 직접 연결하기 용이 |

> 평가는 일반적 특성 요약이며, 비용·카디널리티·보존기간은 워크로드에 따라 달라지므로 **구현 직전 PoC로 재확인** 권장. 2차 출처: [VictoriaMetrics blog](https://victoriametrics.com/blog/prometheus-monitoring-metrics-counters-gauges-histogram-summaries/), [Prometheus storage 비교](https://onidel.com/blog/prometheus-storage-comparison-2025).

---

## 8. 흔한 함정 / 안티패턴 목록

| # | 안티패턴 | 왜 위험한가 | 권장 대응 |
|---|---|---|---|
| 1 | **부하 생성기 병목을 SUT 한계로 오판** | 생성기 CPU 100%면 결과 latency가 "실제보다 훨씬 크게" 부풀고, RPS도 못 채움(`dropped_iterations`↑) | 생성기 CPU ≤ 80% 유지, `iftop`로 네트워크(1Gbit/s) 확인, 생성기 리소스를 별도 메트릭으로 감시 ([k6 large tests](https://grafana.com/docs/k6/latest/testing-guides/running-large-tests/)) |
| 2 | **closed model로 부하 주입(coordinated omission)** | 직전 iteration이 끝나야 다음이 시작 → 시스템이 느려지면 부하도 약해져, 스트레스 시점에 압력이 빠짐 | stress/spike/breakpoint는 **open model**(`*-arrival-rate`)로 도착률을 응답시간과 분리 ([k6 open vs closed](https://grafana.com/docs/k6/latest/using-k6/scenarios/concepts/open-vs-closed/)) |
| 3 | **평균 latency만 본다(p99 누락)** | 평균은 긴 꼬리를 가림("most fast, long tail much slower") | p50/p90/p95/**p99**/max를 함께 threshold에 코드화 ([SRE SLO](https://sre.google/sre-book/service-level-objectives/)) |
| 4 | **percentile을 평균/합산으로 집계** | quantile 평균은 "statistically nonsensical"; 인스턴스별 p99를 평균내면 틀림 | histogram 버킷을 합산 후 `histogram_quantile()`로 서버측 계산 ([Prometheus histograms](https://prometheus.io/docs/practices/histograms/)) |
| 5 | **histogram 버킷 경계 미설계** | **histogram은 관측값(value) 차원에서 오차를 통제**(버킷 경계로 결정)하므로, 버킷이 성기면 꼬리에서 0.94=270ms, 0.96=330ms처럼 넓은 구간 오차가 난다. (반대로 summary는 φ=rank 차원에서 오차 통제) | 관심 percentile 주변 버킷을 촘촘히 배치, OTel 권장 경계 활용 |
| 6 | **think time 없음** | 사용자가 0초 간격 연타하는 비현실적 부하 → 캐시·커넥션 패턴 왜곡 | synthetic user별 현실적 think time/도착 분포 모델링 |
| 7 | **작은/단조로운 테스트 데이터** | 동일 키 반복 → 캐시 과적중으로 latency가 비현실적으로 좋음 | 운영급 카디널리티의 시딩 데이터, idempotency·격리 |
| 8 | **cache cold/warm 혼합** | warmup 안 한 첫 구간 cold 결과와 warm 결과가 섞여 해석 불가 | warmup 구간 분리·제외, cold-start 영향은 별도 가설로 |
| 9 | **soak 생략** | 짧은 테스트는 memory/connection leak·드리프트를 못 잡음 | 주기적 장시간 soak, 시간축 회귀 기울기 모니터 |
| 10 | **여러 변수 동시 변경** | 부하·결함·코드 변경 동시 → 인과 분리 불가 | 한 실행=한 가설; chaos는 최소 blast radius부터 ([Chaos 원칙](https://cloud.google.com/blog/products/devops-sre/getting-started-with-chaos-engineering)) |
| 11 | **recovery time 미측정** | "버텼다"만 알고 ramp-down 후 회복을 모름 | ramp-down 후 정상 SLI 복귀 시간을 지표화 |
| 12 | **외부 의존성 실호출** | partner/email/SMS/payment 실호출로 비용·부작용·쿼터 폭발 | 모킹/샌드박스·destructive 차단·쿼터 가드 |
| 13 | **retry가 실패를 은폐** | 재시도로 success ratio가 좋아 보이나 사용자 latency·부하는 악화(retry storm) | error율과 별개로 retry rate·재시도 후 최종 latency 측정 |
| 14 | **threshold 없이 "그래프만 보기"** | pass/fail 자동화 불가, 회귀 탐지 누락 | SLO를 threshold로 코드화 → CI/CD non-zero exit로 게이트 |

---

## 9. 남은 불확실성 + 구현 전 재검증 항목

| 항목 | 불확실성 | 구현 직전 재검증 방법 |
|---|---|---|
| k6 옵션명/문법 | `latest` 롤링 문서라 executor/threshold 옵션(예: `delayAbortEval`, stage 표기)이 버전에 따라 바뀔 수 있음 | 사용할 k6 **버전 고정** 후 해당 버전 문서/`k6 version`으로 재확인 |
| burn rate 윈도 파라미터 | SRE Workbook 예시(5%/1h→36x, 2%/1h→14.4x 등)는 **30일** SLO 윈도 가정(공식 확인). 우리 SLO 기간·트래픽에 맞춰 재계산 필요 | 실제 SLO 기간·요청량으로 burn rate 임계·윈도 재산출 |
| client-side vs server-side percentile 정합 | k6(클라이언트 측정)와 OTel/Prometheus(서버측 histogram) percentile이 네트워크·집계 차이로 어긋날 수 있음 | 동일 구간을 양쪽에서 측정해 편차 베이스라인화, trace id로 대조 |
| histogram 버킷 설계 | OTel 기본 버킷이 우리 latency 분포(특히 꼬리)에 충분치 않을 수 있음 | 예비 측정으로 실제 분포 확인 후 관심 percentile 주변 버킷 조정 |
| 대체 관측 스택 비용/카디널리티 | 상용 APM·로그 인덱싱은 부하 트래픽에서 비용·카디널리티 폭증 가능(2차 출처 기반) | 짧은 PoC로 부하 1회분 수집량·비용·쿼리 성능 실측 |
| chaos 결함 주입 도구 | 결함 주입(network delay/error)의 구체 도구·blast radius 통제 방식 미확정 | 도구 선정 후 staging에서 최소 blast radius 검증부터 |
| Terraform/AI 자동화 경계 | 프롬프트→구조화 슬롯(§4) 매핑의 안전 기본값·승인 게이트가 환경별로 다름 | prod 부하·destructive·외부호출에 대한 승인/가드레일을 IaC 정책으로 고정 |
| 생성기 분산·지역 분포 | 단일 생성기 대역폭(1Gbit/s) 한계로 대규모/다지역 부하 시 분산 필요 | 목표 RPS×페이로드로 대역폭 산정, 다지역 분산 시 시각 동기화 확인 |
| 게임 CCU 공식·세션 파라미터 | 시간당 세션·평균 세션 길이는 타이틀별 실측 필요; CCU만으론 런치 구동 인자(session creation rate·VM deploy 속도) 누락 | 타이틀 분석 데이터로 CCU·세션 길이 산정, 런치는 세션 생성률도 함께 측정 |
| 실시간 메트릭(tick rate/state sync) | "tick rate"·"state sync latency"는 벤더 공식 메트릭으로 미확정 → `ws_ping`/`ws_msgs_*` 대리 사용 중 | 사용 엔진(Photon/Unity/전용서버) 공식 메트릭 확인 후 매핑 |
| Firebase/GCP 쿼터 수치 변동 | Firestore 단일 문서 쓰기율은 현재 "워크로드 의존"으로 하드 숫자 없음; Functions "기본 최대 인스턴스" 단일 수치는 미확정; Cloud Run revision 기본 최대 인스턴스는 100으로 확인됐지만 서비스 최대치는 regional CPU/메모리/GPU quota 영향을 받음 | 사용할 시점·플랜으로 쿼터/한계 페이지 재확인(미확정 숫자 인용 금지) |
| BaaS 관측 지연 | Firestore 메트릭 1분 샘플·최대 ~4분 반영 지연 → 짧은 테스트 실시간 판정·kill switch에 영향 | burn rate 윈도·abort 임계를 관측 지연을 고려해 재설정 |

---

## 10. 게임 도메인 표준 시나리오의 템플릿화

> 타이틀이 20+개라도 게임의 평균 유저 여정은 "**로그인 → 컨텐츠 진행 → 게임 종료**"로 수렴한다. 이 여정을 매번 새로 쓰지 않고 **파라미터화된 재사용 템플릿**으로 고정하고, 비전문가의 자연어를 사전 검증된 템플릿에 매칭한다. 템플릿에는 SLO 기본값을 내장해, 파라미터만 채워도 §5의 percentile/error budget 판정이 자동 적용되게 한다.

### 10.1 공통 골격 + 타이틀별 파라미터 슬롯

| 여정 단계 | 공통 골격(모든 타이틀 공유) | 타이틀별 파라미터 슬롯 | 내장 SLO 슬롯(기본값 예) |
|---|---|---|---|
| **로그인/인증** | 토큰 발급·세션 시작 | 인증 방식(Firebase Auth/커스텀), 토큰 TTL, 로그인 트래픽 비중 | 로그인 p95/p99 latency, 인증 실패율(error budget) |
| **컨텐츠 진행** | 읽기/쓰기 혼합 게임플레이 호출(진행 저장·랭킹/상점 조회 등) | 읽기:쓰기 비율, 컨텐츠 종류(이벤트·점검 오픈), 호출 시퀀스, **think time 분포** | 진행 API p95/p99 latency, **세션 완주율**, 쓰기 실패율 |
| **게임 종료** | 세션 정리·결과 저장·로그아웃 | 저장 페이로드 크기, 정리 호출 수, idempotency 키 | 종료 저장 성공률, error budget |

- 단계 사이에 **think time**을 둬 현실적 세션을 모델링한다. 0초 연타는 SUT가 아니라 생성기를 과부하시킨다(§11.2, §8 함정 6).
- 슬롯은 §4 입력 체크리스트와 1:1 매핑된다(여정·읽기쓰기 비율·인증·think time·데이터·SLO).

### 10.2 자연어 → 사전 검증 템플릿 매칭 원칙

| 단계 | 처리 |
|---|---|
| 입력 | 사용자는 "이벤트 오픈 때 로그인 폭주 견디는지 봐줘" 같은 **자연어만** 제공 |
| 정규화 | 시스템이 (a) 표준 여정 템플릿 + (b) 테스트 유형(§10.3) + (c) 파라미터 슬롯으로 분해(§4 슬롯 재사용) |
| 결측 보정 | 빠진 파라미터는 템플릿 내장 안전 기본값으로 채움. 단 prod·고부하·destructive·외부호출은 명시 질문/승인 게이트로 승격(§4, workflow §5) |
| 환경 범용 | 템플릿 골격은 AWS/GCP/Azure/Firebase 어디서나 동일. 차이는 파라미터(엔드포인트·인증·쿼터)와 관측 백엔드(§12)뿐 |

### 10.3 신규 컨텐츠/이벤트 오픈 → 테스트 유형 매핑

| 이벤트 상황 | 주 매핑 테스트 유형 | 단일 가설(템플릿 기본) | 근거 |
|---|---|---|---|
| 신규 이벤트·점검 오픈 직후 **동시 접속 폭주** | **Spike**(+ 직전 Load) | 초단기 급증에서 backpressure/오토스케일이 동작하고 사용자 영향이 한정된다 | k6 spike: "sudden and massive rushes", 예시 product launch(PS5) |
| 이벤트 기간 **예상 피크 동접 지속** | **Load(Average-load)** | 예상 피크 CCU에서 p95/p99·error budget 유지 | §3 Load |
| 오픈 **전 임계 용량 산출** | **Breakpoint** | 최초 SLO 위반 CCU/도착률 = X | §3 Breakpoint |
| 장기 상시 컨텐츠 | **Soak** | 장시간 누수·쿼터/비용 드리프트 없음 | §3 Soak |

> 단일 유형으로는 위험을 다 못 없앤다(k6: "No single test type eliminates all risk") → §13 프리셋으로 조합. 근거: [k6 spike testing](https://grafana.com/docs/k6/latest/testing-guides/test-types/spike-testing/).

### 10.4 템플릿에 SLO 기본값 내장 원칙 (비전문가 보호장치)

- 모든 템플릿은 latency를 **평균이 아니라 p95/p99 threshold**로, 실패를 **error budget(burn rate)**로 표현한 SLO 기본값을 내장한다(§5).
- 사용자가 SLO를 지정하지 않아도 기본 threshold가 k6 thresholds로 코드화되어 pass/fail이 자동 산출되고, 위반 시 non-zero exit/`abortOnFail`로 게이트된다(§5, [k6 thresholds](https://grafana.com/docs/k6/latest/using-k6/thresholds/)).
- 결과: 비전문가는 "**템플릿 선택 + 파라미터 입력**"만 하고, percentile/error budget 판정은 시스템이 기본 적용한다.
- 기본값은 타이틀·엔드포인트별 override 가능하되, **평균 latency 단독 기준으로의 다운그레이드는 정책으로 금지**한다.

---

## 11. 게임/실시간 부하의 방법론적 특수성

> 일반 웹 부하는 RPS(throughput) 중심이지만, 게임/실시간은 **세션 기반·동시접속(CCU) 중심**이다. k6는 부하를 두 축으로 모델링한다 — VU(동시성, closed model) vs arrival-rate(throughput, open model). 게임은 "동접 N명이 각자 세션을 유지"가 1차 부하 차원이라 CCU가 축이 된다. (출처: [k6 open vs closed](https://grafana.com/docs/k6/latest/using-k6/scenarios/concepts/open-vs-closed/))

### 11.1 CCU 중심 vs RPS 중심

| 관점 | 일반 웹(RPS 중심) | 게임/실시간(CCU·세션 중심) |
|---|---|---|
| 1차 부하 차원 | 초당 요청(arrival-rate) | 동시 접속 세션 수(VU/연결) |
| k6 모델 | open model로 throughput 고정 | closed/VU로 동접 고정 + 여정 내 호출은 think time로 페이싱 |
| 용량 산정 | 목표 RPS | **CCU = 시간당 세션 × 평균 세션 길이(초) / 3600** |
| 피크 기준 | 피크 RPS | **피크 CCU**(일 평균은 변동을 가림 → 피크 기준) |
| 핵심 주의 | — | CCU만으론 부족: 런치 시 "session creation rate·VM deployment speed"가 실제 구동 인자 |

근거: 동시접속 공식·피크 기준은 [k6 CCU 계산](https://grafana.com/docs/k6/latest/testing-guides/calculate-concurrent-users/)("Concurrent users = Hourly sessions × Average session duration / 3600"; "Daily averages can mask ... variations"), 런치 구동 인자는 [AWS GameLift](https://aws.amazon.com/blogs/gametech/amazon-gamelift-achieves-100-million-concurrently-connected-users-per-game/), CCU를 백엔드 용량 척도로 1k→1M 스케일한 사례는 [AWS GameTech 1M CCU](https://aws.amazon.com/blogs/gametech/load-testing-the-pragma-backend-game-engine-to-1-million-concurrent-users-on-aws/).

### 11.2 think time / 유저 행동 모델

- **think time** = 스크립트가 실제 사용자 지연을 흉내내려 멈추는 시간. 빼면 요청이 과도하게 빨라져 "**SUT가 아니라 생성기 자체를 과부하**"시킨다.
- 추가 기준: 현실적 유저 여정 추종 / 읽기·폼 등 시간소요 액션 모델 / 생성기 CPU > 80%일 때. 단 raw max-throughput stress에는 think time을 넣지 않는다.
- 게임 여정 템플릿(§10)은 단계 사이 think time 분포를 파라미터 슬롯으로 갖는다.

근거: [k6-learn — think time](https://github.com/grafana/k6-learn/blob/main/Modules/II-k6-Foundations/05-Adding-think-time-using-sleep.md)("Think time is the amount of time that a script pauses ... to simulate delays that real users have"; 없으면 "artificially stress the load generator itself rather than the application").

### 11.3 매치메이킹 부하 모델

매치메이킹이 있으면 일반 RPS가 아니라 **매치메이킹 특화 신호**로 본다(AWS GameLift FlexMatch):

| 분석 관점(콘솔 대시보드 그룹) | 의미 | 판정 기준(템플릿 기본) | 대응 CloudWatch 메트릭 |
|---|---|---|---|
| **Time to Ticket Success** | 매치 생성까지 평균 시간(= 매치 대기시간) | 대기시간 p95/p99 상한 | `TimeToTicketSuccess`(실제 메트릭) |
| **Player Demand** | 처리 중 요청 수·신규 요청률 → 수요 급증 **선행 감지** | 급증 구간 대기시간 회복 | `CurrentTickets`/`TicketsStarted`/`PlayersStarted`(`PlayerDemand` 단일 메트릭은 없음) |
| **Match Success/Failure Rates** | 플레이어 그룹핑 성공/실패율 | 실패율 ≤ error budget | `MatchesCreated`/`MatchesPlaced`/`TicketsFailed`/`TicketsTimedOut`로 산출(`MatchSuccessRate` 단일 메트릭은 없음) |

> **주의(정직성):** "Time to Ticket Success / Player Demand / Match Success·Failure Rates"는 GameLift **콘솔 분석 대시보드의 분류 그룹명**이며, `Player Demand`·`Match Success Rate`라는 **CloudWatch 메트릭은 존재하지 않는다**. 실제 CloudWatch 메트릭은 `TicketsStarted`/`TicketsFailed`/`TicketsTimedOut`/`MatchesCreated`/`MatchesPlaced`/`MatchesAccepted`/`PlayersStarted`/`TimeToMatch`/`TimeToTicketSuccess` 등 CamelCase ID다. 구현 시 위 분석 관점을 이들 실제 메트릭으로 매핑한다.

근거: [AWS GameLift FlexMatch 블로그(대시보드 그룹)](https://aws.amazon.com/blogs/gametech/matchmaking-your-way-amazon-gamelift-flexmatch-and-game-session-queues/), [GameLift CloudWatch 메트릭(실제 메트릭명)](https://docs.aws.amazon.com/gameliftservers/latest/developerguide/monitoring-cloudwatch.html).

### 11.4 실시간 상태동기화 / 지속 연결(WebSocket) 부하

- 지속 연결은 구조가 다르다: 각 VU가 연결을 열고 **연결 수명 동안 비동기 이벤트 루프**를 돈다 → **부하 단위는 요청률이 아니라 연결 수**다.
- 측정 신호(k6 내장 WS 메트릭):

| 메트릭 | 의미 | 부하 판정 |
|---|---|---|
| `ws_sessions` | 시작된 WebSocket 세션 수 | 동시 연결/세션 수 유지 |
| `ws_connecting` | 연결 수립 소요 시간 | 수립 latency p95/p99 |
| `ws_session_duration` | 세션 지속 시간 | 조기 종료/끊김 탐지 |
| `ws_msgs_sent` / `ws_msgs_received` | 연결당 송수신 메시지 수 | 연결당 메시지 throughput |
| `ws_ping` | ping↔pong 왕복 시간(= 메시지 왕복 지연) | 왕복 지연 p95/p99 |

- **정직성 주의:** "tick rate", "state sync latency"는 특정 벤더 공식 메트릭으로 확정 검증하지 못했다 → `ws_ping`(왕복 지연)·`ws_msgs_*`(처리량)를 **검증된 대리 지표**로 사용한다(§9 재검증).

근거: [k6 WebSockets](https://grafana.com/docs/k6/latest/using-k6/protocols/websockets/)("asynchronous event loop"), [k6 metrics reference](https://grafana.com/docs/k6/latest/using-k6/metrics/reference/)(ws_* 정의).

---

## 12. BaaS(Firebase 등) 대상 SLO/관측 특수성

> 대상이 관리형 BaaS면, 부하의 "병목"이 우리 코드가 아니라 **서비스 쿼터/요금 한계**일 수 있다. SLO·함정·관측이 달라진다. 관측은 내부 계측이 아니라 **클라우드 네이티브 제공자 메트릭**(Cloud Monitoring)에 의존한다.

### 12.1 BaaS에서 "병목 = 쿼터/요금 한계"

| 영역 | 문서화된 한계/규칙 | 부하 테스트 함의 |
|---|---|---|
| **Firestore 트래픽 증가** | "500/50/5" 규칙: 새 컬렉션에 **최대 500 ops/sec로 시작, 5분마다 50%씩 증가**(점진 램프). hotspotting=사전식 인접 문서 고RPS→contention | **Spike를 Firestore에 직접 주면 앱이 아니라 Firestore ramp-up 한계에 막혀** RESOURCE_EXHAUSTED 가능 → 급증 가설을 ramp 규칙과 분리 해석 |
| **Firestore 단일 문서 쓰기율** | "워크로드에 따라 다름"(현재 공식은 하드 숫자 미명시; 구식 "1/sec"는 인용 금지) | 핫 문서 경합을 쓰기 패턴 함정으로 별도 검증 |
| **Firestore 일일 쿼터** | "hard limits unless otherwise noted"; 무료 등급 읽기 50,000·쓰기 20,000·삭제 20,000 per day(태평양 자정 리셋) | 부하 중 쿼터 소진 = RESOURCE_EXHAUSTED("free tier 초과+billing off" 또는 "daily quota/ramp 초과") — app 5xx 아님 |
| **RTDB 연결/쓰기** | 인스턴스당 동시 연결 **200,000**(Blaze; Spark 100)·쓰기 **1,000/sec**(소프트, 초과 시 rate-limit)·동시 응답 ~100,000/sec | CCU 부하가 인스턴스 한계에 닿으면 샤딩 필요(프로젝트당 최대 1,000 인스턴스) |
| **Functions/Run cold start·동시성·scale cap** | 무트래픽 시 scale-to-zero(기본), cold start 시 글로벌 컨텍스트 재평가; **min instances**로 완화; 기본 동시성 80×vCPU(최대 1000), Cloud Run revision 기본 max instances 100, 서비스 최대치는 regional quota 영향 | spike 초반 cold start와 max instance cap이 latency/429를 유발 → min instances·동시성·max instances를 별도 가설로 분리(§3 Spike) |
| **Functions 쿼터** | (1세대) 최대 동시 호출 3,000(상향 가능)·이벤트율 1,000/sec(상향 불가)·동시 이벤트 데이터 10MB | 한계 도달 = 우리 코드가 아닌 플랫폼 쿼터 |

근거: [Firestore best practices](https://cloud.google.com/firestore/docs/best-practices), [Firestore quotas](https://firebase.google.com/docs/firestore/quotas), [Firestore error codes](https://cloud.google.com/firestore/native/docs/understand-error-codes), [RTDB limits](https://firebase.google.com/docs/database/usage/limits)·[sharding](https://firebase.google.com/docs/database/usage/sharding), [Cloud Run cold start](https://cloud.google.com/run/docs/tips/functions-best-practices)·[concurrency](https://cloud.google.com/run/docs/about-concurrency)·[configuration](https://docs.cloud.google.com/run/docs/configuring)·[max instances](https://docs.cloud.google.com/run/docs/configuring/max-instances-limits), [Functions quotas](https://firebase.google.com/docs/functions/quotas).

### 12.2 BaaS용 SLO 지표 보강 (§5에 더하는 사용량 기반 지표)

| 지표 | 의미 | 판정/주의 |
|---|---|---|
| **RESOURCE_EXHAUSTED / 429 rate** | 쿼터·ramp 초과 신호 | error budget로 카운트. "지수 백오프 재시도" 안내 자체가 쿼터 병목 증거(retry storm 주의 §8-13) |
| **Firestore Document Reads/Writes/Deletes** | 과금·쿼터 소비량·추세 | 부하 1회분 사용량을 비용으로 환산(비용 상한 §4) |
| **Active Connections / Snapshot Listeners** | 동시 연결·실시간 리스너 수 | CCU 부하의 BaaS측 분모 |
| **RTDB 동시 연결 / writes per second** | 인스턴스 한계 대비 여유 | 200k/1k 대비 burn, 샤딩 임계 |
| **Functions execution_count·execution_times·active_instances·user_memory** | 실행(상태별)·지연 분포·인스턴스·메모리 | execution_times **분포**로 p95/p99(평균 금지); active_instances로 cold start/스케일 관찰 |

### 12.3 관측: 클라우드 네이티브 의존

- 관리형 BaaS는 내부를 들여다볼 수 없다 → **제공자 메트릭(Cloud Monitoring)에 의존**. Firestore: Document Reads/Writes/Deletes·Active Connections·Snapshot Listeners(**1분 샘플, 대시보드 반영 최대 ~4분 지연**). Functions: execution_count/times/active_instances/user_memory(Metrics Explorer).
- **함의:** 관측 지연(~4분)을 kill switch·burn rate 윈도 설계에 반영한다(짧은 테스트에서 늦은 신호 주의, §5 burn rate·§9).
- §6·§7의 OTel/Prometheus 위에 클라우드 네이티브(Cloud Monitoring/CloudWatch/Azure Monitor)를 **보강**해 BaaS 쿼터·사용량을 같은 시간축에 정렬한다.

근거: [Firestore monitor usage](https://firebase.google.com/docs/firestore/monitor-usage), [Cloud Functions metrics](https://cloud.google.com/functions/docs/monitoring/metrics)·[GCP metrics 목록](https://cloud.google.com/monitoring/api/metrics_gcp_c).

---

## 13. "이벤트 오픈 대비" 표준 프리셋 권고

> 기획자의 실제 니즈 = "컨텐츠/이벤트 오픈 **전에** 부하를 예측해 라이브 장애를 막는다". 단일 유형으론 부족(k6: "No single test type eliminates all risk")하므로 표준 프리셋으로 조합한다.

### 13.1 "이벤트 오픈 대비" 프리셋 (조합)

| 순서 | 테스트 유형 | 산출물 | 통과 게이트 |
|---|---|---|---|
| 1 | **Load**(예상 피크 CCU) | 예상 피크에서 p95/p99·error budget 충족 여부 | SLO 충족 시 다음 단계 |
| 2 | **Spike**(오픈 순간 급증) | 급증 시 오토스케일/큐/backpressure·cold start 영향 | 사용자 영향 한정·복구 확인 |
| 3 | **Breakpoint**(임계 용량) | 최초 SLO 위반 CCU/도착률 = **임계 용량값** | 예상 피크 대비 헤드룸 산정 |
| (옵션) | **Soak**(장기 이벤트) | 누수·쿼터/비용 드리프트 없음 | 기울기 ≈ 0 |

- **BaaS 대상이면** 각 단계 통과 게이트에 §12 쿼터 가드(ramp 규칙·RESOURCE_EXHAUSTED·연결/쓰기 한계)를 포함한다.
- 프리셋 파라미터: 예상 피크 CCU(= 시간당 세션 × 세션 길이 / 3600), think time 분포, 읽기:쓰기 비율, 대상 환경/쿼터.

### 13.2 비전문가 보호장치: 프리셋에 SLO 기본값 내장

- 프리셋은 §10.4 원칙대로 SLO 기본값(p95/p99 threshold + error budget/burn rate)을 내장 → 사용자는 "**이벤트 오픈 대비**" 프리셋 선택 + 예상 피크 CCU만 입력한다.
- 판정은 자동으로 percentile/error budget 중심. 평균 latency로의 다운그레이드는 금지.
- 산출 리포트: 예상 피크 통과 여부 + 임계 용량(헤드룸) + 급증 복구 + (BaaS면) 쿼터 여유. 경영 요약 ↔ 기술 상세로 분리(workflow §3 Report Composer).

---

### 부록 A. 핵심 정의 인용(원문 근거 요약)

- **Thresholds:** "Thresholds are the pass/fail criteria that you define for your test metrics." threshold 실패 → non-zero exit. "testers use thresholds to codify their SLOs." — [k6](https://grafana.com/docs/k6/latest/using-k6/thresholds/)
- **Open vs Closed:** closed="VU iterations start only when the last iteration finishes"(→ coordinated omission), open="the response times of the target system no longer influence the load on the target system." — [k6](https://grafana.com/docs/k6/latest/using-k6/scenarios/concepts/open-vs-closed/)
- **생성기 병목:** "If k6 uses 100% of the CPU ... result metrics to have a much larger response time than in reality"; 네트워크 1Gbit/s 상한은 `iftop`로 확인. — [k6](https://grafana.com/docs/k6/latest/testing-guides/running-large-tests/)
- **SLI/SLO & 분포:** SLI="ratio of good events / total events"; 평균은 "a long tail of requests"를 가림; 99th=plausible worst-case, 50th=typical. — [Google SRE](https://sre.google/sre-book/service-level-objectives/)
- **Error budget/burn rate:** "Burn rate is how fast, relative to the SLO, the service consumes the error budget." multiwindow multi-burn-rate 권장. — [SRE Workbook](https://sre.google/workbook/alerting-on-slos/)
- **Golden Signals:** latency/traffic/errors/saturation; "latency increases are often a leading indicator of saturation." — [SRE Book](https://sre.google/sre-book/monitoring-distributed-systems/)
- **OTel signals/semconv:** traces/metrics/logs/baggage; `http.server.request.duration`=Histogram(`s`). — [OTel signals](https://opentelemetry.io/docs/concepts/signals/), [OTel HTTP metrics](https://opentelemetry.io/docs/specs/semconv/http/http-metrics/)
- **Prometheus quantile 주의:** summary는 φ(rank) 차원 오차, histogram은 관측값(value/버킷) 차원 오차를 가진다. quantile 평균은 "BAD!"; histogram=서버측 `histogram_quantile()` 집계 가능. — [Prometheus](https://prometheus.io/docs/practices/histograms/)
- **Chaos 원칙:** steady-state hypothesis, 최소 blast radius부터 확장, 실제 장애 모델링. — [Google Cloud](https://cloud.google.com/blog/products/devops-sre/getting-started-with-chaos-engineering)
- **게임 CCU/세션:** "Concurrent users = Hourly sessions * Average session duration (in seconds) / 3600"; 일 평균은 변동을 가리므로 피크 기준. CCU만으론 부족 — 런치 구동 인자는 "game session creation rate and VM deployment speed". — [k6 CCU](https://grafana.com/docs/k6/latest/testing-guides/calculate-concurrent-users/), [AWS GameLift](https://aws.amazon.com/blogs/gametech/amazon-gamelift-achieves-100-million-concurrently-connected-users-per-game/)
- **think time:** "Think time is the amount of time that a script pauses ... to simulate delays that real users have"; 없으면 "artificially stress the load generator itself rather than the application." — [k6-learn](https://github.com/grafana/k6-learn/blob/main/Modules/II-k6-Foundations/05-Adding-think-time-using-sleep.md)
- **매치메이킹:** FlexMatch "Time to Ticket Success"·"Player Demand"·"Match Success/Failure Rates"는 **콘솔 분석 대시보드 그룹명**이다(실제 CloudWatch 메트릭은 `TimeToTicketSuccess`/`TicketsStarted`/`MatchesCreated` 등 — §11.3). — [AWS FlexMatch 블로그](https://aws.amazon.com/blogs/gametech/matchmaking-your-way-amazon-gamelift-flexmatch-and-game-session-queues/), [GameLift CloudWatch 메트릭](https://docs.aws.amazon.com/gameliftservers/latest/developerguide/monitoring-cloudwatch.html)
- **WebSocket 부하 단위:** 각 VU가 "asynchronous event loop"로 연결 수명 동안 유지 → 부하 단위=연결; `ws_ping`="Duration between a ping request and its pong reception". — [k6 WebSockets](https://grafana.com/docs/k6/latest/using-k6/protocols/websockets/), [k6 metrics](https://grafana.com/docs/k6/latest/using-k6/metrics/reference/)
- **Spike=런치/이벤트:** "A spike test verifies whether the system survives ... sudden and massive rushes"; 예 product launch(PS5). — [k6 spike](https://grafana.com/docs/k6/latest/testing-guides/test-types/spike-testing/)
- **Firestore 램프 규칙:** "starting with a maximum of 500 operations per second to a new collection and then increasing traffic by 50% every 5 minutes"; 초과 시 RESOURCE_EXHAUSTED("retry with exponential backoff"). — [Firestore best practices](https://cloud.google.com/firestore/docs/best-practices), [Firestore error codes](https://cloud.google.com/firestore/native/docs/understand-error-codes)
- **RTDB 한계:** 인스턴스당 동시 연결 200,000(Spark 100)·쓰기 1,000/sec(소프트); 초과 시 샤딩(프로젝트당 최대 1,000 인스턴스). — [RTDB limits](https://firebase.google.com/docs/database/usage/limits), [RTDB sharding](https://firebase.google.com/docs/database/usage/sharding)
- **Functions/Run cold start:** "the execution environment is often initialized from scratch, which is called a cold start"; min instances로 완화·기본 scale-to-zero·기본 동시성 80×vCPU(최대 1000)·Cloud Run revision 기본 max instances 100. — [Cloud Run best practices](https://cloud.google.com/run/docs/tips/functions-best-practices), [min-instances](https://cloud.google.com/run/docs/configuring/min-instances), [concurrency](https://cloud.google.com/run/docs/about-concurrency), [configuration](https://docs.cloud.google.com/run/docs/configuring), [max instances](https://docs.cloud.google.com/run/docs/configuring/max-instances-limits)
- **BaaS 관측:** Firestore Cloud Monitoring 메트릭(Document Reads/Writes/Deletes·Active Connections·Snapshot Listeners; "sampled every minute ... up to 4 minutes" 지연); Functions execution_count/execution_times/active_instances. — [Firestore monitor usage](https://firebase.google.com/docs/firestore/monitor-usage), [Cloud Functions metrics](https://cloud.google.com/functions/docs/monitoring/metrics)

> 모든 fetched_at: 2026-06-26. 외부 문서는 데이터로만 취급함.
