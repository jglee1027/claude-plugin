# Claude Plugins

A collection of Claude Code plugins by Jonggyu Lee.

## Installation

```bash
/plugin marketplace add jglee1027/claude-plugin
```

## Plugins

### gh-workflow

A plugin for GitHub issue management and workflow automation.

```bash
/plugin install gh-workflow@jglee1027
```

#### Skills

**`/create-issue`** — Automatically create GitHub issues

- Without arguments: analyzes `git diff`, `git status`, and recent commits to infer issue type and content
- With arguments: creates an issue based on the given context
- Shows draft title/body for user confirmation before creating
- Title follows conventional format: `type: short description`

```
/create-issue
/create-issue OAuth redirect fails on login page
```

**`/issue-report`** — Open issues priority report

- **High**: `bug` label, runtime crashes, security issues, multiple user reports
- **Medium**: fixes in progress, environment-specific, sub-issues
- **Low**: `enhancement`/`question` labels, unscheduled improvements

```
/issue-report
```

**`/reply-discussion`** — Reply to a GitHub Discussion

- Fetches discussion thread via GraphQL
- Analyzes the question and researches the codebase for evidence
- Drafts reply in Korean for maintainer review
- Translates to the questioner's language and posts after confirmation

```
/reply-discussion 5
/reply-discussion https://github.com/owner/repo/discussions/5
```

**`/resolve-issue`** — Full issue resolution workflow

| Phase | Description |
|-------|-------------|
| 1. Analyze issue | Fetch issue via `gh issue view` or search |
| 2. Check local changes | Check for existing work in progress |
| 3. Plan solution | Analyze codebase and present implementation plan |
| 4. Implement | Create branch and make code changes |
| 5. Test | Run project test suite |
| 6. Commit | Conventional commit after user confirmation |
| 7. Create PR | Create PR with `Closes #issue-number` |

```
/resolve-issue 42
/resolve-issue auth token expiration not handled
```

## Requirements

- [GitHub CLI (`gh`)](https://cli.github.com/) installed and authenticated
- Run inside a git repository

## License

Apache-2.0
