# PR Review Team

A PR review plugin that runs specialized reviewers as **parallel subagents**. Each reviewer is a short-lived subagent with its own context window that analyzes the diff from its perspective and returns its findings as its final result; the main instance aggregates everything into one report.

## How It Works

| Aspect | Design |
|--------|--------|
| **Execution** | Reviewers launched as subagents in parallel (all in one response) |
| **Coordination** | None needed - each reviewer works independently |
| **Communication** | Each subagent's final result is its review report |
| **Lifecycle** | Short-lived, gone after the review completes |
| **Independence** | Each reviewer has its own context window |
| **Cost** | Each reviewer is a separate Claude instance |

## Quick Start

```
/pr-review-team:review-pr              # Full review (all 8 reviewers)
/pr-review-team:review-pr tests errors # Only specific reviewers
/pr-review-team:best-practices <question>  # Single-agent best practices research
```

## Review Agents (10 total)

### Core Reviewers (spawned by /review-pr)

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

## Architecture

When you run `/pr-review-team:review-pr`, here's what happens:

```
You (user)
  |
  v
Orchestrator (main Claude instance)
  |
  |-- git diff (determine scope)
  |-- Agent(...) x N  (spawn reviewer subagents in parallel)
  |
  v
┌────┐   ┌────┐   ┌────┐   ┌────┐   ┌────┐
│ R1 │   │ R2 │   │ R3 │   │ R4 │   │... │   (reviewers working in parallel)
└────┘   └────┘   └────┘   └────┘   └────┘
  |          |          |          |          |
  └──────────┴──────────┴──────────┴──────────┘
                        |
                        v
          Subagent results (findings)
                        |
                        v
            Orchestrator aggregates
          and verifies behavioral claims
                        |
                        v
                 PR Review Summary
```

## Usage Patterns

### Full Review

```
/pr-review-team:review-pr
```

Spawns all 8 core reviewers in parallel. Each reviewer independently analyzes the changed code from its perspective, then returns findings to the orchestrator.

### Targeted Review

```
/pr-review-team:review-pr tests errors architecture
```

Only spawns the requested reviewers. Faster and cheaper than a full review.

### Best Practices Research

```
/pr-review-team:best-practices Is this the right way to handle auth?
```

Launches a single best-practices-analyzer agent.

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

- **Parallel by default**: All reviewers are spawned in one response and run simultaneously
- **Token cost scales linearly**: 8 reviewers = ~8x the tokens of one reviewer. Target specific aspects when possible.
- **Run early**: Before creating PR, not after
- **Address critical first**: Agents prioritize findings by severity
- **Iterate**: Run again after fixes to verify
- **Individual agents are cheaper**: For a quick check on one aspect, use the agent directly instead of the full review command

## License

MIT
