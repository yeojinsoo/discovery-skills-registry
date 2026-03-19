# 테스트 검증 방법론 참조 문서

> 스펙 실행 산출물의 실행 기반 검증(test 커맨드)에서 사용하는 판정 체계, 브라우저 테스트 패턴, 복합 판정 전략, 실패 분류, 집계 알고리즘, 선택자 전략을 정의한다.

**기존 체계와의 관계**: Oracle 3-tier(Strong/Weak/Invalid) 등급 분류는 `rules/spec-test-alignment.md` §2-B가 SOT이며, 본 문서는 이를 변경하지 않고 참조/확장한다.

---

## §1. 6상태 판정 체계

test 커맨드의 개별 검증 항목(R-ID 또는 서브태스크 단위)은 아래 6개 상태 중 하나로 판정한다.

| 상태 | 정의 | 전형적 원인 |
|------|------|------------|
| **PASS** | Oracle 조건이 모두 충족됨. 증거가 Predicate의 모든 핵심 명사를 검증 | Strong Oracle exit 0, AXTree 구조 일치, JS 속성 기대값 일치 |
| **FAIL** | Oracle 조건이 하나 이상 미충족됨. 증거가 Predicate 위반을 명시적으로 보여줌 | exit code != 0, 기대 요소 부재, 응답 상태 4xx/5xx |
| **PARTIAL** | 일부 Oracle은 PASS이나 나머지가 FAIL 또는 UNCERTAIN. 부분 달성 | 복수 Predicate 중 일부만 충족, Strong PASS + Weak FAIL 혼재 |
| **NOT_COVERED** | 해당 R-ID/서브태스크에 매핑되는 증거가 없음. 테스트 자체가 미실행 | 테스트 부재, `--passWithNoTests`로 검증 없이 통과, 스펙 도출 실패 |
| **UNCERTAIN** | 증거는 존재하나 판정 확신도가 임계값 미만. 추가 검증 필요 | LLM 시각 판정 불확실, Weak Oracle만 존재, flaky 테스트 |
| **TIMED_OUT** | Oracle 실행이 제한 시간 내에 완료되지 않음 | 서버 무응답, 브라우저 렌더링 지연, DB 쿼리 타임아웃 |

### validate 어휘 매핑

test 커맨드의 6상태와 validate 커맨드(§VALIDATE Phase S)의 판정 어휘는 다음과 같이 대응한다.

| test 상태 | validate 판정 | 매핑 근거 |
|-----------|--------------|----------|
| PASS | ACHIEVED | 코드 증거 + Oracle PASS + 다관점 합의 |
| FAIL | NOT_ACHIEVED | 미달성, 갭 상세 설명 |
| PARTIAL | PARTIAL | 일부 달성, 미달성 조건 명시 |
| NOT_COVERED | NOT_ACHIEVED (Predicate가 구현 범위 내) / UNCERTAIN (Predicate가 외부 의존 또는 테스트 환경 제약) | Scope IN이고 구현이 기대되는 항목이면 NOT_ACHIEVED, 환경 제약으로 검증 불가하면 UNCERTAIN |
| UNCERTAIN | UNCERTAIN | 증거 부족, 추가 검증 필요 |
| TIMED_OUT | UNCERTAIN | 실행 실패이므로 판정 불가, 재시도 권고 |

---

## §2. 브라우저 테스트 4계층 패턴

Playwright MCP를 통한 브라우저 테스트는 검증 대상의 성격에 따라 4개 계층으로 분류한다. 하위 계층(구조)부터 상위 계층(시각)으로 올라갈수록 Oracle 결정론성이 낮아진다. 가능한 한 하위 계층에서 판정을 완료한다.

### 계층 1: 구조 검증 (Structure)

- **대상**: DOM 요소의 존재, 역할, 텍스트 내용
- **도구**: `browser_snapshot` (AXTree)
- **Oracle 성격**: 결정론적 (Strong)
- **패턴**: AXTree에서 role + name 조합으로 요소 존재를 확인
- **예시**: `AXTree에 role="button", name="저장"인 노드가 존재하는가`
- **한계**: CSS 숨김(visibility:hidden, opacity:0) 요소를 놓칠 수 있음. `aria-hidden="true"`인 요소는 트리에서 제외될 수 있음

### 계층 2: 상태 검증 (State)

- **대상**: DOM 요소의 속성값, 계산된 스타일, 애플리케이션 상태
- **도구**: `browser_evaluate` (JavaScript 평가)
- **Oracle 성격**: 결정론적 (Strong)
- **패턴**: JS 표현식으로 속성값을 추출하여 기대값과 비교
- **예시**: `document.querySelector('.total').textContent === '₩15,000'`
- **한계**: Production 빌드에서 dev 전용 속성 접근 불가. 명시적 test hook이 필요할 수 있음

### 계층 3: 상호작용 검증 (Interaction)

- **대상**: 사용자 동작 시나리오의 정상 완수
- **도구**: `browser_click`, `browser_fill_form`, `browser_press_key` 등
- **Oracle 성격**: 결정론적 (Strong, 단 타이밍 관리 필수)
- **패턴**: click -> fill -> submit -> 결과 요소 확인 (계층 1/2로 결과 판정)
- **예시**: 로그인 폼 입력 -> 제출 -> 대시보드 페이지 AXTree에 사용자명 존재 확인
- **한계**: 비동기 동작에서 대기 조건 선택이 판정 신뢰성을 좌우

### 계층 4: 시각 검증 (Visual)

- **대상**: 렌더링 정합성, 레이아웃, 색상, 시각적 배치
- **도구**: `browser_take_screenshot` + LLM 멀티모달 판정
- **Oracle 성격**: 확률론적 (Weak)
- **패턴**: 스크린샷을 LLM에 전달하여 이진 체크리스트로 판정
- **예시**: "버튼이 화면 우하단에 위치하는가?", "에러 메시지가 빨간색으로 표시되는가?"
- **한계**: 시각 판정 트릴레마 — 결정론성, 포착 범위, 낮은 유지보수를 동시에 최대화 불가. Playwright MCP 환경에서 픽셀 비교(toHaveScreenshot) 구현 불가

### Transient DOM 관측 창

Toast 알림, 로딩 스피너, 에러 메시지 등 일시적으로만 DOM에 존재하는 요소는 별도 패턴을 적용한다.

- **원칙**: 등장 감지 후 즉시 판정. 사라진 뒤의 AXTree/스크린샷은 증거로 무효
- **도구**: `browser_wait_for`로 요소 등장을 감지하되, 감지 즉시 판정을 수행
- **패턴**:
  1. `browser_wait_for`로 대상 요소의 등장을 대기 (타임아웃 설정 필수)
  2. 등장 감지 즉시 `browser_snapshot` 또는 `browser_evaluate`로 내용 캡처
  3. 캡처 결과로 판정 수행
- **실패 시**: 타임아웃 도달 -> TIMED_OUT 판정 (요소 미등장은 FAIL이 아닌 TIMED_OUT)

---

## §3. 계층적 복합 판정 전략

결정론적 판정을 최대화하고 LLM 판정은 예외 경로로만 사용한다. 3개 계층(L1-L3)을 순차적으로 시도하며, 상위 계층은 하위 계층에서 판정 불가능할 때만 진입한다.

> 이 전략은 "해법"이 아닌 "비용 관리 체계"이다. L1에서 해결 가능한 판정을 L2 이상으로 올리는 것은 불필요한 비용이다.

### L1: 결정론적 단언

- **진입 조건**: 모든 검증 항목의 기본 진입점
- **방법**: JS 속성 평가(`browser_evaluate`), AXTree 파싱(`browser_snapshot`), exit code, HTTP 상태 코드
- **판정 규칙**: 기대값과의 일치/불일치가 명확하면 PASS 또는 FAIL 확정. 불명확하면 L2로 이관
- **불명확 기준**: (1) 기대값 자체가 정의되지 않음, (2) 결과가 부분 일치, (3) 비동기로 인한 타이밍 불확실

### L2: LLM 이진 체크리스트

- **진입 조건**: L1에서 판정 불가능한 항목만
- **방법**: 스크린샷 + AXTree + JS 평가 결과를 LLM에 전달하고, 이진(예/아니오) 체크리스트로 구조화하여 판정
- **판정 규칙**: 모든 체크리스트 항목이 "예" → PASS, 하나 이상 "아니오" → FAIL, 확신도 낮으면 L3로 이관
- **체크리스트 설계 원칙**: 각 항목은 관찰 가능한 단일 사실로 제한. "적절한가?" 같은 주관적 질문 금지. "X 요소가 Y 위치에 존재하는가?" 형태만 허용

### L3: UNCERTAIN 반환

- **진입 조건**: L2에서도 확정 불가능한 항목
- **판정 규칙**: UNCERTAIN 상태로 반환. 사람(HITL)의 판단이 필요함을 명시
- **출력**: 수집된 모든 증거(스크린샷, AXTree, JS 결과, LLM 판정 이력)를 증거 번들로 첨부

### 계층 전환 요약

```
모든 항목 ─→ [L1 결정론적 단언]
                │
           ┌────┴────┐
        명확함      불명확
           │         │
        PASS/FAIL   [L2 LLM 이진 체크리스트]
                        │
                   ┌────┴────┐
                고확신      UNCERTAIN
                   │         │
                PASS/FAIL   [L3 UNCERTAIN 반환 + HITL]
```

---

## §4. 실패 유형 3분류

테스트 실패 시, 실패의 근본 원인을 3개 유형으로 분류하여 각기 다른 후속 행동을 트리거한다. 분류 불가 시 HITL로 에스컬레이션한다.

| 유형 | 정의 | 진단 기준 | 후속 행동 |
|------|------|----------|----------|
| **구현 실패** | 코드가 Predicate를 충족하지 못함. 스펙은 정확하나 구현이 미흡 | Oracle FAIL + Predicate 문언이 합리적 + 코드에서 누락/오류 식별 가능 | exec 커맨드 재실행 (해당 서브태스크) |
| **테스트 실패** | 테스트 스펙 자체가 현재 구현과 맞지 않음. 구현은 의도대로이나 테스트가 잘못됨 | Oracle FAIL + 코드가 의도대로 동작 + Predicate/Oracle이 실제 동작과 불일치 | 테스트 스펙 재도출 (스펙 도출기 재실행) |
| **스펙 실패** | PROBLEM.md §3의 해결 조건 자체가 현실과 맞지 않음. 요구사항 변경 필요 | Oracle FAIL + Predicate 달성이 원리적으로 불가능하거나 전제 조건이 변경됨 | PROBLEM.md 갱신 (R-ID 수정 또는 제거) |
| **분류 불가** | 위 3개 유형 중 어디에도 명확히 해당하지 않음 | 복합 원인, 환경 의존적 실패, 재현 불가 | HITL 에스컬레이션 |

### 분류 의사결정 흐름

```
FAIL 발생
  │
  ├─ 코드에서 누락/오류 식별 가능?
  │   └─ Yes → 구현 실패 → exec 재실행
  │
  ├─ 코드는 의도대로 동작하나 Oracle이 불일치?
  │   └─ Yes → 테스트 실패 → 스펙 재도출
  │
  ├─ Predicate 자체가 달성 불가능?
  │   └─ Yes → 스펙 실패 → PROBLEM.md 갱신
  │
  └─ 위 모두 아님 → 분류 불가 → HITL
```

---

## §5. 판정 종합 집계 알고리즘

R-ID 단위로 최종 판정을 산출하는 결정론적 8단계 알고리즘이다. 각 단계는 순서대로 적용하며, 판정이 확정되면 이후 단계를 skip한다.

### 사전 조건

- 각 증거에는 Oracle 등급(Strong/Weak/Invalid)이 태깅되어 있어야 한다. Oracle 등급 분류는 `rules/spec-test-alignment.md` §2-B를 SOT로 따른다.
- Invalid Oracle의 증거는 알고리즘 진입 전에 필터링한다. Invalid 증거가 판정에 영향을 미쳐서는 안 된다.

### 8단계

| 단계 | 조건 | 판정 | 근거 |
|:----:|------|------|------|
| 1 | R-ID에 매핑되는 증거만 필터링 | (필터링) | SPEC-TEST 매트릭스의 R-ID 추적 체인 기준 |
| 2 | 필터링 후 증거 없음 | **NOT_COVERED** | 테스트 미실행. PASS도 FAIL도 아닌 별도 상태 |
| 3 | 각 증거에 Oracle 등급 태깅 | (태깅) | Strong/Weak 분류. Invalid는 사전 제거됨 |
| 4 | Strong Oracle 중 FAIL >= 1 | **FAIL** | Strong FAIL은 범위 내(Scope IN) + non-flaky 확인 후에만 적용. flaky로 확인되면 UNCERTAIN으로 재분류 |
| 5 | 모든 Strong PASS + Weak FAIL 존재 | **PARTIAL** | 핵심 검증은 통과했으나 보강 검증에서 미흡 |
| 6 | 모든 Strong PASS + UNCERTAIN만 존재 | **UNCERTAIN** | 핵심 검증은 통과했으나 확률론적 판정이 미확정 |
| 7 | 모든 증거 PASS (Strong + Weak 전부) | **PASS** | 전체 검증 통과 |
| 8 | FAIL + PASS 혼재 (4-7에 해당하지 않는 잔여 케이스) | **PARTIAL** | 복합 결과 — 세부 내역을 증거와 함께 보고 |

### 단계 4 보충: Strong FAIL 적용 조건

Strong FAIL이 최종 판정을 FAIL로 확정하려면 다음 2개 조건을 모두 충족해야 한다:

1. **Scope IN 확인**: 해당 Oracle이 검증하는 대상이 SPEC.md Scope Boundary의 IN 항목에 속하는가
2. **Non-flaky 확인**: 동일 Oracle을 재실행했을 때 동일 결과가 재현되는가 (최소 1회 재실행 권고)

두 조건 중 하나라도 미충족이면 Strong FAIL을 UNCERTAIN으로 재분류한다.

### TIMED_OUT 처리

TIMED_OUT 증거는 집계 알고리즘에서 UNCERTAIN과 동일하게 취급하되, **상태 라벨은 보존**한다. 최종 판정이 UNCERTAIN일 때 TIMED_OUT 증거가 포함되어 있으면 판정 출력에 `[TIMED_OUT 포함 — 재시도 가능]`을 부기한다. 이를 통해 "재시도로 해결 가능한 TIMED_OUT"과 "본질적 불확실성 UNCERTAIN"을 구분하여 후속 액션을 결정할 수 있다.

---

## §6. 선택자 전략

브라우저 테스트에서 DOM 요소를 식별할 때 사용하는 선택자의 우선순위와 각 전략의 특성을 정의한다.

### 우선순위

| 순위 | 전략 | 방법 | 장점 | 단점 | 사용 시점 |
|:----:|------|------|------|------|----------|
| 1 | **AXTree role + name** | `browser_snapshot`에서 role과 name 조합으로 식별 | 구현 변경에 강건. 접근성 표준 기반. 사용자 관점 식별 | role/name이 고유하지 않을 수 있음. 커스텀 컴포넌트에서 role 누락 가능 | 기본 전략. 모든 요소 식별의 첫 시도 |
| 2 | **텍스트 내용** | 요소의 가시적 텍스트로 식별 | 직관적. 사용자가 보는 것과 동일 | 다국어/동적 텍스트에 취약. 동일 텍스트 복수 존재 가능 | AXTree role+name으로 고유 식별 불가 시 |
| 3 | **CSS 선택자** | `document.querySelector` 등으로 CSS 경로 지정 | 정밀한 요소 지정 가능 | DOM 구조 변경에 취약. 유지보수 비용 높음. 사용자 관점과 괴리 | AXTree/텍스트 모두 불가 시 최후 수단 |

### 선택자 결정 흐름

```
요소 식별 필요
  │
  ├─ AXTree에서 role + name으로 고유 식별 가능?
  │   └─ Yes → AXTree role + name 사용
  │
  ├─ 가시적 텍스트로 고유 식별 가능?
  │   └─ Yes → 텍스트 내용 사용
  │
  └─ 위 모두 불가 → CSS 선택자 사용 (DOM 구조 의존 최소화)
```

### 안티패턴

- **깊은 CSS 경로**: `div > div > div > span.text` 같은 다단계 경로는 DOM 리팩토링에 극도로 취약. 가능하면 `[data-testid="..."]` 같은 test hook 사용
- **인덱스 기반 선택**: `:nth-child(3)` 같은 위치 기반 선택은 요소 순서 변경에 취약
- **동적 ID 의존**: 프레임워크가 생성하는 해시 기반 ID(예: `#react-1a2b3c`)는 빌드마다 변경될 수 있음
