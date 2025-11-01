---
name: plan-to-issue
description: Use after writing-plans to post finalized plan as GitHub issue - extracts plan content, formats issue body with task groups, creates issue via gh CLI
---

# Plan to Issue

## Overview

Create GitHub issue from finalized implementation plan for human review and tracking.

**Core principle:** Plans become GitHub issues for async communication and progress tracking.

**Announce at start:** "I'm using the plan-to-issue skill to create a GitHub issue for this plan."

## Input

**Required:**
- Plan file path (e.g., `docs/plans/2025-11-01-feature.md`)

**Optional:**
- Custom issue title (defaults to extracted from plan)

## The Process

### Step 1: Verify Prerequisites

Check that required tools and context are available:

```bash
# Check gh CLI installed
gh --version

# Check authenticated
gh auth status

# Check in git repo with remote
git remote -v
```

If any check fails, provide clear error with installation/setup instructions.

### Step 2: Extract Plan Content

Read plan file (provided as input or from `docs/plans/*.md`):
- Extract title from first `#` heading
- Capture full plan content

### Step 3: Parse Task Groups

Identify task groups by splitting on `---` markers:
- Split plan content on standalone `---` (line boundaries)
- Each section between markers = one group
- If no markers, entire task list = one group
- Number groups sequentially

### Step 4: Format Issue Body

```markdown
## Plan

[Full plan content]

## Task Groups

- Group 1: Tasks X-Y
- Group 2: Tasks A-B
- Group 3: Tasks M-N
```

### Step 5: Create Issue

```bash
gh issue create \
  --title "Plan: [extracted title]" \
  --body "[formatted body]"
```

Capture issue number and URL from output.

### Step 6: Report to User

Output:
```
Issue created: #[number]
URL: [issue_url]

Use this issue number with execute-with-prs to implement task groups.
```

## Error Handling

| Error | Action |
|-------|--------|
| `gh` not installed | Show install instructions for platform |
| Not authenticated | Run `gh auth login` |
| Not in git repo | Navigate to repository first |
| No remote configured | Add remote with `git remote add` |
| Plan file not found | Specify correct path |
| Issue creation fails | Show gh error, check permissions |

## Example Usage

```
# After brainstorming and writing-plans
claude: "I'm using the plan-to-issue skill to create a GitHub issue for this plan."
claude: [Verifies gh CLI, checks auth status]
claude: [Reads docs/plans/2025-11-01-feature.md]
claude: [Parses 3 task groups from plan]
claude: [Creates issue via gh]
claude: "Issue created: #42
         URL: https://github.com/user/repo/issues/42

         Use this issue number with execute-with-prs to implement task groups."
```

## Integration

**Called by:**
- Human after `writing-plans` completes
- Optional step in workflow (plans can exist without issues)

**Pairs with:**
- `writing-plans` - Provides input plan
- `execute-with-prs` - Consumes issue number for execution
