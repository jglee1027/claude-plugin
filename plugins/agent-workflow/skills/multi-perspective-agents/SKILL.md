---
name: multi-perspective-agents
description: Use when the user asks for multi-perspective / team-agent analysis of one topic — spawns an Agent Team (TeamCreate + SendMessage) so each perspective runs in its own split pane (one Claude per teammate). Triggers — "다양한 관점", "팀 에이전트", "multi pane", "dev/QA/perf 분리".
---

# 🤝 Multi-Perspective Agents (Agent Team)

> 한 주제를 여러 렌즈로 **동시에** 분석할 때 — `TeamCreate`로 팀을 띄우고 팀원에게 `SendMessage`로 작업을 분배합니다. tmux 분할 창은 Claude Code가 자동으로 관리하므로 `tail -F`를 직접 다룰 필요가 없습니다.

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
| ⬜ | 현재 세션이 tmux 안에 있음 | `echo $TMUX` (비어 있지 않아야 함) |
| ⬜ | 기존 팀 없음 (한 세션 당 한 팀) | `ls ~/.claude/teams/` |

> ⚠️ 위 항목이 안 맞으면 — 일반 `Agent` 병렬 호출로 폴백하고 사용자에게 사유를 한 줄로 알려주세요.

---

## 🚦 전체 흐름

```text
[1] 주제 파악
     ↓
[2] 팀원 후보 추천 → AskUserQuestion (사용자가 명시 안 했을 때만)
     ↓
[3] TeamCreate → 분할 창 자동 분배
     ↓
[4] 팀원 spawn (Agent + team_name + name) — 모두 한 메시지에서 병렬
     ↓
[5] SendMessage로 각자에게 렌즈/지시 전달
     ↓
[6] 팀원이 idle 알림으로 결과 보고 (자동 전달)
     ↓
[7] 결과 합성 → REPORT.md
     ↓
[8] shutdown_request → TeamDelete
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

## 🏗️ Step 2 — 팀 생성

```text
TeamCreate(
  team_name = "<topic-kebab>",         # 예: "pr527-review"
  agent_type = "lead-reviewer",
  description = "한 줄 목적 + 결과물"
)
```

- 팀명은 **kebab-case**, 충돌 시 `-2`, `-3` 접미사.
- `~/.claude/teams/<team-name>/config.json` 과 `~/.claude/tasks/<team-name>/` 가 자동 생성됩니다.

---

## 👥 Step 3 — 팀원 spawn (한 메시지에서 병렬)

선택된 페르소나 N명을 **단일 어시스턴트 메시지** 안에서 `Agent(...)` N번 호출. **반드시 병렬**.

각 호출에 들어가야 할 파라미터:

| 파라미터 | 값 | 비고 |
|:---|:---|:---|
| `subagent_type` | `Explore` (읽기 전용) 또는 `general-purpose` (편집 필요) | 페르소나에 맞춰 |
| `team_name` | Step 2의 팀명 | 필수 |
| `name` | 페르소나 슬러그 (예: `developer`) | SendMessage에서 이 이름으로 호출 |
| `description` | 5단어 이내 | UI 표시 |
| `prompt` | "팀 합류 후 leader의 지시를 기다리라"는 한 줄 안내 | 본 지시는 SendMessage로 |
| `run_in_background` | **불필요** | 팀원은 자동으로 idle/wake |

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
> 1. 저장소 안에 `docs/analysis/` 가 이미 있으면 → `docs/analysis/<team-name>/` (저장소에 산출물을 함께 보관)
> 2. 프로젝트 메모리 / CLAUDE.md 가 별도 위치를 지정하면 → 그 위치
> 3. 그 외에는 → `/tmp/<team-name>/` (임시)
>
> 어떤 경로를 골랐는지 사용자에게 한 줄로 알려주세요 — "산출물: `docs/analysis/<team-name>/`".

예시 (developer):

```text
SendMessage(
  to = "developer",
  summary = "PR#527 코드 품질 리뷰",
  message = "PR opendataloader-pdf#527을 코드 품질·정확성 관점에서 리뷰해주세요. \
  대상: java/.../DocumentProcessor.java:543 근방, CLIMain.processFile. \
  하지 말 것: 코드 수정, 다른 팀원과 직접 의논(이번 라운드는 독립 검토). \
  산출물: docs/analysis/pr527-review/developer.md (Verdict / Strengths / Concerns(file:line) / Risk). \
  끝나면 저에게 150자 요약 + 산출물 경로를 SendMessage로 보내주세요."
)
```

---

## 📊 Step 5 — 진행 관찰 (자동, 폴링 금지)

- 팀원은 turn이 끝날 때마다 **자동으로 idle 알림**을 보냅니다 → 어시스턴트 conversation에 새 turn으로 도착.
- ❌ **하지 말 것:** `ls <artifact-dir>`, `tail`, `cat`, `sleep` — 폴링 컨텍스트만 낭비.
- ✅ 사용자가 직접 다른 팀원 pane으로 이동해 추가 질문하도록 안내:
  ```text
  💡 진행 중에 특정 팀원에게 직접 질문하려면 Shift+Down으로 해당 pane으로 이동하세요.
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
모든 팀원에게:  SendMessage(message = {"type":"shutdown_request","reason":"complete"})
모두 종료되면: TeamDelete()
```

> 정리 안 하면 다음 세션에서 "한 세션 당 한 팀" 제약에 걸립니다.

---

## 📺 분할 창 동작 (직접 조작 금지)

- `TeamCreate` 직후 Claude Code가 **자동으로 tmux 창을 분할**하고 각 팀원 pane을 띄웁니다.
- ❌ `tmux split-window`, `tail -F`, `tmux send-keys` — **이 스킬에서는 직접 호출하지 않습니다.** 자동 관리되는 레이아웃을 깨트립니다.
- ✅ 사용자가 단축키로 탐색:
  - `Shift+Down` — 팀원 pane 순환
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

---

## ⚠️ 흔한 실수

| 실수 | 처방 |
|:---|:---|
| `tail -F` 패널 만들기 | 자동 분할 창이 있음. tail 제거. |
| 팀원 spawn 시 본 지시를 prompt에 넣기 | 부트스트랩만 — 본 지시는 SendMessage로. |
| 팀원 이름을 UUID로 부르기 | 항상 `name`(developer 등)으로. |
| `Agent` 호출에 `run_in_background: true` | 팀 모드에서는 불필요. 자동으로 idle/wake. |
| 합성 전에 정리(TeamDelete) | 팀원 출력 잃음. REPORT.md 작성 후 정리. |
| 5명 이상 추천 | 토큰 비용 폭증. 기본 3–4명. |

---

## 📚 실전 사례

- **PR opendataloader-pdf#527 (2026-05-21)** — dev/test/perf/arch 4관점 → 약 2분에 동일 원인(파일 핸들 누수) 수렴. 사용자에게 한 줄 결론 + 표 + must-fix 1건 + follow-up 8건으로 종합 보고.

---

## 🔗 관련 메모

- [[feedback_tmux_multi_pane_agents]] — tmux 다중 pane 패턴 메모 (Agent Team 변형 우선, 폴백은 `claude -p`).
