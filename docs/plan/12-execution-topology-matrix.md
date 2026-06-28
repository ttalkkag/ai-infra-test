# 12 · 실행 토폴로지 매트릭스 — 로컬/클라우드 조합 정본

> 작성일: 2026-06-27 (Asia/Seoul). 기준일(웹 검증): 2026-06-27.
> 목적: "로컬 버전"과 "클라우드 버전"을 단일 플래그가 아니라 **제어 평면 위치 / 러너 위치 / 대상 위치 / Provider 경계 / 네트워크 경로**로 분해해 구현 계획의 혼동을 없앤다.
> 본 문서는 계획 문서다. 실제 Dockerfile, Terraform, Kubernetes manifest, cloud apply 코드는 작성하지 않는다.

---

## 0. 핵심 결론

**로컬/클라우드는 이분법이 아니다.** 이 프로젝트에서 "로컬"은 **내가 통제하는 host/container/K8s에서 러너를 실행**한다는 뜻이고, 그 host는 개발자 맥북일 수도 있고 AWS EC2 같은 클라우드 VM일 수도 있다. "클라우드"는 **Cloud Provider의 관리형 부하/러너/인프라 자원**을 사용한다는 뜻이다.

따라서 구현 계획은 다음 5개 축을 별도로 기록해야 한다.

| 축 | 예시 | 왜 분리하나 |
|---|---|---|
| 제어 평면 위치 | local workstation, self-hosted server, cloud service | 웹 앱/API가 어디서 계획·승인·audit을 수행하는가 |
| 러너 위치 | local-host, local-container, remote-container, cloud-managed, cloud-k8s-job | 실제 부하 생성기가 어디서 돈다는 뜻인가 |
| 대상 위치/Provider | mock, AWS, GCP/Firebase, Azure, on-prem | 부하를 받는 시스템이 어디에 있는가 |
| IaC/자원 Provider | none/local, AWS, GCP, Azure, Grafana Cloud, mixed | 어떤 Provider에 plan/apply 권한이 필요한가 |
| 네트워크 경로 | same-host, same-vpc, public-internet, private-link, cross-cloud-public/private | 지연·egress 비용·보안·allowlist 판단 기준 |

---

## 1. 표준 모드

| 모드 | 제어 평면 | 러너 위치 | 대상 | Provider/IaC | 목적 | 구현 단계 |
|---|---|---|---|---|---|---|
| **L0 Local Workstation** | 개발자 로컬 앱 | host-native 또는 Docker `runner-tools` | `mock-target` 컨테이너 | 없음 또는 로컬 에뮬레이터 | M0~M4 개발·CI·목 타깃 실부하 | 이번 범위 |
| **L1 Remote Self-hosted Local** | 클라우드 VM/사내 서버의 앱 | 같은 서버/같은 VPC의 Docker/k3d/kind | `mock-target` 또는 allowlist된 dev/staging | VM/네트워크는 기존 제공, 러너는 self-hosted | "서비스는 클라우드에 올렸지만 러너는 그 서버의 로컬 컨테이너" | M6 배포 변형 |
| **C0 Local Control → Cloud Provider** | 개발자 로컬 앱 | Cloud Provider 관리형 러너 또는 ephemeral cloud runner | dev/staging cloud service | Terraform/OpenTofu plan/apply | 로컬에서 승인하고 cloud 자원을 만들어 실행 | M5 |
| **C1 Cloud Control → Cloud Provider** | 클라우드/사내 배포 앱 | Cloud Provider 관리형 러너 또는 cloud K8s Job | dev/staging/prod-like cloud service | Terraform/OpenTofu + cloud IAM | 운영형 배포. 다중 사용자·SSO·감사 필요 | M6 |
| **X0 Cross-cloud** | local 또는 cloud control | Provider A/Grafana Cloud/다른 클라우드 러너 | Provider B의 서비스 | mixed Provider | 예: AWS 서비스 서버를 GCP 또는 Grafana Cloud 러너에서 스트레스 테스트 | M5+ 선택 |

### 1.1 사용자 예시와 매핑

| 사용자 표현 | 이 문서의 모드 | 해석 |
|---|---|---|
| 로컬에서 로컬 자원(컨테이너)을 활용 | L0 | 맥북/CI에서 Docker compose로 앱·목타깃·러너 실행 |
| 로컬에서 Cloud Provider를 활용 | C0 | 제어 평면은 로컬, 러너/인프라 자원은 cloud provider |
| product를 클라우드에서 구동하고, 해당 클라우드 서버에서 로컬 자원(컨테이너)을 활용 | L1 | 클라우드 VM이나 사내 서버 위 Docker/k3d/kind는 "self-hosted local runner"다. 관리형 부하 서비스가 아니다 |
| 클라우드에서 Cloud Provider를 활용 | C1 | 제어 평면도 클라우드, 러너도 cloud-managed/cloud K8s |
| 서비스 서버는 AWS, 인프라 스트레스 테스트는 GCP | X0 | target_provider=aws, runner_provider=gcp. 두 Provider 정책·비용·네트워크를 모두 검토 |

---

## 2. 결정 규칙

1. **L0만 이번 로컬 MVP의 자동 실행 대상이다.** 대상은 `mock-target`뿐이다.
2. **L1은 "로컬형 러너"지만 대상이 mock이 아니면 승인 필수다.** 러너가 컨테이너라 해도 cloud VM에서 dev/staging 서비스를 치면 blast radius와 네트워크 비용이 생긴다.
3. **C0/C1은 cloud action이다.** `ToolAction.operation in (plan, apply, submit, canary, run)` 중 `apply/submit/run`은 승인·plan_hash·cost cap·teardown이 필요하다.
4. **X0은 가장 보수적으로 취급한다.** target Provider와 runner Provider가 다르면 양쪽 정책, egress 비용, allowlist, 관측 지연, 시간 동기화, 책임자를 모두 확인한다.
5. **prod 직접 고부하는 여전히 기본 금지다.** 토폴로지가 어떻든 `target_env=prod`이면 `DENY`가 기본이다.

---

## 3. ProvisionSpec 매핑

`docs/plan/02-data-model.md`의 `ProvisionSpec`은 아래 필드를 갖는다. 구현 시 `schemas.py`/`db.py`/OpenAPI에서 같은 이름을 유지한다.

| 필드 | 값 예시 | 의미 |
|---|---|---|
| `topology` | `local`, `container`, `k8s`, `managed-cloud` | 기존 큰 분류. 세부 판단은 아래 필드로 한다 |
| `control_plane_location` | `local-workstation`, `self-hosted-server`, `cloud-service` | 계획·승인·audit 앱이 어디서 도는가 |
| `runner_location` | `local-host`, `local-container`, `remote-container`, `cloud-managed`, `cloud-k8s-job` | 실제 부하 생성기 위치 |
| `runner_provider` | `local`, `aws`, `gcp`, `azure`, `grafana-cloud`, `on-prem`, `other` | 러너 자원이 속한 Provider |
| `target_provider` | `local`, `aws`, `gcp`, `azure`, `firebase`, `on-prem`, `other` | SUT가 속한 Provider |
| `network_path` | `same-host`, `same-vpc`, `public-internet`, `private-link`, `cross-cloud-public`, `cross-cloud-private` | 네트워크·egress·allowlist 판단 |

`mode`는 저장하지 않는다. 위 필드 조합에서 계산한다. 예를 들어 `runner_provider != target_provider`이고 둘 다 `local`이 아니면 X0(cross-cloud)로 판정한다.

---

## 4. 모드별 승인·안전 게이트

| 모드 | 자동 허용 가능 | 승인 필수 | 기본 금지 |
|---|---|---|---|
| L0 | 목 타깃 대상 read-only/low cap/canary | 로컬 상한 상향, 장시간 soak | 외부 endpoint, prod, destructive |
| L1 | 목 타깃 컨테이너 대상 low cap | dev/staging 대상, shared VPC, 비용/egress 발생 | prod, 타 팀/타 타이틀 영향 가능 경로 |
| C0 | `plan`/`show-json`/policy check | apply, cloud runner submit, cloud canary/full run | prod 직접 고부하, 장기 static credential |
| C1 | 운영자가 승인한 read-only 조회 | 모든 apply/run, 권한·비용·blast radius 변경 | 승인 없는 실행, 공유 인프라 영향 |
| X0 | 없음에 가깝다 | 양 Provider 정책 확인, egress/cost cap, source IP/load zone allowlist, on-call/time window | Provider 정책 위반, DDoS/volumetric 시뮬, prod 직접 고부하 |

---

## 5. Cross-cloud 실행(X0) 체크리스트

AWS에 있는 서비스 서버를 GCP/Grafana Cloud/Azure 러너로 테스트할 수는 있지만, 다음을 문서화하지 않으면 실행하지 않는다.

- **권한**: target Provider 리소스 소유자와 runner Provider 계정 소유자의 승인자가 다르면 둘 다 승인.
- **정책**: AWS EC2 Testing/DDoS 정책, GCP AUP, Azure ROE, Grafana Cloud k6 약관을 해당 조합에 맞게 확인.
- **네트워크**: source IP/load zone, WAF/Cloud Armor/AWS Shield/ALB allowlist, private link/VPN 여부.
- **비용**: target ingress/egress, runner 비용, NAT/Load Balancer/Cloud Logging 비용을 별도 `cost_estimate`에 포함.
- **관측**: target Provider 메트릭(CloudWatch/Cloud Monitoring/Azure Monitor)과 runner 메트릭의 clock skew·샘플 지연을 리포트 한계에 표시.
- **중단권**: target 측 on-call과 runner 측 operator가 모두 kill switch를 발동할 수 있어야 한다.

---

## 6. 마일스톤 반영

| 마일스톤 | 토폴로지 범위 |
|---|---|
| M0 | L0 compose skeleton(`app`/`mock-target`/`runner-tools`)과 `tools/versions.json` |
| M4 | L0 실제 목 타깃 부하 + Docker smoke 필수 |
| M4.5(선택) | L0/L1 구조 리허설: kind/k3d Job, remote-container 실행 수명주기 검증 |
| M5 | C0 설계·부분 구현: local control에서 cloud plan/apply/canary를 승인 게이트로 연결 |
| M6 | L1/C1/X0 중 조직 배포 모델 선택. SSO, 다중 승인자, Provider별 runtime policy 적용 |

---

## 7. 구현 산출물 기준

구현 단계에서 토폴로지 지원을 선언하려면 최소 산출물이 있어야 한다.

| 모드 | 필요한 산출물 |
|---|---|
| L0 | `docker/compose.local.yml`, `tools/versions.json`, runner-tools 이미지, mock profile, Docker smoke CI |
| L1 | remote compose/k3d/kind runbook, VM bootstrap, source IP 고정/allowlist, remote artifact collection |
| C0 | `ProvisionSpec` cloud fields, `IaCPlanner` plan/show-json, cloud runner adapter dry-run/canary, teardown/TTL |
| C1 | 배포 IaC, SSO/RBAC, audit retention, secrets/OIDC, 운영 런북 |
| X0 | cross-provider approval form, egress/cost estimate, source/load-zone allowlist, dual-provider incident/abort runbook |

---

## 8. 검증한 외부 사실

- Firebase Emulator Suite는 정확성 검증용이며 성능/보안/프로덕션 대체 용도로 쓰면 안 된다. 따라서 에뮬레이터 부하는 성능 근거로 삼지 않는다.
- `terraform test`는 Terraform 1.6에서 테스트 프레임워크가 일반 제공됐고, provider/resource/data source mocking은 Terraform 1.7부터 사용할 수 있다.
- LocalStack은 `tflocal`로 Terraform provider endpoint를 로컬 AWS 에뮬레이터로 돌릴 수 있다. 단 서비스 파리티와 티어 제한은 구현 직전 확인한다.
- k6는 Prometheus remote write output으로 Prometheus remote-write endpoint에 테스트 메트릭을 보낼 수 있다.
- kind는 Docker container node로 로컬 Kubernetes cluster를 만들며 local development/CI에 쓸 수 있다. k3d는 k3s cluster를 Docker에서 쉽게 띄우는 도구다.
- Docker Compose profiles는 하나의 compose 파일에서 용도별 서비스를 선택적으로 활성화하는 기능이다.
- Kubernetes Job은 완료될 때까지 Pod를 재시도하고, `ttlSecondsAfterFinished`로 완료/실패 Job 정리를 자동화할 수 있다.
- AWS는 volumetric network-based DDoS simulation을 EC2 testing guideline 밖으로 명시적으로 금지한다. Google Cloud는 테스트가 본인 프로젝트에만 영향이어야 한다고 안내한다. Azure에서 테스트 트래픽을 발생시켜도 Microsoft ROE와 구독 약관을 따른다.

## 9. 출처

- Firebase Local Emulator Suite: <https://firebase.google.com/docs/emulator-suite>
- Terraform tests / mocking: <https://developer.hashicorp.com/terraform/language/tests>, <https://developer.hashicorp.com/terraform/language/tests/mocking>, <https://www.hashicorp.com/en/blog/terraform-1-6-adds-a-test-framework-for-enhanced-code-validation>
- LocalStack Terraform: <https://docs.localstack.cloud/aws/connecting/infrastructure-as-code/terraform/>, <https://github.com/localstack/terraform-local>
- k6 Prometheus remote write: <https://grafana.com/docs/k6/latest/results-output/real-time/prometheus-remote-write/>
- kind / k3d / Kubernetes Job: <https://kind.sigs.k8s.io/>, <https://k3d.io/>, <https://kubernetes.io/docs/concepts/workloads/controllers/job/>, <https://kubernetes.io/docs/concepts/workloads/controllers/ttlafterfinished/>
- Docker Compose profiles: <https://docs.docker.com/compose/how-tos/profiles/>
- Provider policy references: <https://aws.amazon.com/ec2/testing/>, <https://aws.amazon.com/security/ddos-simulation-testing/>, <https://support.google.com/cloud/answer/6262505>, <https://cloud.google.com/terms/aup>, <https://learn.microsoft.com/en-us/azure/security/fundamentals/pen-testing>, <https://www.microsoft.com/en-us/msrc/pentest-rules-of-engagement>
