# clauroboros

[English](./README.md) | [한국어](./README.ko.md) | [日本語](./README.ja.md)

[Ouroboros](https://github.com/Q00/ouroboros) 워크플로우를 Claude Code 플러그인으로
네이티브 이식.

`ouroboros` Python CLI 를 감싼 래퍼가 **아닙니다**. spec-first 루프
(interview → seed → evaluate → drift → unstuck → ralph)를 Claude Code 슬래시
커맨드와 skill 로 직접 구현합니다. 상태는 프로젝트 루트의 `.ouroboros/`에
저장되며, [Karpathy 코딩 하네스](https://github.com/forrestchang/andrej-karpathy-skills/blob/main/CLAUDE.md)
가 skill 에 번들되어 코딩 작업마다 트리거됩니다.

## 설치

원격 git 저장소에서:
```sh
claude plugin marketplace add shseooo/clauroboros
claude plugin install clauroboros@clauroboros-cc
```

로컬 체크아웃에서 (개발):
```sh
claude plugin marketplace add ./clauroboros
claude plugin install clauroboros@clauroboros-cc
```

## 이식한 개념

| 개념        | Claude Code 메커니즘                                                                                |
| ---------- | --------------------------------------------------------------------------------------------------- |
| Seed       | `.ouroboros/seed.json` (정본) + `seed.yaml` (미러). finalize 후 잠김.                                |
| Interview  | `/ooo-interview <목표>` 가 에이전트에게 Socratic 질문을 한 번에 하나씩 하도록 지시                       |
| Evaluate   | `/ooo-evaluate` 가 테스트 러너 자동 감지 후 실행하고, 코드베이스를 검사하여 각 AC 채점                    |
| Drift      | `/ooo-drift` 가 자가 평가를 요청 → `0.5*goal + 0.3*constraint + 0.2*ontology`                           |
| Unstuck    | `/ooo-unstuck [persona]` 로 5개 lateral persona 중 하나 활성화 (state.json 에 기록)                       |
| Ralph      | `/ooo-ralph [N]` 단일 커맨드 내부에서 evaluate-and-fix 자가 루프 (하드 cap 포함)                         |
| 하네스     | Karpathy 가이드라인이 `skills/clauroboros/SKILL.md` 에 번들 — 코딩 트리거 시 auto-load                   |

## 용어 설명

- **Seed** — 불변 스펙: goal + acceptance criteria + constraints + ontology
  + exposed assumptions. 한 번 잠기면 모든 커맨드가 권위 데이터로 취급.
  `seed.json` (정본) + `seed.yaml` (미러) 로 저장.
- **Acceptance Criterion (AC)** — "완료" 의 일부를 정의하는 검증 가능한 단일
  진술. 각 AC 는 mechanical 테스트 또는 코드베이스 검사로 `pass` / `fail` /
  `n/a` 채점됨. seed 가 잠기려면 AC ≥ 5 필요.
- **Ambiguity score** — 인터뷰 중 스펙이 얼마나 모호한지 0–1 척도. seed 는
  ambiguity ≤ 0.2 일 때만 잠김.
- **Constraint** — 솔루션이 지켜야 할 비기능 요건 (성능, 보안, 호환성,
  의존성 정책 등).
- **Ontology** — 프로젝트 고유 어휘를 고정하는 용어 → 정의 맵. 모든 커맨드와
  AC 가 동일 개념을 참조하도록 함.
- **Persona** — 정체 상태 탈출용 lateral-thinking 렌즈 5종: `inverter`,
  `first-principles`, `naive-newcomer`, `adversary`, `architect`.
  `state.json` 에 기록되어 다음 세션에서 이어받을 수 있음.
- **Drift** — 현재 작업과 잠긴 seed 사이의 발산. 가중치
  `0.5*goal + 0.3*constraint + 0.2*ontology`. ≤ 0.30 OK, 초과면 DRIFTED.
- **Ralph** — 하드 턴 cap 을 가진 자가 evaluate-and-fix 루프. 모든 AC 통과
  (CONVERGED) 또는 cap 도달까지 반복.
- **Hard cap (하드 cap)** — `/ooo-ralph` 가 강제 종료되기 전의 최대 반복 횟수
  (기본 8). 폭주 루프 방지용. cap 도달 시 AC 가 수렴되지 않았다면 AC 를
  약화시켜 거짓 성공을 만드는 대신, 정직하게 멈추고 보고.
- **Scope creep (스코프 크리프)** — 잠긴 seed 로부터의 점진적 발산: 추가
  기능, 목표 확장, 용어 재정의, 제약 완화 등. `/ooo-drift` 로 감지. seed 는
  권고가 아니라 경계선 — 정말로 목표가 바뀌었다면 조용히 확장하지 말고
  멈춰서 `/ooo-interview` 재실행.
- **Karpathy 하네스** — 코딩 행동 가이드라인 (먼저 생각, 외과적 변경, 성공
  기준 정의, 가정 노출). SKILL 에 번들되어 모든 코딩 작업에 적용.

## 슬래시 커맨드

| 커맨드                    | 동작                                                              |
| ------------------------- | ----------------------------------------------------------------- |
| `/ooo-interview <목표>`        | 목표를 seed 로 결정화하기 위한 Socratic 인터뷰 시작                 |
| `/ooo-seed`                   | 잠긴 seed (또는 현재 draft) 출력                                   |
| `/ooo-evaluate`               | mechanical 테스트 + 각 AC 를 파일 레벨 증거로 채점                  |
| `/ooo-drift`                  | 잠긴 seed 대비 drift 자가 평가                                     |
| `/ooo-unstuck [id]`           | lateral persona 5종 중 하나로 전환                                  |
| `/ooo-ralph [on\|off\|N]`     | 하드 cap 포함 인라인 evaluate-and-fix 루프                          |
| `/ooo-status`                 | interview / seed / persona / ralph / drift 상태                    |
| `/ooo-reset`                  | 세션 상태 초기화 (잠긴 seed 는 보존)                               |

## 언제 어떤 커맨드를 써야 하나

| 상황                                                       | 커맨드                   |
| ---------------------------------------------------------- | ------------------------ |
| 모호한 목표로 새 기능/작업 시작                            | `/ooo-interview <목표>`       |
| 현재 스펙 (잠긴 seed 또는 draft) 확인                       | `/ooo-seed`                  |
| 의미 있는 작업 단계 완료 — AC 대비 검증                     | `/ooo-evaluate`              |
| 여러 번 수정 후 scope creep 의심                            | `/ooo-drift`                 |
| 막히거나 루프, 저품질 솔루션이 나올 때                      | `/ooo-unstuck [persona]`     |
| 하드 cap 내 자율적으로 모든 AC 통과 시키기                  | `/ooo-ralph [N]`             |
| 루프상 어디인지 잊었을 때                                   | `/ooo-status`                |
| 진행 상태는 비우되 seed 는 유지                             | `/ooo-reset`                 |

### `/ooo-unstuck` persona 선택 가이드

| 막힌 이유                                            | Persona            |
| ---------------------------------------------------- | ------------------ |
| 설계가 과하게 복잡하게 느껴짐                        | `inverter`         |
| 잘못된 토대 위에 패치를 쌓고 있음                    | `first-principles` |
| 낯선 코드베이스로 가정이 숨어 있음                   | `naive-newcomer`   |
| 솔루션이 정말 맞는지 판단이 안 됨                    | `adversary`        |
| 모듈 경계가 잘못된 것 같음                           | `architect`        |

### 루프 운용 가이드

- 사소하지 않은 작업이면 **코드를 쓰기 전에** `/ooo-interview` 부터 실행. seed 를
  미리 결정화하는 비용이 나중에 다시 쓰는 비용보다 훨씬 작음.
- "완료" 선언 전 `/ooo-evaluate` 를 게이트로 사용 — AC 채점 없이 자기 선언 금지.
- `/ooo-drift` 는 주기적으로 (몇 번 수정 후, 리팩터 후) 실행 — 비용 작고 scope
  creep 조기 감지.
- `/ooo-ralph` 는 남은 작업이 mechanical (실패 AC 와 명확한 수정 경로) 일 때
  사용. **AC 가 잘못된 경우엔 안 됨**. AC 가 잘못이면 멈추고 `/ooo-interview`
  로 seed 재작성. 수렴시키려고 AC 약화는 절대 금지.

## Claude Code 하네스 제약

Claude Code 가 노출하지 않는 기능:
- 매 턴 `before_agent_start` system prompt 주입
- 다음 턴으로의 자동 follow-up 메시지

따라서:
- **Karpathy 하네스** 는 SKILL 에 위치 — description 의 코딩 키워드로 auto-load.
  추가로 모든 커맨드 본문에서 명시적으로 참조하여 항상 적용 보장.
- **Persona 지속성** 은 파일 (state.json) 기반만 가능. SKILL 이 세션 시작 시
  state.json 을 다시 읽도록 지시. 턴 자동 감소 후크는 없음.
- **Ralph** 는 여러 턴이 아니라 단일 커맨드 내부에서 cap 까지 자가 루프.

## 상태 파일

```
.ouroboros/
├── seed.json        # 정본 seed (모든 커맨드가 권위 있는 데이터로 읽음)
├── seed.yaml        # 사람 / Ouroboros 호환 미러
├── state.json       # interview / persona / ralph / acGrades / drift
└── interview.jsonl  # append-only 이벤트 로그
```

## 플러그인 파일 구조

```
clauroboros/
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── commands/
│   ├── ooo-interview.md
│   ├── ooo-seed.md
│   ├── ooo-evaluate.md
│   ├── ooo-drift.md
│   ├── ooo-unstuck.md
│   ├── ooo-ralph.md
│   ├── ooo-status.md
│   └── ooo-reset.md
└── skills/
    └── clauroboros/
        └── SKILL.md
```

## 워크플로우 예시

```
1. /ooo-interview "할 일 관리 CLI 만들기"
   → 에이전트가 한 번에 한 질문씩 Socratic 인터뷰 진행
   → 답변마다 .ouroboros/state.json 의 seedDraft 업데이트
   → ambiguity ≤ 0.2 + AC ≥ 5 도달 시 seed.json + seed.yaml 잠금

2. (코드 작성)

3. /ooo-evaluate
   → npm test / pytest / make test / cargo test / go test 자동 감지 + 실행
   → 각 AC 별로 코드베이스 검사 후 verdict (pass/fail/n-a) 와 증거 기록
   → pass / fail 집계 출력

4. /ooo-drift
   → goal / constraint / ontology divergence 자가 평가
   → weighted ≤ 0.30 이면 OK, 초과면 DRIFTED

5. (막혔을 때) /ooo-unstuck adversary
   → 현재 솔루션을 깨려는 persona 활성화

6. /ooo-ralph 8
   → cap=8 내에서 evaluate → 실패 AC 수정 자가 반복
   → 모두 pass 시 CONVERGED 출력 후 자동 정지
```

## 메모

- 런타임 의존성 없음. 에이전트가 Claude Code 의 내장 `Read`/`Write`/`Edit`/
  `Bash`/`Grep`/`Glob` 툴로 `.ouroboros/` 상태 파일을 직접 관리.
- seed YAML 은 SKILL.md 에 명시된 스키마대로 에이전트가 손으로 작성. 항상
  double-quote 스칼라, 고정 키 순서. 읽기는 JSON 사이드카에서 하므로
  quoting 버그가 루프를 깨지 않음.
- Ralph 는 사용자가 지정한 N (기본 8) 으로 하드 cap. AC 를 약화시켜 "수렴"
  시키는 일은 절대 없음 — 정직하게 통과 못 시키면 멈추고 보고.
