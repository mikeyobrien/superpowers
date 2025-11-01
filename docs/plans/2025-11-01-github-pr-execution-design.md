# GitHub PR-Based Execution Plan Design

**Date:** 2025-11-01
**Status:** Approved

## Overview

Enable execution plans to integrate with GitHub issues and pull requests for human feedback loops during implementation.

## Goals

- **Code review integration:** Execute plan tasks and create PRs automatically so each task group gets reviewed before merging
- **Async communication:** GitHub issues/PRs serve as async feedback channel between human and Claude Code
- **PR best practices:** Configurable task grouping following pull request best practices

## Architecture

### Modular Skill Chain

Two new skills provide GitHub integration:

1. **`superpowers:plan-to-issue`** - Posts finalized plan as GitHub issue
2. **`superpowers:execute-with-prs`** - Executes task groups, creates PRs based on `---` boundaries

Existing skills remain unchanged:
- `brainstorming` / `writing-plans` - Plan creation
- `using-git-worktrees` - Workspace isolation
- `receiving-code-review` - Address PR feedback (human-initiated)
- `verification-before-completion` - Post-execution validation

### Human-in-the-Loop Workflow

```
┌─────────────────────────────────────────────────┐
│ Session 1: Planning                             │
├─────────────────────────────────────────────────┤
│ brainstorming → writing-plans → plan-to-issue   │
│                                   ↓              │
│                          [GitHub Issue Created] │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│ Session 2: Execute First Group                  │
├─────────────────────────────────────────────────┤
│ using-git-worktrees → execute-with-prs          │
│                              ↓                   │
│                      [PR Created for Group 1]   │
└─────────────────────────────────────────────────┘

[Human reviews PR, adds comments with feedback]

┌─────────────────────────────────────────────────┐
│ Session 3: Address Feedback                     │
├─────────────────────────────────────────────────┤
│ receiving-code-review → [PR Updated]            │
└─────────────────────────────────────────────────┘

[Human merges PR]

┌─────────────────────────────────────────────────┐
│ Session 4: Execute Next Group                   │
├─────────────────────────────────────────────────┤
│ execute-with-prs → [PR Created for Group 2]     │
└─────────────────────────────────────────────────┘
```

**Key principle:** Human initiates each session explicitly. No polling or monitoring by Claude Code.

## Skill Details

### `superpowers:plan-to-issue`

**Purpose:** Create GitHub issue from finalized plan

**Input:** Plan markdown from `writing-plans` skill

**Process:**
1. Extract plan title from first heading
2. Format issue body:
   ```markdown
   ## Plan
   [Full plan content]

   ## Task Groups
   - Group 1: Tasks 1-2
   - Group 2: Tasks 3-5
   - Group 3: Task 6-7
   ```
3. Create issue: `gh issue create --title "Plan: {title}" --body "{body}"`
4. Return issue URL to user

**Output:** Issue number + URL

---

### `superpowers:execute-with-prs`

**Purpose:** Execute task group and create PR

**Input:**
- Issue number (to link PR)
- Group number to execute (or "next")

**Process:**
1. Read plan from issue: `gh issue view {number} --json body`
2. Parse task groups (split on `---`)
3. Identify target group (by number or next incomplete)
4. Execute task group tasks directly
5. Create PR:
   ```bash
   gh pr create \
     --title "Group {n}: {summary}" \
     --body "Implements tasks from #{issue}\n\n## Changes\n{task_list}"
   ```
6. Return PR URL to user

**Output:** PR number + URL, instruction to review

## Plan Format Extension

**Changes to `writing-plans` output:**

Add PR boundary markers (`---`) between task groups:

```markdown
# Implementation Plan: Feature Name

## Context
[Background info]

## Tasks

### Database Layer
1. Create migration for users table
2. Add User model with validation

---

### API Layer
3. Create /api/users endpoint
4. Add authentication middleware
5. Write integration tests

---

### Frontend
6. Update UI to call new API
7. Add error handling

## Verification
[How to verify completion]
```

**Parsing logic:**
- Split on `---` (standalone, line boundaries)
- Each section between markers = one PR
- Preserve task numbering across groups
- If no `---` markers, treat entire task list as one group

**Backward compatibility:**
- Existing plans without `---` still work (single PR)
- `execute-with-prs` detects marker presence

## User Experience Flow

**Typical workflow:**

```bash
# Session 1: Planning
user: "Help me add user authentication"
claude: [Uses brainstorming → writing-plans]
claude: "Plan ready. Create GitHub issue?"
user: "yes"
claude: [Uses plan-to-issue]
claude: "Issue created: https://github.com/user/repo/issues/42"

# Session 2: Execute first group
user: "Execute group 1 from #42"
claude: [Uses using-git-worktrees → execute-with-prs]
claude: "PR created: https://github.com/user/repo/pull/15
         Review and provide feedback when ready"

# Session 3: Address feedback
user: "Address feedback on #15"
claude: [Uses receiving-code-review]
claude: "Changes pushed to PR #15"

# (Human merges #15)

# Session 4: Next group
user: "Execute next group from #42"
claude: [Uses execute-with-prs]
claude: "PR created: https://github.com/user/repo/pull/16"
```

**State tracking:**
- Issue tracks overall plan
- PR descriptions link back to issue (`Implements tasks from #42`)
- Human tracks which groups are complete (via merged PRs)
- No persistence needed in Claude Code sessions

**Error handling:**
- If `gh` not installed: Error with install instructions
- If not in git repo: Error directing to clone/init
- If no remote configured: Error with setup instructions

## Implementation Considerations

**Dependencies:**
- `gh` CLI (GitHub official CLI tool)
- Git repository with remote configured
- GitHub authentication via `gh auth login`

**Skill file locations:**
```
skills/plan-to-issue/SKILL.md
skills/execute-with-prs/SKILL.md
```

**Changes to existing skills:**
- `writing-plans`: Document `---` boundary marker convention in skill
- No code changes to existing skills needed

**Testing approach:**
- Create test repo with example plan
- Verify issue creation with `plan-to-issue`
- Verify PR creation with `execute-with-prs`
- Test boundary cases (no markers, single task, empty groups)

**Future enhancements (not in initial design):**
- Auto-detect optimal PR boundaries based on file change patterns
- Support for milestone linking
- Draft PR creation option
- PR template integration

**Out of scope:**
- Automated merging
- CI/CD integration (handled by repo config)
- Multi-repo coordination

## References

- [GitHub Issues as a Notebook System](https://simonwillison.net/2025/May/26/notes/) - Inspiration for using issues as async communication channel
- Hacker News discussion on GitHub issues workflows
