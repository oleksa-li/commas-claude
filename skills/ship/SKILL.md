---
name: ship
description: Create a GitHub PR or GitLab MR with a Claude Impact Score baked into the description, then post a work summary to the JIRA issue. Handles the full ship workflow — gather context, prompt for Claude Impact Score, create PR/MR with the score in the body, post JIRA comment. Works with both github.com (via `gh`) and GitLab instances (via `glab`). Can be extended with project-specific team members and JIRA configuration.
---

# Ship Skill

Create a GitHub PR or GitLab MR with a Claude Impact Score embedded in the description, then post a work summary to the JIRA issue.

**Score goes in the PR/MR description, not a comment.** That way CI runs once on `pull_request: opened` / `merge_request: open`, sees the score in the body, and passes — no separate comment, no re-trigger churn, no waiting for `issue_comment`-driven workflow to catch up.

Platform is auto-detected from the `origin` remote — github.com → GitHub path (`gh` CLI), any `gitlab.*` host → GitLab path (`glab` CLI).

## When to Use

- User says "ship", "create pr", "create mr", "ship it", "pr + jira", "mr + jira"
- User invokes `/ship`
- End of a feature branch workflow

## Arguments

Optional arguments override auto-detection:
- `--reviewer <name>` or `-r <name>` — override reviewer (partial name match against team list)
- `--assignee <name>` or `-a <name>` — override assignee
- `--claude-score <N>` or `--score <N>` — pre-set the Claude Impact Score (skips the interactive prompt). Must be an integer in `-3..+3`.
- `--no-claude-score` — skip the Claude Impact Score prompt entirely. CI will fail the PR/MR until a `CLAUDE: X` is added to the description or a comment.
- `--no-jira` — skip JIRA summary
- `--no-pr` — skip PR/MR creation (only post JIRA summary)
- `--platform <github|gitlab>` — force platform detection (rarely needed; use only if auto-detection picks the wrong one)
- `--slack-channel <name>` — post the review request to this Slack channel. Overrides the channel auto-picked in Step 5 (repo slug / file-extension majority / ask-the-user fallback).
- `--no-slack` — skip the Slack review-request post
- Bare argument — treated as JIRA issue key if it matches the project pattern

## Team Members

**This section should be overridden in the project-level skill.** Map git usernames to platform handles, Slack ID, and real names for auto-assigning and review-request pinging:

| Git username | GitHub handle | GitLab handle | Slack ID | Name |
|--------------|---------------|---------------|----------|------|
| `example-user` | `example-gh` | `example.gl` | `U01ABCDEFGH` | Name |

The **Slack ID** (not display name) is what lets Step 5 tag assignees with a real `<@UXXXXXX>` mention so they get a native push notification.

When the user says a name (e.g., "assign to me", "john reviews"), match it against the team list:
- "me" / "myself" → resolve via `gh api user` (GitHub) or `glab api user` (GitLab) to get the current login
- Partial name match → e.g., "john" → `john-doe`
- Exact platform username also accepted

Pick the handle from the column matching the detected platform.

## Step 0 — Detect Platform

Look at the `origin` remote URL to decide which CLI to use:

```bash
REMOTE_URL=$(git config --get remote.origin.url)
case "$REMOTE_URL" in
  *github.com*)      PLATFORM=github ;;
  *gitlab.*|*gitlab*) PLATFORM=gitlab ;;
  *) echo "Unknown remote host: $REMOTE_URL"; exit 1 ;;
esac
```

If the user passed `--platform`, use that instead.

Also confirm the matching CLI is installed and authenticated:

- **GitHub**: `gh auth status` — must be logged in
- **GitLab**: `glab auth status` — must be logged in to the correct host

If the CLI is missing, tell the user exactly what to install (`brew install gh` / `brew install glab`) and where to get auth set up. Don't try to proceed.

## Step 1 — Gather Context

Run these commands in parallel:

```bash
git status                          # Untracked / modified files
git diff --stat                     # Unstaged changes summary
git log --oneline main..HEAD        # Commits on this branch
git diff main...HEAD --stat         # Full diff stat vs main
git branch --show-current           # Current branch name
```

(If the default branch isn't `main` — e.g., `master`, `develop` — substitute accordingly. Detect via `git symbolic-ref refs/remotes/origin/HEAD`.)

Extract the JIRA issue key from:
1. Explicit argument (e.g., `PROJ-82`)
2. Branch name pattern: `feature/PROJ-XXX_...` or `fix/PROJ-XXX_...`
3. Ask user if not found

## Step 2 — Warn About Uncommitted Changes

If `git status` shows uncommitted changes:
- Show the user what's uncommitted
- Ask if they want to commit first or proceed without those changes
- Do NOT auto-commit

## Step 3 — Ask for the Claude Impact Score

**This step runs BEFORE the PR/MR is created** so the score can go into the description and CI passes on the first run.

### Skip this step if

- User passed `--no-claude-score` — warn and continue (see below)
- User passed `--claude-score <N>` / `--score <N>` — validate `N` is an integer in `-3..+3`, then use it directly

### Display the scale

Show this compact table, then ask for a score:

```
Rate Claude's impact on this PR:

   3  Claude did all the work. You reviewed and made tiny or no adjustments.
   2  Claude did most of the work. You guided it, it delivered.
   1  Claude helped with specific parts, but you did most of the work.
   0  Claude was not used on this task at all.
  -1  Claude's suggestions needed significant rework.
  -2  Claude actively hindered progress in places.
  -3  Claude wasted your time. You'd have been faster without it.
```

### Propose a score and justification based on session context

Before asking the user, synthesize what actually happened in this session and offer a draft:

- **Suggested score**: look at how much of the diff Claude produced vs. edited-after-Claude, how many course-corrections the user made, whether the user expressed frustration or satisfaction, whether generated code required significant rework. Pick a starting score.
- **Suggested justification** (one line): summarise what Claude actually did for this PR — the concrete thing, not a category. Examples: "Claude drafted the new component and tests; I adjusted the RTK Query cache tags", "Claude proposed the migration but got transaction boundaries wrong twice".

Present it like:

```
Based on this session:
  Score: 2
  Justification: Claude drafted the new component and tests; I adjusted the RTK Query cache tags.

What's your Claude Impact Score for this PR? (accept / different score / edit justification)
```

If there's no useful context (e.g., user ran `/ship` in a fresh session), skip the draft and just ask for the score.

### Validate

- Must be an integer
- Must be in range `-3..+3` inclusive
- If invalid, re-prompt with a one-line reminder of the valid range

Negative scores are valid and expected. Do not nudge the user upward — honest data is the point. The suggested score is a draft; always accept what the user types without pushing back.

### If the user declines or skips

If the user says "skip", "no", or passed `--no-claude-score`, show this warning:

```
⚠️  No Claude Impact Score will be in the description.
   CI will fail this PR/MR until a CLAUDE: X is added (body or comment).
   You can add it after the PR/MR is created with:
     # GitHub
     gh pr edit <pr-number> --body '<new body including CLAUDE: X>'
     gh pr comment <pr-number> --body "CLAUDE: <score>"
     # GitLab
     glab mr update <mr-iid> --description '<new description including CLAUDE: X>'
     glab mr note <mr-iid> --message "CLAUDE: <score>"
```

…and continue to Step 4 without a score in the body.

## Step 4 — Create the PR/MR (with score baked in)

### Determine assignee and reviewer

- **Assignee**: Default to current login. Resolve via `gh api user` (GitHub) or `glab api user` (GitLab). Override with `--assignee` or if user says "assign to X".
- **Reviewer**: If not specified, pick the most likely reviewer from the team list (not the assignee). Override with `--reviewer`.

### Build PR/MR content

Analyze ALL commits on the branch (`git log main..HEAD`) and the full diff (`git diff main...HEAD`).

**PR/MR title**: Short (under 70 chars), imperative mood. Prefix with JIRA key: `PROJ-XXX: <description>`.

**PR/MR body** format (Claude Impact Score appears as the last block on its own lines):

```markdown
[PROJ-XXX](<jira-browse-url>/PROJ-XXX)

## Summary
- [2-4 bullet points describing what changed and why]

## Changes
- **Created**: [new files]
- **Modified**: [changed files]
- **Deleted**: [removed files]

## Test plan
- [ ] [Verification steps]

---
CLAUDE: <score>
<justification>
```

The first line must always be the Jira issue link using the key extracted from the branch name. The `CLAUDE: X` line goes at the end, separated by a horizontal rule, so it's visually obvious and never confused with surrounding prose.

Rules for the score block:
- Line reads exactly `CLAUDE: <integer>` — capital `CLAUDE`, colon, single space, integer. This is the format the CI regex keys off; don't wrap it in backticks.
- Justification (if any) goes on the very next line.
- If the user skipped the score, omit the `---` separator and the score block entirely. (CI will fail until a `CLAUDE: X` is added, as warned in Step 3.)

### Push the branch first

Before creating the PR/MR, make sure the branch is on the remote:

```bash
git push -u origin <branch-name>
```

### Create the PR/MR

**GitHub** (`$PLATFORM = github`):

```bash
gh pr create \
  --title "PROJ-XXX: <title>" \
  --assignee <github-username> \
  --reviewer <github-username> \
  --body "$(cat <<'EOF'
<body content including the CLAUDE: X block>
EOF
)"
```

**GitLab** (`$PLATFORM = gitlab`):

```bash
glab mr create \
  --title "PROJ-XXX: <title>" \
  --assignee <gitlab-username> \
  --reviewer <gitlab-username> \
  --source-branch "$(git branch --show-current)" \
  --target-branch main \
  --remove-source-branch \
  --description "$(cat <<'EOF'
<body content including the CLAUDE: X block>
EOF
)"
```

(On GitLab, swap `main` for the actual default branch if different — e.g., `master`, `develop`.)

Save the PR/MR URL and its number/iid from the output — both are needed for the next step.

Confirm to the user:

```
✓ Opened <pr-or-mr-url>
  CLAUDE: <score> embedded in description — CI will see it on the first run.
```

## Step 5 — Request Review in Slack

After the PR/MR is created, confirm the assignment and (optionally) post a review-request message to the right Slack channel. Uses the Slack MCP (`mcp__slack__slack_send_message`) when available; falls back to showing the message for manual posting.

### Skip this step if

- User passed `--no-slack`
- Slack MCP is not connected

### Pick the channel

Default channels:

- `#frontend_code_review` — UI / web / mobile PRs
- `#backend_code_review` — services / APIs / infra / shared libs

Precedence (short-circuit at the first hit):

1. `--slack-channel <name>` explicit flag — use it, no question asked.
2. **Repo-name heuristic (only when unambiguous)**: if the repo slug contains the substring `frontend`, pick `#frontend_code_review`; if it contains `backend`, pick `#backend_code_review`. Do this only when one of the two substrings appears cleanly (e.g., `app-3commas-frontend`, `quantpilot-agentic-backend`) — don't guess from weaker signals like file extensions. `.ts` is ambiguous (Node backends use it too).
3. Otherwise — **ask the user**. Don't guess.

The ask should be explicit and compact:

```
Which channel should the review request go to?
  → #frontend_code_review
  → #backend_code_review

Reply with the channel name (# optional) or "skip" to not post.
```

If the user types a custom channel name (e.g., `#data-eng`), accept it. If they type `skip`, behave as if `--no-slack` was passed.

Either way — auto-picked or asked — show the final channel before posting and let the user change it one last time if they want.

### Confirm assignee + reviewer

Show a one-line summary of what was set at PR/MR creation time and let the user adjust before Slack goes out:

```
Assignee: <name> (@<handle>)
Reviewer: <name> (@<handle>)
Channel:  #<channel-name>

Confirm? (y / change / skip)
```

If the user changes the assignee or reviewer here, update the PR/MR before posting Slack:

**GitHub**:
```bash
gh pr edit <pr-number> --add-reviewer <github-username>
gh pr edit <pr-number> --add-assignee <github-username>
```

**GitLab**:
```bash
glab mr update <mr-iid> --reviewer <gitlab-username>
glab mr update <mr-iid> --assignee <gitlab-username>
```

### Compose the Slack message

Short, scannable, link-first. **Tag the assignee with a real Slack mention** (`<@UXXXXXX>`) so they get a native push, not a plain-text name.

```
<@assignee-slack-id> — new review request on <pr-or-mr-title>
<pr-or-mr-url>
Reviewer: <@reviewer-slack-id-or-name>  •  JIRA: <jira-browse-url>/<PROJ-XXX>
```

Rules:
- The assignee mention (`<@UXXXXXX>`) must be the very first token of the message. Slack treats it as a proper @-mention and will notify that user. Don't wrap it in code-span/backticks and don't use the plain `@name` form — those don't trigger pings.
- The reviewer's Slack ID is nice-to-have but not required for pinging (the assignee is the one on the hook). If you have it, use `<@UYYYYYY>`; otherwise the GitHub/GitLab handle in plain text is fine.
- The team list (Team Members section) should carry a `Slack ID` column alongside git/GitHub/GitLab handles so this lookup is one hop. If the assignee has no Slack ID on record, fall back to `@<handle>` as plain text and warn the user that the mention won't be a real ping.

### Post

Call the Slack MCP:

```
mcp__slack__slack_send_message(channel="<channel-name>", text=<message>)
```

Confirm to the user:
```
✓ Posted review request to #<channel-name>
```

### If Slack MCP is down or missing

1. Show the composed message in a code block
2. Tell the user to paste it into `#<channel-name>` manually
3. Suggest running `/mcp` to reconnect Slack

## Step 6 — Post JIRA Summary

Use `mcp__atlassian__getAccessibleAtlassianResources` to get the cloud ID, then `mcp__atlassian__addCommentToJiraIssue` to post.

### JIRA comment format

```markdown
## Work Summary

### Completed
- [List of completed items with brief descriptions]

### Code Changes
- **Created**: [new files]
- **Modified**: [changed files]
- **Deleted**: [removed files]

### Testing & Quality
- All checks pass (typecheck, lint, stylelint, tests, knip)

### Links
- PR/MR: <pr-or-mr-url>
```

Keep it concise. Focus on what was accomplished, not every detail.

### If Atlassian MCP fails

1. Show the generated JIRA comment to the user in a code block
2. Tell them to post it manually
3. Suggest running `/mcp` to reconnect

## Step 7 — Report Results


Output a summary:
```
Platform: <github|gitlab>
PR/MR: <url>
Claude Impact Score: <score> (embedded in description)
Slack: posted to #<channel-name> (or "skipped — see message above")
JIRA: PROJ-XXX comment posted (or "manual post needed")
Assignee: <name>
Reviewer: <name>
```

## Error Handling

- **No commits on branch**: Warn and abort — nothing to ship.
- **PR/MR already exists**: Show the existing URL. Ask if user wants to update the JIRA comment only. If the existing PR/MR lacks a score, offer to add it via `gh pr edit` / `glab mr update`.
- **Push fails**: Show the error. Don't retry automatically.
- **CLI not installed / not authenticated**: Stop with clear install + auth instructions (`gh auth login` / `glab auth login --hostname <host>`). Never silently fall back to the other platform.
- **Platform auto-detection ambiguous**: If `origin` URL matches neither github.com nor any gitlab host, ask the user to pass `--platform`.
- **Slack channel ambiguous**: If repo name + file extensions don't resolve to frontend or backend, ask the user to pick or pass `--slack-channel`. Don't guess.
- **Slack MCP down**: Show the composed review-request message; tell the user to paste it manually.
- **Atlassian MCP down**: Fall back to showing the comment for manual posting.

## Important Notes

- NEVER force-push
- NEVER amend commits
- NEVER skip pre-commit hooks
- Always push before creating the PR/MR
- Order is fixed: prompt for score → create PR/MR with score in body → confirm assignee + post Slack review request → JIRA summary. Prompting for the score first and baking it into the description means CI passes on the first run — no `issue_comment` re-trigger, no waiting for a reminder bot to notice, no extra pipeline spend.
- The score line must match the CI-enforced format exactly: capital `CLAUDE`, colon, single space, integer. Keep it unwrapped (no backticks around it) so the CI regex matches cleanly.
- Don't include sensitive info (API keys, tokens) in PR/MR or JIRA
