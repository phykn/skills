---
date: 2026-04-11
topic: judge 스킬 — 보이스 우선 + 턴제 교환 구조로 재작성
status: draft
---

# judge 스킬 재설계

## 배경과 문제

`judge` 스킬은 나루호도(변호) / 미츠루기(검사) / 타코(판사) 역할의 서브에이전트를 써서 사용자의 의사결정을 적대적 검토하는 도구다. 원래 의도는 *Ace Attorney* 법정 장면처럼 빠르고 극적이고 재미있어야 하는데, 실제로는 두 가지가 동시에 고장 나 있다.

**문제 1 — 캐릭터 보이스가 죽는다.** SKILL.md에는 캐릭터 성격과 말투 규칙이 정의돼 있지만, 서브에이전트가 실제로 읽는 `role.md`는 `## Steelman` / `## Charitable reading` / `## Key dependencies` 같은 고정 출력 헤더만 명시한다. 그 칸을 채우라고 시키면 LLM은 기계적으로 칸을 메우고 보이스는 증발한다. judge가 stage summary에서 "보이스로 다시 쓰라"는 이중 작업을 떠맡지만, 실제로는 건너뛴다.

**문제 2 — 두 에이전트가 서로의 말을 들은 적이 없다.** 현재 Stage 1·2는 병렬이라 양쪽이 독립적으로 독백한다. 반응할 대상이 없으니 "이의 있습니다!" 같은 순간이 나올 수 없다. Stage 3만 순차인데 그마저 한 라운드에 끝난다.

**부수 문제** — Tier 1/2/3 감사 체계, D1/P1 코드, collusion 체크 등 감사 추적 장치가 지적 작업 공간을 잠식해서 "더 나은 결론에 도달한다"는 본래 목적을 방해한다.

## 해결 방향

1. **보이스가 기본값이 되도록 role.md를 다시 쓴다** — 내용 헤더 템플릿 제거, 캐릭터 성격을 role.md로 이동, 출력은 한 덩어리 산문.
2. **병렬 stage를 순차 4턴 교환으로 바꾼다** — 각 턴은 이전 턴을 읽고 시작해서 실제 대화가 이뤄지게 한다.
3. **감사 추적을 단순화한다** — Tier 표 폐기, 출처 규칙 한 문단으로 축소, 매 턴 직후 즉시 검증.
4. **모든 출력은 쉬운 말로** — 논문체 금지, 일상어 강제.

---

## Section 1 — `role.md` 재작성

서브에이전트가 실제로 읽는 단일 역할 파일. `SIDE=defense | prosecutor` + `TURN=opening | challenge | rebuttal | closing`로 분기한다.

### 구조

```
# role — naruhodo & mitsurugi

## Core principle
너는 양식을 채우는 게 아니다. 네 입장을 네 목소리로 주장한다.
구조는 헤더가 아니라 성격에서 나온다.

## Plain language only
독자가 한 단어라도 멈춰서 해석해야 하면 다시 써라. "구조적 우위",
"기대값을 기울인다" 같은 말 금지. "더 크니까 이긴다", "호랑이 쪽으로
확률이 쏠린다"로 바꿔라. 전문용어는 해당 분야 용어만 허용, 나머지는
일상어로.

## Who you are
- **SIDE=defense → naruhodo**: earnest, scrappy. 궁지에 몰리면 오히려
  불타오른다. 땀 좀 흘리는 건 캐릭터답다.
- **SIDE=prosecutor → mitsurugi**: sharp, arrogant, surgical. 냉정한
  자신감. 소리치지 않는다. 짧은 문장에 날을 세운다.

(SKILL.md의 Character voices 섹션이 여기로 이동한다. SKILL.md에는
한 줄 포인터만 남긴다.)

## The one rule about evidence
네가 내놓는 모든 주장에는 누군가 확인할 수 있는 출처가 붙어야 한다.
URL, 논문 인용, `file:line`, 또는 명시적인 가정에서 끌어낸 추론.
이 넷 중 하나. 출처가 없으면 판사가 기각한다. Tier 같은 건 없다.

## Bash rules (스크립트를 돌릴 때)
- `timeout 60 ...`, 네트워크 금지, 외부 API 금지
- 스크립트 파일을 `<WORK_DIR>/evidence/`에 쓴 다음 돌린다
- 순수 계산, 합성 벤치마크, 반례, 간단한 시뮬레이션만
- 60초 안에 못 끝내면 인용이나 추론으로 돌려라

## Turn instructions

네가 받을 정보: SIDE, TURN, PRIOR TURNS 파일 경로들, WORK_DIR.
이전 턴 파일들을 먼저 읽어라. 그 다음 네 차례에 맞게 말해라.

- **opening**: 첫 턴이다. 사용자 입장을 네 방식으로 세워라. 증거
  1~2개를 들고 나와도 되고 순수 프레이밍이어도 된다. 상대를
  의식하지 마라. 네 핵심 논거가 뭔지 드러내라.
- **challenge**: opening을 읽고 공격한다. 프레이밍의 허점과 반증
  출처를 같이 내민다. 이 턴이 증거 싸움의 중심이다.
- **rebuttal**: opening과 challenge를 읽고 상대의 공격에 하나씩
  답한다 — 반박 / 범위 축소 / 항복 중 하나씩. 필요하면 새 증거
  추가 가능.
- **closing**: 전부 읽고 결정타 또는 "이건 인정한다". **새 증거
  금지.** 이미 나온 것으로 마무리. 인상 정리의 턴이다.

## Output format

한 덩어리 산문으로 써라. 첫 줄에 화자 헤더 하나만 붙인다:
`## naruhodo` 또는 `## mitsurugi`. 그 뒤는 전부 산문. 섹션 헤더
금지. 인용문처럼 말해라. 출처는 문장 안에 자연스럽게 녹이거나
문장 끝에 괄호로.

전체 출력을 `<WORK_DIR>/turn<N>_<name>.md`에 쓴 다음, judge에게는
400자 이하 평이한 요약만 반환해라. 그 요약은 그대로 사용자에게
노출되니까, 네 목소리 그대로 써라.

## Red flags (이러고 있으면 멈추고 다시 써라)
- `## Steelman`, `## Points of attack`, `## Evidence for X` 같은
  내용 헤더를 붙이고 있다
- "The defense argues that..." 같은 중립 해설자 말투를 쓰고 있다
- 말해야 할 때 불릿으로 나열하고 있다
- 주제가 기술적이라고 캐릭터를 놓고 있다
- 문장이 논문처럼 읽힌다

## Target example
(SKILL.md 37–41의 tiger vs lion 예시를 그대로 이식. 이게 "이런
에너지로 쓰라"는 유일한 시범.)
```

### 삭제되는 것
- `## Steelman` / `## Charitable reading` / `## Key dependencies` 템플릿 (stage 1 defense)
- `## Central issue` / `## Points of attack` / `## Questions the judge should resolve` 템플릿 (stage 1 prosecutor)
- `## Evidence for/against` + Tier 1/2/3 서브섹션 (stage 2)
- `## Attacks on defense evidence` + `### Attack N: targets D<n>` 구조화 블록 (stage 3)
- Tier 1/2/3 표 전체

### 살아남는 것
- SIDE 변수 분기
- Bash 안전 규칙
- 파일 쓰기 + 요약 반환 패턴

---

## Section 2 — 4턴 교환 구조

"stage" 개념을 버리고 **기능명이 붙은 4턴**으로 재설계한다. 각 턴은 순차이고, 각 서브에이전트는 이전 턴 파일들을 전부 읽고 시작한다.

| # | 턴 이름 | 화자 | 역할 | 읽는 prior turns |
|---|---|---|---|---|
| 1 | **Opening** | naruhodo | 사용자 입장을 자기 방식으로 세운다. 증거 동반 가능. | 없음 |
| 2 | **Challenge** | mitsurugi | Opening을 공격한다. 프레이밍 허점 + 반증 출처. | Opening |
| 3 | **Rebuttal** | naruhodo | Challenge의 공격에 하나씩 답한다. 필요시 새 증거. | Opening, Challenge |
| 4 | **Closing** | mitsurugi | 결정타 또는 인정. **새 증거 금지.** | 전부 |

그 뒤 taco가 네 파일(`turn1_opening.md` ~ `turn4_closing.md`)을 전부 읽고 평결을 쓴다.

### 왜 mitsurugi가 Closing을 가져가나
- 프로세큐터 캐릭터("냉정한 마지막 한 마디")에 어울린다.
- 검사 측이 공격자 입장이라 클로징이 더 극적이다.
- 방어자가 마지막을 잡으면 "저 그냥 버티기만 했어요"가 되기 쉽다.

### 감사 시점 — 매 턴 직후 즉시

각 턴이 끝나면 judge가 해당 턴 파일을 읽고 출처를 검증한다:

- URL → `WebFetch` 한 번. 404거나 인용 내용과 다르면 해당 주장 기각.
- `file:line` → `Read` 한 번. 파일이 없거나 해당 라인이 인용 내용과 다르면 기각.
- 스크립트 → `Read` 스크립트 + 출력. 주장된 숫자가 스크립트에서 나올 수 없으면 기각 (`fabricated-empirical` — 최악의 실패 모드).
- 일상어 추론은 가정이 명시됐으면 인정, 아니면 기각.
- 출처 자체가 없으면 자동 기각.

기각된 주장은 다음 dispatch의 `PRIOR TURNS` 메모에 "기각됨: <이유>"로 표시해서, 다음 서브에이전트가 그 위에 쌓지 않도록 한다.

### 조기 종료
- **Turn 2 Challenge 후**: mitsurugi가 "공격 없음"이라 선언했거나 모든 공격이 기각됨 → `keep` 즉시 평결, T3·T4 생략.
- **Turn 3 Rebuttal 후**: naruhodo가 모든 공격에 항복(collapse)함 → `reconsider` 즉시 평결, T4 생략.
- 그 외는 T4까지 완주.

---

## Section 3 — SKILL.md 오케스트레이션 수정

### 전역 용어 교체
- `stage` → `turn`
- 파일명 `stageN_<name>.md` → `turn<N>_<name>.md` (1=opening, 2=challenge, 3=rebuttal, 4=closing)
- "Stage summary" → "Turn summary"

### Workflow 섹션 재작성

```
### Stage 0 — Intake  [그대로 유지]

### Turns 1–4 — Adversarial exchange

각 턴은 순차. PRIOR TURNS 파일 경로 리스트를 dispatch에 넘긴다.

Turn 1 (Opening):  SIDE=defense,    inputs: intake
Turn 2 (Challenge): SIDE=prosecutor, inputs: intake, turn1
  → 감사: turn2의 출처 전부 검증
  → 조기종료 체크: 공격 없음 또는 전부 기각 → keep 평결, 스킵
Turn 3 (Rebuttal):  SIDE=defense,    inputs: intake, turn1, turn2
  → 감사: turn3의 새 출처 검증
  → 조기종료 체크: 전부 collapse → reconsider 평결, 스킵
Turn 4 (Closing):   SIDE=prosecutor, inputs: intake, turn1, turn2, turn3
  → 감사: 새 증거 있으면 전부 기각 (규칙 위반)

Turn 4 후: judge가 네 파일을 전부 읽고 verdict.md 작성.
```

### Dispatch template (축약)

```
ROLE FILE: <skill dir>/prompts/role.md
SIDE: defense | prosecutor
TURN: opening | challenge | rebuttal | closing
WORK_DIR: <record_dir>
PRIOR TURNS: <file paths, oldest first, with rejection notes inline if any>

Read your ROLE FILE and the prior turn files. Respond in character for
this turn. Write to <record_dir>/turn<N>_<name>.md. Return a ≤400-char
summary in the user's language — that summary goes straight to the user
with no rewriting, so write it in your voice.
```

### Turn summary (사용자 대면)

각 턴이 끝나면 사용자에게 다음을 노출한다:

1. **서브에이전트가 반환한 ≤400자 요약** — 그대로. 재작성 금지. 이게 보이스 보존의 핵심.
2. **taco의 짧은 해설 한 단락** — "무슨 일이 벌어졌고 다음 턴은 어디로 가는가". 타코 성격(어른스럽고 둘 다 귀엽게 보는) 유지.

### 감사 섹션 (축약, 3줄)

```
매 턴 후, 인용된 각 출처에 대해: URL이면 WebFetch 1회, file:line이면
Read, 스크립트면 Read + 출력 확인. 실패 → 해당 주장 기각, 다음
dispatch의 PRIOR TURNS에 "기각됨: <이유>" 메모. 인용 출처가 없는
주장은 자동 기각.
```

### 삭제되는 것 (SKILL.md에서)
- Tier 1/2/3 감사 상세 규칙
- Collusion check (턴제라 구조적으로 불가)
- D1/P1 넘버링 시스템
- Character voices 본문 (→ role.md로 이동; SKILL.md엔 포인터 한 줄)
- "Judge reads a stage file only when..." 조건부 읽기 가이드 (이제 매 턴 무조건 읽음)

### 살아남는 것
- Stage 0 Intake 프로세스
- Abort conditions
- Tone / Signature moves (Objection/Hold it/Got it) — 오케스트레이터 페이싱에 유용
- Target example — SKILL.md에 유지 (판사의 Turn summary 작성 참고용) **and** role.md에 복제 (서브에이전트의 보이스 모델). 같은 예시가 두 곳에 존재.
- Verdict 두 층 구조 (headline + 감사 추적)
- Terminal summary 포맷
- Plain language 규칙 (SKILL.md Tone 아래 새로 추가)

### Verdict 파일 변경
- "Audit trail" 하위 구조 단순화: 4개 턴 파일로 링크, 기각된 출처만 목록화.
- `D1/P1` 코드 완전 제거.
- 헤드라인 레이어(도메인 언어 제목 + Summary + Why + What you should do)는 그대로.

---

## 마이그레이션 순서

1. `role.md` 재작성 (Section 1) — 기존 파일 전면 교체.
2. `SKILL.md` 수정 (Section 3) — Tier/collusion/D-P 삭제, stage→turn, Workflow 재작성, Dispatch template 교체, Plain language 규칙 추가.
3. 간단한 드라이런으로 검증 — 실제 의사결정 주제 하나로 4턴 완주, 보이스/평이함/출처 검증 모두 관찰.
4. 드라이런 결과에 따라 미세 조정.

## 범위 밖 (이 스펙에서 건드리지 않는 것)
- 다른 skill 파일
- judge의 abort conditions 로직
- Stage 0 Intake 흐름
- Verdict 헤드라인 레이어의 포맷
- `docs/trials/` 디렉토리 위치/이름 규칙

## 리스크와 완화
- **리스크**: 서브에이전트가 여전히 중립 해설자 말투로 빠진다.
  **완화**: Red flags 섹션 + Target example을 role.md에 박음. 두 개가 같이 있어야 부정 강제 + 긍정 모델이 동시에 작동.
- **리스크**: "새 증거 금지" 규칙을 Closing에서 깨뜨린다.
  **완화**: 감사 단계에서 Closing의 새 인용은 자동 기각 — 체크는 기계적.
- **리스크**: 4턴 순차 실행이 체감상 느리다.
  **완화**: 각 턴 직후 Turn summary를 즉시 노출 — 사용자가 진행 상황을 계속 본다.
