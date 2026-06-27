# 06 · 관측 수집 · 해석 · 리포트 (Reporter & Observability)

> 확정 결정·평면 분리는 [[00-overview]] §2 / [[01-architecture]] §1 참조. 본 문서는 **실행 평면이 끝난 뒤**의 read-only 파이프라인(`reporter/collector.py` → `reporter/interpreter.py` → `reporter/composer.py`)을 계획한다.
> 방법론·SLO 근거는 `docs/research/04-test-methodology-slo.md`(이하 **방법론 §N**)에 있고, 본 문서는 그 결론을 **코드 모듈의 입출력·판정 규칙·테스트**로 옮긴다. 외부 문서 URL 전체 레지스트리는 방법론 §2를 단일 출처로 둔다.

---

## 1. 위치·불변식·한 줄 정의

Reporter는 **읽기 전용**이다. 실행 권한 0, 부작용 0(파일/DB write는 자기 산출물 영속화에 한정). 입력은 실행 평면이 남긴 `RunRecord` + artifacts이고, 출력은 `ObservationBundle` → `InterpretationSpec` → `FinalReport`이다.

```
[실행 평면] Runner → RunRecord(+artifacts: k6 summary JSON, Artillery report, mock metrics)
        │  (read-only 경계)
        ▼
collector.py  ── ObservationBundle ──▶ interpreter.py ── InterpretationSpec ──▶ composer.py ── FinalReport
   (정규화)            (불변 입력)         (SLO 판정)         (불변 입력)          (경영/기술 분리)
        └────────────────────────── AuditLog append-only (모든 단계 기록, [[04-safety-approval-audit]]) ──────────┘
```

핵심 불변식 (구현 시 항상 참):

- [ ] Reporter는 `ToolAction`을 만들지도 실행하지도 않는다 (실행은 [[05-runners-and-mock-target]]의 Execution Controller만).
- [ ] **모든 주장(finding/SLO 판정/회귀)은 `evidence_refs`를 가진다.** 근거 없는 결론은 composer가 **fail-closed**로 거부([[00-overview]] §7-5).
- [ ] 판정은 **평균이 아니라 percentile + error budget** 기준(방법론 §5, §8-3·8-4).
- [ ] **부하 생성기 병목**은 SLO 판정보다 먼저 검사하는 1급 게이트(방법론 §8-1).
- [ ] Sandbox(가짜/고정 메트릭)와 Local Runner(실측)는 **동일한 interpreter/composer 코드 경로**를 탄다. 차이는 `ObservationBundle.source`와 라벨뿐 → 파이프라인 전체가 골든 테스트로 닫힌다([[09-testing-strategy]]).

---

## 2. Observation Collector (`reporter/collector.py`, read-only)

### 2.1 입력 소스와 MVP 단계 (D6)

| 단계 | 입력 소스 | `source` | 비고 |
|---|---|---|---|
| **Sandbox MVP** (M3) | 고정 fixture JSON (`eval/observations/*.json`) | `frozen` | 실행 0. 결정적 가짜 메트릭. UI/리포트/해석 파이프라인 검증용 |
| **Local Runner MVP** (M4) | k6 `--summary-export`/handleSummary JSON, Artillery `--output` report JSON, 목 서버 `/metrics`(Prometheus 텍스트) | `measured` | [[05-runners-and-mock-target]]가 목 서버 대상 실측 후 artifact 경로를 `RunRecord.artifacts`에 남김 |
| **Cloud (후속)** | OTel/Prometheus, Cloud Monitoring(BaaS) | `measured-remote` | 관측 지연·서버측 histogram 처리(§7·§9). 본 로컬 범위 **제외** |

> Sandbox와 Local의 유일한 분기는 **collector의 입력 어댑터**다. ObservationBundle 이후는 동일. "report-only/synthetic" 라벨은 `source != measured`일 때 자동 부여(§8).

### 2.2 입력 어댑터 (도구별 정규화)

도구마다 필드명이 다르므로 **표준 `MetricSeries`로 정규화**한다. percentile은 **재계산하지 않고** 도구가 산출한 값을 그대로 운반하되, 그 산출 방식(client-side)을 메타로 기록한다(§6 quantile 평균 금지).

| 도구 | 원천 필드(예) | 표준 키 | 메타 |
|---|---|---|---|
| k6 | `metrics.http_req_duration.values["p(95)"]` | `latency.p95` | `percentile_side=client`, `source_tool=k6` |
| k6 | `http_req_failed.values.rate` | `error.req_failed_rate` | 4xx/5xx 분리는 `http_req_failed`만으론 불가 → status별 `checks`/custom counter 병행 |
| k6 | `dropped_iterations.values.count`, `vus_max` | `generator.dropped_iterations`, `load.vus_max` | **생성기 건전성 신호**(§3.3) |
| k6(WS) | `ws_ping`, `ws_sessions`, `ws_session_duration` | `realtime.ws_ping.p95`, `ccu.ws_sessions` | 게임 실시간 대리 지표(방법론 §11.4) |
| Artillery | `aggregate.summaries["http.response_time"].p95` | `latency.p95` | `percentile_side=client`, `source_tool=artillery` |
| Artillery | `aggregate.counters["http.codes.500"]` | `error.5xx_count` | status 분해는 Artillery가 더 직접적 |
| 목 서버 | `process_cpu_seconds`, 주입 지연/에러율 echo | `target.cpu`, `target.injected_*` | 목 서버는 결정성 확보용 — 실제 SUT 아님(D8) |

```python
# collector.py (의사코드)
def collect(run_record: RunRecord) -> ObservationBundle:
    series: list[MetricSeries] = []
    for art in run_record.artifacts:                 # 읽기만. art.path는 검증된 화이트리스트 경로
        adapter = ADAPTERS[art.kind]                 # k6_summary | artillery_report | mock_metrics | frozen_fixture
        series += adapter.to_metric_series(read_only(art.path))
    return ObservationBundle(
        observation_id=new_id(),
        run_id=run_record.run_id,
        source=infer_source(run_record),             # frozen | measured | measured-remote
        metrics=series,                              # 표준 MetricSeries[] (percentile_side 메타 포함)
        logs=collect_log_refs(run_record),           # 참조만(본문 복사 금지)
        traces=collect_trace_refs(run_record),       # 로컬은 보통 빈 배열
        costs=collect_costs(run_record),             # 로컬 목 서버는 cost=0
        anomalies=detect_raw_anomalies(series),      # dropped_iterations>0, status 급증 등 "관측 사실"만
        collected_as_of=now(),                       # 관측 시각(관측 지연 추적용, §7)
    )
```

- **읽기 전용 강제:** collector는 artifact 경로를 화이트리스트(`RunRecord.artifacts`에 등재된 것만) 외에서 읽지 않고, 쓰기 핸들을 갖지 않는다(계약 테스트, [[09-testing-strategy]]).
- `anomalies`는 **해석이 아니라 관측 사실**이다("dropped_iterations=42"). 원인 판정은 interpreter 책임.
- 누락 신호: required signal checklist(방법론 §5 핵심 지표)에 빠진 항목이 있으면 `missing_signals`로 표시 → interpreter가 confidence를 깎고 `inconclusive` 후보로 본다.

---

## 3. Interpreter (`reporter/interpreter.py`, read-only)

### 3.1 판정 원칙

1. **평균 금지.** latency는 p50/p90/p95/p99/max로만 판정한다(방법론 §5, §8-3).
2. **단일 가설.** `TestPlanSpec`에 묶인 **한 가설**만 평가한다(한 실행=한 가설, 방법론 §3). 가설 범위 밖 관측은 verdict가 아니라 "별도 테스트 필요" finding으로 분류.
3. **생성기 병목 우선.** SLO를 보기 전에 부하 생성기 건전성을 먼저 검사한다(§3.3).
4. **SLO는 발명하지 않는다.** 임계값은 `TestPlanSpec.success_criteria`(템플릿 내장 기본값, 방법론 §10.4)에서 읽는다. interpreter는 임계값을 만들지 않고 적용·대조만 한다.
5. **모든 finding에 evidence_refs.** 판정 근거가 된 `MetricSeries`/log/trace/artifact를 가리킨다.

### 3.2 SLO Pass/Fail 판정 축 (평균이 아닌 분포·예산 기준)

| 축 | 입력 키(예) | 판정 규칙(템플릿 기본 threshold 대조) | 비고 |
|---|---|---|---|
| **latency 분포** | `latency.p50/p90/p95/p99/max` | 각 percentile < 목표. **p99·max 누락 금지** | p99=saturation 선행 신호(방법론 §5) |
| **에러율** | `error.req_failed_rate`, `error.4xx_rate`, `error.5xx_rate`, `error.timeout_rate` | 각 < 목표. 4xx(클라)·5xx(서버) **분리** | retry가 실패를 은폐할 수 있음 → 아래 retry 축 병행 |
| **retry** | `error.retry_rate` | retry storm 감지(성공률↑인데 retry↑) | 방법론 §8-13 |
| **success ratio** | `quality.check_pass_rate`, `journey.session_completion_rate` | good/total ≥ 목표. "good" 정의(status+의미 정확성) 명시 | 여정 완주율로 부분 실패 포착 |
| **saturation** | `target.cpu/mem`, `target.queue_depth`, `target.db_connections` | 안전 한계 내. 기울기(soak) ≈ 0 | 목 서버는 대리값 — 실 SUT는 cloud 단계 |
| **recovery time** | ramp-down 후 정상 SLI 복귀까지 시간 | ≤ 목표 | stress/spike 핵심. 미측정 시 "버텼다"만 앎(방법론 §8-11) |
| **error budget / burn rate** | `error.*_rate` + SLO target | burn rate ≤ 임계(§5) | 짧은 테스트는 변동 큼 → 윈도 주의 |

```python
# interpreter.py (의사코드) — 판정 순서가 중요
def interpret(obs: ObservationBundle, plan: TestPlanSpec,
              baseline: RunRecord | None) -> InterpretationSpec:
    refs, findings, bottlenecks, regressions = [], [], [], []
    confidence = 1.0

    # (A) 중단된 실행은 즉시 aborted (kill switch 등, [[04-safety-approval-audit]])
    if plan.run_status == "aborted":
        return aborted_spec(obs, reason=plan.abort_reason, evidence=refs)

    # (B) 1급 게이트: 부하 생성기 병목 분리 (방법론 §8-1)
    gen = check_generator_health(obs)                # dropped_iterations, generator CPU, vus_max 도달
    if gen.saturated:
        bottlenecks.append(gen.as_bottleneck())      # kind="load_generator"
        findings.append(Finding(
            claim="부하 생성기가 포화되어 latency가 부풀려졌을 수 있다 → SLO 판정 보류",
            evidence_refs=[gen.dropped_iter_ref, gen.cpu_ref]))
        # SUT 한계로 오판 금지 → SLO는 보지 않고 inconclusive
        return InterpretationSpec(outcome="inconclusive", findings=findings,
                                  bottlenecks=bottlenecks, regressions=[],
                                  confidence=min(confidence, 0.5), evidence_refs=refs)

    # (C) 단일 가설에 대한 SLO 축별 대조
    verdicts = []
    for crit in plan.success_criteria:               # 템플릿 내장 SLO만 평가
        m = obs.lookup(crit.metric_key)
        if m is None:
            findings.append(Finding(f"{crit.metric_key} 미관측 → 판정 불가", evidence_refs=[]))
            confidence *= 0.7                        # 누락 신호는 신뢰도 하향
            verdicts.append("inconclusive"); continue
        v = compare(m.value, crit.op, crit.threshold)
        findings.append(Finding(
            claim=f"{crit.metric_key}={m.value} {crit.op} {crit.threshold} → {v}",
            metric_key=crit.metric_key, evidence_refs=[m.ref]))
        refs.append(m.ref); verdicts.append(v)

    # (D) error budget / burn rate (§5)
    eb = error_budget_eval(obs, plan)                # burn_rate, 소비율, 윈도 라벨
    findings.append(Finding(f"burn_rate={eb.burn_rate} (윈도 {eb.window_label})",
                            evidence_refs=[eb.ref]))
    verdicts.append("fail" if eb.burn_rate > eb.threshold else "pass")

    # (E) baseline 회귀 (§4)
    if baseline:
        regressions = baseline_compare(obs, baseline)
        refs += [r.ref for r in regressions]

    outcome = aggregate(verdicts)                    # 하나라도 fail → fail; 미관측 다수 → inconclusive
    confidence = adjust_confidence(confidence, obs.source, obs.percentile_side, eb.window_label)
    return InterpretationSpec(
        interpretation_id=new_id(), run_id=obs.run_id, outcome=outcome,
        findings=findings, bottlenecks=bottlenecks, regressions=regressions,
        confidence=confidence, evidence_refs=refs)
```

`outcome` enum = `pass | fail | inconclusive | aborted`([[02-data-model]] InterpretationSpec와 일치).

### 3.3 부하 생성기 병목을 1급 함정으로 분리

방법론 §8-1: 생성기 CPU 100%면 결과 latency가 실제보다 크게 부풀고 RPS도 못 채워(`dropped_iterations`↑) **SUT 한계로 오판**한다. 그래서 interpreter는 **SLO를 보기 전에** 생성기 건전성을 먼저 본다.

| 신호 | 임계(방법론 §8-1) | 의미 | 판정 영향 |
|---|---|---|---|
| `generator.dropped_iterations` | > 0 | 목표 도착률 미달성 = 생성기가 부하를 못 냄 | **inconclusive 후보 1순위** |
| 생성기 CPU | > 80% | latency inflation 시작 구간 | 측정 latency 신뢰 불가 → 보류 |
| 생성기 네트워크 | ≥ ~1Gbit/s 한계 | 대역폭 포화 | 동상 |
| `load.vus_max` 도달 + 미달 RPS | VU 상한에서 막힘 | open model에서 부하 못 올림 | 보류 |

> 로컬 단일 생성기(맥북)는 본질적으로 이 한계에 잘 부딪힌다. 따라서 로컬 MVP에서 **dropped_iterations와 생성기 CPU를 ObservationBundle 필수 신호**로 강제하고, 누락 시 confidence를 깎는다. "SUT가 느리다"는 결론은 생성기 건전성이 통과된 뒤에만 허용한다.

---

## 4. percentile 산출 주의 (client-side vs server-side)

### 4.1 두 percentile은 같지 않다

| 구분 | 측정 위치 | 포함 구간 | 본 시스템 사용 |
|---|---|---|---|
| **client-side**(k6/Artillery) | 부하 생성기 | 클라↔서버 네트워크 + 큐잉 + 서버처리. 생성기 포화 시 inflation | **로컬 MVP 기본** |
| **server-side histogram**(OTel/Prometheus) | SUT 내부 | 서버 처리 시간(네트워크 제외). `histogram_quantile()`로 버킷에서 산출 | cloud 단계 보강 |

- **로컬은 k6/Artillery의 client-side percentile을 사용**한다. 목 서버는 결정성용이라 서버측 histogram을 신뢰 지표로 두지 않는다. 따라서 모든 로컬 리포트의 latency 주장에는 다음 limitation을 **자동 첨부**한다:
  - "percentile은 client-side(생성기 측정)이며 loopback 네트워크·생성기 부하를 포함한다. 서버측 histogram 교차검증은 cloud/OTel 단계 과제다([[10-open-questions-reverify]])."

### 4.2 quantile 평균 금지

방법론 §8-4: quantile 평균은 통계적으로 무의미하다("BAD!"). 인스턴스/구간별 p99를 평균내면 틀린다.

```python
# 금지: 잘못된 집계
p95 = mean([series_a.p95, series_b.p95])              # ❌ quantile 평균

# 허용: 서버측은 버킷 합산 후 분위수 (cloud 단계)
p95 = histogram_quantile(0.95, sum_buckets([hist_a, hist_b]))   # ✅
# 로컬: 도구가 단일 산출한 client-side percentile을 그대로 운반(재집계 금지)
p95 = obs.lookup("latency.p95").value                 # ✅ (재계산/평균 안 함)
```

- interpreter는 **여러 series의 percentile을 절대 평균·합산하지 않는다.** 집계가 필요하면 버킷 합산 + `histogram_quantile`만 허용(cloud 단계). 로컬은 단일 생성기 단일 산출값을 그대로 쓴다.
- 버킷 경계 설계 주의(방법론 §8-5: 오차는 값이 아니라 rank 기준)는 cloud OTel 단계에서 관심 percentile 주변을 촘촘히. 로컬에는 해당 없음.

### 4.3 error budget / burn rate 윈도 (검증: SRE 예시는 30일 기준)

- `burn_rate = observed_error_rate / error_budget`, `error_budget = 1 − SLO_target`.
- burn rate=1 ≈ **30일 SLO 윈도**의 예산을 30일에 정확히 소진하는 속도. SRE Workbook의 multiwindow multi-burn-rate 예시는 **30일 SLO 기준**이다(공식 확인: 5%/1h→36x, 2%/1h→14.4x; [[00-overview]] §5).
- **로컬 함정:** 테스트는 수십 초~수 분이라 짧은 윈도 burn rate 변동이 크다. 그래서 interpreter는 burn rate 값에 **윈도 라벨**(`window_label="test_window_~Xs (30d-SLO 기준 환산)"`)을 붙여 보고하고, 단독 fail 근거로 과신하지 않는다(confidence 하향). 실 운영 SLO 기간·요청량 기반 재산출은 [[10-open-questions-reverify]].

---

## 5. baseline 비교 (회귀 분석)

### 5.1 무엇을 비교하나

`baseline_compare_report` 템플릿(연구 §11 templates, [[03-engine-template-matching]])과 연결된다. 과거 `RunRecord`의 ObservationBundle과 현재를 **동일 SLO 축·동일 metric_window**로 대조한다.

| 비교 키 | 산출 | 판정 |
|---|---|---|
| `latency.p95` / `latency.p99` | Δ(절대·상대%) | 회귀 임계 초과 시 regression |
| `error.5xx_rate`, `error.req_failed_rate` | Δ | 악화 시 regression |
| burn rate / error budget 소비 | Δ | 예산 소비 가속 시 regression |
| throughput / dropped_iterations | Δ | 생성기 조건 동일성 먼저 확인(§3.3) — 다르면 비교 무효 |

```python
def baseline_compare(cur: ObservationBundle, base: RunRecord) -> list[Regression]:
    base_obs = load_observation(base)                # read-only
    assert_comparable(cur, base_obs)                 # 도구/환경/생성기조건 동일성 검사
    out = []
    for key in COMPARE_KEYS:                          # p95/p99/error/burn-rate ...
        c, b = cur.lookup(key), base_obs.lookup(key)
        if c is None or b is None:                    # 한쪽 미관측 → 비교 불가(회귀 아님)
            continue
        delta_abs, delta_pct = c.value - b.value, pct(c.value, b.value)
        if exceeds(key, delta_abs, delta_pct):       # 키별 회귀 임계
            out.append(Regression(metric_key=key, baseline=b.value, current=c.value,
                                  delta_pct=delta_pct, evidence_refs=[c.ref, b.ref]))
    return out
```

- **동일성 가드:** 생성기 조건(VU/도착률/think time)·환경·도구·버전이 다르면 `assert_comparable`이 비교를 **무효**로 막는다(사과↔오렌지 비교 방지). 차이는 회귀가 아니라 "비교 불가" 사유로 보고.
- 회귀는 `InterpretationSpec.regressions`로 흘러가고, composer가 evidence_refs(baseline·current 양쪽)와 함께 표기.

---

## 6. Report Composer (`reporter/composer.py`)

### 6.1 경영 요약 ↔ 기술 상세 분리 + 근거 강제

`FinalReport`([[02-data-model]])를 만든다. **모든 주장은 evidence_refs로 추적 가능**해야 하고, `template_id`·`match_confidence`·`limitations`는 필수다([[00-overview]] §7-5).

| FinalReport 필드 | 청중 | 내용 |
|---|---|---|
| `executive_summary` | 비기술(기획/PM) | pass/fail 한 줄 + CCU 헤드룸 + 핵심 리스크. **수치엔 evidence_ref 표시** |
| `technical_summary` | 엔지니어/QE | percentile 표, 생성기 건전성, 병목, 회귀, 윈도·측정 한계 |
| `slo_results[]` | 공통 | 축별 observed/threshold/verdict/evidence_refs |
| `risk_assessment` | 공통 | 운영 리스크(예: 예상 피크 미달, 급증 미복구) |
| `recommendations[]` | 공통 | 다음 조치(재시험·헤드룸·쿼터, 가설 단위로) |
| `artifacts[]` | 엔지니어 | 그래프/로그/trace/요약 JSON 참조 |
| `limitations[]` | 공통 | client-side percentile·짧은 윈도·synthetic 여부·관측 지연 등 |
| `template_id`, `match_confidence` | 공통 | 어떤 템플릿·얼마나 확신해 매칭했는지(과장 방지) |

```python
def compose(interp: InterpretationSpec, plan: TestPlanSpec,
            match: TemplateMatch) -> FinalReport:
    # 근거 강제 게이트: 모든 finding/슬로결과/회귀가 evidence_refs를 가져야 함
    for claim in iter_claims(interp):
        if not claim.evidence_refs and claim.kind != "missing_signal":
            raise UnsupportedClaimError(claim)        # fail-closed (계약 테스트)

    slo_results = [to_slo_row(f) for f in interp.findings if f.is_slo]
    limitations = build_limitations(interp, plan)     # source/percentile_side/window/관측지연 자동 수집
    return FinalReport(
        report_id=new_id(),
        template_id=match.template_id,
        match_confidence=match.confidence,            # 그대로 표기(반올림 과장 금지)
        executive_summary=render_exec(interp, plan, ccu_first=True),   # 게임=CCU 중심(§7)
        technical_summary=render_tech(interp),
        slo_results=slo_results,
        risk_assessment=render_risk(interp),
        recommendations=render_next_actions(interp),  # 가설 단위
        artifacts=collect_artifact_refs(interp),
        limitations=limitations + run_mode_labels(interp))   # report-only/dry-run/synthetic
```

### 6.2 출력 포맷

| 포맷 | 용도 | 비고 |
|---|---|---|
| **웹 뷰**(JSON → React) | 1차 소비. 경영/기술 탭 분리, evidence_ref 클릭 시 원천 artifact 드릴다운 | [[07-web-ui-api]]가 렌더 |
| **Markdown export** | 공유·PR 첨부 | FinalReport 직렬화 → 결정적 문자열(스냅샷 테스트 대상, §9) |
| **PDF export** | 비기술 배포 | Markdown→PDF 렌더(같은 본문, 표현만 변환) |

- 동일 `FinalReport`에서 세 포맷이 파생되며 **내용은 동일**하다(표현만 다름). 직렬화는 결정적이어야 스냅샷 테스트가 성립한다.

---

## 7. 게임 맥락: CCU 중심 + 관측 지연

### 7.1 지표는 RPS보다 동시접속(CCU) 중심

방법론 §11.1: 게임/실시간은 세션 기반이라 **1차 부하 차원이 CCU**다. composer는 경영 요약을 **CCU 헤드룸 우선**으로 쓴다(RPS는 보조).

| 리포트 위치 | 게임 표현 |
|---|---|
| executive_summary 1줄 | "예상 피크 **CCU N**에서 SLO 충족 / 임계 용량 **CCU M**(헤드룸 X%)" |
| CCU 분모 | `load.vus_max` 또는 `ccu.ws_sessions`(실시간), Active Connections(BaaS, cloud) |
| 실시간 SLO | `realtime.ws_ping.p95`(왕복 지연), `ccu.ws_sessions` 유지 — tick rate/state sync는 미검증 대리(방법론 §11.4) |
| SLO 기본값 | 표준 시나리오 템플릿에 내장(방법론 §10.4, §13). interpreter는 이를 읽기만 함 |

- "이벤트 오픈 대비" 프리셋(Load→Spike→Breakpoint, 방법론 §13)은 **여러 RunRecord**를 만든다. composer는 이를 한 리포트로 묶되 **각 단계=각 가설**로 분리 서술(임계 용량·급증 복구·헤드룸을 별 항목으로).

### 7.2 관측 지연 주의 (클라우드/BaaS 주석)

- **로컬 목 서버:** 관측 즉시(k6 summary는 실행 종료 시 산출). 관측 지연 거의 0.
- **cloud/BaaS(후속):** Cloud Monitoring은 1분 샘플 + **최대 ~4분 반영 지연**(방법론 §12.3). 함의:
  - collector는 metric의 `collected_as_of`/as-of 타임스탬프를 기록하고, interpreter는 **지연된 신호를 실시간 kill-switch 근거로 과신하지 않는다**.
  - burn rate 윈도·abort 임계를 관측 지연만큼 넓혀 설계(방법론 §9, [[10-open-questions-reverify]]).
  - 이 절은 **cloud 단계 주석**이며 로컬 MVP 코드 경로엔 미적용(인터페이스에 `as_of`만 남겨둠).

---

## 8. 반증 가능성 · 신뢰도 · 정직성 라벨

근거 없는 확신을 만들지 않는 것이 Reporter의 1차 책임이다.

| 규칙 | 구현 |
|---|---|
| **confidence 항상 표기** | `InterpretationSpec.confidence` ∈ [0,1]. source(frozen↓)·percentile_side(client↓)·missing_signals(↓)·짧은 윈도(↓)로 하향 |
| **limitations 항상 표기** | composer가 자동 수집(§4.1, §4.3, §7.2). 비면 검증 실패 |
| **"AI가 테스트를 돌렸다" 과장 금지** | 문구 정책: "시스템이 **승인된 템플릿**을 **목 서버 대상**으로 실행했다"로 한정. 자율성·완전성 주장 금지(방법론 정직성 원칙) |
| **report-only / dry-run / synthetic 라벨** | `source=frozen` → "synthetic data, no real load"; dry-run 산출 → "정적 검증만"; `baseline_compare_report`(report-only) → "실행 없음, 회귀 분석만" |
| **반증 가능 서술** | finding은 "관측값 X가 임계 Y를 위반"처럼 **검증 가능한 형태**. "빠르다/안전하다" 같은 마케팅 형용 금지 |
| **inconclusive를 1급 결과로** | pass를 짜내지 않는다. 생성기 병목·신호 누락·표본 부족은 당당히 `inconclusive` |

---

## 9. 테스트 가능성 ([[09-testing-strategy]])

Reporter는 read-only·결정적이라 **테스트 친화성이 가장 높은 모듈**이다(D9). 고정 입력 → 기대 출력으로 망라한다.

### 9.1 Interpreter 골든 테스트 (고정 메트릭 입력 → 기대 판정)

`eval/observations/*.json` 고정 fixture를 입력해 `InterpretationSpec`을 단정한다. 최소 케이스 망라:

| 케이스 | 고정 입력 | 기대 outcome | 검증 포인트 |
|---|---|---|---|
| **percentile 경계 pass** | p95 = 목표−1ms | `pass` | 경계 직전 통과 |
| **percentile 경계 fail** | p95 = 목표+1ms | `fail` | 평균은 통과해도 p95로 실패해야 함(평균 금지 회귀 방지) |
| **p99 누락 판정 보호** | p50 양호 + p99 위반 | `fail` | p99 누락 시 pass로 새지 않음 |
| **생성기 병목** | `dropped_iterations>0` + 양호 latency | `inconclusive`(bottleneck=load_generator) | SLO 보기 전 1급 게이트 작동(§3.3) |
| **생성기 CPU 포화** | generator CPU 92% | `inconclusive` | latency inflation 보류 |
| **error budget burn** | error_rate로 burn_rate>임계 | `fail` | burn rate 축 단독 fail |
| **error budget 경계** | burn_rate=임계 | `pass` | ≤ 경계 동작 |
| **retry storm** | 성공률↑ + retry_rate↑ | `fail`/경고 | retry가 실패 은폐 못 함(§3.2) |
| **신호 누락** | required 신호 일부 결측 | `inconclusive` + confidence↓ | 과신 방지 |
| **aborted** | run_status=aborted | `aborted` | kill switch 경로 |
| **baseline 회귀** | base p95 < cur p95(임계 초과) | regression 존재 | §5 |
| **baseline 비교 불가** | 생성기 조건 상이 | regression 없음 + "비교 불가" finding | 동일성 가드 |

- **결정성:** fixture는 frozen(가짜) 메트릭이므로 Sandbox MVP와 동일 경로 → 같은 골든이 두 단계를 모두 커버.
- **quantile 평균 회귀 테스트:** 여러 series 입력 시 interpreter가 percentile을 평균내지 않음을 단정(§4.2).

### 9.2 Collector 어댑터 테스트

- k6 summary JSON / Artillery report 샘플 → 표준 `MetricSeries`로 정규화되는지(필드 매핑·`percentile_side` 메타).
- 읽기 전용 계약: 화이트리스트 외 경로 읽기·쓰기 핸들 시도 시 실패(계약 테스트).

### 9.3 Composer 직렬화 스냅샷 테스트

- 동일 `InterpretationSpec` → Markdown export 문자열을 **스냅샷**으로 고정(결정적 직렬화).
- **근거 강제 게이트:** evidence_refs 없는 finding을 넣으면 `UnsupportedClaimError`로 거부됨을 단정([[00-overview]] §7-5 계약).
- 필수 필드(`template_id`·`match_confidence`·`limitations`) 누락 시 검증 실패.
- 라벨 테스트: `source=frozen` → synthetic 라벨, report-only 템플릿 → "실행 없음" 라벨이 출력에 포함.

---

## 10. 형제 문서 연결

- 입력 산출(RunRecord/artifacts)·Execution Controller·kill switch: [[05-runners-and-mock-target]]
- 스키마(ObservationBundle/InterpretationSpec/RunRecord/FinalReport): [[02-data-model]]
- template_id·match_confidence 출처, `baseline_compare_report`: [[03-engine-template-matching]]
- audit log·aborted 경로·근거-강제 정책: [[04-safety-approval-audit]]
- 리포트 웹 뷰 렌더·드릴다운: [[07-web-ui-api]]
- 골든/스냅샷/계약 테스트 철학: [[09-testing-strategy]]
- M3(Sandbox 가짜 리포트)·M4(Local 실측 리포트): [[08-milestones-roadmap]]
- client/server percentile 정합·burn rate 윈도·관측 지연 재검증: [[10-open-questions-reverify]]
- 방법론·SLO·함정 원근거: `docs/research/04-test-methodology-slo.md`
