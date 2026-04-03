---
name: reply-discussion
description: Reply to a GitHub Discussion by analyzing the codebase, drafting in Korean for review, then posting in the questioner's language
argument-hint: "[discussion number or URL]"
---

# Reply to GitHub Discussion

You are replying to a GitHub Discussion for the current repository. Follow each phase carefully.

## Phase 0: Triage (when no argument is provided)

If `$ARGUMENTS` is empty (no discussion number or URL given), run this triage phase before proceeding:

1. Fetch the repo owner/name: `gh repo view --json owner,name`
2. Fetch recent discussions (last 15, ordered by creation date) with comment counts:
   ```bash
   gh api graphql -f query='
   query($owner: String!, $repo: String!) {
     repository(owner: $owner, name: $repo) {
       discussions(first: 15, orderBy: {field: CREATED_AT, direction: DESC}) {
         nodes {
           number
           title
           author { login }
           createdAt
           answerChosenAt
           comments { totalCount }
           category { name }
         }
       }
     }
   }' -F owner='{owner}' -F repo='{repo}'
   ```
3. For discussions that might need replies (Q&A with 0 comments, unanswered, etc.), fetch the **last comment** of each to determine conversation status:
   ```bash
   gh api graphql -f query='...' # use comments(last: 1) with replies(last: 1)
   ```
4. Analyze each discussion and present a **triage table in Korean** with:
   - Discussion number and title
   - Author and language
   - Comment count
   - **Status**: one of:
     - ✅ **답변 완료** — maintainer answered and questioner confirmed/thanked
     - ✅ **대화 종료** — conversation naturally concluded
     - ⚠️ **후속 필요** — someone asked a follow-up or raised a counterpoint with no maintainer response
     - 🔴 **미답변** — no comments at all on a Q&A discussion
     - 💤 **토론 중단** — long period of inactivity with no resolution
   - **Priority** (use emoji for quick visual scanning):
     - 🔴 **P1** — 즉시 답변 필요 (미답변 Q&A, 긴급 후속 필요)
     - 🟡 **P2** — 답변 권장 (후속 질문 있음, 불완전한 답변)
     - 🟢 **P3** — 여유 있음 (토론 중단, 낮은 긴급도)
     - ⚪ **—** — 답변 불필요 (완료/종료)
   - Brief reason for the status/priority assessment
5. Let the user pick which discussion to reply to (or confirm none need replies)

After the user selects a discussion, proceed to Phase 1 with that number.

---

## Phase 1: Fetch the Discussion

Check the argument: `$ARGUMENTS`

1. Extract the discussion number from the argument (number or URL)
2. Fetch the discussion content and all comments using GraphQL:
   ```bash
   gh api graphql -f query='
   query($owner: String!, $repo: String!, $number: Int!) {
     repository(owner: $owner, name: $repo) {
       discussion(number: $number) {
         title
         body
         author { login }
         comments(first: 50) {
           nodes {
             author { login }
             body
             replies(first: 20) {
               nodes {
                 author { login }
                 body
               }
             }
           }
         }
       }
     }
   }' -F owner='{owner}' -F repo='{repo}' -F number=<NUMBER>
   ```
   - Extract `{owner}` and `{repo}` from `gh repo view --json owner,name`
3. Read the full discussion thread — title, body, and all comments/replies

## Phase 2: Analyze the Question

1. Identify the questioner's language from their writing
2. Determine the core intent:
   - What is the user actually trying to achieve?
   - What specific problem are they facing?
   - What has already been answered in existing comments (if any)?
   - What remains unanswered or unclear?
3. If the discussion already has satisfactory answers, inform the user and ask whether to still reply

## Phase 3: Research the Codebase

1. Based on the question analysis, explore the relevant code:
   - Use Grep and Glob to find related files, classes, and functions
   - Read the relevant source code to understand the actual behavior
   - Check documentation, configuration files, and tests for additional context
2. Gather concrete evidence:
   - Specific file paths and line numbers
   - Code snippets that demonstrate the answer
   - Configuration examples if applicable
3. If the question involves a feature or behavior, verify it by reading the actual implementation — do not guess

## Phase 4: Draft in Korean

Present the analysis and draft reply to the user **in Korean**:

1. **질문 분석**: Summarize what the questioner is asking and why
2. **조사 결과**: Share what you found in the codebase with evidence
3. **답변 초안**: The proposed reply content (still in Korean at this stage)
4. Ask the user for feedback:
   - Is the analysis correct?
   - Should anything be added, removed, or changed?
   - Any tone adjustments needed?

## Phase 5: Translate and Confirm

1. Translate the approved draft into the questioner's language
2. Format the reply with proper markdown:
   - Use code blocks with language tags for code snippets
   - Use headers and lists for structure
   - Include file path references where helpful
3. Show the final translated reply to the user
4. Ask for confirmation before posting:
   - Options: "Post as-is", "Edit", "Cancel"

## Phase 6: Post the Reply

1. After user approval, post the comment using GraphQL:
   ```bash
   gh api graphql -f query='
   mutation($discussionId: ID!, $body: String!) {
     addDiscussionComment(input: {discussionId: $discussionId, body: $body}) {
       comment {
         url
       }
     }
   }' -f discussionId='<DISCUSSION_NODE_ID>' -f body='<REPLY_BODY>'
   ```
   - The `discussionId` is the node ID from the Phase 1 query (add `id` field to the discussion query)
2. Display the posted comment URL to the user

## Guidelines

- Always base answers on actual code investigation — never guess or assume
- The Korean draft phase is mandatory — it ensures the user (maintainer) fully understands and approves the answer before posting
- Match the questioner's language in the final reply
- Be helpful and welcoming in tone — the questioner chose to engage with the project
- Include code examples and file references when they help clarify the answer
- If the question reveals a documentation gap, mention it to the user so they can consider updating docs
- Never post without explicit user approval
