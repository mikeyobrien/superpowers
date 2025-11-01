---
name: execute-with-prs
description: Execute task groups from GitHub issue plan and create pull requests - reads issue, parses groups, executes tasks directly, creates PR linking back to issue
---

# Execute with Pull Requests

## Overview

Execute task group from GitHub issue plan and create pull request for human review.

**Core principle:** Each task group becomes a PR for independent review and merge.

**Announce at start:** "I'm using the execute-with-prs skill to execute group [N] from issue #[number]."

## Input

- Issue number (required): GitHub issue containing plan
- Group number (optional): Which task group to execute (defaults to "next")

## Prerequisites

**Before using this skill:**
1. Plan issue must exist (created via `plan-to-issue`)
2. Must be in worktree (use `using-git-worktrees` first)
3. `gh` CLI installed and authenticated

## The Process

### Step 1: Verify Prerequisites

Check required context:

```bash
# Check gh CLI
gh --version

# Check in git worktree
git worktree list

# Verify issue exists
gh issue view [issue_number] --json title,body
```

### Step 2: Read Plan from Issue

```bash
# Fetch issue content
issue_body=$(gh issue view [issue_number] --json body --jq '.body')
```

Extract plan section from issue body (between `## Plan` heading and next heading).

### Step 3: Parse Task Groups

Split plan on `---` markers:
- Identify group boundaries
- Extract task lists for each group
- Number groups (1, 2, 3, ...)

### Step 4: Identify Target Group

Determine which group to execute:
- If group number provided explicitly: Use that group
- If "next" specified: Find first incomplete group
  - Check existing PRs linked to issue
  - Identify groups not yet implemented

### Step 5: Execute Task Group

Announce: "Executing group [N] tasks: [task_titles]"

Execute each task in the group directly:
- Follow TDD workflow if tests are involved
- Implement exactly what tasks specify
- Commit after each significant change
- Monitor for errors or blockers

### Step 6: Verify Implementation

After completing all tasks in the group:
- Run tests if specified in plan
- Check files were created/modified as expected
- Verify clean git status (all changes committed)

### Step 7: Create Pull Request

```bash
# Generate PR title from group summary
title="Group [N]: [concise summary of changes]"

# Generate PR body
body="Implements tasks from #[issue_number]

## Changes
[bulleted list of tasks completed]

## Testing
[test commands from plan verification section]"

# Create PR
gh pr create \
  --title "$title" \
  --body "$body" \
  --base main
```

Capture PR number and URL.

### Step 8: Report to User

Output:
```
Group [N] executed successfully.

PR created: #[pr_number]
URL: [pr_url]

Review the PR and provide feedback as comments.
When ready to address feedback, say: "Address feedback on #[pr_number]"
```

## Error Handling

| Error | Action |
|-------|--------|
| Issue not found | Verify issue number, check repo |
| No task groups in plan | Plan format error, needs `---` markers |
| Group number invalid | List available groups, ask which to execute |
| Executor fails | Report error, ask whether to retry or investigate |
| Tests fail after execution | Report failures, don't create PR |
| PR creation fails | Show gh error, check branch/permissions |

## Workflow Integration

### First Group Execution

```
user: "Execute group 1 from #42"
claude: "I'm using the execute-with-prs skill to execute group 1 from issue #42."
claude: [Reads issue #42, parses groups]
claude: "Executing group 1 tasks: Database schema, User model"
claude: [Executes tasks directly]
claude: [Verifies implementation]
claude: [Creates PR #15]
claude: "Group 1 executed successfully.
         PR created: #15
         URL: [url]

         Review the PR and provide feedback as comments."
```

### Subsequent Group Execution

```
user: "Execute next group from #42"
claude: [Reads issue #42]
claude: [Checks PRs: #15 merged, execute group 2]
claude: "I'm using the execute-with-prs skill to execute group 2 from issue #42."
claude: [Continues as above]
```

### Handling Feedback

Feedback handling is NOT part of this skill. Human initiates:

```
user: "Address feedback on #15"
claude: "I'm using the receiving-code-review skill to address feedback."
claude: [Uses receiving-code-review skill]
```

## Remember

- One group per execution
- Verify tests pass before creating PR
- Link PR back to issue (`#[number]`)
- Wait for human to initiate feedback handling
- Don't auto-merge or monitor PR status

## Integration

**Called by:**
- Human after reviewing issue plan
- Human after previous PR merged

**Requires:**
- `plan-to-issue` - Creates source issue
- `using-git-worktrees` - Provides isolated workspace

**Pairs with:**
- `receiving-code-review` - Addresses PR feedback (human-initiated)
- `verification-before-completion` - Final validation
