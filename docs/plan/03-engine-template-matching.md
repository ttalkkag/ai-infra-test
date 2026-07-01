# 03 · 엔진과 템플릿 매칭

> 확정 결정·불변식은 [[00-overview]] §2·§6, 모듈 경계·의존 방향은 [[01-architecture]] §2·§4 참조. 본 문서는 **제어 평면(실행 권한 0, read-only)** 안의 엔진 코어(`engine/*`)가 자연어 프롬프트를 `TestPlanSpec`까지 닫는 과정을 구현 단위로 분해한다.
> 핵심 불변식 재확인: **자유 생성이 아니라 유한 템플릿 선택 + 슬롯 채움**, **매칭 성공 ≠ 실행 승인**, **clamp 정책은 [[04-safety-approval-audit]]가 소유**.

스키마 원형(ScenarioTemplate / UserIntent / TemplateMatch / TestPlanSpec)은 `docs/research/03-ai-agent-orchestration.md` §7(UserIntent/TestPlanSpec 등)·§9.5(ScenarioTemplate/TemplateMatch)에 정의되어 있고, Pydantic/SQLAlchemy 매핑은 [[02-data-model]]에서 확정한다. 본 문서는 그 스키마를 **소비/생산하는 로직**을 다룬다.

---

## 1. 매칭 파이프라인 5단계 (모듈·산출물 기준)

`engine/orchestrator.py`가 파이프라인 상태기계를 소유하고, `matcher.py`(1~3단계)와 `strategy.py`(4~5단계)에 위임한다. `retriever.py`는 임베딩 인덱스/신선도 보조다. **어느 단계도 실행 권한이 없으며**, 모든 단계 전이는 `AuditLog`에 append-only로 기록된다([[04-safety-approval-audit]]).

| 단계 | 모듈 | 입력 | 출력(산출물) | 실패 시 처리 |
|---|---|---|---|---|
| ① 의도 분류 | `matcher.classify_intent` (orchestrator 호출) | `UserIntent.raw_prompt` | `intent_label`(load/stress/spike/soak/breakpoint/report-only) + `intent_conf` | 분류 불가/모호 → `decision=clarify`, 질문 생성, 파이프라인 정지 |
| ② 의미 검색/분류 | `matcher.retrieve` (+ `retriever`) | prompt, `intent_label` | scored 후보 리스트 `[(template_id, score)]` | 후보 0 또는 인덱스 비어있음 → `decision=reject` |
| ③ top-k 결정 | `matcher.rank_and_decide` | scored 후보, τ_low/τ_high | `TemplateMatch`(decision 포함) | τ 미달/모호 → `clarify` 또는 `reject`/`route-to-expert` |
| ④ 슬롯 채움 | `strategy.fill_slots` | 선택 `ScenarioTemplate`, prompt | `param_slots_filled` + `unfilled_required_slots` | 필수 슬롯 누락 → **추정 금지**, `decision=clarify` |
| ⑤ 안전 clamp | `strategy.assemble_plan` → `safety/clamp.py` | filled slots, `safe_limits`, policy | `TestPlanSpec`(clamp 결과·requires_approval 표시) | 상한 초과/prod 대상 → clamp 또는 `requires_approval`/deny ([[04-safety-approval-audit]] 소유) |

> 5단계 분해는 `docs/research/03` §9.2의 4단계를 구현 관점에서 **②(검색)와 ③(top-k 결정)으로 분리**한 것이다. 결정 로직을 검색에서 떼어내야 임계값 보정(§8)과 단위 테스트(§10)가 독립적으로 가능하다.

### 1.1 Orchestrator 상태기계 (`engine/orchestrator.py`)

```python
# 제어 평면. 부작용 없음(I/O는 주입된 store/llm/index를 통해서만).
class MatchPipelineResult(BaseModel):
    intent: UserIntent
    match: TemplateMatch
    plan: TestPlanSpec | None        # decision != auto-proceed 이면 None
    clarification: list[str] | None  # 사람에게 물을 질문

class Orchestrator:
    def __init__(self, matcher: Matcher, strategy: Strategy, store: Store, audit: Audit): ...

    def run(self, intent: UserIntent) -> MatchPipelineResult:
        self.audit.append("intent.received", intent.request_id)

        # ① 의도 분류
        intent_label, intent_conf = self.matcher.classify_intent(intent.raw_prompt)
        intent = intent.copy(update={"test_type": intent_label})

        # ②③ 검색 + top-k 결정 → TemplateMatch
        match = self.matcher.match(intent, intent_conf)
        self.audit.append("template.matched", match.match_id, match.dict())

        # 게이트: 비-진행 결정이면 ToolAction/Plan 생성 자체를 막는다(불변식)
        if match.decision != "auto-proceed":
            return MatchPipelineResult(intent, match, plan=None,
                                       clarification=match.clarification_questions)

        # ④⑤ 슬롯 채움 + clamp → TestPlanSpec
        match = self.strategy.fill_slots(match, intent)        # param_slots_filled 갱신
        if match.unfilled_required_slots:                      # 필수 누락 → 추정 금지
            match.decision = "clarify"
            return MatchPipelineResult(intent, match, plan=None,
                                       clarification=match.clarification_questions)

        plan = self.strategy.assemble_plan(match, intent)      # 내부에서 safety.clamp 호출
        self.audit.append("plan.assembled", plan.plan_id, {"plan_hash": plan.plan_hash})
        return MatchPipelineResult(intent, match, plan=plan, clarification=None)
```

**불변식 구현점**: `decision != "auto-proceed"`거나 `unfilled_required_slots`가 있으면 `plan=None`을 반환한다 → 하위(승인·실행)에서 받을 `ToolAction`이 존재할 수 없다. `docs/research/03` §9.3의 "needs_clarification/fallback_to_human 동안 ToolAction 생성 금지"를 코드 경계로 강제한다.

### 1.2 단계별 입출력·실패 상세

**① 의도 분류 (`matcher.classify_intent`)**
- 입력: `raw_prompt`. 출력: `(intent_label, intent_conf)`. 방법: `llm.classify(prompt, labels=TEST_TYPES)` (유한 라벨 선택, §5). LLM 미가용 시 키워드 폴백(`keyword` method).
- 실패: 라벨 신뢰도 < `intent_conf_min`이면 `intent_label="ambiguous"` → orchestrator가 `clarify`. "report-only"는 검색 단계를 건너뛰고 곧장 `baseline_compare_report`로 라우팅.

**② 의미 검색/분류 (`matcher.retrieve`)**
- 입력: prompt, `intent_label`. 출력: `[(template_id, score)]`(cosine). 방법: `retriever`가 `ScenarioTemplate.description + example_utterances` 임베딩을 사전계산·캐시(`embedding_ref`)하고, query 임베딩과 코사인 비교. `intent_label`로 후보 사전필터(예: report-only는 report 템플릿만).
- 실패: 인덱스 비어있음/임베딩 호출 실패 → `decision=reject`, audit에 사유. 임베딩 차원·모델 불일치(인덱스 빌드 모델 ≠ query 모델)는 `retriever` 로드시 검증해 차단.

**③ top-k 결정 (`matcher.rank_and_decide`)**: §3에서 상세.

**④ 슬롯 채움 (`strategy.fill_slots`)**: §4에서 상세.

**⑤ 안전 clamp (`strategy.assemble_plan`)**: 채워진 슬롯 + 템플릿 `safe_limits`를 `safety/clamp.py`에 넘긴다. clamp/정책 판정 로직과 `requires_approval` 결정은 **[[04-safety-approval-audit]]가 소유**하고, 엔진은 그 결과를 `TestPlanSpec`에 직렬화만 한다(§9).

### 1.3 Matcher 인터페이스 스케치 (`engine/matcher.py`)

```python
class Matcher:
    def __init__(self, llm: LLMProvider, index: TemplateIndex,
                 cfg: MatchConfig):  # cfg.tau_low, cfg.tau_high, cfg.method
        ...

    def classify_intent(self, prompt: str) -> tuple[str, float]: ...

    def retrieve(self, prompt: str, intent_label: str
                 ) -> list[tuple[str, float]]:        # top-k (template_id, cosine)
        ...

    def rank_and_decide(self, intent: UserIntent, candidates, intent_conf
                        ) -> TemplateMatch:
        # τ 비교 + 단일우세 판정 + decision enum 산출 (§3)
        ...

    def match(self, intent: UserIntent, intent_conf: float) -> TemplateMatch:
        cands = self.retrieve(intent.raw_prompt, intent.test_type)
        return self.rank_and_decide(intent, cands, intent_conf)
```

---

## 2. 소형 템플릿 집합(5~10개) 매칭 전략

초기 템플릿은 5종(§7), 운영 단계에서도 수십 종 규모다. **검색 대상이 작을 때는 임베딩 top-k의 이점(대규모 코퍼스 확장성)이 약해지고, 유한 목록 중 선택하는 LLM 분류기의 정확도·근거제시 이점이 커진다.**

| 축 | embedding-cosine top-k | llm-classifier (유한 목록 선택) |
|---|---|---|
| 적합 규모 | 수백~수천+ (확장성 강점) | 수~수십 (목록을 프롬프트에 다 넣을 수 있음) |
| 지연/비용 | 임베딩 1회(캐시 시 매우 저렴) | LLM 호출 1회(분류용 소형 모델로 절감 가능) |
| 결정 근거 | score만(왜 그 템플릿인지 설명 약함) | 라벨 + 근거 텍스트(가드레일·감사에 유리) |
| 오타·동의어 강건성 | 임베딩이 의미 포착(강함) | 모델 일반화에 의존(대체로 강함) |
| 환각/이탈 위험 | 없음(고정 인덱스에서 최근접만) | 목록 밖 라벨 생성 가능 → structured output enum으로 차단 필요 |
| 결정성(테스트) | 높음(벡터·임계 고정 시 결정적) | 낮음(목 LLM 필요, §10) |
| 임계 보정 | score 분포 보정 용이 | 신뢰도 캘리브레이션 어려움(보조 score 권장) |

**권장: hybrid (1차 LLM 분류기 + 임베딩 보조).**
- 소형 목록에서는 LLM 분류기를 **1차 결정자**로 둔다(유한 enum 선택 + 근거 텍스트). 동시에 임베딩 cosine을 **보조 신호**로 계산해, 둘이 일치하면 신뢰도를 상향, 불일치하면 `clarify`로 강등(교차검증 게이트, `docs/research/03` §9.3).
- 임베딩은 사전계산·캐시되어 거의 무료이므로 보조 신호로 항상 켜둔다. cosine은 `match_confidence`의 수치 백본으로도 쓴다(LLM 자기보고 신뢰도보다 보정 가능, §8).
- 초기 `combine()` 산식은 보수적으로 고정한다. `agree=true`이면 `min(1.0, 0.55*emb_score + 0.35*llm_conf + 0.10)`, `agree=false`이면 `min(0.49, 0.70*emb_score + 0.30*llm_conf)`로 강등해 자동 진행을 막는다. 가중치는 M2 eval에서 보정하되, **불일치가 자동 실행으로 이어지지 않는 불변식**은 유지한다.
- 임베딩 인덱스는 초기에는 `templates/index.json`(template metadata) + `templates/embeddings.npy`(정규화 벡터)로 충분하다. 템플릿 수가 늘거나 순수 임베딩 모드가 필요해질 때만 `sqlite-vec` 등 벡터스토어를 M2 후속 작업으로 도입한다.

```python
def match(self, intent, intent_conf):
    cands = self.retrieve(intent.raw_prompt, intent.test_type)   # 임베딩 보조
    emb_best = cands[0] if cands else (None, 0.0)
    if self.cfg.method in ("llm-classifier", "hybrid"):
        llm_pick, llm_conf, rationale = self.llm.classify(
            intent.raw_prompt, labels=[t.template_id for t in self.index.active()])
        agree = (llm_pick == emb_best[0])
        conf = combine(emb_best[1], llm_conf, agree)             # 일치 시 상향
        method = "hybrid"
    else:  # embedding-cosine 단독 또는 keyword 폴백
        llm_pick, conf, method = emb_best[0], emb_best[1], self.cfg.method
    return self.rank_and_decide(intent, cands, conf,
                                primary=llm_pick, agree=agree)
```

`match_method` enum = **`embedding-cosine` / `llm-classifier` / `hybrid` / `keyword`** (`docs/research/03` §9.5.2, [[02-data-model]]와 통일). `keyword`는 LLM·임베딩 모두 불가한 오프라인 폴백(결정적 단위 테스트의 기본).

---

## 3. 임계값 τ_low/τ_high와 decision 처리

`MatchConfig.tau_low`, `tau_high`는 설정(`config.py`)에서 주입하고 §8에서 오프라인 보정한다. M2 초기값은 **`tau_low=0.55`, `tau_high=0.80`, `margin_min=0.08`**로 둔다. `rank_and_decide`가 `match_confidence`와 **단일 후보 우세(top1 − top2 margin ≥ `margin_min`)**를 함께 본다. 점수만 높고 1·2위가 근소하면 모호로 간주한다.

| 조건 | decision | 처리 | 연결 |
|---|---|---|---|
| `conf ≥ τ_high` **그리고** 단일우세(margin ≥ `margin_min`) | `auto-proceed` | 슬롯 채움 진행(실행은 여전히 승인 정책 적용) | §4 |
| `τ_low ≤ conf < τ_high` 또는 근소차(margin < `margin_min`) | `clarify` | **실행 거부** + top-k 후보 제시, 사람이 선택 | HITL ([[04-safety-approval-audit]]) |
| `conf < τ_low` (사실상 매칭 없음) | `reject` | **실행 거부**. 비전문가 자유생성 폴백 금지 | — |
| `conf < τ_low` **그리고** requester=expert | `route-to-expert` | 전문가 큐레이션/자유생성(승인 경로) | §6 큐레이션 |
| 의도분류 ↔ 검색결과 불일치(hybrid disagree) | `clarify` | 교차검증 실패 → 사람 확인 | §2 교차검증 |

```python
def rank_and_decide(self, intent, candidates, conf, primary=None, agree=True):
    top = candidates[:self.cfg.k]
    margin = (top[0][1] - top[1][1]) if len(top) >= 2 else top[0][1] if top else 0.0
    if not top or conf < self.cfg.tau_low:
        decision = "route-to-expert" if intent.requester_expertise == "expert" else "reject"
        matched = None
    elif conf >= self.cfg.tau_high and margin >= self.cfg.margin_min and agree:
        decision, matched = "auto-proceed", (primary or top[0][0])
    else:
        decision, matched = "clarify", None
    return TemplateMatch(
        match_id=new_id(), intent_ref=intent.request_id,
        matched_template_id=matched, match_method=self.cfg.method,
        match_confidence=round(conf, 4),
        candidates=[{"template_id": t, "score": round(s, 4)} for t, s in top],
        threshold_high=self.cfg.tau_high, threshold_low=self.cfg.tau_low,
        decision=decision, fallback_to_human=(decision != "auto-proceed"),
        clarification_questions=build_clarification(top, intent) if decision == "clarify" else None,
    )
```

**원칙(`docs/research/03` §9.3 재확인)**: "가장 유사한 템플릿"은 항상 무언가를 반환하므로, **모호성을 숨기지 말고 surfacing**한다. 보정 전 기본값은 보수적(높은 τ)으로 둔다 — 거짓 매칭이 불필요한 명확화보다 위험하다.

---

## 4. 슬롯 채움 (structured output, 추정 금지)

`strategy.fill_slots`는 선택 템플릿의 `param_slots` 스키마를 **typed structured output**으로 강제해 프롬프트에서 값을 추출한다. JSON Schema 준수로 환각 enum·누락 키를 차단한다(OpenAI Structured Outputs 근거). **필수 슬롯을 추출하지 못하면 기본값을 만들지 않고(추정 금지) 질문으로 돌린다.**

```python
def fill_slots(self, match: TemplateMatch, intent: UserIntent) -> TemplateMatch:
    tpl = self.store.get_template(match.matched_template_id)
    schema = jsonschema_from_slots(tpl.param_slots)   # required/type/min/max/allowed_values
    filled = self.llm.structured(intent.raw_prompt, schema=schema)  # typed, 검증됨(§5)
    missing = [s.name for s in tpl.param_slots
               if s.required and filled.get(s.name) in (None, "")]
    if missing:
        match.unfilled_required_slots = missing
        match.clarification_questions = [slot_question(tpl, n) for n in missing]
    match.param_slots_filled = filled
    return match
```

**슬롯 스키마(공통 형상; 템플릿별 `param_slots`로 선택·확장)**

| slot | type | required | 비고 / allowed·범위 |
|---|---|---|---|
| `title` | string | yes | 게임 타이틀(타이틀-가변 핵심 키, §6) |
| `env` | enum | yes | local/dev/staging/pre-prod/prod-like/prod — prod류는 `requires_approval` 강제 |
| `host` | string | yes | 대상 host/base URL(env allowlist와 교차검증) |
| `account_pool` | ref | cond | 계정/토큰 풀 참조(여정·인증 템플릿 필수, 로컬은 더미) |
| `peak_ccu` 또는 `peak_rps` | number | yes | 둘 중 하나 필수. `safe_limits.max_vu`/`max_rps`로 clamp(§5단계) |
| `duration` | number(sec) | yes | `max_duration_sec` 이하 |
| `ramp` | object | no | {up_sec, hold_sec, down_sec} (없으면 템플릿 default) |
| `p95_ms` / `p99_ms` | number | no | 없으면 `default_slo` 사용 |
| `error_budget` | number(0~1) | no | error rate 상한(SLO) |
| `cost_cap` / `quota_cap` | number | cond | BaaS/유료 경로 템플릿 필수(Firebase quota·비용) |

- **추정 금지 규칙**: `required && 미추출` → `unfilled_required_slots`. `default`가 정의된 선택 슬롯만 기본값 허용. `peak_ccu`/`peak_rps`는 "둘 중 하나" 조건부 필수(`oneOf`).
- **타입·enum 강제**: structured output 스키마에 `allowed_values`/`min`/`max`를 넣어 모델이 enum 밖 값·범위 밖 수치를 못 내게 한다. 그래도 안전 상한은 §5단계 clamp가 최종 방어한다.

---

## 5. LLM provider 추상화 ([[00-overview]] D5)

교체가 잦다는 전제(D5) 하에 호스티드·로컬 어댑터를 **둘 다 초기 지원**한다. 엔진은 인터페이스(`llm/base.py`)에만 의존하고, 구체 어댑터는 `registry.py`가 설정으로 주입한다([[01-architecture]] §4: `llm`은 비결정적 부작용을 내장하지 않음 → 테스트 시 목 주입).

### 5.1 인터페이스 (`llm/base.py`)

```python
class LLMProvider(Protocol):
    def complete(self, prompt: str, *, max_tokens: int = 512) -> str: ...
    def structured(self, prompt: str, *, schema: dict) -> dict:
        """JSON Schema 준수 보장된 dict 반환. 미준수 시 ProviderError."""
    def embed(self, texts: list[str]) -> list[list[float]]: ...
    def classify(self, prompt: str, *, labels: list[str]
                 ) -> tuple[str, float, str]:        # (label∈labels, conf, rationale)
        """labels 밖 라벨 반환 금지(구현이 강제)."""
    @property
    def capabilities(self) -> ProviderCaps: ...       # native_schema/embed 지원 여부
```

| 어댑터 | 파일 | 대상 | structured 구현 |
|---|---|---|---|
| Hosted | `llm/hosted.py` | Claude / OpenAI 등 | native structured outputs(JSON Schema 모드)·function calling |
| Local | `llm/local.py` | Ollama 등 | json/grammar 제약 디코딩 → Pydantic 검증 → 제한 재시도 |
| Mock | `tests/_mocks/mock_llm.py` | 테스트 | 입력→고정 출력 매핑(결정적, §10) |
| Registry | `llm/registry.py` | 선택 | `config.llm.provider` 기반 인스턴스화 + capabilities 게이트 |

### 5.2 구조화 출력 신뢰성 차이의 추상화·검증

호스티드는 스키마 준수를 네이티브로 보장하지만, 로컬 모델은 보장 강도가 낮다. 인터페이스는 **"검증된 dict 또는 예외"라는 계약을 동일하게** 노출하고, 신뢰성 차이는 어댑터 내부 + 공통 검증 래퍼로 흡수한다.

```python
def structured(self, prompt, *, schema):
    for attempt in range(self.cfg.max_retries):           # 로컬: 보정 재시도, 호스티드: 보통 1회
        raw = self._raw_structured(prompt, schema, repair_hint=attempt > 0)
        ok, obj, err = validate_against_schema(raw, schema)   # 공통 jsonschema/Pydantic 검증
        if ok:
            return obj
    raise ProviderError(f"schema violation after {self.cfg.max_retries}: {err}")
```

- **capabilities 게이트**: `registry`가 `provider.capabilities.native_schema`/`embed` 미지원을 감지하면, `structured`는 제약 디코딩+검증 경로로, `embed` 미지원이면 `method`를 `llm-classifier` 단독으로 강등(임베딩 보조 비활성, §2). 능력 부재가 조용한 오작동이 아니라 **명시적 경로 변경**이 되게 한다.
- **검증 일원화**: 스키마 검증은 어댑터 밖 공통 래퍼에서 한 번만 — 어떤 provider든 엔진이 받는 객체는 동일하게 검증됨이 보장된다. 실패는 hallucination이 아니라 `clarify`/`reject`로 안전하게 닫힌다.

### 5.3 테스트용 목 어댑터

`MockLLMProvider`는 `(method, 입력 해시) → 사전 정의 출력` 매핑으로 동작하며 네트워크·비결정성이 0이다. 평가셋(§8) 회귀 테스트는 목 또는 녹화된(record/replay) 임베딩으로 결정성을 유지한다([[09-testing-strategy]]).

---

## 6. ScenarioTemplate 레지스트리 (YAML)

템플릿은 `templates/*.yaml`로 **버전관리 대상**이며([[01-architecture]] §4), 코드 배포와 분리해 검수·롤백한다. `templates/_schema.md`가 스키마를 문서화한다.

### 6.1 파일 구조·필드

```yaml
# templates/event_open_readiness.yaml
schema_version: "1"
template_id: event_open_readiness
name: "이벤트 오픈 전 라이브 안정성 점검"
description: >
  신규 이벤트/컨텐츠 오픈 전 예상 피크에서 라이브가 버티는지 확인. (의미검색 임베딩 대상)
example_utterances:            # 라우팅용 대표 발화(임베딩 대상)
  - "이벤트 오픈 전에 라이브가 흔들릴지 확인해줘"
  - "신규 컨텐츠 오픈 수준으로 부하 봐줘"
test_type: spike               # 주 분류(조합은 target_pattern/strategy에서 확장)
tool_family: k6                # k6/artillery/locust/jmeter/gatling/managed/report-only
target_pattern: REST           # REST/WebSocket/gRPC/browser/BaaS-mixed
param_slots:                   # 타이틀-가변 파라미터
  - {name: title,    type: string, required: true}
  - {name: env,      type: enum,   required: true, allowed_values: [staging, pre-prod, prod-like]}
  - {name: peak_ccu, type: number, required: true, min: 1, max: 50000}
  - {name: duration, type: number, required: true, min: 60, max: 3600}
  - {name: ramp,     type: object, required: false}
safe_limits:                   # 검증된 안전 한계(비전문가 우회 불가, 내장)
  max_rps: 5000
  max_vu: 50000
  max_duration_sec: 3600
  max_cost: 0
  allowed_envs: [staging, pre-prod, prod-like]
target_whitelist: []           # 허용 타이틀(비면 env allowlist만 적용)
default_slo: {p95_ms: 300, p99_ms: 800, error_rate: 0.01}
requires_approval_default: by-env   # always/by-env/never (prod=always)
status: active                 # draft/reviewed/active/deprecated
curated_by: "<engineer>"
reviewed_at: "2026-06-26T00:00:00Z"
```

필드 의미는 `docs/research/03` §9.5.1(ScenarioTemplate)과 동일하며, [[02-data-model]]에서 Pydantic으로 1:1 매핑한다.

### 6.2 큐레이션 단계 (`status` 전이)

```
draft ──(engineer review)──▶ reviewed ──(policy review)──▶ active ──▶ deprecated
```

| 단계 | 게이트 | 검수 항목 |
|---|---|---|
| draft | 작성자 제출 | description/example_utterances/슬롯 초안 |
| engineer review | 엔지니어 | tool_family·target_pattern 적합성, 스크립트 빌드 가능성, 슬롯 타입 |
| policy review | 정책/안전 | `safe_limits`·`target_whitelist`·`requires_approval_default`·env allowlist |
| active | 양쪽 통과 | 검색 인덱스에 포함(임베딩 사전계산) |
| deprecated | 회수 | 인덱스 제외, 신규 매칭 불가(기존 RunRecord 추적성은 보존) |

**운영 중 즉석 생성 금지**(`docs/research/03` §9.1): 새 시나리오는 반드시 이 파이프라인을 거쳐야 `active`가 된다. `status != active`인 템플릿은 `retriever`가 인덱스에서 배제한다 → 검수 안 된 템플릿으로의 매칭 자체가 불가능.

### 6.3 타이틀-불변 로직 + 타이틀-가변 파라미터 (20+ 타이틀)

20+ 게임 타이틀 대응은 **스크립트 복제가 아니라 슬롯 교체**로 해결한다(`docs/research/03` §9.4, `docs/research/01` §10 템플릿 라이브러리화).
- **타이틀-불변**(템플릿 본문, 검수 1회): 부하 모델(executor/phase), 여정 구조, SLO 형상, 안전 한계.
- **타이틀-가변**(`param_slots`, 매칭마다 채움): `title`/`host`/`account_pool`/`peak_ccu·rps`/`env`/`default_slo` override.

따라서 "A 타이틀도 B처럼 돌려줘"(시나리오 #9)는 같은 `template_id` + 다른 슬롯값으로 처리되고, env allowlist·`target_whitelist`가 staging/sandbox만 자동 후보로 남긴다.

---

## 7. 초기 템플릿 5종

Firebase는 **REST 경로와 실시간 경로를 분리**한다(`docs/research/01` §7~8 · `docs/research/02` §3.5: Firestore 실시간 Listen은 gRPC/WebChannel, RTDB는 WS/long-polling이라 HTTP 템플릿으로 대체 불가).

| template_id | 목적 | 테스트 조합 | 주요 슬롯 | 권장 도구(tool_family) |
|---|---|---|---|---|
| `event_open_readiness` | 이벤트/컨텐츠 오픈 전 라이브 장애 가능성 | load + spike + breakpoint | title, env, peak_ccu/rps, duration, ramp, p95/p99, error_budget | k6 또는 Artillery |
| `common_journey_regression` | 로그인→컨텐츠 진행→종료 공통 여정 성능 회귀 | load | title, env, account_pool, journey_steps, think_time, p95/p99 | k6 / Artillery |
| `firebase_rest_crud_capacity` | Firebase Auth/Functions/Firestore **REST CRUD** 용량 | load + breakpoint | project/env, auth_mode, read_write_ratio, Firestore ramp(500/50/5), App Check debug token 여부, quota/cost_cap | k6 / Artillery |
| `firebase_realtime_sync_probe` | RTDB/Firestore **실시간** 동기화 연결·리스너 부하 | load + soak(소규모) | connection_count, listener_count, message_rate, browser_or_wire_mode, quota/cost_cap | k6 WS/gRPC 또는 browser/Playwright |
| `baseline_compare_report` | 과거 실행 대비 회귀 분석만 | report-only | baseline_run, current_run, metric_window, SLO | Reporter only ([[06-reporter-observability]]) |

- `firebase_rest_crud_capacity`(REST/BaaS-mixed)와 `firebase_realtime_sync_probe`(WebSocket/gRPC)는 `target_pattern`이 달라 의도분류·검색 단계에서 서로 섞이지 않게 한다.
- 실시간 probe는 도구·실행기가 분리되므로 [[05-runners-and-mock-target]]의 어댑터 능력(WS/gRPC 지원)과 교차검증한다.
- `cost_cap`/`quota_cap`은 Firebase 두 템플릿에서 조건부 필수(Firebase 예산 알림은 사용을 cap하지 않음 — [[00-overview]] §5, clamp는 [[04-safety-approval-audit]]).

---

## 8. 평가셋 (`eval/`)

오프라인 평가셋으로 **임계값(τ_low/τ_high·margin_min)과 매칭 정확도를 회귀 테스트**한다([[01-architecture]] §4 `eval/`, [[09-testing-strategy]]).

### 8.1 형식

```jsonl
# eval/matching_cases.jsonl  (발화 30+, persona 균형: planner/qe/client-dev)
{"id":"p01","persona":"planner","utterance":"이벤트 오픈 전에 라이브가 흔들릴지 봐줘",
 "expected_template":"event_open_readiness","expected_decision":"auto-proceed"}
{"id":"q03","persona":"qe","utterance":"staging API 30분 돌려서 p95 300 넘는지 확인",
 "expected_template":"common_journey_regression","expected_decision":"clarify"}
{"id":"c07","persona":"client-dev","utterance":"Firestore에 글 쓰고 읽는 부하 얼마나 버텨?",
 "expected_template":"firebase_rest_crud_capacity","expected_decision":"auto-proceed"}
{"id":"c09","persona":"client-dev","utterance":"랭킹 실시간 동기화가 버티는지 봐줘",
 "expected_template":"firebase_realtime_sync_probe","expected_decision":"auto-proceed"}
{"id":"x02","persona":"planner","utterance":"production에 10만 RPS로 바로 때려",
 "expected_template":null,"expected_decision":"reject"}
{"id":"x05","persona":"qe","utterance":"뭔가 테스트 좀 해줘","expected_template":null,
 "expected_decision":"clarify"}
```

라벨 종류: persona별 명확 매칭(auto-proceed), 모호(clarify), 매칭 없음/위험(reject), 전문가 경로(route-to-expert). 위험 요청(prod/10만 RPS/결제)은 **매칭과 무관하게 reject/approval**로 닫혀야 함을 명시적 케이스로 포함([[00-overview]] §7-1·§7-2).

### 8.2 임계 보정 (거짓매칭 vs 불필요한 명확화 트레이드오프)

```python
# eval/calibrate.py (오프라인, 목/녹화 임베딩으로 결정적)
for tau_high in grid_high:
  for tau_low in grid_low:
     m = run_matcher_over(eval_cases, tau_low, tau_high)   # 실행 없음, 매칭만
     report(tau_low, tau_high,
            false_match_rate = m.wrong_auto_proceed / m.total,        # 위험: 낮춰야
            unnecessary_clarify_rate = m.clarify_on_clear / m.clear,  # 마찰: 낮추면 좋음
            reject_recall = m.reject_caught / m.should_reject)        # 안전: 1.0 목표
```

- **목적함수**: `false_match_rate`(거짓 매칭)와 `reject_recall`(위험 차단)을 1순위 제약으로 두고, 그 다음 `unnecessary_clarify_rate`를 최소화한다. 즉 **거짓 매칭 0·위험 차단 100%를 만족하는 범위에서 명확화 마찰을 줄인다**.
- 보정 전 기본값은 보수적(높은 τ): 의심스러우면 진행보다 명확화(`docs/research/03` §9.3). 보정 산출물(τ 권장값·곡선)은 [[10-open-questions-reverify]]에 기록하고 `config.py` 기본값으로 반영.

---

## 9. 안전 연결 (clamp는 [[04-safety-approval-audit]] 소유)

- 파이프라인 ⑤단계의 `clamp`는 **호출만 엔진에서 하고, 정책·판정·`requires_approval` 결정은 [[04-safety-approval-audit]]가 소유**한다. 엔진은 결과를 `TestPlanSpec`에 직렬화할 뿐 정책을 해석하지 않는다.
- **매칭 성공 ≠ 실행 승인**: `decision=auto-proceed`는 "계획을 만들어도 좋다"일 뿐, 어떤 `ToolAction`도 승인 게이트([[01-architecture]] §3 plan_hash)와 Execution Controller([[05-runners-and-mock-target]])를 거치지 않으면 실행되지 않는다.
- 위험 요청(prod/고부하/결제·외부 파트너/destructive/App Check debug token)은 `match_confidence`와 무관하게 단락(short-circuit)되어 deny 또는 approval로 간다([[00-overview]] §6, `docs/research/05` §3 작업 3분류 제약).
- 엔진이 생성하는 `TemplateMatch`/`TestPlanSpec`/`candidates`/`match_method`는 전부 audit에 기록되어 오매칭 사후분석·임계 재보정의 근거가 된다.

---

## 10. 테스트 가능성 ([[09-testing-strategy]])

D9(테스트 가능성 1급 원칙)에 따라 엔진은 **순수 함수 + 주입된 의존성**으로 설계한다(`llm`/`store`/`index`/`audit` 주입, 부작용 없음).

| 대상 | 방법 | 결정성 확보 |
|---|---|---|
| 의도 분류/검색/top-k 결정 | `MockLLMProvider` + 고정 임베딩으로 단위 테스트 | 입력→출력 고정, τ 경계값 테이블 테스트 |
| 슬롯 채움(추정 금지) | 필수 누락 케이스 → `unfilled_required_slots`·`clarify` 단언 | 목 structured output |
| decision 게이트 불변식 | `decision != auto-proceed` ⇒ `plan is None` 계약 테스트 | ToolAction 미생성 단언 |
| 매칭 정확도 | `eval/matching_cases.jsonl` 회귀 테스트(§8) | 목/녹화 임베딩 |
| τ 보정 안정성 | 보정 스크립트 산출물 스냅샷 비교 | 결정적 그리드 탐색 |
| provider 추상화 | hosted/local/mock가 동일 `structured` 계약 통과 | 스키마 검증 일원화(§5.2) |

- **목 LLM으로 결정적 단위 테스트**: 네트워크·비결정성 0. CI에서 LLM 호출 없이 엔진 전 경로 검증.
- **평가셋 기반 회귀 테스트**: 템플릿/임계 변경 시 false_match·reject_recall 회귀를 자동 감지. 신규 템플릿 `active` 승격 전 평가셋 통과를 게이트로 둔다.

---

## 부록 · 형제 문서 연결

- 스키마 매핑(Pydantic/SQLAlchemy): [[02-data-model]]
- clamp·승인·plan_hash·audit·인젝션 방어: [[04-safety-approval-audit]]
- 실행기(k6/Artillery)·목 서버·Execution Controller: [[05-runners-and-mock-target]]
- 관측·해석·baseline 비교(`baseline_compare_report` 소비처): [[06-reporter-observability]]
- 화면 흐름(명확화 질문·계획 검토 UI): [[07-web-ui-api]]
- 테스트 철학·평가셋·결정성: [[09-testing-strategy]]
- τ 보정 결과·남은 가정: [[10-open-questions-reverify]]
