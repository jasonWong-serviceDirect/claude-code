---
description: "Comprehensive PR review using an agent team with parallel reviewers"
argument-hint: "[review-aspects]"
allowed-tools: ["Bash", "Glob", "Grep", "Read", "Task", "TeamCreate", "TeamDelete", "TaskCreate", "TaskList", "TaskUpdate", "TaskGet", "SendMessage"]
---

# Team-Based PR Review

Run a comprehensive pull request review using an **agent team** where specialized reviewers work as persistent teammates in parallel, coordinating through a shared task list and messaging.

**Review Aspects (optional):** "$ARGUMENTS"

## Review Workflow

### 1. Determine Review Scope

- Run `git diff --name-only` to identify changed files
- Run `git diff` to get the full diff content
- Parse arguments to see if user requested specific review aspects
- Check if a PR already exists: `gh pr view`
- Default: Run all applicable reviews

### 2. Available Review Aspects

| Aspect | Teammate Name | Agent File |
|--------|--------------|------------|
| `code` | code-reviewer | agents/code-reviewer.md |
| `tests` | test-analyzer | agents/pr-test-analyzer.md |
| `comments` | comment-analyzer | agents/comment-analyzer.md |
| `errors` | failure-hunter | agents/silent-failure-hunter.md |
| `types` | type-analyzer | agents/type-design-analyzer.md |
| `architecture` | arch-detector | agents/architecture-smell-detector.md |
| `practices` | practices-analyzer | agents/best-practices-analyzer.md |
| `simplify` | code-simplifier | agents/code-simplifier.md |
| `all` | All of the above | (default) |

### 3. Create the Review Team

Use TeamCreate:

```
TeamCreate(team_name="pr-review", description="Parallel PR review team")
```

### 4. Create Review Tasks

For each applicable review aspect, create a task with TaskCreate. Each task should contain:

- **subject**: Descriptive title (e.g., "Review code quality and project guidelines compliance")
- **description**: Include the list of changed files and what the reviewer should focus on. Be specific about the scope.
- **activeForm**: Present continuous form (e.g., "Reviewing code quality...")

Create ALL tasks before spawning teammates so the task list is populated when teammates start.

### 5. Spawn Reviewer Teammates

For each review aspect, spawn a teammate using the Task tool with these parameters:

```
Task(
  subagent_type="general-purpose",
  team_name="pr-review",
  name="<teammate-name>",        // from the table above
  model="opus",                   // for code-reviewer, arch-detector, practices-analyzer, code-simplifier, react-practices, web-guidelines
                                  // use "sonnet" for test-analyzer, comment-analyzer, failure-hunter, type-analyzer
  prompt="<composed prompt>"
)
```

**Composing teammate prompts:** For each teammate, build a prompt with these three sections:

**Section 1 - Expertise:** Read the corresponding agent file from this plugin's `agents/` directory (paths listed in the table above). Include the agent's full expertise content (everything below the YAML frontmatter) as the teammate's domain knowledge.

**Section 2 - Scope:** Include:
- The list of changed files from git diff
- The full diff content (or relevant portions for large diffs)
- Any user-specified focus areas

**Section 3 - Team Coordination:** Append these instructions to every teammate prompt:

```
## Team Workflow

You are a member of a PR review team. Follow this exact workflow:

1. **Claim your task**: Use TaskList to find available tasks, then use TaskUpdate to set your task to in_progress with yourself as owner.

2. **Perform your review**: Read the changed files, analyze the code according to your expertise, and compile your findings.

3. **Report findings**: Send your complete review report to the team lead using SendMessage:
   - type: "message"
   - recipient: "<team-lead-name>"    (read from team config if needed)
   - content: Your full structured review report
   - summary: "N issues found: X critical, Y important"

4. **Complete your task**: Use TaskUpdate to mark your task as completed.

IMPORTANT: Focus only on your area of expertise. Be thorough but filter aggressively - quality over quantity.
```

**CRITICAL: Spawn ALL teammates in a single response** by making multiple parallel Task tool calls. This maximizes parallelism - all reviewers start simultaneously.

### 6. Monitor Progress

- Teammate messages are delivered automatically as they complete reviews
- Use TaskList periodically to check overall progress if needed
- If a teammate is taking too long or seems stuck, send them a message

### 7. Aggregate Results

After ALL teammates have reported their findings, aggregate into a unified report organized by severity:

- **Critical Issues** (must fix before merge) - across all reviewers
- **Important Issues** (should fix) - across all reviewers
- **Suggestions** (nice to have) - across all reviewers
- **Positive Observations** (what's good) - across all reviewers

De-duplicate any findings that multiple reviewers flagged.

### 8. Present the Report

```markdown
# PR Review Summary (Team Review)

## Reviewers: [list of teammates that participated]

## Critical Issues (X found)
- **[reviewer-name]**: Issue description [`file:line`]
  - Why: explanation
  - Fix: suggestion

## Important Issues (X found)
- **[reviewer-name]**: Issue description [`file:line`]

## Suggestions (X found)
- **[reviewer-name]**: Suggestion [`file:line`]

## Strengths
- What's well-done in this PR (from all reviewers)

## Recommended Action
1. Fix critical issues first
2. Address important issues
3. Consider suggestions
4. Re-run review after fixes
```

### 9. Shutdown Team

After presenting the report:

1. Send shutdown requests to ALL teammates using SendMessage with `type: "shutdown_request"`
2. Wait for shutdown approvals
3. Use TeamDelete to clean up the team

## Usage Examples

**Full team review (default):**
```
/pr-review-team:review-pr
```

**Specific aspects:**
```
/pr-review-team:review-pr tests errors
# Only spawns test-analyzer and failure-hunter teammates

/pr-review-team:review-pr architecture practices
# Only spawns arch-detector and practices-analyzer teammates
```

**All aspects:**
```
/pr-review-team:review-pr all
# Spawns all 8 reviewer teammates in parallel
```

## Tips

- **Teams are parallel by default**: All teammates start simultaneously, no need to request parallel mode
- **Run early**: Before creating PR, not after
- **Focus on changes**: Teammates analyze git diff by default
- **Address critical first**: Fix high-priority issues before lower priority
- **Re-run after fixes**: Verify issues are resolved
- **Use specific aspects**: Target specific reviewers when you know the concern - spawning fewer teammates is faster and cheaper
- **Token cost**: Each teammate is a separate Claude instance. Using `all` spawns 8 instances. Target specific aspects when possible to reduce cost.

## Notes

- Each teammate works independently with its own context window
- Teammates report findings via direct messages to the team lead
- The team lead (you) synthesizes all findings into a unified report
- Agent expertise files in `agents/` can also be used individually as subagents
- Team coordination adds some overhead vs subagents - use this when you want parallel execution and the ability for reviewers to work independently
