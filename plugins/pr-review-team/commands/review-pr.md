---
description: "Comprehensive PR review using parallel subagent reviewers plus Claude Code's built-in bug review"
argument-hint: "[review-aspects] [low|medium|high|xhigh]"
allowed-tools: ["Bash", "Glob", "Grep", "Read", "Task", "Agent", "Skill"]
---

# Subagent-Based PR Review

Run a comprehensive pull request review using **parallel subagents**, where each specialized reviewer runs independently and returns its findings as its final report. Correctness bugs are handed to Claude Code's built-in `/code-review`, which hunts bugs with its own finders (and, at most effort levels, verifiers). The reviewers in this plugin add topics `/code-review` does not cover in depth: tests, comments, error handling, types, architecture, best practices, and simplification. Some overlap is expected (`/code-review` also checks CLAUDE.md rules and some cleanups); step 5 de-duplicates it.

**Arguments (optional):** "$ARGUMENTS"

## Review Workflow

### 1. Determine Review Scope

The review covers everything the branch changes: its commits (pushed and unpushed) plus uncommitted work. A bare `git diff` shows only uncommitted changes, so it misses every committed change. Do not use it alone.

1. Find the base branch. If a PR exists, use its base: `gh pr view --json baseRefName -q .baseRefName`. Otherwise use the default branch: `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`.
2. Fetch the base and find the merge base:
   ```bash
   git fetch origin <base>
   MB=$(git merge-base origin/<base> HEAD)
   ```
3. Get the changes. `git diff $MB` compares the merge base with the working tree, so one diff holds the branch's commits and the uncommitted changes to tracked files:
   ```bash
   git diff --name-only $MB
   git diff $MB
   git ls-files --others --exclude-standard   # new untracked files: read them too
   ```
4. If the diff and the untracked list are both empty, say there is nothing to review and stop.
5. Parse the arguments: review aspects (table below) and an optional effort level for `bugs` (`low`, `medium`, `high` or `xhigh`). Default: all aspects, and effort `medium` for `bugs` (with no level, `/code-review` would reuse whatever level was typed last in the session, so its cost would change from run to run). Never pass `max` or `ultra` unless the user typed it.
6. In the report, name the base branch and say whether uncommitted changes were included.

### 2. Available Review Aspects

| Aspect | Reviewer Name | How it runs |
|--------|--------------|------------|
| `bugs` | code-review | Built-in `/code-review` skill (step 3b) |
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

For each applicable aspect except `bugs`, spawn a subagent using the Agent tool (named Task in older builds) with these parameters:

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
- The list of changed files from `git diff --name-only $MB`, plus new untracked files
- The full diff content from `git diff $MB` (or relevant portions for large diffs)
- The merge-base SHA, so the reviewer can re-run the diff itself
- Any user-specified focus areas
- For code-reviewer only, when the `bugs` aspect also runs: "Correctness bugs are covered by a separate bug review. Skip Bug Detection and focus on Project Guidelines Compliance and Code Quality."

**Section 3 - Reporting Instructions:** Append these instructions to every reviewer prompt:

```
## Reporting

You are one reviewer in a parallel PR review. Perform your review, then return your findings as your final report. Your final message IS your report: it is collected and aggregated with the other reviewers' reports.

What to report:
- Report every finding you can back with a concrete failure scenario (the inputs or state that trigger it and the wrong result), or, for a cleanup finding, a concrete cost (what is duplicated, wasted, or harder to maintain, or which CLAUDE.md rule it breaks).
- Do not drop a finding because you are unsure of it. The orchestrator verifies every finding against the code. This rule overrides any confidence cutoff or "filter aggressively" instruction in your expertise section: findings that reviewers silently drop are the main cause of missed bugs.
- Do not pad. A finding with no concrete failure scenario or cost does not belong in the report.
- If the same mistake appears in several places, report it once and list the other places.

For each finding give:
- `file:line`
- severity, one of:
  - `important`: a problem this change adds or makes worse that should be fixed before merging. A rare trigger does not lower the severity.
  - `pre_existing`: a problem in the codebase that this change did not introduce. If the change re-exposes it, it is `important`.
  - `nit`: worth fixing but not blocking, for example a broken CLAUDE.md rule or a problem that harms nobody.
- summary: two or three short sentences. Start with the problem as a user of the code would see it, then where it happens and what sets it off, then "Fix:" and the change to make.
- failure scenario or cost, as described above.

Start the report with a one-line count ("N findings: X important, Y pre_existing, Z nit"). End with a short list of positive observations, if any.

Focus only on your area of expertise.
```

**CRITICAL: Spawn ALL reviewer subagents in a single response** by making multiple parallel Agent tool calls. This maximizes parallelism - all reviewers start simultaneously.

### 3b. Run the Built-in Bug Review (`bugs` aspect)

After the subagents are spawned, invoke the built-in skill with the Skill tool: `skill: "code-review"`, `args: "<effort> origin/<base>...HEAD plus uncommitted changes"`, where `<effort>` is the level from step 1 (`medium` by default). Keep the words "plus uncommitted changes": without them `/code-review` skips work the user has not committed.

- Follow the instructions the skill loads. It runs in this context while the subagents work; if your Agent calls block until they finish, it simply runs after them.
- If the skill that loads is not Claude Code's built-in bug review (for example, a personal or project skill named `code-review` that does a Standards/Spec review), do not follow it. Note in the report that the `bugs` aspect did not run and why, and continue.
- If `/code-review` reports its findings with the ReportFindings tool, that is fine. Also carry every finding into step 5 with reviewer name `code-review`. The report in step 6 is still the final output of this command.
- Do not apply fixes, post comments, or run any follow-up skill that `/code-review` suggests after its review. Continue with step 4.

### 4. Collect Results

- Each subagent's final report is returned as its result when it completes
- Wait for all reviewers and the `/code-review` pass to finish before aggregating
- If a reviewer fails or returns an empty report, note it in the final summary rather than re-running the whole review

### 5. Aggregate Results

After ALL reviewers have returned their findings, combine them into one candidate list. De-duplicate findings that several reviewers flagged (same defect, same location, same reason): keep the one with the most concrete failure scenario and list every reviewer that raised it. If the same mistake appears in several places, keep one finding and name the other places in it.

### 5.5. Verify Findings

Before presenting findings, verify each one by reading the actual code. Reviewers often make claims like "this error is silent," "this renders in production," "no logging occurs," or "this value is never checked," and these are often wrong because the reviewer only looked at the immediate code block without following the value through downstream calls.

A `/code-review` finding that already carries a CONFIRMED or PLAUSIBLE verdict passed that skill's own verifier: do not verify it again, only de-duplicate it. `/code-review` does not always run a verifier (it depends on the effort level and on whether subagents are available), so a `/code-review` finding with no verdict is checked like any other.

For every finding that needs checking:

1. **Read the actual code** at the cited location (not just the diff)
2. **Trace the variable/value through ALL downstream function calls**, not just the block where the finding is located. For example, if a reviewer says "$amUserId being null is silent," follow $amUserId into every function it's passed to and check whether any of those functions log, throw, or alert.
3. **Check both branches at every conditional** the value flows through
4. **For "renders in production" claims**: verify what actually happens at runtime when the cited condition is met. Do not conflate "code is in the bundle" with "code executes and produces visible output"
5. **For "pre-existing vs introduced" claims**: check whether the behavior existed on the base branch before this PR, and set the severity to `pre_existing` or `important` to match

Give each finding one verdict:
- **CONFIRMED**: you can name the inputs or state that trigger it and the wrong result. Quote the line.
- **PLAUSIBLE**: the mechanism is real but the trigger is uncertain (timing, environment, configuration). State what would confirm it.
- **REFUTED**: the claim is factually wrong (quote the line that shows it), provably impossible (a type, constant or invariant prevents it), already handled (cite the guard), or pure style with no observable effect.

PLAUSIBLE is the default. Do not refute a finding for being "speculative" or "depending on runtime state" when the state is realistic: a race, a null on a rare but reachable path (error handler, cold cache, missing optional field), a falsy zero treated as missing, an off-by-one on a boundary the code does not exclude, a partial failure or retry, an allowlist or regex that lost an anchor.

**Remove REFUTED findings from the report**, and give their count in one line. Keep CONFIRMED and PLAUSIBLE findings. A severity only labels and orders a finding: never drop a finding because of its severity.

### 6. Present the Report

```markdown
# PR Review Summary (Subagent Review)

Base: `<base>` (merge base `<short sha>`); uncommitted changes: included / none
Reviewers: [list of reviewers that ran, and any that failed or were skipped, with why]

## Important (X found)
- **[reviewer-name]** CONFIRMED|PLAUSIBLE: summary [`file:line`]
  - Failure scenario: inputs or state → wrong result
  - Fix: the change to make

## Pre-existing (X found)
- **[reviewer-name]** CONFIRMED|PLAUSIBLE: summary [`file:line`]
  - Fix: the change to make

## Nits (X found)
- **[reviewer-name]**: summary [`file:line`]

Refuted and removed: N findings

## Strengths
- What's well-done in this PR (from all reviewers)

## Recommended Action
1. One step per finding to fix, important ones first, each naming its `file:line` and the fix
2. Then any pre-existing findings or nits worth fixing in this PR
```

Write each Recommended Action step so that it names a specific finding: `/pr-suggest-fixes` turns these steps into PR comments.

## Usage Examples

**Full review (default):**
```
/pr-review-team:review-pr
```

**Specific aspects:**
```
/pr-review-team:review-pr tests errors
# Only spawns test-analyzer and failure-hunter subagents

/pr-review-team:review-pr bugs high
# Only runs the built-in /code-review at high effort
```

**All aspects:**
```
/pr-review-team:review-pr all
# Spawns all 8 reviewer subagents in parallel and runs /code-review
```

## Tips

- **Parallel by default**: Spawn all reviewers in one response so they run simultaneously
- **Run early**: Before creating PR, not after
- **Focus on changes**: Reviewers analyze the diff from the merge base, covering committed and uncommitted work
- **Address important first**: Fix findings that this change adds before pre-existing ones and nits
- **Re-run after fixes**: Verify issues are resolved
- **Use specific aspects**: Target specific reviewers when you know the concern - spawning fewer subagents is faster and cheaper
- **Token cost**: Each subagent is a separate Claude instance. Using `all` spawns 8 instances, and `/code-review` starts its own finder and verifier subagents at medium effort and above. Target specific aspects when possible to reduce cost.

## Notes

- Each reviewer works independently with its own context window
- Reviewers report findings via their final subagent result - no messaging or shared task list involved
- The orchestrator (you) synthesizes all findings into a unified report
- Expertise files in `prompts/` are only loaded when this command is invoked, keeping context free in other conversations
- Subagents are simpler and lower-overhead than agent teams; the trade-off is no mid-review coordination between reviewers, which this workflow doesn't need
- `/ship` also runs the built-in `/code-review`. A personal or project skill named `code-review` replaces the built-in one and stops both this command's `bugs` aspect and `/ship`'s review
