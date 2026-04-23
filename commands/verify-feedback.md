---
allowed-tools: Bash(curl *), Bash(git *), Bash(python3 *), Bash(strings *), Task, Read, Glob, Grep
description: Adversarially verify reviewer feedback on a Bitbucket Cloud PR before acting on it
disable-model-invocation: false
---

Verify reviewer comments on a Bitbucket Cloud pull request by grounding every factual claim in file evidence, classifying scope, and auditing assumptions — BEFORE the author acts on the feedback.

## When to use

- Reviewer left non-trivial comments on your PR.
- Comments span multiple repos, touch shared infrastructure, or make claims about authorship/history/API semantics.
- You suspect at least one comment is wrong, out of scope, or based on a stale assumption.

Do NOT use for: simple nits, typo corrections, style preferences. Those need no verification.

## Prerequisites

Environment variables (same as `/code-review`):
- `BITBUCKET_EMAIL`
- `BITBUCKET_API_TOKEN` (scopes: `pullrequest:read`)

## Inputs

Required:
- `PR_URL` or `(WORKSPACE, REPO_SLUG, PR_ID)` triple
- Affected repo list (defaults to the PR repo; add sibling repos if comments reference them)

## Process

### Step 1: Fetch PR + comments

```bash
REPO_SLUG=$(git remote get-url origin | sed -E 's#.*[:/]([^/]+)/([^/.]+)(\.git)?$#\1/\2#')

# PR detail
curl -s -u "$BITBUCKET_EMAIL:$BITBUCKET_API_TOKEN" \
  "https://api.bitbucket.org/2.0/repositories/$REPO_SLUG/pullrequests/$PR_ID" > /tmp/pr.json

# Comments (paginated)
curl -s -u "$BITBUCKET_EMAIL:$BITBUCKET_API_TOKEN" \
  "https://api.bitbucket.org/2.0/repositories/$REPO_SLUG/pullrequests/$PR_ID/comments?pagelen=100" > /tmp/comments.json

# Diff
curl -s -u "$BITBUCKET_EMAIL:$BITBUCKET_API_TOKEN" \
  "https://api.bitbucket.org/2.0/repositories/$REPO_SLUG/pullrequests/$PR_ID/diff" > /tmp/pr.diff
```

Extract each reviewer comment as a numbered finding. Drop resolved/deleted comments.

### Step 2: Fan out parallel agents via Task tool

Spawn these subagents IN PARALLEL (single message, multiple Task tool calls):

**(a) Per-repo verifier** — one agent per affected repo:
```
Read the PR diff and reviewer comments at /tmp/pr.diff and /tmp/comments.json.
For repo <repo-slug> at <local path>:
  - For every factual claim a reviewer made about THIS repo, read the actual file and
    confirm or refute. Quote file:line evidence. No speculation.
  - Flag claims that reference files that do not exist.
Output: table of [finding-id | claim | evidence | VERIFIED|REFUTED|UNVERIFIABLE].
```

**(b) Scope-guard agent**:
```
Read the PR scope (title + description from /tmp/pr.json) and reviewer comments.
For each finding, classify:
  - IN-SCOPE: directly addresses the PR's stated purpose
  - OUT-OF-SCOPE: valid concern but belongs in a separate PR/issue
  - AMBIGUOUS: depends on interpretation
Output: table of [finding-id | classification | one-line justification].
```

**(c) Assumption-auditor agent**:
```
Read reviewer comments at /tmp/comments.json. For each comment, extract every
assumption the reviewer made. Examples: authorship ("you wrote X"), API vs filesystem
semantics, which component is source of truth, what a name means in this codebase.
For each assumption, confirm or falsify via git log / file read / grep.
Output: table of [finding-id | assumption | CONFIRMED|FALSIFIED|UNKNOWN | evidence].
```

### Step 3: Synthesize adversarial review

Merge the three agent outputs into ONE ranked finding list:

```
| # | Finding | Verified | Scope | Assumption | Severity | Action |
|---|---------|----------|-------|------------|----------|--------|
| 1 | <short> | VERIFIED | IN    | CONFIRMED  | HIGH     | Fix    |
| 2 | <short> | REFUTED  | IN    | FALSIFIED  | N/A      | Push back with evidence |
| 3 | <short> | VERIFIED | OUT   | CONFIRMED  | LOW      | Defer to separate issue |
```

Severity rules:
- HIGH: verified + in-scope + assumption confirmed
- MEDIUM: verified + in-scope + assumption has caveat
- LOW: verified + out-of-scope (legitimate but defer)
- N/A: refuted OR assumption falsified (respond with counter-evidence, do not "fix")

### Step 4: Non-jargon summary

Produce a short plain-language summary aimed at a non-backend reader. No option lists — take a position on each finding. Format:

```
## Verify-feedback summary (PR #<id>)

Reviewer raised <N> points. After verification:
- <M> are real and fixable (HIGH/MEDIUM).
- <K> are based on incorrect assumptions — respond with counter-evidence:
  <one-line per refuted point>
- <L> are valid but out of scope — file a separate issue.

Recommended response: <what to reply on the PR>.
```

## Red Flags

- A verifier agent reports "looks fine" without quoting file:line → agent didn't actually read the file. Re-run that agent with stricter evidence requirement.
- Scope-guard says everything is IN-SCOPE → the PR description is too vague OR the agent is rubber-stamping. Tighten.
- Synthesis produces option lists ("you could either A or B") → not a verification, it's analysis paralysis. Pick per evidence.

## Exit Criteria

- Every reviewer comment has a row in the final table.
- Every HIGH/MEDIUM finding has a file:line citation.
- Every REFUTED finding has counter-evidence the author can paste into a reply.
- Summary names a specific response action, not a menu.
