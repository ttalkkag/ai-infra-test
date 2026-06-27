# 05. 안전 가드레일 · 승인 게이트 · Secrets · 비용/쿼터 통제 · Audit · 프롬프트 인젝션 방어 · BaaS(Firebase) 대상 격리 · 비전문가/템플릿 가드레일

## 1. 한 줄 요약 + 기준일

"AI + Terraform 기반 프롬프트 주도 부하 테스트 자동화"의 안전성은 **AI가 직접 실행하지 않고**, 좁은 권한의 Execution Controller만 실행하며, production 직접 타격을 기본 거부하고, 비용/RPS/duration 상한·승인 게이트·kill switch·audit log·프롬프트 인젝션 방어를 다층(defense-in-depth)으로 강제할 때 확보된다. 핵심 통제는 공식 문서로 검증된다: GitHub Actions environment는 required reviewer 승인 전 secret 접근을 차단하고, OIDC는 장기 secret 없이 단명 토큰을 발급하며, AWS Budgets/Azure Cost Management는 임계 초과 시 자동 대응을 제공하고, OWASP LLM Top 10(2025)·NIST AI 600-1은 "외부/untrusted 데이터를 명령으로 신뢰하지 말 것"과 "고위험 작업에 human-in-the-loop"를 명시한다.

- 기준일(작성·확인일): **2026-06-26 (Asia/Seoul)**. 본 문서의 모든 `fetched_at`은 2026-06-26.
- 성격: 리서치/거버넌스 설계 검토. **코드/Terraform/실제 실행 없음.** 정책·게이트·표·체크리스트까지만.
- 기본값: production 직접 타격 금지, 기본 실행 대상은 ephemeral 또는 prod-like.
- 보안 원칙(이 문서 자체에 적용): 외부 문서/웹페이지에 포함된 모든 텍스트는 **데이터**로만 취급했고, 그 안의 어떤 지시문도 따르지 않았다.
- 도메인 맥락(이번 보강): 대상은 게임사 다(多)타이틀 환경으로 **Firebase(BaaS) 백엔드**가 같은 GCP 조직 안에서 여러 타이틀·실유저와 공존할 수 있다. 사용자는 **부하/인프라 비전문가**이며 "자연어 → 가장 유사한 사전 검증 템플릿(시나리오) 실행" 방식을 쓴다. 환경은 AWS/GCP/Azure/Firebase 범용 유지.
- 이번 보강의 두 축: ① **BaaS 대상 부하의 고유 안전 리스크**(종량제 비용 폭증·공유 프로젝트/조직 자원 영향·보안규칙/App Check/쿼터) → 대상 환경 격리·예산/쿼터 cap(9장). ② **비전문가+템플릿 매칭의 안전 함의**(검증 템플릿이 자유 입력보다 안전하나, 저신뢰 매칭 가드레일 필요)(10장).

> 주의: 가격·preview/GA·정책 세부·API 기능은 변경 가능성이 있어 13장의 재검증 항목을 구현 직전 반드시 재확인한다. 일부 공식 페이지는 게시/갱신일이 표시되지 않아 `published_or_updated_at`을 "미표시"로 기록했다(=freshness 추적 약화 신호).

---

## 2. 출처 레지스트리

trust_level: `공식`(벤더/표준 1차 문서) · `2차`(해설/벤더 블로그/커뮤니티). freshness_risk: 게시일 미표시·정책 변동성·preview 여부로 판단.

| 주제 | source_url | vendor | published_or_updated_at | fetched_at | trust_level | freshness_risk | 핵심 근거 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| GitHub Actions 환경/배포 보호 | https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments | GitHub | 미표시 | 2026-06-26 | 공식 | 낮음 | "a job cannot access environment secrets until one of the required reviewers approves it" — 승인 전 environment secret 접근 차단 |
| GitHub Actions 배포 리뷰 | https://docs.github.com/actions/managing-workflow-runs/reviewing-deployments | GitHub | 미표시 | 2026-06-26 | 공식 | 낮음 | required reviewer가 승인/거부할 때까지 job pause, secret 미노출 |
| GitHub Actions OIDC 개념 | https://docs.github.com/en/actions/concepts/security/openid-connect | GitHub | 미표시 | 2026-06-26 | 공식 | 낮음 | "without having to store any credentials as long-lived GitHub secrets"; cloud가 단일 job에만 유효한 short-lived token 발급 |
| GitHub OIDC 클라우드 구성 | https://docs.github.com/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-cloud-providers | GitHub | 미표시 | 2026-06-26 | 공식 | 낮음 | `permissions: id-token: write` 필요, cloud별 신뢰 구성 |
| AWS Budgets 액션 구성 | https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-controls.html | AWS | 미표시 | 2026-06-26 | 공식 | 중간 | "configure a budget action to run either automatically or after your manual approval"; IAM policy/SCP 적용, EC2/RDS 타깃 |
| AWS Budgets 액션 검토/승인 | https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-action-configure.html | AWS | 미표시 | 2026-06-26 | 공식 | 중간 | 임계 초과 시 자동 실행 vs Alert details에서 수동 실행 선택 |
| AWS 침투테스트 정책 | https://aws.amazon.com/security/penetration-testing/ | AWS | 미표시 | 2026-06-26 | 공식 | 중간 | DoS/DDoS/Simulated DoS/request·port·protocol flooding 금지; Simulated Events form는 시작 2주 전 제출 |
| AWS EC2 Testing/네트워크 스트레스 정책 | https://aws.amazon.com/ec2/testing/ | AWS | 미표시 | 2026-06-26 | 공식 | 중간 | 페이지 원문: "traffic surges exceed **25Gbps** or over **100Gbps**"(Gbps) 시 AWS가 traffic shaping 적용 가능; "Volumetric network-based DDoS simulations are **explicitly prohibited**"(별도 DDoS Simulation Testing 정책). 대량 네트워크 스트레스 테스트는 AWS 네트워크 스트레스 테스트 신청으로 사전 조율 |
| Azure 침투테스트(Pen testing) | https://learn.microsoft.com/en-us/azure/security/fundamentals/pen-testing | Microsoft | 미표시 | 2026-06-26 | 공식 | 중간 | 2017-06-15부터 사전 승인 불필요, 단 Rules of Engagement 준수; DDoS는 승인 파트너(BreakingPoint Cloud) 사용 |
| Microsoft 보안테스트 ROE | https://www.microsoft.com/en-us/msrc/pentest-rules-of-engagement | Microsoft | 미표시 | 2026-06-26 | 공식 | 중간 | "Simulating high traffic loads ... is encouraged"; "DDoS ... strictly prohibited under all circumstances"; 자신/명시적 권한 자산만 |
| Azure Cost Management 예산 튜토리얼 | https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/tutorial-acm-create-budgets | Microsoft | 미표시 | 2026-06-26 | 공식 | 중간 | actual/forecasted 임계 알림, action group 연동 |
| Azure 예산+액션그룹+런북 자동화 | https://techcommunity.microsoft.com/blog/azureinfrastructureblog/automating-azure-openai-cost-control-using-budgets-action-groups-and-automation-/4505164 | Microsoft(블로그) | 2025 | 2026-06-26 | 2차 | 중간 | budget→action group→Automation Runbook으로 임계 초과 시 API 비활성, 수동 복구 패턴 |
| HCP Terraform 비용 추정 | https://developer.hashicorp.com/terraform/cloud-docs/workspaces/cost-estimation | HashiCorp | 미표시 | 2026-06-26 | 공식 | 중간 | "between the plan and apply"; cost 데이터는 Sentinel `tfrun` import로 접근; 기본 비활성 |
| HCP Terraform 정책 결과 | https://developer.hashicorp.com/terraform/cloud-docs/workspaces/policy-enforcement/view-results | HashiCorp | 미표시 | 2026-06-26 | 공식 | 중간 | Plan→run tasks+cost estimation→Sentinel 순서; advisory/soft-mandatory/hard-mandatory |
| HCP Terraform 정책셋 관리 | https://developer.hashicorp.com/terraform/cloud-docs/workspaces/policy-enforcement/manage-policy-sets | HashiCorp | 미표시 | 2026-06-26 | 공식 | 중간 | OPA policy evaluation은 cost estimate 데이터 접근 불가 → 비용 정책은 Sentinel policy check |
| OWASP Top 10 for LLM 2025 | https://owasp.org/www-project-top-10-for-large-language-model-applications/ | OWASP | 2025 | 2026-06-26 | 공식 | 낮음 | LLM01 Prompt Injection ~ LLM10 Unbounded Consumption |
| OWASP LLM01 Prompt Injection | https://genai.owasp.org/llmrisk/llm01-prompt-injection/ | OWASP | 2025 | 2026-06-26 | 공식 | 낮음 | direct/indirect 정의 + 7개 완화책(제약/출력검증/입출력필터/최소권한/고위험 human approval/외부콘텐츠 분리/적대적 테스트) |
| OWASP LLM06 Excessive Agency | https://genai.owasp.org/llmrisk/llm062025-excessive-agency/ | OWASP | 2025 | 2026-06-26 | 공식 | 낮음 | excessive functionality/permission/autonomy; 고위험 작업 human approval; downstream system complete mediation 권고 |
| OWASP LLM Top10 PDF v2025 | https://owasp.org/www-project-top-10-for-large-language-model-applications/assets/PDF/OWASP-Top-10-for-LLMs-v2025.pdf | OWASP | 2024-11(v2025) | 2026-06-26 | 공식 | 낮음 | 전체 카테고리 1차 출처 |
| NIST AI 600-1 GenAI Profile | https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf | NIST | 2024-07, Editorial Review Board 승인 2024-07-25 | 2026-06-26 | 공식 | 중간 | AI RMF 1.0의 Generative AI Profile 최종본. GAI risk를 Govern/Map/Measure/Manage 액션으로 관리 |
| Anthropic 안전 에이전트 프레임워크 | https://www.anthropic.com/news/our-framework-for-developing-safe-and-trustworthy-agents | Anthropic | 2025 | 2026-06-26 | 공식 | 중간 | model/harness/tools/environment 4구성, 각 지점이 oversight 포인트 |
| Anthropic Trustworthy agents | https://www.anthropic.com/research/trustworthy-agents | Anthropic | 2025 | 2026-06-26 | 공식 | 중간 | per-tool permission(always allow/needs approval/block); 사전 plan 검토·승인; 고위험 전 인간 통제 유지 |
| AWS SCP(서비스 제어 정책) | https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html | AWS | 미표시 | 2026-06-26 | 공식 | 낮음 | 조직 계정의 최대 권한 경계(preventive), Deny 우선 |
| OPA + Terraform(공식) | https://www.openpolicyagent.org/docs/terraform | OPA(CNCF) | 미표시 | 2026-06-26 | 공식 | 낮음 | `opa eval`로 plan 파일 직접 평가, Rego 정책 |
| OPA/Sentinel/Checkov 비교 | https://spacelift.io/blog/terraform-policy-as-code | Spacelift(블로그) | 2025 | 2026-06-26 | 2차 | 중간 | plan 기반(OPA/Sentinel) vs HCL static(Checkov) 차이 |
| Vault vs Secrets Mgr vs Key Vault | https://sanj.dev/post/hashicorp-vault-aws-secrets-azure-key-vault-comparison/ | 개인 블로그 | 2026 | 2026-06-26 | 2차 | 중간 | Vault만 진정한 dynamic short-lived secret, AWS/Azure는 rotation 기반 |
| Anthropic API rate/spend limit | https://platform.claude.com/docs/en/api/rate-limits | Anthropic | 미표시 | 2026-06-26 | 공식 | 중간 | RPM/TPM/TPD rate limit + workspace별 spend limit |
| Anthropic Workspaces | https://platform.claude.com/docs/en/manage-claude/workspaces | Anthropic | 미표시 | 2026-06-26 | 공식 | 중간 | workspace당 custom spend/rate limit 설정 |
| OpenAI rate limits | https://platform.openai.com/docs/guides/rate-limits | OpenAI | 미표시 | 2026-06-26 | 공식 | 중간 | RPM/RPD/TPM/TPD + 조직 월간 usage(spend) limit |
| OWASP WSTG Authorization Testing | https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/05-Authorization_Testing/README | OWASP | latest | 2026-06-26 | 공식 | 낮음 | 명시적 권한 없는 테스트는 다수 관할권에서 위법, 권한 확보 원칙 |
| Firebase 청구 경고(예상치 못한 청구) | https://firebase.google.com/docs/projects/billing/avoid-surprise-bills | Google/Firebase | 미표시 | 2026-06-26 | 공식 | 중간 | "Budgets and budget alerts do not cap your usage or charges"; 알림 며칠 지연 가능; 미제한 쿼리·무한 fan-out 주의; Local Emulator 권장 |
| Firebase 요금제(Blaze 종량제) | https://firebase.google.com/docs/projects/billing/firebase-pricing-plans | Google/Firebase | 미표시 | 2026-06-26 | 공식 | 중간 | Blaze = pay-as-you-go(사용량 기반 과금) → 부하=즉시 비용 |
| Firebase 환경 분리(별도 프로젝트) | https://firebase.google.com/docs/projects/dev-workflows/overview-environments | Google/Firebase | 미표시 | 2026-06-26 | 공식 | 낮음 | "Firebase recommends using a separate Firebase project for each environment"; "at least one pre-production environment that's isolated from production data and resources" |
| Firebase App Check | https://firebase.google.com/products/app-check | Google/Firebase | 미표시 | 2026-06-26 | 공식 | 낮음 | "helps protect your backend from abuse, such as billing fraud, phishing, app impersonation, and data poisoning"; 미검증(attestation 없는) 클라이언트 요청 차단 |
| Firebase RTDB 한도 | https://firebase.google.com/docs/database/usage/limits | Google/Firebase | 미표시 | 2026-06-26 | 공식 | 중간 | 동시 연결 한도 = 인스턴스(DB)당 200,000 |
| Firestore 사용량/한도 | https://firebase.google.com/docs/firestore/quotas | Google/Firebase | 미표시 | 2026-06-26 | 공식 | 중간 | per-project 쿼터·한도(대상 측 쿼터가 blast radius 한계) |
| GCP Cloud Quotas 개요 | https://docs.cloud.google.com/docs/quotas/overview | Google Cloud | 미표시 | 2026-06-26 | 공식 | 낮음 | "Your use of a resource in one project doesn't affect your available quota in another project"; rate/allocation quota; 초과 시 차단·실패 |
| GCP 예산/예산알림 | https://docs.cloud.google.com/billing/docs/how-to/budgets | Google Cloud | 미표시 | 2026-06-26 | 공식 | 중간 | "Setting a budget does not cap resource or API consumption" |
| GCP billing 자동 비활성(Pub/Sub) | https://docs.cloud.google.com/billing/docs/how-to/disable-billing-with-notifications | Google Cloud | 미표시 | 2026-06-26 | 공식 | 중간 | 예산 초과 시 billing 자동 비활성 가능; "removes Cloud Billing from your project, shutting down all resources. Resources might be irretrievably deleted" |
| GCP Cloud Security FAQ(펜테스트/AUP) | https://support.google.com/cloud/answer/6262505 | Google Cloud | 미표시 | 2026-06-26 | 공식 | 중간 | 자기 프로젝트 보안평가는 "not required to contact us"; AUP/ToS 준수; "tests only affect your projects (and not other customers' applications)" |

---

## 3. 작업 3분류 (자동 허용 / 인간 승인 필수 / 기본 금지)

분류 기준: 가역성(reversible/costly/irreversible), blast radius, 대상 환경, 비용·데이터 영향. OWASP LLM06 원칙("irreversible 작업은 human approval", "LLM에게 인가 판단을 위임하지 말 것")과 Anthropic per-tool permission 모델을 적용.

### 3.1 자동 허용 (read-only · dry-run · 비파괴, 인간 승인 없이 진행)

| 작업 | 근거 |
| --- | --- |
| 공식 문서/표준 검색, source metadata 생성 | 읽기 전용, 외부 콘텐츠는 데이터로만 취급(OWASP LLM01 "segregate external content") |
| 사용자 의도 → `UserIntent` 구조화 | 부수효과 없음 |
| `TestPlanSpec`/`ProvisionSpec`/`ToolAction` **draft** 생성 | 실행 권한 없는 산출물 |
| static policy check (Checkov/OPA `opa eval` plan 평가) | plan/HCL 분석은 비파괴(OPA Terraform 공식 문서) |
| script template fill + lint/schema validation | 실행하지 않음, `dry_run=true` |
| Terraform `fmt`/`validate`/`plan -out`/`show -json` | plan 생성은 상태 변경 아님(읽기·계획). 단 plan artifact는 승인 단위로 보존 |
| read-only metrics/baseline/trace 조회 | 관측 데이터 읽기, 부하 미발생 |
| HCP cost estimation 조회(plan-apply 사이) | HashiCorp 문서상 plan/apply 사이 추정 단계, 비파괴 |

### 3.2 인간 승인 필수 (가역적이나 비용·blast radius 큼 → 승인 게이트 통과 후)

| 작업 | 근거 |
| --- | --- |
| `terraform apply <approved-plan>` | apply는 실제 리소스 변경. saved plan은 Terraform이 "승인된 것"으로 간주하므로 plan artifact+approval id+hash 결속 필요 |
| `terraform destroy` | 가역적이나 의도치 않은 삭제 위험 |
| staging/shared/prod-like 대상 실제 부하 실행 | 공유 자원 영향, on-call 인지 필요 |
| secrets/privileged identity 사용 | 승인 전 secret 접근 차단(GitHub environment required reviewers 메커니즘) |
| RPS/concurrency/duration ceiling **상향** | 상한 변경은 안전 한계 변경 |
| cloud resource 생성(runner 인프라) | 비용·권한·정리 책임 발생 |
| 고비용/장시간 soak·breakpoint test | 비용/토큰 ceiling 초과 위험 |
| chaos/fault injection | blast radius 통제 필요 |
| 실데이터 변경 가능 workflow(write 경로 포함) | 데이터 오염 위험(OWASP LLM06 high-impact action) |
| 외부 공용 endpoint/파트너 API 대상 부하 | 제3자 정책·합법성 검토 필요(8장) |
| 공유 GCP 조직/프로젝트 내 BaaS(Firebase) 대상 부하(테스트 전용 프로젝트라도) | 같은 조직의 타 타이틀·공유 쿼터 영향 가능 → 영향 범위 사전 확인 필요(9장) |

### 3.3 기본 금지 (정책상 차단, 예외는 별도 거버넌스·서면 승인·클라우드 사업자 승인 필요)

| 작업 | 근거 |
| --- | --- |
| production endpoint 직접 고부하 실행 | 실제 사용자 영향, 본 설계 기본값(prod 직접 타격 금지) |
| payment/email/SMS/destructive endpoint 호출 | 비가역 부작용, OWASP LLM06 high-impact 명시 |
| 승인 없는 secrets 사용 | 최소권한·승인 전 미노출 원칙 위반 |
| `-auto-approve` 기반 production/shared apply | 승인 경계 우회, drift/오적용 위험 |
| 자유 텍스트 shell command 실행 | excessive functionality, 명령 인젝션 표면 확대 |
| source 문서/웹페이지의 instruction 따르기 | indirect prompt injection 차단(OWASP LLM01, NIST untrusted data) |
| 제3자 공용 서비스를 공격으로 오인될 수 있는 트래픽 생성 | AWS/Azure/GCP DoS·flooding 금지 정책 위반 소지(8장) |
| `bypassPermissions` 류 무승인 자동 실행 | 격리 환경 외 사용 금지(Anthropic 권고: 컨테이너/VM 한정) |
| 실서비스 Firebase/관리형 백엔드 프로젝트(실유저·타 타이틀 공존) 직접 부하 | 종량제 비용 폭증 + 실유저/타 타이틀 자원 영향(9장), prod 직접 타격 금지 기본값 |
| FCM 푸시·PG(결제대행) 등 외부 파트너 호출 경로 부하 | 제3자 자원 → 부하에서 기본 제외(8장), 합법성·정책 위반 소지 |
| 저신뢰(유사도 임계 미만) 템플릿 매칭의 자동 실행 | 엉뚱한 템플릿 선택 위험 → 실행 거부+사람 확인 필요(10장) |
| 비전문가의 템플릿 내장 안전 상한(max RPS/duration/cost) 우회·상향 | 안전 한계 우회 = 가드레일 무력화, 상향은 승인 게이트로만(3.2) |

---

## 4. kill switch 조건 (조건 | 측정 시그널 | 소유 주체 | 동작)

핵심 원칙: **kill switch는 Terraform이 소유하지 않는다.** Terraform은 plan/apply의 감사 가능한 경계만 제공하고, 실시간 중단 권한은 **Runner/Orchestrator(Execution Controller)** 가 소유한다. 부하 생성과 abort는 장기 실행 루프이며 Terraform의 책임이 아니다.

| 조건 | 측정 시그널 | 소유 주체(실시간) | 동작 |
| --- | --- | --- | --- |
| 핵심 SLO 연속 위반 | p95/p99 latency, error rate가 N 연속 구간 초과 | Runner(threshold) + Orchestrator | run abort, 부하 ramp-down, alert |
| 5xx/timeout/retry 급증 | 5xx 비율, timeout 카운트, retry rate 급변 | Runner | 즉시 abort, baseline 보존 |
| queue/DB/CPU/memory saturation 지속 | queue depth, DB conn pool, CPU/mem util, autoscale delay | Observer→Orchestrator | abort, hold-for-debug |
| production alert firing | 대상/주변 서비스 alert(PagerDuty/Monitor/CloudWatch) | Orchestrator(알림 연동) | 즉시 중단, on-call 통보 |
| 실제 사용자 영향 관측 | 실사용자 트래픽 오류·지연 상승 | Orchestrator + on-call | 즉시 중단, 사후 검토 |
| 비용/토큰/API 호출 ceiling 초과 | 누적 비용, LLM token, API call 카운트 | Orchestrator(예산 워처) + cloud budget action | abort + 자동 비용 차단(5장) |
| 데이터 오염/중복 쓰기 징후 | write count 이상, idempotency 위반, dup record | Runner/Orchestrator | 즉시 중단, write 경로 격리 |
| on-call/서비스 오너 중단 선언 | 수동 중단 명령 | 인간(최우선) | 무조건 즉시 abort |

> 설계 메모: AWS/Azure 클라우드 예산 액션(5장)은 **비용 ceiling 킬스위치의 백스톱**으로만 본다. 알림은 최대 24시간 지연 가능(Azure 문서)하므로, 실시간 비용·부하 중단은 Orchestrator가 보유한 자체 워처가 1차로 담당해야 한다.

---

## 5. 비용/쿼터/RPS/duration 상한 강제 메커니즘 (클라우드별)

### 5.1 비용 상한 (cloud-native budget)

| 클라우드 | 메커니즘 | 자동 차단 범위 | 한계/주의 | 출처 |
| --- | --- | --- | --- | --- |
| AWS | AWS Budgets + Budget Actions | 임계 초과 시 IAM Deny 정책 적용, SCP 적용, EC2/RDS 인스턴스 중지 — "automatically or after your manual approval" 선택 | management 계정에서 SCP 적용 가능하나 타 계정 EC2/RDS 타깃 불가; 비용 데이터는 후행 집계 → 실시간 아님 | AWS Budgets controls |
| Azure | Cost Management Budgets + Action Groups (+ Automation Runbook) | action group→webhook/Function/Runbook으로 API 비활성 등 자동 대응, 수동 복구 | action group은 subscription/resource group scope만; 알림 최대 24h 지연 | Azure budgets tutorial / techcommunity 패턴 |
| GCP | Cloud Billing Budgets + 프로그래매틱 알림(Pub/Sub→Cloud Function) | budget 자체는 미차단("Setting a budget does not cap resource or API consumption"); Pub/Sub로 임계 초과 시 **billing 자동 비활성** 백스톱 구성 가능 | billing 비활성은 파괴적("shutting down all resources. Resources might be irretrievably deleted") → 테스트 전용 프로젝트에만 | GCP budgets / disable-billing-with-notifications |
| Firebase(BaaS) | Cloud Billing budget alert + App Check + 프로젝트 쿼터 | budget alert는 **cap 아님**("do not cap your usage or charges"), 알림 지연 → 즉시 차단용 아님; 실질 상한은 프로젝트 쿼터·App Check enforcement | Blaze=종량제라 부하=즉시 비용; 실서비스 프로젝트 직접 타격 금지(9장) | Firebase avoid-surprise-bills / App Check |
| HCP Terraform | Cost Estimation + Sentinel(`tfrun.cost_estimate`) | apply **이전** plan 단계에서 예상 비용 기준 정책 거부 | 기본 비활성(조직 설정 enable 필요); OPA evaluation은 cost 데이터 접근 불가 → 비용 정책은 Sentinel | HCP cost-estimation / policy-sets |
| LLM(Anthropic) | Workspace spend limit + rate limit(RPM/TPM/TPD) | workspace당 월 spend 상한, 조직 한도 이하로만 설정 | 토큰 ceiling은 자체 워처로 실시간 보강 권장 | Anthropic rate-limits / Workspaces |
| LLM(OpenAI) | 조직 usage(spend) limit + rate limit(RPM/RPD/TPM/TPD) | 월간 spend 상한, tier별 rate | tier 자동 승급으로 한도 상승 가능 → 명시적 상한 고정 필요 | OpenAI rate-limits |

핵심: 클라우드 비용 액션은 **자동 실행 또는 수동 승인** 두 모드를 지원한다(AWS 명시). 부하 테스트 자동화에서는 (a) 80~90% 임계 알림 + (b) ceiling 도달 시 자동 Deny/중지 + (c) Orchestrator 자체 abort를 **3중 백스톱**으로 둔다.

### 5.2 RPS / concurrency / duration 상한 강제

Terraform/클라우드 예산만으로는 RPS·duration을 직접 못 막는다(비용은 후행 집계). 따라서 다층 강제가 필요하다.

| 통제 지점 | 강제 방법 | 비고 |
| --- | --- | --- |
| TestPlanSpec 스키마 | `safety_limits`(max RPS/VU/duration/cost) 필수 필드, 초과 시 plan 거부 | 정책 게이트에서 1차 차단 |
| Runner 설정 강제 | k6 executors의 `maxVUs`/arrival-rate 상한, duration 상한을 정책 템플릿으로 고정(자유 생성 금지) | script generator가 상한 우회 불가하게 template fill |
| Policy-as-code | Sentinel/OPA로 ProvisionSpec의 runner 수/인스턴스 크기/region 상한 검증 | plan 단계 hard-mandatory |
| 클라우드 쿼터 | runner용 별도 계정/구독/프로젝트의 service quota를 낮게 고정(예: 동시 Fargate task, vCPU) | blast radius 물리적 제한 |
| 대상 측 쿼터(관리형/BaaS) | 부하 **대상**이 관리형 서비스면 대상 프로젝트 쿼터(GCP rate/allocation, Firebase RTDB 동시연결·Firestore ops)가 사실상 blast radius 한계 → 테스트 프로젝트 쿼터를 낮게 고정 | GCP: "초과 시 시스템이 차단·실패"; 프로젝트 격리 시 타 프로젝트 쿼터 무영향(9장) |
| Orchestrator 워처 | 실시간 RPS·duration 모니터, ceiling 초과 시 abort(kill switch) | 후행 비용 집계 지연 보완 |
| 시간창(time window) | 승인된 maintenance window 외 실행 차단 | prod-like 보호 |

---

## 6. secrets/credential handling 원칙 + 대체 수단 비교

### 6.1 원칙

1. **장기 secret 대신 단명 자격증명**: GitHub Actions OIDC로 cloud에 federated identity를 신뢰시키고, 매 job마다 단일 실행에만 유효한 short-lived token을 발급받는다("without having to store any credentials as long-lived GitHub secrets"). 장기 cloud access key를 GitHub secret에 저장하지 않는다.
2. **승인 전 secret 접근 불가**: GitHub environment에 required reviewers를 걸면 "a job cannot access environment secrets until one of the required reviewers approves it." → 승인 게이트(3.2)와 secret 노출 경계를 일치시킨다. prevent self-review로 self-approval도 차단.
3. **ephemeral/write-only 지향**: Terraform `sensitive`만으로는 state/plan 저장을 막지 못한다. 지원되는 provider에서 ephemeral/write-only 패턴을 검토하고, runner 자격은 managed identity/STS 단명 토큰으로 공급한다(state 저장 회피).
4. **최소권한**: runner IAM은 대상 부하 호출과 artifact 쓰기로만 한정(OWASP LLM06 least privilege). LLM/Orchestrator는 secret 접근 권한 자체를 갖지 않는다.
5. **secret은 인가 판단의 주체가 아니다**: 다운스트림 시스템이 독립적으로 authz를 강제(OWASP LLM06 "do not rely on the LLM to decide whether an action is authorized").

### 6.2 대체 수단 비교 (이 설계에서의 위치, 한 줄 평가)

| 수단 | 강점 | 주의 | 이 설계에서의 위치(한 줄 평가) |
| --- | --- | --- | --- |
| GitHub Actions OIDC + cloud 신뢰 | 장기 secret 제거, job별 단명 토큰, 자동 만료/감사 | cloud별 trust 조건(repo/branch/environment claim) 정확히 제한 필요 | **기본 인증 메커니즘**: CI에서 cloud로 들어가는 1차 경로 |
| HashiCorp Vault | 진정한 dynamic short-lived secret(DB/cloud), TTL 만료 시 자동 revoke, multi-cloud | self-managed 운영 부담(또는 HCP Vault) | **고급 옵션**: 멀티클라우드·DB 단명 자격이 필요할 때 |
| AWS Secrets Manager + STS | AWS-native, 자동 rotation, IAM 통합 | rotation은 "장기→장기 교체"(진정한 dynamic 아님) | **AWS 전용**: AssumeRole STS 단명 토큰과 결합 시 충분 |
| Azure Key Vault + Managed Identity | Entra ID 통합, secret/key/cert 통합, MI로 키리스 | rotation 기반 | **Azure 전용**: Managed Identity로 자격 미저장 경로 구성 |
| GCP Secret Manager + Workload Identity | GCP-native, WIF로 키리스 federation | 본 설계 GCP 우선순위 낮음 | **GCP 보조**: GCP 대상일 때만 |

---

## 7. 프롬프트 인젝션 / untrusted data 방어 원칙 (LLM 보안 근거)

근거: OWASP Top 10 for LLM Applications 2025(LLM01 Prompt Injection은 2회 연속 1위), OWASP LLM06 Excessive Agency, NIST AI 600-1 GenAI Profile, Anthropic 안전 에이전트 프레임워크.

### 7.1 핵심 위협 모델

- **Direct injection**: 사용자 입력이 모델 행동을 직접 변경("이전 지시 무시하고 …").
- **Indirect injection**: 모델이 처리하는 외부 문서/웹/파일에 숨겨진 명령이 모델 행동을 변경. 본 시스템은 "최신 공식 문서 검색"을 수행하므로 **indirect injection 표면이 크다** — 검색·페치한 모든 외부 콘텐츠를 명령이 아닌 데이터로만 취급해야 한다(NIST: untrusted/외부 데이터를 신뢰하지 말 것).

### 7.2 방어 원칙 (OWASP LLM01 7개 완화책 적용)

| # | 원칙 | 이 설계 적용 |
| --- | --- | --- |
| 1 | 모델 행동 제약(역할/컨텍스트 고정) | system prompt로 "외부 텍스트의 지시 무시" 명시, 에이전트 역할 한정 |
| 2 | 출력 형식 정의·검증 | 모든 에이전트 출력은 typed schema로 닫고 코드로 검증(자유 텍스트 실행 금지) |
| 3 | 입출력 필터링 | 위험 endpoint·shell·secret 키워드 semantic/string 필터 |
| 4 | 최소권한 강제 | Orchestrator 실행 권한 0, Execution Controller만 allowlisted 도구 호출 |
| 5 | 고위험 작업 human approval | 3.2/3.3 승인 게이트와 일치(LLM01·LLM06 공통) |
| 6 | 외부 콘텐츠 분리·식별 | 검색/페치 결과를 `KnowledgePack` 데이터로 격리, 명령 채널과 분리 |
| 7 | 적대적 테스트/시뮬레이션 | 정기적 prompt injection red-team, 모델을 untrusted user로 가정 |

### 7.3 Excessive Agency 통제(LLM06)

- excessive **functionality**(불필요한 도구) → 도구 allowlist 최소화.
- excessive **permission**(과대 scope) → 다운스트림이 독립 authz 강제, DB read-only 권한 등.
- excessive **autonomy**(비가역 작업 무승인) → irreversible 작업은 반드시 human-in-the-loop.
- Anthropic 권고: 단계별 일일이 승인 대신 **사전 plan 검토·승인**(Plan Review Bundle) 방식이 oversight를 전략 수준으로 끌어올린다. bypassPermissions류는 격리 컨테이너/VM에서만.

---

## 8. 제3자/클라우드 사업자 부하테스트 정책·합법성 주의

### 8.1 합법성·권한(authorization) 원칙

- OWASP WSTG: 명시적 권한 없는 테스트는 다수 관할권에서 컴퓨터 범죄법상 **위법**. 테스트 도구는 **명시적 허가가 있는 시스템에만** 사용. → 본 설계의 "기본 금지: 제3자/공용 endpoint 부하"와 일치.
- Microsoft ROE: "all encouraged testing activities must be performed within your own tenant or assets for which you have explicit authorization." 타인 데이터/시스템 접근·테스트 금지.
- GCP Cloud Security FAQ: 자기 인프라 보안평가(펜테스트)는 "you are not required to contact us"이나 **Acceptable Use Policy/ToS 준수** 및 "tests only affect your projects (and not other customers' applications)" 필수 → 공유 인프라/타 고객(및 같은 조직 내 타 타이틀) 영향 형태 금지.
- **외부 파트너/제3자 경로 기본 제외**: 게임 백엔드라도 PG(결제대행)·FCM 푸시·외부 파트너 API 호출 경로는 **제3자 자원**이므로 부하에서 기본 제외(3.3). 부하 대상은 자기 소유/명시적 권한 자산으로 한정.

### 8.2 AWS 정책 (penetration testing / network stress test)

| 항목 | 정책 | 함의 |
| --- | --- | --- |
| 사전 승인 | 승인된 서비스(EC2, RDS, API Gateway, Lambda, S3, ELB, CloudFront, Aurora, ECS/Fargate, OpenSearch 등)는 사전 승인 불필요 | 자기 소유 자원 한정 |
| 금지 행위 | DoS, DDoS, **Simulated DoS/Simulated DDoS**, port/protocol/request flooding(로그인·API request flooding 포함) 금지 | 고RPS 부하가 flooding으로 오인되지 않도록 자기 자원·정상 트래픽 범위 유지 |
| 네트워크 스트레스 테스트 | `/ec2/testing/` 원문: 트래픽 서지가 **25Gbps 또는 100Gbps**(Gbps, 패킷 아님) 초과 시 AWS가 traffic shaping 적용 가능. 대량 네트워크 스트레스 테스트는 **AWS 네트워크 스트레스 테스트 신청**으로 사전 조율(구 정책의 ">1Gbps 또는 >1Gpps·1분 지속" 트리거는 신청 절차 페이지에서 재확인) | 대규모 부하는 사전 별도 승인 경로 필요 |
| Simulated Events | DDoS 시뮬레이션 등은 Simulated Events form 제출, **시작 2주 전** | 대규모/시뮬레이션은 리드타임 확보 |

### 8.3 Azure 정책

| 항목 | 정책 | 함의 |
| --- | --- | --- |
| 사전 승인 | 2017-06-15부터 Azure 자원 펜테스트 사전 승인 불필요, 단 Microsoft Cloud Unified Penetration Testing **Rules of Engagement 준수** | ROE가 사실상 정책 경계 |
| 부하 테스트 | "Simulating high traffic loads ... to evaluate performance and scalability ... is encouraged"; "Generating traffic to test surge capacity" 허용 | 자기 앱 대상 고트래픽 시뮬레이션은 권장 활동 |
| DoS/DDoS | "Distributed Denial-of-service (DDoS) attacks are strictly prohibited under all circumstances"; DoS 테스트 금지 | 부하 ≠ DoS. DoS/DDoS 형태 금지 |
| DDoS 복원력 테스트 | Microsoft 승인 시뮬레이션 파트너(BreakingPoint Cloud 등) 통해서만 | 자체 DDoS 시뮬레이션 금지 |
| 대상 | 자기 tenant/명시적 권한 자산만 | 타 테넌트 금지 |

### 8.4 GCP / Firebase(BaaS) 정책

| 항목 | 정책 | 함의 |
| --- | --- | --- |
| 사전 승인 | 자기 프로젝트 보안평가(펜테스트)는 사전 통지 불필요, 단 Acceptable Use Policy/ToS 준수 | 자기 자원 한정, 정책 경계 = AUP |
| 대상 제한 | "tests only affect your projects (and not other customers' applications)" | 공유 인프라/타 고객 영향 금지 → **같은 GCP 조직 내 타 타이틀·실유저 영향도 회피 대상** |
| DoS/멀티테넌트 | 서비스 가용성 저해(DoS/DDoS)·과도 트래픽 유발 자동 스캔 금지(AUP) | 부하 ≠ DoS; 종량제 BaaS는 비용·자원 영향 측면에서 추가 주의 |
| BaaS 종량제 | Blaze=사용량 과금 → 부하=즉시 비용, budget alert는 cap 아님 | 실서비스 프로젝트 직접 타격 금지, 테스트 전용 프로젝트 격리(9장) |

> 결론: 본 자동화는 **자기 소유/명시적 권한 자산**에 대해, **DoS·flooding 형태가 아닌** 정상 트래픽 패턴의 부하만 수행한다. AWS·Azure·GCP 모두 **DoS/DDoS·타 고객(멀티테넌트) 영향 형태를 금지**하며, GCP/Firebase 대상은 "tests only affect your projects" 원칙상 같은 조직 내 타 타이틀 영향까지 회피해야 한다. 대량 네트워크 스트레스(AWS는 25/100Gbps 초과 서지에 traffic shaping 적용) 또는 DDoS 시뮬레이션(AWS에서 volumetric DDoS 시뮬 명시적 금지)은 기본 금지로 두고, 필요 시 AWS 네트워크 스트레스 테스트 신청 / DDoS Simulation Testing 정책·Simulated Events form / Azure 승인 파트너 경로로 별도 거버넌스 처리한다. PG·FCM 등 외부 파트너 경로는 부하 대상에서 기본 제외한다.

---

## 9. BaaS(Firebase 등) 대상 부하의 고유 안전 리스크 · 대상 환경 격리

BaaS(Backend-as-a-Service; Firebase 등)는 관리형·**사용량 기반 과금** 구조라 IaaS 대비 안전 모델이 다르다. 게임사처럼 한 GCP 조직 안에 20+ 타이틀이 공존하고 실유저가 붙어 있을 수 있는 경우, 부하는 곧 **비용 폭증**과 **실유저·타 타이틀 자원 영향**으로 직결된다. 본 절은 그 고유 리스크와 통제를 정리하고, AWS/Azure/GCP 멀티 대상에도 동일하게 적용되는 **대상 환경 격리·비용 cap** 원칙으로 일반화한다.

### 9.1 BaaS 고유 안전 리스크 (web 검증, 출처 2장)

| 리스크 | 메커니즘 | 근거(공식) | 통제 |
| --- | --- | --- | --- |
| ① 비용 폭증 | Blaze는 종량제(pay-as-you-go)라 부하=즉시 비용. budget alert는 **사용/요금을 cap하지 않으며**("do not cap your usage or charges") 알림이 며칠 지연될 수 있어 폭증을 사후에야 인지. 미제한 쿼리·무한 fan-out이 비용을 증폭 | Firebase avoid-surprise-bills | 테스트 전용 프로젝트 격리 + 예산 워처(Orchestrator) + (선택) Pub/Sub→billing 자동 비활성 백스톱 |
| ② 같은 프로젝트/조직 자원·실유저 영향 | 같은 Firebase 프로젝트의 Firestore/RTDB/Functions를 공유하면 부하가 타 기능·실유저 지연/오류로 전파. RTDB 동시 연결은 **인스턴스당 200,000** 한도라 부하가 정상 사용자 연결을 잠식 가능 | Firebase RTDB limits | **실서비스 프로젝트 직접 타격 금지**, 테스트 전용 프로젝트 격리(Firebase 공식: 환경별 별도 프로젝트 권장) |
| ③ 보안규칙/App Check/쿼터 우회 위험 | App Check는 "billing fraud, phishing, app impersonation, data poisoning" 등 abuse를 차단하고 미검증 클라이언트 요청을 거부. 부하 클라이언트가 attestation을 못 붙이면 차단되며, 반대로 테스트를 위해 enforcement를 끄면 abuse 표면이 노출 | Firebase App Check | App Check 조정은 **테스트 프로젝트에서만**, 실서비스 enforcement·보안규칙은 불변 |
| ④ 관리형 쿼터 = 실질 blast radius | GCP/Firebase 쿼터(rate/allocation)는 **프로젝트 단위**로 적용되고 "한 프로젝트의 소비가 다른 프로젝트 쿼터에 영향 없음". 초과 시 시스템이 차단·실패시킴 | GCP Cloud Quotas overview | 테스트 프로젝트 쿼터를 낮게 고정 → 물리적 상한(5.2 "대상 측 쿼터") |

### 9.2 핵심 권고 — 대상 환경 격리(멀티클라우드 일반화)

- **테스트 전용 프로젝트/계정/구독 격리**: Firebase 공식은 "환경별 **별도 프로젝트**" 사용을 권장하고 "최소 1개 pre-production 환경을 prod 데이터/자원과 격리"하라고 명시. 동일 원칙을 AWS(계정 분리)·Azure(구독 분리)·GCP(프로젝트 분리)로 일반화한다.
- **실서비스 프로젝트 직접 타격 금지**(본 설계 기본값과 일치, 3.3).
- **예산/쿼터 cap을 1차 상한으로**: budget alert는 cap이 아니므로(공식), 종량제 폭증은 (a) 테스트 프로젝트 격리, (b) 프로젝트 쿼터를 낮게 고정(실질 blast radius), (c) Pub/Sub→billing 자동 비활성 백스톱으로 막는다. 단 billing 비활성은 "shutting down all resources. Resources might be irretrievably deleted"인 파괴적 동작이므로 **테스트 전용 프로젝트에만** 적용한다.

| 대상 유형 | 격리 단위 | 비용 cap 수단 | blast radius 한계 |
| --- | --- | --- | --- |
| Firebase(BaaS) | 테스트 전용 Firebase 프로젝트 | Cloud Billing budget alert(+Pub/Sub 자동 비활성), App Check enforcement | 프로젝트 쿼터(RTDB 동시연결 200k/인스턴스, Firestore ops 한도) |
| GCP(IaaS/관리형) | 테스트 전용 프로젝트 | Cloud Billing budget(+programmatic disable) | 프로젝트 rate/allocation 쿼터 |
| AWS | 테스트 전용 계정 | Budgets + Budget Actions | service quota / SCP 권한 경계 |
| Azure | 테스트 전용 구독 | Cost Management budget + action group | Azure Policy / 쿼터 |

> 일반 원칙: 부하 **대상**이 관리형 서비스일 때 **대상 측 쿼터가 사실상 blast radius 한계**가 된다(공식: 쿼터 초과 시 차단·실패, 프로젝트 간 쿼터 무영향). 따라서 테스트 프로젝트의 쿼터를 의도적으로 낮게 고정하는 것이 가장 신뢰성 높은 물리적 상한이며, 비용 budget(후행·미차단)보다 우선하는 1차 안전장치다.

---

## 10. 비전문가 + 템플릿 매칭의 안전 함의 · 저신뢰 매칭 가드레일

사용자는 부하/인프라 비전문가이며 "자연어 → 가장 유사한 사전 검증 템플릿(시나리오) 실행" 방식을 쓴다. 이 방식의 안전 이점과, 그로 인해 새로 생기는 **저신뢰 매칭** 위험에 대한 가드레일을 정리한다.

### 10.1 검증된 템플릿이 자유 입력보다 안전한 이유

- 비전문가는 위험(고RPS=실유저 영향·비용, DoS 오인, 잘못된 대상 등)을 **모를 수 있다**. 위험 인지를 사용자에게 의존하는 것은 안전하지 않다.
- 자유 텍스트 입력은 명령 인젝션·과대 부하·잘못된 대상 표면을 키운다(7장, 3.3 "자유 텍스트 shell command 금지"와 동일 논리).
- 따라서 "자연어 → 사전 검증 템플릿"은 **안전 한계가 미리 검증된 경로만 노출**하므로 비전문가에게 더 안전한 기본값이다. OWASP LLM01 #2(출력 형식 정의·검증)·#4(최소권한)와 정합한다.

### 10.2 잔존 위험 — 저신뢰(엉뚱한 템플릿) 매칭

자연어→템플릿 매칭은 의미 유사도 기반이라 **엉뚱한 템플릿 선택**(예: 가벼운 smoke 의도가 고RPS soak로 매칭) 위험이 있다. 비전문가는 오선택을 알아채기 어렵다 → OWASP LLM06(불확실·고위험 작업 human approval) 원칙에 따라 가드레일이 필요하다.

### 10.3 가드레일 (3중 + 대상 allowlist)

| 가드레일 | 규칙 | 근거 |
| --- | --- | --- |
| ① 유사도 임계 게이트 | 매칭 신뢰도(0~1)가 **임계 미만이면 자동 실행 거부 + 사람 확인**(의도 재확인 또는 후보 템플릿 제시 후 명시적 선택) | OWASP LLM06 고위험·불확실 작업 human approval; 기존 audit `confidence` 필드 활용 |
| ② 템플릿 내장 안전 한계 | 각 템플릿이 **max RPS/concurrency/duration/cost + 대상 allowlist(환경·도메인)** 를 자체 내장하고, 매칭 결과는 이 한계를 **상속**(자유 생성 금지) | 5.2 "정책 템플릿으로 상한 고정", 7.2 #2 typed schema |
| ③ 비전문가 우회 차단 | 비전문가는 템플릿 안전 상한을 **상향/우회 불가**. 상한 상향은 3.2 "RPS/duration ceiling 상향 = 승인 필수"로만 가능 | 3.2/3.3 승인 게이트 |
| ④ 대상 allowlist 강제 | 템플릿은 실서비스 프로젝트·외부 파트너(PG/FCM) 경로를 대상에서 **배제**, 테스트 전용 환경만 허용 | 9장 격리 원칙, 8장 제3자 제외 |

> 매칭 신뢰도와 선택된 템플릿 ID·버전은 audit에 기록(11장)하여 오선택을 사후 추적·재현 가능하게 한다. 임계값은 도메인 데이터로 보정해야 하며 초기값은 보수적으로(거부 우선) 설정한다.

---

## 11. audit log 필수 필드 체크리스트

모든 실행은 재현·감사 가능해야 한다. 아래 필드를 `RunRecord`/`AuditEvent`에 누적 기록한다.

- [ ] **raw_prompt**: 사용자 원문
- [ ] **normalized_intent**: 구조화된 `UserIntent`
- [ ] **source_metadata**: 검색/페치한 출처 URL·fetched_at·freshness(외부 콘텐츠는 데이터로 기록)
- [ ] **selected_tool + version + api_version**: 선택 도구/버전
- [ ] **terraform_plan_hash + approval_id**: plan artifact 해시와 승인 ID 결속(tamper-proof)
- [ ] **toolaction.args + idempotency_key + timeout_sec**: 실행 인자·중복 방지 키·제한 시간
- [ ] **run start/stop/abort_reason**: 실행 시작·종료·중단 사유(kill switch 트리거 포함)
- [ ] **metrics/logs/traces/artifact refs**: 관측·산출물 참조
- [ ] **report_version + model_version + prompt_version**: 보고서·모델·프롬프트 버전
- [ ] **confidence**: 해석 신뢰도(0~1)
- [ ] **approver identity + self-review 여부**: 누가 승인했는지, self-approval 차단 적용 여부
- [ ] **environment + blast_radius**: 대상 환경과 영향 범위
- [ ] **cost/token actuals vs ceiling**: 실제 비용/토큰 대비 상한
- [ ] **target_isolation_check**: 대상이 테스트 전용 프로젝트/계정/구독인지(BaaS=실서비스 프로젝트 아님) 확인 결과 + 대상 프로젝트/조직 ID(9장)
- [ ] **template_id + version + safety_limits_verified**: 선택된 템플릿 ID·버전과 내장 안전 한계(max RPS/duration/cost·대상 allowlist) 검증 통과 여부(10장)
- [ ] **match_confidence + threshold + gate_outcome**: 자연어→템플릿 매칭 신뢰도, 적용 임계값, 게이트 결과(자동 실행/거부+사람확인)(10장)
- [ ] **third_party_paths_excluded**: PG/FCM 등 외부 파트너 경로 제외 확인(8장)

근거: OWASP LLM06("AI actions are logged, monitored, and auditable"), GitHub environment 승인 추적, Terraform saved plan 승인 결속 필요성. BaaS 대상 격리·템플릿 안전한계·매칭 신뢰도 기록은 비전문가/종량제 환경의 오선택·비용 폭증을 사후 추적·재현하기 위함.

---

## 12. policy-as-code / secrets 거버넌스 대체제 비교

### 12.1 Policy-as-code

| 수단 | 분석 대상 | 강점 | 약점 | 이 설계에서의 위치(한 줄 평가) |
| --- | --- | --- | --- | --- |
| Sentinel (HCP Terraform) | plan(의도된 상태) | HCP 네이티브, cost estimate(`tfrun`) 접근, advisory/soft/hard 등급 | HCP 종속, Sentinel 언어 학습 | **HCP 표준 시 1순위**: 특히 비용 정책에 유일하게 cost 데이터 접근 |
| OPA/Conftest | plan(json/binary), Rego | 오픈소스·멀티스택(K8s/API), `opa eval`로 plan 직접 평가 | Rego 학습, 사전 정책 부재, cost estimate 접근 불가(HCP) | **벤더 중립 1순위**: 커스텀 비즈니스 룰, 비-HCP 환경 |
| Checkov | HCL static(정의된 상태) | 사전 정책 풍부, 빠른 보안 baseline, 멀티 IaC | plan 미반영(런타임 의도 일부 누락) | **shift-left baseline**: PR 단계 빠른 차단 |
| AWS SCP / AWS Config | 계정 권한 경계 / 리소스 상태 | 조직 차원 preventive(Deny 우선), 실행 시점 강제 | preventive만(기존/드리프트 미처리), denial 가시성 낮음 | **물리적 가드레일**: runner 계정 자체를 좁히는 백스톱 |
| Azure Policy | 리소스/관리 작업 | Deny effect, 대규모 subscription 거버넌스 | 대규모 시 latency(DeployIfNotExists) | **Azure 백스톱**: 관리 작업 제한, IAM 보완 |

> 권장 조합: PR 단계 **Checkov**(baseline) → plan 단계 **OPA 또는 Sentinel**(hard-mandatory, 비용 정책은 Sentinel) → 런타임 **SCP/Azure Policy**(계정 경계 백스톱). 단일 수단 의존 금지(defense-in-depth).

### 12.2 Secrets 거버넌스 (6.2와 연계)

| 수단 | 이 설계에서의 위치(한 줄 평가) |
| --- | --- |
| GitHub OIDC | **1차 경로**: CI→cloud 장기 secret 제거 |
| HashiCorp Vault / HCP Vault | **고급**: 진정한 dynamic short-lived(DB/cloud), 멀티클라우드 |
| AWS Secrets Manager + STS | **AWS 전용**: STS 단명 토큰 결합 시 충분 |
| Azure Key Vault + Managed Identity | **Azure 전용**: 키리스 경로 |
| GCP Secret Manager + Workload Identity | **GCP 보조**: GCP 대상 시에만 |

---

## 13. 남은 불확실성 + 구현 전 재검증 항목

| # | 항목 | 불확실성 | 구현 전 재검증 |
| --- | --- | --- | --- |
| 1 | AWS 침투/스트레스 정책 게시일 미표시 | 정책 변동 가능(금지 목록·임계·서비스 목록) | aws.amazon.com/security/penetration-testing 및 ec2/testing 재확인, Simulated Events 리드타임 |
| 2 | Azure ROE 세부·승인 파트너 목록 | 펜테스트 ROE·승인 DDoS 파트너 변경 가능 | Microsoft ROE 페이지 + Azure pen-testing 문서 재확인 |
| 3 | AWS Budgets 액션 자동화 범위 | EC2/RDS 외 대상, cross-account 제약 변동 | budgets-controls/action-configure 재확인, 실시간성 한계 검증 |
| 4 | Azure budget 알림 지연·action group scope | 최대 24h 지연, subscription/RG scope 한정 | 실시간 비용 워처를 Orchestrator 측에 별도 구현 검토 |
| 5 | HCP cost estimation의 OPA 접근 제약 | OPA evaluation은 cost 데이터 접근 불가(현행) | 비용 정책은 Sentinel로 설계, policy-sets 문서 재확인 |
| 6 | OWASP LLM Top 10 차기 개정 | 2025판 기준, 차기판에서 카테고리 변동 가능 | 구현 시점 최신판 확인 |
| 7 | NIST AI 600-1 문서 버전 | 최종본 URL로 교체 완료. 단 NIST AI RMF companion/profile 문서는 후속판 가능 | 구현 시점에 `NIST.AI.600-1` DOI/게시 상태 재확인 |
| 8 | Anthropic/LLM spend·rate limit API | workspace spend limit·tier 자동 승급 동작 변동 가능 | platform.claude.com/openai 한도 페이지 재확인, 명시적 ceiling 고정 |
| 9 | GitHub required reviewers 플랜 제약 | Free/Pro/Team은 public repo만 일부 보호 규칙 | 조직 플랜·repo 가시성에 맞춰 environment 보호 규칙 검증 |
| 10 | Terraform ephemeral/write-only provider 지원 | provider별 지원 상이 | 사용 provider의 ephemeral/write-only 지원 확인 |
| 11 | 비용·쿼터·region 가용성 | 가격/quota 변동 | 공식 pricing/quotas 페이지 별도 확인 |
| 12 | Firebase/GCP billing cap 동작 | budget alert는 미차단, billing 비활성은 파괴적(자원 종료/삭제) | avoid-surprise-bills / disable-billing-with-notifications 재확인, 테스트 프로젝트 한정 적용 검증 |
| 13 | Firebase/GCP 프로젝트 쿼터 수치 | RTDB 200k·Firestore ops·GCP rate quota 값/조정 가능성 변동 | RTDB limits·Firestore quotas·Cloud Quotas 재확인, 테스트 프로젝트 쿼터 하향 고정 검증 |
| 14 | App Check enforcement 범위 | 커버 API·attestation provider별 한도 변동 | App Check 문서 재확인, 실서비스 enforcement 불변·테스트 프로젝트만 조정 검증 |
| 15 | GCP AUP/펜테스트·DoS 정책 | AUP 세부·"only affect your projects" 해석 변동 | Cloud Security FAQ / AUP 재확인, 같은 조직 내 타 타이틀 영향 회피 확인 |
| 16 | 템플릿 매칭 신뢰도 임계값 | 임계값 보정 부재 시 오선택(저신뢰 자동 실행) 위험 | 도메인 데이터로 임계 보정, 초기값 보수적(거부 우선), 매칭 신뢰도·템플릿 ID audit 로깅 검증 |
| 17 | 템플릿 내장 안전 한계 정합성 | 템플릿별 max RPS/duration/cost·대상 allowlist 누락/우회 가능성 | 템플릿 안전 한계 필드 필수화, 비전문가 우회 불가(승인 게이트 경유) 검증 |

---

### 부록: 핵심 출처 링크

- GitHub Actions environments(승인 전 secret 차단): https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments
- GitHub Actions OIDC: https://docs.github.com/en/actions/concepts/security/openid-connect
- AWS Budgets 액션: https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-controls.html
- AWS 침투테스트 정책: https://aws.amazon.com/security/penetration-testing/
- AWS EC2 Testing(네트워크 스트레스): https://aws.amazon.com/ec2/testing/
- Microsoft 보안테스트 ROE: https://www.microsoft.com/en-us/msrc/pentest-rules-of-engagement
- Azure 펜테스트: https://learn.microsoft.com/en-us/azure/security/fundamentals/pen-testing
- Azure Cost Management 예산: https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/tutorial-acm-create-budgets
- HCP Terraform cost estimation: https://developer.hashicorp.com/terraform/cloud-docs/workspaces/cost-estimation
- HCP Terraform 정책 결과: https://developer.hashicorp.com/terraform/cloud-docs/workspaces/policy-enforcement/view-results
- OWASP LLM Top 10(2025): https://owasp.org/www-project-top-10-for-large-language-model-applications/
- OWASP LLM01 Prompt Injection: https://genai.owasp.org/llmrisk/llm01-prompt-injection/
- OWASP LLM06 Excessive Agency: https://genai.owasp.org/llmrisk/llm062025-excessive-agency/
- NIST AI 600-1 GenAI Profile: https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf
- Anthropic 안전 에이전트 프레임워크: https://www.anthropic.com/news/our-framework-for-developing-safe-and-trustworthy-agents
- OPA + Terraform: https://www.openpolicyagent.org/docs/terraform
- AWS SCP: https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html
- Anthropic rate/spend limits: https://platform.claude.com/docs/en/api/rate-limits
- OpenAI rate limits: https://platform.openai.com/docs/guides/rate-limits
- OWASP WSTG Authorization: https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/05-Authorization_Testing/README
- Firebase 청구 경고(예상치 못한 청구 방지): https://firebase.google.com/docs/projects/billing/avoid-surprise-bills
- Firebase 요금제(Blaze 종량제): https://firebase.google.com/docs/projects/billing/firebase-pricing-plans
- Firebase 환경 분리(별도 프로젝트 권장): https://firebase.google.com/docs/projects/dev-workflows/overview-environments
- Firebase App Check: https://firebase.google.com/products/app-check
- Firebase RTDB 한도(동시연결 200k): https://firebase.google.com/docs/database/usage/limits
- Firestore 사용량/한도: https://firebase.google.com/docs/firestore/quotas
- GCP Cloud Quotas 개요: https://docs.cloud.google.com/docs/quotas/overview
- GCP 예산/예산알림: https://docs.cloud.google.com/billing/docs/how-to/budgets
- GCP billing 자동 비활성(Pub/Sub): https://docs.cloud.google.com/billing/docs/how-to/disable-billing-with-notifications
- GCP Cloud Security FAQ(펜테스트/AUP): https://support.google.com/cloud/answer/6262505
