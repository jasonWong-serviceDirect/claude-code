---
description: "Interactive diff review dashboard — curate, edit, and post PR review comments"
allowed-tools: ["Bash", "Glob", "Grep", "Read", "Skill"]
---

# PR Review Dashboard

Create an interactive diff review playground from the current conversation's PR review findings. This command is designed to run **after** `/pr-review-team:review-pr` has completed, but can also work standalone by reading the git diff and analyzing it.

## Workflow

### 1. Gather Review Data

Check the current conversation for PR review findings. If a review was already performed (e.g., via `/pr-review-team:review-pr`), extract all reported issues with their severity, file paths, line numbers, current code, and suggested fixes.

If no prior review exists in the conversation:
- Run `git diff --name-only` and `git diff` to get the current changes
- Run `gh pr view` to get the PR number, repo owner, and repo name
- Ask the user what they'd like to review, or note that this command works best after running `/pr-review-team:review-pr` first

### 2. Gather PR Metadata

- Run `gh pr view --json number,headRepository` to get the PR number
- Run `gh repo view --json owner,name` to get the repo owner and name
- Run `git diff` to get the full diff content

### 3. Invoke the Playground Skill

Use the Skill tool to invoke the `playground` skill with the diff-review template. Pass all the data described below.

## Playground Requirements

The playground must support an **interactive PR commenting workflow**:

1. **Issue cards with Accept/Skip**: Each finding has Accept/Skip buttons in the sidebar.

2. **Editable comment preview**: When a user accepts an issue, show an **editable textarea** pre-filled with a suggested review comment for that issue. The comment should be concise and actionable (e.g., "This sync is not wrapped in a transaction. If `syncLeadIfApplicable()` throws, the call attribute update is already committed while the lead column stays stale. Consider wrapping in `beginTransaction()`/`commit()`/`rollBack()`.").

3. **Target line display**: Next to each editable comment, show the **file path and line number** where the comment will be posted on the PR. Pick the most relevant line from the diff (typically the first changed line related to the issue).

4. **Diff toggle**: Toggle between "Current" and "Suggested" code views for issues that have a suggested fix.

5. **Review type selector**: A segmented control at the top of the prompt panel with three options: **Comment**, **Approve**, and **Request Changes**. Default to "Comment". The selected value maps to the `event` field in the generated `gh api` command (`"COMMENT"`, `"APPROVE"`, or `"REQUEST_CHANGES"`).

6. **Generate button**: A prominent "Generate PR Comments" button at the bottom of the prompt panel. When clicked, it generates a **copyable `gh api` command using `--input` with a JSON heredoc** that will post all accepted (and potentially edited) comments as a PR review using the selected review type from the selector above. Each comment body MUST include a GitHub suggestion block with the suggested fix code, so the PR author can click "Commit suggestion" directly in the GitHub UI to accept the change.

**IMPORTANT**: Do NOT use `-f 'comments[]={...}'` — that format sends the JSON object as a string and GitHub silently drops the comments. Instead, use `--input -` with a heredoc to send a proper JSON body:

```bash
gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews \
  --method POST \
  --input - <<'EOF'
{
  "event": "COMMENT or APPROVE or REQUEST_CHANGES (from review type selector)",
  "body": "review summary text",
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

The `event` field must match the user's selection from the review type selector: `"COMMENT"`, `"APPROVE"`, or `"REQUEST_CHANGES"`. The `body` field must correspond to the selected type: `"looks good, just a few comments"` for COMMENT, `"looks good, just a few suggestions"` for REQUEST_CHANGES, and `"looks good, approved with suggestions"` for APPROVE. Each comment's `body` field must contain the review comment text followed by a GitHub `suggestion` code block with the corrected code. For multi-line suggestions, use the `start_line` field alongside `line` to specify the range.

**CRITICAL JSON formatting rule**: Every `"body"` string value MUST be a single line in the JSON — no literal newlines inside the string. Use `\n` escape sequences for line breaks. Literal newlines inside JSON strings are invalid and will cause a parse error, resulting in comments being silently dropped. This is especially important for suggestion blocks where multi-line code must be joined with `\n`.

7. **Copy flow**: After clicking "Generate PR Comments", the commands appear in the prompt panel with a Copy button. The user copies them and pastes into Claude Code to execute. This is the bridge between the browser-based playground and the CLI.

## Main Panel Layout (CRITICAL — read carefully)

The playground has a **two-panel layout**:

- **Left sidebar**: Issue cards with Accept/Skip buttons (the list of findings)
- **Right main panel**: The **actual git diff** — full code with line numbers, +/- indicators, syntax highlighting, and hunk headers, grouped by file

**The main panel MUST render the full git diff as code**, NOT a summary list of issues. This is the most common failure mode — the main panel ends up showing a duplicate of the sidebar (issue titles with severity badges) instead of the actual diff code. That is wrong.

The correct behavior:
- Main panel shows `diffData` rendered as a code diff viewer (like GitHub's diff view)
- Each file has a header with its path
- Under each file header, the diff hunks are rendered line by line with line numbers, +/- prefixes, and colored backgrounds (green for additions, red for deletions)
- When a user clicks an issue card in the sidebar, the main panel scrolls to the relevant line in the diff and highlights it
- Issues are annotated inline within the diff (e.g., a colored marker or gutter icon at the relevant line), NOT rendered as a separate list

If the diff is very large (>500 lines), include only the hunks relevant to the flagged issues, with file headers and "... (N unchanged lines)" collapse markers between them.

## Data to Pass to the Playground Skill

When invoking the playground, include:

**1. The full git diff content** — pass the raw `git diff` output (or at minimum, the hunks for all files that have issues). This is what gets rendered as code in the main panel. Structure it as `diffData` per the diff-review template:
```javascript
const diffData = [
  {
    file: "path/to/file.php",
    hunks: [
      {
        header: "@@ -41,13 +41,13 @@ function context",
        lines: [
          { type: "context", oldNum: 41, newNum: 41, content: "unchanged line" },
          { type: "deletion", oldNum: 42, newNum: null, content: "removed line" },
          { type: "addition", oldNum: null, newNum: 42, content: "added line" },
        ]
      }
    ]
  }
];
```

**2. Per-issue metadata** — for each review finding:
- The **PR number**, repo owner, and repo name
- The severity level and title
- The **file path and diff line number** where the comment should land (include `start_line` if the suggestion spans multiple lines)
- The current code (what the PR has now)
- The **suggested fix code** (this is critical — it becomes the content of the GitHub `suggestion` block, allowing the PR author to click "Commit suggestion" to accept the fix)
- A **pre-written review comment** (concise, 1-3 sentences explaining the issue — the suggestion block with the fix will be appended automatically)
