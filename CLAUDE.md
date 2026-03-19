# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Claude Code 플러그인 마켓플레이스 모노레포. 마켓플레이스 이름은 `jglee1027`이며 여러 플러그인을 `plugins/` 하위에 관리한다.

## Structure

```
.claude-plugin/
  plugin.json                          # 루트 매니페스트
  marketplace.json                     # 마켓플레이스 카탈로그 (name: jglee1027)
plugins/
  gh-workflow/                         # GitHub 워크플로우 플러그인
    .claude-plugin/plugin.json
    skills/
      create-issue/SKILL.md            # git diff 분석 또는 컨텍스트 기반 이슈 생성
      issue-report/SKILL.md            # 오픈 이슈 중요도별 한국어 보고서
      reply-discussion/SKILL.md        # GitHub Discussion 분석 후 답변 작성·게시
      resolve-issue/SKILL.md           # 이슈 분석 → 구현 → 테스트 → PR 전체 워크플로우
```

## Skills Architecture

Skills are **procedural Markdown instructions** executed step-by-step by Claude Code. Key frontmatter fields:
- `disable-model-invocation: true` — skill is a rigid procedure, not a prompt for generation
- `allowed-tools` — restricts which tools the skill can use
- `argument-hint` — documents expected arguments (`$ARGUMENTS` is substituted at runtime)

All skills rely on the **`gh` CLI** for GitHub interactions and **`git`** for repository operations.

## Adding a New Plugin

1. `plugins/<plugin-name>/` 디렉토리 생성
2. `plugins/<plugin-name>/.claude-plugin/plugin.json` 매니페스트 작성
3. `plugins/<plugin-name>/skills/<skill-name>/SKILL.md` 스킬 추가
4. 루트 `.claude-plugin/marketplace.json`의 `plugins` 배열에 엔트리 추가

## Conventions

- **Commit format**: conventional commits — `type: short description`
- **Branch naming**: `fix/<issue>-<desc>`, `feat/<issue>-<desc>`, `docs/<issue>-<desc>`
- **PRs**: must include `Closes #<issue-number>` for auto-closing
- **User confirmation**: always required before commits, pushes, and PR creation
- **issue-report output**: always in Korean
- **Plugin naming**: kebab-case, no spaces
