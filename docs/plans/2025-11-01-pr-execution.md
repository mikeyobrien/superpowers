# PR-Based Execution Plan Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add two new skills (`plan-to-issue` and `execute-with-prs`) that enable GitHub issue/PR-based execution workflows with human feedback loops.

**Architecture:** Modular skills that integrate with existing workflow. `plan-to-issue` posts finalized plans as GitHub issues. `execute-with-prs` executes task groups (separated by `---` markers) and creates PRs. Human provides feedback via PR comments, which agent addresses using existing `receiving-code-review` skill.

**Note:** Originally designed to use executor subagent, but simplified to direct execution within the skill for better clarity and control. The skill still follows TDD workflow and proper task execution practices.

**Tech Stack:** Bash scripts, `gh` CLI, markdown parsing, existing superpowers skill framework

---

## Task 1: Create plan-to-issue skill structure

**Files:**
- Create: `skills/plan-to-issue/SKILL.md`

**Step 1: Create skill directory**

```bash
mkdir -p skills/plan-to-issue
```

**Step 2: Write skill documentation**

Create `skills/plan-to-issue/SKILL.md` with:
- Skill metadata (name, description)
- Overview explaining purpose
- Input/output specifications
- Step-by-step process
- Error handling
- Examples

**Step 3: Commit**

```bash
git add skills/plan-to-issue/
git commit -m "feat: add plan-to-issue skill structure"
```

---

### Task 2: Implement plan-to-issue skill content

**Files:**
- Modify: `skills/plan-to-issue/SKILL.md`

**Step 1: Add skill metadata and overview**

```markdown
---
name: plan-to-issue
description: Use after writing-plans to post finalized plan as GitHub issue - extracts plan content, formats issue body with task groups, creates issue via gh CLI
---

# Plan to Issue

## Overview

Create GitHub issue from finalized implementation plan for human review and tracking.

**Core principle:** Plans become GitHub issues for async communication and progress tracking.

**Announce at start:** "I'm using the plan-to-issue skill to create a GitHub issue for this plan."
```

**Step 2: Add process documentation**

```markdown
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
```

**Step 3: Add error handling and examples**

```markdown
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
```

**Step 4: Commit**

```bash
git add skills/plan-to-issue/SKILL.md
git commit -m "feat: implement plan-to-issue skill documentation"
```

---

### Task 3: Create execute-with-prs skill structure

**Files:**
- Create: `skills/execute-with-prs/SKILL.md`

**Step 1: Create skill directory**

```bash
mkdir -p skills/execute-with-prs
```

**Step 2: Write skill documentation**

Create `skills/execute-with-prs/SKILL.md` with:
- Skill metadata (name, description)
- Overview explaining purpose
- Input/output specifications
- Step-by-step process
- Error handling
- Examples

**Step 3: Commit**

```bash
git add skills/execute-with-prs/
git commit -m "feat: add execute-with-prs skill structure"
```

---

### Task 4: Implement execute-with-prs skill content

**Files:**
- Modify: `skills/execute-with-prs/SKILL.md`

**Step 1: Add skill metadata and overview**

```markdown
---
name: execute-with-prs
description: Use to execute task groups from GitHub issue plan and create pull requests - reads issue, parses groups, launches executor subagent, creates PR linking back to issue
---

# Execute with Pull Requests

## Overview

Execute task group from GitHub issue plan and create pull request for human review.

**Core principle:** Each task group becomes a PR for independent review and merge.

**Announce at start:** "I'm using the execute-with-prs skill to execute group [N] from issue #[number]."

## Prerequisites

**Before using this skill:**
1. Plan issue must exist (created via `plan-to-issue`)
2. Must be in worktree (use `using-git-worktrees` first)
3. `gh` CLI installed and authenticated
```

**Step 2: Add process documentation**

```markdown
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

### Step 5: Launch Executor for Group

Announce: "Executing group [N] tasks: [task_titles]"

**REQUIRED SUB-SKILL:** Launch `executor` subagent with:
- Prompt: "Execute these tasks: [group_task_list]"
- Working directory: Current worktree
- Wait for completion

Monitor executor output for errors or blockers.

### Step 6: Verify Implementation

After executor completes:
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
```

**Step 3: Add error handling and workflow notes**

```markdown
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
claude: [Launches executor subagent]
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
- `executor` subagent - Implements tasks

**Pairs with:**
- `receiving-code-review` - Addresses PR feedback (human-initiated)
- `verification-before-completion` - Final validation
```

**Step 4: Commit**

```bash
git add skills/execute-with-prs/SKILL.md
git commit -m "feat: implement execute-with-prs skill documentation"
```

---

### Task 5: Update writing-plans skill to document PR boundary markers

**Files:**
- Modify: `skills/writing-plans/SKILL.md`

**Step 1: Read current writing-plans skill**

```bash
cat skills/writing-plans/SKILL.md
```

Identify where to add documentation about `---` markers.

**Step 2: Add PR boundary marker documentation**

Add new section after "Task Structure" section:

```markdown
## PR Boundary Markers (Optional)

**For plans that will use execute-with-prs:**

Add `---` markers (standalone, line boundaries) between task groups:

```markdown
### Task 1: Database Layer
[steps]

### Task 2: User Model
[steps]

---

### Task 3: API Endpoints
[steps]

---

### Task 4: Frontend Integration
[steps]
```

**Parsing logic:**
- Each section between `---` markers = one PR
- Task numbering continues across groups
- If no `---` markers, entire task list = one PR

**When to use:**
- Plan will be executed via `execute-with-prs` skill
- Logical separation points exist (layers, features, components)
- Want independent review/merge for each group

**When to skip:**
- Plan executed in single PR
- All tasks tightly coupled
- Small plan (< 5 tasks)
```

**Step 3: Update examples to show markers**

Find existing task structure examples and add one with `---` markers.

**Step 4: Commit**

```bash
git add skills/writing-plans/SKILL.md
git commit -m "docs: add PR boundary marker convention to writing-plans"
```

---

### Task 6: Add .claude-plugin metadata for new skills

**Files:**
- Modify: `.claude-plugin/plugin.json` (if exists) or equivalent metadata file

**Step 1: Check how skills are registered**

```bash
ls -la .claude-plugin/
cat .claude-plugin/plugin.json 2>/dev/null || echo "No plugin.json found"
```

**Step 2: Add skill entries if needed**

If skills are auto-discovered from `skills/` directory, no action needed.

If manual registration required, add entries for:
- `plan-to-issue`
- `execute-with-prs`

**Step 3: Commit if changes made**

```bash
git add .claude-plugin/
git commit -m "feat: register new PR execution skills"
```

---

### Task 7: Create README for PR execution workflow

**Files:**
- Create: `skills/README-pr-execution.md`

**Step 1: Write workflow documentation**

```markdown
# PR-Based Execution Workflow

## Overview

Execute implementation plans through GitHub issues and pull requests for human review integration.

## Skills

### plan-to-issue
Posts finalized plan as GitHub issue.

**Use when:** Plan is complete and ready for execution tracking.

### execute-with-prs
Executes task groups from issue, creates PRs.

**Use when:** Ready to implement specific task group from issue plan.

## Workflow

```
┌─────────────────────────────────────┐
│ 1. Planning Session                 │
│ brainstorming → writing-plans       │
│         ↓                            │
│   plan-to-issue                     │
│         ↓                            │
│   [Issue #N created]                │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ 2. Execute First Group              │
│ using-git-worktrees                 │
│         ↓                            │
│   execute-with-prs (group 1)        │
│         ↓                            │
│   [PR #M created]                   │
└─────────────────────────────────────┘

[Human reviews, comments feedback]

┌─────────────────────────────────────┐
│ 3. Address Feedback                 │
│ receiving-code-review               │
│         ↓                            │
│   [PR updated]                      │
└─────────────────────────────────────┘

[Human merges PR]

┌─────────────────────────────────────┐
│ 4. Execute Next Group               │
│ execute-with-prs (next)             │
│         ↓                            │
│   [PR #P created]                   │
└─────────────────────────────────────┘
```

## Example

```bash
# Session 1: Planning
user: "Help me add user authentication"
claude: [Uses brainstorming, writing-plans]
user: "Create issue for this plan"
claude: [Uses plan-to-issue]
# Output: Issue #42 created

# Session 2: Execute Group 1
user: "Execute group 1 from #42"
claude: [Uses using-git-worktrees, execute-with-prs]
# Output: PR #15 created

# Session 3: Feedback
user: "Address feedback on #15"
claude: [Uses receiving-code-review]
# Output: PR #15 updated

# Session 4: Next Group
user: "Execute next group from #42"
claude: [Uses execute-with-prs]
# Output: PR #16 created
```

## Design Document

See `docs/plans/2025-11-01-github-pr-execution-design.md` for full design.
```

**Step 2: Commit**

```bash
git add skills/README-pr-execution.md
git commit -m "docs: add PR execution workflow README"
```

---

## Verification

After all tasks complete:

1. **Verify skill files exist:**
   ```bash
   ls -la skills/plan-to-issue/SKILL.md
   ls -la skills/execute-with-prs/SKILL.md
   ```

2. **Verify documentation updated:**
   ```bash
   grep -A 5 "PR Boundary Markers" skills/writing-plans/SKILL.md
   ```

3. **Verify README created:**
   ```bash
   cat skills/README-pr-execution.md
   ```

4. **Verify all commits made:**
   ```bash
   git log --oneline --since="1 hour ago"
   ```

Expected: 6-7 commits following conventional commit format.

5. **Run any existing tests:**
   ```bash
   # Check if test suite exists
   if [ -f Makefile ]; then make test; fi
   if [ -f package.json ]; then npm test; fi
   ```

All tasks complete when verification passes.
