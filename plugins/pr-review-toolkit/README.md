# PR Review Toolkit

A comprehensive collection of specialized agents for thorough pull request review, covering code comments, test coverage, error handling, type design, code quality, architecture smells, best practices research, React performance patterns, web interface guidelines, and code simplification.

## Overview

This plugin bundles 10 expert review agents that each focus on a specific aspect of code quality. Use them individually for targeted reviews or together for comprehensive PR analysis.

## Agents

### 1. comment-analyzer
**Focus**: Code comment accuracy and maintainability

**Analyzes:**
- Comment accuracy vs actual code
- Documentation completeness
- Comment rot and technical debt
- Misleading or outdated comments

**When to use:**
- After adding documentation
- Before finalizing PRs with comment changes
- When reviewing existing comments

**Triggers:**
```
"Check if the comments are accurate"
"Review the documentation I added"
"Analyze comments for technical debt"
```

### 2. pr-test-analyzer
**Focus**: Test coverage quality and completeness

**Analyzes:**
- Behavioral vs line coverage
- Critical gaps in test coverage
- Test quality and resilience
- Edge cases and error conditions

**When to use:**
- After creating a PR
- When adding new functionality
- To verify test thoroughness

**Triggers:**
```
"Check if the tests are thorough"
"Review test coverage for this PR"
"Are there any critical test gaps?"
```

### 3. silent-failure-hunter
**Focus**: Error handling and silent failures

**Analyzes:**
- Silent failures in catch blocks
- Inadequate error handling
- Inappropriate fallback behavior
- Missing error logging

**When to use:**
- After implementing error handling
- When reviewing try/catch blocks
- Before finalizing PRs with error handling

**Triggers:**
```
"Review the error handling"
"Check for silent failures"
"Analyze catch blocks in this PR"
```

### 4. type-design-analyzer
**Focus**: Type design quality and invariants

**Analyzes:**
- Type encapsulation (rated 1-10)
- Invariant expression (rated 1-10)
- Type usefulness (rated 1-10)
- Invariant enforcement (rated 1-10)

**When to use:**
- When introducing new types
- During PR creation with data models
- When refactoring type designs

**Triggers:**
```
"Review the UserAccount type design"
"Analyze type design in this PR"
"Check if this type has strong invariants"
```

### 5. code-reviewer
**Focus**: General code review for project guidelines

**Analyzes:**
- CLAUDE.md compliance
- Style violations
- Bug detection
- Code quality issues

**When to use:**
- After writing or modifying code
- Before committing changes
- Before creating pull requests

**Triggers:**
```
"Review my recent changes"
"Check if everything looks good"
"Review this code before I commit"
```

### 6. code-simplifier
**Focus**: Code simplification and refactoring

**Analyzes:**
- Code clarity and readability
- Unnecessary complexity and nesting
- Redundant code and abstractions
- Consistency with project standards
- Overly compact or clever code

**When to use:**
- After writing or modifying code
- After passing code review
- When code works but feels complex

**Triggers:**
```
"Simplify this code"
"Make this clearer"
"Refine this implementation"
```

**Note**: This agent preserves functionality while improving code structure and maintainability.

### 7. architecture-smell-detector
**Focus**: Architectural smells, workarounds, and kludges

**Analyzes:**
- Type system bypasses (`as any`, non-null assertions)
- Special-case flags and escape hatches
- Missing abstractions and code duplication
- Coupling smells and circular dependencies
- Root cause vs symptom fixes

**When to use:**
- When reviewing non-trivial changes
- When code "feels hacky" or uses workarounds
- When a fix touches many unrelated files

**Triggers:**
```
"Is this the right approach?"
"This feels hacky"
"Check for architectural issues"
"Review this workaround"
```

**Note**: Provides REFACTOR_NOW / REFACTOR_SOON / TACTICAL_OK recommendations.

### 8. best-practices-analyzer
**Focus**: Researching and applying current best practices for your stack

**Analyzes:**
- Current best practices for your specific situation
- Code alignment with authoritative sources
- Stack-specific conventions and patterns
- Deviations from recommended approaches

**When to use:**
- When implementing patterns or technologies
- When unsure if approach follows conventions
- When reviewing stack-specific code

**Triggers:**
```
"Is this the right way to do X?"
"What's the best practice for Y?"
"Am I handling Z correctly?"
"Does this follow React/Next.js/etc conventions?"
```

**Note**: Unlike other agents, this one actively researches via WebSearch to find current authoritative recommendations rather than relying on memorized patterns.

### 9. react-best-practices
**Focus**: React performance patterns and anti-patterns

**Analyzes:**
- Waterfall patterns (sequential awaits)
- Bundle size issues (barrel imports, missing lazy loading)
- Re-render problems (stale closures, missing functional setState)
- Data fetching patterns (missing deduplication)
- Rendering performance (conditional render issues, missing transitions)

**When to use:**
- When reviewing React/TypeScript PRs
- When investigating React performance issues
- When refactoring React components

**Triggers:**
```
"Check my React code for best practices"
"Review React performance patterns"
"Is this component optimized?"
"Check for re-render issues"
```

**Note**: Based on Vercel Engineering guidelines adapted for Vite + React SPAs. Prioritizes issues by impact (CRITICAL > HIGH > MEDIUM > LOW).

### 10. web-design-guidelines
**Focus**: Web interface quality and accessibility

**Analyzes:**
- Accessibility (labels, aria, keyboard support, semantic HTML)
- Focus states (visible indicators, outline handling)
- Forms (autocomplete, types, error handling, labels)
- Animation (reduced motion, transform/opacity only)
- Typography (ellipsis, quotes, tabular nums)
- Performance (virtualization, preconnect, font loading)
- Anti-patterns (blocked zoom/paste, div onClick, missing dimensions)

**When to use:**
- When reviewing UI components
- When checking accessibility compliance
- When auditing UX patterns
- Before finalizing frontend PRs

**Triggers:**
```
"Review my UI"
"Check accessibility"
"Audit design patterns"
"Review UX"
"Check my site against best practices"
```

**Note**: Fetches latest guidelines from Vercel's web-interface-guidelines repo. Focuses on CRITICAL accessibility issues first.

## Usage Patterns

### Individual Agent Usage

Simply ask questions that match an agent's focus area, and Claude will automatically trigger the appropriate agent:

```
"Can you check if the tests cover all edge cases?"
→ Triggers pr-test-analyzer

"Review the error handling in the API client"
→ Triggers silent-failure-hunter

"I've added documentation - is it accurate?"
→ Triggers comment-analyzer
```

### Comprehensive PR Review

For thorough PR review, ask for multiple aspects:

```
"I'm ready to create this PR. Please:
1. Review test coverage
2. Check for silent failures
3. Verify code comments are accurate
4. Review any new types
5. General code review"
```

This will trigger all relevant agents to analyze different aspects of your PR.

### Proactive Review

Claude may proactively use these agents based on context:

- **After writing code** → code-reviewer
- **After adding docs** → comment-analyzer
- **Before creating PR** → Multiple agents as appropriate
- **After adding types** → type-design-analyzer

## Installation

Install from your personal marketplace:

```bash
/plugins
# Find "pr-review-toolkit"
# Install
```

Or add manually to settings if needed.

## Agent Details

### Confidence Scoring

Agents provide confidence scores for their findings:

**comment-analyzer**: Identifies issues with high confidence in accuracy checks

**pr-test-analyzer**: Rates test gaps 1-10 (10 = critical, must add)

**silent-failure-hunter**: Flags severity of error handling issues

**type-design-analyzer**: Rates 4 dimensions on 1-10 scale

**code-reviewer**: Scores issues 0-100 (91-100 = critical)

**code-simplifier**: Identifies complexity and suggests simplifications

**architecture-smell-detector**: Confidence 70-100 with REFACTOR_NOW/SOON/TACTICAL_OK

**best-practices-analyzer**: HIGH/MEDIUM/LOW severity with source citations

**react-best-practices**: CRITICAL/HIGH/MEDIUM/LOW by impact (waterfalls, bundle size, re-renders)

**web-design-guidelines**: CRITICAL/HIGH/MEDIUM/LOW (accessibility, focus, forms, animation)

### Output Formats

All agents provide structured, actionable output:
- Clear issue identification
- Specific file and line references
- Explanation of why it's a problem
- Suggestions for improvement
- Prioritized by severity

## Best Practices

### When to Use Each Agent

**Before Committing:**
- code-reviewer (general quality)
- silent-failure-hunter (if changed error handling)

**Before Creating PR:**
- pr-test-analyzer (test coverage check)
- comment-analyzer (if added/modified comments)
- type-design-analyzer (if added/modified types)
- architecture-smell-detector (if non-trivial changes)
- best-practices-analyzer (if implementing patterns/technologies)
- react-best-practices (if React/TypeScript code)
- web-design-guidelines (if UI components)
- code-reviewer (final sweep)

**After Passing Review:**
- code-simplifier (improve clarity and maintainability)

**During PR Review:**
- Any agent for specific concerns raised
- Targeted re-review after fixes

### Running Multiple Agents

You can request multiple agents to run in parallel or sequentially:

**Parallel** (faster):
```
"Run pr-test-analyzer and comment-analyzer in parallel"
```

**Sequential** (when one informs the other):
```
"First review test coverage, then check code quality"
```

## Tips

- **Be specific**: Target specific agents for focused review
- **Use proactively**: Run before creating PRs, not after
- **Address critical issues first**: Agents prioritize findings
- **Iterate**: Run again after fixes to verify
- **Don't over-use**: Focus on changed code, not entire codebase

## Troubleshooting

### Agent Not Triggering

**Issue**: Asked for review but agent didn't run

**Solution**:
- Be more specific in your request
- Mention the agent type explicitly
- Reference the specific concern (e.g., "test coverage")

### Agent Analyzing Wrong Files

**Issue**: Agent reviewing too much or wrong files

**Solution**:
- Specify which files to focus on
- Reference the PR number or branch
- Mention "recent changes" or "git diff"

## Integration with Workflow

This plugin works great with:
- **build-validator**: Run build/tests before review
- **Project-specific agents**: Combine with your custom agents

**Recommended workflow:**
1. Write code → **code-reviewer**
2. Fix issues → **silent-failure-hunter** (if error handling)
3. Check approach → **best-practices-analyzer** (if using specific patterns/tech)
4. Check React code → **react-best-practices** (if React/TypeScript)
5. Check UI/UX → **web-design-guidelines** (if UI components)
6. Check architecture → **architecture-smell-detector** (if non-trivial changes)
7. Add tests → **pr-test-analyzer**
8. Document → **comment-analyzer**
9. Review passes → **code-simplifier** (polish)
10. Create PR

## Contributing

Found issues or have suggestions? These agents are maintained in:
- User agents: `~/.claude/agents/`
- Project agents: `.claude/agents/` in claude-cli-internal

## License

MIT

## Author

Daisy (daisy@anthropic.com)

---

**Quick Start**: Just ask for review and the right agent will trigger automatically!
