# 11. 로컬 컨테이너 인프라 — 로컬 테스팅 & 클라우드 구조 사전 검증

> 작성일: 2026-06-27 (Asia/Seoul). 기준일(웹 검증): 2026-06-27.
> 목적: 클라우드 실행이 **주목적**이되, 그 전에 **컨테이너(Docker 등)로 로컬에서** (a) 목 타깃에 실제 부하를 가하고, (b) 클라우드 IaC·오케스트레이션 **구조**를 비용 0으로 사전 검증하는 방법을 정리한다.
> 정합성: `docs/plan/05-runners-and-mock-target.md`(러너/목서버), `docs/report.md` §0·§2(M0에서 확정된 `docker/compose.local.yml`의 `app`/`mock-target`/`runner-tools` 3서비스, host-native+Docker 병행은 plan/10 B6), `docs/research/02`(실행 루프=K8s Job), `docs/research/04`(LGTM 관측 스택), `docs/plan/10` B6/B7, `docs/plan/12-execution-topology-matrix.md`의 L0/L1 토폴로지를 확장한다.
> 모든 외부 사실은 웹 검색·공식 문서로 확인했다(§12 출처). 본 문서는 자료 수집·구성안이며 실제 IaC/코드는 포함하지 않는다.

---

## 0. 핵심 원칙 (한 줄)

**로컬 컨테이너는 두 가지를 한다 — ① L0에서 목 타깃에 "실제 부하"를 가해 러너·리포터 파이프라인을 실측 검증, ② L1/C0 전 단계에서 클라우드 IaC·실행평면 "구조"를 에뮬레이터/로컬 K8s로 비용 0 검증.** 단 **에뮬레이터는 구조·정확성용이지 부하 측정용이 아니다**(Firebase 공식: "built for accuracy, not performance"). 즉 **로컬에서 측정하는 부하는 오직 번들 목 타깃 대상**이고, 클라우드 에뮬레이터는 "plan/구조/계약이 맞는가"만 본다. 실제 용량·SLO 측정은 클라우드 단계의 몫이다.

---

## 1. 왜 컨테이너인가 (기획 공백 → 해결)

| 기존 공백(앞선 분석) | 컨테이너로 해결 |
|---|---|
| 05: k6/Artillery 바이너리 설치 "비범위", 버전 고정 주체 불명 | `runner-tools` 컨테이너에 **버전 고정된** k6/Artillery 이미지 → 재현성·설치 자동화 |
| 05: 목 서버 기술 스택 미정 | 목 타깃을 컨테이너 서비스로 고정(FastAPI/Starlette), compose로 기동 |
| 10 B6/B7: host-native+Docker 병행, 토폴로지 명시 | 본 문서가 L0/L1 self-hosted container 구성안을 구체화 |
| 12: L0/L1 토폴로지의 실행 substrate 구분 필요 | L0는 local workstation/container, L1은 remote self-hosted container로 구분 |
| 클라우드 IaC를 로컬에서 검증할 경로 부재(M5 전까지 구조 확인 불가) | LocalStack/에뮬레이터 + `terraform test`로 **plan/구조를 로컬에서** 사전 검증 |
| 02: 실행 루프=K8s Job인데 로컬 검증 수단 없음 | kind/k3d 로컬 K8s에서 Job·타임아웃·정리·kill 구조 검증 |

---

## 2. 로컬 컨테이너 토폴로지 (3계층)

```text
┌─ Layer A. 코어 앱 스택 (실제 부하 가능) ───────────────────────┐
│  app(FastAPI+React) ── mock-target(목 타깃, 유일한 실부하 대상)    │
│        │                     ▲                                   │
│        └─ runner-tools(k6/Artillery 버전고정) ──부하──┘           │
│        └─ observability(Grafana+Prometheus+Tempo+Loki = LGTM)    │
└─────────────────────────────────────────────────────────────────┘
┌─ Layer B. 실행 평면 구조 검증 (부하 아님, 구조만) ───────────────┐
│  kind 또는 k3d (로컬 K8s) → runner를 K8s Job으로 기동             │
│   completions/parallelism/backoffLimit/activeDeadlineSeconds/    │
│   ttlSecondsAfterFinished = kill switch·타임아웃·정리 구조 검증   │
└─────────────────────────────────────────────────────────────────┘
┌─ Layer C. 클라우드 IaC 구조 검증 (비용 0, 부하 아님) ────────────┐
│  AWS  : LocalStack + tflocal (ECS/Fargate/S3/DynamoDB/Cognito)   │
│  GCP/Firebase : Firebase Emulator Suite + GCP 에뮬레이터(Pub/Sub)│
│  Azure: Azurite(스토리지) + terraform validate/plan(ALT는 에뮬 無)│
│  검증기: terraform test(command=plan) · Terratest · checkov/OPA  │
└─────────────────────────────────────────────────────────────────┘
```

- **Layer A만이 실제 부하를 만든다**(목 타깃 대상). Layer B·C는 "구조·계약·정책이 맞는가"만 검증한다. 전체 배포/Provider 조합 정본은 [[12-execution-topology-matrix]]를 따른다.
- 제어/실행 평면 분리·승인 게이트·plan_hash 불변식은 로컬에서도 그대로 유지한다(컨테이너가 이를 약화시키지 않는다).

---

## 3. 러너 컨테이너 (k6 / Artillery)

| 항목 | 구성 |
|---|---|
| k6 이미지 | 공식 `grafana/k6`(Docker Hub). 태그로 **버전 고정**(`tools/versions.json`과 일치). k6 2.0의 JSON summary output을 정본 입력으로 |
| Artillery 이미지 | 공식 `artilleryio/artillery`. 버전 고정 |
| 호출 방식 | Execution Controller가 `docker compose run --rm runner-tools k6 run ...`(또는 `docker run`)으로 격리 실행 → 재현성·정리 자동. host-native 경로는 빠른 개발 루프용 |
| 부하 결과 | k6 `--out experimental-prometheus-rw`(Prometheus remote write)로 관측 스택에 푸시 + `--summary-export`/JSON summary를 collector(06)가 파싱 |
| 생성기 병목 | 컨테이너 리소스 제한(`--cpus`/`--memory`) + psutil/컨테이너 stats로 CPU>80%·`dropped_iterations` 감시(04 §8 함정 1) |

> 공식 k6 docker-compose 예시(`grafana/xk6-output-prometheus-remote`)가 k6+Prometheus+Grafana 일체 구성을 제공한다. `K6_PROMETHEUS_RW_SERVER_URL`로 remote write 타깃 지정.

---

## 4. 목 타깃 컨테이너 (유일한 로컬 실부하 대상)

- **스택**: FastAPI/Starlette 단일 앱(이미 백엔드 스택과 동일, [[05-runners-and-mock-target]] §6.1 확정). WS `/realtime`은 Starlette WebSocket.
- **결정성**: 시드 고정 RNG(`random.Random(seed)`) + 세마포어/토큰버킷으로 지연분포·M/M/1 큐·에러율·동시성을 주입(요청 헤더/엔드포인트로 제어). 같은 입력 → 같은 출력(골든 테스트).
- **컨테이너화 이점**: 리소스 제한으로 "SUT 한계"를 의도적으로 만들어 stress/breakpoint/생성기-병목 분리를 로컬에서 재현. compose 서비스라 1커맨드 기동.
- **불변식**: 로컬 부하는 **목 타깃에만**. 외부/실서비스 엔드포인트는 정책으로 금지(05 §9, 04 §2.4 매핑표).

---

## 5. 관측 스택 컨테이너 (LGTM)

리서치 04 §7의 Grafana 생태계를 로컬 compose로 구성한다.

| 컴포넌트 | 역할 | 비고 |
|---|---|---|
| **Prometheus** | k6 remote write 수신·메트릭 | k6 `experimental-prometheus-rw` 타깃, `--web.enable-remote-write-receiver` |
| **Grafana** | 대시보드 | 공식 "k6 Prometheus" 대시보드(ID 19665) 임포트 |
| **Tempo** | traces | 목 타깃 OTel trace ↔ k6 메트릭 시간축 정렬 |
| **Loki** | logs | 부하 구간 에러 로그 정렬 |

> 목적: report의 "observability 통합 데모" 스토리(k6 threshold→exit 99 게이트 + Prometheus/OTel→Grafana)를 **클라우드 없이 로컬에서** 완성. soak·장기 추세는 Mimir로 확장 가능(운영 복잡도↑).

---

## 6. 실행 평면 로컬 검증 — kind / k3d (구조만)

리서치 02는 "장기 부하 루프·재시도·kill switch·정리는 Terraform이 아니라 **K8s Job/Operator**"라고 결론냈다. 이 실행평면 구조를 클라우드 전에 로컬 K8s에서 검증한다.

| 도구 | 성격 | 선택 기준 |
|---|---|---|
| **kind** (Kubernetes in Docker) | kubeadm 기반 정식 K8s, 프로덕션 유사, **CI 표준** | 실행평면을 "실제에 가깝게" 검증·CI 게이트 |
| **k3d** (k3s in Docker) | 경량(etcd→SQLite), 빠름·저자원 | 일상 개발 루프 |

검증 대상(클라우드 가기 전 로컬에서 확인):
- K8s **Job**: `completions`/`parallelism`(팬아웃), `backoffLimit`(재시도), `activeDeadlineSeconds`(하드 타임아웃=kill), `ttlSecondsAfterFinished`(자동 정리).
- runner 이미지를 Job으로 띄워 **kill switch·timeout·teardown** 동작을 실측(목 타깃 대상).
- Terraform `kubernetes`/`helm` provider로 로컬 클러스터에 러너 인프라를 `plan/apply`해 **IaC 구조**를 검증(아래 §7과 연결).

---

## 7. 클라우드 IaC 구조 로컬 검증 (비용 0, 부하 아님)

> 핵심: 여기서 검증하는 것은 **부하가 아니라 IaC plan/apply 구조·정책·승인 게이트·러너 인프라 토폴로지**다. "리소스가 의도대로 만들어지는가, 정책에 걸리는가, plan_hash↔승인이 결속되는가"를 클라우드 비용 없이 본다.

### 7.1 AWS — LocalStack + tflocal

- **LocalStack**: 로컬 AWS 에뮬레이터(컨테이너). `tflocal`(Terraform 래퍼)이 provider endpoint를 LocalStack으로 override → `terraform plan/apply`를 로컬에서 실행.
- **용도**: 러너 인프라(ECS/Fargate·S3·DynamoDB·IAM·Cognito) 및 AWS DLT 래퍼 IaC의 **구조·의존·정책**을 검증. DLT는 CloudFormation/CDK-first(네이티브 TF 없음)이므로, TF는 계정/네트워크/보조 리소스 구조 검증에 사용.
- **주의**: ECS/Fargate 등 일부 서비스는 **LocalStack Pro 티어**일 수 있음 → 채택 전 티어·파리티 확인. LocalStack은 "완전 파리티"가 아니다(실제 AWS 동작과 차이 가능).

### 7.2 GCP / Firebase — Emulator Suite + GCP 에뮬레이터

- **Firebase Local Emulator Suite**: Firestore·RTDB·Auth·Cloud Functions(beta)·Pub/Sub(beta)·Storage·Hosting 에뮬레이터. **공식 명시: "정확성용이지 성능/보안용 아님"** → **부하 측정 불가**. 용도는 부하 시나리오의 **기능·구조·인증 흐름 검증**(예: "로그인→컨텐츠→종료" 여정이 구조적으로 맞는가)과 CI 통합 테스트.
- **GCP 에뮬레이터**: Firestore 에뮬레이터(`gcloud`, `FIRESTORE_EMULATOR_HOST`), Pub/Sub 에뮬레이터(컨테이너). `giglocal`·`floci-gcp` 같은 다중 GCP 에뮬레이터 이미지, Testcontainers GCloud 모듈도 활용 가능.
- **결론**: GCP/Firebase는 로컬에서 **구조·정확성만** 검증. 실제 BaaS 부하·쿼터·과금 측정은 테스트 전용 프로젝트(클라우드)에서만(05 §9 격리 원칙).

### 7.3 Azure — Azurite + plan/validate

- **Azurite**: 공식 Azure Storage 에뮬레이터(컨테이너). 스토리지 의존 부분 로컬 검증.
- **Azure Load Testing은 로컬 에뮬레이터가 없다**(관리형 서비스). 따라서 `azurerm_load_test` 등 ALT 리소스는 `terraform validate` + `terraform plan` + `terraform test`(plan)로 **구조만** 검증하고, 실제 실행은 클라우드.

### 7.4 IaC 검증기 (plan 단계 — 실제 리소스 생성 없음)

| 도구 | 무엇 | 로컬 검증 위치 |
|---|---|---|
| `terraform validate`/`fmt` | 구문·포맷 | 모든 IaC |
| **`terraform test`**(native, GA 1.6) | `command = plan`이면 실제 인프라 생성 없이 단위 검증, 1.7부터 provider/리소스 **mocking** | plan 단계 계약 검증 |
| **tflocal + LocalStack** | TF를 LocalStack에 apply해 AWS 구조 실측 | AWS 러너 인프라 |
| **Terratest**(Go) | 프로비저닝→검증→정리 자동화, LocalStack 연동 | 통합·스택 레벨 |
| **Testcontainers**(LocalStack/GCloud 모듈) | 테스트 코드에서 에뮬레이터 컨테이너 기동 | 통합 테스트 |
| **Checkov / OPA** | 정책·보안 baseline(plan/HCL) | PR·plan 게이트(05 §12 조합) |

> 권장 흐름(05 §12와 정합): PR단계 **Checkov** → plan단계 **`terraform test`(plan)+OPA/tflocal** → (선택) **Terratest/Testcontainers**로 LocalStack 통합. 전부 **로컬·CI에서 비용 0**, 실제 apply는 클라우드 승인 게이트 후.

---

## 8. docker compose 구성안 (프로파일 분리)

`docker/compose.local.yml`(report §2 M0에서 확정)을 **compose profiles**로 계층화한다.

| profile | 서비스 | 용도 |
|---|---|---|
| `core` (기본) | `app`, `mock-target`, `runner-tools` | 로컬 실부하 + 파이프라인(M0/M4) |
| `observability` | `prometheus`, `grafana`, `tempo`, `loki` | 관측 데모(§5) |
| `cloud-emul` | `localstack`, `firebase-emulators`, `pubsub-emulator`, `azurite` | IaC 구조 검증(§7) |

- 기동 예: `docker compose --profile core up` / `... --profile core --profile observability up`.
- 로컬 K8s(kind/k3d)는 compose 밖의 별도 스크립트(`make kind-up`)로 둔다(Layer B, §6).

---

## 9. 마일스톤 매핑 (기존 08 로드맵에 삽입)

| 마일스톤 | 컨테이너 작업 |
|---|---|
| **M0** | `docker/compose.local.yml`(`core` profile) + `tools/versions.json` 버전 고정 + CI Docker smoke |
| **M4** | `runner-tools`로 k6/Artillery 실부하, `observability` profile, 목 타깃 컨테이너 실측 |
| **M4.5-a**(신규·선택) | kind/k3d로 **실행평면 Job 구조 검증**(타임아웃·정리·kill) — 클라우드 전 비용 0 리허설 |
| **M4.5-b**(선택) | 동일 구조를 remote self-hosted container(L1)로 확장해 source IP·allowlist·egress 조건을 검증([[08-milestones-roadmap]], [[12-execution-topology-matrix]]) |
| **M5 전제** | `cloud-emul` profile + `terraform test`(plan)/tflocal로 **클라우드 IaC 구조 사전 검증** → 실제 apply는 M5 클라우드에서 승인 후 |

---

## 10. 한계·주의 (정직성)

- **에뮬레이터 ≠ 부하 측정.** Firebase Emulator Suite는 공식적으로 "정확성용, 성능/보안용 아님". GCP/LocalStack 에뮬레이터도 동일 — **로컬에서 측정하는 부하는 목 타깃 한정**, 클라우드 에뮬은 구조·정확성만.
- **LocalStack 파리티 불완전 + 일부 서비스 Pro 티어**(ECS/Fargate 등). 로컬 plan이 통과해도 실제 클라우드 차이 가능 → 클라우드 단계 재검증 필수.
- **로컬 K8s(kind/k3d) ≠ 실제 클라우드 스케일·네트워크.** Job 구조·수명주기 검증용이지 용량 측정용 아님.
- **Azure Load Testing·AWS DLT는 로컬 에뮬레이터가 없다**(관리형). 구조는 `terraform plan/test`로만, 실행은 클라우드.
- 따라서 로컬 컨테이너 단계의 산출은 **"구조·계약·정책 검증 + 목 타깃 실부하"**이고, **실제 용량·SLO·비용은 클라우드 단계**에서 측정한다. 이 경계를 리포트에 명시한다.

---

## 11. 결론

컨테이너는 이 프로젝트에서 두 빈틈을 동시에 메운다 — (1) 러너/목타깃 **재현성·설치 자동화**(05 바이너리 공백 해소), (2) 클라우드로 가기 전 **IaC·실행평면 구조를 비용 0으로 리허설**. 클라우드가 주목적이라는 전제는 유지하되, "로컬에서 구조가 맞는지 + 목 타깃에 실부하가 도는지"를 먼저 닫고 클라우드로 승격하는 것이 안전·비용 면에서 합리적이다. 단 **에뮬레이터로 부하를 측정하지 않는다**는 경계를 끝까지 지킨다.

---

## 12. 자료 출처 (2026-06-27 웹 검증)

- k6 Docker/Prometheus: [grafana/k6 (Docker Hub)](https://hub.docker.com/r/grafana/k6) · [k6 Prometheus remote write](https://grafana.com/docs/k6/latest/results-output/real-time/prometheus-remote-write/) · [xk6-output-prometheus-remote docker-compose](https://github.com/grafana/xk6-output-prometheus-remote/blob/main/docker-compose.yml)
- LocalStack/Terraform: [tflocal (terraform-local)](https://github.com/localstack/terraform-local) · [LocalStack Terraform 통합](https://docs.localstack.cloud/aws/integrations/infrastructure-as-code/terraform/) · [LocalStack + Terraform tests framework](https://blog.localstack.cloud/efficient-infrastructure-testing-localstack-terraform-tests-framework/)
- Terraform 테스트: [Terraform Tests (공식)](https://developer.hashicorp.com/terraform/language/tests) · [Terratest](https://terratest.gruntwork.io/)
- Firebase/GCP 에뮬레이터: [Firebase Local Emulator Suite](https://firebase.google.com/docs/emulator-suite) · [Firestore emulator](https://docs.cloud.google.com/firestore/native/docs/emulator) · [Pub/Sub emulator](https://docs.cloud.google.com/pubsub/docs/emulator) · [Testcontainers GCloud 모듈](https://java.testcontainers.org/modules/gcloud/)
- 로컬 K8s: [kind](https://kind.sigs.k8s.io/) · [k3d](https://k3d.io/) · [K8s Job (research/02 근거)](https://kubernetes.io/docs/concepts/workloads/controllers/job/)
