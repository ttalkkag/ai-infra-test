# 05 · 실행기(러너)·Template Composer·번들 목 서버

> 평면 분리·실행 권한 단일화는 [[00-overview]] §6·[[01-architecture]] §1의 불변식을 따른다. 본 문서는 **실행 평면**(`runners/*`)의 인터페이스·구성 스케치와, 검증 대상이 될 **번들 로컬 목 서버**(`mock-target/`)를 설계한다. 실제 코드가 아니라 **골격/슬롯 자리표시자/의사코드**까지만 다룬다.
>
> 범위 경계: 이번 로컬 범위는 k6/Artillery 바이너리를 로컬 또는 Docker `runner-tools`에서 돌려 **번들 목 서버**를 친다 ([[00-overview]] D6·D8). 클라우드 분산 러너(Grafana Cloud k6 / AWS DLT / Azure LT)와 cross-cloud 실행은 **동일 `RunnerAdapter` 인터페이스 뒤의 후속 확장**으로 §8에만 설계한다 ([[00-overview]] D11·D12, [[12-execution-topology-matrix]]).
>
> 관련 문서: 안전·승인·kill switch는 [[04-safety-approval-audit]], 슬롯/매칭은 [[03-engine-template-matching]], 스키마는 [[02-data-model]], 관측·해석은 [[06-reporter-observability]], 테스트 전략은 [[09-testing-strategy]].

---

## 0. 책임 경계 한눈

```
제어 평면(실행 권한 0)                    │  실행 평면(유일 실행 권한)
─────────────────────────────────────────┼────────────────────────────────────
engine/* → ToolAction(draft) 생성          │  runners/controller.py  (Execution Controller)
safety/* → clamp·judge·plan_hash           │    └ runners/base.py     (RunnerAdapter 인터페이스)
                                           │        ├ runners/k6_adapter.py
        승인 게이트 (approval_id+plan_hash) │        ├ runners/artillery_adapter.py
        ───────────────▶                   │        └ runners/compose.py (Template Composer)
                                           │  mock-target/  (검증 대상 SUT, 별도 프로세스)
```

- **`engine`은 `runners`를 import하지 않는다** ([[01-architecture]] §4 의존 방향). 제어 평면은 `ToolAction`만 만들고, 실행은 항상 `controller.py`를 통과한다.
- `compose.py`는 **순수 함수**(슬롯값 → 산출물 파일)라서 부작용이 없다 → 골든 파일 테스트 대상(§9).
- `base.py`/어댑터는 도구별 차이를 흡수하고, `controller.py`는 도구를 모른 채 `RunnerAdapter`만 호출한다(도구 비종속).

---

## 1. RunnerAdapter 인터페이스 (`runners/base.py`)

도구 비종속 계약. **하나의 ToolAction 실행 = validate → compose → dry_run → (승인) → submit → canary → full_run → collect**, 비정상 시 언제나 `abort`. 모든 메서드는 부작용 없는 단계와 부작용 있는 단계를 명확히 분리한다.

```python
# runners/base.py  (의사코드 — 시그니처/계약만)

class RunArtifacts(Protocol):
    summary_json: Path        # 도구 summary export (JSON)
    raw_output: Path          # stdout/stderr/raw metrics
    generator_health: dict    # dropped_iterations, gen CPU%, net 등 (§4·§8 함정1)
    exit_code: int

class RunnerAdapter(Protocol):
    tool: Literal["k6", "artillery", ...]   # ToolAction.tool enum과 1:1

    def validate(self, plan: TestPlanSpec, slots: dict) -> ValidationResult:
        """정적 검증만. 슬롯 타입·필수값·safe_limits 적합·도구 문법 가능성.
        부작용 0. 실패 시 사유 리스트 반환(실행 진입 차단)."""

    def compose(self, plan: TestPlanSpec, slots: dict) -> ComposedArtifact:
        """슬롯값 → 도구별 산출물(스크립트/구성/페이로드). 순수 함수.
        compose.py 위임. '타이틀-가변 파라미터만 주입'(§2)."""

    def dry_run(self, composed: ComposedArtifact) -> DryRunResult:
        """도구의 정적/무부하 검증 경로로 산출물을 '실행 없이' 점검.
        k6=정적 파싱/--no-... , artillery=스키마 검증. 네트워크 트래픽 0."""

    def submit(self, action: ToolAction, composed: ComposedArtifact) -> RunHandle:
        """실제 프로세스 기동 준비(격리 작업 디렉토리·env·핸들 등록).
        controller만 호출. approval_id·plan_hash 검증은 controller 책임."""

    def canary(self, handle: RunHandle) -> CanaryResult:
        """축소 부하(짧은 duration·낮은 arrival-rate)로 1차 실행.
        success_criteria 부분집합 + 생성기 건강 신호로 pass/fail."""

    def full_run(self, handle: RunHandle) -> RunHandle:
        """canary pass 시에만 호출. 본 부하. kill switch 폴링 하에 실행."""

    def abort(self, handle: RunHandle, reason: str) -> None:
        """프로세스 그룹 종료 + 부분 산출물 flush + abort_reason 기록.
        kill switch/timeout/ceiling 초과 시 controller가 호출."""

    def collect(self, handle: RunHandle) -> RunArtifacts:
        """산출물 수집(read-only). summary_json 등 경로 반환.
        해석은 reporter가 담당 — 여기선 파일·신호만 모은다([[06-reporter-observability]])."""
```

계약 불변식:

| 규칙 | 근거 |
|---|---|
| `validate`/`compose`/`dry_run`은 **부작용 0**(네트워크/프로세스 없음) | 골든·정적 테스트 가능(§9) |
| `submit`/`canary`/`full_run`/`abort`만 **부작용 보유**, controller만 호출 | 실행 권한 단일화 ([[04-safety-approval-audit]]) |
| `canary` 미통과 시 `full_run` 호출 금지 (어댑터가 강제 X, controller가 강제) | [[00-overview]] §6 |
| `collect`는 read-only — 해석/판정 안 함 | 평면 분리, reporter가 해석 |

---

## 2. Template Composer (`runners/compose.py`)

슬롯값 → 도구별 산출물. **타이틀-가변 파라미터만 주입**한다(스크립트 본문 골격은 큐레이션된 템플릿에 고정, 자유 생성 아님 — [[03-engine-template-matching]]·research §7.5 "사전 검증 템플릿 + 슬롯 채움").

### 2.1 주입 원칙

```
ScenarioTemplate(골격 = 큐레이터가 검수한 고정 본문)
        +  param_slots_filled (clamp 통과한 가변값만)
        →  ComposedArtifact (실행 산출물)
```

- 주입 대상 = host/endpoint, 환경, 인증값 참조, 목표 RPS/CCU/arrival-rate, VU, duration, ramp stage, think time 분포, 읽기:쓰기 비율, SLO threshold, 데이터 파일 참조.
- **비주입 대상** = 제어 흐름·요청 시퀀스·로깅·threshold 구조. 이건 골격에 고정되어 비전문가가 우회 불가 ([[00-overview]] §6).
- 슬롯값은 `safety/clamp.py`를 **이미 통과한 값만** 들어온다(compose는 clamp를 다시 하지 않되, `validate`에서 safe_limits 재확인 — 방어적 이중화).

### 2.2 도구별 매핑 표

| 슬롯(`param_slots`) | k6 산출물 | Artillery 산출물 |
|---|---|---|
| host/endpoint | `__ENV.TARGET_BASE_URL` | `config.target` / `config.environments.<env>.target` |
| 환경(allowed_envs) | `__ENV.ENV` + 정적 가드 | `config.environments.<env>` |
| 목표 arrival-rate/RPS | `scenarios.*.rate` (arrival-rate executor) | `config.phases[].arrivalRate` |
| VU/preallocated | `preAllocatedVUs`/`maxVUs` | (암묵: arrivalRate×duration) |
| duration·ramp stage | `scenarios.*.stages[]` / `duration` | `config.phases[].duration`/`rampTo` |
| think time 분포 | `sleep()` 슬롯 (분포 파라미터) | `think` 스텝 |
| 읽기:쓰기 비율 | scenario 가중치/분기 슬롯 | `scenario.weight` |
| SLO(p95/p99/error) | `thresholds{}` + `abortOnFail` | `config.ensure{}` (임계) |
| 데이터(계정/시드) | `SharedArray`(CSV/JSON 참조) | `config.payload`(CSV) |
| 출력 | `--out json` / `handleSummary`→JSON | `--output report.json` |

### 2.3 ComposedArtifact 형태 (의사코드)

```python
@dataclass(frozen=True)
class ComposedArtifact:
    tool: str
    entry_file: Path          # k6: scenario.js / artillery: scenario.yml
    aux_files: list[Path]     # CSV payload, env json, lib 골격
    env: dict[str, str]       # k6 __ENV 주입값 (자유 텍스트 아님: 화이트리스트 키만)
    cli_args: list[str]       # controller가 ToolAction으로 감쌀 인자 (자유 shell 금지)
    content_hash: str         # artifact 지문. ToolAction.args.expected_content_hash와 비교
```

> `content_hash`는 `plan_hash`와 같은 값이 아니다. 계획 봉인 시 `ToolAction.args.expected_content_hash`로 기록되고, `plan_hash`는 그 args 값을 포함해 계산된다([[04-safety-approval-audit]], [[02-data-model]] §4). 실행 직전에는 compose 결과의 `content_hash`가 기대값과 같은지 확인한 뒤, 현재 계획 번들의 `plan_hash`를 재계산해 승인값과 비교한다. 같은 슬롯 입력 → 같은 `content_hash`(결정성)가 골든 파일 테스트의 전제(§9).

---

## 3. k6 어댑터 (`runners/k6_adapter.py`)

### 3.1 scenarios / executors (open model 우선)

부하 모델은 `scenarios` 안의 `executor`로 닫는다. **stress/spike/breakpoint/load는 open model = arrival-rate executor**를 기본으로 한다(coordinated omission 회피 — `docs/research/04-test-methodology-slo.md` §8 함정2). CCU 중심 게임 여정은 closed/VU + think time 페이싱(docs/research/04 §11).

| 테스트 유형 | 권장 executor | 모델 |
|---|---|---|
| Load | `constant-arrival-rate` | open |
| Stress / Breakpoint | `ramping-arrival-rate` (+ `abortOnFail`) | open |
| Spike | `ramping-arrival-rate` (가파른 stage) | open |
| Soak | `constant-arrival-rate` (장시간 hold) | open |
| CCU 세션 여정 | `ramping-vus` + 단계 사이 `sleep()` think time | closed |

골격(슬롯 자리표시자만 — **완성 스크립트 아님**):

```javascript
// k6 scenario 골격 (compose가 슬롯만 채움; 본문 구조는 큐레이션 고정)
export const options = {
  scenarios: {
    main: {
      executor: "{{EXECUTOR}}",            // arrival-rate 계열 (open)
      rate: {{TARGET_RATE}}, timeUnit: "1s",
      preAllocatedVUs: {{PRE_VUS}}, maxVUs: {{MAX_VUS}},
      stages: [ /* {{RAMP_STAGES}} */ ],
    },
  },
  thresholds: {
    http_req_duration: ["p(95)<{{P95_MS}}", "p(99)<{{P99_MS}}"],
    http_req_failed:   [{ threshold: "rate<{{ERR_RATE}}", abortOnFail: true,
                          delayAbortEval: "{{ABORT_GRACE}}" }],
    dropped_iterations: ["count<1"],       // 생성기 병목 가드(§3.4)
  },
};
export default function () { /* {{JOURNEY_BODY}} — 골격 고정 */ }
export function handleSummary(data) {
  return { "{{SUMMARY_PATH}}": JSON.stringify(data) };  // JSON summary export
}
```

### 3.2 thresholds = SLO 코드화 + abortOnFail

- `thresholds`는 pass/fail 기준이며 위반 시 k6가 **non-zero exit**으로 종료 → controller가 fail 게이트로 인식(docs/research/04 §5, 부록A). threshold 실패 종료코드는 **99**(`ThresholdsHaveFailed`, k6 소스 `errext/exitcodes/codes.go` 정의; 사용자 문서엔 "non-zero"로만 표기) → controller는 **0 여부로 게이트**(비정상 종료=99=threshold 실패)한다.
- 장기 위반 누적은 `abortOnFail: true`로 즉시 중단(+`delayAbortEval`로 초기 워밍업 구간 평가 유예). 이는 어댑터 자체의 1차 abort이고, 그 위에 controller의 kill switch(§5)가 이중으로 존재한다.
- `default_slo` 슬롯이 비어도 템플릿 내장 기본 threshold가 주입된다 → 비전문가도 percentile/error budget 판정이 자동 적용(docs/research/04 §10.4).

### 3.3 __ENV 슬롯 주입

- 가변 파라미터는 `__ENV.<KEY>`로 주입하고, controller가 `-e KEY=VALUE`(ToolAction.args의 typed env)로 전달한다. "여러 스크립트 대신 환경변수로 스크립트 일부만 가변화"(k6 공식, docs/research/01 출처).
- **자유 텍스트 금지**: env 키는 화이트리스트(`TARGET_BASE_URL`/`ENV`/`TARGET_RATE`/...)만 허용. 임의 키·셸 메타문자는 `validate`에서 거부.

### 3.4 생성기 병목 신호 (1급 관심사)

부하 생성기가 먼저 포화하면 결과가 통째로 거짓이 된다(docs/research/04 §8 함정1). `collect`가 모으는 `generator_health`:

| 신호 | 출처 | 해석 |
|---|---|---|
| `dropped_iterations` | k6 내장 메트릭 | >0 → 목표 도착률 미달성 = 생성기/스케줄 한계 의심 |
| 생성기 CPU% | 로컬 프로세스 샘플(§7) | >80% → 응답시간 inflation 위험, 결과 무효화 후보 |
| 생성기 net 처리량 | 로컬 NIC 샘플 | 1Gbit/s 근접 → 대역폭 병목 |
| `vus` vs `vus_max` | k6 내장 | preAllocated 소진 = arrival-rate 못 따라감 |

> 판정 규칙(어댑터→reporter 전달): `dropped_iterations>0` 또는 `gen_cpu>80%`이면 결과에 `generator_saturated=true` 플래그를 달아, **SUT 한계로 오판하지 않게** 해석을 보류시킨다([[06-reporter-observability]]).

### 3.5 dry-run / 정적 검증 경로

- `dry_run`: 산출물 정적 파싱 + 슬롯 바인딩 점검(네트워크 0). k6 자체 정적 검증은 한정적이므로, **목 서버 대상 canary(짧은 1~5초)**를 사실상의 무해 dry-run으로 함께 둔다.
- MCP `validate_script`/`run_script`(VU≤50·≤5분 제한, preview)는 **선택적 보조**일 뿐, 실행 권한 경로가 아니다 — 신뢰 경로는 어디까지나 controller→어댑터 (docs/research/01 정정 2).

---

## 4. Artillery 어댑터 (`runners/artillery_adapter.py`)

### 4.1 YAML config / scenarios / phases

YAML "값만 채우는" 템플릿 친화 도구. 골격(슬롯 자리표시자만):

```yaml
# artillery scenario 골격 (compose가 슬롯만 채움)
config:
  target: "{{TARGET_BASE_URL}}"
  phases:
    - duration: {{RAMP_SECONDS}}, arrivalRate: {{START_RATE}}, rampTo: {{PEAK_RATE}}
    - duration: {{HOLD_SECONDS}}, arrivalRate: {{PEAK_RATE}}
  payload: { path: "{{ACCOUNTS_CSV}}", fields: [ {{CSV_FIELDS}} ] }
  environments:
    local:   { target: "http://127.0.0.1:{{MOCK_PORT}}" }
    dev:     { target: "{{DEV_TARGET}}" }
  ensure:                                  # 임계 = pass/fail 게이트
    - http.response_time.p95: {{P95_MS}}
    - http.response_time.p99: {{P99_MS}}
    - "http.codes.500": 0                  # 예: 5xx 허용치
  plugins:
    expect: {}                             # 응답 검증 플러그인(선택)
scenarios:
  - weight: {{READ_RATIO}}
    flow: [ /* {{READ_JOURNEY}} — 골격 고정 */ ]
  - weight: {{WRITE_RATIO}}
    flow: [ /* {{WRITE_JOURNEY}} — 골격 고정, think 포함 */ ]
```

### 4.2 매핑·검증

| 요소 | 역할 |
|---|---|
| `config.phases[]` (`arrivalRate`/`rampTo`/`duration`) | open-model 부하 프로파일(ramp/hold) |
| `config.payload` (CSV) | 계정/시드 데이터 — 카디널리티 확보(docs/research/04 §8 함정7) |
| `config.environments.<env>` | 환경별 target 분리, allowed_envs 가드 |
| `config.ensure` | SLO 임계 — 위반 시 non-zero exit(게이트) |
| `plugins.expect` / publish-metrics(OTel) | 응답 검증·관측 export |

- **정적 스키마 검증**(`validate`): YAML 파싱 + Artillery 스키마(필수 키·타입·phases 구조·environments 존재)·slot 바인딩 점검. 네트워크 0.
- **WebSocket/Socket.IO**: `engine: ws`/`engine: socketio` 내장 → `firebase_realtime_sync_probe` 류 실시간 템플릿의 Artillery 경로(docs/research/01 §2.1). k6 대비 "실제 클라이언트 구동" 편의가 강점.
- 분산(Fargate/ACI/Lambda)은 CLI가 리소스 생성·삭제까지 하므로 **로컬 범위 제외**, §8 클라우드 확장에서만 다룬다.

---

## 5. Execution Controller (`runners/controller.py`)

### 5.1 유일 실행 권한 + 입력 계약

```python
# runners/controller.py (의사코드)

def execute(action: ToolAction, plan: TestPlanSpec) -> RunRecord:
    assert isinstance(action, ToolAction)        # typed only — 자유 shell 금지
    require_approval(action)                      # §5.2
    adapter = registry.get(action.tool)           # k6 | artillery (도구 비종속 디스패치)

    vr = adapter.validate(plan, action.args["slots"])
    if not vr.ok: return _reject(action, vr)      # 정적 차단

    composed = adapter.compose(plan, action.args["slots"])
    _verify_artifact_hash(action, composed)       # composed.content_hash == expected_content_hash
    _verify_plan_hash(action)                     # current plan bundle hash == approved plan_hash

    if action.dry_run:                            # 기본값 true (§7)
        dr = adapter.dry_run(composed)
        return _record(action, status="completed", artifacts=dr)

    with _idempotency_guard(action.idempotency_key), _deadline(action.timeout_sec):
        handle = adapter.submit(action, composed)
        cana = adapter.canary(handle)
        if not cana.passed:                       # canary gate
            adapter.abort(handle, "canary-failed")
            return _record(action, status="failed", abort_reason=cana.reason)
        _killswitch.attach(handle)                # §5.3 소유
        handle = adapter.full_run(handle)         # 승격
        arts = adapter.collect(handle)            # read-only
        return _record(action, status=_status_from(arts.exit_code), artifacts=arts)
```

### 5.2 승인 검증 (approval_id + plan_hash)

| 검사 | 규칙 |
|---|---|
| 승인 존재 | `action.requires_approval=true`면 `approval_id` 필수, `ApprovalRequest.status="approved"`만 통과 |
| artifact hash 결합 | `compose` 산출 `content_hash`가 `ToolAction.args.expected_content_hash`와 같아야 실행 — 불일치=artifact 변조로 거부 |
| plan_hash 결합 | 현재 계획 번들의 재계산 `plan_hash`가 승인 시점 `ApprovalRequest.plan_hash`와 같아야 실행 — 불일치=계획 변조로 거부 |
| 만료 | `ApprovalRequest.expires_at` 경과 시 거부 |
| 환경/blast_radius | `environment`/`blast_radius`가 승인 범위 내인지 재확인 |

> 상세 승인 모델·tamper-proof 저장·audit append-only는 [[04-safety-approval-audit]]. 본 컨트롤러는 그 계약의 **집행자**다.

### 5.3 멱등·timeout·canary 승격·kill switch

- **idempotency_key**: 같은 키 재실행 시 기존 RunRecord 반환(중복 부하 방지). RunRecord 스키마의 `run_id`와 묶임([[02-data-model]]).
- **timeout_sec**: 데드라인 초과 시 `abort(handle, "timeout")`.
- **canary→full 승격**: canary가 success_criteria 부분집합 + `generator_health` 정상일 때만 full로 승격. 미통과면 full 금지([[00-overview]] §6).
- **kill switch(컨트롤러 소유)**: 폴링 루프로 abort 조건 감시. 초과 즉시 `abort` + `RunRecord.status="aborting"`.

| kill switch 트리거(로컬 범위) | 평가 소스 |
|---|---|
| SLO 연속 위반 / 5xx·timeout 급증 | k6 threshold·Artillery ensure 실시간 신호 |
| 생성기 포화(`dropped_iterations>0`, gen CPU>80%) | `generator_health`(§3.4) |
| 비용/호출량/duration ceiling 초과 | `safe_limits` 대비 카운터 |
| 데드라인(timeout) 초과 | `_deadline` |
| 사람 중단 선언(UI kill 버튼) | [[07-web-ui-api]] WS 제어 |

> 로컬 목 서버 범위에선 "production alert"·"실사용자 영향" 트리거는 비활성(대상이 목 서버뿐). 클라우드 확장 시 활성화(§8).

---

## 6. 번들 로컬 목 서버 (`mock-target/`) — D8

### 6.1 목적과 D9 연결

실대상 없이 **결정적·재현 가능·비용 0**인 SUT를 제공한다([[00-overview]] D8). 핵심은 **러너 출력이 예측 가능**해지는 것이다: 주입한 지연·에러·동시성 한계가 알려져 있으므로, k6/Artillery 어댑터의 산출물과 reporter의 해석을 **기대값과 대조**할 수 있다 → 이것이 D9(테스트 가능성, [[09-testing-strategy]])를 가능케 한다.

구현 스택은 백엔드와 같은 **FastAPI/Starlette 단일 앱**으로 고정한다. `mock-target/`는 실제 앱 코드와 분리된 프로세스이며, Docker 실행 시에도 별도 서비스로 띄운다. WebSocket은 Starlette WebSocket을 사용하고, 테스트 프로파일은 파일(`mock_profile.yaml`) 또는 제어 엔드포인트로만 바꾼다.

```
주입 파라미터(지연분포·에러율·동시성·시드)  ──알려진 진실(ground truth)──┐
        ▼                                                              ▼
k6/Artillery 어댑터 → 목 서버 → summary_json ──reporter 해석──▶  기대값과 비교 (통합 테스트)
```

### 6.2 주입 가능한 동작 모델

| 동작 | 주입 파라미터 | 모사 방식 |
|---|---|---|
| 지연 분포 | `latency.p50/p95/p99`(ms), 분포형(lognormal 등) | 시드 기반 결정적 표본으로 응답 지연 |
| 에러율 | `error.5xx_rate`, `error.timeout_rate` | 시드 기반 결정적 주입(500 / 무응답) |
| 동시성 한계 | `concurrency.limit` | 활성 요청 카운트 ≥ limit → 초과분 큐잉/거부 |
| 큐/포화 | `queue.depth_max`, `queue.drain_rate` | 세마포어 + 토큰버킷으로 M/M/1 유사 큐를 근사: 적체 시 지연 증가→포화 시 503 |
| 포화 곡선 | limit 근접 시 p99 inflation | "latency=saturation 선행 신호"(docs/research/04) 재현 |

- **결정적 시드**: `seed`(요청 키·인덱스 → 의사난수)로 동일 입력 → 동일 지연·에러 시퀀스. 재현성 = 테스트의 전제.
- 동작은 런타임 구성(`mock_profile.yaml` 또는 제어 엔드포인트)로 주입 — 테스트마다 프로파일 교체.
- 지연 표본은 시드 고정 RNG에서 만들고, 큐/동시성은 wall-clock 대신 요청 순서·가상 service tick을 우선 사용한다. 실제 시간 기반 sleep은 최소화해 CI flaky를 줄인다.

### 6.3 엔드포인트 설계 (골격)

| 엔드포인트 | 프로토콜 | 용도 |
|---|---|---|
| `GET /healthz` | REST | liveness(부하 무관) |
| `GET /read/:key` | REST | 읽기 여정(지연·에러 주입) |
| `POST /write` | REST | 쓰기 여정(idempotency 키 에코·동시성 한계) |
| `GET /journey/*` | REST | 로그인→진행→종료 표준 여정 단계(docs/research/04 §10) |
| `WS /realtime` | WebSocket | `firebase_realtime_sync_probe` 류 지속연결(연결 수명·ping/pong·msg echo) |
| `POST /__control/profile` | REST(제어) | 지연/에러/동시성 프로파일 주입(테스트 셋업 전용) |
| `GET /__control/stats` | REST(제어) | 서버측 활성 연결·큐 깊이 등(ground truth 대조용) |

- WebSocket 경로는 **부하 단위=연결**(docs/research/04 §11.4)을 모사: 연결 수립 지연·세션 지속·메시지 왕복(ping)으로 `ws_*` 대리 지표를 어댑터가 관측 가능하게 한다.
- `__control/*`는 로컬 셋업 전용이며 실행 평면 ToolAction 경로와 분리(러너는 부하 엔드포인트만 친다).

### 6.4 결정성이 주는 검증력

| 검증 대상 | 목 서버가 가능케 하는 것 |
|---|---|
| 어댑터 compose/실행 | 주입 p95=200ms면 k6 summary의 p95가 ~200ms 근방 → 어댑터 경로 정상 확인 |
| 생성기 병목 감지 | 동시성 limit↓ + 높은 rate 주입 → `dropped_iterations>0` 유도 → §3.4 신호 단위 테스트 |
| reporter 해석 | 주입 error_rate=2% → interpreter가 error budget 판정을 옳게 내는지 대조 |
| kill switch | 에러율 급증 프로파일 주입 → controller abort 발화 시점 검증 |

---

## 7. 로컬 실행 플로우

### 7.1 가정·전제

- 로컬 실행은 두 모드를 지원한다.
  - **host-native 모드(기본 개발 루프)**: k6/Artillery 바이너리가 로컬에 설치되어 있다고 가정한다. 설치 자동화는 비범위지만, 버전은 `tools/versions.json`에 고정하고 `k6 version`/`artillery -V`로 검증한다.
  - **Docker 모드(재현성/CI 루프)**: `app`, `mock-target`, `runner-tools`를 분리한 compose 프로파일을 둔다. 실제 compose 파일 구현은 M0 산출물이지만, 계약은 여기서 고정한다.
- 목 서버는 별도 프로세스(다른 포트)로 먼저 기동, 러너가 친다.
- 둘 중 어느 모드든 부하 대상은 번들 목 서버뿐이다. 실서비스/공개 엔드포인트를 로컬 테스트 편의상 허용하지 않는다.

### 7.2 프로세스 격리·자원 상한 (로컬 맥북)

| 항목 | 방식 |
|---|---|
| 작업 디렉토리 | run_id별 격리 디렉토리(`runs/<run_id>/`)에 산출물·env·로그 |
| 프로세스 그룹 | 러너를 새 프로세스 그룹으로 spawn → `abort`가 그룹 전체 종료 |
| env 주입 | 화이트리스트 키만(자유 shell 금지), `ToolAction.args`에서 typed 전달 |
| 자원 상한 | 로컬은 소규모(VU/rate 상한 낮게) — 생성기 CPU>80%면 결과 무효 후보(§3.4) |
| 산출물 수집 | `summary_json`/`raw_output`/`generator_health` → `runs/<run_id>/` → `collect` 반환 |
| 정리 | 성공/실패 무관 부분 산출물 flush 후 핸들 해제(워크스페이스 위생) |

Docker 모드는 host-native와 같은 계약을 검증하는 **재현성 하니스**다.

| 서비스 | 역할 | 제약 |
|---|---|---|
| `app` | FastAPI/API + controller 프로세스 | 실행 권한은 controller만. 외부 네트워크는 기본 차단 또는 allowlist |
| `mock-target` | 번들 SUT | 고정 포트, profile volume read-only, control endpoint는 테스트 네트워크 내부 전용 |
| `runner-tools` | k6/Artillery 바이너리 보유 실행 컨테이너 | `tools/versions.json`과 실제 버전 일치 검사. CPU/mem limit로 생성기 병목을 관측 가능하게 둠 |

CI에서는 Docker 모드를 최소 1회 smoke로 돌려 host 환경 차이를 잡고, 개발 중 빠른 반복은 host-native 모드를 허용한다.

### 7.3 dry_run 기본값 → 승인 후 실제

```
ToolAction.dry_run = true (기본)  ──▶  compose + dry_run(정적/무부하)  ──▶  계획 검토
        │  사람 승인(approval_id + plan_hash)
        ▼
ToolAction.dry_run = false (승인된 액션만)  ──▶  submit → canary → full_run → collect
```

- 기본이 dry_run이라, 승인 전에는 어떤 실제 부하도 발생하지 않는다([[00-overview]] §6, [[04-safety-approval-audit]]).

---

## 8. 클라우드 러너 확장 지점 (후속)

동일 `RunnerAdapter`(§1) 뒤에 클라우드 러너를 **인터페이스만** 끼운다. `controller.py`는 변경 없이 `tool`/`ProvisionSpec.topology`/`runner_location`/`runner_provider`/`target_provider`로 어댑터를 디스패치한다([[02-data-model]], [[12-execution-topology-matrix]]). 실제 구현·실행은 [[08-milestones-roadmap]] M5+, 본 로컬 범위 제외([[00-overview]] D11).

| 후속 어댑터 | 매핑 | 주의 (검증 정정) |
|---|---|---|
| `runners/grafana_cloud_k6.py` | 동일 k6 스크립트 + `options.cloud.distribution`(load zone) | submit/collect를 Cloud REST API **v6**(`api.k6.io/cloud/v6`)로. canary/kill switch 의미는 유지 |
| `runners/aws_dlt_adapter.py` | Taurus on ECS/Fargate, 시나리오 REST/CLI 정의 | **AWS DLT MCP는 read-only**(7 tools), 실행은 REST/CLI 경로로 설계. GitHub 최신 릴리스와 AWS Solutions 랜딩 페이지 표기가 어긋날 수 있으므로 배포 artifact/version pin |
| `runners/azure_lt_adapter.py` | JMeter/Locust 엔진(.jmx/Locust 업로드) | dataplane REST API **`2026-04-01`은 Microsoft Learn reference로 확인**. lifecycle의 `2022-11-01`은 지원 baseline으로 해석하고, Azure 경로 채택 시 지원 엔진/region 및 API 버전 상수를 재확인 |
| `runners/self_hosted_container.py` | cloud VM/사내 서버의 Docker/k3d/kind에서 k6/Artillery 실행 | L1(remote self-hosted local). 관리형 cloud runner가 아니지만 target이 mock이 아니면 승인·allowlist·egress cost가 필요 |

확장 시 불변식 보존: **실행 권한은 여전히 controller만**, ToolAction typed-only, canary→full 승격, 승인+plan_hash, kill switch — 클라우드라고 완화하지 않는다. 차이는 `submit/collect`의 백엔드(로컬 프로세스 → self-hosted container/cloud API)뿐이고, `validate/compose/dry_run`은 가능한 한 로컬과 공유한다. `runner_provider != target_provider`인 X0(cross-cloud)는 양 Provider 정책·egress·allowlist·관측 지연을 승인 번들에 포함한다([[12-execution-topology-matrix]] §5).

---

## 9. 테스트 가능성 ([[09-testing-strategy]])

D9 집행: 어댑터·컨트롤러·목 서버를 계층별로 자동 검증한다.

| 계층 | 대상 | 방식 |
|---|---|---|
| **골든 파일** | `compose.py` (k6 JS / Artillery YAML 산출물) | 슬롯 입력 → 산출물·`content_hash`를 골든과 정확 비교. 결정성 검증 |
| **정적 검증 단위** | `validate`/`dry_run` | 잘못된 슬롯(타입·필수 누락·safe_limits 초과·셸 메타문자) → 거부 확인. 네트워크 0 |
| **생성기 병목 단위** | §3.4 신호 판정 | `dropped_iterations>0`/gen CPU>80% 입력 → `generator_saturated=true` 플래그·해석 보류 확인 |
| **통합(목 서버 상대)** | submit→canary→full_run→collect | 알려진 프로파일(p95/에러율/동시성) 주입 → summary가 기대 근방인지 대조 |
| **Docker smoke** | app + mock-target + runner-tools | compose 프로파일에서 k6 1개·Artillery 1개 최소 시나리오 실행. 버전 매니페스트와 컨테이너 내 바이너리 버전 일치 확인 |
| **계약** | controller 입력 | 승인 없는/만료/plan_hash 불일치 ToolAction → 실행 0 확인([[04-safety-approval-audit]] 계약과 공유) |
| **kill switch** | abort 발화 | 에러 급증/포화 프로파일 → abort 시점·`abort_reason`·`RunRecord.status` 검증 |

> 목 서버의 결정적 시드(§6.2) 덕분에 통합 테스트가 flaky하지 않다 — "주입 진실 ↔ 러너 출력" 대조가 재현 가능(D9의 핵심 메커니즘).

---

## 부록 · 산출물/스키마 정합

| 본 문서 산출물 | 연결 스키마([[02-data-model]]) |
|---|---|
| `ComposedArtifact.content_hash` | `ToolAction.args.expected_content_hash`(artifact 변조 방지), `plan_hash` 입력 결속 |
| `RunArtifacts.summary_json`·`generator_health` | `ObservationBundle.metrics`·`anomalies` ([[06-reporter-observability]]) |
| controller 실행 기록 | `RunRecord`(run_id/status/abort_reason/artifacts) |
| 어댑터 입력 슬롯 | `TestPlanSpec.param_slots`·`ScenarioTemplate.safe_limits/default_slo` |
| ToolAction 필드 | tool/args/approval_id/idempotency_key/timeout_sec/dry_run |
