# PR Review Team

A team-based PR review plugin that uses Claude Code's **agent teams** feature to run specialized reviewers in parallel. Each reviewer is a persistent teammate with its own context, coordinating through a shared task list and inter-agent messaging.

## How It Differs from pr-review-toolkit

| Aspect | pr-review-toolkit (Subagents) | pr-review-team (Agent Teams) |
|--------|-------------------------------|------------------------------|
| **Execution** | Subagents launched sequentially or in parallel | Teammates run as persistent parallel workers |
| **Coordination** | Main context aggregates results | Shared task list + direct messaging |
| **Communication** | Results returned to caller | Teammates message the team lead |
| **Lifecycle** | Short-lived, gone after task | Persistent until shutdown |
| **Independence** | Subagent sees only its prompt | Each teammate has full independent context |
| **Cost** | Lower (results summarized back) | Higher (each teammate is a separate instance) |

**Use pr-review-team when:**
- You want true parallel execution across all reviewers
- You're reviewing a large PR that benefits from independent analysis
- You want reviewers to work with their own full context window

**Use pr-review-toolkit when:**
- You want sequential, focused reviews
- You're reviewing small changes
- You want lower token cost

## Quick Start

```
/pr-review-team:review-pr              # Full team review (all 8 reviewers)
/pr-review-team:review-pr tests errors # Only specific reviewers
/pr-review-team:best-practices <question>  # Single-agent best practices research
```

## Review Agents (10 total)

### Core Review Team (spawned by /review-pr)

| Agent | Focus | Model |
|-------|-------|-------|
| **code-reviewer** | Project guidelines, bugs, code quality | opus |
| **pr-test-analyzer** | Test coverage quality and completeness | inherit |
| **comment-analyzer** | Comment accuracy and maintainability | inherit |
| **silent-failure-hunter** | Silent failures and error handling | inherit |
| **type-design-analyzer** | Type design and invariants | inherit |
| **architecture-smell-detector** | Workarounds, kludges, design smells | opus |
| **best-practices-analyzer** | Current best practices research | opus |
| **code-simplifier** | Code clarity and maintainability | opus |

### Standalone Agents (available individually)

| Agent | Focus | Model |
|-------|-------|-------|
| **react-best-practices** | React performance anti-patterns | opus |
| **web-design-guidelines** | UI/UX and accessibility | opus |

## Team Architecture

When you run `/pr-review-team:review-pr`, here's what happens:

```
You (user)
  |
  v
Team Lead (main Claude instance)
  |
  |-- TeamCreate("pr-review")
  |-- TaskCreate (one per review aspect)
  |-- Task(team_name="pr-review") x N  (spawn teammates)
  |
  v
┌──────────────────────────────────────────────────┐
│                  Shared Task List                 │
│  [ ] Code Review    [ ] Test Analysis            │
│  [ ] Comment Check  [ ] Error Handling           │
│  [ ] Type Design    [ ] Architecture Smells      │
│  [ ] Best Practices [ ] Code Simplification      │
└──────────────────────────────────────────────────┘
  |          |          |          |          |
  v          v          v          v          v
┌────┐   ┌────┐   ┌────┐   ┌────┐   ┌────┐
│ R1 │   │ R2 │   │ R3 │   │ R4 │   │... │   (reviewers working in parallel)
└────┘   └────┘   └────┘   └────┘   └────┘
  |          |          |          |          |
  └──────────┴──────────┴──────────┴──────────┘
                        |
                        v
              SendMessage (findings)
                        |
                        v
                  Team Lead aggregates
                        |
                        v
                 PR Review Summary
                        |
                        v
                  TeamDelete (cleanup)
```

## Usage Patterns

### Full Team Review

```
/pr-review-team:review-pr
```

Spawns all 8 core reviewers in parallel. Each reviewer independently analyzes the changed code from its perspective, then reports findings back to the team lead.

### Targeted Review

```
/pr-review-team:review-pr tests errors architecture
```

Only spawns the requested reviewers. Faster and cheaper than a full review.

### Best Practices Research

```
/pr-review-team:best-practices Is this the right way to handle auth?
```

Launches a single best-practices-analyzer agent (not a full team - overkill for one reviewer).

### Individual Agent Use

All agents are also available as standalone subagents, triggered automatically by context:

```
"Check if the tests cover all edge cases"     -> pr-test-analyzer
"Review the error handling"                    -> silent-failure-hunter
"Is this type well-designed?"                  -> type-design-analyzer
"Check my React code for performance issues"   -> react-best-practices
```

## Agent Details

### Confidence and Scoring

| Agent | Scoring Method |
|-------|---------------|
| code-reviewer | 0-100 confidence (reports >= 80) |
| pr-test-analyzer | 1-10 criticality rating |
| comment-analyzer | Issue categorization (Critical/Improvement/Removal) |
| silent-failure-hunter | CRITICAL/HIGH/MEDIUM severity |
| type-design-analyzer | Four 1-10 dimension ratings |
| architecture-smell-detector | 70-100 confidence + REFACTOR_NOW/SOON/TACTICAL_OK |
| best-practices-analyzer | HIGH/MEDIUM/LOW severity with source citations |
| code-simplifier | Complexity reduction recommendations |
| react-best-practices | CRITICAL/HIGH/MEDIUM/LOW by performance impact |
| web-design-guidelines | CRITICAL/HIGH/MEDIUM/LOW by accessibility impact |

### Output Format

All agents provide structured, actionable output:
- Clear issue identification with severity
- Specific file and line references
- Explanation of why it's a problem
- Concrete suggestions for improvement
- Prioritized by severity

## Workflow Integration

**Before committing:**
```
1. Write code
2. Run: /pr-review-team:review-pr code errors
3. Fix any critical issues
4. Commit
```

**Before creating PR:**
```
1. Stage all changes
2. Run: /pr-review-team:review-pr all
3. Address all critical and important issues
4. Run targeted reviews again to verify
5. Create PR
```

## Tips

- **Teams are parallel by nature**: All teammates start simultaneously
- **Token cost scales linearly**: 8 reviewers = ~8x the tokens of one reviewer. Target specific aspects when possible.
- **Run early**: Before creating PR, not after
- **Address critical first**: Agents prioritize findings by severity
- **Iterate**: Run again after fixes to verify
- **Individual agents are cheaper**: For a quick check on one aspect, use the agent directly instead of the team command

## License

MIT
