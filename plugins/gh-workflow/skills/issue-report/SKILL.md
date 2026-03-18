---
name: issue-report
description: Open issue list and priority report
---

# /issue-report

Fetch all open issues from the current GitHub repository and classify them by priority.

## Steps

1. Fetch all open issues:
   ```bash
   gh issue list --state open --limit 100 --json number,title,labels,createdAt,author,assignees,comments,milestone
   ```

2. Classify issues into **High / Medium / Low** priority:

   ### High
   - Issues with `bug` label
   - Runtime crashes, data loss, or security-related issues
   - Issues reported by multiple users

   ### Medium
   - Issues with fixes in progress or nearly complete
   - Issues occurring only under specific environments/conditions
   - Work items split into sub-issues

   ### Low
   - `enhancement` label (feature requests)
   - `question` label with no analysis yet
   - Improvement requests with no scheduled timeline

3. Output the report in the following format:

   ```
   ## Open Issues Priority Report

   There are currently **N** open issues.

   ---

   ### High Priority

   | # | Title | Labels | Assignee | Status |
   |---|---|---|---|---|
   | #number | title | labels | @assignee | status summary |

   **Reason**: (explain why this is high priority)

   ---

   ### Medium Priority
   (same format)

   ---

   ### Low Priority
   (same format)

   ---

   ### Summary

   | Priority | Count | Key Action |
   |---|---|---|
   | High | N | recommended action |
   | Medium | N | recommended action |
   | Low | N | recommended action |

   (include recommendation for the most urgent work)
   ```

4. Analyze each issue's comments to determine current progress and reflect it in the Status column

## Notes

- Always use the `gh` CLI to fetch issues
- If not in a git repository, display an error message
- Determine actual progress based on issue body and comments
- Show "Unassigned" for issues with no assignee
