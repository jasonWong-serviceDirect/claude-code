---
description: "Comprehensive PR review using parallel subagent reviewers"
argument-hint: "[review-aspects]"
allowed-tools: ["Bash", "Glob", "Grep", "Read", "Task", "Agent"]
---

# Subagent-Based PR Review

Run a comprehensive pull request review using **parallel subagents**, where each specialized reviewer runs independently and returns its findings as its final report.

**Review Aspects (optional):** "$ARGUMENTS"

## Review Workflow

### 1. Determine Review Scope

- Run `git diff --name-only` to identify changed files
- Run `git diff` to get the full diff content
- Parse arguments to see if user requested specific review aspects
- Check if a PR already exists: `gh pr view`
- Default: Run all applicable reviews

### 2. Available Review Aspects

| Aspect | Reviewer Name | Prompt File |
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

### 3. Spawn Reviewer Subagents

For each applicable review aspect, spawn a subagent using the Agent tool (named Task in older builds) with these parameters:

```
Agent(
  subagent_type="general-purpose",
  description="Review <aspect>",
  model="opus",                   // for code-reviewer, arch-detector, practices-analyzer, code-simplifier
                                  // use "sonnet" for test-analyzer, comment-analyzer, failure-hunter, type-analyzer
  prompt="<composed prompt>"
)
```

**Composing reviewer prompts:** For each reviewer, build a prompt with these three sections:

**Section 1 - Expertise:** Read the corresponding prompt file from this plugin's `prompts/` directory (paths listed in the table above). Include the full expertise content (everything below the YAML frontmatter) as the reviewer's domain knowledge.

**Section 2 - Scope:** Include:
- The list of changed files from git diff
- The full diff content (or relevant portions for large diffs)
- Any user-specified focus areas

**Section 3 - Reporting Instructions:** Append these instructions to every reviewer prompt:

```
## Reporting

You are one reviewer in a parallel PR review. Perform your review, then return your findings as your final report.

1. **Perform your review**: Read the changed files, analyze the code according to your expertise, and compile your findings.

2. **Return your report**: Your final message IS your review report — it is collected and aggregated with the other reviewers' reports. Structure it as:
   - A one-line summary: "N issues found: X critical, Y important"
   - Findings grouped by severity (Critical / Important / Suggestions / Positive Observations)
   - For each finding: file:line, description, why it matters, and a suggested fix

IMPORTANT: Focus only on your area of expertise. Be thorough but filter aggressively - quality over quantity.
```

**CRITICAL: Spawn ALL reviewer subagents in a single response** by making multiple parallel Agent tool calls. This maximizes parallelism - all reviewers start simultaneously.

### 4. Collect Results

- Each subagent's final report is returned as its result when it completes
- Wait for all reviewers to finish before aggregating
- If a reviewer fails or returns an empty report, note it in the final summary rather than re-running the whole review

### 5. Aggregate Results

After ALL reviewers have returned their findings, aggregate into a candidate list organized by severity:

- **Critical Issues** (must fix before merge) - across all reviewers
- **Important Issues** (should fix) - across all reviewers
- **Suggestions** (nice to have) - across all reviewers
- **Positive Observations** (what's good) - across all reviewers

De-duplicate any findings that multiple reviewers flagged.

### 5.5. Verify Behavioral Claims

Before presenting findings, verify every reviewer claim about **runtime behavior** by tracing the actual code. Reviewers frequently make claims like "this error is silent," "this renders in production," "no logging occurs," or "this value is never checked" — and these are often wrong because the reviewer only looked at the immediate code block without following the value through downstream calls.

For each finding that asserts a behavioral outcome:

1. **Read the actual code** at the cited location (not just the diff)
2. **Trace the variable/value through ALL downstream function calls**, not just the block where the finding is located. For example, if a reviewer says "$amUserId being null is silent," follow $amUserId into every function it's passed to and check whether any of those functions log, throw, or alert.
3. **Check both branches at every conditional** the value flows through
4. **For "renders in production" claims**: verify what actually happens at runtime when the cited condition is met — don't conflate "code is in the bundle" with "code executes and produces visible output"
5. **For "pre-existing vs introduced" claims**: check whether the behavior existed on the base branch before this PR

Mark each finding as:
- **Verified**: behavioral claim confirmed by tracing the code
- **Invalidated**: behavioral claim is wrong (explain why — e.g., "the null value flows into updateHubspot() which logs a critical error at line 181")
- **Pre-existing**: the behavior exists but was not introduced by this PR (reclassify from Critical/Important to "While you're here" suggestion)

**Remove invalidated findings from the final report entirely.** Reclassify pre-existing findings as suggestions rather than critical/important issues. Only verified findings should appear in the severity-rated sections.

### 6. Present the Report

```markdown
# PR Review Summary (Subagent Review)

## Reviewers: [list of reviewers that participated]

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

## Usage Examples

**Full review (default):**
```
/pr-review-team:review-pr
```

**Specific aspects:**
```
/pr-review-team:review-pr tests errors
# Only spawns test-analyzer and failure-hunter subagents

/pr-review-team:review-pr architecture practices
# Only spawns arch-detector and practices-analyzer subagents
```

**All aspects:**
```
/pr-review-team:review-pr all
# Spawns all 8 reviewer subagents in parallel
```

## Tips

- **Parallel by default**: Spawn all reviewers in one response so they run simultaneously
- **Run early**: Before creating PR, not after
- **Focus on changes**: Reviewers analyze git diff by default
- **Address critical first**: Fix high-priority issues before lower priority
- **Re-run after fixes**: Verify issues are resolved
- **Use specific aspects**: Target specific reviewers when you know the concern - spawning fewer subagents is faster and cheaper
- **Token cost**: Each subagent is a separate Claude instance. Using `all` spawns 8 instances. Target specific aspects when possible to reduce cost.

## Notes

- Each reviewer works independently with its own context window
- Reviewers report findings via their final subagent result - no messaging or shared task list involved
- The orchestrator (you) synthesizes all findings into a unified report
- Expertise files in `prompts/` are only loaded when this command is invoked, keeping context free in other conversations
- Subagents are simpler and lower-overhead than agent teams; the trade-off is no mid-review coordination between reviewers, which this workflow doesn't need
