# 09 · 테스트 전략

> [[00-overview]] D9: **"구현 코드 대부분이 자동 테스트 가능"을 1급 설계 원칙으로 채택**(사용자 강조). 본 문서는 그 원칙을 강제하는 테스트 계층·결정성 장치·게이트를 정의한다. 테스트 가능성이 설계를 이끈다(테스트하기 어려우면 설계를 바꾼다).

---

## 1. 테스트를 가능케 하는 3대 결정성 장치

비결정성(LLM·부하·시간)이 테스트를 막는 주범이다. 세 곳에서 결정성을 주입한다.

| 장치 | 위치 | 효과 |
|---|---|---|
| **목 LLM 어댑터** | `llm/`의 fake provider | 의도분류·슬롯채움·매칭을 고정 입출력으로 → 엔진 단위/회귀 테스트 결정적 ([[03-engine-template-matching]]) |
| **번들 목 서버** | `mock-target/` | 지연/에러율/동시성을 시드 기반 주입 → 러너·해석기 출력 예측 가능 ([[05-runners-and-mock-target]]) |
| **고정/가짜 메트릭** | `reporter/collector.py` 입력 | 해석·리포트를 골든 입력→기대 판정으로 ([[06-reporter-observability]]) |

> 원칙: **외부 의존(LLM API·k6 바이너리·시계)은 인터페이스 뒤로** 숨기고 테스트에서 주입한다. `engine`은 `runners`를 import하지 않으므로([[01-architecture]] §4) 제어 평면을 실행 없이 테스트할 수 있다.

---

## 2. 테스트 계층(피라미드)

| 계층 | 대상 | 도구 | 결정성 |
|---|---|---|---|
| **단위** | 정책 분류·clamp·plan_hash·슬롯 검증·매칭 결정·해석 판정·Composer 산출 | pytest / vitest | 목 주입 |
| **계약(contract)** | API ↔ 스키마(OpenAPI) ↔ 프론트 타입, ToolAction 계약 | schemathesis / OpenAPI 검증 | 정적 |
| **통합** | 엔진 파이프라인 end-to-end(제어 평면), 러너 ↔ 목 서버 실제 실행 | pytest + 목 LLM + 목 서버 | 시드 |
| **E2E** | 웹 전체 흐름(프롬프트→슬롯→승인→실행→리포트) | Playwright | 목 백엔드/목 서버 |
| **평가(eval)** | 매칭 품질 회귀(발화 30+) | pytest 파라미터라이즈 | 평가셋 라벨 |

CI는 단위·계약·통합·E2E를 게이트로, eval은 회귀 추세(임계 하회 시 실패)로 운용.

---

## 3. 반드시 자동 테스트로 고정할 불변식 (안전)

[[04-safety-approval-audit]]의 불변식을 **테스트 케이스 표**로 망라한다. 이 표는 회귀의 1차 방어선이다.

| 불변식 | 테스트 |
|---|---|
| 승인 없이는 어떤 `ToolAction`도 실행 안 됨 | Execution Controller에 미승인 ToolAction 투입 → 거부 단정 |
| plan_hash 불일치/변조 → 실행 거부 | 승인 후 ToolAction 변조 → 거부 |
| 위험 요청(prod/10만 RPS/결제) → 기본 금지 | 분류 표 케이스별 단정 |
| 저신뢰 매칭(τ_low 미만) → 실행 거부 | 평가셋 저신뢰 발화 → reject decision |
| 필수 슬롯 누락 → 추정 없이 질문 | 슬롯 누락 입력 → clarification 단정 |
| clamp 상한 초과 → 우회 불가 | 비전문가 입력이 safe_limits 초과 → 거부/승인 |
| 외부 콘텐츠의 지시문 무시 | sanitize에 주입 공격 문자열 → evidence-only 처리 단정 |
| canary 실패 → full run 미승격 | 목 서버 canary 실패 주입 → full 미진입 |
| kill switch 트리거 → 즉시 abort | ceiling 초과 신호 → Aborting 전이 |

---

## 4. 컴포넌트별 테스트 방법(요약)

- **데이터 모델([[02-data-model]])**: 모델 검증·직렬화 라운드트립, plan_hash 결정성, store CRUD, AuditLog append-only.
- **엔진/매칭([[03-engine-template-matching]])**: 목 LLM 고정 출력으로 파이프라인 단위 테스트, 평가셋 회귀(정확도/거부율/명확화율 추세).
- **안전([[04-safety-approval-audit]])**: §3 불변식 표, 정책 규칙 전수 케이스.
- **러너([[05-runners-and-mock-target]])**: Composer 산출은 **골든 파일**(슬롯→스크립트 골격) 스냅샷, 실제 실행은 목 서버 통합, 생성기 병목 감지 단위.
- **리포트([[06-reporter-observability]])**: 고정 메트릭 → 기대 SLO 판정 골든(percentile 경계·error budget·복구시간·생성기 병목 케이스), 리포트 직렬화 스냅샷.
- **UI/API([[07-web-ui-api]])**: 컴포넌트 테스트(목 API), 계약 테스트(OpenAPI), E2E 전체 흐름·차단·만료.

---

## 5. 커버리지·게이트 기준

- 핵심 경로(engine·safety·runners·reporter·models)는 **분기 커버리지 우선** — 숫자 목표보다 "모든 분류/판정 분기에 케이스 존재"를 우선.
- 안전 불변식(§3)은 **100% 케이스 존재**가 머지 게이트.
- eval 정확도·거부율은 baseline 대비 하회 금지(회귀 게이트).
- 새 템플릿 추가 시: 평가셋 발화 추가 + 골든 산출 추가가 PR 체크리스트.

## 6. 안티패턴(금지)

- 테스트를 통과시키려 실제 부하/실서비스 호출 — **금지**(목 서버만).
- LLM 실호출을 단위 테스트에 포함 — **금지**(목 어댑터). 실 어댑터는 별도 수동/계약 스모크.
- 시간·랜덤 직접 사용으로 비결정 테스트 — 시드/주입으로 대체.
- 불변식 테스트를 skip/comment 처리 — **금지**([[00-overview]] §6).
