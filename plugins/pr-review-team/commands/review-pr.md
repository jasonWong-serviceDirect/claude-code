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

| Aspect | Teammate Name | Prompt File |
|--------|--------------|------------|
| `code` | code-reviewer | prompts/code-reviewer.md |
| `tests` | test-analyzer | prompts/pr-test-analyzer.md |
| `comments` | comment-analyzer | prompts/comment-analyzer.md |
| `errors` | failure-hunter | prompts/silent-failure-hunter.md |
| `types` | type-analyzer | prompts/type-design-analyzer.md |
| `architecture` | arch-detector | prompts/architecture-smell-detector.md |
| `practices` | practices-analyzer | prompts/best-practices-analyzer.md |
| `simplify` | code-simplifier | prompts/code-simplifier.md |
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

**Section 1 - Expertise:** Read the corresponding prompt file from this plugin's `prompts/` directory (paths listed in the table above). Include the full expertise content (everything below the YAML frontmatter) as the teammate's domain knowledge.

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

### 8.5 Diff Review Playground (MANDATORY)

**You MUST invoke the `playground` skill after presenting the report.** This is not optional. Use the Skill tool to invoke the `playground` skill with the diff-review template, passing all issues and their suggested fixes as arguments.

#### Playground Requirements

The playground must support an **interactive PR commenting workflow**:

1. **Issue cards with Accept/Skip**: Each finding has Accept/Skip buttons in the sidebar.

2. **Editable comment preview**: When a user accepts an issue, show an **editable textarea** pre-filled with a suggested review comment for that issue. The comment should be concise and actionable (e.g., "This sync is not wrapped in a transaction. If `syncLeadIfApplicable()` throws, the call attribute update is already committed while the lead column stays stale. Consider wrapping in `beginTransaction()`/`commit()`/`rollBack()`.").

3. **Target line display**: Next to each editable comment, show the **file path and line number** where the comment will be posted on the PR. Pick the most relevant line from the diff (typically the first changed line related to the issue).

4. **Diff toggle**: Toggle between "Current" and "Suggested" code views for issues that have a suggested fix.

5. **Generate button**: A prominent "Generate PR Comments" button at the bottom of the prompt panel. When clicked, it generates a **copyable `gh api` command using `--input` with a JSON heredoc** that will post all accepted (and potentially edited) comments as a PR review. Each comment body MUST include a GitHub suggestion block with the suggested fix code, so the PR author can click "Commit suggestion" directly in the GitHub UI to accept the change.

**IMPORTANT**: Do NOT use `-f 'comments[]={...}'` — that format sends the JSON object as a string and GitHub silently drops the comments. Instead, use `--input -` with a heredoc to send a proper JSON body:

```bash
gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews \
  --method POST \
  --input - <<'EOF'
{
  "event": "COMMENT",
  "body": "a few suggestions",
  "comments": [
    {
      "path": "Dao/CallAttribute.php",
      "line": 37,
      "body": "The comment explaining the issue.\n\n```suggestion\nthe suggested replacement code for that line\n```"
    },
    {
      "path": "Dao/CallAttribute.php",
      "start_line": 56,
      "line": 61,
      "body": "Another comment.\n\n```suggestion\nsuggested code\n```"
    }
  ]
}
EOF
```

The `body` field of the review MUST always be `"a few suggestions"` — never anything else. Each comment's `body` field must contain the review comment text followed by a GitHub `suggestion` code block with the corrected code. For multi-line suggestions, use the `start_line` field alongside `line` to specify the range. All strings in the JSON must use `\n` for newlines (not literal newlines within string values).

6. **Copy flow**: After clicking "Generate PR Comments", the commands appear in the prompt panel with a Copy button. The user copies them and pastes into Claude Code to execute. This is the bridge between the browser-based playground and the CLI.

#### Data to pass to the playground skill

When invoking the playground, include for each issue:
- The **PR number**, repo owner, and repo name
- The severity level and title
- The **file path and diff line number** where the comment should land (include `start_line` if the suggestion spans multiple lines)
- The current code (what the PR has now)
- The **suggested fix code** (this is critical — it becomes the content of the GitHub `suggestion` block, allowing the PR author to click "Commit suggestion" to accept the fix)
- A **pre-written review comment** (concise, 1-3 sentences explaining the issue — the suggestion block with the fix will be appended automatically)

This step is critical because it transforms a wall-of-text review into an interactive tool where the user can curate, edit, and post review comments directly to the PR. Do NOT skip this step.

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
- Expertise files in `prompts/` are only loaded when this command is invoked, keeping context free in other conversations
- Team coordination adds some overhead vs subagents - use this when you want parallel execution and the ability for reviewers to work independently
