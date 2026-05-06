# clauroboros

[English](./README.md) | [한국어](./README.ko.md)

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
claude plugin install ouroboros@ouroboros-cc
```

로컬 체크아웃에서 (개발):
```sh
claude plugin marketplace add ./clauroboros
claude plugin install ouroboros@ouroboros-cc
```

## 이식한 개념

| 개념        | Claude Code 메커니즘                                                                                |
| ---------- | --------------------------------------------------------------------------------------------------- |
| Seed       | `.ouroboros/seed.json` (정본) + `seed.yaml` (미러). finalize 후 잠김.                                |
| Interview  | `/ooo-interview <목표>` 가 에이전트에게 Socratic 질문을 한 번에 하나씩 하도록 지시                   |
| Evaluate   | `/ooo-evaluate` 가 테스트 러너 자동 감지 후 실행하고, 코드베이스를 검사하여 각 AC 채점                |
| Drift      | `/ooo-drift` 가 자가 평가를 요청 → `0.5*goal + 0.3*constraint + 0.2*ontology`                       |
| Unstuck    | `/ooo-unstuck [persona]` 로 5개 lateral persona 중 하나 활성화 (state.json 에 기록)                   |
| Ralph      | `/ooo-ralph [N]` 단일 커맨드 내부에서 evaluate-and-fix 자가 루프 (하드 cap 포함)                     |
| 하네스     | Karpathy 가이드라인이 `skills/ouroboros/SKILL.md` 에 번들 — 코딩 트리거 시 auto-load                   |

## 슬래시 커맨드

| 커맨드                    | 동작                                                              |
| ------------------------- | ----------------------------------------------------------------- |
| `/ooo-interview <목표>`    | 목표를 seed 로 결정화하기 위한 Socratic 인터뷰 시작                 |
| `/ooo-seed`               | 잠긴 seed (또는 현재 draft) 출력                                   |
| `/ooo-evaluate`           | mechanical 테스트 + 각 AC 를 파일 레벨 증거로 채점                  |
| `/ooo-drift`              | 잠긴 seed 대비 drift 자가 평가                                     |
| `/ooo-unstuck [id]`       | lateral persona 5종 중 하나로 전환                                  |
| `/ooo-ralph [on\|off\|N]` | 하드 cap 포함 인라인 evaluate-and-fix 루프                          |
| `/ooo-status`             | interview / seed / persona / ralph / drift 상태                    |
| `/ooo-reset`              | 세션 상태 초기화 (잠긴 seed 는 보존)                               |

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
    └── ouroboros/
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
