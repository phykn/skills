# Refactor 스킬 개정 — 설계 문서

**범위:** `skills/refactor/SKILL.md`을 (1) Feynman 재구성과 MECE 파티션이라는 동등한 두 개의 진단 렌즈를 중심으로 재구성하고, (2) 파일 전반에 누적된 중복 구절을 제거한다.

**비목표:** 스킬의 hand-off 계약 변경 (여전히 plan-only, 실행은 `writing-plans` → `executing-plans`로 위임). Step 4.5 falsification 메커니즘 변경. Finding Format 필드 변경 (`Category` 값 제외).

---

## 동기

현재 스킬의 두 가지 문제:

1. **Feynman 재구성이 유일한 원칙으로 명시되어 있지만, 단위 단위(per-unit) 문제만 잡는다.** Step 3A shallow pass에서 "MECE violations"가 지나가듯 한 번 언급될 뿐, MECE가 일급 렌즈로 승격되지 않는다. 그 결과 구조적 findings — 모듈 간 겹침, 어느 곳에도 없는 관심사, 문제의 분할과 어긋난 폴더 경계 — 은 스킬 안에서 원칙적인 자리를 갖지 못하고 보고에서 누락되는 경향이 있다.

2. **Feynman self-check가 다섯 번 반복된다** (Overview, 전용 섹션, Worked example의 mechanical tests, 서브에이전트 dispatch 템플릿, Failure Modes 표). 이것이 파일 길이와 중복의 가장 큰 원인이다.

사용자가 밝힌 목표: Feynman 방법을 리팩토링에 적용하고, 그 결과로 폴더/코드 구조가 MECE하게 구성되는 것. 둘 다 함께.

## 설계

### 1. 동등한 두 개의 렌즈 (Overview 재작성)

현재의 Overview + "Core Principle" + "Feynman self-check" + "Worked example" 블록을 하나의 Overview로 대체한다:

> 이 스킬은 코드를 두 개의 독립적인 렌즈로 진단한다. 어느 한 쪽이 다른 쪽을 포함하지 않는다; 둘은 서로 다른 findings를 잡는다.
>
> **렌즈 ① — Feynman 재구성 (단위 단위 정당화).**
> 질문: *이 단위가 왜 존재해야 하는가?*
> 잡아내는 것: 존재 이유가 없는 단위 (`dead-code`), 근거 없는 복잡도, 주장하는 invariant를 실제로는 강제하지 않는 `logic`, 목적 없는 작업 (`perf-waste`).
>
> **렌즈 ② — MECE 파티션 (단위 집합의 구조 정당화).**
> 질문: *이 단위들은 문제 공간을 겹침 없이, 빠짐없이 나누는가?*
> 모든 granularity에 적용된다: 폴더, 파일, 함수.
> 잡아내는 것: 불명확한 모듈 경계 (`structure`), 파티션을 반영하지 못하는 이름 (`naming`), 두 칸에 같은 관심사 (`duplication` = overlap), 어느 칸에도 없는 관심사 (`gap`).
>
> 두 렌즈는 한 가지 규칙을 공유한다: **docs, tests, comments에서 의도를 가져오지 말고, 재구성하라.** 재구성이 실패하는 지점이 findings이다.
>
> **Self-check (두 렌즈 모두, 모든 재구성에 적용):** "이 설명이 코드가 *무엇을 하는지*에 대한 평문 번역인가, 아니면 *왜 존재해야 하는지 / 왜 이 파티션인지*에 대한 설명인가? 전자라면 재구성은 실패한 것이며, 그 실패 자체가 finding이다."

Feynman 렌즈의 짧은 worked example(기존 `normalize_email`을 줄인 것)은 유지한다. "Mechanical tests you can apply when unsure" 블록은 삭제 — self-check를 다른 말로 반복할 뿐이다.

### 2. Process 변경 — 렌즈 비대칭이 구조를 결정

핵심 비대칭: **MECE는 전역 뷰가 필요하고, Feynman은 지역 뷰로 충분하다.** MECE 분석을 서브에이전트로 쪼개면 각 서브에이전트가 자기 조각만 보게 되어, 조각 간의 overlap·gap을 탐지할 수 없다. 이 사실이 Step 3A의 구성을 결정한다.

**Step 3A (sweep mode) — "shallow → deep"에서 "MECE pass → Feynman pass"로 개명:**

- **Pass 1 — MECE 렌즈 (main agent만).** 트리 전체를 Glob으로 훑고, 폴더/파일 네이밍과 모듈 경계를 조사한다. 두 가지 산출물을 낸다:
  - MECE findings 직접 생산 (`structure | naming | duplication | gap`) — 이것은 triage 신호가 아니라 일급 findings다.
  - Pass 2를 위한 타깃 리스트. 선정 필터는 기존 그대로 유지 (size, complexity, git churn, random sampling) *+ MECE pass가 "경계가 의심스럽다"고 표시한 파일*.
- **Pass 2 — Feynman 렌즈 (parallel subagents).** 각 서브에이전트가 할당된 타깃에 Feynman 재구성을 수행한다. 서브에이전트는 `dead-code | logic | complexity | perf-waste` findings를 리턴한다. 서브에이전트는 MECE 분석을 하지 않는다 — 전역 뷰가 없기 때문이다.

**Step 3B (targeted mode):** main agent가 좁은 스코프에 두 렌즈를 직접 돌린다. 서브에이전트 없음.

**서브에이전트 dispatch 템플릿:** 한 문장 추가 — "너는 Feynman 렌즈만 돌린다. MECE 분석은 전역 뷰가 필요하므로 main agent가 수행한다. 네 스코프 바깥의 모듈 경계나 파일에 대한 findings는 작성하지 말 것." 템플릿 안의 기존 self-check 문단은 Overview로의 한 줄 참조로 압축한다.

### 3. Finding Format — Category를 렌즈별로 묶고 `gap` 추가

```
Category (렌즈별 그룹):
  Feynman 렌즈: dead-code | logic | complexity | perf-waste
  MECE 렌즈:    structure | naming | duplication | gap
```

`gap`은 신규이며 MECE 렌즈만이 생산할 수 있다 — 의도된 파티션을 재구성했을 때 코드베이스가 어느 곳에서도 다루지 않는 관심사로 드러나는, 빈 칸.

### 4. Step 4.5 falsification — 작은 업데이트

- skip 목록에 `gap` 추가 (`structure | naming | complexity`와 함께) — 빠진 관심사는 실행으로 falsify할 수 없다.
- 독립 문단 "Who runs experiments" 삭제. 그 내용("서브에이전트는 실험을 하지 않는다")은 Step 3A에서 이미 암시된다: 서브에이전트는 Feynman만 돌리고 findings를 반환하며, 실험은 main agent의 단계다.
- 실험 표 주변의 산문을 bullet로 압축. 목표: ~35줄에서 ~20줄로.

### 5. Failure Modes — MECE 행 추가

기존 표에 추가:

| Symptom | Why it is wrong |
|---------|-----------------|
| Sweep 후 `structure`, `duplication`, `gap` findings가 하나도 없음 | MECE 렌즈를 건너뛴 것이다. 실제 sweep 스코프의 코드베이스는 거의 항상 최소 한 개의 경계 이슈가 있다 — MECE findings가 0개라는 것은 main agent가 Feynman만 돌렸다는 의미일 공산이 크다. |

"self-explanatory" 행은 삭제 — self-check의 다섯 번째 재진술이다.

### 6. 중복 정리 대상 (구체 목록)

| 현재 위치 | 처리 |
|---|---|
| Overview L14 "All checks reduce to this one question" | 삭제 — 새 두-렌즈 프레이밍과 모순 |
| "Core Principle" 섹션 | 삭제 — 새 Overview로 병합 |
| "Feynman self-check" 독립 섹션 | 삭제 — 새 Overview로 병합 |
| "Worked example"의 Mechanical tests 1/2/3 | 삭제; 예시 자체는 유지 |
| 서브에이전트 템플릿의 self-check 문단 | 한 줄 참조로 압축 |
| Failure Modes "self-explanatory" 행 | 삭제 |
| Step 4.5 "Who runs experiments" 문단 | 삭제 |
| Step 4.5 산문 | bullet로 압축 |

## 예상 크기

현재: 216줄. 목표: ~150줄. Self-check 언급: 5회 → 1회.

## 리팩토링 제약

- `disable-model-invocation: false`와 `allowed-tools`는 그대로 유지 (tool 변경 없음).
- Finding Format 필드는 `Category` 값을 제외하고 변경 없음.
- Step 번호 (1~6, 그리고 4.5)는 고정 유지 — 다른 스킬의 크로스 레퍼런스가 깨지지 않도록.
- `docs/superpowers/specs/YYYY-MM-DD-refactor-<scope>-design.md` spec 출력 경로 변경 없음.
- 여전히 엄격히 plan-only — 리팩토링 자체의 실행은 하지 않는다.

## 성공 기준

- Overview가 두 렌즈를 ~25줄 이내로 설명하고, self-check를 두 번 이상 언급하지 않는다.
- 신규 독자가 Overview만 보고 어느 렌즈가 어느 Category 값을 잡는지 예측할 수 있다.
- 개정된 스킬을 장난감 코드베이스에 돌렸을 때 MECE finding (`structure`, `naming`, `duplication`, `gap` 중 하나 이상)과 Feynman finding이 각각 최소 한 개씩, 서로 다른 추론 사슬로 생성된다.
- 서브에이전트 dispatch 템플릿이 Feynman 서브에이전트가 자기 스코프 바깥의 경계 수준 finding을 실수로 작성하는 것을 구조적으로 불가능하게 만든다.
