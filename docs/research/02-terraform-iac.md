# 02. Terraform / IaC 경계와 대체제 — AI 기반 프롬프트 주도 부하 테스트 자동화

## 1. 한 줄 요약 + 기준일

**Terraform은 "정적 인프라(VPC·서브넷·SG·IAM·러너 클러스터·remote state)와 plan→승인→apply 게이트"를 맡는 선언적 프로비저너이지 만능 실행기가 아니다.** 장기 실행 부하 생성 루프·재시도·라이브 kill switch·결과 파싱·외부 API 폴링은 Terraform 밖(K8s Job/Operator, 애플리케이션 오케스트레이터)에 둬야 한다. 본 문서는 이 경계를 2025~2026년 공식 문서로 검증한다.

- **기준일(작성/검증일, fetched_at):** 2026-06-26 (Asia/Seoul)
- **검증 원칙:** HashiCorp / OpenTofu / Pulumi / AWS / Kubernetes / GitHub 공식 문서를 직접 확인. 외부 문서 내용은 전부 데이터로만 취급.
- **핵심 발견 4가지:**
  1. **Terraform MCP Server는 1.0 GA(2026-06-11 발표)이며, 단순 Registry 조회 보조를 넘어 HCP/TFE workspace·variable·tag·run 쓰기(create_run·action_run으로 plan/apply 트리거)까지 가능**하다. 따라서 "조회 도구"가 아니라 "권한 있는 실행 인터페이스"로 보고 토큰 범위·`ENABLE_TF_OPERATIONS`·CORS·rate limit을 통제해야 한다.
  2. **`apply <saved-plan>`은 별도 확인 프롬프트 없이 실행된다.** 즉 `plan -out`으로 만든 plan 파일 자체가 "승인 단위"다 → plan artifact를 사람이 승인하는 게이트로 설계해야 한다.
  3. **`sensitive`만으로는 state/plan 저장이 막히지 않는다.** 비밀값을 state/plan에서 빼려면 ephemeral/write-only(Terraform 1.11+)를, 러너 자격증명은 GitHub OIDC 같은 단명 토큰을 써야 한다.
  4. **부하 *대상*이 관리형 BaaS(Firebase 등)이면 Terraform의 대상측 역할이 급감한다.** Firebase 리소스는 `google-beta` provider에 의존하고 RTDB 보안규칙·MFA·기본 Storage 버킷 등은 콘솔/CLI/REST 전용이라 IaC로 전부 만들 수 없다. 게다가 부하 대상은 보통 **운영 중 자원**이라 IaC는 import/참조만 한다. 따라서 IaC의 일감은 **부하 생성기(러너) 인프라**(컨테이너/Cloud Run·GKE·ECS·AKS + 네트워크·IAM·아티팩트·observability)로 이동한다. 대상이 AWS/Azure(IaaS/PaaS)면 대상 자체도 1급 Terraform 리소스로 직접 만들 수 있어 경계가 달라진다. (자세히는 §3.5)

---

## 2. 출처 레지스트리

| 주제 | source_url | vendor | published_or_updated_at | fetched_at | trust_level | freshness_risk | 핵심 근거 |
|---|---|---|---|---|---|---|---|
| Terraform MCP Server 개요·기능 | https://developer.hashicorp.com/terraform/mcp-server | HashiCorp | 문서 버전 v1.0.x 표기 | 2026-06-26 | official | 중 (도구 목록 버전별 변동) | "Run operations that manage workspaces, variables, tags, and variable sets in HCP Terraform or Terraform Enterprise" |
| MCP Server 도구 레퍼런스(read/write 구분) | https://developer.hashicorp.com/terraform/mcp-server/reference | HashiCorp | v1.0.x | 2026-06-26 | official | 중 | `create_run`, `action_run`(apply/discard/cancel), `update_workspace`(requires `ENABLE_TF_OPERATIONS=true`), `delete_workspace_safely` |
| MCP Server 배포·보안 설정 | https://developer.hashicorp.com/terraform/mcp-server/deploy | HashiCorp | v1.0.x | 2026-06-26 | official | 중 | "We recommend running the MCP server locally at `127.0.0.1`"; "By default … runs in `strict` CORS mode and the allowed origins are empty"; "global and per-session rate limits" |
| MCP Server 보안 권고 | https://developer.hashicorp.com/terraform/mcp-server/security | HashiCorp | v1.0.x | 2026-06-26 | official | 중 | "Always validate before doing any write changes." |
| MCP Server GA 발표 | https://www.hashicorp.com/en/blog/terraform-mcp-server-is-now-generally-available | HashiCorp | 2026-06-11 | 2026-06-26 | official(blog) | 낮 | "Terraform MCP server 1.0 … generally available"; "controlled interface that enforces your existing Terraform authentication and authorization" |
| `terraform apply` (saved plan) | https://developer.hashicorp.com/terraform/cli/commands/apply | HashiCorp | latest | 2026-06-26 | official | 낮 | "Terraform performs the operations in the saved plan without prompting you for confirmation"; "ignores this option [-auto-approve] when you pass a previously-saved plan file … interprets the act of passing the plan file as the approval" |
| `terraform plan -out` | https://developer.hashicorp.com/terraform/cli/commands/plan | HashiCorp | latest | 2026-06-26 | official | 낮 | saved plan은 "full configuration, all of the values associated with planned changes, and all of the plan options"를 담고, sensitive 데이터가 **암호화되지 않은 채** 저장됨 |
| `terraform destroy` | https://developer.hashicorp.com/terraform/cli/commands/destroy | HashiCorp | latest | 2026-06-26 | official | 낮 | "This command is just a convenience alias for … `terraform apply -destroy`" |
| sensitive 데이터 관리 | https://developer.hashicorp.com/terraform/language/manage-sensitive-data | HashiCorp | latest | 2026-06-26 | official | 낮 | "Terraform stores values with the `sensitive` argument in both state and plan files, and anyone who can access those files can access your sensitive values" |
| ephemeral 값 | https://developer.hashicorp.com/terraform/language/manage-sensitive-data/ephemeral | HashiCorp | v1.15.x 표기 | 2026-06-26 | official | 낮 | "Terraform does not store information about ephemeral resources in state or plan files" |
| write-only 인자 | https://developer.hashicorp.com/terraform/language/manage-sensitive-data/write-only | HashiCorp | Terraform 1.11+ | 2026-06-26 | official | 낮 | write-only(`_wo`/`_wo_version`)는 state/plan에 미저장; "must use Terraform v.1.11 or later" |
| HCP run 단계/상태 | https://developer.hashicorp.com/terraform/cloud-docs/run/states | HashiCorp | latest | 2026-06-26 | official | 중 | "pending, plan, policy check, apply, and completion"; "waits for user approval before running an apply" (Needs Confirmation) |
| CLI-driven 원격 워크플로 | https://developer.hashicorp.com/terraform/cloud-docs/run/cli | HashiCorp | latest | 2026-06-26 | official | 낮 | "the command line will prompt you for approval before applying"; "runs those policies against all speculative plans and remote applies" |
| Cost estimation | https://developer.hashicorp.com/terraform/cloud-docs/workspaces/cost-estimation | HashiCorp | latest | 2026-06-26 | official | 중 | 비용 추정은 "between the plan and apply" 단계; "Cost estimation is only available in policy checks, HCP Terraform's Legacy policy execution mode" (Sentinel만 cost 데이터 접근, `tfrun` import) |
| Policy enforcement 레벨 | https://developer.hashicorp.com/terraform/cloud-docs/policy-enforcement/manage-policy-sets | HashiCorp | latest | 2026-06-26 | official | 중 | Sentinel: advisory / soft-mandatory(override 가능) / hard-mandatory; OPA: advisory / mandatory |
| Policy enforcement 개요(Sentinel+OPA 공존) | https://developer.hashicorp.com/terraform/cloud-docs/policy-enforcement | HashiCorp | latest | 2026-06-26 | official | 중 | "a policy set can only contain policies written in a single policy framework" (Sentinel·OPA를 같은 workspace에 혼용 가능) |
| Run tasks(외부 통합 게이트) | https://developer.hashicorp.com/terraform/cloud-docs/integrations/run-tasks | HashiCorp | latest | 2026-06-26 | official | 중 | post_plan 등 단계 hook; `task_result_enforcement_level: mandatory`로 "prevent a run from continuing to the apply phase" |
| OpenTofu state/plan 암호화 | https://opentofu.org/docs/language/state/encryption/ | OpenTofu(LF) | latest(무날짜) | 2026-06-26 | official | 중 | "OpenTofu supports encrypting state and plan files at rest"; "The only currently supported encryption method is AES-GCM" |
| OpenTofu 1.7.0 릴리스 | https://opentofu.org/blog/opentofu-1-7-0/ | OpenTofu(LF) | 2024-04-30 | 2026-06-26 | official(blog) | 중 | "End-to-end state encryption"; "drop-in replacement for its predecessor Terraform™ 1.5" |
| OpenTofu 거버넌스 | https://opentofu.org/faq/ | OpenTofu(LF) | latest | 2026-06-26 | official | 낮 | "giving OpenTofu to the Linux Foundation" |
| Terragrunt 개요 | https://docs.terragrunt.com/getting-started/overview | Gruntwork | latest(무날짜, 308 redirect) | 2026-06-26 | official | 중 (호스트/CLI 명령 변경) | "operate as an orchestrator, at a level of abstraction higher than OpenTofu/Terraform"; DAG 순서; `run --all` |
| Crossplane vs Terraform | https://blog.crossplane.io/crossplane-vs-terraform/ | Upbound/Crossplane | 2021-03-02 | 2026-06-26 | vendor-blog | 높 (오래됨, v2 출시) | "Crossplane is a control plane, where Terraform is a command-line tool"; "long lived, always-on control loops" |
| Crossplane Managed Resources(드리프트 자동복구) | https://docs.crossplane.io/latest/managed-resources/managed-resources/ | Crossplane(v2.3) | latest | 2026-06-26 | official | 중 | "Crossplane overrides any changes made to an external resource outside of Crossplane" |
| Pulumi Automation API | https://www.pulumi.com/docs/iac/concepts/automation-api/ | Pulumi | latest(무날짜) | 2026-06-26 | official | 중 | "programmatic interface for running Pulumi programs without the Pulumi CLI … drive the Pulumi engine from within your own application" |
| Pulumi Deployments(관리형 러너) | https://www.pulumi.com/docs/pulumi-cloud/deployments/ | Pulumi | latest | 2026-06-26 | official | 중 | "managed service that runs Pulumi operations"; "Ephemeral environments stood up automatically for each pull request" |
| Pulumi TTL Stacks | https://www.pulumi.com/docs/deployments/deployments/ttl/ | Pulumi | latest | 2026-06-26 | official | 중 | "automatically torn down"; "measured from the stack's **creation** date, not its most recent update" |
| Pulumi CrossGuard(정책) | https://www.pulumi.com/docs/using-pulumi/crossguard/ | Pulumi | latest | 2026-06-26 | official | 중 | "blocking deployments when violations are detected"; enforcement level Advisory/Mandatory |
| AWS CDK `--require-approval` | https://docs.aws.amazon.com/cdk/v2/guide/ref-cli-cmd-deploy.html | AWS | CDK v2 | 2026-06-26 | official | 낮 | `any-change`/`broadening`(default)/`never`; "broadening of permissions or security group rules"; change set는 `prepare-change-set`/`execute-change-set` |
| K8s Job API(완료/병렬/재시도/타임아웃) | https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/job-v1/ | Kubernetes | batch/v1(GA) | 2026-06-26 | official | 낮 | `completions`/`parallelism`/`backoffLimit`/`activeDeadlineSeconds`("system tries to terminate it") |
| K8s ttlSecondsAfterFinished | https://kubernetes.io/docs/concepts/workloads/controllers/ttlafterfinished/ | Kubernetes | v1.23 [stable] | 2026-06-26 | official | 낮 | "clean up finished Jobs … automatically"; "delete it cascadingly" |
| K8s CronJob | https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/ | Kubernetes | v1.21 [stable] | 2026-06-26 | official | 낮 | "creates Jobs on a repeating schedule"; `successfulJobsHistoryLimit`/`failedJobsHistoryLimit` |
| K8s Operator 패턴 | https://kubernetes.io/docs/concepts/extend-kubernetes/operator/ | Kubernetes | 2025-07-16 갱신 | 2026-06-26 | official | 낮 | "software extensions … control loop"; "Controller will normally run outside of the control plane … as a Deployment" |
| GitHub Actions Environments(필수 리뷰어) | https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments | GitHub | rolling(무날짜) | 2026-06-26 | official | 중 | "Only one of the required reviewers needs to approve"; wait timer "between 1 and 43,200 (30 days)" |
| GitHub Actions 환경 보호규칙 | https://docs.github.com/actions/deployment/targeting-different-environments/using-environments-for-deployment | GitHub | rolling | 2026-06-26 | official | 중 | "must follow any protection rules … before running"; "Prevent self-review" |
| GitHub Actions OIDC | https://docs.github.com/en/actions/concepts/security/openid-connect | GitHub | rolling | 2026-06-26 | official | 중 | "short-lived access token that is only valid for a single job, and then automatically expires"; "Eliminates static, long-lived credentials" |
| Firebase + Terraform 시작 가이드(지원/미지원 범위) | https://firebase.google.com/docs/projects/terraform/get-started | Google/Firebase | latest(무날짜) | 2026-06-26 | official | 중 | "You must use the `google-beta` provider because this is a beta release"; RTDB 보안규칙·MFA "not yet supported"; 기본 Cloud Storage 버킷 프로비저닝 2024-10-30~ 미지원(콘솔/REST 사용) |
| google_firestore_database 리소스 | https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/firestore_database | HashiCorp/Google | latest | 2026-06-26 | official | 낮 | database location은 설정 후 영구(변경 불가) |
| google_firebase_project (google-beta) | https://registry.terraform.io/providers/hashicorp/google-beta/latest/docs/resources/firebase_project | HashiCorp/Google | beta | 2026-06-26 | official | 중 | Firebase 활성화는 google-beta provider 전용 리소스 |
| google_firebaserules_ruleset / release | https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/firebaserules_ruleset | HashiCorp/Google | latest | 2026-06-26 | official | 중 | ruleset은 immutable(변경 시 replace); release name `cloud.firestore`로 활성화 |
| GCP Terraform 테스트 베스트프랙티스(ephemeral project) | https://docs.cloud.google.com/docs/terraform/best-practices/testing | Google | latest | 2026-06-26 | official | 중 | "each test create a fresh project or folder"; random provider로 고유 ID; `destroy` 후 추가 정리; `project_cleanup` 모듈("Don't use such tools in a production environment") |

> trust_level 범례: official(공식 문서) > official(blog)/vendor-blog > community. freshness_risk: 낮/중/높.

---

## 3. Terraform 책임 경계

### 3-A. Terraform이 맡을 일 (기본 적용)

| 맡을 일 | 왜 적합한가 | 대표 리소스/관점 |
|---|---|---|
| 네트워크(VPC/VNet, subnet, route, NAT) | 선언적·멱등적이고 변경이 드물며 plan 디프가 의미 있음 | provider 안정 리소스 |
| 보안 경계(SG/firewall, NACL) | 변경이 plan에서 가시화되고 정책(policy-as-code)으로 게이트 가능 | SG 규칙, 0.0.0.0/0 차단 정책 |
| ID/권한(IAM role, SA, managed identity, OIDC trust) | 신뢰관계는 한 번 세팅하고 오래 유지되는 정적 자원 | 러너용 role + OIDC provider trust |
| 아티팩트/로그 버킷, observability 리소스 | 수명이 길고 태그/TTL/cost allocation을 붙이기 좋음 | S3/GCS 버킷, 로그 그룹, 대시보드 |
| 러너 인프라(ECS/Fargate, EKS/GKE/AKS, ACI/Container Apps) | 클러스터/서비스 정의는 선언적; 실제 잡 실행은 별도 | 클러스터, 노드풀, 태스크 정의 |
| provider가 안정 지원하는 부하테스트 리소스 | provider 스키마가 받쳐주면 멱등 관리 가능 | (예: 관리형 부하테스트 서비스 리소스) |
| remote state / workspace / state locking | 협업·동시성 안전의 핵심; plan/apply의 기반 | backend, workspace, lock |
| tags / TTL / cost allocation | 비용·정리 자동화의 메타데이터 소스 | 공통 태그 모듈 |
| policy/cost와 연결되는 plan 단계 | `plan -out` + `show -json`이 정책·비용 검토의 입력 | Sentinel/OPA, cost estimate |

### 3-B. Terraform이 맡으면 안 되는 일 (다른 컴포넌트로)

| 맡으면 안 되는 일 | 그 이유 | 누가 대신 맡나 |
|---|---|---|
| 장기 실행 load generation loop | Terraform은 invoke 시점에만 reconcile하는 1회성 도구. 실행 중 상태를 들고 있지 않음 | **K8s Job**(`completions`/`parallelism`), Operator |
| retry/backoff orchestration | provisioner/스크립트 글루는 멱등성·관측성이 깨지고 재현 불가 | **K8s Job** `backoffLimit`, 오케스트레이터 |
| 라이브 kill switch(실행 중 중단) | apply는 리소스 수렴이 목적; 실행 중 잡을 죽이는 개념이 없음 | **K8s Job** `activeDeadlineSeconds`, 잡 삭제, Operator |
| test result parsing / 평가 | Terraform은 결과 데이터 파이프라인이 아님 | 부하테스트 도구 + 결과 처리 서비스 |
| business workflow 실행 / 외부 API polling | 선언적 IaC 모델 밖. state 오염·드리프트 유발 | 애플리케이션 코드, 워크플로 엔진 |
| ad hoc provisioner shell glue | `local-exec`/`remote-exec`는 멱등성·롤백·가시성 부재(공식적으로도 최후수단 권고) | CI 잡, K8s Job, Operator |
| secrets 조립 / 장기 저장 | `sensitive`도 state/plan에 평문 저장됨 → state가 비밀 저장소화 | ephemeral/write-only, OIDC 단명 토큰, 외부 secrets manager |

> 핵심 전제: **Terraform = 정적 토폴로지 + 신뢰관계 + plan/apply 게이트.** 실행(run-to-completion)·재시도·중단·파싱은 K8s Job/Operator 등 "control loop를 내장한" 컴포넌트의 몫이다. (K8s Operator는 "control loop"를 따르고 보통 control plane 밖에서 Deployment로 동작 — kubernetes.io/operator)

---

## 3.5 부하 대상이 관리형 BaaS(Firebase 등)일 때 IaC 경계 재배치

> 도메인 맥락(기획): 게임 회사·20+ 타이틀, 부하 **대상**이 기존 **Firebase 백엔드**, 사용자는 비전문가. 이때 핵심 원칙은 단순하다 — **부하 대상(BaaS)은 Terraform이 거의 만들지 않는다.** 운영 중 자원이므로 IaC는 대상을 import/참조만 하고, IaC의 진짜 일감은 **부하 생성기(러너) 인프라**로 옮겨간다. 3절의 일반 경계(정적 토폴로지 = IaC, 실행 루프 = 비-IaC)는 그대로이며, 여기서는 "대상이 BaaS일 때 *대상측* IaC 비중이 왜 작아지는지"를 검증한다.

### 3.5-A 부하 대상(BaaS) — IaC 역할 축소

| 대상 레이어 | Terraform이 만들 수 있나 | 실무상 누가/어떻게 | 근거 |
|---|---|---|---|
| Firebase 프로젝트 자체 | 부분 가능(`google_firebase_project`, **google-beta**) | 운영 대상은 이미 존재 → **import/참조만**, 신규 생성 불필요 | Firebase get-started; registry firebase_project |
| Firestore database | 가능(`google_firestore_database`, google-beta), **location 영구** | 부하 대상이면 보통 기존 DB 사용. 데이터·트래픽 살아있는 운영자원 | registry firestore_database |
| RTDB 인스턴스 | 가능(`google_firebase_database_instance`, google-beta) | 인스턴스는 TF 가능하나 **RTDB 보안규칙은 TF 미지원 → Firebase CLI** | Firebase get-started |
| Firestore 보안규칙 | 가능(`firebaserules_ruleset`/`release`)이나 ruleset **immutable** | 규칙 변경은 부하 테스트와 무관(운영 변경) → 보통 IaC 범위 밖 | registry firebaserules |
| 인증 / MFA | **MFA "not yet supported"** | 콘솔 | Firebase get-started |
| 기본 Cloud Storage 버킷 | **2024-10-30부터 TF 프로비저닝 미지원** | 콘솔 / REST API | Firebase get-started |

> 결론: 부하 대상 Firebase는 **운영 중 자원**이고 부하 테스트가 생성·파괴할 대상이 아니다. IaC는 대상을 **읽기/import**만 하고, 테스트가 만든 데이터는 애플리케이션·CLI로 정리한다. "대상측 IaC"는 사실상 거의 비어 있다.

### 3.5-B 부하 생성기(러너) — IaC 핵심 책임 (대상을 두드리는 쪽)

| 러너 레이어 | IaC 책임 | 클라우드별 등가물 |
|---|---|---|
| 러너 컴퓨트 | ✅ 클러스터/서비스 선언적 정의 | Cloud Run·GKE / ECS·Fargate·EKS / Container Apps·AKS |
| egress 네트워크 | ✅ VPC·subnet·NAT(대상 도메인으로 나가는 경로) | 3사 공통 VPC/VNet·라우트 |
| IAM·자격증명 | ✅ **신뢰관계만**(실제 토큰은 런타임 발급) | Workload Identity/OIDC, SA/role |
| 아티팩트·러너 이미지 | ✅ | Artifact Registry / ECR / ACR |
| observability | ✅ 로그·메트릭·대시보드 | Cloud Logging / CloudWatch / Azure Monitor |
| 부하 실행 루프·재시도·**kill switch** | ❌ 비-IaC | K8s Job/CronJob/Operator (§3-B, §4) |

> 즉 대상이 BaaS여도 러너측 IaC 책임은 IaaS/PaaS 대상일 때와 동일하다. 줄어드는 건 "대상 자체를 IaC가 만드는 부분"뿐이다.

### 3.5-C 대상 유형별 IaC 경계가 달라지는 지점

| 대상 유형 | 대상 자체를 IaC가 만드나 | 격리·teardown 단위/전략 | 러너측 IaC |
|---|---|---|---|
| **Firebase/Firestore/RTDB(BaaS), 운영 대상** | 거의 안 만듦 — import/참조(일부 google-beta 리소스 존재) | 대상은 teardown 안 함(운영). **테스트 데이터만** 정리 | 동일(러너가 IaC 책임) |
| **신규 격리 GCP/Firebase 테스트 대상**(스테이징 복제) | `google_project`로 **ephemeral 생성 가능** | "fresh project per test" + random suffix + `destroy` + `project_cleanup` | 동일 |
| **AWS 대상**(DynamoDB·API GW·Lambda 등) | TF 1급 리소스로 직접 생성/관리 광범위 | 리소스 단위 `destroy`; 계정/Organizations로 격리 | 동일 |
| **Azure 대상**(Cosmos DB 등) | TF 1급 리소스로 직접 생성/관리 | **resource group 단위 destroy**로 깔끔 격리 | 동일 |

> 핵심 대비: GCP/Firebase는 **프로젝트**가 격리·일괄 teardown 단위라 ephemeral project 패턴이 공식 권장 흐름이다. AWS는 **계정/Organizations**, Azure는 **resource group**이 그 단위. 단, 부하 *대상*이 운영 중 Firebase면 새 프로젝트 생성은 "대상 복제(스테이징)" 시나리오에 한하고, 운영 대상 직접 부하 시 IaC는 대상을 건드리지 않는다.

### 3.5-D GCP/Firebase IaC 현황 (2026-06-26 검증)

| 항목 | 검증된 현황 | source_url (fetched_at 2026-06-26) |
|---|---|---|
| Firebase 리소스 provider | **google-beta 필요** ("You must use the `google-beta` provider … beta release") | firebase.google.com/docs/projects/terraform/get-started |
| 지원 리소스 | `firebase_project`, `firestore_database`, `firebase_database_instance`(RTDB), `firebaserules_ruleset`/`release`, `firebase_storage_bucket`, `identity_platform_config` 등 | 동 get-started + registry |
| 미지원/제약 | RTDB 보안규칙(TF 미지원→타 툴), MFA("not yet supported"), 기본 Storage 버킷 프로비저닝(2024-10-30~ 미지원) | 동 get-started |
| Firestore location | 설정 후 **영구**(변경 불가) | registry google_firestore_database |
| 보안규칙 리소스 | ruleset **immutable**(변경=replace), release name `cloud.firestore`로 활성화 | registry google_firebaserules_ruleset/release |
| ephemeral 테스트 프로젝트 | 공식: "each test create a fresh project or folder" + random provider로 고유 ID + `destroy` + `project_cleanup` 모듈(**프로덕션 사용 금지**) | docs.cloud.google.com/docs/terraform/best-practices/testing |

> AWS/Azure와의 대비: AWS/Azure 대상은 대부분 **1급 Terraform 리소스**로 직접 생성·teardown이 매끄럽고 별도 beta provider가 불필요하다. 반면 BaaS(Firebase)는 (a) **google-beta 의존**, (b) 일부 기능 **콘솔/CLI/REST 전용**, (c) 운영 대상은 **import 위주** → **대상측 IaC 비중이 본질적으로 작다.** 이 세 가지가 "BaaS 대상에서 Terraform 역할 축소"의 직접 근거다.

### 3.5-E 범용 멀티클라우드 원칙 + 비전문가 게이트

- **원칙(대상 종류 불문)**: 러너 컴퓨트·egress 네트워크·IAM 신뢰관계·아티팩트·observability·`plan`/승인/`destroy` 게이트 = **IaC 책임**. 실행 루프·재시도·라이브 kill switch·결과 파싱 = **비-IaC**(K8s Job/Operator/오케스트레이터). 대상이 Firebase든 AWS든 Azure든 **이 경계는 동일**하며, 달라지는 것은 "대상 자체를 IaC가 얼마나 만드느냐"뿐이다.
- **자연어→템플릿 매칭 UX(비전문가 대상)**: plan 산출·정책 검토·승인 절차를 자동화 뒤에 숨겨 화면을 단순화하되, §4의 **plan artifact 승인 게이트는 우회 불가(hard-mandatory)**로 남긴다 — "쉬워 보이게"는 하되 "건너뛸 수 있게"는 하지 않는다.

---

## 4. 권장 Terraform 워크플로 11단계 (승인·정책·자동화 의미)

> 부하테스트 자동화의 IaC 부분을 "plan artifact를 승인 단위로 삼는" 게이트형 파이프라인으로 설계한다. 각 단계는 사람이 봐야 할 게이트(🚦), 자동 통과(⚙️), 정책 결합(🛡️)을 표시.

| # | 단계 | 명령/행위 | 승인·정책·자동화 의미 |
|---|---|---|---|
| 1 | 포맷·정적검증 | `terraform fmt -check`, `terraform validate` | ⚙️ 자동. 구문/포맷 게이트. 실패 시 즉시 중단(실행 전 위생) |
| 2 | provider/module 버전 고정 검사 | lock 파일·version constraint 확인 | ⚙️ 자동. **공급망 안정성** 게이트. unstable provider/모듈로 인한 비결정성 차단 |
| 3 | 저장 plan 생성 | `terraform plan -out=<planfile>` | ⚙️ 자동 생성하되 **이 파일이 곧 승인 대상**. saved plan에는 전체 설정·입력변수가 들어가고 sensitive가 평문 저장되므로 **민감 아티팩트로 취급**(접근통제·짧은 보관) |
| 4 | 머신리더블 plan 검토 | `terraform show -json <planfile>` | ⚙️ 자동. 정책 엔진·비용 검토·diff 가시화의 **입력**. 사람이 읽는 텍스트가 아니라 JSON을 게이트 근거로 |
| 5 | 정책·쿼터·비용 검토 | Sentinel/OPA + cost estimate (HCP) 또는 외부 정책엔진 | 🛡️ 정책 게이트. HCP에서는 plan→**cost estimation**(plan과 apply 사이)→**policy check** 순. **cost 정책은 Sentinel만 cost 데이터 접근 가능**(OPA 불가). enforcement: advisory/soft-mandatory(override)/hard-mandatory |
| 6 | 사람 승인 | HCP "Needs Confirmation" 또는 외부 승인 게이트 | 🚦 인적 게이트. "HCP Terraform waits for user approval before running an apply." 승인 단위 = **3번 plan artifact** |
| 7 | 승인된 plan 적용 | `terraform apply <approved-planfile>` | ⚙️ 실행. **saved plan 적용 시 별도 프롬프트 없음**(plan 전달 = 승인). `-auto-approve`는 이 경우 무시됨. → 승인은 plan 단계에서 끝나야 함 |
| 8 | smoke / canary | 최소 러너로 헬스/연결 검증 | 🚦 자동 검증 + 실패 시 중단 게이트. 본 부하 전에 인프라 정상성 확인 |
| 9 | full run(부하 실행) | **K8s Job/Operator가 수행**(Terraform 아님) | ⚙️ 실행은 Terraform 밖. Job `parallelism`로 부하 생성, `activeDeadlineSeconds`로 하드 타임아웃 |
| 10 | report | 결과 수집·평가(외부 파이프라인) | ⚙️ Terraform 밖. 결과 파싱/판정은 IaC 책임 아님 |
| 11 | destroy / hold | `terraform destroy`(= `apply -destroy`) 또는 보류 | 🚦/⚙️. 정리는 멱등 destroy로. 기본 apply는 확인을 요구하나 자동화 시 saved destroy-plan(`plan -destroy -out`) + 승인 패턴 권장. 일시 환경은 K8s `ttlSecondsAfterFinished`/Pulumi TTL로 자동 회수 |

> 보강 포인트: HCP **Run tasks**로 5~7단계 사이(post_plan/pre-apply 등)에 외부 검사(보안 스캐너 등)를 mandatory로 끼워 apply 진행을 막을 수 있다.

---

## 5. 안전상 중요한 해석

| 항목 | 공식 근거 | 설계 함의 |
|---|---|---|
| **saved plan = 승인 그 자체** | "Terraform performs the operations in the saved plan **without prompting** you for confirmation" / "interprets the act of passing the plan file as the approval" (apply 문서) | apply 단계에 사람을 두지 말 것. **승인은 `plan -out` 결과물을 사람이 검토·서명하는 지점**으로 이동. CI에서 plan artifact를 승인 게이트(예: GitHub Environments required reviewers)로 보호 |
| **`-auto-approve`의 제한된 의미** | auto plan 모드에서 대화형 승인만 건너뜀; **saved plan 전달 시 무시됨** | "`-auto-approve`를 켰다"가 곧 무통제 적용은 아님. 진짜 위험은 *plan 검토 없이* auto plan을 auto-approve하는 조합. 자동화는 항상 `plan -out`→정책→승인→`apply <planfile>` 경로로 |
| **`sensitive` ≠ 비저장** | "stores values with the `sensitive` argument in **both state and plan files**" | `sensitive`는 출력 가림(redaction)일 뿐. 비밀을 state/plan에서 제거하려면 **ephemeral 값 / write-only 인자(Terraform 1.11+, `_wo`)**. saved plan 파일도 평문 비밀을 담을 수 있어 보관·전송 통제 필요 |
| **ephemeral / write-only 지원 현황** | ephemeral 리소스·값은 "not store … in state or plan files"; write-only는 1.11+ 및 **해당 provider 지원 필요** | 모든 인자가 write-only를 지원하지 않음 → provider 스키마 확인이 전제. 비밀은 가능하면 IaC가 "조립/저장"하지 말고 런타임 주입 |
| **OIDC 단명 토큰(러너 자격증명)** | GitHub OIDC: "short-lived access token that is **only valid for a single job**, and then automatically expires"; "Eliminates static, long-lived credentials" | 러너에 장기 클라우드 키를 심지 말 것. **Terraform은 OIDC 신뢰관계(IAM role + provider)만 정적으로 만들고**, 실제 토큰은 잡 실행 시 OIDC 교환으로 발급 |
| **MCP Server는 쓰기 가능 = 권한 인터페이스** | `create_run`/`action_run`(apply/discard/cancel), `update_workspace`(`ENABLE_TF_OPERATIONS=true` 필요), 변수/태그 쓰기 | MCP를 "조회 보조"로 과소평가 금지. 127.0.0.1 바인딩·strict CORS(기본)·TLS·global/per-session rate limit·**최소 권한 `TFE_TOKEN`**·쓰기 기능 명시적 활성화로 통제. "Always validate before doing any write changes" |
| **destroy의 자동화** | `terraform destroy` = `apply -destroy` 별칭 | 자동 정리도 동일 게이트 적용. 무인 destroy는 `plan -destroy -out`으로 승인 가능한 형태를 만들고 적용. 운영 장수명 리소스는 destroy 대상에서 분리 |

---

## 6. IaC / 오케스트레이션 대체제·보완재 비교

| 후보 | 적합한 역할 | 강점 | 주의점 | 이번 설계 위치 |
|---|---|---|---|---|
| **Terraform** (HCP 포함) | 정적 인프라 + plan/승인/apply 게이트 + 정책·비용 | 광범위 provider, plan diff, Sentinel/OPA+cost, saved-plan 승인 모델 | sensitive 평문 저장, 실행 루프/재시도 부적합, MCP 쓰기 통제 필요 | **기본(IaC 엔진)** |
| **OpenTofu** | Terraform 대체 엔진(`tofu`) | **state+plan 클라이언트측 암호화(AES-GCM, 1.7.0~)**, Linux Foundation 거버넌스, TF 1.5 드롭인 | TF 신규기능과 점진적 divergence, 암호화 문서 무버전 | **대체**(엔진 교체. state에 IAM·SG·endpoint 담기는 부하테스트엔 암호화가 장점) |
| **Terragrunt** | TF/OpenTofu 위 오케스트레이터 | DAG 의존 순서, `run --all` 다중모듈, backend.tf 자동생성(DRY) | 엔진을 대체하지 않음(래퍼), 호스트/CLI 명령(run-all→`run --all`) 변경 | **보완**(러너 인프라 모듈 다단 배치 시) |
| **Crossplane** | K8s 컨트롤플레인형 IaC | 지속적 reconcile, API 주도, self-healing | **always-on 루프가 수동/AI 주도 teardown·kill과 충돌**(드리프트 자동복구), K8s 클러스터 의존, v2 용어 변화 | **보완/대체 패러다임(주의)** — 러너가 이미 K8s-native가 아니면 비권장 |
| **Pulumi Automation API** | 프로그램에서 직접 엔진 구동(CLI 없이) | in-process 타입드 preview/up/destroy, AI 오케스트레이션 루프에 적합 | Pulumi 생태계 종속, 문서 무날짜·SDK 버전 고정 필요 | **보완(또는 오케스트레이션 레이어 대체)** |
| **Pulumi Deployments** | 관리형 러너 + 일시 환경 | Review Stacks(PR별 ephemeral), **TTL 자동 파기**, 스케줄 실행 | TTL은 **생성 시각 기준**(업데이트로 리셋 안 됨), Pulumi Cloud(상용) | **보완**(테스트별 ephemeral 환경 자동 회수) |
| **Pulumi CrossGuard** | 정책 게이트 | preview/up에서 위반 차단, Advisory/Mandatory | "사람이 plan 승인" 버튼은 아님(정책+CI/ESC pause 조합) | **보완**(자동 정책 게이트) |
| **AWS CDK `--require-approval`** | CLI 승인 프롬프트 | `broadening`(기본)=IAM/SG 확장 시 승인 | **IAM/SG 확장만 게이트**, AWS 전용, 실제 적용은 CloudFormation change set(`prepare`/`execute`) | **보완(약함)** — AWS+CFN 전용 환경에서만 |
| **K8s Job** | 부하 잡 run-to-completion | `completions`/`parallelism`(팬아웃)/`backoffLimit`(재시도)/`activeDeadlineSeconds`(하드 타임아웃=kill) | 클러스터 필요; Job 실패는 수동개입 필요할 수 있음 | **보완(실행 엔진)** — Terraform이 안 맡는 실행을 담당 |
| **K8s `ttlSecondsAfterFinished`** | 완료 잡 자동 정리 | 완료/실패 후 cascade 삭제, **GA v1.23** | 0=즉시삭제/미설정=영구보존 의미 구분 | **보완(자동 정리)** |
| **K8s CronJob** | 정기 부하(야간 soak 등) | cron 스케줄, history limit | 100회 초과 미스 시 미실행 등 경계 | **보완(스케줄링)** |
| **K8s Operator 패턴** | 프롬프트 주도 오케스트레이션 로직의 집 | reconcile loop로 lifecycle/retry/kill/결과처리 소유 | 컨트롤러 구현 비용 | **보완(오케스트레이터)** — "Terraform이 하면 안 되는 루프"의 정답 |
| **GitHub Actions Environments** | 인적 승인 게이트 | required reviewers(1인 승인), prevent self-review, wait timer(1~43,200분) | Free/Pro/Team은 public repo 제한 | **보완(승인 게이트)** — plan artifact 승인 지점 |
| **GitHub Actions OIDC** | 단명 클라우드 자격증명 | 잡당 1회 만료 토큰, 장기 시크릿 제거 | `permissions: id-token: write` 등 설정 필요(설정 가이드 별도) | **보완(자격증명 레이어)** |

---

## 7. 남은 불확실성 + 구현 전 재검증 항목

| # | 불확실성 | 재검증 방법/이유 |
|---|---|---|
| 1 | **MCP Server 1.0.x 도구 목록의 정확한 read/write 경계와 `ENABLE_TF_OPERATIONS` 동작** | `/terraform/mcp-server/reference`와 GitHub Releases를 배포 직전 재확인. 쓰기 도구(create_run/action_run/update_workspace)의 권한·플래그가 버전별로 변동 가능 |
| 2 | **MCP Server 보안 세부(CORS allowed origins, rate limit 기본값, TLS 강제)** | `/terraform/mcp-server/deploy#secure-configuration` 환경변수 표를 직접 확인. 원격 노출 시 API gateway·TLS·IP allowlist 필수 |
| 3 | **HCP run 단계 정밀 순서(cost estimation·policy check·Needs Confirmation의 상대 위치)** | 두 문서(run/states vs cost-estimation)의 표현 차이 존재. cost는 "plan과 apply 사이"가 공식. 구현 전 워크스페이스 실런 로그로 실제 순서 확인 |
| 4 | **cost 정책의 Sentinel 전용 제약(Legacy policy checks 모드)** | OPA로는 cost 데이터 접근 불가가 현재 문서. HCP 정책 실행 모드(Legacy vs 신모드) 변경 여부를 재확인 |
| 5 | **write-only 인자의 provider별 지원 범위** | 사용할 provider 리소스가 `_wo` 인자를 지원하는지 provider 문서로 개별 확인(1.11+ 전제) |
| 6 | **OpenTofu state/plan 암호화 provider·알고리즘(현재 AES-GCM 단일)** | `/docs/language/state/encryption/`는 무버전(latest). 채택 OpenTofu 버전의 지원 provider(PBKDF2/KMS류/OpenBao/External)·실험 상태 재확인 |
| 7 | **Crossplane v2.x의 reconcile/Composition API 형상** | 가장 선명한 "control plane vs CLI" 근거가 2021 blog. v2.3 공식 문서로 Claims 비중·namespaced XR·Composition Functions 변화 재검증. always-on 복구가 본 설계의 kill/teardown과 충돌하는지 PoC 필요 |
| 8 | **Pulumi TTL "생성 시각 기준"의 반복 루프 영향** | 장기 반복 provision→test→teardown 루프에서 TTL이 update로 리셋되지 않음 → 의도치 않은 조기 파기 가능. 시나리오 검증 |
| 9 | **GitHub 문서 경로 재편(Environments/OIDC)** | docs.github.com 경로가 최근 재편(`/actions/reference/...`). "Waiting/30일 만료", `id-token: write` 등 일부 세부는 검색 스니펫 기반 → 구현 시 최종 페이지에서 verbatim 확인 |
| 10 | **K8s Job precedence(activeDeadlineSeconds vs backoffLimit) 및 버전 의존 기능** | concepts/job 페이지 일부 트렁케이션. 클러스터 버전에서 Backoff Limit Per Index 등 기능 GA 여부 확인 |
| 11 | **AWS CDK 경로(만약 AWS 전용 분기 시)** | `--require-approval` 기본값(`broadening`)·change set prepare/execute가 진짜 승인 엔진. CFN change set 기반 게이트로 설계해야 함 |
| 12 | **Firebase/GCP provider의 Firebase 리소스 지원 범위(google-beta 의존)** | 구현 직전 registry(hashicorp/google-beta)와 Firebase get-started로 지원 리소스·GA 승격 여부 재확인. RTDB 보안규칙·MFA·기본 Storage 버킷 미지원 상태가 바뀌었는지 확인(현재 콘솔/CLI/REST 전용) |
| 13 | **운영 중 Firebase 대상의 import vs 신규 생성 결정** | 부하 대상이 살아있는 운영 프로젝트면 IaC는 import/참조만. 신규 격리 대상이 필요하면 ephemeral GCP 프로젝트 생성 권한(조직 정책·billing account)·`google_project` 할당량 사전 확인 |
| 14 | **ephemeral 테스트 프로젝트 teardown 신뢰성** | "fresh project per test" + `project_cleanup`는 **프로덕션 사용 금지** 명시. 프로젝트 ID는 전역·영구(재사용 불가) → random suffix 필수. `destroy` 후 잔여 리소스 정리 경로를 검증 |

---

*문서 끝. 본 문서는 리서치 산출물이며 HCL/코드를 포함하지 않는다. 모든 외부 인용은 데이터로만 취급했다.*
