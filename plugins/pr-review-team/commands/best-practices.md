---
description: "Research and apply current best practices for your stack and situation"
argument-hint: "<situation or question>"
allowed-tools: ["Bash", "Glob", "Grep", "Read", "Task", "WebSearch", "WebFetch"]
---

# Best Practices Analysis

Research current best practices for a specific situation and stack, then evaluate code against those practices.

**Situation/Question:** "$ARGUMENTS"

## Instructions

1. **Parse the Request**
   - Identify what the user wants to know about
   - Determine the stack/technologies involved (check package.json, imports, etc.)
   - Understand the specific situation or question

2. **Launch the best-practices-analyzer Agent**

   Use the Task tool to launch the `best-practices-analyzer` agent with:
   - The user's situation/question
   - Relevant file paths to analyze
   - Stack context from the codebase

3. **Return Results**

   The agent will:
   - Research current best practices via WebSearch
   - Analyze code against authoritative sources
   - Provide situation-specific recommendations with source citations

## Usage Examples

```
/pr-review-team:best-practices Is this the right way to handle auth in Next.js?

/pr-review-team:best-practices React state management in this component

/pr-review-team:best-practices TypeScript error handling patterns

/pr-review-team:best-practices Am I using React hooks correctly here?
```

## Notes

- The agent actively researches via WebSearch rather than relying on memorized patterns
- Recommendations include source citations from authoritative sources
- Findings are rated HIGH/MEDIUM/LOW severity
- Context matters - the agent considers your specific situation, not just generic advice
