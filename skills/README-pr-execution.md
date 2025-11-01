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
