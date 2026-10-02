---
name: multi-perspective-agents
description: Use when the user asks for multi-perspective / team-agent analysis of one topic — spawns named teammates into the session's Agent Team (Agent + name, then SendMessage) so each perspective runs as its own Claude, shown in tmux split panes when teammateMode is tmux/auto. Triggers — "다양한 관점", "팀 에이전트", "multi pane", "dev/QA/perf 분리".
---

# 🤝 Multi-Perspective Agents (Agent Team)

> 한 주제를 여러 렌즈로 **동시에** 분석할 때 — 세션의 Agent Team에 팀원을 `name`으로 spawn하고 `SendMessage`로 작업을 분배합니다. 분할 창은 Claude Code가 자동으로 관리하므로 `tail -F`를 직접 다룰 필요가 없습니다.
>
> ℹ️ **버전 메모 (Claude Code 2.1.x 이후):** `TeamCreate`/`TeamDelete` 도구는 없습니다. 팀은 **세션마다 하나씩 암묵적으로** 생기고, `Agent`의 `team_name` 인자는 무시됩니다. 팀원이 **tmux 분할 창**으로 뜰지 **메인 터미널 안(in-process)** 에서 돌지는 `teammateMode` 설정이 정합니다. 기본값은 `in-process`라서 설정하지 않으면 분할 창이 생기지 않습니다.

---

## 🎯 언제 쓰나

| 상황 | 도구 |
|:---|:---|
| ✅ PR/이슈/문서 한 건을 dev·QA·perf·arch 같은 **여러 렌즈로 교차 검토** | **이 스킬** |
| ✅ 사용자가 "팀 에이전트", "다양한 관점", "multi pane" 같은 용어를 직접 언급 | **이 스킬** |
| ❌ 단일 관점·단순 조사 | 단일 `Agent` 호출 |
| ❌ 순차적 작업(앞 단계 결과를 다음 단계가 이어받음) | 단일 세션 또는 `Agent` 직렬 호출 |
| ❌ 동일 파일을 동시에 편집해야 함 | 단일 세션 |
| ❌ 토큰 예산이 빠듯할 때 | 단일 `Agent` (teammate마다 별도 컨텍스트) |

---

## 📋 사전 점검 (시작 전에)

| □ | 항목 | 명령 |
|:-:|:---|:---|
| ⬜ | Agent Teams 기능 활성화 | 환경변수 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` |
| ⬜ | 분할 창 표시 모드 | `~/.claude/settings.json` 의 `"teammateMode"` 가 `"tmux"` 또는 `"auto"` (미설정 = `in-process`) |
| ⬜ | 현재 세션이 tmux 안에 있음 (분할 창을 쓸 때) | `echo $TMUX` (비어 있지 않아야 함) |
| ⬜ | 세션 팀의 실제 백엔드 | `~/.claude/teams/session-<id앞8자리>/config.json` 의 `backendType` (`in-process`면 이번 세션은 분할 창 없음) |

판정 규칙:

- **`teammateMode` 미설정 또는 `in-process`** → 팀 기능은 정상 동작하지만 **분할 창은 생기지 않습니다.** 그대로 진행하되 사용자에게 한 줄로 알립니다:
  ```text
  ℹ️ 팀원은 메인 터미널 안(in-process)에서 실행됩니다. 분할 창을 원하면 settings.json에 "teammateMode": "tmux"를 넣고 세션을 다시 시작하세요(또는 `claude --teammate-mode tmux`).
  ```
- **설정은 `tmux`/`auto`인데 `backendType`이 `in-process`** → 설정이 세션 시작 뒤에 바뀐 경우입니다. 표시 모드는 시작할 때 정해지므로 **세션 재시작**을 안내합니다.
- **분할 창 미지원 터미널** — VS Code 통합 터미널, Windows Terminal, Ghostty는 자체 분할을 지원하지 않습니다. 그 안에서 tmux를 실행해야 합니다.
- **`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` 꺼짐** → 일반 `Agent` 병렬 호출로 폴백하고 사유를 한 줄로 알립니다.
- `settings.json` 은 사용자가 직접 바꾸게 안내합니다(스킬이 임의로 수정하지 않음).

---

## 🚦 전체 흐름

```text
[1] 주제 파악
     ↓
[2] 팀원 후보 추천 → AskUserQuestion (사용자가 명시 안 했을 때만)
     ↓
[3] 산출물 디렉터리 준비 (팀은 세션에 이미 있음 — 생성 단계 없음)
     ↓
[4] 팀원 spawn (Agent + name) — 모두 한 메시지에서 병렬 (teammateMode=tmux면 분할 창 자동)
     ↓
[5] SendMessage로 각자에게 렌즈/지시 전달
     ↓
[6] 팀원이 idle 알림으로 결과 보고 (자동 전달)
     ↓
[7] 결과 합성 → REPORT.md
     ↓
[8] 팀원마다 shutdown_request
```

---

## 🧑‍🤝‍🧑 Step 1 — 팀원 추천 & 선택

**사용자가 페르소나를 명시하지 않았다면**, 주제를 분류한 뒤 후보를 추천합니다.

### 페르소나 카탈로그 (참고용)

| 페르소나 | 렌즈 | 대표 산출물 |
|:---|:---|:---|
| 🔧 **developer** | 코드 품질·정확성·방어 코딩 | APPROVE/REQUEST_CHANGES + concerns |
| 🧪 **tester** | 테스트 커버리지·엣지 케이스·회귀 위험 | 누락 케이스 목록 |
| ⚡ **performance** | 성능·동시성·자원 누수 | 핫스팟·리스크 표 |
| 🏛 **architecture** | 예외 설계·API 계약·일관성·문서화 | 비대칭/장기 유지보수 이슈 |
| 🔐 **security** | 인증·검증·CORS·취약점 | OWASP/PoC |
| 🎨 **ux** | UX·메시지·접근성·로케일 | 문구·플로우 코멘트 |
| 📚 **docs** | 문서·CHANGELOG·README 일관성 | 누락/모순 문서 |
| 🧭 **product** | 사용자 시나리오·범위·우선순위 | scope creep / 누락 시나리오 |
| 🌐 **i18n** | 메시지 외부화·다국어 | 하드코딩 문자열 |
| 📦 **release** | 호환성·deprecation·배포 영향 | 마이그레이션 노트 |

### 주제별 기본 추천 (제안만, 사용자가 수정 가능)

| 주제 유형 | 기본 4인 |
|:---|:---|
| **PR/코드 리뷰** | developer · tester · performance · architecture |
| **버그·장애 가설** | developer · performance · tester · ops |
| **신규 기능 설계** | product · architecture · security · ux |
| **문서/README 리뷰** | docs · developer · product |
| **릴리스 점검** | release · tester · security · docs |

### AskUserQuestion 형식

후보가 정해지면 **반드시** `AskUserQuestion` 으로 확인합니다. (자유 추가/변경 가능)

```text
질문: 어떤 관점으로 팀을 구성할까요?
헤더: 팀 구성
옵션 (multiSelect = true, 최대 4명 추천):
  ▸ developer (코드 품질·정확성)
    tester (테스트·엣지 케이스)
    performance (성능·동시성)
    architecture (예외 설계·일관성)
    security (취약점)
    ux (문구·접근성)
    docs (문서 일관성)
```

> ✋ 사용자가 페르소나를 **이미 명시**했다면 이 단계는 건너뜁니다.
> 👥 **최대 4명까지** — 그 이상은 토큰 비용 대비 효용이 빠르게 떨어집니다.

---

## 🏗️ Step 2 — 팀 (생성 단계 없음)

- 팀은 세션 시작 때 `~/.claude/teams/session-<id>/` 로 **자동 생성**됩니다. 리더는 현재 세션(`team-lead`)입니다.
- 이번 작업의 이름(**`<topic-kebab>`**, 예: `pr527-review`)은 산출물 디렉터리 이름으로만 씁니다. 충돌 시 `-2`, `-3` 접미사.
- **구버전 호환:** `ToolSearch("select:TeamCreate")` 로 `TeamCreate` 가 발견되는 구버전이면, 예전처럼 `TeamCreate(team_name=..., agent_type="lead-reviewer", description=...)` 로 팀을 만든 뒤 진행하고 Step 7에서 `TeamDelete()` 를 호출합니다. 발견되지 않으면 이 단계를 건너뜁니다.

---

## 👥 Step 3 — 팀원 spawn (한 메시지에서 병렬)

선택된 페르소나 N명을 **단일 어시스턴트 메시지** 안에서 `Agent(...)` N번 호출. **반드시 병렬**.

각 호출에 들어가야 할 파라미터:

| 파라미터 | 값 | 비고 |
|:---|:---|:---|
| `subagent_type` | `Explore` (읽기 전용) 또는 `general-purpose` (편집 필요) | 페르소나에 맞춰 |
| `name` | 페르소나 슬러그 (예: `developer`) | **필수.** 이름이 있어야 팀원으로 합류하고 SendMessage로 호출 가능 |
| `description` | 5단어 이내 | UI 표시 |
| `prompt` | "팀 합류 후 leader의 지시를 기다리라"는 한 줄 안내 | 본 지시는 SendMessage로 |
| `team_name` | **넣지 않음** | 현재 버전에서는 무시됨 (구버전에서만 Step 2의 팀명) |
| `run_in_background` | **넣지 않음** | 팀원은 자동으로 idle/wake |

> 💡 spawn 단계에서는 "팀에 합류하고 대기"라는 짧은 부트스트랩 프롬프트만 줍니다. 실제 작업 지시는 SendMessage로 — 그래야 분할 창에서 사용자가 각 팀원과 직접 대화할 여지가 남습니다.

---

## 📬 Step 4 — 작업 지시 (SendMessage)

팀원이 합류하면 각자에게 `SendMessage`로 렌즈와 지시를 전달합니다.

각 메시지에 포함할 항목:

1. **주제 + 렌즈** — 무엇을 어떤 시각으로 볼지
2. **읽어야 할 파일/PR 번호** — 구체 경로
3. **반드시 피할 행위** — 예: "코드는 수정하지 마", "다른 팀원 결과를 미리 보지 마"
4. **산출물 형식** — 섹션 헤더 고정 (Verdict · Strengths · Concerns · Risk level)
5. **결과 보고 방법** — "끝나면 leader에게 SendMessage로 150자 요약 + 풀 리포트는 `<artifact-dir>/<name>.md`에 저장"

> 📁 **산출물 디렉터리(`<artifact-dir>`) 선택 규칙** — 팀 spawn 전에 결정해 모든 팀원에게 일관되게 전달.
> 1. **기본: `<project>/.claude/agent-team/<team-name>/`** — 프로젝트 곁에 두어 찾기 쉽고, `/tmp` 처럼 휘발되지 않아 PR 근거 등 산출물 보존에 적합. git 추적은 `.gitignore` 가 아니라 `.git/info/exclude` 로 차단한다(아래 참조).
> 2. 프로젝트 메모리 / CLAUDE.md 가 별도 위치를 지정하면 → 그 위치
> 3. 사용자가 산출물을 **git에 함께 커밋**하겠다고 명시하면 → 저장소 안 추적 경로 (예: `docs/analysis/<team-name>/`)
> 4. 그 외 폴백 → `/tmp/<team-name>/` (휘발성 임시)
>
> ⚙️ **디렉터리 준비 (기본 경로 선택 시 필수)** — 팀원에게 경로를 알리기 전에 leader가 직접 처리:
> 1. `mkdir -p <project>/.claude/agent-team/<team-name>` 한 줄이면 충분 (상위 `.claude/`·`agent-team/` 도 함께 생성됨). 이미 있으면 그대로 사용.
> 2. **git 저장소가 아니면 이 단계를 건너뛴다** (`git rev-parse --git-dir` 실패 시).
> 3. **git 비추적 보장 — `.gitignore` 는 절대 건드리지 않는다.** `.gitignore` 는 저장소에 커밋되어 다른 개발자에게도 퍼지므로, 개인 산출물 규칙을 넣으면 오염이다. 대신 커밋되지 않는 로컬 전용 파일 `.git/info/exclude` 를 쓴다:
>    ```bash
>    git check-ignore -q .claude/agent-team/.gitkeep \
>      || echo '.claude/agent-team/' >> "$(git rev-parse --git-dir)/info/exclude"
>    ```
>    이미 무시되고 있으면(예: `.claude/` 전체가 ignore) 아무것도 하지 않는다. 중복 추가를 막기 위해 반드시 `git check-ignore` 로 먼저 확인할 것.
>
> 어떤 경로를 골랐는지 사용자에게 한 줄로 알려주세요 — "산출물: `.claude/agent-team/<team-name>/`".

예시 (developer):

```text
SendMessage(
  to = "developer",
  summary = "PR#527 코드 품질 리뷰",
  message = "PR opendataloader-pdf#527을 코드 품질·정확성 관점에서 리뷰해주세요. \
  대상: java/.../DocumentProcessor.java:543 근방, CLIMain.processFile. \
  하지 말 것: 코드 수정, 다른 팀원과 직접 의논(이번 라운드는 독립 검토). \
  산출물: .claude/agent-team/pr527-review/developer.md (Verdict / Strengths / Concerns(file:line) / Risk). \
  끝나면 저에게 150자 요약 + 산출물 경로를 SendMessage로 보내주세요."
)
```

---

## 📊 Step 5 — 진행 관찰 (자동, 폴링 금지)

- 팀원은 turn이 끝날 때마다 **자동으로 idle 알림**을 보냅니다 → 어시스턴트 conversation에 새 turn으로 도착.
- ❌ **하지 말 것:** `ls <artifact-dir>`, `tail`, `cat`, `sleep` — 폴링 컨텍스트만 낭비.
- ✅ 사용자가 직접 팀원에게 추가 질문할 수 있도록 표시 모드에 맞춰 안내:
  ```text
  💡 tmux 분할 창: 해당 팀원 pane을 클릭(또는 tmux prefix로 이동)해 직접 질문하세요.
  💡 in-process: 에이전트 패널에서 ↑↓로 팀원 선택 → Enter로 대화 보기·메시지 보내기 → Esc로 닫기.
  ```

---

## 🧩 Step 6 — 결과 합성

모든 팀원으로부터 보고 수신 후:

1. 각 팀원의 `<name>.md` 를 `Read`로 흡수.
2. `<artifact-dir>/REPORT.md` 작성 — 다음 구조:
   - **한 줄 결론**
   - **관점별 요약표** (페르소나 · 판정 · 리스크 · 핵심 지적)
   - **Must-fix (머지/배포 전 필수)** — file:line 인용
   - **Should (후속 이슈로 분리 권장)** — 표
   - **강점 / 약점**
   - **종합 권고**
3. 사용자에게 한 줄 결론 + 표 + 리포트 경로를 응답.

---

## 🧹 Step 7 — 정리

```text
모든 팀원에게:  SendMessage(to = "<name>", message = {"type":"shutdown_request","reason":"complete"})
(구버전에서 TeamCreate를 썼다면) 모두 종료된 뒤: TeamDelete()
```

> 현재 버전에서는 세션 팀이 세션과 함께 끝나므로 `TeamDelete` 가 필요 없습니다. `~/.claude/teams/session-*` 에 지난 세션 폴더가 쌓일 수 있으나, 삭제는 사용자 확인 후에만 합니다.

---

## 📺 분할 창 동작 (직접 조작 금지)

- `teammateMode` 가 `tmux`(또는 tmux 안에서 `auto`)면, 팀원을 spawn할 때 Claude Code가 **자동으로 tmux 창을 분할**해 각 팀원 pane을 띄웁니다.
- `in-process`(기본값)면 분할 창 없이 메인 터미널의 에이전트 패널에 팀원이 표시됩니다.
- ❌ `tmux split-window`, `tail -F`, `tmux send-keys` — **이 스킬에서는 직접 호출하지 않습니다.** 자동 관리되는 레이아웃을 깨트립니다.
- ✅ 탐색:
  - tmux 분할 창: 팀원 pane 클릭 또는 tmux prefix 이동
  - in-process: `↑↓` 팀원 선택 · `Enter` 대화 보기/메시지 · `Esc` 닫기 · `x` 팀원 중지
  - `Ctrl+T` — 작업 목록 토글 (tmux prefix 충돌 시 미동작 — 정상)

---

## 🚫 빨간 깃발 (멈춰야 할 신호)

| 생각 | 대신 할 것 |
|:---|:---|
| "tail로 로그를 들여다볼까" | ❌ Agent Team은 메시지 기반. tail 불필요. |
| "팀원을 한 명씩 순서대로 spawn" | ❌ 한 메시지에 모든 Agent 호출 → 진짜 병렬. |
| "팀원이 idle인데 죽은 건가" | ❌ idle은 정상 상태. 메시지 보내면 깨어남. |
| "팀원 결과 파일을 미리 cat" | ❌ idle 알림이 자동 도착. 폴링 금지. |
| "사용자가 팀원을 안 정했으니 그냥 4명 박자" | ❌ 먼저 AskUserQuestion으로 확인. |
| "주제가 단순한데 일단 팀부터" | ❌ 한 관점이면 단일 `Agent` 호출. |
| "TeamCreate가 없으니 팀 기능이 안 된다" | ❌ 현재 버전은 세션 팀이 자동 생성됨. `name`으로 spawn하면 팀원이 됨. |
| "분할 창이 안 뜨니 tmux를 직접 나누자" | ❌ `teammateMode` 설정 문제. 설정 후 세션 재시작을 안내. |

---

## ⚠️ 흔한 실수

| 실수 | 처방 |
|:---|:---|
| `tail -F` 패널 만들기 | 자동 분할 창이 있음. tail 제거. |
| 팀원 spawn 시 본 지시를 prompt에 넣기 | 부트스트랩만 — 본 지시는 SendMessage로. |
| 팀원 이름을 UUID로 부르기 | 항상 `name`(developer 등)으로. |
| `Agent` 호출에 `run_in_background: true` | 넣지 않음. 팀원은 자동으로 idle/wake. |
| `Agent` 호출에 `team_name` 지정 | 현재 버전에서는 무시됨. `name`만 지정. |
| `teammateMode` 미설정인 채로 분할 창 기대 | 기본값은 `in-process`. `"teammateMode": "tmux"` 설정 후 재시작. |
| 합성 전에 팀원 종료(shutdown_request) | 후속 질문 기회를 잃음. REPORT.md 작성 후 종료. |
| 5명 이상 추천 | 토큰 비용 폭증. 기본 3–4명. |

---

## 📚 실전 사례

- **PR opendataloader-pdf#527 (2026-05-21)** — dev/test/perf/arch 4관점 → 약 2분에 동일 원인(파일 핸들 누수) 수렴. 사용자에게 한 줄 결론 + 표 + must-fix 1건 + follow-up 8건으로 종합 보고.
- **텀시트 재검토 (2026-10-01, Claude Code 2.1.286)** — 4관점 팀원은 정상 동작했으나 분할 창이 뜨지 않음. 원인: `TeamCreate` 부재(세션 팀 자동 생성) + `teammateMode` 미설정(기본 `in-process`, 팀 config의 `backendType: in-process`). 이 경험으로 사전 점검·Step 2·탐색 안내를 갱신.

---

## 🔗 관련 메모

- [[feedback_tmux_multi_pane_agents]] — tmux 다중 pane 패턴 메모 (Agent Team 변형 우선, 폴백은 `claude -p`).
